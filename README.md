# Injection des dépendances

Mise en œuvre de l'injection des dépendances en Java 21, d'après la vidéo
[« Mise en oeuvre de l'injection des dépendances »](https://www.youtube.com/watch?v=vOLqabN-n2k) (Pr. Mohamed YOUSSFI).

## 1. Objectif

Construire une application **fermée à la modification et ouverte à l'extension** :
on doit pouvoir ajouter une nouvelle implémentation sans toucher au code existant.
On y arrive grâce au **couplage faible** (dépendre d'interfaces, pas de classes)
et à l'**injection des dépendances**.

## 2. Structure du projet

```
src/main/java/net/madani/
├── dao/      IDao, DaoImpl            (couche accès aux données)
├── ext/      DaoImplV2                (extension : nouvelle version)
├── metier/   IMetier, MetierImpl      (couche métier)
└── pres/     Pres1, Pres2, PresSpringXML, PresSpringAnnotation
src/main/resources/config.xml          (configuration Spring XML)
config.txt                             (configuration de l'instanciation dynamique)
```

## 3. Couches DAO et métier

- `IDao` déclare `getData()` ; `DaoImpl` la réalise (version base de données, renvoie 34).
- `IMetier` déclare `calcul()` ; `MetierImpl` la réalise.

`MetierImpl` possède un attribut `private IDao dao` : il dépend de l'**interface**
et non d'une classe concrète, c'est le couplage faible. Cet attribut n'est pas
initialisé, il vaut donc `null` au départ : il faut lui **injecter** un objet,
soit par le **constructeur** `MetierImpl(IDao dao)`, soit par le **setter** `setDao(IDao dao)`.

## 4. Instanciation statique — `Pres1`

```java
DaoImpl d = new DaoImpl();
MetierImpl metier = new MetierImpl(d);   // ou metier.setDao(d);
System.out.println("RES= " + metier.calcul());
```

Sortie :

```
Version base de données
RES= -543.8358768288695
```

Sans injection (`new MetierImpl()` seul), on obtient une `NullPointerException`
car `this.dao` est `null`.

Limite : pour passer à `DaoImplV2` (version capteurs), il faut modifier `Pres1`
et recompiler. Les couches DAO et métier sont fermées à la modification,
mais pas encore la couche présentation.

## 5. Instanciation dynamique — `config.txt` + `Pres2`

`config.txt` (à la racine du projet) contient les noms des classes à utiliser :

```
net.madani.dao.DaoImpl
net.madani.metier.MetierImpl
```

`Pres2` lit ce fichier, charge les classes avec `Class.forName(...)`, puis les
instancie par réflexion :

- injection par le constructeur : `cMetier.getConstructor(IDao.class).newInstance(d)` ;
- injection par le setter : `getDeclaredMethod("setDao", IDao.class).invoke(metier, d)`.

Sorties :

```
Version base de données        (net.madani.dao.DaoImpl)
RES=-543.8358768288695

Version capteurs ....          (net.madani.ext.DaoImplV2)
RES=190.87526861687965
```

Pour changer de version, on modifie seulement `config.txt` : la couche
présentation est maintenant fermée à la modification.

Exceptions possibles : `FileNotFoundException`, `ClassNotFoundException`,
`InstantiationException`, `IllegalAccessException`, `NoSuchMethodException`,
`InvocationTargetException`.

## 6. Spring — version XML

Dépendances Maven : `spring-core`, `spring-context`, `spring-beans` (6.2.3).

`src/main/resources/config.xml` :

```xml
<bean id="d" class="net.madani.ext.DaoImplV2"></bean>
<bean id="metier" class="net.madani.metier.MetierImpl">
    <constructor-arg ref="d"></constructor-arg>
</bean>
```

- `<property name="dao" ref="d"/>` : Spring appelle `setDao(d)` (injection par le setter).
- `<constructor-arg ref="d"/>` : Spring appelle `new MetierImpl(d)` (injection par le constructeur).

`PresSpringXML` charge le contexte avec `new ClassPathXmlApplicationContext("config.xml")`
puis récupère le bean avec `getBean(IMetier.class)`.

```
Version capteurs ....
RES=190.87526861687965
```

## 7. Spring — version annotations

- `@Component("d")` sur `DaoImpl`, `@Repository("d2")` sur `DaoImplV2`,
  `@Service("metier")` sur `MetierImpl`. `@Repository`, `@Service` et `@Controller`
  sont des spécialisations de `@Component` pour les couches DAO, métier et web.
- `@Autowired` demande à Spring d'injecter la dépendance (sur l'attribut, le setter ou le constructeur).
- Avec deux beans de type `IDao` (`d` et `d2`), Spring lève une
  `UnsatisfiedDependencyException` : il ne sait pas lequel choisir.
  `@Qualifier("d2")` indique le bean à injecter.

`PresSpringAnnotation` crée le contexte avec
`new AnnotationConfigApplicationContext("net.madani")`, qui scanne le package de base.

```
Version capteurs ....
RES=190.87526861687965
```

## 8. Conclusion

<!-- À écrire avec tes mots : 2-3 phrases sur l'inversion de contrôle. -->

## Exécution

Ouvrir le projet dans IntelliJ (Maven, JDK 21) et lancer les classes du package
`net.madani.pres`. `Pres2` doit être lancé depuis la racine du projet pour trouver `config.txt`.
