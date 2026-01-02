**수석 감사자**

[0kage](https://x.com/0kage_eth)

[Giovanni Di Siena](https://x.com/giovannidisiena)



**보조 감사자**

[Hans](https://x.com/hansfriese)

---

# 발견 사항 (Findings)
## 정보성 (Informational)


### 잘못 문서화된 에러 선택자(selector)

**설명:** `IWormholeTransceiver::TransferAlreadyCompletedError`의 `bytes4` 에러 선택자가 `0x406e719e`로 잘못 문서화되어 있습니다. 올바른 선택자는 `0xb4c3b00c`입니다.

```solidity
    /// @notice Error when the VAA has already been consumed.
    /// @dev Selector: 0x406e719e.
    /// @param vaaHash The hash of the VAA.
    error TransferAlreadyCompleted(bytes32 vaaHash);
```

**권장 완화 조치:** 선택자를 `0xb4c3b00c`로 업데이트하는 것을 고려하십시오.


### 이벤트 및 에러에 대한 일관성 없는 인라인 문서화

**설명:** 현재 코드 베이스는 이벤트 및 에러에 대해 매개변수 설명, 이벤트의 `topic[0]`, 에러의 `bytes4` 선택자를 포함하는 인라인 문서 표준을 따르고 있습니다. 그러나 일부 이벤트 및 에러에는 매개변수 설명, 선택자 또는 둘 다가 누락되어 있습니다. 이러한 불일치는 코드 가독성과 유지 관리성을 떨어뜨릴 수 있습니다.

다음은 `INttManager.sol`의 몇 가지 예입니다.

```solidity
    /// @notice The caller is not the deployer.
    error UnexpectedDeployer(address expectedOwner, address owner);

    /// @notice Peer for the chain does not match the configuration.
    /// @param chainId ChainId of the source chain.
    /// @param peerAddress Address of the peer nttManager contract.
    error InvalidPeer(uint16 chainId, bytes32 peerAddress);

   /// @notice Peer chain ID cannot be zero.
    error InvalidPeerChainIdZero();

    /// @notice Peer cannot be the zero address.
    error InvalidPeerZeroAddress();

    /// @notice Peer cannot have zero decimals.
    error InvalidPeerDecimals();

```

**권장 완화 조치:** 해당하는 경우 매개변수 설명, `topic[0]` 및 `bytes4` 선택자를 포함하여 모든 이벤트 및 에러 정의에 대해 일관된 문서화를 보장하십시오.



### 인바운드(inbound) 및 아웃바운드(outbound) 한도 설정에 대한 이벤트 부재

**설명:** `NttManager::setPeer` 함수는 외부 체인에 피어 `NttManager` 계약 주소를 설정합니다. 이제 `inboundLimit`은 피어 계약을 설정할 때 입력으로 전달됩니다. 이전 구현에서는 inboundLimit이 `type(uint64).max`로 설정되었습니다. 그러나 이 입력은 `PeerUpdated` 이벤트에서 누락되어 `setPeer` 입력 매개변수의 변경 사항을 반영하지 않습니다.

**권장 완화 조치:** `setPeer` 함수에 의해 설정된 매개변수를 정확하게 반영하기 위해 `PeerUpdated` 이벤트에 `inboundLimit`을 포함하는 것을 고려하십시오. 또한 타사 통합의 맥락에서 인바운드 및 아웃바운드 한도가 서로 다른 대상 체인에 대해 여러 번 업데이트될 수 있으므로, `NttManager` 소유자가 인바운드 또는 아웃바운드 한도를 설정할 때마다 이벤트 방출을 추가하는 것이 좋습니다. 이를 통해 이러한 매개변수 변경의 투명성과 추적성을 향상시킬 수 있습니다.


### `TransferSent` 이벤트에서 인덱싱 누락

**설명:** `INttManager::TransferSent` 이벤트는 메시지가 소스 체인의 `NttManager`에서 전송될 때 방출됩니다. 현재 이벤트 서명은 `recipient` 및 `refundAddress` 매개변수를 인덱싱하지 않습니다. 대규모로 전송이 수행될 때, 이러한 인덱싱 부족은 체인 간 전송의 검색 가능성을 저해할 수 있습니다.

**권장 완화 조치:** 검색 가능성을 높이기 위해 `TransferSent` 이벤트에서 `recipient` 및 `refundAddress` 매개변수를 인덱싱하는 것을 고려하십시오.


\clearpage

