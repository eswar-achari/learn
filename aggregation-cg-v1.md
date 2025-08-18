Absolutely—I'll keep **snake\_case everywhere in Java**, matching your Mongo/source fields 1:1. Below are the key pieces rewritten to use snake\_case for **class names (normal Java)** but **all fields, getters/setters, method params, variables in snake\_case**. This avoids any implicit renaming in MapStruct or Spring Data.

---

# Target Mongo entity (snake\_case fields)

```java
@Document(collection = "target_collection")
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class TargetDocument {

    // top-level attributes
    private String cio_lob_name;
    private String cio_name;
    private String cto_name;
    private String cio_lob_name_one_deep;

    private String ait_no;
    private String ait_name;
    private String ait_app_manager;
    private String ait_app_manager_nbid;
    private String ait_mgmt_support_contact;
    private String ait_mgmt_support_contact_nbid;
    private String ait_tech_exec;
    private String ait_tech_exec_nbid;
    private String ait_app_manager_email;

    private String gis_network_rating_zone;
    private String gis_metric_alignment;
    private String ait_recovery_time_obj;

    private String enterprise_lob_name;
    private String country_name;
    private String region;
    private String gis_asset_category;
    private String device_type;
    private String os_name;

    private List<CveItem> cve_list;
    private List<PriorityItem> priorities;

    private LocalDateTime date_loaded;
}

@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class CveItem {
    private String name;
    private LocalDate first_reported;
    private LocalDate due_date;
    private boolean past_due;
    private LocalDate date_detected;
    private String environment;
    private String remediation_status;
    private int days_open;
    private String severity;
    private String erp_status;
    private String scorecard_source;
    private String observation_title;
    private String observation_description;
    private int instances_found;
}

@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class PriorityItem {
    private String name;
    private int instances_found;
}
```

> Note: Because field names already match Mongo snake\_case, we don’t need `@Field("...")`. If you ever diverge, add `@Field("mongo_field_name")` per field.

---

# MapStruct (snake\_case → snake\_case)

Since names match exactly, we mostly map by name. You can still be explicit where needed.

```java
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.ERROR)
public interface CollectionOneMapper {
    TargetDocument to_target(Document doc);

    // Example when mapping nested/renamed fields inside CveItem
    @Mapping(target = "name", source = "cve")
    @Mapping(target = "first_reported", source = "first_reported")
    CveItem to_cve_item(Document doc);

    default List<CveItem> to_cve_list(List<Document> docs) {
        if (docs == null) return Collections.emptyList();
        return docs.stream().map(this::to_cve_item).collect(Collectors.toList());
    }
}

@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.ERROR)
public interface CollectionTwoMapper {
    TargetDocument to_target(Document doc);
}
```

* Method names also in snake\_case (`to_target`, `to_cve_item`) per your requirement.
* `ReportingPolicy.ERROR` ensures we don’t silently miss fields.

---

# Dynamic mapper factory (snake\_case)

```java
@Component
@RequiredArgsConstructor
public class MapperFactory {

    private final CollectionOneMapper collection_one_mapper;
    private final CollectionTwoMapper collection_two_mapper;

    public TargetDocument map(String collection_name, Document source_doc) {
        switch (collection_name) {
            case "collection_one":
                return collection_one_mapper.to_target(source_doc);
            case "collection_two":
                return collection_two_mapper.to_target(source_doc);
            default:
                throw new IllegalArgumentException("no mapper for collection: " + collection_name);
        }
    }
}
```

---

# Aggregation repository (snake\_case fields in pipeline)

```java
@Repository
@RequiredArgsConstructor
public class AggregationRepository {

    private final MongoTemplate mongo_template;

    public List<Document> aggregate_source_data(String collection_name) {
        try {
            // example grouping + SUM/COUNT; extend as needed
            Aggregation agg = Aggregation.newAggregation(
                Aggregation.group("cio_lob_name", "cio_name", "cto_name", "cve")
                    .sum("instances_found").as("instances_found")
                    .count().as("total_count"),
                Aggregation.project()
                    .and("_id.cio_lob_name").as("cio_lob_name")
                    .and("_id.cio_name").as("cio_name")
                    .and("_id.cto_name").as("cto_name")
                    .and("_id.cve").as("cve")
                    .andInclude("instances_found", "total_count")
                    .andExclude("_id")
            );
            return mongo_template.aggregate(agg, collection_name, Document.class).getMappedResults();
        } catch (Exception e) {
            throw new AggregationException("aggregate error for " + collection_name, e);
        }
    }
}
```

---

# Service (snake\_case, logging, error handling)

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class AggregationService {

    private final AggregationRepository aggregation_repository;
    private final MapperFactory mapper_factory;
    private final MongoTemplate mongo_template;

    public void process_and_persist(String collection_name) {
        log.info("aggregation start: collection={}", collection_name);
        try {
            List<Document> docs = aggregation_repository.aggregate_source_data(collection_name);

            List<TargetDocument> mapped = docs.stream()
                .map(doc -> {
                    // attach cve_list/priorities if you emitted arrays from pipeline
                    TargetDocument t = mapper_factory.map(collection_name, doc);
                    t.set_date_loaded(LocalDateTime.now());
                    return t;
                })
                .collect(Collectors.toList());

            if (!mapped.isEmpty()) {
                mongo_template.insert(mapped, "target_collection");
            }
            log.info("persisted {} docs into target_collection", mapped.size());
        } catch (AggregationException ex) {
            log.error("aggregation failed: collection={}, msg={}", collection_name, ex.getMessage(), ex);
        } catch (Exception ex) {
            log.error("unexpected error: collection={}, msg={}", collection_name, ex.getMessage(), ex);
        }
    }
}
```

---

# Custom exception (snake\_case)

```java
public class AggregationException extends RuntimeException {
    public AggregationException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

---

## Notes on production conventions when using snake\_case in Java

* **Checkstyle/Spotless:** update rules to allow `^[a-z0-9_]+$` for member/param/local names.
* **Lombok:** generated methods will still be `getCio_lob_name()` style; that’s fine since you control the codebase.
* **Jackson (if exposing as JSON):** you’re already in snake\_case, but you can still add:

  ```java
  @JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)
  ```

  at class level to keep consistent if any future fields drift.
* **Spring Data:** since field names match Mongo keys, no aliasing needed. If you ever change a Java name, add `@Field("mongo_key")`.

If you want, I can also push a **complete aggregation pipeline** that builds the `cve_list` and `priorities` arrays inside Mongo (using `$group`, `$push`, `$sum`) so the mapper just picks them up—still all in snake\_case.
