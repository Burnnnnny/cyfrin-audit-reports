**Lead Auditors**

[Dacian](https://x.com/DevDacian)

[Jorge](https://x.com/TamayoNft)

**Assisting Auditors**



---

# 발견 사항 (Findings)
## 중간 위험 (Medium Risk)


### 동일한 지갑을 투자자에게 여러 번 추가할 수 있어 지갑 수를 인위적으로 증가시키고 새 지갑 추가를 되돌리게(revert) 함

**설명:** `GlobalRegistryService::_updateInvestor`는 추가되는 지갑이 이미 이 투자자에게 등록된 경우 `_addWallet`을 호출합니다:
```solidity
for (uint8 i = 0; i < walletAddresses.length; i++) {
    // @audit if it is a wallet and it doesn't belong to this investor, revert
    if (isWallet(walletAddresses[i]) && !CommonUtils.isEqualString(getInvestor(walletAddresses[i]), id)) {
        revert WalletBelongsToAnotherInvestor();
    }
    // @audit otherwise add it - even if it is a wallet that already belongs to this investor!
    else {
        _addWallet(walletAddresses[i], id);
    }
}
```

`GlobalRegistryService::_addWallet`은 차례로 투자자의 `walletCount`를 증가시키고 최대치에 도달하면 되돌립니다:
```solidity
function _addWallet(address walletAddress, string memory id) internal addressNotZero(walletAddress) returns (bool) {
    if (investors[id].walletCount >= MAX_WALLETS_PER_INVESTOR) {
        revert MaxWalletsReached();
    }
    address sender = _msgSender();
    investorsWallets[walletAddress] = Wallet(id, sender);
    investors[id].walletCount++;

    emit GlobalWalletAdded(walletAddress, id, sender);

    return true;
}
```

**영향:** 투자자의 지갑 수는 특히 투자자의 모든 기존 데이터와 일부 수정된 필드로 호출될 수 있는 `GlobalRegistryService::updateInvestor`를 통해 세부 정보를 업데이트할 때 인위적으로 부풀려질 수 있습니다. `MAX_WALLETS_PER_INVESTOR`에 도달하면 더 이상의 업데이트가 불가능합니다.

또한 `GlobalRegistryService::removeWallet`이 투자자의 `walletCount`를 다시 0으로 줄일 수 없으므로 `removeInvestor`가 항상 되돌려지는 등 투자자를 제거할 수 없는 다른 영향도 있습니다.

**개념 증명 (Proof Of Concept):**
먼저 `GlobalRegistryService.sol`에 이 `view` 함수를 추가합니다:
```solidity
    function walletCountByInvestor(string calldata investorId) public view returns (uint256) {
        return investors[investorId].walletCount;
    }
```

그런 다음 `global-registry-service.tests.ts`에 PoC를 추가합니다:
```typescript
  it('Bug - adding same wallet for same investor inflates wallet count', async function () {
    const [, investor] = await hre.ethers.getSigners();
    const { globalRegistryService } = await loadFixture(deployGRS);
    await globalRegistryService.updateInvestor(
      INVESTORS.INVESTOR_ID.INVESTOR_ID_1,
      INVESTORS.INVESTOR_ID.INVESTOR_COLLISION_HASH_1,
      US,
      [investor],
      [1, 2, 4],
      [1, 1, 1],
      [0, 0, 0],
    );

    await globalRegistryService.updateInvestor(
      INVESTORS.INVESTOR_ID.INVESTOR_ID_1,
      INVESTORS.INVESTOR_ID.INVESTOR_COLLISION_HASH_1,
      US,
      [investor],
      [1, 2, 4],
      [1, 1, 1],
      [0, 0, 0],
    );

    expect(await globalRegistryService.walletCountByInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1)).to.equal(2);

  });
```

`npx hardhat test --grep "adding same wallet for same investor inflates wallet count"`로 실행합니다.

**권장 완화 조치:** `GlobalRegistryService::_updateInvestor`에서 추가되는 지갑이 지갑이고 이미 동일한 투자자에게 속해 있다면 아무것도 하지 마십시오. 다음은 `isWallet` 및 `isInvestor` 함수의 중복 작업을 피하면서 이 버그도 수정하는 `_updateInvestor`의 보다 효율적인 구현입니다:
```solidity
function _updateInvestor(string calldata id, address[] memory walletAddresses) internal returns (bool) {
    // revert if max wallet would be breached
    uint256 walletAddressesLen = walletAddresses.length;
    if (walletAddressesLen > MAX_WALLETS_PER_INVESTOR) {
        revert TooManyWallets();
    }

    // register investor if they don't already exist
    if (!isInvestor(id)) {
        _registerInvestor(id);
    }

    for (uint8 i; i < walletAddressesLen; i++) {
        address newWallet = walletAddresses[i];

        // is the wallet already registered to an investor?
        string memory walletExistingInvestor = getInvestor(newWallet);

        // if not then add it
        if(!isInvestor(walletExistingInvestor)) {
            _addWallet(newWallet, id);
        }
        // otherwise revert if it is registered to another investor
        else if(!CommonUtils.isEqualString(walletExistingInvestor, id)) {
            revert WalletBelongsToAnotherInvestor();
        }
        // if it is already registered to this investor, do nothing
    }

    return true;
}
```

**Securitize:** 커밋 [5713fd2](https://github.com/securitize-io/bc-global-registry-service-sc/commit/5713fd25851f6a437b45d947f5d4652f2450fb10)에서 수정됨.

**Cyfrin:** 확인함.


### `ComplianceServiceGlobalWhitelisted::newPreTransferCheck` 및 `preTransferCheck`가 블랙리스트에 오른 사용자의 토큰 전송을 허용함

**설명:** `ComplianceServiceGlobalWhitelisted::newPreTransferCheck` 및 `preTransferCheck`는 토큰 수신자가 블랙리스트에 없는지만 확인하고, 발신자 또한 블랙리스트에 없는지는 확인하지 않습니다:
```solidity
function newPreTransferCheck(
    address _from,
    address _to,
    uint256 _value,
    uint256 _balanceFrom,
    bool _pausedToken
) public view virtual override returns (uint256 code, string memory reason) {
    // First check if recipient is blacklisted
    if (getBlackListManager().isBlacklisted(_to)) {
        return (100, WALLET_BLACKLISTED);
    }

    // Then perform the standard whitelist check
    return super.newPreTransferCheck(_from, _to, _value, _balanceFrom, _pausedToken);
}

function preTransferCheck(address _from, address _to, uint256 _value) public view virtual override returns (uint256 code, string memory reason) {
    // First check if recipient is blacklisted
    if (getBlackListManager().isBlacklisted(_to)) {
        return (100, WALLET_BLACKLISTED);
    }

    // Then perform the standard whitelist check
    return super.preTransferCheck(_from, _to, _value);
}
```

**영향:** 블랙리스트에 오른 사용자가 블랙리스트에 없는 사용자에게 토큰을 전송하여 사실상 블랙리스트를 회피할 수 있습니다.

**권장 완화 조치:** `ComplianceServiceGlobalWhitelisted::newPreTransferCheck` 및 `preTransferCheck`는 `from` 주소가 블랙리스트에 있는 경우 올바른 오류 코드를 반환해야 합니다.

**Securitize:** 커밋 [32d1a02](https://github.com/securitize-io/dstoken/commit/32d1a020f4fad010f656da2a0da739b06d338e65), [a616d39](https://github.com/securitize-io/dstoken/commit/a616d398add96a08e53942a11ba26cfc505a8ef3)에서 수정됨.

**Cyfrin:** 확인함.


### 토큰 이름 업데이트가 허가(permit) 기능을 위한 EIP-712 도메인 구분자를 깨뜨림

**설명:** EIP-712 도메인 구분자는 `__ERC20PermitMixin_init()`에서 초기 토큰 이름으로 컨트랙트 배포 중 한 번 초기화됩니다:
```solidity
function __ERC20PermitMixin_init(string memory name_) internal onlyInitializing {
    __EIP712_init(name_, "1");  // Domain separator set with initial name
    __Nonces_init();
}
```

그러나 `StandardToken`은 Master 역할이 `updateNameAndSymbol`을 통해 토큰 이름을 업데이트할 수 있도록 허용합니다:
```solidity
function updateNameAndSymbol(string calldata _name, string calldata _symbol) external onlyMaster {
    // ...
    name = _name;  // Name updated but EIP-712 domain separator NOT updated
    // ...
}
```

**영향:** 토큰 이름이 변경되면 EIP-712 도메인 구분자는 변경되지 않은 상태로 유지됩니다. 이는 지갑이 허가 서명을 생성하는 데 사용하는 것(현재 토큰 이름)과 컨트랙트가 이를 검증하는 데 사용하는 것(원래 배포 이름) 사이에 불일치를 만듭니다. 잠재적 영향:

1. **완전한 허가 기능 파손**: 이름 변경 후 새로 생성된 허가 서명의 100%가 "Permit: invalid signature"로 검증 실패
2. **조용한 실패 모드**: 사용자 및 통합은 이 불일치를 감지할 프로그래밍 방식이 없음; 오류 메시지는 도메인 구분자 문제임을 나타내지 않음
3. **모든 동적 통합 중단**: 허가 서명을 생성하기 위해 `token.name()`을 가져오는 모든 dApp은 이름 업데이트 후 자동으로 중단됨
4. **일관성 없는 동작**: 이름 변경 전에 서명된 허가는 계속 작동하는 반면, 새로운 허가는 실패하여 혼란스러운 분할 상태 생성
5. **쉬운 복구 경로 없음**: 이를 수정하려면 컨트랙트 업그레이드가 필요하거나 모든 사용자/통합에게 더 이상 사용되지 않는 이름을 사용하도록 지시해야 함(EIP-2612 예상 위반)

**개념 증명 (Proof of Concept):** 새 파일 `test/change.name.permit.test.ts` 추가:
```typescript
import { expect } from 'chai';
import { loadFixture } from '@nomicfoundation/hardhat-toolbox/network-helpers';
import hre from 'hardhat';
import {
  deployDSTokenRegulated,
  INVESTORS,
} from './utils/fixture';
import { buildPermitSignature, registerInvestor } from './utils/test-helper';

describe('M-01: Token Name Update Breaks Permit Functionality - Proof of Concept', function() {

  describe('Demonstrating the Vulnerability', function() {

    it('CRITICAL: Permit fails after name update - All new permits become invalid', async function() {
      const [owner, spender] = await hre.ethers.getSigners();
      const { dsToken } = await loadFixture(deployDSTokenRegulated);

      // Initial state: Token name is "Token Example 1"
      const originalName = await dsToken.name();
      expect(originalName).to.equal('Token Example 1');

      // ✅ STEP 1: Permit works BEFORE name change
      console.log('\n--- BEFORE NAME CHANGE ---');
      const deadline1 = BigInt(Math.floor(Date.now() / 1000) + 3600);
      const value = 100;
      const message1 = {
        owner: owner.address,
        spender: spender.address,
        value,
        nonce: await dsToken.nonces(owner.address),
        deadline: deadline1,
      };

      // User signs with original name "Token Example 1"
      const sig1 = await buildPermitSignature(
        owner,
        message1,
        originalName,  // Uses "Token Example 1"
        await dsToken.getAddress()
      );

      // Permit succeeds with original name
      await dsToken.permit(owner.address, spender.address, value, deadline1, sig1.v, sig1.r, sig1.s);
      console.log('✅ Permit with original name: SUCCESS');
      expect(await dsToken.allowance(owner.address, spender.address)).to.equal(value);

      // ⚠️ STEP 2: Master updates token name
      console.log('\n--- NAME CHANGE ---');
      const newName = 'Token Example 2 - Updated';
      await dsToken.updateNameAndSymbol(newName, 'TX2');
      expect(await dsToken.name()).to.equal(newName);
      console.log(`Token name updated: "${originalName}" → "${newName}"`);

      // ❌ STEP 3: Permit FAILS after name change
      console.log('\n--- AFTER NAME CHANGE ---');
      const deadline2 = BigInt(Math.floor(Date.now() / 1000) + 3600);
      const message2 = {
        owner: owner.address,
        spender: spender.address,
        value: 200,
        nonce: await dsToken.nonces(owner.address),
        deadline: deadline2,
      };

      // User's wallet fetches current name and generates signature
      const currentName = await dsToken.name(); // Returns "Token Example 2 - Updated"
      console.log(`User's wallet uses current name: "${currentName}"`);

      const sig2 = await buildPermitSignature(
        owner,
        message2,
        currentName,  // Uses NEW name "Token Example 2 - Updated"
        await dsToken.getAddress()
      );

      // 🚨 PERMIT FAILS - Domain separator mismatch!
      await expect(
        dsToken.permit(owner.address, spender.address, 200, deadline2, sig2.v, sig2.r, sig2.s)
      ).to.be.revertedWith('Permit: invalid signature');

      console.log('❌ Permit with new name: FAILED - "Permit: invalid signature"');
      console.log('\n🚨 VULNERABILITY CONFIRMED: All new permits fail after name change!');
    });

    it('IMPACT: Old permits continue working while new ones fail - Inconsistent behavior', async function() {
      const [owner, spender] = await hre.ethers.getSigners();
      const { dsToken } = await loadFixture(deployDSTokenRegulated);

      const originalName = await dsToken.name();

      // Generate permit signature BEFORE name change
      const deadline1 = BigInt(Math.floor(Date.now() / 1000) + 3600);
      const message1 = {
        owner: owner.address,
        spender: spender.address,
        value: 100,
        nonce: await dsToken.nonces(owner.address),
        deadline: deadline1,
      };

      const oldSignature = await buildPermitSignature(
        owner,
        message1,
        originalName,
        await dsToken.getAddress()
      );

      // Master changes the name
      await dsToken.updateNameAndSymbol('Token Example 2', 'TX2');
      const newName = await dsToken.name();

      // ✅ OLD permit (signed before name change) still works!
      await dsToken.permit(
        owner.address,
        spender.address,
        100,
        deadline1,
        oldSignature.v,
        oldSignature.r,
        oldSignature.s
      );
      console.log('✅ Old permit (signed before name change): SUCCESS');

      // ❌ NEW permit (signed after name change) fails!
      const deadline2 = BigInt(Math.floor(Date.now() / 1000) + 3600);
      const message2 = {
        owner: owner.address,
        spender: spender.address,
        value: 200,
        nonce: await dsToken.nonces(owner.address),
        deadline: deadline2,
      };

      const newSignature = await buildPermitSignature(
        owner,
        message2,
        newName,  // Uses new name
        await dsToken.getAddress()
      );

      await expect(
        dsToken.permit(owner.address, spender.address, 200, deadline2, newSignature.v, newSignature.r, newSignature.s)
      ).to.be.revertedWith('Permit: invalid signature');

      console.log('❌ New permit (signed after name change): FAILED');
      console.log('\n🚨 INCONSISTENT STATE: Split behavior based on signature timing!');
    });

    it('IMPACT: DApp integrations break silently', async function() {
      const [owner, spender] = await hre.ethers.getSigners();
      const { dsToken } = await loadFixture(deployDSTokenRegulated);

      // Simulate a DApp that dynamically fetches token name
      async function dAppGeneratePermitSignature(tokenContract, ownerSigner, spenderAddress, value, deadline) {
        // Standard DApp implementation: fetch name dynamically
        const tokenName = await tokenContract.name();
        const tokenAddress = await tokenContract.getAddress();

        const message = {
          owner: ownerSigner.address,
          spender: spenderAddress,
          value,
          nonce: await tokenContract.nonces(ownerSigner.address),
          deadline,
        };

        return await buildPermitSignature(ownerSigner, message, tokenName, tokenAddress);
      }

      // ✅ DApp works fine initially
      const deadline1 = BigInt(Math.floor(Date.now() / 1000) + 3600);
      const sig1 = await dAppGeneratePermitSignature(dsToken, owner, spender.address, 100, deadline1);

      await dsToken.permit(owner.address, spender.address, 100, deadline1, sig1.v, sig1.r, sig1.s);
      console.log('✅ DApp integration BEFORE name change: SUCCESS');

      // Master updates name
      await dsToken.updateNameAndSymbol('Token Example 2', 'TX2');
      console.log('\n⚠️  Token name updated to "Token Example 2"');

      // ❌ DApp breaks - it fetches the NEW name but contract validates against OLD name
      const deadline2 = BigInt(Math.floor(Date.now() / 1000) + 3600);
      const sig2 = await dAppGeneratePermitSignature(dsToken, owner, spender.address, 200, deadline2);

      await expect(
        dsToken.permit(owner.address, spender.address, 200, deadline2, sig2.v, sig2.r, sig2.s)
      ).to.be.revertedWith('Permit: invalid signature');

      console.log('❌ DApp integration AFTER name change: FAILED');
      console.log('🚨 DApp has NO way to detect this issue programmatically!');
    });

    it('WORKAROUND: Permit succeeds if user manually uses ORIGINAL name (terrible UX)', async function() {
      const [owner, spender] = await hre.ethers.getSigners();
      const { dsToken } = await loadFixture(deployDSTokenRegulated);

      const originalName = await dsToken.name(); // "Token Example 1"

      // Master updates name
      await dsToken.updateNameAndSymbol('Token Example 2', 'TX2');

      // ❌ Using current name fails
      const deadline1 = BigInt(Math.floor(Date.now() / 1000) + 3600);
      const message1 = {
        owner: owner.address,
        spender: spender.address,
        value: 100,
        nonce: await dsToken.nonces(owner.address),
        deadline: deadline1,
      };

      const sigWithNewName = await buildPermitSignature(
        owner,
        message1,
        await dsToken.name(),  // "Token Example 2" (current)
        await dsToken.getAddress()
      );

      await expect(
        dsToken.permit(owner.address, spender.address, 100, deadline1, sigWithNewName.v, sigWithNewName.r, sigWithNewName.s)
      ).to.be.revertedWith('Permit: invalid signature');
      console.log('❌ Permit with current name "Token Example 2": FAILED');

      // ✅ Using ORIGINAL name works (but terrible UX)
      const deadline2 = BigInt(Math.floor(Date.now() / 1000) + 3600);
      const message2 = {
        owner: owner.address,
        spender: spender.address,
        value: 200,
        nonce: await dsToken.nonces(owner.address),
        deadline: deadline2,
      };

      const sigWithOriginalName = await buildPermitSignature(
        owner,
        message2,
        originalName,  // "Token Example 1" (original)
        await dsToken.getAddress()
      );

      await dsToken.permit(
        owner.address,
        spender.address,
        200,
        deadline2,
        sigWithOriginalName.v,
        sigWithOriginalName.r,
        sigWithOriginalName.s
      );
      console.log('✅ Permit with ORIGINAL name "Token Example 1": SUCCESS');
      console.log('\n🚨 WORKAROUND: Users must use deprecated name - breaks EIP-2612 expectations!');
    });

    it('VERIFICATION: DOMAIN_SEPARATOR remains unchanged after name update', async function() {
      const { dsToken } = await loadFixture(deployDSTokenRegulated);

      const originalName = 'Token Example 1';

      // Get domain separator before name change
      const domainSeparatorBefore = await dsToken.DOMAIN_SEPARATOR();
      console.log('Domain separator BEFORE name change:', domainSeparatorBefore);

      // Compute expected domain separator with original name
      const expectedDomainBefore = hre.ethers.TypedDataEncoder.hashDomain({
        version: '1',
        name: originalName,
        verifyingContract: await dsToken.getAddress(),
        chainId: (await hre.ethers.provider.getNetwork()).chainId,
      });

      expect(domainSeparatorBefore).to.equal(expectedDomainBefore);

      // Update name
      await dsToken.updateNameAndSymbol('Token Example 2', 'TX2');
      const newName = await dsToken.name();
      console.log(`\nName updated: "${originalName}" → "${newName}"`);

      // Get domain separator after name change
      const domainSeparatorAfter = await dsToken.DOMAIN_SEPARATOR();
      console.log('Domain separator AFTER name change:', domainSeparatorAfter);

      // 🚨 DOMAIN SEPARATOR UNCHANGED!
      expect(domainSeparatorAfter).to.equal(domainSeparatorBefore);
      console.log('\n🚨 VERIFIED: Domain separator did NOT update with new name!');

      // Compute what the domain separator SHOULD be with new name
      const expectedDomainWithNewName = hre.ethers.TypedDataEncoder.hashDomain({
        version: '1',
        name: newName,
        verifyingContract: await dsToken.getAddress(),
        chainId: (await hre.ethers.provider.getNetwork()).chainId,
      });

      console.log('\nExpected domain with NEW name:', expectedDomainWithNewName);
      console.log('Actual domain separator:      ', domainSeparatorAfter);
      console.log('Match:', domainSeparatorAfter === expectedDomainWithNewName ? '✅' : '❌');

      // They don't match - this is the root cause
      expect(domainSeparatorAfter).to.not.equal(expectedDomainWithNewName);
    });
  });

  describe('Real-World Attack Scenarios', function() {

    it('SCENARIO 1: Protocol rebranding breaks all user permits', async function() {
      const [owner, user1, user2, dex] = await hre.ethers.getSigners();
      const { dsToken, registryService } = await loadFixture(deployDSTokenRegulated);

      // Setup investors
      await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, user1, registryService);
      await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, user2, registryService);
      await dsToken.issueTokens(user1, 1000);

      console.log('\n📊 SCENARIO: Token rebranding from "Token Example 1" to "Acme Securities Token"');

      // Before rebrand: User1 can use permit to approve DEX
      const deadline1 = BigInt(Math.floor(Date.now() / 1000) + 3600);
      const message1 = {
        owner: user1.address,
        spender: dex.address,
        value: 500,
        nonce: await dsToken.nonces(user1.address),
        deadline: deadline1,
      };

      const sig1 = await buildPermitSignature(
        user1,
        message1,
        await dsToken.name(),
        await dsToken.getAddress()
      );

      await dsToken.permit(user1.address, dex.address, 500, deadline1, sig1.v, sig1.r, sig1.s);
      console.log('✅ User1 successfully approved DEX using permit (before rebrand)');

      // 🏢 PROTOCOL REBRANDS
      await dsToken.updateNameAndSymbol('Acme Securities Token', 'AST');
      console.log('\n🏢 Protocol rebrands to "Acme Securities Token"');

      // After rebrand: All new permits fail
      const deadline2 = BigInt(Math.floor(Date.now() / 1000) + 3600);
      const message2 = {
        owner: user2.address,
        spender: dex.address,
        value: 500,
        nonce: await dsToken.nonces(user2.address),
        deadline: deadline2,
      };

      await dsToken.issueTokens(user2, 1000);

      const sig2 = await buildPermitSignature(
        user2,
        message2,
        await dsToken.name(), // Uses new name
        await dsToken.getAddress()
      );

      await expect(
        dsToken.permit(user2.address, dex.address, 500, deadline2, sig2.v, sig2.r, sig2.s)
      ).to.be.revertedWith('Permit: invalid signature');

      console.log('❌ User2 permit FAILS after rebrand');
      console.log('🚨 Impact: 100% of new users cannot use gasless approvals!');
      console.log('📞 Result: Support tickets flood in, users confused');
    });

    it('SCENARIO 2: Front-end integration breaks without warning', async function() {
      const [owner, user, spender] = await hre.ethers.getSigners();
      const { dsToken, registryService } = await loadFixture(deployDSTokenRegulated);

      await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, user, registryService);
      await dsToken.issueTokens(user, 1000);

      console.log('\n🌐 SCENARIO: Frontend dApp integration');

      // Simulate frontend code
      const frontendPermitFlow = async (token, fromUser, toSpender, amount) => {
        // Standard EIP-2612 implementation in frontend
        const name = await token.name(); // Fetch current name dynamically
        const deadline = BigInt(Math.floor(Date.now() / 1000) + 3600);
        const nonce = await token.nonces(fromUser.address);

        const message = {
          owner: fromUser.address,
          spender: toSpender,
          value: amount,
          nonce,
          deadline,
        };

        const signature = await buildPermitSignature(
          fromUser,
          message,
          name,
          await token.getAddress()
        );

        return { deadline, signature };
      };

      // ✅ Frontend works initially
      const { deadline: d1, signature: s1 } = await frontendPermitFlow(dsToken, user, spender.address, 100);
      await dsToken.permit(user.address, spender.address, 100, d1, s1.v, s1.r, s1.s);
      console.log('✅ Frontend permit flow: SUCCESS (initial deployment)');

      // Master updates name (e.g., for compliance reasons)
      await dsToken.updateNameAndSymbol('Compliant Token v2', 'CTv2');
      console.log('\n⚠️  Master updates name for compliance');

      // ❌ Frontend breaks silently
      const { deadline: d2, signature: s2 } = await frontendPermitFlow(dsToken, user, spender.address, 200);
      await expect(
        dsToken.permit(user.address, spender.address, 200, d2, s2.v, s2.r, s2.s)
      ).to.be.revertedWith('Permit: invalid signature');

      console.log('❌ Frontend permit flow: BROKEN (after name update)');
      console.log('🚨 Error message gives NO hint about name mismatch');
      console.log('😰 Users see "Invalid signature" and blame wallet/frontend');
    });
  });
});
```

`npx hardhat test --grep "Token Name Update Breaks Permit Functionality"`로 실행합니다.

**권장 완화 조치:** 가장 우아한 해결책은 다음과 같습니다:
* `StandardToken`이 `name`을 반환하는 `_name` 함수를 정의합니다:
```solidity
    function _name() internal view virtual override returns (string memory) {
        return name;  // Returns current storage variable
    }
```

* `ERC20PermitMixin`이 `EIP712Upgradeable::_EIP712Name`을 재정의하여 이 함수를 호출합니다:
```solidity
    // ✨ Override to return dynamic name instead of cached name
    function _EIP712Name() internal view virtual override returns (string memory) {
        return _name();  // Calls abstract function implemented by StandardToken
    }

    // Abstract function for StandardToken to implement
    function _name() internal view virtual returns (string memory);
```

**Securitize:** 커밋 [4ebb9b7](https://github.com/securitize-io/dstoken/commit/4ebb9b706e7570ba0f0e295205c79949c16f1b0c)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 낮은 위험 (Low Risk)


### `ERC20PermitMixin::permit`을 직접 호출하는 프론트러닝으로 `StandardToken::transferWithPermit`을 서비스 거부(DoS) 공격할 수 있음

**설명:** `StandardToken::transferWithPermit`은 두 가지 호출을 포함합니다:
* 첫 번째는 `ERC20PermitMixin::permit`에 대한 호출
* 두 번째는 `StandardToken::transferFrom`에 대한 호출

**영향:** 허가 서명과 매개변수가 실행 전에 멤풀에 표시되므로 공격자는 이러한 값을 추출하고 `StandardToken::permit`을 직접 호출하여 트랜잭션을 프론트러닝할 수 있습니다. 이것은 사용자의 nonce를 소비하여 원래 호출인 `StandardToken::transferWithPermit`을 되돌리게(revert) 만들어 승인을 부여하고 토큰을 전송하는 것을 원자적으로 불가능하게 만듭니다.

**개념 증명 (Proof of concept)**
`dstoken-regulated.test.ts`의 `describe('Permit transfer', async function () {` 내부에서 PoC를 실행합니다:
```typescript
it('front-running attack on transferWithPermit()', async () => {
        const [owner, spender, recipient, attacker] = await hre.ethers.getSigners();
        const { dsToken, registryService } = await loadFixture(deployDSTokenRegulated);
        const value = 100;
        const deadline = BigInt(Math.floor(Date.now() / 1000) + 3600);

        // Owner creates a signature to allow spender to transfer tokens to recipient
        const message = {
          owner: owner.address,
          spender: spender.address,
          value,
          nonce: await dsToken.nonces(owner.address),
          deadline,
        };
        const { v, r, s } = await buildPermitSignature(owner, message, await dsToken.name(), await dsToken.getAddress());

        // Register investors and issue tokens to owner; see that the attacker is not even an ibnvestor so it could be any address
        await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, owner, registryService);
        await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, recipient, registryService);

        await dsToken.issueTokens(owner, value);

        // ATTACK SCENARIO 1: Attacker front-runs by calling permit() directly
        await dsToken.connect(attacker).permit(owner.address, spender.address, value, deadline, v, r, s);


        // When the original transferWithPermit() executes, it FAILS
        // because the nonce has already been used
        await expect(
          dsToken.connect(spender).transferWithPermit(owner.address, recipient.address, value, deadline, v, r, s)
        ).to.be.revertedWith('Permit: invalid signature');
      });
```

**권장 완화 조치:** try-catch 패턴을 사용하십시오:
```solidity
function transferWithPermit(
    address from,
    address to,
    uint256 value,
    uint256 deadline,
    uint8 v,
    bytes32 r,
    bytes32 s
) external returns (bool) {
    // Try to execute permit, but don't revert if it fails
    try this.permit(from, msg.sender, value, deadline, v, r, s) {
        // Permit succeeded
    } catch {
        // Permit failed (possibly due to front-running or already executed)
        // Verify we have sufficient allowance to proceed
        require(allowance(from, msg.sender) >= value, "Insufficient allowance");
    }

    // Perform the actual transferFrom
    return transferFrom(from, to, value);
}
```

**Securitize:** 커밋 [d7cf385](https://github.com/securitize-io/dstoken/commit/d7cf3858c371def66e5b37ed0949aa991d0a0234)에서 수정됨.

**Cyfrin:** 확인함.


### `block.number` 기반 타임아웃 메커니즘과의 크로스체인 비호환성

**설명:** `GlobalRegistryService::addGlobalInvestorWallet` 함수는 `blockLimit parameter`를 통해 트랜잭션 최신성을 검증하기 위해 `block.number`를 사용합니다:
```solidity
function addGlobalInvestorWallet(
    string calldata id,
    address walletAddress,
    uint256 blockLimit
) external override whenNotPaused onlySelf newWallet(walletAddress) returns (bool) {
    if (blockLimit < block.number) {
        revert TransactionTooOld();
    }
    // ...
}
```

hardhat 구성에 따르면 이 컨트랙트는 블록 생성 속도가 매우 다른 여러 체인에 배포되도록 설계되었습니다:
* Ethereum Mainnet - 블록 시간: ~12-14초
* Sepolia (Ethereum testnet) - 블록 시간: ~12-14초
* Arbitrum (chainId: 421614) - 블록 시간: ~0.25초 (48-56배 빠름)
* Optimism (chainId: 11155420) - 블록 시간: ~2초 (6-7배 빠름)
* Avalanche/Fuji (chainId: 43113) - 블록 시간: ~2초 (6-7배 빠름)

문제는 블록 생성 속도가 이러한 체인 전반에 걸쳐 극적으로 다양하여 블록 기반 시간 검증을 일관성 없고 신뢰할 수 없게 만든다는 것입니다. 운영자가 `blockLimit = currentBlock + 100`으로 사전 승인된 트랜잭션에 서명하는 경우:
* Ethereum에서: ~100 × 13초 = ~21분 동안 유효
* Arbitrum에서: ~100 × 0.25초 = ~25초 동안 유효
* Optimism/Avalanche에서: ~100 × 2초 = ~3.3분 동안 유효

**영향:** 투자자가 다른 체인 간에 동일한 지갑을 추가하는 경우 한 체인에서는 트랜잭션이 되돌려지고 다른 체인에서는 추가될 수 있습니다.

**권장 완화 조치:** 일관된 크로스체인 동작을 위해 `block.number`를 `block.timestamp`로 대체하십시오.

**Securitize:** 커밋 [8f92757](https://github.com/securitize-io/bc-global-registry-service-sc/commit/8f927571c7526817ffe43c5f37d11560e79809d9), [e99c56f](https://github.com/securitize-io/bc-global-registry-service-sc/commit/e99c56fe94f9b41e0680d2318f504fca33be4919), [920e496](https://github.com/securitize-io/bc-global-registry-service-sc/commit/920e4965bb9306203a8251e58c962f4dfff67a3f)에서 수정됨.

**Cyfrin:** 확인함.


### `GlobalRegistryService::executePreApprovedTransaction`에 대한 서명 기한 누락

**설명:** `GlobalRegistryService::executePreApprovedTransaction`은 `Operator`가 임의의 컨트랙트와 함수를 호출할 수 있도록 허용하지만 현재 의도는 `addGlobalInvestorWallet`을 호출하는 것입니다.

**영향:** `addGlobalInvestorWallet`은 `block.number`를 사용하여 기한을 구현하지만(타임스탬프 사용에 대한 다른 문제 참조), `executePreApprovedTransaction`이 다른 함수를 호출하는 데 사용되는 경우 기한 확인이 구현되지 않거나 다른 많은 곳에서 기한 확인을 복제해야 할 수 있습니다.

**권장 완화 조치:** `GlobalRegistryService::executePreApprovedTransaction`에서 타임스탬프 기반 기한 확인을 구현하십시오. 또한 nonce가 무효화될 수 있도록 관리자나 운영자가 `noncePerInvestor[txData.senderInvestor]`를 늘릴 수 있는 방법을 추가하는 것을 고려하십시오.

**Securitize:** 커밋 [8f92757](https://github.com/securitize-io/bc-global-registry-service-sc/commit/8f927571c7526817ffe43c5f37d11560e79809d9), [e99c56f](https://github.com/securitize-io/bc-global-registry-service-sc/commit/e99c56fe94f9b41e0680d2318f504fca33be4919), [920e496](https://github.com/securitize-io/bc-global-registry-service-sc/commit/920e4965bb9306203a8251e58c962f4dfff67a3f)에서 수정됨

**Cyfrin:** 확인함.

\clearpage
## 정보 (Informational)


### 역할이 실제로 부여되거나 취소되지 않았을 때 오해의 소지가 있는 이벤트 발생시키지 않기

**설명:** `AccessControlUpgradeable::_grantRole` 및 `_revokeRole`은 역할이 실제로 부여되었는지 또는 취소되었는지를 나타내는 `bool`을 [반환](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/access/AccessControlUpgradeable.sol#L204-L231)합니다.

**영향:** 이를 사용하는 일부 함수는 `bool` 반환을 확인하지 않고 이벤트를 발생시킵니다. 역할이 실제로 부여되거나 취소되지 않은 경우 이러한 이벤트는 오해의 소지가 있습니다.

**권장 완화 조치:** 영향을 받는 함수는 다음과 같습니다:
* `GlobalRegistryService::changeAdmin, addOperator, revokeOperator`

이러한 함수에서 `_grantRole` 및 `_revokeRole`의 반환을 확인하고 역할이 실제로 부여되거나 취소된 경우에만 이벤트를 발생시키십시오.

**Securitize:** 커밋 [c7d50ac](https://github.com/securitize-io/bc-global-registry-service-sc/commit/c7d50acbe661aae0edd62e54c91692d3ff3a35b9)에서 수정됨.

**Cyfrin:** 확인함.


### 기본값으로 초기화하지 않기

**설명:** Solidity에서는 기본값으로 초기화하지 마십시오:
```solidity
registry/GlobalRegistryService.sol
169:        for (uint8 i = 0; i < walletAddresses.length; i++) {
```

**Securitize:** 커밋 [5713fd2](https://github.com/securitize-io/bc-global-registry-service-sc/commit/5713fd25851f6a437b45d947f5d4652f2450fb10)에서 수정됨.

**Cyfrin:** 확인함.


### `GlobalRegistryService::executePreApprovedTransaction`은 스마트 지갑 운영자와 호환되지 않음

**설명:** `GlobalRegistryService::executePreApprovedTransaction`은 운영자가 서명을 사용하여 사전 승인된 트랜잭션을 실행할 수 있도록 허용합니다. 그러나 항상 `ECDSA::recover`를 호출하므로 스마트 지갑인 운영자에게는 작동하지 않습니다.

이것이 의도인 것으로 보이지만 스마트 지갑 지원이 필요한 경우 [SignatureChecker](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/SignatureChecker.sol) 라이브러리 사용을 고려하십시오.

**Securitize:** 인지함; 우리는 서명자(항상 "Operator"임)가 스마트 지갑을 사용하지 않는다는 것을 알고 있습니다.


### 더 빠른 오프체인 매개변수 조회를 위해 이벤트 인덱싱 사용

**설명:** `IGlobalRegistryService`의 이벤트는 오프체인에서 매개변수별로 더 빠르게 조회할 수 있도록 이벤트당 가장 중요한 3개의 매개변수에 `indexed` 키워드를 사용해야 합니다.

**Securitize:** 커밋 [0c2321a](https://github.com/securitize-io/bc-global-registry-service-sc/commit/0c2321ab92e5bd47a602d55e765f18d8b7e7fbdf)에서 수정됨.

**Cyfrin:** 확인함.


### 상속된 `AccessControlUpgradeable` 함수가 일부 검증 확인을 우회함

**설명:** `GlobalRegistryService` 컨트랙트는 OpenZeppelin의 `AccessControlUpgradeable` 위에 추가 안전 검사 및 이벤트 발생이 있는 사용자 지정 래퍼 함수(`addOperator()`, `revokeOperator()`, `changeAdmin()`)를 구현합니다. 그러나 컨트랙트는 `AccessControlUpgradeable`의 기본 공용 함수를 재정의하지 않아 관리자와 운영자가 일부 검증 확인을 우회할 수 있습니다.

컨트랙트는 검증이 포함된 래퍼 함수를 생성합니다:
```solidity
function addOperator(address operator)
    external virtual override
    onlyRole(DEFAULT_ADMIN_ROLE)
    addressNotZero(operator)  // Safety check
{
    _grantRole(OPERATOR_ROLE, operator);
    emit OperatorAdded(operator);
}

function changeAdmin(address newAdmin)
    external virtual override
    onlyRole(DEFAULT_ADMIN_ROLE)
    addressNotZero(newAdmin)  // Safety check
{
    _grantRole(DEFAULT_ADMIN_ROLE, newAdmin);
    _revokeRole(DEFAULT_ADMIN_ROLE, _msgSender());
    emit AdminChanged(newAdmin);
}
```

그러나 기본 OpenZeppelin 함수는 재정의 없이 공개적으로 액세스할 수 있습니다:
```solidity
function grantRole(bytes32 role, address account) public virtual onlyRole(getRoleAdmin(role)) {
    _grantRole(role, account);
}

function revokeRole(bytes32 role, address account) public virtual onlyRole(getRoleAdmin(role)) {
    _revokeRole(role, account);
}

function renounceRole(bytes32 role, address callerConfirmation) public virtual {
    if (callerConfirmation != _msgSender()) {
        revert AccessControlBadConfirmation();
    }
    _revokeRole(role, callerConfirmation);
}
```

**영향:** 두 가지 발생 가능성이 낮은 문제가 발생할 수 있습니다:

* 관리자는 후임자를 지정하지 않고 자신의 역할을 포기하여 컨트랙트를 영구적으로 벽돌(brick)로 만들 수 있습니다(이는 `AccessControlUpgradeable::renounceRole`을 통해 수행될 수 있음)
* `AccessControlUpgradeable::grantRole`을 통해 직접 추가하는 운영자는 `OperatorAdded` 이벤트를 발생시키지 않습니다.

**권장 완화 조치:** 상속된 `AccessControlUpgradeable` 컨트랙트에서 사용되지 않을 기본 함수를 되돌리도록(revert) 재정의하는 것을 고려하십시오.

**Securitize:** **Cyfrin:**


### `ComplianceServiceGlobalWhitelisted::getComplianceTransferableTokens`가 블랙리스트에 오른 사용자에게 양수 토큰 금액을 반환함

**설명:** `ComplianceServiceWhitelisted`에서 상속된 `ComplianceServiceGlobalWhitelisted::getComplianceTransferableTokens`는 사용자가 블랙리스트에 있더라도 양수의 양도 가능한 토큰을 반환합니다. 이는 사용자가 블랙리스트에 있어 실제로 양도 가능한 토큰이 0개이기 때문에 오해의 소지가 있습니다.

```solidity
  function getComplianceTransferableTokens(
        address _who,
        uint256 _time,
        uint64 /*_lockTime*/
    ) public view virtual override returns (uint256) {
        require(_time > 0, "Time must be greater than zero");
        return getLockManager().getTransferableTokens(_who, _time);
    }
```

**권장 완화 조치:** 함수는 블랙리스트에 오른 주소에 대해 0을 반환해야 합니다.

**Securitize:** 커밋 [dc11a37](https://github.com/securitize-io/dstoken/commit/dc11a37ca955ecb0ee03baedcf5f580e7085b1bd)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### `EnumerableSet::contains`에 대한 중복 호출 제거

**설명:** `EnumerableSet::_add` 및 `_remove`는 이미 `_contains`를 [호출](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/EnumerableSet.sol#L75)하므로(또는 유사한 로직 수행), 다음에서 요소를 추가하거나 제거하기 전에 이를 호출할 필요가 없습니다:
* `BlackListManager::_addToBlacklist, _removeFromBlacklist`

**권장 완화 조치:** `EnumerableSet::add` 또는 `remove`를 직접 호출하고 `false`를 반환하면 되돌리십시오.

**Securitize:** 커밋 [a878a41](https://github.com/securitize-io/dstoken/commit/a878a41be5769dc3282c8646f6d145946be7d5ff)에서 수정됨.

**Cyfrin:** 확인함.


### `StandardToken::updateNameAndSymbol`에서 기존 이름과 심볼을 캐시한 후 자식 함수에 전달

**설명:** `StandardToken::updateNameAndSymbol`:
* 기존 `name` 및 `symbol`을 읽습니다
* `CommonUtils.isEqualString`을 호출하여 기존 값과 제안된 새 값을 비교합니다
* 다른 경우 이벤트를 발생시키기 위해 스토리지에서 동일한 기존 값을 다시 읽는 `_updateName` 및 `_updateSymbol`을 호출합니다

**권장 완화 조치:** 스토리지에서 읽는 것은 비용이 많이 들기 때문에 이상적으로 `updateNameAndSymbol`은 다음과 같이 해야 합니다:
* 기존 `name` 및 `symbol` 캐시
* `CommonUtils.isEqualString` 호출에 캐시된 값 사용
* 이벤트를 발생시키는 데 사용할 수 있도록 캐시된 값을 `_updateName` 및 `_updateSymbol`에 전달

**Securitize:** 인지함.


### 한 번만 호출되는 작은 `private` 함수 인라인

**설명:** `StandardToken::_updateName` 및 `_updateSymbol`은 `updateNameAndSymbol`에 의해 한 번만 호출되는 매우 작은 `private` 함수입니다.

따라서 인라인하는 것이 가스 효율적입니다. 다음은 `name` 및 `symbol`이 스토리지에서 한 번만 읽히도록 동일한 스토리지 읽기를 캐시하는 구현입니다:
```solidity
function updateNameAndSymbol(string calldata _name, string calldata _symbol) external onlyMaster {
    require(!CommonUtils.isEmptyString(_name), "Name cannot be empty");
    require(!CommonUtils.isEmptyString(_symbol), "Symbol cannot be empty");

    string memory nameCache = name;
    if (!CommonUtils.isEqualString(_name, nameCache)) {
        emit NameUpdated(nameCache, _name);
        name = _name;
    }

    string memory symbolCache = symbol;
    if (!CommonUtils.isEqualString(_symbol, symbolCache)) {
        emit SymbolUpdated(symbolCache, _symbol);
        symbol = _symbol;
    }
}
```

**Securitize:** 인지함.


### 사용되지 않는 `ExecutePreApprovedTransaction::nonce` 제거

**설명:** `ExecutePreApprovedTransaction::nonce`는 실제로 사용되지 않습니다:

* `GlobalRegistryService::hashTx`는 항상 스토리지 `noncePerInvestor[txData.senderInvestor]`에서 현재 nonce를 읽습니다
* 호출자는 서명하기 위해 현재 nonce를 사용했어야 합니다 - 그렇지 않으면 서명 검증이 실패합니다
* 검증이 성공하면 `executePreApprovedTransaction`은 항상 현재 nonce를 1씩 증가시켜 다시 사용될 수 없도록 합니다

따라서 위의 메커니즘은 nonce를 올바르게 검증하며 `ExecutePreApprovedTransaction::nonce`는 사용되지 않으므로 안전하게 제거할 수 있습니다.

**Securitize:** 커밋 [c841572](https://github.com/securitize-io/bc-global-registry-service-sc/commit/c841572de8b7dcfee484f6f7f4ce9a19e579bf21)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage

