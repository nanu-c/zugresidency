<script setup>
import { ref, shallowRef, computed, onMounted, watch } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

const TYPE_COLORS = {
  Hochgeschwindigkeit: '#dc2626',
  EuroCity: '#1d4ed8',
  Fernzug: '#15803d',
  Nachtzug: '#6d28d9',
}
const ALL_TYPES = Object.keys(TYPE_COLORS)

const trainFeatures = shallowRef([])
const searchQuery = ref('')
const activeTypes = ref(new Set(ALL_TYPES))
const onlyInterrail = ref(false)
const selected = ref(null)
const showList = ref(false)
const isLoading = ref(true)

let map = null
const layerMap = new Map()

const visibleSet = computed(() => {
  const q = searchQuery.value.toLowerCase().trim()
  const types = activeTypes.value
  const ir = onlyInterrail.value
  return new Set(
    trainFeatures.value.filter(f => {
      const p = f.properties
      if (!types.has(p.typ)) return false
      if (ir && p.interrail === 'nein') return false
      if (!q) return true
      return (
        p.name.toLowerCase().includes(q) ||
        p.zuggattung.toLowerCase().includes(q) ||
        (p.betreiber || '').toLowerCase().includes(q)
      )
    })
  )
})

const visibleCount = computed(() => visibleSet.value.size)
const isFiltered = computed(() =>
  !!searchQuery.value.trim() || activeTypes.value.size < ALL_TYPES.length || onlyInterrail.value
)

const groupedVisible = computed(() => {
  const result = {}
  for (const typ of ALL_TYPES) {
    const items = [...visibleSet.value]
      .filter(f => f.properties.typ === typ)
      .sort((a, b) => a.properties.name.localeCompare(b.properties.name, 'de'))
    if (items.length) result[typ] = items
  }
  return result
})

function getStyle(feature) {
  const p = feature.properties
  const visible = visibleSet.value.has(feature)
  const isSel = selected.value?.name === p.name
  if (isSel) {
    return { color: '#f59e0b', weight: 6, opacity: 1, dashArray: null }
  }
  return {
    color: visible ? (TYPE_COLORS[p.typ] || '#888') : '#c8cdd6',
    weight: visible ? 3 : 1.5,
    opacity: visible ? 0.85 : 0.15,
    dashArray: visible && p.tageszeit === 'Nacht' ? '8, 5' : null,
  }
}

function refreshStyles() {
  layerMap.forEach((layer, name) => {
    const feature = trainFeatures.value.find(f => f.properties.name === name)
    if (feature) layer.setStyle(getStyle(feature))
  })
  if (selected.value) layerMap.get(selected.value.name)?.bringToFront()
}

function fitToFeature(feature) {
  const coords = feature.geometry.coordinates
  const bounds = L.latLngBounds(coords.map(([lng, lat]) => [lat, lng]))
  map.fitBounds(bounds, { padding: [80, 60], maxZoom: 9 })
}

function initLayer(data) {
  trainFeatures.value = data.features
  L.geoJSON(data, {
    style: getStyle,
    onEachFeature(feature, layer) {
      layerMap.set(feature.properties.name, layer)
      layer.on('click', e => {
        L.DomEvent.stopPropagation(e)
        if (!visibleSet.value.has(feature)) return
        if (selected.value?.name === feature.properties.name) {
          selected.value = null
        } else {
          selected.value = feature.properties
          showList.value = false
        }
      })
      layer.on('mouseover', e => {
        if (visibleSet.value.has(feature) && selected.value?.name !== feature.properties.name) {
          e.target.setStyle({ weight: 5, opacity: 1 })
          e.target.bringToFront()
        }
      })
      layer.on('mouseout', () => {
        layer.setStyle(getStyle(feature))
        if (selected.value) layerMap.get(selected.value.name)?.bringToFront()
      })
    },
  }).addTo(map)
}

function zoomToSelected() {
  const feature = trainFeatures.value.find(f => f.properties.name === selected.value?.name)
  if (feature) fitToFeature(feature)
}

function selectFromList(feature) {
  selected.value = feature.properties
  showList.value = false
  fitToFeature(feature)
}

function toggleType(typ) {
  const s = new Set(activeTypes.value)
  if (s.has(typ)) {
    if (s.size > 1) s.delete(typ)
  } else {
    s.add(typ)
  }
  activeTypes.value = s
}

function resetFilters() {
  searchQuery.value = ''
  activeTypes.value = new Set(ALL_TYPES)
  onlyInterrail.value = false
  selected.value = null
}

onMounted(async () => {
  map = L.map('map', { zoomControl: false }).setView([50, 13], 5)
  L.tileLayer('https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png', {
    attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>, &copy; <a href="https://carto.com/">CARTO</a>',
    maxZoom: 19,
    subdomains: 'abcd',
  }).addTo(map)
  L.control.zoom({ position: 'bottomright' }).addTo(map)
  map.on('click', () => {
    selected.value = null
    showList.value = false
  })

  const res = await fetch('./trains.geojson')
  const data = await res.json()
  isLoading.value = false
  initLayer(data)
})

watch([searchQuery, activeTypes, onlyInterrail, selected], refreshStyles)
</script>

<template>
  <div id="app">
    <div v-if="isLoading" class="loading-overlay">
      <div class="spinner" />
      <p>Lade Züge&hellip;</p>
    </div>

    <header class="top-bar">
      <div class="top-row">
        <div class="brand-group">
          <span class="brand">Zugresidency</span>
          <span class="subtitle">Europäische Fernzüge 2026</span>
        </div>
        <button
          class="btn-list-toggle"
          :class="{ 'btn-list-toggle--active': showList }"
          @click="showList = !showList; if (showList) selected = null"
          aria-label="Zugliste anzeigen"
        >
          <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <line x1="9" y1="6" x2="21" y2="6" /><line x1="9" y1="12" x2="21" y2="12" /><line x1="9" y1="18" x2="21" y2="18" />
            <circle cx="4" cy="6" r="1.2" fill="currentColor" stroke="none" /><circle cx="4" cy="12" r="1.2" fill="currentColor" stroke="none" /><circle cx="4" cy="18" r="1.2" fill="currentColor" stroke="none" />
          </svg>
          Liste
          <span v-if="isFiltered" class="list-toggle-count">{{ visibleCount }}</span>
          <span v-else class="list-toggle-count">{{ trainFeatures.length }}</span>
        </button>
      </div>

      <div class="search-wrap">
        <svg class="icon-search" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <circle cx="11" cy="11" r="8" /><line x1="21" y1="21" x2="16.65" y2="16.65" />
        </svg>
        <input
          v-model="searchQuery"
          type="search"
          placeholder="Zug, Betreiber oder Zuggattung suchen&hellip;"
          class="search-input"
          autocomplete="off"
        >
        <button v-if="searchQuery" class="btn-clear" @click="searchQuery = ''" aria-label="Suche löschen">
          <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">
            <line x1="18" y1="6" x2="6" y2="18" /><line x1="6" y1="6" x2="18" y2="18" />
          </svg>
        </button>
      </div>

      <div class="chips-row">
        <button
          v-for="typ in ALL_TYPES"
          :key="typ"
          class="chip"
          :class="{ 'chip--active': activeTypes.has(typ) }"
          :style="activeTypes.has(typ) ? { '--chip-color': TYPE_COLORS[typ] } : {}"
          @click="toggleType(typ)"
        >
          <span class="chip-dot" :style="{ background: TYPE_COLORS[typ] }" />
          {{ typ }}
        </button>
        <button
          class="chip chip--interrail"
          :class="{ 'chip--interrail-active': onlyInterrail }"
          @click="onlyInterrail = !onlyInterrail"
        >
          <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" style="width:12px;height:12px;flex-shrink:0">
            <rect x="2" y="5" width="20" height="14" rx="2"/><line x1="2" y1="10" x2="22" y2="10"/>
          </svg>
          Interrail
        </button>
        <button v-if="isFiltered" class="chip chip--reset" @click="resetFilters">
          Alle anzeigen
        </button>
      </div>
    </header>

    <div id="map" />

    <div v-if="isFiltered && !showList && !selected && !isLoading" class="count-badge">
      {{ visibleCount }}&thinsp;/&thinsp;{{ trainFeatures.length }} Züge
    </div>

    <!-- List panel -->
    <Transition name="panel">
      <div v-if="showList" class="list-panel">
        <div class="panel-handle" />
        <div class="list-panel-header">
          <h2 class="list-panel-title">
            <template v-if="isFiltered">{{ visibleCount }} Züge</template>
            <template v-else>Alle {{ trainFeatures.length }} Züge</template>
          </h2>
          <button class="btn-close" @click="showList = false" aria-label="Liste schließen">
            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">
              <line x1="18" y1="6" x2="6" y2="18" /><line x1="6" y1="6" x2="18" y2="18" />
            </svg>
          </button>
        </div>

        <div v-if="visibleCount === 0" class="list-empty">
          Keine Züge gefunden.
        </div>

        <div class="list-content">
          <template v-for="typ in ALL_TYPES" :key="typ">
            <div v-if="groupedVisible[typ]" class="list-group">
              <div class="list-group-header">
                <span class="group-dot" :style="{ background: TYPE_COLORS[typ] }" />
                <span class="group-label">{{ typ }}</span>
                <span class="group-count">{{ groupedVisible[typ].length }}</span>
              </div>
              <button
                v-for="feature in groupedVisible[typ]"
                :key="feature.properties.name"
                class="list-item"
                @click="selectFromList(feature)"
              >
                <div class="list-item-stripe" :style="{ background: TYPE_COLORS[feature.properties.typ] }" />
                <div class="list-item-body">
                  <div class="list-item-name">{{ feature.properties.name }}</div>
                  <div class="list-item-meta">
                    {{ feature.properties.zuggattung }}
                    <span class="meta-sep">·</span>
                    {{ feature.properties.betreiber }}
                  </div>
                  <div v-if="feature.properties.dauer" class="list-item-dauer">{{ feature.properties.dauer }}</div>
                  <div v-if="feature.properties.bemerkung" class="list-item-hint">
                    {{ feature.properties.bemerkung }}
                  </div>
                </div>
                <div class="list-item-right">
                  <a
                    v-if="feature.properties.buchung_url"
                    :href="feature.properties.buchung_url"
                    target="_blank"
                    rel="noopener noreferrer"
                    class="list-book-link"
                    @click.stop
                    title="Tickets buchen"
                  >
                    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
                      <path d="M18 13v6a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V8a2 2 0 0 1 2-2h6"/><polyline points="15 3 21 3 21 9"/><line x1="10" y1="14" x2="21" y2="3"/>
                    </svg>
                  </a>
                  <span
                    v-if="feature.properties.tageszeit === 'Nacht'"
                    class="list-night-tag"
                  >Nacht</span>
                  <span
                    v-if="feature.properties.interrail"
                    class="list-ir-tag"
                    :class="{
                      'list-ir-tag--ja':       feature.properties.interrail === 'ja',
                      'list-ir-tag--aufpreis': feature.properties.interrail === 'aufpreis',
                      'list-ir-tag--nein':     feature.properties.interrail === 'nein',
                    }"
                  >IR</span>
                </div>
              </button>
            </div>
          </template>
        </div>
      </div>
    </Transition>

    <!-- Detail panel -->
    <Transition name="panel">
      <div v-if="selected" class="detail-panel">
        <div class="panel-handle" />
        <div class="panel-header">
          <div class="panel-stripe" :style="{ background: TYPE_COLORS[selected.typ] }" />
          <h2 class="panel-title">{{ selected.name }}</h2>
          <button class="btn-close" @click="selected = null" aria-label="Schließen">
            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">
              <line x1="18" y1="6" x2="6" y2="18" /><line x1="6" y1="6" x2="18" y2="18" />
            </svg>
          </button>
        </div>

        <div class="badges">
          <span class="badge" :style="{ background: TYPE_COLORS[selected.typ] }">{{ selected.typ }}</span>
          <span class="badge badge--time">{{ selected.tageszeit === 'Nacht' ? 'Nachtzug' : 'Tagzug' }}</span>
          <span
            v-if="selected.interrail"
            class="badge"
            :class="{
              'badge--ir-ja':       selected.interrail === 'ja',
              'badge--ir-aufpreis': selected.interrail === 'aufpreis',
              'badge--ir-nein':     selected.interrail === 'nein',
            }"
          >
            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" style="width:11px;height:11px">
              <rect x="2" y="5" width="20" height="14" rx="2"/><line x1="2" y1="10" x2="22" y2="10"/>
            </svg>
            {{
              selected.interrail === 'ja'       ? 'Interrail' :
              selected.interrail === 'aufpreis' ? 'Interrail + Aufpreis' :
                                                  'Kein Interrail'
            }}
          </span>
          <span v-if="selected.dauer" class="badge badge--dauer">
            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><circle cx="12" cy="12" r="10"/><polyline points="12 6 12 12 16 14"/></svg>
            {{ selected.dauer }}
          </span>
        </div>

        <div v-if="selected.bemerkung" class="bemerkung-box">
          <span class="bemerkung-label">Hinweis</span>
          {{ selected.bemerkung }}
        </div>

        <dl class="detail-list">
          <dt>Zuggattung</dt>
          <dd>{{ selected.zuggattung }}</dd>
          <dt>Betreiber</dt>
          <dd>{{ selected.betreiber }}</dd>
        </dl>

        <div v-if="selected.stationen && selected.stationen.length > 1" class="route-stations">
          <div
            v-for="(station, i) in selected.stationen"
            :key="i"
            class="route-station"
          >
            <div class="station-track">
              <div v-if="i > 0" class="track-line" />
              <div
                class="station-dot"
                :class="{ 'station-dot--terminal': i === 0 || i === selected.stationen.length - 1 }"
              />
              <div v-if="i < selected.stationen.length - 1" class="track-line" />
            </div>
            <div
              class="station-name"
              :class="{ 'station-name--terminal': i === 0 || i === selected.stationen.length - 1 }"
            >{{ station }}</div>
          </div>
        </div>

        <div class="detail-actions">
          <a
            v-if="selected.buchung_url"
            :href="selected.buchung_url"
            target="_blank"
            rel="noopener noreferrer"
            class="btn-action btn-book"
          >
            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <rect x="2" y="5" width="20" height="14" rx="2"/><line x1="2" y1="10" x2="22" y2="10"/>
            </svg>
            Tickets buchen
          </a>
          <button class="btn-action btn-zoom" @click="zoomToSelected">
            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <circle cx="11" cy="11" r="8" /><line x1="21" y1="21" x2="16.65" y2="16.65" />
              <line x1="11" y1="8" x2="11" y2="14" /><line x1="8" y1="11" x2="14" y2="11" />
            </svg>
            Zur Route
          </button>
          <button class="btn-action btn-back-list" @click="selected = null; showList = true">
            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <line x1="9" y1="6" x2="21" y2="6" /><line x1="9" y1="12" x2="21" y2="12" /><line x1="9" y1="18" x2="21" y2="18" />
              <circle cx="4" cy="6" r="1.2" fill="currentColor" stroke="none" /><circle cx="4" cy="12" r="1.2" fill="currentColor" stroke="none" /><circle cx="4" cy="18" r="1.2" fill="currentColor" stroke="none" />
            </svg>
            Zur Liste
          </button>
        </div>
      </div>
    </Transition>
  </div>
</template>

<style>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

html, body {
  height: 100%;
  font-family: system-ui, -apple-system, 'Segoe UI', sans-serif;
}

#app {
  height: 100dvh;
  position: relative;
  overflow: hidden;
}

/* ── Loading ── */
.loading-overlay {
  position: fixed;
  inset: 0;
  z-index: 9999;
  background: rgba(255, 255, 255, 0.92);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 14px;
  font-size: 14px;
  color: #64748b;
}
.spinner {
  width: 32px;
  height: 32px;
  border: 3px solid #e2e8f0;
  border-top-color: #1d4ed8;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }

/* ── Header ── */
.top-bar {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 800;
  background: rgba(255, 255, 255, 0.97);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border-bottom: 1px solid rgba(0, 0, 0, 0.08);
  padding: 10px 12px 8px;
  display: flex;
  flex-direction: column;
  gap: 7px;
}

.top-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
}
.brand-group {
  display: flex;
  align-items: baseline;
  gap: 7px;
  min-width: 0;
}
.brand {
  font-size: 14px;
  font-weight: 800;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  color: #1e293b;
  white-space: nowrap;
}
.subtitle {
  font-size: 11px;
  color: #94a3b8;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.btn-list-toggle {
  display: flex;
  align-items: center;
  gap: 5px;
  height: 30px;
  padding: 0 10px 0 8px;
  border: 1.5px solid #e2e8f0;
  border-radius: 100px;
  background: #fff;
  font-size: 13px;
  font-weight: 500;
  color: #475569;
  cursor: pointer;
  white-space: nowrap;
  flex-shrink: 0;
  transition: border-color 0.15s, background 0.15s, color 0.15s;
}
.btn-list-toggle svg { width: 15px; height: 15px; }
.btn-list-toggle:hover { border-color: #94a3b8; color: #1e293b; }
.btn-list-toggle--active {
  background: #1e293b;
  border-color: #1e293b;
  color: #fff;
}

.list-toggle-count {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 18px;
  height: 18px;
  padding: 0 4px;
  background: #e2e8f0;
  color: #475569;
  border-radius: 100px;
  font-size: 10px;
  font-weight: 600;
}
.btn-list-toggle--active .list-toggle-count {
  background: rgba(255, 255, 255, 0.2);
  color: #fff;
}

.search-wrap {
  position: relative;
  display: flex;
  align-items: center;
}
.icon-search {
  position: absolute;
  left: 10px;
  width: 15px;
  height: 15px;
  color: #94a3b8;
  pointer-events: none;
}
.search-input {
  width: 100%;
  height: 38px;
  padding: 0 36px 0 32px;
  border: 1.5px solid #e2e8f0;
  border-radius: 10px;
  font-size: 14px;
  color: #1e293b;
  background: #f8fafc;
  outline: none;
  transition: border-color 0.15s, box-shadow 0.15s, background 0.15s;
  -webkit-appearance: none;
  appearance: none;
}
.search-input:focus {
  border-color: #1d4ed8;
  box-shadow: 0 0 0 3px rgba(29, 78, 216, 0.1);
  background: #fff;
}
.search-input::placeholder { color: #94a3b8; }
.search-input::-webkit-search-cancel-button { display: none; }

.btn-clear {
  position: absolute;
  right: 8px;
  width: 22px;
  height: 22px;
  border: none;
  background: #e2e8f0;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #64748b;
  border-radius: 50%;
  padding: 0;
  transition: background 0.15s;
}
.btn-clear svg { width: 11px; height: 11px; }
.btn-clear:hover { background: #cbd5e1; }

.chips-row {
  display: flex;
  gap: 6px;
  overflow-x: auto;
  padding-bottom: 1px;
  scrollbar-width: none;
  -ms-overflow-style: none;
}
.chips-row::-webkit-scrollbar { display: none; }

.chip {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  gap: 5px;
  height: 27px;
  padding: 0 10px;
  border: 1.5px solid #e2e8f0;
  border-radius: 100px;
  background: #fff;
  font-size: 12px;
  font-weight: 500;
  color: #64748b;
  cursor: pointer;
  white-space: nowrap;
  transition: border-color 0.15s, background 0.15s, color 0.15s;
}
.chip:hover:not(.chip--active) { border-color: #94a3b8; color: #1e293b; }
.chip--active {
  background: var(--chip-color, #1d4ed8);
  border-color: var(--chip-color, #1d4ed8);
  color: #fff;
}
.chip--reset { border-style: dashed; color: #94a3b8; }
.chip--reset:hover { border-color: #64748b; color: #475569; }
.chip--interrail { gap: 5px; }
.chip--interrail-active { background: #166534; border-color: #166534; color: #fff; }
.chip-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  flex-shrink: 0;
}
.chip--active .chip-dot { background: rgba(255, 255, 255, 0.5) !important; }

/* ── Map ── */
#map {
  position: fixed;
  inset: 0;
  z-index: 1;
}

/* ── Count badge ── */
.count-badge {
  position: fixed;
  bottom: 88px;
  left: 12px;
  z-index: 800;
  background: rgba(255, 255, 255, 0.95);
  border: 1px solid #e2e8f0;
  border-radius: 20px;
  padding: 4px 12px;
  font-size: 12px;
  font-weight: 500;
  color: #64748b;
  backdrop-filter: blur(4px);
  pointer-events: none;
}

/* ── Shared panel base ── */
.panel-handle {
  width: 36px;
  height: 4px;
  background: #e2e8f0;
  border-radius: 2px;
  margin: 10px auto 14px;
  flex-shrink: 0;
}

.btn-close {
  flex-shrink: 0;
  width: 30px;
  height: 30px;
  border: none;
  background: #f1f5f9;
  border-radius: 8px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #64748b;
  padding: 0;
  transition: background 0.15s;
  margin-top: 1px;
}
.btn-close svg { width: 14px; height: 14px; }
.btn-close:hover { background: #e2e8f0; }

/* ── List panel ── */
.list-panel {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 900;
  background: #fff;
  border-radius: 18px 18px 0 0;
  max-height: 78vh;
  display: flex;
  flex-direction: column;
  box-shadow: 0 -4px 32px rgba(0, 0, 0, 0.14);
  padding: 0 0 24px;
  overscroll-behavior: contain;
}

.list-panel-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 16px 10px;
  flex-shrink: 0;
  border-bottom: 1px solid #f1f5f9;
}
.list-panel-title {
  font-size: 15px;
  font-weight: 700;
  color: #1e293b;
}

.list-empty {
  padding: 32px 16px;
  text-align: center;
  font-size: 14px;
  color: #94a3b8;
}

.list-content {
  overflow-y: auto;
  flex: 1;
  overscroll-behavior: contain;
}

.list-group {
  margin-bottom: 4px;
}

.list-group-header {
  display: flex;
  align-items: center;
  gap: 7px;
  padding: 10px 16px 6px;
  position: sticky;
  top: 0;
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(4px);
  z-index: 1;
}
.group-dot {
  width: 9px;
  height: 9px;
  border-radius: 50%;
  flex-shrink: 0;
}
.group-label {
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: #64748b;
}
.group-count {
  font-size: 11px;
  font-weight: 600;
  color: #94a3b8;
  background: #f1f5f9;
  border-radius: 100px;
  padding: 1px 7px;
  margin-left: auto;
}

.list-item {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  width: 100%;
  padding: 9px 16px 9px 12px;
  border: none;
  background: none;
  cursor: pointer;
  text-align: left;
  transition: background 0.12s;
  border-bottom: 1px solid #f8fafc;
}
.list-item:hover, .list-item:active { background: #f8fafc; }

.list-item-stripe {
  width: 3px;
  min-height: 28px;
  align-self: stretch;
  border-radius: 2px;
  flex-shrink: 0;
}

.list-item-body {
  flex: 1;
  min-width: 0;
}
.list-item-name {
  font-size: 14px;
  font-weight: 600;
  color: #1e293b;
  line-height: 1.3;
  margin-bottom: 2px;
}
.list-item-meta {
  font-size: 12px;
  color: #64748b;
  line-height: 1.4;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.meta-sep { margin: 0 3px; color: #cbd5e1; }
.list-item-dauer {
  font-size: 11px;
  color: #15803d;
  font-weight: 600;
  margin-top: 2px;
}
.list-item-hint {
  font-size: 11px;
  color: #92400e;
  background: #fef3c7;
  border-radius: 4px;
  padding: 2px 6px;
  margin-top: 4px;
  display: inline-block;
  line-height: 1.4;
}

.list-item-right {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 4px;
  flex-shrink: 0;
  align-self: center;
}
.list-night-tag {
  font-size: 10px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  color: #6d28d9;
  background: #ede9fe;
  border-radius: 100px;
  padding: 2px 7px;
  white-space: nowrap;
}
.list-ir-tag {
  font-size: 10px;
  font-weight: 700;
  border-radius: 100px;
  padding: 2px 7px;
  white-space: nowrap;
  letter-spacing: 0.03em;
}
.list-ir-tag--ja      { background: #dcfce7; color: #166534; }
.list-ir-tag--aufpreis{ background: #fef3c7; color: #92400e; }
.list-ir-tag--nein    { background: #f1f5f9; color: #94a3b8; }
.list-book-link {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  border-radius: 7px;
  background: #f0fdf4;
  color: #166534;
  text-decoration: none;
  transition: background 0.15s;
}
.list-book-link svg { width: 13px; height: 13px; }
.list-book-link:hover { background: #dcfce7; }

/* ── Detail panel ── */
.detail-panel {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 910;
  background: #fff;
  border-radius: 18px 18px 0 0;
  padding: 0 16px 40px;
  max-height: 62vh;
  overflow-y: auto;
  box-shadow: 0 -4px 32px rgba(0, 0, 0, 0.16);
  overscroll-behavior: contain;
}

.panel-header {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  margin-bottom: 10px;
}
.panel-stripe {
  width: 4px;
  min-height: 28px;
  border-radius: 2px;
  flex-shrink: 0;
  margin-top: 3px;
}
.panel-title {
  flex: 1;
  font-size: 18px;
  font-weight: 700;
  color: #1e293b;
  line-height: 1.3;
}

.badges {
  display: flex;
  gap: 6px;
  margin-bottom: 12px;
  flex-wrap: wrap;
}
.badge {
  padding: 3px 10px;
  border-radius: 100px;
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  color: #fff;
}
.badge--time {
  background: #f1f5f9;
  color: #64748b;
}
.badge--dauer {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  background: #f0fdf4;
  color: #15803d;
  border: 1px solid #bbf7d0;
}
.badge--dauer svg { width: 12px; height: 12px; }
.badge--ir-ja      { display:inline-flex;align-items:center;gap:4px;background:#dcfce7;color:#166534;border:1px solid #bbf7d0; }
.badge--ir-aufpreis{ display:inline-flex;align-items:center;gap:4px;background:#fef3c7;color:#92400e;border:1px solid #fde68a; }
.badge--ir-nein    { display:inline-flex;align-items:center;gap:4px;background:#f1f5f9;color:#64748b;border:1px solid #e2e8f0; }

.bemerkung-box {
  display: flex;
  flex-direction: column;
  gap: 3px;
  background: #fffbeb;
  border-left: 3px solid #f59e0b;
  border-radius: 0 8px 8px 0;
  padding: 8px 12px;
  margin-bottom: 14px;
  font-size: 13px;
  color: #78350f;
  line-height: 1.5;
}
.bemerkung-label {
  font-size: 10px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: #d97706;
}

.detail-list {
  display: grid;
  grid-template-columns: max-content 1fr;
  gap: 7px 14px;
  font-size: 14px;
  line-height: 1.5;
  margin-bottom: 16px;
}
.detail-list dt {
  color: #94a3b8;
  font-weight: 500;
}
.detail-list dd {
  color: #1e293b;
}

.route-stations {
  display: flex;
  flex-direction: column;
  margin-bottom: 16px;
}
.route-station {
  display: flex;
  align-items: stretch;
  gap: 10px;
}
.station-track {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 16px;
  flex-shrink: 0;
}
.track-line {
  flex: 1;
  width: 2px;
  background: #e2e8f0;
  min-height: 7px;
}
.station-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: #cbd5e1;
  box-shadow: 0 0 0 1.5px #94a3b8;
  flex-shrink: 0;
}
.station-dot--terminal {
  width: 9px;
  height: 9px;
  background: #1e293b;
  box-shadow: 0 0 0 1.5px #1e293b;
}
.station-name {
  font-size: 13px;
  color: #64748b;
  padding: 3px 0;
  line-height: 1.4;
  align-self: center;
}
.station-name--terminal {
  color: #1e293b;
  font-weight: 600;
}

.detail-actions {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.btn-action {
  display: flex;
  align-items: center;
  gap: 6px;
  height: 36px;
  padding: 0 14px;
  border: 1.5px solid #e2e8f0;
  border-radius: 10px;
  background: #f8fafc;
  font-size: 13px;
  font-weight: 500;
  color: #475569;
  cursor: pointer;
  transition: border-color 0.15s, background 0.15s;
}
.btn-action svg { width: 15px; height: 15px; flex-shrink: 0; }
.btn-action:hover { border-color: #94a3b8; background: #fff; color: #1e293b; }
.btn-book {
  text-decoration: none;
  background: #166534;
  border-color: #166534;
  color: #fff;
  font-weight: 600;
}
.btn-book:hover { background: #15803d; border-color: #15803d; color: #fff; }
.btn-zoom { border-color: #bfdbfe; color: #1d4ed8; background: #eff6ff; }
.btn-zoom:hover { border-color: #1d4ed8; background: #dbeafe; color: #1e40af; }

/* ── Transitions ── */
.panel-enter-active,
.panel-leave-active {
  transition: transform 0.28s cubic-bezier(0.32, 0.72, 0, 1), opacity 0.2s;
}
.panel-enter-from,
.panel-leave-to {
  transform: translateY(100%);
  opacity: 0;
}

/* ── Leaflet overrides ── */
.leaflet-control-zoom {
  border-radius: 10px !important;
  overflow: hidden;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.15) !important;
  border: none !important;
}
.leaflet-control-zoom-in,
.leaflet-control-zoom-out {
  width: 36px !important;
  height: 36px !important;
  line-height: 36px !important;
  font-size: 18px !important;
  border: none !important;
  color: #1e293b !important;
}
.leaflet-control-attribution {
  font-size: 10px !important;
  background: rgba(255, 255, 255, 0.8) !important;
}

/* ── Desktop ── */
@media (min-width: 640px) {
  .list-panel {
    left: auto;
    right: 16px;
    bottom: 16px;
    width: 400px;
    border-radius: 16px;
    max-height: 76vh;
  }
  .detail-panel {
    left: auto;
    right: 16px;
    bottom: 16px;
    width: 400px;
    border-radius: 16px;
    max-height: 76vh;
  }
  .panel-enter-from,
  .panel-leave-to {
    transform: translateY(20px);
    opacity: 0;
  }
  .count-badge { bottom: 16px; }
  .subtitle { display: block; }
}
</style>
