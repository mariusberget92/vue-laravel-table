<script>
import { defineComponent } from 'vue';
import VueLaravelTable from '@/vue-laravel-table.vue';

export default defineComponent({
  name: 'ServeDev',
  components: {
    VueLaravelTable
  }
});
</script>

<template>
  <div id="app">
    <vue-laravel-table />
  </div>
</template>
