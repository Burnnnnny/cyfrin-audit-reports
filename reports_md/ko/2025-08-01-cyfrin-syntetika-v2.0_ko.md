**수석 감사**

[Dacian](https://x.com/DevDacian)

[Jorge](https://x.com/TamayoNft)

**보조 감사**

---

# 발견 사항

## 높은 위험 (High Risk)

### `Minter`가 디페깅, 환율 및 소수점 정밀도 불일치를 고려하지 않아 사용자 및 프로토콜에 전체 손실 시나리오 초래 가능

**설명:** `Minter`는 두 가지 주요 자산인 `baseAsset`과 `hilBTCToken` 간의 전환을 지원합니다:
```solidity
    /// @notice The token address of the base asset accepted for deposit (e.g., WBTC or a stablecoin)
    IERC20 public immutable baseAsset;

    /// @notice The token address of the HilBTC ERC-20 contract (which the Minter will mint/burn)
    IMintableERC20 public immutable hilBTCToken;
```

* `baseAsset`은 명시적으로 `wBTC`, `USDC` 또는 `USDT`일 수 있습니다. 이들은 체인에 따라 8, 6 또는 18과 같은 다양한 정밀도를 가질 수 있습니다.
* `hilBTCToken`은 명시적으로 항상 8자리 정밀도를 가진 프로토콜의 합성 비트코인 토큰 `hBTC`입니다.
* `Minter::mint`는 사용자가 `baseAsset`으로 입력 `amount` 토큰을 공급하고 `hilBTCToken`으로 출력 `amount` 토큰을 받을 수 있게 하지만, 다른 소수점 정밀도, 환율 또는 디페그 이벤트에 대한 조정은 없습니다:
```solidity
    function mint(
        address to,
        uint256 amount
    ) external onlyWhitelisted(msg.sender) onlyWhitelisted(to) nonReentrant {
        baseAsset.safeTransferFrom(msg.sender, address(this), amount);
        hilBTCToken.mint(to, amount);
        totalDeposits += amount;
        emit Minted(to, amount);
    }
```

**파급력:** 여러 가지 부정적인 시나리오가 발생할 수 있지만 가장 중요한 예는 다음과 같습니다:
1) `baseAsset`이 8자리 소수점을 사용하는 랩핑된 형태의 비트코인(예: wBTC)을 나타내지만 현재 디페깅되어 실제 비트코인 가격 근처에 미치지 못하는 경우:
* 사용자는 디페그로 인해 탈중앙화 거래소에서 wBTC를 매우 저렴하게 구입합니다.
* 사용자는 `Minter::mint`를 호출하여 `amount = 1e8`을 전달합니다(일반적으로 1 BTC 가치이지만 디페그로 인해 훨씬 적은 가치).
* 사용자는 1e8 가치의 `hBTC`를 받습니다(여기서 1 BTC는 1e8).
* 사용자는 wBTC가 디페깅되었기 때문에 1 BTC 가치의 wBTC를 제공하지 않았음에도 불구하고 1 BTC 가치의 hBTC를 받았으므로 x/hBTC 유동성 풀을 고갈시킬 수 있습니다.
* 킥오프 통화 메모에는 hBTC가 항상 1:1 비율로 BTC로 상환될 수 있다고 명시되어 있으므로, 이는 오프체인 구성 요소를 포함할 가능성이 높지만 준비금을 고갈시키는 또 다른 방법이 될 수도 있습니다.
* 사용자는 또한 hBTC를 스테이킹하여 의도한 것보다 더 많은 수익을 얻을 수 있지만 이는 즉각적인 영향은 적습니다.

2) `baseAsset`이 USDC와 같은 스테이블코인을 나타내는 경우:
* 사용자가 1e6 ($1)을 예치합니다.
* 사용자는 현재 약 $1,180 가치의 1e6 `hBTC`를 받습니다.
* 사용자는 1)과 같이 x/hBTC 유동성 풀 및 기타 유사한 시나리오를 고갈시킬 수 있습니다.

**권장 완화 방법:** 현재 형태의 `Minter`는 8자리 소수점을 사용하는 랩핑된 BTC 표현에만 안전하게 사용할 수 있습니다. 첫 번째 옵션은 `constructor`에서 이것이 사실인지 강제하고 `baseAsset`이 여러 다른 자산이 될 수 있다는 주석을 제거하는 것입니다.

그러나 언급했듯이 `baseAsset`은 다음과 같은 다양한 자산이 되도록 의도되었습니다:
```solidity
    /// @notice The token address of the base asset accepted for deposit (e.g., WBTC or a stablecoin)
    IERC20 public immutable baseAsset;
```

코드의 현재 의도대로 다른 `baseAsset`을 지원하려면 `mint` 및 `redeem` 함수가 다음을 고려해야 합니다:
* `baseAsset`과 `hilBTCToken` 간의 소수점 정밀도 차이
* `baseAsset`과 `hilBTCToken` 간의 환율
* 또는 `hilBTCToken`의 이름을 `hilSyntheticToken`으로 변경하고 합성 토큰이 항상 `baseAsset`과 동등하도록 보장

또한 `wBTC`일지라도 `baseAsset`이 디페깅되어 실제 비트코인보다 훨씬 적은 가치를 가질 수 있지만 `hBTC`는 항상 기본 `BTC`에 대해 1:1로 상환 가능한 디페그 이벤트를 처리해야 합니다. 따라서 디페그가 발생하면 발행(minting) 또는 상환(redeem)이 되돌려져야 합니다. 이 확인을 구현하는 이상적인 방법은 Chainlink 가격 피드를 통해 디페그가 발생한 경우 되돌리는 것입니다.

특정 체인에서 Chainlink를 사용할 수 없는 경우 보조 옵션은 Uniswap V3 TWAP일 수 있습니다. 세 번째 옵션은 `Minter` 계약을 일시 중지 가능하게 만들고 오프체인 봇이 `baseAsset`의 디페그를 모니터링한 다음 디페그가 발생하면 계약을 일시 중지하도록 하는 것이지만, 이는 오프체인 봇이 올바르게 작동하지 않는 것과 관련된 추가 위험을 초래합니다.

**Syntetika:**
커밋 [a70110c](https://github.com/SyntetikaLabs/monorepo/commit/a70110c1399c49667c2ef5b12e7b1d2354c02509), [db8c139](https://github.com/SyntetikaLabs/monorepo/commit/db8c139954cf8beb1b280b8aed78716554821ece)에서 다음을 수행했습니다:
* 주석을 변경하고 `hilBTCToken`의 이름을 `hilSyntheticToken`으로 변경하여 `Minter`의 다른 인스턴스에 대해 다른 합성 토큰 유형을 지원하도록 더 일반화했습니다.
* `baseAsset`과 `hilSyntheticToken`이 동일한 소수점 정밀도를 사용해야 하도록 생성자 내에 소수점 일치 확인을 구현했습니다.

의도는 `Minter` 계약이 스테이블코인 -> `hBTC` 교환을 지원해서는 안 되지만, `Minter` 계약의 각 인스턴스에서 `baseAsset`이 일치하는 합성 토큰과 짝을 이뤄야 한다는 것이었습니다.

커밋 [20045e0](https://github.com/SyntetikaLabs/monorepo/commit/20045e0dc9907c352491bdb343af62a2257b86b7), [2e308e8](https://github.com/SyntetikaLabs/monorepo/commit/2e308e82340336cc9ba5fdea87340099091f26ec)에서 핵심 `Minter` 함수에 일시 중지 기능을 추가했습니다. 오프체인 봇은 각 `Minter` 인스턴스의 `baseAsset`을 모니터링하고 디페그를 감지하면 해당 인스턴스를 일시 중지합니다.

**Cyfrin:** 확인됨; 디페그에 대해 `baseAsset`을 모니터링하고 올바른 `Minter` 인스턴스를 일시 중지하는 오프체인 봇의 도입은 이 프로세스가 오작동할 경우 추가적인 위험을 초래한다는 점에 유의합니다.

\clearpage
## 중간 위험 (Medium Risk)

### 모든 스테이커가 베스팅 기간 동안 인출할 때 수익 토큰이 볼트에 영구적으로 잠길 수 있음

**설명:** 베스팅 기간 동안 모든 스테이커가 인출하면 수익 토큰이 볼트에 영구적으로 잠길 수 있습니다.

**권장 완화 방법:** 모든 주식을 인출할 수 없도록 하십시오; 일반적인 기술은 첫 번째 입금 시 소각 주소로 1000주를 발행하는 것입니다. 이 기술을 사용하면 소유자도 첫 번째 입금을 할 필요가 없습니다.

**Syntetika:**
커밋 [315fd62](https://github.com/SyntetikaLabs/monorepo/commit/315fd620ea68ed473c5169e6b662d341ecb0f92b), [8c6deb1](https://github.com/SyntetikaLabs/monorepo/commit/8c6deb10f509d6cb2f21e5efa2812af00687c1a7)에서 수정됨 - `StakingVault::deposit`은 이제 첫 번째 입금으로 최소 허용 입금 금액(~$10)에 해당하는 양의 주식을 소각합니다. 소유자는 효과적으로 볼트를 보호하기 위해 $10 수수료를 지불하는 것입니다.

**Cyfrin:** 새 코드로 실험하고 PoC를 수정한 결과, 데드 쉐어(dead shares)가 있어도 문제가 여전히 나타날 수 있음을 발견했습니다.

**권장 완화 방법:** * 가장 간단한 옵션은 8시간의 베스팅 기간을 제거하고 수익이 계약에 입금될 때 수집되도록 허용하는 것입니다. 충분히 긴 쿨다운 기간(기본 90일)이 있는 한, 이는 사용자가 수익의 대부분을 수집하기 위해 많은 금액을 입금한 다음 즉시 인출하는 "적시(just in time)" 공격을 억제합니다. 수익 분배 트랜잭션은 프론트러닝을 방지하기 위해 설계된 특정 [서비스](https://docs.flashbots.net/flashbots-protect/overview)를 통해 실행되어 "적시" 공격으로부터 추가로 보호할 수도 있습니다.

* 더 복잡한 옵션은 다음과 같습니다:

1) 데드 쉐어 외에도 소유자가 입금하도록 하여 "잠긴" 토큰이 데드 쉐어뿐만 아니라 소유자의 지분에도 효과적으로 누적되도록 합니다. 이 경우 소유자의 입금액이 데드 쉐어보다 훨씬 큰지 확인하십시오. 소유자가 마지막으로 인출하는 사람인 경우 그 이후에는 더 이상 수익을 분배하지 마십시오.
2) `_withdraw`에서 이 트랜잭션으로 인해 데드 쉐어만 남게 되는지 확인합니다.
3) 참이면, 미베스팅 금액(unvested amount) > 0인지 확인합니다.
4) 참이면, 미베스팅 금액을 재설정하고 미베스팅 금액을 계약 소유자(또는 다른 계약)에게 보냅니다.
```solidity
// 작업 후 데드 쉐어만 남았는지 확인
if (totalSupply() - shares == deadShares) {
    uint256 remainingUnvested = getUnvestedAmount();
    if(remainingUnvested > 0) {
        vestingAmount = 0;
        lastDistributionTimestamp = 0;
        IERC20(asset()).safeTransfer(owner(), remainingUnvested);
    }
}
```

또한 다음과 같은 "일몰(sunset)" 기능을 구현하는 것을 고려하십시오:
* 관리자는 "일몰" 프로세스를 시작할 수 있습니다; 이는 새로운 입금 및 발행을 방지하지만 사용자가 상환하거나 인출할 수 있도록 허용합니다. 또한 6개월 후로 `sunsetTimestamp`를 설정합니다.
* `DEAD_SHARES`만 남거나(모든 스테이커가 인출함) `block.timestamp > sunsetTimestamp`가 되면, 관리자는 모든 자산 토큰을 관리자에게 전송하는 "구조(rescue)"를 수행하는 특수 함수를 호출할 수 있습니다.

이는 볼트를 "일몰"시키고 남은 토큰을 수집하는 좋은 방법을 제공하는 동시에 사용자에게 인출할 충분한 시간을 제공하고 러그풀(rugpull)로부터 보호합니다.

**Syntetika:**
커밋 [1625c09](https://github.com/SyntetikaLabs/monorepo/commit/1625c09f07d8c43c7cfc3b051e12a3a95c26f32d), [8113753](https://github.com/SyntetikaLabs/monorepo/commit/8113753148d12ee24ec26713f3fc76b2cd12cf54), [347330a](https://github.com/SyntetikaLabs/monorepo/commit/347330a078470164c51074b3a8bc2cb141fa9ca8)에서 다음을 통해 수정되었습니다:
* 첫 번째 입금 시 소각 주소로 1000 "데드 쉐어" 발행
* 첫 번째 입금은 관리자만 수행할 수 있으며, 관리자는 데드 쉐어보다 훨씬 더 큰(최소 10배) 입금을 수행합니다.
* 관리자는 이 지분을 활성 상태로 유지하여 "손실된" 미베스팅 금액이 관리자와 데드 쉐어에 누적되도록 합니다.
* 긍정적인 미베스팅 금액이 있는 동안 모든 비 데드 쉐어가 소각될 때 `_withdraw`에 "구조" 메커니즘을 구현하여 긍정적인 미베스팅 금액이 소유자에게 전송됩니다.

**Cyfrin:** 확인됨. 이상적으로는 `StakingVault::_withdraw`의 [전송](https://github.com/SyntetikaLabs/monorepo/blob/audit/issuance/src/vault/StakingVault.sol#L387)이 `transfer` 대신 `safeTransfer`를 사용해야 합니다.

\clearpage
## 낮은 위험 (Low Risk)

### 블랙리스트에 오른 사용자가 쿨다운 기간 후 인출된 자산을 청구할 수 있음

**설명:** `StakingVault::_update`는 블랙리스트에 오른 사용자가 대부분의 작업을 수행하지 못하도록 `notBlacklisted(from) notBlacklisted(to)` 수정자를 사용합니다.

그러나 `StakingVault::claimWithdraw`는 `notBlacklisted` 수정자를 사용하지 않습니다. 따라서 처음에 인출/상환한 후 블랙리스트에 오른 사용자는 쿨다운 기간이 만료되면 여전히 해당 자산을 청구할 수 있습니다.

**권장 완화 방법:** `StakingVault::claimWithdraw`는 적어도 `notBlacklisted(msg.sender)`를 가져야 하며, `notBlacklisted(receiver)`도 가질 수 있지만 사용자가 임의의 주소를 입력할 수 있으므로 두 번째는 덜 효과적입니다.

**Syntetika:**
커밋 [d98afbf](https://github.com/SyntetikaLabs/monorepo/commit/d98afbfd76670a0cbebfb3399f167481344f689d)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 비준수 사용자가 쿨다운 기간 후 인출된 자산을 청구할 수 있음

**설명:** `StakingVault::redeem, withdraw`는 `onlyWhitelisted` 수정자를 사용하여 `msg.sender`가 화이트리스트에 있거나 규정을 준수하는지 확인합니다. 이 수정자는 결국 `Whitelist::isAddressWhitelisted`를 호출합니다:
```solidity
function isAddressWhitelisted(address user) public view returns (bool) {
    if (manualWhitelist[user] || globalWhitelist) {
        return true;
    }

    return complianceChecker.isCompliant(user);
}
```

사용자가 인출/상환을 생성할 때 화이트리스트에 있었거나 규정을 준수했지만 이후 화이트리스트에서 제거되거나 비준수가 된 경우에도 쿨다운 기간이 지난 후 `StakingVault::claimWithdraw`를 호출하여 자산을 인출할 수 있습니다.

**권장 완화 방법:** `StakingVault::claimWithdraw`는 호출자가 여전히 화이트리스트에 있거나 규정을 준수하는지 확인하기 위해 `onlyWhitelisted(msg.sender)` 수정자를 사용해야 합니다; 대상 주소도 화이트리스트에 있는지 확인하기 위해 `onlyWhitelisted(receiver)`를 사용할 수도 있습니다.

**Syntetika:**
커밋 [86384fe](https://github.com/SyntetikaLabs/monorepo/commit/86384fe1504780338649d25f720fb78b25132875)에서 L-4 발견 사항을 해결하기 위해 `StakingVault`에서 화이트리스트 기능을 완전히 제거하여 수정되었습니다.

**Cyfrin:** 확인됨.

### `StakingVault::mint, deposit`에서 `receiver`가 화이트리스트에 있는지 확인 누락

**설명:** `StakingVault::mint, deposit`은 `msg.sender`가 화이트리스트에 있는지만 확인하고 수신자 매개변수가 화이트리스트에 있는지는 확인하지 않습니다. 화이트리스트에 없는 주소는 주식을 인출, 상환 또는 전송할 수 없으므로, 화이트리스트에 없는 수신자에게 발행된 모든 주식은 영구적으로 잠겨 사용할 수 없게 됩니다.
```solidity
 function mint(
        uint256 shares,
        address receiver //@audit 수신자가 화이트리스트에 없을 수 있음?
    ) public override onlyWhitelisted(msg.sender) returns (uint256 assets) {
        ...
    }
```

**파급력:** 사용자 자금의 영구적 손실 또는 소유자가 화이트리스트 권한을 부여할 경우 일시적 손실.

**권장 완화 방법:** mint() 함수에 수신자에 대한 화이트리스트 확인을 추가하십시오:

```diff
function mint(
    uint256 shares,
    address receiver
+ ) public override onlyWhitelisted(msg.sender) onlyWhitelisted(receiver) returns (uint256 assets) {
-   ) public override onlyWhitelisted(msg.sender) returns (uint256 assets) {
 ...
}
// `deposit`에도 유사한 수정
```

**Syntetika:**
커밋 [86384fe](https://github.com/SyntetikaLabs/monorepo/commit/86384fe1504780338649d25f720fb78b25132875)에서 L-4 발견 사항을 해결하기 위해 `StakingVault`에서 화이트리스트 기능을 완전히 제거하여 수정되었습니다.

**Cyfrin:** 확인됨.

### 사용자가 허가 없이 스테이킹하고 수익을 얻을 수 없음

**설명:** 킥오프 통화 및 클라이언트와의 논의에 명시된 프로토콜의 의도는 사용자가 허가 없이 다음을 수행할 수 있어야 한다는 것입니다:
* 탈중앙화 거래소에서 hBTC 구매
* `StakingVault`를 통해 허가 없는 방식으로 hBTC 스테이킹/언스테이킹

**파급력:** 현재 구현에서 `StakingVault`는 많은 핵심 함수에 `onlyWhitelisted` 수정자를 사용하여 탈중앙화 거래소를 사용하여 허가 없이 hBTC를 구매한 사용자가 이후에 hBTC를 스테이킹하고 수익을 얻는 것을 금지합니다.

이를 가능하게 하려면 관리자가 `setGlobalWhitelist`를 호출해야 하며, 이는 어쨌든 화이트리스트 및 규정 준수 확인을 효과적으로 비활성화합니다.

**권장 완화 방법:** `StakingVault`에서 "블랙리스트" 기능만 사용하고 "화이트리스트" 기능을 제거하여 사용자가 허가 없이 스테이킹 및 수익 창출에 참여할 수 있도록 하는 것을 고려하십시오.

**Syntetika:**
커밋 [86384fe](https://github.com/SyntetikaLabs/monorepo/commit/86384fe1504780338649d25f720fb78b25132875)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 제한 없는 `depositAddresses`로 인해 가스 초과로 `CompliantDepositRegistry::challengeLatestBatch`가 되돌려질 수 있음

**설명:** `CompliantDepositRegistry::challengeLatestBatch`는 `depositAddresses.pop()`을 반복적으로 호출하여 최신 배치에서 입금 주소를 제거하는 제한 없는 루프를 포함합니다. `CompliantDepositRegistry::addDepositAddresses`를 통해 대규모 배치의 입금 주소가 추가된 경우, 이 배치를 챌린지하면 과도한 가스를 소비하여 잠재적으로 블록 가스 한도를 초과하고 트랜잭션이 되돌려질 수 있습니다. 이는 합법적인 챌린지를 실행할 수 없는 서비스 거부(DoS) 취약점을 생성합니다.

**파급력:** 대규모 배치는 챌린지할 수 없게 되어 악의적이거나 잘못된 입금 주소가 확정될 수 있습니다.

**권장 완화 방법:**
`CompliantDepositRegistry::addDepositAddresses`를 통해 추가할 수 있는 주소의 양에 대한 제한을 구현하십시오.

**Syntetika:**
커밋 [319e7ea](https://github.com/SyntetikaLabs/monorepo/commit/319e7ead926e9973e1257337893c031522506bab)에서 배치로 취소할 수 있도록 `challengeLatestBatch`를 변경하여 수정되었습니다.

**Cyfrin:** 확인됨.

### `HilBTC` 토큰을 보유한 악의적인 사용자가 블랙리스트 트랜잭션을 프론트러닝할 수 있음

**설명:** `blacklister`가 `HilBTC` 토큰을 보유한 악의적인 사용자를 블랙리스트에 올리려고 시도할 때, 악의적인 사용자는 멤풀을 모니터링하고 블랙리스트가 적용되기 전에 `minter` 계약을 통해 `HilBTC` 토큰을 신속하게 상환하여 블랙리스트 트랜잭션을 프론트러닝할 수 있습니다. 상환되면 사용자는 의도된 블랙리스트 집행 메커니즘을 완전히 우회하여 기본 자산으로 포지션을 성공적으로 종료합니다.

이는 `StakingVault.sol`에서는 발생할 수 없습니다.

**파급력:** 악의적인 사용자가 블랙리스트 호출을 프론트러닝하면 블랙리스트를 피할 수 있습니다.

**권장 완화 방법:**
발행자의 `redeem` 함수에도 쿨다운을 추가하는 것을 고려하십시오.

**Syntetika:**
인지함; 이는 모든 블랙리스트 메커니즘에서 일반적으로 발생하는 경우입니다. 프로토콜은 프론트러닝을 방지하는 [서비스](https://docs.flashbots.net/flashbots-protect/overview)를 통해 이러한 트랜잭션을 수행할 수 있습니다.

이 문제를 처리하기 위해 코드에 복잡성을 추가할 가치가 없다고 생각하며, 이것이 실제 우려 사항인 경우 블랙리스트 트랜잭션을 실행하기 위해 서비스를 사용할 것입니다. 실제로 이 "공격 벡터"는 거의 문제가 되지 않습니다; 이 공격이 발생한 단일 사례도 알지 못합니다.

### 소유자가 블랙리스트에 오른 주소에서 토큰을 소각할 수 없음

**설명:** `HilBTC.sol` 계약의 `burnFrom()` 함수에는 블랙리스트에 오른 주소와 관련하여 논리적 불일치가 포함되어 있습니다. 이 함수는 `owner`에게 허용량을 요구하지 않고( `_spendAllowance` 확인 우회) 모든 사용자로부터 토큰을 소각할 수 있는 특별한 권한을 부여하지만, 기본 `_burn()` 함수는 `notBlacklisted(from)` 수정자를 포함하는 `_update()`를 호출합니다. 이는 소유자가 블랙리스트에 오른 주소에서 토큰을 소각하는 것을 방지합니다.

**파급력:** 소유자가 블랙리스트에 오른 주소에서 토큰을 소각할 수 없습니다.

**권장 완화 방법:** 소유자가 블랙리스트에 오른 사용자 토큰을 소각할 수 있는 기능을 갖기를 원하는 경우, 블랙리스트에 오른 사용자의 직접 `_balances`가 감소되는 특별한 접근 제어 함수를 만드는 것을 고려하십시오.

**Syntetika:**
커밋 [dc14ad2](https://github.com/SyntetikaLabs/monorepo/commit/dc14ad2c7dacf389deb24fcdca155b5b0acb52d4)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)

### 키와 값의 목적을 명시하기 위해 명명된 매핑 매개변수 사용

**설명:** 키와 값의 목적을 명시하기 위해 명명된 매핑 매개변수 사용:
* `Issuance`:
```solidity
vault/StakingVault.sol
37:    mapping(address => UserCooldown) cooldowns;

helpers/Blacklistable.sol
8:    mapping(address => bool) internal _blacklisted;

helpers/Whitelist.sol
6:    /// @notice A mapping of specific user addresses that are allowed to bypass SBT checks
8:    mapping(address => bool) public manualWhitelist;
```

* `Deposit-Registry`:
```solidity
CompliantDepositRegistry.sol
21:    mapping(address => uint) public investorDepositMap;
```

**Syntetika:**
커밋 [6f77988](https://github.com/SyntetikaLabs/monorepo/commit/6f779887cb2ab813c2d15dbc9cca7991a7301367)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### Solidity에서 기본값으로 초기화하지 마십시오

**설명:** Solidity에서 기본값으로 초기화하지 마십시오:
```solidity
ComplianceChecker.sol
44:        for (uint i = 0; i < complianceOptions.length; i++) {
```

**Syntetika:**
커밋 [7c69e94](https://github.com/SyntetikaLabs/monorepo/commit/7c69e94d165cfe4b78ace518d622e25654e482d3)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 명시적인 `uint` 크기 선호

**설명:** 명시적인 `uint` 크기 선호:

**Syntetika:**
커밋 [cb00843](https://github.com/SyntetikaLabs/monorepo/commit/cb0084368c85c4878dfd0af0fbca1c4924a36497)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 명명된 임포트 사용

**설명:** 명명된 임포트 사용; 이는 일부 장소에서 이미 수행되고 있지만 다른 장소에서는 수행되지 않습니다.

**Syntetika:**
커밋 [a8b4853](https://github.com/SyntetikaLabs/monorepo/commit/a8b485381ad97ffb01595e8e5cc3d479126fcee8)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 금액을 다운캐스팅할 때 `SafeCast` 사용 고려

**설명:** 금액을 다운캐스팅할 때 [SafeCast](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/math/SafeCast.sol) 사용 고려:
* `StakingVault.sol`:
```solidity
144:        cooldowns[msg.sender].underlyingAmount += uint152(assetsRedeemed);
165:        cooldowns[msg.sender].underlyingAmount += uint152(assets);
```

**Syntetika:**
커밋 [8d7987c](https://github.com/SyntetikaLabs/monorepo/commit/8d7987cfe72ab33c51b486fd3ac5fe2670292a30)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `assets`가 0이면 `StakingVault::claimWithdraw`는 되돌려져야 함

**설명:** `assets`가 0이면 `StakingVault::claimWithdraw`는 되돌려져야(revert) 합니다.

**Syntetika:**
커밋 [2fe18df](https://github.com/SyntetikaLabs/monorepo/commit/2fe18df4891810f3daea17777ba7e1d9d7c80d0f)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 명명된 반환 변수를 사용할 때 더 이상 사용되지 않는 `return` 문 제거

**설명:** 명명된 반환 변수를 사용할 때 더 이상 사용되지 않는 `return` 문 제거:
* `StakingVault::_withdrawTo, redeemTo`

**Syntetika:**
커밋 [bd4bb12](https://github.com/SyntetikaLabs/monorepo/commit/bd4bb1222c2112bd33d02757872831d7f713dcdc)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 누락된 이벤트 정보 방출

**설명:** 누락된 이벤트 정보 방출:
* `YieldDistributed`는 금액 외에 `timestamp` 매개변수를 가져야 합니다.

**Syntetika:**
커밋 [f4305a6](https://github.com/SyntetikaLabs/monorepo/commit/f4305a630d731455477a9979a9bb9bbdba541f00)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `Minter.sol`에서 `_setGlobalWhitelist` 호출 누락

**설명:** `Minter.sol` 계약은 `onlyWhitelisted` 수정자를 통해 `mint` 및 `redeem` 함수에 대한 접근 제어를 관리하는 `Whitelist.sol` 추상 계약을 상속합니다.

`onlyWhitelisted` 수정자는 `globalWhitelist` 플래그를 확인합니다. `StakingVault.sol` 계약은 `setGlobalWhitelist` 함수를 구현합니다. `StakingVault.sol` 계약이 이를 사용할 것으로 예상하기 때문에 이는 중요합니다. 그러나 `HilBTC`(`StakingVault.sol`의 자산)를 발행하는 `Minter.sol` 계약은 `setGlobalWhitelist`를 구현하지 않습니다.

**파급력:** `setGlobalWhitelist`가 `Minter.sol`에 구현되어 있지 않기 때문에 `setGlobalWhitelist`가 활성화되면 `StakingVault.sol` 계약이 예상대로 작동하지 않습니다.

**권장 완화 방법:** `minter.sol`에 `setGlobalWhitelist` 구현을 고려하십시오.

**Syntetika:**
커밋 [1796e5e](https://github.com/SyntetikaLabs/monorepo/commit/1796e5ec7f50e73e5fc4af32365b936244811217), [86c7b2e](https://github.com/SyntetikaLabs/monorepo/commit/86c7b2e30666f4c6522d48b4f4ed05f5d52238b0)에서 `Minter` 계약에 필요하지 않고 L-4 수정 후에는 전혀 필요하지 않으므로 글로벌 화이트리스트 기능을 제거하여 수정되었습니다.

**Cyfrin:** 확인됨.

### `StakingVault`에서 최소 트랜잭션 금액 강제

**설명:** 일부 정교한 볼트 해킹은 라운딩을 통해 볼트를 조작하기 위해 1 wei와 같은 매우 적은 금액을 사용하여 볼트 트랜잭션을 수행하는 것과 관련이 있습니다.

일반 사용자는 그토록 적은 금액을 사용하여 트랜잭션을 수행하지 않습니다. 따라서 공격자로부터 이 잠재적인 공격 경로를 박탈하기 위해 최소 트랜잭션 금액을 강제하는 것을 고려하십시오.

hBTC는 8자리 소수점을 사용하고 BTC에 대해 1:1로 상환 가능하므로:
* 100000000 = 1 BTC ($118K)
* 10000 = 0.0001 BTC($11.87)

최소 트랜잭션 한도를 관리자가 BTC 가격 변동에 따라 변경할 수 있는 구성 가능한 매개변수로 만들어 ~$10(또는 선호하는 경우 더 높게) 정도로 유지되도록 하는 것을 고려하십시오.

이를 강제하는 가장 좋은 방법은 `ERC4626::_deposit, _withdraw`를 재정의하고 `assets`가 최소 트랜잭션 금액보다 작으면 내부에서 되돌리는 것입니다.

**Syntetika:**
커밋 [5ba3c19](https://github.com/SyntetikaLabs/monorepo/commit/5ba3c199cf571679503f8f472769c8efe869a001)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `StakingVault.sol`에 `redeem` 편의 함수 누락

**설명:** `StakingVault.sol`은 `stake(uint256 assets)`, `unstake(uint256 assets)`, `mint(uint256 shares)`와 같은 여러 편의 함수를 구현합니다.

그러나 `redeem`에 대한 편의 함수가 없으므로 다음과 같은 함수를 추가하는 것을 고려하십시오:
```solidity
function redeem(uint256 shares) external returns (uint256 shares) {
        return redeem(shares, msg.sender, msg.sender);
    }
```

**Syntetika:**
커밋 [1625c09](https://github.com/SyntetikaLabs/monorepo/commit/1625c09f07d8c43c7cfc3b051e12a3a95c26f32d)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `Minter::ownerMint`가 화이트리스트 요구 사항을 우회하고 실제로 토큰을 전송하지 않고 `totalDeposits`를 증가시킴

**설명:** 일반 함수 `Minter::mint, redeem`은 화이트리스트 요구 사항을 강제하고 `totalDeposits`를 증가시키거나 감소시킬 때 항상 토큰을 전송하거나 소각합니다.

반면 관리자 함수 `Minter::ownerMint`는:
* `addressTo`에 대한 화이트리스트 요구 사항을 강제하지 않습니다.
* 실제로 계약에 토큰을 전송하지 않고 `totalDeposits`를 증가시킵니다.

**파급력:** 이 기능을 오용하면 다음이 발생할 수 있습니다:
* 화이트리스트에 없는 주소로 토큰이 발행됨
* `totalDeposits`의 손상으로 계약의 실제 토큰 양과 다를 수 있음
* 관리자는 무한한 `hBTC` 토큰을 스스로 발행하여 탈중앙화 유동성 풀에서 쌍 토큰을 고갈시키는 데 사용할 수 있음

**권장 완화 방법:** 이상적으로 `Minter::ownerMint`는 관리자가 계약에 충분한 `baseAsset` 토큰을 공급하도록 요구해야 합니다.

**Syntetika:**
인지함; 이는 `ownerMint` 함수의 의도된 기능입니다.

### `StakingVault::deposit, mint, redeem, withdraw`가 0을 반환하면 되돌리기

**설명:** 볼트 악용의 일반적인 전술은 볼트가 다음과 같이 조작되는 것입니다:
* `deposit`이 0주를 반환(사용자가 입금하지만 주식을 받지 못함, 효과적으로 볼트에 기부)
* `mint`가 0 자산을 반환(사용자가 자산 입금 없이 주식을 받음)
* `redeem`이 0 자산을 반환(사용자가 주식을 소각했지만 자산을 받지 못함)
* `withdraw`가 0주를 반환(사용자가 주식 소각 없이 자산 인출)

위의 조건 중 하나라도 성공해야 하는 합법적인 사용자 트랜잭션은 없습니다. 공격자에게 이러한 공격 경로를 거부하려면 `StakingVault::deposit, mint, redeem, withdraw`가 0을 반환하면 되돌리십시오.

**Syntetika:**
커밋 [2e72a57](https://github.com/SyntetikaLabs/monorepo/commit/2e72a57bf8463c7a41d5b4e1c030cf1263507d2f)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `distributeYield`에서 타임스탬프를 확인하지 않으면 보상 분배에 대한 DoS가 발생할 수 있음

**설명:** `StakingVault.sol`의 `distributeYield()` 함수는 유효성 검사 없이 타임스탬프 매개변수를 허용합니다. 잘못된 타임스탬프(특히 먼 미래로 설정된 타임스탬프)가 제공되면 `_updateVestingAmount()` 함수에 `lastDistributionTimestamp`로 저장됩니다. 이는 `distributeYield()`에 대한 향후 호출이 `require(getUnvestedAmount() == 0, StillVesting())` 확인에서 실패하므로 베스팅 메커니즘을 영구적으로 중단시킬 수 있습니다.

**파급력:** 소유자가 잘못된 타임스탬프 값을 설정하면 수익 분배 메커니즘에 대한 영구적인 서비스 거부가 발생합니다.

**권장 완화 방법:**
타임스탬프가 미래의 특정 임계값보다 작은지 확인하십시오.

**Syntetika:**
인지함; 사용자는 이미 수익을 제공하기 위해 관리자를 신뢰하고 있으므로 관리자가 `distributeYield` 입력을 올바르게 설정한다고 신뢰하는 것은 훨씬 작은 일입니다.

### `ComplianceChecker::isCompliant`는 규정 준수 옵션에 필수 소울 바운드 토큰이 없는 경우 잘못된 `true`를 반환함

**설명:** `ComplianceChecker::isCompliant`는 모든 규정 준수 요구 사항을 보편적으로 우회할 수 있는 논리적 결함을 포함합니다. 빈 `requiredSBTs` 배열(길이 = 0)이 있는 규정 준수 옵션이 존재하면 내부 검증 루프가 실행되지 않아 기본적으로 규정 준수 변수가 true로 남습니다.

```solidity
function isCompliant(address user) public view returns (bool) {
        // 사용자는 모든 옵션(KYC 또는 KYB)을 준수할 수 있습니다
        uint256 complianceOptionsLength = _complianceOptions.length;
        for (
            uint256 optionIndex;
            optionIndex < complianceOptionsLength;
            optionIndex++
        ) {
            // 그러나 주소는 해당 옵션에 대한 모든 필수 SBT를 가져야 합니다(예: 제재 대상 아님 및 나이>18)
            bool compliant = true;
            uint256 requiredSBTsLength = _complianceOptions[optionIndex]
                .requiredSBTs
                .length;
            for (uint sbtIndex; sbtIndex < requiredSBTsLength; sbtIndex++) {
                if (
                    !_complianceOptions[optionIndex]
                        .requiredSBTs[sbtIndex]
                        .isVerificationSBTValid(user)
                ) {
                    compliant = false;
                    break;
                }
            }
            if (compliant) {
                return true; <-----
            }
        }
        return false;
    }

```

악의적인 사용자는 이를 악용하여 다른 수취인과 함께 `registerDepositAddress`를 호출함으로써 입금 주소에 대한 무단 액세스 권한을 얻을 수 있습니다.

**파급력:** 모든 주소가 검증 없이 입금 주소에 액세스할 수 있습니다.

**권장 완화 방법:** `setComplianceOptions`에서 빈 옵션을 방지하기 위한 검증을 추가하십시오.

**Syntetika:**
인지함; 실제로 규정 준수 옵션은 관리자에 의해 설정되며 항상 적어도 하나의 필수 소울 바운드 토큰(SBT)을 갖기 때문에 이는 문제가 되지 않습니다.

**Cyfrin:**
### 스테이킹 보상을 효율적으로 분배하기 위해 스테이킹 보상 분배기 사용 고려

**설명:** 한 번의 트랜잭션으로 많은 양을 입금하는 대신 [스테이킹 보상 분배기](https://github.com/ethena-labs/bbp-public-assets/blob/main/contracts/contracts/StakingRewardsDistributor.sol)를 사용하여 스테이킹 보상을 효율적으로 간격을 두는 것을 고려하십시오.

현재 코드는 사용자가 많은 금액을 입금한 다음 스테이킹하여 많은 양의 수익을 얻음으로써 `StakingVault::distributeYield` 호출을 프론트러닝하는 "적시(just in time)" 수익 공격을 억제하기 위해 인출 후 쿨다운을 사용합니다.

그러나 이 공격은 사용자가 쿨다운이 만료될 때까지 기다려야 한다는 점을 제외하고는 여전히 실행될 수 있으며, 쿨다운은 최대 90일이 될 수 있습니다. 그러나 쿨다운은 관리자가 0으로 설정할 수 있어 "적시" 공격을 가능하게 합니다.

또 다른 옵션은 프론트러닝을 방지하기 위해 설계된 [서비스](https://docs.flashbots.net/flashbots-protect/overview)를 통해 `StakingVault::distributeYield` 호출을 수행하는 것입니다.

**Syntetika:**
인지함.

### `StakingVault::decimals`가 기본 자산 소수점보다 크거나 같은지 강제

**설명:** [EIP4626](https://eips.ethereum.org/EIPS/eip-4626)은 다음과 같이 명시합니다:
> convertTo 함수가 EIP-4626 볼트의 decimals 변수 사용 필요성을 제거해야 하지만, 가능한 한 기본 토큰의 decimals를 미러링하여 혼란의 소지를 없애고 프론트엔드 및 기타 오프체인 사용자 간의 통합을 단순화하는 것이 강력히 권장됩니다.

그리고 이 [볼트 속성 테스트](https://github.com/crytic/properties/blob/main/contracts/ERC4626/properties/SecurityProps.sol#L8-L11) 세트는 볼트의 소수점이 기본 자산 소수점보다 크거나 같은지 강제합니다:
```solidity
        assertGte(
            vault.decimals(),
            asset.decimals(),
            "The vault's share token should have greater than or equal to the number of decimals as the vault's asset token."
        );
```

**권장 완화 방법:** `StakingVault::constructor`에서 `IERC20Metadata(_asset).decimals() > decimals()`이면 되돌리십시오.

**Syntetika:**
커밋 [ac97972](https://github.com/SyntetikaLabs/monorepo/commit/ac97972d762392dc8465fa70c718fa78615636ff)에서 EIP4626 표준 권장 사항에 따라 소수점 동등성을 강제하여 수정되었습니다.

**Cyfrin:** 확인됨.

### 볼트 주식이 없을 때 `StakingVault::distributeYield`는 되돌려져야 함

**설명:** `StakingVault::distributeYield`는 볼트 주식이 없을 때, 그리고 업데이트된 코드에서는 볼트 주식이 `DEAD_SHARES`만 있을 때 되돌려져야 합니다. 이는 다음과 같이 우아하게 구현될 수 있습니다:
```diff
    function distributeYield(
        uint256 yieldAmount,
        uint256 timestamp
    ) external onlyOwner {
+       require(totalSupply() > DEAD_SHARES, NoStakers());
```

**Syntetika:**
커밋 [1b9d7f8](https://github.com/SyntetikaLabs/monorepo/commit/1b9d7f8968be39a815ced0d1545d9aec54530413)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `deposit-registry` 계약 생성자에서 역할이 설정되지 않음

**설명:** `issuance` 계약에 속하는 계약의 역할은 생성자에서 초기화되고 이 역할에 대한 주소를 업데이트(취소 및 부여)하는 적절한 함수를 포함합니다:

그러나 `deposit-registry`의 계약 생성자에서는 역할이 초기화되지 않습니다:
```solidity
bytes32 public constant COMPLIANCE_ADMIN_ROLE = keccak256("COMPLIANCE_ADMIN_ROLE");

    constructor(address defaultAdmin) {
        // The defaultAdmin can grant other roles later
        _grantRole(DEFAULT_ADMIN_ROLE, defaultAdmin);
    }
```

`ComplianceChecker` 계약의 생성자에서 `COMPLIANCE_ADMIN_ROLE`을 초기화하는 것을 고려하십시오.
`CompliantDepositRegistry` 계약의 생성자에서 `CANCELER_ROLE`을 초기화하는 것을 고려하십시오.

**Syntetika:**
커밋 [9fccd3b](https://github.com/SyntetikaLabs/monorepo/commit/9fccd3b18f1543b10352e8c26fe4c59877bcf11d)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 가스 최적화 (Gas Optimization)

### 동일한 스토리지 읽기를 방지하기 위해 스토리지 캐시

**설명:** 스토리지에서 읽는 것은 비용이 많이 듭니다; 동일한 스토리지 읽기를 방지하기 위해 스토리지를 캐시하십시오:

**Syntetika:**
커밋 [bc24502](https://github.com/SyntetikaLabs/monorepo/commit/bc245024d7a3d4773661a2eb82284653bfa7f46b), [8560039](https://github.com/SyntetikaLabs/monorepo/commit/8560039b80334a3ad234f7f90ff0a55e50d13edd), [bfad835](https://github.com/SyntetikaLabs/monorepo/commit/bfad8350a4b6b843c47bea023237271c198bfa84)에서 수정되었습니다.

**Cyfrin:** 확인됨. 이상적으로는 `StakingVault::withdraw`도 `redeem` 내부의 수정과 유사하게 `cooldownDuration`을 [캐시](https://github.com/SyntetikaLabs/monorepo/blob/audit/issuance/src/vault/StakingVault.sol#L190-L195)해야 합니다.

### 업그레이드 불가능한 계약의 `constructor`에서 한 번만 설정되는 변수는 `immutable`로 선언해야 함

**설명:** 업그레이드 불가능한 계약의 `constructor`에서 한 번만 설정되는 변수는 `immutable`로 선언해야 합니다:
* `CompliantDepositRegistry::complianceChecker`
* `Minter::baseAsset, hilBTCToken`

**Syntetika:**
커밋 [3d1e596](https://github.com/SyntetikaLabs/monorepo/commit/3d1e5967e99de56cb212565eca42845ba8149784)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 지역 변수를 제거할 때 명명된 반환 변수 사용

**설명:** 지역 변수를 제거할 때 명명된 반환 변수 사용:
* `CompliantDepositRegistry::getDepositAddresses`
* `StakingVault::redeem`

**Syntetika:**
커밋 [f8f821d](https://github.com/SyntetikaLabs/monorepo/commit/f8f821de057517f9b94963607050b6da2ee647a3)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 더 빠른 `nonReentrant` 수정자를 위해 `ReentrancyGuardTransient` 사용

**설명:** 더 빠른 `nonReentrant` 수정자를 위해 [ReentrancyGuardTransient](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/ReentrancyGuardTransient.sol) 사용:
* `issuance/src/minter/Minter.sol`

**Syntetika:**
커밋 [d5131f6](https://github.com/SyntetikaLabs/monorepo/commit/d5131f6dba14b2595fae28e34065cb05abb9ed36)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `Minter:redeem` 및 `_onlySender` 함수에서 `from` 매개변수 제거

**설명:** `Minter:redeem`은 `from` 입력 매개변수를 받지만 `_onlySender`를 호출하여 `from == msg.sender`를 강제합니다.

이 경우 `_onlySender` 함수에 대한 `from` 입력 매개변수가 필요하지 않습니다; 둘 다 제거하고 `Minter:redeem` 내부에서 `msg.sender`만 사용하십시오.

**Syntetika:**
커밋 [94a2165](https://github.com/SyntetikaLabs/monorepo/commit/94a21650ac63be3d22c545e629ca0283d9664872)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 한 번만 사용되는 작은 함수는 부모 함수에 인라인되어야 함

**설명:** 한 번만 사용되는 작은 함수는 부모 함수에 인라인되어야 합니다:
* `StakingVault::_updateVestingAmount`

**Syntetika:**
인지함.

\clearpage
