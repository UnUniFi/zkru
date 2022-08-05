いままでのL2は、上のスライドにも書いてるけど、状態遷移式を多項式関数で表現できなければならないという制約により、汎用言語でL2コントラクト書けなかった、ゆえにL1とまったく異なる構造でバイナリ作ってたんやけど、zkru-bank-l2みたいなモジュールつくって、そこでbank sendのmsgを多項式関数のtrace tableに変換して保持してやると、L1の開発者体験でL2開発できる気がする(というかフォークするだけ)
これはcosmos sdkのmodulabilityのおかげ 


zkru-bank-l1 module
L1として機能するチェーンに組み込むモジュール。
zkpの検証者の役割を果たす。
単一のL2ノード運営マシンのアドレスをパラメータにもつ。
MsgDeposit
エンドユーザーがL1からL2に残高デポジット。
L2側はこのトランザクションを検知したらL2の状態に反映。
MsgWithdraw
エンドユーザーがL2からL1に残高引き出す。
L2側はこのトランザクションを検知したらL2の状態に反映。
MsgRollup
L2ノード運営マシンが、定期的に、状態遷移と、L2のマークルツリーに関するゼロ知識証明を提出。
これの証明が不正だったらL2ノード運営マシンのアドレスからslash。 （編集済み） 

zkru-bank-l2 module
L2として機能するチェーンに組み込むモジュール。
zkru-bank-l1を組み込んだチェーンとペアにしてもいいし、Ethereumとペアにしてもいい（その場合はzkru-bank-l1に相当するコントラクトをEthereumにデプロイすることになる）
MsgSend
bankのMsgSendと同等機能。
ただしKeeper内部でzkpのための状態遷移に関するtrace table作成を行い、証明に備える。
MsgMultiSend
bankのMsgMultiSendと同等機能。
ただしKeeper内部でzkpのための状態遷移に関するtrace table作成を行い、証明に備える。
L2ノード運営マシンは、ブロックチェーン用バイナリとは別のrelayerみたいなバイナリで定期的にL1にproveしていく。


理解したこと
zkru-bank-l1は基本的にsolidityで実装し、ethereumにデプロイするモジュールだが、goで実装して別のcosmosチェーンをL1として使うこともできる。
zkru-bank-l2はMsgSendを証明付きで保持する

keeper.SendCoinsは何をするのか
sdk.Content -> sdk.Content 


txのサインをどこで検証しているのかを調べる。
https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/bank/types/msgs.go


設計について
Cosmos SDKはなるべく変更しないように実装する

