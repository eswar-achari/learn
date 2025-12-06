Here is a **much clearer, structured, production-quality prompt** you can use to extract and display the vulnerability data in a **tabular format** and apply filters precisely:

---

### **🔥 Improved Prompt**

```
From the following JSON input, extract and display a table with the columns:

| Title | Severity | URL | Description | Certainty |

Processing rules:
- Only include vulnerabilities where:
    - Severity is "medium" or "high"
    - Certainty is 70 or greater
- If description is not present, leave blank.
- Display the output as a Markdown table with proper column alignment.
- Do not include websites that have zero qualifying vulnerabilities.
- After the table, display a summary:
      - Total vulnerabilities processed
      - Total vulnerabilities displayed
      - Count grouped by severity (medium/high)

Additional formatting rules:
- Truncate long text fields to 120 characters
- Convert severity to Title Case (Medium, High)
- Sort results by severity (High → Medium), then by certainty descending

Finally:
- Recommend alternative display formats (dashboard, grouped JSON, severity heatmap) and generate one best alternative JSON output optimized for APIs.

Input JSON:
<PASTE JSON HERE>
```

---

### **👍 Why This Prompt Is Better**

| Your Original Prompt | Improved Prompt                            |
| -------------------- | ------------------------------------------ |
| Linear instructions  | Structured rules                           |
| No alignment rules   | Markdown formatting rules                  |
| No sorting           | Sorting + grouping                         |
| No summary           | Adds analytics summary                     |
| Only table           | Also requests improved data representation |

---

### **📌 Optional – More Ways to Display Data**

| Format                         | When to Use           |
| ------------------------------ | --------------------- |
| Markdown Table                 | Reports / ChatGPT     |
| JSON (Grouped by Severity)     | API / Backend Systems |
| Severity Heatmap (Color-coded) | UI dashboards         |
| CSV Export                     | Excel / Data analysis |
| Compact bullet summary         | Notification & alerts |

👉 Example Enhanced API-Friendly JSON:

```json
{
  "summary": {
    "total_found": 14,
    "filtered_displayed": 4,
    "by_severity": {
      "high": 3,
      "medium": 1
    }
  },
  "data": {
    "high": [...],
    "medium": [...]
  }
}
```

---

If you want, I can also give you **a second improved prompt optimized for CSV export, PDF reporting, or API response transformation** — tell me which output format you want:
📍 **Markdown**, 📍 **CSV**, 📍 **Excel**, 📍 **JSON grouped**, 📍 **Console table**, or 📍 **PDF report**.
