# Agent Guidelines for todotools

This document provides guidelines for AI agents working on this Go project.

## Project Structure

This project follows the standard Go project layout:

```
todotools/
├── cmd/                    # Main applications for this project (multiple executables)
│   ├── todo/              # Main todo tool (root command)
│   │   └── main.go        # Entry point for todo executable
│   ├── tododiff/          # Diff tool executable
│   │   └── main.go        # Entry point for tododiff executable
│   ├── todopath/          # Path tool executable
│   │   └── main.go        # Entry point for todopath executable
│   ├── todomerge/         # Merge tool executable
│   │   └── main.go        # Entry point for todomerge executable
│   └── ...                # Additional tool executables as needed
├── internal/              # Private application and library code
│   ├── app/              # Application logic
│   ├── config/           # Configuration handling (Viper)
│   └── pkg/              # Internal packages
├── pkg/                   # Library code that's ok to use by external applications
│   ├── diff/             # Diff functionality
│   ├── merge/            # Merge functionality
│   └── sync/             # Sync functionality
├── api/                   # API definition files (OpenAPI/Swagger, Protocol Buffers)
├── web/                   # Web application specific components
├── configs/               # Configuration file templates or default configs
├── scripts/               # Scripts for build, install, analysis, etc.
├── build/                 # Packaging and CI
│   ├── ci/               # CI configurations
│   └── package/          # Cloud, container, OS package configs
├── deployments/           # IaaS, PaaS, orchestration deployment configs
├── test/                  # Additional external test apps and test data
├── docs/                  # Design and user documents
├── tools/                 # Supporting tools for this project
├── vendor/                # Application dependencies (managed by go mod vendor)
├── specs/                 # Specification documents
├── go.mod                 # Go module definition
├── go.sum                 # Go module checksums
├── Taskfile.yml          # go-task task definitions
├── .gitignore            # Git ignore rules
├── LICENSE               # Project license
├── README.md             # Project overview
└── AGENTS.md             # This file
```

## Testing Framework

### Ginkgo & Gomega

This project uses [Ginkgo](https://github.com/onsi/ginkgo) as the testing framework and [Gomega](https://github.com/onsi/gomega) as the matcher library.

#### Installation

```bash
go get -u github.com/onsi/ginkgo/v2/ginkgo
go get -u github.com/onsi/gomega/...
```

#### Test File Structure

- Test files should be named `*_test.go`
- Test suites should be organized in the **same folder** as the code they test, but use a **different package structure** (e.g., `package diff_test` for testing `package diff`)
  - This allows tests to exercise the public API of the package
  - Tests are co-located with the code for better organization
  - Example: `pkg/diff/diff.go` (package diff) has tests in `pkg/diff/diff_test.go` (package diff_test)
- Use `_suite_test.go` for suite setup in each package folder

#### Example Test Structure

```go
package diff_test

import (
    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    "testing"
)

func TestDiff(t *testing.T) {
    RegisterFailHandler(Fail)
    RunSpecs(t, "Diff Suite")
}

var _ = Describe("Diff", func() {
    Context("when comparing two files", func() {
        It("should detect added lines", func() {
            // Test implementation
        })
    })
})
```

#### Running Tests

```bash
# Run all tests
task test

# Run tests with coverage
task test:coverage

# Run specific package tests
ginkgo ./pkg/diff/...

# Run with verbose output
ginkgo -v ./...
```

## Task Runner

### go-task

This project uses [go-task](https://taskfile.dev/) for managing build tasks, testing, and tooling.

#### Installation

```bash
# On macOS
brew install go-task

# Or using go install
go install github.com/go-task/task/v3/cmd/task@latest
```

#### Common Tasks

```bash
task                    # List all available tasks
task build             # Build all executables (todo, tododiff, todopath, todomerge, etc.)
task build:todo        # Build specific tool
task build:tododiff    # Build tododiff tool
task test              # Run all tests
task test:coverage     # Run tests with coverage
task lint              # Run linters
task fmt               # Format code
task install           # Install dependencies and tools
task clean             # Clean build artifacts
```

#### Taskfile.yml Structure

The `Taskfile.yml` should include:
- Build tasks
- Test tasks (using Ginkgo)
- Linting tasks (golangci-lint)
- Installation tasks
- Dependency management tasks

## CLI Framework

### Cobra

This project uses [Cobra](https://github.com/spf13/cobra) for building CLI applications. Each tool is a separate executable.

#### Installation

```bash
go get -u github.com/spf13/cobra@latest
```

#### Structure

This is a **multi-executable project** with separate binaries:
- `cmd/todo/` - Main todo tool (root command)
- `cmd/tododiff/` - Standalone diff tool
- `cmd/todopath/` - Standalone path tool
- `cmd/todomerge/` - Standalone merge tool
- Each tool has its own `main.go` and Cobra command structure
- Shared command logic can be in `internal/app/commands/`

#### Example: Standalone Tool Structure (tododiff)

```go
// cmd/tododiff/main.go
package main

import (
    "fmt"
    "os"
    "github.com/spf13/cobra"
    "github.com/yourusername/todotools/internal/config"
    "github.com/yourusername/todotools/pkg/diff"
)

var rootCmd = &cobra.Command{
    Use:   "tododiff [file1] [file2]",
    Short: "Compare two todo.txt files",
    Long:  `Performs a diff operation between two todo.txt files.`,
    Args:  cobra.ExactArgs(2),
    RunE: func(cmd *cobra.Command, args []string) error {
        cfg, _ := config.Load()
        // Implementation using pkg/diff
        return diff.Compare(args[0], args[1], cfg)
    },
}

func init() {
    rootCmd.Flags().BoolP("color", "c", true, "Colorize output")
    rootCmd.Flags().StringP("format", "f", "unified", "Output format")
}

func main() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

#### Example: Root Tool Structure (todo)

The `todo` tool can serve as a unified entry point with subcommands:

```go
// cmd/todo/main.go - can have subcommands like: todo diff, todo merge, etc.
```

## Configuration Management

### Viper

This project uses [Viper](https://github.com/spf13/viper) for configuration and environment variable management.

#### Installation

```bash
go get -u github.com/spf13/viper@latest
```

#### Configuration Structure

- Configuration file: `configs/config.yaml` (or `.json`, `.toml`)
- Environment variables: prefixed with `TODOTOOLS_`
- Config struct: defined in `internal/config/config.go`

#### Example Configuration

```go
package config

import (
    "github.com/spf13/viper"
)

type Config struct {
    LogLevel    string `mapstructure:"log_level"`
    ColorOutput bool   `mapstructure:"color_output"`
    DefaultPath string `mapstructure:"default_path"`
}

func Load() (*Config, error) {
    // Set defaults
    viper.SetDefault("log_level", "info")
    viper.SetDefault("color_output", true)

    // Config file paths
    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath("./configs")
    viper.AddConfigPath("$HOME/.todotools")
    viper.AddConfigPath(".")

    // Environment variables
    viper.SetEnvPrefix("TODOTOOLS")
    viper.AutomaticEnv()

    // Read config
    if err := viper.ReadInConfig(); err != nil {
        if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
            return nil, err
        }
    }

    var config Config
    if err := viper.Unmarshal(&config); err != nil {
        return nil, err
    }

    return &config, nil
}
```

#### Configuration Precedence

1. Explicit flag values (highest priority)
2. Environment variables
3. Config file values
4. Default values (lowest priority)

#### Environment Variables

All config values can be overridden with environment variables:
```bash
export TODOTOOLS_LOG_LEVEL=debug
export TODOTOOLS_COLOR_OUTPUT=false
export TODOTOOLS_DEFAULT_PATH=/path/to/todo.txt
```

## Development Workflow

### Getting Started

```bash
# Clone the repository
git clone <repository-url>
cd todotools

# Install dependencies and tools
task install

# Run tests
task test

# Build all executables (todo, tododiff, todopath, todomerge, etc.)
task build

# Build specific tool
task build:tododiff

# Run a specific tool (after building)
./bin/tododiff --help
./bin/todo --help
```

### Adding New Features

1. Create feature branch from `main`
2. Add code in appropriate package under `internal/` or `pkg/`
3. Write Ginkgo tests
4. Update Cobra commands if CLI changes are needed
5. Update configuration if new config options are needed
6. Run `task test` and `task lint`
7. Update documentation
8. Create pull request

### Adding New Tools

When adding a new executable tool to the project:

1. Create a new directory under `cmd/` (e.g., `cmd/todosync/`)
2. Add `main.go` with minimal Cobra setup
3. Implement core logic in `pkg/` or `internal/app/`
4. Add corresponding tests in the same folder as the implementation
5. Update `Taskfile.yml` to add build task for the new tool (e.g., `build:todosync`)
6. Update this AGENTS.md file to list the new tool in the project structure
7. Document the new tool in README.md

### Code Style

- Follow standard Go conventions
- Use `gofmt` for formatting (run via `task fmt`)
- Use `golangci-lint` for linting (run via `task lint`)
- Write tests for all new functionality
- Keep functions small and focused
- Document exported functions and types

## Dependencies Management

- Use Go modules (`go.mod` and `go.sum`)
- Update dependencies regularly: `go get -u ./...`
- Vendor dependencies if needed: `go mod vendor`
- Clean up unused dependencies: `go mod tidy`

## Key Libraries

| Library | Purpose | Documentation |
|---------|---------|---------------|
| Cobra | CLI framework | https://github.com/spf13/cobra |
| Viper | Configuration management | https://github.com/spf13/viper |
| Ginkgo | Testing framework | https://github.com/onsi/ginkgo |
| Gomega | Matcher library | https://github.com/onsi/gomega |
| go-task | Task runner | https://taskfile.dev |

## AI Agent Guidelines

When working on this project:

1. **Always check existing structure** before creating new files/packages
2. **Follow the project layout** defined in this document
3. **Multi-executable project** - Each tool (todo, tododiff, todopath, todomerge) is a separate binary with its own `main.go` in `cmd/`
4. **Write tests** using Ginkgo for all new functionality
5. **Use Cobra** for all CLI command implementations
6. **Use Viper** for all configuration and environment variable handling
7. **Update Taskfile.yml** when adding new build/test/tool tasks
8. **Keep cmd/ minimal** - Each `main.go` should be minimal; main application logic goes in `internal/app/` or `pkg/`
9. **Use internal/** for code that shouldn't be imported by external projects
10. **Use pkg/** only for code that's safe for external use (shared between tools)
11. **Document all exported functions** and types with godoc comments
12. **Run tests before committing**: `task test`
13. **Format code before committing**: `task fmt`

## Useful Commands

```bash
# Build all executables
task build

# Build specific tool
go build -o bin/tododiff ./cmd/tododiff
task build:tododiff

# Run a specific tool
./bin/tododiff file1.txt file2.txt
./bin/todo --help

# Initialize a new Ginkgo test suite
cd pkg/diff
ginkgo bootstrap

# Generate a test file
ginkgo generate diff

# Run tests with watch mode
ginkgo watch -r

# Run tests with coverage report
ginkgo -r --cover --coverprofile=coverage.out
go tool cover -html=coverage.out

# Install all required tools
task install:tools

# Update dependencies
go get -u ./...
go mod tidy
```

## Notes

- This project follows semantic versioning
- Keep the README.md updated with user-facing changes
- Update specs/ directory with any specification changes
- Use conventional commits for commit messages
