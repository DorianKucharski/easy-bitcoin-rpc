# Easy Bitcoin RPC

Easy Bitcoin RPC is a simple and easy to use Python library for the Bitcoin Core JSON-RPC interface. It requires Python 3.10+ and depends only on `requests`.

## Installation

Use the package manager [pip](https://pip.pypa.io/en/stable/) to install.

```bash
pip install easybitcoinrpc
```

## Usage

### Connecting

```python
from easybitcoinrpc import RPC

# RPC() returns connection object

rpc = RPC() # defaults: ip=127.0.0.1, port=8332, user="user", password="password", wallet_name=None

rpc = RPC(user="rpcuser", password="rpcpassword")

rpc = RPC(ip="192.168.1.1", port=9999, user="rpcuser", password="rpcpassword")

rpc = RPC(wallet_name="Wallet")

```

Specifying the wallet when the connection is made is the preferred way to access a wallet; loading a wallet after connecting sometimes doesn't work.

### RPC methods

The RPC object has methods for all Bitcoin RPC calls, and those methods are separated into sections as in the [Bitcoin RPC API Reference](https://developer.bitcoin.org/reference/rpc/)


```python
from easybitcoinrpc import RPC
rpc = RPC()

rpc.blockchain # returns object which has all methods from blockchain category
rpc.wallet # returns object which has all methods from wallet category
rpc.util # returns object which has all methods from util category
rpc.mining # returns object which has all methods from mining category
rpc.network # returns object which has all methods from network category
rpc.generating # returns object which has all methods from generating category
rpc.control # returns object which has all methods from control category
rpc.transactions # returns object which has all methods from transactions category

rpc.client.call("getbestblockhash") # any RPC command can be called directly
rpc.client.call("getblock", block_hash, 2) # parameters follow the command
```

All methods have documentation copied from [Bitcoin RPC API Reference](https://developer.bitcoin.org/reference/rpc/), 
they also have parameters specified with their types and their default values.

### Data

Methods for blocks and transactions return frozen dataclasses: `Block`, `Transaction`, `Vin`,
`Vout`, `ScriptSig`, `ScriptPubKey`, `TransactionSummary`. Every field is a plain attribute and
amounts are `Decimal`, never `float`.

```python
from easybitcoinrpc import RPC

rpc = RPC()

block = rpc.blockchain.get_block(669376, 2)
print(block.height, block.mined_at, block.transaction_count)

for transaction in block.transactions:
    if transaction.is_segwit:
        print(transaction.txid, transaction.total_output)
```

`get_block` is overloaded on the verbosity: `0` returns the hex string, `1` and `2` return a
`Block`, and only `2` fills `block.transactions`.

```python
raw_hex = rpc.blockchain.get_block(669376, 0)
with_txids = rpc.blockchain.get_block(669376)
with_transactions = rpc.blockchain.get_block(669376, 2)
```

### Batching

Anything that would otherwise be a loop of calls goes out as one JSON-RPC batch.

```python
blocks = rpc.blockchain.get_blocks([669376, 669377, 669378])
transactions = rpc.transactions.get_transactions([first_txid, second_txid])
```

### Transaction summary

`TransactionSummary` resolves the spent outputs into addresses and amounts. The funding
transactions are fetched in a single batch.

```python
summary = rpc.transactions.get_transaction_summary(txid)

for entry in summary.inputs:
    print(entry.address, entry.amount)

print(summary.total_input, summary.total_output, summary.fee)
```

The price is an argument, so the library never reaches out to a price feed on its own.

```python
from decimal import Decimal

print(summary.value_in(Decimal("42000")))
```

### Errors

A rejected call raises a subclass of `RpcError` carrying the node's `code` and `message`.

```python
from easybitcoinrpc import RPC, WalletNotFoundError, AuthenticationError, RpcError

try:
    rpc.wallet.get_balance()
except WalletNotFoundError:
    ...
except AuthenticationError:
    ...
except RpcError as error:
    print(error.code, error.message)
```

### Closing the connection

`RPC` keeps one pooled HTTP session and works as a context manager.

```python
with RPC(user="rpcuser", password="rpcpassword") as rpc:
    print(rpc.blockchain.get_block_count())
```

## Development

The project uses [uv](https://docs.astral.sh/uv/). These are the same commands the CI workflow runs:

```bash
uv sync                                               # install the project and its dev group
uv run mypy                                           # type-check
uv run python -m unittest discover -s tests -t tests  # run the tests
```

The tests run against a mock node (`tests/mock_node.py`), so no running Bitcoin node is needed.

## License

See [LICENSE](LICENSE).
