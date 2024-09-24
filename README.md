# Fjord-Token-submission

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01.  A DOS of unstakeAll will happen if the user stake and unstake,in the same epoch](#H-01)
    - ### [H-02. A DOS of unstakeAll happen when a user stake and unstakeVested in the same epoch](#H-02)
- ## Medium Risk Findings
    - ### [M-01. unstakeAll don't unstake a deposit made in the current epoch](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: Fjord

### Dates: Aug 20th, 2024 - Aug 27th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-08-fjord)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 2
- Medium: 1
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01.  A DOS of unstakeAll will happen if the user stake and unstake,in the same epoch            



## Summary

When a user stakes or unstakes, their total staked amount is updated if and only if their unredeemed epoch is higher than 0 and less than the current epoch.

&#x20;

The problem occurs when a user has a stake amount in a previous epoch and then stakes another amount and unstakes the amount from the previous epoch in the same epoch.

When they stake the new amount, their unredeemed epoch will be set to the current epoch, and the total staked amount will be updated. However, when they unstake, the unredeemedEpoch will be set to 0 because of this line of code.

## Vulnerability Details

Lets Imagine a user stake 1 fjord Token his unredeemedEpoch will be set to the current epoch as we can see here in the stake function (L-373) : 

```Solidity
  userData[msg.sender].unredeemedEpoch = currentEpoch;
```

After 8 weeks and 8 epochs, they decide to stake 1 more token. This code in the \_redeem function (L-729) will be triggered:

```Solidity
 if (ud.unredeemedEpoch > 0 && ud.unredeemedEpoch < currentEpoch) {
            // 2. Calculate rewards for all deposits since last redeemed, there will be only 1 pending unredeemed epoch
            DepositReceipt memory deposit = deposits[sender][ud.unredeemedEpoch];

            // 3. Update last redeemed and pending rewards
            ud.unclaimedRewards += calculateReward(
                deposit.staked + deposit.vestedStaked, ud.unredeemedEpoch, currentEpoch - 1
            );

            ud.unredeemedEpoch = 0;
            ud.totalStaked += (deposit.staked + deposit.vestedStaked);
        }
```

As we can see, their totalStaked will be set to 1e18, and at the end of the call, the unredeemedEpoch will be set again to the current epoch. Let’s imagine that in the same epoch, they decide to unstake the first deposit.

&#x20;

They will call the unstake function (L-449), but because of this, the if statement will be triggered since the unredeemedEpoch equals the current epoch and the deposit is empty: 

```Solidity
if (dr.staked == 0 && dr.vestedStaked == 0) {
            // no longer a valid unredeemed epoch
            if (userData[msg.sender].unredeemedEpoch == currentEpoch) {
                userData[msg.sender].unredeemedEpoch = 0;
            }
            delete deposits[msg.sender][_epoch];
            _activeDeposits[msg.sender].remove(_epoch);
        }
```

The unredeemedEpoch will be set to 0. The problem occurs here because the deposit will never be added to the totalStaked, as the if statement in the \_redeem function will prevent it.

&#x20;

So when they use unstakeAll, it will never work due to an underflow, where the sum of all the active deposits will be higher than the totalStaked.

```Solidity
        userData[msg.sender].totalStaked -= totalStakedAmount;

```

You can run this test in the StakeRewardScenarios contract in the stakeReward.t.sol file

```Solidity
    function test_DOS_UnstakeAll() public {
        vm.warp(block.timestamp + 7286594);
        vm.roll(block.number + 2);
        fjordStaking.stake(1);
        vm.warp(block.timestamp + 4803108);
        vm.roll(block.number + 1);
        fjordStaking.stake(1);
        fjordStaking.unstake(13, 1);
        vm.warp(block.timestamp + 3637164);
        vm.roll(block.number + 1);
        vm.expectRevert();
        fjordStaking.unstakeAll(); 

    }
```

## Impact

The user will not be able to use the unstakeAll function and will not be able to unstake this deposit. He will not earn any reward for it.

## Tools Used

Echidna

## Recommendations

I think that the best way to avoid this revert is to remove the if statement in the unstake function. This is even more important because this check doesn’t exist in the unstakeAll function.

## <a id='H-02'></a>H-02. A DOS of unstakeAll happen when a user stake and unstakeVested in the same epoch            



## Summary

When a user stakeVested or unstakeVested, their total staked amount is updated if and only if their unredeemed epoch is higher than 0 and less than the current epoch.

&#x20;

The problem occurs when a user has a stake amount in a previous epoch and then stakes another amount and unstake the amount from the previous epoch in the same epoch.

When they stake the new amount, their unredeemed epoch will be set to the current epoch, and the total staked amount will be updated. However, when they unstake, the unredeemedEpoch will be set to 0 because of this line of code.

## Vulnerability Details

Lets Imagine a user stake 1 fjord Token his unredeemedEpoch will be set to the current epoch as we can see here in the stakeVested function (L-397) : 

```Solidity
  userData[msg.sender].unredeemedEpoch = currentEpoch;
```

After 8 weeks and 8 epochs, they decide to stake 1  token. This code in the \_redeem function (L-729) will be triggered:

```Solidity
 if (ud.unredeemedEpoch > 0 && ud.unredeemedEpoch < currentEpoch) {
            // 2. Calculate rewards for all deposits since last redeemed, there will be only 1 pending unredeemed epoch
            DepositReceipt memory deposit = deposits[sender][ud.unredeemedEpoch];

            // 3. Update last redeemed and pending rewards
            ud.unclaimedRewards += calculateReward(
                deposit.staked + deposit.vestedStaked, ud.unredeemedEpoch, currentEpoch - 1
            );

            ud.unredeemedEpoch = 0;
            ud.totalStaked += (deposit.staked + deposit.vestedStaked);
        }
```

As we can see, their totalStaked will be set to the amount in the NFT of the first deposit, and at the end of the call, the unredeemedEpoch will be set again to the current epoch. Let’s imagine that in the same epoch, they decide to unstake the first deposit.

&#x20;

They will call the unstakeVested function (L-540), but because of this, the if statement will be triggered since the unredeemedEpoch equals the current epoch and the deposit is empty: 

```Solidity
if (dr.vestedStaked == 0 && dr.staked == 0) {
            // instant unstake
            if (userData[streamOwner].unredeemedEpoch == currentEpoch) {
                userData[streamOwner].unredeemedEpoch = 0;
            }
            delete deposits[streamOwner][data.epoch];
            _activeDeposits[streamOwner].remove(data.epoch);
        }
```

The unredeemedEpoch will be set to 0. The problem occurs here because the deposit will never be added to the totalStaked, as the if statement in the \_redeem function will prevent it.

&#x20;

So when they use unstakeAll, it will never work due to an underflow, where the sum of all the active deposits will be higher than the totalStaked.

```Solidity
        userData[msg.sender].totalStaked -= totalStakedAmount;

```

You can create a new file in the integration folder and copy paste this code below you just have to provide  an RPC URL in the setUp and run the test :

```Solidity
// SPDX-License-Identifier: GPL-2.0
pragma solidity ^0.8.0;

import "../../src/FjordStaking.sol";
import {FjordPoints} from "../../src/FjordPoints.sol";
import {FjordAuction, ERC20Burnable} from "src/FjordAuction.sol";
import {MockERC20} from "solmate/test/utils/mocks/MockERC20.sol";
import {FjordToken} from "src/FjordToken.sol";
import {AuctionFactory} from "src/FjordAuctionFactory.sol";
import {vm} from "@chimera/Hevm.sol";
import {SafeMath} from "lib/openzeppelin-contracts/contracts/utils/math/SafeMath.sol";
import {ISablierV2LockupLinear} from "lib/v2-core/src/interfaces/ISablierV2LockupLinear.sol";
import {Broker, LockupLinear} from "lib/v2-core/src/types/DataTypes.sol";
import {ISablierV2Lockup} from "lib/v2-core/src/interfaces/ISablierV2Lockup.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ud60x18} from "@prb/math/src/UD60x18.sol";
import "lib/v2-core/src/libraries/Errors.sol";
import {Test, console} from "forge-std/Test.sol";

 contract FjordStakingPoc is Test {
    event LogUint256(string, uint256);
    event LogAddress(string, address);

    address constant USER1 = address(0x10000);
    address constant USER2 = address(0x20000);
    address constant USER3 = address(0x30000);
    address[] internal users;
    FjordPoints internal fjordPoints;
    MockERC20 internal fjordToken;
    MockERC20 internal mockERC20;
    AuctionFactory internal auctionFactory;
    FjordStaking internal fjordStaking;
    address internal adminRewardAddress = vm.addr(25);
    address minter = vm.addr(26);
    address newMinter = vm.addr(27);
    address alice = vm.addr(28);
    address bob = vm.addr(29);
    FjordAuction internal fjordAuction;
    uint256 internal currentSalt;
    address sender;
    address currentTargetContract;
    uint256 lastEpochAddReward;
    address internal constant SABLIER_ADDRESS = address(0x3962f6585946823440d274aD7C719B02b49DE51E);
    ISablierV2LockupLinear SABLIER = ISablierV2LockupLinear(SABLIER_ADDRESS);
    mapping(address => uint256[]) internal fromUserToStreamIDs;
    mapping(uint256 => bool) internal streamIDisCancelable;
    uint256 internal startTime;
    uint256 internal handlerTotalPoints;

    function setUp() public virtual  {
        uint256 fork = vm.createFork("You Rpc URL");
        vm.selectFork(fork);
        fjordToken = new MockERC20("Test", "TT", 18);
        mockERC20 = new MockERC20("Test", "TT", 18);
        fjordPoints = new FjordPoints();
        auctionFactory = new AuctionFactory(address(fjordPoints));
        fjordStaking =
            new FjordStaking(address(fjordToken), adminRewardAddress, SABLIER_ADDRESS, address(0), address(fjordPoints));
        fjordPoints.setStakingContract(address(fjordStaking));
        users.push(USER1);
        users.push(USER2);
        users.push(USER3);
        mockERC20.mint(address(this), 100_000_000_000e18);
        mockERC20.approve(address(auctionFactory), 100_000_000_000e18);
        mockERC20.mint(USER1, 100_000_000_000e18);
        mockERC20.mint(USER2, 100_000_000_000e18);
        mockERC20.mint(USER3, 100_000_000_000e18);
        uint256 balance = uint256(100000000e18) / 3;
        fjordToken.mint(USER1, balance);
        fjordToken.mint(USER2, balance);
        fjordToken.mint(USER3, balance);
        fjordToken.mint(address(this), 100_000_000_000e18);
        fjordToken.approve(address(fjordStaking), 100_000_000_000e18);
        for (uint256 i = 0; i < users.length; i++) {
            fjordStaking.addAuthorizedSablierSender(users[i]);
        }
         fjordStaking.addAuthorizedSablierSender(address(this));

}
function createStream() internal returns (uint256 streamID) {
        return createStream(address(this), mockERC20, false, 100 ether);
    }

    function createStream(MockERC20 asset, bool isCancelable) internal returns (uint256 streamID) {
        return createStream(address(this), asset, isCancelable, 100 ether);
    }

    function createStream(address user, MockERC20 asset, bool isCancelable) internal returns (uint256 streamID) {
        return createStream(user, asset, isCancelable, 100 ether);
    }

    function createStream(address user, MockERC20 asset, bool isCancelable, uint256 amount)
        internal
        returns (uint256 streamID)
    {
        asset.mint(user, amount);
        vm.prank(user);
        asset.approve(address(SABLIER), amount);

        LockupLinear.CreateWithDurations memory params;
        params.sender = user;
        params.recipient = user;
        params.totalAmount = uint128(amount);
        params.asset = IERC20(address(asset));
        params.cancelable = isCancelable;
        params.durations = LockupLinear.Durations({cliff: uint40(2 days), total: uint40(10 days)});
        params.broker = Broker(address(0), ud60x18(0));
        params.transferable = true;
        vm.prank(user);
        streamID = SABLIER.createWithDurations(params);
        vm.prank(user);
        SABLIER.approve(address(fjordStaking), streamID);
    }

    function test_DOS_UnstakeAllVested() public {
        uint256 streamId =  createStream(address(this), fjordToken, false, 10 ether); 
        fjordStaking.stakeVested(streamId);
        vm.warp(block.timestamp + 4798066);
        vm.roll(block.number + 1);
        fjordStaking.stake(1);
        fjordStaking.unstakeVested(streamId);
        vm.warp(block.timestamp + 3672339);
        vm.roll(block.number + 1);
        vm.expectRevert();
        fjordStaking.unstakeAll();
    }
}
```

## Impact

The user will not be able to use the unstakeAll function and will not be able to unstake this deposit. He will not earn any reward for it because it will never be add to the totalStaked of the user.

## Tools Used

Echidna

## Recommendations

I think that the best way to avoid this revert is to remove the if statement in the unstake function. This is even more important because this check doesn’t exist in the unstakeAll function.

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. unstakeAll don't unstake a deposit made in the current epoch            



## Summary

A user can unstake a deposit in 2 situations : 

A deposit can be unstaked in two cases: if it has waited for 6 epochs or if the deposit was made during the same epoch as the unstake call.

The problem is that the unstakeAll function(L-570) ignore all the deposits that didn't wait at least 6 epoch.

## Vulnerability Details

When a user call  unstakeAll it will loop over all the `activeDeposit` of the sender it will check if the activeDeposit is old enought to unstake it as we can see here : 

```Solidity
        uint256[] memory activeDeposits = getActiveDeposits(msg.sender);
        if (activeDeposits.length == 0) revert NoActiveDeposit();

        for (uint16 i = 0; i < activeDeposits.length; i++) {
            uint16 epoch = uint16(activeDeposits[i]);
            DepositReceipt storage dr = deposits[msg.sender][epoch];

            if (dr.epoch == 0 || currentEpoch - epoch <= lockCycle) continue;

            totalStakedAmount += dr.staked;
```

the proble here is that the if statement will make the function ignore the activeDeposit if it has been made in the same epoch as the unstakeAll call. \
\
 This contradicts the unstake function(L-449) as we can see here : 

```Solidity
 // _epoch is same as current epoch then user can unstake immediately
        if (currentEpoch != _epoch) {
            // _epoch less than current epoch then user can unstake after at complete lockCycle
            if (currentEpoch - _epoch <= lockCycle) revert UnstakeEarly();
        }
```

You can run this test in the StakeRewardScenarios contract in the StakeReward.t.sol file : 

```Solidity
 function test_StakeAll_POC() public {
        fjordStaking.stake(1);
        fjordStaking.unstake(1, 1);
        (, uint256 stakedAmount,) = fjordStaking.deposits(address(this), 1);

        assertEq(stakedAmount, 0);
        fjordStaking.stake(1);
        fjordStaking.unstakeAll();
        (, uint256 stakedAmountAfter,) = fjordStaking.deposits(address(this), 1);

        assertEq(stakedAmountAfter, 1);
    }
```





## Impact

The function UnstakeAll don't unstake all the unstakable deposits.

## Tools Used

Echidna

## Recommendations

You can just add a if statement that make sure that the function unstake the deposit of the current epoch.

```Solidity
if(currentEpoch != epoch){
if (dr.epoch == 0 || currentEpoch - epoch <= lockCycle) continue;
}
```





