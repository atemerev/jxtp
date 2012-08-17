# The JXTP protocol specification, version 2.0.0

JXTP is financial information exchange protocol (like FIX), but unlike FIX, simple, JSON-based, and containing only things needed, well, for me.

Documentation is like sex — the good one is awesome, and the bad one is still better than nothing. This particular document is hardly better than masturbation, but this is going to be improved.

Goals of JXTP:

* Be elegant;
* Be as simple as possible, but no simpler;
* Be joy to work with.

I.e., "don't be FIX".

## Meta

Request-response mechanism is implemented with "seq" / "for" attributes.

Each message carries a "seq" attribute.

If incoming message from server is a reply, it carries a "for" attribute with seq value of initial request.

Seq is initialized within each session at both sides at 0.

Required attributes for each message:

* "type" — identifies message type;
* "time" — message timestamp (epoch time, milliseconds);
* "seq" — message sequence number (start with 0 for each session, increments with each message).

Optional attributes common for all messages (therefore, they can't be used in specific messages):

* "for" — indicator of response. If this attribute exists, this message is response for some message sent as a request with specified seq number.
* "nano" — nanoseconds for current millisecond, from 0 to 999999. This might be added for HFT applications.

## Messages

### Session and transport

#### HELLO

Each JXTP server sends HELLO message right after session opens. The "challenge" attribute is used in authentication sequence.

{"type": "hello", "version": "JXTP 2.0.0", "challenge": 973234232, "time": 124235712023, "seq": 0}

#### HEARTBEAT

JXTP server sends HEARTBEAT messages every second. JXTP clients can use it to determine connection quality.

{"type": "beat", "time": 124235712023, "seq": 2}

#### OK

If some message has been send as a request, OK and REJECT messages are universal response. If the response consists of many messages with known number (e.g. positions in portfolio), "count" attribute is required, to tell it. Otherwise, "count" is optional.

{"type": "ok", "for": 243, "time": 124235712023, "count": 3, "seq": 2}

#### REJECT

Universal response for the case something went wrong.

{"type": "reject", "for": 376, "code": 13, "reason": "Sorry Dave, can't do that", "time": 124235712023, "seq": 3}

#### REQUEST

A universal request for some application-specific data (portfolios, account metadata etc.) Response can be single message (dispatched immediately with "for" attribute set with request seq number) or multiple messages (preceded with OK message with subsequent responses count).

{"type": "request", "what": "/portfolio", "time": 124235712023, "seq": 3}

### Authentication and authorization

#### AUTH

If some services provided by JXTP server require authorization, this is the way to do it. "Key" is authorization key for this session, which is calculated using session challenge from HELLO message. Specific authentication/authorization algorithms are presently unspecified.

{"type": "auth", "account": "demo", "key": "64af209a312f832", "time": 124235712023, "seq": 2}

Possible responses: OK, REJECT

### Market data

#### SUBSCRIBE

Subscribe to quote feed for some set of market instruments (as comma-separated list of tickers).

{"type": "subscribe", "feed": "EUR/USD,USD/CHF,GBP/USD", "time": 124235712023, "seq": 3}

Possible responses: OK, REJECT, QUOTE (but not marked as response with "for")

#### UNSUBSCRIBE

{"type": "unsubscribe", "feed": "USD/CHF,GBP/USD", "time": 124235712023, "seq": 2}

#### QUOTE

Market snapshot, where 'p' is price, 'v' is volume available for this price.

{"type": "quote", "instrument": "EUR/USD", "bids": [{"p": 1.1298, "v": 500000}], "asks": [{"p": 1.1301, "v": 100000}, {"p": 1.1322, "v": 700000}], "time": 124235712023, "seq": 423424}

### Execution and position management

#### ORDER

{"type": "order", "instrument": "USD/CHF", "amount": -100000, "mode": "FOK", "price": "0.9125", "time": 124235712023, "seq": 212}

Order ID should be unique within account. UUIDs can be used if necessary.

Order modes supported: IOC, FOK, GTC and GTD.

If GTD mode is specified, "expiration" field is necessary containing expiration timestamp.

In future, "instructions" parameter can be added, containing additional routing and execution instructions.

FOK and IOC orders are executed (or rejected) immediately, returning a single TRADE message. GTC and GTD orders can be executed later, with STATUS messages returned containing order ID.

#### STATUS

Order status. Can be requested by REQUEST("/order/:id") message, or delivered as order status changes.

{"type": "status", "for": 1, "order": 195231, "value": "ACCEPTED", "time": 124235712023, "seq": 234}

{"type": "status", "for": 1, "value": "REJECTED", "code": 3, "reason": "Can't do that", "time": 124235712023, "seq": 323}

{"type": "status", "order": 195231, "value": "FILLED", "time": 124235712023, "seq": 65622}

{"type": "status", "order": 195231, "value": "CANCELLED", "reason", "time": 124235712023, "seq": 231}

#### CANCEL

Request to cancel some order.

{"type": "cancel", "order": 195231, "time": 124235712023, "seq": 231}

#### TRADE

{"type": "trade", "id": 543, "order": 1, "instrument": "EUR/USD", "amount": -50000, price: "1.3764", "time": 124235712023, "seq": 313}

#### POSITION

{"type": "position", "id": 218647, "instrument": "EUR/USD", "amount": -5000}
