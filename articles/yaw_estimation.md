---
title: "吊り荷Yaw制御システムの開発 #03：カメラ画像からYaw角を推定する"
emoji: "🏗️"
type: "tech"
topics: ["python", "pytorch", "画像認識", "simulation"]
published: false
---

## はじめに

[前回](https://zenn.dev/rindajones/articles/payload-yaw-control-02_2)までに、強化学習環境を用いて制御モデルを構築しました。

今回は、制御に必要な吊り荷のYaw角をカメラ画像から推定します。

Blenderで生成した画像を使って物体検出モデルを学習し、検出した特徴点から吊り荷のYaw角を算出します。3種類の吊り荷について推定を行い、制御中の画像を含めて推定精度を評価します。

![](/images/payload-yaw-control/development-flow.png)

本シリーズでは、以下の流れで開発を紹介しています。

[#01 制御環境を構築する](https://zenn.dev/rindajones/articles/a8b6189171848f)
[#02-1 強化学習の環境を構築する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_1)
[#02-2 強化学習による制御モデルを学習する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_2)
**#03 カメラ画像からYaw角を推定する ← 今回**
#04 Raspberry Piで画像ベース制御を実証する

## 開発環境

Yaw角推定モデルの学習および評価には、以下の環境を使用しました。

| 項目 | 環境 |
| --- | --- |
| OS | Ubuntu 24.04 LTS |
| 開発言語 | Python 3.11 |
| 機械学習フレームワーク | PyTorch |
| 物体検出モデル | SSDLite320 MobileNetV3 |
| 3Dシミュレーション | Blender 5.1.2 |

## Yaw角推定モデルの学習から評価まで

今回構築した、Yaw角推定モデルの学習から評価までの全体構成を以下に示します。

![](/images/payload-yaw-control/03/yaw_model_flow_jp.png)

制御あり・制御なしのシミュレーション結果から学習データを作成し、物体検出モデルを学習します。学習済みモデルによる検出結果から吊り荷のYaw角を算出し、推定精度を評価します。

**ここで推定したYaw角は、次のステップで強化学習によるYaw制御の入力として使用します。**

## 学習データの作成

物体検出モデルの学習には、Blenderで生成したシミュレーション画像を使用します。

画像生成には、前回作成した制御ありの推論結果と、スラスターを動作させない制御なしのシミュレーション結果を使用します。

- `auto`：SACによる制御あり
- `baseline`：制御なし

それぞれのシミュレーション結果はCSVファイルとして保存されています。

### Blenderによるシミュレーションの再現

CSVに保存された吊り荷やスラスター機体の位置・姿勢をBlenderへ読み込み、シミュレーション結果をアニメーションとして再現します。

主に以下のスクリプトを使用します。

- `rope_attach.py`：ロープの接続と追従
- `move.py`：CSVから位置・姿勢を読み込み、Blender上のオブジェクトを動かす
- `data_gen.py`：各フレームから学習画像とラベルを生成する

`move.py` では、使用する吊り荷と制御結果CSVを指定します。

（コード例）

続いて `data_gen.py` で、生成する吊り荷と外乱条件、出力先を指定します。

（コード例）

Blender上でこれらのスクリプトを実行し、各条件の画像とラベルを作成します。

```
datasets/raw/
├── baseline
│   ├── hsteel
│   ├── plate
│   └── truss
└── auto
    ├── hsteel
    ├── plate
    └── truss
```
各吊り荷の下には X0Y0、X1Y0、X1Y1 の3条件を配置し、それぞれに学習画像と labels.csv を保存します。
```
auto/
└── truss/
    └── X1Y1/
        ├── images/
        └── labels.csv
```
labels.csv には画像生成時の物体位置などの教師情報を保存します。また、シミュレーション上のYaw角は後の推定精度評価でGround Truthとして使用します。

## 物体検出モデルの学習

...

## Yaw角の推定と精度評価

...