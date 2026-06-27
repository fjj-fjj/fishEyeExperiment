<template>
    <div class="upload-area" ref="uploadAreaRef" @click="triggerFileSelect" @dragover.prevent
        @drop.prevent="handleDrop">
        <canvas ref="canvasRef" class="preview-canvas"></canvas>
        <p v-if="!previewUrl" class="upload-tip">点击或拖拽上传图片</p>
        <input type="file" ref="fileInput" accept="image/*" style="display: none;" @change="handleFileChange" />
    </div>
    <div class="slider-block">
        <span class="demonstration">fx</span>
        <el-slider v-model="fx" :min="0" :max="2000" :step="1" show-input />
        <div class="tip">{{ fx }}</div>
    </div>
    <div class="slider-block">
        <span class="demonstration">Fy</span>
        <el-slider v-model="fy" :min="0" :max="2000" :step="1" show-input />
        <div class="tip">{{ fy }}</div>
    </div>
    <div class="slider-block">
        <span class="demonstration">cx</span>
        <el-slider v-model="cx" :min="0" :max="1000" :step="1" show-input />
        <div class="tip">{{ cx }}</div>
    </div>
    <div class="slider-block">
        <span class="demonstration">cy</span>
        <el-slider v-model="cy" :min="0" :max="1000" :step="1" show-input />
        <div class="tip">{{ cy }}</div>
    </div>
    <div class="slider-block">
        <span class="demonstration">k1</span>
        <el-slider v-model="k1" :min="-1" :max="1" :step="0.01" show-input />
        <div class="tip">{{ k1 }}</div>
    </div>
    <div class="slider-block">
        <span class="demonstration">k2</span>
        <el-slider v-model="k2" :min="-1" :max="1" :step="0.001" show-input />
        <div class="tip">{{ k2 }}</div>
    </div>
    <div class="slider-block">
        <span class="demonstration">k3</span>
        <el-slider v-model="k3" :min="-1" :max="1" :step="0.0001" show-input />
        <div class="tip">{{ k3 }}</div>
    </div>
    <div class="slider-block">
        <span class="demonstration">k4</span>
        <el-slider v-model="k4" :min="-1" :max="1" :step="0.00001" show-input />
        <div class="tip">{{ k4 }}</div>
    </div>
</template>
<script setup lang="js">
import { ref, computed, watch, nextTick, onMounted } from 'vue';
const fileInput = ref(null);
const uploadAreaRef = ref(null);
const previewUrl = ref('');
const canvasRef = ref(null);
const glRef = ref(null);
const imgOriginalSize = ref({ width: 0, height: 0 });
const imgFinalSize = ref({ width: 0, height: 0 });
const imageLoaded = ref(false);
const texture = ref(null);
const program = ref(null);
const positionBuffer = ref(null);
const uniforms = ref({}); // 存储 uniform 位置
const fx = ref(0);
const fy = ref(0);
const cx = ref(1);
const cy = ref(1);
const k1 = ref(0);
const k2 = ref(0);
const k3 = ref(0);
const k4 = ref(0);
const vsSource = `
      attribute vec2 a_position;
      varying vec2 v_texCoord;
      void main() {
        v_texCoord = vec2((a_position.x + 1.0) / 2.0, (1.0 - a_position.y) / 2.0);
        gl_Position = vec4(a_position, 0.0, 1.0);
      }
    `;
const fsSource = `
      precision highp float;
      uniform sampler2D u_texture;
      uniform vec2 u_imageSize;
      uniform vec4 u_distort; // k1,k2,k3,k4
      uniform vec4 u_intrinsic; // fx,fy,cx,cy
      varying vec2 v_texCoord;
      void main() {
        float u = v_texCoord.x * u_imageSize.x;
        float v = v_texCoord.y * u_imageSize.y;
        float fx = u_intrinsic.x;
        float fy = u_intrinsic.y;
        float cx = u_intrinsic.z;
        float cy = u_intrinsic.w;
        float x = (u - cx) / fx;
        float y = (v - cy) / fy;
        float r = sqrt(x * x + y * y);
        if(r < 0.0001) { // 中心像素无畸变
            gl_FragColor = texture2D(u_texture, v_texCoord);
            return;
        }
        float theta = atan(r);
        float theta2 = theta * theta;
        float theta4 = theta2 * theta2;
        float theta6 = theta4 * theta2;
        float theta8 = theta6 * theta2;
        float k1 = u_distort.x;
        float k2 = u_distort.y;
        float k3 = u_distort.z;
        float k4 = u_distort.w;
        float theta_d = theta * (1.0 + k1*theta2 + k2*theta4 + k3*theta6 + k4*theta8);
        float scale = theta_d / r;
        float x_dist = x * scale;
        float y_dist = y * scale;
        float u_src = fx * x_dist + cx;
        float v_src = fy * y_dist + cy;
        vec2 src_tex = vec2(u_src / u_imageSize.x, v_src / u_imageSize.y);
        gl_FragColor = texture2D(u_texture, src_tex);
      }
    `;
const createShader = (gl, type, source) => {
    const shader = gl.createShader(type);
    gl.shaderSource(shader, source);
    gl.compileShader(shader);
    if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
        console.error("Shader compile error:", gl.getShaderInfoLog(shader));
        gl.deleteShader(shader);
        return null;
    }
    return shader;
}
const initWebgl = () => {
    const canvas = canvasRef.value;
    const gl =
        canvas.getContext("webgl") ||
        canvas.getContext("experimental-webgl");
    if (!gl) {
        throw new Error("WebGL not supported");
    }
    glRef.value = gl;
    const vertexShader = createShader(gl, gl.VERTEX_SHADER, vsSource);
    const fragmentShader = createShader(gl, gl.FRAGMENT_SHADER, fsSource);
    program.value = gl.createProgram();
    gl.attachShader(program.value, vertexShader);
    gl.attachShader(program.value, fragmentShader);
    gl.linkProgram(program.value);
    uniforms.value = {
        u_texture: gl.getUniformLocation(program.value, 'u_texture'),
        u_imageSize: gl.getUniformLocation(program.value, 'u_imageSize'),
        u_distort: gl.getUniformLocation(program.value, 'u_distort'),
        u_intrinsic: gl.getUniformLocation(program.value, 'u_intrinsic'),
    };
    if (!gl.getProgramParameter(program.value, gl.LINK_STATUS)) {
        console.error("Program link error:", gl.getProgramInfoLog(program.value));
    }
    positionBuffer.value = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer.value);
    const positions = new Float32Array([-1, 1, -1, -1, 1, 1, 1, -1]);
    gl.bufferData(gl.ARRAY_BUFFER, positions, gl.STATIC_DRAW);
    texture.value = gl.createTexture();
    gl.bindTexture(gl.TEXTURE_2D, texture.value);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
}
const renderFrame = () => {
    const gl = glRef.value;
    if (!gl || !imageLoaded.value) return;
    gl.useProgram(program.value);
    gl.uniform2fv(uniforms.value.u_imageSize, [imgFinalSize.value.width, imgFinalSize.value.height]);
    gl.uniform4fv(uniforms.value.u_distort, [k1.value, k2.value, k3.value, k4.value]);
    gl.uniform4fv(uniforms.value.u_intrinsic, [fx.value, fy.value, cx.value, cy.value]);
    gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer.value);
    const posLoc = gl.getAttribLocation(program.value, "a_position");
    gl.enableVertexAttribArray(posLoc);
    gl.vertexAttribPointer(posLoc, 2, gl.FLOAT, false, 0, 0);
    gl.bindTexture(gl.TEXTURE_2D, texture.value);
    gl.clear(gl.COLOR_BUFFER_BIT);
    gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);
}
const drawImageToCanvas = async () => {
    await nextTick();
    const canvas = canvasRef.value;
    const uploadArea = uploadAreaRef.value;
    const gl = glRef.value;
    if (!canvas || !uploadArea || !previewUrl.value || !gl) return;
    const boxW = uploadArea.clientWidth;
    const boxH = uploadArea.clientHeight;
    canvas.width = boxW;
    canvas.height = boxH;
    const imgW = imgOriginalSize.value.width;
    const imgH = imgOriginalSize.value.height;
    const boxRatio = boxW / boxH;
    const imgRatio = imgW / imgH;
    let finalW, finalH, x, y;
    if (imgRatio > boxRatio) {
        finalW = boxW;
        finalH = boxW / imgRatio;
        x = 0;
        y = (boxH - finalH) / 2;
    } else {
        finalH = boxH;
        finalW = boxH * imgRatio;
        y = 0;
        x = (boxW - finalW) / 2;
    }
    fx.value = finalW;
    fy.value = finalH;
    cx.value = finalW / 2;
    cy.value = finalH / 2;
    imgFinalSize.value = {
        width: finalW,
        height: finalH,
    }
    gl.viewport(x, y, finalW, finalH);
    const img = new Image();
    img.onload = () => {
        gl.bindTexture(gl.TEXTURE_2D, texture.value);
        gl.texImage2D(
            gl.TEXTURE_2D,
            0,
            gl.RGBA,
            gl.RGBA,
            gl.UNSIGNED_BYTE,
            img,
        );
        renderFrame();
    };
    img.src = previewUrl.value;
};
const getImageSize = (file) => {
    return new Promise((resolve) => {
        if (previewUrl.value) {
            URL.revokeObjectURL(previewUrl.value);
        }
        const img = new Image();
        const objectUrl = URL.createObjectURL(file);
        img.onload = () => {
            resolve({ width: img.width, height: img.height });
        };
        img.src = objectUrl;
        previewUrl.value = objectUrl;
    });
};
const processFile = async (file) => {
    if (!file || !file.type.startsWith('image/')) return;
    imageLoaded.value = false;
    const size = await getImageSize(file);
    imgOriginalSize.value = size;
    imageLoaded.value = true;
};
const triggerFileSelect = () => fileInput.value.click();
const handleFileChange = (event) => {
    const file = event.target.files[0];
    if (file) processFile(file);
};
const handleDrop = (event) => {
    const file = event.dataTransfer.files[0];
    if (file) processFile(file);
};
onMounted(() => {
    initWebgl();
})
watch(imageLoaded, (newVal) => {
    if (newVal) drawImageToCanvas();
})
watch([fx, fy, cx, cy, k1, k2, k3, k4], () => {
    if (imageLoaded.value) renderFrame();
})
</script>
<style scoped>
.upload-area {
  width: 1200px;
  height: 500px;
  border: 2px dashed #ccc;
  border-radius: 8px;
  cursor: pointer;
  position: relative;
  overflow: hidden;
}
.preview-canvas {
  width: 100%;
  height: 100%;
  pointer-events: none;
}
.upload-tip {
  color: #999;
  position: absolute;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  pointer-events: none;
  margin-block-start: 0;
  margin-block-end: 0;
}
.slider-block {
  width: 100%;
  display: flex;
  align-items: center;
}
.slider-block .el-slider {
  margin-top: 0;
  margin-left: 12px;
  margin-right: 12px;
}
.slider-block .demonstration {
  font-size: 14px;
  line-height: 44px;
  flex: 0 0 4%;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  margin-bottom: 0;
}
.slider-block .demonstration + .el-slider {
  flex: 0 0 80%;
}
.slider-block .tip {
    flex: 1;
}
</style>