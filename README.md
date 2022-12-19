# Testing & deployment

## First steps

Before running anything else, make sure to:

1. Install the dependencies by running:

```shell
npm install
```

2. Compile the contracts by running (this command creates the necessary typechain files for frontend also):

```shell
npx hardhat compile
```

## Test

Smart contracts tests can be executed by running:

```shell
npx hardhat test test/LandToken.ts
npx hardhat test test/LandMinter.ts
npx hardhat test test/LandStaker.ts
```

## Deploy

1. Create a file called `.env` inside the `smart-contracts` repository. Copy everything from the `.env.example` to `.env`. Assign the correct value to each variable.

2. Choose the network where you want to deploy the contracts: `hardhat`, `localhost`, `mumbai` or `polygon`. Replace `<network name>` with one of those and run:

```shell
npx hardhat run scripts/deploy.ts --network <network name>
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------

## LandMinter Contract Walkthrough (how it works)

This is an upgradeable contract and it can be upgraded after deployment.

### Phases

#### Whitelisted phase

By activating this phase, only whitelisted addresses will be able to mint.
To activate the whitelisting phase, the contract owner needs to call `toggleOnlyWhitelisted` function.
-> this function toggles the `onlyWhitelisted` boolean, which acts as a trigger for phases `(true = whitelisted phase, false = public phase)`
Whitelisted addresses are those whose boolean is set to true inside the `whitelisted` mapping
-> to set their status to `true` call function `setWhitelistStatus` passing in an array of addresses and an array booleans.
Each index inside the array of addresses correlates to the same index inside of the boolean array so the same function can also be used to remove addresses from the whitelist.
For more info, there is more documentation regarding this, and other, functions down below (search for the name of the function).

#### Public phase

This is the default phase (`onlyWhitelisted` boolean is false by default).
If the whitelisted phase is active, the contract owner only needs to call `toggleOnlyWhitelisted` once again to deactivate the whitelisted phase and enter into to the public phase.

### Minting

#### FIRST STEP

Before the minting starts, the contract owner needs to pass pre-defined token ids using `addPremiumsIds` and `addNonPremiumIds` functions.
These functions need to be called for each size separately, meaning that the owner can pass all the ids for one size at a time.
The added token ids are stored in `_mintablePremiumIds` mapping and `availableNonPremiumIds` mapping, as well as `idToSize` mapping for premium tokens.
Once the token ids have been added using the functions above, the minting can start.

#### mintPremium

The `mintPremium` function takes in an id as an argument and mints it to the user for the appropriate price.
This function checks if the wanted token id is available by querying `_mintablePremiumIds`. If it is, it mints the token id, if it's not it reverts.

#### mintNonPremium

The `mintNonPremium` function takes size as an argument and mints a token with a semi-random token id to the user.
This function creates a random index and takes the token id located at that index from the appropriate array of token ids. It then removes that token id from the array.

#### acquireWithToken

The `acquireWithToken` function is used when users want to mint for a discount using partnered tokens and mints a premium Land to the user.
By default, this function does not work and will work only if any ERC721 token addresses are added using `addTokenDiscount` function.

### Additional contract functionality

#### airdrop

The `airdrop` function can be used by the contract owner to mint premium token ids for only gas fee.
It works the same as mintPremium function, except it's free, and can only be called by the contract owner.

#### withdraw

The `withdraw` function can be used by the contract owner to withdraw all the funds inside the LandMinter smart contract and transfer them to shareholders.
Currently, this function transfers 50% of the funds to the `space` address and the other 50% to the `feeRecipient`.

#### renameLand

The `renameLand` function can be used to rename a token. Renaming a token is free the first time, and requires the user to pay the renameFee on any additional calls.
The function itself only emits an event `TokenRenamed` which will be used by the backend to actually rename the token.

#### \_transferFee

The `_transferFee` function automatically transfers fee to the feeRecipient address, when any of the payable functions are called.

#### calculatePrice

The `calculatePrice` function calculates the price based on the given size, and base parcel price (base price can be premium or regular).
It multiplies the base price with the number of parcels that one Land NFT consists of (S = 1, M = 4, ..., D = 600) and also increases the price based on the size premium.
The size premiums are premiums that are charged for larger parcel sizes (M size has a 10% premium, L size has a 15%, and the premium goes up to 30% for a district).
Both size premiums and land sizes are passed when initializing the contract, and can be additionally changed using appropriate setter functions.

#### checkRegularLandAvailability

The `checkRegularLandAvailability` function returns all available non-premium token ids of a certain size, which is passed as an argument.
It can be used by the front end to disable the minting button if there aren't any tokens available.

--------------------------------------------------------------------------------------------------------------------------------------------------------------

## Land (NFT) Contract Walkthrough

This is the actual Land NFT token contract.
It has access controls based on a given role.

### mint

The `mint` functions requires the caller of the function to have a MINTER_ROLE.
If the caller has that role, this function mints a `_tokenId` to the given `_to` address.

### toggleRevealed

The `toggleRevealed` function toggles the `revealed` boolean. Can only be called by the LAND_MANAGER role.
The revealed boolean controls which tokenURI is shown when calling the `tokenURI` function (it can either be unrevealedURI or the normal art uri which consists of \_baseURI/{tokenId})

### tokensOfOwner(address _owner)
Returns an array of tokenIDs owned by the _owner wallet.

Necessary for the frontend to display all the user-owned Land tokens.

--------------------------------------------------------------------------------------------------------------------------------------------------------------

## LandStaker Contract Walkthrough
LandStaker is an upgradeable contract with staking and unstaking functionalities.

### stakeLand(uint256 _tokenId)
Stakes a Land by requiring the user to transfer the Land with the passed tokenId to this contract.
If the transfer is successful (the user needs to approve the transaction), the contract updates it's several mappings:
 - `staked`               which shows if a certain tokenID is staked
 - `staker`               which shows the wallet address of the staked tokenID
 - `stakedAmount`         which shows the amount of Land staked by a certain wallet address
 - `stakerToTokenIds`     which shows an array of tokenIDs staked by a certain wallet address
 - `_stakedTokenToIndex`  this is a private mapping which just stores the index of the staked tokenIDs inside stakerToTokenIds mapping 
                          (used for the _removeFromMapping private function)

Lastly, this function emits an event Staked which is required for the backend to calculate the EXP correctly.
### unstakeLand(uint256 _tokenId)
Unstakes a Land by transfering the passed tokenID back to the staker.
Also updates the several mappings named above.

Emits an event Unstaked which is required for the backend to calculate the EXP correctly.

### getStakedTokens(address _staker)
Returns an array of tokenIDs staked by the _staker.

Necessary, for the frontend to display the staked tokenIDs of the user's wallet.

--------------------------------------------------------------------------------------------------------------------------------------------------------------
# Solidity API

## Land

### constructor

```solidity
constructor(string _name, string _symbol, string baseURI_, string _unrevealedURI) public
```

_Initializes the Land contract_

| Name            | Type   | Description                                                                      |
| --------------- | ------ | -------------------------------------------------------------------------------- |
| \_name          | string | Token name                                                                       |
| \_symbol        | string | Token symbol                                                                     |
| baseURI\_       | string | Base URI                                                                         |
| \_unrevealedURI | string | If the URI has not yet been revealed, the tokenURI function will return this URI |

### mint

```solidity
function mint(address _to, uint256 _tokenId) external
```

Mints a Land NFT to the user.

| Name      | Type    | Description                                               |
| --------- | ------- | --------------------------------------------------------- |
| \_to      | address | Address of the wallet which will receive the minted token |
| \_tokenId | uint256 | Id of the token to be minted                              |

### exists

```solidity
function exists(uint256 _tokenId) external view returns (bool)
```

_Returns if a token with `_tokenId` exists_

| Name      | Type    | Description                                |
| --------- | ------- | ------------------------------------------ |
| \_tokenId | uint256 | Id of the token whose existence is checked |

### setBaseURI

```solidity
function setBaseURI(string baseURI_) external
```

_Sets the baseURI for the Land token_

| Name      | Type   | Description                   |
| --------- | ------ | ----------------------------- |
| baseURI\_ | string | New baseURI of the Land token |

### setUnrevealedURI

```solidity
function setUnrevealedURI(string _unrevealedURI) external
```

_Sets the unrevealedURI for the Land token_

| Name            | Type   | Description                                                                   |
| --------------- | ------ | ----------------------------------------------------------------------------- |
| \_unrevealedURI | string | New unrevealedURI of the Land token (shows when the bool `revealed` is false) |

### setMinterAddress

```solidity
function setMinterAddress(address _minterContract) external
```

_Sets the Land Minter Contract address, and gives it the `MINTER_ROLE`_

| Name             | Type    | Description                    |
| ---------------- | ------- | ------------------------------ |
| \_minterContract | address | Address of the minter contract |

### toggleRevealed

```solidity
function toggleRevealed() external
```

If the `revealed` boolean is false, the tokenURI function returns `unrevealedURI`, otherwise it returns the concatenated `baseURI` and `${tokenID}`

_Toggles the `revealed` boolean which decides the which uri is going to be returned by the tokenURI function_

### tokenURI

```solidity
function tokenURI(uint256 _tokenId) public view virtual returns (string)
```

If the `revealed` boolean is false, this function returns `unrevealedURI`, otherwise it returns the concatenated `baseURI` and `${tokenID}`

### supportsInterface

```solidity
function supportsInterface(bytes4 interfaceId) public view virtual returns (bool)
```

### \_baseURI

```solidity
function _baseURI() internal view returns (string)
```

_Base URI for computing {tokenURI}. If set, the resulting URI for each
token will be the concatenation of the `baseURI` and the `tokenId`. Empty
by default, can be overridden in child contracts._

### burn

```solidity
function burn(uint256 _tokenId) external
```

## LandMinter

### initialize

```solidity
function initialize(uint256 _fee, uint256 _renameFee, uint256 _basePremiumPrice, uint256 _baseRegularPrice, address _feeRecipient, address _landToken, uint256[] sizePremiums_, uint256[] landSizes_) external
```

_Initializes the contract, can only be called once_

| Name               | Type      | Description                                                                                |
| ------------------ | --------- | ------------------------------------------------------------------------------------------ |
| \_fee              | uint256   | Minting fee                                                                                |
| \_renameFee        | uint256   | Rename fee                                                                                 |
| \_basePremiumPrice | uint256   | Base price for a premium Land NFT                                                          |
| \_baseRegularPrice | uint256   | Base price for a regular Land NFT                                                          |
| \_feeRecipient     | address   | Wallet address of the feeRecipient                                                         |
| \_landToken        | address   | Smart contract address of the Land NFT                                                     |
| sizePremiums\_     | uint256[] | Premiums that increase the price, dependent on size, have to be in order [S, M, L, Z, D]   |
| landSizes\_        | uint256[] | Number of parcels a Land of given size is composed of, have to be in order [S, M, L, Z, D] |

### mintPremium

```solidity
function mintPremium(uint256 _tokenId) external payable
```

Mints a premium Land NFT to the user.

| Name      | Type    | Description |
| --------- | ------- | ----------- |
| \_tokenId | uint256 | Token id    |

### mintNonPremium

```solidity
function mintNonPremium(enum ILandMinter.Size _size) external payable
```

Mints a non-premium Land NFT to the user.

| Name   | Type                  | Description                           |
| ------ | --------------------- | ------------------------------------- |
| \_size | enum ILandMinter.Size | Land size => enumerated S, M or LH, L |

### renameLand

```solidity
function renameLand(uint256 _tokenId, string _tokenName) external payable
```

_Emits an event which is used to rename the Land token, free only the first time_

| Name        | Type    | Description                                    |
| ----------- | ------- | ---------------------------------------------- |
| \_tokenId   | uint256 | Id of the token whose name needs to be changed |
| \_tokenName | string  | The name to which the token will be renamed    |

### calculatePrice

```solidity
function calculatePrice(enum ILandMinter.Size _size, uint256 _basePrice) public view returns (uint256 price)
```

_Calculates the price based on `_basePrice`, parcel `_size`, price premium and size multiplier_

| Name        | Type                  | Description                                            |
| ----------- | --------------------- | ------------------------------------------------------ |
| \_size      | enum ILandMinter.Size | Size of the Land token to be minted                    |
| \_basePrice | uint256               | Can be either `premiumBasePrice` or `regularBasePrice` |

### checkRegularLandAvailability

```solidity
function checkRegularLandAvailability(enum ILandMinter.Size _size) public view returns (uint256[])
```

_Checks availability of regular / non-premium Land_

| Name   | Type                  | Description                                 |
| ------ | --------------------- | ------------------------------------------- |
| \_size | enum ILandMinter.Size | Size of Land for which to check availabilty |

### airdrop

```solidity
function airdrop(address[] _accounts, uint256[] _ids) external
```

_Airdrop function callable by LAND_MANAGERs_

| Name       | Type      | Description                                                                |
| ---------- | --------- | -------------------------------------------------------------------------- |
| \_accounts | address[] | Array of wallet addresses to which the tokens with \_ids will be minted to |
| \_ids      | uint256[] | Array of tokenIds which will be minted                                     |

# Address Whitelist

### setWhitelistStatus

```solidity
function setWhitelistStatus(address[] _targets, bool[] _statuses) external
```

_Sets the whitelist status of \_address_

| Name       | Type      | Description                                                   |
| ---------- | --------- | ------------------------------------------------------------- |
| \_targets  | address[] | Array of addresses which status will be updated               |
| \_statuses | bool[]    | Array of boolean variables indicating status for each address |

### toggleOnlyWhitelisted

```solidity
function toggleOnlyWhitelisted() external
```

_Toggles `onlyWhitelisted` boolean which controls if only the whitelisted addresses are able to mint_

### addPremiumIds

```solidity
function addPremiumIds(enum ILandMinter.Size _size, uint256[] _ids) external
```

The `_ids` that are added cannot already be mitned / cannot already exist

_Adds more tokenIds to the `_mintablePremiumIds` mapping and maps the id to size in the `idToSize` mapping_

| Name   | Type                  | Description                                                                  |
| ------ | --------------------- | ---------------------------------------------------------------------------- |
| \_size | enum ILandMinter.Size | Parcel size to which the `_ids` belong                                       |
| \_ids  | uint256[]             | TokenIds that will become available for minting once this function is called |

### addNonPremiumIds

```solidity
function addNonPremiumIds(enum ILandMinter.Size _size, uint256[] _ids) external
```

The `_ids` that are added cannot already be mint-ed / cannot already exist

_Adds more tokenIds to the `availableNonPremiumIds` mapping_

| Name   | Type                  | Description                                                                  |
| ------ | --------------------- | ---------------------------------------------------------------------------- |
| \_size | enum ILandMinter.Size | Parcel size to which the `_ids` belong                                       |
| \_ids  | uint256[]             | TokenIds that will become available for minting once this function is called |

### withdraw

```solidity
function withdraw(uint256 _amount) external
```

_Transfers the share amount of tokens to each of the shareholders_

| Name     | Type    | Description                                                                  |
| -------- | ------- | ---------------------------------------------------------------------------- |
| \_amount | uint256 | Amount of tokens to be withdrawn (can't be more than the contract's balance) |

### \_transferFee

```solidity
function _transferFee(uint256 _amount) private
```

Calculates fee based on `_amount` and `fee` and transfers it to
`feeRecipient`.

# Token Whitelist

### acquireWithToken

```solidity
function acquireWithToken(contract IERC721 token, uint256 _ownedTokenId, uint256 _tokenIdToMint) external payable
```

Acquires an NFT of this contract by proving ownership of the token in `tokenId` belonging to
a contract `tokenAddress` that has a configured discount. This way cheaper prices can be achieved for LAND holders
and potentially other partners. Emits {TokenUsedForDiscount} and requires the user to send the correct amount of
eth as well as to own the tokens within `tokenIds` from `tokenAddress`, and for `tokenAddress` to be a configured token for discounts.

| Name            | Type             | Description                                          |
| --------------- | ---------------- | ---------------------------------------------------- |
| token           | contract IERC721 |                                                      |
| \_ownedTokenId  | uint256          | the token id which is to be used to get the discount |
| \_tokenIdToMint | uint256          | the token id which is going to be minted             |

### tokenDiscounts

```solidity
function tokenDiscounts() external view returns (struct TokenDiscountOutput[])
```

Returns a list of all current tokens configured for discounts and their configurations.

| Name | Type                         | Description |
| ---- | ---------------------------- | ----------- |
| [0]  | struct TokenDiscountOutput[] |             |

### addTokenDiscount

```solidity
function addTokenDiscount(contract IERC721 tokenAddress, struct TokenDiscountConfig config) public
```

Adds an NFT contract and thus all of it's tokens to the discount list.
Emits a {TokenDiscountAdded} event and fails if `tokenAddress` is the zero address
or is already configured.

| Name         | Type                       | Description                                                              |
| ------------ | -------------------------- | ------------------------------------------------------------------------ |
| tokenAddress | contract IERC721           | the address of the NFT contract                                          |
| config       | struct TokenDiscountConfig | the initial configuration as [uint256 price, uint256 limit, bool active] |

### setTokenDiscountActive

```solidity
function setTokenDiscountActive(contract IERC721 tokenAddress, bool active) external
```

Sets the active status of the token discount of `tokenAddress`.
Fails if `tokenAddress` is the zero address or is not already configured.

| Name         | Type             | Description                    |
| ------------ | ---------------- | ------------------------------ |
| tokenAddress | contract IERC721 | the configured token address   |
| active       | bool             | the new desired activity state |

### \_getRemoteNameOrEmpty

```solidity
function _getRemoteNameOrEmpty(address remote) internal view returns (string)
```

### \_getRemoteSymbolOrEmpty

```solidity
function _getRemoteSymbolOrEmpty(address remote) internal view returns (string)
```

### tokensUsedForDiscount

```solidity
function tokensUsedForDiscount(contract IERC721 tokenAddress, uint256 tokenId) external view virtual returns (bool used)
```

Returns whether the token `tokenId` of `tokenAddress` has already been used for a discount.
Fails if `tokenAddress` is the zero address or is not already configured.

| Name         | Type             | Description                       |
| ------------ | ---------------- | --------------------------------- |
| tokenAddress | contract IERC721 | the address of the token contract |
| tokenId      | uint256          | the id to check                   |

| Name | Type | Description                        |
| ---- | ---- | ---------------------------------- |
| used | bool | if the token has already been used |

### removeTokenDiscount

```solidity
function removeTokenDiscount(contract IERC721 tokenAddress) external
```

Removes an NFT contract from the discount list.
Emits a {TokenDiscountRemoved} event and fails if `tokenAddress` is the zero address
or is not already configured.

| Name         | Type             | Description                     |
| ------------ | ---------------- | ------------------------------- |
| tokenAddress | contract IERC721 | the address of the NFT contract |

### tokenDiscountInfo

```solidity
function tokenDiscountInfo(contract IERC721 tokenAddress) external view returns (struct TokenDiscountOutput)
```

Returns the current configuration of the token discount of `tokenAddress`

| Name | Type                       | Description |
| ---- | -------------------------- | ----------- |
| [0]  | struct TokenDiscountOutput |             |

### updateTokenDiscount

```solidity
function updateTokenDiscount(contract IERC721 tokenAddress, struct TokenDiscountConfig config) external
```

Updates an NFT contracts configuration of the discount.
Emits a {TokenDiscountUpdated} event and fails if `tokenAddress` is the zero address
or is not already configured.

| Name         | Type                       | Description                                                          |
| ------------ | -------------------------- | -------------------------------------------------------------------- |
| tokenAddress | contract IERC721           | the address of the NFT contract                                      |
| config       | struct TokenDiscountConfig | the new configuration as [uint256 price, uint256 limit, bool active] |

### resetTokenDiscountUsed

```solidity
function resetTokenDiscountUsed(contract IERC721 tokenAddress) external
```

Resets the usage state of all NFTs of the contract at `tokenAddress`. This allows all token ids
to be used again.
Emits a {TokenDiscountReset} event and fails if `tokenAddress` is the zero address
or is not already configured.

| Name         | Type             | Description                     |
| ------------ | ---------------- | ------------------------------- |
| tokenAddress | contract IERC721 | the address of the NFT contract |

## LandAccessControl

### MINTER_ROLE

```solidity
bytes32 MINTER_ROLE
```

### LAND_MANAGER

```solidity
bytes32 LAND_MANAGER
```

### constructor

```solidity
constructor() internal
```

## TokenDiscountConfig

```solidity
struct TokenDiscountConfig {
  uint256 price;
  uint256 supply;
  bool active;
}
```

## TokenDiscountInput

```solidity
struct TokenDiscountInput {
  contract IERC721 tokenAddress;
  struct TokenDiscountConfig config;
}
```

## TokenDiscountOutput

```solidity
struct TokenDiscountOutput {
  contract IERC721 tokenAddress;
  string name;
  string symbol;
  uint256 usedAmount;
  struct TokenDiscountConfig config;
}
```

## ILand

### SetBaseUri

```solidity
event SetBaseUri(string baseURI)
```

### SetUnrevealedUri

```solidity
event SetUnrevealedUri(string unrevealedURI)
```

### ToggleRevealed

```solidity
event ToggleRevealed(bool revealed)
```

### burn

```solidity
function burn(uint256 _id) external
```

### exists

```solidity
function exists(uint256 _tokenId) external view returns (bool)
```

### mint

```solidity
function mint(address _to, uint256 _id) external
```

## ILandAcquirableWithToken

This interface serves for the extended minting functionality of the Land Artist Contracts.
The general functionality is that special prices can be configured for users to mint if they hold other
NFTs. Each NFT can only be used once to receive this discount, unless specifically reset.

### NullAddress

```solidity
error NullAddress()
```

### TokenNotOwned

```solidity
error TokenNotOwned(contract IERC721 token, uint256 tokenIds)
```

### TokenAlreadyUsed

```solidity
error TokenAlreadyUsed(contract IERC721 token, uint256 tokenId)
```

### TokenNotConfigured

```solidity
error TokenNotConfigured(contract IERC721 token)
```

### TokenNotActive

```solidity
error TokenNotActive(contract IERC721 token)
```

### TokenAlreadyConfigured

```solidity
error TokenAlreadyConfigured(contract IERC721 token)
```

### TokenSupplyExceeded

```solidity
error TokenSupplyExceeded(contract IERC721 token, uint256 supplyCap)
```

### TokenDiscountAdded

```solidity
event TokenDiscountAdded(contract IERC721 tokenAddress, struct TokenDiscountConfig config)
```

Triggers when a token discount is added.

| Name         | Type                       | Description                                                                               |
| ------------ | -------------------------- | ----------------------------------------------------------------------------------------- |
| tokenAddress | contract IERC721           | the addres of the added NFT contract for discounts                                        |
| config       | struct TokenDiscountConfig | a tuple [uint256 price, uint256 limit, bool active] that represents the configuration for |
| the discount |

### TokenDiscountUpdated

```solidity
event TokenDiscountUpdated(contract IERC721 tokenAddress, struct TokenDiscountConfig config)
```

Triggers when a token discount is updated.

| Name         | Type                       | Description                                                                                   |
| ------------ | -------------------------- | --------------------------------------------------------------------------------------------- |
| tokenAddress | contract IERC721           | the addres of the added NFT contract for discounts                                            |
| config       | struct TokenDiscountConfig | a tuple [uint256 price, uint256 limit, bool active] that represents the new configuration for |
| the discount |

### TokenDiscountRemoved

```solidity
event TokenDiscountRemoved(contract IERC721 tokenAddress)
```

Triggers when a token discount is removed.

| Name         | Type             | Description                    |
| ------------ | ---------------- | ------------------------------ |
| tokenAddress | contract IERC721 | the addres of the NFT contract |

### TokenDiscountReset

```solidity
event TokenDiscountReset(contract IERC721 tokenAddress)
```

Triggers when a token discount is reset - meaning all token usage data is reset and all tokens
are marked as unused again.

| Name         | Type             | Description                    |
| ------------ | ---------------- | ------------------------------ |
| tokenAddress | contract IERC721 | the addres of the NFT contract |

### TokenUsedForDiscount

```solidity
event TokenUsedForDiscount(address sender, contract IERC721 tokenAddress, uint256 tokenId)
```

Triggers when a token discount is used for a discount and then marked as used

| Name         | Type             | Description                             |
| ------------ | ---------------- | --------------------------------------- |
| sender       | address          | the user who used the token             |
| tokenAddress | contract IERC721 | the addres of the NFT contract          |
| tokenId      | uint256          | the id of the NFT used for the discount |

### addTokenDiscount

```solidity
function addTokenDiscount(contract IERC721 tokenAddress, struct TokenDiscountConfig config) external
```

Adds an NFT contract and thus all of it's tokens to the discount list.
Emits a {TokenDiscountAdded} event and fails if `tokenAddress` is the zero address
or is already configured.

| Name         | Type                       | Description                                                              |
| ------------ | -------------------------- | ------------------------------------------------------------------------ |
| tokenAddress | contract IERC721           | the address of the NFT contract                                          |
| config       | struct TokenDiscountConfig | the initial configuration as [uint256 price, uint256 limit, bool active] |

### removeTokenDiscount

```solidity
function removeTokenDiscount(contract IERC721 tokenAddress) external
```

Removes an NFT contract from the discount list.
Emits a {TokenDiscountRemoved} event and fails if `tokenAddress` is the zero address
or is not already configured.

| Name         | Type             | Description                     |
| ------------ | ---------------- | ------------------------------- |
| tokenAddress | contract IERC721 | the address of the NFT contract |

### updateTokenDiscount

```solidity
function updateTokenDiscount(contract IERC721 tokenAddress, struct TokenDiscountConfig config) external
```

Updates an NFT contracts configuration of the discount.
Emits a {TokenDiscountUpdated} event and fails if `tokenAddress` is the zero address
or is not already configured.

| Name         | Type                       | Description                                                          |
| ------------ | -------------------------- | -------------------------------------------------------------------- |
| tokenAddress | contract IERC721           | the address of the NFT contract                                      |
| config       | struct TokenDiscountConfig | the new configuration as [uint256 price, uint256 limit, bool active] |

### resetTokenDiscountUsed

```solidity
function resetTokenDiscountUsed(contract IERC721 tokenAddress) external
```

Resets the usage state of all NFTs of the contract at `tokenAddress`. This allows all token ids
to be used again.
Emits a {TokenDiscountReset} event and fails if `tokenAddress` is the zero address
or is not already configured.

| Name         | Type             | Description                     |
| ------------ | ---------------- | ------------------------------- |
| tokenAddress | contract IERC721 | the address of the NFT contract |

### tokenDiscountInfo

```solidity
function tokenDiscountInfo(contract IERC721 tokenAddress) external view returns (struct TokenDiscountOutput config)
```

Returns the current configuration of the token discount of `tokenAddress`

| Name   | Type                       | Description                                                      |
| ------ | -------------------------- | ---------------------------------------------------------------- |
| config | struct TokenDiscountOutput | the configuration as [uint256 price, uint256 limit, bool active] |

### tokenDiscounts

```solidity
function tokenDiscounts() external view returns (struct TokenDiscountOutput[] discounts)
```

Returns a list of all current tokens configured for discounts and their configurations.
Returns a list of all current tokens configured for discounts and their configurations.

| Name      | Type                         | Description                                                                              |
| --------- | ---------------------------- | ---------------------------------------------------------------------------------------- |
| discounts | struct TokenDiscountOutput[] | the configuration as [IERC721 tokenAddress, [uint256 price, uint256 limit, bool active]] |
| Name      | Type                         | Description                                                                              |
| --------- | ---------------------------- | ---------------------------------------------------------------------------------------- |
| discounts | struct TokenDiscountOutput[] | the configuration as [IERC721 tokenAddress, [uint256 price, uint256 limit, bool active]] |

### acquireWithToken

```solidity
function acquireWithToken(contract IERC721 tokenAddress, uint256 _ownedTokenId, uint256 _tokenIdToMint) external payable
```

Acquires an NFT of this contract by proving ownership of the token in `tokenId` belonging to
a contract `tokenAddress` that has a configured discount. This way cheaper prices can be achieved for LAND holders
and potentially other partners. Emits {TokenUsedForDiscount} and requires the user to send the correct amount of
eth as well as to own the tokens within `tokenIds` from `tokenAddress`, and for `tokenAddress` to be a configured token for discounts.

| Name            | Type             | Description                                                       |
| --------------- | ---------------- | ----------------------------------------------------------------- |
| tokenAddress    | contract IERC721 | the address of the contract which is the reference for `tokenIds` |
| \_ownedTokenId  | uint256          | the token id which is to be used to get the discount              |
| \_tokenIdToMint | uint256          | the token id which is going to be minted                          |

### setTokenDiscountActive

```solidity
function setTokenDiscountActive(contract IERC721 tokenAddress, bool active) external
```solidity
function setTokenDiscountActive(contract IERC721 tokenAddress, bool active) external
```

Sets the active status of the token discount of `tokenAddress`.
Fails if `tokenAddress` is the zero address or is not already configured.

| Name         | Type             | Description                    |
| ------------ | ---------------- | ------------------------------ |
| tokenAddress | contract IERC721 | the configured token address   |
| active       | bool             | the new desired activity state |

### tokensUsedForDiscount

```solidity
function tokensUsedForDiscount(contract IERC721 tokenAddress, uint256 tokenId) external view returns (bool used)
```

Returns whether the token `tokenId` of `tokenAddress` has already been used for a discount.
Fails if `tokenAddress` is the zero address or is not already configured.

| Name         | Type             | Description                       |
| ------------ | ---------------- | --------------------------------- |
| tokenAddress | contract IERC721 | the address of the token contract |
| tokenId      | uint256          | the id to check                   |

| Name | Type | Description                        |
| ---- | ---- | ---------------------------------- |
| used | bool | if the token has already been used |

## ILandMinter

### Size

```solidity
enum Size {
  S,
  M,
  L,
  Z,
  D
}
```

### SetFee

```solidity
event SetFee(uint256 fee)
```

### SetRenameFee

```solidity
event SetRenameFee(uint256 renameFee)
```

### SetFeeRecipient

```solidity
event SetFeeRecipient(address feeRecipient)
```

### ToggleRevealed

```solidity
event ToggleRevealed(bool revealed)
```

### ToggleMinting

```solidity
event ToggleMinting(bool mintingEnabled)
```

### ToggleOnlyWhitelisted

```solidity
event ToggleOnlyWhitelisted(bool status)
```

### Withdraw

```solidity
event Withdraw(address[2] shareholders, uint256 amount)
```

### LandSizeChanged

```solidity
event LandSizeChanged(enum ILandMinter.Size size, uint256 value)
```

### SizePremiumChanged

```solidity
event SizePremiumChanged(enum ILandMinter.Size size, uint256 value)
```

### RegularBasePriceChanged

```solidity
event RegularBasePriceChanged(uint256 baseRegularPrice)
```

### PremiumBasePriceChanged

```solidity
event PremiumBasePriceChanged(uint256 basePremiumPrice)
```

### TokenRenamed

```solidity
event TokenRenamed(address owner, uint256 tokenId, string tokenName)
```
