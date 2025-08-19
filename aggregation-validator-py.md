Got it 👍 — you want a **Python utility class** that can help you **validate MongoDB aggregation pipelines** before you run them (catch common mistakes, ensure field consistency, maybe even dry-run the pipeline).

Here’s a **production-style implementation**:

---

# 🐍 `mongo_aggregation_validator.py`

```python
import logging
from pymongo import MongoClient
from pymongo.errors import OperationFailure, PyMongoError


class AggregationValidationError(Exception):
    """Raised when aggregation pipeline validation fails."""


class MongoAggregationValidator:
    """
    Utility class to validate MongoDB aggregation pipelines.
    Provides both syntax checks (dry-run with explain) and schema checks.
    """

    def __init__(self, uri: str, database: str, collection: str, log_level=logging.INFO):
        self.client = MongoClient(uri)
        self.db = self.client[database]
        self.collection = self.db[collection]

        logging.basicConfig(level=log_level, format="%(asctime)s [%(levelname)s] %(message)s")
        self.logger = logging.getLogger(__name__)

    def validate_pipeline(self, pipeline: list, sample: bool = False) -> dict:
        """
        Validate an aggregation pipeline by running explain().
        :param pipeline: The aggregation pipeline (list of stages).
        :param sample: If True, runs pipeline with limit(1) to ensure execution works.
        :return: Explain result dict
        """
        self.logger.info("Validating pipeline with %d stages...", len(pipeline))

        if not isinstance(pipeline, list):
            raise AggregationValidationError("Pipeline must be a list of stages")

        try:
            # Check syntax & execution plan
            explain_result = self.collection.aggregate(pipeline, explain=True)
            self.logger.debug("Explain result: %s", explain_result)

            if sample:
                self.logger.info("Executing sample run for validation...")
                _ = list(self.collection.aggregate(pipeline + [{"$limit": 1}]))
                self.logger.info("Sample run executed successfully")

            return explain_result
        except OperationFailure as e:
            self.logger.error("MongoDB rejected pipeline: %s", e.details)
            raise AggregationValidationError(f"Pipeline validation failed: {e.details}")
        except PyMongoError as e:
            self.logger.error("Unexpected PyMongo error: %s", str(e))
            raise AggregationValidationError(f"Pipeline validation failed: {str(e)}")

    def check_required_fields(self, pipeline: list, required_fields: list) -> None:
        """
        Ensures the pipeline projects all required fields in the final stage.
        :param pipeline: Aggregation pipeline
        :param required_fields: List of required field names
        """
        if not pipeline:
            raise AggregationValidationError("Pipeline is empty")

        last_stage = pipeline[-1]
        if "$project" not in last_stage:
            self.logger.warning("Last stage is not a $project, skipping field validation")
            return

        projected_fields = set(last_stage["$project"].keys())
        missing = [f for f in required_fields if f not in projected_fields]
        if missing:
            raise AggregationValidationError(f"Missing required fields in projection: {missing}")

        self.logger.info("All required fields present in projection ✅")


# ------------------ Example usage ------------------
if __name__ == "__main__":
    validator = MongoAggregationValidator(
        uri="mongodb://localhost:27017",
        database="test_db",
        collection="orders"
    )

    pipeline = [
        {"$match": {"status": "A"}},
        {"$group": {"_id": "$cust_id", "total": {"$sum": "$amount"}}},
        {"$project": {"cust_id": "$_id", "total": 1, "_id": 0}}
    ]

    try:
        result = validator.validate_pipeline(pipeline, sample=True)
        validator.check_required_fields(pipeline, ["cust_id", "total"])
        print("Pipeline is valid ✅")
    except AggregationValidationError as e:
        print("❌ Validation error:", e)
```

---

# ✨ Features

* **`validate_pipeline()`**

  * Uses MongoDB’s `explain` to check syntax/validity.
  * Optionally runs with `$limit:1` to ensure it actually executes.

* **`check_required_fields()`**

  * Ensures pipeline projects required fields in the final `$project`.

* **Custom exception** `AggregationValidationError` for clarity.

* **Logging** integrated (INFO level by default, DEBUG for explain plan).

---

⚡ This way, before pushing new pipelines to prod, you can **validate them safely** in dev/test, catch missing `$project` fields, and ensure Mongo will accept them.

---

👉 Do you want me to also extend this with **unit tests (pytest)** so you can plug into CI/CD for automated validation?
