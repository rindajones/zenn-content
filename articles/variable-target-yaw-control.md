---
title: "吊り荷Yaw制御システムの開発 #05：可変目標Yaw制御を目指す"
emoji: "🎯"
type: "tech"
topics: ["強化学習", "python", "simulation", "robotics"]
published: true
---

# はじめに

[前回](https://zenn.dev/rindajones/articles/raspberry_pi)までの記事では、カメラ画像から吊り荷のYaw角を推定し、その結果を使って4基のスラスターを制御するシステムを構築しました。

これまでの制御では、吊り荷を一定の目標Yaw角へ制御することを目的としていました。

しかし、実際の制御結果を確認していく中で、次の2つを改善したいと考えるようになりました。

- 目標Yaw角を固定値ではなく、任意の角度に変更できるようにする
- 目標角へ到達するだけでなく、できるだけ低いスラスター出力で制御する

そこでここからは、**可変目標Yaw制御**と**低出力制御**に取り組みます。

本記事では、何を目指したのか、その背景と方針についてまとめます。

本シリーズでは、以下の流れで開発を紹介しています。

[#01 制御環境を構築する](https://zenn.dev/rindajones/articles/a8b6189171848f)  
[#02-1 強化学習の環境を構築する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_1)  
[#02-2 強化学習による制御モデルを学習する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_2)  
[#03 カメラ画像からYaw角を推定する](https://zenn.dev/rindajones/articles/yaw_estimation)  
[#04 Raspberry Piで画像ベース制御を実証する](https://zenn.dev/rindajones/articles/raspberry_pi)  
**#05 可変目標Yaw制御を目指す ← 今回**  
[#06-1 可変目標Yaw制御の環境を構築する](https://zenn.dev/rindajones/articles/variable-target-env)
[#06-2 可変目標Yaw制御モデルを学習する](https://zenn.dev/rindajones/articles/variable-target-training)
[#07 3種類の吊り荷で可変目標Yaw制御を試す](https://zenn.dev/rindajones/articles/variable-target-infer)
[#08 可変目標Yaw制御の結果](https://zenn.dev/rindajones/articles/variable-target-summary)



# これまでの制御から見えた課題

これまでの開発では、H鋼、平板、トラスの3種類の吊り荷について、Raspberry Pi上で画像からYaw角を推定しながらリアルタイムに制御できることを確認しました。

一方で、実際の動きやスラスター出力を確認すると、

> **目標角へ到達できれば、それでよいのか**

という疑問が残りました。

特に見直したかったのが、**スラスター出力**と**目標Yaw角の扱い**です。

# スラスター出力を抑える

これまでの制御では、目標Yaw角へ到達することを優先していました。

その結果、目標角には到達するものの、必要以上に大きなスラスター出力を使用したり、出力が大きく変化したりする場面が見られました。

![これまでのスラスター出力](/images/payload-yaw-control/05/old_controls.png)
*これまでの制御におけるスラスター出力の例*

ヒートマップは、横軸が時間、縦軸が4基のスラスター（T1〜T4）、色が正規化したスラスター出力を表しています。

目標角へ到達しているという意味では制御は成功しています。しかし実際の制御を考えると、常に大きな出力を使用することが望ましいとは限りません。

そこで、

> **必要なときだけ、必要な大きさの出力を使用する**

ことを新たな目標としました。

単純にスラスターの最大出力を小さくするのではなく、強化学習モデル自身が状況に応じて低い出力を選択できるようにします。

具体的には、次のような制御を目指します。

1. **最大出力を抑える**
2. **平均的な出力を抑える**
3. **急激な出力変化を抑える**
4. **目標角へ到達した後は低出力を維持する**

目指しているのは、単に出力を小さくすることではありません。

Yaw角を制御するために必要な出力は使用しながら、**不必要な出力をできるだけ使わない制御**です。

![低出力制御の例](/images/payload-yaw-control/05/low-output-control.png)
*低出力化したスラスター制御の例*

# 目標Yaw角を可変にする

もう一つの変更が、**目標Yaw角の可変化**です。

これまでは、学習時に目標Yaw角を固定していました。この方法では、異なる目標角を使用する場合、それぞれの目標に対応したモデルを用意する必要があります。

しかし実際の運用を考えると、

> 0°へ制御するモデル  
> 45°へ制御するモデル  
> 90°へ制御するモデル

のように目標角ごとにモデルを切り替えるよりも、**1つのモデルへ目標角を与えて制御できる方が自然**です。

![可変目標Yaw制御](/images/payload-yaw-control/05/variable-target-yaw.png)
*目標Yaw角を指定して吊り荷を旋回させる*

そこで、目標Yaw角そのものを制御環境へ与え、1つの強化学習モデルで異なる目標Yaw角へ制御できることを目指します。

評価では、代表的な目標角として次の5条件を使用します。

- -90°
- -45°
- 0°
- +45°
- +90°

+方向は反時計回り（CCW）、-方向は時計回り（CW）としています。

![評価する目標Yaw角](/images/payload-yaw-control/05/reached_target_yaws.png)
*評価に使用する5つの目標Yaw角*

# 主な変更点

今回、新しい制御システムを一から作り直すわけではありません。

吊り荷、スラスター、ロープ拘束などの基本的なシミュレーション構成は、これまでに構築したものをそのまま使用します。

主に変更するのは、**強化学習の制御環境と学習方法**です。

| 項目 | これまで | 今回 |
| --- | --- | --- |
| 目標Yaw角 | 固定 | 可変 |
| 学習モデル | 目標条件ごと | 1モデル |
| スラスター出力 | 到達を優先 | 低出力も考慮 |
| 評価 | 固定目標 | 0°、±45°、±90° |

つまり今回の開発では、これまで構築した制御システムをベースとして、

> **「どこへ向けるか」と「どう動かすか」**

を改善します。

# 対象とする吊り荷

対象とする吊り荷もこれまでと同じです。

- H鋼（`HSteel`）
- 平板（`Plate`）
- トラス（`Truss`）

形状や質量特性の異なる3種類の吊り荷に対して、同じ考え方で可変目標Yaw制御と低出力制御を適用します。


## シリーズ構成

本シリーズでは、吊り荷Yaw制御システムの開発過程を段階的に紹介します。

- [#01 制御環境を構築する](https://zenn.dev/rindajones/articles/a8b6189171848f)
- [#02-1 強化学習の環境を構築する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_1)
- [#02-2 強化学習による制御モデルを学習する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_2)
- [#03 カメラ画像からYaw角を推定する](https://zenn.dev/rindajones/articles/yaw_estimation)
- [#04 Raspberry Piで画像ベース制御を実証する](https://zenn.dev/rindajones/articles/raspberry_pi)
- **#05 可変目標Yaw制御を目指す ← 今回**
- [#06-1 可変目標Yaw制御の環境を構築する](https://zenn.dev/rindajones/articles/variable-target-env)
- [#06-2 可変目標Yaw制御モデルを学習する](https://zenn.dev/rindajones/articles/variable-target-training)
- [#07 3種類の吊り荷で可変目標Yaw制御を試す](https://zenn.dev/rindajones/articles/variable-target-infer)
- [#08 可変目標Yaw制御の結果](https://zenn.dev/rindajones/articles/variable-target-summary)



## 関連リンク

プロジェクト全体は GitHub Pages で公開しています。
@[card](https://rindajones.github.io/payload-yaw-control/ja/)