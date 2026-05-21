# Exercice 7 — Métadonnées structurées (Structured Metadata)

## Objectif

Dans cet exercice, j’ai étudié le mécanisme des Structured Metadata dans Grafana Loki.

L’objectif était de :

- extraire un identifiant unique depuis les logs ;
- associer cette donnée aux lignes de logs ;
- éviter de surcharger l’index Loki avec des labels à forte cardinalité ;
- interroger cette métadonnée dans Grafana Explore.

Le champ utilisé était :

```text
trace_id
```

---

# 1. Compréhension du problème de cardinalité

Dans Loki, les labels sont indexés.

Lorsqu’un label possède énormément de valeurs différentes, cela provoque :

- une forte augmentation de la cardinalité ;
- une surcharge mémoire ;
- une dégradation des performances ;
- une augmentation de la taille de l’index.

Un champ comme :

```text
trace_id
```

possède généralement une valeur unique par requête applicative.

Il ne doit donc pas être utilisé comme label Loki classique.

Grafana Loki propose pour cela les :

```text
Structured Metadata
```

Ces métadonnées sont associées aux logs sans être indexées globalement comme des labels.

---

# 2. Modification de la configuration Loki

J’ai commencé par modifier la configuration Loki afin d’autoriser les Structured Metadata.

Dans le fichier :

```text
loki/loki-config.yml
```

j’ai activé :

```yaml
limits_config:
  allow_structured_metadata: true
  reject_old_samples: false
```

Puis j’ai redémarré Loki :

```bash
docker compose restart loki
```

---

# 3. Génération de logs JSON

J’ai ensuite généré des logs JSON contenant :

- un niveau de log ;
- un service ;
- un identifiant de trace ;
- un message.

Exemple de logs produits :

```json
{"level":"info","service":"auth","trace_id":"trace-1001","message":"Connexion utilisateur réussie"}

{"level":"error","service":"payment","trace_id":"trace-2002","message":"Erreur paiement"}

{"level":"warning","service":"api","trace_id":"trace-3003","message":"Latence élevée"}
```

Ces logs étaient écrits dans :

```text
logs/apps/app.log
```

---

# 4. Configuration du pipeline Alloy

J’ai ensuite modifié le pipeline Alloy afin de :

- parser le JSON ;
- créer certains labels Loki classiques ;
- stocker `trace_id` en Structured Metadata.

---

# 5. Parsing du JSON

J’ai utilisé :

```hcl
stage.json
```

afin d’extraire :

- `level`
- `service`
- `trace_id`
- `message`

Configuration utilisée :

```hcl
stage.json {
  expressions = {
    level    = "level",
    service  = "service",
    trace_id = "trace_id",
    message  = "message",
  }
}
```

---

# 6. Création des labels Loki

J’ai transformé les champs :

- `service`
- `level`

en labels Loki grâce à :

```hcl
stage.labels {
  values = {
    service = "",
    level   = "",
  }
}
```

Ces labels restaient indexés dans Loki.

---

# 7. Création de la Structured Metadata

J’ai ensuite utilisé :

```hcl
stage.structured_metadata
```

afin de stocker :

```text
trace_id
```

comme métadonnée structurée.

Configuration utilisée :

```hcl
stage.structured_metadata {
  values = {
    trace_id = "",
  }
}
```

Le champ `trace_id` était donc attaché aux logs sans être ajouté à l’index global Loki.

---

# 8. Modification du contenu final du log

J’ai utilisé :

```hcl
stage.output
```

afin de conserver uniquement le message applicatif :

```hcl
stage.output {
  source = "message"
}
```

---

# 9. Vérification dans Grafana Explore

Depuis Grafana Explore, j’ai exécuté :

```logql
{job="structured-metadata-app"}
```

Les logs apparaissaient correctement :

```text
Connexion utilisateur réussie

Erreur paiement

Latence élevée
```

En ouvrant les détails d’une ligne de log, le champ :

```text
trace_id
```

était bien visible dans les métadonnées.

---

# 10. Vérification de l’absence de surcharge des labels

J’ai ensuite vérifié que :

```text
trace_id
```

n’apparaissait pas comme label principal dans Loki.

Cela confirmait qu’il n’était pas indexé globalement.

Les seuls labels visibles restaient :

- `service`
- `level`
- `job`

---

# 11. Recherche par Structured Metadata

J’ai ensuite effectué une recherche spécifique :

```logql
{job="structured-metadata-app"} | trace_id="trace-2002"
```

Le résultat affichait uniquement :

```text
Erreur paiement
```

Cela démontrait que la Structured Metadata restait interrogeable sans être un label Loki classique.

---
