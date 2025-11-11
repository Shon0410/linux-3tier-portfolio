# LinuC Lv3学習者が構築したWeb/DNS/DB 3層分離構成（設計思想とトラブルシュートまとめ）

## 1. はじめに（学習の動機）

はじめまして。現在、インフラエンジニアとして運用業務に従事しています。
LinuC Level3（304）を取得し、体系的な知識を実践的なスキルに昇華させるため、また、将来的な「設計構築エンジニア」へのキャリアアップを目指し、自宅の仮想環境にWeb/DNS/DBの3層分離システムを構築しました。

この記事は、その構築記録と、特に苦労した「DNS名前解決のトラブルシューティング」のプロセスをまとめたものです。

**【学習に使用した主な教材】**
* Linuxサーバー構築標準教科書
* オープンソースデータベース標準教科書

## 2. システム構成図（最終形）

まず、構築したシステムの全体像です。

「役割の分離（Web/DNS/DB）」、「多層防御（FW/DB認証）」と「DNSの権限委譲」の3点を意識して設計しました。

![system_architecture_diagram.drawio.png]


<img width="739" height="897" alt="system_architecture_diagram drawio" src="https://github.com/user-attachments/assets/df20969b-c97b-41a0-9b1a-2d873304a78c" />

### サーバー一覧（役割とIP）

| 役割 | ホスト名（OS設定） | IPアドレス | 主な役割 / **DNSアクセス名** |
| :--- | :--- | :--- | :--- |
| DNSキャッシュ | `host0.jp` | `192.168.56.100` | DNSキャッシュ、jpゾーン管理 |
| DNSサーバー1 | `host1.example1.jp` | `192.168.56.101` | example1.jp ゾーン管理 |
| DNSサーバー2 | `host2.example2.jp` | `192.168.56.102` | example2.jp ゾーン管理 |
| **Webサーバー** | `host4.example4.jp` | `192.168.56.103` | Apache (httpd) / **(アクセス名: `host4.example1.jp`)** |
| **DBサーバー** | `host3.example3.jp` | `192.168.56.104` | PostgreSQL サービス |

---

## 3. こだわった設計のポイント

単に構築するだけでなく、実務を想定した「設計思想」を3点盛り込みました。

### ポイント1：Web/DB/DNSの「役割分離」

Webサーバー、DBサーバー、DNSサーバーをそれぞれ独立した仮想マシンとして構築しました。

* **設計意図:**
    * **セキュリティリスクの分離:** 最も攻撃を受けやすいWebサーバーが侵害されても、DBサーバーやDNSサーバーへの影響を最小限に抑えるため。
    * **単一障害点（SPOF）の回避:** 1台のマシンが停止しても、他のサービスが連鎖的に停止するのを防ぐため。

### ポイント2：DBサーバーの「多層防御」

DBサーバー（PostgreSQL）へのアクセスを、以下の2段階で厳格に制限しました。

1.  **ファイアウォール（L4）による制限:**
    DBサーバーの`firewalld`設定で、Webサーバー（`.103`）からの**Port 5432**への通信のみを許可。
    ```bash
    # DBサーバー(.104)で実行
    sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.56.103/32" port protocol=tcp port=5432 accept'
    ```
2.  **DB認証（L7）による制限:**
    PostgreSQLの設定ファイル（`pg_hba.conf`）で、Webサーバー（`.103`）から特定のユーザー（`web_user`）による接続のみを許可。
    ```conf
    # DBサーバー(.104)の pg_hba.conf
    host    web_app_db  web_user    192.168.56.103/32  md5
    ```
* **設計意図:** 万が一ファイアウォールが突破されても、DB側の認証で不正アクセスを防ぐ「多層防御」を実現しました。

### ポイント3：DNSの「権限委譲」

**DNSサーバーを3台使い、jpゾーン（.100）が example1.jp と example2.jp の2つのゾーンに権限を委譲**する構成を構築しました。

* **設計意図:** 大規模なネットワークにおいて、ドメイン管理を階層化する実務的なプロセスを学習するため。
* **補足:** 今回のWebアクセス（host4.example1.jp）にはDNSサーバー2は関与しませんが、システム全体としては正常に動作しており、**権限委譲の仕組み**を深く理解できました。

---

## 4. 構築手順の詳細

### 4-1. Webサーバー（192.168.56.103）の構築手順

Webサーバー（Apache/httpd）の構築において、以下の手順を実施しました。

### A. インストールとサービス起動

1.  **Apache (httpd) のインストール**
    ```bash
    sudo dnf install httpd
    ```

2.  **サービス自動起動の設定と起動とその確認**
    
    （OS起動時にhttpdが自動で起動するように設定。statusで"active (running)"になっていることも確認しました）
    ```bash
    sudo systemctl enable httpd
    sudo systemctl start httpd
    sudo systemctl status httpd
    ```

### B. セキュリティ設定（ファイアウォール）

**設計意図:**
Webサーバーとして必要な通信（HTTP）のみを許可し、不要なポートを閉じることで、セキュリティの「最小権限の原則」を適用しました。

1.  **HTTPサービスの許可**
    ```bash
    # publicゾーンでhttp (Port 80) サービスを恒久的に許可
    sudo firewall-cmd --add-service=http --zone=public --permanent
    
    # 設定のリロード
    sudo firewall-cmd --reload

    # ファイアウォールの設定を確認（services行にhttpが追加されたことを確認しました）
    sudo firewall-cmd --list-all
    ```

### C. 構文チェックと動作確認

1.  **Apache設定ファイルの構文チェック**
    （設定ファイル（httpd.conf）はデフォルトのままですが、構文が正しいことを確認しました）
    ```bash
    $ sudo httpd -t
    Syntax OK
    ```

2.  **動作確認**
    （クライアントPCのブラウザ、またはDNSサーバー（.100）から`curl`コマンドで確認）
    ```bash
    $ curl http://host4.example1.jp
    Hello,World
    ```

### 4-2. DNSサーバー（.100, .101, .102）の構築手順
### 1.仮想マシンを三台用意。（DNSキャッシュ、DNSサーバー1、DNSサーバー2）
### A.DNSサーバー1 (.101) の構築 (コンテンツサーバー)
#### 1.BINDのインストール
```bash
#BINDのインストールには、bindパッケージとbind-chrootパッケージが必要。必要なパッケージをインストールするために、dnfコマンドでインストールしました。
sudo dnf install bind bind-chroot
```

#### 2./etc/named.confの基本設定

```bash
sudo vi /etc/named.conf
```

##### ・optionsのlisten-onの行に192.168.56.101を追加。

##### ・allow-queryの行で、localhostからanyに変更。

##### ・recursionをyesからnoに変更。

##### ・下記の正引きゾーンを追加。

```bash
# "include"の上の行に記載。
zone "example1.jp" IN {
type master;
file "example1.jp.zone";
allow-update { none; };
};
```

#### 3.ゾーンファイルの準備
```bash
# ゾーンファイルのテンプレートとなる/var/named/named.emptyファイルをコピー。また、コピー元と同じ所有権（root:named）及びパーミッション（640）にするため、cpコマンドに-pオプションを付けました。
め、cp コマンドに-p オプションを付けて実行
sudo cp -p /var/named/named.empty /var/named/example1.jp.zone
sudo ls -l /var/named/example1.jp.zone
```

#### 4.ゾーンファイルの修正
```bash
# コピー先のファイルを修正。
sudo vi /var/named/example1.jp.zone
```
##### "TTL"行の一つ下の行に"$ORIGIN example1.jp."を記載。

##### SOAレコードを下記のように修正。
```bash
@ IN SOA host1 root
```

##### 下記のNSレコードを追加。
```bash
NS host1.example1.jp.
```

##### 下記のAレコードを追加。
```bash
host1 A 192.168.56.101
```

#### 5.設定ファイルの書式確認
```bash
# /etc/named.confに間違いが無いか確認。
sudo named-checkconf
```
#### 6.ゾーンファイルの書式確認
```bash
# 構文エラーが無いか確認。OKと表示されたら問題無し。
sudo named-checkzone example1.jp /var/named/example1.jp.zone
```

#### 7.BINDの起動と確認
```bash
sudo systemctl start named-chroot

# Active の欄に「active (running)」と表示されていることを確認
sudo systemctl status named-chroot
```

#### 8.自動起動の設定と確認
```bash
sudo systemctl enable named-chroot

# "enabled"と表示されればOK。
sudo systemctl is-enabled named-chroot
```

#### 9.ファイアウォールの設定
```bash
sudo firewall-cmd --add-service=dns --zone=public --permanent
sudo firewall-cmd --reload
```

#### 10.digコマンドでホストの名前解決を確認
```bash
# OSの標準DNS（.100）ではなく、自分自身（.101）に直接問い合わせて確認。
dig host1.example1.jp @192.168.56.101
```

### B.DNSサーバー2 (.102) の構築 (コンテンツサーバー)


#### 1./etc/named.confの基本設定（※2-1の手順を再度実施後）

```bash
sudo vi /etc/named.conf
```

##### ・optionsのlisten-onの行に192.168.56.102を追加。

##### ・allow-queryの行で、localhostからanyに変更。

##### ・recursionをyesからnoに変更。

##### ・下記の正引きゾーンを追加。

```bash
# "include"の上の行に記載。
zone "example2.jp" IN {
type master;
file "example2.jp.zone";
allow-update { none; };
};
```
#### 2.ゾーンファイルの準備

```bash
sudo cp -p /var/named/named.empty /var/named/example2.jp.zone
sudo ls -l /var/named/example2.jp.zone
```

#### 3.ゾーンファイルの修正
```bash
# コピー先のファイルを修正。
sudo vi /var/named/example2.jp.zone
```
##### "TTL"行の一つ下の行に"$ORIGIN example2.jp."を記載。

##### SOAレコードを下記のように修正。
```bash
@ IN SOA host2 root
```

##### 下記のNSレコードを追加。
```bash
NS host2.example2.jp.
```

##### 下記のAレコードを追加。
```bash
host2 A 192.168.56.102
```

#### 4.digコマンドでホストの名前解決を確認（※2-5～2-9の手順を再度実施後）
```bash
dig host2.example2.jp @192.168.56.102
```

### C.DNSキャッシュ (.100) の構築

#### 1./etc/named.confの基本設定（※2-1の手順を再度実施後）

```bash
sudo vi /etc/named.conf
```

##### ・optionsのlisten-onの行に192.168.56.100を追加。

##### ・allow-queryの行で、localhostからanyに変更。

##### ・recursionはyesのままにする！！！

##### ・下記の正引きゾーンを追加。

```bash
# "include"の上の行に記載。
zone "jp" IN {
type master;
file "jp.zone";
allow-update { none; };
};
```
#### 2.ゾーンファイルの準備

```bash
sudo cp -p /var/named/named.empty /var/named/jp.zone
sudo ls -l /var/named/jp.zone
```

#### 3.ゾーンファイルの修正
```bash
# コピー先のファイルを修正。
sudo vi /var/named/jp.zone
```
##### "TTL"行の一つ下の行に"$ORIGIN jp."を記載。

##### SOAレコードを下記のように修正。
```bash
@ IN SOA host0 root
```

##### 下記のNSレコードを追加。
```bash
# example1.jpとexample2.jpのそれぞれのゾーンのNSレコードとAレコードをグルーレコードとして記載することで、循環参照を回避。
example1.jp. NS host1.example1.jp.
example2.jp. NS host2.example2.jp.
```

##### 下記のAレコードを追加。
```bash
# example1.jpとexample2.jpのそれぞれのゾーンのNSレコードとAレコードをグルーレコードとして記載することで、循環参照を回避。
host0 A 192.168.56.100
host1.example1.jp. A 192.168.56.101
host2.example2.jp. A 192.168.56.102
```

#### 4.digコマンドでホストの名前解決を確認（※2-5～2-9の手順を再度実施後）
```bash
# .100が.101と.102に正しく委譲できているか確認。
dig example1.jp @192.168.56.100
dig example2.jp @192.168.56.100

# .100が、.101に問い合わせてWebサーバー(.103)のAレコードを引けるか確認
dig host4.example1.jp @192.168.56.100
```

### 4-3. DBサーバー（192.168.56.104）の構築手順
### 1.仮想マシンを一台用意。
### 2.DBサーバーの構築
#### 1.PostgreSQLのインストール
```bash
sudo dnf install postgresql-server
```
#### 2.データベースの初期化
```bash
sudo postgresql-setup --initdb
```

#### 3.サービス有効化と起動
```bash
sudo systemctl enable postgresql
sudo systemctl start postgresql
```

#### 4.セキュリティ設定（ファイアウォール）
```bash
# Webサーバー(192.168.56.103)からのPort 5432(PostgreSQL)へのアクセスを許可
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.56.103/32" port protocol=tcp port=5432 accept'
```

#### 5.セキュリティ設定のリロード（ファイアウォール）
```bash
sudo firewall-cmd --reload
```

#### 6.リモートアクセス許可設定（postgresql.confの編集（外部IPからの接続を許可））
```bash
# 下記コマンドを入力後、#listen_addresses = 'localhost' の行のコメントアウト（#）を解除し、'*' に変更。
sudo vi /var/lib/pgsql/data/postgresql.conf
```

#### 7.リモートアクセス許可設定（pg_hba.confの編集（WebサーバーのIPとユーザーを許可））
```bash
sudo vi /var/lib/pgsql/data/pg_hba.conf
# 上記コマンドを入力後、ファイルの末尾に、Webサーバー（.103）からの接続を許可する以下の行を追記。
# TYPE  DATABASE    USER        ADDRESS           METHOD
host    web_app_db  web_user    192.168.56.103/32  md5
```

#### 8.PostgreSQLサービスの再起動
```bash
sudo systemctl restart postgresql
```

#### 9.PostgreSQLへ接続
```bash
sudo -u postgres psql
```

#### 10.データベースとユーザーを作成
```bash
-- Webアプリ用のデータベースを作成
CREATE DATABASE web_app_db;

-- Webアプリ用のユーザーを作成（パスワードは強力なものに変更してください）
CREATE USER web_user WITH PASSWORD 'MyStrongP@sswOrd123!';

-- ユーザーにデータベースへの権限を付与
GRANT ALL PRIVILEGES ON DATABASE web_app_db TO web_user;

-- 終了
\q
```

### 3.Webサーバーからの接続テスト
※ここから先はWebサーバー（192.168.56.103）上で実行！！

#### 1.PostgreSQLクライアントツールのインストール（もし未実施の場合）
```bash
sudo dnf install postgresql -y
```

#### 2.接続テストの実行
```bash
# パスワード入力を求められ、web_app_db=> というプロンプトが表示されればOK！
psql -h 192.168.56.104 -U web_user -d web_app_db
```
## 5. 最大の難関：`dig`は成功するのに`curl`が失敗した話

システム構築中、最も時間を要したのが、サーバー内部での名前解決（リゾルバ）の問題でした。

### 発生した問題

DNSキャッシュサーバー（`.100`）やWebサーバー（`.103`）で、以下の現象が発生しました。

1.  `dig host4.example1.jp @127.0.0.1` $\to$ **成功**（ANSWER SECTIONが返る）
2.  `curl http://host4.example1.jp` $\to$ **失敗**（`Could not resolve host`）

`dig`が成功し、`curl`が失敗することから、**「DNSサーバー（BIND）自体は正常だが、OSがBINDに正しく問い合わせできていない」**と切り分けました。

### 調査と解決のプロセス

1.  **仮説1：`systemd-resolved`の競合？**
    * `sudo systemctl status systemd-resolved` $\to$ `Unit not found.`（サービスが動いていないため却下）
2.  **仮説2：`NetworkManager`のキャッシュ？**
    * `sudo systemctl restart NetworkManager` $\to$ 解決せず。
3.  **仮説3：`firewalld`によるローカル通信ブロック？**
    * `sudo systemctl stop firewalld` $\to$ 解決せず。

### 最終的な解決策

原因は、`NetworkManager`が管理する`/etc/resolv.conf`（`nameserver 127.0.0.1`）と、`curl`などが使用するOSのリゾルバ（`nsswitch.conf`の`dns`設定）の連携が不安定になっていることでした。

そこで、`nsswitch.conf`の`hosts: files dns`という設定（DNSより先に`files`を見る）を利用し、OSに名前解決を強制的に認識させました。

```bash
# .100 と .103 の両方で実行
# /etc/hosts ファイルに以下を追記
192.168.56.103  host4.example1.jp
```

## 6. まとめと今後の展望
今回のハンズオンを通じて、単なるサーバー構築の手順だけでなく、多層防御の設計思想や、digとcurlの挙動の違いからOSリゾルバの問題を切り分けるといった、実務的なトラブルシューティング能力を学ぶことができました。

また、今回の記事では詳細は省きましたが、トラブルシューティングの際は、L3はPing疎通、L4はFirewallの設定を確認するなど、OSI参照モデルの層に分けて順番に原因を探っていくことが大事だと学びました。

今後は、この3層構成をベースに、IaC（Terraform/Ansible）による自動化や、AWS SAAの学習を進め、クラウドとオンプレミスの両方に強いインフラエンジニアを目指して研鑽を積みたいと考えています。
