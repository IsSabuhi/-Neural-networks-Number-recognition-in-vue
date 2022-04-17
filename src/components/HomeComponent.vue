<template>
  <v-app class="home">
    <h2 class="ma-5">Лабораторная работа 1. Нейронная сеть, которая распознает цифры от 0 до 9</h2>
    <div class="block">
      <v-sheet class="">
        <vue-drawing-canvas
          ref="VueCanvasDrawing"
          :image.sync="image"
          canvasId="VueCanvasDrawing"
          :width="width"
          :height="height"
          :lineWidth="line"
          saveAs="png"
          :styles="{border: 'solid 3px #000'}"
        />
      <div class="votpblock">
        <v-otp-input
          class="votp"
          length="1"
          type="string"
          v-model="vivod"
        >   
        </v-otp-input>
        </div> 
      </v-sheet>

      <v-sheet class="ma-15">
        <div class="mr-4">
          <v-btn color="error" @click="$refs.VueCanvasDrawing.reset()" class="mr-2"><v-icon>mdi-eraser</v-icon></v-btn>
          <v-btn color="success" @click="recognize()">Распознать</v-btn>
        </div>
        <div class="mr-4 my-2">
          <v-btn color="primary" @click="$refs.inputUpload.click()">Загрузить датасет</v-btn> 
          <input multiple accept=".png" v-show="false" ref="inputUpload" type="file" id="dataset" @change="(e) => loadDataset(e)">
        </div>

        <div class="mt-2">
          <v-btn color="primary" @click="save()">Сохранить веса</v-btn>
        </div>

        <div class="mt-2">
          <v-btn color="primary" @click="$refs.inputWeights.click()">
            Загрузить веса
          </v-btn>
          <input
            v-show="false"
            accept=""
            ref="inputWeights"
            type="file"
            @change="(e) => loadWeights(e)"
          />
        </div> 
      </v-sheet>

    </div>
  </v-app>
</template>

<script>
import VueDrawingCanvas from "vue-drawing-canvas";
var _ = require("lodash");
export default {
  name: "App",
  components: {
    VueDrawingCanvas,
  },
  data: () => ({
    image: "",
    width: 100,
    height: 100,
    line: 8,
    imageArray: null,
    weightMatrix: [],
    Neurons: 10,
    learnSpeed: 0.3,
    errors: 0,
    vivod: ''
  }),
  methods: {
    shuffle(arr = []) {
      let newArray = [];
      for (let i = arr.length - 1; i >= 0; i--) {
        let id = Math.floor(Math.random() * arr.length);
        newArray.push(arr[id]);
        arr.splice(id, 1);
      }
      return newArray;
    },
    async loadDataset(e) {
      let time = performance.now();
      let ctx = document.getElementById("VueCanvasDrawing").getContext("2d");
      let files = e.target.files;
      files = Object.values(files);
      files = this.shuffle(files);
      let vectorsAnswer = [];

      const promise = files.map(
        (file) =>
          new Promise((resolve) => {
            const reader = new FileReader();
            let fileName = file.name;
            reader.onload = (e) => {
              let img = new Image();
              img.onload = () => {

                ctx.drawImage(img, 0, 0);
                const indexCorrect = Number(fileName[0]);
                let answersImg = Array(this.Neurons).fill(0);

                answersImg[indexCorrect] = 1;
                vectorsAnswer.push({
                  answer: answersImg,
                  ImgVector: this.imgData(this.width, this.height),
                });
                resolve();
              };
              img.src = e.target.result;
            };
            reader.readAsDataURL(file);
          })
      );
      await Promise.all(promise);
      let percentCorrect = 0;
      let epoch = 0;
      do {
        percentCorrect = this.TrainDataset(_.shuffle(vectorsAnswer));
        epoch++;
        console.log(percentCorrect);
      } while (percentCorrect < 98);
      const getSeconds = () => {
        let res = (performance.now() - time) / 1000;
        return Number(res.toFixed(3));
      };
      console.log("Обучение завершено");
      this.$refs.VueCanvasDrawing.reset();
      console.table([
        ["Прошло эпох", epoch],
        ["Всего образов", vectorsAnswer.length],
        ["Всего ошибок", this.errors],
        ["Время обучения", getSeconds()],
      ]);
    },
    TrainDataset(vectorsAndAnswers) {
      for (let i = 0, arrLen = vectorsAndAnswers.length; i < arrLen; i++) {
        this.Train(vectorsAndAnswers[i].ImgVector, vectorsAndAnswers[i].answer);
      }
      let correctCount = 0;
      for (let i = 0, arrLen = vectorsAndAnswers.length; i < arrLen; i++) {
        let neuronOutput = this.Predict(vectorsAndAnswers[i].ImgVector, true);
        let sumError = 0;
        for (let j = 0, outputLen = neuronOutput.length; j < outputLen; j++) {
          const error = neuronOutput[j] - vectorsAndAnswers[i].answer[j];
          sumError += error;
        }
        if (sumError == 0) correctCount++;
      }
      console.log(vectorsAndAnswers.length - correctCount + " ошибок");
      this.errors += vectorsAndAnswers.length - correctCount;
      let correctPercent = (correctCount / vectorsAndAnswers.length) * 100;
      return correctPercent;
    },
    Train(ImgVector, answersImg) {
      let answer = this.Predict(ImgVector, true);
      for (
        let i = 0, neuronLen = this.weightMatrix.length;
        i < neuronLen;
        i++
      ) {
        let weightsDelta = answersImg[i] - answer[i];
        for (
          let j = 0, weightsLen = this.weightMatrix[i].length;
          j < weightsLen;
          j++
        ) {
          if (ImgVector[j] === 1 && weightsDelta != 0) {
            this.weightMatrix[i][j] += this.learnSpeed * weightsDelta;
          }
        }
      }
    },
    Predict(ImgVector, Learning) {
      let neuronSums = [];
      for (
        let i = 0, neuronLen = this.weightMatrix.length;
        i < neuronLen;
        i++
      ) {
        let sum = 0;
        for (
          let j = 0, weightsLen = this.weightMatrix[i].length;
          j < weightsLen;
          j++
        ) {
          sum += ImgVector[j] * this.weightMatrix[i][j];
        }
        if (Learning) {
          neuronSums.push(this.threshold(sum));
        } else neuronSums.push(sum);
      }
      return neuronSums;
    },
    threshold(sum) {
      return sum >= 0 ? 1 : 0;
    },
    imgData(width, height) {
      let context = document
        .getElementById("VueCanvasDrawing")
        .getContext("2d")
        .getImageData(0, 0, width, height).data;
      let array = Array.from(context);
      let newArray = array.filter((_, i) => i % 4 == 0);
      for (let i = 0; i < newArray.length; i++) {
        if (newArray[i] == 255) {
          newArray[i] = 0;
        } else newArray[i] = 1;
      }
      return newArray;
    },
    initWeights(canvasSize, Neurons, max, min) {
      let weightRes = [];
      for (let i = 0; i < Neurons; i++) {
        let weights = [];
        for (let j = 0; j < canvasSize ** 2; j++) {
          let randomNumber = Math.random() * (max - min) + min;
          weights.push(randomNumber);
        }
        weightRes.push(weights);
      }
      this.weightMatrix = weightRes;
    },
    recognize() {
      let imageVector = this.Predict(
        this.imgData(this.width, this.height),
        false
      );
      if (!imageVector.some((x) => x > 0)) return alert("Не распознано");
      console.log(imageVector);
      let max = Math.max(...imageVector);
      this.vivod = imageVector.indexOf(max).toString()
    },
    save() {
      const a = document.createElement("a");
      let data = JSON.stringify(this.weightMatrix);
      const file = new Blob([data], { type: "application/json" });
      a.href = URL.createObjectURL(file);
      a.download = "weights.json";
      a.click();
      URL.revokeObjectURL(a.href);
    },
    loadWeights(e) {
      const file = e.target.files[0];
      let reader = new FileReader();
      reader.readAsText(file);
      let context = this;
      reader.onload = function () {
        let res = reader.result;
        context.weightMatrix = JSON.parse(res);
        console.log("Веса загружены");
        console.log(context.weightMatrix);
      };
      reader.onerror = function () {
        console.log(reader.error);
        return;
      };
    },
  },
  mounted() {
    this.initWeights(this.width, this.Neurons, 0.3, -0.3);
  },
};
</script>

<style scoped>
.block {
  display: flex;
  flex-direction: row;
  flex-wrap: nowrap;
  justify-content: center;
  align-items: center;
}
.votp {
  max-width: 115px;
}
.votpblock span{
  font-size: 30px;
  color: red;
}
</style>
