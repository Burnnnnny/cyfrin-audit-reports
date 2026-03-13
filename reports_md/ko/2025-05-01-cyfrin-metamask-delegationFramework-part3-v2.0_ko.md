**수석 감사자**

[0kage](https://twitter.com/0kage_eth)

**보조 감사자**



---

# 결과 (Findings)
## 정보성 (Informational)


### DelegationMetaSwapAdapter에서 0 주소 확인 누락

**설명:** `DelegationMetaSwapAdapter`의 `constructor` 및 `setSwapApiSigner` 함수에서 0 주소 확인이 누락되었습니다.

```solidity

  constructor(
        address _owner,
        address _swapApiSigner,
        IDelegationManager _delegationManager,
        IMetaSwap _metaSwap,
        address _argsEqualityCheckEnforcer
    )
        Ownable(_owner)
    {
        swapApiSigner = _swapApiSigner; //@audit missing address(0) check
        delegationManager = _delegationManager; //@audit missing address(0) check
        metaSwap = _metaSwap; //@audit missing address(0) check
        argsEqualityCheckEnforcer = _argsEqualityCheckEnforcer; //@audit missing address(0) check
        emit SwapApiSignerUpdated(_swapApiSigner);
        emit SetDelegationManager(_delegationManager);
        emit SetMetaSwap(_metaSwap);
        emit SetArgsEqualityCheckEnforcer(_argsEqualityCheckEnforcer);
    }
  function setSwapApiSigner(address _newSigner) external onlyOwner {
        swapApiSigner = _newSigner; //@audit missing address(0) check
        emit SwapApiSignerUpdated(_newSigner);
    }
```



**권장되는 완화 방법:** 0 주소 확인을 추가하는 것을 고려하십시오.

**Metamask:** 커밋 [6912e73](https://github.com/MetaMask/delegation-framework/commit/6912e732e2ed65699152c6bfdb46a0ed433f1263)에서 해결되었습니다.

**Cyfrin:** 해결되었습니다.


### `DelegationMetaSwapAdapter`의 모호한 만료 타임스탬프 유효성 검사

**설명:** DelegationMetaSwapAdapter.sol 컨트랙트에서 `_validateSignature()` 메서드는 서명 만료를 검증할 때 "크거나 같음"(>=) 비교 대신 "큼"(>) 비교를 사용합니다:

```solidity
function _validateSignature(SignatureData memory _signatureData) private view {
    if (block.timestamp > _signatureData.expiration) revert SignatureExpired();

    bytes32 messageHash_ = keccak256(abi.encodePacked(_signatureData.apiData, _signatureData.expiration));
    bytes32 ethSignedMessageHash_ = MessageHashUtils.toEthSignedMessageHash(messageHash_);

    address recoveredSigner_ = ECDSA.recover(ethSignedMessageHash_, _signatureData.signature);
    if (recoveredSigner_ != swapApiSigner) revert InvalidApiSignature();
}
```

이 구현은 만료 타임스탬프의 정확한 순간에 서명이 유효한 상태로 유지되도록 허용하여 의도된 보안 모델에 모호함을 만듭니다.

**파급력:** 만료된 것으로 표시된 서명(만료 타임스탬프가 현재 블록 타임스탬프와 동일함)이 여전히 유효한 것으로 간주되어 직관적이지 않을 수 있으며 혼란을 초래할 수 있습니다.

**권장되는 완화 방법:** 현재 동작이 의도적인 경우 `expiration` 필드의 이름을 `validUpto`로 변경하는 것을 고려하십시오. 대안으로 `expiration`이라는 용어와 의미를 명확히 하려면 `>`를 `>=`로 대체하는 것을 고려하십시오.

**Metamask:** 커밋 [6912e73](https://github.com/MetaMask/delegation-framework/commit/6912e732e2ed65699152c6bfdb46a0ed433f1263)에서 해결되었습니다.

**Cyfrin:** 해결되었습니다.

\clearpage
