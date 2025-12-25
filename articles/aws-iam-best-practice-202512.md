---
title: "Re:AWSを使うにあたりIAMのベストプラクティスをもう一度確認する 202512版"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","iam","security"]
published: true
---
:::message

- この記事は[AWS（Amazon Web Services） Advent Calendar 2025](https://qiita.com/advent-calendar/2025/aws)の25日目です。
- この記事はAIの力を借りて作成しています。ただし、一部内容の作成、内容の確認や修正、壁打ちなどは筆者が主体となって実施しています。

:::

## はじめに

5年ほど前、以下の記事を書きました。
[AWSを使うにあたりIAMのベストプラクティスをもう一度確認する](daa2aaa6cc9907.md)

この記事は多くの方に読んでいただき、今でも参照されることが多い記事となっています。
しかし、AWSのサービスやベストプラクティスは常に進化しています。
この5年でIAM周りもアップデートされ、最新のベストプラクティスも変わってきています。
実際、前回記事で参照していた以下のページもアップデートが入っています。

@[card](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/best-practices.html)

当時からのアップデートと、2025年12月現在のベストプラクティスを改めて整理したいと思います。

## この記事の狙い

2020/12/21以降のIAM関連サービスや周辺機能の変化を整理し、現在のIAMベストプラクティスが「なぜそう言われるのか」の理解に繋げられればと思います。
まずは当時からの変化について見てみます。

## 5年前（2020年）から見て何が変わったか

前回記事の当時は**IAMユーザーとアクセスキーをどう扱うか**が中心でした。2025年12月時点では、運用の前提がいくつか大きく変わっています。

- 人間ユーザーの入口: IAMユーザー前提から、Identity Center＋フェデレーションが現実的な第一選択へ（`aws sso login`の普及でCLIも一時クレデンシャル運用がしやすい）。
  - IAMユーザーのアクセスキーは必要なければ発行しない・IAMユーザーを作成し、セキュリティ設定を適切に施して使う、というのがざっくりとした使い方でしたが、最近はそもそもIAMユーザーすら作らない方法が出てきています。
- 最小権限の作り方: 手作業中心から、Access Analyzerによるポリシー生成/検証/未使用分析で「削る運用」が回せるように。
  - 最小権限を作るのは昔からベストプラクティスとされていましたが、実際に行うのはかなり大変です。そのために有用なサービスが出てきています。
- MFAの実行性: WebAuthn/FIDO対応や複数MFA対応で「必須化」が運用として成立。
  - 当時どの程度まで使えたかはちょっと定かではないのですが、パスキー認証が使えるようになったのは大きなポイントです。
- 組織ガードレール: OrganizationsのRCP/SCPやControl Towerで、アカウント単体ではなく組織単位の統制が前提に。
  - 複数アカウント運用が前提になってきた現在、アカウントをまたいだ権限管理がやりやすくなる手段が提供されるようになりました。
- AWS外ワークロード: IAM Roles Anywhereで「AWS外でもロール＋一時クレデンシャル」を使えるようになった。
  - アクセスキーを使わない一時クレデンシャルの発行方法が増えました。これによってアクセスキーを発行するケースは更に減らせることが期待できます。
- 条件キー/評価ロジックの進化: 条件キー追加や評価ロジックの更新により、境界制御の設計が細かくできるように。
  - IAMポリシーの継続的なアップデートにより、より細かな条件判定ができる世になりました。

この変化を具体的に把握するため、まずはIAM関連サービスのアップデートを整理します。

## IAM関連サービスのアップデート

IAMそのものだけでなく、周辺サービスの変化が実運用に大きく効いてきています。特に影響が大きい領域は以下です。
まずはリストだけ記述し、記事後半でそれぞれがベストプラクティスにどう絡んでいるかについて記載します。

- IAM Identity Center（旧AWS SSO）: 人間ユーザーはフェデレーション前提へ。`aws sso login`などで長期アクセスキーを減らしやすくなりました。
  - https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html
  - https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html
- IAM Access Analyzer: 最小権限の作成や未使用アクセスの棚卸しを実務で回しやすくなりました。
  - https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html
- AWS Organizations（SCP/RCP）: 組織単位のガードレールを前提に設計できるようになりました。
  - https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html#access_policy-types
- AWS Control Tower: ガードレール運用を標準化しやすく、組織統制の実行性が上がりました。
  - https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html
- IAM Roles Anywhere: AWS外のワークロードにも「ロール＋一時クレデンシャル」の運用が広がりました。
  - https://docs.aws.amazon.com/rolesanywhere/latest/userguide/what-is-rolesanywhere.html

### 年表（2020/12/21以降の主な変化）

この辺は自分で調べるのは大変なのでGPT5.2に調査してまとめてもらいました。
過不足や誤りあればご指摘ください。
それにしても、こうやって並べてみると5年も経てば大きく変わるものですね。

- 2021-03-15 ポリシー検証（Policy validation）拡充: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_policy-validator.html
- 2021-04-07 Access Analyzerのポリシー生成: https://docs.aws.amazon.com/IAM/latest/UserGuide/access-analyzer-policy-generation.html
- 2021-04-13 AssumeRoleのアクション監視/制御: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_control-access_monitor.html
- 2021-10-05 IAMベストプラクティスの更新: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
- 2022-05-31 U2F非推奨→WebAuthn/FIDO: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa.html
- 2022-07-14 Identity CenterでPermissions boundary対応: https://docs.aws.amazon.com/singlesignon/latest/userguide/permissionsets.html
- 2022-07-26 AWS SSO→IAM Identity Center改名: https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html#renamed
- 2022-11-16 ルート/IAMユーザーの複数MFA対応: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa.html
- 2023-11-26 Access Analyzer未使用アクセス分析/カスタムチェック: https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html#what-is-access-analyzer-unused-access-analysis
- 2024-11-13 OrganizationsのRCP対応: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html#access_policy-types
- 2024-11-14 ルートアクセス集中管理: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user.html#id_root-user-access-management
- 2025-01-30 ポリシー評価ロジック更新: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html
- 2025-06-06/06-16 OIDC制御/内部アクセス分析: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_oidc_secure-by-default.html
- 2025-08-28 VPCエンドポイント条件キー追加: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html
- 2025-09-04 サービス固有資格情報API条件キー追加: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_iam-condition-keys.html#available-keys-for-iam

次に、現在のIAMベストプラクティス自体がどのように整理されているかを確認してみましょう。

## IAMベストプラクティスの主なアップデート

### 現在のベストプラクティス（公式ページの項目）

現在のベストプラクティスがどうなっているかを項目ごとに見ていきます。
構成上、一部項目は順序を入れ替えています。
引用元
@[card](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/best-practices.html)

#### 人間のユーザーが一時的な認証情報を使用して AWS にアクセスする場合に ID プロバイダーとのフェデレーションを使用することを必須とする

大きく変わった部分です。
特に組織レベルで(※)、IAM Identity CenterによりIAMユーザーを使わないでAWSアカウントへ、特定の権限を与えてアクセスさせるということがやりやすくなりました。
`aws sso login`で、cliやSDKをローカルで動かすときにもIAM Identity Centerの認証を使ってアクセスを提供することも可能です。
※こう書いていますが個人レベルでも容易に使用可能です。特に組織レベルの複雑なアカウント運用でのロール管理が楽になりました。
従来ではログイン用アカウントのIAMユーザーを作成して、AssumeRoleを使用する、ユーザーにはグループでアクセス許可を設定する、というのがベストプラクティスとされていました。
実際、過去の記事ではそのあたりに言及しています。
[個々の IAM ユーザーを作成する](./daa2aaa6cc9907.md#個々の-iam-ユーザーを作成する)
[IAM ユーザーへのアクセス許可を割り当てるためにグループを使います](./daa2aaa6cc9907.md#iam-ユーザーへのアクセス許可を割り当てるためにグループを使います)

もちろん今でもそのような運用をしているケースはありますが、今では少しずつIAM Identity Centerを使ったアプローチを取ることが増えてきました。

詳細はIAM Identity Centerについてもご参照ください。
@[card](https://docs.aws.amazon.com/ja_jp/singlesignon/latest/userguide/what-is.html)

また、2025年11月のアップデートでaws cliから`aws login`で認証をしてアクセスすることも可能になったので、IAM Identity Centerを使っていないケースでもアクセスキーを発行しないでアクセスさせることが可能となりました。
@[card](https://dev.classmethod.jp/articles/aws-cli-aws-login/)

過去記事の[アクセスキーを共有しない](./daa2aaa6cc9907.md#アクセスキーを共有しない)がより強固に行いやすくなりました。素晴らしいことですね。

#### ワークロードが AWS にアクセスする場合に IAM ロールで一時的な資格情報を使用することを必須とする

AWS内のリソースからAWS内の別リソースにアクセスする場合はアクセスキーを使わずIAMロールで権限を渡してアクセスを提供する、というのは従来からのベストプラクティスでした。
[ロールを使用してアクセス許可を委任する](./daa2aaa6cc9907.md#ロールを使用してアクセス許可を委任する)
ここでいうワークロードとは記事中に以下のように書かれています。

> ワークロードとは、アプリケーションやバックエンドプロセスなど、ビジネス価値を提供するリソースやコードの集合体のことです。ワークロードには、Amazon S3からデータを読み取るリクエストなど、AWS のサービス へのリクエストを行うために認証情報を必要とするアプリケーション、運用ツール、コンポーネントが含まれる場合があります。

これはAWSアカウント内に存在することもあれば、そうでない箇所に存在することももちろんあります。
よくあるものだと以下のケースなどがそうではないでしょうか

- GitHub Actionsなどのビルド・デプロイ環境からAWSのサービスにアクセスする
- 他のクラウドやオンプレミス環境にアプリケーションがあるが、AWSのサービスを一部だけ使う

前者については、IAMのIDプロバイダを作成してAssumeRoleWithWebIdentityを使うことで実現できます。
実際にそれが可能になった段階で試した記事を自分も書いています(この記事では公式に対応してない頃に無理やり使ってみたという内容で、)。
[GitHub Actionsのトークンを使ってAWS公式のconfigure-aws-credentialsを使ってみた](./b012f69b49bec095b9f1.md)
後者については、2022年にリリースされたIAM Roles Anywhereを使うことで実現できます。
自分はあまり触ったことがないのですが以下などが参考になります。
@[card](https://aws.amazon.com/jp/blogs/news/extend-aws-iam-roles-to-workloads-outside-of-aws-with-iam-roles-anywhere/)
@[card](https://blog.serverworks.co.jp/iam_roles_anywhere)

割と大掛かりな設定が必要な印象はあります。

#### 多要素認証 (MFA) を必須とする

これは以前からずっとそうですね。
[MFAの有効化](./daa2aaa6cc9907.md#mfaの有効化)
現在はIAMユーザーのみならず、IAM Identity Centerによる認証もMFAが設定できます。
また、ルートユーザーおよびIAMユーザーで複数MFAデバイスの登録が可能になりました。
多要素というだけでなく、パスワードレス認証もやりやすくなっています。
AWSに限らず多要素認証を有効化することはセキュリティ上非常に大事です。

#### 長期的な認証情報を必要とするユースケースのためにアクセスキーを必要な時に更新する

アップデートによってアクセスキーを使わなくて良いケースが増えたものの、それでも未だにアクセスキーが必要なケースが現実には存在することがあります。

- アクセスキーはちゃんとローテーションする
  - 使用中でも新しいアクセスキーに定期的に入れ替えるなどする。
- 使わないアクセスキーは削除する

あたりがここでやってほしいと言っていることかと思います。
参考
@[card](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id-credentials-access-keys-update.html)

また、アクセスキーについては過去記事であれこれ言及しているのでそちらもご参照ください。

#### ルートユーザーの認証情報を保護するためのベストプラクティスに沿う

こちらも以前からあるものです。
ルートユーザーは通常のAWS利用においては必要ないですし万一漏洩するとすべての権限を奪われるので、必要なければ使わない・パスワードを十分複雑にしてMFAを設定する、は徹底しましょう。
以下のページに書いてあることをできる限り守りましょう。
@[card](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/root-user-best-practices.html)

#### 最小特権アクセス許可を適用する

#### AWS 管理ポリシーの使用を開始し、最小特権のアクセス許可に移行する

#### 未使用のユーザー、ロール、アクセス許可、ポリシー、および認証情報を定期的に確認して削除する

#### IAM ポリシーで条件を指定して、アクセスをさらに制限する

同じようなことに言及している項目なのでまとめました。
これも考え方自体は以前と同じです。
[最小限の特権を認める](./daa2aaa6cc9907.md#最小限の特権を認める)

今ではIAM Identity Centerを使った権限管理の方法が提供されているので、以下を読んだり実際に試したりしてみましょう。
@[card](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/access_policies.html)
@[card](https://dev.classmethod.jp/articles/introduction-2024-aws-iam-identity-center/)

実運用ではまず広めの権限を与えて、そこから絞り込んでいくことで最小権限を実現するケースが多いかと思います。
そのために有用なのがIAM Access Analyzerです。
IAM Access Analyzerを使うこと自体が以下のようにベストプラクティスの項目として規定されています。

#### IAM Access Analyzer を使用して、アクセスアクティビティに基づいて最小特権ポリシーを生成する

#### IAM Access Analyzer を使用して、リソースへのパブリックアクセスおよびクロスアカウントアクセスを確認する

#### IAM Access Analyzer を使用して IAM ポリシーを検証し、安全で機能的なアクセス許可を確保する

前述の通り。IAM Access Analyzerを使うと使ってない権限を拾いやすくなります。
また、IAM Access Analyzerを使うことで外部アクセスが可能になっているリソースの検出も行えます。
その他、IAMポリシーの検証まで行えます。
機能として提供されているものが3つもベストプラクティスに入っているのが特徴的ですね。

IAM Access Analyzerについては以下記事が非常にわかりやすいです。
@[card](https://dev.classmethod.jp/articles/introduction-2024-aws-iam-access-analyzer/)

#### 複数のアカウントにまたがるアクセス許可のガードレールを確立する

これはAWS Organizationsを使ったマルチアカウント運用が一般的になった今ならではのベストプラクティスかなと思います。
マルチアカウント運用をする際、役割ごとにアカウントを分けることがあります。
可能なAPI操作をAWSアカウント単位で制御し、そのAWSアカウントで意図しないAWS上の操作をさせないようにするのが[SCP（サービスコントロールポリシー）](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html)、意図せぬリソースへの操作を禁止するものが[RCP（リソースコントロールポリシー）](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps.html)です。
これらは許可ポリシーではなく拒否ポリシーのため、文字通りガードレールとしての役割を果たします。
逆に許可を与えるにはIAMポリシーによるリソースベース・IDベースのポリシーを使う必要があります。
このあたりについては以下をご参照ください。
@[card](https://docs.aws.amazon.com/ja_jp/prescriptive-guidance/latest/security-reference-architecture/organizations.html)

個人レベルでは中々意識することがないかもしれませんが、個人でOrganizationを作ってAWSアカウントを作ってもIAM Identity Centerによってログイン情報管理が容易なので、興味があれば試してみるとわかりやすいかなと思います。

#### アクセス許可の境界を使用して、アカウント内のアクセス許可の管理を委任する

IAMのPermissions Boundaryの話ですね。
説明が難しいところなのですが、IAMユーザーやロールのポリシーだけで制限をするのではなく、それ以外にPermissions Boundaryの設定をすることで、双方の条件にマッチした操作のみが行える、というものです。

イメージしづらいかもしれませんが、これにより「IAMユーザーやロールの作成権限は与えるけど、広めに許可ポリシーを作成したとしてもPermissions Boundaryに条件にマッチしない操作は行えない、という状態が作れるということです。
うっかり広めに権限を作っても実際に与えられる権限を制御できるという解釈かなと思います。
自分で書いてても難しいなと思いますが…。

詳しくは以下などをご参照ください。
@[card](https://docs.aws.amazon.com/ja_jp/prescriptive-guidance/latest/security-reference-architecture/organizations.html)
@[card](https://dev.classmethod.jp/articles/iam-permissions-boundary/)
@[card](https://pages.awscloud.com/rs/112-TZM-766/images/20230224_27th_ISV_DiveDeepSeminar_Permissionsboundary.pdf)

### 変化が与えた影響（ベストプラクティスとの対応）

ざっくりまとめると以下みたいになります。

- フェデレーション/一時クレデンシャル: Identity Centerと`aws sso login`で「長期アクセスキー不要」が現実的に。
  - アクセスキーが必要な場面がさらに絞られました。
- MFA: WebAuthn/FIDOと複数MFA対応で「必須化」が運用として成立。
  - 認証そのものもパスワードレスで行えるようになったうえでMFA複数登録が可能なので、よりセキュアに扱えます。
- 最小権限: Access Analyzerのポリシー生成・検証で具体化しやすい。
- 棚卸し: 未使用アクセス分析で「削る判断」が取りやすい。
  - 最小権限は難しいですがIAM Access Analyzerはその助けになります。
- ガードレール: OrganizationsのRCP/SCPとControl Towerで組織統制が前提に。
  - AWS OrganizationsとControl Towerによって複数AWSアカウントの作成・管理が簡単になりました。
    - そのうえで出来る操作をアカウントレベルで制御する手段が用意されるようになりました。
- ルート保護: ルートアクセス集中管理で組織全体での保護が可能に。
  - Organizationsを使うことで複数アカウントの管理を一元化し、それによりルートアカウント管理も一元化できます。
    - [メンバーアカウントのルートアクセスを一元化する](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_root-enable-root-access.html)

## まとめ

未だに5年前の記事にちょこちょこLikeしていただくのですが、色々変わったんだよなと思っていたのでこの期に5年前のとの変化について書いてみました。
AWSはマルチアカウント周りのサービスがあれこれ整備されてきていて、マルチアカウント前提のプラクティスも増えてきたように思います。
個人で使う分には過剰なものもありますが、個人レベルでも十分使えるやり方や考えもあります。
これからAWSアカウントを新規に使い始めるときは是非参考にしてください。

また、5年前の記事が完全に陳腐化したわけでは全くなく、新しいサービスを必ずしも使っていなかったり、とりあえず触ってみる段階であまり新しいサービスを使わないでアカウントにアクセスしたり、といったケースはまだあるので、そうしている場合は是非参照してみてください。
そして新しいサービスを使える部分があれば少しずつ試して使ってみるといいでしょう。

## 参考

実はこの記事を書こうと思った後に、以下の動画・スライドが公開されていることを知りました。

@[card](https://www.youtube.com/watch?v=kwpjwPijmjU)
@[card](https://speakerdeck.com/nrinetcom/iamnomaniatukunahua-2025)

「IAMのマニアックな話」を執筆された佐々木さんの動画・スライドです。
こちらもぜひ参考にしてみてください。
