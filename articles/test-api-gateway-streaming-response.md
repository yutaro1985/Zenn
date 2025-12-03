---
title: "API Gatewayのresponse streamingでAPI Gatewayのサイズ上限を超えてダウンロードが行えるか試してみた"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","apigateway"]
published: true
published_at: 2025-12-04 07:00
---

:::message
この記事は
[みすてむず いず みすきーしすてむず Advent Calendar 2025](https://adventar.org/calendars/11504)[^1]
および
[AWS（Amazon Web Services） Advent Calendar 2025](https://qiita.com/advent-calendar/2025/aws)
の4日目です。
:::

## はじめに

みなさん、AWS re:Inventの時期ですね！  
AWSをお使いの皆様はこの時期はAWSのアップデート情報を追うのに忙しいかなと思います。  
もしくはre:Inventの現地で発表とセッションを追うのに忙しい方もいらっしゃるかもしれません。  
自分は現地参加は無理なので公式ブログと現地参加の皆さんの情報を頼りにアップデート情報を追っています。

さて、re:Inventの時期にはAWSのアップデートが大量にリリースされるのが恒例 [^2]ですが、今回は11月にアップデートのあったAPI GatewayのResponse Streaming
[に注目します。B@ucardhttps://aws.amazon.com/jp/blogs/compute/building-responsive-apis-with-amazon-api-gateway-response-streaming/)

これによりレスポンスの制限を一部乗り越えられるのではないかと思ったのが動機です。
既に触れられているユースケースでよくあるのは、[Bedrock](https://docs.aws.amazon.com/ja_jp/bedrock/latest/userguide/what-is-bedrock.html)[^3]のレスポンスをLambdaがそのままストリームでAPI Gatewayから返す、というものです。
例えば以下など
@[card](https://dev.classmethod.jp/articles/api-gateway-response-streaming-bedrock/)

この記事ではそれらではなく、Lambdaのresponse streamingと組み合わせて10MBを超えるファイルをAPI Gatewayから返せるか、ということを検証してみます。

## 前提

最初に軽く触れましたが、API Gatewayのレスポンスには制限が存在しました。
ペイロードサイズが10MB(**上限緩和不可**)だったり、タイムアウトが最大29秒（HTTP APIだと30秒っぽい）だったり…。
※参考
@[card](https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/api-gateway-execution-service-limits-table.html)

- ランタイム: Lambda（PythonまたはNode.js）
- API種別: API Gateway REST API を想定（Response Streaming 対応は執筆時点ではREST APIのみ）
- 目的: 「大きなコンテンツをストリーミングでクライアントへ返せるか」を実験し、実務適用時の注意点を整理する

注意点として、Lambdaのレスポンスストリーミングはランタイムによって対応状況が異なります。Node.jsはランタイムネイティブでの対応が案内されています。一方、他言語では AWS Lambda Web Adapter（`awslabs/aws-lambda-web-adapter`）を組み合わせることでHTTPフレームワークのストリーミングをLambdaに橋渡しできます。
本記事ではPython で Web Adapterを使用した手法も触れます。(自分がPythonでLambdaを実装する要件があり検証したいため。)
@[card](https://github.com/awslabs/aws-lambda-web-adapter)

## 検証構成（概略）

- クライアント（ブラウザや `curl`）
- API Gateway（REST API）
  - **レスポンス転送モードを"ストリーム"に設定する**
    - 今回のアップデートでできるようになったことです
  - Lambda統合でLambdaにレスポンスを返させる
- Lambda（ストリーミングで応答を返す）
- S3バケット
  - 今回はS3のAPI経由ではなく、バケットを公開してそこからダウンロードする形で試します。

## セットアップ手順（概要）

1. API Gateway REST API を作成
2. Response Streamingを実装したLambdaを作成
3. 統合に Lambda を指定（Lambdaプロキシ統合）
4. 必要に応じてステージ/ルート/オーソリゼーションを設定
5. `curl`やブラウザからダウンロード挙動を確認（CloudWatch Logs でエラーやタイムアウトも併せて確認）

ローカルでの簡易動作確認やパッケージングには、普段のビルド／デプロイ手段（SAM/Serverless Framework/Terraform/手動アップロードなど）を適用してください。ここでは最小例に留めます。

### Lambdaの実装

API Gatewayにレスポンスを返させるLambdaを先に作成します。
先述の通り、執筆時点ではNodeだけがLambdaのResponse Streamingのみに対応しています。

#### Node.js（ネイティブ）— S3中継の最小例

Node.js ランタイムでは Lambda がResponse Streamingをネイティブにサポートします。公開S3のオブジェクトを`https.get`で取得し、そのままストリーム転送します。

```js
import https from 'node:https';
import { pipeline } from 'node:stream/promises';

// グローバルの awslambda ヘルパを使用
export const handler = awslambda.streamifyResponse(async (event, responseStream, context) => {
  const httpResp = awslambda.HttpResponseStream.from(responseStream, {
    statusCode: 200,
    headers: {
      'Content-Type': 'application/octet-stream',
      // 任意: ダウンロード時のファイル名を指定
      'Content-Disposition': 'attachment; filename="file.bin"'
    }
  });

  const url = 'https://<bucket>.s3.<region>.amazonaws.com/path/to/object';

  await new Promise((resolve, reject) => {
    https.get(url, (res) => {
      if (res.statusCode !== 200) {
        // 上流エラーを簡易的に通知
        httpResp.write(Buffer.from(`Upstream error: ${res.statusCode}\n`));
        httpResp.end();
        reject(new Error(`S3 GET failed: ${res.statusCode}`));
        return;
      }
      pipeline(res, httpResp).then(resolve).catch(reject);
    }).on('error', reject);
  });
});
```

REST API は Lambda プロキシ統合で、該当リソース/メソッドに上記ハンドラを紐づけます。

#### Python（Web Adapter）— S3中継の最小例

Pythonでは AWS Lambda Web Adapter を使ってFlask/FastAPI等のストリーミングを有効化し、公開S3から取得したバイナリをそのまま転送します。

コード例（`app.py`）:

```python
import requests
from flask import Flask, Response

app = Flask(__name__)

S3_URL = "https://<bucket>.s3.<region>.amazonaws.com/path/to/object"
CHUNK_SIZE = 1024 * 1024

def stream_s3():
  with requests.get(S3_URL, stream=True) as r:
    r.raise_for_status()
    for chunk in r.iter_content(chunk_size=CHUNK_SIZE):
      if chunk:
        yield chunk

@app.route("/download")
def download():
  return Response(stream_s3(), mimetype="application/octet-stream")
```

Web Adapter の利用（Layerをアタッチして実行ラッパを有効化）が必要です。公式リポジトリの手順に沿って設定してください。

##### Web Adapter 導入ポイント（Python）

- Layer: 公式のLambda Adapter Layerを関数にアタッチ（リージョン/アーキ別の最新ARNはREADME参照）
- 環境変数（必須）:
  - `AWS_LAMBDA_EXEC_WRAPPER=/opt/bootstrap`
  - `AWS_LWA_INVOKE_MODE=response_stream`
  - `PORT=8080`（または`AWS_LWA_PORT`）
- 環境変数（任意）:
  - `AWS_LWA_ENABLE_COMPRESSION=true`（圧縮を有効化）
  - `RUST_LOG=info`（デバッグ用）
- フレームワーク: Flaskのジェネレータ/ FastAPIの`StreamingResponse`でストリーム返却
- テスト: `curl -v`で`Transfer-Encoding: chunked`や受信進行を確認、CloudWatch Logsで例外/タイムアウトを監視

デプロイ方法（bash、zipパッケージ想定）:

```bash
# 依存取得（Flask）
python3 -m venv .venv
source .venv/bin/activate
pip install flask requests

# パッケージング
mkdir -p dist
cp app.py dist/
# Web Adapter Layerをアタッチする前提。必要ならバイナリ/設定も dist に配置（公式手順に従う）
zip -r dist.zip dist

# Lambda へアップロード（AWS CLI）
aws lambda update-function-code --function-name <your-func> --zip-file fileb://dist.zip
```

API Gateway（REST API）のリソース/メソッドを `/download` に向ければ、チャンクでダウンロードされるはずです。クライアント側の確認例:

```bash
curl -L -o big.bin "https://<api-id>.execute-api.<region>.amazonaws.com/download"
wc -c big.bin
```

#### 期待できる効果と観点

- 大きいコンテンツの送出: 従来の固定上限（単一レスポンスボディサイズ）を避けつつ配信可能
- メモリ消費の平準化: 生成・読み取りしたチャンクを逐次送出
- ユーザー体験の改善: 早期にダウンロードが開始され、全量完了を待たずに受信が進む

#### 制約・注意点（実務での落とし穴）

- ランタイム対応差: Node.jsはネイティブ、Python等はWeb Adapterの導入が必要
- タイムアウト: Lambda実行時間上限内で送出完了が必要（長時間配信は注意）
- ネットワーク途中経路: 一部のプロキシ/CDN/クライアントがチャンク転送に制約を持つことあり
- VPC接続: 関数をVPCに接続するとデフォルトでインターネットへ出られないため、公開S3へ到達するにはNAT構成、またはS3向けVPCエンドポイントの利用が必要
- ヘッダ/圧縮: 事前に `Content-Length` を確定しづらい。圧縮や`Content-Encoding`の扱いは検証必須
- キャッシュ戦略: ストリーミングとキャッシュ制御の組み合わせは要検討（CloudFront経由の場合など）
- エラーハンドリング: 途中失敗時の再開や整合性（部分取得）をどう扱うか

### REST API作成手順

検証用なので簡素な設定でコンソールから作ります。

1. APIを作成: API Gateway コンソールで「REST API」を新規作成
   - ![REST API作成時の設定例](/images/test-api-gateway-streaming-response/create_restapi.png)
2. リソース追加: 例として`/download`リソースを作成
   - ![alt text](/images/test-api-gateway-streaming-response/add_path_to_api_gateway.png) 
3. メソッド追加: `GET`メソッドを作成し、統合タイプを`Lambda プロキシ`に設定、対象のLambda関数を指定
4. 統合リクエスト: 「Response transfer mode」を`Stream`に設定（コンソールのIntegration設定画面）
5. 統合レスポンス: 必要に応じてヘッダ（`Content-Type`等）をパススルー、チャンク転送を阻害する設定がないことを確認
6. デプロイ: 新規または既存ステージへデプロイして、Invoke URLを取得
7. 動作確認: `curl -v`などでヘッダと受信状況を確認（後述の確認例）

## 検証結果（暫定）

本記事の最小検証では、チャンク出力により従来より大きいデータをクライアントへ転送できることを確認しました。実際の上限値や挙動は、API Gatewayの最新仕様・リージョン・ステージ設定・ヘッダ・ランタイムのバージョンに依存する可能性があるため、プロダクション適用前に環境固有の再検証を推奨します。

## まとめ

- API GatewayのStreaming Responseを使うことで、大容量ダウンロードの選択肢が広がる
- Node.jsは素直に対応。PythonではWeb Adapterの利用が現実的
- 運用にあたってはタイムアウト・中間経路・キャッシュ・再取得戦略などを事前に検証

## 参考

- AWS Lambda Web Adapter: <https://github.com/awslabs/aws-lambda-web-adapter>
- Lambda Response Streaming の解説（英語、公式ブログやドキュメントを参照）

## 余談

この記事はCopilot Chatの助けを存分に得ながら作成しました。
他にもいくつかZennに記事を書いてリポジトリ管理しているので、口調や考え方などは他記事を参考にしてもらっています。
もちろん丸投げじゃなくて、ある程度内容を整備してもらって自分で検証した結果を記載しています。
記事を書くのは単純に文章を作るのが大変なので、労力がかかる部分をうまく助けてもらいつつ、記事作成のハードルを下げて新しい記事を書きやすくなるといいなと思っています。

[^1]: [みすてむず いず みすきーしすてむず](https://misskey.systems/) とは、オープンソースのプラットフォームMisskeyのインスタンスのひとつで、主にITに関わる人が参加しています。最近はXよりもそちらに入り浸っています。
[^2]: いわゆる「予選落ち」とか「pre:invent」とか言われているやつです。
[^3]: いろんなAIモデルをAWSの統合APIを通じて利用できるようにしているサービス。AWSがAnthropicに出資していることもあり、Claudeの新バージョンの追加が早い。執筆時点ではGPTやGeminiはOSS版しかなさそう。
