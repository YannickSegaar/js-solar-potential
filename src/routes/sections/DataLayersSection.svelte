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

  import { onMount } from 'svelte';

  import type { MdDialog } from '@material/web/dialog/dialog';
  import Calendar from '../components/Calendar.svelte';
  import Dropdown from '../components/Dropdown.svelte';
  import Expandable from '../components/Expandable.svelte';
  import { getLayer, type Layer } from '../layer';
  import {
    getDataLayerUrls,
    type BuildingInsightsResponse,
    type DataLayersResponse,
    type LayerId,
    type RequestError,
  } from '../solar';
  import InputBool from '../components/InputBool.svelte';
  import Show from '../components/Show.svelte';
  import SummaryCard from '../components/SummaryCard.svelte';
  import type { MdSlider } from '@material/web/slider/slider';

  export let expandedSection: string;
  export let showPanels = true;

  export let googleMapsApiKey: string;
  export let buildingInsights: BuildingInsightsResponse;
  export let geometryLibrary: google.maps.GeometryLibrary;
  export let map: google.maps.Map;

  const icon = 'layers';
  const title = 'Afbeeldingen';

  const dataLayerOptions: Record<LayerId | 'none', string> = {
    none: 'Geen laag',
    mask: 'Dakmasker',
    dsm: 'Digital Surface Model',
    rgb: 'Luchtfoto',
    annualFlux: 'Jaarlijkse zonneschijn',
    monthlyFlux: 'Maandelijkse zonneschijn',
    hourlyShade: 'Uurlijkse schaduw',
  };

  const monthNames = [
    'Jan',
    'Feb',
    'Mar',
    'Apr',
    'Mei',
    'Jun',
    'Jul',
    'Aug',
    'Sep',
    'Okt',
    'Nov',
    'Dec',
  ];

  let dataLayersResponse: DataLayersResponse | undefined;
  let requestError: RequestError | undefined;
  let apiResponseDialog: MdDialog;
  let layerId: LayerId | 'none' = 'monthlyFlux';
  let layer: Layer | undefined;

  let playAnimation = true;
  let tick = 0;
  let month = 0;
  let day = 14;
  let hour = 0;

  let overlays: google.maps.GroundOverlay[] = [];
  let showRoofOnly = false;
  async function showDataLayer(reset = false) {
    if (reset) {
      dataLayersResponse = undefined;
      requestError = undefined;
      layer = undefined;

      // Default values per layer.
      showRoofOnly = ['annualFlux', 'monthlyFlux', 'hourlyShade'].includes(layerId);
      map.setMapTypeId(layerId == 'rgb' ? 'roadmap' : 'satellite');
      overlays.map((overlay) => overlay.setMap(null));
      month = layerId == 'hourlyShade' ? 3 : 0;
      day = 14;
      hour = 5;
      playAnimation = ['monthlyFlux', 'hourlyShade'].includes(layerId);
    }
    if (layerId == 'none') {
      return;
    }

    if (!layer) {
      const center = buildingInsights.center;
      const ne = buildingInsights.boundingBox.ne;
      const sw = buildingInsights.boundingBox.sw;
      const diameter = geometryLibrary.spherical.computeDistanceBetween(
        new google.maps.LatLng(ne.latitude, ne.longitude),
        new google.maps.LatLng(sw.latitude, sw.longitude),
      );
      const radius = Math.ceil(diameter / 2);
      try {
        dataLayersResponse = await getDataLayerUrls(center, radius, googleMapsApiKey);
      } catch (e) {
        requestError = e as RequestError;
        return;
      }

      try {
        layer = await getLayer(layerId, dataLayersResponse, googleMapsApiKey);
      } catch (e) {
        requestError = e as RequestError;
        return;
      }
    }

    const bounds = layer.bounds;
    console.log('Render layer:', {
      layerId: layer.id,
      showRoofOnly: showRoofOnly,
      month: month,
      day: day,
    });
    overlays.map((overlay) => overlay.setMap(null));
    overlays = layer
      .render(showRoofOnly, month, day)
      .map((canvas) => new google.maps.GroundOverlay(canvas.toDataURL(), bounds));

    if (!['monthlyFlux', 'hourlyShade'].includes(layer.id)) {
      overlays[0].setMap(map);
    }
  }

  $: if (layer?.id == 'monthlyFlux') {
    overlays.map((overlay, i) => overlay.setMap(i == month ? map : null));
  } else if (layer?.id == 'hourlyShade') {
    overlays.map((overlay, i) => overlay.setMap(i == hour ? map : null));
  }

  function onSliderChange(event: Event) {
    const target = event.target as MdSlider;
    if (layer?.id == 'monthlyFlux') {
      if (target.valueStart != month) {
        month = target.valueStart ?? 0;
      } else if (target.valueEnd != month) {
        month = target.valueEnd ?? 0;
      }
      tick = month;
    } else if (layer?.id == 'hourlyShade') {
      if (target.valueStart != hour) {
        hour = target.valueStart ?? 0;
      } else if (target.valueEnd != hour) {
        hour = target.valueEnd ?? 0;
      }
      tick = hour;
    }
  }

  $: if (layer?.id == 'monthlyFlux') {
    if (playAnimation) {
      month = tick % 12;
    } else {
      tick = month;
    }
  } else if (layer?.id == 'hourlyShade') {
    if (playAnimation) {
      hour = tick % 24;
    } else {
      tick = hour;
    }
  }

  onMount(() => {
    showDataLayer(true);

    setInterval(() => {
      tick++;
    }, 1000);
  });
</script>

{#if requestError}
  <div class="error-container on-error-container-text">
    <Expandable section={title} icon="error" {title} subtitle={requestError.error.status}>
      <div class="grid place-items-center py-2 space-y-4">
        <div class="grid place-items-center">
          <p class="body-medium">
            Error on <code>dataLayers</code>
            {layerId} request
          </p>
          <p class="title-large">ERROR {requestError.error.code}</p>
          <p class="body-medium"><code>{requestError.error.status}</code></p>
          <p class="label-medium">{requestError.error.message}</p>
        </div>
        <md-filled-button role={undefined} on:click={() => showDataLayer(true)}>
          Retry
          <md-icon slot="icon">refresh</md-icon>
        </md-filled-button>
      </div>
    </Expandable>
  </div>
{:else}
  <Expandable bind:section={expandedSection} {icon} {title} subtitle={dataLayerOptions[layerId]}>
    <div class="flex flex-col space-y-2 px-2">
      <span class="outline-text label-medium">
        <b>{title}</b> geeft ruwe en bewerkte beelden en gedetailleerde details over een gebied rondom
        een locatie.
      </span>

      <Dropdown
        bind:value={layerId}
        options={dataLayerOptions}
        onChange={async () => showDataLayer(true)}
      />

      {#if layerId == 'none'}
        <div />
      {:else if !layer}
        <md-linear-progress four-color indeterminate />
      {:else}
        {#if layer.id == 'hourlyShade'}
          <Calendar bind:month bind:day onChange={async () => showDataLayer()} />
        {/if}

        <InputBool bind:value={showPanels} label="Zonnepanelen" />
        <InputBool bind:value={showRoofOnly} label="Enkel dak" onChange={() => showDataLayer()} />

        {#if ['monthlyFlux', 'hourlyShade'].includes(layerId)}
          <InputBool bind:value={playAnimation} label="Speel animatie af" />
        {/if}
      {/if}
      <div class="flex flex-row">
        <div class="grow" />
        <md-filled-tonal-button role={undefined} on:click={() => apiResponseDialog.show()}>
          API response
        </md-filled-tonal-button>
      </div>

      <md-dialog bind:this={apiResponseDialog}>
        <div slot="headline">
          <div class="flex items-center primary-text">
            <md-icon>{icon}</md-icon>
            <b>&nbsp;{title}</b>
          </div>
        </div>
        <div slot="content">
          <Show value={dataLayersResponse} label="dataLayersResponse" />
        </div>
        <div slot="actions">
          <md-text-button role={undefined} on:click={() => apiResponseDialog.close()}>
            Close
          </md-text-button>
        </div>
      </md-dialog>
    </div>
  </Expandable>
{/if}

<div class="absolute top-0 left-0 w-72">
  {#if expandedSection == title && layer}
    <div class="m-2">
      <SummaryCard {icon} {title} rows={[{ name: dataLayerOptions[layerId], value: '' }]}>
        <div class="flex flex-col space-y-4">
          <p class="outline-text">
            {#if layerId == 'mask'}
              De afbeelding van het gebouwmasker: één bit per pixel die zegt of die pixel wordt beschouwd als
              onderdeel van een dak of niet.
            {:else if layerId == 'dsm'}
              Een afbeelding van het DSM (Digital Surface Model) van de regio. De waarden zijn in meters boven
              EGM96 geoïde (d.w.z. zeeniveau). Ongeldige locaties (waar we geen gegevens hebben) worden opgeslagen
              als -9999.
            {:else if layerId == 'rgb'}
              Een afbeelding van RGB-gegevens (luchtfoto) van de regio.
            {:else if layerId == 'annualFlux'}
              De jaarlijkse fluxkaart (jaarlijks zonlicht op daken) van de regio. Waarden zijn kWh/kW/jaar.
              Dit is een ongemaskeerde flux: de flux wordt berekend voor elke locatie, niet alleen voor daken van gebouwen.
              daken. Ongeldige locaties worden opgeslagen als -9999: locaties buiten ons dekkingsgebied
              zullen ongeldig zijn, en een paar locaties binnen het dekkingsgebied, waar we geen flux konden berekenen, zullen ook ongeldig zijn.
              flux konden berekenen, zullen ook ongeldig zijn.
            {:else if layerId == 'monthlyFlux'}
              De maandelijkse fluxkaart (zonlicht op daken, uitgesplitst per maand) van de regio. De waarden
              zijn kWh/kW/jaar. Het GeoTIFF-beeldbestand waarnaar deze URL verwijst, bevat twaalf
              banden, overeenkomend met januari...december, in volgorde.
            {:else if layerId == 'hourlyShade'}
              Twaalf URL's voor schaduw per uur, overeenkomend met januari...december, in volgorde. Elk
              GeoTIFF-beeldbestand bevat 24 banden, overeenkomend met de 24 uur van de dag.
              Elke pixel is een geheel getal van 32 bits, dat overeenkomt met de (maximaal) 31 dagen van die maand; een
              1 bit betekent dat de corresponderende locatie de zon kan zien op die dag, dat uur, die maand.
              dat uur, van die maand. Ongeldige locaties worden opgeslagen als -9999 (omdat dit
              negatief is, is bit 31 gezet, en geen enkele geldige waarde kan bit 31 gezet hebben omdat dat zou overeenkomen met de 32e dag van de maand.
              overeenkomen met de 32e dag van de maand).
            {/if}
          </p>

          {#if layer.palette}
            <div>
              <div
                class="h-2 outline rounded-sm"
                style={`background: linear-gradient(to right, ${layer.palette.colors.map(
                  (hex) => '#' + hex,
                )})`}
              />
              <div class="flex justify-between pt-1 label-small">
                <span>{layer.palette.min}</span>
                <span>{layer.palette.max}</span>
              </div>
            </div>
          {/if}
        </div>
      </SummaryCard>
    </div>
  {/if}
</div>

<div class="absolute bottom-6 left-0 w-full">
  <div class="md:mr-96 mr-80 grid place-items-center">
    {#if layer}
      <div
        class="flex items-center surface on-surface-text pr-4 text-center label-large rounded-full shadow-md"
      >
        {#if layer.id == 'monthlyFlux'}
          <md-slider
            range
            min={0}
            max={11}
            value-start={month}
            value-end={month}
            on:input={onSliderChange}
          />
          <span class="w-8">{monthNames[month]}</span>
        {:else if layer.id == 'hourlyShade'}
          <md-slider
            range
            min={0}
            max={23}
            value-start={hour}
            value-end={hour}
            on:input={onSliderChange}
          />
          <span class="w-24 whitespace-nowrap">
            {monthNames[month]}
            {day},
            {#if hour == 0}
              12am
            {:else if hour < 10}
              {hour}am
            {:else if hour < 12}
              {hour}am
            {:else if hour == 12}
              12pm
            {:else if hour < 22}
              {hour - 12}pm
            {:else}
              {hour - 12}pm
            {/if}
          </span>
        {/if}
      </div>
    {/if}
  </div>
</div>
