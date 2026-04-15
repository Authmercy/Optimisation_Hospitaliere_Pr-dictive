# Architecture — Hospital Optimizer

## Étape 1 — Les composants nécessaires
Imaginons ce scénario :

> Un médecin ouvre son dashboard le matin et veut voir **combien de patients vont être admis dans les 24 prochaines heures**.

Voici ce qui se passe dans l'ordre :

---

**1. Les données existent quelque part**
```
MIMIC3d (fichier CSV) 
        ↓
PostgreSQL (base de données)
```
Les données historiques des patients sont stockées dans PostgreSQL.

---

**2. On prépare les données**
```
PostgreSQL 
        ↓
PySpark (nettoyage, transformation)
```
PySpark lit les données, supprime les valeurs manquantes, crée les features.

---

**3. Le modèle fait une prédiction**
```
PySpark (données propres)
        ↓
Prophet / LSTM (prédiction)
        ↓
MLflow (sauvegarde le résultat)
```
Prophet prédit le nombre d'admissions. MLflow enregistre l'expérience.

---

**4. La prédiction est exposée**
```
MLflow (modèle entraîné)
        ↓
FastAPI (endpoint /predict)
```
FastAPI crée une URL que n'importe qui peut appeler pour obtenir une prédiction.

---

**5. Le médecin voit le résultat**
```
FastAPI
        ↓
Streamlit (dashboard)
        ↓
Médecin 👨‍⚕️
```

---

Le flux complet est donc :

```
CSV → PostgreSQL → PySpark → Prophet/LSTM → MLflow → FastAPI → Streamlit → Médecin
```

---

## Flux de données complet

┌─────────────┐
│   MIMIC3d   │  ← Données brutes (CSV Kaggle)
│   CSV       │
└──────┬──────┘
│
▼
┌─────────────┐
│  PostgreSQL │  ← Stockage structuré
│             │
└──────┬──────┘
│
▼
┌─────────────┐
│   PySpark   │  ← Nettoyage + Feature Engineering
│  (Jupyter)  │
└──────┬──────┘
│
▼
┌──────────────────────────┐
│        MODÈLES           │
│  Prophet  → admissions   │
│  LSTM     → complications│
│  PuLP     → staffing     │
└──────┬───────────────────┘
│
▼
┌─────────────┐
│   MLflow    │  ← Suivi des expériences + Modèles
│             │
└──────┬──────┘
│
▼
┌─────────────┐
│   FastAPI   │  ← Exposition des prédictions
│             │
└──────┬──────┘
│
▼
┌─────────────┐
│  Streamlit  │  ← Dashboard médecins
│             │
└──────┬──────┘
│
▼
Médecin 👨‍⚕️

## Rôle de chaque outil

| Outil | Rôle | Pourquoi ce choix |
|-------|------|-------------------|
| PostgreSQL | Stockage données | Fiable, structuré, standard médical |
| PySpark | Traitement | Gère de gros volumes de données |
| Prophet | Prédiction admissions | Conçu pour les séries temporelles |
| LSTM | Alertes complications | Capture les patterns complexes |
| PuLP | Optimisation staffing | Résout les problèmes d'allocation |
| MLflow | Suivi ML | Trace chaque expérience et modèle   |
| FastAPI | API REST | Rapide, documentation automatique |
| Streamlit | Dashboard | Simple à coder, interactif |
| diffprivlib | Privacy RGPD | Protège les données patients |

## Étape 3 — Batch vs Streaming

Avant de finaliser l'architecture, il faut répondre à une question importante :

> **Les prédictions doivent-elles être calculées en temps réel ou par lot ?**

---

### Deux approches possibles

**Batch (par lot)**
- On calcule les prédictions **à intervalles fixes** (ex: toutes les heures)
- Moins complexe à développer
- Exemple : "Chaque heure, Prophet prédit les admissions des 24 prochaines heures"

**Streaming (temps réel)**
- On calcule une prédiction **dès qu'un nouvel événement arrive**
- Plus complexe, nécessite Kafka ou Spark Streaming
- Exemple : "Dès qu'un patient entre aux urgences, on recalcule immédiatement"

---

### Pour notre projet, que choisir ?

Regardons nos 3 besoins :

| Besoin | Fréquence nécessaire | Choix |
|--------|---------------------|-------|
| Prédiction admissions 24h | Toutes les heures suffit | **Batch** |
| Optimisation staffing | 1 fois par jour | **Batch** |
| Alertes complications | Dès qu'un patient se dégrade | **Streaming** |

---

### Conclusion pour notre architecture

```
Batch   → Prophet + PuLP   (toutes les heures)
Streaming → LSTM alertes   (temps réel)
```

---
## Batch vs Streaming

| Modèle | Approche | Fréquence | Justification |
|--------|----------|-----------|---------------|
| Prophet | Batch | Toutes les heures | Prédiction admissions 24h à l'avance |
| PuLP | Batch | 1 fois par jour | Planning staffing du lendemain |
| LSTM | Streaming | Temps réel | Alerte dès qu'un patient se dégrade |

## Schéma final

```
BATCH (toutes les heures)
─────────────────────────
PostgreSQL → PySpark → Prophet → MLflow → FastAPI → Streamlit

STREAMING (temps réel)
──────────────────────
Nouveaux événements → LSTM → Alerte → FastAPI → Streamlit
```