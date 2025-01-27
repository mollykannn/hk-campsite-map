<template>
  <div>
    <div class="search-container">
      <div class="search-input-wrapper">
        <input
          type="text"
          v-model="searchQuery"
          @input="handleSearch"
          placeholder="搜尋營地..."
          class="search-input"
        />
        <button v-if="searchQuery" @click="clearSearch" class="clear-button">
          <span class="clear-icon">×</span>
        </button>
      </div>
      <div v-if="searchResults.length > 0" class="search-results">
        <div
          v-for="result in searchResults"
          :key="result.properties.FACILITY_NAME_EN"
          class="search-result-item"
          @click="handleResultClick(result)"
        >
          <div class="result-name">
            {{ result.properties.FACILITY_NAME_TC }}
          </div>
          <div class="result-subtext">
            {{ result.properties.COUNTRY_PARK_TC }}
          </div>
        </div>
      </div>
    </div>
    <div ref="mapRef" class="map-container"></div>
    <SidePanel
      :is-open="isPanelOpen"
      :content="panelContent"
      @close="closePanel"
    />
    <div class="location-controls">
      <button
        @click="toggleLocationTracking"
        class="location-button"
        :class="{ active: isTrackingLocation }"
      >
        <span class="location-icon"></span>
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, watch } from "vue";
import "leaflet/dist/leaflet.css";
import L from "leaflet";
import campsiteData from "../assets/json/campsite.json";
import weatherStationData from "../assets/json/weather_station.json";
import windData from "../assets/json/wind.json";
import proj4 from "proj4";
import SidePanel from "../components/SidePanel.vue";
import { useRoute, useRouter } from "vue-router";
import markericon2x from "leaflet/dist/images/marker-icon-2x.png";
import markericon from "leaflet/dist/images/marker-icon.png";
import markershadow from "leaflet/dist/images/marker-shadow.png";
import campsite_gemini from "../assets/img/campsite_gemini.jpeg";
// 定義 EPSG:2326 的投影參數
const EPSG2326 =
  "+proj=tmerc +lat_0=22.31213333333334 +lon_0=114.1785555555556 +k=1 +x_0=836694.05 +y_0=819069.8 +ellps=intl +towgs84=-162.619,-276.959,-161.764,0.067753,-2.24365,-1.15883,-1.09425 +units=m +no_defs";
const WGS84 = "+proj=longlat +datum=WGS84 +no_defs";

const mapRef = ref(null);
const map = ref(null);
const markers = ref([]);
const weatherStations = ref([]);
const windStations = ref([]);
const searchQuery = ref("");
const searchResults = ref([]);
const isPanelOpen = ref(false);
const panelContent = ref("");
const isTrackingLocation = ref(false);
const watchPositionId = ref(null);
const userMarker = ref(null);
const userAccuracyCircle = ref(null);

const route = useRoute();
const router = useRouter();

// 反向映射表
const reverseCampsiteUrlMap = campsiteData.features.reduce((acc, feature) => {
  acc[feature.properties.FACILITY_NAME_EN.replace(/ /g, "")] =
    feature.properties.FACILITY_NAME_TC;
  return acc;
}, {});

// 獲取營地圖片
const getCampsiteImage = (campsiteName) => {
  const imageMap = {
    // ... 其他營地的圖片映射
  };
  return imageMap[campsiteName] || campsite_gemini;
};

// 計算兩點之間的距離
const calculateDistance = (lat1, lon1, lat2, lon2) => {
  const R = 6371;
  const dLat = ((lat2 - lat1) * Math.PI) / 180;
  const dLon = ((lon2 - lon1) * Math.PI) / 180;
  const a =
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos((lat1 * Math.PI) / 180) *
      Math.cos((lat2 * Math.PI) / 180) *
      Math.sin(dLon / 2) *
      Math.sin(dLon / 2);
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  return R * c;
};

// 找到最近的氣象站
const findNearestWeatherStation = (lat, lng) => {
  let nearestStation = null;
  let minDistance = Infinity;

  weatherStations.value.forEach((station) => {
    const distance = calculateDistance(
      lat,
      lng,
      station.geometry.coordinates[1],
      station.geometry.coordinates[0]
    );
    if (distance < minDistance) {
      minDistance = distance;
      nearestStation = station;
    }
  });

  return nearestStation;
};

// 找到最近的風速監測站
const findNearestWindStation = (lat, lng) => {
  let nearestStation = null;
  let minDistance = Infinity;

  windStations.value.forEach((station) => {
    const distance = calculateDistance(
      lat,
      lng,
      station.geometry.coordinates[1],
      station.geometry.coordinates[0]
    );
    if (distance < minDistance) {
      minDistance = distance;
      nearestStation = station;
    }
  });
  return nearestStation;
};

// 獲取風速資料
const fetchWindData = async (url) => {
  try {
    const proxyUrl = `https://cors-proxy.fringe.zone/${url}`;
    const response = await fetch(proxyUrl);
    const csvText = await response.text();
    const lines = csvText.split("\n");

    const headers = lines[0].split(",");

    const directionIndex = headers.findIndex((h) =>
      h.includes("十分鐘平均風向（方位點）")
    );
    const speedIndex = headers.findIndex((h) =>
      h.includes("十分鐘平均風速（公里/小時）")
    );
    const maxGustIndex = headers.findIndex((h) =>
      h.includes("十分鐘最高陣風風速（公里/小時）")
    );

    const dataLines = lines.slice(1).filter((line) => line.trim());
    const lastLine = dataLines[dataLines.length - 1].split(",");

    return {
      direction: lastLine[directionIndex]?.trim(),
      speed: parseFloat(lastLine[speedIndex]),
      max_gust: parseFloat(lastLine[maxGustIndex]),
    };
  } catch (error) {
    console.error("Error fetching wind data:", error);
    return null;
  }
};

// 創建資訊視窗內容
const createInfoWindowContent = async (properties, nearestStation) => {
  const temperature = nearestStation.temperature;
  const rainfall = nearestStation.rainfall;

  const nearestWindStation = findNearestWindStation(
    nearestStation.geometry.coordinates[1],
    nearestStation.geometry.coordinates[0]
  );

  let windData = null;
  if (nearestWindStation) {
    windData = await fetchWindData(nearestWindStation.properties.Data_url);
  }

  const tempInfo =
    temperature !== null
      ? `<p><strong>溫度:</strong> ${temperature}°C</p>`
      : "";

  const rainfallInfo =
    rainfall !== null ? `<p><strong>雨量:</strong> ${rainfall}mm</p>` : "";

  let windInfo = "<p>無風速資料</p>";
  if (windData) {
    windInfo = `
      <p><strong>最近風速監測站:</strong> ${
        nearestWindStation.properties.AutomaticWeatherStation_uc
      }</p>
      <p><strong>十分鐘平均風向:</strong> ${windData.direction || "N/A"}</p>
      <p><strong>十分鐘平均風速:</strong> ${
        windData.speed?.toFixed(1) || "N/A"
      } km/h</p>
      <p><strong>十分鐘最大陣風:</strong> ${
        windData.max_gust?.toFixed(1) || "N/A"
      } km/h</p>
    `;
  }

  return `
    <div class="info-window">
      <div class="campsite-image">
        <img 
          src="${getCampsiteImage(properties.FACILITY_NAME_EN)}" 
          alt="${properties.FACILITY_NAME_TC}"
        />
      </div>
      <h3>${properties.FACILITY_NAME_TC}</h3>
      <div class="campsite-info">
        <p><strong>營地數目:</strong> ${properties.TENT_SPACE_TC}</p>
        <p><strong>水源:</strong> ${properties.SOURCE_OF_WATER_TC}</p>
        <div class="weather-info">
          <p><strong>最近氣象站:</strong> ${
            nearestStation.properties.weather_station_tc
          }</p>
          ${tempInfo}
          ${rainfallInfo}
          ${windInfo}
        </div>
      </div>
    </div>
  `;
};

// 座標轉換
const convertEPSG2326ToLatLng = (x, y) => {
  const [lng, lat] = proj4(EPSG2326, WGS84, [x, y]);
  return { lat, lng };
};

// 搜尋處理函數
const handleSearch = () => {
  if (!searchQuery.value.trim()) {
    searchResults.value = [];
    return;
  }

  const query = searchQuery.value.toLowerCase();
  searchResults.value = campsiteData.features.filter(
    (feature) =>
      feature.properties.FACILITY_NAME_TC.toLowerCase().includes(query) ||
      feature.properties.FACILITY_NAME_EN.toLowerCase().includes(query) ||
      feature.properties.COUNTRY_PARK_TC.toLowerCase().includes(query) ||
      feature.properties.COUNTRY_PARK_EN.toLowerCase().includes(query)
  );
};

// 處理搜尋結果點擊
const handleResultClick = (result) => {
  const [x, y] = result.geometry.coordinates;
  const position = convertEPSG2326ToLatLng(x, y);

  // 移動地圖到選中的營地
  map.value.setView([position.lat, position.lng], 15);

  // 找到對應的標記
  const marker = markers.value.find(
    (m) =>
      m.getLatLng().lat === position.lat && m.getLatLng().lng === position.lng
  );

  if (marker) {
    const nearestStation = findNearestWeatherStation(
      position.lat,
      position.lng
    );
    markerClickHandler(marker, result, nearestStation);
  }

  // 更新 URL
  const urlId = Object.keys(reverseCampsiteUrlMap).find(
    (key) => reverseCampsiteUrlMap[key] === result.properties.FACILITY_NAME_TC
  );
  if (urlId) {
    router.push({
      query: { place: urlId },
    });
  }

  // 清空搜尋
  searchQuery.value = "";
  searchResults.value = [];
};

// 清空搜尋
const clearSearch = () => {
  searchQuery.value = "";
  searchResults.value = [];
};

// 關閉面板
const closePanel = () => {
  isPanelOpen.value = false;
};

// 標記點擊處理
const markerClickHandler = async (
  marker,
  feature,
  nearestStation,
  updateUrl = true
) => {
  if (updateUrl) {
    const urlId = reverseCampsiteUrlMap[feature.properties.FACILITY_NAME_TC];
    if (urlId && route.query.place !== urlId) {
      router.push({
        query: { place: urlId },
      });
    }
  }

  const content = await createInfoWindowContent(
    feature.properties,
    nearestStation
  );
  panelContent.value = content;
  isPanelOpen.value = true;
};

// 添加獲取天氣數據的函數
const fetchWeatherData = async () => {
  try {
    const response = await fetch(
      "https://data.weather.gov.hk/weatherAPI/opendata/weather.php?dataType=rhrread&lang=tc"
    );
    const data = await response.json();

    // 轉換 API 數據格式以匹配原有的 weatherData 格式
    return data.temperature.data.map((station) => ({
      station_name_tc: station.place,
      temperature: station.value,
      rainfall:
        data.rainfall.data.find((r) => r.place === station.place)?.max ?? 0,
    }));
  } catch (error) {
    console.error("Error fetching weather data:", error);
    return [];
  }
};

// 合併天氣站和天氣數據的函數
const mergeWeatherData = (stations, weather) => {
  return stations.features.map((station) => {
    // 找到對應的天氣數據
    const weatherInfo = weather.find(
      (w) => w.station_name_tc === station.properties.weather_station_tc
    );

    return {
      ...station,
      temperature: weatherInfo?.temperature ?? null,
      rainfall: weatherInfo?.rainfall ?? null,
    };
  });
};

// 初始化地圖
const initMap = () => {
  // 創建地圖實例
  L.Marker.prototype.options.icon = L.icon({
    iconRetinaUrl: markericon2x,
    iconUrl: markericon,
    shadowUrl: markershadow,
    iconSize: [25, 41],
    iconAnchor: [12, 41],
    popupAnchor: [1, -34],
    tooltipAnchor: [16, -28],
    shadowSize: [41, 41],
  });

  map.value = L.map(mapRef.value, {}).setView([22.3193, 114.1694], 11);

  // 添加 OpenStreetMap 圖層
  L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
    attribution: "© OpenStreetMap contributors",
    maxZoom: 19,
  }).addTo(map.value);

  // 添加營地標記
  campsiteData.features.forEach((feature) => {
    const [x, y] = feature.geometry.coordinates;
    const position = convertEPSG2326ToLatLng(x, y);

    const nearestStation = findNearestWeatherStation(
      position.lat,
      position.lng
    );

    const marker = L.marker([position.lat, position.lng], {
      title: feature.properties.FACILITY_NAME_EN,
    }).addTo(map.value);

    marker.on("click", () => {
      markerClickHandler(marker, feature, nearestStation);
    });

    markers.value.push(marker);
  });
};

// 處理位置更新
const handleLocationUpdate = (position) => {
  const pos = {
    lat: position.coords.latitude,
    lng: position.coords.longitude,
  };

  if (!userMarker.value) {
    userMarker.value = L.circleMarker([pos.lat, pos.lng], {
      radius: 8,
      fillColor: "#4285F4",
      fillOpacity: 1,
      color: "#FFFFFF",
      weight: 2,
    }).addTo(map.value);

    userAccuracyCircle.value = L.circle([pos.lat, pos.lng], {
      radius: position.coords.accuracy,
      fillColor: "#4285F4",
      fillOpacity: 0.15,
      color: "#4285F4",
      weight: 1,
    }).addTo(map.value);
  } else {
    userMarker.value.setLatLng([pos.lat, pos.lng]);
    userAccuracyCircle.value.setLatLng([pos.lat, pos.lng]);
    userAccuracyCircle.value.setRadius(position.coords.accuracy);
  }

  if (isTrackingLocation.value) {
    map.value.panTo([pos.lat, pos.lng]);
  }
};

// 開始位置追蹤
const startLocationTracking = () => {
  if ("geolocation" in navigator) {
    // 獲取一次位置並將地圖中心設為用戶位置
    navigator.geolocation.getCurrentPosition(
      (position) => {
        handleLocationUpdate(position);
        map.value.panTo([position.coords.latitude, position.coords.longitude]);
        map.value.setZoom(15);
      },
      (error) => {
        console.error("Error getting location:", error);
        alert("無法獲取您的位置");
        isTrackingLocation.value = false;
      }
    );

    // 持續追蹤位置
    watchPositionId.value = navigator.geolocation.watchPosition(
      handleLocationUpdate,
      (error) => {
        console.error("Error tracking location:", error);
        isTrackingLocation.value = false;
      },
      {
        enableHighAccuracy: true,
        timeout: 5000,
        maximumAge: 0,
      }
    );
  } else {
    alert("您的瀏覽器不支援位置服務");
    isTrackingLocation.value = false;
  }
};

// 停止位置追蹤
const stopLocationTracking = () => {
  if (watchPositionId.value) {
    navigator.geolocation.clearWatch(watchPositionId.value);
    watchPositionId.value = null;
  }

  // 移除位置標記和精確度圓圈
  if (userMarker.value) {
    userMarker.value.remove();
    userMarker.value = null;
  }
  if (userAccuracyCircle.value) {
    userAccuracyCircle.value.remove();
    userAccuracyCircle.value = null;
  }
};

// 切換位置追蹤
const toggleLocationTracking = () => {
  isTrackingLocation.value = !isTrackingLocation.value;
  if (isTrackingLocation.value) {
    startLocationTracking();
  } else {
    stopLocationTracking();
  }
};

// 生命週期鉤子
onMounted(async () => {
  try {
    // 獲取即時天氣數據並合併
    const weatherData = await fetchWeatherData();
    weatherStations.value = mergeWeatherData(weatherStationData, weatherData);
    windStations.value = windData.features;
    initMap();

    // 監聽路由變化
    watch(
      () => route.query.place,
      (newPlace) => {
        if (newPlace && map.value) {
          const campsite = campsiteData.features.find(
            (feature) =>
              feature.properties.FACILITY_NAME_TC ===
              reverseCampsiteUrlMap[newPlace]
          );

          if (campsite) {
            const position = convertEPSG2326ToLatLng(
              campsite.geometry.coordinates[0],
              campsite.geometry.coordinates[1]
            );

            map.value.setView([position.lat, position.lng], 15);

            const marker = markers.value.find(
              (m) =>
                m.getElement()?.title === campsite.properties.FACILITY_NAME_EN
            );

            if (marker) {
              const nearestStation = findNearestWeatherStation(
                position.lat,
                position.lng
              );
              markerClickHandler(marker, campsite, nearestStation, false);
            }
          }
        }
      },
      { immediate: true }
    );
  } catch (error) {
    console.error("Error initializing map:", error);
  }
});

onBeforeUnmount(() => {
  if (map.value) {
    markers.value.forEach((marker) => map.value.removeLayer(marker));
    markers.value = [];
    stopLocationTracking();
    map.value.remove();
  }
});
</script>

<style scoped>
.map-container {
  height: 100vh;
  width: 100%;
}

/* 確保 Leaflet 控制項正確顯示 */
:deep(.leaflet-control-container) {
  position: absolute;
  z-index: 1000;
}

.search-container {
  position: absolute;
  top: 10px;
  left: 55px;
  z-index: 1000;
  width: min(400px, 70%);
}

.search-input-wrapper {
  position: relative;
  width: 100%;
}

.search-input {
  width: 100%;
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 8px;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
  font-size: 16px;
  background: white;
  color: #000;
}

.clear-button {
  position: absolute;
  right: -10px;
  top: 47%;
  transform: translateY(-50%);
  border: none;
  background: transparent;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
}

.clear-icon {
  font-size: 20px;
  color: #666;
  line-height: 1;
}

.clear-button:hover .clear-icon {
  color: #333;
}

.search-results {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  background: white;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  margin-top: 8px;
  max-height: 60vh;
  overflow-y: auto;
}

.search-result-item {
  padding: 12px;
  cursor: pointer;
  border-bottom: 1px solid #eee;
}

.search-result-item:last-child {
  border-bottom: none;
}

.search-result-item:hover {
  background-color: #f5f5f5;
}

.result-name {
  font-weight: 500;
  color: #333;
}

.result-subtext {
  font-size: 14px;
  color: #666;
  margin-top: 4px;
}

/* 手機版樣式調整 */
@media (max-width: 480px) {
  .search-input {
    font-size: 14px;
  }

  .clear-icon {
    font-size: 18px;
  }

  .search-result-item {
    padding: 10px;
  }
}

:deep(.info-window) {
  padding: 20px;
}

:deep(.info-window h3) {
  margin: 0 0 15px 0;
  font-size: 1.5em;
  color: #333;
}

:deep(.info-window p) {
  margin: 8px 0;
  line-height: 1.4;
  color: #666;
}

:deep(.weather-info) {
  margin-top: 15px;
  padding-top: 15px;
  border-top: 1px solid #eee;
}

:deep(.campsite-image) {
  width: calc(100% + 40px);
  height: 280px;
  overflow: hidden;
  margin-bottom: 15px;
  background-size: cover;
  margin: -20px;
  margin-bottom: 0px;
}

:deep(.campsite-image img) {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

:deep(.info-window) {
  padding: 0;
}

:deep(.campsite-info) {
  padding: 15px;
}

:deep(.info-window h3) {
  padding: 0 15px;
  margin: 15px 0;
  font-size: 1.5em;
  color: #333;
}

/* 手機版樣式調整 */
@media (max-width: 480px) {
  :deep(.campsite-image) {
    height: 150px;
  }
}

.location-controls {
  position: absolute;
  right: 10px;
  bottom: 10px;
  z-index: 1000;
}

.location-button {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: white;
  border: none;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.3);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
}

.location-button.active {
  background: #4285f4;
}

.location-icon {
  width: 18px;
  height: 18px;
  background: currentColor;
  mask: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24'%3E%3Cpath d='M12 8c-2.21 0-4 1.79-4 4s1.79 4 4 4 4-1.79 4-4-1.79-4-4-4zm8.94 3A8.994 8.994 0 0 0 13 3.06V1h-2v2.06A8.994 8.994 0 0 0 3.06 11H1v2h2.06A8.994 8.994 0 0 0 11 20.94V23h2v-2.06A8.994 8.994 0 0 0 20.94 13H23v-2h-2.06zM12 19c-3.87 0-7-3.13-7-7s3.13-7 7-7 7 3.13 7 7-3.13 7-7 7z'/%3E%3C/svg%3E")
    no-repeat center;
  -webkit-mask: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24'%3E%3Cpath d='M12 8c-2.21 0-4 1.79-4 4s1.79 4 4 4 4-1.79 4-4-1.79-4-4-4zm8.94 3A8.994 8.994 0 0 0 13 3.06V1h-2v2.06A8.994 8.994 0 0 0 3.06 11H1v2h2.06A8.994 8.994 0 0 0 11 20.94V23h2v-2.06A8.994 8.994 0 0 0 20.94 13H23v-2h-2.06zM12 19c-3.87 0-7-3.13-7-7s3.13-7 7-7 7 3.13 7 7-3.13 7-7 7z'/%3E%3C/svg%3E")
    no-repeat center;
}

.location-button:not(.active) .location-icon {
  color: #666;
}

.location-button.active .location-icon {
  color: white;
}

@media (max-width: 480px) {
  .location-controls {
    bottom: 80px;
    right: 8px;
  }

  .location-button {
    width: 36px;
    height: 36px;
  }

  .location-icon {
    width: 16px;
    height: 16px;
  }
}
</style>
