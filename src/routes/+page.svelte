<script>
  import '../app.css';

  import mapboxgl from 'mapbox-gl';
  import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';
  import { onMount } from 'svelte';

  mapboxgl.accessToken =
    'pk.eyJ1IjoiY2FzaWxsYXNlbnJpcXVlIiwiYSI6ImNrdzFxMW8ybmF2enIydXExb29yeDJ6NHkifQ.D6_IaYDb3vhMEIrPHoCoAA';

  let mapContainer;
  onMount(async () => {
    const map = new mapboxgl.Map({
      container: mapContainer,
      center: [-71.09176402733907, 42.35982366492363],
      zoom: 12,
    });

    await new Promise((resolve) => map.on('load', resolve));

    map.addSource('boston-bike-routes', {
      type: 'geojson',
      data: 'https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D',
    });

    map.addLayer({
      id: 'boston-bike-routes', // A name for our layer (up to you)
      type: "line",
      source: 'boston-bike-routes', // The id we specified in `addSource()`
      // paint: {
      //   // paint params, e.g. colors, thickness, etc.
      // },
    });
  });
</script>

<h1>BlueBikes in Boston and Cambridge</h1>
<h2>Visualizing bikeshare traffic</h2>

<div id="map" bind:this={mapContainer}></div>

<style>
  #map {
    flex: 1;
    background-color: red;
  }
</style>
