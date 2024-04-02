<script>
  import '../app.css';

  import mapboxgl from 'mapbox-gl';
  import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  mapboxgl.accessToken =
    'pk.eyJ1IjoiY2FzaWxsYXNlbnJpcXVlIiwiYSI6ImNrdzFxMW8ybmF2enIydXExb29yeDJ6NHkifQ.D6_IaYDb3vhMEIrPHoCoAA';

  const bikeLanesPaint = {
    'line-color': '#7BEC66',
    'line-width': 3,
    'line-opacity': 0.8,
  };

  let mapContainer;
  let map;
  let stations = [];
  let traffic = [];

  function getCenters(stations) {
    let centers = new Map();

    if (map) {
      for (let station of stations) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let { x, y } = map.project(point);
        centers.set(station, { cx: x, cy: y });
      }
    }

    return centers;
  }

  async function createMap() {
    map = new mapboxgl.Map({
      container: mapContainer,
      center: [-71.09176402733907, 42.35982366492363],
      zoom: 12,
    });

    await new Promise((resolve) => map.on('load', resolve));

    map.addSource('boston-bike-routes', {
      type: 'geojson',
      data: 'https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D',
    });

    map.addSource('cambridge-bike-routes', {
      type: 'geojson',
      data: 'https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson',
    });

    map.addLayer({
      id: 'boston-bike-routes',
      type: 'line',
      source: 'boston-bike-routes',
      paint: bikeLanesPaint,
    });

    map.addLayer({
      id: 'cambridge-bike-routes',
      type: 'line',
      source: 'cambridge-bike-routes',
      paint: bikeLanesPaint,
    });

    [stations, traffic] = await Promise.all([
      d3.csv('https://vis-society.github.io/labs/8/data/bluebikes-stations.csv'),
      d3.csv('https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv'),
    ]);
    console.log("Stations:",stations[0]);
    console.log("Traffic:",traffic[0]);

    const arrivals = d3.rollup(
      traffic,
      (v) => v.length,
      (d) => d.end_station_id
    );
    const departures = d3.rollup(
      traffic,
      (v) => v.length,
      (d) => d.start_station_id
    );


    stations = stations.map((station) => {
      const numArrivals = arrivals.get(station.Number), numDepartures = departures.get(station.Number);
      const totalTraffic = numDepartures + numArrivals;
      return {...station, numArrivals, numDepartures, totalTraffic}      
    })
    console.log(stations);
  }

  onMount(() => {
    createMap()
  });

  $: stationCoords = getCenters(stations);
  $: map?.on('move', (evt) => (stationCoords = getCenters(stations)));
  $: radiusScale = d3.scaleSqrt()
	.domain(d3.extent(stations, d => d.totalTraffic))
	.range([1, 26]);

</script>

<h1>BlueBikes in Boston and Cambridge</h1>
<h2>Visualizing bikeshare traffic</h2>

<div id="map" bind:this={mapContainer}>
  <svg>
    {#each stations as station}
      <circle {...stationCoords.get(station)} r={radiusScale(station.totalTraffic)} fill="steelblue" />
    {/each}
  </svg>
</div>

<style>
  #map {
    flex: 1;
    background-color: #7bec66;

    svg {
      position: absolute;
      z-index: 1;
      width: 100%;
      height: 100%;
      pointer-events: none;

      circle {
        fill-opacity: 40%;
        stroke-opacity: 70%
      }
    }
  }
</style>
