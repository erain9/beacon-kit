# Node Core

## Overview

The node-core package provides the core functionality of the beacon node, including service lifecycle management, component coordination, and the main node implementation. It serves as the central coordinator for all beacon node operations.

## Core Components

### Node

The Node is the main entry point for the beacon node:

```go
type Node struct {
    config     *config.Config
    logger     log.Logger
    services   []Service
    lifecycle  *lifecycle.Lifecycle
    components *Components
}
```

The Node:
- Initializes all components and services
- Manages component lifecycle (start, stop)
- Coordinates communication between components
- Handles graceful shutdown

### Service Interface

Services are long-running components that perform specific functions:

```go
type Service interface {
    // Start the service
    Start(context.Context) error
    
    // Stop the service
    Stop(context.Context) error
    
    // Get service status
    Status() Status
}
```

Common services include:
- P2P networking
- Consensus processing
- Execution client interaction
- API server

### Components

The Components struct holds all the core components of the node:

```go
type Components struct {
    ChainSpec      chain.Spec
    StateProcessor StateProcessor
    StorageBackend *storage.Backend
    APIBackend     *backend.Backend
    APIEngine      *echo.Engine
    APIServer      *server.Server
    DepositStore   depositstore.Store
}
```

These components are injected into the node and other services as needed.

## Lifecycle Management

The lifecycle management system handles component initialization and shutdown:

```go
func (n *Node) Start(ctx context.Context) error {
    // Start components in dependency order
    if err := n.components.StorageBackend.Start(ctx); err != nil {
        return err
    }
    
    if err := n.components.StateProcessor.Start(ctx); err != nil {
        return err
    }
    
    // Start services
    for _, service := range n.services {
        if err := service.Start(ctx); err != nil {
            return err
        }
    }
    
    return nil
}
```

The shutdown process is handled similarly, but in reverse order:

```go
func (n *Node) Stop(ctx context.Context) error {
    // Stop services in reverse order
    for i := len(n.services) - 1; i >= 0; i-- {
        if err := n.services[i].Stop(ctx); err != nil {
            return err
        }
    }
    
    // Stop components in reverse dependency order
    if err := n.components.StateProcessor.Stop(ctx); err != nil {
        return err
    }
    
    if err := n.components.StorageBackend.Stop(ctx); err != nil {
        return err
    }
    
    return nil
}
```

## State Processor

The State Processor is responsible for handling consensus state transitions:

```go
type StateProcessor interface {
    // Start the state processor
    Start(context.Context) error
    
    // Stop the state processor
    Stop(context.Context) error
    
    // Process a block
    ProcessBlock(context.Context, *types.Block) error
    
    // Get the latest state
    GetLatestState(context.Context) (*types.State, error)
}
```

This component:
- Processes incoming blocks
- Updates the consensus state
- Manages the fork choice rule
- Handles finality

## Storage Backend

The Storage Backend manages all persistent data:

```go
type Backend struct {
    depositStore    deposit.Store
    blockStore      block.Store
    stateStore      state.Store
    validatorStore  validator.Store
}
```

This component:
- Provides access to various data stores
- Manages database transactions
- Handles data serialization/deserialization

## API Integration

The node-core package integrates with the API server:

```go
func ProvideNodeAPIBackend(
    in NodeAPIBackendInput,
) *backend.Backend {
    return backend.New(
        in.StorageBackend,
        in.ChainSpec,
        in.StateProcessor,
    )
}

func ProvideNodeAPIEngine() *echo.Engine {
    return echo.NewDefaultEngine()
}
```

These functions:
- Create the API backend with access to node components
- Initialize the API engine
- Set up the API handlers

## Configuration

The node is configured through a configuration system:

```go
type Config struct {
    // General node configuration
    DataDir        string
    ChainSpec      string
    LogLevel       string
    
    // Component-specific configuration
    Storage        storage.Config
    API            api.Config
    P2P            p2p.Config
    Execution      execution.Config
}
```

This configuration can be loaded from:
- Configuration files
- Environment variables
- Command-line flags

## Dependency Injection

The node-core package uses dependency injection to manage component dependencies:

```go
type DepositStoreInput struct {
    depinject.In
    Logger  *phuslu.Logger
    AppOpts config.AppOptions
}

func ProvideDepositStore(in DepositStoreInput) (depositstore.Store, error) {
    // Create deposit store with dependencies
}
```

This approach:
- Makes dependencies explicit
- Simplifies component initialization
- Improves testability

## Event System

The event system allows components to communicate asynchronously:

```go
type EventBus interface {
    // Subscribe to events of a specific type
    Subscribe(eventType string, subscriber Subscriber) (Subscription, error)
    
    // Publish an event
    Publish(event Event) error
}
```

This system:
- Decouples components
- Allows for flexible communication patterns
- Supports multiple subscribers for the same event type

## Service Status

Services report their status to the node:

```go
type Status int

const (
    StatusInitializing Status = iota
    StatusReady
    StatusSyncing
    StatusError
)
```

The node can query service status to determine the overall node status.

## Error Handling

The node-core package includes consistent error handling:

```go
func (n *Node) handleServiceError(service Service, err error) {
    n.logger.Error("service error",
        "service", service,
        "error", err,
    )
    
    // Handle error based on severity
    // ...
}
```

Errors are:
- Logged with appropriate context
- Propagated to the appropriate components
- Handled based on severity

## Example Usage

```go
// Create node configuration
config := &config.Config{
    DataDir:   "/path/to/data",
    ChainSpec: "mainnet",
    LogLevel:  "info",
}

// Create logger
logger := log.NewLogger(config.LogLevel)

// Create node
node, err := node.New(config, logger)
if err != nil {
    // Handle error
}

// Start node
ctx := context.Background()
if err := node.Start(ctx); err != nil {
    // Handle error
}

// Run until signal
signalCh := make(chan os.Signal, 1)
signal.Notify(signalCh, os.Interrupt, syscall.SIGTERM)
<-signalCh

// Stop node
if err := node.Stop(ctx); err != nil {
    // Handle error
}
```

This node-core package provides the foundation for a robust and extensible beacon node implementation. 