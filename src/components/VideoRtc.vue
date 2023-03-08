<script lang="ts" setup>
import axios from 'axios'

const props = defineProps({
  suuid: {
    type: String,
    required: true,
  },
})

const stream = new MediaStream()

const videoRef = ref<HTMLVideoElement>()
const logList = ref<string[]>([])
const config = {
  iceServers: [{
    urls: ['stun:stun.l.google.com:19302'],
  }],
}

const prefixUrl = 'http://autopump-webrtc:8083'
const pc = new RTCPeerConnection(config)

const getRemoteSdp = async () => {
  if (pc && pc.localDescription) {
    const params = new URLSearchParams()
    params.append('suuid', props.suuid)
    params.append('data', btoa(pc.localDescription.sdp))
    const res = await axios.post(`${prefixUrl}/stream/receiver/${props.suuid}`, params)
    try {
      pc.setRemoteDescription(new RTCSessionDescription({
        type: 'answer',
        sdp: atob(res.data),
      }))
    }
    catch (e) {
      console.warn(e)
    }
  }
}

const getCodecInfo = async () => {
  try {
    const res = await axios.get(`${prefixUrl}/stream/codec/${props.suuid}`)
    res.data && res.data.forEach((value: any) => {
      pc.addTransceiver(value.Type, {
        direction: 'recvonly',
        // direction: 'sendrecv',
      })
    })
  }
  catch (e) {
    console.warn(e)
  }
}
const log = (str: string) => {
  logList.value.push(str)
}
onMounted(() => {
  getCodecInfo()
  pc.onnegotiationneeded = async () => {
    const offer = await pc.createOffer()
    await pc.setLocalDescription(offer)
    getRemoteSdp()
  }

  pc.ontrack = (event) => {
    stream.addTrack(event.track)
    if (videoRef.value) {
      videoRef.value.srcObject = stream
      log(`${event.streams.length} track is delivered`)
    }
  }

  pc.oniceconnectionstatechange = (e) => {
    console.log(e, pc.iceConnectionState)
    log(pc.iceConnectionState)
  }
})
</script>

<template>
  <div>
    <h2>{{ suuid }}</h2>
    <video ref="videoRef" class="w-full aspect-video" autoplay muted controls />
    <div>
      <p v-for="(log, idx) in logList" :key="idx">
        {{ log }}
      </p>
    </div>
  </div>
</template>

<style lang="css" scoped>
video::-webkit-media-controls {
  /* display: none !important; */
}

video::-webkit-media-controls-enclosure {
  display: none !important;
}

video::-webkit-media-controls-fullscreen-button {
  display: inline-block !important;
}
</style>
