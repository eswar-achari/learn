I'll create a single executable Hive DB query tool using **pkg** to package everything into one file. Here's the complete solution:

## Package.json
```json
{
  "name": "hive-query-tool-single",
  "version": "1.0.0",
  "description": "Single executable Hive DB Query Tool",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "build": "pkg app.js --target node18-win-x64,node18-linux-x64,node18-macos-x64 --output hive-query-tool",
    "build-all": "pkg app.js --targets node18-win-x64,node18-linux-x64,node18-macos-x64 --output hive-query-tool-${platform}"
  },
  "dependencies": {
    "express": "^4.18.2",
    "hive-driver": "^0.9.0",
    "xlsx": "^0.18.5",
    "cors": "^2.8.5"
  },
  "devDependencies": {
    "pkg": "^5.8.1"
  },
  "pkg": {
    "assets": [
      "views/**/*",
      "public/**/*"
    ]
  }
}
```

## Single File: app.js
```javascript
const express = require('express');
const cors = require('cors');
const { HiveClient, TCLIService, TCLIService_types } = require('hive-driver');
const XLSX = require('xlsx');
const path = require('path');
const fs = require('fs');

class HiveQueryTool {
    constructor() {
        this.app = express();
        this.port = process.env.PORT || 3000;
        this.clients = new Map();
        this.setupMiddleware();
        this.setupRoutes();
        this.setupStaticFiles();
    }

    setupMiddleware() {
        this.app.use(cors());
        this.app.use(express.json());
        this.app.use(express.urlencoded({ extended: true }));
    }

    setupStaticFiles() {
        // Serve embedded HTML, CSS, and JS
        this.app.get('/', (req, res) => {
            res.send(this.getHTML());
        });

        this.app.get('/css/style.css', (req, res) => {
            res.setHeader('Content-Type', 'text/css');
            res.send(this.getCSS());
        });

        this.app.get('/js/app.js', (req, res) => {
            res.setHeader('Content-Type', 'application/javascript');
            res.send(this.getJS());
        });
    }

    setupRoutes() {
        this.app.post('/api/test-connection', this.testConnection.bind(this));
        this.app.post('/api/execute-query', this.executeQuery.bind(this));
        this.app.post('/api/export-excel', this.exportToExcel.bind(this));
    }

    getClientKey(url, username) {
        return `${url}-${username}`;
    }

    async getClient(url, username, password) {
        const key = this.getClientKey(url, username);
        
        if (!this.clients.has(key)) {
            const client = new HiveClient(TCLIService, TCLIService_types);

            const [host, port] = url.includes(':') ? url.split(':') : [url, '10000'];
            
            const connection = await client.connect({
                host: host,
                port: parseInt(port),
                username: username,
                password: password,
            });

            this.clients.set(key, { client, connection });
        }

        return this.clients.get(key);
    }

    async testConnection(req, res) {
        try {
            const { url, username, password } = req.body;
            
            if (!url || !username) {
                return res.status(400).json({ 
                    success: false, 
                    message: 'URL and username are required' 
                });
            }

            const { client } = await this.getClient(url, username, password);
            const session = await client.openSession();
            await session.close();
            
            res.json({ 
                success: true, 
                message: 'Connection successful' 
            });
        } catch (error) {
            res.status(500).json({ 
                success: false, 
                message: `Connection failed: ${error.message}` 
            });
        }
    }

    async executeQuery(req, res) {
        const startTime = Date.now();
        
        try {
            const { url, username, password, query, page = 1, pageSize = 50 } = req.body;
            
            if (!url || !username || !query) {
                return res.status(400).json({
                    success: false,
                    message: 'URL, username, and query are required'
                });
            }

            const { client } = await this.getClient(url, username, password);
            const session = await client.openSession();

            // Get total count for pagination
            const countQuery = `SELECT COUNT(*) as total FROM (${query}) as subquery`;
            const countOperation = await session.executeStatement(countQuery);
            const countResult = await countOperation.fetchAll();
            const totalRecords = parseInt(countResult[0].total);
            await countOperation.close();

            // Execute main query with pagination
            const paginatedQuery = `${query} LIMIT ${pageSize} OFFSET ${(page - 1) * pageSize}`;
            const operation = await session.executeStatement(paginatedQuery);
            const schema = await operation.getSchema();
            const result = await operation.fetchAll();
            const columns = schema.columns.map(col => col.columnName);

            await operation.close();
            await session.close();

            const executionTime = Date.now() - startTime;

            // Convert results to proper format
            const formattedResults = result.map(row => {
                const obj = {};
                columns.forEach((col, index) => {
                    obj[col] = row[col] !== null && row[col] !== undefined ? row[col] : null;
                });
                return obj;
            });

            res.json({
                success: true,
                data: {
                    columns,
                    rows: formattedResults,
                    pagination: {
                        currentPage: parseInt(page),
                        pageSize: parseInt(pageSize),
                        totalRecords,
                        totalPages: Math.ceil(totalRecords / pageSize)
                    },
                    statistics: {
                        executionTime: `${executionTime}ms`,
                        recordsReturned: formattedResults.length,
                        totalRecords
                    }
                }
            });

        } catch (error) {
            res.status(500).json({
                success: false,
                message: `Query execution failed: ${error.message}`
            });
        }
    }

    async exportToExcel(req, res) {
        try {
            const { url, username, password, query, filename = 'hive-export' } = req.body;
            
            if (!url || !username || !query) {
                return res.status(400).json({
                    success: false,
                    message: 'URL, username, and query are required'
                });
            }

            const { client } = await this.getClient(url, username, password);
            const session = await client.openSession();

            const operation = await session.executeStatement(query);
            const schema = await operation.getSchema();
            const result = await operation.fetchAll();
            const columns = schema.columns.map(col => col.columnName);

            await operation.close();
            await session.close();

            // Prepare data for Excel
            const excelData = [
                columns, // Header row
                ...result.map(row => columns.map(col => row[col]))
            ];

            const workbook = XLSX.utils.book_new();
            const worksheet = XLSX.utils.aoa_to_sheet(excelData);
            XLSX.utils.book_append_sheet(workbook, worksheet, 'Hive Data');

            const buffer = XLSX.write(workbook, { type: 'buffer', bookType: 'xlsx' });

            res.setHeader('Content-Type', 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet');
            res.setHeader('Content-Disposition', `attachment; filename=${filename}.xlsx`);
            res.send(buffer);

        } catch (error) {
            res.status(500).json({
                success: false,
                message: `Export failed: ${error.message}`
            });
        }
    }

    getHTML() {
        return `<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Hive DB Query Tool</title>
    <link rel="stylesheet" href="/css/style.css">
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
</head>
<body>
    <div class="container">
        <header>
            <h1><i class="fas fa-database"></i> Hive DB Query Tool</h1>
            <p class="subtitle">Single Executable Version</p>
        </header>

        <div class="connection-panel">
            <h2>Connection Settings</h2>
            <div class="form-group">
                <label for="url">Hive Server URL:</label>
                <input type="text" id="url" placeholder="host:port (e.g., localhost:10000)" value="localhost:10000">
            </div>
            <div class="form-group">
                <label for="username">Username:</label>
                <input type="text" id="username" placeholder="Username" value="hive">
            </div>
            <div class="form-group">
                <label for="password">Password:</label>
                <input type="password" id="password" placeholder="Password">
            </div>
            <div class="button-group">
                <button id="testConnection" class="btn btn-test">
                    <i class="fas fa-plug"></i> Test Connection
                </button>
                <button id="connect" class="btn btn-connect">
                    <i class="fas fa-link"></i> Connect
                </button>
            </div>
            <div id="connectionStatus" class="status"></div>
        </div>

        <div id="queryPanel" class="query-panel" style="display: none;">
            <h2>Query Editor</h2>
            <div class="form-group">
                <label for="query">SQL Query:</label>
                <textarea id="query" rows="6" placeholder="SELECT * FROM your_table LIMIT 100">SELECT * FROM default.sample_table LIMIT 100</textarea>
            </div>
            <div class="button-group">
                <button id="executeQuery" class="btn btn-execute">
                    <i class="fas fa-play"></i> Execute Query
                </button>
                <button id="getStats" class="btn btn-stats">
                    <i class="fas fa-chart-bar"></i> Get Statistics
                </button>
                <button id="exportExcel" class="btn btn-export">
                    <i class="fas fa-file-excel"></i> Export to Excel
                </button>
            </div>

            <div id="statistics" class="statistics-panel" style="display: none;">
                <h3>Query Statistics</h3>
                <div id="statsContent"></div>
            </div>

            <div id="resultsPanel" class="results-panel" style="display: none;">
                <h3>Query Results</h3>
                <div class="pagination-controls">
                    <button id="prevPage" class="btn btn-pagination">
                        <i class="fas fa-chevron-left"></i> Previous
                    </button>
                    <span id="pageInfo">Page 1 of 1</span>
                    <button id="nextPage" class="btn btn-pagination">
                        Next <i class="fas fa-chevron-right"></i>
                    </button>
                    <select id="pageSize">
                        <option value="10">10 rows</option>
                        <option value="25" selected>25 rows</option>
                        <option value="50">50 rows</option>
                        <option value="100">100 rows</option>
                    </select>
                </div>
                <div class="table-container">
                    <table id="resultsTable">
                        <thead></thead>
                        <tbody></tbody>
                    </table>
                </div>
            </div>
        </div>

        <div id="loading" class="loading" style="display: none;">
            <i class="fas fa-spinner fa-spin"></i> Processing...
        </div>

        <footer>
            <p>Hive Query Tool - Running on port ${this.port}</p>
        </footer>
    </div>

    <script src="/js/app.js"></script>
</body>
</html>`;
    }

    getCSS() {
        return `* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    min-height: 100vh;
    color: #333;
}

.container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
    min-height: 100vh;
}

header {
    text-align: center;
    margin-bottom: 30px;
    color: white;
}

header h1 {
    font-size: 2.5em;
    margin-bottom: 5px;
    text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
}

.subtitle {
    font-style: italic;
    opacity: 0.9;
}

.connection-panel, .query-panel {
    background: rgba(255, 255, 255, 0.95);
    padding: 25px;
    border-radius: 15px;
    box-shadow: 0 8px 32px rgba(0,0,0,0.1);
    margin-bottom: 25px;
    backdrop-filter: blur(10px);
}

.form-group {
    margin-bottom: 20px;
}

label {
    display: block;
    margin-bottom: 8px;
    font-weight: 600;
    color: #2c3e50;
}

input, textarea, select {
    width: 100%;
    padding: 12px;
    border: 2px solid #e1e8ed;
    border-radius: 8px;
    font-size: 14px;
    transition: border-color 0.3s;
}

input:focus, textarea:focus, select:focus {
    outline: none;
    border-color: #3498db;
}

textarea {
    font-family: 'Courier New', monospace;
    resize: vertical;
    min-height: 120px;
}

.button-group {
    display: flex;
    gap: 12px;
    flex-wrap: wrap;
    margin: 25px 0;
}

.btn {
    padding: 12px 24px;
    border: none;
    border-radius: 8px;
    cursor: pointer;
    font-size: 14px;
    font-weight: 600;
    transition: all 0.3s;
    display: flex;
    align-items: center;
    gap: 8px;
}

.btn:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(0,0,0,0.2);
}

.btn:active {
    transform: translateY(0);
}

.btn-test { background: linear-gradient(45deg, #ffc107, #ff9800); color: #000; }
.btn-connect { background: linear-gradient(45deg, #28a745, #20c997); color: white; }
.btn-execute { background: linear-gradient(45deg, #007bff, #0056b3); color: white; }
.btn-stats { background: linear-gradient(45deg, #17a2b8, #138496); color: white; }
.btn-export { background: linear-gradient(45deg, #28a745, #20c997); color: white; }
.btn-pagination { background: linear-gradient(45deg, #6c757d, #495057); color: white; }

.status {
    padding: 15px;
    border-radius: 8px;
    margin-top: 15px;
    font-weight: 500;
}

.status.success {
    background: linear-gradient(45deg, #d4edda, #c3e6cb);
    color: #155724;
    border: 1px solid #c3e6cb;
}

.status.error {
    background: linear-gradient(45deg, #f8d7da, #f5c6cb);
    color: #721c24;
    border: 1px solid #f5c6cb;
}

.loading {
    text-align: center;
    padding: 30px;
    font-size: 18px;
    color: white;
    background: rgba(255,255,255,0.1);
    border-radius: 10px;
    backdrop-filter: blur(10px);
}

.results-panel {
    margin-top: 25px;
    animation: slideDown 0.5s ease;
}

@keyframes slideDown {
    from { opacity: 0; transform: translateY(-20px); }
    to { opacity: 1; transform: translateY(0); }
}

.pagination-controls {
    display: flex;
    align-items: center;
    gap: 15px;
    margin-bottom: 20px;
    flex-wrap: wrap;
    padding: 15px;
    background: #f8f9fa;
    border-radius: 8px;
}

.table-container {
    overflow-x: auto;
    border-radius: 8px;
    box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}

table {
    width: 100%;
    border-collapse: collapse;
    background: white;
}

th, td {
    padding: 15px;
    text-align: left;
    border-bottom: 1px solid #e1e8ed;
    white-space: nowrap;
}

th {
    background: linear-gradient(45deg, #3498db, #2980b9);
    color: white;
    font-weight: 600;
    position: sticky;
    top: 0;
}

tr:hover {
    background-color: #f8f9fa;
}

tr:nth-child(even) {
    background-color: #f8f9fa;
}

.statistics-panel {
    background: linear-gradient(45deg, #e9ecef, #dee2e6);
    padding: 20px;
    border-radius: 10px;
    margin: 20px 0;
    animation: slideUp 0.5s ease;
}

@keyframes slideUp {
    from { opacity: 0; transform: translateY(20px); }
    to { opacity: 1; transform: translateY(0); }
}

.statistics-panel h3 {
    margin-bottom: 15px;
    color: #2c3e50;
}

footer {
    text-align: center;
    color: white;
    margin-top: 30px;
    opacity: 0.8;
}

@media (max-width: 768px) {
    .container {
        padding: 10px;
    }
    
    .button-group {
        flex-direction: column;
    }
    
    .btn {
        width: 100%;
        justify-content: center;
    }
    
    .pagination-controls {
        flex-direction: column;
        align-items: stretch;
        text-align: center;
    }
    
    th, td {
        padding: 10px;
        font-size: 14px;
    }
}`;
    }

    getJS() {
        return `class HiveQueryTool {
    constructor() {
        this.currentPage = 1;
        this.pageSize = 25;
        this.totalPages = 1;
        this.currentQuery = '';
        this.connectionConfig = null;
        this.initializeEventListeners();
    }

    initializeEventListeners() {
        document.getElementById('testConnection').addEventListener('click', () => this.testConnection());
        document.getElementById('connect').addEventListener('click', () => this.connect());
        document.getElementById('executeQuery').addEventListener('click', () => this.executeQuery());
        document.getElementById('getStats').addEventListener('click', () => this.showStatistics());
        document.getElementById('exportExcel').addEventListener('click', () => this.exportToExcel());
        document.getElementById('prevPage').addEventListener('click', () => this.changePage(-1));
        document.getElementById('nextPage').addEventListener('click', () => this.changePage(1));
        document.getElementById('pageSize').addEventListener('change', (e) => {
            this.pageSize = parseInt(e.target.value);
            this.currentPage = 1;
            this.executeQuery();
        });
    }

    async testConnection() {
        const config = this.getConnectionConfig();
        if (!config) return;

        this.showLoading(true);
        
        try {
            const response = await fetch('/api/test-connection', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(config)
            });

            const result = await response.json();
            this.showStatus(result.success ? 'success' : 'error', result.message);
        } catch (error) {
            this.showStatus('error', 'Connection test failed: ' + error.message);
        } finally {
            this.showLoading(false);
        }
    }

    async connect() {
        const config = this.getConnectionConfig();
        if (!config) return;

        this.showLoading(true);
        
        try {
            const response = await fetch('/api/test-connection', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(config)
            });

            const result = await response.json();
            
            if (result.success) {
                this.connectionConfig = config;
                document.getElementById('queryPanel').style.display = 'block';
                this.showStatus('success', 'Connected successfully! You can now run queries.');
                // Scroll to query panel
                document.getElementById('queryPanel').scrollIntoView({ behavior: 'smooth' });
            } else {
                this.showStatus('error', result.message);
            }
        } catch (error) {
            this.showStatus('error', 'Connection failed: ' + error.message);
        } finally {
            this.showLoading(false);
        }
    }

    async executeQuery() {
        if (!this.connectionConfig) {
            this.showStatus('error', 'Please connect to database first');
            return;
        }

        const query = document.getElementById('query').value.trim();
        if (!query) {
            this.showStatus('error', 'Please enter a query');
            return;
        }

        this.currentQuery = query;
        this.showLoading(true);

        try {
            const response = await fetch('/api/execute-query', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                    ...this.connectionConfig,
                    query: query,
                    page: this.currentPage,
                    pageSize: this.pageSize
                })
            });

            const result = await response.json();
            
            if (result.success) {
                this.displayResults(result.data);
                this.showStatistics(result.data.statistics);
                this.showStatus('success', 'Query executed successfully!');
            } else {
                this.showStatus('error', result.message);
            }
        } catch (error) {
            this.showStatus('error', 'Query execution failed: ' + error.message);
        } finally {
            this.showLoading(false);
        }
    }

    displayResults(data) {
        const { columns, rows, pagination } = data;
        
        this.currentPage = pagination.currentPage;
        this.totalPages = pagination.totalPages;
        this.updatePaginationControls();

        const table = document.getElementById('resultsTable');
        const thead = table.querySelector('thead');
        const tbody = table.querySelector('tbody');

        thead.innerHTML = '';
        const headerRow = document.createElement('tr');
        columns.forEach(col => {
            const th = document.createElement('th');
            th.textContent = col;
            headerRow.appendChild(th);
        });
        thead.appendChild(headerRow);

        tbody.innerHTML = '';
        rows.forEach(row => {
            const tr = document.createElement('tr');
            columns.forEach(col => {
                const td = document.createElement('td');
                const value = row[col];
                td.textContent = value !== null && value !== undefined ? value.toString() : 'NULL';
                td.title = td.textContent;
                tr.appendChild(td);
            });
            tbody.appendChild(tr);
        });

        document.getElementById('resultsPanel').style.display = 'block';
        document.getElementById('resultsPanel').scrollIntoView({ behavior: 'smooth' });
    }

    updatePaginationControls() {
        document.getElementById('pageInfo').textContent = 
            Page ${this.currentPage} of ${this.totalPages} (${this.getTotalRecords()} total records);
        
        document.getElementById('prevPage').disabled = this.currentPage <= 1;
        document.getElementById('nextPage').disabled = this.currentPage >= this.totalPages;
    }

    getTotalRecords() {
        const statsPanel = document.getElementById('statistics');
        if (statsPanel.style.display !== 'none') {
            const statsContent = document.getElementById('statsContent');
            const totalMatch = statsContent.textContent.match(/Total Records: ([\\d,]+)/);
            return totalMatch ? totalMatch[1] : '0';
        }
        return '0';
    }

    changePage(direction) {
        const newPage = this.currentPage + direction;
        if (newPage >= 1 && newPage <= this.totalPages) {
            this.currentPage = newPage;
            this.executeQuery();
        }
    }

    showStatistics(stats) {
        const statsPanel = document.getElementById('statistics');
        const statsContent = document.getElementById('statsContent');
        
        if (stats) {
            statsContent.innerHTML = \\
                <p><strong>Execution Time:</strong> ${stats.executionTime}</p>
                <p><strong>Records Returned:</strong> ${stats.recordsReturned.toLocaleString()}</p>
                <p><strong>Total Records:</strong> ${stats.totalRecords.toLocaleString()}</p>
                <p><strong>Page Size:</strong> ${this.pageSize}</p>
            ;
            statsPanel.style.display = 'block';
        } else {
            statsPanel.style.display = 'none';
        }
    }

    async exportToExcel() {
        if (!this.connectionConfig || !this.currentQuery) {
            this.showStatus('error', 'Please execute a query first');
            return;
        }

        this.showLoading(true);

        try {
            const response = await fetch('/api/export-excel', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                    ...this.connectionConfig,
                    query: this.currentQuery,
                    filename: 'hive-export-' + new Date().toISOString().split('T')[0]
                })
            });

            if (response.ok) {
                const blob = await response.blob();
                const url = window.URL.createObjectURL(blob);
                const a = document.createElement('a');
                a.href = url;
                a.download = 'hive-export.xlsx';
                document.body.appendChild(a);
                a.click();
                window.URL.revokeObjectURL(url);
                document.body.removeChild(a);
                this.showStatus('success', 'Excel export completed successfully!');
            } else {
                const error = await response.json();
                throw new Error(error.message);
            }
        } catch (error) {
            this.showStatus('error', 'Export failed: ' + error.message);
        } finally {
            this.showLoading(false);
        }
    }

    getConnectionConfig() {
        const url = document.getElementById('url').value.trim();
        const username = document.getElementById('username').value.trim();
        const password = document.getElementById('password').value;

        if (!url) {
            this.showStatus('error', 'Please provide Hive server URL');
            return null;
        }

        if (!username) {
            this.showStatus('error', 'Please provide username');
            return null;
        }

        return { url, username, password };
    }

    showStatus(type, message) {
        const statusEl = document.getElementById('connectionStatus');
        statusEl.textContent = message;
        statusEl.className = status ${type};
        statusEl.style.display = 'block';
        
        // Auto-hide success messages after 5 seconds
        if (type === 'success') {
            setTimeout(() => {
                statusEl.style.display = 'none';
            }, 5000);
        }
    }

    showLoading(show) {
        document.getElementById('loading').style.display = show ? 'block' : 'none';
    }
}

// Initialize application
document.addEventListener('DOMContentLoaded', () => {
    new HiveQueryTool();
    
    // Add some sample interactions
    console.log('Hive Query Tool initialized successfully!');
    
    // Example of adding keyboard shortcut (Ctrl+Enter to execute query)
    document.addEventListener('keydown', (e) => {
        if (e.ctrlKey && e.key === 'Enter') {
            const queryTool = new HiveQueryTool();
            queryTool.executeQuery();
        }
    });
});`;
    }

    start() {
        this.app.listen(this.port, () => {
            console.log(`
🚀 Hive DB Query Tool started!
📍 Local: http://localhost:${this.port}
🌐 Network: http://${this.getIPAddress()}:${this.port}

📊 Features:
✅ Hive Database Connectivity
✅ SQL Query Execution  
✅ Pagination Support
✅ Statistics & Analytics
✅ Excel Export
✅ Responsive Web UI
✅ Single Executable File

Press Ctrl+C to stop the server
            `);
        });
    }

    getIPAddress() {
        const interfaces = require('os').networkInterfaces();
        for (const name of Object.keys(interfaces)) {
            for (const interface of interfaces[name]) {
                if (interface.family === 'IPv4' && !interface.internal) {
                    return interface.address;
                }
            }
        }
        return 'localhost';
    }
}

// Handle command line arguments
const args = process.argv.slice(2);
const portArg = args.find(arg => arg.startsWith('--port='));
const port = portArg ? parseInt(portArg.split('=')[1]) : null;

if (port) {
    process.env.PORT = port;
}

// Start the application
const app = new HiveQueryTool();
app.start();

// Handle graceful shutdown
process.on('SIGINT', () => {
    console.log('\n👋 Shutting down Hive Query Tool...');
    process.exit(0);
});

process.on('uncaughtException', (error) => {
    console.error('Uncaught Exception:', error);
});

process.on('unhandledRejection', (reason, promise) => {
    console.error('Unhandled Rejection at:', promise, 'reason:', reason);
});
```

## Build Instructions

1. **Install dependencies and pkg:**
```bash
npm install
npm install -g pkg
```

2. **Build for your platform:**
```bash
# Build for current platform
npm run build

# Or build for specific platforms
pkg app.js --target node18-win-x64 --output hive-query-tool.exe
pkg app.js --target node18-linux-x64 --output hive-query-tool
pkg app.js --target node18-macos-x64 --output hive-query-tool
```

3. **Run the executable:**
```bash
# Windows
./hive-query-tool.exe

# Linux/Mac
./hive-query-tool

# With custom port
./hive-query-tool --port=8080
```

## Features of the Single Executable

✅ **Single File**: No installation required, just download and run  
✅ **Cross-Platform**: Works on Windows, Linux, and macOS  
✅ **Zero Configuration**: Automatically serves web interface  
✅ **All Original Features**: Connection testing, query execution, pagination, statistics, Excel export  
✅ **Enhanced UI**: Beautiful gradient design with animations  
✅ **Port Configuration**: Custom port via command line argument  
✅ **Error Handling**: Comprehensive error handling and validation  
✅ **Mobile Responsive**: Works on desktop and mobile devices  

## Usage Examples

```bash
# Basic usage (default port 3000)
./hive-query-tool

# Custom port
./hive-query-tool --port=8080

# Run in background (Linux/Mac)
nohup ./hive-query-tool --port=8080 > hive-tool.log 2>&1 &

# Run as service (Windows)
hive-query-tool.exe --port=8080
```

The executable will start a web server and provide URLs to access the Hive query tool. Users can simply download the executable and run it - no Node.js installation required!
