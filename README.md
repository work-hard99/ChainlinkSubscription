# Why Chainlink Vrf2.5 Subscription Not Cancelling and Allowing Fund Withdrawal Even Though No Pending Request Exists?

## The Core Issue

I understand that while a pending request is there, you shouldn't be able to cancel. But the issue here is that once a pending request has failed (as also shown in UI), then the pending requests queue should be cleared automatically or some manual option must be given to the user.

So I tried a solution on Chainlink VRF2.5 (BNB testnet). I reproduced the whole situation that happened on BNB Mainnet. I underfunded my subscription with low LINK. I added a consumer and made a request. Again, my pending request was "pending" for 24 hours, then it showed the status as "failed." This is also mentioned in VRF2.5 documentation that, once a pending request crosses the 24-hour threshold, it will fail. Now I funded my subscription with enough LINK. But still, the pending request didn't clear. I then made a new request which was successful due to enough LINK.

I reviewed the Chainlink VRF2.5 contract on BNB, and it seems there is a bug in it. **Here is the catch** - my previous pending request which now is "failed" still remains in the VRF2_5 contract, --i.e to be exact-- `pendingReqCount` didn't decrease. Thus, the `pendingRequestExists()` method in the contract returns `true`, preventing me from removing my consumers or canceling my subscription.

I don't think Chainlink wants a design choice which ensures that even if **one failed transaction exists in your subscription, you can't cancel your subscription**. 

Failed transactions are not being cleared from `pendingRequests`, which in turn is blocking the removal of consumers and also the cancellation of subscriptions.

## Potential Solution

There is a solution, i.e., the `ownerCancelSubscription` method can cancel my subscription even if a so-called "pending" request exists in the contract. However, I suspect this can only be called by Chainlink authorized owners.

I have tried community forums, Discord, and Telegram. I think this is a major issue in the Chainlink contract. **Please look into this.**

### Subscription Details
- **Subscription ID:** 9281888378053720487440333824297839840858171687640417485419621993930679188239
- **Chainlink VRF2.5 Contract:** [BSCScan](https://bscscan.com/address/0xd691f04bc0c9a24edb78af9e005cf85768f694c9#writeContract)
- **Owner/Admin of Subscription ID:** [BSCScan](https://bscscan.com/address/0xb41b73b30198bee44c41943d5a0887110b85eebb)
- **Consumer Contract (BNB) which called `requestRandomWords` of VRF2.5 (BNB):** `0x42C524ad80cad5dbD9E836fb5136dDeCBd1f725A`

## Consumer Contract Code
```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.19;

import {VRFConsumerBaseV2Plus} from "@chainlink/contracts/src/v0.8/vrf/dev/VRFConsumerBaseV2Plus.sol";
import {VRFV2PlusClient} from "@chainlink/contracts/src/v0.8/vrf/dev/libraries/VRFV2PlusClient.sol";

contract RandomBridge is VRFConsumerBaseV2Plus {
    event RequestSent(uint256 requestId, uint32 numWords, address consumer, uint256 timestamp);
    event RequestFulfilled(uint256 requestId, address consumer, uint256 timestamp);

    struct RequestStatus {
        address consumer;
        bool fulfilled;
        bool exists;
        uint256[] randomWords;
    }

    mapping(uint256 => RequestStatus) private s_requests;
    uint256[] private requestIds;
    uint256 private lastRequestId;

    uint256 private s_subscriptionId;
    bytes32 private keyHash;
    uint32 private callbackGasLimit = 100000;
    uint16 private requestConfirmations = 3;
    address private nftAddress;

    constructor(address vrfCoordinator) VRFConsumerBaseV2Plus(vrfCoordinator) {}

    function setVRFConfigAndNFT(
        uint256 subscriptionId,
        bytes32 _keyHash,
        uint32 gasLimit,
        uint16 confirmations,
        address _nftAddress
    ) external onlyOwner {
        s_subscriptionId = subscriptionId;
        keyHash = _keyHash;
        callbackGasLimit = gasLimit;
        requestConfirmations = confirmations;
        nftAddress = _nftAddress;
    }

    function requestRandomWords(uint32 numWords, bool enableNativePayment)
        external
        returns (uint256 requestId)
    {
        require(
            msg.sender == owner() || msg.sender == nftAddress,
            "Invalid Caller"
        );
        requestId = s_vrfCoordinator.requestRandomWords(
            VRFV2PlusClient.RandomWordsRequest({
                keyHash: keyHash,
                subId: s_subscriptionId,
                requestConfirmations: requestConfirmations,
                callbackGasLimit: callbackGasLimit,
                numWords: numWords,
                extraArgs: VRFV2PlusClient._argsToBytes(
                    VRFV2PlusClient.ExtraArgsV1({
                        nativePayment: enableNativePayment
                    })
                )
            })
        );
        s_requests[requestId] = RequestStatus({
            consumer: msg.sender,
            randomWords: new uint256[](0),
            exists: true,
            fulfilled: false
        });
        requestIds.push(requestId);
        lastRequestId = requestId;
        emit RequestSent(requestId, numWords, msg.sender, block.timestamp);
        return requestId;
    }

    function fulfillRandomWords(
        uint256 _requestId,
        uint256[] calldata _randomWords
    ) internal override {
        require(s_requests[_requestId].exists, "request not found");
        s_requests[_requestId].fulfilled = true;
        s_requests[_requestId].randomWords = _randomWords;
        emit RequestFulfilled(_requestId, s_requests[_requestId].consumer, block.timestamp);
    }
}
```

## And the methods and state variables causing potential error in chainlink vrf2.5 contract - 

```solidity
function pendingRequestExists(uint256 subId) public view override returns (bool) {
    address[] storage consumers = s_subscriptionConfigs[subId].consumers;
    uint256 consumersLength = consumers.length;
    if (consumersLength == 0) {
        return false;
    }
    for (uint256 i = 0; i < consumersLength; ++i) {
        if (s_consumers[consumers[i]][subId].pendingReqCount > 0) {
            return true;
        }
    }
    return false;
}
```

```solidity
function cancelSubscription(uint256 subId, address to) external override onlySubOwner(subId) nonReentrant {
    if (pendingRequestExists(subId)) {
        revert PendingRequestExists();
    }
    _cancelSubscriptionHelper(subId, to);
}
```
The issue is  after 24 hours ,pending request becomes failed request . But then also , we can neither can remove our consumer nor cancel subscription . Let us say consumer is a malicious contract , and it Dos'ed many requests , then subscription owner doesn't have any mechanism which can remove this consumer . As if we try to remove consumer , even 1 failed request will not let it . BECAUSE PENDING REQUESTS AND FAILED REQUESTS ARE BEING TREATED SAME BY Chainlink VRF2_5 Contract .

There is a solution that I can see ,
```
contract VRFCoordinatorV2_5 is VRF, SubscriptionAPI, IVRFCoordinatorV2Plus {
...}
```
vrf2_5 inherits SubscriptionAPI contract . Now in this contract there is a method -

```
/**
   * @notice Owner cancel subscription, sends remaining link directly to the subscription owner.
   * @param subId subscription id
   * @dev notably can be called even if there are pending requests, outstanding ones may fail onchain
   */
  function ownerCancelSubscription(uint256 subId) external onlyOwner {
    address subOwner = s_subscriptionConfigs[subId].owner;
    if (subOwner == address(0)) {
      revert InvalidSubscription();
    }
    _cancelSubscriptionHelper(subId, subOwner);
  }
```

But who is authorised to call this method ? I dont't know , I studied the onwership chain -but it is delegated through multiple proxies and these are the possible owners:
```
[[0xB65B6d224351cD61f856c227b2d5cFc611E9B2d8]
[0x4CD248bC8Ce09eD1c0c0688c75C38902Ac761967]
[0xd8969669B7d71CdF8dEf1FAE729D3149427819A3]
[0xeBa9302747bB9485CBc343ABF131d574d340fdED]
[0xAB70e084DE1Da8289079C0aCc1b40BB05cD7b8EB]
[0x8B2f801068dF07B83A025Ba093Da4A79b7c2ea3E]
[0x0168d1035461A474A3DfCC461570c6C87b62b40D]
[0x7c704eb7c6454D9c2cD3F460386a21ADdA71a8a7]
[0x99B5C70f7cf97a44491466483439B630e875D773]]
```


## Questions and Concerns

Shouldn't there be a **mechanism or SOP** where a subscription owner can cancel subscriptions after requests have been designated as "failed" (i.e., more than 24 hours pending)?

As this issue may happen repeatedly, it is tedious and time-consuming for subscription owners to contact Chainlink support repeatedly.
![subscriptionManager](https://github.com/user-attachments/assets/dd0c8d14-2bd5-49a3-954d-181b2399be99)
