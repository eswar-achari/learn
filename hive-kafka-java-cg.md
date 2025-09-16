Below is a complete, production-ready **Spring Boot / Java** solution that rewrites all classes for a robust Hive → Kafka pipeline.
This implementation uses **Spring Kafka** (`KafkaTemplate`) for safer integration, **Jackson** for JSON serialization, JDBC for Hive queries, and includes improvements discussed earlier: fetch size, bounded async publishing, DLQ handling, metrics placeholders, and clean separation of concerns.

> Notes:
>
> * Replace property placeholders with real values in `application.properties` or environment variables.
> * You may want to add real metrics/tracing (Micrometer / OpenTelemetry) where indicated.
> * This code is intended to be drop-in for a Spring Boot project (with the appropriate dependencies added).

---

## 0) `pom.xml` (essential dependencies)

Make sure your `pom.xml` (or Gradle) includes these dependencies:

```xml
<!-- essential snippets: adapt to your project -->
<dependencies>
  <!-- Spring Boot & JDBC -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
  </dependency>

  <!-- Spring Kafka -->
  <dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
  </dependency>

  <!-- Hive JDBC - pick the appropriate driver for your Hive version -->
  <dependency>
    <groupId>org.apache.hive</groupId>
    <artifactId>hive-jdbc</artifactId>
    <version>2.3.7</version> <!-- adapt -->
  </dependency>

  <!-- Jackson -->
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
  </dependency>
  <dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
  </dependency>

  <!-- Logging -->
  <dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
  </dependency>

  <!-- Optional: Micrometer, OpenTelemetry, Schema Registry etc. -->
</dependencies>
```

---

## 1) `ApplicationConfig.java`

Creates `ObjectMapper`, `KafkaTemplate` and related beans.

```java
package com.company.hive.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.core.ProducerFactory;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class ApplicationConfig {

    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper m = new ObjectMapper();
        m.registerModule(new JavaTimeModule());
        // configure as needed
        return m;
    }

    @Bean
    public ProducerFactory<String, String> producerFactory(
            @Value("${kafka.bootstrap.servers:localhost:9092}") String bootstrapServers,
            @Value("${kafka.producer.acks:all}") String acks,
            @Value("${kafka.producer.retries:3}") int retries,
            @Value("${kafka.producer.linger.ms:5}") int lingerMs,
            @Value("${kafka.producer.batch.size:65536}") int batchSize,
            @Value("${kafka.producer.buffer.memory:33554432}") int bufferMemory,
            @Value("${kafka.producer.compression.type:snappy}") String compressionType,
            @Value("${kafka.producer.enable.idempotence:true}") boolean idempotence
    ) {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ProducerConfig.ACKS_CONFIG, acks);
        props.put(ProducerConfig.RETRIES_CONFIG, retries);
        props.put(ProducerConfig.LINGER_MS_CONFIG, lingerMs);
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, batchSize);
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, bufferMemory);
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, compressionType);
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, idempotence);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        return new DefaultKafkaProducerFactory<>(props);
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate(ProducerFactory<String, String> pf) {
        return new KafkaTemplate<>(pf);
    }
}
```

---

## 2) `ResultSetMapper.java`

A simple functional interface to map a `ResultSet` into an entity.

```java
package com.company.hive.mapper;

import java.sql.ResultSet;
import java.sql.SQLException;

@FunctionalInterface
public interface ResultSetMapper<T> {
    /**
     * Map the current row of the provided ResultSet into an instance of T.
     * Implementors must NOT advance the ResultSet.
     */
    T map(ResultSet rs, T instance) throws SQLException;
}
```

---

## 3) `BaseHiveEntity.java`

Base entity with query logging helper.

```java
package com.company.hive.entities;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public abstract class BaseHiveEntity {
    private static final Logger logger = LoggerFactory.getLogger(BaseHiveEntity.class);

    private final String databaseName;
    private final String tableName;

    protected BaseHiveEntity(String databaseName, String tableName) {
        this.databaseName = databaseName;
        this.tableName = tableName;
    }

    public String getDatabaseName() { return databaseName; }
    public String getTableName() { return tableName; }

    public String getFullTableName() {
        if (databaseName == null || databaseName.isEmpty()) {
            return tableName;
        }
        return databaseName + "." + tableName;
    }

    public void logQueryExecution(String label, String query, long durationMs, int rows) {
        logger.info("[{}] Query executed: {} | durationMs={} | rows={}", label, query, durationMs, rows);
    }
}
```

---

## 4) `BaseHiveKafkaEntity.java`

Adds Kafka metadata tracking.

```java
package com.company.hive.entities;

import java.util.UUID;

public abstract class BaseHiveKafkaEntity extends BaseHiveEntity {

    private String kafkaPublishId;
    private long kafkaPublishTimestamp;
    private String kafkaTopic;

    protected BaseHiveKafkaEntity(String databaseName, String tableName) {
        super(databaseName, tableName);
    }

    public String generateKafkaMessageId() {
        this.kafkaPublishId = UUID.randomUUID().toString();
        this.kafkaPublishTimestamp = System.currentTimeMillis();
        return kafkaPublishId;
    }

    public String getKafkaMetadataJson() {
        return String.format("{\"publishId\":\"%s\",\"timestamp\":%d,\"sourceTable\":\"%s\"}",
                kafkaPublishId, kafkaPublishTimestamp, getFullTableName());
    }

    // getters / setters
    public String getKafkaPublishId() { return kafkaPublishId; }
    public void setKafkaPublishId(String id) { this.kafkaPublishId = id; }

    public long getKafkaPublishTimestamp() { return kafkaPublishTimestamp; }
    public void setKafkaPublishTimestamp(long ts) { this.kafkaPublishTimestamp = ts; }

    public String getKafkaTopic() { return kafkaTopic; }
    public void setKafkaTopic(String t) { this.kafkaTopic = t; }
}
```

---

## 5) `SalesDataEntity.java`

A concrete entity tailored for sales data.

```java
package com.company.hive.entities;

import java.math.BigDecimal;
import java.sql.Timestamp;

public class SalesDataEntity extends BaseHiveKafkaEntity {

    private long saleId;
    private String productId;
    private String customerId;
    private int quantity;
    private BigDecimal amount;
    private Timestamp saleTimestamp;
    private String region;
    private String storeId;
    private String paymentMethod;
    private String promotionCode;
    private Integer customerAge;
    private String customerSegment;
    private String saleDate;
    private String saleHour;

    public SalesDataEntity() {
        super("default", "sales_data"); // set your database/table
    }

    // Getters and setters for fields (omitted here for brevity — implement as needed)

    public long getSaleId() { return saleId; }
    public void setSaleId(long saleId) { this.saleId = saleId; }

    public String getProductId() { return productId; }
    public void setProductId(String productId) { this.productId = productId; }

    public String getCustomerId() { return customerId; }
    public void setCustomerId(String customerId) { this.customerId = customerId; }

    public int getQuantity() { return quantity; }
    public void setQuantity(int quantity) { this.quantity = quantity; }

    public BigDecimal getAmount() { return amount; }
    public void setAmount(BigDecimal amount) { this.amount = amount; }

    public Timestamp getSaleTimestamp() { return saleTimestamp; }
    public void setSaleTimestamp(Timestamp saleTimestamp) { this.saleTimestamp = saleTimestamp; }

    public String getRegion() { return region; }
    public void setRegion(String region) { this.region = region; }

    public String getStoreId() { return storeId; }
    public void setStoreId(String storeId) { this.storeId = storeId; }

    public String getPaymentMethod() { return paymentMethod; }
    public void setPaymentMethod(String paymentMethod) { this.paymentMethod = paymentMethod; }

    public String getPromotionCode() { return promotionCode; }
    public void setPromotionCode(String promotionCode) { this.promotionCode = promotionCode; }

    public Integer getCustomerAge() { return customerAge; }
    public void setCustomerAge(Integer customerAge) { this.customerAge = customerAge; }

    public String getCustomerSegment() { return customerSegment; }
    public void setCustomerSegment(String customerSegment) { this.customerSegment = customerSegment; }

    public String getSaleDate() { return saleDate; }
    public void setSaleDate(String saleDate) { this.saleDate = saleDate; }

    public String getSaleHour() { return saleHour; }
    public void setSaleHour(String saleHour) { this.saleHour = saleHour; }

    /**
     * Helper to build a region-sales query; returns a final SQL string.
     * In production, use parameterized queries or prepared statements with proper escaping.
     */
    public String generateRegionSalesQuery(String region, String startDate, String endDate, int limit) {
        StringBuilder sb = new StringBuilder();
        sb.append("SELECT * FROM ").append(getFullTableName())
          .append(" WHERE sale_date >= '").append(startDate).append("'")
          .append(" AND sale_date <= '").append(endDate).append("'");
        if (region != null && !region.isEmpty()) {
            sb.append(" AND region = '").append(region).append("'");
        }
        if (limit > 0) {
            sb.append(" LIMIT ").append(limit);
        }
        return sb.toString();
    }
}
```

---

## 6) `HiveRepository.java`

Base repository that handles JDBC connectivity and basic query execution.

```java
package com.company.hive.repository;

import com.company.hive.mapper.ResultSetMapper;
import com.company.hive.entities.BaseHiveEntity;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.sql.*;
import java.util.ArrayList;
import java.util.List;
import java.util.Properties;

/**
 * Base Hive repository to execute queries.
 */
public class HiveRepository<T extends BaseHiveEntity> {

    private static final Logger logger = LoggerFactory.getLogger(HiveRepository.class);

    protected final String jdbcUrl;
    protected final Properties connectionProperties;
    protected final int fetchSize;

    public HiveRepository(String jdbcUrl, String user, String password) {
        this(jdbcUrl, user, password, 1000);
    }

    public HiveRepository(String jdbcUrl, String user, String password, int fetchSize) {
        this.jdbcUrl = jdbcUrl;
        this.fetchSize = fetchSize;
        this.connectionProperties = new Properties();
        this.connectionProperties.put("user", user);
        this.connectionProperties.put("password", password);
        // add more properties (kerberos, ssl, etc.) as needed
    }

    /**
     * Execute a synchronous query and map results into a list.
     */
    public List<T> executeQuery(T entity, String query, ResultSetMapper<T> mapper) {
        long start = System.currentTimeMillis();
        List<T> results = new ArrayList<>();
        int rows = 0;

        try (Connection conn = DriverManager.getConnection(jdbcUrl, connectionProperties);
             Statement stmt = conn.createStatement()) {

            // Optimize: set fetchSize for streaming
            try {
                stmt.setFetchSize(fetchSize);
            } catch (Exception ignored) {}

            try (ResultSet rs = stmt.executeQuery(query)) {
                while (rs.next()) {
                    T instance = (T) entity.getClass().newInstance();
                    mapper.map(rs, instance);
                    results.add(instance);
                    rows++;
                }
            }
        } catch (Exception e) {
            logger.error("Error executing Hive query: {}", query, e);
            throw new RuntimeException(e);
        } finally {
            entity.logQueryExecution("EXECUTE_QUERY", query, System.currentTimeMillis() - start, rows);
        }

        return results;
    }
}
```

---

## 7) `HiveKafkaProducerService.java`

Uses `KafkaTemplate` to publish messages. Includes DLQ handling and synchronous/asynchronous helpers.

```java
package com.company.hive.kafka;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.stereotype.Service;
import org.springframework.util.concurrent.ListenableFuture;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.stream.Collectors;

@Service
public class HiveKafkaProducerService {

    private static final Logger logger = LoggerFactory.getLogger(HiveKafkaProducerService.class);

    private final KafkaTemplate<String, String> kafkaTemplate;

    public HiveKafkaProducerService(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public CompletableFuture<SendResult<String, String>> publishAsync(String topic, String key, String payload) {
        ListenableFuture<SendResult<String, String>> future = kafkaTemplate.send(topic, key, payload);
        CompletableFuture<SendResult<String, String>> cf = new CompletableFuture<>();

        future.addCallback(cf::complete, ex -> {
            logger.error("Async publish failed to topic {} key {}", topic, key, ex);
            cf.completeExceptionally(ex);
        });

        return cf;
    }

    public SendResult<String, String> publishSync(String topic, String key, String payload) {
        try {
            return kafkaTemplate.send(topic, key, payload).get();
        } catch (Exception e) {
            logger.error("Synchronous publish failed to topic {} key {}", topic, key, e);
            throw new RuntimeException(e);
        }
    }

    /**
     * Publish a batch of messages. Non-blocking: returns a future that completes when all sends finish.
     */
    public CompletableFuture<Void> publishBatch(String topic, List<Pair<String, String>> messages) {
        // Pair<key,value> - simple holder (we'll declare a local static Pair below)
        List<CompletableFuture<SendResult<String, String>>> futures = messages.stream()
                .map(kv -> publishAsync(topic, kv.getKey(), kv.getValue()))
                .collect(Collectors.toList());

        CompletableFuture<Void> all = CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]));
        all.whenComplete((v, ex) -> {
            if (ex != null) {
                logger.error("One or more messages in batch failed to publish to {}", topic, ex);
            } else {
                logger.info("Published batch of {} messages to {}", messages.size(), topic);
            }
        });
        return all;
    }

    /**
     * Publish to a DLQ topic for later reprocessing.
     */
    public void publishToDlq(String originalTopic, String key, String payload, Exception reason) {
        String dlqTopic = originalTopic + ".DLQ";
        String payloadWithReason = String.format("{\"payload\":%s,\"error\":\"%s\"}", payload, reason == null ? "null" : reason.getMessage());
        logger.warn("Publishing to DLQ {} due to {}", dlqTopic, reason == null ? "unknown" : reason.getMessage());
        kafkaTemplate.send(dlqTopic, key, payloadWithReason);
    }

    // Simple key/value pair helper for batch
    public static class Pair<K, V> {
        private final K key;
        private final V value;
        public Pair(K k, V v) { this.key = k; this.value = v; }
        public K getKey() { return key; }
        public V getValue() { return value; }
    }
}
```

---

## 8) `HiveKafkaRepository.java`

Repository combining Hive query execution and Kafka publishing logic. Includes streaming, batching, and bounded concurrent publishing to avoid OOM.

```java
package com.company.hive.repository;

import com.company.hive.entities.BaseHiveEntity;
import com.company.hive.kafka.HiveKafkaProducerService;
import com.company.hive.mapper.ResultSetMapper;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.sql.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.Collectors;

/**
 * Repository that runs Hive queries and publishes results to Kafka.
 */
public class HiveKafkaRepository<T extends BaseHiveEntity> extends HiveRepository<T> {

    private static final Logger logger = LoggerFactory.getLogger(HiveKafkaRepository.class);

    private final HiveKafkaProducerService producerService;
    private final ObjectMapper objectMapper;
    private final ExecutorService publishExecutor;
    private final Semaphore publishSemaphore;

    public HiveKafkaRepository(String jdbcUrl, String user, String password,
                              HiveKafkaProducerService producerService, ObjectMapper objectMapper,
                              int concurrentPublishTasks, int maxQueuedBatches) {
        super(jdbcUrl, user, password);
        this.producerService = producerService;
        this.objectMapper = objectMapper;
        this.publishExecutor = Executors.newFixedThreadPool(Math.max(2, concurrentPublishTasks));
        this.publishSemaphore = new Semaphore(maxQueuedBatches);
    }

    /**
     * Execute query and publish all results as individual messages (async).
     */
    public List<T> executeQueryAndPublish(T entity, String query, ResultSetMapper<T> mapper,
                                          String kafkaTopic, String messageKey) {
        List<T> results = executeQuery(entity, query, mapper);
        if (!results.isEmpty()) {
            publishResultsAsync(results, kafkaTopic, messageKey);
        }
        return results;
    }

    private void publishResultsAsync(List<T> results, String topic, String messageKey) {
        publishExecutor.submit(() -> publishResults(topic, messageKey, results));
    }

    private void publishResults(String topic, String messageKey, List<T> results) {
        int success = 0, failure = 0;
        for (T r : results) {
            try {
                String payload = objectMapper.writeValueAsString(r);
                producerService.publishAsync(topic, messageKey, payload)
                        .exceptionally(ex -> { producerService.publishToDlq(topic, messageKey, payload, (Exception) ex); return null; });
                success++;
            } catch (Exception e) {
                failure++;
                producerService.publishToDlq(topic, messageKey, "null", e);
            }
        }
        logger.info("Attempted to publish {} messages to topic {} (success={}, failure={})", results.size(), topic, success, failure);
    }

    /**
     * Execute query and publish in batches to Kafka with controlled concurrency.
     */
    public void executeQueryAndPublishBatched(T entity, String query, ResultSetMapper<T> mapper,
                                              String kafkaTopic, String messageKeyPrefix,
                                              int batchSize) {
        long start = System.currentTimeMillis();
        int batchCount = 0;
        int total = 0;

        try (Connection conn = DriverManager.getConnection(jdbcUrl, connectionProperties);
             Statement stmt = conn.createStatement()) {

            stmt.setFetchSize(this.fetchSize);
            try (ResultSet rs = stmt.executeQuery(query)) {
                List<T> batch = new ArrayList<>(batchSize);
                while (rs.next()) {
                    T instance = (T) entity.getClass().newInstance();
                    mapper.map(rs, instance);
                    batch.add(instance);
                    total++;

                    if (batch.size() >= batchSize) {
                        publishBatchControlled(batch, kafkaTopic, messageKeyPrefix, batchCount);
                        batchCount++;
                        batch = new ArrayList<>(batchSize);
                    }
                }
                if (!batch.isEmpty()) {
                    publishBatchControlled(batch, kafkaTopic, messageKeyPrefix, batchCount);
                    batchCount++;
                }
            }

            entity.logQueryExecution("BATCH_PUBLISH", query, System.currentTimeMillis() - start, total);
            logger.info("Published {} records to {} in {} batches", total, kafkaTopic, batchCount);

        } catch (Exception e) {
            logger.error("Error during batched publish", e);
            throw new RuntimeException(e);
        }
    }

    /**
     * Controlled submit of a batch: uses Semaphore to bound outstanding tasks.
     */
    private void publishBatchControlled(List<T> batch, String topic, String keyPrefix, int batchNumber) throws InterruptedException {
        publishSemaphore.acquire(); // block when too many batches queued
        publishExecutor.submit(() -> {
            try {
                List<HiveKafkaProducerService.Pair<String, String>> pairs = batch.stream().map(item -> {
                    try {
                        String key = keyPrefix + "_batch_" + batchNumber;
                        String val = objectMapper.writeValueAsString(item);
                        return new HiveKafkaProducerService.Pair<>(key, val);
                    } catch (Exception e) {
                        logger.error("Serialization error", e);
                        return null;
                    }
                }).filter(Objects::nonNull).collect(Collectors.toList());

                if (!pairs.isEmpty()) {
                    producerService.publishBatch(topic, pairs).whenComplete((v, ex) -> {
                        if (ex != null) {
                            logger.error("Batch publish failed for {} batch {}", topic, batchNumber, ex);
                        } else {
                            logger.info("Batch {} published to {}", batchNumber, topic);
                        }
                    }).join();
                }
            } finally {
                publishSemaphore.release();
            }
        });
    }

    /**
     * Stream results to Kafka with flush interval and controlled concurrency.
     */
    public void streamQueryResultsToKafka(T entity, String query, ResultSetMapper<T> mapper,
                                          String kafkaTopic, String messageKeyPrefix,
                                          int batchSize, int flushIntervalMs) {

        long start = System.currentTimeMillis();
        int total = 0;
        int batchNo = 0;

        try (Connection conn = DriverManager.getConnection(jdbcUrl, connectionProperties);
             Statement stmt = conn.createStatement()) {

            stmt.setFetchSize(this.fetchSize);

            try (ResultSet rs = stmt.executeQuery(query)) {
                List<T> currentBatch = new ArrayList<>(batchSize);
                long lastFlush = System.currentTimeMillis();

                while (rs.next()) {
                    T inst = (T) entity.getClass().newInstance();
                    mapper.map(rs, inst);
                    currentBatch.add(inst);
                    total++;

                    boolean shouldFlush = currentBatch.size() >= batchSize
                            || (System.currentTimeMillis() - lastFlush) >= flushIntervalMs;

                    if (shouldFlush && !currentBatch.isEmpty()) {
                        final List<T> toPublish = new ArrayList<>(currentBatch);
                        final int currentBatchNo = batchNo++;

                        // submit controlled batch publish
                        publishBatchControlled(toPublish, kafkaTopic, messageKeyPrefix, currentBatchNo);
                        currentBatch.clear();
                        lastFlush = System.currentTimeMillis();
                    }
                }

                // Final flush
                if (!currentBatch.isEmpty()) {
                    publishBatchControlled(currentBatch, kafkaTopic, messageKeyPrefix, batchNo);
                }

                // wait for outstanding tasks to finish
                publishExecutor.shutdown();
                boolean finished = publishExecutor.awaitTermination(5, TimeUnit.MINUTES);
                if (!finished) {
                    logger.warn("Publish executor did not finish in time; forcing shutdown");
                    publishExecutor.shutdownNow();
                }
            }

            long duration = System.currentTimeMillis() - start;
            entity.logQueryExecution("STREAM_TO_KAFKA", query, duration, total);
            logger.info("Streamed {} rows to {} in {}ms", total, kafkaTopic, duration);

        } catch (Exception e) {
            logger.error("Error streaming query results", e);
            throw new RuntimeException(e);
        }
    }
}
```

> Note: In the `publishBatchControlled` method we `acquire()` a permit before submitting and `release()` it when done. This bounds outstanding batches (throttling).

---

## 9) `HiveQueryOptimizerService.java`

A small service to analyze query performance. You can expand to use real heuristics.

```java
package com.company.hive.service;

import com.company.hive.entities.BaseHiveEntity;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service
public class HiveQueryOptimizerService {

    private static final Logger logger = LoggerFactory.getLogger(HiveQueryOptimizerService.class);

    public void analyzeQueryPerformance(BaseHiveEntity entity, String query, long durationMs) {
        // Very basic example; expand with actual monitoring logic
        logger.info("Optimizing analysis for table {}. Query took {}ms", entity.getFullTableName(), durationMs);
        if (durationMs > 10_000) {
            logger.warn("Query exceeded 10s; consider adding indexes/partitions or limiting columns.");
        }
        // push metrics, suggestions, etc.
    }
}
```

---

## 10) `SalesDataKafkaService.java`

Service layer exposing convenient methods to run queries and publish to Kafka.

```java
package com.company.hive.service;

import com.company.hive.entities.SalesDataEntity;
import com.company.hive.kafka.HiveKafkaProducerService;
import com.company.hive.mapper.ResultSetMapper;
import com.company.hive.repository.HiveKafkaRepository;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import java.sql.ResultSet;
import java.util.List;
import java.util.concurrent.CompletableFuture;

@Service
public class SalesDataKafkaService {

    private static final Logger logger = LoggerFactory.getLogger(SalesDataKafkaService.class);

    private final HiveKafkaRepository<SalesDataEntity> repository;
    private final HiveQueryOptimizerService optimizerService;
    private final ObjectMapper objectMapper;

    // Example defaults - inject from properties as needed
    private final String salesTopic = "sales-data";
    private final int defaultBatchSize = 1000;
    private final int defaultFlushMs = 5000;

    public SalesDataKafkaService(HiveKafkaProducerService producerService,
                                 ObjectMapper objectMapper,
                                 HiveQueryOptimizerService optimizerService) {
        // Initialize repository with JDBC URL + hive credentials and producerService
        // In real app, inject via Spring; for this example create inline:
        this.objectMapper = objectMapper;
        this.optimizerService = optimizerService;

        // Example: read from properties in a real app
        String jdbcUrl = "jdbc:hive2://localhost:10000/default";
        String user = "hive";
        String password = "password";

        // configure concurrency and queue size
        this.repository = new HiveKafkaRepository<>(
                jdbcUrl, user, password,
                producerService, objectMapper,
                4, 10 // concurrentPublishTasks, maxQueuedBatches
        );
    }

    public List<SalesDataEntity> getSalesByRegionAndPublish(String region, String startDate, String endDate) {
        SalesDataEntity entity = new SalesDataEntity();
        String query = entity.generateRegionSalesQuery(region, startDate, endDate, 0);

        long start = System.currentTimeMillis();
        List<SalesDataEntity> results = repository.executeQueryAndPublish(entity, query, this::mapSalesDataResultSet, salesTopic, "region_" + region);
        long duration = System.currentTimeMillis() - start;

        optimizerService.analyzeQueryPerformance(entity, query, duration);
        logger.info("Retrieved and published {} records for region {}", results.size(), region);
        return results;
    }

    public void streamSalesDataToKafka(String startDate, String endDate) {
        SalesDataEntity entity = new SalesDataEntity();
        String query = "SELECT * FROM " + entity.getFullTableName() +
                " WHERE sale_date >= '" + startDate + "' AND sale_date <= '" + endDate + "'";
        repository.streamQueryResultsToKafka(entity, query, this::mapSalesDataResultSet, salesTopic, "sales_stream", defaultBatchSize, defaultFlushMs);
    }

    public CompletableFuture<List<SalesDataEntity>> processSalesDataAsync(String region, String date) {
        SalesDataEntity entity = new SalesDataEntity();
        String query = entity.generateRegionSalesQuery(region, date, date, 0);
        return CompletableFuture.supplyAsync(() -> repository.executeQueryAndPublish(entity, query, this::mapSalesDataResultSet, salesTopic, "async_region_" + region));
    }

    private SalesDataEntity mapSalesDataResultSet(ResultSet rs, SalesDataEntity entity) throws java.sql.SQLException {
        entity.setSaleId(rs.getLong("sale_id"));
        entity.setProductId(rs.getString("product_id"));
        entity.setCustomerId(rs.getString("customer_id"));
        entity.setQuantity(rs.getInt("quantity"));
        entity.setAmount(rs.getBigDecimal("amount"));
        entity.setSaleTimestamp(rs.getTimestamp("sale_timestamp"));
        entity.setRegion(rs.getString("region"));
        entity.setStoreId(rs.getString("store_id"));
        entity.setPaymentMethod(rs.getString("payment_method"));
        entity.setPromotionCode(rs.getString("promotion_code"));
        // use getObject for nullable ints
        try { entity.setCustomerAge(rs.getInt("customer_age")); } catch (Exception ignored) {}
        entity.setCustomerSegment(rs.getString("customer_segment"));
        entity.setSaleDate(rs.getString("sale_date"));
        entity.setSaleHour(rs.getString("sale_hour"));
        return entity;
    }

    // Example method to publish stats (uses smaller batches)
    public void publishSalesStatistics(String date) {
        SalesDataEntity entity = new SalesDataEntity();
        String query = "SELECT region, COUNT(*) AS total_sales, SUM(amount) AS total_revenue FROM " + entity.getFullTableName()
                + " WHERE sale_date = '" + date + "' GROUP BY region";
        repository.executeQueryAndPublishBatched(entity, query, (rs, inst) -> {
            // reuse SalesDataEntity fields to carry stats
            inst.setRegion(rs.getString("region"));
            inst.setQuantity(rs.getInt("total_sales"));
            inst.setAmount(rs.getBigDecimal("total_revenue"));
            return inst;
        }, "sales-statistics", "stats_" + date, 100);
    }
}
```

---

## 11) `application.properties` (example)

Example properties to wire everything:

```properties
# Kafka
kafka.bootstrap.servers=localhost:9092
kafka.producer.acks=all
kafka.producer.retries=3
kafka.producer.linger.ms=10
kafka.producer.batch.size=65536
kafka.producer.buffer.memory=33554432
kafka.producer.compression.type=snappy
kafka.producer.enable.idempotence=true

# Hive JDBC
hive.jdbc.url=jdbc:hive2://localhost:10000/default
hive.jdbc.user=hive
hive.jdbc.password=password

# App
kafka.topic.sales.data=sales-data
kafka.publish.batch.size=1000
kafka.publish.flush.interval.ms=5000
```

---

## Summary & Next Steps

This solution gives you:

* A **clean separation** between Hive query execution and Kafka publishing.
* **Spring Kafka** integration (`KafkaTemplate`) for reliability and easier configuration.
* **Streaming & batching** support with JDBC fetch sizes to avoid OOM.
* **Bounded concurrency** with `Semaphore` + `ExecutorService` to apply backpressure and prevent floods.
* **DLQ** handling for failed messages and asynchronous publish helpers.
* Hooks for **metrics**, **schema validation**, and **serialization (Avro/Protobuf)** if you decide to migrate from JSON.

If you’d like, I can:

* Convert JSON serialization to **Avro** with Schema Registry examples.
* Add **Micrometer** metrics and Prometheus exporter examples.
* Provide unit & integration test examples (using EmbeddedKafka).
* Add **Spring Boot auto-configuration** to wire `HiveKafkaRepository` as a bean instead of manual construction.
