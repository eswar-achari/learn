Here’s a **step-by-step guide** to check **Hive connectivity via a Spring Boot application** using **`@Autowired` JdbcTemplate**, with the Hive **datasource configured via `application.properties`**.

This approach is production-friendly and makes the connection reusable for ingestion pipelines.

---

## **1. Project Setup**

Create a **Spring Boot project** with Maven or Spring Initializr.

**Dependencies in `pom.xml`:**

```xml
<dependencies>
    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>

    <!-- Spring JDBC -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>

    <!-- Hive JDBC Driver -->
    <dependency>
        <groupId>org.apache.hive</groupId>
        <artifactId>hive-jdbc</artifactId>
        <version>3.1.3</version>
    </dependency>

    <!-- Optional: For logging SQL -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-logging</artifactId>
    </dependency>
</dependencies>
```

> Make sure the Hive JDBC version matches your Hive cluster version.

---

## **2. Configure Datasource in `application.properties`**

This is where you configure Hive connection details.

```properties
# Hive JDBC Connection
spring.datasource.url=jdbc:hive2://192.168.1.100:10000/default
spring.datasource.username=hive_user
spring.datasource.password=hive_password
spring.datasource.driver-class-name=org.apache.hive.jdbc.HiveDriver

# Optional Connection Pool Settings (HikariCP)
spring.datasource.hikari.maximum-pool-size=5
spring.datasource.hikari.minimum-idle=2
spring.datasource.hikari.idle-timeout=30000
spring.datasource.hikari.max-lifetime=600000

# Logging SQL Queries
logging.level.org.springframework.jdbc.core=DEBUG
```

---

## **3. Create a Configuration Class (Optional)**

Spring Boot auto-configures the `DataSource` based on `application.properties`.
If you want to **customize**, explicitly define it.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.core.JdbcTemplate;

import javax.sql.DataSource;

@Configuration
public class HiveConfig {

    // JdbcTemplate bean automatically uses the Hive datasource
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

---

## **4. Create a Repository for Hive Queries**

Here we use **`JdbcTemplate`** to execute Hive queries.

```java
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;
import java.util.List;
import java.util.Map;

@Repository
public class HiveRepository {

    private final JdbcTemplate jdbcTemplate;

    // @Autowired via constructor injection
    public HiveRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public List<Map<String, Object>> getDatabases() {
        String query = "SHOW DATABASES";
        return jdbcTemplate.queryForList(query);
    }

    public List<Map<String, Object>> getTableData(String tableName) {
        String query = "SELECT * FROM " + tableName + " LIMIT 10";
        return jdbcTemplate.queryForList(query);
    }
}
```

---

## **5. Create a Service Layer**

This is a business logic layer for ingestion pipelines.

```java
import org.springframework.stereotype.Service;
import java.util.List;
import java.util.Map;

@Service
public class HiveService {

    private final HiveRepository hiveRepository;

    public HiveService(HiveRepository hiveRepository) {
        this.hiveRepository = hiveRepository;
    }

    public List<Map<String, Object>> fetchDatabases() {
        return hiveRepository.getDatabases();
    }

    public List<Map<String, Object>> fetchSampleData(String tableName) {
        return hiveRepository.getTableData(tableName);
    }
}
```

---

## **6. Create a REST Controller to Test Hive Connection**

Expose endpoints to check Hive connectivity and run queries.

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;
import java.util.List;
import java.util.Map;

@RestController
public class HiveController {

    private final HiveService hiveService;

    public HiveController(HiveService hiveService) {
        this.hiveService = hiveService;
    }

    // Endpoint to test connection
    @GetMapping("/hive/databases")
    public List<Map<String, Object>> getDatabases() {
        return hiveService.fetchDatabases();
    }

    // Endpoint to query specific table
    @GetMapping("/hive/table/{tableName}")
    public List<Map<String, Object>> getTableData(@PathVariable String tableName) {
        return hiveService.fetchSampleData(tableName);
    }
}
```

---

## **7. Run and Verify**

### **Run the application**

```bash
mvn spring-boot:run
```

### **Test the endpoints**

* **Test database listing**

  ```bash
  curl http://localhost:8080/hive/databases
  ```

  **Output:**

  ```json
  [
    {"database_name":"default"},
    {"database_name":"analytics"}
  ]
  ```

* **Test table query**

  ```bash
  curl http://localhost:8080/hive/table/customers
  ```

  **Output:**

  ```json
  [
    {"customer_id":"C001","name":"Alice","email":"alice@example.com"},
    {"customer_id":"C002","name":"Bob","email":"bob@example.com"}
  ]
  ```

---

## **8. Logs for SQL Queries**

Spring will log SQL queries because we enabled:

```properties
logging.level.org.springframework.jdbc.core=DEBUG
```

**Sample Logs:**

```
2025-09-17 12:10:25 DEBUG org.springframework.jdbc.core.JdbcTemplate - Executing SQL query [SHOW DATABASES]
2025-09-17 12:10:26 DEBUG org.springframework.jdbc.core.JdbcTemplate - Executing SQL query [SELECT * FROM customers LIMIT 10]
```

---

## **9. Ingestion Ready Enhancements**

For production-ready ingestion pipelines:

1. **Batch Inserts / Bulk Processing**

   ```java
   jdbcTemplate.batchUpdate(
       "INSERT INTO my_table (id, name) VALUES (?, ?)",
       batchPreparedStatementSetter
   );
   ```

2. **Connection Pooling** via HikariCP (already configured above).

3. **Query Timeout**:

   ```properties
   spring.datasource.hikari.connection-timeout=30000
   ```

4. **Retries & Error Handling**:
   Wrap repository methods with `@Retryable` or custom exception handling.

---

## **10. Final Project Structure**

```
src/main/java
├── com.example.hive
│   ├── HiveApplication.java        # Main Spring Boot app
│   ├── config
│   │   └── HiveConfig.java         # DataSource/JdbcTemplate config
│   ├── controller
│   │   └── HiveController.java     # REST APIs
│   ├── repository
│   │   └── HiveRepository.java     # Hive JDBC queries
│   └── service
│       └── HiveService.java        # Business logic
└── resources
    └── application.properties      # Hive connection config
```

---

## **11. Summary**

| Step                          | Purpose                            |
| ----------------------------- | ---------------------------------- |
| **application.properties**    | Centralized Hive connection config |
| **JdbcTemplate + @Autowired** | Simplifies query execution         |
| **Service Layer**             | Encapsulates business logic        |
| **Controller**                | Exposes REST APIs for testing      |
| **Logging**                   | Tracks Hive queries & debugging    |
| **HikariCP Pooling**          | Optimizes connection handling      |

---

With this setup:

* The Hive connection is fully externalized via `application.properties`.
* `JdbcTemplate` is autowired, making ingestion jobs simple and efficient.
* Logging shows every query executed, which helps for audit and debugging.
* You can now **ingest Hive data into other systems**, like MongoDB or Kafka.
