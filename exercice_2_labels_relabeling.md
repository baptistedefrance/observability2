# Exercice 2 — Comprendre les labels et le mécanisme de Relabeling

## Objectif

Dans cet exercice, j’ai étudié le fonctionnement des labels dans Grafana Loki ainsi que le mécanisme de relabeling via Grafana Alloy.

L’objectif était de :

- supprimer un label natif inutile ;
- ajouter un label statique ;
- extraire dynamiquement un label depuis le contenu des logs ;
- vérifier l’indexation des labels dans Grafana Loki.

---

# 1. Modification du générateur de logs

Afin de pouvoir extraire dynamiquement un niveau de log, j’ai modifié le conteneur de génération de logs.

Le conteneur produit désormais plusieurs types de messages :

```text
INFO Application démarrée
ERROR Connexion impossible
WARNING Ressource lente
```

Ces logs sont générés automatiquement toutes les deux secondes.

Cette étape était nécessaire afin de pouvoir utiliser une expression régulière dans Alloy pour extraire dynamiquement le niveau de log.

---

# 2. Configuration de Grafana Alloy

J’ai ensuite modifié la configuration de Grafana Alloy afin d’ajouter plusieurs traitements sur les logs avant leur envoi vers Loki.

Le pipeline Alloy réalise désormais les actions suivantes :

- découverte automatique des conteneurs Docker ;
- collecte des logs Docker ;
- ajout d’un label statique ;
- extraction dynamique d’un label via une expression régulière ;
- suppression d’un label natif inutile ;
- envoi des logs vers Loki.

---

# 3. Ajout d’un label statique

J’ai ajouté le label suivant :

```text
environment=development
```

Ce label est ajouté automatiquement à tous les logs collectés.

Cette méthode permet de distinguer facilement plusieurs environnements :

- développement ;
- recette ;
- production.

Le traitement utilisé dans Alloy était :

```hcl
stage.static_labels {
  values = {
    environment = "development",
  }
}
```

---

# 4. Extraction dynamique du niveau de log

J’ai ensuite utilisé une expression régulière afin d’extraire automatiquement le niveau de log depuis le message.

Expression utilisée :

```hcl
stage.regex {
  expression = "^(?P<loglevel>INFO|ERROR|WARNING)"
}
```

Cette expression récupère :

- INFO
- ERROR
- WARNING

puis stocke cette valeur dans un champ nommé :

```text
loglevel
```

J’ai ensuite transformé cette valeur en véritable label Loki grâce au stage :

```hcl
stage.labels {
  values = {
    loglevel = "",
  }
}
```

---

# 5. Suppression d’un label natif

Par défaut, Alloy ajoute plusieurs labels automatiquement.

J’ai supprimé le label :

```text
filename
```

avec :

```hcl
stage.label_drop {
  values = ["filename"]
}
```

Cette étape permet de réduire la cardinalité des labels dans Loki.

Réduire le nombre de labels inutiles améliore :

- les performances ;
- l’indexation ;
- la consommation mémoire ;
- les temps de recherche.

---

# 6. Redémarrage des services

Après modification de la configuration, j’ai redémarré les services Docker :

```bash
docker compose down -v
docker compose up -d
```

Puis j’ai vérifié le bon fonctionnement d’Alloy :

```bash
docker logs alloy
```

---

# 7. Vérification dans Grafana

Depuis Grafana Explore, j’ai exécuté plusieurs requêtes LogQL afin de vérifier le bon fonctionnement des labels.

Requête permettant d’afficher uniquement les logs de l’environnement développement :

```logql
{environment="development"}
```

Requête affichant uniquement les erreurs :

```logql
{loglevel="ERROR"}
```

Requête affichant uniquement les logs d’information :

```logql
{loglevel="INFO"}
```

Les résultats confirmaient que les labels étaient correctement indexés par Loki.

---

# 8. Vérification des labels disponibles

Depuis le navigateur de labels Grafana, j’ai vérifié :

- la présence du label `environment` ;
- la présence du label `loglevel` ;
- la disparition du label `filename`.

Cela confirmait que le relabeling était correctement appliqué avant l’ingestion des logs dans Loki.


