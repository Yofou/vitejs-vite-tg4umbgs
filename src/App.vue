<script setup>
import { ref } from 'vue';
import HelloWorld from './components/HelloWorld.vue';

const content = ref('');
</script>

<template>
  <HelloWorld v-model="content" label="Message" :maxLength="350" />
</template>
