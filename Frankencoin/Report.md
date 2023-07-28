# FrankenCoin Audit
### DedOhwale

## Highs

## [H-01] Vulnerability to Front-Running Attack due to Insufficient Checks in Position Contract

## Impact

The Positions contract in the codebase allows for interactions even after the `cooldown` has been set to `expired`. It further permits launching a challenge on a contract from which funds have already been withdrawn. Moreover, the function `initializeClone` does not contain modifiers for checking cooldown, potentially leading to inconsistencies in contract status checks.

This design flaw can lead to a front-running attack where a user seeing a challenge transaction in the mempool can withdraw collateral before the challenge is executed. It also risks loss of funds due to user errors such as sending funds to an expired contract. As the contract's withdraw functions employ proper access control, recovery of these lost funds is impossible.

## Proof of Concept

An attack scenario can be demonstrated by setting up a test case with Alice having a position and Bob launching a challenge:
```javascript
    it("send challenge", async () => {
        challengeAmount = 4; //initialCollateralClone/2;
        let fchallengeAmount = floatToDec18(challengeAmount);
        await mockVOL.connect(accounts[0]).approve(mintingHubContract.address, fchallengeAmount);

        console.log("Block timestamp:", (await ethers.provider.getBlock()).timestamp);
        let tx = await mintingHubContract.connect(accounts[1]).launchChallenge(clonePositionAddr, fchallengeAmount);
        console.log(await mintingHubContract.challenges(0));

        await mintingHubContract.splitChallenge(0,floatToDec18(2));

        await expect(tx).to.emit(mintingHubContract, "ChallengeStarted");
    });
```
In this scenario, Alice observes Bob's transaction in the mempool and withdraws collateral. Bob then challenges an empty position, which results in the loss of funds. It also demonstrates that if the `challengeAmount` is set to 0, it can cause the contract to revert due to a division by zero panic code during split.

## Recommended Mitigation Steps

To prevent this, we recommend the following changes:

1. Add a condition within the `launchChallenge` function in the `MintingHub` contract to check for 0 `_collateralAmount`: `require(_collateralAmount != 0, "Collateral must not be 0");`
2. Add the `noCooldown` modifiers to the `initializeClone` and `notifyChallengeStarted` functions to ensure consistent checks on the contract's status.

## Severity: High

Given that this issue may allow for front-running attacks, as well as potential fund loss due to user errors, it is crucial to address this promptly to maintain the integrity of the contract and the safety of the funds interacting with it.

## Mediums

## [M-01] Potential for MEV Front-Running and Griefing due to Reward Distribution Model

## Impact

The current implementation incentivizes Miner Extractable Value (MEV) front-running and introduces opportunities for griefing due to the design of the share exchange and pricing model.

1. The quantity of ZCHF tokens received from exchanging shares decreases as the total number of shares decrease. Thus, a large exchange being front-run could yield profits.
2. The initial 1000 ZCHF tokens create 1000 shares. However, the subsequent 1000 ZCHF tokens result in only about 260 shares.

The above design can potentially spur an MEV competition for the first depositor and enable griefing afterwards. Particularly, griefing can happen when the total shares are below 1000. Regardless of the expected number of shares, a user will only receive enough shares to raise the total to 1000. This situation could allow for significant system disruption.

## Proof of Concept

Let's consider the following scenario with Alice being the first depositor:
```javascript
    describe("exchanges shares & pricing", () => {
        it("redeem 1 share", async () => {
            let amount = 0; // amount we will deposit
            let fAmount = floatToDec18(1000); // amount we will deposit
            await ZCHFContract.connect(accounts[0]).transferAndCall(equityContract.address, fAmount, 0);

            let BLOCK_MIN_HOLDING_DURATION = 90 * 7200;
            await network.provider.send("hardhat_mine", [BNToHexNoPrefix(BLOCK_MIN_HOLDING_DURATION + 1)]);
            let canRedeem = await equityContract.connect(accounts[0])['canRedeem()']();

            let fAmountShares = floatToDec18(1); // redeem 1 share

            await equityContract.redeem(owner, fAmountShares);

            // Deposit another 1000
            fAmount = floatToDec18(1000); // amount we will deposit
            await ZCHFContract.connect(accounts[1]).transferAndCall(equityContract.address, fAmount, 0);

            // Deposit another 1000
            await ZCHFContract.connect(accounts[0]).transferAndCall(equityContract.address, fAmount, 0);

            // Check that bob has only 1 share, although he deposited 1000 ZCHF
            const bobShares = floatToDec18(1);
            const aliceShares = floatToDec18(1100);
            expect(await equityContract.balanceOf(otherUser)).to.equal(bobShares);
            expect(await equityContract.balanceOf(owner)).to.be.gt(1100);
        });
    });
```
In this case, Alice deposits 1000 ZCHF for 1000 shares. She notices that Bob intends to deposit 1000 ZCHF and manages to front-run Bob's transaction by burning 1 share. Consequently, Bob deposits 1000 ZCHF and receives only 1 share. Then, Alice can deposit 1000 ZCHF more and receive over 100 shares, which significantly disrupts the balance of the system.

## Recommended Mitigation Steps

To alleviate these issues, consider adjusting the share calculation mechanism. Specifically, the function `calculateSharesInternal`, which only works when `totalShares < 1000`, could be used for all scenarios. The exact implementation may require more thoughtful design considerations to maintain the balance and fairness of the system.

## Severity: Medium

While this issue might not lead to immediate loss of funds, it can severely disrupt the balance and fairness of the system. Therefore, it is recommended to address this issue promptly to maintain the integrity of the reward distribution model.


## Lows

## [L-01] Residual Dust in the FPS Contract

## Impact

Due to the current design of the FPS contract, residual dust gets locked in the contract. This results from a requirement for at least 2 shares to always remain within the network. For instance, if Alice deposits first and receives 1000 shares, she can only immediately transfer back 998 shares, leaving a residual dust of 2 shares.

You can find the related contract lines here: [GitHub](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L290-L297)

## Recommended Mitigation Steps

Consider revising the contract to allow all shares to be transferred, thereby eliminating the residual dust issue. This means removing the requirement for at least 1 share to always remain within the system.

## Severity: Low

The residual dust issue does not pose a significant risk to the system's overall security or the users' funds. However, addressing it would enhance the overall efficiency and user experience of the system.

## [L-02] Potential Precision Loss in calculateAssignedReserve() Function

## Impact

The `calculateAssignedReserve()` function currently performs a division before multiplication. This ordering can potentially cause precision loss due to rounding errors, even though these are likely to be quite small.

You can find the related contract lines here: [GitHub](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Frankencoin.sol#L204-L213)

## Recommended Mitigation Steps

It is generally advisable to perform multiplication operations before division operations to preserve precision. Consequently, the division of 1000000 should be moved into the if-else statement. This allows multiplication by `currentReserve` before performing the two divisions:
```solidity
    function calculateAssignedReserve(uint256 mintedAmount, uint32 _reservePPM) internal view returns (uint256) {

        uint256 theoreticalReserve = _reservePPM * mintedAmount;
        uint256 currentReserve = balanceOf(address(reserve));
        if (currentReserve < minterReserve()){
            return theoreticalReserve * currentReserve / minterReserve() / 1000000;
        } else {
            return theoreticalReserve / 1000000;
        }
    }
```

## Severity: Low

This issue does not pose a significant risk to the system's security or the users' funds. However, it's a good practice to address potential precision issues, especially in a financial context, to maintain the system's precision and trustworthiness.

## [L-03] Unbounded Iterations in `_cubicRoot()` Function

## Impact

The `_cubicRoot()` function currently uses the Newton-Raphson method to find roots but doesn't provide a bound for the number of iterations it can perform. While the Newton-Raphson method is known for its robustness and quadratic convergence to the root, it can also diverge in certain edge cases, leading to potentially endless iterations.

You can find relevant sources discussing this topic here:
- [MIT Web](https://web.mit.edu/10.001/Web/Course_Notes/NLAE/node6.html)
- [OpenZeppelin Forum](https://forum.openzeppelin.com/t/calculating-roots-in-solidity/1595/5)

## Recommended Mitigation Steps

To prevent potential divergence and unexpected high gas costs, it is recommended to impose a maximum limit on the number of iterations the function can perform. This limit will ensure the function doesn't drain the user's gas in cases where the Newton-Raphson method diverges.

    function _cubicRoot(uint256 x) internal pure returns (uint256 y) {
        // Implementation goes here with a max iteration limit
    }

## Severity: Low

While not posing an immediate threat to the system's security or users' funds, an unbounded iteration could potentially result in undesirable effects such as excessive gas costs. Therefore, it is beneficial to place a limit on the number of iterations.


## Gas
