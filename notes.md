# Eggstravaganza

## About

EggHuntGame is a gamified NFT experience where participants search for hidden eggs to mint unique Eggstravaganza Egg NFTs.

Players engage in an interactive hunt during a designated game period, and successful egg finds can be deposited into a secure Egg Vault.

## Actors

1. **Game Owner**: The deployer/administrator who starts and ends the game, adjusts game parameters, and manages ownership.
2. **Player**: Participants who call the egg search function, mint Egg NFTs upon successful searches, and may deposit them into the vault.
3. **Vault Owner**: The owner of the EggVault contract responsible for managing deposited eggs.

## Deployment

1. **Deploy EggstravaganzaEggNFT Contract:**\
   This contract handles the minting of unique Egg NFTs.
2. **Deploy EggVault Contract:**\
   Acts as the secure vault for storing Egg NFTs.
3. **Deploy EggHuntGame Contract:**\
   Initialize by passing in the deployed addresses of the Egg NFT and Egg Vault contracts.
4. **Set Ownership:**\
   Ensure the deployer (or designated admin) holds ownership to manage game functions.

# Scope

src/

1. EggHuntGame.sol // Main game contract managing the egg hunt lifecycle and minting process.
2. EggVault.sol // Vault contract for securely storing deposited Egg NFTs.
3. EggstravaganzaNFT.sol // ERC721-style NFT contract for minting unique Egg NFTs.

- EggstravaganzaNFT.sol (done)
