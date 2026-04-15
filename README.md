# 📋 Project.md — Hospital Optimizer

## Objectif du projet
Construire un système de prédiction et d'optimisation hospitalière
basé sur le dataset MIMIC-III, conforme RGPD et HL7 FHIR.

---

## Les 3 objectifs métier

### Objectif 1 — Prédire les admissions (Prophet)
```
On veut savoir : combien de patients vont arriver dans les 24h ?
Cible          : MAE < 2 heures
```

Prophet va analyser le volume d'admissions dans le temps et détecter des patterns :
```
Lundi matin      → pic d'admissions
Nuit du samedi   → beaucoup d'urgences
Été              → moins d'admissions électives
```

### Objectif 2 — Optimiser le staffing (PuLP)
```
On veut savoir : combien d'infirmiers mettre à chaque heure ?
Cible          : réduire les temps d'attente de -20%
```

PuLP va calculer :
```
Si 10 admissions prévues à 14h
Et chaque patient nécessite en moyenne 40 NumChartEvents
Alors il faut X infirmiers pour cette tranche horaire
```

### Objectif 3 — Alertes complications (LSTM)
```
On veut savoir : ce patient va-t-il se dégrader ?
Cible          : précision > 85%
```

Le LSTM apprend :
```
Patient avec NumLabs élevé + NumRx élevé + NumTransfers élevé
→ risque de complication ⚠️
```

---

## Le Dataset

### MIMIC-III complet (5 GB)
- **Source** : https://physionet.org/content/mimiciii/1.4/
- **Accès** : inscription PhysioNet + formation CITI requise (~1 semaine)
- **Contenu** : ~46 000 patients, ~58 000 séjours en ICU (2001-2012)
- **Format** : ~26 fichiers CSV

### MIMIC3d subset (3.4 MB) ✅ utilisé dans ce projet
- **Source** : https://www.kaggle.com/datasets/drscarlat/mimic3d
- **Accès** : libre, pas de formation requise
- **Contenu** : 58 976 séjours × 28 colonnes
- **Format** : 1 fichier CSV

### Différence entre les deux
| | MIMIC-III complet | MIMIC3d subset |
|--|-------------------|----------------|
| Taille | 5 GB | 3.4 MB |
| Fichiers | ~26 tables | 1 CSV |
| Détail | Toutes les mesures | Données agrégées |
| Accès | Inscription requise | Libre |
| Signes vitaux | ✅ | ❌ |
| Notes médecins | ✅ | ❌ |

### Conformité
- **HIPAA** (USA) : données anonymisées, pas de noms ni dates précises
- **RGPD** (Europe) : diffprivlib pour la protection supplémentaire

---

## Les 28 colonnes expliquées

### Groupe 1 — Identification
| Colonne | Explication | Exemple |
|---------|-------------|---------|
| `hadm_id` | Identifiant unique du séjour | 100001 |

### Groupe 2 — Données démographiques
| Colonne | Explication | Exemple |
|---------|-------------|---------|
| `gender` | Genre du patient | M / F |
| `age` | Âge du patient | 35, 59 |
| `ethnicity` | Origine ethnique | WHITE, BLACK... |
| `religion` | Religion | PROTESTANT, NOT SPECIFIED |
| `marital_status` | Situation maritale | SINGLE, MARRIED, DIVORCED |
| `insurance` | Type d'assurance | Private, Medicare, Medicaid |

### Groupe 3 — Admission
| Colonne | Explication | Exemple |
|---------|-------------|---------|
| `admit_type` | Type d'admission | EMERGENCY, ELECTIVE, NEWBORN |
| `admit_location` | D'où vient le patient | EMERGENCY ROOM, CLINIC REFERRAL |
| `AdmitDiagnosis` | Diagnostic à l'entrée | DIABETIC KETOACIDOSIS |
| `AdmitProcedure` | Acte médical à l'entrée | Endosc control gast hem |

### Groupe 4 — Durée et outcome
| Colonne | Explication | Exemple |
|---------|-------------|---------|
| `LOSdays` | Durée de séjour en jours ⭐ variable cible | 6.17 |
| `LOSgroupNum` | Catégorie de durée (1=court, 2=moyen...) | 1, 2, 3 |
| `ExpiredHospital` | Décédé pendant le séjour | 0=vivant, 1=décédé |

### Groupe 5 — Activité médicale (colonnes Num*)
> Ce sont des moyennes par séjour issues de MIMIC-III complet.

| Colonne | Explication |
|---------|-------------|
| `NumCallouts` | Nombre de demandes de transfert |
| `NumDiagnosis` | Nombre de diagnostics posés |
| `NumProcs` | Nombre de procédures médicales |
| `NumCPTevents` | Nombre d'actes facturés |
| `NumInput` | Nombre de perfusions / entrées liquidiennes |
| `NumLabs` | Nombre d'analyses biologiques |
| `NumMicroLabs` | Nombre d'analyses microbiologiques |
| `NumNotes` | Nombre de notes rédigées par les médecins |
| `NumOutput` | Nombre de sorties liquidiennes |
| `NumRx` | Nombre de médicaments prescrits |
| `NumProcEvents` | Nombre d'événements de procédures |
| `NumTransfers` | Nombre de transferts entre services |
| `NumChartEvents` | Nombre de mesures enregistrées |
| `TotalNumInteract` | Total de toutes les interactions médicales |

---

## Lien colonnes → objectifs

### Objectif 1 — Prophet (prédiction admissions)
```
admit_type      → quel type d'admission
admit_location  → d'où viennent les patients
age, gender     → profil des patients
```

### Objectif 2 — PuLP (optimisation staffing)
```
LOSdays         → combien de temps les patients restent
NumTransfers    → combien bougent entre services
admit_type      → urgence ou programmé
NumChartEvents  → charge de travail par patient
```

### Objectif 3 — LSTM (alertes complications)
```
NumLabs         → beaucoup d'analyses = patient surveillé
NumMicroLabs    → infection possible
NumInput/Output → déséquilibre hydrique
NumRx           → beaucoup de médicaments = cas complexe
NumDiagnosis    → plusieurs diagnostics = patient fragile
ExpiredHospital → variable cible (0 ou 1)
```

---

## Colonnes à exclure

| Colonne | Raison |
|---------|--------|
| `hadm_id` | Simple identifiant, pas de valeur prédictive |
| `LOSgroupNum` | Dérivé de `LOSdays` → data leakage |
| `TotalNumInteract` | Corrélé à 0.97 avec `NumChartEvents` → redondant |

---

## Résultats de l'exploration (Jour 1)

### Qualité des données
| Colonne | Valeurs manquantes | % | Traitement |
|---------|-------------------|---|------------|
| `marital_status` | 10 128 | 17.17% | Remplacer par "UNKNOWN" |
| `religion` | 458 | 0.78% | Remplacer par "UNKNOWN" |
| `AdmitDiagnosis` | 25 | 0.04% | Remplacer par "UNKNOWN" |

### Variable cible LOSdays
```
Moyenne  : 10.1 jours
Médiane  : 6.5 jours
Maximum  : 294.6 jours
> 30j    : 5.2% des séjours
```

### Top corrélations avec LOSdays
```
NumCallouts   : 0.206
NumRx         : 0.169
NumTransfers  : 0.168
NumDiagnosis  : 0.156
```

### Insights métier
```
- EMERGENCY représente ~70% des admissions
- Distribution LOSdays asymétrique → transformation log nécessaire
- NumChartEvents et TotalNumInteract quasi identiques (0.97)
- L'âge influence peu directement la durée de séjour (0.033)
```

---

## Architecture Medallion (Bronze / Silver / Gold)

```
🥉 BRONZE  → data/bronze/   Copie brute de mimic3d.csv (Parquet)
🥈 SILVER  → data/silver/   Données nettoyées (imputation, types)
🥇 GOLD    → data/gold/     Features métier pour les modèles
```

---

## Stack technique

| Outil | Rôle |
|-------|------|
| PostgreSQL | Stockage structuré (Gold) |
| PySpark | Traitement des données |
| Prophet | Prédiction admissions |
| LSTM (TensorFlow) | Alertes complications |
| PuLP | Optimisation staffing |
| MLflow | Suivi des expériences ML |
| FastAPI | API REST |
| Streamlit | Dashboard médecins |
| diffprivlib | Conformité RGPD |
| MinIO | Data Lake local (Bronze/Silver) |
