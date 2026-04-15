# 📝 Tâche 2 — Exploration et Analyse des Données MIMIC3d

## Objectif
Charger et analyser le dataset MIMIC3d pour comprendre sa structure,
identifier les problèmes de qualité et dégager les insights métier.

## Dataset
- **Source :** https://www.kaggle.com/datasets/drscarlat/mimic3d
- **Fichier :** `data/raw/mimic3d.csv`
- **Taille :** 58 976 séjours hospitaliers × 28 colonnes

---

## Étape 1 — Observer les données brutes (terminal)

```bash
# Nombre de lignes
wc -l data/raw/mimic3d.csv
# Résultat : 58977 (58976 lignes + 1 entête)

# Voir les colonnes
head -1 data/raw/mimic3d.csv

# Voir les 3 premières lignes
head -3 data/raw/mimic3d.csv
```

---

## Étape 2 — Chargement dans Jupyter

```python
import pandas as pd
import numpy as np

df = pd.read_csv('/home/jovyan/data/raw/mimic3d.csv')

print(f"Lignes    : {df.shape[0]:,}")
print(f"Colonnes  : {df.shape[1]}")
print(df.columns.tolist())
```

**Résultat :**
```
Lignes    : 58,976
Colonnes  : 28
['hadm_id', 'gender', 'age', 'LOSdays', 'admit_type', 'admit_location',
 'AdmitDiagnosis', 'insurance', 'religion', 'marital_status', 'ethnicity',
 'NumCallouts', 'NumDiagnosis', 'NumProcs', 'AdmitProcedure', 'NumCPTevents',
 'NumInput', 'NumLabs', 'NumMicroLabs', 'NumNotes', 'NumOutput', 'NumRx',
 'NumProcEvents', 'NumTransfers', 'NumChartEvents', 'ExpiredHospital',
 'TotalNumInteract', 'LOSgroupNum']
```

---

## Étape 3 — Aperçu et types

```python
df.head()       # 5 premières lignes
df.dtypes       # type de chaque colonne
df.describe()   # statistiques de base
```

---

## Étape 4 — Valeurs manquantes

```python
missing = df.isnull().sum()
missing_pct = (df.isnull().sum() / len(df) * 100).round(2)

result = pd.DataFrame({
    'valeurs_manquantes': missing,
    'pourcentage': missing_pct
})
result = result[result['valeurs_manquantes'] > 0].sort_values('pourcentage', ascending=False)
print(result)
```

**Résultat :**
```
                valeurs_manquantes  pourcentage
marital_status               10128        17.17
religion                       458         0.78
AdmitDiagnosis                  25         0.04
```

**Traitement prévu :**
| Colonne | % manquant | Action |
|---------|------------|--------|
| `marital_status` | 17.17% | Remplacer par `"UNKNOWN"` |
| `religion` | 0.78% | Remplacer par `"UNKNOWN"` |
| `AdmitDiagnosis` | 0.04% | Remplacer par `"UNKNOWN"` |

---

## Étape 5 — Distribution de LOSdays (variable cible)

```python
import matplotlib.pyplot as plt

fig, axes = plt.subplots(1, 2, figsize=(14, 4))

axes[0].hist(df['LOSdays'], bins=50, color='steelblue', edgecolor='white')
axes[0].set_title('Distribution LOSdays (brute)')

axes[1].hist(df['LOSdays'].apply(lambda x: x if x > 0 else 0.01).apply('log'),
             bins=50, color='darkorange', edgecolor='white')
axes[1].set_title('Distribution LOSdays (log)')

plt.tight_layout()
plt.savefig('/home/jovyan/notebooks/los_distribution.png')
plt.show()

print(df['LOSdays'].describe().round(2))
print(f"Séjours > 30 jours : {(df['LOSdays'] > 30).sum()} ({(df['LOSdays'] > 30).mean()*100:.1f}%)")
print(f"Séjours > 7 jours  : {(df['LOSdays'] > 7).sum()} ({(df['LOSdays'] > 7).mean()*100:.1f}%)")
```

**Résultat :**
```
count    58976.00
mean        10.11
std         12.46
min          0.00
25%          3.71
50%          6.46
75%         11.79
max        294.63

Séjours > 30 jours : 3060 (5.2%)
Séjours > 7 jours  : 27088 (45.9%)
```

**Interprétation :**
- Moyenne (10.1j) > Médiane (6.5j) → distribution asymétrique à droite
- 5.2% de séjours très longs (> 30 jours) tirent la moyenne vers le haut
- Maximum à 294 jours → cas extrême à traiter comme outlier

---

## Étape 6 — Distribution des variables catégorielles

```python
cat_cols = ['gender', 'admit_type', 'admit_location', 'insurance', 'ethnicity']

fig, axes = plt.subplots(2, 3, figsize=(16, 8))
axes = axes.flatten()

for i, col in enumerate(cat_cols):
    counts = df[col].value_counts()
    axes[i].barh(counts.index, counts.values, color='steelblue')
    axes[i].set_title(col)

axes[5].set_visible(False)
plt.tight_layout()
plt.savefig('/home/jovyan/notebooks/categorical_distributions.png')
plt.show()
```

**Insights :**
- `admit_type` : EMERGENCY domine (~40 000 cas sur 58 976)
- `ethnicity` : trop de catégories, regroupement nécessaire au Jour 2

---

## Étape 7 — Corrélations avec LOSdays

```python
num_cols = df.select_dtypes(include=np.number).columns.tolist()
correlations = df[num_cols].corr()['LOSdays'].drop('LOSdays')
correlations = correlations.abs().sort_values(ascending=False)
print(correlations.round(3))
```

**Résultat :**
```
LOSgroupNum         0.666   ← EXCLURE (data leakage)
NumCallouts         0.206
NumRx               0.169
NumTransfers        0.168
NumDiagnosis        0.156
NumProcEvents       0.100
NumLabs             0.098
NumProcs            0.088
...
```

---

## Étape 8 — Matrice de corrélation

```python
import seaborn as sns

cols_for_corr = [c for c in num_cols if c not in ['LOSgroupNum', 'hadm_id']]

fig, ax = plt.subplots(figsize=(14, 10))
sns.heatmap(df[cols_for_corr].corr(), annot=True, fmt='.2f',
            cmap='RdBu_r', center=0, ax=ax)
ax.set_title('Matrice de corrélation', fontweight='bold')
plt.tight_layout()
plt.savefig('/home/jovyan/notebooks/correlation_matrix.png')
plt.show()
```

**Insights heatmap :**
- `NumChartEvents` ↔ `TotalNumInteract` : corrélation 0.97 → variables quasi identiques, en garder une seule
- `NumDiagnosis` ↔ `NumTransfers` : 0.78 → multicolinéarité forte
- `LOSdays` faiblement corrélé avec tout → prédiction complexe, variables catégorielles importantes

---

## Synthèse finale

```python
print(f"""
DATASET
  Séjours      : {df.shape[0]:,}
  Variables    : {df.shape[1]}

QUALITÉ
  Manquants    : 3 colonnes (max 17% sur marital_status)
  Outliers     : LOSdays max = {df['LOSdays'].max()} jours

VARIABLE CIBLE (LOSdays)
  Moyenne      : {df['LOSdays'].mean():.1f} jours
  Médiane      : {df['LOSdays'].median():.1f} jours
  > 30 jours   : {(df['LOSdays'] > 30).mean()*100:.1f}%
""")
```

---

## ⚠️ Points importants à retenir

| Concept | Explication |
|---------|-------------|
| **Data leakage** | `LOSgroupNum` est une catégorisation de `LOSdays` → à exclure du modèle |
| **Multicolinéarité** | Deux variables très corrélées entre elles perturbent le modèle → en garder une |
| **Distribution asymétrique** | Moyenne > Médiane → penser à une transformation log pour le modèle |
| **Variables catégorielles** | À encoder (One-Hot ou Target Encoding) avant modélisation |

---

## ✅ Actions Jour 2
1. Exclure `LOSgroupNum` (data leakage)
2. Exclure `TotalNumInteract` (redondant avec `NumChartEvents`)
3. Imputer `marital_status`, `religion`, `AdmitDiagnosis` → `"UNKNOWN"`
4. Regrouper les ethnies en grandes catégories
5. Encoder les variables catégorielles
