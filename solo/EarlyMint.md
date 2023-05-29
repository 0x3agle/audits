### Scope

The following smart contracts were in scope of the audit:
- `EarlyMint.sol`
- `Campaignable.sol`

The following number of issues were found during the audit by [3agle](https://twitter.com/0x_3agle) , categorized by their severity:

|Severity|Count|
|---|---|
|Critical|1|
|High| 2 |
|Medium| 3 |
|Low| 1 |
|QA| 3 |
|**Total**| **10** |

---

## Findings Summary

| ID     | Title                   | Severity |
| ------ | ----------------------- | -------- |
| [C-01]  | Inconsistent Mapping Update leads to severe NFT Assignment issues     | Critical     |
| [H-01]  | Order Reservation Exploit Impairs Campaign Execution     | High     |
| [H-02]  | Potential Denial of Service Risk in EarlyMint Contract     | High     |
| [M-01]  | Excess ETH not refunded in campaign reservations     | Medium     |
| [M-02]  | Improper authorizer address validation invalidates correct signatures for create campaigns and reserve orders   | Medium   |
| [M-03]  | Use of deprecated transfer function in `requestRefund`      | Medium      |
| [L-01]  | Two-step ownership transfer      | Low      |
| [QA-01]  | Initialize to default value      | QA      |
| [QA-02]  | `nonreentrant` should be the first modifier      | QA      |
| [QA-03]  | Refactor      | QA      |

---

## Detailed Findings

### **[C-01]** Inconsistent Mapping Update leads to severe NFT Assignment issues

#### Origin
https://github.com/johnny-sch-course/EarlyMintAudit/blob/main/contracts/EarlyMint.sol#L198-L202

#### Impact
- The smart contract contains a critical vulnerability that can impact the assignment of NFTs to users within a campaign. 
- The issue arises when users update their addresses using the `updateWalletAddress` function. Although the address is successfully updated, the corresponding mapping, specifically the `paidOrdersAddresses` mapping, is not updated accordingly. 
- Consequently, when the `executeMintForCampaign` function is triggered to assign NFTs, it utilizes the outdated mapping, leading to incorrect or missing NFT assignments.
- This vulnerability has a significant impact on the NFT assignment process within campaigns. Users who update their addresses will not receive the NFTs they are entitled to, resulting in frustration and potential financial losses. 
- The incorrect assignment of NFTs undermines the integrity of the campaign and can lead to disputes among users. Additionally, it can disrupt the proper management of NFT ownership and create challenges in reconciling and tracking NFT distributions.

#### Proof of Concept

Link to PoC: [PoC](https://gist.github.com/0x3agle/cf8ce5075e2d5905095e113cad79b482)

>Save this file in the test folder

**Output**
```bash
Address Update
Previous Address: 0x976EA74026E726554dB657fA54763abd0C3a0aa9
Updating Address...
Updated Address: 0x14dC79964da2C08b23698B3D3cc7Ca32193d9955
**********************************************
[*] Executing earlyMintForCampaigns() ...
**********************************************
    ✔ Exploit (184ms)
Address: 0x70997970c51812dc3a010c7d01b50e0d17dc79c8
Orders: 2
Address: 0x3c44cdddb6a900fa2b585dd299e03d12fa4293bc
Orders: 2
Address: 0x90f79bf6eb2c4f870365e785982e1f101e93b906
Orders: 2
Address: 0x15d34aaf54267db7d7c367839aaf71a00a2c6a65
Orders: 2
Address: 0x9965507d1a55bcc2695c58ba16fb37d819b0a4dc
Orders: 2
Address: 0x976ea74026e726554db657fa54763abd0c3a0aa9  // Here: Address not updated!
Orders: 0
```
**Walkthrough**
1.  The initial address shown is `0x976EA74026E726554dB657fA54763abd0C3a0aa9`. 
2.  The address is updated to a new value: `0x14dC79964da2C08b23698B3D3cc7Ca32193d9955`.
3.  The execution of the `earlyMintForCampaigns()` function begins.
4.  The function iterates through a list of addresses and their associated orders.
5.  Each address and the corresponding number of orders are displayed. In this example, there are multiple addresses with 2 orders each.
6.  However, when reaching the last address `0x976ea74026e726554db657fa54763abd0c3a0aa9`, it is observed that the address has not been updated, and the associated orders are shown as 0. This indicates that the mapping was not properly updated, resulting in the inability to assign the expected number of orders and potential loss of ownership for the affected user.

Overall, these steps highlight the vulnerability where updating an address does not properly update the associated mapping, leading to incorrect NFT assignment and potential ownership issues.

**Vulnerable code**
- The vulnerable code is located in the `updateWalletAddress` function:
```solidity
function updateWalletAddress(uint256 _campaignId, address _newAddress) public nonReentrant {
    require(paidOrders[_campaignId][msg.sender] > 0, "You must own at least 1 order to update your address");
    
    paidOrders[_campaignId][_newAddress] = paidOrders[_campaignId][msg.sender];
    delete paidOrders[_campaignId][msg.sender];
}
```
- In this function, the `paidOrders` mapping is updated correctly by assigning the order quantity from the old address to the new address. 
- However, the `paidOrdersAddresses` mapping, which holds the array of addresses for a given campaign, is not updated. 
- This inconsistency between the two mappings can lead to issues when executing the minting process (`executeMintForCampaign`) as it relies on the `paidOrdersAddresses` mapping to determine the addresses to assign NFTs to.

#### Mitigation 
- By updating both the `paidOrders` mapping and the `paidOrdersAddresses` mapping, the address update is properly reflected in the data structure. 
- This ensures that the `executeMintForCampaign` function will assign NFTs to the correct addresses, even after an address update. 
- Here's an updated version of the vulnerable code with the fix:
```solidity
function updateWalletAddress(uint256 _campaignId, address _newAddress) public nonReentrant {
    require(paidOrders[_campaignId][msg.sender] > 0, "You must own at least 1 order to update your address");
    require(_newAddress != address(0), "Invalid new address");
    
    uint8 orders = paidOrders[_campaignId][msg.sender];
    paidOrders[_campaignId][_newAddress] += orders; // Update the mapping with the new address
    delete paidOrders[_campaignId][msg.sender];
    
    // Update the paidOrdersAddresses mapping to reflect the address update
    address[] storage addresses = paidOrdersAddresses[_campaignId];
    for (uint256 i = 0; i < addresses.length; i++) {
        if (addresses[i] == msg.sender) {
            addresses[i] = _newAddress;
            break;
        }
    }
}

```

**Bonus Mitigation**
- A zero address check is also added which was previously not present. 
- Due to missing zero address check, an attacker could have updated his address to address(0). 
- So, when `executeMintForCampaign` would be called, the zero address of attacker will cause a revert while minting the NFTs. 
- That scenario would have rendered the complete campaign in a permanant DoS state (another **potentially** critical vulnerability).

---
### [H-01] Order Reservation Exploit Impairs Campaign Execution

#### Origin
https://github.com/johnny-sch-course/EarlyMintAudit/blob/main/contracts/EarlyMint.sol#L40-L58

#### Impact
- The bug allows attackers to exploit the system by reserving and refunding orders repeatedly, leading to a continuous growth in the **`paidOrdersAddresses`** array. As the array length increases, it results in escalating gas costs and can potentially cause a Denial-of-Service (DoS) situation. 
- While the **`maxOrders`** variable is decremented upon refund, the bug still allows attackers to manipulate the array length by exploiting the refund functionality multiple times within the specified order limit.
- The impact of this vulnerability remains significant as it can disrupt the execution of campaigns. The unbounded growth of the **`paidOrdersAddresses`** array adversely affects the gas consumption and responsiveness of the system, potentially rendering campaign execution unfeasible. 
- Moreover, the abuse of the refund process by attackers not only strains system resources but also hampers the experience of legitimate users who may face delays or inability to mint and receive their NFTs.

#### Proof of Concept

Link to PoC: [PoC](https://gist.github.com/0x3agle/b0d7454ac31f6eb15e841ec245968ca1)

>Save this file in the test folder

**Note**
- To successfully execute the PoC, the following lines should be added to `EarlyMint.sol`
- After line 9: 
```solidity
import "hardhat/console.sol";
```
- After line 122 :        
```solidity
console.log("Address: %s", key);
console.log("Orders: %s", orders);
```

**Walkthrough**
- This PoC is a sample demonstration that an attacker reserves 100 order and then requests a refund for all of them, effectively causing the `paidOrdersAddresses` array to increase in size by 100. 
- So, when `executeMintForCampaigns()` is executed, it will cycle through all the addresses and their values.
- Performing this attack has minimal cost to the attacker as creating new EOA does not cost anything band attacker only pays gas cost. 
- On the other hand, the campaign owner will need to pay a ton of gas cost for executing "Zero Orders"

**Description**
- The vulnerability lies in the **`requestRefund()`** function, which allows users to request a refund for their reserved orders. 
- However, the function only updates the **`paidOrders`** mapping to reset the order count for the user, while failing to remove the user's address from the **`paidOrdersAddresses`** array. 
- This leads to an accumulation of addresses in the array as more users reserve orders and request refunds over time.

**Vulnerable code**

```solidity
function requestRefund(uint256 _campaignId) public nonReentrant {
        Campaign storage campaign = campaignList[_campaignId];
        uint8 walletOrders = paidOrders[_campaignId][msg.sender];
        require(campaign.minted == false, "This campaign has already been minted");
        require(walletOrders > 0, "You don't hold any orders in this campaign");
        uint256 refundAmount = walletOrders * campaign.price;
        payable(msg.sender).transfer(refundAmount);
        campaign.ordersTotal -= walletOrders;
        campaign.campaignBalance -= refundAmount;
        paidOrders[_campaignId][msg.sender] = 0;
		
		// Missing code to remove address from paidOrdersAddresses mapping
		
        emit OrderRefunded(
            Order({
                campaignId: _campaignId,
                purchaserAddress: msg.sender,
                quantity: walletOrders,
                campaignOrdersTotal: campaign.ordersTotal,
                walletOrdersTotal: 0,
                externalId: campaign.externalId
            })
        );
    }
```

- Without removing the user's address from the **`paidOrdersAddresses`** mapping, the length of the array keeps growing indefinitely, potentially resulting in excessive gas costs and an unresponsive contract. 
- This situation can be leveraged by attackers to launch a denial-of-service (DoS) attack on the campaign execution process.

#### Mitigation 
- To mitigate this issue, you need to add code to remove the user's address from the **`paidOrdersAddresses`** mapping when a refund is requested. 
- Here's an updated version of the vulnerable code with the fix:

```solidity
function requestRefund(uint256 _campaignId) public {
        
    	// Refund logic...
    	
        // Remove user's address from paidOrdersAddresses mapping
	    uint256 addressIndex = findAddressIndex(paidOrdersAddresses[_campaignId],  msg.sender);
        if (addressIndex < paidOrdersAddresses[_campaignId].length - 1) {
            paidOrdersAddresses[_campaignId][addressIndex] = paidOrdersAddresses[_campaignId][paidOrdersAddresses[_campaignId].length - 1];
        }
        paidOrdersAddresses[_campaignId].pop();
}
  
function findAddressIndex(address[] memory addressesArray, address targetAddress) internal pure returns (uint256) {
        for (uint256 i = 0; i < addressesArray.length; i++) {
            if (addressesArray[i] == targetAddress) {
                return i;
            }
        }
        return uint256(-1);
    }
```
- With this code, after the refund amount is transferred to the user, it will remove the user's address from the **`paidOrdersAddresses`** mapping by finding the index of the user's address in the array and then replacing it with the last element in the array.
- Finally, it will remove the last element from the array using the **`pop()`** function.


---
### **[H-02]** Potential Denial of Service Risk in EarlyMint Contract

#### Origin
https://github.com/johnny-sch-course/EarlyMintAudit/blob/main/contracts/EarlyMint.sol#L29-L34

#### Impact
- The EarlyMint contract has a functionality that allows users to create campaigns. However, the current implementation of the `ensureUniqueCampaignExternalId` modifier poses a potential denial of service (DoS) risk. 
- Each time a new campaign is created, the modifier iterates through the data of all existing campaigns, which can result in a significant increase in gas consumption as the number of campaigns grows. 
- This inefficient approach may lead to scalability and performance issues in the future, potentially causing a DoS situation.
- The impact of this issue is as follows:
	1. *Gas Consumption*: As the number of campaigns in the contract increases, the gas consumption for creating new campaigns also increases. This can result in higher transaction costs and potential delays in executing transactions.
	2. *Scalability Concerns*: The inefficient iteration through all existing campaigns may hinder the scalability of the EarlyMint contract. As the number of campaigns grows, the time required to create a new campaign may become impractical, leading to poor user experience and potential disruptions to the system.

#### Proof of Concept

Link to PoC: [PoC](https://gist.github.com/0x3agle/58af5d22d81ffe77bfccf14c329e99ef)

>Save this file in the test folder
>*Note: A PoC demonstrating DoS will take ~10-15 mins to complete.*

**Output**
- The running of this PoC was timed at regular intervals to demonstrate the delay in creating every new 100 campaigns

|Campaigns|Duration|
|---|---|
|1-100|11.46 seconds|
|101-200|23.73 seconds|
|201-300|38.55 seconds|
|301-400|68.14 seconds|
|401-500|83.85 seconds|
|501-600|100.20 seconds|
|601-700|114.29 seconds|

- The statistics obtained from running the provided Proof of Concept (PoC) clearly demonstrate the increasing delay in creating new campaigns as the number of campaigns grows. 
- The duration for creating each batch of 100 campaigns progressively increases, indicating a potential performance degradation in the EarlyMint contract. 
- The data reveals a significant difference in duration between the initial batches and the later ones. For instance, creating the first 100 campaigns took approximately 11.46 seconds, while creating the last batch of 100 campaigns (601-700) took approximately 114.29 seconds. 
- This substantial increase in duration highlights the impact of the inefficient implementation on the contract's ability to efficiently process a large number of campaigns.

**Walkthrough**
The provided Proof of Concept (PoC) aims to demonstrate the gas cost issue associated with creating multiple campaigns in the `EarlyMint` contract. Here are the steps involved in the PoC:

1. Deploy the `EarlyMint` contract and the `EarlyMintTestERC721A` contract.
2. Run the "Exploit" test case, which creates 1000 campaigns sequentially.
3. Iterate through a loop from 0 to 999 and perform the following actions for each iteration: 
	1. Generate a unique `id` for the campaign.
	2. Generate a flat signature using the `getCreateSignature` function.
	3. Call the `createCampaign` function in the `EarlyMint` contract, passing the necessary campaign details and the generated signature.
	4. Wait for the transaction to be mined and the campaign to be created.
4.  Once all 1000 campaigns have been created, the test case completes and prints a success message.

- The gas cost increases with each iteration because the `ensureUniqueCampaignExternalId` modifier is iteratively checking the uniqueness of the `externalId` against all previously created campaigns in the `campaignList`. 
- As the number of campaigns grows, the iteration process becomes more time-consuming and resource-intensive, resulting in higher gas costs.

#### Mitigation 
- To address this gas cost issue, an alternative approach can be implemented that avoids the need for iteration over all existing campaigns. 
- One possible solution is to maintain a mapping that keeps track of the existence of `externalId` values. 
- Here's an updated version of the code that incorporates this improvement:
```solidity
mapping(string => bool) public campaignExternalIdExists;

modifier ensureUniqueCampaignExternalId(string memory _externalId) {
    require(!campaignExternalIdExists[_externalId], "External ID already exists");
    _;
}

function createCampaign(
    Campaign memory _campaign,
    bytes memory _signature
)
    public
    isValidCreate(
        _signature,
        _campaign.externalId,
        _campaign.fee,
        address(this),
        authorizerAddress
    )
    ensureUniqueCampaignExternalId(_campaign.externalId)
    nonReentrant
    returns (uint)
{
    require(_campaign.ordersPerWallet > 0, "You must allow at least 1 order per wallet");
    require(_campaign.maxOrders > 0, "You must allow at least 1 order");
    require(_campaign.paymentAddress != address(0), "NFT contract address cannot be 0x0");

    campaignList[campaignCounter] = Campaign({
        id: campaignCounter,
        externalId: _campaign.externalId,
        creator: _campaign.creator,
        paymentAddress: _campaign.paymentAddress,
        minted: false,
        fee: _campaign.fee,
        price: _campaign.price,
        maxOrders: _campaign.maxOrders,
        ordersTotal: 0,
        ordersPerWallet: _campaign.ordersPerWallet,
        campaignBalance: 0
    });
    
    campaignExternalIdExists[_campaign.externalId] = true;
    
    emit CampaignCreated(campaignList[campaignCounter]);
    campaignCounter++;
    return campaignCounter - 1;
}


```

- Explanation of the Updated Approach:
	- Introduced a mapping called `campaignExternalIdExists` to keep track of the existence of `externalId` values. The keys of the mapping represent the `externalId`, and the corresponding values indicate whether the `externalId` has already been used.
	- Modified the `ensureUniqueCampaignExternalId` modifier to check the `campaignExternalIdExists` mapping instead of iterating through the campaigns.
	- When creating a new campaign, before adding it to the `campaignList`, we check if the `externalId` already exists in the `campaignExternalIdExists` mapping. If it exists, the transaction reverts with an error message.
- With this approach, the gas cost is significantly reduced because there is no longer a need to iterate through the entire `campaignList` for each new campaign creation. Instead, the uniqueness of the `externalId` is checked using a simple mapping lookup.

---
### **[M-01]** Excess ETH not refunded in campaign reservations

#### Origin
https://github.com/johnny-sch-course/EarlyMintAudit/blob/main/contracts/EarlyMint.sol#L176-L196

#### Impact
- The `reserveOrder()` in EarlyMint.sol suffers from a vulnerability where excess ETH sent by users during the reservation process is not refunded. 
- The impact of this vulnerability is significant and can have the following consequences:
	1. *Financial Losses*: Users who inadvertently send more ETH than required during the reservation process will not receive a refund for the excess amount. This can result in financial losses for users and undermine their trust in the EarlyMint platform.
	2. *User Experience*: Failing to refund the excess ETH can create a poor user experience, as users expect a fair and transparent reservation process. Users may become frustrated or hesitant to engage in future transactions, impacting the overall adoption and growth of the platform.
- There exists a function `getPriceToReserveOrders()` which gives the exact ETH price for reserving an order in a campaign. 
- But, to increase the robustness and reliability of the EarlyMint protocol, the provided mitigation is hihgly recommended to aviod any unexpected circumstances in the future.
- As this is the form of "leaking value" - it is a medium risk issue.

#### Proof of Concept

Link to PoC: [PoC](https://gist.github.com/0x3agle/ef88d6bc11efdeba79580555a869e910)

>Save this file in the test folder

**Output**
```bash
EarlyMint Balance before reserveOrder: 0
User1 Balance before reserveOrder: 1e+22 // 1000 ETH
***************************************
[*] Reserving Order...
[*] Sending 2 ETH instead of 1 ETH...
[+] Order Reserved...
***************************************
EarlyMint Balance after reserveOrder: 2000000000000000000 // 2 ETH
//Excess of 1 ETH not returned back to the user
User1 Balance after reserveOrder: 9.997999738363427e+21 // 9997.99 ETH
```

#### Mitigation 
- Here's an updated version of the vulnerable code with the fix:

```solidity
function reserveOrder(
    uint256 _campaignId,
    uint8 _requestedOrderQuantity,
    bytes memory _signature
) public payable isValidReserve(_signature, _campaignId, address(this), authorizerAddress) {
    Campaign storage campaign = campaignList[_campaignId];
    uint256 totalPrice = campaign.price * _requestedOrderQuantity;
    require(msg.value >= totalPrice, "Not enough ETH sent");
    require(
        paidOrders[_campaignId][msg.sender] + _requestedOrderQuantity <=
            campaign.ordersPerWallet,
        "You cannot own more than the max orders per wallet"
    );
    require(
        campaign.maxOrders >= campaign.ordersTotal + _requestedOrderQuantity,
        "No more orders available"
    );

    paidOrders[_campaignId][msg.sender] += _requestedOrderQuantity;
    uint8 walletOrders = paidOrders[_campaignId][msg.sender];
    addPaidOrdersAddress(_campaignId, msg.sender);
    campaign.ordersTotal += _requestedOrderQuantity;
    campaign.campaignBalance += campaign.price * _requestedOrderQuantity;

    // Refund excess ETH
    uint256 refundAmount = msg.value - totalPrice;
    if (refundAmount > 0) {
        (bool success, ) = payable(msg.sender).call{value: refundAmount}("");
        require(success, "Failed to refund excess ETH");
    }

    emit OrderCreated(
        Order({
            campaignId: _campaignId,
            purchaserAddress: msg.sender,
            quantity: _requestedOrderQuantity,
            campaignOrdersTotal: campaign.ordersTotal,
            walletOrdersTotal: walletOrders,
            externalId: campaign.externalId
        })
    );
}

```

#### References
- https://code4rena.com/reports/2022-05-factorydao#h-01-speedbumppricegate-excess-ether-did-not-return-to-the-user
- https://code4rena.com/reports/2022-09-party#m-12-excess-eth-is-not-refunded
- https://code4rena.com/reports/2022-05-rubicon#m-04-rubiconrouter-excess-ether-did-not-return-to-the-user

---
### **[M-02]** Improper authorizer address validation invalidates correct signatures for create campaigns and reserve orders

#### Origin
https://github.com/johnny-sch-course/EarlyMintAudit/blob/main/contracts/EarlyMint.sol#L36-L38

#### Impact
- The smart contract contains a vulnerability in the authorizer address validation logic that can invalidate signatures for both create campaigns and reserve orders. 
- This issue arises when the authorizer address is unintentionally set to zero, leading to the rejection of valid signatures and compromising the integrity of the create and reserve processes.
- The vulnerability has the following impact on the contract's functionality:
	1. *Invalidation of create campaign signatures*: When the authorizer address is set to zero, the signatures provided for create campaign requests are considered invalid. This prevents legitimate users from creating campaigns as their signatures fail the validation check, causing disruption to the intended campaign creation process.  
	2. *Invalidation of reserve order signatures*: Similarly, valid signatures for reserve orders are invalidated due to the zero authorizer address. This means that users who attempt to reserve orders with proper signatures will encounter rejection, preventing them from participating in campaigns as intended.
	3. *Disruption of contract functionality*: The vulnerability disrupts the core functionality of the contract by rejecting valid create campaign and reserve order requests. This can result in user frustration, financial loss, and damage to the reputation of the contract.

#### Proof of Concept 

Link to PoC: [PoC](https://gist.github.com/0x3agle/db065fc955824b6647a8e309b0857257)

>Save this file in the test folder

**Description**
- The provided code is a Proof of Concept (PoC) demonstrating the steps to exploit the vulnerability where correct signatures are invalidated for create campaigns and reserve orders due to an unexpected error setting the authorizer address to zero. Let's break down the steps:
	1. Before hook: In this section, the necessary signers and contract instances are set up. The `EarlyMint` contract and `EarlyMintTestERC721A` contract are deployed by the `owner` signer.
	2. **"Correct Signature is invalidated for Create Campaigns"** test: This test aims to showcase the vulnerability in create campaigns. The authorizer address is intentionally set to zero using `em.updateAuthorizerAddress(ethers.constants.AddressZero)` to simulate an unexpected error. Then, a valid signature for creating a campaign is generated using `h.getCreateSignature` and passed to the `em.createCampaign` function. The expectation is that the transaction will be reverted with the reason string "Invalid signature," indicating that the valid signature is invalidated due to the zero authorizer address.
	3. **"Correct Signature is invalidated for Reserve Order"** test: This test demonstrates the vulnerability in reserve orders. First, the authorizer address is set back to `owner.address` using `em.updateAuthorizerAddress(owner.address)` to create a valid campaign. The same valid signature for creating a campaign is generated and used to create a campaign using `em.createCampaign`. Then, the authorizer address is set to zero again to simulate the unexpected error. A user attempts to reserve orders by providing a valid reserve signature (`reserveSig3`) and the required reserve amount. The expectation is that the transaction will be reverted with the reason string "Invalid signature," indicating that the valid reserve signature is invalidated due to the zero authorizer address.

#### Mitigation 
- To address the vulnerability where correct signatures are invalidated for create campaigns and reserve orders due to an unexpected error setting the authorizer address to zero, you can implement a mitigation measure that includes a zero address check in the `updateAuthorizerAddress` function. 
- This check ensures that the new authorizer address cannot be set to zero unintentionally, preventing the unintended invalidation of correct signatures.

```solidity
function updateAuthorizerAddress(address _authorizerAddress) public onlyOwner {
    require(_authorizerAddress != address(0), "Invalid authorizer address");
    authorizerAddress = _authorizerAddress;
}
```

---
### **[M-03]** Use of deprecated transfer function in `requestRefund`

#### Origin
https://github.com/johnny-sch-course/EarlyMintAudit/blob/main/contracts/EarlyMint.sol#L46

#### Impact
- When using `transfer()`, the fixed amount of gas provided (2300 gas units) can limit interactions with other contracts that require more gas to process the transaction. 
- If the ETH is being sent to an EOA then it won't cause issues (may change in future). 
- But, if the reciever is a smart contract then there are several scenarios where `transfer()` will fail:
	1.  If the receiving contract does not have a payable fallback function, the `transfer()` will fail because there is no function to receive the transferred funds.
	2. If the payable fallback function in the receiving contract consumes more than 2300 gas units, the `transfer()` will run out of gas during execution, causing the transaction to revert and the withdrawal to fail.
	3. If the payable fallback function in the receiving contract uses less than 2300 gas units, but the call is made through a proxy contract that consumes additional gas, the total gas usage can exceed 2300. In such cases, the `transfer()` will fail due to an out-of-gas error.

#### Mitigation 
 - To address these limitations, using `call.value()` is a preferred approach as it allows for more flexible gas allocation. 
 - By specifying the exact amount of gas required for the external contract call, you can ensure that contracts with complex fallback functions or interactions through proxies can be properly executed, avoiding potential failures during withdrawals.
- Here's an updated version of the vulnerable code with the fix:

```solidity
function requestRefund(uint256 _campaignId) public nonReentrant {
    Campaign storage campaign = campaignList[_campaignId];
    uint8 walletOrders = paidOrders[_campaignId][msg.sender];
    require(campaign.minted == false, "This campaign has already been minted");
    require(walletOrders > 0, "You don't hold any orders in this campaign");

    uint256 refundAmount = walletOrders * campaign.price;

    campaign.ordersTotal -= walletOrders;
    campaign.campaignBalance -= refundAmount;
    paidOrders[_campaignId][msg.sender] = 0;
	
	//Using Call instead of transfer
    (bool success, ) = payable(msg.sender).call{value: refundAmount}("");
    require(success, "Refund failed");

    emit OrderRefunded(
        Order({
            campaignId: _campaignId,
            purchaserAddress: msg.sender,
            quantity: walletOrders,
            campaignOrdersTotal: campaign.ordersTotal,
            walletOrdersTotal: 0,
            externalId: campaign.externalId
        })
    );
}
```

#### References
- https://code4rena.com/reports/2022-02-redacted-cartel/#m-04-send-ether-with-call-instead-of-transfer
- https://github.com/sherlock-audit/2023-03-sense-judging/issues/33


---
### [L-01] Two-step ownership transfer
- Two-step ownership transfer is recommended, in case a wrong address is supplied ownership is inaccessible.
```solidity
    function transferOwnership(address newOwner) public virtual onlyOwner {
        require(newOwner != address(0), "Ownable: new owner is the zero address");
        _transferOwnership(newOwner);
    }
    
    /**
     * @dev Transfers ownership of the contract to a new account (`newOwner`).
     * Internal function without access restriction.
     */
    function _transferOwnership(address newOwner) internal virtual {

        address oldOwner = _owner;

        _owner = newOwner;

        emit OwnershipTransferred(oldOwner, newOwner);

    }
```
- Consider implementing a two step process where the owner or controller nominates an account and the nominated account needs to call an `acceptOwnership()` function for the transfer of admin to fully succeed. 
- This ensures the nominated EOA account is a valid and active account.
```solidity

function transferOwnership(address newOwner) public onlyOwner {
    require(newOwner != address(0), "Ownable: new owner is the zero address");
    _pendingOwner = newOwner;
    emit OwnershipTransferRequested(_owner, _pendingOwner);
}

function acceptOwnership() public {
    require(msg.sender == _pendingOwner, "Ownable: caller is not the pending owner");
    address previousOwner = _owner;
    _owner = _pendingOwner;
    _pendingOwner = address(0);
    emit OwnershipTransferred(previousOwner, _owner);
}
```

---
### [QA-01] Initialize to default value
- Initialization to 0 or false is not necessary, as these are the default values in Solidity

```diff
EarlyMint.sol

-  uint256 public campaignCounter = 0; 
+  uint256 public campaignCounter; 

```

---
### [QA-02] `nonreentrant` should be the first modifier
- Before the flow enters into any other modifier, it should go through the nonreentrant modifier first.

```diff
EarlyMint.sol

    function createCampaign(
        Campaign memory _campaign,
        bytes memory _signature
    )
        public
+       nonReentrant
        isValidCreate(
            _signature,
            _campaign.externalId,
            _campaign.fee,
            address(this),
            authorizerAddress
        )
        ensureUniqueCampaignExternalId(_campaign.externalId)
-       nonReentrant
        returns (uint)
```

---
### [QA-03] Refactor 
- Refactor variable names to describe their actual usage in `withdraw()` & `withdrawAmount()` functions.
- "owner" -> "success"

```diff
EarlyMint.sol

function withdraw() public onlyOwner {
-     (bool owner, ) = payable(owner()).call{value: address(this).balance}("");
-     require(owner);

+     (bool success, ) = payable(owner()).call{value: address(this).balance}("");
+     require(success, "Transfer Failed");
}

  
function withdrawAmount(uint256 _amount) public onlyOwner {
	require(_amount <= address(this).balance, "Not enough funds");
	
-	(bool owner, ) = payable(owner()).call{value: _amount}("");
-	require(owner);

+	(bool success, ) = payable(owner()).call{value: _amount}("");
+	require(success, "Transfer Failed");
}
```

