# LINE Creators Market「アニメーションスタンプ」制作 完全指示書

この文書だけで、LINE Creators Market向けの「動くスタンプ(APNG)」を規格エラーなしで作れる。

Claude Code固有のツール名は一切使っていない。ブラウザ操作・ターミナル実行・Python実行ができるAI(Grokを含む)であれば、この手順をそのまま実行すればよい。

過去に規格エラーの原因が分からず何度もアップロードに失敗し、丸1セッション分を無駄にした実体験から確定した仕様と落とし穴をまとめてある。

最初からこの手順通りに作れば、同じ往復は起きない。

---

## 0. 前提として必要なもの

- Python 3(ライブラリ: `Pillow`, `numpy`, `scipy`)
- `ffmpeg` / `ffprobe`(動画からのフレーム抽出に使う)
- 画像生成AI(動画モードがあるものが望ましい。Grok Imagine、または ChatGPT の画像生成)を操作できるブラウザアクセス
- ターミナルでコマンドを実行できる環境

インストール例:

```bash
pip install Pillow numpy scipy
brew install ffmpeg
```

---

## 1. なぜこの手順が重要か

LINEの公式ガイドライン(https://creator.line.me/ja/guideline/animationsticker/)には数値の上限は書いてある。

しかし、**アップロードが通るかどうかを実際に左右する条件の一部は、ガイドラインに書かれていない**。

エラーメッセージも「Error」としか出ず、赤いバッジをクリックして初めて理由が分かる。

ここで試行錯誤すると、1枚あたり何度もの往復が発生する。

だから、最初から「通ることが分かっている作り方」で作ることに価値がある。

---

## 2. 確定仕様(公式ガイドライン相当)

| 項目 | 条件 |
|---|---|
| 画像サイズ | 最大 横320×縦270px。縦か横のどちらかは必ず270px以上(両方270px未満はNG) |
| フレーム数 | 5〜20枚 |
| ファイルサイズ | 1枚あたり1MB以下(300KBではない) |
| メイン画像 | 240×240のAPNG(こちらもアニメーション) |
| トークルームタブ画像 | 96×74の静止PNG |
| 背景 | 透過必須 |
| カラー | RGBA。パレット化・インデックスカラーは避け、フルカラーで作る |
| 拡張子 | `.png`(APNGも`.png`のまま) |
| ZIP一括アップロード | `01.png`〜`NN.png`＋`main.png`＋`tab.png`をサブフォルダに入れずフラットにzip化。全体60MB以下 |

個数はLINE側の「スタンプ個数(1パッケージ)」プルダウン(8/16/24/32/40)のいずれかと一致させる。ファイル名は2桁ゼロ埋め(`01`〜)。

---

## 3. 実機検証でしか分からない落とし穴(ここが本題)

ガイドラインの数値を全部満たしていても、次の3つを踏むと「Error」表示になる。しかも理由は赤いバッジをクリックしないと出てこない。

### 3-1. 再生時間は1／2／3／4秒に"ぴったり"合わせる

「4秒以内」ではなく離散値の判定になっている。3.996秒や4.004秒のような端数は、数値上は「4秒以内」でも弾かれる。

**対策＝フレーム数は4000msを割り切れる枚数(5・8・10・16・20)から選び、1コマの長さを全コマ完全に同じ値(4000÷フレーム数)にする**。端数を一切作らない。

### 3-2. ループ回数は1〜4を明示する。0(無限ループ)は不可

さらに、**ループ回数×1コマ分の再生時間、の掛け算がLINE側の判定に影響している可能性が高い**。4秒のアニメを`loop=4`(ループ4回)で作ると「4秒×4回=16秒」と読まれて弾かれ続けた例がある。秒数だけ何度直しても変わらず、`loop=1`に変えて初めて通った。

**対策＝最初から`loop=1`で作る**。ループ回数の上限が4だからといって4を選ばない。

### 3-3. Pillow(PIL)は隣接する2コマが完全に同じ絵だと勝手に1コマへ統合する

これで「5枚のつもりが実は4枚で規格違反」という事故が起きる。原因は2パターンある。

- 2枚のキーフレームをA,B,A,B…と交互配置した後に枚数を間引くと、間引く間隔が元の交互周期(2)の倍数のとき、全コマが同じ絵になる(エイリアシング)。**整数比で割り切れる間引き方を避ける**。
- sin波などの周期関数でフレームを作るとき、フレーム数Nが周波数の係数の約数だと同じ理由で全コマ同一になる(例: `sin(10*ph)`をN=5やN=10でサンプリングすると常にゼロになる)。**Nは周波数の約数にしない**。

**対策＝保存する直前に、全ての隣接フレームのペアがpixel-identicalでないか必ず検証する**。1つでも一致していたら、そのペアの片方を作り直すか配置を変える。

### 3-4. コマごとに再生時間を変えるのはやめる(調査済みの誤ったリード)

「ためのあるお辞儀だから最初と最後のコマだけ長くしたい」のように、コマごとに再生時間を変えたくなることがある。

過去の制作でこれを試したところ(例: 666ms/670ms混在)、Pillowが保存時にPNGの`delay_num`/`delay_den`をコマごとに個別で約分するため、同じファイル内で分母がバラバラになる事象が発生した。

検証の結果、この分母のバラつき自体が直接のエラー原因ではないと分かっている。ただし、原因調査を難しくする余計な変数になるだけで得るものがない。

**最初から全コマ同じ長さ(4000÷フレーム数)にして、この変数自体を作らないこと**。動きに緩急をつけたいときは、再生時間ではなくコマの中身(ポーズの変化量)側で表現する。

---

## 4. 動きを作る前に: まず絵コンテで動作を設計する(ここが一番重要)

最初に「1枚の絵を回転・平行移動させるだけ」の安全な作り方で全部作ってしまい、あとから「体の形が同じまま動かそうとしているから動作していない」と気づいて全部作り直しになった失敗がある。

**この手戻りが一番コストが高い。最初からここを踏むこと**。

### 判定基準: その動きは「回転・平行移動」で正直に表現できるか

1枚の絵を変形させずに動かす技法(回転・平行移動・拡大縮小)は、**その動きが物理的に本当に剛体運動であるときだけ**正直な表現になる。当てはまるのはこの程度。

- 呼吸のような、体全体のわずかな上下(breathing bob)
- その場での回転(目眩・スピン)
- 震え(shiver、小刻みな左右振動)
- しっぽなど、絵から自然に切り離せる部位単体の回転

**それ以外の動き――歩く・手を振る・伸びをする・舐める・跳ねる・転がる・お辞儀する等――は、体のどこかが元の絵と違う形・違う位置を取る必要がある**。これを1枚の絵の回転・平行移動だけで作ると、関節が伸びない/曲がらない/継ぎ目に隙間ができるなど、見た瞬間に不自然だと分かる仕上がりになる。

### 手順

1. **作る予定のポーズ全部を先に一覧にする**。複数の動きを作るときは、最初に「①=歩く」「②=手を振る」のように動作の一覧表を作ってから着手する(あとから重複や被りに気づいて作り直すのを防ぐ)
2. **開始ポーズ(1コマ目)も全スタンプで変える**。ベース絵1枚を全スタンプの動画生成の参照画像に使い回すと、セット全部が同じ構図から動き出して単調に見える。一覧表には動作の列に加えて「開始ポーズ」の列を作り、寝ている・後ろ姿・完全横向き・顔アップ・しゃがみ・小道具つき(布団・草・温泉)のように、構図と姿勢の段階から全部ばらす。制作は各スタンプ2段構えにする。**まず画像生成モードでベース絵から開始ポーズの静止画を描き起こし、その静止画を動画生成モードの参照画像にして動かす**。ベース絵と同じ構図でよいのはセット全体で1個まで
3. 一覧の動きひとつひとつに、上の判定基準を当てはめる。**「回転・平行移動で正直に表現できる」と自信を持って言えないものは、全部「新しいポーズが要る」側に倒す**。安全そうだからという理由で剛体変換を選ばない
4. 新しいポーズが要ると判定したものは、下の「5. 制作手順」でキーフレームを描き起こしてから、2枚(以上)を切り替えるアニメーションとして組む
5. 剛体変換で足りると判定したものだけ、1枚の絵の回転・平行移動で作る
6. 全部作り終えたら、**「この動きは実際にその動作をしているように見えるか」を1枚ずつ自問する**。「なんとなく動いてはいる」で妥協せず、疑わしいものは4に戻って新しいポーズを足す

---

## 5. 制作手順(基本方針: 動画を1本作ってコマを抜き出す)

静止画を1枚ずつ描いてもらう方式は、動きの滑らかさが「実際に描いた枚数」止まりで頭打ちになり、生成・ダウンロード・検証のサイクルも多くコストがかかる。

**動画生成モードを使えば、1回の生成依頼で静止画4〜8枚分に相当する滑らかさが手に入る。新規にセットを作るときは、最初からこの動画方式で着手する**。(個別ポーズを1枚だけ直したいときの手段としては、6章の静止画方式を使う)

### 5-1. 参照画像を用意する

3章の絵コンテ手順2の通り、スタンプごとに描き起こした「開始ポーズ」の静止画を用意する。動画の1コマ目はこの参照画像の構図になるため、ここを変えないとセット全部が同じ構図から始まって単調になる。

### 5-2. 動画を生成する

1. 画像生成AI(Grok Imagineなど)を開き、参照画像を貼り付ける(アップロード機能、またはクリップボード経由の貼り付け)
2. 「画像」モードから「動画」モードに切り替える
3. 依頼文の例: 「このキャラクターの絵柄を一切変えずに、このキャラクターが[動作]するアニメーション。背景は白のままで、カメラは固定。」
4. 生成が終わるまで待つ(数十秒〜数分。短い間隔で何度も確認する必要はなく、10秒程度の待機を2〜3回で足りる)
5. 生成結果を見て「動きが小さい・弱い」と感じたら、**曖昧な「もっとはっきり動かして」ではなく、具体的な変化量や状態を名指しして再依頼する**。「あご先が胸につくくらい」「四本の足全部が地面から完全に離れて体の下に空白ができるように」のように、どこがどこまで動くべきかを言葉にすると一発で決まりやすい

### 5-3. 動画をダウンロードする

- ブラウザのネイティブ「ダウンロード」ボタンは不安定なことが多い(クリックしても保存先に実体が保存されない場合がある)。**「シェア」ボタンから公開リンク(共有URL)を取得し、そのURLを直接`curl`等で取得する方式が確実**。
- 生成結果を採用する前に、ffmpegで2〜3コマ抜き出して目視確認する(共有アカウントを使っている場合、無関係な動画が紛れ込む事故があるため)

```bash
curl -L -o generated.mp4 "<共有経由で取得した動画の直URL>"
ffmpeg -i generated.mp4 -vf "select='eq(n\,0)+eq(n\,30)+eq(n\,60)'" -vsync 0 check_%02d.png
```

### 5-4. 動画からAPNGを作る(スクリプト全文)

以下をそのまま`build_from_video.py`として保存して使う。次の処理をすべて自動で行う。

- 動画を高密度でサンプリングし、フレーム間の動き量(画素差分)を測って、動きが薄い区間を間引き、動いている区間にコマ予算を集中させる(単純に時間で均等抜き出しすると、「間」の区間もコマに含まれ、体感で動きが遅く感じる仕上がりになる。これを避けるための必須ステップ)
- 背景透過(外周と連結した白領域だけをフラッドフィルで透過にする。輪郭内部の白は透過しない)
- 全コマの合成bboxで揃えてクロップ・スケール(270px条件を満たす)
- 隣接コマの重複チェックと自動補正
- 1MBを超えたらフレーム数を20→16→10→8→5の順に自動で下げて保存

```python
#!/usr/bin/env python3
"""動画からLINEスタンプ用APNGを作る(動き量に応じたコマ選択)。

使い方:
    python3 build_from_video.py <動画パス> <出力先NN.png>
"""
import sys
import subprocess
import tempfile
import os
import argparse
import numpy as np
from PIL import Image
from scipy import ndimage


def get_duration(video_path):
    return float(subprocess.run(
        ["ffprobe", "-v", "quiet", "-show_entries", "format=duration",
         "-of", "csv=p=0", video_path], capture_output=True, text=True
    ).stdout.strip())


def extract_dense_frames(video_path, dense_fps, workdir):
    out_pattern = os.path.join(workdir, "dense_%04d.png")
    subprocess.run(
        ["ffmpeg", "-y", "-i", video_path, "-vf", f"fps={dense_fps}", out_pattern],
        check=True, capture_output=True
    )
    files = sorted(f for f in os.listdir(workdir) if f.startswith("dense_"))
    return [os.path.join(workdir, f) for f in files]


def motion_weighted_indices(dense_paths, n_target):
    """密なフレーム列から、動き量の累積が均等になるようにn_target枚選ぶ"""
    grays = [np.asarray(Image.open(p).convert("L")).astype(float) for p in dense_paths]
    diffs = [0.0]
    for i in range(1, len(grays)):
        diffs.append(float(np.abs(grays[i] - grays[i-1]).mean()))
    cum = np.cumsum(diffs)
    if cum[-1] == 0:
        # 動きが検出できない場合は均等割りにフォールバック
        return list(np.linspace(0, len(dense_paths) - 1, n_target).astype(int))
    targets = np.linspace(0, cum[-1], n_target)
    indices = []
    for t in targets:
        idx = int(np.searchsorted(cum, t))
        idx = min(idx, len(dense_paths) - 1)
        indices.append(idx)
    # 重複除去(近すぎる場合は少しずらす)
    for i in range(1, len(indices)):
        if indices[i] <= indices[i-1]:
            indices[i] = min(indices[i-1] + 1, len(dense_paths) - 1)
    return indices


def remove_white_bg(im):
    arr = np.array(im)
    rgb = arr[:, :, :3].astype(int)
    white_mask = np.all(rgb > 240, axis=-1)
    labeled, _ = ndimage.label(white_mask)
    border_labels = set(labeled[0, :]) | set(labeled[-1, :]) | set(labeled[:, 0]) | set(labeled[:, -1])
    border_labels.discard(0)
    bg_mask = np.isin(labeled, list(border_labels))
    alpha = arr[:, :, 3].copy()
    alpha[bg_mask] = 0
    arr[:, :, 3] = alpha
    return Image.fromarray(arr, "RGBA")


def bbox_of(im):
    arr = np.array(im)
    ys, xs = np.where(arr[:, :, 3] > 10)
    if len(xs) == 0:
        return 0, 0, im.width, im.height
    return int(xs.min()), int(ys.min()), int(xs.max()), int(ys.max())


def align_and_crop(frames, pad=20):
    boxes = [bbox_of(f) for f in frames]
    x0 = min(b[0] for b in boxes) - pad
    y0 = min(b[1] for b in boxes) - pad
    x1 = max(b[2] for b in boxes) + pad
    y1 = max(b[3] for b in boxes) + pad
    x0, y0 = max(0, x0), max(0, y0)
    cropped = [f.crop((x0, y0, x1, y1)) for f in frames]
    w, h = cropped[0].size
    scale = min(320 / w, 270 / h)
    new_size = (max(1, int(w * scale)), max(1, int(h * scale)))
    return [f.resize(new_size, Image.LANCZOS) for f in cropped]


def dedupe_adjacent(frames):
    arrs = [np.array(f) for f in frames]
    for i in range(len(arrs) - 1):
        if np.array_equal(arrs[i], arrs[i + 1]):
            arr = arrs[i + 1].copy()
            arr[0, 0, 0] = (int(arr[0, 0, 0]) + 1) % 256
            arrs[i + 1] = arr
            frames[i + 1] = Image.fromarray(arr, "RGBA")
    return frames


def save_sticker(frames, out_path):
    n = len(frames)
    assert n in (5, 8, 10, 16, 20)
    duration = 4000 // n
    frames[0].save(out_path, save_all=True, append_images=frames[1:],
                    duration=duration, loop=1, disposal=0, blend=0, compress_level=9)
    return os.path.getsize(out_path)


def build(video_path, out_path, target_frames=None):
    candidates = [target_frames] if target_frames else [20, 16, 10, 8, 5]
    dur = get_duration(video_path)
    dense_fps = 12  # 密なサンプリング(6秒なら約72枚)
    with tempfile.TemporaryDirectory() as workdir:
        dense_paths = extract_dense_frames(video_path, dense_fps, workdir)
        for n in candidates:
            idxs = motion_weighted_indices(dense_paths, n)
            raw_frames = [Image.open(dense_paths[i]).convert("RGBA") for i in idxs]
            clean = [remove_white_bg(f) for f in raw_frames]
            aligned = align_and_crop(clean)
            aligned = dedupe_adjacent(aligned)
            size = save_sticker(aligned, out_path)
            if size <= 1024 * 1024:
                print(f"OK: {n}フレーム(動き量ベース選択), {size/1024:.0f}KB -> {out_path}")
                return n, size
            print(f"{n}フレームは{size/1024:.0f}KBで1MB超過。フレーム数を落とします。")
    raise RuntimeError("5フレームでも1MBを超えました")


if __name__ == "__main__":
    p = argparse.ArgumentParser()
    p.add_argument("video")
    p.add_argument("out")
    p.add_argument("--target-frames", type=int, default=None)
    args = p.parse_args()
    build(args.video, args.out, args.target_frames)
```

実行例:

```bash
python3 build_from_video.py generated.mp4 01.png
```

背景がキャラクター本体の絵柄内にも白い部分がある場合(例: 白目・白い毛)は、`remove_white_bg`が外周と連結した白領域だけを消す設計なので誤って透過しない。ただし念のため、生成前の動画は必ず背景白一色で依頼すること。

**背景透過はこの`remove_white_bg()`で統一し、`rembg`のような汎用の背景除去ツールに差し替えないこと**。汎用ツールに差し替えると、輪郭にごく薄い影(ハロー)が残ることがあり、他のコマや他のスタンプと並べたときに質感だけ浮いて見える。

### 5-5. 「もっと速く」と言われたときの直し方

コマの中身を作り直す必要はない。フレーム数はどれも1000/2000/3000/4000msのすべてを割り切れる(5・8・10・16・20は2000msも割り切れる)。`save_sticker`関数内の`duration = 4000 // n`を`2000 // n`に変えて保存し直すだけで、同じコマ内容のまま再生速度をきっちり2倍にできる。動画の再生成は不要。

---

## 6. 静止画方式(1匹の途中経過ポーズを1枚だけ直したいときの補助手段)

新しくポーズを1枚だけ描き起こしたいとき(体の輪郭自体が変わる動き=歩く・手を振る・伸びをする等)は、この手順を使う。

1. 元の絵を画像生成AIに画像として渡す
2. 「この画像とまったく同じ絵柄・線の太さ・配色・同じキャラクターの、[動作]を、[変えたい部分だけを具体的に]変えたポーズだけ新しく1枚描いてください」と指定する
   - 悪い例:「手を振っているポーズを描いて」(絵柄がズレやすい)
   - 良い例:「今の画像は右前足を高く上げています。新しい絵ではその前足を下ろして、体の横あたりの低い位置にしてください」(変える部位を名指しする方が一発で決まりやすい)
3. 背景は白一色、スタイルは元画像と完全に同じに揃えるよう毎回明記する
4. 出力を元の絵の足元位置(接地面)に合わせて拡大縮小・位置合わせしてから、2枚(以上)を切り替えるアニメーションとして組む
5. 生成が一発で決まらないときは、「まだ○○のままです。○○を完全に△△にしてください」と、変わっていない部分を名指しして再依頼する(曖昧な「もっとはっきり」だと変化が小さいまま終わることがある)

**静止画のダウンロードも、ブラウザのネイティブ「ダウンロード」ボタンは不安定(クリックしても実体が保存されないことがある)**。生成画像の「共有」ボタンから公開リンクを取得し、それを直接`curl`で取得する方式が確実。Grokの場合は共有リンクが`https://pbs.twimg.com/grok-img-share/{mediaId}.jpg`のような形式になる。

```bash
curl -L -o pose_02.jpg "<共有ボタンから取得した公開URL>"
```

ダウンロード後は必ず画像を開いて中身と解像度を確認する。共有アカウントを使っている場合、無関係な画像が紛れ込む事故があるため、低解像度・別の絵柄のまま採用しない。

---

## 7. コマ数を増やしたいとき、アルファブレンド(クロスフェード)は絶対に使わない

「動きが荒い・コマ送りに見える」という理由でコマ数を増やしたくなっても、**絵柄の違う2枚のポーズ画像を単純にアルファブレンド(クロスフェード)で混ぜて中間コマを作ってはいけない**。2つの絵の輪郭がずれて重なった部分が透けて見える「残像(ゴースト、二重写り)」になる。1枚の絵柄が別の絵柄に溶けて重なった状態は、実際にはどちらのポーズにもなっていない、誰が見てもおかしいと分かる仕上がりになる。

コマ数を増やしたいときは、**実際に別の中間ポーズを画像生成AIに描き起こしてもらい、硬い切り替え(ブレンドなし)だけでコマを構成する**。絵コンテ設計のときと同じで、「行って戻る」動きなら中間ポーズを1枚足してA,B,C,B,Aのように、実在するポーズ同士を切り替える。

動画由来のコマは、線画の抜き出しと違ってわずかな動きブレが乗ることがある。次章のチェックにごく僅差でひっかかることがあるが、該当フレームを目視して、二重写りでなく単一の自然なポーズであれば、誤検知として扱ってよい。

---

## 8. 保存前チェック(そのまま使える。全チェックが空リストになってから次へ進む)

```python
from PIL import Image
import numpy as np
import os


def validate_line_sticker(path, is_main=False):
    """規格チェック。空リストが返れば規格クリア。"""
    im = Image.open(path)
    n = im.n_frames if getattr(im, 'is_animated', False) else 1
    frames, durs = [], []
    for i in range(n):
        im.seek(i)
        frames.append(np.asarray(im.convert('RGBA')))
        durs.append(im.info.get('duration', 0))
    total = sum(durs)
    w, h = im.size
    maxw, maxh = (240, 240) if is_main else (320, 270)

    problems = []
    if not (w <= maxw and h <= maxh):
        problems.append(f'サイズ超過 {w}x{h}')
    if not is_main and not (w >= 270 or h >= 270):
        problems.append(f'縦横どちらも270px未満 {w}x{h}')
    if not is_main and not (5 <= n <= 20):
        problems.append(f'フレーム数が範囲外 {n}')
    if total not in (1000, 2000, 3000, 4000):
        problems.append(f'再生時間が1/2/3/4秒ぴったりでない {total}ms')
    if len(set(durs)) > 1:
        problems.append(f'コマの長さが不揃い {durs}')
    for i in range(len(frames) - 1):
        if np.array_equal(frames[i], frames[i + 1]):
            problems.append(f'隣接フレーム{i},{i+1}が同一(保存時に統合される)')
    kb = os.path.getsize(path) / 1024
    if kb > 1024:
        problems.append(f'ファイルサイズ超過 {kb:.0f}KB')
    return problems


def check_ghosting(path, threshold=0.08):
    """残像(クロスフェードの混ざり)の疑いを検出する。
    非透明ピクセルのうち、中間アルファ値(25〜230)を持つピクセルの比率が
    しきい値を超えたら警告する。"""
    im = Image.open(path)
    n = im.n_frames if getattr(im, 'is_animated', False) else 1
    warnings = []
    for i in range(n):
        im.seek(i)
        arr = np.asarray(im.convert('RGBA'))
        alpha = arr[:, :, 3]
        opaque = int(np.sum(alpha > 230))
        mid = int(np.sum((alpha >= 25) & (alpha <= 230)))
        if opaque > 0 and mid / opaque > threshold:
            warnings.append(f'フレーム{i}: 中間アルファ比率{mid/opaque:.1%}(残像の疑い。目視で確認)')
    return warnings


# 使い方:
# validate_line_sticker('01.png')  が空リストを返せば規格クリア
# validate_line_sticker('main.png', is_main=True)  メイン画像用
# check_ghosting('01.png')  空リスト、または目視確認して問題なければ無視してよい
```

### 8-1. 保存コード(1枚の絵を回転・平行移動だけで作る場合)

4章の判定基準で「剛体変換で足りる」と判定した動き(呼吸・回転・震え・しっぽ単体の回転など)は、動画生成を経由せず、コードで直接フレームを作ってよい。保存時は次の関数をそのまま使う。

```python
def save_line_sticker(frames, path, loop=1):
    """frames: 同じ長さのPIL RGBA Imageのリスト。
    長さは4000msを割り切れる数(5・8・10・16・20)にすること。"""
    n = len(frames)
    assert n in (5, 8, 10, 16, 20), f'フレーム数{n}は4000msを割り切れない。5/8/10/16/20枚にする'
    duration = 4000 // n
    frames[0].save(path, save_all=True, append_images=frames[1:],
                    duration=duration, loop=loop, disposal=0, blend=0, compress_level=9)
```

フレームを作る側(回転角度やずらす距離を計算する部分)は動きの内容によって変わるので、ここでは共通化していない。作ったフレームの並びは保存前に必ず8章の`validate_line_sticker()`と`check_ghosting()`を通す。

1枚(main.png、tab.pngを含む全ファイル)ずつ`validate_line_sticker()`を実行し、空リストになることを確認してから次に進む。`tab.png`は静止画(96×74)なので、このチェック対象には含めなくてよい。サイズだけ手動で確認する。

---

## 9. ZIP一括アップロード用の構成

```
01.png, 02.png, ... NN.png, main.png, tab.png
```

サブフォルダに入れず、上記をフラットに1つのzipにまとめる。個数はLINE側の「スタンプ個数(1パッケージ)」プルダウン(8/16/24/32/40)と一致させる。ファイル名は必ず2桁ゼロ埋め(`01`〜)。zip全体は60MB以下にする。

```bash
zip -j stickers.zip 01.png 02.png 03.png ... main.png tab.png
```

(`-j`でパス情報を捨ててフラットに格納する)

---

## 10. 複数キャラクター・複数ポーズを並行して作るときの注意

- 複数の作業(複数キャラクター、複数ポーズ)を同時並行で進めると、同じ画像生成アカウント・同じダウンロードフォルダを取り違える事故がほぼ毎回起きる。**ダウンロード直後に画像の中身と解像度を必ず確認してから採用する**ことを毎回徹底する
- 画像生成AIの1日の生成上限に達することがある。ChatGPTが使えなければGrok、Grokが使えなければChatGPTのように、同等品質の代替を使ってよい(絵柄再現の精度は同程度)
- 新しいポーズの描き起こしは、並列化してもアカウント上限やタブ混線で早々に止まりやすく、効果が薄い作業だと割り切る。並列で試して事故が増えるようなら、早めに1つずつの逐次処理に切り替える
- ダウンロード後は解像度が元画像(1000px級)相当あることを必ず確認し、小さければ失敗として取得し直す。低解像度のまま妥協しない

---

## 11. この手順でできないこと(人間が行う部分)

`creator.line.me` はブラウザ自動化ツールから安全上の制限で直接操作できない場合がある。その場合は次を人間が行う。

- LINE Creators Marketへの実際のログイン・画像アップロード・「審査をリクエスト」ボタンのクリック
- できるのは「規格に通るファイル一式(ZIP・個別APNG・main・tab)を作って渡す」ところまで

アップロード後にエラーが出た場合は、**赤い「Error」バッジをクリックして出てくる具体的なメッセージを必ず確認する**。「Error」としか表示されていない段階では原因を特定できないので、憶測で直さない。

---

## 12. 作業の通し手順(まとめ)

1. 作るポーズ全部を一覧表にする(動作・開始ポーズの列を必ず作る。4章参照)
2. 各ポーズについて「回転・平行移動で足りるか、新しいポーズが要るか」を判定する
3. 新しいポーズが要るものは、開始ポーズの静止画を作ってから動画生成→`build_from_video.py`でAPNG化(5章)
4. 剛体変換で足りるものは、1枚の絵をコードで回転・平行移動させてAPNG化する(3章の対策を踏まえて`save_sticker`相当の関数で保存)
5. 全ファイルに`validate_line_sticker()`と`check_ghosting()`を実行し、問題ゼロを確認する
6. `main.png`(240×240、アニメーション)と`tab.png`(96×74、静止画)を作る
7. `01.png`〜`NN.png`+`main.png`+`tab.png`をフラットにzip化する
8. zipをユーザーに渡し、LINE Creators Marketへのアップロードと審査リクエストはユーザー自身に行ってもらう
