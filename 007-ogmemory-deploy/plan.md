oG-Memory Refactoring Plan: Convergence to pip-installable Package

 ▎ Goal: Transform oG-Memory into a pip-installable package with og-memory serve command

 ▎ Guiding Principles:
 ▎ 1. No file movement - preserve existing directory structure
 ▎ 2. Minimal code changes - only add deployment wrapper layer
 ▎ 3. Preserve environment variables - no config.toml introduction
 ▎ 4. Record issues only, don't fix them during migration

 Context

 The oG-Memory project currently requires manual deployment:
 1. Run openGauss database
 2. Install AGFS binary manually
 3. Build Docker image or run Python scripts
 4. Configure multiple services

 The goal is to simplify this to:
 pip install og-memory
 og-memory serve  # Starts API + AGFS automatically

 While maintaining compatibility with existing deployments and keeping the OpenClaw plugin separate.

 Implementation Plan

 Phase 1: Update pyproject.toml

 File: /data/Workspace2/oG-Memory/pyproject.toml

 Changes:
 1. Update project name from context-engine to og-memory
 2. Add click and python-dotenv to dependencies
 3. Register CLI entry point:
 [project.scripts]
 og-memory = "og_memory.cli:cli"
 4. Ensure all runtime dependencies are listed:
   - Add missing: flask>=2.0.0, requests>=2.28.0, psycopg2-binary>=2.9.0, rich>=13.0.0
   - Keep existing: pyagfs>=1.4.0, openai>=1.0.0, apscheduler>=3.10.0

 Key consideration: Use packages = [{include = ".", from = "."}] to include all Python files in root directory without moving them.

 Phase 2: Create CLI Module

 New file: /data/Workspace2/oG-Memory/og_memory/cli.py

 Commands to implement:
 - og-memory serve - Main command to start API + AGFS
 - og-memory setup - Interactive setup wizard
 - og-memory check - Environment validation
 - og-memory db init - Initialize database tables
 - og-memory stop - Stop running services

 Core implementation approach:
 1. Use python-dotenv to load .env file (current dir → ~/.og-memory/.env)
 2. Check required environment variables:
   - OPENGUASS_CONNECTION_STRING (required)
   - OPENAI_API_KEY (optional, warn if missing)
 3. AGFS management:
   - Check AGFS_EXTERNAL_URL env var first
   - If not set, detect agfs-server in PATH or ~/.local/bin/
   - Start AGFS subprocess with agfs-server -c agfs-config.yaml
   - Wait for health check (max 10s)
 4. Verify openGauss connectivity
 5. Start existing HTTP server by calling server.app:main() or equivalent
 6. Handle SIGINT/SIGTERM for graceful shutdown

 Phase 3: Create AGFS Manager Module

 New file: /data/Workspace2/oG-Memory/agfs_manager.py

 Features:
 1. Detection:
   - Check which agfs-server and ~/.local/bin/agfs-server
   - Support AGFS_EXTERNAL_URL env var for external AGFS
 2. Startup:
   - Use subprocess.Popen to start AGFS
   - Pass configuration file path
   - Mount prefix from environment
 3. Health check:
   - Poll http://localhost:1833/serverinfo/version
   - Timeout after 10s
 4. Shutdown:
   - SIGTERM → wait 5s → SIGKILL
   - Registered with atexit as fallback
 5. Error handling:
   - Clear error message if AGFS not found with installation command

 Phase 4: Create Environment Template

 New file: /data/Workspace2/oG-Memory/.env.example

 Content based on environment variable analysis:
 # OpenGauss Database
 OPENGUASS_CONNECTION_STRING=host=127.0.0.1 port=8888 dbname=postgres user=aaa password=aaa@123123

 # OpenAI API
 OPENAI_API_KEY=your_openai_api_key_here
 OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
 OPENAI_EMBEDDING_MODEL=text-embedding-ada-002
 OPENAI_LLM_MODEL=gpt-4o-mini

 # oG-Memory Server
 OGMEM_HTTP_PORT=8090
 OG_ACCOUNT_ID=acct-demo
 OG_USER_ID=u-alice
 OG_AGENT_ID=main

 # AGFS Configuration
 AGFS_BASE_URL=http://localhost:1833
 AGFS_MOUNT_PREFIX=/local/plugin

 # Vector Database
 VECTOR_DB_TYPE=opengauss
 OPENGUASS_TABLE_NAME=vector_index
 OPENGUASS_POOL_SIZE=5

 # Provider Configuration
 CONTEXTENGINE_PROVIDER=openai
 enable_cache=true
 cache_max_size=1000

 # Model Parameters
 llm_temperature=0.7
 llm_max_tokens=4096

 Phase 5: Adapt OpenClaw Plugin

 Current plugin: /data/Workspace2/oG-Memory/openclaw_context_engine_plugin/

 No changes needed for the plugin itself, as:
 1. Plugin configuration already supports memoryApiBaseUrl
 2. Plugin will continue to work with standalone HTTP server
 3. Installation remains separate: openclaw plugins install openclaw_context_engine_plugin

 Startup Sequence (og-memory serve)

 [1] load_dotenv()  # Load .env, existing os.getenv() works unchanged
        │
 [2] Check required env vars
     ├── OPENGUASS_CONNECTION_STRING missing? → Error and exit
     └── OPENAI_API_KEY missing? → Warning, continue
        │
 [3] AGFS Management
     ├── AGFS_EXTERNAL_URL set? → Skip, use external URL
     └── Not set →
          ├── Check for agfs-server → Not found? → Error with install command
          ├── Found → Start subprocess with agfs-server -c agfs-config.yaml
          └── Health check timeout → Error
        │
 [4] Verify openGauss connection → Fail? → Error
        │
 [5] Start API server
     └── Call existing server.app:main() (no code changes)
        │
 [Service Ready]

 Ctrl+C or SIGTERM →
     ├── Stop API server
     └── Stop AGFS subprocess (SIGTERM → 5s → SIGKILL)

 Final User Experience

 # Prerequisites: AGFS installed, openGauss running
 # AGFS: curl -fsSL https://raw.githubusercontent.com/c4pt0r/agfs/master/install.sh | sh
 # openGauss: docker run ... or existing instance

 # First time setup
 pip install og-memory
 og-memory setup          # Interactive → generates .env + initializes DB
 og-memory serve          # Starts AGFS + API server automatically

 # Daily usage
 og-memory serve

 # OpenClaw integration (optional)
 openclaw plugins install openclaw_context_engine_plugin

 # CI/Docker environment (fully compatible)
 export OPENGUASS_CONNECTION_STRING="host=... port=... ..."
 export OPENAI_API_KEY="sk-xxx"
 og-memory serve

 Verification

 After implementation, verify:
 1. pip install -e . creates og-memory command
 2. og-memory serve starts both AGFS and API server
 3. AGFS subprocess stops on Ctrl+C
 4. Missing AGFS shows helpful error message
 5. External AGFS mode works with AGFS_EXTERNAL_URL
 6. Existing .env files work without modification
 7. OpenClaw plugin connects to http://localhost:8090