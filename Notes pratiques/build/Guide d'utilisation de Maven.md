
## Qu'est-ce que Maven ?

Apache Maven est un outil de gestion et d'automatisation de build pour les projets Java. Il gère les dépendances, la compilation, les tests, l'empaquetage et le déploiement, à partir d'un fichier de configuration central : `pom.xml`.

## Installation

- **Linux (Debian/Ubuntu)** : `sudo apt install maven`
- **macOS (Homebrew)** : `brew install maven`
- **Windows** : télécharger depuis [maven.apache.org](https://maven.apache.org/download.cgi) et ajouter au `PATH`

Vérifier l'installation :

```bash
mvn -version
```

## Structure standard d'un projet Maven

```
mon_projet/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/       # code source
    │   └── resources/  # fichiers de configuration, ressources
    └── test/
        ├── java/       # tests unitaires
        └── resources/
```

## Le fichier `pom.xml`

Exemple minimal :

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.exemple</groupId>
  <artifactId>mon-projet</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.13.2</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

## Cycle de vie Maven

Maven repose sur un cycle de vie en phases successives. Les principales phases du cycle `default` :

|Phase|Rôle|
|---|---|
|`validate`|Vérifie que le projet est correct|
|`compile`|Compile le code source|
|`test`|Exécute les tests unitaires|
|`package`|Génère le `.jar`/`.war`|
|`verify`|Vérifie les résultats des tests d'intégration|
|`install`|Installe le package dans le dépôt local (`~/.m2`)|
|`deploy`|Déploie le package vers un dépôt distant|

Chaque phase exécute automatiquement les phases précédentes.

## Commandes courantes

```bash
mvn compile          # Compile le code
mvn test             # Exécute les tests
mvn package          # Génère le .jar/.war
mvn install          # Installe dans le dépôt local
mvn clean            # Supprime le dossier target/
mvn clean install    # Nettoie puis reconstruit tout
```

## Gérer les dépendances

Ajouter une dépendance : rechercher son coordonnées (`groupId`, `artifactId`, `version`) sur [Maven Central](https://mvnrepository.com/) et l'ajouter dans `<dependencies>` :

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
  <version>3.3.0</version>
</dependency>
```

Voir l'arbre des dépendances :

```bash
mvn dependency:tree
```

## Plugins utiles

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.13.0</version>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>3.2.5</version>
    </plugin>
  </plugins>
</build>
```

- `maven-compiler-plugin` : configure la compilation Java
- `maven-surefire-plugin` : exécute les tests unitaires
- `maven-shade-plugin` / `maven-assembly-plugin` : créer un jar exécutable avec dépendances ("fat jar")

## Profils Maven

Permettent de personnaliser le build selon un contexte (dev, prod...) :

```xml
<profiles>
  <profile>
    <id>prod</id>
    <properties>
      <env>production</env>
    </properties>
  </profile>
</profiles>
```

Activation :

```bash
mvn install -Pprod
```

## Bonnes pratiques

- Toujours préciser les versions des dépendances (éviter les versions flottantes).
- Utiliser un gestionnaire de versions centralisé avec `<dependencyManagement>` dans les projets multi-modules.
- Ne pas versionner le dossier `target/` (à ajouter dans `.gitignore`).
- Utiliser `mvn clean` régulièrement pour éviter les artefacts obsolètes.

## Ressources

- Documentation officielle : https://maven.apache.org/guides/
- Dépôt central des dépendances : https://mvnrepository.com/