
Le nom complet — **Object Request Broker** — le dit : l'ORB n'est pas un détail d'implémentation, c'est le cœur du système. Sans lui, le code généré depuis l'IDL (stubs et skeletons) ne peut littéralement pas s'exécuter : c'est l'ORB qui les fait tourner.

IDL : interface Description / Definition Language 
## 1. L'analogie : un standard téléphonique

Imagine un standard téléphonique d'entreprise. Tu ne composes pas directement le poste interne de ton correspondant : tu appelles le standard, qui traduit ton appel, trouve la bonne personne (même si elle a changé de bureau) et relie la ligne. L'ORB joue exactement ce rôle entre deux objets qui veulent se parler à travers le réseau, potentiellement dans deux langages différents.

## 2. Les cinq responsabilités concrètes de l'ORB

### 2.1 Le marshalling / unmarshalling

Quand un client appelle une méthode distante, il n'y a pas de mémoire partagée avec le processus serveur. L'ORB doit :

- **Marshalling** (côté appelant) : transformer l'appel de méthode et ses paramètres en une suite d'octets standardisée (le protocole GIOP, transporté sur IIOP)
- **Unmarshalling** (côté receveur) : reconstruire l'appel de méthode à partir de ces octets, dans le langage cible

C'est ce qui permet à un type d'un langage et un type d'un autre langage de « se comprendre » : ce ne sont jamais les types natifs qui voyagent sur le réseau, mais une représentation binaire neutre définie par la spécification CORBA, que chaque ORB sait encoder et décoder.

### 2.2 La résolution de références (IOR)

Une référence CORBA (un objet distant que l'on manipule) est en réalité encodée sous forme d'une chaîne appelée **IOR** (Interoperable Object Reference) — une sorte d'adresse contenant l'IP, le port, et un identifiant d'objet. L'ORB sait décoder une IOR pour ouvrir la connexion réseau correspondante, de façon totalement transparente pour le code applicatif : on manipule un objet de référence typé, jamais l'IOR brute.

### 2.3 L'activation des servants (via le POA)

Côté serveur, quand un appel réseau arrive, quelqu'un doit décider quelle instance d'objet en mémoire doit le traiter. C'est le rôle du **POA** (Portable Object Adapter), un sous-composant de l'ORB : il fait le pont entre l'identifiant reçu sur le réseau et le servant réellement instancié.

### 2.4 La gestion du cycle de vie des connexions

L'ORB maintient les connexions TCP sous-jacentes, gère leur réutilisation (pour ne pas ouvrir une nouvelle connexion à chaque appel), et les threads qui traitent les appels entrants. C'est directement ce comportement qui rend une ressource partagée côté serveur (par exemple une connexion base de données) sensible aux problèmes de concurrence : plusieurs appels peuvent arriver en parallèle sur des threads différents du pool de l'ORB.

### 2.5 Le transport des exceptions système

Si le réseau tombe, si le serveur n'existe pas, si le Naming Service est injoignable : l'ORB lève des exceptions standardisées (`CORBA::COMM_FAILURE`, `CORBA::TRANSIENT`, `CORBA::OBJECT_NOT_EXIST`…) que le code applicatif peut attraper de façon identique quel que soit le langage, malgré des causes réseau très différentes derrière.

## 3. Pourquoi ça ne peut pas être « juste une bibliothèque optionnelle »

Le tout premier appel de n'importe quel programme CORBA, quel que soit le langage :

```cpp
CORBA::ORB_var orb = CORBA::ORB_init(argc, argv);   // C++
```

```java
ORB orb = ORB.init(args, null);                     // Java
```

Tout — le POA, le Naming Service, les stubs générés — est obtenu à partir de cet objet `orb` (`orb->resolve_initial_references(...)`). Il n'existe pas de chemin pour faire du CORBA sans lui : ce n'est pas une option de configuration, c'est le **point d'entrée obligatoire** de toute l'architecture.

## 4. Ce qui, en revanche, reste un choix

Ce qui n'est pas imposé par la spécification OMG, c'est **quelle implémentation d'ORB** on utilise. omniORB, TAO, JacORB, ORBacus implémentent tous la même spécification et savent se parler entre eux via IIOP — c'est justement ce qui permet à un client sur une implémentation d'appeler un serveur sur une autre implémentation sans qu'aucun des deux ne « sache » que l'autre existe en tant qu'implémentation (voir `Concept_CORBA.md`, section 1).

---

**Pour aller plus loin dans ce même dossier :**

- `Concept_CORBA.md` — la norme elle-même et les briques essentielles
- `Manuel_CORBA_Principal.md` — la mise en pratique (installation, IDL, servants, lancement)