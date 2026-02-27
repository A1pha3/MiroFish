# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Setup
```bash
# Install all dependencies (root + frontend + backend)
npm run setup:all

# Or install separately
npm run setup          # Node.js dependencies
npm run setup:backend  # Python dependencies via uv
```

### Running the Application
```bash
# Start both frontend and backend
npm run dev

# Start individually
npm run backend   # Backend on http://localhost:5001
npm run frontend  # Frontend on http://localhost:3000
```

### Testing
Python tests use pytest. Run from the `backend/` directory:
```bash
cd backend && uv run pytest
```

### Building
```bash
npm run build  # Build frontend for production
```

### Docker Deployment
```bash
# Configure .env file first
cp .env.example .env

# Start containers
docker compose up -d
```

The application runs as a single container with both frontend (port 3000) and backend (port 5001) exposed.

## Architecture Overview

MiroFish is a multi-agent swarm intelligence prediction engine with a **Flask backend** and **Vue 3 frontend**.

### Backend Architecture (`backend/app/`)

The backend follows a service-oriented architecture with three main API blueprints:

- **`/api/graph`** - Knowledge graph construction using Zep Cloud
- **`/api/simulation`** - OASIS-based social media simulation (Twitter + Reddit)
- **`/api/report`** - Report generation using ReACT pattern

#### Core Services

**Graph Building** (`services/graph_builder.py`, `services/ontology_generator.py`):
- Extracts entities/relationships from uploaded documents
- Uses Zep Cloud API to build GraphRAG knowledge graphs
- `ontology_generator.py` uses LLM to generate ontology from text

**Simulation Management** (`services/simulation_manager.py`):
- Manages simulation lifecycle (created → preparing → ready → running → completed)
- Reads entities from Zep graphs and generates OASIS agent profiles
- Uses `oasis_profile_generator.py` to create agent personas (supports parallel LLM generation)
- Uses `simulation_config_generator.py` to intelligently generate simulation parameters via LLM

**Simulation Execution** (`services/simulation_runner.py`):
- Runs OASIS simulation scripts as **subprocesses** (not in-process)
- Scripts located in `backend/scripts/`: `run_twitter_simulation.py`, `run_reddit_simulation.py`, `run_parallel_simulation.py`
- Uses **filesystem-based IPC** (`services/simulation_ipc.py`) for communication between Flask and simulation processes
- Monitors action logs and updates Zep graph memory in real-time

**Report Agent** (`services/report_agent.py`):
- Generates prediction reports using ReACT pattern (thought → action → observation)
- Has access to Zep tools: semantic search, entity retrieval, insight extraction, and agent interviews
- Detailed logging to `agent_log.jsonl` for debugging

### Frontend Architecture (`frontend/src/`)

- **Vue 3** with Composition API and **Vite**
- **D3.js** for graph visualization
- **Vue Router** for navigation

Key views:
- `MainView.vue` - Main workflow orchestration (5 steps)
- `SimulationRunView.vue` - Real-time simulation monitoring
- `ReportView.vue` - Report display and interaction
- `InteractionView.vue` - Chat with agents or report agent

Step components correspond to workflow stages:
- `Step1GraphBuild.vue` - Document upload and ontology generation
- `Step2EnvSetup.vue` - Entity selection and profile configuration
- `Step3Simulation.vue` - Simulation execution and monitoring
- `Step4Report.vue` - Report generation
- `Step5Interaction.vue` - Post-simulation agent interaction

## Key Patterns and Conventions

### Zep Cloud Integration
- All graph operations use `zep_cloud` SDK
- Graph sessions store entity/relationship data
- `zep_tools.py` provides unified tool interface for Report Agent
- `zep_graph_memory_updater.py` updates graph memory during simulation

### OASIS Simulation
- **Twitter** requires CSV format for agent profiles
- **Reddit** requires JSON format for agent profiles
- Dual-platform simulations run in parallel via `run_parallel_simulation.py`
- Platform-specific actions configured in `Config.OASIS_TWITTER_ACTIONS` and `Config.OASIS_REDDIT_ACTIONS`

### LLM Client
- `utils/llm_client.py` provides unified OpenAI-compatible interface
- Supports any LLM API following OpenAI SDK format
- Configuration via environment variables: `LLM_API_KEY`, `LLM_BASE_URL`, `LLM_MODEL_NAME`
- Optional boost LLM configuration for performance

### IPC Communication
- Flask sends commands to `ipc_commands/` directory
- Simulation scripts poll for commands and write responses to `ipc_responses/`
- Command types: `interview`, `batch_interview`, `close_env`
- `SimulationIPCClient` (Flask) and `SimulationIPCServer` (scripts)

### File Upload Handling
- Uploads stored in `backend/uploads/`
- Supported formats: PDF, MD, TXT, MARKDOWN
- `utils/file_parser.py` handles text extraction
- Encoding detection for non-UTF8 files via `charset-normalizer`

### Retry Mechanism
- `utils/retry.py` provides `@retry_with_backoff` decorator for LLM API calls
- Supports both sync and async variants
- Configurable max retries, exponential backoff, and jitter
- `RetryableAPIClient` class for batch processing with individual item retry

### Interview Pattern
- When interviewing agents via IPC, prefix prompts with:
  ```
  "结合你的人设、所有的过往记忆与行动，不调用任何工具直接用文本回复我："
  ```
- This prevents agents from calling tools during interviews (see `api/simulation.py`)

## Environment Configuration

Required in `.env`:
```
LLM_API_KEY=your_api_key
LLM_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
LLM_MODEL_NAME=qwen-plus
ZEP_API_KEY=your_zep_api_key
```

Optional boost LLM for parallel operations:
```
LLM_BOOST_API_KEY=your_api_key
LLM_BOOST_BASE_URL=your_base_url
LLM_BOOST_MODEL_NAME=your_model_name
```

## Important Notes

- Python 3.11-3.12 required
- Backend uses `uv` for dependency management (not pip)
- Frontend requires Node.js 18+
- Simulation scripts run as subprocesses - they must handle their own cleanup
- Windows console UTF-8 encoding handled in `run.py`
- All JSON responses disable ASCII escaping for proper Chinese display

## Data Flow: End-to-End Workflow

The application follows a 5-step workflow:

1. **Graph Build** (`Step1GraphBuild.vue` → `/api/graph`):
   - User uploads document (PDF/MD/TXT)
   - `ontology_generator.py` uses LLM to extract entity types
   - `graph_builder.py` chunks text and sends to Zep Cloud API
   - Zep builds GraphRAG knowledge graph with entities and relationships

2. **Environment Setup** (`Step2EnvSetup.vue` → `/api/simulation`):
   - `zep_entity_reader.py` fetches entities from Zep graph
   - User selects entity types to include
   - `oasis_profile_generator.py` creates agent profiles (parallel LLM calls)
   - `simulation_config_generator.py` generates simulation parameters via LLM

3. **Simulation Run** (`Step3Simulation.vue` → `/api/simulation`):
   - `simulation_runner.py` spawns subprocess for OASIS scripts
   - Scripts read profiles and config from `uploads/simulations/{id}/`
   - Agents interact on Twitter/Reddit platforms
   - `zep_graph_memory_updater.py` updates graph with agent actions
   - IPC enables real-time interview commands during simulation

4. **Report Generation** (`Step4Report.vue` → `/api/report`):
   - `report_agent.py` uses ReACT pattern with Zep tools
   - Plans outline → generates sections → refines via reflection
   - Tools: semantic search, entity retrieval, insight extraction, interviews
   - Logs detailed actions to `agent_log.jsonl`

5. **Interaction** (`Step5Interaction.vue`):
   - Chat with individual agents via IPC interview commands
   - Chat with Report Agent for additional analysis
