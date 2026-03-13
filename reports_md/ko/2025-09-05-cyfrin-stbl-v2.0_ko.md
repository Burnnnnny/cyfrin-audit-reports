**수석 감사**

[0kage](https://twitter.com/0kage_eth)

[Stalin](https://twitter.com/0xStalin)

**보조 감사**


---

# 발견 사항

## 중간 위험 (Medium Risk)

### `STBL_PT1_Issuer::generateMetaData`에서의 잘못된 헤어컷 자산 가치 변환

**설명:** `STBL_PT1_Issuer::generateMetaData` (및 이에 상응하는 `STBL_LT1_Issuer`)는 `haircutAmountAssetValue`를 계산할 때 잘못된 오라클 변환 함수를 사용하여 NFT 메타데이터에 수학적으로 잘못된 값이 저장되는 결과를 초래합니다. 이 함수는 입력값(`MetaData.haircutAmount`)이 자산 통화일 것으로 예상하는 `fetchForwardPrice()`를 사용하지만, 헤어컷 금액은 이미 USD로 변환된 상태입니다(USD 기준인 `stableValueGross`에 적용되었기 때문).

이 문제는 USD 금액을 자산 금액을 예상하는 함수에 잘못 전달하는 단위 변환 오류에서 비롯됩니다:

```solidity
function generateMetaData(uint256 assetValue) internal view returns (YLD_Metadata memory MetaData) {
    // ... 기타 계산 ...

    // @audit 1단계: fetchForwardPRice를 사용하여 자산을 USD로 변환
    MetaData.stableValueGross = iSTBL_PT1_AssetOracle(AssetData.oracle)
        .fetchForwardPrice(MetaData.assetValue);

    // @audit 2단계: USD 기준으로 헤어컷 계산
    MetaData = MetaData.calculateDepositFees(); // haircutAmount를 USD로 설정

    // @audit 버그 - 잘못된 변환 함수 사용됨
    MetaData.haircutAmountAssetValue = iSTBL_PT1_AssetOracle(AssetData.oracle)
        .fetchForwardPrice(MetaData.haircutAmount);  // @audit 가격의 역수가 사용되어야 함
        //                 ^^^^^^^^^^^^^^^^^^^^
        // @audit 이는 자산 금액을 예상하지만 USD 금액을 받음
}
```

여기서 올바른 변환은 오라클 가격의 역수를 적용하여 자산 토큰 단위의 헤어컷 가치를 얻는 것입니다.

**파급력:** 핵심 볼트 회계 로직은 영향을 받지 않지만, `haircutAmountAssetValue` 메타데이터가 올바르지 않습니다. 메타데이터를 읽는 오프체인 시스템은 잘못된 값을 얻게 됩니다.

**개념 증명 (Proof of Concept):** 다음은 모의 계산입니다:

```text
1단계: stableValueGross = fetchForwardPrice(100e18)
       = (110000000 * 100e18) / 1e8 = 110e18 USD

2단계: haircutAmount = (110e18 * 500) / 10000 = 5.5e18 USD

3단계 (잘못됨): haircutAmountAssetValue = fetchForwardPrice(5.5e18)
       = (110000000 * 5.5e18) / 1e8 = 6.05e18

3단계 (올바름): haircutAmountAssetValue = fetchInversePrice(5.5e18)
       = (5.5e18 * 1e8) / 110000000 = 5e18
```

**권장 완화 방법:** `haircutAmountAssetValue`를 계산하기 위해 `fetchInversePrice`를 사용하는 것을 고려하십시오.

**STBL:** 커밋 [1adc1f2](https://github.com/USD-Pi-Protocol/contract/commit/1adc1f2d05dcbcee89826ab9b7d625642c0834bd)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `STBL_USST::bridgeBurn`에서의 높은 중앙화 위험

**설명:** `STBL_USST::bridgeBurn`의 현재 구현은 `BRIDGE_ROLE`이 사용자 승인 없이 임의의 주소에서 토큰을 소각할 수 있도록 허용합니다. 이는 프로토콜 자체의 `STBL_Token` 컨트랙트에 이미 더 안전한 접근 방식이 시연되어 있음에도 불구하고 불필요한 중앙화 위험을 초래합니다.

```solidity
// STBL_USST.sol
function bridgeBurn(
    address _from,      // @audit 어떤 주소든 가능
    uint256 _amt,
    bytes memory _data
) external whenNotPaused onlyRole(BRIDGE_ROLE) {
    _burn(_from, _amt);  // @audit 동의 없이 임의의 주소에서 소각
    emit BridgeBurn(_from, _amt, _data);
}
```
위의 접근 방식은 `BRIDGE_ROLE`(`DEFAULT_ADMIN`으로 초기화됨)이 어떤 주소에서든 토큰을 소각할 수 있게 합니다. `STBL_TOKEN`에 이미 구현된 대안적인 구현은 훨씬 안전합니다:

```solidity
// STBL_Token.sol
function bridgeBurn(uint256 _amt) external whenNotPaused onlyRole(BRIDGE_ROLE) {
    _burn(_msgSender(), _amt);  // @audit 호출자(브릿지) 자신의 토큰만 소각
}
```

**파급력:** 브릿지 컨트랙트가 침해당할 경우 단일 트랜잭션으로 대규모 토큰 소각이 가능해질 수 있습니다. 이는 또한 브릿지 컨트랙트에 대한 신뢰 가정과 중앙화 위험을 크게 증가시켜 전체 토큰 생태계에 단일 실패 지점(Single Point of Failure)을 생성합니다.

**권장 완화 방법:** `bridgeBurn` 기능을 구현하기 위해 `STBL_Token` 코드를 사용하는 것을 고려하십시오. 또는, 호출자로부터 컨트랙트로 토큰을 먼저 전송한 다음 소각하십시오.

**STBL:** 커밋 [a737746](https://github.com/USD-Pi-Protocol/contract/commit/a737746e3f136f6c83605228b81b23da23e27183)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 수익 분배에 대한 프론트러닝 공격으로 누적된 보상 탈취 가능성

**설명:** 현재 수익 분배는 다음과 같은 속성을 가집니다:

1. 주기적 분배 - 수익은 `yieldDuration` 초마다 한 번씩 계산되고 분배됩니다. [온체인 컨트랙트](https://etherscan.io/address/0x120F72491DA96CD42ca116AA11caEe206a4a1668#readContract)에 따르면 자산에 대한 `yieldDuration`은 2,592,000초(30일)입니다.
2. 비례 보상 - 모든 예금자는 예치 기간과 관계없이 지분 규모에 비례하여 수익을 받습니다.
3. 예측 가능한 가격 변동 - 외부 오라클 업데이트와 결합된 30일 분배 주기는 MEV 사냥꾼들에게 악용 기회를 제공합니다.

`distributeReward` 함수는 `yieldDuration` 초마다 보상이 분배되도록 허용하며, 보상은 지분 금액에 비례합니다:

```solidity
// STBL_LT1_YieldDistributor.sol

function distributeReward(uint256 reward) external {
    // @audit yieldDuration 초마다 한 번
    if (previousDistribution + AssetData.yieldDuration >= block.timestamp)
        revert STBL_Asset_YieldDurationNotReached(assetID, previousDistribution);

    // @audit 시간 가중치 없는 비례 분배
    rewardIndex += (reward * MULTIPLIER) / totalSupply;
    previousDistribution = block.timestamp;
}
```

`distributeYield` 함수는 업데이트된 오라클 가격을 사용하여 가격 차이를 계산합니다:

```solidity
  function distributeYield() external {
        AssetDefinition memory AssetData = registry.fetchAssetData(assetID);
        uint256 differentialUSD = iCalculatePriceDifferentiation(); //@audit 최신 가격을 기반으로 USD 가치 계산
         if (differentialUSD > 0) {
               iSTBL_LT1_AssetYieldDistributor(AssetData.rewardDistributor)
                .distributeReward(
                    yieldAssetValue.convertFrom18Decimals(
                        DecimalConverter.getTokenDecimals(AssetData.token)
                    )
                );
       }
}
```

위의 설계는 다음과 같은 공격 벡터를 허용합니다 (단순화를 위해 수수료 10% 가정, Alice가 유일한 예금자라고 가정):

- Alice가 $100,000를 예치하고 지분을 나타내는 NFT를 받습니다.
- 수익 기간(30일) 동안 가격 상승분이 누적됩니다 (예: OUSG 가격 상승 또는 USDY 복리로 $10,000).
- Alice는 ~$9,000의 수익을 기대합니다 (10% 프로토콜 수수료 후).
- Bob은 `previousDistribution + yieldDuration`을 모니터링하여 정확한 분배 기간을 식별합니다 (Bob은 오라클 업데이트도 모니터링하며 가격 차이가 수익에 큰 상승 여력을 준다는 것을 알고 있습니다).
- Bob은 `distributeYield`가 호출되기 직전에 $10,000,000를 예치합니다.
- 분배는 비례적으로 이루어집니다. Alice가 30일 내내 스테이킹했음에도 불구하고 Bob이 보상의 대부분을 가져갑니다.

**파급력:** 장기 예금자들은 수익 기간 동안 자본을 잠그는 것을 감수하는 기회주의적인 고래들에게 누적된 보상을 체계적으로 잃을 수 있습니다.

**개념 증명 (Proof of Concept):** `STBL_Test.t.sol`에 다음 테스트를 추가하십시오:

```solidity
    /**
     * @notice 수익 프론트러닝 공격 시연
     * @dev 고래가 최소한의 스테이킹 기간에도 불구하고 수익 보상의 대부분을 차지하는 방법을 보여줍니다.
     */
    function test_YieldMEVAttack() public {
        // 테스트 금액
        address alice = address(0x1001); // 장기 예금자
        address bob = address(0x1002);   // MEV 공격자
        uint256 aliceDeposit = 100000e18; // $100,000 상당
        uint256 bobDeposit = 10000000e18; // $10,000,000 (고래 예금)

        // 1단계: Alice가 일찍 예금하고 전체 기간 동안 보유
        console.log("--- Step 1: Alice (Long-term depositor) deposits $100k ---");

        // LT1 컨트랙트 가져오기 (자산 ID 2 사용)
        STBL_LT1_Issuer issuerContract = STBL_LT1_Issuer(getAssetIssuer(2));
        STBL_LT1_Vault vaultContract = STBL_LT1_Vault(getAssetVault(2));
        STBL_LT1_YieldDistributor yieldDistributorContract = STBL_LT1_YieldDistributor(getAssetYieldDistributor(2));
        STBL_LT1_TestOracle oracleContract = STBL_LT1_TestOracle(getAssetOracle(2));
        STBL_LT1_TestToken token = STBL_LT1_TestToken(getAssetToken(2));

        // Alice에게 토큰 민팅 및 승인
        vm.startPrank(admin);
        token.mintVal(alice, aliceDeposit);
        vm.stopPrank();

        vm.startPrank(alice);
        token.approve(address(vaultContract), aliceDeposit);
        uint256 aliceNftId = issuerContract.deposit(aliceDeposit);
        vm.stopPrank();

        console.log("Alice's NFT ID:", aliceNftId);
        console.log("Alice's deposit amount:", aliceDeposit);

        // 2단계: 거의 전체 수익 기간 동안 대기하고 상당한 수익 생성
        console.log("--- Step 2: Wait 29 days, generate significant yield ---");

        // 자산 설정에서 현재 수익 기간 가져오기
        AssetDefinition memory assetData = registry.fetchAssetData(2);
        uint256 yieldDuration = assetData.yieldDuration;
        console.log("Yield duration (seconds):", yieldDuration);

        // 대부분의 기간 빨리 감기 (Bob을 위해 작은 창 남겨둠)
        uint256 timeToWait = yieldDuration - 86400; // 수익 분배 가능 1일 전
        vm.warp(block.timestamp + timeToWait);
        console.log("Time advanced by:", timeToWait, "seconds (~29 days)");

        // 상당한 가격 상승 시뮬레이션
        uint256 initialPrice = oracleContract.fetchPrice();
        console.log("Initial oracle price:", initialPrice);

        // 10% 가격 상승 생성 (1000 * 0.01 = 10)
        for(uint256 i = 0; i < 1000; i++) {
            oracleContract.setPrice();
        }

        uint256 appreciatedPrice = oracleContract.fetchPrice();
        console.log("Final oracle price:", appreciatedPrice);
        console.log("Price appreciation:", ((appreciatedPrice - initialPrice) * 100) / initialPrice, "%");

        // 3단계: Bob이 모니터링하다가 수익 분배 기간 직전에 예금
        console.log("--- Step 3: Bob monitors yield distribution window ---");

        // Bob은 수익 분배가 가능할 때까지 대기
        vm.warp(block.timestamp + 86401); // 이제 정확히 yieldDuration
        console.log("Advanced to yield distribution window");

        // Bob이 분배 직전에 고래 금액 예금
        vm.startPrank(admin);
        token.mintVal(bob, bobDeposit);
        vm.stopPrank();

        vm.startPrank(bob);
        token.approve(address(vaultContract), bobDeposit);
        uint256 bobNftId = issuerContract.deposit(bobDeposit);
        vm.stopPrank();

        console.log("Bob's NFT ID:", bobNftId);
        console.log("Bob deposits 100x more capital than Alice:", bobDeposit);

        // 4단계: 수익 분배 전 지분 분포 분석
        console.log("--- Step 4: Analyzing stake distribution ---");

        (uint256 aliceStake,,,) = yieldDistributorContract.stakingData(aliceNftId);
        (uint256 bobStake,,,) = yieldDistributorContract.stakingData(bobNftId);
        uint256 totalStake = yieldDistributorContract.totalSupply();

        console.log("Alice stake:", aliceStake);
        console.log("Bob stake:", bobStake);
        console.log("Total stake:", totalStake);

        uint256 aliceSharePct = (aliceStake * 100) / totalStake;
        uint256 bobSharePct = (bobStake * 100) / totalStake;
        console.log("Alice will receive:", aliceSharePct, "% of yield");
        console.log("Bob will receive:", bobSharePct, "% of yield");

        // 5단계: 수익 분배 트리거
        console.log("--- Phase 5: Yield Distribution ---");

        uint256 expectedYield = vaultContract.CalculatePriceDifferentiation();
        console.log("Expected yield amount:", expectedYield);

        if (expectedYield > 0) {
            // 재무부(Treasury)가 수익 분배
            vm.prank(treasury);
            vaultContract.distributeYield();

            console.log("Yield distributed");
        }

        // 6단계: 불공정한 결과 분석
        console.log("--- Step 6: Attack Results ---");

        uint256 aliceRewards = yieldDistributorContract.calculateRewardsEarned(aliceNftId);
        uint256 bobRewards = yieldDistributorContract.calculateRewardsEarned(bobNftId);
        uint256 totalRewards = aliceRewards + bobRewards;

        console.log("Alice earned rewards:", aliceRewards);
        console.log("Bob earned rewards:", bobRewards);

        if (totalRewards > 0) {
            uint256 alicePct = (aliceRewards * 100) / totalRewards;
            uint256 bobPct = (bobRewards * 100) / totalRewards;

            console.log("UNFAIR DISTRIBUTION RESULTS:");
            console.log("Alice (30-day holder):", alicePct, "% of rewards");
            console.log("Bob (last-minute attacker):", bobPct, "% of rewards");

            // 공격 성공 확인
            assertTrue(bobRewards > aliceRewards, "MEV attack failed - Bob should earn more");
        }
```

**권장 완화 방법:** 수익을 더 자주 분배하는 것을 고려하십시오. 빈도가 높을수록 MEV 가능성은 낮아집니다. 이러한 공격에 대한 경제적 인센티브를 크게 완화하기 위해 최소한 주간 빈도를 권장합니다.

**STBL:** 인지함. 이는 구성을 통해 처리할 것이며 수익 기간에 대한 구성을 설정할 때 이를 염두에 둘 것입니다.

**Cyfrin:** 인지함.

### 자산 토큰의 네거티브 리베이스(negative rebasing) 발생 시 `withdrawERC20`에서의 산술 언더플로우

**설명:** 볼트의 회계는 기본 담보의 가격이 영구적으로 상승할 것이라는 전제를 기반으로 합니다. 기본 담보가 가장 위험이 낮지만, 특히 만기 전에 채권이 매각될 경우 자본은 여전히 위험에 노출될 수 있습니다.
예를 들어, ONDO가 기본 채권을 시장 가격에 매각해야 하고 채권 가격이 매입 시점보다 낮다면, 손실이 발생하여 `USDY` 또는 `oUSG` 가격에 네거티브 리베이스가 발생할 가능성이 큽니다.

볼트는 예치 시 가격을 사용하여 예금을 추적하지만, 출금 시에는 현재 시장 가격을 사용하여 계산합니다. 가격이 하락하면 출금 계산 시 볼트의 회계 시스템이 가용하다고 추적한 것보다 더 많은 토큰이 필요하게 됩니다.

네거티브 리베이스(동일한 USD 금액에 대해 더 많은 자산 단위 필요)의 결과는 출금 흐름에 영향을 미칩니다. [`VaultData.assetDepositNet`에서 `withdrawAssetValue (및 수수료)`를 뺄 때](https://github.com/USD-Pi-Protocol/contract/blob/feat/upgradeable/contracts/Assets/LT1/STBL_LT1_Vault.sol#L221-L222), 할인될 자산 단위의 양이 `VaultData.assetDepositNet`의 자산 단위 양보다 커지기 때문에 해당 연산에서 언더플로우가 발생합니다.

```solidity
    function withdrawERC20(
        address _to,
        YLD_Metadata memory MetaData
    ) external isValidIssuer {
        AssetDefinition memory AssetData = registry.fetchAssetData(assetID);

        // 출금 자산 가치 계산
      uint256 withdrawAssetValue = iSTBL_LT1_AssetOracle(AssetData.oracle)
            .fetchInversePrice(
                ((MetaData.stableValueNet + MetaData.haircutAmount) -
                    MetaData.withdrawfeeAmount)
            ); //@audit 최신 가격을 가져옴 -> 가격이 마이너스이면 withdrawAssetValue가 assetDepositNet보다 클 수 있음

        ...


      VaultData.assetDepositNet -= (withdrawAssetValue +
            withdrawFeeAssetValue); //@audit 위 상황이 발생하면 assetDepositNet이 언더플로우됨

        ...
    }
```

**파급력:** 자산 가격의 네거티브 리베이스 발생 시 사용자 출금에 대한 서비스 거부(DoS)가 발생할 수 있습니다.

**개념 증명 (Proof of Concept):** 다음 함수를 `STBL_PT1_TestOracle`에 추가하십시오:

```solidity
    function decreasePriceByPercentage(uint256 basisPoints) external {
        require(basisPoints <= 9999, "Cannot decrease more than 99.99%");
        price = (price * (10000 - basisPoints)) / 10000;
    }
```

그런 다음 `STBL_Test.sol`에 다음 테스트를 추가하십시오:

```solidity
   /// @notice 테스트: 예금 -> 큰 가격 하락 -> 수익 분배 시도 -> 출금
    /// @dev 이는 부정적인 가격 변동 하에서의 회계 무결성과 잠재적인 언더플로우 위험을 테스트합니다.
    function test_DepositWithdraw1PctPriceDecrease() public {
        console.log("=== 1% PRICE DECREASE TEST ===");

        // 컨트랙트 가져오기
        STBL_PT1_TestToken assetToken = STBL_PT1_TestToken(getAssetToken(1));
        STBL_PT1_Issuer stblPT1Issuer = STBL_PT1_Issuer(getAssetIssuer(1));
        STBL_PT1_Vault stblPT1Vault = STBL_PT1_Vault(getAssetVault(1));
        STBL_PT1_TestOracle stblPT1TestOracle = STBL_PT1_TestOracle(getAssetOracle(1));
        STBL_PT1_YieldDistributor yieldDist = STBL_PT1_YieldDistributor(getAssetYieldDistributor(1));

        // === 0단계: 볼트 유동성 확보 ===
        // 가격 하락 후 잔액 부족으로 인한 출금 실패를 방지하기 위해 볼트에 상당한 추가 토큰을 직접 추가
        console.log("--- PHASE 0: ENSURE VAULT LIQUIDITY ---");
        uint256 extraLiquidity = 100000e6; // 100,000 추가 토큰
        vm.startPrank(admin);
        assetToken.mintVal(address(stblPT1Vault), extraLiquidity);
        vm.stopPrank();

        // 재무부(Treasury) 설정
        vm.startPrank(admin);
        registry.setTreasury(admin);
        vm.stopPrank();

        // === 1단계: 초기 예금 ===
        console.log("--- PHASE 1: DEPOSIT ---");
        uint256 depositAmount = 10000e6;

        vm.startPrank(user1);
        assetToken.approve(address(stblPT1Vault), depositAmount);
        uint256 nftId = stblPT1Issuer.deposit(depositAmount);
        vm.stopPrank();

        // 초기 상태 로그
        console.log("Initial deposit NFT ID:", nftId);
        console.log("Initial oracle price:", stblPT1TestOracle.fetchPrice());
        console.log("Initial vault token balance:", assetToken.balanceOf(address(stblPT1Vault)));
        console.log("User USST balance:", usst.balanceOf(user1));

        // 초기 메타데이터 및 볼트 상태 가져오기
        YLD_Metadata memory initialMetadata = yld.getNFTData(nftId);
        VaultStruct memory initialVaultData = stblPT1Vault.fetchVaultData();
        console.log("Asset value:", initialMetadata.assetValue);
        console.log("Stable value net:", initialMetadata.stableValueNet);
        console.log("Initial vault asset deposit net:", initialVaultData.assetDepositNet);
        console.log("Initial vault deposit value USD:", initialVaultData.depositValueUSD);

        // === 2단계: 수익 자격을 위해 시간 전진 ===
        console.log("\n--- PHASE 2: ADVANCE TIME FOR YIELD ELIGIBILITY ---");
        vm.warp(block.timestamp + initialMetadata.Fees.yieldDuration + 1);
        console.log("Advanced time by:", initialMetadata.Fees.yieldDuration + 1, "seconds");

        // === 3단계: 큰 가격 하락 ===
        console.log("\n--- PHASE 3: LARGE PRICE DECREASE ---");

        uint256 initialPrice = stblPT1TestOracle.fetchPrice();
        console.log("Price before decrease:", initialPrice);

        // 10% 가격 하락 시뮬레이션
        stblPT1TestOracle.decreasePriceByPercentage(100);

        uint256 newPrice = stblPT1TestOracle.fetchPrice();
        console.log("Price after decrease:", newPrice);
        console.log("Percentage change:", stblPT1TestOracle.calculatePercentageChange(initialPrice, newPrice));

        // === 4단계: 가격 하락 후 수익 계산 확인 ===
        console.log("\n--- PHASE 4: YIELD CALCULATION AFTER PRICE DECREASE ---");

        // 잠재적 수익 분배 전 볼트 상태 확인
        VaultStruct memory preYieldVaultData = stblPT1Vault.fetchVaultData();
        console.log("Vault asset deposit net (pre-yield attempt):", preYieldVaultData.assetDepositNet);
        console.log("Vault deposit value USD (pre-yield attempt):", preYieldVaultData.depositValueUSD);

        // 가격 차이 계산 - 가격 하락 시 0이어야 함
        uint256 priceDifferential = stblPT1Vault.CalculatePriceDifferentiation();
        console.log("Price differential calculated:", priceDifferential);

        // 가격 하락 시 수익이 분배되지 않음을 검증
        assertEq(priceDifferential, 0, "Price differential should be 0 when price decreases");

        // 수익 분배 시도 - 아무 작업도 수행되지 않아야 함
        vm.startPrank(admin);
        stblPT1Vault.distributeYield();
        vm.stopPrank();

        // 수익 분배 시도 후 볼트 상태 확인
        VaultStruct memory postYieldVaultData = stblPT1Vault.fetchVaultData();
        console.log("Vault asset deposit net (post-yield attempt):", postYieldVaultData.assetDepositNet);
        console.log("Vault yield fees collected:", postYieldVaultData.yieldFees);

        // 수익 분배 중 변경 사항이 발생하지 않았음을 검증
        assertEq(postYieldVaultData.assetDepositNet, preYieldVaultData.assetDepositNet, "assetDepositNet should not change");
        assertEq(postYieldVaultData.yieldFees, preYieldVaultData.yieldFees, "yieldFees should not change");

        // === 5단계: 하락한 가격 조건 하에서의 출금 ===
        console.log("\n--- PHASE 5: WITHDRAWAL UNDER DECREASED PRICE ---");

        // 자산 가격이 하락했을 때 사용자가 여전히 출금할 수 있는지 확인
        uint256 usstBalance = usst.balanceOf(user1);
        console.log("USST balance before withdrawal:", usstBalance);

        vm.startPrank(user1);
        usst.approve(address(usst), usstBalance);
        yld.setApprovalForAll(address(yld), true);

        uint256 preWithdrawVaultBalance = assetToken.balanceOf(address(vault));
        uint256 preWithdrawUserBalance = assetToken.balanceOf(user1);
        console.log("Vault balance before withdrawal:", preWithdrawVaultBalance);
        console.log("User balance before withdrawal:", preWithdrawUserBalance);

        // 출금 시도
        vm.expectRevert();
        stblPT1Issuer.withdraw(nftId, user1);
        vm.stopPrank();

    }
```

**권장 완화 방법:** `withdrawERC20`을 수정하여 가용 회계 잔액으로 출금을 제한함으로써 언더플로우를 방지하는 것을 고려하십시오:

```solidity
function withdrawERC20(address _to, YLD_Metadata memory MetaData) external isValidIssuer {
    AssetDefinition memory AssetData = registry.fetchAssetData(assetID);

    uint256 withdrawAssetValue = iSTBL_PT1_AssetOracle(AssetData.oracle)
        .fetchInversePrice(
            ((MetaData.stableValueNet + MetaData.haircutAmount) - MetaData.withdrawfeeAmount)
        );

    uint256 withdrawFeeAssetValue = iSTBL_PT1_AssetOracle(AssetData.oracle)
        .fetchInversePrice(MetaData.withdrawfeeAmount);

    uint256 totalWithdrawal = withdrawAssetValue + withdrawFeeAssetValue;

    // @audit 가용 잔액으로 출금 제한
    if (VaultData.assetDepositNet < totalWithdrawal) {
        uint256 availableWithdrawal = VaultData.assetDepositNet > withdrawFeeAssetValue
            ? VaultData.assetDepositNet - withdrawFeeAssetValue
            : 0;

        withdrawAssetValue = availableWithdrawal;
        VaultData.assetDepositNet = 0; //@audit 이를 강제로 0으로 설정

    } else {
        VaultData.assetDepositNet -= totalWithdrawal;
    }

    // 함수의 나머지 부분 계속...
}
```

**STBL:** 커밋 [c540943](https://github.com/USD-Pi-Protocol/contract/commit/c54094363b196b534c9c36d563851dff31fe2975)에서 수정되었습니다.

**Cyfrin:** 확인됨. 네거티브 리베이스가 발생할 가능성은 낮지만, 현재 수정 사항은 볼트가 지급 불능 상태가 되면 되돌려(revert)진다는 점에 주목합니다. 그러나 이러한 시나리오에서 이 수정 사항은 초기 사용자가 (지급 불능이 발생하기 전에) 출금할 수 있도록 허용하는 반면, 나중에 출금을 시도하는 사용자에게는 서비스 거부를 초래합니다.

\clearpage
## 낮은 위험 (Low Risk)

### `STBL_Register::setupAsset`에서의 제로 주소 확인 누락

**설명:** `STBL_Register::setupAsset`에서 여러 자산 주소가 제로 주소 확인 없이 초기화됩니다:

- `_contractAddr`
- `_issuanceAddr`
- `_distAddr`
- `_vaultAddr`
- `_oracle`

특히, `setOracle`과 같은 각각의 설정자(setter) 함수를 통해 설정될 때는 이러한 유효성 검사가 있습니다. 또한 자산은 한 번만 설정할 수 있다는 점에 유의해야 합니다.

**권장 완화 방법:** `setupAsset`에 제로 주소 확인을 추가하는 것을 고려하십시오.

**STBL:** 커밋 [a737746](https://github.com/USD-Pi-Protocol/contract/commit/a737746e3f136f6c83605228b81b23da23e27183)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `STBL_Register::setupAsset`에서의 불충분한 수수료 검증으로 인한 언더플로우 가능성

**설명:** `STBL_Register::setupAsset` 및 `STBL_Register::setFees` 함수는 개별 수수료를 검증하지만 누적 영향은 무시합니다. `depositfeeAmount`, `haircutAmount` 및 `insurancefeeAmount`가 총 스테이블 가치(gross stable value)에 대해 계산되는 `STBL_MetadataLib.calculateDepositFees`의 다음 로직을 고려하십시오:

```solidity
function calculateDepositFees(YLD_Metadata memory data) internal pure returns (YLD_Metadata memory) {
    // 모든 예금 시 수수료는 동일한 기준(stableValueGross)으로 계산됨
    data.depositfeeAmount = (data.stableValueGross * data.Fees.depositFee) / 10000;
    data.haircutAmount = (data.stableValueGross * data.Fees.hairCut) / 10000;
    data.insurancefeeAmount = (data.stableValueGross * data.Fees.insuranceFee) / 10000;
    data.withdrawfeeAmount = (data.stableValueGross * data.Fees.withdrawFee) / 10000;
    return data; //@audit depositfeeAmount, haircutAmount 및 insurancefeeAmount는 stableValueGross에 대해 계산됨
}
```

`STBL_LT1_Issuer` 및 `STBL_PT1_Issuer` 컨트랙트 모두 순 스테이블 가치(net stable value)를 다음과 같이 계산합니다:

```solidity
   MetaData.stableValueNet = (MetaData.stableValueGross -
            (MetaData.depositfeeAmount +
                MetaData.haircutAmount +
                MetaData.insurancefeeAmount));
```

이는 모든 수수료의 합계가 베이시스 포인트(basis points) 기준으로 10000을 초과하면, 메타데이터 로직이 순 스테이블 가치를 계산할 때 항상 되돌려짐(revert)을 의미합니다.

**파급력:** 예금 수수료, 헤어컷 및 보험 수수료의 합계가 10000을 초과하는 수수료 조합은 순 스테이블 가치 계산에서 언더플로우를 유발할 수 있습니다.

**권장 완화 방법:** `STBL_Register::setupAsset` 및 `STBL_Register::setFees` 함수에 누적 확인을 도입하는 것을 고려하십시오.

**STBL:** 커밋 [1adc1f2](https://github.com/USD-Pi-Protocol/contract/commit/1adc1f2d05dcbcee89826ab9b7d625642c0834bd)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `STBL_Register::setupAsset`에서의 불충분한 기간 검증으로 인한 사용자 출금 잠금 가능성

**설명:** `STBL_Register::setupAsset` 및 `STBL_Register::setDurations`에서의 기간 검증 부재는 `yieldDuration > duration`인 자산 구성을 허용하여, 사용자 자금을 잠그는 수학적으로 불가능한 출금 조건을 생성합니다.

출금 로직은 사용자가 최소 `yieldDuration` 동안 대기하면서 동시에 `duration`이 만료되기 전에 출금해야 합니다. `yieldDuration > duration`일 때, 사용자가 출금할 수 있는 유효한 시간 창이 존재하지 않습니다.

`STBL_LT1_Issuer`의 `iWithdraw` 함수에는 다음과 같은 확인 사항이 있습니다:

```solidity
function iWithdraw(uint256 _tokenID, address _sender) internal isSetupDone {
    YLD_Metadata memory MetaData = iSTBL_YLD(registry.fetchYLDToken()).getNFTData(_tokenID);
    // 코드..

    // duration이 지난 후에는 사용자가 출금할 수 없도록 보장
        if ((MetaData.depositBlock + MetaData.Fees.duration) < block.timestamp)
            revert STBL_Asset_WithdrawDurationNotReached(assetID, _tokenID);

        // 사용자가 자산을 출금하려면 yield duration을 기다려야 함을 보장
        if (
            (MetaData.depositBlock + MetaData.Fees.yieldDuration) >
            block.timestamp
        ) revert STBL_Asset_YieldDurationNotReached(assetID, _tokenID);

    // @audit 두 조건이 모두 거짓일 때만 출금 진행...
   //@audit yieldDuration > duration일 때, 사용자를 위한 출금 창이 존재하지 않음
}
```

**파급력:** 잘못 구성된 자산 기간은 사용자 출금을 막을 수 있습니다.

**권장 완화 방법:** `setupAsset` 및 `setDurations`에 기간 관계 검증을 추가하는 것을 고려하십시오.

**STBL:** 커밋 [c540943](https://github.com/USD-Pi-Protocol/contract/commit/c54094363b196b534c9c36d563851dff31fe2975)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 활성 예금이 있는 상태에서 `STBL_Register::disableAsset`을 통한 자산 비활성화 시 사용자 출금 DOS 발생 가능

**설명:** `STBL_Register::disableAsset`은 자산을 비활성화하기 전에 활성 예금을 확인하지 않아, 사용자가 예치된 자금을 출금할 수 없는 서비스 거부(DOS) 조건을 생성합니다.

`STBL_Register::disableAsset`은 자산이 현재 활성화되어 있는지만 확인하고 활성 사용자 예금이 있는지 여부는 무시합니다:

```solidity
// STBL_Register.sol - disableAsset()
function disableAsset(uint256 _id) external onlyRole(REGISTER_ROLE) {
    if (assetData[_id].status != AssetStatus.ENABLED)
        revert STBL_AssetNotActive();
    assetData[_id].status = AssetStatus.DISABLED;  // @audit 활성 예금을 확인하지 않음
    emit AssetStateUpdateEvent(_id, true);
}
```

그러나 출금 흐름이 성공적으로 완료되려면 자산이 ENABLED 상태여야 합니다. `STBL_Core::exit` 함수는 자산 상태를 확인하는 `isValidIssuer` 제어자(modifier)를 사용합니다:

```solidity
// STBL_Core.sol
modifier isValidIssuer(uint256 _assetID) {
    AssetDefinition memory AssetData = registry.fetchAssetData(_assetID);
    if (!AssetData.isIssuer(_msgSender())) revert STBL_UnauthorizedIssuer();
    if (!AssetData.isActive()) revert STBL_AssetDisabled(_assetID);  // @audit 비활성화된 자산에 대한 exit 차단
    _;
}

function exit(uint256 _assetID, address _from, uint256 _tokenID, uint256 _value)
    external isValidIssuer(_assetID) {  // @audit 제어자가 비활성화된 자산에 대한 exit 방지
    // ... 토큰 소각 및 예금 감소
}
```

**파급력:** 활성 예금이 있는 비활성화된 자산은 사용자 출금을 막습니다.

**권장 완화 방법:** 활성 예금이 있는 자산의 비활성화를 방지하는 것을 고려하십시오.

**STBL:** 인지함.

**Cyfrin:** 인지함.

### `STBL_Register::addAsset`이 비어 있지 않은 자산 이름을 확인하지 않음

**설명:** `STBL_Register::setupAsset`은 이름이 비어 있는 자산 생성을 허용합니다. 그러나 코드의 다른 곳인 `STBL_AssetDefinitionLib::isValid` 함수는 이름 필드가 비어 있는 자산을 유효하지 않은 것으로 표시합니다. 이는 자산이 프로토콜에서 성공적으로 생성되고 활성화될 수 있지만 `isValid` 함수를 통한 유효성 검사에서는 실패하는 불일치를 생성합니다.

```solidity
// STBL_Register.sol - addAsset function
function addAsset(
    string memory _name,    // @audit 이름 필드에 대한 검증 없음
    string memory _desc,
    uint8 _type,
    bool _aggType
) external onlyRole(REGISTER_ROLE) returns (uint256) {
    unchecked {
        assetCtr += 1;
    }
    assetData[assetCtr].id = assetCtr;
    assetData[assetCtr].name = _name;        // @audit 빈 문자열일 수 있음
    assetData[assetCtr].description = _desc;
    assetData[assetCtr].contractType = _type;
    assetData[assetCtr].isAggreagated = _aggType;
    assetData[assetCtr].status = AssetStatus.INITIALIZED;

    emit AddAssetEvent(assetCtr, assetData[assetCtr]);
    return assetCtr;
}
```
`isValid` 함수는 이 자산을 유효하지 않은 것으로 표시합니다:

```solidity
// STBL_AssetDefinitionLib.sol
function isValid(AssetDefinition memory asset) internal pure returns (bool) {
    return
        asset.id != 0 &&
        bytes(asset.name).length > 0 &&  // ✅ 비어 있지 않은 이름 요구
        asset.token != address(0) &&
        asset.issuer != address(0) &&
        asset.rewardDistributor != address(0) &&
        asset.vault != address(0);
}
```

`isValid()` 함수는 현재 메인 프로토콜 컨트랙트에서는 사용되지 않지만 테스트 파일에서 임포트되는데, 이는 검증을 위해 의도되었으나 프로토콜 흐름에 제대로 통합되지 않았음을 시사합니다.

**파급력:** 일관성 없는 검증 로직 및 잠재적인 오프체인 통합 문제.

**권장 완화 방법:** `addAsset`에 이름 검증을 추가하거나 `isValid`에서 이름 요구 사항을 제거하는 것을 고려하십시오.

**STBL:** 커밋 [4a187a5](https://github.com/USD-Pi-Protocol/contract/commit/4a187a54fa0f5e268f4fb43305e88e37ed512c08)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 출금 중 기간 만료 경계에서의 경쟁 상태 (Race Condition)

**설명:** `STBL_LT1_Issuer` 컨트랙트는 `block.timestamp == depositBlock + duration`일 때 경쟁 상태를 생성하는 출금 유효성 검사에서 일관성 없는 경계 조건 로직을 포함하고 있습니다.

`iWithdraw()` 함수(사용자 출금)에서:

```solidity
if ((MetaData.depositBlock + MetaData.Fees.duration) < block.timestamp)
    revert STBL_Asset_WithdrawDurationNotReached(assetID, _tokenID);
```

`withdrawExpired()` 함수(프로토콜 출금)에서:
```solidity
if ((MetaData.depositBlock + MetaData.Fees.duration) > block.timestamp)
    revert STBL_Asset_WithdrawDurationNotReached(assetID, _tokenID);
```

`depositBlock + duration == block.timestamp`일 때, 사용자와 프로토콜 모두 출금할 수 있습니다. 이는 먼저 채굴되는 트랜잭션이 성공하고 다른 트랜잭션은 되돌려지는(revert) 경쟁 상태를 생성합니다.

**파급력:** 만료 시점에 프로토콜의 개입으로 인해 사용자의 유효한 출금 트랜잭션이 예기치 않게 실패할 수 있습니다.

**개념 증명 (Proof of Concept):** 추후 결정 (TBD).

**권장 완화 방법:** 만료 타임스탬프에서 사용자에게 명확한 우선순위를 부여하도록 경계 조건을 수정하는 것을 고려하십시오.

```solidity
// withdrawExpired()
if ((MetaData.depositBlock + MetaData.Fees.duration) >= block.timestamp) //@audit >를 >=로 수정
    revert STBL_Asset_WithdrawDurationNotReached(assetID, _tokenID);
```
**STBL:** 수정됨.

**Cyfrin:** 확인됨.

### NFT가 비활성화된 경우 재무부(Treasury)가 만료된 자산을 출금할 수 없음

**설명:** `STBL_LT1_Issuer::withdrawExpired` 함수는 비활성화된 NFT에 대해 보상 청구를 시도하지만, `STBL_LT1_YieldDistributor::claim` 함수는 비활성화된 NFT에 대해 호출될 때 명시적으로 되돌립니다(revert).

`STBL_LT1_Issuer.withdrawExpired`에서:

```solidity
// NFT가 비활성화된 경우 수익 청구는 수행되지 않음
if (MetaData.isDisabled) {
    iSTBL_LT1_AssetYieldDistributor(AssetData.rewardDistributor).claim(_tokenID);
}
```
`STBL_LT1_YieldDistributor.claim`에서:

```solidity
if (MetaData.isDisabled) revert STBL_YLDDisabled(id);
```

**파급력:** `withdrawExpired`는 메타데이터가 비활성화되었을 때 서비스 거부를 유발합니다.

**권장 완화 방법:** 발행자(issuer)가 비활성화된 NFT에 대해 호출할 수 있도록 기존 `claim()` 함수를 수정하는 것을 고려하십시오:

```solidity
function claim(uint256 id) external returns (uint256) {
  // 현재 로직

   // @audit 발행자가 비활성화된 NFT에 대해 청구할 수 있도록 허용하되, 일반 사용자는 차단
    if (MetaData.isDisabled && msg.sender != AssetData.issuer) {
        revert STBL_YLDDisabled(id);
    }
}
```

**STBL:** 커밋 [1adc1f2](https://github.com/USD-Pi-Protocol/contract/commit/1adc1f2d05dcbcee89826ab9b7d625642c0834bd)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)

### `ContractType` 필드가 자산이 비 ERC20 토큰일 수 있음을 암시하여 오해를 불러일으킴

**설명:** `AssetDefinition` 구조체는 ERC20, ERC721 및 사용자 정의(Custom) 자산 유형에 대한 지원을 암시하는 `contractType` 필드를 포함하지만, 전체 코드베이스는 암시적으로 모든 자산이 ERC20 토큰이라고 가정합니다. 이는 오해의 소지가 있는 문서화와 잠재적인 통합 문제를 야기합니다.

`AssetDefinition` 구조체는 여러 옵션이 있는 `contractType` 필드를 정의합니다:

```solidity
// STBL_Structs.sol
struct AssetDefinition {
    // ...
    /** @notice 컨트랙트 유형 (ERC20 = 0, ERC721 = 1, Custom = 2) */
    uint8 contractType;
    // ...
}
```
그러나 코드베이스 전체에서 모든 자산 토큰 상호작용은 IERC20 인터페이스를 사용하며, 암시적으로 `contractType`이 ERC20이라고 가정합니다.

**파급력:** 자산 설정 시 오해의 소지가 있는 문서화 및 잠재적인 혼란.

**권장 완화 방법:** 코드베이스에서 암시적으로 ERC20 유형을 가정하는 자산 처리와 일치하지 않으므로 `contractType` 필드를 제거하는 것을 고려하십시오.

**STBL:** 인지함. erc 721 또는 erc1155 기반 부채 상품을 지원하는 프로토콜이 있을 수 있으므로 향후 사용을 위해 유지합니다.

**Cyfrin:** 인지함.

### 중요한 오라클 매개변수 변경에 대한 이벤트 방출 누락

**설명:** 오라클 컨트랙트(`STBL_PT1_Oracle` 및 `STBL_LT1_Oracle`)는 관리자 함수에 의해 중요한 매개변수가 수정될 때 이벤트를 방출하지 않습니다.

다음 관리자 함수들은 중요한 매개변수를 수정하지만 이벤트를 방출하지 않습니다:

- `setPriceDecimals`
- `setPriceThreshold`
- `enableOracle`
- `disableOracle`

**권장 완화 방법:** 위 함수들에 대한 이벤트 선언을 추가하는 것을 고려하십시오.

**STBL:** 커밋 [c540943](https://github.com/USD-Pi-Protocol/contract/commit/c54094363b196b534c9c36d563851dff31fe2975)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `Metadata.depositBlock`에 대한 변수 명명 불일치

**설명:** `Metadata.depositBlock`의 [인라인 문서](https://github.com/USD-Pi-Protocol/contract/blob/main/contracts/lib/STBL_Structs.sol#L26-L27)에 따르면 예금이 생성된 블록 번호를 저장하는 것을 암시하지만, 실제로는 이 변수가 [타임스탬프를 저장합니다.](https://github.com/USD-Pi-Protocol/contract/blob/main/contracts/Assets/LT1/STBL_LT1_Issuer.sol#L351)

**권장 완화 방법:** `Metadata.depositBlock` 변수의 이름을 `Metadata.depositTimestamp`와 같이 더 정확한 이름으로 변경하는 것을 고려하십시오.

**STBL:** 커밋 [c540943](https://github.com/USD-Pi-Protocol/contract/commit/c54094363b196b534c9c36d563851dff31fe2975)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `STBL_PT1_Vault::withdrawFees`에서의 CEI 패턴 위반

**설명:** `STBL_PT1_Vault::withdrawFees` 함수는 수수료를 0으로 재설정하기 전에 외부 토큰 전송을 수행함으로써 Checks-Effects-Interactions (CEI) 패턴을 위반합니다.

이 함수는 재무부로 `safeTransfer()` 호출을 실행한 후 수수료 카운터(`VaultData.depositFees`, `VaultData.withdrawFees` 등)를 0으로 재설정합니다. 이 함수에는 접근 제어가 없다는 점에 유의해야 합니다.

**권장 완화 방법:** 적절한 CEI 패턴을 구현하는 것을 고려하십시오. 이벤트를 방출하고, 카운터를 재설정한 다음 전송을 수행하십시오.

**STBL:** 인지함.

**Cyfrin:** 인지함.

### `STBL_LT1_Issuer::withdrawExpired` 함수의 오해의 소지가 있는 인라인 주석

**설명:** `STBL_LT1_Issuer::withdrawExpired` 함수는 실제 코드 로직 및 의도된 기능과 정면으로 모순되는 오해의 소지가 있는 인라인 주석을 포함하고 있습니다.

```solidity
function withdrawExpired(uint256 _tokenID) external isSetupDone {
    // ... 유효성 검사 ...


>    //ensures that protocol can't withdraw after duration has passed //@audit 부정확한 주석
    if ((MetaData.depositBlock + MetaData.Fees.duration) > block.timestamp)
        revert STBL_Asset_WithdrawDurationNotReached(assetID, _tokenID);

    // ... 출금 로직 ...
}
```

주석에는 "기간이 지난 후에는 프로토콜이 출금할 수 없도록 보장"이라고 되어 있습니다. 그러나 실제 코드 로직은 그 반대입니다.

또한, `iWithdraw` 함수의 revert도 오해의 소지가 있습니다:

```solidity
   if ((MetaData.depositBlock + MetaData.Fees.duration) < block.timestamp)
            revert STBL_Asset_WithdrawDurationNotReached(assetID, _tokenID);
  //@audit 출금 기간이 만료되었을 때 revert되어야 함
```

**권장 완화 방법:** 코드 로직을 정확하게 반영하도록 인라인 주석을 업데이트하는 것을 고려하십시오. 또한 `iWithdraw`의 `STBL_Asset_WithdrawDurationNotReached` 에러를 `STBL_Asset_WithdrawDurationExpired`로 교체하는 것을 고려하십시오.

**STBL:** 수정됨.

**Cyfrin:** 수정됨.

### 누구나 사용자를 대신하여 예금 및 출금 가능

**설명:** 발행자(Issuer) 컨트랙트는 누구나 제한 없이 모든 계정을 대신하여 예금 및 출금을 호출할 수 있도록 허용합니다.

이를 악용하여 더 해로운 공격을 수행할 방법은 발견되지 않았지만, 승인 없이 다른 사용자를 대신하여 작업할 수 있다는 것은 여전히 이상적이지 않습니다.

예금을 위해서는 기본 담보의 승인이 필요하지만, 지갑 UI를 통해 사용자가 무한 승인을 허용하는 것은 드문 일이 아닙니다. 따라서 사용자가 자산을 출금하거나 잔액에 해당 자산을 확보하는 즉시 누군가가 자산을 다시 예치할 수 있습니다.

출금의 경우, 사용자의 자산이 자산 소유자가 출금하려고 의도하기 전에 출금될 수 있습니다.

**권장 완화 방법:** 아무나 허용하는 대신 예금자/출금자를 대신하여 호출할 수 있는 `operators`를 화이트리스트에 추가하는 것을 고려하십시오.

**STBL**
인지함. 이는 다른 프로토콜의 통합을 용이하게 하기 위해 의도적으로 설계되었습니다.

**Cyfrin:** 인지함.

\clearpage
## 가스 최적화 (Gas Optimization)

### `STBL_PT1_YieldDistributor`에서 호출자가 `Issuer`인지 확인하기 위해 불필요한 `_msgSender()` 사용

**설명:** `STBL_PT1_YieldDistributor` 컨트랙트에서 `enableStaking()` 및 `disableStaking()` 함수는 `isIssuer()` 제어자를 사용하여 호출자가 YieldDistributor에 대해 구성된 `assetId`의 승인된 발행자인지 확인합니다.

`STBL_PT1_YieldDistributor.isIssuer()`는 내부 `_msgSender()`를 호출하며, 이는 `ERC2771ContextUpgradeable._msgSender()`를 호출하고 결국 `ContextUpgradeable._msgSender()`를 호출합니다.

`TrustedForwarder`가 `YieldDistributor`가 아니므로 `ERC2771ContextUpgradeable._msgSender()`를 거치지 않고 `msg.sender`를 사용하는 것이 더 최적입니다. 발행자는 `YieldDistributor`를 직접 호출합니다.

```solidity
    modifier isIssuer() {
        AssetDefinition memory AssetData = registry.fetchAssetData(assetID);
@>      if (!AssetData.isIssuer(_msgSender()))
            revert STBL_Asset_InvalidIssuer(assetID);
        _;
    }


    function _msgSender()
        internal
        view
        override(ERC2771ContextUpgradeable)
        returns (address)
    {
@>      return ERC2771ContextUpgradeable._msgSender();
    }

```

`distributeReward()`에도 동일하게 적용됩니다:
```solidity
    function distributeReward(uint256 reward) external {
        ...
@>      if (!AssetData.isVault(_msgSender()))
            revert STBL_Asset_InvalidVault(assetID);
        ...
        IERC20(AssetData.token).safeTransferFrom(
@>          _msgSender(),
            address(this),
            reward
        );
        ...
    }

```

**권장 완화 방법:** `_msgSender()`를 호출하는 대신 `msg.sender`를 사용하는 것을 고려하십시오.

**STBL:** 커밋 [c540943](https://github.com/USD-Pi-Protocol/contract/commit/c54094363b196b534c9c36d563851dff31fe2975)에서 수정되었습니다.

**Cyfrin:** 확인됨.
