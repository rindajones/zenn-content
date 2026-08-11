---
title: "吊り荷Yaw制御システムの開発 #02-2：強化学習による制御モデルを学習する"
emoji: "🏗️"
type: "tech"
topics: ["python", "pybullet", "強化学習", "simulation"]
published: true
---

## はじめに

[前回](https://zenn.dev/rindajones/articles/payload-yaw-control-02_1)までに、吊り荷Yaw制御を強化学習で扱うための環境を構築しました。

今回は、構築した環境を使用してSACによる制御モデルを学習します。3種類の吊り荷と3種類の外乱条件を組み合わせた9つのモデルを作成し、学習過程と制御性能を評価します。

さらに、学習済みモデルによる推論、Blenderでの可視化まで、実際の開発の流れに沿って紹介します。

![](/images/payload-yaw-control/development-flow.png)

本シリーズでは、以下の流れで開発を紹介しています。

[#01 制御環境を構築する](https://zenn.dev/rindajones/articles/a8b6189171848f)
[#02-1 強化学習の環境を構築する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_1)
**#02-2 強化学習による制御モデルを学習する  ← 今回**
#03 カメラ画像からYaw角を推定する  
#04 Raspberry Piで画像ベース制御を実証する


## 制御モデルの学習

#### 学習モデル

制御モデルには、強化学習アルゴリズムのSAC（Soft Actor-Critic）を使用しました。

吊り荷の種類と[外乱条件](https://zenn.dev/rindajones/articles/payload-yaw-control-02_1#外乱)を組み合わせ、それぞれ個別にSACモデルを学習します。


| 吊り荷 | X0Y0 | X1Y0 | X1Y1 |
|---|---:|---:|---:|
| H鋼 | ✓ | ✓ | ✓ |
| 平板 | ✓ | ✓ | ✓ |
| トラス | ✓ | ✓ | ✓ |

3種類の吊り荷 × 3種類の外乱条件について、合計9つのSACモデルを作成しました。

:::message
この段階では、外乱条件ごとに個別のモデルを学習しています。
その後の開発では、複数の外乱条件を一つのモデルで扱う方法へ発展させています。
:::

#### 学習の実行

例えば、H鋼・外乱なし（`X0Y0`）の学習は次のように実行します。

```bash
python -u training/SAC_fixedAlpha.py \
  --payload_name hsteel \
  --wind_amp 0.3 \
  --wind_dir X0Y0 \
  --pth models/sac_hsteel_X0Y0_amp03.pth \
2>&1 | tee training/logs/sac_hsteel_X0Y0_amp03.log
```

吊り荷と外乱条件を引数で指定することで、同じ学習コードを9つの条件で使用しています。

学習環境では、指定された吊り荷と外乱条件を制御環境へ渡し、目標Yaw角を `90°` に設定します。

```python
env_kwargs = dict(
    control_hz=3.0,
    payload_name=args.payload_name,
    wind_amp=args.wind_amp,
    wind_dir=wind_dir,
    rotate_deg=90.0,
    target_yaw_deg=90.0,
)
```

#### SACの学習条件

SACの主要な学習条件は次のように設定しました。

```python
alpha = 0.03

gamma = 0.99
tau = 0.005
batch_size = 256
warmup_steps = 2_000

total_steps = 1_500_000
eval_interval = 10_000
eval_episodes = 10
```

エントロピー係数 `alpha` は自動調整せず、`0.03` に固定しています。

学習は最大 `1,500,000 step` とし、`10,000 step` ごとに学習中の `Actor` を使って `10` 回の評価を行います。最終stepのモデルを採用するのではなく、この定期評価をもとにbestモデルを選択します。

#### モデルの評価と保存

定期評価では、成功率と最終的なYaw角の誤差を確認します。

モデルの選択では、まず成功率を優先し、成功率が同じ場合はスラスター機体と吊り荷のYaw角誤差が小さいモデルを優先しました。

```python
score = (
    -metrics["success_rate"],
    metrics["final_thr_err_abs"] + metrics["final_payload_err_abs"],
)
```

評価結果がそれまでの `best` を上回った場合、その時点のbest `Actor` として保存します。

#### 学習ログ

H鋼・外乱なし（`X0Y0`）を例に、学習の進行を見てみます。

学習初期では、目標Yaw角 `90°` に到達できない `episode` が見られます。

```text
episode 0011 | step 020900 | ... thr_yaw -132.1 | success False
episode 0012 | step 022800 | ... thr_yaw   90.1 | success True
...
episode 0016 | step 030400 | ... thr_yaw   20.8 | success False
episode 0017 | step 032300 | ... thr_yaw   90.4 | success True
```

学習が進むにつれて `90°` 付近への制御に成功する `episode` が増え、`40,000 step` の定期評価では `10` 回の評価すべてで制御に成功しました。

```text
[eval] step 040000 | final_thr_err 0.5 | final_payload_err 0.0 | success 1.00
saved best actor: success_rate=1.00, err_sum=0.52
```

その後も成功率 `1.00` を維持しながらYaw角の誤差が小さくなり、best `Actor` が更新されていきます。

```text
[eval] step 070000 | final_thr_err 0.1 | final_payload_err 0.2 | success 1.00
saved best actor: success_rate=1.00, err_sum=0.31

[eval] step 150000 | final_thr_err 0.1 | final_payload_err 0.1 | success 1.00
saved best actor: success_rate=1.00, err_sum=0.17
```

このように、学習時の報酬だけではなく、実際の制御結果を定期的に評価しながらモデルを選択しています。

#### 9モデルの学習結果

9つのモデルについて、定期評価で初めて成功率 `1.00` に到達した `step` を比較すると、次のようになりました。

| 吊り荷 | X0Y0 | X1Y0 | X1Y1 |
|---|---:|---:|---:|
| H鋼 | 40k | 70k | 60k |
| 平板 | 80k | 140k | 170k |
| トラス | 130k | 390k | 160k |

学習に必要な `step` 数は、吊り荷と外乱条件によって大きく異なりました。

また、一度評価に成功した後も、その性能が維持されるとは限りません。例えば平板・`X1Y1` では `170,000 step` で成功率 `1.00` に到達しましたが、その後の `520,000 step` の評価では制御に失敗しています。

```text
[eval] step 520000 | final_thr_err 92.6 | final_payload_err 0.1 | success 0.00
```

一方、学習途中には成功率 `1.00` かつYaw角誤差の小さい `Actor` が保存されています。

```text
saved best actor: success_rate=1.00, err_sum=0.23
```

この結果からも、最終 `step` のモデルをそのまま採用するのではなく、学習中に定期評価を行い、その時点までで最も評価の高い `Actor` を保存することが重要だと分かります。

9つのモデルについて、定期評価で初めて成功率 `1.00` に到達した `step` と、最終的にbest `Actor`が保存された `step` を比較します。

| 吊り荷 | 外乱 | 初回成功 | 最終best保存 |
|---|---|---:|---:|
| H鋼 | X0Y0 | 40k | 150k |
| H鋼 | X1Y0 | 70k | 160k |
| H鋼 | X1Y1 | 60k | 300k |
| 平板 | X0Y0 | 80k | 190k |
| 平板 | X1Y0 | 140k | 350k |
| 平板 | X1Y1 | 170k | 250k |
| トラス | X0Y0 | 130k | 220k |
| トラス | X1Y0 | 390k | 1,390k |
| トラス | X1Y1 | 160k | 930k |

すべての条件で、初めて成功率 `1.00` に到達した後もbest `Actor`が更新されました。特にトラスでは、初回成功から大きく学習が進んだ後にbest `Actor` が得られる条件もあります。

また、学習中の性能は常に安定して向上するわけではなく、一度成功した後に制御に失敗するケースも確認されました。

今回の学習では、一度の成功を学習完了と判断せず、定期評価を継続しながら、その時点までで最も評価の高い `Actor` を保存する方法を採用しました。


### 制御モデルの評価

#### 推論

学習時に保存したbest `Actor` を使用して推論を行います。

ここでは代表例として、トラス・`X1Y1` の推論を実行します。

```bash
python training/infer.py \
  --algo SAC_fixedAlpha \
  --payload_name truss \
  --model models/sac_truss_X1Y1_amp03.pth \
  --out training/eval/sac_truss_X1Y1_amp03.csv \
  --wind-amp 0.3 \
  --wind-x 1.0 \
  --wind-y 1.0 \
  --crane-csv training/crane_90degree.csv
```

推論では、学習済みモデルから一定周期で4基のスラスター出力を算出し、Yaw角やスラスター出力などの制御結果をCSVファイルへ保存します。

`--crane-csv` では、Blenderでのアニメーション用にクレーンの旋回データを指定しています。

#### Blenderによる可視化

推論結果として保存したCSVをBlenderへ読み込み、制御時の動きを可視化します。

PyBulletによる制御計算とBlenderによる可視化を分離することで、同じ制御結果を3D空間上で確認できます。

@[youtube](SE1dw4ta94s)

動画では、左側にクレーンと吊り荷の全景、右側に吊り荷を上から見た映像を表示しています。右側はスラスター機体の真下に設置したカメラからの視点で、後の画像によるYaw角推定でも使用します。

:::message
この時点では、PyBulletによる制御結果とBlender側のクレーン旋回アニメーションの同期が完全ではなく、
動画の一部でクレーンのワイヤとスラスター機体の位置にずれが見られます。
この点は、その後の開発で修正しています。
:::

#### 制御結果

Blenderで可視化したトラス・`X1Y1` について、同じ推論結果をチャートで確認します。

![Truss / X1Y1 制御結果](/images/payload-yaw-control/02/truss_X1Y1.png)

上段は制御なし（Baseline）、中段はSACによる制御結果、下段は4基のスラスター出力を示しています。スラスター出力はヒートマップで表し、黄色に近いほど出力が大きく、紫色に近いほど小さいことを示します。

制御なしでは吊り荷は目標角へ向かいませんが、SACによる制御では約40秒で目標Yaw角 `90°` へ到達し、その後は目標角付近を維持しています。

HSteel、Plate、Trussの各条件での制御結果は、[GitHub Pages](https://rindajones.github.io/payload-yaw-control/ja/control_model/#chart) にまとめています。


#### 9モデルの精度比較

9つの学習済みモデルについて、同じ条件で推論を行い、制御性能を比較しました。

評価には、次の3つの指標を使用します。

- **到達時間**：目標Yaw角 `90°` に対して誤差 `±5°` 以内の状態を `10 control step` 維持するまでの時間
- **到達後MAE**：目標角へ到達した後のYaw角の平均絶対誤差
- **到達後SD**：目標角へ到達した後のYaw角の標準偏差

| 吊り荷 | 外乱 | 到達時間 [s] | 到達後MAE [°] | 到達後SD [°] |
|---|---|---:|---:|---:|
| H鋼 | X0Y0 | 9.5 | 0.39 | 0.97 |
| H鋼 | X1Y0 | 8.7 | 0.26 | 0.51 |
| H鋼 | X1Y1 | 9.2 | 0.85 | 1.01 |
| 平板 | X0Y0 | 14.4 | 3.19 | 0.48 |
| 平板 | X1Y0 | 15.8 | 0.98 | 0.80 |
| 平板 | X1Y1 | 13.2 | 0.53 | 0.79 |
| トラス | X0Y0 | 29.8 | 0.54 | 1.02 |
| トラス | X1Y0 | 18.1 | 0.51 | 0.85 |
| トラス | X1Y1 | 21.3 | 0.28 | 0.64 |

9つの条件すべてで目標Yaw角への到達を確認できました。

到達時間を見ると、H鋼は約9秒、平板は約13〜16秒、トラスは約18〜30秒となり、吊り荷によって制御特性に違いが見られます。

一方、外乱が大きくなるほど単純に精度が低下する結果にはなりませんでした。例えばトラス・`X1Y1` では、到達後MAE 0.28°、SD 0.64°となり、外乱がある条件でも目標角付近を安定して維持しています。

また、平板・X0Y0では到達後MAEが3.19°と他の条件より大きい一方、SDは0.48°と小さくなっています。これはYaw角が大きく変動しているのではなく、目標 `90°` からややずれた角度で安定していることを示しています。

このように、目標角へ到達できるかだけでなく、到達までの時間、目標角からの誤差、到達後の変動を分けて確認することで、各モデルの制御特性を比較できます。


## シリーズ構成

本シリーズでは、吊り荷Yaw制御システムの開発過程を段階的に紹介します。

- [#01 制御環境を構築する](https://zenn.dev/rindajones/articles/a8b6189171848f)
- [#02-1 強化学習の環境を構築する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_1)
- **#02-2 強化学習による制御モデルを学習する ← 今回**
- #03 カメラ画像からYaw角を推定する
- #04 Raspberry Piで画像ベース制御を実証する


## 関連リンク

プロジェクト全体は GitHub Pages で公開しています。
@[card](https://rindajones.github.io/payload-yaw-control/ja/)