**Lead Auditors**

[Farouk](https://x.com/Ubermensh3dot0)

[0kage](https://twitter.com/0kage_eth)

**Assisting Auditors**

[Alex Roan](https://twitter.com/alexroan)

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

---

# 발견 사항 (Findings)
## 정보 (Informational)


### G2 공개 키 및 G1 서명에 대한 명시적 0 값 확인 누락

**설명:** BLS12381 검증 로직은 0인 G1 공개 키를 명시적으로 거부하지만, 다음은 명시적으로 거부하지 않습니다:
1) 0인 G2 공개 키
2) 0인 G1 서명

```solidity
/**
 * @notice Verify a BLS signature.
 * @param keyG1 The G1 public key.
 * @param messageHash The message hash to verify.
 * @param signatureG1 The G1 signature.
 * @param keyG2 The G2 public key.
 * @return If the signature is valid.
 * @dev Burns the whole gas if pairing precompile fails.
 *      Returns false if the key is zero G1 point.
 */
function verify(
    BLS12381.G1Point memory keyG1,
    bytes32 messageHash,
    BLS12381.G1Point memory signatureG1,
    BLS12381.G2Point memory keyG2
) internal view returns (bool) {
    if (keyG1.x_a == 0 && keyG1.x_b == 0 && keyG1.y_a == 0 && keyG1.y_b == 0) {
        return false;
    }
    BLS12381.G1Point memory messageG1 = BLS12381.hashToG1(abi.encodePacked(messageHash));
    uint256 alpha = uint256(keccak256(abi.encode(signatureG1, keyG1, keyG2, messageG1))) % BLS12381.FR_MODULUS;

    return BLS12381.pairing(
        signatureG1.add(keyG1.scalarMul(alpha)),
        BLS12381.negGeneratorG2(),
        messageG1.add(BLS12381.generatorG1().scalarMul(alpha)),
        keyG2
    );
}
```

BLS12-381 프리컴파일의 동작(잘못된 형식의 곡선 점이 페어링 확인을 통과하는 것을 방지함)으로 인해 이러한 케이스가 실제로 악용될 수는 없지만, 0 값을 가진 키와 서명은 BLS 사양에서 유효한 입력이 아니므로 명확성과 일관성을 위해 거부되어야 합니다.

0인 G2 키를 허용하면 두 번째 페어링 항이 항등원(identity)으로 축소되는 퇴화된 검증 경로가 도입됩니다. 0인 서명 또한 프로토콜 설계 관점에서 유효하지 않으므로 통합자 및 미래의 코드 유지 관리자를 위한 모호성을 피하기 위해 명시적으로 제외되어야 합니다.

**권장 완화 조치:** 곡선이나 페어링 연산, 특히 G2 공개 키와 G1 서명을 호출하기 전에 명시적인 0 확인을 추가하십시오.

**Symbiotic:** 인지함.

\clearpage

