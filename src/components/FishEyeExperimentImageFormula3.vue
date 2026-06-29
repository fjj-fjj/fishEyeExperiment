<template>
    <div class="upload-area" ref="uploadAreaRef">
        <canvas ref="canvasRef" class="preview-canvas" @wheel.prevent="handleWheel"></canvas>
    </div>
    <input type="file" ref="fileInput" accept="image/*" @change="handleFileChange" />
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
        <el-slider v-model="k1" :min="-1" :max="1" :step="0.00001" show-input />
        <div class="tip">{{ k1 }}</div>
    </div>
    <div class="slider-block">
        <span class="demonstration">k2</span>
        <el-slider v-model="k2" :min="-1" :max="1" :step="0.00001" show-input />
        <div class="tip">{{ k2 }}</div>
    </div>
    <div class="slider-block">
        <span class="demonstration">k3</span>
        <el-slider v-model="k3" :min="-1" :max="1" :step="0.00001" show-input />
        <div class="tip">{{ k3 }}</div>
    </div>
    <div class="slider-block">
        <span class="demonstration">k4</span>
        <el-slider v-model="k4" :min="-1" :max="1" :step="0.00001" show-input />
        <div class="tip">{{ k4 }}</div>
    </div>
</template>
<script setup lang="js">
import { ref, watch, nextTick, onMounted, onUnmounted } from 'vue';
const fileInput = ref(null);
const uploadAreaRef = ref(null);
const previewUrl = ref('');
const canvasRef = ref(null);
const imageLoaded = ref(false);
let gl;
let ext;
let vao;
let imgOriginalSize = { width: 0, height: 0 };
let texture;
let program;
let positionBuffer;
let uniforms = {}; // 存储 uniform 位置
let a_position_loc;
let lastBoundTexture;
let viewportOffset = { x: 0, y: 0 };
let zoomLevel = 1.0;
let initialViewport = { x: 0, y: 0, w: 0, h: 0 };
const MIN_ZOOM = 0.1;
const MAX_ZOOM = 10.0;
const fx = ref(0);
const fy = ref(0);
const cx = ref(1);
const cy = ref(1);
const k1 = ref(0);
const k2 = ref(0);
const k3 = ref(0);
const k4 = ref(0);
const uniformBuffers = {
    u_imageSize: new Float32Array(2),
    u_distort: new Float32Array(4),
    u_intrinsic: new Float32Array(4),
};
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
        float scale = theta_d / max(r, 0.0001);
        float x_dist = x * scale;
        float y_dist = y * scale;
        float u_src = fx * x_dist + cx;
        float v_src = fy * y_dist + cy;
        vec2 src_tex = vec2(u_src / u_imageSize.x, v_src / u_imageSize.y);
        if (src_tex.x < 0.0 || src_tex.x > 1.0 || src_tex.y < 0.0 || src_tex.y > 1.0) {
            gl_FragColor = vec4(0.0);
            return;
        }
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
    gl =
        canvas.getContext("webgl") ||
        canvas.getContext("experimental-webgl");
    if (!gl) {
        console.error("WebGL not supported");
        return;
    }
    ext = gl.getExtension('OES_vertex_array_object');
    if (!ext) {
        console.warn('当前浏览器不支持 VAO 扩展！');
        return;
    }
    const vertexShader = createShader(gl, gl.VERTEX_SHADER, vsSource);
    const fragmentShader = createShader(gl, gl.FRAGMENT_SHADER, fsSource);
    if (!vertexShader || !fragmentShader) return;
    program = gl.createProgram();
    gl.attachShader(program, vertexShader);
    gl.attachShader(program, fragmentShader);
    gl.linkProgram(program);
    if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
        console.error("Program link error:", gl.getProgramInfoLog(program));
        return;
    }
    a_position_loc = gl.getAttribLocation(program, "a_position");
    uniforms = {
        u_texture: gl.getUniformLocation(program, 'u_texture'),
        u_imageSize: gl.getUniformLocation(program, 'u_imageSize'),
        u_distort: gl.getUniformLocation(program, 'u_distort'),
        u_intrinsic: gl.getUniformLocation(program, 'u_intrinsic'),
    };
    positionBuffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
    const positions = new Float32Array([-1, 1, -1, -1, 1, 1, 1, -1]);
    gl.bufferData(gl.ARRAY_BUFFER, positions, gl.STATIC_DRAW);
    vao = ext.createVertexArrayOES();
    ext.bindVertexArrayOES(vao);
    gl.enableVertexAttribArray(a_position_loc);
    gl.vertexAttribPointer(a_position_loc, 2, gl.FLOAT, false, 0, 0);
    ext.bindVertexArrayOES(null);
    texture = gl.createTexture();
    gl.bindTexture(gl.TEXTURE_2D, texture);
    lastBoundTexture = texture;
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
    gl.clearColor(0.0, 0.0, 0.0, 0.0);
}
const renderFrame = () => {
    if (!gl || !ext || !vao || !imageLoaded.value) return;
    gl.useProgram(program);
    uniformBuffers.u_imageSize[0] = imgOriginalSize.width;
    uniformBuffers.u_imageSize[1] = imgOriginalSize.height;
    gl.uniform2fv(uniforms.u_imageSize, uniformBuffers.u_imageSize);
    uniformBuffers.u_distort[0] = k1.value;
    uniformBuffers.u_distort[1] = k2.value;
    uniformBuffers.u_distort[2] = k3.value;
    uniformBuffers.u_distort[3] = k4.value;
    gl.uniform4fv(uniforms.u_distort, uniformBuffers.u_distort);
    uniformBuffers.u_intrinsic[0] = fx.value;
    uniformBuffers.u_intrinsic[1] = fy.value;
    uniformBuffers.u_intrinsic[2] = cx.value;
    uniformBuffers.u_intrinsic[3] = cy.value;
    gl.uniform4fv(uniforms.u_intrinsic, uniformBuffers.u_intrinsic);
    gl.activeTexture(gl.TEXTURE0);
    if (lastBoundTexture !== texture) {
        gl.bindTexture(gl.TEXTURE_2D, texture);
        lastBoundTexture = texture;
    }
    gl.uniform1i(uniforms.u_texture, 0);
    ext.bindVertexArrayOES(vao);
    gl.clear(gl.COLOR_BUFFER_BIT);
    gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);
    ext.bindVertexArrayOES(null);
}
const drawImageToCanvas = async () => {
    await nextTick();
    const canvas = canvasRef.value;
    const uploadArea = uploadAreaRef.value;
    if (!canvas || !uploadArea || !previewUrl.value || !gl) return;
    const boxW = uploadArea.clientWidth;
    const boxH = uploadArea.clientHeight;
    canvas.width = boxW;
    canvas.height = boxH;
    const imgW = imgOriginalSize.width;
    const imgH = imgOriginalSize.height;
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
    initialViewport = { x: Math.round(x), y: Math.round(y), w: Math.round(finalW), h: Math.round(finalH) };
    zoomLevel = 1.0;
    viewportOffset = { x: 0, y: 0 };
    fx.value = imgW;
    fy.value = imgH;
    cx.value = imgW / 2;
    cy.value = imgH / 2;
    gl.viewport(initialViewport.x, initialViewport.y, initialViewport.w, initialViewport.h);
    const img = new Image();
    img.onload = () => {
        gl.bindTexture(gl.TEXTURE_2D, texture);
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
        imgOriginalSize.width = img.width;
        imgOriginalSize.height = img.height;
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
const handleWheel = (event) => {
    if (!imageLoaded.value) return;
    event.preventDefault();
    const zoomIntensity = 0.1;
    const wheel = event.deltaY < 0 ? 1 : -1;
    const zoomFactor = Math.exp(wheel * zoomIntensity);
    const newZoom = zoomLevel * zoomFactor;
    zoomLevel = newZoom;
    if (newZoom < MIN_ZOOM || newZoom > MAX_ZOOM) return;
    const canvas = canvasRef.value;
    const rect = canvas.getBoundingClientRect();
    const mouseX = event.clientX - rect.left;
    const mouseY = event.clientY - rect.top;
    const currentCenterX = initialViewport.x + initialViewport.w / 2 + viewportOffset.x;
    const currentCenterY = initialViewport.y + initialViewport.h / 2 + viewportOffset.y;
    const newCenterX = mouseX + (currentCenterX - mouseX) * zoomFactor;
    const newCenterY = mouseY + (currentCenterY - mouseY) * zoomFactor;
    viewportOffset = {
        x: newCenterX - (initialViewport.x + initialViewport.w / 2),
        y: newCenterY - (initialViewport.y + initialViewport.h / 2)
    };
    const newW = initialViewport.w * zoomLevel;
    const newH = initialViewport.h * zoomLevel;
    const newX = initialViewport.x + viewportOffset.x - (newW - initialViewport.w) / 2;
    const newY = initialViewport.y + viewportOffset.y - (newH - initialViewport.h) / 2;
    gl.viewport(Math.round(newX), Math.round(newY), Math.round(newW), Math.round(newH));
    renderFrame();
};
onMounted(async () => {
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