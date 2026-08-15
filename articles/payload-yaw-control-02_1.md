---
title: "吊り荷Yaw制御システムの開発 #02-1：強化学習の環境を構築する"
emoji: "🏗️"
type: "tech"
topics: ["python", "pybullet", "強化学習", "simulation"]
published: True
---

## はじめに

[前回](https://zenn.dev/rindajones/articles/a8b6189171848f)は、BlenderとPyBulletを用いて、吊り荷Yaw制御を検証するためのシミュレーション環境を構築しました。

今回は、この環境を強化学習の制御環境として構成します。吊り荷のYaw角を制御するための観測値、`Action`、`Reward`、`成功条件`を定義し、SACによる学習を実行できる環境を構築します。

強化学習で何を観測し、どのような操作を行い、その結果をどう評価するかという、制御環境の設計を実際の開発の流れに沿って紹介します。

![](/images/payload-yaw-control/development-flow.png)

本シリーズでは、以下の流れで開発を紹介しています。

[#01 制御環境を構築する](https://zenn.dev/rindajones/articles/a8b6189171848f)
**#02-1 強化学習の環境を構築する ← 今回**
[#02-2 強化学習による制御モデルを学習する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_2)
[#03 カメラ画像からYaw角を推定する](https://zenn.dev/rindajones/articles/yaw_estimation)
[#04 Raspberry Piで画像ベース制御を実証する](https://zenn.dev/rindajones/articles/raspberry_pi)
[#05 可変目標Yaw制御を目指す](https://zenn.dev/rindajones/articles/variable-target-yaw-control/)
[#06-1 可変目標Yaw制御の環境を構築する](https://zenn.dev/rindajones/articles/variable-target-env)
[#06-2 可変目標Yaw制御モデルを学習する](https://zenn.dev/rindajones/articles/variable-target-training)
#07 3種類の吊り荷で可変目標Yaw制御を試す  
#08 可変目標Yaw制御の結果と課題



## 開発環境

制御モデルの学習および評価には、以下の環境を使用しました。

| 項目 | 環境 |
| --- | --- |
| OS | Ubuntu 24.04 LTS |
| 開発言語 | Python 3.11 |
| 物理シミュレーション | PyBullet |
| 強化学習環境 | Gymnasium |
| 機械学習フレームワーク | PyTorch |
| 可視化 | Blender 5.1.2 |

## 制御モデルの学習から評価まで

今回構築した、制御モデルの学習から評価までの全体構成を以下に示します。

![](/images/payload-yaw-control/control_model_flow_jp.png)

物理計算には PyBullet を使用し、推論結果の挙動確認には Blender を使用しました。

## 物理シミュレーションと制御環境

強化学習による制御モデルの学習には、物理エンジンとして PyBullet、制御環境のインターフェースとして Gymnasium を使用しました。

吊り荷モデルを PyBullet 上で動作させ、Gymnasium の環境から吊り荷の状態を取得するとともに、4基のスラスターへ制御出力を与えます。

### PyBulletによる物理シミュレーション

PyBulletでは、吊り荷、ロープ、スラスター機体から構成される物理モデルを扱います。

主に計算する要素は以下です。
- 重力
- ロープ拘束
- 慣性モーメント
- スラスター力
- 外乱

吊り荷やスラスター機体の形状、寸法、質量などのモデル構成については、[前回の記事](https://zenn.dev/rindajones/articles/a8b6189171848f#blenderオブジェクトの構成)で紹介しています。

今回の記事では、これらを PyBullet 上でどのように物理シミュレーションとして実装し、強化学習用の制御環境へ接続したかを見ていきます。

```python
self.dt = 1.0 / 240.0

self.client = p.connect(p.GUI if gui else p.DIRECT)
p.setGravity(0, 0, -9.81)
p.setTimeStep(self.dt)
```
物理シミュレーションの時間刻みは `1/240` 秒とし、重力加速度を設定します。学習時には画面描画を必要としないため、PyBullet を `DIRECT` モードで実行できます。

スラスター機体や吊り荷は、質量、形状、姿勢、慣性モーメントなどを設定して PyBullet 上に生成します。
```python
body = p.createMultiBody(
    baseMass=mass,
    baseCollisionShapeIndex=col,
    baseVisualShapeIndex=vis,
    basePosition=pos,
    baseOrientation=p.getQuaternionFromEuler([0, 0, yaw]),
)

p.changeDynamics(
    body,
    -1,
    linearDamping=0.08,
    angularDamping=0.8,
    localInertiaDiagonal=inertia,
)
```
シミュレーション実行時には、ロープによる拘束力、外乱、スラスター力を順に適用し、PyBullet の物理計算を進めます。

#### 外乱

実際の吊り荷では、風などの外乱によって姿勢が変化します。そこで制御環境では、吊り荷およびスラスター機体に外力を与えられるようにしました。

学習・評価では、外乱なしに加えて、X方向・Y方向の成分を組み合わせた複数の条件を使用します。
```python
def _apply_wind(self, t):
    wx = self.wind_dir[0] * self.wind_amp * math.sin(
        2.0 * math.pi * self.wind_freq * t
    )
    wy = self.wind_dir[1] * self.wind_amp * math.sin(
        2.0 * math.pi * self.wind_freq * t + math.pi / 2.0
    )

    force_thr = np.array([
        wx * self.thr_mass,
        wy * self.thr_mass,
        0.0,
    ])

    force_payload = np.array([
        wx * self.payload_cfg["mass"],
        wy * self.payload_cfg["mass"],
        0.0,
    ])

    p.applyExternalForce(...)
```    
外乱はX・Y方向の周期的な外力として与え、スラスター機体と吊り荷の両方に作用させます。外力の大きさはそれぞれの質量に比例させています。

外乱の強さは `wind_amp = 0.3` に固定し、方向を変えた以下の条件を使用しました。

| 条件 | `wind_dir` | 外乱 |
| --- | --- | --- |
| X0Y0 | `(0.0, 0.0)` | 外乱なし |
| X1Y0 | `(1.0, 0.0)` | X方向 |
| X1Y1 | `(1.0, 1.0)` | X・Y方向 |

:::message
この外乱は風荷重そのものを厳密に再現したものではありません。制御性能を確認するため、周期的な水平外乱として簡略化して実装しています。
:::

この外乱を含め、各種の力を毎ステップ適用した後に、PyBullet の物理計算を進めます。
```python
for _ in range(self.steps_per_frame):
    t = self.frame / self.fps
    self._apply_hook_rope()
    self._apply_payload_ropes()
    self._apply_wind(t)
    self._apply_thrusters(self.current_action)
    p.stepSimulation()
```
この物理シミュレーションを、次に Gymnasium をベースとした強化学習用の制御環境から操作します。


### Gymnasiumによる制御環境

強化学習モデルから物理シミュレーションを扱えるように、Gymnasium をベースとして吊り荷Yaw制御用の環境を実装しました。

制御環境は、PyBullet から吊り荷やスラスター機体の状態を取得し、強化学習モデルから出力された `Action` を4基のスラスターへ与えます。

Gymnasiumの `step()` では、主に以下の処理を繰り返します。

1. 強化学習モデルから `Action` を受け取る
2. `Action` を 4基のスラスターへ与える
3. PyBullet で物理シミュレーションを進める
4. 新しい物理状態から `Observation` を生成する
5. `Reward` と終了条件を計算する

制御モデルによる `Action` の更新周期は `3 Hz` としました。一度決定した `Action` は次の制御タイミングまで保持し、その間も PyBullet による物理計算を継続します。

#### Observation

`Observation` は、強化学習モデルが制御を判断するために使用する現在の状態です。今回の Yaw制御では、吊り荷とスラスター機体の Yaw角を中心に、制御に必要な状態を `Observation` として与えます。

主な `Observation` は以下の5つです。

- スラスター機体と目標Yaw角の誤差
- 吊り荷とスラスター機体のYaw角の相対誤差
- Yaw方向の角速度
- スラスター機体のYaw角
- 吊り荷のYaw角

なお、Yaw方向の角速度は今回の学習では使用せず、`0.0` 固定としています。

```python
def _get_obs(self):
    _, thr_orn = p.getBasePositionAndOrientation(self.thr_id)
    _, payload_orn = p.getBasePositionAndOrientation(self.payload_id)

    _, _, self.thr_yaw = p.getEulerFromQuaternion(thr_orn)
    _, _, self.payload_yaw = p.getEulerFromQuaternion(payload_orn)

    self.thr_err = self._wrap_pi(
        self.thr_yaw - self.target_yaw
    )

    self.payload_err = self._wrap_pi(
        self.payload_yaw - self.thr_yaw
    )

    yaw_rate = 0.0

    return np.array([
        self.thr_err,
        self.payload_err,
        yaw_rate,
        self.thr_yaw,
        self.payload_yaw,
    ], dtype=np.float32)
```
`thr_err` はスラスター機体と目標Yaw角との差、`payload_err` は吊り荷とスラスター機体とのYaw角の差です。この2つを `Observation` に含めることで、目標角への旋回だけでなく、スラスター機体と吊り荷の相対姿勢も `制御状態` として与えています。


#### Action

`Action` は、強化学習モデルが制御対象へ与える出力です。

今回のYaw制御では、4基のスラスターそれぞれの出力を `Action` として扱います。スラスターの配置と、それぞれの出力によって発生するYaw回転については、[前回の記事「スラスター出力とYaw回転」](https://zenn.dev/rindajones/articles/a8b6189171848f#スラスター出力とyaw回転)で紹介しています。

各 `Action` の範囲は `0.0〜1.0` とし、`1.0` のとき最大推力 `250 N` を与えます。


```python
self.action_space = spaces.Box(
    low=0.0,
    high=1.0,
    shape=(4,),
    dtype=np.float32,
)
```

4つの `Action` は、左前・左後・右前・右後の4基のスラスターに対応します。
```python
# action = [LF, LR, RF, RR]
```

スラスター出力は、`Action` の値に最大推力を掛けて実際の力へ変換します。
```python
def _apply_thrusters(self, action):
    action = np.asarray(action, dtype=np.float32)
    action = np.clip(action, 0.0, 1.0)

    for i in range(4):
        force = world_dir * (self.max_thrust * action[i])

        p.applyExternalForce(
            objectUniqueId=self.thr_id,
            linkIndex=-1,
            forceObj=force.tolist(),
            posObj=world_pos.tolist(),
            flags=p.WORLD_FRAME,
        )
```
今回の設定では最大推力を `250 N` としているため、例えば `Action = 0.4` の場合、そのスラスターには `100 N` の推力が与えられます。

#### Reward

`Reward` は、強化学習モデルが選択した `Action` を評価するための値です。

今回のYaw制御では、スラスター機体と吊り荷のYaw誤差を小さくしながら、目標角へ近づく動きを評価します。

主な評価要素は以下です。

- スラスター機体と目標Yaw角の誤差
- 吊り荷とスラスター機体のYaw角の相対誤差
- 前ステップから誤差がどれだけ減少したか
- スラスター出力の大きさ
- `Success` 時のボーナス

```python
def _get_reward(self):
    abs_thr = abs(self.thr_err)
    abs_payload = abs(self.payload_err)

    total_err = abs_thr + abs_payload

    if self.prev_total_err is None:
        progress = 0.0
    else:
        progress = self.prev_total_err - total_err

    action_cost = float(np.sum(np.square(self.current_action)))

    reward = 100.0 * progress
    reward -= 1.0 * abs_thr
    reward -= 1.0 * abs_payload
    reward -= 0.01 * action_cost

    success = self._is_success()
    if success:
        reward += 50.0

    self.prev_total_err = total_err

    return reward
```

`progress` は、前ステップと比較してYaw誤差がどれだけ減少したかを表します。誤差が小さくなれば正の値、逆に目標から離れれば負の値になります。

さらに、現在のYaw誤差そのものを減点し、スラスター出力が大きい場合にも小さなペナルティを与えています。`Success` 条件を満たした場合は、追加で `50.0` の `報酬` を与えます。

:::message
**Reward設計について**

この制御モデルでは、まず目標Yaw角へ安定して到達できることを優先しました。

スラスター出力の大きさにもペナルティを与えていますが、その影響は小さく設定しています。そのため、ここではスラスター出力の効率や、より小さな出力による自然な制御までは主な評価対象としていません。

スラスター出力を抑えた制御については、可変目標Yaw制御の開発で改めて検討します。
:::

#### Success判定

目標Yaw角付近を一時的に通過しただけでは、制御に成功したとは判定しません。

スラスター機体と目標Yaw角の誤差、および吊り荷とスラスター機体のYaw角の相対誤差が、ともに `±5°` の範囲に入った状態を一定期間維持した場合に `Success` と判定します。

```python
SUCCESS_STEPS = 10

def _is_success(self):
    stable_now = (
        abs(self.thr_err) < math.radians(5.0)
        and abs(self.payload_err) < math.radians(5.0)
    )

    if stable_now:
        self.success_count += 1
    else:
        self.success_count = 0

    return self.success_count >= SUCCESS_STEPS
```
`SUCCESS_STEPS = 10` とし、条件を10回連続して満たした場合に `Success` と判定します。途中で条件から外れた場合はカウントを `0` に戻します。そのため、目標Yaw角付近を一時的に通過しただけでは成功とは判定されません。

以上の処理を `step()` 内で繰り返すことで、強化学習モデルの `Action` と PyBullet による物理シミュレーションを接続します。

## まとめ

今回は、吊り荷Yaw制御を強化学習で扱うための制御環境を構築しました。`観測値`、`Action`、`Reward`、`成功条件`を定義し、3種類の吊り荷と外乱条件を組み合わせて学習できる環境としています。

次回は、この環境を使用してSACによる制御モデルを学習し、学習結果と制御性能を評価します。

## シリーズ構成

本シリーズでは、吊り荷Yaw制御システムの開発過程を段階的に紹介します。

- [#01 制御環境を構築する](https://zenn.dev/rindajones/articles/a8b6189171848f)
- **#02-1 強化学習の環境を構築する ← 今回**
- [#02-2 強化学習による制御モデルを学習する](https://zenn.dev/rindajones/articles/payload-yaw-control-02_2)
- [#03 カメラ画像からYaw角を推定する](https://zenn.dev/rindajones/articles/yaw_estimation)
- [#04 Raspberry Piで画像ベース制御を実証する](https://zenn.dev/rindajones/articles/raspberry_pi)
- [#05 可変目標Yaw制御を目指す](https://zenn.dev/rindajones/articles/variable-target-yaw-control)
- [#06-1 可変目標Yaw制御の環境を構築する](https://zenn.dev/rindajones/articles/variable-target-env)
- [#06-2 可変目標Yaw制御モデルを学習する](https://zenn.dev/rindajones/articles/variable-target-training)
- #07 3種類の吊り荷で可変目標Yaw制御を試す
- #08 可変目標Yaw制御の結果と課題


## 関連リンク

プロジェクト全体は GitHub Pages で公開しています。
@[card](https://rindajones.github.io/payload-yaw-control/ja/)