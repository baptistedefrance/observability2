# Exercice 4 — Pipelines Alloy et Parsing à la source

## Objectif

Dans cet exercice, j’ai étudié le fonctionnement des pipelines de traitement dans Grafana Alloy.

L’objectif était de :

- produire des logs fictifs au format JSON ;
- parser automatiquement les structures JSON ;
- extraire certaines informations ;
- enrichir les logs avec des labels ;
- filtrer certains événements avant leur envoi vers Loki.

Le traitement était effectué directement au niveau de la couche de collecte grâce au composant :

```text
loki.process
```

---

# 1. Génération de logs JSON

J’ai commencé par créer un fichier de logs applicatifs :

```bash
mkdir -p logs/apps
touch logs/apps/app.log
```

J’ai ensuite généré des logs fictifs au format JSON grâce à une boucle shell.

Les logs produits contenaient :

- un niveau de log ;
- un identifiant utilisateur ;
- un message applicatif.

Exemple de logs générés :

```json
{"level":"info","user_id":"1001","message":"Connexion utilisateur réussie"}
{"level":"debug","user_id":"1002","message":"Debug technique interne"}
{"level":"error","user_id":"1003","message":"Erreur authentification"}
```

Cette étape permettait de simuler une application moderne produisant des logs structurés JSON.

---

# 2. Configuration de Grafana Alloy

J’ai ensuite modifié le pipeline Alloy afin de traiter les logs avant leur envoi vers Loki.

La configuration utilisée comportait :

- `local.file_match`
- `loki.source.file`
- `loki.process`
- `loki.write`

Le composant `loki.process` permettait de créer un véritable pipeline de traitement des logs.

---

# 3. Parsing du JSON

J’ai utilisé le stage :

```hcl
stage.json
```

afin d’extraire automatiquement plusieurs champs JSON :

- `level`
- `user_id`
- `message`

Configuration utilisée :

```hcl
stage.json {
  expressions = {
    level   = "level",
    user_id = "user_id",
    message = "message",
  }
}
```

Grâce à ce traitement, Alloy pouvait manipuler individuellement chaque champ du log JSON.

---

# 4. Filtrage des logs DEBUG

L’objectif suivant était de supprimer les logs de niveau :

```text
debug
```

avant leur ingestion dans Loki.

J’ai utilisé le stage :

```hcl
stage.drop
```

Configuration :

```hcl
stage.drop {
  source = "level"
  value  = "debug"
}
```

Cette configuration supprimait automatiquement tous les logs dont le champ :

```text
level=debug
```

Les logs de debug n’étaient donc jamais envoyés vers Loki.

---

# 5. Extraction dynamique du user_id

J’ai ensuite transformé le champ :

```text
user_id
```

en label Loki grâce au stage :

```hcl
stage.labels
```

Configuration utilisée :

```hcl
stage.labels {
  values = {
    user_id = "",
  }
}
```

Cela permettait ensuite de filtrer les logs directement dans Grafana selon l’utilisateur concerné.

---

# 6. Modification du contenu final du log

J’ai utilisé le stage :

```hcl
stage.output
```

afin de remplacer le contenu brut JSON par le champ :

```text
message
```

Configuration utilisée :

```hcl
stage.output {
  source = "message"
}
```

Ainsi, Grafana affichait directement des messages lisibles au lieu du JSON complet.

---

# 7. Redémarrage et vérification

Après modification de la configuration, j’ai redémarré Alloy :

```bash
docker compose restart alloy
```

Puis j’ai vérifié les logs du service :

```bash
docker logs alloy
```

Aucune erreur n’était présente.

---

# 8. Vérification dans Grafana

Depuis Grafana Explore, j’ai exécuté plusieurs requêtes LogQL.

Requête générale :

```logql
{job="json-app"}
```

Résultat observé :

```text
Connexion utilisateur réussie
Erreur authentification
```

Le log :

```text
Debug technique interne
```

n’apparaissait plus.

Cela confirmait que le filtrage des logs DEBUG fonctionnait correctement.

---

# 9. Vérification des labels dynamiques

J’ai ensuite testé les labels dynamiques :

```logql
{job="json-app", user_id="1001"}
```

et :

```logql
{job="json-app", user_id="1003"}
```

Les résultats étaient correctement filtrés selon l’identifiant utilisateur.

Cela confirmait que le champ JSON `user_id` était bien transformé en label Loki.

---


