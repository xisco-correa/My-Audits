# L-01 'MysteryBox::constructor()' has different rewards compared to 'MysteryBox::openBox()'

## Summary

In the `MysteryBox::constructor()` there are defined values for each reward in terms of **eth**, which we receive when opening a box that costs **ether**, purchased by the user. Then, there is a discrepancy in the `MysteryBox::openBox()` function because the value of the rewards is not the same as the rewards defined in the `MysteryBox::constructor()`.

## Vulnerability Details

As you can see in the `MysteryBox::constructor()`, the following reward values are passed:

* GoldCoin 0.5 ether
* SilverCoin 0.25 ether
* BronzeCoin 0.1 ether
* CoalCoin 0 ether

```Solidity
constructor() payable {
    owner = msg.sender;
    boxPrice = 0.1 ether;
    require(msg.value >= SEEDVALUE, "Incorrect ETH sent");
    // Initialize with some default rewards
    rewardPool.push(Reward("Gold Coin", 0.5 ether));
    rewardPool.push(Reward("Silver Coin", 0.25 ether));
    rewardPool.push(Reward("Bronze Coin", 0.1 ether));
    rewardPool.push(Reward("Coal", 0 ether));
}
```

> [Link to the code](https://github.com/xisco-correa/My-Audits/blob/main/audits/MysteryBox/MysteryBox.sol#L18-L27)

The inconsistency occurs in one of the contract functions `MysteryBox::openBox()` when storing rewards in `mapping(address => Reward[]) public rewardsOwned;`, where it assigns a completely different value for the ***SilverCoin*** reward, giving it a new value of **0.25 ether**, and to ***GoldCoin*** a value of **1 ether**.

```Solidity
function openBox() public {
    require(boxesOwned[msg.sender] > 0, "No boxes to open");

    // Generate a random number between 0 and 99
    uint256 randomValue = uint256(keccak256(abi.encodePacked(block.timestamp, msg.sender))) % 100;

    // Determine the reward based on probability
    if (randomValue < 75) {
        // 75% chance to get Coal (0-74)
        rewardsOwned[msg.sender].push(Reward("Coal", 0 ether));
    } else if (randomValue < 95) {
        // 20% chance to get Bronze Coin (75-94)
        rewardsOwned[msg.sender].push(Reward("Bronze Coin", 0.1 ether));
    } else if (randomValue < 99) {
        // 4% chance to get Silver Coin (95-98)
        rewardsOwned[msg.sender].push(Reward("Silver Coin", 0.5 ether));
    } else {
        // 1% chance to get Gold Coin (99)
        rewardsOwned[msg.sender].push(Reward("Gold Coin", 1 ether));
    }

    boxesOwned[msg.sender] -= 1;
}
```

[Link to the code](https://github.com/xisco-correa/My-Audits/blob/f0ee83e978e585f4177353aed096dbbde9ab6e78/audits/MysteryBox/MysteryBox.sol#L44-L66)

## Impact

This misunderstanding can cause users to have an incorrect perception of what the real rewards of the `MysteryBox` contract are, or the contract owner may be distributing unintended rewards, which can result in the loss of funds. Another possible issue is the lack of data updates. If a user changes the constructor's values, these would not be updated in the `MysteryBox::openBox()` function, leaving the constructor's parameters that reference the rewards in an inconsistent state.

## Tools Used

Manual Review.

## Recommendations

To keep `MysteryBox::constructor()` as it is written and as the only necessary place to update the reward values, the function `MysteryBox::openBox` should be written as follows:

```Solidity
function openBox() public {
    require(boxesOwned[msg.sender] > 0, "No boxes to open");

    // Generate a random number between 0 and 99
    uint256 randomValue = uint256(keccak256(abi.encodePacked(block.timestamp, msg.sender))) % 100;

    // Determine the reward based on probability
    if (randomValue < 75) {
        // 75% chance to get Coal (0-74)
        rewardsOwned[msg.sender].push(rewardPool[3]);
    } else if (randomValue < 95) {
        // 20% chance to get Bronze Coin (75-94)
        rewardsOwned[msg.sender].push(rewardPool[2]);
    } else if (randomValue < 99) {
        // 4% chance to get Silver Coin (95-98)
        rewardsOwned[msg.sender].push(rewardPool[1]);
    } else {
        // 1% chance to get Gold Coin (99)
        rewardsOwned[msg.sender].push(rewardPool[0]);
    }

    boxesOwned[msg.sender] -= 1;
}
```

This way, in `mapping(address => Reward[]) public rewardsOwned;`, which collects the rewards that the user is receiving, it introduces the value of `Reward[] public rewardPool;` pointing to the index of the corresponding reward as stored in the `MysteryBox::constructor()`.

* Position 0 of the array index `Reward[] public rewardPool;` for ***GoldCoin***
* Position 1 of the array index `Reward[] public rewardPool;` for ***Silver Coin***
* Position 2 of the array index `Reward[] public rewardPool;` for ***Bronze Coin***
* Position 3 of the array index `Reward[] public rewardPool;` for ***Coal***

```Solidity
rewardPool.push(Reward("Gold Coin", 0.5 ether));
rewardPool.push(Reward("Silver Coin", 0.25 ether));
rewardPool.push(Reward("Bronze Coin", 0.1 ether));
rewardPool.push(Reward("Coal", 0 ether));
```

## Checking the recommendation

If you want to verify that the rewards have been correctly set in your contract, you can add this function to `MysteryBox`:

```Solidity
function viewRewardsAvailable(uint256 _indexReward) public view returns (string memory, uint256) {
    require(_indexReward < rewardPool.length, "there is no more prizes");
    Reward memory availableReward = rewardPool[_indexReward];
    return (availableReward.name, availableReward.value);
}
```

This function, when given an index parameter, will return the name and the assigned reward in wei. The information is retrieved from `Reward[] public rewardPool;`.
