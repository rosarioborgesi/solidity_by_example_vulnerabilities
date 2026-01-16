## Honeypot Trap

**Reference:** https://solidity-by-example.org/hacks/honeypot/

This demonstrates a **honeypot** - a deliberately vulnerable-looking contract that's actually a trap for attackers. The contract appears to have a reentrancy vulnerability, baiting hackers to exploit it, but secretly prevents the attack from succeeding.

### 🎯 The Concept

**What is a Honeypot?**

A honeypot is a security trap designed to detect, deflect, or study hacking attempts. In smart contracts, it's a contract that:
- ✅ Appears to have an obvious vulnerability (bait)
- ✅ Attracts attackers who think they found an exploit
- ✅ Secretly prevents the attack from succeeding (trap)
- ✅ May even drain the attacker's funds

**This Example:**

The `Bank` contract appears to have a classic reentrancy vulnerability in `withdraw()`. An attacker sees this and deploys an `Attack` contract to exploit it. However, `Bank` secretly uses a `HoneyPot` logger instead of the normal `Logger`, which reverts on withdrawals, causing the attack to fail!

---

### 🚀 Setup

**Start Anvil (local blockchain)**

```bash
anvil
```

**Compile code**
```bash
forge build
```

---

### 🍯 Step 1: Alice Sets the Trap (Deploys HoneyPot)

Alice deploys the malicious logger that will trigger the trap.

**Alice's Details:**
- Address: `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`
- Private Key: `0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80`

```bash
forge create src/honeyPot/HoneyPot.sol:HoneyPot \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast

  # Contract deployed to: 0x5FbDB2315678afecb367f032d93F642f64180aa3
```

💡 **The Secret:** This `HoneyPot` contract looks like a normal `Logger` but reverts when `action == "Withdraw"`!

---

### 🏦 Step 2: Alice Deploys Bank (The Bait)

Alice deploys the `Bank` contract with the `HoneyPot` address, but disguises it as a `Logger`.

**The Deception:** The contract code shows:
```solidity
Logger logger;  // ← Looks innocent!
```

But the constructor receives the **HoneyPot address** instead!

```bash
forge create src/honeyPot/HoneyPot.sol:Bank \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast \
  --constructor-args 0x5FbDB2315678afecb367f032d93F642f64180aa3

  # Contract deployed to: 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
```

---

### 💰 Step 3: Alice Deposits Funds (The Bait)

Alice deposits 1 ETH to make the contract look like a juicy target.

```bash
cast send 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512 \
  "deposit()" \
  --value 1ether \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

**Verify the bait is in place:**

```bash
cast balance 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512 -e
# 1.000000000000000000 ETH
```

🎣 **Perfect bait!** The Bank has 1 ETH and appears vulnerable to reentrancy.

---

### 🔍 Step 4: Eve Discovers the "Vulnerability"

Eve reviews the `Bank` contract and finds the reentrancy vulnerability:

```solidity
function withdraw(uint256 _amount) public {
    require(_amount <= balances[msg.sender], "Insufficient funds");
    
    (bool sent,) = msg.sender.call{value: _amount}("");  // ⚠️ Sends ETH first!
    require(sent, "Failed to send Ether");
    
    balances[msg.sender] -= _amount;  // ❌ Updates balance AFTER sending!
    
    logger.log(msg.sender, _amount, "Withdraw");
}
```

Eve thinks: "Classic reentrancy bug! I can drain this!"

**Eve's Details:**
- Address: `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`
- Private Key: `0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a`

**Eve deploys the Attack contract:**

```bash
forge create src/honeyPot/HoneyPot.sol:Attack \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a \
  --broadcast \
  --constructor-args 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512

  # Contract deployed to: 0x663F3ad617193148711d28f5334eE4Ed07016602
```

---

### 💥 Step 5: The Trap is Sprung!

Eve attempts the reentrancy attack with 1 ETH:

```bash
cast send 0x663F3ad617193148711d28f5334eE4Ed07016602 \
  "attack()" \
  --value 1ether \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a
```

**Result: ❌ Transaction Fails!**

```
Error: Failed to estimate gas: server returned an error response: error code 3: 
execution reverted: Failed to send Ether, data: "0x08c379a0000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000144661696c656420746f2073656e64204574686572000000000000000000000000"
```

🎯 **The honeypot worked!** Eve's attack failed and her 1 ETH is lost!

---

### 🔍 How the Trap Works

#### The Attack Flow (Step by Step)

**Initial State:**
- Bank balance: 2 ETH (Alice's 1 ETH + Attack's deposit of 1 ETH)
- `balances[Attack] = 1 ETH`

**Call 1: First `withdraw(1 ether)`**

```solidity
function withdraw(uint256 _amount) public {
    require(_amount <= balances[msg.sender]);      // ✓ 1 <= 1 (passes)
    
    (bool sent,) = msg.sender.call{value: _amount}("");  // Sends 1 ETH
    // ↓ This triggers Attack.fallback(), which calls withdraw() AGAIN!
```

**Call 2: Second `withdraw(1 ether)` (Reentrant Call)**

```solidity
    require(_amount <= balances[msg.sender]);      // ✓ STILL 1! (line 44 hasn't run yet)
    
    (bool sent,) = msg.sender.call{value: _amount}("");  // Sends the last 1 ETH
    // ↓ Triggers fallback, but Bank.balance = 0, so no more reentrancy
    
    require(sent, "Failed to send Ether");         // ✓ sent = true
    
    balances[msg.sender] -= _amount;               // balances[Attack] = 0
    
    logger.log(msg.sender, _amount, "Withdraw");   // ← THE TRAP SPRINGS HERE!
    // ↓ Calls HoneyPot.log() instead of Logger.log()
    // ↓ HoneyPot checks: action == "Withdraw"? YES!
    // ↓ revert("It's a trap") ← BOOM! 💥
}
```

**Back to Call 1:**

```solidity
    (bool sent,) = msg.sender.call{value: _amount}("");  // Line 41
    // ↑ The reentrant call REVERTED with "It's a trap"
    // ↑ So sent = FALSE (the entire nested call failed!)
    
    require(sent, "Failed to send Ether");         // ✗ Fails!
    // ↑ Reverts with "Failed to send Ether"
}
```

#### Visual Call Stack

```
Attack.attack()
  ├─> Bank.deposit() [Attack deposits 1 ETH]
  └─> Bank.withdraw(1 ETH) [1st call]
        ├─> send 1 ETH to Attack
        └─> Attack.fallback() [receives ETH, sees Bank has 1 ETH left]
              └─> Bank.withdraw(1 ETH) [2nd call - REENTRANT!]
                    ├─> send 1 ETH to Attack [Bank now empty]
                    ├─> balances[Attack] = 0
                    └─> HoneyPot.log("Withdraw")
                          └─> revert("It's a trap") ← TRAP TRIGGERED! 💥
                    [2nd withdraw reverts, returns all state changes]
              [fallback call returns false]
        [1st withdraw: sent = false]
        └─> revert("Failed to send Ether") ← This is the error you see
  [Entire transaction reverts, Eve loses her 1 ETH deposit!]
```

---

### 🧩 Decoding the Error

The error data contains the revert message encoded in ABI format:

```
data: "0x08c379a0000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000144661696c656420746f2073656e64204574686572000000000000000000000000"
```

**Part 1: Error Selector (first 4 bytes)**
```
0x08c379a0  ← Function selector for Error(string)
```

Verify with:
```bash
cast sig "Error(string)"
# Returns: 0x08c379a0
```

**Part 2: Decode the Message**

```bash
cast --to-ascii 0x4661696c656420746f2073656e64204574686572
```

Returns: **"Failed to send Ether"**

**Using cast decode-error**

```bash
cast decode-error  0x08c379a0000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000144661696c656420746f2073656e64204574686572000000000000000000000000
# Error(string)
# "Failed to send Ether"
```

**Why not "It's a trap"?**

The inner error ("It's a trap") is masked by the outer error ("Failed to send Ether"). The EVM only shows the outermost revert reason. The HoneyPot's revert happens deep in the call stack, causing the outer `require(sent)` to fail with its own error message.

---

### 🎯 Why This is Brilliant

**From Eve's Perspective:**
1. ✓ Found a reentrancy vulnerability
2. ✓ Deployed attack contract
3. ✓ Deposited 1 ETH to execute the attack
4. ❌ Transaction failed
5. ❌ Lost 1 ETH trying to steal 2 ETH

**From Alice's Perspective:**
1. ✓ Deployed honeypot with fake vulnerability
2. ✓ Deposited 1 ETH as bait
3. ✓ Attacker found the "vulnerability"
4. ✓ Attacker lost 1 ETH attempting the exploit
5. ✓ Alice's 1 ETH is safe, plus she captured Eve's 1 ETH!

**The Trap Components:**

1. **Bait:** Obvious reentrancy vulnerability in `withdraw()`
2. **Camouflage:** Contract declares `Logger logger` (looks innocent)
3. **Hidden weapon:** Constructor receives `HoneyPot` address (hidden from code inspection)
4. **Trigger:** HoneyPot reverts on "Withdraw" action
5. **Timing:** Revert happens AFTER attacker has deposited funds

---

### 📋 Key Takeaways

- ✅ **Honeypots are traps** designed to catch attackers
- ✅ **Type casting is unchecked** - `Logger(address)` doesn't verify the contract type
- ✅ **Deployment parameters are hidden** - users can't see constructor arguments from code alone
- ✅ **Inner revert reasons are masked** - only the outermost error is shown
- ✅ **Always verify deployed addresses** - check actual contract code, not just the interface
- ✅ **Don't exploit vulnerabilities** - it's illegal and you might get trapped!

**Remember:** If a vulnerability looks too good to be true, it probably is. Always verify external contracts and test thoroughly before risking funds!

