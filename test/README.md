# nkid-ansible

## 階層

```
nkid-ansible
|-- ansible.cfg                           # ansible設定ファイル
|-- site.yml                              # マスターplaybook  
|-- webserver.yml                         # WEB/APのplaybook  
|-- batchserver.yml                       # バッチのplaybook  
|-- inventory
|   |-- prod
|   |   |-- prod                          # 本番環境のインベントリファイル
|   |   `-- group_vars                    # 本番環境用の変数設定
|   |       |-- web.yml
|   |       `-- batch.yml
|   |-- dev
|   |   |-- dev                           # 開発環境共通(バッチ)のインベントリファイル
|   |   `-- group_vars                    # 開発環境共通(バッチ)用の変数設定
|   |       `-- batch.yml
|   |-- dev_s
|   |   |-- dev_s                         # 開発環境(S環境)のインベントリファイル
|   |   `-- group_vars                    # 開発環境(S環境)用の変数設定
|   |       `-- web.yml
|   |-- dev_a
|   |   |-- dev_a                         # 開発環境(A環境)のインベントリファイル
|   |   `-- group_vars                    # 開発環境(A環境)用の変数設定
|   |       `-- web.yml
|   |-- dev_b
|   |   |-- dev_b                         # 開発環境(B環境)のインベントリファイル
|   |   `-- group_vars                    # 開発環境(B環境)用の変数設定
|   |       `-- web.yml
|   |-- dev_c
|   |   |-- dev_c                         # 開発環境(C環境)のインベントリファイル
|   |   `-- group_vars                    # 開発環境(C環境)用の変数設定
|   |       `-- web.yml
|   `-- dev3
|       |-- dev3                         # 開発環境(DEV3環境)のインベントリファイル
|       `-- group_vars                   # 開発環境(DEV環境)用の変数設定
|           `-- web.yml
|
`-- roles
    |-- basic                    　       # OS基本設定のrole
    |   |-- files                         # 配布用のコンフィルを配置※用途毎
    |   |   |-- batch
    |   |   |   `-- logrotate
    |   |   |       |-- apl
    |   |   |       `-- httpd
    |   |   `-- web
    |   |       `-- logrotate
    |   |           |-- httpd
    |   |           |-- kiban
    |   |           `-- postfix_application_log
    |   |-- handlers
    |   |   `-- main.yml                   # 設定変更を反映
    |   `-- tasks
    |       |-- main.yml                   # メインのplaybook
    |       |-- os_setting.yml             # hostname,locale等
    |       |-- useradd.yml                # user/groupadd
    |       |-- bashrc.yml                 # bashrc変更
    |       |-- sudoers.yml                # sudoers変更
    |       |-- sshd.yml                   # sshd_config変更
    |       |-- logrotate.yml              # logrotateファイル配布
    |       |-- chrony.yml                 # Amazon Time Sync追加
    |       `-- cloud-init.yml             # cloud.cfg変更
    |
    |-- cloudwatch-logs                    # CloudWatch Log設定のrole
    |   |-- files                          # 配布用のコンフィルを配置※用途毎
    |   |   |-- batch
    |   |   |   `-- awslogs.conf
    |   |   `-- web
    |   |       `-- awslogs.conf
    |   |-- handlers
    |   |   `-- main.yml                    # cloudwatch-logs-agentインストール後に起動
    |   `-- tasks
    |       |-- main.yml                    # メインのplaybook
    |       |-- install.yml                 # cloudwatch-logs-agentインストール
    |       `-- cleanup.yml                 # 不要ファイル削除
    |
    |-- efs-utils                           # EFSヘルパーツール設定のrole
    |   |-- files                           # rpmのコンパイルが必要なため予めコンパイルしたものを配置
    |   |   `-- amazon-efs-utils-x.xx.x-x.el7.noarch.rpm
    |   |-- handlers
    |   |   `-- main.yml                    # efs-utilsインストール後に起動
    |   `-- tasks
    |       |-- main.yml                    # メインのplaybook
    |       |-- install.yml                 # efs-utilsダウンロード/インストール
    |       `-- cleanup.yml                 # 不要ファイル削除
    |
    `-- yum                                 # パッケージインストールのrole
        |-- handlers
        |   `-- main.yml                    # パッケージインストール後に起動
        `-- tasks
            `-- main.yml                    # メインのplaybook
```

### inventory

環境毎に変数を設定(group_varsで設定)したい為、環境毎にディレクトリ構成を分ける。
hostname等の固定値については、インベントリファイル内のvarsで変数を指定。

```
[web]
xx.xx.xx.xx 
xx.xx.xx.xx

[batch]
localhost                       # バッチサーバ上でデプロイを実施するため、自分自身を指定

[all:vars]
aws_region=ap-northeast-1       # regionの指定

[web:vars]
server_group=web                # グループ名を指定 ※copyモジュールのパスに指定
host_name=nkid1waptyoas         # 共通のホスト名を指定

[batch:vars]
server_group=batch
host_name=nkid1battyoas
```

### group_vars

環境名のディレクトリ配下にgroup_varsディレクトリを作成。
group_vars配下のグループ毎(Web,batch)のファイル配置。※グループ名と同じファイルの変数が設定される。
また、OSユーザのパスワード情報が含まれるため、```ansible-vault```で暗号化する。

```
$ ansible-vault create <ファイル名>
```

### playbook

バッチサーバへの実行はローカル実行となるため、playbookに```connection: local```を記載。

```
---
- hosts: batch
  connection: local
  become: yes
    　　：
```

### roles

OS基本設定は一括りとし、その他はインストールするミドルウェア毎にroleを作成。
※yumでインストールする基本パッケージはyumのroleで一括りにする。
サーバの用途(web,batch)毎に設定する項目に差がある場合はinventoryのグループ名で条件分岐。


```
- name: install common package for web
  yum:
    name={{ packages }}
    state=latest
  vars:
    packages:
    - httpd
    　　：
  when:
   - inventory_hostname in groups["web"]      <---条件分岐 ※webグループのみ実行
```

## 実行方法

**書式**

```
$ ansible-playbook -i inventory/<環境名>/<playbook名> --ask-become-pass --ask-vault-pass
BECOME password: [sshパスワード]
Vault password: [ansible-vaultの復号化パスワード]
```

**本番環境WEB/APの場合**

```
$ ansible-playbook -i inventory/prod webserver.yml --ask-become-pass --ask-vault-pass
```

**本番環境バッチの場合**

```
$ ansible-playbook -i inventory/prod batchserver.yml --ask-become-pass --ask-vault-pass
```

**本番環境全ての場合**

```
$ ansible-playbook -i inventory/prod site.yml --ask-become-pass --ask-vault-pass
```

