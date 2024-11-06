# Pancakeswap-IFOPool-smart-contract-Reviews

The IFOPool smart contract is a contract made to manage funds for a type staking pool, known as an Initial Farm Offering (IFO) pool. Users can deposit tokens, accumulate shares based on their deposits, and withdraw with possible penalties depending on the time of their actions.

The pragma solidity 0.6.12; specifies that this contract should be compiled with Solidity version 0.6.12, ensuring compatibility with that version.

The contract makes use of external contracts which it is importing from OpenZeppelin contracts ( Ownable, Pausable, SafeMath, SafeERC20 ) and an Interface (IMasterChef) 

#### OWNABLE.sol :
This provides access to the OpenZeppelin Ownable contract, which provides authorization and ownership to the IFOPool contract. The Ownable contract grants ownership to the deploying address giving them permission to call onlyOwner functions.

#### PAUSABLE.sol :
The Pausable contract allows the owner to pause all contract functions annotated with whenNotPaused. It’s useful for security as it allows the owner to stop all critical contract operations in case of emergencies.

#### SAFEMATH.sol : 
Seeing as the contract is below 0.8.0, the contract does not have default overflow/underflow checks, The SafeMath library adds overflow protection to arithmetic operations (addition, subtraction, multiplication, and division). In IFOPool, SafeMath is used in several calculations to ensure safety from overflow.

#### SAFEERC20.sol :
This library ensures that ERC20 token transfers (depositing and withdrawing) are handled safely, preventing unexpected failures. By wrapping standard ERC20 calls in SafeERC20 functions, the contract avoids transfer failures from token reverts.

### INTERFACES
IMasterChef.sol : IMasterChef is an interface that interacts with the staking contract. This interface contains the functions required to function, effectively allowing the IFOPool to stake user funds in the external contract.

Usage in IFOPool:--

Staking operations 
```solidity
function enterStaking(uint256 _amount) external;
```

Unstaking: function 
```solidity
leaveStaking(uint256 _amount) external;
```

Emergency withdrawal: 
```solidity
function emergencyWithdraw(uint256 _pid) external;
```

Reward checking: 
```solidity
function pendingCake(uint256 _pid, address _user) external view returns (uint256);
```


### CONTRACT VARIABLES
```solidity
struct UserInfo {
    uint256 shares;
    uint256 lastDepositedTime;
    uint256 cakeAtLastUserAction;
    uint256 lastUserActionTime;
}
```
This struct holds the user's information:

shares: Represents the user's share in the pool.

lastDepositedTime: Timestamp of the user's last deposit, crucial for calculating early withdrawal fees.

cakeAtLastUserAction: Tracks the amount of tokens at the last user action (deposit/withdraw).

lastUserActionTime: Timestamp of the last user action, useful for calculating fees or penalties based on inactivity.

```solidity
struct UserIFOInfo {
    uint256 lastActionBalance;
    uint256 lastValidActionBalance;
    uint256 lastActionBlock;
    uint256 lastValidActionBlock;
    uint256 lastAvgBalance;
}
```

This struct is specific to IFOs and maintains a history of user actions and balances:

- lastActionBalance and lastValidActionBalance store the user’s balance for tracking purposes within IFO validity periods.
- lastActionBlock and lastValidActionBlock track the block numbers associated with these actions.
- lastAvgBalance calculates the average balance within the IFO period, often used to determine eligibility for rewards in an IFO.

```solidity
enum IFOActions {
    Deposit,
    Withdraw
}
```

Enumerates possible actions, allowing the _updateUserIFO function to distinguish between deposits and withdrawals.


### State Variables:

```IERC20 public immutable token:``` The main token used in the pool (CAKE).

```IERC20 public immutable receiptToken:``` The receipt token issued by MasterChef upon staking to represent user's stake in the pool.

```IMasterChef public immutable masterchef:``` MasterChef contract interface, where staking occurs.

```mapping(address => UserInfo) public userInfo:``` Tracks each user's staking details.

```mapping(address => UserIFOInfo) public userIFOInfo:``` Tracks each user's IFO-specific information.

```uint256 public startBlock and uint256 public endBlock:``` Define the block range for IFO eligibility.

```uint256 public totalShares:``` Tracks the total shares issued to all users, representing pool ownership.

```address public admin:``` Administrator’s address, used for exclusive admin functionality.

```address public treasury:``` Treasury address to collect fees.

```Constants for maximum fees and other parameters, like:```

```MAX_PERFORMANCE_FEE, MAX_CALL_FEE, MAX_WITHDRAW_FEE, MAX_WITHDRAW_FEE_PERIOD:``` Define maximum fees and withdrawal limits for the contract.


### Constructor

- Initializes contract with core parameters
- Sets up token approvals for MasterChef
- Ensures start block is in future
- Ensures end block is after start block
- Assigns immutable token addresses
- Sets admin and treasury addresses
- Sets IFO period blocks
- Approves MasterChef for token transfers

The constructor initializes the contract and performs initial checks on startBlock and endBlock. Additionally, it grants infinite approval of the token to the MasterChef contract, allowing IFOPool to stake the deposited tokens without needing re-approvals. This ensures that the pool can stake user deposits immediately without further authorization.

## Modifiers

#### ONLYADMIN:

```solidity
modifier onlyAdmin() {
    require(msg.sender == admin, "admin: wut?");
    _;
} 
```

- This restricts functions it's applied to, to be called only by the contract owner.

#### NOTCONTRACT:

```solidity
modifier notContract() {
    require(!_isContract(msg.sender), "contract not allowed");
    require(msg.sender == tx.origin, "proxy contract not allowed");
    _;
}
```

- Prevents smart contracts or proxies from interacting with this contract by ensuring msg.sender is not a contract and that it directly originates from the user’s wallet. Makes use of internal helper function ```_isContract``` that returns boolean value to verify is caller is contract.

```solidity 
modifier whenNotPaused
``` 

Prevents deposits when contract is paused


```solidity
modifier notContract
``` 

Prevents contract interactions

## Events

```solidity
event Pause();
```
This event is logged whenever the contract’s operational state is paused.

```solidity
event Unpause();
```
This event is logged whenever the contract’s operational state is Unpaused.

```solidity
event Deposit(address indexed sender, uint256 amount, uint256 shares, uint256 lastDepositedTime);
```
Logs details whenever a user deposits funds into the contract. This includes the user’s address, the deposited amount, the shares received, and the time of the deposit.

```solidity
event Withdraw(address indexed sender, uint256 amount, uint256 shares);
```
Logs whenever a user withdraws funds, recording the user’s address, the amount withdrawn, and the shares burned.

```solidity
event Harvest(address indexed sender, uint256 performanceFee, uint256 callFee);
```
Logs when a harvest action is executed. It records the caller’s address and the performance and call fees applied during the harvest.

```solidity
event UpdateEndBlock(uint256 endBlock);
```
Logs whenever the end block for a specific function or staking period is updated.

```solidity
event ZeroFreeIFO(address indexed sender, uint256 currentBlock);
```
Logs when a user or admin interacts with the contract in a way that zeroes out or terminates a free Initial Farm Offering (IFO) opportunity.

```solidity
event UpdateStartAndEndBlocks(uint256 startBlock, uint256 endBlock);
```
Logs updates to the start and end blocks that define the active period for certain contract functionalities, like staking or reward distribution.

```solidity
event UpdateUserIFO(
    address indexed sender,
    uint256 lastAvgBalance,
    uint256 lastActionBalance,
    uint256 lastValidActionBalance,
    uint256 lastActionBlock,
    uint256 lastValidActionBlock
);
```
Logs specific details about a user’s interactions with the IFO. This includes the user’s last average balance, the last and last valid action balances, and the corresponding block numbers.


## FUNCTIONS

#### DEPOSIT() FUNCTION

```solidity
  function deposit(uint256 _amount) external whenNotPaused notContract
```

* Deposits funds into the Cake Vault.
* Only possible when contract not paused as a result of the whenNotPaused modifier. 
* Only callable by EOAs and not contracts
* param - `_amount`: number of tokens to deposit (in CAKE)
* Emits the Deposit event

The deposit function first validates the amount to be deposited. It then calculates the current pool balance and transfers the tokens from the user. The function also calculates the shares as follows:

The shares are equal to the amount for the first deposit. For subsequent deposits, the shares are calculated as (amount * totalShares) / pool.
The function then updates the user information by adding the shares, updating the deposit timestamp, and updating the last action metrics.
It also updates the total shares and IFO tracking, stakes the tokens in MasterChef, and emits a deposit event.


#### GETUSERCREDIT() FUNCTION

```solidity
function getUserCredit(address _user) external view returns (uint256 avgBalance)
```

The contract uses this function to assess a user’s eligibility or participation level in IFOs, ensuring users with higher or sustained balances have better opportunities in IFO allocations.

The getUserCredit function calculates a user’s average balance based on their historical actions, which is relevant to Initial Farm Offering (IFO) eligibility or participation. This average balance is important as it can influence the user’s allocation in IFOs based on their pool contribution over time. The function first retrieves the `UserIFOInfo` data for the given user, then checks if an IFO is currently active by calling `isIFOAvailable()`. If an IFO is available, it calls `_calculateAvgBalance` with the user's historical balance and action data to determine the latest average balance. If an IFO is not available, it sets avgBalance to 0. `_calculateAvgBalance` uses parameters like `lastActionBlock` and `lastValidActionBlock` to calculate a time-weighted balance average that fairly represents the user's stake in the pool. The function returns the calculated average balance (or 0 if no IFO is active).


#### WITHDRAW() FUNCTION

```function withdraw(uint256 _shares) public notContract```

Purpose: Allows a user to withdraw a specified amount of shares from the pool, applies any applicable fees, and updates IFO-related data.

* Dependents :

`available()` - Custom logic for how much the vault allows to be borrowed
`leaveStaking()` - Unstaking function from the IMasterChef Interface
`balanceOf()` - Calculates the total underlying tokens, It includes tokens held by the contract and held in MasterChef

- Checks that _shares is greater than zero, meaning the user is attempting to withdraw a positive amount.
- Ensures the user has enough shares to complete the withdrawal by comparing _shares with user.shares.

Determines the amount of underlying token (currentAmount) corresponding to the _shares the user wants to withdraw, `(balanceOf() * _shares) / totalShares`.
The withdrawal process involves several steps. First, it stores the amount to be withdrawn (currentAmount) for IFO tracking purposes. It then updates the user's and total share balances by deducting the withdrawn shares. If the pool's balance is insufficient to cover the withdrawal, it withdraws the deficit from the IMasterChef staking pool and adjusts the currentAmount accordingly. An early withdrawal fee is applied if the withdrawal occurs within a certain period of the user's last deposit, and the fee is transferred to the treasury. The user's balance information is updated, including their CAKE balance and last action timestamp. The IFO data is also updated to reflect the withdrawal. Finally, the currentAmount is transferred to the user, and a Withdraw event is emitted to log the transaction details.

The `WithdrawAll` function depend on the `withdraw` function to perform.


#### WITHDRAWALL() FUNCTION

```solidity
function withdrawAll() external notContract
```

This provides users with a simple way to withdraw all of their holdings from the pool.

The `withdrawAll` function enables users to withdraw their entire share of funds from the pool in a single action. It achieves this by calling the `withdraw` function with the user's total shares, which then transfers the user's proportional amount of CAKE or other tokens based on their share ownership. This function is protected by the notContract modifier, ensuring that only non-contract addresses can call it, thereby preventing automated contracts or bots from accessing it.


#### EMERGENCYWITHDRAWALL() FUNCTION

```solidity
function emergencyWithdrawAll() external notContract
```

Useful for users who need immediate access to their funds in case of emergencies.

* Dependents : 
`_zeroFreeIFO();` - sets userIFOInfo to initial state
`withdrawV1(userInfo[msg.sender].shares);` - original Withdraws implementation from funds, the logic same as Cake Vault withdraw
    notice this function visibility change to internal, call only be called by `emergencyWithdrawAll` function
    param _shares: Number of shares to withdraw

The `emergencyWithdrawAll` function enables users to withdraw all their funds in emergency situations, simultaneously resetting their IFO-related status and balance records. It executes by first calling _zeroFreeIFO() to clear IFO data, ensuring no residual IFO eligibility or status. Then, it calls withdrawV1 with the user's total shares to facilitate the withdrawal, which may bypass or modify certain conditions like fee structures or lock periods. The notContract modifier ensures only non-contract addresses can initiate this function, preventing automated entities from doing so.


#### HARVEST() FUNCTION

```solidity
function harvest() external notContract whenNotPaused
```

* External
* Cannot be called by contracts 
* Only callable whenNotPaused
* Emits the `Harvest` event

This function reinvests any CAKE tokens earned by the pool back into the MasterChef contract.

The harvest function is designed to reinvest CAKE tokens earned by the pool back into the MasterChef contract, maximizing returns. It can only be called by external addresses when the contract is not paused, ensuring owner control and preventing automated contract exploitation. The function first checks the current CAKE balance, then triggers a harvest action without withdrawing staked tokens to pull in pending rewards. The earned amount is calculated, and if positive, performance and call fees are applied and distributed to the treasury and the caller, respectively. The remaining tokens are then staked back into MasterChef to continue earning rewards, and the last harvest time is updated.
The Harvest event logs the caller, performance fee, and call fee.

* Dependents :

`available()` - Custom logic for how much the vault allows to be borrowed
`safeTransfer()` - Openzeppelin safeTransfer function
`_earn()` - Deposits tokens into MasterChef to earn staking rewards


#### setAdmin() FUNCTION

```solidity
function setAdmin(address _admin) external onlyOwner
```

* External
* Sets admin address
* Only callable by the contract owner.
* Params - `address _admin` address of new admin

This function enables the contract owner to designate or modify the admin address, ensuring it is not a zero address. It is exclusively callable by the owner and serves to define or update the admin responsible for executing administrative tasks within the contract.


#### setTreasury() FUNCTION

```solidity
function setTreasury(address _treasury) external onlyOwner
```

* External
* Sets treasury address
* Only callable by the contract owner.
* Params - `address _treasury` address to the new treasury

Sets the treasury address, where funds or fees collected by the contract might be sent.

This function can only be executed by the contract owner and ensures the address isn't zero. It allows the owner to update the treasury address if necessary.


#### setPerformanceFee() FUNCTION

```solidity
function setPerformanceFee(uint256 _performanceFee) external onlyAdmin
```

* External
* Sets performance fee
* Only callable by the contract admin.

This function allows the contract admin to set a performance fee, which is a percentage-based fee levied on profits. The admin can adjust this fee as needed, but it is restricted from exceeding the predefined maximum limit, MAX_PERFORMANCE_FEE. This ensures that the performance fee remains within a reasonable and controlled range, preventing excessive fees from being imposed. By enabling the admin to adjust the performance fee within these defined limits, the function provides flexibility and adaptability in managing the contract's fee structure.


#### setCallFee() FUNCTION

```solidity
function setCallFee(uint256 _callFee) external onlyAdmin 
```

* External
* Sets call fee
* Only callable by the contract admin.

The `setCallFee` function is designed to set a call fee, which can be applied to transactions initiated by users or other functions. This function is restricted to the admin and ensures that the fee does not exceed the maximum call fee, as defined by MAX_CALL_FEE. The purpose of this function is to allow for the setting of a fee within defined bounds, which can be used to incentivize or cover certain operational costs.


#### setWithdrawFee() FUNCTION

```solidity
function setWithdrawFee(uint256 _withdrawFee) external onlyAdmin 
```

* External
* Sets withdraw fee
* Only callable by the contract admin.

This function allows the admin to set a withdrawal fee, which is a charge applied to users when they withdraw funds from the contract. The admin can adjust this fee within a predefined limit, MAX_WITHDRAW_FEE, ensuring that the fee remains reasonable and controlled.



#### setWithdrawFeePeriod() FUNCTION

```solidity
function setWithdrawFeePeriod(uint256 _withdrawFeePeriod) external onlyAdmin
```

* External
* Sets withdraw fee period
* Only callable by the contract admin.

This function sets the duration during which a withdrawal fee is applicable after a deposit, accessible only to the admin and ensuring the period does not surpass MAX_WITHDRAW_FEE_PERIOD. Its purpose is to discourage short-term withdrawals by imposing a fee within a defined timeframe.



#### updateStartAndEndBlocks() FUNCTION 

```solidity
function updateStartAndEndBlocks(uint256 _startBlock, uint256 _endBlock) external onlyAdmin 
```

* It allows the admin to update start and end blocks
* This function is only callable by owner.
* _startBlock: the new start block
* _endBlock: the new end block
* Evnts - Emits `UpdateStartAndEndBlocks(_startBlock, _endBlock)`

This function is used to update the start and end blocks of a specific activity or contract functionality, such as a staking or IFO period. It is restricted to the admin, with checks ensuring the start block is in the future and precedes the end block. This function is used to control when a certain feature (e.g., rewards program) is active by setting valid block ranges.


#### updateEndBlock() FUNCTION

```solidity
function updateEndBlock(uint256 _endBlock) external onlyAdmin 
```

* It allows the admin to update end block
* This function is only callable by owner.
* _endBlock: the new end block
* Emits the `UpdateEndBlock` event

This function enables the admin to prolong the end block of a specific period, such as staking rewards or IFO, as long as the new end block is in the future. This admin-only functionality provides flexibility in extending the duration of a feature if necessary.


#### emergencyWithdraw() FUNCTION

```solidity
function emergencyWithdraw() external onlyAdmin
``` 

* External
* Withdraws from MasterChef to Vault without caring about rewards.
* EMERGENCY ONLY. Only callable by the contract admin.
* Interacts with `emergencyWithdraw` function from `IMasterChef` Interface

This function allows the admin to withdraw funds from MasterChef to the vault in emergency situations, disregarding rewards. It can only be called by the admin and will pause the contract if it's not already paused. Its primary use is in emergency scenarios where funds need to be recovered.



#### inCaseTokensGetStuck() FUNCTION

```solidity
function inCaseTokensGetStuck(address _token) external onlyAdmin 
```

* Withdraw unexpected tokens sent to the Cake Vault
* Callable by only the Admin
* Intercats with IERC20 `safeTransferFrom` and `balanceOf` functions

This function allows the admin to recover tokens that were mistakenly sent to the vault, excluding the primary deposit and receipt tokens. It ensures that only non-primary tokens are retrieved, safeguarding against accidental token transfers to the vault.



#### pause() FUNCTION

```solidity
function pause() external onlyAdmin whenNotPaused
```

* External
* Triggers stopped state.
* Only possible when contract not paused.
* Callable only by Admin
* Emis the Pause event

Provides an emergency stop mechanism for the contract.

This function pauses the contract's operations, effectively halting most functions, and can only be called by the admin when the contract is not already in a paused state.


#### unpause() FUNCTION


* Returns to normal state
* Only possible when contract is paused.
* Emits the Unpause event

function unpause() external onlyAdmin whenPaused

This function is used to resume the normal operations of the contract after a pause. It can only be executed when the contract is in a paused state and is restricted to the admin. The primary use of this function is to allow the admin to re-enable the contract functions after an emergency pause.



## HELPER FUNCTIONS

#### AVAILABLE() FUNCTION

```solidity
function available() public view returns (uint256)
```

* Public
* Returns uint256

This function returns the balance of CAKE tokens held in the contract. It provides the current balance of CAKE in the contract, which is useful for operations like determining how much CAKE is available for reinvestment or withdrawal. Returns the balance in uint256 OF token.balanceOf(address(this)).

#### _EARN() FUNCTION

```solidity
function _earn() internal
```

* Internal

This function Deposits tokens into MasterChef to earn staking rewards. The current CAKE balance is retrieved via `available()`. If bal is greater than zero, the entire balance is staked by calling enterStaking(bal) on the MasterChef contract. this function compounds rewards by consistently increasing the stake in MasterChef.


#### BALANCEOF() FUNCTION

```solidity
function balanceOf() public view returns (uint256)
```

* Public 
* Returns `uint256` in balance

- This function calculates the total CAKE held by the contract, including staked CAKE in MasterChef.
- Calls `IMasterChef(masterchef).userInfo(0, address(this))` to get the staked CAKE amount in MasterChef.
- Adds this amount to the CAKE balance of the contract `(token.balanceOf(address(this)))`.
- Returns uint256 as the sum represents the total CAKE balance available to the contract.


#### _isContract() FUNCTION

```solidity
function _isContract(address addr) internal view returns (bool)
```

* Params - address _addr (address to be checked for code)
* Returns - `true` or `false` if the address is a contract address or not

This function is used to verify if a given address is a contract by checking the size of the code at that address.

The `_isContract()` function uses inline assembly to determine if a given address is a contract. It does this by checking the size of the code stored at the address ``size := extcodesize(_addr)``. If the size is greater than 0, it returns true, indicating that the address is a contract. This function is used in the notContract modifier, which restricts access to certain functions from contract addresses.


#### calculateHarvestCakeRewards() FUNCTION

```solidity
function calculateHarvestCakeRewards() external view returns (uint256)
```

* External
* Returns `uint256` - the amount of CAKE rewards available to harvest

This function calculates the current pending CAKE rewards that the contract has earned but hasn’t yet claimed from the MasterChef contract.

This function calculates the current pending CAKE rewards that the contract has earned but hasn’t yet claimed from the MasterChef contract. It does this by calling `IMasterChef(masterchef).pendingCake(0, address(this))`, where the 0 represents the pid (pool ID) of the main CAKE staking pool. The function returns the amount of CAKE rewards available for harvesting, which is useful for users or other contracts to determine the amount of CAKE that could be reinvested immediately.


#### calculateTotalPendingCakeRewards() FUNCTION


* External
* Returns `uint256` - the total CAKE rewards

```solidity
function calculateTotalPendingCakeRewards() external view returns (uint256)
```

This function calculates the total amount of CAKE rewards, both those already available in the contract and those still pending in MasterChef.

This function calculates the total CAKE rewards by summing the contract's current CAKE balance, obtained through the `available()` function, and the pending CAKE rewards from MasterChef, retrieved via `IMasterChef(masterchef).pendingCake(0, address(this))`. The result is the total CAKE reward balance, which is useful for determining the overall reward amount without the need for immediate harvesting.


#### totalShares() FUNCTION

* External
* Returns `uint256` - the total shares

```solidity
function totalShares() external view returns (uint256)
```

Purpose: This function returns the total number of shares issued to users in the pool.

This function returns the total number of shares issued to users in the pool, which is stored in the state variable `_totalShares`. This information is crucial for calculating the relative ownership of shares in the pool, particularly for distributing profits and determining user balances.



#### calculateSharePrice FUNCTION

```solidity
function calculateSharePrice() external view returns (uint256)
```

* External
* Returns `uint256` the price per share

This function calculates the current price of each share in terms of CAKE tokens.

The `calculateSharePrice` function determines the current price of each share in CAKE tokens. If there are no shares, it defaults to a one-to-one price in Wei (1e18). Otherwise, it calculates the share price by dividing the balance by the total shares, scaled by 1e18 for precision in Wei. This function is useful for estimating the value of each user's share in the pool.


#### _isIFOAvailable() FUNCTION

```solidity
function _isIFOAvailable() internal view returns (bool) 
```

* Internal
* Returns a boolen value

The `_isIFOAvailable` function determines if the Initial Farm Offering (IFO) is currently active by comparing the current block number to the start block of the IFO. It returns `true` if the current block is after the start block, indicating the IFO is available, and `false` otherwise. This function is used to conditionally execute code that should only run during the IFO period.


#### _isValidActionBlock() FUNCTION

```solidity
function _isValidActionBlock() internal view returns (bool) 
```

* Internal
* This function only be called to judge whether to update last action block.
* Only block number between start block and end block to update last action block.
* Returns `bool`

Prevents actions from updating IFO information once the IFO period has ended.

This function determines if the current block is within the valid range for recording a user's IFO actions, which is between the startBlock and endBlock. It returns true if the current block is within the IFO period and false otherwise.


#### _calculateAvgBalance() FUNCTION

```solidity
function _calculateAvgBalance(
    uint256 _lastActionBlock,
    uint256 _lastValidActionBlock,
    uint256 _lastActionBalance,
    uint256 _lastValidActionBalance,
    uint256 _lastAvgBalance
) internal view returns (uint256 avgBalance)
```

* Params:
- _lastActionBlock – The block when the user last deposited or withdrew.
- _lastValidActionBlock – Last valid action block within the IFO period.
- _lastActionBalance – User’s balance at the last action.
- _lastValidActionBalance – User’s balance at the last valid action within the IFO.
- _lastAvgBalance – The last recorded average balance for the user.

* Internal
* Returns `uint256` avgBalance

This function calculates the user’s latest average balance during the IFO period.

The function first checks if the last action block is after the end block. If it is, the function returns the last average balance without further calculations. For a new IFO participant, the function initializes the last valid action block to the start block and the last average balance to 0. The function then calculates the average balance over the period between the last action and the current block, up to the end block. This updated average balance is used for determining the user’s contribution to the IFO.


#### _updateUserIFO() FUNCTION

```solidity
function _updateUserIFO(uint256 _amount, IFOActions _action) internal
```

* Internal
* Params:
- _amount: the cake amount that need be add or sub
- _action: IFOActions enum element
* Events
- Emits `UpdateUserIFO()` event


This function pdates a user’s IFO information, including balances, action blocks, and average balance, based on whether the user is depositing or withdrawing funds.
Called whenever a user deposits or withdraws funds during the IFO, ensuring accurate record-keeping and contribution tracking.

The `_updateUserIFO` function initiates by calculating the user's latest average balance using `_calculateAvgBalance` if an IFO is currently available. It then updates the user's action balance based on the type of action being performed. For a withdrawal, it subtracts the specified amount from the `lastActionBalance`, whereas for a deposit, it adds the amount to `lastActionBalance`. If the current block is within the IFO period, the function updates the `lastValidActionBalance` and `lastValidActionBlock`. Subsequently, it records the updated `lastAvgBalance` and sets `lastActionBlock` to the current block. Finally, the function emits the UpdateUserIFO event, which logs all the updated IFO-related data for tracking and monitoring purposes.
