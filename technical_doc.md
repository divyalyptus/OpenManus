# OpenManus Agent: Technical Workflow

This document outlines the technical workflow of the OpenManus agent, as primarily defined by `main.py` and the `app.agent.manus.Manus` class. The agent is designed to understand user prompts, interact with various tools (including web browsers and external services via MCP), and accomplish tasks.

## 1. Initialization and Setup

The agent's lifecycle begins in `main.py`, which orchestrates its creation and execution.

**A. Agent Instantiation (`Manus.create()`):**
   - An instance of the `Manus` agent is created using the asynchronous factory method `Manus.create()`. `Manus` itself inherits from the `ToolCallAgent` class, which provides the core logic for tool interaction and the thinking loop.
   - During this creation process:
     - A `BrowserContextHelper` is initialized. This helper is responsible for managing browser state and injecting relevant browser context into prompts when the `BrowserUseTool` is active.
     - **MCP Server Connection:** The agent attempts to connect to "MCP (Multi-Content Prompt) servers" defined in its configuration (`config.mcp_config.servers`).
       - These servers can be of type "sse" (Server-Sent Events, for web-based tools/APIs) or "stdio" (for command-line tools).
       - Tools exposed by successfully connected MCP servers are dynamically added to the `Manus` agent's collection of `available_tools`.

**B. Core Available Tools:**
   By default, the `Manus` agent is equipped with a set of built-in tools:
   - `PythonExecute`: Executes Python code snippets.
   - `BrowserUseTool`: Interacts with a web browser (e.g., navigating, extracting content). This tool works in conjunction with the `BrowserContextHelper`.
   - `StrReplaceEditor`: Performs simple find-and-replace operations on strings or file contents.
   - `AskHuman`: Pauses execution and requests input from the user.
   - `Terminate`: Signals that the agent has completed its task or should stop.

## 2. Task Processing (`agent.run(prompt)`)

Once initialized, the `agent.run(prompt)` method (inherited from `ToolCallAgent`) is invoked with the user's request (currently hardcoded in `main.py` to be: `"which are the 10 top most traded stocks in the indian stock market(BSE) today?"`). This initiates the main processing loop.

**A. The Thinking Loop (`Manus.think()` and `ToolCallAgent.think()`):**
   The agent enters a loop, driven by the `think()` method (with `Manus.think()` overriding and extending `ToolCallAgent.think()`), to decide and execute steps:

   1.  **Prompt Formulation:**
       - The agent constructs a detailed prompt for a Large Language Model (LLM).
       - This prompt typically includes:
         - A **System Prompt** (from `app.prompt.manus.SYSTEM_PROMPT`): Defines the agent's persona, capabilities, tools, and general instructions. It's formatted with the workspace root directory.
         - **Conversation History:** Previous user messages, LLM responses, and tool outputs are included to provide context.
         - The **Current User Prompt** or a **Next Step Prompt** (from `app.prompt.manus.NEXT_STEP_PROMPT`).
         - **Dynamic Browser Context (if applicable):** If the `BrowserUseTool` has been used recently, the `Manus.think()` method calls `browser_context_helper.format_next_step_prompt()`. This injects current browser state information (e.g., page content, open tabs) into the `next_step_prompt`, making the LLM aware of what the browser is currently displaying.

   2.  **LLM Interaction:**
       - The combined prompt is sent to the configured LLM.
       - The LLM's response is expected to be a plan, a request to call a specific tool with certain arguments, or a final answer.

   3.  **Tool Call Execution:**
       - If the LLM requests a tool call:
         - The agent identifies the requested tool from its `available_tools` collection (which includes both built-in and MCP-provided tools).
         - The tool is executed with the arguments specified by the LLM.
         - For example, the LLM might request `BrowserUseTool` to navigate to a URL, or `PythonExecute` to run some analysis code.

   4.  **Observation and Memory:**
       - The output or result of the executed tool is captured.
       - This observation is added to the conversation history (memory).

   5.  **Loop Continuation/Termination:**
       - The loop continues, with the agent re-evaluating the situation (`think()`) based on the new information.
       - The process repeats until:
         - The LLM determines the task is complete and calls the `Terminate` tool.
         - The `AskHuman` tool is invoked, pausing for user intervention.
         - A pre-set maximum number of steps (`max_steps` in `Manus`, defaulting to 20) is reached.
         - An unrecoverable error occurs.

## 3. Resource Cleanup (`agent.cleanup()`)

After the `run` method completes (either successfully or due to an interruption/error), the `finally` block in `main.py` ensures that `agent.cleanup()` is called.

**A. Browser Cleanup:**
   - If the `BrowserUseTool` was utilized, `browser_context_helper.cleanup_browser()` is called to close any open browser instances or related resources.

**B. MCP Server Disconnection:**
   - The agent disconnects from all connected MCP servers using `disconnect_mcp_server()`. This releases connections and cleans up resources associated with these external tools.

This workflow enables the OpenManus agent to tackle complex, multi-step tasks by leveraging an LLM for reasoning and decision-making, and a flexible toolset for interacting with its environment and external services.
