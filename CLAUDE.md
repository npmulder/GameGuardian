# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GameGuardian is a cross-platform parental gaming oversight platform built with .NET 9 and .NET Aspire. The project is in early development stage with just the basic Aspire AppHost setup.

## Technology Stack

- **.NET 9**: Main framework
- **.NET Aspire**: Cloud-native development stack for building distributed applications
- **C#**: Primary programming language with nullable reference types enabled

## Project Structure

```
GameGuardian/
├── GameGuardian.sln                           # Visual Studio solution file
├── src/
│   ├── GameGuardian.AppHost/                  # .NET Aspire AppHost for orchestration
│   │   ├── GameGuardian.AppHost.csproj        # Aspire AppHost project
│   │   └── AppHost.cs                         # Main application host entry point
│   └── GameGuardian.ServiceDefaults/          # Shared service defaults
│       ├── GameGuardian.ServiceDefaults.csproj
│       └── Extensions.cs                      # Common extensions for services
└── docs/
    ├── Project.md                             # Detailed project documentation
    └── README.md                              # Project overview
```

## Common Development Commands

### Build
```bash
dotnet build
```

### Run Application
```bash
dotnet run --project src/GameGuardian.AppHost
```

### Restore Packages
```bash
dotnet restore
```

### Test (when test projects are added)
```bash
dotnet test
```

## Architecture Notes

### .NET Aspire AppHost
- The project uses .NET Aspire for cloud-native development
- `GameGuardian.AppHost` is the orchestration layer that will manage distributed services
- Currently minimal with just basic DistributedApplication setup

### Service Defaults
- `GameGuardian.ServiceDefaults` contains shared configurations for:
  - OpenTelemetry (tracing, metrics, logging)
  - Health checks (health and aliveness endpoints)
  - Service discovery
  - HTTP client resilience
  - Standard ASP.NET Core instrumentation

### Development Environment
- Uses .NET 9.0 target framework
- Implicit usings enabled
- Nullable reference types enabled
- User secrets configured for AppHost project

## Project Roadmap (from docs/Project.md)

The project aims to implement:
- Cross-platform parental gaming oversight
- Real-time monitoring and purchase controls
- UNO Platform frontend for cross-platform UI
- Gaming platform integrations (Steam, Xbox, PlayStation, Nintendo)
- Kubernetes deployment with GitOps
- Comprehensive observability and analytics

## Development Guidelines

- Follow .NET coding conventions
- Use nullable reference types appropriately
- Leverage Aspire service defaults for consistent configuration
- Add health checks for new services
- Instrument code with OpenTelemetry for observability