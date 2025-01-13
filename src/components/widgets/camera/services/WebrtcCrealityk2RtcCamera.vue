<template>
  <video
    ref="streamingElement"
    autoplay
    playsinline
    muted
    :style="cameraStyle"
    :crossorigin="crossorigin"
    @play="updateStatus('connected')"
    @error="updateStatus('error')"
  />
</template>

<script lang="ts">
import { Component, Ref, Mixins } from 'vue-property-decorator'
import consola from 'consola'
import CameraMixin from '@/mixins/camera'

type RTCConfigurationWithSdpSemantics = RTCConfiguration & {
  sdpSemantics: 'unified-plan'
}

@Component({})
export default class WebrtcCrealityk2RtcCamera extends Mixins(CameraMixin) {
  @Ref('streamingElement')
  readonly cameraVideo!: HTMLVideoElement

  pc: RTCPeerConnection | null = null

  // adapted from https://github.com/ayufan/camera-streamer/blob/4203f89df1596cc349b0260f26bf24a3c446a56b/html/webrtc.html

  async startPlayback () {
    const url = this.buildAbsoluteUrl(this.camera.stream_url || '')

    this.pc?.close()

    const iceServers = [
      { urls: 'stun:stun.l.google.com:19302' }
    ]

    try {
      //const rtcSessionDescriptionInit = await response.json() as RTCSessionDescriptionInit

      //this.remoteId = ('id' in rtcSessionDescriptionInit && typeof rtcSessionDescriptionInit.id === 'string')
      //  ? rtcSessionDescriptionInit.id
      //  : null

      const config: RTCConfigurationWithSdpSemantics = {
        sdpSemantics: 'unified-plan'
      }

      config.iceServers = iceServers

      const pc = this.pc = new RTCPeerConnection(config)

      //pc.onicecandidate = event => {
      //  if(event.candidate === null) sendOfferToCall(pc.localDescription.sdp)
      //};
      pc.onicecandidate = async event => {
        if(event.candidate === null) {
          const response = await fetch(`${url}call/webrtc_local`, {
            body: btoa(JSON.stringify({
              type: "offer",
              sdp: pc.localDescription?.sdp
            })),
            headers: {
              'Content-Type': 'plain/text'
            },
            method: 'POST'
          })


          const res = await response.text()

          const jsonRes =  JSON.parse(atob(res))
          if(jsonRes.type == 'answer') {
            pc.setRemoteDescription(new RTCSessionDescription(jsonRes));
          }
        }
      }

      pc.addTransceiver('video', {
        direction: 'recvonly'
      })

      pc.ontrack = (event: RTCTrackEvent) => {
        if (event.track.kind === 'video') {
          this.cameraVideo.srcObject = event.streams[0]
        }
      }
      await pc.createOffer().then(d => pc.setLocalDescription(d));
      console.log("Set Creality K2 Camera")

    } catch (e) {
      consola.error('[WebrtcCamerastreamerCamera] startPlayback', e)
    }
  }

  stopPlayback () {
    this.pc?.close()
    this.pc = null
    this.cameraVideo.src = ''
    this.cameraVideo.srcObject = null
  }
}
</script>