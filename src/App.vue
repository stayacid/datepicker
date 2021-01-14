<template>
  <div id="app">
    <Datepicker/>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import Datepicker from './components/Datepicker.vue';

@Component({
  components: {
    Datepicker,
  },
})
export default class App extends Vue {}
</script>

<style lang="scss">

</style>
