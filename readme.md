# ChromaLux Studio 0.6.6

[English](#english) · [Français](#français)

## English

### Overview

ChromaLux Studio is a native macOS photographic workspace for RAW development,
color grading, and studio production workflows. It brings secure ingest,
session management, non-destructive development, local corrections, tethered
capture, camera calibration, creative compositing, color inspection, export,
and printing into one environment.

The application uses a modular Metal-accelerated pipeline built on two
complementary foundations:

- **HALO**, the recommended RAW demosaicing engine;
- **SCS**, the tonal and chromatic processing domain shared by grading tools,
  perceptual masks, color probes, and scopes.

ChromaLux 0.6.5 is an alpha version for Apple Silicon Macs running macOS 26
Tahoe or later.

### HALO — reconstructing the RAW image

**HALO — Holonomic Adaptive Local Operator** reconstructs camera-linear RGB
from the mosaic recorded by the sensor. This gradient-adaptive demosaicing
engine runs on the GPU through Metal.

HALO analyzes several local directions—horizontal, vertical, and diagonal—as
well as neighborhood structure to arbitrate reconstruction. It is designed to
retain edges and detail while limiting false color, zippering, and
interpolation artifacts. Reconstruction remains in the scene-linear pipeline
to preserve precision and highlight headroom for downstream stages.

The Bayer path is ChromaLux’s recommended engine; AMaZE remains available as
an alternative. Fujifilm X-Trans sensors are also processed by HALO in the
0.6.5 production pipeline, although this route is still marked experimental.
Sigma/Foveon X3F files use a dedicated decoding route.

HALO is not a camera profile: it reconstructs RAW pixels. The camera-specific
color transform is applied afterwards by a compatible SCSP profile or the
decoder’s camera matrix.

### What SCS adds to grading

SCS organizes processing around an explicit separation between scene
luminance or density and chromatic geometry. Its color metric makes it
possible to reason in chromatic distances and directions instead of simple
arithmetic differences in RGB channels.

In practice, this architecture provides:

- more independent controls for light, density, chroma, and chromatic
  direction;
- color corrections designed to limit unwanted luminance or hue variations;
- explicit protection for neutrals, skin tones, and neighboring colors;
- local selections based on chromatic distance, used by local points,
  parametric masks, and AI edge refinement;
- in v2/v3 pipelines, scene-referred processing that retains RAW headroom until
  the display transform;
- continuity across Curves, Levels, Color Editor, Color Balance, Harmony,
  probes, Vectorscope, and Parade;
- separate **Scene**, **Display**, and **Delivery** readings, so the stage at
  which a value is measured remains explicit.

SCS does not replace the camera’s SCSP profile, the display’s ColorSync
profile, or the delivery ICC profile. Each has a distinct responsibility:

```text
RAW → selected decoding and demosaicing → white balance
    → SCSP profile or decoder camera matrix
    → scene-linear grading and SCS tools
        ├─ presentation: Viewer space → optional proof ICC transform
        │                 → display ColorSync profile
        └─ delivery: recipe ICC profile → file or print
```

SCS makes it possible to adjust light, density, or color more precisely while
limiting unwanted effects on the other components of the image. The same
references are used by grading tools, masks, probes, and scopes, making the
result more predictable and easier to control.

### ChromaLux 0.6.5 feature inventory

#### Sessions, ingest, and original protection

- Create and open `.clssession` sessions with **Captures**, **Selects**,
  **Output**, and **Trash** folders.
- Browse within the session or through accessible external folders.
- Import from a card, volume, or folder, with preview and selection before
  copying.
- Configurable destination, folder structure, token-based naming, and common
  metadata.
- Optional backup copy, destination verification or source/destination
  checksum comparison, and ASC-MHL emission.
- Automatic opening of the import workflow when a recognized photo card is
  inserted.
- Post-import erasure disabled by default and protected by strict
  verification, ASC-MHL, typed `ERASE`, and final confirmation.

#### Library, selection, and non-destructive editing

- Folder browser, thumbnail grid, focused photograph, and multi-selection.
- Move or copy photographs with their associated files.
- Split view for up to twelve photographs or variants, Focus loupe, second
  viewer, zoom, pan, fit, and 100% view.
- Zero-to-five-star ratings, picked/rejected/unmarked states, and multiple
  color labels that can synchronize with Finder tags.
- Sort by name, date, rating, color, ISO, size, aperture, shutter speed, or
  extension; quick filtering within the current folder.
- Independent variants per original, including create, clone, delete, and
  explicit duplication to an HDR v3 variant.
- Adjustments, layers, masks, measurements, and compositions stored in
  versioned sidecars; distinct JXL thumbnails per variant.
- Undo/redo, temporary before/after comparison, complete or partial adjustment
  copying, and optional synchronization across a selection.
- Metadata copy/paste, batch editing, and batch export.

#### RAW development and detail

- Explicit development routes for supported `NEF`, `CR2`, `CR3`, `ARW`,
  `DNG`, `RAF`, `IIQ`, and `X3F` files.
- HALO demosaicing for Bayer and X-Trans, AMaZE as a Bayer alternative, plus a
  dedicated Sigma/Foveon route. The HALO X-Trans route remains marked
  experimental.
- **Photography** and **Science** processing intents.
- White balance, temperature, tint, automatic neutralization, and cast
  correction.
- Exposure, balance, brightness, contrast, saturation, and vibrance.
- Per-channel and spatial highlight recovery; shadow, highlight, white, black,
  density, and bleach controls.
- Tone mapping, shadow detail, and extended dynamic-range controls.
- Sharpening, radius, texture, clarity, structure, and luminance/chroma noise
  reduction.
- Camera/ISO noise profiles when available, with a generic fallback.
- Primary, purity, and base-characteristic controls.

#### Camera profiles and color management

- Versioned v1, v2, and v3 color pipelines that preserve historical rendering
  without silent migration.
- High-precision linear transport in v2/v3 pipelines.
- Historical SCSP v1, strict SCSP v2, and bounded dual-illuminant DCP-based
  SCSP v3 camera profiles for three-channel Camera RGB and SDR appearance.
- Free camera-profile selection per variant, explicit decoder-matrix fallback,
  and a separate default per camera model.
- Local import and diagnostics for compatible DCP or SCSP profiles, including
  provenance and hashes. Imported profiles remain experimental, are never
  activated automatically, and require verification of usage rights.
- Dual-illuminant DCP solving through reciprocal-temperature interpolation,
  clamping at illuminant bounds, and rejection of inconsistent structures.
  There is no independent tint axis; `ReductionMatrix` is parsed and validated
  but is not applied by the current three-channel solver.
- A chart-measured Nikon D850 profile qualified for automatic matching; other
  cameras use the decoder matrix until a profile is explicitly selected or
  qualified.
- sRGB, Adobe RGB (1998), and Display P3 Viewer spaces.
- Identification of the active display ColorSync profile.
- Separate profiles for camera input, Viewer rendering, display, delivery,
  proofing, and printing.
- Soft proofing, paper-white simulation, gamut warning, and experimental
  **OKLab v1** perceptual output gamut compression for sRGB, Adobe RGB, and
  Display P3 RGB cubes.
- EDR preview in extended linear Display P3 on a v3 variant and compatible
  Apple display, with automatic SDR fallback; soft proofing also forces SDR.

#### Tone and color grading

- SCS curves for luminance, saturation, and RGB channels.
- Fisher density curve, SCS Levels, and range-based Tone Equalizer.
- Dehaze with depth and warm-tone protection.
- Purple and green fringe correction.
- Global, shadow, midtone, and highlight Color Balance with skin-tone
  protection.
- Color-family Color Editor and advanced skin-tone controls.
- Color Harmony based on master hues.
- Black & White with per-color sensitivity and split toning.
- Creative vignette.
- Histogram and clipping warnings available during grading.

#### Layers and masks

- Switchable adjustment layers, adjustable opacity, and **Normal**,
  **Multiply**, **Screen**, **Overlay**, **Soft Light**, **Difference**,
  **Add**, and **Subtract** blend modes.
- Local Exposure, White Balance, Color Editor, Color Balance, Primaries,
  Fisher Curve, Black & White, and Extended Dynamic Range adjustments.
- Brush, Linear Gradient, Radial Gradient, SCS Local Point, Parametric SCS, and
  multi-sample Chromatic masks.
- Add/subtract painting, feathering, whole-mask blur, handles, and live Viewer
  preview.
- **Image**, **Overlay**, and black-and-white **Matte** views.
- Person and background selection with Apple Vision.
- Object selection with SAM 2.1 Small when its Core ML models are installed.
- Optional SCS color/luminance refinement of Apple Vision or SAM edges.

#### Geometry, optics, and framing

- Rotate, straighten, and crop with aspect ratios, locking, and composition
  guides.
- Vertical, horizontal, or full perspective correction with editable guides.
- Geometric distortion correction from compatible DNG data or an exact
  Lensfun profile.
- Manual distortion and optical-center controls.
- Vignetting and fringe correction in separate manual modules; ChromaLux does
  not automatically apply Lensfun vignetting or lateral chromatic aberration.
- Non-destructive geometry stored per variant.

#### Film Simulator

- Film-curve catalog with Standard and Expert modes.
- Amount, contrast, brightness, toe, shoulder, gamma, and maximum density.
- Pull, Normal, or Push development when supported by the film data.
- Black & White, split toning, and adjustable halation.
- Silver grain with physical size, polydispersity, seed, diffusion, chroma, and
  crosstalk.
- Darkroom print simulation with paper choice, diffusion, warmth, texture, and
  toning.
- Finishing vignette, presets, and transfer of a look across a series.

Film names cover both measured responses and initial models; they do not
certify a specific film, lab, and scanner combination.

#### Creative compositing

- **Double Exposure** from two images, with exposure balancing, transformation
  of the second source, negative, contact, or positive modes, grain, and a
  derived linear DNG.
- **Raw Mockup** for placing, transforming, and grading RAW or rendered sources
  in a composition.
- Raw Mockup connection to the shared Live View stream.
- **ChromaKey** on a photograph or composition: background/protection samples,
  matte construction and finishing, despill, foreground recovery, and
  replacement with a RAW or rendered image.
- Composite, Matte, recovered foreground, background estimate, despill
  difference, and confidence diagnostic views.

Direct ChromaKey replacement inside Live View is not a 0.6.5 feature.

The DNG produced by Double Exposure currently uses a generic sRGB-D65 matrix;
retrieval of each source camera’s actual profile is deferred.

#### Studio tethering

- Integrated Nikon, Canon, and Fujifilm providers, with gPhoto2 as an explicit
  manual fallback.
- Discovery, automatic or manual connection, and a Mock test provider.
- Next-capture folder and naming template.
- Triggering, transactional download, automatic selection, and RAW + JPEG pair
  handling.
- Camera controls according to provider-advertised capabilities.
- Live View in a dedicated window or the main Viewer.
- One Live View stream shared across the window, Viewer, and Raw Mockup.
- JSON diagnostics without photographic content.

Compatibility depends on camera model, firmware, manufacturer driver or SDK,
and the capabilities reported by each device.

#### Studio camera calibration

- ColorChecker 24-patch measurement with manual four-corner placement and
  three reference sets.
- Strict SCSP v2 profile generation, calibration residuals, and rejection of
  inconsistent profiles.
- Managed installation and selection on the active variant.
- Profile Editor for deriving a look from base, tone, color correction, and
  Black & White settings.
- DCP/SCSP profile import and diagnostics.

The D65 choice in the CC24 workflow is not fully operational in 0.6.5: the
generator currently uses the D50 reference.

#### Color probes, scopes, and Science activity

- Point, square, or circular probe in **Scene**, **Display**, or **Delivery**
  domains.
- Floating-point RGB, D50 Lab, HSV, luminance, EV, and nits when the required
  metadata is available.
- Up to eight locked probes with a reference and RGB, HSV, DeltaE00, EV, or
  nits comparisons.
- Scene- or output-domain histogram.
- SCS Vectorscope with skin-tone axis, targets, saturation alerts, zoom, and
  display options.
- SCS Parade in RGBL+Sat, RGB, luminance, saturation, or SCS component modes,
  with Legal, Rec.709, P3, and Rec.2020 targets.
- Reference traces; SCS Parade export to PNG and its density data to CSV;
  False Color overlay export to PNG.
- Luminance, temperature, and saturation false color; channel inspection.
- Grid, crosshair, scale bar, guides, distance/area/angle measurements, text,
  and persistent annotations.
- FITS reading and rotation, Astro/Microscope previews, and specialized PT
  contrast maps.

#### Metadata and local automation

- EXIF metadata reading; editing of descriptive XMP/IPTC fields.
- Session details, production credits, title, description, keywords,
  copyright, and credits.
- Batch keyword add/remove and metadata copy/paste.
- Naming templates using session information, sequence, and available camera
  metadata.
- Local metadata suggestions with Apple Vision and Ollama when a compatible
  vision model is installed.
- Manual validation or automatic acceptance of AI suggestions.

#### Brief, contact sheets, export, and print

- Production Brief with rectangles, ellipses, freehand drawing, brush, text,
  eraser, priorities, colors, references, and separate layers.
- Local French or English dictation and transcription after installing the
  speech model.
- PDF, PNG, Markdown, or JSON Brief reports and optional inclusion in exports.
- Up to four contact-sheet windows; A4, A3, 4K, 8K, or custom sizes; grids,
  margins, logo, captions, and an editable native document.
- Multiple export recipes in one operation, for the focused image or a
  selection.
- 8-bit `JPEG`, 8/16-bit `TIFF` with no compression, LZW, or ZIP, 8/16-bit
  `PNG`, and 8-bit `PSD` for layered Brief output.
- sRGB, Adobe RGB (1998), and ProPhoto RGB delivery profiles; naming,
  subfolders, resizing, and collision policies.
- Print one image, a repeated grid, or a contact sheet, with
  ChromaLux-managed or printer-managed color.
- RGB or CMYK print profiles, soft proof, gamut warning, black-point
  compensation, paper simulation, output sharpening, physical print grain,
  and captions.
- Page export to TIFF, JPEG, or PNG and printing through the macOS dialog.

#### Interface, customization, and developer interfaces

- Specialized activities: Library, Calibration, Tethered Capture,
  Adjustments, Film Simulator, Science, Contact Sheet, Metadata, Brief, Export,
  and Print.
- Reorderable or detachable modules, custom tabs, one or two columns, multiple
  instances, and custom activities.
- Module presets, copy/paste, bypass, reset, and saved complete workspace
  layouts.
- Customizable toolbar, Light/Dark/System themes, Compact/Comfortable density,
  and French/English interface.
- Keyboard shortcut editor, keyboard navigation, and accessibility information
  on major workflows.
- Modular plug-in architecture; native ABI v9, manifests, tether IPC v1, graph
  formats, and developer tools documented for internal or partner use.

### Limitations and experimental features

The following are present but should not be considered fully qualified for
every workflow:

- HALO X-Trans, used by the production pipeline but still marked
  experimental;
- **OKLab v1** perceptual output gamut compression, limited to sRGB, Adobe RGB,
  and Display P3 output and without completed psychovisual validation on a
  real-image corpus; it does not replace rendering through an arbitrary print
  ICC profile;
- EDR preview, limited to v3 variants and compatible Apple displays; paper
  white and creative peak are bounded by display-declared headroom rather than
  instrumental measurements, and physical/multi-display qualification remains
  manual;
- imported DCP or SCSP profiles before visual qualification and usage-rights
  verification;
- Apple Vision and SAM selection, Ollama AI metadata, and local
  transcription, which depend on the system or installed models;
- PT Contrast maps, Astro/Microscope previews, and the still-partial line
  profile;
- Brief PSD/TIFF interoperability, which must be checked in the target
  application;
- tethered capture, which depends on camera, firmware, and provider;
- VoiceOver not replayed end to end, graphical Mask/Perspective handles that
  require a pointing device, and still-partial Perspective keyboard navigation.

Export and printing remain SDR: ChromaLux 0.6.5 does not yet encode HDR10/PQ or
HLG files or their delivery metadata. Calibration **Scientific**,
**Detector**, and **Geometric** panels are placeholders; Node Editor v1 is
read-only; Clone/Heal are disabled.

Plug-in interfaces are intended for internal, private, or partner use: a
plug-in must be signed with the same Team ID and explicitly trusted by
fingerprint. There is no public marketplace or third-party SDK installed by
the DMG yet, and developer command-line tools are not part of the distributed
application.

### Formats and requirements

The import scanner recognizes more formats than the Viewer can develop.
Detection or copying therefore does not guarantee decoding.

- Formats detected by import: `NEF`, `NRW`, `CR2`, `CR3`, `CRW`, `ARW`,
  `SR2`, `SRF`, `RAF`, `RW2`, `ORF`, `PEF`, `DNG`, `IIQ`, `3FR`, `X3F`,
  `GPR`, `JPG`, `JPEG`, `HEIC`, `HEIF`.
- RAW formats actually routed by the Viewer: `NEF`, `CR2`, `CR3`, `ARW`,
  `DNG`, `RAF`, `IIQ`, `X3F`, subject to supported encoding.
- Visible and indexed rendered images: `JPG`, `JPEG`, `PNG`, `TIF`, `TIFF`,
  `HEIC`, `HEIF`, `WEBP`. HEIC/HEIF and WebP depend on decoders available in
  the package or macOS.
- Scientific images: `FITS`, `FIT`, `FTS`.
- Requirements: Apple Silicon Mac, macOS 26 Tahoe or later, compatible Metal
  GPU.
- Version and build: `0.6.5`, `Release` configuration.
- This beta expires on December 1, 2026.

### Package contents

- [ChromaLux-0.6.5.dmg](https://github.com/Igrekess/ChromaLux/releases/download/v0.6.5/ChromaLux-0.6.5.dmg) — signed, notarized, and stapled
  macOS application;
- [French User Guide](documentation_utilisateur/Guide-utilisateur-ChromaLux-FR.pdf);
- [English User Guide](documentation_utilisateur/ChromaLux-User-Guide-EN.pdf);
- [French and English API documentation](documentation_api/README.md).

### Installation

1. Open `ChromaLux-0.6.5.dmg`.
2. Drag `ChromaLux.app` to the `Applications` folder.
3. Launch ChromaLux from `Applications`.

### Integrity

| File | Size | SHA-256 |
|---|---:|---|
| `ChromaLux-0.6.5.dmg` | 199,458,755 bytes | `7db9f6425b8d65694d12fbc0753ee5b687f6d95c09e46eb598c83d2def2e4408` |
| `Guide-utilisateur-ChromaLux-FR.pdf` | 1,918,493 bytes | `22617bcef657ee340b4f13935b8b4ebaa315a63c331b278d46e7927435c1ec1a` |
| `ChromaLux-User-Guide-EN.pdf` | 1,743,707 bytes | `70a3341c29f0b742d2a8405b270a537aba8d3552d9212acd431bd61cec1ecb57` |

Local verification:

```sh
shasum -a 256 ChromaLux-0.6.5.dmg
```

## Français

### Présentation

ChromaLux Studio est un atelier photographique natif pour macOS consacré au
développement RAW, au grading couleur et aux flux de production en studio. Il
réunit dans un même environnement l’import sécurisé, la gestion de sessions,
le développement non destructif, les corrections locales, la capture
connectée, la calibration des boîtiers, les compositions créatives, les outils
de contrôle colorimétrique, l’export et l’impression.

L’application s’appuie sur une chaîne modulaire accélérée par Metal et sur
deux fondations complémentaires :

- **HALO**, le moteur de dématriçage RAW recommandé ;
- **SCS**, le domaine de traitement tonal et chromatique partagé par les
  outils de grading, les masques perceptuels, les pipettes et les scopes.

ChromaLux 0.6.5 est une version alpha pour Mac Apple Silicon sous
macOS 26 Tahoe ou une version plus récente.

### HALO — reconstruire le RAW

**HALO — Holonomic Adaptive Local Operator** reconstruit une image RVB
camera-linear à partir de la mosaïque enregistrée par le capteur. Ce
dématriçage adaptatif aux gradients est exécuté sur le GPU avec Metal.

HALO analyse plusieurs directions locales — horizontale, verticale et
diagonale — ainsi que la structure du voisinage pour arbitrer la
reconstruction. Il est conçu pour préserver les contours et les détails tout
en limitant les fausses couleurs, les effets de fermeture éclair et les
artefacts d’interpolation. La reconstruction reste dans la chaîne linéaire de
scène afin de conserver la précision et la réserve de hautes lumières pour les
étapes suivantes.

Le chemin Bayer est le moteur recommandé de ChromaLux ; AMaZE reste disponible
comme solution alternative. Les capteurs Fujifilm X-Trans sont eux aussi
traités par HALO dans la chaîne de production 0.6.5, même si cette route reste
signalée comme expérimentale. Les fichiers Sigma/Foveon X3F utilisent une
route de décodage dédiée.

HALO n’est pas un profil caméra : il reconstruit les pixels du RAW. La
transformation colorimétrique propre au boîtier est appliquée ensuite par un
profil SCSP compatible ou par la matrice caméra du décodeur.

### Ce que SCS apporte au grading

SCS organise le traitement autour d’une séparation explicite entre la
luminance ou densité de scène et la géométrie chromatique. Sa métrique de
couleur permet de raisonner en distances et en directions chromatiques plutôt
qu’en simples écarts arithmétiques dans les canaux RVB.

Concrètement, cette architecture apporte :

- des réglages de lumière, de densité, de chroma et de direction chromatique
  plus indépendants ;
- des corrections couleur conçues pour limiter les variations de luminosité
  ou de teinte non désirées ;
- des protections explicites des neutres, des tons chair et des couleurs
  voisines ;
- des sélections locales fondées sur une distance chromatique, utilisables
  dans les points locaux, les masques paramétriques et l’affinage des contours
  IA ;
- dans les chaînes v2/v3, un traitement référencé scène qui conserve la réserve
  du RAW jusqu’à la transformation d’affichage ;
- une continuité entre Courbes, Niveaux, Éditeur de couleurs, Balance des
  couleurs, Harmonie, pipettes, Vectorscope et Parade ;
- des lectures séparées des domaines **Scène**, **Affichage** et
  **Livraison**, afin de savoir à quelle étape une valeur est mesurée.

SCS ne remplace ni le profil SCSP du boîtier, ni le profil ColorSync de
l’écran, ni le profil ICC de livraison. Chaque élément possède une
responsabilité distincte :

```text
RAW → décodage et dématriçage sélectionné → balance des blancs
    → profil SCSP ou matrice du décodeur
    → grading linéaire de scène et outils SCS
        ├─ présentation : espace du viewer → épreuve ICC éventuelle
        │                 → profil ColorSync de l’écran
        └─ livraison : profil ICC de recette → fichier ou impression
```

Le SCS permet de corriger plus précisément la lumière, la densité ou la couleur
tout en limitant les effets indésirables sur les autres composantes de l’image.
Les mêmes repères sont utilisés par les outils de grading, les masques, les
pipettes et les scopes, ce qui rend le résultat plus prévisible et plus facile
à contrôler.

### Fonctionnalités de ChromaLux 0.6.5

#### Sessions, import et protection des originaux

- Création et ouverture de sessions `.clssession`, avec dossiers
  **Captures**, **Sélections**, **Sortie** et **Corbeille**.
- Navigation dans la session ou dans des dossiers externes accessibles.
- Import depuis une carte, un volume ou un dossier, avec aperçu et sélection
  avant copie.
- Destination, structure de dossiers, renommage par jetons et métadonnées
  communes configurables.
- Copie de sauvegarde facultative, vérification de destination ou comparaison
  source/destination par somme de contrôle et émission ASC-MHL.
- Ouverture automatique du parcours d’import lorsqu’une carte photo reconnue
  est insérée.
- Effacement après import désactivé par défaut et protégé par la vérification
  stricte, ASC-MHL, la saisie de `ERASE` et une confirmation finale.

#### Photothèque, sélection et traitement non destructif

- Navigateur par dossiers, grille de vignettes, photo focalisée et sélection
  multiple.
- Déplacement ou copie des photos avec leurs fichiers associés.
- Affichage partagé jusqu’à douze photos ou variantes, loupe Focus, second
  viewer, zoom, panoramique, ajustement à la fenêtre et vue 100 %.
- Notes de zéro à cinq étoiles, statuts retenu/rejeté/non marqué et plusieurs
  labels couleur synchronisables avec les tags Finder.
- Tri par nom, date, note, couleur, ISO, taille, ouverture, vitesse ou
  extension ; filtre rapide dans le dossier courant.
- Variantes indépendantes par original, création, clonage, suppression et
  duplication explicite vers une variante HDR v3.
- Ajustements, calques, masques, mesures et compositions conservés dans des
  sidecars versionnés ; miniatures JXL distinctes par variante.
- Annulation/rétablissement, comparaison temporaire avant/après, copie
  complète ou partielle des ajustements et synchronisation optionnelle sur une
  sélection.
- Copie et collage des métadonnées, traitement et export par lot.

#### Développement RAW et détail

- Développement explicite des routes prises en charge pour `NEF`, `CR2`,
  `CR3`, `ARW`, `DNG`, `RAF`, `IIQ` et `X3F`.
- Dématriçage HALO pour Bayer et X-Trans, AMaZE comme alternative Bayer, plus
  une route dédiée Sigma/Foveon. La route HALO X-Trans reste signalée comme
  expérimentale.
- Intentions **Photographie** et **Science**.
- Balance des blancs, température, teinte, neutralisation automatique et
  correction de dominante.
- Exposition, équilibre, luminosité, contraste, saturation et vibrance.
- Récupération des hautes lumières par canal et spatiale, contrôle des ombres,
  hautes lumières, blancs, noirs, densité et blanchiment.
- Tone mapping, détail des ombres et contrôles de plage dynamique étendue.
- Netteté, rayon, texture, clarté, structure et réduction du bruit de luminance
  et de couleur.
- Profils de bruit boîtier/ISO lorsqu’ils existent, avec repli générique.
- Réglage des primaires, de leur pureté et des caractéristiques de base.

#### Profils caméra et gestion colorimétrique

- Chaînes colorimétriques versionnées v1, v2 et v3 pour préserver les rendus
  historiques sans migration silencieuse.
- Transport linéaire haute précision dans les chaînes v2/v3.
- Profils boîtier SCSP v1 historique, SCSP v2 strict et SCSP v3 DCP
  bi-illuminant borné à un Camera RGB trois canaux et une apparence SDR.
- Choix libre du profil caméra par variante, repli explicite vers la matrice du
  décodeur et défaut séparé par modèle de boîtier.
- Import local et diagnostic de profils DCP ou SCSP compatibles, avec
  provenance et empreintes. Les profils importés restent expérimentaux, ne
  sont jamais activés automatiquement et leurs droits d’utilisation doivent
  être vérifiés.
- Résolution DCP bi-illuminant par interpolation en température réciproque,
  bornage aux illuminants extrêmes et refus des structures incohérentes. Il
  n’existe pas d’axe de teinte indépendant ; `ReductionMatrix` est analysée et
  validée, mais n’est pas appliquée par le solveur trois canaux actuel.
- Profil Nikon D850 mesuré sur charte qualifié pour l’association automatique ;
  les autres boîtiers utilisent la matrice du décodeur tant qu’aucun profil
  n’est explicitement choisi ou qualifié.
- Espaces de rendu du viewer sRGB, Adobe RGB (1998) et Display P3.
- Identification du profil ColorSync de l’écran actif.
- Profils distincts pour le boîtier, le viewer, l’écran, la livraison,
  l’épreuvage et l’impression.
- Épreuvage écran, simulation du blanc papier, avertissement de gamut et
  compression perceptuelle de sortie **OKLab v1** expérimentale pour les cubes
  RVB sRGB, Adobe RGB et Display P3.
- Aperçu EDR en Display P3 linéaire étendu sur une variante v3 et un écran
  Apple compatible, avec repli automatique en SDR ; l’épreuvage écran force
  également le SDR.

#### Grading tonal et couleur

- Courbes SCS de luminance, saturation et canaux RVB.
- Courbe de densité Fisher, Niveaux SCS et Égaliseur de tons par plages.
- Suppression de la brume avec profondeur et protection des tons chauds.
- Correction des franges violettes et vertes.
- Balance des couleurs globale et séparée pour ombres, tons moyens et hautes
  lumières, avec protection des tons chair.
- Éditeur de couleurs par familles chromatiques et réglages avancés des tons
  chair.
- Harmonie couleur à partir de teintes maîtresses.
- Noir et blanc avec sensibilités par couleur et virage partiel.
- Vignette créative.
- Histogramme et avertissements d’écrêtage disponibles pendant le grading.

#### Calques et masques

- Calques d’ajustement activables, opacité réglable et modes de fusion
  **Normal**, **Produit**, **Superposition**, **Incrustation**,
  **Lumière tamisée**, **Différence**, **Addition** et **Soustraction**.
- Corrections locales pour Exposition, Balance des blancs, Éditeur de
  couleurs, Balance des couleurs, Primaires, Courbe Fisher, Noir et blanc et
  Plage dynamique étendue.
- Masques au pinceau, dégradés linéaires et radiaux, point local SCS, masque
  SCS paramétrique et masque chromatique multi-prélèvements.
- Peinture en ajout ou soustraction, progressivité, flou global du masque,
  poignées et prévisualisation en direct dans le viewer.
- Vues **Image**, **Superposition** et matte **Noir et blanc**.
- Sélection de personnes et d’arrière-plan avec Apple Vision.
- Sélection d’objets avec SAM 2.1 Small lorsque ses modèles Core ML sont
  installés.
- Affinage facultatif des contours Apple Vision ou SAM par couleur et
  luminance SCS.

#### Géométrie, optique et cadrage

- Rotation, redressement et recadrage avec ratios, verrouillage et guides de
  composition.
- Correction de perspective verticale, horizontale ou complète à l’aide de
  guides éditables.
- Correction de distorsion géométrique à partir des données DNG compatibles ou
  d’un profil Lensfun exact.
- Réglage manuel de la distorsion et du centre optique.
- Vignettage et correction des franges dans des modules manuels distincts ;
  ChromaLux n’applique pas automatiquement le vignettage Lensfun ni
  l’aberration chromatique latérale.
- Persistance non destructive de la géométrie dans chaque variante.

#### Simulateur de film

- Catalogue de courbes film, modes Standard et Expert.
- Intensité, contraste, luminosité, pied, épaule, gamma et densité maximale.
- Développement Pull, Normal ou Push lorsque les données du film le permettent.
- Noir et blanc, virage partiel et halation réglable.
- Grain argentique avec taille physique, polydispersité, graine, diffusion,
  chroma et diaphonie.
- Simulation de tirage argentique avec choix de papier, diffusion, chaleur,
  texture et virage.
- Vignette de finition, préréglages et transfert du rendu sur une série.

Les noms de films couvrent à la fois des réponses mesurées et des modèles
initiaux ; ils ne constituent pas une certification d’un couple précis
film–laboratoire–scanner.

#### Composition créative

- **Double exposition** à partir de deux images, avec équilibrage des
  expositions, transformation de la seconde source, modes négatif, contact ou
  positif, grain et création d’un DNG linéaire dérivé.
- **Raw Mockup** pour placer, transformer et grader des sources RAW ou rendues
  dans une composition.
- Liaison de Raw Mockup au flux Live View partagé.
- **ChromaKey** sur une photo ou une composition : prélèvements de fond et de
  protection, construction et finition du matte, despill, récupération du
  premier plan et remplacement par une image RAW ou rendue.
- Vues de diagnostic Composite, Matte, premier plan récupéré, estimation du
  fond, différence de despill et confiance.

Le remplacement ChromaKey directement dans le Live View n’est pas une fonction
de la 0.6.5.

Le DNG issu de Double exposition utilise actuellement une matrice générique
sRGB-D65 ; la récupération du profil réel de chaque boîtier reste différée.

#### Capture connectée Studio

- Fournisseurs Nikon, Canon et Fujifilm intégrés, avec gPhoto2 comme repli
  manuel explicite.
- Détection, connexion automatique ou manuelle et fournisseur Mock de test.
- Dossier et modèle de nom pour les prochaines captures.
- Déclenchement, téléchargement transactionnel, sélection automatique et
  gestion des couples RAW + JPEG.
- Contrôles caméra selon les capacités annoncées par le fournisseur.
- Live View dans une fenêtre dédiée ou dans le viewer principal.
- Flux Live View unique partagé entre la fenêtre, le viewer et Raw Mockup.
- Export d’un diagnostic JSON sans contenu photographique.

La compatibilité dépend du modèle, du firmware, du pilote ou SDK constructeur
et des capacités annoncées par chaque appareil.

#### Calibration Studio

- Mesure d’une charte ColorChecker 24 plages avec placement manuel des quatre
  coins et trois jeux de références.
- Génération d’un profil SCSP v2 strict, résidus de calibration et refus des
  profils incohérents.
- Installation gérée et sélection du profil sur la variante active.
- Éditeur de profil pour dériver un rendu à partir d’une base, de tonalité, de
  corrections couleur et de noir et blanc.
- Import et diagnostic des profils DCP/SCSP.

Le choix D65 du parcours CC24 n’est pas pleinement opérationnel en 0.6.5 : le
générateur utilise actuellement la référence D50.

#### Pipettes, scopes et activité Science

- Pipette ponctuelle, carrée ou circulaire dans les domaines **Scène**,
  **Affichage** ou **Livraison**.
- Valeurs RVB flottantes, Lab D50, HSV, luminance, EV et nits lorsque les
  métadonnées nécessaires sont disponibles.
- Jusqu’à huit sondes verrouillées avec référence et comparaisons RVB, HSV,
  DeltaE00, EV ou nits.
- Histogramme en domaine scène ou de sortie.
- Vectorscope SCS avec axe des tons chair, cibles, alertes de saturation, zoom
  et options d’affichage.
- Parade SCS en modes RGBL+Sat, RGB, luminance, saturation ou composantes SCS,
  avec cibles Legal, Rec.709, P3 et Rec.2020.
- Traces de référence ; export de la Parade SCS en PNG et de ses densités en
  CSV ; export PNG de l’incrustation Fausses couleurs.
- Fausses couleurs de luminance, température et saturation ; inspection des
  canaux.
- Grille, réticule, barre d’échelle, guides, mesures de distance, surface et
  angle, texte et annotations persistantes.
- Lecture et rotation des fichiers FITS, aperçus Astro/Microscope et cartes de
  contraste PT spécialisées.

#### Métadonnées et automatisation locale

- Lecture des métadonnées EXIF ; édition des champs descriptifs XMP/IPTC.
- Informations de session, crédits de production, titre, description,
  mots-clés, copyright et crédits.
- Ajout ou retrait de mots-clés par lot, copie et collage des métadonnées.
- Modèles de nom utilisant les informations de session, la séquence et les
  données caméra disponibles.
- Suggestions locales de métadonnées avec Apple Vision et Ollama lorsqu’un
  modèle vision compatible est installé.
- Validation manuelle ou acceptation automatique des propositions IA.

#### Brief, planches contact, export et impression

- Brief de production avec rectangles, ovales, dessin libre, pinceau, texte,
  gomme, priorités, couleurs, références et calques séparés.
- Dictée et transcription locale en français ou en anglais après installation
  du modèle vocal.
- Rapports de brief en PDF, PNG, Markdown ou JSON et inclusion optionnelle dans
  les exports.
- Jusqu’à quatre fenêtres de planche contact, formats A4, A3, 4K, 8K ou
  personnalisés, grilles, marges, logo, légendes et document natif modifiable.
- Plusieurs recettes d’export dans une même opération, sur l’image focalisée
  ou une sélection.
- Export `JPEG` 8 bits, `TIFF` 8/16 bits sans compression, LZW ou ZIP,
  `PNG` 8/16 bits et `PSD` 8 bits pour les briefs en calques.
- Profils ICC de livraison sRGB, Adobe RGB (1998) et ProPhoto RGB, renommage,
  sous-dossiers, redimensionnement et politiques de collision.
- Impression d’une image, d’une grille répétée ou d’une planche contact, avec
  gestion couleur ChromaLux ou imprimante.
- Profils d’impression RVB ou CMJN, épreuvage écran, avertissement de gamut,
  compensation du point noir, simulation du papier, netteté de sortie, grain
  de tirage et légendes.
- Export de page en TIFF, JPEG ou PNG et impression via le dialogue macOS.

#### Interface, personnalisation et interfaces développeur

- Activités spécialisées : Photothèque, Calibration, Capture connectée,
  Ajustements, Simulateur de film, Science, Planche contact, Métadonnées,
  Brief, Exporter et Impression.
- Modules réorganisables ou détachables, onglets personnalisables, une ou deux
  colonnes, plusieurs instances et activités personnalisées.
- Préréglages de modules, copie/collage, contournement, réinitialisation et
  dispositions complètes enregistrables.
- Barre d’outils personnalisable, thèmes clair/sombre/système, densités
  Compacte/Confortable et interface française/anglaise.
- Éditeur de raccourcis clavier, navigation clavier et informations
  d’accessibilité sur les principaux parcours.
- Architecture modulaire de plug-ins ; ABI native v9, manifestes, IPC tether
  v1, formats de graphe et outils développeur documentés pour les usages
  internes ou partenaires.

### Limites et fonctions expérimentales

Les éléments suivants sont présents mais ne doivent pas être considérés comme
entièrement qualifiés pour tous les flux :

- route HALO X-Trans, utilisée par la chaîne de production mais encore marquée
  expérimentale ;
- compression perceptuelle du gamut **OKLab v1**, limitée aux sorties sRGB,
  Adobe RGB et Display P3 et sans validation psychovisuelle achevée sur un
  corpus d’images réelles ; elle ne remplace pas le rendu d’un profil
  d’impression ICC arbitraire ;
- aperçu EDR, limité aux variantes v3 et aux écrans Apple compatibles ; le
  blanc papier et le pic créatif sont bornés par la réserve déclarée de l’écran
  et ne constituent pas des mesures instrumentales ; la qualification physique
  et multi-écrans reste manuelle ;
- profils DCP ou SCSP importés avant leur qualification visuelle et la
  vérification de leurs droits d’utilisation ;
- sélections Apple Vision et SAM, métadonnées IA Ollama et transcription
  locale, dépendantes du système ou des modèles installés ;
- cartes Contraste PT, aperçus Astro/Microscope et profil de ligne encore
  partiels ;
- interopérabilité des sorties Brief PSD/TIFF à vérifier dans l’application
  cible ;
- capture connectée dépendante du boîtier, du firmware et du fournisseur ;
- VoiceOver non rejoué de bout en bout, poignées graphiques des masques et de
  Perspective nécessitant un pointeur, et navigation clavier Perspective
  encore partielle.

Les exports et l’impression restent SDR : ChromaLux 0.6.5 n’encode pas encore
de fichiers HDR10/PQ ou HLG ni leurs métadonnées de livraison. Les panneaux
Calibration **Scientifique**, **Détecteur** et **Géométrique** sont des espaces
réservés ; l’Éditeur de nœuds v1 est en lecture seule ; Clone/Heal sont
désactivés.

Les interfaces plug-in sont destinées aux usages internes, privés ou
partenaires : un plug-in doit être signé avec le même Team ID et approuvé
explicitement par empreinte. Il n’existe pas encore de marketplace public ni
de SDK tiers installé par le DMG, et les outils en ligne de commande
développeur ne font pas partie de l’application distribuée.

### Formats et configuration

Le scanner d’import reconnaît davantage de formats que le viewer n’en
développe. La détection ou la copie d’un fichier ne garantit donc pas son
décodage.

- Formats repérés par l’import : `NEF`, `NRW`, `CR2`, `CR3`, `CRW`, `ARW`,
  `SR2`, `SRF`, `RAF`, `RW2`, `ORF`, `PEF`, `DNG`, `IIQ`, `3FR`, `X3F`,
  `GPR`, `JPG`, `JPEG`, `HEIC`, `HEIF`.
- RAW effectivement routés par le viewer : `NEF`, `CR2`, `CR3`, `ARW`, `DNG`,
  `RAF`, `IIQ`, `X3F`, sous réserve de l’encodage pris en charge.
- Images rendues visibles et indexées : `JPG`, `JPEG`, `PNG`, `TIF`, `TIFF`,
  `HEIC`, `HEIF`, `WEBP`. HEIC/HEIF et WebP dépendent des décodeurs présents
  dans le paquet ou dans macOS.
- Images scientifiques : `FITS`, `FIT`, `FTS`.
- Configuration : Mac Apple Silicon, macOS 26 Tahoe ou version plus récente,
  GPU Metal compatible.
- Version et build : `0.6.5`, configuration `Release`.
- Expiration de cette version bêta : 1er décembre 2026.

### Contenu du paquet

- [ChromaLux-0.6.5.dmg](https://github.com/Igrekess/ChromaLux/releases/download/v0.6.5/ChromaLux-0.6.5.dmg) — application macOS signée,
  notarisée et agrafée ;
- [Guide utilisateur en français](documentation_utilisateur/Guide-utilisateur-ChromaLux-FR.pdf) ;
- [User Guide in English](documentation_utilisateur/ChromaLux-User-Guide-EN.pdf) ;
- [Documentation API en français et en anglais](documentation_api/README.md).

### Installation

1. Ouvrir `ChromaLux-0.6.5.dmg`.
2. Faire glisser `ChromaLux.app` vers le dossier `Applications`.
3. Lancer ChromaLux depuis `Applications`.

### Intégrité

| Fichier | Taille | SHA-256 |
|---|---:|---|
| `ChromaLux-0.6.5.dmg` | 199 458 755 octets | `7db9f6425b8d65694d12fbc0753ee5b687f6d95c09e46eb598c83d2def2e4408` |
| `Guide-utilisateur-ChromaLux-FR.pdf` | 1 918 493 octets | `22617bcef657ee340b4f13935b8b4ebaa315a63c331b278d46e7927435c1ec1a` |
| `ChromaLux-User-Guide-EN.pdf` | 1 743 707 octets | `70a3341c29f0b742d2a8405b270a537aba8d3552d9212acd431bd61cec1ecb57` |

Vérification locale :

```sh
shasum -a 256 ChromaLux-0.6.5.dmg
```
