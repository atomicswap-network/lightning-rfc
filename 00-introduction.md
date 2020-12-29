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
この条件に乗った人(受信者といいます)がSecretを明かすと、Bitcoin取引の条件付き支払いが受信者あての支払いに変換されます。

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

## 技術用語集・用語ガイド
元の用語を出来る限りそのまま掲載しています。
基本的に説明文の頭に用語自体の日本語訳をつけています。

* #### *Announcement*:
   * アナウンス、*[Peer](#peer)* 同士で送受信される伝言(Gossip Message)で、
     *[Channel](#channel)* や *[Node](#node)* を発見するのを助けることを意図しています。.

* #### `chain_hash`:
   * ユニーク(被ってはならない)なハッシュで、ブロックチェーンを識別するのに用います(一般的にGenesis Blockのハッシュを用います)。
     これにより、*[Node](#node)* は複数のブロックチェーン上で、*Channel* を作成・参照することができます。
     Nodeは、未知の(別の)`chain_hash`を参照するメッセージを無視します。
     `bitcoin-cli`とは違い、ハッシュは反転させずにそのまま使用します。

     Bitcoinのメインチェーンの場合、`chain_hash`は
     `6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000`
     になります(hexでエンコード済み)。
     

* #### *Channel*:
   * 相互に通貨を交換し合う2 *[Peer](#peer)* 間で開設される高速なオフチェーンの送金方法です。
   資産を取引する際、Peerは署名を交換して更新済みの *[Commitment Transaction](#commitment-transaction)* を作成します。
   * _Channelを終了するメソッド: [Mutual Close(相互終了)](#mutual-close)、[Revoked Transaction Close(取り消された取引での終了)](#revoked-transaction-close)、[Unilateral Close(一方的な終了)](#unilateral-close)_
   * _関係のある用語: [Route](#route)_

* #### *Closing Transaction*:
   * 終了時取引。*[Mutual Close(相互終了)](#mutual-close)* 時に生成される取引の一つです。
     Closing Transactionは _Commitment Transaction_ に似ていますが、保留中の支払いがありません。
   * _関連用語: [Commitment Transaction(言質取引)](#commitment-transaction)、[Funding Transaction(供給取引)](#funding-transaction)、[Penalty Transaction(罰則取引)](#penalty-transaction)_

* #### *Commitment Number*:
   * 言質取引のカウント(もしくはカウンター)。48ビットの値が増え続ける(減ることのない)カウンターで、互いの *[Commitment Transaction](#commitment-transaction)* の数をカウントしています。
     カウンターは、各 *Peer* 内の *Channel* に独立して存在し、0からカウントスタートします。
   * _この用語が説明に含まれる用語(内容): [Commitment Transaction(言質取引)](#commitment-transaction)_
   * _関連用語: [Commitment Transaction(言質取引)](#commitment-transaction)、[Funding Transaction(供給取引)](#funding-transaction)、[Penalty Transaction(罰則取引)](#penalty-transaction)_

* #### *Commitment Revocation Private Key*:
   * 言質撤回秘密鍵。すべての *[Commitment Transaction(言質取引)](#commitment-transaction)* は全ての資産を他の *Peer* にすぐに送金するユニーク(被ってはならない)な言質撤回秘密鍵を持っています。
     この秘密鍵を明らかにするということは、古い言質取引を無効化するということです。
     この撤回をサポートするために、各言質取引で言質撤回公開鍵を参照します(取引の出力内のスクリプトに含まれます)。
   * _この用語が説明に含まれる用語(内容): [Commitment Transaction(言質取引)](#commitment-transaction)_
   * _原案の用語: [Per-commitment Secret(言質ごとのシークレット)](#per-commitment-secret)_

* #### *Commitment Transaction*:
   * 言質取引。*[Funding Transaction(供給取引)](#funding-transaction)* での資産を使用する取引です。
   各 *Peer* はこの取引に対する他のPeerの署名を保持します。 これにより、各Peerは常に使用可能な言質取引を持つことになります。
   新しい言質取引が交渉された後、古い言質取引は破棄されます。  
   * _他のパート: [Commitment Number(言質取引カウント)](#commitment-number)、[Commitment Revocation Private Key(言質撤回秘密鍵)](#commitment-revocation-private-key)、[HTLC](#HTLC-Hashed-Time-Locked-Contract)、[Per-commitment Secret(言質ごとのシークレット)](#per-commitment-secret)、[outpoint](#outpoint)_
   * _関連用語: [Closing Transaction(終了時取引)](#closing-transaction)、[Funding Transaction(供給取引)](#funding-transaction)、[Penalty Transaction(罰則取引)](#penalty-transaction)_
   * _他のタイプ: [Revoked Commitment Transaction(無効化済み言質取引)](#revoked-commitment-transaction)_

* #### *Final Node*:
   * 終点ノード。*[Origin Node(起点ノード)](#origin-node)* からいくつかの *[Hop](#hop)* を経由して支払いされるパケットの最終受信者のことを指します。
     また、チェーンの最後の *[Receiving Peer(受信ピア)](#receiving-peer)* でもあります。
   * _関連カテゴリ: [Node](#node)_
   * _関連用語: [Origin Node(起点ノード)](#origin-node)、[Processing Node(処理ノード)](#processing-node)_

* #### *Funding Transaction*:
   * 供給取引。*[channel](#channel)* 上の両方の *[Peer](#peer)* に支払う不可逆的なオンチェーン取引のことを指します。
   これは双方の同意がある時のみ可能です。  
   * _関連用語: [Closing Transaction(終了時取引)](#closing-transaction)、[Commitment Transaction(言質取引)](#commitment-transaction)、[Penalty Transaction(罰則取引)](#penalty-transaction)_

* #### *Hop*:
   * ホップ。*[Node](#node)* のことを指します。
     一般的に、*[Origin Node(起点ノード)](#origin-node)* と *[Final Node(終点ノード)](#final-node)* をまたぐ中間ノードを指します。
   * _関連カテゴリ: [Node](#node)_

* #### *HTLC*: Hashed Time Locked Contract.
   * 時間が経たないと使えないようにロックされた契約をハッシュしたもの。条件付き支払いは2 *[Peer](#peer)* 間で行われます。
     この条件付き支払いは、受信者が署名と *Payment Preimage(支払いプレイメージ)* を提示することで支払いを行うことができます。
     そうでない場合、支払者は一定時間置いた後にそれを使うことで契約をキャンセルすることができます。
     これらは、[Commitment Transaction(言質取引)](#commitment-transaction)の出力として実装されます。
   * _この用語が説明に含まれる用語(内容): [Commitment Transaction(言質取引)](#commitment-transaction)_
   * _他のパート: [Payment Hash(支払いハッシュ)](#Payment-hash)、[Payment Preimage(支払いプレイメージ)](#Payment-preimage)_

* #### *Invoice*: 
    請求。Lightning Network上での資産リクエスト。
    支払い方法や、支払い金額、有効期限、またはその他の情報などが含まれることがあります。
    これはLightning Network上での支払い方法を表すもので、Bitcoinスタイルのアドレスではありません。

* #### *It's ok to be odd*:
   * 奇数で大丈夫。一部の数値フィールドに適用されるルールで、各機能のオプションや強制的な機能のサポートを示します。
     偶数は、両方のエンドポイントが問題の機能をサポートしなければならないことを示し、
     奇数は、もう一方のエンドポイントがその機能を無視してもよいことを示します。

* #### *MSAT*:
   * ミリサトシ(Bitcoinの最小単位サトシの1000分の1)。フィールド名として使われます。

* #### *Mutual Close*:
   * 相互終了。*[Channel](#channel)* を共同して終了することを指します。
   *[Funding Transaction(供給取引)](#funding-transaction)* の出力を各 *Peer* への無条件支払いにしてブロードキャストすることで達成されます
    (1つの出力が小さすぎて含まれない場合を除く)。
   * _関連用語: [Revoked Transaction Close(取り消された取引での終了)](#revoked-transaction-close)、[Unilateral Close(一方的な終了)](#unilateral-close)_

* #### *Node*:
   * ノード。コンピュータやその他のデバイスで、Lightning Networkを構成するものです。
   * _関連用語: [Peer](#peer)_
   * _他のタイプ: [Final Node(終点ノード)](#final-node)、[Hop](#hop)、[Origin Node(起点ノード)](#origin-node)、[Processing Node(処理ノード)](#processing-node)、[Receiving Node(受信ノード)](#receiving-node), [Sending Node(送信ノード)](#sending-node)_

* #### *Origin Node*:
   * 起点ノード。この *[Node](#node)* は支払いされるパケットの起点で、いくつかの [Hop](#hop) を使って *[Final Node(終点ノード)](#final-node)* に届けられます。
     また、チェーンの最初の [sending peer(送信ピア)](#sending-peer) でもあります。
   * _関連カテゴリ: [Node](#node)_
   * _関連用語: [Final Node(終点ノード)](#final-node)、[Processing Node(処理ノード)](#processing-node)_

* #### *Outpoint*:
  * アウトポイント。未使用の取引の出力を一意に識別する取引ハッシュと出力インデックスのセットです。
    新しいトランザクションを作成する場合、これを入力とする必要があります。
  * _関連用語: [Funding Transaction(供給取引)](#funding-transaction)、[Commitment Transaction(言質取引)](#commitment-transaction)_

* #### *Payment Hash*:
   * 支払いハッシュ。*[HTLC](#HTLC-Hashed-Time-Locked-Contract)* には、*[Payment Preimage(支払いプレイメージ)](#Payment-preimage)* のハッシュである支払いハッシュが含まれています。
   * _この用語が説明に含まれる用語(内容): [HTLC](#HTLC-Hashed-Time-Locked-Contract)_
   * _原案の用語: [Payment Preimage(支払いプレイメージ)](#Payment-preimage)_

* #### *Payment Preimage*:
   * 支払いプレイメージ。支払いが受信されたことを証明するもので、この秘密を知っている唯一の人である最後の受信者によって保持されます。
     最後の受信者は、資金を解放するためにプレイメージを解放します。
     この支払いプレイメージがハッシュされたものを *[Payment Hash(支払いハッシュ)](#Payment-hash)* といい、*[HTLC](#HTLC-Hashed-Time-Locked-Contract)* に含まれます。
   * _この用語が説明に含まれる用語(内容): [HTLC](#HTLC-Hashed-Time-Locked-Contract)_
   * _この用語が由来の用語: [Payment Hash(支払いハッシュ)](#Payment-hash)_

* #### *Peer*:
   * ピア。相互に通信している2つの[Node](#node)のことを指します。
      * Channelを設定する前に、2つのピアが互いに伝言(Gossip Message)のやり取りをする場合があります。
      * 2つのピアは、取引を行うための *[Channel](#channel)* を確立できます。
  * _関連用語: [Node](#node)_

* #### *Penalty Transaction*:
   * 罰則取引。*Commitment Revocation Private Key(言質撤回秘密鍵)* を使用して、
     *[Revoked Commitment Transaction(無効化済み言質取引)](#revoked-commitment-transaction)* のすべての出力を使用する取引です。
     *[Peer](#peers)* は、他のPeerが *[Revoked Commitment Transaction(無効化済み言質取引)](#revoked-commitment-transaction)*
     をブロードキャストすることによって「チート」しようとした場合にこれを使用します。
   * _関連用語: [Closing Transaction(終了時取引)](#closing-transaction)、[Commitment Transaction(言質取引)](#commitment-transaction)、[Funding Transaction(供給取引)](#funding-transaction)_


* #### *Per-commitment Secret*:
   * 言質ごとのシークレット。全ての *[Commitment Transaction(言質取引)](#commitment-transaction)* は言質ごとのシークレットからの鍵を持ちます。
     これは、以前のすべてのコミットメントの一連の言質ごとのシークレットをコンパクトに保存できるように生成されます。
   * _この用語が説明に含まれる用語(内容): [Commitment Transaction(言質取引)](#commitment-transaction)_
   * _この用語が由来の用語: [Commitment Revocation Private Key(言質撤回秘密鍵)](#commitment-revocation-private-key)_

* #### *Processing Node*:
   * 処理ノード。*[Origin Node(起点ノード)](#origin-node)* が *[Final Node(終点ノード)](#final-node)* に向けて送った支払いパケットを処理する *[Node](#node)* のことを指します。
     これは、まずメッセージを受信する *[Receiving Peer(受信ピア)](#receiving-peer)* として機能し、その後パケットを送信する *[Sending Peer(送信ピア)](#sending-peer)* として機能します。
  * _関連カテゴリ: [Node](#node)_
  * _関連用語: [Final Node(終点ノード)](#final-node)、[Origin Node(起点ノード)](#origin-node)_

* #### *Receiving Node*:
   * 受信ノード。メッセージを受信する *[Node](#node)* です。
   * _関連カテゴリ: [Node](#node)_
   * _関連用語: [Sending Node(送信ノード)](#sending-node)_

* #### *Receiving Peer*:
   * 受信ピア。直接接続された *Peer* からメッセージを受信している *[Node](#node)* です。
   * _関連カテゴリ: [Peer](#peer)_
   * _関連用語: [Sending Peer(送信ピア](#sending-peer)_

* #### *Revoked Commitment Transaction*:
   * 無効化済み言質取引。新しい *[Commitment Transaction(言質取引)](#commitment-transaction)* が成立したことによって無効化された古い言質取引のことを指します。
   * _関連カテゴリ: [Commitment Transaction(言質取引)](#commitment-transaction)_

* #### *Revoked Transaction Close*:
   * 無効化済み取引の終了。無効な *[Channel](#channel)* を終了することを指します。
     これは、*Revoked Commitment Transaction(無効化済み言質取引)* をブロードキャストすることで達成されます。
     他の *Peer* は *Commitment Revocation Private Key(言質撤回秘密鍵)* を知っているため、[Penalty Transaction(罰則取引)](#penalty-transaction)を作成できます。
   * _関連用語: [Mutual Close(相互終了)](#mutual-close)、[Unilateral Close(一方的な終了)](#unilateral-close)_

* #### *Route*:
  * ルート。1つ以上の *[Hop](#hop)* を介して *Origin Node(起点ノード)* から *[Final Node(終点ノード)](#final-node)* への支払いを可能にするLightning Network上の道のことを指します。
  * _関連用語: [Channel](#channel)_

* #### *Sending Node*:
   * 送信ノード。メッセージを送信する *[Node](#node)* です。
   * _関連カテゴリ: [Node](#node)_
   * _関連用語: [Receiving Node(受信ノード)](#receiving-node)_

* #### *Sending Peer*:
   * 送信ピア。直接接続された *Peer* へメッセージを送信している *[Node](#node)* です。
   * _関連カテゴリ: [Peer](#Peers)_
   * _関連用語: [Receiving Peer(受信ピア)](#receiving-peer)_.

* #### *Unilateral Close*:
   * 一方的な終了。非協力的に *[Channel](#channel)* を終了することを指します。
     これは *[Commitment Transaction(言質取引)](#commitment-transaction)* をブロードキャストすることで達成されます。
     この取引は *[Closing Transaction(終了時取引)](#closing-transaction)* よりサイズが大きい(要するに効率が悪い)です。
     更に、言質をブロードキャストしたピアは以前に交渉した期間中、自身の資金を操作することができません。
   * _関連用語: [Mutual Close(相互終了)](#mutual-close)、[Revoked Transaction Close(取り消された取引での終了)](#revoked-transaction-close)_

## Theme Song: テーマソング(※未翻訳です)

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

## Authors: 著者

[ FIXME: Insert Author List ]


## Translator: 翻訳者

[@y-chan](https://github.com/y-chan)

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
この仕様書は[Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/)でライセンスされています。
