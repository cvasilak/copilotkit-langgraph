# CopilotKit + LangGraph Todo Demo

## Purpose

This repository serves as both a **showcase** and **template** for building AI agents with CopilotKit and LangGraph. It demonstrates how CopilotKit can drive interactive UI beyond just chat, using a **collaborative todo list** as the primary example.

**Target audience:** Developers evaluating CopilotKit or starting new projects with AI agents.

## Core Concept

The todo list demonstrates **agent-driven UI** where:

- The agent can manipulate application state (adding todos, updating status, organizing tasks)
- Users can interact with the same state (editing titles, checking off tasks, deleting todos)
- Both agent and user changes update the same shared state
- The UI reactively updates based on agent state changes

This uses CopilotKit's **v2 agent state pattern** where state lives in the agent and syncs to the frontend.

## Architecture

This is a **flat npm project** with a Next.js frontend at the root and a Python agent in `agent/`.

### Repository Structure

```
├── src/
│   ├── app/
│   │   ├── page.tsx              # Main page - wires up all components
│   │   └── api/copilotkit/       # CopilotKit API route
│   ├── components/
│   │   ├── canvas/               # Todo list UI
│   │   │   ├── index.tsx         # Canvas container
│   │   │   ├── todo-list.tsx     # Todo list with columns
│   │   │   ├── todo-column.tsx   # Column (pending/completed)
│   │   │   └── todo-card.tsx     # Individual todo card
│   │   ├── example-layout/       # Layout: chat + canvas side-by-side
│   │   └── generative-ui/        # Example generative UI components
│   └── hooks/
│       ├── use-generative-ui-examples.tsx  # Example CopilotKit patterns
│       └── use-example-suggestions.tsx     # Chat suggestions
├── agent/                         # LangGraph Python agent
│   ├── main.py                    # Agent entry point
│   └── src/
│       ├── todos.py               # Todo tools and state schema
│       └── query.py               # Example data query tool
├── scripts/                       # Agent setup and run scripts
│   ├── setup-agent.sh / .bat
│   └── run-agent.sh / .bat
├── package.json                   # Root project config (npm + concurrently)
└── next.config.ts
```

## Key Pattern: Agent State with CopilotKit v2

The todo list uses **CopilotKit v2's agent state pattern** where state lives in the agent backend and syncs bidirectionally with the frontend.

### How It Works

1. **Agent defines state schema and tools** (Python)

   ```python
   # agent/src/todos.py
   from langchain.agents import AgentState as BaseAgentState

   class Todo(TypedDict):
       id: str
       title: str
       description: str
       emoji: str
       status: Literal["pending", "completed"]

   # Extends LangChain's AgentState, which provides `messages` (add_messages reducer)
   class AgentState(BaseAgentState):
       todos: list[Todo]  # no reducer: each update replaces the whole list

   @tool
   def manage_todos(todos: list[Todo], runtime: ToolRuntime) -> Command:
       """Manage the current todos."""
       return Command(update={"todos": todos, ...})
   ```

   - `AgentState` must subclass LangChain's `AgentState` (not plain `TypedDict`) — `create_agent` needs the inherited `messages` key.
   - `todos` has no reducer, so writes are last-write-wins: tools and `agent.setState()` must always send the **full** list.
   - Tools read state via `runtime.state` and write via `Command(update=...)`; a state-updating tool must also return a `ToolMessage` with the matching `tool_call_id`.

2. **Frontend reads from agent state**

   ```typescript
   // src/components/canvas/index.tsx
   const { agent } = useAgent();

   return (
     <TodoList
       todos={agent.state?.todos || []}
       onUpdate={(updatedTodos) => agent.setState({ todos: updatedTodos })}
       isAgentRunning={agent.isRunning}
     />
   );
   ```

3. **User interactions update agent state**

   ```typescript
   // User clicks checkbox → frontend calls agent.setState()
   const toggleStatus = (todo) => {
     const updated = todos.map((t) =>
       t.id === todo.id
         ? { ...t, status: t.status === "completed" ? "pending" : "completed" }
         : t,
     );
     agent.setState({ todos: updated });
   };
   ```

4. **Agent can manipulate state via tools**
   - The agent calls `manage_todos` tool to update the todo list
   - Both user and agent changes update the same `agent.state.todos`
   - Frontend automatically re-renders when state changes

### Why This Pattern?

- **Single source of truth**: State lives in the agent, not duplicated in frontend
- **Bidirectional sync**: User changes → agent state, Agent changes → UI update
- **Simple**: No need for separate frontend state management
- **Observable**: Agent has full visibility into state changes

## Implementation Details

### Agent Backend

**Agent Definition** (`agent/main.py`):

```python
from langchain.agents import create_agent
from copilotkit import CopilotKitMiddleware, StateStreamingMiddleware, StateItem
from langchain_openai import ChatOpenAI
from src.todos import todo_tools, AgentState

model = ChatOpenAI(model="gpt-5.4-mini", model_kwargs={"parallel_tool_calls": False})

agent = create_agent(
    model=model,
    tools=[*todo_tools, ...],  # manage_todos, get_todos
    middleware=[
        CopilotKitMiddleware(),  # syncs graph state <-> frontend agent.state
        StateStreamingMiddleware(  # streams manage_todos args into state.todos live
            StateItem(state_key="todos", tool="manage_todos", tool_argument="todos")
        ),
    ],
    state_schema=AgentState,  # Defines state shape
    system_prompt="You are a helpful assistant..."
)
```

`StateStreamingMiddleware` makes todos appear in the UI while the model is still generating the `manage_todos` call, instead of only after the tool completes. Add a `StateItem` for any new state key that a tool writes wholesale from one of its arguments.

**Todo Tools** (`agent/src/todos.py`):

```python
@tool
def manage_todos(todos: list[Todo], runtime: ToolRuntime) -> Command:
    """Manage the current todos."""
    # Ensure todos have unique IDs
    for todo in todos:
        if "id" not in todo or not todo["id"]:
            todo["id"] = str(uuid.uuid4())

    # Update agent state
    return Command(update={
        "todos": todos,
        "messages": [ToolMessage(...)]
    })

@tool
def get_todos(runtime: ToolRuntime):
    """Get the current todos."""
    return runtime.state.get("todos", [])
```

### Frontend

**Canvas Component** (`src/components/canvas/index.tsx`):

```typescript
export function Canvas() {
  const { agent } = useAgent();  // CopilotKit v2 hook

  return (
    <div className="h-full p-8 bg-gray-50">
      <TodoList
        // Read state from agent
        todos={agent.state?.todos || []}
        // Update state in agent
        onUpdate={(updatedTodos) => agent.setState({ todos: updatedTodos })}
        // React to agent execution
        isAgentRunning={agent.isRunning}
      />
    </div>
  );
}
```

**Todo List** (`src/components/canvas/todo-list.tsx`):

```typescript
export function TodoList({ todos, onUpdate, isAgentRunning }: TodoListProps) {
  const toggleStatus = (todo: Todo) => {
    const updated = todos.map((t) =>
      t.id === todo.id
        ? { ...t, status: t.status === "completed" ? "pending" : "completed" }
        : t
    );
    onUpdate(updated);  // Calls agent.setState()
  };

  const addTodo = () => {
    const newTodo = { id: crypto.randomUUID(), ... };
    onUpdate([...todos, newTodo]);
  };

  return (
    <div className="flex gap-8">
      <TodoColumn title="To Do" todos={pendingTodos} onAddTodo={addTodo} ... />
      <TodoColumn title="Done" todos={completedTodos} ... />
    </div>
  );
}
```

### How State Flows

1. **User adds/edits todo** → Frontend calls `agent.setState({ todos: [...] })`
2. **Agent state updates** → CopilotKit syncs to backend
3. **Agent observes change** → Can respond via `manage_todos` tool
4. **Agent modifies todos** → Calls `manage_todos` tool
5. **State syncs to frontend** → `agent.state.todos` updates
6. **UI re-renders** → React sees new state and updates display

**Key insight**: State lives in the agent, frontend just reads/writes to it via CopilotKit hooks.

## Tech Stack

- **Frontend**: Next.js 16, React 19, TailwindCSS 4
- **Agent**: LangGraph (Python), OpenAI GPT-5.4 mini (via `langchain_openai`)
- **CopilotKit**: React hooks for agent integration (v2)
- **Build**: npm with concurrently for parallel dev processes
- **Other**: Recharts for generative UI examples

## Development

```bash
# Install dependencies (also sets up agent via postinstall)
npm install

# Start both frontend and agent
npm run dev

# Start individually
npm run dev:ui      # Next.js frontend on port 3000
npm run dev:agent   # LangGraph agent on port 8123

# Build
npm run build
```

### Environment Setup

```bash
# Set OpenAI API key
cp .env.example .env
# Edit .env and add your OPENAI_API_KEY
```

## Design Principles

1. **Simple over complex** - The todo list is intentionally simple and focused
2. **CopilotKit v2 patterns** - Uses modern agent state management
3. **Template-first** - Code is meant to be forked and extended
4. **Showcasing agent-driven UI** - Demonstrates AI manipulating application state beyond chat

---

## Key Takeaways for Developers

**State Management Pattern**: This app uses CopilotKit v2's agent state pattern where:

- State is defined in the agent backend (a subclass of LangChain's `AgentState`)
- Frontend reads via `agent.state.todos`
- Frontend writes via `agent.setState({ todos: ... })`
- Agent can modify state via tools (`manage_todos`)
- Changes sync bidirectionally automatically

**When extending this template**:

- Define state schema in the agent (`AgentState`, subclassing LangChain's `AgentState`)
- Create tools that manipulate state via `Command(update={...})`
- Use `Annotated[..., reducer]` on a state key if agent and user writes should merge rather than replace
- Register a `StateItem` in `StateStreamingMiddleware` to stream a tool-written key to the UI live
- Use `useAgent()` hook in frontend to read/write state
- Let CopilotKit handle the sync - no manual state management needed

This pattern works great for **agent-driven applications** where the AI needs to manipulate structured application state, not just chat.
