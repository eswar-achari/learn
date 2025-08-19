Got it 👍 — you want the same **aggregation + dynamic MapStruct mapping + persistence** solution, but instead of building aggregations programmatically with `MongoTemplate`, you’d like to use **Spring Data MongoDB’s `@Aggregation` annotation** on repository interfaces.

That way:

* queries live in repository interfaces
* pipelines are declarative JSON arrays (readable, easy to maintain)
* results are mapped directly into `org.bson.Document` (or DTOs)

---

## ✅ Example: Repository with `@Aggregation`

Here’s how you can re-express the three aggregations (header, CVE list, priorities) using `@Aggregation`:

```java
package com.example.agg.repo;

import org.bson.Document;
import org.springframework.data.mongodb.repository.Aggregation;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface source_collection_repo extends MongoRepository<Document, String> {

    // ---------- HEADER ----------
    @Aggregation(pipeline = {
        "{ '$group': { " +
            " '_id': { " +
            "   'cio_lob_name': '$cio_lob_name', 'cio_name': '$cio_name', 'cto_name': '$cto_name', " +
            "   'cio_lob_name_one_deep': '$cio_lob_name_one_deep', 'ait_no': '$ait_no', " +
            "   'enterprise_lob_name': '$enterprise_lob_name', 'country_name': '$country_name', " +
            "   'region': '$region', 'gis_asset_category': '$gis_asset_category', " +
            "   'device_type': '$device_type', 'os_name': '$os_name' }, " +
            " 'cio_lob_name': { '$first': '$cio_lob_name' }, " +
            " 'cio_name': { '$first': '$cio_name' }, " +
            " 'cto_name': { '$first': '$cto_name' }, " +
            " 'cio_lob_name_one_deep': { '$first': '$cio_lob_name_one_deep' }, " +
            " 'ait_no': { '$first': '$ait_no' }, " +
            " 'ait_name': { '$first': '$ait_name' }, " +
            " 'ait_app_manager': { '$first': '$ait_app_manager' }, " +
            " 'ait_app_manager_nbid': { '$first': '$ait_app_manager_nbid' }, " +
            " 'ait_mgmt_support_contact': { '$first': '$ait_mgmt_support_contact' }, " +
            " 'ait_mgmt_support_contact_nbid': { '$first': '$ait_mgmt_support_contact_nbid' }, " +
            " 'ait_tech_exec': { '$first': '$ait_tech_exec' }, " +
            " 'ait_tech_exec_nbid': { '$first': '$ait_tech_exec_nbid' }, " +
            " 'ait_app_manager_email': { '$first': '$ait_app_manager_email' }, " +
            " 'gis_network_rating_zone': { '$first': '$gis_network_rating_zone' }, " +
            " 'gis_metric_alignment': { '$first': '$gis_metric_alignment' }, " +
            " 'ait_recovery_time_obj': { '$first': '$ait_recovery_time_obj' }, " +
            " 'enterprise_lob_name': { '$first': '$enterprise_lob_name' }, " +
            " 'country_name': { '$first': '$country_name' }, " +
            " 'region': { '$first': '$region' }, " +
            " 'gis_asset_category': { '$first': '$gis_asset_category' }, " +
            " 'device_type': { '$first': '$device_type' }, " +
            " 'os_name': { '$first': '$os_name' } } }, " +
        "{ '$project': { '_id': 0 } }"
    })
    List<Document> aggregate_header();

    // ---------- CVE LIST ----------
    @Aggregation(pipeline = {
        "{ '$group': { " +
            " '_id': { " +
            "   'cio_lob_name': '$cio_lob_name', 'cio_name': '$cio_name', 'cto_name': '$cto_name', " +
            "   'cio_lob_name_one_deep': '$cio_lob_name_one_deep', 'ait_no': '$ait_no', " +
            "   'enterprise_lob_name': '$enterprise_lob_name', 'country_name': '$country_name', " +
            "   'region': '$region', 'gis_asset_category': '$gis_asset_category', " +
            "   'device_type': '$device_type', 'os_name': '$os_name', 'cve': '$cve' }, " +
            " 'first_reported': { '$first': '$first_reported' }, " +
            " 'due_date': { '$first': '$due_date' }, " +
            " 'past_due': { '$first': '$past_due' }, " +
            " 'date_detected': { '$first': '$date_detected' }, " +
            " 'environment': { '$first': '$environment' }, " +
            " 'remediation_status': { '$first': '$remediation_status' }, " +
            " 'days_open': { '$first': '$days_open' }, " +
            " 'last_severity': { '$first': '$last_severity' }, " +
            " 'scorecard_erp_status': { '$first': '$scorecard_erp_status' }, " +
            " 'scorecard_source': { '$first': '$scorecard_source' }, " +
            " 'observation_title': { '$first': '$observation_title' }, " +
            " 'observation_description': { '$first': '$observation_description' }, " +
            " 'instances_found': { '$sum': 1 } } }, " +
        "{ '$project': { " +
            " 'cio_lob_name': '$_id.cio_lob_name', 'cio_name': '$_id.cio_name', 'cto_name': '$_id.cto_name', " +
            " 'cio_lob_name_one_deep': '$_id.cio_lob_name_one_deep', 'ait_no': '$_id.ait_no', " +
            " 'enterprise_lob_name': '$_id.enterprise_lob_name', 'country_name': '$_id.country_name', " +
            " 'region': '$_id.region', 'gis_asset_category': '$_id.gis_asset_category', " +
            " 'device_type': '$_id.device_type', 'os_name': '$_id.os_name', 'cve': '$_id.cve', " +
            " 'first_reported': 1, 'due_date': 1, 'past_due': 1, 'date_detected': 1, " +
            " 'environment': 1, 'remediation_status': 1, 'days_open': 1, 'last_severity': 1, " +
            " 'scorecard_erp_status': 1, 'scorecard_source': 1, 'observation_title': 1, " +
            " 'observation_description': 1, 'instances_found': 1, '_id': 0 } }"
    })
    List<Document> aggregate_cve_items();

    // ---------- PRIORITIES ----------
    @Aggregation(pipeline = {
        "{ '$group': { " +
            " '_id': { " +
            "   'cio_lob_name': '$cio_lob_name', 'cio_name': '$cio_name', 'cto_name': '$cto_name', " +
            "   'cio_lob_name_one_deep': '$cio_lob_name_one_deep', 'ait_no': '$ait_no', " +
            "   'enterprise_lob_name': '$enterprise_lob_name', 'country_name': '$country_name', " +
            "   'region': '$region', 'gis_asset_category': '$gis_asset_category', " +
            "   'device_type': '$device_type', 'os_name': '$os_name', 'last_severity': '$last_severity' }, " +
            " 'instances_found': { '$sum': 1 } } }, " +
        "{ '$project': { " +
            " 'cio_lob_name': '$_id.cio_lob_name', 'cio_name': '$_id.cio_name', 'cto_name': '$_id.cto_name', " +
            " 'cio_lob_name_one_deep': '$_id.cio_lob_name_one_deep', 'ait_no': '$_id.ait_no', " +
            " 'enterprise_lob_name': '$_id.enterprise_lob_name', 'country_name': '$_id.country_name', " +
            " 'region': '$_id.region', 'gis_asset_category': '$_id.gis_asset_category', " +
            " 'device_type': '$_id.device_type', 'os_name': '$_id.os_name', " +
            " 'last_severity': '$_id.last_severity', 'instances_found': 1, '_id': 0 } }"
    })
    List<Document> aggregate_priorities();
}
```

---

## 🔄 Service Layer

Your `aggregation_service` stays almost identical, except now you inject the `source_collection_repo` and call the annotation-based methods instead of `MongoTemplate`.

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class aggregation_service {

    private final source_collection_repo repo;
    private final mapper_factory mapper_factory;
    private final MongoTemplate mongo_template; // still used for persisting final target docs

    public void process_and_persist() {
        log.info("aggregation start with @Aggregation repo");

        List<Document> header_docs = repo.aggregate_header();
        List<Document> cve_docs = repo.aggregate_cve_items();
        List<Document> prio_docs = repo.aggregate_priorities();

        // merge + map same as before …
    }
}
```

---

⚠️ Note: `@Aggregation` is **static per repository**, so if you truly want **dynamic collection switching** (collection\_one, collection\_two, …), you’ll either:

* create **separate repos per source collection** (recommended) and inject dynamically
* or keep `MongoTemplate` for dynamic collection names

---

👉 Do you want me to **refactor the full service + mapper\_factory** so that it can dynamically choose between multiple `@Aggregation`-based repositories (`collection_one_repo`, `collection_two_repo`) instead of `MongoTemplate`?
