**Lead Auditors**

[Hans](https://twitter.com/hansfriese)
**Assisting Auditors**

---

# 발견 사항

## 중간 위험

### 서명자는 기한 전에 서명을 취소할 수 없음.

**심각도:** 중간

**설명:** 서명 후, 서명자는 어떤 이유로든 서명을 취소하고 싶을 수 있습니다. 다른 프로토콜을 확인해보면, 서명자는 넌스(nonce)를 증가시켜 취소할 수 있습니다.
이 프로토콜에서는 OpenZeppelin의 [Nonces](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Nonces.sol) 컨트랙트를 상속하며 기한 전에 서명을 취소할 방법이 없습니다.

**파급력:** 서명자는 원할 때 서명을 무효화할 수 없습니다.

**권장 완화 조치:** 과거 서명을 무효화하기 위해 `increaseNonce()`와 같은 함수를 추가하는 것을 권장합니다.

**클라이언트:**
호출자가 자신의 넌스를 증가시킬 수 있도록 외부 `useNonce()` 함수를 노출하는 기본 `Nonces` 컨트랙트를 추가하여 수정했습니다. 커밋: [`0189a1f`](https://github.com/farcasterxyz/farcaster-contracts-private/commit/0189a1fd308a5976ecdfbce2765b6d7a953eb80f)

**Cyfrin:** 확인됨.

### `IdRegistry`에서 복구(recovery) 주소가 예상치 못하게 업데이트될 수 있음.

**심각도:** 중간

**설명:** 복구 주소를 업데이트하는 함수는 `changeRecoveryAddress()`와 `changeRecoveryAddressFor()` 두 가지가 있습니다.
`changeRecoveryAddress()`는 `changeRecoveryAddressFor()`에서 사용될 대기 중인 서명을 재설정하지 않으므로 아래 시나리오가 가능합니다.

- 앨리스(Alice)는 밥(Bob)을 복구 주소로 설정하기로 결정하고 그에 대한 서명을 생성했습니다.
- 그러나 `changeRecoveryAddressFor()`를 호출하기 전에, 앨리스는 밥이 적합하지 않다는 것을 깨닫고 `changeRecoveryAddress()`를 직접 호출하여 복구 주소를 다른 사람으로 변경했습니다.
- 하지만 그 후 밥이나 다른 누군가가 `changeRecoveryAddressFor()`를 호출하면 밥은 소유자 또한 변경할 수 있습니다.

물론 앨리스는 넌스를 증가시켜 서명을 삭제할 수 있지만, 사용자가 이전 서명을 사용할 수 있도록 허용하는 것은 좋은 접근 방식이 아닙니다.

**파급력:** 복구 주소가 예상치 못하게 업데이트될 수 있습니다.

**권장 완화 조치:** 복구 서명에 현재 복구 주소를 포함해야 합니다.
그러면 복구 주소를 변경한 후 이전 서명은 자동으로 무효화됩니다.

**클라이언트:**
`CHANGE_RECOVERY_ADDRESS_TYPEHASH`에 현재 복구 주소를 추가하여 수정했습니다. 커밋: [`7826446`](https://github.com/farcasterxyz/farcaster-contracts-private/commit/7826446c172d2038ab7b3eeb3073c3a7233061df)

**Cyfrin:** 확인됨.

### `IdRegistry.transfer/transferFor()`가 복구 주소에 의해 취소될 수 있음.

**심각도:** 중간

**설명:** 모든 `fid`에는 소유자와 복구 주소가 존재하며, 각각 동일한 권한을 가지고 있어 어느 쪽이든 다른 쪽을 수정할 수 있습니다.
하지만 `fid`를 전송할 때는 소유자만 변경하며 이 시나리오가 가능합니다.

- `fid(owner, recovery)`를 가진 밥이 그것을 판매하려고 합니다.
- 자금을 받은 후, 그는 `transfer()`를 사용하여 `fid`를 정직한 사용자에게 전송합니다.
- 정직한 사용자가 복구 주소를 업데이트하려고 할 때, 밥이 프론트 러닝으로 `recover()`를 호출하여 계정을 탈취합니다.
- ERC721과 달리, 복구 주소는 NFT의 승인된 사용자(approved user)처럼 행동하여 언제든지 소유권을 변경할 수 있는 권한을 가집니다. 특히, 이 권한은 이전 승인에 의한 후속 업데이트를 방지하기 위해 [전송](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/ERC721.sol#L252) 중에 삭제됩니다.

**파급력:** `IdRegistry.transfer/transferFor()`가 복구 주소에 의해 취소될 수 있습니다.

**권장 완화 조치:** `owner/recovery`를 모두 업데이트하는 `transferAll()`과 같은 함수를 추가하는 것을 권장합니다.

**클라이언트:**
`IdRegistry`에 `transferAndChangeRecovery` 및 `transferAndChangeRecoveryFor`를 추가하여 수정했습니다. 커밋: [`d389f9f`](https://github.com/farcasterxyz/farcaster-contracts-private/commit/d389f9f9e102ea1706f115b0aba2c7e429ba3e9a)

**Cyfrin:** 확인됨.

### 제거 서명(removal signature)이 잘못된 `fid`에 적용될 수 있음.

**심각도:** 중간

**설명:** 제거 서명은 `KeyRegistry.removeFor()`를 사용하여 `fidOwner`의 키를 제거하는 데 사용됩니다. 그리고 서명은 `_verifyRemoveSig()`에서 검증됩니다.

```solidity
    function _verifyRemoveSig(address fidOwner, bytes memory key, uint256 deadline, bytes memory sig) internal {
        _verifySig(
            _hashTypedDataV4(
                keccak256(abi.encode(REMOVE_TYPEHASH, fidOwner, keccak256(key), _useNonce(fidOwner), deadline))
            ),
            fidOwner,
            deadline,
            sig
        );
    }
```

하지만 서명은 제거할 `fid`를 지정하지 않으므로 아래 시나리오가 가능합니다.

- 앨리스는 `fid1`의 소유자이며 `key`를 제거하기 위한 제거 서명을 생성했지만 아직 사용되지 않았습니다.
- 여러 가지 이유로 그녀는 `fid2`의 소유자가 되었습니다.
- `fid2`에도 `key`가 있지만 그녀는 그것을 제거하고 싶지 않습니다.
- 그러나 누군가 그녀의 이전 서명으로 `removeFor()`를 호출하면 `fid2`에서 `key`가 예상치 못하게 제거됩니다.

키가 제거되면 `KeyState`가 `REMOVED`로 변경되며 소유자를 포함한 누구도 복구할 수 없습니다.

**파급력:** 키 제거 서명이 예상치 못한 `fid`에 사용될 수 있습니다.

**권장 완화 조치:** 제거 서명은 다른 `fid`에 대해 무효화되도록 `fid`도 포함해야 합니다.

**클라이언트:**
인지함. 이는 호출자의 할당된 fid를 미리 알지 못해도 단일 트랜잭션으로 fid를 등록하고 키를 추가할 수 있도록 하기 위한 의도적인 설계 트레이드오프입니다. 우리는 이것이 발견 사항에 설명된 결과를 초래한다는 것을 받아들이며, 사용자는 키 레지스트리 작업을 "현재 소유한 fid에 키 추가"로 해석해야 합니다.

넌스(Nonces)는 이 시나리오에 대해 어느 정도 보호를 제공합니다. 앨리스가 `fid1`을 위한 이전 서명을 취소하려면 넌스를 증가시켜 서명을 무효화할 수 있습니다.

**Cyfrin:** 인지함.

## 저위험

### `vaultAddr`의 일관되지 않은 검증

`KeyManager.setVault()` 및 `StorageRegistry.setVault()`에는 address(0)에 대한 검증이 있지만 생성자에서는 확인하지 않습니다.

```solidity
File: audit-farcaster\src\KeyManager.sol
123:         vault = _initialVault;
124:         emit SetVault(address(0), _initialVault);
...
211:     function setVault(address vaultAddr) external onlyOwner {
212:         if (vaultAddr == address(0)) revert InvalidAddress();
213:         emit SetVault(vault, vaultAddr);
214:         vault = vaultAddr;
215:     }
216:
```

**클라이언트:**
내부 논의 후, 우리는 `KeyGateway`에서 지불을 완전히 제거하고 당분간 `KeyRegistry`의 fid당 제한에 의존하기로 결정했습니다. 우리는 게이트웨이 패턴을 그대로 유지하여 향후 필요할 경우 지불을 도입할 수 있는 기능을 제공합니다.

이번 배포로 StorageRegistry를 재배포할 의도는 없지만, 다음 버전의 스토리지 계약에서 이 검증을 추가할 것입니다.

커밋: [`11e2722`](https://github.com/farcasterxyz/farcaster-contracts-private/commit/11e27223625e4c6b5f929398e015ccda740c1593)

**Cyfrin:** 인지함.

### 일부 관리자 기능에 대한 검증 부족

`KeyManager.setUsdFee()` 및 `StorageRegistry.setPrice()`에는 상한선이 없습니다.

프로토콜 소유자는 신뢰할 수 있는 당사자로 간주되지만, `StorageRegistry.setFixedEthUsdPrice()`의 `fixedEthUsdPrice`에는 최소/최대 제한이 있기 때문에 여전히 일관성이 없는 구현입니다.

```solidity
File: audit-farcaster\src\KeyManager.sol
203:     function setUsdFee(uint256 _usdFee) external onlyOwner {
204:         emit SetUsdFee(usdFee, _usdFee);
205:         usdFee = _usdFee;
206:     }

File: audit-farcaster\src\StorageRegistry.sol
716:     function setPrice(uint256 usdPrice) external onlyOwner {
717:         emit SetPrice(usdUnitPrice, usdPrice);
718:         usdUnitPrice = usdPrice;
719:     }
```

**클라이언트:**
내부 논의 후, 우리는 `KeyGateway`에서 지불을 완전히 제거하기로 결정했습니다. (자세한 내용은 7.2.1에 대한 답변 참조).

이번 배포로 StorageRegistry를 재배포할 의도는 없지만, 다음 버전의 스토리지 계약에서 이 검증을 추가할 것입니다.

커밋: [`11e2722`](https://github.com/farcasterxyz/farcaster-contracts-private/commit/11e27223625e4c6b5f929398e015ccda740c1593)

**Cyfrin:** 인지함.

