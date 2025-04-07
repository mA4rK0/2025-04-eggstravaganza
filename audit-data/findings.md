# High

### [H-1] There is no ownership check on `EggVault::depositEgg` function, anyone can call the function (Submitted)

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

```Solidity
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

### [H-2] Weak Randomness in `random` value generated inside `EggHuntGame::searchForEgg`, allows anyone to find eggs very quickly (submitted)

**Description:** Hashing `msg.sender`, `block.timestamp`, `block.prevrandao` together creates a predictable final number. A predictable number is not a good random number.

**Impact:** Malicious users can manipulate these values or know them in advance to find the eggs.

**Proof of Concept:** Validators can know ahead of time the `block.timestamp` and `block.prevrandao` and use that knowledge to predict when / how to participate. See the [solidity blog on prevrando](https://soliditydeveloper.com/prevrandao) here.

**Recommended Mitigation:** Consider using an oracle for your randomness like [Chainlink VRF](https://docs.chain.link/vrf/v2/introduction).

### Likelihood & Impact:

- Impact: HIGH
- Likelihood: MEDIUM
- Severity: HIGH

# Medium

### [M-1] Unused Return Value from External Call May Cause Logic Inconsistencies (Submitted)

**Description:** In the `EggHuntGame::searchForEgg` function, the return value from the external call to `eggNFT::mintEgg` is ignored. The function is defined to return a bool, indicating whether the minting was successful. However, this value is not used, and the contract proceeds to update state variables such as `eggCounter` and `eggsFound[msg.sender]` regardless of the minting outcome. Ignoring return values from external contract calls can lead to inconsistencies in contract state and potential failure to meet functional expectations.

**Impact:** If `eggNFT::mintEgg` fails and returns `false`, the contract will still increment `eggCounter` and increase the `eggsFound` count, even though the NFT was not successfully minted. This creates a mismatch between the recorded egg count and the actual minted NFTs, potentially leading to incorrect user balances, broken logic in later parts of the system, or even exploitable conditions if those inconsistencies are used as assumptions elsewhere in the contract.

**Proof of Concept:**

<details>
<summary>Proof of Code</summary>
Place the following test into `EggHuntGameTest.t.sol`.

```Solidity
interface IEggNFT {
    function mintEgg(address to, uint256 tokenId) external returns (bool);
}
.
.
.
contract EggGameTest is Test {
    EggstravaganzaNFT nft;
    EggVault vault;
    EggHuntGame game;
    address owner;
    address alice;
    address bob;
    address mockNFT = address(0x3);

    error OwnableUnauthorizedAccount(address account);
.
.
.
    function test_UncheckedReturnValue_ShouldNotUpdateStateIfMintFails() public {
        EggHuntGame exploitableGame = new EggHuntGame(mockNFT, address(vault));

        vm.mockCall(
            mockNFT,
            abi.encodeWithSelector(IEggNFT.mintEgg.selector, alice, 1),
            abi.encode(false)
        );

        exploitableGame.setEggFindThreshold(100);
        vm.prank(owner);
        exploitableGame.startGame(100);

        vm.warp(block.timestamp + 10);
        vm.prank(alice);
        exploitableGame.searchForEgg();

        // We expect eggCounter to increment, even though mint failed
        // state was updated despite mint failure
        assertEq(exploitableGame.eggCounter(), 1);
        assertEq(exploitableGame.eggsFound(alice), 1);
    }
}
```

</details>

**Recommended Mitigation:** Always check the return value of external calls that indicate success or failure. In this case, ensure the `mintEgg` call returns `true`.

```diff
    function searchForEgg() external {
        require(gameActive, "Game not active");
        require(block.timestamp >= startTime, "Game not started yet");
        require(block.timestamp <= endTime, "Game ended");

        // Pseudo-random number generation (for demonstration purposes only)
        uint256 random =
            uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender, eggCounter))) % 100;

        if (random < eggFindThreshold) {
            eggCounter++;
            eggsFound[msg.sender] += 1;
-           eggNFT.mintEgg(msg.sender, eggCounter);
+           bool success = eggNFT.mintEgg(msg.sender, eggCounter);
+           assert(success);
            emit EggFound(msg.sender, eggCounter, eggsFound[msg.sender]);
        }
    }
```

### Likelihood & Impact:

- Impact: MEDIUM/HIGH
- Likelihood: MEDIUM
- Severity: MEDIUM

# Low

### [L-1] Missing Event Logging, become difficult when debugging and monitoring (Submitted)

**Description:** The contract lacks event emission for critical state changes. Events are essential for tracking contract activity, debugging, and off-chain indexing.

**Impact:**

1. Debugging and monitoring become difficult without transaction logs
2. Reduced transparency and observability in contract execution

More reference: https://solodit.cyfrin.io/issues/hal-03-lack-of-event-emission-halborn-superhedge-superhedge-v1-core-markdown

**Proof of Concept:**

1. The `EggstravaganzaNFT::setGameContract` function updates `EggstravaganzaNFT::gameContract` variable without emitting an event:

```Solidity
    /// @notice Only the owner can set the game contract allowed to mint eggs.
    function setGameContract(address _gameContract) external onlyOwner {
        require(_gameContract != address(0), "Invalid game contract address");
@>      gameContract = _gameContract;
    }
```

2. The `EggstravaganzaNFT::mintEgg` function mints an NFT but does not log the event:

```Solidity
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

```Solidity
    /// @notice Set the NFT contract address.
    function setEggNFT(address _eggNFTAddress) external onlyOwner {
        require(_eggNFTAddress != address(0), "Invalid NFT address");
@>      eggNFT = EggstravaganzaNFT(_eggNFTAddress);
    }
```

4. The `EggHuntGame::setEggFindThreshold` function set the new find threshold without emitting an event:

```Solidity
    function setEggFindThreshold(uint256 newThreshold) external onlyOwner {
        require(newThreshold <= 100, "Threshold must be <= 100");
@>      eggFindThreshold = newThreshold;
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

3. src/EggHuntGame.sol:

```diff
+   event EggFindThresholdUpdated(uint256 newThreshold);
.
.
.
    function setEggFindThreshold(uint256 newThreshold) external onlyOwner {
        require(newThreshold <= 100, "Threshold must be <= 100");
        eggFindThreshold = newThreshold;
+       emit EggFindThresholdUpdated(newThreshold);
    }
```

### Likelihood & Impact:

- Impact: LOW
- Likelihood: HIGH
- Severity: LOW

### [L-2] Mutable Reference to External Contract (Missed `immutable` Keyword), more gas cost per access (Submitted)

**Description:** The `EggHuntGame::eggNFT` and `EggHuntGame::eggVault` variables are assigned once in the constructor and never updated afterward. These references are ideal candidates for the `immutable` keyword.

**Impact:** Slightly higher gas cost per access and reduced clarity for readers and auditors.

**Proof of Concept:**

Proof of Code:

```Solidity
    EggstravaganzaNFT public eggNFT;
    EggVault public eggVault;
```

**Recommended Mitigation:** Mark `EggHuntGame::eggNFT` and `EggHuntGame::eggVault` variables as `immutable`.

```diff
-   EggstravaganzaNFT public eggNFT;
-   EggVault public eggVault;
+   EggstravaganzaNFT public immutable eggNFT;
+   EggVault public immutable eggVault;
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

```Solidity
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

```Solidity
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
