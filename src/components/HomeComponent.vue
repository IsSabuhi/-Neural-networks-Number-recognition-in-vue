<template>
  <v-app class="home">
    <h2 class="ma-5">Лабораторная работа 1. Нейронная сеть, которая распознает цифры от 0 до 9</h2>
    <div class="block">
      <v-sheet class="">
        <vue-drawing-canvas
          ref="VueCanvasDrawing"
          canvasId="VueCanvasDrawing"
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
          <v-btn color="error" @click="$refs.VueCanvasDrawing.reset()" class="mr-2"><v-icon>mdi-eraser</v-icon></v-btn>
          <v-btn color="success" @click="Predict()">Распознать</v-btn>
        </div>
        <div class="mr-4 my-2">
          <v-btn color="primary" @click="initWeights()">Инициализировать веса</v-btn>
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
          <input v-show="false" ref="inputUpload" type="file" id="load" @change="(e) => processFile(e)">
        </div>         -->
      </v-sheet>

    </div>
  </v-app>
</template>

<script>
import Vue from 'vue';
import VueDrawingCanvas from 'vue-drawing-canvas';
var _ = require("lodash");
export default Vue.extend({
  name: 'HomeComponent',
  components: {
    VueDrawingCanvas
  },
  data: () => ({
    image: "",
    weights: [],
    width: 50,
    height: 50,
    line: 5,
    imageArray: null,
    backgroundImage: null,
    thresholdError: 0,
    speedLearn: 0.3,
    neuronNumber: 10,
    imagesCount: 10,
    
  }),
  methods: {
    // Инициализация весов
    initWeights(thresholdError, speedLearn) {
      for (let i = 0; i < this.neuronNumber; i++) {
      let weightNeuron = [];
        for (let j = 0; j < this.width * this.height; j++) {
          let randomNumber = Math.random() * (0.31 + 0.31) - 0.31;
          weightNeuron.push(randomNumber);
        }
        this.weights.push(weightNeuron);
      }   

      this.thresholdError = thresholdError;
      this.speedLearn = speedLearn;
      console.log(this.weights)
    },
  // Перемешать массив
    shuffle(arr = []) {
    let newArr = [];

    for (let i = arr.length - 1; i >= 0; i--) {
      let id = Math.floor(Math.random() * arr.length);
      newArr.push(arr[id]);
      arr.splice(id, 1);
    }

    return newArr;
    },
    //Функция для преобразования картинок в вектор
    imageData(width, height){
      let context = document.getElementById('VueCanvasDrawing').getContext('2d').getImageData(0, 0, width, height).data
      let array = Array.from(context)
      // array.splice(0,3)
      let newArray = array.filter((_,i) => i % 4 == 0)
      for (let i = 0; i < newArray.length; i++) {
        if (newArray[i] == 255){
          newArray[i] = 0
        }
        else newArray[i] = 1
      }
      console.log(newArray)
      return newArray
    },
    // Функция для загрузки и получения картинок из canvas 
    async loadDataset(e) {
      let ctx = document.getElementById('VueCanvasDrawing').getContext('2d')
      let time = performance.now();
      let files = e.target.files;
      files = Object.values(files);
      files = this.shuffle(files);
      console.log(files)
        
      let vectorsAndAnswer = [];
      
      const promise = files.map(
      (file) =>
      new Promise((resolve) => {
        const reader = new FileReader();
        let fileName = file.name;

        reader.onload = (e) => {
          let img = new Image();
          img.onload = () => {
            this.imageData(this.width, this.height)
            ctx.drawImage(img, 0, 0);
            
            const indexCorrect = Number(fileName[0]);
            let correctAnswers = Array(this.imagesCount).fill(0);

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
      let prevPercent = 0;
      let errorRepeat = 0;

      let epoch = 0;

      do {
        percentCorrect = this.TrainDataset(_.shuffle(vectorsAndAnswer));

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
    // вычисление сигмоиды
    Threshold(sum) {
      return sum >= 0.5 ? 1 : 0;
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

        neuronSums.push(this.Threshold(sum));
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
    this.initWeights()
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
