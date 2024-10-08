# RFP-01: Settlement contract wrapper

## Preamble

Requests for proposals are not to be taken as prescriptive or exhaustive. The community is encouraged to submit proposals that build upon the ideas presented in this document. The scope of the project may change based on the proposal received. The primary intent of this document is to provide a starting point from which to achieve the goals outlined, and what is ultimately implemented may differ from the initial proposal.

## Background

The CoW DAO has recently passed [CIP-44: Reduced bonding requirements](https://snapshot.org/#/cow.eth/proposal/0x1b6f1171633ec3d20c4370db37074aa1bd830486d4d0d6c26165915cc42d9412), which allows solvers to participate in the batch auction with reduced bonding requirements. The reduced bonding requirements are as follows:

- $50k in yield-bearing stable coins or ETH
- 500k COW

The target deposit is:

- $100k in yield-bearing stable coins or ETH
- 1M COW

## Goal

The goals of this project are the following.

1. Implement a `SubPoolFactory` that allows solvers to participate in the batch auction with reduced bonding requirements. The `SubPoolFactory` will deploy a `SubPool` for each solver, which can be funded with arbitrary tokens and amounts.
2. Implement `SignedSettlement`, a wrapper around the `GPv2Settlement` contract that allows solvers to execute their solutions by providing a signature attesting to the solution and their parameters.

### Key Features

- The `SignedSettlement` wrapper gas overhead must be minimal. 
- Solver funds must be only stored in `SubPool`s and not mingled together.
- It must be easy to map solver addresses from and to `SubPool` addresses.  

### Deliverables

- Contract system for `SubPoolFactory`, `SubPool`, and `SignedSettlement`
- Comprehensive documentation
- Open-source code repository

### Interface

The interfaces below are a _starting point_ for the project and are not exhaustive / binding - they have not be audited or exhaustively tested / reasoned. The community is encouraged to submit proposals that build upon the ideas presented in this document.

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
     * Determine if a candidate:
     * 1. Has a registered sub-pool; and
     * 2. The sub-pool has not announced an exit; and
     * 3. The sub-pool has no overdue bills
     * While not enforced on-chain, an address for which this call returns `true` is a
     * solver and is allowed to submit settlement on CoW Protocol through
     * `SignedSettlement`.
     * @param candidate The address to check
     * @return bool if the address should be allowed to submit settlements through
     * `SignedSettlement`
     */
    function canSolve(address candidate) external view returns (bool);
    /**
     * View function for determining the sub-pool for a given solver.
     * Reverts if the input address has no sub-pool to its name.
     * @param solver The solver to retrieve the sub-pool for
     * @return The contract managing the collateral paid by this solver to be
     * allowed to settle transactions on CoW Protocol
     */
    function solverSubPool(address solver) external view returns (SubPool);
    /**
     * View function for determining the backend URI for the sub-pool
     * @return string URI
     */
    function backendUri(SubPool pool) external view returns (string memory);

    // --- permissionless functions ---
    /**
     * Deploy a `SubPool` for a solver and fund it with the minimum required collateral.
     * @notice Deposited collateral consists of `amt` of `token` and `cowAmt` of COW.
     * @param token The nominated token to use for collateral
     * @param amt How much collateral to add to the sub-pool (minimum amount should be
     * confirmed off-chain before calling this function)
     * @param cowAmt Amount of COW to add as collateral (the call will revert if this
     * value is below the minimal collateral needed for COW - acts as spam prevention
     * and gate-keeps `canSolve`).
     * @param solver The address of the solver to create a sub-pool for (the call will
     * revert if the solver already has a sub-pool or is a contract).
     * @param backendUri The URI to send the batches to the solver
     * @dev Emits an event `SubPoolCreated` when creating a sub-pool.
     */
    function create(
        IERC20 token,
        uint256 amt,
        uint256 cowAmt,
        address solver,
        string calldata backendUri
    ) external;

    // --- cow dao bonding pool authed functions ---

    /**
     * Set the minimum delay before a solver can claim back the collateral from its
     * sub-pool
     * @dev Emits an event when setting a delay
     * @param delay The delay in seconds
     */
    function setExitDelay(uint256 delay) external auth;
    /**
     * Set the minimum collateral required for $COW
     * @dev Emits an event when setting the minimum collateral
     * @param cowAmt The minimum amount of COW required
     */
    function setMinCow(uint256 cowAmt) external auth;
    /**
     * Bill a sub-pool and send the proceeds to the caller
     * The `auth` is unable to bill the sub-pool if the sub-pool has announced an exit
     * and the `delay` has elapsed.
     * @param pool Which sub-pool to bill
     * @param tokens The tokens to bill
     *        (token 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE is taken to be the native token)
     * @param amts The amount of tokens to bill
     * @param reason The reason for the bill (fine, etc)
     * @dev Emits an event `Bill` when billing a sub-pool
     */
    function bill(SubPool pool, IERC20[] calldata tokens, uint256[] calldata amts, string calldata reason) external auth;

    // --- SubPool authed functions ---

    /**
     * Indicate a sub-pool's intent to exit the batch auction
     * At this point the solver whose sub-pool it is is now removed from
     * the competition.
     * This then starts the `delay` timer until exit is allowed.
     * @param pool The sub-pool to announce an exit for
     * @dev Emits an event `AnnounceExit`
     */
    function announceExit(SubPool pool) external onlySubpoolAuth;
    /**
     * Withdraw any remaining collateral / cow from the sub-pool.
     * Funds are sent to `msg.sender`.
     * @dev Will revert if the sub-pool has not announced the exit and/or the
     * `delay` has not passed.
     * @dev MUST emit an event `Exit` on the factory
     * @dev This function can be called repeatedly so long as the sub-pool has
     * announced an exit and the `delay` has passed. This allows for the
     * withdrawal of any remaining collateral / cow (or that which was accidentally
     * sent to the sub-pool).
     * @param pool The sub-pool whose exit to finalize
     * @param tokens The tokens to withdraw
     */
    function exit(SubPool pool, IERC20[] calldata tokens) external onlySubpoolAuth;
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
     * - `deadline` - the deadline by which the settlement must be executed (e.g. block number)
     * - `tokens` - the traded tokens vector for the settlement
     * - `clearingPrices` - the clearing prices vector for the settlement
     * - `trades` - the trades vector for the settlement
     * - `interactions` - the interactions vector for the settlement
     *
     * @notice This contract is a facade around the `GPv2Settlement` contract and has no
     *         on-chain state / binding to the concept of a sub-pool. It is merely a
     *         permissioned wrapper around the `settle` function.
     * @dev There must be **NO** use of `SLOAD` / `SSTORE` in the execution path.
     * @param signedData The signed data to execute the settlement
     * @param r The `r` component of the signature
     * @param s The `s` component of the signature
     * @param v The `v` component of the signature
     */
    function signedSettle(
        bytes calldata signedData,
        bytes32 r,
        bytes32 s,
        uint8 v,
    ) external onlySubPool;
}
```
