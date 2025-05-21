<template>
  <div id="app">
    <h1>Vue3 ソリティア</h1>
    <SolitaireGame />
  </div>
</template>

<script setup lang="ts">
import SolitaireGame from './components/SolitaireGame.vue'
</script>

<style scoped>
#app {
  max-width: 900px;
  margin: 0 auto;
  padding: 1rem;
  text-align: center;
}
h1 {
  margin-bottom: 1rem;
}
</style>
