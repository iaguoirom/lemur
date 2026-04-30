# １.解析の実行
```
import deeplabcut

# プロジェクトの設定ファイルと、解析したい動画のパスを定義する
config_path = r"C:\Users\...\config.yaml"
video_path = [r"C:\Users\...\対象の動画.mp4"]

# 解析の実行
# save_as_csv=True にしておくと、h5ファイルに加えてCSVも出力される
deeplabcut.analyze_videos(
    config_path, 
    video_path, 
    videotype='.mp4', 
    save_as_csv=True
)
```

## 2. 抽出された部位（Bodyparts）の特定

```
import pandas as pd

# 解析結果の .h5 ファイルを読み込む
h5_path = r"C:\Users\...\対象の動画DLC_resnet50_...h5"
df = pd.read_hdf(h5_path)

# 階層構造のデータからbodypartsの層だけを抜き出し、重複を消してリスト化する
bodyparts = df.columns.get_level_values('bodyparts').unique().tolist()

print("検出されたパーツ一覧:")
print(bodyparts)
```

- **意味:** DLCの出力データはAIの名前→パーツ名→x, y, 尤度という多重構造になっている。このコードは、その構造の中からパーツ名の階層だけを抽出できる。

## 3. 解析後の視覚化とプロット

数値データをグラフと動画に変換

```
# 1. 診断グラフの生成（尤度、軌跡、ヒストグラムなど）
deeplabcut.plot_trajectories(config_path, video_path)

# 2. 動画の生成（動画の上に点を重ねて出力）
# draw_skeleton=True にすると、設定した骨格の線も描画される
deeplabcut.create_labeled_video(
    config_path, 
    video_path, 
    draw_skeleton=True
)
```

---

## 4. プロット後に確認すべき項目

### ① `_labeled.mp4`（ラベル付き動画）
- **確認事項:** 点が対象物に追従しているか。特定の背景に点が吸い寄せられてないか。
- **判断基準:** 点が激しく点滅したり、画面の端へワープしている場合、尤度が低下している

### ② 4つのグラフの評価
- **plot-likelihood:** 全体的に0.6を下回るパーツがないか。
- **hist_filtered:** 右側に長い線（ありえない長距離移動）が発生していないか。

### ③ エラーフレームの抽出

もし解析結果が崩壊していた場合、AIが間違えた瞬間だけを抽出して、人間が正解を教え直すというアクティブラーニングに移行できる

```
# AIが迷ったフレームだけを自動で抜き出す
deeplabcut.extract_outlier_frames(config_path, video_path)
```

- **意義:** AI自身に自信がなかったシーンや点がワープしたシーンを申告させ、その画像をフォルダに抽出させる。この抽出された画像に対して手動で点を打ち直し、再学習（`deeplabcut.refine_labels` → `deeplabcut.train_network`）させることで精度をあげられる。
# 解析
# 1. データの読み込みと構造の特定

```
import pandas as pd

# HDF5形式の解析データを読み込む
df = pd.read_hdf(file_path) 

# 解析を実行したモデル名を取得
scorer = df.columns.get_level_values('scorer')[0]
```

- **意味**: DLCのデータは階層構造になっている。まず「誰が（scorer）」「どの部位を（bodyparts）」解析したのかを特定する。

## 2. 尤度（自信度）足切り

```
threshold = 0.6
nose_x = df[scorer]['nose']['x'].where(df[scorer]['nose']['likelihood'] >= threshold)
```

- **意味**: `.where()` 条件に合わないデータを消去（NaN化）する
- **意図**: AIが迷った座標をそのまま使うとありえないノイズを生むため、あらかじめ空欄として処理する

## 3. フィルタリング

```
# 中央値フィルタ：突発的なワープを無視する
clean_x = nose_x.rolling(window=5, center=True).median()

# 移動平均：細かい震えを均してトレンドを出す
smooth_x = clean_x.interpolate().rolling(window=10, center=True).mean()
```

- **`.rolling(window=n)`**: 直近 $n$ フレームのデータを確認する
- **`.median()`**: 多数決で中央の値をとる。1フレームだけ飛んだノイズを消す
- **`.interpolate()`**: `where` で消した空欄を、前後の値から「線」でつなぐ

## 4. 運動の算出

```
dx = nose_x.diff() # 前のフレームとの差分（移動量）
dy = nose_y.diff()
velocity = np.sqrt(dx**2 + dy**2) # ピタゴラスの定理で直線距離（速度）を算出
```

- **意味**: 座標 $(x, y)$ の変化量をベクトルとして合成し、スカラー量（速さ）に変換している。


