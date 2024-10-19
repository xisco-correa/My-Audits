# 01-.La variable `string private s_password` se encuentra en estado de default


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

***
# 02-.La función `PasswordStore::setNewPassword()` le falta requerimiento de owner

## Summary
Según la lógica del contrato, solo el owner puede configurar la contraseña. El problema es que en`PasswordStore::setNewPassword()`, cualquiera puede llamar a esta función para configurarla.
> [Link al código](https://github.com/xisco-correa/My-Audits/blob/main/audits/PasswordStore/PasswordStore.sol#L26-L29)

## Vulnerability Details
En la función `PasswordStore::setNewPassword()`: 

```Solidity
 function setPassword(string memory newPassword) external {
        s_password = newPassword;
        emit SetNetPassword();
    }
```
Cualquier usuario llamando a esta función, configura la variable de estado `string private s_password;`


## Impact

Esto no debería ser así porque le contrato pierde la lógica donde solo el owner puede configurar una nueva contraseña, ya que si este guarda su contraseña, pero seguidamente otro usuario llama a esta función, la contraseña del owner será modificada.

## PoC

Ejecutamos una prueba test, donde el `user(2)` llama a la función sin ser el ***owner*** y consigue modificar la variable de estado `string private s_password;`

```Solidity
function setUp() public{    
        user1 = address(1);
        user2 = address(2);
        
        vm.prank(user1);
        passwordStore = new PasswordStore();
        
    }

    function testRandomUserCanSetPassword()public{
        vm.startPrank(user2);
        passwordStore.setPassword("hola");
        vm.stopPrank();

    }
```
 
## Recommended Mitigation

Importar la librería `import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";`, de esta manera las funciones que requiran de permiso ***owner***, se podrá poner un modifier que será `onlyOwner`. Una vez hecho esto la función solo se podrá llamar por el `address` que ha desplegado el contrato.

```Solidity
constructor() Ownable(msg.sender){
    }

    /*
     * @notice This function allows only the owner to set a new password.
     * @param newPassword The new password to set.
     */
    function setPassword(string memory newPassword) external onlyOwner {
        s_password = newPassword;
        emit SetNetPassword();
    }
```
Como puedes ver en el `PasswordStore::constructor()` pasa el `msg.sender`al constructor de `Ownable::constructor()`, para que en la función `PasswordStore::setPassword`  el modifier `onlyOwner` haga la función de solo poder ser llamado por el propietario.



## Tools Used

Revisión Manual.

***

# Título:
La variable `string private s_password;` no es privada

## Summary: 

Lo que ocurre es que en el contrato `PasswordStore` el owner puede configurar su contraseña en la función `PasswordStore:setPassword`  y despúes esta variable de estado se guarda en `string private s_password;` el problema, es que en la blockchain toda la información es accesible.

> [LinkCódigo](https://github.com/xisco-correa/My-Audits/blob/main/audits/PasswordStore/PasswordStore.sol#L14) 

## Vulnerability Details: 

El error del código lo encontramos donde se guarda la contraseña en la variable de estado:

```Solidity
string private s_password;

function setPassword(string memory newPassword) external {
        s_password = newPassword;
        emit SetNetPassword();
    }
```
El problema ocurre porque las variables de estado se pueden consultar en la blockchain. Conociendo el slot de la variable se puede apuntar al contrato para sacar la cadena de bytes32 que guarda esta información y asi extraer la contraseña.

La manera para extraer esta información se puede hacer con las siguientes dos funciones. La primera función permite sacar la longitud de bytes de la contraseña, esto es necesario para despúes saber que longitud tiene el array de Bytes. En la segunda función copiamos la cadena de bytes32 donde está la contraseña dentro del array de bytes personalizado para despúes poder convertirlo a una variable ***string***.

## Impact: 

Nadie debería guardar una información privada en la blockchain. Ya que esta puede ser solicitada.

## PoC: 
Si ejecutas estas funciones en un entorno test en foundry con las herramientas de forge-std verás como se puede exploitear:
```Solidity
function GetNumberOfBytesPassword() public returns (uint256) {
        uint256 slot = 2;  // Slot de almacenamiento para el string (ajustado según el contrato PasswordStore)
        bytes32 passwordInBytes = vm.load(address(passwordStore), bytes32(slot));
        
        uint i = 0;
        uint256 contadorDeBytes = 0;
        while(i<32 && passwordInBytes[i]!=0){
            i++;
            contadorDeBytes++;
        
        }
        return contadorDeBytes;
    }

    function testGetPassword() public returns (string memory){
        uint256 slot = 2;  // Slot de almacenamiento para el string (ajustado según el contrato PasswordStore)
        bytes32 passwordInBytes = vm.load(address(passwordStore), bytes32(slot));
        uint256 nuevoContadorDeBytes = 0;
        uint256 i = 0;
        bytes memory arrayDeBytes = new bytes(11);

        while( i<32 && nuevoContadorDeBytes < arrayDeBytes.length){
            bytes32 passwordInBytes = vm.load(address(passwordStore), bytes32(slot));
            for(i = 0; i<32 && nuevoContadorDeBytes < arrayDeBytes.length; i++){
                
                arrayDeBytes[nuevoContadorDeBytes] = passwordInBytes[i];
                nuevoContadorDeBytes++;
            }
            
            return string(arrayDeBytes);

        }



    }

```

## Tools Used: 

Revisión Manual

## Recommendations: 

No hay una manera de arreglar que una variable sea oculta para todos, esto es por como funciona la blockchain.
