Perfect 👍 since you want a **single pipeline** that produces *header info + CVE counts + severity counts*, here’s how you can do it stage-by-stage in **MongoDB Compass**.

This assumes your documents look something like:

```json
{
  "ait_no": "AIT-001",
  "ait_name": "Example AIT",
  "cve": "CVE-2025-0001",
  "last_severity": "HIGH"
}
```

---

## ✅ Combined Aggregation Pipeline

```javascript
[
  // 1. Group by ait_no and ait_name
  {
    $group: {
      _id: { ait_no: "$ait_no", ait_name: "$ait_name" },
      cves: { $push: "$cve" },
      severities: { $push: "$last_severity" }
    }
  },

  // 2. Reshape document
  {
    $project: {
      _id: 0,
      ait_no: "$_id.ait_no",
      ait_name: "$_id.ait_name",
      cves: 1,
      severities: 1
    }
  },

  // 3. Count CVEs
  {
    $addFields: {
      cve_summary: {
        $map: {
          input: { $setUnion: ["$cves", []] },
          as: "c",
          in: {
            cve: "$$c",
            count: {
              $size: {
                $filter: {
                  input: "$cves",
                  as: "cc",
                  cond: { $eq: ["$$cc", "$$c"] }
                }
              }
            }
          }
        }
      }
    }
  },

  // 4. Count severities
  {
    $addFields: {
      severity_summary: {
        $map: {
          input: { $setUnion: ["$severities", []] },
          as: "s",
          in: {
            severity: "$$s",
            count: {
              $size: {
                $filter: {
                  input: "$severities",
                  as: "ss",
                  cond: { $eq: ["$$ss", "$$s"] }
                }
              }
            }
          }
        }
      }
    }
  },

  // 5. Final shape
  {
    $project: {
      ait_no: 1,
      ait_name: 1,
      cve_summary: 1,
      severity_summary: 1
    }
  }
]
```

---

## ✅ Example Output

```json
{
  "ait_no": "AIT-001",
  "ait_name": "Example AIT",
  "cve_summary": [
    { "cve": "CVE-2025-0001", "count": 3 },
    { "cve": "CVE-2025-0002", "count": 1 }
  ],
  "severity_summary": [
    { "severity": "HIGH", "count": 2 },
    { "severity": "MEDIUM", "count": 2 }
  ]
}
```

---

👉 This way, in **Compass**, you can run the pipeline stage-by-stage:

* After `$group`, you’ll see all raw CVEs + severities.
* After `$addFields`, you’ll see computed counts.
* Final `$project` gives the compact result.

Do you also want me to extend this so it **joins across multiple source collections** (e.g. `collection_one` + `collection_two`) into a single combined target pipeline, or just keep it per-collection?
