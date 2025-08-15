<template>
  <div class="wrapper">
    <MapView />
  </div>
</template>

<script>
import MapView from './components/MapView.vue';

export default {
  components: { MapView },
};
</script>
<style>
#map {
  width: 80vw;
  height: 80vh;
  position: relative;
}
</style>
