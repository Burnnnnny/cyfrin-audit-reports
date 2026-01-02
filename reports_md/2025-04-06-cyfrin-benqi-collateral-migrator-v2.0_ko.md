**Lead Auditors**

[Immeas](https://twitter.com/0ximmeas)

---

# Findings
## Low Risk


### 기존 자금이 플래시 론 수수료에 사용되면 `CollateralMigrator`가 revert됨 (`CollateralMigrator` reverts when pre-existing funds are used for flash loan fees)

**설명:** 두 시장 간에 담보를 마이그레이션하는 흐름은 다음과 같습니다:

1. 최종 사용자가 필요한 스왑 매개변수와 함께 `CollateralMigrator::migrateCollateral`을 호출합니다.
2. `CollateralMigrator`는 Trader Joe 유동성 쌍에서 플래시 론을 받습니다.
3. 플래시 론을 사용하여 `CollateralMigrator`는 대상 시장에 포지션을 엽니다.
4. 이제 포지션이 초과 담보 상태가 되었으므로 원래 소스 시장 포지션을 닫을 수 있습니다.
5. 소스 토큰은 대상 토큰으로 스왑됩니다.
6. 대상 토큰은 관련 수수료와 함께 플래시 론을 상환하는 데 사용됩니다.
7. 남은 대상 토큰은 사용자를 대신하여 대상 시장에 예치됩니다.

`CollateralMigrator` 계약은 실행 후 자금을 보유할 의도가 없습니다. 그러나 계약에 토큰이 남아 있는 경우(예: 이전의 실패한 거래로 인해), 사용자는 이를 사용하여 플래시 론 수수료를 부분적으로 충당하려고 시도할 수 있습니다.

이러한 접근 방식은 [`CollateralMigrator::LBFlashLoanCallback`](https://github.com/woof-software/benqi-collateral-migrator/blob/d91ce3dbf56d3939d54520f6f41afeebbbfc069a/contracts/CollateralMigrator.sol#L488-L511)에서 revert를 유발합니다:

```solidity
uint256 contractBalanceBF = IERC20(trgMarketConfig.flashData.baseToken).balanceOf(address(this));

// 소스 시장에서 대상 시장으로 기본 자산 스왑
_swap(
    ...
);

// 유동성 북(liquidity book) 쌍에 플래시 론 금액 반환
uint256 flashAmountWithFee = flashAmount +
    (trgMarketConfig.flashData.isTokenX ? totalFees.decodeX() : totalFees.decodeY());

// IEIP20NonStandard(flashData.baseToken).transfer(address(flashData.liquidityBookPair), flashAmountWithFee);
IERC20(trgMarketConfig.flashData.baseToken).safeTransfer(
    address(trgMarketConfig.flashData.liquidityBookPair),
    flashAmountWithFee
);

// 시장에 남은 기본 자산 반환
uint256 contractBalanceAF = IERC20(trgMarketConfig.flashData.baseToken).balanceOf(address(this));
// @audit 기존 자금이 수수료 지불에 사용된 경우 여기서 언더플로우 발생
uint256 remainingAmount = contractBalanceAF - contractBalanceAF - contractBalanceBF;
```

기존 자금이 플래시 론 수수료를 지불하는 데 사용되면 `contractBalanceAF`가 `contractBalanceBF`보다 작아지므로 `contractBalanceAF - contractBalanceBF` 라인에서 언더플로우가 발생합니다.

**영향:** 계약에 있는 기존 토큰을 사용하여 플래시 론 수수료를 충당하면 언더플로우로 인해 트랜잭션이 revert됩니다.

**개념 증명 (Proof of Concept):** `CollateralMigrator.test.ts`의 `context("Migration functions")` 아래에 다음 테스트를 추가하세요:
```javascript
it("Migrate will not work when existing balance is used for fees", async function () {
    const { user, collateralMigrator, swapRouter, qiUSDT, qiDAI, USDT, DAI, usdtDaiLBPair } = await loadFixture(setupEnv);

    const fromMarket = qiUSDT.address;
    const toMarket = qiDAI.address;

    const fromAsset = USDT.address;
    const toAsset = DAI.address;
    const migrationAmount = parseEther("200");

    await usdtDaiLBPair.connect(user).setFeeBps(1000); // 10%

    await USDT.connect(user).mint(user.address, migrationAmount);
    await DAI.connect(user).mint(collateralMigrator.address, parseEther("20"));

    // 포지션의 100%에 대한 플래시 론 금액이 필요하다고 시뮬레이션 (환율 1:1)
    const flashAmount = parseEther("200");

    const data = swapRouter.interface.encodeFunctionData("swap", [
        fromAsset,
        toAsset,
        parseEther("200"),
        parseEther("200")
    ]);

    const swapParams = {
        fromAsset,
        toAsset,
        fromAssetAmount: parseEther("200"),
        minToAssetAmount: parseEther("200"),
        data: data
    };

    // collateral migrator가 담보를 사용할 수 있도록 승인
    await USDT.connect(user).approve(qiUSDT.address, MaxUint256);
    await qiUSDT.connect(user).mint(migrationAmount);

    await qiUSDT.connect(user).approve(collateralMigrator.address, migrationAmount);

    await expect(
        collateralMigrator
            .connect(user)
            .migrateCollateral(fromMarket, toMarket, migrationAmount, flashAmount, swapParams)
        ).to.be.reverted; // 언더플로우로 인해 revert
});
```
참고: 이를 위해서는 수수료를 지원하기 위해 `contracts/mocks/MockLBPair.sol`에 다음과 같은 변경이 필요합니다:
```diff
diff --git a/contracts/mocks/MockLBPair.sol b/contracts/mocks/MockLBPair.sol
index 6074fa2..a1d28bd 100644
--- a/contracts/mocks/MockLBPair.sol
+++ b/contracts/mocks/MockLBPair.sol
@@ -16,6 +16,8 @@ contract MockLBPair {
     uint16 private _binStep;
     address private _factory;

+    uint128 feeBps;
+
     constructor(address tokenX, address tokenY, uint16 binStep, address factory) {
         _tokenX = tokenX;
         _tokenY = tokenY;
@@ -42,15 +44,21 @@ contract MockLBPair {
     function flashLoan(ILBFlashLoanCallback receiver, bytes32 amounts, bytes calldata data) external {
         (uint amountX, uint amountY) = amounts.decode();

+        bytes32 fees;
+
         if (amountX > 0 && amountY == 0) {
             IERC20(_tokenX).transfer(address(receiver), amountX);
+            uint128 feeX = uint128(amountX) * feeBps / 10000;
+            fees = feeX.encode(0);
         } else if (amountX == 0 && amountY > 0) {
             IERC20(_tokenY).transfer(address(receiver), amountY);
+            uint128 feeY = uint128(amountY) * feeBps / 10000;
+            fees = uint128(0).encode(feeY);
         } else {
             revert("MockLBPair: INVALID_AMOUNTS");
         }

-        receiver.LBFlashLoanCallback(msg.sender, _tokenX, _tokenY, amounts, 0, data);
+        receiver.LBFlashLoanCallback(msg.sender, _tokenX, _tokenY, amounts, fees, data);
     }

     function encodeAmounts(uint128 amountX, uint128 amountY) external pure returns (bytes32) {
@@ -86,4 +94,8 @@ contract MockLBPair {
     function getFactory() external view returns (address factory) {
         return _factory;
     }
+
+    function setFeeBps(uint128 _feeBps) external {
+        feeBps = _feeBps;
+    }
 }
```

**권장 완화 방법:** 스왑 전후 잔액의 차이를 계산하는 대신, 남은 전체 잔액을 직접 환불하는 것을 고려하세요:

```diff
- uint256 contractBalanceBF = IERC20(trgMarketConfig.flashData.baseToken).balanceOf(address(this));

  ...

  // 남은 기본 자산을 시장에 반환
- uint256 contractBalanceAF = IERC20(trgMarketConfig.flashData.baseToken).balanceOf(address(this));
- uint256 remainingAmount = contractBalanceAF - contractBalanceAF - contractBalanceBF;
+ uint256 remainingAmount = IERC20(trgMarketConfig.flashData.baseToken).balanceOf(address(this));

  if (remainingAmount > 0) {
      _supplyToMarket(user, targetMarket, remainingAmount);
  }
```

**Benqi:** 커밋 [`3828ee7`](https://github.com/woof-software/benqi-collateral-migrator/commit/3828ee73ac4471d58824b2f6051a3a8049b9cadc)에서 수정됨.

**Cyfrin:** 검증됨. 들어오는 잔액이 이제 추적됩니다. 스왑이 플래시 론 및 수수료 실행을 커버할 만큼 충분한 잔액을 남기지 않으면 revert됩니다.


### 담보 마이그레이션 시 소스 토큰의 미세 잔액(dust)이 남게 됨 (Migrating collateral will leave dust amounts of source tokens behind)

**설명:** 담보를 마이그레이션할 때 사용자는 소스 시장 토큰을 얼마나 마이그레이션할지와 관련된 두 가지 별도 금액을 제공합니다: `migrationAmount` 및 `swapParams.fromAssetAmount`.

- `migrationAmount`는 상환할 `qiTokens`(Compound V2 `CTokens`)의 수입니다. 이는 [`CollateralMigrator::LBFlashLoanCallback#L474-L482`](https://github.com/woof-software/benqi-collateral-migrator/blob/d91ce3dbf56d3939d54520f6f41afeebbbfc069a/contracts/CollateralMigrator.sol#L474-L482)에서 처리됩니다:

```solidity
// 소스 시장에서 마이그레이션 금액 상환
if (migrationAmount == type(uint256).max) {
    // 소스 시장의 QiToken 잔액 가져오기
    migrationAmount = IMinimalQiToken(sourceMarket).balanceOf(user);
}
// QiToken을 계약으로 전송
IERC20(sourceMarket).safeTransferFrom(user, address(this), migrationAmount);
// 소스 시장에서 QiToken 상환
if (IMinimalQiToken(sourceMarket).redeem(migrationAmount) != 0) revert RedeemFailed();
```

- `swapParams.fromAssetAmount`는 대상 시장 토큰으로 스왑될 기본 소스 토큰의 양이며, [`CollateralMigrator::LBFlashLoanCallback#L490-L497`](https://github.com/woof-software/benqi-collateral-migrator/blob/d91ce3dbf56d3939d54520f6f41afeebbbfc069a/contracts/CollateralMigrator.sol#L490-L497)에서 볼 수 있습니다:

```solidity
// 소스 시장에서 대상 시장으로 기본 자산 스왑
_swap(
    _swapParams.fromAsset,
    _swapParams.toAsset,
    _swapParams.fromAssetAmount,
    _swapParams.minToAssetAmount,
    _swapParams.data
);
```

문제는 이러한 값 간의 불일치에 있습니다: `migrationAmount`는 이자가 붙는 `qiTokens`의 수량을 나타내며, 상환되는 기본 토큰의 실제 양이 아닙니다. 시간이 지남에 따라 `qiTokens`는 이자를 축적하고 기본 자산의 증가하는 양을 나타냅니다.

사용자는 상환이 발생하기 전인 거래 제출 시점에 `fromAssetAmount`를 제공해야 하므로, 기본 자산을 얼마나 받게 될지 추측해야 합니다. 너무 높게 추측하면 스왑이 revert됩니다. 이를 피하기 위해 사용자는 보수적으로 추측해야 하며, 일부 기본 토큰은 `CollateralMigrator` 계약에서 사용되지 않은 상태로 남게 됩니다.

**영향:** `qiTokens`의 이자 발생 특성으로 인해 사용자는 기본 자산을 얼마나 받게 될지 정확하게 결정할 수 없습니다. 결과적으로 스왑 후 `CollateralMigrator` 계약에 미세한 양의 소스 토큰이 남게 됩니다.

이러한 토큰은 영구적으로 손실되지는 않지만(관리자 전용 `sweep` 함수를 통해 복구 가능), 사용자는 자신의 포지션을 완전히 마이그레이션할 수 없습니다.

**권장 완화 방법:** 부분 스왑을 수행하는 알려진 사용 사례가 없으므로 `fromAssetAmount` 매개변수를 완전히 제거하는 것을 고려하세요. 대신 소스 자산의 전체 잔액을 스왑하세요:

```diff
  // 소스 시장에서 대상 시장으로 기본 자산 스왑
  _swap(
      _swapParams.fromAsset,
      _swapParams.toAsset,
-     _swapParams.fromAssetAmount,
+     IERC20(_swapParams.fromAsset).balanceOf(address(this))
      _swapParams.minToAssetAmount,
      _swapParams.data
  );
```

**Benqi:** 커밋 [`e9525d1`](https://github.com/woof-software/benqi-collateral-migrator/commit/e9525d1676cf49fc59b513bbd3f37f41ef0bbe0c)에서 수정됨.

**Cyfrin:** 검증됨. 계약 잔액이 이제 스왑의 입력 금액으로 사용됩니다.


### 마이그레이션 후 사용자 LTV는 항상 악화됨 (User LTV always worsens after migration)

**설명:** 두 시장 간에 담보를 마이그레이션하는 흐름은 다음과 같습니다:

1. 최종 사용자가 필요한 스왑 매개변수와 함께 `CollateralMigrator::migrateCollateral`을 호출합니다.
2. `CollateralMigrator`는 Trader Joe 유동성 쌍에서 플래시 론을 받습니다.
3. 플래시 론을 사용하여 `CollateralMigrator`는 대상 시장에 포지션을 엽니다.
4. 이제 포지션이 초과 담보 상태가 되었으므로 원래 소스 시장 포지션을 닫을 수 있습니다.
5. 소스 토큰은 대상 토큰으로 스왑됩니다.
6. 대상 토큰은 관련 수수료와 함께 플래시 론을 상환하는 데 사용됩니다.
7. 남은 대상 토큰은 사용자를 대신하여 대상 시장에 예치됩니다.

문제는 플래시 론 수수료가 결과 포지션에 어떤 영향을 미치는지에 있습니다.

다음 예시를 고려해 보세요:

110 USDC에 대해 50 ETH를 빌린 포지션이 있으며, LTV는 약 45%입니다. 담보를 USDC에서 DAI로 마이그레이션하려고 합니다. 단순화를 위해 다음을 가정합니다:
- 10%의 플래시 론 수수료
- 1:1 USDC/DAI 환율

수수료를 충당하기 위해 필요한 추가 10 DAI는 스왑 수익금에서 나와야 하므로, 받을 수 있는 가장 큰 플래시 론은 100 DAI입니다.

그런 다음 110 USDC를 110 DAI로 스왑하고 100 DAI 대출과 10 DAI 수수료를 상환합니다. 남은 DAI는 0입니다. 새로운 담보 포지션은 이제 50 ETH를 뒷받침하는 100 DAI뿐이므로 LTV는 50%로 상승합니다.

플래시 론은 수수료와 함께 상환되어야 하고, 그 수수료는 사용자의 기존 담보에서 나와야 하므로, 모든 마이그레이션은 필연적으로 사용자의 LTV를 증가시켜 안전 마진을 줄입니다.

**영향:** 담보 마이그레이션은 항상 마이그레이션 후 더 높은 LTV를 초래하여 사용자의 청산 위험을 증가시킵니다.

**권장 완화 방법:** 사용자가 플래시 론 수수료를 충당하기 위해 추가 대상 토큰으로 거래를 충전할 수 있도록 하는 매개변수를 추가하는 것을 고려하세요. 이렇게 하면 원래 LTV를 유지할 수 있습니다.

**Benqi:** 커밋 [`afe450e`](https://github.com/woof-software/benqi-collateral-migrator/commit/afe450e3908305e39a732928a06bfdcae7d38d36)에서 수정됨.

**Cyfrin:** 검증됨. 사용자가 충전하여 LTV를 동일하게 유지할 수 있도록 하는 `extraFunds` 매개변수가 `migrateCollateral`에 추가되었습니다.


### `CollateralMigrator::_removeMigrationConfigData`의 무제한 루프가 계약 유지보수를 저해할 수 있음 (Unbounded loop in `CollateralMigrator::_removeMigrationConfigData` may hinder contract maintainability)

**설명:** [`CollateralMigrator::_removeMigrationConfigData`](https://github.com/woof-software/benqi-collateral-migrator/blob/d91ce3dbf56d3939d54520f6f41afeebbbfc069a/contracts/CollateralMigrator.sol#L790-L827)에는 마이그레이션 구성을 제거할 때 무제한 루프가 포함되어 있습니다:
```solidity
function _removeMigrationConfigData(address market) private {
    uint32 length = uint32(_markets.length);
    bool found = false;

    for (uint256 i = 0; i < length; i++) {
        if (_markets[i] == market) {
            found = true;
            // 목록에서 시장 제거
            _markets[i] = _markets[length - 1];
            _markets.pop();

            // 시장 데이터 및 플래시 데이터 제거
            delete _migrationConfigData[market];

            emit MigrationConfigDataRemoved(market);
            break;
        }
    }
    // 시장을 찾을 수 없으면 revert
    if (!found) revert MarketIsNotSupported(market);
}
```
이 구현은 제거할 시장을 찾기 위해 선형 검색을 사용합니다. 많은 수의 시장이 추가된 경우, 특히 배열 끝부분에 위치한 시장의 경우 루프가 과도한 가스를 소비할 수 있습니다. 최악의 경우 트랜잭션이 블록 가스 한도를 초과하여 시장을 제거하는 것이 불가능해질 수 있습니다.

또한 배열 끝부분에 위치한 시장을 제거하면 더 높은 가스 비용이 발생하며, 목록이 커짐에 따라 이는 무시할 수 없는 수준이 될 수 있습니다.

**영향:** 최악의 경우 가스 부족으로 인해 시장 제거가 실패할 수 있습니다. 성공하더라도 제거, 특히 목록 끝부분에 있는 시장의 제거는 불필요하게 비쌀 수 있습니다.

**권장 완화 방법:** 선형 검색을 수행하는 대신 제거할 시장의 인덱스를 허용하도록 함수를 변경하는 것을 고려하세요. 이렇게 하면 무제한 루프가 제거되고 가스 사용량이 줄어듭니다:

```solidity
function _removeMigrationConfigData(uint256 index) external onlyRole(DEFAULT_ADMIN_ROLE) {
    address market = _markets[index];

    _markets[index] = _markets[_markets.length - 1];
    _markets.pop();

    delete _migrationConfigData[market];

    emit MigrationConfigDataRemoved(market);
}
```

**Benqi:** 커밋 [`64aba7b`](https://github.com/woof-software/benqi-collateral-migrator/commit/64aba7b71a824e6ac20ae07c8f7746d785e425e0)에서 수정됨.

**Cyfrin:** 검증됨. 시장을 인덱스에 매핑하는 `_marketIndex`가 추가되었습니다. 이는 `_removeMigrationConfigData`를 호출하는 데 사용되며 루프는 제거되었습니다.

\clearpage
## Informational


### 사용되지 않는 import (Unused imports)

**설명:** 두 개의 사용되지 않는 import가 있습니다:
* [`CommonErrors`, `CollateralMigrator.sol#17`](https://github.com/woof-software/benqi-collateral-migrator/blob/d91ce3dbf56d3939d54520f6f41afeebbbfc069a/contracts/CollateralMigrator.sol#L17):
  ```solidity
  import {CommonErrors} from "./errors/CommonErrors.sol";
  ```
* [`ISwapper`, `SwapModule.sol#L6`](https://github.com/woof-software/benqi-collateral-migrator/blob/d91ce3dbf56d3939d54520f6f41afeebbbfc069a/contracts/modules/SwapModule.sol#L6):
  ```solidity
  import {ISwapper} from "../interfaces/1Inch-v6/ISwapper.sol";
  ```

이들을 제거하는 것을 고려하세요.

**Benqi:** 커밋 [`34d2cca`](https://github.com/woof-software/benqi-collateral-migrator/commit/34d2ccaad91fadbd723ec0e26a7fcdc1484b8028)에서 수정됨.

**Cyfrin:** 검증됨.


### NatSpec `@custom:reverts` 불일치 (NatSpec `@custom:reverts` inconsistencies)

**설명:** 다음은 NatSpec `@custom:reverts` 형식이 일관되지 않은 몇 가지 예입니다:

* [`CollateralMigrator::LBFlashLoanCallback`](https://github.com/woof-software/benqi-collateral-migrator/blob/d91ce3dbf56d3939d54520f6f41afeebbbfc069a/contracts/CollateralMigrator.sol#L438-L442):
  ```solidity
  * @custom:reverts IncorrectToAsset Thrown if the `toAsset` in `swapParams` does not match the target market's base token.
  * @custom:reverts RedeemFailed Thrown if the QiToken redemption process fails.
  * - {MintFailed}: Thrown if the minting of tokens to the target market fails.
  * @custom:reverts MintFailed Thrown if the minting of tokens to the target market fails.
  * @custom:reverts InvalidCallbackHash Thrown if the callback data hash does not match the stored hash.
  ```
  `MintFailed`는 한 번은 `@custom:reverts`와 함께, 한 번은 없이 두 번 언급됩니다.

* [`SwapModule::constructor`](https://github.com/woof-software/benqi-collateral-migrator/blob/d91ce3dbf56d3939d54520f6f41afeebbbfc069a/contracts/modules/SwapModule.sol#L62-L72)
  ```solidity
  /**
   * @notice Initializes the SwapModule with the address of the 1Inch router.
   * @param _swapRouter The address of the 1Inch router contract.
   * @dev Reverts with {InvalidZeroAddress} if `_swapRouter` is the zero address.
   */
  constructor(address _swapRouter) {
      if (_swapRouter == address(0)) {
          revert InvalidZeroAddress();
      }
      SWAP_ROUTER = _swapRouter;
  }
  ```
  사용자 지정 오류 `InvalidZeroAddress`는 문서에 `@custom:reverts`가 누락되어 있습니다.

* [`SwapModule::_swap`](https://github.com/woof-software/benqi-collateral-migrator/blob/d91ce3dbf56d3939d54520f6f41afeebbbfc069a/contracts/modules/SwapModule.sol#L84-L86)
  ```solidity
  * - {SwapFailed} if the call to the 1Inch router fails.
  * - {IncorrectSwapAmount} if the resulting balance does not increase.
  * - {OutputLessThanMinAmount} if the received amount is less than the specified minimum.
  ```
  `@custom:reverts`는 `CollateralMigrator`에서처럼 사용되지 않습니다.

**Benqi:** 커밋 [`053f28a`](https://github.com/woof-software/benqi-collateral-migrator/commit/053f28a6f303d97335a87b46edc2864a379ed098)에서 수정됨.

**Cyfrin:** 검증됨.

\clearpage
