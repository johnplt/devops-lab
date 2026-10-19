# Déploiement d'Apache Airflow 3.3.2 en environnement conteneurisé (Architecture mutualisée)

Ce document détaille la mise en place d'Apache Airflow 3.3.2 sur notre instance Hetzner Cloud (CX23 / Ubuntu 24.04), en l'intégrant au sein d'une infrastructure Docker existante (réseau interne et base de données PostgreSQL centralisée).

## Architecture et Choix Techniques

- Moteur d'exécution : Docker Engine & Docker Compose.

- Réseau : Utilisation d'un réseau Docker externe dédié (app-network) pour permettre la communication inter-conteneurs de manière isolée.

- Base de données : Mutualisation sur un conteneur PostgreSQL centralisé (<POSTGRES_CONTENEUR_NAME>) hébergeant une base dédiée (<AIRFLOW_POSTGRES_DATABASE>), optimisant ainsi l'empreinte mémoire sur le serveur de lab.

## Création de la base de données et l'utilisateur pour Airflow
Cette commande va créer l'utilisateur <AIRFLOW_POSTGRES_USER> et la base <AIRFLOW_POSTGRES_DATABASE>  dans le PostgreSQL existant :
```bash
docker exec -it <POSTGRES_CONTENEUR_NAME> psql -U <POSTGRES_USER> -c "CREATE USER <AIRFLOW_POSTGRES_USER> WITH PASSWORD '<AIRFLOWDB_PASSWORD>';" && \
docker exec -it <POSTGRES_CONTENEUR_NAME> psql -U <POSTGRES_USER> -c "CREATE DATABASE <AIRFLOW_POSTGRES_DATABASE>  OWNER <AIRFLOW_POSTGRES_USER>;"
```

## Créer le dossier pour Airflow et ses sous-dossiers
Airflow a besoin de dossiers spécifiques pour ranger ses DAGs (les workflows Python), ses logs et ses plugins.

```bash
# 1. Créer le dossier principal Airflow et ses sous-dossiers
mkdir -p ~/devops-lab/airflow/dags ~/devops-lab/airflow/logs ~/devops-lab/airflow/plugins

# 2. Se placer dans le dossier airflow
cd ~/devops-lab/airflow
```

## Récupérer le fichier docker-compose.yaml officiel d'Airflow 3.3.2
Plutôt que d'écrire un fichier à l'aveugle, l'équipe d'Airflow fournit un fichier de référence pour la version 3. Sur la VM, dans le dossier ~/devops-lab/airflow, on télécharge le fichier officiel :
```bash
cd ~/devops-lab/airflow
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/3.3.2/docker-compose.yaml'
```

## Adapter le fichier pour utiliser le PostgreSQL partagé
Pour faire fonctionner Airflow 3.3.2 avec notre base de données et notre réseau partagés, les modifications suivantes doivent être apportées au template d'origine.

Ouvrir le fichier téléchargé :
```bash
vi docker-compose.yaml
```


Modifier / ajuster les points clés suivants dans le fichier :

1. L'image : Vérifier qu'elle pointe bien sur la version 3 (ex: apache/airflow:3.3.2).

2. Mise à jour de la chaîne de connexion (SQL_ALCHEMY_CONN) : Pointage direct vers le conteneur externe. Remplacer l'hôte par le nom du conteneur partagé et pointer vers la base dédiée <AIRFLOW_POSTGRES_DATABASE>  :
    
```yaml
AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://<AIRFLOW_POSTGRES_USER> :<AIRFLOWDB_PASSWORD>@<POSTGRES_CONTENEUR_NAME>:5432/<AIRFLOW_POSTGRES_DATABASE>
```

3. Le service triggerer : S'assurer qu'il est bien présent (obligatoire en Airflow 3).

4. Intégration au réseau : Association de tous les services Airflow au réseau externe app-network.

```yaml
networks:
    app-network:
        external: true
```

5. Suppression du service PostgreSQL intégré : Le bloc *postgres:* ainsi que son volume dédié (postgres-db-volume) sont à retirer du fichier Compose, puisque l'instance <POSTGRES_CONTENEUR_NAME> est gérée en amont.


6. Correction de la syntaxe depends_on : Adaptation de la structure des dépendances sous forme de mapping (dictionnaire) pour satisfaire les exigences des versions récentes de Docker Compose (notamment pour la condition service_completed_successfully).


## Configuration des Variables d'Environnement (.env)
Créer un fichier .env à la racine du projet Airflow pour stocker les secrets et paramètres requis par la version 3 d'Airflow :

```env
AIRFLOW_UID=50000
FERNET_KEY=<ta_cle_fernet_securisee_generee_via_python>
_AIRFLOW_WWW_USER_CREATE=true
_AIRFLOW_WWW_USER_USERNAME=admin
_AIRFLOW_WWW_USER_PASSWORD=admin
```

- FERNET_KEY : Clé symétrique encodée en Base64 indispensable au chiffrement des données sensibles (mots de passe, connexions, tokens) en base de données.

- AIRFLOW_UID : Garantit l'exécution des conteneurs avec un utilisateur non-root pour respecter les standards de sécurité.

## Procédure de Déploiement
1. Initialisation de la base de données

    Exécution de la commande d'initialisation pour appliquer les migrations de schéma et créer l'utilisateur administrateur :
```bash
docker compose up airflow-init
```

2. Lancement des services en arrière-plan

    Une fois l'initialisation validée, démarrage de la stack complète :
```bash
docker compose up -d
```

3. Vérification et Accès

    - Contrôle des conteneurs : *docker ps*

    - Interface Web : Accessible sur le port 8080 de la machine (http://<IP_VM>:8080) avec les identifiants définis dans le fichier .env.