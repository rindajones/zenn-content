---
title: "吊り荷Yaw制御システムの開発 #04：Raspberry Piで画像ベース制御を実証する"
emoji: "🏗️"
type: "tech"
topics: ["raspberrypi", "onnx", "python", "強化学習", "robotics"]
published: true
---

## はじめに

[前回](https://zenn.dev/rindajones/articles/yaw_estimation)までに、強化学習による吊り荷Yaw制御モデルを構築し、カメラ画像から吊り荷のYaw角を推定できることを確認しました。

今回は、これらをRaspberry Pi上で組み合わせ、画像から推定したYaw角を制御モデルへ入力することで、画像ベースのリアルタイムYaw制御を行います。

本シリーズでは、以下の流れで開発を紹介しています。

[#01 制御環境を構築する](https://zenn.dev/rindajones/articles/a8b6189171848f)
[#02-1 強化学習の環境を構築する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_1)
[#02-2 強化学習による制御モデルを学習する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_2)
[#03 カメラ画像からYaw角を推定する](https://zenn.dev/rindajones/articles/yaw_estimation)
**#04 Raspberry Piで画像ベース制御を実証する ← 今回**
[#05 可変目標Yaw制御を目指す](https://zenn.dev/rindajones/articles/variable-target-yaw-control/)
[#06-1 可変目標Yaw制御の環境を構築する](https://zenn.dev/rindajones/articles/variable-target-env)
[#06-2 可変目標Yaw制御モデルを学習する](https://zenn.dev/rindajones/articles/variable-target-training)
#07 3種類の吊り荷で可変目標Yaw制御を試す  
#08 可変目標Yaw制御の結果と課題


## Yaw推定モデルのONNX変換

[前回](https://zenn.dev/rindajones/articles/yaw_estimation)は、カメラ画像からフックを検出し、その位置関係から吊り荷のYaw角を推定しました。

ここまではPyTorch上でモデルを実行していましたが、最終的な制御ではRaspberry Pi上でYaw推定と制御モデルの推論を実行します。そこで、まずYaw推定に使用するフック検出モデルをONNX形式へ変換します。

### なぜONNXへ変換するか

モデルの学習にはPyTorchとGPUを使用していますが、Raspberry Piでは学習は行わず、学習済みモデルによる推論だけを実行します。そのためRaspberry Pi側では、PyTorchの学習環境やGPUを必要とせず、CPU上で推論を実行できる構成としました。

今回は推論環境としてONNX Runtimeを使用し、PyTorchで学習したフック検出モデルをONNX形式へ変換します。これにより、**学習環境と実行環境を分離し、Raspberry Pi側には推論に必要な環境だけを構築**できます。

ただし、モデル形式を変換できることと、変換前と同じ推定精度が得られることは別です。そこでONNX変換後についても、**同じデータと評価方法を使用してYaw推定精度を確認**します。


### ONNX変換

フック検出には、PyTorchで学習した `SSDLite320 MobileNetV3` を使用しています。

学習済みモデルを読み込み、`torch.onnx.export()` を使用してONNX形式へ変換しました。

```python
model = ssdlite320_mobilenet_v3_large(
    weights=None,
    weights_backbone=None,
    num_classes=9,
)

ckpt = torch.load(PTH_PATH, map_location="cpu")
model.load_state_dict(ckpt["model"])
model.eval()

dummy = torch.randn(1, 3, 512, 512)

torch.onnx.export(
    model,
    dummy,
    ONNX_PATH,
    export_params=True,
    opset_version=17,
    do_constant_folding=True,
    input_names=["images"],
    output_names=["boxes", "labels", "scores"],
    dynamo=False,
)
```

変換したモデルは、Raspberry Piで使用するモデルとして保存します。

```text
runtime/raspberry_pi/models/yaw_detector.onnx
```

ONNXへの変換そのものは、特別な処理を必要とせず比較的簡単に行えました。

### 変換前後のYaw推定精度比較

ONNX変換によってYaw推定精度が変化していないかを確認します。

同じ評価データを使用して、PyTorch版とONNX版のYaw推定結果を比較しました。

| 吊り荷 | 外乱 | PyTorch MAE [°] | ONNX MAE [°] |
|---|---|---:|---:|
| HSteel | X0Y0 | 0.07 | 0.07 |
| HSteel | X1Y0 | 0.34 | 0.34 |
| HSteel | X1Y1 | 0.42 | 0.42 |
| Plate | X0Y0 | 0.20 | 0.20 |
| Plate | X1Y0 | 0.22 | 0.22 |
| Plate | X1Y1 | 0.24 | 0.24 |
| Truss | X0Y0 | 0.25 | 0.25 |
| Truss | X1Y0 | 0.29 | 0.29 |
| Truss | X1Y1 | 0.63 | 0.63 |

すべての条件で `MAE` は小数第2位まで一致し、今回使用したモデルでは、**ONNX変換によるYaw推定精度の低下は確認されませんでした**。

次に、ONNX版について、制御なし（`Baseline`）と制御あり（`Auto`）のYaw推定精度を比較します。

| 吊り荷 | 外乱 | Baseline MAE [°] | Auto MAE [°] | 差 [°] |
|---|---|---:|---:|---:|
| HSteel | X0Y0 | 0.05 | 0.07 | +0.02 |
| HSteel | X1Y0 | 0.24 | 0.34 | +0.10 |
| HSteel | X1Y1 | 0.39 | 0.42 | +0.03 |
| Plate | X0Y0 | 0.23 | 0.20 | -0.03 |
| Plate | X1Y0 | 0.25 | 0.22 | -0.03 |
| Plate | X1Y1 | 0.27 | 0.24 | -0.03 |
| Truss | X0Y0 | 0.20 | 0.25 | +0.05 |
| Truss | X1Y0 | 0.51 | 0.29 | -0.22 |
| Truss | X1Y1 | 0.65 | 0.63 | -0.02 |

すべての条件で `MAE`は `1°未満` となり、制御中（`Auto`）でも `Baseline` と同程度のYaw推定精度が得られました。

以上から、ONNX変換後もYaw推定精度が維持され、Raspberry Pi上での画像ベース制御に使用できることを確認しました。

## Raspberry Piでの実行構成

Yaw推定モデルをONNX形式へ変換できたので、次にRaspberry Pi上で画像ベース制御を実行する環境を構築します。

Raspberry Piでは、3Dシミュレーターから送られてくるカメラ画像を受信し、Yaw角の推定と制御モデルによる推論を行います。推論結果として得られたスラスター出力を制御環境へ返すことで、
画像を入力とした制御ループを構成します。

### Raspberry Pi環境

画像からのYaw推定と制御モデルの推論には、Raspberry Pi 5を使用しました。

| 項目 | 環境 |
|---|---|
| Raspberry Pi | Raspberry Pi 5 (4GB) |
| CPU | Quad-core Arm Cortex-A76 |
| OS | Debian GNU/Linux 12 (bookworm), 64-bit |
| 推論 | ONNX Runtime / CPU |

![](/images/payload-yaw-control/04/raspberry_pi.png)

Raspberry Pi側ではGPUなどのアクセラレータは使用せず、Yaw推定モデルと制御モデルの両方をCPU上で実行します。


### システム構成

Raspberry Piでは、画像からYaw角を推定し、その推定値を制御モデルへ入力してスラスター出力を決定します。

システム全体の構成を以下に示します。

![](/images/payload-yaw-control/04/system_flow_ja.png)

Raspberry Pi側では、主に以下の処理を実行します。

1. 3Dシミュレーターからカメラ画像を受信
2. ONNXモデルによるフック検出
3. 検出結果から吊り荷Yaw角を推定
4. 推定Yaw角から制御モデルの入力を生成
5. `Actor` によるスラスター出力の推論
6. `Action` をUbuntu側へ送信

Yaw推定と `Actor` の推論にはONNX Runtimeを使用します。

3Dシミュレーションと物理計算はUbuntu側で行い、画像からのYaw推定とスラスター出力の決定をRaspberry Pi側で行います。


## 制御モデルのONNX変換

Yaw推定モデルと同様に、制御モデルについてもRaspberry Pi上で推論するためONNX形式へ変換します。

強化学習にはSACを使用していますが、Raspberry Pi上で制御を実行する際に必要なのは、現在の状態からスラスター出力を推論する `Actor` です。そこで、学習済みSACモデルから `Actor` を取り出し、ONNX形式へ変換しました。

吊り荷ごとに3つの外乱条件（`X0Y0` / `X1Y0` / `X1Y1`）を用意しているため、ONNX変換後の `Actor` は合計9モデルになります。

:::message
この段階では、外乱条件ごとに個別のモデルを学習しています。
その後の開発では、複数の外乱条件を一つのモデルで扱う方法へ発展させています。
:::

```text
runtime/raspberry_pi/models/
├── actor_hsteel_X0Y0_amp03.onnx
├── actor_hsteel_X1Y0_amp03.onnx
├── actor_hsteel_X1Y1_amp03.onnx
├── actor_plate_X0Y0_amp03.onnx
├── actor_plate_X1Y0_amp03.onnx
├── actor_plate_X1Y1_amp03.onnx
├── actor_truss_X0Y0_amp03.onnx
├── actor_truss_X1Y0_amp03.onnx
└── actor_truss_X1Y1_amp03.onnx
```


### Raspberry Piでの推論

Raspberry Piでは、制御対象と外乱条件に対応するONNX `Actor` を読み込み、Yaw推定結果などから生成した制御モデルの入力を与えます。

`Actor` の出力は4基のスラスターに対する制御値です。たとえば `HSteel` / `X1Y1` 条件では、以下の `Actor`モデルを使用します。

```python
MODEL_PATH_ONNX = "models/actor_hsteel_X1Y1_amp03.onnx"
```

実行時には、Yaw推定モデルと `Actor` の両方をONNX Runtime上で動作させます。

これによりRaspberry Pi側では、カメラ画像からYaw角を推定し、その結果をもとに `Actor` が次のスラスター出力を決定するまでの処理を、PyTorchを使用せずに実行できます。

## Raspberry Piによるリアルタイム制御

リアルタイム制御では、Ubuntu側で3Dシミュレーションと物理計算を実行し、Raspberry Pi側でYaw推定とActorによる制御出力の推論を実行します。

### Ubuntu側

Ubuntuでは、Blenderから制御用サーバーを起動します。

```bash
blender5 blender/payload_runtime.blend \
    --python runtime/blender_server.py
```

Blenderで生成したカメラ画像とスラスター機体のYaw角をRaspberry Piへ送信し、Raspberry Piから返された `Action` を制御環境へ適用します。

:::message
本システムでは、スラスター機体のYaw角は取得できるものとしています。
**カメラ画像から推定するのは吊り荷のYaw角**です。
:::

制御結果はCSVとして保存し、後からBlender上で再生することで、吊り荷や制御装置の動きを目視確認できるようにしています。

### Raspberry Pi側

Raspberry Piでは、ONNX Runtimeを使用した制御ループを起動します。

```bash
python -m controller.controller_loop_onnx
```

制御ループでは、以下の処理を繰り返します。

1. Ubuntu側からカメラ画像を受信
2. ONNXモデルによるフック検出
3. 検出結果から吊り荷のYaw角を推定
4. 推定Yaw角から制御モデルの入力を生成
5. `Actor` による4基のスラスター出力を推論
6. `Action` をUbuntu側へ送信

Ubuntu側では受信したActionをスラスターへ適用して物理シミュレーションを進め、その結果から次のカメラ画像を生成します。この処理を繰り返すことで、カメラ画像をフィードバックとしたリアルタイムYaw制御を行います。

## 制御結果

Raspberry Pi上でYaw推定モデルと制御モデルを実行し、画像から推定したYaw角を使用して吊り荷を制御します。評価には、外乱が最も大きい `X1Y1` 条件を使用しました。

開発環境とRaspberry Pi環境について、以下を比較します。

- 吊り荷Yaw角
- 目標Yaw角への到達時間
- 4基のスラスター出力
- 制御中の吊り荷姿勢

目標Yaw角は `90°` とします。

### HSteel

`HSteel` では、Raspberry Pi環境でも制御開始後約10秒で目標Yaw角へ到達し、その後は `90°` 付近を維持しました。

![](/images/payload-yaw-control/04/HSteel_chart.png)

開発環境とほぼ同等の目標到達時間となり、スラスター出力についても同様の傾向が確認できました。

@[youtube](L-A6dEWhI28)

### Plate

`Plate` では、Raspberry Pi環境でも約13秒で目標Yaw角へ到達し、その後は `90°` 付近を維持しました。

![](/images/payload-yaw-control/04/Plate_chart.png)

`HSteel` と同様に、開発環境とほぼ同等の目標到達時間とスラスター出力傾向が確認できました。

@[youtube](KBwYl1DtLdU)


### Truss

`Truss` では、`HSteel`、`Plate` とは異なり、Raspberry Pi環境で目標Yaw角への到達に遅れが見られました。

![](/images/payload-yaw-control/04/Truss_chart.png)

開発環境と似たスラスター出力傾向が見られるものの、目標Yaw角への到達には遅れが生じました。

この違いには、Yaw角の推定誤差が影響した可能性があります。[Truss / X1Y1はYaw角推定誤差が最も大きい条件](https://zenn.dev/rindajones/articles/yaw_estimation#最大誤差の原因)であり、推定値のわずかな違いによって制御モデルの出力が変化したと考えられます。

ただし、最終的には目標Yaw角へ到達し、その後も目標姿勢を維持できることを確認しました。


@[youtube](YOCD4l5bW7o)


### 3 Hzでの処理

本システムでは、画像を入力とした制御処理を `3 Hz` の周期で実行することを想定しています。`3 Hz` では、1回の制御処理に使用できる時間は約 `333 ms` です。

吊り荷のYaw制御は比較的ゆっくりとした動きを対象としており、実際の手動操作を考えても、高い頻度でスラスター出力を更新する必要はありません。

そこで、Raspberry Pi上でカメラ画像を受信してから、吊り荷のYaw角推定、`Actor` による `Action` 推論を行い、`Action` をUbuntu側へ送信するまでの処理時間を計測しました。

計測対象は以下の処理です。

1. 受信画像のデコード
2. ONNXモデルによるフック検出
3. 検出結果から吊り荷のYaw角を推定
4. `Actor` による `Action` 推論
5. `Action` をUbuntu側へ送信

Ubuntu側で行うBlenderの画像生成や物理シミュレーションの処理時間は含めていません。

`1,800` フレームについて計測した結果は以下の通りです。

| 項目 | 処理時間 |
|---|---:|
| 平均 | 68.3 ms |
| 最大 | 124 ms |
| 3 Hzの制御周期 | 333 ms |

すべてのフレームで処理時間は `333 ms` 以内に収まりました。

**Raspberry Pi上で、画像受信後のYaw推定から `Action` 送信までの一連の処理を、`3 Hz` の制御周期内で実行できることを確認しました。**

## まとめ

本シリーズでは、3Dシミュレーションによる制御環境の構築から始め、強化学習によるYaw制御、カメラ画像からのYaw推定、Raspberry Pi上での画像ベース制御までを実装しました。

今回、Yaw推定モデルと制御モデルをONNX形式へ変換し、Raspberry Pi上で推論することで、GPUを使用せずに画像から吊り荷のYaw角を推定し、その結果を用いてスラスターを制御する一連の処理を実行しました。

`HSteel`、`Plate`、`Truss` の3種類の吊り荷について評価した結果、Raspberry Pi環境でも目標Yaw角90°への制御が可能であることを確認しました。

これにより、**カメラ画像 → Yaw推定 → 強化学習モデルによる推論 → スラスター制御**までを一つの閉ループとして動作させることができました。

今回の検証はシミュレーション環境上で行ったものであり、実機へ適用するためには、実環境での画像認識、外乱やアクチュエータ特性への対応など、さらに検証が必要です。一方、今回の制御結果から、制御そのものについても改善したい点が見えてきました。

次の開発では、目標Yaw角を90°固定ではなく可変とし、さらにスラスター出力を抑えた、より自然な制御を目指します。

## シリーズ構成

本シリーズでは、吊り荷Yaw制御システムの開発過程を段階的に紹介します。

- [#01 制御環境を構築する](https://zenn.dev/rindajones/articles/a8b6189171848f)
- [#02-1 強化学習の環境を構築する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_1)
- [#02-2 強化学習による制御モデルを学習する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_2)
- [#03 カメラ画像からYaw角を推定する](https://zenn.dev/rindajones/articles/yaw_estimation)
- **#04 Raspberry Piで画像ベース制御を実証する ← 今回**
- [#05 可変目標Yaw制御を目指す](https://zenn.dev/rindajones/articles/variable-target-yaw-control)
- [#06-1 可変目標Yaw制御の環境を構築する](https://zenn.dev/rindajones/articles/variable-target-env)
- [#06-2 可変目標Yaw制御モデルを学習する](https://zenn.dev/rindajones/articles/variable-target-training)
- #07 3種類の吊り荷で可変目標Yaw制御を試す
- #08 可変目標Yaw制御の結果と課題


## 関連リンク

プロジェクト全体は GitHub Pages で公開しています。
@[card](https://rindajones.github.io/payload-yaw-control/ja/)