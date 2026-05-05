# Plan Notebook 05 — CLERIC-Inspired Learned Image Compression

> **Date** : 2025-04-30  
> **Contexte** : Suite des notebooks 01–04 (analyse, benchmark codecs, compression adaptative, évaluation qualité).  
> **Objectif** : Implémenter et entraîner un modèle de Learned Image Compression (LIC) inspiré de CLERIC (2025), adapté à nos contraintes de stage (2 mois, A100 P2CHPD, pas de code officiel disponible).

---

## 1. Architecture retenue : "CLERIC-inspired simplifié"

### 1.1 Vue d'ensemble

```
Input patch (256×256×3)
    ↓
[Learnable Lifting Scheme]
    → x_LL (H/2×W/2×3) + x_CH (H/2×W/2×9)
    ↓
[Downsample x → x_1/2] (H/2×W/2×3)
    ↓
Concaténer :
    x̃_L = [x_LL, x_1/2]  → (H/2×W/2×6)
    x̃_H = [x_CH, x_1/2]  → (H/2×W/2×12)
    ↓
[Encoder Branch L]          [Encoder Branch H]
ResBlocks + downsample      ResBlocks + downsample
    ↓2                           ↓2
    ↓2                           ↓2
    ↓                           ↓
Latent y_L (H/16×W/16×N)    Latent y_H (H/16×W/16×N)
    ↓
[Concaténer y = [y_L, y_H]] → (H/16×W/16×2N)
    ↓
[Hyperprior Balle2018-style]
    z = h_e(y) → hyper-encode
    p(y|z) = Gaussian(µ=0, σ=h_d(ẑ))
    ↓
Quantification + Entropy coding
    ↓
[Decoder unique gd,LH]
ResBlocks + R2B(t=2) + upsample
    ↓
(H/2×W/2×12)
    ↓
[Split] → x̂_LL, x̂_LH, x̂_HL, x̂_HH
    ↓
[Inverse Lifting Scheme]
    ↓
Reconstruction x̂ (256×256×3)

Loss = R(ŷ) + R(ẑ) + λ·MSE(x, x̂)
```

### 1.2 Simplifications vs CLERIC original

| Composant | CLERIC original | Notre version |
|-----------|----------------|---------------|
| Lifting scheme | CDF 9/7 + CNN learnable P/U | **CDF 9/7 + CNN Conv1×1 learnable** (simplifié mais réversible) |
| Encoder/Decoder blocks | DRB (DCNv2) + R2B | **R2B uniquement** (pas de DCNv2 — trop complexe sans code de référence) |
| Entropy model | MEM++ (multi-référence) | **Hyperprior Balle2018** (standard, bien documenté) |
| Branches encodeur | Deux branches L/H | **Deux branches L/H** (gardé — cœur de l'idée CLERIC) |
| Decoder | Unique gd,LH | **Unique gd,LH** (gardé — plus efficace) |

### 1.3 Hyperparamètres

| Paramètre | Valeur | Justification |
|-----------|--------|---------------|
| N (channels branches encodeur) | **128** | Réduit vs CLERIC (192) pour entraînement plus rapide |
| M (channels latent) | **192** | Réduit vs CLERIC (320) pour accélérer |
| R2B recursions t | **2** | Identique au papier |
| λ (trade-off Rate/Distortion) | **{0.001, 0.005, 0.01, 0.02, 0.05}** | À ajuster selon les résultats |
| Patch train | **256×256** | Identique au papier |
| Patch test | **512×512** | Identique au papier |
| Batch size | **8–16** | Selon mémoire A100 (40 Go → batch 16 facile) |
| Optimizer | **Adam (β1=0.9, β2=0.999)** | Identique au papier |
| LR initiale | **1×10⁻⁴** | Identique au papier |
| LR step 50 | **1×10⁻⁵** | Identique au papier |
| LR step 100 | **1×10⁻⁶** | Identique au papier |
| Max steps | **200–300** | Early stopping patience = 30 |
| Loss distortion | **MSE** | Identique au papier |

---

## 2. Données

### 2.1 Source
- **91 SVS TCGA** dans `/Users/nath/Desktop/data_stage/` (~148 Go)
- Pas de dataset public supplémentaire (décision validée)

### 2.2 Extraction
- **Cible** : ~30 000 patches de 256×256 (vs 21 000 dans le papier CLERIC)
- **Filtre** : uniquement les patches avec >30 % de tissu (fond blanc rejeté)
- **Split** : train 90 % / val 10 %
- **Format** : sauvegarde en `.npy` ou `.pt`

---

## 3. Livrables

| Fichier | Type | Description |
|---------|------|-------------|
| `05a_extraction_patches.ipynb` | Notebook local | Extraction des patches depuis les 91 SVS, filtrage, sauvegarde dataset |
| `05b_cleric_inspired_model.py` | Script Python | Architecture complète (lifting, encoder, hyperprior, decoder, R2B) |
| `05c_train.py` | Script Python | Boucle d'entraînement avec loss RD, logging, checkpointing |
| `05d_evaluate.ipynb` | Notebook | Courbes RD (bpp vs PSNR/MS-SSIM), comparaison avec JPEG/adaptatif |
| `05e_inference_wsi.py` | Script Python (optionnel) | Inférence tuile-par-tuile sur WSI complète pour démo |
| `train_slurm.sh` | Script SLURM | Job batch pour P2CHPD (A100, env conda) |

---

## 4. Calendrier (7 semaines)

### Semaine 1 — Extraction de données (LOCAL)
- Ouvrir les 91 SVS avec OpenSlide, vignettes Level 3
- Segmenter tissu/fond pour chaque lame
- Extraire ~30k patches 256×256 (zones tissu >30 %)
- Split train/val, sauvegarder

### Semaine 2 — Implémentation du modèle (LOCAL)
- Jour 1 : Lifting scheme (forward + inverse), vérifier réversibilité
- Jour 2 : Encoder branches L/H avec ResBlocks + R2B
- Jour 3 : Hyperprior + loss rate-distortion
- Jour 4 : Decoder + inverse lifting, assemblage complet
- Jour 5 : Test forward/backward sur CPU (batch=2)

### Semaine 3 — Transfert & Setup P2CHPD
- Transférer données + code sur cluster
- Créer env conda (PyTorch **dernière version**, CUDA, CompressAI, OpenSlide)
- Test interactif GPU (`salloc --gres=gpu:1 -p gpu-10g`)
- Écrire + tester `train_slurm.sh`

### Semaine 4 — Entraînement complet
- Entraîner pour chaque λ (5 runs)
- Monitoring loss train/val
- Sauvegarder meilleurs checkpoints

### Semaine 5 — Évaluation & Comparaison
- Évaluer chaque checkpoint sur test (512×512)
- Calculer bpp, PSNR, MS-SSIM
- Courbes RD vs JPEG / méthode adaptative (notebook 03)
- Notebook `05d_evaluate.ipynb`

### Semaine 6 — Inférence WSI & Polish (OPTIONNEL)
- Script inférence tuile-par-tuile sur WSI test
- Reconstruction + visualisation côte-à-côte
- Nettoyage code, documentation

### Semaine 7 — Rédaction & Soutenance
- Rédiger section méthodologie (architecture CLERIC-inspired)
- Rédiger résultats (courbes RD, gains vs baseline)
- Préparer slides soutenance

---

## 5. Décisions validées

| # | Question | Réponse |
|---|----------|---------|
| 1 | Nombre de patches | **~30 000** |
| 2 | Installer CompressAI | **Oui** (pour utilitaires d'évaluation) |
| 3 | Version PyTorch | **Dernière disponible** (2.x+) |
| 4 | Dataset public supplémentaire | **Non** (91 SVS TCGA suffisent) |
| 5 | Nom du modèle | **À définir plus tard** |

---

## 6. Infrastructure P2CHPD

### Accès
- Cluster : `p2chpd-login3.univ-lyon1.fr`
- GPU disponibles : A100 / RTX 3090 (partition `gpu-10g`)
- Stockage : `/home_nfs/` (sauvegardé) ou `/scratch/` (rapide, non sauvegardé)
- Soumission : SLURM (`sbatch train_slurm.sh`)

### Environnement conda (à créer)
```bash
conda create -n cleric python=3.11
conda install pytorch torchvision pytorch-cuda=12.1 -c pytorch -c nvidia
pip install compressai
pip install openslide-python scikit-image matplotlib pandas
```

---

## 7. Risques & Mitigations

| Risque | Probabilité | Mitigation |
|--------|-------------|------------|
| Lifting scheme non réversible | Moyenne | Tester MSE < 1e-6 avant toute autre étape |
| Modèle ne converge pas | Moyenne | Commencer λ=0.01, LR basse, gradient clipping |
| P2CHPD indisponible / queue longue | Moyenne | Tester dès S3, utiliser `squeue` pour planifier |
| Résultats moins bons que CLERIC (pas de DCNv2/MEM++) | Élevée | L'accepter, argumenter "simplification réaliste pour un stage" |
| Pas assez de tissu dans certaines lames | Faible | Vérifier distribution % tissu dès S1 |

---

## 8. Prochaine session — Ordre du jour

1. Valider le présent plan (ce fichier)
2. Commencer `05a_extraction_patches.ipynb` :
   - Chargement SVS avec OpenSlide
   - Extraction vignette + segmentation tissu/fond
   - Extraction patches 256×256 filtrés
   - Sauvegarde dataset
3. Décider du nom du modèle
4. Vérifier compte P2CHPD actif + clé SSH configurée

---

## 9. Références clés

- **CLERIC** : Lee et al. 2025, arXiv:2503.23862 — *Learned Image Compression and Restoration for Digital Pathology*
- **Balle 2018** : arXiv:1802.01436 — *Variational image compression with a scale hyperprior* (base du hyperprior)
- **CompressAI** : https://github.com/InterDigitalInc/CompressAI — utilitaires d'évaluation RD

---

*Plan établi le 30/04/2025. Prochaine étape : extraction des patches (Semaine 1).*
