**Lead Auditors**

[DadeKuma](https://x.com/DadeKuma)

[Naman](https://twitter.com/0kage_eth)

**Assisting Auditors**



---

# 발견 사항 (Findings)
## 정보 (Informational)


### 불필요한 `emit_cpi!` 사용으로 인한 CU 비용 증가

**설명:** `whitelist` 명령은 [`emit_cpi!`](https://github.com/securitize-io/bc-solana-whitelist-sc/blob/main/programs/dstoken-whitelist/src/instructions/whitelist.rs#L247)와 `#[event_cpi]` 속성을 사용하여 `Whitelisted` 이벤트를 발생시킵니다.

`emit_cpi!` 매크로는 CPI를 통해 호출되고 이벤트가 호출 프로그램에 전파되어야 하는 프로그램을 위해 설계되었습니다.

그러나 `dstoken-whitelist` 프로그램은 최종 사용자가 직접 호출하도록 의도되었으며, 다른 프로그램에 의해 CPI를 통해 호출되지 않습니다.

**영향:** 불필요할 때 `emit_cpi!`를 사용하는 것은 이점 없이 복잡성을 가중시킵니다:
- 트랜잭션 크기 및 CU 비용 증가
- 현재 이러한 이벤트를 직접 구독하는 것은 [불가능함](https://www.anchor-lang.com/docs/features/events#:~:text=Currently%2C%20event%20data%20emitted%20through%20CPIs%20cannot%20be%20directly%20subscribed%20to.%20To%20access%20this%20data%2C%20you%20must%20fetch%20the%20complete%20transaction%20data%20and%20manually%20decode%20the%20event%20information%20from%20the%20instruction%20data%20of%20the%20CPI.)
- 명령어에 추가적인 이벤트 권한 계정을 전달해야 함

**권장 완화 조치:** `emit_cpi!`를 `emit!`으로 대체하고 `#[event_cpi]` 속성을 제거하는 것을 고려하십시오.

**Securitize:** 수정 사항: https://github.com/securitize-io/bc-solana-whitelist-sc/commit/2a83f4e3f518c9d0801b24d900be56240bd7da31

**Cyfrin:** 확인함.


### 비어 있지 않은 해시로 인해 트랜잭션 크기 제한이 초과될 수 있음

**설명:** `whitelist` 명령은 CPI 데이터 버퍼에 인코딩된 가변 길이 문자열(`investor_id`, `collision_hash`, `proof_hash`)을 허용합니다.

Solana는 1,232바이트의 트랜잭션 크기 제한을 강제하며, 이는 일부 엣지 케이스에서 화이트리스팅 기능을 제한할 수 있습니다.

**영향:** 프로덕션 구성은 `collision_hash` 및 `proof_hash`에 빈 문자열을 사용하여 트랜잭션 크기를 1,232바이트 제한 아래로 유지합니다.

그러나 향후 비어 있지 않은 해시가 사용될 경우, 단일 트랜잭션에 5개 이상의 레벨을 추가할 때 트랜잭션 크기가 제한을 초과하여 트랜잭션이 실패할 수 있습니다.

**개념 증명 (Proof of Concept):** 빈 해시를 사용하는 구성을 사용하면 `whitelist` 명령은 1,232바이트 트랜잭션 제한 아래 안전 마진을 유지합니다:

| 레벨 | 트랜잭션 크기 | 마진 | 상태 |
|--------|------------------|--------|--------|
| 2 | 994 bytes | 238 bytes | Safe |
| 3 | 1,016 bytes | 216 bytes | Safe |
| 5 | 1,060 bytes | 172 bytes | Safe |

비교를 위해, 비어 있지 않은 32자 해시가 사용된 경우 트랜잭션은 5개 이상의 레벨에서 되돌려집니다(revert):

| 레벨 | 트랜잭션 크기 | 마진 | 상태 |
|--------|------------------|--------|--------|
| 2 | 1,098 bytes | 134 bytes | Safe |
| 3 | 1,143 bytes | 89 bytes | Safe |
| 5 | 1,233 bytes | -1 byte | Exceeds limit |

개발 팀은 프로덕션에서 `collision_hash` 및 모든 `proof_hash` 값에 빈 문자열을 사용할 것임을 확인했습니다. `rbac_utils::is_valid_hash` 검증은 빈 문자열을 유효한 것으로 허용합니다:

```rust
pub fn is_valid_hash(hash: &str) -> bool {
     hash.is_ascii() && hash.len() <= 32
}
```

**권장 완화 조치:** 현재 구성에는 완화 조치가 필요하지 않지만, 이 동작을 문서화하는 것을 고려하십시오.

향후 요구 사항이 변경되어 5개 이상의 레벨에서 비어 있지 않은 해시를 사용해야 하는 경우, 되돌리기를 방지하기 위해 함수를 별도의 트랜잭션으로 분할해야 합니다.

**Securitize:** 인지함.

\clearpage

