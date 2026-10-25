# Configuration du Tunnel Cloudflare

Ce document détaille la mise en place d'un tunnel Cloudflare chiffré (`cloudflared`) sur la machine virtuelle Hetzner. Cette configuration permet d'exposer de manière sécurisée les services internes (ex: Airflow, applications web) sur des sous-domaines dédiés sans ouvrir de ports d'entrée (80, 443, etc.) sur le pare-feu (UFW) de la VM.

---

## 1. Installation du client `cloudflared`

L'installation est réalisée sous un utilisateur non-root sécurisé `monuser`.

```bash
# Téléchargement du paquet officiel (.deb) pour Linux amd64
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb

# Installation du paquet via dpkg
sudo dpkg -i cloudflared.deb

# Vérification de la version installée
cloudflared --version
```

---

## 2. Authentification auprès de Cloudflare

Pour lier le serveur au compte Cloudflare et au domaine principal :

```bash
cloudflared tunnel login
```

1. Copier l'URL générée dans le terminal et l'ouvrir dans un navigateur.
2. Sélectionner le nom de domaine pour autoriser l'accès.
3. Le certificat d'origine (`cert.pem`) est téléchargé automatiquement sur la VM.

---

## 3. Création du Tunnel

Création du tunnel nommé `hetzner-vm-tunnel` :

```bash
cloudflared tunnel create hetzner-vm-tunnel
```

> **Note :** Cette commande génère un identifiant unique (UUID) et crée le fichier de paires de clés JSON associé dans le dossier de l'utilisateur.

---

## 4. Configuration du Tunnel et Routage DNS

### A. Fichier de configuration système

Le fichier de configuration principal est placé dans `/etc/cloudflared/config.yml` pour être exploité par le service système.

```bash
# Création du répertoire de configuration système
sudo mkdir -p /etc/cloudflared

# Déplacement des identifiants vers le dossier système
sudo cp /home/monuser/.cloudflared/<UUID_DU_TUNNEL>.json /etc/cloudflared/
sudo cp /home/monuser/.cloudflared/cert.pem /etc/cloudflared/
```

Fichier `/etc/cloudflared/config.yml` :

```yaml
tunnel: <UUID_DU_TUNNEL>
credentials-file: /etc/cloudflared/<UUID_DU_TUNNEL>.json

ingress:
  # Routage du sous-domaine vers le service local (port 80)
  - hostname: test.mondomaine.com
    service: http://localhost:80
  
  # Règle de repli par défaut (4404 si sous-domaine non configuré)
  - service: http_status:404
```

### B. Routage DNS

Association du sous-domaine au tunnel dans la zone DNS Cloudflare :

```bash
cloudflared tunnel route dns hetzner-vm-tunnel test.mondomaine.com
```

---

## 5. Gestion du Service systemd (Daemon)

Pour garantir la persistance du tunnel au redémarrage du serveur et en arrière-plan :

```bash
# Installation du service systemd
sudo cloudflared service install

# Activation et démarrage du service
sudo systemctl daemon-reload
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

### Commandes de vérification et gestion :

* **Statut du service :** `sudo systemctl status cloudflared` *(Touche `q` pour quitter)*
* **Inspection des logs :** `sudo journalctl -u cloudflared.service -n 50 --no-pager`

---

## 6. Validation de l'exposition

La validation s'effectue en consultant l'URL `https://test.mondomaine.com` dans un navigateur. 

Une réponse **502 Bad Gateway** émise par Cloudflare confirme que :
1. La résolution DNS pointe correctement vers le tunnel.
2. Le flux HTTPS/TLS est géré par Cloudflare.
3. Le tunnel transmet le trafic jusqu'à la VM (l'erreur 502 indique simplement qu'aucun serveur web local ne répond encore sur le port 80).

## 7. Validation et Test du Tunnel (Smoke Test)

Pour valider le bon fonctionnement de bout en bout du tunnel Cloudflare avant de déployer tes applications de production, il est possible de déployer un conteneur de test ultra-léger (ex: `hashicorp/http-echo`) et de le lier au tunnel.

### A. Lancer un conteneur de test
Démarre un conteneur léger en exposant son port sur l'hôte (requis pour que `cloudflared` puisse l'atteindre via `localhost`) :

```bash
docker run -d \
  --name web-test \
  -p 5678:5678 \
  --restart unless-stopped \
  hashicorp/http-echo:latest \
  -text="Hello from Hetzner VM via Cloudflare Tunnel!" \
  -listen=":5678"
```
### B. Mettre à jour la configuration du Tunnel

Modifie le fichier de configuration /etc/cloudflared/config.yml pour router le sous-domaine de test vers le port local :

```yaml
tunnel: <UUID_DU_TUNNEL>
credentials-file: /etc/cloudflared/<UUID_DU_TUNNEL>.json

ingress:
  - hostname: test.tonvraiddomaine.com
    service: http://localhost:5678
  - service: http_status:404
```

### C. Redémarrer le service Cloudflare

Appliquer les modifications en redémarrant le service systemd :

```bash
sudo systemctl restart cloudflared
```

### D. Nettoyage après validation
Une fois le test validé sur le navigateur (https://test.tonvraiddomaine.com), nettoyer le conteneur de test et retirer la règle du fichier config.yml pour laisser le tunnel propre :

```bash
docker rm -f web-test
```