<template>
  <div>
    <image-map
        :value="value"
        @input="zones => change({ zones })"/>
  </div>
</template>

<script>
import ImageMap from './components/ImageMap.vue'

export default {
  name: 'app',
  components: {
    ImageMap
  },
  data(){
    return {
      value:{
        uuid: 3721,
        source:'http://n.sinaimg.cn/news/transform/20170330/K2Gk-fycwymx2694027.jpg'
      }
    }
  },
  methods: {
    change(changes) {
      console.log( { ...this.value, ...changes });
    },
  },
}
</script>

<style>
</style>
