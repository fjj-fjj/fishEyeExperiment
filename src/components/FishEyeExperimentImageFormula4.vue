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
        <span class="demonstration">fy</span>
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
        <span class="demonstration">p1</span>
        <el-slider v-model="p1" :min="-1" :max="1" :step="0.00001" show-input />
        <div class="tip">{{ p1 }}</div>
    </div>
    <div class="slider-block">
        <span class="demonstration">p2</span>
        <el-slider v-model="p2" :min="-1" :max="1" :step="0.00001" show-input />
        <div class="tip">{{ p2 }}</div>
    </div>
</template>
<script setup lang="js">
import { ref, watch, nextTick, onMounted, onUnmounted } from 'vue';
const fileInput = ref(null);
const uploadAreaRef = ref(null);
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
let viewportOffset = { x: 0, y: 0 };
let zoomLevel = 1.0;
let isDragging = false;
let lastMouseX = 0;
let lastMouseY = 0;
let previewUrl;
const MIN_ZOOM = 1.0;
const MAX_ZOOM = 10.0;
const fx = ref(0);
const fy = ref(0);
const cx = ref(1);
const cy = ref(1);
const k1 = ref(0);
const k2 = ref(0);
const k3 = ref(0);
const p1 = ref(0);
const p2 = ref(0);
const uniformBuffers = {
    u_imageSize: new Float32Array(2),
    u_distort_radial: new Float32Array(3),
    u_distort_tan: new Float32Array(2),
    u_intrinsic: new Float32Array(4),
};
const vsSource = `
      attribute vec2 a_position;
      uniform float u_Zoom;
      uniform vec2 u_PanOffset;
      varying vec2 v_texCoord;
      void main() {
        v_texCoord = vec2((a_position.x + 1.0) / 2.0, (1.0 - a_position.y) / 2.0);
        gl_Position = vec4(a_position * u_Zoom + u_PanOffset, 0.0, 1.0);
      }
    `;
//布朗-康拉迪模型
const fsSource = `
      precision highp float;
      uniform sampler2D u_texture;
      uniform vec2 u_imageSize;
      uniform vec3 u_distort_radial; // k1,k2,k3
      uniform vec2 u_distort_tan;//p1,p2
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
        float r2 = x * x + y * y;
        float r4 = r2 * r2;
        float r6 = r4 * r2;
        float k1 = u_distort_radial.x;
        float k2 = u_distort_radial.y;
        float k3 = u_distort_radial.z;
        float p1 = u_distort_tan.x;
        float p2 = u_distort_tan.y;
        float radial_factor = 1.0 + k1 * r2 + k2 * r4 + k3 * r6;
        float x_rad = x * radial_factor;
        float y_rad = y * radial_factor;
        float x_tan = 2.0 * p1 * x * y + p2 * (r2 + 2.0 * x * x);
        float y_tan = p1 * (r2 + 2.0 * y * y) + 2.0 * p2 * x * y;
        float x_dist = x_rad + x_tan;
        float y_dist = y_rad + y_tan;
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
    gl.clearColor(0.0, 0.0, 0.0, 0.0);
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
        u_Zoom: gl.getUniformLocation(program, 'u_Zoom'),
        u_PanOffset: gl.getUniformLocation(program, 'u_PanOffset'),
        u_texture: gl.getUniformLocation(program, 'u_texture'),
        u_imageSize: gl.getUniformLocation(program, 'u_imageSize'),
        u_distort_radial: gl.getUniformLocation(program, 'u_distort_radial'),
        u_distort_tan: gl.getUniformLocation(program, 'u_distort_tan'),
        u_intrinsic: gl.getUniformLocation(program, 'u_intrinsic'),
    };
    positionBuffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
    const positions = new Float32Array([-1, 1, -1, -1, 1, 1, 1, -1]);
    gl.bufferData(gl.ARRAY_BUFFER, positions, gl.STATIC_DRAW);
    vao = ext.createVertexArrayOES();
    ext.bindVertexArrayOES(vao);
    gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
    gl.enableVertexAttribArray(a_position_loc);
    gl.vertexAttribPointer(a_position_loc, 2, gl.FLOAT, false, 0, 0);
    ext.bindVertexArrayOES(null);
    texture = gl.createTexture();
    gl.activeTexture(gl.TEXTURE0);
    gl.bindTexture(gl.TEXTURE_2D, texture);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
    gl.useProgram(program);
    gl.uniform1i(uniforms.u_texture, 0);
}
const renderFrame = () => {
    if (!gl || !ext || !vao || !imageLoaded.value) return;
    gl.useProgram(program);
    uniformBuffers.u_imageSize[0] = imgOriginalSize.width;
    uniformBuffers.u_imageSize[1] = imgOriginalSize.height;
    gl.uniform2fv(uniforms.u_imageSize, uniformBuffers.u_imageSize);
    uniformBuffers.u_distort_radial[0] = k1.value;
    uniformBuffers.u_distort_radial[1] = k2.value;
    uniformBuffers.u_distort_radial[2] = k3.value;
    uniformBuffers.u_distort_tan[0] = p1.value;
    uniformBuffers.u_distort_tan[1] = p2.value;
    gl.uniform3fv(uniforms.u_distort_radial, uniformBuffers.u_distort_radial);
    gl.uniform2fv(uniforms.u_distort_tan, uniformBuffers.u_distort_tan);
    uniformBuffers.u_intrinsic[0] = fx.value;
    uniformBuffers.u_intrinsic[1] = fy.value;
    uniformBuffers.u_intrinsic[2] = cx.value;
    uniformBuffers.u_intrinsic[3] = cy.value;
    gl.uniform4fv(uniforms.u_intrinsic, uniformBuffers.u_intrinsic);
    gl.uniform1f(uniforms.u_Zoom, zoomLevel);
    gl.uniform2f(uniforms.u_PanOffset, viewportOffset.x, viewportOffset.y);
    ext.bindVertexArrayOES(vao);
    gl.clear(gl.COLOR_BUFFER_BIT);
    gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);
    ext.bindVertexArrayOES(null);
}
const drawImageToCanvas = async () => {
    await nextTick();
    const canvas = canvasRef.value;
    const uploadArea = uploadAreaRef.value;
    if (!canvas || !uploadArea || !previewUrl || !gl) return;
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
    zoomLevel = 1.0;
    viewportOffset = { x: 0, y: 0 };
    fx.value = imgW;
    fy.value = imgH;
    cx.value = imgW / 2;
    cy.value = imgH / 2;
    gl.viewport(Math.round(x), Math.round(y), Math.round(finalW), Math.round(finalH));
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
    img.src = previewUrl;
};
const processFile = async (file) => {
    if (!file || !file.type.startsWith('image/')) return;
    if (previewUrl) {
        URL.revokeObjectURL(previewUrl);
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
    previewUrl = objectUrl;
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
const applyBoundaryConstraints = () => {
    const distToViewportLeft = viewportOffset.x + 1.0;
    if (distToViewportLeft > zoomLevel) {
        viewportOffset.x = -1.0 + zoomLevel;
    }
    const distToViewportRight = 1.0 - viewportOffset.x;
    if (distToViewportRight > zoomLevel) {
        viewportOffset.x = 1.0 - zoomLevel;
    }
    const distToViewportTop = 1.0 - viewportOffset.y;
    if (distToViewportTop > zoomLevel) {
        viewportOffset.y = 1.0 - zoomLevel;
    }
    const distToViewportBottom = viewportOffset.y + 1.0;
    if (distToViewportBottom > zoomLevel) {
        viewportOffset.y = -1.0 + zoomLevel;
    }
}
const handleWheel = (event) => {
    if (!imageLoaded.value) return;
    event.preventDefault();
    const zoomIntensity = 0.1;
    const wheel = event.deltaY < 0 ? 1 : -1;
    const zoomFactor = Math.exp(wheel * zoomIntensity);
    let newZoom = zoomLevel * zoomFactor;
    if (newZoom < MIN_ZOOM) newZoom = MIN_ZOOM;
    if (newZoom > MAX_ZOOM) newZoom = MAX_ZOOM;
    const canvas = canvasRef.value;
    const rect = canvas.getBoundingClientRect();
    const mouseXNDC = ((event.clientX - rect.left) / canvas.width) * 2.0 - 1.0;
    const mouseYNDC = -(((event.clientY - rect.top) / canvas.height) * 2.0 - 1.0);
    viewportOffset.x = mouseXNDC - (mouseXNDC - viewportOffset.x) * zoomFactor;
    viewportOffset.y = mouseYNDC - (mouseYNDC - viewportOffset.y) * zoomFactor;
    zoomLevel = newZoom;
    applyBoundaryConstraints();
    renderFrame();
};
const handleMouseDown = (event) => {
    if (!imageLoaded.value) return;
    isDragging = true;
    lastMouseX = event.clientX;
    lastMouseY = event.clientY;
    canvasRef.value.style.cursor = 'grabbing';
};
const handleMouseMove = (event) => {
    if (event.buttons === 0 && isDragging) {
        handleMouseUp();
        return;
    }
    if (!isDragging || !imageLoaded.value) return;
    const canvas = canvasRef.value;
    const rect = canvas.getBoundingClientRect();
    const deltaX = event.clientX - lastMouseX;
    const deltaY = event.clientY - lastMouseY;
    const ndcDeltaX = (deltaX / rect.width) * 2.0;
    const ndcDeltaY = (deltaY / rect.height) * 2.0;
    viewportOffset.x += ndcDeltaX;
    viewportOffset.y -= ndcDeltaY;
    applyBoundaryConstraints();
    lastMouseX = event.clientX;
    lastMouseY = event.clientY;
    renderFrame();
};
const handleMouseUp = () => {
    if (isDragging) {
        isDragging = false;
        if (canvasRef.value) {
            canvasRef.value.style.cursor = 'grab';
        }
    }
};
onMounted(async () => {
    initWebgl();
    const canvas = canvasRef.value;
    canvas.addEventListener('mousedown', handleMouseDown);
    canvas.addEventListener('mouseleave', handleMouseUp);
    window.addEventListener('mousemove', handleMouseMove);
    window.addEventListener('mouseup', handleMouseUp);
})
onUnmounted(() => {
    if (previewUrl) {
        URL.revokeObjectURL(previewUrl);
    }
    const canvas = canvasRef.value;
    if (canvas) {
        canvas.removeEventListener('mousedown', handleMouseDown);
        canvas.removeEventListener('mouseleave', handleMouseUp);
    }
    window.removeEventListener('mousemove', handleMouseMove);
    window.removeEventListener('mouseup', handleMouseUp);
})
watch(imageLoaded, (newVal) => {
    if (newVal) drawImageToCanvas();
})
watch([fx, fy, cx, cy, k1, k2, k3, p1, p2], () => {
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

.slider-block .demonstration+.el-slider {
    flex: 0 0 80%;
}

.slider-block .tip {
    flex: 1;
}
</style>