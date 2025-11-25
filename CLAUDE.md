# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

YouTrack MCP is a Model Context Protocol (MCP) server that integrates JetBrains YouTrack with Claude Desktop and other MCP clients. It provides issue management, project management, search, and user management tools.

## Development Commands

### Formatting & Linting (Required Before Push)
```bash
isort . && black .           # Format code (required for PRs)
flake8                       # Lint check
mypy youtrack_mcp            # Type check (required for PRs)
```

### Testing
```bash
# Unit tests (fast, run frequently)
pytest tests/unit/ -v -m unit

# Integration tests
pytest tests/integration/ -v -m integration

# Run single test
pytest tests/unit/test_tools.py::TestToolLoading::test_tool_loading_basic -v

# Coverage report
pytest tests/ --cov=youtrack_mcp --cov-report=html

# Pre-build tests (code quality + unit + integration)
./scripts/test-before-build.sh
```

### Docker
```bash
docker build -t youtrack-mcp-local .
./scripts/test-after-build.sh youtrack-mcp-local
```

### Running the Server
```bash
# stdio mode (for Claude/Cursor integration)
python main.py --transport stdio

# HTTP mode (for API access)
python main.py --transport http
```

## Architecture

### Core Components

- **`main.py`**: Entry point with FastAPI HTTP server and CLI argument handling
- **`youtrack_mcp/server.py`**: `YouTrackMCPServer` class - MCP server implementation using FastMCP/ToolServerBase
- **`youtrack_mcp/config.py`**: Configuration from environment variables, supports both cloud and self-hosted YouTrack
- **`youtrack_mcp/tools/loader.py`**: Tool discovery and registration with priority-based duplicate resolution

### API Layer (`youtrack_mcp/api/`)

- **`client.py`**: Base `YouTrackClient` with retry logic, error handling, and session management
- **`issues.py`**: Issues API operations
- **`projects.py`**: Projects API operations
- **`search.py`**: Search API operations
- **`users.py`**: Users API operations

### Tools Layer (`youtrack_mcp/tools/`)

Tools are organized by domain and registered via `load_all_tools()`:

- **`issues/`**: Modular issue tools split into:
  - `basic_operations.py`: CRUD operations
  - `custom_fields.py`: Custom field management
  - `dedicated_updates.py`: State, priority, assignee, type updates
  - `linking.py`: Issue relationships
  - `diagnostics.py`: Workflow analysis
  - `attachments.py`: File operations
- **`projects.py`**: `ProjectTools` class
- **`users.py`**: `UserTools` class
- **`search.py`**: `SearchTools` class
- **`resources.py`**: `ResourcesTools` class (MCP resources)

### Tool Priority System

When the same tool name exists in multiple classes, `TOOL_PRIORITY` in `loader.py` determines which implementation to use. Higher numbers = higher priority.

### MCP Protocol Integration

The server supports two transport modes:
- **stdio**: For Claude Desktop/Cursor integration (default)
- **http**: REST API at port 8000

Tools are wrapped via `create_bound_tool()` in `mcp_wrappers.py` to handle MCP parameter formats and async execution.

## Environment Variables

```bash
YOUTRACK_URL=https://your-instance.youtrack.cloud
YOUTRACK_API_TOKEN=perm-...
YOUTRACK_VERIFY_SSL=true
YOUTRACK_CLOUD=false  # Set true for cloud-only instances
```

## Test Structure

```
tests/
├── unit/           # Fast, isolated tests (mock external dependencies)
├── integration/    # Component interaction tests (mocked APIs)
├── e2e/            # Real YouTrack instance tests (requires credentials)
└── docker/         # Container functionality tests
```

Use pytest markers: `@pytest.mark.unit`, `@pytest.mark.integration`, `@pytest.mark.e2e`, `@pytest.mark.docker`

## CI/CD Pipeline

The GitHub Actions workflow (`ci.yml`) has three stages:
1. **Testing**: Unit, integration, Docker tests run in parallel
2. **Dev Build**: Pushes `VERSION-wip` tags to Docker Hub + GHCR on main branch
3. **Production Release**: Manual trigger with version bump, creates `latest` tags and GitHub release

Production builds require `confirm_production: true` checkbox.

## Key Patterns

### Custom Field Updates
Use dedicated functions for common fields (they handle API complexities):
```python
update_issue_state("DEMO-123", "In Progress")
update_issue_priority("DEMO-123", "Critical")
update_issue_assignee("DEMO-123", "admin")
```

### Tool Registration
Tools are auto-discovered from `*Tools` classes. Each tool class should:
1. Initialize API clients in `__init__`
2. Expose methods that return JSON-serializable results
3. Optionally implement `get_tool_definitions()` for schema metadata
