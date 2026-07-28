# IPC tether v1

Ce protocole relie ChromaLux aux processus locaux `cls-tether-*`. Il s’agit
d’un contrat interprocessus interne pour des services signés, pas d’une API
réseau publique.

Sources de référence :

- `tether_ipc.h` ;
- `tether_control_channel.h` ;
- `TetherSession.cpp` ;
- `LiveViewConsumer.cpp` ;
- `cls-tether-mock`.

## Architecture

1. L’hôte crée un socket Unix local.
2. Il lance le service en transmettant le chemin dans `CLS_TETHER_SOCK`.
3. Le service se connecte au socket.
4. Le contrôle utilise un objet JSON compact par ligne, délimité par `\n`.
5. Les images Live View passent par un tampon circulaire POSIX en mémoire
   partagée.

Une `TetherSession` pilote un service et une caméra ouverte. La version du
protocole est `1`.

## Enveloppes JSON

Requête :

```json
{"id":1,"kind":"request","method":"tether.hello","args":{}}
```

Réponse réussie :

```json
{"id":1,"kind":"response","method":"tether.hello","result":{}}
```

Événement :

```json
{"id":7,"kind":"event","method":"tether.camera_added","args":{"camera":{}}}
```

Une réponse contient `result` ou `error`, jamais les deux. Les identifiants
mettent en correspondance requêtes et réponses ; les événements sont
asynchrones.

Le lecteur commun des services fournisseurs limite une ligne de contrôle à
1 Mio. L’hôte et le service de simulation n’appliquent pas partout la même
borne en 0.6.5 : une intégration doit rester strictement sous 1 Mio par
message.

## Méthodes

| Méthode | Entrée principale | Résultat |
|---|---|---|
| `tether.hello` | versions hôte/protocole | identité du service, versions, capacités |
| `tether.list_cameras` | objet vide | tableau `cameras` |
| `tether.open` | `camera_id`, `capture_dir` | session, source, shm, capacités éventuelles |
| `tether.close` | objet vide | résultat vide |
| `tether.trigger_capture` | `capture_id` | capture, source et horodatage |
| `tether.get_setting` | `key` | instantané du réglage |
| `tether.set_setting` | `key`, `value` | valeur réellement appliquée |
| `tether.list_settings` | objet vide | inventaire des réglages |
| `tether.live_view_start` | `max_fps`, `format` | format et source |
| `tether.live_view_stop` | objet vide | résultat vide |
| `tether.af_trigger` | objet vide | `af_status` |
| `tether.focus_step` | direction et taille normalisées | statut et pas natifs |
| `tether.health` | objet vide | diagnostic propre au fournisseur |
| `tether.diagnostic_config` | objet vide | état gPhoto2 en lecture seule |

`diagnostic_config` n’est pas une surface de modification des réglages.

## Événements

- `tether.capture_transfer_started`
- `tether.capture_file_ready`
- `tether.capture_complete`
- `tether.setting_changed`
- `tether.camera_added`
- `tether.camera_removed`
- `tether.disconnected`

Un couple RAW+JPEG partage le même `capture_id`. L’hôte peut ainsi regrouper
les fichiers d’une même transaction jusqu’à `capture_complete`.

## Caméras et réglages

Les champs de caméra reconnus comprennent :

```text
id, provider, make, model, transport, serial, firmware,
usb_location, support_level, support_reason, available
```

Un instantané de réglage suit cette forme :

```json
{
  "key": "iso",
  "value": 400,
  "display_name": "ISO",
  "choices": [100, 200, 400],
  "display": ["100", "200", "400"],
  "min": 100,
  "max": 12800,
  "step": 100,
  "read_only": false
}
```

L’UI emploie notamment les clés normalisées `exposure_mode`,
`exposure_comp`, `shutter_us`, `iso`, `aperture`, `wb_mode`, `wb_kelvin`,
`battery` et `meter_ev`. Un fournisseur peut ajouter des clés préfixées par un
espace de noms.

## Capacités

La réponse à `tether.hello` peut annoncer :

- `capture`
- `live_view`
- `settings`
- `shutter_button`
- `ae_lock`
- `af_trigger`
- `focus_step`

L’hôte doit conditionner toute commande optionnelle à la capacité
correspondante. Un ancien service peut répondre avec l’erreur
`1001 unknown_method`.

## Erreurs

L’objet `error` contient au minimum un code et un message.

| Code | Constante | Sens |
|---:|---|---|
| 5 | `kClsTetherErrInternal` | erreur d’entrée/sortie interne |
| 6 | `kClsTetherErrNoCamera` | caméra absente |
| 13 | `kClsTetherErrPermission` | permission refusée |
| 16 | `kClsTetherErrBusy` | ressource occupée |
| 22 | `kClsTetherErrInvalidArg` | argument invalide |
| 95 | `kClsTetherErrUnsupported` | opération non prise en charge |
| 110 | `kClsTetherErrTimeout` | délai dépassé |
| 1000 | `kClsTetherErrProtocol` | erreur de protocole |
| 1001 | `kClsTetherErrUnknownMethod` | méthode inconnue |
| 1002 | `kClsTetherErrSessionNotFound` | session absente |
| 1003 | `kClsTetherErrHandshakeMismatch` | versions incompatibles |
| ≥ 2000 | fournisseur | erreur propre au SDK ou au matériel |

Les raisons de déconnexion sur le fil sont `usb_unplug`, `sdk_crash`,
`daemon_exit` et `host_close`. L’hôte peut aussi produire localement des
raisons telles que `socket_eof`.

## Tampon circulaire du Live View

Le fichier partagé utilise l’ordre natif de la machine.

| Élément | Valeur |
|---|---:|
| magic | `0x434C5354` (`CLST`) |
| version | 1 |
| en-tête du tampon circulaire | 64 octets |
| nombre d’emplacements | 4 |
| taille d’un emplacement | 4 194 304 octets |
| taille totale | 16 777 280 octets |
| en-tête d’une trame | 40 octets |
| charge utile maximale | 4 194 264 octets |
| format JPEG | 1 |
| format RGBA32F | 2 |
| fin de flux | bit 0 de `flags` |

Le service écrit l’en-tête et la charge utile d’un emplacement, puis publie
`head_seq` avec une sémantique release. Le consommateur lit avec une
sémantique acquire, choisit la trame la plus récente et peut abandonner les
trames intermédiaires.

`ClsTetherFrameHeader` fournit `seqno`, l’horodatage microseconde, le format,
les dimensions, la taille utile et les flags.

## Limites d’intégration

- Le socket local ne possède pas d’authentification applicative.
- Une session accepte un seul client.
- Les fournisseurs 0.6.5 n’appliquent pas tous les contrôles de version avec la
  même rigueur.
- Les méthodes optionnelles ne sont pas uniformément disponibles.
- Ce protocole est réservé à l’hôte et à ses services signés ; l’exposer sur le
  réseau sans nouvelle couche de sécurité est hors contrat.
