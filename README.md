# fizz-memory-l2-events

Fizz 記憶系の **L2**(永続層)。すべての `MemoryEvent` をディスクに
append-only で残し、`user_id` でクエリできるようにする。

| 層 | 部品 | 役割 |
|---|---|---|
| L1 | fizz-memory-l1-ring | per-user 直近発言のインメモリ ring(hot path) |
| **L2** | **fizz-memory-l2-events**(これ) | 全発言の永続化と検索 |
| L3 | fizz-memory-l3-summarizer | 閾値超で per-user 要約を rolling 更新 |

旧実装は `node:sqlite` だったが、Almide に SQLite が無いので
**1 行 1 イベントの NDJSON ファイル(append-only)** に置き換えた。採番は
既存最大 id + 1。クエリは全行を読み直して filter する単純実装(配信規模では
十分。ホットパスの即応は L1 ring が担うので L2 は許容)。

## I/O

stdin に 1 行 1 コマンド(NDJSON):

```jsonc
{"op":"append","event":{"id":0,"user_id":7,"session_id":null,"role":"user","content":"こんにちは","ts":100,"tags":[]}}
{"op":"query","user_id":7,"limit":10}
{"op":"wipe"}
```

- `append` → ディスクに追記。`id` が 0 以下なら採番。`{"ok":true,"id":N}` を返す。
- `query` → 該当ユーザの末尾 `limit` 件(古い順)。`limit<=0` で全件。`{"user_id":N,"events":[...]}`。
- `wipe` → ログ全消去。`{"ok":true}`。

保存先は環境変数 `FIZZ_L2_PATH`(既定 `events.jsonl`)。不正な行は stderr に
警告して読み飛ばす。EOF で終了。

## 開発

```sh
almide check src/main.almd
almide test
almide build src/main.almd -o build/fizz-memory-l2-events
```

ツールチェーン: [almide](https://github.com/almide/almide) v0.26.6+。

## 契約

[fizz-protocol](https://github.com/aiviecast/fizz-protocol) の `memory` モジュール
(`MemoryEvent` とその Codec)に依存。
