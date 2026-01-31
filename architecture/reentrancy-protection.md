# Reentrancy Protection Architecture

## Overview

The protocol implements a defense-in-depth approach with three independent
reentrancy protection layers, each using different storage mechanisms and
protecting different code paths.

## Protection Layers

### Layer 1: Message Sender Guard

**Location**: `moneyMarket/core/base/helpers.sol`
**Storage**: Regular storage (`_msgSender`)
**Scope**: All Money Market external entry points

```solidity
modifier _handleMsgDetails() {
    if (_msgSender != address(0)) revert;
    _msgSender = msg.sender;
    _msgValue = msg.value;
    _;
    // cleanup
    delete _msgValue;
    delete _msgSender;
}
```

**Protected Functions**:
- `operate()`
- `liquidate()`
- `changeEmode()`

**Characteristics**:
- Set at entry, cleared at exit
- Blocks all nested calls to protected functions
- Storage-based for maximum reliability

### Layer 2: Operation Control

**Location**: `libraries/operationControl.sol`
**Storage**: Transient storage (EIP-1153)
**Scope**: DEX V2 operation lifecycle

```solidity
function activateOperation() internal {
    assembly {
        if tload(OPERATION_ACTIVE_SLOT) { revert(0, 0) }
        tstore(OPERATION_ACTIVE_SLOT, 1)
    }
}

function deactivateOperation() internal {
    assembly {
        tstore(OPERATION_ACTIVE_SLOT, 0)
    }
}
```

**Protected Functions**:
- `startOperation()` (prevents nesting)
- All functions with `_onlyAfterOperationStarted` (require active operation)

**Characteristics**:
- Transient storage for gas efficiency
- Automatically cleared at transaction end
- Prevents nested operation contexts

### Layer 3: Reentrancy Lock

**Location**: `libraries/reentrancyLock.sol`
**Storage**: Transient storage (EIP-1153)
**Scope**: DEX V2 settlement function

```solidity
function lock() internal {
    assembly {
        if tload(REENTRANCY_LOCK_SLOT) { revert(0, 0) }
        tstore(REENTRANCY_LOCK_SLOT, 1)
    }
}

function unlock() internal {
    assembly {
        tstore(REENTRANCY_LOCK_SLOT, 0)
    }
}
```

**Protected Functions**:
- `settle()`

**Characteristics**:
- Prevents nested settlement calls
- Unlocked after each settle completes
- Allows multiple sequential settles within one operation

## Layer Interaction Matrix

```
                    | Layer 1    | Layer 2    | Layer 3
                    | _msgSender | OpControl  | Reentrancy
--------------------|------------|------------|------------
MM.operate()        | LOCKED     | -          | -
  +-- DEX.startOp() | LOCKED     | LOCKED     | -
       +-- callback | LOCKED     | LOCKED     | -
            +-- DEX.operate()
                    | LOCKED     | LOCKED     | -
            +-- DEX.settle()
                    | LOCKED     | LOCKED     | LOCKED
                 (transfers)
                    | LOCKED     | LOCKED     | UNLOCKED
            +-- DEX.settle()
                    | LOCKED     | LOCKED     | LOCKED
                 (transfers)
                    | LOCKED     | LOCKED     | UNLOCKED
       +-- (return) | LOCKED     | UNLOCKED   | -
  +-- (return)      | UNLOCKED   | -          | -
```

## External Call Analysis

### Safe External Calls

| Call | Protection Active |
|------|-------------------|
| LIQUIDITY.operate() | Layers 1, 2, 3 + Liquidity's own |
| safeTransfer() | Layers 1, 2 (Layer 3 may be unlocked) |
| safeTransferNative() | Layers 1, 2 (Layer 3 may be unlocked) |
| dexCallback() | Layers 1, 2, 3 |
| liquidityCallback() | Layers 1, 2, 3 |

### Callback Restrictions

The `dexCallback()` function includes additional validation:

```solidity
function dexCallback(address token_, address to_, uint256 amount_) external {
    if (msg.sender != address(DEX_V2)) revert;
    if (_msgSender == address(0)) revert;
    if (!(to_ == address(LIQUIDITY) || to_ == address(DEX_V2))) revert;
    // ...
}
```

## Transient Storage Considerations

Transient storage (EIP-1153) provides:

1. **Gas Efficiency**: Cheaper than regular storage for same-transaction data
2. **Automatic Cleanup**: Values cleared at transaction end
3. **Isolation**: Cannot leak state between transactions

The protocol uses transient storage for Layers 2 and 3, while Layer 1 uses
regular storage for maximum reliability as the primary defense.

## Conclusion

The three-layer protection system provides comprehensive coverage:

- **Layer 1** prevents all Money Market reentry
- **Layer 2** prevents nested DEX operations
- **Layer 3** prevents nested settlements

This defense-in-depth approach ensures that even if one layer had a bypass,
the others would maintain protection.
