---
title: "Terraformを普段触っている人がCDK for Terraformを触ってみた"
emoji: "🧐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform","cdk","cdktf"]
published: true
---

:::message
この記事は
[みすてむず いず みすきーしすてむず (3) Advent Calendar 2023](https://adventar.org/calendars/8615)[^1]
および
[AWS CDK Advent Calendar 2023](https://qiita.com/advent-calendar/2023/aws-cdk)
の12日目です。
:::

## はじめに

[CDK for Terraform](https://developer.hashicorp.com/terraform/cdktf?product_intent=terraform)(以下、CDKTFと記述します。)がGAされてから1年以上経ちました。
[CDK for Terraform Is Now Generally Available](https://www.hashicorp.com/blog/cdk-for-terraform-now-generally-available)

リリース当初はさすがにまだ情報も少なく、またCDKとTerraform双方の知見が要求されるので中々新規で手を出すのも難しかったように思っています。
ただ、一部のPJでCDKTFを見るようになったのと、実際に使ってみた人の意見（数は少ないですが）を聞くと、CDKの書き味でTerraformが扱えることでかなり良さそうな感想を挙げていました。

中々実務で触る機会がなく手を出せていなかったのですが、ちょうど直近でCDKTFを使うPJに関わりそうになったのと、個人的にも気になっていたので実際に本記事で試してみます。

## CDK for Terraformについて

まずはそれぞれについて簡潔に説明します。

### CDKとは

[CDK](https://aws.amazon.com/jp/cdk/)とはAWSが提供しているInfrastructure as Codeのためのフレームワークです。
CDKは内部的にはCloudFormationを使っており、ざっくり言うと通常はjsonかYAMLで記述するCloudFormationテンプレートを、プログラミング言語で書けるようにしたフレームワークです。
この記事を書いた2023年12月12日現在ではTypeScript, Python, Java, C#、Goで記述できます[^2]。
詳細はAWSのドキュメントをご参照ください。
[AWS CDK とは](https://docs.aws.amazon.com/ja_jp/cdk/v2/guide/home.html)

内部で動いているのがCloudFormationであるため、基本的にはCloudFormationで扱えるもの[^3]、つまりAWSのサービスや機能が対象になります。

大きなメリットとして、AWSが公式で提供しているものなのでAWSのサポートを受けられることが挙げられます。

### Terraformとは

TerraformとはHashiCorpが提供しているInfrastructure as Codeのためのツール[^4]です。
Terraformには[Provider](https://developer.hashicorp.com/terraform/language/providers)と呼ばれるプラグインのようなものがあり、Terraform公式が提供しているProviderの他にSaaSサービス公式のProviderや、サードパーティのProviderなど多数のProviderが存在します。

どのようなProviderが存在するかは以下から確認できます。
[Terraform Registry](https://registry.terraform.io/browse/providers)
Terraform公式のProvider以外はおおよそですが対象サービスのAPIに対するラッパーとなっています。
設定値を記述することによりサービスのAPIを介して設定するような動きをする、とイメージすればよいです。
Terraformでは、HCLというHashiCorpが提供している独自の言語で記述します。

### CDK for Terraformとは

CDK for Terraform(CDKTF)は、ざっくりいうとCDKの裏側で動かすのをCloudFormationではなくTerraformにしたものです。
CDKを使い、Terraformを介してさまざまなサービスに対する設定を記述できます。
CDK同様、2023年12月12日現在ではTypeScript, Python, Java, C#、Goで記述できます。

[公式ドキュメント中](https://developer.hashicorp.com/terraform/cdktf)にある以下の図がわかりやすいかと思います。

※引用した図です
![CDKTFのイメージ](/images/cdktf-for-usual-terraform-users/cdktf-image.png)

現時点ではメジャーバージョンはまだ0台ですので、これが1になるころにはもしかしたら大きく変更されている可能性があります。

## CDKTFを試してみる

まずは兎にも角にも公式チュートリアルを触ってみましょう。
実は過去にGolangでCDKTFのチュートリアルをやったことがあるのですが、CDKのデファクトはTypeScript、次点がPythonという位置付けになっているので、ここでは一番メジャーと思われるTypeScriptでやってみます。

また、正確な時期はあまり把握してないのですが、2023年のいつごろからか[^5]AWS Terraform Providerの公式ドキュメント上にCDKTFのTypeScriptおよびPythonのサンプルも掲載されるようになりました。
この事により少しだけ書きやすくなったのではないでしょうか。

なお、自分は普段TypeScriptを全然書かないのでそのあたりでハマる可能性が十分にあります。

というわけでやってみましょう。
今回は以下をやってみます。
[Build AWS infrastructure with CDK for Terraform](https://developer.hashicorp.com/terraform/tutorials/cdktf/cdktf-build?variants=cdk-language%3Atypescript)

やってみたところ、さっそくハマりました。

```ts
main.ts(3,29): error TS2307: Cannot find module '@cdktf/provider-aws/lib/provider' or its corresponding type declarations.
main.ts(4,26): error TS2307: Cannot find module '@cdktf/provider-aws/lib/instance' or its corresponding type declarations.
```

これはどうやら以下のIssueの問題のようです
[cdktf forces latest aws provider](https://github.com/hashicorp/terraform-cdk/issues/3188)
さすがにTutorialのほうを更新して欲しいですね…。
ということなので、ビルド済みプロバイダをプロジェクトにインストールして進めます。

```bash
npm install @cdktf/provider-aws
```

また、せっかくなのでEC2に使うAMIは最新のAmazon Linux2023のAMIをパラメータストアから参照するようにしました。

```ts
import { Construct } from "constructs";
import { App, TerraformOutput, TerraformStack } from "cdktf";
import { AwsProvider } from "@cdktf/provider-aws/lib/provider";
import { Instance } from "@cdktf/provider-aws/lib/instance";
import { DataAwsSsmParameter } from "@cdktf/provider-aws/lib/data-aws-ssm-parameter";

class MyStack extends TerraformStack {
  constructor(scope: Construct, id: string) {
    super(scope, id);

    // define rcdesources here
    new AwsProvider(this, "aws", {
      region: "ap-northeast-1",
      profile: "your-profile",  // プロファイルは別途設定済みのものを指定します
    });

    const amazonLinux2023Latest = new DataAwsSsmParameter(
      this,
      "AL2023latest",
      {
        name: "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64",
      }
    );

    const ec2Instance = new Instance(this, "compute", {
      ami: amazonLinux2023Latest.value,
      instanceType: "t2.micro",
    });

    new TerraformOutput(this, "public_ip", {
      value: ec2Instance.publicIp,
    });
  }
}

const app = new App();
new MyStack(app, "aws_instance");

app.synth();
```

チュートリアルではリモートにtfstateを保存するようにしていますが、今回はローカルにしています。
もちろん実運用上はS3なりTerraform Cloudなりを使いましょう。

これで`cdktf synth`をするとjsonでTerraformの実行ファイルがローカルに出力されます。
これ自体に`terraform plan`や`terraform apply`を実行できますが、もちろん今回は以後もCDKTFを使っていきます。

`cdktf deploy`を実行すると以下のような出力があります。

```hcl
aws_instance  Initializing the backend...
aws_instance  
              Successfully configured the backend "local"! Terraform will automatically
              use this backend unless the backend configuration changes.
aws_instance  Initializing provider plugins...
              - Finding hashicorp/aws versions matching "5.30.0"...
aws_instance  - Installing hashicorp/aws v5.30.0...
aws_instance  - Installed hashicorp/aws v5.30.0 (signed by HashiCorp)

              Terraform has created a lock file .terraform.lock.hcl to record the provider
              selections it made above. Include this file in your version control repository
              so that Terraform can guarantee to make the same selections by default when
              you run "terraform init" in the future.
aws_instance  Terraform has been successfully initialized!
              
              You may now begin working with Terraform. Try running "terraform plan" to see
              any changes that are required for your infrastructure. All Terraform commands
              should now work.

              If you ever set or change modules or backend configuration for Terraform,
              rerun this command to reinitialize your working directory. If you forget, other
              commands will detect it and remind you to do so if necessary.
aws_instance  - Fetching hashicorp/aws 5.30.0 for linux_amd64...
aws_instance  - Retrieved hashicorp/aws 5.30.0 for linux_amd64 (signed by HashiCorp)
              - Obtained hashicorp/aws checksums for linux_amd64; Additional checksums for this platform are now tracked in the lock file
aws_instance  Success! Terraform has updated the lock file.

              Review the changes in .terraform.lock.hcl and then commit to your
              version control system to retain the new checksums.
aws_instance  data.aws_ssm_parameter.AL2023latest (AL2023latest): Reading...
aws_instance  data.aws_ssm_parameter.AL2023latest (AL2023latest): Read complete after 0s [id=/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64]
aws_instance  Terraform used the selected providers to generate the following execution plan.
              Resource actions are indicated with the following symbols:
                + create

              Terraform will perform the following actions:
aws_instance    # aws_instance.compute (compute) will be created
                + resource "aws_instance" "compute" {
                    + ami                                  = (sensitive value)
                    + arn                                  = (known after apply)
                    + associate_public_ip_address          = (known after apply)
                    + availability_zone                    = (known after apply)
                    + cpu_core_count                       = (known after apply)
                    + cpu_threads_per_core                 = (known after apply)
                    + disable_api_stop                     = (known after apply)
                    + disable_api_termination              = (known after apply)
                    + ebs_optimized                        = (known after apply)
                    + get_password_data                    = false
                    + host_id                              = (known after apply)
aws_instance  + host_resource_group_arn              = (known after apply)
                    + iam_instance_profile                 = (known after apply)
                    + id                                   = (known after apply)
                    + instance_initiated_shutdown_behavior = (known after apply)
                    + instance_lifecycle                   = (known after apply)
                    + instance_state                       = (known after apply)
                    + instance_type                        = "t2.micro"
                    + ipv6_address_count                   = (known after apply)
                    + ipv6_addresses                       = (known after apply)
                    + key_name                             = (known after apply)
                    + monitoring                           = (known after apply)
                    + outpost_arn                          = (known after apply)
                    + password_data                        = (known after apply)
aws_instance  + placement_group                      = (known after apply)
                    + placement_partition_number           = (known after apply)
                    + primary_network_interface_id         = (known after apply)
                    + private_dns                          = (known after apply)
                    + private_ip                           = (known after apply)
                    + public_dns                           = (known after apply)
                    + public_ip                            = (known after apply)
                    + secondary_private_ips                = (known after apply)
                    + security_groups                      = (known after apply)
                    + source_dest_check                    = true
                    + spot_instance_request_id             = (known after apply)
                    + subnet_id                            = (known after apply)
                    + tags_all                             = (known after apply)
aws_instance  + tenancy                              = (known after apply)
                    + user_data                            = (known after apply)
                    + user_data_base64                     = (known after apply)
                    + user_data_replace_on_change          = false
                    + vpc_security_group_ids               = (known after apply)
                  }

              Plan: 1 to add, 0 to change, 0 to destroy.
              
              Changes to Outputs:
                + public_ip = (known after apply)
              
              Do you want to perform these actions?
                Terraform will perform the actions described above.
                Only 'yes' will be accepted to approve.

Please review the diff output above for aws_instance
❯ Approve  Applies the changes outlined in the plan.
  Dismiss
  Stop
```

Approveを選択して実行すると、`terraform apply`が実行されます。

今回はVPCも何も指定していないのでデフォルトのVPCにEC2が作成されます。
ちゃんとAWSのアカウントのセキュリティ周りをちゃんと設定している人だとデフォルトVPCを削除していることもあるかと思いますので、そういった場合は配置されるサブネットもちゃんと指定しましょう。
また、セキュリティグループも指定していないのでVPCデフォルトのセキュリティグループが適用されます。

```hcl
aws_instance  Enter a value: yes
aws_instance  aws_instance.compute (compute): Creating...
aws_instance  aws_instance.compute (compute): Still creating... [10s elapsed]
aws_instance  aws_instance.compute (compute): Still creating... [20s elapsed]
aws_instance  aws_instance.compute (compute): Still creating... [30s elapsed]
aws_instance  aws_instance.compute (compute): Still creating... [40s elapsed]
aws_instance  aws_instance.compute (compute): Creation complete after 42s [id=i-088a26347b9148f36]
aws_instance  
              Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
              
              Outputs:
aws_instance  public_ip = "xxx.xxx.xxx.xxx"

  aws_instance
  public_ip = xxx.xxx.xxx.xxx
```

以下のようにちゃんとAmazon Linux2023の最新AMIでEC2インスタンスが作成されていますね。

![cdktf deploy実行後](/images/cdktf-for-usual-terraform-users/deployed.png)

ではいったんここで作成したEC2は壊します。
`cdktf destroy`が`terraform destroy`にあたるコマンドです。
※destroyの結果は省略します。

この時点でのコードは以下に保存しています。

@[card](https://github.com/yutaro1985/learn-cdktf-ts/tree/1962f31dd981a07e8528e8487c715c8e3a256077)

## moduleを使ってVPCを作ってみる

ここまでだとチュートリアルをそのままなぞっただけになってしまいます。
それではおもしろくないので、AWSの公式モジュールをラップした独自モジュールを作ってVPCを作るところまでやってみます。

### 公式モジュールについて

AWSは公式にTerraformのモジュールを公開しています。
その中にVPCモジュールが存在します。
[vpc module](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest)
VPCは非常に必要なパラメータと関連リソースが多く、自分でゼロから作るのは結構たいへんです。
とはいえ、公式モジュールもあらゆるケースに対応すべく非常に多くのパラメータがあるため、自分が使うときは公式モジュールをラップして使うことが多いです。
（これがいいのか悪いのかはよくわかりません…）

実際、こんなモジュールを自分で作ったりしてました。[^6]
@[card](https://github.com/yutaro1985/awesome-terraform-modules/tree/main/vpc)

同じコードなのでTypeScriptで書き直そうかなとも思ったのですが、cdktfには`cdktf convert`なるコマンドがあるそうなので、今回はそちらを使ってみます。

`cdktf convert`を使うにあたっては以下Scrapを参考にさせていただきました。
@[card](https://zenn.dev/cumet04/scraps/4323cf43d77b63)

以下のように実行します。
※ローカルにVPCモジュールを落としておき、そのディレクトリ内で実行しています。

```bash
cat *.tf | cdktf convert --language typescript > vpc.ts
```

~~出力されたコードはそのまま使えないので手直しが必要ですが、ゼロから作るよりは大分マシです。~~

結論からいうとこちらはリファクタにめちゃくちゃ時間かかりそうなので今回は断念しました。
convertされたものを手直しするよりもぶっちゃけ作り直したほうが早そうです。

その代わりと言ってはなんですが、GitHub上にあるHCLで書かれたモジュールをそのままプロジェクトに引っ張ってきて使ってみました。

`cdktf.json`のterraforModulesに以下のように記述します。

```json
{
  // terraformModules以外の記述は省略
  "terraformModules": [
    {
      "name": "vpc",
      "source": "git::https://github.com/yutaro1985/awesome-terraform-modules.git//vpc"
    }
  ]
}
```

これによりGitHub上に公開されているモジュールを参照できます。
この状態で`cdktf get`を実行すると`.gen`配下にディレクトリにモジュールのコードが生成されます。

今回はmain.tsを以下のように修正して、VPCを作成したうえでそのVPCのサブネット上にEC2インスタンスを作ってみます。

```typescript
import { Construct } from "constructs";
import { App, TerraformOutput, TerraformStack, Token, Fn } from "cdktf";
import { AwsProvider } from "@cdktf/provider-aws/lib/provider";
import { Instance } from "@cdktf/provider-aws/lib/instance";
import { DataAwsSsmParameter } from "@cdktf/provider-aws/lib/data-aws-ssm-parameter";
import { Vpc } from "./.gen/modules/vpc";

class MyStack extends TerraformStack {
  constructor(scope: Construct, id: string) {
    super(scope, id);

    new AwsProvider(this, "aws", {
      region: "ap-northeast-1",
      profile: "xxxxxxxxxxxxxx",
    });

    // create vpc from module
    const vpc = new Vpc(this, "vpc", {
      projectName: "advent_calendar",
      env: "dev",
      vpcCidrBlock: "10.0.0.0/16",
      azSuffixes: ["a", "c", "d"],
      createSsmEndpoint: true,
    });

    const amazonLinux2023Latest = new DataAwsSsmParameter(
      this,
      "AL2023latest",
      {
        name: "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64",
      }
    );
    const public_subnets = Fn.lookup(
      Token.asAnyMap(vpc.outputsOutput),
      "public_subnets"
    );
    const ec2Instance = new Instance(this, "compute", {
      ami: amazonLinux2023Latest.value,
      instanceType: "t2.micro",
      subnetId: Fn.element(public_subnets, Math.floor(Math.random() * 3)),
    });

    let outputs = new Map();
    outputs.set("public_ip", ec2Instance.publicIp);
    outputs.set("subnet", ec2Instance.availabilityZone);

    new TerraformOutput(this, "outputs", {
      value: outputs,
    });
  }
}

const app = new App();
new MyStack(app, "aws_instance");

app.synth();

```

この状態で`cdktf deploy`することで、VPCを作成しその上にEC2インスタンスが作成されます。

![VPCとEC2の構築完了](/images/cdktf-for-usual-terraform-users/deployed_vpc_and_ec2.png)

苦労したのはVPCモジュールからOutputを取り出す方法でした。
該当のモジュールでは`outputs`という名前でMapでまとめてVPCの情報を出力していました。
それはCDKTFでインポートした際には`vpc.outputsOutput`で取得できますが、これはそのままCDKTFのプログラミング言語でMapとして扱うことはできません。
これは[Tokens](https://developer.hashicorp.com/terraform/cdktf/concepts/tokens)を使って言語の型に変換しなければなりません。
前述の通りこのモジュールではMapでOuputsを出力しているので、TypeScriptのMapに変換します。
Tokenは同ファイル内でimportしておきます。

```typescript
Token.asAnyMap(vpc.outputsOutput)
```

さらに、以下記事を参考にTerrafornの`lookup`関数を使ってMapから値を取り出しています。
@[card](https://dev.classmethod.jp/articles/ckd_for_terraform_first_touch/)

これは`Fn`というライブラリによって使用できます。
[Functions](https://developer.hashicorp.com/terraform/cdktf/concepts/functions)

最終的には以下のようにしてpublic_subnetsのリストからランダムにサブネットを選択しています。
※やった後に思ったのですが、これをやると毎回planで差分が出てしまうはずなので微妙ですね…。

該当箇所だけの抜き出しです。

```typescript
    const public_subnets = Fn.lookup(
      Token.asAnyMap(vpc.outputsOutput),
      "public_subnets"
    );
    const ec2Instance = new Instance(this, "compute", {
      ami: amazonLinux2023Latest.value,
      instanceType: "t2.micro",
      subnetId: Fn.element(public_subnets, Math.floor(Math.random() * 3)),
    });

```

これによりVPCモジュールで作成したVPCのサブネットIDをEC2インスタンスに渡すことができました。
※この時点のソースは以下コミットにあります
@[card](https://github.com/yutaro1985/learn-cdktf-ts/tree/db1782f735daff4a9b2ea4a863ef97b8bf41c3e6)

## 感想

CDKTFを触ってみると、あまりTypeScriptに慣れていない自分でもIDEの強力な補完が効いたり、言語そのものの関数を使って値を設定できたりとメリットを感じることができました。
一方、今回の実験ではやっていないことなのですが、`cdk synth`で出力されるコードはjsonでありHCLではないので、trivyなどのセキュリティスキャンをかけたときにうまくいかないケースがあったりします。
具体的にはjsonをちゃんと認識できずに誤検出したり、コメントによる個別のignoreができなかったりします。
まだまだ発展途上ですが、Terraformを介してCDKでいろんなサービスを操れるのはケースによってメリットがありそうです。
デメリットはCDKそのものとTerraformと、対象のAPIとのそれぞれに学習コストが発生するので、やや学習コストが高くつくことでしょうか。
まだまだ発展途上なプロダクトなので今後に期待したいですね。

[^1]: [みすてむず いず みすきーしすてむず](https://misskey.systems/) とは、オープンソースのプラットフォームMisskeyのインスタンスのひとつで、主にITに関わる人が参加しています。最近はXよりもそちらに入り浸っています。
[^2]: なお、CDK自体はおおむねTypeScriptで作られています。
[^3]: 厳密にはそのうえでCDKがサポートしているサービスや機能が対象になります。
[^4]: TerraformはGolangで作られています。
[^5]: [こちら](https://www.hashicorp.com/blog/new-multi-language-docs-simplify-cdk-for-terraform-adoption)を見るに2023/7/24ごろからでしょうか。
[^6]: 本当はもっと作りたいのですが、中々手が回らず…。
