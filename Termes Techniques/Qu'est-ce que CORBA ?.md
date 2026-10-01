
# [[Une ORB , c'est quoi ?]]
## 1. CORBA est une norme, pas un logiciel

tldr: CORBA est la norme qui décrit ce qu'un ORB doit savoir faire pour permettre l'échange d'objets entre langages ; un ORB est une implémentation concrète qui respecte cette norme

**CORBA (Common Object Request Broker Architecture) n'est ni un logiciel, ni une bibliothèque : c'est une spécification.** Elle est publiée et maintenue par l'**OMG (Object Management Group)**, un consortium industriel — exactement comme le W3C publie les spécifications HTML ou HTTP sans lui-même écrire de navigateur.

Cette spécification décrit un ensemble de règles précises qu'une implémentation doit respecter pour avoir le droit de se dire « conforme CORBA » :

- le format du langage de contrat (**IDL**),
- le protocole réseau binaire (**GIOP**, transporté sur TCP/IP sous le nom **IIOP**),
- le comportement attendu du gestionnaire d'objets serveur (le **POA**),
- le format des références distantes (l'**IOR**),
- le jeu d'exceptions système standard.

**Un ORB (Object Request Broker) est une implémentation concrète de cette spécification** — une bibliothèque logicielle, écrite dans un langage donné, qui met en œuvre ces règles. omniORB (C++), JacORB (Java), TAO (C++), ORBacus sont autant d'implémentations **différentes et concurrentes de la même norme**. C'est exactement la relation entre « HTML » (la norme) et « Chrome, Firefox, Safari » (des implémentations qui rendent tous le même HTML, chacune avec son propre moteur).

**C'est cette conformité à une norme commune — et non un accord particulier entre deux bibliothèques — qui permet l'interopérabilité.** Un client Java (sur JacORB) peut appeler un serveur C++ (sur omniORB) sans qu'aucun des deux ne « sache » quelle implémentation tourne de l'autre côté : ils ne communiquent qu'au travers du protocole IIOP standardisé, que chaque ORB conforme sait parler. Remplacer omniORB par TAO côté serveur, sans toucher une ligne côté client Java, doit continuer à fonctionner — c'est la garantie que donne la norme.

## 2. Les briques essentielles définies par la norme

|Brique|Rôle|Analogie|
|---|---|---|
|**IDL**|Contrat de l'interface, neutre en langage|Un `.proto` (Protobuf) ou un schéma OpenAPI|
|**ORB**|Une implémentation de la norme ; le bus qui achemine les appels réseau|Un navigateur qui implémente le standard HTML|
|**Stub**|Code client généré depuis l'IDL : donne l'illusion d'un appel local|Un client HTTP généré depuis un schéma OpenAPI|
|**Skeleton**|Code serveur généré : reçoit l'appel réseau|Le contrôleur d'un framework web généré|
|**Servant**|La classe métier écrite par le développeur|Le `@RestController` que tu écris toi-même|
|**POA**|Composant de l'ORB qui relie les appels réseau entrants aux servants|Le routeur d'un framework web|
|**Naming Service**|Annuaire nom → référence réseau, lui-même un service CORBA standard|Un DNS interne, ou un registre de services (Eureka, Consul)|

## 3. Le flux d'un appel, dans l'absolu

```
[Client]                       [Naming Service]              [Serveur]
   | 1. lookup("MonService")         |                            |
   |--------------------------------->|                            |
   | 2. retourne l'IOR                |                            |
   |<---------------------------------|                            |
   | 3. appel de méthode (IIOP)                                     |
   |----------------------------------------------------------------->|
   |                                          4. exécute la logique métier
   | 5. résultat renvoyé (IIOP)                                     |
   |<-----------------------------------------------------------------|
```

Le client ne sait jamais dans quel langage le serveur est écrit, ni quel ORB il utilise — il parle IIOP, point final. C'est tout l'intérêt d'une architecture bâtie sur une norme réseau plutôt que sur une bibliothèque partagée entre les deux parties.

## 4. Ce que ça permet, concrètement

Deux services écrits dans deux langages différents — par exemple l'un qui écrit des fichiers, l'autre qui interroge une base de données — peuvent collaborer sans que l'un connaisse l'implémentation de l'autre : chaque langage fait ce qu'il fait de mieux, et l'IDL les fait dialoguer.

## 5. Ce qui découle de cette distinction (norme vs implémentation)

- Le choix de l'ORB (omniORB vs TAO en C++, JacORB vs le CORBA historique du JDK) est une pure question d'implémentation — maintenance, performance, disponibilité — jamais une question de compatibilité avec l'autre langage.
- Le fichier `.idl` n'appartient à aucune implémentation : c'est un artefact de la norme elle-même, qui doit produire du code compatible quel que soit le couple d'ORB utilisé.
- Une exception CORBA système (`CORBA::COMM_FAILURE`, `CORBA::TRANSIENT`...) est définie par la norme, pas par une implémentation particulière — c'est pour ça qu'on la retrouve à l'identique, avec le même nom, dans tous les langages.

---

**Pour aller plus loin dans ce même dossier :**

- `Concept_ORB.md` — ce que fait un ORB en détail (les 5 responsabilités concrètes)
- `Manuel_CORBA_Principal.md` — la mise en pratique (installation, IDL, servants, lancement)
- Spécification CORBA (OMG) : https://www.omg.org/spec/CORBA/