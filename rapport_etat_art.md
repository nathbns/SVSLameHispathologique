# État de l'art : Compression des images en anatomopathologie numérique

## 1. Le format SVS

### 1.1 Définition et origine

Le format **SVS** (*ScanScope Virtual Slide*) est le format propriétaire historique des scanners **Aperio / Leica**. En pratique, un fichier `.svs` est un conteneur **TIFF pyramidal** spécialisé pour les images de lames entières (*Whole Slide Images*, WSI). Ces fichiers mesurent couramment **plusieurs gigaoctets**
### 1.2 Structure interne

Un SVS n'est pas une image géante linéaire. Il est structuré en plusieurs niveaux :

- **Pyramide multi-résolution** : le niveau 0 est la pleine résolution (ex. 40x), suivi de niveaux sous-échantillonnés (4x, 16x, 64x plus petits) pour permettre un affichage fluide.
- **Tuilage** : chaque niveau est découpé en tuiles carrées (souvent 256×256 ou 512×512 pixels). Seules les tuiles visibles sont chargées en mémoire.
- **Images associées** : vignette (*thumbnail*), vue macro, étiquette de la lame.
- **Métadonnées** : fabricant, grossissement, résolution physique (microns/pixel), etc.

### 1.3 Pourquoi les fichiers sont si volumineux ?

Une lame numérisée à 40x peut atteindre **80 000 × 90 000 pixels** (niveau 0). Même compressé en JPEG dans chaque tuile, l'ensemble reste énorme car :
- l'image est en **RGB 24 bits** ;
- la compression JPEG d'origine est conservative (qualité élevée) pour préserver le diagnostic ;
- une grande partie de la surface est du **fond blanc** sans information utile, mais celui-ci reste encodé.

### 1.4 Conséquence pour la compression

Compresser un SVS avec ZIP ou GZIP donne peu de gain car les tuiles sont **déjà compressées**. Pour réduire réellement la taille, il faut donc **recompresser les tuiles** avec des paramètres plus agressifs, changer de codec, ou supprimer les zones inutiles.

---

## 2. Méthodes de compression

### 2.1 JPEG (baseline)

Le JPEG est le codec le plus répandu. Il repose sur la DCT (transformée en cosinus discrète) et une quantification adaptée.

- **Avantage** : universel, rapide, compatible avec tous les visualiseurs.
- **Inconvénient** : artefacts de bloc visibles à fort taux, les pertes sont irréversibles.

### 2.2 JPEG 2000

JPEG 2000 repose sur la **transformée en ondelettes** au lieu de la DCT. Il offre une scalabilité naturelle en résolution et qualité, et supporte le sans perte (5/3) et le avec perte (9/7).

- **Avantage** : meilleur ratio qualité/taille que JPEG à fort taux.
- **Inconvénient** : encodage/décodage plus lent.

### 2.3 JPEG XL

JPEG XL est le codec le plus récent de la famille des JPEG. Il promet des gains significatifs par rapport à JPEG et JPEG 2000, avec un support sans perte et une progressiveité améliorée.

- **Avantage** : très bon ratio qualité/taille, décodage rapide, support du sans perte.
- **Inconvénient** : adoption clinique quasi nulle à ce jour, peu de visualiseurs médicaux le supportent.
- **Article de ref** : Alakuijala et al., *JPEG XL: Image Coding for the Future*, 2021 (disponible sur arXiv et sites du JPEG committee). [Lien](https://arxiv.org/abs/1908.03565)

### 2.4 WebP

WebP (Google) combine compression prédictive et transformée en cosinus. Il propose des modes avec et sans perte.

- **Avantage** : meilleur ratio que JPEG dans de nombreux cas, open source.
- **Inconvénient** : rarement supporté dans les PACS et visualiseurs anatomopathologiques, différents de JPEG.
- **Article de référence** : Google, *WebP Compression Study*, documentation technique. [Lien](https://developers.google.com/speed/webp/docs/compression)

### 2.5 AVIF

AVIF repose sur le codec vidéo AV1. Il offre des taux de compression très agressifs.

- **Avantage** : parmi les meilleurs ratios qualité/taille actuels.
- **Inconvénient** : temps d'encodage très long, support logiciel encore limité.

### 2.6 Compression sans perte (PNG, TIFF Deflate, JPEG-LS)

Ces méthodes garantissent une restitution exacte des pixels.

- **Avantage** : aucune perte d'information.
- **Inconvénient** : gain limité sur des images déjà compressées en JPEG, ne résout pas le problème de stockage.

### 2.7 Compression adaptative et suppression du fond

Plutôt qu'uniforme, la compression peut être adaptée au contenu : zones de tissu = haute qualité, fond blanc = forte compression ou suppression. Codipilly et al. proposent de détecter le tissu, de l'extraire et de le reconditionner dans un fichier plus compact.

- **Avantage** : réduction très forte (jusqu'à ×7 rapportée) quand la lame contient beaucoup de blanc.
- **Inconvénient** : la géométrie originale est modifiée ; moins adapté à l'archivage clinique strict.
- **Article de référence** : Codipilly et al., *Optimizing Storage and Computational Efficiency: An Efficient Algorithm for Whole Slide Image Size Reduction*, Mayo Clinic Proceedings: Digital Health, 2023. [Lien](https://pmc.ncbi.nlm.nih.gov/articles/PMC11975739/)

### 2.8 Compression par représentation neuronale implicite (INR)

Une approche émergée depuis 2020 consiste à remplacer l'image elle-même par une **fonction continue** apprise par un réseau de neurones. Au lieu de stocker des pixels, on stocke les poids d'un petit MLP qui, à partir de coordonnées `(x, y)`, prédit la couleur `RGB`. Cette famille de méthodes est appelée **Implicit Neural Representation** (INR) ou **Neural Radiance Fields** (NeRF) dans le domaine de la vision.

**Principe de fonctionnement** :
1. On entraîne un réseau de neurones sur **une seule image** (overfitting) jusqu'à ce qu'il reproduise fidèlement chaque pixel à partir de ses coordonnées normalisées.
2. La "compression" consiste à jeter l'image originale et à ne conserver que les **poids du réseau**.
3. Pour reconstruire l'image, on évalue le réseau sur une grille de coordonnées.

**CINR** (Lee et al., MICCAI 2024) est une adaptation spécifique à la pathologie numérique. Il combine :
- un **HashGrid encoding** (`tinycudann`) pour capturer les hautes fréquences des textures histologiques,
- un **MLP** classique pour la prédiction globale,
- un **CNN auxiliaire** pour préserver les variations spatiales locales fines (bords cellulaires, noyaux).

- **Avantage** : ratio de compression extrême (×100 à ×1000 potentiel), zoom infini natif (pas besoin de pyramide)
- **Inconvénient** : un modèle différent par image (pas de généralisation), entraînement lent (overfitting par WSI), décodage lent. L'implémentation originale dépend de `tinycudann`, une librairie C++/CUDA (beaucoup plus difficilement apréhendable que du python)
- **Article de référence** : Lee et al., *CINR: Convolutional Implicit Neural Representation of pathology whole-slide images*, MICCAI 2024. [Lien](https://github.com/pnu-amilab/CINR)

### 2.9 Compression apprise (Learned Image Compression — CLERIC)

Les méthodes de **compression apprise** (ou *Learned Image Compression*, LIC) remplacent les codecs classiques (DCT, ondelettes) par des **auto-encodeurs entraînés** sur de larges corpus d'images. L'image est passée dans un encodeur qui produit une représentation latente compacte, elle-même compressée par un codeur entropique. Un décodeur reconstruit ensuite l'image à partir de cette représentation.

**CLERIC** (Lee et al., 2025) est une méthode de compression apprise spécifiquement conçue pour la pathologie numérique. Elle repose sur plusieurs innovations :
- **Lifting scheme** : décomposition de l'image en bandes basses fréquences (L) et hautes fréquences (H), similaire à la transformée en ondelettes mais avec des filtres apprenables.
- **Branches d'encodage séparées L/H** : deux encodeurs traitent indépendamment les basses et hautes fréquences, car elles ont des distributions statistiques différentes.
- **Hyperprior** : un second réseau prédit la distribution des latents pour améliorer le codage entropique.
- **Décodeur unique** : une seule branche de décodage reconstruit l'image à partir des latents L et H fusionnés.
- **R2B (Recursive Residual Blocks)** : blocs récursifs pour améliorer la reconstruction.

- **Avantage** : un seul modèle généralisé pour toutes les images, meilleur ratio qualité/taille que JPEG à fort taux.
- **Inconvénient** : entraînement lourd (nécessite un dataset de milliers de patches et un GPU puissant), décodage plus lent que JPEG, implémentation complexe (lifting scheme, hyperprior, codeur entropique). 
- **Article de référence** : Lee et al., *Learned Image Compression and Restoration for Digital Pathology*, 2025. [Lien arXiv](https://arxiv.org/abs/2503.23862)

---