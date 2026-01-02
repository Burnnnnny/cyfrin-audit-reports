**수석 감사**

[Hans](https://twitter.com/hansfriese)

[Rodion](https://github.com/rodiontr)

**보조 감사**


---

# 발견 사항

## 중간 위험 (Medium Risk)

### 볼트 배포자의 다양한 자산 유형 간 공유 구성 매개변수로 인해 잘못된 가격 책정 및 수수료 계산 발생

**설명:** `VaultDeployer` 및 `SecuritizeVaultDeployer` 계약은 기본 자산 유형에 관계없이 배포된 모든 볼트에 적용되는 공유 구성 매개변수(`navProvider`, `feeManager`, `redemptionAddress`)를 유지합니다. `SegregatedVaultDeployer::deploy()` 또는 `SecuritizeVaultDeployer::deploy()`가 다양한 볼트 유형을 지원하기 위해 서로 다른 `assetToken` 및 `liquidationToken` 매개변수로 호출될 때, 모든 배포에서 동일한 `navProvider`가 사용됩니다. 이는 서로 다른 RWA 자산이 정확한 가치 평가를 위해 자산별 NAV 제공자를 필요로 하기 때문에 치명적인 아키텍처 결함을 생성합니다.

`SecuritizeVaultV2`에서 `navProvider.rate()`는 주식 대 자산 전환 비율을 결정하기 위해 `_convertToShares()`, `_convertToAssets()` 및 `getShareValue()`와 같은 중요한 함수에서 광범위하게 사용됩니다. 서로 다른 자산(예: 부동산 토큰 대 상품 토큰)에 대한 볼트가 동일한 NAV 제공자를 공유하면, 적어도 하나의 자산 유형에 대해 가격 계산이 부정확해집니다.

유사한 문제가 다음과 같이 존재합니다:
- `SecuritizeVaultDeployer.feeManager` - 모든 자산 유형에 동일한 수수료 로직을 적용합니다.
- `SecuritizeVaultDeployer.redemptionAddress` - 서로 다른 상환 메커니즘이 필요할 수 있는 자산에 대해 동일한 상환 계약을 사용합니다.

`SegregatedVault`는 계산에 `navProvider`를 사용하지 않으므로 이 문제의 직접적인 영향을 받지 않지만, 배포 패턴에서는 아키텍처 문제가 지속됩니다.

**파급력:** 잘못된 NAV 제공자가 있는 볼트에 자산을 예치하는 사용자는 잘못된 주식 금액을 받게 되어 경제적 손실을 입을 수 있으며, 공격자가 가치가 낮은 자산을 예치하고 가치가 높은 자산의 NAV 비율을 사용하여 계산된 주식을 받는 잠재적인 악용 기회가 발생할 수 있습니다.

```solidity
// SecuritizeVaultDeployer::deploy() 내부
BeaconProxy proxy = new BeaconProxy(
    upgradeableBeacon,
    abi.encodeWithSelector(
        SecuritizeVaultV2(payable(address(0))).initializeV2.selector,
        name,
        symbol,
        assetToken,      // 배포마다 다름
        redemptionAddress, // 모든 배포에서 동일 - 문제
        liquidationToken,
        navProvider,     // 모든 배포에서 동일 - 문제
        feeManager       // 모든 배포에서 동일 - 문제
    )
);
```

**권장 완화 방법:** 이 매개변수들은 관리자만 관리하도록 의도되었으며, 사용자가 배포 함수에서 지정하는 대신 계약에서 관리하는 이유라는 점을 이해합니다. 다음 두 가지 해결책 중 하나를 고려할 것을 권장합니다.

1. 자산별 구성을 지원하도록 볼트 배포자 아키텍처를 수정하십시오. 다음은 구현 예시입니다.

```diff
+ mapping(address => address) public assetNavProviders;
+ mapping(address => address) public assetFeeManagers;
+ mapping(address => address) public assetRedemptionAddresses;
+
+ function setAssetConfiguration(
+     address assetToken,
+     address navProvider,
+     address feeManager,
+     address redemptionAddress
+ ) external onlyRole(DEFAULT_ADMIN_ROLE) {
+     assetNavProviders[assetToken] = navProvider;
+     assetFeeManagers[assetToken] = feeManager;
+     assetRedemptionAddresses[assetToken] = redemptionAddress;
+ }
```
2. 자산 토큰과 유동 토큰의 모든 쌍에 대해 하나의 배포자를 두려는 의도라면, 배포자에 자산 토큰 주소와 유동 토큰 주소를 nav 제공자와 함께 저장하고 배포 함수에서 `assetToken` 및 `liquidToken` 매개변수를 제거하십시오.

**Securitize:** 커밋 [05044b](https://github.com/securitize-io/bc-rwa-seg-wrap-sc/commit/05044b3f10d66d82ad134fc73f712c62ba5796e2)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 낮은 위험 (Low Risk)

### 업그레이드 가능한 부모 계약에 스토리지 간격이 누락되어 스토리지 슬롯 충돌 위험 발생

**설명:** `VaultDeployer` 추상 계약은 업그레이드 가능하도록 설계되었으며 자식 계약인 `SegregatedVaultDeployer` 및 `SecuritizeVaultDeployer`에 의해 상속됩니다. 그러나 `VaultDeployer`에는 향후 업그레이드를 위해 공간을 예약하는 스토리지 간격(gap) 변수가 없습니다.

현재 `VaultDeployer`는 세 가지 상태 변수를 선언합니다:
- `address public navProvider`
- `address internal admin`
- `address public upgradeableBeacon`

자식 계약은 부모의 스토리지 뒤에 자체 상태 변수를 추가합니다:
- `SecuritizeVaultDeployer`는 `redemptionAddress` 및 `feeManager`를 추가합니다.
- `SegregatedVaultDeployer`는 현재 추가 상태 변수를 추가하지 않습니다.

업그레이드 가능한 계약에서 향후 버전에 부모 계약이 새 상태 변수를 추가하면, 해당 변수는 기존 부모 변수 바로 다음의 스토리지 슬롯에 할당됩니다. 이는 자식 계약 변수가 현재 점유하고 있는 스토리지 슬롯을 덮어쓰게 되어 스토리지 충돌 및 데이터 손상을 초래합니다.

`BaseContract`(조부모)는 `uint256[50] private __gap`으로 스토리지 간격을 적절하게 구현하지만, 중간 `VaultDeployer` 계약은 자체 향후 확장을 위한 공간을 예약하지 않음으로써 이 패턴을 깨뜨립니다.

**파급력:** `VaultDeployer`의 향후 버전에서 새 상태 변수를 추가하면 자식 계약 스토리지 슬롯을 덮어써 데이터 손상을 일으키고 배포된 계약을 사용할 수 없게 만들 수 있습니다.

**권장 완화 방법:** `VaultDeployer` 계약에 스토리지 간격 변수를 추가하여 향후 업그레이드를 위한 공간을 예약하십시오. 기존 스토리지 간격 또는 ERC-7201 네임스페이스 스토리지를 선택하십시오:

**옵션 1: 기존 스토리지 간격**
```diff
abstract contract VaultDeployer is IVaultDeployer, BaseContract {
    bytes32 public constant AGGREGATOR_ROLE = keccak256("AGGREGATOR_ROLE");

    address public navProvider;
    address internal admin;
    address public upgradeableBeacon;

+   // Reserve storage slots for future VaultDeployer upgrades
+   uint256[47] private __gap;

    // ... rest of contract
}
```

**옵션 2: ERC-7201 네임스페이스 스토리지**
```diff
abstract contract VaultDeployer is IVaultDeployer, BaseContract {
    /// @custom:storage-location erc7201:securitize.storage.VaultDeployer
    struct VaultDeployerStorage {
        address navProvider;
        address admin;
        address upgradeableBeacon;
    }

    // keccak256(abi.encode(uint256(keccak256("securitize.storage.VaultDeployer")) - 1)) & ~bytes32(uint256(0xff))
    bytes32 private constant VAULT_DEPLOYER_STORAGE_LOCATION = 0x...;

    function _getVaultDeployerStorage() private pure returns (VaultDeployerStorage storage $) {
        assembly {
            $.slot := VAULT_DEPLOYER_STORAGE_LOCATION
        }
    }

    // Update all variable access to use the storage struct
    // ... rest of contract
}
```

**Securitize:** 커밋 [3048c3](https://github.com/securitize-io/bc-securitize-vault-sc/commit/3048c3ee21d18fe3a30c4d55ec96332f379bbcdc)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 생성자에서 비활성화되지 않은 업그레이드 가능한 계약 초기화로 구현 계약 초기화 허용

**설명:** `SegregatedVault` 계약은 `UUPSUpgradeable`을 확장하는 `BaseContract`를 상속하여 UUPS(Universal Upgradeable Proxy Standard) 패턴을 사용하는 업그레이드 가능한 계약으로 설계되었습니다. 그러나 구현 계약은 생성자에서 초기화(initializer)를 비활성화하지 않아 보안 취약점을 생성합니다.

UUPS 업그레이드 가능 계약에서 구현 계약 자체의 직접 초기화를 방지하려면 생성자에서 구현 계약의 초기화를 비활성화해야 합니다. 이 보호 조치가 없으면 공격자가 구현 계약에서 직접 `SegregatedVault::initialize`를 호출하여 관리자 권한을 부여하고 의도된 프록시 기반 업그레이드 메커니즘을 중단시킬 수 있습니다.

`SegregatedVault::initialize` 함수는 `DEFAULT_ADMIN_ROLE`을 `msg.sender`에게 부여하는데, 구현에서 직접 호출되면 공격자가 됩니다. 이를 통해 역할 관리 및 계약 업그레이드와 같은 관리자 전용 기능에 무단으로 액세스할 수 있습니다.

코드베이스의 다른 업그레이드 가능한 계약에도 유사한 문제가 존재합니다:
- bc-securitize-vault-sc 모듈의 `SecuritizeVaultV2`는 생성자 초기화 비활성화가 없습니다.
- bc-securitize-vault-sc 모듈의 `SecuritizeVault`는 생성자 초기화 비활성화가 없습니다.
- `RWASegWrap`은 생성자 초기화 비활성화가 없습니다.
- `SecuritizeRWASegWrap`은 생성자 초기화 비활성화가 없습니다.

이 모든 계약은 `BaseContract`를 상속하고 구현 계약을 적절히 보호하지 않고 UUPS 업그레이드 가능성을 구현하는 동일한 패턴을 따릅니다.

**파급력:** 공격자는 구현 계약을 직접 초기화하여 무단 관리자 권한을 얻고 모든 프록시 인스턴스에 대한 업그레이드 메커니즘을 손상시킬 수 있습니다.

**권장 완화 방법:** 구현 계약의 직접 초기화를 방지하기 위해 초기화를 비활성화하는 생성자를 추가하십시오:

```diff
contract SegregatedVault is ERC4626Upgradeable, ISegregatedVault, IVaultAccessControl, BaseContract {

+   /// @custom:oz-upgrades-unsafe-allow constructor
+   constructor() {
+       _disableInitializers();
+   }
```

영향을 받는 다른 업그레이드 가능한 계약(`SecuritizeVaultV2`, `SecuritizeVault`, `RWASegWrap`, `SecuritizeRWASegWrap`)에도 동일한 수정 사항을 적용하십시오.

**Securitize:** 커밋 [1261ec](https://github.com/securitize-io/bc-securitize-vault-sc/commit/1261ec95e1f080c628193e19a00bc9e6808ffbaa) 및 [1a2f4c](https://github.com/securitize-io/bc-rwa-seg-wrap-sc/commit/1a2f4c5a4d52e297e2662c15ba50aae30238c093)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `setVault` 함수의 불완전한 매핑 업데이트로 인해 볼트 주소 불일치 발생

**설명:** `RWASegWrap::setVault` 함수를 사용하면 관리자가 특정 볼트 ID에 대한 볼트 주소를 업데이트할 수 있지만, 볼트 주소와 볼트 ID 간의 양방향 매핑을 제대로 유지하지 못합니다. 이 함수는 `vaults[id] = vault`만 업데이트하고 `vaultIds` 매핑은 업데이트하지 않습니다. `vaultIds` 매핑은 새 볼트 주소를 볼트 ID에 매핑하고 이전 볼트 주소에 대한 매핑을 지워야 합니다.
이로 인해 이전 볼트 주소는 `vaultIds` 매핑에서 여전히 볼트 ID에 매핑되어 있는 반면, 새 볼트 주소는 시스템에서 인식되지 않는 불일치가 발생합니다. `vaultIds` 매핑은 `isValidVault` 및 `getAssetId`와 같은 함수에서 볼트 검증에 중요하며, `_addVault`에서 중복 볼트 등록을 방지하는 데에도 사용됩니다. 동일한 문제가 `RWASegWrap`을 상속하고 동일한 `setVault` 구현을 사용하는 `SecuritizeRWASegWrap` 계약에도 존재합니다.

**파급력:** 불완전한 매핑 업데이트로 인해 새 볼트 주소가 시스템에서 유효한 것으로 인식되지 않는 반면, 이전 볼트 주소는 더 이상 활성 상태가 아님에도 여전히 유효한 것처럼 보일 수 있으므로 볼트 작업이 실패하거나 예상치 못한 동작을 할 수 있습니다.

**권장 완화 방법:** 두 매핑을 모두 적절하게 유지 관리하도록 `setVault` 함수를 업데이트하십시오:

```diff
function setVault(
    address vault,
    uint256 id
) public virtual override onlyRole(DEFAULT_ADMIN_ROLE) idNotZero(id) recognizedVault(id) addressNotZero(vault) {
    address oldVault = vaults[id];
    address investorWallet = vaultIdOwnerWallets[id];
    vaults[id] = vault;
+   delete vaultIds[oldVault];
+   vaultIds[vault] = id;
    emit VaultUpdated(oldVault, vault, id, investorWallet);
}
```

**Securitize:** 커밋 [468bae](https://github.com/securitize-io/bc-securitize-vault-sc/commit/468bae9777ad341c700dcf30caec057f6ba101c3)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 안전하지 않은 ERC20 작업으로 인해 비표준 토큰에서 예상치 못한 실패 발생 가능

**설명:** 프로토콜은 OpenZeppelin의 `SafeERC20` 라이브러리 래퍼 대신 직접 `IERC20.transferFrom` 및 `IERC20.approve` 함수 호출을 사용합니다. 이는 부울 값을 반환하지 않거나 비표준 구현이 있는 토큰과 호환성 문제를 일으킵니다.

주요 발생 지점은 `RWASegWrap::_pullAndApprove`이며, 이는 입금 및 발행 작업 중 사용되는 중요한 내부 함수입니다. 사용자가 `depositById` 또는 `mintById`를 호출하면 래퍼 계약은 호출자로부터 자산을 전송하고 볼트가 사용할 수 있도록 승인을 시도합니다. USDT와 같은 토큰의 경우 `approve` 함수가 0이 아닌 허용량에서 다른 0이 아닌 값으로 승인하려고 할 때 실패하거나, `transferFrom`이 부울 값을 반환하지 않아 트랜잭션이 예상치 못하게 되돌려질(revert) 수 있습니다.

유사한 안전하지 않은 작업이 다음에서 발견됩니다:
- `SecuritizeVault::liquidate` - `IERC20Metadata(asset()).approve(address(redemption), assets)` 사용
- `SecuritizeVaultV2::_liquidateTo` - `IERC20Metadata(asset()).approve(address(redemption), assets)` 사용

이 함수들은 자산 예치, 주식 발행 및 청산을 포함한 핵심 프로토콜 작업을 처리하므로 정상적인 프로토콜 기능에 매우 중요합니다.

**파급력:** 사용자는 비표준 ERC20 구현이 있는 토큰을 사용할 때 자산을 예치하거나 주식을 청산할 수 없어 트랜잭션 실패 및 사용자 경험 저하로 이어질 수 있습니다.

**권장 완화 방법:** 모든 직접 `IERC20` 호출을 `SafeERC20` 등가물로 교체하십시오. `RWASegWrap.sol`에 SafeERC20 임포트를 추가하고 안전하지 않은 작업을 업데이트하십시오:

```diff
// RWASegWrap.sol에 임포트 추가
+import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract RWASegWrap is ... {
+   using SafeERC20 for IERC20;

    function _pullAndApprove(address caller, uint256 assets, uint256 vaultId) internal {
        ISegregatedVault vault = ISegregatedVault(vaults[vaultId]);
-       bool success = IERC20(asset).transferFrom(caller, address(this), assets);
-       if (!success) {
-           revert AssetTransferFailed();
-       }
-       success = IERC20(asset).approve(address(vault), assets);
-       if (!success) {
-           revert AssetApprovalFailed();
-       }
+       IERC20(asset).safeTransferFrom(caller, address(this), assets);
+       IERC20(asset).forceApprove(address(vault), assets);
    }
}
```

SecuritizeVault 계약의 경우:

```diff
// SecuritizeVault.liquidate 내부
-IERC20Metadata(asset()).approve(address(redemption), assets);
+IERC20Metadata(asset()).forceApprove(address(redemption), assets);

// SecuritizeVaultV2._liquidateTo 내부
-bool success = IERC20Metadata(asset()).approve(address(redemption), assets);
-if (!success) {
-    revert AssetApprovalFailed();
-}
+IERC20Metadata(asset()).forceApprove(address(redemption), assets);
```

**Securitize:** 커밋 [b64b27](https://github.com/securitize-io/bc-rwa-seg-wrap-sc/commit/b64b2772c6584d9133763a1c128a32d2df9d5ff0)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `getUnderlyingAsset`의 함수 이름과 문서 간의 불일치

**설명:** `RWASegWrap::getUnderlyingAsset` 함수는 이름과 문서가 일치하지 않습니다. 함수 이름 `getUnderlyingAsset`은 기본 자산 주소(RWA 토큰)를 반환해야 함을 시사하지만, 인터페이스 문서는 "Gets the vault address for a given asset ID(주어진 자산 ID에 대한 볼트 주소 가져오기)"라고 명시하여 볼트 계약 주소를 반환해야 함을 나타냅니다.

현재 구현은 볼트 계약 주소인 `vaults[id]`를 반환하여 문서와 일치하지만 함수 이름과 모순됩니다. 이는 개발자가 이름에 따라 기본 자산 주소를 반환할 것으로 예상할 수 있지만 실제로는 볼트 주소를 반환하므로 코드베이스에 혼란을 야기합니다.

```solidity
// 인터페이스 문서: "Gets the vault address for a given asset ID"
// 그러나 함수 이름은 기본 자산을 반환함을 시사함
function getUnderlyingAsset(uint256 id) external view returns (address);

// 구현은 볼트 주소를 반환하여 문서는 일치하지만 이름과는 불일치함
function getUnderlyingAsset(uint256 id) public virtual view override returns (address) {
    return vaults[id]; // 볼트 계약 주소 반환
}
```

ERC4626 볼트 아키텍처에서는 다음과 같은 명확한 구분이 있습니다:
- 볼트 주소: ERC4626 볼트(공유 토큰 계약)의 계약 주소
- 기본 자산 주소: 볼트가 예치금으로 허용하는 토큰의 주소 (`vault.asset()`을 통해 얻을 수 있음)

`SecuritizeRWASegWrap`은 `RWASegWrap`을 상속하고 이 함수를 재정의하지 않으므로 동일한 문제가 존재합니다.

**파급력:** 함수 이름과 문서 간의 불일치는 함수의 의도된 동작에 대한 혼란을 야기하여 개발자가 기본 자산 주소를 기대하지만 볼트 주소를 받는 통합 오류로 이어질 수 있습니다.

**권장 완화 방법:** 불일치를 해결하기 위해 다음 접근 방식 중 하나를 선택하십시오:

옵션 1 - 문서와 일치하도록 함수 이름 변경:
```diff
/**
 * @notice Gets the vault address for a given asset ID.
 * @dev Returns zero address if not found.
 */
- function getUnderlyingAsset(uint256 id) external view returns (address);
+ function getVaultAddress(uint256 id) external view returns (address);
```

옵션 2 - 함수 이름과 일치하도록 문서를 업데이트하고 구현 수정:
```diff
/**
- * @notice Gets the vault address for a given asset ID.
+ * @notice Gets the underlying asset address for a given asset ID.
 * @dev Returns zero address if not found.
 */
function getUnderlyingAsset(uint256 id) external view returns (address);

// 그리고 구현 업데이트:
function getUnderlyingAsset(uint256 id) public virtual view override returns (address) {
-   return vaults[id];
+   return vaults[id] != address(0) ? IERC4626(vaults[id]).asset() : address(0);
}
```

**Securitize:** [23f879](https://github.com/securitize-io/bc-rwa-seg-wrap-sc/commit/23f87990fb28f6c74344cae415ef1f7e9617da4c)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 전송 기능의 잘못된 허용량 확인으로 사용자가 자신의 토큰을 전송할 수 없음

**설명:** 프로토콜은 `RWASegWrap` 및 `SecuritizeRWASegWrap` 계약에서 사용자가 자신의 토큰을 전송할 때도 항상 허용량을 확인하여 표준 토큰 전송 관행을 위반하는 전송 함수를 구현합니다.

`RWASegWrap::safeTransferFrom()`에서 448-450행은 항상 허용량을 확인합니다:
```solidity
uint256 currentAllowance = allowance(from, _msgSender(), id);
if (currentAllowance < value) {
    revert ERC1155MissingApprovalForAll(_msgSender(), from);
}
```

마찬가지로 `RWASegWrap::transferFrom()`은 `ISegregatedVault(vaults[id]).internalTransferFrom(from, to, _msgSender(), value)`를 호출하며, 이는 내부적으로 `_spendAllowance(from, spender, value)`를 호출합니다. 여기서 spender는 `_msgSender()`이므로 자체 전송에 대해서도 허용량 확인을 강제합니다.

표준 토큰 구현 관행에 따르면 허용량 확인은 보낸 사람이 토큰 소유자가 아닌 경우에만 발생해야 합니다. 올바른 동작은 `from != sender`일 때만 승인을 확인하는 OpenZeppelin의 구현에서 볼 수 있습니다:

```solidity
// OpenZeppelin의 ERC1155Upgradeable
address sender = _msgSender();
if (from != sender && !isApprovedForAll(from, sender)) {
    revert ERC1155MissingApprovalForAll(sender, from);
}
```

`SecuritizeRWASegWrap`은 `RWASegWrap`을 상속하므로 두 전송 함수 모두에 대해 동일한 비준수 동작을 갖습니다.

**파급력:** 사용자는 자신에게 허용량을 부여하기 위해 먼저 approve를 호출하지 않고는 자신의 토큰을 전송할 수 없어 불필요한 마찰을 일으키고 표준 토큰 전송 기대를 위반합니다.

**권장 완화 방법:** 표준 토큰 구현 관행에 따라 보낸 사람이 토큰 소유자가 아닌 경우에만 발생하도록 허용량 확인을 수정하십시오:

```diff
function safeTransferFrom(
    address from,
    address to,
    uint256 id,
    uint256 value,
    bytes memory data
) public virtual override whenNotPaused idNotZero(id) recognizedVault(id) {
    if (to == address(0)) {
        revert ERC1155InvalidReceiver(address(0));
    }
    if (from == address(0)) {
        revert ERC1155InvalidSender(address(0));
    }
    uint256 vaultId = getVaultId(from);
    if (id != vaultId) {
        revert InvestorVaultMismatch(id, from);
    }

-   uint256 currentAllowance = allowance(from, _msgSender(), id);
-   if (currentAllowance < value) {
-       revert ERC1155MissingApprovalForAll(_msgSender(), from);
-   }
+   address sender = _msgSender();
+   if (from != sender) {
+       uint256 currentAllowance = allowance(from, sender, id);
+       if (currentAllowance < value) {
+           revert ERC1155MissingApprovalForAll(sender, from);
+       }
+   }

    uint256 currentBalance = balanceOf(from, id);
    if (currentBalance < value) {
        revert ERC1155InsufficientBalance(from, currentBalance, value, id);
    }

    emit TransferSingle(_msgSender(), from, to, id, value);
    ISegregatedVault(vaults[id]).internalTransferFrom(from, to, _msgSender(), value);
    ERC1155Utils.checkOnERC1155Received(_msgSender(), from, to, id, value, data);
}
```

`internalTransferFrom` 함수도 `from != spender`인 경우에만 `_spendAllowance`를 조건부로 호출하도록 업데이트해야 합니다.

**Securitize:** 커밋 [dd0035](https://github.com/securitize-io/bc-securitize-vault-sc/commit/dd0035fa0e700650b948191a70e7d6f9931a828e) 및 [4f0722](https://github.com/securitize-io/bc-rwa-seg-wrap-sc/commit/4f07223535989cc4fe99cfb22648a98addc61539)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 초기화 중 `notEmptyURI` 수정자 누락

**설명:** 현재 `initialize()` 함수에는 빈 URI를 확인하고 비어 있는 경우 트랜잭션을 되돌리는 `notEmptyURI` 수정자가 없습니다:

```
//RWASegWrap.sol#L98-103
    modifier notEmptyUri(string memory newUri) {
        if (bytes(newUri).length == 0) {
            revert EmptyUriInvalid();
        }
        _;
    }
```

```
//RWASegWrap.sol#L131-147
   function initialize(
        string memory baseNameArg,
        string memory baseSymbolArg,
        string memory uriArg,
        address liquidationAssetArg,
        address assetArg,
        address vaultDeployerArg
    )
    public
    virtual
    override
    onlyProxy
    initializer
    addressNotZero(liquidationAssetArg)
    addressNotZero(assetArg)
    addressNotZero(vaultDeployerArg)
    {

```


**파급력:** 검증 불충분, 초기화 과정에서 `projectURI`가 설정되지 않을 수 있습니다.

**권장 완화 방법:** 빈 URI를 확인하는 `notEmptyURI` 수정자를 추가하십시오.

**Securitize**
커밋 [0946fb](https://github.com/securitize-io/bc-securitize-vault-sc/commit/0946fbac2f4dd161c31c2cc8125c1203d6b46590)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)

### 잘못된 함수 문서

**설명:** `SegregatedVault::addAggregator` 함수에는 함수 설명과 매개변수 문서 모두에 "Aggregators" 대신 "Operators"를 언급하는 잘못된 문서가 포함되어 있습니다. 주석은 "Operators can deposit and redeem. Emits AggregatorAdded event(운영자는 입금 및 상환할 수 있습니다. AggregatorAdded 이벤트 방출)"라고 명시되어 있지만 aggregator 기능을 설명해야 하며, 매개변수 설명은 "The address to which the Operator role will be granted(운영자 역할이 부여될 주소)"라고 되어 있지만 Aggregator 역할을 참조해야 합니다.

이는 `addOperator` 함수 문서에서 복사-붙여넣기 오류로 보입니다. 동일한 문제가 `SecuritizeVaultV2::addAggregator`에도 존재합니다.

또한 `SegregatedVault::revokeAggregator`에 또 다른 불일치가 있는데, 주석은 "Revokes the Operator role from an account. Emits a OperatorRevoked event(계정에서 운영자 역할을 취소합니다. OperatorRevoked 이벤트 방출)"라고 명시되어 있지만 Aggregator 역할 및 AggregatorRevoked 이벤트를 참조해야 합니다.

**파급력:** 잘못된 문서는 개발자와 감사자에게 의도된 역할 권한 및 기능에 대해 혼동을 주어 잠재적으로 통합 오류 또는 보안 오해로 이어질 수 있습니다.

**권장 완화 방법:** aggregator 역할 기능 및 매개변수를 올바르게 설명하도록 문서를 업데이트하십시오:

```diff
/**
 * @dev Grants the aggregator role to an account.
- * Operators can deposit and redeem. Emits AggregatorAdded event
+ * Aggregators can manage vault operations and perform deposits/redeems on behalf of the protocol. Emits AggregatorAdded event
 *
- * @param account The address to which the Operator role will be granted.
+ * @param account The address to which the Aggregator role will be granted.
 */
function addAggregator(address account) external addressNotZero(account) onlyRole(DEFAULT_ADMIN_ROLE) {
    _grantRole(AGGREGATOR_ROLE, account);
    emit AggregatorAdded(account);
}
```

`SegregatedVault::revokeAggregator` 및 `SecuritizeVaultV2::addAggregator` 함수에도 유사한 수정을 적용하십시오.

**Securitize:** 커밋 [0e881e](https://github.com/securitize-io/bc-securitize-vault-sc/commit/0e881e38f9ec600d7ee5b1b7555a4ab81eaa04d1) 및 [402daa](https://github.com/securitize-io/bc-rwa-seg-wrap-sc/commit/402daa31d16471b24ea42810d613064b38256a00)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 단일 단계 관리자 전송으로 관리 제어의 영구적 손실 위험 발생

**설명:** `SegregatedVault::changeAdmin` 함수는 새 주소에 즉시 관리자 권한을 부여하고 단일 트랜잭션으로 현재 관리자로부터 권한을 취소하는 단일 단계 관리자 전송 메커니즘을 구현합니다. 이 패턴은 새 관리자가 실제로 계약을 제어할 수 있는지에 대한 검증이 없기 때문에 잘못된 주소가 제공되면 관리 제어가 영구적으로 손실될 수 있는 위험을 생성합니다.

동일한 안전하지 않은 단일 단계 관리자 전송 패턴이 코드베이스의 여러 계약에 걸쳐 구현되어 있습니다:
- `VaultDeployer::changeAdmin`은 즉시 관리자 역할 전송을 수행합니다.
- `SecuritizeVaultV2::changeAdmin`은 동일한 단일 단계 접근 방식을 사용합니다.
- `SecuritizeVault::changeAdmin`도 즉시 전송을 구현합니다.

관리자 역할은 운영자, 청산인 및 수집기(aggregator)를 추가/취소할 수 있는 기능과 볼트 업그레이드 및 구성에 대한 제어를 포함하여 광범위한 권한을 가집니다. 관리자 액세스 권한을 상실하면 이러한 관리 기능에 영구적으로 액세스할 수 없게 되어 잠재적으로 프로토콜 작업이 중단되고 비상 대응이 방지될 수 있습니다.

**파급력:** 관리자 전송 중 오타나 잘못된 주소가 발생하면 계약에 대한 관리 제어가 영구적으로 손실되어 중요 관리 기능에 액세스할 수 없게 되고 잠재적으로 비용이 많이 드는 재배포 절차가 필요할 수 있습니다.

**권장 완화 방법:** 새 관리자가 역할을 명시적으로 수락해야 하는 2단계 관리자 전송 패턴을 구현하십시오:

```diff
+ address public pendingAdmin;
+
+ function transferAdmin(address newAdmin) external addressNotZero(newAdmin) onlyRole(DEFAULT_ADMIN_ROLE) {
+     pendingAdmin = newAdmin;
+     emit AdminTransferStarted(msg.sender, newAdmin);
+ }
+
+ function acceptAdmin() external {
+     if (msg.sender != pendingAdmin) {
+         revert NotPendingAdmin();
+     }
+     address oldAdmin = msg.sender;
+     _grantRole(DEFAULT_ADMIN_ROLE, pendingAdmin);
+     _revokeRole(DEFAULT_ADMIN_ROLE, oldAdmin);
+     delete pendingAdmin;
+     emit AdminChanged(pendingAdmin);
+ }
+
- function changeAdmin(address newAdmin) external addressNotZero(newAdmin) onlyRole(DEFAULT_ADMIN_ROLE) {
-     _grantRole(DEFAULT_ADMIN_ROLE, newAdmin);
-     _revokeRole(DEFAULT_ADMIN_ROLE, msg.sender);
-     emit AdminChanged(newAdmin);
- }
```

`VaultDeployer`, `SecuritizeVaultV2` 및 `SecuritizeVault` 계약에도 유사한 변경 사항을 적용하십시오.

**Securitize:** 인지함.

**Cyfrin:** 인지함.

### 기본 전송 함수 호출 전 safeTransferFrom의 중복 잔액 확인

**설명:** `RWASegWrap::safeTransferFrom()` 함수는 기본 전송 함수를 호출하기 전에 452-454행에서 불필요한 잔액 확인을 수행합니다:

```solidity
uint256 currentBalance = balanceOf(from, id);
if (currentBalance < value) {
    revert ERC1155InsufficientBalance(from, currentBalance, value, id);
}
```

이 잔액 확인은 후속 호출인 `ISegregatedVault(vaults[id]).internalTransferFrom(from, to, _msgSender(), value)`가 내부적으로 ERC20 `_transfer()` 함수를 호출하고, 이 함수가 이미 동일한 잔액 검증을 수행하기 때문에 중복입니다. ERC20 `_update()` 함수(`_transfer()`가 호출함)에는 정확히 동일한 확인이 포함되어 있습니다:

```solidity
uint256 fromBalance = $._balances[from];
if (fromBalance < value) {
    revert ERC20InsufficientBalance(from, fromBalance, value);
}
```

잔액이 부족하면 ERC20 메커니즘이 자동으로 `ERC20InsufficientBalance`와 함께 되돌려지므로 래퍼 수준의 잔액 확인은 중복됩니다.

**파급력:** 중복 잔액 확인은 추가적인 안전성을 제공하지 않으면서 불필요한 가스 소비와 코드 복잡성을 초래합니다.

**권장 완화 방법:** 중복 잔액 확인을 제거하십시오:

```diff
function safeTransferFrom(
    address from,
    address to,
    uint256 id,
    uint256 value,
    bytes memory data
) public virtual override whenNotPaused idNotZero(id) recognizedVault(id) {
    if (to == address(0)) {
        revert ERC1155InvalidReceiver(address(0));
    }
    if (from == address(0)) {
        revert ERC1155InvalidSender(address(0));
    }
    uint256 vaultId = getVaultId(from);
    if (id != vaultId) {
        revert InvestorVaultMismatch(id, from);
    }

    uint256 currentAllowance = allowance(from, _msgSender(), id);
    if (currentAllowance < value) {
        revert ERC1155MissingApprovalForAll(_msgSender(), from);
    }

-   uint256 currentBalance = balanceOf(from, id);
-   if (currentBalance < value) {
-       revert ERC1155InsufficientBalance(from, currentBalance, value, id);
-   }

    emit TransferSingle(_msgSender(), from, to, id, value);
    ISegregatedVault(vaults[id]).internalTransferFrom(from, to, _msgSender(), value);
    ERC1155Utils.checkOnERC1155Received(_msgSender(), from, to, id, value, data);
}
```

**Securitize:** 커밋 [13955e](https://github.com/securitize-io/bc-rwa-seg-wrap-sc/commit/13955e2b094b19d8bfb71020498c4607c127627f)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `setApprovalForAll()` 함수가 자식 계약에서 이중으로 초기화됨

**설명:** 현재 부모 `RWASegWrap` 및 자식 `SecuritizeRWASegWrap` 계약 모두에서 ERC1155 `setApprovalForAll()` 함수의 이중 함수 초기화가 있습니다:

```
    // @inheritdoc IERC1155
    function setApprovalForAll(address, bool) public virtual pure override {
        revert FeatureNotSupported();
    }

```


```
    // @inheritdoc IERC1155
    function setApprovalForAll(address, bool) public virtual pure override {
        revert FeatureNotSupported();
    }
```

**파급력:** 배포 비용 증가.

**권장 완화 방법:** `SecuritizeRWASegWrap`에서 `setApprovalForAll()` 구현을 제거하십시오.

**Securitize**
커밋 [b78f30](https://github.com/securitize-io/bc-rwa-seg-wrap-sc/commit/b78f305a0dcc6a1e7eb425d05bae150f23d2d184)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 수정자만 추가하는 함수 재정의의 코드 중복

**설명:** `SecuritizeRWASegWrap` 계약은 부모 `RWASegWrap` 계약의 4개 함수(`deposit`, `redeem`, `redeemById`, `depositById`)를 동일한 구현 로직으로 재정의하고 `receiverIsSender` 수정자만 추가합니다. 전체 함수 본문을 복제하는 대신, 이 함수들은 추가 수정자를 적용한 후 `super`를 사용하여 부모 구현을 호출해야 합니다.

`SecuritizeRWASegWrap::deposit`, `SecuritizeRWASegWrap::redeem`, `SecuritizeRWASegWrap::redeemById`, `SecuritizeRWASegWrap::depositById`의 현재 구현은 변수 선언, 볼트 해결, 검증 확인 및 내부 함수 호출을 포함하여 `RWASegWrap`의 부모 함수와 정확히 동일한 비즈니스 로직을 반복합니다.

```solidity
// SecuritizeRWASegWrap의 현재 구현
function deposit(uint256 assets, address receiver)
    public override whenNotPaused amountNotZero(assets) addressNotZero(receiver) receiverIsSender(receiver)
    returns (uint256) {
    address caller = _msgSender();
    uint256 vaultId = getVaultId(caller);
    if (vaultId == 0) {
        vaultId = ++latestVaultId;
        vault = _deployVault(vaultId);
        _addVault(address(vault), vaultId, caller);
    }
    return _doDeposit(caller, assets, receiver, vaultId);
}

// RWASegWrap의 부모 구현 (receiverIsSender 수정자가 없는 것을 제외하고 동일한 로직)
function deposit(uint256 assets, address receiver)
    external virtual override whenNotPaused amountNotZero(assets) addressNotZero(receiver)
    returns (uint256) {
    address caller = _msgSender();
    uint256 vaultId = _resolveVaultId(caller);
    return _doDeposit(caller, assets, receiver, vaultId);
}
```

**파급력:** 이 코드 중복은 유지 관리 부담을 증가시키는 동시에 불일치 위험을 초래합니다.

**권장 완화 방법:** 로직을 복제하는 대신 `super` 호출을 사용하도록 재정의된 함수를 리팩터링하십시오:

```diff
function deposit(uint256 assets, address receiver)
    public override whenNotPaused amountNotZero(assets) addressNotZero(receiver) receiverIsSender(receiver)
    returns (uint256) {
-   address caller = _msgSender();
-   uint256 vaultId = getVaultId(caller);
-   if (vaultId == 0) {
-       vaultId = ++latestVaultId;
-       vault = _deployVault(vaultId);
-       _addVault(address(vault), vaultId, caller);
-   }
-   return _doDeposit(caller, assets, receiver, vaultId);
+   return super.deposit(assets, receiver);
}

function redeem(uint256 shares, address receiver, address owner)
    external override whenNotPaused amountNotZero(shares) addressNotZero(receiver) receiverIsSender(receiver) addressNotZero(owner)
    returns (uint256) {
-   address caller = _msgSender();
-   uint256 vaultId = getVaultId(caller);
-   if (vaultId == 0) {
-       revert VaultNotFound();
-   }
-   return _doRedeem(caller, shares, receiver, owner, vaultId);
+   return super.redeem(shares, receiver, owner);
}

function redeemById(uint256 shares, address receiver, address owner, uint256 id)
    external override whenNotPaused amountNotZero(shares) addressNotZero(receiver) addressNotZero(owner) receiverIsSender(receiver) idNotZero(id) recognizedVault(id)
    returns (uint256) {
-   address caller = _msgSender();
-   uint256 vaultId = getVaultId(caller);
-   if (id != vaultId) {
-       revert InvestorVaultMismatch(id, caller);
-   }
-   return _doRedeem(caller, shares, receiver, owner, id);
+   return super.redeemById(shares, receiver, owner, id);
}

function depositById(uint256 assets, address receiver, uint256 id)
    external override whenNotPaused amountNotZero(assets) receiverIsSender(receiver) idNotZero(id) recognizedVault(id)
    returns (uint256) {
-   address caller = _msgSender();
-   uint256 vaultId = getVaultId(caller);
-   if (id != vaultId) {
-       revert InvestorVaultMismatch(id, caller);
-   }
-   return _doDeposit(caller, assets, receiver, vaultId);
+   return super.depositById(assets, receiver, id);
}
```

**Securitize:** 커밋 [6322e3](https://github.com/securitize-io/bc-rwa-seg-wrap-sc/commit/6322e36c86c012123db32e3dcd6cfe9ebd99eed4)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 가스 최적화 (Gas Optimization)

### `caller` 매개변수를 사용할 수 있을 때 `_resolveVaultId`에서 불필요한 `_msgSender()` 호출

**설명:** `RWASegWrap::_resolveVaultId()`에서 함수는 볼트 ID를 해결해야 하는 주소를 나타내는 `caller` 매개변수를 받지만, `_addVault()`를 호출할 때 제공된 `caller` 매개변수 대신 `_msgSender()`를 사용합니다. 이는 불필요한 함수 호출을 생성하고 `caller` 매개변수에 이미 올바른 주소가 포함되어 있기 때문에 잠재적인 불일치를 야기합니다.

```solidity
function _resolveVaultId(address caller) internal returns (uint256) {
    uint256 vaultId = getVaultId(caller);
    if (vaultId == 0) {
        vaultId = ++latestVaultId;
        ISegregatedVault vault = _deployVault(vaultId);
        _addVault(address(vault), vaultId, _msgSender()); // <- 발신자 직접 사용
    }
    return vaultId;
}
```

**파급력:** 불필요한 `_msgSender()` 호출은 추가 가스 소비를 초래하고 동일한 주소를 나타내야 하는 다른 변수를 사용하여 코드 명확성을 떨어뜨립니다.

**권장 완화 방법:** `_addVault()` 호출에서 `_msgSender()`를 `caller` 매개변수로 교체하십시오:

```diff
function _resolveVaultId(address caller) internal returns (uint256) {
    uint256 vaultId = getVaultId(caller);
    if (vaultId == 0) {
        vaultId = ++latestVaultId;
        ISegregatedVault vault = _deployVault(vaultId);
-       _addVault(address(vault), vaultId, _msgSender());
+       _addVault(address(vault), vaultId, caller);
    }
    return vaultId;
}
```

**Securitize:** 커밋 [3e16e8](https://github.com/securitize-io/bc-securitize-vault-sc/commit/3e16e88ea071e3365e7fd0b70789b22c0f717ccd)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 중복된 최대 입금 확인으로 인한 입금 함수의 불필요한 가스 소비

**설명:** `SecuritizeVaultV2::_depositWithFee` 함수는 입금 금액이 최대 허용 입금 한도를 초과하지 않는지 확인하기 위해 `maxDeposit(from)`에 대해 불필요한 확인을 수행합니다. 그러나 `SecuritizeVaultV2`는 부모 `ERC4626Upgradeable` 계약의 `maxDeposit` 함수를 재정의하지 않으므로 `maxDeposit`은 항상 `type(uint256).max`를 반환합니다.

```solidity
// SecuritizeVaultV2::_depositWithFee 내부
function _depositWithFee(uint256 assets, address from) private returns (uint256) {
    address caller = _msgSender();
    uint256 maxAssets = maxDeposit(from); // type(uint256).max 반환
    if (assets > maxAssets) { // 이 조건은 절대 참이 될 수 없음
        revert ERC4626ExceededMaxDeposit(from, assets, maxAssets);
    }
    // ... 함수의 나머지 부분
}

// ERC4626Upgradeable 내부 (SecuritizeVaultV2에 의해 재정의되지 않음)
function maxDeposit(address) public view virtual returns (uint256) {
    return type(uint256).max; // 항상 최대 uint256 값 반환
}
```
이는 `assets > type(uint256).max`가 현실적인 입금 금액에 대해 절대 참이 될 수 없기 때문에 중복 비교를 생성하여 확인을 무의미하고 가스 낭비로 만듭니다. 346-348행의 조건은 어떤 uint256 값도 `type(uint256).max`를 초과할 수 없기 때문에 `ERC4626ExceededMaxDeposit` 되돌림(revert)을 절대 트리거하지 않습니다.

함수 호출 `maxDeposit(from)`과 후속 비교는 모든 입금 작업에서 실행되어 현재 구현에서 아무런 목적이 없는 확인을 위해 불필요하게 가스를 소비합니다.

**파급력:** 이 불필요한 계산은 기능적 이점을 제공하지 않으면서 모든 입금 작업의 가스 비용을 증가시켜 사용자의 거래 비용을 높입니다.

**권장 완화 방법:** `SecuritizeVaultV2`가 사용자 정의 입금 한도를 구현하지 않으므로 불필요한 최대 입금 확인을 제거하십시오:

```diff
function _depositWithFee(uint256 assets, address from) private returns (uint256) {
    address caller = _msgSender();
-   uint256 maxAssets = maxDeposit(from);
-   if (assets > maxAssets) {
-       revert ERC4626ExceededMaxDeposit(from, assets, maxAssets);
-   }

    uint256 fee = 0;
    if (address(feeManager) != address(0)) {
        fee = IFeeManager(feeManager).computeFee(IFeeManager.FeeApplicableOperation.Deposit, assets);
    }

    uint256 shares = previewDeposit(assets - fee);
    _depositAndSendFees(caller, from, assets, fee, shares);
    return shares;
}
```

대안으로, 입금 한도를 향후 구현하려는 경우 적절한 로직으로 `maxDeposit` 함수를 재정의하십시오.

**Securitize:** 커밋 [5105f0](https://github.com/securitize-io/bc-securitize-vault-sc/commit/5105f03e95502fc887241e47f660996e9163d3e8) 및 [57805a](https://github.com/securitize-io/bc-rwa-seg-wrap-sc/commit/57805a83958e87e3f9b3677caf8554c09bd8fad0)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 청산 기능의 가스 비효율적 승인 패턴으로 불필요한 가스 소비 발생

**설명:** `SecuritizeVaultV2::_liquidateTo` 함수는 상환 계약이 구성된 경우 모든 청산 호출에서 상환 계약에 대한 승인 작업을 수행합니다. 489행에서 함수는 각 청산 작업에 대해 `IERC20Metadata(asset()).approve(address(redemption), assets)`를 호출하여 상환할 정확한 자산 금액만 승인합니다.

이 접근 방식은 다음과 같은 이유로 가스 비효율적입니다:

1. **반복적인 SSTORE 작업**: 각 `approve` 호출은 허용량 매핑을 업데이트하기 위해 비용이 많이 드는 SSTORE 작업을 필요로 합니다.
2. **고정된 상환 계약**: 상환 계약은 초기화 중에 설정되며 이후에는 변경할 수 없으므로 영구 승인을 부여하는 것이 안전합니다.

현재 구현은 초기화 중에 상환 계약에 `type(uint256).max` 승인을 부여하는 것이 더 효율적인 접근 방식일 때 불필요하게 모든 청산에서 가스를 소비합니다.

**파급력:** 모든 청산 작업은 중복 승인 호출로 인해 추가 가스를 소비하여 청산을 수행하는 사용자의 거래 비용을 증가시킵니다.

**권장 완화 방법:** 각 청산에서 승인하는 대신 초기화 중에 상환 계약에 최대 승인을 부여하십시오:

```diff
function _initialize(
    string memory _name,
    string memory _symbol,
    address _securitizeToken,
    address _redemptionAddress,
    address _liquidationToken,
    address _navProvider
) private {
    // ... 기존 초기화 로직 ...

    redemption = ISecuritizeOffRamp(_redemptionAddress);
    liquidationToken = IERC20Metadata(_liquidationToken);
    navProvider = ISecuritizeNavProvider(_navProvider);

+   // 상환 계약이 존재하는 경우 최대 승인 부여
+   if (address(redemption) != address(0)) {
+       bool success = IERC20Metadata(_securitizeToken).approve(address(redemption), type(uint256).max);
+       if (!success) {
+           revert AssetApprovalFailed();
+       }
+   }

    _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
}

function _liquidateTo(address to, uint256 shares) internal {
    if (balanceOf(to) < shares) {
        revert NotEnoughShares();
    }
    uint256 assets = convertToAssets(shares);
    _burn(to, shares);

    emit Liquidate(to, assets, shares);

    if (address(0) != address(redemption)) {
-       bool success = IERC20Metadata(asset()).approve(address(redemption), assets);
-       if (!success) {
-           revert AssetApprovalFailed();
-       }
        uint256 balanceBefore = liquidationToken.balanceOf(address(this));
        redemption.redeem(assets, 0);
        uint256 receivedAmount = liquidationToken.balanceOf(address(this)) - balanceBefore;
        liquidationToken.safeTransfer(to, receivedAmount);
    } else {
        liquidationToken.safeTransfer(to, assets);
    }
}
```
**Securitize:** 인지함.

**Cyfrin:** 인지함.

\clearpage
