# API Server

## Overview

The API server is responsible for serving HTTP endpoints and routing requests to the appropriate handlers. It uses the Echo framework for HTTP routing and middleware support.

## Server Architecture

The API server follows a modular design with clear separation of concerns:

1. **Engine**: Abstracts the underlying HTTP server implementation
2. **Server**: Manages the lifecycle of the API server
3. **Handlers**: Process requests and generate responses
4. **Routes**: Define API endpoints and methods

## Server Configuration

The server is configured through the `Config` struct, which includes settings like:

- Host address and port
- Logging configuration
- CORS settings (if applicable)
- Security settings

## Engine Interface

The server uses an `Engine` interface to abstract the underlying HTTP server implementation:

```go
type Engine interface {
    Run(addr string) error
    RegisterRoutes(*handlers.RouteSet, log.Logger)
}
```

This abstraction allows for different HTTP server implementations to be used. The default implementation uses the Echo framework.

## Server Initialization

The server is initialized using the `New` function, which accepts:

- Configuration (`Config`)
- Engine implementation (`Engine`)
- Logger (`log.Logger`)
- Handlers (`handlers.Handlers`)

```go
func New(
    config Config,
    engine Engine,
    logger log.Logger,
    handlers ...handlers.Handlers,
) *Server {
    // Initialize server with components
}
```

## Handler Registration

Handlers are registered with the server during initialization:

```go
for _, handler := range handlers {
    handler.RegisterRoutes(apiLogger)
    engine.RegisterRoutes(handler.RouteSet(), apiLogger)
}
```

Each handler defines its own routes, which are then registered with the engine.

## Logging

The server supports configurable logging:

```go
apiLogger := logger
if !config.Logging {
    apiLogger = noop.NewLogger[log.Logger]()
}
```

When logging is disabled, a no-op logger is used to avoid performance overhead.

## Server Lifecycle

The server supports standard lifecycle methods:

- `Start`: Starts the HTTP server on the configured address
- `Stop`: Gracefully shuts down the server, allowing in-flight requests to complete

These methods support context-based cancelation for proper shutdown behavior.

## Error Handling

Errors are handled consistently throughout the API server:

1. Handler functions return errors to the server
2. The server logs errors at appropriate levels
3. Errors are translated into HTTP responses with appropriate status codes

## Middleware Support

The API server supports middleware for cross-cutting concerns:

- Request logging
- Error handling
- Authentication and authorization
- Request validation
- Response formatting

Middleware can be applied globally or to specific routes.

## Example Usage

```go
// Create engine
engine := echo.NewDefaultEngine()

// Create server
server := server.New(
    server.Config{
        Logging: true,
    },
    engine,
    logger,
    beaconHandler,
    configHandler,
    debugHandler,
)

// Start server
err := server.Start(context.Background())
```

This creates and starts an API server with the specified handlers and configuration. 