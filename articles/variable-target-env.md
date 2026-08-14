---
title: "吊り荷Yaw制御システムの開発 #06-1：可変目標Yaw制御の環境を構築する"
emoji: "🎯"
type: "tech"
topics: ["強化学習", "python", "simulation", "robotics"]
published: true
---

# はじめに

[前回](https://zenn.dev/rindajones/articles/variable-target-yaw-control)は、これまでの吊り荷Yaw制御を発展させ、

- 目標Yaw角を固定値ではなく可変にする
- 目標角へ到達するだけでなく、できるだけ低いスラスター出力で制御する

という2つの目標について紹介しました。今回は、この方針を強化学習へ組み込むために、**制御環境をどのように変更したか**を紹介します。

今回の変更で特に重視したのは、

> **何を「制御に成功した」と考えるか**

です。

目標Yaw角へ到達できても、勢いよく通過しただけだったり、必要以上に大きな出力を使い続けていたりすれば、今回目指している制御とは異なります。

そこで、単に目標角へ到達したかを判定する `physical_success` と、出力の使い方まで含めて評価する `quality_success` を分けて扱うことにしました。

本シリーズでは、以下の流れで開発を紹介しています。

[#01 制御環境を構築する](https://zenn.dev/rindajones/articles/a8b6189171848f)  
[#02-1 強化学習の環境を構築する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_1)  
[#02-2 強化学習による制御モデルを学習する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_2)  
[#03 カメラ画像からYaw角を推定する](https://zenn.dev/rindajones/articles/yaw_estimation)  
[#04 Raspberry Piで画像ベース制御を実証する](https://zenn.dev/rindajones/articles/raspberry_pi)  
[#05 可変目標Yaw制御を目指す](https://zenn.dev/rindajones/articles/variable-target-yaw-control)  
**#06-1 可変目標Yaw制御の環境を構築する ← 今回**  
#06-2 可変目標Yaw制御モデルを学習する  
#07 3種類の吊り荷で可変目標Yaw制御を試す  
#08 可変目標Yaw制御の結果と課題

# 目標Yaw角を可変にする

これまでの制御では、目標Yaw角を固定して学習していました。

今回は、エピソードごとに異なる目標Yaw角を設定できるようにしました。最終的な学習範囲は、

```text
-90° ～ +90°
```

で、5°刻みの37目標を使用します。

```text
-90, -85, -80, ... , -5, 0, +5, ... , +80, +85, +90
```

これにより、目標角ごとに別モデルを用意するのではなく、**1つのモデルで複数の目標Yaw角を扱う**ことを目指します。

学習時に37目標をどのように選択するかは、次回の記事で紹介します。

# 観測値

観測値は7次元としました。

```text
thr_err
payload_err
thr_yaw_rate
payload_yaw_rate
thr_yaw
payload_yaw
current_yaw_command
```

内容は次のとおりです。

| 観測値 | 内容 |
| --- | --- |
| `thr_err` | スラスター機体Yawと目標Yawの誤差 |
| `payload_err` | 吊り荷とスラスター機体の相対Yaw誤差 |
| `thr_yaw_rate` | スラスター機体のYaw角速度 |
| `payload_yaw_rate` | 吊り荷のYaw角速度 |
| `thr_yaw` | スラスター機体の絶対Yaw角 |
| `payload_yaw` | 吊り荷の絶対Yaw角 |
| `current_yaw_command` | 現在適用しているYaw指令 |

今回追加したのが `current_yaw_command` です。

後述するように、今回は1回の制御更新で変更できるYaw指令量に上限を設けています。そのため、モデルが次の行動を決める際には、目標との誤差だけでなく、**現在どの程度の指令が適用されているか**も状態として与えるようにしました。

# 行動

`Actor` が出力する行動は1次元のYaw指令です。

```python
self.action_space = spaces.Box(
    low=-1.0,
    high=1.0,
    shape=(1,),
    dtype=np.float32,
)
```

符号が旋回方向を表します。

```text
+ : 反時計回り（CCW）
- : 時計回り（CW）
```

このYaw指令から、旋回方向と指令の大きさに応じて4基のスラスター出力を決定します。

つまり、強化学習モデルが4基のスラスターを個別に操作するのではなく、**「どちらへ、どの程度回すか」を1つの指令として出力する**構成です。

# 「到達した」だけでは成功としない

今回の環境では、成功を2段階に分けました。

```text
physical_success
        ↓
quality_success
```

`physical_success` は、物理的に目標Yaw角へ到達し、安定した状態に入ったかを判定します。
`quality_success` は、その成功が**低出力という今回の方針も満たしているか**を評価します。

# physical_success：安定して到達したか

まず、目標Yaw角へ入っただけでは成功としません。次の条件を同時に満たす必要があります。

```text
|吊り荷Yaw - 目標Yaw| < 5°
|スラスター機体Yaw角速度| <= 3°/s
|吊り荷Yaw角速度| <= 3°/s
```

さらに、この状態を10フレーム連続して満たした場合に `physical_success` とします。

```python
stable_now = (
    abs(self.payload_target_err) < math.radians(5.0)
    and abs(self.thr_yaw_rate) <= yaw_rate_limit
    and abs(self.payload_yaw_rate) <= yaw_rate_limit
)
```

これは、吊り荷が大きな角速度を持ったまま目標角を一瞬通過しただけの状態を成功と数えないためです。今回の制御では、

> **目標角へ入ることではなく、目標角付近で落ち着くこと**

を物理的な成功としました。

# 到達前と到達後を分けて見る

低出力制御を評価する際には、エピソード全体の平均出力だけを見ると問題があります。例えば、

```text
短時間だけ大出力
↓
その後ずっと低出力
```

という制御では、エピソード全体の平均だけを見ると、最初の大きな出力が目立たなくなります。そこで出力統計を、

```text
reach : 初回到達まで
hold  : 初回到達後
```

に分けました。

`reach` では、目標へ向かう途中にどの程度の出力を使ったかを見ます。
`hold` では、目標到達後にどの程度の出力で姿勢を維持したかを見ます。

今回の低出力制御では、この2つを分けることが重要でした。

# quality_success：どのように到達したか

`quality_success` は、`physical_success` を満たしたうえで、出力の使い方も評価します。

主な条件は、

```text
physical_success
hold期間が100 step以上
reach中の平均指令が上限以内
reach中の高出力率が上限以内
hold中の平均指令 <= 0.20
```

です。コードでは概ね次のような判定になっています。

```python
quality_success = (
    physical_success
    and hold_steps >= 100
    and reach_mean_action <= reach_mean_limit
    and reach_high_ratio <= reach_high_limit
    and hold_mean_action <= 0.20
)
```

ただし、初期状態ですでに成功範囲内にある場合には、実質的な `reach` が存在しないため、`reach` 条件は適用しません。

# 大きな目標角では必要な出力を許す

`reach` 中の出力上限を全目標で同じにすると、大きな旋回角まで必要以上に低出力へ制限してしまいます。そこで `reach` 中の品質条件は、目標角の大きさに応じて変化させました。

平均指令の上限は、

```python
reach_mean_limit = min(
    0.50,
    0.15 + 0.005 * abs(target_deg)
)
```

高出力を使用した割合の上限は、

```python
reach_high_limit = min(
    0.30,
    0.05 + 0.003 * abs(target_deg)
)
```

としています。つまり、大きな角度へ旋回するときには必要な加速を許し、小さな角度では不必要な大出力を抑えます。

目指しているのは、

> **出力を小さくすることそのものではなく、必要な出力だけを使うこと**

です。

# 目標へ近づくほど指令上限を下げる

`Actor` が要求したYaw指令をそのまま使用するのではなく、目標Yaw角との誤差に応じて指令上限を変化させました。

```python
command_limit = min(
    1.0,
    0.15 + 0.01 * abs(error_deg)
)
```

例えば次のようになります。

| 目標角との誤差 | 指令上限 |
| ---: | ---: |
| 0° | 0.15 |
| 20° | 0.35 |
| 60° | 0.75 |
| 90° | 1.00 |

目標から遠いときには大きな出力を許し、近づくほど上限を下げます。単純に最大出力を小さく固定すると、大きく旋回するときに必要な出力まで使えなくなります。

そこで、

> **大きく動かすときは出力を使い、目標へ近づいたら抑える**

という制約にしました。

# 出力の急変を抑える

さらに、1回の制御更新で変更できるYaw指令量を、

```text
command_rate_limit = 0.10
```

に制限しました。

例えば現在の指令が `0.20` なら、次の更新でいきなり `0.80` へ変更することはできません。また、指令変化量の二乗にもコストを与えます。

これにより、最大出力を急にON/OFFするような制御ではなく、**出力を滑らかに変化させる制御**を狙います。

# 平均出力にもコストを与える

各 `step` では、実際に適用したスラスター出力の二乗和にコストを与えます。

```python
action_cost = np.sum(np.square(self.current_action))
thrust_cost_penalty = thrust_cost_weight * action_cost
```

今回の主な学習では、

```text
thrust_cost_weight = 0.30
```

を使用しました。

一方で、低出力を優先しすぎると「何もしない」方策が有利になります。そのため優先順位は、

1. 目標Yaw角へ安定して到達する
2. 成功を満たす範囲で出力を抑える

としています。

# 到達後はさらに出力を下げる

目標へ到達した後には、Raw指令と実際に適用された指令にも追加のコストを与えています。

また、`physical_success` を満たした場合には、保持時の平均指令が小さいほど大きくなる低出力ボーナスを与えます。

```python
low_thrust_bonus = (
    low_thrust_success_bonus_weight
    * (1.0 - efficiency_action)
)
```

ここで重要なのは、`quality_success` を満たしたときだけ突然ボーナスを与えるのではなく、**`physical_success` 後の出力に応じて連続的に報酬を変化させる**ようにしたことです。

品質条件の境界を少し超えただけで報酬が大きく変わることを避け、より低い出力へ向かう学習信号を残しています。

# 今回の「成功」の考え方

今回の環境では、成功を次のように整理しました。

```text
目標角へ入る
    ↓
角速度を落として安定する
    ↓
physical_success
    ↓
到達までの出力は適切だったか
到達後も低出力で保持できたか
    ↓
quality_success
```

前回の記事で書いた、

> **目標角へ到達できれば、それでよいのか**

という疑問に対して、環境側では `physical_success` と `quality_success` を分けることで対応しました。

# 今回のまとめ

今回は、可変目標Yaw制御と低出力制御のために、強化学習環境を変更しました。

主な変更点は次のとおりです。

- 目標Yaw角を可変化
- 7次元の観測に現在のYaw指令を追加
- 1次元のYaw指令から4基のスラスターを制御
- 角度誤差とYaw角速度から `physical_success` を判定
- 到達前を `reach`、到達後を `hold` に分けて出力を評価
- 出力品質まで含めた `quality_success` を追加
- 目標誤差に応じて指令上限を変更
- 指令変化量を制限
- 平均出力と到達後の出力を抑制

これで、

> **どこへ向けるか**

だけでなく、

> **どのようにそこへ到達するか**

まで評価できる環境になりました。

また、目標角との誤差に応じた指令上限や指令変化量の制限などを加え、必要な出力を使いながら、不必要な出力を抑えることを目指しています。

次回は、実際の学習過程と、その中で学習方法をどのように変更していったかを紹介します。


## シリーズ構成

本シリーズでは、吊り荷Yaw制御システムの開発過程を段階的に紹介します。

- [#01 制御環境を構築する](https://zenn.dev/rindajones/articles/a8b6189171848f)
- [#02-1 強化学習の環境を構築する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_1)
- [#02-2 強化学習による制御モデルを学習する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_2)
- [#03 カメラ画像からYaw角を推定する](https://zenn.dev/rindajones/articles/yaw_estimation)
- [#04 Raspberry Piで画像ベース制御を実証する](https://zenn.dev/rindajones/articles/raspberry_pi)
- [#05 可変目標Yaw制御を目指す](https://zenn.dev/rindajones/articles/variable-target-yaw-control)
- **#06-1 可変目標Yaw制御の環境を構築する ← 今回**
- #06-2 可変目標Yaw制御モデルを学習する
- #07 3種類の吊り荷で可変目標Yaw制御を試す
- #08 可変目標Yaw制御の結果と課題

## 関連リンク

プロジェクト全体は GitHub Pages で公開しています。
@[card](https://rindajones.github.io/payload-yaw-control/ja/)
