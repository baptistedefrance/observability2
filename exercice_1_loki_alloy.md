# Exercice 1 — Déploiement mono-nœud Grafana Loki & Grafana Alloy

## Objectif

Dans cet exercice, j’ai déployé une architecture de centralisation de logs basée sur :

- Grafana
- Grafana Loki
- Grafana Alloy
- Docker Compose

L’objectif était de mettre en place une chaîne complète de collecte et de visualisation de logs :

```text
Conteneurs Docker → Grafana Alloy → Loki → Grafana
```

---

# 1. Préparation de l’environnement

J’ai créé une arborescence dédiée au projet afin de séparer les différents fichiers de configuration :

```text
tp-loki-alloy/
├── docker-compose.yml
├── alloy/
│   └── config.alloy
├── loki/
│   └── loki-config.yml
└── grafana/
    └── provisioning/
        └── datasources/
            └── loki.yml
```

Cette organisation permet de maintenir une configuration claire et facilement maintenable.

---

# 2. Déploiement des conteneurs Docker

J’ai utilisé Docker Compose pour déployer les différents services nécessaires :

- Loki : stockage et indexation des logs
- Grafana : visualisation des logs
- Alloy : agent de collecte
- log-generator : conteneur de test générant des logs automatiquement

Le fichier `docker-compose.yml` permet d’automatiser entièrement le déploiement de l’environnement.

J’ai ensuite lancé la stack avec :

```bash
docker compose up -d
```

Puis j’ai vérifié le bon démarrage des conteneurs :

```bash
docker ps
```

---

# 3. Configuration de Loki

J’ai configuré Loki avec un stockage local basé sur le système de fichiers.

Le fichier `loki-config.yml` définit :

- le port d’écoute HTTP ;
- le stockage des logs ;
- le schéma d’indexation ;
- les limites d’ingestion.

Une difficulté rencontrée concernait le montage Docker : le fichier de configuration avait été créé comme un dossier au lieu d’un fichier, ce qui empêchait Loki de démarrer correctement.

Après correction du volume et du fichier YAML, Loki a démarré normalement.

J’ai validé son fonctionnement avec :

```bash
curl http://localhost:3100/ready
```

---

# 4. Configuration de Grafana Alloy

J’ai utilisé Grafana Alloy comme agent de collecte unique.

Son rôle était :

- de découvrir automatiquement les conteneurs Docker ;
- de récupérer leurs logs ;
- d’envoyer ces logs vers Loki.

Le fichier `config.alloy` contient :

- une découverte dynamique des conteneurs Docker ;
- une source Loki Docker ;
- une destination Loki.

J’ai monté le socket Docker :

```text
/var/run/docker.sock
```

afin qu’Alloy puisse accéder aux logs des conteneurs.

---

# 5. Configuration de Grafana

J’ai configuré Grafana afin d’ajouter automatiquement Loki comme datasource grâce au provisioning YAML.

Les identifiants utilisés étaient :

```text
admin / admin
```

J’ai ensuite accédé à l’interface web :

```text
http://localhost:3000
```

---

# 6. Vérification du flux de logs

Afin de générer des logs en continu, j’ai utilisé un conteneur Docker de test :

```text
log-generator
```

Ce conteneur écrivait automatiquement des logs toutes les deux secondes.

J’ai vérifié les logs localement avec :

```bash
docker logs -f log-generator
```

Puis j’ai utilisé Grafana Explore pour interroger Loki.

Requête utilisée :

```logql
{container_name=~".+"}
```

Au départ, aucun log n’apparaissait. Après analyse des logs Alloy, j’ai identifié une erreur :

```text
timestamp too old
```

Loki refusait des anciens logs Docker présents sur la machine.

Pour corriger ce problème, j’ai :

- supprimé les anciens volumes Docker ;
- redémarré la stack ;
- recréé les conteneurs.

Après cette correction, les logs étaient correctement visibles dans Grafana.

---
