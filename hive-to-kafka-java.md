# Enhanced Hive DB Entity Classes with Kafka Integration

I'll extend the previous implementation to include optimized Kafka publishing capabilities after data retrieval from Hive.

## 1. Kafka Configuration and Producer Service

```java
package com.company.hive.kafka;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import java.util.Properties;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

/**
 * Kafka producer service for publishing Hive query results
 */
@Service
public class HiveKafkaProducerService {
    
    private static final Logger logger = LoggerFactory.getLogger(HiveKafkaProducerService.class);
    
    @Value("${kafka.bootstrap.servers:localhost:9092}")
    private String bootstrapServers;
    
    @Value("${kafka.producer.acks:all}")
    private String acks;
    
    @Value("${kafka.producer.retries:3}")
    private int retries;
    
    @Value("${kafka.producer.batch.size:16384}")
    private int batchSize;
    
    @Value("${kafka.producer.linger.ms:1}")
    private int lingerMs;
    
    @Value("${kafka.producer.buffer.memory:33554432}")
    private int bufferMemory;
    
    @Value("${kafka.producer.compression.type:snappy}")
    private String compressionType;
    
    private Producer<String, String> producer;
    
    @PostConstruct
    public void init() {
        Properties props = new Properties();
        props.put("bootstrap.servers", bootstrapServers);
        props.put("acks", acks);
        props.put("retries", retries);
        props.put("batch.size", batchSize);
        props.put("linger.ms", lingerMs);
        props.put("buffer.memory", bufferMemory);
        props.put("compression.type", compressionType);
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        
        producer = new KafkaProducer<>(props);
        logger.info("Kafka producer initialized with servers: {}", bootstrapServers);
    }
    
    @PreDestroy
    public void cleanup() {
        if (producer != null) {
            producer.close(30, TimeUnit.SECONDS);
            logger.info("Kafka producer closed");
        }
    }
    
    /**
     * Publish message to Kafka with callback for async handling
     */
    public Future<RecordMetadata> publishMessage(String topic, String key, String message) {
        ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, message);
        
        return producer.send(record, (metadata, exception) -> {
            if (exception != null) {
                logger.error("Failed to publish message to topic: {}, key: {}", topic, key, exception);
            } else {
                if (logger.isDebugEnabled()) {
                    logger.debug("Message published successfully to topic: {}, partition: {}, offset: {}",
                        metadata.topic(), metadata.partition(), metadata.offset());
                }
            }
        });
    }
    
    /**
     * Publish message to Kafka synchronously
     */
    public RecordMetadata publishMessageSync(String topic, String key, String message) {
        try {
            ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, message);
            return producer.send(record).get(30, TimeUnit.SECONDS);
        } catch (Exception e) {
            logger.error("Failed to publish message synchronously to topic: {}, key: {}", topic, key, e);
            throw new RuntimeException("Kafka publish failed", e);
        }
    }
    
    /**
     * Batch publish messages to Kafka
     */
    public void publishBatch(String topic, List<ProducerRecord<String, String>> records) {
        for (ProducerRecord<String, String> record : records) {
            producer.send(record, (metadata, exception) -> {
                if (exception != null) {
                    logger.error("Failed to publish batch message to topic: {}", topic, exception);
                }
            });
        }
        
        // Flush to ensure all messages are sent
        producer.flush();
        logger.info("Published {} messages to topic: {}", records.size(), topic);
    }
}
```

## 2. Enhanced Repository with Kafka Integration

```java
package com.company.hive.repository;

import com.company.hive.entities.BaseHiveEntity;
import com.company.hive.kafka.HiveKafkaProducerService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Repository;

import java.sql.*;
import java.util.ArrayList;
import java.util.List;
import java.util.Properties;
import java.util.concurrent.CompletableFuture;
import java.util.stream.Collectors;

/**
 * Enhanced Hive repository with Kafka integration for publishing query results
 */
@Repository
public class HiveKafkaRepository<T extends BaseHiveEntity> extends HiveRepository<T> {
    
    private static final Logger logger = LoggerFactory.getLogger(HiveKafkaRepository.class);
    
    private final HiveKafkaProducerService kafkaProducerService;
    private final ObjectMapper objectMapper;
    
    public HiveKafkaRepository(HiveKafkaProducerService kafkaProducerService, 
                              ObjectMapper objectMapper) {
        super("jdbc:hive2://localhost:10000/default", "hive", "password");
        this.kafkaProducerService = kafkaProducerService;
        this.objectMapper = objectMapper;
    }
    
    /**
     * Execute query and publish results to Kafka
     */
    public List<T> executeQueryAndPublish(T entity, String query, ResultSetMapper<T> mapper, 
                                        String kafkaTopic, String messageKey) {
        List<T> results = executeQuery(entity, query, mapper);
        
        if (!results.isEmpty()) {
            publishResultsToKafka(results, kafkaTopic, messageKey);
        }
        
        return results;
    }
    
    /**
     * Execute query with pagination and publish to Kafka in batches
     */
    public void executeQueryAndPublishBatched(T entity, String query, ResultSetMapper<T> mapper,
                                            String kafkaTopic, String messageKeyPrefix,
                                            int batchSize) {
        long startTime = System.currentTimeMillis();
        int totalRecords = 0;
        int batchCount = 0;
        
        try (Connection connection = DriverManager.getConnection(jdbcUrl, connectionProperties);
             Statement statement = connection.createStatement();
             ResultSet resultSet = statement.executeQuery(query)) {
            
            List<T> batch = new ArrayList<>(batchSize);
            
            while (resultSet.next()) {
                T resultEntity = mapper.mapResultSet(resultSet, entity.getClass().newInstance());
                batch.add(resultEntity);
                totalRecords++;
                
                if (batch.size() >= batchSize) {
                    publishBatchToKafka(batch, kafkaTopic, messageKeyPrefix, batchCount);
                    batch.clear();
                    batchCount++;
                }
            }
            
            // Publish remaining records
            if (!batch.isEmpty()) {
                publishBatchToKafka(batch, kafkaTopic, messageKeyPrefix, batchCount);
            }
            
            long executionTime = System.currentTimeMillis() - startTime;
            entity.logQueryExecution("SELECT_AND_PUBLISH", query, executionTime, totalRecords);
            
            logger.info("Published {} records to Kafka topic {} in {} batches", 
                       totalRecords, kafkaTopic, batchCount + 1);
            
        } catch (Exception e) {
            logger.error("Error executing query and publishing to Kafka: {}", query, e);
            throw new RuntimeException("Query execution and Kafka publish failed", e);
        }
    }
    
    /**
     * Execute query asynchronously and publish to Kafka
     */
    public CompletableFuture<List<T>> executeQueryAndPublishAsync(T entity, String query, 
                                                                ResultSetMapper<T> mapper,
                                                                String kafkaTopic, String messageKey) {
        return CompletableFuture.supplyAsync(() -> {
            List<T> results = executeQuery(entity, query, mapper);
            
            if (!results.isEmpty()) {
                CompletableFuture.runAsync(() -> {
                    publishResultsToKafka(results, kafkaTopic, messageKey);
                }).exceptionally(ex -> {
                    logger.error("Async Kafka publish failed for topic: {}", kafkaTopic, ex);
                    return null;
                });
            }
            
            return results;
        });
    }
    
    /**
     * Publish results to Kafka as JSON messages
     */
    private void publishResultsToKafka(List<T> results, String topic, String messageKey) {
        long startTime = System.currentTimeMillis();
        int successCount = 0;
        int failureCount = 0;
        
        for (T result : results) {
            try {
                String jsonMessage = objectMapper.writeValueAsString(result);
                kafkaProducerService.publishMessage(topic, messageKey, jsonMessage);
                successCount++;
            } catch (Exception e) {
                logger.error("Failed to serialize or publish message to Kafka", e);
                failureCount++;
            }
        }
        
        long publishTime = System.currentTimeMillis() - startTime;
        logger.info("Published {} messages to Kafka topic {} ({} failed) in {}ms", 
                   successCount, topic, failureCount, publishTime);
    }
    
    /**
     * Publish batch to Kafka with optimized serialization
     */
    private void publishBatchToKafka(List<T> batch, String topic, String messageKeyPrefix, int batchNumber) {
        try {
            List<ProducerRecord<String, String>> records = batch.stream()
                .map(item -> {
                    try {
                        String key = messageKeyPrefix + "_batch_" + batchNumber;
                        String value = objectMapper.writeValueAsString(item);
                        return new ProducerRecord<>(topic, key, value);
                    } catch (Exception e) {
                        logger.error("Failed to serialize message for Kafka", e);
                        return null;
                    }
                })
                .filter(record -> record != null)
                .collect(Collectors.toList());
            
            kafkaProducerService.publishBatch(topic, records);
            
        } catch (Exception e) {
            logger.error("Failed to publish batch {} to Kafka topic {}", batchNumber, topic, e);
        }
    }
    
    /**
     * Optimized method for large result sets with streaming to Kafka
     */
    public void streamQueryResultsToKafka(T entity, String query, ResultSetMapper<T> mapper,
                                        String kafkaTopic, String messageKeyPrefix,
                                        int batchSize, int flushIntervalMs) {
        long startTime = System.currentTimeMillis();
        int totalRecords = 0;
        int batchCount = 0;
        
        try (Connection connection = DriverManager.getConnection(jdbcUrl, connectionProperties);
             Statement statement = connection.createStatement();
             ResultSet resultSet = statement.executeQuery(query)) {
            
            List<CompletableFuture<Void>> publishFutures = new ArrayList<>();
            List<T> currentBatch = new ArrayList<>(batchSize);
            long lastFlushTime = System.currentTimeMillis();
            
            while (resultSet.next()) {
                T resultEntity = mapper.mapResultSet(resultSet, entity.getClass().newInstance());
                currentBatch.add(resultEntity);
                totalRecords++;
                
                // Check if we should flush the batch
                boolean shouldFlush = currentBatch.size() >= batchSize || 
                                    (System.currentTimeMillis() - lastFlushTime) >= flushIntervalMs;
                
                if (shouldFlush && !currentBatch.isEmpty()) {
                    final List<T> batchToPublish = new ArrayList<>(currentBatch);
                    final int currentBatchCount = batchCount;
                    
                    // Publish asynchronously to not block the result set processing
                    CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
                        publishBatchToKafka(batchToPublish, kafkaTopic, messageKeyPrefix, currentBatchCount);
                    });
                    
                    publishFutures.add(future);
                    
                    currentBatch.clear();
                    batchCount++;
                    lastFlushTime = System.currentTimeMillis();
                }
            }
            
            // Publish any remaining records
            if (!currentBatch.isEmpty()) {
                publishBatchToKafka(currentBatch, kafkaTopic, messageKeyPrefix, batchCount);
            }
            
            // Wait for all async publishes to complete
            CompletableFuture.allOf(publishFutures.toArray(new CompletableFuture[0])).join();
            
            long executionTime = System.currentTimeMillis() - startTime;
            entity.logQueryExecution("STREAM_TO_KAFKA", query, executionTime, totalRecords);
            
            logger.info("Streamed {} records to Kafka topic {} in {} batches, time: {}ms", 
                       totalRecords, kafkaTopic, batchCount + 1, executionTime);
            
        } catch (Exception e) {
            logger.error("Error streaming query results to Kafka: {}", query, e);
            throw new RuntimeException("Streaming to Kafka failed", e);
        }
    }
}
```

## 3. Enhanced Service Layer with Kafka Integration

```java
package com.company.hive.service;

import com.company.hive.entities.SalesDataEntity;
import com.company.hive.kafka.HiveKafkaProducerService;
import com.company.hive.repository.HiveKafkaRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.concurrent.CompletableFuture;

/**
 * Enhanced service with Kafka integration for publishing Hive query results
 */
@Service
public class SalesDataKafkaService {
    
    private static final Logger logger = LoggerFactory.getLogger(SalesDataKafkaService.class);
    
    private final HiveKafkaRepository<SalesDataEntity> hiveKafkaRepository;
    private final HiveQueryOptimizerService optimizerService;
    
    @Value("${kafka.topic.sales.data:sales-data}")
    private String salesDataTopic;
    
    @Value("${kafka.publish.batch.size:1000}")
    private int kafkaBatchSize;
    
    @Value("${kafka.publish.flush.interval.ms:5000}")
    private int kafkaFlushIntervalMs;
    
    public SalesDataKafkaService(HiveKafkaRepository<SalesDataEntity> hiveKafkaRepository,
                               HiveQueryOptimizerService optimizerService) {
        this.hiveKafkaRepository = hiveKafkaRepository;
        this.optimizerService = optimizerService;
    }
    
    /**
     * Get sales by region and publish to Kafka
     */
    public List<SalesDataEntity> getSalesByRegionAndPublish(String region, String startDate, String endDate) {
        SalesDataEntity entity = new SalesDataEntity();
        
        // Generate optimized query
        String query = entity.generateRegionSalesQuery(region, startDate, endDate, 0); // 0 = no limit
        
        // Execute query and publish to Kafka
        long startTime = System.currentTimeMillis();
        List<SalesDataEntity> results = hiveKafkaRepository.executeQueryAndPublish(
            entity, query, this::mapSalesDataResultSet, 
            salesDataTopic, "region_" + region
        );
        
        long executionTime = System.currentTimeMillis() - startTime;
        
        // Analyze query performance
        optimizerService.analyzeQueryPerformance(entity, query, executionTime);
        
        logger.info("Retrieved and published {} sales records for region: {}", results.size(), region);
        
        return results;
    }
    
    /**
     * Stream large sales dataset directly to Kafka with batching
     */
    public void streamSalesDataToKafka(String startDate, String endDate) {
        SalesDataEntity entity = new SalesDataEntity();
        
        String query = "SELECT * FROM " + entity.getFullTableName() + 
                      " WHERE sale_date >= '" + startDate + "'" +
                      " AND sale_date <= '" + endDate + "'";
        
        logger.info("Starting to stream sales data to Kafka for date range: {} to {}", startDate, endDate);
        
        hiveKafkaRepository.streamQueryResultsToKafka(
            entity, query, this::mapSalesDataResultSet,
            salesDataTopic, "sales_stream",
            kafkaBatchSize, kafkaFlushIntervalMs
        );
        
        logger.info("Completed streaming sales data to Kafka for date range: {} to {}", startDate, endDate);
    }
    
    /**
     * Process sales data asynchronously and publish to Kafka
     */
    public CompletableFuture<List<SalesDataEntity>> processSalesDataAsync(String region, String date) {
        SalesDataEntity entity = new SalesDataEntity();
        
        String query = entity.generateRegionSalesQuery(region, date, date, 0);
        
        return hiveKafkaRepository.executeQueryAndPublishAsync(
            entity, query, this::mapSalesDataResultSet,
            salesDataTopic, "async_region_" + region
        ).whenComplete((results, exception) -> {
            if (exception != null) {
                logger.error("Async sales data processing failed for region: {}", region, exception);
            } else {
                logger.info("Async processing completed for region: {}, records: {}", region, results.size());
            }
        });
    }
    
    /**
     * Map result set to SalesDataEntity
     */
    private SalesDataEntity mapSalesDataResultSet(ResultSet resultSet, SalesDataEntity entity) throws SQLException {
        entity.setSaleId(resultSet.getLong("sale_id"));
        entity.setProductId(resultSet.getString("product_id"));
        entity.setCustomerId(resultSet.getString("customer_id"));
        entity.setQuantity(resultSet.getInt("quantity"));
        entity.setAmount(resultSet.getBigDecimal("amount"));
        entity.setSaleTimestamp(resultSet.getTimestamp("sale_timestamp"));
        entity.setRegion(resultSet.getString("region"));
        entity.setStoreId(resultSet.getString("store_id"));
        entity.setPaymentMethod(resultSet.getString("payment_method"));
        entity.setPromotionCode(resultSet.getString("promotion_code"));
        entity.setCustomerAge(resultSet.getInt("customer_age"));
        entity.setCustomerSegment(resultSet.getString("customer_segment"));
        entity.setSaleDate(resultSet.getString("sale_date"));
        entity.setSaleHour(resultSet.getString("sale_hour"));
        
        return entity;
    }
    
    /**
     * Get sales statistics and publish to different Kafka topic
     */
    public void publishSalesStatistics(String date) {
        SalesDataEntity entity = new SalesDataEntity();
        
        String query = "SELECT region, COUNT(*) as total_sales, SUM(amount) as total_revenue " +
                      "FROM " + entity.getFullTableName() +
                      " WHERE sale_date = '" + date + "'" +
                      " GROUP BY region";
        
        hiveKafkaRepository.executeQueryAndPublishBatched(
            entity, query, this::mapSalesStatsResultSet,
            "sales-statistics", "stats_" + date,
            100 // Smaller batch size for stats
        );
    }
    
    /**
     * Map result set for sales statistics
     */
    private SalesDataEntity mapSalesStatsResultSet(ResultSet resultSet, SalesDataEntity entity) throws SQLException {
        // Create a simplified entity for statistics
        entity.setRegion(resultSet.getString("region"));
        // We can use other fields to store stats data
        entity.setQuantity(resultSet.getInt("total_sales"));
        // For demonstration, we're reusing the entity but in real scenario,
        // you might want a different DTO for statistics
        
        return entity;
    }
}
```

## 4. Application Configuration

```java
package com.company.hive.config;

import com.company.hive.kafka.HiveKafkaProducerService;
import com.company.hive.repository.HiveKafkaRepository;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

@Configuration
public class ApplicationConfig {
    
    @Bean
    @Primary
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        // Additional configuration if needed
        return mapper;
    }
    
    @Bean
    public HiveKafkaProducerService hiveKafkaProducerService() {
        return new HiveKafkaProducerService();
    }
    
    // Other beans...
}
```

## 5. Enhanced Base Entity with Kafka Metadata

```java
package com.company.hive.entities;

import java.util.UUID;

/**
 * Enhanced base entity with Kafka publishing metadata
 */
public abstract class BaseHiveKafkaEntity extends BaseHiveEntity {
    
    private String kafkaPublishId;
    private long kafkaPublishTimestamp;
    private String kafkaTopic;
    
    public BaseHiveKafkaEntity(String databaseName, String tableName) {
        super(databaseName, tableName);
    }
    
    /**
     * Generate a unique ID for Kafka message tracking
     */
    public String generateKafkaMessageId() {
        this.kafkaPublishId = UUID.randomUUID().toString();
        this.kafkaPublishTimestamp = System.currentTimeMillis();
        return kafkaPublishId;
    }
    
    /**
     * Get Kafka message metadata as JSON
     */
    public String getKafkaMetadata() {
        return String.format("{\"publishId\":\"%s\",\"timestamp\":%d,\"sourceTable\":\"%s\"}",
                           kafkaPublishId, kafkaPublishTimestamp, getFullTableName());
    }
    
    // Getters and setters
    public String getKafkaPublishId() { return kafkaPublishId; }
    public void setKafkaPublishId(String kafkaPublishId) { this.kafkaPublishId = kafkaPublishId; }
    
    public long getKafkaPublishTimestamp() { return kafkaPublishTimestamp; }
    public void setKafkaPublishTimestamp(long kafkaPublishTimestamp) { 
        this.kafkaPublishTimestamp = kafkaPublishTimestamp; 
    }
    
    public String getKafkaTopic() { return kafkaTopic; }
    public void setKafkaTopic(String kafkaTopic) { this.kafkaTopic = kafkaTopic; }
}
```

## Key Features of the Kafka Integration:

1. **Optimized Publishing**: Batch processing and async operations for efficient Kafka publishing
2. **Streaming Support**: Direct streaming from Hive ResultSet to Kafka without loading all data into memory
3. **Error Handling**: Comprehensive error handling and logging for Kafka operations
4. **Performance Monitoring**: Tracking of publish times and success rates
5. **Flexible Configuration**: Configurable batch sizes and flush intervals
6. **Async Operations**: Non-blocking operations for better throughput
7. **JSON Serialization**: Automatic conversion of entities to JSON for Kafka messages
8. **Metadata Tracking**: Message tracking with unique IDs and timestamps

## Usage Example:

```java
// Get sales data and publish to Kafka
List<SalesDataEntity> sales = salesDataKafkaService.getSalesByRegionAndPublish(
    "north-america", "2023-01-01", "2023-01-31"
);

// Stream large dataset directly to Kafka
salesDataKafkaService.streamSalesDataToKafka("2023-01-01", "2023-12-31");

// Process data asynchronously
CompletableFuture<List<SalesDataEntity>> future = salesDataKafkaService.processSalesDataAsync(
    "europe", "2023-06-01"
);
```

This implementation provides a robust, production-ready solution for fetching data from Hive and publishing it to Kafka with optimized performance, comprehensive logging, and error handling.
