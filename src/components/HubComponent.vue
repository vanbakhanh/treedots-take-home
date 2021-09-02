<template>
  <div class="map">
    <GoogleMap
      style="width: 100%; height: 100%"
      ref="mapRef"
      :api-key="store.state.key"
      :center="store.state.centerLocation.position"
      :zoom="store.state.zoom"
    >
      <Marker
        :options="store.state.myLocation"
        :key="
          store.state.myLocation.position.lat +
          store.state.myLocation.position.lng
        "
      />
      <Marker
        v-for="(marker, index) in store.state.mapMarkers"
        :key="index"
        :options="{
          position: marker.position,
          label: {
            text: marker.label,
            color: !marker.active ? 'white' : 'transparent',
          },
          icon: marker.icon,
          title: marker.title,
        }"
        :clickable="true"
        @click="onMapMarkerClick(index)"
      />
    </GoogleMap>
  </div>
  <q-list bordered separator>
    <q-item>
      <q-item-section>
        <span class="text-weight-medium text-h6 text-dark">Hubs Near You</span>
      </q-item-section>
    </q-item>
  </q-list>
  <q-scroll-area class="list">
    <q-list bordered separator>
      <div v-for="(marker, index) in store.state.mapMarkers" :key="index">
        <q-item
          clickable
          bordered
          v-ripple
          v-if="!marker.active"
          @click="onMapMarkerClick(index)"
        >
          <q-item-section>
            <q-item-label caption>{{ marker.name }}</q-item-label>
          </q-item-section>
          <q-item-section avatar>
            <q-avatar size="24px" color="orange" text-color="white">
              {{ marker.label }}
            </q-avatar>
          </q-item-section>
        </q-item>
      </div>
    </q-list>
  </q-scroll-area>
</template>

<script>
import { defineComponent, onMounted, ref, watchEffect } from 'vue';
import { GoogleMap, Marker } from 'vue3-google-map';
import Vuex from 'vuex';
import hubs from './../data/hubs.json';
import MarkerSVG from '../assets/hub_marker.svg';
import MarkerNoLabelSVG from '../assets/hub_marker_no_label.svg';
import haversine from 'haversine-distance';
export default defineComponent({
  name: 'Hub',
  components: {
    GoogleMap,
    Marker,
  },

  setup() {
    const mapRef = ref(null);

    const store = new Vuex.Store({
      state: {
        centerLocation: {
          position: { lat: 0, lng: 0 },
          icon: null,
        },
        myLocation: {
          position: { lat: 0, lng: 0 },
          icon: null,
        },
        zoom: 15,
        key: process.env.GOOGLE_MAP_KEY,
        mapMarkers: [],
        mapMarkerIcon: MarkerSVG,
        mapMarkerIconNoLabel: MarkerNoLabelSVG,
      },
      mutations: {
        initMapMarkers(state, mapMarkers) {
          state.mapMarkers = mapMarkers;
        },
        updateMapMarkers(state, data) {
          if (state.mapMarkers[data.id].active === true) {
            state.mapMarkers[data.id].active = false;
            state.mapMarkers[data.id].icon = state.mapMarkerIcon;
          } else {
            // update active marker
            for (let i = 0; i < state.mapMarkers.length; i++) {
              state.mapMarkers[i].active = false;
              state.mapMarkers[i].icon = state.mapMarkerIcon;
            }
            state.mapMarkers[data.id].active = data.active;
            state.mapMarkers[data.id].icon = state.mapMarkerIconNoLabel;

            // update center position
            state.centerLocation.position.lat =
              state.mapMarkers[data.id].position.lat;
            state.centerLocation.position.lng =
              state.mapMarkers[data.id].position.lng;
            mapRef.value.map.panTo({
              lat: state.centerLocation.position.lat,
              lng: state.centerLocation.position.lng,
            });
          }
        },
        initMyLocation(state, data) {
          state.myLocation.position.lat = data.lat;
          state.myLocation.position.lng = data.lng;
        },
      },
    });

    const initCurrentPosition = () => {
      navigator.geolocation.getCurrentPosition(
        (position) => {
          store.commit('initMyLocation', {
            lat: position.coords.latitude,
            lng: position.coords.longitude,
          });

          initMapMarkers();
        },
        (error) => {
          console.log(error.message);
        }
      );
    };

    const initMapMarkers = async () => {
      var mapMarkers = [];
      for (const hub of hubs.data) {
        const response = await fetch(
          'https://maps.googleapis.com/maps/api/geocode/json?address=' +
            hub.name +
            '&key=' +
            store.state.key
        );
        const data = await response.json();
        const location = data.results[0].geometry.location;

        mapMarkers.push({
          id: hub.id,
          name: hub.name,
          distance: haversine(location, store.state.myLocation.position),
          position: location,
          active: false,
          icon: store.state.mapMarkerIcon,
          title: hub.name,
        });
      }

      store.commit('initMapMarkers', sortMapMarkers(mapMarkers));
    };

    const sortMapMarkers = (mapMarkers) => {
      mapMarkers = mapMarkers.sort((a, b) => a.distance - b.distance);
      mapMarkers = mapMarkers.map((marker, index) => ({
        ...marker,
        label: String.fromCharCode(65 + index),
      }));
      return mapMarkers;
    };

    const onMapMarkerClick = (key) => {
      store.commit('updateMapMarkers', {
        id: key,
        active: true,
      });
    };

    watchEffect(() => {
      if (mapRef.value?.ready) {
        mapRef.value.map.panTo({
          lat: store.state.myLocation.position.lat,
          lng: store.state.myLocation.position.lng,
        });
      }
    });

    onMounted(() => {
      if (process.env.GOOGLE_MAP_KEY === '') {
        alert('Please config your GOOGLE_MAP_KEY at line 81 in quasar.config.js file!');
      }

      initCurrentPosition();
      initMapMarkers();
    });

    return {
      store,
      onMapMarkerClick,
      mapRef,
    };
  },
});
</script>
