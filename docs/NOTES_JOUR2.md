# 📝 Jour 2 — Pipeline ETL + Feature EOngineering + Validation qualité

## Objectifs accomplis
- ✅ Architecture Medallion Bronze/Silver/Gold
- ✅ Pipeline ETL robuste avec gestion d'erreurs
- ✅ Jointure des 5 tables MIMIC-III
- ✅ Feature Engineering avec PySpark ML
- ✅ Validation qualité avec alerting automatique

---

## Nouveau service ajouté : MinIO

```yaml
# Dans compose.yml
minio:
  image: minio/minio
  container_name: hospital-minio
  ports:
    - "9000:9000"   # API
    - "9001:9001"   # Interface web
  environment:
    MINIO_ROOT_USER: hospital
    MINIO_ROOT_PASSWORD: hospital123
  command: server /data --console-address ":9001"
  volumes:
    - ./data/minio:/data
  networks:
    - hospital-net
```

**Accès** : http://localhost:9001 (hospital / hospital123)

**Note** : La connexion Spark → MinIO nécessite les JARs S3A.
On a utilisé le stockage local à la place.

---

## Architecture Medallion

```
🥉 BRONZE → data/bronze/   Copie brute des données
🥈 SILVER → data/silver/   Données nettoyées
🥇 GOLD   → data/gold/     Features pour les modèles
```

### Règles par couche
| Couche | Règle | Format |
|--------|-------|--------|
| Bronze | Aucune modification | Parquet |
| Silver | Imputation + filtrage outliers | Parquet |
| Gold | Features métier + window functions | Parquet |

---

## Dataset MIMIC-III synthétique

### 5 tables générées
| Table | Lignes | Taille |
|-------|--------|--------|
| PATIENTS | 50 000 | 1.1 MB |
| ADMISSIONS | 84 887 | 13.3 MB |
| ICUSTAYS | 33 955 | 3.0 MB |
| LABEVENTS | 14 601 058 | 1.33 GB |
| PRESCRIPTIONS | 2 190 401 | 181 MB |

### Génération sur Google Colab
```python
# Lancer le script
python generate_mimic_colab.py

# Télécharger les petits fichiers automatiquement
# Pour LABEVENTS → Google Drive
from google.colab import drive
drive.mount('/content/drive')
import shutil
shutil.copy('/content/mimic_synthetic/LABEVENTS.csv',
            '/content/drive/MyDrive/LABEVENTS.csv')
```

---

## Pipeline ETL complet

### Démarrage Spark
```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

spark = SparkSession.builder \
    .master("local[*]") \
    .appName("HospitalOptimizer-ETL") \
    .config("spark.driver.memory", "4g") \
    .getOrCreate()
```

### Extract — Chargement des 5 tables
```python
BASE_PATH = "/home/jovyan/data/raw/mimic_synthetic"

patients      = spark.read.csv(f"{BASE_PATH}/PATIENTS.csv",      header=True, inferSchema=True)
admissions    = spark.read.csv(f"{BASE_PATH}/ADMISSIONS.csv",    header=True, inferSchema=True)
icustays      = spark.read.csv(f"{BASE_PATH}/ICUSTAYS.csv",      header=True, inferSchema=True)
labevents     = spark.read.csv(f"{BASE_PATH}/LABEVENTS.csv",     header=True, inferSchema=True)
prescriptions = spark.read.csv(f"{BASE_PATH}/PRESCRIPTIONS.csv", header=True, inferSchema=True)
```

### Transform — Jointure des tables
```python
# ADMISSIONS + PATIENTS
adm_patients = admissions.join(
    patients.select("subject_id", "gender", "dob", "expire_flag"),
    on="subject_id", how="left"
)

# + ICUSTAYS
adm_icu = adm_patients.join(
    icustays.select("hadm_id", "icustay_id", "first_careunit", "los"),
    on="hadm_id", how="left"
)

# Agréger LABEVENTS par séjour
labs_agg = labevents.groupBy("hadm_id").agg(
    F.count("row_id").alias("num_labs"),
    F.countDistinct("itemid").alias("num_distinct_labs"),
    F.sum(F.when(F.col("flag") == "abnormal", 1).otherwise(0)).alias("num_abnormal_labs")
)

# Agréger PRESCRIPTIONS par séjour
rx_agg = prescriptions.groupBy("hadm_id").agg(
    F.count("row_id").alias("num_rx"),
    F.countDistinct("drug").alias("num_distinct_drugs")
)
```

### Calcul LOSdays
```python
df = df.withColumn(
    "LOSdays",
    F.round(
        (F.unix_timestamp("dischtime") - F.unix_timestamp("admittime")) / 86400, 2
    )
)
```

### Load Bronze
```python
df.write.mode("overwrite").parquet("/home/jovyan/data/bronze/mimic_full_raw.parquet")
```

### Transform Silver
```python
df_silver = df_bronze \
    .fillna("UNKNOWN", subset=["marital_status", "religion", "language"]) \
    .fillna(0, subset=["num_labs", "num_rx", "num_distinct_drugs", "los"]) \
    .filter((F.col("LOSdays") > 0) & (F.col("LOSdays") <= 200)) \
    .drop("expire_flag", "dob")
```

### Transform Gold — Features métier
```python
from pyspark.sql.window import Window

# Score de complexité
df_gold = df_gold.withColumn("complexity_score",
    F.round((df_gold.num_labs + df_gold.num_rx + df_gold.num_distinct_drugs) / 3, 2))

# Taux de labs anormaux
df_gold = df_gold.withColumn("abnormal_lab_rate",
    F.round(F.when(df_gold.num_labs > 0,
        df_gold.num_abnormal_labs / df_gold.num_labs).otherwise(0), 3))

# Patient à risque
df_gold = df_gold.withColumn("high_risk",
    F.when(df_gold.LOSdays > 6.5, 1).otherwise(0))

# Durée moyenne par type admission (window function)
window_admit = Window.partitionBy("admission_type")
df_gold = df_gold.withColumn("avg_los_by_admit_type",
    F.round(F.avg("LOSdays").over(window_admit), 2))

# Durée moyenne par service ICU (window function)
window_icu = Window.partitionBy("first_careunit")
df_gold = df_gold.withColumn("avg_los_by_careunit",
    F.round(F.avg("LOSdays").over(window_icu), 2))
```

### Pipeline ETL avec gestion d'erreurs
```python
import logging
logger = logging.getLogger("HospitalETL")

def extract(path):
    try:
        df = spark.read.csv(path, header=True, inferSchema=True)
        logger.info(f"✅ Extract OK — {df.count():,} lignes")
        return df
    except Exception as e:
        logger.error(f"❌ Extract échoué : {e}")
        return None

def transform_silver(df):
    try:
        df = df.fillna("UNKNOWN", subset=["marital_status", "religion"])
        df = df.filter(df.LOSdays <= 200)
        logger.info(f"✅ Transform Silver OK")
        return df
    except Exception as e:
        logger.error(f"❌ Transform Silver échoué : {e}")
        return None

def load(df, path):
    try:
        df.write.mode("overwrite").parquet(path)
        logger.info(f"✅ Load OK — {path}")
    except Exception as e:
        logger.error(f"❌ Load échoué : {e}")
```

---

## Feature Engineering — Pipeline PySpark ML

```python
from pyspark.ml import Pipeline
from pyspark.ml.feature import (
    StringIndexer, OneHotEncoder,
    VectorAssembler, StandardScaler, Imputer
)

# Imputer
imputer = Imputer(inputCols=final_features,
                  outputCols=[f"{c}_imputed" for c in final_features],
                  strategy="median")

# Encodage catégoriel
indexers = [StringIndexer(inputCol=col, outputCol=f"{col}_index",
                          handleInvalid="keep")
            for col in cat_features]

encoders = [OneHotEncoder(inputCol=f"{col}_index",
                          outputCol=f"{col}_encoded")
            for col in cat_features]

# Assemblage + normalisation
assembler = VectorAssembler(
    inputCols=[f"{c}_imputed" for c in final_features] +
              [f"{col}_encoded" for col in cat_features],
    outputCol="features_raw"
)

scaler = StandardScaler(inputCol="features_raw",
                        outputCol="features",
                        withMean=True, withStd=True)

pipeline = Pipeline(stages=[imputer] + indexers + encoders + [assembler, scaler])
```

---

## Validation qualité

```python
def validate_and_alert(df, rules):
    failed_rules = []
    for name, condition in rules:
        n_fail = (~condition).sum()
        pct    = n_fail / len(df) * 100
        if n_fail > 0:
            failed_rules.append((name, n_fail, pct))
            logger.warning(f"⚠️  ANOMALIE : {name} — {n_fail} lignes ({pct:.2f}%)")
        else:
            logger.info(f"✅ OK : {name}")
    return len(failed_rules) == 0

rules = [
    ("LOSdays entre 0 et 200",         df["LOSdays"].between(0, 200)),
    ("gender = M ou F",                df["gender"].isin(["M", "F"])),
    ("hadm_id jamais null",            df["hadm_id"].notnull()),
    ("num_labs >= 0",                  df["num_labs"] >= 0),
    ("abnormal_lab_rate entre 0 et 1", df["abnormal_lab_rate"].between(0, 1)),
    ("high_risk = 0 ou 1",             df["high_risk"].isin([0, 1])),
]

is_valid = validate_and_alert(df, rules)
```

**Résultat obtenu :**
```
✅ OK : LOSdays entre 0 et 200
✅ OK : gender = M ou F
✅ OK : hadm_id jamais null
✅ OK : num_labs >= 0
✅ OK : abnormal_lab_rate entre 0 et 1
✅ OK : high_risk = 0 ou 1
✅ Toutes les règles validées — données prêtes pour les modèles
```

---

## Résultats finaux Bronze/Silver/Gold

| Couche | Fichier | Lignes | Colonnes |
|--------|---------|--------|----------|
| Bronze | mimic_full_raw.parquet | 84 887 | 28 |
| Silver | mimic_full_clean.parquet | 84 886 | 26 |
| Gold | mimic_full_features.parquet | 84 886 | 31 |

---

## ⚠️ Points importants à retenir

| Concept | Explication |
|---------|-------------|
| **Parquet** | Format binaire rapide, garde les types, compressé |
| **Window Function** | Calcule des stats par groupe sans réduire les lignes |
| **Data Leakage** | Variable dérivée de la cible → exclure du modèle |
| **inferSchema** | Spark détecte automatiquement les types des colonnes |
| **fillna(0)** | Imputer les nums manquants après jointure left |

---

## ✅ Actions Jour 3
1. Entraîner Prophet sur les admissions agrégées par heure
2. Entraîner LSTM sur les features Gold
3. Implémenter PuLP pour l'optimisation staffing
4. Logger tous les modèles dans MLflow
