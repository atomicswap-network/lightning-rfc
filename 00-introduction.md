# BOLT #0: Lightning Networkの紹介と目次

皆さんこんにちは！
このドキュメントは、Bitcoinのオフチェーン取引(ブロックチェーン外の高速な取引)を行え、必要な時はオンチェーン取引を頼って実行される、
Lightning Networkというレイヤ2プロトコルに関して説明しているドキュメントです。通称「Basis of Lightning Technology(BOLT)」といいます。

一部の要件はやや不確実です。私たちは動機と結果に基づく推論を強調してここに記そうとしました。
何か分かりにくいところや間違っているところがありましたら、ご連絡をいただき、改善にお役立てください。

このドキュメントはバージョン0です

1. [BOLT #1](01-messaging.md): Lightning Networkにおける基盤プロトコル
2. [BOLT #2](02-peer-protocol.md): Channel管理のためのピアプロトコル
3. [BOLT #3](03-transactions.md): Bitcoinの取引とScript形式
4. [BOLT #4](04-onion-routing.md): Onionルーティングプロトコル(Torネットワーク関連)
5. [BOLT #5](05-onchain.md): オンチェーン取引の取り扱いについてのおすすめ
7. [BOLT #7](07-routing-gossip.md): P2PノードとChannelの発見
8. [BOLT #8](08-transport.md): 暗号化と認証済みトランスポート
9. [BOLT #9](09-features.md): 割り当て済み特徴フラグ
10. [BOLT #10](10-dns-bootstrap.md): DNSブートストラップとアシストノードの場所
11. [BOLT #11](11-payment-encoding.md): Lightning支払いにおける請求プロトコル

## スパーク: Lightning Networkの簡易的な紹介

Lightning NetworkはChannelを用いたネットワークでBitcoinの高速な支払いを行うためのプロトコルです。

### Channels

Lightning Networkは*Channel*を確立することで機能します。
2人の参加者がLightning Networkの支払いChannelを生成し、
そこにBitcoin NetworkにロックされたBitcoinを含みます(例えば、0.1BTCをNetwork上にロックします)。
これらは双方の署名(Signature)によって支払い可能になります。

最初に、彼ら(2人)がお互いにBitcoin取引を発行し、
Lightning Network上で使用する全てのBitcoin(ここでは0.1BTCとする)を一つのパーティ(アドレス)に集約します。
これらは、後に異なった新しいBitcoin取引によって分けて署名することができます。
例えば、0.09BTCを一方のパーティに、もう一方のパーティに0.01BTCを送金することができます。
そして、以前のBitcoin取引を無効化します。

[BOLT #2: Channel Establishment](02-peer-protocol.md#channel-establishment)を見ることで、Channel確立についてさらに詳しく知ることが出来ます。
また、[BOLT #3: Funding Transaction Output](03-transactions.md#funding-transaction-output)を見ることでChannelを作るにあたってのBitcoin取引形式について知ること出来ます。
更に、[BOLT #5: オンチェーン取引の取り扱いについてのおすすめ](05-onchain.md)を見ることで参加者が拒否したり失敗したりした時の要件や、
Bitcoin取引を使うためのクロス署名について知ることができます。

### 条件付き支払い(Conditional Payment)

Lightning NetworkのChannelは参加者2人の間での支払いのみ可能です。
しかし、Channel同士を接続することで、ネットワークに参加する全ての人との支払いを可能にするネットワークを形成できます。
この際に要求される技術が条件付き支払い(Conditional Payment)です。
例えば、「あなたが6時間以内にSecretを明かせば0.01BTCがもらえます」といった具合に。
この条件に乗った人(受信者といいます)がsecretを明かすと、Bitcoin取引の条件付き支払いが受信者あての支払いに変換されます。

参加者が条件付き支払いを追加するために使用するコマンドは[BOLT #2: Adding an HTLC](02-peer-protocol.md#adding-an-htlc-update_add_htlc)をご覧ください。
また、[BOLT #3: Commitment Transaction](03-transactions.md#commitment-transaction)では完了した際のBitcoin取引のフォーマットについて記載しています。

### Forwarding

上記のような条件付き支払いは安全に、そしてより短いタイムリミットで参加者に転送されなければなりません。
「あなたが5時間以内にSecretを明かせば0.01BTCがもらえます」のような具合に。
これにより、仲介者の信頼なしにChannelをネットワークにつなぐことができます。

[BOLT #4: Packet Structure](04-onion-routing.md#packet-structure) for how payment instructions are transported.
[BOLT #2: Forwarding HTLCs](02-peer-protocol.md#forwarding-htlcs)を見れば、支払いの転送の詳細を知ることができます。
また、支払い指示の転送方法・形式については[BOLT #4: Packet Structure](04-onion-routing.md#packet-structure)をご覧ください。

### Network Topology

支払いを作成するには、参加者はどのChannelを使えば送信できるのか知っておく必要があります。
参加者は他の人にChannelとNodeの作成について尋ね、更新します。

[BOLT #7: P2PノードとChannelの発見](07-routing-gossip.md)では、通信プロトコルに関する詳細を知ることができます。
また、[BOLT #10: DNSブートストラップとアシストノードの場所](10-dns-bootstrap.md)で最初のネットワークブートストラップについて説明しています。

### 支払い請求(Payment Invoicing)

参加者はどのように支払いを行えばよいかを示す請求を受け取ります。

[BOLT #11: Lightning支払いにおける請求プロトコル](11-payment-encoding.md)では、支払者が後で支払いの成功を証明できるように、
支払いの宛先と目的を説明するプロトコルについて説明しています。

## Glossary and Terminology Guide

* #### *Announcement*:
   * A gossip message sent between *[peers](#peers)* intended to aid the discovery of a *[channel](#channel)* or a *[node](#node)*.

* #### `chain_hash`:
   * The uniquely identifying hash of the target blockchain (usually the genesis hash).
     This allows *[nodes](#node)* to create and reference *channels* on
     several blockchains. Nodes are to ignore any messages that reference a
     `chain_hash` that are unknown to them. Unlike `bitcoin-cli`, the hash is
     not reversed but is used directly.

     For the main chain Bitcoin blockchain, the `chain_hash` value MUST be
     (encoded in hex):
     `6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000`.

* #### *Channel*:
   * A fast, off-chain method of mutual exchange between two *[peers](#peers)*.
   To transact funds, peers exchange signatures to create an updated *[commitment transaction](#commitment-transaction)*.
   * _See closure methods: [mutual close](#mutual-close), [revoked transaction close](#revoked-transaction-close), [unilateral close](#unilateral-close)_
   * _See related: [route](#route)_

* #### *Closing transaction*:
   * A transaction generated as part of a *[mutual close](#mutual-close)*. A closing transaction is similar to a _commitment transaction_, but with no pending payments.
   * _See related: [commitment transaction](#commitment-transaction), [funding transaction](#funding-transaction), [penalty transaction](#penalty-transaction)_

* #### *Commitment number*:
   * A 48-bit incrementing counter for each *[commitment transaction](#commitment-transaction)*; counters
    are independent for each *peer* in the *channel* and start at 0.
   * _See container: [commitment transaction](#commitment-transaction)_
   * _See related: [closing transaction](#closing-transaction), [funding transaction](#funding-transaction), [penalty transaction](#penalty-transaction)_

* #### *Commitment revocation private key*:
   * Every *[commitment transaction](#commitment-transaction)* has a unique commitment revocation private-key
    value that allows the other *peer* to spend all outputs
    immediately: revealing this key is how old commitment
    transactions are revoked. To support revocation, each output of the
    commitment transaction refers to the commitment revocation public key.
   * _See container: [commitment transaction](#commitment-transaction)_
   * _See originator: [per-commitment secret](#per-commitment-secret)_

* #### *Commitment transaction*:
   * A transaction that spends the *[funding transaction](#funding-transaction)*.
   Each *peer* holds the other peer's signature for this transaction, so that each
   always has a commitment transaction that it can spend. After a new
   commitment transaction is negotiated, the old one is *revoked*.
   * _See parts: [commitment number](#commitment-number), [commitment revocation private key](#commitment-revocation-private-key), [HTLC](#HTLC-Hashed-Time-Locked-Contract), [per-commitment secret](#per-commitment-secret), [outpoint](#outpoint)_
   * _See related: [closing transaction](#closing-transaction), [funding transaction](#funding-transaction), [penalty transaction](#penalty-transaction)_
   * _See types: [revoked commitment transaction](#revoked-commitment-transaction)_

* #### *Final node*:
   * The final recipient of a packet that is routing a payment from an *[origin node](#origin-node)* through some number of *[hops](#hop)*. It is also the final *[receiving peer](#receiving-peer)* in a chain.
   * _See category: [node](#node)_
   * _See related: [origin node](#origin-node), [processing node](#processing-node)_

* #### *Funding transaction*:
   * An irreversible on-chain transaction that pays to both *[peers](#peers)* on a *[channel](#channel)*.
   It can only be spent by mutual consent.
   * _See related: [closing transaction](#closing-transaction), [commitment transaction](#commitment-transaction), [penalty transaction](#penalty-transaction)_

* #### *Hop*:
   * A *[node](#node)*. Generally, an intermediate node lying between an *[origin node](#origin-node)* and a *[final node](#final-node)*.
   * _See category: [node](#node)_

* #### *HTLC*: Hashed Time Locked Contract.
   * A conditional payment between two *[peers](#peers)*: the recipient can spend
    the payment by presenting its signature and a *payment preimage*,
    otherwise the payer can cancel the contract by spending it after
    a given time. These are implemented as outputs from the
    *[commitment transaction](#commitment-transaction)*.
   * _See container: [commitment transaction](#commitment-transaction)_
   * _See parts: [Payment hash](#Payment-hash), [Payment preimage](#Payment-preimage)_

* #### *Invoice*: A request for funds on the Lightning Network, possibly
    including payment type, payment amount, expiry, and other
    information. This is how payments are made on the Lightning
    Network, rather than using Bitcoin-style addresses.

* #### *It's ok to be odd*:
   * A rule applied to some numeric fields that indicates either optional or
     compulsory support for features. Even numbers indicate that both endpoints
     MUST support the feature in question, while odd numbers indicate
     that the feature MAY be disregarded by the other endpoint.

* #### *MSAT*:
   * A millisatoshi, often used as a field name.

* #### *Mutual close*:
   * A cooperative close of a *[channel](#channel)*, accomplished by broadcasting an unconditional
    spend of the *[funding transaction](#funding-transaction)* with an output to each *peer*
    (unless one output is too small, and thus is not included).
   * _See related: [revoked transaction close](#revoked-transaction-close), [unilateral close](#unilateral-close)_

* #### *Node*:
   * A computer or other device that is part of the Lightning network.
   * _See related: [peers](#peers)_
   * _See types: [final node](#final-node), [hop](#hop), [origin node](#origin-node), [processing node](#processing-node), [receiving node](#receiving-node), [sending node](#sending-node)_

* #### *Origin node*:
   * The *[node](#node)* that originates a packet that will route a payment through some number of [hops](#hop) to a *[final node](#final-node)*. It is also the first [sending peer](#sending-peer) in a chain.
   * _See category: [node](#node)_
   * _See related: [final node](#final-node), [processing node](#processing-node)_

* #### *Outpoint*:
  * A transaction hash and output index that uniquely identify an unspent transaction output. Needed to compose a new transaction, as an input.
  * _See related: [funding transaction](#funding-transaction), [commitment transaction](#commitment-transaction)_

* #### *Payment hash*:
   * The *[HTLC](#HTLC-Hashed-Time-Locked-Contract)* contains the payment hash, which is the hash of the
    *[payment preimage](#Payment-preimage)*.
   * _See container: [HTLC](#HTLC-Hashed-Time-Locked-Contract)_
   * _See originator: [Payment preimage](#Payment-preimage)_

* #### *Payment preimage*:
   * Proof that payment has been received, held by
    the final recipient, who is the only person who knows this
    secret. The final recipient releases the preimage in order to
    release funds. The payment preimage is hashed as the *[payment hash](#Payment-hash)*
    in the *[HTLC](#HTLC-Hashed-Time-Locked-Contract)*.
   * _See container: [HTLC](#HTLC-Hashed-Time-Locked-Contract)_
   * _See derivation: [payment hash](#Payment-hash)_

* #### *Peers*:
   * Two *[nodes](#node)* that are in communication with each other.
      * Two peers may gossip with each other prior to setting up a channel.
      * Two peers may establish a *[channel](#channel)* through which they transact.
   * _See related: [node](#node)_

* #### *Penalty transaction*:
   * A transaction that spends all outputs of a *[revoked commitment
    transaction](#revoked-commitment-transaction)*, using the *commitment revocation private key*. A *[peer](#peers)* uses this
    if the other peer tries to "cheat" by broadcasting a *[revoked commitment
    transaction](#revoked-commitment-transaction)*.
   * _See related: [closing transaction](#closing-transaction), [commitment transaction](#commitment-transaction), [funding transaction](#funding-transaction)_

* #### *Per-commitment secret*:
   * Every *[commitment transaction](#commitment-transaction)* derives its keys from a per-commitment secret,
     which is generated such that the series of per-commitment secrets
     for all previous commitments can be stored compactly.
   * _See container: [commitment transaction](#commitment-transaction)_
   * _See derivation: [commitment revocation private key](#commitment-revocation-private-key)_

* #### *Processing node*:
   * A *[node](#node)* that is processing a packet that originated with an *[origin node](#origin-node)* and that is being sent toward a *[final node](#final-node)* in order to route a payment. It acts as a *[receiving peer](#receiving-peer)* to receive the message, then a [sending peer](#sending-peer) to send on the packet.
   * _See category: [node](#node)_
   * _See related: [final node](#final-node), [origin node](#origin-node)_

* #### *Receiving node*:
   * A *[node](#node)* that is receiving a message.
   * _See category: [node](#node)_
   * _See related: [sending node](#sending-node)_

* #### *Receiving peer*:
   * A *[node](#node)* that is receiving a message from a directly connected *peer*.
   * _See category: [peer](#Peers)_
   * _See related: [sending peer](#sending-peer)_

* #### *Revoked commitment transaction*:
   * An old *[commitment transaction](#commitment-transaction)* that has been revoked because a new commitment transaction has been negotiated.
   * _See category: [commitment transaction](#commitment-transaction)_

* #### *Revoked transaction close*:
   * An invalid close of a *[channel](#channel)*, accomplished by broadcasting a *revoked
    commitment transaction*. Since the other *peer* knows the
    *commitment revocation secret key*, it can create a *[penalty transaction](#penalty-transaction)*.
   * _See related: [mutual close](#mutual-close), [unilateral close](#unilateral-close)_

* #### *Route*: 
  * A path across the Lightning Network that enables a payment
    from an *origin node* to a *[final node](#final-node)* across one or more
    *[hops](#hop)*.
  * _See related: [channel](#channel)_

* #### *Sending node*:
   * A *[node](#node)* that is sending a message.
   * _See category: [node](#node)_
   * _See related: [receiving node](#receiving-node)_

* #### *Sending peer*:
   * A *[node](#node)* that is sending a message to a directly connected *peer*.
   * _See category: [peer](#Peers)_
   * _See related: [receiving peer](#receiving-peer)_.

* #### *Unilateral close*:
   * An uncooperative close of a *[channel](#channel)*, accomplished by broadcasting a
    *[commitment transaction](#commitment-transaction)*. This transaction is larger (i.e. less
    efficient) than a *[closing transaction](#closing-transaction)*, and the *[peer](#peers)* whose
    commitment is broadcast cannot access its own outputs for some
    previously-negotiated duration.
   * _See related: [mutual close](#mutual-close), [revoked transaction close](#revoked-transaction-close)_

## Theme Song

      Why this network could be democratic...
      Numismatic...
      Cryptographic!
      Why it could be released Lightning!
      (Release Lightning!)


      We'll have some timelocked contracts with hashed pubkeys, oh yeah.
      (Keep talking, whoa keep talkin')
      We'll segregate the witness for trustless starts, oh yeah.
      (I'll get the money, I've got to get the money)
      With dynamic onion routes, they'll be shakin' in their boots;
      You know that's just the truth, we'll be scaling through the roof.
      Release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)


      [Chorus:]
      Oh released Lightning, it's better than a debit card..
      (Release Lightning, go release Lightning!)
      With released Lightning, micropayments just ain't hard...
      (Release Lightning, go release Lightning!)
      Then kaboom: we'll hit the moon -- release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)

 
      We'll have QR codes, and smartphone apps, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      P2P messaging, and passive incomes, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      Outsourced closure watch, gives me feelings in my crotch.
      You'll know it's not a brag when the repo gets a tag:
      Released Lightning.


      [Chorus]
      [Instrumental, ~1m10s]
      [Chorus]
      (Lightning! Lightning! Lightning! Lightning!
       Lightning! Lightning! Lightning! Lightning!)


      C'mon guys, let's get to work!


   -- Anthony Towns <aj@erisian.com.au>

## Authors

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
