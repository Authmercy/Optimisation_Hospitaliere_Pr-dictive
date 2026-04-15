# 📝 Tâche 1 — Configuration Environnement Technique

## Objectif
Mettre en place l'environnement de développement local avec Docker et initialiser le repository Git.

---

## Étape 1 — Installer Docker

```bash
# Vérifier l'installation
docker --version
docker compose version
```

**Résultat attendu :**
```
Docker version 25.x.x
Docker Compose version v2.x.x
```

---

## Étape 2 — Créer la structure du projet

```bash
# Créer le dossier principal
mkdir hospital-optimizer
cd hospital-optimizer

# Créer les sous-dossiers
mkdir -p data/raw
mkdir -p data/processed
mkdir -p notebooks
mkdir -p src/api
mkdir -p src/models
mkdir -p src/data
mkdir -p tests
mkdir -p docker
mkdir -p docs
mkdir -p monitoring

# Vérifier la structure
ls -R
```

**Résultat obtenu :**
```
.
├── data
│   ├── processed
│   └── raw
├── docker
├── docs
├── monitoring
├── notebooks
├── src
│   ├── api
│   ├── data
│   └── models
└── tests
```

---

## Étape 3 — Créer le fichier compose.yml

```yaml
services:

  # Jupyter Lab pour coder et explorer les données
  jupyter:
    image: jupyter/pyspark-notebook:spark-3.5.0
    container_name: hospital-jupyter
    ports:
      - "8888:8888"
    volumes:
      - ./notebooks:/home/jovyan/notebooks
      - ./data:/home/jovyan/data
      - ./src:/home/jovyan/src
    environment:
      - JUPYTER_ENABLE_LAB=yes
    networks:
      - hospital-net

  # Base de données PostgreSQL
  postgres:
    image: postgres:16-alpine
    container_name: hospital-postgres
    environment:
      POSTGRES_USER: hospital
      POSTGRES_PASSWORD: hospital
      POSTGRES_DB: hospital_db
    ports:
      - "5433:5432"   # 5433 car le port 5432 est occupé localement
    networks:
      - hospital-net

  # IHM Adminer pour gérer la base de données
  adminer:
    image: adminer
    restart: always
    ports:
      - "8090:8080"
    networks:
      - hospital-net

networks:
  hospital-net:
    driver: bridge
```

### Points importants sur compose.yml
- `ports: "5433:5432"` → format `machine:container`. Le port 5432 était déjà occupé par PostgreSQL local
- `volumes` → relie vos dossiers locaux aux dossiers dans le container
- `networks` → permet aux services de communiquer entre eux
- Dans Adminer, le serveur est `postgres` (nom du service), pas `localhost`

---

## Étape 4 — Lancer et vérifier les services

```bash
# Démarrer tous les services en arrière-plan
docker compose up -d

# Vérifier que les containers tournent
docker compose ps

# Récupérer le lien Jupyter avec token
docker logs hospital-jupyter 2>&1 | grep "http://127"

# Arrêter les services
docker compose down
```

**Services disponibles :**
| Service | URL | Identifiants |
|---------|-----|--------------|
| Jupyter Lab | http://localhost:8888 | token dans les logs |
| Adminer | http://localhost:8090 | voir ci-dessous |
| PostgreSQL | localhost:5433 | hospital / hospital |

**Connexion Adminer :**
```
Système      : PostgreSQL
Serveur      : postgres
Utilisateur  : hospital
Mot de passe : hospital
Base        : hospital_db
```

---

## Étape 5 — Initialiser Git

```bash
# Initialiser le repo
git init
git checkout -b develop

# Créer une branche pour la tâche
git checkout -b feature/day1-setup

# Créer le .gitignore
echo "data/raw/
data/processed/
__pycache__/
*.pyc
.env" > .gitignore

# Committer et pousser
git add .
git commit -m "feat: initial project structure + docker setup"
git push origin feature/day1-setup
```

### Convention Git du projet
```
main          ← code stable, production
develop       ← intégration
feature/*     ← une branche par tâche
```

---

## ⚠️ Problèmes rencontrés

| Problème | Cause | Solution |
|----------|-------|----------|
| Port 5432 déjà utilisé | PostgreSQL installé localement sur Windows | Utiliser le port 5433 côté machine |
| Adminer ne se connecte pas | Utilisation de `localhost` comme serveur | Utiliser `postgres` (nom du service Docker) |
| `cat compose.yml` ne montre pas les changements | Le fichier s'appelle `compose.yml` et non `docker-compose.yml` | Vérifier le nom avec `ls -la` |

---

## ✅ Livrable
- Structure du projet créée
- `compose.yml` configuré avec 3 services
- Jupyter accessible sur http://localhost:8888
- Repo GitLab initialisé avec branche `feature/day1-setup`
