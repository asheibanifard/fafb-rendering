# FAFB Browser Demo

This guide starts from the FAFB project directory and ends with a GPU-backed browser demo that can be shared with a supervisor.

## 1. Requirements

- Linux, macOS, or Windows
- Python 3.10+
- A modern browser with WebGL2 (Chrome, Edge, or Firefox)
- The FAFB project checkout

WebGPU is optional. The viewer tries WebGPU first and falls back to WebGL2. WebGL2 is enough for this demo.

## 2. Enter the project

```bash
cd /path/to/FAFB_project
```

Confirm the main files exist:

```bash
ls adaptive_fam_browser.html fafb_neuron_labels_source.raw fafb_neuron_ids.json
```

The included demo assets are:

- `adaptive_fam_browser.html`: browser viewer
- `fafb_neuron_labels_source.raw`: 128 x 128 x 257 uint8 source label grid
- `fafb_neuron_ids.json`: neuron IDs mapped to label values

## 3. Run locally

Serve the files over HTTP. Do not open the HTML with `file://`; browsers block binary fetches from local files.

For the isolated share bundle:

```bash
python -m http.server 8001 --directory demo-share
```

For the project root:

```bash
python -m http.server 8001
```

Open this address:

```text
http://localhost:8001/adaptive_fam_browser.html
```

A successful load shows `WebGL2 active; loaded 4,096,000 samples on the GPU.` If WebGPU is available, it will be used instead.

## 4. Use the viewer

- **View axis**: change the viewing direction.
- **Depth composite**: preserves depth ordering and is the recommended mode for inspecting neuron shape.
- **Maximum intensity**: shows the brightest selected label along each ray.
- **Neuron IDs**: click checkboxes to select one or several neurons. No Ctrl/Cmd key is required.
- **Opacity**: adjusts depth-composite opacity.
- **Initial volume dimension**: resamples the source label grid to 160^3, 192^3, 224^3, or 256^3 and rebuilds the GPU texture.
- **Mouse drag**: orbit the volume.
- **Mouse wheel**: zoom.

The initial selected neuron is the first ID in `fafb_neuron_ids.json`.

## 5. Run the real Adaptive-FAM `.mfab` demo

The separate MFA demo fetches and decodes an actual FAFB `.mfab` file in JavaScript:

```bash
python -m http.server 8001
```

Open:

```text
http://localhost:8001/mfab_demo.html
```

This demo uses `fafb_demo_block_10.mfab`, evaluates its degree-2 MFA spline, loads the segmentation labels, masks the reconstructed volume by selected neuron IDs, and renders the result with WebGL2. It is a single-block proof of concept; the full FAFB `.mfab` renderer set is not yet complete.

## 6. Share temporarily online

Keep the local HTTP server running. In a second terminal, run:

```bash
npx --yes localtunnel --port 8001
```

Localtunnel prints a public URL such as:

```text
https://example-name.loca.lt
```

Send your supervisor the URL with the page path:

```text
https://example-name.loca.lt/adaptive_fam_browser.html
```

The URL is temporary. The local server and tunnel terminals must remain open. Restarting the tunnel normally creates a new URL.

The recommended command uses `demo-share`, which exposes only the viewer and its required assets rather than the whole research repository.

## 7. Permanent hosting

For a stable URL, publish the contents of `demo-share/` with a static host such as GitHub Pages, Netlify, or a university web server.

The published directory must contain these files at the same level:

```text
index.html
adaptive_fam_browser.html
fafb_neuron_labels_source.raw
fafb_neuron_ids.json
```

Then open:

```text
https://your-host.example/adaptive_fam_browser.html
```

Do not rename the raw or JSON files unless you also update `volumeUrl` and `idsUrl` in the HTML.

## 8. Build the label assets again

If the FAFB source labels change, regenerate the original-grid browser asset from `fafb_multires/mip5/segments.npy`:

```bash
python - <<'PY'
import json
from pathlib import Path

import numpy as np

segments = np.load("fafb_multires/mip5/segments.npy", mmap_mode="r")
values, counts = np.unique(segments, return_counts=True)
ranked = sorted(
    [(int(count), int(value)) for value, count in zip(values, counts) if value != 0],
    reverse=True,
)[:20]
ids = [value for _, value in ranked]
labels = np.zeros(segments.shape, dtype=np.uint8)
flat = segments.reshape(-1)
out = labels.reshape(-1)
for index, neuron_id in enumerate(ids, 1):
    out[flat == neuron_id] = index
labels.tofile("fafb_neuron_labels_source.raw")
Path("fafb_neuron_ids.json").write_text(
    json.dumps({"ids": ids, "counts": [count for count, _ in ranked]}, indent=2)
)
PY

cp adaptive_fam_browser.html fafb_neuron_labels_source.raw fafb_neuron_ids.json demo-share/
```

The browser then resamples this source grid to the selected output dimension. This is intended for an interactive demo, not quantitative analysis.

## 9. Why 1024^3 is not a browser dropdown option

The original FAFB `mip2` intensity grid is 1024^3 and about 1 GiB as uint8. The corresponding segmentation labels are several GiB. GitHub rejects files over 100 MB, and uploading a 1024^3 texture directly would exceed practical browser memory limits.

A full 1024^3 online viewer therefore needs Adaptive-FAM LOD blocks, chunked streaming, and a block manifest. The current 256^3 control resamples from the available 128 x 128 x 257 source label grid; it does not invent new source information.

## 10. Troubleshooting

### `WebGPU setup failed`

This is not fatal. The viewer should continue with WebGL2. Confirm the status says `WebGL2 active`.

### `No WebGL2 context`

Use a current Chrome, Edge, or Firefox and enable hardware acceleration. Remote desktop sessions and virtual browsers may disable GPU contexts.

### A rectangular gray artifact appears

Reload the page after confirming you are using the latest `adaptive_fam_browser.html`. The WebGL2 shader discards rays that miss the volume; older copies may show clamped texture edges as a rectangle.

### The supervisor sees a blank page

Confirm the server is running, use the full `/adaptive_fam_browser.html` path, and verify that the raw and JSON files are hosted beside the HTML file.

### The public tunnel stops working

The localtunnel process or local HTTP server has stopped. Restart both commands and send the new URL.

## 11. Scientific limitation

The browser demo uses a compact 160^3 nearest-neighbor label volume and GPU ray marching. It is appropriate for visual demonstration and neuron selection. Use the original FAFB data and project reconstruction pipeline for measurements or publication figures.
