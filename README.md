# Project MYXOMYCETES: Bio-Mimetic Autonomous Logistics Simulator

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)
![Version](https://img.shields.io/badge/version-2.0.0-blue.svg)
![Technology](https://img.shields.io/badge/tech-Three.js%20%7C%20WebGL%20%7C%20Reaction--Diffusion-orange.svg)

> **[ライブデモを起動する (Live Demo)](https://funmatu.github.io/Myxomycetes-Swarm-Logistics/)**  
> *※ クリックしてブラウザ上で即座にシミュレーションを実行できます。*

---

## 📖 概要 (Overview)

**Project MYXOMYCETES (ミクソミセテス)** は、真正粘菌（*Physarum polycephalum*）の生物学的知能を模倣した、**完全自律分散型の物流搬送システム**シミュレーターです。

従来の中央集権型制御（A*探索や中央サーバーによる交通整理）を一切使用せず、**反応拡散系（Reaction-Diffusion System）** と **多層スカラー場（Multi-Layer Scalar Fields）** の相互作用のみによって、数百台のAGV（無人搬送車）が以下の高度な振る舞いを創発します。

1.  **最短経路の探索と最適化**: 障害物を回避し、最適なルートを自己発見。
2.  **動的な衝突回避**: 互いにフェロモンを出し合い、流体のようにぶつからずにすれ違う。
3.  **群知能による交通制御**: 渋滞が発生すると、後続車が自動的に迂回路を開拓する（ロードバランシング）。
4.  **環境記憶（Stigmergy）**: 成功した経路が「太い管（Vein）」として環境に刻まれ、システム全体の効率が時間とともに向上する。

本プロジェクトは、計算機科学、非線形物理学、生物学の境界領域における「群知能エンジニアリング」の実証実験です。

---

## 🧠 理論的背景とアーキテクチャ (Theoretical Architecture)

本システムの中核は、**「環境こそが計算機である（The Environment is the Calculator）」** という思想に基づいています。エージェント（AGV）は地図を持たず、環境（フィールド）に情報を書き込み、環境から情報を読み取ることで意思決定を行います。

### 1. 多層ホログラフィック・フィールド (Multi-Layer Holographic Field)

シミュレーション空間は、異なる意味を持つ複数のスカラー場（グリッドデータ）が重なり合ったものです。各場は以下の偏微分方程式（反応拡散方程式）に従って時間発展します。

$$ \frac{\partial \phi}{\partial t} = D \nabla^2 \phi - \gamma \phi + S(\mathbf{x}, t) $$

ここで、$\phi$ は場の濃度、$D$ は拡散係数、$\gamma$ は減衰率、$S$ はエージェントによる書き込み（ソース項）です。

| チャンネル (Layer) | 色 (Vis) | 役割 (Function) | 生物学的アナロジー |
|:---:|:---:|:---|:---|
| **Pickup Potential** | 🔴 Red | 集荷場からの匂い。「荷物を取りに来い」という誘引信号。 | 食料の匂い |
| **Delivery Potential** | 🟢 Green | 配送先からの匂い。「ここに届けろ」という誘引信号。 | 巣のフェロモン |
| **Repulsion Field** | 🔵 Blue | 他のAGVや壁からの拒絶信号。衝突回避と渋滞緩和を担う。 | 個体間の斥力 / 毒素 |
| **Vein Memory** | 🟡 Yellow | 配送に成功した経路の記憶。時間の経過とともに強化される。 | 原形質流動の管 |

### 2. エージェントの意思決定モデル (Agent Sensory-Motor Loop)

各AGVは**有限オートマトン（FSM）**を持つ自律エージェントです。AGVは自身の前方3点（左前、正面、右前）のフィールド値をサンプリングし、以下の論理で操舵角 $\theta$ を決定します。

#### センサー方程式
エージェントが位置 $\mathbf{x}$ で感じるポテンシャル $V(\mathbf{x})$ は、現在の目的（Pickup or Delivery）と、回避すべき障害物の線形結合です。

$$ V(\mathbf{x}) = \alpha \cdot \phi_{target}(\mathbf{x}) + \beta \cdot \phi_{vein}(\mathbf{x}) - \omega \cdot \phi_{repulsion}(\mathbf{x}) $$

*   $\phi_{target}$: 目的地へのフェロモン（赤または緑）
*   $\phi_{vein}$: 過去の成功体験（黄色）。「みんなが通っている道」を優先するバイアスとして働く。
*   $\phi_{repulsion}$: 障害物や他者（青）。これを強く避けることで衝突を防ぐ。

#### 操舵ロジック

$$ \Delta \theta \propto \text{argmax}(V_{left}, V_{front}, V_{right}) $$

最もポテンシャルの高い方向へ旋回します。これにより、 **「目的地に向かいつつ」「過去の主要街道を通り」「混雑していない場所を選ぶ」** という複雑な経路選択が一瞬で行われます。

---

## 🛠 技術的実装詳細 (Implementation Details)

### 離散ラプラシアンによる拡散近似
Javascript単一スレッドでの高速動作を実現するため、拡散方程式の $\nabla^2 \phi$（ラプラシアン）を5点ステンシルを用いた差分法で高速に計算しています。

```javascript
// 拡散カーネルの適用（擬似コード）
laplacian = P[y][x-1] + P[y][x+1] + P[y-1][x] + P[y+1][x] - 4*P[y][x];
next_P = (P[y][x] + diffusion_rate * laplacian) * (1.0 - decay_rate);
```

### スタイグマジー（Stigmergy）の実装
エージェント間の直接通信は一切ありません。
1.  AGV A が移動する → その座標の `Repulsion` 場に値を加算。
2.  その値が拡散する。
3.  AGV B がその「勾配」を感じ取り、方向を変える。
この間接通信により、数百台規模でも計算量は $O(N)$ ではなく、フィールドサイズ依存の定数時間（グリッド更新）＋ エージェント数 $N$ に収まり、極めてスケーラブルです。

### GPUアクセラレーションによる可視化
Three.jsの `ShaderMaterial` を使用し、計算された数値データ（`Float32Array`）を `DataTexture` としてGPUに転送。Vertex Shaderで場の濃度を「地形の高さ（Z座標）」に変換し、Fragment Shaderで多層情報を色合成しています。これにより、システムの内部状態を直感的にモニタリング可能です。

---

## 🎮 使用方法とパラメータ (Usage & Controls)

画面右上のGUIパネルで、リアルタイムに実験パラメータを変更可能です。

### Swarm Configuration
*   **AGV Count**: エージェントの数を10〜200の間で増減させます。密度が高まると、より複雑な迂回行動（創発）が観察されます。
*   **Diffusion**: フェロモンの広がる速度。高いと遠くまで情報が届きますが、経路がぼやけます。
*   **Decay**: フェロモンの消える速度。高いと「今」の情報だけが頼りになり、低いと「過去」の情報が長く残ります。

### Environment Control
*   **Randomize Obstacles**: 障害物の配置をランダムに再生成します。AGVは即座に新しい地形に適応し始めます。
*   **Reset Statistics**: スループット計測をリセットします。

### インジケーターの見方
*   **赤 (Red)**: 集荷エリアからの誘引信号。
*   **緑 (Green)**: 配送先からの誘引信号。
*   **青 (Blue)**: 壁や他のAGVがいる場所（立ち入り禁止区域）。
*   **黄 (Yellow)**: よく使われる経路（自動生成された高速道路）。
*   **AGVの色**:
    *   Cyan: 集荷に向かっている (Empty)
    *   Yellow: 配送に向かっている (Loaded)
    *   Magenta: 荷降ろし中/待機中

---

## 🧪 実験と観察のヒント (Experiments)

本シミュレーターで以下の現象を確認してみてください。

1.  **経路の形成**: 初期はランダムに動きますが、1分ほど経つと「黄色い道」ができ、AGVが一列に並んで走行し始めます。
2.  **自己修復**: `Randomize Obstacles` で突然壁を作ると、アリの行列が崩れるように混乱しますが、すぐに新しい迂回路を見つけ出し、流れが再開します。
3.  **対向分離**: 狭い通路では、自然と「右側通行」や「左側通行」のような分離流が発生することがあります（群密度が高い場合）。

---

## 📁 ファイル構成

```text
.
├── index.html       # シミュレーション本体（HTML/JS/CSS完結型）
├── README.md        # 本ドキュメント
└── LICENSE          # MITライセンス
```

---

## 🚀 今後の展望 (Future Roadmap)

*   **動的なタスク割り当て**: 複数の集荷場・配送先がある場合の最適ポテンシャル配分。
*   **3次元空間への拡張**: ドローン群制御への応用。
*   **強化学習の導入**: パラメータ（拡散率やセンサー感度）自体をAGVが学習するメタ最適化。

---

## 📜 ライセンス (License)

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

Developed as a Proof of Concept for Decentralized Swarm Intelligence
