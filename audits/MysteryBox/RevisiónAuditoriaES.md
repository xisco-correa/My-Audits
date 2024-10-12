# L-01`MysteryBox::constructor()` tiene diferentes recompensas que la función `MysteryBox::openBox()`

## Resumen

En el  `MysteryBox::constructor()` hay unos valores definidos para cada premio en valor  de eth, que recibimos al abrir una caja que cuesta ether que el usuario compra.  Después hay discrepancia en la función `MysteryBox::openBox()`  porque el valor de los premios no son los mismos que los premios definidos en `MysteryBox::constructor()` .

## Detalles de Vulnerabilidad

Como puedes ver en el `MysteryBox::constructor()` se pasan los valores de:

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

> [Link al código](https://github.com/xisco-correa/My-Audits/blob/main/audits/MysteryBox/MysteryBox.sol#L18-L27)

La incorcondancia ocurre en una de las funciones del contrato `MysteryBox::openBox()` cuando, al guardar los premios en `mapping(address => Reward[]) public rewardsOwned;` , asigna un valor totalmente diferente para el premio ***SilverCoin*** ,dándole un nuevo valor de **0.25 ether** y a ***GoldCoin*** un  valor de **1 ether**.

```Solidity
Solidity
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

[Link del código](https://github.com/xisco-correa/My-Audits/blob/f0ee83e978e585f4177353aed096dbbde9ab6e78/audits/MysteryBox/MysteryBox.sol#L44-L66)

## Impacto

Este malentendido puede hacer que los usuarios tenga una percepción errónea de cuales son los premios reales del contrato `MysteryBox` , o que el dueño del contrato este repartiendo  premios no deseados , lo que pueden llevar a cabo la pérdida de fondos.
Otro posible problema es una falta de actualización de los datos si un usuario cambia los valores del constructor, estos no se actualizarían en la función `MysteryBox::openBox()` lo que hace que los parámetros del constructor que hacen referencia a los premios queden en un estado inconsciente.

## Tools Used

Revisión Manual.

## Recomendaciones

Para que  `MysteryBox::constructor()` se pueda dejar como está escrito y será el único lugar necesario donde actualizar las recompensas de los premios siempre que en la función \`MusteryBox::openBox se escriba de la siguiente manera.

```Solidity
Solidity
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

De este manera a `mapping(address => Reward[]) public rewardsOwned;`, que recopila las recompensas que el usuario va obteniendo ,se le introduce el valor `Reward[] public rewardPool;` apuntando al índice del premio correspondiente según se han guardado en el `MysteryBox::constructor()`.

* Posición 0 del índice del array `Reward[] public rewardPool;` para ***GoldCoin***
* Posición 1 del índice del array `Reward[] public rewardPool;` para ***Silver Coin***
* Posición 2 del índice del array `Reward[] public rewardPool;` para ***Bronze Coin***
* Posición 2 del índice del array `Reward[] public rewardPool;` para ***Coal***

```Solidity
Solidity
rewardPool.push(Reward("Gold Coin", 0.5 ether));
        rewardPool.push(Reward("Silver Coin", 0.25 ether));
        rewardPool.push(Reward("Bronze Coin", 0.1 ether));
        rewardPool.push(Reward("Coal", 0 ether));
```

## Comprobación de las recomendaciones

Si quieres verificar en tu contrato que los precios se han configurado de manera correcta puedes añadir esta función a `MysteryBox`:
```
Solidity
function viewRewardsAvaiable(uint256 _indexReward)public view returns (string memory, uint256){
        require(_indexReward < rewardPool.length, "there is no more prizes");
        Reward memory avaiableReward = rewardPool[_indexReward];
        return (avaiableReward.name, avaiableReward.value);
    }
```
Esta función cuando le introduzcas el parámetro índice te devolverá el nombre y el premio asignado en wei, la información la obtiene de `Reward[] public rewardPool;`
***
