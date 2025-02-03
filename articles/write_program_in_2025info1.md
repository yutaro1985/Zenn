---
title: "2025年の共通テスト情報1のプログラムを制約を厳しくして作ってみた"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["共通テスト","プログラム","情報1"]
published: true
---

**※2025年の共通テスト情報1の問題、特に第3問について詳細に記述しているため、自分で解いてみる前に情報を入れたくないという方は一旦読むのをやめてください。**

競技プログラミングにちょっと触れたことがある人にとってはさほど難しい話ではないと思います。

## まえがき

2025年度の共通テスト[^1]から情報1が科目に加わりました。
せっかくなのでと思って解いてみたところ現役ITエンジニアとしてはそれほど難しいものではなく、基本的には問題文中に書いてあることが理解できればほぼ回答できるようになっている感じでした。
ほんのちょっとだけ知識問があるぐらいで、あとは大体問題文に書いてあるとおりに進めれば答えが出るといった感じです。
難易度としては、基本情報よりだいぶ簡単なぐらいでしょうか。
ただ、大問4つで60分ということでのんびり時間を使っていると時間内に解くのが難しい可能性はあります。
個人的には第1問で問題文ちゃんと読まなくて間違えたものと、第4問のグラフをきちんと読み解けなくて間違えた2ミスで95点でした。

なお、問題および解答と配点は[こちら](https://mainichi.jp/exam/kyotsu-2025/q/?sub=IN1)で確認しました。

## 第3問のプログラムについて

さて、この記事の本題ですが、第3問にプログラムを作成する問題があります。
プログラム自体は非常に簡素なもので、おそらくFizzBuzzを書ける程度にプログラムに理解があれば十分回答可能なものと思います。

問題自体は流れに沿ってプログラムを作っているというものになっていますが、まとめると以下のような条件のプログラムを作成することになります。

- ある工芸部の生徒が、工芸品の制作の担当割り当てを行う
  - 部員は3人
    - 部員が増えた場合でも対応できるようにする
  - 担当の手が空き次第次の工芸品の制作を開始するように割り当てる
- 9つの工芸品があり、それぞれに制作日数が設定されている
  - 問題文中の設定では1〜4日のいずれかの日数になっている
- 最終的に、工芸品ごとに誰が何日目から何日目までに制作を担当するかを出力する

問題文中の制約では、部員が3人、工芸品が9つしかないので計算量的な制約はまったく気にする必要がありません。
また、同一工芸品における作業分担が発生しないのと、最短スケジュールで行えるようにするのではなく、あくまで順繰りに手が空いたら次の工芸品を作成するという方法のため、最適化を行う必要もありません。

問題中にある疑似プログラムを元に書いたプログラムは以下のようになります。
なお、言語は自分の趣味でGolangにしています。
変数名や数値の入れ方は問題の記載方法に従っています。
また、問題文中では1-indexedで記載されていますが、大抵の言語では配列は0-indexedであるため、こちらも0-indexedで記載しています。

やっていることは非常にシンプルで、それぞれの工芸品について部員の空き日が一番短い部員を全探索し、その部員に工芸品を割り当てるということを繰り返しています。

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	// 各工芸品の制作日数
	Nissu := []int{4, 1, 3, 1, 3, 4, 2, 4, 3}
	kougeihinsu := 9
	// 各部員があと何日で空くかの管理
	Akibi := []int{1, 1, 1}
	buinsu := 3
	for kougeihin := 0; kougeihin < kougeihinsu; kougeihin++ {
		tantou := 0
		// 今一番空き日が短い部員を全探索して特定する
		for buin := 1; buin < buinsu; buin++ {
			if Akibi[buin] < Akibi[tantou] {
				tantou = buin
			}
		}
                // 工芸品をどの部員が何日目から何日目まで制作するかを標準出力する
		fmt.Println("工芸品" + strconv.Itoa(kougeihin+1) + " … " + "部員" + strconv.Itoa(tantou+1) + " : " + strconv.Itoa(Akibi[tantou]) + "日目〜" + strconv.Itoa(Akibi[tantou]+Nissu[kougeihin]-1) + "日目")
                // 工芸品を担当する部員の空き日に担当する工芸品の日数を加算する
		Akibi[tantou] = Akibi[tantou] + Nissu[kougeihin]
	}
}
```

以下で実行結果を確認できます
https://goplay.tools/snippet/ZSsvbSqxxi-

これを実行すると以下のように出力されます。

```
工芸品1 … 部員1 : 1日目〜4日目
工芸品2 … 部員2 : 1日目〜1日目
工芸品3 … 部員3 : 1日目〜3日目
工芸品4 … 部員2 : 2日目〜2日目
工芸品5 … 部員2 : 3日目〜5日目
工芸品6 … 部員3 : 4日目〜7日目
工芸品7 … 部員1 : 5日目〜6日目
工芸品8 … 部員2 : 6日目〜9日目
工芸品9 … 部員1 : 7日目〜9日目
```

問題と一致していることが確認できるはずです。

## 計算量の制約を考える

前述の通り、この問題文では工芸品の数も部員の数も一桁しかないため、計算量的な制約はまったく気にする必要がありません。
もちろん現実世界では人間が処理可能な日数で現実的な人数が処理することになるためこれで特に問題はありません。

このプログラムはforを二重ループにしています。
forは工芸品と部員でループをさせているため、この2つの数がめちゃくちゃ増えた場合には計算量が爆発します。
AtCoderなどをやったことがある方はピンと来るかもしれません。

では、制約を厳しくするために以下のように条件を変更してみましょう。

- 工芸品数を $N$、部員数を $M$ とする。
- $1 <= N <= 10^5$
- $1 <= M <= 10^5$
- 工芸品$N$の制作日数が工芸品ごとに与えられる
  - $1 <= A_i <= 10^5$
- 入力値は以下の要領で与えられる

$N\;M$
$A_1\;A_2\;\dots\;A_N$

上記のプログラムの計算量は$O(NM)$となります。
この制約があると、$N*M$が最大$10^{10}$となり、計算にとても時間がかかってしまいます。
※AtCoderなどではだいたい$10^8$以上の計算量になるとTLE（Time Limit Exceeded）になってしまうと思ってよいです。

工芸品を、作成が終わった部員から順繰りに作成していくという方法を変えないという前提とすると、今一番空き日が短い部員を探すところを効率的に行いたいです。

これはPriority Queue（優先度付きキュー）を使うと効率的に行うことができます。
参考
https://ja.wikipedia.org/wiki/%E5%84%AA%E5%85%88%E5%BA%A6%E4%BB%98%E3%81%8D%E3%82%AD%E3%83%A5%E3%83%BC
https://ufcpp.net/study/algorithm/col_heap.html
※他にもここが参考になるよというサイトがあれば追記するので教えてくれると助かります！

まずキューと呼ばれるデータ構造についてですが、FIFO（先入れ先出し）でデータを格納するデータ構造です。
Priority Queueは格納したデータに優先度となる何かしらを設定し、その優先度が高いものから取り出すデータ構造です。
普通のキューは先に入れたものを先に出すだけなのでデータの追加も取り出しもO(1)（ループが発生せず、1回の計算で終わる）で行えます。
Priority Queueは優先度の計算を行いますが、追加も取り出しも$O(\log N)$で行えます。
これにより、プログラムの計算量を$O(NlogM)$に抑えることができます。
ちなみに$O(\log N)$がどれぐらいかというと、このように書いた場合logの底は2と解釈するのがほとんどです。
つまり、$N=10^5$の場合でも、$logN$は17程度になるので計算量的にはわずかなものになります。

では、上記を踏まえてプログラムを修正してみます。

Golangでは標準ライブラリにPriority Queueがないので実装する必要がありますが、[heap/containerのドキュメントにサンプル実装がある](https://pkg.go.dev/container/heap#example-package-PriorityQueue)のでそれを流用すればよいです。
自分はそれを状況に合わせて修正して使っています。

言語によっては標準ライブラリにPriority Queueがあるのでそれを使えばよさそうです。

また、問題ではわかりやすくするためにわざわざ工芸品の数や制作日数部員数を固定値で記述していますが、そちらは入力値から取得します
自分がAtCoderをやっていた頃にGolangで標準入力を取得するのにはbufioを使っていて関数化しているのでそちらを使わせてもらいます。（実際にはもっと多いのですが、使わないものはここでは省いています）

```go
package main

import (
	"bufio"
	"container/heap"
	"fmt"
	"os"
	"strconv"
)

const (
	initialBufSize = 1e4
	maxBufSize     = 1e6 + 7
)

var buf []byte = make([]byte, initialBufSize)
var sc = bufio.NewScanner(os.Stdin)

func init() {
	sc.Split(bufio.ScanWords)
	sc.Buffer(buf, maxBufSize)
}

func main() {
	// 入力値の取得
	N, M := nextInt(), nextInt()
	Nissu := makeInts(N)
	// 各部員があと何日で空くかの管理
	PQ := InitPQ()
	// 最初は全員1日
	for i := 0; i < M; i++ {
		PQ.HPush(Item{i, 1, 0})
	}

	for i := 0; i < N; i++ {
		// Priority Queueから一番空き日数が短い部員を取得
		tantou := PQ.HPop()
		Akibi := tantou.priority

		fmt.Println("工芸品" + strconv.Itoa(i+1) + " … " + "部員" + strconv.Itoa(tantou.buin+1) + " : " + strconv.Itoa(Akibi) + "日目〜" + strconv.Itoa(Akibi+Nissu[i]-1) + "日目")

		PQ.HPush(Item{tantou.buin, Akibi + Nissu[i], 0})
	}
}

type Item struct {
	buin, priority, index int
}

type PQ []*Item

func InitPQ() PQ {
	pq := make(PQ, 0)
	heap.Init(&pq)
	return pq
}

func (pq PQ) Len() int {
	return len(pq)
}

// 比較して並び替えるロジックの部分。空き日数をpriorityに代入。部員の番号をもたせることによって部員を判別する。
// 一応、優先度が同じ場合は部員番号が若い方を優先するようにしている
func (pq PQ) Less(i, j int) bool {
	if pq[i].priority == pq[j].priority {
		return pq[i].buin < pq[j].buin
	} else {
		return pq[i].priority < pq[j].priority
	}
}

func (pq PQ) Swap(i, j int) {
	pq[i], pq[j] = pq[j], pq[i]
	pq[i].index = i
	pq[j].index = j
}

func (pq *PQ) Push(x interface{}) {
	n := len(*pq)
	item := x.(*Item)
	item.index = n
	*pq = append(*pq, item)
}

func (pq *PQ) HPush(x Item) {
	heap.Push(pq, &x)
}

func (pq *PQ) Pop() interface{} {
	old := *pq
	n := len(old)
	item := old[n-1]
	item.index = -1
	*pq = old[0 : n-1]
	return item
}

func (pq *PQ) HPop() *Item {
	item := heap.Pop(pq).(*Item)
	return item
}

func nextInt() int {
	sc.Scan()
	i, e := strconv.Atoi(sc.Text())
	if e != nil {
		panic(e)
	}
	return i
}

func makeInts(N int) []int {
	res := make([]int, N)
	for i := range res {
		res[i] = nextInt()
	}
	return res
}

```

これで、先の問題と同じ入力値なら同じ答えが出力されることと、適当に生成した大きな工芸品数、部員数の入力値でも時間内に計算が終わることを確認しました。

## さらなる最適化はどうか？

そもそも制作が終わった人から次の工芸品を順繰りに作成するというのが最適なやり方なのか、というところはちょっと自分には評価しきれませんでした。
感覚的にはそれで十分そうですが、もしかしたらもっと良いやり方があるかもしれません。
詳しい方がいたらぜひ補足をお願いします（丸投げ）

## まとめ

情報1のプログラム問題がなんとなくAtCoderのABCっぽい面影があるなと思ってついやってみたくなりました。
もっと応用的にできる方がいたらぜひやってみて解説していただけると嬉しいです。

[^1]: 我々の世代ではセンター試験、それよりももっと前の世代では共通一次と呼ばれていました。
