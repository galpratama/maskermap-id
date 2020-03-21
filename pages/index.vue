<template>
  <v-row no-gutters>
    <v-col>
      <v-card style="height: 83vh; width: 100%">
        <l-map
          v-if="showMap"
          :zoom="zoom"
          :center="center"
          :options="mapOptions"
          style="height: 100%"
          @update:center="centerUpdate"
          @update:zoom="zoomUpdate"
        >
          <l-control-zoom position="topright"></l-control-zoom>
          <l-control position="topleft">
            <v-container grid-list-xs>
              <v-row>
                <v-col>
                  <v-text-field
                    label="Cari lokasi masker..."
                    solo
                    clearable
                    :loading="searchLoading"
                    v-model="search"
                    @input="debounceSearchPlaces()"
                    @keydown.enter="getPlaces()"
                  ></v-text-field>
                  <v-card
                    class="mx-auto"
                    max-width="400"
                    tile
                    v-if="searchPlaces"
                  >
                    <v-list three-line>
                      <v-list-item-group>
                        <v-list-item
                          v-for="place in searchPlaces"
                          :key="place.id"
                          @click="
                            $refs['marker-' + place.id][0].mapObject.openPopup()
                          "
                        >
                          <v-list-item-content>
                            <v-list-item-title class="font-weight-black">
                              {{ place.name }}
                            </v-list-item-title>
                            <v-list-item-subtitle class="text--primary">
                              {{ place.description }}
                            </v-list-item-subtitle>
                            <v-list-item-subtitle v-if="place.story">
                              {{
                                place.story[place.story.length - 1].availability
                              }}
                              &middot;
                              {{
                                place.story[place.story.length - 1].num
                                  ? place.story[place.story.length - 1].num
                                  : 0
                              }}
                              pcs &middot; Rp{{
                                toRupiah(
                                  place.story[place.story.length - 1].price
                                )
                              }}
                            </v-list-item-subtitle>
                          </v-list-item-content>
                          <v-list-item-action>
                            <v-list-item-action-text
                              v-text="
                                $moment
                                  .unix(
                                    place.story[place.story.length - 1].created
                                  )
                                  .fromNow(true)
                              "
                            ></v-list-item-action-text>
                          </v-list-item-action>
                        </v-list-item>
                      </v-list-item-group>
                    </v-list>
                  </v-card>
                </v-col>
              </v-row>
            </v-container>
          </l-control>
          <l-tile-layer :url="url" :attribution="attribution" />
          <template v-for="place in places">
            <l-marker
              :lat-lng="getLatLng(place.lat, place.long)"
              :key="place.id"
              :ref="'marker-' + place.id"
            >
              <l-popup>
                <div @click="innerClick">
                  <h2>
                    {{ place.name }}
                  </h2>
                  <p>
                    {{ place.description }}
                  </p>
                  <v-simple-table>
                    <template v-slot:default>
                      <thead>
                        <tr>
                          <th class="text-left">Ada?</th>
                          <th class="text-left">Harga</th>
                          <th class="text-left">Banyak</th>
                          <th class="text-left">Diupdate</th>
                        </tr>
                      </thead>
                      <tbody>
                        <tr v-for="story in place.story" :key="story.id">
                          <td>{{ story.availability }}</td>
                          <td>{{ toRupiah(story.price) }}</td>
                          <td>{{ story.num ? story.num : 0 }} pcs</td>
                          <td>{{ $moment.unix(story.created).fromNow() }}</td>
                        </tr>
                      </tbody>
                    </template>
                  </v-simple-table>
                </div>
              </l-popup>
            </l-marker>
          </template>
        </l-map>
      </v-card>
    </v-col>
  </v-row>
</template>

<script>
import { latLng } from 'leaflet'
import _ from 'lodash'

export default {
  data() {
    return {
      places: [],
      searchPlaces: null,
      searchLoading: false,
      search: '',
      isTyping: false,
      zoom: 15,
      center: latLng(-6.8685959, 107.5804061),
      url: 'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
      attribution:
        '&copy; <a href="http://osm.org/copyright">OpenStreetMap</a>',
      withPopup: latLng(-6.8685959, 107.5804061),
      currentZoom: 11.5,
      currentCenter: latLng(-6.8685959, 107.5804061),
      mapOptions: {
        zoomSnap: 0.5,
        zoomControl: false
      },
      showMap: true
    }
  },
  async asyncData({ $axios }) {
    try {
      const places = await $axios.$get(
        'https://covid19-api.yggdrasil.id/maskmap/places'
      )
      return { places }
    } catch (error) {
      return { places: null }
    }
  },
  created() {
    this.debounceSearchPlaces = _.debounce(this.getPlaces, 200)
  },
  methods: {
    async getPlaces() {
      this.searchLoading = true
      try {
        const places = await this.$axios.$post(
          'https://covid19-api.yggdrasil.id/maskmap/places',
          {
            name: this.search
          }
        )
        this.searchPlaces = places
      } catch (error) {
        this.searchPlaces = ''
      }
      this.searchLoading = false
    },
    getLatLng(lat, lng) {
      return latLng(lat, lng)
    },
    toRupiah(number) {
      return number.toString().replace(/(\d)(?=(\d{3})+(?!\d))/g, '$1,')
    },
    zoomUpdate(zoom) {
      this.currentZoom = zoom
    },
    centerUpdate(center) {
      this.currentCenter = center
    },
    innerClick() {
      // alert('Click!')
    }
  },
  computed: {
    filteredPlaces() {
      let searchValue = this.search.toLowerCase()
      let filter = place =>
        place.name.toLowerCase().includes(searchValue) ||
        place.description.toLowerCase().includes(searchValue)

      return this.places.filter(filter)
    }
  }
}
</script>
