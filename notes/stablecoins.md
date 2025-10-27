At the end of every day, I review my completed chapters by adding notes to this file. I've been extra verbose in sections I found difficult (e.g. deployment scripts and fuzz testing).

### 3.1 DeFi Introduction
- Quick snapshot: https://defillama.com/
- Intro to Miner/Maximal Extractable Value (MEV): https://docs.flashbots.net/new-to-mev

### 3.2 Project Code Walkthrough
- 2 contracts: `DecentralizedStablecoin.sol` (ERC20) and `DSCEngine.sol` (core protocol)
- Unit, fuzz, and invariant tests
- Deploy scripts

### 3.3 Introduction to Stablecoins
Stablecoins are non-volatile crypto assets.

They may be categorized accordiing to the following:
1. Relative stability: Pegged or Floating
2. Stability method: Governed or Algorithmic
3. Collateral type: Exogenous or Endogenous

Famous paper on endogenous stablecoins (collateral originates from within the protocol, rather than from outside): https://blog.bitmex.com/wp-content/uploads/2018/06/A-Note-on-Cryptocurrency-Stabilisation-Seigniorage-Shares.pdf

### 3.4 DecentralizedStablecoin.sol
Our stablecoin will be:
- Pegged to USD: Using Chainlink pricefeeds and conversion functions between ETH or BTC to USD
- Algorithmic: Over-collateralized loans
- Exogenous: wETH and wBTC originate from outside the protocol

### 3.5 Project Setup - DSCEngine

### 3.6 Create the Deposit Collateral Function
Always consider input sanitization when passing values to functions.

Instead of writing conditionals multiple times, use modifiers instead. For example, for inputs which have to be more than zero:
```Solidity
modifier moreThanZero(uint256 amount) {
    if (amount <= 0) {
        revert DSCEngine_NeedsMoreThanZero();
    }
    _;
}
```

Emit events whenever state is changed.

Recall the Checks, Effects, Interactions pattern. Executing external calls at the end helps protect against reentrancy.

Remember to check the return value of the ERC20 transfer() and transferFrom() functions. If they return a boolean, we can write a function to revert if the transfer returns false:
```Solidity
bool success = IERC20(token).transferFrom(msg.sender, address(this), amount);

if (!success) {
    revert DSCEngine_TransferFailed();
}
```

### 3.7 Quiz

### 3.8 Creating the Mint Function
To mint securely, mint() has to ensure the user's collateral value supports the amount being minted. This will require oracle pricefeeds, value conversions, and more.

Health factor: (Sum of collateral / Sum of Borrows) * Liquidation Threshold.
For example, if Liquidation Threshold == 0.5, the protocol requires 200% collateralization minimum.
If a user's collateral/borrows == 2 (the minimum), health factor == 1.

Remember to adjust for precision / decimal places. To get the correct values you need both inputs to have the same number of decimal places (otherwise the result will be greater/larger than expected).

For example, if ETH => USD conversion returns 400000000000 to 8 decimal places ($4000), while the amount of ETH deposited is 1000000000000000000 to 18 decimal places (1 ETH), you the conversion to return 4000. But directly multiplying the amount of ETH by its conversion return value and scaling back to ETH's precision (1e18 * 4e12 / 1e18)  returns 4e12 (too large). To fix, first multiply 4e12 by 1e10, adjusting its precision to 18 decimal values. Then, 1e18 * 4e22 / 1e18 returns 4e4, which is $4000 (correct).

In our code, we implement it as follows:
```
uint256 private constant ADDITIONAL_FEED_PRECISION = 1e10;
uint256 private constant PRECISION = 1e18;

...
function getUsdValue(address token, uint256 amount) public view returns (uint256) {
    AggregatorV3Interface priceFeed = AggregatorV3Interface(s_priceFeeds[token]);
    (,uint256 price,,,) = priceFeed.latestRoundData();

    return ((uint256(price) * ADDITIONAL_FEED_PRECISION) * amount) / PRECISION;
}
```

### 3.9 Creating and Retrieving the Health Factor
Because the EVM only works with whole numbers, we can implement liquidation thresholds like so:
```
uint256 private constant LIQUIDATION_THRESHOLD = 50;
uint256 private constant LIQUIDATION_PRECISION = 100;
```
Then to calculate the health factor:
```
function _healthFactor(address user) private view returns (uint256) {
    (uint256 totalDscMinted, uint256 collateralValueInUsd) = _getAccountInformation(user);
    uint256 collateralMargin = (collateralValueInUsd * LIQUIDATION_THRESHOLD) / LIQUIDATION PRECISION;
    return collateralMargin * PRECISION / totalDscMinted;
}
```
Remember to multiply before dividing in order to avoid rounding down errors.
And make `collateralMargin` more precise before dividing by totalDscMinted!
Now, this means that the minimum health factor should be set as follows:
```
uint256 private constant MIN_HEALTH_FACTOR = 1e18
```

### 3.10 Finish the Mint Function

### 3.11 Creating the Deployment Script
Deploys and returns our contracts.
```
...
import {Script} from "forge-std/Script.sol";
import {DecentralizedStablecoin} from "../src/DecentralizedStablecoin.sol";
import {DSCEngine} from "../src/DSCEngine.sol";

contract DeployDSC is Script {
    function run() external returns (DecentralizedStablecoin, DSCEngine) {}
}
```
DSCEngine takes several constructor parameters, such as tokenAddresses[], priceFeedAddresses[], and the DecentralizedStablecoin deployment address. To provide these, we can use a `HelperConfig` as follows:
```
...
contract HelperConfig is Script {
    struct NetworkConfig {
        address wethUsdPriceFeed;
        address wbtcUsdPriceFeed;
        address weth;
        address wbtc;
        uint256 deployerKey;
    }
    NetworkConfig public activeNetworkConfig;
    constructor() {}
}
```
The idea is to write various functions that return a NetworkConfig struct filled with the appropriate addresses for the chain in use (Sepolia, local Anvil, etc). For Sepolia, we can find the addresses online. On Anvil, however, we'll have to deploy some mocks (remember to import these in the HelperConfig file):
```
    vm.startBroadcast();
    MockV3Aggregator ethUsdPriceFeed = new MockV3Aggregator(DECIMALS, ETH_USD_PRICE);
    ERC20Mock wethMock = new ERC20Mock();
​
    MockV3Aggregator btcUsdPriceFeed = new MockV3Aggregator(DECIMALS, BTC_USD_PRICE);
    ERC20Mock wbtcMock = new ERC20Mock();
    vm.stopBroadcast();
```
Then write a constructor that chooses which function to call based on the `block.chainid` of our deployment:
```
constructor() {
    if (block.chainid == 11155111) {
        activeNetworkConfig = getSepoliaEthConfig();
    } else {
        activeNetworkConfig = getOrCreateAnvilEthConfig();
    }
}
```
In our deploy script, import the `HelperConfig` contract and enter the following into `run()`:
```
HelperConfig config = new HelperConfig();
(address wethUsdPriceFeed, address wbtcUsdPriceFeed, address weth, address wbtc, uint256 deployerKey) = config.activeNetworkConfig()
```
Now we can create our tokenAddresses and priceFeedAddresses arrays and pass them into our main contract deployments:
```
    tokenAddresses = [weth, wbtc];
    priceFeedAddresses = [wethUsdPriceFeed, wbtcUsdPriceFeed];
​
    vm.startBroadcast();
    DecentralizedStableCoin dsc = new DecentralizedStableCoin();
    DSCEngine engine = new DSCEngine(tokenAddresses, priceFeedAddresses, address(dsc));
    vm.stopBroadcast();
```

### 3.12: Test the DSCEngine Smart Contract
Now, we can do the following in our test contract. First import the main contracts, the forge-std Test contract, and our Deploy contract.
```
contract DSCEngineTest is Test {
    DeployDSC deployer;
    DecentralizedStablecoin dsc;
    DSCEngine dsce;

    function setUp() public {
        deployer = new DeployDSC();
        (dsc, dsce) = deployer.run(); // because our deployer returns these contracts
    }
}
```
For easy access to our mock token and pricefeed addresses, it's convenient to let our deploy script also return our HelperConfig:
```
...
function run() external returns (DecentralizedStablecoin, DSCEngine, HelperConfig)
...
return (dsc, engine, config);
```
Then in our `setUp` function, we can add:
```
(ethUsdPriceFeed, , weth, , ) = config.activeNetworkConfig();
```
Now these can be referenced in our test functions.
However, remember that the HelperConfig only returns the addresses of the tokens and pricefeeds. To access their functions, such as calling `weth.mint()`, we need its interface/mock/contract. So, import `ERC20Mock`, then you can call `ERC20Mock(weth).mint(...)`.

### 3.13 Quiz 7

### 3.14 Create the depositAndMint Function

### 3.15 Create the redeemCollateral Function
Sometimes it's okay to break the CEI pattern.
For example, we want to check whether a transaction breaks the health factor, but if we were to check that first, we'd have to simulate all the transactions first, which would cost a lot of gas. Hence, it's okay to leave `_revertIfHealthFactorIsBroken()` at the end of a function.

### 3.16 Setup Liquidations

### 3.17 Refactor Liquidations
Currently, `redeemCollateral` has `msg.sender` hardcoded as the user for which collateral is redeemed. However, our liquidation mechanism would like for liquidators to redeem another user's collateral if they are under-collateralized. 

We can fix this by writing an internal `_redeemCollateral` function which includes an `address from` and an `address to` as parameters. 

Then, in our public `redeemCollateral` function, we can simply call this internal version, with the from and to addresses hardcoded as `msg.sender`! Meanwhile, in our `liquidate()` function, we can call the internal `_redeemCollateral` function too, except that `from` is the user being liquidated and `to` is set to msg.sender (the liquidator).

### 3.18 DSCEngine Advanced Testing
vm.expectRevert takes `bytes4 revertData` as the first parameter.
Where we have custom errors, we can feed its 4-byte signature into vm.expectRevert via something like `vm.expectRevert(CustomError.seelctor);`. Where the error itself has parameters, we should do `vm.expectRevert(abi.encodeWithSelector(CustomError.selector, para1, para2))`.
See more: https://getfoundry.sh/reference/cheatcodes/expect-revert/

### 3.19 Quiz 8

### 3.20 Create the Fuzz Tests Handler pt.1
When developing a protocol and writing tests, always be asking: "What are my invariants?"
This will make advanced testing easier.

In Foundry, you can automate fuzz testing by adding parameters to your tests and referring to those parameters within the test function. Foundry will automatically supply pseudo-random values to those parameters when testing. For instance:
```
function testAlwaysGetZero(uint256 x) public {
    myContract.doStuff(x);
    assert(myContract.shouldAlwaysBeZero() == 0);
}
```
Configure our fuzzer in `foundry.toml`:
```
[fuzz]
runs = 1000
```
However, this only conducts stateless fuzzing, where the state at the end of every run is refreshed.

To setup stateful fuzz testing, we have to import `StdInvariant.sol` and have our test contract inherit it. Then, set up the target contract by calling `targetContract(address)` in the test's `setUp()` function:
```
function setUp() public {
    myContract = new MyContract();
    targetContract(address(myContract));
}
```
Finally, write our invariant by using the keyword `invariant` at the beginning of a function, then declare our assertion in the function body. The fuzzer will start calling functions in the target contract to try and break our assertion.

### 3.21: Create the Fuzz Tests Handler pt.2
Invariant tests can be "open" or "handler-based".
- Open testing: Fuzzer calls any function randomly, in random orders. Often leads to "bad runs", e.g. fuzzer calling depositCollateral without having minted any collateral.
- Test handlers: Narrow the fuzzer's focus, e.g. by configuring aspects of a contract's state beforehand. Leads to fewer "bad runs", but can make bad assumptions.

First, set up fuzzer options in `foundry.toml`:
```
[invariant]
runs = 128
depth = 128
fail_on_revert = false 
```
Depth refers to the number of calls made in each run.
Create directory `test/fuzz` with files `InvariantsTest.t.sol` and `Handler.t.sol`
The idea with handlers is that our fuzzer (`InvariantsTest.t.sol`) call random functions in `Handler.t.sol` (the new target contract!), which then calls random functions in `DecentralizedStablecoin.sol` and `DSCEngine.sol`. The functions in the handler are better configured (for example, it will pre-mint tokens before calling `deposit` in the core contracts).

### 3.22 Defi Handler Deposit Collateral
Problem, although our depositCollateral tests now pre-mint some tokens, our tests keep reverting because the token addresses are being randomized to values that are neither wBTC or wETH!

To fix this, we can write a helper function that picks a valid collateral based on the random value provided by our framework:
```
function _getCollateralFromSeed(uint256 collateralSeed) private view returns (ERC20Mock) {
    if (collateralSeed % 2 = 0) {
        return weth;
    } else {
        return wbtc;
    }
}
```
Then, our handler's depositCollateral() function can look like this:
```
function depositCollateral(uint256 collateralSeed, uint256 amountCollateral) public {
    ERC20Mock collateral = _getCollateralFromSeed(collateralSeed);
    dsce.depositCollateral(address(collateral), amountCollateral);
}
```
Another problem might be that our function sometimes sends an `amountCollateral` that is either 0 or too big that it overflows, etc. To avoid such errors, we can use the `bound` function provided by the StdUtils library.
- Use input = bound(input, min, max) to specify a range of valid inputs
For instance, `amountCollateral = bound(amountCollateral, 1, type(uint96).max);`

### 3.23 Create the Collateral Redeem Handler

### 3.24 Create the Mint Handler
- Pro tip: You can split your test suite into one file where `fail_on_revert` is set to false so tests continue, and another file where `fail_on_revert` is set to true. Put the appropriate tests into the appropriate file.

### 3.25 Debugging the Fuzz Tests Handler
- Ghost variables: Basically like state variables for the handler
- Useful for debugging, such as tracking how many times a function is actually called
- Problem: Fuzzer calls random functions from random addresses. So, the address that mints DSC in our fuzzer is often different from the one which deposits collateral. This needs fixing.
- One way is to create an address[] of users with deposits, .push the msg.sender to that array after every successful deposit, then use a seed to randomly select one of those users to mint.

### 3.26 Quiz 9

### 3.27 Create the Price Feed Handler
Import `MockV3Aggregator.sol` to mimic the behaviour of a price feed.

### 3.28 Manage Your Oracle Connections
We can write a library to protect against oracle misbehaviours.
For example, to ensure the prices DSCEngine uses aren't "stale", we can do:
```
library OracleLib {
    error OracleLib_StalePrice();

    function staleCheckLatestRoundData(AggregatorV3Interface pricefeed) public view returns(uint80, int256, uint256, uint256, uint80) {
        (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound) = pricefeed.latestRoundData();
        uint256 secondsSinceUpdate = block.timestamp - updatedAt;
        if (secondsSinceUpdate > 3 hours) revert OracleLib_StalePrice();

        return (roundId, answer, startedAt, updatedAt, answeredInRound);
    }
}
```
Then in `DSCEngine.sol`, we can import `OracleLib.sol` and replace all our AggregatorV3Interface calls to `latestRoundData` with `staleCheckLatestRoundData`, which reverts if the oracle's price is stale.
```
using OracleLib for AggregatorV3Interface
```
The above basically tells Solidity: "Attach all functions from the OracleLib library to the AggregatorV3Interface type, so I can call them as if they were methods on that type".

### 3.29 Preparing Your Protocol for an Audit
Audit Readiness Checklist: https://github.com/nascentxyz/simple-security-toolkit/blob/main/audit-readiness-checklist.md

### 3.30 Section Recap
Codehawks Audit of the DSC/DSCE codebase! https://codehawks.cyfrin.io/c/2023-07-foundry-defi-stablecoin





