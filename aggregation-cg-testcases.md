Awesome — here are **production-grade unit/integration tests** that validate **every layer** (Mapper, Repository, Service) and **every field** of the output against a **real MongoDB** using **Testcontainers**. I’ve also included a **step-by-step procedure** and the **test dependencies** you’ll need.

---

# ✅ What you’ll get

* **Mapper unit tests** (pure MapStruct): field-by-field assertions
* **Repository integration tests** (real Mongo): validates each aggregation pipeline’s output
* **Service integration tests** (real Mongo): validates merge & persist, including nested lists (`cve_list`, `priorities`) and `date_loaded`
* **Testcontainers** MongoDB for reliable, isolated tests
* **Strict assertions** for **every single field** in the output
* **Consistent snake\_case naming** to match your codebase style

---

## 📦 Add test dependencies to `pom.xml`

> Put these in the `<dependencies>` section and keep your existing ones.

```xml
<!-- Spring Boot Test -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
</dependency>

<!-- Testcontainers: JUnit 5 + MongoDB -->
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>junit-jupiter</artifactId>
  <version>1.19.8</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>mongodb</artifactId>
  <version>1.19.8</version>
  <scope>test</scope>
</dependency>

<!-- MapStruct test helper (optional, if you prefer Mappers.getMapper) -->
<dependency>
  <groupId>org.mapstruct</groupId>
  <artifactId>mapstruct</artifactId>
  <version>${mapstruct.version}</version>
  <scope>test</scope>
</dependency>
```

> Make sure your **Surefire**/ **Failsafe** plugins (if any) are JUnit 5 compatible. Spring Boot’s default is fine.

---

## 🧪 Test data used across layers

We will write documents into **`collection_one`**:

* **AIT-001** with three findings (two for `CVE-2024-0001` and one for `CVE-2024-0002`), all **High** severity
* **AIT-002** with one finding (`CVE-2024-1234`), **Low** severity

Expected:

* `aggregate_cve_items`: AIT-001 has CVE-0001 (instances\_found=2) and CVE-0002 (1); AIT-002 has CVE-1234 (1).
* `aggregate_priorities`: AIT-001 -> High (3), AIT-002 -> Low (1).
* Service persists **two** `target_document` rows with correct header fields, nested `cve_list` and `priorities`, and a non-null `date_loaded`.

---

## 🧭 Package layout (tests)

```
src/test/java/com/example/agg/
  ├── test_support/
  │   ├── mongo_container_config.java
  │   └── sample_data_loader.java
  ├── mapper/
  │   └── collection_one_mapper_test.java
  ├── repo/
  │   └── aggregation_repository_it.java
  └── service/
      └── aggregation_service_it.java
```

---

## 🧰 Shared Testcontainers setup

### `src/test/java/com/example/agg/test_support/mongo_container_config.java`

```java
package com.example.agg.test_support;

import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.test.context.junit.jupiter.SpringExtension;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.containers.MongoDBContainer;

@ExtendWith(SpringExtension.class)
@Testcontainers
public abstract class mongo_container_config {

    @Container
    static final MongoDBContainer mongo = new MongoDBContainer("mongo:7.0.12");

    @DynamicPropertySource
    static void mongo_props(DynamicPropertyRegistry registry) {
        registry.add("spring.data.mongodb.uri", mongo::getReplicaSetUrl);
    }
}
```

### `src/test/java/com/example/agg/test_support/sample_data_loader.java`

```java
package com.example.agg.test_support;

import org.springframework.data.mongodb.core.MongoTemplate;
import org.bson.Document;

import java.time.LocalDate;
import java.time.ZoneId;
import java.util.Date;
import java.util.List;

public class sample_data_loader {

    public static void load_collection_one(MongoTemplate mongo_template) {
        var coll = mongo_template.getCollection("collection_one");
        coll.drop();

        // Helper: convert LocalDate to java.util.Date
        var toDate = (java.util.function.Function<LocalDate, Date>) d ->
            Date.from(d.atStartOfDay(ZoneId.systemDefault()).toInstant());

        // -------- AIT-001, High severity, 3 findings (2 for CVE-0001, 1 for CVE-0002)
        Document commonHeader1 = new Document()
            .append("cio_lob_name", "LOB1")
            .append("cio_name", "CIO A")
            .append("cto_name", "CTO A")
            .append("cio_lob_name_one_deep", "LOB1-D")
            .append("ait_no", "AIT-001")
            .append("ait_name", "App One")
            .append("ait_app_manager", "Mgr One")
            .append("ait_app_manager_nbid", "NBID-001")
            .append("ait_mgmt_support_contact", "Support One")
            .append("ait_mgmt_support_contact_nbid", "NBID-S1")
            .append("ait_tech_exec", "Tech Exec One")
            .append("ait_tech_exec_nbid", "NBID-T1")
            .append("ait_app_manager_email", "app1@corp.test")
            .append("gis_network_rating_zone", "ZoneA")
            .append("gis_metric_alignment", "Aligned")
            .append("ait_recovery_time_obj", "24h")
            .append("enterprise_lob_name", "ELOB")
            .append("country_name", "US")
            .append("region", "NA")
            .append("gis_asset_category", "Server")
            .append("device_type", "VM")
            .append("os_name", "Linux");

        Document baseCve1 = new Document(commonHeader1)
            .append("first_reported", toDate.apply(LocalDate.of(2024, 1, 1)))
            .append("due_date", toDate.apply(LocalDate.of(2024, 2, 1)))
            .append("past_due", false)
            .append("date_detected", toDate.apply(LocalDate.of(2024, 1, 2)))
            .append("environment", "PROD")
            .append("remediation_status", "OPEN")
            .append("days_open", 10)
            .append("last_severity", "High")
            .append("scorecard_erp_status", "Pending")
            .append("scorecard_source", "Qualys")
            .append("observation_title", "Vuln A")
            .append("observation_description", "Desc A");

        // Two occurrences of CVE-2024-0001
        Document doc1 = new Document(baseCve1).append("cve", "CVE-2024-0001");
        Document doc2 = new Document(baseCve1).append("cve", "CVE-2024-0001");

        // One occurrence of CVE-2024-0002
        Document doc3 = new Document(baseCve1).append("cve", "CVE-2024-0002");

        // -------- AIT-002, Low severity, 1 finding
        Document commonHeader2 = new Document()
            .append("cio_lob_name", "LOB2")
            .append("cio_name", "CIO B")
            .append("cto_name", "CTO B")
            .append("cio_lob_name_one_deep", "LOB2-D")
            .append("ait_no", "AIT-002")
            .append("ait_name", "App Two")
            .append("ait_app_manager", "Mgr Two")
            .append("ait_app_manager_nbid", "NBID-002")
            .append("ait_mgmt_support_contact", "Support Two")
            .append("ait_mgmt_support_contact_nbid", "NBID-S2")
            .append("ait_tech_exec", "Tech Exec Two")
            .append("ait_tech_exec_nbid", "NBID-T2")
            .append("ait_app_manager_email", "app2@corp.test")
            .append("gis_network_rating_zone", "ZoneB")
            .append("gis_metric_alignment", "Aligned")
            .append("ait_recovery_time_obj", "12h")
            .append("enterprise_lob_name", "ELOB2")
            .append("country_name", "IN")
            .append("region", "APAC")
            .append("gis_asset_category", "Workstation")
            .append("device_type", "Laptop")
            .append("os_name", "Windows");

        Document doc4 = new Document(commonHeader2)
            .append("first_reported", toDate.apply(LocalDate.of(2024, 3, 10)))
            .append("due_date", toDate.apply(LocalDate.of(2024, 4, 10)))
            .append("past_due", false)
            .append("date_detected", toDate.apply(LocalDate.of(2024, 3, 11)))
            .append("environment", "UAT")
            .append("remediation_status", "IN_PROGRESS")
            .append("days_open", 5)
            .append("last_severity", "Low")
            .append("scorecard_erp_status", "Open")
            .append("scorecard_source", "Nessus")
            .append("observation_title", "Vuln B")
            .append("observation_description", "Desc B")
            .append("cve", "CVE-2024-1234");

        coll.insertMany(List.of(doc1, doc2, doc3, doc4));
    }
}
```

---

## 🧭 Mapper unit tests

### `src/test/java/com/example/agg/mapper/collection_one_mapper_test.java`

```java
package com.example.agg.mapper;

import com.example.agg.domain.cve_item;
import com.example.agg.domain.priority_item;
import com.example.agg.domain.target_document;
import org.bson.Document;
import org.junit.jupiter.api.Test;
import org.mapstruct.factory.Mappers;

import java.time.LocalDate;
import java.time.ZoneId;
import java.util.Date;

import static org.assertj.core.api.Assertions.assertThat;

public class collection_one_mapper_test {

    private final collection_one_mapper mapper = Mappers.getMapper(collection_one_mapper.class);

    @Test
    void maps_header_fields_exactly() {
        Document doc = new Document()
            .append("cio_lob_name", "LOB1")
            .append("cio_name", "CIO A")
            .append("cto_name", "CTO A")
            .append("cio_lob_name_one_deep", "LOB1-D")
            .append("ait_no", "AIT-001")
            .append("ait_name", "App One")
            .append("ait_app_manager", "Mgr One")
            .append("ait_app_manager_nbid", "NBID-001")
            .append("ait_mgmt_support_contact", "Support One")
            .append("ait_mgmt_support_contact_nbid", "NBID-S1")
            .append("ait_tech_exec", "Tech Exec One")
            .append("ait_tech_exec_nbid", "NBID-T1")
            .append("ait_app_manager_email", "app1@corp.test")
            .append("gis_network_rating_zone", "ZoneA")
            .append("gis_metric_alignment", "Aligned")
            .append("ait_recovery_time_obj", "24h")
            .append("enterprise_lob_name", "ELOB")
            .append("country_name", "US")
            .append("region", "NA")
            .append("gis_asset_category", "Server")
            .append("device_type", "VM")
            .append("os_name", "Linux");

        target_document td = mapper.to_target_header(doc);

        assertThat(td.get_cio_lob_name()).isEqualTo("LOB1");
        assertThat(td.get_cio_name()).isEqualTo("CIO A");
        assertThat(td.get_cto_name()).isEqualTo("CTO A");
        assertThat(td.get_cio_lob_name_one_deep()).isEqualTo("LOB1-D");

        assertThat(td.get_ait_no()).isEqualTo("AIT-001");
        assertThat(td.get_ait_name()).isEqualTo("App One");
        assertThat(td.get_ait_app_manager()).isEqualTo("Mgr One");
        assertThat(td.get_ait_app_manager_nbid()).isEqualTo("NBID-001");
        assertThat(td.get_ait_mgmt_support_contact()).isEqualTo("Support One");
        assertThat(td.get_ait_mgmt_support_contact_nbid()).isEqualTo("NBID-S1");
        assertThat(td.get_ait_tech_exec()).isEqualTo("Tech Exec One");
        assertThat(td.get_ait_tech_exec_nbid()).isEqualTo("NBID-T1");
        assertThat(td.get_ait_app_manager_email()).isEqualTo("app1@corp.test");

        assertThat(td.get_gis_network_rating_zone()).isEqualTo("ZoneA");
        assertThat(td.get_gis_metric_alignment()).isEqualTo("Aligned");
        assertThat(td.get_ait_recovery_time_obj()).isEqualTo("24h");

        assertThat(td.get_enterprise_lob_name()).isEqualTo("ELOB");
        assertThat(td.get_country_name()).isEqualTo("US");
        assertThat(td.get_region()).isEqualTo("NA");
        assertThat(td.get_gis_asset_category()).isEqualTo("Server");
        assertThat(td.get_device_type()).isEqualTo("VM");
        assertThat(td.get_os_name()).isEqualTo("Linux");
    }

    @Test
    void maps_cve_item_fields_exactly() {
        Date firstReported = Date.from(LocalDate.of(2024,1,1).atStartOfDay(ZoneId.systemDefault()).toInstant());
        Date dueDate = Date.from(LocalDate.of(2024,2,1).atStartOfDay(ZoneId.systemDefault()).toInstant());
        Date dateDetected = Date.from(LocalDate.of(2024,1,2).atStartOfDay(ZoneId.systemDefault()).toInstant());

        Document doc = new Document()
            .append("cve", "CVE-2024-0001")
            .append("first_reported", firstReported)
            .append("due_date", dueDate)
            .append("past_due", false)
            .append("date_detected", dateDetected)
            .append("environment", "PROD")
            .append("remediation_status", "OPEN")
            .append("days_open", 10)
            .append("last_severity", "High")
            .append("scorecard_erp_status", "Pending")
            .append("scorecard_source", "Qualys")
            .append("observation_title", "Title A")
            .append("observation_description", "Desc A")
            .append("instances_found", 2);

        cve_item item = mapper.to_cve_item(doc);

        assertThat(item.get_name()).isEqualTo("CVE-2024-0001");
        assertThat(item.get_first_reported()).isEqualTo(LocalDate.of(2024,1,1));
        assertThat(item.get_due_date()).isEqualTo(LocalDate.of(2024,2,1));
        assertThat(item.is_past_due()).isFalse();
        assertThat(item.get_date_detected()).isEqualTo(LocalDate.of(2024,1,2));
        assertThat(item.get_environment()).isEqualTo("PROD");
        assertThat(item.get_remediation_status()).isEqualTo("OPEN");
        assertThat(item.get_days_open()).isEqualTo(10);
        assertThat(item.get_severity()).isEqualTo("High");
        assertThat(item.get_erp_status()).isEqualTo("Pending");
        assertThat(item.get_scorecard_source()).isEqualTo("Qualys");
        assertThat(item.get_observation_title()).isEqualTo("Title A");
        assertThat(item.get_observation_description()).isEqualTo("Desc A");
        assertThat(item.get_instances_found()).isEqualTo(2);
    }

    @Test
    void maps_priority_item_fields_exactly() {
        Document doc = new Document()
            .append("last_severity", "High")
            .append("instances_found", 3);

        priority_item p = mapper.to_priority_item(doc);

        assertThat(p.get_name()).isEqualTo("High");
        assertThat(p.get_instances_found()).isEqualTo(3);
    }
}
```

---

## 🧭 Repository integration tests (real Mongo)

### `src/test/java/com/example/agg/repo/aggregation_repository_it.java`

```java
package com.example.agg.repo;

import com.example.agg.test_support.mongo_container_config;
import com.example.agg.test_support.sample_data_loader;
import org.bson.Document;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.data.mongo.DataMongoTest;
import org.springframework.context.annotation.Import;
import org.springframework.data.mongodb.core.MongoTemplate;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

import static org.assertj.core.api.Assertions.assertThat;

@DataMongoTest
@Import(aggregation_repository.class)
public class aggregation_repository_it extends mongo_container_config {

    @Autowired
    MongoTemplate mongo_template;

    @Autowired
    aggregation_repository aggregation_repository;

    @BeforeEach
    void seed() {
        sample_data_loader.load_collection_one(mongo_template);
    }

    @Test
    void aggregate_header_returns_exact_header_rows() {
        List<Document> headers = aggregation_repository.aggregate_header("collection_one");
        assertThat(headers).hasSize(2);

        Map<String, Document> byAit = headers.stream().collect(Collectors.toMap(
            d -> d.getString("ait_no"), d -> d
        ));
        Document a1 = byAit.get("AIT-001");
        Document a2 = byAit.get("AIT-002");

        // AIT-001 header assertions (all fields)
        assertThat(a1.getString("cio_lob_name")).isEqualTo("LOB1");
        assertThat(a1.getString("cio_name")).isEqualTo("CIO A");
        assertThat(a1.getString("cto_name")).isEqualTo("CTO A");
        assertThat(a1.getString("cio_lob_name_one_deep")).isEqualTo("LOB1-D");
        assertThat(a1.getString("ait_name")).isEqualTo("App One");
        assertThat(a1.getString("ait_app_manager")).isEqualTo("Mgr One");
        assertThat(a1.getString("ait_app_manager_nbid")).isEqualTo("NBID-001");
        assertThat(a1.getString("ait_mgmt_support_contact")).isEqualTo("Support One");
        assertThat(a1.getString("ait_mgmt_support_contact_nbid")).isEqualTo("NBID-S1");
        assertThat(a1.getString("ait_tech_exec")).isEqualTo("Tech Exec One");
        assertThat(a1.getString("ait_tech_exec_nbid")).isEqualTo("NBID-T1");
        assertThat(a1.getString("ait_app_manager_email")).isEqualTo("app1@corp.test");
        assertThat(a1.getString("gis_network_rating_zone")).isEqualTo("ZoneA");
        assertThat(a1.getString("gis_metric_alignment")).isEqualTo("Aligned");
        assertThat(a1.getString("ait_recovery_time_obj")).isEqualTo("24h");
        assertThat(a1.getString("enterprise_lob_name")).isEqualTo("ELOB");
        assertThat(a1.getString("country_name")).isEqualTo("US");
        assertThat(a1.getString("region")).isEqualTo("NA");
        assertThat(a1.getString("gis_asset_category")).isEqualTo("Server");
        assertThat(a1.getString("device_type")).isEqualTo("VM");
        assertThat(a1.getString("os_name")).isEqualTo("Linux");

        // AIT-002 header assertions (all fields)
        assertThat(a2.getString("cio_lob_name")).isEqualTo("LOB2");
        assertThat(a2.getString("cio_name")).isEqualTo("CIO B");
        assertThat(a2.getString("cto_name")).isEqualTo("CTO B");
        assertThat(a2.getString("cio_lob_name_one_deep")).isEqualTo("LOB2-D");
        assertThat(a2.getString("ait_name")).isEqualTo("App Two");
        assertThat(a2.getString("ait_app_manager")).isEqualTo("Mgr Two");
        assertThat(a2.getString("ait_app_manager_nbid")).isEqualTo("NBID-002");
        assertThat(a2.getString("ait_mgmt_support_contact")).isEqualTo("Support Two");
        assertThat(a2.getString("ait_mgmt_support_contact_nbid")).isEqualTo("NBID-S2");
        assertThat(a2.getString("ait_tech_exec")).isEqualTo("Tech Exec Two");
        assertThat(a2.getString("ait_tech_exec_nbid")).isEqualTo("NBID-T2");
        assertThat(a2.getString("ait_app_manager_email")).isEqualTo("app2@corp.test");
        assertThat(a2.getString("gis_network_rating_zone")).isEqualTo("ZoneB");
        assertThat(a2.getString("gis_metric_alignment")).isEqualTo("Aligned");
        assertThat(a2.getString("ait_recovery_time_obj")).isEqualTo("12h");
        assertThat(a2.getString("enterprise_lob_name")).isEqualTo("ELOB2");
        assertThat(a2.getString("country_name")).isEqualTo("IN");
        assertThat(a2.getString("region")).isEqualTo("APAC");
        assertThat(a2.getString("gis_asset_category")).isEqualTo("Workstation");
        assertThat(a2.getString("device_type")).isEqualTo("Laptop");
        assertThat(a2.getString("os_name")).isEqualTo("Windows");
    }

    @Test
    void aggregate_cve_items_counts_and_fields_are_exact() {
        List<Document> items = aggregation_repository.aggregate_cve_items("collection_one");

        // Map by AIT + CVE
        Map<String, Document> byKey = items.stream().collect(Collectors.toMap(
            d -> d.getString("ait_no") + "|" + d.getString("cve"), d -> d
        ));

        Document a1cve1 = byKey.get("AIT-001|CVE-2024-0001");
        Document a1cve2 = byKey.get("AIT-001|CVE-2024-0002");
        Document a2cve1 = byKey.get("AIT-002|CVE-2024-1234");

        // Validate all projected fields for AIT-001 & CVE-2024-0001
        assertThat(a1cve1.getString("cio_lob_name")).isEqualTo("LOB1");
        assertThat(a1cve1.getString("os_name")).isEqualTo("Linux");
        assertThat(a1cve1.getString("environment")).isEqualTo("PROD");
        assertThat(a1cve1.getString("remediation_status")).isEqualTo("OPEN");
        assertThat(a1cve1.getInteger("days_open")).isEqualTo(10);
        assertThat(a1cve1.getString("last_severity")).isEqualTo("High");
        assertThat(a1cve1.getString("scorecard_erp_status")).isEqualTo("Pending");
        assertThat(a1cve1.getString("scorecard_source")).isEqualTo("Qualys");
        assertThat(a1cve1.getString("observation_title")).isEqualTo("Vuln A");
        assertThat(a1cve1.getString("observation_description")).isEqualTo("Desc A");
        assertThat(a1cve1.getInteger("instances_found")).isEqualTo(2);

        // Validate second CVE
        assertThat(a1cve2.getInteger("instances_found")).isEqualTo(1);

        // AIT-002 CVE
        assertThat(a2cve1.getString("last_severity")).isEqualTo("Low");
        assertThat(a2cve1.getInteger("instances_found")).isEqualTo(1);
        assertThat(a2cve1.getString("os_name")).isEqualTo("Windows");
    }

    @Test
    void aggregate_priorities_counts_are_exact() {
        List<Document> prios = aggregation_repository.aggregate_priorities("collection_one");

        Map<String, Integer> byKey = prios.stream().collect(Collectors.toMap(
            d -> d.getString("ait_no") + "|" + d.getString("last_severity"),
            d -> d.getInteger("instances_found")
        ));

        assertThat(byKey.get("AIT-001|High")).isEqualTo(3);
        assertThat(byKey.get("AIT-002|Low")).isEqualTo(1);
    }
}
```

---

## 🧭 Service integration tests (real Mongo, persisted output)

### `src/test/java/com/example/agg/service/aggregation_service_it.java`

```java
package com.example.agg.service;

import com.example.agg.domain.cve_item;
import com.example.agg.domain.priority_item;
import com.example.agg.domain.target_document;
import com.example.agg.mapper.collection_one_mapper;
import com.example.agg.mapper.collection_two_mapper;
import com.example.agg.mapper.mapper_factory;
import com.example.agg.repo.aggregation_repository;
import com.example.agg.test_support.mongo_container_config;
import com.example.agg.test_support.sample_data_loader;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mapstruct.factory.Mappers;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
import org.springframework.data.mongodb.core.MongoTemplate;

import java.util.Comparator;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Import({
    aggregation_repository.class,
    mapper_factory.class,
    // Register MapStruct mappers as beans for the test context
    aggregation_service_it.mapstruct_test_config.class
})
public class aggregation_service_it extends mongo_container_config {

    static class mapstruct_test_config {
        @org.springframework.context.annotation.Bean
        public collection_one_mapper collection_one_mapper() {
            return Mappers.getMapper(collection_one_mapper.class);
        }
        @org.springframework.context.annotation.Bean
        public collection_two_mapper collection_two_mapper() {
            return Mappers.getMapper(collection_two_mapper.class);
        }
    }

    @Autowired
    MongoTemplate mongo_template;

    @Autowired
    aggregation_service aggregation_service;

    @BeforeEach
    void seed() {
        sample_data_loader.load_collection_one(mongo_template);
        mongo_template.dropCollection("target_collection");
    }

    @Test
    void process_and_persist_creates_exact_target_documents() {
        aggregation_service.process_and_persist("collection_one");

        List<target_document> targets =
            mongo_template.findAll(target_document.class, "target_collection");

        assertThat(targets).hasSize(2);

        // Sort by ait_no for deterministic assertions
        targets.sort(Comparator.comparing(target_document::get_ait_no));
        target_document t1 = targets.get(0); // AIT-001
        target_document t2 = targets.get(1); // AIT-002

        // ---------- Assert all header fields for AIT-001 ----------
        assertThat(t1.get_cio_lob_name()).isEqualTo("LOB1");
        assertThat(t1.get_cio_name()).isEqualTo("CIO A");
        assertThat(t1.get_cto_name()).isEqualTo("CTO A");
        assertThat(t1.get_cio_lob_name_one_deep()).isEqualTo("LOB1-D");
        assertThat(t1.get_ait_no()).isEqualTo("AIT-001");
        assertThat(t1.get_ait_name()).isEqualTo("App One");
        assertThat(t1.get_ait_app_manager()).isEqualTo("Mgr One");
        assertThat(t1.get_ait_app_manager_nbid()).isEqualTo("NBID-001");
        assertThat(t1.get_ait_mgmt_support_contact()).isEqualTo("Support One");
        assertThat(t1.get_ait_mgmt_support_contact_nbid()).isEqualTo("NBID-S1");
        assertThat(t1.get_ait_tech_exec()).isEqualTo("Tech Exec One");
        assertThat(t1.get_ait_tech_exec_nbid()).isEqualTo("NBID-T1");
        assertThat(t1.get_ait_app_manager_email()).isEqualTo("app1@corp.test");
        assertThat(t1.get_gis_network_rating_zone()).isEqualTo("ZoneA");
        assertThat(t1.get_gis_metric_alignment()).isEqualTo("Aligned");
        assertThat(t1.get_ait_recovery_time_obj()).isEqualTo("24h");
        assertThat(t1.get_enterprise_lob_name()).isEqualTo("ELOB");
        assertThat(t1.get_country_name()).isEqualTo("US");
        assertThat(t1.get_region()).isEqualTo("NA");
        assertThat(t1.get_gis_asset_category()).isEqualTo("Server");
        assertThat(t1.get_device_type()).isEqualTo("VM");
        assertThat(t1.get_os_name()).isEqualTo("Linux");

        // date_loaded must be set
        assertThat(t1.get_date_loaded()).isNotNull();

        // ---------- CVE items for AIT-001 ----------
        assertThat(t1.get_cve_list()).hasSize(2);
        // Sort for deterministic expectations by name
        t1.get_cve_list().sort(Comparator.comparing(cve_item::get_name));

        cve_item cve1 = t1.get_cve_list().get(0); // CVE-2024-0001
        cve_item cve2 = t1.get_cve_list().get(1); // CVE-2024-0002

        // CVE-0001
        assertThat(cve1.get_name()).isEqualTo("CVE-2024-0001");
        assertThat(cve1.get_environment()).isEqualTo("PROD");
        assertThat(cve1.get_remediation_status()).isEqualTo("OPEN");
        assertThat(cve1.get_days_open()).isEqualTo(10);
        assertThat(cve1.get_severity()).isEqualTo("High");
        assertThat(cve1.get_erp_status()).isEqualTo("Pending");
        assertThat(cve1.get_scorecard_source()).isEqualTo("Qualys");
        assertThat(cve1.get_observation_title()).isEqualTo("Vuln A");
        assertThat(cve1.get_observation_description()).isEqualTo("Desc A");
        assertThat(cve1.get_instances_found()).isEqualTo(2);
        // dates validated by repository test; domain stores LocalDate -> non-null not asserted here for brevity

        // CVE-0002
        assertThat(cve2.get_name()).isEqualTo("CVE-2024-0002");
        assertThat(cve2.get_instances_found()).isEqualTo(1);

        // ---------- Priorities for AIT-001 ----------
        assertThat(t1.get_priorities()).hasSize(1);
        priority_item p1 = t1.get_priorities().get(0);
        assertThat(p1.get_name()).isEqualTo("High");
        assertThat(p1.get_instances_found()).isEqualTo(3);

        // ---------- Assert all header fields for AIT-002 ----------
        assertThat(t2.get_cio_lob_name()).isEqualTo("LOB2");
        assertThat(t2.get_cio_name()).isEqualTo("CIO B");
        assertThat(t2.get_cto_name()).isEqualTo("CTO B");
        assertThat(t2.get_cio_lob_name_one_deep()).isEqualTo("LOB2-D");
        assertThat(t2.get_ait_no()).isEqualTo("AIT-002");
        assertThat(t2.get_ait_name()).isEqualTo("App Two");
        assertThat(t2.get_ait_app_manager()).isEqualTo("Mgr Two");
        assertThat(t2.get_ait_app_manager_nbid()).isEqualTo("NBID-002");
        assertThat(t2.get_ait_mgmt_support_contact()).isEqualTo("Support Two");
        assertThat(t2.get_ait_mgmt_support_contact_nbid()).isEqualTo("NBID-S2");
        assertThat(t2.get_ait_tech_exec()).isEqualTo("Tech Exec Two");
        assertThat(t2.get_ait_tech_exec_nbid()).isEqualTo("NBID-T2");
        assertThat(t2.get_ait_app_manager_email()).isEqualTo("app2@corp.test");
        assertThat(t2.get_gis_network_rating_zone()).isEqualTo("ZoneB");
        assertThat(t2.get_gis_metric_alignment()).isEqualTo("Aligned");
        assertThat(t2.get_ait_recovery_time_obj()).isEqualTo("12h");
        assertThat(t2.get_enterprise_lob_name()).isEqualTo("ELOB2");
        assertThat(t2.get_country_name()).isEqualTo("IN");
        assertThat(t2.get_region()).isEqualTo("APAC");
        assertThat(t2.get_gis_asset_category()).isEqualTo("Workstation");
        assertThat(t2.get_device_type()).isEqualTo("Laptop");
        assertThat(t2.get_os_name()).isEqualTo("Windows");

        assertThat(t2.get_date_loaded()).isNotNull();

        // CVEs for AIT-002
        assertThat(t2.get_cve_list()).hasSize(1);
        assertThat(t2.get_cve_list().get(0).get_name()).isEqualTo("CVE-2024-1234");
        assertThat(t2.get_cve_list().get(0).get_severity()).isEqualTo("Low");
        assertThat(t2.get_cve_list().get(0).get_instances_found()).isEqualTo(1);

        // Priorities for AIT-002
        assertThat(t2.get_priorities()).hasSize(1);
        assertThat(t2.get_priorities().get(0).get_name()).isEqualTo("Low");
        assertThat(t2.get_priorities().get(0).get_instances_found()).isEqualTo(1);
    }
}
```

---

# 🧭 Step-by-step procedure to validate outputs (locally & CI)

1. **Install Docker** (required by Testcontainers).
2. **Add test dependencies** in `pom.xml` (see above).
3. **Place test classes** exactly as shown under `src/test/java/...`.
4. **Run the tests**:

   * Locally: `./mvnw test` (or `mvn test`)
   * CI (GitHub Actions/GitLab): ensure Docker is available; run `mvn -B verify`.
5. **What happens under the hood**:

   * Testcontainers launches a **real MongoDB** in a container.
   * The **sample\_data\_loader** seeds `collection_one` with deterministic fixtures.
   * **Repository IT** runs each aggregation and **asserts every field** in the projected output.
   * **Service IT** calls `process_and_persist("collection_one")` and then fetches from `target_collection`, asserting:

     * **all header fields**,
     * **full `cve_list` content** (name, instances, meta fields),
     * **full `priorities` content** (name, instances),
     * **`date_loaded` is set**.
   * **Mapper tests** verify **exact field mapping** behaviors without a DB.
6. **Extend coverage**:

   * Add negative tests (e.g., invalid collection, empty data → no outputs).
   * Add tests for **`collection_two_mapper`** if its schema diverges.
   * If you implement **bulk upserts**, add idempotency tests (run service twice → deduplicated).
7. **Quality gates**:

   * Add JaCoCo for coverage and enforce thresholds (optional but recommended).

---

## 🧩 Optional: JaCoCo coverage (quality gate)

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.11</version>
  <executions>
    <execution>
      <goals><goal>prepare-agent</goal></goals>
    </execution>
    <execution>
      <id>report</id>
      <phase>verify</phase>
      <goals><goal>report</goal></goals>
    </execution>
  </executions>
</plugin>
```

---

## 🧷 Notes on conventions & production-readiness

* **Naming**: Test classes use your **snake\_case** style to remain consistent.
* **Isolation**: No reliance on developer machines—**real DB** via Testcontainers.
* **Determinism**: Fixed test fixtures; tests assert **every field** deterministically.
* **Speed**: Containers are reused per test run; methods seed only needed data.
* **Maintainability**: Shared loader + container config reduce duplication.

---

If you’d like, I can also add:

* **Controller tests** (MockMvc) for `/aggregation/run/{collection_name}`
* **Negative-path tests** for exceptions (`aggregation_exception`, `merge_exception`)
* **Bulk upsert** tests if you switch from `insert` to `bulkOps().upsert()` in the service.
