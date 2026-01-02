Got it 👍
Below is a **clean, production-ready Python script** that calls the API shown in your screenshot and **safely supports multi-line `content` strings** using Python triple quotes.

I’ll show:

1. ✅ **Recommended approach (requests library)**
2. ✅ How to keep **system + user prompts as multiline**
3. ✅ ENV-based token handling

---

## ✅ Python script to call the API (multiline-safe)

```python
import os
import json
import requests

# ----------------------------
# CONFIGURATION
# ----------------------------
API_URL = "https://<YOUR_API_ENDPOINT>/v1/chat/completions"
API_TOKEN = os.getenv("API_TOKEN")  # export API_TOKEN=xxxxx

MODEL_NAME = "trtllm_25.08_meta-llama_Llama-3.3-70B-Instruct_fp16_tp4_ifb"

# ----------------------------
# MULTI-LINE PROMPTS
# ----------------------------

SYSTEM_PROMPT = """
CVE Information Retrieval**

When a user requests information about a specific Common Vulnerability and Exposure (CVE),
use the provided input to create a dictionary with the following keys:

- cveId: The CVE ID to search for
- description: A brief description of the CVE
- severity: The severity level of the CVE (e.g., low, medium, high)
- publishedDate: The date when the CVE was published
- modifiedDate: The date when the CVE was last modified
- cvssScore: The CVSS score of the CVE
- cvssVector: The CVSS vector string of the CVE

User Input Processing:
Based on the user's question, populate the required parameters in a Python dictionary representation.
Only return the dictionary and do not provide any additional information.
If the user does not provide a specific value, leave the corresponding key empty.
If the user's query does not match any field, mention that field as "USER_LOOKING_FOR".

Current Date:
November 2025
"""

USER_PROMPT = """
Give me the vulnerabilities associated to spring framework annotation
"""

# ----------------------------
# REQUEST PAYLOAD
# ----------------------------
payload = {
    "model": MODEL_NAME,
    "messages": [
        {
            "role": "system",
            "content": SYSTEM_PROMPT
        },
        {
            "role": "user",
            "content": USER_PROMPT
        }
    ],
    "stream": False,
    "temperature": 0.0,
    "max_tokens": 2048
}

# ----------------------------
# API CALL
# ----------------------------
headers = {
    "Authorization": f"Bearer {API_TOKEN}",
    "Content-Type": "application/json"
}

response = requests.post(
    API_URL,
    headers=headers,
    data=json.dumps(payload),
    timeout=60
)

# ----------------------------
# RESPONSE HANDLING
# ----------------------------
if response.status_code == 200:
    result = response.json()
    print(json.dumps(result, indent=2))
else:
    print("Request failed")
    print("Status:", response.status_code)
    print("Response:", response.text)
```

---

## ✅ Why this works well

✔ **Multiline strings**
Uses Python’s `""" """` syntax — no escaping nightmares.

✔ **Readable prompts**
System prompt stays exactly as you designed it.

✔ **Secure token handling**
Token is loaded from ENV, not hardcoded.

✔ **Drop-in replacement for curl**
Matches your original request structure 1:1.

---

## 🔐 Set API token (Mac / Linux)

```bash
export API_TOKEN="your_real_token_here"
```

Windows (PowerShell):

```powershell
setx API_TOKEN "your_real_token_here"
```

---

## 🧠 Pro tip (optional)

If you plan to **dynamically inject user queries**, just replace `USER_PROMPT` at runtime:

```python
def call_api(user_query: str):
    payload["messages"][1]["content"] = user_query
```

---

If you want:

* async (`httpx`)
* streaming support
* MCP / FastMCP wrapper
* LangGraph integration

Say the word 🚀
