# REST in peace

# Disclaimer

Ce document est une sorte de recette pour produire une API "REST" selon les bonnes pratiques en cours.

Il n'est pas exhaustif, des choix sont faits pour rester pragmatique quand il n'y a pas vraiment de bonnes pratiques.

On parle ici d'API REST au sens "marketing" du terme puisque ce document ne vise pas à atteindre le level3 du
[modèle de maturité de Richardson](http://blog.xebia.fr/2010/06/25/rest-richardson-maturity-model/). On parlera pluôt d'une API HTTP++ (définie très justement par [William Durant](https://youtu.be/u_jDzcXCimM?list=PL9zDdgiGjkIc_1wnKTdU68dmVZ77ayPwW)).

# Les bases

### Vocabulaire

URL = https://api.domain.com/v2/items/110e8400-e29b-11d4-a716-446655897563

Ressource = https://api.domain.com/v2/ **items/110e8400-e29b-11d4-a716-446655897563**

Représentation (ici JSON)
```json
{
  "id": "110e8400-e29b-11d4-a716-446655897563",
  "name": "Jo",
  "age": 18,
  "isGeek": true
}
```

### Protocol

Tous les appels doivent être faits via SSL.

### Encodage

Tout est encodé en UTF-8: la réponse (représentation) et la requète (header, body, querystring).

### Versioning

L'api doit être versionnée via l'URL: https://api.domain.com/v2

Pas plus de 2 versions en même temps sinon c'est ingérable.

> Possible via Header `Accept: application/json; version=2` mais par affordance et pour le côté pratique il vaut mieux utiliser l'URL

### Type de données

Type | Description
------------ | -------------
String | Encodée en UTF-8
Integer | Entier signé en 32-bit ou 64-bit
Float | Nombre flottant signé en 32-bit ou 64-bit
Boolean | true ou false
Date | Toujours UTC et au format [ISO8601](https://en.wikipedia.org/wiki/ISO_8601): `2015-07-20T20:10:55Z`

# Ressource

* Toujours au pluriel
* Nommée avec des - ou des _
* Ne reflète pas forcément votre modèle de donnée
* Une ressource = une URL
* Une ressource = plusieurs représentations (JSON, XML, ...)

### Interactions

Pour interagir avec les ressources, on s'appuie sur HTTP.

Requête | Action
------------ | -------------
GET /items | Liste d'item
GET /items/1782 | Item 1782
POST /items | Creation d'un nouvel item
PUT /items | Mise à jour de plusieurs items
PUT /items/1782 | Mise à jour de l'item 1782
DELETE /items/1782 | Suppression de l'item 1782


> PATCH devrait être utilisé pour faire des updates partielles à la place de PUT. Mais il y a du travail en plus si on veut gérer ça [correctement](http://williamdurand.fr/2014/02/14/please-do-not-patch-like-an-idiot/) et rester RESTful, ce qui n'est pas l'objectif de ce document :)

### Relations

On a parfois des relations entre nos ressources. On utilisera la même mécanique via HTTP.

Requête | Action
------------ | -------------
GET /items/1782/comments | Liste de commentaire de l'item 1782
GET /items/1782/comments/56 | Commentaire 56 de l'item #1782
POST /items/1782/comments | Création d'un commentaire pour l'item 1782
PUT /items/1782/comments/56 | Mise à jour du commentaire 56 pour l'item 1782
DELETE /items/1782/comments/56 | Suppression du commentaire 56 pour l'item 1782

### Actions

Il nous faut parfois effectuer des actions sur nos ressources, la  méthode **POST** est utilisée pour ajouter une nouvelle ressource; ici, la nouvelle ressource est l'action.

Requête | Action
------------ | -------------
POST /items/1782/translate | Traduit l'item 1782
POST /items/1782/enable | Active l'item 1782
POST /items/1782/comments/56/star | Met en favori le commentaire 56 de l'item 1782

Bien que généralement à éviter, une action est utile car elle peut permettre de:
* simplifier l'API (avoir une action "enable" plutôt que de devoir mettre à jour la ressource)
* ajouter des "fonctionnalités" qui ne change pas l'état de la ressource (translate)


# Représentation

On ne supporte que le format **JSON** pour la réponse.

> [Plus personne n'utilise XML](http://www.google.com/trends/explore?q=xml+api#q=xml%20api%2C%20json%20api&cmpt=q) sauf dans un contexte grand compte / DSI.

On retourne toujours un JSON pretty print. C'est plus human-friendly et ce n'est pas trop un problème avec la compression gzip.

Les ids des représentations sont des UUID. Cela permet de ne pas se marcher sur les pieds avec les IDs que pourrait avoir à gérer le client pour son business.

On n'enveloppe pas les réponses avec une propriété data ou item ici, ça n'a pas d'intérêt:

```json
{
  "id": "110e8400-e29b-11d4-a716-446655897563",
  "name": "Jo",
  "age": 18,
  "isGeek": true
}
```

Si on a des ressources imbriquées, on retourne:

```json
{
  "id": "110e8400-e29b-11d4-a716-446655897563",
  "name": "Jo",
  "age": 18,
  "isGeek": true,
  "country": {
    "id": "110e8400-e29b-11d4-a716-446655898965"
  }
}
```

Plutôt que:

```json
{
  "id": "110e8400-e29b-11d4-a716-446655897563",
  "name": "Jo",
  "age": 18,
  "isGeek": true,
  "country_id": "110e8400-e29b-11d4-a716-446655898965"
}
```

Ce qui nous permettra éventuellement de retourner la ressource imbriquée inline et ainsi de garder la même structure de représentation et d'éviter des requètes supplémentaires. On pourra utiliser un header `X-Resource-Nested: true` pour indiquer au serveur que l'on veut aussi les ressources imbriquées.

```json
{
  "id": "110e8400-e29b-11d4-a716-446655897563",
  "name": "Jo",
  "age": 18,
  "isGeek": true,
  "country": {
    "id": "110e8400-e29b-11d4-a716-446655898965",
    "name": "France",
    "codeISO": "FR"
  }
}
```


# Requête

On ne supporte que le format **JSON** pour la représentation de la réponse.

La requète doit donc avoir un header `Accept: application/json`

Retourner [`406 Not acceptable`](http://httpstatus.es/406) si on demande autre chose.

> Si on doit gérer XML par exemple `Accept: application/json; application/xml` mais on garde JSON en choix n°1

On accepte du JSON pour le body des requètes dans le cas d'un POST, PUT ou PATCH. Ca nous permet d'avoir la même sérialisation entre le body de la requète et le body de la réponse.
On pourra facilement passer des structures complètes ou partielles de ressources et bénéficier du typage JSON: Array, String, Number, Object, Boolean, Null.

La requête doit donc comporter un header Content-Type: `Content-Type: application/json;charset=utf-8`.

Retourner [`415 Unsupported media type`](http://httpstatus.es/415) si Content-type n'est pas supporté par le serveur.

> On peut supporter `Content-Type: application/x-www-form-urlencoded` en parallèle, à voir en fonction des clients qui consommeront l'API. Mais ça obligera côté serveur à typer les valeurs manuellement et on n'aura pas de structure de ressource out of box.

Il ne faudra pas oublier d'indiquer qu'on veut la réponse gzippée via `Accept-Encoding: gzip`.

> La plupart des clients supportent out of box gzip il ne faut pas s'en priver!

```bash
$ curl -X POST https://api.domain.com/v2/items \
    -H "Content-Type: application/json;charset=utf-8" \
    -H "Accept: application/json" \
    -H "Accept-Encoding: gzip" \
    -d '{"name": "Jo", "age": 18, "isGeek": true}'

{
  "id": "110e8400-e29b-11d4-a716-446655897563",
  "name": "Jo",
  "age": 18,
  "isGeek": true
}
```

# Réponse

Ajouter pour chaque requète un header `X-Request-UUID: 454684315618613`, ceci aidera le client dans son logging, debugging... en identifiant de manière unique chaque requête.

> Utiliser la version 1 de la RFC 4122 car elle est time-based et donc indexable séquentiellement

Il faudra aussi retourner le bon code HTTP:

HTTP status code | Information
------------ | -------------
[`200 Ok`](http://httpstatus.es/200) | GET, PUT, PATCH et DELETE ainsi que pour POST lors d'une "action"
[`201 Created`](http://httpstatus.es/201) | POST lors de la création d'un item
[`202 Accepted`](http://httpstatus.es/204) | La requête est ok, mais on la traitera plus tard
[`204 No Content`](http://httpstatus.es/204) | DELETE sans body
[`206 Partial content`](http://httpstatus.es/206) | Si la réponse ne renvoie pas l'ensemble de la ressource (une liste par ex)

Lors d'un [`200 Ok`](http://httpstatus.es/200) **on doit retourner la représentation**.

Lors d'un [`201 Ok`](http://httpstatus.es/201):
* on doit retourner la représentation
* on doit indiquer l'URL de la nouvelle ressource dans le header `Location: https://api.domain.com/v2/items/1783`

# Error

Lorsqu'une erreur survient, il faut que le client puisse comprendre ce qui se passe et éventuellement agir.
Il faut s'appuyer sur les status HTTP 40x et 50x qui répondent à tous les cas, même si dans la pratique une dizaine suffit. Il faut aussi retourner une réponse avec une structure qui sera toujours la même quelque soit l'erreur:

```json
{
  "code": "error_code",
  "description": "More details about the error here",
  "url": "https://doc.domain.com/api/error/error_code"
}
```

Exemples:

[`400 Bad Request`](http://httpstatus.es/400): on a mal formaté la requête, par exemple un body JSON non parsable.

```json
{
  "code": "invalid_request",
  "message": "Can't parse the request body, JSON not valid.",
  "url": "https://doc.domain.com/api/error/invalid_request"
}
```

[`422 Unprocessable Entity`](http://httpstatus.es/400): la requête est ok, mais les données envoyées ne sont pas valides.

```json
{
  "code": "invalid_item",
  "message": "Name is required, isGeek must be a boolean.",
  "url": "https://doc.domain.com/api/error/invalid_item"
}
```

Dans le cas de la validation, on peut avoir besoin de gérer un retour d'erreur plus fin et donc renvoyer un tableau d'erreurs.

```json
[
    {
        "code": "invalid_item_name",
        "message": "Name is required",
        "url": "https://doc.domain.com/api/error/invalid_item_name"
    },
    {
        "code": "invalid_item_geek",
        "message": "isGeek must be a boolean.",
        "url": "https://doc.domain.com/api/error/invalid_item_geek"
    },
]
```

Les principaux statut HTTP pour gérer les erreurs:

HTTP status code | Information
------------ | -------------
[`400 Bad Request`](http://httpstatus.es/400) | Requête mal formée (body non parsable etc...)
[`401 Unauthorized`](http://httpstatus.es/401) | Authentification invalide
[`403 Forbidden`](http://httpstatus.es/403) | Authentication ok, mais on a pas les droits
[`404 Not Found`](http://httpstatus.es/404) | Resource pas trouvée (inexistante ou suite à un `DELETE`)
[`405 Method Not Allowed`](http://httpstatus.es/405) | Méthode HTTP non autorisée (utilisation d'un `POST` alors qu'on attend un `DELETE`)
[`406 Not acceptable`](http://httpstatus.es/405) | Format de retour non disponible (la requête demande du XML alors qu'on ne gère que du JSON)
[`409 Conflict`](http://httpstatus.es/409) | `POST` une ressource avec un/des champs qui doivent être unique (duplication interdite)<br/>`PATCH` incompatible a été soumis par un autre agent
[`415 Unsupported Media Type`](http://httpstatus.es/415) | Content type pas supporté (on envoie du XML alors qu'on ne suppporte que JSON)
[`422 Unprocessable Entity`](http://httpstatus.es/422) | Tout ce qui touche à la validation
[`429 Too Many Requests`](http://httpstatus.es/429) | Trop de requêtes (on a dépassé le rate limit)
[`500 Internal Server Error`](http://httpstatus.es/500) | Certainement une coquille dans le code ^^
[`503 Service Unvailable`](http://httpstatus.es/429) | Lors d'une maintenance ou si l'on veut couper l'API

# Pagination

## Requête

On utilise la querystring:

```bash
$ curl -X GET https://api.domain.com/v2/items?page=2&per_page=100 \
    -H "Accept: application/json" \
    -H "Accept-Encoding: gzip" \
    -H "If-Modified-Since: Fri, 31 Jul 2015 20:41:30 GMT"
```

> On pourrait utiliser le header `Range` mais par affordance et pour le côté pratique il vaut mieux utiliser la querystring.

## Réponse

Le serveur doit retourner [`206 Partial content`](http://httpstatus.es/206) si on n'a pas toutes les ressources.

Si elles sont toutes retournée [`200 OK`](http://httpstatus.es/200)

Utiliser le header `Link` pour transmettre la pagination:
```http
Link: <https://api.domain.com/v2/items?page=3&per_page=100>; rel="next", <https://api.domain.com/v2/items?page=1&per_page=100>; rel="prev"
```
> Le client n'aura pas à construire la pagination.

> On peut aussi ajouter la première et la dernière page `rel=first` et `rel=last`.

Ajouter un header custom pour indiquer le nombre totale de ressources disponibles:
```http
X-Total-Count: 456
X-Page-Max-Range: 100
```
Le serveur doit retourner [`400 Bad request`](http://httpstatus.es/400) si on dépasse les capacités de l'API.

# Filtering, sort & search

On utilise la querystring:

```bash
$ curl -X GET https://api.domain.com/v2/items?q=toto&isGeek=false&age=18,19&sort=name,id \
    -H "Accept: application/json" \
    -H "Accept-Encoding: gzip" \
    -H "If-Modified-Since: Fri, 31 Jul 2015 20:41:30 GMT"
```

> q pour une recherche fulltext. On peut aussi se servir des filtres pour faire une recherche sur un champ particulier, exemple name=Marado*

# Cache via timestamp

On envoie le header ```If-Modified-Since``` pour valider que la ressource n'a pas été modifiée. Dans ce cas on retourne un [`304 Not Modified`](http://httpstatus.es/304).
Sinon on retourne la ressource avec le header ```Last-Modified```.

> On pourrait utiliser Etag, mais ça nécessite de maintenir un hash ou une révision (si la ressource était versionnée) par exemple, alors qu'on aura toujours/très probablement un timestamp de modification.

# Authentification :construction:

## HTTP basic

```http
Authorization: Basic cGhwOm1lZXR1cA==
```

* username:password encodé en base64
* toujours utiliser avec SSL
* maîtriser le client et le serveur (le user:password est côté client)

> Rapide à mettre en place, mais pas très secure, on doit avoir les credentials sur le client

## JWT :construction:

...

## OAuth2 :construction:

Voir la [doc](http://oauth.net/2)

> Une grande majorité des géants du web l'utilisent

# Rate limiting

Pour garder un niveau de qualité et éviter les abus, il faut mettre en place un système de limitation des appels vers l'API. Classiquement on définit une période (1h) et un nombre de requêtes maximum pour cette période.

Header | Description
------------ | -------------
X-Rate-Limit-Limit | Le nombre de requêtes possibles pendant la période
X-Rate-Limit-Remaining | Le nombre de requêtes qu'il reste pour la période
X-Rate-Limit-Reset | Le nombre de secondes qu'il reste avant de remettre les compteurs à 0

Si on dépasse la limite [`429 Too many requests`](http://httpstatus.es/429)

# CORS

Permet à une API et un client type Web App d'être sur des domaines différents sans que ça pose problème pour XMLHttpRequest.
Le client enverra une requête `OPTIONS` (preflighted request) avant chaque requête `POST`, `PUT`, `PATCH` et `DELETE` pour vérifier ce qui est autorisé.

```bash
$ curl -X OPTIONS https://api.domain.com/v2/items?q=toto&isGeek=false
&age=18,19&sort=name,id \
    -H "Access-Control-Allow-Methods: GET" \
    -H "Access-Control-Allow-Headers: X-Resource-Nested" \
    -H "Origin: http://superappjs.com"
```

En retour le serveur indiquera ce qui est permis, exemple:

```http
Access-Control-Allow-Origin: http://superappjs.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: X-Rate-Limit-Limit, X-Rate-Limit-Remaining, X-Rate-Limit-Reset, X-Total-Count, X-Page-Max-Range, X-Request-UUID, X-Resource-Nested
```

> IE<10 ne supporte pas correctement CORS, dans ce cas il faudra se tourner vers JSONP

# Documentation

C'est un point clé pour que l'API soit populaire. Il faut qu'elle soit maintenue et **facile à maintenir**!
Le mieux c'est que la documentation soit dans le code.

http://apidocjs.com permet, via des annotations dans le code, de générer la documentation complète de votre API.

Il faut mettre des exemples cURL dès que possible.
