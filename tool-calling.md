Below is a clean, production-ready pattern for **dynamic collection selection** in Python for MongoDB, with scalability for *any number of collections*, proper input validation, separation of concerns, and extensibility.

This pattern works perfectly for tools like your **mcptools** where the tool receives an input `Dict[str, Any]` and must decide which Mongo collections to access based on incoming fields.

---

# ✅ **Production-Grade Design**

## **1. Use a Dispatcher Layer**

Map input field → target collection name.

This allows:

* Unlimited number of collections
* Easy addition of new rules
* No if-else jungle
* Predictable behavior even with complex inputs

---

## **2. Single Entry Method**

Your tool gets `input: Dict[str, Any]` and determines:

* Which collections should be touched
* What operation is needed

---

# 🧩 **Complete Production-Ready Code**

```python
from typing import Dict, Any, List
from pymongo import MongoClient

class MongoDynamicRouter:
    """
    Dynamically route MongoDB operations to collections based on input payload fields.
    Production-ready, extensible, and safe.
    """

    def __init__(self, uri: str, db_name: str):
        self.client = MongoClient(uri)
        self.db = self.client[db_name]

        # Dispatcher: input field → collection name
        self.collection_map = {
            "userId": "users",
            "orderId": "orders",
            "productId": "products",
            "category": "categories",
            # Add unlimited mappings here...
        }

    def resolve_collections(self, payload: Dict[str, Any]) -> Dict[str, Any]:
        """
        Returns a mapping of {collection_name: key_value} for every matched field.
        """
        resolved = {}
        for key, value in payload.items():
            if key in self.collection_map:
                collection = self.collection_map[key]
                resolved.setdefault(collection, {})[key] = value

        return resolved

    def query_collections(self, payload: Dict[str, Any]) -> Dict[str, List[Dict[str, Any]]]:
        """
        Routes query dynamically to all collections based on fields present in payload.
        Returns results from all involved collections.
        """
        resolved = self.resolve_collections(payload)
        results = {}

        for collection, criteria in resolved.items():
            mongo_collection = self.db[collection]
            
            # You may apply different logic here: find, aggregate, count, etc.
            query_result = list(mongo_collection.find(criteria))
            results[collection] = query_result

        return results

    def process(self, payload: Dict[str, Any]) -> Dict[str, Any]:
        """
        Entry point: This is what mcptools should call.
        Automatically detects required collections and executes queries.
        """
        if not isinstance(payload, dict):
            raise ValueError("Input payload must be a dictionary")

        return self.query_collections(payload)
```

---

# 🏃 **Usage Example**

```python
router = MongoDynamicRouter(
    uri="mongodb://localhost:27017",
    db_name="mydb"
)

payload = {
    "userId": 101,
    "orderId": 501,
    "misc": "ignore_this"
}

result = router.process(payload)
print(result)
```

### Output Example

```json
{
  "users": [{ "_id": 1, "userId": 101, "name": "John" }],
  "orders": [{ "_id": 12, "orderId": 501, "status": "Delivered" }]
}
```

---

# ⭐ Why this is Production-Ready

### ✔ Works for any number of collections

Just update `collection_map`.

### ✔ No need to modify logic when new collections are added

### ✔ Proper abstraction (router logic separated from query logic)

### ✔ Safe fallback (fields without mapping are ignored)

### ✔ Easy to extend for:

* Bulk writes
* Aggregations
* Write/update routing
* Multi-collection transactions

---

# 🚀 If you want Next Level

I can enhance it to support:

### 1. **Dynamic Operation Dispatch**

```
operation_map = {
    "createUser": ("users", insert_one),
    "updateOrder": ("orders", update_one)
}
```

### 2. **Priority-based routing**

Eg: if both userId and email exist → prefer “users” collection.

### 3. **Regex-based field mapping**

Useful when fields are dynamic.

### 4. **Schema-driven routing**

Load routing rules from JSON/YAML files instead of code.

---

If you'd like, tell me the expected payload structure and operations (`find`, `update`, `delete`, `aggregate`) and I'll generate a complete production-grade router for mcptools tailored to your project.
