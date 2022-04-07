<template>
  <v-app class="home">
    <h2 class="ma-5">Лабораторная работа 1. Нейронная сеть, которая распознает цифры от 0 до 9</h2>
    <div class="block">
      <v-sheet class="">
        <vue-drawing-canvas
          ref="canvas"
          :canvasId="canvas"
          :image.sync="image"
          :width="width"
          backgroundColor="#fff0"
          :height="height"
          :lineWidth="line"
          saveAs="png"
          :styles="{
            border: 'solid 3px #000',
          }"
        />
      </v-sheet>

      <v-sheet class="ma-15">
        <div class="mr-4">
          <v-btn color="error" @click="$refs.canvas.reset()" class="mr-2"><v-icon>mdi-eraser</v-icon></v-btn>
          <v-btn color="success" @click="predict()">Распознать</v-btn>
        </div>
        <div class="mr-4 my-2">
          <v-btn color="primary" @click="calculateWeights()">Инициализировать веса</v-btn>
        </div>
        <div>
          <v-btn color="primary" @click="$refs.inputUpload.click()">Загрузить датасет</v-btn> 
          <input multiple accept=".png" v-show="false" ref="inputUpload" type="file" id="dataset" @change="(e) => loadDataset(e)">
        </div>
        <div class="mt-2">
          <v-btn color="primary" @click="save(weightMatrix)">Сохранить веса</v-btn>
        </div>
        <!-- <div class="mt-2">
          <v-btn color="primary" @click="$refs.inputUpload.click()">Загрузить веса</v-btn> 
          <input multiple v-show="false" ref="inputUpload" type="file" id="load" @change="(e) => processFile(e)">
        </div>         -->
      </v-sheet>

    </div>
  </v-app>
</template>

<script>
import Vue from 'vue';
import VueDrawingCanvas from 'vue-drawing-canvas';
export default Vue.extend({
  name: 'HomeComponent',
  components: {
    VueDrawingCanvas
  },
  data: () => ({
    image: "",
    width: 150,
    height: 150,
    line: 10,
    canvas: "canvas",
    weights: [],
    errors: 0,
    speedLearn: 0.3,
    threshold: 0.01,
    imagesCount: 10,
  }),
  methods: {
    calculateWeights(){
      let weightMax = 0.3
      let weightMin = -weightMax
      this.weightMatrix = this.initWeights(this.width, this.height, weightMax, weightMin)
    },
    initWeights(width, height, max, min) {
      let weights = []
      for (let i = 0; i < width * height; i++) {
        weights.push(Number((Math.random() * (max - min) + min).toFixed(2)))        
      }
      console.log('Веса инициализированы')
      console.log(weights)
      return weights
      
    },
    shuffle(array) {
      array.sort(() => Math.random() - 0.5);
    },
    async loadDataset(e) {
      let time = performance.now();
      let _ = require('lodash');
      let files = e.target.files;
      files = Object.values(files);
      console.log(files)
      files = this.shuffle(files);

      let vectorsAndAnswer = [];

      const promise = files.map(
        (file) =>
          new Promise((resolve) => {
            const reader = new FileReader();
            let fileName = file.name;

            reader.onload = (e) => {
              let img = new Image();
              img.onload = () => {
                canvas.canvas.width = img.width;
                canvas.canvas.height = img.height;
                canvas.ctx.drawImage(img, 0, 0);

                const indexCorrect = Number(fileName[0]);
                let correctAnswers = Array(imagesCount).fill(0);

                correctAnswers[indexCorrect] = 1;
                vectorsAndAnswer.push({
                  answer: correctAnswers,
                  vector: canvas.getVectors(),
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
      let prevPercent = 0;
      let errorRepeat = 0;

      let epoch = 0;

      do {
        percentCorrect = neuron.TrainDataset(_.shuffle(vectorsAndAnswer));

        if (percentCorrect.toFixed(2) == prevPercent.toFixed(2)) {
          errorRepeat++;
        } else {
          errorRepeat = 0;
        }

        epoch++;

        console.log(percentCorrect);
        prevPercent = percentCorrect;
      } while (percentCorrect < 100);

      const getSeconds = () => {
        let res = (performance.now() - time) / 1000;
        return Number(res.toFixed(3));
      };

      console.log('finish');
      console.table([
        ['Прошло эпох', epoch],
        ['Всего образов', vectorsAndAnswer.length],
        ['Всего ошибок', neuron.errors],
        ['Прошло времени', getSeconds()],
      ]);
        
    },

    constructor(imagesRes, neuronNumber, thresholdError, speedLearn) {
        for (let i = 0; i < neuronNumber; i++) {
          let weightNeuron = [];
          for (let j = 0; j < imagesRes * imagesRes; j++) {
            let randomNumber = Math.random() * (0.31 + 0.31) - 0.31;
            weightNeuron.push(randomNumber);
          }
          this.weights.push(weightNeuron);
        }

        this.thresholdError = thresholdError;
        this.speedLearn = speedLearn;
      },

    // вычисление сигмоиды
    $sigmoid(sum) {
      return 1 / (1 + Math.exp(-sum));
    },

    // производная сигмоиды
    $sigmoid_derivative(sigmoid) {
      return sigmoid * (1 - sigmoid);
    },

    Predict(vector) {
    let neuronSums = [];

    for (let i = 0, neuronLen = this.weights.length; i < neuronLen; i++) {
      let sum = 0;
      for (
        let j = 0, weightsLen = this.weights[i].length;
        j < weightsLen;
        j++
      ) {
        sum += vector[j] * this.weights[i][j];
      }

      neuronSums.push(this.$sigmoid(sum));
    }

    return neuronSums;
  },

  Train(vector, correctAnswers) {
    let answer = this.Predict(vector);

    for (let i = 0, neuronLen = this.weights.length; i < neuronLen; i++) {
      let weightsDelta = correctAnswers[i] - answer[i];

      for (
        let j = 0, weightsLen = this.weights[i].length;
        j < weightsLen;
        j++
      ) {
        if (vector[j] === 1) {
          this.weights[i][j] += this.speedLearn * weightsDelta * vector[j];
        }
      }
    }
  },

  TrainDataset(vectorsAndAnswers) {
    for (let i = 0, arrLen = vectorsAndAnswers.length; i < arrLen; i++) {
      this.Train(vectorsAndAnswers[i].vector, vectorsAndAnswers[i].answer);
    }

    let correctCount = 0;

    for (let i = 0, arrLen = vectorsAndAnswers.length; i < arrLen; i++) {
      let neuronOutput = this.Predict(vectorsAndAnswers[i].vector);

      let sumError = 0;
      for (let j = 0, outputLen = neuronOutput.length; j < outputLen; j++) {
        const error = Math.pow(
          neuronOutput[j] - vectorsAndAnswers[i].answer[j],
          2
        );
        sumError += error;
      }

      if (sumError < this.thresholdError) correctCount++;
    }

    console.log(vectorsAndAnswers.length - correctCount + ' ошибок');

    this.errors += vectorsAndAnswers.length - correctCount;

    let correctPercent = (correctCount / vectorsAndAnswers.length) * 100;

    return correctPercent;
  },

    saveWeights() {
    const a = document.createElement('a');

    let data = JSON.stringify(this.weights);

    const file = new Blob([data], { type: 'application/json' });

    a.href = URL.createObjectURL(file);
    a.download = 'weights.json';
    a.click();

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
    },

}, 
mounted(){
    this.calculateWeights()
  }

});
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
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
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
