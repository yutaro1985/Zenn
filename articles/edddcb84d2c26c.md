---
title: "AWS ChatbotとSlackのWorkflowを使ってSlackから特定のインスタンスに特定のコマンドを実行させる"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","chatbot","slack","systemsmanager"]
published: false
---

:::message
この記事は[AWS Advent Calendar 2021](https://qiita.com/advent-calendar/2021/aws)の15日目です。
:::

re:Invent関連で何か書くとはいうものの、re:Inventでのアップデート周りはだいたいクラスメソッドさんの[Developers IO](https://dev.classmethod.jp/)で書かれてしまうので、被らないネタを探すのが難しいですね。

このネタも実は[かぶっている](https://dev.classmethod.jp/articles/aws-chatbot-supports-managing-aws-resources-in-slack/)のですが、せっかくやっとSlackのほうからコマンドを送れるようになったので、それを使ってサーバ上でコマンドを走らせてみよう、というのが今回の記事です。
今まではChatbotと言いながらも、Slackの側から何かをさせることはできませんでしたからね。
厳密にはLambda Functionを経由して何かをさせることはできなくもありませんでしたが、いちいちLambda Functionを作るのは面倒でした。
それがようやく簡単にできるようになったかという思いでした。

## やること

:::message
この機能は執筆時点でpreviewですので、正式にリリースされたときに仕様が変更になっている可能性があります。
:::

大まかに下記のことをやります。

1. AWS ChatbotからSlackのチャンネルに通知が送れるようにする
この部分はこれまでもできたことですね。
まず、AWS Chatbotを使ってSlackに通知が送れる状態にします。
設定方法は下記の記事が詳しいです。
[AWS再入門ブログリレー AWS Chatbot編](https://dev.classmethod.jp/articles/re-introduction-2020-aws-chatbot/)
2. Slackからaws cliを実行できるようにする
今回のアップデートによってできるようになったことです。こちらは先にも挙げたこの記事が詳しいです。
[[プレビュー]AWS ChatbotがSlackでのAWSリソース管理に対応しました！ #reinvent](https://dev.classmethod.jp/articles/aws-chatbot-supports-managing-aws-resources-in-slack/)
なお、制限がありすべてのコマンドが実行可能なわけではありません。
ここではEC2インスタンスへコマンドを送ること**のみ**をできるようにしたいので、権限を**ssmからコマンドを呼び出すことを行うことだけができる範囲に絞ります。**
3. Slackでワークフローを作り、特定のコマンドを呼び出せるようにする
一応、今回新規性のある部分です。
こちらで実際にコマンドを実行してみます。

**以後は上記の1,2が完了している前提で記載します。**

### Slackからコマンドが実行できる権限を設定する

aws cliを実行できるようにする手順の中で、Slackチャンネル側からどのコマンドを実行できるようにするかを、Chatbot側のガードレールとIAMロールの双方で指定する箇所があります。
本当はssmのsend-commandでやるつもりだったんですが、何度試してもSlackからのsend-commandがエラーになってしまったので、今回はAutomationによって実行することにしました。
その場合に必要な権限はおおよそ下記のとおりです。

※下記はIAMロールの権限のため、StartAutomationExecutionの権限を特定のAutomationのarnに絞っています。
これとは別にガードレールに権限を設定する必要がありますが、ガードレールの権限はこれよりも広い必要があります。
Slackチャンネルに与えたいMaxの権限をガードレール側では設定し、個別のIAMロールではガードレールの権限の範囲内で権限を設定します。
試すだけならガードレールとまったく同じ設定にしていてもOKです。
ここではとりあえずワイルドカードで指定しておいて、のちほどAutomationDocumentを作成したあとにarnを確認してから入力してももちろんOKです。
今回は特定のインスタンスのみを対象にRunShellScriptを実行させたいのでSendCommandの権限もそうしていますが、
権限が足りなかったりする場合はコンソールからエラー内容を確認して適切に権限を振ってください。

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "ssm:StartAutomationExecution",
            "Resource": "arn:aws:ssm:ap-northeast-1:xxxxxxxxxxxxx:automation-definition/HogeAutomation:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssm:ListCommands",
                "ssm:DescribeInstanceInformation",
                "ssm:ListCommandInvocations"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "ssm:SendCommand",
            "Resource": [
                "arn:aws:ssm:*:*:document/AWS-RunShellScript",
                "arn:aws:ec2:ap-northeast-1:xxxxxxxxxxxx:instance/i-xxxxxxxxxxxxxx"
            ]
        }
    ]
}
```

### SSM Automationドキュメントを作成する

Automationを実行するためにその内容を定義しましょう。
Systems Managerのコンソールから「ドキュメント」を選択し、Create documentからAutomationを選択します。
このとき、実行したいリージョンになっていることを確認しましょう。

![Systems Managerコンソールからドキュメント作成](/articles/images/SystemsManeger_document.png)

![Automation作成](/articles/images/Automation.png)

SSM Documentの編集は以前は面倒そのものでしたが、今はだいぶ楽になっています。
ただ、今回はやることが決まっているのでエディタから下記を入力します。

```yaml
schemaVersion: '0.3'
mainSteps:
  - name: CommandFromSlack
    action: 'aws:runCommand'
    inputs:
      InstanceIds:
        - i-xxxxxxxxxxxxxx
      DocumentName: AWS-RunShellScript
      Parameters:
        commands: echo "Hello,World!"
```

名前の欄はChatbotのロールやポリシーで個別にドキュメントを指定するならそちらと名前が一致する必要があります。
今回は特定のインスタンスIDに対してRunShellScriptで`echo "Hello,World!"`とコマンド実行させるだけのAutomationです。

![Automation作成画面](/articles/images/Document_editor.png)

### Slackでワークフローを作成する

あとはSlackでワークフローを作るだけです。
Chatbotを連携させたチャンネルでワークフローを作ります。
多分PCからやる必要があったはずです。

Slackの入力欄の左に雷のようなマークがあるのでそちらを押すと、選択肢にワークフロービルダーがあるのでそちらを選択します。

![Slack入力欄](/articles/images/Slack_lightning.png)を押して![ワークフロービルダーを開く](/articles/images/workflow_builder.png)を押す。

その後画面に従って作成していきます。

名前は任意です。
![ワークフローの名前](/articles/images/workflow1.png)
今回はショートカットからの起動を選択します。
![ショートカットで起動](/articles/images/workflow2.png)
次にこれはワークフローを起動するショートカットを追加するチャンネルを選ぶところなのでどこを選んでもかまいません。
![ショートカット追加チャンネルの選択](/articles/images/workflow3.png)

次にワークフローステップの追加で「メッセージの送信」を選択します。
![メッセージの送信の追加](/articles/images/workflow4.png)

その次にはメッセージの送信内容を定義します。
メッセージの送信先は、AWS Chatbotと連携したチャンネルにしてください。
テキストにはaws cliのコマンドを`@aws`に対してメンションで送ります。
このとき、正しく`@aws`へのメンションになっていることを確認してください。
ここではcliでstart-automation-executionを実行しています。
ドキュメント名は自分で作成したドキュメントの名前にしてください。
リージョンはドキュメントを作成したリージョンを明示的に指定しましょう。

```shell
@aws ssm start-automation-execution --document-name HogeAutomation --region ap-northeast-1
```

![メッセージの内容詳細設定](/articles/images/workflow5.png)

ここまで完了したら、「公開する」を押してワークフローを公開します。
このワークフローが同じ組織の中に公開されます。

ワークフローを作成したチャンネルで先程の雷のボタンを押すと、作成したワークフローがあるはずです。
試しに実行してみましょう。

![Slack入力欄](/articles/images/Slack_lightning.png)

うまく行けばAutomationが実行され、Automationの実行ログをコンソールで確認するとHello,World!が出力されているはずです。

## よかったこと

- エンジニアがサーバに入って特定のコマンドを実行しなければいけないケースでも、非エンジニアの方に実行してもらうことが可能になった
Webサーバの再起動など…。もちろん、いたほうがよいですが影響が少ないと言い切れるような場合の話です。

## その他

AWS側もドキュメントにおいてよくあるユースケースをまとめてくれていますのでぜひ参考にしてみてください。
[Using CLI commands with AWS Chatbot - Common use cases](https://docs.aws.amazon.com/ja_jp/chatbot/latest/adminguide/common-use-cases.html)

実は今回やっていることは*Run an Automation runbook*と大して変わりません。
運用にうまく活用していきたいですね。
