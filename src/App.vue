<template>
  <div id="app">
    <div class="faceExpression">{{ faceExpression }}</div>
    <div ref="wrapper" class="wrapper">
      <video ref="video" width="1280" height="720" muted autoplay @play="startDetections"></video>
      <canvas ref="canvas" width="1280" height="720"></canvas>
    </div>
  </div>
</template>

<script>
import * as faceapi from 'face-api.js'

export default {
  name: 'App',
  data() {
    return {
      ctx: null,
      pointSize: 3,
      faceExpression: '',
    }
  },
  mounted() {
    navigator.getUserMedia(
      { video: { width: 1280, height: 720 } },
      stream => (this.$refs.video.srcObject = stream),
      error => console.error(error)
    )
    Promise.all([
      faceapi.loadTinyFaceDetectorModel('./models'),
      faceapi.loadFaceLandmarkModel('./models'),
      faceapi.loadFaceRecognitionModel('./models'),
      faceapi.loadFaceExpressionModel('./models'),
      faceapi.loadAgeGenderModel('./models'),
    ])
    this.ctx = this.$refs.canvas.getContext('2d')
  },
  methods: {
    startDetections() {
      // const canvas = faceapi.createCanvasFromMedia(this.$refs.video)
      // this.$refs.wrapper.appendChild(canvas)
      const videoSize = { width: 1280, height: 720 }
      faceapi.matchDimensions(this.$refs.canvas, videoSize)
      setInterval(async () => {
        const detections = await faceapi
          .detectAllFaces(this.$refs.video, new faceapi.TinyFaceDetectorOptions())
          .withFaceLandmarks()
          .withFaceExpressions()
          .withFaceDescriptors()
          .withAgeAndGender()
        console.log(detections)
        // let key = Object.keys(detections[0].expressions).reduce((key, v) =>
        // detections[0].expressions[v] < detections[0].expressions[key] ? v : key
        // )
        // console.log(key)
        const resizedDetections = faceapi.resizeResults(detections, videoSize)
        // let key = Object.keys(resizedDetections[0].expressions).reduce((key, v) =>
        //   resizedDetections[0].expressions[v] < resizedDetections[0].expressions[key] ? v : key
        // )
        if (resizedDetections[0]) {
          // console.log(resizedDetections[0].expressions)
          const faceExpressionsProbability = Object.values(
            resizedDetections[0].expressions
          ).map(item => Number(item).toFixed(5))
          const maxFaceExpressionsProbability = Math.max.apply(Math, faceExpressionsProbability)

          const faceExpression = Object.keys(resizedDetections[0].expressions).filter(function(
            key
          ) {
            return (
              Number(resizedDetections[0].expressions[key]).toFixed(5) ==
              maxFaceExpressionsProbability
            )
          })[0]
          if (this.faceExpression != faceExpression) this.faceExpression = faceExpression
          // console.log(faceExpression)
        }
        // faceapi.matchDimensions(canvas, videoSize)
        const detection = await faceapi
          .detectSingleFace(this.$refs.video, new faceapi.TinyFaceDetectorOptions())
          .withFaceLandmarks()
        const landmarks = detection.landmarks
        // const landmarksResized = faceapi.resizeResults(landmarks, videoSize)

        // const jawOutline = landmarksResized.getJawOutline()
        // const nose = landmarks.getNose()
        // const mouth = landmarksResized.getMouth()
        // const leftEye = landmarksResized.getLeftEye()
        // const rightEye = landmarksResized.getRightEye()
        // const leftEyeBbrow = landmarksResized.getLeftEyeBrow()
        // const rightEyeBrow = landmarksResized.getRightEyeBrow()
        this.$refs.canvas
          .getContext('2d')
          .clearRect(0, 0, this.$refs.canvas.width, this.$refs.canvas.height)
        // faceapi.draw.drawDetections(this.$refs.canvas, resizedDetections)
        console.log(
          resizedDetections[0].detection.box.x,
          resizedDetections[0].detection.box.y,
          resizedDetections[0].detection.box.width,
          resizedDetections[0].detection.box.height
        )
        const box = new faceapi.draw.DrawBox(resizedDetections[0].detection.box, { label: 'asd' })
        box.draw(this.$refs.canvas)
        // faceapi.draw.DrawBox(
        //   this.ctx,
        //   resizedDetections[0].detection.box.x,
        //   resizedDetections[0].detection.box.y,
        //   resizedDetections[0].detection.box.width,
        //   resizedDetections[0].detection.box.height
        // )
        // faceapi.draw.drawFaceLandmarks(canvas, resizedDetections)
        console.log(landmarks.getLeftEyeBrow())
        this.$refs.canvas.getContext('2d').fillStyle = '#4245f5'
        this.$refs.canvas.getContext('2d').strokeStyle = '#fc03d7'
        this.$refs.canvas.getContext('2d').lineWidth = 1
        // canvas.getContext('2d').beginPath()
        // canvas.getContext('2d').moveTo(0, 0)
        // canvas.getContext('2d').lineTo(100, 100)
        // canvas.getContext('2d').stroke()
        // let startIndex = 0
        // landmarks.getLeftEyeBrow().map((point, index, points) => {
        //   if (index < points.length - 1) {
        //     startIndex++
        //     console.log('index', index, 'index++', startIndex)
        //     const nextPoint = points[startIndex]
        //     console.log(
        //       `line ${index}  x1: ${point.x} y1: ${point.y} x2: ${nextPoint.x} y2: ${nextPoint.y}`
        //     )
        //     // console.log('point', point.x, point.y)
        //     // console.log('nextPoint', points[nextPointIndex].x, points[nextPointIndex].y)
        //     this.ctx.beginPath()
        //     this.ctx.moveTo(point.x, point.y)
        //     this.ctx.lineTo(nextPoint.x, nextPoint.y)
        //     this.ctx.stroke()
        //   }
        //   this.ctx.fillRect(point.x, point.y, 3, 3)
        //   console.log(index, points.length)
        // })
        this.drawSingleFaceLandmard(landmarks.getLeftEyeBrow())
        this.drawSingleFaceLandmard(landmarks.getLeftEye())
        this.drawSingleFaceLandmard(landmarks.getRightEyeBrow())
        this.drawSingleFaceLandmard(landmarks.getRightEye())
        this.drawSingleFaceLandmard(landmarks.getNose())
        this.drawSingleFaceLandmard(landmarks.getMouth())
        this.drawSingleFaceLandmard(landmarks.getJawOutline())

        // faceapi.draw.drawContour(canvas.getContext('2d'), landmarks.getLeftEyeBrow())
        // faceapi.draw.drawContour(canvas.getContext('2d'), landmarks.getLeftEye())
        // faceapi.draw.drawContour(canvas.getContext('2d'), landmarks.getNose())
        faceapi.draw.drawFaceExpressions(this.$refs.canvas, resizedDetections)
      }, 1000)
    },
    drawSingleFaceLandmard(landmark) {
      let startIndex = 0
      landmark.map((point, index, points) => {
        if (index < points.length - 1) {
          startIndex++
          const nextPoint = points[startIndex]
          this.ctx.beginPath()
          this.ctx.moveTo(point.x, point.y)
          this.ctx.lineTo(nextPoint.x, nextPoint.y)
          this.ctx.stroke()
        }
        this.ctx.fillRect(point.x, point.y, this.pointSize, this.pointSize)
      })
    },
  },
}
</script>

<style lang="scss">
body {
  margin: 0;
  padding: 0;
  // background: #111111;

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
