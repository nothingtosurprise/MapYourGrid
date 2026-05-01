---
hide:
  - navigation
  - toc
  - footer
---

<div class="page-headers">
<h1>Map It 📍</h1>
</div>

??? success "INTRODUCTION (Click Me)"
    Welcome to our interactive launchpad and hub for contributing to power grid mapping via OpenStreetMap! Click on a country or state below to start mapping power infrastructure directly in iD or JOSM. :rocket:
    If this is your first time grid mapping, please go through the [JOSM Starter-Kit](https://mapyourgrid.org/starter-kit/#josm-starter-kit) or [iD Starter-Kit](https://mapyourgrid.org/starter-kit/#id-starter-kit). You can use the **#MapYourGrid** hashtag in your changeset to show your support for our initiative when you make an edit! To start mapping, please open [JOSM](https://josm.openstreetmap.de/), ensure that remote control is activated in `Preferences` and load your data: 

    1. The **Default Transmission (50 kV+)** pulls all power infrastructure relevant for the **transmission grid**. For more details about which data is pulled via Overpass please read our [OpenStreetMap Grid Definitions](https://github.com/open-energy-transition/osm-grid-definition). Distribution grids are barely visible in satellite data and should therefore only be mapped in individual cases.
    2. The Osmose, Global Energy Monitor, and Wikidata buttons provide **hint layer** data, which you can read about in our page [Strategies](strategies.md). Please note that hint layers only work at a national level, except for Osmose in certain cases. 

<!-- Beginning of Map section-->
<style>
#map {
    position: relative;
    z-index: 1;
    width: 100%;
    height: 55vh;
    min-height: 400px;
    margin: 20px 0;
    border-radius: 8px;
    border: 1px solid #ddd;
}

@media (max-width: 768px) {
    #map {
        height: 400px;
    }
}
</style>

<!-- Osmose button-->
<div id="osmose-panel" style="display:none; margin-bottom:1em;">
  <label for="osmoseIssue">Issue type:</label>
  <select id="osmoseIssue">
    <optgroup label="Power lines (item 7040)">
                <option value="7040:1">Lone power tower or pole (Class 1)</option>
                <option value="7040:2"selected>Default: Unfinished power transmission line (Class 2) (recommended for beginners ⭐)</option>
                <option value="7040:3">Connection between different voltages (Class 3)</option>
                <option value="7040:4">None power node on power way (Class 4)</option>
                <option value="7040:5">Missing power tower or pole (Class 5)</option>
                <option value="7040:6">Unfinished power distribution line (Class 6)</option>
                <option value="7040:7">Unmatched voltage of line on substation (Class 7)</option>
                <option value="7040:8">Power support line management suggestion (Class 8)</option>
                <option value="7040:95">missing power=line in the area (Class 95)</option>
              </optgroup>
    </select>
  <div class="query-version">Warning: Some countries only work on national level with Osmose</div>
</div>

<!-- GEM button-->
<div id="gem-panel" style="display:none; margin-bottom:1em;">
  <div class="query-version">Warning: GEM only works on national level</div>
</div>

<!-- TZ Mapyoursolar button -->
<div id="solar-panel" style="display:none; margin-bottom:1em;">
  <div class="query-version">Warning: TZ-Solar only works on national level</div>
</div>

<!-- Wikidata button-->
<div id="wikidata-panel" style="display:none; margin-bottom:1em;">
  <label for="wikidataType">Data type:</label>
  <select id="wikidataType">
    <option value="All power-related infrastructure" selected>All power-related infrastructure</option>
    <option value="substations">Substations</option>
    <option value="powerplants">Power Plants</option>
  </select>
  <div class="query-version">Warning: Wikidata only works on national level</div>
</div>

<div id="wind-panel" style="display:none; margin-bottom:1em;">
  <div class="query-version">Warning: GRW Wind only works on national level</div>
</div>

<!-- PPM button -->
<div id="ppm-panel" style="display:none; margin-bottom:1em;">
  <label for="ppmType">Data type:</label>
  <select id="ppmType">
    <option value="Rejected power plants" selected>Rejected power plants</option>
  </select>
  <div class="query-version">Warning: Powerplantmatching only works on national level</div>
</div>


<div id="map"></div>

<link rel="stylesheet" href="https://unpkg.com/leaflet@1.7.1/dist/leaflet.css" />
<script src="https://unpkg.com/leaflet@1.7.1/dist/leaflet.js"></script>

<!-- Start of script! -->
<script>
// ———————————————
// Beginning
// ———————————————

let selectedEditor = 'josm';

// ———————————————
// Name overrides (some names are different, check the API or geojosn)
// ———————————————
const OVERRIDES = {
  osmoseCountries: {
    "Bosnia and Herzegovina": "bosnia_herzegovina",
    "eSwatini": "swaziland",
    "Republic of the Congo": "congo_brazzaville",
    "Democratic Republic of the Congo": "congo_kinshasa",
    "United States": "usa",
    "UnitedStates": "usa"
  },
  osmoseRegions: {
    "Centre-ValdeLoire": "centre",
    "MatoGrossodoSul": "mato_grosso_do_sul",
    "MinasGerais": "minas_gerais",
    "Cataluña": "catalunya",
    "NeiMongol": "inner_mongolia"
  },
  wikidata: {
    "China": "People's_Republic_of_China"
  },
  MapYourSolar: {
   "Ivory Coast": "Cote_d'Ivoire"
  }
};

function getOverride(name, kind) {
  if (!name || !kind) return null;
  return (OVERRIDES[kind] && OVERRIDES[kind][name]) || null;
}
const countryNameMap = {};


// helper to create osmose-compatible keys 
function slugifyForOsmose(name){
  if (!name) return '';
  let s = String(name);
  // 1) split lowercase->Uppercase boundaries: "MatoGrossodoSul" -> "Mato_Grosso_do_Sul"
  s = s.replace(/([a-z0-9])([A-Z])/g, '$1_$2');
  // 2) split Uppercase -> UppercaseLowercase boundaries:
  //    "HTMLParser" -> "HTML_Parser", also handles "CastillaYLeon" -> "Castilla_Y_Leon"
  s = s.replace(/([A-Z])([A-Z][a-z])/g, '$1_$2');
  // 3) Normalize & remove accents
  s = s.normalize('NFD').replace(/[\u0300-\u036f]/g, '');
  // 4) Replace any non-alphanumeric with underscore
  s = s.replace(/[^0-9A-Za-z]+/g, '_');
  // 5) Collapse multiple underscores, trim edges, lowercase
  s = s.replace(/_+/g, '_').replace(/^_|_$/g, '').toLowerCase();
  return s;
}

// ———————————————
// Map
// ———————————————
// Define world bounds (southWest & northEast corners)
const southWest = L.latLng(-90, -200);
const northEast = L.latLng( 90,  200);
const worldBounds = L.latLngBounds(southWest, northEast);

// Create the map with maxBounds & disable world wrapping
const map = L.map('map', {
  worldCopyJump: false,      // disable dragging to duplicate worlds
  minZoom: 2.2,               
  maxZoom: 18,
  maxBounds: worldBounds,    // restrict the view
  maxBoundsViscosity: 0.3    // “sticky” at the edges
}).setView([20, 0], 2);

L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 18,
    attribution: '© <a href="https://www.openstreetmap.org/copyright" target="_blank" rel="noopener noreferrer">OpenStreetMap</a> contributors'
}).addTo(map);

// Add full screen button
L.Control.Fullscreen = L.Control.extend({
  onAdd() {
    const btn = L.DomUtil.create('button', 'leaflet-bar leaflet-control');
    btn.innerHTML = '&#x26F6;';
    btn.title = 'Toggle fullscreen';
    btn.style.cssText = 'width:30px;height:30px;cursor:pointer;font-size:14px;line-height:28px;background:white;border:none;';
    L.DomEvent.disableClickPropagation(btn);
    L.DomEvent.on(btn, 'click', () =>
      document.fullscreenElement ? document.exitFullscreen() : map.getContainer().requestFullscreen()
    );
    return btn;
  },
  onRemove() {},
});
new L.Control.Fullscreen({ position: 'topleft' }).addTo(map);


// Useless at the moment since the regions layer becomes active anyway at a certain zoom
const largeCountries = ['BR', 'US', 'CA', 'IN', 'MX', 'AU', 'CN'];
const zoomThreshold = 5; // Zoom level at which to show regions instead of countries

// Layers for countries and regions
const countriesLayer = L.geoJSON(null, {
    style: { color: '#3388ff', weight: 1 }
}).addTo(map);

const regionsLayer = L.geoJSON(null, {
    style: { color: '#2c1aff', weight: 1 }
});


// Make the popup message bigger and nicer for new mappers (like Ad-blocker...)
map.options.maxPopupWidth = 300;

// ———————————————
// Overpass repo. Each Folder in the queries folder for 1 mode (button)
// ———————————————
const GITHUB_API_QUERIES =
  'https://api.github.com/repos/open-energy-transition/osm-grid-definition/contents/queries';

/** Base URL for raw file fetches (OverpassQL and version.txt) */
const RAW_BASE =
  'https://raw.githubusercontent.com/open-energy-transition/osm-grid-definition/main/queries';

// ———————————————
// Modes and selection
// ———————————————
let currentMode = null;

// 2a) discover all folders under /queries
async function loadModes() {
  const res = await fetch(GITHUB_API_QUERIES);
  if (!res.ok) throw new Error('Cannot load query modes from GitHub API');
  const items = await res.json();
  // keep only directory entries
  const modes = items.filter(i => i.type === 'dir').map(i => i.name);
  if (modes.length === 0) throw new Error('No query folders found');
  return modes;
}

// 2.1a) fetch the version.txt for the given mode
async function fetchVersion(mode) {
  const url = `${RAW_BASE}/${mode}/version.txt`;
  const r   = await fetch(url);
  if (!r.ok) throw new Error('version not found');
  return (await r.text()).trim();
}

// 2b) render one button per folder name
async function initQueryUI() {
  // 1. Load real modes from GitHub
  let modes = await loadModes();
  modes = modes.filter(m => m !== 'Osmose_issues');
  modes.sort((a, b) => {
    if (a === 'Default') return -1;
    if (b === 'Default') return 1;
    return a.localeCompare(b);
  });

  // inject our special modes:
  modes.splice(2, 0, 'Osmose_issues', 'GEM_powerplants', 'MapYourSolar', 'Wikidata', 'GRW_wind' ,'PPM');

  currentMode = modes.includes('Default') ? 'Default' : modes[0];

  // 2. Create two sibling containers, then insert them above the map
  const mapEl = document.getElementById('map');

  // JOSM or ID buttons
  const editorUiWrapper = document.createElement('div');
  editorUiWrapper.id = 'editor-ui-wrapper';
  editorUiWrapper.innerHTML = `
    <strong> Choose Your Editor</strong>
    <div id="editor-choice">
      <button id="josm-btn" class="query-btn active">JOSM</button>
      <button id="id-btn" class="query-btn">iD Editor</button>
    </div>
    <div id="url-container" style="display:none;">
      <p>Copy this URL for iD Editor's "Custom Map Data":</p>
      <div class="url-input-wrapper">
        <input type="text" id="url-display" readonly>
        <button id="copy-btn">Copy</button>
      </div>
    </div>
  `;
  mapEl.parentNode.insertBefore(editorUiWrapper, mapEl);
  const josmBtn = document.getElementById('josm-btn');
  const idBtn = document.getElementById('id-btn');
  const urlContainer = document.getElementById('url-container');
  const editorChoiceDiv = document.getElementById('editor-choice');

  josmBtn.addEventListener('click', () => {
    selectedEditor = 'josm';
    josmBtn.classList.add('active');
    idBtn.classList.remove('active');
    urlContainer.style.display = 'none';

    const existingWarning = document.getElementById('id-editor-warning');
    if (existingWarning) {
      existingWarning.remove();
    }

    const overpassButtons = document.getElementById('overpass-buttons');
    if (overpassButtons) {
        overpassButtons.style.opacity = '1';
        overpassButtons.style.pointerEvents = 'auto';
    }
  });

  idBtn.addEventListener('click', () => {
    selectedEditor = 'id';
    idBtn.classList.add('active');
    josmBtn.classList.remove('active');

    const warningMsg = document.createElement('div');
    warningMsg.id = 'id-editor-warning';
    warningMsg.className = 'query-version';
    warningMsg.textContent = 'Only the "Tools and Hints" work with iD editor';
    warningMsg.style.marginTop = '0.5em';
    editorChoiceDiv.appendChild(warningMsg);
    
    // 3) Disable query buttons if id editor is on
    const overpassButtons = document.getElementById('overpass-buttons');
    if (overpassButtons) {
        overpassButtons.style.opacity = '0.5';
        overpassButtons.style.pointerEvents = 'none';
    }
  });

  // Add instructional text below the map
  const introBox = document.createElement('div');
  introBox.id = 'map-intro';
  introBox.className = 'introbox';
  introBox.innerHTML = `
    <strong>📍LOAD YOUR GRID INFRASTRUCTURE</strong><br/>
    <ol style="margin: 0.5em 0 0 1em; padding-left: 1em;">
      <li>Click on the country you want to load in JOSM. Zoom in to select states or provinces.</li>
      <li>Click on Tools and Hints to download the data layers that will support you in grid mapping.</li>
      <li>Don't forget to checkout the Map Legend, Good First Lines, Hot Keys and Curated Electrical Grid Maps below.</li>
      <li>Remember this is a specific OpenStreetMap extract and some other existing objects may remain hidden while you contribute. Always check OpenStreetMap data <a href="/starter-kit/#5-load-power-infrastructure-into-josm">to avoid any conflicts or double mapping</a> on save.</li>
      
    </ol>
    `;

  // Insert after the map
  mapEl.parentNode.insertBefore(introBox, mapEl.nextSibling);

  // — Row 1 title —
  const overpassTitle = document.createElement('div');
  overpassTitle.className = 'tools-header';
  overpassTitle.textContent = 'Transmission Overpass Query';
  mapEl.parentNode.insertBefore(overpassTitle, mapEl);

  const overpassContainer = document.createElement('div');
  overpassContainer.id = 'overpass-buttons';
  mapEl.parentNode.insertBefore(overpassContainer, mapEl);
  
  // — Row 2 title —
  const toolTitle = document.createElement('div');
  toolTitle.className = 'tools-header';
  toolTitle.textContent = 'Tools and Hints';
  mapEl.parentNode.insertBefore(toolTitle, mapEl);

  const toolContainer = document.createElement('div');
  toolContainer.id = 'tool-buttons';
  mapEl.parentNode.insertBefore(toolContainer, mapEl);

  // 3. Render each mode into the appropriate row
  for (const mode of modes) {
    let group;
    if (mode === 'Osmose_issues') {
      group = renderOsmoseButtonGroup();
    } else if (mode === 'GEM_powerplants') {
      group = renderGEMButtonGroup();
    } else if (mode === 'MapYourSolar') {
      group = renderSolarButtonGroup();
    } else if (mode === 'Wikidata') {
      group = renderWikidataButtonGroup();
    } else if (mode === 'GRW_wind') {
      group = renderWindButtonGroup();
    } else if (mode === 'PPM') {
    group = renderPPMButtonGroup();
    }
    else {
      group = renderModeButtonGroup(mode);
    }

    // Tools go in the second row, everything else in the first
    if (['Osmose_issues', 'GEM_powerplants', 'MapYourSolar', 'Wikidata', 'GRW_wind', 'PPM'].includes(mode)) {
      toolContainer.appendChild(group);
    } else {
      overpassContainer.appendChild(group);
    }
  }

  const panelWrapper = document.createElement('div');
  panelWrapper.id = 'panel-wrapper';
  panelWrapper.style.margin = '1em 0';  // optional spacing

  // grab (and remove) the existing panels from their old position
  const osmose   = document.getElementById('osmose-panel');
  const wikidata = document.getElementById('wikidata-panel');
  const ppm      = document.getElementById('ppm-panel');
  const gem      = document.getElementById('gem-panel');
  const solar     = document.getElementById('solar-panel');
  const wind     = document.getElementById('wind-panel');

  // append them into our wrapper
  panelWrapper.appendChild(osmose);
  panelWrapper.appendChild(wikidata);
  panelWrapper.appendChild(ppm);
  panelWrapper.appendChild(gem);
  panelWrapper.appendChild(solar);
  panelWrapper.appendChild(wind);

  // finally, drop that wrapper just before the map div
  mapEl.parentNode.insertBefore(panelWrapper, mapEl);
}


// ———————————————
// Buttons
// ———————————————
function renderOsmoseButtonGroup() {
  const btn = document.createElement('button');
  btn.textContent = 'Osmose Issues';
  btn.classList.add('query-btn');
  if (currentMode === 'Osmose_issues') btn.classList.add('active');
  btn.onclick = () => selectMode('Osmose_issues', btn);

  const ver = document.createElement('div');
  ver.classList.add('query-version');
  ver.textContent = '';  // no version

  // Osmose website link
  const info = document.createElement('div');
  info.classList.add('query-version');
  info.style.marginTop = '0.2rem';
  info.innerHTML =
   '<a href="https://osmose.openstreetmap.fr/" target="_blank">osmose.openstreetmap.fr/</a>';
   

  const group = document.createElement('div');
  group.classList.add('query-group');
  group.appendChild(btn);
  group.appendChild(ver);
  group.appendChild(info);
  return group;
}

function renderGEMButtonGroup() {
  const btn = document.createElement('button');
  btn.textContent = 'Global Energy Monitor - Power Plants';
  btn.classList.add('query-btn');
  if (currentMode === 'GEM_powerplants') btn.classList.add('active');
  btn.onclick = () => selectMode('GEM_powerplants', btn);
  
  // GEM website link + CC BY 4.0
  const info = document.createElement('div');
  info.classList.add('query-version');
  info.style.marginTop = '0.2rem';
  info.innerHTML =
   '<a href="https://globalenergymonitor.org/" target="_blank">globalenergymonitor.org</a>' +
   ' | CC BY 4.0';

  const group = document.createElement('div');
  group.classList.add('query-group');
  group.appendChild(btn);
  group.appendChild(info);
  return group;
}

function renderSolarButtonGroup() {
  const btn = document.createElement('button');
  btn.textContent = 'Transition Zero - Solar Asset Mapper';
  btn.classList.add('query-btn');
  if (currentMode === 'MapYourSolar') btn.classList.add('active');
  btn.onclick = () => selectMode('MapYourSolar', btn);
  
  // TZ website link + CC BY NC 4.0
  const info = document.createElement('div');
  info.classList.add('query-version');
  info.style.marginTop = '0.2rem';
  info.innerHTML =
   '<a href="https://www.transitionzero.org/products/solar-asset-mapper/download" target="_blank">transitionzero.org</a>' +
   ' | Q3-2025 | CC BY NC 4.0';

  const group = document.createElement('div');
  group.classList.add('query-group');
  group.appendChild(btn);
  group.appendChild(info);
  return group;
}

function renderWikidataButtonGroup() {
  const btn = document.createElement('button');
  btn.textContent = 'Wikidata';
  btn.classList.add('query-btn');
  if (currentMode === 'Wikidata') btn.classList.add('active');
  btn.onclick = () => selectMode('Wikidata', btn);

   // Wiki repo link

  const ver = document.createElement('div');
  ver.classList.add('query-version');
  ver.textContent = ''; // no version for now

  const info = document.createElement('div');
  info.classList.add('query-version');
  info.style.marginTop = '0.2rem';
  info.innerHTML =
   '<a href="https://github.com/open-energy-transition/osm-wikidata-toolset" target="_blank">Repository</a>';

  const group = document.createElement('div');
  group.classList.add('query-group');
  group.appendChild(btn);
  group.appendChild(info);
  return group;
}

function renderWindButtonGroup() {
  const btn = document.createElement('button');
  btn.textContent = 'Global Renewables Watch - Wind Turbines';
  btn.classList.add('query-btn');
  if (currentMode === 'GRW_wind') btn.classList.add('active');
  btn.onclick = () => selectMode('GRW_wind', btn);
  
  // TZ website link + CC BY NC 4.0
  const info = document.createElement('div');
  info.classList.add('query-version');
  info.style.marginTop = '0.2rem';
  info.innerHTML =
   '<a href="https://github.com/microsoft/global-renewables-watch/tree/main" target="_blank">Global Renewables Watch</a>' +
   ' | Q2-2024 ';

  const group = document.createElement('div');
  group.classList.add('query-group');
  group.appendChild(btn);
  group.appendChild(info);
  return group;
}

function renderPPMButtonGroup() {
  const btn = document.createElement('button');
  btn.textContent = 'powerplantmatching';
  btn.classList.add('query-btn');
  if (currentMode === 'PPM') btn.classList.add('active');
  btn.onclick = () => selectMode('PPM', btn);

  const info = document.createElement('div');
  info.classList.add('query-version');
  info.style.marginTop = '0.2rem';
  info.innerHTML =
   '<a href="https://github.com/open-energy-transition/mapit-osm/tree/main" target="_blank">Repository</a>';

  const group = document.createElement('div');
  group.classList.add('query-group');
  group.appendChild(btn);
  group.appendChild(info);
  return group;
}

const versionCache = new Map();
function renderModeButtonGroup(mode) {
  const btn = document.createElement('button');
  // I overrided the button name for Default, but the file in github is still Default
  if (mode === 'Default') {
  btn.textContent = 'Default Transmission (50 kV+)';
  } else {
  btn.textContent = mode.replace(/_/g, ' ');
  }
  btn.classList.add('query-btn');
  if (mode === currentMode) btn.classList.add('active');
  btn.onclick = () => selectMode(mode, btn);

  // —— make version badge into a link to the GitHub folder ——
  const verLink = document.createElement('a');
  verLink.classList.add('query-version');
  verLink.target = '_blank';
  // encode mode so "400kv+" becomes "400kv%2B"
  const repoFolderUrl =
   `https://github.com/open-energy-transition/osm-grid-definition/tree/main/queries/` +
   encodeURIComponent(mode);
  verLink.href = repoFolderUrl;  
  verLink.textContent = 'v…';

  // update the badge in the background (cached to avoid duplicate fetches)
  if (versionCache.has(mode)) {
    verLink.textContent = `v${versionCache.get(mode)}`;
  } else {
    fetchVersion(mode)
      .then(v => {
        versionCache.set(mode, v);
        verLink.textContent = `v${v}`;
      })
      .catch(() => {
        verLink.textContent = 'v?';
      });
  }

  const group = document.createElement('div');
  group.classList.add('query-group');
  group.appendChild(btn);
  group.appendChild(verLink);
  return group;
}

// helper to swap modes and show/hide the Osmose and wikidata UI panels
function selectMode(mode, btn) {
  currentMode = mode;
  document.querySelectorAll('#overpass-buttons .query-btn, #tool-buttons .query-btn')
          .forEach(b => b.classList.toggle('active', b === btn));
  // Osmose
  document.getElementById('osmose-panel').style.display =
    mode === 'Osmose_issues' ? 'block' : 'none';

  // Wikidata
  document.getElementById('wikidata-panel').style.display =
    mode === 'Wikidata' ? 'block' : 'none';

  // PPM    
  document.getElementById('ppm-panel').style.display =
  mode === 'PPM' ? 'block' : 'none';

  //GEM
  document.getElementById('gem-panel').style.display =
  mode === 'GEM_powerplants' ? 'block' : 'none';

  // TZ solar
  document.getElementById('solar-panel').style.display =
  mode === 'MapYourSolar' ? 'block' : 'none';

  // GRW wind
  document.getElementById('wind-panel').style.display =
  mode === 'GRW_wind' ? 'block' : 'none';
}

//for id
function displayUrlForId(url) {
    const container = document.getElementById('url-container');
    const input = document.getElementById('url-display');
    const copyBtn = document.getElementById('copy-btn');

    input.value = url;
    container.style.display = 'block';
    container.scrollIntoView({ behavior: 'smooth', block: 'center' });
    
    copyBtn.textContent = 'Copy';
    copyBtn.onclick = () => {
        navigator.clipboard.writeText(url).then(() => {
            copyBtn.textContent = 'Copied!';
        }).catch(err => {
            alert('Failed to copy. Please select the text and press Ctrl+C.');
        });
    };
}

// ———————————————
// Clicking
// ———————————————
// 2d) unified click handler for country (level 2) & region (level 4)
async function handleAreaClick(iso, level, layer) {
  const name = level === 2 
    ? layer.feature.properties.NAME      // Countries use NAME
    : layer.feature.properties.NAME_1;   // Regions use NAME_1
  const sovName= layer.feature.properties.SOVEREIGNT; // for linking to Osmose

  umami.track('map-click');
  layer.setStyle({ color: '#ff7800' });
  layer.getPopup().setContent(`Loading ${name}… You need to disable your ad blocker for this to work`).update();

  try {
    if (currentMode === 'Osmose_issues') {
      if (level === 4) {
        const regionName = layer.feature.properties.NAME_1 || layer.feature.properties.NAME || '';
        // try: 1) explicit COUNTRY field in regions geojson, 2) parent country via region ISO code using countryNameMap, 3) SOVEREIGNT fallback, 4) original sovName
        let countryNameCandidate = layer.feature.properties.COUNTRY || null;
        if (!countryNameCandidate) {
          // iso might look like "US-CA" or "BR-MS"
          if (typeof iso === 'string' && /[-_]/.test(iso)) {
            const parentIso = iso.split(/[-_]/)[0].toUpperCase();
            countryNameCandidate = countryNameMap[parentIso] || null;
          }
        }
        if (!countryNameCandidate) countryNameCandidate = layer.feature.properties.SOVEREIGNT || sovName;
        await fetchOsmoseAndDownload(countryNameCandidate, regionName, layer);
      } else {
        await fetchOsmoseAndDownload(sovName, null, layer);
      }
    }
    else if (currentMode === 'GEM_powerplants') {
      await fetchGEMAndDownload(sovName);
    }
    else if (currentMode === 'MapYourSolar') {
      const usedSovName = getOverride(sovName, 'MapYourSolar') || sovName;
      await fetchSolarAndDownload(usedSovName);
    }
    else if (currentMode === 'Wikidata') {
      const usedSovName = getOverride(sovName, 'wikidata') || sovName;
      await fetchWikidataAndDownload(usedSovName);
    }
    else if (currentMode === 'GRW_wind') {
      await fetchWindAndDownload(sovName);
    }
    else if (currentMode === 'PPM') {
    await fetchPPMAndDownload(sovName);
    }
    else {
       let tpl = await fetchQuery(currentMode, level);
       const wikidata = layer.feature.properties.wikidata;
       const areaFilter = iso
          ? `["ISO3166-2"="${iso}"]`
          : `["wikidata"="${wikidata}"]`;
       tpl = tpl
          .replace(/\$\{iso\}/g, iso ?? 'NOMATCH')
          .replace(/\$\{wikidata\}/g, layer.feature.properties.wikidata ?? 'NOMATCH');
       sendToJosm(tpl, name); 
    }
  } catch (err) {
    layer.getPopup().setContent(`Error: Some Hint layers only work on a national level!`).update();
  }

setTimeout(() => {
  layer.setStyle({ color: '#3388ff' });

  let html;

  if (selectedEditor === 'id') {
    // Message for iD Editor users
    html = `
      <div class="popup-success">
        <p>🎉 <strong>Great!</strong> Now copy the url above. Then go to iD editor, and press on "Map Data". Then press on the three dots for "custom map data", and paste the url.</p>
      </div>
    `;
  } else {
    html = `
      <div class="popup-success">
        <p> Now go back to <a href="https://josm.openstreetmap.de/">JOSM</a> and check if your data is loading. Depending on the country, this may take <em>60 seconds or more</em>. The grid of some countries are too large to be mapped on a national level. However, you can zoom in and click on provinces or states.<br>⚠️If nothing happens:</p><ol><li> <strong>Please be aware that the servers used to download your data via Overpass are sometimes overloaded, so you may need to click your region again if you receive an error message in JOSM.</strong></li>
          <li>Make sure Remote Control is enabled in JOSM</li>
          <li>If it’s enabled but still not working, toggle it off and on again</li>
          <li>Note that hint layers do not work on regional layers. In this case, please load the data onto a national layer.</li>
          <li><a href="https://mapyourgrid.org/starter-kit/">Look into the Starter-Kit</a>
          <li> Sometimes the overpass server is overloaded, so try again (max 2 times)
        </ol>
      </div>
  `;
  }

  layer.getPopup()
       .setContent(html)
       .update();
}, 2000);
}

// initialize the UI immediately
initQueryUI().catch(console.error);

// ———————————————
// Fecthing functions for each
// ———————————————

// fetch the correct OverpassQL file on demand
async function fetchQuery(mode, adminLevel) {
  const rawUrl =
    `https://raw.githubusercontent.com/open-energy-transition/osm-grid-definition/` +
    `main/queries/${mode}/admin${adminLevel}.overpassql`;
  const r = await fetch(rawUrl);
  if (!r.ok) throw new Error(`Query file not found: ${mode}/admin${adminLevel}`);
  return r.text();
}

// Osmose API fetcher
async function fetchOsmoseAndDownload(countryNameOrSovName, regionName =  null, layer = null) {
  const sel = document.getElementById('osmoseIssue');
  if (!sel || !sel.value) { alert('Please select an issue type first.'); return; }
  const [item, cls] = sel.value.split(':');

  // prefer explicit override, else slugify
  const countryToken = getOverride(countryNameOrSovName, 'osmoseCountries') || slugifyForOsmose(countryNameOrSovName);

  let base = countryToken;
  if (regionName) {
    const regionToken = getOverride(regionName, 'osmoseRegions') || slugifyForOsmose(regionName);
    base = `${countryToken}_${regionToken}`;
  }
  if (!base.endsWith('*')) { base += '*';
  }

  //debug
  const debugMsg = `Osmose query: <code>${base}</code> (item=${item}, class=${cls})`;
  console.log(debugMsg);
  if (layer && layer.getPopup) {
    layer.getPopup().setContent(`<b>Querying Osmose…</b><br>${debugMsg}`).update();
  } else {
    // fallback: tiny console/info alert for debugging
    console.info(debugMsg);
  }

  const apiUrl = 
    `https://osmose.openstreetmap.fr/api/0.3/issues.geojson?` +
    `country=${encodeURIComponent(base)}` +
    `&item=${item}&class=${cls}&limit=5000` +
    `&useDevItem=all`;

  if (selectedEditor === 'id') {
      displayUrlForId(apiUrl);
      return;
  }

  const resp = await fetch(apiUrl);
  if (!resp.ok) throw new Error(`Osmose API ${resp.statusText}`);
  const geojson = await resp.json();

  if (!geojson.features || geojson.features.length === 0) {
    alert(`No issues found for "${sel.options[sel.selectedIndex].text}" in ${regionName ? (regionName + ', ' + countryNameOrSovName) : countryNameOrSovName}. Try another osmose issue type!`);
    return;
  }

  const layerName = `${(regionName ? `${countryNameOrSovName}_${regionName}` : countryNameOrSovName).replace(/\s+/g,'_')}-osmose-${item}-${cls}`;
  geoToJosm(apiUrl, layerName);
}

// GEM fetcher
async function fetchGEMAndDownload(sovName) {
  const folder = 'GEM-Global-Integrated-Power-February-2025-update-II';
  const fileName = sovName.replace(/\s+/g,'_') + '.geojson';
  const url = `https://raw.githubusercontent.com/open-energy-transition/osm-grid-definition/main/GEM/`
            + `${folder}/${fileName}`;

// First, check if the file actually exists to provide a clean error message
  const resp = await fetch(url, { method: 'HEAD' });
  if (!resp.ok) {
    return alert(`No GEM Plants file for ${sovName}.`);
  }

  if (selectedEditor === 'id') {
      displayUrlForId(url);
      return;
  }

  const layerName = `${sovName}-GEM`;
  sendUrlToJosm(url, layerName);
}

// TZ-solar fetcher
async function fetchSolarAndDownload(sovName) {
  const folder = 'TZ-Q32025';
  const fileName = sovName.replace(/\s+/g,'_') + '.geojson';
  const url = `https://raw.githubusercontent.com/open-energy-transition/osm-grid-definition/main/TZ-Solar/`
            + `${folder}/${fileName}`;

// First, check if the file actually exists to provide a clean error message
  const resp = await fetch(url, { method: 'HEAD' });
  if (!resp.ok) {
    return alert(`No Solar file for ${sovName}.`);
  }

  if (selectedEditor === 'id') {
      displayUrlForId(url);
      return;
  }

  const layerName = `${sovName}-solarTZ`;
  sendUrlToJosm(url, layerName);
}

// Wikidata fetcher
async function fetchWikidataAndDownload(sovName) {
  // grab the value of the wikidataType so "substations" or "powerplants" from the button selected (go up the script)
  const type = document.getElementById('wikidataType').value;

  // Here it fetches the folders from the github rpo:
  //   To add, put the value on the left (value is the value of the button wikidatatype), and on the right the name of the geojson folder from the github repo
   const foldertypes = {
    'All power-related infrastructure': '/output_by_qid_v2/geojson_by_country',
    'substations': '/substations/geojson_by_country',
    'powerplants': '/output_by_qid/Q159719_power_plant'
  };

  const folder = foldertypes[type];
  if (!folder) {
    return alert(`Unknown data type: ${type}`);
  }

  const fileName = sovName.replace(/\s+/g,'_') + '.geojson';
  const url = `https://raw.githubusercontent.com/open-energy-transition/osm-wikidata-toolset/main/`
            + `${folder}/${fileName}`;

// First, check if the file actually exists to provide a clean error message
  const resp = await fetch(url, { method: 'HEAD' });
  if (!resp.ok) {
    return alert(`No Wikidata ${type.replace(/_/g,' ')} file for ${sovName}.`);
  }

  if (selectedEditor === 'id') {
      displayUrlForId(url);
      return;
  }

  const layerName = `${sovName}-wikidata`;
  sendUrlToJosm(url, layerName);
}

async function fetchWindAndDownload(sovName) {
  const folder = '2024-q2v1';
  const fileName = sovName.replace(/\s+/g,'_') + '.geojson';
  const url = `https://raw.githubusercontent.com/open-energy-transition/osm-grid-definition/main/wind-renewables-watch/`
            + `${folder}/${fileName}`;

// First, check if the file actually exists to provide a clean error message
  const resp = await fetch(url, { method: 'HEAD' });
  if (!resp.ok) {
    return alert(`No Wind file for ${sovName}.`);
  }

  if (selectedEditor === 'id') {
      displayUrlForId(url);
      return;
  }

  const layerName = `${sovName}-windGRW`;
  sendUrlToJosm(url, layerName);
}

// PPM fetcher
async function fetchPPMAndDownload(sovName) {
  const type = document.getElementById('ppmType').value;

  const foldertypes = {
    'Rejected power plants': '/ppm_rejected_geojson_by_country'
  };

  const folder = foldertypes[type];
  if (!folder) {
    return alert(`Unknown data type: ${type}`);
  }

  const fileName = sovName.replace(/\s+/g, '_') + '.geojson';
  const url = `https://raw.githubusercontent.com/open-energy-transition/mapit-osm/main${folder}/${fileName}`;

  const resp = await fetch(url);
  if (!resp.ok) {
    return alert(`No PPM file found for ${sovName}.`);
  }

  if (selectedEditor === 'id') {
      displayUrlForId(url);
      return;
  }

  const layerName = `${sovName}-ppm`;
  sendUrlToJosm(url, layerName);
}

// ———————————————
// Send to JOSM
// ———————————————

// For direct opening of files in JOSM of hint layers
function sendUrlToJosm(dataUrl, layerName) {
  console.log("Setting JOSM layer name to:", layerName);
  // We must encode the dataUrl itself to pass it as a parameter to the JOSM url.
  const josmUrl = `http://localhost:8111/import?new_layer=true&layer_name=${encodeURIComponent(layerName)}&changeset_tags=hashtags=mapyourgrid&url=${encodeURIComponent(dataUrl)}`;

  console.log(`Sending data URL to JOSM: ${dataUrl}`);
  const iframe = document.createElement('iframe');
  iframe.style.display = 'none';
  iframe.src = josmUrl;
  document.body.appendChild(iframe);
  setTimeout(() => document.body.removeChild(iframe), 1000);
}

// For osmose (added format=geojson in url)
function geoToJosm(dataUrl, layerName) {
  console.log("Setting JOSM layer name to:", layerName);
  // We must encode the dataUrl itself to pass it as a parameter to the JOSM url.
  const geoUrl = `http://localhost:8111/import?new_layer=true&layer_name=${encodeURIComponent(layerName)}&changeset_tags=hashtags=mapyourgrid&url=${encodeURIComponent(dataUrl)}&format=geojson`;

  console.log(`Sending data URL to JOSM: ${dataUrl}`);
  const iframe = document.createElement('iframe');
  iframe.style.display = 'none';
  iframe.src = geoUrl;
  document.body.appendChild(iframe);
  setTimeout(() => document.body.removeChild(iframe), 1000);
}

// JOSM integration function
function sendToJosm(query, areaName) {
  // Encode only the query part
  const encodedQuery = encodeURIComponent(query);
  
  // Construct the Overpass URL by inserting the encoded query.
  const overpassUrl = "https://overpass-api.de/api/interpreter?data=" + encodedQuery;
  
  // Build the final JOSM URL by concatenating the strings manually.
  const josmUrl = `http://localhost:8111/import?new_layer=true&layer_name=${encodeURIComponent(areaName)}&changeset_tags=hashtags=mapyourgrid&url=${overpassUrl}`;
  
  // Load imagery too
  const mapboxImageryUrl = 'http://localhost:8111/imagery?id=Mapbox';
  const bingImageryUrl = 'http://localhost:8111/imagery?id=Bing';
  const esriImageryUrl = 'http://localhost:8111/imagery?id=EsriWorldImagery';
  
  // Keep a log to see the actual URL
  console.log("URL ", josmUrl);
  console.log("Sending Esri Imagery URL: ", esriImageryUrl);
  console.log("Sending Bing Imagery URL: ", bingImageryUrl);
  console.log("Sending Mapbox Imagery URL: ", mapboxImageryUrl);
  
  // Query part
  const iframe = document.createElement('iframe');
  iframe.style.display = 'none';
  iframe.src = josmUrl;
  document.body.appendChild(iframe);
  setTimeout(() => document.body.removeChild(iframe), 1000);
  
  // Imagery part
  const mapboxIframe = document.createElement('iframe');
  mapboxIframe.style.display = 'none';
  mapboxIframe.src = mapboxImageryUrl;
  document.body.appendChild(mapboxIframe);
  setTimeout(() => document.body.removeChild(mapboxIframe), 1000);

  const bingIframe = document.createElement('iframe');
  bingIframe.style.display = 'none';
  bingIframe.src = bingImageryUrl;
  document.body.appendChild(bingIframe);
  setTimeout(() => document.body.removeChild(bingIframe), 1000);

  const esriIframe = document.createElement('iframe');
  esriIframe.style.display = 'none';
  esriIframe.src = esriImageryUrl;
  document.body.appendChild(esriIframe);
  setTimeout(() => document.body.removeChild(esriIframe), 1000);
}

// ———————————————
// Base geo layers (countries and regions)
// ———————————————
// Load country GeoJSON and add interactivity
fetch('../data/countries.geojson')
  .then(response => response.json())
  .then(countries => {
    countriesLayer.addData(countries);

    countriesLayer.eachLayer(layer => {
      const iso  = layer.feature.properties.ISO_A2;
      const name = layer.feature.properties.NAME;

      // Colour when hovered  
      layer.on('mouseover', () => layer.setStyle({ fillColor: '#3388ff', fillOpacity: 0.3 }));
      layer.on('mouseout',  () => countriesLayer.resetStyle(layer));

      if (iso) countryNameMap[iso.toUpperCase()] = name;

      layer.bindPopup(`<b>${name}</b><br>Click to load in JOSM. You need to disable your ad blocker for this to work`);
      layer.on('click', () => {
        // large countries should be clicked at region level when zoomed in
        if (largeCountries.includes(iso) && map.getZoom() >= zoomThreshold) {
          layer
            .getPopup()
            .setContent(`<b>${name}</b><br>Please click on a specific region. You need to disable your ad blocker for this to work`)
            .update();
          return;
        }
        handleAreaClick(iso, 2, layer);
      });
    });
  })
  .catch(error => console.error('Countries GeoJSON error:', error));

// Load region GeoJSON and add interactivity (changed to load only when zoomed in as it's hevay). So regions won't load unless zoom level is right
let regionsLoaded = false;

async function loadRegionsOnce() {
  if (regionsLoaded) return;
  try {
    const resp = await fetch('../data/regionsv2.geojson');
    const regions = await resp.json();

    regionsLayer.addData(regions);
    regionsLayer.eachLayer(layer => {
          const iso3166_2 = layer.feature.properties.ISO_1;
          const name      = layer.feature.properties.NAME;

          // Colour when hovered  
          layer.on('mouseover', () => layer.setStyle({ fillColor: '#3388ff', fillOpacity: 0.3 }));
          layer.on('mouseout',  () => regionsLayer.resetStyle(layer));

          layer.bindPopup(`<b>${name}</b><br>Click to load in JOSM`);
          layer.on('click', () => 
            handleAreaClick(iso3166_2, 4, layer));
      });
      regionsLoaded = true;
      if (map.getZoom() >= zoomThreshold && !map.hasLayer(regionsLayer)) {
        map.addLayer(regionsLayer);
      }
  }   catch(error) {console.error('Regions GeoJSON error:', error);}
}
map.on('zoomend', function() {
  const currentZoom = map.getZoom();
  if (currentZoom >= zoomThreshold) {
    if (!regionsLoaded) loadRegionsOnce();
    if (!map.hasLayer(regionsLayer)) map.addLayer(regionsLayer);
  } else {
    if (map.hasLayer(regionsLayer)) map.removeLayer(regionsLayer);
  }
});

// ———————————————
// End
// ———————————————
</script>

[Discover Good First Lines to Get Started :fontawesome-solid-paper-plane:](good-first-lines.md){ .md-button .md-button--primary }

<!-- ENd-->
??? success "Map Legend for the recommended [MapCSS](starter-kit.md#3-add-visual-clarity-with-custom-map-styles) (Click Me)"
    <img 
      src="https://raw.githubusercontent.com/open-energy-transition/color-my-grid/refs/heads/main/legend/power-grid-legend.svg" 
      class="img-border image-caption" 
      alt="Power Grid Legend"
      style="display: block; margin-left: auto; margin-right: auto;"
      width="600">

??? success "JOSM Hotkeys (Click Me)"
    These JOSM hotkeys have proven helpful for grid mapping:

    | Hotkey         | Function Description                                      |
    |----------------|-----------------------------------------------------------|
    | S              | Select tool (for selecting objects)                       |
    | A              | Add tool (for drawing new nodes or ways)                  |
    | CTRL+C         | Copy selected objects                                     |
    | CTRL+V         | Paste objects from clipboard                              |
    | CTRL+F         | Search (Needs expert mode activated under `View`)         |
    | CTRL+Z         | Undo last action                                          |
    | CTRL+Y         | Redo last undone action                                   |
    | CTRL+SHIFT+C   | Copy coordinates of the selected point(s) to clipboard    |
    | CTRL+W         | Switch between activated Map Paint Style and Wireframe    |
    | CTRL+H         | History (opens history dialog for selected objects)       |
    | CTRL+U         | Update the existing geometries and tags loaded in JOSM    | 
    | Shift+V        | Validate (runs data validation on the current layer)      |
    | Tabulator      | Show/hide Sidebar and Edit toolbar                        |

## <div class="tools-header">Mapping Guidelines</div>
The following list provides the main good practices for mapping different power infrastructure in OpenStreetMap:

* [Power networks](https://wiki.openstreetmap.org/wiki/Power_networks)
* [Power networks/Guidelines](https://wiki.openstreetmap.org/wiki/Power_networks/Guidelines)
* [Power networks/Guidelines/Power lines](https://wiki.openstreetmap.org/wiki/Power_networks/Guidelines/Power_lines)
* [Power networks/Guidelines/Substations](https://wiki.openstreetmap.org/wiki/Power_networks/Guidelines/Substations)
* [Power generation/Guidelines/Hydropower](https://wiki.openstreetmap.org/wiki/Power_generation/Guidelines/Hydropower)
* [Power generation/Guidelines/Solar plants](https://wiki.openstreetmap.org/wiki/Power_generation/Guidelines/Solar_plants)
* [Power networks/Guidelines/Interconnector](https://wiki.openstreetmap.org/wiki/Power_networks/Guidelines/Interconnector)
* [Clarifying power=pole vs power=tower](https://community.openstreetmap.org/t/clarifying-power-pole-vs-power-tower/127382)

!!! Warning "Local Communities, Projects and good practices for mapping"
    **Before you start mapping, please find out about the mapping restrictions in the respective country. In some countries, the mapping of transmission lines is not permitted. Get in touch with local users by finding out about your [local communities](https://community.osm.be/) and [local projects](https://wiki.openstreetmap.org/wiki/Power_networks#Local_projects).  If you can't find a local community, please send us an [email](mailto:MapYourGrid@openenergytransition.org) and we will help you set up a local group.**

    **By following our [Good practices for mapping](./mapping-good-practices.md), we collectively protect the integrity of the OpenStreetMap platform, foster trust with communities, and unlock the power of open data for a more resilient and just energy future. Please do NOT copy any data from hint layer directly into your OpenStreetMap data layer. Every data point in your OpenStreetMap data layer must be manually set and [verified](https://wiki.openstreetmap.org/wiki/Verifiability). The metadata must also be verified against compatible licensed sources or by people on the ground. If you cannot verify the data using satellite images or any other compatible source, please do not add this information from hint layers. This may seem like a high burden at first, but it ensures the high quality of OpenStreetMap.** 

!!! Warning "Risk of Double Mapping"
     Please bear in mind that you have only downloaded transmission grid data for the country, state or province that you selected. This includes power plants, generators, substations, power towers and transmission lines. Other OpenStreetMap objects, such as streets, will not be visible. **Therefore, never use our tools to map objects other than those loaded via Overpass, as otherwise other mappers will have to clean up the duplicate data.**

    Some cross-border transmission lines will still be visible beyond the pink administrative boundaries. However, to edit these, you will need to load both countries. Never map beyond the pink administrative boundaries, as this will most likely result in infrastructure being mapped twice.


## Join the Chat <img src="/icons/discord.svg" alt="Discord" class="social-icon" style="width:1.2em; vertical-align:middle; margin-left:0.5ch;"> {.tools-header style="font-weight:700"}
We welcome everyone to join our [📍-MapYourGrid discord channel](https://discord.gg/a5znpdFWfD). Here you can ask questions, and interact with the community. For mapping specific questions and to participate in our free personalized training, please join our [📍-MapYourGrid-support-and-training](https://discord.gg/fBw7ARTUeR) channel. We share this server with [PyPSA-Earth](https://pypsa-earth.readthedocs.io/en/latest/), a global, open-source energy system model thats uses mainly OpenStreetMap's transmission grid data. Find us also on [Bluesky](https://bsky.app/profile/mapyourgrid.bsky.social).

## <div class="tools-header">Join the Community</div>
We welcome everyone to join our community calls and tutorials, to learn more about the mapping process and the initiative.
<iframe src="https://calendar.google.com/calendar/embed?src=mapyourgrid%40gmail.com&ctz=Europe%2FBerlin" style="border: 0" width="800" height="600" frameborder="0" scrolling="no"></iframe>

