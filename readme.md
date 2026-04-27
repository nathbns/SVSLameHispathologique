Quelles sont les differentes methodes aujourd'hui pour compresser les images DICOM/SVS ? (C'est la meme chose ?)
POur reduire au max la taille des données et pouvoir les visualiser quand meme ? Je veux faire l'etat de l'art de ça.




JPEG2000 ? JPEG XL?

## Liens article / site
1. https://www.johnsnowlabs.com/what-to-know-before-de-identifying-whole-slide-images-wsi/
2. https://dicom.nema.org/dicom/dicomwsi/
3. https://dicom.nema.org/medical/dicom/current/output/chtml/part05/chapter_9.html
4. https://arxiv.org/pdf/2503.18074 (meilleur ressource pour l'instant)
5. https://arxiv.org/html/2503.23862v1

## Liens gh
1. https://github.com/John-P/wsic
2. https://github.com/smujiang/WSI2DICOM 

## format des fichier donnée: svs
https://openslide.org/formats/aperio/


### Etat de l'art 

#### 1 Compression generique

- **JPEG 2000** reste le standard de facto en pathologie (Aperio, Philips). Deux profils : irreversible (9/7) et reversible (5/3).
- **JPEG XL** (ISO/IEC 18181) promet des gains significatifs : meilleure qualite subjective, support sans perte, progressive. Peu de validateurs medicaux a ce jour.
- **HEVC-MSP** (Main Still Picture) : etudie pour remplacer JPEG, mais problemes de licences.

#### 2 Compression pour WSI

- **WSI-specific tiling** : les images sont decoupees en tuiles (256x256 ou 512x512). Chaque tuile est compressee independamment. Cela permet l'acces aleatoire rapide.
- **High Throughput JPEG 2000 (HTJ2K)** : version acceleree de J2K (ISO/IEC 15444-15), jusqu'a 10x plus rapide en decodage. Implemente dans `OpenJPH`.

#### 3 Travaux recents 

1. **Comparaison codecs sur WSI** :
   - _Janowczyk et al._ ont montre que JPEG 2000 Q=70 (Aperio) est un bon compromis, mais que la perte impacte la quantification nucleaire.
   - _Senaras et al._ : comparaison JPEG vs J2K vs JPEG XL sur la classification tissulaire. JPEG XL domine a fort taux.

2. **Compression pour le Deep Learning (task-based)** :
   - L'idee : au lieu de maximiser le PSNR/SSIM, minimiser l'impact sur la performance d'un reseau de neurones.
   - _Tellez et al. (2019)_ : "Quantifying the effects of data compression and augmentation on deep learning" → le DL est robuste a la compression JPEG jusqu'a un certain seuil.
   - _Zhao et al. (2021)_ : optimisation du taux de compression par region (ROI vs fond).

3. **Compression sans perte / numeriquement exacte** :
   - Obligatoire pour certaines applications legales (expertises, essais cliniques).
   - JPEG-LS, JPEG 2000 reversible, JPEG XL sans perte.
   - Taux faibles (~2-3x), mais garantie d'integrite.

4. **Compression neuronale (Learned Image Compression)** :
   - Auto-encodeurs variationnels entraines pour la compression (ex : Ballé et al., 2018).
   - **Probleme** : pas encore standardises, necessitent un decodeur specifique (modele ML), difficilement compatibles DICOM/PACS.
   - Quelques travaux sur les patchs histologiques (pas les WSI entieres).
