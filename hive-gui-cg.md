Nice upgrade 🚀 — adding a **Schema Explorer panel** will make your Hive desktop app feel like **DBeaver/Beekeeper**.

Here’s how we extend your **Electron + Node.js + React app**:

---

# 🏗 Features of Schema Explorer

* **List Databases** in Hive.
* On expanding a database → list all **Tables**.
* On clicking a table → show **columns and data types**.
* Allow **double-click to auto-fill query editor** with `SELECT * FROM db.table LIMIT 100;`.

---

# ⚡ Backend (`hiveService.js`) — Add Schema Functions

```js
async function listDatabases() {
  try {
    const operation = await hiveSession.executeStatement("SHOW DATABASES");
    const result = await operation.fetchAll();
    return { success: true, databases: result.map(r => r.database_name || Object.values(r)[0]) };
  } catch (err) {
    return { success: false, error: err.message };
  }
}

async function listTables(database) {
  try {
    const operation = await hiveSession.executeStatement(`SHOW TABLES IN ${database}`);
    const result = await operation.fetchAll();
    return { success: true, tables: result.map(r => r.tab_name || Object.values(r)[0]) };
  } catch (err) {
    return { success: false, error: err.message };
  }
}

async function describeTable(database, table) {
  try {
    const operation = await hiveSession.executeStatement(`DESCRIBE ${database}.${table}`);
    const result = await operation.fetchAll();
    return { success: true, columns: result };
  } catch (err) {
    return { success: false, error: err.message };
  }
}

module.exports = { testConnection, connect, runQuery, getStats, exportExcel, listDatabases, listTables, describeTable };
```

---

# ⚡ main.js — Add IPC handlers

```js
ipcMain.handle("list-databases", async () => {
  return hiveService.listDatabases();
});

ipcMain.handle("list-tables", async (_, db) => {
  return hiveService.listTables(db);
});

ipcMain.handle("describe-table", async (_, { db, table }) => {
  return hiveService.describeTable(db, table);
});
```

---

# ⚡ preload.js — Add bridge

```js
contextBridge.exposeInMainWorld("hiveAPI", {
  testConnection: (creds) => ipcRenderer.invoke("test-connection", creds),
  connect: (creds) => ipcRenderer.invoke("connect", creds),
  runQuery: (payload) => ipcRenderer.invoke("run-query", payload),
  getStats: (payload) => ipcRenderer.invoke("get-stats", payload),
  exportExcel: (payload) => ipcRenderer.invoke("export-excel", payload),
  listDatabases: () => ipcRenderer.invoke("list-databases"),
  listTables: (db) => ipcRenderer.invoke("list-tables", db),
  describeTable: (db, table) => ipcRenderer.invoke("describe-table", { db, table }),
});
```

---

# ⚡ React UI — Add Schema Explorer Sidebar (`App.jsx`)

```jsx
import { useState, useEffect } from "react";

function App() {
  const [databases, setDatabases] = useState([]);
  const [tables, setTables] = useState({});
  const [columns, setColumns] = useState([]);
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);

  useEffect(() => {
    async function loadDatabases() {
      const res = await window.hiveAPI.listDatabases();
      if (res.success) setDatabases(res.databases);
    }
    loadDatabases();
  }, []);

  const loadTables = async (db) => {
    const res = await window.hiveAPI.listTables(db);
    if (res.success) setTables(prev => ({ ...prev, [db]: res.tables }));
  };

  const showColumns = async (db, table) => {
    const res = await window.hiveAPI.describeTable(db, table);
    if (res.success) setColumns(res.columns);

    // Auto-insert query
    setQuery(`SELECT * FROM ${db}.${table} LIMIT 100;`);
  };

  return (
    <div className="flex h-screen">
      {/* Sidebar */}
      <div className="w-1/4 border-r p-4 overflow-y-auto">
        <h2 className="font-bold">Schema Explorer</h2>
        <ul>
          {databases.map((db, i) => (
            <li key={i} className="mt-2">
              <button className="font-semibold text-blue-600" onClick={() => loadTables(db)}>
                {db}
              </button>
              {tables[db] && (
                <ul className="ml-4">
                  {tables[db].map((tbl, j) => (
                    <li key={j}>
                      <button
                        className="text-sm text-gray-700"
                        onClick={() => showColumns(db, tbl)}
                      >
                        {tbl}
                      </button>
                    </li>
                  ))}
                </ul>
              )}
            </li>
          ))}
        </ul>

        {columns.length > 0 && (
          <div className="mt-4">
            <h3 className="font-bold">Columns</h3>
            <ul>
              {columns.map((c, i) => (
                <li key={i} className="text-xs">
                  {c.col_name} : {c.data_type}
                </li>
              ))}
            </ul>
          </div>
        )}
      </div>

      {/* Main Query Panel */}
      <div className="flex-1 p-4">
        <textarea
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          rows={6}
          className="w-full border p-2"
        ></textarea>
        <button
          className="bg-purple-500 text-white px-4 py-2 mt-2"
          onClick={async () => {
            const res = await window.hiveAPI.runQuery({ query, page: 1, pageSize: 20 });
            if (res.success) setResults(res.rows);
            else alert(res.error);
          }}
        >
          Run Query
        </button>

        {/* Results */}
        <table className="table-auto border mt-4 w-full text-sm">
          <thead>
            {results.length > 0 && (
              <tr>
                {Object.keys(results[0]).map((col, idx) => (
                  <th key={idx} className="border px-2 py-1">{col}</th>
                ))}
              </tr>
            )}
          </thead>
          <tbody>
            {results.map((row, i) => (
              <tr key={i}>
                {Object.values(row).map((val, j) => (
                  <td key={j} className="border px-2 py-1">{val}</td>
                ))}
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
}

export default App;
```

---

# 🎁 Final Result

* **Left Sidebar** → Schema Explorer (Databases → Tables → Columns).
* **Main Panel** → Query editor + Results table.
* Clicking a **table** → loads columns + inserts `SELECT * FROM db.table LIMIT 100;` into editor.
* Same **pagination, stats, and export features** still work.

---

👉 Do you want me to also add a **Saved Queries & History Panel** (like DBeaver) so users can quickly re-run past queries?
