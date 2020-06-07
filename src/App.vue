<template>
  <div id="app">
    <div id="init" v-if="!videoId">
      ここにTwitchの動画ページURLをコピペ :
      <input type="text" @blur="changeTwitchUrl($event)" />
    </div>
    <Twitch v-if="videoId" :video="videoId" />
  </div>
</template>

<script>
import Twitch from "./components/Twitch.vue";

export default {
  name: "App",

  data() {
    return {
      videoId: "556433533"
    };
  },

  methods: {
    changeTwitchUrl(event) {
      const match = event.target.value.match(/videos\/(?<id>\d+)/);
      this.videoId = match.groups && match.groups.id ? match.groups.id : null;
    }
  },

  components: {
    Twitch
  }
};
</script>

<style>
#app {
  font-family: sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
</style>
