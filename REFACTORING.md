

# Code-Refactoring - Azure-Migration

Diese Dokumentation beschreibt alle Code-Änderungen, die für die Migration der REMSFAL-Anwendung auf Azure durchgeführt wurden.

---

## Inhaltsverzeichnis

1. [Übersicht der Änderungen](#übersicht-der-änderungen)
2. [remsfal-core (Shared Library)](#remsfal-core-shared-library)
3. [Platform-Service](#platform-service)
4. [Ticketing-Service](#ticketing-service)
5. [Notification-Service](#notification-service)
6. [OCR-Service](#ocr-service)
7. [Frontend-Service](#frontend-service)
8. [Zusammenfassung](#zusammenfassung)

---

## Übersicht der Änderungen

### Migrations-Matrix

| Original (Lokal/Docker) | Azure-Äquivalent | Betroffene Services |
|-------------------------|------------------|---------------------|
| PostgreSQL (Docker) | PostgreSQL Flexible Server | Platform |
| Apache Cassandra | Cosmos DB (Cassandra API) | Ticketing |
| MinIO (S3-kompatibel) | Azure Blob Storage | Ticketing, OCR |
| Apache Kafka | Azure Event Hubs | Ticketing, Notification, OCR |
| Lokale Secrets | Azure Key Vault | Alle Backend-Services |

### Änderungsübersicht nach Service

| Service | Neue Dateien | Modifizierte Dateien |
|---------|--------------|---------------------|
| **remsfal-core** | 2 | 1 |
| **Platform** | 0 | 2 |
| **Ticketing** | 4 | 2 |
| **Notification** | 0 | 1 |
| **OCR** | 9 | 2 |
| **Frontend** | 1 | 2 |

---

## remsfal-core (Shared Library)

Die Core-Library wurde um eine zentrale Azure Key Vault Integration erweitert, die von allen Backend-Services genutzt wird.

### Neue Dateien

| Datei | Beschreibung |
|-------|--------------|
| `src/main/java/de/remsfal/common/config/AzureKeyVaultConfigSource.java` | MicroProfile ConfigSource für Azure Key Vault Integration |
| `src/main/resources/META-INF/services/org.eclipse.microprofile.config.spi.ConfigSource` | SPI-Registrierung für den ConfigSource |

### Modifizierte Dateien

| Datei | Änderung |
|-------|----------|
| `pom.xml` | Neue Dependencies: `azure-identity` (1.12.0), `azure-security-keyvault-secrets` (4.8.0) |

### AzureKeyVaultConfigSource - Implementierung

```java
public class AzureKeyVaultConfigSource implements ConfigSource {

    // Bekannte Secrets, die aus Key Vault geladen werden
    private boolean isKnownSecret(String propertyName) {
        return propertyName.equals("postgres-connection-string") ||
               propertyName.equals("eventhub-bootstrap-server") ||
               propertyName.equals("eventhub-connection-string") ||
               propertyName.equals("cosmos-contact-point") ||
               propertyName.equals("cosmos-username") ||
               propertyName.equals("cosmos-password") ||
               propertyName.equals("storage-connection-string");
    }
}
```

**Funktionsweise:**
- **Ordinal 270:** Höhere Priorität als `application.properties` (250)
- **Lazy Loading:** Secrets werden erst bei Bedarf geladen und gecacht
- **Dual Authentication:**
  - **Produktion:** Managed Identity via `DefaultAzureCredential`
  - **Entwicklung:** Ebenfalls über `DefaultAzureCredential`, aber auch: Client Secret via `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID`

**Konfiguration:**
```properties
# In application.properties oder als Environment Variable
azure.keyvault.endpoint=https://rmsfl-dev-weu-xxxx-kv.vault.azure.net/
# ODER
AZURE_KEYVAULT_ENDPOINT=https://rmsfl-dev-weu-xxxx-kv.vault.azure.net/
```

---

## Platform-Service

Der Platform-Service benötigte minimale Änderungen, da PostgreSQL bereits als Datenbank verwendet wurde. Nur der JDBC Connection String musste für Azure angepasst werden.

### Modifizierte Dateien

| Datei | Änderung |
|-------|----------|
| `src/main/resources/application.properties` | JDBC Connection String für Azure PostgreSQL Flexible Server |

### application.properties Änderungen

**Lokal (Docker):**
```properties
quarkus.datasource.db-kind=postgresql
%dev.quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/REMSFAL
```

**Azure (Key Vault):**
```properties
# Der Connection String wird automatisch aus Key Vault geladen
# über den AzureKeyVaultConfigSource mit dem Secret-Namen "postgres-connection-string"
quarkus.datasource.jdbc.url=${postgres-connection-string}
```

**Begründung:**
- PostgreSQL wurde bereits lokal verwendet
- Azure PostgreSQL Flexible Server ist vollständig kompatibel
- Nur der Connection String musste angepasst werden
- Liquibase-Migrationen funktionieren weiterhin ohne Anpassung

---

## Ticketing-Service

Der Ticketing-Service benötigte die umfangreichsten Änderungen für die Azure-Migration.

### Neue Dateien

| Datei | Beschreibung |
|-------|--------------|
| `src/main/java/de/remsfal/ticketing/entity/dao/storage/StorageClient.java` | Interface für Storage-Abstraktion |
| `src/main/java/de/remsfal/ticketing/entity/dao/storage/MinioStorageClient.java` | MinIO-Implementierung (lokale Entwicklung) |
| `src/main/java/de/remsfal/ticketing/entity/dao/storage/AzureBlobStorageClient.java` | Azure Blob Storage Implementierung |

### Modifizierte Dateien

| Datei | Änderung |
|-------|----------|
| `pom.xml` | Neue Dependency: `azure-storage-blob` (12.25.1) |
| `src/main/java/de/remsfal/ticketing/entity/dao/FileStorage.java` | Storage Provider Factory Pattern |
| `src/main/java/de/remsfal/ticketing/entity/CassandraExecutor.java` | SSL-Konfiguration für Cosmos DB, Environment-Detection |
| `src/main/resources/application.properties` | Azure-Konfigurationen |

### CassandraExecutor - Cosmos DB SSL-Anpassung

Der `CassandraExecutor` musste für Azure Cosmos DB angepasst werden, da Cosmos DB spezielle SSL-Anforderungen für den Connection-Aufbau hat.

**Problemstellung:**
- Cosmos DB Cassandra API erfordert SSL/TLS für alle Verbindungen
- Der Liquibase JDBC-Treiber für Cassandra unterstützt die speziellen SSL-Anforderungen von Cosmos DB nicht
- Schema-Migrationen über Liquibase schlagen daher bei Cosmos DB fehl

**Lösung:**
1. **Environment-Detection:** Der Executor erkennt automatisch, ob er mit lokalem Cassandra oder Cosmos DB verbunden ist
2. **SSL-Konfiguration:** Für Cosmos DB wird automatisch ein SSL-Context konfiguriert
3. **Schema-Migration nach Terraform:** Die Tabellenerstellung wurde von Liquibase nach Terraform (IaC) verschoben

```java
public void onStartup(@Observes StartupEvent event) {
    // Environment-Detection basierend auf Contact Points
    boolean isCosmosDb = contactPoints.contains("cosmos.azure.com");
    
    if (isCosmosDb) {
        // SSL für Cosmos DB aktivieren
        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(null, trustManagers, new SecureRandom());
        builder.withSslContext(sslContext);
    } else {
        // Kein SSL für lokales Cassandra
        builder.withSslContext(null);
    }
    
    if (migrateAtStart) {
        if (isCosmosDb) {
            // Liquibase JDBC unterstützt Cosmos DB SSL nicht
            // Schema wird über Terraform erstellt
            logger.info("Schema wird über Terraform verwaltet");
        } else {
            // Liquibase für lokales Cassandra
            processChangelogs(session);
        }
    }
}
```

**Terraform Schema-Erstellung:**
```hcl
# In main.tf - Cosmos DB Cassandra Tabellen
resource "azurerm_cosmosdb_cassandra_table" "defects" {
  name                  = "defects"
  cassandra_keyspace_id = azurerm_cosmosdb_cassandra_keyspace.main.id
  schema { ... }
}
```

### Storage Abstraction Pattern

```java
public interface StorageClient {
    void initialize() throws Exception;
    String uploadFile(InputStream inputStream, String fileName, MediaType contentType) throws Exception;
    InputStream downloadFile(String fileName) throws Exception;
    void deleteFile(String fileName) throws Exception;
    boolean fileExists(String fileName) throws Exception;
}
```

**FileStorage.java - Provider Selection:**
```java
@ApplicationScoped
public class FileStorage {
    
    @ConfigProperty(name = "storage.provider", defaultValue = "minio")
    String storageProvider;
    
    public void onStartup(@Observes StartupEvent event) {
        switch (storageProvider.toLowerCase()) {
            case "minio":
                storageClient = new MinioStorageClient(minioClient, bucketName, logger);
                break;
            case "azure":
                storageClient = new AzureBlobStorageClient(azureConnectionString.get(), bucketName, logger);
                break;
        }
    }
}
```

### application.properties Änderungen

```properties
# Key Vault Endpoint
azure.keyvault.endpoint=https://rmsfl-dev-weu-xxxx-kv.vault.azure.net/

# Storage Provider (minio | azure)
storage.provider=azure
%test.storage.provider=minio

# Azure Blob Storage - Connection String aus Key Vault
azure.storage.connection-string=${storage-connection-string}

# Cosmos DB Cassandra API
quarkus.cassandra.contact-points=${cosmos-contact-point}
quarkus.cassandra.auth.username=${cosmos-username}
quarkus.cassandra.auth.password=${cosmos-password}
quarkus.cassandra.local-datacenter=West Europe
quarkus.cassandra.request.timeout=30s

# Azure Event Hubs (Kafka-kompatibel)
mp.messaging.connector.smallrye-kafka.bootstrap.servers=${eventhub-bootstrap-server}
mp.messaging.connector.smallrye-kafka.security.protocol=SASL_SSL
mp.messaging.connector.smallrye-kafka.sasl.mechanism=PLAIN
mp.messaging.connector.smallrye-kafka.sasl.jaas.config=${eventhub-connection-string}
```

**Begründung für Cosmos DB Cassandra API:**
- Bestehende CQL-Queries und Datenmodelle bleiben unverändert
- Kein Code-Refactoring im Repository-Layer notwendig
- Erhöhtes Timeout (30s) wegen höherer Latenz im Vergleich zu lokalem Cassandra
- SSL-Anforderungen erfordern Anpassung des CassandraExecutors
- Schema-Migration über Terraform statt Liquibase (Liquibase JDBC unterstützt Cosmos DB SSL nicht)

---

## Notification-Service

Der Notification-Service benötigte nur Kafka-Konfigurationsänderungen.

### Modifizierte Dateien

| Datei | Änderung |
|-------|----------|
| `src/main/resources/application.properties` | Azure Event Hubs Konfiguration |

### application.properties Änderungen

```properties
# Azure Event Hubs (Kafka-kompatibel)
%dev.mp.messaging.connector.smallrye-kafka.bootstrap.servers=${eventhub-bootstrap-server}
%dev.mp.messaging.connector.smallrye-kafka.security.protocol=SASL_SSL
%dev.mp.messaging.connector.smallrye-kafka.sasl.mechanism=PLAIN
%dev.mp.messaging.connector.smallrye-kafka.sasl.jaas.config=${eventhub-connection-string}
```

**Begründung:**
- SmallRye Kafka unterstützt Event Hubs ohne Code-Änderungen
- Nur SASL_SSL-Konfiguration für sichere Verbindung notwendig

---

## OCR-Service

Der OCR-Service (Python) wurde mit einer vollständigen Multi-Provider-Architektur implementiert.

### Architektur-Übersicht

```
┌─────────────────────────────────────────────────────────────────┐
│                    Environment Variables                        │
│  KAFKA_PROVIDER, STORAGE_PROVIDER, SECRETS_PROVIDER            │
└─────────────────┬──────────────────────┬───────────────────────┘
                  │                      │
                  ▼                      ▼
┌─────────────────────────┐  ┌─────────────────────────────────┐
│   SecretsVaultFactory   │  │      KafkaConsumerFactory       │
│   ├─ LOCAL → ENV        │  │      ├─ LOCAL → Kafka           │
│   └─ AZURE_KEYVAULT     │  │      └─ AZURE → Event Hubs      │
└─────────────────────────┘  └─────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────┐
│   StorageClientFactory  │
│   ├─ LOCAL → MinIO      │
│   └─ AZURE → Blob Store │
└─────────────────────────┘
```

### Neue Dateien

| Datei | Beschreibung |
|-------|--------------|
| `src/core/kafka/client.py` | Kafka Consumer/Producer Factory mit LOCAL/AZURE Support |
| `src/core/storage/base.py` | Abstract Base Class für Storage |
| `src/core/storage/client.py` | Storage Factory |
| `src/core/vault/base.py` | Abstract Base Class für Secrets |
| `src/core/vault/client.py` | Secrets Vault Factory |
| `src/storage/minio/client.py` | MinIO Storage Implementation |
| `src/storage/storageaccount/client.py` | Azure Blob Storage Implementation |
| `src/vault/local/client.py` | Environment Variables Secrets Client |
| `src/vault/keyvault/client.py` | Azure Key Vault Client |

### Modifizierte Dateien

| Datei | Änderung |
|-------|----------|
| `src/kafka_consumer.py` | Nutzt `KafkaConsumerFactory` |
| `src/ocr_engine.py` | Nutzt `StorageClientFactory` |
| `pyproject.toml` | Neue Dependencies für Azure SDK |

### KafkaConsumerFactory Implementation

```python
class KafkaConsumerFactory:
    @staticmethod
    def create(topic: str, group_id: str, type: str = "LOCAL") -> KafkaConsumer:
        config = {
            "bootstrap_servers": [KAFKA_BROKER],
            "group_id": group_id,
            "value_deserializer": lambda v: json.loads(v.decode("utf-8")),
        }

        if type == "AZURE":
            sasl_password = secrets_client.get_secret("KAFKA_SASL_PASSWORD")
            config.update({
                "security_protocol": "SASL_SSL",
                "sasl_mechanism": "PLAIN",
                "sasl_plain_username": "$ConnectionString",
                "sasl_plain_password": sasl_password,
            })
        
        return KafkaConsumer(topic, **config)
```

### Environment Variables für Azure

```bash
# Provider-Auswahl
KAFKA_PROVIDER=AZURE
STORAGE_PROVIDER=AZURE
SECRETS_PROVIDER=AZURE_KEYVAULT

# Azure Key Vault
KEYVAULT_URL=https://rmsfl-dev-weu-xxxx-kv.vault.azure.net/

# Kafka Topics
KAFKA_TOPIC_IN=ocr.documents.to_process
KAFKA_TOPIC_OUT=ocr.documents.processed
KAFKA_GROUP_ID=ocr-service
```

### Secret Mapping (Key Vault → Environment)

| Code-Name | Key Vault Secret |
|-----------|------------------|
| `STORAGE_CONNECTION_STRING` | `storage-connection-string` |
| `KAFKA_SASL_PASSWORD` | `eventhub-sasl-password` |
| `KAFKA_BROKER` | `eventhub-bootstrap-server` |

**Begründung für Multi-Provider-Architektur:**
- Lokale Entwicklung bleibt mit Docker Compose möglich
- Keine Azure-Abhängigkeiten im Development-Modus
- Einfaches Umschalten über Environment Variables

---

## Frontend-Service

Der Frontend-Service wurde mit einem Nginx Reverse Proxy für die Backend-Kommunikation erweitert.

### Neue Dateien

| Datei | Beschreibung |
|-------|--------------|
| `docker/nginx/templates/default.conf.template` | Nginx Template mit Environment Variables |

### Modifizierte Dateien

| Datei | Änderung |
|-------|----------|
| `docker/Dockerfile` | Template-Verarbeitung, rootless nginx |
| `docker/nginx/nginx.conf` | Basis-Konfiguration für tmp-Verzeichnisse |

### Nginx Template (default.conf.template)

```nginx
server {
    listen 80;
    server_name localhost;

    # Static Frontend Files
    location / {
        root   /usr/share/nginx/html;
        index  index.html;
        try_files $uri $uri/ /index.html;
    }

    # Proxy für Ticketing Service (spezifischer Match zuerst)
    location /api/v1/issues {
        proxy_pass ${TICKETING_API_URL}/api/v1/issues;
        proxy_set_header Host ${TICKETING_API_HOST};
        proxy_ssl_verify off;
        proxy_ssl_server_name on;
    }

    # Proxy für Platform Service (catch-all)
    location /api/ {
        proxy_pass ${PLATFORM_API_URL}/api/;
        proxy_set_header Host ${PLATFORM_API_HOST};
        proxy_ssl_verify off;
        proxy_ssl_server_name on;
    }
}
```

### Dockerfile Änderungen

```dockerfile
FROM nginx:stable

# Template für envsubst
COPY docker/nginx/templates /etc/nginx/templates

# conf.d muss für envsubst writable sein
RUN chmod 777 /etc/nginx/conf.d

# Rootless nginx
USER nginx
```

### Environment Variables (Terraform)

```hcl
# Werden von Terraform automatisch gesetzt
env {
  name  = "PLATFORM_API_URL"
  value = "https://rmsfl-dev-weu-ca-platform.${domain}"
}
env {
  name  = "TICKETING_API_URL"
  value = "https://rmsfl-dev-weu-ca-ticketing.${domain}"
}
```

**Begründung:**
- `nginx:stable` Image führt automatisch `envsubst` auf Templates aus
- Keine Entrypoint-Scripts notwendig
- Dynamische Backend-URLs für verschiedene Umgebungen

---

## Zusammenfassung

### Architektur-Entscheidungen

| Entscheidung | Begründung | Auswirkung |
|--------------|------------|------------|
| **PostgreSQL Flexible Server** | PostgreSQL bereits lokal verwendet, Azure-native Unterstützung | Nur Connection String Anpassung |
| **Cosmos DB Cassandra API** | CQL-Kompatibilität, keine Schema-Änderungen | SSL-Konfiguration im CassandraExecutor, Tabellenerstellung in Terraform (Liquibase unterstützt Cosmos DB SSL nicht) |
| **Event Hubs statt Kafka** | Managed Service, KEDA-Integration | Nur Konfigurationsänderungen |
| **Key Vault ConfigSource** | Zentrale Secret-Verwaltung | Keine Secrets im Code/Config |
| **Storage Abstraction** | Multi-Provider Support | Factory Pattern implementiert |
| **Managed Identity** | Keine Credentials im Code | Automatische Rotation |
