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
          <v-btn color="primary" >Инициализировать веса</v-btn>
        </div>
        <div>
          <v-btn color="primary" @click="$refs.inputUpload.click()">Загрузить датасет</v-btn> 
          <input multiple accept=".png" v-show="false" ref="inputUpload" type="file" id="load" @change="(e) => loadDataset(e)">
        </div>
        <div class="mt-2">
          <v-btn color="primary" @click="save(weightMatrix)">Сохранить веса</v-btn>
        </div>
        <div class="mt-2">
          <v-btn color="primary" @click="$refs.inputUpload.click()">Загрузить веса</v-btn> 
          <input v-show="false" ref="inputUpload" type="file" id="load" @change="(e) => processFile(e)">
        </div>        
      </v-sheet>

    </div>
  </v-app>
</template>

<script>
import Vue from 'vue';
import VueDrawingCanvas from 'vue-drawing-canvas';
import lodash from 'lodash'
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
    async loadDataset(e) {
      let time = performance.now();
      let _ = require(lodash);
      let files = e.target.files;
      files = Object.values(files);

      files = canvas.shuffle(files);

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
