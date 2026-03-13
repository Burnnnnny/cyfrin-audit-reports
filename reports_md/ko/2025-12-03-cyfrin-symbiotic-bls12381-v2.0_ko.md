**Lead Auditors**

[Farouk](https://x.com/Ubermensh3dot0)

[qpzm](https://x.com/qpzmly)

**Assisting Auditors**

[Alex Roan](https://twitter.com/alexroan)

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

---

# 발견 사항 (Findings)
## 정보 (Informational)


### 입력이 이차 잉여(quadratic residue)가 아닐 때 `BLS12381::findYFromX`가 되돌리는(revert) 대신 `sqrt(-a)`를 반환함

**설명:** `BLS12381::findYFromX`는 `y = (x^3+4)^((p+1)/4) mod p` 공식을 사용하여 제곱근을 계산합니다. 이 공식은 `x^3+4`가 법 p에 대해 이차 잉여일 때만 올바르게 작동합니다. `x^3+4`가 이차 잉여가 **아닐** 때, 함수는 되돌리는 대신 `sqrt(-(x^3+4))`를 반환합니다.

```solidity
function findYFromX(uint256 x_a, uint256 x_b) internal view returns (uint256 y_a, uint256 y_b) {
    // compute (x**3 + 4) mod p
    (y_a, y_b) = _xCubePlus4(x_a, x_b);

    // compute y = sqrt(x**3 + 4) mod p = (x**3 + 4)^(p+1)/4 mod p
    // ...
}
```

오일러의 기준(Euler's criterion)에 따르면:
- `a`가 이차 잉여인 경우: `a^((p-1)/2) = 1 (mod p)`
- `a`가 이차 잉여가 아닌 경우: `a^((p-1)/2) = -1 (mod p)`

따라서 `a`가 이차 잉여가 아닌 경우:

```
(a^((p+1)/4))^2 = a^((p+1)/2) = a^((p-1)/2) * a = (-1) * a = -a (mod p)
```

결과 `a^((p+1)/4)`는 `sqrt(a)`가 아니라 `sqrt(-a)`를 제공합니다.

**영향:** `findYFromX`는 `KeyBlsBls12381::deserialize`에서 사용됩니다. `x^3+4`가 이차 잉여가 아닌 잘못된 x 좌표가 제공되면, `deserialize`는 되돌리는 대신 곡선 위에 있지 않은 잘못된 점을 반환합니다.

현재 코드베이스에서는 다음과 같은 이유로 영향이 제한적입니다:
1. `KeyRegistry::_setKey`에서 키는 `fromBytes` -> `wrap`을 통해 검증되며, 저장하기 전에 `isOnCurve` 및 `isInSubgroup`을 확인합니다.
2. `deserialize`는 검증된 키가 포함된 신뢰할 수 있는 스토리지에서만 읽습니다.
3. BLS 프리컴파일: BLS12_G1ADD, BLS12_G1MSM, BLS12_PAIRING_CHECK는 잘못된 점을 거부합니다. https://github.com/ethereum/go-ethereum/blob/6f2cbb7a27ba7e62b0bdb2090755ef0d271714be/core/vm/contracts.go#L1211

그러나 적절한 검증 없이 다른 맥락에서 `findYFromX` 또는 `deserialize`가 사용되면 잘못된 곡선 점을 조용히 반환할 수 있습니다.

**개념 증명 (Proof of Concept):** p = 19 (19 = 3 (mod 4))인 예시:

| a | Is QR? | a^5 mod 19 | (a^5)^2 mod 19 | Expected |
|---|--------|------------|---------------|----------|
| 4 | Yes | 17 | 4 | a |
| 3 | No | 15 | 16 | -a = -3 = 16 |
| 2 | No | 13 | 17 | -a = -2 = 17 |
| 5 | Yes | 9 | 5 | a |

이차 잉여가 아닌 경우 `a^((p+1)/4)`는 `sqrt(a)` 대신 `sqrt(-a)`를 제공합니다.

**권장 완화 조치:** 곡선 멤버십 및 하위 그룹 검증을 강제하지 않는 향후 업그레이드를 위해, x^3+4가 이차 잉여가 아닌 엣지 케이스를 적절히 처리하도록 하십시오.

**Symbiotic:** 인지함. 팀은 `BLS12381` 동작을 `BN254`와 유사하게 유지하는 것을 목표로 합니다.



### `isOnCurve`와 `isInSubgroup` 간의 무한대 점(point at infinity) 처리 불일치

**설명:** `BLS12381::isOnCurve` 함수는 무한대 점 `(0, 0, 0, 0)`에 대해 `false`를 반환하는 반면, `isInSubgroup`은 동일한 점에 대해 `true`를 반환합니다. 이는 항등원(identity element)이 검증되는 방식에 불일치를 만듭니다.

`isOnCurve`에서 함수는 $y^2 \equiv x^3 + 4 \pmod{p}$인지 확인합니다:
- `(0, 0, 0, 0)`의 경우: $y^2 = 0$이지만 $x^3 + 4 = 4$
- $0 \neq 4$이므로 함수는 `false`를 반환합니다.

그러나 `isInSubgroup`에서는:
```solidity
function isInSubgroup(G1Point memory point) internal view returns (bool) {
    G1Point memory result = scalar_mul(point, G1_SUBGROUP_ORDER);
    return result.x_a == 0 && result.x_b == 0 && result.y_a == 0 && result.y_b == 0;
}
```

무한대 점은 `0 * order = 0` (어떤 스칼라를 곱해도 항등원은 항등원)이므로 `true`를 올바르게 반환합니다.

수학적으로 무한대 점은 유효한 그룹 요소(항등원)이지만 바이어슈트라스(Weierstrass) 곡선 방정식을 만족하는 아핀(affine) 좌표를 갖지 않습니다.

라이브러리의 다른 함수들은 무한대 점을 올바르게 처리한다는 점에 유의하십시오:
- `add`, `scalar_mul`, `pairing`: EIP-2537 프리컴파일에 의해 처리됨
- `negate`: 명시적 확인이 `(0, 0, 0, 0)`을 변경 없이 반환함
- `isInSubgroup`: `true`를 올바르게 반환함

이로 인해 `isOnCurve`가 항등원 처리에 있어 유일한 예외가 됩니다.

**영향:**
- `isOnCurve`가 암호화 작업 전에 G1 점을 검증하는 데 사용되면 항등원이 잘못 거부됩니다.

**개념 증명 (Proof of Concept):**
```solidity
G1Point memory infinity = G1Point(0, 0, 0, 0);

bool onCurve = BLS12381.isOnCurve(infinity);      // returns false
bool inSubgroup = BLS12381.isInSubgroup(infinity); // returns true

// Inconsistent: valid subgroup element fails curve check
```

**권장 완화 조치:** `isOnCurve`에 무한대 점에 대한 특수 사례 확인을 추가하십시오:

```solidity
function isOnCurve(G1Point memory point) internal view returns (bool) {
    // Point at infinity is a valid curve point (identity element)
    if (point.x_a == 0 && point.x_b == 0 && point.y_a == 0 && point.y_b == 0) {
        return true;
    }
    // ... rest of the function
}
```

**Symbiotic:** [bedb2ea](https://github.com/symbioticfi/relay-contracts/commit/bedb2ea253950f978c07cf80f71ae72ddf313e5a)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### `BLS12381::hashToG1`은 DST에 상수를 사용할 수 있음

**설명:** `BLS12381::hashToG1`에서 도메인 분리 태그(DST) 문자열이 `expandMsg`에 인라인 문자열 리터럴로 전달되어 호출 시마다 불필요한 메모리 할당이 발생합니다.

```solidity
function hashToG1(bytes memory message) internal view returns (G1Point memory result) {
    bytes memory uniform_bytes = expandMsg("BLS_SIG_BLS12381G1_XMD:SHA-256_SSWU_RO_NUL_", message, 0x80);
```
https://github.com/symbioticfi/relay-contracts/blob/main/src/libraries/utils/BLS12381.sol#L287

런타임 시 다음과 같은 일이 발생합니다:
1. 문자열을 위해 메모리에 43바이트 할당
2. 바이트코드에서 메모리로 문자열 리터럴 복사
3. 메모리 확장 비용 발생
4. `expandMsg`에 메모리 포인터 전달

**영향:** 모든 `hashToG1` 호출마다 가스가 낭비됩니다. BLS 서명 검증은 일반적인 작업이므로 이는 누적됩니다.

| 테스트 | 전 | 후 | 절감 |
|------|--------|-------|---------|
| `test_VerifyValidSignature` | 401,631 | 400,494 | 1,137 gas |
| `test_HashToG1_OnCurveAndNonZero` | 20,672 | 20,298 | 374 gas |

**권장 완화 조치:** DST를 상수로 정의하고 특화된 `expandMsgBLS` 함수를 생성하십시오:

```solidity
bytes32 private constant DST_PART1 = "BLS_SIG_BLS12381G1_XMD:SHA-256_S";
bytes11 private constant DST_PART2 = "SWU_RO_NUL_";
uint8 private constant DST_LEN = 43;

function hashToG1(bytes memory message) internal view returns (G1Point memory result) {
    bytes memory uniform_bytes = expandMsgBLS(message, 0x80);
    // ...
}

function expandMsgBLS(bytes memory message, uint8 n_bytes) internal pure returns (bytes memory) {
    bytes memory zpad = new bytes(0x40);
    bytes memory b_0 = abi.encodePacked(zpad, message, uint8(0x00), n_bytes, uint8(0x00), DST_PART1, DST_PART2, DST_LEN);
    // ... rest of expandMsg logic using DST_PART1, DST_PART2, DST_LEN
}
```

이는 DST를 바이트코드에 직접 포함시켜 런타임 메모리 할당을 방지합니다.

**Symbiotic:** 인지함. 팀은 `expandMsg`를 변경하지 않는 것을 선호함.


\clearpage

