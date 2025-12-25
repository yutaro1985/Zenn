---
title: "Re:AWSを使うにあたりIAMのベストプラクティスをもう一度確認する 202512版"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","iam","security"]
published: false
---
:::message
この記事は[AWS（Amazon Web Services） Advent Calendar 2025](https://qiita.com/advent-calendar/2025/aws)の25日目です。
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

## IAM関連サービスのアップデート

## IAMベストプラクティスの主なアップデート
