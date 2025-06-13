# agent-manager/src/main.py

## Overview

This script is the main entry point for the Agent Manager service. It sets up and runs a gRPC server that allows clients to execute and manage browser-based AI agents. The service handles the lifecycle of these agents, including starting browser instances, initializing the agent with appropriate configurations (LLM provider, model, etc.), managing agent state, and streaming results back to the client. It also interacts with a PostgreSQL database to persist session information and agent state.

## Key Components

- `AgentManagerServicer`: The core class that implements the gRPC `AgentManagerService`. It contains the logic for handling agent execution requests.
  - `RunAgent(request, context)`: Handles non-streaming requests to run an agent. It starts a browser, initializes the agent, runs it with the given prompt, and returns the final result.
  - `RunAgentStream(request, context)`: Handles streaming requests to run an agent. This method allows for real-time updates and interaction with the agent, yielding chunks of data (steps, final output, errors, timeouts) as they occur. It can reuse existing browser sessions for chat-like interactions.
  - `_init_agent(...)`: A helper method responsible for creating and configuring an `Agent` instance. It sets up the browser connection (CDP URL), selects the LLM provider (Anthropic, Bedrock, OpenAI, Gemini), and other agent parameters.
  - `_run_agent(...)`: A private helper method that synchronously executes the agent's main `run` logic and formats the output.
  - `_update_agent_session(...)`: Updates or inserts agent session details (CDP URL, VNC URL, machine ID) into the database.
  - `_update_agent_machine_status(...)`: Updates the status of the browser machine (e.g., "running", "stopped") in the database.
  - `_update_agent_state(...)`: Saves the agent's internal state to the database.
  - `_update_agent_status(...)`: Updates the agent's operational status (e.g., "idle", "working") in the database.
  - `_get_agent_state(...)`: Retrieves the last saved state for an agent from the database.
  - `_get_agent_chat_session(...)`: Retrieves comprehensive session details by joining data from `agent_sessions` and `agent_chats` tables.
- `serve()`: An asynchronous function that initializes the global database connection (`db_conn`) and starts the gRPC server, making it listen for incoming requests on the configured port.

## Important Variables/Constants

- `logger`: A standard Python `logging.Logger` instance used for logging messages throughout the application.
- `port`: The network port on which the gRPC server listens. It defaults to "8901" and can be overridden by the `PORT` environment variable.
- `scrapybara`: An instance of the `Scrapybara` client, used to start, stop, and manage browser instances that the agents will control.
- `db_conn`: An `asyncpg.Connection` object representing the connection to the PostgreSQL database. It is initialized in the `serve()` function.

## Usage Examples

This script is typically run as a standalone server:

```bash
python agent-manager/src/main.py
```

Clients would then connect to this server using a gRPC client generated from the corresponding `.proto` file (e.g., `agent_manager_grpc_pb2_grpc`). The client would call `RunAgent` or `RunAgentStream` methods.

Example (conceptual client-side Python):
```python
# import grpc
# import agent_manager_grpc_pb2
# import agent_manager_grpc_pb2_grpc

# channel = grpc.insecure_channel('localhost:8901')
# stub = agent_manager_grpc_pb2_grpc.AgentManagerServiceStub(channel)

# request = agent_manager_grpc_pb2.RunAgentRequest(
#     session_id="some-session-id",
#     prompt="Navigate to example.com and find the title.",
#     model_provider=agent_manager_grpc_pb2.ModelProvider.ANTHROPIC, # Enum value
#     model="claude-3-opus-20240229"
# )
# response = stub.RunAgent(request)
# print(response.result.content)
```

## Dependencies and Interactions

- **gRPC (`grpcio`)**: Used for defining and running the main service API. The script implements a service defined in protobuf (likely `agent_manager_grpc.proto`).
- **AsyncPG (`asyncpg`)**: Used for asynchronous communication with a PostgreSQL database to store and manage agent sessions, state, and status.
- **DotEnv (`python-dotenv`)**: Used to load environment variables (e.g., `DATABASE_URL`, `SCRAPYBARA_API_KEY`, `PORT`) from a `.env` file.
- **Laminar (`lmnr`)**: Used for distributed tracing and observability. It can deserialize parent span contexts from requests.
- **Scrapybara (`scrapybara`)**: Used to manage the lifecycle of browser instances (start, stop, get CDP URL) that the agents use for web interaction.
- **Internal Modules**:
    - `agent_manager_grpc_pb2` and `agent_manager_grpc_pb2_grpc`: Generated Python code from protobuf definitions, providing the gRPC message types and service stubs/servers.
    - `index` (specifically `index.Agent`, `index.agent.agent` and provider classes like `AnthropicProvider`): Contains the core `Agent` logic, data structures for agent communication (e.g., `StepChunk`), and implementations for different LLM providers.
- **Environment Variables**:
    - `PORT`: Defines the port for the gRPC server.
    - `SCRAPYBARA_API_KEY`: API key for the Scrapybara service.
    - `DATABASE_URL`: Connection string for the PostgreSQL database.
    - `BACKEND_URL`, `BACKEND_HTTP_PORT`, `BACKEND_GRPC_PORT`: Potentially for Laminar or other backend service communication.
- **Database**: The script heavily relies on a PostgreSQL database with tables like `agent_sessions` and `agent_chats` to persist and manage agent-related data.
- **LLM Providers**: Interacts with various LLM providers (Anthropic, OpenAI, Google Gemini, AWS Bedrock) through the `Agent` class, which abstracts the specific provider logic.
```
