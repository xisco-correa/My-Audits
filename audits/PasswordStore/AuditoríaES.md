# La variable `string private s_password`se encuentra en estado de default


## Summary

En el contrato `PasswordStore` la variable `s_password`  se encuentera en estado de dafault. Solo una vez llamada la función `PasswordStore::setNewPassword`, esta adquiere un estado. Esto provoca que durante un periodo de tiempo sea vulnerable.

> [Link Código](https://github.com/xisco-correa/My-Audits/blob/main/audits/PasswordStore/PasswordStore.sol#L14-L28)

## Vulnerability Details

Como puedes ver en el contrato `PasswordStore`, la variable no está definida en el constructor:

```Solidity
string private s_password;

    event SetNetPassword();

    constructor() {
        s_owner = msg.sender;
    }
```

Solo adquiere un valor cuando se llama a la función `PasswordStore::setNewPassword`:

```Solidity
function setPassword(string memory newPassword) external {
s\_password = newPassword;
emit SetNetPassword();
}
```
Entonces un usuario antes de que se le de un valor a la variable `string private s_password`, sabe la contraseña que es ***""***


## Impact

El problema ocurre que en este tiempo mientras la variable `s_password` no está definida, el atacante podría acceder donde se requisiera esta contraseña solo pasando el valor ***""***, porque `s_password == "" `hasta que se define en `PasswordStore::setNewPassword`.

```Solidity
function boxLockedWithPassword (string memory _enter_password) external {
        
        require (keccak256(abi.encode(_enter_password)) == keccak256(abi.encode(s_password)), "Incorrect Password");
        emit youHaveAcces();

    }
```
## PoC

Añadimos al contrato la función `PasswordStore::boxLockedWithPassword()` para poder comprobar mejor el error, ya que nos permite, ver si la variable está en estado de default con esta lógica `s_password == "" `. Entonces para desbloquear la función requerirá poner la contaseña que debería ser la variable default **""**.

```Solidity
function boxLockedWithPassword (string memory _enter_password) external {
        
        require (keccak256(abi.encode(_enter_password)) == keccak256(abi.encode(s_password)), "Incorrect Password");
        emit youHaveAcces();

    }
```
Y ejecutamos el test para comprobar si user dos llamando a la función `PasswordStore::boxLockedWithPassword()`  con el valor de entrada **""** puede acceder. Si puede acceder nos demostraŕa que hay una vulnerabilidad.

```Solidity
 function setUp() public{    
        user1 = address(1);
        user2 = address(2);
        
        vm.prank(user1);
        passwordStore = new PasswordStore();
        
    }

    function testBoxLocked()public{
        vm.startPrank(user2);
        passwordStore.boxLockedWithPassword("");
        vm.stopPrank();

    }
```

## Tools Used

Revisión Manual.

## Recommendations

Con requerir que en el constructor ya se asigne una contraseña, haces que la variable de estado `string private s_password`nunca se encuentre en estado de default. 

```Solidity
constructor(string memory antiDefault_password) {
        
        s_password = antiDefault_password;
        s_owner = msg.sender;
    }
```
