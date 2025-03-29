# Consensus

## Overview

The consensus package implements the Ethereum consensus protocol (Proof of Stake) for the beacon node. It manages block creation, validation, finalization, and the overall network consensus.

## Consensus Protocol

The beacon node implements the Ethereum Proof of Stake (PoS) consensus protocol, which uses validators to propose and attest to blocks. The key aspects of this protocol include:

1. **Slot-based time**: Time is divided into slots (12 seconds) and epochs (32 slots)
2. **Validator selection**: Validators are pseudorandomly selected to propose blocks and participate in committees
3. **Block proposals**: Selected validators propose blocks for their assigned slots
4. **Attestations**: Committees of validators vote on block proposals
5. **Finality**: The Casper FFG (Friendly Finality Gadget) finality algorithm determines when blocks are finalized

## Core Components

### State Transition Function

The state transition function is the core of the consensus protocol:

```go
type StateTransition interface {
    // Process a block and update the state
    ProcessBlock(ctx context.Context, state *types.State, block *types.Block) (*types.State, error)
    
    // Process a slot without a block (empty slot)
    ProcessSlot(ctx context.Context, state *types.State) (*types.State, error)
}
```

This function:
- Validates blocks against the current state
- Applies state changes according to the consensus rules
- Ensures all validators follow the same rules

### Fork Choice

The fork choice rule determines the canonical chain:

```go
type ForkChoice interface {
    // Get the head of the chain
    GetHead(ctx context.Context) ([]byte, error)
    
    // Process a new block
    OnBlock(ctx context.Context, block *types.Block) error
    
    // Process a new attestation
    OnAttestation(ctx context.Context, attestation *types.Attestation) error
}
```

The system implements the LMD GHOST (Latest Message Driven Greediest Heaviest Observed SubTree) fork choice rule.

### Validator Management

The validator management system handles validator operations:

```go
type ValidatorManager interface {
    // Get validators for the current epoch
    GetCurrentValidators(ctx context.Context) ([]*types.Validator, error)
    
    // Get committee for a specific slot and index
    GetCommittee(ctx context.Context, slot uint64, index uint64) ([]uint64, error)
    
    // Get proposer for a specific slot
    GetProposer(ctx context.Context, slot uint64) (uint64, error)
}
```

This component:
- Manages validator lifecycle (pending, active, exiting, slashed)
- Computes committee assignments
- Determines block proposers

## Consensus Types

### Block

A block is the fundamental unit of the blockchain:

```go
type Block struct {
    Slot            uint64
    ProposerIndex   uint64
    ParentRoot      []byte
    StateRoot       []byte
    Body            *BlockBody
}

type BlockBody struct {
    Attestations    []*Attestation
    Deposits        []*Deposit
    Exits           []*Exit
    ExecutionPayload *ExecutionPayload
    // Other fields...
}
```

### Attestation

Attestations are votes from validators:

```go
type Attestation struct {
    Data            *AttestationData
    AggregationBits []byte
    Signature       []byte
}

type AttestationData struct {
    Slot            uint64
    Index           uint64
    BeaconBlockRoot []byte
    Source          *Checkpoint
    Target          *Checkpoint
}
```

### Checkpoint

Checkpoints are used for finality:

```go
type Checkpoint struct {
    Epoch           uint64
    Root            []byte
}
```

## Consensus Events

The consensus system processes several types of events:

### Block Processing

When a new block is received:

1. Validate the block signature
2. Validate the block proposal
3. Process the block through the state transition function
4. Update the fork choice with the new block
5. Notify other components of the new block

### Attestation Processing

When attestations are received:

1. Validate the attestation signature
2. Process the attestation
3. Update the fork choice with the new attestation
4. Aggregate attestations when possible

### Slot Processing

For each slot:

1. Process empty slots since the last block
2. If selected as proposer, create and broadcast a block
3. If part of a committee, create and broadcast attestations

## Finality

The finality mechanism determines when blocks are considered final:

```go
type Finality interface {
    // Check if a block is justified
    IsJustified(ctx context.Context, blockRoot []byte) (bool, error)
    
    // Check if a block is finalized
    IsFinalized(ctx context.Context, blockRoot []byte) (bool, error)
    
    // Get the latest justified checkpoint
    GetJustifiedCheckpoint(ctx context.Context) (*types.Checkpoint, error)
    
    // Get the latest finalized checkpoint
    GetFinalizedCheckpoint(ctx context.Context) (*types.Checkpoint, error)
}
```

The system implements the Casper FFG finality algorithm, which requires:
- Supermajority (2/3) of validators to vote for justification
- Two consecutive justified epochs for finalization

## Slashing Protection

The slashing protection system prevents validators from creating slashable offenses:

```go
type SlashingProtection interface {
    // Check if a block proposal is slashable
    IsSlashableBlock(ctx context.Context, validatorIndex uint64, slot uint64, blockRoot []byte) (bool, error)
    
    // Check if an attestation is slashable
    IsSlashableAttestation(ctx context.Context, validatorIndex uint64, data *types.AttestationData) (bool, error)
    
    // Record a block proposal
    RecordBlock(ctx context.Context, validatorIndex uint64, slot uint64, blockRoot []byte) error
    
    // Record an attestation
    RecordAttestation(ctx context.Context, validatorIndex uint64, data *types.AttestationData) error
}
```

This protects validators from accidental slashing due to:
- Double block proposals
- Surrounding/surrounded votes
- Double votes

## Example Workflow

```go
// Initialize consensus components
stateTransition := consensus.NewStateTransition(config)
forkChoice := consensus.NewForkChoice(config)
validatorManager := consensus.NewValidatorManager(config)

// Process a new block
func ProcessBlock(ctx context.Context, block *types.Block) error {
    // Get current state
    state, err := stateStore.GetLatestState(ctx)
    if err != nil {
        return err
    }
    
    // Process block
    newState, err := stateTransition.ProcessBlock(ctx, state, block)
    if err != nil {
        return err
    }
    
    // Store new state
    if err := stateStore.StoreState(ctx, newState); err != nil {
        return err
    }
    
    // Update fork choice
    if err := forkChoice.OnBlock(ctx, block); err != nil {
        return err
    }
    
    // Store block
    return blockStore.StoreBlock(ctx, block)
}
```

This consensus system ensures all nodes reach agreement on the canonical chain in a secure and efficient manner. 