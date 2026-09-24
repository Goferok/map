<template>
  <main class="map-page">
    <div ref="mapContainer" class="map-canvas"></div>

    <button class="settings-button" type="button" title="Загрузить JSON" @click="isSettingsModalOpen = true">
      ⚙
    </button>

    <div class="map-tools">
      <nav class="region-switcher" aria-label="Выбор региона">
        <button
          v-for="region in regions"
          :key="region.id"
          type="button"
          :class="{ 'region-switcher__button--active': activeRegion === region.id }"
          @click="selectRegion(region.id)"
        >
          {{ region.label }}
        </button>
      </nav>

      <button
        class="edit-mode-button"
        :class="{ 'edit-mode-button--active': isEditMode }"
        type="button"
        @click="toggleEditMode"
      >
        {{ isEditMode ? 'Готово' : 'Редактировать' }}
      </button>
    </div>

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

    <aside v-if="isEditMode" class="editor-panel">
      <header>
        <span>Режим редактирования</span>
        <strong>{{ selectedMunicipality?.name || 'Выберите район на карте' }}</strong>
      </header>

      <template v-if="selectedMunicipality && selectedDistrictSettings">
        <label>
          <span>КЭФ</span>
          <select
            :value="selectedDistrictSettings.productionCoefficient"
            @change="updateSelectedSettings('productionCoefficient', $event.target.value)"
          >
            <option v-for="coefficient in coefficientOptions" :key="coefficient" :value="coefficient">
              {{ formatCoefficient(coefficient) }}
            </option>
          </select>
        </label>
        <label>
          <span>Минимальная смета, ₽</span>
          <input
            type="number"
            min="0"
            step="10000"
            :value="selectedDistrictSettings.minEstimate"
            @input="updateSelectedSettings('minEstimate', $event.target.value)"
          />
        </label>
        <label class="editor-panel__checkbox">
          <input
            type="checkbox"
            :checked="selectedDistrictSettings.worksEnabled !== false"
            @change="updateSelectedSettings('worksEnabled', $event.target.checked)"
          />
          <span>Работаем в районе</span>
        </label>
      </template>

      <button type="button" class="editor-panel__export" @click="exportSettings">Скачать JSON с настройками</button>
    </aside>

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

    <div v-if="legendItems.length" class="legend">
      <strong>КЭФ по районам</strong>
      <div class="legend-items">
        <span v-for="item in legendItems" :key="item.key">
          <i :style="{ background: item.color }"></i>
          {{ item.label }}
        </span>
      </div>
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
          Загрузите файл `municipality-map-data.json`, чтобы обновить геометрию, коэффициенты, минимальные сметы и районы,
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
import { computed, onBeforeUnmount, onMounted, ref } from 'vue';
import maplibregl from 'maplibre-gl';

const MUNICIPALITY_MAP_DATA_URL = `${import.meta.env.BASE_URL}data/municipality-map-data.json`;
const MOSCOW_BOUNDARIES_URL = `${import.meta.env.BASE_URL}data/moscow-boundaries.geojson`;
const DEFAULT_SETTINGS_URL = `${import.meta.env.BASE_URL}data/default-municipality-settings.json`;
const DEFAULT_COEFFICIENT = 1;
const DEFAULT_MIN_ESTIMATE = 300000;
const EDITOR_STORAGE_KEY = 'district-coeff-map-settings-v1';
const coefficientOptions = Array.from({ length: 11 }, (_, index) => Number((1 + index * 0.05).toFixed(2)));
const regions = [
  { id: 'spb', label: 'СПб', bounds: [[29.4, 59.5], [31.4, 60.35]] },
  { id: 'msk', label: 'Москва и МО', bounds: [[35.0, 54.8], [40.2, 57.1]] }
];

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
const activeRegion = ref('spb');
const isEditMode = ref(false);
const selectedMunicipality = ref(null);

let map = null;
let hoveredId = null;
let addressMarker = null;

const selectedDistrictSettings = computed(() => {
  if (!selectedMunicipality.value) return null;
  return settings.value[selectedMunicipality.value.id] || null;
});

const legendItems = computed(() => {
  const values = new Map();
  let hasDisabledDistricts = false;

  (municipalities.value?.features || []).forEach((feature) => {
    const id = feature.properties?._mapId;
    const current = settings.value[id];
    if (!current) return;

    if (current.worksEnabled === false) {
      hasDisabledDistricts = true;
      return;
    }

    const coefficient = Number(current.productionCoefficient || DEFAULT_COEFFICIENT);
    if (!Number.isFinite(coefficient)) return;
    const key = coefficient.toFixed(2);
    values.set(key, {
      key,
      value: coefficient,
      label: formatCoefficient(coefficient),
      color: getCoefficientColor(coefficient)
    });
  });

  const items = Array.from(values.values()).sort((a, b) => a.value - b.value);
  if (hasDisabledDistricts) {
    items.unshift({
      key: 'disabled',
      value: -1,
      label: 'Не работаем',
      color: '#dc2626'
    });
  }
  return items;
});
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
    if (!municipalities.value) {
      throw new Error('municipality-map-data.json не содержит geojson');
    }
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
    map.on('click', 'municipality-fills', selectMunicipalityForEdit);
  } catch (error) {
    loadError.value = 'Файл data/municipality-map-data.json не найден или содержит неверный формат.';
    console.warn('[district-coeff-map] failed to load municipalities', error);
  }
}

async function loadSettings() {
  const mapData = await loadSharedMapData();
  settings.value = {
    ...mapData.settings,
    ...loadSavedSettings()
  };
}

async function loadSharedMapData() {
  try {
    const [response, moscowGeojson, defaultSettings] = await Promise.all([
      fetch(MUNICIPALITY_MAP_DATA_URL, { cache: 'no-store' }),
      loadMoscowBoundaries(),
      loadDefaultSettings()
    ]);
    if (!response.ok) throw new Error(`municipality-map-data.json не найден: ${response.status}`);
    const payload = await response.json();
    if (payload?.geojson?.type !== 'FeatureCollection') {
      throw new Error('municipality-map-data.json не содержит geojson FeatureCollection');
    }
    municipalities.value = ensureFeatureIds({
      type: 'FeatureCollection',
      features: [
        ...(payload.geojson.features || []),
        ...moscowGeojson.features
      ]
    });
    return {
      settings: normalizeSettings({
        ...(payload?.settings || {}),
        ...defaultSettings
      })
    };
  } catch (error) {
    loadError.value = 'Файл data/municipality-map-data.json не найден или содержит неверный формат.';
    console.warn('[district-coeff-map] failed to load municipality map data', error);
    return { settings: {} };
  }
}

async function loadDefaultSettings() {
  const response = await fetch(DEFAULT_SETTINGS_URL, { cache: 'no-store' });
  if (!response.ok) throw new Error(`${DEFAULT_SETTINGS_URL} не найден: ${response.status}`);
  const payload = await response.json();
  return payload?.settings || payload || {};
}

async function loadMoscowBoundaries() {
  const response = await fetch(MOSCOW_BOUNDARIES_URL, { cache: 'no-store' });
  if (!response.ok) throw new Error(`${MOSCOW_BOUNDARIES_URL} не найден: ${response.status}`);
  const geojson = await response.json();
  if (geojson?.type !== 'FeatureCollection') throw new Error(`${MOSCOW_BOUNDARIES_URL} содержит неверный формат`);
  return {
    ...geojson,
    features: prepareMoscowBoundaryParts(geojson.features || [])
  };
}

function prepareMoscowBoundaryParts(features) {
  const totalParts = new Map();
  features.forEach((feature) => {
    const osmId = String(feature.properties?.OSM_ID || feature.id || '');
    if (osmId) totalParts.set(osmId, (totalParts.get(osmId) || 0) + 1);
  });

  const partNumbers = new Map();
  return features.flatMap((feature) => {
    const osmId = String(feature.properties?.OSM_ID || feature.id || '');
    if (!osmId || !feature.geometry) return [];
    const region = getMoscowBoundaryRegion(osmId, feature.properties);
    const parentMapId = `${region}-${osmId}`;
    const partNumber = (partNumbers.get(osmId) || 0) + 1;
    partNumbers.set(osmId, partNumber);
    const sourceCoordinates = feature.geometry.type === 'Polygon'
      ? [feature.geometry.coordinates]
      : feature.geometry.coordinates;
    const coordinates = convertWebMercatorToLngLat(sourceCoordinates);

    const mapId = totalParts.get(osmId) > 1
      ? `${parentMapId}-part-${partNumber}`
      : parentMapId;
    return [{
      ...feature,
      geometry: {
        type: 'MultiPolygon',
        coordinates
      },
      properties: {
        ...(feature.properties || {}),
        _mapRegion: region,
        _mapId: mapId,
        _parentMapId: parentMapId
      }
    }];
  });
}

function convertWebMercatorToLngLat(coordinates) {
  if (!Array.isArray(coordinates)) return coordinates;
  if (coordinates.length >= 2 && typeof coordinates[0] === 'number' && typeof coordinates[1] === 'number') {
    const [x, y, ...rest] = coordinates;
    const lng = x * 180 / 20037508.34;
    const lat = 180 / Math.PI * (2 * Math.atan(Math.exp(y * Math.PI / 20037508.34)) - Math.PI / 2);
    return [lng, lat, ...rest];
  }
  return coordinates.map(convertWebMercatorToLngLat);
}

function getMoscowBoundaryRegion(osmId, properties = {}) {
  // Поповка раньше была в московском слое; этот ключ уже есть в сохранённых настройках.
  if (osmId === '21228792') return 'msk';
  return properties.ADMIN_L4 === 'Москва' ? 'msk' : 'mo';
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
    const inherited = next[feature.properties._parentMapId];
    next[id] = inherited
      ? { ...inherited, name: getMunicipalityName(feature.properties) }
      : getDefaultSettings(getMunicipalityName(feature.properties));
    changed = true;
  });
  if (changed) {
    settings.value = next;
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

function toggleEditMode() {
  isEditMode.value = !isEditMode.value;
  if (!isEditMode.value) selectedMunicipality.value = null;
}

function selectMunicipalityForEdit(event) {
  if (!isEditMode.value) return;
  const feature = event.features?.[0];
  if (!feature) return;
  const properties = feature.properties || {};
  const id = properties._mapId;
  if (!id) return;
  if (!settings.value[id]) {
    settings.value = {
      ...settings.value,
      [id]: getDefaultSettings(getMunicipalityName(properties))
    };
  }
  selectedMunicipality.value = { id, name: getMunicipalityName(properties) };
}

function updateSelectedSettings(field, value) {
  const selected = selectedMunicipality.value;
  if (!selected) return;
  const current = settings.value[selected.id] || getDefaultSettings(selected.name);
  const nextValue = field === 'worksEnabled' ? value : Number(value);
  if (field !== 'worksEnabled' && (!Number.isFinite(nextValue) || nextValue < 0)) return;
  settings.value = {
    ...settings.value,
    [selected.id]: {
      ...current,
      [field]: nextValue
    }
  };
  persistSettings();
  syncMapData();
}

function loadSavedSettings() {
  try {
    return normalizeSettings(JSON.parse(localStorage.getItem(EDITOR_STORAGE_KEY) || '{}'));
  } catch {
    return {};
  }
}

function persistSettings() {
  try {
    localStorage.setItem(EDITOR_STORAGE_KEY, JSON.stringify(settings.value));
  } catch (error) {
    console.warn('[district-coeff-map] failed to save edited settings', error);
  }
}

function exportSettings() {
  const payload = JSON.stringify({
    schema: 'production-municipality-map-data',
    version: 1,
    exportedAt: new Date().toISOString(),
    settings: settings.value
  }, null, 2);
  const url = URL.createObjectURL(new Blob([payload], { type: 'application/json' }));
  const link = document.createElement('a');
  link.href = url;
  link.download = 'municipality-map-settings.json';
  link.click();
  URL.revokeObjectURL(url);
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
  if (normalized.includes('санкт-петербург') || normalized.includes('спб') || normalized.includes('ленинградская') || normalized.includes('москва') || normalized.includes('московская')) {
    return query;
  }
  const cityHint = activeRegion.value === 'msk' ? 'Москва' : 'Санкт-Петербург';
  return `${query}, ${cityHint}`;
}

function selectRegion(regionId) {
  const region = regions.find((item) => item.id === regionId);
  if (!region || !map) return;
  activeRegion.value = region.id;
  if (addressMarker) {
    addressMarker.remove();
    addressMarker = null;
  }
  clearMunicipalityHover();
  map.fitBounds(region.bounds, { padding: 56, duration: 750, maxZoom: region.id === 'msk' ? 8.5 : 10 });
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
    const parsed = JSON.parse(await file.text());
    const imported = normalizeSettings(parsed?.settings || parsed);
    settings.value = {
      ...settings.value,
      ...imported
    };
    persistSettings();

    if (parsed?.geojson?.type === 'FeatureCollection') {
      municipalities.value = ensureFeatureIds(parsed.geojson);
    }

    syncMapData();
    importStatus.value = parsed?.geojson
      ? `Загружено: ${Object.keys(imported).length} настроек и ${municipalities.value?.features?.length || 0} районов`
      : `Загружено: ${Object.keys(imported).length} районов`;
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
  if (properties._mapId) return String(properties._mapId);
  if (properties._mapRegion) {
    return `${properties._mapRegion}-${properties.OSM_ID || properties.OSM_IDD || feature.id || index + 1}`;
  }
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
