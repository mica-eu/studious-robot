<template>
  <div id="app">
    <video width="500px" ref="video" controls muted autoplay></video>
    <br />
    Chunks: {{ recordedChunks.length }} Size: {{ numRecordedChunks }} State:
    {{ recorder ? recorder.state : null }}
    <br />
    <hr />
    <select v-model="sourceId" @change="getUserMedia(sourceId)">
      <option value="" selected>Select one source</option>
      <option v-for="source in sources" :key="source.id" :value="source.id">{{
        source.name
      }}</option>
    </select>
    <br />
    <hr />
    <button @click="startRecord">Start Record</button>
    <button @click="stopRecord">Stop Record</button>
    <button @click="() => playRecord()">Play</button>
    <button @click="() => download()">Download</button>
  </div>
</template>

<script>
// eslint-disable-next-line
import { desktopCapturer } from 'electron';
let recordedChunks = [];
let localStream = null;
let recorder = null;

export default {
  name: 'App',
  data() {
    return {
      sources: [],
      sourceId: null,
      recordedChunks: [],
      numRecordedChunks: 0,
      includeMic: false,
      recordOptions: {
        mimeType: 'video/webm; codecs="av01.2.19H.12.0.000.09.16.09.1, flac"',
      },
    };
  },
  methods: {
    cleanRecord() {
      this.$refs.video.src = null;
      recordedChunks = [];
      this.numRecordedChunks = 0;
    },
    startRecord() {
      this.cleanRecord();
      console.log(this.sourceId);
    },
    stopRecord() {
      console.log(this.numRecordedChunks, recordedChunks);
      recorder.stop();
      localStream.getVideoTracks()[0].stop();
    },
    async getSources() {
      return desktopCapturer.getSources({ types: ['screen', 'window'] });
    },
    getUserMedia(sourceId) {
      if (!sourceId) return;

      console.log('Trying get user media for:', sourceId);

      navigator.mediaDevices
        .getUserMedia({
          audio: {
            mandatory: {
              chromeMediaSource: 'desktop',
            },
          },
          video: {
            mandatory: {
              chromeMediaSource: 'desktop',
              chromeMediaSourceId: sourceId,
            },
          },
        })
        .then(this.getMediaStream)
        .catch(this.getUserMediaError);
    },
    getMediaStream(stream) {
      console.log(stream);
      this.$refs.video.srcObject = stream;
      this.$refs.video.play();
      localStream = stream;

      // eslint-disable-next-line
      stream.onended = () => console.log('Media stream ended.');

      try {
        recorder = new MediaRecorder(stream);
      } catch (err) {
        console.log('Media Recorder Error:', err);
      }

      recorder.ondataavailable = this.recorderData;

      recorder.onstop = () => console.log('recorderOnStop fired');
      recorder.start();
      console.log('Recorder is started.');
    },
    recorderData(event) {
      if (event.data && event.data.size > 0) {
        recordedChunks.push(event.data);
        this.numRecordedChunks += event.data.byteLength;
      }
    },
    getUserMediaError(err) {
      console.log('getUserMedia Error:', err);
    },
    playRecord() {
      const blob = new Blob(recordedChunks, this.recordOptions);
      this.$refs.video.src = URL.createObjectURL(blob);
      this.$refs.video.play();
      console.log('Playing video');
    },
    download() {
      const blob = new Blob(recordedChunks, this.recordOptions);
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      document.body.appendChild(a);
      a.style = 'display: none';
      a.href = url;
      a.download = 'electron-screen-recorder.mp4';
      a.click();
      setTimeout(() => {
        document.body.removeChild(a);
        window.URL.revokeObjectURL(url);
      }, 100);
    },
  },
  mounted() {
    this.getSources()
      .then((sources) => {
        this.sources = sources;
      })
      .catch((err) => console.log(err));
  },
};
</script>

<style lang="scss">
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
