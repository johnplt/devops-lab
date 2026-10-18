# Création de l'arborescence sur la VM
```bash
# 1. Créer le dossier principal du projet et le sous-dossier pour PostgreSQL
mkdir -p ~/devops-lab/postgres

# 2. Se placer dans le dossier postgres
cd ~/devops-lab/postgres
```

# Création du réseau Docker partagé
Pour que les futurs conteneurs (PostgreSQL, Airflow, applications) puissent communiquer entre eux sur un réseau interne privé et sécurisé, on va créer un réseau Docker nommé app-network :
```bash
docker network create app-network
```

# Création du fichier docker-compose.yml pour PostgreSQL
Dans le dossier *~/devops-lab/postgres*, on crée le fichier de configuration avec un éditeur de texte comme vi :
```bash
vi docker-compose.yml
```
Contenu du docker compose :
```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: <POSTGRES_CONTENEUR_NAME>
    restart: always
    environment:
      POSTGRES_USER: <POSTGRES_USER>
      POSTGRES_PASSWORD: <POSTGRES_PASSWORD> 
      POSTGRES_DB: <POSTGRES_DB> 
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network

volumes:
  postgres_data:

networks:
  app-network:
    external: true
```

- postgres:16-alpine : Une version légère et performante de PostgreSQL.

- postgres_data (volume) : Permet de conserver tes données même si le conteneur est redémarré ou détruit.

- networks: app-network : Connecte le service à notre réseau privé partagé.

# Lancer le conteneur
Une fois le fichier sauvegardé, on lance PostgreSQL en tâche de fond avec :
```bash
docker compose up -d
```

Puis vérifie qu'il tourne bien :
```bash
docker ps
```

Le conteneur *shared_postgres* doit être en état Up !

# Test rapide de la base de données
Pour vérifier que PostgreSQL réagit parfaitement, on exécute une commande SQL rapide à l'intérieur du conteneur :
```bash
docker exec -it <POSTGRES_CONTENEUR_NAME> psql -U <POSTGRES_USER> -d <POSTGRES_DB> -c "SELECT version();"
```
