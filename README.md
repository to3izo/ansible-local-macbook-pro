# ansible-local-macbook-pro
Ansible ローカル環境自動構築プロジェクト（MacBook Pro/ 個人向け）

## 概要
Ansible を使って、ローカルマシンの環境構築を自動化します。

## ディレクトリ構造
roles 以下は、構造説明のためのサンプル表示です。

```
./
├── README.md
├── .gitignore             # バージョン管理対象外を指定
├── setup_macbook.yml      # メインのプレイブック（実行対象）
├── vars.yml.template      # 変数設定ファイルのテンプレート
└── roles/
    ├── homebrew/          # Homebrewのインストールと設定
    │   └── tasks/
    │       └── main.yml
    ├── git/               # Gitのインストールと設定
    │   └── tasks/
    │       └── main.yml
    └── ssh_key/           # SSH鍵の生成
        └── tasks/
            └── main.yml
```

## 前提条件
本ブロジェクトのプレイブックを動作させるには、以下の前提条件を満たす必要があります。

- macOS（Apple Silicon 推奨）
- 事前に以下の設定・インストールが完了していること
  - Homebrew
  - Ansible (Homebrew 経由)
  - Command Line Tools（`xcode-select --install` 実行で導入）

## 使い方
Ansible で環境構築は、以下のステップで完了します。

### 0. 事前準備
まだマシンに Git コマンドが導入されていない場合、`git clone` はできないため、本リポジトリを ZIP ファイルをダウンロードして、任意の場所で解凍して使用してください。

### 1. 変数ファイルの準備

`vars.yml.template` をコピーして、ファイル名を `vars.yml` に変更

```bash
cp vars.yml.example vars.yml
```

`vars.yml` に記載されている内容を、自分のアカウント情報等に修正してください

```yaml
---
# Git設定
git_user_name: "Taro Yamada"
git_user_email: "taro.yamada@example.com"

# SSH鍵設定
ssh_key_prefix: t-yamada 
ssh_key_comment: "tyamada@macbook-pro-m5"
```

#### 補足: `vars.yml` の変数の内容

| 変数名 | 説明 | デフォルト値 |
|--------|------|--------------|
| `git_user_name` | Gitのユーザー名 | Your Name |
| `git_user_email` | Gitのメールアドレス | your.email@example.com |
| `ssh_key_prefix` | SSH鍵のファイル名 | （コメントアウト） |
| `ssh_key_comment` | SSH鍵のコメント | `USER@HOSTNAME`（自動生成） |

`ssh_key_prefix` を設定すると鍵ファイル名が `id_ed25519_{ssh_key_prefix}` となります。
コメントアウトすると鍵ファイル名は `id_ed25519` となります。

> **Note**: `vars.yml`は`.gitignore`に含まれているため、個人情報が Git にコミットされる心配はありません。


### 2. 導入する内容を確認
以下の箇所については、CLI/GUI ツールをどこまで導入するか決めて、不要なものがあればコメントアウトしてください。（※ コメントアウトすべきものがわからず、動作成功に不安がある場合は、そのまま入れることを推奨します）

- Homebrew インストール内容: `roles/homebrew/tasks/main.yml`
- npm パッケージのグローバルインストール: `roles/node_modules/tasks/main.yml`

上記に関しては、導入後に不要なものがあれば、`brew uninstall`, `npm uninstall` コマンド等で後から簡単にアンインストールできます。

機密情報を扱うタスクがある場合（`become: true`）、追加設定と共に起動方法が変わります。詳しくは、[機密情報を扱うタスクが存在する場合](#機密情報を扱うタスクが存在する場合) を参照してください。

### 3. プレイブックの実行
以下のコマンドで自動構築が完了します。

```bash
ansible-playbook setup_macbook.yml
```

#### 補足 1: 特定の role のみ実行

特定のroleだけを実行したい場合は、`--tags` オプションを使用します

```bash
# macOS の初期設定のみ
ansible-playbook setup_macbook.yml --tags macos_setting

# Git のみセットアップ
ansible-playbook setup_macbook.yml --tags git

# SSH 鍵のみ生成
ansible-playbook setup_macbook.yml --tags ssh_key

# node モジュールの導入のみ
ansible-playbook setup_macbook.yml --tags node_modules

# Homebrewのみインストール
ansible-playbook setup_macbook.yml --tags homebrew
```

#### 補足 2: コマンドラインから変数を指定

`vars.yml` を使わずに、コマンドラインから直接変数を指定することもできます

```bash
ansible-playbook setup_macbook.yml \
  -e "git_user_name='Taro Yamada'" \
  -e "git_user_email='taro@example.com'"
```

## 導入・設定内容の微調整方法
以下の内容については、導入するもの/しないものを簡単に追加・変更することができます。

- Homebrew インストール内容: `roles/homebrew/tasks/main.yml`
- npm パッケージのグローバルインストール: `roles/node_modules/tasks/main.yml`

```yaml:各ロールファイル
name:
  # - podman  # スキップしたいツールをコメントアウト
  - docker
  - <new_module>  # 新たにイントールしたいモジュール名を追加
```

追加できるモジュールについては以下を参照してください：
- **Homebrew パッケージ**: [Homebrew](https://brew.sh/ja) で検索可能
  - Ansibleでの記載方法: 公式ページのコマンドで `--cask` オプションが付いているものは `community.general.homebrew_cask` モジュールの `name` に、付いていないものは `community.general.homebrew` モジュールの `name` に「ツール名」をそのまま指定
    - 例: `docker`, `git`, `node` など
- **npm パッケージ**: [npm公式サイト](https://www.npmjs.com/) で検索可能
  - Ansibleでの記載方法: `shell` モジュールで `npm i -g` または `pnpm i -g` コマンドに続けく「パッケージ名」を指定
    - 例: `@angular/cli`, `vercel`, `typescript` など

> **Tip**: Ansibleモジュールの詳細な使い方は [Ansible公式ドキュメント](https://docs.ansible.com/ansible/latest/collections/index.html) で確認できます。

また、role 自体を作成したり、動作する role を変更する場合は、以下のように修正してください

```yaml:setup_macbook.yml
roles:
  - homebrew
  - git
  # - ssh_key   # 不要な処理をコメントアウト
  - <new_role>  # 新しい role を作成後、プレイブックに追加
```

## 機密情報を扱うタスクが存在する場合
各タスクの中で `become: true` と記載されているステップや、`sudo` コマンドによる管理者権限が必要なステップ、または、API キーなどの機密情報を扱う場合は、**vars.yml** に値を格納せず、`ansible-valut` を使って暗号化して変数名と値を保持することを推奨します。

Vault ファイルを生成

```bash
ansible-vault create vault.yml
```

vim でテキストエディタが開かれた状態になるので、変数と値をセットしてください

```bash
# sudo コマンド等で使用されるパスワードを記載
ansible_become_pass: your_sudo_password
# 他にも機密情報があれば追記
github_token: ghp_xxxxxxxxxxxxx
api_key: secret_api_key_here
database_password: db_password_here
```
vault.yml が追加され、暗号化されたパスワード等が保持されます。

`ansible_become_pass` は特別な変数で、プレイブックや Role 上で変数として呼び出す必要はありません。自動的に使用されます。その他の変数は `{{api_key}}` の様に変数として利用することが可能です。

次に、プレイブックの vars_files に vault.yml を追加します。

```yaml:setup_macbook.yml
vars_files:
  - vars.yml
  - vault.yml  # 👈 コメントを外して利用できる状態にする
```

最後に、valut.yml に渡した値を複合化してプレイブックを実行します。`--ask-vault-pass` オプションで複合化してくれます。
```bash
ansible-playbook setup_macbook.yml --ask-vault-pass
```

その他のコマンド

```bash
# 閲覧のみ (編集不可)
ansible-vault view vault.yml

# 編集
ansible-vault edit vault.yml

# 一時的に復号化 (平文で保存)
ansible-vault decrypt vault.yml

# 再暗号化
ansible-vault encrypt vault.yml

# パスワード変更
ansible-vault rekey vault.yml
```
