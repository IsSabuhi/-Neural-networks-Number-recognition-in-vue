<template>
  <v-app class="home">
    <h2 class="ma-5">Лабораторная работа 1. Нейронная сеть, которая распознает цифры от 0 до 9</h2>
    <div class="block">
      <v-sheet class="">
        <vue-drawing-canvas
          ref="canvas"
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
          <input v-show="false" ref="inputUpload" type="file" id="load" @change="(e) => processFile(e)">
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
    width: 50,
    height: 50,
    line: 10,
    canvas: "canvas",
    weights: [],
    errors: 0,
    speedLearn: 0.3,
    threshold: 0.01,
    imagesCount: 10,
  }),
  methods: {
    calculateWeights() {
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
    async loadDataset(e) {
      let ctx = document.getElementById('VueCanvasDrawing').getContext('2d')
      let files = e.target.files;
      files = Object.values(files);
      
      files = this.shuffle(files);
      console.log(files)
      
      const promise = files.map(
      (file) =>
      new Promise((resolve) => {
        const reader = new FileReader();

        reader.onload = (e) => {
          let img = new Image();
          img.onload = () => {
            this.width = img.width;
            this.height = img.height;
            ctx.drawImage(img, 0, 0);

            resolve();
          };
          img.src = e.target.result;
        };
        reader.readAsDataURL(file);
      })
  );
      await Promise.all(promise);
        
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
