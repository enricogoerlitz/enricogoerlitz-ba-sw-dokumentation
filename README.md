# Software-Dokumentation zur Bachelorarbeit

## Informationen zur Bachelorarbeit

| Feld | Wert |
|------|------|
| **Titel** | Evaluierung und Entwicklung einer Azure-Cloud-Architektur für eine containerisierte Microservice-Anwendung |
| **Studiengang** | Wirtschaftsinformatik |
| **Fachbereich** | 4 |
| **Vorgelegt von** | Enrico Goerlitz |
| **Datum** | Berlin, 01.02.2026 |
| **Erstgutachter** | Prof. Dr. Alexander Stanik |
| **Zweitgutachter** | Prof. Dr. Arif Wider |

---

## Projektübersicht

Diese Dokumentation beschreibt die technische Umsetzung einer Azure-Cloud-Architektur für die Open-Source-Anwendung **REMSFAL**, eine Facility-Management-Software, die als containerisierte Microservice-Anwendung realisiert wurde.

Die Anwendung besteht aus folgenden Services:

| Service | Beschreibung | Repository |
|---------|--------------|------------|
| **Platform-Service** | Haupt-Backend für Benutzerverwaltung, Authentifizierung und CRUD-Operationen | [remsfal-backend](https://github.com/enricogoerlitz/remsfal-backend/tree/Enrico-Goerlitz%23644) |
| **Ticketing-Service** | Dokumentenspeicherung und Ticket-Funktionalität mit Cassandra-Backend | [remsfal-backend](https://github.com/enricogoerlitz/remsfal-backend/tree/Enrico-Goerlitz%23644) |
| **Notification-Service** | E-Mail- und Benachrichtigungsdienst via Kafka-Messaging | [remsfal-backend](https://github.com/enricogoerlitz/remsfal-backend/tree/Enrico-Goerlitz%23644) |
| **OCR-Service** | Dokumenten-OCR-Verarbeitung mittels Kafka-Consumer | [remsfal-ocr](https://github.com/enricogoerlitz/remsfal-ocr/tree/Enrico-Goerlitz%2345) |
| **Frontend-Service** | Vue.js Single-Page-Application | [remsfal-frontend](https://github.com/enricogoerlitz/remsfal-frontend/tree/Enrico-Goerlitz%23828) |

---

## Dokumentationsübersicht

Diese Software-Dokumentation gliedert sich in folgende Bereiche:

### 📦 [Infrastructure as Code (IAC.md)](./IAC.md)

Detaillierte Dokumentation der Azure-Infrastruktur, die mittels Terraform provisioniert wird. Enthält:

- Projektstruktur und verwendete Versionen
- Erläuterung aller Azure-Ressourcen und deren Konfiguration
- Naming Conventions und Tagging-Strategien
- Link zum [GitHub-Repository](https://github.com/enricogoerlitz/remsfal-backend/tree/Enrico-Goerlitz%23644/iac/azure/terraform)

### 🔧 [Code-Refactoring (REFACTORING.md)](./REFACTORING.md)

Dokumentation aller Code-Änderungen, die für die Azure-Migration notwendig waren:

- remsfal-core: org.eclipse.microprofile.config.spi.ConfigSource
- Platform-Service: PostgreSQL-Integration, Key Vault Config Source
- Ticketing-Service: Cosmos DB Cassandra API, Azure Blob Storage
- Notification-Service: Azure Event Hubs Integration
- OCR-Service: Multi-Provider-Architektur für Kafka und Storage
- Frontend-Service: Nginx-Reverse-Proxy-Konfiguration

### 🚀 [Setup-Anleitung (SETUP.md)](./SETUP.md)

Schritt-für-Schritt-Anleitung für das Deployment der Anwendung:

- Terraform-Infrastruktur-Deployment
- Docker-Image-Builds und Push zur Azure Container Registry
- Google OAuth-Konfiguration für den Login-Prozess

---

## Repositories

| Repository | Services | Branch | Link |
|------------|----------|--------|------|
| **remsfal-backend** | CORE, Platform, Ticketing, Notification, IaC | `Enrico-Goerlitz#644` | [GitHub](https://github.com/enricogoerlitz/remsfal-backend/tree/Enrico-Goerlitz%23644) |
| **remsfal-frontend** | Frontend | `Enrico-Goerlitz#828` | [GitHub](https://github.com/enricogoerlitz/remsfal-frontend/tree/Enrico-Goerlitz%23828) |
| **remsfal-ocr** | OCR | `Enrico-Goerlitz#45` | [GitHub](https://github.com/enricogoerlitz/remsfal-ocr/tree/Enrico-Goerlitz%2345) |
