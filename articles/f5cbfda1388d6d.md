---
title: "Console-to-codeで生成したコードでEC2インスタンスを作り、Amazon Qにレビューさせてみた"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [aws]
published: true
---

:::message
この記事は
[みすてむず いず みすきーしすてむず (3) Advent Calendar 2023](https://adventar.org/calendars/8615)[^1]
および
[AWS（Amazon Web Services） Advent Calendar 2023](https://qiita.com/advent-calendar/2020/aws2)の4日目
です。
:::

## はじめに

今年も[re:Invent](https://reinvent.awsevents.com/)の時期が終わりました。
11月〜12月にかけて、この時期はre:Inventに関係するしないにかかわらずAWSのアップデートがたくさん発表されます。
今回はその中でもConsole-to-codeというサービスについて触れてみます。

Console-to-codeは例に漏れずDeveloper's IOですでに触れられています。
[[アップデート] AWS Console-to-Code (Preview)が使用可能になりました #AWSreInvent](https://dev.classmethod.jp/articles/console-to-code/)

:::message
この昨日は本記事執筆の2023/12/4次点ではパブリックプレビューであることにご注意ください。
:::

## Condole-to-codeとは

引用したDeveloper's IOの記事にもありますが、あらためてこちらでも記載します。
Console-to-codeとは、コンソールの操作をもとにCloudFormationやCDKのコードを生成するサービスです。
公式ドキュメントもすでに提供されています。

[Generate code for your console actions using Console-to-Code](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/console-to-code.html?icmpid=docs_console_unmapped)

単純な試してみた記事だとDeveloper's IOと丸かぶりになってしまうので、本記事では公式ドキュメントの内容にもう少し絡めながらこのサービスに触れていきます。

## 使い方

Developer's IOの記事では単純なEC2インスタンス作成を記録してしていますが、もう少し複雑な操作のコードを生成します。

まずはEC2のコンソール上で任意の操作をします。

普通に作ってもおもしろくないので、起動テンプレートを作成しました。

下記のように設定しています。

![起動テンプレートの設定1](/images/Launch-template-settings1.png)
![起動テンプレートの設定2](/images/Launch-template-settings2.png)

詳細設定には別途作ったIAMロールを割り当てる設定をしています。
![起動テンプレートのIAMロール設定](/images/IAM-attachment-to-EC2.png)

この設定で起動テンプレートを作成しました。
![起動テンプレートの作成完了](/images/Complet-launch-template-creation.png)

この時点でConsole-to-codeを覗いてみましょう。
Console-to-codeはEC2のコンソール上からアクセスできます。

![Console-to-codeのアクセス](/images/Console-to-Code-on-Console.png)

すると、起動テンプレートの作成もちゃんと記録されています。

![起動テンプレートでのConsole-to-code](/images/Console-to-code-launch-template.png)

今回はCDKのTypeScriptコードを生成してみます。

生成したコードは以下のとおりです。
※アカウントIDは伏せています。

> Here is the generated infrastructure as code:

```typescript
import * as cdk from '@aws-cdk/core';
import * as ec2 from '@aws-cdk/aws-ec2';

export class CdkStack extends cdk.Stack {
  constructor(scope: cdk.Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);
    
    const launchTemplateData: ec2.CfnLaunchTemplate.LaunchTemplateDataProperty = {
      ImageId: 'ami-04c97e62cb19d53f1',
      MetadataOptions: {
        HttpTokens: 'required',
        HttpEndpoint: 'enabled',
        HttpPutResponseHopLimit: 2
      },
      IamInstanceProfile: {
        Arn: 'arn:aws:iam::xxxxxxxxxxxx:instance-profile/EC2-ssm'  
      },
      PrivateDnsNameOptions: {
        EnableResourceNameDnsARecord: true,
        EnableResourceNameDnsAAAARecord: true
      }
    };
    
    new ec2.CfnLaunchTemplate(this, 'LaunchTemplate', {
      LaunchTemplateName: 'test',
      LaunchTemplateData: launchTemplateData
    });
  }
}
```

> Reasoning:
> - Created a CdkStack class extending cdk.Stack to define the stack.
> - Imported required cdk modules for ec2.
> - Created the LaunchTemplateData property object from the CLI command.
> - Used the ec2.CfnLaunchTemplate construct to create the launch template with name 'test' and provided launch template data.

試しにAutoScaling Groupを作ってみたのですが、どうやらAPIコールをどんどん記録してくれそうですね。

![記録されるアクション](/images/Actions-to-be-recorded.png)

せっかくなのでコード化するのがとってもめんどくさい（当社比）AutoScaling Groupをオプション盛り盛りで作り、それをコード化してみましょう。

![AutoScaling Groupを作ったAPIコールの跡](/images/Try-to-create-Autoscaling-group.png)

このように複数の操作を組み合わせて選択できるのですが、どうやらたくさんのアクションを選ぶエラーになるようです。
一括選択ができるのですが、1つずつ選ぶと5個までしか選べないのでおそらくそこが限界です。
ということなので、できるところまで選択してコードを生成してみました。

> Here is the generated CDK code:

```typescript
import * as cdk from '@aws-cdk/core';
import * as autoscaling from '@aws-cdk/aws-autoscaling';
import * as ec2 from '@aws-cdk/aws-ec2';
import * as elbv2 from '@aws-cdk/aws-elasticloadbalancingv2';
import * as sns from '@aws-cdk/aws-sns';

export class CdkStack extends cdk.Stack {
  constructor(scope: cdk.Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);
    
    const vpc = new ec2.Vpc(this, 'VPC');
    
    const subnets = [
      vpc.selectSubnets({subnetType: ec2.SubnetType.PRIVATE}),
    ];
    
    const asg = new autoscaling.AutoScalingGroup(this, 'ASG', {
      vpc,
      vpcSubnets: subnets,
      instanceType: ec2.InstanceType.of(ec2.InstanceClass.T4G, ec2.InstanceSize.SMALL),
      machineImage: new ec2.AmazonLinuxImage(),
      desiredCapacity: 1,
      minCapacity: 1,
      maxCapacity: 1,
      healthCheck: autoscaling.HealthCheck.ec2(),
    });
    
    asg.scaleOnCpuUtilization('TargetTrackingPolicy', {
      targetUtilizationPercent: 50
    });
    
    const topic = new sns.Topic(this, 'SNSTopic');
    
    asg.notifyScaleUpEvents(topic);
    
    const lb = new elbv2.ApplicationLoadBalancer(this, 'LB', {
      vpc,
      internetFacing: false,
    });
    
    lb.addListener('Listener', {
      port: 80,
    });
    
    lb.addTarget(asg: asg);
  }
}
```

> Reasoning:
> - Imported necessary CDK modules
> - Created a VPC construct to match the VPC in the CLI commands
> - Selected private subnets from the VPC to match the subnet IDs in the CLI
> - Created an AutoScalingGroup with properties matched to the CLI, like desired/min/max capacity, instance type, etc.
> - Added scaling policy and SNS topic based on CLI commands 
> - Created an internal ALB and listener to match the CLI
> - Added the ASG as a target to the ALB

> Let me know if you would like me to explain any part of the generated code in more detail.

## 出力されたコードをAmazon Qにレビューさせてみた

ところで、re:Inventでは生成AI系のサービスについてたくさん発表されましたね。
その中で、おそらく今回の目玉の1つが[Amazon Q](https://aws.amazon.com/jp/q/)ではないでしょうか。
Amazon Qは誤解を恐れずにとってもざっくり言うとAWS版ChatGPTのようなもので、AWSの活用に関することをチャット形式で聞けるAIチャットです。
Amazon Qについて記事を書くと別の記事が書けてしまうので詳細はここでは触れませんが、やってくれることの中の1つにAWS Toolkit上からのCode Whispererの使用があります。
詳しくは以下記事を参照してみてください。
[[やってみた]Amazon Q in IDEsで、Amazon QとAmazon CodeWhispererの組み合わせを使ってみた #AWSreInvent](https://dev.classmethod.jp/articles/try-amazon-q-for-ides/)

AWS ToolkitをVSCodeに入れているので、サブメニューからRefactorを選択してみました。
![VSCodeでのAmazon Q利用](/images/AmazonQ_in_VSCode.png)

結果はこんな感じです。

![Amazon Qによるレビュー](/images/refactor_generated_code_by_Q.png)

これを使えばコンソールからコードを生成して、それをレビューしてもらうまでの一連の流れが効率化できる未来があるかもしれません。

## あとがき

実はEC2じゃないコードを生成しようかなと目論んでいたのですが、現時点ではEC2でしかできなかったのでとりあえずこちらで試しました。
適当にやってみた記事になってしまいましたが、今後の展開が楽しみなサービスですね。
EC2以外のサービスにも早く対応してくれると楽になるので期待したいです。
re:Inventでは生成系AIに関することが多数発表されたので、この記事でネタにしたサービスたちの精度が上がって効率的に運用できる未来が来ることを楽しみにしています。

[^1]: [みすてむず いず みすきーしすてむず](https://misskey.systems/) とは、オープンソースのプラットフォームMisskeyのインスタンスのひとつで、主にITに関わる人が参加しています。最近はXよりもそちらに入り浸っています。
