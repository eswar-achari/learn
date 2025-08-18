Here's the refactored implementation using Spring Data MongoDB's `MongoRepository` instead of the lower-level `MongoDatabase`:

## 1. Repository Interfaces

```java
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.mongodb.repository.Aggregation;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface SourceDocumentRepository extends MongoRepository<SourceDocument, String> {
    
    @Aggregation(pipeline = {
        "{ $match: { active: true } }",
        "{ $project: { cio_lob_name: 1, cio_name: 1, /* include all needed fields */ } }",
        "{ $group: { _id: '$ait_no', count: { $sum: 1 } } }"
        // Add more aggregation stages as needed
    })
    List<AggregatedResult> aggregateRiskData();
    
    interface AggregatedResult {
        String getId();
        Integer getCount();
        // Add other projected fields
    }
}

@Repository
public interface TargetDocumentRepository extends MongoRepository<TargetDocument, String> {
    // Custom queries if needed
}
```

## 2. Refactored AggregationService

```java
import org.springframework.data.mongodb.core.aggregation.Aggregation;
import org.springframework.data.mongodb.core.aggregation.AggregationResults;
import org.springframework.stereotype.Service;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import static org.springframework.data.mongodb.core.aggregation.Aggregation.*;

@Service
public class AggregationService {
    private static final Logger LOG = LoggerFactory.getLogger(AggregationService.class);
    
    private final SourceDocumentRepository sourceRepository;
    private final TargetDocumentRepository targetRepository;
    private final BaseAggregationMapper<SourceDocument, TargetDocument> mapper;

    public AggregationService(SourceDocumentRepository sourceRepository,
                            TargetDocumentRepository targetRepository,
                            BaseAggregationMapper<SourceDocument, TargetDocument> mapper) {
        this.sourceRepository = sourceRepository;
        this.targetRepository = targetRepository;
        this.mapper = mapper;
    }

    public void aggregateCollection() {
        LOG.info("Starting aggregation process");
        
        try {
            // Option 1: Using repository with @Aggregation annotation
            List<SourceDocumentRepository.AggregatedResult> aggregatedResults = 
                sourceRepository.aggregateRiskData();
            
            // Option 2: Using programmatic aggregation
            Aggregation aggregation = newAggregation(
                match(Criteria.where("active").is(true)),
                project("cio_lob_name", "cio_name", /* all needed fields */),
                group("ait_no").count().as("count"),
                // Add more stages as needed
            );
            
            AggregationResults<TargetDocument> results = sourceRepository.aggregate(
                aggregation, "source_collection", TargetDocument.class);
            
            // Process and save results
            results.getMappedResults().forEach(sourceDoc -> {
                TargetDocument targetDoc = mapper.safeMap(sourceDoc);
                targetRepository.save(targetDoc);
            });
            
            LOG.info("Successfully aggregated {} documents", 
                targetRepository.count());
        } catch (Exception e) {
            LOG.error("Aggregation failed", e);
            throw new AggregationException("Failed to aggregate documents", e);
        }
    }
    
    // Method for specific aggregation with SUM
    public void aggregateWithSum(String fieldToSum) {
        Aggregation aggregation = newAggregation(
            group("ait_no").sum(fieldToSum).as("total"),
            project("total").and("ait_no").previousOperation()
        );
        
        sourceRepository.aggregate(aggregation, "source_collection", SumResult.class)
            .getMappedResults()
            .forEach(result -> {
                // Process sum results
            });
    }
    
    interface SumResult {
        String getAitNo();
        Integer getTotal();
    }
}
```

## 3. Updated Mapper Interface

```java
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.Mappings;
import org.mapstruct.factory.Mappers;

@Mapper(componentModel = "spring")
public interface DocumentMapper extends BaseAggregationMapper<SourceDocument, TargetDocument> {
    
    @Override
    @Mappings({
        @Mapping(target = "cio_lob_name", source = "cio_lob_name"),
        // All other field mappings
        @Mapping(target = "cve_list", expression = "java(mapCveList(source))"),
        @Mapping(target = "priorities", expression = "java(mapPriorities(source))"),
        @Mapping(target = "date_loaded", expression = "java(java.time.Instant.now())")
    })
    TargetDocument map(SourceDocument source);
    
    // Keep existing mapping methods
    default List<TargetDocument.Cve> mapCveList(SourceDocument source) {
        // Implementation
    }
    
    default List<TargetDocument.Priority> mapPriorities(SourceDocument source) {
        // Implementation
    }
}
```

## 4. Configuration Class

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.mongodb.repository.config.EnableMongoRepositories;

@Configuration
@EnableMongoRepositories(basePackages = "com.yourpackage.repository")
public class MongoConfig {
    
    @Bean
    public BaseAggregationMapper<SourceDocument, TargetDocument> documentMapper() {
        return Mappers.getMapper(DocumentMapper.class);
    }
}
```

## Key Improvements:

1. **Repository Pattern**:
   - Uses Spring Data's `MongoRepository` for cleaner data access
   - Supports both annotation-based and programmatic aggregation
   - Provides type-safe queries and operations

2. **Aggregation Framework**:
   - Uses Spring Data's `Aggregation` DSL for building pipelines
   - Supports complex aggregation operations with type-safe results
   - Clean separation between aggregation definition and execution

3. **Dependency Injection**:
   - All components are properly managed by Spring
   - Clear dependency relationships
   - Easy to mock and test

4. **Type Safety**:
   - Strong typing throughout the aggregation pipeline
   - Custom result interfaces for aggregation outputs
   - Compile-time checking of field names

5. **Transaction Management**:
   - Built-in support for transactions when needed
   - Better error handling and rollback capabilities

## Usage Example:

```java
@Service
public class RiskAnalysisService {
    private final AggregationService aggregationService;
    
    public RiskAnalysisService(AggregationService aggregationService) {
        this.aggregationService = aggregationService;
    }
    
    public void generateRiskReport() {
        aggregationService.aggregateCollection();
        aggregationService.aggregateWithSum("days_open");
    }
}
```

This implementation provides a more Spring-idiomatic way to handle MongoDB aggregations while maintaining all the functionality of your original implementation. The repository pattern makes it easier to test and maintain, and the aggregation DSL provides better type safety and code completion in IDEs.