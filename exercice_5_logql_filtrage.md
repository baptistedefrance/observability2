# Exercice 5 — Filtrage avancé et requêtes de formatage LogQL

## Objectif

Dans cet exercice, j’ai étudié les fonctionnalités avancées de filtrage et de formatage proposées par LogQL.

L’objectif était de :

- filtrer des logs HTTP ;
- exclure certains codes de réponse ;
- parser automatiquement des logs au format logfmt ;
- reformater dynamiquement l’affichage des logs dans Grafana.

---

# 1. Génération de logs HTTP

J’ai commencé par générer des logs fictifs simulant des requêtes HTTP applicatives.

Les logs étaient produits au format :

```text
logfmt
```

Ce format repose sur des paires :

```text
clé=valeur
```

Exemple de logs générés :

```text
level=INFO method=GET path=/api/users protocol=HTTP/1.1 status=200 message="Utilisateur récupéré"

level=ERROR method=POST path=/api/login protocol=HTTP/1.1 status=500 message="Erreur authentification"

level=WARNING method=GET path=/api/admin protocol=HTTP/1.1 status=403 message="Accès refusé"
```

Ces logs étaient écrits automatiquement dans le fichier :

```text
logs/apps/app.log
```

---

# 2. Configuration de Grafana Alloy

Pour cet exercice, aucune transformation n’était réalisée dans Alloy.

Le rôle d’Alloy consistait uniquement à :

- surveiller le fichier de logs ;
- collecter les lignes ;
- envoyer les logs vers Loki.

Le parsing et le filtrage étaient directement effectués dans LogQL depuis Grafana Explore.

---

# 3. Filtrage des logs avec LogQL

J’ai utilisé la requête suivante :

```logql
{job="http-app"} |= "HTTP/1.1" != "status=200"
```

Cette requête réalisait deux opérations :

- conservation uniquement des lignes contenant :

```text
HTTP/1.1
```

- exclusion des lignes contenant :

```text
status=200
```

Le résultat affichait uniquement :

- les erreurs HTTP 500 ;
- les réponses HTTP 403.

Les réponses HTTP 200 étaient correctement filtrées.

---

# 4. Parsing automatique du format logfmt

J’ai ensuite utilisé le pipeline LogQL :

```logql
| logfmt
```

Ce pipeline permettait à Loki d’interpréter automatiquement les champs :

- `level`
- `method`
- `path`
- `status`
- `message`

Chaque clé devenait ensuite exploitable individuellement dans la requête.

---

# 5. Reformatage de l’affichage des logs

L’objectif suivant était de modifier complètement l’affichage des logs.

J’ai utilisé :

```logql
| line_format
```

Requête complète utilisée :

```logql
{job="http-app"} |= "HTTP/1.1" != "status=200"
| logfmt
| line_format "[{{.level}}] -> {{.message}}"
```

---

# 6. Résultat obtenu

Le résultat affiché dans Grafana était :

```text
[ERROR] -> Erreur authentification

[WARNING] -> Accès refusé
```

Les logs étaient désormais beaucoup plus lisibles.

Les informations inutiles étaient supprimées de l’affichage final.

---

# 7. Analyse des opérateurs utilisés

## Filtre positif

```logql
|= "HTTP/1.1"
```

Cet opérateur conserve uniquement les lignes contenant une chaîne spécifique.

---

## Filtre négatif

```logql
!= "status=200"
```

Cet opérateur exclut les lignes contenant une chaîne spécifique.

---

## Parsing logfmt

```logql
| logfmt
```

Ce pipeline parse automatiquement les paires :

```text
clé=valeur
```

et transforme les champs en variables exploitables.

---

## Reformatage de sortie

```logql
| line_format
```

Ce pipeline permet de reconstruire entièrement l’affichage des logs.

