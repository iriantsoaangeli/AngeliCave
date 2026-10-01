
## Qu'est-ce que CMake ?

CMake est un outil de génération de systèmes de build multiplateforme. Il ne compile pas directement le code : il génère des fichiers de build (Makefiles, projets Ninja, solutions Visual Studio, etc.) à partir de fichiers de configuration `CMakeLists.txt`.

## Installation

- **Linux (Debian/Ubuntu)** : `sudo apt install cmake`
- **macOS (Homebrew)** : `brew install cmake`
- **Windows** : télécharger l'installateur depuis [cmake.org](https://cmake.org/download/)

Vérifier la version installée :

```bash
cmake --version
```

## Structure d'un projet minimal

```
mon_projet/
├── CMakeLists.txt
└── src/
    └── main.cpp
```

### Exemple de `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.20)
project(MonProjet VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(mon_programme src/main.cpp)
```

## Compilation d'un projet (build out-of-source)

Il est recommandé de générer les fichiers de build dans un dossier séparé (`build/`) pour ne pas polluer les sources :

```bash
mkdir build
cd build
cmake ..
cmake --build .
```

Ou en une seule commande depuis la racine (CMake ≥ 3.13) :

```bash
cmake -S . -B build
cmake --build build
```

## Options courantes

|Commande|Rôle|
|---|---|
|`cmake -S . -B build`|Configure le projet|
|`cmake --build build`|Compile le projet|
|`cmake --build build --target clean`|Nettoie les fichiers compilés|
|`cmake --install build`|Installe les binaires générés|
|`cmake -DCMAKE_BUILD_TYPE=Release ..`|Compile en mode optimisé|
|`cmake -DCMAKE_BUILD_TYPE=Debug ..`|Compile en mode debug|

## Ajouter des bibliothèques

```cmake
add_library(ma_lib STATIC src/ma_lib.cpp)
target_link_libraries(mon_programme PRIVATE ma_lib)
```

### Utiliser une dépendance externe (ex: avec `find_package`)

```cmake
find_package(Boost REQUIRED COMPONENTS filesystem)
target_link_libraries(mon_programme PRIVATE Boost::filesystem)
```

### Utiliser `FetchContent` pour récupérer une dépendance

```cmake
include(FetchContent)
FetchContent_Declare(
  googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG release-1.12.1
)
FetchContent_MakeAvailable(googletest)
```

## Gérer plusieurs sous-dossiers

```cmake
add_subdirectory(src)
add_subdirectory(tests)
```

Chaque sous-dossier peut avoir son propre `CMakeLists.txt`.

## Bonnes pratiques

- Toujours faire un build "out-of-source" (dossier `build/` séparé).
- Utiliser `target_include_directories`, `target_link_libraries` en mode moderne (avec `PRIVATE`/`PUBLIC`/`INTERFACE`) plutôt que les anciennes commandes globales (`include_directories`).
- Versionner `CMakeLists.txt` mais **pas** le dossier `build/` (à ajouter dans `.gitignore`).
- Préciser une version minimale de CMake cohérente avec les fonctionnalités utilisées.

## Ressources

- Documentation officielle : https://cmake.org/documentation/
- CMake Cookbook (livre de référence pour aller plus loin)