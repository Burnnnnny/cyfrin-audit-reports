**Lead Auditors**

[Immeas](https://twitter.com/0ximmeas)

[MrPotatoMagic](https://x.com/MrPotatoMagic)

---

# 발견 사항 (Findings)
## 낮은 위험 (Low Risk)


### HyperCore로의 네이티브 HYPE 전송이 작동하지 않음

**설명:** `Hype_Module::hyper_depositSpot`은 `asset == 150`(네이티브 HYPE)인 경우를 포함하여 항상 `IERC20(token).transfer(assetAddress(uint64(asset)), _wei)`를 호출합니다. HyperCore [문서](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/hypercore-less-than-greater-than-hyperevm-transfers#transferring-hype)에 따르면, HYPE(id `150`)는 ERC-20을 통해서가 아니라 특수 시스템 주소 `0x2222…2222`로 네이티브 값(value)으로 전송되어야 합니다. 또한 네이티브 HYPE를 전송할 때 `token` 매개변수는 모호성과 우발적인 ERC-20 경로를 피하기 위해 자리 표시자(영 주소)여야 합니다.

**영향:** 토큰 매개변수가 자리 표시자이거나 최악의 경우 래핑된 버전인 경우 네이티브 HYPE 전송이 작동하지 않을 가능성이 높습니다. 전송은 성공할 수 있지만 HyperCore에서 토큰 크레딧이 실패하여(자산 ID가 잘못되었으므로) 토큰이 손실될 수 있습니다. 또한 네이티브 예치에 대한 `token` 매개변수와 관련된 모호성은 구성 오류 또는 자금의 조용한 오라우팅(misrouting)으로 이어질 수 있습니다.

**권장 완화 조치:**

* `hyper_depositSpot`을 `payable`로 만들고 `asset`에 따라 분기합니다.

  * `asset == 150`(HYPE/네이티브)인 경우: `token == address(0)` 및 `msg.value == _wei`를 요구한 다음 네이티브 값을 `0x2222…2222`로 보냅니다.

    ```diff
      function hyper_depositSpot(
          address token,
          uint32 asset,
          uint64 _wei
    - ) external onlyRole(EXECUTOR_ROLE) nonReentrant {
    + ) external payable onlyRole(EXECUTOR_ROLE) nonReentrant {
    +     if (asset == 150) {
    +         require(token == address(0), "token must be zero for HYPE");
    + 	      require(msg.value == _wei, "msg.value mismatch");
    +         (bool ok, ) = address(0x2222222222222222222222222222222222222222).call{value: _wei}("");
    + 	      require(ok, "native HYPE transfer failed");
    +         return;
    +     }
    +
          IERC20(token).transfer(assetAddress(uint64(asset)), _wei);
      }
    ```
* 선택적으로 API를 오버로드하거나 분할하여(`hyper_depositSpotHype(uint64 _wei)` 대 `hyper_depositSpotERC20(address token, uint32 asset, uint256 amount)`) 모호성을 제거합니다.


**D2:** 커밋 [`134a2b1`](https://github.com/d2sd2s/d2-contracts/commit/134a2b1c4d40de852b60a3124f8e8ded9a025668)에서 수정되었으나, 발신자/운영자가 소유한 자금이 아니라 전략 컨트랙트 내부의 자금을 전송하기 위한 것이므로 payable로 만들고 msg.value를 확인하는 부분은 제외되었습니다.

**Cyfrin:** 확인함. `asset == 150`인 경우 Hype 모듈은 이제 `value: _wei`로 `0x22...22`를 호출합니다.


### 안전하지 않은 ERC20 전송

**설명:** `Hype_Module::hyper_depositSpot`은 `IERC20(token).transfer(...)`를 직접 사용하고 반환 값을 무시합니다. 많은 ERC-20이 비표준(예: USDT)이며 `bool`을 반환하지 않거나 명확하지 않은 방식으로 실패 시 되돌려서(revert), `transfer`/`transferFrom`을 그대로 사용하는 것은 안전하지 않습니다.

**영향:** 토큰 전송이 조용히 실패하거나 토큰마다 일관되지 않게 작동하여, 예치금이 Core에 적립되지 않고 자금이 호출자에게 묶일 수 있습니다.

**권장 완화 조치:**
모든 토큰 상호 작용에 OpenZeppelin의 `SafeERC20`을 사용하십시오.

```solidity
using SafeERC20 for IERC20;

IERC20(token).safeTransfer(assetAddress(uint64(asset)), amount);
```

**D2:** 인지함. 우리는 메인넷에 있지 않으며 모든 토큰은 적절한 호환 ERC20 구현을 사용할 가능성이 높으므로 SafeERC20을 추가하는 것을 건너뜁니다. 화이트리스트에 추가하고 사용하기 전에 볼트에서 지원하는 토큰을 검토할 것입니다.

\clearpage
## 정보 (Informational)


### `nonReentrant`가 첫 번째 수정자가 아님

**설명**
`Hype_Module` 전체에서 `nonReentrant`는 외부 함수에서 `onlyRole(EXECUTOR_ROLE)` 뒤에 나열됩니다. Solidity에서 수정자는 왼쪽에서 오른쪽으로 적용되며 첫 번째 수정자가 가장 바깥쪽 래퍼가 됩니다. 일관된 심층 방어(defense-in-depth)를 위해 `nonReentrant`가 첫 번째여야 후속 수정자(현재 또는 미래) 내의 모든 로직도 보호하여 수정자가 재진입 가드가 설정되기 전에 상태 변경이나 외부 호출을 수행할 위험을 최소화할 수 있습니다.

함수가 `nonReentrant`를 첫 번째 수정자로 갖도록 변경하는 것을 고려하십시오.

**D2:** 커밋 [`c5aeb40`](https://github.com/d2sd2s/d2-contracts/commit/c5aeb405bc5c9b4cd2a173eacf6b8ebbd8890ea8)에서 수정됨.

**Cyfrin:** 확인함.


### 작업 ID에 매직 넘버 대신 상수 사용 고려

**설명:** HyperCore와 상호 작용하는 데 사용되는 작업 ID는 현재 Hype.sol 모듈의 함수에 하드코딩되어 있습니다.

예를 들어 작업 ID = 6은 `spotSend` 작업을 나타냅니다.
```solidity
function hyper_sendSpot(
        uint64 asset,
        uint64 _wei
    ) external onlyRole(EXECUTOR_ROLE) nonReentrant {
        sendAction(6, abi.encode(assetAddress(asset), asset, _wei));
    }
```

**영향:** 위험은 없지만 매직 넘버 대신 상수를 사용하면 매직 넘버의 의도된 목적을 설명하여 코드 유지 관리성과 가독성을 향상시킵니다.

**권장 완화 조치:** 현재 하드코딩된 각 작업 ID에 대한 상수를 구현하는 것을 고려하십시오. 예를 들어 작업 ID 1은 상수 `LIMIT_ORDER_ACTION_ID`로 명명할 수 있습니다.

**D2:** 커밋 [`41470c6`](https://github.com/d2sd2s/d2-contracts/commit/41470c60bd928fb6e67d0db285ef32f0b6490197)에서 수정됨.

**Cyfrin:** 확인함.


### 인터페이스와 구현 간의 매개변수 이름 불일치로 오해 소지 있음

**설명:** `IHype_Module`에서 매개변수 이름이 구현과 다릅니다.

* `hyper_sendSpot(uint64 token, uint64 _wei)` 대 구현에서는 첫 번째 `uint64`에 `asset`을 사용합니다.
* `hyper_addApiWallet(address wallet, string calldata apiKey)` 대 구현에서는 `string calldata`에 `name`을 사용합니다.

`apiKey` 이름은 누군가가 비밀 API 키를 온체인에 제출하게 하여 영구적으로 공개될 수 있으므로 특히 위험합니다.

인터페이스와 구현 매개변수 이름을 정렬하는 것을 고려하십시오(예: `asset` 및 `name/label` 사용).

**D2:** 커밋 [`5c5cec4`](https://github.com/d2sd2s/d2-contracts/commit/5c5cec46325b7ac061d49d8035c4901ed5db4ed4)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### 작업 전송 시 루프 제거

**설명:** `Hype_Module::sendAction`은 루프에서 4바이트 헤더와 페이로드를 수동으로 할당하고 복사합니다. 루프를 피하고 바이트코드를 줄이려면 패킹된 인코딩을 사용하십시오. 예:

```solidity
bytes memory data = abi.encodePacked(bytes4(uint32(0x01000000) | uint32(actionIndex)), action);
```

이렇게 하면 더 적은 가스와 더 적은 코드로 접두사 + 페이로드를 한 번에 빌드합니다.

**D2:** 커밋 [`c5d3193`](https://github.com/d2sd2s/d2-contracts/commit/c5d319387671e889e1d1c6aaf5097b5653af6809)에서 수정됨.

**Cyfrin:** 확인함.


### `Hype_module::assetAddress`를 `pure`로 변경 가능

**설명:** `Hype_module::assetAddress`는 상태를 읽거나 쓰지 않습니다. `internal pure`로 표시하여 컴파일러 최적화를 활성화하고, 정적 분석/staticcall 스타일 사용을 허용하며, 우발적인 상태 액세스를 방지하면서 가스/바이트코드 크기를 약간 줄이십시오.

**D2:** 커밋 [`a217930`](https://github.com/d2sd2s/d2-contracts/commit/a2179308e7ecef2247cd51af52ddb90f4507d896)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage

