Got it 👍 — you want the **entire aggregation flow** re-written using Spring Data MongoDB’s `@Aggregation` annotation (instead of `MongoTemplate`).
That way the repository returns strongly-typed POJOs, ready for mapping with MapStruct if needed.

Here’s a **complete, production-style solution** with:

* Entities (`SourceDocument`, `TargetDocument`, nested `CveItem`, `PriorityItem`)
* Repository with `@Aggregation` annotation
* Service layer (persists to a target collection)
* Logging + error handling

All fields stay in **snake\_case** as you asked.

---

## 📌 Entities

### SourceDocument.java

```java
package com.example.mongo.model;

import lombok.Data;
import org.springframework.data.mongodb.core.mapping.Document;

@Data
@Document(collection = "source_collection")
public class SourceDocument {
    private String cio_lob_name;
    private String cio_name;
    private String cto_name;
    private String cio_lob_name_one_deep;

    private String ait_no;
    private String ait_name;
    private String ait_app_manager;
    private String ait_app_manager_nbid;

    private String cve;
    private String first_reported;
    private String due_date;
    private Boolean past_due;
    private String date_detected;
    private String environment;
    private String remediation_status;
    private Integer days_open;
    private String severity;
    private String erp_status;
    private String scorecard_source;
    private String observation_title;
    private String observation_description;
    private Integer instances_found;
}
```

---

### TargetDocument.java

```java
package com.example.mongo.model;

import lombok.Data;
import org.springframework.data.mongodb.core.mapping.Document;

import java.util.List;

@Data
@Document(collection = "target_collection")
public class TargetDocument {
    private String cio_lob_name;
    private String cio_name;
    private String cto_name;
    private String cio_lob_name_one_deep;

    private String ait_no;
    private String ait_name;
    private String ait_app_manager;
    private String ait_app_manager_nbid;

    private List<CveItem> cve_list;
    private List<PriorityItem> priorities;

    private Integer total_instances;
    private Integer count;
    private String date_loaded;
}
```

---

### CveItem.java

```java
package com.example.mongo.model;

import lombok.Data;

@Data
public class CveItem {
    private String name;
    private String first_reported;
    private String due_date;
    private Boolean past_due;
    private String date_detected;
    private String environment;
    private String remediation_status;
    private Integer days_open;
    private String severity;
    private String erp_status;
    private String scorecard_source;
    private String observation_title;
    private String observation_description;
    private Integer instances_found;
}
```

---

### PriorityItem.java

```java
package com.example.mongo.model;

import lombok.Data;

@Data
public class PriorityItem {
    private String name;
    private Integer instances_found;
}
```

---

## 📌 Repository with @Aggregation

### AggregationRepository.java

```java
package com.example.mongo.repo;

import com.example.mongo.model.TargetDocument;
import org.springframework.data.mongodb.repository.Aggregation;
import org.springframework.data.mongodb.repository.MongoRepository;

import java.util.List;

public interface AggregationRepository extends MongoRepository<TargetDocument, String> {

    @Aggregation(pipeline = {
        "{ $group: { " +
            "_id: { " +
                "cio_lob_name: '$cio_lob_name', " +
                "cio_name: '$cio_name', " +
                "cto_name: '$cto_name', " +
                "cio_lob_name_one_deep: '$cio_lob_name_one_deep', " +
                "ait_no: '$ait_no', " +
                "ait_name: '$ait_name', " +
                "ait_app_manager: '$ait_app_manager', " +
                "ait_app_manager_nbid: '$ait_app_manager_nbid' " +
            "}, " +
            "cve_list: { $push: { " +
                "name: '$cve', " +
                "first_reported: '$first_reported', " +
                "due_date: '$due_date', " +
                "past_due: '$past_due', " +
                "date_detected: '$date_detected', " +
                "environment: '$environment', " +
                "remediation_status: '$remediation_status', " +
                "days_open: '$days_open', " +
                "severity: '$severity', " +
                "erp_status: '$erp_status', " +
                "scorecard_source: '$scorecard_source', " +
                "observation_title: '$observation_title', " +
                "observation_description: '$observation_description', " +
                "instances_found: '$instances_found' " +
            "} }, " +
            "priorities: { $push: { " +
                "name: '$severity', " +
                "instances_found: '$instances_found' " +
            "} }, " +
            "total_instances: { $sum: '$instances_found' }, " +
            "count: { $sum: 1 } " +
        "} }, " +
        "{ $project: { " +
            "cio_lob_name: '$_id.cio_lob_name', " +
            "cio_name: '$_id.cio_name', " +
            "cto_name: '$_id.cto_name', " +
            "cio_lob_name_one_deep: '$_id.cio_lob_name_one_deep', " +
            "ait_no: '$_id.ait_no', " +
            "ait_name: '$_id.ait_name', " +
            "ait_app_manager: '$_id.ait_app_manager', " +
            "ait_app_manager_nbid: '$_id.ait_app_manager_nbid', " +
            "cve_list: 1, " +
            "priorities: 1, " +
            "total_instances: 1, " +
            "count: 1, " +
            "_id: 0 " +
        "} }"
    })
    List<TargetDocument> aggregateData();
}
```

---

## 📌 Service Layer

### AggregationService.java

```java
package com.example.mongo.service;

import com.example.mongo.model.TargetDocument;
import com.example.mongo.repo.AggregationRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.List;

@Service
@RequiredArgsConstructor
@Slf4j
public class AggregationService {

    private final AggregationRepository aggregation_repository;
    private final MongoTemplate mongo_template;

    public void process_and_persist() {
        log.info("Starting aggregation pipeline...");

        try {
            List<TargetDocument> results = aggregation_repository.aggregateData();

            if (results.isEmpty()) {
                log.warn("No documents aggregated");
                return;
            }

            // add a timestamp field before persisting
            results.forEach(r -> r.setDate_loaded(Instant.now().toString()));

            mongo_template.insert(results, "target_collection");
            log.info("Persisted {} aggregated documents into target_collection", results.size());

        } catch (Exception e) {
            log.error("Aggregation failed", e);
            throw new RuntimeException("Aggregation pipeline failed", e);
        }
    }
}
```

---

✅ What this solution gives you:

* Clean **@Aggregation annotation** instead of `MongoTemplate`.
* Strongly typed **POJOs** (`TargetDocument`, `CveItem`, `PriorityItem`).
* **Snake\_case fields** preserved everywhere.
* **Service layer** with logging + error handling.
* Ready for **MapStruct mapping** if you need additional DTOs.

---

Would you like me to also add a **MapStruct mapper** that maps `TargetDocument` → `TargetDTO` (so your API layer doesn’t expose DB schema directly)?
