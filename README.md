# real-wagmi-h3-analysis


### 🔗 Public Finding

[Sherlock H-3 — Creditor can maliciously burn UniV3 position to permanently lock funds](https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/78?utm_source=chatgpt.com)

I started from the affected code in `LiquidityManager.sol`, specifically `_restoreLiquidity()`, and noticed:

```solidity
address creditor = underlyingPositionManager.ownerOf(loan.tokenId);
```

Instead of starting from the published explanation, I tried to understand why the protocol needs `ownerOf(loan.tokenId)` at this point.

I followed `loan.tokenId` backwards and asked:

- What does `tokenId` represent?
- Who controls the corresponding NFT?
- Can that NFT become invalid while the loan is still active?
- What happens if the NFT no longer exists when `_restoreLiquidity()` uses it?

I found that `loan.tokenId` refers to a Uniswap V3 position NFT and that the creditor controls that NFT.

That led me to check the scenario where the creditor burns the NFT while the loan still references the same `tokenId`.

My reasoning was:

```text
loan.tokenId
    ↓
references a V3 NFT
    ↓
creditor controls the NFT
    ↓
NFT can be burned
    ↓
loan.tokenId is still stored
    ↓
NFT no longer exists
    ↓
_restoreLiquidity()
    ↓
ownerOf(loan.tokenId)
    ↓
REVERT
```

So my initial conclusion was that the protocol stores a reference to an externally controlled NFT, but assumes that the NFT will still exist when that reference is consumed later.

After reaching this conclusion, I compared my reasoning with the published H-3 finding, and the core behavior matched.

---

### Affected Code

**Codebase:** `RealWagmi/wagmi-leverage`

**Affected code:**  
`contracts/abstract/LiquidityManager.sol` — `_restoreLiquidity()`

Relevant code:

```solidity
address creditor = underlyingPositionManager.ownerOf(loan.tokenId);
```

The published issue references this commit as the fix:

### 🔗 Fix Commit

[RealWagmi — fix commit 788c72b](https://github.com/RealWagmi/wagmi-leverage/commit/788c72b8242cc704e5db58cfdbfcdf8d4559d4b5?utm_source=chatgpt.com)

I am treating this as the **fix commit**, rather than claiming that it is the vulnerable contest commit.

---

### What I Verified

The important state transition I wanted to verify was:

```text
NFT exists
    ↓
creditor owns NFT
    ↓
creditor burns NFT
    ↓
tokenId is still referenced by the loan
    ↓
NFT no longer exists
    ↓
ownerOf(tokenId) would revert
```

To check this, I wrote my own small mock position manager and test.

I did not start from the published PoC. I created my own mock to reproduce the NFT state transition.

**Test file:**

```text
test/foundry/BurnedUniV3Position.t.sol
```

**Mock:**

```solidity
contract MockPositionManager {
    mapping(uint256 => address) private _owners;

    function mint(address to, uint256 tokenId) external {
        _owners[tokenId] = to;
    }

    function burn(uint256 tokenId) external {
        require(_owners[tokenId] == msg.sender, "not owner");
        delete _owners[tokenId];
    }

    function ownerOf(uint256 tokenId)
        external
        view
        returns (address owner)
    {
        owner = _owners[tokenId];

        require(
            owner != address(0),
            "ERC721: owner query for nonexistent token"
        );
    }
}
```

**Test I wrote:**

```solidity
function test_creditor_can_maliciously_burn_uniV3_position_to_permanently_lock_funds()
    public
{
    uint256 tokenId = 7;

    nft.mint(address(this), tokenId);
    nft.burn(tokenId);

    assertEq(manager.$_getOwnerOf(tokenId), address(0));
}
```

I then ran the test locally with Foundry.

The result was:

```text
[PASS] test_creditor_can_maliciously_burn_uniV3_position_to_permanently_lock_funds()

1 tests passed, 0 failed, 0 skipped
```

This confirmed the important state transition I was checking: after the NFT is burned, the `tokenId` can still be referenced while the NFT itself no longer has an owner.

The test is not intended to independently prove the entire “permanently lock funds” impact. It verifies the underlying condition that makes the vulnerable `ownerOf(tokenId)` call revert.

I also attached a screenshot of the test running successfully in Foundry.

---

### Verification Against the Published Finding

After I had my own state → mutation → consumer → impact reasoning and my own test, I compared it with the published H-3 finding.

The core behavior matched:

```text
My reasoning
    ↓
externally controlled V3 NFT
    ↓
NFT can be burned
    ↓
loan.tokenId remains stored
    ↓
ownerOf(tokenId) is called later
    ↓
revert
    ↓
repayment/recovery can be blocked

Published finding
    ↓
same core issue
```

So for this exercise, I am **not claiming the finding as my own**.

I used the published H-3 finding as the case study, independently traced the behavior, wrote my own test to verify the underlying condition, and then confirmed that my reasoning matched the published finding.

---

### Assumptions

My reasoning depends on:

- The creditor controls the V3 NFT.
- The creditor can burn it.
- The loan continues to reference the burned `tokenId`.
- A later protocol operation uses that `tokenId`.
- `ownerOf()` reverts when the NFT does not exist.
- There is no alternative recovery path that bypasses the destroyed NFT.

### Conclusion

The issue is not that `ownerOf()` itself is incorrect.

The issue is the lifecycle assumption around an externally controlled NFT.

The protocol stores `loan.tokenId`, but the creditor can destroy the NFT that the `tokenId` refers to. When the protocol later consumes that reference, `ownerOf()` can revert and block the dependent repayment/restoration and recovery flow.

The main audit pattern I took from this is:

```text
STATE
loan.tokenId
    ↓
MUTATION
creditor burns NFT
    ↓
CONSUMER
ownerOf(loan.tokenId)
    ↓
IMPACT
revert → repayment/recovery blocked → funds locked
```

This helped me understand the finding as a **state-lifecycle issue** rather than simply an incorrect `ownerOf()` call.