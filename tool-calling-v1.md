Got it.
You want the final solution **in the same structure/style** as the function shown in your screenshot:

```
def fetch_application_details(...)
    ...
    db.<collection>.find_one(...)
    ...
    return {"data": dict(result)}
```

So I will rewrite the **dynamic CVE fetcher** using **your DB access pattern**, with logging, error structure, and the same Mongo access style (`db.<collection>.find_one()`).

---

# ✅ **Final Production-Ready Code (Using Your DB Format)**

```python
from typing import Dict, Any
import logging

logger = logging.getLogger(__name__)

def fetch_cve_details(payload: Dict[str, Any], db) -> Dict[str, Any]:
    """
    Fetch CVE details + CVE ratings based on dynamic payload fields.
    """

    if not payload:
        raise ValueError("Payload must not be empty")

    logger.info(f"Received payload: {payload}")

    # -----------------------------
    # Mapping of input → collection + db_field
    # -----------------------------
    dispatch_map = {
        "cveId": [
            ("cve_details_repo", "cve_no"),
            ("cve_ratings_repo", "cve_no")
        ],
        # add more rules as needed
    }

    # -----------------------------
    # Build dynamic query plan
    # -----------------------------
    query_plan = {}  # { "cve_details_repo": {"cve_no": "..."} }

    for key, value in payload.items():
        if value in ("", None):
            continue

        if key in dispatch_map:
            for collection, field in dispatch_map[key]:
                query_plan.setdefault(collection, {})[field] = value

    logger.info(f"Generated dynamic query plan: {query_plan}")

    if not query_plan:
        logger.warning("No valid fields found to query")
        return {"error": "No queryable fields found in payload"}

    # -----------------------------
    # Execute queries
    # -----------------------------
    results = {}

    for repo_name, criteria in query_plan.items():
        repo = getattr(db, repo_name, None)

        if repo is None:
            logger.error(f"Invalid repository: {repo_name}")
            continue

        logger.info(f"Fetching from {repo_name} with filter {criteria}")

        try:
            document = repo.find_one(criteria)
        except Exception as ex:
            logger.exception(f"Error querying {repo_name}: {ex}")
            results[repo_name] = {"error": str(ex)}
            continue

        if not document:
            logger.warning(f"No document found in {repo_name} for {criteria}")
            results[repo_name] = None
        else:
            logger.info(f"Successfully fetched from {repo_name}")
            results[repo_name] = dict(document)

    return {"data": results}
```

---

# 🧪 **Usage Example (Exactly Like Your Code Style)**

```python
payload = {
    "cveId": "CVE-2021-1234",
    "severity": "",
    "publishedDate": "",
}

response = fetch_cve_details(payload, db)
print(response)
```

---

# ✔ Output Example

```json
{
  "data": {
    "cve_details_repo": {
      "cve_no": "CVE-2021-1234",
      "description": "...",
      ...
    },
    "cve_ratings_repo": {
      "cve_no": "CVE-2021-1234",
      "official_rating": "Priority 4",
      ...
    }
  }
}
```

---

# ⭐ What This Code Matches

✔ Uses your logging style
✔ Uses `db.<collection>.find_one()`
✔ Returns `{ "data": ... }` exactly like your sample
✔ Modular and extendable for any number of collections
✔ Automatically builds queries from payload

---

If you want, I can also:

✅ Add support for `find_many`
✅ Add aggregation pipelines
✅ Add caching
✅ Auto-detect collections based on Mongo indexes

Just tell me!
