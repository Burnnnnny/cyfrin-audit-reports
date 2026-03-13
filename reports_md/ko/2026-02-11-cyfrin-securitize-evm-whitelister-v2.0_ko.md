**Lead Auditors**

[Dacian](https://x.com/DevDacian)

[Stalin](https://x.com/0xStalin)

**Assisting Auditors**

 

---

# 발견 사항 (Findings)
## 정보 (Informational)


### 사용자 정의 값 타입 연산자 정의 지원은 Solidity 0.8.19에서 추가되었으므로 pragma가 잘못됨

**설명:** 다음 파일들은 아래 pragma를 사용합니다.
* `contracts/uniswap/permissionedPools/libraries/PermissionFlags.sol`
* `contracts/uniswap/permissionedPools/BaseAllowListChecker.sol`
```solidity
pragma solidity ^0.8.0;
```

하지만 이는 잘못되었습니다. 사용자 정의 값 타입에 대한 연산자 정의 지원은 Solidity 0.8.19에서 [추가](https://www.soliditylang.org/blog/2023/02/22/user-defined-operators/)되었기 때문입니다.

**권장 완화 조치:** 올바른 pragma를 사용하십시오.
```diff
- pragma solidity ^0.8.0;
+ pragma solidity ^0.8.19;
```

**Securitize:** 다른 컨트랙트와 일관되게 0.8.22를 사용하도록 커밋 [f938341](https://github.com/securitize-io/bc-allowlist-checker-sc/commit/f9383415cfab4a1de5372212cd131ef0d1b9fb22)에서 수정함.

**Cyfrin:** 확인함.


### `PermissionFlags::hasFlag`는 `PermissionFlag::NONE` 검사에 사용할 수 없음

**설명:** `PermissionFlags.sol`은 몇 가지 연산자 함수, 상수, 그리고 `hasFlag` 함수를 정의합니다. 마지막 두 항목을 보면:
```solidity
    PermissionFlag constant NONE = PermissionFlag.wrap(0x0000);
    PermissionFlag constant SWAP_ALLOWED = PermissionFlag.wrap(0x0001);
    PermissionFlag constant LIQUIDITY_ALLOWED = PermissionFlag.wrap(0x0002);
    PermissionFlag constant ALL_ALLOWED = PermissionFlag.wrap(0xFFFF);

    function hasFlag(PermissionFlag permissions, PermissionFlag flag) internal pure returns (bool) {
        return PermissionFlag.unwrap(and(permissions, flag)) != 0;
    }
```

이 구현의 한 가지 부작용은 `hasFlag(ANY_VALUE, PermissionFlags.NONE)`가 항상 `false`를 반환한다는 점입니다.
```solidity
hasFlag(ANY_VALUE, PermissionFlags.NONE)
// return ANY_VALUE & 0x0000 = 0x0000
// return 0x0000 != 0 → false
```

**권장 완화 조치:**
* 직접 동등성 비교를 사용하는 별도 함수를 `PermissionFlags`에 추가하십시오.
```solidity
function hasNoPermissions(PermissionFlag permissions) internal pure returns (bool) {
    return permissions == NONE;
}
```

* 또는 잘못된 사용을 방지하기 위해 입력 `flag`가 `NONE`일 때 `hasFlag`가 revert하도록 하십시오.
```diff
+ error InvalidFlagCheck();

function hasFlag(PermissionFlag permissions, PermissionFlag flag) internal pure returns (bool) {
+   if (flag == NONE) revert InvalidFlagCheck();
    return PermissionFlag.unwrap(and(permissions, flag)) != 0;
}
```

**Securitize:** 실제로 사용되지 않던 기능이어서 커밋 [0b8d506](https://github.com/securitize-io/bc-allowlist-checker-sc/commit/0b8d5061582f9dcac3b3eb2e24d37aba1de5e5bf)에서 제거함.

**Cyfrin:** 확인함.


### `PermissionFlags::hasFlag`가 `PermissionFlag::LIQUIDITY_ALLOWED`와 `PermissionFlag::SWAP_ALLOWED`에 잘못 `ALL_PERMISSIONS`를 부여함

**설명:** `PermissionFlags::hasFlag`는 `LIQUIDITY_ALLOWED` 플래그를 `ALL_ALLOWED`와 비교할 때 true를 반환합니다. 즉, 오직 유동성 추가만 허용된 계정이 모든 작업(`ALL_ALLOWED`)을 수행할 수 있는지 검사하면, 함수는 그 계정이 모든 작업 권한을 가진 것처럼 잘못 판단합니다.
- 동일한 문제는 `SWAP_ALLOWED`에도 적용됩니다.
- `SWAP_ALLOWED` 또는 `LIQUIDITY_ALLOWED` 조합을 검증할 때도, `hasFlag`는 이를 `ALL_ALLOWED`인 것처럼 판단합니다.

**영향:** 유동성 추가 또는 스왑만 허용된 계정이 모든 권한을 가진 것으로 처리되어, 허용되지 않아야 할 작업까지 수행할 수 있게 됩니다.

**개념 증명 (Proof of Concept):**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import {PermissionFlag, PermissionFlags} from "../../contracts/uniswap/permissionedPools/libraries/PermissionFlags.sol";

contract PermissionFlagsTest is Test {
    using PermissionFlags for PermissionFlag;

    // Helper to make tests more readable
    function _has(PermissionFlag permissions, PermissionFlag flag) internal pure returns (bool) {
        return permissions.hasFlag(flag);
    }

    function test_hasFlag_SWAP_ALLOWED() public pure {
        PermissionFlag p = PermissionFlags.SWAP_ALLOWED;
        // assertFalse(_has(p, PermissionFlags.NONE));
        // assertTrue(_has(p, PermissionFlags.SWAP_ALLOWED));
        // assertFalse(_has(p, PermissionFlags.LIQUIDITY_ALLOWED));

        //@audit-issue => SWAP_ALLOWED flag is given ALL_ALLOWED permissions!
        assertTrue(_has(p, PermissionFlags.ALL_ALLOWED)); // single flag != ALL
    }

    function test_hasFlag_LIQUIDITY_ALLOWED() public pure {
        PermissionFlag p;

        p = PermissionFlags.LIQUIDITY_ALLOWED;
        // assertFalse(_has(p, PermissionFlags.NONE));
        // assertFalse(_has(p, PermissionFlags.SWAP_ALLOWED));
        // assertTrue(_has(p, PermissionFlags.LIQUIDITY_ALLOWED));

        //@audit-issue => LIQUIDITY_ALLOWED flag is given ALL_ALLOWED permissions!
        assertTrue(_has(p, PermissionFlags.ALL_ALLOWED));
    }

    // Combined Flags //
    function test_hasFlag_SWAP_or_LIQUIDITY() public pure {
        PermissionFlag p = PermissionFlags.SWAP_ALLOWED | PermissionFlags.LIQUIDITY_ALLOWED;
        assertFalse(_has(p, PermissionFlags.NONE));
        assertTrue(_has(p, PermissionFlags.SWAP_ALLOWED));
        assertTrue(_has(p, PermissionFlags.LIQUIDITY_ALLOWED));

        //@audit-issue => returns true as if it had ALL_ALLOWED permissions
        assertTrue_has(p, PermissionFlags.ALL_ALLOWED));

    }
}
```

**권장 완화 조치:** `hasFlag` 함수를 다음과 같이 리팩터링하는 것을 고려하십시오.
```solidity
    function hasFlag(PermissionFlag permissions, PermissionFlag flag) internal pure returns (bool) {
        if (PermissionFlag.unwrap(flag) == 0) return false;

        return and(permissions, flag) == flag;
    }
}
```

이 수정은 제공된 플래그가 `NONE`인 경우를 처리하고, 주어진 `permissions`가 지정한 `flag`의 모든 비트를 포함하는지 올바르게 검증합니다.

**Securitize:** 실제로 사용되지 않던 기능이어서 커밋 [0b8d506](https://github.com/securitize-io/bc-allowlist-checker-sc/commit/0b8d5061582f9dcac3b3eb2e24d37aba1de5e5bf)에서 제거함.

**Cyfrin:** 확인함.


### 결합 플래그에 대한 `PermissionFlags::hasFlag` 검증이 일관되지 않음

**설명:** `PermissionFlags`는 여러 플래그 조합을 검사해 계정이 그중 하나 이상(OR) 또는 모두(ALL)를 보유하는지 검증할 수 있게 합니다.

문제는 `PermissionFlags::hasFlag`가 `SWAP_ALLOWED`와 `LIQUIDITY_ALLOWED`의 조합을 올바르게 검증하지 못해, true여야 할 상황에서 false를 반환한다는 점입니다.

**영향:** 결합 플래그를 검증하는 데 `PermissionFlags::hasFlag`를 사용하면 잘못된 판정이 발생해, 막혀야 할 실행이 허용될 수 있습니다.

**개념 증명 (Proof of Concept):**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import {PermissionFlag, PermissionFlags} from "../../contracts/uniswap/permissionedPools/libraries/PermissionFlags.sol";

contract PermissionFlagsTest is Test {
    using PermissionFlags for PermissionFlag;

    // Helper to make tests more readable
    function _has(PermissionFlag permissions, PermissionFlag flag) internal pure returns (bool) {
        return permissions.hasFlag(flag);
    }

    function test_hasFlag_SWAP_and_LIQUIDITY() public pure {
        PermissionFlag p = PermissionFlags.SWAP_ALLOWED & PermissionFlags.LIQUIDITY_ALLOWED;
        assertFalse(_has(p, PermissionFlags.NONE));
        assertFalse(_has(p, PermissionFlags.SWAP_ALLOWED));
        assertFalse(_has(p, PermissionFlags.LIQUIDITY_ALLOWED));
        assertFalse(_has(p, PermissionFlags.ALL_ALLOWED));

        assertTrue(p == p);
        //@audit-issue => hasFlag incapable of validating combined flags using AND
        assertTrue(_has(p, p));
    }
}
```

**권장 완화 조치:** `PermissionFlags::hasFlag`의 사용을 개별 플래그 검증으로 제한하는 것을 고려하십시오. 즉, 첫 번째 파라미터 `PermissionFlag permissions`가 예상되는 개별 플래그 값과 맞지 않으면 실행을 되돌리도록 하는 방식입니다.

**Securitize:** 실제로 사용되지 않던 기능이어서 커밋 [0b8d506](https://github.com/securitize-io/bc-allowlist-checker-sc/commit/0b8d5061582f9dcac3b3eb2e24d37aba1de5e5bf)에서 제거함.

**Cyfrin:** 확인함.


### `PermissionFlags::and` 함수는 `SWAP_ALLOWED` AND `LIQUIDITY_ALLOWED` 권한을 `NONE`인 것처럼 인코딩함

**설명:** `PermissionFlags::and` 함수를 사용해 계정에 두 권한이 모두 있어야 한다는 의미의 결합 권한을 계산하면, 결과 비트가 잘못 `NONE` 권한처럼 인코딩됩니다.

```
`SWAP_ALLOWED` & `LIQUIDITY_ALLOWED` => `NONE`
0x0001 & 0x0002 => 0x0000
```

**권장 완화 조치:** [*`PermissionFlags::hasFlag`가 `PermissionFlag::LIQUIDITY_ALLOWED`와 `PermissionFlag::SWAP_ALLOWED`에 잘못 `ALL_PERMISSIONS`를 부여함*](#permissionflagshasflag가-permissionflagliquidityallowed와-permissionflagswapallowed에-잘못-all_permissions를-부여함) 이슈의 권고 사항을 함께 고려하여, 계정에 부여할 권한을 조합할 때는 `and`를 피하는 것이 좋습니다. 대신 계정이 두 권한을 모두 갖는지 확인하려면 다음과 같이 하십시오.
```solidity
    function test_hasFlag_LIQUIDITY_or_SWAP() public pure {
        PermissionFlag p = PermissionFlags.LIQUIDITY_ALLOWED | PermissionFlags.SWAP_ALLOWED;
        // 0x0002          | 0x0001           = 0x0003
        assertTrue(_has(p, p));        // "Does the permission set p contain ALL the bits that are set in p?"
    }
```
따라서 `p.hasFlag(p)`를 호출하면:
- `permissions = p = 0x0003`
- `flag       = p = 0x0003`

계산은 다음과 같습니다.
```
(permissions & flag) == flag
(0x0003 & 0x0003)    == 0x0003
0x0003               == 0x0003   → true!
```

**Securitize:** 실제로 사용되지 않던 기능이어서 커밋 [0b8d506](https://github.com/securitize-io/bc-allowlist-checker-sc/commit/0b8d5061582f9dcac3b3eb2e24d37aba1de5e5bf)에서 제거함.

**Cyfrin:** 확인함.


### 제안된 볼트 화이트리스터 표준에 대한 권장 변경사항 구현

**설명:** 제안된 볼트 화이트리스터 표준에 대한 권장 [변경사항](https://ethereum-magicians.org/t/erc-proposal-vault-whitelister-interface-for-permissioned-erc-20-vaults/27627/2?u=dacian)을 구현하십시오.

**Securitize:** 커밋 [28ce1f8](https://github.com/securitize-io/bc-vault-whitelister/commit/28ce1f8fcdbaa0cbee26373ff6409e06222a8a02), [5ceab6d](https://github.com/securitize-io/bc-vault-whitelister/commit/5ceab6d815cc907fcc3b3fe372366ac4a92a560e)에서 수정함.

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### `BaseWhitelister::addOperator, removeOperator`는 `_grantRole`, `_revokeRole`를 직접 사용하고 `true`를 반환할 때만 이벤트를 발생시켜야 함

**설명:** `BaseWhitelister::addOperator, removeOperator`는 공개 함수 `grantRole, revokeRole`를 사용합니다. 하지만 이는 이상적이지 않습니다. 이유는 다음과 같습니다.
1) 이 함수들은 [modifier](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/access/AccessControlUpgradeable.sol#L141) `onlyRole(getRoleAdmin(role))`를 사용하지만, 이미 `onlyRole(DEFAULT_ADMIN_ROLE)` 수정자로 접근 제어가 적용되어 있습니다.

2) 내부적으로 `_grantRole, _revokeRole`를 호출하고 이 함수들은 [bool을 반환](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/access/AccessControlUpgradeable.sol#L203)합니다. 반환값이 `false`여도 `ProtocolAuthorized, ProtocolRevoked` 이벤트는 여전히 발생하므로, 실제로 권한이 부여되거나 철회되지 않았는데도 이벤트가 나갈 수 있습니다.

**권장 완화 조치:** `BaseWhitelister::addOperator, removeOperator`는 `_grantRole, _revokeRole`를 직접 호출하고, 반환값이 `true`일 때만 이벤트를 발생시켜야 합니다.
```solidity
    function addOperator(address operator) external onlyRole(DEFAULT_ADMIN_ROLE) notZeroAddress(operator) {
        if(_grantRole(OPERATOR_ROLE, operator)) emit ProtocolAuthorized(operator);
    }

    function removeOperator(address operator) external onlyRole(DEFAULT_ADMIN_ROLE) notZeroAddress(operator) {
        if(_revokeRole(OPERATOR_ROLE, operator)) emit ProtocolRevoked(operator);
    }
```

**Securitize:** 커밋 [c0bd66b](https://github.com/securitize-io/bc-vault-whitelister/commit/c0bd66b144d45a281b672d69eac0cc3c3c4c7dc8)에서 수정함.

**Cyfrin:** 확인함.


### `Whitelister::whitelist`는 `RegistryService::addWallet`가 이미 수행하는 검사를 중복 수행함

**설명:** `Whitelister::whitelist`는 투자자가 유효한 investor id를 갖는지, 그리고 주어진 `vaultAddress`가 아직 등록된 wallet이 아닌지를 검사합니다.

하지만 이 함수가 호출하는 `RegistryService::addWallet`는 이미 수정자 `investorExists(_id)`, `newWallet(_address)`를 통해 이러한 검사를 [수행](https://github.com/securitize-io/bc-vault-whitelister/blob/main/3-dstoken-reference/contracts/registry/RegistryService.sol#L163)합니다.

**권장 완화 조치:** 수정자 검사는 구식 텍스트 에러 메시지를 반환하고, 새 검사는 typed error를 반환합니다. 텍스트 기반 에러로 충분하다면 중복 검사를 제거하는 것을 고려하십시오. 반대로 typed error가 필요하다면 현재 코드를 유지할 수 있으며, 그 경우 가스 비용이 약간 더 높아질 뿐입니다.

**Securitize:** 인지함. 중복 검사는 호출 지점에서의 가독성과 명시성을 위해 의도적으로 유지했습니다.

\clearpage
