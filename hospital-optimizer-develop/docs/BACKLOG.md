# 📋 BACKLOG — Hospital Optimizer
Équipe : 3 étudiants | Durée : 5 jours

---

## KPIs

| # | KPI | Cible | Modèle |
|---|-----|-------|--------|
| 1 | MAE prédiction admissions | < 2h | Prophet |
| 2 | Réduction temps d'attente | -20% | PuLP |
| 3 | Précision alertes complications | > 85% | LSTM |
| 4 | Couverture tests | > 80% | pytest |
| 5 | Latence API | < 200ms | FastAPI |

---

## Planning

### Jour 1 — Setup & Exploration ✅
| Tâche | Statut |
|-------|--------|
| Docker + Git + Structure projet | ✅ |
| Exploration MIMIC3d | ✅ |
| Architecture technique | ✅ |
| KPIs + Planification | ✅ |

### Jour 2 — Pipeline de données + ETL
| Tâche | Responsable |
|-------|-------------|
| Optimisation PySpark (partitioning, caching) | Dev 1 |
| Pipeline ETL + Delta Lake | Dev 1 |
| Feature Engineering (window functions) | Dev 2 |
| Data quality checks (Great Expectations) | Dev 2 |
| Tests unitaires pytest-spark | Dev 1 & 2 |

### Jour 3 — Machine Learning
| Tâche | Responsable |
|-------|-------------|
| Modèle Prophet (prédiction admissions) | Dev 1 |
| Modèle LSTM (alertes complications) | Dev 2 |
| Modèle PuLP (optimisation staffing) | Dev 1 & 2 |
| MLflow tracking + model selection | Dev 1 & 2 |

### Jour 4 — Déploiement + Interfaces
| Tâche | Responsable |
|-------|-------------|
| FastAPI (endpoints predict/optimize/alert) | Dev 1 |
| Dashboard Streamlit médecins | Dev 2 |
| Monitoring Prometheus + Grafana | Dev 1 & 2 |
| Tests end-to-end | Dev 1 & 2 |

### Jour 5 — Finalisation + Soutenance
| Tâche | Responsable |
|-------|-------------|
| Tests de charge + optimisation | Dev 1 |
| Documentation technique | Dev 2 |
| Préparation soutenance (15 min) | Équipe |
| Démonstration live | Équipe |

---

## Definition of Done
Une tâche est terminée quand :
- [ ] Code fonctionne sans erreur
- [ ] Tests écrits (coverage > 80%)
- [ ] Code commité via Pull Request sur GitLab
- [ ] Métriques loggées dans MLflow (si ML)
- [ ] Documentation mise à jour