# BasicUniswap Smart Contract Audit Report

## Summary of Findings

During the security audit of the BasicUniswap contract, the following vulnerabilities were identified:

- Lack of Slippage Protection in `swapExactOutput` Function
- Incorrect Formula in `getInputAmountBasedOnOutput` Function
- No Pause Mechanism
- Lack of Access Control <!--  If we implement Pause Mechanism    -->
- Potential for Flash Loan Attacks
- Lack of Maximum Cap on Liquidity
- Observation Attack Due to Fixed Swap Count Incentive
- Precision Loss
- Unused Function parameters and Local variable (Severity: Warning)

## Detailed Findings

### Critical Vulnerabilities

#### 1. Lack of Slippage Protection in `swapExactOutput` Function

**Description:**
The `swapExactOutput` function lacks slippage protection, that means the contract can calculate any input amount and deduct it from the user's account. This can lead to significant losses for users due to unexpected and unfavorable exchange rates.

**Severity:**
High

**Affected Code:**

```solidity
function swapExactOutput(
    IERC20 inputToken,
    IERC20 outputToken,
    uint256 outputAmount,
    uint64 deadline
)
    public
    revertIfZero(outputAmount)
    revertIfDeadlinePassed(deadline)
    returns (uint256 inputAmount)
{
    uint256 inputReserves = inputToken.balanceOf(address(this));
    uint256 outputReserves = outputToken.balanceOf(address(this));

    inputAmount = getInputAmountBasedOnOutput(
        outputAmount,
        inputReserves,
        outputReserves
    );

    _swap(inputToken, inputAmount, outputToken, outputAmount);
}
```

**Details:**
In the above function, the contract calculates the input amount required for a given output amount without considering slippage. As a result, the user might end up paying significantly more than expected, leading to a poor user experience and potential financial losses.

**Mitigation:**
To prevent this issue, introduce a parameter for maximum allowable slippage and ensure that the calculated input amount does not exceed this limit. This will protect users from unfavorable exchange rates.

```solidity
function swapExactOutput(
    IERC20 inputToken,
    IERC20 outputToken,
    uint256 outputAmount,
    uint256 maxInputAmount, // Added parameter for slippage protection
    uint64 deadline
)
    public
    revertIfZero(outputAmount)
    revertIfDeadlinePassed(deadline)
    returns (uint256 inputAmount)
{
    uint256 inputReserves = inputToken.balanceOf(address(this));
    uint256 outputReserves = outputToken.balanceOf(address(this));

    inputAmount = getInputAmountBasedOnOutput(
        outputAmount,
        inputReserves,
        outputReserves
    );

    require(inputAmount <= maxInputAmount, "Slippage protection: Input amount exceeds maximum");

    _swap(inputToken, inputAmount, outputToken, outputAmount);
}
```

#### 2. Incorrect Formula in `getInputAmountBasedOnOutput` Function

**Description**
The `getInputAmountBasedOnOutput` function uses an incorrect formula to calculate inputAmount based on outputAmount. This formula may result in an incorrect calculation of input reserves, potentially charging users multiple times the expected amount.
Multiplies by 10000 instead of 1000, returning incorrect input amount which will be invoked in `swapExactOutput` function.

**Severity:**
High

**Affected code**

```solidity
function getInputAmountBasedOnOutput(
    uint256 outputAmount,
    uint256 inputReserves,
    uint256 outputReserves
)
    public
    pure
    revertIfZero(outputAmount)
    revertIfZero(outputReserves)
    returns (uint256 inputAmount)
{
    return
        ((inputReserves * outputAmount) * 10000) /
        ((outputReserves - outputAmount) * 997);
}
```

**Details**
The formula _(inputReserves _ outputAmount _ 10000) / ((outputReserves - outputAmount) _ 997)\* incorrectly calculates inputAmount, potentially overcharging users due to a faulty calculation of input amount.

**Mitigation**
To resolve this issue, adjust the formula to ensure accurate calculation of inputAmount based on outputAmount.

```solidity
function getInputAmountBasedOnOutput(
    uint256 outputAmount,
    uint256 inputReserves,
    uint256 outputReserves
)
    public
    pure
    revertIfZero(outputAmount)
    revertIfZero(outputReserves)
    returns (uint256 inputAmount)
{
    return (inputReserves * outputAmount * 1000) / ((outputReserves - outputAmount) * 997);
}
```

#### 3. No Pause Mechanism

**Description**
The contract lacks a pause mechanism to temporarily halt critical operations such as deposits, withdrawals, and swaps in case of emergencies. Without this feature, the contract remains fully operational at all times, which could exacerbate risks during unforeseen events or vulnerabilities.

**Severity:**
Medium

**Affected code**
This vulnerability applies to all functions and operations within the contract that should be subject to a pause mechanism.

**Details**
A pause mechanism is essential for decentralized applications (DApps) to respond to emergencies promptly. It allows contract administrators or designated roles to halt specific functionalities temporarily, preventing further transactions until the issue is resolved or mitigated. Without a pause mechanism:

- _Risk Amplification:_ In the event of a discovered vulnerability or exploit, the contract cannot be quickly paused to prevent further potential losses or damages.
- _Operational Disruption:_ Unforeseen events, such as hacks or bugs, may require an immediate halt to transactions to mitigate their impact and protect user funds.

**Mitigation**
Implement a pause function that can be activated by authorized administrators or governance entities. Here’s an example of how a pause mechanism can be implemented:

```solidity
address public admin;
bool public paused;

constructor() {
    admin = msg.sender; // Set the contract deployer as the admin
    paused = false; // Contract starts in active state
}

modifier whenNotPaused() {
    require(!paused, "Contract is paused"); // Check if the contract is paused
    _;
}

modifier onlyAdmin() {
    require(msg.sender == admin, "Unauthorized access"); // Restrict function access to admin only
    _;
}

function pauseContract() external onlyAdmin {
    paused = true; // Pause the contract
}

function unpauseContract() external onlyAdmin {
    paused = false; // Unpause the contract
}
```

#### 4. Lack of Access Control <!--  If we implement Pause Mechanism    -->

**Description**
The contract lacks access control mechanisms, allowing any user to interact with all functions.
While this aligns with a fully decentralized design philosophy, it poses risks in scenarios where certain functions should be restricted to specific roles, such as administrative functions for upgrades or emergency stops.
Without access control, unauthorized users could potentially disrupt or manipulate critical contract functionalities.

**Severity:**
Medium

**Affected code**
This vulnerability applies to all functions and operations within the contract that should be subject to a pause mechanism.

**Details**
Access control is crucial for managing permissions and ensuring that only authorized users or contracts can execute sensitive operations. In the absence of access control:

- _Administrative Functions:_ Functions that handle upgrades, emergency stops, or sensitive parameter adjustments could be manipulated by unauthorized users, compromising the contract's integrity.

- _Sensitive Operations:_ Certain operations, such as fund transfers, token minting/burning, or contract state modifications, may require strict access restrictions to prevent unauthorized actions.

**Mitigation**
Implement access control mechanisms using modifiers or conditional statements to restrict function access to specific roles or addresses. Here’s an example of how access control can be implemented:

```solidity
address public admin;

constructor() {
    admin = msg.sender; // Set the contract deployer as the admin
}

modifier onlyAdmin() {
    require(msg.sender == admin, "Unauthorized access"); // Restrict function access to admin only
    _;
}

function emergencyStop() external onlyAdmin {
    // Implement emergency stop logic here
    // Only the admin can call this function to halt critical contract operations
}
```

#### 5. Potential for Flash Loan Attacks

**Description:**
The contract is vulnerable to flash loan attacks, which allow an attacker to manipulate the pool's reserves within a single transaction using a flash loan. Flash loans enable borrowing of assets without collateral as long as the borrowed funds are returned within the same transaction. This vulnerability arises due to the contract's inability to protect against such loan-based manipulations.

**Severity:**
High

**Affected Code:**
This vulnerability affects any function or operation that involves significant token swaps or liquidity adjustments based on external conditions within a single transaction.

**Details:**
Flash loan attacks exploit the composability of smart contracts, allowing an attacker to borrow a large amount of assets, manipulate the contract's state (e.g., token prices, reserves), and then return the borrowed assets, all within the same transaction. Key risks include:

- _Price Manipulation:_ An attacker can manipulate token prices or reserves to their advantage, potentially causing substantial losses to liquidity providers or users.

- _Liquidity Draining:_ The attacker can drain liquidity pools by borrowing large amounts of assets temporarily, affecting the overall market price and stability of the token pairs.

- _Arbitrage Opportunities:_ Flash loans can be used to exploit price differences across decentralized exchanges (DEXs), maximizing gains at the expense of other users or liquidity providers.

**Mitigation:**
To mitigate flash loan attacks, consider implementing the following strategies:

- *Reentrancy Guards: *Ensure that critical state changes occur before any external calls or interactions to prevent reentrancy attacks that might be combined with flash loans.

- _Transaction Integrity Checks:_ Implement checks to ensure that the contract's state remains consistent throughout the transaction, guarding against manipulations introduced by flash loans.

- _Fee Structures:_ Introduce fee structures or time delays for significant operations that involve liquidity adjustments, discouraging rapid and large-scale manipulations.

- _Integration with Security Modules:_ Utilize flash loan detection services or security modules that monitor and alert on anomalous transaction patterns, helping to detect and mitigate potential attacks.

Certainly! Here's how you can describe the observation attack vulnerability for your readme file:

Certainly! Here's how you can describe the lack of maximum cap on liquidity for your readme file:

#### 6. Lack of Maximum Cap on Liquidity

**Description:**
The contract does not impose an upper limit on the amount of liquidity that can be added to the pool. While this is not inherently a vulnerability, it introduces potential risks. Without a maximum cap, the contract is exposed to larger losses in case of bugs, attacks, or unexpected events that could impact the pool's stability and operations.

**Consideration:**
Consider implementing a maximum cap on liquidity to mitigate potential risks and ensure that the pool remains manageable and resilient to unforeseen circumstances. A maximum cap could limit the amount of liquidity that can be added to the pool, thereby reducing the potential impact of large-scale errors or attacks.

**Impact:**
- **Risk Exposure:** Without a maximum cap, the contract is susceptible to significant losses in the event of a bug, exploit, or market volatility that leads to a sudden imbalance in liquidity.
  
- **Operational Stability:** Introducing a maximum cap helps maintain operational stability by ensuring that the pool remains within manageable limits, reducing the complexity of potential recovery processes in case of emergencies.

**Mitigation:**
Consider implementing a function to set and enforce a maximum cap on liquidity, balancing between operational flexibility and risk management. This function should carefully assess the implications on liquidity providers and trading activities while safeguarding against excessive exposure to unforeseen risks.

**Example Implementation:**
```solidity
uint256 public constant MAX_LIQUIDITY = 1_000_000_000 * 1e18; // Example maximum cap set to 1 billion tokens

/**
 * @notice Adds liquidity to the pool, enforcing a maximum cap on the total liquidity.
 * @param amount The amount of liquidity tokens to mint.
 */
function addLiquidity(uint256 amount) external {
    require(totalLiquidityTokenSupply() + amount <= MAX_LIQUIDITY, "Exceeds maximum liquidity cap");
    
    // Rest of the add liquidity function logic
    // ...
}
```

**Enhancement:**
Implementing a maximum cap on liquidity ensures responsible management of the pool's size and limits potential losses during adverse conditions. It also demonstrates proactive risk management and enhances overall confidence in the contract's reliability and security.


#### 7. Observation Attack Due to Fixed Swap Count Incentive

**Description:**
The `_swap` function provides an incentive to the caller after every `SWAP_COUNT_MAX` (e.g., 10) swap transactions. However, this fixed-count incentive system is susceptible to observation attacks. An observation attack occurs when an attacker monitors the contract's transactions and internal state to predict and exploit the timing of incentive rewards.

**Severity:**
Medium

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
The `_swap` function increments a global `swap_count` variable for every transaction and transfers an additional token to the caller when `swap_count` reaches `SWAP_COUNT_MAX`. This predictable incentive mechanism allows an observer to track the state changes and predict the timing of the bonus token transfer before it occurs.

**Impact:**

- **Predictable Rewards:** An attacker can exploit this predictability by strategically timing their interactions with the contract to receive the bonus token intended for the 10th transaction.
- **Manipulation of User Behavior:** The fixed-count incentive may influence user behavior in unintended ways, potentially skewing trading patterns or causing congestion during specific periods.

**Mitigation:**
To mitigate observation attacks and ensure fair distribution of incentives, consider implementing a user-specific counter mechanism instead of a global `swap_count`. This approach would provide the bonus token to individual users after they complete a specific number of swap transactions, irrespective of other users' activities.

```solidity
mapping(address => uint256) private userSwapCounts;

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

    userSwapCounts[msg.sender]++;
    if (userSwapCounts[msg.sender] >= SWAP_COUNT_MAX) {
        userSwapCounts[msg.sender] = 0;
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
    if (userSwapCounts[msg.sender] == 0) {
        outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
    }
}
```

**Enhancement:**
By implementing a user-specific counter (`userSwapCounts`), the contract ensures that each user receives the incentive after completing their own set of swap transactions. This approach enhances fairness and reduces the susceptibility to observation attacks, maintaining the integrity and predictability of the incentive system.


#### 8. Precision Loss

**Description:**
The contract uses integer arithmetic for its calculations, which is common in Solidity due to the lack of floating-point support. However, this can lead to rounding errors. Over time, especially with frequent small trades, these rounding errors can accumulate, potentially resulting in significant discrepancies in the contract’s balance and the expected outcomes for users.

**Impact:**
- **Rounding Errors:** Integer arithmetic can result in rounding down values, causing users to receive slightly less than expected in each transaction. While each rounding error might be small, they can accumulate over time, leading to significant differences.
- **User Trust:** Accumulated rounding errors can affect user trust, especially if users consistently receive less than they anticipate from trades.
- **Contract Balance:** The contract’s internal balance might not accurately reflect the expected values due to these rounding errors.

**Mitigation:**
To minimize the impact of precision loss, the contract can implement strategies to handle rounding errors more gracefully:

1. **Adjust Calculations:**
   - Include a small buffer in calculations to account for rounding errors.
   - Use techniques such as rounding up or down based on specific conditions to minimize the overall impact.

2. **Audit and Test:**
   - Thoroughly audit and test the contract to identify and understand where rounding errors occur.
   - Implement test cases that simulate frequent small trades to observe and mitigate the accumulation of rounding errors.

3. **User Communication:**
   - Clearly communicate to users about the potential for rounding errors and the measures in place to handle them.

**Enhancement:**
By implementing these mitigation strategies, the contract can reduce the impact of precision loss, providing a more accurate and reliable trading experience for users. This demonstrates a commitment to maintaining the integrity and fairness of the contract's operations, ultimately enhancing user trust and confidence.

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
