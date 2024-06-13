# BasicUniswap Smart Contract Audit Report

## Summary of Findings

During the security audit of the BasicUniswap contract, the following vulnerabilities were identified:

- Reentrancy Attack in `_swap` Function (Severity: Critical)
- Inefficient Use of Public Visibility in `swapExactInput` Function (Severity: Low)
- Unused Function parameters and Local variable (Severity: Warning)

## Detailed Findings

### Critical Vulnerabilities

#### 1. Reentrancy Attack in `_swap` Function

**Description:**
The `_swap` function is vulnerable to a reentrancy attack, allowing an attacker to recursively call the function and manipulate the contract's state before the balance is updated, potentially draining the contract's funds.

**Severity:**
Critical

**Affected Code:**

```solidity
function _swap(
    IERC20 inputToken,
    uint256 inputAmount,
    IERC20 outputToken,
    uint256 outputAmount
 ) private {
     if (
         _isUnknown(inputToken) ||
         _isUnknown(outputToken) ||
         inputToken == outputToken
     ) {
         revert BasicUniswap__InvalidToken();
     }

     swap_count++;
     if (swap_count >= SWAP_COUNT_MAX) {
         swap_count = 0;
         outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
     }
     emit Swap(
         msg.sender,
         inputToken,
         inputAmount,
         outputToken,
         outputAmount
     );

     inputToken.safeTransferFrom(msg.sender, address(this), inputAmount);
     outputToken.safeTransfer(msg.sender, outputAmount);
}
```

**Details:**
In the above function, the contract emits an event and transfers tokens before updating the state. This sequence allows an attacker to reenter the \_swap function before the state update, enabling them to perform actions that could disrupt the contract's state.

**Mitigation:**
To prevent reentrancy attacks, the function should follow the "checks-effects-interactions" pattern, updating the state before making any external calls.

```solidity
function _swap(
    IERC20 inputToken,
    uint256 inputAmount,
    IERC20 outputToken,
    uint256 outputAmount
) private {
    if (
        _isUnknown(inputToken) ||
        _isUnknown(outputToken) ||
        inputToken == outputToken
    ) {
        revert BasicUniswap__InvalidToken();
    }

    // Update the state before making external calls
    swap_count++;
    if (swap_count >= SWAP_COUNT_MAX) {
        swap_count = 0;
    }

    emit Swap( msg.sender, inputToken, inputAmount, outputToken, outputAmount);

    // Perform external calls after state updates
    inputToken.safeTransferFrom(msg.sender, address(this), inputAmount);
    outputToken.safeTransfer(msg.sender, outputAmount);

    // Incentivize swapping after state updates
    if (swap_count == 0) {
        outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
    }
}
```

#### 2. Inefficient Use of Public Visibility in `swapExactInput` Function

**Description**
The `swapExactInput` function could be marked as `external`, this change can lead to gas savings and improve the contract's efficiency.

**Severity:**
Low

**Affected code**

```solidity
function swapExactInput(
        IERC20 inputToken,
        uint256 inputAmount,
        IERC20 outputToken,
        uint256 minOutputAmount,
        uint64 deadline
    )
        public
        revertIfZero(inputAmount)
        revertIfDeadlinePassed(deadline)
        returns (uint256 output)
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

        uint256 outputAmount = getOutputAmountBasedOnInput(
            inputAmount,
            inputReserves,
            outputReserves
        );

        if (outputAmount < minOutputAmount) {
            revert BasicUniswap__OutputTooLow(outputAmount, minOutputAmount);
        }

        _swap(inputToken, inputAmount, outputToken, outputAmount);
    }
```

**Mitigation**
Using external instead of public when appropriate can lead to gas savings because external functions use a different ABI encoding that is slightly more efficient.

```solidity
function swapExactInput(
        IERC20 inputToken,
        uint256 inputAmount,
        IERC20 outputToken,
        uint256 minOutputAmount,
        uint64 deadline
    )
        external
        revertIfZero(inputAmount)
        revertIfDeadlinePassed(deadline)
        returns (uint256 output)
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

        uint256 outputAmount = getOutputAmountBasedOnInput(
            inputAmount,
            inputReserves,
            outputReserves
        );

        if (outputAmount < minOutputAmount) {
            revert BasicUniswap__OutputTooLow(outputAmount, minOutputAmount);
        }

        _swap(inputToken, inputAmount, outputToken, outputAmount);
    }
```

--------------------------------------------------------------------------------**0**----------------------------------------------

### Warning

#### 1. Unused Function parameters and Local variable

**Description:**
The warnings indicates that a function in Solidity code includes parameters that are not used within the function body.
While not critical, unused parameters can lead to confusion and indicate potential errors or unoptimized code.

**Severity:**
Low

**Affected Code:**

```solidity
/* Unused function parameters */
// ---------- 1 ----------------
function swapExactInput(
        IERC20 inputToken,
        uint256 inputAmount,
        IERC20 outputToken,
        uint256 minOutputAmount,
        uint64 deadline
    )
        public
        revertIfZero(inputAmount)
        revertIfDeadlinePassed(deadline)
        returns (uint256 output){}   // This is unused variable here 'output'
.....
// Possible solutions either remove this returns statement or return the output variable

// ---------- 2 ----------------
function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline
    ) {}
    ......
// Possible solutions either remove this uint64 deadline statement or use the revertIfDeadlinePassed(deadline) function


/* Unused Local variables */
// In Deposit function
uint256 poolTokenReserves = i_poolToken.balanceOf(address(this));
```

**Fix:**
To resolve these warnings, We can either remove the unused parameter if it's not needed or comment out the parameter name if it needs to be kept for interface consistency.

```solidity
// ---------- 1 ----------------
function swapExactInput(
        .....
    )
        external
        revertIfZero(inputAmount)
        revertIfDeadlinePassed(deadline)
        returns (
            //UNUSED VAR
            uint256 output
        )
    {
        .......

        if (outputAmount < minOutputAmount) {
            revert BasicUniswap__OutputTooLow(outputAmount, minOutputAmount);
        }

        _swap(inputToken, inputAmount, outputToken, outputAmount);
        output = outputAmount;
    }

// ---------- 2 ----------------
function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
    ) {}
    ......
```
