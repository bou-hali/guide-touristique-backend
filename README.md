# 🏛️ Guide Touristique - Microservices Backend Platform

[![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.x-6DB33F?logo=springboot&logoColor=white)](https://spring.io/projects/spring-boot)
[![Java](https://img.shields.io/badge/Java-17-ED8B00?logo=openjdk&logoColor=white)](https://openjdk.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![Keycloak](https://img.shields.io/badge/Security-Keycloak_OAuth2-008B8B?logo=keycloak&logoColor=white)](https://www.keycloak.org/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

Plateforme backend moderne basée sur une architecture en **microservices** pour la gestion touristique : découverte de circuits, profils de guides locaux certifiés, gestion des réservations et système d'avis vérifiés, sécurisés par jetons OAuth2/JWT (Keycloak).

---

## 📐 Architecture du Système

Le système est découpé en services autonomes disposant chacun de leur propre schéma de base de données (pattern *Database-per-Service*), orchestrés par Docker Compose :

```mermaid
flowchart TD
    Client([Client Web / Frontend]) -->|OAuth2 / JWT| Gateway[Reverse Proxy / API Consumer]

    subgraph Core Microservices
        direction TB
        GuideSvc["👤 guide-service (Port 8081)<br/>Gestion des guides touristiques"]
        CircuitSvc["🗺️ circuit-service (Port 8082)<br/>Parcours, étapes et activités"]
        ResaSvc["📅 reservation-service (Port 8083)<br/>Réservations et créneaux"]
        AvisSvc["⭐ avis-service (Port 8084)<br/>Notes, commentaires & validation"]
    end

    Gateway --> GuideSvc
    Gateway --> CircuitSvc
    Gateway --> ResaSvc
    Gateway --> AvisSvc

    AvisSvc -.->|REST / Feign| ResaSvc
    GuideSvc -.->|REST / WebClient| CircuitSvc
    ResaSvc -.->|REST / WebClient| GuideSvc
    ResaSvc -.->|REST / WebClient| CircuitSvc

    subgraph Infrastructure
        Postgres[(PostgreSQL 16 Multi-DB)]
        Keycloak[🔐 Keycloak IAM / JWT Auth (Externe)]
    end

    GuideSvc -->|guide schema| Postgres
    CircuitSvc -->|circuit schema| Postgres
    ResaSvc -->|reservation schema| Postgres
    AvisSvc -->|avis schema| Postgres
    GuideSvc & CircuitSvc & ResaSvc & AvisSvc -->|Token Verification| Keycloak
```

---

## 🚀 Les Microservices

| Service | Port | Description | Dépendances |
| :--- | :--- | :--- | :--- |
| **`guide-service`** | `8081` | Profils de guides, langues, spécialités, circuits assignés | PostgreSQL, Keycloak |
| **`circuit-service`** | `8082` | Circuits touristiques, monuments, villes, étapes et tarifs | PostgreSQL, Keycloak |
| **`reservation-service`** | `8083` | Réservations de circuits avec guides, statuts et validation | PostgreSQL, Keycloak, Guide/Circuit |
| **`avis-service`** | `8084` | Notes et commentaires vérifiés basés sur l'éligibilité de réservation | PostgreSQL, Keycloak, Reservation |

---

## 🔐 Authentification & Rôles Keycloak

Les microservices agissent en tant que **Resource Servers OAuth2** et valident la signature des jetons JWT émis par un serveur Keycloak externe.

* **Serveur Keycloak** : Externe (doit être accessible sur le port `8180` avec le realm `guide-touristique`).
* **Résolution réseau conteneurs** : `extra_hosts` mappe `host.docker.internal` vers l'hôte pour la validation des clés JWK (`/protocol/openid-connect/certs`).
* **Rôles attendus dans les jetons JWT** (`realm_access.roles`) :
  * `ADMIN` : administration complète (création/mise à jour/suppression des circuits et guides, modération des avis).
  * `GUIDE` : consultation et mise à jour de son profil guide, gestion des créneaux et des réservations assignées.
  * `USER` (ou `TOURISTE`) : création et suivi de réservations, dépôt d'avis vérifié après réalisation d'un circuit.
* **Claims requis** : `sub` (identifiant utilisateur), `name` (nom affiché de l'auteur).

---

## 📡 Endpoints Principaux de l'API REST

### 1. `guide-service` (Port 8081)
| Méthode | Endpoint | Description | Accès / Rôle |
| :--- | :--- | :--- | :--- |
| `GET` | `/api/guides` | Recherche de guides (filtrage par ville, langue) | Public |
| `GET` | `/api/guides/{id}` | Détails d'un guide | Public |
| `GET` | `/api/guides/me` | Profil du guide connecté | Guide authentifié |
| `GET` | `/api/guides/me/circuits` | Circuits assignés au guide | Guide authentifié |
| `POST` | `/api/guides` | Création d'un profil guide | `ROLE_ADMIN` |
| `PUT` | `/api/guides/{id}` | Mise à jour d'un profil guide | `ROLE_ADMIN` / Guide |
| `DELETE`| `/api/guides/{id}` | Archivage d'un profil guide | `ROLE_ADMIN` |

### 2. `circuit-service` (Port 8082)
| Méthode | Endpoint | Description | Accès / Rôle |
| :--- | :--- | :--- | :--- |
| `GET` | `/api/circuits` | Liste des circuits touristiques | Public |
| `GET` | `/api/circuits/{id}` | Détails d'un circuit avec étapes | Public |
| `POST` | `/api/circuits` | Création d'un nouveau circuit | `ROLE_ADMIN` |
| `PUT` | `/api/circuits/{id}` | Mise à jour d'un circuit | `ROLE_ADMIN` |
| `DELETE`| `/api/circuits/{id}` | Suppression d'un circuit | `ROLE_ADMIN` |

### 3. `reservation-service` (Port 8083)
| Méthode | Endpoint | Description | Accès / Rôle |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/reservations` | Créer une réservation pour un circuit | Touriste authentifié |
| `GET` | `/api/reservations/me` | Liste de mes réservations | Touriste authentifié |
| `GET` | `/api/reservations/guide` | Réservations affectées au guide | Guide authentifié |
| `GET` | `/api/reservations/{id}` | Détails d'une réservation | Touriste propriétaire / Guide |
| `GET` | `/api/reservations/{id}/eligibilite` | Vérification d'éligibilité pour dépôt d'avis | Inter-service / Avis |

### 4. `avis-service` (Port 8084)
| Méthode | Endpoint | Description | Accès / Rôle |
| :--- | :--- | :--- | :--- |
| `GET` | `/api/avis/circuit/{circuitId}` | Liste des avis d'un circuit | Public |
| `POST` | `/api/avis` | Déposer un avis après vérification d'éligibilité | `ROLE_TOURISTE` |
| `GET` | `/api/avis/guide/{guideId}/rating` | Note moyenne et métriques d'un guide | Public |
| `DELETE`| `/api/avis/{id}` | Modération d'un avis | `ROLE_ADMIN` |

---

## 🛠️ Stack Technique

- **Langage & Framework** : Java 17, Spring Boot 3 (Spring Web, Spring Data JPA, Spring Security, Flyway).
- **Base de données** : PostgreSQL 16 (Schémas isolés : `guide`, `circuit`, `reservation`, `avis`).
- **Sécurité** : Keycloak (OpenID Connect / OAuth2 Resource Server avec validation JWT et RBAC).
- **Conteneurisation** : Docker, Docker Compose (fichiers Dockerfile multi-stage).
- **Tests** : JUnit 5, Mockito, Spring Boot Test, Testcontainers.

---

## 📋 Prérequis & Démarrage

### Prérequis
- [Docker](https://docs.docker.com/get-docker/) & Docker Compose
- [Keycloak 26+](https://www.keycloak.org/) lancé sur le port 8180 (realm `guide-touristique`)
- [JDK 17+](https://adoptium.net/) & Maven (si exécution locale hors conteneur)

### Démarrage avec Docker Compose

1. **Cloner le dépôt** :
   ```bash
   git clone https://github.com/bou-hali/guide-touristique-backend.git
   cd guide-touristique-backend
   ```

2. **Configurer l'environnement** :
   ```bash
   cp .env.example .env
   ```

3. **Lancer les conteneurs** :
   ```bash
   docker compose up --build -d
   ```

---

## 🔒 Sécurité & Bonnes Pratiques

- **Zéro Secret en Clair** : Aucun mot de passe, clé ou token n'est commité dans le dépôt.
- Les identifiants sont strictement passés par variables d'environnement (`.env` exclu par `.gitignore`).
- Les échanges inter-services sont sécurisés et protégés par jetons porteurs (Bearer JWT).

---

## 👤 Auteur

- **Hajar El Bouhali** - [@bou-hali](https://github.com/bou-hali)
