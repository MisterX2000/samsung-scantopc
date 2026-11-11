# samsung-scantopc — Docker Container

Runs the [samsung-scantopc](https://github.com/kleest/samsung-scantopc) server
inside a container for easier deployment on a server. All configuration is done 
through **environment variables**; no config file needs to be bind-mounted.

For advanced scan options that include image filter functions, an optional
Python file can be mounted into the container (see [Custom Options File](#custom-options-file)).

This repository is a maintained fork of [finnboeger/samsung-scantopc](https://github.com/finnboeger/samsung-scantopc)
providing automated image publishing to GHCR, automatic ADF/Flatbed feeder detection,
graceful offline reconnect handling, and modern Python tooling.

Pre-built Docker images are published automatically to:
`ghcr.io/misterx2000/samsung-scantopc:latest`

> [!IMPORTANT]
> This fork has been specifically developed and tested with the Samsung SCX-472x series (specifically the SCX-4726FN). No other printer models or series have been tested. Other Samsung network scanners supported by the Samsung Unified Linux Driver (SULDR) might work, but are untested in this fork.

---

## Changes from Upstream (finnboeger/samsung-scantopc)

- **Smart Feeder / Flatbed Auto-Detection**:
  - Scanning presets can set `"doc_source": "Auto"` (or omit `doc_source`).
  - Probes the feeder using `ADF Simplex`; if documents are loaded, scans from the ADF feeder, otherwise automatically falls back to `Flatbed`.
- **SANE Session Reset**:
  - Fixes a critical SANE session lock where a feeder probe would leave the SANE handle in an aborted state, previously causing flatbed scans to time out.
- **Graceful Offline Reconnection**:
  - When the printer is switched off or in deep sleep, connection timeouts are caught cleanly and logged as a single status line instead of spamming 15-line stack traces every 10 seconds.
- **Automated GitHub Container Registry (GHCR) CI**:
  - GitHub Actions automatically builds and publishes `linux/amd64` images to `ghcr.io/misterx2000/samsung-scantopc` with semantic versioning (`v*`), `latest`, and commit SHA tags.
- **Code Quality & Packaging**:
  - Added `pyproject.toml` with PEP 621 packaging metadata, dependency definitions, and `ruff` linting/formatting configuration.
  - Formatted and linted entire codebase with `ruff`.
  - Added `uv` support for high-performance dependency management and linting.

---

## Quick Start

### Using the Pre-built GHCR Image (Recommended)

```yaml
services:
  samsung-scantopc:
    image: ghcr.io/misterx2000/samsung-scantopc:latest
    container_name: samsung-scantopc
    network_mode: host
    cap_add:
      - NET_ADMIN
      - NET_BROADCAST
      - NET_RAW
    volumes:
      - ./scans:/scans
      - /etc/localtime:/etc/localtime:ro
    environment:
      TZ: "Europe/Berlin"
      SCANNER_SANE_NAME: "smfp:net;192.168.1.100"
      SERVER_NAME: "MyServer"
      OWNER_UID: 1000
      OWNER: "scanuser"
      SCAN_OUTPUT_DIR: "/scans"
      SCAN_FILENAME_TEMPLATE: "SAMSUNG_SCAN_$${datetime}__$${uid}"
      SCAN_OPTIONS: >-
        [
          {"name": "Color PDF",     "color": "COLOR_TRUE", "resolution": "DPI_300", "format": "FORMAT_PDF",  "size": "SIZE_A4", "doc_source": "Auto"},
          {"name": "Color PDF 600", "color": "COLOR_TRUE", "resolution": "DPI_600", "format": "FORMAT_PDF",  "size": "SIZE_A4", "doc_source": "Auto"},
          {"name": "Color JPG",     "color": "COLOR_TRUE", "resolution": "DPI_300", "format": "FORMAT_JPEG", "size": "SIZE_A4", "doc_source": "Auto"},
          {"name": "Gray PDF",      "color": "COLOR_GRAY", "resolution": "DPI_300", "format": "FORMAT_PDF",  "size": "SIZE_A4", "doc_source": "Auto"}
        ]
    restart: unless-stopped
```

Run:
```bash
docker compose up -d
```

### Building Locally

```bash
docker compose build && docker compose up -d
```

---

## Environment Variables

### Core Feature Flags

| Variable | Default | Description |
|---|---|---|
| `ENABLED_SERVER` | `true` | Set to `false` to start the container without registering with the scanner. |
| `MODIFIED_SANE` | `false` | Enable the TCP/UDP proxy mode required for multipage scanning on some models (e.g. CLX-3305W). |
| `PROXY_DEBUGLEVEL` | `1` | Verbosity of the proxy processes. `0` = silent, `1` = light, `2` = moderate, `3` = every packet. Only relevant when `MODIFIED_SANE=true`. |
| `SCANNER_CACHING` | `true` | Keep the SANE connection open between jobs. Set to `false` if you run more than one server against the same scanner. |

### Scanner / Server Identity

| Variable | Default | Description |
|---|---|---|
| `SCANNER_SANE_NAME` | *(auto-detect)* | Full SANE device string, e.g. `smfp:net;192.168.1.100` or `smfp:SAMSUNG SCX-472x Series on 192.168.1.100`. When omitted the server scans for the first Samsung device every 30 s until one appears. Find the correct value with `scanimage -L` on the host. |
| `SERVER_NAME` | *(hostname)* | Name shown in the scanner's "Scan to PC" destination list. |

### File Ownership & Timezone

| Variable | Default | Description |
|---|---|---|
| `TZ` | `UTC` | Timezone for file timestamps (e.g. `Europe/Berlin`). Mount `/etc/localtime:/etc/localtime:ro` as well. |
| `OWNER_UID` | `1000` | UID of the user that will own saved scan files. |
| `OWNER` | `scanuser` | Username corresponding to `OWNER_UID`. |

### Output Path

The final output path is `SCAN_OUTPUT_DIR/SCAN_FILENAME_TEMPLATE.<ext>`.

| Variable | Default | Description |
|---|---|---|
| `SCAN_OUTPUT_DIR` | `/scans` | Directory inside the container where scans are written. This should match the volume mount target. |
| `SCAN_FILENAME_TEMPLATE` | `SCAN_${date}__${uid}` | Filename template (without extension). Supported substitutions: `${date}` → `YYYY-mm-dd`, `${datetime}` -> `YYYY-mm-dd_HH-MM-SS`, `${uid}` → zero-padded sequence number, `${homedir}` → home directory of `OWNER`. When providing a value via docker-compose.yml remember to escape the `$` by using `$$`. |

### Logging

| Variable | Default | Description |
|---|---|---|
| `LOG_NAME` | `/var/log/samsungScannerServer.log` | Absolute path to the rotating log file. Set to an empty string (`LOG_NAME=`) to disable file logging. |
| `LOG_MAXBYTES` | `100000` | Maximum size of the log file before it is rotated. |
| `LOG_BACKUPCOUNT` | `1` | Number of rotated log file backups to keep. |

### Scan Options (JSON)

When neither an [options file](#custom-options-file) nor `SCAN_OPTIONS` is set,
built-in presets are registered with the scanner (grey/colour × PDF/JPEG at various DPIs).

To override, set `SCAN_OPTIONS` to a JSON array. Each object may contain:

| Key | Default | Accepted values |
|---|---|---|
| `name` | `Unnamed` | Any string — shown on the scanner LCD display (keep under ~16 chars) |
| `color` | `COLOR_GRAY` | `COLOR_GRAY`, `COLOR_TRUE`, `COLOR_MONO` |
| `resolution` | `DPI_300` | `DPI_75`, `DPI_150`, `DPI_200`, `DPI_300`, `DPI_600` |
| `format` | `FORMAT_M_PDF` | `FORMAT_M_PDF`, `FORMAT_S_PDF`, `FORMAT_PDF`, `FORMAT_JPEG`, `FORMAT_M_TIFF`, `FORMAT_S_TIFF` |
| `size` | `SIZE_A4` | `SIZE_A4`, `SIZE_A5`, `SIZE_LETTER`, `SIZE_LEGAL`, `SIZE_B5_JIS`, `SIZE_EXECUTIVE`, `SIZE_FOLIO` |
| `doc_source` | `Auto` | `Auto`, `Flatbed`, `ADF Simplex` |
| `output` | *(derived)* | Override the output path template for this preset only |

> [!NOTE]
> `FORMAT_PDF` should be used instead of `FORMAT_M_PDF` / `FORMAT_S_PDF` on SCX-472x and similar models. Multi-page feeder scans are still automatically concatenated into a single multi-page PDF.

Filter functions **cannot** be expressed in JSON. Use the [custom options file](#custom-options-file) for those.

### Advanced: Conversion Tables

These rarely need changing. Override only if your device reports unusual SANE mode or size strings. Both accept a JSON object.

| Variable | Description |
|---|---|
| `MODES2SANE` | Maps scanner colour-mode tokens to SANE mode strings. Default: `{"COLOR_MONO": "Black and White - Line Art", "COLOR_GRAY": "Grayscale - 256 Levels", "COLOR_TRUE": "Color - 16 Million Colors"}` |
| `SIZE2SANE` | Maps scanner size tokens to SANE page-format strings. When unset, the server queries the device's `CAP.XML` and auto-configures this mapping at startup. Example override: `{"SIZE_A4": "A4", "SIZE_LETTER": "Letter - 8.5\"x11\""}` |

---

## Custom Options File

For presets that use image filter functions, create a Python file on the host and bind-mount it into the container. The server will `exec` this file in place of `SCAN_OPTIONS`.

### Mount path

The default path inside the container is `/etc/samsungScannerServer.options.py`. Override with the `OPTIONS_FILE` environment variable.

```yaml
volumes:
  - ./scans:/scans
  - ./my-options.py:/etc/samsungScannerServer.options.py:ro
```

### File format

The file must define an `OPTIONS` list. It has access to `OUTPUT_PREFIX` and anything already imported in the main script (`Image`, `os`, etc.). Any additional imports must be done inside the file itself.

```python
# my-options.py
from PIL import ImageOps


def contrast_filter(im):
    """Auto-contrast, clipping the brightest/darkest 10 %."""
    return ImageOps.autocontrast(im, 10)


OPTIONS = [
    {
        "name": "Gray PDF (auto-contrast)",
        "color": "COLOR_GRAY",
        "resolution": "DPI_300",
        "format": "FORMAT_M_PDF",
        "size": "SIZE_A4",
        "doc_source": "Auto",
        "output": OUTPUT_PREFIX,  # OUTPUT_PREFIX is available from the parent script
        "filters": [contrast_filter],
    },
    {
        "name": "Color PDF",
        "color": "COLOR_TRUE",
        "resolution": "DPI_300",
        "format": "FORMAT_PDF",
        "size": "SIZE_A4",
        "doc_source": "Auto",
        "output": OUTPUT_PREFIX,
        "filters": [],
    },
]
```

| Variable | Default | Description |
|---|---|---|
| `OPTIONS_FILE` | `/etc/samsungScannerServer.options.py` | Path to the options Python file. If the file does not exist the server falls back to `SCAN_OPTIONS` or the built-in defaults. |

---

## Volumes

| Mount point | Description |
|---|---|
| `/scans` | Output directory for all scanned files. Mount a host directory here, e.g. `./scans:/scans`. |
| `/etc/localtime` | (Optional) Host timezone mount (`/etc/localtime:/etc/localtime:ro`) to ensure correct timestamping matching the host. |
| `/etc/samsungScannerServer.options.py` | (Optional) Custom Python options file for advanced presets and image filters. |

---

## License

This project is licensed under the [GNU General Public License v3.0](LICENSE).

```
Copyright (C) 2026 Mister X
Copyright (C) 2022-2024 Finn Boeger
Copyright (C) 2022-2023 Steffen Klee
Copyright (C) 2012-2013 angelnu & Totally King
```
