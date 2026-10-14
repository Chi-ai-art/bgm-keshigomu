# BGMけしゴム

動画生成AIで作った動画に勝手についたBGMを取り除くブラウザツール(HTML1枚・ビルド不要)。
処理はすべてブラウザ内で行い、ファイルはどこにも送信しない。運営費0円の構成。

- 公開URL: https://chi-ai-art.github.io/bgm-keshigomu/ (2026-10-14 公開。スマホでも使える)
- リポジトリ: `Chi-ai-art/bgm-keshigomu`(**Public**。秘密は入っていない)。GitHub Pages、main への push で自動デプロイ
- ローカル確認: `.claude/launch.json` の `bgm-remover`(ポート8770)

## 残す音のモード
- **声だけ残す**: vocals だけ。BGM = drums + bass + other
- **声と効果音を残す**: vocals + drums + other から取り出したこすれ・打撃の音(HPSS、メディアン17)。そのあとピークに合わせて音量を上げる(最大10倍)。鉛筆の音・足音などはここで残る
- **仕上げ(両モード共通)**: 残す音とBGMの推定をSTFTで比べ、BGMが強い部分を小さくする(ウィーナー風マスク `k²/(k²+α·b²)`)
- **消す強さ**: ふつう = HPSSのmargin 2・α 1、強め = margin 3・α 4。テスト動画の「BGMだけ鳴っている1秒」で、残ったBGMは 仕上げなし -35dB → ふつう -54dB → 強め -64dB

## AIモデルの使い分け(スマホはメモリ不足でページが落ちるため)
| 名前 | ファイル | 大きさ | 使う場面 | 元のモデルとの差 |
|---|---|---|---|---|
| full | Hugging Face の htdemucs_embedded.onnx | 180MB | パソコン | — |
| fp16 | `models/htdemucs_fp16.onnx`(全部16bit) | 92MB | GPUが使えるスマホ | 出力の差 16dB(同じモデルを2回動かしたときは63dB)。耳で問題ないか確認中 |
| lite | `models/htdemucs_lite.onnx`(MatMul/Gemmだけ8bit) | 96MB | GPUが使えないスマホ | 58dB(ほぼ同じ)。ただしGPUなしなので遅い(速いPCでも15秒の動画で約4分) |

- `?model=full|fp16|lite` をURLにつけると、どの機器でも切り替えて試せる
- Convまで8bitにすると精度が大きく落ちた(周波数側の出力で差 5dB以下)
- fp16で正規化だけ32bitに残す変換は、onnxruntime の float16 変換の不具合(名前の重複)でモデルが壊れた

## スマホ対応
- 処理中は Wake Lock で画面をつけたままにする(画面が消えると止まるため)
- `navigator.share` が使える端末では「写真アプリなどに保存・共有」ボタンを出す(iPhoneで写真アプリに保存できる)

## しくみ
1. `AudioContext.decodeAudioData` で動画から音声を取り出す(44.1kHzに変換)
2. Demucs(htdemucs)のONNX版を onnxruntime-web(WebGPU、なければWASM)で動かし、4トラックに分離
   - 声 = vocals、BGM = drums + bass + other
   - モデル(約180MB)は Hugging Face から取得し、Cache API に保存して2回目以降は再ダウンロードしない
3. ffmpeg.wasm で元の映像(再エンコードなし)と声トラックを mp4 にまとめ直す

## 外部から読み込むもの
- onnxruntime-web 1.30.0(jsDelivr)
- @ffmpeg/core 0.12.10(jsDelivr)
- モデル: https://huggingface.co/timcsy/demucs-web-onnx

## 同梱しているもの(vendor/)
- `demucs-web/` … demucs-web 1.0.2 の src(MIT)
- `ffmpeg/` … @ffmpeg/ffmpeg 0.12.15 の dist/esm(MIT)。Workerを同じオリジンから起動する必要があるのでCDNではなく同梱

## 速さの目安
15秒の動画: 初回はモデルDL込みで数十秒〜、2回目以降は約30秒(WebGPUありのPC)。
