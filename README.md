# Casa Duro — Modular Configurator App

A **single, self-contained interactive app** for designing modular buildings in 3D,
generating a Bill of Materials, and running structural calculations via **CalcTree**.

It is built to be **embedded into a platform you build later**: it exposes a clean
JavaScript API (`window.CasaDuroApp`) and a `postMessage` bridge, so a host page can
drive it and read its data without touching the internals. This repo is *just the app* —
not the platform.

## Run it

Open `index.html` in any modern browser. No build step, no server. Three.js and the
icon font load from CDN.

- **Tools:** Module, Window, Door, Ext/Int Panel, Partition (keys `m w d e i p`, `s` = select)
- **Navigate:** orbit = right-drag / alt-drag · pan = shift-drag · zoom = scroll · `Shift+Z` = fit
- Place components on the ground, edit them in **Properties**, see live **BOM**, and press
  **Run Structural** for a CalcTree analysis.

## Embedding API (`window.CasaDuroApp`)

```js
const app = window.CasaDuroApp;            // available after the 'ready' event

app.getCatalog();                          // materials, window/door/panel types
app.addObject('module', { x:0, z:0, w:3, l:6, h:3 });
app.addObject('window', { x:1.5, z:0, libItem:'win-m' });
const model = app.getModel();              // { objects:[...] }  (serialisable)
app.loadModel(model);                      // restore a saved design
app.clear();

app.getBOM();                              // [{category,description,width,height,depth,material,qty}]
app.exportBOM();                           // downloads CSV

app.getStructuralInputs();                 // loads, spans, storeys, footprint
await app.runStructuralAnalysis();         // -> { inputs, result, at }
app.getLastAnalysis();

// events
const off = app.on('analysis:complete', d => console.log(d.result));
app.on('*', ({event, detail}) => {/* every event */});
off();                                     // unsubscribe
```

### Events emitted
`ready`, `tool:changed`, `object:added`, `object:changed`, `object:removed`,
`selection:changed`, `model:loaded`, `bom:exported`, `analysis:complete`.

### Cross-origin embedding (iframe)

When the app runs inside an iframe, every event is forwarded to the parent:

```js
// host page
iframe.contentWindow.postMessage(
  { target:'casaduro', method:'addObject', args:['module', { x:0, z:0 }], id:1 }, '*');

window.addEventListener('message', e => {
  if (e.data?.source !== 'casaduro') return;
  if (e.data.replyTo) { /* command result for id */ }
  else { /* live event: e.data.event, e.data.detail */ }
});
```

Any `CasaDuroApp` method can be invoked this way; the result is posted back with the
matching `replyTo` id.

## CalcTree wiring

Structural calcs run through CalcTree when configured; otherwise the app uses a
**transparent local estimate** (clearly labelled "Local estimate" in the UI) so it stays
fully functional out of the box.

Edit the `CALCTREE` config block near the top of the script in `index.html`, or set it at
runtime:

```js
app.configureCalcTree({
  enabled: true,
  endpoint: 'https://api.calctree.com/v1/calculations/run',
  apiKey: 'YOUR_KEY',
  modelId: 'YOUR_PUBLISHED_MODEL_ID',
  // map app inputs -> your CalcTree input parameter names
  inputMap: { maxAxialLoad_kN:'axial_load', maxSpan_m:'span', /* ... */ },
  // map CalcTree outputs -> app result fields
  outputMap: { utilisation:'utilisation', pass:'pass', governing:'governing_check' },
});
```

The app computes engineering inputs from the 3D model (dead/live loads, storeys from
stacked modules, max span, axial load per column), sends them to CalcTree, and renders the
returned utilisation / pass-fail / governing check. Adjust `inputMap`/`outputMap` to match
your published CalcTree sheet's parameter names.

> The local estimate is a rough placeholder for demo use only — **not** a substitute for a
> stamped structural calculation. Wire CalcTree (and have an EOR review) for real results.
