**수석 감사자**

[Dacian](https://twitter.com/devdacian)

[0kage](https://twitter.com/0kage_eth)

**보조 감사자**



---

# 발견 사항 (Findings)
## 치명적 위험 (Critical Risk)


### Polygon 체인 재구성(reorg)으로 미스터리 박스 등급이 변경될 수 있으며 검증자가 이를 악용할 수 있음

**설명:** [`REQUEST_CONFIRMATIONS = 3`](https://github.com/Earnft/smart-contracts/blob/43d3a8305dd6c7325339ed35d188fe82070ee5c9/contracts/MysteryBox.sol#L26)은 Polygon에 대해 너무 작은 값입니다. [체인 재구성이 3 블록 깊이보다 자주 발생하기 때문입니다](https://polygonscan.com/blocks_forked?p=1).

**영향:** 체인 재구성은 블록과 트랜잭션 순서를 변경하여 난수(randomness) 결과를 바꿉니다. 희귀한 박스를 당첨받았던 사람이 재구성 중 난수 결과 변경으로 인해 일반 박스로 바뀌거나 그 반대의 경우가 발생할 수 있습니다.

또한 이는 의도적으로 체인의 기록을 다시 써서 난수 요청을 다른 블록으로 강제 이동시켜 난수 결과를 변경할 수 있는 [검증자에 의해 악용될 수 있습니다](https://docs.chain.link/vrf/v2/security/#choose-a-safe-block-confirmation-time-which-will-vary-between-blockchains). 이를 통해 검증자는 새로운 난수 값을 얻을 수 있으며, 만약 그들이 미스터리 박스를 민팅하고 있다면 더 희귀한 박스를 얻기 위해 더 유리한 난수 결과가 나오도록 트랜잭션을 이동시킬 수 있습니다.

**권장 완화 조치:** `REQUEST_CONFIRMATIONS = 30`은 Polygon에서 매우 안전해 보입니다. 체인 재구성이 이보다 깊은 블록 깊이를 가지는 경우는 매우 드물기 때문입니다. 가끔 발생하는 것은 큰 문제가 아니지만, 항상 발생한다면("3" 설정은 이를 보장함) 좋지 않으며 검증자가 악용할 가능성이 있습니다.

**Mode:**
커밋 [85b2012](https://github.com/Earnft/smart-contracts/commit/85b20121604b5d162bb14c2c96731b8345ca1cb3)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 미스터리 박스 전송 시 토큰 상환 불가 문제

**설명:** `MysteryBox`는 `ERC1155` 계약으로, 사용자는 내장된 전송 기능을 통해 다른 주소로 전송할 수 있을 것으로 기대합니다. 그러나 `MysteryBox::claimMysteryBoxes()`는 미스터리 박스 소유권을 추적하는 내부 매핑이 전송 시 업데이트되지 않기 때문에 호출자가 박스를 민팅한 주소와 동일하지 않으면 [revert](https://github.com/Earnft/smart-contracts/blob/43d3a8305dd6c7325339ed35d188fe82070ee5c9/contracts/MysteryBox.sol#L296)됩니다.

**영향:** 사용자가 미스터리 박스를 전송하면 토큰 상환이 불가능해집니다(bricked). 사용자는 자신이 제어하는 다른 주소로 미스터리 박스를 전송하거나(예: 첫 번째 주소가 해킹당한 경우), `ERC1155` 판매를 지원하는 OpenSea 같은 플랫폼에서 미스터리 박스를 판매하기를 합리적으로 기대합니다.

**권장 완화 조치:** `ERC1155` 전송 훅(hook)을 재정의하여 미스터리 박스 전송을 막거나, 미스터리 박스가 전송될 때 내부 매핑을 업데이트하여 새 소유자 주소가 토큰을 상환할 수 있도록 하십시오. 두 번째 옵션은 미스터리 박스 보유자가 토큰에 매도 압력을 가하지 않고도 유동성에 접근할 수 있게 하여 미스터리 박스를 위한 "2차 시장"을 조성하므로 프로토콜에 더 매력적일 수 있습니다.

**Mode:**
커밋 [a65a50c](https://github.com/Earnft/smart-contracts/commit/a65a50ca8af4d6abc58d3c429785bcd82182c04e)에서 `ERC1155::_beforeTokenTransfer()`를 재정의하여 미스터리 박스 전송을 방지함으로써 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 고위험 (High Risk)


### `MysteryBox::fulfillRandomWords()`의 잘못된 검사로 인해 동일한 요청이 여러 번 이행되는 것을 막지 못함

**설명:** 동일한 요청이 여러 번 이행되는 것을 방지하려는 [검사](https://github.com/Earnft/smart-contracts/blob/43d3a8305dd6c7325339ed35d188fe82070ee5c9/contracts/MysteryBox.sol#L221-L222)를 고려해 보십시오:
```solidity
if (vrfRequests[_requestId].fulfilled) revert InvalidVrfState();
```

문제는 `vrfRequests[_requestId].fulfilled`가 어디에서도 `true`로 설정되지 않으며, `vrfRequests[_requestId]`는 함수 끝에서 [삭제](https://github.com/Earnft/smart-contracts/blob/43d3a8305dd6c7325339ed35d188fe82070ee5c9/contracts/MysteryBox.sol#L244-L245)된다는 것입니다.

**영향:** 동일한 요청이 여러 번 이행될 수 있어 이전에 생성된 무작위 시드(seed)를 덮어쓸 수 있습니다. 또한 미스터리 박스 민터(minter)인 악의적인 제공자는 희귀한 미스터리 박스를 얻을 때까지 새로운 난수를 생성할 수 있습니다.

**권장 완화 조치:** `vrfRequests[_requestId].fulfilled = true`로 설정하십시오.

`activeVrfRequests`와 `fulfilledVrfRequests` 2개의 매핑을 포함하는 최적화된 버전을 고려해 보십시오:
* `if(fulfilledVrfRequests[_requestId])`이면 revert
* 그렇지 않으면 `fulfilledVrfRequests[_requestId] = true` 설정
* `activeVrfRequests[_requestId]`에서 일치하는 활성 요청을 메모리로 가져와서 정상적으로 처리 계속
* 마지막에 `delete activeVrfRequests[_requestId]`

이렇게 하면 `fulfilledVrfRequests`에 `requestId` : `bool` 쌍만 영구적으로 저장됩니다.

`MysteryBox::fulfillBoxAmount()`에서도 유사한 접근 방식을 고려하십시오.

**Mode:**
커밋 [85b2012](https://github.com/Earnft/smart-contracts/commit/85b20121604b5d162bb14c2c96731b8345ca1cb3), [c4c50ed](https://github.com/Earnft/smart-contracts/commit/c4c50edcd2a3f9fc2da4e1934bcfa1d3cbd85809), [d5b14d8](https://github.com/Earnft/smart-contracts/commit/d5b14d80dae0cc78ab63537d405c8c49a6238a57), [5df2b82](https://github.com/Earnft/smart-contracts/commit/5df2b824dba5b25a0d8db28fa10de4a4bc52ec3b)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 소유자가 상환 토큰을 러그풀(rug-pull)하여 미스터리 박스 계약을 지불 불능 상태로 만들고 보유자가 상환하지 못하게 할 수 있음

**설명:** [`MysteryBox::ownerWithdrawEarnm()`](https://github.com/Earnft/smart-contracts/blob/43d3a8305dd6c7325339ed35d188fe82070ee5c9/contracts/MysteryBox.sol#L765-L777)은 소유자가 계약의 전체 상환 토큰 잔액을 자신에게 전송할 수 있게 하여, 미스터리 박스 상환에 사용되어야 할 토큰을 러그풀할 수 있습니다.

**영향:** 계약이 완전히 지불 불능 상태가 되어 미스터리 박스 소유자가 상환할 수 없게 됩니다.

**권장 완화 조치:** 계약은 항상 현재 민팅되어 청구되지 않은 모든 미스터리 박스에 대한 최대 상환 부채를 지불하는 데 필요한 토큰을 보유해야 합니다. 소유자는 잉여 금액(총 부채를 초과하는 금액)만 인출할 수 있어야 합니다.

미스터리 박스가 민팅될 때 총 부채가 증가하고 미스터리 박스가 청구될 때 총 부채가 감소하도록 추적하고, 소유자는 이 값을 초과하는 잉여 토큰만 인출할 수 있도록 하는 것을 고려하십시오.

**Mode:**
커밋 [db7b48e](https://github.com/Earnft/smart-contracts/commit/db7b48e69c33e327d613f88035c8335531572e8d), [edefb61](https://github.com/Earnft/smart-contracts/commit/edefb61534ecee1a2f6cb7e687c113a1f7b82056), [a65a50c](https://github.com/Earnft/smart-contracts/commit/a65a50ca8af4d6abc58d3c429785bcd82182c04e)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `batchesAmount`에 대한 잘못된 상한선으로 인해 50억 개 대신 5억 개의 토큰만 미스터리 박스 보유자에게 분배됨

**설명:** `setBatchesAmount()`는 최대 `batchesAmount`를 100으로 [제한](https://github.com/Earnft/smart-contracts/blob/43d3a8305dd6c7325339ed35d188fe82070ee5c9/contracts/MysteryBox.sol#L782)하지만 이는 잘못되었습니다. 모든 배치는 약 500만 토큰으로 상환할 수 있는 미스터리 박스를 릴리스하며 총 50억 토큰이 있으므로 전체 공급량을 분배하려면 1000 배치가 필요합니다.

**영향:** 100 배치로 잘못 제한하면 50억 토큰 전체를 분배할 수 없고 5억 토큰만 분배하게 됩니다.

**권장 완화 조치:** 전체 토큰 분배를 허용하기 위해 `batchesAmount`를 1000으로 제한하십시오.

**Mode:**
커밋 [ae3dc68](https://github.com/Earnft/smart-contracts/commit/ae3dc68db8c723293df01cb14297dc3264a21dbe)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 중급 위험 (Medium Risk)


### `MysteryBox::revealMysteryBoxes()`에서 초과 ETH가 사용자에게 환불되지 않음

**설명:** `MysteryBox::revealMysteryBoxes()`는 `msg.value >= mintFee`인 경우 [실행을 허용](https://github.com/Earnft/smart-contracts/blob/43d3a8305dd6c7325339ed35d188fe82070ee5c9/contracts/MysteryBox.sol#L196-L198)하지만, `msg.value > mintFee`인 경우 초과 ETH가 사용자에게 환불되지 않고 `operatorAddress`로 전송됩니다.

**영향:** 사용자는 `mintFee`를 초과하는 ETH를 잃게 됩니다.

**권장 완화 조치:** 초과 ETH를 사용자에게 환불하거나 `msg.value != mintFee`인 경우 revert 하십시오.

**Mode:**
커밋 [85b2012](https://github.com/Earnft/smart-contracts/commit/85b20121604b5d162bb14c2c96731b8345ca1cb3)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### Chainlink Any API 사용 시 외부 어댑터의 요청 시간 초과로 인해 민팅이 무기한 중단될 수 있음

**설명:** Mode는 Chainlink Any API를 통합하여 외부 어댑터와 상호 작용하고, 사용자 코드와 지갑 주소를 확인하여 민팅할 박스 수를 결정합니다. 이 시스템은 `direct-request` 작업 유형을 사용하여 `ChainlinkRequested` 이벤트 방출에 따라 작업을 트리거합니다. 그러나 주목할 만한 문제가 있습니다. 초기 GET 요청이 시간 초과되면 이러한 요청은 무기한 대기 상태로 남을 수 있습니다. 현재 설계에는 대기 중인 요청을 취소하고 새 요청을 생성하는 조항이 없습니다.

**영향:** 외부 어댑터가 즉시 응답하지 않으면 초기 요청 후 코드가 삭제되므로 사용자는 다른 민팅 요청을 제출할 수 없습니다. 이로 인해 사용자가 코드를 잃고 미스터리 박스 보상을 받지 못할 수 있습니다.

**권장 완화 조치:** 요청 시간 초과 시 코드 수신자가 호출할 수 있는 기능을 구현하는 것을 고려하십시오. 이 기능은 내부적으로 `ChainlinkClient:cancelChainlinkRequest`를 호출하고 `MysteryBox` 계약에 콜백을 포함하여 원본과 동일한 데이터를 사용하여 새 요청을 시작해야 합니다. 즉, 새 요청에 대해 코드/사용자 주소와 이전에 생성된 난수를 재사용하는 것을 의미합니다.

**Mode:**
인지함.

\clearpage
## 저위험 (Low Risk)


### 반환 데이터가 필요하지 않을 때 가스 그리핑(gas griefing) 공격을 방지하기 위해 저수준 `call()` 사용

**설명:** 반환 데이터가 필요하지 않을 때 `call()`을 사용하면 반환된 거대한 데이터 페이로드로 인한 가스 그리핑 공격에 불필요하게 노출됩니다. [예](https://github.com/Earnft/smart-contracts/blob/43d3a8305dd6c7325339ed35d188fe82070ee5c9/contracts/MysteryBox.sol#L197-L198):
```solidity
(bool sent, ) = address(operatorAddress).call{value: msg.value}("");
if (!sent) revert Unauthorized();
```
는 다음과 동일합니다:
```solidity
(bool sent, bytes memory data) = address(operatorAddress).call{value: msg.value}("");
if (!sent) revert Unauthorized();
```
두 경우 모두 반환 데이터가 전혀 필요하지 않음에도 불구하고 반환 데이터가 메모리에 복사되어야 하므로 계약이 가스 그리핑 공격에 노출됩니다.

**영향:** 계약이 불필요하게 가스 그리핑 공격에 노출됩니다.

**권장 완화 조치:** 반환 데이터가 필요하지 않은 경우 저수준 호출을 사용하십시오. 예:

```solidity
bool sent;
assembly {
    sent := call(gas(), receiver, amount, 0, 0, 0, 0)
}
if (!sent) revert Unauthorized();
```
[ExcessivelySafeCall](https://github.com/nomad-xyz/ExcessivelySafeCall) 사용을 고려하십시오.

**Mode:**
커밋 [85b2012](https://github.com/Earnft/smart-contracts/commit/85b20121604b5d162bb14c2c96731b8345ca1cb3)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)


### `MysteryBox::claimMysteryBoxes()`에 대한 중복 `boxId` 입력 방지

**설명:** 특정 상황에서 악용될 수 있으므로 [`MysteryBox::claimMysteryBoxes()`](https://github.com/Earnft/smart-contracts/blob/43d3a8305dd6c7325339ed35d188fe82070ee5c9/contracts/MysteryBox.sol#L271)에 대한 중복 `boxId` 입력을 방지하는 것을 고려하십시오.

**영향:** 공격자가 중복 입력을 사용하여 토큰 청구를 악용할 수 있습니다.

**권장 완화 조치:** 중복 입력이 발생하면 revert 하십시오. `boxId`는 고유하므로 중복 입력은 악의적인 공격의 명백한 징후입니다.

**Mode:**
커밋 [85b2012](https://github.com/Earnft/smart-contracts/commit/85b20121604b5d162bb14c2c96731b8345ca1cb3), [3713107](https://github.com/Earnft/smart-contracts/commit/3713107bb24382bda0fb6ac2eb51e9c64c39c98d)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `MysteryBox::claimMysteryBoxes()`는 `amountToClaim == 0`으로 인해 revert 될 때 사용자 정의 오류를 반환해야 함

**설명:** `MysteryBox::claimMysteryBoxes()`는 `amountToClaim == 0`으로 인해 revert 될 때 [사용자 정의 오류를 반환](https://github.com/Earnft/smart-contracts/blob/43d3a8305dd6c7325339ed35d188fe82070ee5c9/contracts/MysteryBox.sol#L305)해야 합니다. 현재는 상환되는 미스터리 박스에 대해 계약의 토큰 잔액이 부족한 경우와 동일한 오류인 `InsufficientEarnmBalance`를 반환합니다.

**영향:** 오해의 소지가 있는 오류가 반환됩니다.

**권장 완화 조치:** 사용자 정의 오류를 반환하십시오.

**Mode:**
커밋 [85b2012](https://github.com/Earnft/smart-contracts/commit/85b20121604b5d162bb14c2c96731b8345ca1cb3)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 집중된 미스터리 박스 보상으로 인한 EarnNM 토큰의 가격 변동성 위험 잠재

**설명:** 현재 시스템의 미스터리 박스 보상 분배 메커니즘은 무작위성을 기반으로 하므로 짧은 기간 내에 대량의 토큰이 유통될 위험이 있습니다. 특히 1-2일과 같은 짧은 기간 동안 상당한 수의 고가치 박스(예: 신화 1개, 전설 2개, 에픽 10개)가 할당되는 비정상적인 상황이 발생할 수 있습니다. 또한 짧은 기간 동안 상당한 양의 박스가 민팅될 가능성도 있습니다. 결과적으로 이러한 모든 박스의 베스팅 기간이 끝나면 동시에 EarnM 토큰이 릴리스될 가능성이 있습니다.

EarnM 토큰은 수수료 기반 토큰(예: 프로토콜 수수료에 연결된 토큰 가치)이나 토큰 보유를 장려하는 스테이킹 메커니즘이 아닙니다. 사실상 현재 설계에는 수요 동인과 공급 억제 요인이 없습니다.

**영향:** 특히 시장 침체기 동안의 강한 매도 압력은 유동성 풀에서 가격 조작 위험으로 이어질 수 있습니다. 이러한 상당한 가격 하락은 사용자들 사이에 공황을 불러일으켜 50/90%의 손실(haircut)에도 불구하고 미스터리 박스를 상환하도록 유도할 수 있습니다. 이러한 조치는 매도를 증폭시켜 Terra Luna와 같은 토큰에서 볼 수 있었던 이전 시장 붕괴와 유사한 심각한 시나리오로 이어질 가능성이 있습니다.

**권장 완화 조치:** EarnM 토큰 유동성 풀의 규모와 범위에 대한 불확실성을 감안하여, 팀은 베스팅 후 잠재적 매도 압력을 상쇄할 수 있는 충분한 유동성을 확보할 것을 권장합니다. 선제적인 유동성 관리는 중요한 기간 동안 토큰 가치를 안정화하는 데 결정적일 수 있습니다.

**Mode:**
인지함.


### 보상 코드 생성기와 Chainlink 노드 운영자가 동일한 엔터티임에 따른 중앙화 위험

**설명:** MODE에서 보상 코드를 관리하기 위한 현재 시스템 아키텍처는 코드 생성과 Chainlink 노드 운영 모두 MODE 팀이 제어하는 중앙 집중식입니다. 이러한 코드를 추적하고 관리하는 엔드포인트는 공개되지 않습니다. 이 설정에서 Chainlink Any API를 사용하는 것은 단일 노드 운영자(MODE 팀 자체)가 관리하므로 가치가 제한적입니다. 이러한 중앙화는 탈중앙화 오라클 네트워크의 잠재적 이점을 약화시킵니다.

**영향:** 이 설정은 Chainlink 인프라와 관련된 탈중앙화 이점을 제공하지 않으면서 LINK 수수료를 포함한 불필요한 복잡성과 비용을 초래합니다.

**권장 완화 조치:** 이 문제를 해결하기 위해 두 가지 잠재적 대안을 고려할 수 있습니다:

1. **외부 노드 운영자 참여:** 보상 코드 확인 작업을 외부 노드 운영자에게 위임합니다. 이 접근 방식에는 `Chainlink:setChainlinkOracle`을 호출하는 함수를 생성하여 향후 오라클 업데이트를 허용하는 것이 포함됩니다. 향후 엔드포인트를 공개하면 MODE가 필요에 따라 새 운영자를 임명할 수 있습니다.

2. **사내 추적을 통한 단순화:** 노드 운영자가 코드 생성 엔터티와 동일하게 유지되는 경우 프로세스를 단순화하는 것을 고려하십시오. 코드와 주소를 각 박스 수량에 연결하는 온체인 매핑을 유지하십시오. `apiAddress`가 허용 가능한 박스 수량으로 `MysteryBox::associateOneTimeCodeToAddress`를 트리거할 때마다 이 매핑을 업데이트하십시오. 이 간소화된 접근 방식은 Chainlink 오라클 및 외부 어댑터의 필요성을 우회하여 LINK 수수료와 복잡성을 줄이면서 현재 수준의 중앙화를 유지합니다.

VRF(Verifiable Random Function) 작업과 관련된 가스 제한을 감안하여 단일 트랜잭션에서 미스터리 박스 민팅을 용이하게 하기 위해 선택한 설계임을 인정합니다. 이러한 제약 조건 하에서 MODE 팀의 접근 방식은 합리적이었습니다.

**Mode:**
인지함.

\clearpage
## 가스 최적화 (Gas Optimization)


### `baseMetadataURI`는 이미 `ERC1155`에 저장되어 있고 `name`은 사용되지 않으므로 저장소에서 제거

**설명:** `baseMetadataURI`는 이미 `ERC1155`에 저장되어 있고 `name`은 전혀 사용되지 않으므로 [저장소](https://github.com/Earnft/smart-contracts/blob/43d3a8305dd6c7325339ed35d188fe82070ee5c9/contracts/MysteryBox.sol#L53-L54)에서 제거하십시오.

**영향:** 불필요한 값을 저장소에 쓰는 데 따른 추가 저장 비용 및 추가 가스 비용.

**권장 완화 조치:** `baseMetadataURI` 및 `name`을 저장소에서 제거하십시오.

**Mode:**
커밋 [85b2012](https://github.com/Earnft/smart-contracts/commit/85b20121604b5d162bb14c2c96731b8345ca1cb3)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 지속적인 앞뒤 변환을 피하기 위해 `tierId`를 `uint8` 또는 `uint256`으로 표준화

**설명:** 지속적인 앞뒤 변환을 피하기 위해 `tierId`를 `uint8` 또는 `uint256`으로 표준화하십시오.

**영향:** `tierId`에 대해 서로 다른 유형을 가지면 변환해야 할 뿐만 아니라 왜 다른 곳과 다른지에 대한 복잡성과 혼란을 증가시킵니다.

**권장 완화 조치:** `tierId`를 `uint8` 또는 `uint256`으로 표준화하십시오.

**Mode:**
커밋 [85b2012](https://github.com/Earnft/smart-contracts/commit/85b20121604b5d162bb14c2c96731b8345ca1cb3)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `boxId`가 주소와 등급에 고유하므로 `boxId` 저장소 매핑 단순화

**설명:** `boxId`는 여러 주소나 등급이 동일한 `boxId`를 가질 수 없도록 고유하므로, 최소한 [2개의 저장소 매핑](https://github.com/Earnft/smart-contracts/blob/43d3a8305dd6c7325339ed35d188fe82070ee5c9/contracts/MysteryBox.sol#L64-L65)인 `addressToTierToBoxIdToBlockTs`와 `addressToBoxIdToTier`를 잠재적으로 단순화할 수 있습니다.

다른 중첩 매핑을 리팩터링하여 복잡성을 줄이고 단순화하는 것을 고려하십시오.

**영향:** 저장소 매핑은 이미 꽤 복잡하여 오류가 발생하기 쉽고, 이 2가지가 구현된 방식은 읽기/쓰기에 더 많은 가스를 필요로 합니다.

**권장 완화 조치:** `boxId`가 주소 및 등급에 고유하다는 사실을 활용하여 이러한 매핑을 단순화하십시오.

**Mode:**
커밋 [85b2012](https://github.com/Earnft/smart-contracts/commit/85b20121604b5d162bb14c2c96731b8345ca1cb3), [efa8199](https://github.com/Earnft/smart-contracts/commit/efa8199895c7f5c76b5ac3c81bceaa94c8838eb2), [9c5ac66](https://github.com/Earnft/smart-contracts/commit/9c5ac662180602a3b1addf57c791e562c0ab9cd7)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 상태 변수는 저장소에서 다시 읽는 대신 스택 변수에 캐시해야 함

**설명:** 상태 변수는 저장소에서 다시 읽는 대신 스택 변수에 캐시해야 합니다.

* `MysteryBox::fulfillRandomWords()`는 `vrfRequests[_requestId]`를 3번 읽습니다. 메모리에 한 번 읽은 다음 메모리에서 읽어 다중 저장소 읽기를 피하는 것을 고려하십시오.
* `MysteryBox::fulfillBoxAmount()`는 `eaRequestToAddress[_requestId]`를 캐시하고 `delete addressToRandomNumber[sender]`를 수행할 수 있습니다.
* `MysteryBox::_assignTierAndMint()`는 `uint256 newBoxId = ++boxIdCounter;`를 가진 다음 함수의 나머지 부분에서 `newBoxId`를 사용해야 합니다.

**영향:** 가스 최적화

**권장 완화 조치:** 상태 변수는 저장소에서 다시 읽는 대신 스택 변수에 캐시해야 합니다.

**Mode:**
커밋 [85b2012](https://github.com/Earnft/smart-contracts/commit/85b20121604b5d162bb14c2c96731b8345ca1cb3), [c4c50ed](https://github.com/Earnft/smart-contracts/commit/c4c50edcd2a3f9fc2da4e1934bcfa1d3cbd85809), [d5b14d8](https://github.com/Earnft/smart-contracts/commit/d5b14d80dae0cc78ab63537d405c8c49a6238a57), [5df2b82](https://github.com/Earnft/smart-contracts/commit/5df2b824dba5b25a0d8db28fa10de4a4bc52ec3b)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 여러 변수를 피하고 코드를 단순화하기 위해 `MysteryBox::_determineTier()`에서 역방향 루프 사용

**설명:** `MysteryBox::_determineTier()`에서 여러 변수를 피하고 코드를 단순화하기 위해 [역방향 루프](https://github.com/Earnft/smart-contracts/blob/43d3a8305dd6c7325339ed35d188fe82070ee5c9/contracts/MysteryBox.sol#L380-L383)를 사용하십시오.

**영향:** 가스 최적화 및 더 간단한 코드.

**권장 완화 조치:** 설명을 참조하십시오.

**Mode:**
커밋 [4d56069](https://github.com/Earnft/smart-contracts/commit/4d560697f7dd6fa4f6b6303cca3e21c4025bee5b)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `MysteryBox::_calculateAmountToClaim()`의 계산 단순화

**설명:** `return (tokens * (10**EARNM_DECIMALS)) / divisor;` 라인을 매번 실행하고 [다른 분기](https://github.com/Earnft/smart-contracts/blob/43d3a8305dd6c7325339ed35d188fe82070ee5c9/contracts/MysteryBox.sol#L416-L417)와 불필요한 `%` 계산을 삭제하십시오.

**영향:** 가스 최적화 및 더 깨끗하고 간단한 코드.

**권장 완화 조치:** 설명을 참조하십시오.

**Mode:**
커밋 [85b2012](https://github.com/Earnft/smart-contracts/commit/85b20121604b5d162bb14c2c96731b8345ca1cb3)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `MysteryBox::_calculateVestingPeriodPerBox()`에서 사용되지 않는 `category` 제거

**설명:** `MysteryBox::_calculateVestingPeriodPerBox()`에서 [사용되지 않는](https://github.com/Earnft/smart-contracts/blob/43d3a8305dd6c7325339ed35d188fe82070ee5c9/contracts/MysteryBox.sol#L296) `category`를 제거하십시오.

**영향:** 가스 최적화 및 더 간단하고 깨끗한 코드.

**권장 완화 조치:** 설명을 참조하십시오.

**Mode:**
커밋 [85b2012](https://github.com/Earnft/smart-contracts/commit/85b20121604b5d162bb14c2c96731b8345ca1cb3)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `MysteryBox::_assignTierAndMint()`에서 루프 전에 `boxAmount < 100`을 한 번만 확인

**설명:** `boxAmount` 입력은 정적이므로 `MysteryBox::_assignTierAndMint()`의 루프 전에 [`boxAmount < 100` 확인](https://github.com/Earnft/smart-contracts/blob/43d3a8305dd6c7325339ed35d188fe82070ee5c9/contracts/MysteryBox.sol#L479)을 한 번만 수행하십시오.

**영향:** 가스 최적화.

**권장 완화 조치:** 설명을 참조하십시오.

**Mode:**
커밋 [06a6a4f](https://github.com/Earnft/smart-contracts/commit/06a6a4f6f12e5a52f797af26c4a27a4994fe6ce1)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
