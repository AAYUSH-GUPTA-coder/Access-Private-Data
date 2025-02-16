# Accessing `private` data

When we start writing smart contracts and come across visibility modifiers like `public`, `private`, etc. we may think that if you want some variable's value to be readable by the public you need to declare it `public`, and that `private` variables cannot be read by anyone but the smart contract itself.

But, Ethereum is a public blockchain. So what does `private` data even mean?

In this level, we will see how you can actually read `private` variable values from any smart contract, and also clarify what `private` actually stands for - which is definitely not private data!

Lets go 🚀

---

## What does `private` mean?

Function (and variable) visibility modifiers only affect the visibility of the function - and do not prevent access to their values. We know that `public` functions are those which can be called both externally by users and smart contracts, and also by the smart contract itself.

Similarly, `internal` functions are those which can only be called by the smart contract itself, and outside users and smart contracts cannot call those functions. `external` functions are the opposite, where they can only be called by external users and smart contracts, but not the smart contract that has the function itself.

`private`, similarly, just affects who can call that function. `private` and `internal` behave mostly similarly, except the fact that `internal` functions are also callable by derived contracts, whereas `private` functions are not.

---

So for example, if `Contract A` has a function `f()` which is marked `internal`, a second `Contract B` which inherits `Contract A` like

```
contract B is A {
  ...
}
```

can still call `f()`.

However, if `Contract A` has a function `g()` which is marked `private`, `Contract B` cannot call `g()` even if it inherits from `A`.

The same is true for variables, as variables are basically just functions. `private` variables can only be accessed and modified by the smart contract itself, not even derived contracts. However, this does not mean that external parties cannot read the value.

## BUIDL

We will build a simple contract, along with a Hardhat Test, to demonstrate this. Our contract will attempt to store data in `private` variables hoping that nobody will be able to read it's value.

The contract will take in `password` and `username` in its contructor and will store them in private variables.

User will somehow be able to access those private variables.

### Concepts

To understand how this works, recall from the Ethereum Storage and Execution level that variables in Solidity are stored in 32 byte (256 bit) storage slots, and that data is stored sequentially in these storage slots based on the order in which these variables are declared.

Storage is also optimized such that if bunch of variables can fit in one slot, they are put in the same slot. This is called variable packing, and we will learn more about this later.

---

- To setup a Hardhat project, Open up a terminal and execute these commands in a new folder

  ```bash
  npm init --yes
  npm install --save-dev hardhat
  ```

- If you are on Windows, please do this extra step and install these libraries as well :)

  ```bash
  npm install --save-dev @nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers ethers
  ```

- In the same directory where you installed Hardhat run:

  ```bash
  npx hardhat
  ```

  - Select `Create a basic sample project`
  - Press enter for the already specified `Hardhat Project root`
  - Press enter for the question on if you want to add a `.gitignore`
  - Press enter for `Do you want to install this sample project's dependencies with npm (@nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers ethers)?`

Now you have a hardhat project ready to go!

Let's start by creating a `Login.sol` file inside the `contracts` folder. Add the following lines of code to your file

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

contract Login {

    // Private variables
    // Each bytes32 variable would occupy one slot
    // because bytes32 variable has 256 bits(32*8)
    // which is the size of one slot

    // Slot 0
    bytes32 private username;
    // Slot 1
    bytes32 private password;

    constructor(bytes32  _username, bytes32  _password) {
        username = _username;
        password = _password;
    }
}
```

Since both declared variables are `bytes32` variables, we know that each variable takes up exactly one storage slot. Since the order matters, we know that `username` will take up `Slot 0` and `password` will take up `Slot 1`.

Therefore, instead of attempting to read these variable values by calling the contract, which is not possible, we can just access the storage slots directly. Since Ethereum is a public blockchain, all nodes have access to all the state.

Let's write a Hardhat Test to demonstrate this functionality.

---

Create a new file `attack.js` inside the `test` folder and add the following lines of code

```javascript
const { ethers } = require("hardhat");
const { expect } = require("chai");

describe("Attack", function () {
  it("Should be able to read the private variables password and username", async function () {
    // Deploy the login contract
    const loginFactory = await ethers.getContractFactory("Login");

    // To save space, we would convert the string to bytes32 array
    const usernameBytes = ethers.utils.formatBytes32String("test");
    const passwordBytes = ethers.utils.formatBytes32String("password");

    const loginContract = await loginFactory.deploy(
      usernameBytes,
      passwordBytes
    );
    await loginContract.deployed();

    // Get the storage at storage slot 0,1
    const slot0Bytes = await ethers.provider.getStorageAt(
      loginContract.address,
      0
    );
    const slot1Bytes = await ethers.provider.getStorageAt(
      loginContract.address,
      1
    );

    // We are able to extract the values of the private variables
    expect(ethers.utils.parseBytes32String(slot0Bytes)).to.equal("test");
    expect(ethers.utils.parseBytes32String(slot1Bytes)).to.equal("password");
  });
});
```

In this test, we first create `usernameBytes` and `passwordBytes`, which are `bytes32` versions of a short string to behave as our username and password. We then deploy the `Login` contract with those values.

After the contract is deployed, we use `provider.getStorageAt` to read storage slot values at `loginContract.address` for slots 0 and 1 directly, and extract the byte values from it.

Then, we can compare the retrieved values - `slot0Bytes` against `usernameBytes` and `slot1Bytes` against `passwordBytes` to ensure they are, in fact, equal.

If the tests pass, it means we were successfully able to read the values of the private variables directly without needing to call functions on the contract at all.

Finally, lets run this test and see if it works. On your terminal type:

```bash
npx hardhat test
```

The tests should pass. Wow, we could actually read the password!

## Prevention

NEVER store private information on a public blockchain. No other way around it.
