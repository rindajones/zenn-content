---
title: "吊り荷Yaw制御システムの開発 #03：カメラ画像からYaw角を推定する"
emoji: "🏗️"
type: "tech"
topics: ["python", "pytorch", "画像認識", "simulation"]
published: true
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
[#04 Raspberry Piで画像ベース制御を実証する](https://zenn.dev/rindajones/articles/raspberry_pi)
[#05 可変目標Yaw制御を目指す](https://zenn.dev/rindajones/articles/variable-target-yaw-control/)
[#06-1 可変目標Yaw制御の環境を構築する](https://zenn.dev/rindajones/articles/variable-target-env)
#06-2 可変目標Yaw制御モデルを学習する
#07 3種類の吊り荷で可変目標Yaw制御を試す  
#08 可変目標Yaw制御の結果と課題


## 開発環境

Yaw角推定の学習および評価には、以下の環境を使用しました。

| 項目 | 環境 |
| --- | --- |
| OS | Ubuntu 24.04 LTS |
| 開発言語 | Python 3.11 |
| 機械学習フレームワーク | PyTorch |
| 物体検出モデル | SSDLite320 MobileNetV3 |
| 3Dシミュレーション | Blender 5.1.2 |

## Yaw角推定から評価まで

今回構築した、Yaw角推定から評価までの全体構成を以下に示します。

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

主に以下のスクリプトを使用します。なお、`rope_attach.py` と `move.py` については、[#01のBlender Pythonスクリプト](https://zenn.dev/rindajones/articles/a8b6189171848f#blender-pythonスクリプト)で説明しています。

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
`OBJ_TYPE` で吊り荷、`XY` で外乱条件、`OUT_DIR_A` で制御あり／制御なしを指定します。生成した画像は `images/`、教師情報は `labels.csv` に保存します。

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
各吊り荷の下には `X0Y0`、`X1Y0`、`X1Y1` の3条件を配置し、それぞれに学習画像と `labels.csv` を保存します。
```
auto/
└── truss/
    └── X1Y1/
        ├── images/
        └── labels.csv
```

`labels.csv` には、画像生成時の検出対象の位置情報とシミュレーション上のYaw角などを保存します。検出対象の位置情報は物体検出モデルの学習に使用し、Yaw角は後の推定精度評価で Ground Truth として使用します。

### カメラ画像と検出対象

カメラはスラスター機体の下部に設置し、吊り荷を上方から撮影します。

![](/images/payload-yaw-control/03/camera.png)


カメラ画像では、吊り荷に取り付けた色分けされたフックを物体検出の対象とします。

![](/images/payload-yaw-control/03/payloads.png)

H鋼では2点、平板とトラスでは4点のフックを検出します。これらのフックの位置関係を、後段のYaw角算出に使用します。


## 物体検出モデルの学習

### 物体検出モデル

最終的にはRaspberry Pi上で画像認識とYaw制御をリアルタイムに実行するため、物体検出モデルには比較的小規模な **SSDLite320 MobileNetV3** を使用します。学習後のモデルは `ONNX` 形式へ変換し、Raspberry Pi上では `ONNX Runtime` を使用して推論します。

今回は事前学習済みの重みを使用せず、Blenderで生成したシミュレーション画像のみを使用してモデルを学習します。

### 検出クラス

検出対象は、吊り荷に取り付けた色分けされたフックです。

H鋼では2点、平板とトラスでは4点のフックを使用します。吊り荷の形状やフックの種類を区別するため、検出対象を合計 `8クラス` として定義しました。

`SSDLite320 MobileNetV3` では、これら8クラスに `background` を加えた `9クラス` としてモデルを構築します。

```python
model = ssdlite320_mobilenet_v3_large(
    weights=None,
    weights_backbone=None,
    num_classes=9,
)
```

### 学習データ

学習には、前節で生成した `baseline` と `auto` の画像を使用します。

3種類の吊り荷と3種類の外乱条件を組み合わせ、すべてのデータセットを結合して1つの物体検出モデルを学習します。

```python
for data_type in ["baseline", "auto"]:
    for obj_type in ["hsteel", "plate", "truss"]:
        for cond in ["X0Y0", "X1Y0", "X1Y1"]:
            ds = PayloadDataset(
                root=DATASET + f"{data_type}",
                obj_types=(obj_type,),
                conditions=(cond,),
            )
            train_parts.append(ds)

train_set = ConcatDataset(train_parts)
```

これにより、

- 制御あり／制御なし：2条件
- 吊り荷：3種類
- 外乱：3条件

を組み合わせた18条件の画像を学習に使用します。

### 学習条件

主な学習条件を以下に示します。

| 項目 | 設定 |
| --- | --- |
| モデル | SSDLite320 MobileNetV3 |
| 検出クラス | 8クラス + background |
| Batch size | 16 |
| Epoch | 150 |
| Optimizer | AdamW |
| Learning rate | 5e-4 |
| Weight decay | 1e-4 |

`150 epochs` まで学習し、学習 `loss` が最小となったモデルを保存します。

### フックの検出結果

学習した物体検出モデルを使用して、H鋼、平板、トラスのフックを検出した例を以下に示します。

![](/images/payload-yaw-control/03/hook_detection.png)

画像中の `GT` はGround Truth、`TP` は正しく検出されたフック、`FN` は検出できなかったフックを示しています。

すべてのフレームですべてのフックを検出できるとは限らず、吊り荷の姿勢などによって一部のフックが検出できない場合があります。

次に、検出したフックの位置関係から吊り荷の向きを求め、Yaw角を算出します。


## Yaw角の推定と精度評価

検出したフックの位置関係から吊り荷の代表方向を求め、時系列の情報も使用して最終的なYaw角を算出します。

今回の方法では、吊り荷の形状とフックの配置が既知であり、画像上の吊り荷形状から代表軸を定義できることを前提としています。本検証では、H鋼、平板、トラスのように代表方向を定義しやすい吊り荷を対象としています。

Yaw角の算出では、まず検出したフックから利用可能な2点を選択し、その2点を結ぶ方向を基準とします。この基準方向に対して複数の走査線を設定し、画像中の吊り荷領域との交点から中心位置を求めます。得られた中心点に直線をフィッティングし、吊り荷の代表軸としてYaw角を算出します。

フックの組み合わせは吊り荷の形状に応じて選択しますが、H鋼、平板、トラスでYaw角を求める基本的な処理は共通です。

以下は、Yaw角を算出する過程を可視化した例です。フック間の方向を基準として `31本` の走査線を設定し、吊り荷領域から得られた中心点に直線をフィッティングしています。

![](/images/payload-yaw-control/03/yaw_estimations.png)

フックを検出できないフレームが含まれる場合もあるため、**すべてのフレームですべてのフックが検出されることを前提とはしていません**。検出が一時的に欠落した場合は、直前までの検出結果を利用してYaw角を補正します。


### 推定精度の評価

物体検出から求めたYaw角は、カメラを搭載したスラスター機体に対する吊り荷の相対Yaw角です。

評価では、推定した相対Yaw角にスラスター機体のYaw角を加えて世界座標系のYaw角へ変換し、Blenderのシミュレーションから取得した吊り荷のYaw角を Ground Truth として比較します。

```python
yaw_world_est_deg = wrap_deg(thr_yaw + yaw_est)
yaw_world_gt_deg = wrap_deg(payload_yaw)

yaw_err_deg = abs(
    wrap_deg(yaw_world_est_deg - yaw_world_gt_deg)
)
```

評価時の検出スコア閾値は `0.5` としました。また、一時的にフックを検出できない場合は最大 `5フレーム` まで直前の検出情報を保持し、Yaw角が前回値から `10°` を超えて急変する場合は推定値を除外します。

評価には、学習データと同様に以下の条件を使用しました。

- `baseline`：制御なし
- `auto`：SACによる制御あり
- 吊り荷：H鋼、平板、トラス
- 外乱：`X0Y0`、`X1Y0`、`X1Y1`

各条件 `380枚` の画像についてYaw角を推定し、`MAE`、`RMSE`、`最大誤差` を算出します。

![](/images/payload-yaw-control/03/yaw_eval_summary.png)


18条件すべてで `MAE` は `1°未満` となりました。一方、フレーム単位の最大誤差は `auto / Truss / X1Y1` で **2.68°** となりました。


### 制御あり・制御なしの比較

今回のYaw角推定は、最終的に強化学習によるYaw制御中に使用します。

そのため、スラスターを動作させない `baseline` と、SACによって吊り荷を制御している `auto` で推定精度に差が生じるかを比較しました。

![](/images/payload-yaw-control/03/compare_chart.png)

全条件の `MAE` を比較すると、`baseline` と `auto` の間に大きな差は見られませんでした。

- `baseline`：平均MAE 0.32°、最大MAE 0.77°
- `auto`：平均MAE 0.34°、最大MAE 0.69°

制御によって吊り荷の姿勢が変化する状態でも、Yaw角の推定精度は概ね維持されています。


### 最大誤差の原因

全条件で最大となった誤差は、`auto / Truss / X1Y1` の **2.68°** でした。

最大誤差が `2.0°以上` となったフレームを確認すると、いずれもTrussのフックの一部を検出できていませんでした。

![](/images/payload-yaw-control/03/errors.png)

上図では、Truss右側のフックが画像端に近づき、一部が画角外となっています。

フックを検出できないフレームでは、直前までの検出結果を利用する時系列処理によってYaw角を補正します。今回の評価では、このような検出欠落を含む場合でも、最大誤差は `2.68°` でした。

以上から、シミュレーション画像では制御あり・制御なしのいずれについても、Yaw制御の入力として使用できる推定精度が得られました。

次回は、このYaw角推定をRaspberry Pi上で実行し、強化学習モデルと組み合わせたリアルタイムYaw制御を検証します。


## シリーズ構成

本シリーズでは、吊り荷Yaw制御システムの開発過程を段階的に紹介します。

- [#01 制御環境を構築する](https://zenn.dev/rindajones/articles/a8b6189171848f)
- [#02-1 強化学習の環境を構築する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_1)
- [#02-2 強化学習による制御モデルを学習する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_2)
- **#03 カメラ画像からYaw角を推定する ← 今回**
- [#04 Raspberry Piで画像ベース制御を実証する](https://zenn.dev/rindajones/articles/raspberry_pi)
- [#05 可変目標Yaw制御を目指す](https://zenn.dev/rindajones/articles/variable-target-yaw-control)
- [#06-1 可変目標Yaw制御の環境を構築する](https://zenn.dev/rindajones/articles/variable-target-env)
- #06-2 可変目標Yaw制御モデルを学習する
- #07 3種類の吊り荷で可変目標Yaw制御を試す
- #08 可変目標Yaw制御の結果と課題


## 関連リンク

プロジェクト全体は GitHub Pages で公開しています。
@[card](https://rindajones.github.io/payload-yaw-control/ja/)