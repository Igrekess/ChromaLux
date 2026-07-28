# SDK natif des plug-ins

## Statut et compatibilité

Le contrat binaire public est l’en-tête C99
`chromalux_plugin.h`. En 0.6.5 :

```c
#define CLS_PLUGIN_API_VERSION 9u
```

L’hôte appelle `cls_plugin_init(9, plugin_dir)`, puis exige
`plugin->api_version == 9`. La comparaison est stricte : un binaire compilé
pour une autre version est rejeté. Un plug-in doit vérifier
`host_api_version` avant toute allocation et retourner `NULL` s’il ne sait pas
servir cette version.

Il n’existe pas encore de couche de compatibilité binaire ascendante. Une
modification de la disposition d’une structure impose donc une recompilation.

## Points d’entrée et cycle de vie

Chaque bibliothèque doit au minimum fournir ces deux points d’entrée :

```c
CLS_EXPORT CLSPlugin* cls_plugin_init(uint32_t host_api_version,
                                      const char* plugin_dir);
CLS_EXPORT void cls_plugin_shutdown(CLSPlugin* plugin);
```

Le chargeur principal 0.6.5 ne rejette pas immédiatement une bibliothèque
qui omet `cls_plugin_shutdown`, mais elle ne respecte alors pas le contrat de
l’en-tête et `cls_plugin_test` la refuse.

Le cycle de vie observé par l’hôte est :

1. chargement de la bibliothèque et appel de `cls_plugin_init` ;
2. lecture de `CLSPlugin` et de ses `CLSNodeDescriptor` ;
3. création d’une instance par `descriptor.vtable.create(plugin_ctx)` ;
4. notification des valeurs numériques par défaut avec
   `on_param_changed` ;
5. appels de `process()` ;
6. `destroy()` de toutes les instances ;
7. `cls_plugin_shutdown(plugin)` ;
8. déchargement de la bibliothèque.

### Propriété de la mémoire

- Les chaînes et descripteurs statiques appartiennent au plug-in. L’hôte ne
  les libère pas.
- `CLSNode*` est opaque et appartient au plug-in entre `create` et `destroy`.
- Les textures, services et structures de requête sont empruntés à l’hôte.
- Un plug-in ne doit conserver aucun pointeur issu de `CLSExecRequest` après
  le retour de `process()`.

### Threads

- `create`, `destroy` et `on_param_changed` sont appelés sur le thread
  principal.
- `process` peut être appelé sur n’importe quel thread, mais jamais
  simultanément pour la même instance `CLSNode`.
- `process` ne doit pas effectuer de long travail CPU bloquant ni attendre
  inutilement le GPU.

## Codes de résultat

| Code | Signification |
|---|---|
| `CLS_OK` | succès |
| `CLS_ERR_INVALID_ARG` | argument, port ou capacité invalide |
| `CLS_ERR_OUT_OF_MEMORY` | allocation impossible |
| `CLS_ERR_GPU` | création, encodage ou exécution GPU en échec |
| `CLS_ERR_UNSUPPORTED` | opération reconnue mais non disponible |
| `CLS_ERR_INTERNAL` | erreur interne sans catégorie plus précise |

## Ports

| Type | Transport actuel |
|---|---|
| `CLS_PORT_IMAGE` | texture RGBA32F |
| `CLS_PORT_IMAGE_HALF` | texture RGBA16Float |
| `CLS_PORT_MASK` | texture R32F |
| `CLS_PORT_IMAGE_R32F` | texture R32F |
| `CLS_PORT_IMAGE_R16U` | texture R16Uint, notamment pour le Bayer |
| `CLS_PORT_METADATA` | pointeur vers `CLSMetadataBlob` dans le champ `texture` |
| `CLS_PORT_CHANNEL_SCS` | déclaré dans l’ABI, pas routé par le Scheduler 0.6.5 |
| `CLS_PORT_SCALAR` | déclaré dans l’ABI, pas routé par le Scheduler 0.6.5 |

Le graphe accepte les connexions croisées `IMAGE` ↔ `IMAGE_HALF`. Le format
réel d’un port peut aussi être surchargé par l’hôte dans certaines chaînes de
traitement ;
le plug-in doit lire `CLSPortData.type`.

## Paramètres

Les valeurs numériques sont placées dans `CLSExecRequest.param_values` :

- `FLOAT`, `INT` et `BOOL` sont actifs ;
- `ENUM` transporte l’index sélectionné sous forme de `double` ;
- `COLOR` est déclaré et possède `default_color[4]`, mais ne dispose pas
  encore d’une chaîne complète et uniforme dans tous les consommateurs ;
- `CURVE` et `BAND_PICKER` sont des surfaces partielles/réservées.

`FILE_PATH`, `TEXT`, `CURVE` et `BAND_PICKER` transportent leur valeur par
`setStringParam` puis `param_strings`. Pour les deux derniers, la charge utile est
une sérialisation JSON ou une liste propre au consommateur ; leur persistance
et leurs consommateurs restent incomplets en 0.6.5. Ces pointeurs
appartiennent à l’hôte et ne sont valides que pendant `process()`. L’ABI ne
contient pas de `default_string`. Dans l’implémentation actuelle, une mise à
jour par `setStringParam` ne déclenche pas `on_param_changed` : le plug-in lit
la valeur dans la requête de traitement.

`min_float` et `max_float` décrivent l’interface, mais ne remplacent pas une
validation côté plug-in.

## Métadonnées

`cls_metadata.h` expose
`CLSMetadataBlob`. Le transport actuel est :

- JSON UTF-8 ;
- capacité par défaut de 8 192 octets ;
- données pas nécessairement terminées par `\0` ;
- `size` doit rester inférieur ou égal à `capacity` ;
- le plug-in écrit les octets et met `size` à jour, sans libérer le tampon.

La durée de vie du blob est celle de la sortie gérée par le Scheduler.

`cls_metadata_json.h` fournit un analyseur lexical C++17 uniquement sous forme
d’en-tête. C’est une API source, pas une ABI ni un parseur
JSON général. Il ne prend notamment pas en charge `\uXXXX`, les commentaires,
les virgules finales, `NaN`, `Infinity` ou un BOM ; sa conversion numérique
emploie un tampon interne limité à 63 caractères.

## Exécution tuilée, LOD et ROI

`CLSExecRequest` décrit la tuile demandée par `tile_x`, `tile_y`, `tile_w`,
`tile_h`, les dimensions complètes et `lod_level`.

Chaque `CLSPortData` fournit :

- la texture et son format ;
- la largeur et la hauteur du tampon fourni ;
- `origin_x` et `origin_y`, exprimés dans l’espace du LOD ;
- `full_w` et `full_h`, également dans l’espace du LOD.

`full_w == 0` désigne un ancien appel plein cadre ; le plug-in doit alors
interpréter l’origine comme `(0, 0)` et les dimensions complètes comme celles
du port.

Un nœud qui change la géométrie doit se baser sur les dimensions de sa sortie,
pas supposer que toutes les entrées et sorties ont la même taille.

`modify_roi_in` permet de convertir une tuile de sortie demandée en région
d’entrée. En ABI v9, la même ROI est appliquée à toutes les entrées image :
il n’existe pas encore de ROI distincte par port.

`halo_radius` exprime le voisinage nécessaire à un filtre. Le Scheduler 0.6.5
calcule le halo maximal utile au graphe et élargit les tuiles pour éviter les
coutures ; un ancien commentaire de l’en-tête mentionnant systématiquement un
plein cadre ne décrit plus toute l’implémentation courante.

## Metal et tampon de commandes partagé

`metal_device` et `metal_command_queue` sont empruntés. Un plug-in ne doit pas
créer sa propre file d’attente.

Un descripteur peut annoncer :

```c
execution_caps = CLS_NODE_CAP_SHARED_METAL_CB;
```

Lorsque l’hôte active aussi ce chemin, `metal_execution` fournit un
`command_buffer` emprunté. Le plug-in peut y créer et fermer ses encodeurs,
mais il ne doit pas :

- retenir le tampon ;
- appeler `commit` ou `waitUntilCompleted` ;
- ajouter un gestionnaire de fin d’exécution.

Le Scheduler possède la soumission. Le plug-in doit conserver un chemin
fonctionnel lorsque `metal_execution == NULL`, car le tampon partagé dépend du
contrôle à l’exécution `CLS_SHARED_TILE_CB`.

## Services de l’hôte

`CLSHostServices_v1.version` vaut `1`. Toujours vérifier la table et chaque
fonction de rappel avant de les appeler.

- `gpu_read_back` effectue une lecture synchrone. L’implémentation 0.6.5 du
  Scheduler ne prend en charge que les textures RGBA32F.
- `run_batch_subgraph` est réservé mais retourne actuellement
  `CLS_ERR_UNSUPPORTED` et place la sortie à `NULL`.

## Construction dans le dépôt

L’utilitaire CMake interne s’utilise ainsi :

```cmake
cls_add_plugin(example_plugin
    SOURCES plugin.mm
    MANIFEST manifest.json
    KERNEL kernel.metal
    LIBRARY example_plugin
)
```

Il construit la dylib, copie le manifeste et compile éventuellement un
`.metallib`. `cls_add_plugin` dépend toutefois de la structure du monorepo :
ce n’est pas encore un package CMake exportable pour un projet externe.

Exemples de référence :

- `src/plugins/test/identity/` ;
- `src/plugins/test/constant_color/` ;
- `cmake/ChromaluxPlugin.cmake`.

Validation structurelle :

```sh
cmake --build build-work --target cls_plugin_test test_identity
build-work/bin/cls_plugin_test build-work/plugins/test_identity
```

Le résultat attendu est `14/14 PASS`. Cet outil ne traite pas de pixels ; ses
limites sont détaillées dans [les outils développeur](developer-cli.md).
