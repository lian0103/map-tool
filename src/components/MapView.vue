<template>
  <div id="map"></div>

  <!-- 右側控制面板 -->
  <div class="control-panel" :class="{ 'closed-mobile': !controlOpenMobile }">
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

      <!-- 資料集切換 -->
      <div style="margin-top:8px;">
        <label>資料集</label>
        <select v-model="currentDataset" @change="changeDataset">
          <option v-for="d in datasets" :key="d.id" :value="d.id">{{ d.label }}</option>
        </select>
      </div>

      <!-- 行政區下拉多選 -->
      <div style="margin-top:6px;">
        <label>選擇行政區（可多選）</label>
        <div class="district-list">
          <div class="district-item" v-for="d in districts" :key="d">
            <label>
              <input
                type="checkbox"
                :value="d"
                v-model="selectedDistricts"
                @change="createVillageLayer"
              />
              <span class="district-name">{{ d }}</span>
            </label>
          </div>
        </div>
      </div>
    </div>

    <button @click="undoLastPoint">撤銷最後一點</button>
    <button @click="clearRoute">清空路線</button>
  </div>

  <!-- 手機尺寸的切換按鈕 -->
  <button class="control-toggle" @click="toggleControl" aria-label="切換控制面板">☰</button>
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
      villageGeojson: null,
      villageLabels: [],
      districts: [],
      selectedDistricts: [],
      // dataset control
      datasets: [
        { id: 'newtaipei', label: '新北市', file: './newTaipeiCity_village.json' },
        { id: 'taoyuan', label: '桃園市', file: './taoyuan_village.json' },
      ],
      currentDataset: 'newtaipei',
      controlOpenMobile: true,
    };
  },
  mounted() {
    this.initMap();
    this.loadFromLocalStorage();
    this.loadVillageLayer(this.currentDataset);
  },
  methods: {
    toggleControl() {
      this.controlOpenMobile = !this.controlOpenMobile;
    },
    changeDataset() {
      // 移除現有圖層與標籤，清空選取，並載入新資料集
      if (this.villageLayer) {
        try {
          this.map.removeLayer(this.villageLayer);
        } catch (e) {}
        this.villageLayer = null;
      }
      if (this.villageLabels && this.villageLabels.length) {
        this.villageLabels.forEach((l) => {
          try {
            this.map.removeLayer(l);
          } catch (e) {}
        });
      }
      this.villageLabels = [];
      this.selectedDistricts = [];
      this.loadVillageLayer(this.currentDataset);
    },

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
    loadVillageLayer(datasetId) {
      const ds = this.datasets.find((d) => d.id === datasetId) || this.datasets[0];
      fetch(ds.file)
        .then((res) => res.json())
        .then((data) => {
          // 儲存原始 geojson
          this.villageGeojson = data;

          // Normalization: 將不同來源可能的屬性統一成 __district / __village
          data.features.forEach((f) => {
            const p = f.properties || {};
            p.__district = p.ADMIT || p.TOWNNAME || p.TOWN || p.COUNTY || p.CITY || p.COUNTYNAME || null;
            p.__village = p.ADMIV || p.VILLNAME || p.T_NAME || p.NAME || null;
            f.properties = p;
          });

          // 建立行政區清單，但先不加到地圖上
          const admits = new Set();
          data.features.forEach((f) => {
            if (f.properties && f.properties.__district) admits.add(f.properties.__district);
          });
          this.districts = Array.from(admits).sort();

          // 預設不選任何行政區
          this.selectedDistricts = [];

          // 建立初始圖層（依 selectedDistricts）
          this.createVillageLayer();
          if (this.showVillages) this.toggleVillageLayer();
        })
        .catch((err) => console.error('載入 geojson 失敗', ds.file, err));
    },

    createVillageLayer() {
      if (!this.villageGeojson) return;

      // 移除舊的圖層與標籤
      if (this.villageLayer) {
        try {
          this.map.removeLayer(this.villageLayer);
        } catch (e) {}
        this.villageLayer = null;
      }
      if (this.villageLabels && this.villageLabels.length) {
        this.villageLabels.forEach((l) => {
          try {
            this.map.removeLayer(l);
          } catch (e) {}
        });
      }
      this.villageLabels = [];

      // 篩選 features
      const filteredFeatures = this.villageGeojson.features.filter((f) =>
        this.selectedDistricts.includes(f.properties.__district)
      );

      const filtered = { ...this.villageGeojson, features: filteredFeatures };

      this.villageLayer = L.geoJSON(filtered, {
        style: {
          color: '#555',
          weight: 1,
          fillColor: '#ffffff',
          fillOpacity: 0.3,
        },
      });

      // 建立里名 label（只為篩選後的 features）
      filteredFeatures.forEach((feature) => {
        const latlng = this.getPolygonCentroid(feature.geometry);
        const label = L.marker(latlng, {
          icon: L.divIcon({
            className: 'village-label',
            html: feature.properties.__village || feature.properties.ADMIV || '',
            iconSize: [70, 20],
            iconAnchor: [35, 10],
          }),
          interactive: false,
        });
        this.villageLabels.push(label);
      });

      // 若目前設定為顯示，加入地圖
      if (this.showVillages) {
        this.villageLayer.addTo(this.map);
        this.villageLabels.forEach((l) => l.addTo(this.map));
      }
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
  watch: {
    selectedDistricts() {
      // 當選取行政區改變且圖層已顯示時，重新建立圖層
      if (this.showVillages) this.createVillageLayer();
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

/* 手機響應式 */
.control-toggle {
  display: none;
}

@media (max-width: 600px) {
  .control-panel {
    position: fixed;
    left: 10px;
    right: 10px;
    top: auto;
    bottom: 10px;
    width: auto;
    max-height: 50vh;
    overflow: auto;
    padding: 8px;
    font-size: 14px;
  }

  .control-panel.closed-mobile {
    display: none;
  }

  .control-toggle {
    display: block;
    position: fixed;
    right: 16px;
    bottom: 16px;
    z-index: 1100;
    background: #fff;
    border: 1px solid #ddd;
    border-radius: 6px;
    padding: 8px 10px;
    box-shadow: 0 2px 6px rgba(0,0,0,0.12);
    font-size: 18px;
  }

  /* 讓 select 在多選時更好操作 */
  .control-panel select[multiple] {
    height: 120px;
  }
}

.district-list {
  border: 1px solid #e6e6e6;
  border-radius: 4px;
  padding: 6px;
  max-height: 140px;
  overflow: auto;
  background: #fafafa;
}
.district-item {
  padding: 4px 2px;
}
.district-item label {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  width: 40px;
}
.district-name {
  flex: 1;
  white-space: nowrap;
}
</style>
