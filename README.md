# mysql-lm-bench

InnoDB スピンロックの `innodb_spin_wait_pause_multiplier` パラメータが
TPS・レイテンシに与える影響を測定するベンチマークリポジトリ。

`large-multiplier` パッチを適用した MySQL 8.0 バイナリと sysbench を同梱しており、
`git clone` → セットアップ → ベンチ実行まで 1 台のサーバ上で完結する。

---

## ディレクトリ構成

```
mysql-lm-bench/
├── installs/
│   └── large-multiplier/       # パッチ済み MySQL インスタンス
│       ├── bin/
│       │   ├── mysqld          # mysqld バイナリ（Git LFS）
│       │   ├── mysqld_safe     # mysqld_safe（Git LFS）
│       │   └── mysql           # mysql クライアント（Git LFS）
│       └── BUILD-INFO.txt      # ビルド情報（ソースコミット・パッチ名等）
├── bin/
│   ├── sysbench                # sysbench ラッパースクリプト
│   └── sysbench.bin            # sysbench バイナリ本体（Git LFS）
├── lib/
│   └── libmysqlclient.so.24    # sysbench が依存する MySQL クライアントライブラリ（Git LFS）
├── share/
│   └── *.lua                   # sysbench の Lua テストスクリプト（oltp_read_write 等）
└── scripts/
    ├── setup-server.sh         # セットアップ（依存パッケージ・初期化・起動・prepare）
    ├── bench-multiplier.sh     # multiplier 実験ベンチマーク
    ├── init-instance.sh        # MySQL インスタンス初期化（my.cnf 生成・--initialize-insecure）
    ├── connect.sh              # mysql クライアントで接続
    ├── start.sh                # mysqld 起動
    ├── stop.sh                 # mysqld 停止
    └── status.sh               # mysqld 稼働確認
```

### バイナリについて

`installs/large-multiplier/bin/mysqld` は `innodb_spin_wait_pause_multiplier` の
上限を 100 → 10000 に拡張するパッチを適用した MySQL 8.0 ビルド。
通常の mysqld では multiplier=100 を超える値を設定しても 100 でクランプされる。

`bin/sysbench.bin` は Ubuntu 22.04 向けにビルドした sysbench 1.1.0。
依存ライブラリ `libmysqlclient.so.24` を `lib/` に同梱し、
`bin/sysbench` ラッパーが `LD_LIBRARY_PATH` を設定して呼び出す。

---

## 実行手順

### 1. clone

```bash
git clone git@github.com:hikaru2003/mysql-lm-bench.git
cd mysql-lm-bench
git lfs pull
```

### 2. セットアップ（初回のみ）

```bash
scripts/setup-server.sh large-multiplier \
  --prepare \
  --port 3307 \
  --mysqld-cores 0-15 \
  --sysbench-cores 16-19
```

- 必要な apt パッケージを自動インストール
- MySQL インスタンスを `installs/large-multiplier/` 以下に初期化
- mysqld を起動し、sysbench 用テーブルを作成（sbtest DB）

2 回目以降は `data/ibdata1` が存在すれば初期化をスキップし、
`etc/my.cnf` が欠損している場合のみ再生成する。

### 3. ベンチマーク実行

```bash
scripts/bench-multiplier.sh large-multiplier \
  --server <サーバ識別名> \
  --multipliers 0,5,10,25,50,75,100,150,200,300,500 \
  --threads 4,8,16,32,64 \
  --mysqld-cores 0-15 \
  --sysbench-cores 16-19 \
  --runs 5 \
  --time 60
```

`--server` に指定した名前が結果ディレクトリ名になる（例: `node0`, `skylake01`）。

実行順序: 全 multiplier を 1 周するのを `(runs + 1)` 回繰り返す。
- round 1: warmup（記録しない）
- round 2〜6: 本番計測

### 4. 結果の確認

```
experiments/results/large-multiplier/<サーバ名>/multiplier_experiment/
  summary.txt     # メタ情報 + 平均 TPS・レイテンシ
  metrics.tsv     # 全 run の生メトリクス（TSV）
  raw/            # sysbench 生出力（t<T>_m<M>_r<R>.txt）
```

---

## large-multiplier パッチについて

`installs/large-multiplier/applied.patch` を参照。
InnoDB のスピンループ内 `ut_delay()` に渡す乗数の上限値を変更している。
ソースコミット: `f9c88132a87a7d9e740e50bce2621999695bd3fe`（MySQL 8.0）
