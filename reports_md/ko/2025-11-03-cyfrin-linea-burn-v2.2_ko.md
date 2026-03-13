**수석 감사관**

[0xStalin](https://x.com/0xStalin)

[MrPotatoMagic](https://x.com/MrPotatoMagic)

**보조 감사관**



---

# 발견 사항 (Findings)
## 낮음 위험 (Low Risk)


### ExactInputSingleParams의 틱 간격(Tick spacing) 타입 불일치

**설명:** V3DexSwap은 [Ramses V3 Swap Router](https://lineascan.build/address/0x8BE024b5c546B5d45CbB23163e1a4dca8fA5052A#code)를 사용하여 WETH를 Linea로 스왑합니다.

그러나 exactInputSingle 함수에 매개변수로 전달되는 ExactInputSingleParams 구조체 정의가 V3DexSwap과 라우터 계약 간에 다릅니다.

**V3SwapDex 정의**:
```solidity
struct ExactInputSingleParams {
    address tokenIn;
    address tokenOut;
    uint24 tickSpacing; <<
    address recipient;
    uint256 deadline;
    uint256 amountIn;
    uint256 amountOutMinimum;
    uint160 sqrtPriceLimitX96;
  }
```

**Ramses Router**:

```solidity
struct ExactInputSingleParams {
        address tokenIn;
        address tokenOut;
        int24 tickSpacing; <<
        address recipient;
        uint256 deadline;
        uint256 amountIn;
        uint256 amountOutMinimum;
        uint160 sqrtPriceLimitX96;
    }
```

보시다시피 tickSpacing 멤버의 타입이 `uint24`와 `int24`로 서로 다릅니다.

**영향:** 이로 인해 POOL_TICK_SPACING 값이 `type(int24).max` 즉 8388607보다 큰 경우, 라우터에서 음수 값으로 잘못 해석되어 해당 풀이 존재하지 않으면 되돌려지거나(revert) 누구나 RamsesV3에서 프론트런(frontrun)하여 생성할 수 있는 의도하지 않은 풀을 통해 스왑이 성공할 수 있습니다.

**개념 증명 (Proof of Concept):** **권장 완화 방법:**
V3DexSwap의 구조체 정의를 업데이트하여 tickSpacing에 `uint24` 대신 `int24`를 사용하십시오.

**Linea:** 커밋 [be1cbc](https://github.com/Consensys/linea-monorepo/pull/1620/commits/be1cbce5ad0410d004e09d0f559522e4eee22daa)에서 수정됨.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)


### `V3DexSwap.swap`의 일관되지 않은 마감 기한(deadline) 확인

**설명:** V3DexSwap의 swap() 함수는 `block.timestamp`가 `_deadline`보다 엄격히 작은지 확인하는 아래 스니펫의 require 확인을 구현합니다.

```solidity
require(_deadline > block.timestamp, DeadlineInThePast());
```

그러나 [Ramses V3 Swap Router](https://lineascan.build/address/0x8BE024b5c546B5d45CbB23163e1a4dca8fA5052A#code)는 `block.timestamp`가 마감 기한과 같을 때에도 스왑이 발생하도록 허용합니다.

```solidity
modifier checkDeadline(uint256 deadline) {
        if (_blockTimestamp() > deadline) revert Old();
        _;
    }
```

**영향:** 이러한 불일치로 인해 `_deadline`이 `V3DexSwap.swap` 함수에 `block.timestamp`로 전달되면 라우터에서 허용되는 유효한 값임에도 불구하고 호출이 되돌려집니다.

**개념 증명 (Proof of Concept):** **권장 완화 방법:**

**Linea:** 커밋 [e531e7](https://github.com/Consensys/linea-monorepo/pull/1620/commits/be1cbce5ad0410d004e09d0f559522e4eee22daa)에서 수정됨.

**Cyfrin:** 확인됨.


### 배포된 프록시가 이미 초기화되었으므로 `RollupRevenueVault::initialize`의 불필요한 구현

**설명:** `RollupRevenueVault` 계약은 [프록시](https://lineascan.build/address/0xfd5fb23e06e46347d8724486cdb681507592e237)의 구현을 업그레이드하는 데 사용됩니다.
프록시는 이미 초기화되어 있으므로 `_initialized` 플래그가 이미 1로 설정되어 있습니다. 즉, 업그레이드 후에도 `initializer` 수정자(modifier)를 호출하는 함수는 다시 실행될 수 없습니다.
이는 새 구현의 값을 초기화하는 유일한 대안이 `RollupRevenueVault::initializeRolesAndStorageVariables` 함수에 의해 호출되는 `reinitializer` 수정자를 호출하는 것임을 의미합니다.
```solidity
    function initialize(
        ...
@>  ) external initializer {
        ...
    }

    function initializeRolesAndStorageVariables(
        ...
@>  ) external reinitializer(2) {
        ...
        );
    }
```

**개념 증명 (Proof of Concept):** 다음 PoC를 실행하여 업그레이드가 `initializeRolesAndStorageVariables` 함수를 호출할 때만 작동하고 `initialize` 함수를 호출할 때는 되돌려지는지 확인하십시오.
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.30;

import "forge-std/Test.sol";
import {
    TransparentUpgradeableProxy,
    ITransparentUpgradeableProxy
} from "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";

import {RollupRevenueVault} from "src/operational/RollupRevenueVault.sol";

import {Vm} from "forge-std/Vm.sol";

contract RollupRevenueVaultUpgradeSimulationTest is Test {
    ITransparentUpgradeableProxy public proxy;
    RollupRevenueVault public newImpl;
    RollupRevenueVault public vaultUpgraded; // Proxy cast to upgraded interface

    bytes32 ADMIN_SLOT = 0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103;
    bytes32 IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    // Existing proxy address on Linea
    address public proxyAddr = 0xFD5FB23e06e46347d8724486cDb681507592e237;

    // Fork URL for Linea mainnet
    string public lineaRpc = "https://linea-mainnet.g.alchemy.com/v2/<ALCHEMY_API_KEY>";

    // Variables for initialization
    address public admin;
    address public invoiceSubmitter;
    address public burner;
    address public invoicePaymentReceiver;
    address public tokenBridge = address(100);
    address public messageService = address(101);
    address public l1LineaTokenBurner = 0x000000000000000000000000000000000000dEaD;
    address public lineaToken = address(102);
    address public dex = address(103);
    uint256 public lastInvoiceDate;

    function setUp() public {
        // Create and select fork of Linea mainnet
        uint256 forkId = vm.createFork(lineaRpc);
        vm.selectFork(forkId);

        // Set the proxy
        proxy = ITransparentUpgradeableProxy(payable(proxyAddr));

        assertGt(address(proxy).balance, 0);

                //@audit-info => Doesn't work because the caller is not the admin :) !
                // admin = proxy.admin();

        // Get the current admin
        admin = getAdminAddress(address(proxy));

        // Set mock/test values for roles and receiver (in production, these would be specific addresses)
        invoiceSubmitter = vm.addr(1); // Placeholder, replace with actual if known
        burner = vm.addr(2); // Placeholder, replace with actual if known
        invoicePaymentReceiver = vm.addr(3); // Placeholder, replace with actual if known
        lastInvoiceDate = block.timestamp; // Use current timestamp for simulation

        // Deploy the new implementation
        newImpl = new RollupRevenueVault();

        // Encode the initialize calldata for upgrade (initialize())
        bytes memory initDataUpgrade_initialize = abi.encodeWithSelector(
            RollupRevenueVault.initialize.selector,
            lastInvoiceDate,
            admin, // Keep the same admin
            invoiceSubmitter,
            burner,
            invoicePaymentReceiver,
            tokenBridge,
            messageService,
            l1LineaTokenBurner,
            lineaToken,
            dex
        );

//@audit => upgrade fails when calling initialize because of the initializer modifier
        vm.prank(admin);
        vm.expectRevert();
        proxy.upgradeToAndCall(address(newImpl), initDataUpgrade_initialize);

        // Encode the reinitialization calldata for upgrade (reinitializer(2))
        bytes memory initDataUpgrade_reinitialize = abi.encodeWithSelector(
            RollupRevenueVault.initializeRolesAndStorageVariables.selector,
            lastInvoiceDate,
            admin, // Keep the same admin
            invoiceSubmitter,
            burner,
            invoicePaymentReceiver,
            tokenBridge,
            messageService,
            l1LineaTokenBurner,
            lineaToken,
            dex
        );

//@audit => upgrade succeeds when calling initializeRolesAndStorageVariables because of the reinitializer modifier
        vm.prank(admin);
        proxy.upgradeToAndCall(address(newImpl), initDataUpgrade_reinitialize);


        // Cast the proxy to the new interface
        vaultUpgraded = RollupRevenueVault(payable(address(proxy)));
    }

    function testUpgradeSimulation() public {
        // Verify the implementation has been updated
        assertEq(getImplementationAddress(address(proxy)), address(newImpl));

        // // Verify initialization values after upgrade
        assertEq(vaultUpgraded.lastInvoiceDate(), lastInvoiceDate);
        assertEq(vaultUpgraded.invoicePaymentReceiver(), invoicePaymentReceiver);
        assertEq(vaultUpgraded.dex(), dex);
        assertEq(address(vaultUpgraded.tokenBridge()), tokenBridge);
        assertEq(address(vaultUpgraded.messageService()), messageService);
        assertEq(vaultUpgraded.l1LineaTokenBurner(), l1LineaTokenBurner);
        assertEq(vaultUpgraded.lineaToken(), lineaToken);

        // Verify invoice arrears starts at 0
        assertEq(vaultUpgraded.invoiceArrears(), 0);

        // Verify constants
        assertEq(vaultUpgraded.ETH_BURNT_PERCENTAGE(), 20);

        // Verify the proxy still holds its balance (should be unchanged)
        assertGt(address(proxy).balance, 0); // From explorer, ~198 ETH
    }

    function getAdminAddress(address proxy) internal view returns (address) {
        address CHEATCODE_ADDRESS = 0x7109709ECfa91a80626fF3989D68f67F5b1DD12D;
        Vm vm = Vm(CHEATCODE_ADDRESS);

        bytes32 adminSlot = vm.load(proxy, ADMIN_SLOT);
        return address(uint160(uint256(adminSlot)));
    }

    function getImplementationAddress(address proxy) internal view returns (address) {
        address CHEATCODE_ADDRESS = 0x7109709ECfa91a80626fF3989D68f67F5b1DD12D;
        Vm vm = Vm(CHEATCODE_ADDRESS);

        bytes32 implementationSlot = vm.load(proxy, IMPLEMENTATION_SLOT);
        return address(uint160(uint256(implementationSlot)));
    }
}
```

**권장 완화 방법:** `RollupRevenueVault::initialize`를 제거하십시오. 업그레이드를 위해 값을 다시 초기화하는 데는 `RollupRevenueVault::initializeRolesAndStorageVariables`만 필요합니다.

**Linea:** [PR 1604](https://github.com/Consensys/linea-monorepo/pull/1604)에서 수정됨.

**Cyfrin:** 확인됨. initialize 함수가 제거되었습니다.


### `RollupRevenueVault`가 0 ETH를 받을 때 `EthReceived` 이벤트 방출

**설명:** [`RollupRevenueVault::receive`](https://github.com/Consensys/linea-monorepo/blob/8285efababe0689aec5f0a21a28212d9d22df22e/contracts/src/operational/RollupRevenueVault.sol#L292-L294) && [`RollupRevenueVault:fallback`](https://github.com/Consensys/linea-monorepo/blob/8285efababe0689aec5f0a21a28212d9d22df22e/contracts/src/operational/RollupRevenueVault.sol#L285-L287) 함수는 실제로 계약에 전송된 네이티브 토큰이 있는지 여부와 관계없이 호출될 때마다 `EthReceived` 이벤트를 방출합니다.

특히 `fallback()`의 경우, ABI에 지정된 함수와 일치하지 않는 호출을 함수가 포착할 때마다 이벤트가 방출됩니다.


**권장 완화 방법:** `fallback` 또는 `receive` 함수에서 네이티브 토큰을 받지 못한 경우 이벤트 방출을 건너뛰는 것을 고려하십시오.

**Linea:** [PR 1604](https://github.com/Consensys/linea-monorepo/pull/1604)에서 수정됨.

**Cyfrin:** 확인됨. `receive()` 및 `fallback()` 함수 모두 `msg.value`가 0이면 되돌립니다(revert).


### 스왑에 대해 예상되는 `minAmountOut`보다 적은 LINEA 토큰을 받는 것을 방지하기 위한 유효성 검사 부족

**설명:** 덱스(dex) 스와퍼가 지정된 `minAmountOut`보다 적은 토큰을 받는 것을 방지하기 위한 [`V3DexSwap::swap` ](https://github.com/Consensys/linea-monorepo/blob/8285efababe0689aec5f0a21a28212d9d22df22e/contracts/src/operational/V3DexSwap.sol#L50-L74)에 대한 유효성 검사가 없습니다. 대부분의 라우터가 실제로 수신된 `amountOut`이 최소 `minAmountOut`이 되도록 강제하는 것은 사실입니다. 그러나 이 확인을 수행하는 라우터에 전적으로 의존하는 것은 덱스 스와퍼가 이 확인을 수행하지 않는 라우터와 작동하도록 업데이트되는 경우 잠재적인 문제를 야기합니다.

**권장 완화 방법:** 스왑에 대해 수신된 LINEA 토큰이 최소 예상 `minAmountOut`인지 확인하는 것을 고려하십시오.

**Linea:** [PR 1604](https://github.com/Consensys/linea-monorepo/pull/1604)에서 수정됨.

**Cyfrin:** 확인됨. 호출자가 최소한 지정된 `_minLineaOut`의 LINEA 토큰을 받았는지 확인하는 체크가 추가되었습니다.


### 매개변수 초기화 시 이벤트 방출 없음

**설명:** [`L1LineaTokenBurner`](https://github.com/Consensys/linea-monorepo/blob/8285efababe0689aec5f0a21a28212d9d22df22e/contracts/src/operational/L1LineaTokenBurner.sol#L21-L27) 및 [`V3DexSwap` ](https://github.com/Consensys/linea-monorepo/blob/8285efababe0689aec5f0a21a28212d9d22df22e/contracts/src/operational/V3DexSwap.sol#L31-L41) 계약의 생성자에는 초기화된 매개변수 값을 기록하기 위한 이벤트 방출이 없습니다.
[`RollupRevenueVault`](https://github.com/Consensys/linea-monorepo/blob/8285efababe0689aec5f0a21a28212d9d22df22e/contracts/src/operational/RollupRevenueVault.sol#L98-L160)에서 값을 다시 초기화할 때도 동일한 상황이 발생합니다.

**권장 완화 방법:** 초기화된 매개변수 값을 기록하기 위해 이벤트를 방출하는 것을 고려하십시오.

**Linea:** [PR 1604](https://github.com/Consensys/linea-monorepo/pull/1604)에서 수정됨.

**Cyfrin:** 확인됨.


### `_minLineaOut > 0` 조건은 악의적인 BURNER_ROLE 동작에 대한 보호를 제공하지 않음

**설명:** RollupRevenueVault의 burnAndBridge 함수는 BURNER_ROLE만 호출할 수 있습니다. 이 호출 동안 BURNER_ROLE은 임의의 `_swapData`를 전달하여 V3DexSwap 계약을 호출할 수 있습니다. swap() 함수에는 다음 확인이 포함되어 있습니다:
```solidity
require(_minLineaOut > 0, ZeroMinLineaOutNotAllowed());
```

그러나 `_minLineaOut`을 1 wei로 전달할 수 있으므로 이 확인은 BURNER_ROLE의 악의적인 동작에 대한 보호를 제공하지 않습니다.

**영향:** 소각 및 브리지 메커니즘의 일부로 L1에 브리지하려는 의도인 ETH의 손실.

**개념 증명 (Proof of Concept):** **권장 완화 방법:**
`_minLineaOut`이 초과해서는 안 되는 스왑 금액에 대해 구성 가능한 슬리피지 퍼센트를 구현하거나 이 위험을 인정하는 것을 고려하는 것이 좋습니다.

**Linea:** 인지함.

**Cyfrin:** 인지함.


### submitInvoice()의 `_invoiceAmount != 0` 조건은 기존 부채 청산을 방해할 수 있음

**설명:** submitInvoice() 함수는 아래의 확인을 구현합니다:

```solidity
require(_invoiceAmount != 0, ZeroInvoiceAmount());
```

그러나 INVOICE_SUBMITTER 역할이 충분한 ETH 잔액을 사용할 수 있게 된 경우에만 `invoiceArrears`를 청산하려는 경우 그렇게 할 수 없습니다. 예를 들면 다음과 같습니다:

 - T1에서 계약에 충분한 네이티브 토큰 잔액이 없으므로 `invoiceArrears` = 1e18이라고 가정합니다.
 - T2에서 계약은 `invoiceArrears`에 저장된 기존 부채를 청산하는 데 사용할 수 있는 1e18 네이티브 토큰 잔액을 받습니다.
 - 그러나 submitInvoice()의 확인으로 인해 `_invoiceAmount`를 0으로 전달하여 `invoiceArrears`를 청산할 수 없습니다.

**영향:** INVOICE_SUBMITTER는 다음 송장 제출까지 기다릴 수 있지만, 이는 더 빨리 지불될 수 있었던 기존 부채의 지불을 지연시킵니다.

**개념 증명 (Proof of Concept):** **권장 완화 방법:**
이 동작을 인정하거나 다음 수정 사항 중 하나를 구현하는 것을 고려하십시오:
1. `_invoiceAmount != 0` 조건을 제거합니다.
2. `invoiceArrears` 청산을 허용하는 별도의 함수를 구현합니다.

**Linea:** [PR 1637](https://github.com/Consensys/linea-monorepo/pull/1637/files)에서 수정됨.

**Cyfrin:** 확인됨.


### 토큰 브리지 상태 일시 중지로 인해 소각 및 브리지 메커니즘이 지연될 수 있음

**설명:** RollupRevenueVault 계약은 BURNER_ROLE에 burnAndBridge() 함수를 제공합니다. 이 프로세스에서 Linea 토큰은 L1으로 브리지되고 그곳에서 소각됩니다.

브리징을 위해 `tokenBridge` 서비스가 사용되며, 이는 [일시 중지됨](https://github.com/tree/contract-freeze-2025-10-12/blob/503a36c900419e9c6df00f5dad3d4c54d83f0578/linea/contracts/src/bridging/token/TokenBridgeBase.sol#L210)으로 인해 되돌려질(revert) 수 있습니다.

**영향:** 소각 및 브리지 메커니즘 실행이 DOS(거부)될 수 있습니다.

**개념 증명 (Proof of Concept):** **권장 완화 방법:**
여기서 위험을 인정하고 그러한 시나리오를 처리하기 위한 적절한 조치를 취하도록 고려하십시오.

**Linea:** 인지함. 소각 작업을 일시 중지하면 되므로 이는 허용 가능합니다.

**Cyfrin:** 확인됨.


### updateInvoiceArrears() 함수는 `invoiceArrears`가 변경되지 않은 경우에도 `lastInvoiceDate`를 업데이트함

**설명:** updateInvoiceArrears() 함수를 사용하면 DEFAULT_ADMIN_ROLE이 `invoiceArrears` 변수와 `lastInvoiceDate`를 그에 따라 업데이트할 수 있습니다. 그러나 `invoiceArrears` 변수가 변경되지 않은 경우에도 `lastInvoiceDate`는 여전히 다른 타임스탬프로 업데이트될 수 있습니다.

```solidity
function updateInvoiceArrears(
    uint256 _newInvoiceArrears,
    uint256 _lastInvoiceDate
  ) external onlyRole(DEFAULT_ADMIN_ROLE) {
    require(_lastInvoiceDate >= lastInvoiceDate, InvoiceDateTooOld());

    invoiceArrears = _newInvoiceArrears;
    lastInvoiceDate = _lastInvoiceDate;

    emit InvoiceArrearsUpdated(_newInvoiceArrears, _lastInvoiceDate);
  }
```


**영향:** `invoiceArrears`가 업데이트되지 않은 경우에도 `lastInvoiceDate` 변수가 업데이트됩니다.

**개념 증명 (Proof of Concept):** **권장 완화 방법:**
`_newInvoiceArrears`가 `invoiceArrears`와 같지 않은지 확인하는 체크를 구현하여 이 동작을 허용하지 않는 것을 고려하십시오. 또는 그러한 동작이 의도된 경우 함수 이름을 `updateInvoiceArrearsAndLastInvoiceDate`로 변경하는 것을 고려하십시오.

**Linea:** 인지함. 이는 다양한 유연한 상황과 느린 인프라 청구 업데이트를 수정할 수 있는 방법을 설명합니다.

**Cyfrin:** 인지함.


### updateInvoiceArrears() 함수가 이전 값을 방출하지 않음

**설명:** updateInvoiceArrears() 함수는 이벤트에서 이전 값과 새 값을 모두 방출하는 다른 설정자(setter) 함수 updateL1LineaTokenBurner, updateDex 및 updateInvoicePaymentReceiver와의 일관성을 유지하기 위해 이전 `invoiceArrears` 및 `lastInvoiceDate` 값을 방출하는 것을 고려해야 합니다.

```solidity
function updateInvoiceArrears(
    uint256 _newInvoiceArrears,
    uint256 _lastInvoiceDate
  ) external onlyRole(DEFAULT_ADMIN_ROLE) {
    require(_lastInvoiceDate >= lastInvoiceDate, InvoiceDateTooOld());

    invoiceArrears = _newInvoiceArrears;
    lastInvoiceDate = _lastInvoiceDate;

    emit InvoiceArrearsUpdated(_newInvoiceArrears, _lastInvoiceDate);
  }
```

**영향:** **개념 증명 (Proof of Concept):**

**권장 완화 방법:** `InvoiceArrearsUpdated` 이벤트에서 이전 `invoiceArrears` 및 `lastInvoiceDate` 값을 방출하십시오.

**Linea:** 커밋 [6c65701](https://github.com/Consensys/linea-monorepo/pull/1620/commits/6c6570123e25d5e9ecb98df5b663f5030281f3bb)에서 수정됨.

**Cyfrin:** 확인됨.

\clearpage
## 가스 최적화 (Gas Optimization)


### 스왑 중 불필요한 ETH에서 WETH로의 변환

**설명:** V3DexSwap 계약에서 아래 스니펫에서 볼 수 있듯이 라우터를 통해 스왑을 처리하기 전에 ETH가 WETH로 변환됩니다.

```solidity
IWETH9(WETH_TOKEN).deposit{ value: msg.value }();
IWETH9(WETH_TOKEN).approve(ROUTER, msg.value);
```

그러나 라우터가 직접 ETH 대 Linea 스왑을 지원하므로 이는 필요하지 않습니다. 아래에서 관찰할 수 있듯이 exactInputSingle 함수는 함수 호출 시 직접 ETH 전송을 허용하도록 payable로 표시되어 있습니다. uniswapV3SwapCallback()이 pay() 함수를 호출할 때 라우터의 실행 경로에서 토큰이 WETH인 경우 라우터의 계약 잔액을 처리합니다.

```solidity
File: SwapRouter.sol

/// @inheritdoc ISwapRouter
    function exactInputSingle(
        ExactInputSingleParams calldata params
    ) external payable override checkDeadline(params.deadline) returns (uint256 amountOut) {


File: PeripheryPayments.sol

/// @param token The token to pay
    /// @param payer The entity that must pay
    /// @param recipient The entity that will receive payment
    /// @param value The amount to pay
    function pay(
        address token,
        address payer,
        address recipient,
        uint256 value
    ) internal {
        if (token == WETH9 && address(this).balance >= value) {
            // pay with WETH9
            IWETH9(WETH9).deposit{value: value}(); // wrap only what is needed to pay
            IWETH9(WETH9).transfer(recipient, value);
        } else if (payer == address(this)) {
            // pay with tokens already in the contract (for the exact input multihop case)
            TransferHelper.safeTransfer(token, recipient, value);
        } else {
            // pull payment
            TransferHelper.safeTransferFrom(token, payer, recipient, value);
        }
    }
```

**영향:** 이는 위험을 초래하지 않지만 스왑 중에 불필요한 가스 비용이 발생합니다.

**개념 증명 (Proof of Concept):** **권장 완화 방법:**
WETH로 변환하는 대신 스왑 중에 ETH를 직접 사용하는 것을 고려하십시오.

**Linea:** 커밋 [a0b875](https://github.com/Consensys/linea-monorepo/pull/1620/commits/a0b875154412d5f543ad44994ca464219957b30e) 및 [dab9ebfd](https://github.com/Consensys/linea-monorepo/pull/1604/commits/dab9ebfdf65be8fbfecb9e2bc79d62056a60218b#diff-f36dd3901bbfa7a03d4cb920bfd8350752560a511fd7daf4a4fad33c7bbe6105)에서 수정됨.

**Cyfrin:** 확인됨. 이제 DexSwap에는 ETH를 Linea로 직접 스왑하는 버전과 WETH를 Linea로 스왑하는 버전의 두 가지 버전이 있습니다. 둘 다 검증되었습니다.


### `L1LineaTokenBurner`에서 LINEA 토큰 공급을 동기화(sync)하기 위한 중복 호출

**설명:** `L1LineaTokenBurner::claimMessageWithProof`는 L2에 배포된 `RollupRevenueVault`의 브리지 및 소각 작업을 완료하는 역할을 합니다.

`L1LineaTokenBurner`는 L1에서 메시지를 청구하고, 브리지된 토큰을 받아 소각합니다. 또한 계약에 이미 있는 모든 토큰을 소각할 수도 있습니다.

중복성은 마지막 소각 이후 경과된 시간이나 소각된 LINEA 토큰의 양에 관계없이 `L1LineaTokenBurner::claimMessageWithProof`가 호출될 때마다 `LINEA_TOKEN::syncTotalSupplyToL2`를 호출하는 것입니다.

`LINEA_TOKEN::syncTotalSupplyToL2`는 언제든지 누구나 호출할 수 있고 함수가 현재 총 공급량을 사용한다는 점을 감안할 때, `L1LineaTokenBurner`는 호출할 때마다 L2 공급량을 동기화하지 않도록 최적화될 수 있습니다.

**권장 완화 방법:** `L1LineaTokenBurner::claimMessageWithProof`에서 `LINEA_TOKEN::syncTotalSupplyToL2`에 대한 호출을 제거하는 것을 고려하십시오. 대신 LINEA 토큰의 여러 소각을 `LINEA_TOKEN::syncTotalSupplyToL2`에 대한 단일 호출로 묶는 대안을 모색하십시오.
- L2 공급량을 언제 동기화해야 하는지 결정하는 기준을 정의하십시오. 이는 일정량의 LINEA 토큰이 소각된 후 또는 지정된 기간이 지난 후에 발생할 수 있습니다.

**Linea:** 커밋 [43ed33](https://github.com/Consensys/linea-monorepo/pull/1604/commits/43ed33d00873756bb13ab5de5adafeba61d1fd4b)에서 수정됨.

**Cyfrin:** 확인됨.

\clearpage