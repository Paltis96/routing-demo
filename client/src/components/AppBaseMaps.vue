<template>
  <v-row class="ma-2">
    <v-col cols="12">
      <v-card class="pt-2 rounded-lg elevation-0">
        <v-card-title>Current base map</v-card-title>
        <v-card-text>
          <v-list class="rounded-lg bg-grey-lighten-4">
            <v-list-item
              :title="appStore.mapsTileSyles.title"
              :prepend-avatar="appStore.mapsTileSyles.img"
            >
            </v-list-item>
          </v-list>
        </v-card-text>
      </v-card>
    </v-col>

    <v-col cols="12">
      <v-card class="rounded-lg elevation-0">
        <v-text-field
          @keyup.enter="getBaseMapList"
          @click:clear="getBaseMapList"
          :loading="loading"
          clearable
          v-model="query"
          label="Basemaps search"
          class="ma-4"
          placeholder="Enter base map provider name"
          variant="outlined"
          hide-details
        ></v-text-field>
        <v-card-title v-if="baseMapList.length > 0"
          >Base maps: {{ baseMapList.length }}
        </v-card-title>
        <v-card-text v-if="baseMapList.length > 0">
          <v-list>
            <v-list-item
              v-for="item in baseMapList"
              :key="item.id"
              :title="item.name"
              :value="item"
              :subtitle="item.cumulative_status"
              :prepend-avatar="item.icon"
              active-color="primary"
              @click="setBaseMap(item)"
            >
            </v-list-item> </v-list></v-card-text
      ></v-card>
    </v-col>
  </v-row>
</template>

<script setup lang="ts">
import { ref } from "vue";
import axios from "axios";
import { useAppStore } from "@/store/app";

const appStore = useAppStore();

const query = ref("");
const baseMapList: any = ref([]);
const loading = ref(false);

function setBaseMap(e: any) {
  appStore.setBaseMap(e);
}

async function getBaseMapList() {
  const re = /{x}|{y}|{z}/;
  baseMapList.value = [];
  if (query.value.length == 0) return;
  loading.value = true;
  axios
    .get(
      `https://qms.nextgis.com/api/v1/geoservices/?type=tms&?cumulative_status=works&search=${query.value}`
    )
    .then((res: any) => {
      res.forEach(async (element: any) => {
        axios
          .get(`https://qms.nextgis.com/api/v1/geoservices/${element.id}/`)
          .then((res: any) => {
            if (res.url.search(re) !== -1) {
              res.url = res.url.replace("http:", "https:");
              res.icon =
                res.icon === null
                  ? "https://cdn-icons-png.flaticon.com/512/831/831295.png"
                  : `https://qms.nextgis.com/api/v1/icons/${res.icon}/content`;
              baseMapList.value.push(res);
            }
          })
          .catch((err) => {
            console.log(err);
          });
      });
    })
    .catch((err) => {
      console.log(err);
    })
    .finally(() => {
      loading.value = false;
    });
}
</script>

<style scoped></style>
