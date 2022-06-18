<template>
  <div class="home">
    <MapComponent />

    <!-- <button id="scroll-to-bottom" @click="scrollToBottom(1000)">Play</button> -->
  </div>
</template>

<script>
import * as d3 from 'd3'
// @ is an alias to /src
import MapComponent from '@/components/MapComponent.vue'

export default {
  name: 'HomeView',
  components: {
    MapComponent,
  },
  methods: {
    scrollToBottom: function (duration = 300) {
      const initY = window.scrollY
      let start = null

      const ease = d3.easeSinOut
      const step = (timestamp) => {
        start = start || timestamp
        const progress = timestamp - start,
          // Growing from 0 to 1
          time = Math.min(1, (timestamp - start) / duration)

        window.scrollTo(0, initY + ease(time) * initY)
        if (progress < duration) {
          window.requestAnimationFrame(step)
        }
      }

      window.requestAnimationFrame(step)
    },
  },
}
</script>
<style>
html {
  scroll-behavior: smooth;
}
.home {
  height: 6000vh;
}
#scroll-to-bottom {
  font-size: 3rem;
  padding: 1rem;
  position: fixed;
  top: 1em;
  left: 1em;
  z-index: 100;
}
</style>
