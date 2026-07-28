# Outils développeur en ligne de commande

Les outils ci-dessous sont produits dans `<build>/bin`. Ils ne sont pas copiés
dans le DMG utilisateur. Beaucoup nécessitent macOS, Metal, les plug-ins
construits ou des jeux de données de test locaux ; leurs sorties ne sont pas
toutes des API stables.

## `cls_plugin_test`

Usage :

```sh
build-work/bin/cls_plugin_test build-work/plugins/test_identity
```

Le programme effectue quatorze contrôles :

1. présence de `manifest.json` ;
2. JSON lisible ;
3. schéma reconnu ;
4. champs obligatoires présents ;
5. bibliothèque présente ;
6. `dlopen` réussi ;
7. symbole `cls_plugin_init` trouvé ;
8. initialisation non nulle ;
9. ABI binaire exactement égale à 9 ;
10. au moins un descripteur ;
11. `type_id`, `display_name` et vtable complets ;
12. égalité entre `nodes[]` et les descripteurs ;
13. cycle `create`/`destroy` ;
14. `shutdown` puis déchargement.

Résultat attendu :

```text
14/14 PASS
```

Codes de sortie : `0` succès, `1` échec de validation et `2` erreur d’usage.

Limites importantes :

- aucun appel à `process()` ;
- aucun contrôle des pixels, shaders, performances ou budgets ;
- aucune vérification de la signature et de la confiance utilisateur ;
- aucune comparaison de `manifest.api_version` ;
- l’outil exécute malgré tout `dlopen`, `init` et `create` : ne jamais
  l’utiliser sur un binaire non fiable.

## Profils caméra

### `cls_camera_profile_diagnostic`

Diagnostique un profil local et, optionnellement, sa concordance avec une
identité caméra :

- sortie texte ou JSON avec `--format text|json` ;
- lecture seule par défaut ;
- `--import-local` est la seule opération de mutation ;
- un profil importé ainsi reste `Experimental`, avec droits `Unknown`, et
  n’est pas auto-sélectionné.

Les codes de sortie documentés par l’outil sont `0`, `2`, `3`, `4`, `5` et
`64`. Consulter `--help` pour la signification liée au mode choisi.

### `cls_dcp_to_scsp`

Convertit un DCP bi-illuminant complet vers un SCSP v3 :

```sh
build-work/bin/cls_dcp_to_scsp \
  --dcp camera.dcp \
  --out camera.scsp
```

Le chemin v3 refuse un profil simple qui ne fournit pas le contrat
bi-illuminant complet. `--legacy-v1-matrix` demande explicitement la production
d’un SCSP v1 matriciel à partir de ce type de source. La commande écrit le
chemin fourni par `--out`.

## Sessions et catalogue

`cls_session` expose notamment :

```text
create, scan, list, search,
set-rating, set-keywords,
add-color-tag, remove-color-tag,
set-color-label, ingest
```

`set-color-label` est conservé pour compatibilité mais déprécié. `ingest`
possède des options de source, destination, sauvegardes, vérification et MHL ;
en 0.6.5, cette sous-commande est implémentée mais omise de l’aide générale.

## Options diagnostiques de l’application

Le binaire principal accepte entre autres :

```sh
build-work/bin/ChromaLux.app/Contents/MacOS/ChromaLux --version
build-work/bin/ChromaLux.app/Contents/MacOS/ChromaLux --safe-mode
build-work/bin/ChromaLux.app/Contents/MacOS/ChromaLux --plugin-smoke
```

- `--safe-mode` interdit les plug-ins utilisateur.
- `--plugin-smoke` charge et inventorie les plug-ins visibles. Une compilation
  de développement peut aussi contenir les plug-ins de test ; l’application
  packagée dans le DMG 0.6.5 est, elle, validée avec
  `loaded=64 manifests=64`.

## Outils internes et QA

Ces binaires sont des bancs de test internes, sans promesse de compatibilité
de leur interface :

- DAG et performances : `cls_gpu_bench`, `cls_dag_run`, `cls_tile_bench` ;
- RAW et rendu : `cls_raw_decode_smoke`, `cls_thumb_jpeg_smoke`,
  `cls_amaze_smoke`, `cls_halo_smoke`, `cls_xtrans_smoke`,
  `cls_color_smoke`, `cls_develop` ;
- mesures : `cls_noise_probe`, `cls_demosaic_bench`, `cls_recover_bench`,
  `cls_tonemap_bench`, `cls_localcontrast_bench` ;
- grain : `clsfg-render`.

Leur présence dépend des options CMake et de la plateforme. Utiliser
`cmake --build build-work --target help` pour connaître les cibles de la
compilation courante.
