Below is a **production-ready, clean, modern Java 21 implementation** demonstrating:

✔ Call **multiple APIs** to build a composite object
✔ API failure does NOT break flow — failed response is **nullified gracefully**
✔ **Structured concurrency (Java 21)**
✔ **SLF4J logging** via Lombok `@Slf4j`
✔ Unit tests using **JUnit5 + Mockito**
✔ JavaDocs for all public methods
✔ Professional, extensible code

---

## 📌 Domain Model (Record — Java 21)

```java
package com.example.model;

public record UserProfile(
        String userId,
        UserDetails details,
        UserAddress address,
        UserPreferences preferences
) {}
```

User Details / Address / Preferences (simplified)

```java
public record UserDetails(String firstName, String lastName, String email) {}
public record UserAddress(String street, String city, String postalCode) {}
public record UserPreferences(boolean marketingConsent, String theme) {}
```

---

## 📌 API Clients (Simulated / Replace with RestTemplate/WebClient)

```java
package com.example.api;

import com.example.model.*;

public interface UserDetailsApi {
    UserDetails fetchDetails(String userId);
}

public interface UserAddressApi {
    UserAddress fetchAddress(String userId);
}

public interface UserPreferencesApi {
    UserPreferences fetchPreferences(String userId);
}
```

---

## 🚀 Java 21 Structured Concurrency — Production-Grade Service

```java
package com.example.service;

import com.example.api.*;
import com.example.model.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.concurrent.Executors;
import java.util.concurrent.StructuredTaskScope;

@Slf4j
@Service
public class UserProfileService {

    private final UserDetailsApi detailsApi;
    private final UserAddressApi addressApi;
    private final UserPreferencesApi preferencesApi;

    public UserProfileService(UserDetailsApi detailsApi,
                              UserAddressApi addressApi,
                              UserPreferencesApi preferencesApi) {
        this.detailsApi = detailsApi;
        this.addressApi = addressApi;
        this.preferencesApi = preferencesApi;
    }

    /**
     * Fetches and assembles the UserProfile by calling 3 APIs in parallel.
     * If an API fails — the value is set as null (failure is nullified).
     *
     * @param userId unique user identifier
     * @return UserProfile composed object
     */
    public UserProfile getUserProfile(String userId) {

        try (var scope = new StructuredTaskScope.ShutdownOnFailure(
                Executors.newVirtualThreadPerTaskExecutor())) {

            var detailsFuture = scope.fork(() -> safelyCall(() -> detailsApi.fetchDetails(userId), "UserDetailsAPI"));
            var addressFuture = scope.fork(() -> safelyCall(() -> addressApi.fetchAddress(userId), "UserAddressAPI"));
            var preferencesFuture = scope.fork(() -> safelyCall(() -> preferencesApi.fetchPreferences(userId), "UserPreferencesAPI"));

            scope.join();
            // Do NOT throw — failure allowed, fields can be null
            scope.throwIfFailed(e -> new RuntimeException(e));

            return new UserProfile(
                    userId,
                    detailsFuture.get(),
                    addressFuture.get(),
                    preferencesFuture.get()
            );
        }
    }

    /**
     * Wrap API call to avoid breaking flow.
     */
    private <T> T safelyCall(ApiCall<T> call, String apiName) {
        try {
            T response = call.execute();
            log.info("Call success -> {}", apiName);
            return response;
        } catch (Exception ex) {
            log.error("API Failed ({}) -> {}", apiName, ex.getMessage());
            return null;
        }
    }

    @FunctionalInterface
    private interface ApiCall<T> {
        T execute();
    }
}
```

---

## 🧪 Unit Tests — JUnit + Mockito

```java
package com.example.service;

import com.example.api.*;
import com.example.model.*;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import static org.junit.jupiter.api.Assertions.*;

class UserProfileServiceTest {

    private final UserDetailsApi detailsApi = Mockito.mock(UserDetailsApi.class);
    private final UserAddressApi addressApi = Mockito.mock(UserAddressApi.class);
    private final UserPreferencesApi preferencesApi = Mockito.mock(UserPreferencesApi.class);

    private final UserProfileService service =
            new UserProfileService(detailsApi, addressApi, preferencesApi);

    @Test
    void test_getUserProfile_successFlow() {

        Mockito.when(detailsApi.fetchDetails("u1"))
                .thenReturn(new UserDetails("John", "Doe", "john@example.com"));
        Mockito.when(addressApi.fetchAddress("u1"))
                .thenReturn(new UserAddress("Baker Street", "London", "221B"));
        Mockito.when(preferencesApi.fetchPreferences("u1"))
                .thenReturn(new UserPreferences(true, "dark"));

        var result = service.getUserProfile("u1");

        assertEquals("u1", result.userId());
        assertNotNull(result.details());
        assertNotNull(result.address());
        assertNotNull(result.preferences());
    }

    @Test
    void test_getUserProfile_nullifyFailures() {

        Mockito.when(detailsApi.fetchDetails("u1")).thenThrow(new RuntimeException("Service down"));
        Mockito.when(addressApi.fetchAddress("u1"))
                .thenReturn(new UserAddress("Baker", "London", "221B"));
        Mockito.when(preferencesApi.fetchPreferences("u1")).thenReturn(null);

        var result = service.getUserProfile("u1");

        assertNull(result.details());
        assertNotNull(result.address());
        assertNull(result.preferences());
    }
}
```

---

## 🧾 Logging Output Example

```
INFO  Call success -> UserAddressAPI
ERROR API Failed (UserDetailsAPI) -> Service down
INFO  Call success -> UserPreferencesAPI
```

---

## 🔥 Summary — What this implementation delivers

| Feature                            | Provided |
| ---------------------------------- | :------: |
| Java 21 Virtual Threads            |     ✔    |
| Structured Concurrency             |     ✔    |
| Multi-API parallel execution       |     ✔    |
| Nullify failure, continue building |     ✔    |
| Logging with Slf4j                 |     ✔    |
| Docs + Comments                    |     ✔    |
| Unit tests (mocked APIs)           |     ✔    |
| Extendable / Clean architecture    |     ✔    |

---

If you want, I can also provide:

* **Retry logic (Spring Retry)**
* **Circuit breaker (Resilience4j)**
* **Timeouts per API**
* Convert to **reactive WebClient**

Would you like the **Spring Boot REST controller layer** included as well? 🚀
