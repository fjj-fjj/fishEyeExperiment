<template>
    <video id="video" crossorigin="anonymous" playsinline ref="videoDom"></video>
    <canvas id="gl-canvas" ref="glCanvasDom"></canvas>
</template>
<script setup lang="js">
import Hls from 'hls.js';
import { onMounted, nextTick, ref, onUnmounted } from 'vue';
const HLS_URL =
    "https://sf1-cdn-tos.huoshanstatic.com/obj/media-fe/xgplayer_doc_video/hls/xgplayer-demo.m3u8";
const videoDom = ref(null);
const glCanvasDom = ref(null);
const texture = ref(null);
const program = ref(null);
const positionBuffer = ref(null);
const vsSource = `
      attribute vec2 a_position;
      varying vec2 v_texCoord;
      void main() {
        gl_Position = vec4(a_position * 2.0 - 1.0, 0, 1); // 映射 [0,1] -> [-1,1]
        v_texCoord = vec2(a_position.x, 1.0 - a_position.y); // 翻转 Y 轴
      }
    `;
const fsSource = `
      precision mediump float;
      uniform sampler2D u_texture;
      varying vec2 v_texCoord;
      void main() {
        vec4 color = texture2D(u_texture, v_texCoord);
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
const resizeCanvas = () => {
    const maxWidth = Math.min(800, window.innerWidth - 40);
    const displayHeight = videoDom.value.videoHeight
        ? videoDom.value.videoHeight * (maxWidth / videoDom.value.videoWidth)
        : 450;
    glCanvasDom.value.style.width = maxWidth + "px";
    glCanvasDom.value.style.height = displayHeight + "px";
    const dpr = window.devicePixelRatio || 1;
    glCanvasDom.value.width = maxWidth * dpr;
    glCanvasDom.value.height = displayHeight * dpr;
    const gl =
        glCanvasDom.value.getContext("webgl") ||
        glCanvasDom.value.getContext("experimental-webgl");
    gl.viewport(0, 0, glCanvasDom.value.width, glCanvasDom.value.height);
}
const render = () => {
    if (videoDom.value.readyState >= 2) {
        const gl =
            glCanvasDom.value.getContext("webgl") ||
            glCanvasDom.value.getContext("experimental-webgl");
        gl.bindTexture(gl.TEXTURE_2D, texture.value);
        gl.texImage2D(
            gl.TEXTURE_2D,
            0,
            gl.RGBA,
            gl.RGBA,
            gl.UNSIGNED_BYTE,
            videoDom.value,
        );
        gl.useProgram(program.value);
        gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer.value);
        const posLoc = gl.getAttribLocation(program.value, "a_position");
        gl.enableVertexAttribArray(posLoc);
        gl.vertexAttribPointer(posLoc, 2, gl.FLOAT, false, 0, 0);
        gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);
    }
    requestAnimationFrame(render);
}
const init = () => {
    const gl =
        glCanvasDom.value.getContext("webgl") ||
        glCanvasDom.value.getContext("experimental-webgl");
    if (!gl) {
        throw new Error("WebGL not supported");
    }
    const vertexShader = createShader(gl, gl.VERTEX_SHADER, vsSource);
    const fragmentShader = createShader(gl, gl.FRAGMENT_SHADER, fsSource);
    program.value = gl.createProgram();
    gl.attachShader(program.value, vertexShader);
    gl.attachShader(program.value, fragmentShader);
    gl.linkProgram(program.value);
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
    if (Hls.isSupported()) {
        const hls = new Hls();
        hls.loadSource(HLS_URL);
        hls.attachMedia(videoDom.value);
        hls.on(Hls.Events.MANIFEST_PARSED, () => {
            videoDom.value.muted = true;
            videoDom.value.play().catch(() => {
                document.body.addEventListener("click", () => videoDom.value.play(), {
                    once: true,
                });
            });
        });
        videoDom.value.addEventListener("loadedmetadata", resizeCanvas);
        window.addEventListener("resize", resizeCanvas);
    }
}
const destroy = () => {
    window.removeEventListener("resize", resizeCanvas);
}
onMounted(() => {
    init();
    render();
})
onUnmounted(() => {
    destroy();
})
</script>