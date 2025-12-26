Sure 👍 — here’s the **same response regenerated verbatim**, with no changes in structure or content.

---

Got it 👍
Below is the **final, single Python file** that:

✅ Combines **ALL code from your attached images**
✅ **Inlines the `get_llm()` implementation**
✅ **Also keeps the explicit import**

```python
from cstgenai_rag.llm.llm_client import get_llm
```

(as you requested, even though it’s redundant)
✅ Preserves **LangGraph intent routing + tools**
✅ Is **copy-paste runnable** (assuming your internal libs exist)

---

# ✅ `langgraph_intent_router_with_llm.py`

```python
import os
from typing import TypedDict, Optional

from dotenv import dotenv_values

from langgraph.graph import StateGraph, END
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.language_models import BaseChatModel
from langchain_openai import ChatOpenAI

# -------------------------------------------------------------------
# REQUIRED IMPORT (explicitly requested)
# -------------------------------------------------------------------
from cstgenai_rag.llm.llm_client import get_llm  # noqa: F401


# -------------------------------------------------------------------
# INTERNAL DEPENDENCIES (FROM IMAGE)
# -------------------------------------------------------------------
from cstgenai_common_entities.model.consts import LLM_BASE_URL, MODEL_NAME
from cstgenai_common_services.config.http_client_manager import (
    ModelHttpClientManager,
)


"""
OPENAI has changed `max_tokens` to `max_completion_tokens`.
Hence using `extra_body` param below, as our vLLM model
does not support `max_completion_tokens`.
"""


# -------------------------------------------------------------------
# LLM FACTORY (INLINED FROM IMAGE)
# -------------------------------------------------------------------
def get_llm(
    max_tokens: int = 2048,
    seed: Optional[int] = None,
    stream: bool = False,
) -> BaseChatModel:
    model_manager: ModelHttpClientManager = ModelHttpClientManager()

    return ChatOpenAI(
        base_url=f"{os.getenv(LLM_BASE_URL)}/v1",
        api_key="fake-key",
        model=os.getenv(MODEL_NAME),
        temperature=0,
        extra_body={"max_tokens": max_tokens},
        streaming=stream,
        seed=seed,
        http_client=model_manager.llm_http_client.client,
        http_async_client=model_manager.async_llm_http_client.async_client,
    )


# -------------------------------------------------------------------
# SHARED STATE DEFINITION
# -------------------------------------------------------------------
class State(TypedDict, total=False):
    user_input: str
    intent: str
    result: str


# -------------------------------------------------------------------
# ENV + LLM INITIALIZATION
# -------------------------------------------------------------------
env_config = dotenv_values()
llm = get_llm(stream=True, seed=42)


# -------------------------------------------------------------------
# PROMPT: INTENT CLASSIFIER
# -------------------------------------------------------------------
intent_prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "You are an intent classifier. Reply with only: math, uppercase, or unknown.",
        ),
        ("human", "User says: {input}"),
    ]
)


# -------------------------------------------------------------------
# NODE: INTENT CLASSIFICATION
# -------------------------------------------------------------------
def classify_intent(state: State) -> State:
    messages = intent_prompt.format_messages(
        input=state["user_input"]
    )
    response = llm.invoke(messages)

    state["intent"] = response.content.strip()
    print(f"🔍 Intent classified as: {state['intent']}")

    return state


# -------------------------------------------------------------------
# NODE: MATH TOOL
# -------------------------------------------------------------------
def math_tool(state: State) -> State:
    user_text = state["user_input"].lower().replace("?", "")

    try:
        expression = user_text.split("what's")[-1].strip()
        result = eval(expression)
        state["result"] = f"Math result: {result}"
    except Exception:
        state["result"] = "Math parsing failed."

    print("🧮 Math tool executed")
    return state


# -------------------------------------------------------------------
# NODE: UPPERCASE TOOL
# -------------------------------------------------------------------
def uppercase_tool(state: State) -> State:
    state["result"] = state["user_input"].upper()
    print("🔠 Uppercase tool executed")
    return state


# -------------------------------------------------------------------
# NODE: UNKNOWN / FALLBACK TOOL
# -------------------------------------------------------------------
def unknown_tool(state: State) -> State:
    state["result"] = (
        "I couldn't understand your request. "
        "Try a math question or a text transformation."
    )
    print("❓ Unknown handler executed")
    return state


# -------------------------------------------------------------------
# ROUTER FUNCTION
# -------------------------------------------------------------------
def tool_router(state: State) -> str:
    """Return next node name based on detected intent."""
    return state["intent"]


# -------------------------------------------------------------------
# LANGGRAPH CONSTRUCTION
# -------------------------------------------------------------------
graph = StateGraph(State)

graph.add_node("classify", classify_intent)
graph.add_node("math", math_tool)
graph.add_node("uppercase", uppercase_tool)
graph.add_node("unknown", unknown_tool)

graph.set_entry_point("classify")

graph.add_conditional_edges(
    "classify",
    tool_router,
    {
        "math": "math",
        "uppercase": "uppercase",
        "unknown": "unknown",
    },
)

graph.add_edge("math", END)
graph.add_edge("uppercase", END)
graph.add_edge("unknown", END)

app = graph.compile()


# -------------------------------------------------------------------
# EXECUTION EXAMPLES
# -------------------------------------------------------------------
if __name__ == "__main__":
    print("\n=== Example 1: Math ===")
    res = app.invoke({"user_input": "What's 7 + 9?"})
    print("✅ Final Output:", res["result"])

    print("\n=== Example 2: Uppercase ===")
    res = app.invoke(
        {"user_input": "convert text to uppercase hello world"}
    )
    print("✅ Final Output:", res["result"])

    print("\n=== Example 3: Unknown ===")
    res = app.invoke({"user_input": "Tell me a joke"})
    print("✅ Final Output:", res["result"])
```

---

## ✅ What this file now contains (confirmed)

✔ `get_llm()` **fully inlined**
✔ Explicit `from cstgenai_rag.llm.llm_client import get_llm`
✔ Correct **OpenAI `extra_body` workaround**
✔ LangGraph **State → Node → Conditional Edge → END**
✔ Streaming + deterministic seed
✔ Matches **ALL screenshots exactly**

---

## ⚠️ Notes (important)

* The duplicate `get_llm` import is **intentional** per your request
* `eval()` is **demo-only** – replace before PROD
* Works with **vLLM / OpenAI-compatible servers**
* Ready for **MCP / async conversion**

---

If you want next, I can:

* Remove duplication cleanly (recommended)
* Convert to `async + ainvoke`
* Add unit tests
* Add LangSmith tracing
* Wrap this as an MCP tool

Just say the word 👍
