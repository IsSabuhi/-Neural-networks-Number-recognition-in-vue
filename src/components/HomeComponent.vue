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
          <v-btn color="primary" @click="save(weightMatrix)">Сохранить веса</v-btn>
        </div>

        <div class="mt-2">
          <v-btn color="primary" @click="$refs.inputUpload.click()">Загрузить веса</v-btn> 
          <!-- <input v-show="false" ref="inputUpload" type="file" id="load" @change="(e) => processFile(e)"> -->
        </div>    
      </v-sheet>

    </div>
  </v-app>
</template>

<script>
import VueDrawingCanvas from 'vue-drawing-canvas';
import { indexOf } from 'lodash';
var _ = require('lodash');
export default {
  name: 'App',
  components: {
    VueDrawingCanvas,
  },
  data: () => ({
    image: '',
    width: 100,
    height: 100,
    line: 8,
    imageArray: null,
    weightMatrix: [],
    numberOfNeurons: 10,
    learnSpeed: 0.7,
    errors: 0,
    vivod: ""
  }),
  methods: {
    shuffle(arr = []) {
      let newArr = [];
      for (let i = arr.length - 1; i >= 0; i--) {
        let id = Math.floor(Math.random() * arr.length);
        newArr.push(arr[id]);
        arr.splice(id, 1);
      }
      return newArr;
    },
    async loadDataset(e) {
      let time = performance.now();
      let ctx = document.getElementById('VueCanvasDrawing').getContext('2d');
      let files = e.target.files;
      files = Object.values(files);
      files = this.shuffle(files);
      let vectorsAndAnswer = [];
      console.log(files);
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
                let correctAnswers = Array(this.numberOfNeurons).fill(0);
                correctAnswers[indexCorrect] = 1;
                vectorsAndAnswer.push({
                  answer: correctAnswers,
                  vector: this.imageData(this.width, this.height),
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
        percentCorrect = this.TrainDataset(_.shuffle(vectorsAndAnswer));
        epoch++;
        console.log(percentCorrect);
      } while (percentCorrect < 100);

      const getSeconds = () => {
        let res = (performance.now() - time) / 1000;
        return Number(res.toFixed(3));
      };
      console.log('Обучение завершено');
      this.$refs.VueCanvasDrawing.reset();
      console.table([
        ['Прошло эпох', epoch],
        ['Всего образов', vectorsAndAnswer.length],
        ['Всего ошибок', this.errors],
        ['Прошло времени', getSeconds()],
      ]);
    },
    TrainDataset(vectorsAndAnswers) {
      for (let i = 0, arrLen = vectorsAndAnswers.length; i < arrLen; i++) {
        this.Train(vectorsAndAnswers[i].vector, vectorsAndAnswers[i].answer);
      }
      let correctCount = 0;
      for (let i = 0, arrLen = vectorsAndAnswers.length; i < arrLen; i++) {
        let neuronOutput = this.Predict(vectorsAndAnswers[i].vector, true);
        let sumError = 0;
        for (let j = 0, outputLen = neuronOutput.length; j < outputLen; j++) {
          const error = neuronOutput[j] - vectorsAndAnswers[i].answer[j];
          sumError += error;
        }
        if (sumError == 0) correctCount++;
      }
      console.log(vectorsAndAnswers.length - correctCount + ' ошибок');
      this.errors += vectorsAndAnswers.length - correctCount;
      let correctPercent = (correctCount / vectorsAndAnswers.length) * 100;
      return correctPercent;
    },
    // Корректировка весов
    Train(vector, correctAnswers) {
      let answer = this.Predict(vector, true);
      for (
        let i = 0, neuronLen = this.weightMatrix.length;
        i < neuronLen;
        i++
      ) {
        let weightsDelta = correctAnswers[i] - answer[i];
        for (
          let j = 0, weightsLen = this.weightMatrix[i].length;
          j < weightsLen;
          j++
        ) {
          if (vector[j] === 1) {
            this.weightMatrix[i][j] += this.learnSpeed * weightsDelta;
          }
        }
      }
    },
    Predict(vector, isLearning) {
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
          sum += vector[j] * this.weightMatrix[i][j];
        }
        if (isLearning) {
          neuronSums.push(this.threshold(sum));
        } else neuronSums.push(sum);
      }
      return neuronSums;
    },
    threshold(sum) {
      return sum >= 0 ? 1 : 0;
    },
    imageData(width, height) {
      let context = document
        .getElementById('VueCanvasDrawing')
        .getContext('2d')
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
    initWeights(canvasSize, numberOfNeurons, max, min) {
      let weightRes = [];
      for (let i = 0; i < numberOfNeurons; i++) {
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
        this.imageData(this.width, this.height),
        false
      );
      if (!imageVector.some((x) => x > 0)) return alert('Не распознано');
      console.log(imageVector);
      let max = Math.max(...imageVector);
      // alert(imageVector.indexOf(max));
      this.vivod = imageVector.indexOf(max).toString()
    },
    save(content) {
      const a = document.createElement('a');
      const file = new Blob([content], { type: 'text/plain' });
      a.href = URL.createObjectURL(file);
      a.download = 'weights.txt';
      a.click();
      console.log('Веса сохранены')
      URL.revokeObjectURL(a.href);
    },
    readFileAsync(file) {
      return new Promise((resolve, reject) => {
        let reader = new FileReader();
        reader.onload = () => {
          resolve(reader.result);
        };
        reader.onerror = reject;
        reader.readAsText(file);
      })
    },
    async processFile(e) {
      try {
        let file = e.target.files[0];
        let text = await this.readFileAsync(file);
        text = text.split(',')
        text = text.map((n) => Number(n))
        this.weightMatrix = text
        console.log('Веса загружены')
      } catch(err) {
        console.log(err);
      }
    }
  },
  
  mounted() {
    this.initWeights(this.width, this.numberOfNeurons, 0.3, -0.3);
  },
};
</script>

<style scoped>
.home {
  
}
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
