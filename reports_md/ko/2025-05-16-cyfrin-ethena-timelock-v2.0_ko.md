**수석 감사자**

[Dacian](https://x.com/DevDacian)

[Hans](https://x.com/hansfriese)

**보조 감사자**

[Alex Roan](https://twitter.com/alexroan)

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

---

# 결과 (Findings)
## 낮은 위험 (Low Risk)


### `EthenaTimelockController` 컨트랙트에 ETH가 남는 것을 방지하기 위해 value 매개변수가 `msg.value`와 일치하는 경우에만 실행 허용

**설명:** `EthenaTimelockController::execute` 및 `executeWhitelistedBatch`는 `msg.value`가 입력된 `value`/`values` 매개변수와 같은지 확인하지 않고 실행을 허용합니다.

이로 인해 ETH가 컨트랙트에 일시적으로 갇힐 수 있으며, `msg.value`가 0이지만 `value` 입력이 0이 아닌 후속 실행을 통해 "구조"할 수는 있습니다.

**권장되는 완화 방법:** 다음과 같이 `EthenaTimelockController`가 양수의 ETH 잔액으로 트랜잭션을 완료하지 않도록 불변성을 강화하십시오:
* `execute`에서 `msg.value != value`인 경우 revert
* `executeWhitelistedBatch`에서 `msg.value != sum(values)`인 경우 revert

모든 실행은 입력된 `msg.value`를 모두 사용해야 하며 `EthenaTimelockController` 컨트랙트에 실행으로 인한 ETH가 남아 있어서는 안 된다는 아이디어입니다.

**Ethena:** 커밋 [89d4190](https://github.com/ethena-labs/timelock-contract/commit/89d41901be3387c11c2150c19eb99883ed807d79)에서 `execute`, `executeBatch` 및 `executeWhitelistedBatch`에 이 불변성을 적용하여 수정되었습니다.

**Cyfrin:** 확인됨.


### `TimelockController::executeBatch`를 통해 재진입 보호를 우회할 수 있음

**설명:** `EthenaTimelockController::execute`는 `TimelockController::execute`를 재정의(override)하고 `nonReentrant` 수정자를 추가하여 다시 호출되는 재진입 호출을 방지합니다.

그러나 `TimelockController::executeBatch`는 재정의되지 않았으므로 해당 경로로 재진입이 여전히 발생할 수 있습니다. 재진입 회피 외에는 추가로 악용될 수 있는 것으로 보이지 않습니다.

**개념 증명 (Proof of Concept):**
`test/EthenaTimelockController.t.sol`에서 `MaliciousReentrant::maliciousExecute`를 다음과 같이 변경하십시오:
```solidity
function maliciousExecute() external {
    if (!reentered) {
        reentered = true;
        // re-enter the timelock through executeBatch
        bytes memory data = abi.encodeWithSignature("maliciousFunction()");
        address[] memory targets = new address[](1);
        targets[0] = address(this);
        uint256[] memory values = new uint256[](1);
        values[0] = 0;
        bytes[] memory payloads = new bytes[](1);
        payloads[0] = data;
        timelock.executeBatch(targets, values, payloads, bytes32(0), bytes32(0));
    }
}
```

그런 다음 관련 테스트를 실행하십시오: `forge test --match-test testExecuteWhitelistedReentrancy -vvv`. 예상된 재진입 오류가 더 이상 발생하지 않아 테스트가 실패하는 것을 확인할 수 있습니다.

**권장되는 완화 방법:** `EthenaTimelockController`에서 `TimelockController::executeBatch`를 재정의하여 `nonReentrant` 수정자를 추가한 다음 부모 함수를 호출하십시오.

**Ethena:** 커밋 [89d4190](https://github.com/ethena-labs/timelock-contract/commit/89d41901be3387c11c2150c19eb99883ed807d79#diff-8ca72e61ebf9a693737b5c9052aa3814e8b291e3d6dd0341fe88b5b5e781427bR147)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `TimelockController`는 존재하지 않는 컨트랙트에서 실행할 때 revert하지 않음

**설명:** `TimelockController::_execute`는 다음을 수행합니다:
```solidity
function _execute(address target, uint256 value, bytes calldata data) internal virtual {
    (bool success, bytes memory returndata) = target.call{value: value}(data);
    Address.verifyCallResult(success, returndata);
}
```

`target`이 존재하지 않는 컨트랙트이지만 `data`에 매개변수가 있는 유효한 예상 함수 호출이 포함된 경우 `call`은 `true`를 반환합니다. `Address.verifyCallResult`는 이 경우를 잡아내지 못합니다.

**개념 증명 (Proof of Concept):**
`test/EthenaTimelockController.sol`에 PoC 함수를 추가하십시오:
```solidity
function testExecuteNonExistentContract() public {
    bytes memory data = abi.encodeWithSignature("DONTEXIST()");
        _scheduleWaitExecute(address(0x1234), data);
}
```

실행: `forge test --match-test testExecuteNonExistentContract -vvv`

**권장되는 완화 방법:** OpenZeppelin에 이 버그를 보고했지만 현재 구현이 더 유연하다고 답변했습니다. 우리는 이 평가에 동의하지 않으며 유효한 calldata가 있지만 대상에 코드가 없을 때 `TimelockController::_execute`가 revert하지 않는 것은 올바르지 않다고 생각합니다.

**Ethena:** 커밋 [e58c547](https://github.com/ethena-labs/timelock-contract/commit/e58c547e3bcbea79d9df7121b5bb04626a2b72e0#diff-8ca72e61ebf9a693737b5c9052aa3814e8b291e3d6dd0341fe88b5b5e781427bR191-R197)에서 `data.length > 0 && target.code.length == 0`인 경우 revert하도록 `_execute`를 재정의하여 수정되었습니다.

**Cyfrin:** 확인됨.


### `EthenaTimelockController::addToWhitelist` 및 `removeFromWhitelist`가 존재하지 않는 `target` 주소에 대해 revert하지 않음

**설명:** `EthenaTimelockController::addToWhitelist` 및 `removeFromWhitelist`는 `target` 주소가 존재하지 않는 경우(코드가 없는 경우) revert해야 합니다.

**개념 증명 (Proof of Concept):** `test/EthenaTimelockController.t.sol`에 PoC 함수를 추가하십시오:
```solidity
function testAddNonExistentContractToWhitelist() public {
    bytes memory addToWhitelistData = abi.encodeWithSignature(
        "addToWhitelist(address,bytes4)", address(0x1234), bytes4(keccak256("DONTEXIST()"))
    );
    _scheduleWaitExecute(address(timelock), addToWhitelistData);
}
```

실행: `forge test --match-test testAddNonExistentContractToWhitelist -vvv`

**권장되는 완화 방법:** `target.code.length == 0`인 경우 revert하십시오.

**Ethena:** 커밋 [89d4190](https://github.com/ethena-labs/timelock-contract/commit/89d41901be3387c11c2150c19eb99883ed807d79#diff-8ca72e61ebf9a693737b5c9052aa3814e8b291e3d6dd0341fe88b5b5e781427bR6-R76)에서 코드가 없는 대상을 화이트리스트에 추가할 수 없도록 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)


### 명명된 임포트(named imports) 사용

**설명:** 명명된 임포트를 사용하십시오:
```diff
- import "@openzeppelin/contracts/governance/TimelockController.sol";
- import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
+ import {TimelockController, Address} from "@openzeppelin/contracts/governance/TimelockController.sol";
+ import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
```

**Ethena:** 커밋 [89d4190](https://github.com/ethena-labs/timelock-contract/commit/89d41901be3387c11c2150c19eb99883ed807d79#diff-8ca72e61ebf9a693737b5c9052aa3814e8b291e3d6dd0341fe88b5b5e781427bL4-R5)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 명명된 매핑(named mappings) 사용

**설명:** 키와 값의 목적을 명시적으로 나타내기 위해 명명된 매핑을 사용하십시오:
```diff
-    mapping(address => mapping(bytes4 => bool)) private _functionWhitelist;
+    mapping(address target => mapping(bytes4 selector => bool allowed)) private _functionWhitelist;
```

**Ethena:** 커밋 [89d4190](https://github.com/ethena-labs/timelock-contract/commit/89d41901be3387c11c2150c19eb99883ed807d79#diff-8ca72e61ebf9a693737b5c9052aa3814e8b291e3d6dd0341fe88b5b5e781427bL28-R29)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 초기에 `address(0)`에 `EXECUTOR_ROLE` 또는 `WHITELISTED_EXECUTOR_ROLE` 권한 부여 허용하지 않음

**설명:** 클라이언트는 처음에 `EXECUTOR_ROLE`을 닫아두고 나중에 열 수 있다고 명시했습니다.

따라서 `EthenaTimelockController::constructor`는 `executors` 또는 `whitelistedExecutors` 입력 배열의 요소 중 하나라도 `address(0)`인 경우 revert해야 합니다.

**Ethena:** 인지함; 여기서는 선택권을 유지하는 것을 선호합니다.


### 상태가 실제로 변경되는 경우에만 이벤트 방출

**설명:** `EthenaTimelockController`의 여러 함수는 단순히 스토리지에 쓰기만 하고 현재 스토리지 값을 읽어 변경 여부를 확인하지 않기 때문에 상태가 변경되지 않더라도 이벤트를 방출합니다.

이상적으로 이러한 함수는 상태가 변경되지 않은 경우 revert하거나 적어도 이벤트를 방출하지 않아야 합니다:
* `addToWhitelist`
* `removeFromWhitelist`

**Ethena:** 인지함.

\clearpage
## 가스 최적화 (Gas Optimization)


### 더 효율적인 `nonReentrant` 수정자를 위해 `ReentrancyGuardTransient` 사용

**설명:** 더 효율적인 `nonReentrant` 수정자를 위해 [`ReentrancyGuardTransient`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/ReentrancyGuardTransient.sol)를 사용하십시오:
```diff
- import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
+ import {ReentrancyGuardTransient} from "@openzeppelin/contracts/utils/ReentrancyGuardTransient.sol";

- contract EthenaTimelockController is TimelockController, ReentrancyGuard {
+ contract EthenaTimelockController is TimelockController, ReentrancyGuardTransient {
```

**Ethena:** 커밋 [89d4190](https://github.com/ethena-labs/timelock-contract/commit/89d41901be3387c11c2150c19eb99883ed807d79#diff-8ca72e61ebf9a693737b5c9052aa3814e8b291e3d6dd0341fe88b5b5e781427bR5-R19)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
