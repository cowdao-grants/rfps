# RFP-01: Settlement contract wrapper

## Background

The CoW DAO has recently passed [CIP-44: Reduced bonding requirements](https://snapshot.org/#/cow.eth/proposal/0x1b6f1171633ec3d20c4370db37074aa1bd830486d4d0d6c26165915cc42d9412), which allows solvers to participate in the batch auction with reduced bonding requirements. The reduced bonding requirements are as follows:

- $50k in yield-bearing stable coins or ETH
- 500k COW

The target deposit is:

- $100k in yield-bearing stable coins or ETH
- 1M COW

## Goal

The goal of this project is to implement a `SubPoolFactory` that allows solvers to participate in the batch auction with reduced bonding requirements. The `SubPoolFactory` will deploy a `SubPool` for each solver, which will be funded to at least the minimum required amount. The `SubPoolFactory` will also provide a wrapper around the `GPv2Settlement` contract that allows solvers to execute their solutions by providing a signature attesting to the solution and their parameters.

### Key Features

- Ultra gas efficient signed settlement function
- No co-mingling of funds between solvers
- Easily observable on-chain addresses for MEV Blocker rebates
- Permissionless access to the batch auction with reduced bonding requirements

### Deliverables

- Contract system for `SubPoolFactory`, `SubPool`, and `SignedSettlement`
- Comprehensive documentation
- Open-source code repository

### Interface

```solidity
interface Auth {
    /**
     * Allows `usr` to perform root-level functions
     */
    function addOwner(address usr) external auth;
    /**
     * Denies `usr` from being able to perform root-level functions
     */
    function removeOwner(address usr) external auth;
    /**
     * Determine if `usr` is able to perform root-level functions
     */
    function isOwner(address usr) external view returns (bool);
}

interface SubPool is Auth {
    /**
     * Determine the amount required to `heal` the pool
     * @return amt of immutable collateralToken
     * @return cowAmt of COW
     */
    function dues() external view returns (uint256 amt, uint256 cowAmt);
    /**
     * Pull in from `msg.sender` the dues required to `heal` the pool
     */
    function heal() external;
}

interface SubPoolFactory is Auth {
    // --- view functions ---

    /**
     * View function for determining the minimum collateral required if using `token`
     * If the token is not a valid collateral, this function reverts.
     * @param token The token to retrieve the collateral requirement for
     */
    function collateral(IERC20 token) external view returns (uint256);
    /**
     * Determine if an address has a collateralized sub-pool and has not announced an exit.
     * While not enforced on-chain, an address for which this call returns `true` is a
     * solver and is allowed to submit settlement on CoW Protocol through
     * `SignedSettlement`.
     * @param candidate The address to check
     * @return bool if the address should be allowed to submit settlements through
     * `SignedSettlement`
     */
    function canSolve(address candidate) external view returns (bool);
    /**
     * View function for determining the sub-pool for a given solver
     * @param solver The solver to retrieve the sub-pool for
     * @return SubPool
     */
    function solverSubPool(address solver) external view returns (SubPool);
    /**
     * View function for determining the backend URI for the sub-pool
     * @return string URI
     */
    function backendUri(SubPool pool) external view returns (string memory);

    // --- permissionless functions ---
    /**
     * Deploy a `SubPool` deterministically based on `msg.sender`.
     * @notice The `msg.sender` SHOULD NOT be a contract.
     * @param token The nominated token to use for collateral
     * @param amt How much collateral to add to the sub-pool (the call will revert if
     * this value is below the minimal collateral needed for `token`)
     * @param cowAmt Amount of COW to add as collateral
     * @param backendUri The URI to send the batches to the solver
     * @dev Emits an event `SubPoolCreated` when creating a sub-pool
     */
    function create(IERC20 token, uint256 amt, uint256 cowAmt, string calldata backendUri) external;

    // --- cow dao bonding pool authed functions ---

    /**
     * Set the minimum delay before a solver can claim back the collateral from its
     * sub-pool
     * @dev Emits an event when setting a delay
     * @param delay The delay in seconds
     */
    function setExitDelay(uint256 delay) external auth;
    /**
     * Allow `token` to be used as collateral for the sub-pool.
     * @dev Emits an event `AllowCollateral` when allowing/updating a token
     */
    function allowCollateral(IERC20 token, uint256 min) external auth;
    /**
     * Revoke `token` as a collateral for sub-pools
     * @dev Emits an event `RevokeCollateral` when revoking a token
     */
    function revokeCollateral(IERC20 token) external auth;
    /**
     * Fine a sub-pool and send the proceeds to the caller
     * @param pool Which sub-pool to fine
     * @param tokens The tokens to fine
     * @param amts The amount of tokens to fine
     * @dev Emits an event `Fine` when fining a sub-pool
     */
    function fine(SubPool pool, IERC20[] calldata tokens, uint256[] calldata amts) external auth;

    // --- SubPool authed functions ---

    /**
     * Indicate a sub-pool's intent to quit the batch auction
     * At this point the solver whose sub-pool it is is now removed from
     * the competition.
     * This then starts the `delay` timer until exit is allowed.
     * @param pool The sub-pool to announce an exit for
     * @dev Emits an event `AnnounceExit`
     */
    function announceDestroy(SubPool pool) external onlySubpoolAuth;
    /**
     * Withdraw any remaining collateral / cow from the sub-pool.
     * Funds are sent to `msg.sender`.
     * @dev Should emit an event `SubPoolDestroyed` on the factory
     */
    function finalizeExit(SubPool pool) external onlySubpoolAuth;
    /**
     * Set the backend URI for the sub-pool
     * @param pool The sub-pool to set the URI for
     * @param newUri The new URI to set
     * @dev Emits an event `SetBackendUri` on the factory when setting a URI
     */
    function setBackendUri(SubPool pool, string calldata newUri) external onlySubpoolAuth;
}

interface SignedSettlement {
    /**
     * Signed settlement function that is callable by `msg.sender` if `msg.sender` is in
     * receipt of a valid signature. The signer is a trusted entity and bears the responsibility
     * of ensuring that `msg.sender` is correctly collateralised.
     *  
     * The signed data **MUST** include:
     * - `solver` - the address of the sub-bonded solver, i.e. `msg.sender`
     * - Replay protection - an expiry based on some configurable method (e.g. block number)
     *
     * @notice This contract is a facade around the `GPv2Settlement` contract and has no
     *         on-chain state / binding to the concept of a sub-pool. It is merely a
     *         permissioned wrapper around the `settle` function.
     * @dev There must be **NO** use of `SLOAD` / `SSTORE` in the execution path.
     */
    function signedSettle(
        bytes calldata signedData,
        bytes calldata unsignedData,
    ) external onlySubPool;
}
```
