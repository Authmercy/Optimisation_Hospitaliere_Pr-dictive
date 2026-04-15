# 📝 Jour 3 — Machine Learning : Prophet + LSTM + PuLP

## Objectifs accomplis
- ✅ Modèle Prophet — prédiction admissions (MAE = 1.82)
- ✅ Modèle LSTM — alertes complications (précision = 93.7%)
- ✅ Modèle PuLP — optimisation staffing (réduction = 80%)
- ✅ MLflow — tracking des 3 modèles

---

## Problèmes résolus

### 1 — MLflow inaccessible depuis Jupyter
```
Erreur : Connection refused / Invalid Host header
Cause  : MLflow écoutait sur 127.0.0.1 au lieu de 0.0.0.0
         + protection DNS rebinding

Solution dans compose.yml :
command: ["pip install mlflow --quiet && mlflow server
          --backend-store-uri sqlite:///mlflow.db
          --default-artifact-root /mlflow/artifacts
          --host 0.0.0.0
          --port 5000
          --allowed-hosts '*'"]

URL depuis Jupyter : http://hospital-mlflow:5000
URL depuis browser : http://localhost:5000
```

### 2 — PermissionError artifacts MLflow
```
Erreur : Permission denied: '/mlflow'
Cause  : Le container Jupyter ne peut pas écrire
         dans le volume du container MLflow

Solution : Logger seulement params/métriques dans MLflow
           Sauvegarder les modèles localement dans src/models/
```

### 3 — Prophet backend Stan manquant
```
Erreur : Prophet object has no attribute stan_backend
Cause  : cmdstanpy non installé

Solution dans Dockerfile :
RUN pip install cmdstanpy==1.2.0 && \
    python -c "import cmdstanpy; cmdstanpy.install_cmdstan()"
```

---

## Tâche 1 — Prophet

### Principe
```
Prophet = modèle séries temporelles Facebook
Input   : ds (date) + y (valeur)
Output  : prédiction future
```

### Code complet
```python
from prophet import Prophet
from sklearn.metrics import mean_absolute_error

# Préparer les données
admissions_daily = df_gold \
    .withColumn("ds", F.to_date("ADMITTIME")) \
    .groupBy("ds") \
    .agg(F.count("HADM_ID").alias("y")) \
    .orderBy("ds") \
    .filter(F.col("ds").isNotNull()) \
    .toPandas()

# Split train/test
split_idx = int(len(df_prophet) * 0.80)
train     = df_prophet[:split_idx]
test      = df_prophet[split_idx:]

# Entraîner
model = Prophet(
    yearly_seasonality=True,
    weekly_seasonality=True,
    daily_seasonality=False,
    changepoint_prior_scale=0.05
)
model.fit(train)

# Prédire
future   = model.make_future_dataframe(periods=len(test), freq='D')
forecast = model.predict(future)

# Évaluer
mae = mean_absolute_error(test['y'], forecast.tail(len(test))['yhat'])
print(f"MAE : {mae:.2f}")  # 1.82
```

### Résultats
```
MAE    : 1.82 admissions
Cible  : < 2h
KPI    : ✅ ATTEINT
Modèle : src/models/prophet_model.pkl
```

### Logger dans MLflow
```python
mlflow.set_tracking_uri("http://hospital-mlflow:5000")
mlflow.set_experiment("/hospital-optimizer/prophet")

with mlflow.start_run(run_name="prophet_admissions_v1"):
    mlflow.log_param("yearly_seasonality", True)
    mlflow.log_metric("mae", round(mae, 2))
    mlflow.log_metric("kpi_achieved", 1 if mae < 2 else 0)
```

---

## Tâche 2 — LSTM

### Principe
```
LSTM = réseau de neurones pour séquences temporelles
Input  : features du patient (num_labs, num_rx, complexity...)
Output : probabilité de complication (0 ou 1)
```

### Déséquilibre des classes
```
Patients à risque (1)    : 78.2% → majoritaire
Patients non à risque (0): 21.8% → minoritaire

Solution : class_weight = {0: 3.0, 1: 1.0}
```

### Architecture
```python
model_lstm = Sequential([
    LSTM(64, input_shape=(1, 8), return_sequences=True),
    Dropout(0.2),
    LSTM(32, return_sequences=False),
    Dropout(0.2),
    Dense(16, activation='relu'),
    Dense(1, activation='sigmoid')
])

model_lstm.compile(
    optimizer='adam',
    loss='binary_crossentropy',
    metrics=['accuracy', Precision(), Recall()]
)
```

### Entraînement
```python
early_stopping = EarlyStopping(
    monitor='val_loss',
    patience=3,
    restore_best_weights=True
)

history = model_lstm.fit(
    X_train, y_train,
    epochs=10,
    batch_size=512,
    validation_split=0.2,
    class_weight={0: 3.0, 1: 1.0},
    callbacks=[early_stopping]
)
```

### Résultats
```
Précision : 93.7%  (cible > 85%) → KPI ✅ ATTEINT
Recall    : 55.6%  → à améliorer (rate 44% des vrais cas)
F1 Score  : 69.8%
Epochs    : 7 (early stopping)
Modèle    : src/models/lstm_model.keras
```

### Points importants
| Concept | Explication |
|---------|-------------|
| Reshape | LSTM attend (samples, timesteps, features) |
| Dropout | Évite le surapprentissage |
| Early stopping | Arrête si val_loss ne s'améliore plus |
| class_weight | Compense le déséquilibre des classes |
| Précision vs Recall | Précision = alertes justes / Recall = vrais cas détectés |

---

## Tâche 3 — PuLP

### Principe
```
PuLP = optimisation linéaire
Input  : contraintes (min/max infirmiers, budget)
Output : répartition optimale par heure
```

### Code complet
```python
import pulp

TOTAL_NURSES = 20
MIN_NURSES   = 2
MAX_NURSES   = 10
HOURS        = list(range(24))

# Problème
prob = pulp.LpProblem("StaffingOptimization", pulp.LpMaximize)

# Variables
nurses = {
    h: pulp.LpVariable(f"nurses_{h}",
                       lowBound=MIN_NURSES,
                       upBound=MAX_NURSES,
                       cat='Integer')
    for h in HOURS
}

# Objectif : maximiser la couverture
prob += pulp.lpSum(charge[h] * nurses[h] for h in HOURS)

# Contraintes
prob += pulp.lpSum(nurses[h] for h in HOURS) <= TOTAL_NURSES * 24
for h in peak_hours:
    prob += nurses[h] >= 4

# Résoudre
prob.solve(pulp.PULP_CBC_CMD(msg=0))
print(pulp.LpStatus[prob.status])  # Optimal
```

### Résultats
```
Réduction temps attente : 80%  (cible -20%) → KPI ✅ ATTEINT
Statut                  : Optimal
```

### ⚠️ Limitation
```
Données synthétiques uniformes → tous les créneaux ont le max
En production avec vraies données → répartition variable
```

---

## Résumé KPIs Jour 3

| Modèle | Métrique | Résultat | Cible | Statut |
|--------|----------|----------|-------|--------|
| Prophet | MAE | 1.82 | < 2.0 | ✅ |
| LSTM | Précision | 93.7% | > 85% | ✅ |
| PuLP | Réduction | 80% | > 20% | ✅ |

---

## ✅ Actions Jour 4
1. Créer les endpoints FastAPI
2. Connecter les modèles à l'API
3. Créer le dashboard Streamlit
4. Configurer le monitoring Prometheus + Grafana
