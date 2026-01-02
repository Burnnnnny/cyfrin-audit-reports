**Lead Auditors**

[RajKumar](https://x.com/0xRajkumar)

[JesJupyter](https://x.com/jesjupyter)

[Ctrus](https://x.com/ctrusonchain)

**Assisting Auditors**

[Alexzoid](https://x.com/alexzoid_eth) (Formal Verification)



---

# Findings
## High Risk


### Users can cast vote without actually holding tokens

**Description:** `voting.rs`에서 사용자는 각 클라이언트 계정에 보유한 `drvs` 토큰으로 프로토콜 결정에 투표할 수 있습니다. 그러나 이 메커니즘은 활성 투표 기간 동안 입금 및 출금 타이밍을 악용하여 부풀려진 토큰 잔액으로 투표할 수 있는 사용자에 의해 남용될 수 있습니다. 악의적인 사용자는 기간 내에 소량의 토큰을 입금하고 해당 토큰을 출금한 후에도 실제로 보유하지 않은 상태에서 해당 토큰으로 투표할 수 있습니다.

Step by step walkthrough:
투표 기간 1: 슬롯 60-100 (voting_counter = 1)
투표 기간 2: 슬롯 100-140 (voting_counter = 2)
투표 기간 3: 슬롯 140-180 (voting_counter = 3)

Step 1: 초기 입금 (슬롯 50 - 기간 1 이전): 사용자가 50,000 DRVS 토큰을 입금합니다.
```rust
 // `client_community.rs`의 `update()` 함수 - 입금 중
if available_tokens != self.header.drvs_tokens {
    // 50,000 != 0 - 참(TRUE)

    community_state.header.upgrade()?.drvs_tokens += 50,000;

    if self.header.slot <= community_state.header.voting_start_slot {
        // 0 <= 60 - 참(TRUE)
        self.header.current_voting_counter = 0;
        self.header.current_voting_tokens = 50,000;
    }
    self.header.slot = 50;
    self.header.drvs_tokens = 50,000;
}
```
Step 2: 사용자가 기간 1(슬롯 80)에 투표:
```rust
// voting.rs에서
if data.voting_counter != community_state.header.voting_counter {
    // 1 == 1 - 확인 통과
}

if client_community_state.header.slot <= community_state.header.voting_start_slot {
    // 50 <= 60 - 참(TRUE)
    voting_tokens = client_community_state.header.drvs_tokens; // 50,000
    client_community_state.header.current_voting_counter = 1;
    client_community_state.header.current_voting_tokens = 50,000;
} else {
    voting_tokens = client_community_state.header.current_voting_tokens;
}

if voting_tokens > 0 {
    if client_community_state.header.last_voting_counter >= client_community_state.header.current_voting_counter {
        // 0 >= 1 - 거짓(FALSE), 확인 통과
    }

    // 투표가 성공적으로 행사됨
    community_account_header.voting_incr += 50,000; // 예: 사용자 투표 증가(INCREMENT)
    client_community_state.header.last_voting_counter = 1;
    client_community_state.header.last_voting_tokens = 50,000;
}
```
Step 3: 사용자가 기간 2(슬롯 120)에 50,000 토큰을 추가로 입금:
```rust
// client_community.rs의 update() 함수 - 입금 중
if available_tokens != self.header.drvs_tokens {
    // 100,000 != 50,000 - 참(TRUE)

    community_state.header.upgrade()?.drvs_tokens += 50,000;

    if self.header.slot <= community_state.header.voting_start_slot {
        // 50 <= 100 - 참(TRUE)
        self.header.current_voting_counter = 2; // 기간 2로 업데이트
        self.header.current_voting_tokens = 100,000; //
    }
    self.header.slot = 120; // 확인 후 현재 슬롯으로 업데이트됨
    self.header.drvs_tokens = 100,000;
}
```
Step 4: 사용자가 슬롯 121에서 100,000 토큰 전액 출금
```rust
// client_community.rs의 update() 함수 - 출금 중
if available_tokens != self.header.drvs_tokens {
    // 0 != 100,000 - 참(TRUE)

    community_state.header.upgrade()?.drvs_tokens -= 100,000;

    if self.header.slot <= community_state.header.voting_start_slot {
        //120 <= 100 - 거짓(FALSE) (거짓이므로 여기로 들어가지 않으며 voting tokens은 업데이트되지 않음)
        // if 블록 진입 안 함 - `current_voting_tokens`는 100,000으로 유지됨!
    }
    self.header.slot = 121;
    self.header.drvs_tokens = 0; // 이제 사용자는 0 토큰을 보유함
}
```
공격자는 이 라운드에 투표하지 않았으므로, 투표를 호출하고 실제로 보유하지도 않은 100,000 토큰으로 투표를 행사합니다.
```rust
// voting.rs에서
if data.voting_counter != community_state.header.voting_counter {
    // 2 == 2 - 확인 통과
}

if client_community_state.header.slot <= community_state.header.voting_start_slot {
    // 121 <= 100 - 거짓(FALSE), 여기로 진입하지 않음
} else {
    voting_tokens = client_community_state.header.current_voting_tokens; // 100,000! (오래된 토큰 잔액)
}

if voting_tokens > 0 {
    if client_community_state.header.last_voting_counter >= client_community_state.header.current_voting_counter {
        // 1 >= 2 - 거짓(FALSE), 확인 통과
    }

    // 사용자는 보유하지도 않은 100,000 토큰으로 투표함
    match data.choice {
        VoteOption::DECREMENT => {
            community_account_header.voting_decr += 100,000; //
        }
        VoteOption::INCREMENT => {
            community_account_header.voting_incr += 100,000; //
        }
        _ => community_account_header.voting_unchange += 100,000, //
    }

    client_community_state.header.last_voting_counter = 2;
    client_community_state.header.last_voting_tokens = 100,000;
}
```
**Impact:** 사용자는 실제로 토큰을 보유하지 않고도 투표를 행사할 수 있습니다.

**Recommended Mitigation:** 권장 사항은 전체 시스템에 대한 지식이 쌓이고 감사가 더 진행된 후에 제시할 수 있습니다.

**Deriverse**
커밋에서 수정됨: https://github.com/deriverse/protocol-v1/commit/aa2c1caaaa32b05f63d50056c50040d0edea8290

**Cyfrin:** 검증되었습니다.


### Missing Version Validation for Private Clients Account in `new_private_client`

**Description:** `new_private_client()` 함수는 제공된 `private_clients_acc` 계정이 `root_state`에 지정된 프로토콜 버전과 일치하는지 검증하지 못합니다.

서로 다른 프로토콜 버전은 서로 다른 PDA 시드(버전 포함)를 사용하므로, 각 버전은 고유한 격리된 `private_clients_acc`를 갖습니다. 그러나 이 함수는 이러한 대응 관계를 확인하지 않아 잠재적으로 버전 간 계정 오용을 허용할 수 있습니다.

```rust
// src/program/processor/new_root_account.rs:291
let seed = get_seed_bytes(version, account_type::PRIVATE_CLIENTS);
// get_seed_bytes는 시드에 버전을 포함합니다 (라인 73-77)
let bump_seed = check_new_program_account(
    private_clients_acc,
    &drvs_auth,
    program_id,
    &seed,
    AccountType::PrivateClients,
)?;
```

그러나 `new_private_client()`에서 함수는 다음만 검증합니다:
1. 관리자가 운영자인지
2. 비공개 모드가 활성화되어 있는지
3. 지갑 소유자가 시스템 프로그램인지

다음은 **검증하지 않습니다**:
- `private_clients_acc.owner`가 `program_id`와 일치하는지
- `private_clients_acc` 판별자(discriminator) 태그가 `PRIVATE_CLIENTS`인지
- `private_clients_acc` 판별자 버전이 `root_state.version`과 일치하는지
- `private_clients_acc` PDA 주소가 `root_state.version`에 대해 예상되는 주소와 일치하는지

```rust
// src/program/processor/new_private_client.rs:70-100
let private_clients_acc: &AccountInfo<'_> = next_account_info!(account_iter)?;
// ...
let root_state: &RootState = RootState::from_account_info(root_acc, program_id)?;

// ❌ 여기서 `private_clients_acc`에 대한 검증 없음

if !root_state.is_private_mode() {
    bail!(DeriverseErrorKind::MustBeInPrivateMode);
}

// 버전 확인 없이 private_clients_acc를 직접 사용
let private_clients = private_clients_acc
    .try_borrow_data()
    .map_err(|err| drv_err!(err.into()))?;
```

**Impact:** **버전 간 계정 조작(Cross-Version Account Manipulation):** 한 프로토콜 버전(예: 버전 1)에 대해 승인된 운영자가 잠재적으로 다른 버전(예: 버전 2)의 `private_clients_acc`를 전달하여 다른 버전의 비공개 클라이언트 대기열을 무단으로 수정할 수 있습니다. 이는 프로토콜 버전 간의 의도된 격리를 위반합니다.

**Recommended Mitigation:** 계정 주소가 `root_state.version`에서 파생된 예상 PDA와 일치하는지 확인하십시오.

**Deriverse:** 커밋 [6165c9a](https://github.com/deriverse/protocol-v1/commit/6165c9a9c80d9c895dbda04098d8f3521ab79e2c)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Airdrop Implementation Issues: Unvalidated Ratio and Potential Transfer Direction Mismatch

**Description:** `airdrop` 함수는 사용자가 획득한 `points`를 DRVS 토큰으로 변환하기 위한 것입니다. 그러나 구현에는 다음과 같은 문제가 있습니다:

**Issue 1: 검증되지 않은 비율(Ratio) 매개변수**

`AirdopOnChain` 특성(trait) 구현은 데이터 형식만 검증하고 `ratio` 값은 검증하지 않습니다:

```rust
impl AirdopOnChain for AirdropData {
    fn new(instruction_data: &[u8]) -> Result<&Self, DeriverseError> {
        bytemuck::try_from_bytes::<Self>(instruction_data)
            .map_err(|_| drv_err!(InvalidClientDataFormat))  // 형식 확인만 함
    }
}
```

`ratio`는 경계 확인 없이 바로 사용됩니다:

```rust
amount = ((amount as f64) * data.ratio) as u64;  // ratio에 대한 검증 없음!
```

이로 인해 사용자는 `ratio`를 임의의 값(예: `1,000,000.0` 등)으로 설정할 수 있으며, 이는 토큰 공급 인플레이션이나 기타 계산 오류로 이어질 수 있습니다.

**Issue 2: 잠재적인 전송 방향 불일치 (팀 확인 필요)**


현재 구현은 토큰을 사용자(`drvs_client_associated_token_acc`)로부터 프로그램(`drvs_program_token_acc`)으로 전송합니다:

```rust
let transfer_to_taker_ix = spl_token_2022::instruction::transfer_checked(
    &spl_token_2022::id(),
    drvs_client_associated_token_acc.key,  // FROM: 사용자 계정
    drvs_mint.key,
    drvs_program_token_acc.key,            // TO: 프로그램 계정
    signer.key,                            // Authority: 사용자
    &[signer.key],
    amount,
    decs_count as u8,
)?;

invoke(
    &transfer_to_taker_ix,
    &[
        token_program.clone(),
        drvs_client_associated_token_acc.clone(),  // 사용자 계정 (출처)
        drvs_mint.clone(),
        drvs_program_token_acc.clone(),            // 프로그램 계정 (목적지)
        signer.clone(),                            // 사용자 서명
    ],
)?;
```

이는 다음과 일치하지 않습니다:
- 사용자에게 토큰을 보내는 것을 암시하는 "airdrop"이라는 함수 이름
- 사용자가 누적된 포인트에 대해 보상을 받는 예상 동작

일반적으로 프로토콜은:
- 토큰을 사용자에게 직접 전송하거나
- 사용자에게 토큰을 발행(mint)하거나,
- 토큰을 발행하고 즉시 입금한 다음 `client_state.add_asset_tokens(amount as i64)?;`로 클라이언트 상태를 업데이트해야 합니다.

그럼에도 불구하고, 이 문제는 여전히 팀의 확인이 필요합니다.

**Impact:**
- 검증되지 않은 `ratio`로 인해 사용자는 매우 큰 값(예: `1,000,000.0`)을 설정할 수 있습니다.
- 구현이 초기 설계와 일치하지 않아 사용자에게 손실을 초래할 수 있습니다.

**Recommended Mitigation:**
1. `ratio`가 프로토콜에 의해 제어되어야 하는 경우 명령 데이터에서 제거하고 프로토콜 상태를 기반으로 계산하십시오.
2. `잠재적인 전송 방향 불일치`가 확인되면 전송 로직을 교체하는 것이 좋습니다. 버그가 아닌 경우 함수 이름과 문서를 업데이트하여 이 동작을 명확하게 반영해야 합니다.

**deriverse:**
커밋 [f03ba7](https://github.com/deriverse/protocol-v1/commit/f03ba71ca5c0fcbc481a84e0487381f40f8985ed)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `update_records` always resets `fees_ratio` to 1 whenever a new currency is added

**Description:** `update_records` 함수는 모든 통화 토큰에 대해 더 많은 `ClientCommunityRecord`를 포함하여 `client_community_acc`를 업데이트하는 데 사용됩니다. 그러나 이 함수 내에서 새 통화가 추가될 때마다 `fees_ratio`가 항상 `1.0`으로 재설정됩니다. 이는 함수가 모든 기존 `ClientCommunityRecord` 항목을 반복하고 각 반복 중에 `fees_ratio`를 `1.0`으로 설정하기 때문에 발생합니다.

```rust
            self.data = unsafe {
                Vec::from_raw_parts(
                    dividends_ptr as *mut ClientCommunityRecord,
                    self.header.count as usize,
                    self.header.count as usize,
                )
            };
            for (d, b) in self.data.iter_mut().zip(community_state.base_crncy.iter()) {
                d.crncy_token_id = b.crncy_token_id;
                d.fees_ratio = 1.0;
            }
```
시나리오:
1. 사용자가 x 통화 토큰을 선불로 지불하고 50% 할인을 받을 자격을 얻습니다.
2. 나중에 관리자가 새 통화를 추가합니다.
3. 그 후 사용자가 일정량의 DRVS 토큰을 입금합니다. 그러나 `update_records` 함수가 호출되면 새 통화 추가로 인해 `fees_ratio`가 1.0으로 재설정되어 사용자가 이전에 획득한 할인이 제거됩니다.




**Impact:** 이는 수수료 할인을 위해 이미 선불 결제를 한 사용자에게 손실을 초래하며, 이제 더 높은 수수료를 지불해야 합니다.

**Recommended Mitigation:** 새로 추가된 ClientCommunityRecord 항목만 반복하여 이 문제를 완화하십시오.

**Deriverse:** 커밋 [66c878](https://github.com/deriverse/protocol-v1/commit/66c878370dc8041d6544b8fdee636102ce00fe8c)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Stale Funds Data in Leverage Validation Due to Missing Funding Rate Settlement

**Description:** `perp_change_leverage`에서 코드는 `check_soc_loss`를 호출하지만 레버리지 제약 조건을 검증하기 전에 `check_funding_rate`를 생략합니다. 결과적으로 `check_client_leverage`는 평가를 계산하고 레버리지 제한을 확인할 때 오래된 자금 데이터를 사용합니다. 이는 유효하지 않은 레버리지 변경을 허용할 수 있습니다.

`perp_change_leverage.rs`에서:

```rust
    engine.check_soc_loss(client_state.temp_client_id)?;
    if engine.check_short_margin_call()? < MAX_MARGIN_CALL_TRADES {
        engine.check_long_margin_call()?;
    }
    engine.check_rebalancing()?;
    engine.change_edge_px(client_state.temp_client_id);

    engine.check_client_leverage(client_state.temp_client_id)?;

```

그러나 `fund`가 최신 상태인지 확인하기 위해 레버리지 검증 전에 `check_funding_rate` 호출이 있어야 합니다.
```rust
    pub fn check_funding_rate(&mut self, temp_client_id: ClientId) -> Result<bool, DeriverseError> {
        let info = unsafe { &mut *(self.client_infos.offset(*temp_client_id as isize)) };
        let info5 = unsafe { &mut *(self.client_infos5.offset(*temp_client_id as isize)) };
        let perps = info.total_perps();
        let mut change = false;
        if perps != 0 {
            if self.state.header.perp_funding_rate != info5.last_funding_rate {
                let funding_funds = -(perps as f64
                    * (self.state.header.perp_funding_rate - info5.last_funding_rate))
                    .round() as i64;
                if funding_funds != 0 {
                    info.add_funds(funding_funds).map_err(|err| drv_err!(err))?;
                    info5.funding_funds = info5
                        .funding_funds
                        .checked_add(funding_funds)
                        .ok_or(drv_err!(DeriverseErrorKind::ArithmeticOverflow))?;
                    self.state.header.perp_funding_funds -= funding_funds;
                    change = true;
```

마지막 정산 이후 펀딩 비율이 변경된 경우 `info.funds`는 오래된 상태이므로 잘못된 평가 및 레버리지 제약 조건 확인으로 이어집니다.

```rust
    engine.check_client_leverage(client_state.temp_client_id)?;

    pub fn check_client_leverage(&self, temp_client_id: ClientId) -> DeriverseResult {
        let info = unsafe { &*(self.client_infos.offset(*temp_client_id as isize)) };
        let info2 = unsafe { &*(self.client_infos2.offset(*temp_client_id as isize)) };
        let evaluation = self.calculate_evaluation(info)?;
        let perp_value = self.calculate_perp_value(info);

        self.check_client_leverage_from(info, info2, evaluation, perp_value)
    }
```

반면, `new_perp_order`는 `check_funding_rate`를 올바르게 호출합니다:

```rust

    engine.check_funding_rate(client_state.temp_client_id)?;
    engine.check_rebalancing()?;
    engine.change_edge_px(client_state.temp_client_id);

    engine.check_client_leverage_shift(
        perp_client_info,
        perp_client_info2,
        old_leverage,
        old_perps,
    )?;
```

또한 `perp_order_cancel`도 이 업데이트가 부족합니다.

**Impact:** 레버리지 검증은 오래된 펀딩 지급을 기반으로 할 수 있습니다.

**Recommended Mitigation:** `engine.check_funding_rate(client_state.temp_client_id)?;`를 추가하여 레버리지 검증 전에 `info.funds`가 최신 펀딩 비율 정산을 반영하도록 보장하십시오.

**Deriverse:** 커밋 [74f9650](https://github.com/deriverse/protocol-v1/commit/74f9650ef966b422d95a384bb1e39f4f7bd9cf22)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Fee Discount Calculation Ignores Base Currency Value Differences

**Description:** `fees_deposit`의 수수료 할인 계산은 실제 기본 통화 간의 가치 차이를 고려하지 않고 소수점 제수로 나눈 원시 토큰 금액을 사용합니다. 이는 가치가 서로 다른 여러 기본 통화가 지원되는 경우 불공정한 할인 분배로 이어질 수 있습니다.

`fees_deposit`에서 `prepayment` 금액은 다음과 같이 계산됩니다:
```rust
let dec_factor = get_dec_factor(community_state.base_crncy[crncy_index].decs_count) as f64;
let prepayment = data.amount as f64 / dec_factor;
let fees_discount = community_state.fees_discount(prepayment);
```
계산은 소수점 자릿수에 대해서만 정규화할 뿐 실제 기본 통화의 가치는 고려하지 않습니다. 예를 들어:
- 1000 USDC 입금 ($1000 가치)
- 가치가 낮은 기본 통화 1000 토큰 입금 (개당 $0.001 = 총 $1 가치)
- 1000 SOL 입금은 꽤 불가능할 것임

1000배의 가치 차이에도 불구하고 둘 다 동일한 할인율을 받습니다.

**문서에 언급된 대로 거버넌스를 통해 추가 기본 통화를 지원하도록 프로토콜이 확장됨에 따라 이 문제는 더욱 문제가 될 수 있습니다. 서로 다른 기본 통화는 시장 가치가 매우 다를 수 있지만 현재 구현은 원시 토큰 수를 기준으로 동일하게 취급합니다.**

문서 https://deriverse.gitbook.io/deriverse-v1/launchpad/launchpad#supported-base-currency 에서:

```plaintext
Supported Base Currency
Current Support:

USDC: Circle USD Stablecoin

Future Expansion:

- Additional base currencies may be added through governance

- Multi-denomination support under consideration

- Community can propose new base currencies
```

**Impact:** 불공정한 할인 분배: 가치가 낮은 기본 통화를 입금한 사용자는 훨씬 적은 실제 가치로 가치가 높은 통화를 입금한 사용자와 동일한 할인 임계값을 달성할 수 있습니다.

**Recommended Mitigation:** 상대적 가치를 나타내는 각 기본 통화에 대한 가치 정규화 계수를 저장하십시오.

**Deriverse:** 커밋 [35602457](https://github.com/deriverse/protocol-v1/commit/35602457ebebccacca51749aad6270077724fb38)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Missing ps check causes Dividend Loss

**Description:** perp 엔진의 수수료 분배 로직에서 `match_ask_orders` 및 `match_bid_orders` 함수는 현물 풀에 유동성 공급자가 있는지 확인하지 않고 풀 수수료를 할당합니다. 이는 유동성 공급자가 없을 때 DRVS 토큰 보유자에 대한 배당금 손실을 초래합니다.

코드는 `ps`가 0인지 여부에 관계없이 항상 수수료의 일부를 `pool_fees`에 할당합니다:

```rust
let delta = self.state.header.protocol_fees - prev_fees;
let pool_fees = ((1.0 - self.spot_pool_ratio) * delta as f64) as i64;
self.state.header.protocol_fees -= pool_fees;
self.state.header.pool_fees = self
    .state
    .header
    .pool_fees
    .checked_add(pool_fees)
    .ok_or(drv_err!(DeriverseErrorKind::ArithmeticOverflow))?;
```


**Impact:** `ps == 0`일 때, (배당금을 위해) `protocol_fees`에 100% 할당되어야 하는 수수료가 잘못 분할되어 `(1.0 - spot_pool_ratio) * delta`가 대신 `pool_fees`로 이동합니다.

유동성 공급자가 없을 때 DRVS 토큰 보유자는 수수료의 일부가 풀에 잘못 할당되므로 받아야 할 배당금보다 적게 받습니다.

**Recommended Mitigation:** `match_ask_orders()` 및 `match_bid_orders()` 함수 **모두**에서 풀 수수료를 할당하기 전에 `ps == 0`에 대한 확인을 추가하십시오:

```rust
let delta = self.state.header.protocol_fees - prev_fees;
let pool_fees = if self.state.header.ps == 0 {
    0  // 유동성 공급자가 없을 때 풀 할당 없음
} else {
    ((1.0 - self.spot_pool_ratio) * delta as f64) as i64
};
self.state.header.protocol_fees -= pool_fees;
self.state.header.pool_fees = self
    .state
    .header
    .pool_fees
    .checked_add(pool_fees)
    .ok_or(drv_err!(DeriverseErrorKind::ArithmeticOverflow))?;
```

이렇게 하면 현물 풀에 유동성 공급자가 없을 때(`ps == 0`), 모든 수수료가 `protocol_fees`에 남아 `dividends_allocation` 함수를 통해 DRVS 토큰 보유자에게 배당금으로 적절하게 분배됩니다.

**Note**: 이 수정 사항은 **두** `match_ask_orders()` 및 `match_bid_orders()` 함수 모두에 적용되어야 합니다. 두 함수 모두 동일한 버그 로직을 포함하고 있기 때문입니다.

**Deriverse:** 커밋 [5dd59a](https://github.com/deriverse/protocol-v1/commit/5dd59a3c105207206c196c4abe06d53dc837426d)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Price manipulation during initial ps minting can cause user losses

**Description:** 새 상품이 생성되고 풀이 비어 있을 때(`ps == 0`), 첫 번째 유동성 공급자의 입금 금액은 `last_px`, `best_bid`, `best_ask`에서 파생된 가격을 사용하여 계산됩니다. 악의적인 사용자는 첫 번째 LP 공급자보다 먼저 주문을 하여 `best_bid` 또는 `best_ask`를 조작할 수 있으며, 이로 인해 첫 번째 LP 공급자가 잘못된 가격으로 자금을 입금하게 되어 사용자 자금 손실이 발생할 수 있습니다.

시나리오:
1. 적정 초기 가격을 나타내는 `last_px = 1000`으로 새 상품이 생성됩니다.
2. 공격자는 유동성 공급자가 참여하기 전에 매우 낮은 매도 주문(예: 가격 = 1)을 합니다.
3. 첫 번째 유동성 공급자가 유동성을 추가하려고 시도하고 가격은 `px_f64 = last_px.max(best_bid).min(best_ask)`로 계산되어 `px_f64 = 1000.max(0).min(1) = 1`이 되며, LP 공급자는 이 조작된 가격을 기준으로 통화 토큰을 제공하게 됩니다.
4. 결과: 유동성 공급자는 진정한 공정 시장 가격을 기준으로 의도한 것보다 훨씬 더 많은 자산 토큰을 공급하게 됩니다. 나중에 공격자는 더 유리한 가격으로 자산을 통화로 교환하고 가치를 추출하여 정직한 참여자에게 손실을 입힐 수 있습니다.

**Impact:** 유동성이 조작된 가격으로 공급되어야 하므로 사용자 손실로 이어질 수 있습니다.

**Recommended Mitigation:** 잠재적 완화책은 Uniswap V2에서 사용되는 메커니즘과 유사하게 초기 유동성 제공 중에 사용자가 자산 및 통화 금액을 선택할 수 있도록 하는 것입니다.

**Deriverse:** 커밋 [66c878](https://github.com/deriverse/protocol-v1/commit/66c878370dc8041d6544b8fdee636102ce00fe8c), [cf0573](https://github.com/deriverse/protocol-v1/commit/cf0573fbec84bc4d5b7fa89dd0dad8cab1d01f85)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.




### Incorrect Fee Scaling in `spot_lp`

**Description:** `spot_lp` 명령의 수수료 계산은 자산의 최소 수량을 구매하는 데 필요한 금액을 결정하려고 할 때 `last_px`를 사람이 읽을 수 있는 가격으로 변환할 때 잘못된 소수점 계수를 사용합니다. 코드는 `9 + instr_state.asset_token_decs_count - instr_state.crncy_token_decs_count`여야 하는 올바른 소수점 대신 `get_dec_factor(asset_token_decs_count)`로 `last_px`를 나누어 `9 != crncy_token_decs_count`일 때 잘못된 수수료 계산으로 이어집니다.

`src/program/processor/spot_lp.rs`에서 수수료 계산은 `last_px`를 잘못 변환합니다:
```rust
fees = 1
    + ((instr_state.header.last_px as f64
        / get_dec_factor(instr_state.header.asset_token_decs_count) as f64)
        as i64)
        .max(1);
```

`last_px` 정밀도: `src/program/spot/engine.rs` 및 `src/state/instrument.rs`에 따르면 `last_px`는 정밀도 `dec_factor = 10^(9 + asset_token_decs_count - crncy_token_decs_count)`로 계산됩니다. **따라서 `px`의 최종 정밀도는 `9`가 됩니다.**
```rust
        let mut px = (self.state.header.crncy_tokens as f64 * self.df
            / self.state.header.asset_tokens as f64) as i64; // <= crncy_token_decs_count + 9 + asset_token_decs_count - crncy_token_decs_count - asset_token_decs_count = 9
```

실제로 `px`는 다음인 `dec_factor`로 나누어야 합니다: `get_dec_factor ( 9 + instr_state.asset_token_decs_count - instr_state.crncy_token_decs_count)` 그러나 현재는 `get_dec_factor (asset_token_decs_count)`일 뿐입니다.

`crncy_token_decs_count != 9` (예: `crncy_token_decs_count = 6`)일 때 수수료가 잘못 계산됩니다.

**Example**:
`asset_token_decs_count = 8`이고 `crncy_token_decs_count = 7`이며 `asset_token = 2 crncy (소수점 무시)`라고 가정합니다. 현재 계산 사용:

```
px = 2 * 1e9
(instr_state.header.last_px as f64
        / get_dec_factor(instr_state.header.asset_token_decs_count) as f64) = 2*1e9 / 1e8 = 20.
```

코드는 `asset`의 최소 1단위를 구매하려면 `crncy`의 최소 `~20`단위가 필요함을 시사합니다. 실제로 올바른 값은 소수 정밀도의 차이를 반영하여 `crncy`의 최소 `0.2`단위여야 합니다.


**Impact:** `crncy_token_decs_count != 9`일 때 사용자에게 잘못된 수수료가 청구되어 과다 지불로 이어질 수 있습니다.

**Recommended Mitigation:** 올바른 소수점 계수를 사용하도록 수수료 계산을 업데이트하십시오.

**Deriverse:** 커밋 [65dfb8](https://github.com/deriverse/protocol-v1/commit/65dfb890559a4b20202d3fb43942d8759c2d6476)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### DoS attack exhausting account space and halting spot trading

**Description:** 임시 현물 클라이언트 ID 할당 시스템은 주문이 생성될 때 `client_infos_acc` 및 `client_infos2_acc` 계정 모두에 공간을 예약하지만 할당 해제는 특정 정리 함수가 호출될 때만 발생합니다. 공격자는 이를 악용하여 많은 소규모 주문을 생성하고 정리 경로를 피함으로써 사용 가능한 계정 공간을 점진적으로 고갈시키고 DoS 조건을 유발할 수 있습니다.

**Attack Vector**: 공격자는 여러 계정에 걸쳐 수많은 소규모 지정가 주문을 생성하고 `move_spot_avail_funds`를 호출하지 않으며 `finalize_spot`을 트리거하는 작업을 피할 수 있습니다. 결과적으로 각 임시 클라이언트 ID는 무기한 할당된 상태로 유지됩니다. 시간이 지남에 따라 이는 `client_infos_acc` 및 `client_infos2_acc` 계정 모두의 메모리 공간을 점진적으로 고갈시켜 합법적인 사용자가 정상적으로 작동하지 못하게 합니다.

Solana 계정의 최대 저장 한도는 10MB입니다. `SPOT_CLIENT_INFO_SIZE`(32바이트)의 크기를 기준으로 `client_infos_acc` 및 `client_infos2_acc` 계정은 약 312,500개의 항목을 저장할 수 있습니다. 자산 XY의 가격이 1 USDC인 XY/USDC와 같은 상품의 경우, 단일 임시 클라이언트 ID를 생성하려면 사용자는 최소 2 XY 토큰을 주문해야 합니다. 전체 312,500개 할당에 도달하려면 공격자는 이론적으로 2 × 312,500 토큰이 필요합니다.

**Impact:** 현물 기능의 DoS를 유발할 수 있지만 공격자가 공격을 실행하기에 충분한 자금이 있어야 합니다.

**Recommended Mitigation:** 사용자의 마지막 주문이 실행되는 즉시 공간을 할당 해제하거나 필요할 때 사용자에 대해 `finalize_spot`을 호출할 수 있는 전용 할당 해제 메커니즘을 제공하십시오.

**Deriverse:** 커밋 [e0773a](https://github.com/deriverse/protocol-v1/commit/e0773a57954df33113befd2c2a93bc6f5c4192b6), [105e46](https://github.com/deriverse/protocol-v1/commit/105e463b8ae3aac6440959441c356975f14670bc)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.




### Attacker can cause losses to others by partially closing their position at an unfavorable price

**Description:** 새로운 perp 주문 흐름에서 `check_client_leverage_from`을 사용하여 레버리지가 유효한지 확인합니다. 그러나 이 확인은 다음 조건 중 하나라도 참인 경우에만 실행됩니다:
```rust
current_perps.signum() != old_perps.signum()
current_leverage > old_leverage
old_leverage == i64::MAX as f64
```
사용자는 이 동작을 악용하여 인위적으로 불리한 가격(롱일 때 매우 낮은 가격, 숏일 때 매우 높은 가격)으로 포지션을 매도할 수 있습니다. 이로 인해 사회화된 손실이 다른 사용자에게 부과될 수 있습니다.

Example: 사용자는 청산 지점에서 불과 몇 lamports 떨어진 곳에 새 주문을 생성할 수 있습니다. 예를 들어, 사용자가 롱 포지션일 때 청산에 매우 가까운 매도(ask) 주문을 생성하고 잠재적으로 메커니즘을 악용할 수 있습니다.

구체적으로 매수(bid) 주문이 없거나 무시할 수 있는 수준인 경우, 사용자는 매우 낮은 가격에 매도(ask) 주문을 할 수 있습니다. 그 직후 다른 주소를 사용하여 동일한 저가에 매수 주문을 생성하여 자신의 매도 주문을 구매할 수 있습니다. 이로 인해 손실이 발생하고 이는 다른 참가자에게 사회화됩니다.

이 악용은 의미 있는 매수 유동성이 존재하지 않는다는 조건에 의존합니다.

Scenerio: 사용자가 10배 레버리지로 1 BTC 롱 포지션을 가지고 있습니다. 자금은 -90k이고 엣지 가격은 90k이며 청산 임계값은 0입니다.

이제 매수 주문이 없고 시장 가격(mark price)과 지수 가격(index price)이 모두 $90,100라고 가정합니다.
사용자는 $80,000에 0.5 BTC에 대한 매도(ASK) 주문을 생성합니다.
포트폴리오(perp)에서 포지션은 0.5 BTC가 됩니다.
주문에서 perp는 0.5 BTC가 됩니다.

이 시점에서:
이전 레버리지는 90k / 100 = 900이고 이전 총 perp는 1 BTC입니다.
새 레버리지도 변경되지 않았습니다(여전히 90k / 100 = 900).
old_leverage는 i64::MAX as f64가 아닙니다.
그리고 중요한 것은 현재와 이전 perp 값 모두 동일한 부호(signum)를 갖습니다.

`check_client_leverage_from`은 부호가 동일하게 유지되고 현재 레버리지가 이전 레버리지와 같으며(그 자체로 최대값이 아님) 실행되지 않습니다.

그 후, 사용자는 다른 주소에서 매수 주문을 생성하고 80k에 매도 주문을 구매하여 메커니즘을 악용할 수 있습니다.

나중에 사용자의 원래 포지션인 0.5 BTC가 청산되면 손실이 발생하여 다른 사용자에게 사회화됩니다.

**Impact:** 다른 사용자에게 손실을 초래하므로 영향이 큽니다.


**Recommended Mitigation:**
1. `margin_call`이 거짓이고 손실이 0보다 큰 경우 트랜잭션을 되돌려야 하는지 확인하는 검사를 추가하십시오.
```rust
     if loss > 0 && !args.margin_call {
           return Err;
     }
```
2. 사용자가 롱 포지션에 있고 매도 주문을 생성할 때 주문 가격은 임계 가격보다 커야 합니다. 반대로 사용자가 숏 포지션에 있는 경우 매수 주문 가격은 임계 가격보다 낮아야 합니다.

**Deriverse:** 커밋 [26d87c](https://github.com/deriverse/protocol-v1/commit/26d87c791fd7edf5971dfbcd41b702ee1a1218b8), [319890](https://github.com/deriverse/protocol-v1/commit/3198908f063537fc8b9ff6c9e1b6bcd8933e0847)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.




### Makers' rebates are not paid in case of swap

**Description:** 주문이 일치될 때 더 나은 가격을 제공하는 것을 기준으로 AMM과 오더북 중에서 선택합니다. 오더북이 존재하지 않는 `swap`의 경우 AMM과만 거래하고, 존재하는 경우 둘 중 더 나은 것을 선택합니다.. 임시 클라이언트의 기본 계정을 생성합니다.. 사용 가능한 주문을 찾는 동안 `client-community`를 [`none`](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/processor/swap.rs#L180)으로 전달합니다.
```rust
// swap.rs 내부
    if buy {
        if price > px || engine.cross(price, OrderSide::Ask) {
            (q, traded_sum, trades, _) =
                engine.match_orders(price, qty, &mut client_state, None, OrderSide::Ask)?;
        }
```
`engine.rs`의 `match_orders()` 함수 내부에서 `client-community`가 `None`이었기 때문에 `fee-rate & ref-discounts`는 [0](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/spot/engine.rs#L1370)입니다.
```rust
        let fill_static_args = FillStaticArgs {
            fee_rate,
            ref_discount,
            taker_client_id: client.id,
            side,
        };
```
오더북이 더 나은 가격을 제공하는 경우 해당 경로를 거칩니다.. `match_orders()` 내부; [여기](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/spot/engine.rs#L1463)
```rust
                   if remaining_qty > 0 {
                        self.fill(
                            &mut line,
                            &mut remaining_qty,
                            &mut sum,
                            &mut trades,
                            &mut total_fees,
                            &mut total_rebates,
                            &mut fees_prepayment,    // 0
                            &fill_static_args, // 0으로 된 수수료율 포함
                            //ref_discount,
                            //client.id,
                            //fee_rate,
                            //side,
                        )?;
                    }
                }
```
`fill()` 내부에서 [`fill_static_args.fee_rate == 0.0`](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/spot/engine.rs#L1218)이므로 수수료를 청구/공제하지 않으며, 이 경우 둘 다(수수료율 및 리베이트)가 [0](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/spot/engine.rs#L1219)이 되었기 때문에 메이커에게 리베이트를 주지 않습니다.... 메이커의 주문은 리베이트를 받지 않고 채워지며 사용자는 프로토콜 수수료를 지불하지 않고 거래했습니다.
```rust
            let (fees, rebates) = if fill_static_args.fee_rate == 0.0 {
                (0, 0)
            } else {
                (
                    self.get_fees(
                        fees_prepayment,
                        traded_crncy,
                        fill_static_args.fee_rate,
                        fill_static_args.ref_discount,
                    )?,
                    ((traded_crncy as i128 * self.rebates_rate) >> DEC_PRECISION) as i64,
                )
            };
```

**Impact:** 프로토콜은 스왑에서 의도한 수수료를 잃고 메이커는 리베이트를 받지 못합니다.

**Recommended Mitigation:** 현재 조건을 다음과 같이 변경하십시오:
```rust
let (fees, rebates) = if self.fee_rate == 0.0 {
    (0, 0)
} else {
    (
        self.get_fees(
            fees_prepayment,
            traded_crncy,
            fill_static_args.fee_rate,
            fill_static_args.ref_discount,
        )?,
        ((traded_crncy as i128 * self.rebates_rate) >> DEC_PRECISION) as i64,
    )
};
```
**Deriverse**
커밋에서 수정됨: [b368de](https://github.com/deriverse/protocol-v1/commit/b368de93d68a635ddca7355ad9f81a69102f66d9)

**Cyfrin:** 검증되었습니다.


### `perp-change-leverage` uses stale `perp-underlying-px`

**Description:** [perp_change_leverage](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/processor/perp_change_leverage.rs) 함수는 [check_long_margin_call](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/processor/perp_change_leverage.rs#L99), [check_short_margin_call](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/processor/perp_change_leverage.rs#L98) 및 [check_client_leverage](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/processor/perp_change_leverage.rs#L104)와 같이 업데이트된/최신 `perp_underlying_px`를 포함하는 다양한 확인을 수행하기 전에 [engine.state.set_underlying_px(accounts_iter)?]()를 호출하지 못합니다. 이 함수는 `perp_underlying_px` 필드를 현재 [last_px](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/state/instrument.rs#L229)(오라클 지원이 비활성화되었으므로)와 동기화할 책임이 있습니다. 이 호출이 없으면 모든 후속 계산은 다른 함수에 의해 마지막으로 업데이트된 시점의 오래된 `perp_underlying_px` 값을 사용합니다.

```rust
    //@audit  perp_withdraw, perp_order_cancel, perp_mass_cancel 등에 존재함
    //engine.state.set_underlying_px(accounts_iter)?;
    engine.check_soc_loss(client_state.temp_client_id)?;
    if engine.check_short_margin_call()? < MAX_MARGIN_CALL_TRADES {
        engine.check_long_margin_call()?;
    }
    engine.check_rebalancing()?;
    engine.change_edge_px(client_state.temp_client_id);

    engine.check_client_leverage(client_state.temp_client_id)?;
```
대조적으로 `perp_underlying_px`를 포함하는 다른 함수들은 기초 가격을 마지막 가격으로 업데이트하기 위해 [set_underlying_px](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/state/instrument.rs#L224)를 올바르게 호출합니다(`new_perp_order`, `perp-withdraw`, `perp-order-cancel`, `perp-mass-cancel` 등).
참고: 유사한 문제가 `perp-statistics-reset`에 있습니다.

**Impact:** `perp_underlying_px`와 관련된 모든 다음 계산 및 확인은 오래된 값을 기반으로 하므로 트레이들에게 유리하거나 불리하게 작용할 수 있어 문제가 될 수 있습니다.

**Recommended Mitigation:** `perp_underlying_px`를 `last_px`와 동기화하는 누락된 가격 업데이트를 추가하십시오.
```rust
engine.state.set_underlying_px(accounts_iter)?;
```
**Deriverse:** 커밋에서 수정됨: https://github.com/deriverse/protocol-v1/commit/4f7bc8ac68325aa93b339ff91c0ac794ea17ffd9

**Cyfrin:** 검증되었습니다.


### Shorts can withdraw full available funds instead of being restricted to margin call limits in `perp-withdraw`

**Description:** [perp_withdraw.rs](https://github.com/deriverse/protocol-v1/blob/f611019674f668681c3dbe074f8b7bc2ba846c97/src/program/processor/perp_withdraw.rs#L99)에서 마진 콜 감지 로직은 숏 포지션에 대해 감지하지 못합니다. 여기서 [is_long_margin_call()](https://github.com/deriverse/protocol-v1/blob/f611019674f668681c3dbe074f8b7bc2ba846c97/src/program/processor/perp_withdraw.rs#L147)은 롱 및 숏 포지션을 모두 확인하는 대신 두 번 호출됩니다:
```rust
        engine.check_long_margin_call()?;
        engine.check_short_margin_call()?;
        //@audit 둘 다 롱임, 하나는 숏이어야 함
        let margin_call = engine.is_long_margin_call() || engine.is_long_margin_call();
        if !margin_call {
            engine.check_rebalancing()?;
}
```
사용자에게 마진 콜(손실/담보 부족) 상태인 숏 무기한 포지션이 있을 때 `margin_call` 변수는 다음 이유로 잘못하여 거짓(false)으로 평가됩니다:
- `is_long_margin_call()`은 숏 포지션에 대해 `false`를 반환합니다.
- 중복 호출도 `false`를 반환합니다.
- 결과: [margin_call = false || false = false]

**Impact:**
1. 우회된 출금 제한
출금 금액 계산은 마진 콜 상태에 따라 다릅니다:
```rust
let amount = if margin_call {
    // 제한된 출금 - 요구 사항을 초과하는 증거금만
    let margin_call_funds = funds.min(engine.get_avail_funds(client_state.temp_client_id, true)?);
    if margin_call_funds <= 0 {
        bail!(ImpossibleToWithdrawFundsDuringMarginCall);
    }
    // 제한된 금액
} else if data.amount == 0 {
    funds  // 전체 자금 사용 가능 - 제한 없음
} else {
    // 일반 출금
};
```
이를 통해 마진 콜 상태의 숏 포지션을 가진 사용자는 마진 콜 한도에 제한되는 대신 전체 사용 가능한 자금을 인출할 수 있습니다. 이를 통해 숏 포지션의 잠재적 손실을 충당하기 위해 잠가야 하는 담보를 추출할 수 있습니다.

2. 잘못된 리밸런싱 실행
마진 콜 상태의 숏 포지션에 대해 `margin_call`이 잘못하여 `false`인 경우, `check_rebalancing()`이 호출되어서는 안 될 때 호출됩니다..

**Recommended Mitigation:** 현재 조건을 다음과 같이 교체하십시오:
```rust
let margin_call = engine.is_long_margin_call() || engine.is_short_margin_call();
```
**Deriverse:** 커밋에서 수정됨: https://github.com/deriverse/protocol-v1/commit/4f7bc8ac68325aa93b339ff91c0ac794ea17ffd9

**Cyfrin:** 검증되었습니다.


### Margin call detection functions ignore liquidation threshold

**Description:** 마진 콜 활성 여부를 결정하는 데 사용되는 `is_long_margin_call()` 및 `is_short_margin_call()` 함수는 `liquidation_threshold` 매개변수를 고려하지 않습니다. 이는 시스템이 마진 콜을 감지할 때와 실제 마진 콜이 발생할 때 사이에 불일치를 생성합니다.
```rust
pub fn is_long_margin_call(&self) -> bool {
```rust
    let root = self.long_px.get_root::<i128>();
    if root.is_null() {
        false
    } else {
        self.state.header.perp_underlying_px < (root.max_node().key() >> 64) as i64
    }
}

pub fn is_short_margin_call(&self) -> bool {
    let root = self.short_px.get_root::<i128>();
    if root.is_null() {
        false
    } else {
        self.state.header.perp_underlying_px > (root.min_node().key() >> 64) as i64
    }
}
```
실제 청산 로직(Actual Liquidation Logic):
```rust
pub fn check_long_margin_call(&mut self) -> Result<i64, DeriverseError> {
    let mut trades = 0;
    // 청산 임계값 적용
    let margin_call_px = (self.state.header.perp_underlying_px as f64
        * (1.0 - self.state.header.liquidation_threshold)) as i64;

    loop {
        let root = self.long_px.get_root::<i128>();
        if root.is_null() || trades >= MAX_MARGIN_CALL_TRADES {
            break;
        }
        let node = root.max_node();
        let px = (node.key() >> 64) as i64;

        // 엣지 가격과 임계값 조정 가격 비교
        if px > margin_call_px {
            // ... 청산 로직 ...
        }
    }
    Ok(trades)
}
```
동일한 `liquidation_threshold`가 `check_short_margin_call` 중에도 적용됩니다.


마진 콜이 발생하고 있는 상황에서도 마진 콜이 거짓(false)으로 유지되어 다음과 같은 문제가 발생합니다:
* `perp_spot_price_for_withdrawal` 동결 메커니즘이 활성화되어야 할 때 활성화되지 않습니다.
* 무기한 출금 흐름에서 `get_avail_funds`는 `margin_call`이 거짓일 때만 호출됩니다.
* 마진 콜 활성 조건 중에도 리밸런싱이 호출됩니다.

**Impact:** 이는 사용자가 활성 마진 콜 중에 평소보다 더 많은 금액을 인출할 수 있게 하여 의도된 인출 제한을 효과적으로 우회합니다.

**Recommended Mitigation:** 마진 콜 감지 함수에 청산 임계값을 사용하여 정확한 마진 콜 감지를 보장하고 잘못된 출금 및 리밸런싱 동작을 방지하십시오.

**Deriverse:** 커밋 [4f7bc8](https://github.com/deriverse/protocol-v1/commit/4f7bc8ac68325aa93b339ff91c0ac794ea17ffd9)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Multiple inconsistencies in `sell-market-seat`

**Description:** [sell_market_seat](https://github.com/deriverse/protocol-v1/blob/4f7bc8ac68325aa93b339ff91c0ac794ea17ffd9/src/program/processor/sell_market_seat.rs#L89)는 코드베이스의 다른 perp 함수와 비교할 때 두 가지 불일치를 포함합니다:

Issue 1: 펀딩 비율 계산에 오래된 기초 가격 사용:
- 이 함수는 `set_underlying_px()`를 통해 기초 가격을 먼저 업데이트하지 않고 `change_funding_rate()`를 호출합니다:
```rust
// sell_market_seat.rs
let mut engine = PerpEngine::new(
    &ctx,
    signer,
    system_program,
    community_state.perp_fee_rate(),
    community_state.margin_call_penalty_rate(),
    community_state.spot_pool_ratio(),
)?;

let instrument: &mut InstrAccountHeader = InstrAccountHeader::from_account_info(
    ctx.instr_acc,
    program_id,
    Some(data.instr_id),
    root_state.version,
)?;

// @audit set_underlying_px() 없이 change_funding_rate()가 먼저 호출됨, 오래된 가격이 사용됨
engine.change_funding_rate();
```
펀딩 비율 계산은 `perp_underlying_px`에 의존하여 무기한 가격과 현물 가격 간의 편차를 결정합니다. 이 값을 업데이트하지 않으면 글로벌 펀딩 비율은 오래된 가격 데이터를 사용하여 계산됩니다.

Issue 2: 클라이언트의 펀딩 비율 업데이트 누락:
`change_funding_rate()`를 호출한 후, 이 함수는 `check_funding_rate(client_state.temp_client_id)`를 호출하지 않아 종료하는 클라이언트에게 보류 중인 펀딩 지급을 적용하지 않습니다:
```rust
engine.change_funding_rate();  // 글로벌 펀딩 비율 업데이트

let info = client_state.perp_info()?;

//@audit check_funding_rate(client_state.temp_client_id)가 호출되지 않음
// 보류 중인 펀딩이 클라이언트의 자금에 적용되지 않음

if info.perps != 0 || info.in_orders_funds != 0 || info.in_orders_perps != 0 {
    bail!(ImpossibleToClosePerpPosition);
}

// 보류 중인 펀딩을 포함하지 않는 오래된 자금 값 사용
let collactable_losses = info.funds().min(client_state.perp_info4()?.soc_loss_funds);
engine.state.header.perp_insurance_fund += collactable_losses;

// 클라이언트는 보류 중인 펀딩이 적용되지 않은 상태로 자금을 받음
client_state.add_crncy_tokens((info.funds() - collactable_losses).max(0))?;
```
`check_funding_rate()` 함수는 클라이언트의 포지션을 기반으로 보류 중인 펀딩 지급을 적용합니다:

- funding_rate > 0일 때: 롱이 숏에게 지불
- funding_rate < 0일 때: 숏이 롱에게 지불
함수가 `info.perps == 0`(오픈 포지션 없음)을 요구하더라도, 클라이언트는 이전에 포지션을 보유했을 때의 미수령 펀딩 지급이 아직 `info.funds` 잔액에 정산되지 않았을 수 있습니다.

**Impact:** 오래된 기초 가격으로 인해 프로토콜이 잘못된 글로벌 펀딩 비율을 적용하게 되며, 좌석(seat) 철수 전에 클라이언트의 펀딩 지급을 업데이트하지 않음으로써 오래된 `info.funds`를 사용하게 되어, 사용자가 포지션 종료 시 받아야 할 돈보다 적게 받거나 더 많이 받게 됩니다.

**Recommended Mitigation:** 글로벌 비율과 사용자 자금이 최신 상태가 되도록 두 함수를 모두 먼저 호출하십시오.

**Deriverse:** 커밋 [319890](https://github.com/deriverse/protocol-v1/commit/3198908f063537fc8b9ff6c9e1b6bcd8933e0847)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Improper header handling in `SpotFeesReport` logging causes DoS on swap instruction

**Description:** `match_orders`에서 `ref_payment`는 `client.header`가 None인 경우 기본값을 0으로 처리하지만, `ref_client_id`는 `ok_or`를 사용하고 `header`가 None이면 오류를 반환합니다. 이로 인해 `total_fees` > 0이고 `header`가 None일 때 함수가 실패합니다.

```rust
            solana_program::log::sol_log_data(&[bytemuck::bytes_of::<SpotFeesReport>(
                &SpotFeesReport {
                    tag: log_type::SPOT_FEES,
                    fees: total_fees,
                    ref_payment,
                    ref_client_id: client
                        .header
                        .as_ref()
                        .ok_or(drv_err!(DeriverseErrorKind::ClientPrimaryAccountMustBeSome))?
                        .ref_client_id,
                    ..SpotFeesReport::zeroed()
                },
            )]);
```

이는 사용자가 스왑 명령을 사용할 때 항상 발생합니다.

**Impact:** 스왑 명령이 실행되고 `total_fees > 0`이 참일 때마다 영구적인 DOS가 발생하므로 영향이 큽니다.

**Recommended Mitigation:** 헤더가 None일 때 되돌리는(reverting) 대신 이를 우아하게 처리하는 것을 고려하십시오.

**Deriverse:** 커밋 [4f7bc8](https://github.com/deriverse/protocol-v1/commit/4f7bc8ac68325aa93b339ff91c0ac794ea17ffd9)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.




### Missing Signer Verification in `voting_reset` Function Allows Unauthorized Execution

**Description:** `src/program/processor/voting_reset.rs`의 `voting_reset` 함수는 `admin` 계정의 공개 키가 운영자 주소와 일치하는지만 확인하고 `admin` 계정이 실제로 트랜잭션의 서명자인지는 확인하지 않습니다. **이로 인해 공격자는 운영자 주소와 일치하지만 서명이 필요하지 않은 계정을 전달하여 `voting reset` 명령을 실행할 수 있으며, 적절한 권한 없이 중요한 투표 매개변수를 재설정할 수 있습니다.**

```rust
    let root_state: &RootState = RootState::from_account_info(root_acc, program_id)?;

    if root_state.operator_address != *admin.key {
        bail!(InvalidAdminAccount {
            expected_address: root_state.operator_address,
            actual_address: *admin.key
        })
    }
```

**Impact:** 공격자는 운영자의 개인 키를 소유하지 않고도 `voting_reset`을 실행하여 적절한 권한 없이 중요한 투표 매개변수를 재설정할 수 있습니다.


**Recommended Mitigation:** 운영자 주소 확인 전에 서명자 확인 검사를 추가하십시오.

```rust
    if !admin.is_signer {
        bail!(MustBeSigner {
            address: *admin.key
        })
    }
```
**Deriverse:** 커밋 [bfc0c96](https://github.com/deriverse/protocol-v1/commit/bfc0c96991d10b19da49c63483999955a810c62b)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Using the underlying price instead of the perp price can cause problems

**Description:** 현재 우리는 매수/매도 작업과 펀딩 비율 계산에만 perp 가격을 사용합니다. 그러나 perp 가치를 계산할 때는 기초 가격에 의존합니다:

perp 출금 시나리오:
시장 가격(mark price)은 100k이고 지수 가격(index price)도 100k입니다.
사용자는 100k에 10배 레버리지로 1 BTC 무기한 계약을 매수합니다. 명목 포지션은 1 BTC이고 사용자 계정 잔액은 -90k가 됩니다.
나중에 지수 가격이 120k로 상승하고 시장 가격은 변동이 없습니다.
이 시점에서 사용자는 18k를 인출할 수 있게 되어 계정 잔액은 -108k가 됩니다.
그러나 실제 perp 가격은 여전히 1 BTC당 100k입니다. 하지만 사용자의 자금은 -108k이고 포지션은 100k만 커버할 수 있습니다.

이 문제는 perp 가치를 계산하기 위해 기초 가격을 사용하고 있기 때문에 발생합니다.

**Impact:** 사용자가 실제 perp 가격을 기준으로 해야 하는 것보다 더 많은 자금을 인출할 수 있으므로 영향이 큽니다.

**Recommended Mitigation:** 이 문제를 완화하기 위해 기초 가격 대신 perp 가격을 사용하는 것을 고려하십시오.

**Deriverse:** 커밋 [1ef948](https://github.com/deriverse/protocol-v1/commit/1ef948af18b47f3e502d0054d760212d4b6263f1)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.




### Spot price manipulation can lead to unfair liquidations

**Description:** 무기한 상품이 오라클 피드 없이 구성된 경우, 시스템은 청산 계산을 위해 현물 시장의 `last_px`를 `perp_underlying_px`로 사용합니다. 현물 가격(`last_px`)은 오더북 주문이나 AMM 거래를 통해 조작될 수 있어 공격자가 건전한 무기한 포지션의 불공정한 청산을 트리거할 수 있습니다. 이 취약점은 악의적인 행위자가 조작된 가격으로 청산을 강제하여 사용자에게 상당한 금전적 손실을 입힐 수 있게 합니다.

오라클이 구성되지 않은 경우 현물 가격이 직접 무기한 기초 가격이 됩니다:

조작된 `perp_underlying_px`를 기반으로 청산이 트리거됩니다:

```rust
pub fn check_long_margin_call(&mut self) -> Result<i64, DeriverseError> {
    let margin_call_px = (self.state.header.perp_underlying_px as f64
        * (1.0 - self.state.header.liquidation_threshold)) as i64;

    // edge_px > margin_call_px이면 포지션이 청산됨
    // ...
}

pub fn check_short_margin_call(&mut self) -> Result<i64, DeriverseError> {
    let margin_call_px = (self.state.header.perp_underlying_px as f64
        * (1.0 + self.state.header.liquidation_threshold)) as i64;

    // edge_px < margin_call_px이면 포지션이 청산됨
    // ...
}
```

**Impact:** 건전한 포지션이 불공정하게 청산될 수 있어 영향이 큽니다. 또한 공격자는 이익을 위해 이 동작을 악용할 수 있습니다.

**Recommended Mitigation:** 외부 오라클이나 TWAP을 사용하는 것을 고려하십시오.

**Deriverse:** 커밋 [fc0013](https://github.com/deriverse/protocol-v1/commit/fc0013bc5add2c0ad0eac3f31bfc37f32c87c07c)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.



\clearpage
## Medium Risk


### `create_account` can be dosed with pre-funding

**Description:** `new_holder_account`와 같은 명령은 `invoke_signed`와 함께 `system_instruction::create_account`를 사용하여 PDA를 생성합니다. Solana 시스템 프로그램 규칙에 따르면 대상이 이미 **0이 아닌 lamports 또는 데이터로 존재**하는 경우 `create_account`가 실패합니다.

공격자는 PDA를 미리 계산하고 SOL을 전송하여 후속 `create_account`가 실패하도록 유도하고 계정 초기화 및 다음 로직 흐름을 차단할 수 있습니다.

```rust
let account_size = size_of::<HolderAccountHeader>();
invoke_signed(
    &system_instruction::create_account(
        holder_admin.key,
        holder_acc.key,
        Rent::default().minimum_balance(account_size),
        account_size as u64,
        program_id,
    ),
    &[holder_admin.clone(), holder_acc.clone()],
    &[&[HOLDER_SEED, holder_admin.key.as_ref(), &[bump_seed]]],
)
.map_err(|err| drv_err!(err.into()))?;
```

참조: https://x.com/r0bre/status/1887939134385172496

참고: `system_instruction::create_account`는 `src/program/processor/new_holder_account.rs, src/program/processor/new_root_account.rs, src/program/create_client_account.rs, src/state/candles.rs, src/state/instrument.rs, src/state/perps/perp_trade_header.rs, src/state/spots/spot_account_header.rs, src/state/token.rs`에도 존재합니다.

**Impact:**
- 모든 행위자는 최소한의 lamport 금액으로 미리 자금을 조달하여 중요한 PDA(`holder accounts, root accounts, headers, tokens, etc`) 생성을 차단할 수 있습니다.

**Proof of Concept:**
```rust
/// PDA DOS 공격 취약점을 시연하기 위한 테스트
///
/// 이 테스트는 공격자가 PDA 주소를 미리 계산하고 미리 자금을 조달하여
/// 관리자가 Holder Account를 생성하지 못하도록 하는 것을 시뮬레이션합니다.
#[tokio::test]
async fn test_holder_account_dos_attack() {
    let program_id = Pubkey::from_str(PROGRAM_ID).unwrap();

    // 프로그램 테스트 환경 생성
    let mut test = ProgramTest::new(
        "smart_contract",
        program_id,
        processor!(smart_contract::process_instruction),
    );

    // 관리자 및 공격자 계정 설정
    let admin_signer = Keypair::from_bytes(CLIENTS[0].kp).unwrap();
    let attacker = Keypair::new();

    // 초기 SOL 잔액이 있는 계정을 테스트 환경에 추가
    test.add_account(
        admin_signer.pubkey(),
        Account {
            lamports: 10_000_000_000, // 10 SOL
            data: vec![],
            owner: system_program::id(),
            executable: false,
            rent_epoch: 0,
        },
    );
    test.add_account(
        attacker.pubkey(),
        Account {
            lamports: 10_000_000_000, // 10 SOL
            data: vec![],
            owner: system_program::id(),
            executable: false,
            rent_epoch: 0,
        },
    );

    // 공격자는 프로토콜과 동일한 시드를 사용하여 PDA 주소를 미리 계산합니다.
    let (pda_address, _) = Pubkey::find_program_address(
        &[HOLDER_SEED, admin_signer.pubkey().as_ref()],
        &program_id
    );

    println!("🎯 Attacker pre-calculated PDA address: {}", pda_address);

    // 테스트 컨텍스트 시작
    let mut ctx = test.start_with_context().await;

    // 공격자는 PDA 주소에 미리 자금을 조달하여 잔액이 0이 아니도록 합니다.
    let transfer_instruction = system_instruction::transfer(
        &attacker.pubkey(),
        &pda_address,
        1_000_000, // 0.001 SOL 전송, 최소 잔액 흉내
    );

    let transfer_transaction = Transaction::new_signed_with_payer(
        &[transfer_instruction],
        Some(&attacker.pubkey()),
        &[&attacker],
        ctx.last_blockhash,
    );

    let result = ctx.banks_client.process_transaction(transfer_transaction).await;
    assert!(result.is_ok(), "Attacker should be able to pre-fund PDA address");

    println!("✅ Attacker successfully pre-funded PDA address");

    // 관리자가 Holder Account 생성 시도
    println!("Admin attempts to create Holder Account");
    let mut tx = Transaction::new_with_payer(
        &[Instruction::new_with_bytes(
            program_id,
            &[0], // NewHolderInstruction
            vec![
                AccountMeta {
                    pubkey: admin_signer.pubkey(),
                    is_signer: true,
                    is_writable: true,
                },
                AccountMeta {
                    pubkey: pda_address,
                    is_signer: false,
                    is_writable: true,
                },
                AccountMeta {
                    pubkey: system_program::ID,
                    is_signer: false,
                    is_writable: false,
                },
            ],
        )],
        Some(&admin_signer.pubkey()),
    );
    tx.sign(&[&admin_signer], ctx.last_blockhash);

    // PDA가 선점되어 있으므로 관리자 생성은 실패해야 합니다.
    let result = ctx.banks_client.process_transaction(tx).await;

    if result.is_err() {
        println!("🚨 DOS ATTACK SUCCESSFUL: Admin failed to create holder account");
    } else {
        println!("❌ Test failed: Admin creation should have failed but didn't");
    }
}

```

테스트 결과:
```plaintext
running 1 test
[2025-10-30T01:37:51.898533000Z INFO  solana_program_test] "smart_contract" builtin program
🎯 Attacker pre-calculated PDA address: 5zAb3ZhCNjTwoMxK39fheR4fXbuArMJVN9bhaeQkZHrq
[2025-10-30T01:37:52.011771000Z DEBUG solana_runtime::message_processor::stable_log] Program 11111111111111111111111111111111 invoke [1]
[2025-10-30T01:37:52.011914000Z DEBUG solana_runtime::message_processor::stable_log] Program 11111111111111111111111111111111 success
✅ Attacker successfully pre-funded PDA address
Admin attempts to create Holder Account
[2025-10-30T01:37:52.013452000Z DEBUG solana_runtime::message_processor::stable_log] Program Drvrseg8AQLP8B96DBGmHRjFGviFNYTkHueY9g3k27Gu invoke [1]
[2025-10-30T01:37:52.013485000Z DEBUG solana_runtime::message_processor::stable_log] Program Drvrseg8AQLP8B96DBGmHRjFGviFNYTkHueY9g3k27Gu invoke [1]
[2025-10-30T01:37:52.013963000Z DEBUG solana_runtime::message_processor::stable_log] Program 11111111111111111111111111111111 invoke [1]
[2025-10-30T01:37:52.014048000Z DEBUG solana_runtime::message_processor::stable_log] Program 11111111111111111111111111111111 invoke [2]
[2025-10-30T01:37:52.014070000Z DEBUG solana_runtime::message_processor::stable_log] Create Account: account Address { address: 5zAb3ZhCNjTwoMxK39fheR4fXbuArMJVN9bhaeQkZHrq, base: None } already in use
[2025-10-30T01:37:52.014101000Z DEBUG solana_runtime::message_processor::stable_log] Program 11111111111111111111111111111111 failed: custom program error: 0x0
{"code":100,"error":{"Custom":0},"location":{"file":"src/program/processor/new_holder_account.rs","line":63},"msg":"System error Custom program error: 0x0"}
[2025-10-30T01:37:52.014285000Z DEBUG solana_runtime::message_processor::stable_log] Program Drvrseg8AQLP8B96DBGmHRjFGviFNYTkHueY9g3k27Gu failed: custom program error: 0x64
[2025-10-30T01:37:52.014303000Z DEBUG solana_runtime::message_processor::stable_log] Program Drvrseg8AQLP8B96DBGmHRjFGviFNYTkHueY9g3k27Gu failed: custom program error: 0x64
🚨 DOS ATTACK SUCCESSFUL: Admin failed to create holder account
test instructions::test_holder_dos::test_holder_account_dos_attack ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 1 filtered out; finished in 0.24s
```

**Recommended Mitigation:** PDA에 대해 `create_account`에 의존하지 마십시오. 대신 다음을 수행하여 새로운 PDA 흐름과 미리 자금이 조달된 PDA 흐름을 모두 지원하십시오:
1. 자금 조달 (필요한 경우),
2. 할당
3. invoke_signed로 PDA 지정.

**Deriverse:** 커밋 [df95974](https://github.com/deriverse/protocol-v1/commit/df95974c9c967e79e35403b313ee2e299f01a2ef) 및 [9b8e442](https://github.com/deriverse/protocol-v1/commit/9b8e442ec18843e4e8b77975486d37a19bf9c9e2)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Fixed token account size causes initialization failures for token accounts whose mints have token 22 extensions are active

**Description:**  `token.rs`의 `create_token` 함수는 민트(mint)가 Token 2022 확장을 활성화했는지 여부에 관계없이 SPL 토큰 계정을 생성할 때 165바이트의 하드코딩된 크기를 사용합니다. 이는 추가 계정 공간이 필요한 확장이 있는 민트에 대해 초기화 실패를 유발합니다.

```rust
let spl_lamports = rent.minimum_balance(165);
invoke(
    &system_instruction::create_account(
        creator.key,
        program_acc.key,
        spl_lamports,
        165, // @audit 왜 165로 하드코딩되어 있습니까?
        token_program_id.key,
    ),
    &[creator.clone(), program_acc.clone()],
)
```
상수 165바이트는 레거시 SPL-Token 크기입니다. 165바이트만 할당하면 [여기](https://github.com/solana-program/token-2022/blob/50a849ef96634e02208086605efade0b0a9f5cd4/program/src/processor.rs#L187-L191)에서 `InvalidAccountData`로 `initialize_account3`가 실패합니다.


**Impact:**
- 확장이 있는 Token 2022 민트에 대해 토큰 생성이 실패합니다.

**Recommended Mitigation:** 민트의 소유자가 token 2022 프로그램인 경우, 하드코딩된 165바이트 대신 크기를 먼저 계산한 다음 계산된 공간을 할당하십시오.

```rust
let account_size = if token_program == TokenProgram::Token2022 {
    use spl_token_2022::extension::StateWithExtensions;
    StateWithExtensions::<spl_token_2022::state::Account>::try_get_account_len(mint)?
} else {
    165 // 표준 SPL 토큰 계정 크기
};

let spl_lamports = rent.minimum_balance(account_size);
invoke(
    &system_instruction::create_account(
        creator.key,
        program_acc.key,
        spl_lamports,
        account_size as u64,
        token_program_id.key,
    ),
    &[creator.clone(), program_acc.clone()],
)
```
**Deriverse:** 커밋에서 수정됨: https://github.com/deriverse/protocol-v1/commit/6e8b8b011e69c356a81e4cc1f8cbe14adf5bf0a6

**Cyfrin:** 검증되었습니다.


### Accounts may be created with incorrect rent-exemption due to `Rent::default` usage

**Description:** 이 DOS 취약점은 계정 생성 중 하드코딩된 `rent` 매개변수를 사용하여 발생하며, 그 결과 임대료 면제 요건을 충족하지 못하는 자금 부족 계정이 생성됩니다.

결함이 있는 구현은 임대료 계산을 위해 온체인 `Rent sysvar` 대신 `Rent::default`에 의존하여 임대료 면제에 충분한 자금이 없는 계정을 생성합니다.

```rust
    let rent = &Rent::default();
    ...
    let spl_lamports: u64 = rent.minimum_balance(165);

```

구현은 온체인 Rent `sysvar`(예: `Rent::get` 또는 rent `sysvar` 계정 전달)를 절대 읽지 않으므로 계산된 최소 잔액은 클러스터의 실제 임대료 매개변수를 반영하지 않을 수 있습니다.


**Impact:** 결과적으로 계정이 임대료 면제 없이 생성되어 DOS를 유발할 수 있습니다.

**Recommended Mitigation:** `Rent::default` 대신 `Rent::get`을 사용하는 것이 좋습니다.

**Deriverse:** 커밋 [d319206](https://github.com/deriverse/protocol-v1/commit/d319206f269efdd6ba1de8ae5966e02f9ffcfec7)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `change_points_program_expiration` is permissionless

**Description:** `change_points_program_expiration` 함수는 `admin` 계정이 실제로 트랜잭션에 서명했는지 확인하지 못합니다. 함수는 제공된 관리자 계정 키가 루트 상태의 예상 `operator_address`와 일치하는지 확인하지만, `admin.is_signer`를 사용하여 계정이 서명자인지 확인하지는 않습니다. 주석은 관리자가 서명자여야 함을 나타내지만 (// [*`create_account` can be dosed with pre-funding*](#createaccount-can-be-dosed-with-prefunding) - Admin (Signer)), 이 요구 사항은 코드에서 시행되지 않으므로 일반 사용자는 실제 관리자의 서명 없이 관리자 공개 키를 전달하고 이 명령을 실행할 수 있습니다.
```rust
// 관리자 키가 일치하는지만 확인하고 관리자가 실제로 트랜잭션에 서명했는지는 확인하지 않음;
if root_state.operator_address != *admin.key {
    bail!(InvalidAdminAccount {
        expected_address: root_state.operator_address,
        actual_address: *admin.key,
    });
}
// 누락됨: if !admin.is_signer { ... }
```
**Impact:** 관리자 전용 기능에 대한 무단 액세스.

**Recommended Mitigation:** `admin` 주소가 저장된 주소와 일치할 뿐만 아니라 서명자인지 확인하십시오.
```rust
    if !admin.is_signer {
        bail!(InvalidAdminAccount {
            expected_address: root_state.operator_address,
            actual_address: *admin.key,
        });
    }
```
**Deriverse:** 커밋에서 수정됨: [8a479a](https://github.com/deriverse/protocol-v1/commit/8a479a06d86536deeb65eeeb2379df5520ba4595)

**Cyfrin:** 검증되었습니다.


### Anyone can sell anyone else's market seat

**Description:** `sell_market_seat` 함수에서 `check_signer` 매개변수가 잘못하여 false로 설정되어 있어, 사용자가 적절한 권한 없이 다른 사용자의 마켓 좌석(market seat)을 강제로 판매할 수 있습니다.
```rust
let mut client_state = ClientPrimaryState::new_for_perp(
    program_id,
    client_primary_acc,
    &ctx,
    signer,
    system_program,
    0,
    false,  // alloc = false (올바름)
    false,  // @audit 왜 false입니까? 다른 사람의 것을 제공할 수 있습니까?
)?;
```
`check_signer` 플래그를 false로 설정했기 때문에 `new_for_perp`는 `ClientPrimaryAccountHeader::from_account_info_unchecked`를 호출합니다. `from_account_info_unchecked`는 `signer` 계정이 서명자인지만 확인하고, 서명자 계정이 실제로 `ClientPrimaryAccountHeader`에 저장된 `wallet_address`인지 확인하지 않습니다.. 즉, 누구나 누구의 유효한 `ClientPrimaryState`를 전달하고 동의 없이 좌석을 판매할 수 있습니다.

공격 시나리오
- 공격자는 마켓 좌석이 있는 `client_primary_acc`를 찾습니다.
- 공격자는 자신의 지갑을 `signer`로, 피해자의 `client_primary_acc`를 사용하여 `sell_market_seat`를 호출합니다.
- 서명자가 클라이언트 계정을 소유하고 있는지 확인하는 유효성 검사가 수행되지 않습니다.
- 피해자의 마켓 좌석이 판매되고 자금은 피해자의 계정으로 이동하며, 공격자는 동의 없이 피해자의 포지션을 닫았습니다.

**Impact:** 모든 사용자가 다른 사용자의 좌석을 닫을 수 있습니다.

**Recommended Mitigation:** `buy_market_seat`에서 수행한 것처럼 `is_signer` 플래그를 true로 설정하여 전달하십시오.
```rust
// ...기존 코드...
let mut client_state = ClientPrimaryState::new_for_perp(
    program_id,
    client_primary_acc,
    &ctx,
    signer,
    system_program,
    0,
    false,
    true,   // check_signer = true
)?;
// ...기존 코드...
```
**Deriverse**
커밋에서 수정됨: [9a5678](https://github.com/deriverse/protocol-v1/commit/9a56789cd5f5b479022da7c4da97f8ef1ee6aaf8)

**Cyfrin:** 검증되었습니다.


### Voting is allowed even after voting period's end time.

**Description:** `voting()` 함수는 투표를 처리하고 기록하기 전에 투표 기간이 종료되었는지 확인하지 않습니다. 함수는 맨 마지막에 있는 `finalize_voting()` 호출에서만 투표 종료 시간을 확인하지만, 그 시점에서는 사용자의 투표가 이미 행사되어 투표 집계에 포함되었습니다. 이를 통해 마지막 사용자는 공식 투표 기간이 만료된 후에도 다른 사람이 최종 확정을 트리거하기 전에 함수를 호출하는 한 투표를 제출할 수 있습니다.
```rust
//이 기간의 종료 시간이 지났더라도 여기에 투표가 포함되었습니다.
        match data.choice {
            VoteOption::DECREMENT => {
                community_account_header.voting_decr += voting_tokens;
            }
            VoteOption::INCREMENT => {
                community_account_header.voting_incr += voting_tokens;
            }
            _ => community_account_header.voting_unchange += voting_tokens,
        }
....
// 투표 최종 확정 호출은 나중에 이루어지므로, 늦은 사용자의 투표가 포함되었습니다.
        community_state.finalize_voting(time, clock.slot as u32)?;
```
**Impact:**
- 악의적인 행위자는 투표 기간이 공식적으로 종료된 후에도 투표할 수 있습니다.
- 늦은 투표자(많은 토큰을 보유한 경우)는 현재 투표 집계에 대한 지식을 얻고 자신에게 유리하게 결과에 영향을 미치도록 전략적으로 투표할 수 있습니다.

**Recommended Mitigation:** 투표 행사 및 `finalize voting` 호출 전에 사용자가 공식 `CommunityState.header.voting_end_time` 시간 이후에 투표하려고 할 때 오류를 발생시키는 확인을 구현하십시오.

**Deriverse**
커밋에서 수정됨: [a5194d]9https://github.com/deriverse/protocol-v1/commit/a5194d26218f0828e83481ebb2a6f7071773b13a)

**Cyfrin:** 검증되었습니다.


### Missing Slippage Protection in Market Seat Buy/Sell Operations

**Description:** `buy_market_seat()` 및 `sell_market_seat()` 함수는 실행 시점의 현재 `perp_clients_count`를 기반으로 좌석 가격을 동적으로 계산하지만 슬리피지 보호를 제공하지 않습니다. 사용자는 최대/최소 허용 가격을 지정할 수 없어 예기치 않은 가격 변동에 노출됩니다.

`buy_market_seat()`에서 좌석 가격은 다음과 같이 계산됩니다:

```rust
let seat_price = PerpEngine::get_place_buy_price(
    instrument.perp_clients_count,
    instrument.crncy_token_decs_count,
)?;

instrument.seats_reserve += seat_price;
let price = data.amount + seat_price;
// ... 검증 없이 가격이 공제됨
client_state.sub_crncy_tokens(price)?;
```

마찬가지로 `sell_market_seat()`에서 판매 가격이 계산됩니다:

```rust
let seat_price = PerpEngine::get_place_sell_price(
    instrument.perp_clients_count,
    instrument.crncy_token_decs_count,
)?;

client_state.add_crncy_tokens(seat_price)?;
```

가격 계산 함수(`get_place_buy_price()` 및 `get_place_sell_price()`)는 좌석이 추가될 때마다 가격이 상승하는 본딩 커브(bonding curve) 모델을 사용합니다. 가격은 다음을 기준으로 계산됩니다:

```rust
pub fn get_place_buy_price(supply: u32, dec_factor: u32) -> Result<i64, DeriverseError> {
    let df = get_dec_factor(dec_factor);
    Ok(get_reserve(supply + 1, df)? - get_reserve(supply, df)?)
}

pub fn get_place_sell_price(supply: u32, dec_factor: u32) -> Result<i64, DeriverseError> {
    let df = get_dec_factor(dec_factor);
    Ok(get_reserve(supply, df)? - get_reserve(supply - 1, df)?)
}
```
문제점:
1. 트랜잭션 제출과 실행 사이에 다른 합법적인 시장 트랜잭션이 `perp_clients_count`를 변경하여 실제 실행 가격이 사용자가 예상한 것과 다를 수 있습니다.
2. 사용자는 구매 시 최대 허용 가격이나 판매 시 최소 허용 가격을 지정할 방법이 없습니다.
3. 가격은 사용자 기대치에 대한 검증 없이 계산되어 즉시 적용됩니다.
4. 시장 활동이 활발한 기간 동안, 동시 좌석 구매/판매는 상당한 가격 변동(price drift)을 일으킬 수 있습니다.

**Impact:**
- **예측할 수 없는 실행:** 사용자는 정상적인 시장 상황에서도 자신의 트랜잭션이 허용 가능한 가격으로 실행될 것이라는 보장이 없습니다.
- **열악한 사용자 경험:** 사용자는 합법적인 동시 시장 활동으로 인한 불리한 가격 변동으로부터 자신을 보호할 수 없습니다.

**Recommended Mitigation:** 사용자가 명령 데이터에 최대/최소 허용 가격을 지정할 수 있도록 하여 슬리피지 보호를 추가하고, 실행 전에 계산된 가격을 이 제한과 비교하여 검증하십시오.

**Deriverse:** 커밋 [a8181f3](https://github.com/deriverse/protocol-v1/commit/a8181f37e475eb1144f39490b62e45a476b2455d)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Missing Quorum Requirement in Governance Voting

**Description:** `finalize_voting()`의 거버넌스 투표 시스템에는 정족수 요구 사항이 없어 총 토큰 공급량의 최소 비율이 참여했는지 확인하지 않고 상대적인 투표 수에 따라 프로토콜 매개변수를 변경할 수 있습니다. 이를 통해 소수의 토큰 보유자가 프로토콜 거버넌스 결정을 통제할 수 있어 시스템의 탈중앙화 특성을 약화시킵니다.


```rust
let decr = community_account_header.voting_decr;
let incr = community_account_header.voting_incr;
let unchange = community_account_header.voting_unchange;
let tag = community_account_header.voting_counter % 6;

if decr > unchange && decr > incr {
    // 감소(DECREMENT) 적용 - 정족수 확인 없음
    match tag {
        0 => { community_account_header.spot_fee_rate -= 1; }
        // ... 기타 매개변수
    }
} else if incr > unchange && incr > decr {
    // 증가(INCREMENT) 적용 - 정족수 확인 없음
    match tag {
        0 => { community_account_header.spot_fee_rate += 1; }
        // ... 기타 매개변수
    }
}
```

`voting_supply`가 추적되어 `drvs_tokens`(총 공급량)로 설정되지만, 투표에 충분한 토큰이 참여했는지 확인하는 데는 사용되지 않습니다:

```rust
community_account_header.voting_supply = community_account_header.drvs_tokens;
```


**The Problem:**
1. 투표 결과를 적용하기 전에 최소 참여 임계값(정족수)을 확인하지 않습니다.
2. 다른 사람이 아무도 투표하지 않으면 소량의 토큰을 가진 단일 투표자가 프로토콜 변경을 결정할 수 있습니다.
3. 총 투표권(`decr + incr + unchange`)은 `voting_supply` 또는 최소 임계값과 비교되지 않습니다.
4. 이는 중요한 결정에 의미 있는 커뮤니티 참여가 필요한 일반적인 거버넌스 모범 사례를 위반합니다.

**Impact:**
- **거버넌스 공격 벡터:** 악의적인 행위자는 활동이 적은 기간을 기다려 불리한 매개변수 변경을 강행할 수 있습니다.
- **탈중앙화 약화:** 투표 시스템은 결정이 커뮤니티의 의미 있는 부분을 대표하도록 보장하지 못합니다.

**Recommended Mitigation:** 투표 결과를 적용하기 전에 최소 참여를 검증하는 정족수 요구 사항을 추가하십시오. 정족수는 총 투표 공급량의 백분율이어야 합니다.
```rust
        let total_votes = decr + incr + unchange;
        let voting_supply = community_account_header.voting_supply;

        // 정족수 확인 추가 (예: 최소 5% 참여 필요)
        const MIN_QUORUM_PERCENTAGE: i64 = 5; // 투표 공급량의 5%
        let min_quorum = (voting_supply * MIN_QUORUM_PERCENTAGE) / 100;
```

**deriverse:**
커밋 [b4e1045](https://github.com/deriverse/protocol-v1/commit/b4e10453eb5fe7a0866a264fac4187167640f998)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Inconsistent Token Account Address Requirements Between Deposit and Withdraw

**Description:** `deposit()` 및 `withdraw()` 함수는 토큰 계정 주소와 관련하여 일관성 없는 요구 사항을 가지고 있습니다:

- **`deposit()`**: 서명자가 소유한 모든 토큰 계정에서의 입금을 허용하며, 연결된 토큰 계정(ATA)을 요구하지 않습니다.
- **`withdraw()`**: 서명자의 ATA 주소로만 출금해야 합니다.


**Deposit Function - ATA 요구 사항 없음:**

```rust
// deposit.rs 라인 145-154
let old_token = check_spl_token(
    client_associated_token_acc,
    program_token_acc,
    token_state,
    token_program_id,
    mint,
    signer.key,  // 소유자가 서명자인지만 확인
    &pda,
    data.token_id,
)?;
```

`check_spl_token()` 함수(`helper.rs` 내)는 다음만 검증합니다:
- 토큰 계정 소유자가 서명자임 (라인 207)
- 민트 주소 일치
- 토큰 프로그램 일치

```rust
        if client_token != *mint.key {
            bail!(InvalidMintAccount {
                token_id: token_state.id,
                expected_address: *mint.key,
                actual_address: client_token,
            });
        }

        let client_token_owner = client_token_acc.data.borrow()[32..].as_ptr() as *const Pubkey;

        if *client_token_owner != *signer {
            bail!(DeriverseErrorKind::InvalidTokenOwner {
                token_id: token_state.id,
                address: *client_token_acc.key,
                expected_adderss: *signer,
                actual_address: *client_token_owner,
            });
        }
```

**주소가 ATA인지 확인하지 않으므로**, 사용자는 자신이 제어하는 모든 토큰 계정에서 입금할 수 있습니다.

**Withdraw Function - ATA 필요:**

```rust
// withdraw.rs 라인 127-138
let expected_address = get_associated_token_address_with_program_id(
    signer.key,
    mint_acc.key,
    token_program_id.key,
);
if expected_address != *client_associated_token_acc.key {
    bail!(InvalidAssociatedTokenAddress {
        token_id: token_state.id,
        expected_address: expected_address,
        actual_address: *client_associated_token_acc.key,
    });
}
```

`withdraw()` 함수는 목적지가 서명자의 ATA 주소일 것을 명시적으로 요구하며 다른 토큰 계정 주소는 거부합니다.

이는 토큰 계정의 소유자가 불변인 Token-2022에서는 잘 작동합니다. 그러나 레거시 토큰 프로그램에서 사용자는 모든 토큰 계정의 소유권을 이전할 수 있습니다. 자금을 입금한 사용자가 더 이상 연결된 토큰 계정을 소유하고 있지 않은 경우, ATA 소유자가 서명자여야 함을 확인하므로 출금 기능을 사용하여 자금을 인출할 수 없게 됩니다.
여기 참조: https://github.com/solana-program/token/blob/main/program/src/processor.rs#L441

**Impact:** 사용자 자금 손실을 초래할 수 있습니다. 시나리오는 다음과 같습니다:
1. 사용자가 ATA를 보유하고 있었으며 해당 ATA의 소유권을 이전했습니다.
2. 사용자가 다른 토큰 계정을 사용하여 입금했습니다.
3. 사용자가 더 이상 원래 ATA를 소유하지 않고 우리의 로직은 ATA 소유자가 사용자와 일치해야 한다고 확인하므로, 모든 출금 시도는 실패합니다. 결과적으로 사용자는 자금을 인출할 수 없습니다.

**Recommended Mitigation:** 다음 중 하나를 수행하십시오:
- 두 함수를 일관성 있게 만드십시오 (ATA 요구/모든 토큰 계정 허용)
- 현재 비대칭이 의도적인 경우 왜 입금은 유연성을 허용하고 출금은 ATA를 요구하는지 명확하게 문서화하십시오.

**Deriverse:** 커밋 [1e3d88](https://github.com/deriverse/protocol-v1/commit/1e3d8857266eb82b1920b5edd09009041e96fafe)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Silent Error Handling in `clean_generic` Introduces Multiple Risks

**Description:** `ClientPrimaryState`의 `clean_generic()` 함수는 `if let Ok(header)`를 사용하여 계정 파싱에 대한 조용한(silent) 오류 처리를 구현합니다. 이는 잘못된 상태 업데이트, 계정 비동기화, 근본적인 명령 문제의 마스킹으로 이어질 수 있는 세 가지 잠재적 문제를 야기합니다.

이 함수는 계정이 쌍(`maps_acc` 계정 다음에 `client_infos_acc`)으로 제공될 것으로 예상합니다. 루프 카운터는 성공적으로 처리된 각 쌍에 대해 2씩 증가하며 반복당 2개의 계정이 소비된다고 가정합니다.

```rust
            if let Ok(header) = header { // 조용한 실패 처리
                counter += 2;
                let client_infos_acc = next_account_info!(accounts_iter)?;

                SpotTradeAccountHeader::<SPOT_CLIENT_INFOS>::validate(
                    client_infos_acc,
                    program_id,
                    Some(header.instr_id),
                    version,
                )?;

                self.client_infos_acc = client_infos_acc;
                self.maps_acc = maps_acc;
                self.resolve_instr(client_infos_acc, false)?;
                self.finalize_spot()?;
            }
        }
        Ok(())
    }

```


- Issue 1: 명령 수준 문제를 마스킹하는 조용한 오류 처리

**단일 파싱 실패는 전체 명령이 잘못 형성되었거나 계정 구조가 근본적으로 잘못되었음을 나타낼 수 있습니다. 실행을 조용히 계속함으로써 함수는 이러한 중요한 오류를 마스킹하고 추가 위험을 초래할 수 있습니다.**

- Issue 2: 계정 쌍 구조에 대한 잘못된 가정

조용한 오류 처리는 `[bad_maps_acc, good_maps_acc, good_client_infos_acc]`와 같은 특정 계정 순서 패턴을 가정합니다. 그러나 이 가정은 취약하며 종종 부정확합니다. 실제 계정 순서는 이 패턴을 따르지 않을 수 있습니다. 예를 들어 `maps_acc`가 실패하면 다음 계정은 다른 `maps_acc`(가정된 대로)일 수 있지만 이전 쌍의 `client_infos_acc`일 수도 있습니다.

- Issue 3: 반복자-카운터 비동기화

`maps_acc` 파싱에 실패하면 `next_account_info!(accounts_iter)?`는 이미 반복자를 진행시켜 하나의 계정을 소비했지만, 파싱 실패 시 카운터는 증가하지 않습니다. 이로 인해 비동기화가 생성됩니다:

- **반복자 위치**: 1 계정 진행 (`maps_acc` 소비됨)
- **카운터 값**: 변경되지 않음 (증가 없음)
- **예상 동작**: 카운터는 소비된 계정을 추적해야 함


6개의 계정 `[map1(bad), map2(ok), client_info2(ok), map3(bad), map3(ok), client_info3(ok)]`과 `length = 6`이 주어진 경우:

1. 반복 1: `map1(bad)` 읽기 → 파싱 실패 → 카운터 = 1, 반복자 위치 1
2. 반복 2: `map2(ok)` 읽기 → 파싱 성공 → 카운터 = 3, 반복자 위치 3 (`client_info2` 읽은 후)
3. 반복 3: `map3(bad)` 읽기 → 파싱 실패 → 카운터 = 3, 반복자 위치 4
4. 반복 4: `map3(ok)` 읽기 → 파싱 성공 → 카운터 = 5, 반복자 위치 6 (`client_info3` 읽은 후)
5. 루프 조건 `counter < length` (5 < 6)은 여전히 참이지만 반복자는 소진됨

이러한 비동기화는 다음을 유발할 수 있습니다:
- 의도한 범위를 벗어난 계정 읽기
- 잘못된 계정 쌍 처리
- 반복자가 소진되었을 때 `next_account_info!`가 호출되면 잠재적인 패닉 발생

**Impact:**
1. **명령 무결성 위반**: 조용한 오류 처리는 중요한 명령 수준 문제를 마스킹하여 잘못 형성되거나 악의적인 명령이 부분적으로 실행되도록 허용합니다.

2. **잘못된 계정 쌍**: 실패한 계정 뒤에 올바른 쌍이 온다는 가정은 종종 위반됩니다.

3. **반복자-카운터 비동기화**: 파싱 실패 시 반복자 위치와 카운터가 불일치하여 다음을 유발합니다:
   - 후속 반복에서 잘못된 계정 읽기
   - 계정을 순서 없이 처리하거나 필요한 계정 건너뛰기
   - 소진된 반복자에서 `next_account_info!`가 호출될 때 잠재적 패닉

**Recommended Mitigation:** 함수는 조용히 계속하는 대신 파싱 오류 시 빠르게 실패(fail fast)해야 합니다. 이는 세 가지 문제를 모두 해결합니다.

**Deriverse:** 커밋 [1c2f2a5](https://github.com/deriverse/protocol-v1/commit/1c2f2a5c3f6475eea2fdbbd9f1b77595fbd622e1)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `ClientCommunityState::update` function does not update the `rate` before calculating new `dividends_value`

**Description:** `ClientCommunityState::update` 함수는 사용자가 보유한 DRVS 토큰 양에 따라 배당금을 분배할 책임이 있습니다:
```rust
for (i, d) in self.data.iter_mut().enumerate() {
    let amount = (((community_state.base_crncy[i].rate - d.dividends_rate)
        * self.header.drvs_tokens as f64) as i64)
        .max(0);
    d.dividends_value += amount;
}
```
`community_state.base_crncy[i].rate` 값은 배당금 할당 함수를 통해 매시간 업데이트됩니다.

그러나 마지막 할당 이후 1시간 이상 지났지만 배당금 할당 함수가 호출되지 않고 현재 구현이 이러한 경우에 `rate`를 업데이트하지 않을 가능성이 있습니다. 결과적으로 계산은 오래된 `rate`에 의존할 수 있으며 `dividends_value`는 오래된 `community_state.base_crncy[i].rate`를 사용하여 계산될 수 있습니다.

시나리오:
1. 방금 1시간이 지났지만 배당금 할당 함수가 호출되지 않았습니다.
2. 사용자가 DRVS 토큰을 출금하기 위해 출금을 호출하며, 이 또한 비율(rate)을 업데이트하지 않습니다.
3. 결과적으로 사용자는 계산이 오래된 비율을 기반으로 하기 때문에 더 낮은 배당금 가치를 받습니다.

위에 설명된 시나리오는 출금 기능과 관련이 있지만 입금에서도 유사한 문제가 발생할 수 있습니다:
1. 방금 1시간이 지났지만 배당금 할당 함수가 호출되지 않았습니다.
2. 사용자가 DRVS 토큰을 구매하고 입금을 호출합니다. 그러나 우리는 이전 비율을 사용자의 `dividends_rate`로 기록합니다.
3. 그러면 사용자는 배당금 할당 함수를 호출하고 이전 시간 동안 해당 DRVS 토큰을 보유하지 않았기 때문에 받을 자격이 없는 배당금을 받을 수 있습니다.

**Impact:** 업데이트 전에 배당금 할당 함수가 호출되지 않고 1시간 이상 경과한 경우, 일부 사용자는 불공정하게 손실을 입는 반면 다른 사용자는 받을 자격이 없는 이익을 얻을 수 있습니다.

**Recommended Mitigation:** `dividends_value` 및 `dividends_rate`를 업데이트하기 전에 1시간이 지난 경우 `community_state.base_crncy[i].rate`를 업데이트하십시오.

**Deriverse:** 커밋 [ca593e](https://github.com/deriverse/protocol-v1/commit/ca593e2bc30b93a7a4a53c69ceb5b91a282c8955)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.




### Lack of slippage in `spot_lp` liquidity operations due to relying on the changing `header.crncy_tokens` and `header.asset_tokens`

**Description:** `spot_lp` 함수는 유동성을 추가하거나 제거할 때 슬리피지 보호가 부족합니다.

대부분의 경우 함수는 **사용자가 최소 출력 금액이나 최대 허용 슬리피지를 지정하도록 허용하지 않고** 실행 시점의 현재 풀 상태를 기반으로 필요한 자산 및 통화 토큰을 계산합니다. 이는 트랜잭션 제출과 실행 사이의 풀 상태 변경으로 인해 사용자에게 불리한 실행 가격을 노출시킵니다.

`src/program/processor/spot_lp.rs`에서 유동성 추가 시(라인 189-194), 함수는 현재 풀 상태를 기반으로 필요한 토큰을 직접 계산합니다:

```rust
trade_crncy_tokens = ((instr_state.header.crncy_tokens + instr_state.header.pool_fees)
    as f64
    * amount as f64
    / instr_state.header.ps as f64) as i64;
trade_asset_tokens = (instr_state.header.asset_tokens as f64 * amount as f64
    / instr_state.header.ps as f64) as i64;
```

풀의 asset_tokens 및 crncy_tokens는 정상적인 거래 작업 중에 수정됩니다 (`change_tokens` 및 `change_mints`를 통해 `engine.rs`에서 확인됨).

```rust
                self.change_tokens(traded_qty, side)?;
                self.change_mints(traded_mints, side)?;
                self.log_amm(traded_qty, traded_mints, side);
```

그러나 spot_lp 함수는:
- 사용자가 최소 허용 출력 금액을 지정하도록 허용하지 않습니다 (Uniswap V2의 `amountAMin` 및 `amountBMin`과 유사).
- 실행 가격이 허용 가능한 범위 내에 있는지 확인하지 않습니다.

**Impact:** 사용자는 풀 상태 변경으로 인해 유동성을 추가/제거할 때 예기치 않은 손실을 입을 수 있습니다.

**Recommended Mitigation:** SpotLpData 구조체에 슬리피지 보호 매개변수를 추가하고 `trade_asset_tokens` 및 `trade_crncy_tokens` 계산 후 유효성 검사를 구현하십시오.

**Deriverse:** 커밋 [337383](https://github.com/deriverse/protocol-v1/commit/3373834b7988ba52810515f514664e0c80ca2c8c)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Referral Incentives Disabled for All Legitimate Users During Any Liquidation

**Description:** 엔진이 청산이 필요한 상품(`is_long_margin_call` 또는 `is_short_margin_call`)을 감지하면, 이후의 모든 `new_perp_order`에 대해 `margin_call` 플래그가 true로 설정됩니다. 이 플래그는 `match_{ask,bid}_orders`로 변경 없이 전달되며, `true`인 동안 추천금 지급을 비활성화합니다. 결과적으로 청산과 관련 없는 일반 주문을 제출하는 사용자를 포함한 모든 사용자는 청산 후보가 남아 있는 한 추천 보상을 받거나/생성하는 것을 중단하게 됩니다. 이 전체 스위치는 실제 청산 거래에만 의도되었을 가능성이 큽니다.

`new_perp_order.rs`에서 코드는 `margin_call = engine.is_long_margin_call() || engine.is_short_margin_call();`를 설정합니다.

```rust
    let margin_call = engine.is_long_margin_call() || engine.is_short_margin_call();
    if !margin_call {
        engine.state.header.perp_spot_price_for_withdrowal = engine.state.header.perp_underlying_px;
    }
```

해당 부울(boolean)은 `MatchOrdersStaticArgs`를 통해 `PerpEngine::match_{ask,bid}_orders`로 전달됩니다.

```rust
        if engine.cross(price, OrderSide::Ask) {
            (remaining_qty, _, ref_payment) = engine.match_ask_orders(
                Some(&mut client_community_state),
                &MatchOrdersStaticArgs {
                    price,
                    qty: data.amount,
                    ref_discount,
                    ref_ratio: header.ref_program_ratio,
                    ref_expiration: header.ref_program_expiration,
                    ref_client_id: header.ref_client_id,
                    trades_limit: 0,
                    margin_call,
                    client_id: client_state.temp_client_id,
                },
            )?;
        }
```

추천 리베이트는 `perp_engine.rs`에서 `!args.margin_call`을 조건으로 합니다: `margin_call`이 `true`이면 ref_payment는 강제로 0이 됩니다.
```rust
        let ref_payment = if self.time < args.ref_expiration && !args.margin_call {
            ((fees - rebates) as f64 * args.ref_ratio) as i64
        } else {
            0
        };
```

청산 루틴(`check_long_margin_call, check_short_margin_call`)도 명시적으로 `margin_call: true`를 전달하지만 청산으로 인한 체결과 일반 주문 간에 구분이 없습니다.
```rust
    if buy {
        if engine.check_short_margin_call()? < MAX_MARGIN_CALL_TRADES {
            engine.check_long_margin_call()?;
        }
    } else if engine.check_long_margin_call()? < MAX_MARGIN_CALL_TRADES {
        engine.check_short_margin_call()?;
    }
```

따라서 청산 후보가 있으면 청산 대상이 누구인지에 관계없이 모든 트레이더에 대한 추천 보상이 전역적으로 차단됩니다.


**Impact:** 다른 계정이 청산 중일 때마다 합법적인 사용자는 예상되는 추천 인센티브를 잃습니다. 즉각적인 자금 손실은 아니지만 스트레스 상황에서 모든 참가자에게 영향을 미치는 시스템적인 인센티브 실패를 나타냅니다.

**Recommended Mitigation:** `margin_call` 플래그를 실제로 청산 흐름의 일부인 거래로 제한하십시오.

**Deriverse:** 커밋 [bc9bd6](https://github.com/deriverse/protocol-v1/commit/bc9bd6ab49dd15dcf2c3d83559fa4ad0bd6777d9)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Slippage Guard Could be Too Loose For Leveraged Perp Markets

**Description:** 무기한 "시장가" 주문은 현물 주문에 사용되는 것과 동일한 하드코딩된 `±12.5%` 가격 제한으로 대체됩니다. 이 한도는 현물에는 허용될 만하지만 레버리지된 perp 거래에는 위험할 정도로 느슨합니다. 사용자는 기준 가격에서 12.5% 떨어진 곳에서 한 번에 체결될 수 있으며 사용자의 레버리지 요소만큼 손실이 확대됩니다. 레버리지가 높을 때 이 슬리피지를 강화하거나 더 엄격한 한도를 적용할 수 있는 기능이 없습니다.

`new_perp_order.rs`에서 비지정가 매수는 기본값이 `px + (px >> 3)`이고 매도는 `px - (px >> 3)`입니다. 즉, 현재 기초 가격의 ±12.5%입니다.

```rust
    let (price, min_tokens) = PerpParams::get_settings(
        if data.order_type == OrderType::Limit as u8 {
            data.price
        } else if buy {
            px + (px >> 3)
        } else {
            px - (px >> 3)
        },
        data.amount,
        if buy { OrderSide::Bid } else { OrderSide::Ask },
        px,
        data.ioc,
        engine.dc,
    )?;
```

현물 주문은 동일한 ±12.5% 창을 재이용하며, 현물 포지션은 레버리지가 없기 때문에 이는 허용됩니다.

그러나 Perp 주문은 프로토콜 최대(`MAX_PERP_LEVERAGE`)까지 레버리지될 수 있습니다. 레버리지된 시장가 주문을 `-12.5%(또는 +12.5%)`에서 채우면, 사용자가 시장 가격 근처에서 거래하려고 의도했더라도 극단적인 시장 상황에서 즉시 사용자 증거금의 상당 부분을 소비하고 의도하지 않은 청산을 거의 유발할 수 있습니다.

또한 사용자는 시장가 주문을 완전히 피하지 않는 한(`limit+IOC` 사용) 더 엄격한 한도를 구성할 방법이 없는데, 이는 비현실적입니다. 많은 트레이더는 여전히 시장가 주문이 합리적인 슬리피지 보호를 받을 것이라고 기대합니다.

**Impact:** 변동성이 크거나 유동성이 적은 상황에서 시장가 주문에 의존하는 레버리지 트레이더는 매우 불리한 가격(최대 `12.5%` 차이)으로 체결되어 막대한 손실이나 즉각적인 청산으로 이어질 수 있습니다.

**Recommended Mitigation:** 궁극적으로 팀이 얼마나 엄격하게 할지 결정해야 하지만, 현재의 12.5% 일괄 제한은 레버리지 시장 위험 관리와 맞지 않으므로 강화해야 합니다.

가능하면 레버리지 인식 슬리피지 한도(예: 레버리지가 증가함에 따라 허용 오차 축소)를 구현하거나 사용자가 프로토콜 정의 최대값 내에서 사용자 지정 슬리피지 한도를 지정하도록 허용하십시오.

또 다른 권장 사항은 더 안전한 기본값을 선택하기 위해 주요 CEX/DEX perp 제품을 벤치마킹하는 것입니다.

**Deriverse:** 커밋 [b2ff47aa](https://github.com/deriverse/protocol-v1/commit/b2ff47aa00fc88daec2eb15339751e27fb23a723)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Referral discount is not applied when `fees_prepayment` is zero in `PerpEngine::fill`

**Description:** `PerpEngine::fill`에서 `fees_prepayment`가 0일 때 `ref_payment`를 `ref_address`로 보내더라도 추천 할인을 적용하지 않습니다.

```rust
                if *fees_prepayment == 0 {
                    let fee = (traded_crncy as f64 * self.fee_rate) as i64;
                    taker_info.sub_funds(fee).map_err(|err| drv_err!(err))?;
                    fee
                } else {
```

`self.fee_rate`는 `perp_fee_rate`와 같습니다.

**Impact:** 추천 할인이 적용되지 않아 사용자가 추가 자금을 지불해야 하므로 사용자 손실이 발생합니다.


**Recommended Mitigation:** `self.fee_rate`를 부과하는 대신 추천 할인이 적용된 수수료율인 `(1.0 - args.ref_discount) * self.fee_rate`를 부과해야 합니다.



**Deriverse:** 커밋 [91bffc](https://github.com/deriverse/protocol-v1/commit/91bffc86cf2e6e441ca8a526d68808dbc19a122a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.




### Operator precedence Issue during points program expiration validation in `change_points_program_expiration`

**Description:** `change_points_program_expiration`에서 `new_expiration_time`에 대한 유효성 검사는 `!(data.new_expiration_time < root_state.points_program_expiration)` 대신 `!data.new_expiration_time < root_state.points_program_expiration`을 사용하여 의도하지 않은 동작을 유발합니다.

```rust
if !data.new_expiration_time < root_state.points_program_expiration {
    bail!(InvalidNewExpirationTime {
        program_name: "Points Program".to_string(),
        new_time: data.new_expiration_time,
        old_time: root_state.points_program_expiration
    });
}
```
이는 비트 단위 NOT 연산자가 미만 비교 연산자(`<`)보다 우선 순위가 높기 때문에 발생합니다. 이로 인해 표현식은 다음과 같이 평가됩니다:
```rust
(!data.new_expiration_time) < root_state.points_program_expiration
```
의도한 것은 다음과 같습니다:
```rust
!(data.new_expiration_time < root_state.points_program_expiration)
```

**Impact:** `change_points_program_expiration` 함수를 사용하면 포인트 프로그램 만료 시간을 줄일 수 없습니다. `u32`에 `!`를 적용하면 비트 단위 NOT 연산이 수행되기 때문입니다. 이는 `u32::MAX`에 가까운 값을 생성하며, 이는 `points_program_expiration`을 줄이려고 할 때 거의 항상 합리적인 만료 시간보다 큽니다.

**Recommended Mitigation:** 유효성 검사 중에 `!(data.new_expiration_time < root_state.points_program_expiration)`을 사용하십시오. 수정 사항은 다음과 같습니다:
```rust
    if !(data.new_expiration_time < root_state.points_program_expiration){
        bail!(InvalidNewExpirationTime {
            program_name: "Points Program".to_string(),
            new_time: data.new_expiration_time,
            old_time: root_state.points_program_expiration
        });
    }
```

**Deriverse:** 커밋 [eae149](https://github.com/deriverse/protocol-v1/commit/eae1494725cf3ebdc5800fa39bb80b6c90c62478)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.



### Attacker can extract value by buying and selling the seat

**Description:** 사용자가 좌석을 구매할 때마다 통화 토큰으로 `seat_price`를 지불해야 합니다. `seat_price`는 구매 시점의 `perp_clients_count`에 따라 다릅니다. `perp_clients_count`가 높으면 사용자는 더 많은 좌석 비용을 지불합니다. 더 낮은 경우, 좌석 비용은 적습니다.

```rust
    pub fn get_place_buy_price(supply: u32, dec_factor: u32) -> Result<i64, DeriverseError> {
        let df = get_dec_factor(dec_factor);
        Ok(get_reserve(supply + 1, df)? - get_reserve(supply, df)?)
    }
```


공격자는 이 동작을 악용하여 사용자가 훨씬 더 높은 좌석 가격을 지불하게 만드는 시나리오를 만들어 이익을 추출할 수 있습니다. 예를 들어 소수점 6자리의 X 통화를 가정합니다:

1. 초기에 `perp_clients_count`는 199,999입니다.
2. 공격자는 나중에 이익을 얻기 위해 1,000좌석을 구매합니다.
3. 사용자 트랜잭션 동안 `perp_clients_count`는 200,999이고, 사용자는 26,030,289의 좌석 가격을 지불합니다.
5. 공격자는 1,000좌석을 판매하고 1,030,788의 이익을 추출합니다.
6. 공격자가 이 공격을 수행하지 않았다면 사용자는 좌석에 대해 24,999,501만 지불하면 되었습니다.

선행 매매(front-running)는 필요하지 않습니다. 공격자는 좌석을 미리 구매하고 나중에 판매하여 상품을 사용하지 않고 자금을 추출할 수 있습니다.



**Impact:** 공격자는 이 동작을 통해 사용자 자금을 추출하여 사용자가 예상보다 더 많이 지불하게 할 수 있습니다.


**Recommended Mitigation:** 사용자가 원래 지불한 좌석 가격을 저장하고 나중에 사용자가 좌석을 판매할 때 구매 시 지불한 것과 동일한 금액을 반환하는 것이 좋습니다.

**Deriverse:** 커밋 [a80b0e](https://github.com/deriverse/protocol-v1/commit/a80b0ebf90f707d8e527cbcccce0790f35676d13)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Fee Prepayment Locked Due to Asset Record Cleanup after `fees_deposit`

**Description:** 사용자가 `fees_deposit`을 통해 수수료를 선납할 때, 입금으로 인해 crncy_tokens가 0으로 줄어들면(직후 또는 나중에), `clean_and_check_token_records()`에 의해 자산 기록이 지워집니다. 나중에 사용자가 `fees_withdraw`를 통해 출금을 시도할 때 `alloc=false`인 `resolve()` 호출은 자산 기록을 찾지 못하여 `client_community_state`에 선불 금액이 저장되어 있음에도 불구하고 선불 수수료를 출금할 수 없게 됩니다.

문제는 다음 순서로 발생합니다(아래는 예시):
- `fees_deposit`에서: `client_state.sub_crncy_tokens(data.amount)`는 통화 토큰 잔액을 줄입니다.

```rust
    client_community_state.data[crncy_index].fees_prepayment += data.amount;
    client_state.sub_crncy_tokens(data.amount)?;
```

- `fees_deposit`에서: `client_state.clean_and_check_token_records()`가 호출되며, 이는 `value == 0`일 때 자산 기록을 지웁니다:

```rust
    client_state.clean_and_check_token_records()?;
```

```rust
    pub fn clean_and_check_token_records(&mut self) -> DeriverseResult {
        for r in self.assets.iter_mut() {
            if r.asset_id != 0
                && (r.asset_id & 0xFF000000) != AssetType::SpotOrders as u32
                && (r.asset_id & 0xFF000000) != AssetType::Perp as u32
            {
                match r.value.cmp(&0) {
                    std::cmp::Ordering::Equal => r.asset_id = 0,
                    std::cmp::Ordering::Less => bail!(InsufficientFunds),
                    std::cmp::Ordering::Greater => {}
                }
            }
        }
        Ok(())
    }
```

- `fees_withdraw`에서: `client_state.resolve(AssetType::Token, data.token_id, TokenType::Crncy, false)`가 `alloc=false`로 호출됩니다.

```rust
    client_community_state.update(&mut client_state, &mut community_state, None)?;
    client_state.resolve(AssetType::Token, data.token_id, TokenType::Crncy, false)?;
    let dec_factor = get_dec_factor(community_state.base_crncy[crncy_index].decs_count) as f64;
    let amount = (data.amount as f64 / dec_factor) as i64;
```

- `find_or_alloc_asset`에서: `alloc=false`이고 자산 기록을 찾을 수 없으면 `AssetNotFound error:`를 반환합니다.

```rust
        if asset_index == NULL_INDEX {
            realloc = true;
            if !alloc {
                bail!(AssetNotFound {
                    asset_type: asset,
                    id,
                });
            }
```

- `fees_prepayment`가 여전히 `client_community_state.data[crncy_index].fees_prepayment`에 저장되어 있어도 출금은 실패합니다.

근본적인 원인은 `fees_withdraw`가 `add_crncy_tokens()`를 호출하기 전에 `crncy_tokens` 포인터를 설정하기 위해 `resolve()`가 성공해야 하기 때문입니다. 그러나 `alloc=false`인 `resolve()`는 입금 후 지워진 자산 기록을 재생성할 수 없습니다.


**Impact:** crncy_tokens를 0으로 줄이는 수수료 선납을 입금한 사용자(또는 그 이후)는 선불 수수료를 인출할 수 없습니다. `asset`을 생성하려면 다시 입금해야 합니다. 이를 위해서는 추가적인 수동 작업이 필요합니다.

**Recommended Mitigation:** 통화 토큰 자산을 확인할 때 `alloc=true`를 사용하도록 `fees_withdraw`를 변경하십시오:

```rust
client_state.resolve(AssetType::Token, data.token_id, TokenType::Crncy, true)?;
```

**Deriverse:** 커밋 [6662b16](https://github.com/deriverse/protocol-v1/commit/6662b16894ab4462ea9002f355bfd9c9e60784d9)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Decimal Mismatch in Fee Prepayment Accounting Causes Incorrect Balance Tracking

**Description:** 수수료 선납이 저장되고 인출되는 방식 사이에는 심각한 소수점 불일치가 있습니다. `fees_deposit`에서 `fees_prepayment` 필드는 원시 `data.amount` 값(토큰의 기본 소수점 단위)을 저장하지만, `fees_withdraw`에서는 `data.amount / dec_factor`(사람이 읽을 수 있는 단위)를 뺍니다.

이로 인해 출금 후 저장된 선불 잔액이 잘못되는 심각한 회계 오류가 발생하며, 잘못된 오프체인 이벤트 로깅으로 이어집니다.

이 문제는 일관성 없는 단위 처리로 인해 발생합니다:

`fee_deposit`에서:
```rust
let prepayment = data.amount as f64 / dec_factor;
...
client_community_state.data[crncy_index].fees_prepayment += data.amount;  // 원시 값 저장
...
client_state.sub_crncy_tokens(data.amount)?;
```

그러나 `fee_withdraw`에서:

```rust
let amount = (data.amount as f64 / dec_factor) as i64;  // dec_factor로 나눔
client_community_state.data[crncy_index].fees_prepayment -= amount;  // 나눠진 값을 뺌
```

소수점 6자리 예시 (dec_factor = 1,000,000):
- 사용자 1토큰 입금: `fees_prepayment += 1,000,000 (1,000,000으로 저장됨)`
- 사용자 1토큰 출금: `amount = 1,000,000 / 1,000,000 = 1, fees_prepayment -= 1`
- 결과: `fees_prepayment = 0` 대신 `999,999`

추가 문제:
- 이벤트 로깅 불일치: 로그 이벤트는 `data.amount`(원시 값)를 기록하지만 실제 출금 금액은 `data.amount / dec_factor`여서 잘못된 오프체인 회계를 유발합니다.
```rust
    solana_program::log::sol_log_data(&[bytemuck::bytes_of::<FeesWithdrawReport>(
        &FeesWithdrawReport {
            tag: log_type::FEES_WITHDRAW,
            client_id: client_state.id,
            token_id: data.token_id,
            amount: data.amount,
            time: clock.unix_timestamp as u32,
            ..FeesWithdrawReport::zeroed()
        },
    )]);
```

**Workaround: 현재 구현에는 원래 `data.amount`에 `dec_factor`를 곱하는 해결 방법이 있지만 이는 `SPOT_MAX_AMOUNT` 제한에 대해 검증되어야 합니다.**

**Impact:**
- 오프체인 회계 오류: 이벤트 로그에 잘못된 금액이 표시되어 오프체인 시스템이 잘못된 값을 추적하게 합니다.
- 원래 출금이 작동하지 않음: 해결 방법이 적용되지 않는 한 원래 `fees_withdraw`는 올바르게 작동하지 않습니다.

**Recommended Mitigation:** `data.amount`를 일관되게 사용하십시오:

```rust
   client_community_state.data[crncy_index].fees_prepayment -= data.amount;  // 원시 값 사용
   ...
   client_state.add_crncy_tokens(data.amount)?;  // 원시 값 사용
```

**Deriverse:** 커밋 [1dcab9d](https://github.com/deriverse/protocol-v1/commit/1dcab9df3dc962b79c7a6458da810c36d3397f3e)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Users can sell their market seat without paying loss coverage

**Description:** 사용자가 손실을 입으면 사용 가능한 보험 기금에서 충당되며 `taker_info4.loss_coverage` 내부에 저장되고 추적됩니다. 이는 보험 기금이 사용자에게 발생한 이러한 손실을 메우기 위해 사용되었기 때문에 사용자가 포지션을 닫기 전에 프로토콜에 빚진 것입니다. `sell-market-seat` 내부에서 [close_account](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/processor/sell_market_seat.rs#L127)를 호출하는데, 사용자의 손실 커버리지를 확인하지 않고 다음만 확인합니다:
```rust
            if (*self.perp_info3).bids_entry != NULL_ORDER
                || (*self.perp_info3).asks_entry != NULL_ORDER
                || self.current_instr_index >= self.assets.len()
            {
                bail!(ClientDataDestruction);
            }
```
사용자가 과거에 손실을 입었고 보험 기금이 이 손실을 보상한 경우 좌석을 닫기 전에 이 자금을 갚아야 합니다. [`try_to_close_perp`](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/state/client_primary.rs#L701) 함수는 이를 올바르게 확인합니다. 사용자에게 대기 중인 손실 커버리지가 있는 경우 사용자 자금에서 이를 차감합니다. 이러한 방식으로 손실을 갚고 포지션을 닫습니다.
```rust
            if (*self.perp_info4).loss_coverage > 0 {
                let delta = (*self.perp_info4)
                    .loss_coverage
                    .min((*self.perp_info).funds);
                if delta > 0 {
                    engine.state.header.perp_insurance_fund += delta;
                    (*self.perp_info4).loss_coverage -= delta;
                    (*self.perp_info).funds -= delta;
                }
            }
```

**Impact:** 사용자는 발생한 손실을 지불하지 않고 좌석을 닫을 수 있습니다.

**Recommended Mitigation:** `sell-market-seat` 내부에서 `close_perp` 대신 `try_to_close_perp`를 호출하십시오.

**Deriverse:** 커밋에서 수정됨: [e5af702](https://github.com/deriverse/protocol-v1/commit/e5af70204e812ed9f388a0dea23ff89fd15f2394)

**Cyfrin:** 검증되었습니다.


### Users are getting back their `soc-loss-funds` while selling their market seat

**Description:** `sell-market-seat` 내부에서 [soc-loss-funds](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/processor/sell_market_seat.rs#L122)가 음수일 때, 이는 사용자가 프로토콜의 soc loss 자금에 추가하고 기여했으며 자신의 편에 있는 다른 사람들을 대신하여 손실을 감수했음을 의미합니다... 이 경우 사용자는 해당 자금을 돌려받고 있으며, 이 자금은 보험 기금에서도 차감되고 있습니다. 일반적인 아이디어는 사용자가 soc loss에 기여한 것을 돌려주지 않는 것이지만, 여기서는 사용자가 돌려받고 있습니다.
```rust
    let collactable_losses = info.funds().min(client_state.perp_info4()?.soc_loss_funds);
    engine.state.header.perp_insurance_fund += collactable_losses;

    client_state.add_crncy_tokens((info.funds() - collactable_losses).max(0))?;
```
예:
- 사용자가 일부 손실을 입었고 그의 soc loss는 +20으로 업데이트됩니다.
- [check-soc-loss](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/perp/perp_engine.rs#L2216)에 대한 다음 호출에서 그는 +30을 갚고, 사용자의 순(net) soc loss는 -10이 됩니다. 이는 사용자가 프로토콜의 soc loss에 기여했음을 의미하며 이 금액이 `self.state.header.perp_soc_loss_funds`에도 추가되는 것을 볼 수 있습니다.
- 이제 사용자가 `soc-loss-funds`가 -10으로 설정된 상태에서 마켓 좌석을 판매하려고 합니다..... 그는해서는 안 되지만 이 금액을 [돌려받습니다](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/processor/sell_market_seat.rs#L125).

**Impact:** 사용자는 soc loss에 지불한 것을 돌려받습니다.

**Recommended Mitigation:** 사용자의 `soc-loss-funds`가 음수로 나오는 경우 이를 회계 처리하지 마십시오.

**Deriverse:** 커밋에서 수정됨: [e5af702](https://github.com/deriverse/protocol-v1/commit/e5af70204e812ed9f388a0dea23ff89fd15f2394)

**Cyfrin:** 검증되었습니다.


### Users can provide old price feeds to trade in their favor

**Description:** [set_underlying_px](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/state/instrument.rs#L224) 함수는 코드의 많은 곳에서 호출되며 오라클이 설정된 경우 사용자가 제공한 오라클 피드에서 기초 가격을 설정합니다. `set-underlying-px`는 사용자가 제공한 피드에 의존하지만 가격 게시 시간이나 신뢰 구간(confidence interval)을 확인하지 않습니다. 몇 시간 된 가격(또는 의도적으로 동결된 가격)도 최신 가격으로 허용됩니다. 오라클 데이터가 최신인지, 오라클 슬롯/타임스탬프가 최근인지, 오라클 신뢰도가 좋은지 확인하지 않으며, 피드 계정 주소가 예상과 일치하는지만 확인합니다.
```rust
            let feed_id = next_account_info!(accounts_iter)?;
            if self.header.feed_id == *feed_id.key {
                let oracle_px =
                    i64::from_le_bytes(feed_id.data.borrow()[73..81].try_into().unwrap());
                let oracle_exponent =
                    9 + i32::from_le_bytes(feed_id.data.borrow()[89..93].try_into().unwrap());
                let mut dc: i64 = 1;
```
계정이 올바른 피드 ID이더라도 내부 데이터는 오래된 것일 수 있습니다. Pyth 형식과 같이 타임스탬프, 슬롯, conf 등이 있는 피드 필드를 확인하지 않기 때문입니다. 이는 피드가 얼마나 최신인지 알려줍니다.

**Impact:** 거래는 오래된 데이터에서 계산되어 공격자가 과다 지불하거나 과소 지불하게 할 수 있습니다.

**Recommended Mitigation:** 최신인지 확인하기 위해 피드의 타임스탬프를 확인하십시오. 특정 분까지의 오래된 가격을 허용하고 간격이 허용된 것보다 크면 수락하지 마십시오.

**Deriverse**
우리는 오라클 지원을 제외합니다.

**Cyfrin:** 검증되었습니다.


### Users incur losses when selling seats

**Description:** 사용자가 좌석을 매수하거나 매도할 때마다, 우리는 새/현재 포지션과 이전 포지션의 `get_reserve` 차이를 사용하여 그 순간의 좌석 가격을 계산합니다.
```rust
    pub fn get_place_buy_price(supply: u32, dec_factor: u32) -> Result<i64, DeriverseError> {
        let df = get_dec_factor(dec_factor);
        Ok(get_reserve(supply + 1, df)? - get_reserve(supply, df)?)
    }

    pub fn get_place_sell_price(supply: u32, dec_factor: u32) -> Result<i64, DeriverseError> {
        let df = get_dec_factor(dec_factor);
        Ok(get_reserve(supply, df)? - get_reserve(supply - 1, df)?)
    }
```
`get_reserve` 함수는 최대값이 9223372036854775807인 i64를 반환합니다. 그리고 `get_reserve` 계산은 f64로 수행되며 f64의 결과가 이 제한을 초과하면 i64로 다시 캐스팅할 때 값이 최대 i64 값(9223372036854775807)으로 고정(clamp)됩니다.

Example: 249,995에서 250,000까지의 좌석 가격은 `crncy_token_decs_count`가 9일 때 값이 0이 됩니다.


**Impact:** 이는 예기치 않은 동작으로 이어져 일반 사용자에게 손실을 초래할 수 있으며 `perp_clients_count`가 250000에 가까우면 공격자가 가치를 추출할 수 있는 기회를 만듭니다.



**Recommended Mitigation:** `get_reserve`는 f64를 반환해야 합니다. 두 `get_reserve` 호출의 결과 간의 차이를 계산한 후 최종 값을 다시 i64로 변환하여 더 정확한 결과를 얻을 수 있습니다.

**Deriverse:** 커밋 [c8d26d](https://github.com/deriverse/protocol-v1/commit/c8d26d57c1d9f24add2ed449f422d79ec983c13a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.



### `perp_statistics_reset` can be used by users to skip `collactable-losses` while selling market seat

**Description:** `perp-statistics-reset` 내부에서 우리는 사용자가 손실을 입었는지 확인하지 않고 [soc loss funds](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/processor/perp_statistics_reset.rs#L91)를 0으로 설정합니다.
```rust
// perp-statistics-reset 내부
    client_state.perp_info4()?.soc_loss_funds = 0;
```
사용자가 손실을 입었다면 이 자금은 양수일 것이며, 이는 사용자가 마켓 좌석을 판매할 때 프로토콜이 수집하여 보험 기금에 추가해야 하는 자금 금액임을 의미합니다. 우리는 이를 0으로 설정하고 나중에 `check-soc-loss`를 호출하여 이 자금을 음수로 만듭니다. 이제 사용자가 `sell-market-seat`를 호출하면 이 자금을 돌려받고 [보험 기금](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/processor/sell_market_seat.rs#L123)도 이 금액만큼 줄어들어 잘못됩니다.
```rust
// sell-market-seat 내부
    let collactable_losses = info.funds().min(client_state.perp_info4()?.soc_loss_funds);
    engine.state.header.perp_insurance_fund += collactable_losses;

    client_state.add_crncy_tokens((info.funds() - collactable_losses).max(0))?;

```
사용자가 이전에 손실을 입었고 그의 `soc-loss-funds`가 양수라면 `perp-statistics-reset`을 사용하도록 허용해서는 안 됩니다. 그는 단순히 "포지션 종료 시 프로토콜에 빚진 금액"을 지우는 데 사용할 수 있습니다.

**Impact:** 사용자는 이를 악용하여 프로토콜에 빚진 금액을 저장하는 soc loss 기록을 지울 수 있습니다.

**Recommended Mitigation:** 사용자의 soc loss를 가져오고, 그것이 양수이면(사용자가 프로토콜에 금액을 빚지고 있음을 의미함), 통계 재설정을 피하고 되돌리(revert)십시오.

**Deriverse:** 커밋에서 수정됨: [e5af702](https://github.com/deriverse/protocol-v1/commit/e5af70204e812ed9f388a0dea23ff89fd15f2394)

**Cyfrin:** 검증되었습니다.


### Missing Array Synchronization in `dividends_claim` Prevents Users from Claiming Dividends for Newly Added Base Currencies

**Description:** `dividends_claim` 명령은 배당금을 처리하기 전에 `client_community_state.data`를 `community_state.base_crncy`와 동기화하지 않습니다. 새 기본 통화가 커뮤니티에 추가되고 사용자가 동기화를 트리거하는 작업(`deposit` 또는 `trading` 등)을 수행하지 않은 경우, 사용자의 `client_community_state.data` 배열은 community_state.base_crncy보다 짧습니다. 이로 인해 zip 반복자가 조기에 중지되어 사용자가 동기화를 트리거하는 다른 작업을 수행할 때까지 새로 추가된 기본 통화에 대한 배당금을 청구할 수 없습니다.

`src/state/client_community.rs`에서 업데이트 함수는 처리 전에 배열을 동기화합니다:

```rust
pub fn update(
    &mut self,
    client_primary_state: &mut ClientPrimaryState<'a, 'info>,
    community_state: &mut CommunityState,
    payer: Option<&'a AccountInfo<'info>>,
) -> DeriverseResult {
    self.update_records(community_state, payer)?;  // 배열 동기화
    // ... 나머지 로직
}
```

update_records 함수는 `client_community_state.data`가 `community_state.base_crncy`와 일치하는지 확인합니다:

```rust
fn update_records(
    &mut self,
    community_state: &CommunityState,
    payer: Option<&'a AccountInfo<'info>>,
) -> DeriverseResult {
    // ... 필요한 경우 재할당 ...
    self.header.count = community_state.header.count;
    // ... 새 데이터 배열 생성 ...
    for (d, b) in self.data.iter_mut().zip(community_state.base_crncy.iter()) {
        d.crncy_token_id = b.crncy_token_id;
        d.fees_ratio = 1.0;
    }
}
```

그러나 `src/program/processor/dividends_claim.rs`에서 명령은 동기화 없이 직접 반복합니다:

```rust
for (d, b) in client_community_state
    .data
    .iter_mut()
    .zip(community_state.base_crncy.iter_mut())
{
    // 배당금 처리...
}
```

- `deposit`과 같은 다른 명령은 `client_community_state.update()`를 호출하여 `update_records`를 트리거합니다.
- `dividends_claim`은 처리 전에 `update_records`나 업데이트를 호출하지 않습니다.
- `client_community_state.data.len() < community_state.base_crncy.len()`인 경우 더 짧은 배열이 끝나면 zip 반복자가 중지됩니다.
- 사용자는 다른 작업을 수행할 때까지 새로 추가된 기본 통화에 대한 배당금을 청구할 수 없습니다.

**Impact:** 새 기본 통화가 추가된 후 작업을 수행하지 않은 사용자는 해당 통화에 대한 배당금을 청구할 수 없습니다. 사용자는 배당금을 청구하기 전에 동기화를 트리거하기 위해 추가 작업(예: 입금)을 수행해야 합니다.

**Recommended Mitigation:** dividends_claim에서 배당금을 처리하기 전에 동기화를 추가하십시오.

**Deriverse:** 커밋 [5f5460](https://github.com/deriverse/protocol-v1/commit/5f5460d75277aa1c946501680577adc33382b7b5)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `min_qty` Bypass via `IOC` limit order

**Description:** 현물 주문은 사용자 제공 지정가에서 최소 주문 수량 `min_qty`를 계산합니다. 공격자는 임의로 큰 지정가를 설정하여 `min_qty`를 아주 작은 값으로 낮추면서 실제 실행 가격은 여전히 시장 가격 근처로 고정시킬 수 있습니다. 이는 IOC 주문에 대한 의도된 최소 주문 크기 강제 집행을 우회합니다.

`SpotParams::get_settings`는 `ipx` 변수를 통해 IOC 가격에 대해 ±12.5% 고정(clamp)을 시행하지만, 최소 토큰은 `IOC` 모드가 활성화되어 있어도 원래 `px` 인수(사용자가 제공한 가격)에서 계산됩니다:

```rust
        if ioc != 0 {
            if px < last_px - max_diff {
                ipx = last_px - max_diff;
            } else if px > last_px + max_diff {
                ipx = last_px + max_diff;
            } else {
                ipx = px;
            }
        } else if px < last_px - max_diff || px > last_px + max_diff {
            bail!(DeriverseErrorKind::InvalidPrice {
                price: px,
                min_price: last_px - max_diff,
                max_price: last_px + max_diff,
            });
        } else {
            ipx = px;
        }
...
        let min_tokens = min_qty(px, dc);
        // TODO 제거
        if qty != 0 && qty.abs() < min_tokens {
            bail!(DeriverseErrorKind::InvalidQuantity {
                value: qty,
                min_value: min_tokens,
                max_value: SPOT_MAX_AMOUNT,
            });
        }
```

`min_qty`가 제한 없는 px를 사용하기 때문에, **매수 주문의 경우** 공격자는 `LIMIT order`로 `ioc != 0`을 보내고 엄청난 `data.price`를 선택할 수 있습니다. `min_qty`는 `PRICE_TOKENS`에서 매우 작은 임계값을 가져와 공격자가 먼지(dust) 크기의 주문을 제출할 수 있게 합니다. 엔진은 나중에 실행 가능한 가격을 시장의 ±12.5% 이내로 고정하므로 거래는 여전히 정상 시장 가격으로 진행되지만 금액은 의도한 최소값보다 훨씬 낮습니다.

예를 들어 현재 가격이 `500_000_000`인 경우, 공격자/사용자는 `px=100_000_000_000_000`을 만들어 `20_000`만큼 적은 금액을 지불하여 먼지 주문을 만들 수 있습니다.

**Impact:** `IOC` 주문에 대한 최소 주문 크기 제한을 우회할 수 있습니다. 공격자는 큰 `data.price`를 제공하여 대량의 먼지 거래를 생성할 수 있습니다.

**Recommended Mitigation:** 현재 시장 가격을 사용하여 `min_qty`를 계산하는 것을 고려하십시오.

**Deriverse:** 커밋 [134db8b](https://github.com/deriverse/protocol-v1/commit/134db8b1dac9e48641dac1a6b95bcf34637f695f)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Wrong accounting of fee in perp engine

**Description:** [perp_engine.rs](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/perp/perp_engine.rs#L1202) 내의 perp 수수료 계산 로직에서, 사용자가 거래 수수료를 완전히 커버하지 못하는 부분적인 수수료 선납을 가지고 있을 때, 함수는 `fees_prepayment + extra_fee` 대신 [discount_sum + extra_fee](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/perp/perp_engine.rs#L1202)를 잘못 반환합니다.

버그는 [discount_sum](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/perp/perp_engine.rs#L1198)이 수수료 금액 자체가 아니라 선납으로 커버된 `거래 가치`([fees_prepayment / fee_rate]로 계산됨)를 나타내기 때문에 발생합니다. 이로 인해 수수료 반환 값이 엄청나게 부풀려집니다.

```rust
                    } else {
                        let discount_sum = (*fees_prepayment as f64 / args.fee_rate) as i64;
                        let extra_fee = ((traded_crncy - discount_sum) as f64
                            * (1.0 - args.ref_discount)
                            * self.fee_rate) as i64;
                        let fee = discount_sum + extra_fee; // 여기, discount_sum은 수수료가 아니라 거래 가치입니다.
                        taker_info
                            .sub_funds(extra_fee)
                            .map_err(|err| drv_err!(err))?;
                        *fees_prepayment = 0;
                        fee
                    }
```
다음 시나리오를 고려하십시오:
입력:
- traded_crncy = $10,000 (거래 가치)
- self.fee_rate = 0.001 (0.1% 기본 수수료)
- ref_discount = 0.20 (20% 추천 할인)
- args.fee_rate = 0.001 * 0.80 = 0.0008 (할인된 요율)
- fees_prepayment = $5

STEP 1: `discount_fee` 계산
discount_fee = 0.0008 * $10,000 = $8
$8 <= $5인가? 아니요는 부분 적용 분기로 진입함을 의미합니다.

STEP 2: `discount_sum` 계산
discount_sum = $5 / 0.0008 = $6,250 (이는 선납으로 커버되는 거래 가치입니다)

STEP 3: `extra_fee` 계산
remaining_trade = $10,000 - $6,250 = $3,750
extra_fee = $3,750 * 0.80 * 0.001 = $3

반환값 (잘못됨):
fee = discount_sum + extra_fee = $6,250 + $3 = $6,253 : 이는 잘못되었으며, 부풀려졌습니다.

올바른 반환값:
fee = fees_prepayment + extra_fee = $5 + $3 = $8 ← 올바름

사용자가 실제로 지불하는 금액:
- 소비된 선납금: $5
- 자금에서 extra_fee: $3
- 총 지불액: $8 (올바른 금액 차감됨)

그러나 프로토콜 기록: 수수료로 $6,253을 기록하며 이는 너무 많이 부풀려져 완전히 부정확합니다.

**Impact:** 프로토콜은 부풀려진 수수료를 기록하여 나중에 이에 의존하는 나머지 회계를 손상시킬 것입니다.

**Recommended Mitigation:** fill() 함수에서 다음과 같이 변경하십시오:
```rust
} else {
    let discount_sum = (*fees_prepayment as f64 / args.fee_rate) as i64;
    let extra_fee = ((traded_crncy - discount_sum) as f64
        * (1.0 - args.ref_discount)
        * self.fee_rate) as i64;
    // 수정: discount_sum(거래 가치)이 아닌 fees_prepayment(실제 수수료 금액) 사용
    let fee = (*fees_prepayment) + extra_fee;
    taker_info
        .sub_funds(extra_fee)
        .map_err(|err| drv_err!(err))?;
    *fees_prepayment = 0;
    fee
}
```
**Deriverse**
커밋에서 수정됨: [94da2f1](https://github.com/deriverse/protocol-v1/commit/94da2f1ed0a85b0a8f247bbd5da9cd7dc04f82ba)

**Cyfrin:** 검증되었습니다.


### User's chosen leverage is overwritten to `max-leverage` on every perp operation

**Description:** `client_primary.rs`에서 [new_for_perp](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/state/client_primary.rs#L344) 함수는 `alloc` 매개변수가 참이든 거짓이든 관계없이 모든 호출에서 무조건 사용자의 레버리지를 설정합니다.
```rust
    let mut client_state = ClientPrimaryState::new_for_perp(
        program_id,
        client_primary_acc,
        &ctx,
        signer,
        system_program,
        0, //항상 0으로 전달됨
        false,
        true,
    )?;
```
[leverage = 0](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/state/client_primary.rs#L470)이 전달되면(`perp_withdraw`, `perp_deposit`, `perp_mass_cancel`, `perp_order_cancel`, `perp_quotes_replace`, `perp_statistics_reset`, `buy_market_seat`, `sell_market_seat`와 같은 대부분의 perp 작업에서 발생), 코드는 사용자의 레버리지를 `max_leverage`로 설정합니다.

```rust
if leverage > 0 {
    //지우고 설정
    unsafe {
        (*state.perp_info2).mask &= 0xFFFFFF00;
        (*state.perp_info2).mask |= (leverage as u32).min(header.max_leverage as u32);
    }
} else {
    // 레버리지 == 0일 때, max_leverage로 설정 (항상 기본값)
    unsafe {
        (*state.perp_info2).mask &= 0xFFFFFF00;
        (*state.perp_info2).mask |= header.max_leverage as u32;
    }
}
Ok(state)
```
다음 시나리오를 고려하십시오:
1. 사용자가 perp_change_leverage(5)를 호출하여 레버리지를 5x로 설정
   - 레버리지가 5로 저장됨

2. 사용자가 증거금을 더 추가하기 위해 `perp_deposit()` 호출
   - leverage=0으로 `new_for_perp()` 호출됨
   - 레버리지가 `max_leverage`(예: 15x)로 재설정됨

3. 사용자는 이제 5x 대신 15x 레버리지를 가짐

**Impact:** 레버리지를 보수적인 값(예: 2x 또는 3x)으로 신중하게 설정한 사용자는 perp 작업을 수행한 후 `max_leverage`로 조용히 재설정됩니다. 이는 사용자가 모르는 사이에 청산 위험을 크게 증가시킵니다.

**Recommended Mitigation:** 명시적으로 요청된 경우에만 레버리지를 업데이트하십시오(즉, `leverage > 0`일 때). 레버리지를 `max_leverage`로 덮어쓰는 else 분기를 제거하십시오:
```rust
if leverage > 0 {
    // 명시적으로 제공된 경우에만 레버리지 업데이트
    unsafe {
        (*state.perp_info2).mask &= 0xFFFFFF00;
        (*state.perp_info2).mask |= (leverage as u32).min(header.max_leverage as u32);
    }
}
// 레버리지 == 0일 때, 기존 레버리지 유지
```
**Deriverse:** 커밋에서 수정됨: https://github.com/deriverse/protocol-v1/commit/e15ad9372bf10322c6d05c234189576b8aad3690

**Cyfrin:** 검증되었습니다.


### Redundant State Updates in `fill` Function Cause Issues

**Description:** `fill` 함수에는 `write_last_tokens`에 의해 이미 적절하게 처리된 `last_asset_tokens` 및 `last_crncy_tokens`에 대한 **중복 상태 업데이트**가 포함되어 있습니다. 이러한 중복성은 동일한 슬롯 내에서 여러 주문이 체결될 때 데이터 손실, 잘못된 누적 및 상태 불일치를 유발합니다.

- 중복 업데이트
`fill` 함수에서 `last_asset_tokens` 및 `last_crncy_tokens`가 중복 업데이트됩니다:
```rust
self.state.header.last_asset_tokens = traded_qty;
self.state.header.last_crncy_tokens = traded_crncy;
```
`match_orders`가 완료된 후 `write_last_tokens`는 누적된 값으로 이미 호출됩니다:

```rust
engine.write_last_tokens(traded_qty, traded_sum, trades, px)?;
```

`write_last_tokens` 함수는 다음을 사용하여 이러한 상태 업데이트를 적절하게 처리합니다:
- 슬롯 경계 확인
- 슬롯이 일치할 때 누적 로직
- 슬롯이 다를 때 재설정 로직

```rust
if self.state.header.slot == self.slot {
    self.state.header.last_crncy_tokens = self
        .state
        .header
        .last_crncy_tokens
        .checked_add(traded_crncy_tokens)
        .ok_or(drv_err!(DeriverseErrorKind::ArithmeticOverflow))?;
    self.state.header.last_asset_tokens = self
        .state
        .header
        .last_asset_tokens
        .checked_add(traded_asset_tokens)
        .ok_or(drv_err!(DeriverseErrorKind::ArithmeticOverflow))?;
} else {
    self.state.header.slot = self.slot;
    self.state.header.last_crncy_tokens = traded_crncy_tokens;
    self.state.header.last_asset_tokens = traded_asset_tokens;
}
```

중복성으로 인해 발생하는 문제
- fill 루프 내 데이터 손실: `fill` 함수의 루프 내에서 각 반복은 이전 값을 덮어쓰고 마지막 주문의 값만 보존합니다:
```
   while !order.is_null() && *remaining_qty > 0 {
       // ... 주문 처리 ...
       self.state.header.last_asset_tokens = traded_qty;  // 이전 값을 덮어씁니다!
       self.state.header.last_crncy_tokens = traded_crncy; // 이전 값을 덮어씁니다!
   }
```
- 다중 fill 호출: `match_orders` 함수는 `fill`을 여러 번 호출할 수 있으며, 각각 부분 데이터로만 상태를 덮어씁니다.

- 잘못된 누적: `write_last_tokens`가 subsequently 호출될 때. 슬롯이 일치하면 총 누적 값을 `fill`에서 잘못 설정된 값에 추가합니다. 이는 이중 계산 또는 잘못된 합계를 유발합니다.

예: 이전에 `last_asset_tokens = 200`인 경우, `fill`이 `last_asset_tokens = 100`(마지막 주문만)으로 설정하면, `write_last_tokens`는 총 `traded_qty = 500`을 추가하여 `200 + 500` 대신 `600`이 됩니다.

**Impact:** 잘못된 상태: `write_last_tokens`가 값을 누적할 때 잘못 설정된 값에 추가되어 잘못된 합계가 발생합니다.

**Recommended Mitigation:** `fill` 함수에서 중복 상태 업데이트를 제거하십시오.

**Deriverse:** 커밋 [0be264f1](https://github.com/deriverse/protocol-v1/commit/0be264f1a0a727aa525ddab8d29c2a74c83294d7)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Missing Trade Count Updates in `reversed_swap` Function for Buy Orders

**Description:** `swap` 작업에서 `buy` 주문에 사용되는 `reversed_swap` 함수는 `trade` 개수를 제대로 추적하고 반환하지 못하여 상태에 잘못된 거래 통계가 기록됩니다. `match_orders`와의 이러한 불일치로 인해 매수 주문 거래가 0으로 기록되어 거래량 및 거래 횟수 지표가 부정확해집니다.

`swap` 흐름에서 매수 주문은 `reversed_swap`을 사용하는 반면 매도 주문은 `match_orders`를 사용합니다.

```rust
    if buy {
        if price > px || engine.cross(price, OrderSide::Ask) {
            let total_fees;
            let remaining_sum;
            let input_sum = (data.amount as f64 / (1.0 + fee_rate)) as i64;

            (remaining_sum, traded_qty, total_fees) = engine.reversed_swap(price, input_sum)?;

```

2가지 문제가 있습니다:

Issue 1: 누락된 `trades` 반환 값

`reversed_swap` 함수는 내부적으로 거래를 추적하지만 이 값을 반환하지 않습니다:

```rust
            (remaining_sum, traded_qty, total_fees) = engine.reversed_swap(price, input_sum)?;
```

`trades` 변수를 내부적으로 유지 관리하더라도 `(remaining_sum, qty, total_fees)`를 반환하지만 `trades`는 반환하지 않습니다.

따라서 `trades` 변수는 `0`으로 유지되며, 이 잘못된 값이 `write_last_tokens`에 전달됩니다:

```rust
    if traded_qty > 0 && traded_sum > 0 {
        let candles = uninit_candles.init(&engine)?;
        engine.write_candles(candles, traded_qty, traded_sum)?;
        engine.write_last_tokens(traded_qty, traded_sum, trades, px)?;
    } else {
        bail!(DeriverseErrorKind::FailedToSwap {
            price,
            side: if buy { OrderSide::Ask } else { OrderSide::Bid }
        });
    }
```

Issue 2: AMM 거래에 대한 거래 횟수 증가 누락
match_orders와 달리 reversed_swap은 AMM 거래 후 `trades`를 증가시키지 않습니다. `match_orders`에서는 모든 AMM 거래가 카운터를 증가시킵니다:

```rust
self.change_tokens(traded_qty, side)?;
self.change_mints(traded_mints, side)?;
self.log_amm(traded_qty, traded_mints, side);
self.set_px();
trades += 1;
```

그러나 `reversed_swap`에서는 `trades` 증가 없이 AMM 거래가 발생합니다:

```rust
self.change_tokens(traded_qty, side)?;
self.change_mints(traded_mints, side)?;
self.log_amm(traded_qty, traded_mints, side);
self.set_px();
total_fees = total_fees
    .checked_add((traded_mints as f64 * self.fee_rate) as i64)
    .ok_or(drv_err!(DeriverseErrorKind::ArithmeticOverflow))?;
break;
```

AMM 거래가 `trades` 증가 없이 실행되는 유사한 누락이 발생합니다.



**Impact:**
- 잘못된 거래 통계: 매수 주문 거래가 0으로 기록됩니다.
- 데이터 불일치: 매수 및 매도 주문이 다르게 처리되어 비대칭 보고가 발생합니다.

**Recommended Mitigation:** `reversed_swap`이 `trades`를 반환하도록 수정하고 `match_orders`와 일치하도록 각 AMM 거래 후 `trades`를 증가시키십시오.

**Deriverse:** **Cyfrin:**


### Missing `change_funding_rate` Call After Price Update in `perp_mass_cancel` and `perp_order_cancel`

**Description:** `perp_order_cancel` 및 `perp_mass_cancel` 함수에서 코드는 `set_underlying_px`를 통해 기초 가격(`perp_underlying_px`)을 업데이트하지만 `check_rebalancing`(및 잠재적으로 `mass_cancel`)을 호출하기 전에 `change_funding_rate`를 호출하지 못합니다. `check_rebalancing`은 내부적으로 `check_funding_rate`를 호출합니다. 이로 인해 펀딩 비율 계산이 새로운 기초 가격을 반영하도록 업데이트되지 않은(가격이 올바르게 업데이트되었음에도 불구하고) 오래된 글로벌 펀딩 비율 값으로 수행되어 잠재적으로 사용자에게 잘못된 펀딩 수수료 계산이 발생할 수 있습니다.

펀딩 비율 메커니즘은 다음과 같이 작동합니다:
- `change_funding_rate`는 글로벌 펀딩 비율 상태를 업데이트합니다.
```rust
    pub fn change_funding_rate(&mut self) {
        let time_delta = self.time - self.state.header.perp_funding_rate_time;
        if time_delta > 0 && self.state.header.perp_price_delta != 0.0 {
            self.state.header.perp_funding_rate +=
                ((time_delta as f64) / DAY as f64) * self.state.header.perp_price_delta;
        }
        self.state.header.perp_price_delta =
            (self.market_px() - self.state.header.perp_underlying_px) as f64 * self.rdf;
        self.state.header.perp_funding_rate_time = self.time;
    }
    /*
```
- `check_funding_rate`는 개별 클라이언트에게 글로벌 펀딩 비율을 적용합니다:
```rust
    pub fn check_funding_rate(&mut self, temp_client_id: ClientId) -> Result<bool, DeriverseError> {
        let info = unsafe { &mut *(self.client_infos.offset(*temp_client_id as isize)) };
        let info5 = unsafe { &mut *(self.client_infos5.offset(*temp_client_id as isize)) };
        let perps = info.total_perps();
        let mut change = false;
        if perps != 0 {
            if self.state.header.perp_funding_rate != info5.last_funding_rate {
                let funding_funds = -(perps as f64
                    * (self.state.header.perp_funding_rate - info5.last_funding_rate))
                    .round() as i64;
```

따라서 `perp_funding_rate`는 `check_funding_rate`가 호출되기 전에 매번 새로 고쳐져야 합니다.

문제는 `perp_order_cancel` 및 `perp_mass_cancel`에 있습니다:

```rust
// perp_order_cancel
engine.state.set_underlying_px(accounts_iter)?;
// ... 주문 취소 로직 ...
engine.check_rebalancing()?;  // 내부적으로 check_funding_rate() 호출

// perp_mass_cancel
engine.state.set_underlying_px(accounts_iter)?;
engine.mass_cancel(client_state.temp_client_id)?;  // 라인 1888에서 check_funding_rate() 호출
// ... 마진 콜 확인 ...
engine.check_rebalancing()?;  // 또한 라인 2363에서 check_funding_rate() 호출
```

`perp_underlying_px`가 업데이트되지만 `change_funding_rate`가 호출되지 않으면 글로벌 `perp_funding_rate`가 최신 가격 변경을 반영하지 않을 수 있습니다.

**Impact:** `change_funding_rate`가 호출되지 않았기 때문에 펀딩 비율 계산이 업데이트된 기초 가격을 반영하지 않는 오래된 글로벌 펀딩 비율 값을 사용하므로, 주문 취소 시 사용자에게 잘못된 펀딩 수수료가 부과될 수 있습니다.

**Recommended Mitigation:** 두 취약한 함수 모두에서 `set_underlying_px` 직후에 `change_funding_rate` 호출을 추가하십시오.

**Deriverse:** 커밋 [74f9650](https://github.com/deriverse/protocol-v1/commit/74f9650ef966b422d95a384bb1e39f4f7bd9cf22)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Inefficient rebalancing can cause the loss of users

**Description:** 리밸런싱은 5분마다 한 번만 호출할 수 있습니다. 이는 `check_rebalancing` 함수를 통해 시행되며, 실행될 때마다 25개의 리밸런싱 호출만 허용합니다. 이 제한은 심각한 문제를 일으킬 수 있습니다.

예를 들어 오픈 포지션이 200,000개이고 사용자가 perp 관련 작업을 수행할 때마다 `check_rebalancing`이 트리거되는 경우, 모든 포지션을 리밸런싱하려면 5분마다 약 8,000건의 사용자 트리거 호출이 필요합니다. 사용자 작업 수가 더 적으면 많은 포지션이 제때 리밸런싱되지 않습니다.

이로 인해 포지션이 너무 늦게 리밸런싱되어 부적절한 청산(조기 또는 지연 청산)으로 이어질 수 있으며 잠재적으로 다른 사용자의 사회화된 손실(socialized losses)을 증가시킬 수 있습니다.

**Impact:** 이 제한으로 인해 perp 관련 거래가 매우 적은 경우 포지션이 훨씬 나중까지 리밸런싱되지 않아 부적절한 청산이 발생할 수 있습니다.

**Recommended Mitigation:** 5분마다 사용자 포지션을 리밸런싱하기 위해 호출할 수 있는 함수를 구현하십시오.

**Deriverse:** 커밋 [7873db](https://github.com/deriverse/protocol-v1/commit/7873db50be8fc7298817e097d0f55bf165ceae04), [76dbbe](https://github.com/deriverse/protocol-v1/commit/76dbbec20fdd2e09fa5722e993da897bf0da2f8b)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.




### Incorrect Margin Call State Detection After Liquidation in `perp_withdraw`

**Description:** perp_withdraw에서 청산 확인을 실행한 후 마진 콜 상태가 잘못 결정됩니다. 함수는 먼저 `check_long_margin_call()` 및 `check_short_margin_call()`을 호출하여 포지션을 성공적으로 청산할 수 있지만, 그 다음 `is_long_margin_call()` 및 `is_short_margin_call()`을 사용하여 `margin_call` 상태를 확인합니다. 이러한 확인 함수는 트리에서 현재 가장 위험도가 높은 노드를 검사하므로(청산 후 변경되었을 수 있음), 청산이 막 발생했을 때에도 `false`를 반환하여 잘못된 출금 금액 계산으로 이어질 수 있습니다.

이 문제는 `src/program/processor/perp_withdraw.rs`에서 발생합니다:

```rust
engine.check_long_margin_call()?;
engine.check_short_margin_call()?;
let margin_call = engine.is_long_margin_call() || engine.is_long_margin_call();
```

문제점:
- 청산 실행: `check_long_margin_call`은 `long_px` 트리를 반복하고 위험도가 높은 포지션(`max_node`)을 찾아 청산합니다. 포지션이 성공적으로 청산되면 `change_edge_px(temp_client_id)`를 호출하여 트리에서 노드를 제거하거나 업데이트합니다.

```rust
                    let (_, t, _) = self.match_bid_orders(
                        None,
                        &MatchOrdersStaticArgs {
                            price: margin_call_px,
                            qty: info.perps,
                            ref_discount: 0f64,
                            ref_ratio: 0f64,
                            ref_expiration: 0,
                            ref_client_id: ClientId(0),
                            client_id: temp_client_id,
                            trades_limit: MAX_MARGIN_CALL_TRADES - trades,
                            margin_call: true,
                        },
                    )?;
                    self.check_funding_rate(temp_client_id)?;
                    self.change_edge_px(temp_client_id); // <= 여기
```

```rust
        if info2.px_node != NULL_NODE {
            if info2.mask & 0x80000000 == 0 {
                let node: NodePtr<i128> =
                    unsafe { NodePtr::get(self.long_px.entry, info2.px_node) };
                self.long_px.delete(node);
            } else {
                let node: NodePtr<i128> =
                    unsafe { NodePtr::get(self.short_px.entry, info2.px_node) };
                self.short_px.delete(node);
            }
        }
```

- 청산 후 상태 확인: 청산 후 `is_long_margin_call()`은 `long_px` 트리에서 현재 `max_node()`를 확인합니다. 그러나 이 노드는 방금 청산된 노드와 다를 수 있습니다. 그리고 다른 청산 상태를 가질 수 있습니다.

```rust
    pub fn is_long_margin_call(&self) -> bool {
        let root = self.long_px.get_root::<i128>();
        if root.is_null() {
            false
        } else {
            ((self.state.header.perp_underlying_px as f64
                / (1.0 + self.state.header.liquidation_threshold)) as i64)
                < (root.max_node().key() >> 64) as i64
        }
    }
```

거짓 부정(False Negative) 시나리오:
- 노드 A가 가장 위험한 노드임 (`px > margin_call_px`)
- `check_long_margin_call`이 노드 A를 성공적으로 청산함
- 노드 A가 `change_edge_px`를 통해 제거/업데이트됨
- 루프가 계속되어 노드 B를 확인
- 노드 B의 `px <= margin_call_px`이면 루프가 중단됨
- `is_long_margin_call`이 노드 B를 확인하고 false를 반환함
그러나 노드 A에 대해 청산이 이미 발생했으므로 시스템은 마진 콜 상태로 간주되어야 합니다.

청산이 발생했지만 `margin_call`이 잘못하여 `false`로 설정된 경우, 출금 계산은 마진 콜 경로 대신 일반 경로를 사용하여 잘못된 `amount` 계산을 유발합니다.

```rust
    let amount = if margin_call {
        let margin_call_funds =
            funds.min(engine.get_avail_funds(client_state.temp_client_id, true)?);
        if margin_call_funds <= 0 {
            bail!(ImpossibleToWithdrawFundsDuringMarginCall);
        }
        if data.amount == 0 {
            margin_call_funds
        } else {
            if margin_call_funds < data.amount {
                if funds >= data.amount {
                    bail!(ImpossibleToWithdrawFundsDuringMarginCall);
                }
                bail!(InsufficientFunds);
            }
            data.amount
        }
    } else if data.amount == 0 {
        funds
    } else {
        if funds < data.amount {
            bail!(InsufficientFunds);
        }
        data.amount
    };
```

또한 `check_rebalancing`이 불필요하게 호출됩니다.
```rust
    if !margin_call {
        engine.check_rebalancing()?;
    }
```

**Impact:** 청산이 발생했지만 `margin_call`이 잘못하여 false로 설정된 경우, 함수는 `margin_call`에 따라 다르게 동작합니다.

**Recommended Mitigation:** 반환 값 확인: `check_long_margin_call` 및 `check_short_margin_call`은 실행된 거래 수를 반환합니다. 이 반환 값을 사용하여 청산이 발생했는지 확인하십시오.


**Deriverse:** 커밋 [74f9650](https://github.com/deriverse/protocol-v1/commit/74f9650ef966b422d95a384bb1e39f4f7bd9cf22)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Stale edge price in liquidation tree after perp withdraw call

**Description:** `perp_withdraw`에서 무기한 포지션에서 자금을 인출한 후, 함수는 `change_edge_px`를 호출하여 청산 트리에서 청산 가격(엣지 가격)을 업데이트하지 않습니다.

자금이 인출되면 `info.funds`가 감소하여 `total_funds`에 영향을 미치고 결과적으로 엣지 가격에 영향을 미칩니다. 그러나 `change_edge_px`가 호출되지 않기 때문에 청산 트리(`long_px` 및 `short_px`)는 오래된 엣지 가격을 계속 저장합니다.

**Impact:** 출금 후 `change_edge_px`가 호출되지 않아 청산 트리가 오래된 엣지 가격을 유지합니다. 이는 잘못된 마진 콜 감지 및 청산 우선순위로 이어집니다.


**Recommended Mitigation:** 함수의 끝에서 자금을 인출하는 사용자에 대해 `change_edge_px`를 호출해야 합니다.

**Deriverse:** 커밋 [4f7bc8](https://github.com/deriverse/protocol-v1/commit/4f7bc8ac68325aa93b339ff91c0ac794ea17ffd9)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.



### Missing Update of `perp_spot_price_for_withdrowal` in `perp_withdraw` Function

**Description:** `perp_withdraw` 함수는 마진 콜이 없을 때 `perp_spot_price_for_withdrowal` 필드(이후 커밋의 `perp_long_spot_price_for_withdrowal` 및 `perp_short_spot_price_for_withdrowal` 포함)를 업데이트하지 못하는 반면, 다른 유사한 함수(`new_perp_order` 및 `perp_quotes_replace`)는 이 필드를 적절하게 업데이트합니다. 이러한 불일치로 인해 후속 마진 콜 시나리오에서 오래된 가격 데이터가 사용되어 잠재적으로 출금 계산에 영향을 미칠 수 있습니다.

`perp_spot_price_for_withdrowal` 필드는 마진 콜 상황에서 사용 가능한 자금을 계산할 때 `get_avail_funds` 함수에서 사용됩니다. 향후 작업에서 정확한 계산을 보장하기 위해 마진 콜이 없을 때 이 필드는 현재 `perp_underlying_px`로 업데이트되어야 합니다.

`new_perp_order` 및 `perp_quotes_replace`에서는 다음과 같습니다:

```rust
    if !long_margin_call {
        engine.state.perp_long_spot_price_for_withdrowal = engine.state.perp_underlying_px;
    }
    if !short_margin_call {
        engine.state.perp_short_spot_price_for_withdrowal = engine.state.perp_underlying_px;
    }
```

이제 `perp_withdraw`에서는 다음과 같습니다:

```rust
let margin_call = engine.is_long_margin_call() || engine.is_long_margin_call();
if !margin_call {
    engine.check_rebalancing()?;
}
```

**Impact:** `perp_withdraw`가 마진 콜 없이 호출되면 다른 함수가 업데이트하더라도 `perp_spot_price_for_withdrowal` 필드(이후 커밋의 `perp_long_spot_price_for_withdrowal` 및 `perp_short_spot_price_for_withdrowal` 포함)가 오래된 값을 유지할 수 있습니다. `perp_withdraw` 작업 후(동일한 트랜잭션 또는 후속 트랜잭션에서) 마진 콜이 발생하면 `get_avail_funds` 함수는 `margin_call=true`일 때 오래된 가격을 사용하여 잘못된 사용 가능 자금 계산으로 이어질 수 있습니다.

**Recommended Mitigation:** 일관성을 유지하고 가격을 최신 상태로 유지하기 위해 누락된 업데이트를 추가하십시오.

**Deriverse:** 커밋 [66c878](https://github.com/deriverse/protocol-v1/commit/66c878370dc8041d6544b8fdee636102ce00fe8c)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.




### Margin call uses stale edge price

**Description:** `check_long_margin_call` 및 `check_short_margin_call` 함수는 펀딩 비율 업데이트 및 리밸런싱을 적용하기 전에 가격 트리에서 검색된 엣지 가격을 기반으로 포지션을 청산해야 하는지 여부를 결정합니다. 이로 인해 펀딩 비율 변경 후의 현재 상태를 반영하지 않는 오래된 엣지 가격을 사용하여 포지션이 청산되어 불공정한 청산이 발생합니다.

`check_funding_rate` 및 `check_soc_loss`는 청산 결정이 내려진 후에 호출되지만, 이러한 함수는 사용자의 자금을 수정할 수 있어 청산 발생 여부에 직접적인 영향을 미칩니다.

```rust
pub fn check_long_margin_call(&mut self) -> Result<i64, DeriverseError> {
    let mut trades = 0;
    let margin_call_px = (self.state.header.perp_underlying_px as f64
        * (1.0 - self.state.header.liquidation_threshold)) as i64;

    loop {
        let root = self.long_px.get_root::<i128>();
        if root.is_null() || trades >= MAX_MARGIN_CALL_TRADES {
            break;
        }
        let node = root.max_node();
        let px = (node.key() >> 64) as i64;  // 오래된 엣지 가격 가져옴

        if px > margin_call_px {  // 오래된 가격으로 결정
            let temp_client_id = ClientId(node.link());

            // 청산 결정 후에 펀딩 비율 확인됨
            self.check_funding_rate(temp_client_id)?;
            self.check_soc_loss(temp_client_id)?;

            // ... 청산 로직 ...

            // 청산 후에 엣지 가격 업데이트됨
            self.change_edge_px(temp_client_id);
        }
    }
}
```


**Impact:** 사용자는 청산되지 않아야 할 때 청산되어 불필요한 자금 손실을 초래할 수 있습니다.

**Recommended Mitigation:** 포지션을 청산해야 하는지 결정하기 위해 `check_funding_rate` 및 `check_soc_loss`를 적용한 후 업데이트된 엣지 가격을 사용하는 것을 고려하십시오.

**Deriverse:** 커밋 [1efef6](https://github.com/deriverse/protocol-v1/commit/1efef63dfea11c8e031d1fe1fb0b48875a856153)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.




### Insurance fund decrease when `margin_call_penalty_rate` is less than `rebates_rate`

**Description:** `margin_call_penalty_rate`가 `rebates_rate`(`fee_rate * REBATES_RATIO`)보다 작으면 마진 콜 중 보험 기금이 증가하는 대신 감소합니다.

마진 콜 동안 보험 기금은 다음과 같이 업데이트됩니다: `perp_insurance_fund += (new_fees - rebates)`, 여기서:
- `new_fees = traded_crncy * margin_call_penalty_rate`
- `rebates = traded_crncy * fee_rate * REBATES_RATIO`

`margin_call_penalty_rate < fee_rate * REBATES_RATIO`이면 `(new_fees - rebates)`가 음수가 되어 보험 기금이 증가해야 할 때 감소하게 됩니다.

**Impact:** 보험 기금은 보충되는 대신 마진 콜 중에 고갈될 수 있습니다.

**Recommended Mitigation:** 마진 콜 페널티가 항상 보험 기금에 긍정적으로 기여하도록 보장하기 위해 `margin_call_penalty_rate >= fee_rate * REBATES_RATIO`인지 확인하십시오.

**Deriverse:** 커밋 [1ef948](https://github.com/deriverse/protocol-v1/commit/1ef948af18b47f3e502d0054d760212d4b6263f1)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.



\clearpage
## Low Risk


### Casting from `u64` to `i64` causes genuine deposit requests to fail in `deposit` function

**Description:** `deposit.rs`의 `deposit` 함수는 `deposit_all`이 `true`로 설정된 경우 `u64` 유형인 `amount` 값(트레이더가 입금하려는 토큰 양)을 `i64`로 캐스팅합니다. 토큰 양이 `i64::MAX`(9,223,372,036,854,775,807)를 초과하면 릴리스 빌드의 Rust 기본 오버플로 동작으로 인해 캐스트가 음수 값으로 래핑됩니다. 금액은 나중에 SPL 토큰 전송을 위해 u64로 다시 캐스팅되지만 `u64`로 다시 캐스팅하기 전에 이 확인이 있습니다.
```rust
    if !(1..=SPOT_MAX_AMOUNT).contains(&amount) {
        bail!(InvalidQuantity {
            value: amount,
            min_value: 1,
            max_value: SPOT_MAX_AMOUNT,
        });
    }
```
여기서 목표는 1과 36028797018963967 사이의 금액에 하한과 상한을 두는 것입니다. `u64`에서 `i64`로 금액을 캐스팅했으므로 금액이 크면 음수로 바뀌었을 수 있으며 이 음수는 원하는 범위에 있지 않으므로 트랜잭션이 진행되지 않습니다.

**Impact:** 특히 큰 토큰 소수점 민트에 대한 진정한 트랜잭션이 되돌려질(reverted) 수 있습니다.

**Proof of Concept:**
```rust
fn main() {
    let a: i64;

    let b: u64 = 15_000_000_000_000_000_000;

    a = b as i64;  // 캐스팅

    println!("a = {}", a);  //
}
Output: a = -3446744073709551616
```

**Recommended Mitigation:** 입력 금액을 `i64`로 변환하지 말고 대신 다음과 같이 할 수 있습니다:
```rust
const SPOT_MAX_AMOUNT_U64: u64 = SPOT_MAX_AMOUNT as u64;
```
**Deriverse:** 커밋 [801209](https://github.com/deriverse/protocol-v1/commit/801209dc5d425dd0a3177d4a41660e0e4ed91bda)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Expired Private Client Cannot Be Re-added to Queue

**Description:** `new_private_client()` 함수는 이전 항목이 만료된 지갑의 재추가를 잘못 거부합니다. 이 함수는 만료 상태를 확인하기 전에 중복 지갑 주소를 확인하여 만료된 기록이 빈 슬롯으로 처리되는 것을 방지합니다.

기록 삽입 로직(라인 131-133)에서 코드는 지갑이 이미 존재하는지 확인합니다:
```rust
if record.wallet == *wallet.key && record.creation_time != 0 {
    return Err(AlreadyExists(index));
}
```
그러나 이 확인은 만료 유효성 검사 전에 발생합니다. `PrivateClient::is_vacant()` 메서드(`src/state/private_client.rs`에 정의됨)에 따르면 기록은 다음 중 하나인 경우 비어 있는 것으로 간주되어야 합니다:
1. `creation_time == 0` (초기화되지 않음), 또는
2. `current_time > expiration_time` (만료됨)

문제는 삽입 위치를 찾기 위해 기록을 반복할 때 만료 상태에 관계없이 일치하는 지갑이 발견되면 라인 131의 중복 확인이 즉시 오류를 반환한다는 것입니다. 이로 인해 코드가 라인 136의 `is_vacant()` 확인에 도달하지 못하게 되어 만료된 기록을 재사용 가능한 슬롯으로 올바르게 식별하지 못합니다.

**Impact:** **큐 슬롯 고갈:** 만료된 개인 클라이언트는 갱신할 수 없습니다.


**Recommended Mitigation:** 오류를 반환하기 전에 만료 상태를 확인하도록 중복 지갑 확인을 수정하십시오. 지갑이 일치하고 기록이 여전히 유효한(만료되지 않은) 경우에만 `AlreadyExists`를 반환하십시오.

**Deriverse:** 커밋 [a626a26](https://github.com/deriverse/protocol-v1/commit/a626a2626a483e55f76b582f5bd49ff2a7b2d62a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Account Count Validation Mismatch in `new_base_crncy` Instruction

**Description:** `new_base_crncy` 명령은 선언된 최소 계정 수와 필요한 실제 계정 수 사이에 불일치가 있습니다.

함수 주석과 구현 모두 `9`개의 계정이 필요하다고 나타내지만 `NewBaseCrncyInstruction::MIN_ACCOUNTS`는 `8`로 설정되어 있습니다.

```rust
pub fn new_base_crncy(program_id: &Pubkey, accounts: &[AccountInfo], _: &[u8]) -> DeriverseResult {
    // New Base Currency Instruction
    // 1 - Admin (Signer)
    // 2 - Root Account (Read Only)
    // 3 - Token Account
    // 4 - Program Token Account (Signer when creating a new token)
    // 5 - Deriverse Authority Account (Read Only)
    // 6 - Token Program ID (Read Only)
    // 7 - Mint Address (Read Only, when wSOL can not be old native mint)
    // 8 - System Program (Read Only)
    // 9 - Community Account
```

함수는 먼저 8개의 계정을 읽은 다음 나중에 9번째 계정을 읽습니다:

```rust:145:src/program/processor/new_base_crncy.rs
    let community_acc = next_account_info!(accounts_iter)?;
```


그러나 유효성 검사 확인은 8개의 계정만 요구합니다:

```rust:50-55:src/program/processor/new_base_crncy.rs
    if accounts.len() < NewBaseCrncyInstruction::MIN_ACCOUNTS {
        bail!(InvalidAccountsNumber {
            expected: NewBaseCrncyInstruction::MIN_ACCOUNTS,
            actual: accounts.len(),
        });
    }
```

여기서 `MIN_ACCOUNTS`는 다음과 같이 정의됩니다:

```rust:281-285:constants.rs
    pub struct NewBaseCrncyInstruction;
    impl DrvInstruction for NewBaseCrncyInstruction {
        const INSTRUCTION_NUMBER: u8 = 4;
        const MIN_ACCOUNTS: usize = 8;
    }
```

**Impact:**
1. **잘못된 유효성 검사**: 명령은 실제로 `9`개가 필요할 때 `8`개의 계정을 허용하여 유효성 검사와 실제 계정 요구 사항 간의 불일치를 만듭니다.
2. **늦은 실패**: `8`개의 계정만 있는 트랜잭션은 초기 확인을 통과하지만 실행 중에 나중에 실패합니다.

**Recommended Mitigation:** 실제 계정 요구 사항과 일치하도록 `MIN_ACCOUNTS`를 `9`로 업데이트하십시오:

```rust
pub struct NewBaseCrncyInstruction;
impl DrvInstruction for NewBaseCrncyInstruction {
    const INSTRUCTION_NUMBER: u8 = 4;
    const MIN_ACCOUNTS: usize = 9;  // 8에서 9로 변경됨
}
```

**Deriverse:** 커밋 [f54117b0](https://github.com/deriverse/protocol-v1/commit/f54117b09012e11e0003844f5f855e6f878d3f73)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Inflexible Voting System Prevents Rapid Parameter Adjustments

**Description:** 거버넌스 투표 시스템은 어떤 매개변수를 수정할 수 있는지가 `voting_counter % 6`에 의해 결정되는 엄격한 순환 메커니즘을 사용합니다.

이로 인해 각 매개변수가 6번의 투표 기간마다 한 번만 투표될 수 있는 고정된 6개 매개변수 순환 주기가 생성됩니다. 14일의 투표 기간과 결합하면 특정 매개변수는 약 84일(6기간 × 14일) 후에 다시 수정될 수 있어, 긴급 상황에 대응하거나 동일한 매개변수를 연속적으로 조정하는 프로토콜의 능력을 심각하게 제한합니다.

매개변수 선택은 `finalize_voting()`에서 결정됩니다:

```rust
let tag = community_account_header.voting_counter % 6;
match tag {
    0 => spot_fee_rate,
    1 => perp_fee_rate,
    2 => spot_pool_ratio,
    3 => margin_call_penalty_rate,
    4 => fees_prepayment_for_max_discount,
    _ => max_discount,
}
```

`voting_counter`는 `finalize_voting()`에서 1씩만 증가할 수 있습니다:

```rust
community_account_header.voting_counter += 1;
```

그리고 `next_voting()`에서 카운터는 0일 때만 1로 증가되거나, 확정될 때(1씩 증가)만 가능합니다:

```rust
if community_state.header.voting_counter == 0 {
    community_state.header.upgrade()?.voting_counter += 1;
}
community_state.finalize_voting(clock.unix_timestamp as u32, clock.slot as u32)?;
```

**The Problem:**
1. 투표 라운드를 건너뛰거나 특정 매개변수를 직접 타겟팅하는 메커니즘이 없습니다.
2. `voting_counter`는 순차적으로만 증가할 수 있으며 건너뛸 수 없습니다.
3. 매개변수에 긴급 조정이나 연속 수정이 필요한 경우 프로토콜은 전체 6개 매개변수 주기를 기다려야 합니다.
4. 운영자나 관리자가 순환 일정을 재정의할 수 있는 비상 메커니즘이 존재하지 않습니다.

**Example Scenario:**
1. 투표 기간 1 (`voting_counter` = 1): 커뮤니티가 `perp_fee_rate` 감소(태그 `1`)에 투표합니다.
2. 14일 후 변경 사항이 적용되고 `voting_counter`는 2가 됩니다.
3. 시장 상황이 변경되어 `perp_fee_rate`에 대한 또 다른 즉각적인 조정이 필요합니다.
4. 프로토콜은 voting_counter = `7, 13, 19` 등(매 6번째 기간)을 기다려야 합니다.
5. 이는 `perp_fee_rate`에 다시 투표할 수 있기까지 약 `70일`(5개 기간 더 × 14일)을 기다려야 함을 의미합니다.

```rust
#[cfg(not(feature = "test-sbf"))]
pub fn voting_end(time: u32) -> u32 {
    let days = (time - SETTLEMENT) / DAY;
    days * DAY + 14 * DAY + SETTLEMENT
}

```

**Impact:**
- **시장 상황에 대한 대응 지연:** 프로토콜은 연속적인 매개변수 조정이 필요한 긴급 상황에 신속하게 대응할 수 없습니다.
- **비효율적인 거버넌스:** 매개변수가 최적 값에 도달하기 위해 여러 번 조정해야 하는 경우 프로세스는 몇 주가 아니라 몇 달(5*14 = 70일)이 걸립니다.

**Recommended Mitigation:** 정상적인 운영을 위해 기본 순차 증가를 유지하면서 극단적인 상황에서 운영자가 다음 `voting_counter` 값을 지정할 수 있도록 하는 선택적 메커니즘을 추가하는 것을 고려하십시오.

**Deriverse::**
커밋 [bb853ad](https://github.com/deriverse/protocol-v1/commit/bb853adf0cfecf974b1b1933a6192b4dfb7e42ae)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Dividend Calculation Uses Stale Token Balance in Subsequent `update()` Calls After `fees_deposit` with DRVS

**Description:** `fees_deposit()` 함수는 `client_state.sub_crncy_tokens()` 전에 `client_community_state.update()`를 호출하여, 감소 후의 실제 토큰 잔액을 반영하지 않는 오래된 값으로 `self.header.drvs_tokens`가 업데이트되게 합니다. 후속 명령(예: `deposit`, `spot_quotes_replace`)에서 `update()`가 다시 호출되면 실제 현재 잔액 대신 이 오래된 저장된 값을 사용하여 배당금을 계산합니다.

**Prerequisites**: 이 문제는 `fees_deposit`이 `token_id = 0`(DRVS 토큰)으로 호출될 때만 발생합니다.

이 문제는 다음과 같은 이유로 발생합니다:

1. **작업 순서**: `fees_deposit()`에서 `update()`가 `sub_crncy_tokens()` 전에 호출됩니다.
2. **오래된 잔액 저장**: `update()`는 현재 잔액(감소 전)을 읽고 `self.header.drvs_tokens`에 저장하지만, `sub_crncy_tokens()`는 그 후에 실제 잔액을 줄입니다.
3. **오래된 배당금 계산**: 후속 명령에서 `update()`가 다시 호출되면 실제 현재 잔액 대신 `self.header.drvs_tokens`(감소 전의 오래된 저장된 값)를 사용하여 배당금을 계산합니다.

```rust
// src/program/processor/fees_deposit.rs
client_community_state.update(&mut client_state, &mut community_state)?;
client_state.resolve(AssetType::Token, data.token_id, TokenType::Crncy, false)?;
// ... 수수료 계산 로직 ...
client_state.sub_crncy_tokens(data.amount)?;
```

```rust
// src/state/client_community.rs
        if available_tokens != self.header.drvs_tokens {
```rust
            for (i, d) in self.data.iter_mut().enumerate() {
                let amount = (((community_state.base_crncy[i].rate - d.dividends_rate)
                    * self.header.drvs_tokens as f64) as i64)
                    .max(0);
                d.dividends_value += amount;
            }
```

**문제를 보여주는 예시 시나리오:**

초기 상태: 사용자는 100 DRVS 토큰을 보유, `self.header.drvs_tokens = 100`

1. **첫 번째 `fees_deposit(token_id=0, amount=10)` 호출:**
   - `update()` 호출:
     - `resolve(AssetType::Token, 0, ...)`이 DRVS 토큰 확인 (token_id = 0)
     - `available_tokens = 100` (감소 전 현재 잔액)
     - `self.header.drvs_tokens = 100` (저장된 값)
     - 서로 같으므로 배당금 계산이 발생하지 않음
     - `self.header.drvs_tokens`가 `100`으로 업데이트됨
   - `sub_crncy_tokens(10)` 호출, 실제 잔액은 `90`이 됨
   - **결과**: `self.header.drvs_tokens = 100` (오래됨), 그러나 `실제 잔액 = 90`

2. **`update()`를 호출하는 다음 명령 (예: `deposit`, `spot_quotes_replace`):**
   - `update()` 다시 호출:
     - `available_tokens = 90` (실제 현재 잔액, 또는 다른 수치일 수 있음)
     - `self.header.drvs_tokens = 100` (1단계의 오래된 저장 값)
     - 서로 다르므로 `self.header.drvs_tokens = 100`을 사용하여 배당금 계산이 발생함
     - **문제**: 배당금은 `100` 토큰을 기준으로 계산되지만, 사용자는 이 기간 동안 `90` 토큰만 보유함
   - **결과**: 사용자는 `90`이 아닌 `100` 토큰에 대해 계산된 배당금을 받아, 받아야 할 것보다 더 많이 받음

근본 원인은 `fees_deposit()`에서 `update()`가 `sub_crncy_tokens()`가 잔액을 줄이기 전에 잔액을 저장하여, 저장된 값과 실제 잔액 간의 불일치를 생성하기 때문입니다. 나중에 `update()`가 다시 호출되면 배당금 계산에 이 오래된 저장 값을 사용합니다.

**Impact:** **배당금 과다 지급**: `fees_deposit(token_id=0)` 호출 후 후속 명령에서 `update()`가 호출되면, 사용자는 실제로 보유한 것보다 더 높은 토큰 잔액에 대해 계산된 배당금을 받게 되어 프로토콜의 재정적 손실로 이어집니다.

**Recommended Mitigation:** `drvs_token`이 항상 최신 상태가 되도록 순서를 재배열해야 합니다.

**Deriverse:** 커밋 [4df80d](https://github.com/deriverse/protocol-v1/commit/4df80d97a4b72e144ccabf6956d10d463d4ca91e)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Griefing Attack: Malicious Takers Can Force Order Cancellation by Partial Filling Below Minimum Quantity

**Description:** 악의적인 테이커(taker)는 남은 수량이 `min_qty` 임계값 아래로 떨어지도록 의도적으로 주문을 부분 체결함으로써 메이커(maker)의 주문이 강제로 취소되도록 자동 주문 취소 메커니즘을 악용할 수 있습니다.

이런 일이 발생하면 시스템은 자동으로 주문을 취소하고 남은 잠긴 자금을 메이커에게 환불합니다. 그러나 환불된 금액이 `min_qty`보다 적기 때문에 메이커는 이 자금으로 새 주문을 할 수 없어 정상적인 거래 운영을 방해하는 그리핑(griefing) 공격을 효과적으로 생성합니다.

취약점은 현물 거래 엔진의 `fill` 함수에 존재합니다. 주문이 부분적으로 채워지면(last = true), 시스템은 `erase_client_order(order, node, true, side)`를 통해 자동으로 주문을 취소합니다.


```rust
// src/program/spot/engine.rs:1265-1279
if last {
    order.decr_qty(traded_qty).map_err(|err| drv_err!(err))?;
    order.decr_sum(traded_crncy).map_err(|err| drv_err!(err))?;
    order.set_time(self.time);
    if order.qty() < min_qty {  // ⚠️ 확인
        let node = self.get_node_ptr(order.link(), fill_static_args.side);
        self.erase_client_order(order, node, true, fill_static_args.side)?;
        // ... 주문 취소됨 및 자금 환불됨
    }
}
```

공격 시나리오:
- 메이커가 100 토큰 수량으로 주문을 넣음 (`min_qty = 2`)
- 악의적인 테이커가 의도적으로 99 토큰을 매칭시켜 `1` 토큰을 남김
- `1 < min_qty (2)`이므로 시스템이 자동으로 주문을 취소함
- 남은 `1` 토큰이 `erase_client_order`를 통해 메이커에게 환불됨
- `min_qty`보다 작으므로 메이커는 환불된 1 토큰으로 새 주문을 할 수 없음


**Impact:**
- 그리핑 공격: 공격자는 주문을 체계적으로 타겟팅하고 `min_qty` 미만의 먼지(dust) 수량을 남겨 취소를 강제할 수 있습니다.
- **`min_qty` 미만의 환불된 금액은 새 주문을 하는 데 사용할 수 없어 소액의 자금을 효과적으로 잠급니다**. 먼지 수량을 처리하는 방법이 불확실하기 때문에 이것이 주문 취소 프로세스에만 국한되지 않고 전체 리포지토리에 걸쳐 있다고 생각합니다.

**Recommended Mitigation:** 이에 대해 두 가지 가능한 제안이 있습니다:
- 부분 체결 시 자동 취소 방지
- 사용자가 AMM에서 직접 먼지 수량을 판매할 수 있는 방법 추가

**Deriverse:** 커밋 [134db8b](https://github.com/deriverse/protocol-v1/commit/134db8b1dac9e48641dac1a6b95bcf34637f695f)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 사용자는 시장가 주문을 사용하여 소액을 처리할 수 있습니다.


### Missing `last_time` Update in `spot_lp` Causes Incorrect Daily Trade Statistics

**Description:** `spot_lp` 함수는 `instr_state.header.last_time`을 사용하여 하루가 지났는지 확인하지만 이 필드를 업데이트하지 않습니다.

대조적으로, 다른 거래 함수(`swap, new_spot_order, spot_quotes_replace`)는 `engine.write_last_tokens()`를 통해 `last_time`을 업데이트합니다. 이러한 불일치로 인해 최근에 일반 거래 작업이 발생하지 않았을 때 잘못된 일일 LP 거래 통계가 발생합니다.

`src/program/processor/spot_lp.rs`에서 함수는 다음을 사용하여 하루가 지났는지 확인합니다:

```solidity
if instr_state.header.last_time < fixing_time {
    instr_state.header.lp_prev_day_trades = instr_state.header.lp_day_trades;
    instr_state.header.lp_day_trades = 1;
} else {
    instr_state.header.lp_day_trades += 1;
}
```

그러나 `instr_state.header.last_time`은 `spot_lp` 함수에서 업데이트되지 않는 반면 다른 거래 함수는 이를 업데이트합니다.

다음 경우를 고려하십시오:
- 마지막 일반 거래(swap/new_spot_order)가 1일 전에 발생하여 last_time을 해당 타임스탬프로 설정함
- 오늘 여러 번의 spot_lp 작업이 발생함
- 각 spot_lp에 대해:
    - `last_time`은 1일 전 상태로 유지됨
    - `fixing_time`은 오늘의 결제 시간임
    - `last_time < fixing_time`은 항상 참임
    - `lp_day_trades`는 증가하는 대신 1로 재설정됨


**Impact:** 이러한 경우(마지막 일반 거래가 1일 전에 발생), 같은 날의 모든 LP 거래가 그날의 첫 번째 거래로 계산되어 정확한 일일 통계가 손실됩니다.

**Recommended Mitigation:** 날짜 교차 확인 후 `spot_lp` 함수에서 `last_time`을 업데이트하십시오.

```rust
instr_state.header.lp_day_trades += 1;
}
instr_state.header.last_time = time;  // 이 라인 추가
```

**Deriverse:** 커밋 [29281c5](https://github.com/deriverse/protocol-v1/commit/29281c5e489ad01efbb5124e62c45f55aa309348)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Inconsistent Price Reference Used for Trade Execution Logic in `new_spot_order` and `swap`

**Description:** `new_spot_order` 및 `swap` 함수는 주문 실행 여부를 결정하기 위한 기준 가격(`px`)을 얻기 위해 서로 다른 방법을 사용하므로 코드베이스에 불일치가 발생합니다.


`src/program/processor/new_spot_order.rs`에서:
```rust
let px = engine.market_px();
```

`src/program/processor/swap.rs`에서:
```rust
let px = engine.state.header.last_px;
```

`src/program/processor/spot_quotes_replace.rs`에서:
```rust
let px = engine.state.header.last_px;
```

**차이점:**

`market_px()` 함수(`engine.rs`의 1898-1906 라인)는 다음을 반환합니다:
- `best_ask` (만약 `best_ask < last_px`)
- `best_bid` (만약 `best_bid > last_px`)
- `last_px` (그 외의 경우)

즉, `market_px()`는 실제 마지막 거래 가격(`last_px`)이 아니라 호가창 가격(`best_ask` 또는 `best_bid`)을 반환할 수 있습니다.

**Impact:** 두 함수 모두 대체 수단으로 `|| engine.cross(...)`를 사용하지만, 다른 `px` 값은 보안 취약점을 직접 유발하지는 않을 수 있지만 다음과 같은 이유로 미묘한 동작 차이를 초래할 수 있습니다:

- `px` 값은 `drv_update`에 전달되어 가격 고정(fixing price) 계산에 사용됩니다.
- 또한 `px` 값은 `write_last_tokens`에 전달되어 날짜 경계를 넘을 때 `last_close` 및 `day_low`를 설정하는 데 사용됩니다.

**Recommended Mitigation:** 모든 현물 거래 함수에서 가격 참조를 표준화하십시오.

**Deriverse:** 커밋 [f7e57dc](https://github.com/deriverse/protocol-v1/commit/f7e57dcf4bac3b7ec6bf4fdc07198abb7a9a443e)에서 수정되었습니다.

**Cyfrin:** 검증됨


### Missing Signer and New Account Validation for `asset_token_program_acc` in `new_instrument`

**Description:** `new_instrument` 명령은 새 자산 토큰 계정을 생성할 때 `asset_token_program_acc`에 대한 유효성 검사가 부족합니다. 프로그램 토큰 계정이 서명자이고 새 계정인지 명시적으로 검증하는 `new_base_crncy`와 달리 `new_instrument`는 이러한 확인을 생략하여 일관성 없는 오류 처리로 이어집니다.

new_instrument.rs에서 `is_new_account(asset_token_acc)`가 true이면 코드는 내부적으로 `create_programs_token_account`를 호출하는 `TokenState::create_token`을 직접 호출합니다. 이 함수는 시스템 작업을 수행하므로 `asset_token_program_acc`가 서명자여야 합니다:

```rust
    if is_new_account(asset_token_acc) {
        let decimals = *asset_mint
            .data
            .borrow()
            .get(MINT_DECIMALS_OFFSET)
            .ok_or_else(|| drv_err!(DeriverseErrorKind::InvalidClientDataFormat))?
            as u32;

        if !(MIN_DECS_COUNT..=MAX_DECS_COUNT).contains(&decimals) {
            bail!(DeriverseErrorKind::InvalidDecsCount {
                decs_count: decimals,
                min: MIN_DECS_COUNT,
                max: MAX_DECS_COUNT,
                token_address: *asset_mint.key,
            });
        }

        #[cfg(feature = "native_mint_2022")]
        if *asset_mint.key == spl_token::native_mint::ID {
            bail!(LegacyNativeMintNotSupported);
        }
        #[cfg(not(feature = "native_mint_2022"))]
        if *asset_mint.key == spl_token_2022::native_mint::ID {
            bail!(Token2022NativeMintNotSupported);
        }

        TokenState::create_token(
            root_state,
            asset_mint,
            asset_token_acc,
            drvs_auth_acc,
            &drvs_auth,
            bump_seed,
            program_id,
            asset_token_program_acc,
            token_program,
            signer,
            decimals,
        )?;
```

그러나 new_instrument는 다음을 검증하지 않습니다:
- `asset_token_program_acc.is_signer`가 참인지 여부
- `asset_token_program_acc`가 새 계정인지 여부 (`is_new_account(asset_token_program_acc)`)

그러나 주석에는 새 토큰을 생성할 때 이 계정이 서명자여야 한다고 나와 있으므로 이것이 필요합니다.

```rust
///
/// [*`NewInstrumentInstruction` 명령 중 `NewInstrumentData` 구조체 생성 시 잘못된 가격 검증*](#newinstrumentinstruction-명령-중-newinstrumentdata-구조체-생성-시-잘못된-가격-검증) - Asset Tokens Program Account `[SPL, if new_token signer]` - Spl token account
///
```

대조적으로 `new_base_crncy.rs`는 이러한 유효성 검사를 명시적으로 수행합니다:

```rust
if is_new_account(token_acc) {
    if !is_new_account(program_acc) {
        bail!(InvalidNewAccount { ... });
    }
    if !program_acc.is_signer {
        return Err(drv_err!(MustBeSigner { ... }));
    }
    // ...
}
```

참고: `new_root_account`에 대해서도 유사하게 작동하는데, 이에 대한 서명자도 확인해야 합니까?

```rust
///
/// [*변수의 오타 오류*](#변수의-오타-오류) - Deriverse Program Account `[SPL]` - Spl token account
///
```

**Impact:**
- 일관성 없는 오류 처리: 유사한 명령 전반에 걸쳐 다른 유효성 검사 패턴으로 인해 초기 유효성 검사보다는 CPI 호출 중에 실패가 발생합니다.

**Recommended Mitigation:** `is_new_account(asset_token_acc)`가 true일 때 `new_base_crncy.rs`의 패턴과 일치하도록 new_instrument.rs에 명시적 유효성 검사를 추가하십시오.

**Deriverse:** 커밋 [96f4e923](https://github.com/deriverse/protocol-v1/commit/96f4e92343fce5610ed89a3063bfed053bc578e5)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Referrer cannot be set after account creation

**Description:** 추천 시스템에는 치명적인 제한이 있습니다. 사용자는 새 계정을 생성할 때 **첫 입금 호출** 중에만 추천인을 설정할 수 있습니다. 계정 생성 후 추천인을 설정하거나 업데이트하는 기능이 없으므로 다음과 같은 몇 가지 문제가 발생합니다:

1. **첫 번째 사용자는 아무도 추천할 수 없음**: 계정을 생성한 첫 번째 사용자는 다음과 같은 이유로 아무도 추천할 수 없습니다:
   - `new_ref_link`를 통해 추천 링크를 생성할 수 있지만
   - 다른 사용자가 없으므로 자신의 첫 입금 중에 추천인을 설정할 수 없음
   - 첫 입금 후 추천인을 설정하는 기능이 없음

2. **추천인 설정을 놓친 사용자는 나중에 설정할 수 없음**: 첫 입금 중에 `ref_id`를 제공하지 않은 사용자는 추천인 설정 로직이 계정 생성 중에만 실행되므로 후속 입금에서 추천인을 설정할 수 없습니다.

**Impact:**
1. **첫 번째 사용자 문제**: 계정을 생성한 가장 첫 번째 사용자는 추천 링크를 생성할 수 있음에도 불구하고 아무도 추천할 수 없습니다. 이는 얼리 어답터를 위한 추천 프로그램을 중단시킵니다.

2. **영구적 제한**: 첫 입금 중에 `ref_id`를 포함하는 것을 잊은 사용자는 영구적으로 추천 프로그램에서 차단됩니다.

**Recommended Mitigation:** 계정 생성 후 사용자가 추천인을 설정할 수 있도록 허용하는 새 명령 `set_referrer`를 생성하십시오(적절한 제한 포함, 예: `ref_address`가 현재 설정되지 않은 경우에만).

**Deriverse:** 커밋 [993cf7](https://github.com/deriverse/protocol-v1/commit/993cf75f48a34aaae99421093ce59cb4f5e61f6b)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.



### Inconsistent Price Calculation for Fee in Spot LP Trading

**Description:** `spot_lp` 명령의 수수료 계산은 유동성을 추가할 때 사용되는 것과 동일한 가격 제약(`best_bid, best_ask`) 및 단위 변환(`RDF`)을 적용하지 않고 `last_px`를 직접 사용합니다. 이러한 불일치로 인해 `last_px`가 유효한 시장 가격 범위를 벗어날 때 잘못된 수수료 계산이 발생할 수 있습니다.

`src/program/processor/spot_lp.rs`에는 두 가지 다른 가격 계산 접근 방식이 있습니다:

1. 유동성 추가 시:

```rust
let px_f64 = instr_state
    .header
    .last_px
    .max(instr_state.header.best_bid)
    .min(instr_state.header.best_ask) as f64
    * RDF;
```

이 계산은 가격을 `[best_bid, best_ask]` 범위로 제한합니다.

2. 수수료 계산 시

```rust
fees = 1
    + ((instr_state.header.last_px as f64
        / get_dec_factor(instr_state.header.asset_token_decs_count) as f64)
        as i64)
        .max(1);
```

가격 범위 제약 없이 `last_px`를 직접 사용합니다.

**Impact:** `last_px`가 `[best_bid, best_ask]`를 벗어난 경우(예: 시장 변동성 또는 호가창 변경으로 인해), 잘못된 가격을 사용하여 수수료가 계산되어 사용자에게 과다 청구하거나 과소 청구할 수 있습니다.

**Recommended Mitigation:** 유동성 추가와 동일한 가격 로직을 사용하도록 수수료 계산을 업데이트하십시오:


**Deriverse:** 커밋 [40e36f70](https://github.com/deriverse/protocol-v1/commit/40e36f70a57a89a76ec19f9d825852bced7002dd)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `get_place_buy_price` function does not currently support the maximum `MAX_SUPPLY` value.

**Description:** 사용자가 좌석을 매수하거나 매도할 때마다, 우리는 새/현재 포지션과 이전 포지션의 get_reserve 차이를 사용하여 그 순간의 좌석 가격을 계산합니다.

```rust
    pub fn get_place_buy_price(supply: u32, dec_factor: u32) -> Result<i64, DeriverseError> {
        let df = get_dec_factor(dec_factor);
        Ok(get_reserve(supply + 1, df)? - get_reserve(supply, df)?)
    }

    pub fn get_place_sell_price(supply: u32, dec_factor: u32) -> Result<i64, DeriverseError> {
        let df = get_dec_factor(dec_factor);
        Ok(get_reserve(supply, df)? - get_reserve(supply - 1, df)?)
    }
```
`get_reserve`에서 우리는 `difference_to_max`를 계산하는데, 사용자가 250,000번째 포지션에 있을 때 0이 될 수 있습니다. 결과적으로 `reserve`는 무한대가 되어 `get_reserve`가 i64::MAX를 반환하게 합니다.

```rust
pub fn get_reserve(supply: u32, dec_factor: i64) -> Result<i64, DeriverseError> {
    let difference_to_max = MAX_SUPPLY - supply as i64;
    if difference_to_max < 0 {
        bail!(InvalidSupply {
            supply,
            supply_difference: difference_to_max as u32
        });
    }

    let reserve = MAX_SUPPLY as f64 * INIT_SEAT_PRICE * supply as f64 / difference_to_max as f64;

    return Ok((reserve * dec_factor as f64) as i64);
}
```

**Impact:** 250,000번째 포지션에서 구매하는 사용자는 249,999번째 포지션에서 구매하는 사용자보다 훨씬 더 많은 비용을 지불하게 됩니다. `supply + 1`에 대한 `get_reserve`가 i64::MAX를 반환하기 때문입니다.

**Recommended Mitigation:** 마지막 사용자에 대한 `difference_to_max`를 계산할 때, reserve가 무한대가 되지 않도록 +1을 추가해야 합니다.

**Deriverse:** 커밋 [c8d26d](https://github.com/deriverse/protocol-v1/commit/c8d26d57c1d9f24add2ed449f422d79ec983c13a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.




### `get_current_leverage` calculates leverage incorrectly for long positions

**Description:** 마진을 계산할 때 우리는 `(-(info.funds.min(perp_value as i64))).max(0)`을 사용합니다. 롱 포지션에서는 사용자 자금이 음수가 되고 숏 포지션에서는 `perp_value`가 음수가 됩니다.

그러나 레버리지를 계산할 때 우리는 `position_size / value`를 사용합니다. 롱 포지션의 경우 사용자 자금은 사실상 포지션을 오픈하는 데 사용된 초기 자금에서 포지션 크기를 뺀 것과 같으므로 잘못된 레버리지 계산으로 이어집니다.

시나리오:
100,000 USDC 가격의 비트코인에 대해 10배 롱 포지션을 오픈하려는 10,000 USDC 자금을 가진 사용자.
사용자는 10,000 USDC를 증거금으로 사용하고 1 BTC 롱 포지션에 진입합니다. 구매 후 사용자 자금은 –90,000 USDC가 되고 무기한 포지션은 1 BTC가 됩니다.

그러나 현재 레버리지를 계산할 때 함수는 10배 대신 9배를 반환합니다. 레버리지를 다음과 같이 계산하기 때문입니다: `max(-min(-90000, 100000), 0) / 10,000`

**Impact:** 이는 `check_client_leverage_shift`에서 구 레버리지와 새 레버리지를 비교하기 때문에 실질적인 영향은 없습니다. 레버리지가 증가하면 항상 새 레버리지가 더 높아진다는 논리가 여전히 유효하므로 현재 문제는 발생하지 않습니다.

**Recommended Mitigation:** 현재 레버리지를 계산할 때 `perp_value`를 사용해야 합니다.


**Deriverse:** 커밋 [e4c0a6](https://github.com/deriverse/protocol-v1/commit/e4c0a6fbc8ea72190df0b9e7df16e3db09d99a71)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Margin call limit bypass

**Description:** 마진 콜에 대한 제한 확인을 우회하여 마진 콜의 총 횟수가 `MAX_MARGIN_CALL_TRADES`를 초과하도록 허용할 수 있습니다.

이 문제는 첫 번째 `check_{short,long}_margin_call` 이후 다음 `check_{short,long}_margin_call`에서 거래 카운터가 재설정되어 의도한 제한을 넘어서는 추가 거래를 실행할 수 있기 때문에 발생합니다.

예:
`MAX_MARGIN_CALL_TRADES`가 10으로 설정됨.
`check_short_margin_call`이 9를 반환 (9번의 마진 콜이 실행됨을 의미).
9 < 10이므로 `check_long_margin_call`이 호출됨.
`check_long_margin_call`이 추가로 10번의 마진 콜을 실행함.

결과적으로 총 9 + 10 = 19번의 마진 콜이 실행되며, 이는 의도한 제한인 10을 초과합니다.

**Impact:** `MAX_MARGIN_CALL_TRADES` 제한을 초과할 수 있으며, 이는 더 많은 실행 비용으로 이어질 수 있습니다.

**Recommended Mitigation:** 두 함수 모두에서 누적 마진 콜 수를 추적하고 추가 마진 콜을 실행하기 전에 MAX_MARGIN_CALL_TRADES와 비교하십시오.

**Deriverse:** 커밋 [4f7bc8](https://github.com/deriverse/protocol-v1/commit/4f7bc8ac68325aa93b339ff91c0ac794ea17ffd9)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.




### Missing Fixing Window Data Accumulation After Daily Reset in `drv_update`

**Description:** `drv_update`가 새 날짜에 대한 고정(fixing) 데이터를 재설정할 때, 현재 `last_asset_tokens` 및 `last_crncy_tokens`가 새 날짜의 고정 데이터에 누적되는 대신 버려져 불완전한 고정 가격 계산으로 이어집니다.

`src/state/instrument.rs`에서 `drv_update` 함수는 일일 재설정 로직을 처리합니다:
```rust
if current_date > last_fixing_date {
    // 이전 날의 데이터를 기반으로 새 고정 가격 계산
    let new_fixing_px: i64 = if self.header.fixing_crncy_tokens
        > get_dec_factor(self.header.crncy_token_decs_count)
    {
        (self.header.fixing_crncy_tokens as f64 * self.header.dec_factor as f64
            / self.header.fixing_asset_tokens as f64) as i64
    } else {
        prev_px
    };
    self.header.fixing_asset_tokens = 0;  // 재설정
    self.header.fixing_crncy_tokens = 0;  // 재설정
    self.header.fixing_time = time;
    // ... 분산 및 fixing_px 업데이트
} else {
    // 고정 창 내에 있는 경우에만 누적
    let sec = time % DAY;
    if last_asset_tokens > 0 && (SETTLEMENT - FIXING_DURATION..SETTLEMENT).contains(&sec) {
        self.header.fixing_asset_tokens += last_asset_tokens;
        self.header.fixing_crncy_tokens += last_crncy_tokens;
    }
}
```

새 날짜(`current_date > last_fixing_date`)에 진입하면 코드는 `fixing_asset_tokens` 및 `fixing_crncy_tokens`를 `0`으로 재설정합니다.
그러나 현재 시간이 여전히 고정 창(`SETTLEMENT - FIXING_DURATION..SETTLEMENT`) 내에 있으면 현재 `last_asset_tokens` 및 `last_crncy_tokens`를 새 날짜의 고정 데이터에 누적해야 합니다. **현재, 이 값들은 재설정 후 버려지며 코드는 현재 시간이 고정 창 내에 있는지 확인하지 않습니다**


**Impact:** 일일 재설정 직후이지만 고정 창 내에서 발생하는 거래량은 고정 가격 계산에 포함되지 않습니다.

**Recommended Mitigation:** 고정 데이터를 재설정한 후 현재 시간이 고정 창 내에 있는지 확인하고 해당되는 경우 현재 데이터를 누적하십시오.

**Deriverse:** 커밋 [d17502c5](https://github.com/deriverse/protocol-v1/commit/d17502c5b1c9fc44abad88365f7d60b3b3325a26)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `get_by_tag` tries to access out of bound index

**Description:** [get_by_tag](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/constants.rs#L4)는 루프 조건에 1만큼 차이 나는(off-by-one) 오류를 포함합니다. 함수는 올바른 [while i < continer.candles.len()]() 대신 [while i <= continer.candles.len()](https://github.com/deriverse/protocol-v1/blob/30b06d2da69e956c000120cdc15907b5f33088d7/src/program/constants.rs#L9)을 사용하여 반복하므로 마지막 유효 인덱스를 지나 반복할 때 범위를 벗어난 배열 액세스를 유발합니다.
```rust
pub const fn get_by_tag<const TAG: u32>(
    continer: CandleRegister,
) -> Result<CandleParams, DeriverseError> {
    let mut i = 0;

    while i <= continer.candles.len() {  // 버그: `<=`가 아니라 `<`여야 함
        if continer.candles[i].tag == TAG {  // i == len일 때 범위를 벗어남
            return Ok(continer.candles[i]);
        }
        i += 1;
    }
    // ...
}
```
이 함수는 다른 캔들 관련 작업에서 코드베이스 전체에 걸쳐 호출되며 모든 경로가 영향을 받습니다. 배열의 앞부분에서 TAG가 발견되면 정상적인 작업 중에는 버그가 나타나지 않을 수 있지만, 일반적인 오류를 반환하는 대신 범위를 벗어난 인덱싱으로 인해 i == len()일 때 패닉이 발생합니다.
```rust
    Err(DeriverseError {
        error: DeriverseErrorKind::CandleWasNotFound { tag: TAG },
        location: ErrorLocation {
            file: file!(),
            line: line!(),
        },
    })
```

**Impact:** 요청된 TAG가 루프가 소진되기 전에 발견되지 않거나 i가 [continer.candles.len()]()에 도달하면 코드는 배열 범위를 벗어난 메모리에 액세스하려고 시도하여 적절한 오류를 발생하는 대신 프로그램 패닉을 유발합니다.

**Recommended Mitigation:**
```rust
pub const fn get_by_tag<const TAG: u32>(
    continer: CandleRegister,
) -> Result<CandleParams, DeriverseError> {
    let mut i = 0;

    while i < continer.candles.len() {  // 수정됨: `<=` 대신 `<` 사용
        if continer.candles[i].tag == TAG {
            return Ok(continer.candles[i]);
        }
        i += 1;
    }

    Err(DeriverseError {
        error: DeriverseErrorKind::CandleWasNotFound { tag: TAG },
        location: ErrorLocation {
            file: file!(),
            line: line!(),
        },
    })
}
```
**Deriverse:** 커밋에서 수정됨: [4e88698](https://github.com/deriverse/protocol-v1/commit/4e8869833c3de69ad84ed5b9f32fff3b560c5b93)

**Cyfrin:** 검증되었습니다.


### Incorrect Amount Logged in `PerpWithdrawReport`

**Description:** `perp_withdraw` 함수는 `PerpWithdrawReport` 이벤트에서 실제 인출된 금액(`amount`) 대신 요청된 금액(`data.amount`)을 기록합니다. 이는 특히 `data.amount == 0`(모든 사용 가능한 자금 인출)이거나 마진 콜 제한이 적용될 때 기록된 금액과 실제 이체된 자금 간에 불일치를 생성합니다.

`perp_withdraw` 함수에서 실제 인출 금액은 다양한 조건에 따라 계산됩니다:
```rust
let amount = if margin_call {
    let margin_call_funds =
        funds.min(engine.get_avail_funds(client_state.temp_client_id, true)?);
    if margin_call_funds <= 0 {
        bail!(ImpossibleToWithdrawFundsDuringMarginCall);
    }
    if data.amount == 0 {
        margin_call_funds  // 실제 금액은 data.amount와 다를 수 있음
    } else {
        // ... 유효성 검사 로직 ...
        data.amount
    }
} else if data.amount == 0 {
    funds  // 실제 금액은 data.amount와 다를 수 있음
} else {
    // ... 유효성 검사 로직 ...
    data.amount
};
```

실제 자금은 계산된 `amount` 변수를 사용하여 이체됩니다:

```rust
client_state.add_crncy_tokens(amount)?;
client_state
    .perp_info()?
    .sub_funds(amount)
    .map_err(|err| drv_err!(err))?;
```

그러나 로그 항목은 실제 `amount` 대신 `data.amount`를 기록합니다:

```rust
solana_program::log::sol_log_data(&[bytemuck::bytes_of::<PerpWithdrawReport>(
    &PerpWithdrawReport {
        tag: log_type::PERP_WITHDRAW,
        client_id: client_state.id,
        instr_id: data.instr_id,
        amount: data.amount,  // ❌ amount여야 함
        time: ctx.time,
        ..PerpWithdrawReport::zeroed()
    },
)]);
```

**Impact:** 로그가 실제 인출을 반영하지 않아 회계를 복잡하게 만듭니다.


**Recommended Mitigation:** 실제 인출된 `amount`를 기록하도록 로그 항목을 업데이트하십시오.

**Deriverse:** 커밋 [2a0fe33](https://github.com/deriverse/protocol-v1/commit/2a0fe336bb2a13bb3e96e1f065b907bc2d3e31bf)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Silent Failure in `voting_reset`

**Description:** `voting_reset` 함수는 쓰기를 위해 커뮤니티 계정 헤더 업그레이드를 시도할 때 실패를 조용히 무시합니다. `community_acc` 계정이 쓰기 가능한 것으로 표시되지 않은 경우 `upgrade()` 호출이 실패하지만 함수는 여전히 `Ok(())`를 반환하여 호출자에게 투표 매개변수가 성공적으로 재설정되지 않았음에도 재설정되었다는 잘못된 인상을 줍니다.

voting_reset 함수는 `if let Ok(header) = community_state.header.upgrade()`를 사용하여 조건부로 커뮤니티 계정 헤더를 업데이트합니다. 그러나 이 패턴은 계정이 쓰기 가능으로 표시되지 않을 때 발생하는 오류를 조용히 반환합니다.

```rust
    if let Ok(header) = community_state.header.upgrade() {
        header.spot_fee_rate = START_SPOT_FEE_RATE;
        header.perp_fee_rate = START_PERP_FEE_RATE;
        header.max_discount = START_MAX_DISCOUNT;
        header.margin_call_penalty_rate = START_MARGIN_CALL_PENALTY_RATE;
        header.fees_prepayment_for_max_discount = START_FEES_PREPAYMENT_FOR_MAX_DISCOUNT;
        header.spot_pool_ratio = START_SPOT_POOL_RATIO;
    }
```

코드베이스 전체에서 `upgrade`를 사용하는 다른 모든 함수는 `?` 연산자를 사용하여 오류를 적절하게 전파합니다:

```rust
    if let Some(ref mut header) = client_state.header {
        header.upgrade()?.points = 0;
        header.upgrade()?.mask &= 0xFFFFFFFFFFFFFF;
    }
```

```rust
        if community_state.header.voting_counter == 0 {
            community_state.header.upgrade()?.voting_counter += 1;
        }
```

**Impact:** `community_acc` 계정이 쓰기 가능한 것으로 잘못 표시되지 않은 경우(호출자의 버그 또는 구성 오류로 인해), 함수는 성공하는 것처럼 보이지만 상태 업데이트는 발생하지 않습니다.

**Recommended Mitigation:** `upgrade()` 실패 시 오류를 적절하게 전파하도록 코드를 변경하십시오.

**Deriverse:** 커밋 [9ef2d7](https://github.com/deriverse/protocol-v1/commit/9ef2d7602f2964b210e36b8d1de3360992d9caa2)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.

\clearpage
## Informational


### Incorrect Price Validation When Creating `NewInstrumentData` Struct during `NewInstrumentInstruction` instruction

**Description:** `NewInstrumentInstruction` 명령 중 `new` 메서드를 사용하여 `NewInstrumentData`를 생성할 때 현재 유효성 검사 로직은 가격 필드를 확인할 때 제외적 상한(exclusive upper bound)을 사용합니다. 결과적으로 MAX_PRICE와 동일한 가격은 유효한 것으로 간주되어야 함에도 불구하고 잘못 거부됩니다.

```rust
    fn new(instruction_data: &[u8], tokens_count: u32) -> Result<&Self, DeriverseError> {
        let data = bytemuck::try_from_bytes::<Self>(instruction_data)
            .map_err(|_| drv_err!(InvalidClientDataFormat))?;

        if data.crncy_token_id >= tokens_count {
            bail!(InvalidTokenId {
                id: data.crncy_token_id,
            })
        }

        if !(MIN_PRICE..MAX_PRICE).contains(&data.price) {                  //<- 여기
            bail!(InvalidPrice {
                price: data.price,
                min_price: MIN_PRICE,
                max_price: MAX_PRICE,
            })
        }

        return Ok(data);
    }
```

Rust에서 a..b 구문은 상한(b)을 제외하는 범위를 정의하는 반면, a..=b는 b를 유효한 값으로 허용하는 포함적 범위를 정의합니다.


**Impact:** 이 버그는 MAX_PRICE와 동일한 가격을 가진 합법적인 상품이 생성되는 것을 방지합니다.

**Recommended Mitigation:** 이 문제를 해결하려면 유효성 검사에서 포함적 범위(a..=b)를 사용하여 MAX_PRICE와 동일한 가격이 확인을 통과하도록 해야 합니다.
```rust
    fn new(instruction_data: &[u8], tokens_count: u32) -> Result<&Self, DeriverseError> {
        let data = bytemuck::try_from_bytes::<Self>(instruction_data)
            .map_err(|_| drv_err!(InvalidClientDataFormat))?;

        if data.crncy_token_id >= tokens_count {
            bail!(InvalidTokenId {
                id: data.crncy_token_id,
            })
        }

        if !(MIN_PRICE..=MAX_PRICE).contains(&data.price) {                  //<- 여기
            bail!(InvalidPrice {
                price: data.price,
                min_price: MIN_PRICE,
                max_price: MAX_PRICE,
            })
        }

        return Ok(data);
    }
```


**Deriverse:** 커밋 [aa6136](https://github.com/deriverse/protocol-v1/commit/aa613649a9dcd3394890a03e964a6d2b6b6570ad)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.




### typo error in variables

**Description:** 변수 `mint_token_prgoram_version`에 오타가 포함되어 있습니다: "prgoram"은 "program"이어야 합니다. 이 오타는 오류 메시지에서도 반복됩니다.

```rust

    let data = NewInstrumentData::new(instruction_data, root_state.tokens_count)?;
    let mint_token_prgoram_version = TokenProgram::new(asset_mint.owner)?;

    let token_program_version = TokenProgram::new(token_program.key)?;

    if token_program_version != mint_token_prgoram_version {
        bail!(DeriverseErrorKind::InvalidMintProgramId {
            expected: token_program_version,
            actual: mint_token_prgoram_version,
            mint_address: *asset_mint.key,
        });
    }
```



**Impact:** 기능에는 영향을 미치지 않지만 코드 가독성과 일관성을 떨어뜨립니다.

**Recommended Mitigation:** 변수 이름을 mint_token_program_version으로 변경하십시오:
```rust
-    let mint_token_prgoram_version = TokenProgram::new(asset_mint.owner)?;
+    let mint_token_program_version = TokenProgram::new(asset_mint.owner)?;

    let token_program_version = TokenProgram::new(token_program.key)?;

-    if token_program_version != mint_token_prgoram_version {
+    if token_program_version != mint_token_program_version {
        bail!(DeriverseErrorKind::InvalidMintProgramId {
            expected: token_program_version,
            actual: mint_token_program_version,
            mint_address: *asset_mint.key,
        });
    }
```
**Deriverse:** 커밋 [a3c75ae8](https://github.com/deriverse/protocol-v1/commit/a3c75ae87eb1f7122b7be778223950aae3cd91b5)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Transfers are noop when `lamports_diff`  is zero

**Description:** 코드베이스에는 `lamports_diff` 전송 금액이 0보다 큰지 확인하지 않고 공간 및 크기 재할당을 위해 lamports 전송을 수행하는 여러 인스턴스가 포함되어 있습니다. `lamports_diff`를 평가하기 위해 언더플로 가능성이 있는 경우 뺄셈을 제한하는 `saturating_sub`를 사용하고 있습니다.
```rust
// 여기서 계정에 임대료를 충당할 충분한 lamports가 이미 있는 경우 `lamports_diff`는 0으로 나옵니다.
let lamports_diff = new_minimum_balance.saturating_sub(account.lamports());
invoke(
    &system_instruction::transfer(signer.key, account.key, lamports_diff),
    &[signer.clone(), account.clone(), system_program.clone()],
)?;
```
`lamports_diff`가 0이면 코드는 여전히 시스템 프로그램의 전송 명령에 대한 불필요한 CPI 호출을 수행하여 계산 단위를 낭비하고 아무런 작업도 수행하지 않습니다(no op).

다음 파일들에 존재합니다: `perp_engine.rs`, `new_base_crncy.rs`, `new_operator.rs`, `engine.rs`, `candles.rs`, `client_community.rs`, `client_primary.rs`

**Impact:** `lamports_diff`==0인 경우 불필요한 cu 손실.

**Recommended Mitigation:** `lamports_diff` > 0인 경우에만 전송을 수행하십시오.
```rust
let lamports_diff = new_minimum_balance.saturating_sub(account.lamports());
if lamports_diff > 0 {
    invoke(
        &system_instruction::transfer(signer.key, account.key, lamports_diff),
        &[signer.clone(), account.clone(), system_program.clone()],
    )?;
}
```
**Deriverse:** 커밋에서 수정됨: [7091f4](https://github.com/deriverse/protocol-v1/commit/7091f48316d77e6769ee8fddeadd4d3a250ba1e1)

**Cyfrin:** 검증되었습니다.


### Entrypoint panics on empty `instruction_data`

**Description:** 문제는 opcode 디스패치 중에 첫 번째 요소에 액세스하기 전에 `instruction_data` 배열에 대한 범위 확인이 누락되어 발생합니다.

```rust
entrypoint!(process_instruction);

pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    match instruction_data[0] {
        NewHolderInstruction::INSTRUCTION_NUMBER => new_holder_account(program_id, accounts)?,
        NewOperatorInstruction::INSTRUCTION_NUMBER => {
            new_operator(program_id, accounts, instruction_data)?
        }
```


**Impact:** 이는 올바른 오류 메시지와 함께 정상적으로 종료되는 대신 `instruction_data`가 비어 있을 때 범위를 벗어난 읽기 및 프로그램 패닉을 초래할 수 있습니다.

**Recommended Mitigation:** 이 취약점을 완화하려면 프로그램은 항목에 액세스하기 전에 `instruction_data`의 길이를 검증하고 충돌에 의존하는 대신 오류를 정상적으로 처리해야 합니다.

**Deriverse:** 커밋 [8a2bd16](https://github.com/deriverse/protocol-v1/commit/8a2bd16db9fd84126fdf58c7f7eeb7b13410ba54)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Unnecessary Lamports Transfer Without Checking Existing Balance

**Description:** `new_private_client()` 함수는 계정에 이미 충분한 lamports가 있는지 먼저 확인하지 않고 `private_clients_acc` 계정으로 전체 `minimum_balance` 금액을 전송합니다. 이로 인해 계정에 임대료 요구 사항을 충당할 충분한 자금이 이미 포함되어 있는 경우에도 불필요한 전송이 발생합니다.
3. 그러나 **`READY_TO_DRV_UPGRADE`는 절대 독립적으로 확인되지 않습니다:**
   - `upgrade_to_perp` (라인 148-150)에서 `READY_TO_PERP_UPGRADE`만 확인됩니다:
   ```rust
   if instr_header.mask & instr_mask::READY_TO_PERP_UPGRADE == 0
       || instr_header.mask & instr_mask::PERP != 0
   ```
   - `READY_TO_DRV_UPGRADE`는 어떤 업그레이드 함수에서도 **절대 확인되지 않습니다**.

4. **`update_variance`에서 AND 로직과 함께만 사용됨:**
   - `update_variance` (라인 51-52)에서 두 플래그가 함께 확인됩니다:
   ```rust
   if self.mask & instr_mask::READY_TO_DRV_UPGRADE == 0
       && self.mask & instr_mask::READY_TO_PERP_UPGRADE == 0
   ```
   - 항상 같이 설정되므로 둘 다 확인하는 것은 중복입니다.

**Impact:** **데드 코드 / 중복**: `READY_TO_DRV_UPGRADE`는 다음과 같은 이유로 목적이 없는 것으로 보입니다:
   - DRV 업그레이드 기능이 없습니다.
   - 항상 `READY_TO_PERP_UPGRADE`와 함께 설정됩니다.
   - 독립적으로 확인되거나 사용되지 않습니다.

**Recommended Mitigation:** DRV 업그레이드가 계획되어 있지 않다면 플래그를 완전히 제거하십시오.

**Deriverse:** 커밋 [75da0c](https://github.com/deriverse/protocol-v1/commit/75da0cddb50f53240b8221dde055453a57495e0f)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Incorrect mask check in `check_points` function

**Description:** `check_points` 함수는 마일스톤 포인트를 확인해야 하는지 여부를 결정하기 위해 잘못된 비트마스크(0x7FF)를 사용합니다. 이 마스크는 비트 0–10을 포함하지만, 비트 0, 1, 2, 9, 10, 11, 12, 13을 확인해야 합니다. 결과적으로 사용자가 이미 8점을 모두 받았더라도 함수는 조건을 잘못 평가합니다.

```rust
if self.mask & 0x7FF != 0x7FF {
```

**Impact:** 마스크 값 0x7FF는 모든 마일스톤이 달성되었을 때 검사를 건너뛰어 함수를 최적화하려는 의도였지만 올바르게 작동하지 않습니다.

**Recommended Mitigation:** 0x3E07에 대해 확인해야 합니다. 만약 (mask & 0x3E07) != 0x3E07 이 거짓이면 포인트 확인을 건너뛰어야 합니다.
```rust
            if self.mask & 0x3E07 != 0x3E07 {
```

**Deriverse:** 커밋 [3ffcaa](https://github.com/deriverse/protocol-v1/commit/3ffcaadf2fb7210a0215d5c482a4c10a1bb77299)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.



\clearpage
