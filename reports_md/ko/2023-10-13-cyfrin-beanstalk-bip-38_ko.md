**수석 감사자 (Lead Auditors)**

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

[Carlos Amarante](https://twitter.com/carlitox477)

**보조 감사자 (Assisting Auditors)**


---

# 발견 사항 (Findings)
## 높은 중요도 (High Risk)

### BEAN:3CRV에서 BEAN:ETH로의 unripe LP 마이그레이션이 자본 재조달(recapitalization) 회계 오류를 고려하지 않음 (Migration of unripe LP from BEAN:3CRV to BEAN:ETH does not account for recapitalization accounting error)
**설명:** 전역 [`AppStorage::recapitalized`](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/AppStorage.sol#L485) 상태는 Fertilizer를 USDC로 구매하고 BEAN:3CRV LP를 위해 BEAN과 페어링했을 때 자본이 재조달된 달러 금액을 나타냅니다. 이 기초 유동성을 제거하고 unripe LP 마이그레이션 중에 3CRV를 WETH로 교환할 때 BCM은 약간의 슬리피지를 경험할 가능성이 매우 높습니다. 이는 스왑이 OTC 거래가 아닌 공개 시장에서 이루어지는 경우 더욱 그렇지만, 어느 쪽이든 결과 WETH(및 그에 따른 BEAN:ETH LP)의 달러 가치는 마이그레이션 전 BEAN:3CRV일 때보다 적을 가능성이 높습니다. 현재 [`UnripeFacet::addMigratedUnderlying`](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/barn/UnripeFacet.sol#L257)은 unripe LP의 기초가 되는 BEAN:ETH LP 토큰 잔액을 업데이트하여 마이그레이션을 완료하지만, 위에서 설명한 달러 가치의 변화는 고려하지 않습니다. 현재 구현을 기반으로 BCM은 더 적은 달러 가치를 전송하여 마이그레이션을 완료할 가능성이 매우 높지만 자본 재조달 상태는 동일하게 유지되어 [`LibUnripe::percentLPRecapped`](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/libraries/LibUnripe.sol#L30-L36) 및 `LibUnripe::add/removeUnderlying`의 불일치를 초래합니다. 이는 `LibUnripeConvert`에서 urBEAN ↔ urBEANETH 변환에 사용됩니다. 따라서 전역 자본 재조달 상태는 마이그레이션 완료 시 자본 재조달의 실제 달러 가치를 반영하도록 업데이트되어야 합니다.

**파급력:** Fertilizer 구매자에 의해 충분한 자금이 조달되면 불충분한 기초 BEAN:ETH LP로 자본 재조달이 완료된 것으로 간주될 수 있습니다. 이는 실제 자본 재조달 금액이 [`LibFertilizer::remainingRecapitalization`](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/libraries/LibFertilizer.sol#L159-L163)에서 총 달러 부채를 계산하는 데 사용되는 [`C::dollarPerUnripeLP`](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/C.sol#L190-L192)에 의해 지정된 것보다 적을 것이기 때문에 사용자 자금 손실에 해당합니다.

**권장 완화 방안:** 마이그레이션 완료 시점에 `s.recapitalized`를 새로운 BEAN:ETH LP의 오라클 USD 금액으로 재할당하십시오.

```diff
    function addMigratedUnderlying(address unripeToken, uint256 amount) external payable nonReentrant {
        LibDiamond.enforceIsContractOwner();
        IERC20(s.u[unripeToken].underlyingToken).safeTransferFrom(
            msg.sender,
            address(this),
            amount
        );
        LibUnripe.incrementUnderlying(unripeToken, amount);

+       uint256 recapitalized = amount.mul(LibEthUsdOracle.getEthUsdPrice()).div(1e18);
+       require(recapitalized != 0, "UnripeFacet: cannot calculate recapitalized");
+       s.recapitalized = s.recapitalized.add(recapitalized);
    }
```

**Beanstalk Farms:** 이것은 의도적인 것입니다. 슬리피지 비용은 Unripe LP 토큰 보유자에게 귀속됩니다. 이는 BIP 초안에 명시되어야 합니다.

**Cyfrin:** 인정함.


## 중간 중요도 (Medium Risk)

### FIFO의 마지막 요소가 지불되면 페그(peg) 위에서 `SeasonFacet::gm`에 대한 서비스 거부(DoS) 공격을 허용하는 새로운 Fertilizer ID의 불충분한 검증 (Insufficient validation of new Fertilizer IDs allow for a denial-of-service (DoS) attack on `SeasonFacet::gm` when above peg, once the last element in the FIFO is paid)

**설명:** Fertilizer NFT는 만료일이 없는 채권으로 해석될 수 있으며 Beans로 상환되며 이자(Humidity)를 포함합니다. 이 채권은 FIFO 목록에 배치되며 [2022년 4월 익스플로잇](https://docs.bean.money/almanac/farm/barn) 중에 도난당한 7,700만 달러의 유동성을 재조달하기 위한 것입니다. 1 Fertilizer는 1 USD 상당의 WETH로 구매할 수 있습니다. BIP-38 이전에는 이 구매가 USDC를 사용하여 이루어졌습니다.

각 fertilizer는 `s.bpf`에 의존하는 Id로 식별되며, 이는 Fertilizer당 지불된 Beans의 누적 금액을 나타냅니다. 이 값은 [`Sun::rewardToFertilizer`](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/sun/SeasonFacet/Sun.sol#L153)가 호출될 때마다 증가합니다. 이 함수는 Bean 가격이 페그보다 높을 경우 `SeasonFacet::gm`에 의해 호출됩니다. 따라서 Fertilizer ID는 [지불될 Beans의 양](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/libraries/LibFertilizer.sol#L64-L66) 외에도 [발행 순간의](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/libraries/LibFertilizer.sol#L45-L51) `s.bpf`에 따라 달라집니다.

FIFO 목록에는 다음 구성 요소가 있습니다:
* `s.fFirst`: 다음에 지불될 Fertilizer에 해당하는 Fertilizer Id.
* `s.fLast`: 마지막으로 지불될 Fertilizer인 가장 높은 활성 Fertilizer Id.
* `s.nextFid`: Fertilizer Id에서 Fertilizer id로의 매핑으로, [연결 리스트(linked list)](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/AppStorage.sol#L477-L477)의 다음 요소를 나타냅니다. Id가 0을 가리키면 다음 요소가 없는 것입니다.

이 FIFO 목록과 관련된 메서드는 다음과 같습니다:
`LibFertilizer::push`: FIFO 목록에 요소를 추가합니다.
`LibFertilizer::setNext`: 주어진 fertilizer id에 대해 목록의 다음 요소에 대한 포인터를 추가합니다.
`LibFertilizer::getNext`: 목록에서 다음 요소를 가져옵니다.

이 목록의 의도된 동작은 새 fertilizer가 새 Id로 발행될 때마다 목록 끝에 새 요소를 추가하는 것입니다. 목록에 대한 중간 추가는 이전에는 Beanstalk DAO에서만 허용되었지만, `FertilizerFacet::addFertilizerOwner`가 제거됨에 따라 현재 업그레이드에서는 이 기능이 더 이상 사용되지 않습니다.

*BEAN:3CRV MetaPool을 BEAN:ETH Well로 교체한 결과:*
이 업그레이드 전에는 `LibFertilizer::addUnderlying`의 Curve 의존성으로 인해 `LibFertilizer::addFertilizer`를 통한 0 Fertilizer 추가가 불가능했습니다:

```solidity
// Previous code

    function addUnderlying(uint256 amount, uint256 minAmountOut) internal {
        //...
        C.bean().mint(
            address(this),
            newDepositedBeans.add(newDepositedLPBeans)
        );

        // Add Liquidity
        uint256 newLP = C.curveZap().add_liquidity(
            C.CURVE_BEAN_METAPOOL, // where to add liquidity
            [
                newDepositedLPBeans, // BEANS to add
                0,
                amount, // USDC to add
                0
            ], // how much of each token to add
            minAmountOut // min lp ampount to receive
        ); // @audit-ok Does not admit depositing 0 --> https://etherscan.io/address/0x5F890841f657d90E081bAbdB532A05996Af79Fe6#code#L487

        // Increment underlying balances of Unripe Tokens
        LibUnripe.incrementUnderlying(C.UNRIPE_BEAN, newDepositedBeans);
        LibUnripe.incrementUnderlying(C.UNRIPE_LP, newLP);

        s.recapitalized = s.recapitalized.add(amount);
    }
```

그러나 Wells 통합과 관련된 의존성 변경으로 인해 이 제한은 더 이상 유지되지 않습니다:
```solidity
    function addUnderlying(uint256 usdAmount, uint256 minAmountOut) internal {
        AppStorage storage s = LibAppStorage.diamondStorage();
        // Calculate how many new Deposited Beans will be minted
        uint256 percentToFill = usdAmount.mul(C.precision()).div(
            remainingRecapitalization()
        );
        uint256 newDepositedBeans;
        if (C.unripeBean().totalSupply() > s.u[C.UNRIPE_BEAN].balanceOfUnderlying) {
            newDepositedBeans = (C.unripeBean().totalSupply()).sub(
                s.u[C.UNRIPE_BEAN].balanceOfUnderlying
            );
            newDepositedBeans = newDepositedBeans.mul(percentToFill).div(
                C.precision()
            );
        }

        // Calculate how many Beans to add as LP
        uint256 newDepositedLPBeans = usdAmount.mul(C.exploitAddLPRatio()).div(
            DECIMALS
        );

        // Mint the Deposited Beans to Beanstalk.
        C.bean().mint(
            address(this),
            newDepositedBeans
        );

        // Mint the LP Beans to the Well to sync.
        C.bean().mint(
            address(C.BEAN_ETH_WELL),
            newDepositedLPBeans
        );

        // @audit If nothing was previously deposited this function returns 0, IT DOES NOT REVERT
        uint256 newLP = IWell(C.BEAN_ETH_WELL).sync(
            address(this),
            minAmountOut
        );

        // Increment underlying balances of Unripe Tokens
        LibUnripe.incrementUnderlying(C.UNRIPE_BEAN, newDepositedBeans);
        LibUnripe.incrementUnderlying(C.UNRIPE_LP, newLP);

        s.recapitalized = s.recapitalized.add(usdAmount);
    }
```

새로운 통합은 0 Fertilizer 추가를 시도할 때 되돌리지(revert) 않으므로 이제 자체 참조 노드를 FIFO 목록 끝에 추가할 수 있습니다. 단, 이것이 `FertilizerFacet.mintFertilizer(0, 0, 0, mode)`를 두 번 호출하여 현재 시즌에 대해 발행된 첫 번째 Fertilizer NFT인 경우에만 가능합니다. 중복 id를 방지하기 위해 수행된 [검증](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/libraries/LibFertilizer.sol#L57-L58)은 주어진 Id에 대한 Fertilizer 양이 0으로 유지되므로 잘못 우회됩니다.

```solidity
    function push(uint128 id) internal {
        AppStorage storage s = LibAppStorage.diamondStorage();
        if (s.fFirst == 0) {
            // Queue is empty
            s.season.fertilizing = true;
            s.fLast = id;
            s.fFirst = id;
        } else if (id <= s.fFirst) {
            // Add to front of queue
            setNext(id, s.fFirst);
            s.fFirst = id;
        } else if (id >= s.fLast) { // @audit this block is entered twice
            // Add to back of queue
            setNext(s.fLast, id); // @audit the second time, a reference is added to the same id
            s.fLast = id;
        } else {
            // Add to middle of queue
            uint128 prev = s.fFirst;
            uint128 next = getNext(prev);
            // Search for proper place in line
            while (id > next) {
                prev = next;
                next = getNext(next);
            }
            setNext(prev, id);
            setNext(id, next);
        }
    }
```
처음에는 무해해 보일 수 있지만, 달리 덮어쓰지 않는 한 이 요소는 제거할 수 없습니다:

```solidity
    function pop() internal returns (bool) {
        AppStorage storage s = LibAppStorage.diamondStorage();
        uint128 first = s.fFirst;
        s.activeFertilizer = s.activeFertilizer.sub(getAmount(first)); // @audit getAmount(first) would return 0
        uint128 next = getNext(first);
        if (next == 0) { // @audit next != 0, therefore this conditional block is skipped
            // If all Unfertilized Beans have been fertilized, delete line.
            require(s.activeFertilizer == 0, "Still active fertilizer");
            s.fFirst = 0;
            s.fLast = 0;
            s.season.fertilizing = false;
            return false;
        }
        s.fFirst = getNext(first); // @audit this gets s.first again
        return true; // @audit always returns true for a self-referential node
    }
```

`LibFertilizer::pop`은 비료를 줄 때 [`Sun::rewardBeans`](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/sun/SeasonFacet/Sun.sol#L97)를 통해 호출되는 [`Sun::rewardToFertilizer`](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/sun/SeasonFacet/Sun.sol#L132-L150)에서 사용됩니다. 이 함수는 현재 Bean 가격이 페그보다 높은 경우 [`Sun::stepSun`](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/sun/SeasonFacet/Sun.sol#L73)을 통해 호출됩니다. 이 요소에 도달한다고 가정할 때 목록에서 마지막 요소를 팝(pop)하는 것을 방지함으로써 `while` 루프가 계속 실행되어 무한 루프가 발생하여 페그 위에서 [`SeasonFacet::gm`](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/sun/SeasonFacet/SeasonFacet.sol#L59)에 대한 서비스 거부가 발생합니다.

이 문제의 가장 주목할만한 점은 페그 위에 있고 이미 완전히 자본이 재조달된 상태에서 이 상태를 강제할 수 있다는 것입니다. 관련 Beans로 추가 Fertilizer를 발행하는 것이 불가능하다는 점을 감안할 때, 이는 BEAN 가격이 페그보다 높을 경우 자본 재조달이 완료되면 `SeasonFacet::gm`에 대해 DoS 공격을 수행할 수 있음을 의미합니다.

**파급력:** 완전히 자본이 재조달되거나 Fertilizer FIFO 목록의 마지막 요소에 도달하면 Bean 가격이 페그보다 높을 경우 `SeasonFacet::gm`에 대해 서비스 거부(DoS) 공격을 수행할 수 있습니다.

**개념 증명 (Proof of Concept):** [이 코드로 된 PoC](https://gist.github.com/carlitox477/1b0dde178288982f4e25d40b9e43e626)는 다음과 같이 실행할 수 있습니다:
1. `Beantalk/protocol/test/POCs/mint0Fertilizer.test.js` 파일 생성
2. `Beantalk/protocol`로 이동
3. `yarn test --grep "DOS last fertilizer payment through minting 0 fertilizers"` 실행

**권장 완화 방안:** 설명하기 복잡한 문제임에도 불구하고 해결책은 아래와 같이 `LibFertilizer::addFertilizer`에서 `>`를 `>=`로 대체하는 것만큼 간단합니다:

```diff
    function addFertilizer(
        uint128 season,
        uint256 fertilizerAmount,
        uint256 minLP
    ) internal returns (uint128 id) {
        AppStorage storage s = LibAppStorage.diamondStorage();

        uint128 fertilizerAmount128 = fertilizerAmount.toUint128();

        // Calculate Beans Per Fertilizer and add to total owed
        uint128 bpf = getBpf(season);
        s.unfertilizedIndex = s.unfertilizedIndex.add(
            fertilizerAmount.mul(bpf)
        );
        // Get id
        id = s.bpf.add(bpf);
        // Update Total and Season supply
        s.fertilizer[id] = s.fertilizer[id].add(fertilizerAmount128);
        s.activeFertilizer = s.activeFertilizer.add(fertilizerAmount);
        // Add underlying to Unripe Beans and Unripe LP
        addUnderlying(fertilizerAmount.mul(DECIMALS), minLP);
        // If not first time adding Fertilizer with this id, return
-       if (s.fertilizer[id] > fertilizerAmount128) return id;
+       if (s.fertilizer[id] >= fertilizerAmount128) return id; // prevent infinite loop in `Sun::rewardToFertilizer` when attempting to add 0 Fertilizer, which could DoS `SeasonFacet::gm` when recapitalization is fulfilled
        // If first time, log end Beans Per Fertilizer and add to Season queue.
        push(id);
        emit SetFertilizer(id, bpf);
    }
```

**Beanstalk Farms:** [4489cb8](https://github.com/BeanstalkFarms/Beanstalk/pull/655/commits/4489cb869b1a1f8a2535a04364460c79ffb75b11) 커밋 해시에서 `mintFertilizer` 함수에 > 0 검사를 추가했습니다.

**Cyfrin:** 인정함. Beanstalk Farms 팀은 `FertilizerFacet::mintFertilizer`에 검증을 추가하기로 했습니다. 이 대안은 제안된 대안보다 가스를 더 절약합니다. 그러나 이 문제는 향후 `LibFertilizer::addFertilizer`가 다른 곳에서 사용되는 경우 고려되어야 합니다. 이는 `FertilizerFacet::addFertilizerOwner`의 경우이지만 소유자가 이러한 유형의 트랜잭션을 보내지 않을 것이라고 가정되므로 문제가 되지 않을 것입니다.


\clearpage
## 낮은 중요도 (Low Risk)

### `MetadataFacet::uri`의 속성에서 메타데이터 특성(traits)의 잘못된 처리 (Incorrect handling of metadata traits in the attributes of `MetadataFacet::uri`)

**설명:** 완전히 온체인인 메타데이터의 경우 외부 클라이언트는 토큰의 URI에 메타데이터와 base64 인코딩된 SVG 이미지가 포함된 base64 인코딩된 JSON 객체가 포함되기를 기대합니다. 이전에 제기된 바와 같이, 이러한 속성이 메타데이터 특성으로 활용되도록 의도된 경우 `MetadataFacet::uri`에서 [attributes 변수](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/metadata/MetadataFacet.sol#L38-L47)의 패킹된 인코딩을 JSON 객체 배열로 올바르게 처리하지 못하면 결과적으로 비표준 JSON 메타데이터가 [반환](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/metadata/MetadataFacet.sol#L48-L55)되므로 외부 클라이언트에서 완전히 활용할 수 없습니다.

**파급력:** OpenSea와 같은 외부 클라이언트는 현재 비표준 JSON 형식으로 인해 Beanstalk 토큰 메타데이터 특성을 표시할 수 없습니다.

**권장 완화 방안:** 인라인 메타데이터 속성을 메타데이터 특성 객체 배열로 리팩토링하여 결과 인코딩된 바이트가 유효한 JSON이 되도록 보장하십시오.

**Beanstalk Farms:** [47fef03](https://github.com/BeanstalkFarms/Beanstalk/pull/655/commits/47fef03a37527c839acd4696db08fbf0bbcd5a71) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.


\clearpage
## 정보성 (Informational)

### `withdrawSeasons` 상태의 재설정이 BIP-36 업그레이드의 일부로 온체인에서 실행되지 않았음 (Resetting of `withdrawSeasons` state was not executed on-chain as part of the BIP-36 upgrade)
`InitBipNewSilo::init`에 `s.season.withdrawSeasons = 0`의 [추가](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/init/InitBipNewSilo.sol#L42-L43)는 BIP-36 업그레이드의 일부로 실행된 [버전](https://etherscan.io/address/0xf6c77e64473b913101f0ec1bfb75a386aba15b9e#code)에는 존재하지 않았던 것으로 보입니다. 따라서 Beanstalk의 상태가 이 변경 사항을 정확하게 반영하도록 하려면 온체인에서 이 로직을 실행하기 위해 또 다른 업그레이드를 수행해야 합니다.

**Beanstalk Farms:** [cca6250](https://github.com/BeanstalkFarms/Beanstalk/pull/655/commits/cca625052179764c930be707a68a43952ec54ddf) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.

### 초기화 컨트랙트에 대한 변경은 온체인에서 실행된 후에는 권장되지 않음 (Changes to initialization contracts are not recommended after they are executed on-chain)
이 기록을 재생하여 Beanstalk 프로토콜을 새로 배포할 경우 Beanstalk의 현재 상태를 반영하도록 하기 위해 BIP 초기화 컨트랙트에 대한 특정 수정이 소급적으로 이루어졌다는 것을 이해합니다. `InitDiamond::init`에 대한 [수정](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/init/InitDiamond.sol#L62) 중 하나인 `stemStartSeason`을 0으로 설정하는 것은 `LibSilo`의 마이그레이션 로직이 우회되는 것처럼 보이므로 무해해 보이지만, [Season diff](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/libraries/Silo/LibLegacyTokenSilo.sol#L469)를 계산할 때 `LibLegacyTokenSilo::_calcGrownStalkForDeposit` 내에서 언더플로우가 발생할 것입니다. 이 문제는 `InitBipNewSilo::init`가 실행되어 `stemStartSeason` 상태를 [실행된 시즌](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/init/InitBipNewSilo.sol#L76-L77)으로 설정할 때까지 존재할 것입니다. 따라서 프로토콜의 메커니즘 개발, 버그 및 관련 업그레이드/완화에 대한 정확한 기록을 유지하기 위해 초기화 스크립트가 온체인에서 실행된 후에는 수정되지 않도록(ossified) 하는 것이 좋습니다.

**Beanstalk Farms:** 이러한 변경의 목적은 Beanstalk의 향후 배포를 미래에 대비하기 위함입니다. 누군가 새로운 Beanstalk을 배포하려는 경우 모든 업그레이드가 이미 구현된 상태에서 프로토콜이 예상대로 계속 작동하는 것이 중요합니다.

새로운 Beanstalk은 `InitDiamond`로만 초기화되고 Beanstalk은 자동으로 최신 버전이 될 것으로 예상됩니다. 다른 `Init` 컨트랙트는 이전 버전에서 다음 버전으로 마이그레이션하기 위한 용도로만 사용됩니다.

`LibLegacyTokenSilo`는 Silo V2에 대한 레거시 지원 및 마이그레이션 기능을 제공하는 데만 사용됩니다. 여기에는 `MigrationFacet`, `LegacyClaimWithdrawalFacet` 및 `SiloExit`의 `seasonToStem(address token, uint32 season)`이 포함됩니다. 새로운 Beanstalk은 Silo V3 업그레이드로 즉시 배포되므로 Silo V2와 하위 호환되거나 V2에서 V3로의 마이그레이션을 어떤 식으로든 지원할 이유가 없습니다.

**Cyfrin:** 인정함.

### BEAN:3CRV MetaPool에서 유동성을 제거하고 BEAN:ETH Well에 유동성을 추가할 때 슬리피지 보호가 부족하여 샌드위치 공격으로 인한 자금 손실이 발생할 수 있음 (Lack of slippage protection when removing liquidity from BEAN:3CRV MetaPool and adding liquidity to BEAN:ETH Well could result in loss of funds due to sandwich attack)
현재 BIP-38 사양에 제공된 *마이그레이션 프로세스*의 두 번째 및 세 번째 단계는 이 BIP 범위에 포함되지 않습니다. 이 단계와 관련된 주요 위험은 수행할 스왑의 규모를 고려할 때 BEAN:3CRV LP 토큰을 BEAN:ETH Well LP 토큰으로 교환하는 것입니다. 이 스왑을 실행하는 트랜잭션의 샌드위치 공격은 자금 손실을 초래할 수 있습니다. 따라서 이를 방지하려면 합리적인 슬리피지 매개변수를 사용하는 것이 필수적입니다. MetaPool에서 유동성을 제거하고 Well에 [유동성을 추가](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/scripts/beanEthMigration.js#L39)할 때 `beanEthMigration.js` 마이그레이션 스크립트 내에서 현재 0 [슬리피지 매개변수](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/scripts/beanEthMigration.js#L26)를 사용하는 것은 테스트 목적으로만 의도된 것으로 이해됩니다. 3CRV -> WETH 스왑 경로는 MEV 보호가 있는 DEX 애그리게이터에서 BCM이 수동으로 실행하거나 OTC 스왑을 통해 실행되며, BCM은 유동성을 제거/추가할 때 슬리피지 매개변수의 적절한 사용을 보장할 것입니다. 이것이 사실인지 확인하는 것이 필수적입니다.

**Beanstalk Farms:** 이 스크립트는 테스트를 돕기 위해 마이그레이션을 모의(mock)하는 데만 사용될 것으로 예상됩니다. 메인넷에서 코드를 실행하는 데 절대 사용되지 않을 것으로 예상되므로 슬리피지 매개변수가 추가되지 않습니다.

**Cyfrin:** 인정함.

### `LibEthUsdOracle::getEthUsdPrice` 설계 변경 사항이 문서화되어야 함 (`LibEthUsdOracle::getEthUsdPrice` design changes should be documented)
BIP-38 이전에는 `LibEthUsdOracle::getEthUsdPrice` 함수가 다음과 같은 동작을 했습니다:
1. Chainlink ETH/USD 오라클과 Uniswap ETH/USDC TWAP 오라클(15분 윈도우 고려) 가격의 차이가 `0.5%` 미만인 경우 두 값의 평균을 반환했습니다. 이제 이 차이는 `0.3%` 미만이어야 합니다.
2. Chainlink ETH/USD 오라클과 Uniswap ETH/USDC TWAP 오라클(15분 윈도우 고려)의 차이가 Chainlink ETH/USD 오라클과 Uniswap ETH/USDT TWAP 오라클(15분 윈도우 고려)의 차이보다 큰 경우:
    * Chainlink ETH/USD 오라클과 Uniswap ETH/USDT TWAP 오라클(15분 윈도우 고려) 가격의 차이가 `2%` 미만이면 이 두 가격의 평균을 반환했습니다. 이제 이 차이는 `1%` 미만이어야 합니다.
    * 그렇지 않으면 오라클이 고장 났거나 오래된 것으로 간주되어 0을 반환했습니다. 이제는 Chainlink ETH/USD 오라클 가격이 정확하다고 가정하고 이를 반환합니다.
3. 그렇지 않은 경우:
    * Chainlink ETH/USD 오라클과 Uniswap ETH/USDC TWAP 오라클(15분 윈도우 고려) 가격의 차이가 `2%` 미만이면 이 두 가격의 평균을 반환했습니다. 이제 이 차이는 `1%` 미만이어야 합니다.
    * 그렇지 않으면 오라클이 고장 났거나 오래된 것으로 간주되어 0을 반환했습니다. 이제는 Chainlink ETH/USD 오라클 가격이 정확하다고 가정하고 이를 반환합니다.

본질적으로 이 함수는 이제 Chainlink ETH/USD 가격이 오래되거나 고장 나지 않는 한(0을 반환하는 경우) 정확하다고 가정합니다. 이 가격과 Uniswap ETH/USDC TWAP 오라클 가격 또는 Uniswap ETH/USDT TWAP 오라클 가격 간의 차이가 특정 임계값을 벗어나는 경우 값 중 하나를 고려하고 평균을 냅니다. 이전에는 이 차이가 특정 범위 내에 있지 않으면 오라클이 고장 난 것으로 간주되었습니다.

**Beanstalk Farms:** 이 변경은 실제로 BIP-37이 배포되기 전에 이루어졌지만, 이 수정 사항은 이전 Cyfrin 감사에서 누락되었습니다. 따라서 BIP-38의 일부로 `getEthUsdPrice`의 기능이 변경되지 않았습니다.

`LibEthUsdOracle`의 주석이 정확하지 않아 [968f783](https://github.com/BeanstalkFarms/Beanstalk/pull/655/commits/968f783d3d062b93f9f692accc9e7ad60d4f1ab6) 커밋에서 업데이트되었습니다.

**Cyfrin:** 인정함. 주석이 이제 코드의 의도와 일치합니다.

### (더 이상 사용되지 않는) Fertilizer의 FIFO 목록 중간 추가와 관련된 `LibFertilizer::push`의 로직 제거 (`LibFertilizer::push` logic related to (deprecated) intermediate addition of Fertilizer to FIFO list should be removed)
FIFO 목록에 대한 중간 추가는 이전에는 Beanstalk DAO에서만 허용되었지만 `FertilizerFacet::addFertilizerOwner`가 제거됨에 따라 현재 업그레이드에서는 이 기능이 더 이상 사용되지 않습니다. 결과적으로 `LibFertilizer::push`의 [해당 로직](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/libraries/LibFertilizer.sol#L139-L147)은 이제 도달할 수 없는 코드이므로 제거해야 합니다.

**Beanstalk Farms:** `push(...)` 함수는 여전히 [여기](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/libraries/LibFertilizer.sol#L60)에서 내부적으로 사용됩니다.

강조 표시된 세그먼트는 더 이상 도달할 수 없지만 어떤 이유로 습도가 변경되는 경우 다시 도달할 수 있습니다. 이러한 이유로 그대로 두기로 결정했습니다.

**Cyfrin:** 인정함.

### `MetadataFacet::uri` 면책 조항을 메타데이터 속성에서 설명으로 이동하는 것을 고려 (Consider moving the `MetadataFacet::uri` disclaimer from metadata attributes to the description)
`MetadataFacet::uri` 내의 [면책 조항](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/metadata/MetadataFacet.sol#L46)은 현재 JSON 속성의 끝에 있지만, 대신 메타데이터 설명 내에 배치하는 것이 더 나을 수 있습니다.

**Beanstalk Farms:** 면책 조항 배치는 주로 [Uniswap V3의 NFT](https://opensea.io/assets/ethereum/0xc36442b4a4522e871399cd717abdd847ab11fe88/528320)에서 영감을 얻었으므로 속성 섹션이 유지하기에 적절한 곳이라고 생각합니다.

**Cyfrin:** 인정함.

### `MetadataImage::sciNotation`의 잘못된 주석 수정 (Incorrect comment in `MetadataImage::sciNotation` should be corrected)
`MetadataImage::sciNotation`은 입력 Stem을 문자열 표현으로 변환하기 위한 것으로, 값이 [1e5보다 큰](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/metadata/MetadataImage.sol#L539) 경우 과학적 표기법을 사용합니다. 임계값으로 [1e7을 참조](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/metadata/MetadataImage.sol#L538)하는 관련 주석이 잘못되었으므로 1e5로 수정해야 합니다.

**Beanstalk Farms:** [81e452e](https://github.com/BeanstalkFarms/Beanstalk/commit/81e452e41c2533dfc49543dc70fba15ed3c6cc2f) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.

### `InitBipBasinIntegration::init`에서 "Seeds"에 대한 지속적인 참조는 혼란스러움 (Continued reference to "Seeds" in `InitBipBasinIntegration::init` is confusing)
"Seeds" 용어가 더 이상 사용되지 않으므로 [지속적인 참조](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/init/InitBipBasinIntegration.sol#L31-L33)는 혼란스럽고 모든 인스턴스는 대신 시즌당 BDV당 획득한 Stalk를 참조하도록 업데이트되어야 합니다.

**Beanstalk Farms:** [ba1d42b](https://github.com/BeanstalkFarms/Beanstalk/pull/655/commits/ba1d42bc9159881143c5f23ab03a7ba8078bd4b0) 커밋에서 이름이 업데이트되었습니다.

### `InitBipBasinIntegration` NatSpec 제목 태그가 파일/컨트랙트 이름과 일치하지 않음 (`InitBipBasinIntegration` NatSpec title tag is inconsistent with the file/contract name)
`InitBipBasinIntegration` NatSpec의 [제목 태그](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/init/InitBipBasinIntegration.sol#L17)가 파일/컨트랙트 이름과 일치하지 않으므로 일치하도록 업데이트해야 합니다.

**Beanstalk Farms:** [c03f635](https://github.com/BeanstalkFarms/Beanstalk/pull/655/commits/c03f635ef655eb80a2f6a270c41f19bcbd4a66ad) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.

### `WellPrice::getConstantProductWell`의 조건부 블록 제거 가능 (Conditional block in `WellPrice::getConstantProductWell` can be removed)
`WellPrice::getConstantProductWell`의 [else 블록](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/ecosystem/price/WellPrice.sol#L64-L67)은 Bean 가격을 결정할 수 없는 경우를 처리하는데, `pool.price`의 기본값이 이미 0이므로 필요하지 않으며 제거할 수 있습니다.

**Beanstalk Farms:** [8aae31d](https://github.com/BeanstalkFarms/Beanstalk/pull/655/commits/8aae31d683aeec50ccbc17985701b46223cc0a1d) 커밋에서 제거되었습니다.

**Cyfrin:** 인정함.

### `WellPrice::getDeltaB`의 안전하지 않은 캐스트 (Unsafe cast in `WellPrice::getDeltaB`)
오버플로우 가능성은 낮지만 `WellPrice::getDeltaB`에 [안전하지 않은 캐스트](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/ecosystem/price/WellPrice.sol#L97)가 있으며 이를 안전한 캐스트로 대체할 수 있습니다.

**Beanstalk Farms:** [ff742a6](https://github.com/BeanstalkFarms/Beanstalk/pull/655/commits/ff742a6f5b0b166df988a2422e475d314b948fc9) 커밋에서 수정되었습니다.

### `FertilizerFacet::getMintFertilizerOut` NatSpec의 오타 (Typo in `FertilizerFacet::getMintFertilizerOut` NatSpec)
[`FertilizerFacet::getMintFertilizerOut`](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/beanstalk/barn/FertilizerFacet.sol#L108)의 NatSpec은 현재 Fertilizer를 `Fertilize`로 참조하고 있으므로 수정해야 합니다.

**Beanstalk Farms:** [373c094](https://github.com/BeanstalkFarms/Beanstalk/pull/655/commits/373c0948cce9730446111a943a4fd96dabd90025) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.

### `LibSilo::_mow` 내 주석의 오타 (Typo in comment within `LibSilo::_mow`)
`LibSilo::_mow`의 다음 [오타](https://github.com/BeanstalkFarms/Beanstalk/blob/12c608a22535e3a1fe379db1153185fe43851ea7/protocol/contracts/libraries/Silo/LibSilo.sol#L351-L352)를 수정해야 합니다:

```diff
- //sop stuff only needs to be updated once per season
- //if it started raininga nd it's still raining, or there was a sop
+ // sop stuff only needs to be updated once per season
+ // if it started raining and it's still raining, or there was a sop
```

**Beanstalk Farms:** [d27567c](https://github.com/BeanstalkFarms/Beanstalk/pull/655/commits/d27567c5f84bf07d604397f4d4549570ac9fb8c4) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.

