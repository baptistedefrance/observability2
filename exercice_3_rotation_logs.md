# Exercice 3 — Rotation des logs et découverte dynamique

## Objectif

Dans cet exercice, j’ai étudié le fonctionnement de la collecte de fichiers logs avec Grafana Alloy ainsi que la gestion des rotations de fichiers.

L’objectif était de :

- surveiller dynamiquement un répertoire de logs ;
- collecter des fichiers `.log` ;
- simuler une activité applicative ;
- effectuer une rotation manuelle des fichiers ;
- vérifier qu’Alloy conserve correctement sa position de lecture sans perte ni duplication des logs.

---

# 1. Préparation du répertoire de logs

J’ai commencé par créer un répertoire dédié aux fichiers de logs :

```bash
mkdir -p logs/apps
```

Puis j’ai créé un premier fichier de logs vide :

```bash
touch logs/apps/app.log
```

Ce fichier représentait le fichier de log principal de l’application simulée.

---

# 2. Configuration du montage Docker

J’ai ensuite modifié le service Alloy dans le fichier `docker-compose.yml`.

Le dossier local :

```text
./logs
```

a été monté dans le conteneur Alloy sous :

```text
/var/log/apps
```

Cela permettait à Grafana Alloy d’accéder directement aux fichiers logs présents sur le système hôte.

---

# 3. Configuration de Grafana Alloy

J’ai configuré Alloy afin qu’il surveille automatiquement tous les fichiers :

```text
/var/log/apps/apps/*.log
```

La configuration utilisée reposait sur :

- `local.file_match`
- `loki.source.file`
- `loki.write`

Le composant `local.file_match` permettait de découvrir dynamiquement les fichiers logs.

Le composant `loki.source.file` assurait ensuite leur lecture continue et leur transmission vers Loki.

---

# 4. Génération de logs applicatifs

Afin de simuler une activité applicative, j’ai utilisé une boucle shell générant des logs en continu :

```bash
while true; do
  echo "$(date) INFO Application active" >> logs/apps/app.log
  sleep 2
done
```

Cette commande ajoutait automatiquement une nouvelle ligne dans le fichier `app.log` toutes les deux secondes.

---

# 5. Vérification dans Grafana

Depuis Grafana Explore, j’ai utilisé la requête suivante :

```logql
{job="applications"}
```

Les logs apparaissaient correctement en temps réel dans Grafana.

Cela confirmait que :

- Alloy détectait correctement le fichier ;
- la lecture des logs fonctionnait ;
- Loki stockait correctement les données ;
- Grafana affichait correctement les logs.

---

# 6. Simulation d’une rotation de logs

J’ai ensuite simulé une rotation manuelle des fichiers logs.

Pour cela, j’ai renommé le fichier principal :

```bash
mv logs/apps/app.log logs/apps/app.log.1
```

Puis j’ai créé un nouveau fichier vide :

```bash
touch logs/apps/app.log
```

Cette opération reproduisait le fonctionnement d’un outil de rotation comme :

- logrotate ;
- rsyslog ;
- applications Java ;
- serveurs web.

---

# 7. Reprise de l’écriture des logs

Après la rotation, j’ai continué à écrire des logs dans le nouveau fichier :

```bash
while true; do
  echo "$(date) INFO Nouveau fichier après rotation" >> logs/apps/app.log
  sleep 2
done
```

---

# 8. Vérification du comportement d’Alloy

Depuis Grafana, j’ai observé que :

- Alloy continuait la lecture sans interruption ;
- les anciens logs restaient présents ;
- les nouveaux logs étaient immédiatement collectés ;
- aucun doublon n’apparaissait ;
- aucune perte de logs n’était visible.

Cela démontrait que Grafana Alloy gérait correctement :

- les changements d’inode ;
- les renommages de fichiers ;
- la découverte dynamique des nouveaux fichiers.
