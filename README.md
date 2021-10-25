# TIBC
Terse Interchain Standards (TICS) for the Cosmos network & interchain ecosystem.

## Synopsis

This repository is the canonical location for development and documentation of the terse inter-blockchain communication protocol (TIBC).

TIBC protocol, based on IBC protocol, simplifies and updates some IBC designs and implementations to reduce complexity of connections with heterogeneous blockchains and to enhance the abilities of cross-chain NFT, smart-contract, and services interactions.

This repository shall be used to consolidate design rationale, protocol semantics, and encoding descriptions for TIBC, including both the core transport, authentication, & ordering layer (TIBC/TAO) and the application layers describing packet encoding & processing semantics (TIBC/APP).

Contributions are welcome.

## Standardisation
Please see [ICS 1](https://github.com/cosmos/ibc/blob/master/spec/ics-001-ics-standard/README.md) for a description of what a standard entails.

To propose a new standard, open an issue.

To start a new standardisation document, copy the template and open a PR.

See PROCESS.md for a description of the standardisation process.

See STANDARDS_COMMITTEE.md for the membership of the core standardisation committee.

See CONTRIBUTING.md for contribution guidelines.

## Interchain Standards

All standards at or past the "Draft" stage are listed here in order of their ICS/TICS numbers, sorted by category.

### Meta

| Interchain Standard Number               | Standard Title             | Stage |
| ---------------------------------------- | -------------------------- | ----- |
| [1](https://github.com/cosmos/ibc/blob/master/spec/ics-001-ics-standard/README.md) | ICS Specification Standard | N/A   |

### Core

| Terse Interchain Standard Number                                    | Standard Title             | Stage |
| ------------------------------------------------------------- | -------------------------- | ----- |
| [2](core/tics-002-client-semantics/README.md)             | Client Semantics           | Candidate |
| [4](core/tics-004-port-and-packet-semantics/README.md) | Port & Packet Semantics | Candidate |
| [24](core/tics-024-host-Environments/README.md)           | Host Requirements          | Candidate |
| [26](core/tics-026-packet-routing/README.md)              | Packet Routing             | Candidate |

### Client

| Terse Interchain Standard Number                                      | Standard Title             | Stage |
| --------------------------------------------------------------- | -------------------------- | ----- |
| [7](client/tics-007-tendermint-client/README.md)            | Tendermint Client          | Candidate |
| [8](client/tics-008-bsc-client/README.md)            | BSC Client          | Candidate |


### Relayer

| Terse Interchain Standard Number                                       | Standard Title             | Stage |
| ---------------------------------------------------------------- | -------------------------- | ----- |
| [18](relayer/tics-018-relayer-algorithms/README.md)          | Relayer Algorithms         | Candidate |

### App

| Terse Interchain Standard Number                               | Standard Title          | Stage |
| -------------------------------------------------------- | ----------------------- | ----- |
| [30](apps/nft/tics-030-non-fungible-token-transfer.md) | Non-fungible Token Transfer | Candidate |
