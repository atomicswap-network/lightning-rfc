# BOLT #1: Lightning Networkにおける基盤プロトコル

## 概要

このプロトコルは、個々のメッセージの枠組みを処理する、基盤となる認証済みの順序付けられた転送機構を前提としています。
[BOLT #8](08-transport.md)では、Lightning Network上で標準的に使われる転送レイヤーについて指定しています。
なお、これは上記の保証を満たす任意の転送方式に置き換えることができます。

Lightning NetworkにおけるデフォルトTCPポートは9735です。
これは16進数の `0x2607` に対応します(UnicodeにおけるLIGHTNINGという文字の番号です<sup>[1](#reference-1)</sup>)

特に指定がない限り、すべてのデータフィールドは符号なしのビッグエンディアンです。

## 目次

  * [Connection Handling and Multiplexing(接続の取り扱いと多重化)](#connection-handling-and-multiplexing)
  * [Lightning Message Format(Lightningメッセージの形式について)](#lightning-message-format)
  * [Type-Length-Value Format(TLVの形式について)](#type-length-value-format)
  * [Fundamental Types(主要なタイプ)](#fundamental-types)
  * [Setup Messages(セットアップメッセージについて)](#setup-messages)
    * [The `init` Message(`init`メッセージについて)](#the-init-message)
    * [The `error` Message(`error`メッセージについて)](#the-error-message)
  * [Control Messages(操作メッセージについて)](#control-messages)
    * [The `ping` and `pong` Messages(`ping` と `pong` メッセージについて)](#the-ping-and-pong-messages)
  * [Appendix A: BigSize Test Vectors(付録A: 大きなサイズのテストの方向性)](#appendix-a-bigsize-test-vectors)
  * [Appendix B: Type-Length-Value Test Vectors(付録B: TLVテストの方向性)](#appendix-b-type-length-value-test-vectors)
  * [Appendix C: Message Extension(付録C: メッセージ拡張)](#appendix-c-message-extension)
  * [Acknowledgments(謝辞)](#acknowledgments)
  * [References(参考文献)](#references)
  * [Authors(著者)](#authors)

## Connection Handling and Multiplexing
#### 接続の取り扱いと多重化

実装では、ピアごとに1つの接続を確立し、使う必要があります。Channel IDを含んだChannelメッセージはこの確立された1つの接続を介して多重化されます。

## Lightning Message Format
#### Lightningメッセージの形式について

複合化された全てのLightning Networkメッセージの形は次の通りです。

1. `type`: 2バイトのビッグエンディアンフィールドで、メッセージのタイプを示します
2. `payload`: 長さが可変なデータ本体です。 メッセージの残りの部分を構成するもので、
   `type` に基づいた形式になります。
3. `extension`: [TLV](#type-length-value-format)オプション用のフィールドです。

`type` フィールドは `payload` フィールドの解釈方法を指定します。
ここのタイプの形式は、このリポジトリ内の仕様で定義されています。
タイプは _It's ok to be odd(奇数なら大丈夫)_ ルールに従っていているので、受信者がそれを理解しているかどうかを確認せずに、ノードが奇数タイプのものを送信する場合があります。

メッセージは論理的に5つのグループにグループ化され、設定されている最上位ビットの順に並べられます:

  - Setup & Control (タイプ: `0`-`31`): メッセージはコネクションセットアップ、コントロール、特徴のサポート、エラーの報告に関係します(以下で説明します)
  - Channel (タイプ `32`-`127`): メッセージはマイクロペイメントチャンネルのセットアップかシャットダウンに使われています ([BOLT #2](02-peer-protocol.md)で説明されています)
  - Commitment (タイプ `128`-`255`): メッセージは現在の言質取引を更新することに関係します。更新内容には、HTLCのほか、料金の更新を追加したり、無効化したり、決済したりすること、また、署名の交換が含まれます([BOLT #2](02-peer-protocol.md)で説明されています)
  - Routing (タイプ `256`-`511`): NodeやChannelのアナウンスを含むメッセージ、そのほかにアクティブなRouteの探索するためにも用います([BOLT #7](07-routing-gossip.md)で説明されています)
  - Custom (タイプ `32768`-`65535`): 実験的、アプリケーション特有のメッセージです。

メッセージのサイズは転送レイヤーに収まるように、2バイトの正の整数である必要があります。よって、65535バイトが最大になります。

送信ノード:
  - 事前の交渉なしに、ここに列挙されていない規則のタイプのメッセージを送信してはなりません。
  - 事前の交渉なしに、ここに列挙されていない規則のタイプのTLVレコードを`extension`に入れて送信してはなりません。
  - この仕様書内のオプションを交渉時に取り付けます。
    - 注釈付きの全てのフィールドはこのオプションをつけなければなりません。
  - カスタムメッセージを定義する場合:
    - 他のカスタム`type`との衝突を避けるため、ランダムに`type`を選ぶべきです。
    - [このIssue](https://github.com/lightningnetwork/lightning-rfc/issues/716) で列挙されているほかの実験と競合しないように`type`を選ぶべきです。
    - 通常のノードが追加データを無視すべき場合には、奇数の`type`を選ぶべきです。
    - 通常のノードがメッセージを拒否し、接続を閉じるべき場合には、偶数の`type`を選ぶべきです。

受信ノード:
  - 受信したメッセージが未知の _odd(奇数)_ タイプの場合:
    - 受信したメッセージは無視されなければなりません。
  - 受信したメッセージが未知の _even(偶数)_ タイプの場合:
    - 接続を閉じなければなりません。
    - Channelを失敗させる(閉じる)ほうが良いです。
  - 受信したメッセージが既知のメッセージで、長さが不十分な内容である場合:
    - 接続を閉じなければなりません。
    - Channelを失敗させる(閉じる)ほうが良いです。
  - 受信したメッセージが、`extension`を含む場合:
    - `extension`は無視した方が良いです。
    - 無視しない場合で、`extension`が無効な場合:
      - 接続を閉じなければなりません。
      - Channelを失敗させる(閉じる)ほうが良いです。

### 理論的根拠

`SHA2`の標準とBitcoinの公開鍵は両方ともビッグエンディアンでエンコードされています。
したがって、ほかのフィールドに別のエンディアンを使うことはめったにありません。

メッセージの長さは、暗号化ラッピングにより、65535バイトに制限されています。
いかなる場合でもプロトコル内のメッセージの長さがこの制限を超えることはありません。

_It's ok to be odd(奇数なら大丈夫)_ ルールはクライアント内での交渉や、特殊なコーディングなしに将来的なオプション拡張を実装可能にします。
_extension_ フィールドは同様に、送信者が追加TLVデータに将来的な拡張機能を含めることで使用可能にします。
なお、_extension_ フィールドは、`payload` フィールドが、メッセージの最大長である65535バイトを使い切っていない時にのみ追加できることに注意してください。

実装では、メッセージデータを8バイトごとに整列する(分ける?)ことが好まれるかもしれません(ここでは、あらゆるタイプの普通の整列要件の最大値)。
ただし、_type_ フィールドの後に6バイトのパディング(0?)を追加することは無駄であると見なされました。
整列は、6バイトのプレパディングを使用してメッセージをバッファに復号化することで実現できます。

## Type-Length-Value Format

Throughout the protocol, a TLV (Type-Length-Value) format is used to allow for
the backwards-compatible addition of new fields to existing message types.

A `tlv_record` represents a single field, encoded in the form:

* [`bigsize`: `type`]
* [`bigsize`: `length`]
* [`length`: `value`]

A `tlv_stream` is a series of (possibly zero) `tlv_record`s, represented as the
concatenation of the encoded `tlv_record`s. When used to extend existing
messages, a `tlv_stream` is typically placed after all currently defined fields.

The `type` is encoded using the BigSize format. It functions as a
message-specific, 64-bit identifier for the `tlv_record` determining how the
contents of `value` should be decoded. `type` identifiers below 2^16 are
reserved for use in this specification. `type` identifiers greater than or equal
to 2^16 are available for custom records. Any record not defined in this
specification is considered a custom record. This includes experimental and
application-specific messages.

The `length` is encoded using the BigSize format signaling the size of
`value` in bytes.

The `value` depends entirely on the `type`, and should be encoded or decoded
according to the message-specific format determined by `type`.

### Requirements

The sending node:
 - MUST order `tlv_record`s in a `tlv_stream` by strictly-increasing `type`,
   hence MUST not produce more than a single TLV record with the same `type`
 - MUST minimally encode `type` and `length`.
 - When defining custom record `type` identifiers:
   - SHOULD pick random `type` identifiers to avoid collision with other
     custom types.
   - SHOULD pick odd `type` identifiers when regular nodes should ignore the
     additional data.
   - SHOULD pick even `type` identifiers when regular nodes should reject the
     full tlv stream containing the custom record.
 - SHOULD NOT use redundant, variable-length encodings in a `tlv_record`.

The receiving node:
 - if zero bytes remain before parsing a `type`:
   - MUST stop parsing the `tlv_stream`.
 - if a `type` or `length` is not minimally encoded:
   - MUST fail to parse the `tlv_stream`.
 - if decoded `type`s are not strictly-increasing (including situations when
   two or more occurrences of the same `type` are met):
   - MUST fail to parse the `tlv_stream`.
 - if `length` exceeds the number of bytes remaining in the message:
   - MUST fail to parse the `tlv_stream`.
 - if `type` is known:
   - MUST decode the next `length` bytes using the known encoding for `type`.
   - if `length` is not exactly equal to that required for the known encoding for `type`:
     - MUST fail to parse the `tlv_stream`.
   - if variable-length fields within the known encoding for `type` are not minimal:
     - MUST fail to parse the `tlv_stream`.
 - otherwise, if `type` is unknown:
   - if `type` is even:
     - MUST fail to parse the `tlv_stream`.
   - otherwise, if `type` is odd:
     - MUST discard the next `length` bytes.

### Rationale

The primary advantage in using TLV is that a reader is able to ignore new fields
that it does not understand, since each field carries the exact size of the
encoded element. Without TLV, even if a node does not wish to use a particular
field, the node is forced to add parsing logic for that field in order to
determine the offset of any fields that follow.

The strict monotonicity constraint ensures that all `type`s are unique and can
appear at most once. Fields that map to complex objects, e.g. vectors, maps, or
structs, should do so by defining the encoding such that the object is
serialized within a single `tlv_record`. The uniqueness constraint, among other
things, enables the following optimizations:
 - canonical ordering is defined independent of the encoded `value`s.
 - canonical ordering can be known at compile-time, rather than being determined
   dynamically at the time of encoding.
 - verifying canonical ordering requires less state and is less-expensive.
 - variable-size fields can reserve their expected size up front, rather than
   appending elements sequentially and incurring double-and-copy overhead.

The use of a bigsize for `type` and `length` permits a space savings for small
`type`s or short `value`s. This potentially leaves more space for application
data over the wire or in an onion payload.

All `type`s must appear in increasing order to create a canonical encoding of
the underlying `tlv_record`s. This is crucial when computing signatures over a
`tlv_stream`, as it ensures verifiers will be able to recompute the same message
digest as the signer. Note that the canonical ordering over the set of fields
can be enforced even if the verifier does not understand what the fields
contain.

Writers should avoid using redundant, variable-length encodings in a
`tlv_record` since this results in encoding the length twice and complicates
computing the outer length. As an example, when writing a variable length byte
array, the `value` should contain only the raw bytes and forgo an additional
internal length since the `tlv_record` already carries the number of bytes that
follow. On the other hand, if a `tlv_record` contains multiple, variable-length
elements then this would not be considered redundant, and is needed to allow the
receiver to parse individual elements from `value`.

## Fundamental Types

Various fundamental types are referred to in the message specifications:

* `byte`: an 8-bit byte
* `u16`: a 2 byte unsigned integer
* `u32`: a 4 byte unsigned integer
* `u64`: an 8 byte unsigned integer

Inside TLV records which contain a single value, leading zeros in
integers can be omitted:

* `tu16`: a 0 to 2 byte unsigned integer
* `tu32`: a 0 to 4 byte unsigned integer
* `tu64`: a 0 to 8 byte unsigned integer

The following convenience types are also defined:

* `chain_hash`: a 32-byte chain identifier (see [BOLT #0](00-introduction.md#glossary-and-terminology-guide))
* `channel_id`: a 32-byte channel_id (see [BOLT #2](02-peer-protocol.md#definition-of-channel-id))
* `sha256`: a 32-byte SHA2-256 hash
* `signature`: a 64-byte bitcoin Elliptic Curve signature
* `point`: a 33-byte Elliptic Curve point (compressed encoding as per [SEC 1 standard](http://www.secg.org/sec1-v2.pdf#subsubsection.2.3.3))
* `short_channel_id`: an 8 byte value identifying a channel (see [BOLT #7](07-routing-gossip.md#definition-of-short-channel-id))
* `bigsize`: a variable-length, unsigned integer similar to Bitcoin's CompactSize encoding, but big-endian.  Described in [BigSize](#appendix-a-bigsize-test-vectors).

## Setup Messages

### The `init` Message

Once authentication is complete, the first message reveals the features supported or required by this node, even if this is a reconnection.

[BOLT #9](09-features.md) specifies lists of features. Each feature is generally represented by 2 bits. The least-significant bit is numbered 0, which is _even_, and the next most significant bit is numbered 1, which is _odd_.  For historical reasons, features are divided into global and local feature bitmasks.

The `features` field MUST be padded to bytes with 0s.

1. type: 16 (`init`)
2. data:
   * [`u16`:`gflen`]
   * [`gflen*byte`:`globalfeatures`]
   * [`u16`:`flen`]
   * [`flen*byte`:`features`]
   * [`init_tlvs`:`tlvs`]

1. `tlv_stream`: `init_tlvs`
2. types:
    1. type: 1 (`networks`)
    2. data:
        * [`...*chain_hash`:`chains`]


The optional `networks` indicates the chains the node is interested in.

#### Requirements

The sending node:
  - MUST send `init` as the first Lightning message for any connection.
  - MUST set feature bits as defined in [BOLT #9](09-features.md).
  - MUST set any undefined feature bits to 0.
  - SHOULD NOT set features greater than 13 in `globalfeatures`.
  - SHOULD use the minimum length required to represent the `features` field.
  - SHOULD set `networks` to all chains it will gossip or open channels for.

The receiving node:
  - MUST wait to receive `init` before sending any other messages.
  - MUST combine (logical OR) the two feature bitmaps into one logical `features` map.
  - MUST respond to known feature bits as specified in [BOLT #9](09-features.md).
  - upon receiving unknown _odd_ feature bits that are non-zero:
    - MUST ignore the bit.
  - upon receiving unknown _even_ feature bits that are non-zero:
    - MUST fail the connection.
  - upon receiving `networks` containing no common chains
    - MAY fail the connection.
  - if the feature vector does not set all known, transitive dependencies:
    - MUST fail the connection.

#### Rationale

There used to be two feature bitfields here, but for backwards compatibility they're now
combined into one.

This semantic allows both future incompatible changes and future backward compatible changes. Bits should generally be assigned in pairs, in order that optional features may later become compulsory.

Nodes wait for receipt of the other's features to simplify error
diagnosis when features are incompatible.

Since all networks share the same port, but most implementations only
support a single network, the `networks` fields avoids nodes
erroneously believing they will receive updates about their preferred
network, or that they can open channels.

### The `error` Message

For simplicity of diagnosis, it's often useful to tell a peer that something is incorrect.

1. type: 17 (`error`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u16`:`len`]
   * [`len*byte`:`data`]

The 2-byte `len` field indicates the number of bytes in the immediately following field.

#### Requirements

The channel is referred to by `channel_id`, unless `channel_id` is 0 (i.e. all bytes are 0), in which case it refers to all channels.

The funding node:
  - for all error messages sent before (and including) the `funding_created` message:
    - MUST use `temporary_channel_id` in lieu of `channel_id`.

The fundee node:
  - for all error messages sent before (and not including) the `funding_signed` message:
    - MUST use `temporary_channel_id` in lieu of `channel_id`.

A sending node:
  - when sending `error`:
    - MUST fail the channel referred to by the error message.
  - SHOULD send `error` for protocol violations or internal errors that make channels unusable or that make further communication unusable.
  - SHOULD send `error` with the unknown `channel_id` in reply to messages of type `32`-`255` related to unknown channels.
  - MAY send an empty `data` field.
  - when failure was caused by an invalid signature check:
    - SHOULD include the raw, hex-encoded transaction in reply to a `funding_created`, `funding_signed`, `closing_signed`, or `commitment_signed` message.
  - when `channel_id` is 0:
    - MUST fail all channels with the receiving node.
    - MUST close the connection.
  - MUST set `len` equal to the length of `data`.

The receiving node:
  - upon receiving `error`:
    - MUST fail the channel referred to by the error message, if that channel is with the sending node.
  - if no existing channel is referred to by the message:
    - MUST ignore the message.
  - MUST truncate `len` to the remainder of the packet (if it's larger).
  - if `data` is not composed solely of printable ASCII characters (For reference: the printable character set includes byte values 32 through 126, inclusive):
    - SHOULD NOT print out `data` verbatim.

#### Rationale

There are unrecoverable errors that require an abort of conversations;
if the connection is simply dropped, then the peer may retry the
connection. It's also useful to describe protocol violations for
diagnosis, as this indicates that one peer has a bug.

It may be wise not to distinguish errors in production settings, lest
it leak information — hence, the optional `data` field.

## Control Messages

### The `ping` and `pong` Messages

In order to allow for the existence of long-lived TCP connections, at
times it may be required that both ends keep alive the TCP connection at the
application level. Such messages also allow obfuscation of traffic patterns.

1. type: 18 (`ping`)
2. data:
    * [`u16`:`num_pong_bytes`]
    * [`u16`:`byteslen`]
    * [`byteslen*byte`:`ignored`]

The `pong` message is to be sent whenever a `ping` message is received. It
serves as a reply and also serves to keep the connection alive, while
explicitly notifying the other end that the receiver is still active. Within
the received `ping` message, the sender will specify the number of bytes to be
included within the data payload of the `pong` message.

1. type: 19 (`pong`)
2. data:
    * [`u16`:`byteslen`]
    * [`byteslen*byte`:`ignored`]

#### Requirements

A node sending a `ping` message:
  - SHOULD set `ignored` to 0s.
  - MUST NOT set `ignored` to sensitive data such as secrets or portions of initialized
memory.
  - if it doesn't receive a corresponding `pong`:
    - MAY terminate the network connection,
      - and MUST NOT fail the channels in this case.
  - SHOULD NOT send `ping` messages more often than once every 30 seconds.

A node sending a `pong` message:
  - SHOULD set `ignored` to 0s.
  - MUST NOT set `ignored` to sensitive data such as secrets or portions of initialized
 memory.

A node receiving a `ping` message:
  - SHOULD fail the channels if it has received significantly in excess of one `ping` per 30 seconds.
  - if `num_pong_bytes` is less than 65532:
    - MUST respond by sending a `pong` message, with `byteslen` equal to `num_pong_bytes`.
  - otherwise (`num_pong_bytes` is **not** less than 65532):
    - MUST ignore the `ping`.

A node receiving a `pong` message:
  - if `byteslen` does not correspond to any `ping`'s `num_pong_bytes` value it has sent:
    - MAY fail the channels.

### Rationale

The largest possible message is 65535 bytes; thus, the maximum sensible `byteslen`
is 65531 — in order to account for the type field (`pong`) and the `byteslen` itself. This allows
a convenient cutoff for `num_pong_bytes` to indicate that no reply should be sent.

Connections between nodes within the network may be long lived, as payment
channels have an indefinite lifetime. However, it's likely that
no new data will be
exchanged for a
significant portion of a connection's lifetime. Also, on several platforms it's possible that Lightning
clients will be put to sleep without prior warning. Hence, a
distinct `ping` message is used, in order to probe for the liveness of the connection on
the other side, as well as to keep the established connection active.

Additionally, the ability for a sender to request that the receiver send a
response with a particular number of bytes enables nodes on the network to
create _synthetic_ traffic. Such traffic can be used to partially defend
against packet and timing analysis — as nodes can fake the traffic patterns of
typical exchanges without applying any true updates to their respective
channels.

When combined with the onion routing protocol defined in
[BOLT #4](04-onion-routing.md),
careful statistically driven synthetic traffic can serve to further bolster the
privacy of participants within the network.

Limited precautions are recommended against `ping` flooding, however some
latitude is given because of network delays. Note that there are other methods
of incoming traffic flooding (e.g. sending _odd_ unknown message types, or padding
every message maximally).

Finally, the usage of periodic `ping` messages serves to promote frequent key
rotations as specified within [BOLT #8](08-transport.md).

## Appendix A: BigSize Test Vectors

The following test vectors can be used to assert the correctness of a BigSize
implementation used in the TLV format. The format is identical to the
CompactSize encoding used in bitcoin, but replaces the little-endian encoding of
multi-byte values with big-endian.

Values encoded with BigSize will produce an encoding of either 1, 3, 5, or 9
bytes depending on the size of the integer. The encoding is a piece-wise
function that takes a `uint64` value `x` and produces:
```
        uint8(x)                if x < 0xfd
        0xfd + be16(uint16(x))  if x < 0x10000
        0xfe + be32(uint32(x))  if x < 0x100000000
        0xff + be64(x)          otherwise.
```

Here `+` denotes concatenation and `be16`, `be32`, and `be64` produce a
big-endian encoding of the input for 16, 32, and 64-bit integers, respectively.

A value is said to be _minimally encoded_ if it could not be encoded using
fewer bytes. For example, a BigSize encoding that occupies 5 bytes
but whose value is less than 0x10000 is not minimally encoded. All values
decoded with BigSize should be checked to ensure they are minimally encoded.

### BigSize Decoding Tests

The following is an example of how to execute the BigSize decoding tests.
```golang
func testReadBigSize(t *testing.T, test bigSizeTest) {
        var buf [8]byte 
        r := bytes.NewReader(test.Bytes)
        val, err := tlv.ReadBigSize(r, &buf)
        if err != nil && err.Error() != test.ExpErr {
                t.Fatalf("expected decoding error: %v, got: %v",
                        test.ExpErr, err)
        }

        // If we expected a decoding error, there's no point checking the value.
        if test.ExpErr != "" {
                return
        }

        if val != test.Value {
                t.Fatalf("expected value: %d, got %d", test.Value, val)
        }
}
```

A correct implementation should pass against these test vectors:
```json
[
    {
        "name": "zero",
        "value": 0,
        "bytes": "00"
    },
    {
        "name": "one byte high",
        "value": 252,
        "bytes": "fc"
    },
    {
        "name": "two byte low",
        "value": 253,
        "bytes": "fd00fd"
    },
    {
        "name": "two byte high",
        "value": 65535,
        "bytes": "fdffff"
    },
    {
        "name": "four byte low",
        "value": 65536,
        "bytes": "fe00010000"
    },
    {
        "name": "four byte high",
        "value": 4294967295,
        "bytes": "feffffffff"
    },
    {
        "name": "eight byte low",
        "value": 4294967296,
        "bytes": "ff0000000100000000"
    },
    {
        "name": "eight byte high",
        "value": 18446744073709551615,
        "bytes": "ffffffffffffffffff"
    },
    {
        "name": "two byte not canonical",
        "value": 0,
        "bytes": "fd00fc",
        "exp_error": "decoded bigsize is not canonical"
    },
    {
        "name": "four byte not canonical",
        "value": 0,
        "bytes": "fe0000ffff",
        "exp_error": "decoded bigsize is not canonical"
    },
    {
        "name": "eight byte not canonical",
        "value": 0,
        "bytes": "ff00000000ffffffff",
        "exp_error": "decoded bigsize is not canonical"
    },
    {
        "name": "two byte short read",
        "value": 0,
        "bytes": "fd00",
        "exp_error": "unexpected EOF"
    },
    {
        "name": "four byte short read",
        "value": 0,
        "bytes": "feffff",
        "exp_error": "unexpected EOF"
    },
    {
        "name": "eight byte short read",
        "value": 0,
        "bytes": "ffffffffff",
        "exp_error": "unexpected EOF"
    },
    {
        "name": "one byte no read",
        "value": 0,
        "bytes": "",
        "exp_error": "EOF"
    },
    {
        "name": "two byte no read",
        "value": 0,
        "bytes": "fd",
        "exp_error": "unexpected EOF"
    },
    {
        "name": "four byte no read",
        "value": 0,
        "bytes": "fe",
        "exp_error": "unexpected EOF"
    },
    {
        "name": "eight byte no read",
        "value": 0,
        "bytes": "ff",
        "exp_error": "unexpected EOF"
    }
]
```

### BigSize Encoding Tests

The following is an example of how to execute the BigSize encoding tests.
```golang
func testWriteBigSize(t *testing.T, test bigSizeTest) {
        var (
                w   bytes.Buffer
                buf [8]byte
        )
        err := tlv.WriteBigSize(&w, test.Value, &buf)
        if err != nil {
                t.Fatalf("unable to encode %d as bigsize: %v",
                        test.Value, err)
        }

        if bytes.Compare(w.Bytes(), test.Bytes) != 0 {
                t.Fatalf("expected bytes: %v, got %v",
                        test.Bytes, w.Bytes())
        }
}
```

A correct implementation should pass against the following test vectors:
```json
[
    {
        "name": "zero",
        "value": 0,
        "bytes": "00"
    },
    {
        "name": "one byte high",
        "value": 252,
        "bytes": "fc"
    },
    {
        "name": "two byte low",
        "value": 253,
        "bytes": "fd00fd"
    },
    {
        "name": "two byte high",
        "value": 65535,
        "bytes": "fdffff"
    },
    {
        "name": "four byte low",
        "value": 65536,
        "bytes": "fe00010000"
    },
    {
        "name": "four byte high",
        "value": 4294967295,
        "bytes": "feffffffff"
    },
    {
        "name": "eight byte low",
        "value": 4294967296,
        "bytes": "ff0000000100000000"
    },
    {
        "name": "eight byte high",
        "value": 18446744073709551615,
        "bytes": "ffffffffffffffffff"
    }
]
```

## Appendix B: Type-Length-Value Test Vectors

The following tests assume that two separate TLV namespaces exist: n1 and n2.

The n1 namespace supports the following TLV types:

1. `tlv_stream`: `n1`
2. types:
   1. type: 1 (`tlv1`)
   2. data:
     * [`tu64`:`amount_msat`]
   1. type: 2 (`tlv2`)
   2. data:
     * [`short_channel_id`:`scid`]
   1. type: 3 (`tlv3`)
   2. data:
     * [`point`:`node_id`]
     * [`u64`:`amount_msat_1`]
     * [`u64`:`amount_msat_2`]
   1. type: 254 (`tlv4`)
   2. data:
     * [`u16`:`cltv_delta`]

The n2 namespace supports the following TLV types:

1. `tlv_stream`: `n2`
2. types:
   1. type: 0 (`tlv1`)
   2. data:
     * [`tu64`:`amount_msat`]
   1. type: 11 (`tlv2`)
   2. data:
     * [`tu32`:`cltv_expiry`]

### TLV Decoding Failures

The following TLV streams in any namespace should trigger a decoding failure:

1. Invalid stream: 0xfd
2. Reason: type truncated

1. Invalid stream: 0xfd01
2. Reason: type truncated

1. Invalid stream: 0xfd0001 00
2. Reason: not minimally encoded type

1. Invalid stream: 0xfd0101
2. Reason: missing length

1. Invalid stream: 0x0f fd
2. Reason: (length truncated)

1. Invalid stream: 0x0f fd26
2. Reason: (length truncated)

1. Invalid stream: 0x0f fd2602
2. Reason: missing value

1. Invalid stream: 0x0f fd0001 00
2. Reason: not minimally encoded length

1. Invalid stream: 0x0f fd0201 000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
2. Reason: value truncated

The following TLV streams in either namespace should trigger a
decoding failure:

1. Invalid stream: 0x12 00
2. Reason: unknown even type.

1. Invalid stream: 0xfd0102 00
2. Reason: unknown even type.

1. Invalid stream: 0xfe01000002 00
2. Reason: unknown even type.

1. Invalid stream: 0xff0100000000000002 00
2. Reason: unknown even type.

The following TLV streams in namespace `n1` should trigger a decoding
failure:

1. Invalid stream: 0x01 09 ffffffffffffffffff
2. Reason: greater than encoding length for `n1`s `tlv1`.

1. Invalid stream: 0x01 01 00
2. Reason: encoding for `n1`s `tlv1`s `amount_msat` is not minimal

1. Invalid stream: 0x01 02 0001
2. Reason: encoding for `n1`s `tlv1`s `amount_msat` is not minimal

1. Invalid stream: 0x01 03 000100
2. Reason: encoding for `n1`s `tlv1`s `amount_msat` is not minimal

1. Invalid stream: 0x01 04 00010000
2. Reason: encoding for `n1`s `tlv1`s `amount_msat` is not minimal

1. Invalid stream: 0x01 05 0001000000
2. Reason: encoding for `n1`s `tlv1`s `amount_msat` is not minimal

1. Invalid stream: 0x01 06 000100000000
2. Reason: encoding for `n1`s `tlv1`s `amount_msat` is not minimal

1. Invalid stream: 0x01 07 00010000000000
2. Reason: encoding for `n1`s `tlv1`s `amount_msat` is not minimal

1. Invalid stream: 0x01 08 0001000000000000
2. Reason: encoding for `n1`s `tlv1`s `amount_msat` is not minimal

1. Invalid stream: 0x02 07 01010101010101
2. Reason: less than encoding length for `n1`s `tlv2`.

1. Invalid stream: 0x02 09 010101010101010101
2. Reason: greater than encoding length for `n1`s `tlv2`.

1. Invalid stream: 0x03 21 023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb
2. Reason: less than encoding length for `n1`s `tlv3`.

1. Invalid stream: 0x03 29 023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb0000000000000001
2. Reason: less than encoding length for `n1`s `tlv3`.

1. Invalid stream: 0x03 30 023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb000000000000000100000000000001
2. Reason: less than encoding length for `n1`s `tlv3`.

1. Invalid stream: 0x03 31 043da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb00000000000000010000000000000002
2. Reason: `n1`s `node_id` is not a valid point.

1. Invalid stream: 0x03 32 023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb0000000000000001000000000000000001
2. Reason: greater than encoding length for `n1`s `tlv3`.

1. Invalid stream: 0xfd00fe 00
2. Reason: less than encoding length for `n1`s `tlv4`.

1. Invalid stream: 0xfd00fe 01 01
2. Reason: less than encoding length for `n1`s `tlv4`.

1. Invalid stream: 0xfd00fe 03 010101
2. Reason: greater than encoding length for `n1`s `tlv4`.

1. Invalid stream: 0x00 00
2. Reason: unknown even field for `n1`s namespace.

### TLV Decoding Successes

The following TLV streams in either namespace should correctly decode,
and be ignored:

1. Valid stream: 0x
2. Explanation: empty message

1. Valid stream: 0x21 00
2. Explanation: Unknown odd type.

1. Valid stream: 0xfd0201 00
2. Explanation: Unknown odd type.

1. Valid stream: 0xfd00fd 00
2. Explanation: Unknown odd type.

1. Valid stream: 0xfd00ff 00
2. Explanation: Unknown odd type.

1. Valid stream: 0xfe02000001 00
2. Explanation: Unknown odd type.

1. Valid stream: 0xff0200000000000001 00
2. Explanation: Unknown odd type.

The following TLV streams in `n1` namespace should correctly decode,
with the values given here:

1. Valid stream: 0x01 00
2. Values: `tlv1` `amount_msat`=0

1. Valid stream: 0x01 01 01
2. Values: `tlv1` `amount_msat`=1

1. Valid stream: 0x01 02 0100
2. Values: `tlv1` `amount_msat`=256

1. Valid stream: 0x01 03 010000
2. Values: `tlv1` `amount_msat`=65536

1. Valid stream: 0x01 04 01000000
2. Values: `tlv1` `amount_msat`=16777216

1. Valid stream: 0x01 05 0100000000
2. Values: `tlv1` `amount_msat`=4294967296

1. Valid stream: 0x01 06 010000000000
2. Values: `tlv1` `amount_msat`=1099511627776

1. Valid stream: 0x01 07 01000000000000
2. Values: `tlv1` `amount_msat`=281474976710656

1. Valid stream: 0x01 08 0100000000000000
2. Values: `tlv1` `amount_msat`=72057594037927936

1. Valid stream: 0x02 08 0000000000000226
2. Values: `tlv2` `scid`=0x0x550

1. Valid stream: 0x03 31 023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb00000000000000010000000000000002
2. Values: `tlv3` `node_id`=023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb `amount_msat_1`=1 `amount_msat_2`=2

1. Valid stream: 0xfd00fe 02 0226
2. Values: `tlv4` `cltv_delta`=550

### TLV Stream Decoding Failure

Any appending of an invalid stream to a valid stream should trigger
a decoding failure.

Any appending of a higher-numbered valid stream to a lower-numbered
valid stream should not trigger a decoding failure.

In addition, the following TLV streams in namespace `n1` should
trigger a decoding failure:

1. Invalid stream: 0x02 08 0000000000000226 01 01 2a
2. Reason: valid TLV records but invalid ordering

1. Invalid stream: 0x02 08 0000000000000231 02 08 0000000000000451
2. Reason: duplicate TLV type

1. Invalid stream: 0x1f 00 0f 01 2a
2. Reason: valid (ignored) TLV records but invalid ordering

1. Invalid stream: 0x1f 00 1f 01 2a
2. Reason: duplicate TLV type (ignored)

The following TLV stream in namespace `n2` should trigger a decoding
failure:

1. Invalid stream: 0xffffffffffffffffff 00 00 00
2. Reason: valid TLV records but invalid ordering

## Appendix C: Message Extension

This section contains examples of valid and invalid extensions on the `init`
message. The base `init` message (without extensions) for these examples is
`0x001000000000` (all features turned off).

The following `init` messages are valid:

- `0x001000000000`: no extension provided
- `0x00100000000001012a030104`: the extension contains two _odd_ TLV records (with types `0x01` and `0x03`)

The following `init` messages are invalid:

- `0x00100000000001`: the extension is present but truncated
- `0x00100000000002012a`: the extension contains unknown _even_ TLV records (assuming that TLV type `0x02` is unknown)
- `0x001000000000010101010102`: the extension TLV stream is invalid (duplicate TLV record type `0x01`)

Note that when messages are signed, the _extension_ is part of the signed bytes.
Nodes should store the _extension_ bytes even if they don't understand them to
be able to correctly verify signatures.

## Acknowledgments

[ TODO: (roasbeef); fin ]

## References

1. <a id="reference-1">http://www.unicode.org/charts/PDF/U2600.pdf</a>

## Authors

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
