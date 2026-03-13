**수석 감사자 (Lead Auditors)**

[Hans](https://twitter.com/hansfriese)

**보조 감사자 (Assisting Auditors)**

[0kage](https://twitter.com/0kage_eth)


---

# 발견 사항 (Findings)
## 중간 중요도 (Medium Risk)
### 비표준 ERC20 토큰이 지원되지 않음 (Non-standard ERC20 tokens are not supported)
**심각도:** 중간 (Medium)

**설명:** 프로토콜은 사용자가 예치할 수 있도록 `deposit()` 함수를 구현했습니다.
```solidity
DepositVault.sol
37:     function deposit(uint256 amount, address tokenAddress) public payable {
38:         require(amount > 0 || msg.value > 0, "Deposit amount must be greater than 0");
39:         if(msg.value > 0) {
40:             require(tokenAddress == address(0), "Token address must be 0x0 for ETH deposits");
41:             uint256 depositIndex = deposits.length;
42:             deposits.push(Deposit(payable(msg.sender), msg.value, tokenAddress));
43:             emit DepositMade(msg.sender, depositIndex, msg.value, tokenAddress);
44:         } else {
45:             require(tokenAddress != address(0), "Token address must not be 0x0 for token deposits");
46:             IERC20 token = IERC20(tokenAddress);
47:             token.safeTransferFrom(msg.sender, address(this), amount);
48:             uint256 depositIndex = deposits.length;
49:             deposits.push(Deposit(payable(msg.sender), amount, tokenAddress));//@audit-issue fee-on-transfer, rebalancing tokens will cause problems
50:             emit DepositMade(msg.sender, depositIndex, amount, tokenAddress);
51:
52:         }
53:     }
```
L49 행을 보면 프로토콜은 `amount`만큼의 토큰이 전송되었다고 가정함을 알 수 있습니다.
그러나 전송 수수료 토큰(fee-on-transfer tokens)이나 리밸런싱 토큰과 같은 일부 비표준 ERC20 토큰의 경우에는 이것이 사실이 아닙니다.
(비표준 weird ERC20 토큰에 대해서는 [여기](https://github.com/d-xo/weird-erc20)를 참조하세요)

예를 들어, 토큰 전송 시 수수료가 발생하는 경우 실제로 전송된 금액은 제공된 매개변수 `amount`보다 적을 것이며 `deposits`는 잘못된 상태 값을 갖게 됩니다. 현재 구현은 전액 인출만 허용하므로 이는 토큰이 컨트랙트에 영구적으로 잠기게 됨을 의미합니다.

**파급력:** 비표준 ERC20 토큰이 사용되는 경우 토큰이 컨트랙트에 영구적으로 잠길 수 있습니다.

**권장 완화 방안:**
- `Deposit` 구조체에 `balance`와 같은 또 다른 필드를 추가하는 것을 권장합니다.
- 사용자가 부분적으로 인출할 수 있도록 하고 성공적인 인출에 대해 `balance` 필드를 적절히 감소시키는 것을 권장합니다.
이러한 변경이 이루어질 경우 변경이 필요한 다른 부분이 있음에 유의하세요. 예를 들어, 인출 함수는 인출 금액이 원래 예치 금액과 같을 필요가 없도록 업데이트되어야 합니다.

**프로토콜:**
비표준 ERC-20 토큰을 지원하도록 컨트랙트가 업데이트되었습니다. 서명 로직을 복잡하게 만들 수 있으므로 사용자가 부분적으로 인출하는 것을 허용하지 않기로 결정했으며, 현재로서는 전액 인출만 실행할 수 있습니다.

**Cyfrin:** [405fa78](https://github.com/HyperGood/woosh-contracts/commit/405fa78a2c0cf8b8ab8943484cb95b5c8807cbfb) 커밋에서 검증되었습니다.

### transfer 대신 call 사용 (Use call instead of transfer)
**심각도:** 중간 (Medium)

**설명:** 두 인출 함수 모두에서 네이티브 ETH 인출을 위해 `transfer()`가 사용됩니다.
transfer() 및 send() 함수는 고정된 양인 2300 가스를 전달합니다. 역사적으로 이 함수들은 재진입 공격을 방지하기 위해 가치 전송에 사용하는 것이 권장되었습니다. 그러나 EVM 명령어의 가스 비용은 하드 포크 중에 크게 변경될 수 있으며, 이는 가스 비용에 대해 고정된 가정을 하는 이미 배포된 컨트랙트 시스템을 망가뜨릴 수 있습니다. 예를 들어 EIP 1884는 SLOAD 명령어의 비용 증가로 인해 여러 기존 스마트 컨트랙트를 망가뜨렸습니다.

**파급력:** 주소에 대해 더 이상 사용되지 않는(deprecated) transfer() 함수를 사용하면 다음과 같은 경우 트랜잭션이 필연적으로 실패합니다:
- 청구자 스마트 컨트랙트가 payable 함수를 구현하지 않는 경우.
- 청구자 스마트 컨트랙트가 2300 가스 단위 이상을 사용하는 payable 폴백을 구현하는 경우.
- 청구자 스마트 컨트랙트가 2300 가스 단위 미만이 필요한 payable 폴백 함수를 구현하지만 프록시를 통해 호출되어 호출의 가스 사용량이 2300을 초과하는 경우.

또한 일부 다중 서명 지갑의 경우 2300 가스 이상을 사용하는 것이 필수적일 수 있습니다.

**권장 완화 방안:** transfer() 대신 call()을 사용하세요.

**프로토콜:**
동의합니다. transfer는 스마트 컨트랙트 지갑에서 문제를 일으키고 있었습니다.

**Cyfrin:** [7726ae7](https://github.com/HyperGood/woosh-contracts/commit/7726ae72118cfdf91ceb9129e36662f69f4d42de) 커밋에서 검증되었습니다.

## 낮은 중요도 (Low Risk)
### deposit 함수가 CEI 패턴을 따르지 않음 (The deposit function is not following CEI pattern)
**심각도:** 낮음 (Low)

**설명:** 프로토콜은 사용자가 예치할 수 있도록 `deposit()` 함수를 구현했습니다.
```solidity
DepositVault.sol
37:     function deposit(uint256 amount, address tokenAddress) public payable {
38:         require(amount > 0 || msg.value > 0, "Deposit amount must be greater than 0");
39:         if(msg.value > 0) {
40:             require(tokenAddress == address(0), "Token address must be 0x0 for ETH deposits");
41:             uint256 depositIndex = deposits.length;
42:             deposits.push(Deposit(payable(msg.sender), msg.value, tokenAddress));
43:             emit DepositMade(msg.sender, depositIndex, msg.value, tokenAddress);
44:         } else {
45:             require(tokenAddress != address(0), "Token address must not be 0x0 for token deposits");
46:             IERC20 token = IERC20(tokenAddress);
47:             token.safeTransferFrom(msg.sender, address(this), amount);//@audit-issue against CEI pattern
48:             uint256 depositIndex = deposits.length;
49:             deposits.push(Deposit(payable(msg.sender), amount, tokenAddress));
50:             emit DepositMade(msg.sender, depositIndex, amount, tokenAddress);
51:
52:         }
53:     }
```
L47 행을 보면 토큰 전송이 프로토콜의 회계 상태를 업데이트하기 전에 발생하여 CEI 패턴에 위배됨을 알 수 있습니다.
프로토콜이 모든 ERC20 토큰을 지원하려고 하기 때문에 훅이 있는 토큰(예: ERC777)은 재진입에 악용될 수 있습니다.
이로 인해 명시적인 손실을 초래하는 익스플로잇을 확인할 수는 없지만, 가능한 재진입 공격을 방지하기 위해 CEI 패턴을 따르는 것이 여전히 강력히 권장됩니다.

**권장 완화 방안:** `deposits` 상태를 업데이트한 후 토큰 전송을 처리하세요.

**프로토콜:**
[7726ae7](https://github.com/HyperGood/woosh-contracts/commit/7726ae72118cfdf91ceb9129e36662f69f4d42de) 커밋에서 수정되었습니다.

**Cyfrin:** 우리는 최종 커밋 [b21d23e](https://github.com/HyperGood/woosh-contracts/commit/b21d23e661b0f25f0e757dc00ee90e4464730b1b)에서 프로토콜이 여전히 CEI 패턴을 따르지 않는다는 점에 주목합니다.
그러나 비표준 ERC20 토큰을 지원하기 위한 [405fa78](https://github.com/HyperGood/woosh-contracts/commit/405fa78a2c0cf8b8ab8943484cb95b5c8807cbfb)의 수정으로 인해 CEI 패턴을 따르는 것이 불가능하다는 점도 인정합니다.
현재 구현에 대해 이 문제로 인한 재진입 공격 벡터가 없음을 확인했습니다. 프로토콜이 향후 `deposits` 상태와 관련된 업그레이드를 의도하는 경우 가능한 재진입 공격을 방지하기 위해 [OpenZeppelin의 ReentrancyGuard](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/8a0b7bed82d6b8053872c3fd40703efd58f5699d/contracts/security/ReentrancyGuard.sol#L22)를 사용하는 것을 권장합니다.
`deposits` 상태가 다른 목적으로 활용될 경우 CEI 패턴의 부족은 여전히 [읽기 전용 재진입](https://officercia.mirror.xyz/DBzFiDuxmDOTQEbfXhvLdK0DXVpKu1Nkurk0Cqk3QKc) 측면에서 문제가 될 수 있음을 유의하십시오.

## 정보성 발견 사항 (Informational Findings)
### 비표준 nonce 사용 (Nonstandard usage of nonce)
**심각도:** 정보성 (Informational)

**설명:** 프로토콜은 두 가지 인출 함수 `withdrawDeposit()`과 `withdraw()`를 구현했습니다.
`withdrawDeposit()` 함수는 예금자 자신이 사용하도록 설계된 반면, `withdraw()` 함수는 예금자의 서명을 가진 사람이면 누구나 사용하도록 설계되었습니다.
`withdraw()` 함수에는 `nonce` 매개변수가 있지만 이 매개변수의 사용법은 nonce의 일반적인 의미와 일치하지 않습니다.
```solidity
DepositVault.sol
59:     function withdraw(uint256 amount, uint256 nonce, bytes memory signature, address payable recipient) public {
60:         require(nonce < deposits.length, "Invalid deposit index");
61:         Deposit storage depositToWithdraw = deposits[nonce];//@audit-info non aligned with common understanding of nonce
62:         bytes32 withdrawalHash = getWithdrawalHash(Withdrawal(amount, nonce));
63:         address signer = withdrawalHash.recover(signature);
64:         require(signer == depositToWithdraw.depositor, "Invalid signature");
65:         require(!usedWithdrawalHashes[withdrawalHash], "Withdrawal has already been executed");
66:         require(amount == depositToWithdraw.amount, "Withdrawal amount must match deposit amount");
67:
68:         usedWithdrawalHashes[withdrawalHash] = true;
69:         depositToWithdraw.amount = 0;
70:
71:         if(depositToWithdraw.tokenAddress == address(0)){
72:             recipient.transfer(amount);
73:         } else {
74:             IERC20 token = IERC20(depositToWithdraw.tokenAddress);
75:             token.safeTransfer(recipient, amount);
76:         }
77:
78:         emit WithdrawalMade(recipient, amount);
79:     }
```
일반적인 사용법에서 nonce는 EOA의 최신 트랜잭션을 추적하는 데 사용되며 일반적으로 사용자의 트랜잭션에서 증가합니다. 서명자에 의한 이전 서명을 무효화하는 데 효과적으로 사용될 수 있습니다.
그러나 현재 구현을 보면 `nonce` 매개변수는 단순히 특정 인덱스의 `deposit`을 참조하기 위한 인덱스로 사용됩니다.

이는 잘못된 명명이며 사용자를 혼란스럽게 할 수 있습니다.

**권장 완화 방안:** 프로토콜이 nonce를 사용하여 일종의 무효화 메커니즘을 제공하려는 경우 각 사용자에 대한 nonce를 저장하는 별도의 매핑이 있어야 합니다. 현재 nonce를 사용하여 서명을 생성할 수 있으며 예금자는 nonce를 증가시켜 이전 서명을 무효화할 수 있어야 합니다. 또한 재생 공격(replay attack)을 방지하기 위해 `withdraw()` 호출이 성공할 때마다 nonce를 증가시켜야 합니다.
이 수정 조치를 통해 해시는 항상 최신 nonce를 사용하여 결정되고 nonce가 자동으로 무효화(성공적인 호출 시 증가하기 때문)되므로 매핑 `usedWithdrawalHashes`를 완전히 제거할 수 있습니다.

이것이 프로토콜이 의도한 바가 아니라면, `withdrawDeposit()` 함수에 구현된 것처럼 nonce 매개변수의 이름을 `depositIndex`로 변경할 수 있습니다.

**클라이언트:**
[b21d23e](https://github.com/HyperGood/woosh-contracts/commit/b21d23e661b0f25f0e757dc00ee90e4464730b1b) 커밋에서 수정되었습니다.

**Cyfrin:** 검증됨.

### 인출 함수의 불필요한 매개변수 amount (Unnecessary parameter amount in withdraw function)

**심각도:** 정보성 (Informational)

**설명:** `withdraw()` 함수에는 `amount` 매개변수가 있지만 이 매개변수의 필요성을 이해할 수 없습니다.
L67 행에서 amount는 전체 예치 금액과 같아야 합니다. 이는 사용자가 인출 금액을 선택할 수 있는 유연성이 없다는 것을 의미하며, 결국 매개변수가 전혀 필요하지 않다는 것을 의미합니다.
```solidity
DepositVault.sol
59:     function withdraw(uint256 amount, uint256 nonce, bytes memory signature, address payable recipient) public {
60:         require(nonce < deposits.length, "Invalid deposit index");
61:         Deposit storage depositToWithdraw = deposits[nonce];
62:         bytes32 withdrawalHash = getWithdrawalHash(Withdrawal(amount, nonce));
63:         address signer = withdrawalHash.recover(signature);
64:         require(signer == depositToWithdraw.depositor, "Invalid signature");
65:         require(!usedWithdrawalHashes[withdrawalHash], "Withdrawal has already been executed");
66:         require(amount == depositToWithdraw.amount, "Withdrawal amount must match deposit amount");//@audit-info only full withdrawal is allowed
67:
68:         usedWithdrawalHashes[withdrawalHash] = true;
69:         depositToWithdraw.amount = 0;
70:
71:         if(depositToWithdraw.tokenAddress == address(0)){
72:             recipient.transfer(amount);
73:         } else {
74:             IERC20 token = IERC20(depositToWithdraw.tokenAddress);
75:             token.safeTransfer(recipient, amount);
76:         }
77:
78:         emit WithdrawalMade(recipient, amount);
79:     }
```

**권장 완화 방안:** 프로토콜이 전액 인출만 허용하려는 경우 이 매개변수를 완전히 제거할 수 있습니다(가스 절약에도 도움이 됨).
불필요한 매개변수는 함수의 복잡성을 증가시키고 오류 발생 가능성을 높입니다.

**클라이언트:**
동의합니다. 전액 인출만 허용됩니다. 제거되었습니다.

**Cyfrin:** [b21d23e](https://github.com/HyperGood/woosh-contracts/commit/b21d23e661b0f25f0e757dc00ee90e4464730b1b) 커밋에서 검증되었습니다.

### 내부적으로 사용되지 않는 함수는 external로 표시될 수 있음 (Functions not used internally could be marked external)

**심각도:** 정보성 (Informational)

**설명:** 적절한 가시성 수정자를 사용하는 것은 의도하지 않은 함수 액세스를 방지하기 위한 좋은 관행입니다.
또한 함수를 `public` 대신 `external`로 표시하면 가스를 절약할 수 있습니다.

```solidity
File: DepositVault.sol

37:     function deposit(uint256 amount, address tokenAddress) public payable

59:     function withdraw(uint256 amount, uint256 nonce, bytes memory signature, address payable recipient) public

81:     function withdrawDeposit(uint256 depositIndex) public
```

**권장 완화 방안:** 내부적으로 사용되지 않는 함수에 대해 가시성 수정자를 `external`로 변경하는 것을 고려하십시오.

**클라이언트:**
수정됨.

**Cyfrin:** [b21d23e](https://github.com/HyperGood/woosh-contracts/commit/b21d23e661b0f25f0e757dc00ee90e4464730b1b) 커밋에서 검증되었습니다.

## 가스 최적화 (Gas Optimizations)
### `address(0)` 확인을 위해 어셈블리 사용 (Use assembly to check for `address(0)`)
인스턴스당 6 가스 절약

```solidity
File: DepositVault.sol

40:             require(tokenAddress == address(0), "Token address must be 0x0 for ETH deposits");

45:             require(tokenAddress != address(0), "Token address must not be 0x0 for token deposits");

71:         if(depositToWithdraw.tokenAddress == address(0)){

90:         if(depositToWithdraw.tokenAddress == address(0)){

```
**클라이언트:**
수정됨.

**Cyfrin:** [b21d23e](https://github.com/HyperGood/woosh-contracts/commit/b21d23e661b0f25f0e757dc00ee90e4464730b1b) 커밋에서 검증되었습니다.

### 변경되지 않는 함수 인수에 메모리 대신 calldata 사용 (Use calldata instead of memory for function arguments that do not get mutated)
가능한 경우 데이터 유형을 `memory` 대신 `calldata`로 표시하십시오. 이렇게 하면 데이터가 자동으로 메모리에 로드되지 않습니다. 함수에 전달된 데이터가 변경될 필요가 없는 경우(예: 배열의 값 업데이트) `calldata`로 전달할 수 있습니다. 한 가지 예외는 인수가 `memory` 스토리지를 지정하는 인수를 취하는 다른 함수로 나중에 전달되어야 하는 경우입니다.

```solidity
File: DepositVault.sol

55:     function getWithdrawalHash(Withdrawal memory withdrawal) public view returns (bytes32)

59:     function withdraw(uint256 amount, uint256 nonce, bytes memory signature, address payable recipient) public

```
**클라이언트:**
인정함.

**Cyfrin:** 인정함.

### 사용자 정의 오류 사용 (Use Custom Errors)

오류 문자열을 사용하는 대신 배포 및 런타임 비용을 줄이려면 [사용자 정의 오류(Custom Errors)](https://blog.soliditylang.org/2021/04/21/custom-errors/)를 사용해야 합니다. 이렇게 하면 배포 및 런타임 비용을 모두 절약할 수 있습니다.

```solidity
File: DepositVault.sol

38:         require(amount > 0 || msg.value > 0, "Deposit amount must be greater than 0");

40:             require(tokenAddress == address(0), "Token address must be 0x0 for ETH deposits");

45:             require(tokenAddress != address(0), "Token address must not be 0x0 for token deposits");

60:         require(nonce < deposits.length, "Invalid deposit index");

64:         require(signer == depositToWithdraw.depositor, "Invalid signature");

65:         require(!usedWithdrawalHashes[withdrawalHash], "Withdrawal has already been executed");

66:         require(amount == depositToWithdraw.amount, "Withdrawal amount must match deposit amount");

82:         require(depositIndex < deposits.length, "Invalid deposit index");

84:         require(depositToWithdraw.depositor == msg.sender, "Only the depositor can withdraw their deposit");

85:         require(depositToWithdraw.amount > 0, "Deposit has already been withdrawn");

```
**클라이언트:**
수정됨.

**Cyfrin:** [b21d23e](https://github.com/HyperGood/woosh-contracts/commit/b21d23e661b0f25f0e757dc00ee90e4464730b1b) 커밋에서 검증되었습니다.


### 부호 없는 정수 비교에 > 0 대신 != 0 사용 (Use != 0 instead of > 0 for unsigned integer comparison)

```solidity
File: DepositVault.sol

38:         require(amount > 0 || msg.value > 0, "Deposit amount must be greater than 0");

38:         require(amount > 0 || msg.value > 0, "Deposit amount must be greater than 0");

39:         if(msg.value > 0)

85:         require(depositToWithdraw.amount > 0, "Deposit has already been withdrawn");

```

**클라이언트:**
수정됨.

**Cyfrin:** [b21d23e](https://github.com/HyperGood/woosh-contracts/commit/b21d23e661b0f25f0e757dc00ee90e4464730b1b) 커밋에서 검증되었습니다.

