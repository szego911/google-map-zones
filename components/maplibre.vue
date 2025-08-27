<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref } from "vue";
import maplibregl from "maplibre-gl";
import MapboxDraw from "maplibre-gl-draw";

type ZoneProps = { name?: string; color?: string; fee?: number };

const mapEl = ref<HTMLDivElement | null>(null);
let map: maplibregl.Map | null = null;
let draw: MapboxDraw | null = null;

const selectedId = ref<string | null>(null);
const propsForm = ref<ZoneProps>({ name: "", color: "#fc743a", fee: 0 });

function applyPropsToFeature(id: string, props: ZoneProps) {
  console.log(props);
  if (!map || !draw) return;
  const f = draw.get(id);
  if (!f) return;
  f.properties = { ...(f.properties || {}), ...props };
}

function exportGeoJSON() {
  if (!draw) return;
  const data = draw.getAll();
  console.log(data);
  const blob = new Blob([JSON.stringify(data, null, 2)], {
    type: "application/json",
  });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = "zones.geojson";
  a.click();
  URL.revokeObjectURL(url);
}

function importGeoJSON(e: Event) {
  const file = (e.target as HTMLInputElement).files?.[0];
  if (!file || !draw) return;
  file.text().then((txt) => {
    const geo = JSON.parse(txt);
    draw!.deleteAll();
    draw!.add(geo);
  });
}

onMounted(() => {
  // minimál stílus OSM raszterrel (API-kulcs nélkül)
  const style: maplibregl.StyleSpecification = {
    version: 8,
    sources: {
      osm: {
        type: "raster",
        tiles: [
          "https://a.tile.openstreetmap.org/{z}/{x}/{y}.png",
          "https://b.tile.openstreetmap.org/{z}/{x}/{y}.png",
          "https://c.tile.openstreetmap.org/{z}/{x}/{y}.png",
        ],
        tileSize: 256,
        attribution: "© OpenStreetMap contributors",
      },
    },
    layers: [{ id: "osm", type: "raster", source: "osm" }],
  };

  map = new maplibregl.Map({
    container: mapEl.value as HTMLDivElement,
    style,
    center: [20.149, 46.253],
    zoom: 12,
  });

  draw = new MapboxDraw({
    displayControlsDefault: false,
    controls: {
      polygon: true,
      trash: true,
      combine_features: false,
      uncombine_features: false,
    },
    styles: [
      // fill
      {
        id: "gl-draw-polygon-fill-inactive",
        type: "fill",
        filter: ["all", ["==", "active", "false"], ["==", "$type", "Polygon"]],
        paint: { "fill-color": ["get", "color"], "fill-opacity": 0.25 },
      },
      {
        id: "gl-draw-polygon-fill-active",
        type: "fill",
        filter: ["all", ["==", "active", "true"], ["==", "$type", "Polygon"]],
        paint: { "fill-color": ["get", "color"], "fill-opacity": 0.35 },
      },
      {
        id: "gl-draw-polygon-stroke-active",
        type: "line",
        filter: ["all", ["==", "active", "true"], ["==", "$type", "Polygon"]],
        paint: { "line-color": ["get", "color"], "line-width": 2 },
      },
      {
        id: "gl-draw-polygon-stroke-inactive",
        type: "line",
        filter: ["all", ["==", "active", "false"], ["==", "$type", "Polygon"]],
        paint: { "line-color": ["get", "color"], "line-width": 2 },
      },
    ],
  });

  map.addControl(draw as any, "top-left");
  map.addControl(new maplibregl.NavigationControl(), "top-left");

  map.on("draw.create", (e: any) => {
    const id = e.features[0].id;
    selectedId.value = String(id);
    const defaults: ZoneProps = { name: "Új zóna", color: "#fc743a", fee: 0 };
    draw!.setFeatureProperty(id, "name", defaults.name);
    draw!.setFeatureProperty(id, "color", defaults.color);
    draw!.setFeatureProperty(id, "fee", defaults.fee);
    propsForm.value = { ...defaults };
  });

  map.on("draw.selectionchange", (e: any) => {
    const f = e.features?.[0];
    if (!f) {
      selectedId.value = null;
      return;
    }
    selectedId.value = String(f.id);
    propsForm.value = {
      name: f.properties?.name || "",
      color: f.properties?.color || "#fc743a",
      fee: Number(f.properties?.fee || 0),
    };
  });
});

onBeforeUnmount(() => {
  map?.remove();
});

function startDrawing() {
  if (draw) {
    draw.changeMode("draw_polygon");
  }
}
</script>

<template>
  <div class="flex gap-4">
    <div ref="mapEl" class="flex-1 h-[800px] rounded-2xl shadow-lg" />

    <div class="w-80 flex flex-col gap-3">
      <h3 class="text-lg font-semibold">Zóna beállítások</h3>

      <div v-if="selectedId" class="space-y-2">
        <label class="block text-sm">Név</label>
        <input
          v-model="propsForm.name"
          class="w-full border rounded px-3 py-2"
        />

        <label class="block text-sm">Szín</label>
        <input
          v-model="propsForm.color"
          type="color"
          class="h-10 w-16 p-0 border rounded"
        />

        <label class="block text-sm">Kiszállítási díj (HUF)</label>
        <input
          v-model.number="propsForm.fee"
          type="number"
          min="0"
          class="w-full border rounded px-3 py-2"
        />

        <button
          class="w-full bg-black text-white rounded-xl py-2 font-medium"
          @click="applyPropsToFeature(selectedId!, propsForm)"
        >
          Mentés a kijelölt zónára
        </button>
      </div>

      <div class="h-px bg-gray-200"></div>

      <button
        class="w-full bg-gray-900 text-white rounded-xl py-2"
        @click="exportGeoJSON"
      >
        Export GeoJSON
      </button>

      <pre>{{ draw?.getAll() }}</pre>

      <button
        class="w-full bg-green-600 text-white rounded-xl py-2"
        @click="startDrawing"
      >
        Új zóna rajzolása
      </button>

      <label class="text-sm">Import GeoJSON</label>
      <input type="file" accept=".json,.geojson" @change="importGeoJSON" />
    </div>
  </div>
</template>

<style scoped>
@import "maplibre-gl/dist/maplibre-gl.css";
@import "maplibre-gl-draw/dist/mapbox-gl-draw.css";
</style>
