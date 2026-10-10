# Activité pratique n° 2 — Architecture microservices avec Spring Cloud

Projet maintenu par **Mustapha Aarab** — [mostaphaaarabe663@gmail.com](mailto:mostaphaaarabe663@gmail.com).

Cette application de démonstration gère des clients, des produits et des factures. Les services utilisent Spring Cloud Config pour la configuration centralisée, Eureka pour la découverte et Spring Cloud Gateway comme point d'entrée HTTP. Le service de facturation interroge les services clients et produits avec OpenFeign.

## Services

| Dossier | Rôle | Port |
| --- | --- | --- |
| `config-service` | Serveur de configuration | 9999 |
| `discovery-service` | Annuaire Eureka | 8761 |
| `customer-service` | Clients et projections Spring Data REST | 8081 |
| `inventory-service` | Catalogue de produits | 8082 |
| `billing-service` | Factures et enrichissement des lignes de facture | 8083 |
| `gatewey-service` | Passerelle HTTP et découverte des routes | 8888 |

Le dossier `config-repo` contient les configurations partagées et les profils `dev` et `prod`. Le serveur de configuration lit ce dossier dans la branche `main` de ce dépôt. Les packages Java utilisent le préfixe `net.aarab`.

## Prérequis

- JDK 21 avec `JAVA_HOME` configuré.
- Maven 3.9 ou les scripts Maven Wrapper fournis dans chaque service.
- Accès à Internet pour télécharger les dépendances et lire la configuration Git.

Versions déclarées : Spring Boot 3.3.5 et Spring Cloud 2023.0.3. Les bases H2 sont en mémoire : leur contenu est recréé au démarrage.

## Compilation

Chaque service possède son propre `pom.xml`. Le POM à la racine est un petit projet Java indépendant ; il ne compile pas les six services.

Depuis la racine, dans PowerShell :

```powershell
$services = 'config-service', 'discovery-service', 'customer-service', 'inventory-service', 'gatewey-service', 'billing-service'
foreach ($service in $services) {
    mvn -f "$service/pom.xml" clean package -DskipTests
    if ($LASTEXITCODE -ne 0) { throw "Échec de compilation : $service" }
}
```

Pour utiliser le wrapper à la place de Maven installé, ouvrir le dossier du service et lancer `./mvnw.cmd clean package -DskipTests` sous Windows, ou `./mvnw clean package -DskipTests` sous Linux/macOS.

## Démarrage

Ouvrir un terminal par service et lancer les commandes suivantes depuis la racine, dans cet ordre. Attendre que chaque service soit démarré avant de passer au suivant.

```powershell
mvn -f discovery-service/pom.xml spring-boot:run
mvn -f config-service/pom.xml spring-boot:run
mvn -f customer-service/pom.xml spring-boot:run
mvn -f inventory-service/pom.xml spring-boot:run
mvn -f gatewey-service/pom.xml spring-boot:run
mvn -f billing-service/pom.xml spring-boot:run
```

Avant de lancer la facturation, vérifier dans Eureka que `CUSTOMER-SERVICE` et `INVENTORY-SERVICE` sont enregistrés et disponibles. Le service de facturation les appelle au démarrage pour créer les données de démonstration. La découverte peut prendre quelques instants après leur démarrage.

## Vérification

- Eureka : <http://localhost:8761>
- Configuration des clients : <http://localhost:9999/customer-service/default>
- Clients : <http://localhost:8081/api/customers>
- Produits : <http://localhost:8082/api/products>
- Facture enrichie : <http://localhost:8083/bills/1>
- Clients via la passerelle : <http://localhost:8888/customer-service/api/customers>
- Produits via la passerelle : <http://localhost:8888/inventory-service/api/products>
- Facture via la passerelle : <http://localhost:8888/billing-service/bills/1>
- Paramètres partagés : <http://localhost:8081/testConfig1>
- Paramètres du service clients : <http://localhost:8081/testConfig2>

Le fichier `gatewey-service/src/main/resources/a.yml` contient un exemple de routes statiques ; il n'est pas chargé par défaut. La passerelle utilise la découverte dynamique configurée dans son code Java.

Les tests fournis chargent les contextes Spring. Le test du service clients a besoin des paramètres du Config Server ; celui de la facturation appelle aussi les services clients et produits. Une fois ces dépendances démarrées, les tests d'un service peuvent être lancés avec `mvn -f <service>/pom.xml test`.

Les profils `dev` et `prod` utilisent tous deux H2 en mémoire. Ils servent à illustrer la configuration par profil et ne constituent pas une configuration de déploiement en production.
