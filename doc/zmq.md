# Block and Transaction Broadcasting with ZeroMQ

[ZeroMQ](https://zeromq.org/) is a lightweight wrapper around TCP
connections, inter-process communication, and shared-memory,
providing various message-oriented semantics such as publish/subscribe,
request/reply, and push/pull.

The Bitcoin Core daemon can be configured to act as a trusted "border
router", implementing the bitcoin wire protocol and relay, making
consensus decisions, maintaining the local blockchain database,
broadcasting locally generated transactions into the network, and
providing a queryable RPC interface to interact on a polled basis for
requesting blockchain related data. However, there exists only a
limited service to notify external software of events like the arrival
of new blocks or transactions.

The ZeroMQ facility implements a notification interface through a set
of specific notifiers. Currently there are notifiers that publish
blocks and transactions. This read-only facility requires only the
connection of a corresponding ZeroMQ subscriber port in receiving
software; it is not authenticated nor is there any two-way protocol
involvement. Therefore, subscribers should validate the received data
since it may be out of date, incomplete or even invalid.

ZeroMQ sockets are self-connecting and self-healing; that is,
connections made between two endpoints will be automatically restored
after an outage, and either end may be freely started or stopped in
any order.

Because ZeroMQ is message oriented, subscribers receive transactions
and blocks all-at-once and do not need to implement any sort of
buffering or reassembly.

## Prerequisites

The ZeroMQ feature in Bitcoin Core requires the ZeroMQ API >= 4.0.0
[libzmq](https://github.com/zeromq/libzmq/releases).
For version information, see [dependencies.md](dependencies.md).
Typically, it is packaged by distributions as something like
*libzmq3-dev*. The C++ wrapper for ZeroMQ is *not* needed.

In order to run the example Python client scripts in the `contrib/zmq/`
directory, one must also install [PyZMQ](https://github.com/zeromq/pyzmq)
(generally with `pip install pyzmq`), though this is not necessary for daemon
operation.

## Enabling

By default, the ZeroMQ feature is not automatically compiled.
To enable, use `-DWITH_ZMQ=ON` when configuring the build system:

    $ cmake -B build -DWITH_ZMQ=ON

To actually enable operation, one must set the appropriate options on
the command line or in the configuration file.

## Usage

Currently, the following notifications are supported:

    -zmqpubhashtx=address
    -zmqpubhashblock=address
    -zmqpubrawblock=address
    -zmqpubrawtx=address
    -zmqpubsequence=address

The socket type is PUB and the address must be a valid ZeroMQ socket
address. The same address can be used in more than one notification.
The same notification can be specified more than once.

The option to set the PUB socket's outbound message high water mark
(SNDHWM) may be set individually for each notification:

    -zmqpubhashtxhwm=n
    -zmqpubhashblockhwm=n
    -zmqpubrawblockhwm=n
    -zmqpubrawtxhwm=n
    -zmqpubsequencehwm=n

The high water mark value must be an integer greater than or equal to 0.

For instance:

    $ bitcoind -zmqpubhashtx=tcp://127.0.0.1:28332 \
               -zmqpubhashtx=tcp://192.168.1.2:28332 \
               -zmqpubhashblock="tcp://[::1]:28333" \
               -zmqpubrawtx=unix:/tmp/bitcoind.tx.raw \
               -zmqpubhashtxhwm=10000

`bitcoin node` or `bitcoin gui` can also be substituted for `bitcoind`.

Notification types correspond to message topics (details in next section). For instance,
for the notification `-zmqpubhashtx` the topic is `hashtx`. These options can also be
provided in bitcoin.conf.

### Message format

All ZMQ messages share the same structure with three parts: _topic_ string,
message _body_, and _message sequence number_:

    | topic     | body                                                 | message sequence number  |
    |-----------+------------------------------------------------------+--------------------------|
    | rawtx     | <serialized transaction>                             | <4-byte LE uint>         |
    | hashtx    | <reversed 32-byte transaction hash>                  | <4-byte LE uint>         |
    | rawblock  | <serialized block>                                   | <4-byte LE uint>         |
    | hashblock | <reversed 32-byte block hash>                        | <4-byte LE uint>         |
    | sequence  | <reversed 32-byte block hash>C                       | <4-byte LE uint>         |
    | sequence  | <reversed 32-byte block hash>D                       | <4-byte LE uint>         |
    | sequence  | <reversed 32-byte transaction hash>R<8-byte LE uint> | <4-byte LE uint>         |
    | sequence  | <reversed 32-byte transaction hash>A<8-byte LE uint> | <4-byte LE uint>         |

where:

 - message sequence number represents message count to detect lost messages, distinct for each topic
 - all transaction and block hashes are in _reversed byte order_ (i. e. with bytes
   produced by hashing function reversed), the same format as the RPC interface and block
   explorers use to display transaction and block hashes

#### rawtx

Notifies about all transactions, both when they are added to mempool or when a new block
arrives. This means a transaction could be published multiple times: first when it enters
mempool and then again in each block that includes it. The body part of the message is the
serialized transaction.

#### hashtx

Notifies about all transactions, both when they are added to mempool or when a new block
arrives. This means a transaction could be published multiple times: first when it enters
mempool and then again in each block that includes it. The body part of the message is the
32-byte transaction hash in reversed byte order.

#### rawblock

Notifies when the chain tip is updated. When assumeutxo is in use, this notification will
not be issued for historical blocks connected to the background validation chainstate. The
body part of the message is the serialized block.

#### hashblock

Notifies when the chain tip is updated. When assumeutxo is in use, this notification will
not be issued for historical blocks connected to the background validation chainstate. The
body part of the message is the 32-byte block hash in reversed byte order.

#### sequence

The 8-byte LE uints correspond to _mempool sequence number_ and the types of bodies are:

   - `C` : block with this hash connected
   - `D` : block with this hash disconnected
   - `R` : transaction with this hash removed from mempool for non-block inclusion reason
   - `A` : transaction with this hash added to mempool

### Implementing ZMQ client

ZeroMQ endpoint specifiers for TCP (and others) are documented in the
[ZeroMQ API](https://libzmq.readthedocs.io/en/zeromq4-x/).

Client side, then, the ZeroMQ subscriber socket must have the
ZMQ_SUBSCRIBE option set to one or either of these prefixes (for
instance, just `hash`); without doing so will result in no messages
arriving. Please see [`contrib/zmq/zmq_sub.py`](/contrib/zmq/zmq_sub.py) for a working example.

The ZMQ_PUB socket's ZMQ_TCP_KEEPALIVE option is enabled. This means that
the underlying SO_KEEPALIVE option is enabled when using a TCP transport.
The effective TCP keepalive values are managed through the underlying
operating system configuration and must be configured prior to connection establishment.

For example, when running on GNU/Linux, one might use the following
to lower the keepalive setting to 10 minutes:

    sudo sysctl -w net.ipv4.tcp_keepalive_time=600

Setting the keepalive values appropriately for your operating environment may
improve connectivity in situations where long-lived connections are silently
dropped by network middle boxes.

Also, the socket's ZMQ_IPV6 option is enabled to accept connections from IPv6
hosts as well. If needed, this option has to be set on the client side too.

## Remarks

From the perspective of bitcoind, the ZeroMQ socket is write-only; PUB
sockets don't even have a read function. Thus, there is no state
introduced into bitcoind directly. Furthermore, no information is
broadcast that wasn't already received from the public P2P network.

No authentication or authorization is done on connecting clients; it
is assumed that the ZeroMQ port is exposed only to trusted entities,
using other means such as firewalling.

Note that for `*block` topics, when the block chain tip changes,
a reorganisation may occur and just the tip will be notified.
It is up to the subscriber to retrieve the chain from the last known
block to the new tip. Also note that no notification will occur if the tip
was in the active chain, as would be the case after calling the `invalidateblock` RPC.
In contrast, the `sequence` topic publishes all block connections and
disconnections.

There are several possibilities that ZMQ notification can get lost
during transmission depending on the communication type you are
using. Bitcoind appends an up-counting sequence number to each
notification which allows listeners to detect lost notifications.

The `sequence` topic refers specifically to the mempool sequence
number, which is also published along with all mempool events. This
is a different sequence value than in ZMQ itself in order to allow a total
ordering of mempool events to be constructed.
