**Lead Auditors**

[Immeas](https://twitter.com/0ximmeas)

[ChainDefenders](https://x.com/defendersaudits) ([0x539](https://x.com/1337web3) & [PeterSR](https://x.com/PeterSRWeb3))

---

# 발견 사항 (Findings)
## 높은 위험 (High Risk)


### 단일 인출 실패(revert)가 `BasisTradeVault` 인출 큐를 차단할 수 있음

**설명:** [`BasisTradeVault::processWithdrawal`](https://github.com/buttonxyz/button-protocol/blob/9002f2b0d05ba80039bd942c809dbe5bc1a252c9/src/BasisTradeVault.sol#L446-L477)은 큐의 헤드에 있는 정확히 하나의 요청을 처리하고 요청의 `user`에게 최종 ERC20 `safeTransfer`를 수행합니다. 해당 전송이 되돌려지면(revert) 전체 트랜잭션이 되돌려지고 헤드 항목은 제자리에 유지됩니다. 이 함수는 항상 `queueHead`를 대상으로 하며 실패한 항목을 건너뛰거나, 격리하거나, 편집할 수 있는 방법을 제공하지 않으므로, 단일 인출 실패는 전체 큐를 영구적으로 차단합니다(Head-of-Line Blocking). 일반적인 되돌림(revert) 원인은 다음과 같습니다.

* 수신자가 토큰에 의해 블랙리스트/차단됨 (예: USDC/USDT 규정 준수 목록).
* 반올림/수수료로 인해 요청에 대해 계산된 `assets`이 `0`이 되고, 토큰이 0 금액 전송에서 되돌려짐.

이는 우발적으로 발생하거나 처리할 수 없는 요청을 헤드에 배치하여 프로토콜을 방해하는 데 사용될 수 있습니다. 큐는 컨트랙트 업그레이드 또는 수동 개입이 있을 때까지 멈춘 상태로 유지됩니다.

**영향:** 멈춘 요청 뒤에 있는 모든 사용자에 대해 인출 처리가 무기한 중단되어 심각한 인출 지연과 사용자 신뢰 손실을 초래할 수 있습니다.

**권장 완화 조치:** 다음 구현을 고려하십시오.
* 엄격한 큐에서 타임락 + 사용자 풀(user-pull)/관리자 푸시(admin-push) 모델로 재설계: 잠금 해제 가능한 청구(claim)를 기록하고 각 사용자가 타임락 후 직접 `processWithdrawal`을 호출하도록 합니다.
* 건너뛰기/격리 메커니즘 추가: 헤드 인출이 실패하면 "동결된" 세트로 이동하고(지분은 에스크로 상태로 유지하고 자산은 예약), `queueHead`를 진행하여 다른 사람들이 진행할 수 있도록 합니다. 사용자가 지급 주소를 업데이트하고 에이전트가 정책 내에서 재시도/취소할 수 있는 기능을 제공하십시오.

**Button:** 커밋 [`9cde24c`](https://github.com/buttonxyz/button-protocol/commit/9cde24caa4b3f5f37a059bb2fde172cfa374d3a9)에서 요청 기반 시스템으로 이동하여 수정했습니다. 사용자가 수동으로 인출을 청구할 수 있는 기능은 추가하지 않았습니다. 이것이 필요해진다면 현재의 요청 패턴으로 이를 지원하도록 컨트랙트를 매우 쉽게 업그레이드할 수 있습니다.

**Cyfrin:** 확인함. 큐가 제거되었으며 인출은 에이전트에 의해 요청별로 수행됩니다.

\clearpage
## 중간 위험 (Medium Risk)


### Ownable과 AccessControl의 조합으로 인해 관리자 기능이 상실될 수 있음

**설명:** `BasisTradeTailor`와 `BasisTradeVault`는 `Ownable(2Step)Upgradeable`과 `AccessControlUpgradeable`을 혼합하여 사용합니다. 여러 관리자 래퍼(wrapper)는 `onlyOwner`이지만 내부적으로 `grantRole` / `revokeRole`을 호출하며, 이는 호출자가 역할의 관리자(일반적으로 `DEFAULT_ADMIN_ROLE`)를 보유해야 합니다. 예:

```solidity
function grantAdmin(address account) external onlyOwner {
    grantRole(ADMIN_ROLE, account); // requires DEFAULT_ADMIN_ROLE too
}
```

이것은 이중 요구 사항을 생성합니다. 호출자는 `owner`이자 `DEFAULT_ADMIN_ROLE`이어야 합니다. 이러한 신원이 달라지면(예: 기본 관리자를 부여하지 않고 소유권 이전), 거버넌스가 취약해지고 혼란스러워집니다.

**영향:** "소유자"는 역할(관리자/에이전트 부여/취소)을 관리할 수 없을 수 있으며, `owner`가 아닌 `DEFAULT_ADMIN_ROLE` 보유자는 업그레이드할 수 없습니다(`_authorizeUpgrade`가 `onlyOwner`이므로). 또한 추가 래퍼는 기능을 복제하고 이득 없이 공격 표면/바이트코드/ABI를 확대합니다.

**Proof of Concept:** 다음 테스트를 `BasisTradeVault.t.sol`에 추가하십시오(Tailor에 대해서도 매우 유사한 테스트가 작동함).
```solidity
function test_OwnerLosesGrantAbility() public {
    _setupComplete();

    // owner transfers ownership
    vm.prank(deployer);
    vault.transferOwnership(alice);
    assertEq(vault.owner(), alice);

    // new owner cannot grant admin (or agent) roles
    vm.prank(alice);
    vm.expectRevert(
        abi.encodeWithSelector(
            IAccessControl.AccessControlUnauthorizedAccount.selector,
            alice,
            bytes32(0x00)
        )
    );
    vault.grantAdmin(bob);
}
```

**권장 완화 조치:** Ownable을 완전히 제거하여 AccessControl로 통합하는 것을 고려하십시오. `onlyRole(DEFAULT_ADMIN_ROLE)`로 특권 기능을 제어하고 동일한 역할을 통해 업그레이드를 승인하십시오.

```solidity
function grantAdmin(address account) external onlyRole(DEFAULT_ADMIN_ROLE) {
    grantRole(ADMIN_ROLE, account);
}

function _authorizeUpgrade(address impl) internal override onlyRole(DEFAULT_ADMIN_ROLE) {}
```
그런 다음 맞춤형 래퍼 `grantAdmin`, `revokeAdmin`, `grantAgent`, `revokeAgent`를 모두 삭제하십시오. `DEFAULT_ADMIN_ROLE`은 이미 `grantRole` / `revokeRole`을 직접 호출할 권한이 있으므로 이러한 래퍼는 중복됩니다. 이를 제거하면 바이트코드/ABI가 줄어들고 표면적(surface area)이 감소하며 컨트랙트가 정리됩니다.
잠금을 방지하려면: `AccessControlEnumerableUpgradeable`을 사용하고 최소한 하나의 기본 관리자가 남아 있도록 강제하십시오.
```solidity
function _revokeRole(bytes32 role, address account) internal override {
    if (role == DEFAULT_ADMIN_ROLE) {
        require(getRoleMemberCount(DEFAULT_ADMIN_ROLE) > 1, "keep >=1 default admin");
    }
    super._revokeRole(role, account);
}
```
또는 `Ownable`을 유지하는 것이 선호되는 경우 소유권 변경 시 역할을 동기화하십시오(새 소유자에게 `DEFAULT_ADMIN_ROLE`을 부여하고 이전 소유자에게서 취소).

**Button:** `AccessControlEnumerableUpgradeable`로만 이동하여 커밋 [`32f8ca9`](https://github.com/buttonxyz/button-protocol/commit/32f8ca9c9e08986a554e12d3581178419b3d71f9)에서 수정했습니다.

**Cyfrin:** 확인함. `AccessControlEnumerableUpgradeable`이 이제 Tailor와 Vault 모두에 사용됩니다. grant/revoke 호출도 AccessControl 자체 호출을 위해 제거되었습니다.

\clearpage
## 낮은 위험 (Low Risk)


### 최소 입금 강제 누락

**설명:** `BasisTradeVault::deposit` 함수는 최소 입금 금액을 강제하지 않습니다. 먼지 입금을 허용하면 몇 가지 바람직하지 않은 상황이 발생할 수 있습니다.

1.  **경제적 비효율성:** 사용자는 트랜잭션 가스비가 입금 가치보다 훨씬 높은 소액을 입금할 수 있습니다.
2.  **방해 가능성:** 공격자가 많은 소액 입금으로 저장소를 스팸(spam)하는 시나리오를 가능하게 할 수 있으며, 이는 직접적인 보안 위협은 아니지만 성가신 일이 될 수 있습니다.

컨트랙트는 기본 ERC4626 구현에 의해 가장 심각한 문제로부터 보호되지만, 합리적인 최소 입금 금액을 강제하는 것은 사용자 보호 및 컨트랙트 견고성을 위한 좋은 관행입니다.

**권장 완화 조치:** 관리자가 설정할 수 있는 새로운 상태 변수 `minDepositAmount`를 도입하십시오. 입금된 `assets`이 이 최소 금액보다 크거나 같아야 하도록 `deposit` 함수를 수정하십시오.

**Button:** 커밋 [`9cde24c`](https://github.com/buttonxyz/button-protocol/commit/9cde24caa4b3f5f37a059bb2fde172cfa374d3a9)에서 수정됨.

**Cyfrin:** 확인함. 관리자가 설정할 수 있는 최소 입금액이 이제 강제됩니다.


### `BasisTradeTailor`가 ERC-165를 준수하지 않음

**설명:** `BasisTradeTailor` 컨트랙트는 `ITailor` 인터페이스를 상속하고 구현합니다. ERC-165 표준에 따르면, `supportsInterface` 함수는 `ITailor` 및 `IERC1822Proxiable`(`UUPSUpgradeable`에서 옴)의 인터페이스 ID로 쿼리될 때 `true`를 반환해야 합니다.

그러나 현재 `supportsInterface` 구현은 `super.supportsInterface(interfaceId)`만 호출하며, 이는 검사를 부모 `AccessControlUpgradeable` 컨트랙트에 위임합니다. 부모 컨트랙트는 `ITailor` 및 `IERC1822Proxiable` 인터페이스를 인식하지 못하므로 인터페이스 ID에 대해 `false`를 반환합니다. 이는 컨트랙트가 실제로 구현하는 인터페이스를 지원하지 않는다고 잘못 보고함을 의미하며, 인터페이스 감지를 위해 ERC-165에 의존하는 다른 컨트랙트와의 상호 작용을 깨뜨릴 수 있습니다.

**권장 완화 조치:** `supportsInterface` 함수는 `super` 함수를 호출하는 것 외에도 `ITailor` 및 `IERC1822Proxiable` 인터페이스 ID를 명시적으로 확인하도록 업데이트되어야 합니다. 이는 컨트랙트가 주어진 인터페이스의 구현을 올바르게 알리도록 보장합니다.

```solidity
// ...existing code...

import {IERC1822Proxiable} from "@openzeppelin/contracts/interfaces/draft-IERC1822.sol";

// ...existing code...

    /**
     * @notice Override supportsInterface to resolve multiple inheritance
     */
    function supportsInterface(bytes4 interfaceId)
        public
        view
        override(AccessControlUpgradeable)
        returns (bool)
    {
        return interfaceId == type(ITailor).interfaceId ||
        return interfaceId == type(IERC1822Proxiable).interfaceId ||
        super.supportsInterface(interfaceId);
    }

// ...existing code...
```

**Button:** 커밋 [`32f8ca9`](https://github.com/buttonxyz/button-protocol/commit/32f8ca9c9e08986a554e12d3581178419b3d71f9)에서 수정됨.

**Cyfrin:** 확인함. 권장 사항이 구현되었습니다.


### `BasisTradeVault.totalPendingWithdrawals`와 `BasisTradeTailor.withdrawalRequests[pocket]` 간의 상태 불일치(State Drift)

**설명:** `BasisTradeVault`는 `totalPendingWithdrawals`에서 보류 중인 인출을 추적하는 반면 `BasisTradeTailor`는 자체 `withdrawalRequests[pocket]`을 유지합니다. 이 두 카운터는 서로 다른 코드 경로에서 변경됩니다.
* `BasisTradeVault::requestRedeem`은 Vault 카운터를 올리고 Tailor 카운터를 새로운 합계로 설정합니다.
* `BasisTradeVault::processWithdrawal`은 Vault 카운터만 줄입니다.
* `BasisTradeTailor::processWithdrawal`은 Tailor 카운터만 줄입니다.

Vault가 `updateWithdrawalRequest(uint256 amount)`를 노출하지만, 여전히 경쟁 상태(race condition)가 발생하기 쉽고 두 집계가 정상적인 작업으로 표류(drift)하도록 허용하는 합계 설정(set-the-total) 모델을 사용합니다. "합계 설정"을 통한 수동 재동기화는 취약하며 동시 요청 중에 올바른 값을 덮어쓸 수 있습니다.

다음 재설계 중 하나를 고려하십시오.
* Vault를 진실의 단일 소스(single source of truth)로 만들고 Tailor의 `withdrawalRequests`를 완전히 제거하십시오. Tailor는 자금을 이동시키는 실행자 역할만 합니다. 봇과 운영자는 `vault::totalPendingWithdrawals`만 읽습니다.
* 합계 설정에서 Vault에 의해 보호되는 델타 기반 회계로 전환: 대신 `tailor::processWithdrawal`이 델타만큼 증가하도록 하고, `requestRedeem`에서 델타를 보내도록 하십시오.

**Button:** [``](https://github.com/buttonxyz/button-protocol/commit/9cde24caa4b3f5f37a059bb2fde172cfa374d3a9)에서 수정됨. 오프체인 에이전트만 인출 요청 금액을 설정하도록 업데이트되었습니다. 입금과 인출 간의 순 결제(net settling)를 수행하려면 이것이 필요합니다.

**Cyfrin:** 확인함. `BasisTradeVault`에서 `totalPendingWithdrawals`가 제거되었습니다.


### HyperEVM 및 HyperCore의 토큰에 대한 소수점 불일치

**설명:** `transferToCore` 함수는 지정된 `amount`의 기본 자산을 포켓에서 HyperCore로 전송합니다. 그러나 HyperEVM 토큰과 HyperCore 토큰 간의 잠재적인 소수점 정밀도 차이를 고려하지 않습니다. 두 시스템이 서로 다른 소수점 구성(예: 6 소수점 vs 18 소수점)을 사용하는 경우 전송된 `amount`는 HyperCore에서 HyperEVM에서 의도한 것과 크게 다른 값을 나타낼 수 있습니다. 불일치에 따라 자금 손실이나 잔액 인플레이션이 발생할 수 있습니다.

**영향:** 에이전트는 소수점 불일치로 인해 의도치 않게 예상보다 많거나 적은 토큰을 전송할 수 있습니다.

**권장 완화 조치:** HyperEVM과 HyperCore 간에 전송할 때 소수점 정규화 메커니즘을 도입하는 것을 고려하십시오.

**Button:** 인지함. HyperCore 소수점에 대해 사용할 수 있는 온체인 정보가 많지 않으므로 에이전트에서 오프체인으로 이 문제를 해결할 것입니다.


### 배포 스크립트에 암호화되지 않은 개인 키 필요

**설명:** 여러 배포/운영 스크립트에서는 개인 키를 환경 변수에서 로드하고 스크립트 내에서 직접 사용해야 합니다. 예:

```solidity
// DeployBasisTradeTailor.s.sol
uint256 deployerPrivateKey = vm.envUint("DEPLOYER_PRIVATE_KEY");
uint256 adminPrivateKey    = vm.envUint("ADMIN_PRIVATE_KEY");
...
vm.startBroadcast(deployerPrivateKey);
...
vm.startBroadcast(adminPrivateKey);

// DeployBasisTradeVault.s.sol
uint256 deployerPrivateKey = vm.envUint("DEPLOYER_PRIVATE_KEY");
...
vm.startBroadcast(deployerPrivateKey);

// DeployMockPocketOracle.s.sol
uint256 deployerPrivateKey = vm.envUint("DEPLOYER_PRIVATE_KEY");
...
vm.startBroadcast(deployerPrivateKey);

// DeployMocks.s.sol
uint256 deployerPrivateKey = vm.envUint("DEPLOYER_PRIVATE_KEY");
...
vm.startBroadcast(deployerPrivateKey);
```

원시 개인 키를 `.env`(일반 텍스트)를 통해 저장하고 로드하는 것은 운영 보안 위험입니다. 키는 버전 제어, 로그, 쉘 기록, 잘못 구성된 백업 또는 손상된 개발자 머신/CI 러너를 통해 유출될 수 있습니다.

더 안전한 접근 방식은 스크립트에 키를 포함하지 않고 Foundry의 [지갑 관리](https://getfoundry.sh/forge/reference/script/) 및 키스토어 지원을 사용하는 것입니다. 권장 패턴:

1. [`cast`](https://getfoundry.sh/cast/reference/wallet/import/)를 사용하여 암호화된 로컬 키스토어로 키 가져오기(머신당 한 번):

```bash
cast wallet import deployerKey --interactive
cast wallet import adminKey --interactive
cast wallet import agentKey --interactive   # if needed
```

2. CLI에서 서명자가 제공되도록 매개변수 없는 브로드캐스팅을 사용하도록 스크립트 변경:

```solidity
// before: vm.startBroadcast(deployerPrivateKey);
vm.startBroadcast();
// ...
vm.stopBroadcast();
```

3. 각 역할에 민감한 단계를 적절한 계정으로 실행(다른 서명자가 필요한 경우 별도의 실행 또는 별도의 스크립트로 분할):

```bash
# Deployer phase
forge script script/DeployBasisTradeTailor.s.sol:DeployBasisTradeTailor \
  --rpc-url "$RPC_URL" --broadcast --account deployerKey --sender <deployer_addr> -vvv

# Admin phase (grants/approvals)
forge script script/ConfigureBasisTradeTailor.s.sol:ConfigureBasisTradeTailor \
  --rpc-url "$RPC_URL" --broadcast --account adminKey --sender <admin_addr> -vvv
```

이 방법은 개인 키를 유휴 상태에서 암호화된 상태로 유지하고 일반 텍스트 환경 변수를 통해 노출하지 않습니다. 대안으로 하드웨어 지갑(`--ledger`)을 고려하고 공유 환경에서 `.env`에 원시 키가 포함되지 않도록 하십시오.

추가 지침은 Patrick의 [이 설명 비디오](https://www.youtube.com/watch?v=VQe7cIpaE54)를 참조하십시오.

**Button:** 커밋 [`c89bce0`](https://github.com/buttonxyz/button-protocol/commit/c89bce0f88770f473524e997eb47fca7dccae0e0)에서 수정됨.

**Cyfrin:** 확인함. 키에 키스토어가 사용됩니다.


### 큰 가격 변동 중 실행 시 가격이 책정되는 인출 문제

**설명:** 인출은 요청 시점에 "가격 고정"됩니다. `requestRedeem`은 요청 시점의 환율을 사용하여 `shares`와 계산된 `assetsAfterFee = previewRedeem(shares)`를 저장합니다. 나중에 에이전트가 `processWithdrawal`을 호출하면 볼트는 에스크로된 `shares`를 소각하지만 해당 주식이 실행 시점에 가치가 있는 금액이 아니라 저장된 자산 금액을 지급합니다.

**영향:** 그 사이에 주가가 하락한 경우(예: 오라클 업데이트, Core PnL 손실, 디페그), 초기 요청자는 현재 가격에 비해 사실상 과다 지급을 받게 되며, 부족분은 남은 주주들에게 전가됩니다. 극심한 하락장에서는 이것이 뱅크런 역학을 가속화하고 의도한 것보다 빨리 볼트를 고갈시켜 잠재적으로 지급 불능 지점까지 이르게 할 수 있습니다.

**권장 완화 조치:** 실행 시점의 가격 사용을 고려하십시오. 요청 시에는 `shares`만 저장하고 처리 시점에 현재 환율(즉, `previewRedeem(shares)`)을 사용하여 `assetsAfterFee`를 계산하십시오. 허용 가능한 슬리피지 경계(프로토콜 기본값 및/또는 사용자 제공)가 있는 실행 안전장치를 사용할 수도 있습니다.

**Button:** 커밋 [`9cde24c`](https://github.com/buttonxyz/button-protocol/commit/9cde24caa4b3f5f37a059bb2fde172cfa374d3a9)에서 실행 시 가격 책정으로 이동하여 수정했습니다.

**Cyfrin:** 확인함. 가격은 이제 실행 시점에 책정됩니다.


### 0 주식 발행(mint)에 대한 확인 부족

**설명:** `previewDeposit(assets)`는 반올림 및/또는 입금 수수료로 인해 소액 입금에 대해 합법적으로 `0` 주식을 반환할 수 있습니다. 입금 경로가 이를 방지하지 않으면 사용자는 자산을 볼트로 전송하고 0 주식을 받을 수 있습니다(의도하지 않은 "기부").

0 주식에 대한 확인을 추가하는 것을 고려하십시오.
```solidity
function previewDeposit(uint256 assets) public view virtual override returns (uint256) {
    uint256 fee = _extractFeeFromTotal(assets, depositFeeBps);
    require(fee < assets, "Deposit fee exceeds assets");
    uint256 shares = super.previewDeposit(assets - fee);
    require(shares > 0, "0 shares");
    return shares;
}
```


**Button:** 커밋 [`9cde24c`](https://github.com/buttonxyz/button-protocol/commit/9cde24caa4b3f5f37a059bb2fde172cfa374d3a9)에서 수정됨.

**Cyfrin:** 확인함. `previewDeposit`은 이제 `> 0 shares`가 발행되는지 확인합니다.

\clearpage
## 정보 (Informational)


### 중복 변수 문(Redundant variable statements)

**설명:** `BasisTradeVault`에서 함수 `maxMint`, `mint`, `withdraw` 및 `redeem`은 표준 ERC4626 인터페이스의 재정의(override)입니다. 이 컨트랙트에서 이러한 함수는 사용자 정의 입금 및 인출 흐름(예: 표준 `withdraw` 및 `redeem` 대신 `requestWithdraw` 및 `requestRedeem` 사용)을 강제하기 위해 의도적으로 비활성화됩니다.

이러한 함수는 비활성화되어 즉시 되돌려지거나 고정 값을 반환하므로 해당 매개변수(`receiver`, `shares`, `assets`, `owner`)는 함수 본문 내에서 사용되지 않습니다. 코드는 매개변수 이름을 별도의 줄에 배치(예: `receiver;`)하여 이를 명시적으로 인정하며, 이는 사용되지 않는 변수에 대한 컴파일러 경고를 끄지만 중복됩니다.

**권장 완화 조치:** 명확성을 향상시킬 수 있는 사용되지 않는 매개변수를 나타내는 다른 방법은 매개변수 없이 함수 선언을 작성하는 것입니다(예: `foo(uint256, address)`). 이는 매개변수가 의도적으로 사용되지 않음을 알리는 Solidity의 일반적인 규칙입니다.

```solidity
// ...existing code...
    /**
     * @notice Mint function is disabled
     * @dev This vault only supports asset-based deposits
     */
    function mint(uint256 /*shares*/, address /*receiver*/) public virtual override returns (uint256) {
        revert("Mint disabled: use deposit");
    }

    // ============================================
// ...existing code...
    /**
     * @notice Standard withdraw function is disabled
     * @dev Users must use requestWithdraw instead
     */
    function withdraw(
        uint256 /*assets*/,
        address /*receiver*/,
        address /*owner*/
    ) public virtual override returns (uint256) {
        revert("Withdraw disabled: use requestWithdraw");
    }

    /**
     * @notice Standard redeem function is disabled
     * @dev Users must use requestRedeem instead
     */
    function redeem(
        uint256 /*shares*/,
        address /*receiver*/,
        address /*owner*/
    ) public virtual override returns (uint256) {
        revert("Redeem disabled: use requestRedeem");
    }

// ...existing code...
    /**
     * @notice Returns the maximum shares that can be minted
     * @dev Always returns 0 as mint is disabled
     * @param receiver Address that would receive the shares
     * @return Always 0 (mint disabled)
     */
    function maxMint(address /*receiver*/) public view virtual override returns (uint256) {
        return 0; // Mint is disabled
    }
}
```

**Button:** 커밋 [`9d8ed75`](https://github.com/buttonxyz/button-protocol/commit/9d8ed75bd5ed4957c7b23f9b06ff362b7bb218a4)에서 수정됨.

**Cyfrin:** 확인함.


### 인출 및 상환 요청 취소 누락

**설명:** `BasisTradeVault`의 인출 프로세스는 2단계 메커니즘입니다. 사용자가 먼저 `requestWithdraw` 또는 `requestRedeem`을 호출하면 요청이 큐에 배치되고 볼트 주식이 컨트랙트 내에 에스크로됩니다. 실제로 사용자에게 기본 자산을 전송하는 두 번째 단계인 `processWithdrawal`은 `AGENT_ROLE`을 가진 특권 주소만 실행할 수 있습니다.

이 설계는 상당한 중앙 집중화 위험을 초래합니다. 에이전트가 악의적이 되거나, 손상되거나, 단순히 임무 수행을 중단하면 `processWithdrawal` 호출을 거부할 수 있습니다. 결과적으로 모든 보류 중인 인출 요청은 무기한 큐에 갇히게 됩니다.

인출을 요청한 사용자는 자신의 주식이 컨트랙트에 잠겨 있으며 주식을 되찾기 위해 일방적으로 요청을 취소할 방법이 없습니다. 즉, 자금이 사실상 동결되어 에이전트의 활성(liveness) 및 협조에 전적으로 의존하게 됩니다.

**권장 완화 조치:** 이를 완화하기 위해 사용자가 자신의 보류 중인 인출 요청을 취소할 수 있는 함수를 도입해야 합니다. 이 함수는 에스크로된 주식을 사용자에게 반환하여 인출 큐에서 효과적으로 스테이킹을 해제합니다.

사용자가 요청한 후 특정 시간이 지난 후에만 요청을 취소할 수 있도록 이 취소 함수에 타임락을 추가할 수 있습니다. 이렇게 하면 사용자가 큐를 스팸하는 것을 방지하는 동시에 에이전트가 응답하지 않을 경우 탈출구를 제공할 수 있습니다.

**Button:** 인지함, 그대로 둡니다. 볼트가 이미 베이시스 거래를 청산하고 인출을 처리하기 위해 비용을 지불했을 수 있으므로 취소는 까다롭습니다. 실패한 요청에 대한 실제 패턴을 프로덕션에서 보기 위해 인출을 수정하는 기능을 추가하지 않았습니다. 특히 ERC20 컨트랙트에서 블랙리스트에 오른 사용자가 해당 자산을 복구하도록 돕는 것은 의미가 없다고 생각합니다.


### 테스트에서 지수 표기법 사용 고려

**설명:** 테스트에서는 종종 소수점 금액을 `100 * 10**6`으로 씁니다. 대신 과학적 표기법(`100e6`)을 사용하는 것이 더 간결하고 가독성을 높이며(시각적 토큰/0이 더 적음) 지수 오류를 줄이므로 고려하십시오.

**Button:** 커밋 [`9d8ed75`](https://github.com/buttonxyz/button-protocol/commit/9d8ed75bd5ed4957c7b23f9b06ff362b7bb218a4)에서 수정됨.

**Cyfrin:** 확인함.


### 사용되지 않는 `Pocket::approve` 함수

**설명:** `Pocket` 컨트랙트는 `approve` 함수를 정의하지만 이 함수는 현재 시스템 아키텍처 내에서 호출되지 않습니다. 사용되지 않거나 참조되지 않는 코드를 유지하면 향후 업그레이드에서 오용되거나 더 이상 유효하지 않은 가정을 생성할 수 있으므로 프로토콜의 공격 표면이 증가합니다.

**권장 완화 조치:** 승인 함수가 필요하지 않으면 완전히 제거하십시오.

**Button:** 인지함. 새 컨트랙트에서 사용될 것입니다.


### `ERC4626::mint` 구현 고려

**설명:** `BasisTradeVault`는 현재 `mint()` 및 `previewMint()`를 비활성화(둘 다 되돌림)하여 "정확한 주식을 위한 발행(mint for exact shares)" 흐름(예: 슬리피지 인식 입금 또는 마이그레이터)에 의존하는 ERC-4626 도구, 라우터, 볼트 애그리게이터 및 시뮬레이터와의 상호 운용성을 줄입니다. 다음 구현을 고려하십시오.

* `shares`를 발행하기 위해 사용자가 제공해야 하는 *총(gross)* 자산이 반환되도록 입금 수수료를 위에 추가하여 순 자산 요구 사항을 **총액(gross up)**으로 계산하는 `previewMint(shares)`
* `mint(shares, receiver)`를 활성화하여 표준 ERC-4626 경로(새 `previewMint`를 호출함)를 사용하는 동시에 `deposit`과 동일한 제어(화이트리스트/TVL 상한)를 강제합니다.
  이는 수수료 의미론(제공된 총액에서 수수료 추출)을 유지하고 기존 ERC-4626 통합 및 도구와의 호환성을 실질적으로 개선합니다.

```solidity
/**
 * @notice Preview the *gross* assets required to mint `shares`
 * @dev We gross-up the net assets (from base ERC4626 math) by adding the deposit fee on top.
 */
function previewMint(uint256 shares) public view virtual override returns (uint256) {
    require(shares > 0, "Cannot mint 0 shares");

    // Net assets required by base ERC4626 math (what must end up in the vault)
    uint256 netAssets = super.previewMint(shares);

    // Fee is charged on top of the net assets
    uint256 fee = _calculateFeeAmount(netAssets, depositFeeBps);
    return netAssets + fee; // gross = net + fee
}

/**
 * @notice Mint `shares` to `receiver`, charging the deposit fee on top
 * @dev Enforces the same whitelist/TVL checks as `deposit`.
 *      `super.mint` will use `previewMint` (gross) and enforce `maxMint(receiver)`.
 */
function mint(uint256 shares, address receiver) public virtual override checkDepositWhitelist(receiver) returns (uint256 assets) {
    return super.mint(shares, receiver);
}

/**
 * @notice Maximum shares that can be minted for `receiver`
 * @dev Derived from the gross-asset TVL cap (`maxDeposit`) by converting gross→net (remove fee),
 *      then net→shares using base ERC4626 math. Uses Floor rounding to stay within cap.
 */
function maxMint(address receiver) public view virtual override returns (uint256) {
    // Respect deposit whitelist if enabled
    if (depositWhitelistEnabled && !depositWhitelist[receiver]) {
        return 0;
    }

    // `maxDeposit(receiver)` returns the remaining *gross* capacity (fee included)
    uint256 grossCap = maxDeposit(receiver);
    if (grossCap == 0) return 0;

    // Convert gross -> net by extracting the fee portion from the gross amount
    uint256 feeFromGross = _extractFeeFromTotal(grossCap, depositFeeBps);
    uint256 netCap = grossCap - feeFromGross;

    // Convert the net-asset capacity to shares
    return _convertToShares(netCap, Math.Rounding.Floor);
}
```
`BasisTradeVault.t.sol`에 이에 대한 테스트도 추가하십시오(`maxDeposit`에 대한 테스트도 추가됨).
(테스트 코드는 원본과 동일하므로 생략)

**Button:** Cyfrin 팀에 의해 커밋 [`d38f046`](https://github.com/buttonxyz/button-protocol/commit/d38f046befbc5deba426eb6cabac65703cd643b5)에서 수정됨.

**Cyfrin:** 확인함. `mint`가 구현되었습니다.


### On-behalf-of 인출 활성화 고려

**설명:** `BasisTradeVault`의 인출 큐는 항상 요청자에게 지불하는 자체 시작 요청만 지원합니다(`receiver = msg.sender`). 더 나은 ERC-4626 정렬 및 구성 가능성(라우터, 관리자, 타사 통합)을 위해 기존 큐 API를 확장하여 명시적 `receiver`를 허용하고 큐 항목에 호출자 / 소유자 / 수신자를 유지하십시오. 그런 다음 처리할 때 이 값을 사용하여 자산을 `receiver`에게 전송하고 표준 ERC-4626 `Withdraw(caller, receiver, owner, assets, shares)` 이벤트를 올바르게 방출하십시오.

새 주소를 유지하도록 큐 항목 확장을 고려하십시오.
```solidity
struct WithdrawalRequest {
    address owner;     // share owner whose shares were escrowed
    address receiver;  // recipient of assets on payout
    address caller;    // who enqueued the request
    uint256 shares;
    uint256 assets;
    uint256 timestamp;
}
```
기존 이벤트 업데이트:
```solidity
struct WithdrawalRequested {
    address owner;
    address receiver;
    address caller;
    uint256 shares;
    uint256 assets;
    uint256 timestamp;
}
```
그리고 허용량 소비와 함께 요청 함수 서명을 변경합니다.
```
function requestWithdraw(
    uint256 assets,
    address receiver,
    address owner
) external returns (uint256 queuePosition) {
    require(assets > 0, "Cannot withdraw 0 assets");
    uint256 grossAssets = assets + _calculateFeeAmount(assets, withdrawalFeeBps);
    uint256 shares = _convertToShares(grossAssets, Math.Rounding.Ceil);
    return requestRedeem(shares, receiver, owner);
}

function requestRedeem(
    uint256 shares,
    address receiver,
    address owner
) public requirePocket returns (uint256 queuePosition) {
    address caller = msg.sender;

    require(receiver != address(0), "Invalid receiver");
    require(owner != address(0), "Invalid owner");
    require(shares > 0, "Cannot redeem 0 shares");

    // spend allowance when acting on behalf of `owner`
    if (caller != owner) {
        _spendAllowance(owner, caller, shares);
    }

    // ...

    emit WithdrawalRequested(caller, owner, receiver, assetsAfterFee, shares, fee, queuePosition);
}
```
마지막으로 처리 과정에서 저장된 신원을 사용합니다.
```solidity
function processWithdrawal() external onlyAgent {
    require(queueHead < queueTail, "No withdrawals to process");

    uint256 currentQueuePosition = queueHead;
    WithdrawalRequest memory request = withdrawalQueue[currentQueuePosition];
    require(request.owner != address(0), "Invalid withdrawal request");

    // Ensure sufficient vault liquidity
    uint256 vaultBalance = IERC20(asset()).balanceOf(address(this));
    require(vaultBalance >= request.assets, "Insufficient vault balance for withdrawal");

    // Burn escrowed shares and advance queue
    _burn(address(this), request.shares);
    totalEscrowedShares -= request.shares;
    totalPendingWithdrawals -= request.assets;
    delete withdrawalQueue[currentQueuePosition];
    queueHead++;

    // Pay the correct receiver
    IERC20(asset()).safeTransfer(request.receiver, request.assets);

    // Emit canonical ERC-4626 Withdraw with correct identities
    emit Withdraw(request.caller, request.receiver, request.owner, request.assets, request.shares);
    emit WithdrawalProcessed(request.owner, request.assets, currentQueuePosition);
}
```

이 변경 사항은 현재 안전성(주식은 즉시 에스크로됨)을 유지하고 상호 운용성을 개선하며 이벤트 및 지급이 실제 호출자/소유자/수신자 역할을 반영하도록 보장합니다.


**Button:** 커밋 [`9cde24c`](https://github.com/buttonxyz/button-protocol/commit/9cde24caa4b3f5f37a059bb2fde172cfa374d3a9)에서 구현됨.

**Cyfrin:** 확인함. 호출자/소유자/수신자가 이제 저장되고 이벤트에 사용되며 `requestRedeem`에서 승인이 사용됩니다.


### SPDX 라이선스 식별자 누락

**설명:** 모든 Solidity 소스 파일에 SPDX 라이선스 식별자가 누락되었습니다. 각 파일의 맨 위에 라이선스 헤더를 추가하여 라이선스를 명확히 하고 도구 경고(Solc/linters/CI)를 끄십시오.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;
```

이것이 없으면 일부 도구는 파일을 라이선스가 없는 것으로 처리하거나 경고를 내보내며, 이는 다운스트림 재사용 및 규정 준수를 방해할 수 있습니다.

**Button:** 인지함. 아직 라이선스를 결정하지 않았지만 저장소를 공개하기 전에 이 헤더를 추가할 것입니다.


### `BasisTradeVault::totalAssets`가 포켓 보유 자금을 과소 계산할 수 있음

**설명:** [`BasisTradeVault::totalAssets`](https://github.com/buttonxyz/button-protocol/blob/9002f2b0d05ba80039bd942c809dbe5bc1a252c9/src/BasisTradeVault.sol#L252-L283)는 현재 포켓의 잔액을 보고하기 위해(아직 확정되지 않은) 오라클에 의존합니다. 오라클은 포켓에 있는 자산을 제외하여 포켓 보유 자금을 놓칠 수 있으며, 이로 인해 미리보기 및 상한이 왜곡될 수 있습니다. 이는 오라클의 책임으로 간주될 수 있지만, `totalAssets()`가 `IERC20(asset()).balanceOf(pocket)`을 명시적으로 포함하거나 오라클 개발에 이를 포함하도록 하여 포켓 보유 자산을 포함하도록 하는 것을 고려하십시오.

**Button:** [`9cde24c`](https://github.com/buttonxyz/button-protocol/commit/9cde24caa4b3f5f37a059bb2fde172cfa374d3a9)에서 수정됨. PPS 기반 오라클로 이동했습니다.

**Cyfrin:** 확인함. 오라클은 이제 총 자산을 결정하는 데 사용되는 주당 가격만 반환합니다.


### 키와 값의 목적을 명시적으로 기록하기 위해 명명된 매핑 매개변수 사용

**설명:** 키와 값의 목적을 명시적으로 기록하기 위해 명명된 매핑 매개변수를 사용하십시오.

* [`BasisTradeTailor`](https://github.com/buttonxyz/button-protocol/blob/9002f2b0d05ba80039bd942c809dbe5bc1a252c9/src/BasisTradeTailor.sol#L41-L47)
  ```solidity
  // Mappings
  /// @notice Maps pocket address to the user who controls it
  mapping(address => address) public pocketUser;
  /// @notice Tracks pending withdrawal amounts for each pocket
  mapping(address => uint256) public withdrawalRequests;
  /// @notice Whitelist of addresses allowed to create pockets
  mapping(address => bool) public creationWhitelist;
  ```

* [`BasisTradeVault`](https://github.com/buttonxyz/button-protocol/blob/9002f2b0d05ba80039bd942c809dbe5bc1a252c9/src/BasisTradeVault.sol#L80):
  ```solidity
  mapping(address => bool) public depositWhitelist;
  ```

* [`PocketFactory`](https://github.com/buttonxyz/button-protocol/blob/9002f2b0d05ba80039bd942c809dbe5bc1a252c9/src/PocketFactory.sol#L28):
  ```solidity
  mapping(address => bool) public approvedTailors;
  ```

**Button:** 커밋 [`a9ba276`](https://github.com/buttonxyz/button-protocol/commit/a9ba276e0536c38f8a89fb1105ca7f3a0d918519)에서 수정됨.

**Cyfrin:** 확인함.


### 명명된 반환 변수를 사용할 때 쓸모없는 return 문 제거

**설명:** 명명된 반환 값 또는 `return` 문 중 하나를 제거하십시오.

* [`BasisTradeVault::requestWithdraw`](https://github.com/buttonxyz/button-protocol/blob/9002f2b0d05ba80039bd942c809dbe5bc1a252c9/src/BasisTradeVault.sol#L396-L404)
  ```solidity
  function requestWithdraw(uint256 assets) external returns (uint256 queuePosition) {
      // ...

      return requestRedeem(shares);
  }
  ```

* [BasisTradeVault::](https://github.com/buttonxyz/button-protocol/blob/9002f2b0d05ba80039bd942c809dbe5bc1a252c9/src/BasisTradeVault.sol#L412-L444)
  ```solidity
  function requestRedeem(uint256 shares) public requirePocket returns (uint256 queuePosition) {
      // ...

      return queuePosition;
  }
  ```

* [`Pocket::exec`](https://github.com/buttonxyz/button-protocol/blob/9002f2b0d05ba80039bd942c809dbe5bc1a252c9/src/Pocket.sol#L94-L106)
  ```solidity
  function exec(address target, bytes calldata data) external onlyOwner returns (bytes memory result) {
      // ...

      return result;
  }
  ```

**Button:** 커밋 [`9d8ed75`](https://github.com/buttonxyz/button-protocol/commit/9d8ed75bd5ed4957c7b23f9b06ff362b7bb218a4)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### `BasisTradeVault::depositToTailor`에서 중복된 `approve(0)`

**설명:** `BasisTradeVault::depositToTailor`는 `tailor`에게 허용량을 부여하고 `tailor.deposit(pocket, amount)`를 호출한 다음 허용량을 다시 0으로 설정합니다.

```solidity
IERC20(asset()).forceApprove(address(tailor), amount);
tailor.deposit(pocket, amount);
IERC20(asset()).forceApprove(address(tailor), 0); // redundant
```

`BasisTradeTailor::deposit`은 `safeTransferFrom(msg.sender, pocket, amount)`를 통해 정확히 `amount`를 가져오며, 이는 전체 허용량을 소비합니다. 표준 ERC20의 경우 호출 후 허용량은 이미 `0`이므로 후행 `forceApprove(..., 0)`은 불필요한 스토리지 쓰기 및 외부 호출을 수행합니다.

마지막 0 설정 호출을 제거하는 것을 고려하십시오.

**Button:** 커밋 [`9d8ed75`](https://github.com/buttonxyz/button-protocol/commit/9d8ed75bd5ed4957c7b23f9b06ff362b7bb218a4)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage

