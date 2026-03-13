**수석 감사 (Lead Auditors)**

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

[Carlos Amarante](https://twitter.com/carlitox477)

**보조 감사 (Assisting Auditors)**

[Dacian](https://twitter.com/DevDacian)


---

# 발견 사항 (Findings)
## 높은 위험 (High Risk)


### 수정된 Facet 및 종속성이 수정된 Facet을 `bips::bipSeedGauge`에 추가하지 않으면 프로토콜이 손상됨

**설명:** Diamond Proxy 업그레이드 시, 수정된 Facet은 [`bips.js`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/scripts/bips.js) 내의 관련 함수에 포함되어야 합니다. 현재 [`bipSeedGauge` 함수](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/scripts/bips.js#L198-L240)는 이전 업그레이드 이후 수정된 `FieldFacet`, `BDVFacet`, `ConvertFacet`, 및 `WhitelistFacet`이 누락된 것으로 보입니다. 또한, 라이브러리가 수정된 Facet의 추가가 고려되지 않아 프로토콜을 손상시키는 여러 문제가 발생합니다.

**영향:** 언뜻 보기에 이러한 Facet이나 사용된 라이브러리가 Beanstalk의 핵심 비즈니스 로직에 대한 중요한 수정을 포함하지 않는 것처럼 보이므로 영향이 낮다고 생각할 수 있습니다. 그러나 여러 Facet에서 사용하는 다른 라이브러리에 상당한 변경이 있었기 때문에 실제로는 그렇지 않습니다. 더 심각한 문제 중 하나는 의도한 것보다 훨씬 많은 양의 Stalk가 발행되어 프로토콜 회계가 깨지는 것입니다.

**개념 증명 (Proof of Concept):** 다음 명령을 실행하여 모든 수정된 Facet 목록을 얻을 수 있습니다:
```bash
git diff --stat 7606673..dfb418d -- ".sol" ":\!protocol/test/" ":\!protocol/contracts/mocks/*" | grep "Facet.sol"
```
출력:
```
protocol/contracts/beanstalk/barn/UnripeFacet.sol  | 360 +++++++++------
protocol/contracts/beanstalk/field/FieldFacet.sol  |   4 +-
protocol/contracts/beanstalk/silo/BDVFacet.sol     |  12 +-
protocol/contracts/beanstalk/silo/ConvertFacet.sol |   8 +-
.../contracts/beanstalk/silo/WhitelistFacet.sol    |  79 +++-
.../contracts/beanstalk/sun/GaugePointFacet.sol    |  39 ++
.../beanstalk/sun/SeasonFacet/SeasonFacet.sol      | 120 +++--
.../sun/SeasonFacet/SeasonGettersFacet.sol         | 248 ++++++++++
```

다음 명령을 실행하여 모든 수정된 라이브러리 목록을 얻을 수 있습니다:
```bash
git diff --stat 7606673..dfb418d -- "*.sol" ":\!protocol/test/*" ":\!protocol/contracts/mocks/*" | grep "Lib.*\.sol"
```
출력:
```
.../contracts/libraries/Convert/LibChopConvert.sol |  60 +++
.../contracts/libraries/Convert/LibConvert.sol     |  42 +-
.../contracts/libraries/Convert/LibConvertData.sol |   3 +-
.../libraries/Convert/LibUnripeConvert.sol         |  18 +-
.../contracts/libraries/Convert/LibWellConvert.sol |   3 +-
.../contracts/libraries/Curve/LibBeanMetaCurve.sol |  15 +
.../contracts/libraries/Curve/LibMetaCurve.sol     |  61 ++-
protocol/contracts/libraries/LibCases.sol          | 161 +++++++
protocol/contracts/libraries/LibChop.sol           |  65 +++
protocol/contracts/libraries/LibEvaluate.sol       | 297 ++++++++++++
protocol/contracts/libraries/LibFertilizer.sol     |   2 +
protocol/contracts/libraries/LibGauge.sol          | 330 +++++++++++++
protocol/contracts/libraries/LibIncentive.sol      |  20 +-
.../contracts/libraries/LibLockedUnderlying.sol    | 509 +++++++++++++++++++++
protocol/contracts/libraries/LibUnripe.sol         | 180 ++++++--
.../libraries/Minting/LibCurveMinting.sol          |  26 +-
.../contracts/libraries/Minting/LibWellMinting.sol |  30 +-
.../libraries/Oracle/LibBeanEthWellOracle.sol      |  55 ---
.../contracts/libraries/Oracle/LibEthUsdOracle.sol |   3 +-
.../contracts/libraries/Oracle/LibUsdOracle.sol    |  18 +-
.../libraries/Silo/LibLegacyTokenSilo.sol          |   4 -
protocol/contracts/libraries/Silo/LibTokenSilo.sol |  18 +-
protocol/contracts/libraries/Silo/LibWhitelist.sol | 181 ++++++--
.../libraries/Silo/LibWhitelistedTokens.sol        |  47 ++
protocol/contracts/libraries/Well/LibWell.sol      | 217 ++++++++-
```

깨진 Stalk 회계를 입증하기 위해 다음과 같은 코드화된 개념 증명이 작성되었습니다:
```javascript
const { expect } = require('chai');
const { takeSnapshot, revertToSnapshot } = require("../utils/snapshot.js");
const { BEAN, BEAN_3_CURVE, UNRIPE_BEAN, UNRIPE_LP, WETH, BEAN_ETH_WELL, PUBLIUS, ETH_USD_CHAINLINK_AGGREGATOR } = require('../utils/constants.js');
const { bipSeedGauge } = require('../../scripts/bips.js');
const { getBeanstalk } = require('../../utils/contracts.js');
const { impersonateBeanstalkOwner, impersonateSigner } = require('../../utils/signer.js');
const { ethers } = require('hardhat');

const { impersonateBean, impersonateEthUsdChainlinkAggregator} = require('../../scripts/impersonate.js');
let silo, siloExit, bean

let grownStalkBeforeUpgrade, grownStalkAfterUpgrade
let snapshotId
let whitelistedTokenSnapshotBeforeUpgrade, whitelistedTokenSnapshotAfterUpgrade

const whitelistedTokens = [BEAN, BEAN_3_CURVE, UNRIPE_BEAN, UNRIPE_LP, BEAN_ETH_WELL]

const whitelistedTokensNames = ["BEAN", "BEAN:3CRV CURVE LP", "urBEAN", "urBEAN:WETH", "BEAN:WETH WELLS LP"]
const beanHolderAddress = "0xA9Ce5196181c0e1Eb196029FF27d61A45a0C0B2c"
let beanHolder

/**
 * Async function
 * @returns tokenDataSnapshot: Mapping from token address to (name,stemTip)
 */
const getTokenDataSnapshot = async()=>{
  tokenDataSnapshot = new Map()

  for(token of whitelistedTokens){
    tokenDataSnapshot.set(token,{
      name: whitelistedTokensNames[whitelistedTokens.indexOf(token)],
      stemTip: await silo.stemTipForToken(token)
    })
  }
  return tokenDataSnapshot
}



const forkMainnet = async()=>{
  try {
    await network.provider.request({
      method: "hardhat_reset",
      params: [
        {
          forking: {
            jsonRpcUrl: process.env.FORKING_RPC,
            blockNumber: 18619555-1 //a random semi-recent block close to Grown Stalk Per Bdv pre-deployment
          },
        },
      ],
    });
  } catch(error) {
    console.log('forking error in seed Gauge');
    console.log(error);
    return
  }
}

const initializateContractsPointers = async(beanstalkAddress)=>{
  tokenSilo = await ethers.getContractAt('TokenSilo', beanstalkAddress);
  seasonFacet = await ethers.getContractAt('ISeasonFacet', beanstalkAddress);
  siloFacet = await ethers.getContractAt('SiloFacet', beanstalkAddress);
  silo = await ethers.getContractAt('ISilo', beanstalkAddress);
  siloExit = await ethers.getContractAt('SiloExit', beanstalkAddress);
  admin = await ethers.getContractAt('MockAdminFacet', beanstalkAddress);
  well = await ethers.getContractAt('IWell', BEAN_ETH_WELL);
  weth = await ethers.getContractAt('IWETH', WETH)
  bean = await ethers.getContractAt('IBean', BEAN)
  beanEth = await ethers.getContractAt('IWell', BEAN_ETH_WELL)
  beanEthToken = await ethers.getContractAt('IERC20', BEAN_ETH_WELL)
  unripeLp = await ethers.getContractAt('IERC20', UNRIPE_LP)
  beanMetapool = await ethers.getContractAt('MockMeta3Curve', BEAN_3_CURVE)
  chainlink = await ethers.getContractAt('MockChainlinkAggregator', ETH_USD_CHAINLINK_AGGREGATOR)
}

const impersonateOnchainSmartContracts = async() => {
  publius = await impersonateSigner(PUBLIUS, true)
  await impersonateEthUsdChainlinkAggregator()
  await impersonateBean()
  owner = await impersonateBeanstalkOwner()
}

const deposit = async(signer, tokenDataSnapshot) => {
  await network.provider.send("hardhat_setBalance", [
    signer.address,
    "0x"+ethers.utils.parseUnits("1000",18).toString()
  ]);


  beanToDeposit = await bean.balanceOf(signer.address)
  // console.log(`Beans to deposit: ${ethers.utils.formatUnits(beanToDeposit,6)}`)
  beanStemTip = tokenDataSnapshot.get(BEAN).stemTip
  await siloFacet.connect(signer).deposit(BEAN,beanToDeposit,0) // 0 = From.EXTERNAL
  return await tokenSilo.getDepositId(BEAN, beanStemTip, )
}

describe('Facet upgrade POC', function () {
  before(async function () {

    // Get users to impersonate
    [user, user2] = await ethers.getSigners()

    // fork mainnet
    await forkMainnet()

    // Replace on chain smart contract for testing
    await impersonateOnchainSmartContracts()
    beanHolder = await impersonateSigner(beanHolderAddress)

    this.beanstalk = await getBeanstalk()

    await initializateContractsPointers(this.beanstalk.address)

    // Before doing anything we record some state variables that should hold
    whitelistedTokenSnapshotBeforeUpgrade = await getTokenDataSnapshot()

    // We do a deposit
    depositId = await deposit(beanHolder,whitelistedTokenSnapshotBeforeUpgrade)

    grownStalkBeforeUpgrade = await siloExit.balanceOfGrownStalk(beanHolder.address,BEAN)

    // seed Gauge
    await bipSeedGauge(true, undefined, false)
    console.log("BIP-39 initiated\n")

    whitelistedTokenSnapshotAfterUpgrade = await getTokenDataSnapshot(silo)
    grownStalkAfterUpgrade = await siloExit.balanceOfGrownStalk(beanHolder.address,BEAN)
  });

  beforeEach(async function () {
    snapshotId = await takeSnapshot()
  });

  afterEach(async function () {
    await revertToSnapshot(snapshotId)
  });


  describe('init state POC', async function () {

    it("Grown stalk backward compatibility",async()=>{
      expect(grownStalkBeforeUpgrade).to.be.eq(grownStalkAfterUpgrade, "Grown stalk for a deposit after BIP-39 upgrade is not the same than before the upgrade")
    })


    it('Stem tip backward compatibility',async()=>{
      expect(whitelistedTokenSnapshotBeforeUpgrade.get(BEAN).stemTip).to.be.equal(whitelistedTokenSnapshotAfterUpgrade.get(BEAN).stemTip,"BEAN stem tip is not the same than after the upgrade")
      expect(whitelistedTokenSnapshotBeforeUpgrade.get(BEAN_3_CURVE).stemTip).to.be.equal(whitelistedTokenSnapshotAfterUpgrade.get(BEAN_3_CURVE).stemTip,"BEAN:3CRV Curve LP stem tip is not the same than after the upgrade")
      expect(whitelistedTokenSnapshotBeforeUpgrade.get(BEAN_ETH_WELL).stemTip).to.be.equal(whitelistedTokenSnapshotAfterUpgrade.get(BEAN_ETH_WELL).stemTip,"BEAN:3CRV Curve LP stem tip is not the same than after the upgrade")
      expect(whitelistedTokenSnapshotBeforeUpgrade.get(UNRIPE_BEAN).stemTip).to.be.equal(whitelistedTokenSnapshotAfterUpgrade.get(UNRIPE_BEAN).stemTip,"BEAN:3CRV Curve LP stem tip is not the same than after the upgrade")
      expect(whitelistedTokenSnapshotBeforeUpgrade.get(UNRIPE_LP).stemTip).to.be.equal(whitelistedTokenSnapshotAfterUpgrade.get(UNRIPE_LP).stemTip,"BEAN:3CRV Curve LP stem tip is not the same than after the upgrade")

    })
  })
})
```
되돌아가는 기대값(reverting expectations)에서 알 수 있듯이 업그레이드에 `SiloFacet`을 추가하지 않으면 마일스톤 stem 업데이트와 grown stalk 회계가 깨집니다.

새로운 게이지 포인트 시스템(앞으로는 잘리지 않은 값을 사용)과 함께 사용하기 위해 이전 마일스톤 stem을 조정하는 것과 관련된 문제에 대한 해결책이 `SiloFacet` 업데이트 없이 구현되면 이전 `LibTokenSilo::stemTipForToken` 구현이 사용됩니다. 이로 인해 업그레이드 전에 수행된 예금은 의도한 것보다 훨씬 더 많은 grown stalk를 받게 됩니다.

**권장 완화 방법:** 항상 수정된 모든 Facet과 라이브러리 종속성이 수정된 모든 Facet을 업그레이드 스크립트에 추가해야 합니다. 향후 업그레이드 프로세스에서 유사한 오류를 잡기 위해 업그레이드 시뮬레이션 테스트 스위트를 개발하는 것이 좋습니다.


### 앞으로는 잘리지 않은 값을 사용하는 새로운 게이지 포인트 시스템과 함께 사용하기 위해 이전 마일스톤 stem을 조정해야 함

**설명:** Beanstalk Silo 내에서 특정 토큰의 마일스톤 stem은 마지막 `stalkEarnedPerSeason` 업데이트 시 이 토큰에 대한 BDV당 누적 grown stalk 양입니다. 이전에는 마일스톤 stem이 잘린(truncated) 표현으로 저장되었지만, 시드 게이지 시스템은 이제 grown stalk의 새로운 세분성과 이러한 값이 업데이트되는 빈도로 인해 잘리지 않은 형식으로 값을 저장합니다.

업그레이드 시, 각 토큰에 대한 이전(잘린) 마일스톤 stem은 `1e6`만큼 곱하여 게이지 포인트 시스템과 함께 사용하도록 조정해야 합니다. 그렇지 않으면 [stem tip을 계산할 때](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibTokenSilo.sol#L388-L391) 소수점 불일치가 발생합니다.

```solidity
_stemTipForToken = s.ss[token].milestoneStem +
    int96(s.ss[token].stalkEarnedPerSeason).mul(
        int96(s.season.current).sub(int96(s.ss[token].milestoneSeason))
    );
```

**영향:** 이전 마일스톤 stem(잘림)과 새 마일스톤 stem(잘리지 않음, BIP-39 업그레이드 후 첫 번째 `gm` 호출 후) 간의 소수점 혼합은 기존 grown stalk 회계를 깨뜨려 예금자의 grown stalk 손실을 초래합니다.

**개념 증명 (Proof of Concept):** [이전 구현](https://github.com/BeanstalkFarms/Beanstalk/blob/7606673/protocol/contracts/libraries/Silo/LibTokenSilo.sol#L376-L391)은 소수점 4자리의 BDV당 누적 stalk를 반환합니다:
```solidity
    function stemTipForToken(address token)
        internal
        view
        returns (int96 _stemTipForToken)
    {
        AppStorage storage s = LibAppStorage.diamondStorage();

        // SafeCast unnecessary because all casted variables are types smaller that int96.
        _stemTipForToken = s.ss[token].milestoneStem +
        int96(s.ss[token].stalkEarnedPerSeason).mul(
            int96(s.season.current).sub(int96(s.ss[token].milestoneSeason))
        ).div(1e6); //round here
    }
```

이것은 수학적으로 다음과 같이 추상화될 수 있습니다:
$$StemTip(token) = getMilestonStem(token) + (current \ season - getMilestonStemSeason(token)) \times \frac{stalkEarnedPerSeason(token)}{10^{6}}$$

$10^{6}$으로 나누는 것은 이전에 stem tip이 소수점 4자리만 가졌기 때문에 발생합니다. 이 나눗셈은 마지막 6자리를 고려하지 않음으로써 이전 버전과의 호환성을 허용합니다. 따라서 stem tip은 **항상** 소수점 4자리를 가져야 합니다.

마일스톤 stem은 이제 모든 [LP 가격 오라클이 각각의 검사를 통과](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibGauge.sol#L65-L67)하는 한 [각 `gm` 호출에서 업데이트됩니다](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibGauge.sol#L265-L268). 특히 마일스톤 stem은 이제 10자리(잘리지 않음)로 저장되므로 추상화의 두 번째 항은 `LibTokenSilo::stemTipForTokenUntruncated`에서 `10^{6}` 나눗셈을 생략했습니다.

그러나 기존 마일스톤 stem이 $10^{6}$만큼 조정되지 않으면 업그레이드 중 및 후속 `gm` 호출에서 수행되는 덧셈은 의미가 없습니다. 이것은 업그레이드 내에서 반드시 처리되어야 하며 그렇지 않으면 `LibTokenSilo.stemTipForToken`을 호출하는 프로토콜의 모든 부분은 BEAN:ETH Well LP(Silo v3 업그레이드 후 생성되었으므로)를 제외하고 잘못된 값을 받게 됩니다.

이 함수가 사용되는 몇 가지 인스턴스는 다음과 같습니다:
* [`EnrootFacet::enrootDeposit`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/silo/EnrootFacet.sol#L91)
* [`EnrootFacet::enrootDeposits`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/silo/EnrootFacet.sol#L131)
* [`MetaFacet::uri`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/metadata/MetadataFacet.sol#L36)
* [`ConvertFacet::_withdrawTokens`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/silo/ConvertFacet.sol#L129-148)
* [`LibSilo::__mow`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibSilo.sol#L382)
* [`LibSilo::_removeDepositFromAccount`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibSilo.sol#L545)
* [`LibSilo::_removeDepositsFromAccount`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibSilo.sol#L604)
* [`Silo::_plant`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/silo/SiloFacet/Silo.sol#L110)
* [`TokenSilo::_deposit`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/silo/SiloFacet/TokenSilo.sol#L173)
* [`TokenSilo::_transferDeposits`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/silo/SiloFacet/TokenSilo.sol#L367)
* [`LibLegacyTokenSilo::_mowAndMigrate`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibLegacyTokenSilo.sol#L306)
* [`LibTokenSilo::_mowAndMigrate`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibLegacyTokenSilo.sol#L306)

관찰할 수 있듯이 프로토콜의 중요한 부분이 손상되어 추가적인 연쇄 문제가 발생합니다.

**권장 완화 방법:** 각 토큰에 대한 기존 마일스톤 stem을 확장(scale up)하십시오:
```diff
for (uint i = 0; i < siloTokens.length; i++) {
+   s.ss[siloTokens[i]].milestoneStem = int96(s.ss[siloTokens[i]].milestoneStem.mul(1e6));
```

\clearpage
## 중간 위험 (Medium Risk)


### `LibLockedUnderlying::getPercentLockedUnderlying`의 잘못된 소수점 처리로 인해 잘못된 값이 반환되어, `unripe asset supply < 10M`일 때 `SeasonFacet::gm`에 대한 각 후속 호출에서 온도 및 Bean 대 maxLP gaugePoint per BDV 비율 업데이트에 영향을 미침

**설명:** Barn Raise 및 관련 Unripe 자산의 기본 Bean으로 인해 거래 가능한 Bean의 수는 총 Bean 공급량과 동일하지 않습니다. L2SR 계산 내에서 "잠긴 유동성(locked liquidity)"이라는 용어는 해당 Fertilizer가 지불될 때까지 Chopping을 통해 회수할 수 없는 BEAN:ETH WELL의 유동성 부분을 나타냅니다.

해당 기본 자산에 대한 교환 비율은 다음 공식으로 요약할 수 있습니다:

$$\frac{Paid Fertilizer}{Minted Fertilizer} \times \frac{totalUnderlying(urAsset)}{supply(urAsset)}$$

두 번째 요소는 각 unripe 자산을 뒷받침하는 기본 자산의 양을 나타내고 첫 번째 요소는 이미 지불된 Fertilizer 비율을 기반으로 기본 자산의 분포를 나타냅니다.

사용자가 unripe 자산을 Chop하면 페널티가 적용된 양의 기본 자산과 교환되어 소각됩니다. 남은 기본 자산은 이제 남은 unripe 자산 보유자들 사이에서 공유됩니다. 즉, 다른 사용자가 주어진 자본 재구성 비율로 동일한 양의 unripe 자산을 Chop하려고 하면 더 많은 양의 기본 자산을 받게 됩니다.

예를 들어 다음을 가정합니다:
* 발행된 Fertilizer의 50%가 지불됨
* 현재 공급량 70M
* 기본 금액 22M

Alice가 1M unripe 토큰을 Chop하면:
$$1,000,000 \times 0.50 \times \frac{22,000,000}{70,000,000} =$$
$$1,000,000 \times 0.50 \times 0.31428 =$$
$$1,000,000 \times 0.50 \times 0.31428 =$$
$$1,000,000 \times 0.15714285 = $$
$$157,142.85$$

Bob이 동일한 양의 토큰을 Chop하면:
$$1,000,000 \times 0.50 \times \frac{22,000,000-157,142.85}{70,000,000 - 1,000,000} =$$
$$1,000,000 \times 0.50 \times \frac{21,842,857.15}{69,000,000} =$$
$$1,000,000 \times 0.50 \times \frac{21,842,857.15}{69,000,000} =$$
$$1,000,000 \times 0.50 \times 0.3165 =$$
$$158,281.57$$

총 unripe 자산 공급을 한 단계로 Chop한다는 가정이 매우 낮기 때문에 Beanstalk Farms 팀은 unripe 자산 보유자당 평균 unripe 자산을 기반으로 오프체인 회귀를 수행하기로 결정했습니다. 이는 현재 unripe 자산 공급을 기반으로 자산당 잠긴 기본 토큰 비율에 대한 근사치를 산출합니다. 온체인 조회 테이블을 사용하여 이 회귀 값을 검색합니다. 그러나 구현상의 문제는 반복 시뮬레이션이 수행된 간격인 인라인 조건부 공급 상수 [`1_000_000`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibLockedUnderlying.sol#L61), [`5_000_000`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibLockedUnderlying.sol#L62), 및 [`10_000_000`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibLockedUnderlying.sol#L63)과 비교할 때 unripe 토큰 소수점을 고려하지 않는다는 것입니다. 이러한 상수는 나타내려는 숫자의 고정 소수점 표현이 아니므로 [소수점 6자리 공급](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibLockedUnderlying.sol#L60)과의 [비교](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibLockedUnderlying.sol#L286)가 올바르지 않습니다.

**영향:** unripe 자산은 소수점 6자리를 가지므로 `LibLockedUnderlying::getPercentLockedUnderlying`은 [이 조건부 분기](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibLockedUnderlying.sol#L63-L173)를 실행하는 경향이 있어 unripe 자산의 공급이 10M 미만일 때마다 잠긴 기본 자산의 계산이 올바르지 않게 됩니다.

주어진 시나리오에서 이 오류는 L2SR의 잘못된 계산으로 이어져 [`SeasonFacet::gm`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/sun/SeasonFacet/SeasonFacet.sol#L51) 내의 [`Weather::calcCaseIdandUpdate`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/sun/SeasonFacet/Weather.sol#L68) 호출에서 온도 및 Bean 대 maxLP gaugePoint per BDV 비율이 업데이트되는 방식에 영향을 미칩니다.

**개념 증명 (Proof of Concept):** Beanstalk Farms 팀이 제공한 CSV를 기반으로 이 문제를 입증하기 위해 차등 테스트(부록 A 참조)가 작성되었습니다. CSV 수정 사항은 다음과 같습니다:
* 헤더 추가: recapPercentage, urSupply, lockedPercentage
* 공백 없이 CSV 생성
* 첫 번째 열을 소수점 3자리로 반올림
* 세 번째 열의 경우 `e18`을 삭제하고 값을 소수점 18자리로 반올림

**권장 완화 방법:** unripe 공급과 비교되는 각 인라인 상수를 소수점 6자리만큼 조정하십시오.

향후 유사한 경우에 대해 예상 출력과 실제 출력 간의 차등 테스트는 미리 계산된 오프체인 값에 의존하는 이러한 유형의 버그를 잡는 데 효과적입니다.


### 게이지 포인트 업데이트는 Sunrise 시점의 순간적인 값이 아니라 시간 가중 평균 예치 LP BDV를 고려하여 이루어져야 함

**설명:** 시드 게이지 시스템이 도입되기 전에는 화이트리스트 자산에 대한 BDV당 Grown Stalk가 정적이었으며 거버넌스를 통해서만 변경할 수 있었습니다. 시드 게이지 시스템은 이제 Beanstalk가 시즌당 발행해야 하는 BDV당 Grown Stalk 양을 목표로 할 수 있도록 하며, 해당 시즌에 발행된 Grown Stalk가 화이트리스트 LP 토큰 간에 어떻게 분배되어야 하는지 결정하기 위해 게이지 포인트가 도입되었습니다.

게이지 포인트는 매 시즌마다 `SeasonFacet::gm` 내에서 `LibGauge::stepGauge`가 호출될 때 업데이트됩니다. 이 게이지 포인트 업데이트는 현재 `gm` 호출 시 [순간적인 총 예치 LP BDV를 고려](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibGauge.sol#L115-L122)하여 수행됩니다. 그러나 이 값은 조작될 수 있으므로 시드 게이지 시스템은 대신 이전 시즌 기간 동안의 시간 가중 평균 예치 LP BDV를 사용해야 합니다.

**영향:** 주어진 화이트리스트 LP에 대한 게이지 포인트는 시즌당 1포인트씩만 증가/감소할 수 있고 Bean 대 max LP GP per BDV 비율은 100%로 제한되므로 이 공격을 수행할 유인은 비교적 낮습니다. 그러나 Sunrise 호출 직전에 대규모 예금을 하고 직후에 인출하면 조작이 발생하여 시드 게이지 시스템이 의도한 대로 작동하지 않을 수 있습니다.

**권장 완화 방법:** 순간적인 값을 사용하는 대신 이전 시즌 기간 동안의 시간 가중 평균 예치 LP BDV를 계산하는 것을 고려하십시오. 블록 내 조작을 피하기 위해 각 블록 계산에 포함할 BDV는 이전 블록 말의 BDV여야 합니다. 이러한 값은 저장되어야 하며 어떤 방식으로든 총 예치 BDV를 수정하는 함수가 호출될 때마다 업데이트가 트리거되어야 합니다.


### `InitBipSeedGauge`의 게이지 포인트 상수는 예치된 BDV의 비율로 조정되어야 함

**설명:** [현재 초기 게이지 포인트(GP) 분배](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/init/InitBipSeedGauge.sol#L68-L69)는 각 LP에 대한 LP당 예치된 BDV를 고려하여 결정되어야 하는 반면, 각 LP에 대한 시즌당 BDV당 grown stalk에만 전적으로 기반합니다.

게이지 시스템의 동작을 뒷받침하는 다음 수학을 고려하면:
$$depositedBDVRatio(LP) = \frac{silo.totalDepositedBDV(LP)}{\sum_{wlpt}^{wlpt \in Whitelisted \ LP \ Tokens} silo.totalDepositedBDV} $$
$GP_{s}(LP) =$
1. $depositedBDVRatio(LP) > LP.optimalDepositedBDVRatio \land GP_{s-1}(LP) \leq 1gp \Rightarrow GP_{s}(LP) = 0$
2. $depositedBDVRatio(LP) > LP.optimalDepositedBDVRatio \land GP_{s-1}(LP) > 1gp \Rightarrow GP_{s}(LP) = GP_{s-1}(LP)- 1gp$
3. $depositedBDVRatio(LP) \leq LP.optimalDepositedBDVRatio \Rightarrow GP_{s}(LP) = GP_{s-1}(LP) + 1gp$

공식이 이전 $GP_{s-1}(LP)$에 의존함을 알 수 있습니다. 여기서 $s$는 현재 시즌 번호와 예치된 BDV 비율을 나타냅니다. 또한 이 메커니즘의 의도는 Beanstalk 프로토콜이 각 LP에 대해 사전 정의된 최적의 예치된 BDV 비율을 갖도록 장려하는 것임이 분명합니다. 결과적으로 GP의 초기 할당은 이러한 의도를 고려해야 합니다.

**영향:** 잘못된 초기 GP 분배는 의도하지 않은 초기 동작을 초래할 수 있으며, [`GaugePointFacet::defaultGaugePointFunction`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/sun/GaugePointFacet.sol#L26-L38)에 정의된 대로 게이지 포인트는 시즌당 1포인트씩만 증가/감소할 수 있으므로 이를 수정하는 데 상당한 시간이 걸릴 수 있습니다.

**개념 증명 (Proof of Concept):**
```solidity
// InitBipSeedGauge.sol
uint128 beanEthGp = uint128(s.ss[C.BEAN_ETH_WELL].stalkEarnedPerSeason) * 500 * 1e12;
uint128 bean3crvGp = uint128(s.ss[C.CURVE_BEAN_METAPOOL].stalkEarnedPerSeason) * 500 * 1e12
```

관찰된 바와 같이 초기 GP 할당은 BIP-39 이전의 시즌당 획득한 stalk에 의해 결정되며 값은 다음과 같습니다:
* BEAN:3CRV Curve LP: `3.25e6`
* BEAN:ETH Well LP: `4.5e6`

이 값들은 LP당 예치된 총 BDV와 상관관계가 없습니다. 결과적으로 GP의 초기 할당은 잘못된 값으로 이루어집니다.

**권장 완화 방법:** [1 게이지 포인트가 1e18과 같다](https://github.com/BeanstalkFarms/Beanstalk/blob/08ca0d7d495c94f2a4366fb7f99da561b74cc1c0/protocol/contracts/beanstalk/sun/GaugePointFacet.sol#L19)는 점을 고려하여 다음과 같이 수정해야 합니다:

```diff
//  InitBipSeedGauge.sol
+   // BDV has 6 decimals
+   uint256 beanEthBDV = s.siloBalances[C.BEAN_ETH_WELL].depositedBdv
+   uint256 bean3crvBDV = s.siloBalances[C.CURVE_BEAN_METAPOOL].depositedBdv
+   uint256 lpTotalBDV = beanEthBDV + bean3crvGp
-   uint128 beanEthGp = uint128(s.ss[C.BEAN_ETH_WELL].stalkEarnedPerSeason) * 500 * 1e12;
-   uint128 bean3crvGp = uint128(beanEthBDV) * 500 * 1e12
+   // Assume 1 BDV = 1GP for initialization
+   uint128 beanEthGp = uint128(beanEthBDV * 10e6).div(lpTotalBDV) * 1e12;
+   uint128 bean3crvGp = uint128(bean3crvBDV * 10e6).div(lpTotalBDV) * 1e12
```


### `InitBipSeedGauge::init`에서 사용하기 위한 마이그레이션되지 않은 BDV의 잘못된 계산

**설명:** `InitBipSeedGauge::init`의 [상수](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/init/InitBipSeedGauge.sol#L35-L39)에 대한 현재 값은 추정치이며 확정된 것이 아닙니다. BDV를 올바르게 계산하기 위해 Beanstalk Farms 팀은 BIP-38이 실행된 블록에서 남은 마이그레이션되지 않은 모든 예금을 마이그레이션하는 시뮬레이션을 수행하여 `BDVFacet::unripeLPToBDV`의 기본 자산에 해당하는 BDV 변경 사항을 고려하고 유동성 마이그레이션 시 발생한 슬리피지의 영향을 받도록 합니다. `deposits.json` 파일에는 Silo V3 배포 블록 `17671557`의 미해결 예금 목록이 포함되어 있으므로 스크립트는 이 시점 이후의 모든 `removeDeposit` 이벤트를 마이그레이션되지 않은 BDV에서 제거할 예금으로 간주합니다. Enroot 수정 배포 블록 `17251905`부터 필터링하여 계정이 Enroot 수정 후 Silo V3가 배포되기 전에 예금을 제거한 경우 예금이 마이그레이션되지 않았음에도 불구하고 마이그레이션된 것으로 부적절하게 가정합니다. 또한 스크립트가 BIP-38 실행 블록 `18392690`에서 메인넷을 포크하고 있으므로 이벤트 필터링을 위한 종료 블록으로 `18480579`를 사용하는 것은 올바르지 않습니다.

상태 변경이 이미 적용되었고 마이그레이션 트랜잭션이 블록의 상단/하단이 아니라고 가정할 때, BIP-38 실행 전 블록까지 포크/필터링하고 수동으로 고려해야 할 마이그레이션 트랜잭션 전/후에 마이그레이션이 발생했는지 확인하는 것이 바람직할 수 있다는 경우도 고려되었습니다. BIP-38 업그레이드가 발생한 블록을 추가로 검사한 결과 이벤트가 발생하지 않았으므로 이것이 필요하지 않은 것으로 보입니다.

마이그레이션되지 않은 Bean BDV 값의 추가 불일치가 Beanstalk Farms 팀에 의해 확인되었습니다. Silo V3 이후 `Sun::rewardToSilo`의 구현은 Silo에 발행된 Bean 양만큼 [BDV를 증가](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/sun/SeasonFacet/Sun.sol#L201-L204)시키지만 이전에 획득한 모든 Bean은 고려되지 않습니다. 따라서 Silo V3 배포 시 `SiloExit::totalEarnedBeans`가 반환한 값을 합계에 추가해야 합니다.

**영향:** 아래와 같이 계산된 마이그레이션되지 않은 BDV가 잘못되었습니다. 현재 구현은 의도한 것보다 작은 값을 반환하므로 총 예치 BDV는 일부 예금을 고려하지 못하고 의도한 것보다 낮아집니다.

현재 구현의 출력:
```
unmigrated:  {
  '0x1BEA0050E63e05FBb5D8BA2f10cf5800B6224449': BigNumber { value: "3209210313166" },
  '0x1BEA3CcD22F4EBd3d37d731BA31Eeca95713716D': BigNumber { value: "6680992571569" },
  '0xBEA0000029AD1c77D3d5D23Ba2D8893dB9d1Efab': BigNumber { value: "304630107407" },
  '0xc9C32cd16Bf7eFB85Ff14e0c8603cc90F6F2eE49': BigNumber { value: "26212521946" }
}
```

수정된 출력:
```
unmigrated:  {
  '0x1BEA0050E63e05FBb5D8BA2f10cf5800B6224449': BigNumber { value: "3736196158417" },
  '0x1BEA3CcD22F4EBd3d37d731BA31Eeca95713716D': BigNumber { value: "7119564766493" },
  '0xBEA0000029AD1c77D3d5D23Ba2D8893dB9d1Efab': BigNumber { value: "689428296238" },
  '0xc9C32cd16Bf7eFB85Ff14e0c8603cc90F6F2eE49': BigNumber { value: "26512602424" }
}
```

**권장 완화 방법:** 다음 diff를 적용하십시오:
```diff
// L645
- const END_BLOCK = 18480579;
+ const END_BLOCK = BLOCK_NUMBER;

// L811-812
- //get every transaction that emitted the RemoveDeposit event after block 17251905
+ //get every transaction that emitted the RemoveDeposit event after block 17671557
- let events = await queryEvents("RemoveDeposit(address,address,uint32,uint256)", removeDepositInterface, 17251905); //update this block to latest block when running actual script, in theory someone could have migrated meanwhile
+ let events = await queryEvents("RemoveDeposit(address,address,uint32,uint256)", removeDepositInterface, 17671557);
```
Silo에 이전에 발행된 Bean 양 검색:
```bash
cast call 0xC1E088fC1323b20BCBee9bd1B9fC9546db5624C5 "totalEarnedBeans()" --rpc-url ${FORKING_RPC} --block "17671557"
```

\clearpage
## 낮은 위험 (Low Risk)


### `LibWhitelist::verifyTokenInLibWhitelistedTokens`의 누락된 유효성 검사

**설명:** [`LibWhitelistedToken.sol`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibWhitelistedTokens.sol)이 도입되기 전에 Beanstalk에는 화이트리스트 토큰을 반복할 수 있는 방법이 없었습니다. 새 자산이 화이트리스트에 추가되지만 `LibWhitelistedToken.sol`은 업데이트되지 않는 업그레이드를 방지하기 위해 [`LibWhitelist::verifyTokenInLibWhitelistedTokens`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibWhitelist.sol#L196-L217)는 토큰이 올바른 배열에 있고 잘못된 배열에 없는지 확인합니다.

`LibWhitelistedTokens::getWhitelistedWellLpTokens`는 화이트리스트 LP 토큰의 하위 집합을 반환해야 하지만 이는 보장되지 않습니다. 이 경우 토큰이 Bean이거나 Unripe 토큰인 경우 `LibWhitelist::verifyTokenInLibWhitelistedTokens` 내의 첫 번째 `else` 블록에서도 토큰이 화이트리스트 Well LP 토큰 배열에 없는지 확인해야 합니다.

**권장 완화 방법:**
```diff
} else {
    checkTokenNotInArray(token, LibWhitelistedTokens.getWhitelistedLpTokens());
+   checkTokenNotInArray(token, LibWhitelistedTokens.getWhitelistedWellLpTokens());
}
```


### 음수 `int96` 값에서 잠재적으로 안전하지 않은 캐스트

**설명:** [`LibSilo::stalkReward`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibSilo.sol#L634-L638), [`LibTokenSilo::grownStalkForDeposit`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibTokenSilo.sol#L422), 및 [`LibTokenSilo::calculateGrownStalkAndStem`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibTokenSilo.sol#L452)에서 stem을 조작할 때와 같이 `int96` 값에 대해 계산이 수행되는 경우 Beanstalk는 `LibSafeMathSigned96` 라이브러리를 사용합니다. 새 예금의 stem이 주어진 토큰의 stem tip을 초과해서는 안 된다는 불변성을 기반으로 이 값을 `uint256`으로 캐스팅하는 것은 두 stem 값의 차이가 음수여서는 안 되기 때문에 괜찮습니다. 그러나 이 불변성을 위반하는 버그가 발생할 경우 음수 `int96` 값이 매우 큰 `uint256` 값으로 캐스팅되어 잠재적으로 엄청난 양의 stalk가 발행될 수 있습니다.

이 문제는 `LibTokenSilo::grownStalkForDeposit`에서 이미 충분히 [완화](https://github.com/BeanstalkFarms/Beanstalk/blob/08ca0d7d495c94f2a4366fb7f99da561b74cc1c0/protocol/contracts/libraries/Silo/LibTokenSilo.sol#L419-L422)되었으며 `LibTokenSilo::calculateGrownStalkAndStem`의 인스턴스는 이 상태에 도달할 수 없는 것으로 보입니다. 뺄셈 결과가 양수이고 따라서 `uint256`으로의 캐스트가 안전한지 확인하기 위해 유사한 추가 로직을 `LibSilo::stalkReward`에 추가해야 합니다.

**영향:** 현재 악용 가능하지 않은 것으로 보이지만 `LibSilo::stalkReward`에서 특정 예금에 대한 stem 또는 특정 토큰에 대한 stem tip 계산 버그로 인해 많은 양의 stalk가 잘못 발행될 수 있습니다.

**개념 증명 (Proof of Concept):** 다음 forge 테스트는 이 문제를 보여줍니다:
```solidity
contract TestStemsUnsafeCasting is Test {
    using LibSafeMathSigned96 for int96;

    function stalkReward(int96 startStem, int96 endStem, uint128 bdv)
        internal
        view
        returns (uint256)
    {
        int96 reward = endStem.sub(startStem).mul(int96(bdv));
        console.logInt(reward);
        console.logUint(uint128(reward));

        return uint128(reward);
    }

    function test_stalk_reward() external {
        uint256 reward = stalkReward(1200, 1000, 1337);
        console.logUint(reward);
    }
}
```

**권장 완화 방법:** `int96`에서 안전하게 캐스트를 수행하거나 stem 뺄셈 결과가 음수일 수 있는 경우를 처리하기 위해 추가 로직을 추가하십시오.


### `LibWell::getWellPriceFromTwaReserves`에서 두 준비금 모두 확인해야 함

**설명:**
```solidity
function getWellPriceFromTwaReserves(address well) internal view returns (uint256 price) {
    AppStorage storage s = LibAppStorage.diamondStorage();
    // s.twaReserve[well] should be set prior to this function being called.
    // 'price' is in terms of reserve0:reserve1.
    if (s.twaReserves[well].reserve0 == 0) {
        price = 0;
    } else {
        price = s.twaReserves[well].reserve0.mul(1e18).div(s.twaReserves[well].reserve1);
    }
}
```

현재 `LibWell::getWellPriceFromTwaReserves`는 0번째 준비금(Wells의 경우 Bean)의 시간 가중 평균 준비금이 0이면 가격을 0으로 설정합니다. `LibWell::setTwaReservesForWell`의 구현과 Pump 실패 시 빈 준비금 배열을 반환한다는 점을 감안할 때, 익스플로잇이나 마이그레이션 시나리오를 제외하고는 다른 준비금 없이 하나의 준비금이 0이 되는 경우를 만나는 것은 불가능해 보입니다. 따라서 가능성은 낮지만 가격 계산 시 `reserve1`이 0으로 나누어지는 잠재적인 오류를 피하기 위해 두 준비금이 모두 0이 아닌지 확인하는 것이 가장 좋습니다. 여기서 되돌리면 `SeasonFacet::gm`의 DoS가 발생하기 때문입니다.

```solidity
function setTwaReservesForWell(address well, uint256[] memory twaReserves) internal {
    AppStorage storage s = LibAppStorage.diamondStorage();
    // if the length of twaReserves is 0, then return 0.
    // the length of twaReserves should never be 1, but
    // is added for safety.
    if (twaReserves.length < 1) {
        delete s.twaReserves[well].reserve0;
        delete s.twaReserves[well].reserve1;
    } else {
        // safeCast not needed as the reserves are uint128 in the wells.
        s.twaReserves[well].reserve0 = uint128(twaReserves[0]);
        s.twaReserves[well].reserve1 = uint128(twaReserves[1]);
    }
}
```

또한 `LibWell::setTwaReservesForWell`의 주석으로 식별된 검사를 올바르게 구현하려면 배열 길이가 1보다 작거나 같으면 스토리지의 시간 가중 평균 준비금을 재설정해야 합니다.

**권장 완화 방법:**
```diff
// LibWell::getWellPriceFromTwaReserves`
- if (s.twaReserves[well].reserve0 == 0) {
+ if (s.twaReserves[well].reserve0 == 0 || s.twaReserves[well].reserve1 == 0) {
        price = 0;
} else {

// LibWell::setTwaReservesForWell
- if (twaReserves.length < 1) {
+ if (twaReserves.length <= 1) {
    delete s.twaReserves[well].reserve0;
    delete s.twaReserves[well].reserve1;
} else {
```


### `LibGauge::updateGaugePoints`에서 0으로 나누기로 인한 `SeasonFacet::gm`의 잠재적 DoS

[`LibGauge::updateGaugePoints`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibGauge.sol#L84)에는 현재 잠재적인 0으로 나누기로 인해 `SeasonFacet::gm`을 의도치 않게 DoS할 수 있는 엣지 케이스가 존재합니다. Beanstalk 프로토콜에 새로 화이트리스트에 추가된 LP 토큰이 하나만 있고 따라서 예치된 BDV가 없는 경우 실행이 되돌려져 Beanstalk가 다음 시즌으로 진행되지 않습니다. 기존 화이트리스트 LP 토큰이 남아 있는 한 Beanstalk가 이 문제를 겪을 가능성은 낮지만 향후 유동성 마이그레이션 발생 시 문제가 될 수 있는 작은 가능성이 있으므로 그에 따라 처리해야 합니다.

```diff
...
// if there is only one pool, there is no need to update the gauge points.
if (whitelistedLpTokens.length == 1) {
    // Assumes that only Wells use USD price oracles.
    if (LibWell.isWell(whitelistedLpTokens[0]) && s.usdTokenPrice[whitelistedLpTokens[0]] == 0) {
        return (maxLpGpPerBdv, lpGpData, totalGaugePoints, type(uint256).max);
    }
    uint256 gaugePoints = s.ss[whitelistedLpTokens[0]].gaugePoints;
+   if (s.siloBalances[whitelistedLpTokens[0]].depositedBdv != 0) {
        lpGpData[0].gpPerBdv = gaugePoints.mul(BDV_PRECISION).div(
            s.siloBalances[whitelistedLpTokens[0]].depositedBdv
        );
+   }
    return (
        lpGpData[0].gpPerBdv,
        lpGpData,
        gaugePoints,
        s.siloBalances[whitelistedLpTokens[0]].depositedBdv
    );
}
...
```


### 소액의 unripe 토큰 인출은 BDV와 Stalk를 감소시키지 않음

**설명:** `bdvCalc(amountDeposited) < amountDeposited`인 모든 화이트리스트 토큰의 경우 사용자는 해당 토큰을 예치한 다음 BDV 및 Stalk 감소를 피하기 위해 소액으로 인출할 수 있습니다. 이는 [`LibTokenSilo::removeDepositFromAccount`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibTokenSilo.sol#L248)에서 0 정밀도 손실로 반올림되는 것을 악용하여 달성됩니다:

```solidity
// @audit small unripe bean withdrawals don't decrease BDV and Stalk
// due to rounding down to zero precision loss. Every token where
// `bdvCalc(amountDeposited) < amountDeposited` is vulnerable
uint256 removedBDV = amount.mul(crateBDV).div(crateAmount);
```

**영향:** 공격자는 BDV 및 Stalk를 줄이지 않고 예치된 자산을 인출할 수 있습니다. 이 공격을 수행하는 비용은 공격자가 얻을 수 있는 가치보다 클 가능성이 높지만, 특히 BIP-39에 Unripe Chop Convert가 도입된 것을 고려할 때 이 버그와 관련하여 의도하지 않은 다른 결과가 발생할 수 있으므로 잠재적인 영향을 더 면밀히 조사해야 합니다(Unripe 토큰의 부풀려진 BDV가 예금이 ripe 토큰으로 전환되면 지속되어 이 BDV가 다른 곳에서 사용/조작되는 방식에 따라 잠재적으로 가치를 추출할 수 있음).

이 버그에 대한 다른 주요 고려 사항은 예치된 자산을 인출할 때 Stalk가 손실되어야 한다는 메커니즘을 깨뜨리고 `totalDepositedBdv`를 인위적으로 높게 유지하여 토큰의 `totalDepositedBdv` 값이 모든 개별 예금의 BDV 값의 합이어야 한다는 불변성을 위반한다는 것입니다.

**개념 증명 (Proof of Concept):** `describe("1 deposit, some", async function () {` 섹션 아래 `SiloToken.test.js`에 이 PoC를 추가하십시오:

```javascript
it('audit small unripe bean withdrawals dont decrease BDV and Stalks', async function () {
    let initialUnripeBeanDeposited    = to6('10');
    let initialUnripeBeanDepositedBdv = '2355646';
    let initialTotalStalk = pruneToStalk(initialUnripeBeanDeposited).add(toStalk('0.5'));

    // verify initial state
    expect(await this.silo.getTotalDeposited(UNRIPE_BEAN)).to.eq(initialUnripeBeanDeposited);
    expect(await this.silo.getTotalDepositedBdv(UNRIPE_BEAN)).to.eq(initialUnripeBeanDepositedBdv);
    expect(await this.silo.totalStalk()).to.eq(initialTotalStalk);

    // snapshot EVM state as we want to restore it after testing the normal
    // case works as expected
    let snapshotId = await network.provider.send('evm_snapshot');

    // normal case: withdrawing total UNRIPE_BEAN correctly decreases BDV & removes stalks
    const stem = await this.silo.seasonToStem(UNRIPE_BEAN, '10');
    await this.silo.connect(user).withdrawDeposit(UNRIPE_BEAN, stem, initialUnripeBeanDeposited, EXTERNAL);

    // verify UNRIPE_BEAN totalDeposited == 0
    expect(await this.silo.getTotalDeposited(UNRIPE_BEAN)).to.eq('0');
    // verify UNRIPE_BEAN totalDepositedBDV == 0
    expect(await this.silo.getTotalDepositedBdv(UNRIPE_BEAN)).to.eq('0');
    // verify silo.totalStalk() == 0
    expect(await this.silo.totalStalk()).to.eq('0');

    // restore EVM state to snapshot prior to testing normal case
    await network.provider.send("evm_revert", [snapshotId]);

    // re-verify initial state
    expect(await this.silo.getTotalDeposited(UNRIPE_BEAN)).to.eq(initialUnripeBeanDeposited);
    expect(await this.silo.getTotalDepositedBdv(UNRIPE_BEAN)).to.eq(initialUnripeBeanDepositedBdv);
    expect(await this.silo.totalStalk()).to.eq(initialTotalStalk);

    // attacker case: withdrawing small amounts of UNRIPE_BEAN doesn't decrease
    // BDV and doesn't remove stalks. This lets an attacker withdraw their deposits
    // without losing Stalks & breaks the invariant that the totalDepositedBDV should
    // equal the sum of the BDV of all individual deposits
    let smallWithdrawAmount = '4';
    await this.silo.connect(user).withdrawDeposit(UNRIPE_BEAN, stem, smallWithdrawAmount, EXTERNAL);

    // verify UNRIPE_BEAN totalDeposited has been correctly decreased
    expect(await this.silo.getTotalDeposited(UNRIPE_BEAN)).to.eq(initialUnripeBeanDeposited.sub(smallWithdrawAmount));
    // verify UNRIPE_BEAN totalDepositedBDV remains unchanged!
    expect(await this.silo.getTotalDepositedBdv(UNRIPE_BEAN)).to.eq(initialUnripeBeanDepositedBdv);
    // verify silo.totalStalk() remains unchanged!
    expect(await this.silo.totalStalk()).to.eq(initialTotalStalk);
});
```
실행: `npx hardhat test --grep "audit small unripe bean withdrawals dont decrease BDV and Stalks"`.

Beanstalk의 현재 및 BIP-39 이후 배포에서 이 버그의 존재를 입증하기 위해 추가 메인넷 포크 테스트가 작성되었습니다(부록 B 참조).

**권장 완화 방법:** `removedBDV == 0`인 경우 `LibTokenSilo::removeDepositFromAccount`는 되돌려져야(revert) 합니다. 유사한 검사가 이미 [`LibTokenSilo::depositWithBDV`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibTokenSilo.sol#L141)에 존재하지만 부분 인출에 대해 `removedBDV`를 계산할 때 `removeDepositFromAccount()`에는 누락되었습니다.

프로토콜 불변성의 파괴는 아직 식별되지 않았지만 핵심 속성이 유지되지 않는 경우 존재할 수 있는 다른 심각한 문제로 이어질 수 있습니다. 팀이 BIP-39 업그레이드 전이나 그 일환으로 가능한 한 빨리 이 버그를 수정하는 것을 고려할 것을 강력히 촉구합니다.


### 안전하지 않은 다운캐스트로 인해 대규모 부분 인출 시 Stalk 보상이 소각되지 않음

**설명:** `SiloFacet::withdrawDeposit`를 호출할 때 `removedBDV`를 `uint128 -> int96`으로 [안전하지 않게 다운캐스트](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibSilo.sol#L631-L636)하여 `LibSilo::stalkReward`가 0을 반환하므로 대규모 부분 인출에 대해 Stalk 보상이 소각되지 않을 수 있습니다.

**영향:** 대규모 부분 인출 시 Stalk 보상이 소각되지 않습니다.

**개념 증명 (Proof of Concept):** `describe("deposit", function () {` 섹션 아래 `SiloToken.test.js`에 추가하십시오:
```javascript
    describe("audit withdrawing deposited asset for large BDV value", function () {
      // values found via fuzz testing
      let beanDeposit       = "79228162514264337593543950337";
      let problemRemovedBdv = "79228162514264337593543950336";

      beforeEach(async function () {
        await this.season.teleportSunrise(10);
        this.season.deployStemsUpgrade();

        await this.siloToken.connect(user).approve(this.silo.address, beanDeposit);
        await this.siloToken.mint(userAddress, beanDeposit);
        await this.silo.connect(user).deposit(this.siloToken.address, beanDeposit, EXTERNAL);
      });

      it("audit stalk rewards not burned when withdrawing deposited asset for large BDV value", async function () {
        let initialTotalStalk = beanDeposit + "0000";

        // verify initial state
        expect(await this.silo.getTotalDeposited(this.siloToken.address)).to.eq(beanDeposit);
        // siloToken has 1:1 BDV calc
        expect(await this.silo.getTotalDepositedBdv(this.siloToken.address)).to.eq(beanDeposit);
        expect(await this.silo.totalStalk()).to.eq(initialTotalStalk);

        // fast forward to build up some stalk rewards
        await this.season.teleportSunrise(20);

        // snapshot EVM state as we want to restore it after testing the normal
        // case works as expected
        let snapshotId = await network.provider.send("evm_snapshot");

        // normal case: withdraw the entire deposited amount
        const stem = await this.silo.seasonToStem(this.siloToken.address, "10");
        await this.silo.connect(user).withdrawDeposit(this.siloToken.address, stem, beanDeposit, EXTERNAL);

        // verify token.totalDeposited == 0
        expect(await this.silo.getTotalDeposited(this.siloToken.address)).to.eq("0");
        // verify token.totalDepositedBDV == 0
        expect(await this.silo.getTotalDepositedBdv(this.siloToken.address)).to.eq("0");
        // verify totalStalk == 0; both the initial stalk & stalk rewards were burned
        expect(await this.silo.totalStalk()).to.eq("0");

        // restore EVM state to snapshot prior to testing normal case
        await network.provider.send("evm_revert", [snapshotId]);

        // re-verify initial state
        expect(await this.silo.getTotalDeposited(this.siloToken.address)).to.eq(beanDeposit);
        // siloToken has 1:1 BDV calc
        expect(await this.silo.getTotalDepositedBdv(this.siloToken.address)).to.eq(beanDeposit);
        expect(await this.silo.totalStalk()).to.eq(initialTotalStalk);

        // problem case: partial withdraw a precise amount causing
        // by LibTokenSilo::removeDepositFromAccount() to calculate & return
        // `removedBDV` to a known exploitable value. This causes LibSilo::stalkReward()
        // to return 0 due to an unsafe downcast of `removedBDV` from uint128 -> int96
        // meaning stalk rewards are not burned when the withdrawal occurs
        await this.silo.connect(user).withdrawDeposit(this.siloToken.address, stem, problemRemovedBdv, EXTERNAL);

        // verify token.totalDeposited has been correcly decremented
        expect(await this.silo.getTotalDeposited(this.siloToken.address)).to.eq("1");
        // verify token.totalDepositedBDV == 1 as siloToken has 1:1 BDV calc
        expect(await this.silo.getTotalDepositedBdv(this.siloToken.address)).to.eq("1");

        // verify totalStalk == 10000 which fails and instead 10010 is returned.
        //
        // A return of 10010 is incorrect as there is only 1 BEAN left deposited
        // so totalStalk should equal 10000 as the 10 stalk rewards should have
        // been burned with the withdrawal, but this didn't happen due to the
        // unsafe downcast in LibSilo::stalkReward() causing stalkReward() to
        // return 0
        expect(await this.silo.totalStalk()).to.eq("10000");
      });
    });
```

**권장 완화 방법:** 다운캐스트 결과가 오버플로되어 Stalk가 소각되지 않는 경우 인출은 되돌려져야 합니다. 이는 안전한 다운캐스트를 수행하거나 0이 아닌 BDV를 인출할 때 0이 아닌 Stalk 양이 소각되는지 확인함으로써 달성할 수 있습니다. [`LibTokenSilo::toInt96`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibTokenSilo.sol#L478-L481)은 해당 계약에서 입력 유효성 검사 및 캐스팅에 사용됩니다.

\clearpage
## 정보성 (Informational)


### `Storage::SiloSettings`의 잘못된 스토리지 슬롯 주석

스토리지의 주문 구조체 멤버는 변경되지 않은 것으로 보이지만, `AppStorage.sol`의 [`Storage::SiloSettings`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/AppStorage.sol#L393-L404) 스토리지 슬롯 주석이 잘못되었으므로 다음과 같이 업데이트해야 합니다:

```diff
struct SiloSettings {
    bytes4 selector; // ─────────────┐ 4
-   uint32 stalkEarnedPerSeason; //  │ 4  (16)
+   uint32 stalkEarnedPerSeason; //  │ 4  (8)
-   uint32 stalkIssuedPerBdv; //     │ 4  (8)
+   uint32 stalkIssuedPerBdv; //     │ 4  (12)
-   uint32 milestoneSeason; //       │ 4  (12)
+   uint32 milestoneSeason; //       │ 4  (16)
    int96 milestoneStem; //          │ 12 (28)
    bytes1 encodeType; // ───────────┘ 1  (29)
    // 3 bytes are left here.
    uint128 gaugePoints; //   ────--------───┐ 16
    bytes4 gpSelector; //                    │ 4   (20)
    uint96 optimalPercentDepositedBdv; // ───┘ 12  (32)
}
```


### `LibLockedUnderlying` 회귀는 예상되는 동작을 나타내지 않을 수 있음

`LibEvaluate`에서 L2SR을 결정하는 데 사용되는 잠긴 유동성 비율은 오프체인 선형 회귀를 기반으로 하는 온체인 조회 테이블의 구현을 통해 얻어집니다. Unripe Bean과 Unripe LP 모두에 대해 허용 가능한 것으로 간주되는 가정은 각 단계에서 46,659개의 Unripe 토큰이 Chop된다는 것입니다. 이 숫자는 Unripe 토큰 수를 Unripe 토큰 보유자 수로 나누어 계산되며, 결과적으로 0이 아닌 잔액을 가진 Farmer당 평균 46,659개의 Unripe 토큰을 보유하게 됩니다. 2000은 Unripe Bean 및 Unripe LP 보유자 수에 대한 약간의 과대평가로 사용됩니다. 과대평가는 더 보수적인 L2SR을 초래하므로 허용됩니다.

그러나 이 평균은 각 Chop에서 예상되는 것을 정확하게 나타내지 않을 수 있습니다. 예를 들어, 9명의 사용자가 각각 100,000개의 Unripe 토큰을 가지고 있고 한 명의 사용자가 510만 개의 Unripe 토큰을 가지고 있는 시나리오를 생각해 보십시오.

$$\frac{Number\ of \ Unripe\; Tokens}{Number\ of \ Unripe\ Token \ Holders}= \frac{9 \times 100.000 + 5.100.000}{10} =$$

$$\frac{900.000 + 5.100.000}{10} = \frac{6.000.000}{10}=600.000$$

이 경우 회귀는 각 단계에서 `600,000`개의 Unripe 토큰이 Chop된다고 간주하며, 이는 실제로 단 한 명의 사용자에 의해서만 수행될 수 있습니다. 따라서 여기서는 평균보다는 최빈값이나 중앙값을 사용하는 것이 더 좋습니다.


### PR 및 인라인 주석의 오래된 시드 게이지 시스템 문서

PR 및 인라인 주석 모두에 시드 게이지 시스템 문서가 오래된 여러 인스턴스가 있습니다. 예를 들어:
- `LibWhitelist::updateGaugeForToken`은 게이지 포인트를 변경하는 것을 허용하지 않습니다. 이는 [`WhitelistFacet::updateGaugeForToken`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/silo/WhitelistFacet.sol#L172)의 주석과 반대됩니다.
- [`LibGauge::getBeanToMaxLpGpPerBdvRatioScaled`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibGauge.sol#L316-L329)의 동작은 실제 동작의 역으로 잘못 문서화되어 있습니다. 실제로는 `f(0) = MIN_BEAN_MAX_LPGP_RATIO` 및 `f(100e18) = MAX_BEAN_MAX_LPGP_RATIO`입니다.
- PR에 명시된 대로 게이지 포인트는 `100e18`로 정규화되지 않습니다.
- [`MIN_BEAN_MAX_LP_GP_PER_BDV_RATIO`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibGauge.sol#L32) 상수는 PR에 명시된 `25e18`이 아니라 실제로 `50e18`입니다.
- `gpPerBdv` 및 `beanToMaxLpGpPerBdvRatio`는 모두 소수점 18자리 정밀도를 갖지만, [여러](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibGauge.sol#L153) [주석](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibGauge.sol#L210)에서 이 변수들이 소수점 6자리 정밀도를 갖는다고 [잘못](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibGauge.sol#L215) 명시하고 있습니다.


### `LibChop`과 `LibUnripe` 간의 중복 코드

[`LibUnripe::_getPenalizedUnderlying`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibUnripe.sol#L143-L151) 및 [`LibUnripe::isUnripe`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibUnripe.sol#L217-L223)의 중복 버전이 [`LibChop`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibChop.sol#L38-L64)에 추가되었습니다. 실행되는 로직이 동일하므로 필요하지 않으며 코드 재사용을 위해 이러한 중복 버전을 제거할 수 있습니다.


### urBEAN3CRV 변환에 대한 오래된 참조

`LibConvert::getMaxAmountIn` 및 `LibConvert::getAmountOut`에는 urBEAN3CRV 및 BEAN:3CRV로의 변환을 참조하는 여러 인스턴스가 있습니다. 기본 유동성이 이제 BEAN:ETH Well로 마이그레이션되었으므로 Unripe Bean 또는 Unripe LP를 BEAN:3CRV로 변환하는 것은 더 이상 불가능합니다. 따라서 다음 diff를 적용해야 합니다:

```diff
...
-   // urBEAN3CRV Convert
+   // urBEAN:ETH Convert
    if (tokenIn == C.UNRIPE_LP){
-       // urBEAN:3CRV -> urBEAN
+       // urBEAN:ETH -> urBEAN
        if(tokenOut == C.UNRIPE_BEAN)
            return LibUnripeConvert.lpToPeg();
-       // UrBEAN:3CRV -> BEAN:3CRV
+       // UrBEAN:ETH -> BEAN:ETH
-       if(tokenOut == C.CURVE_BEAN_METAPOOL)
+       if(tokenOut == C.BEAN_ETH_WELL)
        return type(uint256).max;
    }

    // urBEAN Convert
    if (tokenIn == C.UNRIPE_BEAN){
-       // urBEAN -> urBEAN:3CRV LP
+       // urBEAN -> urBEAN:ETH LP
        if(tokenOut == C.UNRIPE_LP)
            return LibUnripeConvert.beansToPeg();
        // UrBEAN -> BEAN
...
-   /// urBEAN:3CRV LP -> urBEAN
+   /// urBEAN:ETH LP -> urBEAN
    if (tokenIn == C.UNRIPE_LP && tokenOut == C.UNRIPE_BEAN)
        return LibUnripeConvert.getBeanAmountOut(amountIn);

-   /// urBEAN -> urBEAN:3CRV LP
+   /// urBEAN -> urBEAN:ETH LP
    if (tokenIn == C.UNRIPE_BEAN && tokenOut == C.UNRIPE_LP)
        return LibUnripeConvert.getLPAmountOut(amountIn);
...
-   // UrBEAN:3CRV -> BEAN:3CRV
+   // UrBEAN:ETH -> BEAN:ETH
-   if (tokenIn == C.UNRIPE_LP && tokenOut == C.CURVE_BEAN_METAPOOL)
+   if (tokenIn == C.UNRIPE_LP && tokenOut == C.BEAN_ETH_WELL)
        return LibChopConvert.getConvertedUnderlyingOut(tokenIn, amountIn);
...
```


### 기타 NatSpec 및 인라인 주석 오류

다음과 같은 NatSpec 오류가 식별되었습니다:
- [`UnripeFacet::balanceOfPenalizedUnderlying`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/barn/UnripeFacet.sol#L208-L214) NatSpec은 [`UnripeFacet::balanceOfUnderlying`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/barn/UnripeFacet.sol#L194-L201)에서 잘못 복사되었으므로 이 특정 함수에 맞게 수정해야 합니다.
- [`LibWell::getTwaReservesForWell`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Well/LibWell.sol#L189)의 NatSpec이 올바르지 않습니다. 이 함수가 `{AppStorage.usdTokenPrice}`에 저장된 `USD / TKN` 가격을 반환한다고 명시하고 있지만 실제로는 `TKN / USD` 가격이므로 그에 따라 업데이트해야 합니다.
- [`Weather::updateTemperature`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/sun/SeasonFacet/Weather.sol#L90) 구현을 설명하는 [주석](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/sun/SeasonFacet/Weather.sol#L90)은 `uint32(-change)`를 잘못 참조하고 있으며 `uint256(-change)`여야 합니다.
- `LibCases`의 상수의 동작을 설명하는 [주석](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibCases.sol#L55)은 `Bean2maxLpGpPerBdv set to 10% of current value`라고 잘못 명시하고 있으며 현재 값의 50%여야 합니다.
- `InitBipNewSilo`의 [주석](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/init/InitBipNewSilo.sol#L77)은 `stemStartSeason`이 `uint32`로 저장된다고 명시하도록 변경되었지만 실제로는 `uint16`입니다.
- `AppStorage`의 [`int8[32] cases`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/AppStorage.sol#L456) 멤버에 대한 NatSpec은 오래되었으며 [멤버 자체](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/AppStorage.sol#L507)와 함께 `bytes32[144] casesV2`를 위해 더 이상 사용되지 않음(deprecated)으로 표시되어야 합니다.
- `AppStorage`의 NatSpec에서 [`TwaReserves`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/AppStorage.sol#L499)의 경우 약간의 오류가 있으며 대신 `twaReserves`여야 합니다.
- [`deprecated_beanEthPrice`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/AppStorage.sol#L564)는 현재 `AppStorage`의 NatSpec에 존재하지 않습니다. 이 멤버가 더 이상 사용되지 않는 이유에 대한 설명과 함께 추가되어야 합니다.


### `try/catch` 블록을 사용하여 `LibWell`의 Beanstalk Pump에서 시간 가중 평균 준비금을 읽어야 함

[`LibWell::getTwaReservesFromBeanstalkPump`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Well/LibWell.sol#L242) 및 [`LibWell::getTwaLiquidityFromBeanstalkPump`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Well/LibWell.sol#L262)에는 Beanstalk Pump에서 시간 가중 평균 준비금을 직접 읽는 인스턴스가 있습니다. [`LibWellMinting::twaDeltaB`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Minting/LibWellMinting.sol#L160)의 구현과 달리 이 함수들은 호출을 `try/catch` 블록으로 래핑하지 않습니다. `LibWell::getTwaReservesFromStorageOrBeanstalkPump`의 실행이 `LibWell::getTwaReservesFromBeanstalkPump`의 [호출](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Well/LibWell.sol#L228)에 도달하지 않으므로(여기서는 준비금이 이미 스토리지에 설정되어 있으므로) Beanstalk Sunrise 메커니즘에 영향을 주지 않아야 합니다. 그러나 [`LibEvaluate::calcLPToSupplyRatio`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibEvaluate.sol#L222) 및 [`SeasonGettersFacet::getBeanEthTwaUsdLiquidity`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/sun/SeasonFacet/SeasonGettersFacet.sol#L227-L232)([`SeasonGettersFacet::getTotalUsdLiquidity`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/sun/SeasonFacet/SeasonGettersFacet.sol#L238-L240)에서도 사용됨)가 문제가 있을 경우 되돌려지지 않도록 Pump 실패를 우아하게 처리하는 것을 고려하십시오.


### BDV당 평균 grown stalk의 사용이 올바르게 문서화되지 않음

게이지 포인트 시스템의 본질은 BDV(Bean-denominated value)를 기반으로 화이트리스트 LP 및 BEAN 예금 간에 새로운 stalk를 분배하는 것입니다. BDV당 평균 grown stalk는 또한 다음 비율을 기반으로 해당 기본 자산에 연결된 unripe 자산의 BDV를 고려합니다:

$$\frac{paidFertilizer}{mintedFertilzier} \times \frac{totalUnderlying(urAsset)}{supply(urAsset)}$$

BDV당 평균 grown stalk에 대해 unripe 자산의 BDV를 고려하는 것은 수학적으로 정확하지만, 이 지표는 BDV당 평균 grown stalk의 일부가 발행되지 않아 의미론적 의미를 잃게 되므로 실용적인 의미가 부족합니다.

```solidity
// LibGauge::updateGrownStalkEarnedPerSeason
uint256 totalBdv = totalLpBdv.add(beanDepositedBdv);
...
uint256 newGrownStalk = uint256(s.seedGauge.averageGrownStalkPerBdvPerSeason)
    .mul(totalBdv) // This BDV does not include unripe asset BDV
    .div(BDV_PRECISION);
```

보시다시피 이 계산의 명확한 의도는 한 시즌 동안 `newGrownStalk`를 발행하는 것이지만 화이트리스트 LP 및 BEAN에 해당하는 BDV만 고려합니다. 결코 발행되지 않는다는 점을 감안할 때, BDV당 나머지 grown stalk는 암시적으로 소각된 것으로 간주될 수 있습니다. 이 설계 결정은 발행되지 않은 grown stalk가 어떻게 고려되는지 명확히 하여 더 잘 문서화되어야 합니다.


### `ConvertFacet::_withdrawTokens`의 불필요한 코드 중복 통합

`ConvertFacet::_withdrawTokens`는 [L119-132](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/silo/ConvertFacet.sol#L119-L132)에서 다음 코드를 복제한 다음 [L137-151](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/silo/ConvertFacet.sol#L137-L151)에서 다시 복제합니다.

필요할 때만 `amounts[i]`를 업데이트하도록 `if` 조건을 변경한 다음 현재 각 `if/else` 분기에 있는 것과 동일한 처리를 수행하여 중복 코드를 제거하도록 리팩토링하는 것을 고려하십시오:

```solidity
while ((i < stems.length) && (a.tokensRemoved < maxTokens)) {
    if (a.tokensRemoved.add(amounts[i]) >= maxTokens) {
        amounts[i] = maxTokens.sub(a.tokensRemoved);
    }

    //keeping track of stalk removed must happen before we actually remove the deposit
    //this is because LibTokenSilo.grownStalkForDeposit() uses the current deposit info
    depositBDV = LibTokenSilo.removeDepositFromAccount(
        msg.sender,
        token,
        stems[i],
        amounts[i]
    );
...
```


\clearpage
## 가스 최적화 (Gas Optimization)


### 조건이 충족되면 `LibWhitelist` 루프에서 조기 종료

주어진 주소가 `LibWhitelist::checkTokenInArray` 또는 `LibWhitelist::checkTokenNotInArray`에 전달된 배열에서 발견되면, 불필요한 추가 루프 반복을 피하기 위해 이러한 함수를 일찍 종료(break)할 수 있습니다.


### `LibBytes::packAddressAndStem`이 동일한 매개변수로 두 번 계산됨

`LibSilo::_removeDepositsFromAccount`는 `LibTokenSilo::removeDepositFromAccount`가 이미 동일한 매개변수로 동일한 함수를 [호출](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibTokenSilo.sol#L239)한 후 `LibBytes::packAddressAndStem`을 [호출](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibSilo.sol#L597)합니다.

`LibSilo::_removeDepositsFromAccount`의 각 루프 반복에 대해 `LibBytes::packAddressAndStem`을 한 번 계산한 다음 결과를 `LibTokenSilo::removeDepositFromAccount` 호출의 매개변수로 전달하도록 리팩토링하는 것을 고려하십시오.


### `LibTokenSilo::stemTipForToken`이 동일한 매개변수로 여러 번 계산됨

`LibTokenSilo::stemTipForToken`은 `LibSilo::_removeDepositsFromAccount`에서 동일한 [매개변수](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/Silo/LibSilo.sol#L604)로 여러 번 계산됩니다. 이는 대량 인출 중에 동일한 `token` 매개변수로 `LibTokenSilo::stemTipForToken`이 항상 호출되어 변경되지 않는 스토리지에 대해 4번의 SLOAD 작업을 수행하므로 가스를 낭비합니다.

루프에 들어가기 전에 stem tip을 한 번 계산한 다음 결과를 `stalkReward()`에 매개변수로 전달하는 것을 고려하십시오.

동일한 문제가 `ConvertFacet::_withdrawTokens`에서도 발생합니다.


### `SiloFacet::transferDeposits`는 `LibSiloPermit::_spendDepositAllowance`를 한 번만 호출해야 함

`SiloFacet::transferDeposits`는 현재 입력 `amounts` 배열을 반복하고 각 `amounts[i]`에 대해 `LibSiloPermit::_spendDepositAllowance`를 한 번씩 [호출](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/silo/SiloFacet/SiloFacet.sol#L185)합니다.

대신 입력을 반복할 때 각 `amounts[i]`에 대해 증가하는 `totalAmount` 스택 변수를 갖는 것을 고려하십시오. 그런 다음 초기 루프가 완료된 후 `totalAmount`로 `LibSiloPermit::_spendDepositAllowance`를 호출하여 상당한 수의 스토리지 읽기 및 쓰기를 절약하십시오.


### 추가 스토리지 읽기를 방지하기 위해 업데이트된 잔여 금액 캐시

`FundraiserFacet::fund`는 계산된 [`remaining - amount`](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/field/FundraiserFacet.sol#L124)를 저장한 다음 [L125](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/field/FundraiserFacet.sol#L125)에서 스토리지를 설정하고 [L128](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/beanstalk/field/FundraiserFacet.sol#L128)에서 완료를 확인하는 데 사용해야 합니다. 이렇게 하면 L128에서 스토리지를 다시 읽는 것을 방지할 수 있습니다. 쉬운 해결책 중 하나는 기존 `remaining` 스택 변수를 재사용하는 것입니다.

```solidity
remaining = remaining - amount; // Note: SafeMath is redundant here.
s.fundraisers[id].remaining = remaining;
emit FundFundraiser(msg.sender, id, amount);

// If completed, transfer tokens to payee and emit an event
if (remaining == 0) {
    _completeFundraiser(id);
}
```


### 추가 스토리지 읽기를 방지하기 위해 자본 재구성된 금액 캐시

`LibFertilizer::remainingRecapitalization`은 `s.recapitalized`를 캐시한 다음 [L166-167](https://github.com/BeanstalkFarms/Beanstalk/blob/dfb418d185cd93eef08168ccaffe9de86bc1f062/protocol/contracts/libraries/LibFertilizer.sol#L166-L167)에서 캐시된 스택 변수를 사용하여 스토리지에서 동일한 값을 두 번 읽는 것을 방지해야 합니다.

\clearpage

