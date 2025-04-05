# High

### [H-1] There is no ownership check on `EggVault::depositEgg` function, anyone can call the function

**Description:** The `EggVault::depositEgg` function has no checking system for a depositor, allowing anyone to call this function without ensuring that the `depositor` is the rightful owner of the `tokenId`.

**Impact:**

1. Attackers can withdraw other people's NFTs
2. Original depositor cannot withdraw his NFT

**Proof of Concept:**

1. Mint egg to vault
2. Attacker calls `EggVault::depositEgg` and the function will think that this is a valid `depositor`
3. Attacker calls `EggVault::withdrawEgg`, freely withdraw his stolen NFT from the vault.

**Proof of Code:**

Add the following code to the `EggHuntGameTest.t.sol` file.

```javascript
    function test_Attacker_Steals_Egg_FromVault() public {
        vm.prank(address(game));
        nft.mintEgg(address(vault), 999);

        // assuming bob is the attacker
        vm.prank(bob);
        vault.depositEgg(999, bob);

        assertEq(vault.eggDepositors(999), bob);
        assertTrue(vault.isEggDeposited(999));

        vm.prank(bob);
        vault.withdrawEgg(999);

        assertEq(nft.ownerOf(999), bob);
    }
```

**Recommended Mitigation:** To prevent this problem, we should add an NFT ownership check before assigning a `depositor`.

```diff
    function depositEgg(uint256 tokenId, address depositor) public {
        require(eggNFT.ownerOf(tokenId) == address(this), "NFT not transferred to vault");
+       require(eggNFT.ownerOf(tokenId) == msg.sender, "Caller is not the owner");
        require(!storedEggs[tokenId], "Egg already deposited");
        storedEggs[tokenId] = true;
        eggDepositors[tokenId] = depositor;
        emit EggDeposited(depositor, tokenId);
    }
```

### Likelihood & Impact:

- Impact: HIGH
- Likelihood: MEDIUM
- Severity: HIGH

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

1. src/EggstravaganzaNFT.sol:

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
+       emit GameContractUpdated(_gameContract);
    }

    /// @notice Public function to mint a new Eggstravaganza NFT.
    /// Only the approved game contract can mint eggs.
    function mintEgg(address to, uint256 tokenId) external returns (bool) {
        require(msg.sender == gameContract, "Unauthorized minter");
        _mint(to, tokenId);
        totalSupply += 1;
+       emit EggMinted(to, tokenId);
        return true;
    }
```

2. src/EggVault.sol:

```diff
+   event EggNFTSet(address indexed newEggNFT);
    /// @notice Set the NFT contract address.
    function setEggNFT(address _eggNFTAddress) external onlyOwner {
        require(_eggNFTAddress != address(0), "Invalid NFT address");
        eggNFT = EggstravaganzaNFT(_eggNFTAddress);
+       emit EggNFTSet(_eggNFTAddress);
    }
```

### Likelihood & Impact:

- Impact: LOW
- Likelihood: HIGH
- Severity: LOW

# Gas

### [G-1] `public` functions not used internally could be marked `external`

**Description:** Using a `public` function can cost a lot of gas, if not used internally.

**Proof of Concept:**

- src/EggVault.sol:

```javascript
    function depositEgg(uint256 tokenId, address depositor) public {
.
.
.
    function withdrawEgg(uint256 tokenId) public {
.
.
.
    function isEggDeposited(uint256 tokenId) public view returns (bool) {
```

**Recommended Mitigation:** consider marking it as `external` if it is not used internally.

```diff
-   function depositEgg(uint256 tokenId, address depositor) public {
+   function depositEgg(uint256 tokenId, address depositor) external {
.
.
.
-   function withdrawEgg(uint256 tokenId) public {
+   function withdrawEgg(uint256 tokenId) external {
.
.
.
-   function isEggDeposited(uint256 tokenId) public view returns (bool) {
+   function isEggDeposited(uint256 tokenId) external view returns (bool) {
```

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
