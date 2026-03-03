# Azure Migration Assessment Report

**Application**: Assets Manager  
**Assessment Date**: 2026-03-03  
**Current Stack**: Java 8, Spring Boot 2.7.18, AWS S3, RabbitMQ, PostgreSQL  
**Target Stack**: Java 21, Spring Boot 3.3.x, Azure Blob Storage, Azure Service Bus, Azure Database for PostgreSQL  

---

## Executive Summary

This assessment analyzes the **Assets Manager** application — a multi-module Spring Boot microservices system consisting of a **web** module (file upload/management UI) and a **worker** module (asynchronous thumbnail generation). The application currently runs on Java 8 with Spring Boot 2.7.18 and uses AWS S3 for file storage, RabbitMQ for messaging, and PostgreSQL for metadata persistence.

The migration to Azure involves **four major workstreams**:
1. Java and Spring Boot version upgrade
2. AWS to Azure cloud service migration
3. Security and authentication modernization
4. Containerization and deployment to Azure Container Apps

---

## Table of Contents

1. [Critical Issues](#1-critical-issues)
2. [Java Version Upgrade (8 → 21)](#2-java-version-upgrade-8--21)
3. [Spring Boot Upgrade (2.7 → 3.x)](#3-spring-boot-upgrade-27--3x)
4. [AWS S3 to Azure Blob Storage Migration](#4-aws-s3-to-azure-blob-storage-migration)
5. [RabbitMQ to Azure Service Bus Migration](#5-rabbitmq-to-azure-service-bus-migration)
6. [Database Migration Assessment](#6-database-migration-assessment)
7. [Authentication and Security](#7-authentication-and-security)
8. [Deprecated API Usage](#8-deprecated-api-usage)
9. [Dependency Vulnerability Assessment](#9-dependency-vulnerability-assessment)
10. [Configuration Changes](#10-configuration-changes)
11. [Testing Impact](#11-testing-impact)
12. [Recommended Migration Order](#12-recommended-migration-order)
13. [Risk Matrix](#13-risk-matrix)

---

## 1. Critical Issues

### 1.1 — javax.persistence → jakarta.persistence Namespace Change

| Severity | Impact | Effort |
|----------|--------|--------|
| 🔴 Critical | Both modules | Medium |

Spring Boot 3.x requires the Jakarta EE 9+ namespace. All `javax.persistence.*` imports must change to `jakarta.persistence.*`.

**Affected files:**
- `web/src/main/java/.../model/ImageMetadata.java` — Lines 4 (`import javax.persistence.*`)
- `worker/src/main/java/.../model/ImageMetadata.java` — Lines 3-5 (`import javax.persistence.*`)
- `web/src/main/java/.../service/LocalFileStorageService.java` — Line 14 (`import javax.annotation.PostConstruct`)
- `worker/src/main/java/.../service/LocalFileProcessingService.java` — Line 6 (`import javax.annotation.PostConstruct`)
- `web/src/main/java/.../config/WebMvcConfig.java` — Lines 12-13 (`import javax.servlet.http.*`)

**Required changes:**
```java
// Before
import javax.persistence.*;
import javax.annotation.PostConstruct;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

// After
import jakarta.persistence.*;
import jakarta.annotation.PostConstruct;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
```

### 1.2 — AWS SDK Dependency Must Be Replaced with Azure SDK

| Severity | Impact | Effort |
|----------|--------|--------|
| 🔴 Critical | Both modules | High |

Both modules depend on `software.amazon.awssdk:s3:2.25.13`. This must be replaced with the Azure Storage Blob SDK.

**Affected Maven POMs:**
- `web/pom.xml` — AWS SDK S3 dependency (lines 35-37, 70-76)
- `worker/pom.xml` — AWS SDK S3 dependency (lines 31-33, 58-65)

**Required dependency change:**
```xml
<!-- Remove -->
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3</artifactId>
</dependency>

<!-- Add -->
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-storage-blob</artifactId>
</dependency>
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-identity</artifactId>
</dependency>
```

### 1.3 — Static AWS Credentials Must Be Replaced with Azure Managed Identity

| Severity | Impact | Effort |
|----------|--------|--------|
| 🔴 Critical | Security | Medium |

Both `AwsS3Config.java` files use hardcoded access key/secret key authentication which is a security anti-pattern. Azure migration should use Managed Identity (passwordless authentication).

**Affected files:**
- `web/src/main/java/.../config/AwsS3Config.java` — Uses `AwsBasicCredentials` with `@Value("${aws.accessKey}")`
- `worker/src/main/java/.../config/AwsS3Config.java` — Uses `AwsBasicCredentials` with `@Value("${aws.accessKeyId}")`

---

## 2. Java Version Upgrade (8 → 21)

### 2.1 — Source Compatibility

| Issue | File | Line | Description |
|-------|------|------|-------------|
| Java version property | `pom.xml` | 19 | `<java.version>8</java.version>` must change to `21` |
| Raw type usage | Multiple | Various | Review for potential issues with stricter type checks in Java 21 |

### 2.2 — Removed/Changed APIs in Java 9-21

The following APIs used in the code were affected by Java modularization (Java 9+):

| API | Status | Impact |
|-----|--------|--------|
| `javax.annotation.PostConstruct` | Removed from JDK | Must use `jakarta.annotation.PostConstruct` or Spring `@PostConstruct` |
| `javax.imageio.*` | Still available | No change needed (part of `java.desktop` module) |
| `java.awt.*` | Still available | No change needed (part of `java.desktop` module) |

### 2.3 — Java 21 Benefits Available Post-Migration

- Virtual Threads (Project Loom) — could improve throughput for image processing worker
- Record types — could simplify `ImageProcessingMessage`, `S3StorageItem` model classes
- Sealed classes — could formalize `StorageService`/`FileProcessor` hierarchies
- Pattern matching for `instanceof` — simplify type checks
- Text blocks — simplify string templates in logging

---

## 3. Spring Boot Upgrade (2.7 → 3.x)

### 3.1 — Breaking Changes

| Issue | Severity | Files Affected |
|-------|----------|----------------|
| Jakarta EE 9 namespace migration (`javax.*` → `jakarta.*`) | 🔴 Critical | 5 files (see §1.1) |
| `WebMvcConfigurerAdapter` removed | 🔴 Critical | `WebMvcConfig.java` |
| `HandlerInterceptorAdapter` removed | 🔴 Critical | `WebMvcConfig.java` (inner class) |
| `spring.jpa.properties.hibernate.dialect` auto-detected | 🟡 Warning | `application.properties` (both modules) |
| Minimum Java 17 required | 🟡 Warning | `pom.xml` |

### 3.2 — WebMvcConfigurerAdapter Removal

**File**: `web/src/main/java/.../config/WebMvcConfig.java`

`WebMvcConfigurerAdapter` was deprecated in Spring 5 and **removed in Spring 6** (used by Spring Boot 3.x). The code already has `@SuppressWarnings("deprecation")` indicating awareness of the issue.

```java
// Before (current code - line 17)
public class WebMvcConfig extends WebMvcConfigurerAdapter {

// After
public class WebMvcConfig implements WebMvcConfigurer {
```

### 3.3 — HandlerInterceptorAdapter Removal

**File**: `web/src/main/java/.../config/WebMvcConfig.java` (line 49)

`HandlerInterceptorAdapter` was deprecated in Spring 5 and **removed in Spring 6**. The inner class `FileOperationLoggingInterceptor` must implement `HandlerInterceptor` instead.

```java
// Before (current code - line 49)
private static class FileOperationLoggingInterceptor extends HandlerInterceptorAdapter {

// After
private static class FileOperationLoggingInterceptor implements HandlerInterceptor {
```

### 3.4 — Auto-Configuration Changes

Spring Boot 3.x changes:
- `spring.jpa.properties.hibernate.dialect` is auto-detected and no longer needs to be explicitly set
- `spring.main.banner-mode` default changed
- Actuator endpoint path changes (if actuator is added later)
- New `spring.docker.compose` support available for dev environments

---

## 4. AWS S3 to Azure Blob Storage Migration

### 4.1 — Service Layer Changes (Web Module)

**File**: `web/src/main/java/.../service/AwsS3Service.java`

This is the **highest-effort migration** item. The entire S3 service must be rewritten to use Azure Blob Storage.

| S3 Operation | Azure Equivalent | Complexity |
|-------------|-----------------|------------|
| `S3Client.listObjectsV2()` | `BlobContainerClient.listBlobs()` | Low |
| `S3Client.putObject()` | `BlobClient.upload()` | Low |
| `S3Client.getObject()` | `BlobClient.openInputStream()` | Low |
| `S3Client.deleteObject()` | `BlobClient.delete()` | Low |
| `PutObjectRequest.builder()` | `BlobClient` methods | Medium |
| `RequestBody.fromInputStream()` | `BlobClient.upload(InputStream, long)` | Low |

**Key API mapping:**
```java
// AWS S3
S3Client s3Client;
s3Client.putObject(PutObjectRequest.builder().bucket(bucket).key(key).build(), 
    RequestBody.fromInputStream(inputStream, size));

// Azure Blob Storage
BlobContainerClient containerClient;
BlobClient blobClient = containerClient.getBlobClient(key);
blobClient.upload(inputStream, size, true);
```

### 4.2 — Service Layer Changes (Worker Module)

**File**: `worker/src/main/java/.../service/S3FileProcessingService.java`

| S3 Operation | Azure Equivalent |
|-------------|-----------------|
| `S3Client.getObject(GetObjectRequest)` | `BlobClient.openInputStream()` |
| `S3Client.putObject(PutObjectRequest, RequestBody.fromFile())` | `BlobClient.uploadFromFile()` |
| `S3Client.utilities().getUrl()` | `BlobClient.getBlobUrl()` |

### 4.3 — Configuration Changes

**Files**: `web/.../config/AwsS3Config.java`, `worker/.../config/AwsS3Config.java`

Both AWS configuration classes must be replaced with Azure Blob Storage configuration using `DefaultAzureCredential` (Managed Identity).

```java
// Azure replacement
@Configuration
public class AzureBlobStorageConfig {
    @Value("${azure.storage.account-name}")
    private String accountName;

    @Value("${azure.storage.container-name}")
    private String containerName;

    @Bean
    public BlobServiceClient blobServiceClient() {
        return new BlobServiceClientBuilder()
            .endpoint("https://" + accountName + ".blob.core.windows.net")
            .credential(new DefaultAzureCredentialBuilder().build())
            .buildClient();
    }

    @Bean
    public BlobContainerClient blobContainerClient(BlobServiceClient blobServiceClient) {
        return blobServiceClient.getBlobContainerClient(containerName);
    }
}
```

### 4.4 — Model/Naming Impact

The `S3StorageItem` model class and references throughout use "S3" in naming. While this is a cosmetic issue, it should be renamed for clarity:

**Affected files:**
- `web/.../model/S3StorageItem.java` → `StorageItem.java`
- `web/.../service/StorageService.java` — return type `List<S3StorageItem>`
- `web/.../service/AwsS3Service.java` — all S3StorageItem references
- `web/.../service/LocalFileStorageService.java` — all S3StorageItem references
- `web/.../controller/S3Controller.java` — class name and S3StorageItem references
- `web/.../model/ImageMetadata.java` — `s3Key` and `s3Url` field names

---

## 5. RabbitMQ to Azure Service Bus Migration

### 5.1 — Dependency Changes

```xml
<!-- Remove -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>

<!-- Add (version managed by Spring Cloud Azure BOM) -->
<dependency>
    <groupId>com.azure.spring</groupId>
    <artifactId>spring-cloud-azure-starter-servicebus-jms</artifactId>
</dependency>

<!-- Add BOM to dependencyManagement -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.azure.spring</groupId>
            <artifactId>spring-cloud-azure-dependencies</artifactId>
            <version>${spring-cloud-azure.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 5.2 — Configuration Changes

**File**: `web/.../config/RabbitConfig.java`, `worker/.../config/RabbitConfig.java`

Both RabbitConfig classes must be replaced with Service Bus configuration.

| RabbitMQ Concept | Azure Service Bus Equivalent |
|-----------------|------------------------------|
| Queue | Queue |
| Exchange | Topic |
| `RabbitTemplate` | `JmsTemplate` or `ServiceBusSenderClient` |
| `@RabbitListener` | `@JmsListener` or `ServiceBusProcessorClient` |
| `Channel.basicAck()` | `ServiceBusReceivedMessageContext.complete()` |
| `Channel.basicNack()` | `ServiceBusReceivedMessageContext.abandon()` |
| `Jackson2JsonMessageConverter` | JSON message body serialization |
| `AcknowledgeMode.MANUAL` | `ServiceBusReceiveMode.PEEK_LOCK` |

### 5.3 — Message Processing Changes

**Files affected:**
- `web/.../service/AwsS3Service.java` (line 85) — `rabbitTemplate.convertAndSend()`
- `web/.../service/LocalFileStorageService.java` (line 105) — `rabbitTemplate.convertAndSend()`
- `web/.../service/BackupMessageProcessor.java` — `@RabbitListener` + `Channel` operations
- `worker/.../service/AbstractFileProcessingService.java` (line 24) — `@RabbitListener` + manual acknowledgment

The manual acknowledgment pattern in the worker module is a significant change:

```java
// Current RabbitMQ pattern (AbstractFileProcessingService.java)
@RabbitListener(queues = IMAGE_PROCESSING_QUEUE)
public void processImage(final ImageProcessingMessage message, 
                       Channel channel, 
                       @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) {
    // ... processing ...
    channel.basicAck(deliveryTag, false);    // Success
    channel.basicNack(deliveryTag, false, false);  // Failure
}

// Azure Service Bus equivalent
@ServiceBusListener(destination = IMAGE_PROCESSING_QUEUE)
public void processImage(final ImageProcessingMessage message,
                       ServiceBusReceivedMessageContext context) {
    // ... processing ...
    context.complete();   // Success
    context.abandon();    // Failure (or deadLetter())
}
```

### 5.4 — Application Properties Changes

```properties
# Remove
spring.rabbitmq.host=${SPRING_RABBITMQ_HOST:localhost}
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest

# Add
spring.cloud.azure.servicebus.namespace=${AZURE_SERVICEBUS_NAMESPACE}
spring.cloud.azure.servicebus.credential.managed-identity-enabled=true
```

---

## 6. Database Migration Assessment

### 6.1 — PostgreSQL Compatibility

| Aspect | Current | Azure Target | Impact |
|--------|---------|-------------|--------|
| Database engine | PostgreSQL 16 | Azure Database for PostgreSQL Flexible Server | ✅ Compatible |
| JDBC Driver | `org.postgresql:postgresql` | Same | ✅ No change |
| Hibernate dialect | `PostgreSQLDialect` | Same (auto-detected in Spring Boot 3) | ✅ Minor |
| DDL strategy | `hibernate.ddl-auto=update` | Same (⚠️ not recommended for production) | 🟡 Warning |
| Connection string | `jdbc:postgresql://${POSTGRES_HOST}:5432/assets_manager` | `jdbc:postgresql://${server}.postgres.database.azure.com:5432/assets_manager` | ✅ Config change |

### 6.2 — Authentication Change

Current: Username/password (`postgres:postgres`)  
Target: Azure AD / Managed Identity passwordless authentication

```properties
# Add for Azure passwordless authentication
spring.datasource.azure.passwordless-enabled=true
spring.cloud.azure.credential.managed-identity-enabled=true
spring.cloud.azure.credential.client-id=${AZURE_CLIENT_ID}
```

### 6.3 — Schema Management Warning

⚠️ Both modules use `spring.jpa.hibernate.ddl-auto=update`. This is acceptable for development but **not recommended for production** on Azure. Consider:
- Adding **Flyway** or **Liquibase** for schema migration management
- Setting `ddl-auto=validate` in production

---

## 7. Authentication and Security

### 7.1 — Credential Hardcoding (Security Risk)

| File | Issue | Severity |
|------|-------|----------|
| `web/application.properties` | `aws.accessKey=your-access-key`, `aws.secretKey=your-secret-key` | 🔴 Critical |
| `worker/application.properties` | `aws.accessKeyId=your-access-key-Id`, `aws.secretKey=your-secret-key` | 🔴 Critical |
| `web/application.properties` | `spring.datasource.username=postgres`, `spring.datasource.password=postgres` | 🔴 Critical |
| `worker/application.properties` | `spring.datasource.username=postgres`, `spring.datasource.password=postgres` | 🔴 Critical |
| `web/application.properties` | `spring.rabbitmq.username=guest`, `spring.rabbitmq.password=guest` | 🟡 Warning |

### 7.2 — Azure Managed Identity (Target Architecture)

Azure migration should use **passwordless authentication** for all services:

| Service | Authentication Method |
|---------|----------------------|
| Azure Blob Storage | `DefaultAzureCredential` (Managed Identity) |
| Azure Service Bus | `DefaultAzureCredential` (Managed Identity) |
| Azure Database for PostgreSQL | Azure AD / Managed Identity passwordless |

### 7.3 — Inconsistent Property Naming

The AWS access key property name differs between modules:
- Web: `aws.accessKey`
- Worker: `aws.accessKeyId`

This inconsistency should be resolved during migration. The worker module's `aws.accessKeyId` follows the AWS SDK v2 naming conventions and is the correct form. However, since both will be removed in favor of Azure Managed Identity, this is a moot point post-migration.

---

## 8. Deprecated API Usage

### 8.1 — Deprecated Classes and Methods

| Deprecated API | File | Replacement | Spring Boot 3 Status |
|---------------|------|-------------|---------------------|
| `WebMvcConfigurerAdapter` | `WebMvcConfig.java:17` | `WebMvcConfigurer` | **Removed** |
| `HandlerInterceptorAdapter` | `WebMvcConfig.java:49` | `HandlerInterceptor` | **Removed** |
| `@SuppressWarnings("deprecation")` | `WebMvcConfig.java:16` | Remove after fixing | N/A |

### 8.2 — Missing `@Bean` Annotation

**File**: `web/.../config/AwsS3Config.java` (line 23)

The `s3Client()` method is missing the `@Bean` annotation. This means the S3Client is **not being registered as a Spring Bean**, potentially causing injection failures.

```java
// Current (MISSING @Bean)
public S3Client s3Client() {

// Fix
@Bean
public S3Client s3Client() {
```

Note: The worker module's `AwsS3Config.java` correctly includes `@Bean` (line 22).

---

## 9. Dependency Vulnerability Assessment

### 9.1 — Spring Boot 2.7.18 Known CVEs

Spring Boot 2.7.x reached **end of OSS support** in November 2023. Several critical CVEs affect this version:

| CVE | Component | Severity | Description |
|-----|-----------|----------|-------------|
| Multiple | Spring Framework 5.3.x | Various | Multiple security fixes in Spring 6.x |
| Multiple | Hibernate 5.x | Various | Fixed in Hibernate 6.x (Spring Boot 3.x) |

### 9.2 — AWS SDK 2.25.13

This version should be checked for known vulnerabilities. It will be removed during migration, resolving any issues.

### 9.3 — Recommended Spring Boot Target

**Spring Boot 3.3.x** (latest stable) with:
- Spring Framework 6.1.x
- Hibernate 6.4.x
- Jakarta EE 10

---

## 10. Configuration Changes

### 10.1 — Web Module application.properties

```properties
# === REMOVE ===
aws.accessKey=your-access-key
aws.secretKey=your-secret-key
aws.region=us-east-1
aws.s3.bucket=your-bucket-name
spring.rabbitmq.host=${SPRING_RABBITMQ_HOST:localhost}
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
spring.datasource.username=postgres
spring.datasource.password=postgres
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

# === ADD ===
# Azure Blob Storage
azure.storage.account-name=${AZURE_STORAGE_ACCOUNT_NAME}
azure.storage.container-name=${AZURE_STORAGE_BLOB_CONTAINER_NAME}

# Azure Service Bus
spring.cloud.azure.servicebus.namespace=${AZURE_SERVICEBUS_NAMESPACE}
spring.cloud.azure.servicebus.credential.managed-identity-enabled=true

# Azure Identity
spring.cloud.azure.credential.managed-identity-enabled=true
spring.cloud.azure.credential.client-id=${AZURE_CLIENT_ID}

# PostgreSQL (passwordless)
spring.datasource.url=jdbc:postgresql://${POSTGRES_SERVER}.postgres.database.azure.com:5432/assets_manager
spring.datasource.azure.passwordless-enabled=true
```

### 10.2 — Worker Module application.properties

Same changes as web module, plus removal of `aws.accessKeyId` (note: different property name from web).

---

## 11. Testing Impact

### 11.1 — Existing Test Coverage

| Module | Test File | Status |
|--------|-----------|--------|
| Worker | `S3FileProcessingServiceTest.java` | ⚠️ Must be rewritten for Azure |
| Web | No tests found | 🔴 No test coverage |

### 11.2 — Tests That Must Change

**`S3FileProcessingServiceTest.java`**:
- All `@Mock S3Client` references → `@Mock BlobContainerClient`
- `GetObjectRequest` / `PutObjectRequest` assertions → Azure Blob client assertions
- Test data setup for Azure SDK classes

### 11.3 — Recommended New Tests

- Azure Blob Storage integration tests (upload, download, delete, list)
- Azure Service Bus message send/receive tests
- End-to-end image processing flow test
- Configuration validation tests for Azure properties
- Health check tests for Azure service connectivity

---

## 12. Recommended Migration Order

### Phase 1: Java & Spring Boot Upgrade (Low Risk)
1. ✅ Upgrade Java version: 8 → 21 (`pom.xml`)
2. ✅ Upgrade Spring Boot: 2.7.18 → 3.3.x (`pom.xml`)
3. ✅ Migrate `javax.*` → `jakarta.*` imports (5 files)
4. ✅ Fix `WebMvcConfigurerAdapter` → `WebMvcConfigurer`
5. ✅ Fix `HandlerInterceptorAdapter` → `HandlerInterceptor`
6. ✅ Fix missing `@Bean` on web `AwsS3Config.s3Client()`
7. ✅ Run tests, verify builds

### Phase 2: AWS → Azure Service Migration (High Risk)
8. ✅ Replace AWS S3 SDK with Azure Blob Storage SDK
9. ✅ Rewrite `AwsS3Config` → `AzureBlobStorageConfig` (both modules)
10. ✅ Rewrite `AwsS3Service` → `AzureBlobStorageService`
11. ✅ Rewrite `S3FileProcessingService` → `AzureBlobFileProcessingService`
12. ✅ Update tests for Azure SDK

### Phase 3: RabbitMQ → Azure Service Bus Migration (Medium Risk)
13. ✅ Replace `spring-boot-starter-amqp` with Azure Service Bus dependency
14. ✅ Rewrite `RabbitConfig` → `ServiceBusConfig` (both modules)
15. ✅ Update message sending in storage services
16. ✅ Rewrite `AbstractFileProcessingService` message listener
17. ✅ Rewrite `BackupMessageProcessor`

### Phase 4: Security & Authentication (Medium Risk)
18. ✅ Implement Managed Identity for all Azure services
19. ✅ Remove hardcoded credentials
20. ✅ Configure passwordless PostgreSQL authentication
21. ✅ Add environment-specific configuration profiles

### Phase 5: Containerization & Deployment (Low Risk)
22. ✅ Create Dockerfiles for both modules
23. ✅ Configure Azure Container Apps deployment
24. ✅ Set up CI/CD pipeline

---

## 13. Risk Matrix

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| Spring Boot 3.x incompatibility | Medium | High | Incremental upgrade via 2.7→3.0→3.3 |
| Azure Service Bus message format incompatibility | Medium | High | Thorough testing of message serialization |
| Azure Blob Storage API differences | Low | Medium | Well-documented Azure SDK; straightforward mapping |
| Database connection with Managed Identity | Medium | Medium | Use Azure Service Connector for configuration |
| Image processing (java.awt) in containers | Low | High | Ensure headless AWT support in Docker base image |
| Performance differences between RabbitMQ and Service Bus | Low | Medium | Load testing post-migration |

---

## Appendix: Complete File Impact Matrix

| File | Changes Required | Effort |
|------|-----------------|--------|
| `pom.xml` (root) | Java version 8→21, Spring Boot 2.7→3.3 | Low |
| `web/pom.xml` | Replace AWS SDK with Azure SDK, add Azure identity | Medium |
| `worker/pom.xml` | Replace AWS SDK with Azure SDK, add Azure identity | Medium |
| `web/.../config/AwsS3Config.java` | Complete rewrite → AzureBlobStorageConfig | Medium |
| `worker/.../config/AwsS3Config.java` | Complete rewrite → AzureBlobStorageConfig | Medium |
| `web/.../config/RabbitConfig.java` | Complete rewrite → ServiceBusConfig | Medium |
| `worker/.../config/RabbitConfig.java` | Complete rewrite → ServiceBusConfig | Medium |
| `web/.../config/WebMvcConfig.java` | Fix deprecated classes, javax→jakarta | Low |
| `web/.../service/AwsS3Service.java` | Complete rewrite → AzureBlobStorageService | High |
| `worker/.../service/S3FileProcessingService.java` | Complete rewrite → AzureBlobFileProcessingService | High |
| `web/.../service/LocalFileStorageService.java` | javax→jakarta import | Low |
| `worker/.../service/LocalFileProcessingService.java` | javax→jakarta import | Low |
| `web/.../service/BackupMessageProcessor.java` | RabbitMQ→Service Bus listener | Medium |
| `worker/.../service/AbstractFileProcessingService.java` | RabbitMQ→Service Bus listener, javax→jakarta | Medium |
| `web/.../model/ImageMetadata.java` | javax→jakarta imports | Low |
| `worker/.../model/ImageMetadata.java` | javax→jakarta imports | Low |
| `web/.../model/S3StorageItem.java` | Rename to StorageItem (optional) | Low |
| `web/application.properties` | Replace AWS/RabbitMQ with Azure config | Medium |
| `worker/application.properties` | Replace AWS/RabbitMQ with Azure config | Medium |
| `worker/.../service/S3FileProcessingServiceTest.java` | Complete rewrite for Azure SDK | Medium |

**Total files requiring changes: 20**  
**Estimated effort: 3-5 developer days**
