**Lead Auditors**

[Dacian](https://x.com/DevDacian)

[Jorge](https://x.com/TamayoNft)

**Assisting Auditors**

[0ximmeas](https://x.com/0ximmeas)

[Stalin](https://x.com/0xStalin)

---

# 발견 사항 (Findings)
## 치명적 위험 (Critical Risk)


### 어떤 스테이커든 OOG revert를 유발해 슬래싱을 완전히 피할 수 있음

**설명:** 이 이슈는 vault 순회를 악용해 OOG를 유발한다는 점에서 다른 보상 관련 이슈와 유사하지만, 수정 방식과 영향은 별개입니다.

**영향:** 슬래싱이 사실상 불가능해집니다.

**권장 완화 조치:** 한 계정이 등록할 수 있는 vault 수에 합리적인 상한(예: 10개)을 두십시오.

**StatusL2:** [6697432](https://github.com/status-im/status-network-monorepo/commit/6697432b862ee515d1783941a78d07e9a91991eb)에서 수정됨.

**Cyfrin:** 확인함.


### 사용자가 에어드롭을 두 번 청구할 수 있음

**설명:** `KarmaAirdrop.sol`은 `merkleRoot` 갱신 기능 때문에 pause될 수 있는데, 이 과정에서 이전에 청구되지 않은 토큰이 새 merkle root로 다시 포함됩니다. 이때 청구 함수가 pause되지 않으면 이중 청구가 가능합니다.

**영향:** `merkleRoot` 업그레이드 중 미청구 사용자가 이중 청구하여 토큰을 훔칠 수 있습니다.

**권장 완화 조치:** `KarmaAirdrop::claim`에 `whenNotPaused`를 추가하십시오.

**StatusL2:** [ebbf84b](https://github.com/status-im/status-network-monorepo/commit/ebbf84b63eb3b4559c93d130b883d78c9cb040f8)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 높은 위험 (High Risk)


### 악의적 사용자가 보상을 받고 싶을 때 임의로 vault를 exit시킬 수 있음

**설명:** 보상은 계정의 모든 vault를 순회하면서 누적 보상을 계산해 상환하는 방식입니다. 이 구조를 악용하면 악성 사용자가 vault 상태를 강제로 불리하게 만들 수 있습니다.

**영향:** 사용자는 시스템 전체에서 완전히 빠져나오기 전까지 보상을 상환받지 못할 수 있습니다.

**권장 완화 조치:** vault 등록은 factory만 할 수 있도록 접근 제어를 추가하십시오.

**StatusL2:** [5e93ecb](https://github.com/status-im/status-network-monorepo/commit/5e93ecbf02b2038d88cdafcd9eb99ed32872e38e)에서 수정됨.

**Cyfrin:** 확인함.


### 컨트랙트가 pause되면 악의적 사용자가 공짜 보상을 받을 수 있음

**설명:** `StakeVault`의 leave 경로에는 `try/catch`가 있어 특정 pause 또는 업그레이드 실패 상황에서 사용자가 기대하지 않은 이득을 얻을 수 있습니다.

**영향:** 일부 사용자가 다른 사용자의 비용으로 무상 보상을 받을 수 있습니다.

**권장 완화 조치:** 가장 안전한 방법은 `try/catch`를 제거하고, 애초에 나쁜 업그레이드가 일어나지 않도록 운영 절차를 강화하는 것입니다.

**StatusL2:** [d813449](https://github.com/status-im/status-network-monorepo/commit/d8134493d8cc9f86f36744cf308c3c8a18a09c7f)에서 수정됨.

**Cyfrin:** 확인함.


### 관련 없는 account를 넣어 reveal delay를 우회할 수 있음

**설명:** `RLN`은 슬래싱 보호를 위해 commit-reveal 방식을 사용하지만, reveal 단계에서 제공되는 account가 실제 슬래시 대상인지 충분히 검증하지 않습니다.

**영향:** 슬래셔는 딜레이를 완전히 우회하고 자신이 보상을 가져갈 수 있습니다.

**권장 완화 조치:** 제공된 account가 실제 슬래시 대상 계정인지 강제하십시오. 가장 쉬운 방법은 reveal 시 공개되는 PK를 `members[poseidonHash(privateKey)]`의 사용자와 대조하는 것입니다.

**StatusL2:** [62021fc](https://github.com/status-im/status-network-monorepo/commit/62021fce4df10101598e02004049d7a14e7f2d00)에서 수정됨.

**Cyfrin:** 확인함.


### 다른 슬래셔의 reveal delay를 늘릴 수 있음

**설명:** `RLN`의 commit 저장 구조에 commit한 호출자 정보가 포함되지 않아, 다른 슬래셔가 타인의 reveal delay를 의도적으로 늘릴 수 있습니다.

**영향:** 어떤 슬래셔든 다른 슬래셔의 reveal delay를 크게 늘릴 수 있습니다.

**권장 완화 조치:** `slashCommitments`의 key에 `msg.sender`까지 포함하거나, commit자를 별도로 저장하십시오.

**StatusL2:** [f15f5e9](https://github.com/status-im/status-network-monorepo/commit/f15f5e9997243a1991c4dadc98f790bfe57ae100)에서 수정됨.

**Cyfrin:** 확인함.


### 사용자가 lock을 우회하고 언제든지 stake를 출금할 수 있음

**설명:** migration과 vault 상태 조합을 악용하면, 사용자가 lock 기간을 실질적으로 우회하면서도 긴 lock에 대한 더 높은 multiplier 혜택은 그대로 누릴 수 있습니다.

**영향:** 사용자는 최대 lock 기간을 선택해도 실질적인 downside 없이 언제든 출금할 수 있습니다. 이 방식으로 다른 사용자 보상을 훔치듯 최대 3배 더 많은 보상을 얻을 수 있습니다.

**권장 완화 조치:** `vault.hasLeft == true`인 vault로의 migration을 금지하는 것이 한 가지 방법입니다.

**StatusL2:** [fb6c8ef](https://github.com/status-im/status-network-monorepo/commit/fb6c8ef112477f831067edaf3abd3aa82b0503d9)에서 수정됨.

**Cyfrin:** 확인함.


### global MP cap 불변식이 unstake 시 깨져 언더플로우와 영구 DoS를 유발할 수 있음

**설명:** 프로토콜은 `totalMPAccrued <= totalMaxMP` 불변식을 가정하고, `_totalMP()`에서 unchecked subtraction으로 신규 MP를 cap합니다. 이 불변식이 깨지면 전역 MP 업데이트가 고장납니다.

**영향:** `totalMPAccrued > totalMaxMP` 상태가 되면, 시간이 조금이라도 지난 뒤 `_updateGlobalState()`를 호출하는 모든 동작이 revert합니다. 스테이킹, 언스테이킹, vault 업데이트, 보상 분배가 모두 영구적으로 막힐 수 있습니다.

**권장 완화 조치:** `_totalMP()` 내부에서 `totalMPAccrued >= totalMaxMP`인 경우를 방어적으로 처리하고, subtraction 전에 값을 clamp하십시오.

**StatusL2:** [56a7b64](https://github.com/status-im/status-network-monorepo/commit/56a7b64a782150b2a87563621212b076e35f84f5)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 중간 위험 (Medium Risk)


### `ERC20VotesUpgradeable::getPastTotalSupply`가 값을 과대평가함

**설명:** Karma 토큰은 voting에 사용되며 checkpoint를 유지합니다. 하지만 아직 분배되지 않은 토큰이 과거 total supply 계산에 잘못 반영됩니다.

**영향:** quorum 계산 등에 쓰는 total supply가 실제보다 높아집니다.

**권장 완화 조치:** 팀 TBD. 보상 분배 전 mint/burn이 `totalSupply`에 미치는 영향을 재설계해야 할 가능성이 큽니다.

**StatusL2:** acknowledged. 팀은 `getPastTotalSupply`에 의존할 계획이 없다고 밝혔습니다.


### `Karma.sol`에서 사용자가 과도하게 슬래시될 수 있음

**설명:** `Karma::_calculateSlashAmount`는 낮은 잔액에 대해서도 `MIN_SLASH_AMOUNT = 1e18` 또는 전체 잔액까지 끌어올려 계산할 수 있습니다.

**영향:** 소액 잔액 사용자가 과도하게 슬래시됩니다.

**권장 완화 조치:** 최소 금액 적용은 최종 슬래시 금액에 한 번만 수행하도록 `Karma::_slash`를 조정하십시오.

**StatusL2:** [cc00600](https://github.com/status-im/status-network-monorepo/commit/cc00600de5f0b67552a370d198cf6675d0d6fffb)에서 수정됨.

**Cyfrin:** 확인함.


### commit-reveal이 slash frontrunning을 충분히 막지 못함

**설명:** `RLN`의 commit-reveal은 private key를 가진 다른 슬래셔가 보상 수취 주소만 바꿔 frontrun하는 상황을 막기 위한 것이지만, 현재 구현만으로는 보호가 충분하지 않습니다.

**영향:** frontrunning 방어가 불충분합니다.

**권장 완화 조치:** `slash()`를 internal로 바꿔 commit-reveal 경로로만 슬래싱할 수 있게 하거나, commit slash가 있는 동안에는 `slash()`를 직접 호출하지 못하게 하십시오.

**StatusL2:** [0644175](https://github.com/status-im/status-network-monorepo/commit/0644175caf0904af9eee182b7d4c7dc39ade9823)에서 수정됨.

**Cyfrin:** 확인함.


### `Karma.sol`은 서명 관련 변수를 초기화하지 않음

**설명:** 현재 초기화는 몇 개 initializer만 호출하며, EIP712 서명용 `name`, `version` 계열 초기화가 누락되어 있습니다.

**영향:** `name = ""`, `version = 0` 상태가 되어, 지갑 호환성이 깨지고 `delegateBySig`를 제대로 사용할 수 없습니다.

**권장 완화 조치:** `ERC20Permit` 초기화를 수행하십시오.

**StatusL2:** [7d447fd](https://github.com/status-im/status-network-monorepo/commit/7d447fd66db824bf210214ef5c06cc4fb2e3dcd8)에서 수정됨.

**Cyfrin:** 확인함.


### 공격자가 사용자의 에어드롭 청구를 막을 수 있음

**설명:** `KarmaAirdrop::claim`는 에어드롭 청구 후 `ERC20Votes::delegateBySig`를 호출하는데, 이 경로가 실패하면 청구 자체가 막힐 수 있습니다.

**영향:** 사용자는 에어드롭을 청구하지 못할 수 있습니다.

**권장 완화 조치:** `delegateBySig` 호출을 `try/catch`로 감싸고, 실패 시에도 allowance가 충분하면 청구가 진행되도록 하십시오.

**StatusL2:** [f9b97ab](https://github.com/status-im/status-network-monorepo/commit/f9b97ab93956cc4d0fbd8ca3687c25a2dceef856)에서 수정됨.

**Cyfrin:** 확인함.


### 사용자가 migration 중 누적 보상을 잃을 수 있음

**설명:** staking position을 새 vault로 migration할 때, 새 vault에 이미 있던 `rewardsAccrued` 상태를 덮어쓸 수 있습니다.

**영향:** 사용자는 migration 중 누적 보상을 잃을 수 있습니다.

**권장 완화 조치:** 사용자가 비어 있는 vault로만 migration하도록 강제하십시오.

**StatusL2:** [e038ece](https://github.com/status-im/status-network-monorepo/commit/e038ece7ff3df9936af0d8bdf920bbf31c0d8e80)에서 수정됨.

**Cyfrin:** 확인함.


### `DeployProtocol.s.sol`이 잘못된 전송 허용 대상을 설정함

**설명:** 배포 스크립트는 `StakeManager.sol`과 `SimpleKarmaDistributor.sol` 둘 다 Karma 토큰 전송이 가능해야 하는데, 실제로는 `stakeManager`를 두 번 넣고 있습니다.

**영향:** `SimpleKarmaDistributor::redeemRewards`가 revert하고, 그 결과 슬래싱도 함께 고장납니다.

**권장 완화 조치:** 스크립트에서 올바른 컨트랙트 주소를 각각 허용 대상으로 등록하십시오.

**StatusL2:** [025f790](https://github.com/status-im/status-network-monorepo/commit/025f790f92f9fa0f40bb905b9db3a7a5473134e6)에서 수정됨.

**Cyfrin:** 확인함.


### 어떤 reward distributor가 pause되면 슬래싱 전체가 동작하지 않음

**설명:** `Karma::_slash`는 모든 reward distributor를 순회하며 `redeemRewards`를 호출합니다. 이때 그중 하나라도 pause되어 revert하면 전체 슬래싱이 실패합니다.

**영향:** reward distributor 중 하나만 pause되어도 슬래싱이 동작하지 않습니다.

**권장 완화 조치:** `IRewardDistributor`에 `isPaused()`를 추가하고, 슬래싱 시 pause된 distributor는 건너뛰십시오.

**StatusL2:** [99b73b0](https://github.com/status-im/status-network-monorepo/commit/99b73b0d6060d2c9eee916006b57ad3f5d3913c1)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 낮은 위험 (Low Risk)


### 언스테이킹 반올림이 사용자에게 유리함

**설명:** 언스테이킹 시 MP 감소량을 계산할 때 `mulDiv` 내림 반올림을 사용해, 실제보다 적은 MP만 차감됩니다.

**영향:** 반올림 오류가 사용자에게 유리하게 작용합니다.

**권장 완화 조치:** 올림 반올림을 사용하십시오.

**StatusL2:** [ff85b3a](https://github.com/status-im/status-network-monorepo/commit/ff85b3a7f1eaba433ba3dc1f5ec1a08158462be9)에서 수정됨.

**Cyfrin:** 확인함.


### 자기 자신으로의 migration이 가능함

**설명:** `staked balance == 0`인 경우 self-migration이 가능하며, 이때 `delete vaultData[msg.sender];` 같은 이상한 상태 전이가 생길 수 있습니다.

**영향:** 의도하지 않은 상태 전이가 발생합니다.

**권장 완화 조치:** self-migration을 금지하십시오.

**StatusL2:** [08ada30](https://github.com/status-im/status-network-monorepo/commit/08ada30122a7fc604d6be284457f8f9b13677cf2)에서 수정됨.

**Cyfrin:** 확인함.


### `totalMPAccrued` 누적은 첫 stake 시점부터 시작되어야 함

**설명:** `StakeManager`는 배포 시점의 `block.timestamp`로 `lastMPUpdatedTime`을 초기화하지만, 첫 스테이킹 시 이 값이 적절히 갱신되지 않는 논리 결함이 있습니다.

**영향:** 다행히 `totalMPAccrued`는 온체인 핵심 로직에 직접 쓰이지 않아 실질적인 영향은 거의 없습니다.

**권장 완화 조치:** 첫 stake 시에도 `lastMPUpdatedTime`를 항상 현재 시각으로 업데이트하십시오.

**StatusL2:** [56a7b64](https://github.com/status-im/status-network-monorepo/commit/56a7b64a782150b2a87563621212b076e35f84f5)에서 수정됨.

**Cyfrin:** 확인함.


### `totalMaxMP`가 개별 `maxMP`를 잘못 집계함

**설명:** 각 stake는 4년치 MP 상한을 가지는데, 전역 집계 과정에서 이 개별 상한이 정확히 반영되지 않습니다.

**영향:** `StakeManager::totalMPAccrued`와 `StakeManager::totalMP`가 실제보다 부풀려집니다.

**권장 완화 조치:** 수정은 단순하지 않으므로 전역 MP 집계 로직을 재검토하십시오.

**StatusL2:** [56a7b64](https://github.com/status-im/status-network-monorepo/commit/56a7b64a782150b2a87563621212b076e35f84f5)에서 수정됨.

**Cyfrin:** 확인함.


### 문서와 달리 슬래싱 후에는 새 사용자를 등록할 수 없음

**설명:** `RLN.sol`의 `SET_SIZE`는 등록 가능한 최대 사용자 수를 정의하지만, 슬래싱 시 레지스트리에서 사용자를 삭제해도 새 등록을 다시 받을 수 있는 경로가 없습니다.

**영향:** 문서 설명과 달리 슬래싱 이후 새 사용자를 등록할 수 없습니다.

**권장 완화 조치:** 누락된 기능을 추가하거나 문서를 수정하십시오.

**StatusL2:** [2c73d4d](https://github.com/status-im/status-network-monorepo/commit/2c73d4d6a597983e5170c6d1779e49100eed90f6)에서 수정됨.

**Cyfrin:** 확인함.


### `KarmaNFT.sol`이 mint 이벤트를 잘못 시뮬레이션함

**설명:** 실제 mint가 아닌데도 `Transfer` 이벤트를 mint처럼 emit합니다.

**영향:** 잘못된 이벤트가 기록됩니다.

**권장 완화 조치:** 이벤트 semantics를 실제 동작과 맞추십시오.

**StatusL2:** [72cd30d](https://github.com/status-im/status-network-monorepo/commit/72cd30d61a02aa85c11f1bd460e07e047e855333)에서 수정됨.

**Cyfrin:** 확인함.


### `InitializeKarmaTiersScript`는 decimals를 고려하지 않음

**설명:** 스크립트가 raw token amount를 그대로 사용해, 18 decimals를 가진 Karma 토큰 기준으로 모든 사용자가 가장 높은 티어가 될 수 있습니다.

**영향:** `KarmaTiers.sol`이 잘못된 threshold 값으로 초기화됩니다.

**권장 완화 조치:** tier amount에 `e18` 정밀도를 반영하십시오.

**StatusL2:** [fa6a44e](https://github.com/status-im/status-network-monorepo/commit/fa6a44ee7abb7cc48168e5d89fb2be85026061cb)에서 수정됨.

**Cyfrin:** 확인함.


### 오프체인/온체인 Poseidon 해시 불일치로 RLN identity가 슬래시 불가능해질 수 있음

**설명:** `PoseidonHasher`는 상태 초기화에 raw EVM addition을 사용합니다. wider system이 arbitrary 32-byte private key를 허용한다면, 오프체인과 온체인 해시 결과가 갈릴 여지가 있습니다.

**영향:** 일부 RLN identity가 영구적으로 슬래시 불가능해질 수 있습니다.

**권장 완화 조치:** 프로토콜 설계 의도에 맞춰 키 범위를 제한하거나, 오프체인/온체인 Poseidon 계산이 완전히 일치하도록 한 가지 기준으로 통일하십시오.

**StatusL2:** [4ead416](https://github.com/status-im/status-network-monorepo/commit/4ead4169eedf20f5de8de5573fe45f1d31f99417)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 정보성 (Informational)


### `RLN::setSlashRevealWindowTime`의 Natspec이 잘못됨

**설명:** 주석은 최대값이 `365 days`라고 적고 있지만 실제 코드는 `1 days`로 제한합니다.

**권장 완화 조치:** Natspec을 실제 코드와 맞추십시오.

**StatusL2:** [1f8414d](https://github.com/status-im/status-network-monorepo/commit/1f8414daa55e8955de7baef428e5ea2c14104327)에서 수정됨.

**Cyfrin:** 확인함.


### `Karma._onlySlasher`의 이름이 실제 권한 범위와 일치하지 않음

**설명:** `_onlySlasher`는 Admin도 허용하면서 이름에는 Admin이 드러나지 않아 혼동을 줍니다.

**권장 완화 조치:** 이름을 실제 허용 역할과 맞추거나, 역할 구성을 더 명확히 표현하십시오.

**StatusL2:** [030efdd](https://github.com/status-im/status-network-monorepo/commit/030efdda72d94969304d2f377ceeb9fc38d5a05a)에서 수정됨.

**Cyfrin:** 확인함.


### `KarmaTiers.sol` constructor는 단순화할 수 있음

**설명:** constructor에서 불필요하게 owner를 다시 설정합니다.

**권장 완화 조치:** `KarmaTiers.sol`, `KarmaNFT.sol`, `BaseNFTMetadataGenerator.sol`의 constructor에서 `transferOwnership(msg.sender)`를 제거하십시오.

**StatusL2:** [606e3d1](https://github.com/status-im/status-network-monorepo/commit/606e3d14a76f6568865267df0db56db5963440ca)에서 수정됨.

**Cyfrin:** 확인함.


### 라이브러리의 사용되지 않는 함수는 제거를 고려

**설명:** `StakeMath::_estimateLockTime`, `MultiplierPointMath::_lockTimeAvailable` 등 여러 internal 함수가 실제 프로토콜에서 사용되지 않습니다.

**권장 완화 조치:** 제거를 고려하십시오.

**StatusL2:** 인지함. 팀은 UI용 view 함수를 추가할 계획이라고 밝혔습니다.


### `MultiplierPointMath.sol`의 수식 단순화 고려

**설명:** 일부 수식은 100을 중간값으로 두고 있어 더 단순하게 정리할 수 있습니다.

**권장 완화 조치:** 수식을 단순화해 가독성과 유지보수성을 높이십시오.

**StatusL2:** [16c5b3c](https://github.com/status-im/status-network-monorepo/commit/16c5b3c64009042515491d8ad39d7e812ec4a841)에서 수정됨.

**Cyfrin:** 확인함.


### `DeployProtocol.s.sol`은 RLN 컨트랙트를 배포하지 않음

**설명:** 자동 배포/설정 스크립트가 `RLN.s.sol`을 호출하지 않아 `RLN.sol`, `PoseidonHasher.sol`이 함께 배포되지 않습니다.

**권장 완화 조치:** `RLN.s.sol`을 실행하고 `RLN.sol`을 Karma의 slasher로 추가하십시오.

**StatusL2:** 인지함. 팀은 별도 배포가 가능하다고 밝혔습니다.


### `Karma::removeRewardDistributor`는 모든 virtual Karma를 burn함

**설명:** emergency 용도로 reward distributor를 제거할 때 full balance를 burn하는 동작이 들어 있습니다.

**권장 완화 조치:** emergency 시 잘못된 virtual Karma가 보고될 수 있어 현재 burn이 들어가 있지만, 의도를 더 명확히 문서화하거나 흐름을 재검토하는 것이 좋습니다.

**StatusL2:** [19557eb](https://github.com/status-im/status-network-monorepo/commit/19557eb034dbb532c745fd910d6dd2e2a26ab9ee)에서 수정됨.

**Cyfrin:** 확인함.


### emergency mode는 `StakeManager.sol`의 악성 업그레이드까지 막아 주지 못함

**설명:** 문서는 emergency mode가 `StakeManager.sol`의 악성 업그레이드 대응 시나리오 중 하나라고 설명하지만, 실제 설계로는 완전한 보호를 제공하지 못합니다.

**영향:** `StakeManager.sol` owner가 탈취되면 사용자의 스테이킹 토큰 출금을 막아버릴 수 있습니다.

**권장 완화 조치:** 현재 설계로는 완전한 방어가 쉽지 않으므로, 최소한 문서에 한계를 명확히 반영하십시오.

**StatusL2:** [7401475](https://github.com/status-im/status-network-monorepo/commit/7401475b4918319e627eb3ba7df24a498f8d063f)에서 수정됨.

**Cyfrin:** 확인함.


### emergency mode를 켜면 이후 슬래싱이 동작하지 않음

**설명:** `RLN.sol`은 `Karma::slash`를 호출하고, 이 함수는 reward distributor를 순회하며 `redeemRewards`가 revert하지 않는다고 가정합니다. 그러나 `StakeManager::redeemRewards`는 emergency mode에서 revert합니다.

**권장 완화 조치:** emergency mode를 켠 뒤에는 해당 reward distributor를 제거하고, reward snapshot을 회수한 뒤 손실된 Karma 잔액을 직접 다시 발행하는 방식이 필요합니다.

**StatusL2:** [a5d51d5](https://github.com/status-im/status-network-monorepo/commit/a5d51d55f07dbbed4e3edd34042e0989d0e29f5a)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
