**수석 감사**

[Immeas](https://twitter.com/0ximmeas)

[Chinmay](https://x.com/dev_chinmayf)

---

# 발견 사항

## 정보성 (Informational)

### L1과 L2 토큰 간의 총 공급 한도 불일치

**설명:** OpenZeppelin의 [`ERC20VotesUpgradeable`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v5.4/contracts/token/ERC20/extensions/ERC20VotesUpgradeable.sol#L45-L47)은 다음을 강제합니다:

```solidity
function _maxSupply() internal view virtual returns (uint256) {
    return type(uint208).max;
}
```

이는 투표 체크포인트(vote‑checkpoint) 값을 208비트 이내로 유지하기 위함입니다. 결과적으로 `2^208 − 1`을 초과하는 L1 총 공급량은 표준 ERC20 구현이 `type(uint256).max`를 사용하므로 L1에서는 유효하지만 L2에서는 유효하지 않게 됩니다. 그러나 `type(uint208).max`는 현실적인 토큰 발행량보다 천문학적으로 크기 때문에 실제로는 발생할 가능성이 매우 낮습니다.

엄격한 대칭이 선호되는 경우, [`ERC20CappedUpgradeable`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v5.4/contracts/token/ERC20/extensions/ERC20CappedUpgradeable.sol)을 사용하거나 `mint()`에서 수동 `require(totalSupply() + mintAmount <= type(uint208).max)`를 통해 L1에서도 동일한 `uint208` 한도를 강제하여 두 체인의 공급 한도를 정렬하는 것을 고려하십시오.

**Linea:** 인지함.

### `L2LineaToken` 인터페이스와 구현 간의 매개변수 이름 불일치

**설명:** [`IL2LineaToken::syncTotalSupplyFromL1`](https://github.com/Consensys/audit-2025-07-linea-tokens/blob/44640f0965a5c7465b99769a5d241a9a1cb3a2ef/src/L2/interfaces/IL2LineaToken.sol#L38) 인터페이스와 그 구현체인 [`L2LineaToken::syncTotalSupplyFromL1`](https://github.com/Consensys/audit-2025-07-linea-tokens/blob/44640f0965a5c7465b99769a5d241a9a1cb3a2ef/src/L2/L2LineaToken.sol#L104) 간에 매개변수 명명에 불일치가 있습니다. 인터페이스는 (`_l1BlockTimestamp`, `_l1TotalSupply`)라는 이름을 사용하는 반면, 구현체는 계약의 상태 변수를 반영하는 더 자세한 이름(`_l1LineaTokenTotalSupplySyncTime`, `_l1LineaTokenSupply`)을 사용합니다. ABI는 호환되지만, 이러한 용어 불일치는 문서를 읽거나 바인딩을 생성할 때 혼란을 줄 수 있습니다.

구현체의 상태 필드와 일치시키고 명확하고 일관된 문서를 유지하기 위해 인터페이스와 NatSpec 주석 모두에서 `_l1LineaTokenTotalSupplySyncTime` 및 `_l1LineaTokenSupply`를 사용하는 것을 고려하십시오.

**Linea:** [PR#17](https://github.com/Consensys/linea-tokens/pull/17), 커밋 [`1296069`](https://github.com/Consensys/linea-tokens/pull/17/commits/1296069ed398e72d9a57f71a02b2ee93fbbc5e47)에서 수정되었습니다.

**Cyfrin:** 확인됨. 인터페이스 및 해당 nat-spec에서 매개변수 이름이 변경되었습니다.

### 우발적인 소유권 및 관리자 권한 포기 방지

**설명:** 상속된 `renounceOwnership()`과 `AccessControlUpgradeable`의 `renounceRole(DEFAULT_ADMIN_ROLE, msg.sender)`는 모두 마지막 권한자가 자신을 제거할 수 있게 하여, 잠재적으로 계약을 영구적으로 소유자나 관리자가 없는 상태로 만들어 `withdraw()` 또는 역할로 보호되는 작업과 같은 중요 기능을 차단할 수 있습니다.

`TokenAirdrop`에서 `renounceOwnership()`을 재정의하여 항상 되돌리도록(revert) 하고, 마찬가지로 `renounceRole`을 재정의하여 `DEFAULT_ADMIN_ROLE`이 포기되는 것을 방지하는 것을 고려하십시오.

**Linea:** [PR#19](https://github.com/Consensys/linea-tokens/pull/19), 커밋 [`babc8ca`](https://github.com/Consensys/linea-tokens/pull/19/commits/babc8ca99fe0ee7b69e53cbc0b48a3e31b9778e6) 및 [`a302e77`](https://github.com/Consensys/linea-tokens/pull/19/commits/a302e77baee0061f4d44b9805c751aea5fcd9098)에서 수정되었습니다.

**Cyfrin:** 확인됨. `renounceOwnership`이 재정의되어 되돌립니다.

### 사용자 대면 호출을 위한 비상 일시 중지 메커니즘 구현 고려

**설명:** `TokenAirdrop`과 `LineaToken` 모두 라이브 상태가 되면 예기치 않은 버그나 악용 발생 시 중단할 수 없는 중요한 작업을 노출합니다:

* `TokenAirdrop::claim`
  일시 중지 가드가 없으면 "factor" 토큰의 잘못된 계산이나 악의적인 동작(예: 결함이 있는 `balanceOf` 또는 오버플로/라운딩 악용)으로 인해 에어드랍 풀이 돌이킬 수 없을 정도로 고갈되거나 잠길 수 있습니다.

* `LineaToken::syncTotalSupplyToL2`
  이 함수는 온체인 상태를 L2로 브리징합니다. L2 업그레이드로 버그가 발생하거나 메시지 서비스가 수수료 의미를 변경하는 경우, 이를 중지할 기능이 없으면 반복 호출이 실패하거나 크로스체인 상태를 손상시킬 수 있습니다.

OpenZeppelin의 `Pausable` (`Upgradeable`)을 통합하여 소유자/관리자가 중요한 문제 발생 시 계약을 일시 중지할 수 있도록 하는 것을 고려하십시오.

**Linea:** 인지함. 이는 사용자가 항상 자신의 토큰에 액세스할 수 있도록 하기 위한 의도적인 것입니다.

### `L2LineaToken`의 사용되지 않는 AccessControl

**설명:** `L2LineaToken`은 `AccessControlUpgradeable`을 상속하고 초기화 시 `DEFAULT_ADMIN_ROLE`을 부여하지만, 함수(`mint`, `burn`, `syncTotalSupplyFromL1`) 중 어느 것도 역할 검사로 보호되지 않습니다. 결과적으로 AccessControl 메커니즘은 실제로 어떠한 권한도 강제하지 않습니다. `AccessControlUpgradeable` 제거를 고려하십시오.

**Linea:** 인지함. 향후 잊혀지지 않도록 의도적으로 남겨두었습니다.

\clearpage
