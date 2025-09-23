Got it 👍 — if **`gis_id` is unique per row**, then batching *per gis\_id* doesn’t really make sense (each batch would only contain one record).

So we have two clean upgrade paths:

---

### ✅ 1. SQL query (still sort by `gis_id`)

Sorting ensures you process in ascending order of the unique id:

```sql
SELECT ...
FROM daily_observations_scorecard
ORDER BY gis_id;
```

---

### ✅ 2. Java code (batch sequential records by gis\_id order)

Here we’ll frame **fixed-size batches** (say 500 or 1000 records) in gis\_id order, and dispatch batch-wise to Kafka.

```java
int batchSize = 500;  // configurable
List<VulnerabilityDetails> batch = new ArrayList<>();

try (Stream<VulnerabilityDetails> stream =
         hiveJdbcTemplate.queryForStream(query, argMap, mapper)) {

    Iterator<VulnerabilityDetails> iterator = stream.iterator();

    while (iterator.hasNext()) {
        VulnerabilityDetails v = iterator.next();
        batch.add(v);

        if (batch.size() >= batchSize) {
            dispatcher.dispatch(new ArrayList<>(batch)); // send batch
            tracker.increment(jobName, batch.size());

            log.info("JobName: {} --> Dispatched {} records so far",
                     jobName, tracker.getCount(jobName));

            batch.clear();

            // optional throttling
            try {
                Thread.sleep(Duration.ofMillis(50));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    // final flush if leftovers exist
    if (!batch.isEmpty()) {
        dispatcher.dispatch(batch);
        tracker.increment(jobName, batch.size());
        log.info("JobName: {} --> Final dispatch of {} records",
                 jobName, batch.size());
    }
}
```

---

### 🔑 Key Points

* **`ORDER BY gis_id`** ensures deterministic order in SQL.
* Java batches records in sequence (not by gis\_id groups, since it’s unique).
* Each `dispatcher.dispatch(batch)` call pushes a collection to Kafka.
* `tracker.increment(jobName, batch.size())` keeps count by batch instead of per record.

---

👉 Do you want me to make the **batch size configurable from application properties** (e.g., `kafka.batch.size=500` in YAML/properties), so you can tune it without code change?
