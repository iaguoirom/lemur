# 1. 基礎情報（メタデータ）
この部分は、システムが「誰の、何のプロジェクトか」を認識するための単なるラベルで解析結果には影響しない

```
Task: Macaquehashiru
scorer: Akane
date: Apr15

# 複数匹を同時に追跡するかどうか（今回は単一の猿なのでfalse）
multianimalproject: false 
identity:
project_path: C:\Users\akane\OneDrive\Macaquehashiru-Akane-2026-04-15
```

## 2. 解析対象と抽出ルール

AIが「どこを見るべきか」を定義する

```
video_sets:
  C:\Users\...\147085_1280x720.mp4:
    crop: 0, 1280, 0, 720  # 動画のどの範囲を解析するか（現在は全画面）
bodyparts:
- nose
- left_eye
... (計17パーツ)
```

- **意味**: ここに列挙された名前が、出力されるCSVや `.h5` ファイルの列の名前に直結

- **変数**:
```
start: 0
stop: 1
numframes2pick: 20
```
将来手動での再学習を行う際の設定
動画全体の中から、AIが自動で20枚の画像（フレーム）をランダムに抽出し、人間にここに点を打って正解を教えてくれと要求してくる

## 3. 視覚化（プロット）のルール

グラフや動画を出力する際の見た目の設定

```
skeleton:
- - bodypart1
  - bodypart2
- - objectA
  - bodypart3
skeleton_color: black
pcutoff: 0.6
dotsize: 6
alphavalue: 0.7
colormap: rainbow
```

- **`pcutoff: 0.6`**: AI足切りライン。 尤度が0.6未満の点は、動画上にプロットしないという初期設定になっている
- **`skeleton:`**: 現在、デフォルトのダミー文字列（`bodypart1`など）が入っているため機能していない。もし骨格の線を動画に描画したい場合は、ここを以下のように繋ぎたいパーツのペアに書き換えることで、点と点の間に線が引かれる。

```
skeleton:
 - - left_shoulder
 - left_elbow
 - - left_elbow
 - left_wrist
```

## 4. 脳（ニューラルネットワーク）の構造
AIの学習と推論に関するアルゴリズムの設定
```
TrainingFraction:
- 0.95
iteration: 0
default_net_type: resnet_50
default_augmenter: imgaug
snapshotindex: -1
batch_size: 8
```

- **`resnet_50`**: 画像認識における視覚野のモデル構造、50層の深さを持つネットワーク
- **`batch_size: 8`**: 1回の計算で何枚の画像を同時に処理するかという設定、現在、CPUのみの演算環境で動かしているため、この数値をこれ以上大きくするとメモリの処理限界を超えてシステムがクラッシュする可能性がある
## 5. その他
変えるとしたらここ
1. **`pcutoff`** (プロット動画の足切りラインを厳しくするか、甘くするか)
2. **`skeleton`** (点だけでなく、関節を線で繋いで視覚化したい場合)
