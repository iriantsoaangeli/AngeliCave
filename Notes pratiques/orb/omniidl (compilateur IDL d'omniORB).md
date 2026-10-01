# [[Une ORB , c'est quoi ?]]
## 1. Ce que fait omniidl

`omniidl` lit un fichier `.idl` (purement déclaratif : pas de `if`, pas de boucle, pas de corps de fonction, uniquement des signatures — voir `Concept_CORBA.md`) et **génère du code source C++ ordinaire** — pas un exécutable, juste des fichiers `.cc`/`.hh` que l'on compile ensuite soi-même avec `g++`.

```
Demo.idl  ──(omniidl)──►  fichiers C++ (Demo.hh, DemoSK.cc, ...)
```

`omniidl` n'est **pas un outil séparé d'omniORB : c'est le compilateur IDL fourni avec omniORB lui-même**. Il n'existe pas d'alternative indépendante d'omniORB pour cette raison — une fois qu'on choisit omniORB comme ORB C++, on utilise nécessairement omniidl pour générer du code compatible avec ses classes runtime (`CORBA::ORB_var`, `PortableServer::POA`, etc.). Chaque implémentation d'ORB impose son propre compilateur IDL, généré pour produire du code compatible avec ses propres classes runtime (`tao_idl` pour TAO, le compilateur JacORB pour Java — voir `Guide_JacORB.md`).

## 2. Installation

```bash
sudo apt update
sudo apt install -y omniorb omniorb-nameserver libomniorb4-dev omniidl
omniidl -h        # doit afficher l'aide du compilateur
```

**Piège fréquent** : si `omniidl` est introuvable, c'est généralement `libomniorb4-dev` (paquet de dev, séparé du paquet runtime) qui manque sur certaines distributions.

## 3. Invocation

```bash
mkdir -p cpp/generated
omniidl -bcxx -Cgenerated idl/Demo.idl
```

Décomposons la commande :

- `omniidl` : le compilateur
- `-bcxx` : « backend C++ » — dans quel langage générer (il existe aussi un backend Python, par exemple, pour d'autres usages)
- `-Cgenerated` : place les fichiers générés dans le dossier `generated/`
- `idl/Demo.idl` : le fichier source

## 4. Ce que ça produit

Dans `cpp/generated/` :

|Fichier|Contenu|
|---|---|
|`Demo.hh`|Déclarations C++ : classes, structures (ex. `Demo::Produit`), les séquences (ex. `Demo::ProduitSeq`), et les classes `POA_Demo::*` à hériter pour écrire un servant|
|`DemoSK.cc`|Le **skeleton** : code qui reçoit l'appel réseau et le redirige vers le servant applicatif|
|_(parfois `Demo.cc` selon la version)_|Vérifie toujours avec `ls cpp/generated/` ce qui a réellement été produit — ça varie selon la version d'omniORB|

On inclut ensuite `Demo.hh` dans son code (`#include "generated/Demo.hh"`).

Pour chaque interface et chaque struct déclarés dans l'IDL, le compilateur génère précisément :

1. **Le mapping des types** : un `struct` IDL devient une vraie classe C++ avec les bons types natifs (`Demo::Produit`)
2. **Le stub client** : le code qui donne l'illusion d'appeler une méthode locale, alors qu'il sérialise l'appel et l'envoie sur le réseau
3. **Le skeleton serveur** : le code qui reçoit l'appel réseau et le redirige vers l'implémentation applicative
4. **Les classes utilitaires** (`_Helper`, `_Holder`, `_var`, `_ptr`) : des outils pour manipuler proprement les références CORBA et les types complexes, avec la bonne gestion mémoire (voir `Guide_Cpp_Pratique.md` pour la gestion mémoire C++ en général)

## 5. Ce qu'on doit retenir en une phrase

> `omniidl` n'est qu'un générateur de code de plomberie répétitif. Le seul artefact écrit à la main, c'est le servant (la classe qui étend `POA_Demo::MonInterface`) — tout le reste est mécaniquement dérivé de l'IDL.

## 6. Pièges spécifiques à omniidl

|Piège|Symptôme|Solution|
|---|---|---|
|`omniidl` absent|Commande introuvable|Installer `libomniorb4-dev`|
|IDL modifié sans régénération|Erreurs de compilation, ou pire, bug silencieux si les signatures se ressemblent|Toujours relancer `omniidl` après toute modification de l'IDL, puis recompiler|
|Édition manuelle du code généré|Les modifications disparaissent à la prochaine régénération, comportement incohérent|Ne jamais éditer `Demo.hh`/`DemoSK.cc` à la main ; corriger l'IDL ou le servant|

## 7. Cheat-sheet

```bash
omniidl -h                                  # aide
omniidl -bcxx -Cgenerated idl/Demo.idl       # génération standard
ls cpp/generated/                            # vérifier ce qui a été produit
```

---

**Voir aussi :**

- `Guide_JacORB.md` — l'équivalent côté Java
- `Manuel_CORBA_Principal.md` — où cette commande s'insère dans le flux complet du projet