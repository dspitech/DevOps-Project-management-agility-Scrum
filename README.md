# Lab DevOps - Project management & agility - Scrum

### Nom : Lo | Prénom : Pape | Email : pape.lo@estiam.com

<div align="center">

![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)
![Python](https://img.shields.io/badge/Python_3.12-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Groovy](https://img.shields.io/badge/Groovy-4298B8?style=for-the-badge&logo=apachegroovy&logoColor=white)
![ngrok](https://img.shields.io/badge/ngrok-1F1E37?style=for-the-badge&logo=ngrok&logoColor=white)

**Pipeline CI/CD Jenkins automatisé pour SentimentAI · Docker-out-of-Docker**
*TP2 · Formation DevOps · Pipeline as Code · Build · Test · Push*

[Objectif](#contexte-et-objectifs) • [Glossaire](#glossaire--définitions-clés) • [Architecture](#architecture-du-pipeline) • [Jenkins](#partie-1--installer-jenkins-via-docker) • [Jenkinsfile](#partie-2--écrire-le-jenkinsfile) • [Job](#partie-3--créer-et-exécuter-le-job-jenkins) • [Webhook](#partie-4--webhook--déclenchement-automatique) • [Synthèse](#partie-5--questions-de-synthèse)

</div>

# TP 1 - Git & Docker

> **Objectif :** Repository Git public + image Docker fonctionnelle  
> **Outils :** Git, Docker, Docker Compose, Make  
> **Projet :** SentimentAI - API REST d'analyse de sentiments (FastAPI/Python)

---

## Contexte

Vous intégrez **StartupIA**, une entreprise qui développe une plateforme SaaS d'analyse de sentiments pour les avis clients (e-commerce, réseaux sociaux, CRM). Votre mission est de mettre en place l'infrastructure DevOps de l'API **SentimentAI** depuis le dépôt Git jusqu'à l'image Docker, en préparation du pipeline CI/CD automatisé construit dans les TPs suivants.

SentimentAI est une API REST développée en FastAPI/Python. Elle reçoit un texte en entrée, l'analyse et retourne un label (`POSITIF`, `NÉGATIF` ou `NEUTRE`) accompagné d'un score de confiance entre 0 et 1.

### Roadmap de la formation

| TP | Contenu |
|----|---------|
| **TP 1** ← vous êtes ici | Git, Docker Compose, SentimentAI v0.1 |
| TP 2 | Jenkins pipeline - build, test, push |
| TP 3 | SonarQube, Trivy - Qualité & Sécurité |
| TP 4 | Terraform IaC, Docker provider |
| TP 5 | Monitoring, Prometheus, Grafana |

---

## 0. Notions fondamentales

> Cette section présente les concepts clés mobilisés dans ce TP. Elle est à lire avant de commencer les manipulations.

### 0.1 DevOps - Définition

**DevOps** (contraction de *Development* et *Operations*) est une culture et un ensemble de pratiques visant à unifier le développement logiciel et l'exploitation des systèmes. L'objectif est de raccourcir le cycle de livraison tout en améliorant la fiabilité des déploiements.

Les quatre piliers du DevOps sont :

| Pilier | Description |
|--------|-------------|
| **CI** - Intégration Continue | Fusionner fréquemment le code et le tester automatiquement |
| **CD** - Déploiement Continu | Livrer automatiquement chaque version validée en production |
| **Infrastructure as Code** | Gérer les serveurs et réseaux via du code versionné (Terraform) |
| **Monitoring** | Observer le comportement du système en production (Prometheus, Grafana) |

---

### 0.2 Git - Concepts clés

**Git** est un système de contrôle de version distribué créé par Linus Torvalds en 2005. Il permet de suivre l'évolution d'un projet dans le temps, de collaborer à plusieurs sans écraser le travail des autres, et de revenir à n'importe quel état passé du code.

#### Vocabulaire essentiel

| Terme | Définition |
|-------|------------|
| **Repository (repo)** | Dossier versionné contenant tout l'historique du projet sous forme de snapshots |
| **Commit** | Instantané (snapshot) de l'état du projet à un instant T, identifié par un hash SHA-1 unique |
| **Staging area** | Zone intermédiaire où l'on prépare les fichiers avant de les inclure dans un commit (`git add`) |
| **Branch** | Ligne de développement parallèle permettant de travailler en isolation sans toucher `main` |
| **Remote** | Copie distante du repository hébergée sur un serveur (GitHub, GitLab, Bitbucket…) |
| **Clone** | Copie complète d'un repository distant sur la machine locale, historique inclus |
| **Push** | Envoi des commits locaux vers le repository distant |
| **Pull** | Récupération et intégration des commits distants dans la branche locale |

#### Le cycle de vie d'un fichier Git

```
Untracked  ──git add──►  Staged  ──git commit──►  Committed
                                                       │
                                               git push │
                                                       ▼
                                                   Remote (GitHub)
```

#### Conventional Commits

La convention **Conventional Commits** normalise les messages de commit pour les rendre lisibles par des humains et des outils (génération automatique de changelogs, déclenchement de pipelines).

Format : `<type>(<scope optionnel>): <description>`

| Type | Usage |
|------|-------|
| `feat` | Nouvelle fonctionnalité |
| `fix` | Correction de bug |
| `docs` | Modification de documentation |
| `chore` | Tâche de maintenance (CI, deps…) |
| `test` | Ajout ou modification de tests |
| `refactor` | Réécriture de code sans changement de comportement |

Exemple : `feat: initialiser la structure SentimentAI`

#### Tags Git

Un **tag** est un pointeur nommé et permanent vers un commit précis. Il sert à marquer des versions livrables du logiciel (ex. `v0.1.0`, `v1.2.3`).

- **Tag léger** (`git tag v0.1.0`) : simple alias vers un commit, sans métadonnées.
- **Tag annoté** (`git tag -a v0.1.0 -m "message"`) : objet Git complet contenant auteur, date, message et signature GPG optionnelle. **Toujours préférer les tags annotés en production.**

---

### 0.3 Docker - Concepts clés

**Docker** est une plateforme de conteneurisation qui permet d'empaqueter une application avec toutes ses dépendances dans une unité isolée et portable appelée **conteneur**.

#### Différence entre une VM et un conteneur

| | Machine Virtuelle (VM) | Conteneur Docker |
|---|---|---|
| **Isolation** | Système d'exploitation complet | Processus isolé partageant le noyau hôte |
| **Taille** | Plusieurs Go | Quelques dizaines à centaines de Mo |
| **Démarrage** | Minutes | Secondes |
| **Portabilité** | Limitée | Totale : "works everywhere" |
| **Performance** | Overhead hyperviseur | Proche du natif |

#### Vocabulaire essentiel

| Terme | Définition |
|-------|------------|
| **Image** | Modèle immuable et versionné d'un conteneur, construit à partir d'un `Dockerfile`. Analogue à une classe en POO. |
| **Conteneur** | Instance en cours d'exécution d'une image. Analogue à un objet instancié depuis une classe. |
| **Dockerfile** | Fichier texte décrivant les étapes de construction d'une image, du plus stable au plus volatile. |
| **Layer (couche)** | Chaque instruction `RUN`, `COPY`, `ADD` dans un Dockerfile crée une couche immuable mise en cache. |
| **Registry** | Dépôt distant d'images Docker (Docker Hub, GitHub Container Registry, AWS ECR…). |
| **Tag** | Étiquette identifiant une version d'une image (ex. `sentiment-ai:latest`, `sentiment-ai:v0.1.0`). |
| **Volume** | Mécanisme de persistance des données au-delà du cycle de vie d'un conteneur. |
| **Port mapping** | Redirection d'un port hôte vers un port conteneur (`-p 8080:8000` : hôte→conteneur). |

#### Le système de cache par layers

Docker met en cache chaque couche du `Dockerfile`. Si une couche change, **toutes les couches suivantes sont invalidées et recalculées**. C'est pourquoi il faut placer les instructions qui changent rarement en tête du Dockerfile :

```
FROM python:3.11-slim          ← Layer 1 : change jamais (image de base)
COPY requirements.txt .        ← Layer 2 : change rarement
RUN pip install -r req.txt     ← Layer 3 : change rarement → mis en cache !
COPY src/ ./src/               ← Layer 4 : change souvent → invalidation ici
```

Si seul un fichier dans `src/` change, Docker réutilise les layers 1, 2 et 3 depuis le cache et ne recalcule que la layer 4. Sans cette organisation, `pip install` serait relancé à chaque modification de code - ce qui prendrait plusieurs minutes.

---

### 0.4 Docker Compose - Concepts clés

**Docker Compose** est un outil permettant de définir et d'orchestrer des applications multi-conteneurs via un fichier YAML unique (`docker-compose.yml`). Il remplace les longues commandes `docker run` par une configuration déclarative.

#### Concepts clés

| Concept | Définition |
|---------|------------|
| **Service** | Un conteneur défini dans `docker-compose.yml`, avec ses ports, volumes, variables d'environnement et dépendances. |
| **Réseau** | Canal de communication isolé entre services. Les conteneurs d'un même réseau se découvrent par leur nom de service (DNS interne Docker). |
| **Volume** | Stockage persistant partagé entre services ou entre le conteneur et l'hôte. |
| **Healthcheck** | Sonde périodique qui vérifie si un service fonctionne correctement. Docker marque le conteneur `healthy` ou `unhealthy`. |
| **`restart: unless-stopped`** | Politique de redémarrage automatique du conteneur si il plante, sauf si arrêté manuellement. |

---

### 0.5 FastAPI et Pydantic - Concepts clés

**FastAPI** est un framework web Python moderne conçu pour créer des API REST performantes. Il repose sur les *type hints* Python et génère automatiquement la documentation interactive (Swagger UI, accessible sur `/docs`).

**Pydantic** est une bibliothèque de validation de données basée sur les types Python. Elle garantit que les données entrantes respectent un schéma précis : types, longueurs, contraintes. En cas de violation, elle retourne automatiquement une erreur HTTP `422 Unprocessable Entity`.

**Uvicorn** est un serveur ASGI (Asynchronous Server Gateway Interface) léger et performant, utilisé pour servir les applications FastAPI en production.

---

### 0.6 Makefile - Concepts clés

Un **Makefile** est un fichier de recettes qui automatise des tâches répétitives via la commande `make`. Il sert à la fois d'outil d'automatisation et de documentation des commandes du projet.

- **Cible** (`target`) : nom de la tâche à exécuter (ex. `build`, `test`).
- **Recette** : commandes shell exécutées quand la cible est appelée (indentées avec une **tabulation**, jamais des espaces).
- **`.PHONY`** : déclaration indiquant que les cibles ne correspondent pas à des fichiers réels - évite des conflits si un fichier du même nom existe.

---

## Déploiement

### Déployer la VM Linux Ubuntu sur Azure

- Se connecter à Azure 
![image](https://hackmd.io/_uploads/BykOE7bfGl.png)
- Lancer le Cloud Shell et choisir PowerShell
![image](https://hackmd.io/_uploads/B18cNQ-ffe.png)
![image](https://hackmd.io/_uploads/H1eTVQ-GMg.png)
- Déployer la VM
```powershell
# Cloner La configuration de la VM
git clone https://github.com/dspitech/DevOps-VM-Ubuntu-Terraform-Azure.git
```
![image](https://hackmd.io/_uploads/rk4_HQ-Gfx.png)

```powershell
# Se placer dans le répertoire du projet
cd DevOps-VM-Ubuntu-Terraform-Azure
```
![image](https://hackmd.io/_uploads/ryNqBXWzzg.png)
![image](https://hackmd.io/_uploads/Bym3HX-GMg.png)
![image](https://hackmd.io/_uploads/r1rCSX-fze.png)


```powershell
chmod +x ./setup-backend.sh
./setup-backend.sh
```
![image](https://hackmd.io/_uploads/SkTHL7WGze.png)
![image](https://hackmd.io/_uploads/HkFP87bGfe.png)


```powershell
# vérifier le nom du Storage Account créé
az storage account list --resource-group OpenLab-TFState-RG --query "[].name" -o tsv
# Puis mettez-le dans backend.tf : storage_account_name = "openlabtfstate14523"   # ← le nom réel
```
![image](https://hackmd.io/_uploads/By_FIQZzzx.png)
![image](https://hackmd.io/_uploads/BkoTLXWfMe.png)


```powershell
# Initialiser Terraform (téléchargement des providers)
# Formater les fichiers Terraform selon les conventions
# Vérifier la syntaxe et la cohérence de la configuration
# Afficher les ressources qui vont être créées/modifiées
# Déployer l'infrastructure sans confirmation interactive

terraform init && terraform fmt && terraform validate && terraform plan && terraform apply -auto-approve
```
![image](https://hackmd.io/_uploads/rkmWvmZffe.png)
![image](https://hackmd.io/_uploads/S1CXv7-zGe.png)
![image](https://hackmd.io/_uploads/ByDiP7ZGMl.png)
![image](https://hackmd.io/_uploads/rJFfu7WGGg.png)

```powershell
# Télécharger la clé privée SSH générée par Terraform
download ./openlab_rsa
```
![image](https://hackmd.io/_uploads/ByH6PmbGMx.png)

```powershell
# Se connecter à la machine distante via SSH
# Sous Windows PowerShell :
ssh -i "C:\Users\dev\Downloads\openlab_rsa" labadmin@4.225.216.24
```
![image](https://hackmd.io/_uploads/S1sLuQ-GMx.png)


## 1. Git - Initialiser le projet

### Prérequis

- Docker (avec Docker Compose)
- Make
- Git

```bash
docker --version && docker compose version && make --version | head -n 1 && git --version
```
![image](https://hackmd.io/_uploads/rJDJYmbffx.png)

Git est le système de contrôle de version utilisé tout au long de la formation. Chaque modification du code est tracée sous forme de commit. GitHub sera l'hébergeur distant : c'est là que Jenkins ira chercher le code à chaque déclenchement du pipeline.

### 1.1 Créer le repository distant

Connectez-vous à GitHub et créez un nouveau repository **public** avec les paramètres suivants :

| Champ | Valeur attendue |
|-------|-----------------|
| Nom | `sentiment-ai` (exactement, sans majuscule) |
| Visibilité | Public |
| Initialisation | Cocher : README + `.gitignore` Python + Licence MIT |

```bash
# Se connecter à Github
gh auth login
```
![image](https://hackmd.io/_uploads/rkZMYQWfGx.png)
![image](https://hackmd.io/_uploads/Bk_mK7-Mfe.png)
![image](https://hackmd.io/_uploads/HkX4t7Zffx.png)
![image](https://hackmd.io/_uploads/HkMrYmZfMg.png)
![image](https://hackmd.io/_uploads/rycIt7Zzfx.png)
![image](https://hackmd.io/_uploads/r1tdt7WGzl.png)
![image](https://hackmd.io/_uploads/H1uFYmbMzg.png)
![image](https://hackmd.io/_uploads/ByV2FQbfze.png)

```bash
# Créer le repo
gh repo create sentiment-ai \
  --public \
  --license MIT \
  --gitignore Python
```
> **Pourquoi cocher le `.gitignore` Python dès la création ?**  
> GitHub génère un fichier préconfiguré qui exclut déjà `__pycache__/`, `.venv/`, `*.pyc` et autres artefacts Python indésirables, évitant toute configuration manuelle.
![image](https://hackmd.io/_uploads/rk9CY7bfMe.png)
![image](https://hackmd.io/_uploads/S1Lx9XbfGl.png)
![image](https://hackmd.io/_uploads/HyPbqQZGze.png)

### 1.2 Cloner et configurer

Une fois le repository créé, clonez-le localement. Remplacez `VOTRE_PSEUDO` par votre identifiant GitHub.

```bash
# Cloner le repo localement
git clone https://github.com/dspitech/sentiment-ai.git
```
![image](https://hackmd.io/_uploads/BkeEcm-zfe.png)

```bash
# Configurer votre identité Git globale (si pas encore fait)
git config --global user.name "Pape Lo"
git config --global user.email "pape.lo@estiam.com"

```bash
# Vérification
git config --global --list
```
![image](https://hackmd.io/_uploads/SJ8ucQbzze.png)

```bash
# Vérifier l'état du dépôt : doit afficher "nothing to commit"
git status
```

![image](https://hackmd.io/_uploads/ByxVimZzze.png)

```bash
# Consulter l'historique : un seul commit initial (le README)
git log --oneline
```
![image](https://hackmd.io/_uploads/HJdBjmWGMg.png)

> **Identité Git et traçabilité**  
> La configuration `user.name` et `user.email` est essentielle en contexte professionnel. En entreprise, ces informations doivent correspondre à votre identité réelle - elles apparaissent dans chaque commit de l'historique.

### 1.3 Créer la structure du projet

L'arborescence ci-dessous respecte les conventions Python (séparation `src/` et `tests/`) et anticipe les besoins des TPs suivants : `.github/workflows/` accueillera les GitHub Actions, et `Dockerfile` / `docker-compose.yml` seront remplis dans la suite de ce TP.

```bash
# Créer les dossiers nécessaires
mkdir -p src tests .github/workflows

# Créer les fichiers Python (vides pour l'instant)
touch src/__init__.py src/main.py src/model.py src/schemas.py
touch tests/__init__.py tests/test_api.py

# Créer les fichiers de configuration DevOps
touch requirements.txt Dockerfile docker-compose.yml .dockerignore Makefile

# Vérifier la structure complète créée
find . -not -path './.git/*' | sort
```

**Structure attendue :**

```
.
├── .dockerignore
├── .github/
│   └── workflows/
├── .gitignore
├── Dockerfile
├── Makefile
├── docker-compose.yml
├── requirements.txt
├── src/
│   ├── __init__.py
│   ├── main.py
│   ├── model.py
│   └── schemas.py
└── tests/
    ├── __init__.py
    └── test_api.py
```
![image](https://hackmd.io/_uploads/BJijs7bzzl.png)

### 1.4 Remplir les fichiers Python

Le modèle utilisé est volontairement simplifié (liste de mots-clés). Cette approche naïve suffit pour valider le pipeline - un vrai modèle ML pourrait le remplacer sans changer l'interface de l'API.

#### 1.4.1 `src/schemas.py` - Modèles de données Pydantic

```bash
cat > src/schemas.py <<'EOF'
from pydantic import BaseModel, Field
from typing import Literal


class PredictionRequest(BaseModel):
    # Le texte à analyser : obligatoire, entre 1 et 5000 caractères
    text: str = Field(..., min_length=1, max_length=5000)


class PredictionResponse(BaseModel):
    # Le label retourné est contraint à 3 valeurs possibles
    label: Literal["POSITIVE", "NEGATIVE", "NEUTRAL"]
    score: float  # Score de confiance entre 0.0 et 1.0
    text: str  # Texte original retourné pour traçabilité
EOF
```
![image](https://hackmd.io/_uploads/B1az3Q-MMl.png)


#### 1.4.2 `src/model.py` - Modèle de sentiment simplifié

```bash
cat > src/model.py <<'EOF'
class SentimentModel:
    def __init__(self):
        # Ce message sera visible dans "docker logs sentiment"
        print("[SentimentModel] Modèle chargé")

    def predict(self, text: str) -> dict:
        text_lower = text.lower()

        positive_words = [
            "bien",
            "super",
            "excellent",
            "parfait",
            "bon",
            "aime",
            "adore"
        ]

        negative_words = [
            "mal",
            "nul",
            "horrible",
            "mauvais",
            "déteste",
            "pire"
        ]

        pos = sum(1 for w in positive_words if w in text_lower)
        neg = sum(1 for w in negative_words if w in text_lower)

        if pos > neg:
            return {
                "label": "POSITIVE",
                "score": round(0.6 + 0.1 * pos, 2),
                "text": text
            }

        elif neg > pos:
            return {
                "label": "NEGATIVE",
                "score": round(0.6 + 0.1 * neg, 2),
                "text": text
            }

        return {
            "label": "NEUTRAL",
            "score": 0.5,
            "text": text
        }
EOF
```
![image](https://hackmd.io/_uploads/BkaL27bfze.png)

#### 1.4.3 `src/main.py` - Application FastAPI

`main.py` est le point d'entrée de l'API. Il expose deux endpoints :
- `/health` pour les healthchecks Docker et Kubernetes
- `/predict` pour les prédictions de sentiment

```bash
cat > src/main.py <<'EOF'
from fastapi import FastAPI
from src.schemas import PredictionRequest, PredictionResponse
from src.model import SentimentModel


app = FastAPI(title="SentimentAI", version="0.1.0")


# Le modèle est chargé une seule fois au démarrage du serveur
model = SentimentModel()


@app.get("/health")
def health():
    """
    Endpoint de healthcheck utilisé par Docker et les load balancers.
    """
    return {"status": "ok"}


@app.post("/predict", response_model=PredictionResponse)
def predict(request: PredictionRequest):
    """
    Analyse le sentiment du texte fourni et retourne un label + score.
    """
    return model.predict(request.text)
EOF
```
![image](https://hackmd.io/_uploads/rkVi27bMzx.png)

#### 1.4.4 `tests/test_api.py` - Tests unitaires et d'intégration

```bash
cat > tests/test_api.py <<'EOF'
from fastapi.testclient import TestClient
from src.main import app


client = TestClient(app)


def test_health():
    """
    Vérifie que l'endpoint /health répond avec status 200.
    """
    r = client.get("/health")
    assert r.status_code == 200


def test_predict_positive():
    """
    Vérifie qu'une prédiction retourne la bonne structure de réponse.
    """
    r = client.post("/predict", json={"text": "Ce produit est excellent !"})

    assert r.status_code == 200

    data = r.json()

    assert data["label"] in ["POSITIVE", "NEGATIVE", "NEUTRAL"]
    assert 0 <= data["score"] <= 1


def test_predict_empty_fails():
    """
    Vérifie que Pydantic rejette un texte vide avec une erreur 422.
    """
    r = client.post("/predict", json={"text": ""})

    assert r.status_code == 422
def test_predict_negative():
    """
    Vérifie qu'un texte négatif retourne le label NEGATIVE.
    """
    r = client.post("/predict", json={"text": "Ce produit est horrible"})

    assert r.status_code == 200

    data = r.json()

    assert data["label"] == "NEGATIVE"
    assert 0 <= data["score"] <= 1


def test_predict_neutral():
    """
    Vérifie qu'un texte sans mot positif ou négatif retourne NEUTRAL.
    """
    r = client.post("/predict", json={"text": "La météo est aujourd'hui"})

    assert r.status_code == 200

    data = r.json()

    assert data["label"] == "NEUTRAL"
    assert data["score"] == 0.5
EOF
```
![image](https://hackmd.io/_uploads/B1iA3mbMGg.png)


#### 1.4.5 `requirements.txt` - Dépendances Python

```bash
cat > requirements.txt <<'EOF'
fastapi==0.109.0
uvicorn==0.27.0
pydantic==2.5.3
pytest==7.4.4
pytest-cov==4.1.0
httpx==0.26.0
EOF
```
![image](https://hackmd.io/_uploads/HkgMamZGMg.png)

> Les versions sont **épinglées** (numéros exacts). C'est une bonne pratique DevOps : elle garantit que le même environnement est recréé identiquement en local, dans Docker, dans Jenkins et en production.
![image](https://hackmd.io/_uploads/HktBpX-zGg.png)

### 1.5 Premier commit et push

```bash
# Ajouter tous les fichiers au staging
git add .

# Vérifier ce qui va être commité avant de valider
git diff --staged --stat

# Créer le commit avec un message conventionnel
git commit -m "feat: initialiser la structure SentimentAI"

# Pousser vers GitHub
git push origin main

# Vérifier que le commit apparaît bien dans l'historique
git log --oneline
```
![image](https://hackmd.io/_uploads/rJYcpQZfGx.png)
![image](https://hackmd.io/_uploads/r1g66m-GMl.png)
![image](https://hackmd.io/_uploads/r1d0pXbGfe.png)
![image](https://hackmd.io/_uploads/r16gCm-Mzx.png)

---

## 2. Docker - Conteneuriser l'API

La conteneurisation permet d'empaqueter l'application avec toutes ses dépendances dans une image Docker reproductible. Une fois conteneurisée, SentimentAI fonctionnera de manière identique en local, dans Jenkins lors des tests, et en production. C'est le principe fondamental du DevOps : *"works on my machine"* devient *"works everywhere"*.

### 2.1 Écrire le Dockerfile

L'ordre des instructions est crucial pour les performances de cache (voir section 0.3). On copie `requirements.txt` **avant** le code source.

```
nano Dockerfile
```

```dockerfile
FROM python:3.11-slim

# Définir le répertoire de travail dans le conteneur
WORKDIR /app

# Étape 1 : copier UNIQUEMENT le fichier de dépendances
# Cette couche sera mise en cache tant que requirements.txt ne change pas
COPY requirements.txt .

# Étape 2 : installer les dépendances (couche mise en cache)
RUN pip install --no-cache-dir -r requirements.txt

# Étape 3 : copier le code source (invalidé à chaque modification du code)
COPY src/ ./src/
COPY tests/ ./tests/

# Documenter le port utilisé par l'application
EXPOSE 8000

# Commande de démarrage du serveur Uvicorn
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```
![image](https://hackmd.io/_uploads/SkVwCQWGGe.png)

> **Pourquoi `python:3.11-slim` et non `python:3.11` ?**  
> L'image `slim` pèse ~150 Mo contre ~900 Mo pour l'image complète. Une image plus petite = moins de surface d'attaque, un transfert plus rapide vers le registry et un démarrage plus rapide.

### 2.2 Écrire le .dockerignore

Le `.dockerignore` exclut les fichiers inutiles ou sensibles du contexte de build, réduisant le temps de transfert et le risque de fuites de secrets.
![image](https://hackmd.io/_uploads/BkWykVZGGg.png)

```bash
cat > .dockerignore <<'EOF'
# Exclure le dépôt Git (inutile dans l'image, très volumineux)
.git/
.github/

# Exclure les fichiers compilés Python (régénérés automatiquement)
__pycache__/
*.pyc
*.pyo

# Exclure les artefacts de tests et de couverture
.pytest_cache/
htmlcov/
coverage.xml

# Exclure les secrets et fichiers d'environnement locaux
.env
.env.*

# Exclure les fichiers Terraform (ajoutés en TP4)
*.tfstate
.terraform/
EOF
```
![image](https://hackmd.io/_uploads/r1ABJ4Zffg.png)

### 2.3 Builder et tester l'image

```bash
# Construire l'image et la tagger "sentiment-ai:latest"
docker build -t sentiment-ai:latest .
```
![image](https://hackmd.io/_uploads/B13iyNbMMg.png)
![image](https://hackmd.io/_uploads/rkTpy4ZGzl.png)


```bash
# Lancer le conteneur en arrière-plan (-d) avec redirection de port
docker run -d --name sentiment -p 8080:8000 sentiment-ai:latest
```
![image](https://hackmd.io/_uploads/SJWllVbMGx.png)

```bash
# Vérifier que le conteneur est bien démarré
docker ps
```
![image](https://hackmd.io/_uploads/SJvZeEZGMe.png)

```bash
# Tester l'endpoint de healthcheck
curl http://localhost:8080/health
# Réponse attendue : {"status":"ok"}
```

![image](https://hackmd.io/_uploads/SJqml4ZGfl.png)

```bash
# Tester une prédiction de sentiment
curl -X POST http://localhost:8080/predict \
  -H "Content-Type: application/json" \
  -d '{"text": "Ce produit est excellent !"}'
# Réponse attendue : {"label":"POSITIVE","score":0.7,"text":"..."}
```
![image](https://hackmd.io/_uploads/SkKUlEZGMg.png)


```bash
# Consulter les logs du conteneur pour vérifier le démarrage
docker logs sentiment
```
![image](https://hackmd.io/_uploads/SynPxVWMzl.png)


```bash
# Nettoyer : arrêter et supprimer le conteneur
docker stop sentiment && docker rm sentiment
```
![image](https://hackmd.io/_uploads/ry9cg4WGGl.png)

> **`docker stop` vs `docker rm`**  
> `docker stop` envoie SIGTERM et attend un arrêt propre (timeout 10s, puis SIGKILL). Le conteneur reste dans l'état `stopped`. `docker rm` le supprime définitivement. Toujours `stop` avant `rm`, ou `docker rm -f` pour forcer.

---

## 3. Docker Compose

Docker Compose centralise la configuration de toute la stack dans un fichier YAML unique, remplaçant les longues commandes `docker run`. C'est le standard pour orchestrer les environnements de développement et de staging.

### 3.1 Écrire le docker-compose.yml

```bash
cat > docker-compose.yml <<'EOF'
version: '3.9'

services:
  sentiment-ai:
    build: .
    container_name: sentiment-staging
    ports:
      - "8080:8000"   # hôte:conteneur
    environment:
      - ENV=development
    networks:
      - cicd-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s      # fréquence des vérifications
      timeout: 10s       # délai max avant échec
      retries: 3         # nombre d'échecs avant "unhealthy"
      start_period: 10s  # délai de grâce au démarrage

# Réseau dédié : Jenkins et SonarQube le rejoindront dans les TPs suivants
networks:
  cicd-network:
    driver: bridge
EOF
```
![image](https://hackmd.io/_uploads/H1jAgV-Mze.png)


> **Pourquoi un réseau `cicd-network` dédié ?**  
> Les conteneurs d'un même réseau Docker nommé se découvrent par leur nom de service (DNS interne). Dans les TPs suivants, Jenkins contactera SonarQube via `http://sonarqube:9000` plutôt qu'une IP instable.

### 3.2 Lancer, tester et arrêter la stack

```bash
# Démarrer la stack en arrière-plan (-d = detached)
docker compose up -d
```
![image](https://hackmd.io/_uploads/HJrmbNWGMg.png)


```bash
# Vérifier l'état des services (attendre ~30s pour le healthcheck)
docker compose ps
```
![image](https://hackmd.io/_uploads/HyWHW4bMzl.png)


```bash
# Tester que l'API répond correctement
curl http://localhost:8080/health
```
![image](https://hackmd.io/_uploads/rJsLZNWGzx.png)


```bash
# Suivre les logs en temps réel (Ctrl+C pour quitter)
docker compose logs -f sentiment-ai
```
![image](https://hackmd.io/_uploads/ryyKW4WMGe.png)


```bash
# Arrêter et supprimer les conteneurs (réseau et volumes conservés)
docker compose down
```

```bash
# Pour tout supprimer y compris les volumes :
docker compose down -v
```
![image](https://hackmd.io/_uploads/BkX6b4-GGl.png)

---

## 4. Makefile & Tag Git

Un Makefile automatise les tâches répétitives et sert de **documentation vivante** des commandes du projet. Tout nouveau développeur comprend immédiatement comment builder, tester et déployer en lisant les cibles.

### 4.1 Écrire le Makefile

>  **Attention :** l'indentation des recettes doit utiliser des **tabulations** (`Tab`), pas des espaces.

```bash
cat > Makefile <<'EOF'
IMAGE_NAME = sentiment-ai
PORT       = 8080

.PHONY: build run test stop clean tag

build:
	docker build -t $(IMAGE_NAME):latest .

run:
	docker compose up -d

# Lance les tests DANS le conteneur Docker
test:
	docker run --rm \
		-v $(PWD):/app \
		-w /app \
		$(IMAGE_NAME):latest \
		pytest tests/ -v --cov=src --cov-report=term-missing

stop:
	docker compose down

clean:
	docker compose down
	docker rmi $(IMAGE_NAME):latest || true

tag:
	git tag -a v0.1.0 -m "Initial SentimentAI release"
	git push origin v0.1.0
EOF
```
![image](https://hackmd.io/_uploads/ry57fEbMze.png)

> **Pourquoi tester dans le conteneur ?**  
> Cela garantit que les tests s'exécutent dans le même environnement que la production. Un test qui passe localement mais échoue dans Docker révèle une dépendance manquante ou une hypothèse incorrecte sur l'environnement.

### 4.2 Lancer les tests

```bash
make test
```
![image](https://hackmd.io/_uploads/SyP8S4-zfx.png)


### 4.3 Créer le tag de version v0.1.0

```bash
# Créer le tag v0.1.0 annoté et le pousser vers GitHub
make tag

# Vérifier que le tag est bien créé localement
git tag -l

# Vérifier que le tag apparaît dans l'historique
git log --oneline --decorate
```
![image](https://hackmd.io/_uploads/HJH0fEWzMg.png)
![image](https://hackmd.io/_uploads/SynkmEbMGg.png)
![image](https://hackmd.io/_uploads/SkzX7N-MMx.png)
![image](https://hackmd.io/_uploads/Bkm4XNWGMx.png)


---

## 5. Réponses aux questions

### Question 1.1 - Rôle du `.gitignore` et `__pycache__/`

**Rôle du `.gitignore`**

Le fichier `.gitignore` liste les fichiers et dossiers que Git doit ignorer, c'est-à-dire ne jamais tracer ni inclure dans les commits. Il permet de maintenir le repository propre en excluant les fichiers générés automatiquement, les secrets (`.env`), les artefacts de compilation et les fichiers propres à l'environnement local de chaque développeur.

**Pourquoi ne pas commiter `__pycache__/` ?**

Le dossier `__pycache__/` contient les fichiers `.pyc`, qui sont des versions compilées en bytecode des modules Python. Ces fichiers sont générés automatiquement par l'interpréteur à chaque exécution et sont propres à la version locale de Python — ils diffèrent d'une machine à l'autre. Les commiter polluerait l'historique Git avec des fichiers binaires inutiles, provoquerait des conflits de merge constants entre développeurs, et alourdirait le repository sans aucune valeur ajoutée.

---

### Question 1.2 - `git add .` vs `git add -p`

**`git add .`**

Ajoute **tous** les fichiers modifiés et non-trackés du répertoire courant (et de ses sous-dossiers) à la staging area en une seule commande. C'est rapide mais sans discernement : on commit tout ce qui a changé.

**`git add -p` (patch mode)**

Lance un mode interactif qui présente chaque modification (hunk) une par une et demande si on souhaite l'inclure dans le prochain commit (`y` = yes, `n` = no, `s` = split, `e` = edit). Cela permet de créer des commits **atomiques et cohérents**, où chaque commit ne contient qu'un seul changement logique.

**Quand préférer `git add -p` ?**

On préfère `git add -p` lorsqu'on a travaillé sur plusieurs fonctionnalités ou corrections en parallèle et que l'on souhaite les séparer en commits distincts. Par exemple, si un fichier contient à la fois un correctif de bug et le début d'une nouvelle feature, `git add -p` permet de commiter uniquement le correctif dans un premier commit (`fix:`), puis la feature dans un second (`feat:`). C'est une pratique essentielle pour maintenir un historique Git lisible et faciliter les revues de code (`git blame`, `git bisect`).

---

### Question 2.1 - Cache Docker lors du build

**Couches mises en cache (`CACHED`) :**

Lors d'un second `docker build` sans modification, les couches suivantes sont lues depuis le cache :
- `FROM python:3.11-slim` - l'image de base est déjà téléchargée localement.
- `COPY requirements.txt .` - le fichier n'a pas changé, le hash est identique.
- `RUN pip install --no-cache-dir -r requirements.txt` - aucune dépendance n'a changé, Docker réutilise la couche buildée précédemment. C'est la couche la plus longue à recalculer (~30s à plusieurs minutes).

**Couches recalculées :**

- `COPY src/ ./src/` et `COPY tests/ ./tests/` - si un fichier source change, ces layers sont invalidés et tout ce qui suit est recalculé.

**Pourquoi c'est important en pratique ?**

Sans cette organisation, la moindre modification d'un fichier `.py` déclencherait un `pip install` complet à chaque build, rendant le pipeline CI/CD inutilisable en termes de temps.

---

### Question 2.2 - Invalidation du cache après modification d'un fichier Python

**Observation lors d'un second build sans modification :**

Toutes les couches affichent `CACHED` dans la sortie. Le build est quasi-instantané car Docker n'exécute aucune instruction - il reconstruit l'image entièrement depuis le cache.

**Quelle instruction perd le cache si on modifie un fichier dans `src/` ?**

L'instruction `COPY src/ ./src/` est invalidée. Docker calcule un hash de tous les fichiers copiés : si un seul fichier change, le hash diffère et la couche est recalculée. Puisqu'elle est invalidée, **toutes les instructions suivantes** le sont aussi - ici seulement `COPY tests/` et `CMD`, qui sont rapides.

**Explication par le principe des layers :**

Les layers Docker sont immuables et empilés de manière séquentielle. Le cache d'une layer est valide uniquement si :
1. L'instruction Dockerfile est identique à la précédente exécution, **ET**
2. La layer parente est elle-même issue du cache.

Dès qu'une layer est invalidée, toute la chaîne en dessous l'est aussi - c'est le comportement déterministe du cache Docker. C'est la raison fondamentale pour laquelle on structure le Dockerfile du **plus stable au plus volatile** : `FROM` → dépendances → code source.

---

### Question 3.1 - Utilité du healthcheck en déploiement automatisé

**Définition du healthcheck ici :**

Toutes les 30 secondes, Docker exécute `curl -f http://localhost:8000/health` à l'intérieur du conteneur. Si la commande réussit (code HTTP 200), le conteneur est marqué `healthy`. Après 3 échecs consécutifs, il passe à `unhealthy`.

**Pourquoi c'est indispensable en déploiement automatisé ?**

Sans healthcheck, Docker considère qu'un conteneur est opérationnel dès lors que son processus principal a démarré - même si l'application a planté silencieusement, est bloquée dans une boucle infinie, ou si le serveur web n'écoute pas encore. Dans un pipeline CI/CD ou avec un orchestrateur comme Kubernetes, cela signifie que du trafic pourrait être redirigé vers un conteneur non fonctionnel, provoquant des erreurs 502 ou 503 pour les utilisateurs finaux.

Le healthcheck permet à Docker (et aux orchestrateurs) de distinguer « le processus tourne » de « l'application répond correctement ». Un outil comme Kubernetes peut ainsi :
- Retarder l'envoi de trafic jusqu'à ce que le conteneur soit `healthy` (readiness probe).
- Redémarrer automatiquement un conteneur devenu `unhealthy` (liveness probe).
- Éviter de déployer une version défectueuse en production (rolling update avec vérification).

---

### Question 4.1 - Résultats des tests et coverage

**Résultat attendu de `make test` :**

![image](https://hackmd.io/_uploads/HyDBr4bGzl.png)


**Ce que couvre chaque test :**

| Test | Ce qu'il vérifie |
|------|-----------------|
| `test_health` | L'endpoint `/health` répond HTTP 200 - prouve que FastAPI démarre correctement |
| `test_predict_positive` | `/predict` retourne un JSON valide avec `label` et `score` dans les bonnes plages |
| `test_predict_empty_fails` | Pydantic rejette un texte vide avec HTTP 422 - valide la validation des entrées |

**En cas d'échec :** la cause la plus fréquente est un `__init__.py` manquant dans `src/` ou `tests/`, qui empêche Python de les traiter comme des packages importables.

---

### Question 4.2 - Tag annoté vs tag léger

**Tag léger (`git tag v0.1.0`) :**

Un tag léger est un simple alias (pointeur) vers un commit. Il ne stocke que le hash du commit cible - aucune métadonnée supplémentaire. Il ne crée pas d'objet Git dédié et n'apparaît pas dans `git log` avec `--decorate` de la même façon.

**Tag annoté (`git tag -a v0.1.0 -m "message"`) :**

Un tag annoté est un **objet Git à part entière**, stocké dans la base de données Git avec :
- Le nom du créateur et son email
- La date et l'heure de création
- Un message descriptif
- Une signature GPG optionnelle pour authentifier la version

**Pourquoi préférer les tags annotés en production ?**

| Critère | Tag léger | Tag annoté |
|---------|-----------|------------|
| Métadonnées (auteur, date) | ✗ | ✓ |
| Message de release | ✗ | ✓ |
| Signature GPG possible | ✗ | ✓ |
| Visible dans `git describe` | Parfois | ✓ |
| Interprété comme release par GitHub | ✗ | ✓ |

En production, les tags annotés permettent de savoir **qui** a validé la mise en production, **quand** et **pourquoi**. Ils sont reconnus par GitHub comme des releases officielles (affichées dans l'onglet *Releases*) et peuvent être signés cryptographiquement pour garantir leur authenticité. Dans les pipelines CI/CD, Jenkins et GitHub Actions peuvent déclencher des déploiements spécifiques uniquement sur les tags annotés, offrant ainsi un contrôle plus fin sur ce qui part en production.

---

## 6. Récapitulatif - À rendre

| Livrable | Critère de validation |
|----------|-----------------------|
| Repository GitHub `sentiment-ai` public | Structure complète visible sur GitHub |
| Tag `v0.1.0` | Visible dans `git tag -l` et sur GitHub (onglet Releases) |
| `Dockerfile` + `.dockerignore` + `docker-compose.yml` | Commités dans le repo |
| `Makefile` | Cibles `build`, `run`, `test`, `clean`, `tag` fonctionnelles |
| Screenshot `make test` | 3 tests verts avec coverage |
| Screenshot `docker compose ps` | Conteneur en statut `healthy` |
| Réponses aux questions | Sections 5.1 à 5.7 complétées |

### Prérequis pour le TP2

Le TP2 installera Jenkins via Docker et créera un pipeline Groovy (`Jenkinsfile`) complet avec les stages : **Checkout → Lint → Build → Test → Push**. Avant de passer au TP2, vérifiez que :

- Votre repo `sentiment-ai` est accessible **publiquement** sur GitHub
- La commande `make test` passe avec **3 tests verts**
- L'image Docker se build sans erreur avec `docker build`

---

# TP 2 - Jenkins Pipeline CI/CD

> **Formation DevOps** · Pipeline automatisé build / test / push pour le projet **SentimentAI**

---

## Contexte et objectifs

**SentimentAI** est désormais versionnée dans Git et conteneurisée avec Docker (TP1). Ce TP automatise le cycle build / test / push :

À chaque `git push`, Jenkins récupère le code, le lint, construit l'image Docker, lance les tests et pousse l'image vers le registry si l'on est sur la branche `main`.

| Objectif | Livrable attendu |
|---|---|
| Installer Jenkins en conteneur (DooD) | Jenkins accessible sur `localhost:8080` |
| Écrire un pipeline as code | `Jenkinsfile` commité à la racine du repo |
| Automatiser le déclenchement | Webhook GitHub ou Poll SCM fonctionnel |
| Pousser l'image vers le registry | Image visible dans GitHub Packages (`ghcr.io`) |

### Roadmap de la formation

```
TP1              TP2              TP3              TP4              TP5
Git + Docker  →  Jenkins      →  SonarQube    →  Terraform    →  Monitoring
+ Compose        Pipeline         + Trivy          IaC Docker      Prometheus
SentimentAI      build+test       Qualité &                        + Grafana
v0.1             +push            Sécurité
```

---

## Définitions clés

### Jenkins

**Jenkins** est un serveur d'automatisation open-source écrit en Java. Il permet de mettre en place des pipelines CI/CD (Intégration Continue / Déploiement Continu) qui s'exécutent automatiquement à chaque modification du code source.

### CI/CD

| Terme | Définition |
|---|---|
| **CI** - Intégration Continue | Pratique qui consiste à intégrer régulièrement le code dans un dépôt partagé, accompagné de tests automatiques. |
| **CD** - Déploiement Continu | Extension de la CI : chaque build validé est automatiquement déployé en production ou vers un registry. |
| **Pipeline** | Séquence d'étapes automatisées (stages) définissant le cycle de vie d'un build. |
| **Stage** | Étape individuelle d'un pipeline (ex. : Lint, Build, Test, Push). |

### Jenkinsfile

Fichier texte au format **Groovy** versionné dans le dépôt Git qui décrit le pipeline Jenkins. C'est le principe du **Pipeline as Code** : chaque modification du pipeline est traçable, reviewable en Pull Request et rollbackable comme n'importe quel fichier de code.

### Docker-out-of-Docker (DooD)

Technique permettant à un **conteneur Docker** (ici Jenkins) d'exécuter des commandes `docker build` en montant le socket Docker de l'hôte (`/var/run/docker.sock`). Jenkins ne fait pas tourner son propre daemon Docker, il pilote celui de la machine hôte.

```
Machine hôte
├── Docker daemon  ←── socket : /var/run/docker.sock
└── Conteneur Jenkins
    └── monte /var/run/docker.sock  →  peut faire docker build, docker push…
```

> **Risque de sécurité** : monter le socket Docker donne au conteneur Jenkins les mêmes droits que `root` sur l'hôte. En production, on préférera [Kaniko](https://github.com/GoogleContainerTools/kaniko) ou [Buildah](https://buildah.io/), qui ne nécessitent pas de daemon Docker.

### Agent Jenkins

> Un **agent** est le nœud sur lequel s'exécute le pipeline.

| Déclaration | Comportement |
|---|---|
| `agent any` | S'exécute sur n'importe quel agent disponible (nœud Jenkins). |
| `agent { docker { image 'python:3.11' } }` | Lance un conteneur Docker éphémère avec l'image spécifiée. Utile pour isoler les dépendances. |

### Fail Fast

Principe CI/CD qui consiste à **détecter les erreurs le plus tôt possible** dans le pipeline pour éviter de gaspiller du temps de build. Le stage Lint s'exécute avant le Build pour échouer immédiatement si le code contient des erreurs de syntaxe.

### Quality Gate

Seuil de qualité minimal qu'un build doit franchir pour passer. Dans ce TP, le seuil est une **couverture de tests ≥ 70 %** (`--cov-fail-under=70`). Ce seuil sera renforcé à 80 % avec SonarQube au TP3.

### Image Tag & Traçabilité

Chaque image Docker est taguée avec le **SHA court du commit Git** (`git rev-parse --short HEAD`), par exemple `sentiment-ai:a3f8c12`. Cela permet de retrouver exactement quel commit correspond à quelle image déployée en production.

### Poll SCM vs Webhook

| Méthode | Principe | Délai | Charge serveur |
|---|---|---|---|
| **Poll SCM** | Jenkins interroge GitHub à intervalle fixe | Jusqu'à 5 min | Requêtes périodiques même sans changement |
| **Webhook** | GitHub notifie Jenkins à chaque `git push` | Quelques secondes | Requête uniquement en cas de push |

### withCredentials

Bloc Groovy qui injecte des secrets (tokens, mots de passe) comme variables d'environnement **au moment de l'exécution**. Les valeurs ne sont jamais visibles dans les logs Jenkins (remplacées par `****`). Le Jenkinsfile peut être commité dans Git sans exposer les secrets.

---

## Architecture du pipeline

```
git push
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│                     Jenkins Pipeline                    │
│                                                         │
│  ┌──────────┐  ┌──────┐  ┌────────────┐  ┌──────────┐  │
│  │ Checkout │→ │ Lint │→ │Build &Test │→ │  Push    │  │
│  │          │  │flake8│  │Docker+pytest│  │(main only)│ │
│  └──────────┘  └──────┘  └────────────┘  └──────────┘  │
│                                                         │
│  post { always { docker compose down -v } }             │
└─────────────────────────────────────────────────────────┘
                                                │
                                                ▼
                                       ghcr.io/PSEUDO/sentiment-ai
                                       :a3f8c12  (SHA tag)
                                       :latest
```

---

## Prérequis

- Docker installé et en cours d'exécution sur votre machine
- Un compte GitHub avec un dépôt `sentiment-ai` (TP1 complété)
- Git configuré localement (`git config --global user.name/email`)
- Accès à `localhost:8080` (aucun autre service ne doit utiliser ce port)

---

## Partie 1 - Installer Jenkins via Docker

### Concepts abordés

- **Volume Docker** : espace de stockage persistant indépendant du cycle de vie du conteneur.
- **Docker socket** : interface Unix permettant à un processus de communiquer avec le daemon Docker.

### 1.1 Lancer Jenkins

Jenkins sera installé dans un conteneur Docker qui a accès au daemon Docker de l'hôte via la technique **DooD**.

```bash
# Créer un volume pour persister les données Jenkins entre les redémarrages
docker volume create jenkins-data
```
![image](https://hackmd.io/_uploads/ByPBhVbMMl.png)

```bash
# Lancer Jenkins avec accès au Docker de l'hôte
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins-data:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```
![image](https://hackmd.io/_uploads/ByV_3E-Mzx.png)


```bash
# Vérifier que Jenkins démarre correctement
docker logs -f jenkins
# Attendez la ligne : Jenkins is fully up and running
```
![image](https://hackmd.io/_uploads/HyyLaEbffl.png)

```bash
# Vérifier que Jenkins peut accéder à Docker
docker exec -u jenkins jenkins docker ps
```

**Si erreur `executable file not found`** → installer Docker dans le conteneur :

```bash
docker exec -u root jenkins bash -c "
  apt-get update -q &&
  apt-get install -y docker.io
"
```
![image](https://hackmd.io/_uploads/Sk6g0Ebfzl.png)


**Si erreur `permission denied`** → corriger les permissions du socket :

![image](https://hackmd.io/_uploads/B1tfA4ZGMx.png)

```bash
docker exec -u root jenkins chmod 666 /var/run/docker.sock

# Retester
docker exec -u jenkins jenkins docker ps
```
![image](https://hackmd.io/_uploads/H1fE04-ffe.png)
![image](https://hackmd.io/_uploads/ryb8R4-MGl.png)


> **Rôle des montages :**
> - `/var/run/docker.sock` : expose le socket Unix du daemon Docker de l'hôte à l'intérieur du conteneur Jenkins. Sans cela, Jenkins ne pourrait pas exécuter `docker build` et `docker push` depuis le pipeline.
> - `jenkins-data` : persiste les jobs, builds, plugins et credentials même si le conteneur est recréé ou mis à jour.

### 1.2 Première configuration Jenkins

1. Ouvrez `http://localhost:8080` dans votre navigateur.
2. Récupérez le mot de passe initial :
   ```bash
   docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
   ```
3. Collez ce mot de passe dans l'interface Jenkins.
4. Choisissez **« Install suggested plugins »** et attendez la fin de l'installation.
5. Créez votre compte administrateur (notez login / mot de passe).
6. Cliquez **« Save and Finish »** → **« Start using Jenkins »**.

![image](https://hackmd.io/_uploads/ryrnAVZMGl.png)
![image](https://hackmd.io/_uploads/SkuR0VZMMl.png)
![image](https://hackmd.io/_uploads/B1Q11BWzMe.png)
![image](https://hackmd.io/_uploads/HJo_1HZMzx.png)
![image](https://hackmd.io/_uploads/By5pkB-Mze.png)
![image](https://hackmd.io/_uploads/ryOCkBZGzx.png)
![image](https://hackmd.io/_uploads/rJv1lr-fMx.png)


### 1.3 Installer les plugins nécessaires

Jenkins a besoin du plugin **Docker Pipeline** pour exécuter des commandes Docker dans un Jenkinsfile.

1. Jenkins → Tableau de bord → **Administrer Jenkins** → **Plugins**
2. Onglet **« Available plugins »** → cherchez et installez :
   - `Docker Pipeline`
   - `Git`
   - `Pipeline`
   - `Blue Ocean`
3. Redémarrez Jenkins après l'installation si demandé.

![image](https://hackmd.io/_uploads/B17IgH-fGg.png)

- Vérification
```bash
docker exec jenkins sh -c '
echo "Docker Pipeline:"; ls /var/jenkins_home/plugins/docker-workflow.jpi 2>/dev/null && echo OK;
echo "Git:"; ls /var/jenkins_home/plugins/git.jpi 2>/dev/null && echo OK;
echo "Pipeline:"; ls /var/jenkins_home/plugins/workflow-aggregator.jpi 2>/dev/null && echo OK;
echo "Blue Ocean:"; ls /var/jenkins_home/plugins/blueocean.jpi 2>/dev/null && echo OK;
'
```

![image](https://hackmd.io/_uploads/Sy2NGHbMMe.png)

### 1.4 Configurer les credentials GitHub

Jenkins a besoin d'un token GitHub pour cloner le repo et pousser les images vers `ghcr.io`. **Le token ne doit jamais être écrit en clair dans le Jenkinsfile.**

**Créer un token sur GitHub :**

1. GitHub → Settings → Developer settings → Personal access tokens → **Tokens (classic)**
2. Generate new token → cochez : `repo`, `read:packages`, `write:packages`
3. Copiez le token généré *(affiché une seule fois)*.
![image](https://hackmd.io/_uploads/H1PQoHbGzx.png)

```bash
# Vérifier que GitHub CLI (gh) est installé et afficher sa version
gh --version

# Se connecter à GitHub depuis le terminal (authentification du compte)
gh auth login

# Mettre à jour le token GitHub existant avec les permissions nécessaires :
# - repo : accès aux dépôts GitHub
# - read:packages : lecture des packages GitHub
# - write:packages : écriture/publication des packages GitHub
gh auth refresh -h github.com -s repo,read:packages,write:packages

# Vérifier l'état de la connexion GitHub et les permissions du token
gh auth status

# Afficher le token GitHub actuellement utilisé
gh auth token
```
![image](https://hackmd.io/_uploads/SkjdmS-fMg.png)
![image](https://hackmd.io/_uploads/H1Sg4rbMMe.png)
![image](https://hackmd.io/_uploads/HJkm4BbGMg.png)

**Enregistrer le token dans Jenkins :**

1. Jenkins → Administrer Jenkins → Credentials → System → **Global credentials**
2. Add Credentials → Kind : **Username with password**
3. Username : votre pseudo GitHub | Password : le token | ID : `github-token`
4. Cliquez **Create**.
![image](https://hackmd.io/_uploads/HyYc4BWGMe.png)
![image](https://hackmd.io/_uploads/SktjEHWfGe.png)
![image](https://hackmd.io/_uploads/HJmM3SWGzg.png)
![image](https://hackmd.io/_uploads/B1MXhHbfzx.png)



---

### ❓ Questions - Partie 1

**Question 1.1**
Faites un screenshot de la page d'accueil Jenkins (Dashboard) avec votre compte connecté. Quel est le rôle du volume `jenkins-data` monté sur `/var/jenkins_home` ?
![image](https://hackmd.io/_uploads/HkNeUrbfGl.png)

Le volume `jenkins-data` monté sur `/var/jenkins_home` joue le rôle de **stockage persistant** pour Jenkins.

Par défaut, toutes les données Jenkins (jobs, historique des builds, plugins installés, credentials, configuration) sont stockées dans `/var/jenkins_home` à l'intérieur du conteneur. Or, un conteneur Docker est **éphémère** : si on le supprime, recrée ou met à jour, tout son contenu interne est perdu.

En montant le volume `jenkins-data` sur ce chemin, on découple les données du conteneur :

- Si on fait `docker rm jenkins` puis `docker run ... jenkins/jenkins:lts`, Jenkins retrouve exactement l'état où on l'avait laissé — jobs configurés, builds passés, plugins, credentials inclus.
- On peut mettre à jour l'image Jenkins (passage de `lts` à une version plus récente) sans perdre aucune configuration.
- On peut sauvegarder les données Jenkins simplement en sauvegardant le volume.

En résumé : **le conteneur est jetable, les données ne le sont pas**.

**Question 1.2**
Expliquez en deux phrases pourquoi on monte `/var/run/docker.sock` dans le conteneur Jenkins. Quel risque de sécurité cela représente-t-il ? Comment le limiterait-on en production ?

`/var/run/docker.sock` est le socket Unix par lequel on communique avec le daemon Docker de l'hôte. En le montant dans le conteneur Jenkins, on lui permet d'envoyer des commandes Docker (docker build, docker push, docker run) directement au daemon de la machine hôte, sans avoir à faire tourner un second daemon Docker à l'intérieur du conteneur.

---

## Partie 2 - Écrire le Jenkinsfile

### 2.1 Structure de base

Créez le fichier `Jenkinsfile` à la racine de votre projet `sentiment-ai` :

```bash
cat > Jenkinsfile <<'EOF'
pipeline {
  agent any

  environment {
    IMAGE_NAME = 'sentiment-ai'
    REGISTRY   = 'ghcr.io/VOTRE_PSEUDO'

    IMAGE_TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
  }

  stages {
    // Les 4 stages seront ajoutés ici
  }

  post {
    always {
      sh 'docker compose down -v 2>/dev/null || true'
    }
    success {
      echo "Pipeline réussi ! Image : ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
    }
    failure {
      echo 'Pipeline échoué. Consultez les logs ci-dessus.'
    }
  }
}
EOF
```
![image](https://hackmd.io/_uploads/ByABUH-zfg.png)

- Modifier le pseudo
![image](https://hackmd.io/_uploads/HyqRIrWGMx.png)
![image](https://hackmd.io/_uploads/HyzlDS-zfg.png)


**Comprendre `environment { }` et `IMAGE_TAG`**
`IMAGE_TAG` utilise `sh(...)` pour exécuter une commande shell et capturer sa sortie. `git rev-parse --short HEAD` retourne les 7 premiers caractères du SHA du dernier commit.
Résultat : chaque build produit une image taguée de façon unique, par exemple `sentiment-ai:a3f8c12`. On peut toujours retrouver exactement quel commit correspond à quelle image déployée en production.

### 2.2 Stage 1 - Checkout

Le stage **Checkout** demande à Jenkins de cloner le code source depuis le repository configuré dans le job. `checkout scm` est une abréviation Groovy qui utilise automatiquement la configuration SCM du job Jenkins.

```groovy
stage('Checkout') {
  steps {
    checkout scm
    echo "Branche : ${env.BRANCH_NAME}"
    echo "Commit  : ${env.GIT_COMMIT}"
    sh 'git log --oneline -5'
  }
}
```

### 2.3 Stage 2 - Lint

Le **Lint** analyse le code Python avec `flake8` avant même de construire l'image Docker. C'est le principe **Fail Fast** : détecter les erreurs de syntaxe le plus tôt possible. `flake8` est lancé dans un conteneur Docker éphémère — aucune dépendance sur l'agent Jenkins.

```groovy
stage('Lint') {
  steps {
    sh '''
      docker run --rm \
        --volumes-from jenkins \
        -w $WORKSPACE \
        python:3.12-slim \
        sh -c "pip install flake8 -q && flake8 src/ --max-line-length=100"
    '''
  }
}
```
![image](https://hackmd.io/_uploads/BJuAvSZMzg.png)


> **Erreurs flake8 courantes et corrections**
>
> | Code | Signification | Correction |
> |---|---|---|
> | `W292` | Pas de saut de ligne en fin de fichier | Ajouter une ligne vide à la fin |
> | `W291` | Espaces en fin de ligne | Supprimer les espaces trailing |
> | `E302` | 2 lignes vides attendues | Ajouter des lignes vides entre fonctions |
> | `E501` | Ligne trop longue (> 100 caractères) | Couper la ligne |
> | `F401` | Import non utilisé | Supprimer l'import |

Pour corriger automatiquement :

```bash
pip install autopep8
autopep8 --in-place --aggressive src/main.py src/model.py src/schemas.py
flake8 src/ --max-line-length=100
git add src/
git commit -m "fix: correct flake8 style errors"
git push origin main
```

### 2.4 Stage 3 - Build & Test

Ce stage construit l'image Docker avec le SHA Git comme tag, puis lance `pytest` à l'intérieur de cette image. **Tester dans l'image buildée garantit que les tests s'exécutent dans le même environnement que la production.**

```groovy
stage('Build & Test') {
  steps {
    sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
    sh """
      docker run --rm \
        ${IMAGE_NAME}:${IMAGE_TAG} \
        pytest tests/ -v \
          --cov=src \
          --cov-report=xml:coverage.xml \
          --cov-report=term-missing \
          --cov-fail-under=70
    """
  }
  post {
    failure {
      echo 'Tests échoués ou coverage insuffisant (< 70%)'
    }
  }
}
```

**Comprendre `--cov-fail-under=70`**
Si la couverture de code par les tests est inférieure à 70 %, `pytest` retourne un code d'erreur non nul. Jenkins interprète ce code d'erreur : le stage échoue et le pipeline s'arrête immédiatement. C'est un **Quality Gate** minimal intégré directement dans le pipeline. Ce seuil sera renforcé avec SonarQube au TP3 (80 % minimum).

### 2.5 Stage 4 - Push (conditionnel)

Le **Push** ne s'exécute que sur la branche `main`. Les branches de feature sont buildées et testées mais leurs images ne sont pas poussées vers le registry — cela évite de polluer le registry avec des images non validées.

```groovy
stage('Push') {
  when { branch 'main' }
  steps {
    withCredentials([usernamePassword(
      credentialsId: 'github-token',
      usernameVariable: 'REGISTRY_USER',
      passwordVariable: 'REGISTRY_PASS'
    )]) {
      sh """
        echo \$REGISTRY_PASS | docker login ghcr.io \
          -u \$REGISTRY_USER --password-stdin
        docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:latest
        docker push ${REGISTRY}/${IMAGE_NAME}:latest
      """
    }
  }
}
```
![image](https://hackmd.io/_uploads/HJAUOrWzMg.png)


**Sécurité - `withCredentials`**
Ne jamais écrire un token ou mot de passe directement dans le Jenkinsfile : il serait stocké dans Git et visible par tous. `withCredentials` injecte les secrets comme variables d'environnement au moment de l'exécution. Les valeurs ne sont jamais visibles dans les logs Jenkins (remplacées par `****`). Le Jenkinsfile peut être commité dans Git sans risque.

#### Fichier complet Jenkinsfile

```bash
pipeline {
  agent any

  environment {
    IMAGE_NAME = 'sentiment-ai'
    REGISTRY   = 'ghcr.io/dspitech'
    REGISTRY_IMAGE = "${REGISTRY}/${IMAGE_NAME}"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        script {
          env.IMAGE_TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        }
      }
    }

    stage('Info') {
      steps {
        sh 'git log --oneline -3'
        sh 'echo "Workspace OK"'
      }
    }

    stage('Lint') {
      steps {
        sh '''
          docker run --rm \
            -v $WORKSPACE:/app \
            -w /app \
            python:3.12-slim \
            sh -c "pip install flake8 -q && flake8 ."
        '''
      }
    }

    stage('Build Docker') {
      steps {
        sh '''
          docker build -t sentiment-ai:${IMAGE_TAG} .
        '''
      }
    }

    stage('Test') {
      steps {
        sh '''
          docker run --rm \
            sentiment-ai:${IMAGE_TAG} \
            pytest tests -v
        '''
      }
    }

    stage('Push to GHCR') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'github-token',
          usernameVariable: 'GITHUB_USER',
          passwordVariable: 'GITHUB_TOKEN'
        )]) {
          sh '''
            echo $GITHUB_TOKEN | docker login ghcr.io -u $GITHUB_USER --password-stdin

            docker tag sentiment-ai:${IMAGE_TAG} ${REGISTRY_IMAGE}:${IMAGE_TAG}
            docker tag sentiment-ai:${IMAGE_TAG} ${REGISTRY_IMAGE}:latest

            docker push ${REGISTRY_IMAGE}:${IMAGE_TAG}
            docker push ${REGISTRY_IMAGE}:latest
          '''
        }
      }
    }

  }

  post {
    success {
      echo "Pipeline OK - Image pushed: ${REGISTRY_IMAGE}:${IMAGE_TAG}"
    }
    failure {
      echo "Pipeline FAILED"
    }
  }
}
```
### 2.6 Commiter le Jenkinsfile

```bash
git add Jenkinsfile
git commit -m "ci: add Jenkinsfile with 4 stages"
git push origin main
```

---

### ❓ Questions - Partie 2

**Question 2.1**
À quoi sert le bloc `post { always { } }` dans le pipeline ? Pourquoi ajoute-t-on `|| true` à la commande `docker compose down` ?

Le bloc `post` définit des actions à exécuter **après la fin des stages**, quel que soit le résultat du pipeline. Il existe plusieurs conditions :

| Condition | Déclenchement |
|---|---|
| `always` | Toujours, succès ou échec |
| `success` | Uniquement si tous les stages ont réussi |
| `failure` | Uniquement si au moins un stage a échoué |
| `unstable` | Si le build est instable (tests avec warnings) |

Dans notre pipeline, `post { always { } }` contient :

```groovy
sh 'docker compose down -v 2>/dev/null || true'
```

Son rôle est de **nettoyer les conteneurs et volumes de test** après chaque build, peu importe ce qui s'est passé. Sans ce bloc, si un stage échoue en plein milieu d'un `docker compose up`, les conteneurs resteraient actifs et consommeraient des ressources inutilement - et les builds suivants pourraient entrer en conflit avec ces conteneurs orphelins.


`|| true` est un opérateur shell qui dit : **"si la commande de gauche échoue, exécute `true` à la place"**. `true` retourne toujours le code de sortie `0` (succès).

Sans `|| true`, deux scénarios posent problème :

**Scénario 1 - Le stage a échoué avant même le `docker compose up`**
Aucun conteneur n'a été lancé. `docker compose down` ne trouve rien à arrêter et retourne une erreur. Jenkins interprète cette erreur et **marque le build comme échoué** à cause du nettoyage, masquant la vraie cause de l'échec.

**Scénario 2 - `docker compose` n'est pas installé sur l'agent**
Même problème : la commande échoue, le pipeline plante sur le nettoyage.

Avec `|| true`, si `docker compose down` échoue pour n'importe quelle raison, le shell retourne `0` et Jenkins continue sans marquer le build comme échoué à cause du nettoyage.


En résumé : `|| true` rend le nettoyage **non bloquant** — il est tenté dans tous les cas, mais son échec éventuel n'impacte pas le résultat final du build.

**Question 2.2**
Expliquez la différence entre `agent any` et `agent { docker { image 'python:3.11' } }`. Dans quel cas utiliseriez-vous le second ?

**Question 2.3**
Pourquoi le stage Push utilise-t-il `when { branch 'main' }` ? Que se passerait-il si on poussait une image pour chaque branche feature ?

---

## Partie 3 - Créer et exécuter le job Jenkins

### 3.1 Créer un job Pipeline dans Jenkins

1. Jenkins → **Nouveau Item** → Nom : `sentiment-ai-pipeline` → Type : **Pipeline** → OK
2. Section **General** : cochez **« GitHub project »** → URL de votre repo GitHub
3. Section **Build Triggers** : cochez **« Poll SCM »** → Schedule : `H/5 * * * *`
   *(Jenkins vérifiera les nouveaux commits toutes les 5 minutes)*
4. Section **Pipeline** :
   - Definition : `Pipeline script from SCM`
   - SCM : `Git`
   - Repository URL : URL HTTPS de votre repo
   - Credentials : `github-token`
   - Branch Specifier : `*/main`
   - Script Path : `Jenkinsfile`
5. Cliquez **Save**.

![image](https://hackmd.io/_uploads/r1mwKBZGMe.png)
![image](https://hackmd.io/_uploads/Sk_TnSZMGe.png)
![image](https://hackmd.io/_uploads/B1wAnB-zGl.png)
![image](https://hackmd.io/_uploads/HyVJ6S-Mfl.png)
![image](https://hackmd.io/_uploads/H1tbTHbGGx.png)
![image](https://hackmd.io/_uploads/rJCf6B-fzg.png)

### 3.2 Lancer le premier build

```bash
# Vérifier que le Jenkinsfile est bien poussé sur main
git log --oneline origin/main | head -3
```
![image](https://hackmd.io/_uploads/rycFpH-GGl.png)

Dans Jenkins, cliquez sur le job `sentiment-ai-pipeline` puis **Build Now**. Surveillez le build en temps réel : cliquez sur le numéro du build → **Console Output**.
![image](https://hackmd.io/_uploads/ry-1WLbffg.png)

![image](https://hackmd.io/_uploads/H1T_mIbzGx.png)


| Indicateur | Signification | Action si problème |
|---|---|---|
| Build stable | Tous les stages ont réussi | Parfait ! |
| Build échoué | Au moins un stage a échoué | Consulter Console Output |
| Build instable | Tests avec warnings | Lire les warnings pytest |

### 3.3 Déboguer un stage en échec

```bash
# Erreur 1 : Docker non accessible depuis Jenkins
docker exec jenkins docker ps
# Si erreur : vérifier que /var/run/docker.sock est bien monté

# Erreur 2 : Permission denied sur docker.sock
docker exec -u root jenkins chmod 666 /var/run/docker.sock

# Erreur 3 : Module Python non trouvé dans les tests
# → Vérifier que "COPY src/" est bien dans le Dockerfile

# Erreur 4 : Credentials incorrects pour le push
# → Revérifier l'ID dans withCredentials(credentialsId: 'github-token')
```

---

### ❓ Questions - Partie 3

**Question 3.1**
Faites un screenshot du pipeline après le premier build réussi (vue stages ou Console Output). Quel tag a été attribué à l'image Docker construite ? Retrouvez cette valeur dans les logs Jenkins.
![image](https://hackmd.io/_uploads/B1-xZ8bGfx.png)

```Bash
git rev-parse --short HEAD
```
![image](https://hackmd.io/_uploads/Hk7rbLbGGe.png)

**Tag = 997da42**

**Question 3.2**
Faites un second build en modifiant un fichier source (par exemple, ajouter un commentaire dans `src/main.py`). Le pipeline se relance-t-il automatiquement au bout de 5 minutes ? Vérifiez sur GitHub : l'image apparaît-elle dans les **Packages / Registry** ?

- On ajoute un commentaire dan sle fichier 

```bash
echo "# test pipeline second build" >> src/main.py
```
![image](https://hackmd.io/_uploads/S1YAWIZzfx.png)

- Commit + push

```
git add src/main.py
git commit -m "test: trigger second Jenkins build"
git push origin main
```
![image](https://hackmd.io/_uploads/HkWWm8ZzGx.png)

Le pipeline se relance automatiquement après un push GitHub grâce au mécanisme de polling/trigger SCM Jenkins. Cependant, l’image Docker n’est pas encore visible dans GitHub Packages, car la phase de push vers le registry n’est pas exécutée dans le pipeline actuel. 
![image](https://hackmd.io/_uploads/r1tFHL-MMx.png)

Après avoir apporter des modifications au niveau du fichier **Jenkinsfile**,  l'image est visible dans Github Packages et connecter avec le repo.
![image](https://hackmd.io/_uploads/H1HZuL-zfe.png)

---

## Partie 4 - Webhook · Déclenchement automatique

Actuellement Jenkins vérifie les nouveaux commits toutes les 5 minutes (Poll SCM). Avec un webhook, GitHub notifie Jenkins **instantanément** à chaque `git push`.

### 4.1 Exposer Jenkins avec ngrok

Jenkins tourne en local sur votre machine. GitHub ne peut pas y accéder directement. **ngrok** crée un tunnel public temporaire qui rend Jenkins accessible depuis l'extérieur.

```bash
# Ajouter la clé GPG
curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null

# Ajouter le dépôt
echo "deb https://ngrok-agent.s3.amazonaws.com bookworm main" | sudo tee /etc/apt/sources.list.d/ngrok.list

# Mettre à jour et installer
sudo apt update
sudo apt install ngrok

# Vérifier la version
ngrok --version
# Vérifier l'installation
ngrok --version

# Configurer votre token (syntaxe v3)
# Pour avoir le token aller  sur le site officiel et créer un compte; à la fin vous aurez le token. Site : https://dashboard.ngrok.com/
ngrok config add-authtoken + Mettre le token

# Démarrer le tunnel
nohup ngrok http 8080 --log=stdout --log-level=info > ngrok.log 2>&1 &

# Attendre le démarrage
sleep 3

# Récupérer l'URL
curl -s http://localhost:4040/api/tunnels | python3 -c "import sys, json; data=json.load(sys.stdin); print('🌐 URL publique :', data['tunnels'][0]['public_url'])"

```

 **ngrok non disponible ?**
Si ngrok n'est pas disponible, continuez avec le Poll SCM configuré à `H/5 * * * *`. Le pipeline se déclenchera automatiquement toutes les 5 minutes - suffisant pour ce TP. En entreprise, Jenkins est généralement hébergé sur un serveur avec une IP publique fixe.

### 4.2 Configurer le webhook sur GitHub

1. Repository GitHub → Settings → **Webhooks** → **Add webhook**
2. Payload URL : `https://VOTRE_URL_NGROK/github-webhook/`
3. Content type : `application/json`
4. Which events : **Just the push event**
5. Active : coché → **Add webhook**

![image](https://hackmd.io/_uploads/HkDSQv-fMx.png)




### 4.3 Activer les triggers GitHub dans Jenkins

1. Jenkins → votre job → **Configurer**
2. Section **Build Triggers** → cochez : `GitHub hook trigger for GITScm polling`
3. Sauvegardez.
![image](https://hackmd.io/_uploads/H1SwpLbMMx.png)


### Tester ngrok
![image](https://hackmd.io/_uploads/HyDaWv-zfx.png)
![image](https://hackmd.io/_uploads/rJ_xzDWffx.png)


### 4.4 Tester le déclenchement automatique

```bash
# Créer une branche feature et faire une modification
git checkout -b feat/test-webhook
echo '# test webhook' >> README.md
git add README.md
git commit -m "test: verify webhook triggers Jenkins pipeline"
git push origin feat/test-webhook

# Observer dans Jenkins : un nouveau build démarre automatiquement
# (dans les secondes qui suivent le push si ngrok est actif)

# Après vérification, nettoyer la branche
git checkout main
git branch -d feat/test-webhook
git push origin --delete feat/test-webhook
```
![image](https://hackmd.io/_uploads/rJwdMwZMzl.png)

---

### ❓ Questions - Partie 4

**Question 4.1**
Le pipeline s'est-il déclenché automatiquement après le push ? Faites un screenshot du build automatique. Quelle est la différence entre Poll SCM et un webhook en termes de délai et de charge serveur ?

![image](https://hackmd.io/_uploads/SJF2XD-zze.png)
Oui le pipeline s'est déclenché automatiquement.
![image](https://hackmd.io/_uploads/By8gEvZGze.png)
![image](https://hackmd.io/_uploads/Bks-NvZMfg.png)

**Poll SCM** : 
Jenkins interroge GitHub à intervalle fixe (ici H/5 * * * *, soit toutes les 5 minutes) pour vérifier si de nouveaux commits sont apparus. C'est Jenkins qui prend l'initiative de demander.

**Webhook** :
GitHub notifie Jenkins instantanément à chaque git push. C'est GitHub qui prend l'initiative d'envoyer une requête HTTP vers Jenkins.

#### Comparaison directe

| Critère | Poll SCM | Webhook |
|----------|----------|----------|
| **Délai de déclenchement** | Jusqu'à 5 min (selon l'intervalle configuré) | Quelques secondes |
| **Qui initie la communication** | Jenkins interroge GitHub | GitHub notifie Jenkins |
| **Charge réseau** | Requêtes constantes même sans commit | Requête uniquement en cas de push |
| **Charge sur Jenkins** | Réveil périodique inutile s'il n'y a pas de commit | Sollicité uniquement lorsque nécessaire |
| **Configuration requise** | Aucune — Jenkins accessible uniquement en local | Jenkins doit être accessible depuis Internet (ngrok, IP publique) |
| **Fiabilité** | Fonctionne même derrière un pare-feu | Dépend de la disponibilité du tunnel ou de l'IP publique |

Console Output :

```
Started by GitHub push by dspitech
Obtained Jenkinsfile from git https://github.com/dspitech/sentiment-ai.git
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins
 
[Pipeline] End of Pipeline
Finished: SUCCESS

```


---

## Partie 5 - Questions de synthèse

### A - Architecture Jenkins

**A1 - Rôle des 4 stages**

| Stage | Rôle |
|---|---|
| **Checkout** | Clone le code source depuis GitHub vers l'agent Jenkins pour que les stages suivants puissent travailler dessus. |
| **Lint** | Analyse statique du code Python avec `flake8` pour détecter les erreurs de syntaxe et de style avant de builder. |
| **Build & Test** | Construit l'image Docker taguée avec le SHA Git et exécute les tests pytest avec vérification de la couverture de code. |
| **Push** | Pousse l'image validée vers le registry `ghcr.io` uniquement si on est sur la branche `main`. |


**A2 - Qu'est-ce qu'un agent Jenkins ?**

Un **agent** est le nœud (machine ou environnement) sur lequel Jenkins exécute les étapes du pipeline.

| Déclaration | Comportement | Cas d'usage |
|---|---|---|
| `agent any` | S'exécute sur n'importe quel nœud Jenkins disponible, dans l'environnement tel quel de la machine | Pipeline simple, outils déjà installés sur l'agent |
| `agent { docker { image 'python:3.11' } }` | Lance un conteneur Docker éphémère avec l'image spécifiée, le pipeline s'exécute dedans, le conteneur est détruit après | Isolation totale des dépendances, reproductibilité garantie |

Le second est préférable quand on veut s'assurer que le pipeline s'exécute dans un environnement **identique à chaque build**, indépendamment de ce qui est installé sur l'agent Jenkins.



**A3 - Pourquoi `withCredentials` plutôt qu'écrire le token en clair ?**

Un token écrit directement dans le Jenkinsfile est **stocké dans Git** et devient visible par toute personne ayant accès au repo - y compris dans l'historique des commits, même si on le supprime plus tard.

`withCredentials` résout ce problème de trois façons :

- Les secrets sont stockés dans le **coffre-fort chiffré de Jenkins**, pas dans le code
- Ils sont injectés comme variables d'environnement **uniquement au moment de l'exécution**
- Ils sont **masqués dans les logs** Jenkins (remplacés par `****`)

Le Jenkinsfile peut donc être commité publiquement sans aucun risque.

---

### B - CI/CD et Qualité

**B1 - Le concept Fail Fast**

Le **Fail Fast** consiste à placer les vérifications les moins coûteuses en premier dans le pipeline, pour échouer le plus tôt possible si le code est invalide - sans perdre de temps à builder ou tester du code mal écrit.

Si le code est mal écrit, c'est le **stage Lint** qui doit échouer en premier, car il s'exécute sur le code brut sans même construire l'image Docker.

---

**B2 - Pourquoi ne pas pousser une image pour chaque branche feature ?**

Plusieurs raisons :

- **Pollution du registry** : des dizaines de branches feature génèrent des dizaines d'images non validées qui s'accumulent dans `ghcr.io`, rendant le registry ingérable.
- **Fausse impression de stabilité** : une image taguée dans le registry peut laisser croire qu'elle est prête à être déployée, alors qu'elle vient d'une branche non relue et non mergée.
- **Coût de stockage** : chaque image pèse plusieurs centaines de Mo. Pousser pour chaque branche multiplie inutilement les coûts.

Seul `main` représente le code **validé, reviewé et mergé** - c'est le seul état qui mérite d'être publié dans le registry.

---

**B3 - Workflow complet en 5 étapes**

```
1. git push origin main
        │
        ▼
2. GitHub notifie Jenkins via webhook
        │
        ▼
3. Jenkins exécute le pipeline : Lint → Build → Tests
        │
        ▼
4. Si tous les stages sont verts, docker push vers ghcr.io
        │
        ▼
5. Image disponible dans le registry : ghcr.io/pseudo/sentiment-ai:a3f8c12
```


### C - Traçabilité et Versionnement

**C1 - Retrouver le code source d'une image déployée**

`a3f8c12` est le SHA court du commit Git qui a produit cette image. Pour retrouver le code exact :

```bash
# Dans le repo Git local
git checkout a3f8c12
# Ou consulter directement sur GitHub
```

C'est précisément l'intérêt de taguer avec le SHA plutôt qu'un numéro de version arbitraire : le tag **est** un pointeur direct vers le commit, sans ambiguïté possible.

---

**C2 - Pourquoi deux tags `:SHA` et `:latest` ?**

Les deux tags servent des usages différents et complémentaires :

| Tag | Rôle | Utilisé par |
|---|---|---|
| `:a3f8c12` (SHA) | Tag **immuable** - pointe toujours vers exactement ce build, ne changera jamais | Production, rollback, audit, debugging |
| `:latest` | Tag **mobile** - pointe toujours vers le dernier build de `main` | Développement local, CI des projets dépendants |

En pratique :
- On **déploie en production** avec le tag SHA - on sait exactement ce qu'on déploie et on peut rollback vers un SHA précis.
- On utilise `:latest` pour récupérer rapidement la dernière version stable sans connaître le SHA, mais on ne s'en sert jamais pour un déploiement en production car sa cible change à chaque build.

---

## Récapitulatif - À rendre

- `Jenkinsfile` commité à la racine du repo `sentiment-ai`
-  Screenshot du pipeline Jenkins avec les **4 stages tous verts**
-  Screenshot de la **Console Output** du premier build réussi
-  Image Docker visible dans **GitHub Packages** avec le tag SHA Git
-  Screenshot du déclenchement automatique via webhook ou Poll SCM
-  Réponses aux questions **1.1, 1.2, 2.1, 2.2, 2.3, 3.1, 3.2, 4.1** et synthèse

---

## Prérequis pour le TP3

Le TP3 ajoutera **SonarQube** et **Trivy** au pipeline Jenkins pour l'analyse de qualité du code et le scan de sécurité des images Docker. Avant de passer au TP3, vérifiez que :

- Votre `Jenkinsfile` est commité sur `main` et le pipeline passe en **vert**
-  L'image Docker est visible dans GitHub Packages (`ghcr.io`)
-  Le fichier `coverage.xml` est bien généré dans le workspace Jenkins

---

