# Supply Chain - Satisfaction des clients

[![CI Tests Docker](https://github.com/askeur/supply_chain_satisfaction_clients/workflows/CI%20Tests%20Docker/badge.svg)](https://github.com/askeur/supply_chain_satisfaction_clients/actions)

> Projet Data Engineer 2025 — DataScientest  
> Analyse de la satisfaction client à partir des avis Trustpilot

**Auteurs :** Nabila Askeur  
**Mentor :** Vincent V.  
**Code source :** supply_chain_satisfaction_clients  
Projet créé en septembre 2025, disponible sur demande  
(Contact : [askeurnabila@gmail.com](mailto:askeurnabila@gmail.com))

<p align="left">
  <img src="accueil_ihm.png" alt="Supply Chain - Satisfaction des clients" width="800"/>
</p>
---

##  Table des matières

- [Vue d'ensemble](#-vue-densemble)
- [Architecture](#-architecture)
- [Sources de données](#-sources-de-données)
- [Pipeline ETL](#-pipeline-etl)
- [Machine Learning](#-machine-learning)
- [API et Application](#-api-et-application)
- [Automatisation](#-automatisation)
- [Installation](#-installation)
- [Utilisation](#-utilisation)
- [Monitoring](#-monitoring)
- [Dépannage](#-dépannage)

---

##  Vue d'ensemble

### Objectifs du projet

Analyser la satisfaction client à partir des avis Trustpilot en construisant une infrastructure complète de Data Engineering :

1. **Collecte de données** : Scraping multi-méthodes (Selenium, API HTTP, Kaggle)
2. **Stockage** : Architecture multi-bases (PostgreSQL, Elasticsearch, Snowflake)
3. **Analyse ML** : Modèles de sentiment (TF-IDF, BiLSTM, XLM-RoBERTa)
4. **Production** : API FastAPI + Interface Streamlit
5. **Automatisation** : Orchestration Airflow + CI/CD GitHub Actions

### Données cibles

- **Entreprises** : Catalogue par catégorie avec KPIs (TrustScore, nombre d'avis, distribution des étoiles)
- **Avis détaillés** : Texte, note, date, langue, réponse entreprise, localisation
- **Volume** : ~158 000 avis, ~2 500 entreprises, ~29 000 auteurs

---

##  Architecture

### Stack technique
```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                        │
│  ┌──────────────────┐          ┌──────────────────┐         │
│  │   Streamlit UI   │◄────────►│   FastAPI REST   │         │
│  │   (Port 8501)    │          │   (Port 8000)    │         │
│  └──────────────────┘          └──────────────────┘         │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                     Data Layer                              │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │ PostgreSQL  │  │Elasticsearch │  │  Snowflake   │        │
│  │   (Local)   │  │   + Kibana   │  │   (Cloud)    │        │
│  └─────────────┘  └──────────────┘  └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Orchestration & Monitoring Layer               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   Airflow    │  │   Grafana    │  │GitHub Actions│       │
│  │ (Port 8080)  │  │ (Port 3000)  │  │   (CI/CD)    │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### Services Docker

| Service | Image | Port | Description |
|---------|-------|------|-------------|
| **postgres** | postgres:15 | 5432 | Base relationnelle (dev/test) |
| **elasticsearch** | elasticsearch:8.11.0 | 9200 | Recherche full-text |
| **kibana** | kibana:8.11.0 | 5601 | Visualisation Elasticsearch |
| **airflow-webserver** | apache/airflow:2.7.0 | 8080 | Orchestration ETL |
| **airflow-scheduler** | apache/airflow:2.7.0 | - | Planificateur de tâches |
| **api** | Custom (FastAPI) | 8000 | API ML inference |
| **app** | Custom (Streamlit) | 8501 | Interface utilisateur |
| **grafana** | grafana/grafana:latest | 3000 | Monitoring |

---

##  Sources de données

### Trois méthodes complémentaires

#### 🔷 Méthode 1 : Scraping Selenium (Web UI)
- **Approche** : Navigation automatisée avec Selenium Headless
- **Scripts** : `scrape_companies.py`, `scrape_reviews.py`
- **Avantages** : Simule un comportement utilisateur réel
- **Destination** : PostgreSQL (Local) - Environnement de test

#### 🔷 Méthode 2 : API HTTP directe (Parsing JSON)
- **Approche** : Extraction du JSON `__NEXT_DATA__` embarqué
- **Scripts** : `scrape_companies_api.py`, `scrape_reviews_api.py`
- **Avantages** : 3-5x plus rapide, moins gourmand en ressources
- **Destination** : Snowflake (Cloud) - Données temps réel production

#### 🔷 Méthode 3 : Dataset Kaggle
- **Source** : [jerassy/trustpilot-reviews-123k](https://www.kaggle.com/datasets/jerassy/trustpilot-reviews-123k)
- **Volume** : ~123 000 avis pré-collectés
- **Script** : `kaggle_companies_reviews.py`
- **Usage** : Validation externe, enrichissement

### Tableau comparatif

| Critère | Selenium | API HTTP | Kaggle |
|---------|----------|----------|--------|
| **Vitesse** | 🟡 Moyenne (10-15s/page) | 🟢 Rapide (2-4s/page) | 🟢 Immédiate |
| **Ressources** | 🔴 Élevées (RAM + CPU) | 🟢 Faibles | 🟢 Minimales |
| **Fiabilité** | 🟡 Bonne | 🟢 Très bonne | 🟢 Excellente |
| **Fraîcheur** | 🟢 Temps réel | 🟢 Temps réel | 🔴 Statique |
| **Volume** | 🟡 Moyen (limité) | 🟡 Moyen (limité) | 🟢 Élevé (123k) |

---

##  Pipeline ETL

### Architecture multi-bases
```
┌──────────────────────────────────────────────────────────────┐
│                     DATA SOURCES                             │
│  [Selenium] → [API HTTP] → [Kaggle Dataset]                  │
└──────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│                  ETL POSTGRESQL + ES                         │
│                                                              │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────┐     │
│  │   Extract   │───►│  Transform   │───►│     Load     │     │
│  │   (CSV)     │    │  (Cleaning)  │    │ (PostgreSQL) │     │
│  └─────────────┘    └──────────────┘    └──────────────┘     │
│                                               │              │
│                                               ▼              │
│                                     ┌──────────────────┐     │
│                                     │  Elasticsearch   │     │
│                                     │   (Indexation)   │     │
│                                     └──────────────────┘     │
└──────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│                    ETL SNOWFLAKE                             │
│                                                              │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────┐     │
│  │     RAW     │───►│     CORE     │───►│   Analytics  │     │
│  │  (Brutes)   │    │ (Normalisé)  │    │  (Requêtes)  │     │
│  └─────────────┘    └──────────────┘    └──────────────┘     │
└──────────────────────────────────────────────────────────────┘
```

### Modèle relationnel
```
categories (1)────(n) companies (1)────(n) reviews (n)────(1) authors
    │                      │                   │
    │                      │                   │
   id                  category_id         author_id
   name                company_id          company_id
```

### PostgreSQL vs Snowflake

| Aspect | PostgreSQL | Snowflake |
|--------|------------|-----------|
| **Usage** | Dev/test local | Production cloud |
| **Volume** | ~158k avis | ~135k avis (après dédup) |
| **Architecture** | Monolithique | RAW → CORE (2 layers) |
| **Performance** | Bonne (local) | Excellente (columnar) |
| **Coût** | Gratuit | Pay-per-use |

---

<p align="left">
  <img src="snoflake.png" alt="Supply Chain - Snowflake" width="800"/>
</p>

##  Machine Learning

### Pipeline d'analyse de sentiment

**Notebook** : `notebooks/sentiment_analysis_3classes.ipynb` (exécuté sur Google Colab GPU)

### Trois modèles comparés

#### 1️ TF-IDF + Logistic Regression
- **Type** : Baseline classique
- **Avantages** : Rapide, interprétable
- **Langues** : Anglais uniquement (traduction nécessaire)

#### 2️ BiLSTM (Deep Learning)
- **Type** : Réseau récurrent bidirectionnel
- **Avantages** : Capture du contexte séquentiel
- **Langues** : Anglais (traduction nécessaire)

#### 3️ XLM-RoBERTa (Transformers)
- **Type** : Modèle multilingue state-of-the-art
- **Modèle** : `cardiffnlp/twitter-xlm-roberta-base-sentiment`
- **Avantages** : Très haute précision, support natif multilingue
- **Langues** : Multilingue (aucune traduction nécessaire)

### Classes de sentiment
```python
def map_rating(rating):
    if rating <= 2: return 0  # Négatif
    elif rating == 3: return 2  # Neutre
    elif rating >= 4: return 1  # Positif
```

### Résultats comparatifs

| Modèle | Précision | Multilingue | Vitesse |
|--------|-----------|-------------|---------|
| TF-IDF + LogReg | ⭐⭐⭐ | ❌ | 🟢 Rapide |
| BiLSTM | ⭐⭐⭐⭐ | ❌ | 🟡 Moyenne |
| XLM-RoBERTa | ⭐⭐⭐⭐⭐ | ✅ | 🟡 Moyenne |

---

##  API et Application

### API FastAPI

**Fichier** : `api/main.py`

#### Endpoints principaux
```bash
GET  /health              # Statut de l'API
GET  /models              # Liste des modèles disponibles
POST /predict             # Prédiction unitaire
POST /predict_batch       # Prédiction par lot
GET  /stats               # Statistiques d'utilisation
```

<p align="left">
  <img src="api.png" alt="Supply Chain - API" width="800"/>
</p>

#### Exemple d'utilisation
```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Service excellent et livraison rapide !",
    "model": "xlm"
  }'
```

**Réponse :**
```json
{
  "sentiment": "positive",
  "confidence": 0.92,
  "model_used": "xlm-roberta"
}
```

<p align="left">
  <img src="model_ihm.png" alt="Supply Chain - Models" width="800"/>
</p>

### Application Streamlit

**Fichier** : `app/app_supply_chain.py`

#### Fonctionnalités

-  **Authentification** : Système utilisateur/admin avec PostgreSQL
-  **Exploration** : Visualisations interactives (Plotly)
-  **Recherche** : Full-text search via Elasticsearch
-  **Analyse ML** : Prédiction de sentiment en temps réel
-  **Dashboards** : KPIs par entreprise/catégorie

#### Identifiants par défaut

- **Admin** : `admin` / `admin123`
- **Utilisateur** : Inscription libre

---

<p align="left">
  <img src="authentification.png" alt="Supply Chain - Authentification" width="800"/>
</p>


##  Automatisation

### DAG Airflow

**Fichier** : `airflow/dags/trustpilot_etl_to_snowflake.py`

#### Architecture du DAG
```
trustpilot_etl_to_snowflake
│
├── scraping_group (parallèle)
│   ├── scrape_companies (9 catégories, max 10 pages)
│   └── scrape_reviews (par entreprise, max 5 pages)
│
├── init_snowflake
│   └── Création WAREHOUSE, DATABASE, SCHEMAS, STAGE
│
├── load_raw_data
│   └── COPY INTO RAW.companies_raw, RAW.reviews_raw
│
└── load_core_data
    └── MERGE INTO CORE.categories, companies, authors, reviews
```
<p align="center">
  <img src="airflow.png" alt="Supply Chain -  DAG Airflow" width="600"/>
</p>

#### Configuration

- **Schedule** : `@daily` (exécution quotidienne)
- **Retries** : 1 tentative après 5 minutes
- **Alertes** : Email en cas d'échec
- **Tags** : `trustpilot`, `scraping`, `snowflake`

```
<p align="left">
  <img src="grafana_scraping.png" alt="Supply Chain -  Grafana" width="800"/>
</p>

### CI/CD GitHub Actions

**Fichier** : `.github/workflows/docker-compose-ci.yml`

#### Pipeline automatisé

1.  **Build** : Construction des images Docker
2.  **Deploy** : Lancement des services
3.  **Tests** : Validation PostgreSQL + API + Test d'integrité
4.  **Cleanup** : Nettoyage automatique

**Déclenchement** : Push sur `master` ou Pull Request

---

##  Installation

### Prérequis

- Docker Desktop 20.10+
- Docker Compose 2.0+
- 8 GB RAM minimum
- 10 GB espace disque

### Installation avec Docker (recommandé)

#### 1️ Cloner le projet
```bash
git clone https://github.com/askeur/supply_chain_satisfaction_clients.git
cd supply_chain_satisfaction_clients
```

#### 2️ Configurer les variables d'environnement
```bash
cat > .env << EOF
# PostgreSQL
POSTGRES_USER=postgres
POSTGRES_PASSWORD=password
POSTGRES_DB=trustpilot_db

# Elasticsearch
ELASTIC_PASSWORD=elastic_password

# Snowflake (optionnel)
SNOWFLAKE_ACCOUNT=votre_account
SNOWFLAKE_USER=votre_user
SNOWFLAKE_PASSWORD=votre_password
SNOWFLAKE_WAREHOUSE=votre_warehouse
EOF
```

#### 3️ Lancer les services
```bash
# Construction des images
docker-compose build

# Démarrage des services
docker-compose up -d

# Vérification
docker-compose ps

# Suivre les logs
docker-compose logs -f
```

#### 4️ Accès aux services

| Service | URL | Identifiants |
|---------|-----|--------------|
| Application Streamlit | http://localhost:8501 | admin / admin123 |
| API FastAPI | http://localhost:8000 | - |
| Documentation API | http://localhost:8000/docs | - |
| Kibana | http://localhost:5601 | - |
| Elasticsearch | http://localhost:9200 | - |
| Airflow | http://localhost:8080 | admin / admin |
| Grafana | http://localhost:3000 | admin / admin |
| PostgreSQL | localhost:5432 | postgres / password |

---

##  Utilisation

### Scraping manuel
```bash
# Méthode API HTTP (recommandée)
python src/scraper/scrape_companies_api.py
python src/scraper/scrape_reviews_api.py

# Méthode Selenium (alternative)
python src/scraper/scrape_companies.py
python src/scraper/scrape_reviews.py

# Dataset Kaggle
python src/scraper/kaggle_companies_reviews.py
```

### Pipeline ETL
```bash
# PostgreSQL + Elasticsearch (local)
python src/etl/etl_postgresql_loader.py

# Snowflake (cloud)
python src/etl/etl_snowflake_loader.py
```

### Tests
```bash
# Tests d'intégration
pytest tests/test_ci.py -v

# Tests API
curl http://localhost:8000/health
curl http://localhost:8000/models
```

---

##  Monitoring

### Grafana Dashboards

**URL** : http://localhost:3000/dashboards

#### Métriques surveillées

**Scraping**
- Durée d'exécution
- Statut (OK/ERROR)
- Nombre de lignes insérées
- Historique des runs

**API**
- Latence moyenne
- Codes HTTP (200, 400, 500)
- Endpoints les plus utilisés
- Taux d'erreurs

### Logs Airflow
```bash
# Logs du scheduler
docker-compose logs -f airflow-scheduler

# Logs d'une tâche spécifique
# Accessible via l'interface Airflow : http://localhost:8080
```

---

##  Dépannage

### Les services ne démarrent pas
```bash
# Vérifier les ports utilisés
netstat -ano | findstr "5432 8000 8501 9200"

# Libérer les ports si nécessaire
docker-compose down
```

### Elasticsearch ne répond pas
```bash
# Vérifier le statut
docker-compose ps elasticsearch

# Redémarrer le service
docker-compose restart elasticsearch

# Vérifier les logs
docker-compose logs elasticsearch
```

### PostgreSQL : erreur de connexion
```bash
# Vérifier le statut
docker-compose ps postgres

# Tester la connexion
docker exec -it postgres psql -U postgres

# Réinitialiser ( perte de données)
docker-compose down -v
docker-compose up -d
```

### Airflow : DAG invisible
```bash
# Vérifier la présence du DAG
ls airflow/dags/

# Redémarrer le scheduler
docker-compose restart airflow-scheduler

# Vérifier les logs
docker-compose logs airflow-scheduler
```

---

##  Structure du projet
```
supply_chain_satisfaction_clients/
│
├── .github/workflows/          # CI/CD GitHub Actions
├── airflow/                    # DAGs et configuration Airflow
├── api/                        # API FastAPI
├── app/                        # Interface Streamlit
    ├── auth/                   # Authentification                       
├── data/                       # Données brutes (CSV)
│   ├── kaggle_raw/
│   ├── scraping_raw/
│   └── scraping_api_raw/
├── docker/                     # Dockerfiles
├── models/                     # Modèles ML entraînés
├── notebooks/                  # Notebooks Jupyter
├── src/   
│   ├── database/               # Schémas SQL et backups
│   ├── etl/                    # Pipelines ETL ( postgres & snowflake)
│   └── scraper/                # Scripts de scraping
├── tests/                      # Tests d'intégration
├── docker-compose.yml          # Orchestration Docker
├── requirements.txt            # Dépendances Python
└── README.md
```

---

##  Licence

Ce projet est développé à des fins pédagogiques dans le cadre de la formation Data Engineer 2025 chez DataScientest.

---

## 👥 Contributeurs

- **Nabila Askeur** - [GitHub](https://github.com/askeur)

**Mentor** : Vincent V

---

##  Documentation complète

Pour plus de détails, consultez le [rapport final PDF](rapport_final.pdf) qui contient :
- Analyse exploratoire des données (EDA)
- Diagrammes UML détaillés
- Résultats complets des modèles ML
- Logs d'exécution des pipelines

---

** Si ce projet vous a été utile, n'hésitez pas à lui donner une étoile !**
