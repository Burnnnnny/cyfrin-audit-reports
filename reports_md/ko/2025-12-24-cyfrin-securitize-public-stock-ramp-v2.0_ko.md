**Lead Auditors**

[Dacian](https://x.com/DevDacian)

[Jorge](https://x.com/TamayoNft)

**Assisting Auditors**

[0ximmeas](https://x.com/0ximmeas)

[Stalin](https://x.com/0xStalin)

---

# 발견 사항 (Findings)
## 중간 위험 (Medium Risk)


### `SecuritizeAmmNavProvider`의 중요한 상태 변경 함수에 `whenNotPaused` modifier가 없음

**설명:** `SecuritizeAmmNavProvider`는 `PausableUpgradeable`을 상속하는 `BaseContract`를 상속하지만, 주요 상태 변경 함수들에 `whenNotPaused`가 적용되어 있지 않습니다.

**영향:** `SecuritizeAmmNavProvider`를 pause해도 실제로는 상태 변경 함수가 계속 호출될 수 있어 pause가 무력화됩니다. 다른 NAV provider 구현을 보면 의도된 동작으로 보이지 않습니다.

**권장 완화 조치:** `resetBaseline`, `setPriceScaleFactor`, `executeBuyBase`, `executeSellBase` 같은 주요 상태 변경 함수에 `whenNotPaused`를 추가하십시오.

**Securitize:** [f09cb9a](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/f09cb9a621b2fee4890d4cca7b952ec0398a6d1e)에서 수정됨.

**Cyfrin:** 확인함.


### `PublicStockOnRamp::swap`에서 `investorExists` modifier를 잘못 사용함

**설명:** `PublicStockOnRamp::swap`은 투자자 등록 여부를 잘못 검증합니다. 이 함수는 `investorExists` modifier를 사용하지만, 동시에 `onlyRole(OPERATOR_ROLE)`도 사용하므로 `msg.sender`는 항상 operator입니다. 실제 투자자는 `_investorWallet`인데 이 주소는 registry에 대해 검증되지 않습니다.

**영향:**
- operator 자신이 유효한 investor wallet을 가진 경우, 등록되지 않은 투자자를 위해 swap을 실행할 수 있어 investor registration 시스템을 우회할 수 있습니다.
- 반대로 operator가 유효한 investor wallet이 아니면 `swap`이 revert되어 서비스 거부가 발생합니다.

**권장 완화 조치:** `investorExists`를 없애고, `msg.sender` 대신 `_investorWallet` 파라미터를 직접 검증하는 inline check 또는 새 modifier를 사용하십시오.

**Securitize:** [090cd62](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/090cd62fd656fb0aaf969de1bcc0a84db5523581)에서 수정됨.

**Cyfrin:** 확인함.


### 반올림으로 `curvePriceWad`가 0이 되면 잘못된 가격 산정 또는 DoS가 발생할 수 있음

**설명:** `SecuritizeAmmNavProvider::_curveBuy`와 `_curveSell`에서 reserve 불균형이 심해지면 `curvePriceWad`가 정수 나눗셈 과정에서 0으로 내려갈 수 있습니다. 이후 `_pricingFromCurveBuy/_pricingFromCurveSell`에 이 값이 들어가면:
- `priceScaleFactor >= 2`일 때는 사용자가 anchor price 대비 과도하게 유리한 가격으로 거래하게 되고,
- `priceScaleFactor == 1`일 때는 0으로 나누기가 발생해 해당 방향의 거래가 영구적으로 막힐 수 있습니다.

**영향:** 설정값에 따라 두 가지 실패 모드가 생깁니다.
- 잘못된 가격으로 거래가 체결되어 유동성 제공자나 프로토콜이 손해를 봄
- 특정 거래 방향이 division by zero로 영구 DoS에 빠짐

**권장 완화 조치:** `_curveBuy`와 `_curveSell` 모두에서 `curvePriceWad > 0`을 강제하십시오. 또한 `priceScaleFactor`도 최소 2 이상이 되도록 제한하는 것이 안전합니다.

**Securitize:** [bcd6e87](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/bcd6e87a866f83ed33b468b9dd2acebac9eca1fd)에서 수정됨.

**Cyfrin:** 확인함.


### `SecuritizeOnRamp::swap`과 `SecuritizeOffRamp::redeem`이 investor 주소 대신 operator를 전달해 DoS가 발생함

**설명:** 두 함수 모두 `onlyRole(OPERATOR_ROLE)`를 사용합니다. 그런데 내부적으로 `BaseOnRamp::_swap` 또는 `BaseOffRamp::_redeem` 같은 하위 함수로 내려갈 때 `_msgSender()`를 investor wallet로 넘깁니다. operator와 investor는 다른 주체이므로 이 값은 잘못된 주소입니다.

**영향:** 가장 가능성이 높은 결과는 일시적인 서비스 거부입니다. operator가 investor가 아니므로 호출이 revert됩니다. 업그레이드 가능한 컨트랙트라 수정은 가능하지만, 그 전까지는 핵심 흐름이 막힐 수 있습니다.

**권장 완화 조치:** 이 함수들이 투자자가 직접 호출하는 함수라면 `onlyRole(OPERATOR_ROLE)`을 제거하십시오. 반대로 operator가 호출해야 한다면 investor 주소를 입력받는 별도 파라미터를 두어야 합니다.

**Securitize:** [5cbd6d8](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/5cbd6d82e9bff57c6441334897e8ef0c9c594825)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 낮은 위험 (Low Risk)


### `RedStoneNavProvider::rate`는 oracle이 음수 값을 반환하면 비정상적으로 큰 값을 돌려줄 수 있음

**설명:** `RedStoneNavProvider::rate`는 `int256` oracle 값을 받은 뒤 이를 `uint256`으로 강제 변환합니다. 음수 값을 검증하지 않기 때문에, 음수 oracle 값이 거대한 양수로 변환될 수 있습니다.

**영향:** NAV 계산이 비정상적으로 커지고, downstream 프로토콜이 잘못된 가격을 사용하게 됩니다.

**권장 완화 조치:** oracle 값이 0이 아니라 `0보다 큰 값`인지 강제하고, 필요하다면 stale check와 min/max threshold 같은 일반적인 oracle 검증도 추가하십시오.

**Securitize:** [ec23faf](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/ec23faf7f773b7a42229cb5f7f6ae3dd51a07772)에서 수정됨.

**Cyfrin:** 확인함.


### `SecuritizeAmmNavProvider`의 quote 함수는 baseline reset 동작을 반영하지 않음

**설명:** `quoteBuyBase`와 `quoteSellBase`는 실제 체결 시 `executeBuyBase`/`executeSellBase`에서 수행되는 `_checkAndResetBaseline` 로직을 시뮬레이션하지 않습니다.

**영향:** quote가 실제 실행 가격과 달라질 수 있어, 오프체인 통합이 잘못된 가격을 사용자에게 보여줄 수 있습니다.

**권장 완화 조치:** 이 제한을 명확히 문서화하거나, quote 함수에도 baseline reset 시뮬레이션을 추가하십시오.

**Securitize:** [71e40b8](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/71e40b8fd5387aa8382cb85fe0de0c89ad399125), [d05482b](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/d05482b2d42a2c9df7451affb634444e5f545f51)에서 수정됨.

**Cyfrin:** 확인함.


### `RedStoneNavProvider`에는 price feed를 갱신하는 함수가 없음

**설명:** `RedStoneNavProvider`는 초기화 시 price feed 주소를 설정하지만, 이후 이를 변경할 수단이 없습니다. feed 주소 변경이 필요하면 새 컨트랙트를 배포하고 모든 OnRamp/OffRamp에서 provider를 갱신해야 합니다.

**영향:** 간단한 파라미터 변경으로 해결할 문제도 전체 컨트랙트 재배포와 운영 절차를 필요로 하게 됩니다.

**권장 완화 조치:** admin이 price feed 주소를 갱신할 수 있는 함수를 추가하십시오.

**Securitize:** [4146a77](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/4146a77b4e5d3a72e67f164ef6e1ca3a12c99657)에서 수정됨.

**Cyfrin:** 확인함.


### `PublicStockOnRamp` / `PublicStockOffRamp` 서명에는 투자자 nonce와 deadline이 없어 operator가 여러 번 재사용할 수 있음

**설명:** 두 컨트랙트의 EIP-712 서명 구조에는 deadline/expiration과 nonce가 포함되어 있지 않습니다. 따라서 서명은 만료 없이 유효하고, operator가 같은 서명을 여러 번 재사용할 수도 있습니다.

**영향:** 투자자가 한 번 서명한 거래는 시장 상황이 바뀌어도 취소할 수 없고, operator는 수일 또는 수주 뒤에 불리한 시점에 실행할 수 있습니다. nonce가 없으므로 동일 서명 재사용도 가능합니다.

**권장 완화 조치:** 서명 메시지에 deadline과 nonce를 포함시키고, `NoncesUpgradeable`과 유사한 방식으로 이를 관리하십시오.

**Securitize:** [85142ed](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/85142edf57f24a4af30f2e46382188adac60fbc2)에서 수정됨.

**Cyfrin:** 확인함.


### `SecuritizeAmmNavProvider`의 virtual reserve 반올림 침식으로 DoS가 발생할 수 있음

**설명:** constant-product 가상 AMM 수학은 매 거래마다 정수 나눗셈 truncation 때문에 `k`를 조금씩 잃습니다. 이후 `_checkAndResetBaseline`가 depleted reserve를 기준으로 baseline을 새로 잡으면 이 현상이 가속됩니다. 결국 reserve 중 하나가 0으로 떨어져, `initialized` modifier 때문에 모든 거래가 막힐 수 있습니다.

**영향:** admin이 `resetBaseline`으로 수동 복구할 때까지 전체 거래가 멈춥니다.

**권장 완화 조치:** reserve가 위험 구간으로 떨어지지 않도록 최소 reserve threshold를 두고, 초기화/리셋/매매 경로 모두에서 이 하한을 강제하십시오.

**Securitize:** [1919a89](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/1919a8993ec7e0cd9f1931b4bb02ec622321c0fe)에서 수정됨.

**Cyfrin:** 확인함.


### `SecuritizeAmmNavProvider`의 quote/buy 함수에 zero output 검증이 없음

**설명:** `quoteBuyBase`, `quoteSellBase`, `executeBuyBase`, `executeSellBase`는 출력값이 0인지 검증하지 않습니다. 작은 거래량, 극단적 가격 불균형, low-decimal asset 조합에서는 `baseOut`, `quoteOut`, `execPrice`가 0이 될 수 있습니다.

**영향:**
- 사용자가 실제 토큰을 보내고도 아무것도 받지 못하는 조용한 손실 가능성
- 상태는 업데이트됐지만 출력은 0인 비정상 거래
- 오프체인 quote가 0을 반환해 잘못된 기대를 유발

**권장 완화 조치:** 네 함수 모두에서 `baseOut`, `quoteOut`, `execPrice`가 0이 아니도록 검증하십시오.

**Securitize:** [affb350](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/affb35097f9b638d6b1cfe4f58b42fdf79bc8778)에서 수정됨.

**Cyfrin:** 확인함.


### `RedStoneNavProvider::rate`는 정규화 후 값이 0이 될 수 있음

**설명:** 현재 zero check는 raw oracle 값에만 적용됩니다. `Helper::normalizeRate`에서 decimals 차이 때문에 나눗셈이 일어나면, raw value는 0이 아니어도 normalized result가 0으로 떨어질 수 있습니다.

**영향:** downstream 프로토콜은 자산 가치가 0인 것처럼 계산할 수 있고, division-by-zero 또는 잘못된 가격 계산으로 이어질 수 있습니다.

**권장 완화 조치:** normalized result에도 `> 0` 검사를 추가하십시오.

**Securitize:** [f4bed90](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/f4bed908104433d41035215a5315718dcc5669a9)에서 수정됨.

**Cyfrin:** 확인함.


### 상속되는 업그레이드형 컨트랙트는 ERC7201 namespaced storage 또는 storage gap을 사용해야 함

**설명:** 업그레이드형 베이스 컨트랙트를 다른 컨트랙트가 상속하는 구조에서는, 업그레이드 중 storage collision을 막기 위해 ERC7201 namespaced storage 또는 storage gap이 필요합니다.

**영향:** 이런 보호 장치가 없으면 업그레이드 시 storage collision이 발생할 수 있습니다.

**권장 완화 조치:** 이상적으로는 모든 업그레이드형 컨트랙트가 ERC7201 namespaced storage를 사용하도록 정리하십시오.

**Securitize:** [1656d74](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/1656d7494ff4129a1c77dc7d46a58a5999e9b9c4)에서 수정됨.

**Cyfrin:** 확인함.


### `SecuritizeOnRamp`에는 투자자가 자신의 nonce를 무효화하는 수단이 없음

**설명:** 투자자는 승인 서명 후 마음이 바뀌더라도 nonce를 직접 증가시키거나 무효화할 수 없습니다. nonce는 `executePreApprovedTransaction`이 성공할 때만 증가합니다.

**영향:** operator는 투자자가 더 이상 원하지 않는 승인도 계속 실행할 수 있습니다.

**권장 완화 조치:** OZ `NoncesUpgradeable`의 `_useNonce`를 노출하는 방식처럼, 사용자가 자기 nonce를 직접 무효화할 수 있게 하십시오.

**Securitize:** 현재는 이 기능을 노출하고 싶지 않다고 인지함.


### `SecuritizeAmmNavProvider::executeBuyBase`는 `execPrice`를 scale down할 때 사용자에게 유리하게 반올림함

**설명:** WAD 정밀도 가격을 자산의 native decimals로 내릴 때 floor division을 사용합니다. buy 경로에서는 가격을 낮게 반올림할수록 사용자가 더 적게 지불하게 되어, 모든 거래에서 buyer 쪽으로 유리하게 누적됩니다.

**영향:** 거래당 손실은 작아도 시간이 지나면 프로토콜이 지속적으로 가치 누수를 겪습니다.

**권장 완화 조치:** `executeBuyBase`의 최종 `execPrice`는 올림(round up) 처리하고, 필요하다면 `Math::mulDiv` 같은 명시적 반올림 유틸을 사용하십시오.

**Securitize:** [0b268e8](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/0b268e8282cfcacf4cbcd32e4b908a87af04b0e8)에서 수정됨.

**Cyfrin:** 확인함.


### DS Token buy 시 rate 반올림 방향이 사용자에게 유리함

**설명:** `Helper::normalizeRate`는 항상 내림 반올림합니다. `SecuritizeOnRamp::calculateDsTokenAmount`는 이 rate를 분모에 사용하므로, 반올림으로 rate가 작아질수록 사용자는 받아야 할 것보다 약간 더 많은 DSToken을 받게 됩니다.

**영향:** 사용자에게 미세하지만 일관되게 더 많은 DSToken이 발행됩니다.

**권장 완화 조치:** `ISecuritizeNavProvider::rate`와 `Helper::normalizeRate`에 반올림 방향을 명시할 수 있도록 확장하고, 분기점에서 `Math::mulDiv` 같은 유틸을 사용해 의도한 방향으로 반올림하십시오.

**Securitize:** 인터페이스 변경은 원치 않아 인지 상태로 남김.


### block number 기반 deadline은 체인별로 신뢰성이 다름

**설명:** `SecuritizeOnRamp::subscribe`는 `block.number`를 deadline으로 사용합니다. 하지만 이 프로토콜은 Ethereum L1뿐 아니라 Arbitrum, Optimism 등 블록 생성 속도가 크게 다른 체인에도 배포될 예정입니다.

**영향:** 같은 `_blockLimit` 값이라도 체인마다 유효 시간이 크게 달라집니다. L2에서는 사용자가 예상보다 훨씬 빨리 `TransactionTooOldError`에 걸릴 수 있습니다.

**권장 완화 조치:** 체인 독립적인 deadline을 위해 `block.number` 대신 `block.timestamp`를 사용하십시오.

**Securitize:** 실제 운영에서는 백엔드가 체인별로 block span을 조정하므로 현재 변경 계획은 없다고 인지함.


### `SecuritizeAmmNavProvider`는 AMM 핵심 불변식인 "`k`는 감소하면 안 된다"를 위반함

**설명:** `_curveBuy`와 `_curveSell` 모두 새 reserve 계산에서 내림 반올림을 사용합니다. 그 결과 사용자는 수학적으로 맞는 값보다 조금 더 많은 출력을 받고, `k`는 시간이 지날수록 감소합니다.

**영향:** 거래당 value leakage는 작지만, 프로토콜 방향으로 반올림해야 한다는 AMM 설계 원칙과 어긋납니다.

**권장 완화 조치:** `Math::ceilDiv` 등 올림 반올림을 사용해 `k`가 감소하지 않도록 하십시오.

**Securitize:** [04d2392](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/04d2392c4944aead08bf7a4793fd57e625918910)에서 수정됨.

**Cyfrin:** 확인함.


### pool price와 anchor price가 다를 때 `SecuritizeAmmNavProvider` 거래는 value leakage를 일으킬 수 있음

**설명:** smoothing formula의 비대칭성과 정수 truncation 때문에, `CLOSED_MARKET` 상태에서 같은 `anchorPriceWad`로 BUY한 뒤 즉시 SELL하면 처음보다 더 많은 quote를 되돌려받을 수 있습니다.

**영향:** 공격 규모는 작지만 virtual pool에서 가치가 천천히 빠져나갈 수 있습니다.

**권장 완화 조치:** smoothing formula를 보다 대칭적으로 다시 설계해 round trip 이득이 발생하지 않도록 하십시오.

**Securitize:** [`0b268e8`](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/0b268e8282cfcacf4cbcd32e4b908a87af04b0e8), [`7d7a4cf`](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/7d7a4cf2de1eaff5633ea9ea2a6c12b9a2f8be3f)에서 수식을 조정함.

**Cyfrin:** 부분 확인함. 여전히 round trip으로 이익이 날 수 있지만, 가스와 프로토콜 fee를 고려하면 실제 공격 벡터로는 비실용적입니다.

\clearpage
## 정보성 (Informational)


### mapping key/value의 의미가 드러나도록 named mapping parameter를 사용하기

**설명:** 일부 mapping은 이름이 있는 key/value를 사용하지만 일부는 그렇지 않습니다. 의미가 분명하도록 named mapping parameter를 일관되게 적용하는 것이 좋습니다.

**Securitize:** [ef5367e](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/ef5367e1db99cb7c72239f5702c817390d23236c)에서 수정됨.

**Cyfrin:** 확인함.


### `PublicStockOffRamp`, `SecuritizeOffRamp::redeem`에도 최소 상환 금액을 강제하는 것이 좋음

**설명:** `PublicStockOnRamp`는 dust trade를 막기 위해 최소 구독 금액 검사를 수행하지만, 두 offramp의 `redeem`에는 동일한 `_assetAmount` 검증이 없습니다.

**영향:** 아주 작은 redemption 요청도 허용되어 operator 가스를 낭비시키거나 rounding 이슈를 유발할 수 있습니다.

**권장 완화 조치:** on-ramp와 같은 패턴으로 최소 redemption amount 검사를 추가하십시오.

**Securitize:** 인지함.


### `CollateralLiquidityProvider::initialize`에서 `liquidityToken` 검증이 빠져 있음

**설명:** `setExternalCollateralRedemption`는 새 external collateral redemption의 liquidity token이 현재 `liquidityToken`과 일치하는지 확인하지만, `initialize`는 같은 검증 없이 값을 설정합니다.

**영향:** 배포 시 admin이 다른 liquidity token을 쓰는 외부 redemption 컨트랙트를 실수로 넣을 수 있습니다.

**권장 완화 조치:** `CollateralLiquidityProvider::initialize`에도 동일한 liquidity token 검증을 추가하십시오.

**Securitize:** [d8fd4fb](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/d8fd4fb9c38ed4d006df7ace366d30de239d6d4a)에서 수정됨.

**Cyfrin:** 확인함.


### offramp 로직에서 liquidity provider recipient를 바꿀 방법이 없음

**설명:** `AllowanceLiquidityProvider`와 `CollateralLiquidityProvider`는 recipient 주소를 초기화 시에만 설정하며, 이후 갱신 함수가 없습니다.

**영향:** recipient 지갑이 손상되거나 운영상 교체가 필요해도, 전체 컨트랙트 업그레이드 없이는 바꿀 수 없습니다.

**권장 완화 조치:** 두 provider 모두에 `setRecipient` 함수를 추가하십시오.

**Securitize:** 인지함.


### single step redemption과 two step redemption의 로직이 동등하지 않음

**설명:** `RedemptionManager::executeSingleStepRedemption`은 `CollateralLiquidityProvider`와 함께 쓸 때 여러 문제가 있습니다.
- fee를 실제 공급량이 아니라 요청량 기준으로 계산함
- fee 지급 경로의 `supplyTo` 반환값을 무시함
- 외부 redemption fee를 두 번 유발할 수 있음

**영향:** fee 회계가 부정확해지고, two-step 경로보다 비효율적이며 더 많은 외부 비용이 발생할 수 있습니다.

**권장 완화 조치:** single step redemption과 two step redemption의 동작을 일치시키는 방향으로 정리하십시오.

**Securitize:** 인지함.


### `liquidityToken` 처리에 표준 IERC20 대신 `SafeERC20`을 사용해야 함

**설명:** 온램프/오프램프는 외부 liquidity token(예: 스테이블코인)과 상호작용하므로, 다양한 토큰 구현과의 호환성을 위해 `SafeERC20::forceApprove`, `safeTransfer`, `safeTransferFrom` 등을 사용하는 것이 안전합니다.

**영향:** 비표준 ERC20 구현과 상호작용할 때 예기치 않은 실패 가능성을 줄일 수 있습니다.

**권장 완화 조치:** 관련 전송/승인 로직을 `SafeERC20` 기반으로 교체하십시오.

**Securitize:** [a694dc3](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/a694dc32386e038f8541ef79155b7a06a905fc52)에서 수정됨.

**Cyfrin:** 확인함.


### `ExecutePreApprovedTransaction` struct의 nonce 필드는 사용되지 않음

**설명:** struct에는 nonce 필드가 있지만 실제 해시 계산에서는 이 입력값을 전혀 사용하지 않고, 내부 `noncePerInvestor` 매핑 값만 씁니다.

**영향:** 호출자는 의미 없는 nonce를 입력해야 하고, API도 혼란스러워집니다.

**권장 완화 조치:** struct에서 사용되지 않는 nonce 필드를 제거하십시오.

**Securitize:** 백엔드 변경 부담 때문에 현재는 인지 상태로 유지.


### `SecuritizeInternalNavProvider::addRateUpdater/removeRateUpdater`에는 중복된 접근 제어 체크가 있음

**설명:** 함수 자체가 `onlyRole(DEFAULT_ADMIN_ROLE)`로 보호되는데, 내부에서 다시 `grantRole/revokeRole`를 호출해 동일한 권한 검사를 한 번 더 수행합니다.

**영향:** 기능상 문제는 없지만 불필요한 가스와 중복 검사가 발생합니다.

**권장 완화 조치:** 이미 modifier로 접근 제어가 보장되므로 `_grantRole`, `_revokeRole`를 직접 사용하십시오.

**Securitize:** [77a9a52](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/77a9a5295ca55f3d62df3ecc8517217598bf4deb)에서 수정됨.

**Cyfrin:** 확인함.


### `_pricingFromCurveBuy/_pricingFromCurveSell`의 중복 코드를 내부 함수로 리팩터링하기

**설명:** 두 함수는 마지막 출력 계산 한 줄을 제외하면 거의 동일합니다.

**영향:** 직접적인 취약점은 아니지만 코드 중복이 유지보수 비용을 높입니다.

**권장 완화 조치:** 공통 로직을 `_computeExecPrice` 같은 내부 함수로 추출하십시오.

**Securitize:** 인지함.


### `MbpsFeeManager::setFeePercentageMBPS`는 최대치를 넘는 fee 설정을 허용함

**설명:** 현재 구현에는 fee 상한 검사가 없어, 관리자가 실수로 과도한 fee를 설정할 수 있습니다.

**영향:** 과도한 fee 설정으로 시스템 경제성이 훼손될 수 있습니다.

**권장 완화 조치:** 합리적인 최대 fee 상수를 두고 그 이하만 허용하십시오.

**Securitize:** 인지함.

\clearpage
## 가스 최적화 (Gas Optimization)


### 선언 순서를 바꿔 storage packing 개선

**설명:** 예를 들어 `SecuritizeAmmNavProvider`에서 `lastMarketStatus`를 `asset` 바로 뒤에 두면 packing이 개선됩니다.

**Securitize:** [41815fa](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/41815fa7c45fe54ed46ed3d87aa81ea1a57af3fb)에서 수정됨.

**Cyfrin:** 확인함.


### 이벤트를 먼저 emit해 이전 값 저장용 로컬 변수를 제거하기

**설명:** 상태 변경 전후 값을 기록하기 위해 별도 로컬 변수를 만들지 않고, 현재 storage 값을 이용해 먼저 이벤트를 emit하면 더 간단해질 수 있습니다.

**Securitize:** [41538fa](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/41538faf0df8e31fd4a51f2a478fc6a48a7d6f3a), [7f500c1](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/7f500c1d40da961f709fdeeb9551ef1b6f258363)에서 수정됨.

**Cyfrin:** 확인함.


### 동일 storage read를 캐시하기

**설명:** `priceScaleFactor`, `feeManager`, `liquidityProviderWallet`처럼 한 경로에서 여러 번 읽는 storage 값은 캐시해서 재사용하는 편이 낫습니다.

**Securitize:** [e631361](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/e631361371ccaabceabd8ba1a200f4fd1200e54f), [ea883b9](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/ea883b99af2eb802ffe49c7338379b1e31cd76de), [d32d6a0](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/d32d6a0d6cf9a4215360deb11711740450d1db48)에서 수정됨.

**Cyfrin:** 확인함.


### 이벤트 emit 시 이미 알고 있는 값을 storage에서 다시 읽지 않기

**설명:** `_resetBaseline` 등에서는 방금 계산한 값을 그대로 이벤트에 쓰면 되는데, 다시 storage를 읽고 있습니다.

**Securitize:** [fe9f910](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/fe9f910f1fe6b73aa63171a2e8f4cb55f0092123), [5d39cc1](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/5d39cc1122fabdef280dd0da4ecc5d984f6355a1)에서 수정됨.

**Cyfrin:** 확인함.


### Solidity에서 기본값 초기화는 생략하기

**설명:** `bool shouldReset = false;` 같은 기본값 초기화는 불필요하므로 제거할 수 있습니다.

**Securitize:** [7594671](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/75946718b5129603545c364a2d9c6f57902200d9), [6833173](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/68331735079e5964e636c6f139e44649c7903483)에서 수정됨.

**Cyfrin:** 확인함.


### 상위 함수에서 필요한 storage를 한 번만 읽고 하위 함수로 전달하기

**설명:** 부모 함수가 어떤 storage를 읽고, 자식 함수도 같은 값을 다시 읽는 패턴이 반복됩니다. 값이 중간에 바뀌지 않는다면 상위 함수에서 캐시 후 전달하는 편이 더 효율적입니다.

**Securitize:** 큰 내부 함수 리팩터링을 피하기 위해 현재는 인지 상태로 유지.


### named return variable을 사용해 로컬 변수 제거하기

**설명:** `CountryValidator::getCountry`, `PublicStockOnRamp::calculateDsTokenAmount` 등은 named return을 쓰면 로컬 변수 일부를 제거할 수 있습니다.

**Securitize:** [f6055a0](https://github.com/securitize-io/bc-on-off-ramp-sc/commit/f6055a0a59d63360a6bd8c6aee6c8ca3631832e7)에서 수정됨.

**Cyfrin:** 확인함.


### 반복 계산은 캐시하기

**설명:** 예를 들어 `CountryValidator::validateCountryCode`의 `bytes(_country).length`처럼 같은 계산이 여러 번 쓰이면 캐시하는 편이 낫습니다.

**Securitize:** 인지함.


### `SecuritizeAmmNavProvider`에서 asset decimals를 초기화 시 캐시하기

**설명:** `asset`는 초기화 후 바뀌지 않으므로, `quote/execute` 경로마다 `asset.decimals()`를 다시 부르지 말고 `SCALE_DOWN` 같은 형태로 한 번 캐시할 수 있습니다.

**Securitize:** 인지함.


### `nonZeroNavRate` modifier는 동일한 외부 호출을 두 번 발생시킴

**설명:** `SecuritizeOnRamp`/`SecuritizeOffRamp`의 `nonZeroNavRate`는 `navProvider.rate()`를 한 번 호출하고, 이후 실제 계산 함수가 같은 호출을 다시 수행합니다.

**영향:** 동일 결과를 두 번 얻기 위해 같은 외부 호출을 반복합니다.

**권장 완화 조치:** modifier 대신 내부 함수로 바꾸고, rate를 캐시해서 하위 함수에 전달하십시오.

**Securitize:** 인지함.


### fee 계산이 두 번 일어남

**설명:** `SecuritizeOnRamp::subscribe/swap`는 `calculateDsTokenAmount`에서 fee를 계산한 뒤, `_executeLiquidityTransfer`에서 다시 같은 fee를 계산합니다.

**영향:** 중복 storage read와 중복 외부 호출이 발생합니다.

**권장 완화 조치:** 상위 함수에서 fee를 한 번만 계산하고 하위 함수에 전달하십시오.

**Securitize:** 인지함.


### `initializedNavProvider`, `nonZeroLiquidityProvider` modifier를 내부 함수로 바꾸기

**설명:** modifier가 이미 storage slot을 읽고 있는데, 실제 함수 본문이 같은 slot을 다시 읽고 있습니다.

**영향:** 동일 storage read가 반복됩니다.

**권장 완화 조치:** modifier를 `_getNavProviderStrict()` 같은 내부 함수로 바꾸고, 반환값을 캐시해서 사용하십시오.

**Securitize:** 인지함.


### `_curveBuy`, `_curveSell`의 불필요한 로컬 변수 제거

**설명:** `X`, `Y`, `kLocal`은 한 번만 읽히는 값이므로 굳이 로컬 변수로 둘 필요 없이 storage를 직접 읽는 쪽이 더 단순합니다.

**Securitize:** [5ca8229](https://bitbucket.org/securitize_dev/bc-nav-provider-sc/commits/5ca82293e642b42741dcded549d89e2f74a0f757)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
