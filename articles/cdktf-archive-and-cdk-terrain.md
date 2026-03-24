---
title: "archiveされたCDK for Terraformとコミュニティフォークのcdk-terrainを調べる"
emoji: "🧭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform","cdktf","opentofu","cdk"]
published: true
---

この記事は2026年3月時点の公開情報をもとに整理しています。状況は変わりうるため、採用前には公式リポジトリとリリースノートも併せて確認してください。

## はじめに

少し前に、以下の記事を書きました。

@[card](https://zenn.dev/yutaro1985/articles/cdktf-for-usual-terraform-users)

当時は、CDK for Terraform（以下、CDKTF）を「Terraformをプログラミング言語で扱うための選択肢」として触ってみる、という温度感で記事を書いていました。
ただ、その後CDKTFは大きく状況が変わりました。

HashiCorpはCDKTFをdeprecatedとし、2025年12月10日にGitHubリポジトリをarchiveしています。
一方で、そこで終わりではなく、コミュニティフォークとしてcdk-terrainが立ち上がっています。

この記事では、この2点を整理します。

### 3行まとめ

- CDKTFは2025年12月10日にarchiveされ、本家による保守は終了した
- 移行先としては、AWS CDKと密結合している場合はAWS CDK、それ以外はHCLベースのTerraformへの移行が本家から案内されている
- もうひとつの選択肢として、コミュニティフォークのcdk-terrainが登場している

## CDK for Terraform はどうなったのか

まず事実関係です。

HashiCorpのCDKTFリポジトリにはSunset Noticeが追加されています。CDKTFは2025年12月10日でsunsetし、その後はarchiveしてread-onlyにすると明記されています。

@[card](https://github.com/hashicorp/terraform-cdk)

公式ドキュメント側でもdeprecated扱いになっており、以後は保守・開発・互換性更新をしないことが案内されています。

@[card](https://developer.hashicorp.com/terraform/cdktf)

ここで重要なのは、単に新機能が増えないという話ではなく、今後のTerraform本体やProviderの更新に対して互換性が追従されない前提になったことです。
つまり、既存案件でただちに動かなくなるとは限らないものの、新規採用の心理的ハードルはかなり上がったと見てよさそうです。

### 本家が案内している移行先

HashiCorpは移行先として、まずHCLベースのTerraformへの移行を案内しています。
CDKTFからは `cdktf synth --hcl` によりHCLを出力し、それを起点に通常のTerraform運用へ移ることができます。

加えて、公式のsunset noticeでは、AWS CDKと密結合している場合はAWS CDKへの移行も案内されています。
逆に、AWS CDKを使っていない場合は、標準的なTerraformとHCLへの移行を強く推奨する、というのが本家の温度感です。

この方針は自然です。
なぜなら、長期的な保守性やエコシステムの中心という意味では、やはりTerraformの標準的な書き方はHCLだからです。

一方で、「CDKの書き味でTerraformを使いたい」というモチベーション自体がなくなったわけではありません。
そこで出てきたのがcdk-terrainです。

## cdk-terrain とは何か

cdk-terrainは、CDKTFのコミュニティフォークです。
Open Constructs Foundation配下で継続開発されており、READMEやFAQでは「community-driven continuation」という表現が使われています。

@[card](https://github.com/open-constructs/cdk-terrain)

@[card](https://cdktn.io)

FAQでは、HashiCorpのdeprecation announcementを受けて、コミュニティが継続開発するためのforkとして立ち上がったことが説明されています。

また、OpenTofu互換を重視していることはFAQでも明記されていますが、OpenTofuとは正式に提携した関係ではなく、独立したコミュニティ主導のプロジェクトとして運営されています。

@[card](https://github.com/open-constructs/cdk-terrain/blob/main/FAQ.md)

2026年3月時点では、少なくとも以下の点は確認できます。

- GitHub上で継続的にコミットされている
- v0.22.0がリリースされている
- 独自ドキュメントサイトと移行ガイドが用意されている
- Terraformに加えてOpenTofuも意識した立ち位置を取っている

つまり、単なる「とりあえずforkしました」という状態ではなく、継続を前提にした受け皿として動き始めている、と見てよさそうです。

## 何がどう変わるのか

cdk-terrainの最初の大きな変更は、プロジェクト名とパッケージ名の整理です。
v0.22.0のリリースノートでは、以下のrenameが案内されています。

@[card](https://github.com/open-constructs/cdk-terrain/releases/tag/v0.22.0)

```diff txt
- cdktf
+ cdktn

- cdktf-cli
+ cdktn-cli

- @cdktf/*
+ @cdktn/*
```

TypeScriptであれば、イメージとしては以下のような変更になります。

```diff ts
- import { App, TerraformStack } from "cdktf";
+ import { App, TerraformStack } from "cdktn";

- import { AwsProvider } from "@cdktf/provider-aws/lib/provider";
+ import { AwsProvider } from "@cdktn/provider-aws/lib/provider";
```

CLIも同様に変わります。

```diff bash
- cdktf get
- cdktf synth
- cdktf deploy
+ cdktn get
+ cdktn synth
+ cdktn deploy
```

一方で、互換性を意識して残されているものもあります。
移行ガイドでは、以下はそのままとされています。

| 項目 | 変更有無 | 備考 |
| ---- | ---- | ---- |
| `cdktf.json` | 変更なし | 設定ファイル名はそのまま |
| `cdktf.out/` | 変更なし | 出力先もそのまま |
| `CDKTF_*` 環境変数 | 変更なし | 既存運用を崩さないため |
| Terraform provider source | 変更なし | `hashicorp/aws` などはそのまま |
| state | 変更なし | 既存の `.tfstate` には直接影響しない |

このあたりは、既存CDKTFユーザーが移りやすいようにかなり配慮されています。

## 移行時に気を付けたいポイント

移行ガイドを見ると、基本方針はかなり明快です。
依存関係、import、CLIコマンドを `cdktf` 系から `cdktn` 系へ置き換えるのが中心です。

@[card](https://cdktn.io/docs/release/upgrade-guide-v0-22)

ただし、気を付けたい点もあります。

### prebuilt providerの状況を確認する

READMEやFAQでは、人気の高いプロバイダの事前ビルド済みパッケージをコミュニティで維持していく方針が示されています。
ただし、移行期には`@cdktf/provider-*`と`@cdktn/provider-*`の混在も起こりえます。

移行ガイドでも、二重依存は一時的には可能だが、長期的には推奨しないとされています。
型の衝突や依存関係の複雑化が起きうるためです。

そのため、対象のプロバイダが`@cdktn/provider-*`として公開されているかは先に確認したほうがよさそうです。
もし未整備なら、`cdktn get` によるローカル生成も選択肢になります。

### 対応言語は採用前に再確認したい

cdk-terrainのドキュメントを見ると、対応言語の説明にはまだ少し揺れがあります。

具体的には、READMEには「We currently support TypeScript, Python and Go.」とあります。
一方でFAQには、別の書き方があります。
次の記述です。

> Initial releases will support TypeScript and Python. Support for additional languages (Java, C#, Go) will be added based on community demand and contributor availability.

さらにトップページではTypeScript、Python、Java、C#、Goの5言語を前面に出しており、移行ガイドも5言語分の手順を用意しています。

このため、「どの言語が完全に使えるのか」という一点だけを見ると、ドキュメント全体の表現はまだ完全にはそろい切っていないように見えます。
少なくとも、自分がこのように判断した理由は、README、FAQ、サイト、移行ガイドで言語サポートの見せ方が少しずつ違っているためです。

少なくとも現時点では、TypeScriptとPythonが最も確度の高い選択肢と見てよさそうです。
一方で、Java、C#、Goについてはドキュメント上では見えているものの、採用前に最新の対応状況を再確認したほうが安全です。

ですので、実際に採用を考えるなら、使いたい言語のサンプル、最新リリース、issueの状況まで含めて確認したほうが安全です。

### 新規採用なら「なぜHCLではなくこれを選ぶか」を明確にしたい

CDKTFがarchiveされた現在、単に「HCLよりプログラミング言語が好きだから」というだけでは、採用理由としてはやや弱めです。

それでもcdk-terrainを選ぶ価値があるのは、たとえば以下のようなケースです。

- すでにCDKTF資産があり、思想や抽象化を引き継ぎたい
- 複雑なロジックや再利用パターンをコードで管理したい
- Terraform/OpenTofuのプロバイダやモジュール資産は使い続けたい

逆に、そこまで強い理由がないなら、HCLベースのTerraformに寄せるほうが保守しやすい場面も多そうです。

## いまの自分の見立て

2023年にCDKTFを触ったときは、「まだ荒削りだが、CDKの書き味でTerraformを扱えるのはおもしろい」という感想でした。
その後、本家がarchiveされたことで、CDKTFそのものを今から新規採用する選択はかなり難しくなりました。

一方で、cdk-terrainが出てきたことで、「Terraform/OpenTofuの世界で、プログラミング言語ベースのIaCを続けたい」という需要自体は消えていないことも見えてきました。

現時点での整理としては、こんな感じです。

| 選択肢 | 向いていそうなケース |
| ---- | ---- |
| AWS CDKへ移行 | もともとAWS CDKと密結合しており、AWS中心で構成を寄せたい |
| HCLベースのTerraformへ移行 | 標準的で長期保守しやすい構成を優先したい |
| cdk-terrainを検討 | CDKTF資産を引き継ぎたい、またはコードベースの抽象化を強く使いたい |
| 既存CDKTFを継続利用 | 短期的にすぐ移れないが、将来的な移行計画は持っておきたい |

## まとめ

CDKTFはarchiveされましたが、「Terraform/OpenTofuをプログラミング言語で扱いたい」というアプローチ自体が完全に終わったわけではありません。
その受け皿として、cdk-terrainが立ち上がっています。

ただ、自分の感覚としては「本家が終わったから次はcdk-terrain一択だ」と考える人は、そこまで多くないはずです。
そもそもCDKTF自体も、一部では使われていたものの、Terraformの標準的な書き方であるHCLを置き換えるほど積極的に広がったわけではない、というのが自分の認識です。

そのため、cdk-terrainは「CDKTFの代替として全員が自動的に移る先」というよりも、もともとCDKTFのアプローチに価値を感じていた人たちにとっての受け皿、と見るほうが自然だと思っています。
判断するとしたら、以下のような観点で見るのがよさそうです。

- 自分たちは本当にコードベースのIaCを必要としているか
- providerや言語サポートは足りているか
- HCLへ寄せたほうが運用コストを下げられないか

自分としては、既存CDKTFユーザーにとってcdk-terrainは十分に追う価値のあるプロジェクトだと見ています。
一方で、新規採用については、HCLベースのTerraformと比較しながら冷静に判断するのがよさそうです。
CDKとTerraform双方について知見がある人ならメリットを十分に享受できると考えますが、そうでなければどうしても学習コストが高くつきます。
また、いずれを使うにしても扱う対象のAPIについての理解も合わせて求められるため、IaCに対するハードルが新規採用ではどうしても跳ね上がってしまう部分が否めません。
今後どのようにcdk-terrainが進んでいくかは注視する価値がありますが、新規採用の判断ができる条件はまだ限定的かなと感じています。
少なくとも2026年3月時点では、業務で積極的に新規採用を勧められる段階かというと、まだ慎重に見たほうがよさそうです。
ただ、こうした選択肢がコミュニティ主導で続いていくこと自体には大きな意味があると思っており、個人的にはもっと盛り上がってくれることを願っています。
