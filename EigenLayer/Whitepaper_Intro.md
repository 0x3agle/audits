# Whitepaper Notes - Intro to EigenLayer
>EigenLayer is as an **open marketplace** where **Actively Validated Sevices can rent pooled security** provided by "Ethereum validators" or "stakers" using restaking.

### Roles
- **Validators** are participants in the Ethereum PoS consensus who stake their ETH to help secure the network, and they can also use Liquid staking tokens to deposit into EigenLayer and provide additional services to earn rewards on their assets.
- **Stakers**, on the other hand, can deposit any ERC20/ETH into EigenLayer to stake their assets and earn rewards.
- **Operators** are the users who run the software built on top of EigenLayer. Operators register in EigenLayer, allowing stakers to delegate to them, and then opt-in to any mix of services built on top of EigenLayer.
- **Services / Middleware** refers to the software built on top of EigenLayer, and can encompass a wide variety of services.

>**DELEGATION**: The trust model in EigenLayer requires restakers (validators/stakers) to place some trust in their chosen operator. 
>
>If the operator fails to fulfill its obligations, both the operator's deposited stake and the stake delegated to the operator will be subject to slashing. This means that restakers must do their due diligence on the operator they choose to delegate their stake to.


### Restaking

> Staking is the process of holding and locking up a certain amount of cryptocurrency or other digital assets in order to support the operations of a blockchain network or decentralized application (dApp) and to earn rewards in return.

- Different ways of restaking for validators:
	1. **Native restaking** (*ETH -> EigenLayer*): Validators can restake their staked ETH natively by pointing their withdrawal credentials to the EigenLayer contracts. This is equivalent to L1 to EigenLayer yield stacking.
	2. **LSD restaking** (*LSD -> EigenLayer*): Validators can restake by staking their LSDs, i.e., ETH already restaked via protocols like Lido and Rocket Pool, by transferring their LSDs into the EigenLayer smart contracts. This is equivalent to DeFi to EigenLayer yield stacking.
	3. **ETH LP restaking** (*LP Token (ETH/ERC20 pair) -> EigenLayer*): Validators stake the LP token of a pair which includes ETH. This is equivalent to DeFi to EigenLayer yield stacking.
	4. **LSD LP restaking** (*LP Token (LSD/ETH pair) -> EigenLayer*): Validators stake the LP token of a pair which includes a liquid staking ETH token, such as Curve's stETH-ETH LP token, thus taking the L1 to DeFi to EigenLayer yield stacking route.
- The assets deposited into EigenLayer are under the **control of EigenLayer’s smart contracts**, and these staked assets act as collateral for the various services {also referred as AVSs or modules} (rollups, bridges, and data availability networks) built on top of EigenLayer. 
- By requiring staked assets to be locked up, EigenLayer ensures that its operators have a financial incentive to act honestly and securely. This is because if an operator were to behave maliciously or perform poorly, they would risk losing their staked assets through slashing conditions imposed by the services they serve.
- For example, the operator could restake their 10 ETH into one AVS with 10 ETH, or into multiple AVSs with a combined total of 10 ETH.

### Slashing Mechanism
- Cryptoeconomic security quantifies the cost that an adversary must bear in order to cause a protocol to lose a desired security property. This is referred to as the Cost-of-Corruption (CoC).
- Secure system: CoC > PfC (Profit-from-Corruption).
- A key function of the EigenLayer smart contracts is to hold the withdrawal credentials of Ethereum Proof-of-Stake (PoS) stakers. 
- If a staker who is restaked on EigenLayer is proven to have behaved adversarially while participating in an AVS, then that staker’s ETH will be subject to slashing. 
- Since the withdrawal address of the staker is set to the EigenLayer contracts, when the staker withdraws their ETH from participation in Ethereum consensus through EigenLayer, the withdrawn ETH will be slashed according to the on-chain slashing contract of the AVS.

>*Note*: EigenLayer does not issue a fungible token representing restaked positions, as each restaker may opt in to validating different combinations of modules and thus be subject to different sets of slashing risks.

### Risk Management
EigenLayer addresses two categories of risks: operator collusion and unintended slashing. 
1. For **operator collusion**, the ideal case is when all operators restake into all AVSs, making the cost of corrupting any AVS built on EigenLayer proportional to the total amount of stake in EigenLayer. However, in a realistic case where only a subset of operators opt in to a given AVS, where some set of operators may collude to steal funds from a set of AVSs. EigenLayer proposes solutions such as:
	1. restricting the PfC of any particular AVS, 
	2. creating an open-source cryptoeconomic dashboard to monitor operators' participation in restaking
	3. incentivizing operators to participate in only a low number of AVSs.
1. For **unintended slashing**, EigenLayer proposes two lines of defense: 
	1. security audits 
	2. the ability to veto slashing events through a governance layer in EigenLayer comprised of prominent members of the Ethereum and EigenLayer community. 
	These solutions aim to minimize the risk of unintended slashing before an AVS ossifies.
