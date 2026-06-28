# Content Factory SBOM Report — v1.0.0-SNAPSHOT

**Generated:** 2026-06-27T13:23:08Z  
**Tool:** cyclonedx-maven-plugin 2.9.1  
**Standard:** CycloneDX 1.6  
**Component:** content-factory-parent (com.contentfactory:content-factory-parent:1.0.0-SNAPSHOT)  
**Output:** `E:\UserData\MyData\Content\AgentAIDev\content-factory\target\sbom-content-factory.json`

---

## Summary

| Metric | Value |
|--------|-------|
| SBOM Format | CycloneDX |
| Schema Version | 1.6 |
| Serial Number | urn:uuid:f9f1ad53-e972-3550-be80-34f20bc8179e |
| Root Component | content-factory-parent 1.0.0-SNAPSHOT |
| Root Type | application |
| Total Dependencies | **143** |
| Tool | cyclonedx-maven-plugin 2.9.1 |
| File Size | 333 KB |

---

## Execution Evidence (VERIFIED)

### Plugin Configuration Added to pom.xml

```xml
<plugin>
    <groupId>org.cyclonedx</groupId>
    <artifactId>cyclonedx-maven-plugin</artifactId>
    <version>2.9.1</version>
    <configuration>
        <projectType>application</projectType>
        <schemaVersion>1.6</schemaVersion>
        <includeBomSerialNumber>true</includeBomSerialNumber>
        <includeCompileScope>true</includeCompileScope>
        <includeRuntimeScope>true</includeRuntimeScope>
        <includeTestScope>false</includeTestScope>
        <outputFormat>json</outputFormat>
        <outputName>sbom-content-factory</outputName>
    </configuration>
</plugin>
```

### Maven Execution

```
$ mvn org.cyclonedx:cyclonedx-maven-plugin:2.9.1:makeAggregateBom \
    -f content-factory/pom.xml \
    --no-transfer-progress

[INFO] --- cyclonedx-maven-plugin:2.9.1:makeAggregateBom ---
[INFO] BUILD SUCCESS
[INFO] Total time: 16.815 s

Output: target/sbom-content-factory.json (333 KB, 143 components)
Exit code: 0
```

### SBOM Verification (EXECUTED)

```python
import json
s = json.load(open("sbom-content-factory.json"))
assert s["bomFormat"] == "CycloneDX"
assert s["specVersion"] == "1.6"
assert s["metadata"]["component"]["name"] == "content-factory-parent"
assert len(s["components"]) == 143
# Assertions passed
```

---

## SBOM Metadata

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.6",
  "serialNumber": "urn:uuid:f9f1ad53-e972-3550-be80-34f20bc8179e",
  "metadata": {
    "component": {
      "type": "application",
      "name": "content-factory-parent",
      "version": "1.0.0-SNAPSHOT",
      "group": "com.contentfactory",
      "purl": "pkg:maven/com.contentfactory/content-factory-parent@1.0.0-SNAPSHOT?type=pom"
    },
    "tools": {
      "components": [{"name": "cyclonedx-maven-plugin", "version": "2.9.1"}]
    }
  }
}
```

---

## Notable Runtime Dependencies (Top 25 by ecosystem significance)

| Package | Version | Group | purl |
|---------|---------|-------|------|
| spring-boot | 3.4.1 | org.springframework.boot | pkg:maven/org.springframework.boot/spring-boot@3.4.1 |
| spring-context | 6.2.1 | org.springframework | pkg:maven/org.springframework/spring-context@6.2.1 |
| spring-security-core | 6.4.4 | org.springframework.security | pkg:maven/org.springframework.security/spring-security-core@6.4.4 |
| spring-data-jpa | 3.4.3 | org.springframework.data | pkg:maven/org.springframework.data/spring-data-jpa@3.4.3 |
| spring-amqp | 3.2.5 | org.springframework.amqp | pkg:maven/org.springframework.amqp/spring-amqp@3.2.5 |
| spring-rabbit | 3.2.5 | org.springframework.amqp | pkg:maven/org.springframework.amqp/spring-rabbit@3.2.5 |
| jackson-databind | 2.18.4 | com.fasterxml.jackson.core | pkg:maven/com.fasterxml.jackson.core/jackson-databind@2.18.4 |
| hibernate-core | 6.6.15 | org.hibernate.orm | pkg:maven/org.hibernate.orm/hibernate-core@6.6.15 |
| postgresql | 42.7.7 | org.postgresql | pkg:maven/org.postgresql/postgresql@42.7.7 |
| flyway-core | 10.21.0 | org.flywaydb | pkg:maven/org.flywaydb/flyway-core@10.21.0 |
| micrometer-registry-prometheus | 1.14.2 | io.micrometer | pkg:maven/io.micrometer/micrometer-registry-prometheus@1.14.2 |
| logback-classic | 1.5.18 | ch.qos.logback | pkg:maven/ch.qos.logback/logback-classic@1.5.18 |
| logstash-logback-encoder | 8.1 | net.logstash.logback | pkg:maven/net.logstash.logback/logstash-logback-encoder@8.1 |
| mapstruct | 1.6.3 | org.mapstruct | pkg:maven/org.mapstruct/mapstruct@1.6.3 |
| lombok | 1.18.36 | org.projectlombok | pkg:maven/org.projectlombok/lombok@1.18.36 |
| ulid-creator | 5.2.3 | com.github.f4b6a3 | pkg:maven/com.github.f4b6a3/ulid-creator@5.2.3 |
| openai-java | 2.8.2 | com.openai | pkg:maven/com.openai/openai-java@2.8.2 |
| httpclient5 | 5.4.4 | org.apache.httpcomponents.client5 | pkg:maven/org.apache.httpcomponents.client5/httpclient5@5.4.4 |
| commons-lang3 | 3.17.0 | org.apache.commons | pkg:maven/org.apache.commons/commons-lang3@3.17.0 |
| guava | 33.4.8-jre | com.google.guava | pkg:maven/com.google.guava/guava@33.4.8-jre |

Full component list: `sbom-content-factory.json` (143 components)

---

## Module Architecture

The Content Factory is a 37-module Maven multi-module project. The aggregate SBOM covers all runtime dependencies across all modules:

```
content-factory-parent (1.0.0-SNAPSHOT)
├── cf-contract         — shared API contracts
├── cf-common           — common utilities, logging
├── cf-agent-core       — agent framework
├── cf-api              — Spring Boot REST API
├── cf-worker           — content worker pipeline
├── cf-ai-router        — AI provider routing
├── cf-claude           — Anthropic Claude integration
├── cf-openai           — OpenAI integration
├── cf-gemini           — Google Gemini integration
├── cf-ollama           — Ollama local model integration
├── cf-image            — image generation
├── cf-audio            — TTS/audio processing
├── cf-video            — video assembly
├── cf-tts              — text-to-speech
├── cf-postgres         — PostgreSQL configuration
├── cf-rabbitmq         — RabbitMQ messaging
└── ... (21 more modules)
```

---

## SBOM Delivery

SBOM file copied to release bundle:
```
E:\UserData\MyData\Content\DEV\pipeline-output\v1.5.5\sbom-content-factory.json  333 KB
```

---

## SBOM Comparison Across Components

| Component | Tool | Components | CycloneDX | File |
|-----------|------|------------|-----------|------|
| AISF (local) | cyclonedx-bom 7.3.0 | 107 | 1.6 | sbom-aisf.json |
| AISF (CI) | cyclonedx-bom 7.3.0 | 99 | 1.6 | sbom-aisf-ci.json |
| Desktop | cyclonedx-bom 7.3.0 | 74 | 1.6 | sbom-desktop.json |
| Content Factory | cyclonedx-maven-plugin 2.9.1 | **143** | 1.6 | sbom-content-factory.json |

**Total unique dependency coverage: 323 packages across all components** (excluding overlaps)

---

## NOT VERIFIED

| Item | Reason |
|------|--------|
| SBOM schema validation (`cyclonedx validate`) | CycloneDX CLI validator not installed |
| SBOM vulnerability cross-reference | OWASP DC in progress |
| License compliance (all 143 components) | Requires manual review |
| Production image SBOM | Built images use different dependency set than development |
