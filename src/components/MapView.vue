<template>
  <div id="map"></div>

  <!-- 右側控制面板 -->
  <div class="control-panel">
    <h3>Marker 編輯</h3>
    <div v-if="selectedMarker">
      <label>標記類型</label>
      <select
        v-model="selectedMarker.customData.type"
        @change="updateMarkerIcon"
      >
        <option value="icon">圖示</option>
        <option value="text">文字</option>
      </select>

      <label>文字</label>
      <input
        v-model="selectedMarker.customData.text"
        @input="updateMarkerIcon"
      />

      <label>文字顏色</label>
      <input
        type="color"
        v-model="selectedMarker.customData.textColor"
        @input="updateMarkerIcon"
      />

      <label>背景顏色</label>
      <input
        type="color"
        v-model="selectedMarker.customData.bgColor"
        @input="updateMarkerIcon"
      />

      <label>字體大小(px)</label>
      <input
        type="number"
        min="8"
        max="50"
        v-model.number="selectedMarker.customData.fontSize"
        @input="updateMarkerIcon"
      />
    </div>
    <p v-else>點擊 Marker 以編輯</p>

    <hr />
    <label>路線顏色</label>
    <input type="color" v-model="routeColor" @input="updateRouteColor" />

    <hr />
    <div>
      <label>
        <input
          type="checkbox"
          v-model="showVillages"
          @change="toggleVillageLayer"
        />
        顯示各里邊界
      </label>
    </div>

    <button @click="undoLastPoint">撤銷最後一點</button>
    <button @click="clearRoute">清空路線</button>
  </div>
</template>

<script>
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';
import * as turf from '@turf/turf';

export default {
  data() {
    return {
      map: null,
      routePoints: [],
      routeLine: null,
      markers: [],
      routeColor: '#ff0000',
      selectedMarker: null,
      defaultMarker: {
        type: 'icon', // 預設為圖示
        text: '標註',
        textColor: '#000000',
        bgColor: '#ffff99',
        fontSize: 14,
      },
      showVillages: false,
      villageLayer: null,
    };
  },
  mounted() {
    this.initMap();
    this.loadFromLocalStorage();
    this.loadVillageLayer();
  },
  methods: {
    initMap() {
      this.map = L.map('map').setView([24.958, 121.403], 12);
      L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '© OpenStreetMap contributors',
        crossOrigin: true,
      }).addTo(this.map);
      this.map.on('click', this.onMapClick);
    },
    onMapClick(e) {
      const latlng = [e.latlng.lat, e.latlng.lng];
      this.routePoints.push(latlng);
      const marker = this.createMarker(latlng, { ...this.defaultMarker });
      this.markers.push(marker);
      if (!this.routeLine) {
        this.routeLine = L.polyline(this.routePoints, {
          color: this.routeColor,
        }).addTo(this.map);
      } else {
        this.routeLine.setLatLngs(this.routePoints);
      }
      this.saveToLocalStorage();
    },
    loadVillageLayer() {
      fetch('./villages.geojson')
        .then((res) => res.json())
        .then((data) => {
          // 建立 polygon 層，但先不加到地圖上
          this.villageLayer = L.geoJSON(data, {
            style: {
              color: '#555',
              weight: 1,
              fillColor: '#ffffff',
              fillOpacity: 0.3,
            },
          });

          // 建立里名 label
          this.villageLabels = [];
          data.features.forEach((feature) => {
            const latlng = this.getPolygonCentroid(feature.geometry);
            const label = L.marker(latlng, {
              icon: L.divIcon({
                className: 'village-label',
                html: feature.properties.ADMIV,
                iconSize: [70, 20],
                iconAnchor: [35, 10],
              }),
              interactive: false,
            });
            this.villageLabels.push(label);
          });

          // 初始顯示
          if (this.showVillages) this.toggleVillageLayer();
        })
        .catch((err) => console.error('載入 villages.geojson 失敗', err));
    },

    toggleVillageLayer() {
      if (!this.villageLayer) return; // 圖層尚未載入
      if (this.showVillages) {
        this.villageLayer.addTo(this.map);
        this.villageLabels.forEach((l) => l.addTo(this.map));
      } else {
        this.map.removeLayer(this.villageLayer);
        this.villageLabels.forEach((l) => this.map.removeLayer(l));
      }
    },
    getPolygonCentroid(geometry) {
      const centroid = turf.centroid(geometry);
      const [lng, lat] = centroid.geometry.coordinates;
      return [lat, lng]; // Leaflet 需要 [lat, lng]
    },

    createMarker(latlng, { type, text, textColor, bgColor, fontSize }) {
      let icon;
      if (type === 'text') {
        icon = L.divIcon({
          className: 'custom-marker',
          html: `<div style="
            background-color:${bgColor};
            color:${textColor};
            font-size:${fontSize}px;
            padding:2px 5px;
            border-radius:3px;
            display:inline-block;
          ">${text}</div>`,
        });
      } else {
        icon = L.icon({
          iconUrl:
            'https://unpkg.com/leaflet@1.9.4/dist/images/marker-icon.png',
          iconSize: [25, 41],
          iconAnchor: [12, 41],
          popupAnchor: [1, -34],
          shadowUrl:
            'https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png',
          shadowSize: [41, 41],
        });
      }

      const marker = L.marker(latlng, { icon, draggable: true }).addTo(
        this.map
      );

      marker.customData = { type, text, textColor, bgColor, fontSize };

      marker.on('drag', (ev) => {
        const index = this.markers.indexOf(marker);
        if (index !== -1) {
          this.routePoints[index] = [ev.latlng.lat, ev.latlng.lng];
          if (this.routeLine) this.routeLine.setLatLngs(this.routePoints);
        }
        this.saveToLocalStorage();
      });

      marker.on('click', () => {
        this.selectedMarker = marker;
      });

      return marker;
    },

    updateMarkerIcon() {
      if (!this.selectedMarker) return;
      const { type, text, textColor, bgColor, fontSize } =
        this.selectedMarker.customData;

      let icon;
      if (type === 'text') {
        icon = L.divIcon({
          className: 'custom-marker',
          html: `<div style="
            background-color:${bgColor};
            color:${textColor};
            font-size:${fontSize}px;
            padding:2px 5px;
            border-radius:3px;
            display:inline-block;
          ">${text}</div>`,
        });
      } else {
        icon = L.icon({
          iconUrl:
            'https://unpkg.com/leaflet@1.9.4/dist/images/marker-icon.png',
          iconSize: [25, 41],
          iconAnchor: [12, 41],
          popupAnchor: [1, -34],
          shadowUrl:
            'https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png',
          shadowSize: [41, 41],
        });
      }

      this.selectedMarker.setIcon(icon);
      this.saveToLocalStorage();
    },

    undoLastPoint() {
      if (this.routePoints.length === 0) return;
      this.routePoints.pop();
      const marker = this.markers.pop();
      if (marker) this.map.removeLayer(marker);
      if (this.routeLine) {
        if (this.routePoints.length === 0) {
          this.map.removeLayer(this.routeLine);
          this.routeLine = null;
        } else {
          this.routeLine.setLatLngs(this.routePoints);
        }
      }
      this.saveToLocalStorage();
    },

    clearRoute() {
      this.routePoints = [];
      this.markers.forEach((m) => this.map.removeLayer(m));
      this.markers = [];
      if (this.routeLine) {
        this.map.removeLayer(this.routeLine);
        this.routeLine = null;
      }
      this.selectedMarker = null;
      localStorage.removeItem('mapData');
    },

    updateRouteColor() {
      if (this.routeLine) this.routeLine.setStyle({ color: this.routeColor });
      this.saveToLocalStorage();
    },

    saveToLocalStorage() {
      const data = {
        routePoints: this.routePoints,
        routeColor: this.routeColor,
        markers: this.markers.map((m) => ({
          latlng: m.getLatLng(),
          customData: m.customData,
        })),
      };
      localStorage.setItem('mapData', JSON.stringify(data));
    },

    loadFromLocalStorage() {
      const saved = localStorage.getItem('mapData');
      if (!saved) return;
      const { routePoints, routeColor, markers } = JSON.parse(saved);
      this.routePoints = routePoints;
      this.routeColor = routeColor;

      markers.forEach((m) => {
        const marker = this.createMarker(
          [m.latlng.lat, m.latlng.lng],
          m.customData
        );
        this.markers.push(marker);
      });

      if (routePoints.length > 0) {
        this.routeLine = L.polyline(routePoints, { color: routeColor }).addTo(
          this.map
        );
      }
    },
  },
};
</script>

<style>
.control-panel {
  position: absolute;
  top: 10px;
  right: 10px;
  width: 220px;
  background: white;
  padding: 10px;
  border-radius: 5px;
  z-index: 1000;
  font-size: 14px;
}
.control-panel input,
.control-panel select {
  width: 100%;
  margin-bottom: 5px;
}
.custom-marker div {
  white-space: nowrap;
  cursor: move;
  user-select: none;
}
.village-label {
  font-weight: bold;
  color: #333; /* 文字顏色 */
  background: rgba(40, 200, 200,.4); /* 半透明白底 */
  padding: 0px 2px;
  border-radius: 4px;
  text-align: center;
  white-space: nowrap;
  pointer-events: none; /* 不攔截滑鼠事件 */
}
</style>
