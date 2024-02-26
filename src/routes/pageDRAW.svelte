<!-- YRS: Deze code load de drawing library en staat toe dat er op de Google Maps een polygon kan worden getekend. Let wel op:
De andere sections van de app doen het niet (geen building insight, geen search bar, geen data layers, etc.)-->

<!--
 Copyright 2023 Google LLC

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

      https://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
 -->

 <script lang="ts">
    /* global google */
  
    import * as GMAPILoader from '@googlemaps/js-api-loader';
    const { Loader } = GMAPILoader;
    import { onMount } from 'svelte';
  
    import SearchBar from './components/SearchBar.svelte';
    import Sections from './sections/Sections.svelte';
  
    const googleMapsApiKey = import.meta.env.VITE_GOOGLE_MAPS_API_KEY;
    const defaultPlace = {
      name: 'Protium, Schweitzerlaan 12, 9728NP Groningen, Nederland',
      address: 'Schweitzerlaan 12, 9728NP Groningen, Nederland',
    };
    let location: google.maps.LatLng | undefined;
    const zoom = 20;
  
    // Initialize app.
    let mapElement: HTMLElement;
    let map: google.maps.Map;
    let geometryLibrary: google.maps.GeometryLibrary;
    let mapsLibrary: google.maps.MapsLibrary;
    let drawingLibrary: google.maps.DrawingLibrary;
    let placesLibrary: google.maps.PlacesLibrary;
    let drawingManager: google.maps.drawing.DrawingManager; // Add this line to declare the drawingManager
  
    onMount(async () => {
      // Load the Google Maps libraries.
      const loader = new Loader({ 
        apiKey: googleMapsApiKey, 
        libraries: ["geometry", "places", "drawing"] // Add "drawing" to the libraries array
      });
      await loader.load().then(() => {
        const geocoder = new google.maps.Geocoder();
        geocoder.geocode({ address: defaultPlace.address }, async (results, status) => {
          if (status === 'OK' && results[0]) {
            location = results[0].geometry.location;
            map = new google.maps.Map(mapElement, {
              center: location,
              zoom: zoom,
              tilt: 0,
              mapTypeId: 'satellite',
              mapTypeControl: false,
              fullscreenControl: false,
              rotateControl: false,
              streetViewControl: false,
              zoomControl: false,
            });
  
            // Initialize the Drawing Manager here
            drawingManager = new google.maps.drawing.DrawingManager({
              drawingMode: google.maps.drawing.OverlayType.POLYGON,
              drawingControl: true,
              drawingControlOptions: {
                position: google.maps.ControlPosition.TOP_CENTER,
                drawingModes: [google.maps.drawing.OverlayType.POLYGON]
              },
              polygonOptions: {
                fillColor: '#ffff00',
                fillOpacity: 0.1,
                strokeWeight: 5,
                strokeColor: '#FF0000',
                clickable: true,
                editable: true,
                zIndex: 1
              }
            });
            drawingManager.setMap(map);
          }
        });
      });
    });
  </script>
  
  <!-- Top bar -->
  <div class="flex flex-row h-full">
    <!-- Main map -->
    <div bind:this={mapElement} class="w-full" />
  
  <!-- Side bar -->
  <aside class="flex-none md:w-96 w-80 p-2 pt-3 overflow-auto">
    <div class="flex flex-col space-y-2 h-full">
  
      <!-- Your Company Logo at the Top -->
      <div class="flex flex-col items-center w-full">
        <img src="/src/routes/Protium Logo Centered.svg" alt="Protium Company Logo" class="w-auto h-20 my-4" />
      </div>
  
      {#if placesLibrary && map}
        <SearchBar bind:location {placesLibrary} {map} initialValue={defaultPlace.name} />
      {/if}
  
      {#if location}
        <Sections {location} {map} {geometryLibrary} {googleMapsApiKey} />
      {/if}
  
        <!-- Your Customer's Company Logo at the Bottom -->
        <div class="flex-grow"></div> <!-- This ensures that the customer logo stays at the bottom -->
        <div class="flex flex-col items-center w-full pb-4">
          <img src="/src/routes/RomAIx - Logo Design.svg" alt="RomAIx Company Logo" class="w-auto h-20 my-4" />
        </div>
  
      </div>
    </aside>
  </div>
  