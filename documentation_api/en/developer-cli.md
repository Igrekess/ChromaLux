# Developer command-line tools

The tools below are produced in `<build>/bin`. They are not copied into the
user DMG. Many require macOS, Metal, built plug-ins, or local fixtures; their
outputs are not all stable APIs.

## `cls_plugin_test`

Usage:

```sh
build-work/bin/cls_plugin_test build-work/plugins/test_identity
```

The program performs fourteen checks:

1. `manifest.json` is present;
2. the JSON is readable;
3. the schema is recognized;
4. required fields are present;
5. the library is present;
6. `dlopen` succeeds;
7. the `cls_plugin_init` symbol is found;
8. initialization returns a non-null value;
9. the binary ABI is exactly equal to 9;
10. at least one descriptor is present;
11. `type_id`, `display_name`, and the vtable are complete;
12. `nodes[]` matches the descriptors;
13. the `create`/`destroy` cycle succeeds;
14. `shutdown` is called, followed by unloading.

Expected result:

```text
14/14 PASS
```

Exit codes: `0` for success, `1` for a validation failure, and `2` for a usage
error.

Important limitations:

- it does not call `process()`;
- it does not check pixels, shaders, performance, or budgets;
- it does not verify user signatures or trust;
- it does not compare `manifest.api_version`;
- the tool nevertheless executes `dlopen`, `init`, and `create`: never use it
  on an untrusted binary.

## Camera profiles

### `cls_camera_profile_diagnostic`

Diagnoses a local profile and, optionally, whether it matches a camera
identity:

- text or JSON output with `--format text|json`;
- read-only by default;
- `--import-local` is the only mutating operation;
- a profile imported this way remains `Experimental`, with `Unknown` rights,
  and is not selected automatically.

The exit codes documented by the tool are `0`, `2`, `3`, `4`, `5`, and `64`.
See `--help` for their meaning in the selected mode.

### `cls_dcp_to_scsp`

Converts a complete dual-illuminant DCP to SCSP v3:

```sh
build-work/bin/cls_dcp_to_scsp \
  --dcp camera.dcp \
  --out camera.scsp
```

The v3 path rejects a simple profile that does not provide the complete
dual-illuminant contract. `--legacy-v1-matrix` explicitly requests the
production of a matrix-based SCSP v1 from this type of source. The command
writes to the path provided through `--out`.

## Sessions and catalog

`cls_session` provides, among others:

```text
create, scan, list, search,
set-rating, set-keywords,
add-color-tag, remove-color-tag,
set-color-label, ingest
```

`set-color-label` is retained for compatibility but deprecated. `ingest` has
source, destination, backup, verification, and MHL options; in 0.6.5, this
subcommand is implemented but omitted from the general help.

## Application diagnostic options

The main binary accepts, among others:

```sh
build-work/bin/ChromaLux.app/Contents/MacOS/ChromaLux --version
build-work/bin/ChromaLux.app/Contents/MacOS/ChromaLux --safe-mode
build-work/bin/ChromaLux.app/Contents/MacOS/ChromaLux --plugin-smoke
```

- `--safe-mode` disables user plug-ins.
- `--plugin-smoke` loads and inventories visible plug-ins. A development build
  may also contain test plug-ins; the application packaged in the 0.6.5 DMG is
  validated with `loaded=64 manifests=64`.

## Internal and QA tools

These binaries are internal harnesses with no compatibility guarantee for
their interface:

- DAG and performance: `cls_gpu_bench`, `cls_dag_run`, `cls_tile_bench`;
- RAW and rendering: `cls_raw_decode_smoke`, `cls_thumb_jpeg_smoke`,
  `cls_amaze_smoke`, `cls_halo_smoke`, `cls_xtrans_smoke`,
  `cls_color_smoke`, `cls_develop`;
- measurements: `cls_noise_probe`, `cls_demosaic_bench`,
  `cls_recover_bench`, `cls_tonemap_bench`, `cls_localcontrast_bench`;
- grain: `clsfg-render`.

Their availability depends on CMake options and the platform. Use
`cmake --build build-work --target help` to list the targets in the current
build.
