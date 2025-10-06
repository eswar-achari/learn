Here’s a **PlantUML** diagram representing the flow and logic shown in your image — including `PENPAL-UI`, `PENPAL-API`, and `GES` components with success/failure paths and authorization checks.

```plantuml
@startuml
skinparam style strict
skinparam rectangle {
  BackgroundColor #F9F9F9
  BorderColor #333
  RoundCorner 15
}
skinparam arrow {
  Color #444
  Thickness 1.2
}

title PENPAL Authorization Evaluation Flow

actor "User (NBK123)" as User
rectangle "PENPAL-UI" as UI {
  User --> UI : Request VIEW\nDocument-123
  UI --> API : Send VIEW Request
}

rectangle "PENPAL-API" as API {
  API : Extract classification = PUBLIC\nExtract sec-func = IAM\nfrom Elastic
  API --> GES : GES.isAllowed(sub, action, resource)
}

rectangle "GES System" as GES {
  GES : Evaluate Policy
  GES : subject = NBK123\nresource = DATA/IAM/PUBLIC\naction = VIEW
  GES --> API : Decision: ALLOWED or DENIED
}

API --> UI : Return Result
UI --> User : ✅ Success (Document-123)\n❌ Failure (Document-456)

note right of API
Success Case:
  classification = PUBLIC
  sec-func = IAM
  → GES.isAllowed → ALLOWED

Failure Case:
  classification = PRIVATE
  sec-func = IAM
  → GES.isAllowed → DENIED
end note

@enduml
```

---

### 🧭 Explanation:

* **Actors:**

  * `User` initiates a document view request.
* **PENPAL-UI:** Forwards request and displays result.
* **PENPAL-API:** Fetches classification & sec-function from Elastic.
* **GES:** Evaluates `(subject, action, resource)` and returns `ALLOW` or `DENY`.
* **Two outcomes:**
  ✅ `NBK123 / Document-123 / VIEW` → Success (Public IAM)
  ❌ `NBK456 / Document-456 / VIEW` → Failure (Private IAM)

---

Would you like me to convert this flow into a **Mermaid** diagram as well for markdown-based rendering (e.g., in GitHub/Notion)?
