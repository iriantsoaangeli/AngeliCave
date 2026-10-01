## Table des matières
---

[[Qu'est-ce que CORBA ?]]
[[Une ORB , c'est quoi ?]]

---

1. Installation de l'environnement
2. L'IDL partagé — l'exemple : un annuaire de personnes
3. Plan des classes : métier vs obligatoires (indépendant du client/serveur)
4. Le Naming Service — le principe, commun aux deux scénarios
5. Scénario A — Serveur C++, Client Java
6. Scénario B — Serveur Java, Client Java
7. Table des pièges
8. Cheat-sheet

---

## 1. Installation de l'environnement (Linux Debian/Ubuntu)

```bash
# omniORB (C++) — nécessaire uniquement si tu fais le Scénario A (serveur C++)
sudo apt update
sudo apt install -y omniorb omniorb-nameserver libomniorb4-dev omniidl
omniidl -h
which omniNames
```

```bash
# JacORB (Java) — nécessaire dans les deux scénarios (client Java toujours présent,
# serveur Java dans le scénario B)
cd /opt
sudo wget https://www.jacorb.org/releases/3.9/jacorb-3.9-binary.zip
sudo unzip jacorb-3.9-binary.zip -d jacorb
```

Dans `~/.bashrc` :

```bash
export JACORB_HOME=/opt/jacorb
export PATH=$JACORB_HOME/bin:$PATH
export CLASSPATH=$JACORB_HOME/lib/jacorb-3.9.jar:$JACORB_HOME/lib/slf4j-api-1.7.14.jar:$JACORB_HOME/lib/idl.jar
```

> Le Naming Service (`omniNames`) reste fourni par omniORB même dans le Scénario B (Java pur) — voir section 4 : ce n'est pas parce que les deux bouts sont en Java qu'il faut un annuaire différent.

---

## 2. L'IDL partagé — l'exemple : un annuaire de personnes

Le fichier le plus important du projet : tous les langages en jeu génèrent leur code à partir de lui — **littéralement le même fichier** (voir `Concept_CORBA.md` pour pourquoi c'est cette identité qui garantit l'interopérabilité).

**L'IDL, c'est là où on décrit le contrat (quels types, quelles méthodes, quelles exceptions) que les deux langages vont respecter — et c'est à partir de ce même contrat que chaque compilateur génère ses propres classes, natives à son langage. Les classes elles-mêmes ne sont jamais partagées ; seul le contrat qui les décrit l'est.**

`annuaire/idl/Annuaire.idl` :

```idl
module Annuaire {

  exception OperationFailed {
    string reason;
  };

  struct Personne {
    string nom;
    string date_naissance;   // format "AAAA-MM-JJ"
  };

  typedef sequence<Personne> PersonneSeq;

  interface AnnuaireService {
    PersonneSeq getToutesLesPersonnes()
      raises (OperationFailed);

    void ajouterPersonne(in string nom, in string date_naissance)
      raises (OperationFailed);
  };

};
```

|Élément IDL|Rôle|Équivalent Java|
|---|---|---|
|`module Annuaire`|Namespace, évite les collisions|`package`|
|`interface AnnuaireService`|Le service distant|Une interface Java|
|`struct Personne`|Le type de données échangé|Un `record`/POJO simple|
|`exception OperationFailed`|Erreur applicative typée, transportable réseau|Une classe d'exception|
|`sequence<Personne>`|Une liste de taille variable|`List<Personne>` / `Personne[]`|
|`raises (...)`|Les exceptions qu'une méthode peut lever|`throws`|

**Piège — régénération** : après toute modification de `Annuaire.idl`, régénérer **des deux côtés impliqués** et recompiler. Aucun compilateur IDL ne prévient si un servant implémente une ancienne version du contrat.

**Piège — édition du code généré** : ne jamais modifier à la main le code produit par `omniidl` ou par le compilateur JacORB.

---

## 3. Plan des classes : métier vs obligatoires (indépendant du client/serveur)

Cette section répond à une seule question — **« cette classe, elle fait quoi ? »** — sans jamais se demander qui l'utilise (client ou serveur), ni dans quel scénario. Cette distinction-là vient seulement en section 5 et 6.

### 3.1 Classes métier — leur contenu dépend de TON IDL

Ce sont les classes qui changeraient si tu changeais de domaine (annuaire de personnes → catalogue produits → réservations...). Générées à partir de l'IDL, mais leur **forme** reflète directement ce que toi tu as déclaré.

|Classe|Générée depuis|Ce qu'elle fait|
|---|---|---|
|`Personne` (C++ et Java, deux classes natives séparées)|`struct Personne`|Le DTO : porte les données échangées (nom, date de naissance). Aucune logique, juste des champs.|
|`PersonneSeq` (C++) / `Personne[]` (Java)|`sequence<Personne>`|Une liste de `Personne` — ce que retourne `getToutesLesPersonnes()`.|
|`AnnuaireService` (l'interface pure, générée dans les deux langages)|`interface AnnuaireService`|Le contrat : la liste des méthodes appelables à distance, avec leurs types. Ni client ni serveur ne l'implémentent directement — c'est une référence, un type.|
|`OperationFailed`|`exception OperationFailed`|L'erreur applicative typée, transportable jusqu'à l'appelant, quel que soit son langage.|

```cpp
// C++
Annuaire::Personne p;
p.nom = CORBA::string_dup("Ada Lovelace");
p.date_naissance = CORBA::string_dup("1815-12-10");
```

```java
// Java
Personne p = new Personne();
p.nom = "Ada Lovelace";
p.date_naissance = "1815-12-10";
```

**Ton servant** (la seule classe que tu écris entièrement toi-même, pas générée) appartient aussi à cette famille métier au sens large — c'est elle qui contient la vraie logique (stocker une personne, la retourner). Son écriture concrète dépend du scénario (sections 5 et 6).

### 3.2 Classes obligatoires — leur mécanisme est toujours le même, quel que soit ton IDL

Ce sont les classes générées **pour n'importe quel IDL**, sans exception — leur rôle ne change jamais, seul leur nom (préfixé par ton interface) varie.

**`_var` (C++ uniquement)** — pointeur intelligent spécifique à CORBA, gère sa propre mémoire par comptage de références (voir `Guide_Cpp_Pratique.md` §1.2 pour la gestion mémoire C++ en général).

```cpp
Annuaire::PersonneSeq_var seq = new Annuaire::PersonneSeq();
// ...
return seq._retn();   // transfère la propriété mémoire à l'appelant, sans copie
```

N'existe pas en Java : le GC s'en charge.

**`_Helper`** — classe utilitaire statique, une par interface, pour le **narrowing** : convertir une référence CORBA générique en type concret.

```java
AnnuaireService service = AnnuaireServiceHelper.narrow(nc.resolve_str("AnnuaireService"));
if (service == null) { /* échec silencieux si le type ne correspond pas */ }
```

```cpp
Annuaire::AnnuaireService_var service = Annuaire::AnnuaireService::_narrow(obj);
if (CORBA::is_nil(service)) { /* idem */ }
```

Côté C++, pas de classe `_Helper` séparée — la méthode `_narrow` est directement sur la classe du service.

**`_Holder` (Java uniquement)** — simule les paramètres `out`/`inout` de l'IDL, que Java ne sait pas exprimer nativement.

```java
// Si l'IDL avait : void chercher(in string nom, out Personne resultat);
PersonneHolder resultat = new PersonneHolder();
service.chercher("Ada Lovelace", resultat);
Personne p = resultat.value;
```

Non utilisé dans `AnnuaireService` (aucun `out`/`inout` dans notre IDL) — mentionné pour être complet.

**Le skeleton** — reçoit l'appel réseau déjà décodé et le redirige vers le servant. En C++, un fichier séparé (`AnnuaireSK.cc`) ; en Java, intégré dans la classe `*POA`. **Jamais écrit ni lu.**

**La classe `*POA`** — la seule classe générée que le code du développeur touche directement : une classe de base que le servant doit étendre pour se connecter à l'ORB et au POA (voir `Concept_ORB.md` §2.3).

```cpp
class AnnuaireService_impl : public POA_Annuaire::AnnuaireService { /* ... */ };
```

```java
public class AnnuaireServiceImpl extends AnnuaireServicePOA { /* ... */ }
```

### 3.3 Récapitulatif

|Classe|Type|Change avec l'IDL ?|Qui l'écrit ?|
|---|---|---|---|
|`Personne`, `PersonneSeq`, `AnnuaireService` (interface), `OperationFailed`|Métier|Oui — reflète ton domaine|Générée, jamais éditée|
|`_var` (C++)|Obligatoire|Non — même mécanisme partout|Générée, jamais éditée|
|`_Helper`|Obligatoire|Non|Générée, jamais éditée|
|`_Holder` (Java)|Obligatoire|Non|Générée, jamais éditée|
|Skeleton|Obligatoire|Non|Générée, jamais lue|
|`*POA`|Obligatoire|Non|Générée — **tu en hérites**, seul contact direct|
|**Ton servant**|Métier (logique)|Oui — toute la logique t'appartient|**Toi, entièrement**|

Cette carte reste valable dans les deux scénarios qui suivent — ce qui change entre les deux, ce n'est **pas** cette liste de classes, mais **qui héberge quoi**.

---

## 4. Le Naming Service — le principe, commun aux deux scénarios

Peu importe le scénario, un seul et même annuaire fait le lien entre un nom (`"AnnuaireService"`) et une référence réseau — voir `Concept_CORBA.md` pour le rôle du Naming Service, et les échanges précédents sur l'IOR.

```bash
mkdir -p ~/corba_ns && cd ~/corba_ns
omniNames -start 2809
```

**Important : ça n'active rien d'autre.** L'annuaire démarre vide ; chaque serveur (C++ ou Java, peu importe) doit ensuite s'enregistrer lui-même en se lançant. Si tu listes l'annuaire avant qu'un serveur ne se soit lancé, il est normal qu'il soit vide.

```bash
nameclt list   # diagnostic universel, quel que soit le scénario
```

**Piège** : relancer `omniNames -start` sans nettoyer d'anciens logs (`omninames-<hostname>.log`) peut le faire repartir dans un état incohérent — `rm -f omninames-*.log` avant chaque relance propre.

Les deux sections suivantes montrent comment **chaque** serveur (C++ ou Java) s'y connecte concrètement.

---

## 5. Scénario A — Serveur C++, Client Java

### 5.1 Générer le code

```bash
mkdir -p cpp/generated
omniidl -bcxx -Cgenerated idl/Annuaire.idl

mkdir -p java/generated
java -cp $CLASSPATH org.jacorb.idl.parser -d java/generated idl/Annuaire.idl
```

### 5.2 Le serveur C++ (le servant)

```cpp
// AnnuaireService_impl.hh
#ifndef ANNUAIRESERVICE_IMPL_HH
#define ANNUAIRESERVICE_IMPL_HH
#include "generated/Annuaire.hh"
#include <vector>
#include <mutex>

class AnnuaireService_impl : public POA_Annuaire::AnnuaireService {
public:
  Annuaire::PersonneSeq* getToutesLesPersonnes() override;
  void ajouterPersonne(const char* nom, const char* date_naissance) override;
private:
  std::vector<Annuaire::Personne> personnes_;
  std::mutex mutex_;
};
#endif
```

```cpp
// AnnuaireService_impl.cc
#include "AnnuaireService_impl.hh"

Annuaire::PersonneSeq* AnnuaireService_impl::getToutesLesPersonnes() {
  std::lock_guard<std::mutex> lock(mutex_);
  Annuaire::PersonneSeq_var seq = new Annuaire::PersonneSeq();
  seq->length(personnes_.size());
  for (size_t i = 0; i < personnes_.size(); ++i) {
    seq[i].nom = CORBA::string_dup(personnes_[i].nom);
    seq[i].date_naissance = CORBA::string_dup(personnes_[i].date_naissance);
  }
  return seq._retn();
}

void AnnuaireService_impl::ajouterPersonne(const char* nom, const char* date_naissance) {
  std::lock_guard<std::mutex> lock(mutex_);
  Annuaire::Personne p;
  p.nom = CORBA::string_dup(nom);
  p.date_naissance = CORBA::string_dup(date_naissance);
  personnes_.push_back(p);
}
```

```cpp
// server_main.cc
#include "AnnuaireService_impl.hh"
#include <omniORB4/CORBA.h>
#include <iostream>

int main(int argc, char** argv) {
  try {
    CORBA::ORB_var orb = CORBA::ORB_init(argc, argv);
    CORBA::Object_var poaObj = orb->resolve_initial_references("RootPOA");
    PortableServer::POA_var poa = PortableServer::POA::_narrow(poaObj);
    PortableServer::POAManager_var pman = poa->the_POAManager();

    AnnuaireService_impl* servant = new AnnuaireService_impl();
    poa->activate_object(servant);
    CORBA::Object_var ref = servant->_this();

    CORBA::Object_var nsObj = orb->resolve_initial_references("NameService");
    CosNaming::NamingContextExt_var nc = CosNaming::NamingContextExt::_narrow(nsObj);
    CosNaming::Name name;
    name.length(1);
    name[0].id = CORBA::string_dup("AnnuaireService");
    nc->rebind(name, ref);

    pman->activate();
    std::cout << "Serveur AnnuaireService (C++) pret." << std::endl;
    orb->run();
  } catch (CORBA::Exception& ex) {
    std::cerr << "Erreur CORBA : " << ex._name() << std::endl;
    return 1;
  }
  return 0;
}
```

### 5.3 Le client Java

```java
public class ClientMain {
    public static void main(String[] args) {
        try {
            ORB orb = ORB.init(args, null);
            NamingContextExt nc = NamingContextExtHelper.narrow(
                orb.resolve_initial_references("NameService"));

            org.omg.CORBA.Object obj = nc.resolve_str("AnnuaireService");
            AnnuaireService service = AnnuaireServiceHelper.narrow(obj);
            if (service == null) {
                System.err.println("Impossible de resoudre AnnuaireService");
                return;
            }

            service.ajouterPersonne("Ada Lovelace", "1815-12-10");
            Personne[] personnes = service.getToutesLesPersonnes();
            for (Personne p : personnes) {
                System.out.println(p.nom + " - né(e) le " + p.date_naissance);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### 5.4 Compilation

```makefile
# cpp/Makefile
CXX = g++
CXXFLAGS = -std=c++17 -I.
LDFLAGS = -lomniORB4 -lomnithread -lomniDynamic4 -lCOS4

GEN_SRCS = generated/AnnuaireSK.cc
SERVER_SRCS = server_main.cc AnnuaireService_impl.cc $(GEN_SRCS)

all: annuaire_server
annuaire_server: $(SERVER_SRCS)
	$(CXX) $(CXXFLAGS) -o $@ $^ $(LDFLAGS)
```

```bash
make -C cpp
cd java && javac -cp generated:$CLASSPATH -d classes generated/Annuaire/*.java src/*.java
```

### 5.5 Lancement

```bash
# Terminal 1
omniNames -start 2809

# Terminal 2 — le serveur C++, s'enregistre au démarrage
cd cpp && ./annuaire_server -ORBInitRef NameService=corbaname::localhost:2809

# Terminal 3 — le client Java, a besoin de java/jacorb.properties (Guide_JacORB.md §5)
cd java && java -Djacorb.config.dir=. -cp .:generated:$CLASSPATH ClientMain
```

**Piège spécifique à ce scénario** : `_narrow` (C++) et `narrow` (Java) sont deux méthodes différentes, sur des classes différentes — erreur de copier-coller fréquente en passant d'un langage à l'autre.

---

## 6. Scénario B — Serveur Java, Client Java

Même IDL, même Naming Service — mais cette fois, **les deux bouts sont en Java**. Le point pédagogique important : même si tout est dans le même langage, **rien ne change dans la mécanique CORBA elle-même** — ce sont toujours deux processus séparés (deux JVM différentes), qui communiquent par IIOP en passant par le Naming Service, exactement comme dans le scénario A. Le langage commun ne crée aucun raccourci.

### 6.1 Générer le code

```bash
mkdir -p java/generated
java -cp $CLASSPATH org.jacorb.idl.parser -d java/generated idl/Annuaire.idl
```

Une seule génération suffit ici — les deux processus (serveur et client) partagent le même code généré, puisqu'ils sont dans le même langage.

### 6.2 Le serveur Java (le servant)

```java
public class AnnuaireServiceImpl extends AnnuaireServicePOA {
    private final List<Personne> personnes = new ArrayList<>();

    @Override
    public synchronized Personne[] getToutesLesPersonnes() throws OperationFailed {
        return personnes.toArray(new Personne[0]);
    }

    @Override
    public synchronized void ajouterPersonne(String nom, String dateNaissance) throws OperationFailed {
        Personne p = new Personne();
        p.nom = nom;
        p.date_naissance = dateNaissance;
        personnes.add(p);
    }
}
```

Remarque, en miroir du servant C++ (section 5.2) : ici pas de `CORBA::string_dup`, pas de type `_var`, pas de `._retn()` — ce sont des mécanismes propres à la gestion mémoire C++ (voir `Guide_Cpp_Pratique.md` §1.2), absents en Java où le GC s'en occupe. `synchronized` joue le même rôle protecteur que le `std::mutex` du scénario A.

```java
public class ServerMain {
    public static void main(String[] args) {
        try {
            ORB orb = ORB.init(args, null);
            POA rootPOA = POAHelper.narrow(orb.resolve_initial_references("RootPOA"));
            rootPOA.the_POAManager().activate();

            AnnuaireServiceImpl servant = new AnnuaireServiceImpl();
            org.omg.CORBA.Object ref = rootPOA.servant_to_reference(servant);

            NamingContextExt nc = NamingContextExtHelper.narrow(
                orb.resolve_initial_references("NameService"));
            nc.rebind(nc.to_name("AnnuaireService"), ref);

            System.out.println("Serveur AnnuaireService (Java) pret.");
            orb.run();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### 6.3 Le client Java

**Identique mot pour mot à celui du scénario A** (section 5.3) — c'est le point à retenir : le client ne sait pas, et n'a pas besoin de savoir, si le serveur en face est écrit en C++ ou en Java. Il consulte le Naming Service, fait un `narrow`, et appelle — le langage du serveur n'apparaît nulle part dans son code.

### 6.4 Compilation

```bash
cd java
mkdir -p classes
javac -cp generated:$CLASSPATH -d classes generated/Annuaire/*.java src/*.java
```

Un seul dossier `classes/`, un seul `CLASSPATH` — les deux `main()` (`ServerMain` et `ClientMain`) y cohabitent, chacun lancé séparément.

### 6.5 Lancement

```bash
# Terminal 1
omniNames -start 2809

# Terminal 2 — le serveur Java, nécessite java/jacorb.properties (Guide_JacORB.md §5)
cd java && java -Djacorb.config.dir=. -cp .:generated:$CLASSPATH ServerMain

# Terminal 3 — le client Java, la MÊME config jacorb.properties
cd java && java -Djacorb.config.dir=. -cp .:generated:$CLASSPATH ClientMain
```

**Piège spécifique à ce scénario** : comme les deux processus sont des JVM, il est tentant de croire qu'ils peuvent « se voir » directement — ce n'est jamais le cas. Sans Naming Service commun et sans `jacorb.properties` correctement chargé **des deux côtés**, le client Java ne trouve pas le serveur Java, exactement comme dans le scénario A (`Guide_JacORB.md` §5, le piège le plus fréquent).

---

## 7. Table des pièges

|#|Piège|Scénario concerné|Symptôme|Solution|
|---|---|---|---|---|
|1|IDL modifié sans régénération|A et B|Erreurs ou bug silencieux|Régénérer partout où l'IDL est utilisé|
|2|Édition manuelle du code généré|A et B|Comportement incohérent|Ne jamais éditer, corriger l'IDL/le servant|
|3|`_narrow`/`narrow` échoue silencieusement|A et B|Crash/NPE plus loin|Vérifier `CORBA::is_nil()` / `!= null`|
|4|Implémenter l'interface brute au lieu de `*POA`|A et B|Erreurs d'activation CORBA|Toujours étendre la classe `*POA`|
|5|NameService non résolu côté C++|A uniquement|`InvalidName` au démarrage|`-ORBInitRef` au lancement du serveur C++|
|6|`jacorb.properties` manquant/mal chargé|A et B (tout process Java)|Interop muette, aucune erreur claire|`-Djacorb.config.dir`, tester `nameclt list`|
|7|Ordre de lancement des processus|A et B|`TRANSIENT`/`COMM_FAILURE`|Toujours : Naming Service → serveur(s) → client|
|8|Relance d'omniNames avec anciens logs|A et B|Démarrage incohérent|Nettoyer `omninames-*.log` avant relance|
|9|`narrow` (Java) vs `_narrow` (C++) confondus|A uniquement (deux langages en jeu)|Erreur de compilation à la copie de code|Vérifier la convention du langage cible|
|10|Ressource partagée (`std::vector`/`List`) accédée sans protection|A et B|Corruption de données|`std::mutex` (C++) / `synchronized` (Java)|

---

## 8. Cheat-sheet

```bash
# --- Naming Service (commun aux deux scénarios) ---
omniNames -start 2809
nameclt list

# --- Génération ---
omniidl -bcxx -Cgenerated idl/Annuaire.idl                                # C++, scénario A
java -cp $CLASSPATH org.jacorb.idl.parser -d generated idl/Annuaire.idl   # Java, scénarios A et B

# --- Scénario A : Serveur C++, Client Java ---
./annuaire_server -ORBInitRef NameService=corbaname::localhost:2809
java -Djacorb.config.dir=. -cp .:generated:$CLASSPATH ClientMain

# --- Scénario B : Serveur Java, Client Java ---
java -Djacorb.config.dir=. -cp .:generated:$CLASSPATH ServerMain
java -Djacorb.config.dir=. -cp .:generated:$CLASSPATH ClientMain
```

---

## Pour aller plus loin

- `Concept_CORBA.md`, `Concept_ORB.md` — théorie
- `Guide_omniidl.md`, `Guide_JacORB.md` — outils, en détail et séparément
- `Guide_Cpp_Pratique.md` — C++ générique, réutilisable hors CORBA
- Spécification CORBA (OMG) : https://www.omg.org/spec/CORBA/
- Documentation omniORB : https://omniorb.sourceforge.io/
- Documentation JacORB : https://www.jacorb.org/