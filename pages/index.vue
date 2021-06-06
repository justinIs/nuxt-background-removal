<template>
<div class="container">
    <div class="video-container">
        <CanvasPublisher v-if="session" class="video" :session="session" />
    </div>
    <div class="controls-container">
        <div>Controls</div>
        <div>
            <button @click="start">
                Start
            </button>
        </div>
    </div>
</div>
</template>

<script lang="ts">
import { Component, Vue } from 'nuxt-property-decorator'
import OT from '@opentok/client'
import CanvasPublisher from '~/components/CanvasPublisher.vue'

@Component({
    components: { CanvasPublisher }
})
export default class Index extends Vue {
    session: OT.Session | null = null

    start() {
        const apiKey = process.env.API_KEY
        const sessionId = process.env.SESSION_ID
        const token = process.env.OT_TOKEN
        if (!apiKey || !sessionId || !token) {
            throw new Error('Could not get env tokens')
        }

        const session = OT.initSession(apiKey, sessionId)
        session.connect(token, (err) => {
            if (err) {
                alert('Could not connect session: ' + err)
            } else {
                this.session = session
            }
        })
    }
}
</script>

<style lang="scss">
.container {
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

        .video, canvas {
            width: 50%;
        }
    }

    .controls-container {
        margin-top: 20px;
    }
}
</style>
