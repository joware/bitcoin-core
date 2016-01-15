# bitcoin-core
A modern Bitcoin Core REST and RPC client to execute administrative tasks, wallet operations and queries about network and the blockchain.

## Status
[![npm version][npm-image]][npm-url] [![build status][travis-image]][travis-url]

## Installation
Install the package via `npm`:

```sh
npm install bitcoin-core --save
```

## Usage
### Arguments
1. `[agentOptions]` _(Object)_: Optional `agent` [options](https://github.com/request/request#using-optionsagentoptions) to configure SSL/TLS.
2. `[headers=false]` _(boolean)_: Whether to return the response headers.
3. `[host=localhost]` _(string)_: The host to connect to.
4. `[network=mainnet]` _(string)_: The network
5. `[password]` _(string)_: The RPC server user password.
6. `[port=[network]]` _(string)_: The RPC server port.
7. `[ssl]` _(boolean|Object)_: Whether to use SSL/TLS with strict checking (_boolean_) or an expanded config (_Object_).
8. `[ssl.enabled]` _(boolean)_: Whether to use SSL/TLS.
9. `[ssl.strict]` _(boolean)_: Whether to do strict SSL/TLS checking (certificate must match host).
10. `[timeout=30000]` _(number)_: How long until the request times out (ms).
11. `[username]` _(number)_: The RPC server user name.
12. `[version]` _(string)_: Which version to check methods for ([read more](#versionchecking)).

### Examples
#### Using network mode

```js
const Client = require('bitcoin-core');
const client = new Client({
  network: 'regtest'
});
```

#### Using a custom port

```js
const client = new Client({
  port: 28332
});
```

#### Connecting to an SSL/TLS server with strict checking enabled

```js
const fs = require('fs');
const client = new Client({
  agentOptions: {
    ca: fs.readFileSync('/etc/ssl/bitcoind/cert.pem')),
  },
  ssl: true
});
```

#### Connecting to an SSL/TLS server without strict checking enabled

```js
const client = new Client({
  ssl: {
    enabled: true,
    strict: false
  }
});
```

#### Using promises to process the response

```
// Using promise style:
client.getInfo().then((help) => console.log(help));
```

#### Using callbacks to process the response

```
client.getInfo((error, help) => console.log(help));
```

#### Returning headers in the response

```js
const client = new Client({
  headers: true
});

// Promise style with headers enabled:
client.getInfo().then(([body, headers]) => console.log(body, headers));

// Await style based on promises with headers enabled:
const [body, headers] = await client.getInfo();
console.log(body, headers);
```

### Version Checking
By default, all methods are exposed on the client independently of the version it is connecting to. This is the most flexible option as defining methods for unavailable RPC calls does not cause any harm and the library is capable of handling a `Method not found` response error correctly.

```js
const client = new Client({
  version: '0.12.0'
});

client.command('foobar');
// RpcError: -32601 Method not found
```

However, if you prefer to be on the safe side, you can enable strict version checking. This will validate all method calls before executing the actual RPC request:

```js
const client = new Client({
  version: '0.12.0'
});

client.getHashesPerSec();
// Method "gethashespersec" is not supported by version "0.12.0"
```

If you want to enable strict version checking for the bleeding edge version, you may set a very high version number to exclude recently deprecated calls:

```js
const client = new Client({
  version: `${Number.MAX_SAFE_INTEGER}.0.0`
});

// Method `getwork` was deprecated in 0.11.0, so version 9007199254740991.0.0 will
// definitely not have it too.
client.getWork();
// Throws 'Method "getwork" is not supported by version "9007199254740991.0.0"'.
```

To avoid potential issues with prototype references, all methods are still enumerable on the library client prototype.

### RPC
Start the `bitcoind` with the RPC server enabled and optionally configure a username and password:

```sh
bitcoind -rpcuser=foo -rpcpassword=bar -server
```

These configuration values may also be set on the `bitcoin.conf` file of your platform installation.

By default, port `8332` is used to listen for requests in `mainnet` mode, or `18332` in `testnet` or `regtest` modes. Use the `network` property to initialize the client on the desired mode and automatically set the respective default port. You can optionally set a custom port of your choice too.

The RPC services binds to the localhost loopback network interface, so use `rpcbind` to change where to bind to and `rpcallowip` to whitelist source IP access.

#### Batch requests
Batched requests are support by passing an array to the `command` method with a `method` and optionally, `parameters`. The return value will be an array with all the responses.

```js
const batch = [
  { method: 'getnewaddress', parameters: [] },
  { method: 'getnewaddress', parameters: [] }
]

new Client().command(batch).then((responses) => console.log(responses)));

// Or, using ES2015 destructuring.
new Client().command(batch).then(([firstAddress, secondAddress]) => console.log(firstAddress, secondAddress)));
```

Note that batched requests will only throw an error if the batch request itself cannot be processed. However, each individual response may contain an error akin to an individual request.

```js
const batch = [
  { method: 'foobar', params: [] },
  { method: 'getnewaddress', params: [] }
]

// Logs `mkteeBFmGkraJaWN5WzqHCjmbQWVrPo5X3, { [RpcError: Method not found] message: 'Method not found', name: 'RpcError', code: -32601 }`.
new Client().command(batch).then(([address, error]) => console.log(address, error)));
```

### REST
Support for the REST interface is still **experimental** and the API is still subject to change. These endpoints are also **unauthenticated** so [there are certain risks which you should be aware](https://github.com/bitcoin/bitcoin/blob/master/doc/REST-interface.md#risks), specifically of leaking sensitive data of the node if not correctly protected.

Error handling is still fragile so avoid passing user input.

Start the `bitcoind` with the REST server enabled:

```sh
bitcoind -server -rest
```

These configuration values may also be set on the `bitcoin.conf` file of your platform installation. Use `txindex=1` if you'd like to enable full transaction query support (note: this will take a considerable amount of time on the first run).

### getTransactionByHash()
Given a transaction hash, returns a transaction in binary, hex-encoded binary, or JSON formats.

##### Arguments
1. `hash` _(string)_: the transaction hash.
2. `[summary=false]` _(boolean)_: whether to return just the transaction hash, thus saving memory.
3. `[extension=json]` _(string)_: return in binary (`bin`), hex-encoded binary (`hex`), or JSON format.

### getBlockByHash()
Given a block hash, returns a block, in binary, hex-encoded binary or JSON formats.

##### Arguments
1. `hash` _(string)_: the block hash.
2. `[extension=json]` _(string)_: return in binary (`bin`), hex-encoded binary (`hex`), or JSON format.

### getBlockHeadersByHash()
Given a block hash, returns amount of block headers in upward direction.

##### Arguments
1. `hash` _(string)_: the block hash.
2. `count` _(number)_: the number of blocks to count in upward direction.
3. `[extension=json]` _(string)_: return in binary (`bin`), hex-encoded binary (`hex`), or JSON format.

### getBlockchainInformation()
Returns various state info regarding block chain processing.

### getUnspentTransactionOutputs()
Query unspent transaction outputs (UTXO) for a given set of outpoints. See [BIP64](https://github.com/bitcoin/bips/blob/master/bip-0064.mediawiki) for input and output serialisation.

##### Arguments
1. `outpoints` _(array\<Object\>|Object)_: the outpoint to query in the format `{ id: "<txid>", index: "<index>" }`.
2. `[extension=json]` _(string)_: return in binary (`bin`), hex-encoded binary (`hex`), or JSON format.

### getMemoryPoolContent()
Returns transactions in the transaction memory pool.

### getMemoryPoolInformation()
Returns various information about the transaction memory pool. Only supports JSON as output format.
- size: the number of transactions in the transaction memory pool.
- bytes: size of the transaction memory pool in bytes.
- usage: total transaction memory pool memory usage.

### SSL
This client supports SSL out of the box. Simply pass the SSL public certificate to the client and optionally disable strict SSL checking which will bypass SSL validation (the connection is still encrypted but the server it is connecting to may not be trusted). This is, of course, discouraged unless for testing purposes when using something like self-signed certificates.

#### Generating an self-signed certificates for testing purposes
Please note that the following procedure should only be used for testing purposes.

Generate an self-signed certificate together with an unprotected private key:

```sh
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 3650 -nodes
```

#### Connecting via SSL
On Bitcoin Core <0.12, you can start the `bitcoind` RPC server directly with SSL:

```sh
bitcoind -rpcuser=foo -rpcpassword=bar -rpcssl -rpcsslcertificatechainfile=/etc/ssl/bitcoind/cert.pem -rpcsslprivatekeyfile=/etc/ssl/bitcoind/key.pem -server
```

On Bitcoin Core >0.12, use must use `stunnel` (`brew install stunnel` or `sudo apt-get install stunnel4`) or an HTTPS reverse proxy to configure SSL since the built-in support for SSL has been removed. The trade off with `stunnel` is performance and simplicity versus features, as it lacks more powerful capacities such as Basic Authentication and caching which are standard in reverse proxies.

You can use `stunnel` by configuring `stunnel.conf` with the following service requirements:

```
[bitcoin]
accept = 28332
connect = 18332
cert = /etc/ssl/bitcoind/cert.pem
key = /etc/ssl/bitcoind/key.pem
```

The `key` option may be omitted if you concatenating your private and public certificates into a single `stunnel.pem` file.

On some versions of `stunnel` it is also possible to start a service using command line arguments. The equivalent would be:

```sh
stunnel -d 28332 -r 127.0.0.1:18332 -p stunnel.pem -P ''
```

Then pass the public certificate to the client:

```js
const Client = require('bitcoin-core');
const fs = require('fs');
const client = new Client({
  agentOptions: {
    ca: fs.readFileSync('/etc/ssl/bitcoind/cert.pem')),
  },
  port: 28332,
  ssl: true
});
```

## Tests

```sh
npm test
```

## License
MIT

[npm-image]: https://img.shields.io/npm/v/bitcoin-core.svg?style=flat-square
[npm-url]: https://npmjs.org/package/bitcoin-core
[travis-image]: https://img.shields.io/travis/seegno/bitcoin-core.svg?style=flat-square
[travis-url]: https://travis-ci.org/seegno/bitcoin-core