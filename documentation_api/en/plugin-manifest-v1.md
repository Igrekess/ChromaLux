# Plug-in Manifest v1

The `manifest.json` file describes the package that the loader must open. Its
schema is `cls.plugin.manifest.v1`.

## Recommended Example

```json
{
  "schema": "cls.plugin.manifest.v1",
  "id": "org.chromalux.partner.example",
  "version": "1.0.0",
  "api_version": 9,
  "name": "Example",
  "vendor": "Vendor",
  "license": "Proprietary",
  "library": "libexample.dylib",
  "nodes": ["example.pass"],
  "requires_gpu": true,
  "requires_metal": true,
  "min_host_version": "0.6.5"
}
```

## Fields and Current Validation

| Field | Required | Handling in 0.6.5 |
|---|---|---|
| `schema` | yes | must be exactly `cls.plugin.manifest.v1` |
| `id` | yes | string; the reverse-DNS convention is not validated |
| `version` | yes | string; semver syntax is not validated |
| `api_version` | yes | parsed integer, but not used as a gate by the loader |
| `name` | yes | string |
| `vendor` | yes | string |
| `license` | yes | string |
| `library` | yes | relative path to the library |
| `nodes` | yes | non-empty array of strings |
| `min_host_version` | yes | parsed string, not yet applied during loading |
| `requires_gpu` | no | Boolean, defaults to `false`, not yet applied |
| `requires_metal` | no | Boolean, defaults to `false`, not yet applied |

Unknown fields are ignored, including the legacy `description` and `params`
fields. Parameters exposed to the host come exclusively from the
`CLSNodeDescriptor` entries returned by the binary.

## Declared Version and Actual ABI

`manifest.api_version` and `CLSPlugin::api_version` are two different pieces
of data:

- the parser requires the JSON field to exist;
- the loader does not compare its value with the ABI;
- the loader does, however, strictly require the binary to return
  `CLSPlugin::api_version == CLS_PLUGIN_API_VERSION`, which is `9`.

The repository still contains historical manifests with values `1`, `3`,
`5`, `6`, `7`, and `9`, while their rebuilt binaries expose ABI 9. A new
manifest should declare `9` to avoid this ambiguity, even though this field
is not yet a gate.

## Checks Performed During Loading

The loader:

- parses the manifest and checks its schema;
- requires the library to exist within the plug-in directory;
- rejects an absolute path, `..` traversal, or an escaping symbolic link;
- loads with `RTLD_NOW | RTLD_LOCAL`;
- looks up `cls_plugin_init`;
- requires a non-null result and a binary ABI exactly equal to 9.

The loader does not yet check:

- equality between `manifest.api_version` and the binary ABI;
- `min_host_version`;
- the `requires_gpu` and `requires_metal` flags;
- consistency of identifiers and versions between the JSON and `CLSPlugin`;
- equality of `nodes[]` and the descriptors.

`cls_plugin_test` additionally checks `nodes[]`, the vtable, and the
`create`/`destroy` cycle.

## Directory Layout

A minimal package is:

```text
example/
├── manifest.json
├── libexample.dylib
└── kernel.metallib
```

The `.metallib` is optional. For a user plug-in, the `example/` folder resides
under:

```text
<QStandardPaths::AppDataLocation>/Plugins/
```

## Signing and Trust on macOS

Before any `dlopen`, a user plug-in:

- must have a valid signature;
- must be signed with the same Apple Team ID as ChromaLux;
- must be explicitly approved by the user;
- is then allowed by the exact SHA-256 fingerprint of the library;
- is rejected when `CLS_SAFE_MODE=1`.

An update that changes the bytes changes the fingerprint and requires new
approval. This model is suitable for internal, private, or partner plug-ins.
It is not yet a distribution policy for an independent third-party
marketplace.

Sources: `PluginManifest.cpp`,
`PluginLoader.cpp`, and
`Application.cpp`.
