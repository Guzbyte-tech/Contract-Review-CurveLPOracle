A review of the `CurveLPOracle` and `CurveLPOracleFactory` contract.

---

# Contract Review: `CurveLPOracle` and `CurveLPOracleFactory`

## Table of Contents

1. [Introduction](#introduction)
2. [Overview of Curve Pools and StableSwap Invariant](#overview-of-curve-pools-and-stableswap-invariant)
3. [Purpose and Functionality of the Contract](#purpose-and-functionality-of-the-contract)
4. [Contract: CurveLPOracleFactory](#contract-curvelporaclefactory)
   - 4.1 [State Variables](#state-variables-in-curvelporaclefactory)
   - 4.2 [Functions](#functions-in-curvelporaclefactory)
5. [Contract: CurveLPOracle](#contract-curvelporacle)
   - 5.1 [State Variables](#state-variables-in-curvelporacle)
   - 5.2 [Functions](#functions-in-curvelporacle)
6. [Security and Gas Optimization Considerations](#security-and-gas-optimization-considerations)
7. [Conclusion](#conclusion)

---

### 1. Introduction

The `CurveLPOracle` and `CurveLPOracleFactory` contracts are part of a decentralized finance (DeFi) protocol for obtaining price data from Curve liquidity pools (LPs). These contracts are developed by the Dai Foundation and provide an on-chain oracle for fetching and managing prices of Curve LP tokens, which represent a user's share in a Curve pool.

### 2. Overview of Curve Pools, Oracle and StableSwap Invariant

#### **Oracles**
In blockchain and DeFi, an oracle is a service that supplies external data (such as asset prices) to smart contracts. Since blockchains are isolated from the outside world, oracles are essential for pulling real-world information, such as cryptocurrency prices, fiat currency exchange rates, or weather data, into smart contracts. In this case, CurveLPOracle provides real-time price data for LP tokens, based on underlying pool assets and their current virtual price in Curve.

#### **Curve Pools**
Curve pools are DeFi liquidity pools designed for stablecoins or similar assets like USDT, USDC, DAI etc. They allow users to swap between assets with minimal slippage due to their specialized design, which leverages the StableSwap invariant. Curve pools consist of two or more stablecoins (or similar assets) and offer yield opportunities by allowing users to provide liquidity and earn fees from swaps.

#### **StableSwap Invariant**
The StableSwap invariant is a mathematical formula designed by Curve Finance to maintain liquidity and ensure low slippage when swapping stablecoins. The formula defines the pool's asset reserves and pricing, maintaining stable prices around the equilibrium and creating efficient trades between similar assets.

In this contract, the Curve pool's "virtual price" is used to determine the value of the LP tokens. The virtual price represents the price of 1 LP token in terms of the pool's underlying assets. This virtual price can change based on the pool's total liquidity and balance, indirectly accounting for the StableSwap invariant’s effects on LP token value.

### 3. Purpose and Functionality of the Contract

The primary goal of the `CurveLPOracle` contract is to serve as an oracle for Curve LP tokens, periodically updating and broadcasting the token’s price by aggregating prices of each asset in the Curve pool. The `CurveLPOracleFactory` contract is a factory that deploys instances of `CurveLPOracle` for different Curve pools.

Key functionalities include:
- **Fetching and Updating Prices**: Fetches price data from external oracles for each asset in the Curve pool, calculates the LP token price, and broadcasts it.
- **Access Control**: Limits access to certain functions to only authorized addresses (admins or whitelisted users).
- **Interval Updates**: Enforces a minimum interval between price updates, to manage data flow and gas costs.
- **Reentrancy Protection**: Protects the contract from reentrant calls to prevent unauthorized access or liquidity pool manipulation.

---

### 4. Contract: CurveLPOracleFactory

The `CurveLPOracleFactory` contract deploys instances of `CurveLPOracle`, setting up an oracle for each Curve pool with customized parameters.

#### 4.1 State Variables in CurveLPOracleFactory

- **`ADDRESS_PROVIDER`**: An `AddressProviderLike` interface that holds the address provider contract for Curve, which allows this factory to access Curve’s registry and fetch addresses of available pools. It's set as `immutable`, meaning it cannot be changed after deployment.

#### 4.2 Functions in CurveLPOracleFactory

#### Constructor
```constructor(address addressProvider)
```
- **`addressProvider`**: The address of an `AddressProviderLike` contract. This provider is a utility contract in Curve that allows the contract to access the registry (a contract that keeps ttrack of all the actove liquididty pools on the platform) of Curve pools and query specific details about them. The `ADDRESS_PROVIDER` variable is set to this address, making it available to other functions within `CurveLPOracleFactory`.

#### `build` Function
```
function build(
    address _owner,
    address _pool,
    bytes32 _wat,
    address[] calldata _orbs,
    bool _nonreentrant
) external returns (address orcl)
```
This function creates and deploys a new `CurveLPOracle` instance.

- **`_owner`**: The address to be granted admin privileges (`wards`) mapping for the new `CurveLPOracle`. This address can perform sensitive actions such as stopping and starting updates, adjusting the update frequency (`hop`), and managing whitelisted addresses.
- **`_pool`**: The address of the Curve liquidity pool for which the oracle will provide pricing. This pool address is used to get information such as virtual price and supported tokens.

- **`_wat`**: A `bytes32` identifier for the asset whose price is being tracked by the oracle. This label is typically used in DeFi protocols to distinguish between different assets.

- **`_orbs`**: An array of addresses of external oracles, each corresponding to a token in the Curve pool. Each oracle in `_orbs` provides a price for a specific token, and the length of this array should match the number of tokens in the pool.

- **`_nonreentrant`**: A boolean flag indicating whether the `CurveLPOracle` should prevent reentrant calls to the Curve pool when updating liquidity. This is a security measure to ensure the oracle doesn’t interact with a pool that could allow malicious reentrant calls.


---

### 5. Contract: CurveLPOracle

The `CurveLPOracle` contract holds the main logic for fetching, updating, and providing prices of Curve LP tokens.

#### 5.1 State Variables in CurveLPOracle

- **Authorization Mappings**
  - `wards`: Stores addresses with admin privileges.
  - `bud`: Stores whitelisted addresses for restricted functions.

- **Core Oracle Settings**
  - `src`: The address of the LP token associated with the Curve pool.
  - `stopped`: A flag for stopping updates to the oracle.
  - `hop`: The minimum time interval between updates.
  - `zph`: The next timestamp when updates are allowed.

- **Price Feed Storage**
  - `cur`: Stores the current price and its validity status.
  - `nxt`: Stores the next calculated price and its validity status.
  - `orbs`: Array of addresses for each token's price oracle in the Curve pool.

- **Pool Information**
  - `pool`: The Curve pool’s address.
  - `wat`: Identifier for the token being tracked.
  - `ncoins`: The number of assets in the Curve pool.
  - `nonreentrant`: A flag for reentrancy protection.

#### 5.2 Functions in CurveLPOracle

- **Authorization Functions**
##### `rely` and `deny` Functions
```
function rely(address _usr) external auth
function deny(address _usr) external auth
```
These functions grant or revoke admin privileges.

- **`_usr`**: The address to be granted (`rely`) or revoked (`deny`) admin privileges. Only admins can call these functions (enforced by the `auth` modifier).

#### State Variables and Initialization

##### Constructor
```
constructor(
    address _ward,
    address _pool,
    bytes32 _wat,
    address[] memory _orbs,
    bool _nonreentrant
)
```
- **`_ward`**: The address to be granted initial admin (ward) privileges for the contract.
- **`_pool`**: The Curve pool address associated with this oracle.
- **`_wat`**: A label representing the asset the oracle is tracking.
- **`_orbs`**: Array of oracle addresses, each representing a token in the Curve pool.
- **`_nonreentrant`**: Boolean to control reentrancy checks with the Curve pool.

#### Oracle Management and Price Updates

##### `step` Function
```
function step(uint16 _hop) external auth
```
The `step` function changes the minimum interval between oracle updates.

- **`_hop`**: The new minimum time interval (in seconds) between updates. This is stored in `hop` and is used to throttle updates by requiring this period to elapse before the next update.

##### `link` Function
```
function link(uint256 _id, address _orb) external auth
```
The `link` function allows updating the oracle address for a specific token in the Curve pool.

- **`_id`**: The index of the token in the pool (zero-based). Each token in the pool is represented by an index, and this allows changing the oracle associated with that specific token.
- **`_orb`**: The new oracle address for the token at index `_id`. The address must be non-zero.

##### `poke` Function
```
function poke() external payable
```
The `poke` function fetches the latest prices from the token oracles and calculates the current LP token price based on the lowest token price and the virtual price of the Curve pool.

- **No parameters**: Although it’s marked `payable`, it doesn’t accept ETH. This marking is solely to save gas.

#### Price Retrieval Functions

##### `peek` Function
```
function peek() external view toll returns (bytes32, bool)
```
The `peek` function retrieves the current price of the LP token without requiring the price to be valid.

- **No parameters**: Calls to `peek` retrieve the current LP token price and whether it’s valid (e.g., has been updated since initialization). Only addresses whitelisted in `bud` (using the `toll` modifier) can call this function.

##### `peep` Function
```
function peep() external view toll returns (bytes32, bool)
```
The `peep` function retrieves the next LP token price in the queue, indicating what the price will be after the next update.

- **No parameters**: Calls to `peep` return the next queued price for the LP token and whether it’s valid. Like `peek`, this function is restricted to whitelisted addresses.

##### `read` Function
```
function read() external view toll returns (bytes32)
```
The `read` function retrieves the current LP token price, ensuring it’s valid. If the price isn’t valid, it reverts.

- **No parameters**: This function returns the current price but only if it’s marked valid. It’s useful for cases where the caller requires a valid price or the transaction should fail.

#### Whitelisting Controls

##### `kiss` and `diss` Functions
```
function kiss(address _a) external auth
function kiss(address[] calldata _a) external auth
function diss(address _a) external auth
function diss(address[] calldata _a) external auth
```
These functions manage whitelisted addresses that can access certain restricted functions (e.g., `peek`, `peep`, `read`).

- **`_a`**: An address or array of addresses to be added or removed from the whitelist. This is done by setting the corresponding value in `bud` to 1 (whitelisted) in `kiss` and to 0 (removed) in `diss`.

---

### Fallback Functions

##### `receive` Function
```
receive() external payable
```
This fallback function allows the contract to receive ETH, though no ETH should be sent to it in normal usage. Marking it `payable` saves gas by eliminating additional checks for ETH transfers when calling `poke`.

---

This detailed breakdown explains each parameter within the functions, making it easier to understand their roles within the `CurveLPOracleFactory` and `CurveLPOracle` contracts.