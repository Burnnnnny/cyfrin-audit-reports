**Lead Auditors**

[Immeas](https://x.com/0ximmeas)

[MrPotatoMagic](https://x.com/MrPotatoMagic)

**Assisting Auditors**



---

# 발견 사항 (Findings)
## 낮은 위험 (Low Risk)


### `managerSplit`이 `BASIS_POINTS`를 초과하도록 잘못 설정될 수 있음

**설명:** `FeeManager`는 매니저 수수료(예: `managerSplit`)를 `<= BASIS_POINTS`(100%)인지 검증하지 않고 설정할 수 있게 합니다. 그 결과 `BASIS_POINTS`보다 큰 잘못된 값이 설정될 수 있습니다.

**영향:** `managerSplit > BASIS_POINTS`가 설정되면 수수료 계산이 비정상적이 되거나 revert될 수 있습니다. 특히 `BASIS_POINTS - managerSplit`처럼 보완적인 "protocol split"을 계산하는 로직은 언더플로우로 revert될 수 있어, 수수료 정산 경로가 깨지고 운영 장애(예: 수수료 수집 또는 분배 불가)를 일으킬 수 있습니다.

**권장 완화 조치:** `_requireValidFeeStructure`에서 매니저 수수료 설정 시 명시적인 경계 검사를 추가하십시오.
```
  function _requireValidFeeStructure(FeeStructure memory fees) private pure {
+    if (fees.managerSplit > BASIS_POINTS) revert InvalidManagerSplit();
      if (fees.performanceFee > MAX_PERFORMANCE_FEE) revert InvalidPerformanceFee();
      if (fees.establishmentFee > MAX_ESTABLISHMENT_FEE) revert InvalidEstablishmentFee();
  }
```


**Accountable:** 커밋 [`fa6f74c`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/fa6f74c23dc359176ed2dbb9faba4f0b4077e2b2)에서 수정됨

**Cyfrin:** 확인함. 이제 `managerSplit`은 `BASIS_POINTS` 미만으로 검증됩니다.


### `AccountableOpenTerm::accrueInterest`가 연체 상태를 갱신하지 않음

**설명:** `AccountableOpenTerm::accrueInterest`는 `_accrueInterest()`만 호출하고 `_updateDelinquentStatus()`는 호출하지 않습니다. 반면 `updateLateStatus()`는 이자를 누적한 뒤 연체 상태도 함께 갱신합니다.

**영향:** 서드파티(keeper/UI)가 연체 상태를 갱신하지 않은 채 이자 누적만 진행시킬 수 있습니다. 이후 다른 호출이 상태를 갱신하기 전까지 연체 상태가 오래된 값으로 남을 수 있어, `delinquencyStartTime` 갱신 지연이나 벌금 적용 시점 오류 같은 일시적 불일치가 발생할 수 있습니다. 이는 연체 상태에 의존하는 모니터링 및 자동화에 영향을 줄 수 있습니다.

**권장 완화 조치:** (a) `accrueInterest()`가 `_updateDelinquentStatus()`도 함께 호출하게 하거나, (b) keeper가 연체 상태 정확성이 필요할 때는 `updateLateStatus()`를 호출해야 한다고 문서화하거나, (c) `accrueInterest()`를 제거하고 `updateLateStatus()`만 사용하도록 하십시오(또는 그 반대).

**Accountable:** 커밋 [`fa6f74c`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/fa6f74c23dc359176ed2dbb9faba4f0b4077e2b2)에서 수정됨

**Cyfrin:** 확인함. 이제 `accrueInterest`는 `_updateDelinquentStatus()`도 호출합니다.


### `AccountableAsyncRedeemVault::maxDeposit` / `maxMint`는 이자가 누적되지 않은 `scaleFactor` 때문에 오래된 값을 반환하여 EIP-4626 준수성을 깨뜨릴 수 있음

**설명:** `AccountableOpenTerm.maxDeposit()`는 `principalAssets = debtShares * _scaleFactor`를 사용해 남은 수용량을 계산하지만, `_scaleFactor`는 `_accrueInterest()`가 실행될 때만 갱신됩니다. `maxDeposit()`는 view 함수이며 "가상 이자 누적"을 하지 않기 때문에, 오래된 `_scaleFactor`를 기반으로 값을 반환할 수 있습니다. 이로 인해 vault의 `maxMint()` / `maxDeposit()`도 현재 경제 상태와 맞지 않는 제한 값을 보고할 수 있습니다.

**영향:** 통합체나 사용자는 실제보다 높은 `maxDeposit` / `maxMint` 값을 보게 될 수 있고, 이후 상태 변경 경로에서 이자가 누적되고 용량 제한이 적용되면서 `deposit()` / `mint()`가 예상치 못하게 revert될 수 있습니다. 이는 [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626)의 다음 요구사항과 맞지 않습니다.
> MUST return the maximum amount of assets `deposit` would allow to be deposited for `receiver` and not cause a revert, which MUST NOT be higher than the actual maximum that would be accepted

**권장 완화 조치:** view 함수에서 "가상 이자 누적"을 사용하십시오. 즉 상태를 쓰지 않고도 현재 시점 기준 최신 scale factor를 계산해 `maxDeposit`과 `maxMint`의 기반이 되는 share-price/limit 계산(및 관련 preview/limit 함수들)에 사용하십시오. 이를 위해 `_accrueInterest()`를 view 부분과 state-changing 부분으로 나누는 방식을 고려할 수 있습니다.


**Accountable:** 커밋 [`5891946`](https://github.com/Accountable-Protocol/credit-vaults-internal/pull/54/commits/58919467552c7993b115bc1c30c7f8520de2c2c3)에서 수정됨

**Cyfrin:** 확인함. `_accrueInterest`는 이제 `_previewAccruedInterest`로 분리되었고, 이 함수가 `_accrueInterest`에서 호출됩니다.


### treasury 주소를 변경하기 전에 프로토콜 수수료가 자동 수집되지 않음

**설명:** `setTreasury` 함수는 기존 `treasury`를 새 주소로 갱신하는 데 사용됩니다. 하지만 treasury 주소를 바꾸기 전에 기존에 적립된 프로토콜 수수료를 현재 treasury로 자동 수집하지 않습니다. 큰 문제는 아닐 수 있으나, treasury가 의도적이든 실수든 dead address나 잘못된 주소로 바뀌면 문제가 될 수 있습니다.

```solidity
function withdrawProtocolFee(address asset) public nonReentrant onlyTreasury {
        uint256 amount = protocolFees[asset];
        if (amount > 0) {
            protocolFees[asset] = 0;
            IERC20(asset).safeTransfer(treasury, amount);

            emit Withdraw(asset, address(0), treasury, amount);
        }
    }

function setTreasury(address treasury_) public onlyOwner {
        if (treasury_ == address(0)) revert ZeroAddress();
        address oldTreasury = treasury;
        treasury = treasury_;
        emit TreasurySet(oldTreasury, treasury_);
    }
```

**권장 완화 조치:** treasury 주소를 갱신하기 전에 누적된 프로토콜 수수료를 먼저 수집하는 것을 고려하십시오. 추가로 treasury 주소 변경에 2단계 전송 절차를 도입하는 것도 고려할 수 있습니다.

**Accountable:** 인지함. 수집되지 않은 수수료가 유실될 dead address로 treasury를 바꿀 경우는 없다고 판단합니다.


### 이자 누적 누락으로 인해 연체 상태가 잘못 갱신될 수 있음

**설명:** `AccountableOpenTerm` 컨트랙트에서는 `_scaleFactor`가 `_accrueInterest` 함수에서 갱신됩니다. 따라서 정확한 `_scaleFactor`에 의존하는 모든 동작은 먼저 이자를 누적해야 합니다.

```solidity
_scaleFactor += baseInterest + delinquencyFee;
```

`_calculateRequiredLiquidity` 함수는 `_scaleFactor`를 사용해 필요한 준비금을 계산하고, 이 값은 다시 `_isDelinquent`에서 대출이 연체 상태인지 판정하는 데 사용됩니다. `_isDelinquent`가 반환한 boolean은 `_updateDelinquentStatus`가 대출 상환 상태를 최신으로 갱신하는 데 사용됩니다. 그런데 `setReserveThreshold`는 `_updateDelinquentStatus`를 호출하면서 이자를 누적하지 않습니다. 그 결과, 연체 상태는 오래된 `_scaleFactor`를 기준으로 갱신됩니다.

```solidity
function setReserveThreshold(uint256 threshold)
        external
        override(AccountableStrategy, IAccountableStrategy)
        onlyManager
    {
        if (threshold > BASIS_POINTS) revert ThresholdTooHigh();
        _loan.reserveThreshold = threshold;

        _updateDelinquentStatus();

        emit ReserveThresholdSet(threshold);
    }
```

**권장 완화 조치:** 연체 상태를 갱신하기 전에 이자를 먼저 누적하는 것을 고려하십시오.

**Accountable:** 커밋 [`809813f`](https://github.com/Accountable-Protocol/credit-vaults-internal/pull/54/commits/809813f5d12c28977c93075d459a38e5fa0014ae)에서 수정됨

**Cyfrin:** 확인함. 이제 대출이 진행 중이면 `_updateDelinquentStatus` 전에 `_accrueInterest`가 호출됩니다.

\clearpage
## 정보 (Informational)


### 사용되지 않는 에러

**설명:** `src/constants/Errors.sol`에 정의된 다음 미사용 에러들을 실제로 사용하거나 제거하는 것을 고려하십시오.

- [Line: 40](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/277d154d9faf9164c6cd32d66cf38f12a73c5087/src/constants/Errors.sol#L40)

	```solidity
	error CancelDepositRequestFailed();
	```

- [Line: 67](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/277d154d9faf9164c6cd32d66cf38f12a73c5087/src/constants/Errors.sol#L67)

	```solidity
	error NoCancelRedeemRequest();
	```

- [Line: 79](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/277d154d9faf9164c6cd32d66cf38f12a73c5087/src/constants/Errors.sol#L79)

	```solidity
	error NoQueueRequests();
	```

- [Line: 113](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/277d154d9faf9164c6cd32d66cf38f12a73c5087/src/constants/Errors.sol#L113)

	```solidity
	error InterestAlreadyClaimed();
	```

- [Line: 122](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/277d154d9faf9164c6cd32d66cf38f12a73c5087/src/constants/Errors.sol#L122)

	```solidity
	error InvalidVaultManager();
	```

- [Line: 156](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/277d154d9faf9164c6cd32d66cf38f12a73c5087/src/constants/Errors.sol#L156)

	```solidity
	error ZeroAmount();
	```

**Accountable:** 커밋 [`fa6f74c`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/fa6f74c23dc359176ed2dbb9faba4f0b4077e2b2)에서 수정됨

**Cyfrin:** 확인함.


### 사용되지 않는 상태 변수

**설명:** 상수 `FeeManager._protocolSplit`은 사용되지 않습니다. 이 미사용 변수를 제거하거나 실제로 사용하는 것을 고려하십시오.

**Accountable:** 커밋 [`fa6f74c`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/fa6f74c23dc359176ed2dbb9faba4f0b4077e2b2)에서 수정됨

**Cyfrin:** 확인함.


### 사용되지 않는 import

**설명:** `src/strategies/AccountableStrategy.sol`에 중복 import 구문이 있습니다. 제거를 고려하십시오.


- [Line: 6](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/277d154d9faf9164c6cd32d66cf38f12a73c5087/src/strategies/AccountableStrategy.sol#L6)

	```solidity
	import {RewardsType} from "../interfaces/IRewards.sol";
	```

- [Line: 8](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/277d154d9faf9164c6cd32d66cf38f12a73c5087/src/strategies/AccountableStrategy.sol#L8)

	```solidity
	import {IRewardsFactory} from "../interfaces/IRewardsFactory.sol";
	```

**Accountable:** 커밋 [`fa6f74c`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/fa6f74c23dc359176ed2dbb9faba4f0b4077e2b2)에서 수정됨

**Cyfrin:** 확인함.


### 이벤트 없이 상태가 변경됨

**설명:** 다음 함수는 중요한 상태를 변경하지만 이벤트를 발생시키지 않습니다. 오프체인 인덱서가 이 변경을 추적할 수 있도록 이벤트를 추가하는 것을 고려하십시오.

* [AccountableOpenTerm::setProposer](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/277d154d9faf9164c6cd32d66cf38f12a73c5087/src/strategies/AccountableOpenTerm.sol#L197-L199)

**Accountable:**
커밋 [`fa6f74c`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/fa6f74c23dc359176ed2dbb9faba4f0b4077e2b2)에서 수정됨

**Cyfrin:** 확인함.


### 선상환 수수료에 상한이 없음

**설명:** 선상환 수수료는 상한 없이 설정될 수 있습니다. 거버넌스가 지나치게 큰 선상환 수수료를 설정하면 조기 상환이 사실상 불가능해지거나 예기치 않은 동작 또는 revert가 발생할 수 있습니다. 이는 사용자 신뢰와 예측 가능성 측면의 리스크를 만듭니다.

선상환 수수료 설정 시 상한을 두는 것을 고려하십시오. 예: `if(prepaymentFee > MAX_PREPAYMENT_FEE)`.

**Accountable:** 커밋 [`fa6f74c`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/fa6f74c23dc359176ed2dbb9faba4f0b4077e2b2)에서 수정됨

**Cyfrin:** 확인함.


### `AccountableOpenTerm::repay`에서 `_updateDelinquentStatus`가 양쪽 분기에서 모두 호출됨

**설명:** `AccountableOpenTerm::repay`에서는 두 분기 모두에서 `_updateDelinquentStatus`가 호출됩니다.
```solidity
if (_loan.outstandingPrincipal == 0) {
    loanState = LoanState.Repaid;
    _updateDelinquentStatus();
} else {
    _updateDelinquentStatus();
}
```
다음과 같이 단순화할 수 있습니다.
```solidity
if (_loan.outstandingPrincipal == 0) {
    loanState = LoanState.Repaid;
}
_updateDelinquentStatus();
```

**Accountable:** 커밋 [`fa6f74c`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/fa6f74c23dc359176ed2dbb9faba4f0b4077e2b2)에서 수정됨

**Cyfrin:** 확인함.


### `AccountableOpenTerm` 함수들이 잘못 `View Functions` 헤더 아래에 있음

**설명:** `AccountableOpenTerm::updateLateStatus`, `accrueInterest`, `processAvailableWithdrawals` 함수들은 모두 다음 헤더 아래에 위치해 있습니다.
```solidity
// ========================================================================== //
//                          View Functions                                    //
// ========================================================================== //
```
하지만 이 함수들은 view 함수가 아니므로 다른 위치로 옮기는 것을 고려하십시오.

**Accountable:** 커밋 [`fa6f74c`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/fa6f74c23dc359176ed2dbb9faba4f0b4077e2b2)에서 수정됨

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### 상태 변수는 immutable로 선언할 수 있음

**설명:** 생성자에서만 변경되는 상태 변수는 `immutable`로 선언하면 가스를 절약할 수 있습니다. 생성자에서만 변경되는 다음 상태 변수들에 `immutable` 속성 추가를 고려하십시오.

- [AccountableVault.sol#L44](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/277d154d9faf9164c6cd32d66cf38f12a73c5087/src/vault/AccountableVault.sol#L44):

	```solidity
	    IStrategyVaultHooks public strategy;
	```

- [AccountableVault.sol#L47](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/277d154d9faf9164c6cd32d66cf38f12a73c5087/src/vault/AccountableVault.sol#L47)

	```solidity
	    uint256 public precision;
	```

 - [AccessBase.sol#15](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/0cee6b3d1713a5f5fd21412d89d3cb4da7537a16/src/access/AccessBase.sol#L15)

	```solidity
	    PermissionLevel public permissionLevel;
	```

**Accountable:** 커밋 [`fa6f74c`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/fa6f74c23dc359176ed2dbb9faba4f0b4077e2b2)에서 수정됨

**Cyfrin:** 확인함.


### 불필요한 `FeeManager::managerSplit` 호출

**설명:** `FeeManager::_collectFeeSplit`가 `else` 분기로 들어가면 `managerSplit(strategy)`를 두 번 호출합니다.
```solidity
if (managerSplit(strategy) == 0) {
    managerFee = 0;
    protocolFee = amount;
} else {
    managerFee = _split(amount, managerSplit(strategy));
    protocolFee = amount - managerFee;
}
```
이는 불필요합니다. 한 번만 호출하도록 다음과 같이 바꾸는 것을 고려하십시오.
```solidity
uint256 managerSplit = managerSplit(strategy);
if (managerSplit == 0) {
    managerFee = 0;
    protocolFee = amount;
} else {
    managerFee = _split(amount, managerSplit);
    protocolFee = amount - managerFee;
}
```

**Accountable:** 커밋 [`fa6f74c`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/fa6f74c23dc359176ed2dbb9faba4f0b4077e2b2)에서 수정됨

**Cyfrin:** 확인함.


### `AccountableAsyncRedeemVault::maxRedeem`의 불필요한 검사

**설명:** `AccountableAsyncRedeemVault::maxRedeem`의 마지막 검사는 불필요합니다. 값이 0이면 어차피 0이 반환되기 때문입니다.
```solidity
function maxRedeem(address controller) public view override returns (uint256 maxShares) {
    VaultState storage state = _vaultStates[controller];
    maxShares = state.redeemShares;
    if (maxShares == 0) return 0; // @audit-issue GAS unnecessary
}
```

**Accountable:** 커밋 [`78cd5c7`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/78cd5c7350cfc60367a2a7d7553c3e766d7064fc)에서 수정됨

**Cyfrin:** 확인함.


### `state.redeemShares == 0`일 때 `AccountableAsyncRedeemVault::maxWithdraw`를 더 최적화할 수 있음

**설명:** `AccountableAsyncRedeemVault::maxWithdraw`에서 `redeemShares` 검사를 먼저 하면 `redeemShares`가 0일 때 스토리지 읽기 하나를 줄일 수 있습니다.
```diff
  function maxWithdraw(address controller) public view override returns (uint256 maxAssets) {
      VaultState storage state = _vaultStates[controller];
+     if (state.redeemShares == 0) return 0;
      maxAssets = state.maxWithdraw;
-     if (state.redeemShares == 0) return 0;
  }
```

**Accountable:** 커밋 [`78cd5c7`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/78cd5c7350cfc60367a2a7d7553c3e766d7064fc)에서 수정됨

**Cyfrin:** 확인함.


### 이벤트를 더 일찍 발생시켜 가스를 절약할 수 있음

**설명:** `setTreasury` 함수는 `TreasurySet` 이벤트를 발생시키기 위해 `oldTreasury` 메모리 변수를 만듭니다. 하지만 상태 변경 전에 이벤트를 발생시키면 이 메모리 변수는 필요하지 않습니다.


`FeeManager.sol`
```solidity
function setTreasury(address treasury_) public onlyOwner {
        if (treasury_ == address(0)) revert ZeroAddress();
        address oldTreasury = treasury;
        treasury = treasury_;
        emit TreasurySet(oldTreasury, treasury_);
    }
```

`AccountableOpenTerm`
```solidity
function approveInterestRateChange() external onlyManager {
        uint256 pendingRate_ = pendingInterestRate;

        _accrueInterest();

        _loan.interestRate = pendingRate_;
        delete pendingInterestRate;

        _updateInterestParams();

        emit InterestRateApproved(pendingRate_);
    }
```

`AccountableStrategy.sol`
```solidity
function acceptBorrowerRole() external virtual {
        if (msg.sender != pendingBorrower) revert InvalidPendingBorrower();

        address oldBorrower = borrower;
        borrower = msg.sender;
        pendingBorrower = address(0);

        emit BorrowerChanged(oldBorrower, msg.sender);
    }
```

**권장 완화 조치:** 다음과 같이 함수를 갱신하는 방안을 고려하십시오.
````solidity
function setTreasury(address treasury_) public onlyOwner {
        if (treasury_ == address(0)) revert ZeroAddress();
        emit TreasurySet(treasury, treasury_);
        treasury = treasury_;
    }
````

**Accountable:** 커밋 [`78cd5c7`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/78cd5c7350cfc60367a2a7d7553c3e766d7064fc)에서 수정됨

**Cyfrin:** 확인함.

\clearpage
