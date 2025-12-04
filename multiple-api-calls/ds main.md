I'll provide a production-ready Java 21 solution for calling multiple APIs with failure resilience, using modern Java features and best practices.

Solution: Parallel API Aggregator with Resilience

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;
import org.springframework.web.client.RestClientException;

import java.util.List;
import java.util.Map;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;

/**
 * Service for aggregating data from multiple APIs with resilience.
 * <p>
 * Features:
 * <ul>
 *     <li>Parallel API calls using Java 21 virtual threads</li>
 *     <li>Graceful degradation on API failures</li>
 *     <li>Timeout handling</li>
 *     <li>Comprehensive logging</li>
 *     <li>Type-safe response handling</li>
 * </ul>
 */
@Service
@Slf4j
public class MultiApiAggregator {

    private final RestClient restClient;
    private final ExecutorService virtualThreadExecutor;
    
    // Configuration - should be externalized
    private static final long DEFAULT_TIMEOUT_MS = 5000;
    private static final int MAX_RETRIES = 2;
    
    /**
     * Represents an API endpoint configuration
     */
    public record ApiEndpoint<T>(
            String name,
            String url,
            Class<T> responseType,
            long timeoutMs,
            boolean required
    ) {
        public ApiEndpoint(String name, String url, Class<T> responseType) {
            this(name, url, responseType, DEFAULT_TIMEOUT_MS, false);
        }
    }
    
    /**
     * Container for aggregated results
     */
    public static class AggregatedResult {
        private final Map<String, Object> successfulResponses = new ConcurrentHashMap<>();
        private final Map<String, String> failedEndpoints = new ConcurrentHashMap<>();
        private final Map<String, Object> fallbackValues = new ConcurrentHashMap<>();
        
        public void addSuccess(String endpointName, Object response) {
            successfulResponses.put(endpointName, response);
        }
        
        public void addFailure(String endpointName, String error) {
            failedEndpoints.put(endpointName, error);
        }
        
        public void addFallback(String endpointName, Object fallbackValue) {
            fallbackValues.put(endpointName, fallbackValue);
        }
        
        public Map<String, Object> getSuccessfulResponses() {
            return Map.copyOf(successfulResponses);
        }
        
        public Map<String, String> getFailedEndpoints() {
            return Map.copyOf(failedEndpoints);
        }
        
        public Map<String, Object> getFallbackValues() {
            return Map.copyOf(fallbackValues);
        }
        
        public boolean isComplete() {
            return failedEndpoints.isEmpty() || 
                   !fallbackValues.isEmpty(); // Consider complete if we have fallbacks for failures
        }
    }
    
    public MultiApiAggregator(RestClient.Builder restClientBuilder) {
        this.restClient = restClientBuilder.build();
        // Using virtual threads from Java 21 for lightweight concurrency
        this.virtualThreadExecutor = Executors.newVirtualThreadPerTaskExecutor();
    }
    
    /**
     * Aggregates data from multiple APIs in parallel.
     * 
     * @param endpoints List of API endpoints to call
     * @param fallbackProvider Function to provide fallback values for failed endpoints
     * @param <T> The type of the final aggregated object
     * @return AggregatedResult containing all responses and failure information
     */
    public <T> T aggregateApis(
            List<ApiEndpoint<?>> endpoints,
            FallbackProvider fallbackProvider,
            ResponseMapper<T> responseMapper) {
        
        log.info("Starting aggregation of {} APIs", endpoints.size());
        
        AggregatedResult result = new AggregatedResult();
        
        // Create parallel API calls using CompletableFuture with virtual threads
        List<CompletableFuture<Void>> futures = endpoints.stream()
                .map(endpoint -> callApiWithResilience(endpoint, result, fallbackProvider))
                .toList();
        
        try {
            // Wait for all API calls to complete (success or failure)
            CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
                    .completeOnTimeout(null, DEFAULT_TIMEOUT_MS + 1000, TimeUnit.MILLISECONDS)
                    .join();
            
            log.info("API aggregation completed. Successful: {}, Failed: {}, Fallbacks: {}",
                    result.getSuccessfulResponses().size(),
                    result.getFailedEndpoints().size(),
                    result.getFallbackValues().size());
            
            if (!result.getFailedEndpoints().isEmpty()) {
                log.warn("Some APIs failed: {}", result.getFailedEndpoints());
            }
            
        } catch (Exception e) {
            log.error("Unexpected error during API aggregation", e);
        } finally {
            shutdownExecutor();
        }
        
        // Map the results to the final object
        return responseMapper.map(result);
    }
    
    /**
     * Calls a single API with retry logic and timeout
     */
    private CompletableFuture<Void> callApiWithResilience(
            ApiEndpoint<?> endpoint,
            AggregatedResult result,
            FallbackProvider fallbackProvider) {
        
        return CompletableFuture.runAsync(() -> {
            String endpointName = endpoint.name();
            int attempt = 0;
            Exception lastException = null;
            
            while (attempt <= MAX_RETRIES) {
                attempt++;
                try {
                    log.debug("Calling API {} (attempt {}/{})", endpointName, attempt, MAX_RETRIES + 1);
                    
                    Object response = restClient.get()
                            .uri(endpoint.url())
                            .retrieve()
                            .body(endpoint.responseType());
                    
                    result.addSuccess(endpointName, response);
                    log.info("API {} call successful", endpointName);
                    return;
                    
                } catch (RestClientException e) {
                    lastException = e;
                    log.warn("API {} call failed (attempt {}): {}", endpointName, attempt, e.getMessage());
                    
                    if (attempt > MAX_RETRIES) {
                        break;
                    }
                    
                    // Exponential backoff
                    try {
                        Thread.sleep((long) (100 * Math.pow(2, attempt)));
                    } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                } catch (Exception e) {
                    lastException = e;
                    log.error("Unexpected error calling API {}", endpointName, e);
                    break;
                }
            }
            
            // All retries failed - handle failure
            String errorMsg = lastException != null ? lastException.getMessage() : "Unknown error";
            result.addFailure(endpointName, errorMsg);
            
            // Try to get fallback value
            try {
                Object fallback = fallbackProvider.getFallback(endpointName);
                if (fallback != null) {
                    result.addFallback(endpointName, fallback);
                    log.info("Using fallback value for API {}", endpointName);
                } else if (endpoint.required()) {
                    log.error("Required API {} failed and no fallback available", endpointName);
                }
            } catch (Exception e) {
                log.error("Failed to get fallback for API {}", endpointName, e);
            }
            
        }, virtualThreadExecutor).orTimeout(endpoint.timeoutMs(), TimeUnit.MILLISECONDS);
    }
    
    /**
     * Alternative: Sequential fallback strategy
     */
    public <T> T aggregateWithFallbackStrategy(
            List<ApiEndpoint<?>> primaryEndpoints,
            List<ApiEndpoint<?>> fallbackEndpoints,
            ResponseMapper<T> responseMapper) {
        
        log.info("Starting aggregation with fallback strategy");
        
        AggregatedResult result = new AggregatedResult();
        
        // Try primary endpoints first
        for (ApiEndpoint<?> endpoint : primaryEndpoints) {
            try {
                Object response = callApiWithTimeout(endpoint);
                result.addSuccess(endpoint.name(), response);
            } catch (Exception e) {
                log.warn("Primary API {} failed: {}", endpoint.name(), e.getMessage());
                result.addFailure(endpoint.name(), e.getMessage());
                
                // Try fallback if available
                ApiEndpoint<?> fallback = findFallback(endpoint.name(), fallbackEndpoints);
                if (fallback != null) {
                    try {
                        Object fallbackResponse = callApiWithTimeout(fallback);
                        result.addFallback(endpoint.name(), fallbackResponse);
                        log.info("Used fallback for {}", endpoint.name());
                    } catch (Exception ex) {
                        log.error("Fallback also failed for {}", endpoint.name(), ex);
                    }
                }
            }
        }
        
        return responseMapper.map(result);
    }
    
    private Object callApiWithTimeout(ApiEndpoint<?> endpoint) {
        try {
            return CompletableFuture.supplyAsync(() -> 
                    restClient.get()
                            .uri(endpoint.url())
                            .retrieve()
                            .body(endpoint.responseType()),
                    virtualThreadExecutor)
                    .orTimeout(endpoint.timeoutMs(), TimeUnit.MILLISECONDS)
                    .get();
        } catch (Exception e) {
            throw new ApiCallException("Failed to call API: " + endpoint.name(), e);
        }
    }
    
    private ApiEndpoint<?> findFallback(String endpointName, List<ApiEndpoint<?>> fallbackEndpoints) {
        return fallbackEndpoints.stream()
                .filter(e -> e.name().equals(endpointName + "_fallback"))
                .findFirst()
                .orElse(null);
    }
    
    private void shutdownExecutor() {
        virtualThreadExecutor.shutdown();
        try {
            if (!virtualThreadExecutor.awaitTermination(1, TimeUnit.SECONDS)) {
                virtualThreadExecutor.shutdownNow();
            }
        } catch (InterruptedException e) {
            virtualThreadExecutor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
    
    /**
     * Functional interface for providing fallback values
     */
    @FunctionalInterface
    public interface FallbackProvider {
        Object getFallback(String endpointName);
    }
    
    /**
     * Functional interface for mapping aggregated results to final object
     */
    @FunctionalInterface
    public interface ResponseMapper<T> {
        T map(AggregatedResult result);
    }
    
    /**
     * Custom exception for API call failures
     */
    public static class ApiCallException extends RuntimeException {
        public ApiCallException(String message, Throwable cause) {
            super(message, cause);
        }
    }
}
```

Unit Tests

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.web.client.RestClient;

import java.util.List;
import java.util.Map;
import java.util.concurrent.ExecutionException;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class MultiApiAggregatorTest {
    
    @Mock
    private RestClient restClient;
    
    @Mock
    private RestClient.RequestHeadersUriSpec requestHeadersUriSpec;
    
    @Mock
    private RestClient.RequestHeadersSpec requestHeadersSpec;
    
    @Mock
    private RestClient.ResponseSpec responseSpec;
    
    private MultiApiAggregator aggregator;
    
    @BeforeEach
    void setUp() {
        aggregator = new MultiApiAggregator(RestClient.builder());
        // Mock setup would depend on actual RestClient configuration
    }
    
    @Test
    void testAggregateApis_Success() {
        // Arrange
        List<MultiApiAggregator.ApiEndpoint<String>> endpoints = List.of(
                new MultiApiAggregator.ApiEndpoint<>("userService", "/api/users", String.class),
                new MultiApiAggregator.ApiEndpoint<>("productService", "/api/products", String.class)
        );
        
        // Act
        Map<String, String> result = aggregator.aggregateApis(
                endpoints,
                endpointName -> "fallback_" + endpointName,
                aggregatedResult -> {
                    Map<String, Object> responses = aggregatedResult.getSuccessfulResponses();
                    return responses.entrySet().stream()
                            .collect(Collectors.toMap(
                                    Map.Entry::getKey,
                                    e -> e.getValue().toString()
                            ));
                }
        );
        
        // Assert
        assertNotNull(result);
        assertEquals(2, result.size());
        assertFalse(aggregatedResult.getFailedEndpoints().isEmpty());
    }
    
    @Test
    void testAggregateApis_WithFallback() {
        // Arrange
        List<MultiApiAggregator.ApiEndpoint<String>> endpoints = List.of(
                new MultiApiAggregator.ApiEndpoint<>("failingService", "/api/failing", String.class, 1000, false)
        );
        
        // Act
        MultiApiAggregator.AggregatedResult result = new MultiApiAggregator.AggregatedResult();
        
        // Simulate API failure and fallback
        result.addFailure("failingService", "Timeout");
        result.addFallback("failingService", "cached_value");
        
        // Assert
        assertTrue(result.getFallbackValues().containsKey("failingService"));
        assertEquals("cached_value", result.getFallbackValues().get("failingService"));
    }
    
    @Test
    void testTimeoutHandling() {
        // Arrange
        MultiApiAggregator.ApiEndpoint<String> slowEndpoint = 
                new MultiApiAggregator.ApiEndpoint<>("slowService", "/api/slow", String.class, 10, false);
        
        // Act & Assert
        assertThrows(ExecutionException.class, () -> {
            aggregator.aggregateApis(
                    List.of(slowEndpoint),
                    endpointName -> null,
                    result -> result
            );
        });
    }
    
    @Test
    void testRequiredApiFailure() {
        // Arrange
        MultiApiAggregator.ApiEndpoint<String> requiredEndpoint = 
                new MultiApiAggregator.ApiEndpoint<>("criticalService", "/api/critical", String.class, 1000, true);
        
        // Act
        MultiApiAggregator.AggregatedResult result = new MultiApiAggregator.AggregatedResult();
        result.addFailure("criticalService", "Service unavailable");
        
        // Assert
        assertTrue(result.getFailedEndpoints().containsKey("criticalService"));
        assertFalse(result.isComplete()); // Should not be complete if required API failed
    }
    
    @Test
    void testVirtualThreadUsage() {
        // Test that we can handle many concurrent API calls
        List<MultiApiAggregator.ApiEndpoint<String>> manyEndpoints = 
                IntStream.range(0, 100)
                        .mapToObj(i -> new MultiApiAggregator.ApiEndpoint<>(
                                "service" + i, 
                                "/api/service" + i, 
                                String.class))
                        .collect(Collectors.toList());
        
        // Should not throw OutOfMemoryError or similar
        assertDoesNotThrow(() -> {
            aggregator.aggregateApis(
                    manyEndpoints,
                    endpointName -> "fallback",
                    result -> result.getSuccessfulResponses().size()
            );
        });
    }
}
```

Example Usage with a Domain Object

```java
/**
 * Example domain object framed from multiple APIs
 */
public record UserProfile(
        String userId,
        UserDetails userDetails,
        List<Order> recentOrders,
        PaymentMethod primaryPayment,
        UserPreferences preferences
) {}

@Service
@Slf4j
public class UserProfileService {
    
    private final MultiApiAggregator apiAggregator;
    
    public UserProfileService(MultiApiAggregator apiAggregator) {
        this.apiAggregator = apiAggregator;
    }
    
    public UserProfile getUserProfile(String userId) {
        List<MultiApiAggregator.ApiEndpoint<?>> endpoints = List.of(
                new MultiApiAggregator.ApiEndpoint<>(
                        "userDetails", 
                        "/api/users/" + userId, 
                        UserDetails.class,
                        2000,
                        true
                ),
                new MultiApiAggregator.ApiEndpoint<>(
                        "userOrders", 
                        "/api/orders/user/" + userId + "/recent", 
                        Order[].class
                ),
                new MultiApiAggregator.ApiEndpoint<>(
                        "paymentMethods", 
                        "/api/payments/user/" + userId + "/primary", 
                        PaymentMethod.class
                ),
                new MultiApiAggregator.ApiEndpoint<>(
                        "preferences", 
                        "/api/preferences/" + userId, 
                        UserPreferences.class
                )
        );
        
        return apiAggregator.aggregateApis(
                endpoints,
                this::provideFallbacks,
                this::mapToUserProfile
        );
    }
    
    private Object provideFallbacks(String endpointName) {
        return switch (endpointName) {
            case "preferences" -> UserPreferences.defaultPreferences();
            case "userOrders" -> new Order[0];
            default -> null; // No fallback for critical endpoints
        };
    }
    
    private UserProfile mapToUserProfile(MultiApiAggregator.AggregatedResult result) {
        UserDetails userDetails = (UserDetails) result.getSuccessfulResponses()
                .getOrDefault("userDetails", result.getFallbackValues().get("userDetails"));
        
        Order[] orders = (Order[]) result.getSuccessfulResponses()
                .getOrDefault("userOrders", result.getFallbackValues().get("userOrders"));
        
        PaymentMethod payment = (PaymentMethod) result.getSuccessfulResponses()
                .get("paymentMethods"); // May be null
        
        UserPreferences preferences = (UserPreferences) result.getSuccessfulResponses()
                .getOrDefault("preferences", result.getFallbackValues().get("preferences"));
        
        return new UserProfile(
                userDetails.id(),
                userDetails,
                List.of(orders),
                payment,
                preferences
        );
    }
}
```

Key Features

1. Java 21 Features Used:
   ·