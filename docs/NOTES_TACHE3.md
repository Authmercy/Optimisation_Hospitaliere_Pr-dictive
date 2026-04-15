# 📝 Tâche 3 — Design de l'Architecture Technique

## Objectif
Définir comment les données circulent depuis MIMIC3d jusqu'au médecin,
et justifier les choix technologiques.

---

## Les 5 couches du système

| Couche | Outil | Rôle |
|--------|-------|------|
| 1. Données | PostgreSQL | Stocker les données structurées |
| 2. Traitement | PySpark (Jupyter) | Nettoyer et préparer les données |
| 3. Modèles | Prophet + LSTM + PuLP | Prédire et optimiser |
| 4. Suivi ML | MLflow | Enregistrer les expériences et modèles |
| 5. API | FastAPI | Exposer les prédictions |
| 6. Dashboard | Streamlit | Afficher les résultats au médecin |
| 7. Privacy | diffprivlib | Protéger les données patients (RGPD) |

---

## Flux de données complet

```
┌─────────────┐
│   MIMIC3d   │  ← Données brutes (CSV Kaggle)
│   CSV       │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  PostgreSQL │  ← Stockage structuré
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
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   FastAPI   │  ← Exposition des prédictions
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Streamlit  │  ← Dashboard médecins
└──────┬──────┘
       │
       ▼
    Médecin 👨‍⚕️
```

---

## Batch vs Streaming

### Définitions

| Approche | Principe | Complexité |
|----------|----------|------------|
| **Batch** | Calcul à intervalles fixes | Simple |
| **Streaming** | Calcul dès qu'un événement arrive | Complexe |

### Choix pour notre projet

| Modèle | Approche | Fréquence | Justification |
|--------|----------|-----------|---------------|
| Prophet | Batch | Toutes les heures | Prédiction admissions 24h à l'avance |
| PuLP | Batch | 1 fois par jour | Planning staffing du lendemain |
| LSTM | Streaming | Temps réel | Alerte dès qu'un patient se dégrade |

### Schéma Batch vs Streaming

```
BATCH (toutes les heures)
─────────────────────────
PostgreSQL → PySpark → Prophet → MLflow → FastAPI → Streamlit

STREAMING (temps réel)
──────────────────────
Nouveaux événements → LSTM → Alerte → FastAPI → Streamlit
```

---

## Justification des choix technologiques

| Outil | Pourquoi ce choix |
|-------|-------------------|
| PostgreSQL | Fiable, structuré, standard en milieu médical |
| PySpark | Conçu pour les gros volumes de données |
| Prophet | Conçu spécifiquement pour les séries temporelles |
| LSTM | Capture les patterns complexes dans le temps |
| PuLP | Résout les problèmes d'optimisation linéaire |
| MLflow | Trace chaque expérience, compare les modèles |
| FastAPI | Rapide, génère la documentation automatiquement |
| Streamlit | Simple à coder, interactif, idéal pour prototypes |
| diffprivlib | Conformité RGPD, protège les données patients |

---

## ✅ Livrable
- Fichier `docs/architecture.md` créé et commité
- Flux de données documenté
- Choix batch/streaming justifiés
