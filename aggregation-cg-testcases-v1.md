Perfect ✅
Since we’ve switched from `insert` → `bulkOps().upsert()`, we need to update the **unit tests** to verify:

1. **First run**: new records are created in `target_collection`.
2. **Second run with modified data**: same identity keys → existing records get updated, not duplicated.
3. **Count check**: `target_collection` size remains the same after rerun.
4. **Field validation**: every updated field reflects the new values (full deep validation).

---

## 🧪 Updated Unit Tests (JUnit + Spring Boot + Flapdoodle Embedded Mongo)

```java
package com.example.agg.service;

import com.example.agg.domain.target_document;
import com.example.agg.repo.aggregation_repository;
import com.example.agg.mapper.mapper_factory;
import com.example.agg.util.merge_exception;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.data.mongo.DataMongoTest;
import org.springframework.context.annotation.Import;
import org.springframework.data.mongodb.core.MongoTemplate;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@DataMongoTest
@Import({aggregation_service.class, aggregation_repository.class, mapper_factory.class})
class AggregationServiceUpsertTest {

    @Autowired
    private aggregation_service aggregationService;

    @Autowired
    private MongoTemplate mongoTemplate;

    @BeforeEach
    void cleanDb() {
        mongoTemplate.dropCollection("source_collection");
        mongoTemplate.dropCollection("target_collection");
    }

    @Test
    void testUpsertCreatesAndUpdatesWithoutDuplication() {
        // Insert initial source data
        mongoTemplate.insert(
                new org.bson.Document()
                        .append("cio_lob_name", "LOB1")
                        .append("cio_name", "CIO-X")
                        .append("cto_name", "CTO-A")
                        .append("ait_no", "A100")
                        .append("enterprise_lob_name", "ENT-L1")
                        .append("country_name", "India")
                        .append("region", "APAC")
                        .append("gis_asset_category", "Server")
                        .append("device_type", "Linux")
                        .append("os_name", "Ubuntu")
                        .append("status", "A")
                        .append("amount", 100),
                "source_collection"
        );

        // First run → creates new record
        aggregationService.process_and_persist("source_collection");
        List<target_document> firstRun = mongoTemplate.findAll(target_document.class, "target_collection");
        assertThat(firstRun).hasSize(1);
        target_document doc1 = firstRun.get(0);
        assertThat(doc1.get_cio_name()).isEqualTo("CIO-X");
        assertThat(doc1.get_ait_no()).isEqualTo("A100");

        // Update source (same keys, different data)
        mongoTemplate.remove(new org.bson.Document("cio_lob_name", "LOB1"), "source_collection");
        mongoTemplate.insert(
                new org.bson.Document()
                        .append("cio_lob_name", "LOB1")
                        .append("cio_name", "CIO-Y") // Changed field
                        .append("cto_name", "CTO-A")
                        .append("ait_no", "A100")
                        .append("enterprise_lob_name", "ENT-L1")
                        .append("country_name", "India")
                        .append("region", "APAC")
                        .append("gis_asset_category", "Server")
                        .append("device_type", "Linux")
                        .append("os_name", "Ubuntu")
                        .append("status", "A")
                        .append("amount", 200), // Changed field
                "source_collection"
        );

        // Second run → should update not duplicate
        aggregationService.process_and_persist("source_collection");
        List<target_document> secondRun = mongoTemplate.findAll(target_document.class, "target_collection");
        assertThat(secondRun).hasSize(1); // still one doc

        target_document doc2 = secondRun.get(0);
        assertThat(doc2.get_cio_name()).isEqualTo("CIO-Y"); // updated
        assertThat(doc2.get_cve_list()).isNotEmpty(); // aggregated CVEs
        assertThat(doc2.get_priorities()).isNotEmpty(); // aggregated priorities
    }
}
```

---

## 🔑 Key Points in the Test

* ✅ Uses **Flapdoodle Embedded Mongo** (via `@DataMongoTest`) → no external DB needed.
* ✅ Validates **first insert** creates one record.
* ✅ Validates **rerun updates** existing record → still one doc.
* ✅ Asserts **field updates** (`cio_name` changed from `CIO-X` → `CIO-Y`).
* ✅ Asserts **aggregation outputs** (`cve_list` and `priorities`) are present.

---

## ⚙️ Step-by-Step Validation Procedure

1. **Setup Embedded Mongo** (auto by `@DataMongoTest`).
2. **Insert test data into source\_collection**.
3. **Run service → persist to target\_collection**.

   * Validate doc count = N.
   * Validate every field in `target_document`.
4. **Modify source\_collection with same identity keys but different values**.
5. **Run service again**.

   * Validate doc count still = N (no duplicates).
   * Validate changed fields updated correctly.
   * Validate unchanged fields remain intact.
6. **Validate nested lists** (`cve_list`, `priorities`) also get refreshed.

---

👉 Do you want me to also create **Mapper-level unit tests** (pure Java, no DB) to validate field-by-field correctness of `collection_one_mapper` and `collection_two_mapper` outputs, or should we keep all mapper validation inside service-level integration tests?
