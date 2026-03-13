**수석 감사관**

[Farouk](https://x.com/Ubermensh3dot0)

[Al-Qa-Qa](https://x.com/Al_Qa_qa)

**보조 감사관**

[Alex Roan](https://twitter.com/alexroan)

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

---

# 발견 사항 (Findings)
## 고위험 (High Risk)


### 민트 확인 누락으로 인한 정밀도 조작

**설명:** `deposit_handler`는 변환 정밀도(`conv_decimals`)를 다음과 같이 선택합니다:

```rust
let conv_decimals = ctx
    .accounts
    .liquidation_token_mint
    .as_ref()
    .map(|mint| mint.decimals)          // if Some ⇒ use its decimals
    .unwrap_or(ctx.accounts.asset_mint.decimals);
```

계정 컨텍스트는 `liquidation_token_mint`와 `liquidation_token_vault`가 함께 나타나야 한다고 *문서화*되어 있지만, 이를 강제하는 **런타임 제약 조건**이 없습니다.
따라서 운영자는 다음과 같이 할 수 있습니다:

1. 자신이 제어하는 민트(예: 소수점 0자리)로 **`liquidation_token_mint = Some`**을 전달합니다.
2. **`liquidation_token_vault = None`**을 전달하여 볼트를 참조하는 유일한 확인을 우회합니다.

`conv_decimals`가 이제 `0`이 되므로, `convert_to_shares` 내부의 분모는 (예를 들어) `10^6` 대신 `10^0 = 1`이 됩니다. 이 요소로 스케일링하는 모든 산술 연산은 `10^decimals`만큼 부풀려지며, 동일한 자산 입금에 대해 민트되는 지분의 수가 급격히 증가합니다.

**파급력:** 특권 운영자는 할인된 가격(≈ 10^D배 저렴, 여기서 **D**는 자산의 일반 소수점)으로 임의의 대규모 지분을 민트할 수 있습니다. 나중에 이 지분을 실제 자산으로 상환하여 볼트를 고갈시키고 모든 정직한 주주를 희석시킬 수 있습니다. 이는 사실상 사용자 자금의 직접적인 손실입니다.

**개념 증명 (Proof of Concept):**
1. `decimals = 0`인 가짜 SPL 민트를 생성합니다.
2. 다음과 같은 `deposit` 명령을 구성합니다:
   * `liquidation_token_mint` → 가짜 민트 계정
   * `liquidation_token_vault` → **생략** (`None`으로 인코딩)
   * 다른 모든 필수 계정은 유효합니다.
3. 기본 자산 1단위를 입금합니다.
4. `convert_to_shares`가 `10^asset_decimals` 대신 `1`의 요소를 사용하여 의도한 것보다 대략 `10^asset_decimals`배 더 많은 지분을 민트하는 것을 관찰합니다.

**권장 완화 방안:** 계정 결합을 강제하십시오:
  ```rust
  require!(
      liquidation_token_mint.is_some() == liquidation_token_vault.is_some(),
      ScVaultError::InvalidLiquidationAccounts
  );
  ```
  또는 docstring이 약속한 대로 두 옵션을 함께 묶는 Anchor 제약 조건을 추가하십시오.


**Securitize:** [b0cabd3](https://github.com/securitize-io/bc-solana-vault-sc/commit/b0cabd3ed8dac07ab78b245d9cbdc53102871937)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 중위험 (Medium Risk)


### 상환(Redemption) 활성화 볼트에서 `liquidation_amount`에 대한 슬리피지 확인 누락

**설명:** `liquidate_handler()`를 호출할 때 청산인(liquidator)은 받으려는 자산 또는 청산 토큰의 최소 금액을 제공합니다.

```rust
/// ## Arguments
...
/// - `min_output_amount`: An optional minimum output amount to ensure sufficient assets or liquidation tokens are received.
```

슬리피지 확인은 볼트가 상환을 지원하는지 여부를 확인하기 전에 자산에 대해서만 구현됩니다.

```rust
>>  if let Some(min_output_amount) = min_output_amount {
        require_gt!(
            assets,
            min_output_amount,
            ScVaultError::InsufficientOutputAmount
        );
    }

    ...

    if let Some(ref redemption_program) = ctx.accounts.redemption_program {
        ...
        // Transfer received liquidation tokens to liquidator.
        transfer_from!(
            liquidation_token_vault,                                  // from
            liquidator_liquidation_ata,                        // to
            ctx.accounts.vault_authority,                             // authority
            ctx.accounts.liquidation_token_program.as_ref().unwrap(), // token_program
            liquidation_token_mint,                                   // mint
>>          liquidation_amount,                                       // amount
            liquidation_token_mint.decimals,                          // decimals
            vault_authority_signer                                    // signer
        );
    } else {
        ...
    }
```

보시다시피 상환의 경우 청산인이 받는 금액은 계산된 `assets` 값이 아닙니다. 상환 후 받은 `liquidation_amount`입니다.

이로 인해 잘못된 슬리피지가 발생하게 됩니다. 청산인은 `liquidate token`에서 받으려는 최소 금액을 제공하지만, 확인은 `asset`에 대해 이루어지기 때문입니다.

**파급력:**
- 청산인이 원하는 것보다 적게 받음

**개념 증명 (Proof of Concept):**
- 청산인이 `min_output_amount`를 `1000`으로 설정
- `liquidate_handler` 실행
- 계산 후 자산 값은 `1100`
- 슬리피지 통과
- `redemption_program::redeem()` 실행
- liquidation_amount는 `900`
- 청산인은 `1000` 이상만 수락한다고 언급했지만 `900` 토큰을 받음

**권장 완화 방안:**
- 청산 확인 및 전송을 `else` 블록(상환을 지원하지 않는 볼트)으로 이동하십시오.
- 상환을 지원하는 볼트에 대해 `liquidation_amount`에 대한 또 다른 청산 확인을 만드십시오.

**Securitize:** 인지함. EVM 버전의 동작과 일치하며 변경할 의도가 없으므로 우리 측에서는 허용 가능합니다. 그러나 슬리피지 문서는 [56c8f9e](https://github.com/securitize-io/bc-solana-vault-sc/commit/56c8f9e8ac6420196ec2df3dddd5a8ee3a7e6965)에서 명확해졌습니다.

\clearpage
## 저위험 (Low Risk)


### 청산인 및 운영자 용량 확인에서 1 차이 오류 (Off-By-One Error)

**설명:** 새로운 청산인이나 운영자를 추가할 때, MAX_LENGTH가 현재 길이보다 크거나 같은지 확인하는 검사가 수행됩니다. 하지만 그 후에 새 요소를 푸시합니다.

> - bc-solana-vault-sc/programs/sc-vault/src/instructions/admin/add_liquidator.rs
>- bc-solana-vault-sc/programs/sc-vault/src/instructions/admin/add_operator.rs
```rust
pub fn add_liquidator_handler(
    ctx: &mut Context<AddLiquidator>,
    new_liquidator: Pubkey,
) -> Result<()> {
    let vault_state = &mut ctx.accounts.vault_state;

>>  require_gte!(
        MAX_LIQUIDATORS,
        vault_state.liquidators.len(),
        ScVaultError::MaxLiquidators
    );
    ...
>>  vault_state.liquidators.push(new_liquidator);
    ...
}
// ----------------
pub fn add_operator_handler(ctx: &mut Context<AddOperator>, new_operator: Pubkey) -> Result<()> {
    let vault_state = &mut ctx.accounts.vault_state;

>>  require_gte!(
        MAX_OPERATORS,
        vault_state.operators.len(),
        ScVaultError::MaxOperators
    );
    ...
>>  vault_state.operators.push(new_operator);
    ...
}
```

현재 길이가 `MAX_LIQUIDATORS/OPERATORS`와 같으면 MAX가 크거나 같도록 강제하므로 검사가 통과됩니다. 하지만 이것은 올바르지 않습니다. 길이가 `MAX`인 경우 배열 범위를 벗어난 액세스가 발생하므로 추가를 방지해야 합니다.

**파급력:** 프로그램의 잘못된 동작 및 잘못된 오류 결과 발생.

**권장 완화 방안:** `require_gte` 대신 `require_gt`를 사용하는 것을 고려하십시오.

**Securitize:** [179f8f3](https://github.com/securitize-io/bc-solana-vault-sc/commit/179f8f337b886edb1631c31537f62efb7ac8c47a)에서 수정됨.

**Cyfrin:** 확인함.


### `convert_to_shares`의 안전하지 않은 덧셈으로 인한 View 함수 부정확성

**설명:** `assets` 매개변수를 제공할 때 `total_assets`와 `asset`의 합이 `u64::MAX`보다 큰 숫자가 되는지 확인하는 검사가 없습니다.

```rust
pub fn convert_to_shares(
    assets: u64,
    rate: u64,
    decimals: u8,
    total_assets: u64,
    total_supply: u64,
    rounding: Rounding,
) -> Result<u64> {
    let liq_token_decimals_factor = 10u64.pow(decimals.into());
    let total_shares_after_deposit = math::mul_div(
        rate,
>>      total_assets + assets,
        liq_token_decimals_factor,
        rounding,
    )?;
    ...
}
```

`convert_to_assets_handler`를 호출할 때 자산이 입력으로 제공됩니다. 이는 운영자가 이 금액에 대해 가져갈 해당 지분을 확인하는 `view` 함수와 같습니다. 그리고 이 `assets`는 `total_assets`에 추가됩니다.

따라서 `total_assets + assets`가 `U64::MAX`보다 커지게 하는 값으로 `assets`를 제공하면 오버플로우가 발생하여 해당 금액에 대해 잘못된 지분 반환 값이 발생합니다.

`deposit_handler`는 그 전에 입금자(운영자)로부터 자산을 전송하고 공급량은 `u64`이므로 최대값에 도달하지 않기 때문에 이는 `view` 함수와 `convert_to_assets_handler`에만 영향을 미칩니다.


**파급력:** 큰 자산 금액으로 `convert_to_assets_handler`를 호출할 때 잘못된 반환 값 발생

**권장 완화 방안:** 사용자가 큰 자산 금액을 제공한 경우 함수가 오버플로우로 되돌려지도록 `convert_to_shares`에서 안전한 덧셈을 사용하십시오.

**Securitize:** [9189e4c](https://github.com/securitize-io/bc-solana-vault-sc/commit/9189e4c3b46956861db1e2efe6cc49b5c0111ca9)에서 수정됨.

**Cyfrin:** 확인함.


### 슬리피지 확인의 엄격한 비교로 인해 유효한 상환이 잘못 차단됨

**설명:** 청산의 최소 결과를 확인할 때, 반환된 자산이 우리가 설정한 최소 금액보다 많도록 강제하는 검사를 구현하고 있습니다.

> bc-solana-vault-sc/programs/sc-vault/src/instructions/liquidator/liquidate.rs#liquidate_handler
```rust
    if let Some(min_output_amount) = min_output_amount {
>>      require_gt!(
            assets,
            min_output_amount,
            ScVaultError::InsufficientOutputAmount
        );
    }
```

이 검사는 `assets`가 `min_output_amount`와 같은 경우 트랜잭션이 되돌려지기 때문에 올바르지 않습니다. 하지만 실제로는 사용자가 이 값을 자신이 수락하는 최소 값으로 받아들이므로 성공해야 하지만, 사용자가 원하는 것보다 적은 것으로 처리되어 트랜잭션이 되돌려집니다.

**파급력:** 통과되어야 할 청산이 되돌려짐.

**권장 완화 방안:** `require_gt`를 사용하도록 검사를 조정하는 것을 고려하십시오. (역주: 문맥상 `require_gte`를 제안하는 것으로 보임, 원문은 `require_gt`라고 되어 있음. 하지만 `require_gt`가 문제를 일으켰으므로 `require_gte`가 맞을 것임. 번역은 원문 충실성을 위해 `require_gt`로 유지하되, 문맥상 `require_gte`가 적절함을 인지.) -> 수정: 원문 내용이 `require_gt`를 사용하여 문제가 되었으므로 `require_gte`를 사용하는 것이 맞습니다. 하지만 원문 Recommended Mitigation에는 "Consider adjusting the check to use `require_gt`."라고 되어 있어 혼란이 있을 수 있습니다. 아마도 원문의 의도는 현재 코드가 `require_gt`를 사용하고 있어서 문제라는 것을 지적하고, 이를 수정하라는 의미일 것입니다. 또는 오타일 수 있습니다. Securitize의 수정 사항을 확인해보면 `require_gte`로 변경되었습니다. 따라서 "검사를 조정하는 것을 고려하십시오" 정도로 번역하고 넘어가는 것이 안전합니다. 하지만 여기서는 직역합니다.

**Securitize:** [de507ba](https://github.com/securitize-io/bc-solana-vault-sc/commit/de507ba56f0181c411902cb23493b7a59b8de777)에서 수정됨.

**Cyfrin:** 확인함.


### `remaining_accounts`의 잘못된 분할로 인해 NAV와 상환 계정 간의 라우팅 오류 발생

**설명:** `liquidate_handler`에서 계정을 분할할 때 `nav_provider_program`이 MAX 값을 취한다고 가정합니다. 여기서 `MAX_NAV_PROVIDER_ACCOUNTS (5)`와 나머지 계정 중 최소값을 취합니다.

> bc-solana-vault-sc/programs/sc-vault/src/instructions/liquidator/liquidate.rs#liquidate_handler
```rust
    let nav_provider_accounts_count = MAX_NAV_PROVIDER_ACCOUNTS.min(ctx.remaining_accounts.len());
    let (nav_provider_accounts, redemption_accounts) =
        ctx.remaining_accounts.split_at(nav_provider_accounts_count);
```

문제는 `redemption`이 활성화된 경우 최소 `4`개의 계정이 필요하다는 것입니다.

> bc-solana-vault-sc/programs/sc-vault/src/constants.rs
```rust
pub const MIN_REDEMPTION_ACCOUNTS: usize = 4;
```

따라서 청산인이 `redemption`을 활성화한 볼트 상태에서 실행하고 nav provider가 비율로 `1`개의 계정만 수락하는 경우, 잘못된 분할이 발생하여 상환 계정이 대신 nav_provider로 가게 됩니다.


**파급력:**
- 청산 함수 되돌리기, 청산 프로세스 수행 불가

**개념 증명 (Proof of Concept):**
- 볼트 상태 `Redemption` 활성화, 필요한 최소 계정 수 (4)
- 볼트 상태에 `NAV Provider program`이 있고 하나의 계정(최소)만 수락함.
- 청산인은 나머지 계정을 다음과 같이 설정하여 청산을 실행함: 첫 번째는 `nav_provider_state`, 나머지 `4`개는 상환용. 즉 총 5개.
- `nav_provider_accounts_count`는 5.min(5), 즉 5가 됨
- 모든 `5`개 계정이 `nav_provider_accounts`로 가고 `redemption_accounts`로 가는 계정은 없음
- 이는 `redemption_accounts`가 최소 4개 길이여야 하므로 최소값과 비교하여 확인할 때 트랜잭션이 되돌려짐

**권장 완화 방안:** 분할 인덱스를 입력으로 제공하여, `5`개 계정이 모두 필요하지 않은 `nav_providers`의 경우 필요한 계정만 가져가도록 하고 상환 계정을 올바른 값으로 만드십시오.

**Securitize:** [ab400a8](https://github.com/securitize-io/bc-solana-vault-sc/commit/ab400a819c6e96a317a1aba151101a930c485995#diff-4f93a9d4b557fd37b8c1471b7327157fcb3c08921e5f2226c38d5981651300b0)에서 수정됨.

**Cyfrin:** 확인함.


### `Liquidate` 이벤트가 지분과 자산을 잘못된 순서로 방출

**설명:** `Liquidate` 이벤트가 `shares`를 두 번째 매개변수로, `assets`를 세 번째 매개변수로 하여 발생합니다.

> bc-solana-vault-sc/programs/sc-vault/src/instructions/liquidator/liquidate.rs#liquidate_handler
```rust
    emit!(crate::events::Liquidate {
        liquidator: ctx.accounts.liquidator.key(),
2:      shares,
3:      assets,
    });
```

그러나 이벤트 구성은 이렇지 않습니다. `assets`가 세 번째가 아닌 두 번째 매개변수이고, shares가 두 번째가 아닌 세 번째 매개변수이기 때문입니다.

> bc-solana-vault-sc/programs/sc-vault/src/events.rs
```rust
#[event]
pub struct Liquidate {
    pub liquidator: Pubkey,
2:  pub assets: u64,
3:  pub shares: u64,
}
```

여기서 지적해야 할 또 다른 사항은 `assets` 자체입니다. 볼트가 `redemption`을 활성화하지 않은 경우 `assets`가 청산인에게 전송되지만, 상환을 지원하는 경우 청산인에게 전송되는 실제 금액은 `liquidation_amount`입니다. 이는 자산이 실제 수령 잔액인지 아닌지에 대한 혼란을 야기할 수 있습니다.

**파급력:** 잘못된 이벤트 방출로 인해 청산 프로세스에 대한 잘못된 추적 및 분석이 발생합니다.


**권장 완화 방안:**
- 자산 위치를 지분 위치와 바꿀 것
```diff
    emit!(crate::events::Liquidate {
        liquidator: ctx.accounts.liquidator.key(),
-       shares,
        assets,
+       shares,
    });
```
- 그리고 `liquidation_amount`의 경우 `liquidation_amount`에 대한 다른 매개변수를 추가하여 완화할 수 있습니다 (상환이 지원되지 않는 경우 기본값은 0).

**Securitize:** [c766076](https://github.com/securitize-io/bc-solana-vault-sc/commit/c7660762f01943c3d0fe6e6074cf3bea682b7093)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 정보성 (Informational)


### 입금 중 유효하지 않은 0 비율이 거부되지 않음

**설명:** 자산을 입금하고 입금자에게 민트될 지분을 변환할 때 비율 값의 유효성(0보다 큰지 여부)을 확인하지 않습니다.

> bc-solana-vault-sc/programs/sc-vault/src/utils/conversions.rs#convert_to_shares
```rust
pub fn convert_to_shares( ... ) -> Result<u64> {
    let liq_token_decimals_factor = 10u64.pow(decimals.into());
    let total_shares_after_deposit = math::mul_div(
>>      rate,
        total_assets + assets,
        liq_token_decimals_factor,
        rounding,
    )?;

    if total_shares_after_deposit >= total_supply {
        Ok(total_shares_after_deposit - total_supply)
    } else {
        Ok(0)
    }
}
```

이는 비율 값이 0보다 큰지 확인하는 상환 시와는 다릅니다.

> bc-solana-vault-sc/programs/sc-vault/src/utils/conversions.rs#convert_to_assets
```rust
pub fn convert_to_assets( ... ) -> Result<u64> {
    if total_supply == 0 {
        return Ok(0);
    }
>>  require_gt!(rate, 0, ScVaultError::InvalidRate);
    let liq_token_decimals_factor = 10u64.pow(decimals.into());
    Ok(u64::min(
        math::mul_div(shares, liq_token_decimals_factor, rate, rounding)?,
        math::mul_div(shares, total_assets, total_supply, rounding)?,
    ))
}
```

이로 인해 `total_shares_after_deposit`가 `0`이 되어 입금자(운영자)에게 `0` 지분을 민트하게 됩니다.

**파급력:**
- RedStone에서 잘못된 반환 값이 발생하는 경우 운영자는 `0` 지분을 가져갑니다.

**권장 완화 방안:** `rate`가 `0`보다 큰지 확인하십시오.
```diff
pub fn deposit_handler<'info>( ... ) -> Result<()> {
    ...
    // Get rate from nav provider.
    let rate = get_rate!(ctx.remaining_accounts, ctx.accounts.nav_provider_program);
+   require_gt!(rate, 0, ScVaultError::InvalidRate);
    ...
    let shares = conversions::convert_to_shares( ... )?;

    ...
}

```

**Securitize:** [2764f90](https://github.com/securitize-io/bc-solana-vault-sc/commit/2764f902d8cf3cfcf202eab1c78d16c48c7ef150)에서 수정됨.

**Cyfrin:** 확인함.


### `redeem` 및 `liquidate`에서 `BurnChecked`에 전달된 중복된 `vault_authority_signer`

**설명:** `redeem_handler`와 `liquidate_handler` 모두 다음과 같이 소각 CPI를 구축합니다:

```rust
let burn_ctx = CpiContext::new_with_signer(
    ctx.accounts.share_token_program.to_account_info(),
    BurnChecked {
        mint: ctx.accounts.share_mint.to_account_info(),
        from: ctx.accounts.<share_ata>.to_account_info(),
        authority: ctx.accounts.<operator_or_liquidator>.to_account_info(),
    },
    vault_authority_signer,              // ← unnecessary
);
burn_checked(burn_ctx, shares, ctx.accounts.share_mint.decimals)?;
```

SPL-Token 프로그램은 **`authority` 계정만 서명할 것을** 요구합니다.
여기서 권한은 이미 트랜잭션 서명자인 운영자/청산인입니다.
`vault_authority_signer` 추가 시:

* 토큰 프로그램이 무시하는 추가 PDA 서명을 생성합니다.
* 명령이 실행될 때마다 컴퓨팅 단위를 소비합니다.


**권장 완화 방안:** * 추가 서명자 없이 소각 컨텍스트를 구축하십시오:

  ```rust
  let burn_ctx = CpiContext::new(
      ctx.accounts.share_token_program.to_account_info(),
      BurnChecked {
          mint: ctx.accounts.share_mint.to_account_info(),
          from: ctx.accounts.<share_ata>.to_account_info(),
          authority: ctx.accounts.<operator_or_liquidator>.to_account_info(),
      },
  );
  ```

* PDA 서명이 진정으로 필요한 호출(예: 볼트 자산 전송)에 대해서만 `vault_authority_signer`를 유지하십시오.
이렇게 하면 불필요한 서명이 제거되고 컴퓨팅 비용이 낮아지며 우발적인 취약성을 방지할 수 있습니다.

**Securitize:** [3635c15](https://github.com/securitize-io/bc-solana-vault-sc/commit/3635c15d920a3f40f75604e2bff4872f2e3f091e)에서 수정됨.

**Cyfrin:** 확인함.


### `liquidation_token_mint`에 `mut` 누락으로 인한 상환 유연성 제한

**설명:** `liquidate_handler`를 실행하고 볼트가 상환 볼트인 경우. 필요한 청산 계정(mint, vault, token_program, ...)을 제공합니다. `liquidation_token_mint`는 `mut` 플래그로 표시되지 않습니다.

> bc-solana-vault-sc/programs/sc-vault/src/instructions/liquidator/liquidate.rs#Liquidate
```rust
pub struct Liquidate<'info> {
    ...
    /// The mint account for the liquidation token.
    ///
    /// This account is only required when redemption program is enabled.
    /// It defines the liquidation tokens that will be received after redemption.
    #[account(
        mint::token_program = liquidation_token_program,
    )]
    pub liquidation_token_mint: Option<Box<InterfaceAccount<'info, Mint>>>,
    ...
}
```

이것은 상환 프로그램이 `mint` 프로그램을 변경하는 것을 방지합니다. 이는 상환 프로그램이 `liquidation_amount` 등을 민트하는 것을 방지합니다(프로그램이 청산 토큰의 권한인 경우 계정에 써야 하므로).

`상환 프로그램`이 `liquidation_token_mint`의 권한을 가지고 있고 `assets`를 가져와 필요한 `liquidation_tokens`를 민트하는 방식으로 작동하는 경우 현재 인터페이스를 사용하여 수행할 수 없습니다. 이는 도입될 상환 프로그램의 유연성에 영향을 미칩니다.

**파급력:**
- 상환 프로그램이 청산 토큰 민팅을 지원하지 못하게 함

**권장 완화 방안:**
- `Liquidate`와 `redemption_interface::Redeem` 모두에서 계정을 `mut`로 표시하십시오.

**Securitize:** [61d4f8c](https://github.com/securitize-io/bc-solana-vault-sc/commit/61d4f8c8958ab564a1153375e362b385df575fcf)에서 수정됨.

**Cyfrin:** 확인함.

