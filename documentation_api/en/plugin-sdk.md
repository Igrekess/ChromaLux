# Native Plug-in SDK

## Status and Compatibility

The public binary contract is the C99 header
`chromalux_plugin.h`. In 0.6.5:

```c
#define CLS_PLUGIN_API_VERSION 9u
```

The host calls `cls_plugin_init(9, plugin_dir)`, then requires
`plugin->api_version == 9`. The comparison is strict: a binary compiled for
another version is rejected. A plug-in must check `host_api_version` before
performing any allocation and return `NULL` if it cannot support that version.

There is not yet a forward binary-compatibility layer. A change to the layout
of a structure therefore requires recompilation.

## Entry Points and Lifecycle

Each library must provide at least these two entry points:

```c
CLS_EXPORT CLSPlugin* cls_plugin_init(uint32_t host_api_version,
                                      const char* plugin_dir);
CLS_EXPORT void cls_plugin_shutdown(CLSPlugin* plugin);
```

The main 0.6.5 loader does not immediately reject a library that omits
`cls_plugin_shutdown`, but such a library does not comply with the header
contract and is rejected by `cls_plugin_test`.

The lifecycle observed by the host is:

1. load the library and call `cls_plugin_init`;
2. read `CLSPlugin` and its `CLSNodeDescriptor` entries;
3. create an instance through `descriptor.vtable.create(plugin_ctx)`;
4. notify numeric default values through `on_param_changed`;
5. call `process()`;
6. call `destroy()` on every instance;
7. call `cls_plugin_shutdown(plugin)`;
8. unload the library.

### Memory Ownership

- Static strings and descriptors belong to the plug-in. The host does not
  free them.
- `CLSNode*` is opaque and belongs to the plug-in between `create` and
  `destroy`.
- Textures, services, and request structures are borrowed from the host.
- A plug-in must not retain any pointer obtained from `CLSExecRequest` after
  `process()` returns.

### Threads

- `create`, `destroy`, and `on_param_changed` are called on the main thread.
- `process` may be called on any thread, but never concurrently for the same
  `CLSNode` instance.
- `process` must not perform long blocking CPU work or wait unnecessarily for
  the GPU.

## Result Codes

| Code | Meaning |
|---|---|
| `CLS_OK` | success |
| `CLS_ERR_INVALID_ARG` | invalid argument, port, or capacity |
| `CLS_ERR_OUT_OF_MEMORY` | allocation failed |
| `CLS_ERR_GPU` | GPU creation, encoding, or execution failed |
| `CLS_ERR_UNSUPPORTED` | recognized operation, but unavailable |
| `CLS_ERR_INTERNAL` | internal error without a more specific category |

## Ports

| Type | Current transport |
|---|---|
| `CLS_PORT_IMAGE` | RGBA32F texture |
| `CLS_PORT_IMAGE_HALF` | RGBA16Float texture |
| `CLS_PORT_MASK` | R32F texture |
| `CLS_PORT_IMAGE_R32F` | R32F texture |
| `CLS_PORT_IMAGE_R16U` | R16Uint texture, notably for Bayer data |
| `CLS_PORT_METADATA` | pointer to `CLSMetadataBlob` in the `texture` field |
| `CLS_PORT_CHANNEL_SCS` | declared in the ABI, not routed by the 0.6.5 Scheduler |
| `CLS_PORT_SCALAR` | declared in the ABI, not routed by the 0.6.5 Scheduler |

The graph accepts cross-connections between `IMAGE` and `IMAGE_HALF`. The
actual format of a port may also be overridden by the host in some pipelines;
the plug-in must read `CLSPortData.type`.

## Parameters

Numeric values are placed in `CLSExecRequest.param_values`:

- `FLOAT`, `INT`, and `BOOL` are active;
- `ENUM` transports the selected index as a `double`;
- `COLOR` is declared and has `default_color[4]`, but does not yet have a
  complete and uniform path through all consumers;
- `CURVE` and `BAND_PICKER` are partial/reserved surfaces.

`FILE_PATH`, `TEXT`, `CURVE`, and `BAND_PICKER` transport their value through
`setStringParam` and then `param_strings`. For the latter two, the payload is
a JSON serialization or a consumer-specific list; their persistence and
consumers remain incomplete in 0.6.5. These pointers belong to the host and
are valid only during `process()`. The ABI has no `default_string`. In the
current implementation, an update through `setStringParam` does not trigger
`on_param_changed`: the plug-in reads the value from the processing request.

`min_float` and `max_float` describe the interface, but do not replace
plug-in-side validation.

## Metadata

`cls_metadata.h` exposes
`CLSMetadataBlob`. The current transport is:

- UTF-8 JSON;
- default capacity of 8,192 bytes;
- data that is not necessarily terminated by `\0`;
- `size` must remain less than or equal to `capacity`;
- the plug-in writes the bytes and updates `size`, without freeing the
  buffer.

The blob lifetime is that of the output managed by the Scheduler.

`cls_metadata_json.h` provides a
header-only C++17 tokenizer. It is a source API, not an ABI or a general JSON
parser. In particular, it does not support `\uXXXX`, comments, trailing
commas, `NaN`, `Infinity`, or a BOM; its numeric conversion uses an internal
buffer limited to 63 characters.

## Tiled Execution, LOD, and ROI

`CLSExecRequest` describes the requested tile through `tile_x`, `tile_y`,
`tile_w`, `tile_h`, the full dimensions, and `lod_level`.

Each `CLSPortData` provides:

- the texture and its format;
- the width and height of the supplied buffer;
- `origin_x` and `origin_y`, expressed in LOD space;
- `full_w` and `full_h`, also expressed in LOD space.

`full_w == 0` denotes a legacy full-frame call; the plug-in must then
interpret the origin as `(0, 0)` and the full dimensions as those of the
port.

A node that changes geometry must use its output dimensions rather than
assuming that all inputs and outputs have the same size.

`modify_roi_in` converts a requested output tile into an input region. In ABI
v9, the same ROI is applied to every image input: there is not yet a distinct
ROI per port.

`halo_radius` expresses the neighborhood required by a filter. The 0.6.5
Scheduler computes the maximum useful halo for the graph and expands tiles to
avoid seams; an older header comment that always mentions full-frame
processing no longer describes the whole current implementation.

## Metal and the Shared Command Buffer

`metal_device` and `metal_command_queue` are borrowed. A plug-in must not
create its own queue.

A descriptor may advertise:

```c
execution_caps = CLS_NODE_CAP_SHARED_METAL_CB;
```

When the host also enables this path, `metal_execution` supplies a borrowed
`command_buffer`. The plug-in may create and close encoders on it, but it
must not:

- retain the buffer;
- call `commit` or `waitUntilCompleted`;
- add a completion handler.

The Scheduler owns submission. The plug-in must retain a working path when
`metal_execution == NULL`, because the shared buffer depends on the
`CLS_SHARED_TILE_CB` runtime gate.

## Host Services

`CLSHostServices_v1.version` is `1`. Always check the table and each callback
before calling them.

- `gpu_read_back` performs a synchronous read. The 0.6.5 Scheduler
  implementation supports RGBA32F textures only.
- `run_batch_subgraph` is reserved but currently returns
  `CLS_ERR_UNSUPPORTED` and sets the output to `NULL`.

## Building Within the Repository

The internal CMake helper is used as follows:

```cmake
cls_add_plugin(example_plugin
    SOURCES plugin.mm
    MANIFEST manifest.json
    KERNEL kernel.metal
    LIBRARY example_plugin
)
```

It builds the dylib, copies the manifest, and optionally compiles a
`.metallib`. However, `cls_add_plugin` depends on the monorepo structure: it
is not yet an exportable CMake package for an external project.

Reference examples:

- `src/plugins/test/identity/`;
- `src/plugins/test/constant_color/`;
- `cmake/ChromaluxPlugin.cmake`.

Structural validation:

```sh
cmake --build build-work --target cls_plugin_test test_identity
build-work/bin/cls_plugin_test build-work/plugins/test_identity
```

The expected result is `14/14 PASS`. This tool does not process pixels; its
limitations are detailed in the [developer tools](developer-cli.md).
