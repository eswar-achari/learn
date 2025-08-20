# Unit Tests for MongoDB Aggregation Implementation

## Test Strategy

We'll create comprehensive unit tests that:
1. Test individual components in isolation using mocks
2. Test the full integration with an embedded MongoDB
3. Validate aggregation results against expected outputs

## Dependencies for Testing

Add these to your `pom.xml`:

```xml
<dependencies>
    <!-- Test Dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <version>${spring-boot.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>de.flapdoodle.embed</groupId>
        <artifactId>de.flapdoodle.embed.mongo</artifactId>
        <version>4.11.0</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <version>1.19.3</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>mongodb</artifactId>
        <version>1.19.3</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## Test Data Setup

Create a test data utility class:

```java
// test/java/com/example/agg/util/test_data_util.java
package com.example.agg.util;

import org.bson.Document;
import java.time.LocalDate;
import java.util.Arrays;
import java.util.List;

public class test_data_util {

    public static List<Document> create_test_documents() {
        return Arrays.asList(
            new Document()
                .append("cio_lob_name", "LOB1")
                .append("cio_name", "CIO1")
                .append("cto_name", "CTO1")
                .append("cio_lob_name_one_deep", "LOB1_DEEP")
                .append("ait_no", "AIT001")
                .append("ait_name", "Application1")
                .append("enterprise_lob_name", "Enterprise1")
                .append("country_name", "USA")
                .append("region", "North America")
                .append("gis_asset_category", "Server")
                .append("device_type", "Physical")
                .append("os_name", "Linux")
                .append("cve", "CVE-2023-1234")
                .append("first_reported", java.util.Date.from(LocalDate.of(2023, 1, 15).atStartOfDay(java.time.ZoneId.systemDefault()).toInstant()))
                .append("last_severity", "High")
                .append("scorecard_erp_status", "Open"),
                
            new Document()
                .append("cio_lob_name", "LOB1")
                .append("cio_name", "CIO1")
                .append("cto_name", "CTO1")
                .append("cio_lob_name_one_deep", "LOB1_DEEP")
                .append("ait_no", "AIT001")
                .append("ait_name", "Application1")
                .append("enterprise_lob_name", "Enterprise1")
                .append("country_name", "USA")
                .append("region", "North America")
                .append("gis_asset_category", "Server")
                .append("device_type", "Physical")
                .append("os_name", "Linux")
                .append("cve", "CVE-2023-1234")
                .append("first_reported", java.util.Date.from(LocalDate.of(2023, 1, 20).atStartOfDay(java.time.ZoneId.systemDefault()).toInstant()))
                .append("last_severity", "High")
                .append("scorecard_erp_status", "Open"),
                
            new Document()
                .append("cio_lob_name", "LOB1")
                .append("cio_name", "CIO1")
                .append("cto_name", "CTO1")
                .append("cio_lob_name_one_deep", "LOB1_DEEP")
                .append("ait_no", "AIT001")
                .append("ait_name", "Application1")
                .append("enterprise_lob_name", "Enterprise1")
                .append("country_name", "USA")
                .append("region", "North America")
                .append("gis_asset_category", "Server")
                .append("device_type", "Physical")
                .append("os_name", "Linux")
                .append("cve", "CVE-2023-5678")
                .append("first_reported", java.util.Date.from(LocalDate.of(2023, 2, 10).atStartOfDay(java.time.ZoneId.systemDefault()).toInstant()))
                .append("last_severity", "Medium")
                .append("scorecard_erp_status", "In Progress")
        );
    }
    
    public static Document create_expected_header() {
        return new Document()
            .append("cio_lob_name", "LOB1")
            .append("cio_name", "CIO1")
            .append("cto_name", "CTO1")
            .append("cio_lob_name_one_deep", "LOB1_DEEP")
            .append("ait_no", "AIT001")
            .append("ait_name", "Application1")
            .append("enterprise_lob_name", "Enterprise1")
            .append("country_name", "USA")
            .append("region", "North America")
            .append("gis_asset_category", "Server")
            .append("device_type", "Physical")
            .append("os_name", "Linux");
    }
}
```

## Unit Tests for Mappers

```java
// test/java/com/example/agg/mapper/collection_one_mapper_test.java
package com.example.agg.mapper;

import com.example.agg.domain.cve_item;
import com.example.agg.domain.priority_item;
import com.example.agg.domain.target_document;
import org.bson.Document;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.time.LocalDate;
import java.util.Collections;
import java.util.Date;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
public class collection_one_mapper_test {

    @Autowired
    private collection_one_mapper mapper;

    @Test
    public void test_to_target_header() {
        Document source = new Document()
            .append("cio_lob_name", "TestLOB")
            .append("cio_name", "TestCIO")
            .append("ait_no", "TEST001")
            .append("enterprise_lob_name", "TestEnterprise");
        
        target_document result = mapper.to_target_header(source);
        
        assertNotNull(result);
        assertEquals("TestLOB", result.get_cio_lob_name());
        assertEquals("TestCIO", result.get_cio_name());
        assertEquals("TEST001", result.get_ait_no());
        assertEquals("TestEnterprise", result.get_enterprise_lob_name());
    }

    @Test
    public void test_to_cve_item() {
        Date testDate = new Date();
        Document source = new Document()
            .append("cve", "CVE-2023-0001")
            .append("first_reported", testDate)
            .append("last_severity", "High")
            .append("instances_found", 5);
        
        cve_item result = mapper.to_cve_item(source);
        
        assertNotNull(result);
        assertEquals("CVE-2023-0001", result.get_name());
        assertEquals("High", result.get_severity());
        assertEquals(5, result.get_instances_found());
    }

    @Test
    public void test_to_cve_list() {
        Document doc1 = new Document().append("cve", "CVE-2023-0001");
        Document doc2 = new Document().append("cve", "CVE-2023-0002");
        
        List<cve_item> result = mapper.to_cve_list(List.of(doc1, doc2));
        
        assertEquals(2, result.size());
        assertEquals("CVE-2023-0001", result.get(0).get_name());
        assertEquals("CVE-2023-0002", result.get(1).get_name());
    }

    @Test
    public void test_to_priority_item() {
        Document source = new Document()
            .append("last_severity", "Critical")
            .append("instances_found", 10);
        
        priority_item result = mapper.to_priority_item(source);
        
        assertNotNull(result);
        assertEquals("Critical", result.get_name());
        assertEquals(10, result.get_instances_found());
    }

    @Test
    public void test_to_priority_list() {
        Document doc1 = new Document().append("last_severity", "High");
        Document doc2 = new Document().append("last_severity", "Medium");
        
        List<priority_item> result = mapper.to_priority_list(List.of(doc1, doc2));
        
        assertEquals(2, result.size());
        assertEquals("High", result.get(0).get_name());
        assertEquals("Medium", result.get(1).get_name());
    }

    @Test
    public void test_to_local_date_conversion() {
        Date testDate = Date.from(LocalDate.of(2023, 5, 15).atStartOfDay(java.time.ZoneId.systemDefault()).toInstant());
        LocalDate result = mapper.to_local_date(testDate);
        
        assertEquals(LocalDate.of(2023, 5, 15), result);
    }

    @Test
    public void test_to_local_date_null_input() {
        assertNull(mapper.to_local_date(null));
    }
}
```

## Unit Tests for Repository Layer

```java
// test/java/com/example/agg/repo/aggregation_repository_test.java
package com.example.agg.repo;

import com.example.agg.util.test_data_util;
import com.mongodb.client.MongoClient;
import org.bson.Document;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
@Testcontainers
public class aggregation_repository_test {

    @Container
    static MongoDBContainer mongoDBContainer = new MongoDBContainer("mongo:6.0");

    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.mongodb.uri", mongoDBContainer::getReplicaSetUrl);
    }

    @Autowired
    private aggregation_repository repository;

    @Autowired
    private MongoTemplate mongo_template;

    @BeforeEach
    public void setup() {
        // Clean up and insert test data
        mongo_template.dropCollection("test_collection");
        List<Document> testData = test_data_util.create_test_documents();
        mongo_template.insert(testData, "test_collection");
    }

    @Test
    public void test_aggregate_header() {
        List<Document> result = repository.aggregate_header("test_collection");
        
        assertNotNull(result);
        assertEquals(1, result.size());
        
        Document header = result.get(0);
        assertEquals("LOB1", header.getString("cio_lob_name"));
        assertEquals("AIT001", header.getString("ait_no"));
        assertEquals("Enterprise1", header.getString("enterprise_lob_name"));
    }

    @Test
    public void test_aggregate_cve_items() {
        List<Document> result = repository.aggregate_cve_items("test_collection");
        
        assertNotNull(result);
        assertEquals(2, result.size()); // Should have 2 distinct CVEs
        
        // Find CVE-2023-1234 which should have count of 2
        Document cve1234 = result.stream()
            .filter(doc -> "CVE-2023-1234".equals(doc.getString("cve")))
            .findFirst()
            .orElse(null);
            
        assertNotNull(cve1234);
        assertEquals(2, cve1234.getInteger("instances_found"));
        assertEquals("High", cve1234.getString("last_severity"));
        
        // Find CVE-2023-5678 which should have count of 1
        Document cve5678 = result.stream()
            .filter(doc -> "CVE-2023-5678".equals(doc.getString("cve")))
            .findFirst()
            .orElse(null);
            
        assertNotNull(cve5678);
        assertEquals(1, cve5678.getInteger("instances_found"));
        assertEquals("Medium", cve5678.getString("last_severity"));
    }

    @Test
    public void test_aggregate_priorities() {
        List<Document> result = repository.aggregate_priorities("test_collection");
        
        assertNotNull(result);
        assertEquals(2, result.size()); // Should have 2 distinct priorities
        
        // Find High priority which should have count of 2
        Document highPriority = result.stream()
            .filter(doc -> "High".equals(doc.getString("last_severity")))
            .findFirst()
            .orElse(null);
            
        assertNotNull(highPriority);
        assertEquals(2, highPriority.getInteger("instances_found"));
        
        // Find Medium priority which should have count of 1
        Document mediumPriority = result.stream()
            .filter(doc -> "Medium".equals(doc.getString("last_severity")))
            .findFirst()
            .orElse(null);
            
        assertNotNull(mediumPriority);
        assertEquals(1, mediumPriority.getInteger("instances_found"));
    }

    @Test
    public void test_aggregate_header_empty_collection() {
        mongo_template.dropCollection("empty_collection");
        
        List<Document> result = repository.aggregate_header("empty_collection");
        
        assertNotNull(result);
        assertTrue(result.isEmpty());
    }
}
```

## Unit Tests for Service Layer

```java
// test/java/com/example/agg/service/aggregation_service_test.java
package com.example.agg.service;

import com.example.agg.domain.target_document;
import com.example.agg.mapper.mapper_factory;
import com.example.agg.repo.aggregation_repository;
import com.example.agg.util.test_data_util;
import org.bson.Document;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
@Testcontainers
public class aggregation_service_test {

    @Container
    static MongoDBContainer mongoDBContainer = new MongoDBContainer("mongo:6.0");

    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.mongodb.uri", mongoDBContainer::getReplicaSetUrl);
    }

    @Autowired
    private aggregation_service service;

    @Autowired
    private aggregation_repository repository;

    @Autowired
    private MongoTemplate mongo_template;

    @Autowired
    private mapper_factory mapper_factory;

    @BeforeEach
    public void setup() {
        // Clean up and insert test data
        mongo_template.dropCollection("test_collection");
        mongo_template.dropCollection("target_collection");
        
        List<Document> testData = test_data_util.create_test_documents();
        mongo_template.insert(testData, "test_collection");
    }

    @Test
    public void test_process_and_persist() {
        service.process_and_persist("test_collection");
        
        // Verify results were persisted to target collection
        List<target_document> results = mongo_template.findAll(target_document.class, "target_collection");
        
        assertNotNull(results);
        assertEquals(1, results.size()); // Should have one aggregated document
        
        target_document result = results.get(0);
        
        // Verify header fields
        assertEquals("LOB1", result.get_cio_lob_name());
        assertEquals("AIT001", result.get_ait_no());
        assertEquals("Enterprise1", result.get_enterprise_lob_name());
        
        // Verify CVE list
        assertNotNull(result.get_cve_list());
        assertEquals(2, result.get_cve_list().size());
        
        // Verify CVE-2023-1234 has count of 2
        var cve1234 = result.get_cve_list().stream()
            .filter(cve -> "CVE-2023-1234".equals(cve.get_name()))
            .findFirst()
            .orElse(null);
        assertNotNull(cve1234);
        assertEquals(2, cve1234.get_instances_found());
        assertEquals("High", cve1234.get_severity());
        
        // Verify priorities
        assertNotNull(result.get_priorities());
        assertEquals(2, result.get_priorities().size());
        
        // Verify High priority has count of 2
        var highPriority = result.get_priorities().stream()
            .filter(p -> "High".equals(p.get_name()))
            .findFirst()
            .orElse(null);
        assertNotNull(highPriority);
        assertEquals(2, highPriority.get_instances_found());
        
        // Verify date_loaded was set
        assertNotNull(result.get_date_loaded());
    }

    @Test
    public void test_process_and_persist_empty_collection() {
        mongo_template.dropCollection("empty_collection");
        
        service.process_and_persist("empty_collection");
        
        // Verify no documents were persisted to target collection
        List<target_document> results = mongo_template.findAll(target_document.class, "target_collection");
        assertTrue(results.isEmpty());
    }

    @Test
    public void test_identity_key_generation() {
        Document doc = test_data_util.create_expected_header();
        
        // Use reflection to test private method
        java.lang.reflect.Method method = null;
        try {
            method = aggregation_service.class.getDeclaredMethod("identity_key", Document.class);
            method.setAccessible(true);
            
            String result = (String) method.invoke(service, doc);
            
            assertNotNull(result);
            assertTrue(result.contains("LOB1"));
            assertTrue(result.contains("AIT001"));
            assertTrue(result.contains("Enterprise1"));
        } catch (Exception e) {
            fail("Failed to test identity_key method: " + e.getMessage());
        }
    }
}
```

## Integration Test with Comparison

```java
// test/java/com/example/agg/integration/aggregation_integration_test.java
package com.example.agg.integration;

import com.example.agg.domain.target_document;
import com.example.agg.service.aggregation_service;
import com.example.agg.util.test_data_util;
import org.bson.Document;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
@Testcontainers
public class aggregation_integration_test {

    @Container
    static MongoDBContainer mongoDBContainer = new MongoDBContainer("mongo:6.0");

    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.mongodb.uri", mongoDBContainer::getReplicaSetUrl);
    }

    @Autowired
    private aggregation_service service;

    @Autowired
    private MongoTemplate mongo_template;

    @BeforeEach
    public void setup() {
        mongo_template.dropCollection("source_collection");
        mongo_template.dropCollection("target_collection");
        
        List<Document> testData = test_data_util.create_test_documents();
        mongo_template.insert(testData, "source_collection");
    }

    @Test
    public void test_full_aggregation_pipeline() {
        // Execute the full aggregation pipeline
        service.process_and_persist("source_collection");
        
        // Verify the results
        List<target_document> results = mongo_template.findAll(target_document.class, "target_collection");
        
        assertEquals(1, results.size());
        target_document aggregated = results.get(0);
        
        // Verify header information
        assertEquals("LOB1", aggregated.get_cio_lob_name());
        assertEquals("AIT001", aggregated.get_ait_no());
        
        // Verify CVE aggregation
        assertEquals(2, aggregated.get_cve_list().size());
        
        // CVE-2023-1234 should have 2 instances
        var cve1234 = aggregated.get_cve_list().stream()
            .filter(cve -> "CVE-2023-1234".equals(cve.get_name()))
            .findFirst()
            .orElse(null);
        assertNotNull(cve1234);
        assertEquals(2, cve1234.get_instances_found());
        
        // CVE-2023-5678 should have 1 instance
        var cve5678 = aggregated.get_cve_list().stream()
            .filter(cve -> "CVE-2023-5678".equals(cve.get_name()))
            .findFirst()
            .orElse(null);
        assertNotNull(cve5678);
        assertEquals(1, cve5678.get_instances_found());
        
        // Verify priority aggregation
        assertEquals(2, aggregated.get_priorities().size());
        
        // High priority should have 2 instances
        var highPriority = aggregated.get_priorities().stream()
            .filter(p -> "High".equals(p.get_name()))
            .findFirst()
            .orElse(null);
        assertNotNull(highPriority);
        assertEquals(2, highPriority.get_instances_found());
        
        // Medium priority should have 1 instance
        var mediumPriority = aggregated.get_priorities().stream()
            .filter(p -> "Medium".equals(p.get_name()))
            .findFirst()
            .orElse(null);
        assertNotNull(mediumPriority);
        assertEquals(1, mediumPriority.get_instances_found());
        
        // Compare with expected counts from source data
        long sourceCount = mongo_template.getCollection("source_collection").countDocuments();
        assertEquals(3, sourceCount);
        
        // Verify that the sum of all CVE instances equals the source count
        int totalCveInstances = aggregated.get_cve_list().stream()
            .mapToInt(cve -> cve.get_instances_found())
            .sum();
        assertEquals(sourceCount, totalCveInstances);
        
        // Verify that the sum of all priority instances equals the source count
        int totalPriorityInstances = aggregated.get_priorities().stream()
            .mapToInt(priority -> priority.get_instances_found())
            .sum();
        assertEquals(sourceCount, totalPriorityInstances);
    }
}
```

## Test Configuration

```java
// test/java/com/example/agg/test_config.java
package com.example.agg;

import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.data.mongodb.core.MongoTemplate;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;

@TestConfiguration
public class test_config {

    @Bean
    public MongoTemplate mongoTemplate() {
        // Use embedded MongoDB for testing
        MongoClient mongoClient = MongoClients.create("mongodb://localhost:27017/test");
        return new MongoTemplate(mongoClient, "test");
    }
}
```

## Running the Tests

To run these tests, use the following Maven command:

```bash
mvn test
```

Or run specific test classes:

```bash
mvn test -Dtest=aggregation_integration_test
```

## Expected Test Results

The tests should validate that:

1. **Mapper Tests**: All field mappings work correctly, including date conversions
2. **Repository Tests**: Aggregation pipelines return the expected counts and groupings
3. **Service Tests**: The merging logic correctly combines header, CVE, and priority data
4. **Integration Tests**: The full pipeline produces the expected output with correct counts

The comparison between source data and aggregated results should show:
- The same number of source documents (3)
- Correct grouping into a single target document
- Correct COUNT operations (2 for CVE-2023-1234, 1 for CVE-2023-5678)
- Correct priority counts (2 High, 1 Medium)
- All instances accounted for in both CVE and priority aggregations

This comprehensive test suite ensures that the MongoDB aggregation implementation works correctly and produces the expected results.
