<template>
  <div id="app" class="container">
    <MainPage />
  </div>
</template>

<script>
import MainPage from "./components/MainPage.vue";

export default {
  name: "App",
  components: {
    MainPage,
  },
};
</script>

<style>
* {
  padding: 0;
  margin: 0;
  box-sizing: border-box;
}
html {
  background-color: #0d0d0d;
  color: white;
}

button {
  all: unset;
}

input {
  all: unset;
}

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  margin-top: 20px;
}
.container {
  max-width: 1200px;
  margin: 0 auto;
}
</style>
