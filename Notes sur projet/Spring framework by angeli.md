
**DOCUMENTATION GENEREE PAR IA**
**AZA MBA MATOKY BE LOATRA**

## 1) Objectif du projet
Ce projet est un mini framework MVC basé sur des servlets Java.

L’objectif est de reproduire le fonctionnement d’un système simple de type Spring :
- détection automatique des contrôleurs,
- annotations pour les routes,
- mapping URL -> méthode,
- traitement des requêtes GET/POST,
- retour de vue ou de réponse directe.

---

## 2) Structure du projet

```text
spring-framework-by-angeli/
└── src/main/java/angeli/sprint/
    ├── annotation/
    │   ├── Controller.java
    │   ├── URL.java
    │   └── WebAPI.java
    ├── model/
    │   └── ModelAndView.java
    ├── servlet/
    │   ├── FrontControllerServlet.java
    │   ├── PageWriter.java
    │   └── listener/
    │       └── ContextListener.java
    ├── url/
    │   └── URLMethod.java
    └── utils/
        ├── URLHandler.java
        ├── URLParser.java
        └── reflect/
            ├── ClassPathScanner.java
            └── Mapper.java
```

---

## 3) Rôle des classes principales

### `@Controller`
Marque une classe comme contrôleur.

```java
@Controller
public class HomeController {
}
```

### `@URL`
Associe une méthode à une URL et à une méthode HTTP.

```java
@URL(value = "/accueil", method = "GET")
public ModelAndView accueil() {
    return new ModelAndView("home");
}
```

### `ModelAndView`
Objet qui transporte :
- le nom de la vue,
- les données à injecter dans la page.

```java
ModelAndView mv = new ModelAndView("home");
mv.setAttribute("title", "Bienvenue");
```

### `ContextListener`
Se lance au démarrage. Il sert à :
- scanner les classes annotées,
- récupérer les méthodes annotées @URL,
- créer les maps GET/POST,
- mettre les infos dans le contexte servlet.

### `FrontControllerServlet`
Point d’entrée unique pour toutes les requêtes.
Il :
- lit l’URL demandée,
- vérifie la bonne map GET/POST,
- exécute la méthode correspondante,
- affiche la vue ou écrit la réponse directement.

### `URLParser`
Extrait l’URI depuis la requête HTTP pour comparer avec les routes enregistrées.

### `URLHandler`
Vérifie si :
- l’URL existe,
- elle retourne une vue,
- elle retourne un objet JSON/texte.

### `ClassPathScanner`
Utilise ClassGraph pour retrouver automatiquement les classes et méthodes annotées.

### `Mapper`
Crée la correspondance :
```text
URL -> méthode Java
```

---

## 4) Cycle d’exécution d’une requête

1. Le contexte servlet démarre.
2. `ContextListener` scan les contrôleurs et les routes.
3. `Mapper` stocke les méthodes dans des maps GET/POST.
4. Une requête arrive dans `FrontControllerServlet`.
5. `URLParser` extrait l’URI.
6. `URLHandler` vérifie si l’URL existe.
7. La méthode associée est invoquée.
8. Si la méthode retourne `ModelAndView`, la vue est affichée.
9. Sinon, la réponse est écrite directement comme JSON/texte.

---

## 5) Retour possible d’une méthode

### Vue JSP/HTML
```java
@URL(value = "/home", method = "GET")
public ModelAndView home() {
    ModelAndView mv = new ModelAndView("home");
    mv.setAttribute("nom", "Angeli");
    return mv;
}
```

### Réponse directe
```java
@WebAPI(contentType = "application/json")
@URL(value = "/api/test", method = "GET")
public String test() {
    return "{\"ok\": true}";
}
```

---

## 6) Ajouter une route
Pour créer une route, il faut :
1. créer une classe annotée `@Controller`,
2. ajouter une méthode annotée `@URL`,
3. choisir un chemin et une méthode HTTP,
4. retourner `ModelAndView` ou un objet/String.

Exemple complet :

```java
@Controller
public class HomeController {

    @URL(value = "/", method = "GET")
    public ModelAndView index() {
        ModelAndView mv = new ModelAndView("index");
        mv.setAttribute("title", "Bienvenue");
        return mv;
    }

    @URL(value = "/api/users", method = "GET")
    public String users() {
        return "[{\"id\":1,\"name\":\"Angeli\"}]";
    }
}
```

---

## 7) Règles à retenir
- Une classe contrôleur doit avoir `@Controller`.
- Une méthode accessible par URL doit avoir `@URL`.
- Le point d’entrée est `FrontControllerServlet`.
- Le démarrage est piloté par `ContextListener`.
- `ModelAndView` sert pour les pages web.
- `@WebAPI` sert pour les réponse JSON/texte.
- Le mapping principal se fait via `ClassPathScanner` + `Mapper` + `URLHandler`.

C’est la base minimale pour reprendre le framework et l’étendre sans perdre le fil.