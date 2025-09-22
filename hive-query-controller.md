Perfect 👍
You want a **REST controller** where:

* Client sends a query (Hive SQL).
* Backend executes that query on Hive.
* Results are framed as JSON and sent in the HTTP response.

Here’s a **Spring Boot** example using the **Hive JDBC driver**.

---

## 1. Maven Dependencies (`pom.xml`)

```xml
<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Hive JDBC -->
    <dependency>
        <groupId>org.apache.hive</groupId>
        <artifactId>hive-jdbc</artifactId>
        <version>3.1.3</version>
    </dependency>

    <!-- JSON Handling -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
</dependencies>
```

---

## 2. Application Properties (`application.yml`)

```yaml
spring:
  datasource:
    url: jdbc:hive2://localhost:10000/default
    username: hiveuser
    password: hivepass
    driver-class-name: org.apache.hive.jdbc.HiveDriver
```

---

## 3. Service to Query Hive

```java
import org.springframework.stereotype.Service;
import javax.sql.DataSource;
import java.sql.*;
import java.util.*;

@Service
public class HiveQueryService {

    private final DataSource dataSource;

    public HiveQueryService(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public List<Map<String, Object>> executeQuery(String query) throws SQLException {
        List<Map<String, Object>> result = new ArrayList<>();

        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery(query)) {

            ResultSetMetaData metaData = rs.getMetaData();
            int columnCount = metaData.getColumnCount();

            while (rs.next()) {
                Map<String, Object> row = new LinkedHashMap<>();
                for (int i = 1; i <= columnCount; i++) {
                    row.put(metaData.getColumnLabel(i), rs.getObject(i));
                }
                result.add(row);
            }
        }
        return result;
    }
}
```

---

## 4. REST Controller

```java
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.sql.SQLException;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/hive")
public class HiveQueryController {

    private final HiveQueryService hiveQueryService;

    public HiveQueryController(HiveQueryService hiveQueryService) {
        this.hiveQueryService = hiveQueryService;
    }

    @PostMapping("/query")
    public ResponseEntity<?> executeQuery(@RequestBody Map<String, String> request) {
        String query = request.get("query");

        if (query == null || query.trim().isEmpty()) {
            return ResponseEntity.badRequest().body("Query cannot be empty");
        }

        try {
            List<Map<String, Object>> results = hiveQueryService.executeQuery(query);
            return ResponseEntity.ok(results);
        } catch (SQLException e) {
            return ResponseEntity.internalServerError().body("Error executing Hive query: " + e.getMessage());
        }
    }
}
```

---

## 5. Example Request

```http
POST /api/hive/query
Content-Type: application/json

{
  "query": "SELECT region, month, SUM(amount) as total_sales FROM hive_sales GROUP BY region, month"
}
```

---

## 6. Example Response

```json
[
  { "region": "North", "month": "2025-08", "total_sales": 150000.75 },
  { "region": "South", "month": "2025-08", "total_sales": 120500.25 }
]
```

---

✅ This REST controller is generic – you can run **any Hive SQL** and get results as JSON.
⚠️ Security Note: For production, you should **whitelist queries** or use parameterized templates to prevent misuse.

---

Do you also want me to extend this controller to **compare Hive results with MongoDB (validation API)** in the same service?
