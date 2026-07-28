# Formats JSON de graphe et de nœuds

Ces formats sont utilisés entre le DAG C++ et le Graph Editor. Ils sont
persistants, mais restent des formats internes sans JSON Schema publié ni
garantie d’interopérabilité tierce en 0.6.5.

## Graphe `.cls.graph`

L’implémentation courante :

- écrit `cls.graph.v2` ;
- lit `cls.graph.v1` et `cls.graph.v2` ;
- émet un JSON compact compatible avec la représentation LiteGraph.

Exemple réduit :

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

Un lien est un n-uplet :

```text
[link_id, src_node_id, src_slot, dst_node_id, dst_slot, port_type]
```

Valeurs de port émises :

```text
image, mask, channel_scs, metadata, scalar,
image_r32f, image_r16u, image_half
```

### Écriture

Pour chaque nœud, le sérialiseur écrit :

- un identifiant entier LiteGraph local au fichier ;
- `type`, provenant du `type_id` du descripteur ;
- `cls_node_id`, identifiant chaîne stable du DAG ;
- position, état replié et taille d’affichage ;
- paramètres ;
- inventaire des entrées et sorties avec leurs liens.

Les paramètres numériques sont des nombres JSON. `FILE_PATH` et `TEXT` sont
des chaînes. La persistance complète de `CURVE` et `BAND_PICKER` n’est pas
encore garantie.

### Lecture

Le lecteur :

- exige un schéma v1 ou v2 ;
- vérifie tous les `type` avant la première création de nœud ;
- restaure `cls_node_id`, la position, l’état replié et les paramètres connus ;
- ignore un paramètre inconnu ;
- recrée les liens depuis le tableau `links`.

Il ne restaure pas comme données métier `size`, `inputs`, `outputs`,
`litegraph_compat`, `last_node_id`, `last_link_id`, `version`, le type textuel
du tuple de lien ni la plupart de `extra`.

Le lecteur ajoute les nœuds au graphe fourni : l’appelant doit lui transmettre
un graphe neuf. Une erreur tardive lors de la reconstruction d’un lien peut
laisser ce graphe partiellement rempli.

### Entrée de masque virtuelle

En v2, un nœud image reconnu comme compatible avec les masques peut recevoir
une entrée virtuelle `mask`. Son `dst_slot` vaut `n_input_ports`, c’est-à-dire
un index sentinelle juste après les entrées déclarées par le plug-in. Le
Scheduler compose ensuite l’entrée et la sortie du nœud au moyen de ce masque.

## `cls.node.manifest.v1`

Cette charge utile est générée depuis le registre vivant par
`NodeRegistryExporter.cpp`.
Il alimente le Graph Editor. Il ne faut pas le confondre avec le
`manifest.json` qui installe une bibliothèque.

Exemple d’objet de nœud :

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

Types de paramètres émis :

```text
float, int, bool, enum, color, curve,
band_picker, file_path, text
```

- Un enum ajoute `enum_values`.
- Une couleur ajoute `default_color` sous forme RGBA.
- Le nom d’affichage et la catégorie possèdent des valeurs de repli.
- L’entrée de masque virtuelle suit la même règle que dans le graphe v2.

Sources et tests :

- `GraphSerializer.cpp` ;
- `NodeRegistryExporter.cpp` ;
- `bridge.ts` ;
- `test_graph_serializer.cpp`.
