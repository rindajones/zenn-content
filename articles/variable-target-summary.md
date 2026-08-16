---
title: "吊り荷Yaw制御システムの開発 #08：可変目標Yaw制御の結果"
emoji: "🎯"
type: "tech"
topics: ["強化学習", "python", "simulation", "robotics"]
published: true
---
# はじめに

[前回](https://zenn.dev/rindajones/articles/variable-target-infer)は、H鋼、平板、トラスの3種類の吊り荷について、可変目標Yaw制御の学習過程と推論結果を比較しました。

今回は、これまでの可変目標Yaw制御の結果をまとめ、実際の推論動画から制御動作を確認します。

本シリーズでは、以下の流れで開発を紹介しています。

[#01 制御環境を構築する](https://zenn.dev/rindajones/articles/a8b6189171848f)  
[#02-1 強化学習の環境を構築する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_1)
[#02-2 強化学習による制御モデルを学習する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_2)
[#03 カメラ画像からYaw角を推定する](https://zenn.dev/rindajones/articles/yaw_estimation)
[#04 Raspberry Piで画像ベース制御を実証する](https://zenn.dev/rindajones/articles/raspberry_pi)
[#05 可変目標Yaw制御を目指す](https://zenn.dev/rindajones/articles/variable-target-yaw-control)
[#06-1 可変目標Yaw制御の環境を構築する](https://zenn.dev/rindajones/articles/variable-target-env)
[#06-2 可変目標Yaw制御モデルを学習する](https://zenn.dev/rindajones/articles/variable-target-training)
[#07 3種類の吊り荷で可変目標Yaw制御を試す](https://zenn.dev/rindajones/articles/variable-target-infer)
**#08 可変目標Yaw制御の結果 ← 今回**


# 可変目標Yaw制御

今回の開発では、H鋼、平板、トラスの3種類について、それぞれ `-90° ～ +90°` の37目標を1つの `Actor` で制御することを目指しました。

3種類とも可変目標Yaw制御を実現できましたが、学習過程や必要なスラスター出力、その使い方には吊り荷による違いが見られました。また、目標Yaw角を可変にするだけでなく、スラスター出力を抑えた制御も実現しました。

以下は、従来の固定目標Yaw制御と今回の制御について、目標Yaw角90°での結果を比較したものです。

![従来の制御との比較](/images/payload-yaw-control/08/comparison.png)


# 制御結果を動画で確認する

ここからは、H鋼、平板、トラスについて実際の制御動作を確認します。

[前回](https://zenn.dev/rindajones/articles/variable-target-infer)でYaw角とスラスター出力の時間変化を示した `±45°` と `±90°` を使用します。

## H鋼

H鋼について、目標Yaw角 `±45°`、`±90°` の制御結果を確認します。

まず、吊り荷全体の動きを外部から確認します。

@[youtube](383pdCdNIpw)

続いて、同じ制御を制御機体に搭載したカメラから確認します。

@[youtube](HnV-sN08X3U)

:::message
搭載カメラの映像は、Yaw角推定に使用するカメラから見たものです。実際のシステムでは、このカメラ画像から吊り荷のYaw角を推定し、その結果を制御モデルへ入力します。
:::

## 平板

平板について、目標Yaw角 `±45°`、`±90°` の制御結果を確認します。

まず、吊り荷全体の動きを外部から確認します。

@[youtube](SlR9EslKa0U)

続いて、同じ制御を制御機体に搭載したカメラから確認します。

@[youtube](7qr0Pxtwld4)

## トラス

トラスについて、目標Yaw角 `±45°`、`±90°` の制御結果を確認します。

まず、吊り荷全体の動きを外部から確認します。

@[youtube](RicThtpBQjs)

続いて、同じ制御を制御機体に搭載したカメラから確認します。

@[youtube](cM1SmUSgN8E)


以上が、今回開発した可変目標Yaw制御と低出力制御の結果です。

## シリーズ構成

本シリーズでは、吊り荷Yaw制御システムの開発過程を段階的に紹介しています。

- [#01 制御環境を構築する](https://zenn.dev/rindajones/articles/a8b6189171848f)
- [#02-1 強化学習の環境を構築する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_1)
- [#02-2 強化学習による制御モデルを学習する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_2)
- [#03 カメラ画像からYaw角を推定する](https://zenn.dev/rindajones/articles/yaw_estimation)
- [#04 Raspberry Piで画像ベース制御を実証する](https://zenn.dev/rindajones/articles/raspberry_pi)
- [#05 可変目標Yaw制御を目指す](https://zenn.dev/rindajones/articles/variable-target-yaw-control)
- [#06-1 可変目標Yaw制御の環境を構築する](https://zenn.dev/rindajones/articles/variable-target-env)
- [#06-2 可変目標Yaw制御モデルを学習する](https://zenn.dev/rindajones/articles/variable-target-training)
- [#07 3種類の吊り荷で可変目標Yaw制御を試す](https://zenn.dev/rindajones/articles/variable-target-infer)
- **#08 可変目標Yaw制御の結果 ← 今回**

## 関連リンク

プロジェクト全体は GitHub Pages で公開しています。

@[card](https://rindajones.github.io/payload-yaw-control/ja/)

