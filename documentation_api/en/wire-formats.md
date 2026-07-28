# Graph and node JSON formats

These formats are used between the C++ DAG and the Graph Editor. They are
persistent, but remain internal formats without a published JSON Schema or a
third-party interoperability guarantee in 0.6.5.

## `.cls.graph` graph

The current implementation:

- writes `cls.graph.v2`;
- reads `cls.graph.v1` and `cls.graph.v2`;
- emits compact JSON compatible with the LiteGraph representation.

Reduced example:

```json
{
  "schema": "cls.graph.v2",
  "litegraph_compat": true,
  "last_node_id": 1,
  "last_link_id": 0,
  "nodes": [
    {
      "id": 1,
      "type": "test.identity",
      "cls_node_id": "n1",
      "pos": [0, 0],
      "flags": {"collapsed": false},
      "size": [180, 60],
      "params": {},
      "inputs": [],
      "outputs": []
    }
  ],
  "links": [],
  "extra": {"cls_schema_version": 2},
  "version": 0.4
}
```

A link is a tuple:

```text
[link_id, src_node_id, src_slot, dst_node_id, dst_slot, port_type]
```

Emitted port values:

```text
image, mask, channel_scs, metadata, scalar,
image_r32f, image_r16u, image_half
```

### Writing

For each node, the serializer writes:

- an integer LiteGraph identifier local to the file;
- `type`, taken from the descriptor's `type_id`;
- `cls_node_id`, the DAG's stable string identifier;
- position, collapsed state, and display size;
- parameters;
- an inventory of inputs and outputs with their links.

Numeric parameters are JSON numbers. `FILE_PATH` and `TEXT` are strings.
Complete persistence of `CURVE` and `BAND_PICKER` is not yet guaranteed.

### Reading

The reader:

- requires a v1 or v2 schema;
- checks all `type` values before creating the first node;
- restores `cls_node_id`, position, collapsed state, and known parameters;
- ignores an unknown parameter;
- recreates links from the `links` array.

It does not restore `size`, `inputs`, `outputs`, `litegraph_compat`,
`last_node_id`, `last_link_id`, `version`, the link tuple's textual type, or
most of `extra` as application data.

The reader adds nodes to the graph it receives: the caller must pass it a new
graph. A late error while reconstructing a link may leave that graph partially
populated.

### Virtual mask input

In v2, an image node recognized as mask-compatible may receive a virtual
`mask` input. Its `dst_slot` is `n_input_ports`, meaning a sentinel index
immediately after the inputs declared by the plug-in. The Scheduler then
composes the node's input and output using this mask.

## `cls.node.manifest.v1`

This payload is generated from the live registry by
`NodeRegistryExporter.cpp`.
It feeds the Graph Editor. It must not be confused with the `manifest.json`
that installs a library.

Example node object:

```json
{
  "schema": "cls.node.manifest.v1",
  "type_id": "scs.example",
  "display_name": "Example",
  "category": "SCS Color",
  "input_ports": [
    {"name": "in", "type": "image_half"}
  ],
  "output_ports": [
    {"name": "out", "type": "image_half"}
  ],
  "params": [
    {
      "name": "amount",
      "display_name": "Amount",
      "type": "float",
      "default": 0,
      "min": 0,
      "max": 100
    }
  ]
}
```

Emitted parameter types:

```text
float, int, bool, enum, color, curve,
band_picker, file_path, text
```

- An enum adds `enum_values`.
- A color adds `default_color` in RGBA form.
- The display name and category have fallbacks.
- The virtual mask input follows the same rule as in the v2 graph.

Sources and tests:

- `GraphSerializer.cpp`;
- `NodeRegistryExporter.cpp`;
- `bridge.ts`;
- `test_graph_serializer.cpp`.
