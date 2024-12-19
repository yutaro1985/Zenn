---
title: "CloudFrontの新しいログ形式に対応したAthenaのテーブルを作ってみた"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","cloudfront","athena"]
published: true
---

:::message
この記事は
[みすてむず いず みすきーしすてむず その2 Advent Calendar 2024](https://adventar.org/calendars/10269)[^1]  
及び
[AWS（Amazon Web Services） Advent Calendar 2024](https://qiita.com/advent-calendar/2024/aws)
の18日目です。
:::

2024年のre:Inventで発表された…わけではないのですが、11月のアップデート[^2]でCloudFrontのログ出力の設定が新しくなりました。

設定方法についてはもちろんすでにClassmethodさんのブログに書かれています
というかそれで知りました
[CloudFront 標準ログがアップデート！新機能のパーティションやJSON出力を試してみた](https://dev.classmethod.jp/articles/cloudfront-access-log-update-202411/)

これにより、さまざまな出力形式が設定できるようになりました。
今まではCloudFrontのログはS3の指定パス上に、パスも切られずにズラッと出力するのみだったのでだいぶ扱いやすくなったのではないでしょうか。
CloudWatch Logs、Kinesis Firehoseにも送れるようになったので、取り扱いやすくもなりました。

本記事の執筆時点ではまだAWSの日本語ドキュメントには記載がないため、いったん英語のドキュメントをご参照ください。
[Configure standard logging (v2)](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/standard-logging.html)

実際にこのアップデートを元にS3でさっそくjson形式でログを出力してみました。
jsonで出力できるのでAthenaでクエリを送るのも簡単になっただろうと思ったのですが、思わぬところに落とし穴があったので今回はそのお話をします。

## CloudFrontのログ出力設定

設定自体は同じになってしまうので、詳細は[先のブログ](https://dev.classmethod.jp/articles/cloudfront-access-log-update-202411/)をご参照ください。
Amazon S3(レガシー)は旧来のログ出力設定で、もともとCloudFrontのログ出力設定をしてた場合はこれが設定されています。
こちらを新規で選択する理由はないので、S3に直接出力する場合は新形式を選びましょう。

今回の設定内容の主なポイントとしては以下です。（必ずしも推奨ではないです）

- 出力可能な項目をすべて出力
  - 出力設定の「Field Selection」のところですが、デフォルトだと選択されていない項目があります
    - timestamp(ms)とかは常に入れてもよいかもしれないです
- ログの出力パスは1時間ごとに設定
  - 今回は`/{DistributionId}/{yyyy}/{MM}/{dd}/{HH}`とします
    - 今のところは変数として使えるものをすべて使うとこんな感じになりますが、1時間単位でなく日単位で設定するなどももちろん可能です
    - 時間はUTCで切られることに注意してください
- Hive互換の形式にはしない
- Output formatはJSONに設定
  - W3C、Parquet、Plain Textが選択できます
    - ParquetにするとVended Logsの料金が追加でかかると思われるので注意

ちなみに従来はログ設定をした後に出力先のS3バケットにバケットポリシーを別途追加する必要があるのがそこそこなハマりポイントでしたが、この新設定ではバケットポリシーは自動で設定されます。

## Athenaでのテーブル作成

### 旧形式のログ用のテーブル作成クエリから新形式のクエリに変更する

新形式になり、ログの出力項目がいくつか増えましたがそれ以外は基本的に同じ項目です。

既存のCloudFrontのログに対してクエリを送るためのテーブルを作成するクエリは以下ドキュメントに記載があります。
[Create a table for CloudFront standard logs](https://docs.aws.amazon.com/athena/latest/ug/create-cloudfront-table-standard-logs.html)
※データベースは作成済みとします。

```sql:旧来のCloudFrontのログ用のAthenaのテーブル作成クエリ
CREATE EXTERNAL TABLE IF NOT EXISTS cloudfront_standard_logs (
  `date` DATE,
  time STRING,
  x_edge_location STRING,
  sc_bytes BIGINT,
  c_ip STRING,
  cs_method STRING,
  cs_host STRING,
  cs_uri_stem STRING,
  sc_status INT,
  cs_referrer STRING,
  cs_user_agent STRING,
  cs_uri_query STRING,
  cs_cookie STRING,
  x_edge_result_type STRING,
  x_edge_request_id STRING,
  x_host_header STRING,
  cs_protocol STRING,
  cs_bytes BIGINT,
  time_taken FLOAT,
  x_forwarded_for STRING,
  ssl_protocol STRING,
  ssl_cipher STRING,
  x_edge_response_result_type STRING,
  cs_protocol_version STRING,
  fle_status STRING,
  fle_encrypted_fields INT,
  c_port INT,
  time_to_first_byte FLOAT,
  x_edge_detailed_result_type STRING,
  sc_content_type STRING,
  sc_content_len BIGINT,
  sc_range_start BIGINT,
  sc_range_end BIGINT
)
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY '\t'
LOCATION 's3://amzn-s3-demo-bucket/'
TBLPROPERTIES ( 'skip.header.line.count'='2' )
```

ステータスコードをINTにするかSTRINGにするかはどっちがいいんでしょうね…。

新形式のログ設定ですべての項目を出力した場合、以下の項目が増えます。

- `timestamp`
- `DistributionId`
- `timestamp(ms)`
- `origin-fbl`
- `origin-lbl`
- `asn`

origin-fbl、origin-lblはオリジンからCloudFrontへの最初のバイトと最後のバイトのレイテンシで、小数が入ります。

新形式にない項目を追加し、形式がjsonになったことを踏まえてクエリを編集すると以下のようになります。
※Partition Projectionの設定はいったん省略します。

```sql:新形式のCloudFrontのログ用のAthenaのテーブル作成クエリ（
CREATE EXTERNAL TABLE IF NOT EXISTS cloudfront_standard_logs (
  timestamp BIGINT,
  DistributionId STRING,
  `date` DATE,
  time STRING,
  x_edge_location STRING,
  sc_bytes BIGINT,
  c_ip STRING,
  cs_method STRING,
  cs_host STRING,
  cs_uri_stem STRING,
  sc_status INT,
  cs_referrer STRING,
  cs_user_agent STRING,
  cs_uri_query STRING,
  cs_cookie STRING,
  x_edge_result_type STRING,
  x_edge_request_id STRING,
  x_host_header STRING,
  cs_protocol STRING,
  cs_bytes BIGINT,
  time_taken FLOAT,
  x_forwarded_for STRING,
  ssl_protocol STRING,
  ssl_cipher STRING,
  x_edge_response_result_type STRING,
  cs_protocol_version STRING,
  fle_status STRING,
  fle_encrypted_fields INT,
  c_port INT,
  time_to_first_byte FLOAT,
  x_edge_detailed_result_type STRING,
  sc_content_type STRING,
  sc_content_len BIGINT,
  sc_range_start BIGINT,
  sc_range_end BIGINT,
  `timestamp(ms)` BIGINT,
  origin_fbl FLOAT,
  origin_lbl FLOAT,
  asn INT
)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
LOCATION 's3://amzn-s3-demo-bucket/'
TBLPROPERTIES (
    'classification' = 'json',
);
```

が、**これはテーブルを作成できたとしても[^3]うまくクエリが実行できません。**

### うまくいかない理由

実際に最初の設定で出力されたjsonのログを見てみます。
※一部値を編集しています

```json:全ての項目を出力した場合の実際のログ形式
{
  "timestamp": "1734528645",
  "DistributionId": "EDFDVBD6EXAMPLE",
  "date": "2024-12-18",
  "time": "13:30:45",
  "x-edge-location": "SFO53-P7",
  "sc-bytes": "1352",
  "c-ip": "10.111.111.111",
  "cs-method": "GET",
  "cs(Host)": "exapmlehogehoge.cloudfront.net",
  "cs-uri-stem": "/index.html",
  "sc-status": "200",
  "cs(Referer)": "-",
  "cs(User-Agent)": "Mozilla/5.0%20(Macintosh;%20Intel%20Mac%20OS%20X%2010_15_7)%20AppleWebKit/537.36%20(KHTML,%20like%20Gecko)%20Chrome/124.61.652.785%20Safari/537.36",
  "cs-uri-query": "-",
  "cs(Cookie)": "-",
  "x-edge-result-type": "Miss",
  "x-edge-request-id": "xztq278Hco1x_LBixzRdOl2i0z3sYQOlJxFDKUSkv-R96pa7XurWOA==",
  "x-host-header": "exapmlehogehoge.cloudfront.net",
  "cs-protocol": "https",
  "cs-bytes": "493",
  "time-taken": "0.365",
  "x-forwarded-for": "-",
  "ssl-protocol": "TLSv1.3",
  "ssl-cipher": "TLS_AES_128_GCM_SHA256",
  "x-edge-response-result-type": "Miss",
  "cs-protocol-version": "HTTP/2.0",
  "fle-status": "-",
  "fle-encrypted-fields": "-",
  "c-port": "56089",
  "time-to-first-byte": "0.364",
  "x-edge-detailed-result-type": "Miss",
  "sc-content-type": "text/html",
  "sc-content-len": "-",
  "sc-range-start": "-",
  "sc-range-end": "-",
  "timestamp(ms)": "1734528645492",
  "origin-fbl": "0.363",
  "origin-lbl": "0.364",
  "asn": "2516"
}
```

1. フィールド名を見ると、括弧が入っていたり、キャメルケースがとケバブケースが混在しているなど、割とめちゃくちゃです。
テーブル作成する時にテーブルのカラム名とログのフィールドの名前が一致しない場合はマッピングの設定をいれる必要があり、このままではうまく動きません。
よって、**テーブルのカラム名をログのフィールド名に合わせる**または**テーブルのカラム名とフィールド名が異なる部分にマッピングの設定を入れる**必要があります。
1. ドキュメント上では言及されていませんが、**存在する場合は数値が入るのに存在しない時は文字列の「-」が入るやっかいな項目があります**。
自分が確認した限り、具体的には以下です。
   - `fle-encrypted-fields`
   - `sc-content-len`
   - `sc-range-start`
   - `sc-range-end`
   - `origin-fbl`
   - `origin-lbl`
これらの項目をINT、BIGINT、FLOATにすると、**数値が入らず文字列の「-」になっている時にクエリがエラーになります**。
それを防ぐため、**これらの項目はSTRINGで定義してください**。
※もしかしたらこれ以外にもあるかもしれないので、仮にそのような項目があったらコメントで教えてください。

### テーブル作成クエリの修正

ここまでのことを踏まえて、以下のように修正します。

- フィールド名とカラム名をマッピングする
  - jsonのフィールド名は色々混在している上、ハイフンが入っていてあんまりいい感じではないので、カラム名をスネークケースで統一します。
- 数値と文字列の「-」が混在する項目をSTRINGで定義する

また、先のクエリではPartition Projectionの設定を省略しましたが、ここでついでにPartiotion Projectionの設定を追加します[^4]。
今回はディストリビューションIDはパーティションに含めていませんが、含める場合はその設定をPartiotion Projectionの設定に追加してください。
個人的にはディストリビューション事にテーブルを作ったほうがよいと思いますが、ディストリビューションがたくさんある時はもしかしたら話が変わるかもしれません。

```sql:修正後のCloudFrontのログ用のAthenaのテーブル作成クエリ
CREATE EXTERNAL TABLE cloudfront_standard_logs (
  timestamp BIGINT,
  distribution_id STRING,
  date DATE,
  time STRING,
  x_edge_location STRING,
  sc_bytes BIGINT,
  c_ip STRING,
  cs_method STRING,
  cs_host STRING,
  cs_uri_stem STRING,
  sc_status INT,
  cs_referer STRING,
  cs_user_agent STRING,
  cs_uri_query STRING,
  cs_cookie STRING,
  x_edge_result_type STRING,
  x_edge_request_id STRING,
  x_host_header STRING,
  cs_protocol STRING,
  cs_bytes BIGINT,
  time_taken FLOAT,
  x_forwarded_for STRING,
  ssl_protocol STRING,
  ssl_cipher STRING,
  x_edge_response_result_type STRING,
  cs_protocol_version STRING,
  fle_status STRING,
  fle_encrypted_fields STRING,
  c_port INT,
  time_to_first_byte FLOAT,
  x_edge_detailed_result_type STRING,
  sc_content_type STRING,
  sc_content_len STRING,
  sc_range_start STRING,
  sc_range_end STRING,
  timestamp_ms BIGINT,
  origin_fbl STRING,
  origin_lbl STRING,
  asn INT
)
PARTITIONED BY (partition_date STRING)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
WITH SERDEPROPERTIES (
  'case.insensitive' = 'true',
  'mapping.distribution_id' = 'DistributionId',
  'mapping.x_edge_location' = 'x-edge-location',
  'mapping.sc_bytes' = 'sc-bytes',
  'mapping.c_ip' = 'c-ip',
  'mapping.cs_method' = 'cs-method',
  'mapping.cs_host' = 'cs(Host)',
  'mapping.cs_uri_stem' = 'cs-uri-stem',
  'mapping.sc_status' = 'sc-status',
  'mapping.cs_referer' = 'cs(Referer)',
  'mapping.cs_user_agent' = 'cs(User-Agent)',
  'mapping.cs_uri_query' = 'cs-uri-query',
  'mapping.cs_cookie' = 'cs(Cookie)',
  'mapping.x_edge_result_type' = 'x-edge-result-type',
  'mapping.x_edge_request_id' = 'x-edge-request-id',
  'mapping.x_host_header' = 'x-host-header',
  'mapping.cs_protocol' = 'cs-protocol',
  'mapping.cs_bytes' = 'cs-bytes',
  'mapping.time_taken' = 'time-taken',
  'mapping.x_forwarded_for' = 'x-forwarded-for',
  'mapping.ssl_protocol' = 'ssl-protocol',
  'mapping.ssl_cipher' = 'ssl-cipher',
  'mapping.x_edge_response_result_type' = 'x-edge-response-result-type',
  'mapping.cs_protocol_version' = 'cs-protocol-version',
  'mapping.fle_status' = 'fle-status',
  'mapping.fle_encrypted_fields' = 'fle-encrypted-fields',
  'mapping.c_port' = 'c-port',
  'mapping.time_to_first_byte' = 'time-to-first-byte',
  'mapping.x_edge_detailed_result_type' = 'x-edge-detailed-result-type',
  'mapping.sc_content_type' = 'sc-content-type',
  'mapping.sc_content_len' = 'sc-content-len',
  'mapping.sc_range_start' = 'sc-range-start',
  'mapping.sc_range_end' = 'sc-range-end',
  'mapping.timestamp_ms' = 'timestamp(ms)',
  'mapping.origin_fbl' = 'origin-fbl',
  'mapping.origin_lbl' = 'origin-lbl'
)
LOCATION 's3://amzn-s3-demo-bucket/EDFDVBD6EXAMPLE/'
TBLPROPERTIES (
    'classification' = 'json',
    'projection.enabled' = 'true',
    'projection.partition_date.type' = 'date',
    'projection.partition_date.range' = '2024/12/18/00,NOW',
    'projection.partition_date.format' = 'yyyy/MM/dd/HH',
    'projection.partition_date.interval' = '1',
    'projection.partition_date.interval.unit' = 'HOURS',
    'storage.location.template' = 's3://amzn-s3-demo-bucket/EDFDVBD6EXAMPLE/${partition_date}'
);
```

このようにテーブルを作成することで、新形式のCloudFrontのログに対してAthenaでクエリを問題なく実行できるようになります。

#### マッピングの設定について

`WITH SERDEPROPERTIES`の中に`'mapping.テーブルのカラム名' = 'jsonのフィールド名'`という形でマッピングの設定を入れます。
たとえば`'mapping.distribution_id' = 'DistributionId'`という設定は、jsonのフィールド名`DistributionId`をテーブルのカラム名`distribution_id`にマッピングする設定です。
`'case.insensitive' = 'true'`はフィールド名が大文字小文字を区別しないようにする設定です。

#### Partition Projectionの設定について

参考資料もあるのでざっくりと説明します。
まず、何でパーティションを切るかを定義します。
`PARTITIONED BY (partition_date STRING)`の部分で、パーティションに使う項目を定義します。
ここではpartition_dateとしましたが、名前はカラム名と被らなければ基本的に任意です。
予約語とも被らないほうがよいでしょう。
他にパーティションを増やしたい時はカンマで区切って追加します。

Partiotion Projectionの設定は`TBLPROPERTIES`に追加します。
`'projection.enabled' = 'true'`でPartiotion Projectionを有効にします。
`'projection.partition_date.type' = 'date'`でパーティションの型を指定します。
`'projection.partition_date.range' = '2024/12/18/00,NOW'`でパーティションの範囲を指定します。
`'projection.partition_date.format' = 'yyyy/MM/dd/HH'`でS3のパス上の文字列をどのように読み取るかを指定します。
`'projection.partition_date.interval' = '1'`でパーティションの間隔を指定します。
`'projection.partition_date.interval.unit' = 'HOURS'`でパーティションの単位を指定します。
`'storage.location.template' = 's3://amzn-s3-demo-bucket/EDFDVBD6EXAMPLE/${partition_date}'`で、S3上のパスのどの部分をパーティションの文字列として認識するかを指定します。

Partiotion Projection（パーティション射影）については以下を参照してください。
[Amazon Athena でパーティション射影を使用する](https://docs.aws.amazon.com/ja_jp/athena/latest/ug/partition-projection.html)
[Amazon Athena のPartition Projection(パーティション射影)の設定方法の確認と、手動でパーティションを追加する場合との比較](https://dev.classmethod.jp/articles/tried-to-check-how-to-set-amazon-athena-partition-projection/)

## その他の考えられる方法

試してないので「多分できるだろうなあ」ぐらいの感覚ですが、ざっくり思いつくのは以下です。

- Firehoseに送り、Lambdaを使って適切に項目を変換する
  - CloudWatch Logsに送った後にCloudWatch LogsからFirehoseに送るでも多分同じような感じかなと思います
- S3にログ出力後、Glueやその他ETLツールを使ってログの整形する

また、JSONでない形式で出力する場合については検証していないので、その場合どうなるかは…どなたか検証してください（丸投げ）

## まとめ

CloudFrontのログの仕様はこれまでは正直不便なものでした。
アップデートによりログ出力の設定が柔軟になり、だいぶ扱いやすくなりました。
ただ、Athenaで検索するにあたってはまだ情報が整備されていないので、この生地がその手助けになれば幸いです。

[^1]: [みすてむず いず みすきーしすてむず](https://misskey.systems/) とは、オープンソースのプラットフォームMisskeyのインスタンスのひとつで、主にITに関わる人が参加しています。最近はXよりもそちらに入り浸っています。
[^2]: いわゆる「予選落ち」とか「pre:invent」とか言われているやつです。
[^3]: どうせうまく行かないということでテーブルを作成できるようにするまでデバッグするのも諦めてしまったのでやや適当です。
[^4]: パーティションを切らないとクエリが常に全量検索になります。WHERE句でパーティションを指定することにより、検索範囲に制限をかけて読み込むデータ量を減らし、料金の節約とクエリの実行時間を短縮できます。
