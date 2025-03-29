# Components

## Overview

The components package provides the core building blocks for the beacon node's functionality. It uses dependency injection to manage component dependencies and lifecycle, making the system more modular and testable.

## Dependency Injection

The system uses dependency injection to manage component dependencies:

```go
type NodeAPIBackendInput struct {
    depinject.In

    ChainSpec      chain.Spec
    StateProcessor StateProcessor
    StorageBackend *storage.Backend
}

func ProvideNodeAPIBackend(
    in NodeAPIBackendInput,
) *backend.Backend {
    return backend.New(
        in.StorageBackend,
        in.ChainSpec,
        in.StateProcessor,
    )
}
```

This approach:
- Makes dependencies explicit
- Improves testability
- Simplifies component initialization
- Enables runtime dependency configuration

## Core Components

### API Components

#### Node API Engine

The Node API Engine provides the HTTP server implementation:

```go
func ProvideNodeAPIEngine() *echo.Engine {
    return echo.NewDefaultEngine()
}
```

This component is responsible for:
- HTTP request handling
- Route registration
- Middleware configuration

#### Node API Backend

The Node API Backend provides the business logic for the API:

```go
type NodeAPIBackendInput struct {
    depinject.In

    ChainSpec      chain.Spec
    StateProcessor StateProcessor
    StorageBackend *storage.Backend
}

func ProvideNodeAPIBackend(
    in NodeAPIBackendInput,
) *backend.Backend {
    return backend.New(
        in.StorageBackend,
        in.ChainSpec,
        in.StateProcessor,
    )
}
```

This component:
- Implements API business logic
- Interacts with storage
- Processes state changes

### Storage Components

#### Deposit Store

The Deposit Store manages validator deposit data:

```go
type DepositStoreInput struct {
    depinject.In
    Logger  *phuslu.Logger
    AppOpts config.AppOptions
}

func ProvideDepositStore(in DepositStoreInput) (depositstore.Store, error) {
    // Initialize and return deposit store
}
```

This component:
- Stores validator deposits
- Provides access to deposit data
- Manages deposit-related operations

### State Components

#### State Processor

The State Processor manages the consensus state:

```go
type StateProcessor interface {
    // Process state transitions
    ProcessBlock(ctx context.Context, block *consensus.Block) error
    
    // Get current state
    GetState(ctx context.Context) (*consensus.State, error)
}
```

This component:
- Processes state transitions
- Validates blocks
- Manages the consensus state

## Component Lifecycle

Components may have lifecycle methods:

### Start

Initializes and starts the component:

```go
func (c *Component) Start(ctx context.Context) error {
    // Initialize resources
    // Start background processes
    return nil
}
```

### Stop

Gracefully shuts down the component:

```go
func (c *Component) Stop(ctx context.Context) error {
    // Release resources
    // Stop background processes
    return nil
}
```

These methods ensure proper resource allocation and cleanup.

## Component Configuration

Components are configured through configuration objects:

```go
type Config struct {
    // Configuration fields
}
```

Configuration can be loaded from:
- Configuration files
- Environment variables
- Command-line flags

## Component Dependencies

Components declare their dependencies through interfaces:

```go
type Component struct {
    logger   log.Logger
    storage  Storage
    services []Service
}
```

This approach:
- Reduces coupling between components
- Improves testability
- Enables mock implementations for testing

## Component Registration

Components are registered with the dependency injection system:

```go
container := depinject.NewContainer()
container.Register(
    ProvideNodeAPIEngine,
    ProvideNodeAPIBackend,
    ProvideDepositStore,
)
```

This allows the system to resolve dependencies and create component instances.

## Component Testing

Components can be tested in isolation by providing mock dependencies:

```go
func TestComponent(t *testing.T) {
    // Create mock dependencies
    mockLogger := mock.NewLogger()
    mockStorage := mock.NewStorage()
    
    // Create component with mock dependencies
    component := NewComponent(mockLogger, mockStorage)
    
    // Test component behavior
    result := component.DoSomething()
    
    // Assert expectations
    assert.Equal(t, expectedResult, result)
}
```

This approach makes it easy to test components in isolation from their dependencies. 