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
  let trips = [];
  let departures = [];
  let arrivals = [];

  function getCoords(station) {
    let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
    let { x, y } = map.project(point);
    return { cx: x, cy: y };
  }
  
  function minutesSinceMidnight(date) {
	  return date.getHours() * 60 + date.getMinutes();
  }

  async function createMap() {
    map = new mapboxgl.Map({
      container: mapContainer,
      center: [-71.09176402733907, 42.35982366492363],
      zoom: 12,
      style: 'mapbox://styles/casillasenrique/clukyyerk007v01pb6r107k1o'
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

    stations = await d3.csv('https://vis-society.github.io/labs/8/data/bluebikes-stations.csv')
    trips = await d3.csv('https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv').then((trips) => {
      for (let trip of trips) {
        // Convert trip start/end times to dates
        trip.started_at = new Date(trip.started_at)
        trip.ended_at = new Date(trip.ended_at)
      }
      return trips;
    });

    departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
    arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

    stations = stations.map(station => {
	    let id = station.Number;
	    station.arrivals = arrivals.get(id) ?? 0;
	    station.departures = departures.get(id) ?? 0;
      station.totalTraffic = station.departures + station.arrivals
	    return station;
    });


    console.log("Stations:",stations[0]);
    console.log("Traffic:",trips[0]);
  }
  
  
  onMount(() => {
    createMap()
  });
  
  let mapViewChanged = 0;
  
  $: map?.on("move", e => mapViewChanged++);
  $: radiusScale = d3.scaleSqrt()
	.domain([0, d3.max(stations, d => d.totalTraffic)])
	.range(timeFilter === -1? [0, 25] : [3, 50]);
  
  let timeFilter = -1;
  $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
  .toLocaleString("en", {timeStyle: "short"});
  
  let filteredStations = [];

  $: filteredTrips = timeFilter === -1? trips : trips.filter(trip => {
	  let startedMinutes = minutesSinceMidnight(trip.started_at);
	  let endedMinutes = minutesSinceMidnight(trip.ended_at);
	  return Math.abs(startedMinutes - timeFilter) <= 60
	       || Math.abs(endedMinutes - timeFilter) <= 60;
  });

  $: filteredDepartures = timeFilter === -1? departures
	: d3.rollup(filteredTrips, v => v.length, d => d.start_station_id);
  $: filteredArrivals = timeFilter === -1? arrivals : d3.rollup(filteredTrips, v => v.length, d => d.end_station_id);

  $: filteredStations = timeFilter === -1 ? stations : stations.map(station => {
	station = {...station};
	let id = station.Number;
	station.arrivals = filteredArrivals.get(id) ?? 0;
	station.departures = filteredDepartures.get(id) ?? 0;
	station.totalTraffic = station.arrivals + station.departures;
	return station;
});
</script>

<header>
	<h1>🚴🏼‍♀️ Bikewatching</h1>
	<label>
		Filter by time:
		<input type="range" min="-1" max="1439" bind:value={timeFilter}>
		{#if timeFilter === -1 }
		<em>(any time)</em>
		{:else }
		<time>{timeFilterLabel}</time>
		{/if}
	</label>
</header>
  
<div id="map" bind:this={mapContainer}>
  <svg>
    {#key mapViewChanged}
      {#each filteredStations as station}
        <circle {...getCoords(station)} r={radiusScale(station.totalTraffic)} fill="steelblue" >
        	<title>{station.totalTraffic} trips ({station.departures} departures, { station.arrivals} arrivals)</title>
        </circle>
      {/each}
    {/key}
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
        stroke-opacity: 70%;
        pointer-events: auto;
      }
    }

  }

  header {
	display: flex;
	gap: 1em;
	align-items: baseline;
	margin-bottom: 1em;

	label {
		margin-left: auto;

		input {
			width: 30em;
		}

		em, time {
			display: block;
			text-align: right;
		}

		em {
			opacity: .6;
		}
	}
}
</style>
