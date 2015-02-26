---
layout: manual
title: genn.ai and Trident
---

## Meetup.comのAPIを用いた比較例


meetup.comとは、勉強会やミートアップの告知と参加者管理を行うウェブサービスで、全世界182カ国2000万人以上が利用している。
ここで公開されている、参加、不参加の登録に関するストリームデータをサンプルとして、
genn.aiとTridentで同様の処理を実装したときの比較を行う。

処理の内容は、
一度参加するとして登録したミートアップをキャンセルした人をリアルタイムに抽出
する処理とした。
同処理を実装するため、人毎かつイベント毎にYESで登録された情報を整理しておき、
同じ条件(人、イベンド)であとからNOが送られ来ることを検知する。

### 1.タプル化

現状、genn.aiではRESTでのデータ投入が必要なため、同API(WebSocket)への接続を行い、
受け取ったイベンド(参加/不参加)を都度POSTするプログラムを実装した。

Tridentでは、必ずタプルの発行を行うSPOUTを実装する必要があるため、
今回はその中で同APIへの接続とタプルの発行(後続へのemit)を行う実装とした。

    FROM meetuprsvp
    USING kafka_spout()
    EACH *,
      cast((mtime DIV 1000) AS TIMESTAMP) AS mtime_date,
      cast((event.time DIV 1000) AS TIMESTAMP) AS event_time_date
    INTO source;

なお、このときのgenn.aiでは、同RESTにてJSONとして各データ項目へのアクセスが可能となっており、
また、クエリ記述として同時に文字列として受け取った時刻情報の日付型への変換を行っている。

Tridentにおいてこれに当たる処理、各データのパースや型変換の処理は、Trident用の関数を記述することで可能となる。

    ＊＊＊

### 2.引き当て準備

genn.aiでは人毎かつイベンド毎のタプル待ち合わせを条件付きのストリームJOINとして実装できる。
このため、まずYESで登録されたタプルのストリームと、NOのストリームを作成する。

    FROM source(meetuprsvp) AS s_yes
    FILTER response == 'yes'
    EACH member.member_id AS member_id,
      member.member_name AS member_name,
      event.event_name AS event_name,
      event.event_id AS event_id
    INTO stream_yes;

    FROM source(meetuprsvp) AS s_no
    FILTER response == 'no'
    EACH member.member_id AS member_id,
      member.member_name AS member_name,
      event.event_name AS event_name,
      event.event_id AS event_id
    INTO stream_no;

TridentでのストリームJOINは、条件を使った待ち合わせ等できない
（単純にタイミングが合ったタプルを結合できるだけ）。

そのため、まずYESのものを全てストアに格納しておき、
これにNOのタプルが来たら都度クエリをかけてキャンセルがあったかどうかを判定する。
Tridentにはストリームの処理結果を格納する機能があるため、
ここにストアごとに作成されたストアエンジン(state)を設定することで比較的容易にデータの書き出しが可能である。
今回、ストアエンジンにはOSSとして公開されているMongo-Tridentを用いた。

    ＊＊＊

### 3.引き当て

先の通りgenn.aiではストリームJOINの条件として人とイベンドを設定し、マッチしたものを後続に流すクエリを書く。
この時、時刻条件の確認(何分前まで遡って調べるか)の指定も行う。
なお、genn.aiではMongoDBの利用は読み出し/書き出しともに標準で実装されているため、クエリ内記述のみで対応できる。

    FROM(
      stream_yes(s_yes) JOIN stream_no(s_no)
      ON s_yes.member_id = s_no.member_id
         AND s_yes.event_id = s_no.event_id
      TO s_yes.member_name AS member_name,
         s_yes.member_id AS member_id,
         s_yes.event_id AS event_id,
         s_yes.event_name AS event_name,
         s_yes.mtime_date AS yes_mtime,
         s_no.mtime_date AS no_mtime
      EXPIRE 3min ) AS joined_stream
    FILTER yes_mtime < no_mtime
    EMIT * USING mongo_persist('test', 'rsvp_tmp_join');

Tridentでは、この条件をMongoDBの検索クエリとして実現するが、
検索自体はTridentとしての機能がないため、ストアエンジンを直接操作して検索してくるTrident関数を、先のものと同様に実装する必要がある。

なお、時刻条件の確認はMongoDB側でもTrident側でも可能だが、処理の分散性を考えTrident側での実装とした。

    ＊＊＊

### 4.全体(Control/Confgtol flow)

genn.aiでは、CLIを用いてクエリを記述して順次ゆき、できたらそれをSUBMITすることで処理を動かすことができる。

Tridentでも、処理自体をプログラムとして記述する必要はあるものの、ほぼ同様の順序で処理を書いてゆくことが可能である。これを後から実行することで処理を動かすことができる。

### まとめ

genn.aiはリアルタイム処理を簡単に実現できるよう機能準備されたフレームワークであるため、
この処理ロジック自体は全て独自のクエリで記述することで実現できた。

Tridentでも、もちろん処理の実装は可能であったが、準備されている機能がプリミティブであるため、
幾つかの機能は関数として、低レベルな処理も含めてコード化する必要があった
（条件抽出の実装はストアを活用した実装など）。

これまでの対比を図解すると以下のとおり。

<iframe src="https://docs.google.com/presentation/d/1KhzALhhVRZK6oJ3RGKptICes8nNrf_WTm3dj_3YPsaY/embed?start=false&loop=false&delayms=3000" frameborder="0" width="480" height="299" allowfullscreen="true" mozallowfullscreen="true" webkitallowfullscreen="true"></iframe>

なお、Tridentは幾つかのタプルをまとめて処理する方式であるため、
クエリをかけるためのストアへの書き出し中にクエリ発行されてしまい、条件抽出に失敗するケースがあるものと思われる。
ただこれは、
今回の実装では対応しなかったがTridentはSPOUTの作り込み如何で処理失敗時のタプル再送が可能である。
逆に、genn.aiではこの再処理に対するサポートは実装されていない。

また、これも今回は利用していないが、genn.aiでは受け取ったタプルを使って、別の処理(例えば、逆に不参加としたが心変わりして参加とした人を抽出する処理など)を稼働中に追加、削除することが可能である。
これは、比較的高頻度で処理の内容を調整したい場合、もしくはリアルタイムデータに対しての処理をトライ＆エラーで作り込みしてゆく場合には極めて有効だと思われる。




