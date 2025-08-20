Perfect — thanks for sharing the updated `aggregation-cg-v2.md`!
Right now your `aggregation_service` persists with:

```java
mongo_template.insert(out, "target_collection");
```

You asked to **switch this to `bulkOps().upsert()`**, which is the correct production-level approach for deduplication (avoid duplicates on reruns) and efficient batch writes.

Here’s the revised **`process_and_persist`** method:

```java
public void process_and_persist(String collection_name) {
    log.info("aggregation start: collection={}", collection_name);

    List<Document> header_docs = aggregation_repository.aggregate_header(collection_name);
    List<Document> cve_docs = aggregation_repository.aggregate_cve_items(collection_name);
    List<Document> prio_docs = aggregation_repository.aggregate_priorities(collection_name);

    Map<String, target_document> by_key = new LinkedHashMap<>();

    try {
        for (Document h : header_docs) {
            String key = identity_key(h);
            target_document t = mapper_factory.to_target_header(collection_name, h);
            t.set_cve_list(new ArrayList<>());
            t.set_priorities(new ArrayList<>());
            t.set_date_loaded(LocalDateTime.now());
            by_key.put(key, t);
        }

        Map<String, List<Document>> cve_by_key = cve_docs.stream()
                .collect(Collectors.groupingBy(this::identity_key));
        for (Map.Entry<String, List<Document>> e : cve_by_key.entrySet()) {
            target_document t = by_key.get(e.getKey());
            if (t == null) continue;
            List<cve_item> cves = mapper_factory.to_cve_list(collection_name, e.getValue());
            t.get_cve_list().addAll(cves);
        }

        Map<String, List<Document>> prio_by_key = prio_docs.stream()
                .collect(Collectors.groupingBy(this::identity_key));
        for (Map.Entry<String, List<Document>> e : prio_by_key.entrySet()) {
            target_document t = by_key.get(e.getKey());
            if (t == null) continue;
            List<priority_item> ps = mapper_factory.to_priority_list(collection_name, e.getValue());
            t.get_priorities().addAll(ps);
        }

    } catch (Exception ex) {
        log.error("merge error: {}", ex.getMessage(), ex);
        throw new merge_exception("failed to merge aggregated parts", ex);
    }

    List<target_document> out = new ArrayList<>(by_key.values());
    if (!out.isEmpty()) {
        BulkOperations bulkOps = mongo_template.bulkOps(BulkOperations.BulkMode.UNORDERED, target_document.class, "target_collection");

        for (target_document doc : out) {
            Query query = new Query(Criteria.where("cio_lob_name").is(doc.get_cio_lob_name())
                    .and("cio_name").is(doc.get_cio_name())
                    .and("cto_name").is(doc.get_cto_name())
                    .and("cio_lob_name_one_deep").is(doc.get_cio_lob_name_one_deep())
                    .and("ait_no").is(doc.get_ait_no())
                    .and("enterprise_lob_name").is(doc.get_enterprise_lob_name())
                    .and("country_name").is(doc.get_country_name())
                    .and("region").is(doc.get_region())
                    .and("gis_asset_category").is(doc.get_gis_asset_category())
                    .and("device_type").is(doc.get_device_type())
                    .and("os_name").is(doc.get_os_name())
            );

            Update update = new Update()
                    .set("cio_lob_name", doc.get_cio_lob_name())
                    .set("cio_name", doc.get_cio_name())
                    .set("cto_name", doc.get_cto_name())
                    .set("cio_lob_name_one_deep", doc.get_cio_lob_name_one_deep())
                    .set("ait_no", doc.get_ait_no())
                    .set("ait_name", doc.get_ait_name())
                    .set("ait_app_manager", doc.get_ait_app_manager())
                    .set("ait_app_manager_nbid", doc.get_ait_app_manager_nbid())
                    .set("ait_mgmt_support_contact", doc.get_ait_mgmt_support_contact())
                    .set("ait_mgmt_support_contact_nbid", doc.get_ait_mgmt_support_contact_nbid())
                    .set("ait_tech_exec", doc.get_ait_tech_exec())
                    .set("ait_tech_exec_nbid", doc.get_ait_tech_exec_nbid())
                    .set("ait_app_manager_email", doc.get_ait_app_manager_email())
                    .set("gis_network_rating_zone", doc.get_gis_network_rating_zone())
                    .set("gis_metric_alignment", doc.get_gis_metric_alignment())
                    .set("ait_recovery_time_obj", doc.get_ait_recovery_time_obj())
                    .set("enterprise_lob_name", doc.get_enterprise_lob_name())
                    .set("country_name", doc.get_country_name())
                    .set("region", doc.get_region())
                    .set("gis_asset_category", doc.get_gis_asset_category())
                    .set("device_type", doc.get_device_type())
                    .set("os_name", doc.get_os_name())
                    .set("cve_list", doc.get_cve_list())
                    .set("priorities", doc.get_priorities())
                    .set("date_loaded", doc.get_date_loaded());

            bulkOps.upsert(query, update);
        }

        bulkOps.execute();
        log.info("aggregation done: upserted {} docs into target_collection", out.size());
    }
}
```

### ✅ Key Changes

* Replaced `insert()` with **`bulkOps(BulkMode.UNORDERED, target_document.class, "target_collection")`**.
* Added **`upsert(query, update)`** for each document, using the identity fields as the query.
* This ensures **idempotency** (reruns won’t create duplicates, they’ll update existing records).

---

Would you like me to also:

1. Update your **unit tests** to validate the new `upsert` behavior (checking that updates work correctly instead of duplicate inserts)?
2. Or keep them as-is and only add new tests for the `bulkOps` logic?
