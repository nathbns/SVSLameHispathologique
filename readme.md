# Compression d'Images SVS/DICOM

## Contexte
Les Whole Slide Images (WSI) utilisées en anatomopathologie numérique atteignent 
des résolutions extrêmes (jusqu'à 100 000 × 100 000 pixels), générant des fichiers 
de plusieurs gigaoctets. 

## Objectifs du stage
1. Analyser les formats et propriétés des images WSI (.svs)
2. Benchmarker les codecs de compression classiques (JPEG, JPEG2000, JPEG XL, etc.)
3. Concevoir une méthode de compression adaptative aux caractéristiques des lames
4. Évaluer l'impact qualitatif des compressions sur le diagnostic
5. Implémenter un modèle de Learned Image Compression inspiré de CLERIC

### Carnet de bords à la semaine (mise à jours chaque semaine):

**Semaine 1**:
- Phase de compréhension du sujet, des problématiques et plus précisément du fonctionnement des images WSI (Whole Slide Image)
- Recherche de l'état de l'art, codec utilisé pour les formats de compression, le SOTA actuelle, et ce qui est encore à l'état de recherche
- Premières expérimentation, en réalisant un benchmark des formats de compression les plus simples. Afin de me rendre compte des différences et des faiblesses de chaque méthodes de compression basique.

**Semaine 2 (en cours)**:

## Structure du projet
### `expe_compression_classique/` - Fondations 
- **01** — Analyse des lames SVS (formats, métadonnées, structure pyramide)
- **02** — Benchmark comparatif des codecs (JPEG, JPEG XL, AVIF, WebP, HEIC...)
- **03** — Compression adaptative par seuillage du fond blanc
- **04** — Évaluation qualité (PSNR, SSIM) et impact visuel

### `compression_adaptative/` — Implémentation détaillée
- Notebook détaillé de la méthode de compression adaptative
- Avec tests sur des images

### `cleric/` — Learned Image Compression
- Plan et architecture du modèle CLERIC-inspired
- Scripts d'entraînement et d'évaluation
- pas encore d'implémentation

### Données
- **91 lames SVS TCGA** 
- OBJ: 
    - Patches d'entraînement : ~30 000 tuiles 256×256 extraites des zones tissulaires

## Références
- CLERIC (Lee et al., 2025) — Learned Image Compression for Digital Pathology
- Ballé et al. (2018) — Variational Image Compression with a Scale Hyperprior
- [Tes autres sources]
---