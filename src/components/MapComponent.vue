<template>
  <section>
    <h1 id="loading" v-if="!loaded">Loading...</h1>
    <section
      ref="mapContainer"
      id="map-container"
      :style="{
        opacity: loaded ? 1 : 0.2,
      }"
    ></section>
  </section>
</template>

<script>
import * as d3 from 'd3'
import * as turf from '@turf/turf'
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
    }
  },
  computed: {},
  mounted: function () {
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

    this.map.on('load', () => {
      // use d3 to load the geojson route data
      d3.json('tracks01.geojson').then((data) => {
        this.coordinates = data.features[0].geometry.coordinates

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
          exaggeration: 2.2,
        })
      })
    })

    this.map.addControl(new maplibregl.NavigationControl())

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
  methods: {},
}
</script>

<style>
#map-container {
  position: fixed;
  top: 0;
  bottom: 0;
  width: 100%;
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
