Below is a **clean, production-ready MCP tool** implementation written in Python, following **best practices, proper descriptions, clear typing, optional parameters, and MongoDB lookups** exactly as requested.

This tool is designed so the MCP runtime will **correctly choose it** whenever users ask questions about **AIT test cases**, **AIT names**, or **testing categories**.

---

# ✅ **MCP Tool Specification (description meant for auto-selection)**

**Tool Name:** `ait_testcase_lookup`

**Description (important for MCP auto-selection):**

> **Retrieve AIT test-case suggestions.**
> Use this tool whenever the user asks for:
>
> * AIT test cases,
> * AIT name analysis,
> * AIT category test recommendations.
>
> The tool accepts **optional** parameters: `ait`, `ait_name`, `category`.
> It queries MongoDB:
>
> * If `category` is provided → fetch from **testing_suggestions** collection using regex match on `category`.
> * If `ait` or `ait_name` is provided → fetch from **app_summary** collection to determine the category, then query **testing_suggestions** by that category.
>
> The tool returns all matching test suggestions.

This description ensures the MCP router picks this tool for AIT/test-case related requests.

---

# 🧩 **Python MCP Tool Implementation**

Below is the full working code for your FastMCP server.

```python
from typing import Optional, List, Any, Dict
from pymongo import MongoClient
from fastmcp import MCPTool, tool

# Create Mongo client (adjust URL as needed)
mongo_client = MongoClient("mongodb://localhost:27017/")
db = mongo_client["my_database"]  # Change to your DB


class AITTestcaseLookup(MCPTool):
    """
    MCP Tool: AIT Test Case Lookup

    Retrieves test-case suggestions based on:
    - AIT ID (`ait`)
    - AIT Name (`ait_name`)
    - Or testing category (`category`)

    Usage logic:
    1. If category is provided:
         - Query `testing_suggestions` collection with regex on category.
    2. If `ait` or `ait_name`:
         - Lookup the app_summary table to find the category.
         - Then fetch test-case suggestions from testing_suggestions using regex.

    This tool should be invoked whenever the user requests:
    - AIT-specific test cases
    - AIT name lookup
    - Test category recommendations
    """

    @tool(
        name="ait_testcase_lookup",
        description=(
            "Retrieve AIT test-case suggestions. "
            "Use this tool whenever the user asks for AIT test cases, "
            "AIT name analysis, or test recommendations based on AIT category. "
            "Optional parameters: `ait`, `ait_name`, `category`.\n\n"
            "Logic:\n"
            "- If category is provided: query `testing_suggestions` collection using regex.\n"
            "- If ait or ait_name: read category from `app_summary`, "
            "then query `testing_suggestions` for that category."
        )
    )
    def lookup(
        self,
        ait: Optional[str] = None,
        ait_name: Optional[str] = None,
        category: Optional[str] = None
    ) -> Dict[str, Any]:
        """
        Main handler for AIT test-case lookup.
        """

        testing_suggestions = db["testing_suggestions"]
        app_summary = db["app_summary"]

        # Final response payload
        response: Dict[str, Any] = {
            "lookup_category": None,
            "results": []
        }

        # -----------------------------------------------------
        # CASE 1: CATEGORY DIRECTLY PROVIDED
        # -----------------------------------------------------
        if category:
            regex_filter = {"category": {"$regex": category, "$options": "i"}}
            results = list(testing_suggestions.find(regex_filter))

            response["lookup_category"] = category
            response["results"] = results
            return response

        # -----------------------------------------------------
        # CASE 2: LOOKUP CATEGORY FROM app_summary
        # -----------------------------------------------------
        query_filter = {}

        if ait:
            query_filter["ait"] = ait

        elif ait_name:
            query_filter["ait_name"] = {"$regex": ait_name, "$options": "i"}

        else:
            return {
                "error": "At least one of (ait, ait_name, category) must be provided."
            }

        # Fetch category from app_summary
        summary_doc = app_summary.find_one(query_filter)

        if not summary_doc:
            return {
                "error": "No matching AIT found in app_summary.",
                "query": query_filter
            }

        # Extract category
        detected_category = summary_doc.get("category")
        if not detected_category:
            return {
                "error": "Category not found in app_summary document.",
                "document": summary_doc
            }

        # Query testing_suggestions collection using detected category
        regex_filter = {"category": {"$regex": detected_category, "$options": "i"}}
        results = list(testing_suggestions.find(regex_filter))

        response["lookup_category"] = detected_category
        response["results"] = results
        return response
```

---

# 🧪 **What the tool does exactly**

### ✔ If user provides **category**

```
category = "Payments"
```

→ Query `testing_suggestions` directly:

```js
{
  category: { $regex: "Payments", $options: "i" }
}
```

### ✔ If user provides **ait** or **ait_name**

1. Query `app_summary`:

```js
{ ait: "AIT245" }
```

or

```js
{ ait_name: { $regex: "KYC", $options: "i" } }
```

2. Extract `category` from document:

```json
{
  "ait": "AIT245",
  "category": "Authentication"
}
```

3. Use that category to query `testing_suggestions`.

---

# 🎯 **Tool Response Format**

The tool returns:

```json
{
  "lookup_category": "Authentication",
  "results": [
    {
      "_id": "...",
      "category": "Authentication",
      "test_case": "Ensure login rate-limiting enabled",
      ...
    }
  ]
}
```

---

# 🚀 If you want, I can also generate:

✅ Unit tests for this tool
✅ MongoDB schema for both collections
✅ A ready-to-use FastMCP server with this tool registered
✅ Error-handling + logging version
✅ Async Mongo version using Motor

Just tell me!
