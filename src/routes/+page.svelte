<!-- YRS: Deze code load de drawing library en staat toe dat er op de Google Maps een polygon kan worden getekend. Ook de andere sections zijn zichtbaar en werken! -->


<!--
... (License and imports omitted for brevity)
-->


<script lang="ts">
  /* global google */
  import * as GMAPILoader from '@googlemaps/js-api-loader';
  const { Loader } = GMAPILoader;
  import { onMount, afterUpdate } from 'svelte';
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
  let placesLibrary: google.maps.PlacesLibrary;
  let drawingManager: google.maps.drawing.DrawingManager;
  let selectedPolygon: google.maps.Polygon | null = null;
  let polygons: google.maps.Polygon[] = [];
  let infoWindows: google.maps.InfoWindow[] = [];
  let roofsizeDrawing = 0;
  onMount(async () => {
    const loader = new Loader({
      apiKey: googleMapsApiKey,
      libraries: ["geometry", "places", "drawing"]
    });
    const libraries = {
      geometry: loader.importLibrary('geometry'),
      maps: loader.importLibrary('maps'),
      places: loader.importLibrary('places'),
    };
    geometryLibrary = await libraries.geometry;
    mapsLibrary = await libraries.maps;
    placesLibrary = await libraries.places;
    const geocoder = new google.maps.Geocoder();
    const geocoderResponse = await geocoder.geocode({
      address: defaultPlace.address,
    });
    const geocoderResult = geocoderResponse.results[0];
    location = geocoderResult.geometry.location;
    map = new mapsLibrary.Map(mapElement, {
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
    drawingManager = new google.maps.drawing.DrawingManager({
      drawingMode: google.maps.drawing.OverlayType.POLYGON,
      drawingControl: true,
      drawingControlOptions: {
        position: google.maps.ControlPosition.TOP_CENTER,
        drawingModes: [google.maps.drawing.OverlayType.POLYGON],
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
 
 
    google.maps.event.addListener(drawingManager, 'polygoncomplete', function(polygon) {
      polygons.push(polygon);
      // Automatically update the coordinates and area when the polygon is edited
      // Listen for the 'set_at' and 'insert_at' events for polygon paths
      google.maps.event.addListener(polygon.getPath(), 'set_at', updatePolygonDetails);
      google.maps.event.addListener(polygon.getPath(), 'insert_at', updatePolygonDetails);
 
 
      const InfoBoxLoc = polygonCenter(polygon);
      const infowindow = new google.maps.InfoWindow({
        content: generateInfoWindowContent(polygon),
        position: InfoBoxLoc,
      });
      infowindow.open(map);
      infoWindows.push(infowindow); // Track the infowindow
 
 
      google.maps.event.addListener(infowindow, 'domready', () => {
        attachInfoWindowEventListeners(infowindow, polygon);
      });
 
 
      google.maps.event.addListener(polygon, 'click', () => {
        infowindow.open(map);
      });
 
 
      drawingManager.setDrawingMode(null); // Exit drawing mode
      updateRoofsizeDrawing(); // Update the roofsizeDrawing when a new polygon is completed
    });
  });
 
 
  afterUpdate(() => {
    // Update the total area whenever the component updates
    updateRoofsizeDrawing();
  });
 
 
  function polygonCenter(poly) {
    let lowx, highx, lowy, highy, lats = [], lngs = [], vertices = poly.getPath();
    for (let i = 0; i < vertices.getLength(); i++) {
      lngs.push(vertices.getAt(i).lng());
      lats.push(vertices.getAt(i).lat());
    }
    lats.sort();
    lngs.sort();
    lowx = lats[0];
    highx = lats[vertices.getLength() - 1];
    lowy = lngs[0];
    highy = lngs[vertices.getLength() - 1];
    const center_x = lowx + (highx - lowx) / 2;
    const center_y = lowy + (highy - lowy) / 2;
    return new google.maps.LatLng(center_x, center_y);
  }
 
 
  function updatePolygonDetails() {
    // This function updates the roof size drawing and refreshes info windows content
    updateRoofsizeDrawing();
    infoWindows.forEach((infowindow, index) => {
      const polygon = polygons[index];
      infowindow.setContent(generateInfoWindowContent(polygon));
    });
  }
 
 
  function generateInfoWindowContent(polygon) {
    const area = google.maps.geometry.spherical.computeArea(polygon.getPath()).toFixed(2);
    const coordinates = polygon.getPath().getArray().map(coord => ({ lat: coord.lat(), lng: coord.lng() }));
    return `
      <table>
        <tr>
          <td style="text-align: center;"><strong>Naam:</strong></td>
          <td><input type='text' id='Naam'/> </td>
        </tr>
        <tr>
          <td><input type='button' value='save' data-action='save' style="font-weight: bold; text-decoration: underline;" /></td>
          <td><input type='button' value='delete' data-action='delete' style="font-weight: bold; text-decoration: underline;" /></td>
        </tr>
        <tr>
          <td>Coordinates:</td>
          <td>${JSON.stringify(coordinates)}</td>
        </tr>
        <tr>
          <td>Area:</td>
          <td>${area} m²</td>
        </tr>
      </table>
    `;
  }
 
 
  function attachInfoWindowEventListeners(infowindow, polygon) {
    const saveButton = document.querySelector('td > input[type="button"][data-action="save"]');
    const deleteButton = document.querySelector('td > input[type="button"][data-action="delete"]');
    if (saveButton && deleteButton) {
      saveButton.addEventListener('click', () => {
        const fieldname = escape(document.getElementById("Naam").value);
        console.log(fieldname);
        // No need to manually update coordinates and area here anymore, as they are updated automatically
      });
      deleteButton.addEventListener('click', () => {
        infowindow.close();
        polygon.setMap(null);
        polygons = polygons.filter(p => p !== polygon);
        infoWindows = infoWindows.filter(iw => iw !== infowindow);
        updatePolygonDetails(); // Update the roofsizeDrawing after deletion
      });
    }
  }
  
  function updateRoofsizeDrawing() {
    roofsizeDrawing = polygons.reduce((total, p) => total + google.maps.geometry.spherical.computeArea(p.getPath()), 0);
    console.log('roofsizeDrawing:', roofsizeDrawing);
  }
 </script>
 <!-- Top bar -->
 <div class="flex flex-row h-full">
  <!-- Main map -->
  <div bind:this={mapElement} class="w-full"></div>
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
        <!-- <img src="/src/routes/RomAIx - Logo Design.svg" alt="RomAIx Company Logo" class="w-auto h-20 my-4" /> -->
      </div>
    </div>
  </aside>
</div>
