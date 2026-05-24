# CLAUDE.md

このリポジトリで Claude Code が作業するときの前提情報・規約.

---

## プロジェクト概要

LDRP (Delivery Routing Problem) のマルチエージェント学習環境。複数のエージェントが配送タスクを処理する問題を、強化学習で解く実験プラットフォーム。

- **経路計画**: PBS (探索), IQL / QMIX / VDN / MAA2C (epymarl 経由の MARL)
- **タスク割当**: TP (近接優先), FIFO, PPO
- **環境ラッパ**: SafeEnv (= 衝突を事前回避), 生 DrpEnv (= 回避なし)

---

## 開発環境 (固定)

- **Python**: conda env `ldrp` を使う (`./setup_env.sh` で作成)
  - Python **3.9** / gym 0.26.2 / numpy 1.26.4 / torch 2.8.0 / networkx 3.2.1 / PyYAML 6.0.3
  - **Python 3.10+ 専用構文 (`X | None` 等の PEP 604) は使わない**。`Optional[X]` を使う
  - システム Python (numpy 2 系) は gym 0.26 と非互換 → 使わない
- **CUDA**: GPU があれば自動利用 (コードは `torch.cuda.is_available()` で分岐済み)

実行時は必ず `conda activate ldrp` で env に入る (または直接 `/path/to/envs/ldrp/bin/python ...`)。

---

## ファイル構成

```
LDRP/
├── CLAUDE.md                       # 本ファイル
├── README.md                       # Quick Start
├── setup_env.sh                    # conda env セットアップ (= 1 コマンド)
├── requirements.txt                # 依存パッケージ (= pin 済み)
├── runner.py                       # 推論/学習ループ
├── test.py                         # gym.make + Runner 起動 (CLI: map agent planner task)
├── train.py                        # epymarl サブプロセス起動 (= MARL 学習)
├── run.py                          # test.py のバッチ実行 (= 複数組合せ並列)
├── src/
│   ├── config/default.yaml         # 全 yaml キー
│   ├── main/drp_env/
│   │   ├── drp_env.py              # env 本体. step() に SafeEnv との連携あり
│   │   ├── __init__.py             # gym register (drp-..., drp_safe-...)
│   │   ├── EE_map.py               # グラフ構築 + 衝突判定
│   │   ├── map/                    # 各マップの node.csv / edge.csv
│   │   ├── state_repre/            # 観測表現 (onehot_fov 等)
│   │   └── wrapper/safe_marl.py    # SafeEnv ラッパ (drp_safe-...)
│   ├── all_policy/                 # 経路計画 (PBS, MARL 推論)
│   ├── task_assign/                # タスク割当 (PPO/TP/FIFO)
│   └── epymarl/                    # MARL 学習フレームワーク (= 外部依存)
└── logs/                           # 実行ログ (run.py の出力先)
```

---

## 不変条件 (Invariants) — これを破らない

- env 名は `drp_env:drp_safe-{N}agent_{map}-v2` (= SafeEnv) または `drp_env:drp-{N}agent_{map}-v2` (= 生 DrpEnv)
- `test.py` は CLI 引数 4 つ (map_name, agent_num, path_planner, task_assigner) で `default.yaml` の対応 4 項目を上書き。他は yaml のまま
- `task_flag=True` 時、エピソードは collision または time_limit 到達まで継続
- collision 判定は `EE_map.collision_detect` で **物理距離 < 5** ハードコード
- 既存ファイルへの編集は最小限。新機能は新ディレクトリに分離

### SafeEnv と PBS のトレードオフ (= `pbs_mode` フラグで切替)

[src/main/drp_env/drp_env.py](src/main/drp_env/drp_env.py) の `step()` 内、待機分岐に `if self.pbs_mode:` でガードされた代入 `self.current_goal_prepare[i] = action_i` がある:

| `pbs_mode` | 待機分岐の動作 | 影響 |
|---|---|---|
| `False` (デフォルト) | 代入をスキップ → `current_goal` は None のまま | **SafeEnv が待機 agent も保護** ✓ MARL 系 (QMIX/IQL/VDN/MAA2C) で衝突減 |
| `True` | 代入実行 → `current_goal = 待機ノード id` | **PBS の path 計画が正しく動く** ✓ ただし SafeEnv の保護は失う |

`test.py` は `config.path_planner == "pbs"` のとき自動で `pbs_mode=True` を `gym.make()` に渡す。MARL 系では `False` (= デフォルト) のまま。**通常はこの flag を触らない**。

---

## ユーザー嗜好

- **応答は日本語で簡潔に**。冗長な接頭辞・末尾サマリは付けない
- **ツール呼び出しの `description` (承認プロンプトに表示される説明文) は必ず日本語で書く**。
  - `Bash` の `description`: 例 `"git status を確認"`, `"ldrp 環境で test.py を起動"`
  - `Agent` / `TodoWrite` の `content` / `activeForm`: 同上
  - 理由: 承認画面で「何をしようとしているか」がユーザーに即座に伝わるようにするため
- **コード/設定を変更したら、応答中で必ず以下をセットで書く**:
  1. **修正の意図** (なぜこの変更が必要か)
  2. **修正によって何がどう変わるか** (挙動・出力・互換性・既存ファイルへの影響)
- スコープを広げる前に確認質問する
- 完了報告は 1〜2 行
- ユーザーの指示なしに先回り判断しない (= 「冗長を削っておきました」のような勝手な拡張は禁止)

---

## 禁止事項

- `git commit --amend` で履歴書き換え (常に新コミット)
- `git push --force` (確認なし)
- `pkill` 等プロセス強制終了 (事前提案・確認)
- `--no-verify` でフックスキップ
- 大きな未承認スコープ変更 (リファクタ含む)

---

## CLI コマンド早見表

```bash
# 単発実行 (引数 4 つ全部指定)
python test.py map_8x5 4 qmix tp

# default.yaml で実行 (引数なし)
python test.py

# バッチ (= run.py 内で複数組合せを subprocess 並列)
python run.py

# MARL 学習 (epymarl サブプロセス起動)
python train.py
```

| カテゴリ | 利用可能な値 |
|---|---|
| `path_planner` | `pbs`, `iql`, `qmix`, `vdn`, `maa2c` |
| `task_assigner` | `tp`, `fifo`, `ppo` |
| `map_name` | `map_3x3`, `map_5x4`, `map_8x5`, `map_10x{6,8,10}`, `map_aoba0{0,1}`, `map_kyodai`, `map_osaka`, `map_paris`, `map_shibuya`, `map_shijo` |

---

## 検証スニペット (= 動作確認の定型)

env の基本動作を手早く確認:

```python
import gym, numpy as np, sys
sys.path.append('.'); sys.path.append('./src/main')
import drp_env

env = gym.make('drp_env:drp_safe-3agent_map_5x4-v2',
               state_repre_flag='onehot_fov',
               reward_list={'goal':100,'collision':-100,'wait':-10.,'move':-1},
               time_limit=8, task_flag=True, task_list=None)
np.random.seed(0)
env.reset(); t=0; done=False
while not done and t < 10:
    ta = [-1] * 3   # 全 agent タスク割当なし
    a = {'pass': [np.random.randint(env.unwrapped.n_nodes) for _ in range(3)],
         'task': ta}
    obs, rew, term, info = env.unwrapped.step(a)
    done = all(term); t += 1
print(f"finished at t={t}, info={info}")
```

期待: 数ステップ〜time_limit まで動いて停止 (衝突 or timeup)。

---

## トラブルシューティング

| 症状 | 原因と対処 |
|---|---|
| `np.bool8` 等のエラー | base conda env で実行している. `conda activate ldrp` |
| `conda: command not found` | anaconda / miniconda 未インストール. インストール後シェル再起動 |
| `!!!collision!!!` が多発 | SafeEnv は **全衝突を防げない**. 中継ノード収束 (= 別 edge 上で交差) は防げない. `path_planner=pbs` で抑制可能 |
| `Fatal Python error: PyEval_RestoreThread` | Ctrl+C 割込み時の GIL 例外. CLI Python の既知の挙動. 再実行で通る場合あり |
| `pip install` で torch が遅い | M1/M2 Mac で 1〜2 分は正常 (= MPS 対応版を取得中) |
| smac のインストールに失敗 | `git` 未インストール. `brew install git` / `apt install git` |

---

## 開発フロー (推奨)

```bash
# 1. 修正前にスモークテスト
python test.py map_5x4 3 qmix tp   # 数十秒で完走するか確認

# 2. 修正

# 3. 修正後の動作確認 (= 修正と無関係な部分が壊れていないか)
python test.py map_5x4 3 qmix tp

# 4. 影響範囲確認 (= 別 map / 別 planner で同じ修正が問題ないか)
python test.py map_8x5 4 qmix tp
python test.py map_8x5 4 pbs tp     # pbs_mode が走る経路
```
