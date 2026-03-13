**Lead Auditors**

[Farouk](https://x.com/Ubermensh3dot0)

[JesJupyter](https://x.com/jesjupyter)

[Ctrus](https://x.com/ctrusonchain)
**Assisting Auditors**



---

# 발견 사항 (Findings)
## 치명적 위험 (Critical Risk)


### 투자자가 아무 대가 없이 DsToken을 민팅할 수 있음

**설명:** swap 로직에서는 투자자로부터 유동성을 받아 custodian의 liquidity token 계정으로 전송한 뒤, 그에 상응하는 ds/spl 토큰을 투자자에게 민팅합니다. `swap-ds-token`과 `subscribe-ds-token`에서는 `asset_transfer_hook_accounts`, `asset_provider_accounts`, `nav_provider_accounts`를 모두 사용자가 제공한 remaining accounts에서 가져오며, 실제로 구현된 검사는 remaining accounts 길이가 기대 길이보다 큰지만 확인하는 수준입니다. 투자자는 `nav-provider-accounts`를 검증 때문에 위조할 수 없고, `asset-transfer-hook-accounts`도 위조할 수 없는데 그렇지 않으면 transfer가 실패해 전체 트랜잭션이 실패하기 때문입니다. 문제는 `asset-provider-accounts`입니다. 각 함수에서 필요한 검증을 마친 뒤 이 계정들은 `swap-process`로 전달됩니다. `swap-process`는 사용자로부터 유동성을 vault로 옮기고, on-ramp의 asset provider의 `supply-to` 메서드를 통해 ds/spl 토큰을 발행합니다. 이때 asset provider가 `MintingAssetProvider`이고 token type이 `DsTokens`라면, on-ramp의 서명 권한을 넘겨주면서(우리 프로그램이 seed로 서명함) `rwa-rbac` 프로그램의 `issue-tokens` 함수로 CPI를 수행합니다.

```rust
                rwa_rbac::cpi::issue_tokens(
                    CpiContext::new_with_signer(
                        rwa_rbac_program.to_account_info(),
                        rwa_rbac::cpi::accounts::IssueTokens {
```
하지만 `rwa_rbac` 프로그램 계정은 remaining accounts에서 가져온 것이며, 의도한 `rwa-rbac` 프로그램이 맞는지 key/address로 검증하지 않았습니다.

```rust
                let rwa_rbac_program = &additional_accounts[0];
```

투자자는 remaining accounts를 통해 자신의 악성 프로그램 계정을 `rwa-rbac`로 넘길 수 있습니다. 이 악성 `rwa-rbac` 프로그램은 동일한 이름의 `issue-tokens` 명령을 구현할 수 있고, 그 내부 로직은 전적으로 공격자 손에 있습니다. 예를 들어 전달된 signer 권한을 악용해 다음과 같은 공격을 수행할 수 있습니다.
- on-ramp의 asset token vault와 liquidity token vault는 모두 이 authority가 소유하므로, authority가 이 트랜잭션에 서명한 상태에서 악성 사용자는 `issue-tokens` 안에서 token-22의 transfer checked를 호출해 두 vault를 자신의 토큰 계정으로 모두 빼낼 수 있습니다.
호출 흐름: 우리 프로그램 -> 가짜 `rwa-rbac` 프로그램(공격자 제어) -> `token22`(`transfer` 실행)

- 더 심각한 공격은 가짜 `rwa-rbac`의 `issue-tokens`를 다음처럼 구현하는 것입니다.
우리 프로그램 -> 가짜 `rwa-rbac` 프로그램 -> 실제 `rwa-rbac` 프로그램 -> `token22`(`mint` 실행)
공격자는 우리 프로그램에 `rwa-rbac` 프로그램으로 자신의 가짜 프로그램을 넘기고, 우리는 그 가짜 프로그램으로 CPI를 하면서 PDA signer 권한을 함께 넘깁니다. 이후 공격자의 프로그램은 실제 `rwa-rbac`가 기대하는 형식의 `cpi_data`를 위조하면서 토큰 `amount`를 매우 크게 부풀린 뒤, 다시 실제 `rwa-rbac` 프로그램으로 CPI를 보내게 됩니다. 그러면 실제 `rwa-rbac`는 그 부풀려진 수량만큼 토큰을 투자자에게 발행합니다. 이 방식으로 투자자는 사실상 무한한 수의 dsToken을 민팅할 수 있습니다.

**영향:** 투자자는 자신에게 정당하게 주어져야 할 양이 아니라, 무한하거나 임의의 수량의 DsToken을 민팅할 수 있습니다.

**개념 증명 (Proof of Concept):** 다음 시나리오를 가정합니다.
- 악성 투자자가 다음 설정을 가진 on-ramp를 찾습니다.
[asset_provider = MintingAssetProvider]
[asset_token_type = DsToken]
[investor_subscription_enabled = true]
- 투자자는 `issue_tokens` 명령을 동일한 instruction discriminator로 구현한 악성 프로그램을 배포하고, 그 구현은 다음과 같습니다.
1. `securitize-on-ramp`로부터 `on_ramp_authority` signer와 함께 CPI 호출을 받음
2. 원래 전달된 `amount` 파라미터를 무시함
3. `amount = u64::MAX`(또는 임의의 큰 값)로 새 CPI 데이터를 구성함
4. 이 부풀려진 수량으로 실제 `rwa_rbac` 프로그램의 `issue-tokens`를 호출함
- 투자자는 최소한의 정상 `liquidity_amount`(예: 10 USDC)로 `swap-ds-tokens`를 호출하고 `asset_provider_accounts[0]`에 자신의 악성 프로그램 주소를 넣습니다.
- 투자자의 10 USDC는 vault로 전송되고, `swap_process`는 `MintingAssetProvider::supply_to()`를 호출합니다. 이때 실제 `rwa_rbac` 대신 투자자의 악성 프로그램으로 CPI가 실행되고, 그 악성 프로그램이 다시 실제 `rwa_rbac::issue_tokens`를 `amount = u64::MAX`로 호출합니다. 최종적으로 실제 `rwa_rbac`는 `u64::MAX` 수량의 토큰을 투자자의 토큰 계정으로 민팅합니다.
- 투자자는 10 USDC와 트랜잭션 수수료만으로 약 `u64::MAX` 개의 DS 토큰을 받습니다.

**권장 완화 조치:** 사용자가 제공한 `rwa-rbac` 프로그램 ID가 실제/정상 `rwa-rbac` 프로그램인지 검증하십시오.
```rust
let rwa_rbac_program = &additional_accounts[0];
require_keys_eq!(
    rwa_rbac_program.key(),
    rwa_rbac::ID,  // import한 crate에 선언된 프로그램 ID를 사용
    SecuritizeOnRampError::InvalidProgram
);
```
**Securitize:** [07f8e44](https://github.com/securitize-io/bc-solana-on-off-ramp-sc/commit/07f8e446b5e79acdc0b50f31e355b38259692971)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 중간 위험 (Medium Risk)


### DS Token 발행 CPI에서 `payer`를 검증하지 않아 프로토콜이나 registrar가 생성 수수료를 대신 낼 수 있음

**설명:** `MintingAssetProvider::supply_to`(DsToken 분기)에서 `rwa_rbac::issue_tokens` CPI용 `payer` 계정은 `additional_accounts[1]`에서 직접 가져오며, 의도된 수수료 부담 주체인지 전혀 검증하지 않습니다.

```rust
                let rwa_rbac_program = &additional_accounts[0];
                let payer: &_ = &additional_accounts[1];
                let controller_authority = &additional_accounts[2];
                // ... other accounts ...
                rwa_rbac::cpi::issue_tokens(
                    CpiContext::new_with_signer(
                        rwa_rbac_program.to_account_info(),
                        rwa_rbac::cpi::accounts::IssueTokens {
                            payer: payer.to_account_info(),
                            user: authority.to_account_info(),
                            // ...
                        },
                        supply_signer,
                    ),
                    // ...
                )?
```


이로 인해 다음이 가능해집니다.

1. **PDA drain:** `swap_ds_token`(및 operator swap)에서 호출자는 `payer`를 `on_ramp_authority` PDA로 설정할 수 있습니다. 이 PDA는 이미 `supply_signer`를 통해 서명하고 있으므로, 여기에 lamport가 조금이라도 쌓여 있으면(실수로 송금되었거나 미래 기능 변경으로 인해) 토큰 계정 생성 같은 CPI 수수료 지불에 그 lamport가 사용될 수 있고, 결과적으로 프로토콜 자금이 빠져나갑니다.
2. **투자자 대신 registrar가 지불:** 프로토콜은 **payer가 투자자인지 검증하지 않습니다**. `subscribe_ds_token`에서는 사용자가 `remaining_accounts`를 통해 `asset_provider_accounts`를 직접 넘깁니다. 사용자는 `asset_provider_accounts[1]`(payer)을 `registrar_authority`로 설정할 수 있습니다. `registrar_authority`는 해당 명령에서 `Signer`이므로, `rwa_rbac::issue_tokens` CPI는 이를 payer로 받아들이고, 결과적으로 **프로토콜(registrar)** 이 투자자의 토큰 계정 생성 비용(`init_if_needed` 등)을 대신 부담하게 됩니다. 이는 **사용자(투자자)가 자신의 계정 생성 비용을 내야 한다**는 설계 의도와 어긋납니다.


하위 `rwa_rbac::issue_tokens` 로직은 보통 계정 생성에 `payer`를 사용합니다(예: `payer = payer`인 `init_if_needed`). 따라서 `payer`에 들어간 주체가 실제로 rent/생성 비용을 부담하게 됩니다. 검증이 없으면 그 대상이 투자자가 아니라 `on_ramp PDA`나 `registrar`가 될 수 있습니다.
```rust
#[account(
        init_if_needed,
        payer = payer,
        associated_token::token_program = token_program,
        associated_token::mint = asset_mint,
        associated_token::authority = to,
    )]
    pub token_account: Box<InterfaceAccount<'info, TokenAccount>>,
```

**영향:**
- **On-ramp authority PDA:** 이 PDA가 lamport를 보유하게 되는 순간(실수 송금, 향후 기능 변경 등), 아무 사용자가 `swap_ds_token`이나 operator swap에서 `payer`를 PDA로 지정해 그 lamport를 `issue_tokens` 수수료(예: 토큰 계정 생성비)로 소진시킬 수 있습니다. 현재 PDA가 잔액을 보유하지 않는다면 영향이 제한적이지만, 설계/사용 방식이 변할 경우 실제 리스크가 됩니다.
- **Registrar가 payer가 되는 경우:** payer 검증이 없기 때문에 사용자는 `subscribe_ds_token`에서 `registrar_authority`를 payer로 설정해 투자자 토큰 계정 생성 비용을 프로토콜이 대신 부담하게 만들 수 있습니다. 이는 프로토콜에 직접적인 경제적 비용을 발생시키며, 의도된 "사용자 부담" 설계와 정면으로 충돌합니다.


**권장 완화 조치:** `payer`가 `investor`인지 보장해야 합니다.

**Securitize:** [4382392](https://github.com/securitize-io/bc-solana-on-off-ramp-sc/commit/438239289ef5427a4f5158c92b2c477193e92bf2)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 낮은 위험 (Low Risk)


### 등록 계정이 생략되면 지갑 신원 검증 없이 DS Token 구독이 진행될 수 있음

**설명:** `subscribe_ds_token` 명령은 선택적 등록/신원 계정들이 전부 `None`인 경로를 허용합니다. 이 분기에서는 등록 CPI도 발생하지 않고, swap이 실행되기 전 온체인 `wallet_identity` 제약도 강제되지 않습니다. `swap_ds_token`은 항상 PDA 제약을 통해 `wallet_identity`를 검증하지만, 이 흐름에서는 asset provider나 token이 별도로 신원 검증을 강제하지 않는다면 등록되지 않은 지갑으로도 DS 토큰이 mint/transfer될 수 있습니다.

```rust
(None, None, None, None, None, None, None, None, None) => {
    require_eq!(
        register_investor_cpi_data_len,
        0,
        SecuritizeOnRampError::InvalidRegisterInvestorConfig
    );
    require_eq!(
        add_levels_cpi_data_len,
        0,
        SecuritizeOnRampError::InvalidAddLevelsConfig
    );

    // All optional accounts for registering investor and adding levels are None
    require!(
        ctx.accounts.identity_metadata_registry_program.is_none()
            && ctx.accounts.investor.is_none()
            && ctx.accounts.wallet_identity.is_none()
            && ctx.accounts.policy_engine_program.is_none()
            && ctx.accounts.tracker_account.is_none()
            && ctx
                .accounts
                .event_authority_identity_metadata_registry
                .is_none()
            && ctx.accounts.policy_engine.is_none(),
        SecuritizeOnRampError::InvalidRegisterInvestorConfig
    )
}
```

- `swap_ds_token`
```rust
/// Check that the investor wallet is registered
#[account(
    constraint = wallet_identity.wallet == investor_wallet.key()
        @ SecuritizeOnRampError::Forbidden,
    seeds = [investor_wallet.key().as_ref(), asset_mint.key().as_ref()],
    seeds::program = identity_registry::ID,
    bump,
)]
pub wallet_identity: Box<Account<'info, identity_registry::WalletIdentity>>,
```

**영향:** DS token 강제가 asset provider나 token hook 수준에서 보장되지 않는다면, 등록되지 않은 지갑이 DS 토큰을 받을 수 있어 컴플라이언스/신원 요구사항을 위반하게 됩니다.

**권장 완화 조치:** `subscribe_ds_token`에서 등록을 수행하지 않는 경우에도 `wallet_identity` 검증을 요구하거나, DS token 구독에는 등록 CPI 계정을 반드시 제공하도록 강제하십시오.

**Securitize:** [8d87df5](https://github.com/securitize-io/bc-solana-on-off-ramp-sc/commit/8d87df515949a722d05860bd8f1fa23a1410c901)에서 수정됨.

**Cyfrin:** 확인함.


### Operator swap 서명이 재사용 가능하며 특정 on-ramp나 mint에 바인딩되지 않음

**설명:** operator swap 서명 페이로드(`SwapSplTokenMessage`)에는 nonce가 없고 `on_ramp_state`, `asset_mint`, `liquidity_mint` 식별자도 포함되어 있지 않습니다. 그 결과 유효한 서명은 만료 시각 이전까지 재사용될 수 있고, 다른 on-ramp 인스턴스에도 재활용될 수 있습니다. operator를 신뢰한다는 가정 때문에 심각도는 낮게 분류되지만, operator 키나 서명된 페이로드가 재사용되거나 유출될 경우 피해 범위를 넓힙니다.
```rust
pub fn validate_investor_signature<'info>(
    ixs_account: &AccountInfo<'info>,
    expected_message: SwapSplTokenMessage,
) -> Result<()> {
    let ix_account = sysvar_instructions::get_instruction_relative(-1, ixs_account)?;

    require_gte!(
        expected_message.deadline,
        Clock::get()?.unix_timestamp,
        SecuritizeOnRampError::ExpiredSignature
    );

    utils::ed25519::validate_ed25519_ix(&ix_account)?;

    let ix_data = &ix_account.data;
    let public_key_bytes = &ix_data[16..48];

    require!(
        Pubkey::new_from_array(public_key_bytes.try_into().unwrap())
            == expected_message.investor_wallet,
        SecuritizeOnRampError::InvalidEd25519Instruction
    );

    let actual_message_hash = &ix_data[112..];

    let expected_message_bytes = expected_message.try_to_vec()?;
    let pid = crate::ID;
    let expected_message_hash =
        hashv(&[SWAP_SPL_TOKEN_TAG, pid.as_ref(), &expected_message_bytes]).to_bytes();

    require!(
        actual_message_hash == expected_message_hash,
        SecuritizeOnRampError::InvalidEd25519Instruction
    );

    Ok(())
}
```


**영향:** 재사용(replay)이나 on-ramp 간 교차 재사용이 가능하지만, 이를 악용하려면 신뢰된 operator가 해당 트랜잭션을 제출해야 합니다.

**권장 완화 조치:** 서명 메시지에 `on_ramp_state`, `asset_mint`, `liquidity_mint`를 포함하고, 투자자별 nonce를 추가해 온체인에서 저장 및 소모하도록 하십시오. 이렇게 하면 신뢰 가정을 유지하면서도 트랜잭션 간 혹은 on-ramp 인스턴스 간 재사용을 막을 수 있습니다.

**Securitize:** EVM 버전과 유사하게 맞추기 위해 nonce를 포함하지 않았음을 인지함.


### 초기화 시 `asset_vault`를 생략할 수 있지만 실제 swap은 이를 필수로 요구함

**설명:** `initialize`는 `asset_vault`를 선택적 계정(`Option<Box<InterfaceAccount<TokenAccount>>>`)으로 정의하고 `init_if_needed`를 사용합니다. 이 때문에 `asset_vault` 없이도 on-ramp 인스턴스를 만들 수 있습니다. 그러나 swap 명령들(`swap_spl_token`, `swap_ds_token`, `subscribe_ds_token`)은 `asset_vault`를 필수 계정으로 요구하며, 없으면 실패합니다. 특히 two-step transfer에서 이런 상태의 on-ramp는 실질적으로 swap에 사용할 수 없게 됩니다.

```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        mut,
        constraint = is_initializer_allowed(&admin.key())
            @ crate::errors::SecuritizeOnRampError::Forbidden,
    )]
    pub admin: Signer<'info>,

    #[account(
        mint::token_program = asset_token_program,
        mint::authority = asset_mint_authority,
    )]
    pub asset_mint: Box<InterfaceAccount<'info, Mint>>,
    /// CHECK: Mint authority for the asset mint
    pub asset_mint_authority: AccountInfo<'info>,
```

```rust
    #[account(
        init_if_needed,
        payer = admin,
        associated_token::mint = asset_mint,
        associated_token::authority = on_ramp_authority,
        associated_token::token_program = asset_token_program,
    )]
    pub asset_vault: Option<Box<InterfaceAccount<'info, TokenAccount>>>,
```

**영향:** swap을 수행할 수 없는 상태로 on-ramp가 초기화될 수 있습니다.

**권장 완화 조치:** `initialize` 명령에서 `asset_vault`를 필수 계정으로 만드십시오.

**Securitize:** [e2807db](https://github.com/securitize-io/bc-solana-on-off-ramp-sc/commit/e2807db4f483cc3acf421f479248cf8f7acc66bd)에서 수정됨.

**Cyfrin:** 확인함.


### `update_nav_provider`에서 NAV Provider가 실행 가능한 프로그램인지 확인하지 않음

**설명:** on-ramp 프로그램의 `update_nav_provider` 명령은 새로운 NAV provider를 **instruction data**(`NavProvider`)로만 받으며, 내부에 포함된 program ID가 실행 가능한 프로그램인지 검증하지 않습니다.

```rust
pub fn update_nav_provider_handler(
    ctx: &mut Context<UpdateNavProvider>,
    nav_provider: NavProvider,
) -> Result<()> {
```

반면 off-ramp 프로그램의 대응 명령은 새로운 NAV provider를 **계정(account)** 으로 받고 `#[account(executable)]`를 강제해, 저장되는 값이 실제 프로그램임을 보장합니다.

```rust
    /// New NAV provider program (must be executable)
    ///
    /// CHECK: Admin must provide a valid NAV provider program
    #[account(executable)]
    pub new_nav_provider: AccountInfo<'info>,
}
```

On-ramp에서는 `NavProvider`가 instruction data에서 역직렬화되므로, 어떤 pubkey든 상태에 기록될 수 있습니다. 새 NAV provider는 instruction data로만 전달되고, 계정도 없고 executable 체크도 없습니다.

**영향:** 이 불일치로 인해 on-ramp는 임의의 pubkey(예: 실행 불가능한 계정이나 PDA)를 NAV provider program ID로 저장할 수 있으며, 다른 코드가 이 값을 신뢰할 경우 실패하는 CPI, 예기치 않은 동작, 오용으로 이어질 수 있습니다.

**권장 완화 조치:** off-ramp와 동일하게 on-ramp에서도 새 NAV provider가 실행 가능한 프로그램인지 검증하십시오.

**Securitize:** [b4ba06a](https://github.com/securitize-io/bc-solana-on-off-ramp-sc/commit/b4ba06a8ab722b8770bffd46b10c85102104b0c8)에서 수정됨.

**Cyfrin:** 확인함.


### `collector-token-account`에 대한 검증이 불충분함

**설명:** `initialize`와 `update_fee_manager_handler` 명령의 `collector_token_account`는 적절히 검증되지 않습니다. `update_fee_manager`를 통해 [FeeManager]를 갱신할 때, 프로토콜은 수수료 비율(numerator)만 검증하고 `collector_token_account` 주소는 검증하지 않습니다.

[mpbs_fee_manager.rs]의 [validate()] 함수는 fee numerator만 확인합니다.
```rust
impl FeeManagerTrait for MbpsFeeManager {
    fn validate(&self) -> Result<()> {
        require!(
            self.numerator <= Self::MAX_FEE_NUMERATOR,
            SecuritizeOnRampError::MaxFeeExceeded
        );
        Ok(())
    }
}
```
swap 시점에는 [swap_ds_token.rs]와 [swap_spl_token.rs]에서 [fee_collector_ta]가 저장된 주소와 일치하는지 검증합니다.
```rust
#[account(
    mut,
    address = on_ramp_state.fee_manager.fee_collector_token_account()
        @ crate::errors::SecuritizeOnRampError::InvalidFeeCollector,
    token::mint = liquidity_mint,
    token::token_program = liquidity_token_program,
)]
pub fee_collector_ta: Box<InterfaceAccount<'info, TokenAccount>>,
```
하지만 이 검증은 swap 시점에만 일어납니다. 따라서 관리자가 잘못된 `collector_token_account` 주소(예: 잘못된 mint, frozen account, 닫힌 계정, 존재하지 않는 계정)를 설정하면, 제약 검증이 실패해 관리자가 설정을 고칠 때까지 모든 swap이 막히게 됩니다.

**영향:** swap에 대한 일시적 서비스 거부(DoS)

**권장 완화 조치:** UpdateFeeManager 명령에서 `collector_token_account`를 검증해 다음을 보장하십시오.
- 존재하며 유효한 token account일 것
- 올바른 mint(`[liquidity_mint]`)를 가질 것
- frozen 상태가 아닐 것

**Securitize:** [be1ac28](https://github.com/securitize-io/bc-solana-on-off-ramp-sc/commit/be1ac28823ec2719d8ac9956780b7fa176da01a4)에서 수정됨.

**Cyfrin:** 확인함.


### 투자자가 operator를 괴롭힐 수 있음

**설명:** `swap_spl_token` 흐름에서 투자자 토큰 계정의 잔액 검사는 실행 경로 끝부분, 즉 `swap_process()` 내부에서 이뤄집니다.
```rust
    // in `swap_logic.rs`
    // Transfer liquidity from investor
    require_gte!(
        params.investor_liquidity_ta.amount,
        params.liquidity_amount,
        SecuritizeOnRampError::InsufficientBalance
    );
```
이 시점 이전에는 이미 다음과 같은 계산 비용이 큰 작업들이 수행된 뒤입니다.
- pause 상태 체크
- token type 검증
- 최소 구독 금액 체크
- Ed25519 서명 검증(비용 큼 - `validate_investor_signature`)
- remaining accounts 검증
- 수수료 계산
- AMM `execute_buy_base` CPI 호출(비용 큼)
- asset amount 계산

마지막으로 `swap_process()` 내부에서 슬리피지 검증과 `liquidity_amount_excluding_fee` 계산을 마친 뒤에야, 투자자 토큰 잔액이 liquidity amount보다 큰지 확인합니다. 악성 투자자는 이런 연산 순서를 악용해 operator를 괴롭힐 수 있습니다(operator가 signer라서 트랜잭션 수수료를 냄). CU가 커질수록 수수료도 커지므로, 많은 검증과 CPI는 CU 사용량을 크게 늘리고 그 비용은 operator가 부담합니다.

**영향:** 악성 투자자가 operator에게 큰 트랜잭션 수수료를 부담시키고, 의도적으로 실패하는 트랜잭션을 발생시키는 방식으로 괴롭힐 수 있습니다.

**권장 완화 조치:** 비싼 연산을 하기 전에 `swap_spl_token_handler` 초반으로 잔액 검사를 옮기십시오.
```rust
pub fn swap_spl_token_handler<'info>(
    ctx: &Context<'_, '_, '_, 'info, SwapSplToken<'info>>,
    liquidity_amount: u64,
    min_out_amount: u64,
    deadline: i64,
    asset_provider_accounts_count: u8,
    nav_provider_params: &NavProviderParams,
) -> Result<()> {
    let on_ramp_state = &ctx.accounts.on_ramp_state;

    require!(!on_ramp_state.is_paused, SecuritizeOnRampError::Paused);

    // Early balance check - fail fast before expensive operations
    require_gte!(
        ctx.accounts.investor_liquidity_ta.amount,
        liquidity_amount,
        SecuritizeOnRampError::InsufficientBalance
    );

    require!(
        on_ramp_state.asset_token_type == crate::states::TokenType::SplToken,
        SecuritizeOnRampError::InvalidTokenType
    );

    // ...existing code...
```
참고로, 투자자가 freeze된 토큰 계정을 통해 동일한 괴롭힘을 시도할 수 있으므로 `investor_asset_ta`가 초기에 frozen 상태가 아닌지도 함께 확인해야 합니다.

**Securitize:** 인지함.



### 국가 제한이 초기화 시 비어 있는 값으로 설정되어, 모든 관할권 사용자가 프로토콜을 이용할 수 있음

**설명:** 새로운 off-ramp state가 `initialize`로 생성될 때 `countries_restriction`은 `CountriesRestriction::default()`(모든 비트가 0인 비트맵)로 하드코딩됩니다. 따라서 관리자가 `update_countries_restriction`를 명시적으로 호출하기 전까지는 어떤 국가도 제한되지 않습니다.

```rust
    let off_ramp_state_inst = OffRampState {
        admin: ctx.accounts.admin.key(),
        asset_mint: ctx.accounts.asset_mint.key(),
        asset_policy,
        asset_token_type,
        bump: ctx.bumps.off_ramp_state,

        id: counter,
        is_paused: false,
        off_ramp_authority_bump: ctx.bumps.off_ramp_authority,
        nav_provider,
        liquidity_mint: ctx.accounts.liquidity_mint.key(),
        fee_manager,
        countries_restriction: CountriesRestriction::default(),
        operators: vec![],
        two_step_transfer: false,
        liquidity_provider,
    };
```


`CountriesRestriction`은 32바이트 비트맵이며 기본값은 모두 0입니다.

```rust
/// Bitmap for restricting up to 256 countries
#[derive(AnchorDeserialize, AnchorSerialize, Clone, Debug, InitSpace, Default)]
pub struct CountriesRestriction([u8; 32]);
```

모든 비트가 0이면 어떤 국가 인덱스에 대해서도 `is_restricted(idx)`는 false를 반환합니다.

```rust
    /// Returns true if the country is restricted
    pub fn is_restricted(&self, idx: u8) -> bool {
        let byte = self.0[(idx / 8) as usize];
        let bit = idx % 8;
        (byte & (1 << bit)) != 0
    }
```

실제 redemption 시에는 다음 위치에서 제한을 검사합니다.

```rust
    require!(
        !off_ramp_state
            .countries_restriction
            .is_restricted(redeemer_country),
        SecuritizeOffRampError::RestrictedCountry,
    );
```


따라서 최초 `initialize`부터 첫 `update_countries_restriction`가 실행될 때까지(그리고 그 호출이 누락되거나 지연되면 무기한), 어느 국가도 제한되지 않으며 어떤 관할권의 사용자든 redemption을 수행할 수 있습니다.

**영향:** **컴플라이언스/규제 리스크**: 프로토콜이 출시 시점부터 특정 국가를 제한해야 하는 요구사항(예: 제재, 라이선스)이 있다면, 관리자가 비트맵을 갱신하기 전까지는 비준수 상태가 됩니다.

**권장 완화 조치:** `initialize` 명령에 선택적(또는 필수) 인자를 추가해, 배포자가 같은 트랜잭션에서 초기 비트맵도 설정할 수 있게 하십시오.

**Securitize:** 초기화 로직을 더 무겁고 컴플라이언스 설정과 강하게 결합시키고 싶지 않으며, 필요하면 관리자가 `initialize`와 `update_countries_restriction`를 하나의 트랜잭션으로 묶어 실행할 수 있다는 입장을 밝혔습니다.



### Allowance Liquidity Provider가 실제 잔액으로 상한 처리하지 않고 `delegated_amount`만 사용함

**설명:** `AllowanceLiquidityProvider`의 `available_liquidity`는 source token account의 `delegated_amount`만 반환합니다.

```rust
        let source_token_account =
            TokenAccount::try_deserialize(&mut &source_token_account_info.data.borrow()[..])?;

        require_keys_eq!(
            source_token_account.delegate.unwrap_or_default(),
            expected_delegate.key(),
            SecuritizeOffRampError::InvalidLiquidityProviderConfiguration
        );

        Ok(source_token_account.delegated_amount)
```

 SPL Token에서는 승인 이후 `self-transfer`로 토큰을 계정 밖으로 옮겨도 `delegated_amount`가 줄어들지 않습니다. 따라서 보고되는 "가용 유동성"이 실제 현재 잔액을 초과할 수 있습니다. 그 결과 유동성이 과대 표시되고, LP 잔액이 보고된 allowance보다 적을 경우 redemption은 transfer 시점에 실패할 수 있습니다.

```rust
                if !self_transfer {
                    source_account.delegated_amount = source_account
                        .delegated_amount
                        .checked_sub(amount)
                        .ok_or(TokenError::Overflow)?;
                    if source_account.delegated_amount == 0 {
                        source_account.delegate = COption::None;
                    }
                }
```

또한 `calculate_effective_liquidity_amount`는 실제 토큰 잔액이나 가용 유동성을 전혀 고려하지 않고 요청된 수량을 그대로 반환합니다.


**영향:**
- **잘못된 유동성 보고:** off-ramp와 통합자는 LP 잔액이 더 낮더라도 `delegated_amount`를 기준으로 "available liquidity"를 보여줄 수 있습니다. 예를 들어 LP가 다른 곳으로 토큰을 옮긴 뒤에도 그렇습니다.
- **실패하는 redemption:** 사용자나 operator가 실제로는 transfer를 감당하지 못하는 수량으로 redemption을 시작할 수 있고, source account 잔액 부족으로 실패합니다. 이는 실패한 트랜잭션과 나쁜 UX를 만들며, redemption 시도에 대한 일종의 DoS로 볼 수도 있습니다.

**권장 완화 조치:** `available_liquidity`를 현재 잔액으로 상한 처리해, 예를 들어 `Ok(source_token_account.delegated_amount.saturating_min(source_token_account.amount))`를 반환하도록 하십시오.

**Securitize:** [cdd059f](https://github.com/securitize-io/bc-solana-on-off-ramp-sc/commit/cdd059f8d883c827f0e6ae14dcfe9067a32d082f)에서 수정됨.

**Cyfrin:** 확인함.


### Rate CPI 컨텍스트가 불완전하고 NAV Provider 최소 계정 수 요구가 과소함

**설명:** on-ramp와 off-ramp 프로그램은 NAV provider의 `rate` 명령을 CPI로 호출하지만, `on_ramp_authority` 또는 `off_ramp_authority` PDA로 서명하지 않습니다. CPI는 다음과 같이 서명 없는 컨텍스트로 구성됩니다.

```rust
let get_rate_ctx = CpiContext::new(
    nav_provider_program.to_account_info(),
    nav_provider_interface::cpi::accounts::Rate {
        asset_mint: asset_mint.to_account_info(),
        nav_provider_state: nav_provider_state.to_account_info(),
    },
)
.with_remaining_accounts(nav_provider_accounts[2..].to_vec());
```

현재는 NAV provider 프로그램이 프로토콜의 신뢰 경계 안에 있고, 어떤 프로그램을 설정할지는 admin이 제어하므로 안전합니다. 하지만 향후 NAV provider가 호출자 인증을 필요로 하게 되면(예: 권한 있는 ramp 프로그램만 rate 조회 허용, 호출자별 rate limit, on-ramp/off-ramp 호출자에 따라 다른 가격 정책 적용 등), 현재처럼 서명 없는 CPI 패턴으로는 프로토콜 업그레이드 없이는 이를 지원할 수 없습니다.

같은 패턴은 off-ramp의 `get_rate`와 AMM NAV provider 경로(`execute_buy_base`, `execute_sell_base`, `quote_buy_base`, `quote_sell_base`)에도 사용되며, 이들 역시 ramp authority PDA로 서명하지 않습니다.

**영향:** NAV provider 프로그램은 자신을 호출한 ramp 프로그램의 신원을 검증할 수 없습니다. 현재 모든 NAV provider가 신뢰 경계 안에 있고 호출자 인증을 요구하지는 않지만, 미래에 호출자에 따라 접근 제어나 동작 차등이 필요해지면 양 프로그램 모두의 인터페이스를 바꿔야 하므로 확장성이 떨어집니다.

**권장 완화 조치:** NAV provider CPI 호출을 `on_ramp_authority` / `off_ramp_authority` PDA로 서명하고, Rate 인터페이스 struct에 authority 계정을 포함시키십시오. 그러면 NAV provider는 필요할 때만 선택적으로 호출자를 검증할 수 있습니다.
```rust
let get_rate_ctx = CpiContext::new_with_signer(
    nav_provider_program.to_account_info(),
    nav_provider_interface::cpi::accounts::Rate {
        asset_mint: asset_mint.to_account_info(),
        nav_provider_state: nav_provider_state.to_account_info(),
        caller_authority: on_ramp_authority.to_account_info(),
    },
    &[&on_ramp_authority_seeds],
)
.with_remaining_accounts(nav_provider_accounts[2..].to_vec());
```

**Securitize:** [c8dd8d9](https://github.com/securitize-io/bc-solana-on-off-ramp-sc/commit/c8dd8d9da9a8efe67bf60658e3ec7b8aec7194bd)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 정보성 (Informational)


### On-Ramp `initialize`는 zero genesis hash를 거부해야 함

**설명:** on-ramp의 `initialize` 명령은 `[0; 32]`를 "genesis hash PDA가 아직 설정되지 않음"을 나타내는 sentinel로 사용하지만, 동일한 값이 `genesis_hash` 명령 인자로 들어왔을 때는 이를 거부하지 않습니다.

```rust
    // Genesis hash is a singleton config used for signature domain separation.
    // It is set once on first initialize; subsequent initializes must match.
    if ctx.accounts.genesis_hash.hash == [0; 32] {
        ctx.accounts.genesis_hash.set_inner(GenesisHash {
            hash: genesis_hash,
            bump: ctx.bumps.genesis_hash,
        });
    } else {
        require!(
            ctx.accounts.genesis_hash.hash == genesis_hash,
            crate::errors::SecuritizeOnRampError::GenesisHashMismatch
        );
    }
```

즉 **instruction argument**인 `genesis_hash != [0; 32]`를 선제적으로 확인하는 코드가 없기 때문에, 잘못된 결과(0 값 저장)를 막지 못합니다.

```rust
pub fn initialize_handler(
    ctx: &mut Context<Initialize>,
    fee_manager: crate::FeeManager,
    asset_provider: crate::AssetProvider,
    nav_provider: crate::NavProvider,
    custodian_wallet: Pubkey,
    genesis_hash: [u8; 32],
) -> Result<()> {
```

그 결과 첫 번째 `initializer`는 이 singleton genesis hash를 전부 0인 값으로 설정할 수 있습니다. 이 값은 이후 `swap_spl_token` 등의 서명 검증에서 클러스터별 도메인 구분자(domain separator)로 사용됩니다. 게다가 현재 설계는 이 `genesis_hash`가 모든 on-ramp state에서 일관되게 유지된다는 것도 보장하지 못합니다.

**영향:** `[0; 32]`를 "초기화되지 않음" 전용 값으로 예약하지 않으면, sentinel의 의미가 흐려집니다.

**권장 완화 조치:** handler 시작부에서 zero hash를 거부해, 이 값이 오직 "uninitialized" 상태만 나타내고 실제 genesis hash로는 저장되지 않게 하십시오.

**Securitize:** [5b035d1](https://github.com/securitize-io/bc-solana-on-off-ramp-sc/commit/5b036d1a53e59a2a9c7d59a43bb98195e6240517)에서 수정됨.

**Cyfrin:** 확인함.


### On-Ramp `UpdateAssetProvider`의 중복된 `#[instruction(asset_provider)]` 속성

**설명:** `UpdateAssetProvider` accounts struct는 `#[instruction(asset_provider: crate::AssetProvider)]`를 통해 instruction argument `asset_provider`를 선언하지만, 어떤 account constraint도(`constraint`, `seeds`, `has_one` 등) 이 변수를 참조하지 않습니다.

```rust
#[derive(Accounts)]
#[instruction(asset_provider: crate::AssetProvider)]
pub struct UpdateAssetProvider<'info> {
    pub admin: Signer<'info>,

    #[account(
        mut,
        has_one = admin @ SecuritizeOnRampError::Forbidden,
        seeds = [ON_RAMP_STATE_SEED, on_ramp_state.id.to_le_bytes().as_ref()],
        bump = on_ramp_state.bump,
    )]
    pub on_ramp_state: Box<Account<'info, OnRampState>>,
}
```
 이 argument는 handler에서만 사용됩니다. 따라서 clarity 관점에서 이 attribute는 중복이며 제거할 수 있습니다.

**영향:** 해당 코드는 중복이므로 코드 품질 차원에서 제거 가능합니다.

**권장 완화 조치:** 사용되지 않는 attribute를 제거하십시오.

**Securitize:** [b29a57](https://github.com/securitize-io/bc-solana-on-off-ramp-sc/commit/b29a576d2d9def6eb57461206943e5e115673ee8)에서 수정됨.

**Cyfrin:** 확인함.


### 호환되지 않는 NavProvider가 설정될 수 있어 일시적으로 swap이 실패할 수 있음

**설명:** 프로토콜에는 `Dstoken`과 `SplToken`이라는 두 종류의 swap token type이 있습니다. 각 tokenType은 특정 `NavProvider`와 결합되어야 합니다. DsToken에는 StandardNavProvider가 필요하고, splToken에는 AmmNavProvider가 필요합니다. 잘못된 NavProvider가 설정되면 오류가 발생합니다.
ds token swap의 경우:
```rust
        NavProvider::AmmNavProvider(_) => {
            return err!(SecuritizeOnRampError::UnsupportedNavProvider);
        }
```
spl token swap의 경우:
```rust
        NavProvider::StandardNavProvider(_) => {
            return err!(SecuritizeOnRampError::UnsupportedNavProvider)
        }
```
하지만 `update_nav_provider_handler`는 전달된 `NavProvider`가 해당 on-ramp의 tokenType과 호환되는지 확인하지 않습니다. 관리자가 실수로 호환되지 않는 NavProvider를 설정하면 수정되기 전까지 swap이 실패하게 됩니다.

참고로 `update_asset_provider_handler`도 비슷한 호환성 검증이 없습니다.

**영향:** swap이 일시적으로 실패합니다.

**권장 완화 조치:** 호환성 검증을 추가하십시오.
```rust
pub fn update_nav_provider_handler(
    ctx: &mut Context<UpdateNavProvider>,
    nav_provider: NavProvider,
) -> Result<()> {
    let on_ramp_state = &mut ctx.accounts.on_ramp_state;

    let old_nav_provider = on_ramp_state.nav_provider;

    require!(
        old_nav_provider != nav_provider,
        SecuritizeOnRampError::NoChange
    );

    // Validate NavProvider is compatible with asset_token_type
    match (&on_ramp_state.asset_token_type, &nav_provider) {
        (TokenType::DsToken, NavProvider::AmmNavProvider(_)) => {
            return err!(SecuritizeOnRampError::UnsupportedNavProvider);
        }
        (TokenType::SplToken, NavProvider::StandardNavProvider(_)) => {
            return err!(SecuritizeOnRampError::UnsupportedNavProvider);
        }
        _ => {}
    }

    on_ramp_state.nav_provider = nav_provider;

    // ...existing code...
    Ok(())
}
```
**Securitize:** [d14f30e](https://github.com/securitize-io/bc-solana-on-off-ramp-sc/commit/d14f30e6b549999addef7bdc9e80824435ca9acc)에서 수정됨.

**Cyfrin:** 확인함.


### Off-Ramp `update_fee_manager`는 `liquidity_mint`를 검증하지 않음

**설명:** on-ramp 프로그램에서 `update_fee_manager`는 `on_ramp_state`가 `has_one = liquidity_mint`를 만족하도록 하고, `liquidity_mint` 계정도 함께 받아 명령을 명시적으로 해당 mint에 바인딩합니다.

```rust
    #[account(
        mut,
        has_one = liquidity_mint @ SecuritizeOnRampError::InvalidMint,
        has_one = admin @ SecuritizeOnRampError::Forbidden,
        seeds = [ON_RAMP_STATE_SEED, on_ramp_state.id.to_le_bytes().as_ref()],
        bump = on_ramp_state.bump,
    )]
    pub on_ramp_state: Box<Account<'info, OnRampState>>,

    #[account(
        mint::token_program = liquidity_token_program,
    )]
    pub liquidity_mint: Box<InterfaceAccount<'info, Mint>>,

    pub liquidity_token_program: Interface<'info, TokenInterface>,
```

반면 off-ramp 프로그램의 `update_fee_manager`는 `OffRampState`에도 `liquidity_mint` 필드가 있음에도 불구하고, 이 mint를 기준으로 한 검증을 수행하지 않습니다. 이 불일치는 잘못된 state를 실수로 대상으로 삼기 쉽게 만들고, 두 프로그램의 제약 일관성도 약화시킵니다.

```
    #[account(
        mut,
        has_one = admin @ SecuritizeOffRampError::Forbidden,
        seeds = [OFF_RAMP_STATE_SEED, off_ramp_state.id.to_le_bytes().as_ref()],
        bump = off_ramp_state.bump,
    )]
    pub off_ramp_state: Box<Account<'info, OffRampState>>,
```

**영향:** on-ramp와 off-ramp는 `update_fee_manager`를 제약하는 방식이 서로 다릅니다. liquidity mint 계정과 `has_one = liquidity_mint`를 요구하면, "어떤 mint에 대한 어떤 state인지"라는 불변식을 명확히 하고 off-ramp를 on-ramp 및 다른 off-ramp 명령들과 맞출 수 있습니다.

**권장 완화 조치:** off-ramp도 on-ramp 및 다른 off-ramp 명령과 동일하게 정렬하십시오.

**Securitize:** [bbcdfe0](https://github.com/securitize-io/bc-solana-on-off-ramp-sc/commit/bbcdfe045b6a2983b7d54a220a1fe31032a3e3e8)에서 수정됨.

**Cyfrin:** 확인함.


### liquidity amount에 대한 검사가 부족해 불필요하게 compute를 낭비함

**설명:** `[redeem_ds_token_handler]`와 `[redeem_spl_token_handler]`는 모두 NAV rate와 asset amount로부터 `[liquidity_amount]`를 계산한 뒤, liquidity provider의 `[source_token_account]`가 redemption을 감당할 충분한 잔액(또는 위임 allowance)을 실제로 보유하는지 먼저 확인하지 않고 바로 `[redemption_manager::redeem()]`로 들어갑니다.

```rust
let liquidity_amount = utils::token_calculator::calculate_liquidity_amount(
    asset_amount,
    rate,
    ctx.accounts.asset_mint.decimals,
    ctx.accounts.liquidity_mint.decimals,
)?;

require_gt!(liquidity_amount, 0, SecuritizeOffRampError::ZeroAmount);

// No check that the liquidity provider can actually fulfill `liquidity_amount`

let (fee_amount, user_supplied_amount) = redemption_manager::redeem(RedemptionParams {
    // ...
})?;
```
two-step redemption 흐름 `[execute_two_step_redemption.rs]`에서는, redeemer의 asset token이 먼저 `[asset_vault]`로 전송된 뒤 liquidity provider의 `[supply_to]`가 잔액 부족이나 delegation 부족으로 실패합니다. Solana의 원자성 덕분에 전체 트랜잭션은 revert되어 자금이 손실되지는 않지만, 실패가 CPI 체인 깊숙한 곳에서 발생해 명확한 오류로 일찍 잡히지 않습니다. 그 결과 operator나 investor가 불필요한 compute를 더 부담하게 됩니다.

**영향:** 늦게 발생하는 실패는 compute unit을 낭비하고, 명확한 InsufficientLiquidity 오류 대신 SPL Token transfer 오류를 발생시키며, 사용자 경험을 악화시킵니다. operator가 중개하는 SPL token redemption에서는 이런 늦은 실패가 반복되면 서명이 만료되어 redeemer가 오프체인 서명 흐름을 다시 거쳐야 할 수도 있습니다.

**권장 완화 조치:** `[liquidity_amount]` 계산 직후, `[redemption_manager::redeem()]`를 호출하기 전에 조기 유동성 충분성 검사를 추가하십시오. 이 시점에는 `[liquidity_provider_accounts]`(그 안에 `[source_token_account]` 포함)가 이미 준비되어 있습니다.
```rust
let liquidity_amount = utils::token_calculator::calculate_liquidity_amount(
    asset_amount,
    rate,
    ctx.accounts.asset_mint.decimals,
    ctx.accounts.liquidity_mint.decimals,
)?;

require_gt!(liquidity_amount, 0, SecuritizeOffRampError::ZeroAmount);

// Add early liquidity check
let available = off_ramp_state
    .liquidity_provider
    .available_liquidity(off_ramp_state, liquidity_provider_accounts)?;
require_gte!(
    available,
    liquidity_amount,
    SecuritizeOffRampError::InsufficientLiquidity
);

let (fee_amount, user_supplied_amount) = redemption_manager::redeem(RedemptionParams {
    // ...existing code...
})?;
```
**Securitize:** 인지함.



### 투자자의 국가가 금지되면 기존 dsToken이 잠길 수 있음

**설명:** off-ramp 프로그램은 `[OffRampState]`에 저장된 `[countries_restriction]` 비트맵을 통해 DS token redemption 시 국가 기반 제한을 강제합니다. redemption 처리 전 `[redeem_ds_token_handler]`는 투자자의 온체인 `[IdentityAccount]`에서 국가를 읽고 제한되지 않았는지 확인합니다.
```rust
let redeemer_country = ctx.accounts.identity_account.country;

require!(
    !off_ramp_state
        .countries_restriction
        .is_restricted(redeemer_country),
    SecuritizeOffRampError::RestrictedCountry,
);
```
하지만 반대 방향 흐름(투자자가 liquidity token을 보내고 asset token을 받는 흐름)을 담당하는 on-ramp 프로그램에는 국가 제한 메커니즘이 전혀 없습니다. `[OnRampState]` struct에는 아예 `[countries_restriction]` 필드가 존재하지 않습니다.
```rust
pub struct OnRampState {
    pub id: u64,
    pub admin: Pubkey,
    pub asset_mint: Pubkey,
    pub asset_token_type: TokenType,
    pub liquidity_mint: Pubkey,
    pub nav_provider: NavProvider,
    pub is_paused: bool,
    pub fee_manager: FeeManager,
    pub custodian_wallet: Pubkey,
    pub bump: u8,
    pub on_ramp_authority_bump: u8,
    pub min_subscription_amount: u64,
    pub investor_subscription_enabled: bool,
    pub two_step_transfer: bool,
    pub asset_provider: AssetProvider,
    pub operators: Vec<Pubkey>,
    // No countries_restriction field
}
```
세 가지 swap 함수 모두 투자자 국가를 확인하는 체크가 없습니다.
- `swap_spl_tokens`는 operator가 서명하므로 **금지 국가 투자자에 대해서는 operator가 서명하지 않을 것이라고 신뢰**할 수 있습니다.
- `subscribe_ds_tokens`는 registrar authority가 서명하므로 마찬가지로 어느 정도 가정할 수 있습니다.
- 하지만 `swap_ds_tokens`는 투자자만 트랜잭션에 서명하고 자신의 `wallet_identity` 계정을 제공합니다. 여기서 확인하는 것은 `wallet_identity`가 서명한 투자자 지갑에 속하는지만이며, on-ramp 프로그램은 `swap_ds_tokens` 처리 시 국가 제한을 강제하지 않습니다.
```rust
    /// Check that the investor wallet is registered
    #[account(
        constraint = wallet_identity.wallet == investor_wallet.key()
            @ SecuritizeOnRampError::Forbidden,
        seeds = [investor_wallet.key().as_ref(), asset_mint.key().as_ref()],
        seeds::program = identity_registry::ID,
        bump,
    )]
    pub wallet_identity: Box<Account<'info, identity_registry::WalletIdentity>>,
```

`swap_ds_tokens`는 `wallet_identity` 계정이 있어야 하므로 이전에 `subscribe_ds_tokens`를 한 번 수행했어야 하지만, 이후 swap 과정에서 국가 재검증은 전혀 이뤄지지 않습니다.


이로 인해 다음과 같은 상태 전이 불일치가 생깁니다.
1. 제한되지 않은 국가의 투자자가 `subscribe_ds_tokens`를 호출해 `wallet_identity`를 만들거나 이미 DS token을 보유합니다.
2. 나중에 관리자가 off-ramp의 `countries_restriction` 비트맵을 갱신해 해당 국가를 금지합니다.
3. 기존 투자자는 on-ramp에서 국가 제한을 검사하지 않으므로 여전히 `swap_ds_tokens`를 호출할 수 있습니다.
4. 투자자는 추가 DS token을 획득합니다.
5. 그러나 off-ramp에서 redemption을 시도하면 `redeem_ds_token_handler`가 `RestrictedCountry`로 거부합니다.

일반적으로 투자자가 국가 금지 이전부터 DS token을 보유하고 있었다면, 그 토큰은 즉시 상환 불가능 상태가 됩니다. 결과적으로 DS token은 경제적으로 잠긴 자산이 될 수 있습니다. 투자자는 토큰을 보유하지만, 프로토콜이 지정한 redemption 메커니즘을 통해 빠져나올 수 없습니다. 관리자가 나중에 off-ramp 국가 제한을 해제하지 않는 한 이 잠금은 지속되며, 이는 애초의 규제 목적과도 충돌할 수 있습니다.

**영향:** 등록 이후 국가가 제한된 투자자는:
- 계속해서 `swap_ds_tokens`를 통해 DS token을 취득할 수 있음
- 하지만 off-ramp를 통해서는 해당 토큰을 상환할 수 없음
- 기존에 가지고 있던 DS token도 영구적으로 상환 불가능해질 수 있음

제한 국가 투자자가 on-ramp로 자산 토큰을 취득할 수 있지만, off-ramp를 통해 이를 상환할 수 없게 되어 자금이 잠기게 됩니다.

**권장 완화 조치:** `[OnRampState]`에도 동일한 `[CountriesRestriction]` 비트맵인 `[countries_restriction]` 필드를 추가하고, 투자자가 직접 상호작용하는 모든 on-ramp 명령에서 국가 검사를 강제하십시오.

**Securitize:** 이는 의도된 동작이며 요구사항과 일치한다고 인지함.
