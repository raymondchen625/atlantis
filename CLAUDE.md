# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Atlantis is a self-hosted Terraform Pull Request Automation tool written in Go. It listens for Terraform pull request events via webhooks from various VCS providers (GitHub, GitLab, Bitbucket, Azure DevOps, Gitea) and runs `terraform plan`, `import`, and `apply` commands remotely, commenting back on pull requests with the output.

## Development Commands

### Building
```bash
make build              # Build the atlantis binary
make build-service      # Same as make build
go install             # Compile and install to $GOPATH/bin
```

### Testing
```bash
make test              # Run unit tests (excludes integration tests with -short flag)
make test-all          # Run all tests including integration (requires actual terraform)
make test-coverage     # Generate test coverage report
make test-coverage-html # Generate and open HTML coverage report
```

Run tests in Docker:
```bash
make docker/test       # Run unit tests in Docker
make docker/test-all   # Run all tests in Docker
```

### Linting and Formatting
```bash
make lint              # Run golangci-lint locally
make check-lint        # Run linter in CI/CD mode (downloads specific version)
make check-fmt         # Check if code is formatted correctly
make fmt               # Format code with goimports
```

### Documentation Website
```bash
npm run website:dev    # Run docs site locally on http://localhost:8080
npm run website:build  # Build documentation site
npm run website:lint   # Lint markdown files
npm run website:lint-fix # Fix markdown linting issues
```

### E2E Tests
```bash
make end-to-end-deps   # Install Playwright e2e dependencies
make end-to-end-tests  # Run e2e tests
npm run e2e           # Run Playwright e2e tests directly
```

### Mocks
```bash
make go-generate       # Run go generate across all packages
make regen-mocks       # Delete and regenerate all pegomock mocks
go generate <file>     # Regenerate mocks for a specific file
```

### Running Locally
```bash
# Compile and install
go install

# Run server (requires GitHub token and webhook secret)
atlantis server --gh-user <username> --gh-token <token> \
  --repo-allowlist <repo> --gh-webhook-secret <secret> --log-level debug

# Or use Docker Compose (create atlantis.env first with required env vars)
make build-service
docker-compose up --detach
docker-compose logs --follow
```

## Architecture

### High-Level Component Flow
1. **Webhook Handler** (`server/controllers/events`) - Receives VCS webhooks
2. **Event Parser** - Parses webhook payloads into internal models
3. **Command Router** (`server/events/command`) - Routes commands (plan, apply, import, unlock, etc.)
4. **Project Finder** - Discovers which Terraform projects are affected
5. **Workflow Engine** (`server/events`) - Executes workflows defined in atlantis.yaml
6. **Runtime Executor** (`server/core/runtime`) - Runs Terraform commands
7. **Result Renderer** (`server/events/templates`) - Formats output as markdown
8. **VCS Client** (`server/events/vcs`) - Posts comments back to pull requests

### Key Packages

**`server/core/`** - Core business logic
- `boltdb/` - BoltDB database implementation for locks and state
- `redis/` - Redis implementation for locks (alternative to BoltDB)
- `config/` - Configuration parsing and validation (atlantis.yaml and server-side config)
- `locking/` - Distributed locking to prevent concurrent operations
- `runtime/` - Terraform command execution (init, plan, apply, import, state-rm)
- `terraform/` - Terraform client wrapper

**`server/events/`** - Event handling system
- `command/` - Command processing and routing
- `models/` - Core data models (Pull, Repo, User, ProjectCommandContext)
- `templates/` - Markdown rendering templates for PR comments
- `vcs/` - VCS provider integrations (GitHub, GitLab, Bitbucket, Azure DevOps, Gitea)
- `webhooks/` - Webhook parsing and validation

**`server/controllers/`** - HTTP request handlers
- `events/` - Main webhook endpoint controller
- `jobs/` - Background job status endpoints
- `locks/` - Lock management UI

**`cmd/`** - CLI commands (server, version, testdrive)

### Configuration System

Atlantis uses a two-tier configuration system:

1. **Repo-level config** (`atlantis.yaml` in repo root) - Defines projects, workflows, and autoplanning
2. **Server-side config** (passed via `--repo-config` flag) - Global policies, allowed workflows, repo settings

Both use the same YAML schema defined in `server/core/config/raw/` and validated in `server/core/config/valid/`.

### Workflow Execution

Workflows define a series of steps (init, plan, apply, etc.) that run for each project:
- Default workflows: `plan` and `apply`
- Custom workflows defined in atlantis.yaml
- Each step is a `StepRunner` interface implementation
- Steps can be custom commands (run steps) or built-in (init, plan, apply, policy_check, import, state_rm)

### VCS Provider Pattern

All VCS integrations implement the `Client` interface in `server/events/vcs/`. Each provider has its own package with:
- Authentication (token, GitHub App, OAuth)
- Webhook parsing
- API client for comments, status updates, file fetching
- Merge methods

### Locking Strategy

Atlantis uses distributed locking to prevent:
1. **Project locks** - Prevent concurrent plans/applies on same project
2. **Global apply locks** - Optional feature to disable all applies
3. **Backend locking** - BoltDB (single instance) or Redis (multi-instance)

Locks are stored with project context (repo, pull number, workspace, project name) and released on:
- Successful apply
- Manual unlock command
- PR closed/merged

## Testing Practices

### Test Organization
- Co-locate tests with source: `foo.go` → `foo_test.go`
- Use `{package}_test` for testing external interfaces
- Use `{file}_internal_test.go` for testing internal/private functions
- Test data in `testdata/` directories
- Golden files for comparing command outputs

### Testing Utilities
- Use `github.com/runatlantis/atlantis/testing` utilities: `Assert()`, `Equals()`, `Ok()`
- Mocking framework: `github.com/petergtz/pegomock/v4`
- Deep comparison: `github.com/go-test/deep`

### Mock Generation
Each mocked interface has a `//go:generate` comment above it:
```go
//go:generate pegomock generate -m --package mocks -o mocks/mock_foo.go Foo
```

To regenerate a specific mock:
```bash
go generate path/to/file.go
```

Or regenerate all mocks:
```bash
make regen-mocks
```

## Code Style Guidelines

### Logging
- Use `ctx.Log` (available in most methods)
- Levels: debug (developers), info (users), warn (possible issues), error (definite issues)
- All logs MUST be lowercase
- Quote string variables with `%q`: `ctx.Log.Info("cleaning dir %q", dir)`
- Never use colons `:` in logs (reserved for error chains)

### Errors
- Always use lowercase
- Use `errors.Wrap(err, "context")` instead of `fmt.Errorf`
- Describe what was occurring, NOT "failed to" or "unable to" or "error occurred when"
- Good: `"cloning repository"`, `"running git clone"`
- Bad: `"failed to clone repository"`, `"error occurred when running git clone"`

### Commit Messages
- Use Conventional Commits format: `fix:`, `feat:`, `chore:`, etc.
- Sign commits with `-s` flag (DCO requirement)
- Keep commits atomic and focused

## CI/CD

GitHub Actions workflows in `.github/workflows/`:
- `testing.yml` - Run tests on PRs
- `lint.yml` - Linter checks
- `website.yml` - Build and deploy docs
- `release.yml` - Create releases
- Renovate bot for automated dependency updates

## Important Constraints

- Support multiple VCS providers through common interface
- Maintain backward compatibility for atlantis.yaml format
- Thread-safe locking for concurrent webhook processing
- Security: validate webhooks, sanitize user inputs, restrict file access
- Both Terraform and OpenTofu are supported via `--default-tf-distribution` flag
