Excellent 👍 — Let’s create a **robust Python script** that will:

✅ Export selected collections from a **source MongoDB** (say, DEV or STAGING)
✅ Import them into a **target MongoDB** (say, PROD)
✅ Handle multiple collections automatically
✅ Work cross-platform (Windows/macOS/Linux)

---

## 🚀 Python Script: `mongo_export_import.py`

Here’s a clean, production-safe version ⬇️

```python
import os
import subprocess
from datetime import datetime

# ========== CONFIGURATION ==========
# MongoDB connection details
SOURCE_URI = "mongodb://localhost:27017/dev_db"     # e.g., your staging/QA DB
TARGET_URI = "mongodb://prodhost:27017/prod_db"     # e.g., your PROD DB

# Collections to transfer
COLLECTIONS = [
    "users",
    "orders",
    "transactions",
    "products"
]

# Export directory
EXPORT_DIR = "./mongo_exports"
# ===================================


def ensure_export_dir():
    """Create export directory if not exists."""
    os.makedirs(EXPORT_DIR, exist_ok=True)
    print(f"✅ Export directory ready at: {EXPORT_DIR}")


def export_collections():
    """Export all collections from source MongoDB using mongodump."""
    print("📤 Exporting collections from source MongoDB...")
    for collection in COLLECTIONS:
        export_path = os.path.join(EXPORT_DIR, collection)
        os.makedirs(export_path, exist_ok=True)

        cmd = [
            "mongodump",
            "--uri", SOURCE_URI,
            "--collection", collection,
            "--out", export_path
        ]

        print(f"→ Exporting {collection} ...")
        subprocess.run(cmd, check=True)
        print(f"✅ Exported {collection} to {export_path}")


def import_collections():
    """Import exported collections into target MongoDB using mongorestore."""
    print("📥 Importing collections into target MongoDB...")
    for collection in COLLECTIONS:
        import_path = os.path.join(EXPORT_DIR, collection)

        cmd = [
            "mongorestore",
            "--uri", TARGET_URI,
            "--nsInclude", f"*.{collection}",
            "--dir", import_path,
            "--drop"  # Drops existing collection before restore
        ]

        print(f"→ Importing {collection} ...")
        subprocess.run(cmd, check=True)
        print(f"✅ Imported {collection} into target DB")


def main():
    start_time = datetime.now()
    print(f"🚀 Mongo Export/Import started at {start_time.strftime('%Y-%m-%d %H:%M:%S')}")

    ensure_export_dir()
    export_collections()
    import_collections()

    print(f"🎉 Completed successfully in {(datetime.now() - start_time).seconds} seconds!")


if __name__ == "__main__":
    main()
```

---

## 🧩 How it works

| Step | Description                                                         |
| ---- | ------------------------------------------------------------------- |
| 1️⃣  | The script creates an `./mongo_exports/` directory                  |
| 2️⃣  | Runs `mongodump` for each collection from your source DB            |
| 3️⃣  | Runs `mongorestore` to import each dump into target DB              |
| 4️⃣  | Uses `--drop` to ensure the target collections are replaced cleanly |
| 5️⃣  | Provides clear console progress logs                                |

---

## ⚙️ Requirements

### 🧰 Install MongoDB tools

Make sure `mongodump` and `mongorestore` are available:

```bash
brew install mongodb-database-tools      # macOS
# OR
sudo apt install mongodb-database-tools  # Ubuntu/Debian
```

### 🐍 Python dependencies

No external libraries required — uses only standard library + MongoDB CLI tools.

---

## 🧠 Example Usage

```bash
python3 mongo_export_import.py
```

Output:

```
🚀 Mongo Export/Import started at 2025-10-27 10:40:15
✅ Export directory ready at: ./mongo_exports
→ Exporting users ...
✅ Exported users to ./mongo_exports/users
→ Exporting orders ...
✅ Exported orders to ./mongo_exports/orders
📥 Importing collections into target MongoDB...
→ Importing users ...
✅ Imported users into target DB
🎉 Completed successfully in 12 seconds!
```

---

## 💡 Optional Enhancements

If you’d like, I can extend it with:

* `.env` config support (for secure DB URIs)
* Automatic **gzip compression** of dumps
* Email/Slack notification on completion
* Parallel export/import for faster execution
* Filtering by date or query (via `--query` flag)

---

Would you like me to **add gzip compression & environment-based configuration** (using `.env` file)?
That’s the best practice for migrating between environments like DEV → PROD securely.
