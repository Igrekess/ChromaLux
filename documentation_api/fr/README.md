# API et interfaces développeur — ChromaLux 0.6.5

Cette documentation décrit les contrats effectivement présents dans l’arbre
source de ChromaLux 0.6.5. ChromaLux ne fournit pas, à ce stade, d’API
HTTP/REST, de SDK installé par le DMG ni de kit tiers autonome.

## Carte des interfaces

| Surface | Version | Public visé | Statut en 0.6.5 |
|---|---:|---|---|
| ABI binaire des plug-ins | 9 | Équipe et partenaires | Contrat C strict ; recompilation à chaque changement d’ABI |
| Transport des métadonnées | lié à l’ABI 9 | Auteurs de plug-ins | ABI C active |
| Utilitaire JSON de métadonnées | sans version binaire | Auteurs C++ | API source C++17, pas une ABI |
| Manifeste de plug-in | `cls.plugin.manifest.v1` | Équipe et partenaires | Format parsé ; certains champs ne sont pas encore des contrôles |
| IPC des services tether | 1 | Auteurs de fournisseurs de caméras | Protocole local interne, pas une API réseau |
| Graphe | écriture v2, lecture v1/v2 | Outils ChromaLux | Format persistant interne, sans JSON Schema |
| Manifeste du registre de nœuds | `cls.node.manifest.v1` | Graph Editor | Charge utile générée, distincte du manifeste d’installation |
| Outils en ligne de commande | sans version globale | Développement et QA | Binaires issus de la compilation, absents du DMG utilisateur |

## Documentation

- [SDK natif des plug-ins](plugin-sdk.md)
- [Manifeste de plug-in v1](plugin-manifest-v1.md)
- [IPC tether v1](tether-ipc-v1.md)
- [Formats JSON de graphe et de nœuds](wire-formats.md)
- [Outils développeur en ligne de commande](developer-cli.md)

## Sources normatives

Quand un commentaire ou un ancien plan diverge de l’implémentation, les
en-têtes publics, le chargeur, les sérialiseurs et leurs tests priment :

- `chromalux_plugin.h` ;
- `cls_metadata.h` ;
- `cls_metadata_json.h` ;
- `PluginManifest.cpp` et
  `PluginLoader.cpp` ;
- `tether_ipc.h` ;
- `GraphSerializer.cpp` ;
- `NodeRegistryExporter.cpp`.

Les fichiers sous `docs/plans/` et `docs/specs/` décrivent l’intention ou
l’historique d’un chantier. Ils ne constituent pas la référence normative de
la version 0.6.5.

## Frontières actuelles

- Les classes C++ de l’hôte (`Graph`, `Scheduler`, `PluginLoader`,
  `PluginRegistry`, etc.) ne sont pas une API publique stable.
- `cls_composition_mode.h` est un contrat C++ interne ciblé, pas une extension
  générale du SDK.
- Les utilitaires de `src/plugins/builtin/common/` sont réutilisables dans le
  monorepo, mais ne sont ni installés ni exportés.
- Le bridge Python/LLM possède sa propre documentation dans
  `services/cls-llm-python/README.md`.
- Les interfaces Photoshop, les formats SCSP et XMP, les caches et les bases
  SQLite ne font pas partie de cette première référence API.

## Lacunes connues

- aucune règle CMake `install()`/`export()` pour produire un SDK autonome ;
- aucun exemple maintenu hors du dépôt ;
- aucun JSON Schema ou fichier OpenAPI ;
- aucun paquet SDK versionné ;
- aucune référence de symboles générée par Doxygen, MkDocs ou équivalent ;
- la politique de confiance macOS convient aux plug-ins internes et
  partenaires, mais pas encore à un écosystème tiers indépendant.

Ces limites sont documentées pour éviter de confondre une interface présente
dans le code avec une promesse de compatibilité publique.
