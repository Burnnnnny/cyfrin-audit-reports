**수석 감사자**

[Hans](https://twitter.com/hansfriese)


---

# 발견 사항 (Findings)
## 중대 위험 (Critical Risk)


### 상환 중 유동성 금액 계산 시 소수점 자릿수 오류

**설명:** `SecuritizeRedemption` 계약은 투자자가 자산(DS Token)을 유동성 토큰(Stable Coin)으로 상환할 수 있도록 `redeem()` 공개 함수를 제공합니다.
`_amount` 매개변수는 상환할 자산의 양을 나타내며 자산 토큰의 소수점 자릿수를 따릅니다.
이 함수는 `navProvider`가 제공하는 환율을 활용하며, 이 환율은 스테이블 코인의 소수점 자릿수를 따릅니다.
```solidity
SecuritizeRedemption.sol
77:     function redeem(uint256 _amount) whenNotPaused external override {
78:         uint256 rate = navProvider.rate();
79:         require(rate != 0, "Rate should be defined");
80:         require(asset.balanceOf(msg.sender) >= _amount, "Redeemer has not enough balance");
81:         require(address(liquidityProvider) != address(0), "Liquidity provider should be defined");
82:         require(liquidityProvider.availableLiquidity() >= _amount, "Not enough liquidity");
83:
84:         ERC20 stableCoin = ERC20(address(liquidityProvider.liquidityToken()));
85:         uint256 liquidity = _amount * rate / (10 ** stableCoin.decimals());
86:
87:         liquidityProvider.supplyTo(msg.sender, liquidity);
88:         asset.transferFrom(msg.sender, liquidityProvider.recipient(), _amount);
89:
90:         emit RedemptionCompleted(msg.sender, _amount, liquidity, rate);
91:     }
```
반환되는 유동성 금액이 계산되는 L85를 보면, `liquidity`가 자산 토큰의 소수점 자릿수를 따르게 되는데, 이는 유동성 토큰(스테이블 코인)의 소수점 자릿수와 다를 수 있습니다.
`CollateralLiquidityProvider::supplyTo()` 함수가 제공된 금액 그대로 `liquidityToken`을 전송하므로, 여기서 `liquidity` 값은 스테이블 코인의 소수점 자릿수여야 함을 확인할 수 있습니다.
`liquidityToken=USDC`가 6자리 소수점이고 `asset`이 18자리 소수점인 표준 ERC20인 현실적인 시나리오를 가정하면, 이 취약점은 실제로 필요한 가치의 1e12배를 상환자에게 전송하게 됩니다.

**파급력:** 현실적인 시나리오에서 프로토콜에 심각한 금전적 손실을 입힐 수 있으므로 중대(CRITICAL) 위험으로 평가합니다.

**권장 완화 조치:**
```diff
-         uint256 liquidity = _amount * rate / (10 ** stableCoin.decimals());
+         uint256 liquidity = _amount * rate / (10 ** asset.decimals());
```

**Securitize:** 커밋 [3977ca](https://bitbucket.org/securitize_dev/bc-redemption-sc/commits/3977ca8ffb259a01e8dab894745751cf2150abf4)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `SecuritizeSwap::buy` 함수에서 `msg.sender`에 대한 검증 누락

**설명:** `SecuritizeSwap::buy` 함수는 투자자가 `stableCoinToken`을 `dsToken`으로 교환(swap)하는 데 사용되며, `SecuritizeSwap::swap` 함수와의 주요 차이점은 `buy`는 누구나 호출할 수 있는 반면 `swap` 함수는 `Issuer` 또는 `Master`만 호출할 수 있다는 것입니다.
문제는 `buy()` 함수가 호출자가 실제로 `_senderInvestorId`의 투자자인지 검증하지 않는다는 것입니다.
```solidity
SecuritizeSwap.sol
104:     function buy(//@audit-info public call that investors can directly call
105:         string memory _senderInvestorId,
106:         address _investorWallet,
107:         uint256 _stableCoinAmount,
108:         uint256 _blockLimit,
109:         uint256 _issuanceTime
110:     ) public override whenNotPaused {
111:         require(_blockLimit >= block.number, "Transaction too old");
112:         require(stableCoinToken.balanceOf(_investorWallet) >= _stableCoinAmount, "Not enough stable tokens balance");
113:         require(IDSRegistryService(dsToken.getDSService(DS_REGISTRY_SERVICE)).isInvestor(_senderInvestorId), "Investor not registered");
114:         require(navProvider.rate() > 0, "NAV Rate must be greater than 0");
115:         (uint256 dsTokenAmount, uint256 currentNavRate) = calculateDsTokenAmount(_stableCoinAmount);
116:         stableCoinToken.transferFrom(_investorWallet, issuerWallet, _stableCoinAmount);
117:
118:         dsToken.issueTokensCustom(_investorWallet, dsTokenAmount, _issuanceTime, 0, "", 0);
119:
120:         emit Buy(msg.sender, _stableCoinAmount, dsTokenAmount, currentNavRate, _investorWallet);
121:     }
```
구현에서 볼 수 있듯이, 함수는 제공된 `_senderInvestorId`가 실제로 등록되었는지만 확인하고 `msg.sender`가 해당 ID에 할당된 투자자인지는 확인하지 않습니다.
이 취약점은 투자자가 `SecuritizeSwap` 계약에 승인(allowance)을 부여한 한, 누구나 해당 투자자가 `stableCoinToken`으로 `dsToken`을 구매하도록 강제할 수 있게 합니다.

**파급력:** 누구나 투자자의 경제적 상태를 마음대로 변경할 수 있으므로 중대(CRITICAL) 위험으로 평가합니다.

**권장 완화 조치:** `msg.sender`가 레지스트리 서비스를 활용하는 실제 투자자인지 검증하십시오.

**Securitize:** 커밋 [b09460](https://bitbucket.org/securitize_dev/securitize-swap/commits/b094604b341123a49c8abbd6e1c3d53d7c102f28)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 고위험 (High Risk)


### `SecuritizeRedemption::redeem()` 함수의 `availableLiquidity`에 대한 잘못된 검증

**설명:** `SecuritizeRedemption::redeem(uint256 _amount)` 함수는 투자자가 자산(DSToken)을 유동성(스테이블 코인)으로 상환하는 데 사용됩니다. `_amount` 매개변수는 호출자가 상환하고자 하는 자산의 양입니다.
```solidity
SecuritizeRedemption.sol
77:     function redeem(uint256 _amount) whenNotPaused external override {
78:         uint256 rate = navProvider.rate();
79:         require(rate != 0, "Rate should be defined");
80:         require(asset.balanceOf(msg.sender) >= _amount, "Redeemer has not enough balance");
81:         require(address(liquidityProvider) != address(0), "Liquidity provider should be defined");
82:         require(liquidityProvider.availableLiquidity() >= _amount, "Not enough liquidity");//@audit-issue WRONG
83:
84:         ERC20 stableCoin = ERC20(address(liquidityProvider.liquidityToken()));
85:         uint256 liquidity = _amount * rate / (10 ** stableCoin.decimals());
86:
87:         liquidityProvider.supplyTo(msg.sender, liquidity);
88:         asset.transferFrom(msg.sender, liquidityProvider.recipient(), _amount);
89:
90:         emit RedemptionCompleted(msg.sender, _amount, liquidity, rate);
91:     }
```
구현을 보면 `liquidityProvider`가 충분한 유동성을 가지고 있는지 확인하는 라인이 있습니다.
하지만 유동성이 아닌 자산의 양인 `_amount`라는 잘못된 값과 비교하고 있습니다.
대신 L85에서 계산된 `liquidity` 값과 비교해야 합니다.
이 잘못된 검사는 실제 `availableLiquidity()`가 필요한 양보다 적은데도 검사가 통과하거나(환율이 1보다 큰 경우), 충분한 유동성이 있는데도 검사가 실패하는(환율이 1보다 작은 경우) 상황으로 이어질 수 있습니다.
두 번째 경우가 첫 번째 경우보다 더 심각합니다.

**파급력:** 환율이 1보다 훨씬 작고 자산과 유동성 토큰의 소수점 자릿수에 차이가 있을 때 상환이 대부분 실패할 것이므로 고위험(HIGH)으로 평가합니다.

**권장 완화 조치:** 유동성 계산 후로 검증을 이동하고 올바른 값과 비교하도록 수정하십시오.

**Securitize:** 커밋 [3977ca](https://bitbucket.org/securitize_dev/bc-redemption-sc/commits/3977ca8ffb259a01e8dab894745751cf2150abf4)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `CollateralLiquidityProvider::setCollateralProvider`의 접근 제어 누락

**설명:** `CollateralLiquidityProvider` 계약은 담보를 가져오고 유동성 토큰으로 상환을 처리하기 위해 `collateralProvider`에 의존합니다. 이 상태 변수는 아래와 같이 `setCollateralProvider()` 함수를 통해 변경할 수 있습니다.
```solidity
CollateralLiquidityProvider.sol
79:     function setCollateralProvider(address _collateralProvider) external {//@audit-issue CRITICAL missing access control
80:         collateralProvider = _collateralProvider;
81:     }
```
구현을 보면 함수가 어떤 접근 제어로도 보호되지 않아 누구나 `collateralProvider`를 원하는 대로 변경할 수 있습니다.
악의적인 `collateralProvider`를 사용하면 `availableLiquidity` 함수가 항상 0을 반환하도록 만들 수 있습니다.

**파급력:** 공격자가 핵심 상태 변수인 `collateralProvider`를 원하는 대로 변경하여 영구적인 서비스 거부(DoS)를 포함한 다양한 치명적인 결과를 초래할 수 있으므로 고위험(HIGH)으로 평가합니다.

**권장 완화 조치:** `setCollateralProvider()` 함수에 `onlyOwner` 수정자(modifier)를 추가하십시오.

**Securitize:** 커밋 [3977ca](https://bitbucket.org/securitize_dev/bc-redemption-sc/commits/3977ca8ffb259a01e8dab894745751cf2150abf4)에서 수정되었습니다.

**Cyfrin:** 확인됨.




### `CollateralLiquidityProvider::setExternalCollateralRedemption`의 접근 제어 누락

**설명:** `CollateralLiquidityProvider` 계약은 유동성을 제공하기 위해 `externalCollateralRedemption`에 의존하며, 이 상태 변수는 아래와 같이 `setExternalCollateralRedemption()` 함수를 통해 변경할 수 있습니다.
```solidity
CollateralLiquidityProvider.sol
75:     function setExternalCollateralRedemption(address _externalCollateralRedemption) external override {//@audit-issue CRITICAL missing access control
76:         externalCollateralRedemption = IRedemption(_externalCollateralRedemption);
77:     }
```
구현을 보면 함수가 어떤 접근 제어로도 보호되지 않아 누구나 `externalCollateralRedemption`을 원하는 대로 변경할 수 있습니다.
악의적인 `externalCollateralRedemption`을 사용하면 핵심 함수인 `supplyTo`를 무용지물로 만들 수 있습니다.

**파급력:** 공격자가 핵심 상태 변수인 `externalCollateralRedemption`을 원하는 대로 변경하여 영구적인 서비스 거부(DoS)를 포함한 다양한 치명적인 결과를 초래할 수 있으므로 고위험(HIGH)으로 평가합니다.

**권장 완화 조치:** `setExternalCollateralRedemption()` 함수에 `onlyOwner` 수정자(modifier)를 추가하십시오.

**Securitize:** 커밋 [3977ca](https://bitbucket.org/securitize_dev/bc-redemption-sc/commits/3977ca8ffb259a01e8dab894745751cf2150abf4)에서 수정되었습니다.

**Cyfrin:** 확인됨.


\clearpage
## 중급 위험 (Medium Risk)


### `dsToken` 구매 시 `_investorWallet`에 대한 유효성 검사 없음

**설명:** `SecuritizeSwap.buy()` 함수에서 `swap()` 함수와 달리 `_investorWallet`이 `_senderInvestorId`의 지갑 목록에 이미 추가되었는지 확인하는 유효성 검사가 없습니다. 결과적으로 사용자는 추가되지 않은 지갑을 사용하여 `dsToken`을 구매할 수 있습니다.

다음 코드 스니펫은 `swap()` 함수 내의 지갑 유효성 검사 부분을 보여줍니다. 먼저 지갑이 이미 추가되었는지 확인합니다. 지갑이 추가되지 않은 경우 지갑을 추가합니다. 지갑이 이미 추가된 경우 지갑이 투자자의 소유인지 확인합니다. 그러나 `buy()` 함수에는 이러한 단계가 포함되어 있지 않습니다.

```solidity
    File: securitize_dev-securitize-swap-47704357c2da\contracts\swap\SecuritizeSwap.sol

88:         //Check if new wallet should be added
89:         string memory investorWithNewWallet = IDSRegistryService(dsToken.getDSService(DS_REGISTRY_SERVICE)).getInvestor(_newInvestorWallet);
90:         if(CommonUtils.isEmptyString(investorWithNewWallet)) {
91:             IDSRegistryService(dsToken.getDSService(DS_REGISTRY_SERVICE)).addWallet(_newInvestorWallet, _senderInvestorId);
92:         } else {
93:             require(CommonUtils.isEqualString(_senderInvestorId, investorWithNewWallet), "Wallet does not belong to investor");
94:         }
```

**파급력:** 사용자가 추가되지 않은 지갑을 사용하여 `dsToken`을 구매할 수 있습니다.

**권장 완화 조치:** `buy()` 함수에 지갑에 대한 유효성 검사를 포함해야 합니다.

**Securitize:** 커밋 [b09460](https://bitbucket.org/securitize_dev/securitize-swap/commits/b094604b341123a49c8abbd6e1c3d53d7c102f28)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 업그레이드 가능한 계약에 스토리지 갭(storage gap)이 없어 스토리지 슬롯 충돌이 발생할 수 있음

**설명:** 업그레이드 가능한 계약의 경우 "기존 배포와의 스토리지 호환성을 손상시키지 않으면서 향후에 새로운 상태 변수를 자유롭게 추가할 수 있도록"(OpenZeppelin 인용) 스토리지 갭이 있어야 합니다. 그렇지 않으면 새로운 구현 코드를 작성하기가 매우 어려울 수 있습니다. 스토리지 갭이 없으면 기본 계약에 새 변수가 추가될 경우 자식 계약의 변수가 업그레이드된 기본 계약에 의해 덮어씌워질 수 있습니다. 이는 자식 계약에 의도하지 않은 매우 심각한 결과를 초래하여 잠재적으로 사용자 자금 손실을 일으키거나 계약이 완전히 오작동하게 만들 수 있습니다.

이 문서의 하단 부분을 참조하십시오: https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable

**파급력:** 업그레이드 가능한 계약에 스토리지 갭이 없으면 스토리지 슬롯 충돌이 발생할 수 있습니다.

**개념 증명 (PoC):** 다음을 포함하여 코드 베이스의 여러 계약이 업그레이드 가능한 계약으로 의도되었습니다.

securitize_dev-bc-redemption-sc-32e23d5318be\contracts\utils\BaseContract.sol
securitize_dev-bc-nav-provider-sc-dce942a8a54a\contracts\utils\BaseContract.sol

그러나 이러한 계약 중 어느 것도 스토리지 갭을 포함하지 않습니다. 스토리지 갭은 "기존 배포와의 스토리지 호환성을 손상시키지 않으면서 향후에 새로운 상태 변수를 자유롭게 추가할 수 있게" 해주므로 업그레이드 가능한 계약에 필수적입니다.

**권장 완화 조치:** 업그레이드 가능한 계약의 끝에 다음과 같이 적절한 스토리지 갭을 추가하는 것이 좋습니다. OpenZeppelin 업그레이드 가능 계약 템플릿을 참조하십시오.

```solidity
uint256[50] private __gap;
```
**Securitize:** 커밋 [3977ca](https://bitbucket.org/securitize_dev/bc-redemption-sc/commits/3977ca8ffb259a01e8dab894745751cf2150abf4) 및 [334f49](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/334f4996bc79d0489122210ed54c5596bdcf4eb7)에서 수정되었습니다.


**Cyfrin:** 확인됨.



### 투자자가 자신의 논스(nonce)를 증가시킬 수 있는 기능 부재

**설명:** `SecuritizeSwap::executePreApprovedTransaction()` 함수는 유효한 서명을 제공하여 누구나 사전 승인된 트랜잭션을 실행할 수 있게 합니다.
이 함수는 내부 함수 `doExecuteByInvestor()`에 의존하며, 해시는 투자자의 논스를 포함한 다양한 속성을 기반으로 계산됩니다.
이 논스 메커니즘은 동일한 트랜잭션이 실행되는 것을 방지하기 위해 존재합니다.
일반적으로 논스 메커니즘은 무효화 메커니즘과 함께 제공되어 논스 소유자가 선제적으로 논스를 증가시켜 현재 논스를 무효화할 수 있게 합니다.
현재 설계는 논스를 증가시키는 방법을 제공하지 않으며, 이는 투자자가 이미 서명된 해시를 무효화할 수 없음을 의미합니다.

**파급력:** 투자자가 이미 서명된 해시를 무효화할 방법이 없습니다.

**권장 완화 조치:** 투자자가 자신의 논스를 증가시켜 이미 서명된 해시를 무효화할 수 있는 함수를 추가하십시오.

**Securitize:** 투자자가 자신의 논스를 수정할 수 없으므로 이 제안을 적용하지 않을 것입니다.
이는 계약과 백엔드를 통해서만 수행됩니다. 또한 NAV Rate 변경이 제어됩니다.

**Cyfrin:** 인지함.


### `SecuritizeSwap::executePreApprovedTransaction()` 함수에서 `msg.value` 검증 누락

**설명:** `SecuritizeSwap`은 `SecuritizeSwap::executePreApprovedTransaction` 함수를 사용하여 누구나 사전 승인된 트랜잭션을 실행할 수 있게 합니다. 이 함수는 프로토콜이 일시 중지되지 않았고 `params` 매개변수의 길이가 2인지 확인한 다음 `doExecuteByInvestor()`를 호출합니다.
```solidity
SecuritizeSwap.sol
191:         uint256 value = _params[0];
192:         uint256 gasLimit = _params[1];
193:         assembly {
194:             let ptr := add(_data, 0x20)
195:             let result := call(gasLimit, _destination, value, ptr, mload(_data), 0, 0)
196:             let size := returndatasize()
197:             returndatacopy(ptr, 0, size)
198:             switch result
199:             case 0 {
200:                 revert(ptr, size)
201:             }
202:             default {
203:                 return(ptr, size)
204:             }
205:         }
```
`doExecuteByInvestor`의 구현을 보면 `params[0]`이 `_destination`을 호출하는 데 값으로 사용됩니다.
그러나 `params[0]`이 실제 `msg.value`와 같은지 검증되지 않았으며 초과된 값을 환불하는 메커니즘도 없습니다.
이 문제로 인해 제공된 `msg.value`가 `params[0]`(서명된 해시에 포함됨)보다 크면 잔여 네이티브 토큰이 계약에 영구적으로 잠기게 됩니다.

**파급력:** 호출자가 과도한 `msg.value`로 호출할 가능성은 낮으므로 중간(MEDIUM) 위험으로 평가합니다.

**권장 완화 조치:** `msg.value`가 `params[0]`과 같도록 요구하거나 초과 금액을 호출자에게 반환하십시오.

**Securitize:** 커밋 [b09460](https://bitbucket.org/securitize_dev/securitize-swap/commits/b094604b341123a49c8abbd6e1c3d53d7c102f28)에서 함수를 수정했습니다. 이 경우 지불 불가능한(non-payable) 트랜잭션에 대해 OpenZeppelin `Address.functionCall(_destination, _data)` 함수를 사용했습니다. msg.value는 0입니다.
또한 `SecuritizeSwap::executePreApprovedTransaction()` 함수는 payable이 아닙니다.

**Cyfrin:** 확인됨.

\clearpage
## 저위험 (Low Risk)


### `SecuritizeSwap.calculateDsTokenAmount()` 함수에서의 정밀도 손실

**설명:** `stableCoinDecimals`가 `dsTokenDecimals`보다 큰 경우 계산은 두 부분으로 나뉩니다: `L242`에서 `(10 ** (stableCoinDecimals - dsTokenDecimals))`로 나눈 다음 `L245`에서 `10 ** stableCoinDecimals`를 곱합니다. 이로 인해 정밀도 손실이 발생할 수 있습니다.

```solidity
        if (stableCoinDecimals <= dsTokenDecimals) {
            adjustedStableCoinAmount = _stableCoinAmount * (10 ** (dsTokenDecimals - stableCoinDecimals));
        } else {
242         adjustedStableCoinAmount = _stableCoinAmount / (10 ** (stableCoinDecimals - dsTokenDecimals));
        }
        // The InternalNavSecuritizeImplementation uses rate expressed with same number of decimals as stableCoin
245     uint256 dsTokenAmount = adjustedStableCoinAmount * 10 ** stableCoinDecimals / currentNavRate;
```

**파급력:** 계산된 `dsTokenAmount`가 원래보다 적어 사용자 자금 손실이 발생합니다.

**개념 증명 (PoC):**

사실 올바른 공식은 다음과 같습니다.
```solidity
        dsTokenAmount = _stableCoinAmount * 10 ** dsTokenDecimals / currentNavRate;
```

다음 시나리오를 고려해 보겠습니다.
1. `stableCoinDecimals = 18`
2. `dsTokenDecimals = 6`
3. `currentNavRate = 1e12` (`stableCoin : dsToken = 1e6 : 1`을 의미)
4. `_stableCoinAmount = 1e18 - 1`

현재 로직은 다음과 같이 계산합니다.
- `L242`: `adjustedStableCoinAmount = (1e18 - 1) / (10 ** (18 - 6)) = 1e6 - 1`
- `L245`: `dsTokenAmount = (1e6 - 1) * 10 ** 18 / 1e12 = 1e12 - 1e6`

그러나 실제 올바른 `dsTokenAmount`는 `(1e18 - 1) * 1e6 / 1e12 = 1e12 - 1`이어야 하며, `1e6 - 1`의 차이가 발생합니다.

**권장 완화 조치:** 중간 변수 `adjustedStableCoinAmount`는 불필요합니다.

```diff
    function calculateDsTokenAmount(uint256 _stableCoinAmount) internal view returns (uint256, uint256) {
        uint256 stableCoinDecimals = ERC20(address(stableCoinToken)).decimals();
        uint256 dsTokenDecimals = ERC20(address(dsToken)).decimals();
        uint256 currentNavRate = navProvider.rate();

-       uint256 adjustedStableCoinAmount;
-       if (stableCoinDecimals <= dsTokenDecimals) {
-           adjustedStableCoinAmount = _stableCoinAmount * (10 ** (dsTokenDecimals - stableCoinDecimals));
-       } else {
-           adjustedStableCoinAmount = _stableCoinAmount / (10 ** (stableCoinDecimals - dsTokenDecimals));
-       }
-       // The InternalNavSecuritizeImplementation uses rate expressed with same number of decimals as stableCoin
-       uint256 dsTokenAmount = adjustedStableCoinAmount * 10 ** stableCoinDecimals / currentNavRate;

+       uint256 dsTokenAmount = _stableCoinAmount * 10 ** dsTokenDecimals / currentNavRate;

        return (dsTokenAmount, currentNavRate);
    }
```


**Securitize:** 커밋 [b09460](https://bitbucket.org/securitize_dev/securitize-swap/commits/b094604b341123a49c8abbd6e1c3d53d7c102f28)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `SecuritizeRedemption.updateLiquidityProvider` 함수가 잘못된 정보를 방출함

**설명:** `SecuritizeRedemption.updateLiquidityProvider` 함수는 `oldProvider`가 `liquidityProvider`로 업데이트되었다는 정보를 방출합니다.
하지만 이 함수는 L66에서 `liquidityProvider` 대신 `_liquidityProvider`를 `oldProvider`에 할당합니다.

`updateLiquidityProvider` 함수에서 `oldProvider`에 `liquidityProvider` 대신 새 변수 `_liquidityProvider`를 할당합니다.

```solidity
File: securitize_dev-bc-redemption-sc-32e23d5318be\contracts\redemption\SecuritizeRedemption.sol
65:     function updateLiquidityProvider(address _liquidityProvider) onlyOwner external override {
66:         address oldProvider = address(_liquidityProvider);
67:         liquidityProvider = ILiquidityProvider(_liquidityProvider);
68:         emit LiquidityProviderUpdated(oldProvider, address(liquidityProvider));
69:     }
```

L67에서 `liquidityProvider`도 `_liquidityProvider`와 같습니다.
L68에서 동일한 변수를 방출하므로 이는 잘못되었습니다.


**Securitize:** 커밋 [3977ca](https://bitbucket.org/securitize_dev/bc-redemption-sc/commits/3977ca8ffb259a01e8dab894745751cf2150abf4)에서 수정되었습니다.

**Cyfrin:** 확인됨.




### 다중 초기화(initializer)를 갖는 좋지 않은 관행

**설명:** `SecuritizeSwap` 계약에 `initialize()`라는 이름의 함수가 두 개 있음을 발견했습니다.
두 함수 모두 공개(public)이며 `initializer` 및 `forceInitializeFromProxy` 수정자 뒤에 있고, 메인 초기화 함수(테스트 스위트가 일반적인 사용 사례를 보여준다고 가정할 때)는 4개의 매개변수를 가진 함수이며 이것이 다른 하나를 호출합니다.
이는 좋은 관행이 아니며 수정을 강력히 권장합니다.
3개의 매개변수를 가진 `initialize()` 함수를 적절한 이름(아마도 `_initialize()`)을 가진 내부(internal) 함수로 변경하고 수정자는 메인 공개 초기화 함수에만 적용할 것을 권장합니다.

**Securitize:** 커밋 [b09460](https://bitbucket.org/securitize_dev/securitize-swap/commits/b094604b341123a49c8abbd6e1c3d53d7c102f28)에서 수정되었습니다.

**Cyfrin:** 확인됨.




### 구매 시 투자자를 위한 슬리피지(slippage) 제어 누락

**설명:** 프로토콜은 투자자가 `SecuritizeSwap::buy()` 함수를 사용하여 `stableCoinToken`으로 `dsToken`을 구매할 수 있도록 합니다.
이 함수는 `navProvider.rate()`를 기반으로 호출자에게 보낼 `dsToken`의 양을 계산하며, 투자자가 자신의 구매에 어떤 환율이 적용될지 예측할 방법이 없습니다.

```solidity
SecuritizeSwap.sol
104:     function buy(
105:         string memory _senderInvestorId,
106:         address _investorWallet,
107:         uint256 _stableCoinAmount,
108:         uint256 _blockLimit,
109:         uint256 _issuanceTime
110:     ) public override whenNotPaused {
111:         require(_blockLimit >= block.number, "Transaction too old");//@audit-ok
112:         require(stableCoinToken.balanceOf(_investorWallet) >= _stableCoinAmount, "Not enough stable tokens balance");
113:         require(IDSRegistryService(dsToken.getDSService(DS_REGISTRY_SERVICE)).isInvestor(_senderInvestorId), "Investor not registered");
114:         require(navProvider.rate() > 0, "NAV Rate must be greater than 0");
115:         (uint256 dsTokenAmount, uint256 currentNavRate) = calculateDsTokenAmount(_stableCoinAmount);
116:         stableCoinToken.transferFrom(_investorWallet, issuerWallet, _stableCoinAmount);
117:
118:         dsToken.issueTokensCustom(_investorWallet, dsTokenAmount, _issuanceTime, 0, "", 0);//@audit-issue missing protection against change in the NAV rate
119:
120:         emit Buy(msg.sender, _stableCoinAmount, dsTokenAmount, currentNavRate, _investorWallet);
121:     }
122:
```
어떤 이유로든 특정 시간대 동안 `navProvider.rate()`가 변경되면, 투자자는 환율 변경으로 인한 손실을 감수할 수밖에 없습니다.

**권장 완화 조치:** 함수에 `_minDsTokenAmount` 매개변수를 추가하여 호출자가 예상되는 최소 금액을 지정할 수 있도록 하십시오.
또한 `calculateDsTokenAmount()` 함수를 공개(public)로 만들어 사용자가 수령할 금액을 미리 예상할 수 있도록 하는 것이 좋습니다.

**Securitize:** 커밋 [b09460](https://bitbucket.org/securitize_dev/securitize-swap/commits/b094604b341123a49c8abbd6e1c3d53d7c102f28)에서 수정되었습니다.
`dsTokenAmount`는 고정되고 유동성 토큰이 NAV 환율에 따라 변동되도록 구매 함수 정의를 수정했습니다.
또한 급격한 변동으로 인한 가격 변화를 방지하기 위해 `_maxStableCoinAmount` 매개변수를 추가했습니다.

**Cyfrin:** 확인됨.


### `ecrecover()` 결과에 대한 0 확인 누락

**설명:** `SecuritizeSwap::doExecuteByInvestor()` 함수는 제공된 서명이 실제로 `ROLE_ISSUER` 또는 `ROLE_MASTER` 역할을 가진 행위자에 의해 서명되었는지 검증합니다.
검증 과정에서 `ecrecover()`가 사용되지만 결과가 0이 아닌지 검증하지 않습니다.
다음 라인이 `recovered`의 역할을 검증한다는 것은 이해되지만, 향후 예상치 못한 문제를 위해 이 확인을 구현하는 것이 강력히 권장됩니다.



**Securitize:** 커밋 [b09460](https://bitbucket.org/securitize_dev/securitize-swap/commits/b094604b341123a49c8abbd6e1c3d53d7c102f28)에서 수정되었습니다.

**Cyfrin:** 확인됨.



### `CollateralLiquidityProvider::setExternalCollateralRedemption`에서 입력 검증 누락

**설명:** _이 발견 사항은 접근 제어에 대한 다른 발견 사항이 완화되었다고 가정합니다_.
`CollateralLiquidityProvider` 계약은 상환을 처리하기 위해 `externalCollateralRedemption`에 의존합니다.
`supplyTo()` 함수는 이 계약의 핵심 함수입니다.
```solidity
CollateralLiquidityProvider.sol
61:     function supplyTo(address _redeemer, uint256 _amount) whenNotPaused onlySecuritizeRedemption public override {
62:         //take collateral funds from collateral provider
63:         IERC20(externalCollateralRedemption.asset()).transferFrom(collateralProvider, address(this), _amount);
64:
65:         //approve external redemption
66:         IERC20(externalCollateralRedemption.asset()).approve(address(externalCollateralRedemption), _amount);
67:
68:         //get liquidity
69:         externalCollateralRedemption.redeem(_amount);
70:
71:         //supply _redeemer
72:         liquidityToken.transfer(_redeemer, _amount);
73:     }
```
L72와 함께 L69를 보면, 함수는 `liquidityToken = externalCollateralRedemption.liquidity`라고 가정하고 있습니다.
그러나 이것은 `setExternalCollateralRedemption` 함수에서 검증되지 않으며 불일치를 유발할 위험이 있습니다.

**파급력:** 다른 발견 사항을 완화하는 적절한 접근 제어에 의해 함수가 보호된다고 가정할 때 파급력을 낮음(LOW)으로 평가합니다.

**권장 완화 조치:** `setExternalCollateralRedemption()` 함수에서 `liquidityToken = externalCollateralRedemption.liquidity`임을 검증하십시오.

**Securitize:** 커밋 [3977ca](https://bitbucket.org/securitize_dev/bc-redemption-sc/commits/3977ca8ffb259a01e8dab894745751cf2150abf4)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)


### 일부 이벤트에 대해 잘못된 값 방출

**설명:** `SecuritizeSwap::swap()` 함수에서 성공적인 스왑 후 이벤트가 방출됩니다.
하지만 이벤트는 `from` 값의 자리에 `msg.sender`를 방출하고 있는데, `swap()` 함수의 `msg.sender`는 issuer 또는 master뿐입니다.
```solidity
SecuritizeSwap.sol
101: emit Swap(msg.sender, _valueDsToken, _valueStableCoin, _newInvestorWallet);
```
**Securitize:** 수정하지 않음. Buy 이벤트의 첫 번째 인수는 `address indexed _from`이며, 이 경우 `msg.sender`가 맞습니다.

**Cyfrin:** 인지함.


### `override` 키워드의 일관성 없는 사용

**설명:** 여러 곳에서 `override` 키워드가 일관성 없이 사용되고 있음을 발견했습니다.
`override` 키워드를 일관되고 합리적으로 사용할 것을 권장합니다.

```solidity
CollateralLiquidityProvider.sol
75:     function setExternalCollateralRedemption(address _externalCollateralRedemption) external override { //@audit override keyword
76:         externalCollateralRedemption = IRedemption(_externalCollateralRedemption);
77:     }
78:
79:     function setCollateralProvider(address _collateralProvider) external { //@audit no override keyword
80:         collateralProvider = _collateralProvider;
81:     }
```

**Securitize:** 커밋 [b09460](https://bitbucket.org/securitize_dev/securitize-swap/commits/b094604b341123a49c8abbd6e1c3d53d7c102f28) 및 [334f49](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/334f4996bc79d0489122210ed54c5596bdcf4eb7)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 잘못된/오해의 소지가 있는 주석

**설명:** 여러 위치에서 주석이 잘못되었거나 오해의 소지가 있습니다. 이러한 잘못된/오해의 소지가 있는 주석은 계약에 대한 예상치 못한 해석과 잠재적인 보안 문제로 이어질 수 있습니다. 이러한 잘못된 또는 오해의 소지가 있는 주석을 수정할 것을 권장합니다.

```solidity
ISecuritizeNavProvider.sol
6:  * @dev Defines a common interface to get NAV (Native Asset Value) Rate to //@audit Incomplete comment

SecuritizeInternalNavProvider.sol
9:      * @dev rate: NAV rate expressed with 6 decimals//@audit-issue INFO incomplete comment
14:     * @dev Throws if called by any account other than the owner.//@audit-issue INFO wrong comment

SecuritizeRedemption.sol
26: * @dev Emitted when the Settlement address changes//@audit-issue INFO wrong comment

BaseSecuritizeSwap.sol
98: * @param sigR R signature//@audit-issue INFO wrong comment, should be sigS

BaseSecuritizeSwap.sol
102: * @param params array of params. params[0] = value, params[1] = gasLimit, params[2] = blockLimit//@audit-issue INFO wrong comment, params length is two

SecuritizeSwap.sol
131: * @param sigR R signature//@audit-issue INFO wrong comment, should be sigS

IDSToken.sol
16: * @param _cap address The address which is going to receive the newly issued tokens//@audit-issue INFO wrong comment
```

**Securitize:** 커밋 [3977ca](https://bitbucket.org/securitize_dev/bc-redemption-sc/commits/3977ca8ffb259a01e8dab894745751cf2150abf4), [b09460](https://bitbucket.org/securitize_dev/securitize-swap/commits/b094604b341123a49c8abbd6e1c3d53d7c102f28), 및 [334f49](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/334f4996bc79d0489122210ed54c5596bdcf4eb7)에서 수정되었습니다.

**Cyfrin:** 확인됨.


\clearpage

