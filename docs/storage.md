# Storage

## Overview

The storage package provides data persistence and retrieval capabilities for the beacon node. It abstracts database operations and provides a clean interface for backend services.

## Database Layer

### Database Interface

The database module provides a generalized interface for database operations through the Cosmos DB package:

```go
func OpenDB(rootDir string, backendType dbm.BackendType) (dbm.DB, error) {
    dataDir := filepath.Join(rootDir, "data")
    return dbm.NewDB("application", backendType, dataDir)
}
```

This function:
- Initializes a database connection
- Configures the database location
- Specifies the backend type (e.g., LevelDB, BadgerDB)

### Database Backend Types

The system supports multiple database backend types:

- **LevelDB**: A fast key-value storage library
- **BadgerDB**: An embeddable, persistent, and fast key-value database
- **MemDB**: An in-memory database for testing

The backend type can be configured through application options.

## Storage Components

### Storage Backend

The storage backend provides a unified interface for storage operations:

```go
type Backend struct {
    depositStore    deposit.Store
    blockStore      block.Store
    stateStore      state.Store
    validatorStore  validator.Store
}
```

This component:
- Manages multiple specialized stores
- Provides transaction support
- Handles data serialization/deserialization

### Deposit Store

The Deposit Store manages validator deposit data:

```go
type Store interface {
    // Store a new deposit
    StoreDeposit(ctx context.Context, deposit *types.Deposit) error
    
    // Get deposits in a range
    GetDeposits(ctx context.Context, fromIndex, toIndex uint64) ([]*types.Deposit, error)
    
    // Get deposit by index
    GetDeposit(ctx context.Context, index uint64) (*types.Deposit, error)
    
    // Get total number of deposits
    GetDepositCount(ctx context.Context) (uint64, error)
}
```

### Block Store

The Block Store manages consensus blocks:

```go
type Store interface {
    // Store a new block
    StoreBlock(ctx context.Context, block *types.Block) error
    
    // Get block by root
    GetBlock(ctx context.Context, root []byte) (*types.Block, error)
    
    // Get blocks in a range
    GetBlocks(ctx context.Context, fromSlot, toSlot uint64) ([]*types.Block, error)
}
```

### State Store

The State Store manages consensus state:

```go
type Store interface {
    // Store state
    StoreState(ctx context.Context, state *types.State) error
    
    // Get state by root
    GetState(ctx context.Context, root []byte) (*types.State, error)
    
    // Get latest state
    GetLatestState(ctx context.Context) (*types.State, error)
}
```

## Data Models

Storage components use well-defined data models to ensure consistency:

```go
type Deposit struct {
    PublicKey       []byte
    WithdrawalCredentials []byte
    Amount          uint64
    Signature       []byte
    Index           uint64
}

type Block struct {
    Slot            uint64
    ProposerIndex   uint64
    ParentRoot      []byte
    StateRoot       []byte
    Body            *BlockBody
}

type State struct {
    Slot            uint64
    BlockRoot       []byte
    Validators      []*Validator
    Balances        []uint64
    // Other state fields...
}
```

## Transactions

For operations that require atomicity, storage components support transactions:

```go
func (s *Store) WithTransaction(ctx context.Context, fn func(ctx context.Context) error) error {
    // Begin transaction
    tx, err := s.db.BeginTx()
    if err != nil {
        return err
    }
    
    // Create context with transaction
    txCtx := context.WithValue(ctx, txKey, tx)
    
    // Execute function
    if err := fn(txCtx); err != nil {
        // Rollback on error
        tx.Discard()
        return err
    }
    
    // Commit transaction
    return tx.Commit()
}
```

This ensures data consistency across multiple operations.

## Data Serialization

Storage components handle data serialization/deserialization:

```go
func (s *Store) serializeDeposit(deposit *types.Deposit) ([]byte, error) {
    // Serialize deposit to bytes
    return json.Marshal(deposit)
}

func (s *Store) deserializeDeposit(data []byte) (*types.Deposit, error) {
    // Deserialize bytes to deposit
    var deposit types.Deposit
    if err := json.Unmarshal(data, &deposit); err != nil {
        return nil, err
    }
    return &deposit, nil
}
```

## Key Management

Storage components use consistent key schemes:

```go
func depositKey(index uint64) []byte {
    key := make([]byte, 9)
    key[0] = depositPrefix
    binary.BigEndian.PutUint64(key[1:], index)
    return key
}

func blockKey(root []byte) []byte {
    key := make([]byte, 1+len(root))
    key[0] = blockPrefix
    copy(key[1:], root)
    return key
}
```

This ensures data is stored and retrieved consistently.

## Storage Paths

Data is stored in a directory structure:

```
<rootDir>/
  data/
    application.db/ (or other database files)
```

## Example Usage

```go
// Open database
db, err := OpenDB("/path/to/root", dbm.GoLevelDBBackend)
if err != nil {
    // Handle error
}

// Create deposit store
depositStore := deposit.NewStore(db)

// Store deposit
err = depositStore.StoreDeposit(ctx, deposit)
if err != nil {
    // Handle error
}

// Get deposit
retrievedDeposit, err := depositStore.GetDeposit(ctx, index)
if err != nil {
    // Handle error
}
```

This storage system provides a flexible and consistent way to manage beacon node data. 