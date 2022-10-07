<script setup lang="ts">
import { onMounted, ref, onUnmounted, computed } from 'vue'

import '@tensorflow/tfjs'
import * as coco from '@tensorflow-models/coco-ssd'

const devices = ref<MediaDeviceInfo[]>([])
const video = ref<HTMLVideoElement>()
const model = ref<Readonly<coco.ObjectDetection>>()
const predictions = ref<coco.DetectedObject[]>([])
const timer = ref<NodeJS.Timer>()
const isPredicting = ref(false)
const isVideoSteaming = ref(false)

onMounted(async function () {
    const allDevices = await navigator.mediaDevices.enumerateDevices();
    devices.value = allDevices.filter(device => device.kind === "videoinput")
    video.value?.addEventListener("play", async () => {
        if (isPredicting.value) return
        model.value = Object.freeze(await coco.load())
        timer.value = setInterval(predict, 1_000)
        isPredicting.value = true
    })
})

onUnmounted(() => {
    if (timer.value)
        clearInterval(timer.value)
})


const setVideoSteam = async (deviceId: string) => {
    const stream = await navigator.mediaDevices.getUserMedia({
        video: { deviceId }
    })
    if (stream && video.value) {
        video.value.srcObject = stream
        isVideoSteaming.value = true
    } else isVideoSteaming.value = false
}

const predict = async () => {
    if (!model.value || !video.value) return
    try {
        predictions.value = await model.value.detect(video.value)
    } catch (err) {
        //  console.error("predict error", err)
    }
}

const predictionsC = computed(() => predictions.value.map(({ bbox: [x, y, w, h], score, ...o }) => ({
    x: x.toFixed(2),
    y: y.toFixed(2),
    w: w.toFixed(2),
    h: h.toFixed(2),
    score: score.toFixed(2),
    ...o
})))

</script>

<template>
    <h3>Device selection</h3>
    <ul>
        <li v-for="device in devices" @click="setVideoSteam(device.deviceId)">
            {{device.label || device.deviceId}}
        </li>
    </ul>

    <div :style="{position: 'relative'}">
        <div :style="{position: 'absolute'}">
            <div v-for="prediction in predictionsC" class="prediction" :style="{top: `${prediction.y}px`, left: `${prediction.x}px`, width:
            `${prediction.w}px`, height: `${prediction.h}px`}">
                <span>{{prediction.class}} <i>({{prediction.score}}%)</i></span>
            </div>
            <video ref="video" class="video" autoplay muted></video>
        </div>
    </div>

    <h2 v-if="isPredicting && predictions.length">{{predictions.length}} prediction</h2>
    <h2 v-else-if="isPredicting">Waiting for prediction...</h2>
    <h2 v-else-if="isVideoSteaming">Waiting for prediction model initialization...</h2>
    <h2 v-else>Waiting for device selection...</h2>
</template>

<style scoped>
:root,
* {
    color: #888;
}

h2 {
    position: fixed;
    bottom: 1rem;
}

li {
    cursor: pointer;
}

.video {
    border: 1px solid plum;
    border-radius: 0.1rem;
    width: 100%;
}

.prediction {
    align-items: center;
    border: 1px dashed salmon;
    color: salmon;
    display: flex;
    justify-content: center;
    position: absolute;
}

.prediction span {
    background-color: rgba(0, 0, 0, 0.5);
    border-radius: 0.1rem;
    display: inline-block;
    padding: 0.1rem;
}

.prediction span:first-letter {
    text-transform: capitalize;
}
</style>
