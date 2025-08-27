<script setup lang="ts">
import { ref, onMounted } from "vue";
import { Loader } from "@googlemaps/js-api-loader";

const mapEl = ref<HTMLDivElement | null>(null);
let map: google.maps.Map | null = null;
let activePath: google.maps.LatLngLiteral[] = [];
let tempPolygon: google.maps.Polygon | null = null;
const polygons = ref<google.maps.Polygon[]>([]);
const isDrawing = ref(false);

async function initMap() {
  const loader = new Loader({
    apiKey: "AIzaSyA8b_VjlyxIZK9SzLNwHDXmUB3Uo2ahAmA",
    version: "weekly",
    libraries: ["geometry"],
  });

  const google = await loader.load();

  map = new google.maps.Map(mapEl.value as HTMLElement, {
    center: { lat: 46.25079, lng: 20.1302 }, // Budapest
    zoom: 12,
  });
}

function startDrawing() {
  if (!map) return;
  isDrawing.value = true;
  activePath = [];

  tempPolygon = new google.maps.Polygon({
    map,
    path: [],
    strokeColor: "#FF0000",
    strokeOpacity: 0.8,
    strokeWeight: 2,
  });

  map.addListener("click", handleMapClick);
  map.addListener("rightclick", finishDrawing); // jobb klikk = befejezés
}

function handleMapClick(e: google.maps.MapMouseEvent) {
  if (!e.latLng) return;
  const point = { lat: e.latLng.lat(), lng: e.latLng.lng() };
  activePath.push(point);
  tempPolygon!.setPath(activePath);
}

function finishDrawing() {
  if (!map || activePath.length < 3) {
    resetDrawing();
    return;
  }

  const polygon = new google.maps.Polygon({
    paths: activePath,
    map,
    fillColor: "#FF0000",
    fillOpacity: 0.35,
    strokeColor: "#FF0000",
    strokeWeight: 2,
    editable: true,
  });

  polygons.value.push(polygon);
  console.log("Polygon kész:", activePath);

  resetDrawing();
}

function resetDrawing() {
  isDrawing.value = false;
  activePath = [];
  if (tempPolygon) {
    tempPolygon.setMap(null);
    tempPolygon = null;
  }
  google.maps.event.clearListeners(map!, "click");
  google.maps.event.clearListeners(map!, "rightclick");
}

onMounted(() => {
  initMap();
});
</script>

<template>
  <div class="relative w-full h-[600px]">
    <div class="flex gap-3">
      <button
        class="z-10 ml-[300px] bg-green-600 text-white px-4 py-2 rounded shadow"
        :disabled="isDrawing"
        @click="startDrawing"
      >
        Új zóna rajzolása
      </button>
      <button
        class="z-10 ml-[300px] bg-red-600 text-white px-4 py-2 rounded shadow"
        :disabled="!isDrawing"
        @click="finishDrawing"
      >
        Befejezés
      </button>
    </div>

    <div ref="mapEl" class="w-full h-full rounded-xl shadow" />
  </div>
</template>
