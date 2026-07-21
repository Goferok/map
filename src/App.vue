<template>
  <main class="map-page">
    <div ref="mapContainer" class="map-canvas"></div>

    <button class="settings-button" type="button" title="Загрузить JSON" @click="isSettingsModalOpen = true">
      ⚙
    </button>

    <form class="address-search" @submit.prevent="searchAddress">
      <input
        v-model="addressQuery"
        type="search"
        placeholder="Поиск адреса"
        autocomplete="off"
      />
      <button type="submit" :disabled="isAddressSearching">
        {{ isAddressSearching ? 'Ищем' : 'Найти' }}
      </button>
      <span v-if="addressSearchError">{{ addressSearchError }}</span>
    </form>

    <div
      v-if="hoverInfo"
      class="district-tooltip"
      :style="{ left: `${hoverInfo.x}px`, top: `${hoverInfo.y}px` }"
    >
      <strong>{{ hoverInfo.name }}</strong>
      <span v-if="!hoverInfo.worksEnabled" class="district-tooltip__danger">Не работаем в районе</span>
      <span>Коэффициент: {{ formatCoefficient(hoverInfo.productionCoefficient) }}</span>
      <span>Минимальная смета: {{ formatMoney(hoverInfo.minEstimate) }}</span>
    </div>

    <div v-if="isSettingsModalOpen" class="settings-modal-backdrop" @click.self="isSettingsModalOpen = false">
      <section class="settings-modal">
        <header>
          <div>
            <span>Настройки карты</span>
            <h2>Загрузка JSON</h2>
          </div>
          <button type="button" class="settings-modal-close" @click="isSettingsModalOpen = false">×</button>
        </header>

        <div v-if="loadError" class="alert">{{ loadError }}</div>
        <p>
          Загрузите файл `municipality-settings.json`, чтобы обновить коэффициенты, минимальные сметы и районы,
          где не работаем.
        </p>

        <label class="json-upload">
          <strong>Выбрать JSON</strong>
          <span>{{ importStatus || 'Файл не выбран' }}</span>
          <input type="file" accept="application/json,.json" @change="importSettings" />
        </label>
      </section>
    </div>
  </main>
</template>

<script setup>
import { onBeforeUnmount, onMounted, ref } from 'vue';
import maplibregl from 'maplibre-gl';

const MUNICIPALITIES_URL = `${import.meta.env.BASE_URL}data/spb_municipalities.geojson`;
const SETTINGS_URL = `${import.meta.env.BASE_URL}data/municipality-settings.json`;
const LOCAL_SETTINGS_KEY = 'district_coeff_map_settings';
const DEFAULT_COEFFICIENT = 1;
const DEFAULT_MIN_ESTIMATE = 300000;

const mapContainer = ref(null);
const loadError = ref('');
const isSettingsModalOpen = ref(false);
const importStatus = ref('');
const municipalities = ref(null);
const settings = ref({});
const hoverInfo = ref(null);
const addressQuery = ref('');
const addressSearchError = ref('');
const isAddressSearching = ref(false);

let map = null;
let hoveredId = null;
let addressMarker = null;

onMounted(async () => {
  await loadSettings();
  initMap();
});

onBeforeUnmount(() => {
  if (addressMarker) addressMarker.remove();
  if (map) map.remove();
});

function initMap() {
  map = new maplibregl.Map({
    container: mapContainer.value,
    center: [30.3159, 59.9391],
    zoom: 9.6,
    minZoom: 7,
    style: {
      version: 8,
      sources: {
        osm: {
          type: 'raster',
          tiles: ['https://tile.openstreetmap.org/{z}/{x}/{y}.png'],
          tileSize: 256,
          attribution: '© OpenStreetMap contributors'
        }
      },
      layers: [
        {
          id: 'osm',
          type: 'raster',
          source: 'osm'
        }
      ]
    }
  });

  map.addControl(new maplibregl.NavigationControl({ visualizePitch: false }), 'top-right');
  map.addControl(new maplibregl.FullscreenControl(), 'top-right');
  map.addControl(new maplibregl.ScaleControl({ maxWidth: 120, unit: 'metric' }), 'bottom-left');

  map.on('load', loadMunicipalities);
}

async function loadMunicipalities() {
  try {
    const response = await fetch(MUNICIPALITIES_URL);
    if (!response.ok) throw new Error(`GeoJSON не найден: ${response.status}`);
    const geojson = await response.json();
    municipalities.value = ensureFeatureIds(geojson);
    ensureDefaultSettings();

    map.addSource('municipalities', {
      type: 'geojson',
      data: buildMapGeojson(),
      promoteId: '_mapId'
    });

    map.addLayer({
      id: 'municipality-fills',
      type: 'fill',
      source: 'municipalities',
      paint: {
        'fill-color': ['coalesce', ['get', '_fillColor'], 'rgba(47, 128, 237, 0.04)'],
        'fill-opacity': [
          'case',
          ['boolean', ['feature-state', 'hover'], false],
          0.72,
          ['coalesce', ['get', '_fillOpacity'], 0.46]
        ]
      }
    });

    map.addLayer({
      id: 'municipality-lines',
      type: 'line',
      source: 'municipalities',
      paint: {
        'line-color': '#111827',
        'line-width': [
          'case',
          ['boolean', ['feature-state', 'hover'], false],
          1.4,
          0.7
        ],
        'line-opacity': 0.86
      }
    });

    map.on('mousemove', handleMapMouseMove);
    map.on('mouseleave', clearMunicipalityHover);
  } catch (error) {
    loadError.value = 'Файл с границами районов не найден. Положите GeoJSON в public/data/spb_municipalities.geojson';
    console.warn('[district-coeff-map] failed to load municipalities', error);
  }
}

async function loadSettings() {
  const sharedSettings = await loadSharedSettings();
  const localSettings = loadLocalSettings();
  settings.value = {
    ...sharedSettings,
    ...localSettings
  };
}

async function loadSharedSettings() {
  try {
    const response = await fetch(SETTINGS_URL, { cache: 'no-store' });
    if (!response.ok) return {};
    return normalizeSettings(await response.json());
  } catch (error) {
    console.warn('[district-coeff-map] failed to load shared settings', error);
    return {};
  }
}

function loadLocalSettings() {
  try {
    return normalizeSettings(JSON.parse(localStorage.getItem(LOCAL_SETTINGS_KEY) || '{}'));
  } catch {
    return {};
  }
}

function ensureFeatureIds(geojson) {
  return {
    ...geojson,
    features: (geojson.features || []).map((feature, index) => {
      const properties = feature.properties || {};
      const id = getMunicipalityId(feature, index);
      return {
        ...feature,
        id,
        properties: {
          ...properties,
          _mapId: id
        }
      };
    })
  };
}

function ensureDefaultSettings() {
  const next = { ...settings.value };
  let changed = false;
  (municipalities.value?.features || []).forEach((feature) => {
    const id = feature.properties._mapId;
    if (next[id]) return;
    next[id] = getDefaultSettings(getMunicipalityName(feature.properties));
    changed = true;
  });
  if (changed) {
    settings.value = next;
    persistSettings();
  }
}

function buildMapGeojson() {
  return {
    ...municipalities.value,
    features: (municipalities.value?.features || []).map((feature) => {
      const id = feature.properties._mapId;
      const current = settings.value[id] || getDefaultSettings(getMunicipalityName(feature.properties));
      const fill = getFillStyle(current);
      return {
        ...feature,
        properties: {
          ...feature.properties,
          productionCoefficient: current.productionCoefficient,
          minEstimate: current.minEstimate,
          worksEnabled: current.worksEnabled !== false,
          _fillColor: fill.color,
          _fillOpacity: fill.opacity
        }
      };
    })
  };
}

function syncMapData() {
  if (!map?.getSource('municipalities') || !municipalities.value) return;
  map.getSource('municipalities').setData(buildMapGeojson());
}

function getFillStyle(current) {
  if (current.worksEnabled === false) return { color: '#dc2626', opacity: 0.44 };
  return { color: getCoefficientColor(Number(current.productionCoefficient || 0)), opacity: 0.55 };
}

function getCoefficientColor(value) {
  if (value >= 1.3) return '#f97316';
  if (value >= 1.25) return '#facc15';
  if (value >= 1.2) return '#84cc16';
  if (value >= 1.15) return '#22c55e';
  if (value >= 1.1) return '#2dd4bf';
  if (value >= 1.05) return '#38bdf8';
  return '#d9d978';
}

function handleMapMouseMove(event) {
  if (!map?.getLayer('municipality-fills')) return;
  const features = map.queryRenderedFeatures(event.point, { layers: ['municipality-fills'] });
  const feature = features[0];
  if (!feature) {
    clearMunicipalityHover();
    return;
  }

  const props = feature.properties || {};
  const id = props._mapId;
  setHoveredId(id);
  map.getCanvas().style.cursor = 'pointer';
  hoverInfo.value = {
    x: Math.min(event.point.x + 18, window.innerWidth - 260),
    y: Math.min(event.point.y + 18, window.innerHeight - 130),
    name: getMunicipalityName(props),
    worksEnabled: props.worksEnabled !== false && props.worksEnabled !== 'false',
    productionCoefficient: props.productionCoefficient,
    minEstimate: props.minEstimate
  };
}

function clearMunicipalityHover() {
  map.getCanvas().style.cursor = '';
  setHoveredId(null);
  hoverInfo.value = null;
}

function setHoveredId(id) {
  if (hoveredId !== null && map?.getSource('municipalities')) {
    map.setFeatureState({ source: 'municipalities', id: hoveredId }, { hover: false });
  }
  hoveredId = id;
  if (hoveredId !== null && map?.getSource('municipalities')) {
    map.setFeatureState({ source: 'municipalities', id: hoveredId }, { hover: true });
  }
}

async function searchAddress() {
  const query = addressQuery.value.trim();
  if (!query || isAddressSearching.value) return;

  isAddressSearching.value = true;
  addressSearchError.value = '';

  try {
    const searchQuery = withCityHint(query);
    const url = new URL('https://nominatim.openstreetmap.org/search');
    url.searchParams.set('format', 'jsonv2');
    url.searchParams.set('limit', '1');
    url.searchParams.set('countrycodes', 'ru');
    url.searchParams.set('addressdetails', '1');
    url.searchParams.set('q', searchQuery);

    const response = await fetch(url.toString(), {
      headers: {
        Accept: 'application/json'
      }
    });
    if (!response.ok) throw new Error(`Nominatim ${response.status}`);

    const results = await response.json();
    const result = results?.[0];
    if (!result) {
      addressSearchError.value = 'Адрес не найден';
      return;
    }

    const lng = Number(result.lon);
    const lat = Number(result.lat);
    if (!Number.isFinite(lng) || !Number.isFinite(lat)) {
      addressSearchError.value = 'Не удалось получить координаты';
      return;
    }

    showAddressMarker([lng, lat], result.display_name || query);
  } catch (error) {
    addressSearchError.value = 'Ошибка поиска адреса';
    console.warn('[district-coeff-map] failed to search address', error);
  } finally {
    isAddressSearching.value = false;
  }
}

function withCityHint(query) {
  const normalized = query.toLowerCase();
  if (normalized.includes('санкт-петербург') || normalized.includes('спб') || normalized.includes('ленинградская')) {
    return query;
  }
  return `${query}, Санкт-Петербург`;
}

function showAddressMarker(coordinates, label) {
  if (!map) return;
  if (addressMarker) addressMarker.remove();

  const markerElement = document.createElement('div');
  markerElement.className = 'address-marker';
  markerElement.title = label;

  addressMarker = new maplibregl.Marker({ element: markerElement, anchor: 'center' })
    .setLngLat(coordinates)
    .setPopup(new maplibregl.Popup({ offset: 18 }).setHTML(`<strong>${escapeHtml(label)}</strong>`))
    .addTo(map);

  map.flyTo({
    center: coordinates,
    zoom: Math.max(map.getZoom(), 14),
    speed: 1.2,
    curve: 1.2,
    essential: true
  });
}

async function importSettings(event) {
  const file = event.target.files?.[0];
  event.target.value = '';
  if (!file) return;
  try {
    const imported = normalizeSettings(JSON.parse(await file.text()));
    settings.value = {
      ...settings.value,
      ...imported
    };
    persistSettings();
    syncMapData();
    importStatus.value = `Загружено: ${Object.keys(imported).length} районов`;
  } catch (error) {
    loadError.value = 'Не удалось прочитать JSON с настройками районов.';
    console.warn('[district-coeff-map] failed to import settings', error);
  }
}

function normalizeSettings(payload) {
  if (!payload || typeof payload !== 'object') return {};
  const entries = Array.isArray(payload)
    ? payload.map((item) => [item?.id, item])
    : Object.entries(payload);
  return entries.reduce((acc, [id, value]) => {
    const cleanId = String(id || '').trim();
    if (!cleanId || !value || typeof value !== 'object') return acc;
    acc[cleanId] = {
      name: value.name || '',
      productionCoefficient: Number(value.productionCoefficient || value.coefficient || DEFAULT_COEFFICIENT),
      minEstimate: Number(value.minEstimate || value.minimumEstimate || DEFAULT_MIN_ESTIMATE),
      worksEnabled: value.worksEnabled !== false
    };
    return acc;
  }, {});
}

function persistSettings() {
  localStorage.setItem(LOCAL_SETTINGS_KEY, JSON.stringify(settings.value));
}

function getDefaultSettings(name = '') {
  return {
    name,
    productionCoefficient: DEFAULT_COEFFICIENT,
    minEstimate: DEFAULT_MIN_ESTIMATE,
    worksEnabled: true
  };
}

function getMunicipalityId(feature, index) {
  const properties = feature.properties || {};
  return String(properties.oktmo || properties.OKTMO || properties.id || properties.ID || feature.id || `municipality-${index + 1}`);
}

function getMunicipalityName(properties = {}) {
  return properties.name || properties.NAME || properties.title || properties.NAME_RU || properties.ADMIN_L5 || 'Без названия';
}

function formatMoney(value) {
  const amount = Number(value || 0);
  if (!Number.isFinite(amount) || amount <= 0) return 'не задана';
  return `${Math.round(amount).toLocaleString('ru-RU')} ₽`;
}

function formatCoefficient(value) {
  const amount = Number(value || DEFAULT_COEFFICIENT);
  return amount.toLocaleString('ru-RU', { minimumFractionDigits: 1, maximumFractionDigits: 2 });
}

function escapeHtml(value) {
  return String(value ?? '')
    .replaceAll('&', '&amp;')
    .replaceAll('<', '&lt;')
    .replaceAll('>', '&gt;')
    .replaceAll('"', '&quot;')
    .replaceAll("'", '&#039;');
}

</script>
