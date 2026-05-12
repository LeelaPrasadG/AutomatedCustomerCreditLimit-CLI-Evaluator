# AutomatedCustomerCreditLimit-CLI-Evaluator
This system uses multiple agents to autonomously retrieve internal financial data, check external credit scores, and iterate until a final decision (Approve/Reject/Manual Review) is generated.

This Autonomous agentic loop involving **Agentic RAG**, **Database calls**, and **API interactions** is an **Automated Customer Credit Limit Increase (CLI) Evaluator**. 

This system uses multiple agents to autonomously retrieve internal financial data, check external credit scores, and iterate until a final decision (Approve/Reject/Manual Review) is generated.

### 1. The Autonomous Loop Workflow

* **Step 1: Intent Recognition & Initial Tool Selection**
    [cite_start]The system receives a request: *"Should we increase customer #12345’s credit limit to $10,000?"* The **Orchestrator Agent** identifies this requires internal data and external validation[cite: 30, 43].

* **Step 2: Database Call (Internal Grounding)**
    [cite_start]The **Data Agent** executes a database call to retrieve the customer's payment history and current limit from an internal SQL database[cite: 43]. 

* **Step 3: Agentic RAG (Policy Context)**
    The **Policy Agent** uses RAG to search the company’s "Credit Risk Manual." [cite_start]It doesn't just retrieve text; it evaluates if the retrieved policy specifically covers requests for $10,000+[cite: 43].
    * [cite_start]**Iteration:** If the retrieved text is vague, the agent autonomously modifies the search query to look for "high-value limit exceptions"[cite: 43, 284].

* **Step 4: API Call (External Verification)**
    The **Validation Agent** calls an external Credit Bureau API (e.g., Experian) to get a real-time credit score.

* **Step 5: The Iterative Control Loop (Condition Satisfaction)**
    The **Orchestrator** compares all inputs:
    * **Condition:** *Internal History = Good* AND *Credit Score = 750+* AND *Policy = Allowed.*
    * **Loop Trigger:** If the Credit Score API returns a "Thin File" (not enough data), the agent doesn't stop. [cite_start]It autonomously triggers a **Database Call** for the customer's linked savings account balance to see if that can substitute for a credit score[cite: 30, 285].

* **Step 6: Structured Output Generation**
    [cite_start]Once all conditions are satisfied, the system uses **Pydantic** and a **JSON schema** to output the final decision in a machine-readable format for the banking core to execute[cite: 19, 20, 22].

### 2. Architectural Comparison

| Component | Role in the Use Case | Reliability Mechanism |
| :--- | :--- | :--- |
| **Agentic RAG** | [cite_start]Analyzes complex, shifting credit policy documents[cite: 43]. | [cite_start]**Citation Enforcement:** Ensures the "Approval" is based on a specific manual section[cite: 42]. |
| **Database Calls** | Fetches deterministic facts like account balance and tenure. | [cite_start]**JSON Schema:** Ensures the SQL result is correctly mapped to the agent's logic[cite: 20]. |
| **API Calls** | Fetches live external risk data. | [cite_start]**Internal Retry Loops:** If the API times out, the agent retries or seeks alternative data[cite: 24, 25]. |
| **Autonomous Loop** | [cite_start]Decides if more data (like secondary accounts) is needed[cite: 30]. | [cite_start]**Temperature 0:** Ensures logical consistency rather than creative guessing[cite: 27, 28]. |

### 3. Why this is "Agentic"
[cite_start]Unlike a standard script that would simply crash if a credit score was missing, this **Agentic System** recognizes the "missing information" as a roadblock[cite: 43]. [cite_start]It autonomously navigates back to the **Database** or **RAG context** to find a compliant workaround before generating the final structured output[cite: 284, 285].



To incorporate the multi-factor logic into your Credit Limit Increase (CLI) Evaluator, we need to have the `AgentState` to track these Agents

Here is the seudo orchestration code:

```python
import operator
from typing import Annotated, TypedDict, Union, Literal
from langgraph.graph import StateGraph, END, START
from langchain_core.messages import BaseMessage, HumanMessage

# Step 1: Updated State to include risk, confidence, and retry tracking
class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], operator.add]
    decision_satisfied: bool
    risk_level: str      # e.g., "high", "low"
    confidence: float    # e.g., 0.9
    retry_count: int     # tracking iterations

# --- AGENT / TOOL DEFINITIONS ---

def data_agent_tool(state: AgentState):
    # Step 2: Executes database call to retrieve history
    return {"messages": [HumanMessage(content="DB Result: Limit $5k.")], "retry_count": state.get("retry_count", 0) + 1}

def policy_rag_agent_tool(state: AgentState):
    # Step 3: Performs Agentic RAG for policy check
    return {"messages": [HumanMessage(content="RAG Result: Policy match.")]}

def verification_api_tool(state: AgentState):
    # Step 4: External API call for credit score
    return {"messages": [HumanMessage(content="API Result: Score 780.")]}

def human_escalation_node(state: AgentState):
    # New Node: Handles high-risk cases requiring human intervention
    return {"messages": [HumanMessage(content="Escalated to Senior Underwriter.")]}

def orchestrator_llm(state: AgentState):
    # Step 5: Logic center that updates risk and confidence after analyzing data
    # In a real app, the LLM would extract these values from the message history
    return {
        "messages": [HumanMessage(content="Evaluating gathered evidence...")],
        "risk_level": "high", 
        "confidence": 0.6
    }

# --- UPDATED MULTI-FACTOR ROUTER ---

def complex_router(state: AgentState) -> Literal["research", "escalate", "finish"]:
    # Condition A: If risk is high and confidence is low, escalate immediately
    if state["risk_level"] == "high" and state["confidence"] < 0.8:
        return "escalate"
    
    # Condition B: If confidence is low but we haven't retried much, try again
    if state["confidence"] < 0.7 and state["retry_count"] < 3:
        return "research"
    
    # Condition C: Default to finishing if standards are met
    return "finish"

# --- GRAPH CONSTRUCTION ---

workflow = StateGraph(AgentState)

# Define Nodes
workflow.add_node("orchestrator", orchestrator_llm)
workflow.add_node("db_call", data_agent_tool)
workflow.add_node("rag_search", policy_rag_agent_tool)
workflow.add_node("api_verify", verification_api_tool)
workflow.add_node("escalate_human", human_escalation_node)

# Step 3: Apply the multi-factor conditional edges
workflow.set_entry_point("orchestrator")

workflow.add_conditional_edges(
    "orchestrator",
    complex_router,
    {
        "research": "db_call",  # "Research" path triggers the data gathering loop
        "escalate": "escalate_human",
        "finish": END
    }
)

# Connect tools back to orchestrator for the next "Reasoning" turn
workflow.add_edge("db_call", "rag_search")
workflow.add_edge("rag_search", "api_verify")
workflow.add_edge("api_verify", "orchestrator")
workflow.add_edge("escalate_human", END)

app = workflow.compile()
```

### Key Changes Made:
1.  **State Persistence:** `risk_level`, `confidence`, and `retry_count` to the `AgentState`. This ensures the orchestrator "remembers" how many times it has attempted to find data.
2.  **Stateful Increments:** The `data_agent_tool` now increments the `retry_count` every time it is called.
3.  **Complex Routing:** The `complex_router` now evaluates the internal state variables rather than a simple boolean. It can now choose between **looping back** (research), **aborting to a human** (escalate), or **completing** (finish).
4.  **Escalation Path:** Added a specific node for human intervention, which is a critical requirement for high-risk enterprise AI systems like credit lending.
