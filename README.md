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

「役割の分離（Web/DNS/DB）」「多層防御（FW/DB認証）」「DNSの権限委譲」の3点を意識して設計しました。

![system_architecture_diagram.drawio.png]


<img width="739" height="897" alt="system_architecture_diagram drawio" src="https://github.com/user-attachments/assets/df20969b-c97b-41a0-9b1a-2d873304a78c" />

### サーバー一覧（役割とIP）

| 役割 | ホスト名 | IPアドレス | 主な役割 |
| :--- | :--- | :--- | :--- |
| DNSキャッシュ | `host0.jp` | `192.168.56.100` | DNSキャッシュ、jpゾーン管理 |
| DNSサーバー1 | `host1.example1.jp` | `192.168.56.101` | example1.jp ゾーン管理 |
| DNSサーバー2 | `host2.example2.jp` | `192.168.56.102` | example2.jp ゾーン管理 |
| **Webサーバー** | `host4.example4.jp` | `192.168.56.103` | Apache (httpd) (アクセス名: host4.example1.jp)|
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

## 4. 最大の難関：`dig`は成功するのに`curl`が失敗した話

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

## 5. まとめと今後の展望
今回のハンズオンを通じて、単なるサーバー構築の手順だけでなく、多層防御の設計思想や、digとcurlの挙動の違いからOSリゾルバの問題を切り分けるといった、実務的なトラブルシューティング能力を学ぶことができました。

また、今回の記事では詳細は省きましたが、トラブルシューティングの際は、L3はPing疎通、L4はFirewallの設定を確認するなど、OSI参照モデルの層に分けて順番に原因を探っていくことが大事だと学びました。

今後は、この3層構成をベースに、IaC（Terraform/Ansible）による自動化や、AWS SAAの学習を進め、クラウドとオンプレミスの両方に強いインフラエンジニアを目指して研鑽を積みたいと考えています。
