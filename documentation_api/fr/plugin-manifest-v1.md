# Manifeste de plug-in v1

Le fichier `manifest.json` décrit le paquet que le chargeur doit ouvrir. Son
schéma est `cls.plugin.manifest.v1`.

## Exemple recommandé

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

## Champs et validation actuelle

| Champ | Requis | Traitement en 0.6.5 |
|---|---|---|
| `schema` | oui | doit valoir exactement `cls.plugin.manifest.v1` |
| `id` | oui | chaîne ; la convention reverse-DNS n’est pas validée |
| `version` | oui | chaîne ; la syntaxe semver n’est pas validée |
| `api_version` | oui | entier parsé, mais pas utilisé comme contrôle par le chargeur |
| `name` | oui | chaîne |
| `vendor` | oui | chaîne |
| `license` | oui | chaîne |
| `library` | oui | chemin relatif vers la bibliothèque |
| `nodes` | oui | tableau non vide de chaînes |
| `min_host_version` | oui | chaîne parsée, pas encore appliquée au chargement |
| `requires_gpu` | non | booléen, `false` par défaut, pas encore appliqué |
| `requires_metal` | non | booléen, `false` par défaut, pas encore appliqué |

Les champs inconnus sont ignorés, y compris les anciens champs `description`
et `params`. Les paramètres exposés à l’hôte viennent exclusivement des
`CLSNodeDescriptor` retournés par le binaire.

## Version déclarée et ABI réelle

`manifest.api_version` et `CLSPlugin::api_version` sont deux données
différentes :

- le parseur exige que le champ JSON existe ;
- le chargeur ne compare pas sa valeur à l’ABI ;
- le chargeur exige en revanche strictement que le binaire retourne
  `CLSPlugin::api_version == CLS_PLUGIN_API_VERSION`, soit `9`.

Le dépôt contient encore des manifestes historiques portant les valeurs
`1`, `3`, `5`, `6`, `7` et `9`, alors que leurs binaires reconstruits
exposent l’ABI 9. Un nouveau manifeste doit déclarer `9` pour éviter cette
ambiguïté, même si ce champ n’est pas encore un contrôle.

## Contrôles effectués au chargement

Le chargeur :

- parse le manifeste et vérifie son schéma ;
- exige que la bibliothèque existe dans le répertoire du plug-in ;
- rejette un chemin absolu, une traversée `..` ou un lien symbolique sortant ;
- charge avec `RTLD_NOW | RTLD_LOCAL` ;
- recherche `cls_plugin_init` ;
- exige un résultat non nul et une ABI binaire exactement égale à 9.

Le chargeur ne vérifie pas encore :

- l’égalité entre `manifest.api_version` et l’ABI binaire ;
- `min_host_version` ;
- les drapeaux `requires_gpu` et `requires_metal` ;
- la cohérence des identifiants et versions entre JSON et `CLSPlugin` ;
- l’égalité de `nodes[]` avec les descripteurs.

`cls_plugin_test` ajoute notamment le contrôle de `nodes[]`, de la vtable et du
cycle `create`/`destroy`.

## Arborescence

Un paquet minimal est :

```text
example/
├── manifest.json
├── libexample.dylib
└── kernel.metallib
```

Le `.metallib` est facultatif. Pour un plug-in utilisateur, le dossier
`example/` réside sous :

```text
<QStandardPaths::AppDataLocation>/Plugins/
```

## Signature et confiance sur macOS

Avant tout `dlopen`, un plug-in utilisateur :

- doit posséder une signature valide ;
- doit être signé avec le même Team ID Apple que ChromaLux ;
- doit être approuvé explicitement par l’utilisateur ;
- est ensuite autorisé par l’empreinte SHA-256 exacte de la bibliothèque ;
- est refusé lorsque `CLS_SAFE_MODE=1`.

Une mise à jour modifiant les octets change l’empreinte et demande une nouvelle
approbation. Ce modèle convient aux plug-ins internes, privés ou partenaires.
Il ne constitue pas encore une politique de distribution pour un marché tiers
indépendant.

Sources : `PluginManifest.cpp`,
`PluginLoader.cpp` et
`Application.cpp`.
