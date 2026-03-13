**수석 감사**

[Farouk](https://x.com/Ubermensh3dot0)

[Naman](https://x.com/namx05)

**보조 감사**

[Alex Roan](https://twitter.com/alexroan)

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

---

# 발견 사항

## 높은 위험 (High Risk)

### 상환(Redeem) 시 깨진 신원-지갑 바인딩으로 인한 국가 제한 우회

**설명:** `redeem` 명령어는 모두 동일한 사람과 지갑을 참조해야 하는 세 가지 신원 객체인 `IdentityRegistryAccount`, `IdentityAccount`, `WalletIdentity`를 허용합니다. 계정 제약 조건은 다음만 보장합니다:
- `IdentityRegistryAccount`가 `asset_mint`와 일치함.
- `IdentityAccount`가 해당 레지스트리에 속함.
- `WalletIdentity`가 `(redeemer, asset_mint)`에 대한 PDA임.

제공된 `IdentityAccount`가 `WalletIdentity` 및 `redeemer`와 연결된 것인지에 대한 온체인 어설션(assertion)이 없습니다. 결과적으로 호출자는 자신의 지갑에 대한 유효한 `WalletIdentity`와 허용된 국가를 가진 다른 사람의 `IdentityAccount`를 혼합하여 국가 확인을 통과할 수 있습니다.

```rust
/// 자산 민트 준수를 위한 신원 레지스트리
#[account(
    has_one = asset_mint,
    seeds = [asset_mint.key().as_ref()],
    seeds::program = ::identity_registry::ID,
    bump = identity_registry.bump,
)]
pub identity_registry: Box<Account<'info, IdentityRegistryAccount>>,

/// 국가 정보가 포함된 사용자의 신원 계정
#[account(
    has_one = identity_registry,
    seeds = [identity_registry.key().as_ref(), identity_account.owner.as_ref()],
    seeds::program = ::identity_registry::ID,
    bump
)]
pub identity_account: Box<Account<'info, IdentityAccount>>,

/// 지갑을 신원 계정에 연결
#[account(
    seeds = [redeemer.key().as_ref(), asset_mint.key().as_ref()],
    seeds::program = ::identity_registry::ID,
    bump,
)]
pub wallet_identity: Box<Account<'info, WalletIdentity>>,
```

**파급력:** 국가 제한 우회. 제한된 국가의 지갑이 허용된 국가를 보고하는 다른 사용자의 `IdentityAccount`를 제공하여 상환할 수 있습니다.

**권장 완화 방법:** 제공된 `IdentityAccount`와 `WalletIdentity` 간의 연관성을 요구하는 것을 고려하십시오.

**Securitize:** [78ad18d](https://github.com/securitize-io/bc-solana-redemption-sc/commit/78ad18d0dc78f7468be0092667046bea021b7875)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 중간 위험 (Medium Risk)

### `off_ramp_authority`에 대한 사전 생성된 ATA를 통한 초기화(Initialize) DoS

**설명:** `initialize` 명령어는 프로그램 PDA `off_ramp_authority`에 대한 **연관 토큰 계정(Associated Token Account, ATA)** 으로 유동성 볼트를 생성합니다:
```rust
#[account(
    init,
    payer = admin,
    associated_token::mint = liquidity_token_mint,
    associated_token::authority = off_ramp_authority,
    associated_token::token_program = liquidity_token_program,
)]
pub liquidity_token_vault: Box<InterfaceAccount<'info, TokenAccount>>;
```
연관 토큰 계정은 전역적으로 파생 가능하며 소유자의 서명 없이 **누구나** 어떤 소유자에 대해서도 생성할 수 있습니다. `off_ramp_state`와 `off_ramp_authority` PDA 모두 공개 시드(카운터 및 상태 키)에서 결정론적으로 파생되므로, 공격자는 볼트 ATA를 미리 계산하여 먼저 생성할 수 있습니다. 나중에 `initialize`가 `init`과 함께 실행될 때, Anchor는 "already in use(이미 사용 중)" 오류와 함께 전체 트랜잭션을 되돌립니다(revert).

**파급력:** 주어진 `(off_ramp_state, liquidity_token_mint)`에 대한 프로그램 초기화에 대한 강력한 서비스 거부(DoS). 공격자는 예상되는 각 off_ramp ID에 대해 ATA를 반복적으로 미리 생성하여 관리자가 매개변수를 변경하지 않는 한 배포를 차단할 수 있습니다. 이는 공격자에게 저렴하며 반복될 수 있습니다.

**권장 완화 방법:** 초기화를 멱등적(idempotent)으로 만들고 사전 생성에 면역이 되도록 `init_if_needed`로 전환하십시오:
```rust
#[account(
    init_if_needed,
    payer = admin,
    associated_token::mint = liquidity_token_mint,
    associated_token::authority = off_ramp_authority,
    associated_token::token_program = liquidity_token_program,
)]
pub liquidity_token_vault: Box<InterfaceAccount<'info, TokenAccount>>;
```
이는 기존에 올바른 ATA가 있으면 수락하고 진행합니다.

**Securitize:** [1a8a098](https://github.com/securitize-io/bc-solana-redemption-sc/commit/1a8a0989c940eb8978ff3556bfc513ee0606f6dc)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 공식 프로그램 ID 하에서의 무허가 OffRampState 초기화로 스푸핑된 "공식" 인스턴스 가능

**설명:** `initialize` 명령어는 서명자가 새로운 `OffRampState`를 생성하고 그 `admin`이 될 수 있게 합니다. 전역 `OffRampCounter`는 `init_if_needed`이며 보호되지 않고, 새로운 상태 PDA는 `[OFF_RAMP_STATE_SEED, off_ramp_counter.counter.to_le_bytes()]`에서 파생됩니다. 초기화 자를 공식 Securitize 운영자와 연결하는 허용 목록(allowlist)이나 레지스트리 확인이 없습니다.
즉, 누구나 동일한 프로그램 ID로 OffRamp 인스턴스를 시작하고 `Initialized` 이벤트를 방출할 수 있으며, 이는 공식적인 Securitize 지원 오프 램프인 것처럼 마케팅될 수 있습니다.
```rust
/// 고유한 오프 램프 ID 생성을 위한 전역 카운터
#[account(
    init_if_needed,
    payer = admin,
    space = 8 + OffRampCounter::INIT_SPACE,
    seeds = [OFF_RAMP_COUNTER_SEED],
    bump,
)]
pub off_ramp_counter: Box<Account<'info, OffRampCounter>>,

/// 구성 및 설정을 포함하는 오프 램프 상태
#[account(
    init,
    payer = admin,
    space = 8 + OffRampState::INIT_SPACE,
    seeds = [OFF_RAMP_STATE_SEED, off_ramp_counter.counter.to_le_bytes().as_ref()],
    bump,
)]
pub off_ramp_state: Box<Account<'info, OffRampState>>,
```

**파급력:** 제3자가 임의의 수수료, NAV 제공자 및 수신자 정책을 사용하여 유사한 인스턴스를 배포한 다음, 동일한 프로그램 ID로 호스팅되기 때문에 "Securitize 오프 램프"로 제시할 수 있습니다.

**권장 완화 방법:** `authorized_initializer` 또는 허용 목록을 저장하는 `GlobalConfig` PDA를 추가하십시오. `initialize`에서 `admin` 서명자가 해당 목록에 있는지 요구하십시오.

**Securitize:** [30362cf](https://github.com/securitize-io/bc-solana-redemption-sc/commit/30362cf3d6b349cad72134f843808464d7477502)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 낮은 위험 (Low Risk)

### 유동성 토큰 금액 계산기에서 u128을 u64로 캐스팅할 때의 조용한 잘림(Silent truncation)

**설명:** `utils::token_calculator::calculate_liquidity_token_amount` 함수는 `u128`로 출력을 계산하고 일반 `as u64` 캐스트로 반환합니다:
```rust
let result = /* u128 math */;
Ok(result as u64)
```
`result`가 `u128`에는 맞지만 `u64::MAX`를 초과하면, 캐스트는 에러 없이 상위 비트를 잘라냅니다. 그러면 함수는 의도한 것보다 훨씬 작은 숫자를 보고합니다. 견적(quote) 경로와 `redeem` 모두 이 헬퍼를 사용하므로, 잘림으로 인해 조용히 과소 지불될 수 있습니다.

**파급력:** 조용한 과소 지불 및 잘못된 회계.

**권장 완화 방법:** 손실이 발생할 수 있는 캐스트를 확인된 변환으로 교체하십시오:
```rust
use core::convert::TryFrom;

let result_u64 = u64::try_from(result)
    .map_err(|_| SecuritizeOffRampError::Overflow)?;
Ok(result_u64)
```

**Securitize:** [7172884](https://github.com/securitize-io/bc-solana-redemption-sc/commit/71728848f7ae01c3b686b343210bd6ae3143ab85)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)

### 수수료 징수자(Fee collector) 필드의 지갑 대 토큰 계정 모호성

**설명:** `FeeManager::collector`는 지갑처럼 이름이 붙여진 `Pubkey`이지만, 이를 사용하는 모든 곳에서는 유동성 민트에 대한 **토큰 계정 주소**를 예상합니다. `initialize` 및 `update_fee_manager`에서 `fee_collector_ta.address == fee_manager.fee_collector()`를 요구하고 `token::mint = liquidity_token_mint`를 강제합니다.

통합자가 토큰 계정 pubkey 대신 지갑 pubkey로 `collector`를 설정하면, 명령어가 실패하거나 환경 전반에 걸쳐 수수료가 잘못 라우팅됩니다.

```rust
/// 수수료 전략을 위한 수수료 관리자 열거형
pub enum FeeManager {
    /// Mbps 기반 수수료 관리자
    MbpsFeeManager(MbpsFeeManager),
}
```
```rust
/// Mbps 기반 수수료 관리자 (베이시스 포인트)
pub struct MbpsFeeManager {
    /// 수수료 분자 (bps)
    pub numerator: u32,
    /// 수수료 징수자 주소
    pub collector: Pubkey,
}
```

**권장 완화 방법:** 의도를 반영하도록 필드 이름을 `collector_token_account`로 변경하십시오.

**Securitize:** [ab7f4d2](https://github.com/securitize-io/bc-solana-redemption-sc/commit/ab7f4d2f9110df2bf37ec1d2c8f7e2f4b1545310)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### 2단계 소유권 및 권한 이전 검증 누락

**설명:** `change_admin_handler`와 `change_liquidity_withdraw_authority_handler`는 모두 제안된 새 키의 확인을 요구하지 않고 단일 트랜잭션에서 중요한 제어 필드(`off_ramp_state.admin` 및 `off_ramp_state.liquidity_withdraw_authority`)를 직접 재할당합니다. 이 1단계 전송 모델은 우발적인 잘못된 구성이나 악성 키 주입의 위험을 증가시킵니다. 또한 두 함수 모두 기본 제로 주소(`Pubkey::default()`) 할당에 대해 검증하지 않아, 사용할 수 없는 권한을 할당하여 시스템을 영구적으로 잠글 수 있습니다.

**파급력:** 특권 서명자가 실수로 또는 악의적으로 새 권한을 기본 주소로 설정하면 관리 또는 유동성 인출 권한이 영구적으로 손실될 수 있습니다. 또한 2단계 수락 프로세스가 없으면 의도된 새 소유자의 동의 없이 일방적인 이전이 가능해져 운영 안전성이 저하되고 잠재적인 거버넌스 분쟁이 발생할 수 있습니다.

**권장 완화 방법:**
- 현재 소유자가 새 권한을 제안하고 제안된 권한이 완료 전에 명시적으로 수락해야 하는 2단계 전송 프로세스를 도입하십시오.
- 또한 제로 주소에 대한 할당을 방지하기 위해 기본 키가 아닌지 검증을 시행하십시오.

**Securitize:** 인지함.

### 유동성 볼트 잔액 검증 누락

**설명:** `withdraw_liquidity_handler`에서 유동성 토큰은 `liquidity_token_vault`에서 인출자의 연관 토큰 계정으로 전송됩니다. 명령어는 인출 금액이 0보다 크고 시스템이 일시 중지되지 않았는지 확인하지만, 전송을 시도하기 전에 볼트가 실제로 최소 `amount` 토큰을 보유하고 있는지 확인하지 않습니다.
이 누락은 가용 볼트 잔액을 초과하는 인출 시도를 허용할 수 있으며, 토큰 프로그램 구현에 따라 실패한 트랜잭션이나 의도하지 않은 프로그램 동작으로 이어질 수 있습니다.

**파급력:** 볼트에 요청한 것보다 적은 토큰이 포함되어 있으면 런타임에 전송이 실패하여 불필요한 트랜잭션 실패가 발생할 수 있습니다.

**권장 완화 방법:** 전송을 실행하기 전에 `liquidity_token_vault.amount >= amount`인지 확인하는 잔액 확인을 추가하십시오.

```diff
+ let liquidity_vault = &ctx.accounts.liquidity_token_vault;
+    require_gte!(
+      liquidity_vault.amount,
+      amount,
+    SecuritizeOffRampError::InsufficientLiquidity
+);
```

**Securitize:** [3163cd9](https://github.com/securitize-io/bc-solana-redemption-sc/commit/3163cd9a818e0d83222d9cc74edbd6a4e4fa2d1c)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
