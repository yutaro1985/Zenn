---
title: "Cloud9の環境にVSCodeのRemote SSHで接続してみた"
emoji: "🔗"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","vscode","cloud9","ssm"]
published: true
---

:::message
この記事は
[みすてむず いず みすきーしすてむず (3) Advent Calendar 2023](https://adventar.org/calendars/8615)[^1]
および
[AWS（Amazon Web Services） Advent Calendar 2023 シリーズ2](https://qiita.com/advent-calendar/2020/aws2)
の15日目です。
:::

## はじめに

[Cloud9](https://aws.amazon.com/jp/cloud9/)とは、オンラインのIDE(統合開発環境)であり、ブラウザ上からエディタで開発できる環境を提供してくれます。
かつてはCloud9のサービス自体が独立していましたが、現在はAWSのサービスとして提供されています。
AWSではEC2インスタンス上で動く環境として構築されています
Cloud9のUI上で開発するのも悪くないのですが、やっぱりVSCodeが使いたい時ってありますよね。
ということで、今回はCloud9の環境にVSCodeのRemote SSHで接続してみます。

これをやるならEC2に開発環境を構築すればいいじゃないかという気がしないでもないですが、開発はCloud9の環境で行う必要があるケースなどあると思います。
そういったケースでいつも使用可能とは限らないですが、手段としてはこういう物があるよというのを今回は提供します。

※今回はVSCodeのRemote SSHを使いますが、同等の機能がある他のIDEでもできるかもしれません。

## Remote SSHとは

公式ドキュメントでは以下に記述があります。
@[card](https://code.visualstudio.com/docs/remote/ssh)

簡単に言うとサーバに対してのSSH接続を介して、サーバ上のファイルをローカルのVSCodeで編集できる機能です。
これを使うとローカルのVSCodeのプラグインを使用してサーバ上のファイルを編集しながら開発できます。
Remote SSHで接続した環境には専用の設定を個別で入れられます。

今回は普通にSSHで接続するケースと、2020年のアップデートで対応したセッションマネージャー経由の接続をするケース、そしてセッションマネージャーで接続するケースでAWS SSOを使うケースを紹介します。

## 普通にSSHで接続する

こちらの場合はパブリックサブネットにCloud9のインスタンスを作成し、そちらに対してSSHで接続します。

せっかくなので[先日の記事](../articles/cdktf-for-usual-terraform-users.md)のCDKTFを使ってCloud9の環境を作成してみます。
IAMユーザー名の部分は環境変数で渡すようにしています。
※もちろん手動で作っても問題ありません。

```typescript
import { Cloud9EnvironmentEc2 } from "@cdktf/provider-aws/lib/cloud9-environment-ec2";
import { Cloud9EnvironmentMembership } from "@cdktf/provider-aws/lib/cloud9-environment-membership";
import { DataAwsIamUser } from "@cdktf/provider-aws/lib/data-aws-iam-user";
import { DataAwsInstance } from "@cdktf/provider-aws/lib/data-aws-instance";
import { Eip } from "@cdktf/provider-aws/lib/eip";
import { AwsProvider } from "@cdktf/provider-aws/lib/provider";
import { App, Fn, TerraformOutput, TerraformStack, Token } from "cdktf";
import { Construct } from "constructs";
import { Vpc } from "./.gen/modules/vpc";

interface MyConfig {
  name: any;
}

class MyStack extends TerraformStack {
  constructor(scope: Construct, id: string, config: MyConfig) {
    super(scope, id);

    new AwsProvider(this, "aws", {
      region: "ap-northeast-1",
    });

    // create vpc from module
    const vpc = new Vpc(this, "vpc", {
      projectName: "advent_calendar",
      env: "dev",
      vpcCidrBlock: "10.0.0.0/16",
      azSuffixes: ["a", "c", "d"],
      createSsmEndpoint: true,
    });

    const public_subnets = Fn.lookup(
      Token.asAnyMap(vpc.outputsOutput),
      "public_subnets"
    );
    const example = new Cloud9EnvironmentEc2(this, "example", {
      instanceType: "t3.medium",
      name: config.name,
      connectionType: "CONNECT_SSH",
      subnetId: Fn.element(public_subnets, 0),
      imageId: "amazonlinux-2-x86_64",
    });
    const cloud9Instance = new DataAwsInstance(this, "cloud9_instance", {
      filter: [
        {
          name: "tag:aws:cloud9:environment",
          values: [example.id],
        },
      ],
    });
    const cloud9Eip = new Eip(this, "cloud9_eip", {
      domain: "vpc",
      instance: Token.asString(cloud9Instance.id),
    });
    const iamUserName = process.env.MYIAM as string;

    const myuser = new DataAwsIamUser(this, "myuser", {
      userName: iamUserName,
    });
    const awsCloud9EnvironmentMembershipTest = new Cloud9EnvironmentMembership(
      this,
      "test_2",
      {
        environmentId: example.id,
        permissions: "read-write",
        userArn: myuser.arn,
      }
    );
    /*This allows the Terraform resource name to match the original name. You can remove the call if you don't need them to match.*/
    awsCloud9EnvironmentMembershipTest.overrideLogicalId("test");

    let outputs = new Map();
    outputs.set("cloud9_public_ip", cloud9Eip.publicIp);
    outputs.set("cloud9_instance_id", cloud9Instance.id);

    new TerraformOutput(this, "outputs", {
      value: outputs,
    });
  }
}

const app = new App();
new MyStack(app, "aws_instance", { name: "advent_calendar" });

app.synth();
```

ローカルからはAssumeRoleでロールを参照して実行しているので、そのままだとCloud9のインスタンスを自分のIAMユーザーで参照できません。
そこで自分のIAMユーザーに対してCloud9の環境を共有しています。
Cloud9の構築のコードはドキュメントのサンプルほぼそのまんまです。
今回はUbuntuではなくAmazon Linux2[^2]を使っています。

これでSSHでアクセス可能なCloud9のインスタンスができました。
ただし、この状態ではCloud9のインスタンスにアクセスできるキーペアがないので、適当な鍵を作ってCloud9のインスタンスの`~/.ssh/authorized_keys`に登録しておきます。
SSHの鍵の作り方は以下の記事などを参考にしてみてください。
[新しい SSH キーを生成して ssh-agent に追加する](https://docs.github.com/ja/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent#generating-a-new-ssh-key)
（新しい SSH キーを生成するのところだけでよいです。）
使えるオプションなどは下記などを参照
[【 ssh-keygen 】コマンド――SSHの公開鍵と秘密鍵を作成する](https://atmarkit.itmedia.co.jp/ait/articles/1908/02/news015.html)

今回はローカル端末で下記のコマンドを実行します。
（Cloud9上で作ってローカルに秘密鍵をコピーするのも最悪まあいいでしょう）

```bash
ssh-keygen -t ed25519 -f ~/.ssh/cloud9
```

これで`~/.ssh/cloud9`と`~/.ssh/cloud9.pub`ができます。
`~/.ssh/cloud9.pub`の中身をCloud9のインスタンスの`~/.ssh/authorized_keys`に登録します。

色んなやり方がありますが、今回はローカル側で`cat ~/.ssh/cloud9.pub`で鍵の内容を表示し、Cloud9の方で`vim ~/.ssh/authorized_keys`を実行して、`i`で挿入モードに入り、`~/.ssh/cloud9.pub`の内容を貼り付けます。
以下のように公開鍵を貼り付けたら`:wq`で保存してvimを終了します[^3]。
せっかくCloud9を使っているので、Cloud9上からauthorized_keysを編集してもよいですね。

![authorized_keys](/images/connect-cloud9-via-remote-ssh/authorized_keys.png)

今回は方式をED25519にしています。
ED25519はRSAよりもセキュアで、鍵の長さも短くて済むのでお勧めです。
ただ、ものによっては対応してないケースがあるのでその場合はECDSAを使うか、それえもダメならRSAを使ってください。
古いドキュメントとかだと当然のようにRSAを使っているケースがほとんどですが、その場合は4096bit以上を使うことを推奨します。

Cloud9の画面上のターミナルでは自分がアクセスしているIAMユーザー名が表示されていますが、Amazon Linux2では`ec2-user`でアクセスしていることになっています。

```bash
xxxxxxxxxx:~/environment $ whoami
ec2-user
```

また、EC2のコンソールからCloud9が立てたインスタンスを確認し、セキュリティグループを編集してSSHの接続許可を追加します。
一応、追加するのはマイIPだけにしておきましょう。

![セキュリティグループ](/images/connect-cloud9-via-remote-ssh/security_group.png)

では設定したらローカルからsshで接続できることを確認してみましょう。
初回はフィンガープリントが表示されるので`yes`を入力します。
接続に必要なパブリックIPアドレスはEC2のコンソールから確認できます。

```bash
ssh -i .ssh/cloud9 ec2-user@xxx.xxx.xxx.xxx                                        
The authenticity of host 'xxxxxxxxxxxx' can't be established.
ED25519 key fingerprint is SHA256:MLPYLDF3kwGk3DYOUp/bZ7U/OU8uwEIIflR8a8QQq8Y.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'xxxxxxxxx' (ED25519) to the list of known hosts.
   ,     #_
   ~\_  ####_        Amazon Linux 2
  ~~  \_#####\
  ~~     \###|       AL2 End of Life is 2025-06-30.
  ~~       \#/ ___
   ~~       V~' '->
    ~~~         /    A newer version of Amazon Linux is available!
      ~~._.   _/
         _/ _/       Amazon Linux 2023, GA and supported until 2028-03-15.
       _/m/'           https://aws.amazon.com/linux/amazon-linux-2023/
```

ここまでできれば後はVSCodeのRemote SSHで接続するだけです。
概ね以下記事のとおりなのでこちらも参考にしてください。
@[card](https://code.visualstudio.com/docs/remote/ssh-tutorial)

ローカル端末側の`~/.ssh/config`に以下のように接続情報を追記します。

```ssh
Host cloud9test
HostName (Cloud9のインスタンスのパブリックIP)
  Port 22
  User ec2-user
  IdentityFile ~/.ssh/cloud9
```

これでVSCodeのRemote SSHからこの設定内容を指定できます。
VSCodeのRemote SSHの拡張機能をインストールしたら、左下の歯車アイコンの下に`><`みたいなアイコンがあるので、そこをクリックして`Remote-SSH: Connect to Host...`を選択します。（言語設定により表示が異なります）

うまく設定できてれば、VSCodeでCloud9のインスタンスに接続できます。
「フォルダを開く」で任意のディレクトリを選択して開くとCloud9（が動いているEC2インスタンス）上のディレクトリをローカルのVSCodeで開けます。

## セッションマネージャーで接続する

ここまでは普通の方法です。
次は2020年のアップデートで対応したセッションマネージャー経由の接続をするケースです。
セッションマネージャーはEC2インスタンス上にSSHのプロトコルを使うことなく接続できる機能です。
詳細は以下ドキュメントや記事などを参照
@[card](https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/session-manager.html)
@[card](https://dev.classmethod.jp/articles/ec2-access-with-session-manager/)
これを使うとSSHのポートを開ける必要がなくなります。
また、設定が揃っていればプライベートサブネットのインスタンスにも接続できます。

セッションマネージャーの機能の中に、SSHのポートを許可していない状態でもにSSHコマンドで接続できるというものがあり、今回はそれを使います。
@[card](https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/session-manager-getting-started-enable-ssh-connections.html)

セッションマネージャーについては過去に別記事を書いているのでよかったらこちらもご参照ください。
https://zenn.dev/yutaro1985/articles/f55d41fd88f83e

では、SSMで接続する設定でCloud9の環境を作成してみます。
さっきのコードの`Cloud9EnvironmentEc2`の`connectionType`を`CONNECT_SSM`に変更します。
また、SubnetIdをプライベートサブネットに変更します。

変更箇所のみ抜粋します。

```typescript
    const private_subnets = Fn.lookup(
      Token.asAnyMap(vpc.outputsOutput),
      "public_subnets"
    );
    const example = new Cloud9EnvironmentEc2(this, "example", {
      instanceType: "t3.medium",
      name: config.name,
      connectionType: "CONNECT_SSM",
      subnetId: Fn.element(private_subnets, 0),
      imageId: "amazonlinux-2-x86_64",
    });
```

実際には変更してデプロイしようとしたら削除できずエラーになったので、一度`cdktf destroy`したり、できないものは手動で消したりしてから再度`cdktf deploy`しました。
（恐らくrole-session-nameを指定していないせいでAssumeRoleするときのセッション名がランダムになってしまいCloud9の所有者が作成者と紐づかなくなってしまうのが原因だと思われるので、環境変数その他の方法でちゃんと指定しましょう。）

### 通常のIAMによるアクセス

セッションマネージャーでSSH鍵を使った接続をできるようにし、sshコマンドを使ってアクセスできる状態に持っていきます。
鍵をAuthorized_keysに追記するところは普通にSSHするときと同じなので割愛します。

公式ドキュメントに従うと`~/.ssh/config`に以下のように設定することになります。

```ssh
# SSH over Session Manager
host i-* mi-*
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
```

しかし、これの問題点はすべてのインスタンスに対して、aws cliが認識するデフォルトのプロファイルを使って接続しようとすることです。

今回は対象がCloud9のインスタンスですので、基本的に対象となるインスタンスが変わることはあんまりないでしょう。
ということで、代わりに以下のように記述します。

```ssh
Host cloud9ssm
HostName i-xxxxxxxxxxxxxxxxxxxxxx # ここにはCloud9の環境であるEC2インスタンスのIDを指定します
  Port 22
  User ec2-user
  IdentityFile ~/.ssh/cloud9
  ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters portNumber=%p --profile (AWS CLIのプロファイル名)"
```

すでにaws cliでセッションマネージャーが実行できるプロファイルを設定済みであることが条件です。
セッションマネージャーを使うためだけの権限を持ったIAMユーザーを作成してもよいですが、AssumeRoleを使うなどするとよりセキュアになります。
記事にあるようにMFAを設定することも可能ですが、MFAを設定している状態でこれをやるには[awsmfa](https://github.com/future-architect/awsmfa)などのツールを使わないとやや面倒です。

@[card](https://blog.serverworks.co.jp/tech/2018/07/25/cliassumerole/)

このあたりも細かく説明したいのですが話がそれるので今回はすみませんが割愛します…。

このように`~/.ssh/config`に設定しておくと、`ssh cloud9ssm`でCloud9のインスタンスに接続できます。

あとは通常のSSHのときと同じく、VSCodeのRemote SSHでここで設定した接続設定を使ってあげれば接続できるはずです。

※実はこれと同じことをやってみた記事がすでに存在しますのでそちらもぜひ参考にしてみてください。
@[card](https://dev.classmethod.jp/articles/how-to-use-aws-cloud-9-with-vscode-remote-ssh/)
そもそも、[公式ブログにやり方が書いてあったりもします](https://aws.amazon.com/jp/blogs/architecture/field-notes-use-aws-cloud9-to-power-your-visual-studio-code-ide/)。

### AWS SSOによるアクセス

さて、それではAWS SSOを使う場合はどうなるでしょうか。
この場合、AWS SSOの認証情報をどうやって設定するかが問題になります。
ここで細かく説明するとまた長くなるので、ここは先人の知恵をお借りします。

@[card](https://dev.classmethod.jp/articles/aws-cli-for-iam-identity-center-sso/)
@[card](https://blog.serverworks.co.jp/sso-gui-cli-access)

ざっくり要約すると以下のようになります。

- aws cli v2をインストールする
  - v1は非対応です
- `aws configure sso`を使って認証プロファイルを作成する。
- `~/.aws/config`に追加された情報を確認する
  - プロファイルが追加されているので、必要があれば追加されたプロファイルを任意の名称に変更する
- 実際に使用する際は`aws sso login --sso-session hoge`を実行するとブラウザ上でSSOの許可を求められるので、該当のAWS SSOアカウントの情報を入力するか、すでにログイン済みのブラウザで許可する

ここまで実行すると、プロファイルを指定することでAWS SSOの認証情報を使ってaws cliを実行できるようになります。
あとは通常のIAMによるアクセスのときと同様に`~/.ssh/config`にここで設定したプロファイルを指定した設定を追記すれば同様に接続できるようになります。
SSOで入ったアカウントからさらにSwitch Roleで別のロールを使うこともあるかと思いますが、その場合はここで設定したプロファイルをsource_profileに設定し、AssumeRoleをするためのprofileを作成すればOKです。
既に貼りましたが、設定の仕方は以下の記事が参考になります。
@[card](https://blog.serverworks.co.jp/tech/2018/07/25/cliassumerole/)

### IAMおよびSSOでのアクセスの注意事項

IAMユーザーでMFAを設定している場合や、SSOの場合は認証の期限が切れたら再度認証する必要があります。
また、当然ながらCloud9のインスタンスが停止している場合は接続できませんので、Cloud9のインスタンスが起動されている状態で行ってください。

## まとめ

長くなりましたが、Remote SSHを使ってCloud9の環境に接続する方法を紹介しました。
ローカルに好きな環境を作って開発できるときは良いのですが、Cloud9を使う必要があるケースもあるはずですので、そういった際に参考になれば幸いです。

[^1]: [みすてむず いず みすきーしすてむず](https://misskey.systems/) とは、オープンソースのプラットフォームMisskeyのインスタンスのひとつで、主にITに関わる人が参加しています。最近はXよりもそちらに入り浸っています。
[^2]: Cloud9のインスタンスは記事執筆時点ではまだAmazon Linux2023に対応していません。ものによっては必要なライブラリを新しいバージョンにするのが面倒だったりするので、気になる方はUbuntuを使ってもよいでしょう。
[^3]: vimの細かい使い方についてはここでは説明しません。
