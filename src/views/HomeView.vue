<template>
  <div class="h-screen max-h-screen relative">
    <div v-if="loading" class="text-xl">loading...</div>

    <div id="map" class="h-full w-full z-0"></div>

    <AttackInfo :attackData="attackData" v-if="showInfo" />
    <transition>
      <div
        class="absolute m-4 top-0 left-0 text-gray-800 text-8xl font-bold"
        v-if="zoom < 6"
      >
        Terrorist attacks in the Middle East (1990-2022)
      </div>
      <div
        class="absolute m-4 top-0 left-0 bg-gray-100 p-2 text-gray-800 flex flex-col items-start rounded"
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
      class="absolute m-4 bottom-0 left-0 text-gray-800 text-6xl font-semibold"
    >
      {{ nameHover }}
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
    let cloroDataReg;
    let cloroDataCou;
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
    // let cfg = {
    //   // radius should be small ONLY if scaleRadius is true (or small radius is intended)
    //   radius: 1,
    //   maxOpacity: 0.8,
    //   // scales the radius based on map zoom
    //   scaleRadius: true,
    //   // if set to false the heatmap uses the global maximum for colorization
    //   // if activated: uses the data maximum within the current map boundaries
    //   //   (there will always be a red spot with useLocalExtremas true)
    //   useLocalExtrema: true,
    //   // which field name in your data represents the latitude - default "lat"
    //   latField: "lat",
    //   // which field name in your data represents the longitude - default "lng"
    //   lngField: "lng",
    //   // which field name in your data represents the data value - default "value"
    //   valueField: "count",
    // };

    // function fetchJSON(url) {
    //   return fetch(url).then(function (response) {
    //     return response.json();
    //   });
    // }

    // function getAttackInfo(id) {
    //   console.log(id);
    //   axios
    //     .get("http://localhost:3000/api/attackInfo", {
    //       params: {
    //         eventid: id,
    //       },
    //     })
    //     .then((res) => {
    //       attackData.value = res.data;
    //       showInfo.value = true;

    //       console.log(attackData.value);
    //     });
    // }
    // function getCloroData() {
    //   axios
    //     .get("http://localhost:3000/api/")
    //     .then((res) => {
    //       cloroData = res.data.rows;
    //       console.log(cloroData);
    //       // for (let i = 0; i < cloroData.length; i++) {
    //       //   cloroData[i].corr_provstate = parseInt(cloroData[i].corr_provstate);
    //       // }
    //       const resultObject = {};

    //       for (let i = 0; i < cloroData.length; i++) {
    //         const obj = cloroData[i];
    //         const key = obj.corr_provstate;
    //         const value = parseInt(obj.count);

    //         resultObject[key] = value;
    //       }

    //       cloroData = resultObject;

    //       cloroplethLayerReg = L.geoJson(geojsonData, {
    //         style: style,
    //         onEachFeature: function (feature, layer) {
    //           layer.on({
    //             mouseover: function (e) {
    //               let region = e.target.feature.properties["NAME_1"];
    //               let att_num = Object.hasOwn(cloroData, region)
    //                 ? cloroData[e.target.feature.properties["NAME_1"]]
    //                 : 0;
    //               nameHover.value = region + ": " + att_num + " attacks";
    //             },
    //             mouseout: function () {
    //               nameHover.value = "";
    //             },
    //             click: function (e) {
    //               map.fitBounds(e.target.getBounds());
    //             },
    //           });
    //         },
    //       });
    //       cloroplethLayerReg.addTo(map);
    //     })
    //     .catch((err) => {
    //       console.log(err);
    //     });
    // }
    // function getClusteredAttacks() {
    //   axios.get("http://localhost:3000/api/").then((res) => {
    //     data = res.data;
    //     let temp_data = res.data.rows;
    //     for (let i = 0; i < temp_data.length; i++) {
    //       if (heatmapData.max < temp_data[i].weight) {
    //         heatmapData.max = temp_data[i].weight;
    //       }
    //       heatmapData.data.push({
    //         lat: temp_data[i].latitude,
    //         lng: temp_data[i].longitude,
    //         count: temp_data[i].weight,
    //       });
    //     }
    //     heatmapLayer.setData(heatmapData);

    //     // console.log(heatmapData.value);
    //   });
    // }
    function getIconColor(n_casualty) {
      if (n_casualty > 9) {
        return red_icon;
      } else if (n_casualty > 0) {
        return orange_icon;
      } else {
        return yellow_icon;
      }
    }

    // function getAttacks() {
    //   axios
    //     .get("http://localhost:3000/api/markers", {
    //       params: {
    //         sw1: bounds["_southWest"]["lng"],
    //         sw2: bounds["_southWest"]["lat"],
    //         ne1: bounds["_northEast"]["lng"],
    //         ne2: bounds["_northEast"]["lat"],
    //         // zoom: zoom.value,
    //       },
    //     })
    //     .then((res) => {
    //       data = res.data;
    //       // layerGroup = L.layerGroup().addTo(map);

    //       for (let i = 0; i < data.rowCount; i++) {
    //         let marker = L.marker(
    //           [data.rows[i].latitude, data.rows[i].longitude],
    //           {
    //             icon: getIconColor(data.rows[i].nkill),
    //           }
    //         )
    //           .addTo(map)
    //           .bindPopup(
    //             `<b>${data.rows[i].eventid}</b><br>${data.rows[i].new_date}`
    //           );
    //         marker.att_id = data.rows[i].eventid;
    //         markers.push(marker);
    //         // layerGroup.addLayer(marker);
    //         // oms.addMarker(marker);
    //       }
    //     })
    //     .catch((err) => {
    //       console.log(err);
    //     });
    // }

    // async function displayAttackMarkers() {
    //   // markerClusterLayer.clearLayers();

    //   let curr_env = featureCollection([
    //     envelope(
    //       featureCollection([
    //         point([bounds["_southWest"]["lng"], bounds["_southWest"]["lat"]]),
    //         point([bounds["_northEast"]["lng"], bounds["_northEast"]["lat"]]),
    //       ])
    //     ),
    //   ]);
    //   let points_in_view = await within(attackPoints, curr_env);
    //   console.log(points_in_view);
    //   if (points_in_view.features.length > 0) {
    //     for (let i = 0; i < points_in_view.features.length; i++) {
    //       console.log("point added");
    //       let marker = L.marker(
    //         [
    //           points_in_view.features[i].geometry.coordinates[1],
    //           points_in_view.features[i].geometry.coordinates[0],
    //         ],
    //         {
    //           icon: getIconColor(
    //             parseInt(points_in_view.features[i].properties.nkill)
    //           ),
    //         }
    //       ).on("click", function () {
    //         // map.fitBounds(e.target.getBounds());
    //         // console.log(e.target.att_id);
    //         // getAttackInfo(e.target.att_id);
    //         if (showInfo.value) {
    //           showInfo.value = false;
    //           attackData.value = {};
    //         }

    //         attackData.value = points_in_view.features[i].properties;
    //         showInfo.value = true;
    //       });
    //       markers.push(marker);

    //       markerClusterLayer.addLayer(marker);
    //     }
    //     // markerClusterLayer.addTo(map);
    //   }
    // }
    function styleReg(feature) {
      // console.log(feature.properties["NAME_1"], cloroData);
      return {
        fillColor: getColor(cloroDataReg[feature.properties["NAME_1"]]),
        weight: 2,
        opacity: 1,
        color: "white",
        dashArray: "3",
        fillOpacity: 0.7,
      };
    }

    function styleCou(feature) {
      // console.log(feature.properties["NAME_1"], cloroData);
      return {
        fillColor: getColor(cloroDataCou[feature.properties["NAME_ENGL"]]),
        weight: 2,
        opacity: 1,
        color: "white",
        dashArray: "3",
        fillOpacity: 0.7,
      };
    }

    function getColor(d) {
      return d > 5000
        ? "#FF0000"
        : d > 2000
        ? "#FF3300"
        : d > 500
        ? "#FF6600"
        : d > 200
        ? "#FF9900"
        : d > 100
        ? "#FFCC00"
        : d > 50
        ? "#FFFF00"
        : d > 10
        ? "#FFFF33"
        : "#FFEDA0";
    }
    async function createCloroData(mode) {
      let toRtn = {};
      for (let i = 0; i < attackPoints.features.length; i++) {
        if (Object.hasOwn(toRtn, attackPoints.features[i].properties[mode])) {
          toRtn[attackPoints.features[i].properties[mode]] += 1;
        } else {
          toRtn[attackPoints.features[i].properties[mode]] = 1;
        }
      }
      return toRtn;
    }
    onMounted(async () => {
      loading.value = true;
      attackPoints = await attacks;
      geojsonDataReg = await geojson_reg;
      geojsonDataCou = await geojson_cou;
      cloroDataReg = await createCloroData("corr_provstate");
      cloroDataCou = await createCloroData("corr_country");

      loading.value = false;

      // geojson_reg = axios
      //   .get("../assets/middle_east.geojson")
      //   .then((res) => console.log(res.data))
      //   .catch((err) => console.log(err));
      // console.log(geojson_reg);
      let baseLayer = L.tileLayer(
        "https://tile.openstreetmap.org/{z}/{x}/{y}.png",
        {
          maxZoom: 16,
          minZoom: 5,

          attribution:
            '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        }
      );

      // getCloroData();

      map = L.map("map", {
        // zoomSnap: 0.25,
        // wheelPxPerZoomLevel: 150,
        // zoomDelta: 0.25,
        // wheelPxPerZoomLevel: 1200,

        maxBounds: [
          [11.567611981603534, 64.18993949091978],

          [40.35145642514104, 32.25375932486969],
        ],
        zoomControl: false,

        layers: [baseLayer],
      }).setView([27, 38.25], 5);

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
              let att_num = Object.hasOwn(cloroDataReg, region)
                ? cloroDataReg[e.target.feature.properties["NAME_1"]]
                : 0;
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
              let att_num = Object.hasOwn(cloroDataCou, country)
                ? cloroDataCou[e.target.feature.properties["NAME_ENGL"]]
                : 0;
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

      // L.control
      //   .zoom({
      //     position: "bottomright",
      //   })
      //   .addTo(map);
      // markerClusterLayer = L.markerClusterGroup().addTo(map);

      // for (let i = 0; i < attackPoints.features.length; i++) {
      //   // console.log(booleanWithin(attackPoints.features[i], curr_env));
      //   // if (booleanWithin(attackPoints.features[i], curr_env)) {
      //   console.log("point added");
      //   let marker = L.marker(
      //     [
      //       attackPoints.features[i].geometry.coordinates[1],
      //       attackPoints.features[i].geometry.coordinates[0],
      //     ],
      //     {
      //       icon: getIconColor(
      //         parseInt(attackPoints.features[i].properties.nkill)
      //       ),
      //     }
      //   );
      //   markers.push(marker);
      //   markerClusterLayer.addLayer(marker);
      //   // }
      // }

      // bounds = await map.getBounds();

      // var popup = new L.Popup();

      // oms.addListener("click", function (marker) {
      //   showInfo.value = false;
      //   getAttackInfo(marker.att_id);

      //   // popup.setContent(marker.desc);
      //   // popup.setLatLng(marker.getLatLng());
      //   // map.openPopup(popup);
      // });

      // map.on("dragend", async function () {
      //   // if (zoom.value > 11) {
      //   //   await displayAttackMarkers();
      //   //   map.addLayer(markerClusterLayer);
      //   // }
      //   if (zoom.value > 11) {
      //     prev_bounds = await bounds;
      //     bounds = await map.getBounds();

      //     let envel = envelope(
      //       featureCollection([
      //         point([bounds["_southWest"]["lat"], bounds["_southWest"]["lng"]]),
      //         point([bounds["_northEast"]["lat"], bounds["_northEast"]["lng"]]),
      //       ])
      //     );

      //     let prev_envel = envelope(
      //       featureCollection([
      //         point([
      //           prev_bounds["_southWest"]["lat"],
      //           prev_bounds["_southWest"]["lng"],
      //         ]),
      //         point([
      //           prev_bounds["_northEast"]["lat"],
      //           prev_bounds["_northEast"]["lng"],
      //         ]),
      //       ])
      //     );

      //     let diff = difference(envel, prev_envel);

      //     let points_to_check = featureCollection(
      //       markers.map((marker) => {
      //         return marker.toGeoJSON();
      //       })
      //     );
      //     console.log(points_to_check);

      //     let to_del = within(points_to_check, featureCollection([diff]));

      //     // let diff =
      //     for (let i = 0; i < to_del.length; i++) {
      //       map.removeLayer(markers[i]);
      //       markers.splice(i, 1);
      //       console.log("removed");
      //     }

      //     console.log(bounds);
      //   }
      // });

      map.on("zoomend", async function () {
        // bounds = await map.getBounds();
        // prev_bounds = bounds;

        zoom.value = map.getZoom();
        console.log(zoom.value);
      });
    });
    watch(zoom, async () => {
      if (zoom.value == 5) {
        showInfo.value = false;
      }
      if (zoom.value <= 6) {
        if (cloroplethLayerReg != null) {
          map.removeLayer(cloroplethLayerReg);
        }
        map.addLayer(cloroplethLayerCou);
      } else if (zoom.value <= 10 && zoom.value > 6) {
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
