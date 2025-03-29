# Architecture Overview

## Introduction

Beacon-Kit is a modular implementation of an Ethereum consensus client (beacon node) with a clean, extensible architecture. The system is designed with separation of concerns in mind, following best practices for Go application development.

## High-Level Architecture

The system follows a layered architecture:

1. **API Layer**: Provides HTTP endpoints using the Echo framework
2. **Handler Layer**: Contains route definitions and handler implementations
3. **Backend Layer**: Implements business logic and domain operations
4. **Storage Layer**: Manages data persistence and retrieval
5. **Consensus Layer**: Implements the Ethereum consensus protocol
6. **Execution Layer**: Interfaces with the execution client

```
┌───────────────────────────────────────────────────────────┐
│                      API Layer (Echo)                      │
└───────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────┐
│                      Handler Layer                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐ │
│  │ Beacon API  │  │  Config API  │  │     Debug API     │ │
│  └─────────────┘  └──────────────┘  └───────────────────┘ │
└───────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────┐
│                      Backend Layer                         │
└───────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────┐
│                      Storage Layer                         │
└───────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴──────────┐
                    ▼                    ▼
┌───────────────────────┐    ┌───────────────────────┐
│    Consensus Layer    │    │   Execution Layer     │
└───────────────────────┘    └───────────────────────┘
```

## Key Components

### API Server
The API server provides the entry point for external systems to interact with the beacon node. It uses the Echo framework for HTTP routing and request handling.

### Handler System
Handlers define API endpoints and route incoming requests to the appropriate backend services. They follow a composable pattern with a base handler implementation.

### Backend Services
Backend services implement the core business logic of the beacon node. They interact with storage components and provide functionality to handlers.

### Storage
The storage layer manages data persistence and retrieval operations. It abstracts database operations and provides a clean interface for backend services.

### Consensus
The consensus layer implements the Ethereum consensus protocol, managing block creation, validation, and finalization.

### Execution
The execution layer interfaces with the execution client (like Geth or Nethermind), handling execution payloads and state transitions.

## Dependency Injection

The system uses dependency injection to manage component dependencies, making the code more testable and modular. Components receive their dependencies through constructors rather than creating them directly.

## Cross-Cutting Concerns

### Logging
A structured logging system is used throughout the application, providing consistent log formatting and levels. Logging can be disabled for specific components when needed.

### Error Handling
Error handling follows a consistent pattern, with errors propagating up the call stack and being logged at appropriate levels. The system uses custom error types for specific error scenarios.

### Configuration
Configuration is managed through a centralized system that supports different sources (file, environment variables) and validation.

## Communication Flow

1. Client sends HTTP request to API server
2. API server routes request to appropriate handler
3. Handler validates request and calls backend service
4. Backend service performs business logic, interacting with storage as needed
5. Backend service may interact with consensus or execution layers
6. Response flows back through the layers to the client

This architecture provides a clear separation of concerns, making the system more maintainable, testable, and extensible. 