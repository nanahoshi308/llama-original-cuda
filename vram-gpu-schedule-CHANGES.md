# vram-gpu-schedule.patch — 変更内容まとめ

パッチメンテナンス用に、`vram-gpu-schedule.patch` に含まれる変更を機能ごとに整理したもの。

## 1. 新機能: 複数GPUのVRAM対応スケジューリング(元パッチの本体)

| ファイル | 内容 |
|---|---|
| `common/arg.cpp` / `common/arg.h` / `common/common.h` | 新CLIオプション追加: `--models-gpu-schedule`(有効化)、`--models-gpu-margin-mb`(GPU毎に確保する空きVRAMの余白、既定512MiB)、`--models-gpu-evict-timeout`(busyなモデルの追い出し待機秒数、既定30秒)。また `--list-devices-json` を追加(各GPUの空き/合計VRAMをJSONで出力する内部ヘルパー)。プリセット専用オプション `pin`(このモデルを自動追い出し対象から除外)も有効化。 |
| `tools/server/server-models.cpp` / `.h` | 本体ロジック: `query_gpu_devices()`(`--list-devices-json` を子プロセスとして呼び現在の空きVRAMを取得)、`estimate_model_vram()`(`common/fit.h` と同じ仕組みでモデル+コンテキスト+計算バッファの必要VRAMを見積もる)、`place_model()`(見積もりに基づき配置先GPUを決定、足りなければ同一GPU上の非busy・非pinモデルをLRU順に追い出す)。`server_model_meta` に `gpu_device`/`vram_bytes` フィールドを追加。 |
| `tools/server/README.md` | 上記オプション・「GPU-aware scheduling」セクションのドキュメント追加。 |
| `tools/server/tests/unit/test_router_gpu_schedule.py`(新規)/ `tools/server/tests/utils.py` | 元パッチに付属していたテスト(空き容量がある場合のロード、LRU追い出し、busy中モデルの保護、載らない場合のエラー)。フェイクGPU環境変数(`LLAMA_TEST_FAKE_GPUS`/`_FILE`、`LLAMA_TEST_FAKE_MODEL_VRAM`)によるテスト用フック込み。 |

## 2. 追加修正① — 「スティッキーデバイス」バグ

**症状**: 一度スケジューラがGPUを割り当てたモデルは、アンロード後に再ロードしても常に同じGPUへ固定され、VRAM状況が変わっても再評価されない。空いている方のGPUを使わずOOMで落ちることがあった。

**原因**: `place_model()` が配置決定を `meta.preset` の `LLAMA_ARG_DEVICE` オプションに書き込んでいたが、この同じオプションを「ユーザーが手動で `--device` を指定したか」の判定にも使っていた(`// manual placement always wins`)。そのため一度スケジューラが決めたGPUが、次回ロード時に「ユーザー手動指定」と誤認され、再評価がスキップされていた。

**修正箇所**: `tools/server/server-models.cpp` の `server_models::update_status()`。モデルが `UNLOADED` になった時点で、スケジューラが割り当てた `gpu_device`/`LLAMA_ARG_DEVICE` をクリアし、次回ロードで空きVRAMを再評価させる。

```cpp
// a device chosen by the GPU scheduler must not masquerade as a manual --device
// pin on the next load: place_model()'s "never second-guess a manual pin" check
// reads this same LLAMA_ARG_DEVICE option, so if we left it set here, the model
// would be stuck retrying (and potentially OOM'ing on) whatever GPU it happened
// to land on last time, even after that GPU fills up and another one frees up
if (args.status == SERVER_MODEL_STATUS_UNLOADED && !meta.gpu_device.empty()) {
    meta.preset.unset_option("LLAMA_ARG_DEVICE");
    meta.gpu_device.clear();
    meta.vram_bytes = 0;
}
```

**回帰テスト**: `test_router_gpu_schedule_reevaluates_placement_after_unload`(2GPU環境をシミュレートし、1回目はGPU0、GPU0が埋まった状態での再ロードでGPU1へ正しく移ることを検証)。

**検証方法**: 修正を外した状態でビルド→テスト失敗を確認、修正を戻してビルド→テスト成功を確認(fail/passのビゼクション)。実機でもOrnith1.0-9B/qwen3.5-9bを使って再現・修正確認済み。

## 3. 追加修正② — VRAM見積もりが `ctx-size`/`cache-type` を無視する

**症状**: 実際には空きVRAMのあるGPUでも「足りない」と誤判定され、既存モデルの不要な追い出しが発生。

**原因**: `estimate_model_vram()` が `preset.apply_to_params(params, {LLAMA_ARG_MODEL, ...})` のように model/mmproj/hf-repo系キーだけを渡していたが、この `handled_keys` 引数は除外リストではなく「**指定したキーだけ適用する**」包含フィルタ(`preset.cpp` の実装通り)。結果、`ctx-size`・`cache-type-k`・`cache-type-v`・`flash-attn` 等の実設定が一切見積もりに反映されず、`common_params` のデフォルト(`n_ctx=0` → モデルのネイティブ最大コンテキスト、KVキャッシュはF16非量子化)で計算されていた。

**修正箇所**: 同じく `estimate_model_vram()` 内。

修正前:
```cpp
common_params params;
meta.preset.apply_to_params(params, {
    "LLAMA_ARG_MODEL",
    "LLAMA_ARG_MODEL_URL",
    "LLAMA_ARG_MMPROJ",
    "LLAMA_ARG_MMPROJ_URL",
    "LLAMA_ARG_MMPROJ_AUTO",
    "LLAMA_ARG_HF_REPO",
    "LLAMA_ARG_HF_REPO_FILE",
});
params.offline = true;
```

修正後:
```cpp
common_params params;
meta.preset.apply_to_params(params);
params.offline = true;
```

モデルパス解決(hf-repo等のオフライン安全な解決)は変更後も従来通り `common_models_handler_apply()` で行われるため、挙動は変わらない。

**回帰テスト**: `test_router_gpu_schedule_estimate_honors_ctx_size`(同一モデル・同一VRAM予算で `ctx-size=16` と `ctx-size=16777216` を比較し、後者が正しく拒否されることを検証。`--fit` によるctx自動縮小を無効化するため両プリセットに `fit = false` を明示)。

**検証方法**: 修正を外した状態では巨大ctxのモデルが「載る」と誤判定され実ロード時に289GB相当のバッファ確保を試みてクラッシュすることを確認。修正後は事前に正しく拒否されることを確認(fail/passのビゼクション)。

## 補足

- テストスイート実行時、環境によっては `ggml-org/tinygemma3-GGUF` 等のHF-repo経由モデルがキャッシュスキャンで見つからず404になることがある(このマシンのテスト環境固有の問題で、コード側の欠陥ではない可能性が高い)。これが原因で `test_router_gpu_schedule_evicts_lru_same_device` 等、上記2件の追加修正と無関係なテストが失敗することがあったが、コードの不具合ではなく別途調査が必要。
- 上記2件の追加修正の検証には、この環境依存キャッシュスキャンに依存しない自己完結型のテスト(ローカルファイルパス・fake GPU環境変数を使用)を用いた。
