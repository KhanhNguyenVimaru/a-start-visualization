<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import * as L from 'leaflet'
import 'leaflet/dist/leaflet.css'
import type { LineString } from 'geojson'

type SelectionMode = 'start' | 'end'

interface LatLngPoint {
  lat: number
  lon: number
}

interface RouteSummary {
  distance: number
  duration: number
  geometry: LineString
}

interface AStarFrame {
  index: number
  lat: number
  lon: number
  g: number
  h: number
  f: number
}

const mapContainer = ref<HTMLDivElement | null>(null)
const map = ref<L.Map | null>(null)
const startPoint = ref<LatLngPoint | null>(null)
const endPoint = ref<LatLngPoint | null>(null)
const activeSelection = ref<SelectionMode>('start')
const loading = ref(false)
const error = ref('')
const routeSummary = ref<RouteSummary | null>(null)
const frames = ref<AStarFrame[]>([])
const currentFrameIndex = ref(0)
const followExplorer = ref(true)

const routeLayer = ref<L.GeoJSON | null>(null)
const explorationMarker = ref<L.CircleMarker | null>(null)
const startMarker = ref<L.Marker | null>(null)
const endMarker = ref<L.Marker | null>(null)
let requestToken = 0

const hasBothPoints = computed(() => Boolean(startPoint.value && endPoint.value))
const currentFrame = computed(() => frames.value[currentFrameIndex.value] ?? null)
const closedNodes = computed(() =>
  frames.value.slice(0, currentFrameIndex.value + 1).slice(-6).reverse(),
)
const openNodes = computed(() =>
  frames.value.slice(currentFrameIndex.value + 1, currentFrameIndex.value + 6),
)

onMounted(() => {
  if (!mapContainer.value) return

  const createdMap = L.map(mapContainer.value, { zoomControl: true }).setView(
    [21.0285, 105.8542],
    13,
  )
  map.value = createdMap

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution:
      '&copy; <a href="https://www.openstreetmap.org/copyright" target="_blank">OSM</a>',
    maxZoom: 19,
  }).addTo(createdMap)

  createdMap.on('click', handleMapClick)
})

onBeforeUnmount(() => {
  map.value?.off('click', handleMapClick)
  map.value?.remove()
})

watch([startPoint, endPoint], ([start, end]) => {
  if (start && end) {
    fetchRoute()
  } else {
    cleanupRoute()
  }
})

watch(currentFrameIndex, () => updateExplorationMarker())

watch(
  () => frames.value.length,
  () => {
    currentFrameIndex.value = 0
    updateExplorationMarker()
  },
)

function handleMapClick(event: L.LeafletMouseEvent) {
  const { lat, lng } = event.latlng
  if (activeSelection.value === 'start') {
    setStart({ lat, lon: lng })
    activeSelection.value = 'end'
  } else {
    setEnd({ lat, lon: lng })
  }
}

function setStart(point: LatLngPoint) {
  startPoint.value = point
  placeMarker(startMarker, point, '#16a34a', 'S')
}

function setEnd(point: LatLngPoint) {
  endPoint.value = point
  placeMarker(endMarker, point, '#dc2626', 'G')
}

function placeMarker(
  markerRef: typeof startMarker | typeof endMarker,
  point: LatLngPoint,
  color: string,
  label: string,
) {
  const currentMap = map.value
  if (!currentMap) return
  const mapInstance = currentMap as L.Map
  const icon = L.divIcon({
    html: `<span class="pin" style="background:${color}">${label}</span>`,
    className: 'pin-wrapper',
    iconSize: [28, 34],
    iconAnchor: [14, 34],
  })

  if (markerRef.value) {
    markerRef.value.setLatLng([point.lat, point.lon])
  } else {
    markerRef.value = L.marker([point.lat, point.lon], { icon }).addTo(mapInstance)
  }
}

async function fetchRoute() {
  const start = startPoint.value
  const end = endPoint.value
  if (!start || !end) return
  const token = ++requestToken
  loading.value = true
  error.value = ''

  const { lat: startLat, lon: startLon } = start
  const { lat: endLat, lon: endLon } = end
  const query =
    `https://router.project-osrm.org/route/v1/driving/` +
    `${startLon},${startLat};${endLon},${endLat}` +
    `?overview=full&geometries=geojson&steps=true`

  try {
    const response = await fetch(query)
    if (!response.ok) {
      throw new Error('Không thể kết nối OSRM')
    }
    const payload = await response.json()
    const route = payload?.routes?.[0]
    if (!route || token !== requestToken) return

    routeSummary.value = {
      distance: route.distance,
      duration: route.duration,
      geometry: route.geometry,
    }

    drawRoute(route.geometry)
    frames.value = createFrames(route.geometry.coordinates, end)
    focusOnRoute()
  } catch (err) {
    console.error(err)
    error.value =
      err instanceof Error ? err.message : 'Đã có lỗi khi tìm đường, thử lại sau nhé.'
    cleanupRoute()
  } finally {
    if (token === requestToken) {
      loading.value = false
    }
  }
}

function createFrames(coords: LineString['coordinates'], goal: LatLngPoint): AStarFrame[] {
  let g = 0
  const framesList: AStarFrame[] = []

  for (let i = 0; i < coords.length; i += 1) {
    const node = coords[i]
    if (!node) continue
    const [lon, lat] = node as [number, number]
    if (i > 0) {
      const previous = coords[i - 1] as [number, number]
      g += haversine(previous[1], previous[0], lat, lon)
    }
    const h = haversine(lat, lon, goal.lat, goal.lon)
    framesList.push({
      index: i,
      lat,
      lon,
      g,
      h,
      f: g + h,
    })
  }

  return framesList
}

function haversine(lat1: number, lon1: number, lat2: number, lon2: number) {
  const R = 6371000
  const toRad = (deg: number) => (deg * Math.PI) / 180
  const dLat = toRad(lat2 - lat1)
  const dLon = toRad(lon2 - lon1)
  const a =
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos(toRad(lat1)) * Math.cos(toRad(lat2)) * Math.sin(dLon / 2) * Math.sin(dLon / 2)
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a))
  return R * c
}

function drawRoute(geometry: LineString) {
  const currentMap = map.value
  if (!currentMap) return
  const mapInstance = currentMap as L.Map
  removeRouteLayer()
  routeLayer.value = L.geoJSON(geometry, {
    style: {
      color: '#2563eb',
      weight: 5,
      opacity: 0.85,
    },
  }).addTo(mapInstance)
}

function focusOnRoute() {
  const currentMap = map.value
  if (!currentMap || !routeLayer.value) return
  const mapInstance = currentMap as L.Map
  const bounds = routeLayer.value.getBounds()
  mapInstance.fitBounds(bounds, { padding: [24, 24] })
  updateExplorationMarker()
}

function updateExplorationMarker() {
  const currentMap = map.value
  if (!currentMap) return
  const mapInstance = currentMap as L.Map
  const frame = currentFrame.value
  if (!frame) {
    explorationMarker.value?.remove()
    explorationMarker.value = null
    return
  }

  const latLng: L.LatLngTuple = [frame.lat, frame.lon]
  if (explorationMarker.value) {
    explorationMarker.value.setLatLng(latLng)
  } else {
    explorationMarker.value = L.circleMarker(latLng, {
      radius: 8,
      weight: 2,
      color: '#f59e0b',
      fillColor: '#fcd34d',
      fillOpacity: 0.9,
    }).addTo(mapInstance)
  }

  if (followExplorer.value) {
    mapInstance.panTo(latLng, { animate: true, duration: 0.4 })
  }
}

function cleanupRoute() {
  routeSummary.value = null
  frames.value = []
  removeRouteLayer()
}

function removeRouteLayer() {
  routeLayer.value?.remove()
  routeLayer.value = null
  explorationMarker.value?.remove()
  explorationMarker.value = null
}

function resetAll() {
  startPoint.value = null
  endPoint.value = null
  activeSelection.value = 'start'
  cleanupRoute()
  startMarker.value?.remove()
  startMarker.value = null
  endMarker.value?.remove()
  endMarker.value = null
}

function swapPoints() {
  if (!startPoint.value || !endPoint.value) return
  const oldStart = { ...startPoint.value }
  startPoint.value = { ...endPoint.value }
  endPoint.value = oldStart
  placeMarker(startMarker, startPoint.value, '#16a34a', 'S')
  placeMarker(endMarker, endPoint.value, '#dc2626', 'G')
}

const canSwap = computed(() => Boolean(startPoint.value && endPoint.value))

const formattedStart = computed(() => formatPoint(startPoint.value))
const formattedEnd = computed(() => formatPoint(endPoint.value))

function formatPoint(point: LatLngPoint | null) {
  if (!point) return '--'
  return `${point.lat.toFixed(5)}, ${point.lon.toFixed(5)}`
}

function formatDistance(distance: number) {
  if (distance >= 1000) {
    return `${(distance / 1000).toFixed(2)} km`
  }
  return `${distance.toFixed(0)} m`
}

function formatDuration(durationSeconds: number) {
  const minutes = Math.round(durationSeconds / 60)
  if (minutes < 60) {
    return `${minutes} phút`
  }
  const hours = Math.floor(minutes / 60)
  const rest = minutes % 60
  return `${hours} giờ ${rest} phút`
}
</script>

<template>
  <div class="app-shell">
    <header class="hero">
      <div>
        <p class="kicker">A* demo • OSRM</p>
        <h1>Visualization tìm đường theo phong cách Google Maps</h1>
        <p class="subtitle">
          Chọn 2 điểm bất kỳ trên nền bản đồ OSM, hệ thống sẽ gọi OSRM và tái hiện lại logic
          A* thông qua các node trên đường đi.
        </p>
      </div>
      <div class="hero-actions">
        <button class="ghost" @click="swapPoints" :disabled="!canSwap">Đổi vị trí</button>
        <button class="ghost" @click="resetAll">Làm mới</button>
      </div>
    </header>

    <main class="layout">
      <section class="map-panel">
        <div class="map-toolbar">
          <div>
            <p class="label">Chế độ chọn</p>
            <div class="mode-toggle">
              <button
                class="mode"
                :class="{ active: activeSelection === 'start' }"
                @click="activeSelection = 'start'"
              >
                Điểm bắt đầu
              </button>
              <button
                class="mode"
                :class="{ active: activeSelection === 'end' }"
                @click="activeSelection = 'end'"
              >
                Điểm kết thúc
              </button>
            </div>
          </div>
          <div class="coords">
            <div>
              <p class="label">Start</p>
              <p class="coord">{{ formattedStart }}</p>
            </div>
            <div>
              <p class="label">Goal</p>
              <p class="coord">{{ formattedEnd }}</p>
            </div>
          </div>
        </div>
        <div ref="mapContainer" class="map-container"></div>
      </section>

      <section class="details-panel">
        <h2>Trạng thái A*</h2>
        <p class="hint">
          OSRM trả về 1 đường đi. Ta xem mỗi điểm của polyline này như node mà A* mở rộng.
          Slider phía dưới mô phỏng thứ tự duyệt và giá trị \(f = g + h\).
        </p>

        <div v-if="error" class="state error">{{ error }}</div>
        <div v-else-if="loading" class="state">Đang tìm đường với OSRM...</div>
        <template v-else>
          <div v-if="routeSummary" class="summary">
            <div class="summary-grid">
              <div>
                <p class="label">Quãng đường</p>
                <strong>{{ formatDistance(routeSummary.distance) }}</strong>
              </div>
              <div>
                <p class="label">Thời gian ước lượng</p>
                <strong>{{ formatDuration(routeSummary.duration) }}</strong>
              </div>
              <div class="follow-toggle">
                <label>
                  <input type="checkbox" v-model="followExplorer" />
                  Theo sát node đang xét
                </label>
              </div>
            </div>

            <div v-if="frames.length" class="slider">
              <input
                type="range"
                :max="frames.length - 1"
                min="0"
                v-model.number="currentFrameIndex"
              />
              <p class="label">
                Node #{{ currentFrame?.index ?? 0 }} • g={{ currentFrame?.g.toFixed(0) }}m •
                h={{ currentFrame?.h.toFixed(0) }}m
              </p>
            </div>

            <div v-if="currentFrame" class="frame-card">
              <p class="coord">
                {{ currentFrame.lat.toFixed(5) }}, {{ currentFrame.lon.toFixed(5) }}
              </p>
              <div class="metrics">
                <div>
                  <span>g (đi được)</span>
                  <strong>{{ formatDistance(currentFrame.g) }}</strong>
                </div>
                <div>
                  <span>h (ước lượng)</span>
                  <strong>{{ formatDistance(currentFrame.h) }}</strong>
                </div>
                <div>
                  <span>f</span>
                  <strong>{{ formatDistance(currentFrame.f) }}</strong>
                </div>
              </div>
            </div>

            <div class="table-wrapper" v-if="frames.length">
              <div>
                <h3>Closed set gần nhất</h3>
                <table>
                  <thead>
                    <tr>
                      <th>#</th>
                      <th>Lat</th>
                      <th>Lon</th>
                      <th>f</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr v-for="node in closedNodes" :key="`closed-${node.index}`">
                      <td>{{ node.index }}</td>
                      <td>{{ node.lat.toFixed(4) }}</td>
                      <td>{{ node.lon.toFixed(4) }}</td>
                      <td>{{ formatDistance(node.f) }}</td>
                    </tr>
                  </tbody>
                </table>
              </div>

              <div>
                <h3>Open set kế tiếp</h3>
                <table>
                  <thead>
                    <tr>
                      <th>#</th>
                      <th>Lat</th>
                      <th>Lon</th>
                      <th>f</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr v-for="node in openNodes" :key="`open-${node.index}`">
                      <td>{{ node.index }}</td>
                      <td>{{ node.lat.toFixed(4) }}</td>
                      <td>{{ node.lon.toFixed(4) }}</td>
                      <td>{{ formatDistance(node.f) }}</td>
                    </tr>
                  </tbody>
                </table>
              </div>
            </div>
          </div>
          <p v-else class="state muted">Nhấn vào bản đồ để chọn điểm bắt đầu và kết thúc.</p>
        </template>
      </section>
    </main>

    <footer>
      OSRM public demo server • dùng để minh hoạ A* nên số liệu có thể khác Google Maps.
    </footer>
  </div>
</template>

<style scoped>
:global(body) {
  font-family: 'Inter', system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  background: #0f172a;
  color: #e2e8f0;
}

.app-shell {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  gap: 24px;
  padding: 32px clamp(16px, 4vw, 64px);
}

.hero {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
  gap: 16px;
  align-items: flex-end;
}

.hero h1 {
  font-size: clamp(24px, 4vw, 36px);
  margin: 4px 0;
}

.subtitle {
  color: #cbd5f5;
  max-width: 640px;
}

.kicker {
  text-transform: uppercase;
  font-size: 12px;
  letter-spacing: 0.4em;
  color: #a5b4fc;
}

.hero-actions {
  display: flex;
  gap: 12px;
}

.ghost {
  border: 1px solid rgba(148, 163, 184, 0.4);
  color: #e2e8f0;
  background: transparent;
  padding: 8px 16px;
  border-radius: 999px;
  cursor: pointer;
  transition: border-color 0.2s ease, color 0.2s ease;
}

.ghost:disabled {
  cursor: not-allowed;
  opacity: 0.5;
}

.ghost:not(:disabled):hover {
  border-color: #e2e8f0;
}

.layout {
  display: grid;
  grid-template-columns: 1.1fr 0.9fr;
  gap: clamp(16px, 2vw, 32px);
}

@media (max-width: 960px) {
  .layout {
    grid-template-columns: 1fr;
  }

  .hero {
    flex-direction: column;
    align-items: flex-start;
  }
}

.map-panel {
  background: #0b1120;
  border-radius: 20px;
  border: 1px solid rgba(148, 163, 184, 0.2);
  overflow: hidden;
}

.map-toolbar {
  display: flex;
  justify-content: space-between;
  gap: 16px;
  padding: 16px 20px;
  border-bottom: 1px solid rgba(148, 163, 184, 0.2);
  flex-wrap: wrap;
}

.label {
  color: #94a3b8;
  font-size: 12px;
  letter-spacing: 0.05em;
}

.mode-toggle {
  display: flex;
  gap: 8px;
  margin-top: 8px;
}

.mode {
  border: 1px solid rgba(148, 163, 184, 0.2);
  background: transparent;
  color: #e2e8f0;
  border-radius: 10px;
  padding: 6px 12px;
  cursor: pointer;
}

.mode.active {
  background: #1d4ed8;
  border-color: #2563eb;
}

.coords {
  display: flex;
  gap: 24px;
}

.coord {
  font-size: 14px;
  color: #e2e8f0;
}

.map-container {
  height: clamp(320px, 60vh, 520px);
}

:global(.leaflet-container) {
  width: 100%;
  height: 100%;
  border-bottom-left-radius: 20px;
  border-bottom-right-radius: 20px;
}

.details-panel {
  background: #0b1120;
  border: 1px solid rgba(148, 163, 184, 0.2);
  border-radius: 20px;
  padding: 24px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.details-panel h2 {
  margin: 0;
}

.hint {
  font-size: 14px;
  color: #cbd5f5;
}

.summary-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
  gap: 12px;
  align-items: center;
}

.summary-grid strong {
  font-size: 20px;
}

.follow-toggle label {
  display: flex;
  gap: 8px;
  align-items: center;
  font-size: 14px;
  color: #e2e8f0;
}

.slider input {
  width: 100%;
  accent-color: #f59e0b;
}

.frame-card {
  padding: 16px;
  border-radius: 16px;
  background: rgba(46, 16, 101, 0.4);
  border: 1px solid rgba(99, 102, 241, 0.3);
}

.metrics {
  margin-top: 12px;
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
}

.metrics span {
  font-size: 12px;
  color: #cbd5f5;
}

.table-wrapper {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 16px;
}

table {
  width: 100%;
  border-collapse: collapse;
  font-size: 13px;
}

th,
td {
  padding: 8px;
  border-bottom: 1px solid rgba(148, 163, 184, 0.2);
  text-align: left;
}

.state {
  padding: 12px 14px;
  border-radius: 12px;
  background: rgba(15, 23, 42, 0.6);
}

.state.error {
  border: 1px solid rgba(248, 113, 113, 0.4);
  color: #fecaca;
  background: rgba(127, 29, 29, 0.3);
}

.state.muted {
  color: #94a3b8;
  border: 1px dashed rgba(148, 163, 184, 0.4);
}

.pin-wrapper {
  width: 0;
  height: 0;
}

.pin {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 999px;
  color: #0f172a;
  font-weight: 700;
  box-shadow:
    0 10px 20px rgba(0, 0, 0, 0.3),
    inset 0 -2px 4px rgba(0, 0, 0, 0.2);
}

footer {
  text-align: center;
  font-size: 12px;
  color: #94a3b8;
  margin-top: auto;
}
</style>
