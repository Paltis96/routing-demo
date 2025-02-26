<template>
  <div class="pa-4">
    <v-select
      variant="outlined"
      v-model="searchMode"
      label="Search mode"
      :items="['id', 'coord']"
    ></v-select>
    <v-text-field
      @keyup.enter="getPoiInRadius"
      :loading="loading"
      @click:clear="errorMessages = ''"
      clearable
      v-model.trim="query"
      label="Search"
      :placeholder="searchMode == 'id' ? 'Enter location_id' : 'Enter coord'"
      :error-messages="errorMessages"
      variant="outlined"
    ></v-text-field>
    <v-card-title v-if="routesLen > 0">Routes: {{ routesLen }}</v-card-title>

    <v-card-text v-if="routesLen > 0">
      <v-table
        hover
        fixed-header
        height="50vh"
        @mouseleave="hoverId = ''"
        class="mb-4"
      >
        <thead>
          <tr>
            <th class="text-left">
              <v-checkbox
                density="compact"
                hide-details="auto"
                v-model="allSelected"
                @click="togleSelection"
              ></v-checkbox>
            </th>
            <th class="text-left">To point</th>
            <th class="text-left">Name</th>
            <th class="text-left">type</th>
            <th class="text-left">length km</th>
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="item in routesProps"
            :key="item.name"
            @mouseover="hoverId = item.location_id"
          >
            <td>
              <v-checkbox
                density="compact"
                hide-details="auto"
                v-model="selectedRoutesId"
                :value="item.location_id"
              ></v-checkbox>
            </td>
            <td>{{ item.location_id }}</td>
            <td>{{ item.location_name }}</td>
            <td>{{ item.location_type }}</td>
            <td>{{ item.length_km }}</td>
          </tr>
        </tbody>
      </v-table>
      <v-btn @click="downloadContent" color="blue  elevation-0 ">
        Export to kml {{ selectedRoutesLen }}</v-btn
      >
    </v-card-text>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, computed, watch } from "vue";
import { useAppStore } from "@/store/app";
import axios from "axios";
import { geomEach } from "@turf/meta";
// @ts-ignore
import { saveAs } from "file-saver";
// @ts-ignore
import tokml from "@maphubs/tokml";

const appStore = useAppStore();
const query = ref("");
const loading = ref(false);
const selectedRoutesId: any = ref([]);
const hoverId = ref("");
const errorMessages = ref("");
const searchMode = ref("id");
let routes: any = reactive({});

const routesLen = computed(() => {
  return routes.features ? routes.features.length : 0;
});
const selectedRoutesLen = computed(() => {
  return selectedRoutesId.value.length;
});
const allSelected = computed(() => {
  return routesLen.value == selectedRoutesLen.value;
});
const routesProps = computed(() => {
  return routes.features.map((feature: any) => feature.properties);
});
const selectedRowsGeom = computed(() => {
  const output = { ...routes };
  output.features = output.features.filter((prop: any) => {
    const id = prop.properties.location_id;
    return selectedRoutesId.value.includes(id);
  });
  return output;
});

watch(selectedRoutesId, (newId: string[]) => {
  appStore.updateRoutesFilter(newId);
});

watch(hoverId, (newId: string) => {
  appStore.updateHoverFilter(newId);
});
const togleSelection = () => {
  selectedRoutesId.value =
    selectedRoutesId.value.length > 0
      ? []
      : routesProps.value.map((item: any) => item.location_id);
};

function getRoutes(lon: number, lat: number) {
  axios
    .get("/routes", {
      params: { lon, lat },
    })
    .then((new_routes) => {
      routes = Object.assign(routes, new_routes);
      appStore.setSourceData("routes", routes);
    })
    .then(() => {
      selectedRoutesId.value = routesProps.value.map(
        (item: any) => item.location_id
      );
    })
    .catch((err) => {
      console.log(err);
    })
    .finally(() => {
      loading.value = false;
    });
}

async function getPoiInRadius() {
  errorMessages.value = ''
  let coords: any[];
  appStore.clearMap();
  if (searchMode.value != "id") {
    const loc = query.value.split(",");
    if (loc.length != 2) {
      errorMessages.value = "Unprocessable Entity";
      return;
    }
    axios
      .get("/near-locations-by-coord", {
        params: { lon: loc[0], lat: loc[1] },
      })
      .then((res) => {
        appStore.setSourceData("pois_in_buffer", res);
        coords = loc;
        appStore.setMarker(coords, "");
      })
      .then(() => {
        getRoutes(coords[0], coords[1]);
      })
      .catch((err) => {
        if (err.response.data.detail)
          errorMessages.value = err.response.data.detail;
        else errorMessages.value = "Error";
        loading.value = false;
      });
  } else {
    const loc_id = query.value;
    loading.value = true;
    axios
      .get("/near-locations", { params: { id: loc_id } })
      .then((res) => {
        appStore.setSourceData("pois_in_buffer", res);
        geomEach(
          res as any,
          (
            currentGeometry: any,
            featureIndex: number,
            featureProperties: any
          ) => {
            if (featureProperties.location_id == loc_id) {
              coords = currentGeometry.coordinates;
              appStore.setMarker(coords, "");
              return;
            }
          }
        );
      })
      .then(() => {
        getRoutes(coords[0], coords[1]);
      })
      .catch((err) => {
        if (err.response.data.detail)
          errorMessages.value = err.response.data.detail;
        else errorMessages.value = "Error";
        loading.value = false;
      });
  }
}

function downloadContent(key: any, type = "kml") {
  const data = selectedRowsGeom.value;
  let blob;
  switch (type) {
    case "geojson":
      blob = new Blob([JSON.stringify(data)], {
        type: "application/json",
      });
      saveAs(blob, `sample.geojson`);
      break;
    case "kml":
      blob = new Blob([tokml(data)], {
        type: "application/xml",
      });
      saveAs(blob, `sample.kml`);
      break;
    default:
      break;
  }
}
</script>

<style scoped></style>
