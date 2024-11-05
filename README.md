# Flash Loan Smart Contract Interface Documentation

This repository includes interfaces for implementing a flash loan system using the ERC-3156 standard as well as MakerDAO-specific flash loan contracts.

## Overview

Flash loans allow users to borrow tokens instantly and pay them back within the same transaction. If the borrowed amount and any associated fees are not returned in the same transaction, the loan is reverted. This repository includes the following interfaces:

- **IERC3156FlashLender**: Defines the flash loan lender functions for ERC-3156 compliant lenders.
- **IERC3156FlashBorrower**: Defines the flash loan borrower functions for ERC-3156 compliant borrowers.
- **IVatDaiFlashLender**: A MakerDAO-specific lender interface for DAI flash loans.
- **IVatDaiFlashBorrower**: A MakerDAO-specific borrower interface for DAI flash loans.

---

## IERC3156FlashLender

This interface specifies the functions that a flash lender must implement, as per the ERC-3156 standard.

### Functions

#### `maxFlashLoan(address token) → uint256`

- **Description**: Returns the maximum amount of a specific token that can be borrowed in a flash loan.
- **Parameters**:
  - `token`: The address of the token to be borrowed.
- **Returns**: The maximum amount of `token` that can be borrowed.
- **Usage**: Call this function to check the available amount of a specific token for borrowing before initiating a flash loan.

#### `flashFee(address token, uint256 amount) → uint256`

- **Description**: Calculates the fee to be charged for a specified flash loan amount.
- **Parameters**:
  - `token`: The address of the token being borrowed.
  - `amount`: The amount of tokens to be borrowed.
- **Returns**: The fee (in tokens) to be added on top of the principal for the loan.
- **Usage**: This function helps the borrower determine the total repayment amount (principal + fee).

#### `flashLoan(IERC3156FlashBorrower receiver, address token, uint256 amount, bytes calldata data) → bool`

- **Description**: Initiates a flash loan. Transfers `amount` tokens to the `receiver`, then triggers a callback to the receiver's `onFlashLoan` function to execute arbitrary logic.
- **Parameters**:
  - `receiver`: The contract address receiving the tokens and the callback.
  - `token`: The address of the token being borrowed.
  - `amount`: The number of tokens to be borrowed.
  - `data`: Additional data to be passed to the callback function, allowing the borrower to customize behavior.
- **Returns**: `true` if the flash loan is successfully executed and repaid within the transaction.
- **Usage**: Call this function to start a flash loan with specified parameters. After tokens are transferred, the `receiver`’s `onFlashLoan` callback is triggered.

---

## IERC3156FlashBorrower

This interface defines the function that a flash borrower must implement to handle flash loan callbacks.

### Functions

#### `onFlashLoan(address initiator, address token, uint256 amount, uint256 fee, bytes calldata data) → bytes32`

- **Description**: Callback function that the flash lender calls during a flash loan. This function contains the logic to be executed with the borrowed funds.
- **Parameters**:
  - `initiator`: The original address initiating the flash loan.
  - `token`: The address of the borrowed token.
  - `amount`: The borrowed amount.
  - `fee`: The fee associated with the loan.
  - `data`: Arbitrary data sent from the lender to guide borrower-specific logic.
- **Returns**: The keccak256 hash of "ERC3156FlashBorrower.onFlashLoan" to indicate successful completion of the callback.
- **Usage**: Implement this function in a contract to define the actions to be performed with the borrowed tokens. Ensure that the borrowed amount plus the fee is repaid by the end of the function.

---

## IVatDaiFlashLender (MakerDAO DAI Flash Loan Interface)

This interface is specifically designed for handling flash loans in DAI using MakerDAO’s `vat` system. It has a unique function for managing DAI flash loans.

### Functions

#### `vatDaiFlashLoan(IVatDaiFlashBorrower receiver, uint256 amount, bytes calldata data) → bool`

- **Description**: Initiates a DAI flash loan, transferring the specified `amount` to the `receiver` and triggering the receiver’s `onVatDaiFlashLoan` callback.
- **Parameters**:
  - `receiver`: The contract address receiving the DAI and the callback.
  - `amount`: The amount of DAI to be borrowed, measured in `rad` (an internal unit used by MakerDAO where 1 `rad` = 10^45 DAI).
  - `data`: Additional data for the borrower to customize the loan behavior.
- **Returns**: `true` if the flash loan and repayment are successful.
- **Usage**: Use this function to initiate a MakerDAO-specific DAI flash loan. Once tokens are transferred, the `receiver`’s `onVatDaiFlashLoan` function will be called for executing the loan logic.

---

## IVatDaiFlashBorrower (MakerDAO DAI Flash Loan Callback Interface)

This interface specifies the callback function required by borrowers when taking DAI flash loans via the `IVatDaiFlashLender`.

### Functions

#### `onVatDaiFlashLoan(address initiator, uint256 amount, uint256 fee, bytes calldata data) → bytes32`

- **Description**: Callback function that executes during a DAI flash loan, allowing the borrower to perform custom logic.
- **Parameters**:
  - `initiator`: The address initiating the flash loan.
  - `amount`: The amount of DAI borrowed, in `rad`.
  - `fee`: The additional amount of DAI required to repay the loan, also in `rad`.
  - `data`: Arbitrary data passed from the lender for custom logic.
- **Returns**: The keccak256 hash of "IVatDaiFlashLoanReceiver.onVatDaiFlashLoan" to confirm successful completion of the callback.
- **Usage**: Implement this function to handle the specific operations performed with borrowed DAI. Ensure repayment of `amount + fee` is completed within this function.

---

## Flow of a Flash Loan Transaction

1. **Initialization**: The borrower contract checks the maximum available amount (`maxFlashLoan`) and the associated fee (`flashFee`) using the `IERC3156FlashLender` interface.
  
2. **Flash Loan Execution**: If the amount is available and the fee is acceptable, the borrower calls `flashLoan` on `IERC3156FlashLender`, specifying the receiver, token, amount, and any additional data.

3. **Callback Execution**: The lender transfers the tokens to the borrower contract and invokes `onFlashLoan` on the borrower’s side (or `onVatDaiFlashLoan` for DAI loans). The borrower performs any required actions, such as arbitrage or liquidation, within the callback function.

4. **Repayment**: By the end of `onFlashLoan` or `onVatDaiFlashLoan`, the borrowed amount plus the fee must be returned to the lender. If not, the entire transaction is reverted.

