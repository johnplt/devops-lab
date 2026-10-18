# Provisionnement et Configuration Initiale du Serveur
1. Génération de la paire de clés SSH (Local)
Création d'une clé SSH sécurisée dédiée à la VM Hetzner sur la machine locale :
```bash
ssh-keygen -t ed25519 -C "hetzner-devops-lab" -f ~/.ssh/id_ed25519_hetzner
```
2. Configuration du Serveur sur Hetzner CloudInstance : 
    - Hetzner CX23 (Architecture x86 Intel/AMD, 2 vCPU, 4 GB RAM, 40 GB NVMe SSD).
    - Localisation : Helsinki.
    - OS : Ubuntu 24.04 LTS.
    - Réseau : IPv4 + IPv6.   


3. Authentification et Injection de la Clé SSH
En raison d'un oubli d'association de la clé SSH lors du provisioning initial dans la console Hetzner, l'authentification s'est faite en deux temps :
    - Première connexion via le mot de passe temporaire (reçu par email) :
    ```bash
    ssh root@<IP_PUBLIQUE_VM>
    ```
    - Injection manuelle de la clé SSH publique sur la VM pour activer l'accès sans mot de passe :
    ```bash
    mkdir -p ~/.ssh && chmod 700 ~/.ssh
    echo "ssh-ed25519 <VOTRE_CLE_PUBLIQUE_ICI> hetzner-devops-lab" >> ~/.ssh/authorized_keys
    chmod 600 ~/.ssh/authorized_keys
    ```
    - Validation de l'accès SSH sécurisé depuis la machine locale sans mot de passe :
    ```bash
    ssh -i ~/.ssh/id_ed25519_hetzner root@<IP_PUBLIQUE_VM>
    ```

# Mise à jour & Sécurisation de base
1. Mettre à jour les paquets Ubuntu :
```bash
apt update && apt upgrade -y
```

2. Configurer le pare-feu de base (UFW) pour autoriser uniquement SSH pour l'instant :
```bash
ufw allow OpenSSH
ufw enable
```

# Installation propre de Docker & Docker Compose

- Installation de Docker via le script officiel
```bash
curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh
```

- Vérification de l'installation
```bash
docker --version && docker compose version
```
