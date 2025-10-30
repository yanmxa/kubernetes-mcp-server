# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Kubernetes MCP Server** - a Go-based native implementation of a Model Context Protocol (MCP) server that provides AI tools direct access to Kubernetes and OpenShift cluster operations without requiring external CLI dependencies like `kubectl` or `helm`.

Key architecture:
- **Go native implementation** - No external tool dependencies
- **MCP server** - Implements Model Context Protocol for AI tools
- **Multi-platform distribution** - Native binaries, npm packages, Python packages, and container images
- **Toolset-based architecture** - Modular tool groups (config, core, helm)

## Development Commands

### Build and Development
- `make build` - Build the project binary
- `make build-all-platforms` - Build for all supported platforms (Linux, macOS, Windows, x86_64, ARM64)
- `make clean` - Remove all build artifacts
- `make tidy` - Tidy up Go modules
- `make format` - Format the code

### Testing and Quality
- `make test` - Run all tests (`go test -count=1 -v ./...`)
- `make lint` - Run golangci-lint for code quality
- Individual test files can be run with standard Go test commands

### Inspection and Development
- `npx @modelcontextprotocol/inspector@latest $(pwd)/kubernetes-mcp-server` - Run with MCP inspector after building

## Architecture

### Core Packages Structure
- **`cmd/kubernetes-mcp-server/`** - Main entry point
- **`pkg/mcp/`** - MCP server implementation and configuration
- **`pkg/kubernetes/`** - Kubernetes client management and operations
- **`pkg/toolsets/`** - Modular toolset system (config, core, helm)
- **`pkg/api/`** - API interfaces and types
- **`pkg/config/`** - Configuration management
- **`pkg/helm/`** - Helm chart operations
- **`pkg/version/`** - Version information

### Key Components
1. **Manager** (`pkg/kubernetes/kubernetes.go`) - Central Kubernetes client manager
2. **MCP Configuration** (`pkg/mcp/mcp.go`) - Handles toolset registration and tool filtering
3. **Toolsets** (`pkg/toolsets/`) - Modular tool groups:
   - `config` - Kubernetes configuration management
   - `core` - Pod, resource, event, namespace operations
   - `helm` - Helm chart operations

### Publishing
- **NPM packages** - Multi-architecture with optional dependencies
- **Python packages** - Built with `uv`
- **Container images** - Multi-platform Docker builds

## Configuration Options

The server supports command-line configuration:
- `--port` - HTTP/SSE mode
- `--log-level` - Kubernetes-style logging levels (0-9)
- `--kubeconfig` - Custom kubeconfig path
- `--list-output` - Output format (yaml, table)
- `--read-only` - Prevent write operations
- `--disable-destructive` - Disable destructive operations
- `--toolsets` - Enable specific toolset groups

## Testing Strategy

- Comprehensive test suite with `*_test.go` files
- Test data in `pkg/mcp/testdata/`
- Integration tests for Kubernetes operations
- Toolset-specific testing