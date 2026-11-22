# AWD – Plateforme de recrutement en microservices

Atelier **Architecture Web Distribuée (4SSA7)** – Année universitaire 2025-2026.

Ce dépôt contient la migration d'une application Spring Boot **monolithique** (candidats, jobs, candidatures) vers une architecture **microservices** avec un serveur de découverte **Eureka** et une **API Gateway**.

---

## Architecture

```
                      ┌────────────────────┐
   Client  ─────────► │   API Gateway      │  port 8085
 (navigateur,         │ (Spring Cloud      │
  Postman, Swagger)   │  Gateway)          │
                      └─────────┬──────────┘
                                │  lb://nom-du-service
                      ┌─────────▼──────────┐
                      │   Eureka Server    │  port 8761
                      │ (annuaire services)│
                      └─────────┬──────────┘
              ┌─────────────────┼──────────────────┐
              ▼                 ▼                  ▼
   ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐
   │ candidat-service  │ │ candidat-service  │ │   job-service     │
   │ instance 1 : 8081 │ │ instance 2 : 8083 │ │     port 8082     │
   │ Base H2 (mémoire) │ │ Base H2 (mémoire) │ │   Base MySQL      │
   └───────────────────┘ └───────────────────┘ └───────────────────┘
```

## Contenu du dépôt

| Dossier | Rôle | Port | Base de données |
|---|---|---|---|
| `eureka-server` | Serveur de découverte (Eureka) | 8761 | – |
| `candidat-service` | Gestion des candidats et adresses | 8081 (et 8083 pour la 2e instance) | H2 (en mémoire) |
| `job-service` | Gestion des jobs et catégories | 8082 | MySQL (`job_db`) |
| `api-gateway` | Point d'entrée unique, routage et load balancing | 8085 | – |

## Technologies

- Java 21
- Spring Boot 3.5.6 (Eureka Server : Spring Boot 4.0.x)
- Spring Cloud 2025.0.0 (Gateway, Eureka Client, LoadBalancer)
- Spring Data JPA, Bean Validation
- H2 et MySQL
- springdoc-openapi (Swagger UI)
- Maven

## Prérequis

- JDK 21
- Maven (ou le wrapper `mvnw` fourni dans chaque projet)
- MySQL démarré en local (par exemple avec XAMPP), utilisateur `root` sans mot de passe

La base `job_db` est créée automatiquement (`createDatabaseIfNotExist=true`).

## Lancer le projet

Démarrer les applications **dans cet ordre**, en attendant le message `Started ...` à chaque fois :

1. **MySQL**
2. **eureka-server** → http://localhost:8761
3. **job-service** (port 8082)
4. **candidat-service**, instance 1 (port 8081, port par défaut)
5. **candidat-service**, instance 2 (port 8083) : lancer avec l'argument `--server.port=8083`
6. **api-gateway** (port 8085)

Exemple en ligne de commande, depuis le dossier de chaque projet :

```bash
./mvnw spring-boot:run
# 2e instance de candidat-service :
./mvnw spring-boot:run "-Dspring-boot.run.arguments=--server.port=8083"
```

Dans Eureka (http://localhost:8761), on doit voir : `CANDIDAT-SERVICE` (2 instances), `JOB-SERVICE` (1 instance) et `GATEWAY` (1 instance).

## Documentation des API (Swagger)

| Service | URL |
|---|---|
| candidat-service (8081) | http://localhost:8081/swagger-ui.html |
| candidat-service (8083) | http://localhost:8083/swagger-ui.html |
| job-service | http://localhost:8082/swagger-ui.html |

La console H2 de `candidat-service` est disponible sur `http://localhost:8081/h2-console`
(JDBC URL : `jdbc:h2:mem:candidat_db`, utilisateur : `sa`, mot de passe vide).

## Routes de l'API Gateway

Toutes les requêtes passent par `http://localhost:8085`.

| Chemin | Service cible | Type de routage |
|---|---|---|
| `/api/candidates/**` | `candidat-service` | `lb://candidat-service` (dynamique, via Eureka) |
| `/api/jobs/**` | `job-service` | `lb://job-service` |
| `/api/categories/**` | `job-service` | `lb://job-service` |

Configuration (`api-gateway/src/main/resources/application.properties`) :

```properties
spring.application.name=gateway
server.port=8085

eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
eureka.client.register-with-eureka=true
eureka.client.fetch-registry=true

spring.cloud.gateway.routes[0].id=CANDIDAT
spring.cloud.gateway.routes[0].uri=lb://candidat-service
spring.cloud.gateway.routes[0].predicates[0]=Path=/api/candidates/**

spring.cloud.gateway.routes[1].id=JOB
spring.cloud.gateway.routes[1].uri=lb://job-service
spring.cloud.gateway.routes[1].predicates[0]=Path=/api/jobs/**,/api/categories/**

logging.level.org.springframework.cloud.gateway=DEBUG
logging.level.reactor.netty.http.client=DEBUG
```

Approche **statique** (partie 2 du TP) : il suffit de remplacer `lb://candidat-service` par `http://localhost:8081` et `lb://job-service` par `http://localhost:8082`. Dans ce cas, Eureka n'est pas utilisé et la Gateway ne répartit plus la charge.

## Load balancing

### Question 2.b – Algorithme utilisé par défaut

Le filtre `ReactiveLoadBalancerClientFilter` de la Gateway choisit une instance parmi celles enregistrées dans Eureka. L'algorithme par défaut de Spring Cloud LoadBalancer est le **Round Robin** (`RoundRobinLoadBalancer`).

Test : chaque instance de `candidat-service` possède sa propre base H2 (un candidat « Ali » créé sur 8081, un candidat « Sara » créé sur 8083). Les rechargements successifs de `http://localhost:8085/api/candidates` renvoient alternativement Sara, Ali, Sara, Ali… et les logs de la Gateway montrent le port de destination qui alterne entre 8083 et 8081.

### Question 3 – Autres algorithmes et test de Random

Algorithmes existants dans Spring Cloud LoadBalancer :

| Algorithme | Principe |
|---|---|
| Round Robin (par défaut) | Les instances sont utilisées à tour de rôle |
| Random | Une instance est choisie au hasard |
| Weighted | Choix selon un poids associé à chaque instance |
| Zone Preference | Préférence pour les instances de la même zone |
| Same Instance Preference | Reste sur la même instance tant qu'elle est disponible |
| Hint-based | Choix selon un indice transmis dans l'en-tête de la requête |

Pour tester **Random** uniquement sur `candidat-service`, une classe de configuration (sans `@Configuration`) est créée dans la Gateway :

```java
public class RandomLoadBalancerConfig {

    @Bean
    ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(Environment environment,
                                                            LoadBalancerClientFactory factory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
                factory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }
}
```

puis elle est appliquée au service concerné dans la classe principale :

```java
@SpringBootApplication
@EnableDiscoveryClient
@LoadBalancerClient(name = "candidat-service", configuration = RandomLoadBalancerConfig.class)
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}
```

Résultat observé après redémarrage de la Gateway : les réponses de `http://localhost:8085/api/candidates` ne sont plus alternées régulièrement, mais aléatoires (par exemple : Sara, Sara, Sara, Ali, Sara, Ali, Ali). `job-service` reste en Round Robin.

## Statique ou dynamique ?

| Approche | Utilise Eureka | Avantages | Inconvénients |
|---|---|---|---|
| Statique (`http://localhost:8081`) | Non | Simple et rapide, fonctionne sans Eureka | Si le port ou l'adresse change, la Gateway ne trouve plus le service ; pas de load balancing |
| Dynamique (`lb://candidat-service`) | Oui | Pas besoin de connaître l'URL exacte, load balancing et scalabilité automatiques | Dépendance à Eureka, délai de découverte au démarrage |

## Remarques

- Chaque instance de `candidat-service` utilise sa propre base H2 en mémoire : les données ne sont pas partagées entre les instances et sont perdues à l'arrêt.
- Le mode « self-preservation » d'Eureka est désactivé (`eureka.server.enable-self-preservation=false`) pour retirer rapidement du registre les instances arrêtées. Le message d'avertissement rouge du tableau de bord est normal en local.
- Le package `candidature` du monolithe n'a pas été migré : il ne faisait pas partie du périmètre de cet atelier.

## Auteur

Dépôt GitHub : `sayarinessrine`
