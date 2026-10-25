# Installation Docker Engine, Groupe Non-Root & Réseau Interne (`app-network`)

Ce document décrit l'installation du moteur Docker sous l'utilisateur d'administration non-root (`monuser`), la suppression de la dépendance aux privilèges `root` directs pour les conteneurs, et la mise en place du réseau privé partagé.

---

## 1. Installation propre de Docker Engine & Docker Compose

L'installation est réalisée depuis les dépôts officiels Docker sous un utilisateur non-root.

### A. Nettoyage et dépendances système
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release git
```

### B. Configuration du dépôt officiel Docker
```bash
# Ajout de la clé GPG officielle
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Configuration du dépôt APT stable
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### C. Installation du Moteur Docker et des Plugins
```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

## 2. Configuration du Groupe Docker (Exécution non-Root)

Pour éviter d'exécuter le démon Docker via `sudo` et sécuriser l'exécution des conteneurs :

```bash
# 1. Ajouter l'utilisateur au groupe docker
sudo usermod -aG docker monuser

# 2. Recharger le groupe dans la session courante sans redémarrer
newgrp docker
```

### Validation du fonctionnement
```bash
# Vérifier les versions
docker --version
docker compose version

# Valider le lancement d'un conteneur sans sudo
docker run hello-world
```

---

## 3. Réseau Docker Partagé Inter-Conteneurs (`app-network`)

Pour permettre la communication interne sécurisée entre tous nos services applicatifs (PostgreSQL géré via Ansible, Airflow, Cloudflare Tunnel, APIs) sans exposer leurs ports sur Internet :

```bash
# Création du réseau bridge isolé
docker network create app-network

# Vérification du réseau
docker network ls
```

---

## 4. Architecture Globale du devops-lab

```text
                                  [ Internet ]
                                       │
                         ┌─────────────┴─────────────┐
                         │   Cloudflare Zero Trust   │
                         └─────────────┬─────────────┘
                                       │ (Tunnel Chiffré)
 ┌─────────────────────────────────────▼─────────────────────────────────────┐
 │ VM Hetzner (CX23 - Ubuntu 24.04 LTS) — User: monuser                      │
 │                                                                           │
 │  ┌─────────────────────────────────────────────────────────────────────┐  │
 │  │ Docker Internal Network (app-network)                               │  │
 │  │                                                                     │  │
 │  │   ┌────────────────┐     ┌────────────────┐     ┌────────────────┐  │  │
 │  │   │ Cloudflared    │────►│ PostgreSQL     │────►│ Apache Airflow │  │  │
 │  │   │ Tunnel Client  │     │ (via Ansible)  │     │ & Applications │  │  │
 │  │   └────────────────┘     └────────────────┘     └────────────────┘  │  │
 │  └─────────────────────────────────────────────────────────────────────┘  │
 └───────────────────────────────────────────────────────────────────────────┘
```

- **Base de données PostgreSQL** : Déployée de manière automatisée via les playbooks Ansible (`/ansible`) sur le réseau `app-network`.
- **Exposition sécurisée** : Le client `cloudflared` sera connecté sur `app-network` pour router le trafic en HTTPS vers les sous-domaines (ex: `airflow.mondomaine.com`) sans aucun port ouvert sur le pare-feu UFW.

---

## 5. Bonnes Pratiques d'Exploitation (`monuser`)

- **Projets / Stacks** : Tous les fichiers `docker-compose.yml` doivent être hébergés dans le dossier personnel `~/devops-lab/`.
- **Réseau Externe** : Chaque `docker-compose.yml` doit spécifier le réseau privé :
  ```yaml
  networks:
    app-network:
      external: true
  ```
- **Supervision rapide** :
  ```bash
  docker ps                # Conteneurs en cours d'exécution
  docker stats             # Consommation CPU / RAM
  docker logs -f <NAME>    # Consultation des logs en direct
  ```