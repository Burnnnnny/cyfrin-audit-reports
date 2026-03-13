**수석 감사**

[Farouk](https://x.com/Ubermensh3dot0)

[Kose](https://x.com/0xKose)

**보조 감사**

---

# 발견 사항

## 높은 위험 (High Risk)

### 다수의 `vault_gkhan_account` 사용으로 `complete_vest`에서 DoS 발생 가능

**설명:** 프로그램은 다음 사항만 확인하여 `vault_gkhan_account`를 검증합니다:

* `vault_gkhan_account.owner == state.key()` 그리고
* `vault_gkhan_account.mint  == state.belo_mint`

이는 해당 계정이 ATA(Associated Token Account) 프로그램을 통해 생성된 정식 볼트 ATA인지 또는 하드코딩된 시드(seeds)로 파생된 PDA인지 **확인하지 않습니다**.
결과적으로 소유자(owner) 필드가 `state` PDA로 설정된 *모든* SPL 토큰 계정은 공격자에 의해 생성되었고 공격자의 긴밀한 권한 제어 하에 있더라도 제약 조건을 통과합니다.

1. **Alice**가 정상적인 입금을 합니다.
   * `100 BELO`를 공식 볼트 ATA(원래 초기화된 것)에 입금합니다.
   * `100 xBELO`를 받습니다.

2. **Bob**이 유령 볼트(phantom vault)를 만듭니다.
   * `owner = state_pda`, `mint = belo_mint`인 새 토큰 계정 `phantom_vault`를 생성하고 자신을 `close_authority`로 설정합니다.
   * `deposit`을 호출하여 `phantom_vault`를 `vault_gkhan_account`로 제공하고 `50 BELO`를 입금합니다.
   * `50 xBELO`를 받습니다.

3. Bob은 나중에 `complete_vest`를 호출하지만(베스팅 기간 후) **공식** 볼트 ATA를 전달합니다.
   * `50 BELO`가 공식 볼트에서 전송되고 그의 `burn_amount`가 거기서 소각됩니다.

4. Alice의 베스팅이 완료되면 공식 볼트에는 `50 BELO`만 보유하고 있으므로 그녀의 트랜잭션이 실패하여 사실상 그녀의 자금을 훔치거나(잠그게) 됩니다.

**파급력:** 공격자는 "유령" 볼트를 생성하여 이를 통해 입금한 다음 나중에 *실제* 볼트에서 상환할 수 있습니다. 이는 합법적인 볼트 잔액을 고갈시켜 프로토콜을 지급 불능 상태로 만들고 정직한 사용자가 베스팅된 토큰을 청구할 수 없게 만듭니다.

**권장 완화 방법:**
- **볼트 주소를 결정론적으로 바인딩하십시오.**
  고정된 시드로 볼트 PDA를 파생시키십시오. 예:
  ```rust
  let (vault_key, _bump) = Pubkey::find_program_address(
      &[b"vault", state.belo_mint.as_ref()],
      program_id,
  );
  require!(vault_gkhan_account.key() == vault_key, ErrorCode::InvalidTokenAccount);
  ```
- **또는** ATA 파생을 엄격하게 강제하십시오: 다음을 요구하십시오
  `vault_gkhan_account.key() == get_associated_token_address(&state.key(), &state.belo_mint)`.

**Doryoku:**
[f527d44](https://github.com/Warlands-Nft/xbelo/commit/f527d44e791a57cff7c81e102d5522ffca8489ca) 및 [034eaac](https://github.com/Warlands-Nft/xbelo/commit/034eaac1863cd4e409b7777220a5e083bdbde030)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 확장이 포함된 Raydium Token-2022 민트의 경우 하드코딩된 165바이트 볼트 계정이 너무 작음

**설명:** `stake_clmm_position`은 165바이트 크기의 Raydium 포지션 민트에 대한 볼트 토큰 계정을 생성합니다.

```rust
        // Create vault token account if it doesn't exist
        let vault_account_info = ctx.accounts.vault_position_token_account.to_account_info();
        if vault_account_info.data_is_empty() {
            // Create the vault token account
            let vault_account_space = 165; // Standard token account size
            let vault_account_lamports = ctx.accounts.rent.minimum_balance(vault_account_space);

            let farm_key = ctx.accounts.farm.key();
            let position_mint_key = ctx.accounts.position_mint.key();
            let vault_seeds: &[&[u8]] = &[
                VAULT_SEED,
                farm_key.as_ref(),
                position_mint_key.as_ref(),
                &[ctx.bumps.vault_position_token_account],
            ];
            let vault_signer_seeds = &[vault_seeds];

            // Create account
            anchor_lang::system_program::create_account(
                CpiContext::new_with_signer(
                    ctx.accounts.system_program.to_account_info(),
                    anchor_lang::system_program::CreateAccount {
                        from: ctx.accounts.user.to_account_info(),
                        to: vault_account_info.clone(),
                        to: vault_account_info.clone(),
                    },
                    vault_signer_seeds,
                ),
                vault_account_lamports,
                vault_account_space as u64,
                &ctx.accounts.token_program.key(),
            )?;

            // Initialize the token account
            anchor_spl::token_interface::initialize_account3(
                CpiContext::new(
                    ctx.accounts.token_program.to_account_info(),
                    InitializeAccount3 {
                        account: vault_account_info.clone(),
                        mint: ctx.accounts.position_mint.to_account_info(),
                        authority: ctx.accounts.vault_authority.to_account_info(),
                    },
                ),
            )?;
        }
```

상수 **165바이트**는 레거시 SPL-Token 크기입니다.
Raydium CLMM 포지션 NFT는 여러 온체인 확장(예: `MetadataPointer`, `MintCloseAuthority`)을 추가하는 **Token-2022** 민트입니다.

165바이트만 할당하면 `initialize_account3`이 [InvalidAccountData](https://github.com/solana-program/token-2022/blob/50a849ef96634e02208086605efade0b0a9f5cd4/program/src/processor.rs#L187-L191)로 실패하게 됩니다.

**파급력:** 사용자는 합법적인 Raydium 포지션을 스테이킹할 수 없으며 트랜잭션이 중단되어 파밍 풀이 차단됩니다.

**권장 완화 방법:** 민트의 확장에 따라 필요한 공간을 동적으로 계산하는 것이 좋습니다. 다음은 Raydium 코드베이스의 [예시](https://github.com/raydium-io/raydium-clmm/blob/835bc892352cd2be94365b49d024620a9b51f627/programs/amm/src/util/token.rs#L389-L445)입니다.

**Doryoku:**
[fad227e](https://github.com/Warlands-Nft/belo_clmm_lp_farming/commit/fad227e80c53e207a8836365ed0c8449a17f2992)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 낮은 위험 (Low Risk)

### xBELO 민트에 대한 하드코딩된 `9` 소수점 자리는 기본 BELO 민트와 불일치할 수 있음

**설명:** `initialize` 중에 프로그램은 다음과 같이 xBELO 수령 토큰을 생성합니다:

```rust
#[account(
    init, payer = admin,
    mint::decimals = 9,            // ← hard-coded
    mint::authority = state,
    mint::freeze_authority = state
)]
pub xgkhan_mint: Account<'info, Mint>,
```

소수점 자릿수는 `belo_mint.decimals`를 복사하는 대신 **9**로 고정됩니다.
정식 BELO 민트가 다른 정밀도(예: 6 또는 18)로 배포된 경우 볼트에 잠긴 BELO와 미상환 xBELO 간의 1 : 1 회계 가정이 위반됩니다.

**파급력:** 이로 인해 BELO와 xBELO 간의 1 : 1 회계 가정이 깨질 수 있습니다.

**권장 완화 방법:** * `initialize` 시 **`gkhan_mint`의 소수점 자릿수를 읽고 재사용하십시오**:

```diff
      #[account(
          init, payer=admin,
-         mint::decimals=9,
+         mint::decimals=gkhan_mint.decimals,
          mint::authority=state,
          mint::freeze_authority=state
      )]
      pub xgkhan_mint: Account<'info, Mint>,
```

**Doryoku:**
[74212b7](https://github.com/Warlands-Nft/xbelo/commit/74212b762fc90a0a1060522f50e5be96bf10a892)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 소각 로직의 부적절한 반올림으로 사용자가 페널티를 피할 수 있음

**설명:** 베스팅은 양수 값이라면 최소값 요구 사항 없이 진행될 수 있습니다.
소각될 금액은 현재 베스팅 시간에 따라 2%에서 50% 사이로 구성되어 있습니다.
`burn_amount`는 다음과 같이 계산됩니다:
```rust
        let burn_amount = amount
            .saturating_mul(burn_pct)
            .checked_div(100)
            .ok_or(ErrorCode::ArithmeticError)?;
```
여기서 `amount`는 사용자가 베스팅하는 정확한 금액을 나타내고, `burn_pct`는 소각 비율을 나타내며 그 값은 항상 **2**에서 **50** 사이입니다.
소각된 금액이 **100**보다 낮은 값을 곱한 `amount`로 계산된 다음 **100**으로 나누어지므로 이 값은 내림(rounding down)됩니다.
수수료/소각 관련 값을 올림(round up)하는 것이 항상 최적이며 이 구현은 이를 위반할 뿐만 아니라 사용자가 소각을 완전히 없애기 위해 프로그램을 조작할 수 있는 기회를 만듭니다.

**파급력:** 다음은 파급력을 보여주는 예시 시나리오입니다:
- Alice는 최대 베스팅 시간으로 **49** 토큰을 베스팅하므로 burn_pct는 **2**가 됩니다.
- 이로 인해 burn_amount는 **0**이 됩니다.
- Alice는 소각 없이 베스팅했으며 제한 없이 이를 반복할 수 있습니다.

**파급력:** 물질적 영향은 매우 제한적이지만 사용자가 아무것도 소각하지 않고 베스팅할 수 있게 합니다.
계산은 또한 모든 베스팅에 대해 소각 금액을 **1**만큼 과소평가하게 만듭니다.

**권장 완화 방법:** 나눗셈이 수행된 후 `burn_amount`에 **1**을 더하는 것을 고려하십시오:
```diff
        let burn_amount: u64 = amount
            .saturating_mul(burn_pct)
+           .checked_add(99)
+           .ok_or(ErrorCode::ArithmeticError)?;
            .checked_div(100)
            .ok_or(ErrorCode::ArithmeticError)?;
```
**Doryoku:**
[2a4e99c](https://github.com/Warlands-Nft/xbelo/commit/2a4e99cda5c5be1fae8f68023b7ec197a3229fc7)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 베스팅 기간이 범위를 벗어나 즉시 출금이 가능할 수 있음

**설명:** 베스팅에는 최소 및 최대 기간에 대한 제한이 있습니다. 초기 구성 내에서 최소 기간은 *10일*이고 최대 기간은 *180일*입니다.

그러나 `vest`는 베스팅 기간이 범위 내에 있는지 확인하지 않습니다(구체적으로 관련 확인이 주석 처리되어 있음):
```solidity
        // Validate vest duration is within bounds
        // require!(dur_secs >= min_secs, ErrorCode::VestDurationTooShort);
        // require!(dur_secs <= max_secs, ErrorCode::VestDurationTooLong);
```
이로 인해 사용자는 최소 기간보다 짧게 베스팅하고 최소 기간을 베스팅한 것처럼 처리될 수 있습니다.
특히 *1초* 동안 베스팅한 다음 *10일* 동안 베스팅된 것처럼 금액을 인출하는 것이 가능합니다.

**파급력:** 베스팅 기간 경계가 보호되지 않아 사용자가 최소 기간을 우회하고 연속적인 입금 및 출금을 수행하여 베스팅 없이 자금을 받을 수 있습니다.

**권장 완화 방법:** 베스팅 기간에 대한 확인의 주석 처리를 제거하십시오.

**Doryoku:**
[d16e369](https://github.com/Warlands-Nft/xbelo/commit/d16e36999cb5f00474fbdb2666ddd7f26a3a31c9)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 소각 비율의 가파른 단계적 급증

**설명:** `burn_pct`(소각 비율)는 정밀도 승수(multiplier)를 사용하지 않으며 초기 구성에서 최소 **2** 및 최대 **50**이 될 수 있습니다. `burn_pct`에 대한 계산은 다음과 같습니다:
```rust
            (state.min_burn_pct as u64)
                .saturating_sub(((over_min * burn_range) / time_range) as u64)
```
초기 구성으로 이러한 변수를 변경하면 다음 공식으로 계산을 근사할 수 있습니다:
```rust
    50 - 48 * ( (vest_duration - 10 days ) / 170 days )
```
이 공식은 대략 **3.5일** 간격으로 발생하는 가파른 단계적 점프로 구성됩니다.

따라서 소각될 토큰의 비율은 *10일* 동안 베스팅하는 사용자와 *13.4일* 동안 베스팅하는 사용자에 대해 동일합니다.

**권장 완화 방법:** 단계적 점프를 부드럽게 하기 위해 소각 비율 계산에 베이시스 포인트(bips) 또는 다른 고정밀 스케일러를 도입하십시오. 베이시스 포인트를 사용한 구현 예:
```diff
-        state.min_burn_pct = 50; // 50% @ min
-        state.max_burn_pct = 2; //  2% @ max
+        state.min_burn_pct = 5_000; // 50% in basis points (bips) @ min
+        state.max_burn_pct = 200; //  2% in basis points (bips) @ max
```
```diff
        let burn_amount = amount
            .saturating_mul(burn_pct)
-            .checked_div(100)
+            .checked_div(10_000) // %100 in basis points (bips)
            .ok_or(ErrorCode::ArithmeticError)?;
```
**Doryoku:**
[034eaac](https://github.com/Warlands-Nft/xbelo/commit/034eaac1863cd4e409b7777220a5e083bdbde030)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)

### 2단계 소유권 이전 고려

**설명:** `xbelo` 및 `clmm_lp_farming` 프로그램 모두에서 소유권은 `transfer_admin`이라는 하나의 함수 호출 내에서 이전됩니다.
이 패턴은 위험을 수반하며, 실수할 경우 `xbelo` 프로그램의 베스팅 매개변수 업데이트 기능을 잃고, 이 함수들이 관리자만 호출할 수 있다는 점을 고려할 때 `clmm_lp_farming` 프로그램의 일시 중지 기능을 잃게 될 수 있습니다.

**권장 완화 방법:** 더 안전한 소유권 이전을 위해 2단계 소유권 이전을 구현하는 것을 고려하십시오. 이를 위해서는 두 가지 함수가 필요합니다:
1- 다음 관리자 지정: 현재 관리자가 `new_admin`을 제안합니다.
2- 소유권 수락: `new_admin`이 소유권을 수락합니다.

**Doryoku:**
[41676f1](https://github.com/Warlands-Nft/xbelo/commit/41676f1ad0572f9b08fcd53c1d0a4a39b4fb9685) 및 [fad227e](https://github.com/Warlands-Nft/belo_clmm_lp_farming/commit/fad227e80c53e207a8836365ed0c8449a17f2992)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 불필요한 Rent 계정 사용

**설명:** `xbelo` 프로그램의 `Initialize` 지침 컨텍스트(Accounts struct) 내에서 시스템 계정 `rent`가 제공됩니다.

그러나 `rent` 계정은 초기화의 어떤 작업에도 활용되지 않으며 필요하지도 않습니다.

**권장 완화 방법:** `rent` 시스템 계정을 제거하는 것을 고려하십시오.

**Doryoku:**
[41676f1](https://github.com/Warlands-Nft/xbelo/commit/41676f1ad0572f9b08fcd53c1d0a4a39b4fb9685)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 베스팅 매개변수 업데이트가 충분히 제약되지 않음

**설명:** `xbelo` 프로그램의 `update_vest_params` 함수를 사용하면 관리자가 최소-최대 베스팅 기간과 최소-최대 소각 비율을 업데이트할 수 있습니다.
그러나 상호 연결된 변수 간의 잘못된 구성을 방지할 만큼 충분히 제약되지 않습니다. 따라서 함수의 기능을 깨뜨리는 값을 제공할 수 있습니다. 예를 들어 `vest_min`은 `vest_max`보다 클 수 있으며, 이는 기간 확인을 우회할 수 있는 유효한 `vest_duration`이 없으므로 `vesting`을 완전히 방지합니다.

**권장 완화 방법:** 다음 확인을 추가하는 것을 고려하십시오:
```rust
require!(new_vest_max > new_vest_min, ErrorCode::InvalidParams);
require!(new_min_burn_pct > new_max_burn_pct, ErrorCode::InvalidParams);
```
**Doryoku:**
[4ecb4a0](https://github.com/Warlands-Nft/xbelo/commit/4ecb4a089595eee98fff5be4f2be570023ca2511)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 이니셜라이저가 프론트러닝되어 프로그램 제어권을 얻을 수 있음

**설명:** `xbelo` 프로그램의 `initialize` 함수와 `clmm_lp_farm` 프로그램의 `initialize_farm` 함수는 배포 후 누구나 호출할 수 있습니다. 이러한 함수가 프로그램의 관리자를 설정하고 중요한 값을 초기화한다는 점을 고려할 때, 이를 악의적으로 실행하면 프로그램의 제어권이 호출자에게 넘어갈 수 있습니다.

**권장 완화 방법:** 프로그램의 배포 및 초기화가 동일한 트랜잭션에서 발생하도록 하거나 `Farm` 시드에 관리자 주소를 추가하십시오.

**Doryoku:**
[41676f1](https://github.com/Warlands-Nft/xbelo/commit/41676f1ad0572f9b08fcd53c1d0a4a39b4fb9685) 및 [fad227e](https://github.com/Warlands-Nft/belo_clmm_lp_farming/commit/fad227e80c53e207a8836365ed0c8449a17f2992)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `xbelo` 프로그램의 불필요한 유틸리티 함수

**설명:** `xgkhan` 프로그램에는 활용되지도 않고 필요하지도 않은 몇 가지 함수가 있습니다:
- `transfer_hook()`: `xGKHAN`은 확장이 있는 TOKEN2022 토큰이 아니라 일반 SPL 토큰입니다. 따라서 훅 함수가 필요하지 않습니다.
- `transfer_xgkhan()`: 항상 실패하는 이 사용자 정의 전송 함수는 어떠한 기능도 제공하지 않습니다. `xGKHAN` 토큰은 사용자에게 발행된 후 동결되므로 토큰 프로그램 수준에서 전송이 실패합니다. 이 함수는 토큰 전송 중에 활용되지 않습니다.
- `verify_non_transferable()`: SPL 토큰 계정에는 이미 `TokenAccount::is_frozen` 함수를 통한 내장 동결 상태 확인 기능이 있습니다. 이 두 함수 모두 동일한 상태를 확인하므로 토큰 프로그램 자체에서 이미 사용할 수 있는 기능을 제공할 필요가 없습니다.

**권장 완화 방법:** 앞서 언급한 함수와 지침 컨텍스트(Accounts struct)를 제거하는 것을 고려하십시오.

**Doryoku:**
[41676f1](https://github.com/Warlands-Nft/xbelo/commit/41676f1ad0572f9b08fcd53c1d0a4a39b4fb9685)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `Initialize` 및 `Vest` 지침의 중복 지침 속성

**설명:** `xbelo` 프로그램의 `Initialize` 및 `Vest` 계정 구조체에 각각 `#[instruction()]` 및 `#[instruction(amount: u64, vest_duration: i64)]` 속성이 제공됩니다.
이 속성은 계정 유효성 검사 중에 액세스해야 하는 지침 매개변수를 선언합니다.
첫 번째 것은 매개변수를 선언하지 않고 필요한 것도 없으며, 후자는 계정 유효성 검사 제약 조건에서 사용되지 않는 매개변수(`amount` 및 `vest_duration`)를 선언하므로 두 가지 모두 안전하게 제거할 수 있습니다.

**권장 완화 방법:** 필요하지 않으므로 `Initialize` 및 `Vest` 계정 구조체에서 `#[instruction(...)]` 속성을 제거하는 것을 고려하십시오.

**Doryoku:**
[41676f1](https://github.com/Warlands-Nft/xbelo/commit/41676f1ad0572f9b08fcd53c1d0a4a39b4fb9685)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 지침 로직 및 계정 컨텍스트에서 동일한 조건이 재검증됨

**설명:** `clmm_lp_farming` 프로그램에서 특정 검증이 두 번 수행됩니다: 한 번은 함수에서, 한 번은 계정 검증에서:
- `user_position_token_account.amount == 1`은 `stake_clmm_position` 함수와 `StakeCLMMPosition` 구조체 모두에서 확인됩니다.
- `!stake.claimed`는 `withdraw_clmm_position` 함수와 `WithdrawCLMMPosition` 구조체 모두에서 확인됩니다.

**권장 완화 방법:** 코드 중복을 방지하기 위해 이러한 검증 중 하나를 제거하는 것을 고려하십시오.

**Doryoku:**
[fad227e](https://github.com/Warlands-Nft/belo_clmm_lp_farming/commit/fad227e80c53e207a8836365ed0c8449a17f2992)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 관리자 이전에 대한 이벤트 방출 누락

**설명:** `clmm_lp_farming` 프로그램의 `transfer_admin` 함수는 프로그램의 중요한 상태를 변경하지만 이벤트를 방출하지 않습니다.

**권장 완화 방법:** 더 나은 오프체인 추적 및 투명성을 위해 이 함수에서 이벤트를 방출하는 것을 고려하십시오.

**Doryoku:**
[fad227e](https://github.com/Warlands-Nft/belo_clmm_lp_farming/commit/fad227e80c53e207a8836365ed0c8449a17f2992) 및 [41676f1](https://github.com/Warlands-Nft/xbelo/commit/41676f1ad0572f9b08fcd53c1d0a4a39b4fb9685)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### StakeCounter 구조체가 선언되었지만 활용되지 않음

**설명:** `clmm_lp_farming` 프로그램에는 `count` 필드를 포함하는 `StakeCounter` 구조체가 있습니다. 그러나 이 구조체는 프로그램 어디에서도 활용되지 않습니다:

```rust
#[account]
pub struct StakeCounter {
    pub count: u64,
}

impl StakeCounter {
    pub const LEN: usize = 8;
}
```

**권장 완화 방법:** `LEN` 구현과 함께 `StakeCounter` 구조체를 제거하는 것을 고려하십시오.

**Doryoku:**
[fad227e](https://github.com/Warlands-Nft/belo_clmm_lp_farming/commit/fad227e80c53e207a8836365ed0c8449a17f2992)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `total_position_staked` 업데이트의 안전하지 않은 산술 연산

**설명:** `clmm_lp_farming` 프로그램은 `stake_clmm_position` 및 `withdraw_clmm_position` 함수에서 각각 `total_positions_staked`를 **1**만큼 증가 및 감소시킵니다:
```rust
        ctx.accounts.farm.total_positions_staked += 1;
```
```rust
        ctx.accounts.farm.total_positions_staked -= 1;
```
Rust는 명시적으로 명시하지 않는 한(예: `Cargo.toml`에 `overflow-checks = true` 추가) 릴리스 모드에서 오버플로를 허용하고 래핑(wrap around)하므로 이러한 작업은 안전하지 않습니다.

**권장 완화 방법:** 오버플로를 방지하기 위해 `checked_add()` 및 `checked_sub` 함수를 사용하는 것을 고려하십시오.

**Doryoku:**
[fad227e](https://github.com/Warlands-Nft/belo_clmm_lp_farming/commit/fad227e80c53e207a8836365ed0c8449a17f2992)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 이벤트에서 활용되지 않는 `position_collection` 필드

**설명:** 사용자가 스테이킹 및 인출을 할 때 각각 `CLMMPositionStaked` 및 `CLMMPositionWithdrawn` 이벤트가 방출됩니다.
이 두 이벤트 모두 `position_collection`이라는 `Pubkey` 필드를 포함합니다.
그러나 두 작업 모두에서 해당 필드는 `Pubkey::default()`로 방출됩니다:
```rust
        emit!(CLMMPositionStaked {
            user: user_stake.user,
            position_mint: ctx.accounts.position_mint.key(),
            clmm_pool: ctx.accounts.farm.clmm_pool,
            duration_months,
            start_time: user_stake.start_ts,
            end_time: user_stake.start_ts + user_stake.duration,
            position_collection: Pubkey::default(),
        });
```
따라서 사용되지 않습니다.

**권장 완화 방법:** 오프체인 시스템에 기능이 없는 경우 `CLMMPositionStaked` 및 `CLMMPositionWithdrawn` 이벤트에서 `position_collection` 필드를 제거하는 것을 고려하십시오.

**Doryoku:**
[fad227e](https://github.com/Warlands-Nft/belo_clmm_lp_farming/commit/fad227e80c53e207a8836365ed0c8449a17f2992)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
