# devops-lab
Documentation technique des procédures devops de mes projets

## Architecture cible

```
┌─────────────────────────────────────────────────────────────────┐
│                      HETZNER CLOUD INSTANCE                     │
│  (Ubuntu 24.04 LTS / IP Publique / Docker + Docker Compose)     │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    DOCKER NETWORK (internal)              │  │
│  │                                                           │  │
│  │   ┌──────────────────┐          ┌──────────────────────┐  │  │
│  │   │  PostgreSQL 16   │◄─────────┤ Apache Airflow       │  │  │
│  │   │  (Base partagée) │          │ (Webserver/Scheduler)│  │  │
│  │   └────────┬─────────┘          └──────────────────────┘  │  │
│  │            │                                              │  │
│  │            ├────────────────────┬─────────────────────┐   │  │
│  │            ▼                    ▼                     ▼   │  │
│  │   ┌──────────────────┐ ┌──────────────────┐ ┌───────────┐ │  │
│  │   │  App 1 (Render)  │ │  App 2 (Render)  │ │  App 3 ...│ │  │
│  │   └──────────────────┘ └──────────────────┘ └───────────┘ │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Structure du repo

```
devops-lab/
└── ansible/
    ├── inventory/
    │   └── hosts.yaml
    ├── roles/
    │   ├── docker/
    │   │   └── tasks/
    │   │       └── main.yaml
    │   ├── postgres/
    │   │   ├── tasks/
    │   │   │   └── main.yaml
    │   │   ├── templates/
    │   │   │   └── docker-compose.yaml.j2
    │   │   └── vars/
    │   │       └── main.yaml
    │   └── airflow/
    │       ├── tasks/
    │       │   └── main.yaml
    │       ├── templates/
    │       │   └── docker-compose.yaml.j2
    │       └── vars/
    │           └── main.yaml
    └── site.yaml
```

- /docs/01-provisioning-vm.md : Guide de création de la VM, configuration SSH, pare-feu (UFW), durcissement de sécurité.
- /docs/02-architecture-docker-postgres.md : Mise en place du réseau Docker interne et du conteneur PostgreSQL mutualisé (gestion des volumes persistant, variables d'environnement).
- /docs/03-airflow-deployment.md : Installation d'Airflow (avec l'approche CeleryExecutor ou LocalExecutor selon ton besoin), connexion au Postgres partagé, et configuration des DAGs.
- /ansible : Scripts Ansible pour automatiser les déploiements

