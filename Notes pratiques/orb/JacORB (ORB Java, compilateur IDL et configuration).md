# [[Une ORB , c'est quoi ?]]

## 1. Pourquoi JacORB

`org.omg.CORBA`/`idlj`, le CORBA historiquement intégré au JDK, a été retiré à partir de Java 11 (JEP 320) — ce n'est **pas** un désaveu de la norme CORBA elle-même (voir `Concept_CORBA.md`), seulement le retrait d'_une_ implémentation devenue obsolète. JacORB est aujourd'hui le seul ORB Java encore activement maintenu (ORBacus et OpenORB/Apache sont abandonnés). Si l'on est contraint à un JDK 8 legacy, le CORBA intégré reste utilisable, mais JacORB est recommandé même dans ce cas si le choix est possible.

## 2. Installation

JacORB n'est pas dans les dépôts apt standards. On le récupère en `.zip` :

```bash
cd /opt
sudo wget https://www.jacorb.org/releases/3.9/jacorb-3.9-binary.zip
sudo unzip jacorb-3.9-binary.zip -d jacorb
```

Dans `~/.bashrc` :

```bash
export JACORB_HOME=/opt/jacorb
export PATH=$JACORB_HOME/bin:$PATH
export CLASSPATH=$JACORB_HOME/lib/jacorb.jar:$JACORB_HOME/lib/slf4j-api.jar:$JACORB_HOME/lib/logback-classic.jar:$JACORB_HOME/lib/logback-core.jar:$CLASSPATH
```

## 3. Le compilateur IDL de JacORB

Comme `omniidl` côté C++ (voir `Guide_omniidl.md`), JacORB embarque son propre compilateur IDL — chaque implémentation d'ORB impose le sien, pour générer du code compatible avec ses propres classes runtime.

```bash
mkdir -p java/generated
idl -d java/generated idl/Demo.idl
```

- `idl` : le script de lancement du compilateur (l'équivalent du `protoc` pour Protobuf)
- `-d java/generated` : dossier de sortie
- `idl/Demo.idl` : le fichier source, identique à celui utilisé côté C++

**Piège fréquent — le nom de la commande change selon la version** : parfois `idl`, parfois `jacidl.sh`. Si aucun des deux n'existe, appeler directement la classe Java du compilateur (elle, ne change jamais de nom) :

```bash
java -cp $CLASSPATH org.jacorb.idl.parser -d java/generated idl/Demo.idl
```

C'est la solution la plus fiable en cas de doute sur la version — elle fonctionne toujours, quel que soit le script wrapper disponible.

## 4. Ce que ça produit

Dans `java/generated/Demo/` (un dossier par module IDL) — plusieurs fichiers **par interface et par struct** :

|Fichier|Rôle|
|---|---|
|`FileWriterService.java`|L'interface Java pure|
|`FileWriterServicePOA.java`|Classe **abstraite** que le servant doit étendre (`extends`) — connecte automatiquement la classe à l'ORB|
|`FileWriterServiceHelper.java`|Utilitaire statique, sert surtout à `narrow(obj)` : convertir une référence CORBA générique en type concret|
|`FileWriterServiceHolder.java`|Utilitaire pour les paramètres `out`/`inout` — Java n'a pas de sortie par référence nativement, l'IDL si|
|`Produit.java`, `ProduitSeqHelper.java`, etc.|Un fichier par type déclaré dans l'IDL|

**À ne jamais faire** : éditer ces fichiers générés à la main — que ce soit `*POA`, `*Helper` ou `*Holder`. Si un comportement doit changer, le problème est dans l'IDL (la forme du contrat) ou dans le servant, jamais dans le code de plomberie généré.

## 5. Configuration : pointer JacORB vers le bon Naming Service

C'est le point le plus délicat de l'usage de JacORB en interopérabilité. `java/jacorb.properties` :

```properties
org.omg.CORBA.ORBClass=org.jacorb.orb.ORB
org.omg.CORBA.ORBSingletonClass=org.jacorb.orb.ORBSingleton
ORBInitRef.NameService=corbaname::localhost:2809
```

Lancement avec ce fichier chargé :

```bash
java -Djacorb.config.dir=. -cp .:generated:$CLASSPATH FileServerMain
```

**Piège le plus fréquent avec JacORB, à retenir absolument** : si `jacorb.properties` n'est pas trouvé (mauvais `CLASSPATH` ou mauvais `-Djacorb.config.dir`), JacORB **ne plante pas** — il démarre silencieusement avec sa propre configuration par défaut, qui pointe vers un Naming Service différent de celui attendu. Le serveur Java « s'enregistre » ailleurs, un client tiers ne le trouve jamais, et il n'y a **aucun message d'erreur explicite** pointant vers la vraie cause.

Premier réflexe en cas d'échec silencieux (nécessite `nameclt`, fourni par omniORB) :

```bash
nameclt list
```

Si le service Java attendu n'apparaît pas dans cette liste après son lancement, la configuration JacORB n'a pas été chargée.

## 6. narrow — la convention Java

```java
FileWriterService fileWriter = FileWriterServiceHelper.narrow(obj);
```

En Java, le narrowing passe par la méthode statique `narrow` (sans underscore) de la classe `*Helper` générée — à distinguer de la convention C++ (`_narrow`, avec underscore, voir `Guide_omniidl.md`/`Manuel_CORBA_Principal.md`). Erreur de copier-coller fréquente entre les deux langages.

`narrow` échoue **silencieusement** (retourne un objet nil, pas d'exception) si le type demandé ne correspond pas à l'objet réellement enregistré sous ce nom. Toujours vérifier :

```java
if (fileWriter == null) { /* gérer l'échec */ }
```

## 7. Pièges spécifiques à JacORB

|Piège|Symptôme|Solution|
|---|---|---|
|Nom du compilateur IDL variable selon la version|`idl` introuvable|Appeler `org.jacorb.idl.parser` directement|
|Édition manuelle du code généré|Comportement incohérent, modifications perdues|Ne jamais éditer, corriger l'IDL ou le servant|
|`jacorb.properties` introuvable ou mal chargé|Naming Service différent, échec d'interopérabilité **silencieux**|Vérifier `-Djacorb.config.dir`, tester avec `nameclt list`|
|`narrow` (Java) confondu avec `_narrow` (C++)|Erreur de compilation lors d'un copier-coller entre langages|Vérifier la convention du langage cible|
|Implémenter l'interface `*Operations` brute au lieu de `*POA`|Erreurs d'activation CORBA|Toujours étendre la classe `*POA` générée|
|Compilation Java depuis le mauvais dossier|« package Demo does not exist »|Compiler depuis la racine qui contient le dossier du module généré|

## 8. Cheat-sheet

```bash
# Génération
java -cp $CLASSPATH org.jacorb.idl.parser -d generated idl/Demo.idl

# Lancement avec la config JacORB
java -Djacorb.config.dir=. -cp .:generated:$CLASSPATH MaClasseMain

# Compilation
javac -cp generated:$CLASSPATH -d classes generated/Demo/*.java src/*.java

# Diagnostic Naming Service
nameclt list
```

---

**Voir aussi :**

- `Guide_omniidl.md` — l'équivalent côté C++
- `Manuel_CORBA_Principal.md` — où ces commandes s'insèrent dans le flux complet du projet