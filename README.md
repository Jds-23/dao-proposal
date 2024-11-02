# Governance Protocol Documentation
## ⚠️ Disclaimer: This is underdevelopement. Changes are still frequent.

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Core Components](#core-components)
4. [Contract Interactions](#contract-interactions)
5. [Security Model](#security-model)
6. [Integration Guide](#integration-guide)
7. [Advanced Features](#advanced-features)
8. [Best Practices](#best-practices)

## Overview

The Maya Governance Protocol is a modular and extensible system for creating and managing Decentralized Autonomous Organizations (DAOs). It provides a comprehensive framework for on-chain governance, treasury management, and proposal execution.

### Key Features
- Modular architecture with factory patterns
- Flexible proposal system
- Secure treasury management
- Role-based access control
- Extensible design for custom implementations

## Architecture

### Component Hierarchy

1. **Factory Layer**
   - DAOFactory: Creates and manages DAO instances
   - ProposalFactory: Handles proposal type registration
   - VaultFactory: Manages vault creation and assignment

2. **Core Layer**
   - DAO: Main governance contract
   - Vault: Treasury management
   - BaseProposal: Abstract proposal implementation

3. **Extension Layer**
   - FundableProposal: Proposals requiring treasury funds
   - NonFundableProposal: Standard governance proposals
   - Custom proposal types

## Core Components

### DAOFactory

The DAOFactory contract serves as the entry point for creating new DAOs and managing their whitelist status.

#### Key Functions

```solidity
function createDAO(
    string memory name,
    address[] memory initialGovernors,
    uint256 votingPeriod,
    uint256 votingDelay,
    uint256 proposalThreshold
) external returns (address)
```

Parameters:
- `name`: Identifier for the DAO
- `initialGovernors`: Initial set of governance participants
- `votingPeriod`: Duration of voting in blocks
- `votingDelay`: Delay before voting starts
- `proposalThreshold`: Minimum threshold for proposal creation

#### Access Control
- `WHITELISTER_ROLE`: Required for whitelisting DAOs
- `DEFAULT_ADMIN_ROLE`: Protocol administrator role

### DAO Contract

The core governance contract implementing voting logic and proposal management.

#### State Variables
```solidity
string public name;
uint256 public votingPeriod;
uint256 public votingDelay;
uint256 public proposalThreshold;
```

#### Key Functions

1. **Proposal Management**
```solidity
function registerProposal(address proposalAddress) external
function vote(uint256 proposalId, bool support) external
```

2. **Governance Operations**
```solidity
function execute(uint256 proposalId) external
function cancel(uint256 proposalId) external
```

#### Events
```solidity
event ProposalCreated(uint256 indexed proposalId, address proposer);
event VoteCast(address indexed voter, uint256 proposalId, bool support);
```

### Proposal System

#### BaseProposal
Abstract contract defining core proposal functionality.

```solidity
abstract contract BaseProposal {
    enum ProposalState { 
        Pending, 
        Active, 
        Canceled, 
        Defeated, 
        Succeeded, 
        Executed 
    }
    
    function execute() external virtual;
    function cancel() external virtual;
    function state() external view virtual returns (ProposalState);
}
```

#### FundableProposal
Extended proposal type for treasury interactions.

```solidity
contract FundableProposal is BaseProposal {
    IVault public vault;
    IERC20 public token;
    uint256 public requestedAmount;
    bool public fundsReleased;
}
```

## Contract Interactions

### Proposal Lifecycle

1. **Creation**
   ```mermaid
   graph TD
       A[Create Proposal] --> B[Voting Delay]
       B --> C[Voting Period]
       C --> D{Succeeded?}
       D -->|Yes| E[Queue]
       D -->|No| F[Defeated]
       E --> G[Execute]
   ```

2. **Voting Process**
   - Voting delay period starts
   - Active voting period
   - Vote counting and result determination
   - Execution delay (optional timelock)
   - Execution or cancellation

### Treasury Management

#### Vault Integration
```solidity
interface IVault {
    function deposit(address token, uint256 amount) external;
    function withdraw(address token, uint256 amount) external;
    function executeTransaction(
        address target,
        uint256 value,
        bytes memory data
    ) external returns (bool);
}
```

## Security Model

### Access Control Hierarchy

1. **Protocol Level**
   - DEFAULT_ADMIN_ROLE: Protocol governance
   - WHITELISTER_ROLE: DAO whitelisting

2. **DAO Level**
   - GOVERNOR_ROLE: DAO governance
   - PROPOSAL_ROLE: Proposal creation

### Security Considerations

1. **Reentrancy Protection**
   ```solidity
   modifier nonReentrant() {
       require(!_locked, "ReentrancyGuard: reentrant call");
       _locked = true;
       _;
       _locked = false;
   }
   ```

2. **Timelock Security**
   - Mandatory delay for sensitive operations
   - Cancellation capabilities
   - Emergency controls

3. **Treasury Safety**
   - Multi-signature requirements
   - Transaction limits
   - Whitelisted tokens

## Integration Guide

### Deploying a New DAO

1. **Deploy Factory Contracts**
```javascript
const daoFactory = await DAOFactory.deploy();
const vaultFactory = await VaultFactory.deploy();
```

2. **Create DAO Instance**
```javascript
const dao = await daoFactory.createDAO(
    "MyDAO",
    initialGovernors,
    votingPeriod,
    votingDelay,
    proposalThreshold
);
```

3. **Configure Vault**
```javascript
const vault = await vaultFactory.createVault(dao.address);
await dao.setVault(vault.address);
```

### Creating Custom Proposals

1. **Extend BaseProposal**
```solidity
contract CustomProposal is BaseProposal {
    function execute() external override {
        // Custom execution logic
    }
}
```

2. **Register Proposal Type**
```javascript
await dao.registerProposalType(customProposal.address);
```

## Advanced Features

### Vote Delegation
```solidity
function delegate(address delegatee) external {
    require(delegatee != address(0), "Invalid delegatee");
    _delegate(msg.sender, delegatee);
}
```

### Quadratic Voting
Implementation example for quadratic voting weight calculation:
```solidity
function _getVotingPower(address voter) internal view returns (uint256) {
    uint256 baseVotes = token.balanceOf(voter);
    return sqrt(baseVotes);
}
```

## Best Practices

### Proposal Creation
1. Clear description and documentation
2. Adequate voting period
3. Appropriate quorum requirements
4. Emergency cancellation mechanisms

### Treasury Management
1. Regular audits
2. Transaction limits
3. Multi-signature requirements
4. Whitelisted tokens only

### Governance Parameters
1. **Recommended Settings**
   - Voting Period: 3-7 days
   - Voting Delay: 1-2 days
   - Proposal Threshold: 1% of total supply
   - Quorum: 4% of total supply

2. **Parameter Adjustment**
   - Gradual changes
   - Community consensus
   - Emergency procedures

## Testing and Deployment

### Test Coverage
Ensure comprehensive testing of:
1. Proposal creation and execution
2. Voting mechanics
3. Treasury operations
4. Access control
5. Edge cases

### Deployment Checklist
- [ ] Deploy factories
- [ ] Initialize DAO
- [ ] Configure vault
- [ ] Set up roles
- [ ] Verify contracts
- [ ] Test basic operations

## Upgrade Paths

### Proxy Pattern Integration
```solidity
contract DAOProxy is TransparentUpgradeableProxy {
    constructor(
        address _logic,
        address admin_,
        bytes memory _data
    ) TransparentUpgradeableProxy(_logic, admin_, _data) {}
}
```

### Version Control
- Semantic versioning
- Upgrade coordination
- Backward compatibility
