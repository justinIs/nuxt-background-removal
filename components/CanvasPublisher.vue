<template>
<div class="publisher-component">
    <div v-show="false">
        <video
            ref="localVideo"
            :width="videoWidth"
            :height="videoHeight"
            playsinline
            autoplay
            muted
        />
        <canvas ref="outputCanvas" :width="videoWidth" :height="videoHeight" />
    </div>
    <div ref="publisherWrapper" class="publisher-wrapper" />
</div>
</template>

<script lang="ts">
import { Component, Prop, Ref, Vue } from 'nuxt-property-decorator'
import OT from '@opentok/client'
import * as bodyPix from '@tensorflow-models/body-pix'
import { BodyPix } from '@tensorflow-models/body-pix'
import '@tensorflow/tfjs'
import * as workerTimers from 'worker-timers'

@Component
export default class OtPublisher extends Vue {
    @Prop({ required: true }) session!: OT.Session

    @Ref('publisherWrapper') publisherWrapper!: HTMLDivElement
    @Ref('localVideo') localVideo!: HTMLVideoElement
    @Ref('outputCanvas') outputCanvas!: HTMLCanvasElement

    publisher: OT.Publisher | null = null

    quality = 'high'

    backgroundImage: HTMLImageElement | undefined

    processCanvas = false

    get videoWidth() {
        return this.quality === 'high' ? 640 : 320
    }

    get videoHeight() {
        return this.quality === 'high' ? 480 : 240
    }

    get canvasVideoTrack() {
        return (this.outputCanvas as any).captureStream().getVideoTracks()[0]
    }

    created() {
        this.backgroundImage = new Image()
        this.backgroundImage.src = require('~/assets/icons/mars_background.jpg')
    }

    async mounted() {
        await this.initCanvasPublisher()

        this.publisher = OT.initPublisher(this.publisherWrapper, {
            insertMode: 'append',
            width: '100%',
            height: '100%',
            videoSource: this.canvasVideoTrack
        })
        this.session.publish(this.publisher)
    }

    stop() {
        this.publisher?.destroy()
        this.processCanvas = false
        ;(this.localVideo.srcObject as MediaStream).getTracks().forEach(track => track.stop())
    }

    private async initCanvasPublisher() {
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

        const playPromise = new Promise<void>((resolve) => {
            this.localVideo.addEventListener('play', () => {
                console.log('playing local video now')
                resolve()
            })
        })
        const loadeddataPromise = new Promise<void>((resolve) => {
            this.localVideo.addEventListener('loadeddata', () => {
                console.log('local video loadeddata')
                resolve()
            })
        })

        this.localVideo.play()
        await Promise.all([playPromise, loadeddataPromise])
        this.startCanvasRendering()
    }

    private async startCanvasRendering() {
        const canvasContext = this.outputCanvas.getContext('2d')
        if (!canvasContext) {
            throw new Error('Could not get canvas context to start canvas rendering')
        }
        const net = await bodyPix.load({
          architecture: 'MobileNetV1',
          multiplier: 1,
          quantBytes: 4,
          outputStride: 8
        })
        const canvasClone = (canvasContext.canvas.cloneNode(false) as HTMLCanvasElement).getContext('2d')
        if (!canvasClone) {
            throw new Error('Unable to clone canvas context')
        }
        this.processCanvas = true
        this.renderCanvas(canvasContext, canvasClone, net)
    }

    private async renderCanvas(context: CanvasRenderingContext2D, cloneContext: CanvasRenderingContext2D, net: BodyPix) {
        // Segment input video to identify background pixels vs body pixels
        const segmentation = await net.segmentPerson(this.localVideo, {
            flipHorizontal: true,
            internalResolution: 'high',
            segmentationThreshold: 0.4
        })

        // Draw the video to a canvas and use segmentation to remove background
        cloneContext.drawImage(this.localVideo, 0, 0, this.videoWidth, this.videoHeight)
        const imgData = cloneContext.getImageData(0, 0, this.videoWidth, this.videoHeight)
        const pixels = imgData.data
        for (let i = 0; i < pixels.length; i += 4) {
            if (segmentation.data[i / 4] === 0) {
                pixels[i + 3] = 0
            }
        }
        cloneContext.putImageData(imgData, 0, 0)
        if (!this.backgroundImage) {
            throw new Error('Unable to get background image')
        }
        context.drawImage(this.backgroundImage, 0, 0, this.videoWidth, this.videoHeight)
        context.drawImage(cloneContext.canvas, 0, 0, this.videoWidth, this.videoHeight)

        if (this.processCanvas) {
            workerTimers.setTimeout(this.renderCanvas.bind(this, context, cloneContext, net), 30)
        }
    }
}
</script>

<style lang="scss">
.publisher-component {
    width: 100%;
    height: 100%;

    .publisher-wrapper {
        width: 100%;
        height: 100%;
    }
}
</style>
