# TP3 : Microservices « Calculatrice » avec Spring Boot

**Matière :** Architecture MicroServices | **Filière :** MPSRCC1
**Étudiante :** Fatma Chérif | **Année universitaire :** 2026-2027

## 1. Structure du dépôt

```
tp3/
├── calcul-service/        (port 8081)
├── historique-service/    (port 8082)
├── captures/
└── README.md
```

## 2. Partie A : Questions de conception

1. **Avantage / inconvénient du calcul par requête HTTP.**
   Avantage : `calcul-service` est indépendant, on peut le développer, le redéployer ou le multiplier sans toucher à `historique-service`.
   Inconvénient : un appel réseau est plus lent qu'un appel de méthode, et `historique-service` dépend de la disponibilité de `calcul-service`.
2. **Si `calcul-service` est arrêté.**
   Continuent de fonctionner : `GET /operations`, `GET /operations/{id}` et `DELETE /operations/{id}` (elles lisent ou suppriment seulement des données en mémoire).
   Échouent : `GET /calcul/*` (service arrêté), `POST /operations` et `PUT /operations/{id}` (elles ont besoin d'un calcul).
3. **Même port impossible.** Un port ne peut être écouté que par un seul processus à la fois. Le second démarrage échoue avec « Port already in use ».
4. **Code de statut.** **503 Service Unavailable** : le client n'a rien fait de faux et le serveur n'a pas de bug, mais il ne peut pas joindre un service dont il dépend.

## 3. Partie B : Création de `calcul-service`

Projet généré avec Spring Initializr : Maven, Java [17 ou 21], Spring Boot 4.0.9, Group `com.exemple`, Artifact `calcul-service`, Package `com.exemple.calcul`, Packaging Jar, dépendance Spring Web.

Structure :
```
com.exemple.calcul
├── CalculServiceApplication
├── controller/CalculController
├── service/Calculatrice
└── model/Resultat
```

## 4. Partie C : Premier microservice

### Lancement de calcul-service

![Console calcul-service](captures/console-8081.png)

Le Tomcat embarqué démarre sur le port 8081.

### Tableau de correspondance Spring / JAX-RS

| Spring (TP3) | JAX-RS (TP2) | Rôle dans cet exemple |
|---|---|---|
| `@RestController` | (pas d'équivalent direct) | Déclare la classe comme contrôleur REST : la valeur retournée devient le corps de la réponse |
| `@RequestMapping("/calcul")` | `@Path("/calcul")` sur la classe | URI de base de toutes les méthodes du contrôleur |
| `@GetMapping("/somme")` | `@GET` + `@Path("/somme")` | Méthode qui répond à `GET /calcul/somme` |
| `produces = MediaType.TEXT_PLAIN_VALUE` | `@Produces(MediaType.TEXT_PLAIN)` | La réponse est du texte brut |
| `@RequestParam("a") int a` | `@QueryParam("a") int a` | Lit le paramètre `a` dans `?a=2&b=3` |
| `ResponseEntity.ok(...)` | `Response.ok(...).build()` | Construit une réponse de statut 200 |

### Origine des parties de l'URL `http://localhost:8081/calcul/somme?a=2&b=3`

| Partie | Origine |
|---|---|
| `localhost:8081` | Machine locale et port fixé par `server.port` dans `application.properties` |
| `/calcul` | `@RequestMapping("/calcul")` de la classe |
| `/somme` | `@GetMapping("/somme")` de la méthode |
| `?a=2&b=3` | Paramètres de requête, lus par `@RequestParam` |

**Question C.1.** Ont disparu : `RestWebApp` (nom de l'application `.war` déployée dans Tomcat) et `/api` (`@ApplicationPath`). Un microservice est une application autonome avec son propre Tomcat embarqué : il n'y a plus de `.war` à déployer, donc plus de nom d'application, et plus de classe d'activation JAX-RS.

### Tests

Test dans le navigateur : `http://localhost:8081/calcul/somme?a=2&b=3`

![somme dans Chrome](captures/somme-chrome.png)

**Question C.2.** Un `POST` sur `/calcul/somme` renvoie le statut **[405 Method Not Allowed : à vérifier sur ta capture]**, car la méthode n'est mappée que sur `GET`. [Au TP2, le statut était **[à comparer avec ton TP2]**.]

![POST dans Postman](captures/post-405.png)

## 5. Partie D : Calculatrice complète

| Requête | Statut | Corps |
|---|---|---|
| `GET /calcul/soustraction?a=10&b=4` | 200 | `6` |
| `GET /calcul/produit?a=6&b=7` | 200 | `42` |
| `GET /calcul/division?a=7&b=2` | 200 | `3.5` |
| `GET /calcul/division?a=7&b=0` | 400 | `Erreur : division par 0 impossible` |
| `GET /calcul/produit/6/7` | 200 | `42` |

Division par zéro :

![division par 0](captures/division-400.png)

### Observations (D.3)

Paramètre `b` absent, puis paramètre `a` non entier :

![paramètres invalides](captures/parametres-400.png)

**Question D.1.** Spring renvoie **400 Bad Request** quand un paramètre est absent ou n'est pas un entier : la méthode n'est pas exécutée. Au TP2, JAX-RS donnait 0 à un entier absent et calculait quand même. L'attitude de Spring est la plus conforme à REST : une requête incomplète ou invalide est une erreur du client, et on le lui signale au lieu de produire un résultat faux.

**Question D.2.** Le JSON d'erreur contient les champs : **[timestamp, status, error, path : à vérifier sur ta capture]**.

**Question D.3.**

| Spring | JAX-RS |
|---|---|
| `@RequestParam` | `@QueryParam` |
| `@PathVariable` | `@PathParam` |

Pour lire l'identifiant dans `/operations/{id}`, on utilise **`@PathVariable`**, car `id` fait partie du chemin et non d'une chaîne de requête.

## 6. Partie E : Échanger des données en JSON

`GET /calcul/somme/detail?a=2&b=3` :

![JSON détaillé](captures/json-detail.png)

### En-tête Accept

Requête avec `Accept: application/json` (200), puis avec `Accept: application/xml` (406) :

![Accept json et xml](captures/accept-406.png)

**Question E.1.** Oui, les deux JSON sont équivalents pour un client : un objet JSON est un ensemble de paires clé/valeur non ordonnées, seul le contenu compte. Cette fois, la conversion a été faite par **Jackson**, inclus dans `spring-boot-starter-webmvc` (au TP2, c'était JSON-B).

**Question E.2.** Spring compare l'en-tête `Accept` du client avec les formats que l'application sait produire. Jackson ne produit que du JSON et aucun convertisseur XML n'est présent, donc aucun format ne convient : Spring répond **406 Not Acceptable**, même si le code ne mentionne aucun format.

### Exercice E.4 : `divisionDetail`

| Requête | Statut | Corps |
|---|---|---|
| `GET /calcul/division/detail?a=6&b=3` | 200 | `{"operation":"division","a":6,"b":3,"resultat":2.0}` |
| `GET /calcul/division/detail?a=6&b=0` | 400 | (vide) |
