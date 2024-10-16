## Summary

En el contrato `PasswordStore` la variable `s_password`  se encuentera en estado de dafault. Solo una vez llamada la función `PasswordStore::setNewPassword`, esta adquiere un estado. Esto provoca que durante un periodo de tiempo sea vulnerable.

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

## Impact

El problema ocurre que en este tiempo mientras la variable `s_password` no está definida, el atacante podría acceder donde se requisiera esta contraseña solo pasando el valor **""** porque \`s\_password == ""

## Tools Used

Revisión Manual.

## Recommendations

## Checking the recommendation

```Solidity
```
