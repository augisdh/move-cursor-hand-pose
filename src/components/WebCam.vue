<template>
  <div class="webcam">
    <div ref="mouse" class="mouse"></div>
    <div>
      <canvas
        ref="canvas"
        class="canvas"
        :width="width"
        :height="height"
      ></canvas>
      <video ref="video" id="video" :width="width" :height="height"></video>
    </div>
    <div>
      <button @click="enableHandPose">Enable Hand Pose</button>
      <button @click="enabledHandPose = !enabledHandPose">
        CHANGE Hand Pose state
      </button>
    </div>

    enabledHandPose: {{ enabledHandPose }}
  </div>
</template>

<style lang="scss" scoped>
// .canvas {
//   position: absolute;
// }
.mouse {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  width: 10px;
  height: 15px;
  background: red;
  cursor: pointer;
}
</style>

<script lang="ts">
import Vue from "vue";
import * as handpose from "@tensorflow-models/handpose";
require("@tensorflow/tfjs-backend-cpu");
require("@tensorflow/tfjs-backend-webgl");

export default Vue.extend({
  name: "webcam",
  data: () => ({
    video: {} as any,
    canvas: {} as any,
    ctx: {} as any,
    enabledHandPose: false,
    width: 640,
    height: 480,
    model: null as any,
    loadedModel: false,
    fingerLookupIndices: {
      thumb: [0, 1, 2, 3, 4],
      indexFinger: [0, 5, 6, 7, 8],
      middleFinger: [0, 9, 10, 11, 12],
      ringFinger: [0, 13, 14, 15, 16],
      pinky: [0, 17, 18, 19, 20],
    } as any,
    indexFinger: {} as any,
    prevIndexFinger: null as any,
    thumb: {} as any,
    middleFinger: {} as any,
    ringFinger: {} as any,
    pinky: {} as any,
    testX: 0,
    testY: 0,
  }),
  mounted() {
    this.video = this.$refs.video;
    this.canvas = this.$refs.canvas;
    this.ctx = this.canvas.getContext("2d");
    this.streamWebCam();
  },
  methods: {
    // WEBCAM steaming
    streamWebCam() {
      try {
        navigator.getUserMedia(
          { video: true },
          (stream) => {
            this.video.srcObject = stream;
            this.video.play();
            this.loadModel();
          },
          (err) => {
            console.error(err);
          }
        );
      } catch (error) {
        console.error(error);
      }
    },
    runGrayscale() {
      this.ctx.drawImage(this.video, 0, 0, this.width, this.height);
      const frame = this.ctx.getImageData(0, 0, this.width, this.height);
      const l = frame.data.length / 4;

      for (let i = 0; i < l; i++) {
        const grey =
          (frame.data[i * 4 + 0] +
            frame.data[i * 4 + 1] +
            frame.data[i * 4 + 2]) /
          3;

        frame.data[i * 4 + 0] = grey;
        frame.data[i * 4 + 1] = grey;
        frame.data[i * 4 + 2] = grey;
      }
      console.log(frame);
      this.ctx.putImageData(frame, 0, 0);
    },
    // PANTING
    drawKeypoints(result: any) {
      // this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
      this.indexFinger = result[0].annotations.indexFinger;
      this.drawSelected(this.indexFinger);
      this.thumb = result[0].annotations.thumb;
      this.drawSelected(this.thumb);
      this.middleFinger = result[0].annotations.middleFinger;
      this.drawSelected(this.middleFinger);
      this.pinky = result[0].annotations.pinky;
      this.drawSelected(this.pinky);
      this.ringFinger = result[0].annotations.ringFinger;
      this.drawSelected(this.ringFinger);

      const fingers = Object.keys(this.fingerLookupIndices);
      fingers.forEach((finger: any) => {
        const points = this.fingerLookupIndices[finger].map(
          (idx: any) => result[0].landmarks[idx]
        );
        this.drawPath(points, false);
      });
    },
    drawPath(points: any, closePath: any) {
      const region = new Path2D();
      region.moveTo(points[0][0], points[0][1]);
      for (let i = 1; i < points.length; i++) {
        const point = points[i];
        region.lineTo(point[0], point[1]);
      }

      if (closePath) {
        region.closePath();
      }
      this.ctx.stroke(region);
    },
    drawPoint(y: any, x: any, r: any) {
      this.ctx.beginPath();
      this.ctx.arc(x, y, r, 0, 2 * Math.PI);
      this.ctx.fill();
    },
    drawSelected(fingerArr: any) {
      fingerArr.forEach((finger: any, index: number) => {
        if (index < 1) return;
        const y = finger[0];
        const x = finger[1];
        this.ctx.fillStyle = "red";
        this.drawPoint(x - 2, y - 2, 3);
      });
    },
    // HANDPOSE shit
    async loadModel() {
      try {
        this.model = await handpose.load();
        this.loadedModel = true;
      } catch (error) {
        this.loadedModel = false;
        console.error(`ERROR: ${error}`);
      }
    },
    async enableHandPose() {
      this.runGrayscale();
      if (!this.loadedModel || !this.model) return;
      console.log("RUNNNIG");
      try {
        const result = await this.model.estimateHands(this.video);
        console.log(result);
        if (!result.length) {
          console.log("NO PREDICTIONS");
        } else {
          this.drawKeypoints(result);
          this.movementCursor();
        }
        console.log("FINISHED");

        if (this.enabledHandPose) {
          setTimeout(() => this.enableHandPose(), 100);
        }
      } catch (error) {
        this.enabledHandPose = false;
        console.error(`ERROR enable: ${error}`);
      }
    },
    // Move cursor with index finger.
    // Index finger tip should be highest to move cursor.
    // Ex: this.indexFinger[3][1] index [3] is finger tip location [1] is y position.
    movementCursor() {
      // CHECKING FINGER TIP
      const indexFingerTipY = this.indexFinger[3][1];
      const middleFingerTipY = this.middleFinger[3][1];
      const otherFingerTipsY = [
        this.thumb[3][1],
        this.ringFinger[3][1],
        this.pinky[3][1],
      ].sort((first: number, second: number) => (first > second ? 0 : -1));
      const movingCursorY =
        indexFingerTipY <= otherFingerTipsY[0] &&
        middleFingerTipY <= otherFingerTipsY[0];

      // CHECKING FINGER SECOND FROM THE TOP
      const indexFingerSecondY = this.indexFinger[2][1];
      const middleFingerSecondY = this.middleFinger[2][1];
      const otherFingerSecondsY = [
        this.thumb[2][1],
        this.ringFinger[2][1],
        this.pinky[2][1],
      ].sort((first: number, second: number) => (first > second ? 0 : -1));
      const movingCursorSecondY =
        indexFingerSecondY <= otherFingerSecondsY[0] &&
        middleFingerSecondY <= otherFingerSecondsY[0];

      // CHECKING FINGER SECOND FROM THE TOP
      const indexFingerThirdY = this.indexFinger[1][1];
      const middleFingerThirdY = this.middleFinger[1][1];
      const otherFingerThirdsY = [
        this.thumb[1][1],
        this.ringFinger[1][1],
        this.pinky[1][1],
      ].sort((first: number, second: number) => (first > second ? 0 : -1));
      const movingCursorThirdY =
        indexFingerThirdY <= otherFingerThirdsY[0] &&
        middleFingerThirdY <= otherFingerThirdsY[0];

      if (!this.prevIndexFinger) {
        this.prevIndexFinger = this.indexFinger[3];
      }

      if (movingCursorY && movingCursorSecondY && movingCursorThirdY) {
        // X and Y position diffs
        const xDiff = this.prevIndexFinger[0] - this.indexFinger[3][0];
        const yDiff = this.prevIndexFinger[1] - this.indexFinger[3][1];

        // Screen border
        const SCREEN_WIDTH = window.innerWidth;
        const SCREEN_HEIGHT = window.innerHeight;

        // Mouse to move / how much
        const mouse: any = this.$refs.mouse;
        const moveY =
          Math.sign(yDiff) === -1 ? mouse.offsetTop + 20 : mouse.offsetTop - 20;
        const moveX =
          Math.sign(xDiff) === -1
            ? mouse.offsetLeft - 20
            : mouse.offsetLeft + 20;
        // x, y diffs converted to positive number
        const xDiffMathed = Math.sign(xDiff) === -1 ? xDiff * -1 : xDiff;
        const yDiffMathed = Math.sign(yDiff) === -1 ? yDiff * -1 : yDiff;
        // MOVE DIV if all TRUE
        if (Math.sign(moveX) === 1 && moveX < SCREEN_WIDTH && xDiffMathed > 7) {
          mouse.style.left = `${moveX}px`;
          this.ctx.font = "24px Arial";
          this.ctx.fillText("MOVING SIDE", 10, 50);
        }
        if (
          Math.sign(moveY) === 1 &&
          moveY < SCREEN_HEIGHT &&
          yDiffMathed > 4
        ) {
          mouse.style.top = `${moveY}px`;
          this.ctx.font = "24px Arial";
          this.ctx.fillText("MOVING UP/DOWN", 10, 50);
        }

        // SAVE INDEX FINGER FOR LATER COMPARISON
        this.prevIndexFinger = this.indexFinger[3];
      } else {
        this.ctx.font = "24px Arial";
        this.ctx.fillText("NOT MOVING", 10, 50);
        this.prevIndexFinger = null;
      }
    },
  },
});
</script>