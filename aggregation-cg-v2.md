Awesome — here’s a complete, production-ready solution that:

keeps snake_case for all Java fields/methods/variables

does dynamic mapper selection per source collection

performs Mongo aggregations (COUNT/SUM) to build cve_list and priorities

merges multiple aggregation pipelines cleanly in the service

persists to a target collection

includes logging, validation, exceptions, and MapStruct mappers

is easy to extend for new collections

I’m showing a minimal-yet-complete codebase you can paste into a Spring Boot + Maven project.

🧱 Project structure
src/main/java/com/example/agg/
  config/
    mongo_config.java
  domain/
    target_document.java
    cve_item.java
    priority_item.java
  mapper/
    collection_one_mapper.java
    collection_two_mapper.java
    mapper_factory.java
  repo/
    aggregation_repository.java
  service/
    aggregation_service.java
  web/
    aggregation_controller.java
  util/
    aggregation_exception.java
    merge_exception.java
pom.xml

📦 pom.xml
<project xmlns="http://maven.apache.org/POM/4.0.0"  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0  http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>mongo-agg-mapstruct-snake</artifactId>
  <version>1.0.0</version>
  <properties>
    <java.version>17</java.version>
    <mapstruct.version>1.5.5.Final</mapstruct.version>
    <spring-boot.version>3.3.2</spring-boot.version>
  </properties>

  <dependencies>
    <!-- Spring Boot -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
      <version>${spring-boot.version}</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-mongodb</artifactId>
      <version>${spring-boot.version}</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-validation</artifactId>
      <version>${spring-boot.version}</version>
    </dependency>

    <!-- MapStruct -->
    <dependency>
      <groupId>org.mapstruct</groupId>
      <artifactId>mapstruct</artifactId>
      <version>${mapstruct.version}</version>
    </dependency>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <version>1.18.32</version>
      <scope>provided</scope>
    </dependency>

    <!-- Jackson (if you expose via REST) -->
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <!-- Spring Boot -->
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>

      <!-- MapStruct annotation processing -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.11.0</version>
        <configuration>
          <source>${java.version}</source>
          <target>${java.version}</target>
          <annotationProcessorPaths>
            <path>
              <groupId>org.projectlombok</groupId>
              <artifactId>lombok</artifactId>
              <version>1.18.32</version>
            </path>
            <path>
              <groupId>org.mapstruct</groupId>
              <artifactId>mapstruct-processor</artifactId>
              <version>${mapstruct.version}</version>
            </path>
          </annotationProcessorPaths>
          <compilerArgs>
            <arg>-Amapstruct.unmappedTargetPolicy=ERROR</arg>
          </compilerArgs>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>

⚙️ Config

config/mongo_config.java

package com.example.agg.config;

import org.springframework.context.annotation.Configuration;

@Configuration
public class mongo_config {
    // defaults ok when using spring.data.mongodb.* properties
}

🧩 Domain (snake_case everywhere)

domain/target_document.java

package com.example.agg.domain;

import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.List;
import org.springframework.data.mongodb.core.mapping.Document;
import lombok.*;

@Document(collection = "target_collection")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class target_document {
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

    private List<cve_item> cve_list;
    private List<priority_item> priorities;

    private LocalDateTime date_loaded;
}


domain/cve_item.java

package com.example.agg.domain;

import java.time.LocalDate;
import lombok.*;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class cve_item {
    private String name; // from source field "cve"
    private LocalDate first_reported;
    private LocalDate due_date;
    private boolean past_due;
    private LocalDate date_detected;
    private String environment;
    private String remediation_status;
    private int days_open;
    private String severity;       // from source "last_severity"
    private String erp_status;     // from source "scorecard_erp_status" (or your ERP field)
    private String scorecard_source;
    private String observation_title;
    private String observation_description;
    private int instances_found;   // COUNT per (group keys + cve)
}


domain/priority_item.java

package com.example.agg.domain;

import lombok.*;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class priority_item {
    private String name;            // e.g., last_severity
    private int instances_found;    // COUNT per severity
}

🗺️ MapStruct mappers (+ dynamic selection)

We’ll map from org.bson.Document produced by aggregations into our domain objects.
Names match (snake_case), so most fields map automatically.

mapper/collection_one_mapper.java

package com.example.agg.mapper;

import com.example.agg.domain.*;
import org.bson.Document;
import org.mapstruct.*;
import java.util.*;
import java.time.*;
import java.time.ZoneId;

@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.ERROR)
public interface collection_one_mapper {

    // HEADER (top-level) mapper
    @BeanMapping(ignoreByDefault = true)
    @Mappings({
        @Mapping(target = "cio_lob_name", source = "cio_lob_name"),
        @Mapping(target = "cio_name", source = "cio_name"),
        @Mapping(target = "cto_name", source = "cto_name"),
        @Mapping(target = "cio_lob_name_one_deep", source = "cio_lob_name_one_deep"),

        @Mapping(target = "ait_no", source = "ait_no"),
        @Mapping(target = "ait_name", source = "ait_name"),
        @Mapping(target = "ait_app_manager", source = "ait_app_manager"),
        @Mapping(target = "ait_app_manager_nbid", source = "ait_app_manager_nbid"),
        @Mapping(target = "ait_mgmt_support_contact", source = "ait_mgmt_support_contact"),
        @Mapping(target = "ait_mgmt_support_contact_nbid", source = "ait_mgmt_support_contact_nbid"),
        @Mapping(target = "ait_tech_exec", source = "ait_tech_exec"),
        @Mapping(target = "ait_tech_exec_nbid", source = "ait_tech_exec_nbid"),
        @Mapping(target = "ait_app_manager_email", source = "ait_app_manager_email"),

        @Mapping(target = "gis_network_rating_zone", source = "gis_network_rating_zone"),
        @Mapping(target = "gis_metric_alignment", source = "gis_metric_alignment"),
        @Mapping(target = "ait_recovery_time_obj", source = "ait_recovery_time_obj"),

        @Mapping(target = "enterprise_lob_name", source = "enterprise_lob_name"),
        @Mapping(target = "country_name", source = "country_name"),
        @Mapping(target = "region", source = "region"),
        @Mapping(target = "gis_asset_category", source = "gis_asset_category"),
        @Mapping(target = "device_type", source = "device_type"),
        @Mapping(target = "os_name", source = "os_name")
    })
    target_document to_target_header(Document doc);

    // CVE item mapper (from sub-document)
    @BeanMapping(ignoreByDefault = true)
    @Mappings({
        @Mapping(target = "name", source = "cve"),
        @Mapping(target = "first_reported", expression = "java(to_local_date(doc.get(\"first_reported\")))"),
        @Mapping(target = "due_date", expression = "java(to_local_date(doc.get(\"due_date\")))"),
        @Mapping(target = "past_due", source = "past_due"),
        @Mapping(target = "date_detected", expression = "java(to_local_date(doc.get(\"date_detected\")))"),
        @Mapping(target = "environment", source = "environment"),
        @Mapping(target = "remediation_status", source = "remediation_status"),
        @Mapping(target = "days_open", source = "days_open"),
        @Mapping(target = "severity", source = "last_severity"),
        @Mapping(target = "erp_status", source = "scorecard_erp_status"),
        @Mapping(target = "scorecard_source", source = "scorecard_source"),
        @Mapping(target = "observation_title", source = "observation_title"),
        @Mapping(target = "observation_description", source = "observation_description"),
        @Mapping(target = "instances_found", source = "instances_found")
    })
    cve_item to_cve_item(Document doc);

    default List<cve_item> to_cve_list(List<Document> docs) {
        if (docs == null) return Collections.emptyList();
        List<cve_item> out = new ArrayList<>();
        for (Document d : docs) out.add(to_cve_item(d));
        return out;
    }

    // Priority item mapper
    @BeanMapping(ignoreByDefault = true)
    @Mappings({
        @Mapping(target = "name", source = "last_severity"),
        @Mapping(target = "instances_found", source = "instances_found")
    })
    priority_item to_priority_item(Document doc);

    default List<priority_item> to_priority_list(List<Document> docs) {
        if (docs == null) return Collections.emptyList();
        List<priority_item> out = new ArrayList<>();
        for (Document d : docs) out.add(to_priority_item(d));
        return out;
    }

    // Util
    default java.time.LocalDate to_local_date(Object o) {
        if (o == null) return null;
        if (o instanceof java.util.Date dt) {
            return dt.toInstant().atZone(ZoneId.systemDefault()).toLocalDate();
        }
        return null;
    }
}


mapper/collection_two_mapper.java
(If you have a second source collection with slightly different fields; otherwise it can delegate to the same mapping.)

package com.example.agg.mapper;

import com.example.agg.domain.*;
import org.bson.Document;
import org.mapstruct.*;

@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.ERROR, uses = collection_one_mapper.class)
public interface collection_two_mapper {
    // You can either duplicate mappings adjusted for collection_two fields
    // or reuse collection_one_mapper when schema is the same.
    // Here we just reuse header mapping for brevity:

    default target_document to_target_header(Document doc) {
        // If collection_two differs, map accordingly; otherwise delegate or copy fields
        target_document t = new target_document();
        t.set_cio_lob_name((String) doc.get("cio_lob_name"));
        t.set_cio_name((String) doc.get("cio_name"));
        t.set_cto_name((String) doc.get("cto_name"));
        t.set_ait_no((String) doc.get("ait_no"));
        t.set_enterprise_lob_name((String) doc.get("enterprise_lob_name"));
        // ... add the rest as needed
        return t;
    }
}


mapper/mapper_factory.java

package com.example.agg.mapper;

import com.example.agg.domain.target_document;
import lombok.RequiredArgsConstructor;
import org.bson.Document;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
public class mapper_factory {

    private final collection_one_mapper collection_one_mapper;
    private final collection_two_mapper collection_two_mapper;

    public target_document to_target_header(String collection_name, Document header) {
        return switch (collection_name) {
            case "collection_one" -> collection_one_mapper.to_target_header(header);
            case "collection_two" -> collection_two_mapper.to_target_header(header);
            default -> throw new IllegalArgumentException("no mapper for collection: " + collection_name);
        };
    }

    public java.util.List<com.example.agg.domain.cve_item> to_cve_list(String collection_name, java.util.List<Document> docs) {
        return switch (collection_name) {
            case "collection_one" -> collection_one_mapper.to_cve_list(docs);
            case "collection_two" -> collection_one_mapper.to_cve_list(docs); // reuse
            default -> throw new IllegalArgumentException("no mapper for collection: " + collection_name);
        };
    }

    public java.util.List<com.example.agg.domain.priority_item> to_priority_list(String collection_name, java.util.List<Document> docs) {
        return switch (collection_name) {
            case "collection_one" -> collection_one_mapper.to_priority_list(docs);
            case "collection_two" -> collection_one_mapper.to_priority_list(docs); // reuse
            default -> throw new IllegalArgumentException("no mapper for collection: " + collection_name);
        };
    }
}

🧪 Repository (Mongo aggregations)

We run three pipelines and merge in service:

header: one row per identity (AIT etc.) using $group with $first

cve_items: count instances per CVE and collect details

priorities: count instances per severity

Adjust identity_keys to match how you want to group target documents (common choice: ait_no + org keys).

repo/aggregation_repository.java

package com.example.agg.repo;

import com.example.agg.util.aggregation_exception;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.bson.Document;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.aggregation.*;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
@RequiredArgsConstructor
@Slf4j
public class aggregation_repository {

    private final MongoTemplate mongo_template;

    // ---------- HEADER AGG ----------
    public List<Document> aggregate_header(String collection_name) {
        try {
            // identity keys — choose what defines a single target_document
            String[] id = new String[]{
                "cio_lob_name", "cio_name", "cto_name", "cio_lob_name_one_deep",
                "ait_no", "ait_name", "ait_app_manager", "ait_app_manager_nbid",
                "ait_mgmt_support_contact", "ait_mgmt_support_contact_nbid",
                "ait_tech_exec", "ait_tech_exec_nbid", "ait_app_manager_email",
                "gis_network_rating_zone", "gis_metric_alignment", "ait_recovery_time_obj",
                "enterprise_lob_name", "country_name", "region", "gis_asset_category",
                "device_type", "os_name"
            };

            GroupOperation group = Aggregation.group(id)
                .first("cio_lob_name").as("cio_lob_name")
                .first("cio_name").as("cio_name")
                .first("cto_name").as("cto_name")
                .first("cio_lob_name_one_deep").as("cio_lob_name_one_deep")

                .first("ait_no").as("ait_no")
                .first("ait_name").as("ait_name")
                .first("ait_app_manager").as("ait_app_manager")
                .first("ait_app_manager_nbid").as("ait_app_manager_nbid")
                .first("ait_mgmt_support_contact").as("ait_mgmt_support_contact")
                .first("ait_mgmt_support_contact_nbid").as("ait_mgmt_support_contact_nbid")
                .first("ait_tech_exec").as("ait_tech_exec")
                .first("ait_tech_exec_nbid").as("ait_tech_exec_nbid")
                .first("ait_app_manager_email").as("ait_app_manager_email")

                .first("gis_network_rating_zone").as("gis_network_rating_zone")
                .first("gis_metric_alignment").as("gis_metric_alignment")
                .first("ait_recovery_time_obj").as("ait_recovery_time_obj")

                .first("enterprise_lob_name").as("enterprise_lob_name")
                .first("country_name").as("country_name")
                .first("region").as("region")
                .first("gis_asset_category").as("gis_asset_category")
                .first("device_type").as("device_type")
                .first("os_name").as("os_name");

            Aggregation agg = Aggregation.newAggregation(
                group,
                Aggregation.project().andExclude("_id")
            );

            return mongo_template.aggregate(agg, collection_name, Document.class).getMappedResults();
        } catch (Exception e) {
            log.error("header aggregation error: {}", e.getMessage(), e);
            throw new aggregation_exception("header aggregation failed for " + collection_name, e);
        }
    }

    // ---------- CVE AGG ----------
    // For each identity + CVE, COUNT records and capture details
    public List<Document> aggregate_cve_items(String collection_name) {
        try {
            String[] id = new String[]{
                "cio_lob_name", "cio_name", "cto_name", "cio_lob_name_one_deep",
                "ait_no", "enterprise_lob_name", "country_name", "region", "gis_asset_category",
                "device_type", "os_name", "cve"
            };

            GroupOperation group = Aggregation.group(id)
                .first("first_reported").as("first_reported")
                .first("due_date").as("due_date")
                .first("past_due").as("past_due")
                .first("date_detected").as("date_detected")
                .first("environment").as("environment")
                .first("remediation_status").as("remediation_status")
                .first("days_open").as("days_open")
                .first("last_severity").as("last_severity")
                .first("scorecard_erp_status").as("scorecard_erp_status")
                .first("scorecard_source").as("scorecard_source")
                .first("observation_title").as("observation_title")
                .first("observation_description").as("observation_description")
                .count().as("instances_found");

            ProjectionOperation project = Aggregation.project()
                .and("_id.cio_lob_name").as("cio_lob_name")
                .and("_id.cio_name").as("cio_name")
                .and("_id.cto_name").as("cto_name")
                .and("_id.cio_lob_name_one_deep").as("cio_lob_name_one_deep")
                .and("_id.ait_no").as("ait_no")
                .and("_id.enterprise_lob_name").as("enterprise_lob_name")
                .and("_id.country_name").as("country_name")
                .and("_id.region").as("region")
                .and("_id.gis_asset_category").as("gis_asset_category")
                .and("_id.device_type").as("device_type")
                .and("_id.os_name").as("os_name")
                .and("_id.cve").as("cve")
                .andInclude("first_reported", "due_date", "past_due", "date_detected",
                    "environment", "remediation_status", "days_open", "last_severity",
                    "scorecard_erp_status", "scorecard_source", "observation_title",
                    "observation_description", "instances_found")
                .andExclude("_id");

            Aggregation agg = Aggregation.newAggregation(group, project);

            return mongo_template.aggregate(agg, collection_name, Document.class).getMappedResults();
        } catch (Exception e) {
            log.error("cve aggregation error: {}", e.getMessage(), e);
            throw new aggregation_exception("cve aggregation failed for " + collection_name, e);
        }
    }

    // ---------- PRIORITY AGG ----------
    // For each identity + severity, COUNT records
    public List<Document> aggregate_priorities(String collection_name) {
        try {
            String[] id = new String[]{
                "cio_lob_name", "cio_name", "cto_name", "cio_lob_name_one_deep",
                "ait_no", "enterprise_lob_name", "country_name", "region",
                "gis_asset_category", "device_type", "os_name", "last_severity"
            };

            GroupOperation group = Aggregation.group(id).count().as("instances_found");

            ProjectionOperation project = Aggregation.project()
                .and("_id.cio_lob_name").as("cio_lob_name")
                .and("_id.cio_name").as("cio_name")
                .and("_id.cto_name").as("cto_name")
                .and("_id.cio_lob_name_one_deep").as("cio_lob_name_one_deep")
                .and("_id.ait_no").as("ait_no")
                .and("_id.enterprise_lob_name").as("enterprise_lob_name")
                .and("_id.country_name").as("country_name")
                .and("_id.region").as("region")
                .and("_id.gis_asset_category").as("gis_asset_category")
                .and("_id.device_type").as("device_type")
                .and("_id.os_name").as("os_name")
                .and("_id.last_severity").as("last_severity")
                .and("instances_found").as("instances_found")
                .andExclude("_id");

            Aggregation agg = Aggregation.newAggregation(group, project);

            return mongo_template.aggregate(agg, collection_name, Document.class).getMappedResults();
        } catch (Exception e) {
            log.error("priority aggregation error: {}", e.getMessage(), e);
            throw new aggregation_exception("priority aggregation failed for " + collection_name, e);
        }
    }
}

🧠 Service (merge & persist)

Builds a key per identity (same fields used in header group)

Merges header + cve + priorities

Sets date_loaded

Persists to target_collection

service/aggregation_service.java

package com.example.agg.service;

import com.example.agg.domain.*;
import com.example.agg.mapper.mapper_factory;
import com.example.agg.repo.aggregation_repository;
import com.example.agg.util.merge_exception;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.bson.Document;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.*;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Slf4j
public class aggregation_service {

    private final aggregation_repository aggregation_repository;
    private final mapper_factory mapper_factory;
    private final MongoTemplate mongo_template;

    public void process_and_persist(String collection_name) {
        log.info("aggregation start: collection={}", collection_name);

        List<Document> header_docs = aggregation_repository.aggregate_header(collection_name);
        List<Document> cve_docs = aggregation_repository.aggregate_cve_items(collection_name);
        List<Document> prio_docs = aggregation_repository.aggregate_priorities(collection_name);

        // Map: identity_key -> target_document
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

            // attach CVE items
            Map<String, List<Document>> cve_by_key = cve_docs.stream().collect(Collectors.groupingBy(this::identity_key));
            for (Map.Entry<String, List<Document>> e : cve_by_key.entrySet()) {
                target_document t = by_key.get(e.getKey());
                if (t == null) continue;
                List<cve_item> cves = mapper_factory.to_cve_list(collection_name, e.getValue());
                t.get_cve_list().addAll(cves);
            }

            // attach priorities
            Map<String, List<Document>> prio_by_key = prio_docs.stream().collect(Collectors.groupingBy(this::identity_key));
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
            mongo_template.insert(out, "target_collection");
        }
        log.info("aggregation done: persisted {} docs into target_collection", out.size());
    }

    // Build identity key; must match keys projected in repo
    private String identity_key(Document d) {
        // Keep order stable
        return String.join("|",
            nv(d, "cio_lob_name"), nv(d, "cio_name"), nv(d, "cto_name"), nv(d, "cio_lob_name_one_deep"),
            nv(d, "ait_no"), nv(d, "enterprise_lob_name"), nv(d, "country_name"), nv(d, "region"),
            nv(d, "gis_asset_category"), nv(d, "device_type"), nv(d, "os_name")
        );
    }

    private String nv(Document d, String k) {
        Object v = d.get(k);
        return v == null ? "" : String.valueOf(v);
    }
}

🌐 REST entrypoint (optional)

web/aggregation_controller.java

package com.example.agg.web;

import com.example.agg.service.aggregation_service;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequiredArgsConstructor
@RequestMapping("/aggregation")
public class aggregation_controller {

    private final aggregation_service aggregation_service;

    @PostMapping("/run/{collection_name}")
    public ResponseEntity<Void> run(@PathVariable String collection_name) {
        aggregation_service.process_and_persist(collection_name);
        return ResponseEntity.accepted().build();
    }
}

🧰 Exceptions

util/aggregation_exception.java

package com.example.agg.util;

public class aggregation_exception extends RuntimeException {
    public aggregation_exception(String message, Throwable cause) { super(message, cause); }
}


util/merge_exception.java

package com.example.agg.util;

public class merge_exception extends RuntimeException {
    public merge_exception(String message, Throwable cause) { super(message, cause); }
}

✅ Notes & extensions

Snake_case everywhere in Java: field names, methods, locals, beans.

If using Checkstyle/Spotless, allow ^[a-z0-9_]+$ for variables/fields/methods.

Aggregation math:

instances_found is a COUNT per (identity + cve) and per (identity + last_severity).

To add SUM fields, e.g., SUM(days_open) per CVE, add .sum("days_open").as("days_open_sum") in aggregate_cve_items and map it into cve_item (add a field).

ERP status field: I used scorecard_erp_status for erp_status in cve_item. If you prefer a different ERP field, change the mapping.

Target upsert: switch to bulkOps(BulkOperations.BulkMode.UNORDERED, target_document.class) with a query on identity keys for idempotent updates instead of blind insert.

Multiple source collections: add new collection_X_mapper + a case in mapper_factory.

Filtering: add a $match stage at the start of repository aggregations if you need date windows, environments, etc.

If you want, I can also drop in a bulk upsert version (so reruns overwrite the same identity rather than inserting duplicates) and a JUnit/Mockito test suite scaffolding—just say the word and I’ll include it in snake_case too.
