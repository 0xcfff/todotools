# Dev Container Configuration

This dev container provides a complete development environment for todotools.

## What's Included

- **Go 1.25**: Matching the project's go.mod requirement
- **GitHub CLI**: For GitHub integration
- **Zsh**: Enhanced shell with modern defaults
- **VS Code Extensions**:
  - Go language support with gopls
  - Task runner integration
  - GitLens for advanced Git features
  - Spell checker for documentation

## Features

- Auto-installs task runner on container creation
- Downloads Go dependencies automatically
- Mounts your local .gitconfig for consistent Git settings
- Pre-configured Go tooling (goimports, golangci-lint)
- Format on save enabled
- Organize imports on save

## Usage

1. Open this project in VS Code
2. Click "Reopen in Container" when prompted (or use Command Palette: "Dev Containers: Reopen in Container")
3. Wait for the container to build and initialize
4. Start developing!

## Building and Running

Use the task runner:
```bash
task build      # Build the project
task run        # Run the application
task clean      # Clean build artifacts
```

Or use Go directly:
```bash
go build -o bin/todo ./cmd/todo
go run ./cmd/todo
```
