**Lead Auditors**

[0kage](https://twitter.com/0kage_eth)

[Chinmay Farkya](https://twitter.com/dev_chinmayf)

**Assisting Auditors**


---

# 발견 사항 (Findings)
## 중간 위험 (Medium Risk)


### 무제한 토큰 재할당 권한으로 인한 중앙 집중화 위험 생성

**설명:** `WorldLibertyFinancialV2::ownerReallocateFrom` 함수는 소유자에게 주소에서 토큰을 소각하고 다른 주소로 발행할 수 있는 무제한 권한을 제공하여 컨트랙트 전체에 구현된 모든 보안 메커니즘을 완전히 우회합니다.

이것이 법적 규정 준수 시나리오를 위해 의도되었을 수 있지만, 이 기능은 토큰의 탈중앙화 특성을 훼손하는 상당한 중앙 집중화 위험과 거버넌스 조작 기회를 생성합니다.

```solidity
//WorldLibertyFinancialV2.sol
function ownerReallocateFrom(
    address _from,
    address _to,
    uint256 _value
) public onlyOwner {
    _burn(_from, _value);  // No approval, no checks
    _mint(_to, _value);    // No restrictions
}
```

이 함수는 모든 보호 메커니즘을 우회합니다.
- 블랙리스트에 오른 주소에서 압류하거나 보낼 수 있음
- 컨트랙트가 일시 중지된 상태일 때 전송할 수 있음
- 타임락 없음 - 투표 마감 전/후에 계정 간에 토큰을 즉시 이동할 수 있음
- `reallocation`에 특정한 이벤트 방출이 최소화됨
- 재할당에 대한 속도 제한 없음 - 계정 간에 임의의 금액을 재할당할 수 있음

**영향:** 소유자가 임의로 압류할 수 있다면 사용자는 자신의 토큰을 진정으로 "소유"하지 않는 것입니다.

**권장 완화 조치:** 이 함수의 사용에 대해 하나 이상의 안전 장치를 추가하는 것을 고려하십시오.

- 이 함수가 호출될 상황(예: 법원 명령에 따른 압류 등)에 대한 명확한 문서화
- 이 함수가 호출될 때 특정 이벤트 방출 추가
- 재할당이 임계값 이상인 경우 거버넌스 승인 추가
- 재할당이 임계값 이상인 경우 시간 지연 추가

**WLFI:**
커밋 [b567696](https://github.com/worldliberty/usd1-protocol/blob/b56769613b6438b62b8b4133a63fca727fdbc631/contracts/wlfi/WorldLibertyFinancialV2.sol#L170)에서 수정됨

**Cyfrin:** 확인함. 특정 사용 사례 문서가 추가되고 추가 안전 장치가 구현되었습니다.


### 베스터(Vester) 템플릿 오구성이 잠재적으로 토큰 청구를 차단할 수 있음

**설명:** `WorldLibertyFinancialVester` 컨트랙트는 템플릿 `capPerUser` 값의 합계가 사용자의 총 할당량에 미치지 못할 때 사용자 토큰을 일시적으로 액세스할 수 없게 만들 수 있습니다.

사용자는 활성화 중에 전체 할당량을 베스터로 전송하지만, 소유자가 추가 템플릿을 추가하거나 기존 템플릿을 수정할 때까지 템플릿 상한(cap)으로 덮이는 부분만 다시 청구할 수 있습니다.

컨트랙트의 설계는 다음을 허용합니다.

- 사용자 할당량은 어떤 금액이든 될 수 있음 (`_activateVest`에서 설정)
- 템플릿 상한은 사용자당 최대 잠금 해제 가능한 금액을 정의함
- 템플릿 상한이 전체 사용자 할당량을 커버하는지 확인하는 유효성 검사가 없음

컨트랙트 소유자가 템플릿 사용자 상한을 잘못 구성/수정하면 사용자는 잠재적으로 토큰의 일부를 베스터 컨트랙트 내에서 청구할 수 없게 될 수 있습니다.

```solidity
function _unlockedTotal(uint8 _category, uint112 _allocation) internal view returns (uint256) {
    uint256 totalUnlocked = 0;
    uint256 remainingCap = _allocation;  // Start with full allocation

    for (uint8 i; i < count; ) {
        uint256 segmentCap = t.capPerUser < remainingCap ? t.capPerUser : remainingCap;
        uint256 unlocked = _segmentUnlocked(t, segmentCap);
        totalUnlocked += unlocked;

        // @audit remainingCap reduced by template cap, not allocation
        remainingCap -= segmentCap;

        unchecked { ++i; }
    }

    // @audit Any remaining allocation is ignored and becomes inaccessible
    return totalUnlocked;  // Missing: + remainingCap
}
```

**영향:** 잘못 구성된 템플릿은 사용자가 전체 베스팅을 완료한 후에도 청구할 수 없는 토큰으로 이어질 수 있습니다. 사용자는 템플릿 커버리지가 증가할 때까지 토큰의 일부를 청구할 수 없습니다.

**권장 완화 조치:** 템플릿 상한이 예상되는 사용자 할당량을 커버해야 한다는 점을 명확하게 문서화하는 것을 고려하십시오. 관리자가 오구성 시나리오를 방지할 수 있도록 인라인 및 인터페이스 주석을 모두 추가하십시오. 대안으로 모든 카테고리에 대해 마지막 템플릿으로 `remainder` 템플릿을 추가하는 것을 필수로 만드는 것을 고려하십시오.

**WLFI:**
커밋 [b567696](https://github.com/worldliberty/usd1-protocol/blob/b56769613b6438b62b8b4133a63fca727fdbc631/contracts/wlfi/interfaces/IWorldLibertyFinancialVester.sol#L27)에서 수정됨

**Cyfrin:** 확인함. 사용자당 고정 상한에서 백분율 할당으로 이동했습니다.


### WLFI 소유자가 직접 베스터 활성화를 통해 레거시 사용자를 DoS할 수 있음

**설명:** `WorldLibertyFinancialVester::ownerActivateVest` 함수는 정상적인 활성화 흐름을 우회하는 데 사용될 수 있으며, 잠재적으로 레거시 사용자에 대한 서비스 거부(DoS)를 유발할 수 있습니다. 소유자가 잘못된 매개변수로 베스터에서 사용자를 직접 활성화하면 사용자는 정상적인 활성화 흐름을 완료할 수 없으며 잘못된 베스팅 매개변수에 갇히게 됩니다.

컨트랙트에는 서로 조정되지 않는 두 가지 독립적인 활성화 경로가 있습니다.

정상 경로: `WLFI V2 → Registry::wlfiActivateAccount → Vester::wlfiActivateVest` (조정됨, 할당 및 카테고리에 레지스트리 데이터 사용)
우회 경로: `Owner → Vester::ownerActivateVest` (직접, 입력으로 소유자 지정 매개변수 사용)

베스터 컨트랙트는 이중 초기화를 방지하지만 경로 간의 매개변수 일관성을 검증하지 않습니다.

정상적인 활성화는 다음과 같습니다.

```solidity
// WorldLibertyFinancialV2.sol
function _activateAccount(address _account) internal {
    REGISTRY.wlfiActivateAccount(_account);                    // @note -> Mark as activated in registry
    uint8 category = REGISTRY.getLegacyUserCategory(_account);
    uint112 allocation = REGISTRY.getLegacyUserAllocation(_account);

    _approve(_account, address(VESTER), 0);
    _approve(_account, address(VESTER), allocation);           // @note -> Set allowance

    VESTER.wlfiActivateVest(_account, category, allocation);   // @note -> Activate vesting
    assert(allowance(_account, address(VESTER)) == 0);
}
```

우회된 베스팅 경로는 다음과 같습니다.

```solidity
// WorldLibertyFinancialVester.sol
function ownerActivateVest(address _user, uint8 _category, uint112 _amount)
    external
    onlyWorldLibertyOwner(msg.sender)
{
    _activateVest(_user, _category, _amount);  // @audit No coordination with Registry/WLFI V2
    // @audit any amount that is approved by user can be taken -> not registry allocation
   // @audit vesting can be in any category -> not necessarily category in registry
}

function _activateVest(address _user, uint8 _category, uint112 _amount) internal {
    UserInfo storage userInfo = $.users[_user];
    if (userInfo.initialized) {
        revert AlreadyInitialized(_user);      // @audit if user tries to activate later, he will be DOS'ed here     }
    // ... activation logic
}
```

**영향:** 소유자의 조치로 인해 레거시 사용자 활성화에 대한 서비스 거부가 발생할 수 있습니다. 레거시 사용자는 잘못된 베스팅 매개변수에 갇혀 스스로 수정할 수 없습니다.

**권장 완화 조치:** 베스팅 매개변수를 검증하고 활성화되지 않은 경우 레거시 사용자를 활성화하는 것을 고려하십시오.

```solidity
function ownerActivateVest(address _user, uint8 _category, uint112 _amount) external {
    // For legacy users: validate parameters and sync registry
    if (REGISTRY.isLegacyUser(_user)) {
        // @audit Validate parameters match registry data
        require(_category == REGISTRY.getLegacyUserCategory(_user), "CATEGORY_MISMATCH");
        require(_amount == REGISTRY.getLegacyUserAllocation(_user), "ALLOCATION_MISMATCH");

        // @audit Auto-sync registry state to maintain consistency
        if (!REGISTRY.isLegacyUserAndIsActivated(_user)) {
            REGISTRY.ownerActivateAccount(_user);
        }
    }

    _activateVest(_user, _category, _amount);
}
```

**WLFI:**
커밋 [b567696](https://github.com/worldliberty/usd1-protocol/blob/b56769613b6438b62b8b4133a63fca727fdbc631/contracts/wlfi/WorldLibertyFinancialV2.sol#L314)에서 수정됨

**Cyfrin:** 확인함. `ownerActivateVest`가 제거되었습니다.


### 투표권에서 제외된 주소가 위임자(delegatee)를 통하거나 토큰을 전송하여 투표권을 다시 얻을 수 있음

**설명:** WFI V2 토큰에는 `_excludedVotingPower` 상태를 true로 표시하여 사용자의 투표권을 제외하는 방법이 있습니다. 그 의도는 이 후에 사용자의 잔액이 더 이상 투표에 사용될 수 없도록 하는 것입니다. 현재 잔액을 address(0)에 위임하여 현재 위임자가 더 이상 사용할 수 없게 하고, isExcluded(account) == true이면 getVotes()가 0을 반환하도록 합니다.

하지만 이것은 두 가지 방법으로 우회할 수 있습니다.
- 계정은 `delegate()`를 호출하여 임의의 주소 X에 다시 위임할 수 있으며, 그러면 주소 X는 위임자로서 계정의 투표권을 사용할 수 있게 됩니다(getVotes는 위임 체크포인트를 검색함).
- 계정은 이 토큰을 자신이 제어하는 다른 주소 Y로 전송할 수 있으며, 그러면 Y는 해당 투표권을 사용할 권리를 갖게 됩니다.

이 문제는 내부 `_delegate()` 함수가 계정이 제외된 후 새 위임을 생성하는 것을 차단하지 않고, `_update()` 함수가 주소가 이미 투표에서 제외되었을 때 토큰을 전송하는 것을 차단하지 않기 때문에 발생합니다.

이와 대조적으로 계정이 블랙리스트에 오른 경우 투표권을 제거하는 프로세스는 동일하며, `notBlacklisted(_account)` 수정자를 통해 재위임 및 전송이 방지됩니다.

```solidity
    function _delegate(
        address _account,
        address _delegatee
    )
        notBlacklisted(_msgSender())
        notBlacklisted(_account)
        notBlacklisted(_delegatee)
        internal
        override
    {
        super._delegate(_account, _delegatee);
    }
```

결과적으로 사용자의 자체 투표권은 getVotes()를 통해 0을 반환하지만 위임자의 투표권은 `_delegateCheckpoints[account].latest()`를 통해 측정되며, 여기에는 이제 이 새로운 위임 후 "사용자" 투표권도 포함됩니다. 마찬가지로 토큰을 전송하면 관련 투표권도 제외되지 않은 새 주소로 전송되어 사용할 수 있게 됩니다.

이것은 사용자의 투표권이 여전히 사용 중이므로 excludedVotingPower 상태를 갖는 것의 의미를 우회합니다.

**영향:** 제외된 유권자는 여전히 투표를 위임/토큰을 전송하여 자신의 투표권을 행사할 수 있습니다.

**권장 완화 조치:** 계정의 `excludedVotingPower` 상태가 true인 경우 `_delegate()` 및 `_update()` 함수에서 되돌리는(revert) 것을 고려하십시오. 이것은 또한 제외된 계정에서의 모든 종류의 전송/소각을 차단한다는 점에 유의하십시오.

**WLFI:**
커밋 [b567696](https://github.com/worldliberty/usd1-protocol/blob/b56769613b6438b62b8b4133a63fca727fdbc631/contracts/wlfi/WorldLibertyFinancialV2.sol#L314)에서 수정됨

**Cyfrin:** 확인함.


### 일관성 없는 투표권 구현으로 인해 온체인 거버넌스 통합이 깨짐

**설명:** OpenZeppelin의 [GovernorVotesUpgradeable](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/e3ba7f6a236c55e3fb7e569ecd6043b11d567c3d/contracts/governance/extensions/GovernorVotesUpgradeable.sol#L80)은 모든 투표권 계산에 `getPastVotes`를 사용합니다.

```solidity
    /**
     * Read the voting weight from the token's built in snapshot mechanism (see {Governor-_getVotes}).
     */
    function _getVotes(
        address account,
        uint256 timepoint,
        bytes memory /*params*/
    ) internal view virtual override returns (uint256) {
        return token().getPastVotes(account, timepoint);
    }
```

[Governor](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/e3ba7f6a236c55e3fb7e569ecd6043b11d567c3d/contracts/governance/GovernorUpgradeable.sol#L668)의 `_castVote` 함수는 이 `_getVotes` 메서드를 사용하여 투표 가중치를 검색합니다.

```solidity
    /**
     * @dev Internal vote casting mechanism: Check that the vote is pending, that it has not been cast yet, retrieve
     * voting weight using {IGovernor-getVotes} and call the {_countVote} internal function.
     *
     * Emits a {IGovernor-VoteCast} event.
     */
    function _castVote(
        uint256 proposalId,
        address account,
        uint8 support,
        string memory reason,
        bytes memory params
    ) internal virtual returns (uint256) {
        _validateStateBitmap(proposalId, _encodeStateBitmap(ProposalState.Active));

@>        uint256 totalWeight = _getVotes(account, proposalSnapshot(proposalId), params);
        uint256 votedWeight = _countVote(proposalId, account, support, totalWeight, params);

       // @more code

        return votedWeight;
    }
```
WLFI의 `getVotes` 재정의(override)는 잔액과 베스팅 토큰을 모두 올바르게 포함하고 블랙리스트 및 제외된 계정을 확인하지만, `getPastVotes` 함수는 재정의되지 않았습니다.

**영향:** UI/프런트엔드는 `getVotes`를 통해 사용자에게 전체 투표권(잔액 + 베스팅)을 표시할 수 있지만 실제 투표 결과는 다른 투표권을 사용합니다. 또한 블랙리스트 및 제외된 계정도 유효한 투표권을 갖습니다. `getPastVotes`가 그러한 계정을 고려하지 않기 때문입니다.

**권장 완화 조치:** 과거 블록에 대해 `getVotes`를 사용하는 스냅샷을 구현하고 오프체인 거버넌스 및 실행 프로세스를 문서화하는 것을 고려하십시오. 또한 온체인 투표를 비활성화하기 위해 `getPastVotes`에서 되돌리는(revert) 것을 고려하십시오.

```solidity
function getPastVotes(address account, uint256 timepoint)
    public view override returns (uint256) {
    revert("WLFI: Use getVotes() at historical block via RPC");
}

function getPastTotalSupply(uint256 timepoint)
    public view override returns (uint256) {
    revert("WLFI: Use totalSupply() at historical block via RPC");
}
```

**WLFI:**
커밋 [269f5c1](https://github.com/worldliberty/usd1-protocol/commit/269f5c10e02d7dfe0985c4364bcbe803b1e8932b)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 낮은 위험 (Low Risk)


### `WorldLibertyFinancialV2.initialize()`에서 승인된 서명자에 대한 영(zero) 주소 유효성 검사 누락

**설명:** `WorldLibertyFinancialV2::initialize()` 함수는 `_authorizedSigner` 매개변수가 영 주소가 아닌지 검증하지 않습니다.

이 매개변수는 레거시 사용자가 자신의 계정을 자체 활성화할 수 있도록 하는 activateAccount() 함수에 중요합니다.

```solidity
function initialize(address _authorizedSigner) external reinitializer(/* version = */ 2) {
    __EIP712_init(name(), "2");

    V2 storage $ = _getStorage();
    _ownerSetAuthorizedSigner($, _authorizedSigner); // @audit No zero address validation
}
```

`ownerSetAuthorizedSigner`에도 동일한 문제가 존재합니다.

**영향:** `activateAccount()` 함수는 `InvalidSignature()`와 함께 항상 되돌려집니다. `ECDSA.recover()`는 유효한 서명에 대해 영 주소를 반환하지 않기 때문입니다.

**권장 완화 조치:** `initialize()` 함수에 영 주소 유효성 검사를 추가하는 것을 고려하십시오.

**WLFI:**
커밋 [b567696](https://github.com/worldliberty/usd1-protocol/blob/b56769613b6438b62b8b4133a63fca727fdbc631/contracts/wlfi/WorldLibertyFinancialV2.sol#L410)에서 수정됨

**Cyfrin:** 확인함.


### `MAX_VOTING_POWER` 변경에 대한 거버넌스 보호 없음

**설명:** `WorldLibertyFinancialV2::ownerSetMaxVotingPower()`는 소유자가 제한 없이 언제든지 최대 투표권 상한을 변경할 수 있도록 합니다. 이는 `ERC20VotesUpgradeable`을 상속하는 거버넌스 토큰이며 외부 거버넌스 시스템(OZ Governor)과 함께 사용될 가능성이 높으므로, 소유자는 투표권 상한 변경 시기를 전략적으로 조절하여 투표 결과를 조작할 수 있습니다.

```solidity
//WorldLibertyFinancialV2.sol
function ownerSetMaxVotingPower(uint256 _maxVotingPower) external onlyOwner {
    require(
        _maxVotingPower > 0 && _maxVotingPower <= (5_000_000_000 * 1 ether),
        "Invalid max voting power"
    );
    MAX_VOTING_POWER = _maxVotingPower; // @audit No governance protections
    emit SetMaxVotingPower(_maxVotingPower);
}
```

`getVotes()` 함수는 이 상한을 즉시 적용합니다.

```solidity
if (votingPower > MAX_VOTING_POWER) {
    return MAX_VOTING_POWER; // @audit Immediate capping on voting power
}
```

**영향:** 소유자는 활성 제안 중에 MAX_VOTING_POWER를 줄여 특정 제안에 반대할 수 있는 대규모 보유자를 무력화할 수 있습니다.

**권장 완화 조치:** 최대 투표권 변경을 하기 전에 충분한 통지를 제공하기 위해 타임락 메커니즘을 구현하는 것을 고려하십시오.

**WLFI:**
인지함. 우리는 지금 투표에 Snapshot만 사용하고 있으며 현재로서는 온체인 투표를 할 계획이 없습니다.

**Cyfrin:** 인지함.


### 계정 활성화 시 약한 서명 검증

**설명:** `WorldLibertyFinancialV2::activateAccount` 함수는 EIP-712 표준을 따르는 대신 서명 검증을 위해 계정 주소의 간단한 해시를 사용합니다.

컨트랙트는 설정 중에 EIP-712 인프라를 초기화하지만, 활성화 함수는 이 표준을 우회하고 기본 `keccak256(abi.encode(account))` 해시를 사용합니다. 이는 서명 해싱에 대한 확립된 보안 모범 사례에서 벗어납니다.

```solidity
function activateAccount(bytes calldata _signature) external {
    address account = _msgSender();
    bytes32 hash = keccak256(abi.encode(account)); // @account simple hash, no EIP-712

    if (authorizedSigner() != ECDSA.recover(hash, _signature)) {
        revert InvalidSignature();
    }

    _activateAccount(account);
}
```

**영향:** WLFI가 향후 여러 체인으로 확장되면 서명이 체인 간에 재생(replay)될 수 있습니다. 대안으로 컨트랙트가 새 프록시나 구현으로 마이그레이션된 경우 현재 컨트랙트에 대해 생성된 서명이 새 배포에서 작동할 수 있습니다.

또한 컨트랙트가 EIP712를 구현하므로 오프체인 시스템은 보안을 위해 EIP-712 구조화된 데이터를 기대합니다.

**권장 완화 조치:** 현재 실제적인 위험은 다음에 의해 완화됩니다.

- 이중 활성화 보호
- 단일 체인 배포 가정

그럼에도 불구하고 보안 모범 사례를 따르고 컨트랙트의 미래를 보장하기 위해 EIP-712 서명 검증을 구현하는 것을 고려하십시오.

**WLFI:**
커밋 [b567696](https://github.com/worldliberty/usd1-protocol/blob/b56769613b6438b62b8b4133a63fca727fdbc631/contracts/wlfi/WorldLibertyFinancialV2.sol#L41)에서 수정됨.

**Cyfrin:** 확인함.


### 가디언이 소유자의 긴급 일시 중지를 재정의(override)할 수 있음

**설명:** 이 컨트랙트는 소유자와 가디언 간에 대칭적인 `pause/unpause` 권한을 구현하여 소유자가 보안 또는 운영상의 이유로 의도적으로 일시 중지한 경우에도 가디언이 컨트랙트 일시 중지를 해제할 수 있도록 합니다. 이는 가디언이 소유자의 긴급 결정을 재정의할 수 있는 권한 계층 충돌을 생성하여 잠재적으로 보안 대응 및 운영 통제를 약화시킬 수 있습니다.

```solidity
function guardianUnpause() external onlyGuardian whenPaused {
    // @audit - do you think only the owner should be able to unpause?
    _unpause();
}
```
참고: 코드의 주석은 개발팀이 보안 관점에서 이 설계 선택에 플래그를 지정했음을 나타냅니다.

긴급 상황이나 보안 위반 기간 동안 소유자는 컨트랙트 상태에 대한 최종 통제권을 가져야 합니다. 컨트랙트를 일시 중지하는 것은 위험이 낮지만 일시 중지를 해제하는 것은 계층적 액세스가 필요한 고위험 작업입니다. 일반적인 보안 관행은 다음과 같습니다.

```text
Multiple parties can pause (defensive action, low risk)
Only highest authority can unpause (requires careful consideration)
```

**영향:** 가디언 재정의는 컨트랙트 일시 중지/해제 상태에 대한 소유자의 권한을 약화시킬 수 있습니다.

**권장 완화 조치:** 가디언에 대한 `unpause` 옵션을 제거하는 것을 고려하십시오.

**WLFI:**
커밋 [b567696](https://github.com/worldliberty/usd1-protocol/blob/b56769613b6438b62b8b4133a63fca727fdbc631/contracts/wlfi/WorldLibertyFinancialV2.sol#L214)에서 수정됨

**Cyfrin:** 확인함.

\clearpage
## 정보 (Informational)


### 금액을 안전하게 다운캐스트하기 위해 `SafeCast` 사용

**설명:** 금액을 안전하게 다운캐스트하려면 [SafeCast](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/math/SafeCast.sol)를 사용하거나 이 다운캐스트가 안전하다는 주석을 추가하십시오.
```solidity
wlfi/WorldLibertyFinancialRegistry.sol
64:                amount: uint112(_amounts[i]),
```

**WLFI:**
커밋 [b567696](https://github.com/worldliberty/usd1-protocol/blob/b56769613b6438b62b8b4133a63fca727fdbc631/contracts/wlfi/WorldLibertyFinancialRegistry.sol#L81C49-L81C104)에서 수정됨

**Cyfrin:** 확인함.


### 명명된 반환(named returns)을 사용할 때 쓸모없는 `return` 문 제거

**설명:** 명명된 반환을 사용할 때 쓸모없는 `return` 문을 제거하십시오.
* `WorldLibertyFinancialV2::getVotes` - L260의 마지막 `return votingPower;`

**WLFI:**
커밋 [b567696](https://github.com/worldliberty/usd1-protocol/blob/b56769613b6438b62b8b4133a63fca727fdbc631/contracts/wlfi/WorldLibertyFinancialV2.sol#L284)에서 수정됨

**Cyfrin:** 수정됨.


### `_checkNotBlacklisted`의 잘못된 오류 메시지

**설명:** 다음 함수의 오류 메시지는 `WLFI: caller is blacklisted`라고 말하지만, 확인은 호출자 주소가 아니라 입력 `account` 주소에 적용됩니다.

```solidity
  function _checkNotBlacklisted(address _account) internal view {
        require(
            _account == address(0) || !_getStorage().blacklistStatus[_account],
            "WLFI: caller is blacklisted"
        );
    }
```

**권장 완화 조치:** 오류 메시지를 `WLFI: account is blacklisted`로 변경하는 것을 고려하십시오.

**WLFI:**
커밋 [b567696](https://github.com/worldliberty/usd1-protocol/blob/b56769613b6438b62b8b4133a63fca727fdbc631/contracts/wlfi/WorldLibertyFinancialV2.sol#L440)에서 수정됨

**Cyfrin:** 확인함.


### WLF V2 토큰 컨트랙트에서 `renounceOwnership()`이 차단되어야 함

**설명:** WLF 토큰 컨트랙트의 소유자는 VESTER 및 Registry 컨트랙트에서도 많은 관리 권한을 갖습니다.

WLF 컨트랙트는 `renounceOwnership()` 함수도 있는 `Ownable2StepUpgradeable`로부터 소유권 기능을 상속합니다.

실수로 호출되면 컨트랙트 소유권이 영원히 손실됩니다.

**권장 완화 조치:** renounceOnwership()이 호출되는 것을 차단하는 것이 모범 사례입니다.

```solidity

/// @notice Explicitly disallow renouncing ownership
function renounceOwnership() public payable override onlyOwner {
     revert OwnerRequired();
}

```

**WLFI:**
수정됨.

**Cyfrin:** 확인함.


### 생성자에서 `_disableInitializers()` 누락

**설명:** `WorldLibertyFinancialV2`, `WorldLibertyFinancialVester` 및 `WorldLibertyFinancialRegistry` 컨트랙트는 업그레이드 가능하지만 생성자에서 `_disableInitializers()`를 호출하지 않습니다. 업그레이드 가능한 컨트랙트 패턴에서 이 호출은 구현(로직) 컨트랙트가 직접 초기화되는 것을 방지하기 위한 모범 사례입니다.

이는 프록시의 동작에 영향을 미치지 않지만, 특히 블록 탐색기와 같이 프록시와 구현 컨트랙트가 모두 보이는 환경에서 구현 컨트랙트의 우발적 또는 악의적인 사용을 방지하는 데 도움이 됩니다.

**권장 완화 조치:** `WorldLibertyFinancialV2` 컨트랙트의 생성자에 다음 줄을 추가하는 것을 고려하십시오.

```solidity
    _disableInitializers();
```

`WorldLibertyFinancialVester` 및 `WorldLibertyFinancialVester` 컨트랙트에 이 줄이 있는 생성자를 추가하십시오.

이는 구현 컨트랙트가 독립적으로 초기화될 수 없도록 보장합니다.

**WLFI:**
수정됨.

**Cyfrin:** 확인함.


### `ownerSetVotingPowerExcludedStatus()`가 onlyOwner 수정자를 두 번 적용함

**설명:** WLF V2 컨트랙트에서 `ownerSetVotingPowerExcludedStatus()` 함수는 `onlyOwner` 수정자를 두 번 적용합니다.
- 외부 `ownerSetVotingPowerExcludedStatus()` 함수에서 처음으로
- 동일한 호출 흐름 내의 내부 `_ownerSetVotingPowerExcludedStatus()` 함수에서 다시

`_ownerSetVotingPowerExcludedStatus()`에 대한 두 번째 onlyOwner 수정자는 불필요합니다.

**권장 완화 조치:** `_ownerSetVotingPowerExcludedStatus()` 함수에서 onlyOwner 수정자를 제거하십시오.

**WLFI:**
커밋 [b567696](https://github.com/worldliberty/usd1-protocol/blob/b56769613b6438b62b8b4133a63fca727fdbc631/contracts/wlfi/WorldLibertyFinancialV2.sol#L387)에서 수정됨

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### `calldata` 배열 길이를 캐시하지 않는 것이 더 저렴함

**설명:** [`calldata` 배열 길이를 캐시하지 않는 것이 더 저렴합니다](https://github.com/devdacian/solidity-gas-optimization?tab=readme-ov-file#6-dont-cache-calldata-length-effective-009-cheaper):
* `WorldLibertyFinancialRegistry::agentBulkInsertLegacyUsers`
```solidity
54:        uint256 usersLength = _users.length;
55:        for (uint256 i; i < usersLength; ++i) {
```

**WLFI:**
수정됨.

**Cyfrin:** 확인함.


### Solidity에서는 기본값으로 초기화하지 마십시오

**설명:** Solidity에서는 기본값으로 초기화하지 마십시오.
```solidity
WorldLibertyFinancialVester.sol
228:        uint256 totalUnlocked = 0;
```

**WLFI:**
수정됨.

**Cyfrin:** 확인함.

\clearpage

