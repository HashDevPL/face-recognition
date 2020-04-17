<template>
  <div id="app">
    <div class="sidebar">
      <div class="controler">
        <div class="title">Detect every {{ detectionIntervalTime }}ms</div>
        <input
          type="range"
          class="interval"
          v-model="detectionIntervalTime"
          min="100"
          max="2000"
          step="100"
        />
      </div>
      <div class="options">
        <div class="title">Detect</div>
        <div class="checkbox">
          <input type="checkbox" id="leftEyeDetection" v-model="detect.leftEye" />
          <label for="leftEyeDetection">Left eye</label>
        </div>
        <div class="checkbox">
          <input type="checkbox" id="leftEyeBrowDetection" v-model="detect.leftEyeBrow" />
          <label for="leftEyeBrowDetection">Left eye brow</label>
        </div>
        <div class="checkbox">
          <input type="checkbox" id="lrightEyeDetection" v-model="detect.rightEye" />
          <label for="lrightEyeDetection">Right eye</label>
        </div>
        <div class="checkbox">
          <input type="checkbox" id="rightEyeBrowDetection" v-model="detect.rightEyeBrow" />
          <label for="rightEyeBrowDetection">Right eye brow</label>
        </div>
        <div class="checkbox">
          <input type="checkbox" id="noseDetection" v-model="detect.nose" />
          <label for="noseDetection">Nose</label>
        </div>
        <div class="checkbox">
          <input type="checkbox" id="mouthDetection" v-model="detect.mouth" />
          <label for="mouthDetection">Mouth</label>
        </div>
        <div class="checkbox">
          <input type="checkbox" id="jawOutlineDetection" v-model="detect.jawOutline" />
          <label for="jawOutlineDetection">Jaw outline</label>
        </div>
      </div>
      <div class="options">
        <div class="title">Predict</div>
        <div class="checkbox">
          <input type="checkbox" id="genderPrediction" v-model="predict.gender" />
          <label for="genderPrediction">Gender</label>
        </div>
        <div class="checkbox">
          <input type="checkbox" id="agePrediction" v-model="predict.age" />
          <label for="agePrediction">Age</label>
        </div>
        <div class="checkbox">
          <input type="checkbox" id="expressionPrediction" v-model="predict.expression" />
          <label for="expressionPrediction">Expression</label>
        </div>
      </div>
    </div>
    <div class="content">
      <div class="wrapper">
        <video
          ref="video"
          :width="videoSize.width"
          :height="videoSize.height"
          muted
          autoplay
        ></video>
        <canvas ref="canvas" :width="videoSize.width" :height="videoSize.height"></canvas>
      </div>
      <div class="loader" v-if="!loaded">
        <svg
          xmlns="http://www.w3.org/2000/svg"
          xmlns:xlink="http://www.w3.org/1999/xlink"
          width="400px"
          height="400px"
          viewBox="0 0 100 100"
          preserveAspectRatio="xMidYMid"
        >
          <path
            fill="none"
            stroke="#fc03d7"
            stroke-width="6"
            stroke-dasharray="42.76482137044271 42.76482137044271"
            d="M24.3 30C11.4 30 5 43.3 5 50s6.4 20 19.3 20c19.3 0 32.1-40 51.4-40 C88.6 30 95 43.3 95 50s-6.4 20-19.3 20C56.4 70 43.6 30 24.3 30z"
            stroke-linecap="round"
            style="transform:scale(1);transform-origin:50px 50px"
          >
            <animate
              attributeName="stroke-dashoffset"
              repeatCount="indefinite"
              dur="2.5s"
              keyTimes="0;1"
              values="0;256.58892822265625"
            ></animate>
          </path>
        </svg>
      </div>
    </div>
  </div>
</template>

<script>
import * as faceapi from 'face-api.js'

export default {
  name: 'App',
  data() {
    return {
      videoSize: { width: 1280, height: 720 },
      loaded: false,
      ctx: null,
      pointSize: 3,
      detectionIntervalTime: 500,
      detectionInterval: null,
      detect: {
        leftEye: true,
        leftEyeBrow: true,
        rightEye: true,
        rightEyeBrow: true,
        nose: true,
        mouth: true,
        jawOutline: true,
      },
      predict: { gender: true, age: true, expression: true },
    }
  },
  mounted() {
    navigator.getUserMedia(
      { video: this.videoSize },
      stream => (this.$refs.video.srcObject = stream),
      error => console.error(error)
    )
    Promise.all([
      faceapi.loadTinyFaceDetectorModel('./models'),
      faceapi.loadFaceLandmarkModel('./models'),
      faceapi.loadFaceExpressionModel('./models'),
      faceapi.loadAgeGenderModel('./models'),
    ]).then(this.startDetections())
    this.ctx = this.$refs.canvas.getContext('2d')
  },
  methods: {
    startDetections() {
      faceapi.matchDimensions(this.$refs.canvas, this.videoSize)
      this.createInterval()
    },
    createInterval() {
      this.detectionInterval = setInterval(async () => {
        const detections = await faceapi
          .detectAllFaces(this.$refs.video, new faceapi.TinyFaceDetectorOptions())
          .withFaceLandmarks()
          .withFaceExpressions()
          .withAgeAndGender()

        if (detections.length > 0) this.loaded = true

        const resizedDetections = faceapi.resizeResults(detections, this.videoSize)
        const landmarks = []

        detections.map(item => landmarks.push(item.landmarks))

        this.$refs.canvas
          .getContext('2d')
          .clearRect(0, 0, this.$refs.canvas.width, this.$refs.canvas.height)

        resizedDetections.forEach(item =>
          this.drawSingleFaceDetection(item.detection.box, item.gender, item.age, item.expressions)
        )

        this.ctx.fillStyle = '#4245f5'
        this.ctx.strokeStyle = '#fc03d7'
        this.ctx.lineWidth = 1

        this.drawFaceLandmarks(landmarks)
      }, this.detectionIntervalTime)
    },
    deleteInterval() {
      clearInterval(this.detectionInterval)
    },
    drawSingleFaceDetection(box, gender, age, faceExpressions) {
      const normalizedGender = gender ? gender.charAt(0).toUpperCase() + gender.slice(1) : '?'
      const normalizedAge = age ? age.toFixed(0) : '?'
      const normalizedFaceExpression = faceExpressions
        ? this.getFaceExpression(faceExpressions)
        : '?'

      let faceBoxLabel = ''

      if (this.predict.gender) {
        faceBoxLabel += normalizedGender
      }
      if (this.predict.gender && this.predict.age) {
        faceBoxLabel += '/'
      }
      if (this.predict.age) {
        faceBoxLabel += normalizedAge
      }
      if (this.predict.gender && !this.predict.age && this.predict.expression) {
        faceBoxLabel += '/'
      }
      if (this.predict.age && this.predict.expression) {
        faceBoxLabel += '/'
      }
      if (this.predict.expression) {
        faceBoxLabel += normalizedFaceExpression
      }

      const faceBox = new faceapi.draw.DrawBox(box, {
        label: faceBoxLabel,
        lineWidth: 1,
        boxColor: '#fc03d7',
        drawLabelOptions: { fontColor: '#000000' },
      })

      faceBox.draw(this.$refs.canvas)
    },
    drawFaceLandmarks(landmarks) {
      landmarks.forEach(item => {
        if (this.detect.leftEyeBrow) {
          this.drawSingleFaceLandmard(item.getLeftEyeBrow())
        }
        if (this.detect.leftEye) {
          this.drawSingleFaceLandmard(item.getLeftEye())
        }
        if (this.detect.rightEyeBrow) {
          this.drawSingleFaceLandmard(item.getRightEyeBrow())
        }
        if (this.detect.rightEye) {
          this.drawSingleFaceLandmard(item.getRightEye())
        }
        if (this.detect.nose) {
          this.drawSingleFaceLandmard(item.getNose())
        }
        if (this.detect.mouth) {
          this.drawSingleFaceLandmard(item.getMouth())
        }
        if (this.detect.jawOutline) {
          this.drawSingleFaceLandmard(item.getJawOutline())
        }
      })
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
    getFaceExpression(expressions) {
      const faceExpressionsProbability = Object.values(expressions).map(item =>
        Number(item).toFixed(5)
      )
      const maxFaceExpressionsProbability = Math.max.apply(Math, faceExpressionsProbability)

      const faceExpression = Object.keys(expressions).filter(function(key) {
        return Number(expressions[key]).toFixed(5) == maxFaceExpressionsProbability
      })[0]
      return faceExpression.charAt(0).toUpperCase() + faceExpression.slice(1)
    },
  },
  watch: {
    detectionIntervalTime() {
      this.deleteInterval()
      this.createInterval()
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

    .sidebar {
      flex-basis: 15%;
      max-width: calc(100% - 1280px);
      height: 100%;
      background: #000;
      color: #c1c1c1;
      font-weight: bold;

      .title {
        font-size: 18px;
        margin-bottom: 10px;
      }

      .controler {
        padding: 10px;
        .interval {
          margin: 10px 0 10px 0;
          appearance: none;
          cursor: pointer;
          -webkit-appearance: none;
          height: 10px;
          background: #838383;
          outline: none;

          &::-webkit-slider-thumb {
            -webkit-appearance: none;
            appearance: none;
            width: 15px;
            height: 20px;
            background: #fc03d7;
          }

          &::-moz-range-thumb {
            width: 20px;
            height: 20px;
            background: #fc03d7;
          }
        }
      }

      .options {
        padding: 10px;

        .checkbox {
          width: fit-content;
          margin: 0 0 10px 0;

          input[type='checkbox'] {
            visibility: hidden;
            position: absolute;
          }

          label {
            display: flex;
            align-items: center;
            height: 20px;
            &:hover {
              cursor: pointer;
            }
            &:before {
              content: '';
              display: block;
              width: 15px;
              height: 15px;
              background: none;
              border: 1px solid #fc03d7;
              margin-right: 5px;
            }
          }

          input:checked + label {
            &:before {
              background: #fc03d7;
            }
          }
        }
      }
    }

    .content {
      flex-basis: 85%;
      min-width: 1280px;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100%;
      position: relative;

      .wrapper {
        position: relative;
        canvas {
          position: absolute;
          top: 0;
          left: 0;
        }
      }

      .loader {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        display: flex;
        justify-content: center;
        align-items: center;
        background: rgba(0, 0, 0, 0.9);
      }
    }
  }
}
</style>
