<template>
    <div class="upload-area" ref="uploadAreaRef" @click="triggerFileSelect" @dragover.prevent
        @drop.prevent="handleDrop">
        <canvas ref="canvasRef" class="preview-canvas"></canvas>
        <p v-if="!previewUrl" class="upload-tip">点击或拖拽上传图片</p>
        <input type="file" ref="fileInput" accept="image/*" style="display: none;" @change="handleFileChange" />
    </div>
    <div class="slider-block">
        <span class="demonstration">Fx</span>
        <el-slider v-model="Fx" :min="-1" :max="2" :step="0.01" show-input />
        <div class="tip">{{ Fx }}</div>
    </div>
    <div class="slider-block">
        <span class="demonstration">Fy</span>
        <el-slider v-model="Fy" :min="-1" :max="2" :step="0.01" show-input />
        <div class="tip">{{ Fy }}</div>
    </div>
    <div class="slider-block">
        <span class="demonstration">a</span>
        <el-slider v-model="a" :min="0" :max="4" :step="0.001" show-input />
        <div class="tip">{{ a }}</div>
    </div>
    <div class="slider-block">
        <span class="demonstration">b</span>
        <el-slider v-model="b" :min="0" :max="4" :step="0.001" show-input />
        <div class="tip">{{ b }}</div>
    </div>
    <div class="slider-block">
        <span class="demonstration">scale</span>
        <el-slider v-model="scale" :min="0" :max="10" :step="0.001" show-input />
        <div class="tip">{{ scale }}</div>
    </div>
</template>
<script setup lang="js">
import { ref, computed, watch, nextTick, onMounted, watchEffect } from 'vue';
const fileInput = ref(null);
const uploadAreaRef = ref(null);
const previewUrl = ref('');
const canvasRef = ref(null);
const imgOriginalSize = ref({ width: 0, height: 0 });
const imageLoaded = ref(false);
const texture = ref(null);
const program = ref(null);
const positionBuffer = ref(null);
const Fx = ref(0);
const Fy = ref(0);
const a = ref(1);
const b = ref(1);
const scale = ref(1);
const vsSource = `
      attribute vec2 a_position;
      varying vec2 v_ndcCoord;
      void main() {
        v_ndcCoord = vec2(a_position * 2.0 - 1.0); // 映射 [0,1] -> [-1,1];
        gl_Position = vec4(v_ndcCoord, 0, 1);
      }
    `;
const fsSource = `
      precision mediump float;
      uniform sampler2D u_texture;
      varying vec2 v_ndcCoord;
      uniform vec3 uLensS;
      uniform vec2 uLensF;
      vec2 GLCoord2TextureCoord(vec2 glCoord) {
	    return glCoord  * vec2(1.0, -1.0)/ 2.0 + vec2(0.5, 0.5);
      }
      void main() {
        float scale = uLensS.z;
        float Fx = uLensF.x;
	    float Fy = uLensF.y;
        vec2 vMapping = v_ndcCoord;
        vMapping.x = v_ndcCoord.x + ((pow(v_ndcCoord.y, 2.0)/scale)*v_ndcCoord.x/scale)*-Fx;
	    vMapping.y = v_ndcCoord.y + ((pow(v_ndcCoord.x, 2.0)/scale)*v_ndcCoord.y/scale)*-Fy;
        vMapping *= uLensS.xy;
        vMapping /= scale;
	    vMapping = GLCoord2TextureCoord(vMapping);
        vec4 color = texture2D(u_texture, vMapping);
        gl_FragColor = color;
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
    const vertexShader = createShader(gl, gl.VERTEX_SHADER, vsSource);
    const fragmentShader = createShader(gl, gl.FRAGMENT_SHADER, fsSource);
    program.value = gl.createProgram();
    gl.attachShader(program.value, vertexShader);
    gl.attachShader(program.value, fragmentShader);
    gl.linkProgram(program.value);
    gl.uLensS = gl.getUniformLocation(program.value, "uLensS");
    gl.uLensF = gl.getUniformLocation(program.value, "uLensF");
    if (!gl.getProgramParameter(program.value, gl.LINK_STATUS)) {
        console.error("Program link error:", gl.getProgramInfoLog(program.value));
    }
    positionBuffer.value = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer.value);
    const positions = new Float32Array([0, 0, 1, 0, 0, 1, 1, 1]);
    gl.bufferData(gl.ARRAY_BUFFER, positions, gl.STATIC_DRAW);
    texture.value = gl.createTexture();
    gl.bindTexture(gl.TEXTURE_2D, texture.value);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
}
const resizeWebglDrawRegion = (x, y, width, height) => {
    const canvas = canvasRef.value;
    const gl =
        canvas.getContext("webgl") ||
        canvas.getContext("experimental-webgl");
    gl.viewport(x, y, width, height);
}
const render = (source) => {
    const canvas = canvasRef.value;
    const gl =
        canvas.getContext("webgl") ||
        canvas.getContext("experimental-webgl");
    gl.bindTexture(gl.TEXTURE_2D, texture.value);
    gl.texImage2D(
        gl.TEXTURE_2D,
        0,
        gl.RGBA,
        gl.RGBA,
        gl.UNSIGNED_BYTE,
        source,
    );
    gl.useProgram(program.value);
    gl.uniform2fv(gl.uLensF, [Fx.value, Fy.value]);
    gl.uniform3fv(gl.uLensS, [
        a.value,
        b.value,
        scale.value,
    ]);
    gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer.value);
    const posLoc = gl.getAttribLocation(program.value, "a_position");
    gl.enableVertexAttribArray(posLoc);
    gl.vertexAttribPointer(posLoc, 2, gl.FLOAT, false, 0, 0);
    gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);
}
const drawImageToCanvas = async () => {
    await nextTick();
    const canvas = canvasRef.value;
    const uploadArea = uploadAreaRef.value;
    if (!canvas || !uploadArea || !previewUrl.value) return;
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
    const img = new Image();
    img.onload = () => {
        resizeWebglDrawRegion(x, y, finalW, finalH);
        render(img);
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
watch([imageLoaded, Fx, Fy, a, b, scale], ([imageLoaded]) => {
    if (imageLoaded) drawImageToCanvas();
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