<template>
  <div>
    <section class="pa5 fixed top-0 w-100" v-if="coordinates.length < 1">
      <h2>Drop a .gpx file to see it on the map</h2>
      <form
        id="drop-form"
        ref="dropForm"
        @drop="fileDrop($event)"
        @dragover="fileDragover"
        @dragleave="fileDragleave"
        class="w-100 vh-50 bg-light-gray pa4 flex flex-column justify-center items-center"
      >
        <span class="drop-files">Drop files here</span>
      </form>
    </section>
    <section v-show="coordinates.length > 1">
      <section class="show-map">
        <h1 id="loading" v-if="!loaded">Loading...</h1>
        <section
          ref="mapContainer"
          id="map-container"
          :style="{
            opacity: loaded ? 1 : 0.2,
          }"
        ></section>
      </section>
    </section>
  </div>
</template>

<script>
import * as d3 from 'd3'
import * as turf from '@turf/turf'
// import * as gpxparser from 'gpxparser'
import * as togeojson from 'togeojson'
import maplibregl from 'maplibre-gl'

export default {
  name: 'MapComponent',
  props: {
    trackGeoJson: {
      type: Object,
      required: false,
    },
  },
  data: function () {
    return {
      map: null,
      coordinates: [],
      scrollScale: null,
      loaded: false,
      files: [],
    }
  },
  computed: {},
  mounted: function () {
    // this.initMap()

    // scroll the page to the top if it isn't already with vanilla JS
    window.scrollTo(0, 0)

    this.bindFileDropEvents()
    // add listener for window scroll
    window.addEventListener('scroll', () => {
      const scrollTop = window.pageYOffset || document.documentElement.scrollTop
      const scrollBottom = scrollTop + window.innerHeight

      if (!this.scrollScale) return
      const scrollToCoordinateIndex = Math.round(this.scrollScale(scrollBottom))
      if (!this.coordinates[scrollToCoordinateIndex]) return
      if (!scrollToCoordinateIndex) return

      this.map.setCenter([
        this.coordinates[scrollToCoordinateIndex][0],
        this.coordinates[scrollToCoordinateIndex][1],
      ])
    })
  },
  methods: {
    initMap() {
      // check if we are on mobile or not
      const isMobile = window.innerWidth < 768
      this.map = new maplibregl.Map({
        container: 'map-container',
        interactive: !isMobile,
        pitch: 80,
        // Use a free map style from Stamen
        style: {
          version: 8,
          sources: {
            'raster-tiles': {
              type: 'raster',
              tiles: [
                'https://stamen-tiles.a.ssl.fastly.net/terrain/{z}/{x}/{y}.jpg',
              ],
              tileSize: 256,
              attribution:
                'Map tiles by <a target="_top" rel="noopener" href="http://stamen.com">Stamen Design</a>, under <a target="_top" rel="noopener" href="http://creativecommons.org/licenses/by/3.0">CC BY 3.0</a>. Data by <a target="_top" rel="noopener" href="http://openstreetmap.org">OpenStreetMap</a>, under <a target="_top" rel="noopener" href="http://creativecommons.org/licenses/by-sa/3.0">CC BY SA</a>',
            },
          },
          layers: [
            {
              id: 'simple-tiles',
              type: 'raster',
              source: 'raster-tiles',
              minzoom: 0,
              maxzoom: 22,
            },
          ],
        },
        // center: [-74.006, 40.7128],
        // center on Newburgh, NY
        center: [-74.021, 41.5],
        zoom: 13,
      })

      // disable map zoom when using scroll
      this.map.scrollZoom.disable()

      // allow further pitch over default of 60
      this.map.setMaxPitch(80)

      this.map.addControl(new maplibregl.NavigationControl())

      return this.map
    },
    bindFileDropEvents() {
      /*
					Listen to all of the drag events and bind an event listener to each
					for the fileform.
				*/
      // eslint-disable-next-line no-extra-semi
      ;[
        'drag',
        'dragstart',
        'dragend',
        'dragover',
        'dragenter',
        'dragleave',
        'drop',
      ].forEach(
        function (evt) {
          /*
						For each event add an event listener that prevents the default action
						(opening the file in the browser) and stop the propagation of the event (so
						no other elements open the file in the browser)
					*/

          const dropformEl = this.$refs.dropForm
          dropformEl.addEventListener(
            evt,
            function (e) {
              e.preventDefault()
              e.stopPropagation()
            }.bind(this),
            false
          )
        }.bind(this)
      )
    },
    fileDrop(event) {
      for (let i = 0; i < event.dataTransfer.files.length; i++) {
        this.files.push(event.dataTransfer.files[i])
      }

      // read the first file blog in the array
      const fileReader = new FileReader()
      fileReader.addEventListener('load', (event) => {
        // console.log('fileReader.result', event.target.result)
        const gpx = new DOMParser().parseFromString(
          event.target.result,
          'text/xml'
        )
        this.coordinates = togeojson.gpx(gpx).features[0].geometry.coordinates
        this.initMap().on('load', () => {
          this.addCoordinatesToMap()
        })
      })

      fileReader.readAsText(this.files[0])
      // this.coordinates = this.files[0]
    },
    addCoordinatesToMap() {
      const scrollMax = this.$parent.$el.offsetHeight
      this.scrollScale = d3
        .scaleLinear()
        .range([0, this.coordinates.length - 1])
        .domain([0, scrollMax])
      this.loaded = true
      this.map.addSource('route', {
        type: 'geojson',
        lineMetrics: true,
        data: {
          type: 'Feature',
          properties: {},
          geometry: {
            type: 'LineString',
            coordinates: this.coordinates,
          },
        },
      })
      this.map.addLayer({
        id: 'route',
        type: 'line',
        source: 'route',
        layout: {
          'line-join': 'round',
          'line-cap': 'round',
        },
        paint: {
          'line-color': 'red',
          'line-width': 10,
          // 'line-gradient' must be specified using an expression
          // with the special 'line-progress' property
          'line-gradient': [
            'interpolate',
            ['linear'],
            ['line-progress'],
            0,
            'blue',
            0.1,
            'royalblue',
            0.3,
            'cyan',
            0.5,
            'lime',
            0.7,
            'yellow',
            1,
            'red',
          ],
        },
      })
      // const bounds = coordinates.reduce(function (bounds, coord) {
      //   return bounds.extend(coord)
      // }, new maplibregl.LngLatBounds(coordinates[0], coordinates[0]))
      const bounds = new maplibregl.LngLatBounds(
        this.coordinates[0],
        this.coordinates[0]
      )
      // this.map.fitBounds(bounds, {
      //   padding: -200,
      // })
      this.map.setCenter([this.coordinates[0][0], this.coordinates[0][1]])
      this.map.addSource('terrain', {
        type: 'raster-dem',
        url: 'https://api.maptiler.com/tiles/terrain-rgb/tiles.json?key=dW1bjSk5npya8SZXgGuw',
      })
      this.map.setTerrain({
        source: 'terrain',
        // exaggeration: 2.2,
        exaggeration: 3,
      })
    },
  },
}
</script>

<style>
#map-container {
  position: fixed;
  top: 0;
  bottom: 0;
  width: 100%;
  background: linear-gradient(
    0deg,
    rgba(36, 0, 0, 1) 56%,
    rgb(249 209 89) 62%,
    rgb(81 151 173) 80%,
    rgb(2 92 179) 95%
  );
}

#loading {
  z-index: 250;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  font-size: 192px;
  font-weight: bold;
  color: black;
}
</style>
