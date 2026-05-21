# Exercice 6 — Métriques LogQL et expressions d’alerte

## Objectif

Dans cet exercice, j’ai étudié l’utilisation de LogQL pour transformer des logs textuels en véritables métriques de séries temporelles exploitables pour le monitoring et l’alerting.

L’objectif était de :

- calculer un taux d’erreurs par seconde ;
- regrouper les résultats par service ;
- créer une expression d’alerte Grafana ;
- détecter automatiquement une augmentation anormale des erreurs applicatives.

---

# 1. Génération des logs applicatifs

J’ai commencé par générer des logs simulant plusieurs services applicatifs.

Les logs contenaient :

- un niveau de log ;
- un nom de service ;
- un message.

Exemple de logs générés :

```text
level=ERROR service=auth message="Erreur authentification"

level=ERROR service=payment message="Erreur paiement"

level=INFO service=auth message="Connexion réussie"
```

Ces logs étaient écrits automatiquement dans le fichier :

```text
logs/apps/app.log
```

---

# 2. Objectif de la requête métrique

L’objectif était de transformer les logs de niveau :

```text
ERROR
```

en métriques temporelles.

Le résultat attendu était :

- un taux d’erreurs par seconde ;
- calculé sur une fenêtre de 5 minutes ;
- regroupé par service.

---

# 3. Requête LogQL métrique

J’ai utilisé la requête suivante dans Grafana Explore :

```logql
sum by (service) (
  rate({job="http-app"} |= "level=ERROR" | logfmt [5m])
)
```

---

# 4. Analyse détaillée de la requête

## Sélection des logs

```logql
{job="http-app"}
```

Cette partie sélectionne uniquement les logs provenant de l’application HTTP surveillée.

---

## Filtrage des erreurs

```logql
|= "level=ERROR"
```

Ce filtre conserve uniquement les logs contenant :

```text
level=ERROR
```

Les logs de niveau INFO ou WARNING sont ignorés.

---

## Parsing des champs logfmt

```logql
| logfmt
```

Cette instruction parse automatiquement les paires :

```text
clé=valeur
```

et extrait notamment :

- `service`
- `level`
- `message`

---

## Fenêtre temporelle

```logql
[5m]
```

Cette fenêtre indique que le calcul est effectué sur les 5 dernières minutes.

---

## Calcul du taux

```logql
rate(...)
```

Cette fonction calcule le nombre moyen de logs par seconde.

Le résultat obtenu représente donc :

```text
erreurs/seconde
```

---

## Regroupement par service

```logql
sum by (service)
```

Cette opération regroupe les résultats selon le champ :

```text
service
```

Chaque service possède donc sa propre métrique.

---

# 5. Résultat obtenu

Le résultat affiché dans Grafana ressemblait à :

```text
service="auth"      1.0

service="payment"   1.0
```

Cela signifiait :

- environ 1 erreur/seconde pour le service `auth` ;
- environ 1 erreur/seconde pour le service `payment`.

---

# 6. Création de l’expression d’alerte

L’objectif suivant était de créer une alerte Grafana se déclenchant lorsque :

```text
plus de 5 erreurs/seconde sont observées pendant plus de 2 minutes
```

Expression utilisée :

```logql
sum by (service) (
  rate({job="http-app"} |= "level=ERROR" | logfmt [5m])
) > 5
```

---

# 7. Configuration de l’alerte Grafana

Dans Grafana Alerting, j’ai configuré :

## Requête A

```logql
sum by (service) (
  rate({job="http-app"} |= "level=ERROR" | logfmt [5m])
)
```

---

## Condition

```text
WHEN A IS ABOVE 5
```

---

## Durée minimale

```text
FOR 2m
```

Cette configuration permettait d’éviter les faux positifs liés à des pics temporaires très courts.

---

# 8. Simulation d’une surcharge d’erreurs

Afin de tester l’alerte, j’ai volontairement généré un grand nombre d’erreurs :

```bash
while true; do
  for i in $(seq 1 10); do
    echo 'level=ERROR service=auth message="Erreur massive authentification"' >> logs/apps/app.log
  done
  sleep 1
done
```

Cette boucle générait plus de :

```text
10 erreurs/seconde
```

sur le service :

```text
auth
```

---

# 9. Vérification de l’alerte

Après quelques minutes, Grafana déclenchait correctement une alerte sur le service :

```text
auth
```

Le service :

```text
payment
```

ne déclenchait aucune alerte car son taux d’erreurs restait inférieur au seuil configuré.

---
