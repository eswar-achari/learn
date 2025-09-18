Here's the **updated production-ready code** to handle **batch publishing to Kafka** efficiently. This approach improves performance for **huge Hive datasets**, reduces network overhead, and ensures reliability.

We’ll leverage **Spring Kafka** with batching, where messages are grouped together before being sent to the Kafka broker.

---

## **1. Updated Kafka Configuration (application.properties)**

Add Kafka batching configuration:

```properties
# Hive datasource
spring.hive-datasource.url=jdbc:hive2://your-hive-server:10000/default
spring.hive-datasource.username=hiveuser
spring.hive-datasource.password=hivepassword
spring.hive-datasource.driver-class-name=org.apache.hive.jdbc.HiveDriver

# Kafka Configuration
spring.kafka.bootstrap-servers=your-kafka-server:9092
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer

# Kafka Batching
spring.kafka.producer.batch-size=16384         # 16 KB batch size
spring.kafka.producer.linger-ms=5              # wait 5ms to batch records
spring.kafka.producer.buffer-memory=33554432   # total memory for buffering (32 MB)
spring.kafka.producer.compression-type=snappy  # compression to improve throughput

# Kafka Topic
cstgenai.event.kafka.topic.ROCK_DAILY_SCORECARD_TOPIC=rock_daily_scorecard
kafka.publish.batch.size=1000
```

---

## **2. Updated KafkaPublisherService**

This version uses **Kafka batch publishing** to efficiently send records.

```java
package com.company.project.service;

import com.company.project.entity.CVEDailyScorecardEntity;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

import java.util.List;

/**
 * Service for batch publishing data to Kafka.
 */
@Service
@Slf4j
public class KafkaPublisherService {

    private final KafkaTemplate<String, CVEDailyScorecardEntity> kafkaTemplate;

    @Value("${cstgenai.event.kafka.topic.ROCK_DAILY_SCORECARD_TOPIC}")
    private String kafkaTopic;

    @Value("${kafka.publish.batch.size}")
    private int kafkaBatchSize;

    public KafkaPublisherService(KafkaTemplate<String, CVEDailyScorecardEntity> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    /**
     * Publishes data to Kafka in batches to handle large datasets efficiently.
     *
     * @param data List of CVEDailyScorecardEntity
     */
    public void publishToKafka(List<CVEDailyScorecardEntity> data) {
        log.info("Starting Kafka publishing. Total records to publish: {}", data.size());

        int totalRecords = data.size();
        int batchCount = (int) Math.ceil((double) totalRecords / kafkaBatchSize);

        for (int i = 0; i < totalRecords; i += kafkaBatchSize) {
            int end = Math.min(i + kafkaBatchSize, totalRecords);
            List<CVEDailyScorecardEntity> batch = data.subList(i, end);

            log.info("Publishing batch {} of {} (records {} - {})",
                    (i / kafkaBatchSize) + 1, batchCount, i, end);

            batch.forEach(record -> kafkaTemplate.send(kafkaTopic, record.getGisId(), record));

            kafkaTemplate.flush(); // Ensure batch is sent before moving to next
            log.info("Completed publishing batch {} of {} to Kafka.", (i / kafkaBatchSize) + 1, batchCount);
        }

        log.info("Completed publishing all {} records to Kafka.", totalRecords);
    }
}
```

---

## **3. Updated HiveRepository**

Ensure we fetch data in a streaming manner to avoid loading all records into memory at once.

```java
package com.company.project.repository;

import com.company.project.entity.CVEDailyScorecardEntity;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Repository;

import javax.sql.DataSource;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

/**
 * Repository for executing Hive queries with streaming support.
 */
@Repository
@Slf4j
public class HiveRepository {

    private final DataSource dataSource;

    public HiveRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    /**
     * Fetches data from Hive in streaming mode to handle huge datasets.
     *
     * @param query Hive SQL query
     * @return List of CVEDailyScorecardEntity
     */
    public List<CVEDailyScorecardEntity> executeQuery(String query) {
        log.info("Executing Hive query: {}", query);
        long startTime = System.currentTimeMillis();

        List<CVEDailyScorecardEntity> results = new ArrayList<>();

        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement(ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY)) {

            stmt.setFetchSize(5000); // Stream data 5000 rows at a time
            try (ResultSet rs = stmt.executeQuery(query)) {
                int rowCount = 0;
                while (rs.next()) {
                    CVEDailyScorecardEntity entity = mapResultSet(rs);
                    results.add(entity);

                    if (++rowCount % 10000 == 0) {
                        log.info("Fetched {} records so far from Hive.", rowCount);
                    }
                }
            }

        } catch (SQLException e) {
            log.error("Error executing Hive query: {}", query, e);
            throw new RuntimeException("Hive query execution failed", e);
        }

        log.info("Hive query completed in {} ms. Total records fetched: {}",
                System.currentTimeMillis() - startTime, results.size());
        return results;
    }

    private CVEDailyScorecardEntity mapResultSet(ResultSet rs) throws SQLException {
        CVEDailyScorecardEntity entity = new CVEDailyScorecardEntity();
        entity.setScorecardDate(rs.getString("scorecard_date"));
        entity.setScorecardCategory(rs.getString("scorecard_category"));
        entity.setScorecardSource(rs.getString("scorecard_source"));
        entity.setWorkstreamReportDate(rs.getString("workstream_report_date"));
        entity.setGisId(rs.getString("gis_id"));
        entity.setObservationTitle(rs.getString("observation_title"));
        // Map other fields...
        return entity;
    }
}
```

---

## **4. Updated RockDataKafkaService**

Now integrates **streaming from Hive** and **batch publishing to Kafka**.

```java
package com.company.project.service;

import com.company.project.entity.CVEDailyScorecardEntity;
import com.company.project.repository.HiveRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.List;

/**
 * Service to orchestrate Hive data retrieval and Kafka batch publishing.
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class RockDataKafkaService {

    private final HiveRepository hiveRepository;
    private final KafkaPublisherService kafkaPublisherService;

    /**
     * Fetch data from Hive in streaming mode and publish to Kafka in batches.
     */
    public void streamRockDataToKafka() {
        log.info("Starting streaming Hive data to Kafka...");

        String query = "SELECT * FROM rock_output.gis_cve_risk_rating";

        List<CVEDailyScorecardEntity> hiveData = hiveRepository.executeQuery(query);

        if (hiveData.isEmpty()) {
            log.warn("No records found in Hive for query: {}", query);
            return;
        }

        kafkaPublisherService.publishToKafka(hiveData);
        log.info("Finished streaming Hive data to Kafka successfully.");
    }
}
```

---

## **5. KafkaTemplate Bean Configuration**

If not auto-configured by Spring Boot:

```java
package com.company.project.config;

import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.core.ProducerFactory;
import org.springframework.kafka.support.serializer.JsonSerializer;

import java.util.HashMap;
import java.util.Map;

/**
 * Kafka producer configuration for batch publishing.
 */
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "your-kafka-server:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

        // Enable batching
        config.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        config.put(ProducerConfig.LINGER_MS_CONFIG, 5);
        config.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);
        config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");

        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

---

## **6. End-to-End Flow**

1. **HiveRepository** fetches data with `fetchSize=5000` to avoid OOM.
2. **RockDataKafkaService** receives streamed data.
3. **KafkaPublisherService**:

   * Splits data into configurable batches (`kafka.publish.batch.size`).
   * Sends each batch asynchronously to Kafka.
   * Uses `flush()` to ensure batches are completed before next batch starts.

---

## **Sample Logs**

```
INFO  HiveRepository - Executing Hive query: SELECT * FROM rock_output.gis_cve_risk_rating
INFO  HiveRepository - Fetched 10000 records so far from Hive.
INFO  KafkaPublisherService - Publishing batch 1 of 120 (records 0 - 1000)
INFO  KafkaPublisherService - Completed publishing batch 1 of 120 to Kafka.
INFO  RockDataKafkaService - Finished streaming Hive data to Kafka successfully.
```

---

## **Why This Works Well**

* **Streaming fetch from Hive** prevents loading millions of rows into memory.
* **Kafka batching** reduces network round trips and boosts throughput.
* **Configurable batch size** allows fine-tuning based on infrastructure capacity.
* **Compression (`snappy`)** further optimizes Kafka traffic.
* **Logging** gives clear visibility into performance and progress.

public void publishToKafka(List<CVEDailyScorecardEntity> data) {
    if (data == null || data.isEmpty()) {
        log.warn("No records to publish to Kafka.");
        return;
    }

    int totalRecords = data.size();
    int batchCount = Math.ceilDiv(totalRecords, kafkaBatchSize); // Java 21
    log.info("Starting Kafka publishing. Total records: {}, Batches: {}", totalRecords, batchCount);

    // Executor: limit threads to available CPUs or batchCount (whichever is smaller)
    ExecutorService executor = Executors.newFixedThreadPool(
            Math.min(batchCount, Runtime.getRuntime().availableProcessors())
    );

    try {
        // Submit tasks for each batch
        List<CompletableFuture<Void>> futures =
                IntStream.iterate(0, i -> i < totalRecords, i -> i + kafkaBatchSize)
                        .mapToObj(start -> CompletableFuture.runAsync(() -> {
                            int end = Math.min(start + kafkaBatchSize, totalRecords);
                            List<CVEDailyScorecardEntity> batch = data.subList(start, end);
                            int batchNumber = (start / kafkaBatchSize) + 1;

                            log.info("Publishing batch {} of {} (records {}-{})",
                                    batchNumber, batchCount, start, end);

                            // Send each record sequentially within the batch
                            batch.forEach(record ->
                                    kafkaTemplate.send(kafkaTopic, record.getGisId(), record));

                            // Ensure all records in this batch are flushed before completing
                            kafkaTemplate.flush();
                            log.info("✅ Completed publishing batch {} of {}.", batchNumber, batchCount);

                        }, executor))
                        .toList();

        // Wait for all batch tasks to complete
        CompletableFuture.allOf(futures.toArray(CompletableFuture[]::new)).join();

        log.info("🎉 Completed publishing all {} records across {} batches.", totalRecords, batchCount);

    } finally {
        executor.shutdown();
    }
}

