# Claude Development Guide for Zen MCP Server

**Version:** 4.8.3 | **Last Updated:** 2025-11-17

This comprehensive guide contains everything needed for AI assistants to effectively develop and maintain the Zen MCP Server. It covers project architecture, development workflows, coding conventions, testing procedures, and operational commands.

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture & Structure](#architecture--structure)
3. [Key Conventions & Patterns](#key-conventions--patterns)
4. [Development Workflows](#development-workflows)
5. [Quick Reference Commands](#quick-reference-commands)
6. [Testing Guide](#testing-guide)
7. [Component Deep Dives](#component-deep-dives)
8. [Adding New Features](#adding-new-features)
9. [Common Troubleshooting](#common-troubleshooting)
10. [Recent Updates](#recent-updates)

---

## Project Overview

### What is Zen MCP Server?

Zen MCP Server is a production-ready **Model Context Protocol (MCP) server** that enables Claude to orchestrate multiple AI models for enhanced code analysis, debugging, and collaborative development. It's essentially "Claude Code for Claude Code" - allowing Claude to delegate specialized tasks to the most appropriate AI model.

**Core Capabilities:**
- **Multi-Model Orchestration** - Claude intelligently selects from Gemini Pro/Flash, O3/O4, GROK, OpenRouter, Ollama, and custom endpoints
- **True AI Collaboration** - Multi-turn conversations with context preservation across tools and models
- **9 Specialized Tools** - chat, thinkdeep, codereview, precommit, debug, analyze, refactor, tracer, testgen
- **Context Revival** - Revolutionary system to recover context after Claude's memory resets
- **Production-Ready** - Docker deployment, Redis persistence, comprehensive logging, extensive testing

### Key Technologies

- **MCP Protocol** - JSON-RPC communication with Claude
- **Pydantic** - Request/response validation and type safety
- **Redis** - Conversation threading and persistence
- **Docker** - Containerized deployment
- **pytest** - Testing framework (361+ unit tests)
- **Google GenAI SDK** - Gemini model integration
- **OpenAI SDK** - O3/O4 model integration

### Project Metrics

- **Lines of Code:** ~15,000+ (excluding tests)
- **Unit Tests:** 361+ (100% passing)
- **Integration Tests:** 24 scenarios
- **Tools:** 9 specialized developer tools
- **Providers:** 7 (Gemini, OpenAI, X.AI, OpenRouter, Custom, Ollama, vLLM)
- **Documentation:** 9 comprehensive guides + inline docs

---

## Architecture & Structure

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Claude CLI                            │
│                  (User's Terminal/Desktop)                   │
└────────────────────────┬────────────────────────────────────┘
                         │ MCP Protocol (JSON-RPC)
                         │ stdio transport
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    Zen MCP Server                            │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  server.py - MCP Protocol Handler                   │    │
│  │  - Tool discovery & registration                    │    │
│  │  - Request routing                                  │    │
│  │  - Response formatting                              │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                         │                                    │
│  ┌──────────────────────┴──────────────────────────────┐    │
│  │            Tool Registry (tools/)                    │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐            │    │
│  │  │   chat   │ │codereview│ │  debug   │  + 6 more  │    │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘            │    │
│  │       └────────────┴────────────┘                   │    │
│  │                     │                                │    │
│  │            ┌────────▼────────┐                       │    │
│  │            │  BaseTool       │                       │    │
│  │            │  - Validation   │                       │    │
│  │            │  - Memory mgmt  │                       │    │
│  │            │  - File handling│                       │    │
│  │            └────────┬────────┘                       │    │
│  └─────────────────────┼────────────────────────────────┘    │
│                        │                                     │
│  ┌─────────────────────┴────────────────────────────────┐   │
│  │        Provider Registry (providers/)                │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐        │   │
│  │  │ Gemini │ │ OpenAI │ │  GROK  │ │OpenRtr │ +more  │   │
│  │  └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘        │   │
│  │      └──────────┴──────────┴──────────┘              │   │
│  │                     │                                 │   │
│  │         ┌───────────▼───────────┐                     │   │
│  │         │  ModelProviderRegistry│                     │   │
│  │         │  - Auto model selection│                    │   │
│  │         │  - Fallback handling  │                     │   │
│  │         │  - Capability detection│                    │   │
│  │         └──────────┬────────────┘                     │   │
│  └────────────────────┼──────────────────────────────────┘   │
│                       │                                      │
│  ┌────────────────────┴──────────────────────────────────┐  │
│  │        Utilities (utils/)                             │  │
│  │  - conversation_memory.py (Redis threading)           │  │
│  │  - file_utils.py (File handling & validation)         │  │
│  └───────────────────────────────────────────────────────┘  │
└────────────┬───────────────────────────────────┬────────────┘
             │                                   │
             ▼                                   ▼
    ┌────────────────┐                 ┌────────────────┐
    │ Redis Container│                 │ External Models│
    │ (Conversation  │                 │ - Gemini API   │
    │  Threading)    │                 │ - OpenAI API   │
    └────────────────┘                 │ - X.AI API     │
                                       │ - OpenRouter   │
                                       │ - Custom APIs  │
                                       └────────────────┘
```

### Directory Structure (Detailed)

```
zen-mcp-server/
├── server.py                    # MCP server entry point (47K lines with logging)
├── zen_server.py                # Simple wrapper script
├── config.py                    # Centralized configuration (version, models, limits)
│
├── tools/                       # Tool implementations (273K total)
│   ├── __init__.py              # Tool exports
│   ├── base.py                  # Abstract base class (89K - critical!)
│   ├── models.py                # Shared Pydantic models
│   ├── chat.py                  # General development chat (8.7K)
│   ├── thinkdeep.py            # Extended reasoning (11.5K)
│   ├── codereview.py           # Professional code review (14.7K)
│   ├── precommit.py            # Pre-commit validation (29K)
│   ├── debug.py                # Debugging assistant (10.4K)
│   ├── analyze.py              # Smart file analysis (8.7K)
│   ├── refactor.py             # Intelligent refactoring (28.3K)
│   ├── tracer.py               # Static code analysis prompts (18.3K)
│   ├── testgen.py              # Test generation (21.7K)
│   └── listmodels.py           # Model enumeration (11.5K)
│
├── providers/                   # AI provider integrations (119K total)
│   ├── __init__.py              # Provider exports
│   ├── base.py                  # Provider base class (7.7K)
│   ├── registry.py              # Provider registry & routing (19.1K)
│   ├── gemini.py               # Google Gemini integration (15K)
│   ├── openai.py               # OpenAI integration (6.9K)
│   ├── openai_compatible.py    # OpenAI-compatible wrapper (27.7K)
│   ├── xai.py                  # X.AI GROK integration (5K)
│   ├── openrouter.py           # OpenRouter integration (6.7K)
│   ├── openrouter_registry.py  # OpenRouter model registry (9.4K)
│   └── custom.py               # Custom endpoint support (11.8K)
│
├── systemprompts/              # Tool-specific prompts (62K total)
│   ├── __init__.py              # Prompt exports
│   ├── chat_prompt.py          # Chat system prompt
│   ├── thinkdeep_prompt.py     # Extended reasoning prompt
│   ├── codereview_prompt.py    # Code review prompt
│   ├── precommit_prompt.py     # Pre-commit prompt
│   ├── debug_prompt.py         # Debugging prompt
│   ├── analyze_prompt.py       # Analysis prompt
│   ├── refactor_prompt.py      # Refactoring prompt (20K - largest)
│   └── testgen_prompt.py       # Test generation prompt
│
├── utils/                      # Utility modules
│   ├── conversation_memory.py  # Redis-based threading system
│   ├── file_utils.py           # File handling & path translation
│   └── __init__.py
│
├── tests/                      # Unit tests (657K total, 361+ tests)
│   ├── conftest.py             # pytest configuration & fixtures
│   ├── test_*.py               # 47+ test files
│   └── triangle.png            # Test image asset
│
├── simulator_tests/            # Integration tests (221K total)
│   ├── base_test.py            # Test base class
│   ├── test_*.py               # 24 integration test scenarios
│   └── __init__.py
│
├── docs/                       # Documentation (9 comprehensive guides)
│   ├── adding_providers.md     # How to add new AI providers
│   ├── adding_tools.md         # How to create new tools
│   ├── advanced-usage.md       # Advanced features & configuration
│   ├── context-revival.md      # Context revival system explained
│   ├── contributions.md        # Contributing guidelines
│   ├── custom_models.md        # Custom model configuration
│   ├── logging.md              # Logging system documentation
│   ├── testing.md              # Testing guide
│   └── troubleshooting.md      # Common issues & solutions
│
├── conf/                       # Configuration files
│   └── custom_models.json      # OpenRouter/custom model definitions
│
├── scripts/                    # Utility scripts
├── examples/                   # Example usage scenarios
│
├── .github/                    # GitHub Actions CI/CD
│   └── workflows/
│       ├── test.yml            # Unit test runner
│       ├── docker-test.yml     # Integration test runner
│       ├── build_and_publish_docker.yml  # Docker registry publishing
│       └── auto-version.yml    # Semantic versioning automation
│
├── run-server.sh              # Docker deployment script (17K)
├── code_quality_checks.sh     # Quality assurance script
├── communication_simulator_test.py  # Integration test runner
├── log_monitor.py             # Real-time log monitoring
├── docker-compose.yml         # Docker services configuration
├── Dockerfile                 # Container definition
├── requirements.txt           # Python dependencies
├── pyproject.toml            # Project metadata & tool config
├── pytest.ini                # pytest configuration
├── README.md                 # User-facing documentation (44K)
├── CLAUDE.md                 # This file - Developer guide
├── UPDATES.md                # Project status report
└── LICENSE                   # Apache 2.0 license
```

---

## Key Conventions & Patterns

### Coding Standards

#### 1. **Type Safety with Pydantic**
All tool requests and responses use Pydantic models for validation:

```python
from pydantic import BaseModel, Field

class MyToolRequest(ToolRequest):
    """Extend ToolRequest for tool-specific parameters."""
    prompt: str = Field(..., description="User's question or request")
    files: Optional[list[str]] = Field(None, description="File paths")
```

#### 2. **Naming Conventions**

- **Classes:** PascalCase (e.g., `ChatTool`, `GeminiProvider`)
- **Functions/Methods:** snake_case (e.g., `execute_tool`, `validate_request`)
- **Constants:** UPPER_SNAKE_CASE (e.g., `DEFAULT_MODEL`, `MAX_TOKENS`)
- **Private Members:** Leading underscore (e.g., `_internal_helper`)

#### 3. **Error Handling Pattern**

All tools follow this error handling pattern:

```python
try:
    # Validate request
    request = self.validate_request(arguments)

    # Execute core logic
    result = self._execute_core_logic(request)

    # Return formatted response
    return self.format_response(result)

except FileNotFoundError as e:
    return self.format_error(f"File not found: {e}")
except ValueError as e:
    return self.format_error(f"Invalid input: {e}")
except Exception as e:
    logger.exception("Unexpected error")
    return self.format_error(f"Internal error: {e}")
```

#### 4. **Logging Standards**

Use structured logging with appropriate levels:

```python
import logging
logger = logging.getLogger(__name__)

# Debug - Detailed diagnostic information
logger.debug(f"Processing request with {len(files)} files")

# Info - General informational messages
logger.info(f"Tool {tool_name} executed successfully")

# Warning - Potential issues that don't prevent execution
logger.warning(f"File {path} not found, skipping")

# Error - Errors that prevent operation
logger.error(f"Failed to connect to provider: {error}")

# Critical - System-level failures
logger.critical(f"Redis connection lost, threading disabled")
```

#### 5. **Documentation Standards**

All modules, classes, and public functions require docstrings:

```python
"""
Module-level docstring explaining purpose and key components.

This module implements the XYZ functionality for...
"""

class MyClass:
    """
    Class-level docstring explaining responsibility.

    This class handles XYZ by doing ABC...
    """

    def my_method(self, param: str) -> str:
        """
        Method-level docstring with parameters and return value.

        Args:
            param: Description of parameter

        Returns:
            Description of return value

        Raises:
            ValueError: When parameter is invalid
        """
```

### Architectural Patterns

#### 1. **Tool Pattern - Template Method**

All tools inherit from `BaseTool` and implement required methods:

```python
class MyTool(BaseTool):
    """Tools must implement these abstract methods."""

    def get_name(self) -> str:
        """Return tool name for MCP registry."""
        return "my_tool"

    def get_description(self) -> str:
        """Return tool description for Claude."""
        return "Does something useful"

    def get_input_schema(self) -> dict:
        """Return JSON schema for tool parameters."""
        return MyToolRequest.model_json_schema()

    def execute(self, arguments: dict) -> list[TextContent]:
        """Execute tool logic and return formatted response."""
        # Implementation here
```

#### 2. **Provider Pattern - Strategy**

Providers implement a common interface, allowing runtime selection:

```python
class MyProvider(ModelProvider):
    """Providers must implement these methods."""

    def get_provider_type(self) -> ProviderType:
        """Return provider type identifier."""
        return ProviderType.CUSTOM

    def get_available_models(self) -> List[str]:
        """List models this provider supports."""
        return ["my-model-1", "my-model-2"]

    async def generate_content(self, request) -> str:
        """Generate content using provider's API."""
        # API call implementation
```

#### 3. **Registry Pattern - Service Locator**

Central registries manage tool and provider discovery:

```python
# Provider registry
registry = ModelProviderRegistry()
registry.register_provider(GeminiProvider())
registry.register_provider(OpenAIProvider())

# Get provider for model
provider = registry.get_provider_for_model("gemini-2.5-pro")

# Auto-select best model
model = registry.select_model_for_tool("chat", task_description)
```

#### 4. **Conversation Threading Pattern**

Redis-based conversation persistence with file deduplication:

```python
# Create new conversation thread
thread_id = create_thread()

# Add turn with file tracking
add_turn(
    thread_id=thread_id,
    role="user",
    content="analyze this code",
    files=["src/main.py"]
)

# Get thread with deduplication
thread = get_thread(thread_id)
# Returns only files not seen in previous turns
```

### Code Quality Requirements

#### Pre-Commit Checklist
Before any commit:
1. ✅ All code passes `ruff check . --fix`
2. ✅ All code formatted with `black .`
3. ✅ Imports sorted with `isort .`
4. ✅ All 361+ unit tests pass
5. ✅ No new linting warnings introduced
6. ✅ Type hints added for new functions
7. ✅ Docstrings added for new classes/methods

#### Testing Requirements
- **Unit tests required** for all new tools and providers
- **Integration tests required** for new conversation features
- **Minimum 80% code coverage** for new code
- **All tests must pass** before merge

---

## Development Workflows

### Setting Up Development Environment

```bash
# 1. Clone repository
git clone https://github.com/BeehiveInnovations/zen-mcp-server.git
cd zen-mcp-server

# 2. Create virtual environment
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Copy environment configuration
cp .env.example .env
nano .env  # Add your API keys

# 5. Start Docker services
./run-server.sh

# 6. Verify installation
python -m pytest tests/ -v
```

### Standard Development Cycle

#### Before Making Changes
```bash
# 1. Activate virtual environment
source venv/bin/activate

# 2. Pull latest changes
git pull origin main

# 3. Run quality checks to ensure clean baseline
./code_quality_checks.sh

# 4. Start server in development mode
./run-server.sh
```

#### While Making Changes
```bash
# 1. Make code changes in your editor

# 2. Run affected unit tests frequently
python -m pytest tests/test_my_feature.py -v

# 3. Check logs for errors
docker exec zen-mcp-server tail -f /tmp/mcp_server.log

# 4. Test interactively with Claude CLI
# (Make changes to code, restart server, test with Claude)
```

#### After Making Changes
```bash
# 1. Run quality checks (auto-fixes issues)
./code_quality_checks.sh

# 2. Run relevant integration tests
python communication_simulator_test.py --individual <test_name>

# 3. Verify server logs show no errors
docker exec zen-mcp-server tail -n 100 /tmp/mcp_server.log

# 4. Update documentation if needed
# Edit README.md, CLAUDE.md, or docs/* as appropriate
```

#### Before Committing
```bash
# 1. Final quality check - MUST PASS 100%
./code_quality_checks.sh

# 2. Run all integration tests (if time permits)
python communication_simulator_test.py

# 3. Update version in config.py if needed
# Only for release commits, follow semantic versioning

# 4. Update UPDATES.md with changes
# Document what changed and why

# 5. Commit with descriptive message
git add .
git commit -m "feat: Add new feature XYZ

- Implemented ABC
- Fixed DEF
- Updated documentation

Closes #123"

# 6. Push to branch (not main directly)
git push origin feature/my-feature
```

### Creating a New Branch

```bash
# 1. Ensure main is up to date
git checkout main
git pull origin main

# 2. Create feature branch
git checkout -b feature/descriptive-name
# OR for bug fixes:
git checkout -b fix/bug-description

# 3. Make changes and commit
# Follow standard development cycle above

# 4. Push to remote
git push -u origin feature/descriptive-name

# 5. Create pull request on GitHub
# Ensure all CI checks pass before requesting review
```

---

## Quick Reference Commands

### Code Quality Checks

**CRITICAL: Always run before commits!**

```bash
# Activate virtual environment first
source venv/bin/activate

# Run all quality checks (linting, formatting, tests)
./code_quality_checks.sh
```

This script automatically runs:
- Ruff linting with auto-fix
- Black code formatting
- Import sorting with isort
- Complete unit test suite (361 tests)
- Verification that all checks pass 100%

### Server Management

#### Start/Restart the Server
```bash
# Start or restart the Docker containers
./run-server.sh
```

This script will:
- Build/rebuild Docker images if needed
- Start the MCP server container (`zen-mcp-server`)
- Start the Redis container (`zen-mcp-redis`)
- Set up proper networking and volumes

#### Check Server Status
```bash
# Check if containers are running
docker ps

# Look for these containers:
# - zen-mcp-server (MCP server)
# - zen-mcp-redis (Redis for threading)

# Check container health
docker stats zen-mcp-server

# View container details
docker inspect zen-mcp-server
```

### Log Management

#### View Server Logs
```bash
# View last 500 lines of server logs
docker exec zen-mcp-server tail -n 500 /tmp/mcp_server.log

# Follow logs in real-time
docker exec zen-mcp-server tail -f /tmp/mcp_server.log

# View specific number of lines
docker exec zen-mcp-server tail -n 100 /tmp/mcp_server.log

# Search logs for specific patterns
docker exec zen-mcp-server grep "ERROR" /tmp/mcp_server.log
docker exec zen-mcp-server grep "tool_name" /tmp/mcp_server.log
docker exec zen-mcp-server grep "continuation_id" /tmp/mcp_server.log
```

#### Monitor Tool Executions
```bash
# View tool activity log (focused on tool calls)
docker exec zen-mcp-server tail -n 100 /tmp/mcp_activity.log

# Follow tool activity in real-time
docker exec zen-mcp-server tail -f /tmp/mcp_activity.log

# Use the dedicated log monitor
python log_monitor.py
```

The `log_monitor.py` script provides real-time view of:
- Tool calls and completions
- Conversation resumptions and context
- Errors and warnings from all log files
- File rotation handling

#### All Available Log Files
```bash
# Main server log (all activity)
docker exec zen-mcp-server tail -f /tmp/mcp_server.log

# Tool activity only (TOOL_CALL, TOOL_COMPLETED, etc.)
docker exec zen-mcp-server tail -f /tmp/mcp_activity.log

# Debug information (provider details)
docker exec zen-mcp-server tail -f /tmp/gemini_debug.log

# Overflow logs (when main log gets too large)
docker exec zen-mcp-server tail -f /tmp/mcp_server_overflow.log
```

#### Debug Container Issues
```bash
# Check container logs (Docker level)
docker logs zen-mcp-server
docker logs zen-mcp-redis

# Execute interactive shell in container
docker exec -it zen-mcp-server /bin/bash

# Inside container, you can:
# - Check Python processes: ps aux | grep python
# - Test Redis: redis-cli -h redis ping
# - View environment: env | grep API
# - Check file permissions: ls -la /workspace
```

---

## Testing Guide

### Unit Tests (361+ Tests)

#### Run All Unit Tests
```bash
# Activate virtual environment
source venv/bin/activate

# Run all unit tests with verbose output
python -m pytest tests/ -v

# Run with coverage report
python -m pytest tests/ --cov=. --cov-report=html
# View coverage report: open htmlcov/index.html
```

#### Run Specific Tests
```bash
# Run specific test file
python -m pytest tests/test_refactor.py -v

# Run specific test class
python -m pytest tests/test_refactor.py::TestRefactorTool -v

# Run specific test function
python -m pytest tests/test_refactor.py::TestRefactorTool::test_format_response -v

# Run tests matching pattern
python -m pytest tests/ -k "conversation" -v

# Run tests with specific marker
python -m pytest tests/ -m "integration" -v
```

#### Debug Failing Tests
```bash
# Run with detailed output and stop on first failure
python -m pytest tests/ -vv -x

# Run with Python debugger on failure
python -m pytest tests/ --pdb

# Show print statements
python -m pytest tests/ -s
```

### Integration Tests (24 Scenarios)

**IMPORTANT:** Integration tests use live Docker containers and require API keys configured in `.env`.

#### List Available Tests
```bash
python communication_simulator_test.py --list-tests
```

#### Run All Integration Tests
```bash
# Run complete test suite
python communication_simulator_test.py

# Run with verbose output
python communication_simulator_test.py --verbose

# Force rebuild environment before testing
python communication_simulator_test.py --rebuild
```

#### Run Individual Tests (Recommended)
```bash
# RECOMMENDED: Run tests individually for better isolation

# Basic functionality tests
python communication_simulator_test.py --individual basic_conversation
python communication_simulator_test.py --individual content_validation
python communication_simulator_test.py --individual logs_validation
python communication_simulator_test.py --individual redis_validation

# Conversation threading tests
python communication_simulator_test.py --individual cross_tool_continuation
python communication_simulator_test.py --individual cross_tool_comprehensive
python communication_simulator_test.py --individual conversation_chain_validation
python communication_simulator_test.py --individual per_tool_deduplication

# Model-specific tests
python communication_simulator_test.py --individual model_thinking_config
python communication_simulator_test.py --individual o3_model_selection
python communication_simulator_test.py --individual o3_pro_expensive
python communication_simulator_test.py --individual openrouter_fallback
python communication_simulator_test.py --individual openrouter_models
python communication_simulator_test.py --individual ollama_custom_url
python communication_simulator_test.py --individual xai_models

# Tool-specific tests
python communication_simulator_test.py --individual testgen_validation
python communication_simulator_test.py --individual refactor_validation
python communication_simulator_test.py --individual line_number_validation

# Advanced feature tests
python communication_simulator_test.py --individual token_allocation_validation
python communication_simulator_test.py --individual vision_capability

# Run with verbose for debugging
python communication_simulator_test.py --individual logs_validation --verbose
```

#### Run Multiple Specific Tests
```bash
# Run multiple tests in sequence
python communication_simulator_test.py --tests basic_conversation content_validation redis_validation
```

#### Debug Integration Tests
```bash
# Keep containers running after test for manual inspection
python communication_simulator_test.py --keep-logs

# Then manually inspect:
docker logs zen-mcp-server
docker exec zen-mcp-server cat /tmp/mcp_server.log
```

### Test Organization

**Unit Tests (`tests/`):**
- Fast, isolated, no external dependencies
- Mock providers and Redis connections
- Test individual components in isolation
- Run frequently during development

**Integration Tests (`simulator_tests/`):**
- Realistic scenarios with live Docker containers
- Real API calls (requires API keys)
- Test end-to-end workflows
- Run before commits and in CI/CD

---

## Component Deep Dives

### 1. MCP Server (`server.py`)

**Responsibility:** Handle MCP protocol communication, route requests to tools, format responses.

**Key Functions:**
- `list_tools()` - Returns tool registry for Claude to discover
- `list_prompts()` - Returns structured prompt support (/zen:tool:model)
- `get_prompt()` - Handles structured prompt requests
- `call_tool()` - Routes tool execution requests

**Important Notes:**
- Uses stdio transport (reads from stdin, writes to stdout)
- All logging goes to stderr and `/tmp/mcp_server.log`
- Handles graceful shutdown on SIGTERM/SIGINT
- Implements rotating log files to prevent disk fill

### 2. Tool Base Class (`tools/base.py`)

**Critical File - Read This First!**

This 89K file is the heart of the tool system. It provides:

**Core Functionality:**
- Request validation with Pydantic
- File path security and translation
- Provider selection and model routing
- Conversation threading integration
- Token limit management
- Response formatting

**Key Methods:**
```python
# Must be implemented by tools
get_name() -> str
get_description() -> str
get_input_schema() -> dict
execute(arguments: dict) -> list[TextContent]

# Provided by base class
validate_request(arguments) -> ToolRequest
format_response(content: str) -> list[TextContent]
format_error(error: str) -> list[TextContent]
get_provider(model: str) -> ModelProvider
```

**Conversation Threading:**
```python
# Create new thread
thread_id = create_thread()

# Add turn
add_turn(thread_id, "user", "prompt", files=["file.py"])

# Get thread (with deduplication)
thread = get_thread(thread_id)
```

### 3. Provider Registry (`providers/registry.py`)

**Responsibility:** Route model requests to appropriate providers, handle auto-selection.

**Auto Mode Logic:**
```python
def select_model_for_tool(tool_name: str, context: str) -> str:
    """
    Claude sees MODEL_CAPABILITIES_DESC in tool schema.
    When DEFAULT_MODEL=auto, Claude picks model based on:
    - Context window needs
    - Reasoning complexity
    - Speed requirements
    - Cost considerations
    """
```

**Provider Priority:**
1. Check if model explicitly requested
2. Try native provider (Gemini, OpenAI, X.AI)
3. Try OpenRouter if configured
4. Try custom endpoint if configured
5. Raise error if no provider found

### 4. Conversation Memory (`utils/conversation_memory.py`)

**Redis-Based Threading System:**

```python
# Thread structure in Redis
{
    "thread_id": "uuid-string",
    "created_at": 1234567890,
    "turns": [
        {
            "role": "user",
            "content": "analyze this",
            "files": ["src/main.py"],
            "timestamp": 1234567890
        },
        {
            "role": "assistant",
            "content": "analysis results...",
            "timestamp": 1234567891
        }
    ],
    "file_history": ["src/main.py"],  # Deduplicated files
    "tool_name": "analyze"
}
```

**Key Features:**
- 3-hour automatic expiration
- Up to 10 turns per thread
- Cross-tool continuation support
- Automatic file deduplication

### 5. File Utilities (`utils/file_utils.py`)

**Path Translation:**
```python
# Claude sends: /Users/john/project/src/main.py
# Docker sees: /workspace/src/main.py
translate_path_for_environment(path) -> str
```

**File Reading:**
```python
# Secure file reading with validation
read_file_content(path: str) -> str
read_files(paths: List[str]) -> List[dict]
```

**Security Features:**
- Path traversal prevention
- Symlink protection
- Permission validation
- Size limit enforcement

---

## Adding New Features

### Adding a New Tool

**See:** `docs/adding_tools.md` for complete guide.

**Quick Steps:**

1. **Create tool file:** `tools/my_tool.py`

```python
from tools.base import BaseTool, ToolRequest
from pydantic import Field

class MyToolRequest(ToolRequest):
    prompt: str = Field(..., description="User's request")
    files: Optional[list[str]] = Field(None, description="Files to process")

class MyTool(BaseTool):
    def get_name(self) -> str:
        return "my_tool"

    def get_description(self) -> str:
        return "Does something useful for development"

    def get_input_schema(self) -> dict:
        return MyToolRequest.model_json_schema()

    def execute(self, arguments: dict) -> list[TextContent]:
        request = MyToolRequest(**arguments)
        # Implementation here
        return self.format_response("result")
```

2. **Create system prompt:** `systemprompts/my_tool_prompt.py`

3. **Register tool:** Add to `tools/__init__.py`

4. **Add unit tests:** `tests/test_my_tool.py`

5. **Add integration test:** `simulator_tests/test_my_tool_validation.py`

6. **Update documentation:** Add to README.md and docs/

### Adding a New Provider

**See:** `docs/adding_providers.md` for complete guide.

**Quick Steps:**

1. **Create provider file:** `providers/my_provider.py`

```python
from providers.base import ModelProvider, ProviderType

class MyProvider(ModelProvider):
    def get_provider_type(self) -> ProviderType:
        return ProviderType.CUSTOM

    def get_available_models(self) -> List[str]:
        return ["my-model-1", "my-model-2"]

    async def generate_content(self, request) -> str:
        # API call implementation
        pass
```

2. **Register provider:** Add to `providers/__init__.py` and `providers/registry.py`

3. **Add configuration:** Update `.env.example` with new API key

4. **Add tests:** `tests/test_my_provider.py`

5. **Update documentation:** Add to README.md and docs/custom_models.md

### Modifying System Prompts

System prompts are critical for tool behavior. When modifying:

1. **Understand current behavior** - Test extensively before changes
2. **Make incremental changes** - One improvement at a time
3. **Test with multiple models** - Gemini and O3 may respond differently
4. **Validate edge cases** - Empty inputs, large files, errors
5. **Update tests** - Ensure prompt changes don't break tests

**Best Practices:**
- Be explicit about output format requirements
- Include examples of desired behavior
- Specify what NOT to do
- Use numbered lists for multi-step instructions
- Test token consumption impact

---

## Common Troubleshooting

### Container Issues

#### Server Won't Start
```bash
# Check if containers exist
docker ps -a

# Remove old containers and rebuild
docker rm -f zen-mcp-server zen-mcp-redis
./run-server.sh

# Check for port conflicts
lsof -i :6379  # Redis port
```

#### Server Crashes
```bash
# Check container logs
docker logs zen-mcp-server

# Common issues:
# - Missing API keys (check .env file)
# - Redis connection failed (check zen-mcp-redis running)
# - Out of memory (check docker stats)

# Restart containers
docker restart zen-mcp-server zen-mcp-redis
```

#### Permission Errors
```bash
# Check volume mounts
docker inspect zen-mcp-server | grep Mounts -A 20

# Verify WORKSPACE_ROOT in .env
echo $WORKSPACE_ROOT

# Fix permissions (if needed)
chmod -R 755 /path/to/workspace
```

### Test Failures

#### Unit Tests Failing
```bash
# Run specific failing test with verbose
python -m pytest tests/test_failing.py -vv

# Common issues:
# - Redis mock not properly configured (check conftest.py)
# - Provider mock missing methods
# - File paths incorrect in tests

# Check if virtual environment activated
which python  # Should show venv/bin/python
```

#### Integration Tests Failing
```bash
# Check API keys configured
cat .env | grep API_KEY

# Rebuild containers
python communication_simulator_test.py --rebuild

# Run with verbose to see API responses
python communication_simulator_test.py --individual failing_test --verbose

# Common issues:
# - API rate limits exceeded
# - Invalid API keys
# - Network connectivity issues
# - Docker containers not running
```

### Linting Issues

```bash
# Auto-fix most issues
ruff check . --fix
black .
isort .

# Check what would change without applying
ruff check .
black --check .
isort --check-only .

# Common issues:
# - Line too long (break into multiple lines)
# - Unused imports (remove them)
# - Missing type hints (add them)
# - Import ordering (isort fixes automatically)
```

### Redis Connection Issues

```bash
# Check Redis container running
docker ps | grep redis

# Test Redis connectivity from server container
docker exec zen-mcp-server redis-cli -h redis ping
# Should return: PONG

# Check Redis data
docker exec -it zen-mcp-redis redis-cli
> KEYS *
> GET conversation:thread_id

# Clear Redis (careful!)
docker exec zen-mcp-redis redis-cli FLUSHALL
```

### File Path Issues

```bash
# Common problem: Claude sees /Users/john/project
# Docker sees: /workspace

# Check WORKSPACE_ROOT in .env
docker exec zen-mcp-server env | grep WORKSPACE

# Test path translation
docker exec zen-mcp-server python -c "
from utils.file_utils import translate_path_for_environment
print(translate_path_for_environment('/Users/john/project/file.py'))
"
```

---

## Recent Updates

### Version 4.8.3 (Current)
**Released:** 2025-06-16

**Changes:**
- ✅ Fixed O3-Pro connection issues (#56)
- ✅ Added dedicated O3-Pro tests
- ✅ Improved shorthand input prompt handling
- ✅ Enhanced error messages for model validation

**Impact:** O3-Pro model now works reliably, better prompts for natural language model selection.

### Version 4.8.2

**Changes:**
- ✅ Fixed Gemini parameter order regression (#62, PR #60)
- ✅ Added regression tests for model restrictions
- ✅ Improved Google model validation logic consistency

**Impact:** GOOGLE_ALLOWED_MODELS restriction now works correctly, preventing future regressions.

### Version 4.8.1

**Changes:**
- ✅ New `listmodels` tool for model discovery
- ✅ Optimized OpenRouter registry loading with class-level caching
- ✅ Integration tests for all API key combinations
- ✅ Fixed missing 'low' severity in codereview
- ✅ Schema now lists all models including locally available ones

**Impact:** Better visibility into available models, improved performance for OpenRouter users.

### Version 4.8.0

**Major Features:**
- ✅ Complete image/vision support integration
- ✅ Prompt shorthand support (/zen:tool:model syntax)
- ✅ Enhanced advanced usage documentation
- ✅ Redis mocking improvements in tests

**Impact:** Can now analyze images, screenshots, diagrams. Simplified tool invocation syntax.

### Previous Milestones

- **v4.7.x** - Context revival system, cross-tool continuation
- **v4.6.x** - Web search integration, dynamic collaboration
- **v4.5.x** - X.AI GROK integration, thinking modes
- **v4.0.x** - Multi-model orchestration, auto mode
- **v3.x** - OpenRouter and custom endpoint support
- **v2.x** - OpenAI O3 integration
- **v1.x** - Initial release with Gemini support

---

## Additional Resources

### Documentation Files

1. **README.md** - User-facing documentation, quickstart, tool descriptions
2. **UPDATES.md** - Project status report, what's done, what's next
3. **docs/advanced-usage.md** - Advanced features, configuration, workflows
4. **docs/adding_tools.md** - Step-by-step tool development guide
5. **docs/adding_providers.md** - Step-by-step provider integration guide
6. **docs/context-revival.md** - Technical deep-dive on conversation system
7. **docs/contributions.md** - Contributing guidelines, PR process
8. **docs/custom_models.md** - Custom model configuration guide
9. **docs/testing.md** - Testing strategies and best practices
10. **docs/troubleshooting.md** - Common issues and solutions

### External Links

- **GitHub Repository:** https://github.com/BeehiveInnovations/zen-mcp-server
- **Issues:** https://github.com/BeehiveInnovations/zen-mcp-server/issues
- **MCP Protocol:** https://modelcontextprotocol.com
- **Gemini API:** https://ai.google.dev/
- **OpenAI API:** https://platform.openai.com/

### Getting Help

1. **Check documentation** - Start with README.md and this file
2. **Search issues** - Someone may have had the same problem
3. **Check logs** - Most issues show up in `/tmp/mcp_server.log`
4. **Run quality checks** - `./code_quality_checks.sh` catches many issues
5. **Create issue** - If stuck, create detailed GitHub issue with logs

---

## Environment Requirements

### System Requirements

- **Python:** 3.8 or higher
- **Docker:** 20.10 or higher
- **Docker Compose:** 2.0 or higher (usually included with Docker Desktop)
- **Git:** Any recent version
- **Memory:** 2GB RAM minimum (4GB recommended)
- **Disk:** 2GB free space for Docker images

### Python Dependencies

See `requirements.txt` for complete list:

```
mcp>=1.0.0                # MCP protocol implementation
google-genai>=1.19.0      # Gemini API
openai>=1.0.0             # OpenAI API
pydantic>=2.0.0           # Data validation
redis>=5.0.0              # Conversation persistence

# Development dependencies
pytest>=7.4.0
pytest-asyncio>=0.21.0
pytest-mock>=3.11.0
```

### API Keys

At least one of the following is required:

- **GEMINI_API_KEY** - For Gemini Pro/Flash models
- **OPENAI_API_KEY** - For O3/O4 models
- **XAI_API_KEY** - For GROK models
- **OPENROUTER_API_KEY** - For access to 100+ models
- **CUSTOM_API_URL** - For Ollama, vLLM, or custom endpoints

---

## File Structure Reference

**Critical Files:**
- `server.py` - MCP server entry point
- `config.py` - Version and configuration constants
- `tools/base.py` - Tool base class (READ THIS FIRST!)
- `providers/registry.py` - Provider routing logic
- `utils/conversation_memory.py` - Threading system

**Configuration:**
- `.env` - Environment variables (API keys, settings)
- `conf/custom_models.json` - Custom model definitions
- `pyproject.toml` - Project metadata
- `pytest.ini` - Test configuration

**Scripts:**
- `./code_quality_checks.sh` - Quality assurance (REQUIRED before commits)
- `./run-server.sh` - Docker deployment
- `communication_simulator_test.py` - Integration test runner
- `log_monitor.py` - Real-time log monitoring

---

## Summary

This guide provides everything needed for AI assistants to effectively work on the Zen MCP Server codebase. Key takeaways:

✅ **Always run quality checks** - `./code_quality_checks.sh` before commits
✅ **Follow patterns** - Use BaseTool, ModelProvider abstractions
✅ **Test thoroughly** - Unit tests + integration tests required
✅ **Document changes** - Update README, CLAUDE.md, UPDATES.md
✅ **Use type hints** - Pydantic models for all requests/responses
✅ **Log appropriately** - Use structured logging with correct levels
✅ **Handle errors gracefully** - Comprehensive error handling required

**When in doubt:**
1. Check existing tool implementations for patterns
2. Review documentation in `docs/`
3. Run tests to validate changes
4. Check logs for debugging

**Remember:** This is a production system with real users. Code quality, testing, and documentation are not optional.

---

**Guide End - Happy Coding! 🚀**
