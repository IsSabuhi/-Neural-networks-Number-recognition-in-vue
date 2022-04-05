<template>
  <v-app class="">
    <div class="pa-3">

      <v-sheet class="">
        <h1 class="">Рисовалка</h1>
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

      <v-sheet class="my-2">
        <div class="mr-4">
          <v-btn color="primary" @click="$refs.sketch.reset()" class="mr-2"><v-icon>mdi-eraser</v-icon></v-btn>
          <v-btn color="primary" @click="predict()">Распознать</v-btn>
        </div>
        <div class="mr-4 my-2">
          <v-btn color="primary" @click="setCross(); recalculateWeights()" class="mr-2">Крестик </v-btn>
          <v-btn color="primary" @click="setCircle(); recalculateWeights()">Нолик</v-btn>
        </div>
        <div class="">
          <v-btn color="primary" @click="save(weightMatrix)">Сохранить веса</v-btn>
        </div>
        <div class="mt-2">
          <v-btn color="primary" @click="$refs.inputUpload.click()">Загрузить веса</v-btn> 
          <input v-show="false" ref="inputUpload" type="file" id="load" @change="(e) => processFile(e)">
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
export default Vue.extend({
  name: 'HomeComponent',
  components: {
    VueDrawingCanvas
  },
  data: () => ({
    image: "",
    width: 150,
    height: 150,
    line: 25,
    canvas: "canvas",
    weights: [],
    errors: 0,
    speedLearn: 0.3,
    threshold: 0.01,
    imagesCount: 10,
  }),
  mounted() {
    // Загрузка датасета
async function loadDataset(e) {
  let time = performance.now();

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
}

  }

});
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
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
