<template>
  <div id="app">
    <button @click="visible = !visible">显示/隐藏标尺</button>
    <button @click="presetLine = []">清空</button>
    <button @click="presetLine = [{ type: 'h', site: 100 }, { type: 'v', site: 200 }]">修改</button>
    <section style="margin: 50px;padding: 50px;border: 1px solid red;height: 600px;">
      <v-ruler
        :parent="true"
        :is-scale-revise="true"
        v-model="presetLine"
        :visible.sync="visible"
      >
        <img src="https://cn.vuejs.org/images/logo.png" style="width: 300px;height: 300px;margin: 100px;" alt="">
        <!-- <iframe src="https://www.baidu.com" width="500" height="300" /> -->
      </v-ruler>
    </section>
  </div>
</template>

<script>
import VRuler from './components/v-ruler'
export default {
  name: 'app',
  components:{
    VRuler
  },
  data () {
    return {
      presetLine: [{ type: 'h', site: 200 }, { type: 'v', site: 100 }],
      visible: true
    }
  },
  watch: {
    visible(val) {
      console.log(val)
    }
  }
}
</script>

<style lang="scss">
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
</style>
