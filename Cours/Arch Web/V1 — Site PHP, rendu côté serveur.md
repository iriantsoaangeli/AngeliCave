Première version de la série « Une page, quatre architectures ».
Objectif : afficher une liste de formations sous forme de cards.
Ici, **le serveur fabrique toute la page** : PHP interroge SQLite et renvoie un HTML complet.

## Contenu

| Fichier | Rôle |
|---|---|
| `index.php` | requête SQL + boucle PHP qui génère les cards |
| `db.php` | connexion SQLite (base créée automatiquement au premier lancement) |
| `init.sql` | création de la table `formations` et 6 lignes d'exemple |
| `style.css` | mise en forme des cards |
| `seed.php`, `seed.sql` | génération de milliers de lignes pour le test de charge |

## Lancer

Prérequis : PHP 8 avec l'extension `pdo_sqlite` (`php -m | grep sqlite`).

```bash
cd v1-php
php -S localhost:8000
```

Ouvrir http://localhost:8000

## À observer

1. Afficher le **code source** de la page (Ctrl+U) : les cards y sont déjà écrites.
   Le navigateur n'a rien calculé, il affiche.
2. Dans `index.php`, repérer les trois couches mélangées dans un même fichier :
   accès aux données (SQL), logique (boucle PHP), présentation (HTML).
3. La fonction `e()` appelle `htmlspecialchars` : on n'insère jamais une donnée brute
   dans du HTML (protection contre les injections XSS).

## Test de charge

```bash
php seed.php          # ajoute 10 000 formations
php seed.php 50000    # ou une autre quantité
php seed.php reset    # revenir aux 6 formations d'origine
```

Recharger la page et observer dans les outils développeur (F12, onglet Réseau,
cache désactivé) la taille et le temps de chargement du document HTML.

En ligne de commande :

```bash
curl -o /dev/null -s -w "%{size_download} octets, %{time_total}s\n" http://localhost:8000/
```

Conservez vos mesures : vous les comparerez avec la version suivante.

## Exercices

1. Ajouter une pagination (`LIMIT` / `OFFSET`, 20 cards par page) et mesurer à nouveau
   avec 10 000 lignes.
2. Déplacer la requête SQL dans un fichier `modele.php` qui expose une fonction
   `listerFormations()` : premier pas vers une séparation des responsabilités (MVC).
3. Ajouter une page `detail.php?id=3` qui affiche une seule formation.
