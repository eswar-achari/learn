// ─────────────────────────────────────────────────────────────────────────────
// Project: mongo-agg-mapstruct
// Description: Dynamic MapStruct-based field mapping + Mongo aggregation service
//              with SUM/COUNT support and persistence to a new collection.
// Tech: Spring Boot, Spring Data MongoDB, MapStruct, Lombok, Jakarta Validation
// ─────────────────────────────────────────────────────────────────────────────

// ================================
// File: pom.xml
// ================================
/*
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>mongo-agg-mapstruct</artifactId>
  <version>1.0.0</version>
  <properties>
    <java.version>17</java.version>
    <mapstruct.version>1.5.5.Final</mapstruct.version>
    <spring-boot.version>3.3.2</spring-boot.version>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>${spring-boot.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-mongodb</artifactId>
    </dependency>
    <dependency>
      <groupId>org.mapstruct</groupId>
      <artifactId>mapstruct</artifactId>
      <version>${mapstruct.version}</version>
    </dependency>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <optional>true</optional>
    </dependency>
    <dependency>
      <groupId>jakarta.validation</groupId>
      <artifactId>jakarta.validation-api</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.mapstruct</groupId>
      <artifactId>mapstruct-processor</artifactId>
      <version>${mapstruct.version}</version>
      <scope>provided</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.11.0</version>
        <configuration>
          <source>${java.version}</source>
          <target>${java.version}</target>
          <annotationProcessorPaths>
            <path>
              <groupId>org.mapstruct</groupId>
              <artifactId>mapstruct-processor</artifactId>
              <version>${mapstruct.version}</version>
            </path>
            <path>
              <groupId>org.projectlombok</groupId>
              <artifactId>lombok</artifactId>
              <version>1.18.32</version>
            </path>
          </annotationProcessorPaths>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
*/

// ================================
// File: src/main/java/com/example/agg/Application.java
// ================================
package com.example.agg;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
  public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
  }
}

// ================================
// File: src/main/java/com/example/agg/domain/SourceType.java
// ================================
package com.example.agg.domain;

public enum SourceType {
  ORDERS, USERS
}

// ================================
// File: src/main/java/com/example/agg/domain/NormalizedRecord.java
// ================================
package com.example.agg.domain;

import lombok.*;
import jakarta.validation.constraints.*;

/** Canonical model that mappers map into (pre-aggregation projection schema). */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class NormalizedRecord {
  /** Optional grouping key fields (example) */
  @NotBlank
  private String tenantId;

  /** Common amount field to be summed */
  @PositiveOrZero
  private Double amount;

  /** Optional category */
  private String category;

  /** Timestamp field */
  private java.time.Instant eventTime;
}

// ================================
// File: src/main/java/com/example/agg/domain/AggregatedResult.java
// ================================
package com.example.agg.domain;

import lombok.*;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class AggregatedResult {
  private String tenantId;
  private String category;     // can be null if not in group-by
  private long count;          // COUNT(*), or COUNT docs per group
  private double totalAmount;  // SUM(amount)
}

// ================================
// File: src/main/java/com/example/agg/mapping/DynamicMapper.java
// ================================
package com.example.agg.mapping;

import com.example.agg.domain.NormalizedRecord;
import org.bson.Document;
import java.util.Map;

/**
 * Strategy contract for source-specific mapping.
 * Implementations:
 *  - Provide a MapStruct-powered Java mapper for post-aggregation mapping when needed
 *  - Provide a projection spec for Mongo $project to align source fields to canonical names
 */
public interface DynamicMapper {
  /** Map a raw Mongo Document (already projected) to the canonical NormalizedRecord. */
  NormalizedRecord toNormalized(Document projected);

  /**
   * Field projection: key=canonicalField, value=sourceFieldPath
   * e.g., {"tenantId":"tenant_id", "amount":"price.total", "eventTime":"createdAt"}
   */
  Map<String, String> projectionSpec();
}

// ================================
// File: src/main/java/com/example/agg/mapping/OrdersMapper.java
// ================================
package com.example.agg.mapping;

import com.example.agg.domain.NormalizedRecord;
import org.bson.Document;
import org.mapstruct.*;
import org.springframework.stereotype.Component;

@Mapper(componentModel = "spring")
public interface OrdersMapper extends DynamicMapper {

  @Override
  @Mapping(target = "tenantId", expression = "java(doc.getString(\"tenant_id\"))")
  @Mapping(target = "amount", expression = "java( doc.getEmbedded(java.util.List.of(\"price\", \"total\"), Number.class) == null ? 0d : doc.getEmbedded(java.util.List.of(\"price\", \"total\"), Number.class).doubleValue() )")
  @Mapping(target = "category", expression = "java(doc.getString(\"segment\"))")
  @Mapping(target = "eventTime", expression = "java( java.time.Instant.ofEpochMilli( doc.getDate(\"createdAt\").getTime()) )")
  NormalizedRecord toNormalized(Document doc);

  @Override
  default java.util.Map<String, String> projectionSpec() {
    return java.util.Map.of(
        "tenantId", "tenant_id",
        "amount", "price.total",
        "category", "segment",
        "eventTime", "createdAt"
    );
  }
}

// ================================
// File: src/main/java/com/example/agg/mapping/UsersMapper.java
// ================================
package com.example.agg.mapping;

import com.example.agg.domain.NormalizedRecord;
import org.bson.Document;
import org.mapstruct.*;

@Mapper(componentModel = "spring")
public interface UsersMapper extends DynamicMapper {

  @Override
  @Mapping(target = "tenantId", expression = "java(doc.getString(\"tenant\"))")
  @Mapping(target = "amount", constant = "0.0") // Users collection has no amount
  @Mapping(target = "category", expression = "java(doc.getString(\"plan\"))")
  @Mapping(target = "eventTime", expression = "java( java.time.Instant.ofEpochSecond( ((Number)doc.getOrDefault(\"signupEpoch\", 0)).longValue()) )")
  com.example.agg.domain.NormalizedRecord toNormalized(org.bson.Document doc);

  @Override
  default java.util.Map<String, String> projectionSpec() {
    return java.util.Map.of(
        "tenantId", "tenant",
        "amount", "0",           // constant 0 for amount
        "category", "plan",
        "eventTime", "signupEpoch" // epoch seconds
    );
  }
}

// ================================
// File: src/main/java/com/example/agg/mapping/MapperRegistry.java
// ================================
package com.example.agg.mapping;

import com.example.agg.domain.SourceType;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

import java.util.EnumMap;
import java.util.Map;

@Component
@RequiredArgsConstructor
public class MapperRegistry {
  private final OrdersMapper ordersMapper;
  private final UsersMapper usersMapper;

  private Map<SourceType, DynamicMapper> cache;

  public DynamicMapper resolve(SourceType type) {
    if (cache == null) {
      cache = new EnumMap<>(SourceType.class);
      cache.put(SourceType.ORDERS, ordersMapper);
      cache.put(SourceType.USERS, usersMapper);
    }
    DynamicMapper mapper = cache.get(type);
    if (mapper == null) throw new IllegalArgumentException("Unsupported source type: " + type);
    return mapper;
  }
}

// ================================
// File: src/main/java/com/example/agg/service/AggregationSpec.java
// ================================
package com.example.agg.service;

import lombok.*;
import java.util.*;

/** Specification for grouping and aggregations. */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class AggregationSpec {
  /** Fields (canonical) to group by, e.g., ["tenantId", "category"] */
  @Builder.Default
  private List<String> groupBy = new ArrayList<>();

  /** Canonical numeric fields to SUM, e.g., ["amount"] */
  @Builder.Default
  private List<String> sumFields = new ArrayList<>();

  /** Whether to compute COUNT documents per group. */
  @Builder.Default
  private boolean includeCount = true;
}

// ================================
// File: src/main/java/com/example/agg/service/MongoAggregationService.java
// ================================
package com.example.agg.service;

import com.example.agg.domain.AggregatedResult;
import com.example.agg.domain.NormalizedRecord;
import com.example.agg.domain.SourceType;
import com.example.agg.mapping.DynamicMapper;
import com.example.agg.mapping.MapperRegistry;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.bson.Document;
import org.springframework.dao.DataAccessException;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.aggregation.*;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.*;
import java.util.stream.Collectors;

import static org.springframework.data.mongodb.core.aggregation.Aggregation.*;

@Slf4j
@Service
@RequiredArgsConstructor
public class MongoAggregationService {

  private final MongoTemplate mongoTemplate;
  private final MapperRegistry mapperRegistry;

  /**
   * Execute aggregation on a source collection, using the DynamicMapper for that source,
   * apply SUM/COUNT per spec, and persist results to target collection.
   *
   * @param sourceType enum indicating which mapper/collection schema to use
   * @param sourceCollection actual Mongo collection name
   * @param targetCollection destination collection to persist aggregated results
   * @param spec aggregation spec
   * @param matchCriteria optional filter (can be null)
   * @return list of AggregatedResult also persisted to targetCollection
   */
  public List<AggregatedResult> aggregateAndPersist(
      SourceType sourceType,
      String sourceCollection,
      String targetCollection,
      AggregationSpec spec,
      Criteria matchCriteria
  ) {
    long start = System.currentTimeMillis();
    DynamicMapper mapper = mapperRegistry.resolve(sourceType);

    try {
      // 1) $project to canonical fields as strings/values projected from source
      Document projectDoc = buildProjectDocument(mapper.projectionSpec());
      ProjectionOperation project = new ProjectionOperation(projectDoc);

      // 2) optional $match
      MatchOperation match = (matchCriteria != null) ? match(matchCriteria) : null;

      // 3) $group according to spec
      GroupOperation group = groupBy(spec.getGroupBy());
      if (spec.isIncludeCount()) group = group.count().as("count");
      for (String sumField : spec.getSumFields()) {
        group = group.sum("$" + sumField).as("sum_" + sumField);
      }

      // 4) Build pipeline
      List<AggregationOperation> ops = new ArrayList<>();
      ops.add(project);
      if (match != null) ops.add(match);
      ops.add(group);

      Aggregation agg = newAggregation(ops);
      log.info("Executing aggregation on '{}' with ops={} spec={}", sourceCollection, ops.size(), spec);

      AggregationResults<Document> results = mongoTemplate.aggregate(agg, sourceCollection, Document.class);

      // 5) Map group output Documents to AggregatedResult
      List<AggregatedResult> mapped = results.getMappedResults().stream()
          .map(this::toAggregatedResult)
          .collect(Collectors.toList());

      // 6) Persist to target collection with metadata
      List<Document> toPersist = mapped.stream().map(this::toDocument).collect(Collectors.toList());
      if (!toPersist.isEmpty()) {
        mongoTemplate.insert(toPersist, targetCollection);
        log.info("Persisted {} aggregated docs to '{}'.", toPersist.size(), targetCollection);
      } else {
        log.warn("No documents to persist for '{}'.", targetCollection);
      }
      long tookMs = System.currentTimeMillis() - start;
      log.info("Aggregation completed in {} ms (source='{}', target='{}')", tookMs, sourceCollection, targetCollection);
      return mapped;

    } catch (IllegalArgumentException iae) {
      log.error("Invalid arguments for aggregation: {}", iae.getMessage(), iae);
      throw iae;
    } catch (DataAccessException dae) {
      log.error("Mongo access error during aggregation: {}", dae.getMessage(), dae);
      throw dae;
    } catch (Exception ex) {
      log.error("Unexpected error during aggregation", ex);
      throw new RuntimeException("Aggregation failed", ex);
    }
  }

  private Document buildProjectDocument(Map<String, String> projectionSpec) {
    Document proj = new Document();
    for (Map.Entry<String, String> e : projectionSpec.entrySet()) {
      String canonical = e.getKey();
      String src = e.getValue();
      if (isConstant(src)) {
        // support constants like "0" or "'literal'"
        Object val = parseConstant(src);
        proj.append(canonical, val);
      } else if ("eventTime".equals(canonical) && isEpochNumericField(src)) {
        // Convert epoch seconds to Date -> then to ISO instant if needed
        proj.append(canonical, "$" + src); // keep numeric; conversion optional downstream
      } else {
        proj.append(canonical, "$" + src);
      }
    }
    return new Document("$project", proj);
  }

  private boolean isConstant(String expr) {
    return expr != null && !expr.isBlank() && (expr.matches("^[-]?[0-9]+(\\.[0-9]+)?$") || (expr.startsWith("'") && expr.endsWith("'")));
  }

  private Object parseConstant(String expr) {
    if (expr.startsWith("'") && expr.endsWith("'")) return expr.substring(1, expr.length()-1);
    if (expr.contains(".")) return Double.parseDouble(expr);
    return Long.parseLong(expr);
  }

  private boolean isEpochNumericField(String src) {
    return src != null && !src.isBlank() && !src.contains(".") && !isConstant(src);
  }

  private GroupOperation groupBy(List<String> groupBy) {
    if (groupBy == null || groupBy.isEmpty()) return Aggregation.group();
    // _id: { f1: "$f1", f2: "$f2" }
    GroupOperation op = Aggregation.group(groupBy.stream().map(f -> Fields.field(f, "$" + f)).toArray(Fields.Field[0]));
    return op;
  }

  private AggregatedResult toAggregatedResult(Document d) {
    Document id = (Document) d.get("_id");
    String tenantId = id != null ? id.getString("tenantId") : null;
    String category = id != null ? id.getString("category") : null;
    Number count = (Number) d.getOrDefault("count", 0);
    Number sumAmount = (Number) d.getOrDefault("sum_amount", 0);
    return AggregatedResult.builder()
        .tenantId(tenantId)
        .category(category)
        .count(count.longValue())
        .totalAmount(sumAmount.doubleValue())
        .build();
  }

  private Document toDocument(AggregatedResult r) {
    Document doc = new Document();
    if (r.getTenantId() != null) doc.append("tenantId", r.getTenantId());
    if (r.getCategory() != null) doc.append("category", r.getCategory());
    doc.append("count", r.getCount());
    doc.append("totalAmount", r.getTotalAmount());
    doc.append("generatedAt", Date.from(Instant.now()));
    return doc;
  }
}

// ================================
// File: src/main/java/com/example/agg/web/AggregationController.java
// ================================
package com.example.agg.web;

import com.example.agg.domain.AggregatedResult;
import com.example.agg.domain.SourceType;
import com.example.agg.service.AggregationSpec;
import com.example.agg.service.MongoAggregationService;
import jakarta.validation.Valid;
import lombok.Data;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@Slf4j
@RestController
@RequestMapping("/api/aggregate")
@RequiredArgsConstructor
public class AggregationController {

  private final MongoAggregationService service;

  @PostMapping
  public ResponseEntity<List<AggregatedResult>> run(@Valid @RequestBody AggregationRequest req) {
    log.info("Incoming aggregation request: {}", req);
    Criteria match = null;
    if (req.getTenantId() != null) {
      match = Criteria.where("tenantId").is(req.getTenantId());
    }
    List<AggregatedResult> results = service.aggregateAndPersist(
        req.getSourceType(),
        req.getSourceCollection(),
        req.getTargetCollection(),
        req.getSpec(),
        match
    );
    return ResponseEntity.ok(results);
  }

  @Data
  public static class AggregationRequest {
    @Valid
    private SourceType sourceType;
    private String sourceCollection;
    private String targetCollection;
    private String tenantId; // optional filter example
    private AggregationSpec spec;
  }
}

// ================================
// File: src/test/java/com/example/agg/service/MongoAggregationServiceTest.java
// ================================
package com.example.agg.service;

import com.example.agg.domain.SourceType;
import com.example.agg.mapping.MapperRegistry;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.aggregation.AggregationResults;
import org.bson.Document;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;

public class MongoAggregationServiceTest {
  @Test
  void smoke() {
    MongoTemplate template = Mockito.mock(MongoTemplate.class);
    MapperRegistry registry = Mockito.mock(MapperRegistry.class);
    var svc = new MongoAggregationService(template, registry);

    // We only verify that exceptions are not thrown for empty results
    Mockito.when(template.aggregate(any(), anyString(), eq(Document.class)))
        .thenReturn(new AggregationResults<>(List.of(), new Document()));

    var spec = AggregationSpec.builder()
        .groupBy(java.util.List.of("tenantId", "category"))
        .sumFields(java.util.List.of("amount"))
        .includeCount(true)
        .build();

    // Expect IllegalArgumentException due to missing mapper
    assertThrows(IllegalArgumentException.class, () ->
        svc.aggregateAndPersist(SourceType.ORDERS, "orders", "orders_daily", spec, null));
  }
}

// ================================
// File: README.md
// ================================
/*
# Mongo Aggregation with Dynamic MapStruct Mappers

## How it works
- **DynamicMapper (Strategy)**: per source collection schema. Each implementation provides:
  - `projectionSpec()` for $project stage (source → canonical field names)
  - `toNormalized(Document)` for Java-side mapping when needed.
- **MapperRegistry** selects mapper by `SourceType` at runtime.
- **MongoAggregationService** builds pipeline dynamically:
  1. `$project` via mapper's projection
  2. optional `$match`
  3. `$group` by fields from `AggregationSpec` with `COUNT` and `SUM` support
  4. maps results to `AggregatedResult` and **persists** to `targetCollection`.

## Example Request
```bash
curl -X POST http://localhost:8080/api/aggregate \
  -H 'Content-Type: application/json' \
  -d '{
        "sourceType":"ORDERS",
        "sourceCollection":"orders",
        "targetCollection":"orders_daily_agg",
        "tenantId":"t1",
        "spec":{
          "groupBy":["tenantId","category"],
          "sumFields":["amount"],
          "includeCount":true
        }
      }'
```

## Notes
- Add additional mappers for more collections by implementing `DynamicMapper` with MapStruct.
- For high volume, consider using `$merge` stage instead of client-side insert.
- Validate `AggregationSpec` inputs and whitelist canonical fields.
- Add indexes on `tenantId`, `category`, and date buckets on the target collection as needed.
*/
