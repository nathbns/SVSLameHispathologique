# Le format SVS : comprendre les fichiers de lames histopathologiques

## 1. Le format TIFF — Fondation du SVS

### 1.1 Qu'est-ce que le TIFF ?

**TIFF** (*Tagged Image File Format*) est un format d'image créé en 1986 par Aldus (devenu Adobe). C'est un format **conteneur** flexible : il ne définit pas un seul type d'image, mais un **cadre structurel** dans lequel on peut encoder des images de différentes manières.

Un fichier TIFF est organisé en **IFD** (*Image File Directory*), chacun décrivant une image avec des **tags** (métadonnées) :

| Tag | Nom | Description |
|-----|-----|-------------|
| 256 | `ImageWidth` | Largeur en pixels |
| 257 | `ImageLength` | Hauteur en pixels |
| 259 | `BitPerSample` | Bits par échantillon (8, 16…) |
| 262 | `PhotometricInterpretation` | Interprétation couleur (RGB, YCbCr…) |
| 273 | `StripOffsets` | Position des données dans le fichier |
| 277 | `SamplesPerPixel` | Canaux (1=NVG, 3=RGB, 4=RGBA) |
| 278 | `RowsPerStrip` | Lignes par bande |
| 282 | `XResolution` | Résolution horizontale |
| 283 | `YResolution` | Résolution verticale |
| 322 | `TileWidth` | Largeur des tuiles |
| 323 | `TileLength` | Hauteur des tuiles |
| 324 | `TileOffsets` | Position des tuiles dans le fichier |
| 325 | `TileByteCounts` | Taille en octets de chaque tuile |
| 339 | `SampleFormat` | Format des échantillons |

### 1.2 Deux modes d'organisation

#### Mode *Stripped* (bandes)
L'image est découpée en **bandes horizontales** contiguës. Adapté aux petites images. C'est le mode par défaut de la plupart des images TIFF.

```
┌──────────────────────┐
│     Bande 0          │  ← Strip 0
├──────────────────────┤
│     Bande 1          │  ← Strip 1
├──────────────────────┤
│     Bande 2          │  ← Strip 2
├──────────────────────┤
│     ...              │
└──────────────────────┘
```

#### Mode *Tiled* (tuiles)
L'image est découpée en **tuiles carrées** (ex. 256×256). Adapté aux très grandes images car il permet un **accès aléatoire** rapide : pour afficher une zone, on ne charge que les tuiles correspondantes.

```
┌────┬────┬────┬────┐
│ T0 │ T1 │ T2 │ T3 │  ← Tuiles indépendantes
├────┼────┼────┼────┤     chacune compressée séparément
│ T4 │ T5 │ T6 │ T7 │
├────┼────┼────┼────┤
│ T8 │ T9 │T10 │T11 │
└────┴────┴────┴────┘
```

### 1.3 IFD multiples = Pyramide

Un fichier TIFF peut contenir **plusieurs IFD** (directories), chacun décrivant une image différente. Dans le contexte des WSI, on utilise cette propriété pour créer une **pyramide multi-résolution** :

```
IFD 0 : 100000 × 80000  (niveau 0, pleine résolution, 40×)
IFD 1 :  25000 × 20000  (niveau 1, ×4 downsample, 10×)
IFD 2 :   6250 ×  5000  (niveau 2, ×16 downsample, 2.5×)
IFD 3 :   3125 ×  2500  (niveau 3, ×32 downsample, vignette)
```

Chaque niveau est une version sous-échantillonnée du niveau précédent, permettant un **zoom fluide** dans les visualiseurs : on charge le niveau adapt à l'écran.

### 1.4 Compression dans le TIFF

Le tag `Compression` (259) indique le codec utilisé **au sein de chaque bande ou tuile** :

| Valeur | Codec | Description |
|--------|-------|-------------|
| 1 | Aucune | Données brutes (non compressé) |
| 5 | LZW | Compression sans perte Lempel-Ziv-Welch |
| 8 | Deflate | Compression sans perte (zlib) |
| 7 | JPEG | JPEG baseline (avec perte) |
| 33003 | JPEG 2000 YCbCr | Ondelettes, codec Aperio |
| 33005 | JPEG 2000 RGB | Ondelettes, codec Aperio |

**Important** : la compression s'applique **tuile par tuile**, pas au fichier global. C'est pourquoi compresser un SVS avec ZIP/GZIP ne sert à rien : les tuiles JPEG sont déjà compressées et ne se compressent plus.

---

## 2. Le format SVS en détail

### 2.1 Qu'est-ce qu'un fichier SVS ?

**SVS** (*ScanScope Virtual Slide*) est le format propriétaire des scanners **Aperio** (acquise par Leica). En pratique, un fichier `.svs` est un **TIFF pyramidal tuilé** avec des conventions spécifiques Aperio.

C'est le format le plus répandu en anatomopathologie numérique, notamment dans les jeux de données publics comme **TCGA** (The Cancer Genome Atlas).

### 2.2 Détection

OpenSlide identifie un fichier comme Aperio SVS si :
1. Le fichier est un TIFF valide
2. La première image est **tuilée** (tiled)
3. Le tag `ImageDescription` commence par `"Aperio"`

### 2.3 Structure interne d'un fichier SVS

Un fichier SVS contient plusieurs images TIFF (IFD) dans cet ordre :

```
┌─────────────────────────────────────────────────┐
│ IFD 0 : Image pleine résolution (niveau 0)     │
│         - Tuilée (240×240 ou 256×256 px)       │
│         - Compression JPEG ou JPEG 2000        │
│         - Ex: 105575 × 87609 pixels            │
├─────────────────────────────────────────────────┤
│ IFD 1 : Vignette (thumbnail)                    │
│         - Mode stripped (pas tuilée)            │
│         - ~1024 × 768 pixels                   │
├─────────────────────────────────────────────────┤
│ IFD 2 : Niveau pyramidal ×4                    │
│         - Tuilée, même taille de tuiles        │
│         - Même compression que IFD 0           │
├─────────────────────────────────────────────────┤
│ IFD 3 : Niveau pyramidal ×16                   │
├─────────────────────────────────────────────────┤
│ IFD 4 : Niveau pyramidal ×32                   │
├─────────────────────────────────────────────────┤
│ IFD optionnel : Image label (étiquette)         │
│         - Mode stripped                         │
├─────────────────────────────────────────────────┤
│ IFD optionnel : Image macro (vue d'ensemble)    │
│         - Mode stripped                         │
└─────────────────────────────────────────────────┘
```

**Exemple concret** (tiré de nos expériences avec TCGA) :
- Niveau 0 : 105575 × 87609 pixels (downsample ×1)
- Niveau 1 : 26393 × 21902 pixels (downsample ×4)
- Niveau 2 : 6598 × 5475 pixels (downsample ×16)
- Niveau 3 : 3299 × 2737 pixels (downsample ×32)
- Tuiles : 240 × 240 pixels
- Taille du fichier : **1.1 Go**

### 2.4 Métadonnées Aperio

Le tag `ImageDescription` du premier IFD contient une chaîne structurée, par exemple :

```
Aperio Image Library v11.2.1
46000x32914 [0,100 46000x32814] (240x240) JPEG/RGB Q=70
|AppMag = 40|StripeWidth = 992|ScanScope ID = SS6000|
Filename = CMU-1.svs|Date = 12/29/09|Time = 09:59:15|
User = b414003c-95c6-48b0-9369-0090d2eb482e|
MPP = 0.4990
```

On y trouve :
- **Dimensions** de l'image et de chaque niveau
- **AppMag** : grossissement (40×)
- **MPP** : microns par pixel (résolution physique, ex. 0.4990 µm/px)
- **StripeWidth** : largeur des bandes/tuiles
- **Qualité JPEG** (ici Q=70)
- **Compression** : JPEG/RGB ou JPEG 2000

Toutes les paires clé=valeur après `|` sont accessibles via OpenSlide avec le préfixe `aperio.`.

### 2.5 Tuilage et accès aléatoire

La caractéristique clé du SVS est le **tuilage**. Pour une image de 105575 × 87609 avec des tuiles de 240×240 :

- Nombre de tuiles en largeur : ⌈105575 / 240⌉ = **440**
- Nombre de tuiles en hauteur : ⌈87609 / 240⌉ = **365**
- Total au niveau 0 : **440 × 365 ≈ 160 600 tuiles**

Chaque tuile est compressée **indépendamment** en JPEG. Pour afficher une petite zone de l'image, le visualiseur charge uniquement les tuiles concernées — pas besoin de charger les 1.1 Go en mémoire.

### 2.6 Pourquoi les fichiers SVS sont si volumineux ?

1. **Taille de l'image** : une lame à 40× peut atteindre 80 000 × 90 000 pixels en RGB 24 bits
2. **Pas de redondance inter-tuiles** : chaque tuile est compressée indépendamment, pas de référence croisée
3. **Qualité conservatrice** : le JPEG original est généralement à Q=70-80, ce qui laisse peu de marge de recompression
4. **Fond blanc inutile** : une grande partie de la surface est du fond blanc (pas de tissu), mais il est encodé comme n'importe quelle zone
5. **Multiples niveaux** : la pyramide ajoute ~33% de données supplémentaires

### 2.7 Les deux types de compression dans un SVS

Selon le tag `Compression`, les tuiles d'un SVS peuvent être encodées en :

- **JPEG baseline** (compression = 7) : le plus courant, utilisé par la majorité des scanners Aperio
- **JPEG 2000** (compression = 33003 pour YCbCr, 33005 pour RGB) : utilisé par certains scanners Aperio, meilleure qualité mais décodage plus lent

### 2.8 Lecture et manipulation avec Python

**OpenSlide** — bibliothèque de référence pour lire les SVS :

```python
import openslide

slide = openslide.OpenSlide("fichier.svs")

# Dimensions de chaque niveau
for i in range(slide.level_count):
    print(f"Niveau {i} : {slide.level_dimensions[i]} "
          f"(downsample x{slide.level_downsamples[i]:.1f})")

# Lire une région (position, niveau, taille)
region = slide.read_region((x, y), level, (w, h))

# Propriétés
print(slide.properties)         # métadonnées Aperio
print(slide.level_dimensions)   # dimensions de chaque niveau

slide.close()
```

**pyvips** — pour la conversion et la recompression :

```python
import pyvips

img = pyvips.Image.new_from_file("fichier.svs")

img.tiffsave(
    "sortie.tiff",
    tile=True, tile_width=256, tile_height=256,
    pyramid=True,
    compression="jpeg", Q=75
)
```

### 2.9 Autres formats WSI rencontrés en histopathologie

| Format | Scanneur | Structure | Compression | Mono-fichier ? |
|--------|-----------|-----------|-------------|----------------|
| `.svs` | Aperio/Leica | TIFF pyramidal tuilé | JPEG / JPEG 2000 | Oui |
| `.ndpi` | Hamamatsu | TIFF-like non standard | JPEG | Oui |
| `.vms` | Hamamatsu | Multi-fichier INI + JPEG | JPEG | Non |
| `.tiff` | Philips | TIFF pyramidal tuilé | JPEG 2000 | Oui |
| `.mrxs` | 3D-Histech | Dossier + tuiles | JPEG / J2K | Non |

Tous ces formats partagent le même principe : une **pyramide multi-résolution** découpée en tuiles, compressées individuellement. OpenSlide les lit tous avec la même API.

---

## 3. Pipeline de compression d'un SVS

### 3.1 Pourquoi la compression n'est pas triviale

Un fichier SVS est **déjà compressé** (JPEG par tuile). Les approches naïves échouent :

- **ZIP/GZIP sur le fichier entier** : les tuiles JPEG sont déjà compressées, le gain est négligeable (<2%)
- **Recompression JPEG à qualité plus élevée** : JPEG Q=90 sur un original Q≈70 produit un fichier **3× plus gros** sans gain de qualité (démontré dans nos expériences : 1133.7 Mo → 3303.4 Mo)
- **Compression sans perte (Deflate, LZW, PNG)** : en décompressant puis recompressant sans perte, le fichier est **plus gros** car on stocke les pixels reconstruits sans les artefacts de compression originale

Pour réduire réellement la taille, il faut **recompresser les tuiles avec perte** à un quality level inférieur, ou changer de stratégie.

### 3.2 Pipeline de recompression

```
fichier.svs (1.1 Go)
     │
     ▼
┌─────────────────────────────────────┐
│ 1. Lecture avec OpenSlide / pyvips │
│    - Décodage des tuiles JPEG      │
│    - Reconstruction de l'image     │
└─────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────┐
│ 2. Optionnel : segmentation tissu  │
│    / fond                           │
│    - Seuillage couleur              │
│    - Remplacement du fond par du   │
│      blanc pur                      │
└─────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────┐
│ 3. Recompression                    │
│    - Choix du codec (JPEG, J2K,    │
│      JPEG XL, WebP, AVIF…)         │
│    - Choix du quality level        │
│    - Tuilage (256×256 typique)      │
└─────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────┐
│ 4. Sauvegarde                       │
│    - TIFF pyramidal (compatible     │
│      OpenSlide)                     │
│    - Reconstruction de la pyramide │
│      en 4 niveaux                   │
└─────────────────────────────────────┘
     │
     ▼
fichier.tiff compressé (0.6 – 1.0 Go)
```

### 3.3 Résultats expérimentaux

Résultats obtenus sur la lame `TCGA-3M-AB46` (origine : 1133.7 Mo, tissu : 31.3% de la surface) :

#### Compression JPEG uniforme (sans masque)

| Méthode | Qualité | Taille (Mo) | Ratio | Remarque |
|---------|---------|-------------|-------|----------|
| Original SVS | Q≈70-80 | 1133.7 | 1.00 | Référence |
| JPEG Q=50 | 50 | 696.8 | 1.63 | Perte visible |
| JPEG Q=75 | 75 | 922.6 | 1.23 | Bon compromis |
| JPEG Q=90 | 90 | 3303.4 | 0.34 | Plus gros (recompression !) |

#### Compression adaptative (fond blanc remplacé puis JPEG)

| Méthode | Qualité | Taille (Mo) | Ratio | Gain |
|---------|---------|-------------|-------|------|
| Original SVS | Q≈70-80 | 1133.7 | 1.00 | — |
| Adaptatif Q=50 | 50 | 598.3 | 1.89 | 47.2% |
| Adaptatif Q=75 | 75 | 774.8 | 1.46 | 31.7% |
| Adaptatif Q=85 | 85 | 972.9 | 1.17 | 14.2% |
| Adaptatif Q=95 | 95 | 3438.3 | 0.33 | −203.3% |

**Observations clés** :

1. **Recompression à qualité égale ou supérieure = fichier plus gros** : l'original est déjà JPEG Q≈70-80. Le recompresser à Q=85 donne déjà un fichier plus gros que l'original. À Q=95, c'est catastrophique.

2. **Le fond blanc est un gouffre** : avec seulement 31.3% de tissu, 68.7% de la surface est du fond qui ne contient aucune information diagnostique. Le remplacer par du blanc pur (255, 255, 255) permet à JPEG de compresser ces tuiles extrêmement efficacement.

3. **L'approche adaptative** (remplacement du fond + JPEG) offre les meilleurs résultats à qualité égale : à Q=75, on passe de 922.6 Mo (classique) à 774.8 Mo (adaptatif), soit 16% de gain supplémentaire.

### 3.4 Méthode de segmentation du fond

La segmentation tissu/fond utilisée dans nos expériences repose sur trois critères appliqués sur la vignette basse résolution :

```python
# 1. Pixels saturés (proches du blanc)
mask_white = (R >= 200) & (G >= 200) & (B >= 200)

# 2. Zones grises (R, G, B très proches = pas de tissu)
rgb_max = image.max(axis=2)
rgb_min = image.min(axis=2)
mask_gray = (rgb_max - rgb_min) <= 15

# 3. Le fond = blanc OU gris
mask_background = mask_white | mask_gray
mask_tissue = ~mask_background

# 4. Nettoyage morphologique (supprimer les petits objets < 500 px)
# + remplissage des trous dans le tissu
```

Le masque obtenu est ensuite redimensionné à la résolution complète et appliqué avec `pyvips.ifthenelse()` : si le pixel est du tissu, on garde l'original ; si c'est du fond, on remplace par du blanc pur.

### 3.5 Schéma décisionnel pour le choix de compression

```
                    Besoin clinique ?
                         │
              ┌──────────┼──────────┐
              │          │          │
         Diagnostic   Recherche   Archivage
              │          │          │
              ▼          ▼          ▼
      JPEG Q=75-85   Variable   Sans perte
      (ou adaptatif)  │      (JPEG-LS, J2K
              │        │       reversible)
              │    ┌───┴───┐
              │    │       │
              │  DL/IA   Visuel
              │    │       │
              │    ▼       ▼
              │  Q=50-70  Q=70-85
              │  (robuste) (subjectif)
              │
              └─── Le fond blanc peut être
                   remplacé par du blanc pur
                   pour gain supplémentaire
```

---

## 4. Note sur DICOM

**DICOM** (*Digital Imaging and Communications in Medicine*) est le standard de référence pour l'imagerie médicale en milieu hospitalier. Ce n'est pas un format d'image comme le SVS, mais un **protocole complet** couvrant le format de fichier, le transport réseau, et le modèle de données (Patient → Study → Series → Instance).

Pour les WSI, DICOM définit le **Supplement 145** (*VL Whole Slide Microscopy Image Storage*) qui structure une lame comme une série d'instances, chacune contenant un niveau pyramidal avec des frames tuilées. Les codecs supportés sont JPEG, JPEG 2000 et HTJ2K (JPEG XL et AVIF ne sont pas encore des syntaxes de transfert DICOM officielles).

La conversion SVS → DICOM est possible avec des outils comme **wsic** ou **WSI2DICOM**, mais elle introduit une complexité significative (génération d'UIDs, mapping des métadonnées, risque de double compression). Dans le cadre de ce stage, nous restons dans l'écosystème TIFF/SVS, mais il est important de garder à l'esprit que l'interopérabilité clinique à long terme passe par DICOM.

---

## 5. Points clés à retenir

1. **SVS = TIFF spécialisé** : un fichier `.svs` est un TIFF pyramidal tuilé avec des métadonnées Aperio. Ce n'est pas un format indépendant.

2. **La compression s'applique aux tuiles** : chaque tuile est compressée individuellement en JPEG ou JPEG 2000. Compresser le fichier `.svs` avec ZIP ne fonctionne pas.

3. **La pyramide est essentielle** : les niveaux de résolution multiples sont nécessaires pour le zoom fluide. Toute recompression doit reconstruire cette pyramide.

4. **Le fond blanc est un gouffre** : sur une lame typique, 60-70% de la surface est du fond blanc sans information diagnostique. Le remplacer par du blanc pur avant compression est un gain facile et sans impact clinique.

5. **Recompresser à qualité supérieure = contre-productif** : l'original est déjà JPEG Q≈70-80. Recompresser à Q=90 ou plus produit un fichier plus gros sans regain de qualité réelle.

6. **L'approche adaptative est la plus efficace** : segmenter tissu/fond, remplacer le fond par du blanc pur, puis compresser le tissu à qualité préservée offre le meilleur compromis taille/qualité.