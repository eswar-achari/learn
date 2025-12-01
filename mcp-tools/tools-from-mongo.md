Below is a **production-ready Python implementation** that dynamically **creates MCP tools from a MongoDB collection** at application startup, following the **MCP Tool Transformation pattern**.

This solution uses:

* **Motor** (async MongoDB client)
* **FastMCP** (modern MCP server)
* **Environment variables via `pydantic-settings`**
* **Clean architecture & conventions**
* **Dynamic tool registration**
* **Automatic generation of a tool per DB entry**

---

# ✅ **Final Architecture Overview**

```
app/
 ├── config.py          # ENV settings
 ├── db.py              # MongoDB connection
 ├── models.py          # Pydantic model for DB documents
 ├── tool_loader.py     # Dynamic MCP tool creation
 ├── server.py          # MCP server startup
 └── main.py            # Entry point
```

---

# ✅ **1. config.py — ENV Handling**

```python
from pydantic_settings import BaseSettings
from pydantic import Field

class Settings(BaseSettings):
    mongo_uri: str = Field(..., env="MONGO_URI")
    mongo_db: str = Field(default="mcp")
    mongo_collection: str = Field(default="profiles")

    class Config:
        env_file = ".env"

settings = Settings()
```

---

# ✅ **2. db.py — MongoDB Client**

```python
from motor.motor_asyncio import AsyncIOMotorClient
from app.config import settings

client = AsyncIOMotorClient(settings.mongo_uri)
db = client[settings.mongo_db]

def get_collection():
    return db[settings.mongo_collection]
```

---

# ✅ **3. models.py — Pydantic Document Model**

```python
from pydantic import BaseModel
from typing import Optional, Dict, Any

class ProfileModel(BaseModel):
    Profile: str
    Description: Optional[str] = None
    name: str
    Meta: Optional[Dict[str, Any]] = None
    isActive: bool
```

---

# ✅ **4. tool_loader.py — Dynamic MCP Tool Creation**

This file generates tools **dynamically** from Mongo records.

```python
from fastmcp import MCPTool
from app.db import get_collection
from app.models import ProfileModel


async def load_dynamic_tools(server):
    """Load tools dynamically from MongoDB at startup."""

    collection = get_collection()

    async for document in collection.find({"isActive": True}):
        model = ProfileModel(**document)

        tool_name = f"profile_{model.name.replace(' ', '_').lower()}"

        # Create dynamic tool
        @server.tool(name=tool_name, description=model.Description or "Dynamic Profile Tool")
        async def generated_tool(input_data: dict, model=model):
            """
            Generic tool generated per profile.
            """
            return {
                "profile": model.Profile,
                "meta": model.Meta,
                "input_received": input_data
            }
```

✔ Every DB document becomes a **new MCP tool** like:

```
profile_admin_users
profile_customer_lookup
profile_alert_dispatch
```

---

# ✅ **5. server.py — MCP Server Setup**

```python
from fastmcp import FastMCP
from app.tool_loader import load_dynamic_tools

server = FastMCP()

@server.lifecycle.startup
async def on_startup():
    await load_dynamic_tools(server)
```

---

# ✅ **6. main.py — Application Entry Point**

```python
import uvicorn
from app.server import server

if __name__ == "__main__":
    uvicorn.run(server.app, host="0.0.0.0", port=8000)
```

---

# ⭐ Result

When your application starts:

* Connects to MongoDB
* Reads all records with `isActive: true`
* Creates a **separate MCP tool per document**
* Tools automatically appear under the MCP server

No manual coding per tool — fully data-driven!

---

# 📌 Example Mongo Document

```json
{
  "Profile": "Customer Lookup",
  "Description": "Search customer based on provided filters",
  "name": "customer_lookup",
  "Meta": { "version": "1.0" },
  "isActive": true
}
```

Creates a tool: **profile_customer_lookup**

---

# Do you want me to also generate:

✅ Unit tests
✅ Dockerfile + Kubernetes deployment
✅ Mongo seed scripts
✅ Support for tool input/output schemas (Pydantic models)

Just tell me!
