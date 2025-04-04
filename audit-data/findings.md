### [S-#] TITLE (Root Cause + Impact)

**Description:**

**Impact:**

**Proof of Concept:**

**Recommended Mitigation:**

# Low

### [L-1] Missing Event Logging, become difficult when debugging and monitoring

**Description:** The contract lacks event emission for critical state changes. Events are essential for tracking contract activity, debugging, and off-chain indexing.

**Impact:**

1. Debugging and monitoring become difficult without transaction logs
2. Reduced transparency and observability in contract execution

More reference: https://solodit.cyfrin.io/issues/hal-03-lack-of-event-emission-halborn-superhedge-superhedge-v1-core-markdown

**Proof of Concept:**

1. The `EggstravaganzaNFT::setGameContract` function updates `EggstravaganzaNFT::gameContract` variable without emitting an event:

```javascript
    /// @notice Only the owner can set the game contract allowed to mint eggs.
    function setGameContract(address _gameContract) external onlyOwner {
        require(_gameContract != address(0), "Invalid game contract address");
@>      gameContract = _gameContract;
    }
```

2. The `EggstravaganzaNFT::mintEgg` function mints an NFT but does not log the event:

```javascript
    /// @notice Public function to mint a new Eggstravaganza NFT.
    /// Only the approved game contract can mint eggs.
    function mintEgg(address to, uint256 tokenId) external returns (bool) {
        require(msg.sender == gameContract, "Unauthorized minter");
        _mint(to, tokenId);
@>      totalSupply += 1;
        return true;
    }
```

3. The `EggVault::setEggNFT` function set the NFT contract address without emitting an event:

```javascript
    /// @notice Set the NFT contract address.
    function setEggNFT(address _eggNFTAddress) external onlyOwner {
        require(_eggNFTAddress != address(0), "Invalid NFT address");
@>      eggNFT = EggstravaganzaNFT(_eggNFTAddress);
    }
```

**Recommended Mitigation:** Introduce event logging for critical state changes to improve contract transparency and traceability. Modify the functions to emit these events and add the following event declarations:

1. EggstravaganzaNFT.sol:

```diff
+   event GameContractUpdated(address indexed newGameContract);
+   event EggMinted(address indexed to, uint256 indexed tokenId);
.
.
.
    /// @notice Only the owner can set the game contract allowed to mint eggs.
    function setGameContract(address _gameContract) external onlyOwner {
        require(_gameContract != address(0), "Invalid game contract address");
        gameContract = _gameContract;
-   }
+       emit GameContractUpdated(_gameContract);
+   }

    /// @notice Public function to mint a new Eggstravaganza NFT.
    /// Only the approved game contract can mint eggs.
    function mintEgg(address to, uint256 tokenId) external returns (bool) {
        require(msg.sender == gameContract, "Unauthorized minter");
        _mint(to, tokenId);
        totalSupply += 1;
-       return true;
-   }
+       emit EggMinted(to, tokenId);
+       return true;
+   }
```

2. EggVault.sol:

```diff
+   event EggNFTSet(address indexed newEggNFT);
    /// @notice Set the NFT contract address.
    function setEggNFT(address _eggNFTAddress) external onlyOwner {
        require(_eggNFTAddress != address(0), "Invalid NFT address");
        eggNFT = EggstravaganzaNFT(_eggNFTAddress);
-   }
+       emit EggNFTSet(_eggNFTAddress);
+   }
```

# Likelihood & Impact:

- Impact: LOW
- Likelihood: HIGH
- Severity: LOW

# Informational

### [I-1] Floating pragmas

**Description:** Contracts should use strict versions of solidity. Locking the version ensures that contracts are not deployed with a different version of solidity than they were tested with. An incorrect version could lead to uninteded results.

https://solodit.cyfrin.io/issues/n-01-use-of-floating-pragma-code4rena-rubicon-rubicon-git

**Proof of Concept:** There are 3 different smart contracts, each using a Floating Pragma

```javascript
    // src/EggstravaganzaNFT.sol
    pragma solidity ^0.8.23;

    // src/EggVault.sol
    pragma solidity ^0.8.23;

    // src/EggHuntGame.sol
    pragma solidity ^0.8.23;
```

**Recommended Mitigation:** Lock up pragma versions.

```diff
- pragma solidity ^0.8.23;
+ pragma solidity 0.8.23;
```
