# Plan de présentation — Soutenance de début de stage

---

## Slide 1 — Page de titre

**Compression des lames histopathologiques numérisées au format SVS**
État de l'art et plan de travail

- Stagiaire : [Prénom Nom]
- Tuteur : [Nom du tuteur]
- Structure : HCL — Hospices Civils de Lyon
- Date : [Date]

---

## Slide 2 — Le contexte : l'anatomopathologie numérique

**De la lame de verre au fichier informatique**

- L'anatomopathologie : discipline médicale qui diagnostique les maladies par l'examen microscopique de tissus
- Historiquement : le pathologiste regarde la lame au microscope optique
- Depuis ~2010 : les **scanners de lames** (*whole slide scanners*) numérisent la lame entière à fort grossissement (20×, 40×)
- Résultat : une **Whole Slide Image** (WSI) — une image géante consultation à distance, archivage, IA

**Au HCL :**
- Scanners Aperio/Leica → fichiers au format **.svs**
- Des centaines de lames scannées par jour → accumulation massive de données

---

## Slide 3 — Le problème : des fichiers énormes

**Une lame numérisée = un fichier gigantesque**

- Une lame à 40× peut atteindre **80 000 × 90 000 pixels** en RGB
- Résultat : **1 à 3 Go par fichier**
- Un service d'anatomopathologie scanne des centaines de lames par jour
- Stockage annuel : **plusieurs dizaines de térabytes**
- Coûts : serveurs, sauvegardes, refroidissement, maintenance
- Impact écologique : consommation énergétique directe

**Le défi** : comment réduire la taille de ces fichiers **sans perdre d'information diagnostique** ?

---

## Slide 4 — Pourquoi c'est difficile à compresser ?

**Les fichiers SVS sont déjà compressés**

- Chaque tuile est encodée en JPEG ou JPEG 2000
- Compresser le fichier `.svs` avec ZIP/GZIP → **gain < 2%**
- La compression sans perte (PNG, Deflate) → **fichier plus gros** (pixels reconstruits > JPEG original)
- Recompresser en JPEG à qualité supérieure (Q=90) → **fichier 3× plus gros** sans regain de qualité

**La seule option** : recompresser les tuiles avec un codec plus efficace ou à qualité inférieure, en préservant l'information diagnostique.

---

## Slide 5 — Le format SVS : qu'est-ce que c'est ?

**SVS = ScanScope Virtual Slide (Aperio / Leica)**

- Format propriétaire des scanners Aperio
- En réalité : un **fichier TIFF pyramidal tuilé** avec des métadonnées Aperio
- Extension : `.svs`
- Le format le plus répandu en anatomo numérique (jeux TCGA, scanners HCL)

**Un fichier .svs contient :**
- Niveau 0 : image pleine résolution (40×)
- Niveaux 1-3 : versions sous-échantillonnées (10×, 2.5×) pour le zoom fluide
- Vignette, image de l'étiquette, vue macro

---

## Slide 6 — Structure interne : la pyramide

**Principe de la pyramide multi-résolution**

```
Niveau 0 : 105 575 × 87 609 px  (×1)   ← résolution maximale
Niveau 1 :  26 393 × 21 902 px  (×4)   ← zoom intermédiaire
Niveau 2 :   6 598 ×  5 475 px  (×16)  ← navigation
Niveau 3 :   3 299 ×  2 737 px  (×32)  ← vignette
```

- Chaque niveau est une version réduite du précédent
- Le visualiseur charge uniquement le niveau adapt à l'écran
- Pas besoin de charger 1 Go en mémoire → **accès aléatoire par tuiles**

---

## Slide 7 — Structure interne : le tuilage

**Chaque niveau est découpé en tuiles indépendantes**

- Taille typique : 240×240 ou 256×256 pixels par tuile
- Chaque tuile est compressée **indépendamment** en JPEG
- Pour notre lame : ⌈105575/240⌉ × ⌈87609/240⌉ ≈ **160 600 tuiles** au niveau 0

**Conséquence directe** :
- Pour afficher une zone → on ne charge que les tuiles correspondantes
- Aucune redondance entre tuiles → chaque JPEG est autonome
- Compresser le `.svs` en ZIP → ça ne sert à rien

---

## Slide 8 — Le fond blanc : un gouffre de stockage

**Une observation clé : la majeure partie de l'image est du vide**

- Sur une lame typique, **60-70% de la surface** est du fond blanc (pas de tissu)
- Ce fond ne contient **aucune information diagnostique**
- Mais il est encodé comme n'importe quelle zone de tissu
- Sur notre lame test : seulement **31.3% de tissu**, 68.7% de fond

→ C'est une **opportunité majeure** pour la compression : traiter le fond différemment du tissu

---

## Slide 9 — Objectifs du stage

**Objectif principal** : Évaluer et comparer les méthodes de compression des images SVS en anatomopathologie, avec un focus sur le compromis taille/qualité/préservation diagnostique.

**Objectifs spécifiques :**

1. **Caractériser** un échantillon de lames SVS (taille, structure, % de tissu)
2. **Cartographier** l'état de l'art des codecs de compression applicables aux WSI
3. **Benchmarker** les codecs (JPEG, JPEG 2000, JPEG XL, WebP, AVIF) à différents niveaux de qualité
4. **Implémenter et tester** une approche adaptative (compression différenciée tissu/fond)
5. **Évaluer** l'impact de la compression sur la qualité visuelle et, si possible, sur les performances de deep learning
6. **Estimer** l'impact écologique du gain de stockage (To économisés → kWh → kgCO₂e)

---

## Slide 10 — État de l'art : codecs existants

| Codec | Perte ? | Ratio typique | Vitesse | Usage médical |
|---|---|---|---|---|
| **JPEG** | Oui | Moyen | Très rapide | Universel (déjà dans SVS) |
| **JPEG 2000** | Oui / Non | Bon | Lent | Standard Aperio/Philips |
| **JPEG XL** | Oui / Non | Très bon | Rapide | Faible adoption |
| **WebP** | Oui / Non | Bon | Rapide | Faible |
| **AVIF** | Oui | Très bon | Lent (enc.) | Quasi nul |
| **JPEG-LS** | Non | Faible | Rapide | Archives légales |

**Ce que l'état de l'art nous dit :**
- JPEG 2000 est le standard médical de facto (Aperio, Philips)
- JPEG XL promet les meilleurs ratio mais n'est pas adopté en clinique
- La compression adaptative (traiter fond vs tissu) est prometteuse mais peu standardisée
- Les études récentes montrent que le deep learning est robuste à la compression JPEG jusqu'à un certain seuil (Tellez et al., 2019)

---

## Slide 11 — Approche adaptative : l'idée centrale

**Principe : traiter différemment le tissu et le fond**

```
Image originale → Segmentation tissu/fond
                        │
               ┌────────┴────────┐
               │                 │
          Zone tissu         Zone fond
          (conserver la     (remplacer par
           qualité)         blanc pur 255,255,255)
               │                 │
               └────────┬────────┘
                        │
                   Recompression
                        │
                        ▼
                  Image compressée
                  (taille réduite)
```

**Pourquoi ça fonctionne :**
- Le blanc pur se compresse extrêmement bien en JPEG (tuiles quasi identiques)
- Le tissu est compressé à qualité préservée
- L'information diagnostique est intacte

---

## Slide 12 — Méthodologie prévue

**Plan de travail en 5 étapes :**

| Étape | Description | Livrable |
|---|---|---|
| 1. Analyse des lames | Caractériser taille, structure, % tissu sur échantillon HCL | Notebook d'analyse |
| 2. Benchmark des codecs | Tester JPEG, J2K, JPEG XL, WebP, AVIF à plusieurs qualités | Tableaux de résultats taille/SSIM/PSNR/temps |
| 3. Compression adaptative | Implémenter segmentation tissu/fond + recompression différenciée | Notebook + pipeline |
| 4. Évaluation qualité | SSIM, PSNR, évaluation visuelle, éventuellement impact DL | Métriques comparatives |
| 5. Impact écologique | Estimer To économisés, kWh | Calculs et graphiques |

---

## Slide 13 — Premiers résultats exploratoires

**Tests sur une lame TCGA (1133.7 Mo, 31.3% tissu)**

| Méthode | Qualité | Taille (Mo) | Ratio |
|---|---|---|---|
| Original SVS | Q≈70-80 | 1133.7 | 1.00 |
| JPEG Q=75 (classique) | 75 | 922.6 | 1.23 |
| JPEG Q=50 (classique) | 50 | 696.8 | 1.63 |
| Adaptatif Q=75 (fond blanc) | 75 | 774.8 | 1.46 |
| Adaptatif Q=50 (fond blanc) | 50 | 598.3 | 1.89 |

**Première observation** : l'approche adaptative amène ~16% de gain supplémentaire par rapport au JPEG classique à même qualité.

---

## Slide 14 — Outils et environnement de travail

**Bibliothèques Python utilisées :**

- **OpenSlide** : lecture des fichiers SVS (accès par tuiles, métadonnées)
- **pyvips** : conversion et recompression efficace des WSI
- **scikit-image** : métriques SSIM, PSNR
- **NumPy / Matplotlib** : manipulation d'images et visualisation
- **scipy.ndimage** : segmentation morphologique (nettoyage du masque tissu)

**Pipeline actuel :**
- Lecture SVS → segmentation tissu/fond sur vignette → application du masque → recompression JPEG/TIFF pyramidal
- Tout en notebooks Jupyter pour la reproductibilité

---

## Slide 15 — Ce qui reste à faire

**Travaux prévus pour les prochaines semaines :**

1. **Étendre le benchmark** à plusieurs lames HCL (différents % tissu, colorations H&E/IHC)
2. **Tester JPEG XL et AVIF** comme codecs alternatifs
3. **Métriques avancées** : MS-SSIM, LPIPS (perceptuel), temps d'encodage/décodage
4. **Impact sur le deep learning** : comparer les performances d'un modèle de segmentation sur images compressées vs originales
5. **Calcul de l'impact écologique** : projection sur l'ensemble du service HCL
6. **Vérifier la compatibilité** avec les visualiseurs du HCL (QuPath, OpenSlide, etc.)

---

## Slide 16 — Résumé

1. **Le problème est réel** : les fichiers SVS sont énormes (1-3 Go) et déjà compressés → les approches naïves ne fonctionnent pas

2. **Le SVS est un TIFF pyramidal tuilé** : comprendre cette structure est indispensable pour savoir où et comment compresser

3. **Le fond blanc est une opportunité** : 60-70% de la surface ne contient aucune information diagnostique

4. **L'approche adaptative est prometteuse** : traiter le tissu et le fond différemment donne les meilleurs résultats

5. **L'enjeu est double** : réduire les coûts de stockage ET minimiser l'impact écologique, tout en préservant la qualité diagnostique
