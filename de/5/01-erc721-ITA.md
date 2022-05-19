---
title: I token su Ethereum
actions:
  - 'Verifica la risposta'
  - 'hints'
requireLogin: true
material:
  editor:
    language: sol
    startingCode:
      "zombieownership.sol": |
        // Inizia qui 
      "zombieattack.sol": |
        pragma solidity ^0.4.19;
        
        import "./zombiehelper.sol";
        
        contract ZombieAttack is ZombieHelper {
        uint randNonce = 0;
        uint attackVictoryProbability = 70;
        
        function randMod(uint _modulus) internal returns(uint) {
        randNonce++;
        return uint(keccak256(now, msg.sender, randNonce)) % _modulus;
        }
        
        function attack(uint _zombieId, uint _targetId) external ownerOf(_zombieId) {
        Zombie storage myZombie = zombies[_zombieId];
        Zombie storage enemyZombie = zombies[_targetId];
        uint rand = randMod(100);
        if (rand <= attackVictoryProbability) {
        myZombie.winCount++;
        myZombie.level++;
        enemyZombie.lossCount++;
        feedAndMultiply(_zombieId, enemyZombie.dna, "zombie");
        } else {
        myZombie.lossCount++;
        enemyZombie.winCount++;
        _triggerCooldown(myZombie);
        }
        }
        }
      "zombiehelper.sol": |
        pragma solidity ^0.4.19;
        
        import "./zombiefeeding.sol";
        
        contract ZombieHelper is ZombieFeeding {
        
        uint levelUpFee = 0.001 ether;
        
        modifier aboveLevel(uint _level, uint _zombieId) {
        require(zombies[_zombieId].level >= _level);
        _;
        }
        
        function withdraw() external onlyOwner {
        owner.transfer(this.balance);
        }
        
        function setLevelUpFee(uint _fee) external onlyOwner {
        levelUpFee = _fee;
        }
        
        function levelUp(uint _zombieId) external payable {
        require(msg.value == levelUpFee);
        zombies[_zombieId].level++;
        }
        
        function changeName(uint _zombieId, string _newName) external aboveLevel(2, _zombieId) ownerOf(_zombieId) {
        zombies[_zombieId].name = _newName;
        }
        
        function changeDna(uint _zombieId, uint _newDna) external aboveLevel(20, _zombieId) ownerOf(_zombieId) {
        zombies[_zombieId].dna = _newDna;
        }
        
        function getZombiesByOwner(address _owner) external view returns(uint[]) {
        uint[] memory result = new uint[](ownerZombieCount[_owner]);
        uint counter = 0;
        for (uint i = 0; i < zombies.length; i++) {
        if (zombieToOwner[i] == _owner) {
        result[counter] = i;
        counter++;
        }
        }
        return result;
        }
        
        }
      "zombiefeeding.sol": |
        pragma solidity ^0.4.19;
        
        import "./zombiefactory.sol";
        
        contract KittyInterface {
        function getKitty(uint256 _id) external view returns (
        bool isGestating,
        bool isReady,
        uint256 cooldownIndex,
        uint256 nextActionAt,
        uint256 siringWithId,
        uint256 birthTime,
        uint256 matronId,
        uint256 sireId,
        uint256 generation,
        uint256 genes
        );
        }
        
        contract ZombieFeeding is ZombieFactory {
        
        KittyInterface kittyContract;
        
        modifier ownerOf(uint _zombieId) {
        require(msg.sender == zombieToOwner[_zombieId]);
        _;
        }
        
        function setKittyContractAddress(address _address) external onlyOwner {
        kittyContract = KittyInterface(_address);
        }
        
        function _triggerCooldown(Zombie storage _zombie) internal {
        _zombie.readyTime = uint32(now + cooldownTime);
        }
        
        function _isReady(Zombie storage _zombie) internal view returns (bool) {
        return (_zombie.readyTime <= now);
        }
        
        function feedAndMultiply(uint _zombieId, uint _targetDna, string _species) internal ownerOf(_zombieId) {
        Zombie storage myZombie = zombies[_zombieId];
        require(_isReady(myZombie));
        _targetDna = _targetDna % dnaModulus;
        uint newDna = (myZombie.dna + _targetDna) / 2;
        if (keccak256(_species) == keccak256("kitty")) {
        newDna = newDna - newDna % 100 + 99;
        }
        _createZombie("NoName", newDna);
        _triggerCooldown(myZombie);
        }
        
        function feedOnKitty(uint _zombieId, uint _kittyId) public {
        uint kittyDna;
        (,,,,,,,,,kittyDna) = kittyContract.getKitty(_kittyId);
        feedAndMultiply(_zombieId, kittyDna, "kitty");
        }
        }
      "zombiefactory.sol": |
        pragma solidity ^0.4.19;
        
        import "./ownable.sol";
        
        contract ZombieFactory is Ownable {
        
        event NewZombie(uint zombieId, string name, uint dna);
        
        uint dnaDigits = 16;
        uint dnaModulus = 10 ** dnaDigits;
        uint cooldownTime = 1 days;
        
        struct Zombie {
        string name;
        uint dna;
        uint32 level;
        uint32 readyTime;
        uint16 winCount;
        uint16 lossCount;
        }
        
        Zombie[] public zombies;
        
        mapping (uint => address) public zombieToOwner;
        mapping (address => uint) ownerZombieCount;
        
        function _createZombie(string _name, uint _dna) internal {
        uint id = zombies.push(Zombie(_name, _dna, 1, uint32(now + cooldownTime), 0, 0)) - 1;
        zombieToOwner[id] = msg.sender;
        ownerZombieCount[msg.sender]++;
        NewZombie(id, _name, _dna);
        }
        
        function _generateRandomDna(string _str) private view returns (uint) {
        uint rand = uint(keccak256(_str));
        return rand % dnaModulus;
        }
        
        function createRandomZombie(string _name) public {
        require(ownerZombieCount[msg.sender] == 0);
        uint randDna = _generateRandomDna(_name);
        randDna = randDna - randDna % 100;
        _createZombie(_name, randDna);
        }
        
        }
      "ownable.sol": |
        /**
        * @title Ownable
        * @dev The Ownable contract has an owner address, and provides basic authorization control
        * functions, this simplifies the implementation of "user permissions".
        */
        contract Ownable {
        address public owner;
        
        event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
        
        /**
        * @dev The Ownable constructor sets the original `owner` of the contract to the sender
        * account.
        */
        function Ownable() public {
        owner = msg.sender;
        }
        
        
        /**
        * @dev Throws if called by any account other than the owner.
        */
        modifier onlyOwner() {
        require(msg.sender == owner);
        _;
        }
        
        
        /**
        * @dev Allows the current owner to transfer control of the contract to a newOwner.
        * @param newOwner The address to transfer ownership to.
        */
        function transferOwnership(address newOwner) public onlyOwner {
        require(newOwner != address(0));
        OwnershipTransferred(owner, newOwner);
        owner = newOwner;
        }
        
        }
    answer: |
      pragma solidity ^0.4.19;
      
      import "./zombieattack.sol";
      
      contract ZombieOwnership is ZombieAttack {
      
      }
---
Parliamo di ***tokens***.

Se conosci Ethereum da un pò di tempo, probabilmente hai già sentito parlare di token - principalmente i ***token ERC20***.

Un ***token*** su blockchain Ethereum è essezialmente uno smart contract che segue regole comuni — in particolare, implementa un insieme di funzioni standard che tutti gli altri token condividono, come ad esempio `transfer(address _to, uint256 _value)` e `balanceOf(address _owner)`.

Internamente lo smart contract contiene una mappatura, `mapping(address => uint256) balances`, che tiene traccia del saldo di ciascun indirizzo.

In pratica, un token è un contratto che tiene traccia di chi possiede una certa quantità di token e di alcune funzioni che consentono agli utenti di trasferire i loro token ad altri indirizzi.

### Perché è importante?

Poiché tutti i token ERC20 condividono lo stesso insieme di funzioni con gli stessi nomi, è possibile interagire con essi nello stesso modo.

Ciò significa che se si costruisce un'applicazione in grado di interagire con un token ERC20, questa è anche in grado di interagire con qualsiasi altro token ERC20. In questo modo è possibile aggiungere facilmente altri token alla vostra applicazione, senza bisogno di una codifica personalizzata. È sufficiente inserire l'indirizzo del nuovo contratto del token e la vostra applicazione avrà un altro token da utilizzare.

Prendiamo per esempio un exchange. Quando un exchange aggiunge un nuovo token ERC20, in realtà deve solo aggiungere un altro smart contract con cui dialogare. Gli utenti possono dire al contratto di inviare i token all'indirizzo del portafoglio dell'exchange e l'exchange può dire al contratto di rimandare i token agli utenti quando richiedono un prelievo.

L'exchange ha bisogno di implementare questa logica di trasferimento solo una volta, quando vuole aggiungere un nuovo token ERC20, dovrà semplicemente aggiungere l'indirizzo del nuovo contratto al suo database.

### Esistono diversi standard di token

I token ERC20 sono molto utili quando devono compiere ruoli come quello di una valuta. Ma non sono particolarmente utili per rappresentare gli zombie nel nostro gioco di zombie.

Per prima cosa, gli zombie non sono divisibili come le valute: posso inviarti 0.237 ETH, ma trasferirvi 0.237 di uno zombie. Non avrebbe davero senso! 

Secondo, tutti gli zombie non sono creati equamente. Il tuo zombie "**Steve**" di Livello 2 ha caratteristiche e attributi differenti rispetto al mio di Livello 732 "**H4XF13LD MORRIS**".</p> 

C'è un altro standard di token che si adatta molto meglio ai crypto-collectibles come i CryptoZombie — vengono chiamati ***token ERC721.***

***I token ERC721 *** non **sono** sostituibili poiché si presume che ognuno di essi sia unico e inoltre non sono divisibili. Si possono scambiare solo in unità intere e ognuna ha un ID unico. Sono quindi perfetti per rendere commerciabili i nostri crypto zombie.

> Si noti che l'utilizzo di un token standard come ERC721 ha il vantaggio di non dover implementare la logica di asta o di deposito a garanzia all'interno del nostro contratto che determina come i giocatori possono scambiare/vendere i nostri zombie. Se ci conformiamo alle specifiche, qualcun altro potrebbe costruire una piattaforma di scambio per asset ERC721 negoziabili, e i nostri zombie ERC721 sarebbero utilizzabili su quella piattaforma. Ci sono quindi evidenti vantaggi nell'utilizzare uno standard di token invece di creare una propria logica di scambio.

## Mettiti alla prova

Nel prossimo capitolo approfondiremo l'implementazione dell'ERC721. Ma prima, impostiamo la struttura dei file per questa lezione.

Memorizzeremo la logica ERC721 in un contratto chiamato `ZombieOwnership`.

1. Dichiara la versione `pragma` all' inizio del file (se lo necessiti, controlla i file delle precedenti lezioni per la sintassi).

2. Questo file dovrebbe contenere `import` da`zombieattack.sol`.

3. Dichiara un nuovo contratto, chiamato `ZombieOwnership`, che eredita da `ZombieAttack`. Lascia il corpo del contratto vuoto per ora.
