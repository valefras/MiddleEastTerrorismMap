<template>
  <div class="h-screen max-h-screen relative">
    <div v-if="loading" class="text-xl">loading...</div>

    <div id="map" class="h-full w-full z-0"></div>

    <AttackInfo :attackData="attackData" v-if="showInfo" />
    <transition>
      <div
        class="absolute m-4 top-0 left-0 text-gray-800 dark:text-gray-200 text-6xl font-bold"
        v-if="zoom < 11"
      >
        Terrorist attacks in the Middle East (1990-2022)
      </div>
      <div
        class="absolute m-4 top-0 left-0 bg-gray-800 p-2 text-gray-200 flex flex-col items-start rounded"
        v-else
      >
        <div>
          <img src="@/assets/yellow.png" class="h-4 w-4 inline-block" />
          0 casualties
        </div>
        <div>
          <img src="@/assets/orange.png" class="h-4 w-4 inline-block" />
          1+ casualties
        </div>
        <div>
          <img src="@/assets/red.png" class="h-4 w-4 inline-block" />
          10+ casualties
        </div>
      </div>
    </transition>
    <div
      class="absolute m-4 bottom-0 left-0 text-gray-800 dark:text-gray-200 text-6xl font-semibold"
    >
      {{ nameHover }}
    </div>

    <div
      class="absolute bottom-8 right-16 dark:text-gray-200 text-gray-800 bg-slate-200 dark:bg-slate-700 rounded-md px-4 py-2 hover:bg-slate-300 dark:hover:bg-slate-800 cursor-pointer shadow-lg dark:shadow-slate-600"
      @click="changeTheme"
    >
      Change map
    </div>
  </div>
  <!-- <HelloWorld msg="Welcome to Your Vue.js App" /> -->

  <!-- <div class="flex absolute z-9 w-full bottom-8 justify-center"></div> -->
</template>

<script>
// @ is an alias to /src
// import HelloWorld from "@/components/HelloWorld.vue";
import AttackInfo from "@/components/AttackInfo.vue";
import L from "leaflet";
import { onMounted, ref, watch } from "vue";
// import axios from "axios";
// import "heatmap.js";
// import HeatmapOverlay from "heatmap.js/plugins/leaflet-heatmap";
// import "overlapping-marker-spiderfier-leaflet/dist/oms";
// import { point, featureCollection } from "@turf/helpers";
// import envelope from "@turf/envelope";
// // import difference from "@turf/difference";
// // import booleanWithin from "@turf/boolean-within";
// import within from "@turf/within";
import geojson_reg from "@/assets/middle_east_reg.geojson";
import geojson_cou from "@/assets/ME_eurostat.geojson";
import orange from "@/assets/orange.png";
import yellow from "@/assets/yellow.png";
import red from "@/assets/red.png";
import attacks from "@/assets/attacks.geojson";
import "leaflet.markercluster/dist/MarkerCluster.css";
import "leaflet.markercluster/dist/MarkerCluster.Default.css";
import "leaflet.markercluster/dist/leaflet.markercluster.js";

export default {
  name: "HomeView",
  components: {
    AttackInfo,
  },

  setup() {
    const nameHover = ref("");
    let map;
    const zoom = ref(5);
    // let bounds;
    // let prev_bounds;
    const loading = ref(false);
    let attackPoints;
    // let layerGroup;
    // let oms;
    let cloroplethLayerReg;
    let cloroplethLayerCou;
    let geojsonDataCou;
    let geojsonDataReg;
    let geojson_markerCluster;

    // let quants;
    // let baseLayer;
    // let data;
    // let prev_bounds;
    // let markers = [];
    const attackData = ref();
    // const attackId = ref(0);
    const showInfo = ref(false);
    let markerClusterLayer;

    var yellow_icon = L.icon({
      iconUrl: yellow,
      iconSize: [20, 20], // size of the icon
      iconAnchor: [10, 10], // point of the icon which will correspond to marker's location
    });
    var red_icon = L.icon({
      iconUrl: red,
      iconSize: [20, 20], // size of the icon
      iconAnchor: [10, 10], // point of the icon which will correspond to marker's location
    });

    var orange_icon = L.icon({
      iconUrl: orange,
      iconSize: [20, 20], // size of the icon
      iconAnchor: [10, 10], // point of the icon which will correspond to marker's location
    });

    function getIconColor(n_casualty) {
      if (n_casualty > 9) {
        return red_icon;
      } else if (n_casualty > 0) {
        return orange_icon;
      } else {
        return yellow_icon;
      }
    }

    function styleReg(feature) {
      return {
        fillColor: getColorReg(feature.properties["n_attacks"]),
        weight: 0.5,
        opacity: 1,
        color: "black",
        fillOpacity: localStorage.theme == "dark" ? 0.8 : 0.7,
      };
    }

    function styleCou(feature) {
      return {
        fillColor: getColorCou(feature.properties["n_attacks"]),
        weight: 0.5,
        opacity: 1,
        color: "black",
        fillOpacity: localStorage.theme == "dark" ? 0.8 : 0.7,
      };
    }

    function getColorCou(d) {
      const naturalBreaks = [0, 170, 536, 1097, 2081, 2849, 27311];
      const colors = [
        "#ffffb2",
        "#fed976",
        "#feb24c",
        "#fd8d3c",
        "#f03b20",
        "#bd0026",
      ];
      return d > naturalBreaks[5]
        ? colors[5]
        : d > naturalBreaks[4]
        ? colors[4]
        : d > naturalBreaks[3]
        ? colors[3]
        : d > naturalBreaks[2]
        ? colors[2]
        : d > naturalBreaks[1]
        ? colors[1]
        : colors[0];
    }

    function getColorReg(d) {
      const naturalBreaks = [0, 67, 225, 429, 697, 972, 1362, 2291, 3507, 8067];
      const colors = [
        "#ffffcc",
        "#ffeda0",
        "#fed976",
        "#feb24c",
        "#fd8d3c",
        "#fc4e2a",
        "#e31a1c",
        "#bd0026",
        "#800026",
      ];
      return d > naturalBreaks[8]
        ? colors[8]
        : d > naturalBreaks[7]
        ? colors[7]
        : d > naturalBreaks[6]
        ? colors[6]
        : d > naturalBreaks[5]
        ? colors[5]
        : d > naturalBreaks[4]
        ? colors[4]
        : d > naturalBreaks[3]
        ? colors[3]
        : d > naturalBreaks[2]
        ? colors[2]
        : d > naturalBreaks[1]
        ? colors[1]
        : colors[0];
    }

    let baseLayerDark = L.tileLayer(
      "https://tiles.stadiamaps.com/tiles/alidade_smooth_dark/{z}/{x}/{y}{r}.png",
      {
        maxZoom: 16,
        minZoom: 5,
        attribution:
          '&copy; <a href="https://stadiamaps.com/">Stadia Maps</a>, &copy; <a href="https://openmaptiles.org/">OpenMapTiles</a> &copy; <a href="http://openstreetmap.org">OpenStreetMap</a> contributors',
      }
    );
    let baseLayerLight = L.tileLayer(
      "https://tiles.stadiamaps.com/tiles/alidade_smooth/{z}/{x}/{y}{r}.png",
      {
        maxZoom: 16,
        minZoom: 5,
        attribution:
          '&copy; <a href="https://stadiamaps.com/">Stadia Maps</a>, &copy; <a href="https://openmaptiles.org/">OpenMapTiles</a> &copy; <a href="http://openstreetmap.org">OpenStreetMap</a> contributors',
      }
    );

    function changeTheme() {
      if (localStorage.theme == "dark") {
        document.body.classList.remove("dark");
        localStorage.theme = "light";
        map.removeLayer(baseLayerDark);
        map.addLayer(baseLayerLight);
      } else {
        document.body.classList.add("dark");
        localStorage.theme = "dark";

        map.removeLayer(baseLayerLight);
        map.addLayer(baseLayerDark);
      }
    }
    async function setTheme() {
      if (localStorage.theme == "dark") {
        document.body.classList.add("dark");
        map.addLayer(baseLayerDark);
      } else {
        document.body.classList.remove("dark");
        map.addLayer(baseLayerLight);
      }
    }
    onMounted(async () => {
      loading.value = true;
      attackPoints = await attacks;
      geojsonDataReg = await geojson_reg;
      geojsonDataCou = await geojson_cou;
      // cloroDataReg = await createCloroData("corr_provstate");
      // cloroDataCou = await createCloroData("corr_country");
      loading.value = false;

      map = L.map("map", {
        // zoomSnap: 0.25,
        // wheelPxPerZoomLevel: 150,
        // zoomDelta: 0.25,
        // wheelPxPerZoomLevel: 1200,

        maxBounds: [
          [6.567611981603534, 70.18993949091978],

          [43.35145642514104, 24.25375932486969],
        ],
        zoomControl: false,

        layers: [],
      }).setView([27, 38.25], 5);

      L.control
        .zoom({
          position: "bottomright",
          zoomDelta: 0.1,
          zoomSnap: 0,
          // zoomDelta: 0.1,
          // zoomSnap: 0,
        })
        .addTo(map);

      await setTheme();

      geojson_markerCluster = L.geoJson(attackPoints, {
        pointToLayer: function (feature, latlng) {
          return L.marker(latlng, {
            icon: getIconColor(parseInt(feature.properties.nkill)),
          });
        },
        onEachFeature: function (feature, layer) {
          layer.on({
            click: function () {
              console.log(feature.properties);
              if (showInfo.value) {
                showInfo.value = false;
                attackData.value = {};
              }
              attackData.value = feature.properties;
              showInfo.value = true;
            },
          });
        },
      });

      markerClusterLayer = L.markerClusterGroup();

      markerClusterLayer.addLayer(geojson_markerCluster);

      cloroplethLayerReg = L.geoJson(geojsonDataReg, {
        style: styleReg,
        onEachFeature: function (feature, layer) {
          layer.on({
            mouseover: function (e) {
              let region = e.target.feature.properties["NAME_1"];
              let att_num = e.target.feature.properties["n_attacks"];
              nameHover.value = region + ": " + att_num + " attacks";
            },
            mouseout: function () {
              nameHover.value = "";
            },
            click: function (e) {
              map.fitBounds(e.target.getBounds());
            },
          });
        },
      });

      // cloroplethLayerReg.addTo(map);

      cloroplethLayerCou = L.geoJson(geojsonDataCou, {
        style: styleCou,
        onEachFeature: function (feature, layer) {
          layer.on({
            mouseover: function (e) {
              let country = e.target.feature.properties["NAME_ENGL"];
              let att_num = e.target.feature.properties["n_attacks"];
              nameHover.value = country + ": " + att_num + " attacks";
            },
            mouseout: function () {
              nameHover.value = "";
            },
            click: function (e) {
              map.fitBounds(e.target.getBounds());
            },
          });
        },
      });

      cloroplethLayerCou.addTo(map);

      map.on("zoomend", async function () {
        // bounds = await map.getBounds();
        // prev_bounds = bounds;

        zoom.value = map.getZoom();
        console.log(zoom.value);
      });
    });
    watch(zoom, async () => {
      if (zoom.value < 11) {
        showInfo.value = false;
      }
      if (zoom.value < 6) {
        if (cloroplethLayerReg != null) {
          map.removeLayer(cloroplethLayerReg);
        }
        map.addLayer(cloroplethLayerCou);
      } else if (zoom.value <= 10 && zoom.value > 5) {
        console.log("display cloro");
        // if (layerGroup != null) {
        //   map.removeLayer(layerGroup);
        // }
        map.removeLayer(cloroplethLayerCou);

        if (markerClusterLayer != null) {
          map.removeLayer(markerClusterLayer);
        }
        // if (cloroplethLayerReg == null) {
        map.addLayer(cloroplethLayerReg);
        // }

        // map.addLayer(heatmapLayer);
        // map.removeLayer(layerGroup);
      } else {
        // await displayAttackMarkers();
        map.removeLayer(cloroplethLayerReg);

        console.log("display markers");

        map.addLayer(markerClusterLayer);

        // getAttacks();
      }
      console.log("zoom changed");
    });
    return {
      showInfo,
      attackData,
      nameHover,
      loading,
      zoom,
      changeTheme,
    };
  },
};
</script>
<style>
/* we will explain what these classes do next! */
.v-enter-active,
.v-leave-active {
  transition: opacity 1s ease;
}

.v-enter-from,
.v-leave-to {
  opacity: 0;
}
</style>
