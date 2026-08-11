---
title: "吊り荷Yaw制御システムの開発 #03：カメラ画像からYaw角を推定する"
emoji: "🏗️"
type: "tech"
topics: ["python", "pytorch", "画像認識", "simulation"]
published: false
---

## はじめに

[前回](https://zenn.dev/rindajones/articles/payload-yaw-control-02_2)までに、強化学習環境を用いて制御モデルを構築しました。

今回のYaw角推定では、カメラ画像からYaw角を直接推定するのではなく、まず物体検出モデルで吊り荷のフックを検出します。検出したフックの位置関係から画像処理によって吊り荷の向きを求め、Yaw角を算出します。

つまり、Yaw角推定は次の2段階で行います。

1. **物体検出：画像から吊り荷のフックを検出する**
2. **画像処理：検出したフックの位置関係からYaw角を算出する**

Blenderで生成した画像を使ってフック検出モデルを学習し、3種類の吊り荷について、制御中の画像を含めてYaw角の推定精度を評価します。

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
なお、`rope_attach.py` と `move.py` については、[#01のBlender Pythonスクリプト](https://zenn.dev/rindajones/articles/a8b6189171848f#blender-pythonスクリプト)で説明しています。

- `rope_attach.py`：ロープの接続と追従
- `move.py`：CSVから位置・姿勢を読み込み、Blender上のオブジェクトを動かす

### 学習画像とラベルの生成

 `data_gen.py` で、生成する吊り荷と外乱条件、出力先を指定します。

```python
OBJ_TYPE = "truss"   # "hsteel", "plate", "truss"
XY = "X1Y1"

OUT_DIR_A = OUT_DIR / "baseline"   # "baseline" or "auto"

OUT_DIR_B = OUT_DIR_A / OBJ_TYPE
OUT_DIR_C = OUT_DIR_B / XY

IMG_DIR = OUT_DIR_C / "images"
LABEL_CSV = OUT_DIR_C / "labels.csv"
```
OBJ_TYPE で吊り荷、XY で外乱条件、OUT_DIR_A で制御あり／制御なしを指定します。生成した画像は images/、教師情報は labels.csv に保存します。

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

`labels.csv` には、画像生成時の検出対象の位置情報とシミュレーション上のYaw角などを保存します。
検出対象の位置情報は物体検出モデルの学習に使用し、Yaw角は後の推定精度評価でGround Truthとして使用します。

### カメラ画像と検出対象

カメラはスラスター機体の下部に設置し、吊り荷を上方から撮影します。

![](/images/payload-yaw-control/03/camera.png)


カメラ画像では、吊り荷を支持するフックを物体検出の対象とします。

![](/images/payload-yaw-control/03/payloads.png)

H鋼では2点、平板とトラスでは4点のフックを検出します。
これらのフックの位置関係を、後段のYaw角算出に使用します。

## 物体検出モデルの学習

...

## Yaw角の推定と精度評価

...