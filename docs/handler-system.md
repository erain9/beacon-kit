# Handler System

## Overview

The handler system provides a structured approach to defining and managing API endpoints. It uses a composable pattern with a base handler implementation and specialized handlers for specific API areas.

## Core Components

### Route

A `Route` represents a single API endpoint:

```go
type Route struct {
    Method  string
    Path    string
    Handler handlerFn
}
```

Routes define:
- HTTP method (`GET`, `POST`, etc.)
- URL path
- Handler function

Routes can be decorated with middleware, such as logging:

```go
func (r *Route) DecorateWithLogs(logger log.Logger) {
    handler := r.Handler
    r.Handler = func(ctx Context) (any, error) {
        logger.Info("received request", "method", r.Method, "path", r.Path)
        res, err := handler(ctx)
        if err != nil {
            logger.Error("error handling request", "error", err)
        }
        logger.Info("request handled")
        return res, err
    }
}
```

### RouteSet

A `RouteSet` groups related routes under a common base path:

```go
type RouteSet struct {
    BasePath string
    Routes   []*Route
}
```

RouteSet allows for organizing routes by functionality or module.

### Handler Function

All handler functions follow a consistent signature:

```go
type handlerFn func(c Context) (any, error)
```

This pattern ensures:
- Consistent error handling
- Consistent response formatting
- Easy composition with middleware

### BaseHandler

The `BaseHandler` provides common functionality for all handlers:

```go
type BaseHandler struct {
    routes *RouteSet
    logger log.Logger
}
```

BaseHandler acts as a foundation for specialized handlers, providing common methods and fields.

### Handlers Interface

All handlers implement the `Handlers` interface:

```go
type Handlers interface {
    RegisterRoutes(logger log.Logger)
    RouteSet() *RouteSet
}
```

This interface ensures that all handlers can:
- Register their routes with the API server
- Provide access to their route set

## Specialized Handlers

The system includes several specialized handlers for different API areas:

### Beacon Handler

Handles beacon-related API endpoints:

```go
type Handler struct {
    *handlers.BaseHandler
    backend Backend
}
```

### Config Handler

Manages configuration-related endpoints:

```go
type Handler struct {
    *handlers.BaseHandler
    backend Backend
}
```

### Debug Handler

Provides debugging endpoints:

```go
type Handler struct {
    *handlers.BaseHandler
    backend Backend
}
```

### Node Handler

Manages node-related endpoints:

```go
type Handler struct {
    *handlers.BaseHandler
}
```

## Backend Integration

Handlers delegate business logic to backend services through interfaces:

```go
type Backend interface {
    // Backend methods...
}
```

This separation of concerns:
- Improves testability
- Decouples request handling from business logic
- Allows for different backend implementations

## Handler Registration

Handlers are registered with the API server during initialization:

```go
func (h *Handler) RegisterRoutes(logger log.Logger) {
    h.logger = logger
    
    // Register routes
    h.routes = handlers.NewRouteSet("/path", []*handlers.Route{
        {
            Method:  http.MethodGet,
            Path:    "/endpoint",
            Handler: h.HandleEndpoint,
        },
        // More routes...
    })
    
    // Apply middleware
    for _, route := range h.routes.Routes {
        route.DecorateWithLogs(logger)
    }
}
```

## Error Handling

Handlers use a consistent error handling pattern:

```go
func (h *Handler) HandleEndpoint(c Context) (any, error) {
    // Process request
    result, err := h.backend.DoSomething()
    if err != nil {
        return nil, errors.Wrap(err, "failed to do something")
    }
    
    // Return response
    return result, nil
}
```

Errors are propagated up to the API server for logging and formatting into HTTP responses.

## Example Handler

```go
// Create handler
handler := NewHandler(backend)

// Register routes
handler.RegisterRoutes(logger)

// Add to server
server.AddHandler(handler)
```

This pattern makes it easy to add new API endpoints by creating specialized handlers and registering them with the server. 