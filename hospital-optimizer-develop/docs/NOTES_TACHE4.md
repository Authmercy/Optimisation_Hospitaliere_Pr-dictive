# 📝 Tâche 4 — Planification et Métriques

## Objectif
Définir les KPIs du projet, le planning sur 5 jours
et les critères d'acceptation (Definition of Done).

---

## Les KPIs

### Définition
Un KPI (Key Performance Indicator) est une mesure qui indique
si on a atteint un objectif.

### KPIs du projet

| # | KPI | Cible | Modèle | Comment le mesurer |
|---|-----|-------|--------|--------------------|
| 1 | MAE prédiction admissions | < 2h | Prophet | `mean_absolute_error(y_true, y_pred)` |
| 2 | Réduction temps d'attente | -20% | PuLP | Comparer avant/après optimisation |
| 3 | Précision alertes complications | > 85% | LSTM | `precision_score(y_true, y_pred)` |
| 4 | Couverture tests | > 80% | pytest | `pytest --cov` |
| 5 | Latence API | < 200ms | FastAPI | Mesurer temps de réponse |

### Explications des KPIs difficiles

**KPI 2 — PuLP (optimisation staffing)**
```
PuLP reçoit :
  - Contraintes : nombre d'infirmiers disponibles
  - Objectif    : minimiser les temps d'attente

PuLP calcule :
  - 8h-14h  : 1 infirmier
  - 14h-20h : 2 infirmiers (pic d'admissions)
  - 20h-8h  : 1 infirmier

Résultat : temps d'attente réduit de 20%
```

**KPI 3 — LSTM (alertes complications)**
```
Le LSTM apprend à partir des données historiques :
  Patient A : NumLabs=50, NumRx=20 → pas de complication
  Patient B : NumLabs=80, NumRx=35 → complication ⚠️

Précision 85% signifie :
  Sur 100 alertes émises :
    85 sont justifiées ✅
    15 sont fausses    ❌
```

---

## Planning J1 → J5

| Jour | Thème | Contenu principal |
|------|-------|-------------------|
| J1 | Setup & Exploration | Docker, Git, EDA, Architecture |
| J2 | Pipeline de données + ETL | PySpark avancé, Delta Lake, Feature Engineering |
| J3 | Machine Learning | Prophet, LSTM, PuLP, MLflow |
| J4 | Déploiement + Interfaces | FastAPI, Streamlit, Monitoring |
| J5 | Finalisation + Soutenance | Tests de charge, Documentation, Démo |

### Détail Jour 2
```
9h-10h30  : Optimisation PySpark (partitioning, caching)
10h45-12h : Pipeline ETL + Delta Lake
13h-15h   : Feature Engineering (window functions)
15h15-16h : Data quality checks (Great Expectations)
16h30-17h : Point d'avancement + debugging
```

### Détail Jour 3
```
- MLlib : pipelines ML distribués
- Prophet : prédiction admissions
- LSTM   : alertes complications
- PuLP   : optimisation staffing
- MLflow : tracking + model selection
```

### Détail Jour 4
```
- Docker : containerisation
- FastAPI : endpoints /predict /optimize /alert
- Streamlit : dashboard médecins
- Prometheus + Grafana : monitoring
```

### Détail Jour 5
```
- Tests de charge
- Documentation technique
- Soutenance : 15 min présentation + 10 min questions
- Démonstration live
```

---

## Definition of Done

Une tâche est **terminée** quand :

| Critère | Vérification |
|---------|-------------|
| Code fonctionne sans erreur | Exécution manuelle |
| Tests écrits (coverage > 80%) | `pytest --cov` |
| Code commité via Pull Request | GitLab |
| Métriques loggées dans MLflow | Interface MLflow |
| Documentation mise à jour | README ou docs/ |

---

## Fichier créé
- `docs/BACKLOG.md` — backlog complet commité sur GitLab

---

## ✅ Livrable
- KPIs définis et mesurables
- Planning J1-J5 aligné avec le programme du formateur
- Definition of Done claire pour toute l'équipe
