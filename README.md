# Rapport du Projet Digital Banking - Partie Backend

## Introduction

Ce projet s'inscrit dans le cadre du cours "Architecture JEE et Systèmes Distribués" et concerne le développement d'une application bancaire numérique. La partie backend, présentée dans ce rapport, est développée avec Spring Boot et suit une architecture en couches permettant la gestion des clients, des comptes bancaires (courants et d'épargne) et des opérations bancaires (débit, crédit, transfert).

L'application repose sur le framework Spring Boot 3.4.5 et utilise Java 17, avec une base de données MariaDB pour la persistance des données. Elle expose une API REST complète permettant d'effectuer toutes les opérations nécessaires à la gestion bancaire.

## Table des matières

1. [Structure du projet](#structure-du-projet)
2. [Entités et modèle de données](#entités-et-modèle-de-données)
3. [Couche de persistance (Repositories)](#couche-de-persistance-repositories)
4. [Couche service](#couche-service)
5. [Mappers et DTOs](#mappers-et-dtos)
6. [Contrôleurs REST](#contrôleurs-rest)
7. [Configuration et dépendances](#configuration-et-dépendances)
8. [Conclusion](#conclusion)

## Structure du projet

Le projet Digital Banking est structuré selon l'architecture en couches standard de Spring Boot, avec une séparation claire des responsabilités. Cette organisation facilite la maintenance et l'évolution future de l'application.

La structure des packages est la suivante:

```
ma.enset.digitalbanking
    ├── DigitalBankingApplication.java (Point d'entrée de l'application)
    ├── controllers (Exposition des API REST)
    │   ├── BankAccountRestAPI.java
    │   └── CustomerRestController.java
    ├── dtos (Objets de transfert de données)
    │   ├── AccountHistoryDTO.java
    │   ├── AccountOperationDTO.java
    │   ├── BankAccountDTO.java
    │   ├── CurrentBankAccountDTO.java
    │   ├── CustomerDTO.java
    │   └── SavingBankAccountDTO.java
    ├── entities (Entités persistantes)
    │   ├── AccountOperation.java
    │   ├── BankAccount.java
    │   ├── CurrentAccount.java
    │   ├── Customer.java
    │   └── SavingAccount.java
    ├── enums (Types énumérés)
    │   ├── AccountStatus.java
    │   └── OperationType.java
    ├── exceptions (Gestion des erreurs)
    ├── mappers (Conversion entre entités et DTOs)
    └── repositories (Accès aux données)
        ├── AccountOperationRepository.java
        ├── BankAccountRepository.java
        └── CustomerRepository.java
    └── services (Logique métier)
        ├── BankAccountService.java
        └── impl (Implémentations des services)
```

Cette architecture respecte les principes SOLID et permet une séparation claire des préoccupations :
- Les contrôleurs gèrent uniquement les requêtes HTTP
- Les services contiennent la logique métier
- Les repositories gèrent l'accès aux données
- Les DTOs assurent le transfert sécurisé des données entre les couches
- Les mappers convertissent les entités en DTOs et vice versa

Cette organisation modulaire facilite les tests unitaires et l'intégration de nouvelles fonctionnalités.

## Entités et modèle de données

Le modèle de données du projet digital banking est conçu autour de trois entités principales: Customer, BankAccount et AccountOperation. Ces entités reflètent le domaine métier d'une application bancaire.

### La classe Customer

La classe Customer représente un client de la banque:

```java
@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Customer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;
    @OneToMany(mappedBy = "customer")
    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    private List<BankAccount> bankAccounts;
}
```

Cette classe utilise:
- L'annotation `@Entity` pour indiquer qu'elle sera persistée en base de données
- Les annotations Lombok (`@Data`, `@NoArgsConstructor`, `@AllArgsConstructor`) pour réduire le code boilerplate
- Une relation `@OneToMany` avec les comptes bancaires, indiquant qu'un client peut avoir plusieurs comptes
- L'annotation `@JsonProperty(access = JsonProperty.Access.WRITE_ONLY)` pour éviter les cycles de sérialisation JSON

### La classe BankAccount

BankAccount est une classe abstraite qui représente un compte bancaire générique:

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "TYPE", length = 4)
@Data
@NoArgsConstructor
@AllArgsConstructor
public abstract class BankAccount {
    @Id
    private String id;
    private double balance;
    private Date createdAt;
    @Enumerated(EnumType.STRING)
    private AccountStatus status;
    @ManyToOne
    private Customer customer;

    @OneToMany(mappedBy = "bankAccount", fetch = FetchType.LAZY)
    private List<AccountOperation> accountOperations;
}
```

Points importants:
- L'utilisation de `@Inheritance(strategy = InheritanceType.SINGLE_TABLE)` et `@DiscriminatorColumn` pour implémenter l'héritage dans la base de données avec une stratégie de table unique
- Le statut du compte est représenté par une énumération
- La relation bidirectionnelle avec Customer (`@ManyToOne`) et AccountOperation (`@OneToMany`)
- Le chargement paresseux (LAZY) des opérations pour optimiser les performances

### Les classes CurrentAccount et SavingAccount

Ces deux classes héritent de BankAccount et représentent les types spécifiques de comptes:

```java
@Entity
@DiscriminatorValue("CA")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class CurrentAccount extends BankAccount {
    private double overDraft;
}
```

```java
@Entity
@DiscriminatorValue("SA")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class SavingAccount extends BankAccount {
    private double interestRate;
}
```

- `CurrentAccount` (compte courant) possède un attribut `overDraft` qui représente le découvert autorisé
- `SavingAccount` (compte d'épargne) possède un attribut `interestRate` qui représente le taux d'intérêt

Cette conception utilise l'héritage pour factoriser les propriétés communes tout en permettant des spécialisations par type de compte.

### La classe AccountOperation

Cette classe représente les opérations effectuées sur un compte:

```java
@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
public class AccountOperation {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Date operationDate;
    private double amount;
    @Enumerated(EnumType.STRING)
    private OperationType type;
    private String description;
    @ManyToOne
    private BankAccount bankAccount;
}
```

Cette classe permet de:
- Stocker l'historique des opérations (date, montant, description)
- Différencier les types d'opérations (DEBIT, CREDIT) via l'énumération OperationType
- Lier chaque opération à un compte bancaire spécifique

### Les énumérations

Pour compléter le modèle, deux énumérations sont définies:

```java
public enum AccountStatus {
    CREATED, ACTIVATED, SUSPENDED
}
```

```java
public enum OperationType {
    DEBIT, CREDIT
}
```

Ce modèle de données est conçu pour être à la fois flexible et performant, permettant de représenter les concepts essentiels d'une application bancaire tout en facilitant les opérations CRUD et les requêtes complexes.

## Couche de persistance (Repositories)

La couche de persistance est implémentée à l'aide de Spring Data JPA, qui simplifie considérablement l'accès aux données en fournissant des méthodes prédéfinies et en permettant la création de requêtes personnalisées de manière déclarative.

### CustomerRepository

Cette interface gère la persistance des clients:

```java
public interface CustomerRepository extends JpaRepository<Customer, Long> {
    // Méthodes de recherche personnalisées
}
```

L'extension de `JpaRepository<Customer, Long>` fournit automatiquement les opérations CRUD de base ainsi que les méthodes de pagination et de tri. Cette interface permet de:
- Rechercher un client par son ID
- Récupérer tous les clients
- Sauvegarder ou mettre à jour un client
- Supprimer un client

### BankAccountRepository

Cette interface gère la persistance des comptes bancaires:

```java
public interface BankAccountRepository extends JpaRepository<BankAccount, String> {
    // Méthodes spécifiques aux comptes bancaires
}
```

Elle permet de:
- Rechercher un compte par son ID (qui est une chaîne de caractères)
- Gérer tous les types de comptes (courants et d'épargne) grâce à l'héritage
- Effectuer des opérations de recherche avancées

### AccountOperationRepository

Cette interface gère la persistance des opérations bancaires:

```java
public interface AccountOperationRepository extends JpaRepository<AccountOperation, Long> {
    List<AccountOperation> findByBankAccountId(String accountId);
    Page<AccountOperation> findByBankAccountIdOrderByOperationDateDesc(String accountId, Pageable pageable);
}
```

Points importants:
- La méthode `findByBankAccountId` permet de récupérer toutes les opérations d'un compte spécifique
- La méthode `findByBankAccountIdOrderByOperationDateDesc` retourne les opérations paginées et triées par date décroissante

L'utilisation de Spring Data JPA dans cette couche offre plusieurs avantages:
- Réduction significative du code boilerplate
- Implémentation automatique des méthodes standard
- Possibilité de créer des requêtes complexes sans écrire de SQL
- Support intégré pour la pagination et le tri

## Couche service

La couche service contient la logique métier de l'application et sert d'intermédiaire entre les contrôleurs REST et les repositories. Elle est organisée autour d'interfaces et de leurs implémentations, suivant ainsi le principe d'inversion de dépendance (DIP).

### Interface BankAccountService

Cette interface définit toutes les opérations métier disponibles:

```java
public interface BankAccountService {
    CustomerDTO saveCustomer(CustomerDTO customerDTO);
    CurrentBankAccountDTO saveCurrentBankAccount(double initialBalance, double overDraft, Long customerId) throws CustomerNotFoundException;
    SavingBankAccountDTO saveSavingBankAccount(double initialBalance, double interestRate, Long customerId) throws CustomerNotFoundException;
    List<CustomerDTO> listCustomers();
    BankAccountDTO getBankAccount(String accountId) throws BankAccountNotFoundException;
    void debit(String accountId, double amount, String description) throws BankAccountNotFoundException, BalanceNotSufficientException;
    void credit(String accountId, double amount, String description) throws BankAccountNotFoundException;
    void transfer(String accountIdSource, String accountIdDestination, double amount) throws BankAccountNotFoundException, BalanceNotSufficientException;
    List<BankAccountDTO> bankAccountList();
    CustomerDTO getCustomer(Long customerId) throws CustomerNotFoundException;
    CustomerDTO updateCustomer(CustomerDTO customerDTO);
    void deleteCustomer(Long customerId);
    List<AccountOperationDTO> accountHistory(String accountId);
    AccountHistoryDTO getAccountHistory(String accountId, int page, int size) throws BankAccountNotFoundException;
}
```

Cette interface couvre toutes les fonctionnalités nécessaires pour:
- Gérer les clients (création, modification, suppression, consultation)
- Gérer les comptes bancaires (création de comptes courants et d'épargne)
- Effectuer des opérations bancaires (débit, crédit, transfert)
- Consulter l'historique des opérations avec support de pagination

### Implémentation BankAccountServiceImpl

L'implémentation du service:
- Utilise les repositories pour accéder aux données
- Applique les règles métier (par exemple, vérifier le solde avant un débit)
- Gère les conversions entre entités et DTOs via les mappers
- Lance des exceptions spécifiques en cas d'erreur

Par exemple, pour la méthode de débit:

```java
@Override
public void debit(String accountId, double amount, String description) throws BankAccountNotFoundException, BalanceNotSufficientException {
    BankAccount bankAccount = bankAccountRepository.findById(accountId)
            .orElseThrow(() -> new BankAccountNotFoundException("Bank account not found"));
    if (bankAccount.getBalance() < amount) {
        throw new BalanceNotSufficientException("Balance not sufficient");
    }
    AccountOperation accountOperation = new AccountOperation();
    accountOperation.setType(OperationType.DEBIT);
    accountOperation.setAmount(amount);
    accountOperation.setDescription(description);
    accountOperation.setOperationDate(new Date());
    accountOperation.setBankAccount(bankAccount);
    accountOperationRepository.save(accountOperation);
    bankAccount.setBalance(bankAccount.getBalance() - amount);
    bankAccountRepository.save(bankAccount);
}
```

Cette implémentation:
1. Vérifie l'existence du compte
2. Vérifie si le solde est suffisant
3. Crée une nouvelle opération de type DEBIT
4. Met à jour le solde du compte

La méthode de transfert est particulièrement intéressante car elle combine deux opérations:

```java
@Override
public void transfer(String accountIdSource, String accountIdDestination, double amount) throws BankAccountNotFoundException, BalanceNotSufficientException {
    debit(accountIdSource, amount, "Transfer to " + accountIdDestination);
    credit(accountIdDestination, amount, "Transfer from " + accountIdSource);
}
```

Cette approche garantit l'atomicité de l'opération de transfert et réutilise les méthodes existantes pour maintenir la cohérence du code.

## Mappers et DTOs

Le pattern DTO (Data Transfer Object) est utilisé pour séparer les entités persistantes des objets échangés avec les clients de l'API. Ce pattern offre plusieurs avantages:
- Protection des entités contre les modifications non autorisées
- Contrôle précis des données exposées
- Réduction du couplage entre les couches
- Adaptation de la structure des données aux besoins spécifiques des clients

### Les DTOs

Voici les principaux DTOs utilisés dans l'application:

#### CustomerDTO

```java
@Data
public class CustomerDTO {
    private Long id;
    private String name;
    private String email;
}
```

#### BankAccountDTO et ses sous-classes

```java
@Data
public class BankAccountDTO {
    private String id;
    private double balance;
    private Date createdAt;
    private AccountStatus status;
    private CustomerDTO customerDTO;
    private String type;
}

@Data @EqualsAndHashCode(callSuper = true)
public class CurrentBankAccountDTO extends BankAccountDTO {
    private double overDraft;
}

@Data @EqualsAndHashCode(callSuper = true)
public class SavingBankAccountDTO extends BankAccountDTO {
    private double interestRate;
}
```

#### AccountOperationDTO et AccountHistoryDTO

```java
@Data
public class AccountOperationDTO {
    private Long id;
    private Date operationDate;
    private double amount;
    private OperationType type;
    private String description;
}

@Data
public class AccountHistoryDTO {
    private String accountId;
    private double balance;
    private int currentPage;
    private int totalPages;
    private int pageSize;
    private List<AccountOperationDTO> accountOperationDTOS;
}
```

### Les Mappers

Les mappers sont responsables de la conversion entre les entités et les DTOs. Dans ce projet, ils sont implémentés à l'aide de méthodes dédiées ou potentiellement avec des outils comme MapStruct.

Exemple de code de mapper:

```java
@Service
public class BankAccountMapperImpl {
    public CustomerDTO fromCustomer(Customer customer) {
        CustomerDTO customerDTO = new CustomerDTO();
        customerDTO.setId(customer.getId());
        customerDTO.setName(customer.getName());
        customerDTO.setEmail(customer.getEmail());
        return customerDTO;
    }

    public Customer fromCustomerDTO(CustomerDTO customerDTO) {
        Customer customer = new Customer();
        customer.setId(customerDTO.getId());
        customer.setName(customerDTO.getName());
        customer.setEmail(customerDTO.getEmail());
        return customer;
    }

    public SavingBankAccountDTO fromSavingBankAccount(SavingAccount savingAccount) {
        SavingBankAccountDTO savingBankAccountDTO = new SavingBankAccountDTO();
        savingBankAccountDTO.setId(savingAccount.getId());
        savingBankAccountDTO.setBalance(savingAccount.getBalance());
        savingBankAccountDTO.setCreatedAt(savingAccount.getCreatedAt());
        savingBankAccountDTO.setStatus(savingAccount.getStatus());
        savingBankAccountDTO.setCustomerDTO(fromCustomer(savingAccount.getCustomer()));
        savingBankAccountDTO.setInterestRate(savingAccount.getInterestRate());
        savingBankAccountDTO.setType("SavingAccount");
        return savingBankAccountDTO;
    }

    // Autres méthodes de mappage...
}
```

L'utilisation des mappers centralise la logique de conversion et facilite la maintenance. Si les structures des entités ou des DTOs changent, seules les méthodes de mappage correspondantes doivent être modifiées.

## Contrôleurs REST

Les contrôleurs REST exposent les fonctionnalités de l'application via des endpoints HTTP. Ils sont le point d'entrée de l'API et utilisent les services pour exécuter les opérations métier.

### CustomerRestController

Ce contrôleur gère les opérations liées aux clients:

```java
@RestController
@AllArgsConstructor
@Slf4j
public class CustomerRestController {
    private BankAccountService bankAccountService;

    @GetMapping("/customers")
    public List<CustomerDTO> customers() {
        return bankAccountService.listCustomers();
    }

    @GetMapping("/customers/{id}")
    public CustomerDTO getCustomer(@PathVariable(name = "id") Long customerId) throws CustomerNotFoundException {
        return bankAccountService.getCustomer(customerId);
    }

    @PostMapping("/customers")
    public CustomerDTO saveCustomer(@RequestBody CustomerDTO customerDTO) {
        return bankAccountService.saveCustomer(customerDTO);
    }

    @PutMapping("/customers/{customerId}")
    public CustomerDTO updateCustomer(@PathVariable Long customerId, @RequestBody CustomerDTO customerDTO) {
        customerDTO.setId(customerId);
        return bankAccountService.updateCustomer(customerDTO);
    }

    @DeleteMapping("/customers/{id}")
    public void deleteCustomer(@PathVariable Long id) {
        bankAccountService.deleteCustomer(id);
    }
}
```

Points importants:
- Utilisation des annotations @GetMapping, @PostMapping, @PutMapping et @DeleteMapping pour définir les méthodes HTTP
- Annotation @RequestBody pour désérialiser le corps de la requête
- Annotation @PathVariable pour extraire les variables du chemin URL
- Injection du service via le constructeur (facilité par @AllArgsConstructor)
- Logging avec SLF4J (@Slf4j)

### BankAccountRestAPI

Ce contrôleur gère les opérations liées aux comptes bancaires:

```java
@RestController
@AllArgsConstructor
public class BankAccountRestAPI {
    private BankAccountService bankAccountService;

    @GetMapping("/accounts/{accountId}")
    public BankAccountDTO getBankAccount(@PathVariable String accountId) throws BankAccountNotFoundException {
        return bankAccountService.getBankAccount(accountId);
    }
    
    @GetMapping("/accounts")
    public List<BankAccountDTO> listAccounts(){
        return bankAccountService.bankAccountList();
    }
    
    @GetMapping("/accounts/{accountId}/operations")
    public List<AccountOperationDTO> getHistory(@PathVariable String accountId){
        return bankAccountService.accountHistory(accountId);
    }

    @GetMapping("/accounts/{accountId}/pageOperations")
    public AccountHistoryDTO getAccountHistory(
            @PathVariable String accountId,
            @RequestParam(name="page",defaultValue = "0") int page,
            @RequestParam(name="size",defaultValue = "5")int size) throws BankAccountNotFoundException {
        return bankAccountService.getAccountHistory(accountId,page,size);
    }
}
```

Fonctionnalités:
- Récupération des informations d'un compte spécifique
- Liste de tous les comptes
- Historique des opérations d'un compte
- Historique paginé des opérations avec gestion des paramètres de requête (@RequestParam)

Ces contrôleurs adhèrent aux principes REST:
- Utilisation des méthodes HTTP standard (GET, POST, PUT, DELETE)
- Manipulation des ressources via des URL significatives
- Communication sans état
- Format d'échange standard (JSON)

## Configuration et dépendances

Le projet Digital Banking s'appuie sur Spring Boot et diverses dépendances pour faciliter le développement. Voici les principales configurations et dépendances utilisées.

### Configuration Maven (pom.xml)

Le fichier pom.xml définit toutes les dépendances du projet:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <scope>runtime</scope>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.mariadb.jdbc</groupId>
        <artifactId>mariadb-java-client</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>2.2.0</version>
    </dependency>
</dependencies>
```

Principales dépendances:
- **Spring Data JPA**: Pour la persistance des données et l'ORM
- **Spring Web**: Pour créer des applications web et des API REST
- **Spring DevTools**: Pour le rechargement automatique pendant le développement
- **MariaDB**: Pilote JDBC pour la connexion à la base de données MariaDB
- **Lombok**: Pour réduire le code boilerplate (getters, setters, constructeurs)
- **Spring Boot Test**: Pour les tests unitaires et d'intégration
- **SpringDoc OpenAPI**: Pour la documentation automatique de l'API REST avec Swagger

### Configuration d'application (application.properties)

Le fichier application.properties contient les paramètres de configuration de l'application:

```properties
spring.datasource.url=jdbc:mariadb://localhost:3306/bank
spring.datasource.username=root
spring.datasource.password=
spring.datasource.driver-class-name=org.mariadb.jdbc.Driver
spring.jpa.hibernate.ddl-auto=create
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MariaDBDialect
spring.jpa.show-sql=true
server.port=8085
```

Cette configuration:
- Définit les paramètres de connexion à la base de données MariaDB
- Configure Hibernate pour créer automatiquement le schéma de base de données
- Active l'affichage des requêtes SQL pour le débogage
- Définit le port du serveur sur 8085

### Point d'entrée de l'application

La classe principale de l'application initialise Spring Boot et peut contenir des configurations supplémentaires:

```java
@SpringBootApplication
public class DigitalBankingApplication {
    public static void main(String[] args) {
        SpringApplication.run(DigitalBankingApplication.class, args);
    }
    
    // Beans de configuration supplémentaires si nécessaire
}
```

Cette architecture de configuration facilite:
- Le déploiement dans différents environnements (développement, test, production)
- La modification des paramètres sans changer le code
- L'intégration avec d'autres systèmes et services

## Conclusion

Le projet Digital Banking est une application backend robuste qui implémente les fonctionnalités essentielles d'une banque numérique. Cette première partie du projet a permis de mettre en place:

1. Une architecture en couches bien structurée qui suit les bonnes pratiques de développement Java et Spring Boot
2. Un modèle de données complet avec des relations bien définies entre les entités
3. Une API REST complète permettant la gestion des clients, des comptes et des opérations bancaires
4. Une séparation claire entre les données internes (entités) et les données exposées (DTOs)
5. Un système de gestion d'erreurs avec des exceptions métier spécifiques

Les principes architecturaux suivis dans ce projet offrent plusieurs avantages:
- Maintenabilité: organisation claire du code facilitant les évolutions futures
- Testabilité: séparation des responsabilités permettant des tests unitaires ciblés
- Évolutivité: possibilité d'ajouter de nouvelles fonctionnalités sans impacter les fonctionnalités existantes
- Sécurité: contrôle des données exposées via les DTOs

Ce backend constitue une base solide pour la suite du projet qui intégrera une interface utilisateur. La prochaine étape consistera à développer un frontend moderne qui communiquera avec cette API pour offrir une expérience utilisateur complète et intuitive.

Les perspectives d'amélioration incluent:
- L'ajout d'un système d'authentification et d'autorisation
- L'implémentation de la validation des données
- L'intégration de tests automatisés plus complets
- L'optimisation des performances pour une meilleure scalabilité

Ce rapport a présenté en détail la conception et l'implémentation de la partie backend du projet Digital Banking, démontrant l'application des concepts et des technologies modernes d'architecture JEE et de systèmes distribués.