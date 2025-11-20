<template>
  <div class="wrapper">
    <div class="ad-banner" role="banner" aria-label="site-ad">
      新北市市議員(土樹三鶯)擬參選人
      <span class="ad-name"> 吳亞倫 </span>
      團隊製作。
    </div>
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
:root {
  --ad-height: 40px;
}

.wrapper {
  display: flex;
  flex-direction: column;
  height: 100vh;
}

.ad-banner {
  flex: 0 0 var(--ad-height);
  height: var(--ad-height);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0 12px;
  background: #ffd54f; /* 醒目底色 */
  color: #111;
  font-weight: 800;
  font-size: 16px;
  text-align: center;
  letter-spacing: .3px;
  z-index: 1200;
}
.ad-banner .ad-name {
  margin: 0 6px;
}

#map {
  width: 100vw;
  height: 100vh;
  position: relative;
}

@media (max-width: 600px) {
  :root { --ad-height: 48px; }
  .ad-banner {
    height: auto;
    padding: 6px 10px;
    font-size: 13px;
    line-height: 1.1;
    white-space: normal;
  }
}
</style>
