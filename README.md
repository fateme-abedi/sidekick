# 🤖 Sidekick — Agentic AI Personal Assistant

Sidekick is an **Agentic AI personal assistant** built with Python and the LangChain ecosystem.

The project combines an LLM-powered agent with real-world tools such as web search, Wikipedia, browser automation, filesystem access, push notifications, and human-in-the-loop interactions.

Unlike a simple chatbot, Sidekick is designed to **perform tasks**, evaluate its own progress, recover from tool failures, and ask the user for help only when human intervention is genuinely required.

---

## ✨ Features

* 🧠 **LLM-powered Agent** using LangChain's `create_agent`
* 🔍 **Web Search** using Google Serper
* 📚 **Wikipedia Search**
* 🌐 **Browser Automation** using Playwright MCP
* 📁 **Filesystem Access** using MCP
* 🔔 **Push Notifications** using Pushover
* 👤 **Human-in-the-Loop** for login, CAPTCHA and 2FA
* 🛡️ **Middleware-based Tool Error Handling**
* 🔄 **Evaluator Loop** for checking task completion
* 📋 **Structured Evaluation Output** using Pydantic
* 🔐 Environment-based API key management using `.env`
* ✈️ Specialized workflow for flight searches through Google Flights
* 🧩 **Model Context Protocol (MCP)** integration
* ⚡ Asynchronous MCP session management using `asyncio`

---

# 🏗️ Architecture

The core idea of Sidekick is to put an intelligent evaluation loop around a single agent.

```text
                        ┌─────────────────────┐
                        │       User          │
                        └──────────┬──────────┘
                                   │
                                   ▼
                        ┌─────────────────────┐
                        │   Evaluator Loop    │
                        │                     │
                        │ Is the task done?   │
                        └──────────┬──────────┘
                                   │
                                   ▼
                        ┌─────────────────────┐
                        │   Sidekick Agent    │
                        │    create_agent     │
                        └──────────┬──────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
                    ▼              ▼              ▼
               Web Search      Wikipedia       MCP Tools
                                                  │
                              ┌───────────────────┼──────────────────┐
                              │                   │                  │
                              ▼                   ▼                  ▼
                         Playwright          Filesystem        Other Tools
                              │
                              ▼
                         Web Browser

                                   │
                                   ▼
                         ┌─────────────────────┐
                         │    Middleware       │
                         │                     │
                         │ Error Handling      │
                         │ Guardrails          │
                         │ Human Approval      │
                         └─────────────────────┘
```

---

# 🧠 How the Agent Works

Sidekick does not simply generate an answer and stop.

The general workflow is:

```text
User Request
     │
     ▼
Agent works on the task
     │
     ▼
Agent produces a result
     │
     ▼
Evaluator checks the result
     │
     ├── Success ──────────────► Return result
     │
     ├── User input required ──► Ask user
     │
     └── Not successful ───────► Give feedback
                                      │
                                      ▼
                               Agent tries again
```

This creates a simple **agent → evaluator → retry** architecture.

---

# 🧩 Main Components

## 1. Sidekick Agent

The main worker is created using LangChain's `create_agent`.

The agent receives a system prompt that defines:

* its role
* available capabilities
* browser usage rules
* human intervention rules
* flight search workflow
* task completion requirements

The agent is instructed to continue working until:

1. the success criteria are satisfied, or
2. it genuinely needs information or action from the user.

---

## 2. Evaluator

The evaluator determines whether the agent actually completed the task.

Evaluation results are represented using a Pydantic model:

```python
class EvaluatorOutput(BaseModel):
    feedback: str
    success_criteria_met: bool
    user_input_needed: bool
```

The evaluator provides three important pieces of information:

| Field                  | Purpose                                    |
| ---------------------- | ------------------------------------------ |
| `feedback`             | Explains what the agent should improve     |
| `success_criteria_met` | Determines whether the task is complete    |
| `user_input_needed`    | Determines whether the user must intervene |

This structured output makes the control flow predictable.

---

# 🛠️ Tools

Sidekick can interact with the outside world through multiple tools.

### Google Search

Google Serper is wrapped as a LangChain tool:

```python
search = GoogleSerperRun(
    api_wrapper=GoogleSerperAPIWrapper()
)
```

The agent can use this tool to search the web.

### Wikipedia

Wikipedia is exposed as another tool:

```python
wikipedia_lookup = WikipediaQueryRun(
    api_wrapper=WikipediaAPIWrapper()
)
```

### Push Notifications

Sidekick can send notifications to the user's phone through Pushover.

```python
@tool
def send_push_notification(text: str) -> str:
    ...
```

### Human Help

For tasks that require a human, such as:

* logging in
* solving CAPTCHA
* completing two-factor authentication

the agent can use:

```python
@tool
def request_human_help(instructions: str) -> str:
    ...
```

---

# 🌐 MCP Integration

One of the important parts of this project is its use of the **Model Context Protocol (MCP)**.

Sidekick connects to multiple MCP servers.

Currently, the architecture includes:

### Playwright MCP

Provides browser automation capabilities.

```text
Agent
  │
  ▼
Playwright MCP
  │
  ▼
Real Web Browser
```

The agent can navigate websites, inspect pages, interact with web content, and perform browser-based tasks.

### Filesystem MCP

Provides access to a controlled sandbox filesystem.

```text
Agent
  │
  ▼
Filesystem MCP
  │
  ▼
Sandbox Directory
```

The filesystem is restricted to the configured sandbox directory.

---

# 🔌 MCP Session Management

The project maintains persistent MCP sessions using asynchronous Python.

The `McpSessions` class is responsible for:

* creating MCP clients
* opening MCP sessions
* loading MCP tools
* keeping sessions alive
* shutting down MCP servers cleanly

Conceptually:

```text
McpSessions
     │
     ├── Playwright Session
     │
     └── Filesystem Session
```

Keeping the browser session alive is particularly important because the browser needs to preserve its state across multiple tool calls.

---

# 🛡️ Middleware

Sidekick uses middleware to control and protect agent-tool interactions.

For example, `TolerateToolErrors` prevents temporary tool failures from immediately terminating the agent.

```python
class TolerateToolErrors(AgentMiddleware):

    async def awrap_tool_call(self, request, handler):
        try:
            return await handler(request)

        except Exception as error:
            return ToolMessage(
                content=f"That tool call failed: {error}. "
                        "Try another approach.",
                tool_call_id=request.tool_call["id"],
            )
```

Instead of:

```text
Tool Error
   │
   ▼
Application crashes
```

the project uses:

```text
Tool Error
   │
   ▼
Middleware catches error
   │
   ▼
ToolMessage sent to Agent
   │
   ▼
LLM understands the failure
   │
   ▼
Agent tries another approach
```

This makes the agent more resilient.

---

# 👤 Human-in-the-Loop

Some browser tasks cannot safely or practically be completed autonomously.

Examples include:

* authentication
* CAPTCHA
* two-factor authentication
* approving sensitive actions

Sidekick can recognize these situations and ask the user for assistance.

The intended workflow is:

```text
Agent
  │
  ▼
Reaches human-only action
  │
  ▼
request_human_help()
  │
  ▼
User performs the action
  │
  ▼
Agent continues
```

This creates a **human + agent collaboration model** instead of requiring full automation.

---

# ✈️ Example: Flight Search

The agent has a specialized instruction for flight searches.

Instead of interacting with the Google Flights UI unnecessarily, it can navigate directly to a natural-language Google Flights query.

For example:

```text
flights from New York to London
leaving 14 July
returning 21 July
```

The browser can then be used to inspect the available flight information.

The system prompt encourages the agent to prefer page snapshots over unnecessary clicking.

---

# 🔐 Environment Variables

API keys and sensitive configuration should not be hard-coded into the source code.

Create a `.env` file:

```env
GOOGLE_API_KEY=your_google_api_key
SERPER_API_KEY=your_serper_api_key
PUSHOVER_TOKEN=your_pushover_token
PUSHOVER_USER=your_pushover_user
```

Then load the variables using:

```python
from dotenv import load_dotenv

load_dotenv(override=True)
```

> Never commit your `.env` file to GitHub.

Add it to `.gitignore`:

```gitignore
.env
.venv/
__pycache__/
*.pyc
```

---

# 📦 Installation

## 1. Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/YOUR_REPOSITORY.git
cd YOUR_REPOSITORY
```

## 2. Create a virtual environment

Using Python:

```bash
python -m venv .venv
```

Activate it on Windows:

```powershell
.venv\Scripts\activate
```

---

## 3. Install dependencies

Install the required Python packages:

```bash
pip install -r requirements.txt
```

If the project uses `uv`, dependencies can alternatively be installed with:

```bash
uv sync
```

---

## 4. Configure environment variables

Create:

```text
.env
```

and add the required API keys.

---

## 5. Install Node.js / npm

The MCP servers are launched through `npx`, so Node.js and npm must be installed.

The project uses MCP servers such as:

```text
@playwright/mcp
@modelcontextprotocol/server-filesystem
```

---

# ▶️ Running the Project

After configuring the environment:

```bash
python sidekick.py
```

Depending on the project structure, the application can also be started through its main entry point.

---

# 📁 Project Structure

A simplified structure looks like:

```text
sidekick/
│
├── sidekick.py
├── sidekick_tools.py
├── requirements.txt
├── .env
├── .gitignore
└── README.md
```

### `sidekick.py`

Contains the main Sidekick agent, evaluator loop, middleware and orchestration logic.

### `sidekick_tools.py`

Contains custom tools and MCP session management.

### `.env`

Stores API credentials and configuration.

### `requirements.txt`

Contains Python dependencies.

---

# 🧱 Technologies

The project demonstrates several modern AI engineering concepts:

* **Python**
* **LangChain**
* **LangGraph ecosystem**
* **LLM Agents**
* **Pydantic**
* **Model Context Protocol (MCP)**
* **Playwright**
* **Google Serper**
* **Wikipedia API**
* **Pushover**
* **asyncio**
* **Middleware**
* **Human-in-the-Loop**
* **Structured Output**
* **Tool Calling**
* **Agent Evaluation**

---

# 🎯 Learning Objectives

This project was built to explore practical Agentic AI concepts, including:

1. Building tool-using LLM agents
2. Connecting agents to external services
3. Using MCP servers
4. Browser automation with Playwright
5. Managing asynchronous MCP sessions
6. Designing middleware around agents
7. Handling tool failures gracefully
8. Implementing human-in-the-loop workflows
9. Using Pydantic for structured LLM output
10. Building evaluator-driven agent loops
11. Designing agents that can recover from failures
12. Separating agent reasoning from application control flow

---

# 🔄 Agent vs. Traditional Chatbot

A traditional chatbot usually follows:

```text
User
 ↓
LLM
 ↓
Answer
```

Sidekick follows a more agentic architecture:

```text
User
 ↓
Agent
 ↓
Tools
 ↓
Environment
 ↓
Result
 ↓
Evaluator
 ↓
 ├── Complete → User
 │
 ├── Need user → User
 │
 └── Incomplete → Agent → Tools → ...
```

The key difference is that Sidekick is designed to **take actions and verify progress**, rather than only generate text.

---

# 🚧 Project Status

This project is currently a learning/experimental implementation of an Agentic AI personal assistant.

The architecture is intentionally designed to demonstrate modern agent patterns rather than provide a production-ready personal assistant.

Future improvements may include:

* More robust evaluator strategies
* Persistent conversation memory
* Better human approval workflows
* More MCP integrations
* Additional browser automation capabilities
* Better security and permission controls
* Improved observability and tracing
* More sophisticated task planning
* Cost and token monitoring
* Production deployment

---

# 📚 Concepts Demonstrated

```text
                    Agentic AI
                        │
        ┌───────────────┼────────────────┐
        │               │                │
      Agents           Tools            MCP
        │               │                │
        │        ┌──────┼──────┐         │
        │        │      │      │         │
        │      Search  Wiki  Push    Playwright
        │                              Filesystem
        │
        ├── Middleware
        │
        ├── Evaluator
        │
        ├── Structured Output
        │
        └── Human-in-the-Loop

