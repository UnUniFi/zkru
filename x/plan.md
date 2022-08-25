<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>
<script type="text/x-mathjax-config">
    MathJax.Hub.Config({ tex2jax: {inlineMath: [['$', '$']]}, messageStyle: "none" });
</script>

# ZK-Rollupの設計

## TL;DR
Cosmos SDK for zkruにおいて決定すべきこと

- DAをどうするか(データをL1に置くかcosmos側に置くか)
- Merkle treeをどうするか
- cosmos側のコンセンサスをどう扱うか

## ZK-Rollupの構造
普通のL1チェーンにおいて、ブロック提案者は次の手順でトランザクションtxを処理する

1. txのフォーマットが正しく、署名が正しくなされていることを検証する。
2. txで要請された処理（送金など）を行いマークル木の状態を更新する。

ブロックに含まれる全てのtxを処理し、最終的に得られたマークル木のルートをブロックヘッダーに含め、処理を行ったtxとともにブロードキャストする。

この過程は次の式で表すことができる。

$$
mkr_{i+1} \leftarrow f([tx_1, ..., tx_n], mk_i)
$$

$mk_i$は$i$番目のブロックのマークル木、$mkr_{i+1}$は$i+1$番目のブロックのマークルルート、$[tx_1, ..., tx_n]$はブロックに含まれるトランザクションである。

ZK-Rollupにおいては、これらの処理を担当するaggregatorは、$f$のZKPを提出することで不正が無いことを証明する。

aggregatorはproofを作成し、L1コントラクトはそのproofを検証し、マークルルートの遷移を受け入れる。

$$
\begin{align}
proof \leftarrow Prove(f, [tx_1, ..., tx_n], mk_i)\\
0/1 \leftarrow Verify(proof, [tx_1, ..., tx_n],mkr_i, mkr_{i+1})
\end{align}
$$

証明の作成にはマークル木の全体が必要だが、検証はマークルルートのみで良い。

## Data availability

上記の手順において、L1コントラクトでVerify関数には[tx_1, ..., tx_n]が引数として入っている。
これは省略することもできるが、そのときにData availability問題が発生する可能性がある。

Data availability問題とは、悪意のあるaggregatorが、非公開のtxを用いてproofを作成し、マークルルートの遷移を行うことである。他の人は現在のマークル木を知らない(マークルルートしか知らない)ので、proofを作成することができない。従ってトークンがロックされてしまう。

aggregatorは他人のトークンを奪うことはできないが、身代金を請求するなどインセンティブはあるので、これは防がなければいけない。

通常は[tx_1, ..., tx_n]を検証可能性を保ったままなるべく圧縮し、L１コントラクトのcalldataなどの比較的安価な保存する。

しかし、やはりガス代は掛かってしまうので平均的なzk rollupでは一回の検証当たり$500程度のガス代が必要であると言われている。

その解決案として、[tx_1, ..., tx_n]を含めず、代わりにDA用のチェーンを作り、そこにデータを保存するという方法も有力である(zkSyncのzkPorterなど）。

zkru for cosmos SDKでは、コンセンサスエンジンが動いているので、そこでコンセンサスの取れたブロックのDAは保証されていると仮定して、Verify関数にはトランザクションを含めないというのが良いと思われる。デメリットとしては、コンセンサスが破綻した際にDAが失われること(トークンが永遠にロックされる)。

## txの検証
zkruにおいて、txの署名検証のzkpを作成することは必須である。何故なら、もし通常のcosmosチェーンのように、コンセンサスを用いて署名検証の正当性を保証するのであれば、コンセンサスが破綻した場合に攻撃者が不正なproofを作成することが可能になるからである。

cosmos sdkは次の手順でtxの署名を作成する。

1. 送金額や送金先、送金元のアドレスなどの情報をprotobufによってシリアライズする。
2. シリアライズ化したデータに署名(ECDSA of secp256k1)を行う。

1,2ともにplonky2を使えば難しくないと考えている。しかしながら、もしこれができない場合は、ユーザーがtxを出す際、memo欄にzkフレンドリーなシリアライズ化を行ったzkフレンドリーな署名を添付してもらい、proofの作成にはそちらを利用するという方法が考えられる。ただ、このmemo方式の場合、署名が2つ存在することになり、そのうちひとつは無視されるのであまりきれいな実装にならないという問題はある。またkeplerなどの既存のウォレットが利用できないという欠点もある。


## Merkle tree
zkruでは、cosmos sdkで実装されているmerkle treeは利用することができず、zkフレンドリーなものを利用する必要がある(zkpで効率よく計算できるハッシュ関数を利用する必要がある)。

この問題には大きく分けて2通り解決策が考えられる。

1. cosmos sdkのmerkle treeをzkフレンドリーなものに置換する。
2. cosmos sdkのmerkle treeと、zkru用のmerkle treeを独立に共存させる。

1の場合、実装はかなり大掛かりになる。merkle treeを利用する全てのモジュールに手を加える必要がある(例. PoSコンセンサスなど)。

2の場合、既存のmerkle treeの状態とzkru用のmerkle treeの状態は独立なので、コンセンサス・ガバナンス用のトークン(管理はcomos sdk側のmerkle tree)とzkru上で利用できるトークン(管理はzkru側のmerkle tree)の２種類にトークンが共存することになる。

IBCはcosmos sdk側のmerkle treeを利用するので、２の場合zkru用トークンのIBC転送はできなくなる。

1の場合はmerkle treeの規格が変わるため、ライトクライアントを再設計する必要がある可能性がある。またその場合、接続先のチェーンにもライトクライアントの新規格に対応してもらう必要がある。ただ、ZKPベースのライトクライアントを利用したIBCなどの実装がやりやすくなるメリットもある。

## コンセンサス
zkru for cosmos sdkは、通常のzkruと異なり、コンセンサスエンジンを持つ。このコンセンサスエンジンをどのように活用するのかという問題がある。

通常のzkruはコンセンサスが不要なので、L1へのコミットは誰でも好きなときに好きなtxをまとめてコミットできる。

一方でzkru for cosmos sdkには一定時間ごとにtxがブロックにまとめられ、コンセンサスが行われる。そのため、zkru for comos sdkには普通のzkruが持たない問題が生じる。それは、cosmos側でコンセンサスが取れていない(が有効な)ブロックをL1にコミットすることをどうやって防ぐかという問題である。L1側のzkp verifierはcosmos側でコンセンサスが取れたブロックか、ブロックとそのzkpだけでは判断することができない。

cosmos側とL1側で不整合が生じた場合に、cosmosの方を優先するのであれば、これはzkruの意味がない。

一方で、L1はcosmos側でコンセンサスが取れているか判断する術はない。

解決策としては、主に２つある。

- cosmos側とL1側で不整合が生じた際、投票などのメカニズムでL1との不整合を解決する方法。これは、何かしらのインセンティブ設計(スラッシュなど)が必要になる。欠点はzkruであるにも関わらずL1側のセキュリティを完全には継承できないこと。

- cosmosのコンセンサス情報もzkp化してL1に提出し、コンセンサスが取れたことを確認したものだけ受理する方法。欠点は実装の複雑化。