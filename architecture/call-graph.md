# Call Graph Documentation

## Primary Entry Points

### MoneyMarket.operate()

```
MoneyMarket.operate(nftId, positionIndex, actionData)
|
|-- _handleMsgDetails modifier
|   |-- Set _msgSender = msg.sender
|   +-- Set _msgValue = msg.value
|
|-- _spell(OPERATE_MODULE_IMPLEMENTATION)
|   |
|   |-- [nftId == 0] Create new NFT
|   |   +-- mint() -> _mint()
|   |
|   |-- [positionIndex == 0] Create new position
|   |   +-- _createPosition()
|   |
|   +-- [positionIndex > 0] Modify existing position
|       |-- [Type 1] _processNormalSupplyAction()
|       |-- [Type 2] _processNormalBorrowAction()
|       +-- [Type 3/4] DEX_V2.startOperation()
|
+-- _handleMsgDetails cleanup
    |-- Refund remaining ETH
    +-- Clear _msgSender, _msgValue
```

### DEX_V2.startOperation()

```
DEX_V2.startOperation(data)
|
|-- OC.activateOperation()
|   +-- Set transient OPERATION_ACTIVE = 1
|
|-- callback -> MoneyMarket.startOperationCallback()
|   |
|   |-- _spell(CALLBACK_MODULE_IMPLEMENTATION)
|   |   |
|   |   |-- [amount0 == 0 && amount1 == 0] Fee collection
|   |   |   |-- DEX_V2.operate(withdraw/payback with 0)
|   |   |   |-- _updateAndCollectFees()
|   |   |   |-- Check position empty -> delete
|   |   |   +-- _feeSettle()
|   |   |
|   |   |-- [D3 deposit] Smart collateral supply
|   |   |   |-- DEX_V2.operate(D3_DEPOSIT)
|   |   |   |-- _depositSettle()
|   |   |   +-- _checkAndUpdateCaps()
|   |   |
|   |   |-- [D3 withdraw] Smart collateral withdraw
|   |   |   |-- DEX_V2.operate(D3_WITHDRAW)
|   |   |   |-- _withdrawSettle()
|   |   |   |-- Check position empty -> delete
|   |   |   +-- _checkHf()
|   |   |
|   |   |-- [D4 borrow] Smart debt borrow
|   |   |   |-- DEX_V2.operate(D4_BORROW)
|   |   |   |-- _borrowSettle()
|   |   |   |-- _updateFeeStored()
|   |   |   |-- _checkAndUpdateCaps()
|   |   |   +-- _checkHf()
|   |   |
|   |   +-- [D4 payback] Smart debt payback
|   |       |-- DEX_V2.operate(D4_PAYBACK)
|   |       |-- _paybackSettle()
|   |       +-- Check position empty -> delete
|   |
|   +-- Return to DEX_V2
|
|-- PT.requireAllPendingTransfersCleared()
|
+-- OC.deactivateOperation()
```

### DEX_V2.settle()

```
DEX_V2.settle(token, supplyAmount, borrowAmount, storeAmount, to, isCallback)
|
|-- _onlyAfterOperationStarted modifier
|-- _reentrancyLock modifier
|
|-- Calculate net amounts
|
|-- [netAmount <= 0] Update storage first (CEI for outbound)
|   +-- _updateSettledAmountsOnStorage()
|
|-- Token transfer path selection
|   |
|   |-- [Skip liquidity - net in]
|   |   |-- dexCallback() OR safeTransferFrom()
|   |   +-- Update _unaccountedBorrowAmount
|   |
|   |-- [Skip liquidity - net out]
|   |   |-- Update _unaccountedBorrowAmount
|   |   +-- safeTransfer()
|   |
|   +-- [Call liquidity]
|       +-- _callLiquidityLayer()
|           +-- LIQUIDITY.operate()
|               +-- [if input needed] liquidityCallback()
|
|-- [netAmount > 0] Update storage last (CEI for inbound)
|   +-- _updateSettledAmountsOnStorage()
|
+-- Emit LogSettle
```

## Settlement Functions

```
_depositSettle(token, amount, feeAccrued, to)
+-- DEX_V2.settle(token, +amount-fee, 0, +fee, to, IS_CALLBACK)

_withdrawSettle(token, amount, feeAccrued, to)
+-- DEX_V2.settle(token, -(amount+fee), 0, +fee, to, IS_CALLBACK)

_borrowSettle(token, amount, feeAccrued, to)
+-- DEX_V2.settle(token, -fee, +amount, +fee, to, IS_CALLBACK)

_paybackSettle(token, amount, feeAccrued, to)
+-- DEX_V2.settle(token, -fee, -amount, +fee, to, IS_CALLBACK)

_feeSettle(token, feeAccrued, feeCollection, to)
+-- DEX_V2.settle(token, -feeAccrued, 0, feeAccrued-collection, to, IS_CALLBACK)
```
