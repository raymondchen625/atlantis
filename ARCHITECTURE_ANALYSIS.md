# Atlantis Project Architecture Analysis

## Project Overview

**Atlantis** is a self-hosted Terraform Pull Request Automation tool written in Go. It listens for Terraform pull request events via webhooks from various VCS providers and runs `terraform plan`, `import`, and `apply` commands remotely, commenting back on pull requests with the output.

- **License**: Apache 2.0
- **Primary Language**: Go 1.25.3
- **Repository**: github.com/runatlantis/atlantis
- **Supported Terraform Distributions**: Terraform and OpenTofu

---

## Architecture Overview

### High-Level Component Flow

```
1. VCS Webhook → Webhook Handler
2. Event Parser → Command Router
3. Project Finder → Workflow Engine
4. Runtime Executor → Result Renderer
5. VCS Client → PR Comment
```

### Architectural Patterns

- **Event-Driven Architecture**: Webhook-based event processing
- **Layered Architecture**: Clear separation between controllers, events, core logic
- **Plugin Architecture**: Support for multiple VCS providers through common interfaces
- **Command Pattern**: Commands (plan, apply, unlock, etc.) are processed through command runners
- **Strategy Pattern**: Different workflows and step runners based on configuration

---

## Core Technologies and Frameworks

### Backend (Go)

#### Web Server & Routing
- **gorilla/mux** (v1.8.1) - HTTP router and URL matcher
- **urfave/negroni** (v3.1.1) - Idiomatic HTTP middleware
- **gorilla/websocket** (v1.5.3) - WebSocket implementation for real-time updates

#### Command-Line Interface
- **spf13/cobra** (v1.9.1) - CLI framework
- **spf13/viper** (v1.20.1) - Configuration management
- **spf13/pflag** (v1.0.10) - POSIX/GNU-style flags

#### VCS Provider Integrations
- **github.com/google/go-github/v71** (v71.0.0) - GitHub API client
- **github.com/shurcooL/githubv4** - GitHub GraphQL API
- **github.com/bradleyfalzon/ghinstallation/v2** (v2.15.0) - GitHub App authentication
- **gitlab.com/gitlab-org/api/client-go** (v0.118.0) - GitLab API client
- **code.gitea.io/sdk/gitea** (v0.21.0) - Gitea API client
- **github.com/drmaxgit/go-azuredevops** (v0.13.2) - Azure DevOps API client

#### Database & Locking
- **go.etcd.io/bbolt** (v1.4.3) - BoltDB embedded key-value database (default)
- **github.com/redis/go-redis/v9** (v9.7.3) - Redis client (alternative locking backend)
- **github.com/alicebob/miniredis/v2** (v2.34.0) - Redis mock for testing

#### Terraform Integration
- **github.com/hashicorp/terraform-config-inspect** - Parse Terraform configuration
- **github.com/hashicorp/go-version** (v1.7.0) - Version parsing and comparison
- **github.com/hashicorp/hc-install** (v0.9.2) - Terraform installer
- **github.com/hashicorp/go-getter/v2** (v2.2.3) - Download files from various sources
- **github.com/opentofu/tofudl** (v0.0.1) - OpenTofu downloader
- **github.com/hashicorp/hcl/v2** (v2.23.0) - HCL parser

#### Logging & Metrics
- **go.uber.org/zap** (v1.27.0) - Structured logging
- **github.com/uber-go/tally/v4** (v4.1.17) - Metrics collection
- **github.com/cactus/go-statsd-client/v5** (v5.1.0) - StatsD client

#### Template & Markdown Processing
- **github.com/Masterminds/sprig/v3** (v3.3.0) - Template functions
- **github.com/microcosm-cc/bluemonday** (v1.0.27) - HTML sanitization
- **github.com/mitchellh/colorstring** - Terminal color formatting

#### Security & Validation
- **github.com/go-playground/validator/v10** (v10.26.0) - Struct validation
- **github.com/golang-jwt/jwt/v5** (v5.2.2) - JWT implementation
- **github.com/slack-go/slack** (v0.16.0) - Slack notifications

#### Utilities
- **github.com/google/uuid** (v1.6.0) - UUID generation
- **github.com/pkg/errors** (v0.9.1) - Error handling with stack traces
- **github.com/mohae/deepcopy** - Deep copy utility
- **github.com/bmatcuk/doublestar/v4** (v4.8.1) - Glob pattern matching
- **github.com/moby/patternmatcher** (v0.6.0) - Docker-style pattern matching

#### Testing
- **github.com/stretchr/testify** (v1.10.0) - Testing toolkit
- **github.com/petergtz/pegomock/v4** (v4.2.0) - Mock generation
- **github.com/go-test/deep** (v1.1.1) - Deep comparison for testing

### Frontend (Documentation Website)

#### Framework & Build Tools
- **vitepress** (v1.6.4) - Vue-based static site generator
- **vite** (v6.4.1) - Build tool
- **vue** (v3.5.22) - Frontend framework

#### Documentation Tools
- **markdown-it-footnote** (v4.0.0) - Markdown footnotes
- **markdownlint-cli** (v0.45.0) - Markdown linter
- **mermaid** (v11.12.0) - Diagram rendering
- **vitepress-plugin-mermaid** (v2.0.17) - Mermaid plugin for VitePress

#### Testing
- **@playwright/test** (v1.56.1) - End-to-end testing framework
- **@types/node** (v22.13.4) - TypeScript definitions

---

## Build System and Tools

### Build Tools

#### Make
Primary build system with targets including:
```
make build              # Build the atlantis binary
make test              # Run unit tests
make test-all          # Run all tests including integration
make lint              # Run golangci-lint
make fmt               # Format code with goimports
make regen-mocks       # Regenerate pegomock mocks
make website:dev       # Run docs site locally
make end-to-end-tests  # Run e2e tests
```

#### Go Build Configuration
- **CGO**: Disabled (`CGO_ENABLED=0`)
- **Target OS**: Linux (GOOS=linux)
- **Target Architecture**: AMD64 (GOARCH=amd64)
- **Build Flags**: `-trimpath -ldflags "-s -w"` for smaller binaries

### Code Quality Tools

#### Linting (golangci-lint v1.64.4)
Enabled linters:
- **gochecknoinits** - Check for init functions
- **gosec** - Security checks
- **misspell** - Spelling errors
- **revive** - Go linter
- **testifylint** - Testify usage
- **unconvert** - Unnecessary type conversions
- **gofmt** - Code formatting

#### Formatting
- **goimports** - Automatic import organization and formatting

### Containerization

#### Docker Multi-Stage Build
**Base Image Options**:
- Alpine 3.21.3 (default, lightweight)
- Debian 12.12-slim (alternative)

**Build Stages**:
1. **Builder Stage**: Go 1.25.3-alpine
   - Compiles Atlantis binary
   - Downloads dependencies

2. **Dependencies Stage**:
   - Downloads Terraform versions (1.8.5, 1.9.8, 1.10.5, 1.13.4)
   - Downloads OpenTofu (1.10.6)
   - Installs conftest (0.63.0)
   - Installs git-lfs (3.7.1)

3. **Final Stage**: Alpine or Debian
   - Minimal runtime image
   - Non-root user (atlantis)
   - Health check on port 4141

**Included Tools**:
- Git (for repository operations)
- curl (for health checks and downloads)
- openssh (for SSH git operations)
- unzip (for extracting downloads)
- dumb-init (for proper signal handling)

---

## Key Package Structure

### `cmd/`
Command-line interface implementation
- `server.go` - Server command with extensive flag configuration
- `version.go` - Version information
- `testdrive.go` - Quick setup for testing

### `server/`
Core server implementation

#### `server/controllers/`
HTTP request handlers
- `events/` - Webhook endpoint controllers and validators
- `jobs_controller.go` - Background job status
- `locks_controller.go` - Lock management UI
- `status_controller.go` - Health checks and status
- `github_app_controller.go` - GitHub App setup
- `websocket/` - Real-time updates via WebSocket

#### `server/events/`
Event handling system (core business logic)
- **Command Processing**:
  - `plan_command_runner.go`
  - `apply_command_runner.go`
  - `import_command_runner.go`
  - `policy_check_command_runner.go`
  - `unlock_command_runner.go`

- **Event Parsing**:
  - `event_parser.go` - Parse webhook payloads
  - `comment_parser.go` - Parse PR comments for commands

- **Project Management**:
  - `project_command_builder.go` - Build project contexts
  - `project_command_runner.go` - Execute commands on projects
  - `project_finder.go` - Discover affected projects

- **Workflow Execution**:
  - `working_dir.go` - Manage working directories
  - `command_runner.go` - Orchestrate command execution
  - `pre_workflow_hooks_command_runner.go`
  - `post_workflow_hooks_command_runner.go`

- **VCS Integration** (`events/vcs/`):
  - `github_client.go`
  - `gitlab_client.go`
  - `azuredevops_client.go`
  - `bitbucketcloud/` and `bitbucketserver/`
  - `gitea/`
  - `client.go` - Common VCS interface

- **Models** (`events/models/`):
  - Core data structures (Pull, Repo, User, Project)
  - Command contexts and results

- **Templates** (`events/templates/`):
  - Markdown templates for PR comments

#### `server/core/`
Core business logic

- **`boltdb/`** - BoltDB implementation for persistence
- **`redis/`** - Redis implementation for distributed locking
- **`locking/`** - Distributed lock management to prevent concurrent operations
- **`config/`** - Configuration parsing and validation
  - `raw/` - Raw configuration structures
  - `valid/` - Validated configuration
  - `parser_validator.go` - Configuration validation logic

- **`runtime/`** - Terraform execution
  - `init_step_runner.go`
  - `plan_step_runner.go`
  - `apply_step_runner.go`
  - `import_step_runner.go`
  - `policy_check_step_runner.go`
  - `custom_step_runner.go`
  - `env_step_runner.go`
  - `executor.go` - Command execution orchestration
  - `cache/` - Terraform version and plugin caching

- **`terraform/`** - Terraform client wrapper

---

## Configuration System

### Two-Tier Configuration

1. **Repo-Level Config** (`atlantis.yaml` in repository root)
   - Defines projects within the repository
   - Custom workflows
   - Autoplan configuration
   - Project-specific settings

2. **Server-Side Config** (passed via `--repo-config` flag)
   - Global policies across all repositories
   - Allowed workflows
   - Repository-level restrictions
   - Workflow overrides

### Configuration Schema
Both configurations use YAML defined in:
- `server/core/config/raw/` - Raw structures
- `server/core/config/valid/` - Validated structures

---

## Workflow System

### Default Workflows
- **plan**: init → plan
- **apply**: init → apply

### Custom Workflows
Defined in `atlantis.yaml`:
```yaml
workflows:
  custom:
    plan:
      steps:
      - init
      - plan
      - show
    apply:
      steps:
      - apply
      - run: custom-script.sh
```

### Step Runners
Each workflow step is implemented as a `StepRunner`:
- Built-in steps: init, plan, apply, import, state_rm, policy_check
- Custom steps: run, env
- Each implements the `StepRunner` interface

### Autoplan System
- Triggered on PR open/update
- File pattern matching using Docker-ignore syntax
- Module dependency tracking
- Configurable via `--autoplan-file-list` flag

---

## Locking Strategy

### Lock Types

1. **Project Locks**
   - Prevent concurrent plan/apply on the same project
   - Stored with full context: repo, PR number, workspace, project name
   - Released on successful apply or manual unlock

2. **Global Apply Locks**
   - Optional feature to disable all applies across system
   - Useful for maintenance windows

### Lock Backends

1. **BoltDB** (default, `--locking-db-type=boltdb`)
   - Embedded key-value store
   - Single-instance deployments
   - File-based: `~/.atlantis/atlantis.db`

2. **Redis** (`--locking-db-type=redis`)
   - Distributed locking
   - Multi-instance deployments
   - Configurable TLS support

---

## Security Features

### Webhook Validation
- GitHub: HMAC signature validation
- GitLab: Secret token validation
- Bitbucket: Webhook secret validation
- Azure DevOps: Basic authentication
- Gitea: Secret validation

### Repository Allowlist
- Required flag: `--repo-allowlist`
- Pattern-based matching
- Prevents unauthorized repository access

### Authentication Methods
- **GitHub**: Personal access token or GitHub App
- **GitLab**: Personal access token
- **Bitbucket**: App password
- **Azure DevOps**: Personal access token
- **Gitea**: Token authentication

### Code Execution Safety
- Sandboxed working directories
- File path restrictions
- Command allowlist (`--allow-commands`)
- Team-based authorization (GitHub teams, GitLab groups)

---

## CI/CD Pipeline

### GitHub Actions Workflows

1. **test.yml** - Unit and integration tests
2. **lint.yml** - Code linting
3. **website.yml** - Documentation building and deployment
4. **atlantis-image.yml** - Docker image building
5. **release.yml** - Release automation
6. **codeql.yml** - Security scanning
7. **dependency-review.yml** - Dependency security checks
8. **renovate-config.yml** - Automated dependency updates

### Testing Strategy

#### Unit Tests
```bash
go test -short $(PKG)
```
- Co-located with source files
- Excludes integration tests with `-short` flag
- Uses testify for assertions
- Pegomock for mocking

#### Integration Tests
```bash
go test -timeout=300s $(PKG)
```
- Requires actual Terraform installation
- Tests end-to-end workflows
- Uses real VCS interactions (mocked)

#### Test Utilities
- `github.com/runatlantis/atlantis/testing` - Custom assertions
- Golden files for output comparison
- Mock HTTP servers for VCS testing

#### End-to-End Tests
- Playwright-based (configured but tests not found in current scan)
- Tests complete PR workflows
- Validates UI interactions

---

## Deployment Options

### Docker (Recommended)
```bash
docker run -p 4141:4141 runatlantis/atlantis server \
  --gh-user=myuser --gh-token=token \
  --repo-allowlist='github.com/myorg/*'
```

### Docker Compose
Includes `docker-compose.yml` for local development

### Binary
```bash
atlantis server \
  --gh-user=myuser \
  --gh-token=token \
  --repo-allowlist='github.com/myorg/*'
```

### Kubernetes
Community-maintained Helm charts available

---

## Performance Optimizations

### Parallel Execution
- `--parallel-plan` - Run plans in parallel
- `--parallel-apply` - Run applies in parallel
- `--parallel-pool-size` - Configure worker pool (default: 15)

### Caching
- Terraform plugin cache (`--use-tf-plugin-cache`)
- Version binary caching in `~/.atlantis/bin`
- Go module caching in Docker builds

### Shallow Clones
- `--checkout-depth` - Limit git clone depth for merge strategy
- `--skip-clone-no-changes` - Skip cloning if no projects changed

---

## Monitoring and Observability

### Metrics
- Uber-go/tally integration
- Prometheus export support
- StatsD support
- Configurable namespace: `--stats-namespace`

### Logging
- Structured logging with Zap
- Log levels: debug, info, warn, error
- Lowercase logging standard
- Context-aware logging (`ctx.Log`)

### Health Checks
- `/healthz` endpoint
- Readiness checks
- Docker healthcheck built-in

### Profiling
- Optional pprof endpoints (`--enable-profiling-api`)
- CPU and memory profiling
- Goroutine analysis

---

## Code Style and Standards

### Logging Guidelines
- All logs MUST be lowercase
- Quote string variables with `%q`
- Never use colons (reserved for error chains)
- Use appropriate levels (debug for developers, info for users)

### Error Handling
- Use `pkg/errors` for wrapping with context
- Always lowercase error messages
- Describe action being performed, not failure
- Example: "cloning repository" not "failed to clone repository"

### Commit Standards
- Conventional Commits format (fix:, feat:, chore:)
- Sign commits with `-s` (DCO requirement)
- Atomic, focused commits

### Testing Standards
- Co-locate tests with source
- Use `_test` package for external interface testing
- Golden files in `testdata/` directories
- Regenerate mocks with `go generate`

---

## External Integrations

### Policy Checks
- Open Policy Agent (Conftest) support
- Custom policy checking via `--enable-policy-checks`
- Policy check step in workflows

### Slack Notifications
- Optional Slack integration (`--slack-token`)
- Webhook-based notifications
- Custom notification templates

### Terraform Cloud/Enterprise
- Remote backend support
- TFE token management
- Local execution mode support

---

## Development Tools

### Mock Generation
```bash
make regen-mocks  # Regenerate all mocks
go generate ./...  # Generate from go:generate comments
```

### Local Development
```bash
go install                    # Install to $GOPATH/bin
atlantis server --config ...  # Run locally
```

### Testing
```bash
make test              # Unit tests
make test-all          # All tests
make test-coverage     # Coverage report
docker-compose up      # Local environment
```

---

## Key Design Decisions

### Why Go?
- Excellent concurrency support (goroutines)
- Strong standard library
- Fast compilation and execution
- Easy cross-platform deployment
- Native HTTP server support

### Why BoltDB by Default?
- Embedded database (no external dependencies)
- Simple single-instance deployment
- Sufficient for most use cases
- Low operational overhead

### Why Multiple VCS Providers?
- Enterprise diversity (different orgs use different VCS)
- No vendor lock-in
- Common interface abstracts differences

### Why Webhook-Based?
- Real-time PR updates
- No polling overhead
- Secure (validated signatures)
- Standard across all VCS providers

---

## Scalability Considerations

### Single Instance
- BoltDB for locking
- Sufficient for small/medium teams
- Simple deployment

### Multi-Instance
- Redis for distributed locking
- Load balancer for high availability
- Shared data directory or S3
- Stateless design enables horizontal scaling

### Resource Requirements
- Minimal: 512MB RAM, 1 CPU
- Scales with number of concurrent operations
- Disk space for cloned repositories
- Network bandwidth for git operations

---

## Extension Points

### Custom Step Runners
Implement `StepRunner` interface for custom workflow steps

### Custom Webhooks
HTTP webhooks for external integrations

### Custom Templates
Override markdown templates in `~/.markdown_templates`

### Custom Workflows
Define in `atlantis.yaml` or server-side config

### Pre/Post Workflow Hooks
Execute custom commands before/after workflows

---

## Related Documentation

- Official Docs: https://runatlantis.io
- GitHub: https://github.com/runatlantis/atlantis
- Docker Hub: https://hub.docker.com/r/runatlantis/atlantis

---

## Summary

Atlantis is a mature, well-architected Terraform automation platform built with Go. Its modular design, extensive VCS support, flexible workflow system, and containerized deployment make it suitable for teams of all sizes. The codebase demonstrates strong engineering practices including comprehensive testing, clear separation of concerns, and production-ready features like distributed locking and metrics.

**Key Strengths**:
- Multi-VCS provider support
- Flexible workflow system
- Strong security features
- Extensive testing
- Production-ready deployment options
- Active maintenance and community

**Ideal Use Cases**:
- Teams using Terraform across multiple repositories
- Organizations requiring PR-based Terraform workflows
- Multi-cloud infrastructure management
- Compliance-driven infrastructure changes
