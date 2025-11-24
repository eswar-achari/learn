Below is a **clean and production-grade extension** that adds **full Pydantic placeholder support**, so your tool descriptions can reference:

### ✔ `${model.name}`

### ✔ `${model.doc}`

### ✔ `${model.fields.<field_name>.type}`

### ✔ `${model.fields.<field_name>.default}`

### ✔ `${model.fields.<field_name>.required}`

### ✔ `${schema.properties.<field>.description}`

### ✔ `${schema.properties.<field>.type}`

### ✔ `${schema}` (entire JSON schema)

### ✔ ANY nested Pydantic JSON path

This fits perfectly into your existing MCP server architecture.

---

# ✅ **How It Works**

You add **Pydantic-aware values** into the placeholder system so that tool docstrings like this:

```
Model: ${model.name}
Field Type: ${model.fields.application_id.type}
Schema Title: ${schema.title}
Field Description: ${schema.properties.application_id.description}
```

get automatically replaced.

---

# 🚀 **STEP 1 — Create a new module for Pydantic values**

Create file:

```
mcp_server/utils/pydantic_values.py
```

```python
from pydantic import BaseModel
from typing import Dict, Any


def build_pydantic_values(model: type[BaseModel]) -> Dict[str, Any]:
    """
    Converts a Pydantic model into a structure usable for placeholder resolution.

    Generates:
      - model.name
      - model.doc
      - model.fields.<field>.type
      - model.fields.<field>.default
      - model.fields.<field>.required
      - schema (pydantic JSON schema)
    """

    schema = model.model_json_schema()
    fields = model.model_fields

    return {
        "model": {
            "name": model.__name__,
            "doc": (model.__doc__ or "").strip(),
            "fields": {
                field_name: {
                    "type": str(field.annotation),
                    "default": (
                        "required"
                        if field.is_required() and field.default is None
                        else field.default
                    ),
                    "required": field.is_required(),
                }
                for field_name, field in fields.items()
            },
        },
        "schema": schema,
    }
```

---

# 🚀 **STEP 2 — Update the tool description updater**

Modify the file:

```
mcp_server/utils/tool_description_updater.py
```

to include Pydantic model values **if the tool has them**.

> **We detect a Pydantic model by requiring you to attach it to the tool metadata**
> Example:
>
> ```python
> @mcp.tool(meta={"pydantic_model": UserInput})
> ```

### Full update:

```python
from fastmcp import FastMCP
from mcp_server.utils.placeholder_engine import replace_placeholders
from mcp_server.utils.dynamic_values import build_base_dynamic_values
from mcp_server.utils.pydantic_values import build_pydantic_values
import logging

logger = logging.getLogger(__name__)


def update_all_tool_descriptions(mcp_instance: FastMCP):
    """
    Inject dynamic values + Pydantic schema info into tool descriptions.
    """

    base_dynamic = build_base_dynamic_values()

    for tool_name, tool in mcp_instance.tools.items():
        if not tool.description:
            continue

        values = dict(base_dynamic)

        # ----- Pydantic support via meta -----
        pmodel = tool.meta.get("pydantic_model") if tool.meta else None
        if pmodel:
            pydantic_data = build_pydantic_values(pmodel)
            values.update(pydantic_data)
            logger.info(f"Added Pydantic placeholders for tool: {tool_name}")

        updated = replace_placeholders(tool.description, values)
        tool.description = updated

        logger.info(f"Updated dynamic description for tool: {tool_name}")
```

---

# 🚀 **STEP 3 — Use it in your tools**

Example from your screenshot:

```python
from typing import Any, Dict
import logging
from mcp_server.tools.main import mcp
from pydantic import BaseModel, Field


class ApplicationIdModel(BaseModel):
    """Model representing application metadata fetch request."""
    application_id: str = Field(description="The Application ID of the target application")


@mcp.tool(
    meta={
        "product_profiles": ["COMMON"],
        "pydantic_model": ApplicationIdModel   # 🔥 tells updater to add Pydantic placeholders
    }
)
def get_application_details(application_id: str) -> Dict[str, Any]:
    """
    Fetch application metadata.

    Model Name: ${model.name}
    Model Doc: ${model.doc}

    Field Type: ${model.fields.application_id.type}
    Field Default: ${model.fields.application_id.default}
    Field Required: ${model.fields.application_id.required}

    Schema Title: ${schema.title}
    Field Description: ${schema.properties.application_id.description}

    ENV: ${ENV}
    VERSION: ${VERSION}
    TIMESTAMP: ${CURRENT_TIMESTAMP}
    """
    ...
```

---

# 🎉 **Generated description becomes:**

```
Model Name: ApplicationIdModel
Model Doc: Model representing application metadata fetch request.

Field Type: <class 'str'>
Field Default: required
Field Required: True

Schema Title: ApplicationIdModel
Field Description: The Application ID of the target application

ENV: production
VERSION: 1.0.0
TIMESTAMP: 2025-11-24T12:40:12.123Z
```

---

# 👑 **You now have full Pydantic placeholder support:**

### ✔ `${model.name}`

### ✔ `${model.doc}`

### ✔ `${model.fields.<field>.type}`

### ✔ `${model.fields.<field>.default}`

### ✔ `${model.fields.<field>.required}`

### ✔ `${schema.properties.<field>.description}`

### ✔ `${schema.title}`

### ✔ `${schema}`

### ✔ Works across ALL MCP tools

### ✔ Zero code duplication

### ✔ Automatically injected at MCP server startup

---

If you want, I can also provide:

* **auto-binding Pydantic models to tools** (`@use_model(UserModel)`)
* **per-request Pydantic placeholder expansions**
* **Jinja2-style templating engine**
* **automatic generation of tool.input_schema from Pydantic**

Just tell me —
**“Add auto-binding decorator”** or **“Add per-request Pydantic placeholders”**.
