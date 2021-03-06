Manage an Ethereum account with Web3j
========

The Ethereum blockchain is often compared to a World Computer with a global state. The global state grows after each new block and is composed of many accounts organised in a [Merkle tree](https://en.wikipedia.org/wiki/Merkle_tree).

![](https://imgur.com/iQLdaOW.png)

Each account has a state composed of information such as balance, nonce, storageRoot and codeHash, and is identified by a 20 bytes address (for example: `0x66aac71c0c81ec00aebead84914a10e307a4cbf9`).

There are two types of accounts:
- Externally owned accounts, which are controlled by private keys and have no code associated with them.
- Contract accounts, which are controlled by their contract code and have code associated with them.

![](https://imgur.com/3dlka35.png)


In this tutorial, we will focus on externally owned accounts to retrieve information such as a balance, create or open an account and send transactions to another account using the Java library Web3j.

<br />

## 1. Retrieve public information about an account

The Ethereum blockchain is a public shared ledger which can be queried to retrieve information about the state at a different time (block #).


### Get account's balance

Every accounts have a balance of the Ethereum native cryptocurrency called **Ether**. Using our Web3j instance (see article-1), it is possible to retrieve the balance of an account at a given block using the function `web3.ethGetBalance(<accountAddress>, <blockNo>).send()`

<br />

The balance is stored by default in the smallest denomination of ether called *wei* (1 ether = 10^18 wei) but Web3j provides a convenient utility class `Convert` to convert values between different units.

<br />

- Retrieve the latest balance (latest block) of an account:

```java
EthGetBalance balanceWei = web3.ethGetBalance("0xF0f15Cedc719B5A55470877B0710d5c7816916b1", DefaultBlockParameterName.LATEST).send();

BigDecimal balanceInEther = Convert.fromWei(balanceWei.getBalance().toString(), Unit.ETHER);
```

![](https://imgur.com/S7w0eEH.png)

The latest balance of the account `0xF0f15Cedc719B5A55470877B0710d5c7816916b1` is *33.25 ether*.


<br />

- Retrieve the balance of an account at a specific block:

```java
EthGetBalance balance = web3.ethGetBalance("0xF0f15Cedc719B5A55470877B0710d5c7816916b1", new DefaultBlockParameterNumber(3000000)).send();

BigDecimal balanceInEther = Convert.fromWei(balance.getBalance().toString(), Unit.ETHER);
```

![](https://imgur.com/PuUtKHV.png)

The balance at block #3,000,000 of the account `0xF0f15Cedc719B5A55470877B0710d5c7816916b1` is *8.12 ethers*.

<br />

### Get account's nonce

Another information included in the state of an account is the *nonce*, a sequence number symbolizing the number of transactions performed by an account.

Web3j offers the method `web3.ethGetTransactionCount(<accountAddress>, <blockNo>).send()` to retrieve the nonce at a given block number.

```java
EthGetTransactionCount ethGetTransactionCount = web3.ethGetTransactionCount("0xF0f15Cedc719B5A55470877B0710d5c7816916b1", DefaultBlockParameterName.LATEST).send();

BigInteger nonce =  ethGetTransactionCount.getTransactionCount();
```

![](https://imgur.com/uJ2bcNk.png)



<br />

## 2. Open or create an account

In order to control an externally owned account and the fund allocated on it, it is required to own the 32 bytes **Private Key** associated to this account. A private key is a very confidential information so it usually doesn't come in clear text like `3a1076bf45ab87712ad64ccb3b10217737f7faacbf2872e88fdd9a537d8fe266` but is secured and encrypted in a wallet. There is many form of wallets (more or less secured and practical):

![](https://imgur.com/N74l0TI.png)      ![](https://imgur.com/m4JjJsM.png)      ![](https://imgur.com/X8mANUY.png)


In this section, we will learn how to load an existing wallet and create a new one with Web3j to instanciate a *Credentials* object which can be used to sign and send transaction securely on the Ethereum blockchain.

<br />

### Load a wallet

#### From a JSON encryted keystore

The first form of wallet is the JSON encryted keystore, it is a password-encrypted version of the private key. This is the most standard way used by clients such as [Pantheon](https://pegasys.tech/) or [Geth](https://geth.ethereum.org/) but also by Online tool like [MyEtherWallet](https://www.myetherwallet.com/) to secure a private key from potential attackers.

Web3j provides a utility class called `WalletUtils` to load a wallet into a `Credentials` object (wrapper containing the account address and the keypair).

```java
String walletPassword = "secr3t";
String walletDirectory = "/path/to/wallets";
String walletName = "UTC--2019-06-20T08-55-56.200000000Z--fd7d68e16ef61868f3e325fafdf2fc1ec0b77649.json";

// Load the JSON encryted wallet
Credentials credentials = WalletUtils.loadCredentials(walletPassword, walletDirectory + "/" + walletName);

// Get the account address
String accountAddress = credentials.getAddress();

// Get the unencrypted private key into hexadecimal
String privateKey = credentials.getEcKeyPair().getPrivateKey().toString(16);
```

![](https://imgur.com/p92p616.png)

<br />

#### From a Mnemonic phrase

Another common form of private key you might have heard of is the **Mnemonic sentence** (or seed phrase) which converts the 32 bytes key to a group of 12 easy to remember words. For example: `candy maple cake sugar pudding cream honey rich smooth crumble sweet treat`. This form has been established by Bitcoin under the proposal [BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki).

A mnemonic controls multiple private keys because of a mechanism to derive deterministically the mnemonic from a path.

The mnemonic can optionally be encrypted with a password.


```java
String password = null; // no encryption
String mnemonic = "candy maple cake sugar pudding cream honey rich smooth crumble sweet treat";

Credentials credentials = WalletUtils.loadBip39Credentials(password, mnemonic);
```

![](https://imgur.com/xN2Ruaj.png)

By default, Web3j uses a derivation path equals to `m/44'/60'/0'/1`. However, it is possible to open another account on a different path:

```java
String password = null; // no encryption
String mnemonic = "candy maple cake sugar pudding cream honey rich smooth crumble sweet treat";

//Derivation path wanted: // m/44'/60'/0'/0
int[] derivationPath = {44 | Bip32ECKeyPair.HARDENED_BIT, 60 | Bip32ECKeyPair.HARDENED_BIT, 0 | Bip32ECKeyPair.HARDENED_BIT, 0,0};

// Generate a BIP32 master keypair from the mnemonic phrase
Bip32ECKeyPair masterKeypair = Bip32ECKeyPair.generateKeyPair(MnemonicUtils.generateSeed(mnemonic, password));

// Derived the key using the derivation path
Bip32ECKeyPair  derivedKeyPair = Bip32ECKeyPair.deriveKeyPair(masterKeypair, derivationPath);

// Load the wallet for the derived key
Credentials credentials = Credentials.create(derivedKeyPair);
```

![](https://imgur.com/eEgEdOY.png)

<br />

#### From a Private key

As mentionned before, a private key is a 32 bytes long number. To parse a private key with Web3j, only need to pass the private key to the class `Credentials`.

```java
String pk = "c87509a1c067bbde78beb793e6fa76530b6382a4c0241e5e4a9ec0a0f44dc0d3";

Credentials credentials = Credentials.create(pk);
```

![](https://imgur.com/svlvLnF.png)


<br />

### Create a wallet

Finally, if you don't already have an account and want to create a new one from scratch. Web3j's `WalletUtils` offers a simple method to create a JSON encryptd keystore.

```java
String walletPassword = "secr3t";
String walletDirectory = "/path/to/destination/";

String walletName = WalletUtils.generateNewWalletFile(password, new File(directory));
System.out.println("wallet location: " + directory + "/" + walletName);


Credentials credentials = WalletUtils.loadCredentials(password, directory + "/" + walletName);

String accountAddress = credentials.getAddress();
System.out.println("Account address: " + credentials.getAddress());
```

![](https://imgur.com/kbcemsH.png)

<br />

## 3. Send a transaction

Now we have learnt how to retrieve public information (state) like the balance from an account and how to open an account using different methods, we can send a transaction to another account.

A transaction on the Ethereum blockchain is composed of several information:
- **nonce:** a count of the number of transaction sent by the sender
- **gasPrice (in wei):** the amount the sender is willing to pay per unit of gas required to execute the transaction.
- **gasLimit:** the maximum amount of gas the sender is willing to pay to execute this transaction.
- **to:** The address of the recipient account
- **value (in wei):** the amount of Wei to be transferred from the sender to the recipient. In a contract-creating transaction, this value serves as the starting balance within the newly created contract account.
- **signature:** Cryptographic signature that identified the sender of the transaction (from)
- **data:** Optional field used to communicate with a smart contract (encoded string including the function name and the parameters)

There is two ways to send a transaction to the blockchain:
- **Via the Ethereum node:**
This involves sending a non-signed transaction to the Ethereum client having the account *unlocked*.
***I personnaly don't recommend this method which might put your account at risk if the Ethereum node isn't correctly protected***

- **Offline transaction:**
The concept here is to first construct the transaction object `rawTransaction` and sign it with your private key (Web3j Credential). Secondly send it to the Ethereum node via the JSON-RPC API to be propagated across the network.

Once a transaction is broadcast to the network, a transaction hash is returned to the client as a ticket but the transaction isn't performed yet. A set of miners/validators present on the network will pick up all the pending transactions, group them into the next block and agree on the validity. Once verified, the transaction is mined into the new block. At this point, the client can claim a transaction receipt by transaction hash to aknowledge the good execution of his transaction.

![](https://web3j.readthedocs.io/en/latest/_images/web3j_transaction.png)

<br />

### Send fund from one account to another

#### 1. Load an account and get the nonce

As explained in the previous sections, we need to load an account from one the methods and retrieve the nonce value of this account:

```java
String walletPassword = "secr3t";
String walletPath = "/path/to/wallet/UTC--2019-06-20T11-41-39.478000000Z--256c75c85f9c27ac5b2a22f085d9643f7ed91dc1.json";

// Decrypte and open the wallet into a Credential object
Credentials credentials = WalletUtils.loadCredentials(walletPassword, walletPath);etBalance().toString(), Unit.ETHER));

// Get nonce
EthGetTransactionCount ethGetTransactionCount = web3.ethGetTransactionCount(credentials.getAddress(), DefaultBlockParameterName.LATEST).send();
BigInteger nonce =  ethGetTransactionCount.getTransactionCount();
```

#### 2. Configure recipient account and amount to send

In the next step, we are just going to configure the amount (in Wei) to send to a recipient account.

```java
// Recipient account
String recipientAddress = "0xDD6325C45aE6fAbD028D19fa1539663Df14813a8";

// Value to Transfer
BigInteger value = Convert.toWei("1", Unit.ETHER).toBigInteger();
```

#### 3. Configure Gas parameters

Gas represents the fees of the network which will be taken by the miner who mines the block which includes your transaction.

When sending a transaction, two parameters are important:
- **Gas Limit (in unit):** Gas limit refers to the maximum amount of gas you’re willing to spend on a particular transaction. After the transaction is executed, if too much gas (gasLimit) was sent, the remaining gas is refunded to the sender.

- **Gas Price (in wei):** Amount of Ether you’re willing to pay for every unit of gas

```java
// A transfer cost 21,000 units of gas
BigInteger gasLimit = BigInteger.valueOf(21000);

// I am willing to pay 1Gwei (1,000,000,000 wei or 0.000000001 ether) for each unit of gas consumed by the transaction.
BigInteger gasPrice = Convert.toWei("1", Unit.GWEI).toBigInteger();
```


#### 4. Prepare the raw transaction

A raw transaction for a transfer of funds contains all the transaction data fields except:
- data: not a smart contract transaction
- signature: signature not signed yet

```java
// Prepare the rawTransaction
RawTransaction rawTransaction  = RawTransaction.createEtherTransaction(
	nonce,
	gasPrice,
	gasLimit,
	recipientAddress,
	value);
```

#### 5. Signature

The signing part requires the rawTransaction as well as the `credentials` (keypair) used to cryptographically sign the transaction.

```java
// Sign the transaction
byte[] signedMessage = TransactionEncoder.signMessage(rawTransaction, credentials);

// Convert it to Hexadecimal String to be sent to the node
String hexValue = Numeric.toHexString(signedMessage);
```

#### 6. Send to the node via JSON-RPC

Final step consists in sending the transaction signed to the node so it can be verified and broadcasted to the network. In case of success, the method returns a response only composed of the transaction hash.

```java
// Send transaction
EthSendTransaction ethSendTransaction = web3.ethSendRawTransaction(hexValue).send();

// Get the transaction hash
String transactionHash = ethSendTransaction.getTransactionHash();
```


#### 7. Wait for the transaction to be mined.

As explained previously, when the signed transaction is propagated to the network, depending on many factors (gas price, network congestion) it can take some time to see the transaction mined and added to the last block.

That's why the following code consists of a simple loop to verify every 3 seconds if the transaction is mined by calling the method `web3.ethGetTransactionReceipt(<txhash>).send()`.

```java
// Wait for transaction to be mined
Optional<TransactionReceipt> transactionReceipt = null;
do {
  EthGetTransactionReceipt ethGetTransactionReceiptResp = web3.ethGetTransactionReceipt(transactionHash).send();
  transactionReceipt = ethGetTransactionReceiptResp.getTransactionReceipt();

  Thread.sleep(3000); // Retry after 3 sec
} while(!transactionReceipt.isPresent());
```


#### Result

Here is the full version of the code including everything explained in this article:

```java
// Transaction.java
package io.kauri.tutorials.java_ethereum;

import java.io.IOException;
import java.math.BigInteger;
import java.util.Optional;

import org.web3j.crypto.Credentials;
import org.web3j.crypto.RawTransaction;
import org.web3j.crypto.TransactionEncoder;
import org.web3j.protocol.Web3j;
import org.web3j.protocol.core.DefaultBlockParameterName;
import org.web3j.protocol.core.methods.response.EthGetTransactionCount;
import org.web3j.protocol.core.methods.response.EthGetTransactionReceipt;
import org.web3j.protocol.core.methods.response.EthSendTransaction;
import org.web3j.protocol.core.methods.response.TransactionReceipt;
import org.web3j.protocol.http.HttpService;
import org.web3j.utils.Convert;
import org.web3j.utils.Convert.Unit;
import org.web3j.utils.Numeric;

public class Transaction {

  public static void main(String[] args)  {

    System.out.println("Connecting to Ethereum ...");
    Web3j web3 = Web3j.build(new HttpService("https://rinkeby.infura.io/v3/083836b2784f48e19e03487eb3209923"));
    System.out.println("Successfuly connected to Ethereum");

    try {
      String pk = "CHANGE_ME"; // Add a private key here

      // Decrypt and open the wallet into a Credential object
      Credentials credentials = Credentials.create(pk);
      System.out.println("Account address: " + credentials.getAddress());
      System.out.println("Balance: " + Convert.fromWei(web3.ethGetBalance(credentials.getAddress(), DefaultBlockParameterName.LATEST).send().getBalance().toString(), Unit.ETHER));

      // Get the latest nonce
      EthGetTransactionCount ethGetTransactionCount = web3.ethGetTransactionCount(credentials.getAddress(), DefaultBlockParameterName.LATEST).send();
      BigInteger nonce =  ethGetTransactionCount.getTransactionCount();

      // Recipient address
      String recipientAddress = "0xAA6325C45aE6fAbD028D19fa1539663Df14813a8";

      // Value to transfer (in wei)
      BigInteger value = Convert.toWei("1", Unit.ETHER).toBigInteger();

      // Gas Parameters
      BigInteger gasLimit = BigInteger.valueOf(21000);
      BigInteger gasPrice = Convert.toWei("1", Unit.GWEI).toBigInteger();

      // Prepare the rawTransaction
      RawTransaction rawTransaction  = RawTransaction.createEtherTransaction(
                 nonce,
                 gasPrice,
                 gasLimit,
                 recipientAddress,
                 value);

      // Sign the transaction
      byte[] signedMessage = TransactionEncoder.signMessage(rawTransaction, credentials);
      String hexValue = Numeric.toHexString(signedMessage);

      // Send transaction
      EthSendTransaction ethSendTransaction = web3.ethSendRawTransaction(hexValue).send();
      String transactionHash = ethSendTransaction.getTransactionHash();
      System.out.println("transactionHash: " + transactionHash);

      // Wait for transaction to be mined
      Optional<TransactionReceipt> transactionReceipt = null;
      do {
        System.out.println("checking if transaction " + transactionHash + " is mined....");
            EthGetTransactionReceipt ethGetTransactionReceiptResp = web3.ethGetTransactionReceipt(transactionHash).send();
            transactionReceipt = ethGetTransactionReceiptResp.getTransactionReceipt();
            Thread.sleep(3000); // Wait 3 sec
      } while(!transactionReceipt.isPresent());

      System.out.println("Transaction " + transactionHash + " was mined in block # " + transactionReceipt.get().getBlockNumber());
      System.out.println("Balance: " + Convert.fromWei(web3.ethGetBalance(credentials.getAddress(), DefaultBlockParameterName.LATEST).send().getBalance().toString(), Unit.ETHER));


    } catch (IOException | InterruptedException ex) {
      throw new RuntimeException(ex);
    }
  }
}

```


![](https://imgur.com/8XU21KA.gif)


<br />

Now you understand the core principles behind sending transactions with Web3j, I can tell you a secret. Web3j provides a Utility class called 'Transfer' which takes care of everything (nonce, gas, transaction receipt polling, etc.) in one line of code.

```java
TransactionReceipt receipt = Transfer.sendFunds(web3, credentials, recipientAddress, BigDecimal.valueOf(1), Unit.ETHER).send();
```

<br />

## Summary

In this article, we learnt that the Ethereum Global State is actually composed of a mapping of all accounts states. Each account state can be queried to get information like the balance and the nonce.

An account is controlled by the person owning the private key of this account. The private key can have many forms and is usually secured in a wallet. Web3j allows to open a wallet from a JSON encrypted file, a mnemonic phrase or directly from the private key.

To send a transaction between two accounts, Web3j can generate a transaction oject, sign it and propagate it to the network to finally pool the Blockchain in order to get the transaction receipt when it's been mined.


<br />

## Resources
- [Ethereum Unit converter (WEI, GWEI, ETHER, ....)](https://etherconverter.online/)
- [Web3j Transaction doc](https://web3j.readthedocs.io/en/latest/transactions.html#transaction-signing-via-an-ethereum-client)
- [Web3j RawTransaction Integration Tests](https://github.com/web3j/web3j/blob/master/integration-tests/src/test/java/org/web3j/protocol/scenarios/CreateRawTransactionIT.java)
- [Ethereum - What is Gas Price and Limit](https://masterthecrypto.com/ethereum-what-is-gas-gas-limit-gas-price/)
- [Diving into Ethereum World State](https://medium.com/cybermiles/diving-into-ethereums-world-state-c893102030ed)
