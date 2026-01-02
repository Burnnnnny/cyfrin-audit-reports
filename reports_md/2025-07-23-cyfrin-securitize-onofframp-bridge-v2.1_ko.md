**수석 감사**

[Hans](https://twitter.com/hansfriese)

ChainDefenders ([1337web3](https://x.com/1337web3) & [PeterSRWeb3](https://x.com/PeterSRWeb3))

**보조 감사**

---

# 발견 사항

## 치명적 위험 (Critical Risk)

### 서명 확인 시 nonce 유효성 검사 누락으로 트랜잭션 재생 공격 가능

**설명:** `SecuritizeOnRamp::executePreApprovedTransaction` 함수는 트랜잭션을 실행하기 전에 트랜잭션 데이터에 제공된 nonce가 투자자의 예상 nonce와 일치하는지 확인하지 않습니다. 함수는 EIP-712 서명에 서명된 메시지의 일부로 nonce가 포함되어 있는지 확인하지만, `txData.nonce`가 현재 `noncePerInvestor[txData.senderInvestor]` 값과 같은지 확인하지 않습니다. 함수는 서명 확인 후에만 저장된 nonce를 증가시키므로 이전의 유효한 서명을 원래 nonce 값으로 재생할 수 있습니다.

`SecuritizeOnRamp.sol`의 184-199행에 있는 현재 구현은 다음과 같습니다:
```solidity
function executePreApprovedTransaction(
    bytes memory signature,
    ExecutePreApprovedTransaction calldata txData
) public override whenNotPaused {
    bytes32 digest = hashTx(txData);
    address signer = ECDSA.recover(digest, signature);

    // Check recovered address role
    IDSTrustService trustService = IDSTrustService(dsToken.getDSService(dsToken.TRUST_SERVICE()));
    uint256 signerRole = trustService.getRole(signer);
    if (signerRole != trustService.EXCHANGE() && signerRole != trustService.ISSUER()) {
        revert InvalidEIP712SignatureError();
    }
    noncePerInvestor[txData.senderInvestor] = noncePerInvestor[txData.senderInvestor] + 1;
    Address.functionCall(txData.destination, txData.data);
}
```

이 취약점으로 인해 공격자는 이전에 유효한 서명된 트랜잭션을 재생하여 동일한 서명으로 여러 구독 또는 기타 작업을 실행할 수 있습니다.

**파급력:** 공격자는 이전에 유효한 EIP-712 서명된 트랜잭션을 재생하여 중복 구독 작업을 실행할 수 있으며, 이는 의도하지 않은 토큰 교환 및 회계 실패로 이어질 수 있습니다.

**개념 증명 (Proof of Concept):**
아래의 PoC를 `on-ramp.test.ts`에 추가하십시오.
```typescript
  it('Should allow replay attack - nonce not validated properly', async function () {
      // Setup initial balances and approvals for multiple transactions
      await usdcMock.mint(unknownWallet, 2e6); // Mint 2M USDC for two transactions
      await dsTokenMock.issueTokens(assetProviderWallet, 2e6); // Issue 2e6 DS tokens

      const liquidityFromInvestor = usdcMock.connect(unknownWallet) as Contract;
      await liquidityFromInvestor.approve(onRamp, 2e6); // Approve 2 USDC

      const dsTokenFromAssetProviderWallet = dsTokenMock.connect(assetProviderWallet) as Contract;
      await dsTokenFromAssetProviderWallet.approve(assetProvider, 2e6); // Approve 2e6 DS tokens

      const calculatedDSTokenAmount = await onRamp.calculateDsTokenAmount(1e6);

      // Create first transaction with nonce 0
      const subscribeParams = [
          '1',
          await unknownWallet.getAddress(),
          'US',
          [],
          [],
          [],
          980000,
          1e6,
          blockNumber + 10,
          HASH,
      ];
      const txData = await buildTypedData(onRamp, subscribeParams);
      const signature = await eip712OnRamp(eip712Signer, await onRamp.getAddress(), txData);

      // Verify initial nonce is 0
      expect(await onRamp.nonceByInvestor('1')).to.equal(0);

      // Execute first transaction successfully
      await expect(onRamp.executePreApprovedTransaction(signature, txData))
          .emit(onRamp, 'Swap')
          .withArgs(onRamp, calculatedDSTokenAmount, 1e6, unknownWallet);

      // Verify nonce is now 1 after first transaction
      expect(await onRamp.nonceByInvestor('1')).to.equal(1);

      // Verify first transaction effects
      expect(await dsTokenMock.balanceOf(unknownWallet)).to.equal(calculatedDSTokenAmount);
      expect(await usdcMock.balanceOf(unknownWallet)).to.equal(1e6); // 1e6 remaining

      // VULNERABILITY: Replay the same transaction with the same signature and nonce 0
      // This should fail but doesn't because nonce validation is missing
      await expect(onRamp.executePreApprovedTransaction(signature, txData))
          .emit(onRamp, 'Swap')
          .withArgs(onRamp, calculatedDSTokenAmount, 1e6, unknownWallet);

      // Verify the replay attack succeeded - investor got double the tokens
      expect(await dsTokenMock.balanceOf(unknownWallet)).to.equal(calculatedDSTokenAmount * 2n);
      expect(await usdcMock.balanceOf(unknownWallet)).to.equal(0); // All USDC spent

      // Verify nonce was incremented again (now 2) even though we replayed nonce 0
      expect(await onRamp.nonceByInvestor('1')).to.equal(2);
  });
```
**권장 완화 방법:** 트랜잭션을 실행하기 전에 nonce 유효성 검사를 추가하여 제공된 nonce가 투자자의 예상 nonce와 일치하는지 확인하십시오:

```diff
function executePreApprovedTransaction(
    bytes memory signature,
    ExecutePreApprovedTransaction calldata txData
) public override whenNotPaused {
+   // Validate nonce matches expected value
+   if (txData.nonce != noncePerInvestor[txData.senderInvestor]) {
+       revert InvalidEIP712SignatureError();
+   }
+
    bytes32 digest = hashTx(txData);
    address signer = ECDSA.recover(digest, signature);

    // Check recovered address role
    IDSTrustService trustService = IDSTrustService(dsToken.getDSService(dsToken.TRUST_SERVICE()));
    uint256 signerRole = trustService.getRole(signer);
    if (signerRole != trustService.EXCHANGE() && signerRole != trustService.ISSUER()) {
        revert InvalidEIP712SignatureError();
    }
    noncePerInvestor[txData.senderInvestor] = noncePerInvestor[txData.senderInvestor] + 1;
    Address.functionCall(txData.destination, txData.data);
}
```

**Securitize:** 커밋 [65179b](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/65179bcf41ed859106069dcaa751f5a2cec3038e)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 중간 위험 (Medium Risk)

### `executeTwoStepRedemption`에서 `minOutputAmount`의 잘못된 사용으로 인해 불필요한 되돌림 발생 가능

**설명:** `RedemptionManager::executeTwoStepRedemption` 함수에서 다음 호출이 수행됩니다:

```solidity
params.liquidityProvider.supplyTo(contractAddress, params.liquidityTokenAmount, params.minOutputAmount);
```

여기서 `params.minOutputAmount`는 유동성 공급자로부터 예상되는 최소 반환 값으로 사용됩니다. 그러나 이 값은 함수 뒷부분에서 적용되는 수수료 공제를 고려하지 않습니다.

`supplyTo` 호출 직후 계약은 슬리피지 보호 확인을 수행합니다:

```solidity
uint256 offRampBalance = params.liquidityProvider.liquidityToken().balanceOf(contractAddress);
uint256 fee = _getFee(params.feeManager, offRampBalance);

if (offRampBalance - fee < params.minOutputAmount) {
    revert Errors.SlippageControlError();
}
```

유동성 공급자가 정확히 `minOutputAmount`를 반환하면 해당 금액에서 수수료를 공제하면 `offRampBalance - fee`가 `minOutputAmount` 아래로 떨어져 유동성 공급자가 최소 요구 사항을 충족하더라도 슬리피지 오류가 발생합니다.

문제는 슬리피지 확인 자체에 있는 것이 아니라 수수료를 정확하게 계산하고 있습니다. 문제는 `supplyTo`에 전달된 `minOutputAmount`가 나중의 슬리피지 확인과 일관성을 보장하기 위해 수수료도 포함해야 한다는 것입니다.

**파급력:** 유동성 공급자가 `minOutputAmount` 요구 사항을 충족하더라도 슬리피지 오류로 인해 예상치 못한 트랜잭션 되돌림이 발생할 수 있습니다.

**권장 완화 방법:** `supplyTo`에 대한 호출을 업데이트하여 `minOutputAmount` 매개변수에 예상 수수료를 포함하도록 하십시오. 예:

```solidity
uint256 expectedFee = _getFee(params.feeManager, params.minOutputAmount);
params.liquidityProvider.supplyTo(contractAddress, params.liquidityTokenAmount, params.minOutputAmount + expectedFee);
```

이렇게 하면 수수료 후 금액이 예상 최소값을 충족하고 슬리피지 보호 확인 로직과 일치하게 됩니다.

**Securitize:** 커밋 [54243f](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/54243f7e6716826c30c9561c6390fa0e05440252)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `SecuritizeBridge::bridgeDSTokens`에서 계정 추상화 지갑에 대한 잘못된 주소 처리

**설명:** `SecuritizeBridge.bridgeDSTokens` 함수는 소스 체인의 사용자 지갑 주소(`msg.sender`)가 대상 체인의 원하는 수신자 주소와 동일하다고 가정합니다. 이 가정은 계정 추상화(AA) 지갑에 대해서는 올바르지 않습니다.

계정 추상화 지갑은 EOA(Externally Owned Accounts)와 달리 동일한 논리적 사용자에 대해 서로 다른 체인에서 서로 다른 주소를 가질 수 있는 스마트 계약 기반 지갑(예: Safe, Argent, ERC-4337 지갑)입니다.

이는 다음과 같은 이유로 발생합니다:
- AA 지갑은 체인별 매개변수가 있는 팩토리 계약을 사용하여 배포됩니다.
- 배포 솔트(salt), nonce 또는 팩토리 주소가 체인마다 다를 수 있습니다.
- 동일한 사용자의 지갑 로직은 서로 다른 체인에서 서로 다른 계약 주소를 생성합니다.

이로 인해 브리지된 DSToken이 대상 체인의 제어되지 않는 주소로 발행되어 자금이 영구적으로 손실될 수 있는 보안 위험이 발생합니다.

**파급력:** 토큰은 대상 체인에서 사용자가 제어할 수 없는 주소로 발행될 수 있으며, 투자자 등록이 잘못된 주소에 연결되어 KYC/AML 가정을 깨뜨릴 수 있습니다.

**개념 증명 (Proof of Concept):**
1. 사용자 Alice는 이더리움에서 주소 `0xAaa...111`의 Safe 지갑을 사용합니다.
2. Alice는 Polygon으로 1000 DSToken을 브리지하기 위해 `bridgeDSTokens`를 호출합니다.
3. 계약은 `msg.sender`(`0xAaa...111`)를 대상 주소로 인코딩합니다.
4. Polygon에서 Alice의 Safe 지갑 주소는 `0xBbb...222`입니다(다른 팩토리 배포).
5. DSToken은 Polygon의 `0xAaa...111`로 발행되며 Alice는 여기에 액세스할 수 없습니다.
6. Alice의 자금은 영구적으로 손실됩니다.

**권장 완화 방법:** 사용자가 대상 주소를 지정할 수 있도록 명시적인 `destinationWallet` 매개변수를 추가하십시오:
```solidity
function bridgeDSTokens(
    uint16 targetChain,
    uint256 value,
    address destinationWallet
) external override payable whenNotPaused {
    require(destinationWallet != address(0), "Invalid destination wallet");

    // ... existing validation code ...

    wormholeRelayer.sendPayloadToEvm{value: msg.value} (
        targetChain,
        targetAddress,
        abi.encode(
            investorDetail.investorId,
            value,
            destinationWallet, // ← User-specified destination
            investorDetail.country,
            investorDetail.attributeValues,
            investorDetail.attributeExpirations
        ),
        0,
        gasLimit,
        whChainId,
        msg.sender
    );
}
```

**Securitize:** 인지함.

**Cyfrin:** 인지함.

### 브리지 수신 함수의 일시 중지 수정자가 전송 중인 메시지에 대해 수신 실패를 유발함

**설명:** `USDCBridge::receivePayloadAndUSDC` 및 `SecuritizeBridge::receiveWormholeMessages` 함수는 `whenNotPaused` 수정자에 의해 보호되므로 각 브리지 계약이 일시 중지될 때 이러한 함수가 되돌려집니다(revert). [Wormhole 문서](https://wormhole.com/docs/products/messaging/guides/wormhole-relayers/#delivery-statuses)에 따르면 수신 함수가 되돌려지면 메시지 상태는 "Receiver Failure"가 되며 자동 재시도 메커니즘을 사용할 수 없습니다. 수신 실패에서 복구하는 유일한 방법은 소스 체인에서 전체 프로세스를 다시 시작하는 것입니다.

```solidity
// USDCBridge.sol
function receivePayloadAndUSDC(
    bytes memory payload,
    uint256 amountUSDCReceived,
    bytes32 sourceAddress,
    uint16 sourceChain,
    bytes32 deliveryHash
) internal override onlyWormholeRelayer whenNotPaused {
    // Function will revert if contract is paused
    // ...
}

// SecuritizeBridge.sol
function receiveWormholeMessages(
    bytes memory payload,
    bytes[] memory additionalVaas,
    bytes32 sourceBridge,
    uint16 sourceChain,
    bytes32 deliveryHash
) public override payable whenNotPaused {
    // Function will revert if contract is paused
    // ...
}
```

이로 인해 브리지 계약에서 제공하는 내장 복구 메커니즘 없이 전송 중인 메시지와 관련된 자금이 갇히는 운영 문제가 발생합니다.

문제가 되는 시나리오:
1. 사용자가 체인 A에서 체인 B로 크로스 체인 전송을 시작합니다.
2. 체인 B의 브리지 계약이 긴급 상황이나 유지 관리로 인해 일시 중지됩니다.
3. Wormhole 릴레이어가 메시지를 체인 B로 전달하려고 시도합니다.
4. `whenNotPaused` 수정자로 인해 수신 함수가 되돌려집니다.
5. 메시지 상태가 영구적으로 "Receiver Failure"가 됩니다.
6. 자동 복구 메커니즘 없이 자금이 갇힙니다.

**파급력:** 브리지 계약이 일시 중지될 때 전송 중인 크로스 체인 메시지와 관련된 자금이 갇혀 자산을 복구하기 위해 수동 개입이 필요합니다.

**권장 완화 방법:** 수신 실패를 방지하기 위해 수신 함수에서 `whenNotPaused` 수정자를 제거하십시오.
또한 수신 프로세스 성공 플래그와 함께 수신된 메시지를 추적하고 관리자가 실패한 메시지를 다시 시도할 수 있도록 허용하는 것을 고려하십시오.

**Securitize:** 커밋 [97e37b](https://bitbucket.org/securitize_dev/bc-securitize-bridge-sc/commits/97e37bed37168bc1ca73fb18f06fbae06161819d)에서 부분적으로 수정되었으며, `USDCBridge` 수신 함수에 대해 `whenNotPaused`가 제거되었습니다.

**Cyfrin:** 확인됨.

### `CollateralLiquidityProvider::availableLiquidity`의 가용 유동성 계산이 담보 자산과 유동성 토큰 간의 1:1 비율을 가정함

**설명:** `CollateralLiquidityProvider::availableLiquidity` 함수는 담보 제공자가 보유한 담보 자산의 잔액을 잘못 반환하며, 담보 자산과 상환자에게 실제로 제공될 유동성 토큰 간의 1:1 비율을 가정합니다. 실제 상환자에게 제공되는 유동성은 `externalCollateralRedemption.redeem()` 함수를 통과하며, 이 함수는 1:1 가정을 깨뜨리는 수수료, 환율 또는 기타 변환 메커니즘을 적용할 수 있기 때문에 이 가정은 결함이 있습니다.

```solidity
function availableLiquidity() external view returns (uint256) {
    return IERC20(externalCollateralRedemption.asset()).balanceOf(collateralProvider);
}

function _availableLiquidity() private view returns (uint256) {
    return IERC20(externalCollateralRedemption.asset()).balanceOf(collateralProvider);
}

function supplyTo(
    address redeemer,
    uint256 amount,
    uint256 minOutputAmount
) public whenNotPaused onlySecuritizeRedemption {
    if (amount > _availableLiquidity()) {
        revert InsufficientLiquidity(amount, _availableLiquidity());
    }

    // ... collateral transfer and redemption logic ...

    // The actual liquidity provided is calculated here, not the raw collateral amount
    uint256 assetsAfterExternalCollateralRedemptionFee = externalCollateralRedemption.calculateLiquidityTokenAmount(
        amount
    );

    liquidityToken.transfer(redeemer, assetsAfterExternalCollateralRedemptionFee);
}
```

`CollateralLiquidityProvider::supplyTo`가 호출되면 흐름에는 다음이 포함됩니다: 담보 제공자로부터 담보 자산 전송, 담보를 유동성 토큰으로 변환하기 위한 `externalCollateralRedemption.redeem()` 호출, `externalCollateralRedemption.calculateLiquidityTokenAmount()`를 사용하여 실제 유동성 금액 계산, 그리고 마지막으로 계산된 유동성 토큰을 상환자에게 전송.
`availableLiquidity()` 함수는 원시 담보 자산 잔액을 사용하는 대신 외부 상환 계약을 쿼리하여 제공될 수 있는 실제 유동성을 결정해야 합니다. (예: `externalCollateralRedemption.calculateLiquidityTokenAmount(IERC20(externalCollateralRedemption.asset()).balanceOf(collateralProvider));`)

또한 외부 `availableLiquidity()` 함수는 의도된 디자인 패턴에 반하여 내부 `_availableLiquidity()` 함수를 호출하는 대신 로직을 복제하여 불필요한 코드 중복을 생성합니다.

**파급력:** 사용자와 통합 시스템은 가용 유동성에 대한 잘못된 정보를 받을 수 있으며, 실제 전환 가능한 유동성이 보고된 담보 자산 잔액보다 적을 경우 트랜잭션이 실패할 수 있습니다.

**권장 완화 방법:** 외부 상환 계약을 쿼리하여 제공할 수 있는 실제 유동성을 계산하도록 `availableLiquidity()` 함수를 업데이트하고, 의도한 대로 내부 `_availableLiquidity()` 함수를 호출하도록 함수를 수정하십시오:

```diff
function availableLiquidity() external view returns (uint256) {
-    return IERC20(externalCollateralRedemption.asset()).balanceOf(collateralProvider);
+    return _availableLiquidity();
}

function _availableLiquidity() private view returns (uint256) {
-    return IERC20(externalCollateralRedemption.asset()).balanceOf(collateralProvider);
+    uint256 collateralBalance = IERC20(externalCollateralRedemption.asset()).balanceOf(collateralProvider);
+    return externalCollateralRedemption.calculateLiquidityTokenAmount(collateralBalance);
}
```

**Securitize:** 커밋 [1da35c](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/bf970d6cc4152c1b22e386b6acc6095aece8f12a) 및 [4a426e](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/4a426e689586a37cdcb463dba2f670fd58190ef9)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 낮은 위험 (Low Risk)

### 브리지 가스 제한 매개변수에 하한 검증이 없어 잠재적인 전달 실패로 이어짐

**설명:** `USDCBridge` 계약의 `updateGasLimit` 함수는 관리자가 `gasLimit` 매개변수를 임의의 값으로 설정할 수 있도록 합니다. 이 매개변수는 크로스 체인 USDC 전송을 인용하고 전송할 때, 특히 대상 체인에서 `receivePayloadAndUSDC` 함수의 실행을 위한 가스 제한으로 사용됩니다. 관리자가 이 값을 너무 낮게 설정하면 크로스 체인 메시지 전달이 대상 체인에서 실행을 완료할 만큼 충분한 가스를 갖지 못하게 됩니다.

**파급력:** `gasLimit`이 필요한 임계값 아래로 설정되면 브리지에 의해 시작된 모든 크로스 체인 전달은 `receivePayloadAndUSDC` 실행 중 가스 부족(out-of-gas) 오류로 인해 지속적으로 실패합니다. 이는 사용자가 브리지를 사용하여 체인 간에 USDC를 성공적으로 전송하는 것을 방지하여 사실상 브리지의 핵심 기능을 중단시킵니다. 자금이 갇히거나 실패한 전달을 해결하기 위해 수동 개입이 필요할 수 있습니다.

**권장 완화 방법:** 관리자가 `gasLimit`을 안전한 운영 임계값 아래로 설정하지 못하도록 `updateGasLimit` 함수에 최소 가스 제한 확인을 구현하십시오. 이 임계값은 추가 안전 마진을 포함하여 `receivePayloadAndUSDC` 함수의 최대 예상 가스 사용량을 기반으로 결정되어야 합니다. 선택적으로 이 최소값 아래로 가스 제한을 설정하려고 시도하면 이벤트를 방출하거나 트랜잭션을 되돌리십시오.

**Securitize:** 인지함.

**Cyfrin:** 인지함.

### `liquidityToken`에 대한 유효성 검사 누락

**설명:** `CollateralLiquidityProvider::initialize()` 함수는 전달된 `securitizeOffRamp`의 `liquidityProvider()`의 `liquidityToken`이 예상 토큰과 일치하는지 **확인하지 않습니다**. 이 확인은 `setExternalCollateralRedemption()`에는 있지만 여기에는 누락되어 있습니다.

**파급력:** 일치하지 않는 `liquidityToken`은 잘못된 구성, 자금 손실 또는 의도하지 않은 자산 상호 작용으로 이어질 수 있습니다.

**권장 완화 방법:**
`initialize()` 함수에 `securitizeOffRamp`의 유동성 공급자의 `liquidityToken`이 `_liquidityToken` 매개변수와 일치하는지 확인하는 검사를 추가하십시오:

```solidity
function initialize(
    address _liquidityToken,
    address _recipient,
    address _securitizeOffRamp
) public onlyProxy initializer {
    if (_recipient == address(0) || _liquidityToken == address(0) || _securitizeOffRamp == address(0)) {
        revert NonZeroAddressError();
    }

    address expectedToken = ILiquidityProvider(
        ISecuritizeOffRamp(_securitizeOffRamp).liquidityProvider()
    ).liquidityToken();

    if (expectedToken != _liquidityToken) {
        revert LiquidityTokenMismatch();
    }

    __BaseContract_init();
    recipient = _recipient;
    liquidityToken = IERC20(_liquidityToken);
    securitizeOffRamp = ISecuritizeOffRamp(_securitizeOffRamp);
}
```

**Securitize:** 인지함.

**Cyfrin:** 인지함.

### 초기화 중에 `liquidityProviderWallet`이 설정되지 않음

**설명:** `AllowanceLiquidityProvider::initialize`에서 주요 속성 중 하나인 `liquidityProviderWallet`이 초기화되지 않습니다. 이 속성은 public 변수로 선언되어 있지만 계약 초기화 중에 설정되지 않습니다. 계약이 `initialize`의 어느 시점에서도 이 값을 강제하거나 설정하지 않으므로 `liquidityProviderWallet`에 의존하는 모든 기능이 잘못 작동할 수 있습니다.

```solidity
address public liquidityProviderWallet;
```

**파급력:** `liquidityProviderWallet`이 설정될 때까지 `_availableLiquidity()` 및 `supplyTo()`와 같은 함수는 기본 address(0) 값에 의존합니다. 이로 인해 다음이 발생할 수 있습니다:

- 잘못된 유동성 값 반환(일반적으로 0).
- transferFrom(address(0), ...)이 실패하므로 상환 중 실패 또는 예상치 못한 동작 발생.

**권장 완화 방법:** `_liquidityProviderWallet` 매개변수를 수락하고 검증 및 할당되도록 `initialize` 함수를 업데이트하십시오:

```solidity
function initialize(
    address _liquidityToken,
    address _recipient,
    address _securitizeOffRamp,
    address _liquidityProviderWallet
) public onlyProxy initializer {
    if (_recipient == address(0)) revert NonZeroAddressError();
    if (_liquidityToken == address(0)) revert NonZeroAddressError();
    if (_securitizeOffRamp == address(0)) revert NonZeroAddressError();
    if (_liquidityProviderWallet == address(0)) revert NonZeroAddressError();

    __BaseContract_init();
    recipient = _recipient;
    liquidityToken = IERC20(_liquidityToken);
    securitizeOffRamp = ISecuritizeOffRamp(_securitizeOffRamp);
    liquidityProviderWallet = _liquidityProviderWallet;
}
```

이를 통해 `liquidityProviderWallet`이 계약 초기화 중에 한 번 설정되고 실수로 또는 악의적으로 초기화되지 않은 상태로 남지 않도록 합니다.

**Securitize:** 커밋 [ab08ae](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/ab08aea2c8c67ca311dcf46cd747621f84a14505)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 업그레이드 가능한 기본 계약에 스토리지 간격 누락

**설명:** `bc-securitize-bridge-sc` 리포지토리의 계약 `/contracts/utils/BaseContract`는 업그레이드 가능하지만(`UUPSUpgradeable`, `OwnableUpgradeable` 및 `PausableUpgradeable` 상속) 스토리지 간격(storage gap)을 포함하지 않습니다.

스토리지 간격은 상속하는 자식 계약의 스토리지 레이아웃에 영향을 주지 않고 향후 업그레이드에서 새 상태 변수를 기본 계약에 추가할 수 있도록 하는 데 필수적입니다.

**파급력:** 향후 `BaseContract` 버전에서 새 상태 변수를 추가하면 자식 계약에서 스토리지 충돌이 발생할 수 있습니다.

**권장 완화 방법:**
`BaseContract`에 스토리지 간격을 추가하십시오.
```solidity
uint256[50] private __gap;
```

**Securitize:** 커밋 [1da35c](https://bitbucket.org/securitize_dev/bc-securitize-bridge-sc/commits/1da35cde31a53e7b2de56de0d313ebdcb80cbfa3)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 단일 단계 소유권 이전 패턴은 권장되지 않음

**설명:** 두 `BaseContract` 구현은 현재 OpenZeppelin의 `OwnableUpgradeable`을 상속하며, 이는 **단일 단계 소유권 이전** 패턴을 사용합니다. 이 접근 방식은 위험합니다. 잘못된 주소가 새 소유자로 설정되면 계약이 관리 기능(`onlyOwner` 메서드)에 영구적으로 액세스할 수 없게 될 수 있습니다.

**파급력:** 잘못 구성된 소유권 이전은 중요한 관리 기능을 잠글 수 있어 잠재적으로 운영을 중단시키거나 긴급 업그레이드를 요구할 수 있습니다.

**권장 완화 방법:** OpenZeppelin의 **Ownable2StepUpgradeable** 계약을 채택하십시오. 이 2단계 소유권 이전 패턴은 새 소유자가 역할을 명시적으로 수락하도록 하여 우발적 잠금 위험을 줄입니다.

```diff
-abstract contract BaseContract is UUPSUpgradeable, PausableUpgradeable, OwnableUpgradeable {
+abstract contract BaseContract is UUPSUpgradeable, PausableUpgradeable, Ownable2StepUpgradeable {
```

**Securitize:** 인지함.

**Cyfrin:** 인지함.

### `_msgSender()` 대신 `msg.sender`를 사용하여 메타 트랜잭션 지원 방해

**설명:** 코드베이스의 여러 계약은 `_msgSender()` 대신 `msg.sender`를 직접 사용하여 적절한 메타 트랜잭션 지원을 방해합니다. 이러한 불일치는 시스템 전반의 초기화 및 핵심 기능 모두에 영향을 미칩니다.

영향을 받는 계약 및 함수는 다음과 같습니다:

- `BaseContract::__BaseContract_init()`
- `SecuritizeOffRamp::redeem()`
- `SecuritizeBridge::bridgeDSTokens()`
- `SecuritizeBridge::validateLockedTokens()`

프로토콜은 다른 함수 및 액세스 제어 패턴에서 `_msgSender()`를 올바르게 사용하여 메타 트랜잭션 지원에 대한 인식을 나타내지만 모든 함수에 일관되게 적용되지는 않았습니다.

사용자가 신뢰할 수 있는 전달자(forwarder)를 통해 메타 트랜잭션을 수행할 때:
1. 전달자는 사용자를 대신하여 계약을 호출합니다.
2. `msg.sender`를 사용하는 함수는 사용자 주소 대신 전달자의 주소를 받습니다.
3. 검증, 권한 부여 및 비즈니스 로직이 실패하거나 올바르게 작동하지 않습니다.

**파급력:** 전달자 계약이 의도한 사용자 대신 트랜잭션 전송자가 되므로 메타 트랜잭션 기능이 깨져 잠재적으로 권한 부여 실패, 잘못된 이벤트 방출 및 부적절한 검증 로직이 발생할 수 있습니다.

**권장 완화 방법:** 메타 트랜잭션을 지원하려면 모든 `msg.sender` 인스턴스를 `_msgSender()`로 교체하십시오:

```diff
// BaseContract.sol
function __BaseContract_init() internal onlyInitializing {
    __UUPSUpgradeable_init();
    __Pausable_init();
-   __Ownable_init(msg.sender);
+   __Ownable_init(_msgSender());
}

// SecuritizeOffRamp.sol
function redeem(uint256 assetAmount, uint256 minOutputAmount) external whenNotPaused nonZeroNavRate nonZeroLiquidityProvider {
    uint256 rate = navProvider.rate();

-   RedemptionValidator.validateRedemption(msg.sender, assetAmount, asset);
+   RedemptionValidator.validateRedemption(_msgSender(), assetAmount, asset);

-   CountryValidator.validateCountryRestriction(msg.sender, dsServiceConsumer, restrictedCountries);
+   CountryValidator.validateCountryRestriction(_msgSender(), dsServiceConsumer, restrictedCountries);

    // ... calculations ...

    RedemptionManager.RedemptionParams memory params = RedemptionManager.RedemptionParams({
        asset: asset,
        liquidityProvider: liquidityProvider,
        feeManager: feeManager,
        assetAmount: assetAmount,
        liquidityTokenAmount: liquidityTokenAmount,
        minOutputAmount: minOutputAmount,
-       redeemer: msg.sender,
+       redeemer: _msgSender(),
        assetBurn: assetBurn
    });

    // ... execution logic ...

    emit RedemptionCompleted(
-       msg.sender,
+       _msgSender(),
        assetAmount,
        liquidityTokenAmount,
        rate,
        fee,
        address(liquidityProvider.liquidityToken())
    );
}

// SecuritizeBridge.sol
function bridgeDSTokens(uint16 targetChain, uint256 value) external override payable whenNotPaused {
    uint256 cost = quoteBridge(targetChain);
    require(msg.value >= cost, "Transaction value should be equal or greater than quoteBridge response");
-   require(dsToken.balanceOf(msg.sender) >= value, "Not enough balance in source chain to bridge");
+   require(dsToken.balanceOf(_msgSender()) >= value, "Not enough balance in source chain to bridge");

    // ... validation logic ...

-   require(registryService.isWallet(msg.sender), "Investor not registered");
+   require(registryService.isWallet(_msgSender()), "Investor not registered");

-   string memory investorId = registryService.getInvestor(msg.sender);
+   string memory investorId = registryService.getInvestor(_msgSender());

    // ... other logic ...

-   dsToken.burn(msg.sender, value, BRIDGE_REASON);
+   dsToken.burn(_msgSender(), value, BRIDGE_REASON);

    wormholeRelayer.sendPayloadToEvm{value: msg.value}(
        targetChain,
        targetAddress,
        abi.encode(
            investorDetail.investorId,
            value,
-           msg.sender,
+           _msgSender(),
            investorDetail.country,
            investorDetail.attributeValues,
            investorDetail.attributeExpirations
        ),
        0,
        gasLimit,
        whChainId,
-       msg.sender
+       _msgSender()
    );

-   emit DSTokenBridgeSend(targetChain, address(dsToken), msg.sender, value);
+   emit DSTokenBridgeSend(targetChain, address(dsToken), _msgSender(), value);
}

function validateLockedTokens(string memory investorId, uint256 value, IDSRegistryService registryService) private view {
    // ... compliance service logic ...

-   uint256 availableBalanceForTransfer = complianceService.getComplianceTransferableTokens(msg.sender, block.timestamp, uint64(lockPeriod));
+   uint256 availableBalanceForTransfer = complianceService.getComplianceTransferableTokens(_msgSender(), block.timestamp, uint64(lockPeriod));
    require(availableBalanceForTransfer >= value, "Not enough unlocked balance in source chain to bridge");
}
```

**Securitize:** 커밋 [045925](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/045925798158710fef70ecdd0e47da1974b37bfd) 및 커밋 [1da35c](https://bitbucket.org/securitize_dev/bc-securitize-bridge-sc/commits/1da35cde31a53e7b2de56de0d313ebdcb80cbfa3)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `whChainId`는 불변 상수로 저장되어서는 안 됨

**설명:** 배포 중에 설정된 불변 `whChainId` 상수를 사용하는 `SecuritizeBridge` 계약은 체인 포크 중에 심각한 취약점을 생성합니다. 저장된 whChainId가 더 이상 실제 `chain ID`(block.chainid)와 일치하지 않으면 `sendPayloadToEvm`을 통한 크로스 체인 작업이 잘못된 소스 체인 ID를 사용할 수 있습니다.

**파급력:** 이 불일치로 인해 크로스 체인 메시지 전달이 실패하거나 환불이 잘못 처리되어 계약 기능이 중단될 수 있습니다. 결과적으로 환불 흐름에 자금이 잠겨 사용자의 재정적 손실로 이어지고 시스템의 신뢰성에 대한 신뢰가 약화될 수 있습니다.

**권장 완화 방법:** 불변 `whChainId`를 동적 체인 ID 검색으로 교체하십시오:

```solidity
// Remove: uint16 public immutable whChainId;

function getCurrentChainId() public view returns (uint16) {
    return uint16(block.chainid);
}

// In bridgeDSTokens function:
wormholeRelayer.sendPayloadToEvm{value: msg.value} (
    targetChain,
    targetAddress,
    abi.encode(/* payload data */),
    0,
    gasLimit,
    getCurrentChainId(),
    msg.sender
);
```

**Securitize:** 거부됨. `whChainId`는 EVM 체인 ID가 아니라 [여기](https://wormhole.com/docs/build/reference/chain-ids/)에 정의된 Wormhole 관련 체인 ID입니다.

**Cyfrin:** 인지함.

### `bridgeDSTokens` 함수의 환불 주소는 구성 가능해야 함

**설명:** `SecuritizeBridge::bridgeDSTokens` 함수에서 `msg.sender`가 환불 주소로 지정됩니다.
```solidity
        // Send Relayer message
        wormholeRelayer.sendPayloadToEvm{value: msg.value} (
            targetChain,
            targetAddress,
            abi.encode(
                investorDetail.investorId,
                value,
                msg.sender,
                investorDetail.country,
                investorDetail.attributeValues,
                investorDetail.attributeExpirations
            ), // payload
            0, // no receiver value needed since we"re just passing a message
            gasLimit,
            whChainId,
            msg.sender //@audit refund address, any leftover gas will be sent to this address
        );
```
Wormhole은 남은 가스를 `refundAddress`로 환불합니다. 그러나 `msg.sender`가 기본 가스 토큰을 인출하는 기능을 구현하지 않는 스마트 계약인 경우 잠재적인 환불이 영구적으로 액세스할 수 없게 됩니다. 이로 인해 원래 사용자가 복구할 수 없는 자금이 잠길 수 있습니다.

**권장 완화 방법:** `msg.sender`로 기본 설정하는 대신 환불 주소를 명시적으로 지정하는 매개변수를 추가하십시오. 이를 통해 사용자 또는 호출 계약이 환불을 받기 위한 대체 또는 외부 소유 계정(EOA)을 정의할 수 있습니다.

```solidity
function bridgeDSTokens(..., address refundAddress) external {
    require(refundAddress != address(0), "Invalid refund address");
    ...
}
```

**Securitize:** 인지함.

**Cyfrin:** 인지함.

### 공식적이지 않은 wormhole-solidity-sdk npm 패키지 사용은 보안 및 유지 관리 위험을 초래함

**설명:** 코드베이스의 브리지 계약은 npm의 `wormhole-solidity-sdk` 버전 0.9.0을 사용하고 있으며, 이는 Wormhole 팀에서 비공식 배포라고 확인했습니다. Wormhole 팀에 따르면 `sullof <francesco@sullo.co>`가 게시한 npm 패키지는 공식 릴리스가 아니며 유일하게 승인된 버전은 GitHub에서 사용할 수 있는 v0.1.0입니다. 공식 권장 접근 방식은 `forge install wormhole-foundation/wormhole-solidity-sdk@v0.1.0`을 사용하는 것입니다.

이 문제의 영향을 받는 계약은 다음과 같습니다:
- `SecuritizeBridge.sol` - `IWormholeReceiver` 및 `IWormholeRelayer` 가져오기
- `WormholeCCTPUpgradeable.sol` - `IWormholeRelayer`, `IWormhole` 및 `ITokenMessenger` 가져오기
- `USDCBridge.sol` - `CCTPSender` 및 `CCTPReceiver`를 통해 `WormholeCCTPUpgradeable`에서 상속
- `RelayerMock.sol` - `IWormholeRelayer` 가져오기

비공식 패키지는 `package.json`에 `"wormhole-solidity-sdk": "^0.9.0"`으로 종속성으로 선언되어 있으며 크로스 체인 메시지 전달 및 CCTP(Circle Cross-Chain Transfer Protocol) 기능을 위해 브리지 구현 전체에서 사용됩니다.

**파급력:** 비공식 SDK를 사용하면 코드베이스가 확인되지 않은 타사 코드에 의존하므로 잠재적인 보안 취약성, 호환성 문제 및 유지 관리 문제가 발생합니다.

**권장 완화 방법:** 비공식 npm 패키지를 공식 GitHub 릴리스로 교체하십시오.

**Securitize:** 커밋 [1da35c](https://bitbucket.org/securitize_dev/bc-securitize-bridge-sc/commits/1da35cde31a53e7b2de56de0d313ebdcb80cbfa3)에서 수정되었습니다.

**Cyfrin:** 확인됨. 패키지를 사용하는 다른 사람들에게 경고하기 위해 [트윗](https://x.com/hansfriese/status/1945048296461848901)을 게시했으며 작성자가 응답하여 프로토콜이 아닌 개인적인 용도로만 의도된 것임을 확인했습니다. 더 이상의 혼란을 방지하기 위해 작성자는 [패키지](https://x.com/sullof/status/1945490920809304324)를 내렸습니다.

\clearpage
## 정보성 (Informational)

### 온램프 계약의 오해의 소지가 있는 주석 및 문서 불일치

**설명:** 온램프 시스템의 여러 계약에는 잘못된 주석, 잘못된 문서 및 실제 기능을 잘못 나타내는 인터페이스 불일치가 포함되어 있습니다.

- `ISecuritizeOnRamp` 인터페이스는 코드베이스 어디에서도 방출되지 않는 `Buy` 이벤트를 문서화하고 `toggleInvestorSubscription` 문서에서 존재하지 않는 `swapFor` 함수를 참조합니다.
- `ISecuritizeOnRamp` 인터페이스는 `nonceByInvestor` 및 `calculateDsTokenAmount`를 상태 변경 함수로 잘못 선언하지만 실제로는 구현에서 뷰 함수입니다.
- `MintingAssetProvider`는 실제 계약 이름 대신 `@title IAssetProvider`를 사용하여 문서화되고 있는 계약에 대한 혼란을 야기합니다.
- `IAssetProvider::securitizeOnRamp` 함수 문서는 "on ramp contract" 대신 "on ramo contract"를 참조하는 오타를 포함합니다.
- `MpbsFeeManager::setRedemptionFee` 함수 문서는 수수료 백분율이 MBPS여야 하는데 베이시스 포인트(basis points) 단위라고 언급합니다.

**파급력:** 이러한 오해의 소지가 있는 주석은 개발자가 존재하지 않는 기능을 기대하여 계약을 잘못 통합하게 할 수 있습니다.

**Securitize:** 커밋 [2b6c3a](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/2b6c3a8efdc23b4e2fc5fed273987830fbeaee18)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 인터페이스 구현 함수에 불필요한 override 키워드

**설명:** `SecuritizeOnRamp` 계약의 여러 함수는 `override` 키워드를 불필요하게 사용합니다. Solidity에서 `override` 키워드는 인터페이스 함수를 구현할 때가 아니라 부모 계약의 함수를 재정의할 때만 필요합니다.

다음 함수는 불필요하게 `override` 키워드를 사용합니다:
- `SecuritizeOnRamp::nonceByInvestor`
- `SecuritizeOnRamp::subscribe`
- `SecuritizeOnRamp::swap`
- `SecuritizeOnRamp::executePreApprovedTransaction`
- `SecuritizeOnRamp::calculateDsTokenAmount`
- `SecuritizeOnRamp::updateAssetProvider`
- `SecuritizeOnRamp::updateNavProvider`
- `SecuritizeOnRamp::updateMinSubscriptionAmount`
- `SecuritizeOnRamp::updateBridgeParams`
- `SecuritizeOnRamp::toggleInvestorSubscription`

**파급력:** 불필요한 `override` 키워드는 계약의 상속 구조에 대한 혼란을 야기합니다.

**권장 완화 방법:** `override` 키워드를 제거하십시오.

**Securitize:** 커밋 [bf7b87](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/bf7b873d62fd346493c5726ea0a0726088926136)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `swap` 함수가 기존 투자자 구매에 대해 잘못된 이벤트 유형을 방출함

**설명:** `SecuritizeOnRamp::swap` 함수는 기존 투자자가 자산을 구매할 때 예상되는 `Buy` 이벤트 대신 `Swap` 이벤트를 방출합니다. 인터페이스 문서에 따르면 `Swap` 이벤트는 "새로운 구독 계약"을 위한 것이며 `Buy` 이벤트는 "기존 투자자가 자산을 구매할 때" 방출되어야 합니다.
`swap` 함수는 기존 등록 투자자(`investorExists` 수정자에 의해 강제됨)가 추가 자산을 구매하도록 특별히 설계되었으므로 의미상 "스왑" 작업이 아닌 "구매" 작업입니다.

**파급력:** 오프체인 시스템 및 이벤트 리스너는 기존 투자자 구매를 새로운 구독 계약으로 잘못 분류하여 투자자 활동에 대한 부정확한 추적 및 보고로 이어질 수 있습니다.

**권장 완화 방법:** `swap` 함수에서 `Swap` 이벤트 방출을 적절한 `Buy` 이벤트로 교체하십시오:

```diff
- emit Swap(_msgSender(), dsTokenAmount, _liquidityAmount, _msgSender());
+ emit Buy(_msgSender(), _liquidityAmount, dsTokenAmount, navProvider.rate());
```

**Securitize:** [2b6c3a](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/2b6c3a8efdc23b4e2fc5fed273987830fbeaee18)에서 수정되었습니다. Buy 이벤트는 더 이상 사용되지 않고 삭제되었습니다.

**Cyfrin:** 확인됨.

### 업그레이드 가능한 계약의 생성자에 `_disableInitializers()` 누락

**설명:** 다음 계약은 업그레이드 가능하지만 생성자에서 `_disableInitializers()`를 호출하지 않습니다:
- `MintingAssetProvider`
- `AllowanceAssetProvider`
- `SecuritizeOnRamp`
- `MbpsFeeManager`
- `SecuritizeOffRamp`
- `AllowanceLiquidityProvider`
- `CollateralLiquidityProvider`

업그레이드 가능한 계약 패턴(예: OpenZeppelin의 UUPS 또는 Transparent 프록시를 사용하는 패턴)에서 구현(로직) 계약은 프록시와 독립적으로 배포됩니다. 구현 계약이 생성자에서 `_disableInitializers()`를 호출하지 않으면 누구나 직접 초기화할 수 있으며, 이는 의도되지 않은 것이며 보안 위험을 초래할 수 있습니다. ([참조](https://docs.openzeppelin.com/upgrades-plugins/writing-upgradeable#initializing_the_implementation_contract))

**파급력:** 구현 계약이 직접 초기화되면 공격자는 자신을 소유자로 설정하거나 다른 권한 있는 역할을 할당하여 잠재적으로 업그레이드 프로세스를 방해하거나 혼란을 초래할 수 있습니다. 이는 프록시의 상태에 직접적인 영향을 미치지 않지만 업그레이드 가능성을 깨거나 서비스 거부를 허용하거나 시스템에서 예상치 못한 동작을 생성할 수 있습니다.

**권장 완화 방법:** 영향을 받는 각 계약에 `_disableInitializers()`를 호출하는 생성자를 추가하십시오. 이렇게 하면 구현 계약이 초기화되거나 다시 초기화될 수 없으므로 프록시 컨텍스트 외부에서 무단 또는 우발적인 초기화를 방지할 수 있습니다.

```solidity
constructor() {
    _disableInitializers();
}
```

영향을 받는 각 계약에 이것을 추가하십시오.

**Securitize:** 커밋 [088048](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/0880486f4e75df252c5e6a773b2f09a4956fdb87)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 국가 코드 확인이 충분하지 않음

**설명:** `CountryValidator` 클래스에서 `validateCountryCode` 메서드는 주어진 국가 코드의 유효성을 확인하기 위한 것입니다. 그러나 현재 잘못된 코드를 올바르게 필터링하지 않습니다. 결과적으로 유효한 [ISO 3166-1](https://en.wikipedia.org/wiki/ISO_3166-1)(및 alpha-2 또는 alpha-3) 국가 코드가 아닌 XX 또는 YYX와 같은 입력이 잘못되어 유효한 것으로 간주됩니다.

**권장 완화 방법:** 허용된 국가 코드 맵을 사용하는 것을 고려하십시오.

**Securitize:** 인지함. 엄격한 ISO 국가 코드 유효성 검사는 잘못된 코드가 엣지 케이스이며 온보딩 또는 KYC 중에 오프체인에서 처리될 수 있으므로 온체인에서 필요하지 않습니다. 온체인 형식 확인은 우리의 사용 사례에 충분합니다.

**Cyfrin:** 인지함.

### 수수료 관리자 계약의 혼란스러운 변수 명명

**설명:** 수수료 관리자 계약은 수수료 백분율 값에 대해 혼란스러운 변수 이름을 사용합니다.
`MbpsFeeManager`에서 변수 `fee`는 실제 수수료 금액이 아니라 MBPS(milli basis points) 단위의 수수료 백분율을 나타냅니다. 주석조차 "mbps로 표시된 수수료 (1000 mbps = 1%)"라고 명확히 하고 계산 공식 `(amount * fee + FEE_DENOMINATOR - 1) / FEE_DENOMINATOR`는 `fee`가 백분율 비율로 사용됨을 보여줍니다. 그러나 변수 이름 `fee`는 일반적으로 백분율 비율이 아닌 실제 수수료 금액을 암시합니다. 예를 들어, 수수료 관리자 계약은 `getFee(uint256 amount)` 함수를 노출합니다.

이 명명 규칙은 `fee`가 계산에 사용되는 백분율 비율이 아닌 실제 수수료 금액을 나타낼 것으로 기대하는 사람들에게 혼란을 줍니다.

**파급력:** 혼란스러운 변수 이름은 통합 오류, 수수료 계산 오해, 수수료 관리자와 상호 작용하는 계약의 잠재적인 버그로 이어질 수 있습니다.

**권장 완화 방법:** 수수료 변수 이름을 백분율을 나타내도록 명확하게 변경하십시오:

```diff
- uint256 public fee;
+ uint256 public feeMBPS;
```
더 명확하게 하려면 `feePercentageMBPS`를 사용하는 것을 고려하십시오.

**Securitize:** 커밋 [6a5d45](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/6a5d45daf788fe573cb435f24a033472b336b21a)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 주소 유효성 검사 수정자 `SecuritizeOffRamp::addressNonZero`의 사용되지 않는 매개변수

**설명:** `SecuritizeOffRamp::initialize()`, `SecuritizeOffRamp::updateLiquidityProvider()` 및 `SecuritizeOffRamp::updateNavProvider()` 함수에서 사용되는 `addressNonZero` 수정자는 `string memory parameter` 인수를 허용하지만 수정자 로직 내에서는 절대 사용하지 않습니다.
이 매개변수는 검증 중인 주소 매개변수에 대한 컨텍스트를 제공하기 위한 것으로 보이지만 오류 처리에서는 사용되지 않습니다.

```solidity
modifier addressNonZero(address _address, string memory parameter) {
    if (_address == address(0)) {
        revert NonZeroAddressError();
    }
    _;
}
```

수정자는 "asset", "navProvider", "feeManager", "liquidityProvider"와 같은 설명적인 문자열로 호출되지만 이 컨텍스트 정보는 오류 보고나 검증 로직에서 활용되지 않습니다. 비교를 위해 코드베이스의 다른 계약은 불필요한 매개변수를 취하지 않고 검증을 올바르게 구현하는 `USDCBridge.sol`의 `addressNotZero`와 같은 유사한 주소 검증 수정자를 사용합니다.

**파급력:** 사용되지 않는 매개변수는 일관성 없는 코드 패턴을 생성하고 주소 유효성 검사가 실패할 때 의미 있는 오류 컨텍스트를 제공할 기회를 놓칩니다.

**권장 완화 방법:** 다른 계약의 유사한 검증 패턴과 일관성을 유지하기 위해 `addressNonZero` 수정자에서 사용되지 않는 매개변수를 제거하십시오:

```diff
- modifier addressNonZero(address _address, string memory parameter) {
+ modifier addressNonZero(address _address) {
    if (_address == address(0)) {
        revert NonZeroAddressError();
    }
    _;
}
```

그리고 문자열 매개변수를 제거하도록 모든 사용 사이트를 업데이트하십시오:

```diff
- addressNonZero(_asset, "asset")
+ addressNonZero(_asset)
- addressNonZero(_navProvider, "navProvider")
+ addressNonZero(_navProvider)
- addressNonZero(_feeManager, "feeManager")
+ addressNonZero(_feeManager)
- addressNonZero(_liquidityProvider, "liquidityProvider")
+ addressNonZero(_liquidityProvider)
```

**Securitize:** 커밋 [fd5511](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/fd5511077e368ee1d84f2695fe67b77bb185d6a3)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `calculateLiquidityTokenAmountWithoutFee`의 불필요한 복잡성

**설명:** `SecuritizeOffRamp::calculateLiquidityTokenAmountWithoutFee` 함수는 자산과 유동성 토큰 간의 소수점 변환을 처리하는 3개의 분기 조건부 로직을 통해 불필요한 복잡성을 포함합니다.

이 중복 분기는 코드 복잡성, 가스 소비 및 분기 간의 일관성 없는 반올림 동작 가능성을 증가시킵니다. 동일한 기능을 유지하면서 함수를 상당히 단순화할 수 있습니다.

**권장 완화 방법:** `rate` 매개변수가 유동성 소수점으로 표현되므로 함수는 모든 소수점 시나리오를 처리하는 단일 계산으로 단순화될 수 있습니다:

```solidity
function calculateLiquidityTokenAmountWithoutFee(
    uint256 assetAmount,
    uint256 rate,
    uint256 liquidityDecimals,
    uint256 assetDecimals
) internal pure returns (uint256) {
    return (assetAmount * rate) / (10 ** assetDecimals);
}
```

**Securitize:** [2bb438](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/2bb4384976d1a75a00678c0ea74b403d95efb656) 및 [0deec2](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/0deec2c111de751fed8dc8b7e7faa06d129ec07c)에서 수정되었습니다.

**Cyfrin:** 확인됨.
NAV 비율의 소수점과 관련하여 의사 소통에 오류가 있었고 원래 권장 사항은 정확하지 않았습니다.
팀은 비율이 다른 유동성 토큰 소수점이 아니라 자산 토큰 자체의 소수점 단위임을 확인했습니다.
이는 다른 프로토콜에서 일반적인 관행이 아니며 약간의 이상한 계산을 도입하지만 완화된 공식 자체는 기술적으로 정확합니다.

### 사용되지 않는 라이브러리 및 구조체 정의로 배포 비용 증가 및 코드 명확성 감소

**설명:** `CCTPMessageLib` 라이브러리와 `CCTPMessage` 구조체는 정의되어 있지만 CCTP 통합 계약 전체에서 사용되지 않습니다. `WormholeCCTPUpgradeable.sol`에서 라이브러리는 `message` 및 `signature` 필드를 포함하는 `CCTPMessage` 구조체를 정의하고 계약은 `using CCTPMessageLib for *`로 이를 가져옵니다. 그러나 `CCTPBase::redeemUSDC` 함수는 정의된 구조체를 활용하는 대신 `abi.decode(cctpMessage, (bytes, bytes))`를 사용하여 수동으로 CCTP 메시지를 디코딩합니다.

```solidity
function redeemUSDC(bytes memory cctpMessage) internal returns (uint256 amount) {
    (bytes memory message, bytes memory signature) = abi.decode(cctpMessage, (bytes, bytes));
    uint256 beforeBalance = IERC20(USDC).balanceOf(address(this));
    circleMessageTransmitter.receiveMessage(message, signature);
    return IERC20(USDC).balanceOf(address(this)) - beforeBalance;
}
```

동일한 문제가 업스트림 wormhole SDK의 `CCTPBase.sol` 파일에도 존재하므로 적절한 정리 없이 복사되었을 수 있음을 시사합니다.

**파급력:** 사용되지 않는 코드는 기능적 이점을 제공하지 않으면서 배포 가스 비용을 증가시키고 코드 유지 관리성을 감소시킵니다.

**권장 완화 방법:** 계약에서 사용되지 않는 `CCTPMessageLib` 라이브러리와 `using` 문을 제거하십시오:

```diff
- library CCTPMessageLib {
-     struct CCTPMessage {
-         bytes message;
-         bytes signature;
-     }
- }

abstract contract CCTPSender is CCTPBase {
    uint8 internal constant CONSISTENCY_LEVEL_FINALIZED = 15;

-   using CCTPMessageLib for *;

    mapping(uint16 => uint32) public chainIdToCCTPDomain;
```

**Securitize:** 커밋 [97e37b](https://bitbucket.org/securitize_dev/bc-securitize-bridge-sc/commits/97e37bed37168bc1ca73fb18f06fbae06161819d)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 외부 담보 상환 호출에 슬리피지 보호 누락

**설명:** `CollateralLiquidityProvider::supplyTo()`에서 함수는 `minOutputAmount` 매개변수로 `0`을 사용하여 `externalCollateralRedemption.redeem(collateralAmount, 0)`을 호출하여 외부 상환 트랜잭션에 대한 슬리피지 보호를 제공하지 않습니다.
이 함수는 `onlySecuritizeRedemption` 수정자에 의해 보호되며 슬리피지 제어가 더 높은 수준에서 구현되는 `SecuritizeOffRamp` 계약에 의해서만 호출되도록 의도되었지만, 향후 구현에서 다른 통합이 이 함수를 사용하는 경우 잠재적인 문제로 이어질 수 있습니다.

**권장 완화 방법:** 예상 유동성 금액을 기반으로 적절한 최소 출력 금액을 계산하고 합리적인 슬리피지 허용 오차를 적용하십시오. 또는 `minOutputAmount` 매개변수를 수락하거나 예상 출력을 기반으로 계산하도록 함수를 수정하는 것을 고려하십시오.

**Securitize:** 인지함.

**Cyfrin:** 인지함.

\clearpage
