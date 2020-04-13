<template>
  <div id="app">
    <div ref="wrapper" class="wrapper">
      <video ref="video" width="1280" height="720" muted autoplay @play="startDetections"></video>
    </div>
  </div>
</template>

<script>
import * as faceapi from 'face-api.js'

export default {
  name: 'App',
  mounted() {
    navigator.getUserMedia(
      { video: { width: 1280, height: 720 } },
      stream => (this.$refs.video.srcObject = stream),
      error => console.error(error)
    )
    Promise.all([
      faceapi.nets.tinyFaceDetector.loadFromUri('./models'),
      faceapi.nets.faceLandmark68Net.loadFromUri('./models'),
      faceapi.nets.faceRecognitionNet.loadFromUri('./models'),
      faceapi.nets.faceExpressionNet.loadFromUri('./models'),
      faceapi.nets.ageGenderNet.loadFromUri('./models'),
    ])
  },
  methods: {
    startDetections() {
      const canvas = faceapi.createCanvasFromMedia(this.$refs.video)
      this.$refs.wrapper.appendChild(canvas)
      const videoSize = { width: this.$refs.video.width, height: this.$refs.video.height }
      faceapi.matchDimensions(canvas, videoSize)
      setInterval(async () => {
        const detections = await faceapi
          .detectAllFaces(this.$refs.video, new faceapi.TinyFaceDetectorOptions())
          .withFaceLandmarks()
          .withFaceExpressions()
          .withFaceDescriptors()
          .withAgeAndGender()
        // console.log(detections)
        // let key = Object.keys(detections[0].expressions).reduce((key, v) =>
        // detections[0].expressions[v] < detections[0].expressions[key] ? v : key
        // )
        // console.log(key)
        const resizedDetections = faceapi.resizeResults(detections, videoSize)
        // let key = Object.keys(resizedDetections[0].expressions).reduce((key, v) =>
        //   resizedDetections[0].expressions[v] < resizedDetections[0].expressions[key] ? v : key
        // )
        // console.log(key)
        // console.log(resizedDetections[0].expressions)
        canvas.getContext('2d').clearRect(0, 0, canvas.width, canvas.height)
        faceapi.draw.drawDetections(canvas, resizedDetections)
        faceapi.draw.drawFaceLandmarks(canvas, resizedDetections)
        const asd = faceapi.draw.drawFaceExpressions(canvas, resizedDetections, 0.05)
        console.log(asd)
      }, 500)
    },
  },
}
</script>

<style lang="scss">
body {
  margin: 0;
  padding: 0;
  background: #111111;

  #app {
    font-family: Avenir, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    color: #2c3e50;
    height: 100vh;
    width: 100vw;
    display: flex;
    justify-content: center;
    align-items: center;

    .wrapper {
      position: relative;
      canvas {
        position: absolute;
        top: 0;
        left: 0;
      }
    }
  }
}
</style>
