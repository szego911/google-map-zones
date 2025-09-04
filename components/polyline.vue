<script setup lang="ts">
import { Loader } from "@googlemaps/js-api-loader";
import { markRaw } from "vue";

const mapEl = ref<HTMLDivElement | null>(null);
const map = shallowRef<google.maps.Map | null>(null);
const polyline = shallowRef<google.maps.Polyline | null>(null);





const drawingMode = (deliveryZone: deliveryZone) => {
    if (!map.value) return

    newDeliveryZone.value = new google.maps.Polyline({
      map: map.value,
      path: newDeliveryZonePath.value,
      geodesic: false,
      strokeColor: '#FF00FF',
      strokeOpacity: 1.0,
      strokeWeight: 2,
      editable: true,
      clickable: false,
      zIndex: 2,
    })

    drawingDeliveryZone.value = deliveryZone

    clickHandle = map.value.addListener('click', addPointToPolyline)
    dblClickHandle = map.value.addListener('dblclick', console.log('Double click on drawing mode'))
  }














const { $maps } = useNuxtApp();

// A vonal pontjai (LatLngLiteral[])
const path = ref<google.maps.LatLngLiteral[]>([
  { lat: 46.252, lng: 20.141 },
  { lat: 46.256, lng: 20.155 },
  { lat: 46.26, lng: 20.17 },
]);

onMounted(async () => {
  const loader = new Loader({
    apiKey: "AIzaSyA8b_VjlyxIZK9SzLNwHDXmUB3Uo2ahAmA",
    version: "weekly",
    libraries: ["geometry"],
  });

  const google = await loader.load();

  map.value = new google.maps.Map(mapEl.value, {
    center: { lat: 46.256, lng: 20.155 },
    zoom: 13,
  });

  // Polyline létrehozása és megjelenítése
  const pl = new google.maps.Polyline({
    map: map.value, // <-- fontos: a tényleges Map példány
    path: path.value, // LatLngLiteral[] vagy MVCArray
    strokeColor: "#1E88E5",
    strokeOpacity: 0.9,
    strokeWeight: 3,
    geodesic: true,
    clickable: true, // eseményekhez
    zIndex: 2,
  });

  polyline.value = markRaw(pl); // ne tedd reaktívvá

  map.value.addListener("click", addPoint);
});

// Új pont hozzáadása
function addPoint(p: google.maps.MapMouseEvent) {
  path.value.push(p.latLng);
  polyline.value?.setPath(path.value);
}

// Eltávolítás a térképről
function removePolyline() {
  polyline.value?.setMap(null);
  polyline.value = null;
}
</script>

<template>
  <div ref="mapEl" style="height: 500px"></div>

  <v-btn class="tw-mt-2" @click="addPoint({ lat: 46.265, lng: 20.183 })">
    Új pont hozzáadása
  </v-btn>
  <v-btn
    class="tw-mt-2"
    color="error"
    variant="outlined"
    @click="removePolyline"
  >
    Polyline eltávolítása
  </v-btn>
</template>






