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
import { ref, watch, nextTick, onMounted, onUnmounted } from 'vue';
const fileInput = ref(null);
const uploadAreaRef = ref(null);
const previewUrl = ref('');
const canvasRef = ref(null);
const glRef = ref(null);
const extRef = ref(null);
const vaoRef = ref(null);
const uniforms = ref({}); // 存储 uniform 位置
const a_position_loc = ref(null);
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
const uniformBuffers = {
    lensF: new Float32Array(2),
    lensS: new Float32Array(3),
};
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
	    return glCoord * vec2(1.0, -1.0) / 2.0 + vec2(0.5, 0.5);
      }
      void main() {
        float scale = uLensS.z;
        float Fx = uLensF.x;
	    float Fy = uLensF.y;
        vec2 vMapping = v_ndcCoord;
        vMapping.x = v_ndcCoord.x + ((v_ndcCoord.y * v_ndcCoord.y) / scale) * v_ndcCoord.x / scale * -Fx;
	    vMapping.y = v_ndcCoord.y + ((v_ndcCoord.x * v_ndcCoord.x) / scale) * v_ndcCoord.y / scale * -Fy;
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
        console.error("WebGL not supported");
        return;
    }
    glRef.value = gl;
    const ext = gl.getExtension('OES_vertex_array_object');
    if (!ext) {
        console.warn('当前浏览器不支持 VAO 扩展！');
        return;
    }
    extRef.value = ext;
    const vertexShader = createShader(gl, gl.VERTEX_SHADER, vsSource);
    const fragmentShader = createShader(gl, gl.FRAGMENT_SHADER, fsSource);
    if (!vertexShader || !fragmentShader) return;
    program.value = gl.createProgram();
    gl.attachShader(program.value, vertexShader);
    gl.attachShader(program.value, fragmentShader);
    gl.linkProgram(program.value);
    if (!gl.getProgramParameter(program.value, gl.LINK_STATUS)) {
        console.error("Program link error:", gl.getProgramInfoLog(program.value));
        return;
    }
    a_position_loc.value = gl.getAttribLocation(program.value, "a_position");
    uniforms.value = {
        uLensS: gl.getUniformLocation(program.value, "uLensS"),
        uLensF: gl.getUniformLocation(program.value, "uLensF"),
    }
    positionBuffer.value = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer.value);
    const positions = new Float32Array([0, 0, 1, 0, 0, 1, 1, 1]);
    gl.bufferData(gl.ARRAY_BUFFER, positions, gl.STATIC_DRAW);
    const vao = ext.createVertexArrayOES();
    ext.bindVertexArrayOES(vao);
    gl.enableVertexAttribArray(a_position_loc.value);
    gl.vertexAttribPointer(a_position_loc.value, 2, gl.FLOAT, false, 0, 0);
    ext.bindVertexArrayOES(null);
    vaoRef.value = vao;
    texture.value = gl.createTexture();
    gl.bindTexture(gl.TEXTURE_2D, texture.value);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
    gl.clearColor(0.0, 0.0, 0.0, 0.0);
}
const renderFrame = () => {
    const canvas = canvasRef.value;
    const gl = glRef.value;
    const ext = extRef.value;
    if (!gl || !ext || !vaoRef.value || !imageLoaded.value) return;
    gl.useProgram(program.value);
    uniformBuffers.lensF[0] = Fx.value;
    uniformBuffers.lensF[1] = Fy.value;
    gl.uniform2fv(uniforms.value.uLensF, uniformBuffers.lensF);
    uniformBuffers.lensS[0] = a.value;
    uniformBuffers.lensS[1] = b.value;
    uniformBuffers.lensS[2] = scale.value;
    gl.uniform3fv(uniforms.value.uLensS, uniformBuffers.lensS);
    ext.bindVertexArrayOES(vaoRef.value);
    gl.bindTexture(gl.TEXTURE_2D, texture.value);
    gl.clear(gl.COLOR_BUFFER_BIT);
    gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);
    ext.bindVertexArrayOES(null);
}
const drawImageToCanvas = async () => {
    await nextTick();
    const canvas = canvasRef.value;
    const gl = glRef.value;
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
    gl.viewport(Math.round(x), Math.round(y), Math.round(finalW), Math.round(finalH));
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
const processFile = async (file) => {
    if (!file || !file.type.startsWith('image/')) return;
    if (previewUrl.value) {
        URL.revokeObjectURL(previewUrl.value);
    }
    const img = new Image();
    const objectUrl = URL.createObjectURL(file);
    imageLoaded.value = false;
    img.onload = () => {
        imgOriginalSize.value = { width: img.width, height: img.height };
        imageLoaded.value = true;
    };
    img.onerror = () => {
        console.error("图片加载失败");
        URL.revokeObjectURL(objectUrl);
    };
    img.src = objectUrl;
    previewUrl.value = objectUrl;
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
onUnmounted(() => {
    if (previewUrl.value) {
        URL.revokeObjectURL(previewUrl.value);
    }
})
watch(imageLoaded, (newVal) => {
    if (newVal) drawImageToCanvas();
})
watch([Fx, Fy, a, b, scale], () => {
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