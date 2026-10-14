# BGMけしゴム

動画生成AIで作った動画に勝手についたBGMを取り除くブラウザツール(HTML1枚・ビルド不要)。
処理はすべてブラウザ内で行い、ファイルはどこにも送信しない。運営費0円の構成。

- 状態: 未公開(ローカルのみ・リポジトリ未作成)
- ローカル確認: `.claude/launch.json` の `bgm-remover`(ポート8770)

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
