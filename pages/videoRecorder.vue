<template>
<div class="page-container">
    <div class="video-container">
        <div v-show="false" class="_refs">
            <video
                ref="localVideo"
                :width="videoWidth"
                :height="videoHeight"
                playsinline
                autoplay
                muted
            />
        </div>
        <canvas ref="outputCanvas" :width="videoWidth" :height="videoHeight" />
        <video ref="recordingVideo" playsinline autoplay muted controls />
    </div>
    <div class="controls-container">
        <div>Controls</div>
        <div>
            Media Stream Status: <strong>{{ mediaStreamStatus }}</strong>
        </div>
        <div>
            Recording Status: <strong>{{ recordingStatus }}</strong>
        </div>
        <div>
            <button @click="toggleMediaStream">
                {{ toggleMediaStreamButtonText }}
            </button>
            <button @click="startRecording">
                Record
            </button>
            <a ref="downloadButton">Download</a>
        </div>
    </div>
</div>
</template>

<script lang="ts">
import { Component, Ref, Vue } from 'nuxt-property-decorator'
import * as workerTimers from 'worker-timers'

enum Status {
    initial = 'initial',
    started = 'started',
    stopped = 'stopped'
}

@Component
export default class VideoRecorder extends Vue {
    @Ref('localVideo') localVideo!: HTMLVideoElement
    @Ref('recordingVideo') recordingVideo!: HTMLVideoElement
    @Ref('outputCanvas') outputCanvas!: HTMLCanvasElement
    @Ref('downloadButton') downloadButton!: HTMLAnchorElement

    mediaStreamStatus: Status = Status.initial
    recordingStatus = Status.initial

    quality = 'high'

    get toggleMediaStreamButtonText() {
        return this.mediaStreamStatus === 'started' ? 'Stop Media Stream' : 'Start Media Stream'
    }

    get videoWidth() {
        return this.quality === 'high' ? 640 : 320
    }

    get videoHeight() {
        return this.quality === 'high' ? 480 : 240
    }

    async mounted() {
        await this.initMediaStream()
    }

    private async initMediaStream() {
        const devices = await navigator.mediaDevices.enumerateDevices()
        const videoDevice = devices.find(device => device.kind === 'videoinput')
        if (!videoDevice) {
            throw new Error('Could not get video device')
        }

        console.log('initMediaStream')
        this.localVideo.srcObject = await navigator.mediaDevices.getUserMedia({
            video: {
                deviceId: videoDevice.deviceId,
                frameRate: 30,
                width: { ideal: this.videoWidth },
                height: { ideal: this.videoHeight }
            }
        })

        this.localVideo.addEventListener('play', () => {
            console.log('playing local video now')
            this.renderCanvas()
        })
        this.localVideo.play()

        this.mediaStreamStatus = Status.started
    }

    private stopMediaStream() {
        const stream = this.localVideo.srcObject as MediaStream
        if (!stream) {
            throw new Error('Could not get local stream to stop')
        }
        stream.getTracks().forEach(track => track.stop())
        this.mediaStreamStatus = Status.stopped
    }

    private renderCanvas() {
        const canvasContext = this.outputCanvas.getContext('2d')
        if (!canvasContext) {
            throw new Error('Could not get output canvas context')
        }

        canvasContext.drawImage(this.localVideo, 0, 0, this.videoWidth, this.videoHeight)
        // window.requestAnimationFrame(this.renderCanvas.bind(this))
        workerTimers.setTimeout(this.renderCanvas.bind(this), (1000 / 30))
    }

    toggleMediaStream() {
        switch (this.mediaStreamStatus) {
            case Status.initial:
            case Status.stopped:
                this.initMediaStream()
                break
            case Status.started:
                this.stopMediaStream()
                break
        }
    }

    async startRecording() {
        this.recordingStatus = Status.started
        // convert recording to gif with:
        // ffmpeg -ss 00:00:00.000 -i /Users/justinwilliams/Downloads/RecordedVideo.webm -pix_fmt rgb24 -r 10 -s 320x240 -t 00:00:10.000 output.gif
        const recorder = new MediaRecorder((this.outputCanvas as any).captureStream())
        const data: Blob[] = []

        recorder.ondataavailable = event => data.push(event.data)
        recorder.start()

        console.log(`${recorder.state} for 30 seconds...`)
        const stopped = new Promise<Event>((resolve, reject) => {
            recorder.onstop = resolve
            recorder.onerror = event => reject((event as any).name)
        })

        const recorded = new Promise<void>(resolve => setTimeout(() => {
            console.log('Stopping recording, state:', recorder.state)
            recorder.stop()
            resolve()
        }, 30000))

        await Promise.all([
            stopped,
            recorded
        ])

        this.recordingStatus = Status.stopped

        const recordedBlob = new Blob(data, { type: 'video/webm' })
        this.recordingVideo.src = URL.createObjectURL(recordedBlob)
        this.downloadButton.href = this.recordingVideo.src
        this.downloadButton.download = 'RecordedVideo.webm'

        console.log('Successfully recorded ' + recordedBlob.size + ' bytes of ' +
            recordedBlob.type + ' media.')
    }
}
</script>

<style lang="scss">
.page-container {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;

    .video-container {
        width: 840px;
        height: 340px;

        padding: 20px;
        border: 2px solid;

        display: flex;

        video, canvas {
            width: 50%;
        }
    }

    .controls-container {
        margin-top: 20px;
    }
}
</style>
