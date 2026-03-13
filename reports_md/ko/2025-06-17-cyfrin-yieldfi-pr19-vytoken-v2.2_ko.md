**수석 감사자**

[Immeas](https://twitter.com/0ximmeas)

**보조 감사자**

[Alex Roan](https://twitter.com/alexroan)

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

---

# 발견 사항 (Findings)
## 저위험 (Low Risk)


### Manager를 통한 즉시 출금이 출금 수수료를 우회함

**설명:** 사용자가 출금을 수행할 때, 금액이 충분히 적고 `Manager`에 충분한 자산이 있는 경우 [`Manager::redeem`](https://github.com/YieldFiLabs/contracts/blob/e43fa029e2af65dae447882c53777e3bed387385/contracts/core/Manager.sol#L203-L208)에서 즉시 출금이 완료될 수 있습니다:

```solidity
// if redeeming yToken.asset() and vaultAssetAmount is less than maxRedeemCap and balance of contract is greater than vaultAssetAmount, redeem immediately and return
if (_asset == IERC4626(_yToken).asset() && vaultAssetAmount <= maxRedeemCap[_yToken] && IERC20(_asset).balanceOf(address(this)) >= vaultAssetAmount) {
    IERC20(_asset).safeTransfer(_receiver, vaultAssetAmount);
    emit InstantRedeem(caller, _yToken, _asset, _receiver, vaultAssetAmount);
    return;
}
```

그러나 이는 비동기 [`_withdraw`](https://github.com/YieldFiLabs/contracts/blob/e43fa029e2af65dae447882c53777e3bed387385/contracts/core/Manager.sol#L379-L395) 흐름에 적용된 수수료를 우회합니다.

**영향:** 출금은 `ERC4626` YToken 볼트와 `Manager` 계약을 통해 직접 시작할 수 있습니다. 이를 통해 사용자는 `Manager`를 통한 즉시 출금을 선택하여 YToken 출금 경로에 적용된 수수료를 우회할 수 있습니다.

**권장되는 완화 방법:** 즉시 출금 흐름에서도 수수료를 징수하는 것을 고려하십시오.

**YieldFi:** 인지하였습니다. 현재 수수료가 0으로 설정되어 있으므로 프로토콜 수수료에 영향을 미치지 않습니다.

\clearpage
## 정보성 (Informational)


### YToken 계약에서 `isNewYToken` 생략 가능

**설명:** 기초 자산의 회계를 지원하기 위해 `mintYToken`에 새 매개변수 `isNewYToken`이 도입되었습니다. 이 매개변수는 [`dYTokenL1::mintYToken`](https://github.com/YieldFiLabs/contracts/blob/702a931df3adb2f6e48807203cdc7a92604ea249/contracts/core/tokens/dYTokenL1.sol#L67-L81) 및 [`dYTokenL2::mintYToken`](https://github.com/YieldFiLabs/contracts/blob/702a931df3adb2f6e48807203cdc7a92604ea249/contracts/core/tokens/dYTokenL2.sol#L65-L79) 계약에서 `dYTokens` 발행이 기초 `YTokens`의 잔고도 업데이트해야 하는지 여부를 결정하는 데 사용됩니다.

그러나 이 매개변수는 [`YToken`](https://github.com/YieldFiLabs/contracts/blob/702a931df3adb2f6e48807203cdc7a92604ea249/contracts/core/tokens/YToken.sol#L215-L225) 및 [`YTokenL2`](https://github.com/YieldFiLabs/contracts/blob/702a931df3adb2f6e48807203cdc7a92604ea249/contracts/core/tokens/YTokenL2.sol#L198-L208) 구현에서 사용되지 않습니다:

```solidity
function mintYToken(address to, uint256 shares, bool isNewYToken) external virtual {
    require(msg.sender == manager, "!manager");
    _mint(to, shares);
}
```

매개변수를 생략하여 사용되지 않는 상태를 명시적으로 만드는 것을 고려하십시오:

```diff
- function mintYToken(address to, uint256 shares, bool isNewYToken) external virtual {
+ function mintYToken(address to, uint256 shares, bool ) external virtual {
```

**YieldFi:** 커밋 [`a3a9bad`](https://github.com/YieldFiLabs/contracts/commit/a3a9badf7a2ef877e128add79f52453a5cbc0fa5)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. `isNewYToken`이 위 함수 매개변수 선언에서 제거되었습니다.


### `YToken::_withdraw`에서 중복된 `virtual` 선언

**설명:** [풀 리퀘스트](https://github.com/YieldFiLabs/contracts/pull/19)에서 [`YToken::_withdraw`](https://github.com/YieldFiLabs/contracts/blob/702a931df3adb2f6e48807203cdc7a92604ea249/contracts/core/tokens/YToken.sol#L193) 함수는 `virtual`로 선언되어 파생 계약에서 오버라이드할 수 있도록 업데이트되었습니다. 그러나 실제로 `dYToken` 구현 중 어느 곳에서도 오버라이드되지 않습니다.

`YToken::_withdraw` 및 `YTokenL2::_withdraw` 모두에서 `virtual` 수정자를 제거하여 의도를 명확히 하고 오해의 소지가 있는 확장성을 피하는 것을 고려하십시오.

**YieldFi:** 인지하였습니다.


### 즉시 출금의 가격 변동 민감성

**설명:** 즉시 출금은 사용자가 기초 자산을 즉시 출금할 수 있도록 하므로 경제적 비효율성을 초래하는 방식으로 가격 변동에 반응할 가능성이 있습니다. 가격이 L2의 오라클을 통해 전달되기 때문에 두 가지 잠재적인 악용 벡터가 생성됩니다:

1. 크로스체인 차익거래(Arbitrage): L1과 L2 간의 큰 가격 차이는 체인 간 차익거래를 수행하는 사용자가 악용할 수 있습니다.
2. 가격 업데이트 샌드위치 공격(Sandwiching): 사용자가 상당한 가격 변동을 관찰하면 변경 직전에 입금하고 직후에 출금하여 기존 보유자의 비용으로 이익을 얻을 수 있습니다. 이는 반대로도 작동합니다. 사용자는 큰 하락 직전에 출금하여 다른 사람들이 겪을 손실을 피할 수 있습니다.

이러한 행동은 사용자가 한 번에 상환할 수 있는 금액을 제한하는 즉시 출금 상한선과 즉시 상환에 사용할 수 있는 제한된 잔고만 유지함으로써 어느 정도 완화됩니다.

그러나 L1과 L2 간의 가격 델타를 모니터링하거나 상당한 가격 변동을 감지하는 것이 유익할 수 있습니다. 이러한 경우 변동성이 큰 조건에서 잠재적인 악용을 방지하기 위해 계약을 일시 중지하는 것을 고려하십시오.

**YieldFi:** 인지하였습니다. L1과 L2 간의 가격 차이와 단기 가격 변동은 일반적으로 작습니다. 정상적인 조건에서는 이러한 행동이 수익성이 없을 것입니다. 치명적인 사건이 발생하는 경우, 변경 사항이 해결되는 동안 영향을 받는 볼트를 일시 중지할 수 있습니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### `isNewYToken == false`일 때 `dYToken::mintYToken`에서 불필요한 계산 방지

**설명:** 새로운 [`dYToken::mintYToken`](https://github.com/YieldFiLabs/contracts/blob/702a931df3adb2f6e48807203cdc7a92604ea249/contracts/core/tokens/dYTokenL1.sol#L67-L81)에는 입금 또는 누적된 수수료를 통해 생성된 새로 발행된 `dYTokens`를 처리하기 위한 특별한 로직이 있습니다:

```solidity
function mintYToken(address to, uint256 shares, bool isNewYToken) external override {
    require(msg.sender == manager, "!manager");
    uint256 assets = convertToAssets(shares);

    // if isNewYToken i.e external deposit has triggered minting of dyToken, we mint yToken to this contract
    if(isNewYToken) {
        // corresponding shares of yToken based on assets
        uint256 yShares = YToken(yToken).convertToShares(assets);
        // can pass isNewYToken here as it is not used in yToken
        ManageAssetAndShares memory manageAssetAndShares = ManageAssetAndShares({
            yToken: yToken,
            shares: yShares,
            assetAmount: assets,
            updateAsset: true,
            isMint: true,
            isNewYToken: isNewYToken
        });
        IManager(manager).manageAssetAndShares(address(this), manageAssetAndShares);
    }
    // minting dYToken to receiver
    _mint(to, shares);
}
```

`assets` 변수는 `if (isNewYToken)` 블록 내에서만 사용됩니다. 선언을 블록 내부로 이동하면 불필요한 계산을 피하여 `isNewYToken == false`일 때 가스를 절약할 수 있습니다:

```diff
function mintYToken(address to, uint256 shares, bool isNewYToken) external override {
    require(msg.sender == manager, "!manager");
-   uint256 assets = convertToAssets(shares);

    // if isNewYToken i.e external deposit has triggered minting of dyToken, we mint yToken to this contract
    if(isNewYToken) {
+       uint256 assets = convertToAssets(shares);
        // corresponding shares of yToken based on assets
        uint256 yShares = YToken(yToken).convertToShares(assets);
```

**YieldFi:** 커밋 [`f1f6996`](https://github.com/YieldFiLabs/contracts/commit/f1f69960c4d6d84aa8fe7658ac535a79fb77f505)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. `convertToAssets`가 이제 if 문 내부로 이동되었습니다.

\clearpage
