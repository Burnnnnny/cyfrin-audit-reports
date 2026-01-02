**Lead Auditors**

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

[Immeas](https://twitter.com/0ximmeas)

**Assisting Auditors**



---

# Findings
## High Risk


### Zeeve admin could drain `ValidatorRewarder` by abusing off-chain BLS validation due to `QI` rewards being granted to failed registrations

**Description:** [`Ignite::releaseLockedTokens`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L580-L583)는 노드 등록이 실패했음을 나타내고 [`Ignite::redeemAfterExpiry`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L407) 호출을 통해 사용자가 스테이크를 회수할 수 있도록 하기 위한 매개변수로 [`failed`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L578) 불리언을 받습니다. [`Ignite::registerWithAvaxFee`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L251) 및 [`Ignite::registerWithErc20Fee`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L281)로 수행된 등록은 [첫 번째 조건 분기](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L416-L419) 내에서 처리됩니다. 그러나 [`Ignite::registerWithStake`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L202) 및 [`Ignite::registerWithPrevalidatedQiStake`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L361)를 통해 수행된 등록은 아래 표시된 [마지막 조건 분기](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L459-L471)까지 고려되지 않습니다:

```solidity
} else {
    avaxRedemptionAmount = avaxDepositAmount + registration.rewardAmount;
    qiRedemptionAmount = qiDepositAmount;

    if (qiRewardEligibilityByNodeId[nodeId]) {
        qiRedemptionAmount += validatorRewarder.claimRewards(
            registration.validationDuration,
            registration.tokenDeposits.tokenAmount
        );
    }

    minimumContractBalance -= avaxRedemptionAmount;
}
```

이는 `AVAX` 스테이크로 수행된 등록의 경우 보상 금액이 `0`에서 업데이트되지 않으므로 괜찮습니다. 그러나 사전 검증된 QI 스테이크로 수행된 등록의 경우 `ValidatorRewarder::claimRewards` 호출이 무조건 실행되어 전체 기간 동안의 원래 스테이크와 QI 보상을 반환합니다.

추가적으로, Zeeve 관리자는 이 동작을 악용하여 `ValidatorRewarder` 컨트랙트에서 `QI` 토큰을 빼낼 수 있습니다. 프론트엔드 검사를 우회하기 위해 배포된 컨트랙트와 직접 상호 작용한다고 가정할 때, `StakingContract::registerNode`에 잘못된 BLS 증명을 제공할 수 있습니다. 이 BLS 증명은 `NewRegistration` 이벤트가 감지될 때 오프체인 서비스에 의해 검증되며, 유효하지 않은 경우 `Ignite::releaseLockedTokens`가 호출됩니다.

Zeeve는 어느 정도 신뢰할 수 있는 엔터티이지만, 버너(burner) 사용자 주소로 매우 쉽고 비교적 눈에 띄지 않게 스테이킹할 수 있으며, 실패한 등록에 대한 `QI` 보상의 잘못된 처리와 오프체인 로직의 동작으로 인해 BLS 증명 검증 실패를 강제하여 `ValidatorRewarder` 컨트랙트를 고갈시킬 수 있습니다.

**Impact:** `Ignite::registerWithPrevalidatedQiStake`를 통해 수행된 실패한 등록의 사용자에게 `QI` 보상이 지급됩니다. Zeeve 관리자가 악용할 경우 `ValidatorRewarder` 컨트랙트 잔액 전체가 유출될 수 있습니다.

**Proof of Concept:** 다음 테스트를 `describe("Superpools")` 아래 `Ignite.test.js`에 추가할 수 있습니다:

```javascript
it("earns qi rewards for failed registrations", async function () {
  await validatorRewarder.setTargetApr(1000);
  await ignite.setValidatorRewarder(validatorRewarder.address);
  await grantRole("ROLE_RELEASE_LOCKED_TOKENS", admin.address);

  // AVAX $20, QI $0.01
  const qiStake = hre.ethers.utils.parseEther("200").mul(2_000);
  const qiFee = hre.ethers.utils.parseEther("1").mul(2_000);

  // approve Ignite to spend pre-validated QI (bypassing StakingContract)
  await qi.approve(ignite.address, qiStake.add(qiFee));
  await ignite.registerWithPrevalidatedQiStake(
    admin.address,
    "NodeID-Superpools1",
    "0x" + blsPoP.toString("hex"),
    86400 * 28,
    qiStake.add(qiFee),
  );

  // registration of node fails
  await ignite.releaseLockedTokens("NodeID-Superpools1", true);

  const balanceBefore = await qi.balanceOf(admin.address);
  await ignite.connect(admin).redeemAfterExpiry("NodeID-Superpools1");

  const balanceAfter = await qi.balanceOf(admin.address);

  // stake + rewards are returned to the user
  expect(Number(balanceAfter.sub(balanceBefore))).to.be.greaterThan(Number(qiStake));
});
```

**Recommended Mitigation:** `Ignite::releaseLockedTokens`에서 `qiRewardEligibilityByNodeId` 상태를 재설정하여 실패한 등록에 대해 QI 보상을 지급하지 않도록 하십시오:

```diff
    } else {
        minimumContractBalance += msg.value;
        totalSubsidisedAmount -= 2000e18 - msg.value;
+       qiRewardEligibilityByNodeId[nodeId] = false;
    }
```

**BENQI:** 커밋 [0255923](https://github.com/Benqi-fi/ignite-contracts/pull/16/commits/0255923e9c0d23c3fde71a6bdf1237ec4677c02d)에서 수정됨.

**Cyfrin:** 확인됨. 실패한 등록에 대해 `QI` 보상이 더 이상 부여되지 않습니다.

\clearpage
## Medium Risk


### Redemption of slashed registrations could result in DoS due to incorrect state update

**Description:** PAYG(Pay As You Go) 등록, 실패한 검증인 관련 잠재적 환불, 만료 후 스테이크 상환과 관련된 로직으로 인해 최소 `AVAX` 잔액이 `Ignite` 컨트랙트 내에 유지되어야 합니다. [`IgniteStorage::minimumContractBalance`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/IgniteStorage.sol#L97-L98) 상태 변수는 이 잔액을 추적하고 [`Ignite::withdraw`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L556-L572)가 검증을 시작하기 위해 적절한 양의 `AVAX`를 전송하도록 보장하는 역할을 합니다.

특정 등록에 대한 검증 기간이 만료되면, 상환 가능한 토큰과 함께 특권 `ROLE_RELEASE_LOCKED_TOKENS` 행위자에 의해 [`Ignite::releaseLockedTokens`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L574-L707)가 호출됩니다. 등록이 슬래싱(slashed)된 경우, `minimumContractBalance`는 슬래싱된 금액을 제외한 `msg.value`를 포함하도록 [업데이트됩니다](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L695-L697). 마지막으로 원래 등록자가 토큰을 상환하기 위해 [`Ignite::redeemAfterExpiry`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L402-L481)를 호출하면, 이 출금된 금액을 차감하기 위해 `minimumContractBalance` 상태가 다시 업데이트됩니다.

그러나 이 [상태 업데이트](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L458)는 올바르지 않습니다. `avaxDepositAmount`(이미 계산된 슬래싱 금액 포함)가 아닌 `avaxRedemptionAmount`를 감소시켜야 하기 때문입니다. 따라서 `minimumContractBalance`는 의도한 것보다 작아져, 감소의 언더플로로 인해 상환이 되돌아가거나(revert) `Ignite::withdraw` 호출이 컨트랙트에 의무를 이행할 자금이 충분하지 않은 상태로 남게 될 수 있습니다.

**Impact:** 현재 상환이 슬래싱되고 상태 업데이트가 언더플로되거나, 이전 상환이 슬래싱되고 의도한 것보다 더 많은 `AVAX`가 인출된 경우 상환이 되돌아갈 수 있습니다.

**Proof of Concept:** 다음 테스트를 `ignite.test.js`의 `describe("users can withdraw tokens after the registration becomes withdrawable")` 아래에 추가할 수 있습니다:

```javascript
it("with slashing using a slash percentage", async function () {
  // Add AVAX slashing percentage to trigger the bug (50% so the numbers are easy)
  await ignite.setAvaxSlashPercentage("5000");

  // Register NodeID-1 for two weeks with 1000 AVAX and 200k QI
  await ignite.registerWithStake("NodeID-1", blsPoP, 86400 * 14, {
    value: hre.ethers.utils.parseEther("1000"),
  });

  // Release the registration with `msg.value` equal to AVAX deposit amount to trigger slashing
  await ignite.releaseLockedTokens("NodeID-1", false, {
    value: hre.ethers.utils.parseEther("1000"),
  });

  // The slashed amount is decremented from the minumum contract balance
  expect(await ignite.minimumContractBalance()).to.equal(hre.ethers.utils.parseEther("500"));

  // Reverts on underflow since it tries to subtract 1000 (avaxDepositAmount) from 500 (minimumContractBalance)
  await expect(ignite.redeemAfterExpiry("NodeID-1")).to.be.reverted;
});
```
AVAX 슬래시 비율이 0이므로 기존 배포된 버전의 컨트랙트에서는 문제가 되지 않습니다.

**Recommended Mitigation:** `avaxDepositAmount` 대신 `avaxRedemptionAmount`만큼 `minimumContractBalance`를 감소시키십시오:

```diff
if (registration.slashed) {
    avaxRedemptionAmount = avaxDepositAmount - avaxDepositAmount * registration.avaxSlashPercentage / 10_000;
    qiRedemptionAmount = qiDepositAmount - qiDepositAmount * registration.qiSlashPercentage / 10_000;

-   minimumContractBalance -= avaxDepositAmount;
+   minimumContractBalance -= avaxRedemptionAmount;
} else {
```

**BENQI:** 커밋 [fb686b8](https://github.com/Benqi-fi/ignite-contracts/pull/16/commits/fb686b85ca88b0d90404f46f959f505fc1674fc0)에서 수정됨.

**Cyfrin:** 확인됨. 올바른 상태 업데이트가 이제 적용됩니다.


### The default admin role controls all other roles within `StakingContract`

**Description:** `StakingContract` 내에는 [`grantAdminRole()`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L986), [`revokeAdminRole()`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L1017), 및 [`updateAdminRole()`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L1048) 함수에 구현된 대로 Zeeve와 BENQI 관리자/슈퍼 관리자 역할 간의 의도된 분리가 있습니다. 의도는 관리자 역할이 해당 슈퍼 관리자에 의해 관리되도록 하는 것입니다. 그러나 `AccessControlUpgradeable::_setRoleAdmin`은 어떤 역할에 대해서도 호출되지 않으며 현재 구현은 컨트랙트가 초기화될 때 일시 중지 목적으로 [BENQI 슈퍼 관리자에게 부여된](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L169) 기본 관리자 역할을 고려하지 못합니다. 결과적으로 BENQI 슈퍼 관리자는 `AccessControlUpgradeable::grantRole` 및 `AccessControlUpgradeable::revokeRole`을 직접 호출하여 다른 모든 역할을 관리하는 데 사용될 수 있습니다. 이 동작은 `Ignite` 및 `ValidatorRewarder`에서 적절한 역할을 부여하는 데 사용되지만 `StakingContract`에서는 바람직하지 않습니다.

**Impact:** BENQI 슈퍼 관리자에게 부여된 기본 관리자 역할은 다른 모든 역할을 제어하는 데 사용될 수 있습니다.

**Proof of Concept:** 다음 테스트를 `stakingContract.test.js`의 `describe("updateAdminRole")` 아래에 추가할 수 있습니다:

```javascript
it("allows BENQI_SUPER_ADMIN to update ZEEVE_SUPER_ADMIN", async function () {
    const zeeveSuperAdminRole = await stakingContract.ZEEVE_SUPER_ADMIN_ROLE();

    // BENQI_SUPER_ADMIN can use OpenZepplin's grantRole to alter ZEEVE_SUPER_ADMIN_ROLE
    await stakingContract.connect(benqiSuperAdmin).grantRole(zeeveSuperAdminRole, otherUser.address);
    await stakingContract.connect(benqiSuperAdmin).revokeRole(zeeveSuperAdminRole, zeeveSuperAdmin.address);

    expect(await stakingContract.hasRole(zeeveSuperAdminRole, otherUser.address)).to.be.true;
    expect(await stakingContract.hasRole(zeeveSuperAdminRole, zeeveSuperAdmin.address)).to.be.false;
});
```

제공된 픽스처를 사용하여 동등한 Foundry 테스트를 실행할 수 있습니다:

```solidity
function test_defaultAdminControlsAllRolesPoC() public {
    assertEq(stakingContract.getRoleAdmin(stakingContract.DEFAULT_ADMIN_ROLE()), stakingContract.DEFAULT_ADMIN_ROLE());
    assertEq(stakingContract.getRoleAdmin(stakingContract.BENQI_SUPER_ADMIN_ROLE()), stakingContract.DEFAULT_ADMIN_ROLE());
    assertEq(stakingContract.getRoleAdmin(stakingContract.BENQI_ADMIN_ROLE()), stakingContract.DEFAULT_ADMIN_ROLE());
    assertEq(stakingContract.getRoleAdmin(stakingContract.ZEEVE_SUPER_ADMIN_ROLE()), stakingContract.DEFAULT_ADMIN_ROLE());
    assertEq(stakingContract.getRoleAdmin(stakingContract.ZEEVE_ADMIN_ROLE()), stakingContract.DEFAULT_ADMIN_ROLE());

    address EXTERNAL_ADDRESS = makeAddr("EXTERNAL_ADDRESS");
    vm.startPrank(BENQI_SUPER_ADMIN);
    stakingContract.grantRole(stakingContract.ZEEVE_SUPER_ADMIN_ROLE(), EXTERNAL_ADDRESS);
    vm.stopPrank();
    assertTrue(stakingContract.hasRole(stakingContract.ZEEVE_SUPER_ADMIN_ROLE(), EXTERNAL_ADDRESS));
}
```

**Recommended Mitigation:** 다음 중 하나를 고려하십시오:
1. 초기화 중 적절한 역할 관리자 설정.
2. 기본 관리자 역할을 제거하고 별도의 일시 중지자 역할 생성.
3. OpenZeppelin 함수를 재정의하여 직접 호출 방지.

**BENQI:** 커밋 [491a278](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/commit/491a278be80605bbf23cef71bed5227ea11d201e)에서 수정됨.

**Cyfrin:** 확인됨. OpenZeppelin 함수가 재정의되었습니다.


### Inconsistent transfers of native tokens could result in unexpected loss of funds

**Description:** `Ignite` 및 `StakingContract` 모두의 여러 프로토콜 함수는 사용자 및/또는 프로토콜 수수료/슬래싱된 토큰 수신자에게 기본 `AVAX`를 전송합니다.

`Ignite` 전체 [[1](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L215), [2](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L427), [3](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L477), [4](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L563), [5](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L639), [6](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L699)]에서 저수준 `.call()` 패턴이 사용됩니다. 그러나 이 동일한 동작은 `StakingContract`에서 따르지 않습니다. [433](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L433) 및 [484](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L484) 라인에서는 `.transfer()` 함수가 사용되고, [728](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L728-L733) 라인에서는 `_transferETHAndWrapIfFailWithGasLimit()`이 사용되며 모두 `2300` 가스 급여를 사용합니다.

이러한 다른 함수들은 잠재적인 재진입 공격을 방지하기 위해 사용되었을 수 있지만, [여기](https://consensys.io/diligence/blog/2019/09/stop-using-soliditys-transfer-now/)에 설명된 대로 가스 비용 변경을 완화하기 위해 저수준 호출을 사용하는 기본 토큰 전송이 `.transfer()`보다 선호됩니다. 433 및 484 라인의 인스턴스는 `WAVAX` 폴백이 없고 [여기](https://solodit.xyz/issues/m-01-swapsol-implements-potentially-dangerous-transfer-code4rena-tally-tally-contest-git) 및 연결된 예제에 설명된 대로 예상치 못한 자금 손실을 초래할 수 있으므로 특히 문제가 됩니다.

**Impact:** 전송 수신자(사용자 및 Zeeve 지갑 모두 해당)가 지불 가능한 폴백 함수를 구현하지 못하는 스마트 컨트랙트이거나 폴백 함수가 2300 가스 단위 이상을 사용하는 경우 예상치 못한 자금 손실이 발생할 수 있습니다. 예를 들어 수신자가 폴백 함수 로직으로 인해 실행에 2300 가스 이상을 사용하는 스마트 계정인 경우 이런 일이 발생할 수 있습니다.

**Recommended Mitigation:** `StakingContract`의 기본 토큰 전송 인스턴스를 저수준 호출을 사용하도록 수정하고 재진입을 방지하기 위해 필요한 조정을 하는 것을 고려하십시오.

**BENQI:** 커밋 [ee98629](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/ee98629daa2b815de4102d9af86f27fb36af67cf)에서 수정됨.

**Cyfrin:** 확인됨. `_transferETHAndWrapIfFailWithGasLimit()` 함수가 이제 `StakingContract` 전체에서 사용됩니다.


### Redemption of failed registration fees and pre-validated QI is not guaranteed to be possible

**Description:** [`Ignite::registerWithStake`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L202)는 수혜자(이 경우 `msg.sender`)가 `AVAX`를 받을 수 있는지 확인하기 위해 검증의 일부로 [저수준 호출](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L215-L216)을 수행합니다:

```solidity
// Verify that the sender can receive AVAX
(bool success, ) = msg.sender.call("");
require(success);
```

그러나 이것은 [`Ignite::registerWithAvaxFee`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L251)에서 누락되어 있으므로, 전송자가 `AVAX`를 받을 수 없는 컨트랙트인 경우 실패한 등록 수수료가 상환 가능하다는 보장이 없습니다.

마찬가지로 [`Ignite::registerWithPrevalidatedQiStake`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L361)는 수혜자에 대해 그러한 검증을 수행하지 않습니다. 스테이크 요구 사항이 `QI`로 제공되므로 문제가 없어 보일 수 있지만, [`Ignite::redeemAfterExpiry`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L407)에는 사전 검증된 `QI` 스테이크에 대해 0값(zero-value) 전송을 시도하는 [저수준 호출](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L477-L478)이 있습니다:

```solidity
(bool success, ) = msg.sender.call{ value: avaxRedemptionAmount}("");
require(success);
```

지정된 수혜자가 지불 가능한 폴백/수신(payable fallback/receive) 함수가 없는 컨트랙트인 경우 이 호출은 실패합니다. 또한 이 수혜자 컨트랙트가 불변인 경우 `QI` 스테이크는 업그레이드되지 않는 한 `Ignite` 컨트랙트에 잠기게 됩니다.

**Impact:** 실패한 `AVAX` 등록 수수료 및 사전 검증된 `QI` 스테이크는 `Ignite` 컨트랙트에 잠긴 채로 남게 됩니다.

**Proof of Concept:** 다음 독립형 Forge 테스트는 위에서 설명한 동작을 보여줍니다:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.15;

import "forge-std/Test.sol";

contract A {}

contract TestPayable is Test {
    address eoa;
    A a;

    function setUp() public {
        eoa = makeAddr("EOA");
        a = new A();
    }

    function test_payable() external {
        // Attempt to call an EOA with zero-value transfer
        (bool success, ) = eoa.call{value: 0 ether}("");

        // Assert that the call succeeded
        assertEq(success, true);

        // Attempt to call a contract that does not have a payable fallback/receive function with zero-value transfer
        (success, ) = address(a).call{value: 0 ether}("");

        // Assert that the call failed
        assertEq(success, false);
    }
}
```

**Recommended Mitigation:** `Ignite::registerWithAvaxFee` 및 `Ignite::registerWithPrevalidatedQiStake`에 검증을 추가하는 것을 고려하십시오. `Ignite::registerWithPrevalidatedQiStake` 내에서 저수준 호출을 수행하는 경우 `nonReentrant` 수정자를 추가하는 것도 고려하십시오.

**BENQI:** 커밋 [7d45908](https://github.com/Benqi-fi/ignite-contracts/pull/16/commits/7d45908fce2eefec90e5a67963311b250ae8c748)에서 수정됨. 커밋 [f671224](https://github.com/Benqi-fi/ignite-contracts/blob/f67122426c5dff6023da1ec9602c1959703db28e/src/Ignite.sol#L478-L481)에서 호출 전에 0이 아닌 검사가 추가되었으므로 사전 검증된 QI 스테이크 등록에 대해 더 이상 기본 토큰 전송이 없습니다.

**Cyfrin:** 확인됨. 저수준 호출이 `Ignite::registerWithAvaxFee`에 추가되었으며 사전 검증된 QI 스테이크 등록은 상환 시 더 이상 0값 호출을 하지 않습니다.


### Ignite fee is not returned for pre-validated `QI` stakes in the event of registration failure

**Description:** 사전 검증된 `QI` 스테이크에 적용되는 `1 AVAX` [Ignite 수수료](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L387)는 등록 시 [수수료 수령자에게 지급됩니다](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L388). 이 등록이 실패하면(예: 오프체인 BLS 증명 검증으로 인해) `Ignite::releaseLockedTokens`가 호출되면 등록이 [인출 가능으로 표시됩니다](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L620). 그러나 수수료는 이미 지불되었고 [사용자의 스테이크 금액에서 공제되었으므로](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L397), 환불된 `QI` 스테이크와 함께 반환되지 않습니다. 이 동작은 등록 실패 시 일반적으로 환불 불가능한 수수료를 환불하는 다른 등록 방법과 다릅니다.

**Impact:** 호스팅된 Zeeve 검증인에 등록한 사용자는 등록이 실패하면 Ignite 수수료를 환불받지 못합니다.

**Recommended Mitigation:** 등록 실패 시 사전 검증된 `QI` 스테이크에 대해 Ignite 수수료를 환불하십시오.

**BENQI:** 커밋 [f671224](https://github.com/Benqi-fi/ignite-contracts/commit/f67122426c5dff6023da1ec9602c1959703db28e)에서 수정됨.

**Cyfrin:** 확인됨. 수수료는 이제 `Ignite::releaseLockedTokens` 호출 중 성공적인 등록에서 수취합니다.

\clearpage

## Low Risk


### `StakingContract` refunds are affected by global parameter updates

**Description:** [`StakingContract::refundStakedAmount`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L691-741)이 BENQI 관리자에 의해 호출되면, 전역적으로 정의된 `refundPeriod`를 사용하여 다음 검증이 수행됩니다:

```solidity
require(
    block.timestamp > record.timestamp + refundPeriod,
    "Refund period not reached"
);
```

[`StakingContract::StakeRecord`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L69-77) 구조체에는 해당 멤버가 없으므로 스테이킹 당시의 `refundPeriod` 값을 저장하지 않습니다. 그러나 [`StakingContract::setRefundPeriod`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L675-689)가 업데이트된 기간으로 호출되면 기존 레코드의 기간이 예상보다 짧거나 길 수 있습니다.

**Impact:** 기존 레코드의 환불 기간은 전역 매개변수 업데이트의 영향을 받을 수 있습니다.

**Recommended Mitigation:** `StakeRecord` 구조체에 스테이킹 당시의 `refundPeriod` 값을 저장하기 위한 추가 멤버를 추가하는 것을 고려하십시오.

**BENQI:** 인지됨, 예상대로 작동함.

**Cyfrin:** 인지됨.


### Insufficient validation of Chainlink price feeds

**Description:** Chainlink `AggregatorV3Interface::latestRoundData`가 반환한 `price` 및 `updatedAt` 값의 유효성 검사는 다음 함수 내에서 수행됩니다:
- [`StakingContract::_validateAndSetPriceFeed`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L923-925)
- [`StakingContract::_getPriceInUSD`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L937-943)
- [`Ignite::_initialisePriceFeeds`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L187-L189)
- [`Ignite::registerWithStake`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L218-L223)
- [`Ignite::registerWithErc20Fee`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L291-L296)
- [`Ignite::registerWithPrevalidatedQiStake`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L373-L378)
- [`Ignite::addPaymentToken`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L806-L808)
- [`Ignite::configurePriceFeed`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L856-L858)

그러나 현재 존재하지 않는 권장되는 추가 유효성 검사가 아래에 표시되어 있습니다:

```solidity
(uint80 roundId, int256 price, , uint256 updatedAt, ) = priceFeed.latestRoundData();
if(roundId == 0) revert InvalidRoundId();
if(updatedAt == 0 || updatedAt > block.timestamp) revert InvalidUpdate();
```

**Impact:** 가장 중요한 가격 및 최신성 검증이 이미 존재하므로 영향은 제한적입니다.

**Recommended Mitigation:** 이 추가 유효성 검사를 포함하고 단일 내부 함수로 통합하는 것을 고려하십시오.

**BENQI:** 인지됨, 현재 유효성 검사가 충분한 것으로 간주됨.

**Cyfrin:** 인지됨.


### Incorrect operator when validating subsidisation cap

**Description:** 새 등록이 생성될 때 `Ignite::_registerWithChecks`는 등록에 대한 보조금 금액이 기존 총 보조금 금액에 추가될 때 최대값을 초과하지 않는지 [검증합니다](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L925-L928):

```solidity
require(
    totalSubsidisedAmount + subsidisationAmount < maximumSubsidisationAmount,
    "Subsidisation cap exceeded"
);
```

그러나 이 비교를 수행할 때 올바르지 않은 연산자가 사용됩니다.

**Impact:** 최대 보조금 금액을 정확히 충족하는 등록은 되돌려집니다(revert).

**Recommended Mitigation:**
```diff
    require(
-       totalSubsidisedAmount + subsidisationAmount < maximumSubsidisationAmount,
+       totalSubsidisedAmount + subsidisationAmount <= maximumSubsidisationAmount,
        "Subsidisation cap exceeded"
    );
```

**BENQI:** 커밋 [37446f6](https://github.com/Benqi-fi/ignite-contracts/pull/16/commits/37446f681d9f09000bb22682a9a153a0c7b23548)에서 수정됨.

**Cyfrin:** 확인됨. 연산자가 변경되었습니다.


### `StakingContract::slippage` can be outside of `minSlippage` or `maxSlippage`

**Description:** BENQI 관리자가 [`StakingContract::setSlippage`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L313-328)를 호출하여 `slippage` 상태를 설정할 때, 이 새 값이 `minSlippage` 및 `maxSlippage` 임계값 사이인지 검증됩니다:

```solidity
require(
    _slippage >= minSlippage && _slippage <= maxSlippage,
    "Slippage must be between min and max"
);
```

관리자는 또한 [`StakingContract::setMinSlippage`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L330-345) 및 [`StakingContract::setMaxSlippage`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L347-360)를 통해 `minSlippage` 및 `maxSlippage`를 모두 변경할 수 있습니다. 그러나 이 함수들 중 어느 것도 `slippage` 상태 변수가 경계 내에 유지되는지 확인하는 유효성 검사가 없습니다.

**Impact:** `StakingContract::setMaxSlippage` 또는 `StakingContract::setMinSlippage`를 잘못 사용하면 `slippage` 상태 변수가 범위를 벗어날 수 있습니다.

**Proof of Concept:** 다음 테스트를 `describe("setSlippage")` 아래 `stakingContract.test.js`에 추가할 수 있습니다:

```javascript
it("can have slippage outside of max and min", async function () {
    // slippage is set to 4
    await stakingContract.connect(benqiAdmin).setSlippage(4);
    // max slippage is updated below it
    await stakingContract.connect(benqiAdmin).setMaxSlippage(3);

    const slippage = await stakingContract.slippage();
    const maxSlippage = await stakingContract.maxSlippage();
    expect(slippage).to.be.greaterThan(maxSlippage);
});
```

**Recommended Mitigation:** `slippage` 상태 변수가 `setMin/MaxSlippage()`를 사용하여 설정된 경계 내에 있는지 검증하는 것을 고려하십시오.

**BENQI:** 커밋 [96e1b96](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/commit/96e1b9610970259cffdf44fd7cf1af527016b0ce) 및 [dbb13c5](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/commit/dbb13c55047e4ce52e39f833196fc78ed5c0cf8a)에서 수정됨.

**Cyfrin:** 확인됨.


### Lack of user-defined slippage and deadline parameters in `StakingContract::swapForQI` may result in unfavorable `QI` token swaps

**Description:** 사용자가 `StakingContract`와 상호 작용하여 호스팅된 노드를 프로비저닝할 때 [`StakingContract::stakeWithAVAX`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L457-L511) 또는 [`StakingContract::stakeWithERC20`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L513-L577)의 두 가지 방법 중에서 선택할 수 있습니다. 스테이킹된 토큰이 `QI`가 아닌 경우, `StakingContract::swapForQI`가 호출되어 Trader Joe를 통해 스테이킹된 토큰을 `QI`로 스왑합니다. 생성되면 검증인 노드는 `StakingContract::registerNode`를 통해 `QI`를 사용하여 [`Ignite`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L353-L400)에 [등록](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L441-448)됩니다.

`QI`로의 스왑 내에서 `amountOutMin`은 Chainlink 가격 데이터와 프로토콜에서 정의한 슬리퍼지 매개변수를 사용하여 [계산](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L844)됩니다:

```solidity
// Get the best price quote
uint256 slippageFactor = 100 - slippage; // Convert slippage percentage to factor
uint256 amountOutMin = (expectedQiAmount * slippageFactor) / 100; // Apply slippage
```

수신된 `QI`의 실제 금액이 이 `amountOutMin` 미만이면 트랜잭션이 [되돌아가지만](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L897-L900), 사용자는 더 유리한 스왑 실행을 보장하기 위해 더 작은 슬리퍼지 허용 오차를 원하는 경우, 프로토콜 정의 슬리퍼지에 의해 제한되어 기본 설정을 반영하지 못할 수 있습니다.

또한 `StakingContract::swapForQI`에서 `block.timestamp`로 지정된 스왑 [마감 기한](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L863)은 트랜잭션이 블록에 포함될 때마다 마감 기한 유효성 검사가 통과하므로 보호 기능을 제공하지 않습니다:

```solidity
uint256 deadline = block.timestamp;
```

이는 사용자를 불리한 가격 변동에 노출시킬 수 있으며 사용자가 자신의 마감 기한 매개변수를 제공할 수 있는 옵션을 제공하지 않습니다.

**Impact:** 사용자는 프로토콜에 의해 설정된 고정된 슬리퍼지 허용 오차로 인해 예상보다 적은 `QI` 토큰을 받을 수 있으며, 이는 잠재적으로 불리한 스왑 결과를 초래할 수 있습니다.

**Recommended Mitigation:** 사용자가 스왑 작업에 대해 `minAmountOut` 슬리퍼지 매개변수와 `deadline` 매개변수를 제공할 수 있도록 허용하는 것을 고려하십시오. 사용자가 지정한 `minAmountOut`이 더 큰 경우 프로토콜의 슬리퍼지 조정 금액을 덮어써야 합니다.

**BENQI:** 인지됨, Chainlink 가격 책정에 따라 `StakingContract::swapForQI` 내부에 이미 슬리퍼지 확인이 있습니다.

**Cyfrin:** 인지됨.

\clearpage
## Informational


### `AccessControlUpgradeable::_setupRole` is deprecated

**Description:** [`ValidatorRewarder::initialize`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/ValidatorRewarder.sol#L38-L59)에서 `DEFAULT_ADMIN_ROLE`은 [`AccessControlUpgradeable::_setupRole`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/ValidatorRewarder.sol#L54)을 사용하여 할당됩니다:

```solidity
_setupRole(DEFAULT_ADMIN_ROLE, _admin);
```

이 메서드는 OpenZeppelin의 [문서](https://docs.openzeppelin.com/contracts/4.x/api/access#AccessControl-_setupRole-bytes32-address-) 및 NatSpec에 기록된 대로 `AccessControlUpgradeable::_grantRole`을 선호하여 더 이상 사용되지 않습니다(deprecated):

```solidity
/**
 * @dev Grants `role` to `account`.
 * ...
 * NOTE: This function is deprecated in favor of {_grantRole}.
 */
function _setupRole(bytes32 role, address account) internal virtual {
    _grantRole(role, account);
}
```

`Ignite::initialize`도 [이것을 사용하지만](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L136), 컨트랙트가 이미 초기화되었으므로 우려할 사항은 아닙니다.

**Recommended Mitigation:** `ValdiatorRewarder::initialize` 및 가능하면 `Ignite::initialize`에서도 `AccessControlUpgradeable::_grantRole`을 사용하는 것을 고려하십시오.

**BENQI:** 커밋 [8db7fb5](https://github.com/Benqi-fi/ignite-contracts/commit/8db7fb5d4c27be03aa8c48437a17d9cca3bbc32d)에서 수정됨.

**Cyfrin:** 확인됨.


### Unchained initializers should be called instead

**Description:** 현재 구현에서는 즉각적인 문제가 아니지만, 초기화 함수를 직접 사용하는 대신 unchained(체인되지 않은) 등가물을 사용하는 것이 좋습니다. [`ValidatorRewarder::initialize`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/ValidatorRewarder.sol#L51-L52) 및 [`StakingContract::initialize`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L259-261)는 향후 [잠재적인 중복 초기화](https://docs.openzeppelin.com/contracts/5.x/upgradeable#multiple-inheritance)를 방지하도록 수정되어야 합니다.

이는 [`Ignite::initialize`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L132-134)에도 해당되지만, 컨트랙트가 이미 초기화되었으므로 우려할 사항은 아닙니다.

**Recommended Mitigation:** `ValidatorRewarder::initialize`, `StakingContract::initialize`, 그리고 가능하면 `Ignite::initialize`에서도 unchained 초기화 함수를 사용하는 것을 고려하십시오.

**BENQI:** 커밋 [cd4d43e](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/commit/cd4d43ea8397357b503a127d7ac7966b625a21b7)에서 수정됨.

**Cyfrin:** 확인됨. Unchained 초기화 함수가 이제 사용됩니다.


### Missing `onlyInitializing` modifier in `StakingContract`

**Description:** 현재 다른 곳에서 함수를 호출하는 것은 불가능하지만, [`StakingContract::initializeRoles`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L152) 및 [`StakingContract::setInitialParameters`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L185)는 초기화 중에만 호출되도록 제한되어야 하지만 `onlyInitializing` 수정자가 누락되어 있습니다. 후자는 적용된 [`StakingContract::_initializePriceFeeds`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L286)에 대한 [내부 호출](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L222-228)에 의해 처리됩니다.

**Recommended Mitigation:** `StakingContract::initializeRoles`와 가능하면 `StakingContract::setInitialParameters`에도 `onlyInitializing` 수정자를 추가하는 것을 고려하십시오.

**BENQI:** 커밋 [cd4d43e](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/commit/cd4d43ea8397357b503a127d7ac7966b625a21b7)에서 수정됨.

**Cyfrin:** 확인됨. 수정자가 추가되었습니다.


### Unnecessary `amount` parameter in `StakingContract::stakeWithERC20`

**Description:** [`StakingContract::stakeWithERC20`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L513-L577)을 통해 노드를 프로비저닝할 때 사용자는 지원되는 ERC20 토큰으로 지불할 수 있습니다. [`totalRequiredToken`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L527-530)은 [`avaxStakeAmount`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L35)(Ignite에 노드를 등록하는 데 필요) 및 [`hostingFee`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L525)(호스팅을 위해 Zeeve에 지불)를 기반으로 계산되며, 그 후 사용자로부터 컨트랙트로 [전송](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L534-539)됩니다:

```solidity
require(isTokenAccepted(token), "Token not accepted");
uint256 hostingFee = calculateHostingFee(duration);

uint256 totalRequiredToken = convertAvaxToToken(
    token,
    avaxStakeAmount + hostingFee
);

require(amount >= totalRequiredToken, "Insufficient token");

// Transfer tokens from the user to the contract
IERC20Upgradeable(token).safeTransferFrom(
    msg.sender,
    address(this),
    totalRequiredToken
);
```

사용자가 제공한 [`amount`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L521) 매개변수는 `totalRequiredToken`을 커버하는지 확인하는 데만 사용됩니다. 그러나 사용자가 컨트랙트에 전송에 대한 충분한 허용량을 제공하지 않은 경우 실행이 되돌아가므로 `amount` 매개변수는 불필요합니다.

**Recommended Mitigation:** `StakingContract::stakeWithERC20`에서 `amount` 매개변수를 제거하는 것을 고려하십시오.

**BENQI:** 인지됨. 목적은 프론트엔드에서 전달된 값이 `totalRequiredToken`보다 낮지 않은지 확인하여 일종의 슬리퍼지 확인 역할을 하는 것입니다. totalRequiredToken보다 작으면 사용자가 예상한 것보다 더 많은 토큰이 차감될 수 있습니다.

**Cyfrin:** 인지됨.


### Staking amount in QI should be calculated differently

**Description:** 현재 스테이크 토큰이 `QI`인 경우 `stakingAmountInQi`는 아래와 같이 [계산](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L542-544)됩니다:

```solidity
stakingAmountInQi = totalRequiredToken - convertAvaxToToken(token, hostingFee);
```

그러나 이로 인해 1 wei의 정밀도 손실이 발생할 수 있습니다.

**Proof of Concept:** Forge 픽스처와 소스 코드 내 로그를 사용하여 테스트되었습니다.

**Recommended Mitigation:** `avaxStakeAmount`를 기반으로 직접 `stakingAmountInQi`를 계산하는 것을 고려하십시오.

**BENQI:** 인지됨. 1 wei 정밀도 손실은 괜찮습니다.

**Cyfrin:** 인지됨.


### Tokens with more than `18` decimals will not be supported

**Description:** 현재 [`StakingContract::_getPriceInUSD`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L953)의 소수점 처리 로직으로 인해 `18` 소수점 이상의 토큰은 지원되지 않습니다:

```solidity
uint256 decimalDelta = uint256(18) - tokenDecimalDelta;
```

그리고 [`Ignite::registerWithErc20Fee`:](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L299)

```solidity
uint tokenAmount = uint(avaxPrice) * registrationFee / uint(tokenPrice) / 10 ** (18 - token.decimals());
```

**Recommended Mitigation:** 더 큰 소수점을 가진 토큰을 지원해야 하는 경우 이 로직을 수정하십시오.

**BENQI:** 인지됨, 예상대로 작동함.

**Cyfrin:** 인지됨.


### Incorrect revert strings in `StakingContract::revokeAdminRole`

**Description:** `StakingContract::revokeAdminRole`에는 잘못 복사된 것으로 보이는 두 개의 되돌리기 문자열 [[1](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L1021), [2](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L1036)]이 있으며 각각 다음과 같아야 합니다:
- "Cannot revoke role from the zero address"
- "Attempting to revoke an unrecognized role"

```solidity
function revokeAdminRole(bytes32 role, address account) public {
    // Ensure the account parameter is not a zero address to prevent accidental misassignments
    require(
        account != address(0),
        "Cannot assign role to the zero address"
    );

    /* snip: other conditionals */

    } else {
        // Optionally handle cases where an unknown role is attempted to be granted
        revert("Attempting to grant an unrecognized role");
    }

    /*snip: internal call & event emission */
}
```

**Recommended Mitigation:** 제안된 대로 되돌리기 문자열을 수정하십시오.

**BENQI:** 커밋 [cd4d43e](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/commit/cd4d43ea8397357b503a127d7ac7966b625a21b7) 및 [99d7f25](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/commit/99d7f25404a8693f5385d91863b098cd0639bb35)에서 수정됨.

**Cyfrin:** 확인됨. 되돌리기 문자열이 수정되었습니다.


### Placeholder recipient constants in `Ignite` should be updated before deployment

**Description:** `Ignite`의 [`FEE_RECIPIENT`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L43) 및 [`SLASHED_TOKEN_RECIPIENT`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L44) 상수가 테스트 목적으로 수정되었음을 이해하지만, 배포 전에 유효한 값으로 되돌려 수수료와 슬래싱된 토큰이 손실되지 않도록 하는 것이 중요합니다.

```solidity
address public constant FEE_RECIPIENT = 0xaAaAaAaaAaAaAaaAaAAAAAAAAaaaAaAaAaaAaaAa; // @audit-info - update placeholder values
address public constant SLASHED_TOKEN_RECIPIENT = 0xbBbBBBBbbBBBbbbBbbBbbbbBBbBbbbbBbBbbBBbB;
```

**Recommended Mitigation:** `Ignite` 컨트랙트 업그레이드를 수행하기 전에 상수를 업데이트하십시오.

**BENQI:** 인지됨, 이미 배포 체크리스트에 있음.

**Cyfrin:** 인지됨.


### Missing modifiers

**Description:** 재진입 가드 및 일시 중지 기능에 OpenZeppelin 라이브러리를 사용했음에도 불구하고 모든 외부 함수에 `nonReentrant` 및 pausable 수정자가 적용된 것은 아니므로 교차 함수 재진입이 가능할 수 있으며 의도하지 않았을 때 함수가 호출될 수 있습니다. 구체적으로:

- [`Ignite::registerWithStake`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L202)는 다른 등록 함수와 달리 `whenNotPaused` 수정자가 누락되어 있습니다.
- [`Ignite::registerWithPrevalidatedQiStake`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L361)는 `nonReentrant` 및 `whenNotPaused` 수정자가 모두 누락되어 있습니다.
- `Ignite::registerWithPrevalidatedQiStake`를 호출하는 [`StakingContract::registerNode`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L400)에도 `whenNotPaused` 수정자가 적용되지 않습니다.
- `whenPaused` 및 `whenNotPaused` 수정자는 두 컨트랙트의 일시 중지 가능한 함수에 적용되지 않습니다. 엄격히 요구되는 것은 아니지만 주의가 필요합니다.

**Recommended Mitigation:** 적절한 경우 필요한 수정자를 추가하십시오.

**BENQI:** 등록 함수는 일시 중지 확인을 시행하는 `_register()`를 호출합니다. `nonReentrant` 수정자는 `Ignite::registerWithPrevalidatedStake`가 안전하지 않은 외부 호출이 없는 허가형 함수이므로 추가되지 않았습니다. `whenNotPaused` 수정자는 커밋 [4956824](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/commit/4956824ad9703927c1eab68aa9b2e215cf91f62b)에서 `StakingContract::registerNode`에 추가되었습니다.

**Cyfrin:** 확인됨.


### Incorrect assumption that Chainlink price feeds will always have the same decimals

**Description:** `Ignite`에는 Chainlink 가격 피드의 소수점 정밀도가 동일하다고 가정하는 몇 가지 인스턴스 [[1](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L227), [2](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L299), [3](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L381)]가 있습니다. 현재 `AVAX`, `QI` 및 기타 모든 `USD` 피드가 `8` 소수점 정밀도로 가격을 반환하므로 문제가 발생하지 않지만, `ETH` 피드는 [여기](https://ethereum.stackexchange.com/questions/92508/do-all-chainlink-feeds-return-prices-with-8-decimals-of-precision)에 설명된 대로 `18` 소수점 정밀도로 가격을 반환하므로 명시적으로 처리해야 합니다.

**Recommended Mitigation:** Chainlink 가격 피드 소수점을 명시적으로 처리하는 것을 고려하십시오.

**BENQI:** 인지됨, USD 피드는 항상 8자리의 소수점을 갖습니다.

**Cyfrin:** 인지됨.


### Typo in `Ignite::registerWithPrevalidatedQiStake` NatSpec

**Description:** `Ignite::registerWithPrevalidatedQiStake` [NatSpec](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L355)에 오타가 있습니다.

**Recommended Mitigation:**
```diff
  /**
   * @notice Register a new node with a prevalidated QI deposit amount
-  * @param  beneficiary User no whose behalf the registration is made
+  * @param  beneficiary User on whose behalf the registration is made
   * @param  nodeId Node ID of the validator
   * @param  blsProofOfPossession BLS proof of possession (public key + signature)
   * @param  validationDuration Duration of the validation in seconds
   * @param  qiAmount The amount of QI that was staked
  */
```

**BENQI:** 커밋 [50c7c1a](https://github.com/Benqi-fi/ignite-contracts/pull/16/commits/50c7c1a81cbe7058116e53d178f61f828993ebc9)에서 수정됨.

**Cyfrin:** 확인됨.


### Magic numbers should be replaced by constant variables

**Description:** 매직 넘버 `10_000`, `2000e18`, `201`/`201e18`은 `Ignite` 컨트랙트 전체에서 사용되지만 대신 상수 변수로 만들어야 합니다.

**Recommended Mitigation:** 위에 설명된 매직 넘버 대신 상수를 사용하십시오.

**BENQI:** 인지됨, 변경하지 않음.

**Cyfrin:** 인지됨.


### Misalignment of `pause()` and `unpause()` access controls across contracts

**Description:** `Ignite`, `ValidatorRewarder` 및 `StakingContract`의 세 가지 컨트랙트 모두 특권이 있는 계정에 의해 트리거될 수 있는 일시 중지 기능이 있지만 모두 액세스 제어를 다르게 구현합니다:

- `Ignite`에서 [`pause()`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L709-L716)는 `ROLE_PAUSE` 역할이 부여된 계정만 호출할 수 있으며 [`unpause()`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L718-L725)의 경우 `ROLE_UNPAUSE` 역할입니다.

- `ValidatorRewarder`에서 [`pause()`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/ValidatorRewarder.sol#L126-L135) 및 [`unpause()`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/ValidatorRewarder.sol#L137-L146)는 모두 `ROLE_PAUSE` 역할이 부여된 계정만 호출할 수 있습니다. `ROLE_UNPAUSE` 역할은 [정의](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/ValidatorRewarder.sol#L22)되어 있지만 사용되지 않습니다.

- `StakingContract`에서 [`pause()`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L627-L632) 및 [`unpause()`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L634-L639)는 `DEFAULT_ADMIN_ROLE` 역할이 부여된 계정으로 제한됩니다.

**Recommended Mitigation:** 가장 유연성을 제공하는 `Ignite`의 `ROLE_PAUSE`/`ROLE_UNPAUSE` 설정을 사용하여 모든 컨트랙트 간의 역할 구성을 정렬하는 것을 고려하십시오.

**BENQI:** 인지됨, 변경하지 않음.

**Cyfrin:** 인지됨.


### Inconsistent price validation in `Ignite::registerWithStake`

**Description:** [`Ignite::registerWithErc20Fee`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L294), [`Ignite::registerWithPrevalidatedQiStake`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L376), 및 [`StakingContract::_getPriceInUSD`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L939-942)에서 가격은 `0`보다 큰지 검증됩니다. 그러나 [`Ignite::registerWithStake`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L221)에서 `AVAX` 가격은 `QI` 가격보다 큰지 검증됩니다. `AVAX` 가격은 현재 `QI` 가격보다 상당히 높으므로 원치 않는 되돌림이 발생하지 않지만, 이 검증은 다른 인스턴스와 일치하지 않으므로 수정해야 합니다.

**Recommended Mitigation:** `QI` 가격 대신 `AVAX` 가격이 `0`보다 크도록 요구하도록 검증을 수정하십시오.

**BENQI:** 인지됨, 변경하지 않음.

**Cyfrin:** 인지됨.

\clearpage
## Gas Optimization


### Unnecessary validation of `EnumerableSet` functions

**Description:** `EnumerableSet::add` 및 `EnumerableSet::remove`를 호출할 때 요소가 세트 내에 [이미 존재하는지](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/2f0bc58946db746c0d17a2b9d9a8e13f5a8edd7f/contracts/utils/structs/EnumerableSet.sol#L66) 먼저 확인할 필요가 없습니다. 이러한 함수는 내부적으로 동일한 유효성 검사를 수행하기 때문입니다. 대신 반환 값을 확인해야 합니다.

인스턴스 포함: [`StakingContract::addToken`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L579-595), [`StakingContract::removeToken`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L597-610), [`Ignite::addPaymentToken`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L785-L811), 및 [`Ignite::removePaymentToken`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L813-L830).

**Recommended Mitigation:** 예시로 다음 diff를 고려하십시오:

```diff
function addToken(
    address token,
    address priceFeedAddress,
    uint256 maxPriceAge
) external onlyRole(BENQI_ADMIN_ROLE) {
-    require(!acceptedTokens.contains(token), "Token already exists");

    _validateAndSetPriceFeed(token, priceFeedAddress, maxPriceAge);
-    acceptedTokens.add(token);
+    require(acceptedTokens.add(token), "Token already exists");
    emit TokenAdded(token);
}
```

**BENQI:** 커밋 [4956824](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/commit/4956824ad9703927c1eab68aa9b2e215cf91f62b) 및 [420ace6](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/commit/420ace61c598ca1fffc97c9d095b3fe0aafedc97)에서 수정됨.

**Cyfrin:** 확인됨. 유효성 검사가 업데이트되었습니다.


### Unnecessary validation in `StakingContract::registerNode`

**Description:** 사용자를 대신하여 새 검증인 노드가 생성되면 Zeeve 관리자는 `StakingContract::registerNode`를 호출하여 이를 보고합니다. 이 함수는 `Ignite`의 요구 사항에 따라 노드를 등록하기 위해 `Ignite::registerWithPrevalidatedQiStake`를 호출하기 전에 몇 가지 유효성 검사를 수행합니다.

아래에 표시된 `StakingContract::registerNode`에서 수행되는 이 유효성 검사 중 일부는 불필요하며 제거할 수 있습니다.

```solidity
require(
    bytes(nodeId).length > 0 && blsProofOfPossession.length > 0,
    "Invalid node or BLS key"
);
require(
    igniteContract.registrationIndicesByNodeId(nodeId) == 0,
    "Node ID already registered"
);
```

`nodeId`, `blsProofOfPossesion` 및 등록 인덱스에 대한 [이 모든 유효성 검사](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L406-413)는 `Ignite::_register`에서 [다시 수행됩니다](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L973-L979).

```solidity
// Retrieve the staking details from the stored records
require(stakeRecords[user].stakeCount > 0, "Staking details not found");
require(index < stakeRecords[user].stakeCount, "Index out of bounds"); // Ensures the index is valid

StakeRecord storage record = stakeRecords[user].records[index]; // Access the record by index
```

[이러한 요구 사항](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L414-416)이 제거되면 유효하지 않은 인덱스 또는 0 스테이크 카운트로 인해 초기화되지 않은 `StakeRecord`가 [반환됩니다](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L418). 따라서 순차적인 모든 요구 사항에서 실행이 되돌아갑니다.

```solidity
require(record.timestamp != 0, "Staking details not found");
require(isValidDuration(record.duration), "Invalid duration");
// Ensure the staking status is Provisioning
require(
    record.status == StakingStatus.Provisioning,
    "Invalid staking status"
);
```

그럼에도 불구하고 [타임스탬프 유효성 검사](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L419)는 기존 레코드가 초기화되지 않은 `timestamp`를 가질 수 있는 방법이 없고, [`status`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L422-425)에 대한 후속 확인에 의해 레코드 존재가 보장되므로 불필요합니다. 이는 레코드의 존재를 보장하는 데 필요하지 않고 [`Ignite::_regiserWithChecks`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L930-L936)에서 다시 수행되므로 [`duration`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L420) 유효성 검사 또한 불필요함을 의미합니다.

**Recommended Mitigation:** 위에서 설명한 불필요한 유효성 검사를 제거하는 것을 고려하십시오.

**BENQI:** 인지됨. 중복 확인으로 유지됨.

**Cyfrin:** 인지됨.


### Unnecessary conditional block in `Ignite::getTotalRegistrations` can removed

**Description:** [`Ignite::getTotalRegistrations`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L483-L494)의 조건문은 기본 자리표시자 등록 외에 등록이 없는 경우를 처리하기 위한 것입니다. 그러나 `registrations.length`가 `1`인 경우 이 확인 없이 함수가 여전히 `0`을 반환하고 기본 자리표시자 등록이 존재하므로 `registrations.length`가 `0`인 상태에 도달할 수 없으므로 이것은 불필요합니다.

```solidity
function getTotalRegistrations() external view returns (uint) {
    if (registrations.length <= 1) {
        return 0;
    }

    // Subtract 1 because the first registration is a dummy registration
    return registrations.length - 1;
}
```

**Recommended Mitigation:** 조건부 블록을 제거하는 것을 고려하십시오.

**BENQI:** 커밋 [58af671](https://github.com/Benqi-fi/ignite-contracts/pull/16/commits/58af6717bb96410e364f3da3f57a85e7577cac36)에서 수정됨.

**Cyfrin:** 확인됨. 유효성 검사가 제거되었습니다.


### Unnecessary validation in `Ignite::getRegistrationsByAccount`

**Description:** [`Ignite::getRegistrationsByAccount`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L512-L536)에는 함수에 인수로 전달된 인덱스에 대해 수행되는 [유효성 검사](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L526-L527)가 있습니다:

```solidity
require(from < to, "From value must be lower than to value");
require(to <= numRegistrations, "To value must be at most equal to the number of registrations");
```

[여기](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L529)의 언더플로 또는 [여기](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L532)의 인덱스 경계 초과로 인해 호출이 되돌아가므로 이것은 필요하지 않습니다. `to == from`인 경우 빈 배열이 반환됩니다.

**Recommended Mitigation:** 위에 표시된 유효성 검사를 제거하는 것을 고려하십시오.

**BENQI:** 커밋 [82cf4fc](https://github.com/Benqi-fi/ignite-contracts/pull/16/commits/82cf4fc9f753339460e41bc240a572d89c6fd7a8)에서 수정됨.

**Cyfrin:** 확인됨. 유효성 검사가 제거되었습니다.


### Unnecessary price feed address validation in `Ignite::configurePriceFeed`

**Description:** 가격 피드 주소 자체에 대해 유효성 검사를 수행하지 않고 반환된 데이터만 확인하는 [`Ignite::addPaymentToken`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L791)과 달리, [`Ignite::configurePriceFeed`](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L838)는 먼저 가격 피드 주소가 `address(0)`이 아닌지 [검증](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L844)합니다. `AggregatorV3Interface::latestRoundData`에 대한 [호출](https://github.com/Benqi-fi/ignite-contracts/blob/bbca0ddb399225f378c1d774fb70a7486e655eea/src/Ignite.sol#L856)은 반환 데이터를 abi 디코딩하는 동안 되돌아가므로 이것은 필요하지 않습니다.

**Proof of Concept:** 다음 독립형 Forge 테스트를 사용하여 이를 입증할 수 있습니다:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.15;

import "forge-std/Test.sol";

contract TestZeroAddressCall is Test {
    function test_zeroAddressCall() external {
        vm.expectRevert(); // see: https://book.getfoundry.sh/cheatcodes/expect-revert#gotcha:~:text=%E2%9A%A0%EF%B8%8F%20Gotcha%3A%20Usage%20with%20low%2Dlevel%20calls
        (bool revertsAsExpected, bytes memory returnData) =
            address(0).call(abi.encodeWithSelector(AggregatorV3Interface.latestRoundData.selector));
        assertFalse(revertsAsExpected); // the call itself does not revert

        vm.expectRevert(); // it's the decode step that reverts
        (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound) =
            abi.decode(returnData, (uint80, int256, uint256, uint256, uint80));
    }
}

interface AggregatorV3Interface {
    function latestRoundData()
        external
        view
        returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound);
}

```

**Recommended Mitigation:** 유효성 검사를 제거하는 것을 고려하십시오.

**BENQI:** 커밋 [7cbe588](https://github.com/Benqi-fi/ignite-contracts/pull/16/commits/7cbe58883cad6da79831b83fff96c2fefe348cdb)에서 수정됨.

**Cyfrin:** 확인됨. 유효성 검사가 제거되었으며 해당 테스트가 업데이트되었습니다.

\clearpage
