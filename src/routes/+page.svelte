<script lang="ts">
  import Card from "$lib/components/ui/card/card.svelte";
  import * as Tabs from "$lib/components/ui/tabs/index.js";
  import { Checkbox } from "$lib/components/ui/checkbox/index.js";
  import LightSwitch from "$lib/components/ui/light-switch/light-switch.svelte";

  import maplibregl from "maplibre-gl";
  import "maplibre-gl/dist/maplibre-gl.css";

  import LanguageToggle from "$lib/LanguageToggle.svelte";
  import { ModeWatcher } from "mode-watcher";
  import TaalOverzicht from "./TaalOverzicht.svelte";

  let mapContainer: HTMLDivElement | undefined = $state();
  let map: maplibregl.Map | undefined;
  let markers: maplibregl.Marker[] = [];
  let schoolLabelMarker: maplibregl.Marker | undefined;

  import {
    loadObservations,
    loadSchoolData,
    type Observation,
  } from "$lib/data/loadObservations";

  type Stadsdeel = {
    id: number;
    naam: string;
    centroid: [number, number];
  };

  type Locatie = {
    naam: string;
    coordinaten: [number, number];
    type: "school" | "bibliotheek" | "cultureel";
    stadsdeel: string;
  };

  let locale = $state("nl");

  let stadsdelen: Stadsdeel[] = $state([]);
  let locaties: Locatie[] = $state([]);
  let observaties = $state<Observation[]>([]);

  let hoveredStadsdeelId = $state<number | null>(null);
  let selectedStadsdeelId = $state<number | null>(null);

  let hoveredLocatieId = $state<number | null>(null);
  let selectedLocatieId = $state<number | null>(null);

  let showLocaties = $state(true);
  let showLabels = $state(true);

  let locationMarkers = $state<maplibregl.Marker[]>([]);
  let stadsdeelMarkers = $state<maplibregl.Marker[]>([]);

  function clearMarkers() {
    locationMarkers.forEach((m) => m.remove());
    locationMarkers = [];
  }

  function addLabelMarkers(geojson: GeoJSON.FeatureCollection) {
    stadsdeelMarkers.forEach((m) => m.remove());
    stadsdeelMarkers = [];
    geojson.features.forEach((feature) => {
      const name = feature.properties?.Stadsdeel;
      const centroid = feature.properties?.Centroid;
      if (!name || !centroid) return;

      let [lng, lat] = centroid as [number, number];

      const el = document.createElement("div");
      el.className = "stadsdeel-label";
      el.textContent = name;

      const marker = new maplibregl.Marker({ element: el, anchor: "center" })
        .setLngLat([lng, lat])
        .addTo(map!);
      stadsdeelMarkers.push(marker);
    });
  }

  function addLocationMarkers() {
    locationMarkers.forEach((m) => m.remove());
    locationMarkers = [];

    locaties.forEach((l) => {
      const el = document.createElement("div");
      el.className = "school-label-permanent";
      el.textContent = l.naam;

      if (l.coordinaten.some((coord) => isNaN(coord))) {
        console.warn(`Ongeldige coördinaten voor ${l.naam}:`, l.coordinaten);
        return;
      }
      const marker = new maplibregl.Marker({ element: el, anchor: "bottom" })
        .setLngLat([l.coordinaten[1], l.coordinaten[0]])
        .addTo(map!);

      el.style.display = showLabels && showLocaties ? "block" : "none";
      locationMarkers.push(marker);
    });
  }

  $effect(() => {
    if (!map) return;
    if (map.getLayer("locatie-circles")) {
      map.setLayoutProperty(
        "locatie-circles",
        "visibility",
        showLocaties ? "visible" : "none",
      );
    }
  });

  $effect(() => {
    locationMarkers.forEach((marker) => {
      const el = marker.getElement();
      el.style.display = showLocaties && showLabels ? "block" : "none";
    });
  });

  function showSchoolLabel(name: string, lngLat: maplibregl.LngLat) {
    hideSchoolLabel();
    const el = document.createElement("div");
    el.className = "school-label";
    el.textContent = name;
    schoolLabelMarker = new maplibregl.Marker({ element: el, anchor: "bottom" })
      .setLngLat(lngLat)
      .addTo(map!);
  }

  function hideSchoolLabel() {
    schoolLabelMarker?.remove();
    schoolLabelMarker = undefined;
  }

  function updateHighlightSource() {
    const src = map?.getSource("stadsdelen-highlight") as
      | maplibregl.GeoJSONSource
      | undefined;
    if (!src) return;

    const activeId = selectedStadsdeelId ?? hoveredStadsdeelId;

    if (activeId === undefined) {
      src.setData({ type: "FeatureCollection", features: [] });
      return;
    }

    const feature = cachedGeojson?.features.find((f) => f.id === activeId);
    if (!feature) {
      src.setData({ type: "FeatureCollection", features: [] });
      return;
    }

    src.setData({
      type: "FeatureCollection",
      features: [
        {
          ...feature,
          properties: {
            ...feature.properties,
            _selected: selectedStadsdeelId === activeId,
          },
        },
      ],
    });
  }

  let cachedGeojson: GeoJSON.FeatureCollection | undefined;

  $effect(() => {
    if (!mapContainer) return;

    map = new maplibregl.Map({
      container: mapContainer,
      style: "https://basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json",
      center: [4.9041, 52.3676],
      zoom: 11,
    });

    map.on("load", async () => {
      const [geojsonRes, locatiesRes] = await Promise.all([
        fetch("./stadsdelen.json"),
        fetch("./locaties.json"),
      ]);
      const geojson: GeoJSON.FeatureCollection = await geojsonRes.json();
      const scholen: {
        Coordinaten: string;
        Scholen: string;
        Buurt: string;
      }[] = await locatiesRes.json();
      cachedGeojson = geojson;

      stadsdelen = geojson.features.map((f) => ({
        id: f.id as number,
        naam: f.properties?.Stadsdeel,
        centroid: f.properties?.Centroid as [number, number],
      }));

      locaties = scholen.map((l, i) => ({
        naam: l.Scholen,
        coordinaten: l.Coordinaten.split(",").map((s) =>
          parseFloat(s.trim()),
        ) as [number, number],
        type: "school",
        stadsdeel: l.Buurt,
        id: i,
      }));

      const locatiesGeojson: GeoJSON.FeatureCollection = {
        type: "FeatureCollection",
        features: locaties.map((l) => ({
          type: "Feature",
          id: l.id,
          properties: {
            id: l.id,
            naam: l.naam,
            type: l.type,
            buurt: l.stadsdeel,
          },
          geometry: {
            type: "Point",
            coordinates: [l.coordinaten[1], l.coordinaten[0]],
          },
        })),
      };

      observaties = await loadObservations("nl");

      let schoolData = await loadSchoolData();
      let lastId = observaties[observaties.length - 1]?.observationId ?? 0;
      const ongeldigeLocaties = new Set();
      schoolData.forEach((row, i) => {
        const school = locaties.find((s) => s.naam === row.locationName);
        let stadsdeel = school?.stadsdeel;
        if (!school || !stadsdeel) {
          ongeldigeLocaties.add(row.locationName);
          return;
        }

        row.languages.forEach((lang, i) => {
          observaties.push({
            id: lastId + i + 1,
            stadsdeel,
            locationName: school.naam,
            displayName: lang,
            code: row.codes[i],
          });
        });
      });

      console.log(
        "Ongeldige locaties in schooldata:",
        Array.from(ongeldigeLocaties),
      );

      console.log("Observaties na toevoegen schooldata:", observaties);

      map!.addSource("stadsdelen", { type: "geojson", data: geojson });
      map!.addSource("stadsdelen-highlight", {
        type: "geojson",
        data: { type: "FeatureCollection", features: [] },
      });
      map!.addSource("locaties", {
        type: "geojson",
        data: locatiesGeojson,
        promoteId: "id",
      });

      map!.addLayer({
        id: "stadsdelen-fill",
        type: "fill",
        source: "stadsdelen",
        paint: { "fill-color": "#00aa00", "fill-opacity": 0.1 },
      });

      map!.addLayer({
        id: "stadsdelen-outline",
        type: "line",
        source: "stadsdelen",
        paint: { "line-color": "#00aa00", "line-width": 1.5 },
      });

      map!.addLayer({
        id: "stadsdelen-highlight-fill",
        type: "fill",
        source: "stadsdelen-highlight",
        paint: {
          "fill-color": [
            "case",
            ["boolean", ["get", "_selected"], false],
            "#005500",
            "#00aa00",
          ],
          "fill-opacity": [
            "case",
            ["boolean", ["get", "_selected"], false],
            0.3,
            0.2,
          ],
        },
      });

      map!.addLayer({
        id: "stadsdelen-highlight-outline",
        type: "line",
        source: "stadsdelen-highlight",
        paint: {
          "line-color": "#007700",
          "line-width": [
            "case",
            ["boolean", ["get", "_selected"], false],
            3,
            2.5,
          ],
        },
      });

      map.addLayer({
        id: "locatie-circles",
        type: "circle",
        source: "locaties",
        paint: {
          "circle-radius": [
            "case",
            ["boolean", ["feature-state", "hovered"], false],
            8,
            5,
          ],
          "circle-color": "#ff4444",
          "circle-stroke-width": 1.5,
          "circle-stroke-color": "#ffffffbb",
          "circle-opacity": [
            "case",
            ["boolean", ["feature-state", "hovered"], false],
            1,
            0.85,
          ],
        },
      });

      map!.addLayer(
        {
          id: "extra-dimmer",
          type: "background",
          paint: {
            "background-color": "#000000",
            "background-opacity": 0.5,
          },
        },
        "stadsdelen-fill",
      );

      addLabelMarkers(geojson);
      addLocationMarkers();

      let lastHoveredLocatieId: number | null = null;

      map!.on("mousemove", "locatie-circles", (e) => {
        if (!e.features?.length) return;

        const f = e.features[0];
        const id = f.id;

        if (typeof id !== "number") return;

        if (lastHoveredLocatieId !== null && lastHoveredLocatieId !== id) {
          map!.setFeatureState(
            { source: "locaties", id: lastHoveredLocatieId },
            { hovered: false },
          );
          hideSchoolLabel();
        }

        hoveredLocatieId = id;
        lastHoveredLocatieId = id;

        map!.setFeatureState({ source: "locaties", id }, { hovered: true });

        showSchoolLabel(f.properties!.naam, e.lngLat);
      });

      map!.on("mouseleave", "locatie-circles", () => {
        map!.getCanvas().style.cursor = "";

        if (hoveredLocatieId !== null) {
          map!.setFeatureState(
            { source: "locaties", id: hoveredLocatieId },
            { hovered: false },
          );
        }

        hoveredLocatieId = null;
        hideSchoolLabel();
      });

      map!.on("mousemove", "stadsdelen-fill", (e) => {
        if (hoveredLocatieId !== null) return;

        if (!e.features?.length) return;
        const id = e.features[0].id;

        hoveredStadsdeelId = id as number;
        updateHighlightSource();
      });

      map!.on("mouseleave", "stadsdelen-fill", () => {
        hoveredStadsdeelId = null;
        updateHighlightSource();
        map!.getCanvas().style.cursor = "";
      });

      map!.on("click", (e) => {
        const locaties = map!.queryRenderedFeatures(e.point, {
          layers: ["locatie-circles"],
        });

        if (locaties.length > 0) {
          const feature = locaties[0];
          const id = feature.properties?.id ?? feature.id;

          if (id !== undefined && id !== null) {
            selectedLocatieId = Number(id);
            selectedStadsdeelId = null;
            updateHighlightSource();
            return;
          }
        }

        const stadsdelen = map!.queryRenderedFeatures(e.point, {
          layers: ["stadsdelen-fill"],
        });

        if (stadsdelen.length > 0) {
          const id = stadsdelen[0].id;
          if (id !== undefined) {
            selectedStadsdeelId = id as number;
            selectedLocatieId = null;
            updateHighlightSource();
            return;
          }
        }

        selectedLocatieId = null;
        selectedStadsdeelId = null;
        updateHighlightSource();
      });
    });

    return () => {
      clearMarkers();
      hideSchoolLabel();
      map?.remove();
    };
  });
</script>

<ModeWatcher />

<header class="p-6 border-b border-gray-300 dark:border-gray-700 relative">
  <h1 class="text-xl font-bold">Talenkaart Amsterdam</h1>
  <div class="flex gap-2 fixed top-0 right-0 m-2">
    <LanguageToggle
      class="z-50"
      onclick={() => (locale = locale === "nl" ? "en" : "nl")}
      content={locale === "nl" ? "🇳🇱 " : "🇬🇧 "}
    />
    <LightSwitch></LightSwitch>
  </div>
</header>

<!-- <div
  class="px-12 py-4 flex gap-6 items-center bg-gray-50 dark:bg-gray-900 border-b border-gray-200 dark:border-gray-800"
>
  <label class="flex items-center cursor-pointer select-none">
    <input type="checkbox" bind:checked={showLocaties} class="mr-2 w-4 h-4" />
    <span class="text-sm font-medium">Locaties tonen</span>
  </label>

  <label
    class="flex items-center cursor-pointer select-none"
    class:opacity-50={!showLocaties}
  >
    <input
      type="checkbox"
      bind:checked={showLabels}
      disabled={!showLocaties}
      class="mr-2 w-4 h-4"
    />
    <span class="text-sm font-medium">Labels tonen</span>
  </label>
</div> -->

<Card class="m-12 p-0 relative">
  <div bind:this={mapContainer} class="w-full h-[600px] rounded-lg" />
  <div
    class="absolute bottom-4 left-4 bg-white dark:bg-gray-800 p-2 rounded-lg shadow-md"
  >
    <ul class="text-[13px] font-[600] text-gray-700 dark:text-gray-300">
      <li>
        <Checkbox bind:checked={showLocaties} class="inline"></Checkbox>
        <span style:color="green">--</span> Stadsdelen
      </li>
      <li>
        <Checkbox bind:checked={showLocaties} class="inline"></Checkbox>
        <span style:color="red">●</span> Scholen
      </li>
      <li>
        <Checkbox bind:checked={showLocaties} class="inline"></Checkbox>
        <span style:color="blue">●</span> Bibliotheken
      </li>
    </ul>
  </div>
</Card>

{#if selectedLocatieId !== null}
  {@const naam = locaties.find((l) => l.id === selectedLocatieId)?.naam}
  {@const type = locaties.find((l) => l.id === selectedLocatieId)?.type}
  {@const obs = observaties.filter((o) => o.locationName === naam)}
  <Card class="m-12 p-5">
    <h2 class="text-xl font-semibold mb-4">
      {naam}
      (<i>{type}</i>)
    </h2>
    {obs.length} observaties.
    <br />
    <TaalOverzicht observations={obs} {locale} allObservations={observaties} />
  </Card>
{/if}

{#if selectedStadsdeelId !== null}
  {@const naam = stadsdelen.find((s) => s.id === selectedStadsdeelId)?.naam}
  {@const loc = locaties.filter((l) => l.stadsdeel === naam)}
  {@const obs = observaties.filter((o) => o.stadsdeel === naam)}
  <Card class="m-12 p-5">
    <h2 class="text-xl font-semibold mb-4">
      Amsterdam → Stadsdeel
      <i>
        {stadsdelen.find((s) => s.id === selectedStadsdeelId)?.naam}
      </i>
    </h2>
    <Tabs.Root value="observaties" class="">
      <Tabs.List>
        <Tabs.Trigger value="observaties">Talen</Tabs.Trigger>
        <Tabs.Trigger value="locaties">Locaties</Tabs.Trigger>
      </Tabs.List>
      <Tabs.Content value="locaties" class="m-0">
        <i class="font-medium mb-2">
          {loc.length}
          {loc.length == 1 ? "locatie" : "locaties"} in {naam}:
        </i>
        <ul class="list-disc list-inside">
          {#each loc as locatie}
            <li>{locatie.naam} ({locatie.type})</li>
          {/each}
        </ul>
      </Tabs.Content>
      <Tabs.Content value="observaties">
        Onder de <b>{obs.length}</b> ondervraagden in <i>{naam}</i> worden de
        volgende talen gesproken:
        <TaalOverzicht observations={obs} {locale} />
      </Tabs.Content>
    </Tabs.Root>
  </Card>
{/if}

<style>
  @import url("https://fonts.googleapis.com/css2?family=Bebas+Neue&family=Open+Sans:ital,wght@0,300..800;1,300..800&family=Teko:wght@300..700&display=swap");

  :global(.stadsdeel-label) {
    font-family: "Open Sans", serif;
    font-size: 14px;
    font-weight: 600;
    color: #eeffee;
    text-shadow: 1px 2px 0px #000;
    pointer-events: none;
    white-space: nowrap;
    letter-spacing: 0.04em;
    text-transform: uppercase;
  }

  :global(.school-label) {
    font-family: "Bebas Neue", serif;
    font-size: 16px;
    color: #331100;
    background: rgba(255, 255, 255, 0.92);
    padding: 2px 4px;
    border-radius: 4px;
    white-space: nowrap;
    box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
    pointer-events: none;
    margin-bottom: -100px;
  }

  :global(.school-label-permanent) {
    font-family: "Bebas Neue", serif;
    font-size: 11px;
    color: #331100;
    background: rgba(255, 255, 255, 0.85);
    padding: 0px 3px;
    border-radius: 4px;
    white-space: nowrap;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.3);

    pointer-events: none;
    margin-top: -10px;
  }
</style>
