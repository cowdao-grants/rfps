# RFP-04: Migrating E2E Tests to Foundry

## Preamble

Requests for proposals are not to be taken as prescriptive or exhaustive. The community is encouraged to submit proposals that build upon the ideas presented in this document. The scope of the project may change based on the proposal received. The primary intent of this document is to provide a starting point from which to achieve the goals outlined, and what is ultimately implemented may differ from the initial proposal.

## Background

CoW Protocol is committed to maintaining a robust and reliable testing framework for its smart contracts. Currently, the end-to-end (E2E) tests for the CoW Protocol contracts repository are implemented using Hardhat. However, to align with the evolving needs of the developer community and to leverage the capabilities of the Foundry testing framework, there is a need to migrate the existing E2E tests from Hardhat to Foundry. This migration is intended to improve test performance, developer experience, and the overall reliability of the testing process.

## Goal

The goals of this project are the following:

1. Migrate all existing E2E tests (as described in issue [106](https://github.com/cowprotocol/contracts/issues/106)) from Hardhat to Foundry.
2. Ensure that all E2E tests that are suitable as fork tests are implemented accordingly.
3. Integrate the existing unit test libraries with the new Foundry E2E tests to ensure consistency and reusability across the testing suite.
4. Modify the CI/CD pipeline to ensure that the E2E tests are deterministic, with tests pinned to a specific block number.

### Key Features

- All E2E tests should be migrated to Foundry while maintaining or improving the current test coverage and performance.
- Tests that can be categorized as fork tests should be implemented as such in Foundry, leveraging Foundry's forking capabilities.
- The existing unit test libraries should be reused in the E2E tests to maintain consistency and avoid code duplication.
- Each E2E test migration must be submitted in a separate Pull Request (PR). The PR should follow best practices for documentation, including a clear description of the changes, references to related issues, and any relevant context or considerations. This documentation must be consistent with the standards already established in the CoW Protocol contracts repository.
- The CI/CD pipeline must be adjusted to ensure that the E2E tests are deterministic. This includes pinning tests to a specific block number to ensure consistent and reliable test results across different environments.

### Deliverables

- Fully migrated E2E test suite using Foundry, with each test in its own well-documented PR against the `contracts` repository.
- Updated CI/CD pipeline to include deterministic E2E tests.

### Further References

- The [CoW AMM repository](https://github.com/cowprotocol/cow-amm/tree/test-interface/test/fork) is an example of a CoW Protocol repository where forked end-to-end tests are implemented.
