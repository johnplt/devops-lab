# Provisionnement, Domaine DNS & Durcissement Sécurité (Phase 1 & Phase 2)

Ce document retrace le provisionnement de l'infrastructure, la gestion du nom de domaine personnalisé ainsi que la sécurisation complète (hardening) de la machine virtuelle Hetzner.

---

## 1. Spécifications du Serveur & Clé SSH

### A. Caractéristiques de l'instance Hetzner
- **Provider** : Hetzner Cloud
- **Instance** : CX23 (2 vCPU, 4 Go RAM, 40 Go NVMe SSD)
- **Localisation** : Helsinki (hel1)
- **OS** : Ubuntu 24.04 LTS
- **Réseau** : Adresse IPv4 + IPv6 dédiées

### B. Génération de la clé SSH (Machine Locale)
```bash
ssh-keygen -t ed25519 -C "hetzner-devops-lab" -f ~/.ssh/id_ed25519_hetzner
```

---

## 2. Gestion du Domaine, Portfolio & CI/CD

### A. Acquisition et Gestion DNS (Cloudflare Registrar)
- **Nom de domaine** : `mondomaine.com` acheté et géré directement via **Cloudflare Registrar**.
- **Gestion DNS & SSL** : Serveurs de noms (Nameservers) gérés en natif par Cloudflare avec certificats SSL/TLS universels automatiques.

### B. Déploiement Portfolio & Redirection (Cloudflare Pages)
- **Hébergement Statique** : Déploiement du portfolio HTML/CSS/JS sur **Cloudflare Pages**.
- **Routage Custom Domains** :
  - Apex domain : `mondomaine.com` (Actif)
  - Subdomain : `www.mondomaine.com` (Actif avec redirection transparente)

### C. Sécurisation du Dépôt & Intégration Continue (CI/CD)
- **Visibilité du Dépôt GitHub** : Dépôt du portfolio en mode **`Private`**.
- **Pipeline CI/CD** : Intégration native entre GitHub Private et Cloudflare Pages. Tout `git push` sur la branche principale déclenche automatiquement un build et un déploiement sans exposer le code source au public.

---

## 3. Phase 2 — Hardening & Sécurisation Système (VM Hetzner)

Pour appliquer le principe du moindre privilège, l'accès SSH direct en `root` est entièrement désactivé au profit d'un utilisateur d'administration non-root.

### A. Création de l'utilisateur dédié
Connecté initialement en `root` sur la VM :
```bash
# 1. Créer l'utilisateur non-root
adduser monuser

# 2. Accorder les privilèges d'administration via sudo
usermod -aG sudo monuser
```

### B. Copie de la clé SSH vers `monuser`
Depuis la **machine locale** :
```bash
ssh-copy-id -i ~/.ssh/id_ed25519_hetzner monuser@<IP_PUBLIQUE_VM>
```

Validation de la connexion par clé :
```bash
ssh -i ~/.ssh/id_ed25519_hetzner monuser@<IP_PUBLIQUE_VM>
```

### C. Verrouillage Strict du Service SSH (`/etc/ssh/sshd_config`)
Modification du fichier de configuration du démon SSH :
```bash
sudo nano /etc/ssh/sshd_config
```

Application des directives de sécurité :
```text
PermitRootLogin no
PasswordAuthentication no
```

Redémarrage du service SSH pour appliquer le verrouillage :
```bash
sudo systemctl restart ssh
```

---

## 4. Configuration du Pare-feu (UFW)

Activation du firewall applicatif Ubuntu pour ne laisser ouvert que le port SSH strict :

```bash
# Autoriser le trafic SSH
sudo ufw allow OpenSSH

# Activer le pare-feu
sudo ufw enable

# Vérifier le statut
sudo ufw status verbose
```