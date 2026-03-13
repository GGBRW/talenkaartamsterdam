<script lang="ts">
  import {
    Map,
    MapControls,
    MapMarker,
    MarkerContent,
    MarkerPopup,
    MarkerLabel,
    MarkerTooltip,
  } from "$lib/components/ui/map";
  import { Card } from "$lib/components/ui/card";
  import { Slider } from "$lib/components/ui/slider";
  import ArrowBigLeft from "@lucide/svelte/icons/arrow-big-left";
  import * as XLSX from "xlsx";
  import { ModeWatcher } from "mode-watcher";

  import LanguageToggle from "../../lib/LanguageToggle.svelte";
  import Stadsdelen from "$lib/Stadsdelen.svelte";
  import LightSwitch from "$lib/components/ui/light-switch/light-switch.svelte";

  const SPREADSHEET_URL = "./Multilingual_Amsterdam_voorGees.xlsx";
  const POSTCODES_URL = "./pc4geoloc.json";
  const ISO_CODES_URL = "./iso_codes.json";
  let isoCodes = {};

  const TURUGVAL_POSTCODES = {
    Buurtcampus: "1067",
    Poort: "1102",
  };

  const TERUGVAL_TALEN = {
    '"nigeriaans"': "pcm",
    'ara ("Marokkaans")': "ary",
    "ara (Marokkaans)": "ary",
    chinees: "cmn",
    '"chinees"': "cmn",
    '"indiaas"': "Indiaas (hindi?)",
    '"afrikaans"': "afr",
    g: "G?",
  };

  type LanguageObservation = {
    observationId: number;
    code: string;
    displayName: string;
    postcode: string;
    lng: number;
    lat: number;
    spreadLng: number;
    spreadLat: number;
    locationName: string;
    stadsdeel: string;
    interviewers: string;
    interviewDate: string;
    proficient: boolean;
    basic: boolean;
    homeLanguage: boolean;
  };

  function makeDisplayName(code: string, locale: string) {
    if (!code) return "(!)";
    if (isoCodes.find((c) => c.code === code)) {
      return isoCodes.find((c) => c.code === code)?.NL || code;
    }
    if (TERUGVAL_TALEN[code]) return code + " (!!)";

    try {
      const dn = new Intl.DisplayNames([locale], { type: "language" });
      const display = dn.of(code);

      if (!display || display.toLowerCase() === code.toLowerCase()) {
        return `${code} (!)`;
      }

      return display;
    } catch {
      return `${code} (!)`;
    }
  }

  function spreadObservation(coords, spreadAmount = 0.02) {
    const spread = spreadAmount * Math.random();
    const angle = Math.random() * 2 * Math.PI;
    coords[0] += Math.cos(angle) * spread;
    coords[1] += Math.sin(angle) * spread;
    return coords;
  }

  function createLanguageObservation(
    base: {
      observationId: number;
      code: string;
      postcode: string;
      locationName: string;
      stadsdeel: string;
      interviewers: string;
      interviewDate: string;
      proficient?: boolean;
      basic?: boolean;
      homeLanguage?: boolean;
    },
    locale: string,
    postcodeLookup: Record<string, { lng: number; lat: number }>,
  ): LanguageObservation {
    let postcodeFourDigits = base.postcode?.toString().match(/(\d{4})/)?.[1];
    if (!postcodeFourDigits) {
      const possibleMatch = Object.entries(TURUGVAL_POSTCODES).find(([key]) =>
        base.locationName.includes(key),
      );
      if (possibleMatch) {
        postcodeFourDigits = possibleMatch[1];
      }
    }

    let coords = { lng: 0, lat: 0 };
    if (postcodeFourDigits) {
      const pc = postcodeFourDigits.toString();
      coords = postcodeLookup[pc] ?? coords;
      // coords = spreadObservation(coords);
      base.postcode = pc;
    }

    let spread = spreadObservation([...coords]);

    return {
      observationId: base.observationId,
      code: base.code,
      displayName: makeDisplayName(base.code, locale),
      postcode: base.postcode,
      lat: coords?.[0] ?? 0,
      lng: coords?.[1] ?? 0,
      spreadLat: spread[0],
      spreadLng: spread[1],
      locationName: base.locationName,
      stadsdeel: base.stadsdeel,
      interviewers: base.interviewers,
      interviewDate: base.interviewDate,
      proficient: base.proficient ?? false,
      basic: base.basic ?? false,
      homeLanguage: base.homeLanguage ?? false,
    };
  }

  async function load(locale: string) {
    const [spreadsheetRes, postcodeRes, isoCodesRes] = await Promise.all([
      fetch(SPREADSHEET_URL),
      fetch(POSTCODES_URL),
      fetch(ISO_CODES_URL),
    ]);

    const buffer = await spreadsheetRes.arrayBuffer();
    const workbook = XLSX.read(buffer);
    const postcodeLookup = await postcodeRes.json();
    isoCodes = await isoCodesRes.json();

    const sheetNames = ["Zuidoost", "NieuwWest"];
    const observations: LanguageObservation[] = [];

    let observationId = 0;
    for (const name of sheetNames) {
      const sheet = workbook.Sheets[name];
      const rows = XLSX.utils.sheet_to_json(sheet, { header: 1 });

      const dataRows = rows.slice(2);

      for (const row of dataRows) {
        if (!row || row.length === 0) return;

        const locationName = row[0];
        const interviewers = row[1];
        const interviewDate = row[2];
        const postcode = row[3];

        const proficientLangs = row[5] ?? "";
        const basicLangs = row[6] ?? "";
        const homeLangs = row[7] ?? "";

        const proficientList = proficientLangs
          .split(",")
          .map((x) => x.trim())
          .filter(Boolean);
        const basicList = basicLangs
          .split(",")
          .map((x) => x.trim())
          .filter(Boolean);
        const homeList = homeLangs
          .split(",")
          .map((x) => x.trim())
          .filter(Boolean);

        const allLangs = new Set([
          ...proficientList,
          ...basicList,
          ...homeList,
        ]);

        for (const code of allLangs) {
          observations.push(
            createLanguageObservation(
              {
                observationId,
                code,
                postcode,
                locationName,
                stadsdeel: name,
                interviewers,
                interviewDate,
                proficient: proficientList.includes(code),
                basic: basicList.includes(code),
                homeLanguage: homeList.includes(code),
              },
              locale,
              postcodeLookup,
            ),
          );
        }

        observationId++;
      }
    }

    return observations;
  }

  let isLoading = $state(true);
  let locale = $state("nl");
  let observations: LanguageObservation[] = $state([]);
  let uniqueObservations = $state(false);

  $effect(() => {
    isLoading = true;
    const currentLocale = locale;

    load(locale).then((data) => {
      if (currentLocale === locale) {
        observations = data;
        console.log(observations);
        isLoading = false;
        spreadObservations();
      }
    });
  });

  let spreadAmount = $state(0.02);
  let spreadAmountBuurtcampus = $state(0.02);
  let spreadAmountPoort = $state(0.02);
  function spreadObservations() {
    observations = observations.map((obs) => {
      let amount = spreadAmount;
      if (obs.postcode === "1067" && obs.locationName.includes("Buurtcampus"))
        amount = spreadAmountBuurtcampus;
      else if (obs.postcode === "1102" && obs.locationName.includes("Poort"))
        amount = spreadAmountPoort;

      const spread = spreadObservation([obs.lat, obs.lng], amount);
      return { ...obs, spreadLat: spread[0], spreadLng: spread[1] };
    });
  }

  setTimeout(() => {
    console.log(
      Array.from(new Set(observations.map((item) => item.code)).values()),
    );
  }, 1000);

  let selectedObject = $state<{ name: string } | null>(null);

  let locaties = [
    { name: "Buurtcampus", postcode: "1067", lat: 52.382, lng: 4.803 },
    { name: "Poort", postcode: "1102", lat: 52.319, lng: 4.942 },
  ];
</script>

<title>Talen van Amsterdam</title>

<ModeWatcher />

<header class="flex p-4 border-b-1 border-muted/80">
  <h1 class=" text-[18px] font-semibold">Talen van Amsterdam</h1>
  <div class="flex gap-2 fixed top-0 right-0 m-2">
    <LanguageToggle
      class="z-50"
      onclick={() => (locale = locale === "nl" ? "en" : "nl")}
      content={locale === "nl" ? "🇳🇱 " : "🇬🇧 "}
    />
    <LightSwitch />
  </div>
</header>

<Card class="h-[500px] overflow-hidden p-0 m-5">
  <Map center={[4.8952, 52.3702]} zoom={12}>
    <MapControls />
    {#each !uniqueObservations ? Array.from(new globalThis.Map(observations.map( (item) => [item.observationId, item], )).values()) : observations as observation}
      <!-- {#if observation.postcode !== "1067" && observation.postcode !== "1102"} -->
      <MapMarker
        longitude={observation.spreadLng}
        latitude={observation.spreadLat}
      >
        <MarkerContent>
          <div
            class="bg-blue-900 size-2.5 rounded-full border-1 border-blue-100"
          ></div>
        </MarkerContent>

        <MarkerTooltip>
          {observation.displayName}
        </MarkerTooltip>

        <MarkerPopup>
          <div class="space-y-.7">
            <p>
              <span class="font-medium">Talen: </span>
              <span class="text-muted-foreground text-xs">
                {observation.displayName}
              </span>
            </p>
            <p>
              <span class="font-medium">Postcode: </span>
              <span class="text-muted-foreground text-xs">
                {observation.postcode}
              </span>
            </p>
            <p>
              <span class="font-medium">Locatie: </span>
              <span class="text-muted-foreground text-xs">
                {observation.lat.toFixed(3)},{observation.lng.toFixed(3)}
              </span>
            </p>
            <p>
              <span class="font-medium">Locatienaam: </span>
              <span class="text-muted-foreground text-xs">
                {observation.locationName}
              </span>
            </p>
            <p>
              <span class="font-medium">Stadsdeel: </span>
              <span class="text-muted-foreground text-xs">
                {observation.stadsdeel}
              </span>
            </p>
            <p>
              <span class="font-medium">Interviewer(s): </span>
              <span class="text-muted-foreground text-xs">
                {observation.interviewers}
              </span>
            </p>
            <p>
              <span class="font-medium">Datum interview: </span>
              <span class="text-muted-foreground text-xs">
                {observation.interviewDate}
              </span>
            </p>
            <p>
              <span class="font-medium">Observatie ID: </span>
              <span class="text-muted-foreground text-xs">
                {observation.observationId}
              </span>
            </p>
            <p>
              <span class="font-medium">Proficient: </span>
              <span class="text-muted-foreground text-xs">
                {observation.proficient ? "Ja" : "Nee"}
              </span>
            </p>
            <p>
              <span class="font-medium">Basic: </span>
              <span class="text-muted-foreground text-xs">
                {observation.basic ? "Ja" : "Nee"}
              </span>
            </p>
            <p>
              <span class="font-medium">Home language: </span>
              <span class="text-muted-foreground text-xs">
                {observation.homeLanguage ? "Ja" : "Nee"}
              </span>
            </p>
          </div>
        </MarkerPopup>
      </MapMarker>
      <!-- {/if} -->
    {/each}
    <Stadsdelen
      geojsonData={"./stadsdelen.json"}
      onselect={(stadsdeel) =>
        (selectedObject = stadsdeel ? { name: stadsdeel } : null)}
    />
    {#each locaties as locatie}
      <MapMarker
        longitude={locatie.lng}
        latitude={locatie.lat}
        onclick={() => setTimeout(() => (selectedObject = locatie), 1)}
      >
        <MarkerContent>
          <div class="bg-blue-100 size-6 rounded-full border-2 border-blue-900">
            <div class="rounded-full m-1 bg-pink-400 size-3"></div>
          </div>
          <MarkerLabel position="bottom">
            {locatie.name}
          </MarkerLabel>
        </MarkerContent>

        <MarkerTooltip>
          {locatie.name}
        </MarkerTooltip>
      </MapMarker>
    {/each}
  </Map>
</Card>

<Card class="mx-5 mb-5 p-5">
  {#if observations}
    {@const colors = [
      "hsl(205, 100%, 37%)",
      "hsl(45, 100%, 50%)",
      "hsl(270, 100%, 50%)",
      "hsl(120, 100%, 50%)",
      "hsl(195, 100%, 50%)",
      "hsl(20, 100%, 50%)",
      "hsl(240, 100%, 50%)",
      "hsl(48, 50%, 50%)",
      "hsl(340, 80%, 50%)",
    ]}
    {@const observationsFiltered = (() => {
      if (!selectedObject) return observations;
      if (selectedObject.name === "Buurtcampus")
        return observations.filter((o) => o.postcode === "1067");
      if (selectedObject.name === "Poort")
        return observations.filter((o) => o.postcode === "1102");
      return observations.filter(
        (o) =>
          o.stadsdeel.toLowerCase() ===
          selectedObject.name.replace("-", "").toLowerCase(),
      );
    })()}

    {@const grouped = globalThis.Map.groupBy(
      observationsFiltered,
      (o) => o.observationId,
    )}

    {@const num = grouped.size}

    {@const langs = Array.from(new Set(observationsFiltered.map((o) => o.code)))
      .map((code) => {
        const occ = [...grouped.values()].filter((obs) =>
          obs.some((o) => o.code === code),
        ).length;
        return {
          code,
          occ,
          pcntg: (occ / num) * 100,
        };
      })
      .sort((a, b) => b.occ - a.occ)}

    {@const coTalen = new globalThis.Map(
      langs.map(({ code }) => {
        const ids = [...grouped.entries()]
          .filter(([_, obs]) => obs.some((o) => o.code === code))
          .map(([id]) => id);
        const companions = ids
          .flatMap((id) => grouped.get(id) ?? [])
          .filter((o) => o.code !== code)
          .map((o) => o.code);
        const counts = globalThis.Map.groupBy(companions, (c) => c);
        const top = [...counts.entries()]
          .map(([c, arr]) => ({ code: c, count: arr.length }))
          .sort((a, b) => b.count - a.count);
        const topCode = top[0]?.code ?? null;
        const topPcntg = top[0] ? (top[0].count / ids.length) * 100 : 0;
        return [code, { topCode, topPcntg, alle: top }];
      }),
    )}

    <div class="flex items-center gap-2">
      {#if selectedObject}
        <ArrowBigLeft size="16" opacity=".67" />
      {/if}
      <h1 class="font-bold text-lg">
        {selectedObject ? selectedObject.name : "Alle talen"} ({observationsFiltered.length})
      </h1>
    </div>
    <div class="flex h-2 overflow-hidden rounded-full bg-gray-200">
      {#each langs as lang, i}
        <div
          class="h-full"
          style={`width: ${lang.pcntg}%; background-color: ${
            colors[Math.min(i, colors.length - 1)]
          }`}
        ></div>
      {/each}
    </div>
    <ul>
      {#each Array.from(langs.sort((a, b) => b.pcntg - a.pcntg)) as lang, i}
        {@const co = coTalen.get(lang.code)}
        {@const coDisplayName = makeDisplayName(co?.topCode ?? "", locale)}

        {#if i == colors.length - 1}
          <span
            style={`color: ${colors[Math.min(langs.indexOf(lang), colors.length - 1)]};`}
            >●</span
          >
          <b><i>Andere talen:</i></b>
          <span class="text-muted-foreground text-[12px]">
            {langs
              .slice(colors.length - 1)
              .reduce((acc, curr) => acc + curr.pcntg, 0)
              .toFixed(1)}%
          </span>
        {/if}
        <li>
          <span
            style={`color: ${colors[Math.min(langs.indexOf(lang), colors.length - 1)]};`}
            style:opacity={i > colors.length - 2 ? 0 : 1}>●</span
          >
          {makeDisplayName(lang.code, "nl")}

          <span class="text-muted-foreground text-[12px]"
            ><b>{lang.pcntg.toFixed(1)}%</b> ({lang.occ} sprekers)</span
          >

          <span class="text-muted-foreground text-[12px]">
            → meest gesproken in combinatie met <b>{coDisplayName}</b>
            ({co?.topPcntg.toFixed(1)}%)</span
          >
        </li>
      {/each}
    </ul>
  {/if}
</Card>

<div class="mx-5 mb-5 text-sm text-muted-foreground">
  Elke taal als losse observaties bekijken:
  <input
    type="checkbox"
    class="relative top-[2px] left-1"
    bind:checked={uniqueObservations}
  />

  <br /><br />
  Verspreiding:
  <input
    type="range"
    min="0"
    max="0.1"
    step="0.005"
    bind:value={spreadAmount}
    onchange={spreadObservations}
    class="relative top-1 cursor-pointer accent-red-500 appearance-none rounded-full bg-gray-500/15"
  />
  <b>{spreadAmount}</b>
  <br /><br />
  Verspreiding Buurtcampus (1067):
  <input
    type="range"
    min="0"
    max="0.1"
    step="0.005"
    bind:value={spreadAmountBuurtcampus}
    onchange={spreadObservations}
    class="relative top-1 cursor-pointer accent-red-500 appearance-none rounded-full bg-gray-500/15"
  />
  <b>{spreadAmountBuurtcampus}</b>
  <br /><br />
  Verspreiding Poort (1102):
  <input
    type="range"
    min="0"
    max="0.1"
    step="0.005"
    bind:value={spreadAmountPoort}
    onchange={spreadObservations}
    class="relative top-1 cursor-pointer accent-red-500 appearance-none rounded-full bg-gray-500/15"
  />
  <b>{spreadAmountPoort}</b>

  <br /><br />

  <p>
    <b>Aantal gesproken talen:</b>
    {observations ? observations.length : "Laden..."}
  </p>
  <p>
    <b>Aantal observaties:</b>
    {observations
      ? new Set(observations.map((o) => o.observationId)).size
      : "Laden..."}
  </p>
  <p>
    <b>Postcodes ({new Set(observations.map((o) => o.postcode)).size}):</b>
    {@html Array.from(new Set(observations.map((o) => o.postcode)))
      .map((x) =>
        observations.filter((o) => o.postcode === x).length > 0
          ? `${x} (${observations.filter((o) => o.postcode === x).length})`
          : x,
      )
      .filter((x) => x !== "Unknown")
      .join(", ")}
  </p>
  <p>
    <b>Talen ({new Set(observations.map((o) => o.displayName)).size}):</b>
    {@html Array.from(new Set(observations.map((o) => o.displayName)))
      .map((x) =>
        observations.filter((o) => o.displayName === x).length > 0
          ? `${x} (${observations.filter((o) => o.displayName === x).length})`
          : x,
      )
      .map((x) => x.replace("(!)", "<b class='text-blue-500'>!</b>"))
      .map((x) => x.replace("(!!)", "<b class='text-red-500'>!!</b>"))
      .filter((x) => x !== "Unknown")
      .join(", ")}
  </p>
</div>
