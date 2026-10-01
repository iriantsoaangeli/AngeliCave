
## Sommaire

1. [Compilation de base](#1-compilation-de-base)
2. [Syntaxe C++ — référence exhaustive](#2-syntaxe-c-r%C3%A9f%C3%A9rence-exhaustive)
3. [Pointeurs et gestion de mémoire](#3-pointeurs-et-gestion-de-m%C3%A9moire)
4. [X DevAPI (MySQL Connector/C++)](#4-x-devapi-mysql-connectorc)
5. [API JDBC-like (MySQL Connector/C++ classique)](#5-api-jdbc-like-mysql-connectorc-classique)
6. [Documentation téléchargeable (équivalent JDK)](#6-documentation-t%C3%A9l%C3%A9chargeable-%C3%A9quivalent-jdk)

---

## 1. Compilation de base

```bash
# Compiler un fichier unique
g++ -std=c++20 -Wall -Wextra -o programme fichier.cpp

# Exécuter
./programme
```

- `-std=c++20` : version du standard (c++11, c++14, c++17, c++20, c++23 disponibles selon la version de g++).
- `-Wall -Wextra` : active les warnings utiles, à garder systématiquement.
- Vérifier la version de g++ installée : `g++ --version`.

---

## 2. Syntaxe C++ — référence exhaustive

### 2.1 Structure d'un programme

```cpp
#include <iostream>   // directive préprocesseur : inclusion d'en-tête

int main(int argc, char* argv[]) {   // point d'entrée obligatoire
    std::cout << "Hello, world!" << std::endl;
    return 0;   // code de retour au shell (0 = succès)
}
```

### 2.2 Types de base et variables

```cpp
bool        b  = true;
char        c  = 'A';
int         i  = 42;
unsigned int ui = 42u;
short       s  = 10;
long        l  = 100000L;
long long   ll = 10000000000LL;
float       f  = 3.14f;
double      d  = 3.14159;
wchar_t     wc = L'A';

// Nombres avec séparateur de chiffres '_' (C++14+), lisibilité seulement
long population = 8'100'000'000;

// auto : déduction de type à la compilation
auto x = 5;        // int
auto y = 5.0;       // double

// const : valeur non modifiable après initialisation
const int MAX = 100;

// constexpr : évalué à la compilation (plus fort que const)
constexpr int SIZE = 10;

// références : alias d'une variable existante, jamais nul, non réassignable
int a = 1;
int& ref = a;   // ref EST a

// decltype : récupère le type d'une expression
decltype(a) autreEntier = 5;
```

### 2.3 Le underscore `_`

Trois usages distincts en C++ :

```cpp
// 1. Séparateur de chiffres (C++14+), purement visuel
int million = 1'000'000;

// 2. Convention de nommage (membres privés, ex. m_valeur ou valeur_)
class Compte {
    double solde_;   // convention "trailing underscore"
public:
    explicit Compte(double solde) : solde_(solde) {}
};

// 3. Placeholder "ignoré" dans les structured bindings (C++26) ou,
//    en pratique courante avant cela, simple variable jetable
auto [_, valeurUtile] = std::pair{0, 42};   // '_' signale "je m'en fiche"
```

### 2.4 Opérateurs

```cpp
// Arithmétiques
+  -  *  /  %

// Relationnels
== != < > <= >=

// Logiques
&& || !

// Bit à bit
&  |  ^  ~  <<  >>

// Affectation
=  +=  -=  *=  /=  %=  &=  |=  ^=  <<=  >>=

// Ternaire
int max = (a > b) ? a : b;

// Virgule (séquence, rare, souvent dans les boucles for)
for (int i = 0, j = 10; i < j; ++i, --j) {}

// sizeof / typeid
sizeof(int);            // taille en octets
typeid(a).name();       // nom du type (démangé selon compilateur)
```

### 2.5 L'opérateur `::` (résolution de portée)

```cpp
namespace maths {
    int carre(int x) { return x * x; }
}

int carre = maths::carre(4);   // :: accède à un membre de namespace

class Robot {
public:
    static int compteur;       // déclaration
    void avancer();             // déclaration
};
int Robot::compteur = 0;        // définition hors classe -> ::
void Robot::avancer() {         // définition d'une méthode hors classe -> ::
    ++compteur;
}

int global = 1;
void f() {
    int global = 2;
    std::cout << ::global;      // :: seul = portée globale
}
```

### 2.6 L'opérateur/symbole `:`

Cinq usages différents à ne pas confondre :

```cpp
// 1. Liste d'initialisation de membres (constructeur)
class Point {
    int x_, y_;
public:
    Point(int x, int y) : x_(x), y_(y) {}   // ':' introduit la liste
};

// 2. Héritage
class Animal {};
class Chien : public Animal {};              // ':' introduit la base

// 3. Spécificateurs d'accès
class Exemple {
public:      // ':' termine le mot-clé d'accès
    int a;
private:
    int b;
protected:
    int c;
};

// 4. Opérateur ternaire (partie "sinon")
int r = (a > 0) ? 1 : -1;

// 5. Label (goto / switch / boucles nommées via goto)
switch (a) {
    case 1:
        break;
    default:
        break;
}
```

### 2.7 Le destructeur `~`

```cpp
class Fichier {
public:
    Fichier()  { std::cout << "ouverture\n"; }
    ~Fichier() { std::cout << "fermeture (RAII)\n"; }   // '~' = destructeur
};
// Appelé automatiquement à la fin de la portée de l'objet
```

### 2.8 Boucles

```cpp
// for classique
for (int i = 0; i < 5; ++i) { std::cout << i; }

// range-for (C++11+) : parcourt tout conteneur/tableau
std::vector<int> v = {1, 2, 3};
for (int elem : v) { std::cout << elem; }
for (auto& elem : v) { elem *= 2; }   // référence -> modifie v

// while
int n = 5;
while (n > 0) { --n; }

// do-while (corps exécuté au moins une fois)
int m = 0;
do { ++m; } while (m < 3);

// break / continue
for (int i = 0; i < 10; ++i) {
    if (i == 5) break;      // sort de la boucle
    if (i % 2 == 0) continue; // passe à l'itération suivante
}
```

### 2.9 Fonctions

```cpp
// Déclaration / définition
int addition(int a, int b) { return a + b; }

// Surcharge (même nom, signatures différentes)
double addition(double a, double b) { return a + b; }

// Arguments par défaut
int puissance(int base, int exp = 2) { /* ... */ return base; }

// Passage par référence (évite la copie, permet modification)
void incrementer(int& x) { ++x; }

// Passage par référence constante (évite la copie, lecture seule)
void afficher(const std::string& s) { std::cout << s; }

// Fonctions inline (suggestion au compilateur)
inline int carre(int x) { return x * x; }

// Fonctions variadiques modernes (templates variadiques)
template<typename... Args>
void logAll(Args... args) { (std::cout << ... << args); }   // fold expression C++17

// Pointeurs de fonction
int (*ptrFonction)(int, int) = addition;

// std::function (enveloppe générique d'appelable)
#include <functional>
std::function<int(int,int)> op = addition;

// Lambdas
auto lambda = [](int a, int b) -> int { return a + b; };
auto compteur = [captureParValeur = 0](int a) mutable { return captureParValeur += a; };
```

### 2.10 Namespaces

```cpp
namespace projet {
    namespace utils {
        void log(const std::string& msg);
    }
}
// Alias de namespace
namespace pu = projet::utils;

// using pour éviter de préfixer (à limiter dans les .h)
using namespace std;
using std::cout;   // ciblé, préférable
```

### 2.11 Structures de contrôle

```cpp
if (a > 0) { /* ... */ }
else if (a == 0) { /* ... */ }
else { /* ... */ }

switch (a) {
    case 1: /* ... */ break;
    case 2:
    case 3: /* fallthrough volontaire */ break;
    default: /* ... */ break;
}
```

### 2.12 Classes — vue d'ensemble

```cpp
class Vehicule {
private:
    std::string marque_;
    int vitesse_ = 0;                 // valeur par défaut (in-class initializer)

public:
    // Constructeur par défaut
    Vehicule() = default;

    // Constructeur paramétré + liste d'initialisation
    explicit Vehicule(std::string marque) : marque_(std::move(marque)) {}

    // Constructeur de copie
    Vehicule(const Vehicule& autre) = default;

    // Constructeur de déplacement (move semantics, cf. section 3)
    Vehicule(Vehicule&& autre) noexcept = default;

    // Constructeur délégant
    Vehicule(std::string marque, int vitesse) : Vehicule(marque) {
        vitesse_ = vitesse;
    }

    // Destructeur
    virtual ~Vehicule() = default;

    // Méthode const : ne modifie pas l'objet
    int vitesse() const { return vitesse_; }

    // Méthode virtuelle : redéfinissable dans les classes filles
    virtual void accelerer() { ++vitesse_; }

    // Membre statique : partagé par toutes les instances
    static int nbInstances;

    // friend : accès aux membres privés depuis l'extérieur
    friend void reinitialiser(Vehicule& v);

    // Surcharge d'opérateur
    bool operator==(const Vehicule& autre) const {
        return marque_ == autre.marque_;
    }
};
int Vehicule::nbInstances = 0;
```

### 2.13 Héritage, polymorphisme, abstraction

```cpp
// Héritage simple
class Voiture : public Vehicule {
public:
    explicit Voiture(std::string marque) : Vehicule(std::move(marque)) {}

    // override : garantit qu'on redéfinit bien une méthode virtuelle existante
    void accelerer() override { /* comportement spécifique */ }
};

// Héritage multiple
class Amphibie : public Voiture, public Bateau { /* ... */ };

// final : interdit toute redéfinition/héritage ultérieur
class VoitureSport final : public Voiture {
    void accelerer() override final { /* ... */ }
};

// Classe abstraite : au moins une fonction virtuelle pure (= 0)
class FormeGeometrique {
public:
    virtual double aire() const = 0;   // pas de corps -> abstraction
    virtual ~FormeGeometrique() = default;
};

class Cercle : public FormeGeometrique {
    double rayon_;
public:
    explicit Cercle(double r) : rayon_(r) {}
    double aire() const override { return 3.14159 * rayon_ * rayon_; }
};

// Polymorphisme dynamique via pointeur/référence sur la base
void afficherAire(const FormeGeometrique& forme) {
    std::cout << forme.aire();   // appelle la bonne version selon le type réel
}
```

### 2.14 Templates (abstraction générique)

```cpp
// Fonction template
template<typename T>
T maximum(T a, T b) { return (a > b) ? a : b; }

// Classe template
template<typename T>
class Pile {
    std::vector<T> elements_;
public:
    void empiler(const T& val) { elements_.push_back(val); }
    T depiler() {
        T val = elements_.back();
        elements_.pop_back();
        return val;
    }
};
Pile<int> p;   // instanciation concrète

// Spécialisation de template
template<>
class Pile<bool> { /* implémentation optimisée bits */ };
```

### 2.15 Structs, enums, unions

```cpp
struct Point3D { double x, y, z; };   // comme class, mais public par défaut

enum class Couleur { Rouge, Vert, Bleu };   // enum typé et scopé (recommandé)
Couleur c = Couleur::Rouge;

union Donnee {   // partage la même zone mémoire entre membres
    int entier;
    float flottant;
};
```

### 2.16 Gestion des exceptions

```cpp
try {
    throw std::runtime_error("erreur explicite");
} catch (const std::runtime_error& e) {
    std::cerr << e.what();
} catch (...) {
    std::cerr << "erreur inconnue";
}
```

### 2.17 Alias de type

```cpp
typedef unsigned long ulong_t;   // ancienne syntaxe
using UlongT = unsigned long;    // syntaxe moderne, préférée (supporte les templates)
```

---

## 3. Pointeurs et gestion de mémoire

### 3.1 Bases

```cpp
int valeur = 10;
int* ptr = &valeur;   // & = adresse de ; * (déclaration) = "pointeur vers"
std::cout << *ptr;     // * (déréférencement) = "valeur pointée"
*ptr = 20;             // modifie valeur via le pointeur

int* ptrNul = nullptr; // pointeur nul (C++11+), à préférer à NULL/0
```

### 3.2 Pile (stack) vs tas (heap)

- **Stack** : allocation automatique, libérée à la sortie de portée. Rapide, taille limitée.
- **Heap** : allocation manuelle (`new`), doit être libérée explicitement (`delete`), sinon fuite mémoire.

```cpp
void f() {
    int surStack = 1;             // libéré automatiquement à la fin de f()
    int* surHeap = new int(1);    // reste alloué jusqu'à delete
    delete surHeap;               // libération manuelle obligatoire
}
```

### 3.3 Allocation dynamique

```cpp
// Objet unique
int* p = new int(42);
delete p;
p = nullptr;   // bonne pratique après delete

// Tableau dynamique
int* tab = new int[10];
delete[] tab;   // delete[] obligatoire pour les tableaux, sinon comportement indéfini

// Objet de classe
Vehicule* v = new Vehicule("Renault");
delete v;   // appelle le destructeur puis libère la mémoire
```

### 3.4 Pointeurs et tableaux

```cpp
int arr[5] = {1, 2, 3, 4, 5};
int* p = arr;        // un tableau "décaie" en pointeur sur son premier élément
std::cout << *(p + 2); // arithmétique de pointeur == arr[2]
std::cout << p[2];     // équivalent, plus lisible
```

### 3.5 const et pointeurs

```cpp
const int* p1;        // pointeur vers un int constant : *p1 non modifiable
int* const p2 = &valeur; // pointeur constant : p2 ne peut pas pointer ailleurs
const int* const p3 = &valeur; // les deux à la fois
```

### 3.6 Pointeurs de pointeurs et pointeurs de fonction

```cpp
int a = 5;
int* p = &a;
int** pp = &p;   // pointeur vers pointeur

void saluer() { std::cout << "salut"; }
void (*ptrFonc)() = saluer;
ptrFonc();
```

### 3.7 Dangers classiques

```cpp
// Fuite mémoire : oubli de delete
void fuite() {
    int* p = new int(5);
    // pas de delete -> mémoire jamais libérée
}

// Pointeur pendouillant (dangling) : usage après delete
int* p = new int(5);
delete p;
// std::cout << *p;   // comportement indéfini, UB

// Double free : delete appelé deux fois
delete p;
// delete p;   // UB, à éviter absolument
```

### 3.8 Smart pointers (C++11+) — solution moderne recommandée

```cpp
#include <memory>

// unique_ptr : propriété exclusive, non copiable, déplaçable
std::unique_ptr<Vehicule> uv = std::make_unique<Vehicule>("Peugeot");
// libéré automatiquement à la sortie de portée, pas de delete manuel

// shared_ptr : propriété partagée, compteur de références
std::shared_ptr<Vehicule> sv1 = std::make_shared<Vehicule>("Tesla");
std::shared_ptr<Vehicule> sv2 = sv1;   // compteur = 2
// libéré quand le compteur atteint 0

// weak_ptr : référence non-possédante, évite les cycles de shared_ptr
std::weak_ptr<Vehicule> wv = sv1;
if (auto locked = wv.lock()) { /* utiliser locked si encore vivant */ }
```

### 3.9 Sémantique de déplacement (move semantics)

```cpp
#include <utility>

std::vector<int> creerVecteur() {
    std::vector<int> v = {1, 2, 3};
    return v;   // move implicite au retour (copie évitée)
}

std::vector<int> v1 = {1, 2, 3};
std::vector<int> v2 = std::move(v1);   // transfert de propriété, v1 devient "vide"
// v1 est dans un état valide mais non spécifié après le move -> ne plus l'utiliser

// Référence rvalue && dans un constructeur de déplacement
class Buffer {
    int* data_;
public:
    Buffer(Buffer&& autre) noexcept : data_(autre.data_) {
        autre.data_ = nullptr;   // vide l'objet source
    }
};
```

### 3.10 RAII (Resource Acquisition Is Initialization)

Principe central de la gestion mémoire en C++ moderne : toute ressource (mémoire, fichier, verrou) est liée à la durée de vie d'un objet. Le constructeur acquiert, le destructeur `~` libère automatiquement. Les smart pointers, `std::vector`, `std::fstream` etc. appliquent tous ce principe — d'où la règle **« pas de `new`/`delete` nu en code moderne, toujours passer par un objet RAII »**.

---

## 4. X DevAPI (MySQL Connector/C++)

L'X DevAPI est l'API moderne (document store + SQL) de MySQL Connector/C++, basée sur le protocole X.

### 4.1 Installation (Ubuntu/Debian)

```bash
sudo apt-get update
sudo apt-get install libmysqlcppconn-dev
```

Ce paquet installe les en-têtes et bibliothèques pour l'X DevAPI **et** l'API JDBC-like (section 5).

### 4.2 Connexion et CRUD sur une collection (document store)

```cpp
// fichier : devapi_exemple.cpp
#include <mysqlx/xdevapi.h>
#include <iostream>

using namespace mysqlx;

int main() {
    try {
        // 1. Connexion : mysqlx://utilisateur:motdepasse@hote:port
        Session session("mysqlx://root:motdepasse@127.0.0.1:33060");

        // 2. Sélection/obtention du schéma (base de données)
        Schema db = session.getSchema("test");

        // 3. Création d'une collection (équivalent d'une table document)
        Collection coll = db.createCollection("ma_collection", true);

        // 4. Insertion de documents
        coll.add(R"({ "nom": "Alice", "age": 30 })")
            .add(R"({ "nom": "Bob", "age": 25 })")
            .execute();

        // 5. Recherche
        DocResult resultats = coll.find("age > :age")
                                   .bind("age", 20)
                                   .execute();

        for (DbDoc doc = resultats.fetchOne(); doc; doc = resultats.fetchOne()) {
            std::cout << doc << std::endl;
        }

        // 6. Suppression
        coll.remove("nom = :nom").bind("nom", "Bob").execute();

    } catch (const mysqlx::Error& e) {
        std::cerr << "Erreur X DevAPI : " << e.what() << std::endl;
    }
    return 0;
}
```

### 4.3 Exécuter du SQL classique via X DevAPI

```cpp
Session session("mysqlx://root:motdepasse@127.0.0.1:33060");
SqlResult res = session.sql("SELECT nom, age FROM utilisateurs WHERE age > ?")
                        .bind(18)
                        .execute();
for (Row row = res.fetchOne(); row; row = res.fetchOne()) {
    std::cout << row[0] << " a " << row[1] << " ans\n";
}
```

### 4.4 Compilation

```bash
g++ -std=c++17 devapi_exemple.cpp -o devapi_exemple \
    -I/usr/include/mysql-cppconn \
    -lmysqlcppconnx
```

> Sur certaines distributions/versions, l'include path est `/usr/include/mysql-cppconn-8` et la lib `-lmysqlcppconn8`. Vérifier avec `dpkg -L libmysqlcppconn-dev | grep xdevapi.h` en cas d'erreur de compilation.

---

## 5. API JDBC-like (MySQL Connector/C++ classique)

API historique qui imite volontairement le JDBC de Java (`Driver`, `Connection`, `Statement`, `PreparedStatement`, `ResultSet`). Utilise le protocole MySQL classique (port 3306), pas besoin du plugin X.

### 5.1 Installation

Même paquet que la section 4 :

```bash
sudo apt-get install libmysqlcppconn-dev
```

### 5.2 Connexion et requêtes

```cpp
// fichier : jdbc_exemple.cpp
#include <mysql/jdbc.h>     // en-tête unique depuis Connector/C++ 8.0.16+
#include <iostream>
#include <memory>

int main() {
    try {
        // 1. Obtenir le driver (équivalent DriverManager côté JDBC Java)
        sql::mysql::MySQL_Driver* driver = sql::mysql::get_driver_instance();

        // 2. Connexion (unique_ptr = RAII, pas de delete manuel)
        std::unique_ptr<sql::Connection> con(
            driver->connect("tcp://127.0.0.1:3306", "root", "motdepasse"));

        con->setSchema("test");

        // 3. Statement simple
        std::unique_ptr<sql::Statement> stmt(con->createStatement());
        stmt->execute("CREATE TABLE IF NOT EXISTS utilisateurs (id INT PRIMARY KEY AUTO_INCREMENT, nom VARCHAR(50), age INT)");

        // 4. PreparedStatement (placeholders '?', protège des injections SQL)
        std::unique_ptr<sql::PreparedStatement> pstmt(
            con->prepareStatement("INSERT INTO utilisateurs (nom, age) VALUES (?, ?)"));
        pstmt->setString(1, "Alice");
        pstmt->setInt(2, 30);
        pstmt->executeUpdate();

        // 5. Lecture des résultats
        std::unique_ptr<sql::ResultSet> res(
            stmt->executeQuery("SELECT id, nom, age FROM utilisateurs"));
        while (res->next()) {
            std::cout << res->getInt("id") << " - "
                      << res->getString("nom") << " - "
                      << res->getInt("age") << std::endl;
        }

    } catch (sql::SQLException& e) {
        std::cerr << "Erreur SQL (" << e.getErrorCode() << ") : " << e.what() << std::endl;
    }
    return 0;
}
```

### 5.3 Compilation

```bash
g++ -std=c++17 jdbc_exemple.cpp -o jdbc_exemple \
    -I/usr/include/mysql-cppconn \
    -lmysqlcppconn
```

### 5.4 Parallèle avec JDBC (Java)

|JDBC (Java)|Connector/C++ JDBC-like|
|---|---|
|`DriverManager.getConnection()`|`driver->connect(...)`|
|`Connection`|`sql::Connection`|
|`Statement`|`sql::Statement`|
|`PreparedStatement`|`sql::PreparedStatement`|
|`ResultSet`|`sql::ResultSet`|
|`SQLException`|`sql::SQLException`|
|Garbage collector|`std::unique_ptr` / `delete` manuel (RAII)|

---

## 6. Documentation téléchargeable (équivalent JDK)

Comme la documentation JDK que l'on peut télécharger pour une consultation hors-ligne, voici les équivalents C++ / MySQL Connector/C++ officiels et vérifiés :

- **Référence C++ complète hors-ligne (langage + bibliothèque standard)**, équivalent direct des "docs JDK" : archive HTML téléchargeable sur [https://en.cppreference.com/w/Cppreference:Archives](https://en.cppreference.com/w/Cppreference:Archives)
    
- **MySQL Connector/C++ 9.0 — Developer Guide, PDF téléchargeable** (installation, X DevAPI, API JDBC-like, tout en un document) : [https://downloads.mysql.com/docs/connector-cpp-9.0-en.pdf](https://downloads.mysql.com/docs/connector-cpp-9.0-en.pdf)
    
- **Référence en ligne (classes X DevAPI, JDBC-like, exemples officiels)** : [https://dev.mysql.com/doc/dev/connector-cpp/](https://dev.mysql.com/doc/dev/connector-cpp/)
    
- **Page de téléchargement des paquets Connector/C++ (Linux/Windows/macOS, sources incluses)** : [https://dev.mysql.com/downloads/connector/cpp/](https://dev.mysql.com/downloads/connector/cpp/)
    

---

## Résumé rapide (aide-mémoire)

|Symbole/Notion|Rôle|
|---|---|
|`::`|Résolution de portée (namespace, classe, global)|
|`:`|Init-list constructeur / héritage / accès / ternaire / label|
|`~`|Destructeur|
|`_`|Séparateur de chiffres / convention de nommage / placeholder|
|`*`|Déclaration de pointeur / déréférencement|
|`&`|Adresse-de / référence / rvalue-ref si `&&`|
|`new` / `delete`|Allocation/libération manuelle sur le tas|
|`unique_ptr` / `shared_ptr`|Gestion mémoire moderne (RAII)|
|`virtual` / `= 0`|Polymorphisme / classe abstraite|
|`mysqlx::Session`|Point d'entrée X DevAPI|
|`sql::mysql::get_driver_instance()`|Point d'entrée API JDBC-like|