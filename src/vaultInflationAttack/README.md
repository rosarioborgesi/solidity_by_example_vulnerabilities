# 💰 Vault Inflation Attack

**Reference:** https://solidity-by-example.org/hacks/vault-inflation/

---

## 📌 What is a Vault?

A **vault** is a smart contract that:
1. **Accepts deposits** of tokens from users
2. **Issues shares** representing proportional ownership
3. **Deploys funds** into various strategies to generate yield
4. **Allows withdrawals** where users redeem shares for their portion of assets (including profits/losses)

Think of it like a **mutual fund** or **ETF** for crypto, but fully automated and trustless.

### Primary Use Cases

#### 1. 💸 Yield Aggregation (Most Common)
**Example: Yearn Finance Vaults**

```
User deposits 100 USDC
    ↓
Vault gives user shares
    ↓
Vault automatically:
  - Lends USDC on Aave (5% APY)
  - Farms with liquidity pool tokens (8% APY)
  - Stakes in protocols (12% APY)
  - Auto-compounds rewards
    ↓
User withdraws → gets back 100 USDC + profits
```

#### 2. 💧 Liquidity Provision
**Example: Uniswap V3 Liquidity Vaults**

```
User deposits ETH + USDC
    ↓
Vault provides liquidity to Uniswap V3
    ↓
Earns trading fees automatically
Rebalances positions when price changes
    ↓
User withdraws → gets ETH + USDC + fees earned
```

#### 3. 🏦 Lending Protocols
**Example: Compound, Aave**

```
User deposits USDC
    ↓
Receives cUSDC (Compound) or aUSDC (Aave)
    ↓
These tokens represent shares in the lending pool
Value increases as interest accrues
    ↓
User redeems cUSDC → gets back USDC + interest
```

The **ERC-4626 standard** defines a common interface for vaults.

---
## 🔧 How the Vault Works

### Example: Alice Deposits 50 Tokens

**Initial State (Before Alice's Deposit):**

```
Vault Token Balance: 100 tokens
Total Shares Issued: 100 shares
Alice's Balance: 0 shares
```

---

#### Step 1: Calculate Shares

```solidity
shares = (_amount * totalSupply) / token.balanceOf(address(this));
shares = (50 * 100) / 100;
shares = 5000 / 100;
shares = 50;
```

**Alice will receive 50 shares** for her 50 tokens.

**Why 50 shares?**
- The vault has 100 tokens worth 100 shares → 1 token = 1 share ratio
- Alice deposits 50 tokens → she gets 50 shares
- She deposited 50 out of 150 total = 33.3% of the vault
- She receives 50 out of 150 total shares = 33.3% ownership ✅

---

#### Step 2: Mint Shares

**State after minting:**

```
totalSupply: 100 → 150 shares
Alice's Balance: 0 → 50 shares
```

---

#### Step 3: Transfer Tokens

Transfers 50 tokens from Alice to the Vault.

**State after transfer:**

```
Vault Token Balance: 100 → 150 tokens
Alice's Token Balance: (her balance - 50)
```

---

#### ✅ Final State (After Alice's Deposit)

```
Vault Token Balance: 150 tokens
Total Shares Issued: 150 shares
Alice's Share Balance: 50 shares
```

**Alice's ownership:** 50 shares / 150 total shares = **33.33%** of the vault

---

### Example: Alice Withdraws 50 Shares

**Initial State:**

```
Vault Token Balance: 150 tokens
Total Shares Issued: 150 shares
Alice's Share Balance: 50 shares
Other Users' Shares: 100 shares
```

---

#### Step 1: Calculate Token Amount to Return

Alice calls `withdraw(50)`:

```solidity
amount = (50 * 150) / 150 = 50 tokens
```

**Alice will receive 50 tokens** for her 50 shares.

---

#### Step 2: Burn Alice's Shares

```
Total Shares: 150 → 100 shares
Alice's Shares: 50 → 0 shares
```

⚠️ **Important:** Shares are burned BEFORE tokens are transferred.

---

#### Step 3: Transfer Tokens to Alice

```
Vault Token Balance: 150 → 100 tokens
Alice's Token Balance: +50 tokens
```

---

### 📊 Understanding `previewRedeem()`

`previewRedeem()` is an **ERC-4626 standard function** that calculates how many tokens you would receive if you redeemed your shares **right now**.

```solidity
vault.previewRedeem(shares0)
```

**This means:** "If user[0] withdrew all their shares right now, how many tokens would they get back?"

It's a **preview** function - it doesn't actually withdraw anything, it just **simulates** the withdrawal.

---
## ⚠️ The Vulnerability: Vault Inflation Attack

Vault shares can be **inflated by directly transferring (donating) ERC20 tokens** to the vault, bypassing the `deposit()` function.

**An attacker can exploit this to steal other users' deposits.**

---

### 🎯 Attack Overview

1. **User 0** (attacker) front-runs **User 1**'s deposit
2. **User 0** deposits `1 wei`
3. **User 0** "donates" `100 * 10^18` tokens directly to the vault (inflates share value)
4. **User 1** deposits `100 * 10^18` tokens → receives **0 shares** (rounds down!)
5. **User 0** withdraws all `200 * 10^18 + 1` tokens

**Result:** User 0 steals User 1's entire deposit! 💸

---

## 🧪 Demonstration

### Running the Test

The test is in `test/ValueTest.t.sol`.

**Run the test:**

```bash
forge test --mc VaultTest -vv
```

---

### Expected Output

```
Logs:
  --- users[0] deposit ---
  vault total supply 1
  vault balance 1
  users[0] shares 1
  users[1] shares 0
  users[0] redeemable 1
  users[1] redeemable 0
  
  --- users[0] donate ---
  vault total supply 1
  vault balance 100000000000000000001
  users[0] shares 1
  users[1] shares 0
  users[0] redeemable 100000000000000000001
  users[1] redeemable 0
  
  --- users[1] deposit ---
  vault total supply 1
  vault balance 200000000000000000001
  users[0] shares 1
  users[1] shares 0
  users[0] redeemable 200000000000000000001
  users[1] redeemable 0
```

---

## 🔍 Step-by-Step Breakdown

### Step 1: Users[0] Deposits 1 Token (Initial Setup)

```
vault total supply: 1 share
vault balance: 1 token
users[0] shares: 1
users[0] redeemable: 1 token
```

**Normal operation:** 1 token → 1 share

```solidity
vault.deposit(1);  // users[0] calls this
```

---

### Step 2: Users[0] "Donates" 100 Tokens (THE ATTACK!)

```
vault total supply: 1 share (UNCHANGED! ❌)
vault balance: 100000000000000000001 (100.000000000000000001 tokens)
users[0] shares: 1 (still just 1 share!)
users[0] redeemable: 100000000000000000001 (now worth 100+ tokens! 📈)
```

**The attack:**

```solidity
token.transfer(address(vault), 100 * 10**18);  // users[0] calls this
```

**What happened:**
- User[0] sent 100 tokens **directly** to the vault (bypassing `deposit()`)
- Total shares stayed at `1` ❌
- But vault balance jumped to 100+ tokens
- **Share price is now inflated:** 1 share = 100+ tokens!

---

### Step 3: Users[1] Deposits 100 Tokens (THE VICTIM!)

```
vault total supply: 1 share (STILL UNCHANGED!)
vault balance: 200000000000000000001 (200+ tokens)
users[0] shares: 1
users[1] shares: 0 ❌❌❌ ZERO SHARES!
users[0] redeemable: 200000000000000000001 (ENTIRE VAULT!)
users[1] redeemable: 0 (NOTHING!)
```

**The calculation that destroyed user[1]:**

```solidity
shares = (amount * totalSupply) / balance
shares = (100 * 10^18 * 1) / (100 * 10^18 + 1)
shares = 100,000,000,000,000,000,000 / 100,000,000,000,000,000,001
shares = 0.999999999999999999990000...
shares = 0 (integer division rounds down!)
```

**User[1] deposited 100 tokens but received 0 shares!** 💸

---

## 💡 Why Does users[1] Have 0 Shares?

This is the **heart of the attack**!

### State BEFORE users[1]'s Deposit:

```
vault balance: 100000000000000000001 tokens (100.000000000000000001)
totalSupply: 1 share
```

### The Share Calculation:

```solidity
shares = (amount * totalSupply) / token.balanceOf(address(this));
shares = (100 * 10^18 * 1) / 100000000000000000001;
```

**More clearly:**

```
shares = 100,000,000,000,000,000,000 / 100,000,000,000,000,000,001
```

**Mathematical result:**

```
shares = 0.99999999999999999999...
```

**But Solidity uses integer division, which ROUNDS DOWN:**

```
shares = 0
```

---

## 🎯 Why This Attack Worked

1. **First depositor advantage**: User[0] was first, so they set the initial share price

2. **Direct transfer inflates share price**: By sending tokens directly (not via `deposit()`), they made 1 share worth 100+ tokens

3. **Rounding down to zero**: When User[1] tried to deposit, the calculation rounded down to 0 shares

4. **Complete theft**: User[1]'s tokens went into the vault, but they got no ownership

---

## 🛡️ Prevention Techniques

### 1. Minimum Shares
Require a minimum number of shares on first deposit:

```solidity
if (totalSupply == 0) {
    require(_amount >= 1e6, "initial deposit too small");
    shares = _amount;
}
```

### 2. Internal Balance Tracking
Track expected balance to detect donations:

```solidity
uint256 internal expectedBalance;

function deposit(uint256 _amount) external {
    require(token.balanceOf(address(this)) == expectedBalance, "unexpected donation");
    // ... rest of logic
    expectedBalance += _amount;
}
```

### 3. Dead Shares
Burn some initial shares (like Uniswap V2):

```solidity
if (totalSupply == 0) {
    shares = _amount;
    _mint(address(0), 1000); // Burn minimum liquidity
}
```

### 4. Decimal Offset (OpenZeppelin ERC4626)
Use virtual shares/assets to make attacks economically infeasible:

```solidity
uint256 private constant _DECIMALS_OFFSET = 10**9;
// Multiplies share calculations by offset
```

---

## 🎯 Key Takeaways

1. **Never trust `balanceOf` alone** - Direct transfers can manipulate it

2. **First deposit is critical** - Attackers can front-run to set favorable conditions

3. **Integer division rounds down** - This creates the 0-share vulnerability

4. **ERC-4626 vaults need protections** - Always implement anti-inflation measures

5. **This is a real threat** - Many DeFi protocols have been vulnerable to this attack

---

## 📚 Summary

**The Vulnerability:** Vault share calculations can be manipulated by directly transferring tokens to inflate the share price, causing subsequent deposits to mint 0 shares due to integer division rounding.

**The Attack:** Deposit tiny amount → donate large amount → victim deposits → victim gets 0 shares → attacker withdraws everything.

**The Fix:** Implement minimum shares, internal balance tracking, dead shares, or decimal offset to make the attack economically infeasible or impossible.