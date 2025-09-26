# 構成図
![スクリーンショット 2025-08-18 16.31.02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4081358/5523a3dd-eba2-44c3-9d6b-da3755391d1f.png)

# 構築環境
### ホストOS
・macOS Sonoma 15.1（Apple M1チップ）

# サーバー構築
## WordPressサーバー
・Webサーバー：Nginx（latest）
・PHP：8.3
・データベース：MySQL 8.0.40
・CMS：WordPress
### ゲストOS
・Ubuntu 22.04（Multipass上で構築）

## Zabbixサーバー
・Zabbix：5.0.x
・Webサーバー：Nginx（latest）
・PHP：7.4
・データベース：MySQL 8.0.40
### ゲストOS
・Ubuntu 20.04（Multipass上で構築）


## 本環境のディレクトリ構成

```
.
├── ansible.cfg
├── group_vars
│   ├── all.yml
│   ├── dev_wp_servers.yml
│   ├── zabbix_servers.yml
│   └── zabbix-api.yml
├── inventory.ini
├── playbook.yml
└── roles
    ├── mysql
    │   ├── files
    │   │   └── mysqld_custom.cnf
    │   ├── handlers
    │   │   └── main.yml
    │   ├── host_vars
    │   │   ├── wp_server.yml
    │   │   └── zabbix_server.yml
    │   ├── tasks
    │   │   └── main.yml
    │   └── templates
    │       └── root_my.cnf.j2
    ├── nginx
    │   ├── files
    │   │   └── nginx.conf
    │   ├── handlers
    │   │   └── main.yml
    │   ├── tasks
    │   │   └── main.yml
    │   └── templates
    │       ├── dev.techbull.cloud.conf.j2
    │       └── zabbix.techbull.cloud.conf.j2
    ├── php
    │   ├── files
    │   │   └── www.conf
    │   ├── handlers
    │   │   └── main.yml
    │   └── tasks
    │       └── main.yml
    ├── users
    │   ├── tasks
    │   │   └── main.yml
    │   ├── templates
    │   │   └── menta_sudoers.j2
    │   └── vars
    │       └── main.yml
    ├── wordpress
    │   ├── tasks
    │   │   └── main.yml
    │   └── templates
    │       └── wp-config.php.j2
    ├── zabbix-agent
    │   ├── files
    │   │   ├── ss_get_mysql_stats.php
    │   │   └── template_db_mysql.conf
    │   ├── handlers
    │   │   └── main.yml
    │   ├── tasks
    │   │   └── main.yml
    │   └── templates
    │       └── zabbix_agentd.conf.j2
    ├── zabbix-api
    │   └── tasks
    │       └── main.yml
    └── zabbix-server
        ├── handlers
        │   └── main.yml
        ├── tasks
        │   └── main.yml
        ├── templates
        │   ├── zabbix_server.conf.j2
        │   └── zabbix.conf.php.j2
        └── vars
            └── main.yml
```

# 実装
## multipassでサーバーを作成
### 新しい VM を起動
Ubuntu は20.04以下を選択します
```
% multipass launch --name zabbix-server --cpus 2 --memory 4G --disk 10G 20.04
```
### Multipass内で menta ユーザーを作成
作成したサーバに入る
```
% multipass shell zabbix-server
```
ユーザー・グループを追加
```
$ sudo adduser menta
$ sudo usermod -aG sudo menta
```
### 作成したユーザーでパスワードなしでコマンド実行できる様変更

```
$ export TERM=xterm
$ sudo visudo

#表示されたファイルの末尾に以下を追加する
menta ALL=(ALL) NOPASSWD:ALL

#追加後 Ctrl+O → Enter → Ctrl+X で保存して閉じる
```

### MacでSSH鍵ペアを作成
ターミナルに戻り、ssh認証鍵を作成する

```
% cd ~/.ssh
% ssh-keygen -t rsa
```

以下の様な表示が出る
鍵ファイル名をzabbix-server、パスワードを設定せずにエンターを押下する。

```
Generating public/private rsa key pair.
Enter file in which to save the key (/c/Users/PC_User/.ssh/id_rsa): #zabbix-serverと入力
Enter passphrase (empty for no passphrase): #何も入力せず、Enterを押下
Enter same passphrase again: #何も入力せず、Enterを押下
```
接続先情報をconfigに設定

これを行うことで本来だと
ssh -i ~/.ssh/zabbix-server menta@192.168.64.47と入力しないといけないところ
ssh dev.techbull.cloudと短くすることができます

```
vi ~/.ssh/config
#以下を設定
Host dev.techbull.cloud　# 任意のホスト名
Hostname 192.168.64.47　# multipass listにてIPアドレス確認
User menta 
Port 22
IdentityFile ~/.ssh/zabbix-server　# 作成した鍵のファイル
```
### 公開鍵のコピー・sshd_configの設定
VM側で、ターミナルで作成した公開鍵を設定するためメモしておく
```
% cat ~/.ssh/zabbix-server.pub
```

VMに移動し、sshd_configの設定
```
% multipass shell zabbix-server
 
$ sudo su
$ vi /etc/ssh/sshd_config

#表示されたファイルの末尾に以下を追加する
PubkeyAuthentication yes
PermitRootLogin yes
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication no
```
### VMにターミナルで作成した公開鍵配置

ユーザmentaにスイッチし、ターミナル環境で作成した公開鍵を配置する。
※スイッチ前にユーザmentaのホームディレクトリが作成されていないため、作成し、所有者・所有グループを設定する。

```
$ sudo mkdir -p /home/menta
$ sudo chown menta:menta /home/menta
$ sudo su - menta
$ mkdir ~/.ssh
$ vi ~/.ssh/authorized_keys #控えておいた公開鍵を貼り付ける
```
設定を反映させるため、sshdサービスを再起動。
```
$ sudo systemctl restart ssh
```

### VMへssh接続
ターミナル環境に戻り、sshのconfigに設定したホスト名でリモートホストにSSH接続できることを確認する。
入るためのコマンドは違いますが% multipass shell zabbix-server
と同じVMに入ることができます
```
% ssh dev.techbull.cloud
```
## AnsibleにてZabbixの環境構築

### ansible.cfg(設定ファイル)の作成
Anisbleの設定ファイル”ansible.cfg”をansibleディレクトリ直下に作成・設定します。

### `ansible.cfg`

```
[defaults]
interpreter_python = /usr/bin/python3
inventory = ~/git/menta/ansible/inventory.ini
retry_files_enabled = False
log_path = ~/.ansible/ansible.log
host_key_checking = False
gathering = smart

[privilege_escalation]
become = True
become_method = sudo
become_user = root
```
上記で設定していること
・Python 3 を使う
・インベントリは ~/git/menta/ansible/inventory.ini に固定
・リトライファイルは作らない
・ログは ~/.ansible/ansible.log に保存
・SSH ホストキーの確認を無効化
・facts 収集は smart モード
・全タスク、デフォルトで root 権限（sudo）で実行

### `inventory.ini`
```

[dev_wp_servers]
dev.techbull.cloud ansible_user=menta ansible_host=192.168.64.47 ansible_ssh_private_key_file=~/.ssh/dev-wp-server

[zabbix_servers]
zabbix.techbull.cloud ansible_user=menta ansible_host=192.168.64.46 ansible_ssh_private_key_file=~/.ssh/zabbix-server_menta

[zabbix_api]
zabbix-server ansible_host=192.168.64.46

```
今回使用するホストは3つ
- [dev_wp_servers]：Wordpress用
- [zabbix_servers]：zabbix-servers用
- [zabbix_api]：zabbix-api用




### `Playbook`
プレイブックは依存関係の少ない基盤から順番に構築していきます

```
---
- name: Setup users
  hosts: zabbix_servers:dev_wp_servers  
  become: yes
  become_user: root
  become_method: sudo
  vars_files:
    - ./.env.yml
  roles:
    - users
    
- name: Setup Wordpress
  hosts: dev_wp_servers
  gather_facts: false
  vars_files:
    - ./.env.yml
  roles:
    - nginx
    - php
    - mysql
    - wordpress
    - zabbix-agent

- name: Setup Zabbix core components
  hosts: zabbix_servers
  gather_facts: false
  vars_files:
    - ./.env.yml
  roles:
    - nginx
    - php
    - mysql
    - zabbix-server
    - zabbix-agent  

# Zabbixサーバーが立ち上がって、APIが応答できる状態になるまで待つ
- name: Wait for Zabbix API to respond with valid version info
  hosts: zabbix_servers
  gather_facts: false
  tasks:
    - name: Check Zabbix API version
      uri:
        url: https://{{ ansible_host }}/api_jsonrpc.php
        method: POST
        body: '{"jsonrpc":"2.0","method":"apiinfo.version","id":1,"auth":null,"params":{}}'
        headers:
          Content-Type: application/json
        status_code: 200
        validate_certs: no
      register: result
      until: result.json.result is defined
      retries: 10
      delay: 6

- name: Register host to Zabbix via API
  hosts: zabbix_api
  gather_facts: false
  become: false
  vars_files:
    - ./.env.yml
  roles:
    - zabbix-api


```


### zabbix-serverロール

### `zabbix-server/tasks/main.yml`

```
 vars:
    zabbix_version: "5.0"
    zabbix_repo_pkg: "/tmp/zabbix-release_latest_{{ zabbix_version }}+ubuntu20.04_all.deb"
    mysql_root_user: root
    mysql_root_password: "{{ mysql_root_password }}"
    zabbix_db_name: zabbix
    zabbix_db_user: zabbix
    zabbix_db_password: "{{ zabbix_db_password }}"

  tasks:
    # 1. リポジトリ導入
    - name: Download Zabbix release package
      ansible.builtin.get_url:
        url: "https://repo.zabbix.com/zabbix/{{ zabbix_version }}/ubuntu-arm64/pool/main/z/zabbix-release/zabbix-release_latest_{{ zabbix_version }}+ubuntu20.04_all.deb"
        dest: "{{ zabbix_repo_pkg }}"

    - name: Check if Zabbix repo already exists
      ansible.builtin.stat:
        path: /etc/apt/sources.list.d/zabbix.list
      register: zabbix_repo_file

    - name: Install Zabbix release package if repo not set
      ansible.builtin.apt:
        deb: "{{ zabbix_repo_pkg }}"
      when: not zabbix_repo_file.stat.exists

    # 2. 必要なパッケージ導入
    - name: Install Zabbix components
      ansible.builtin.apt:
        name:
          - zabbix-server-mysql
          - zabbix-frontend-php
          - zabbix-nginx-conf
          - zabbix-agent
        state: present
        update_cache: yes

    # 3. DB 準備
    - name: Ensure Zabbix database exists
      community.mysql.mysql_db:
        name: "{{ zabbix_db_name }}"
        state: present
        encoding: utf8
        collation: utf8_bin
        login_user: "{{ mysql_root_user }}"
        login_password: "{{ mysql_root_password }}"

    - name: Ensure Zabbix DB user exists
      community.mysql.mysql_user:
        name: "{{ zabbix_db_user }}"
        password: "{{ zabbix_db_password }}"
        host: localhost
        priv: "{{ zabbix_db_name }}.*:ALL"
        state: present
        login_user: "{{ mysql_root_user }}"
        login_password: "{{ mysql_root_password }}"

    - name: Check if 'users' table exists
      community.mysql.mysql_query:
        login_user: "{{ zabbix_db_user }}"
        login_password: "{{ zabbix_db_password }}"
        login_db: "{{ zabbix_db_name }}"
        query: >
          SELECT COUNT(*) AS count
          FROM information_schema.tables
          WHERE table_schema='{{ zabbix_db_name }}'
          AND table_name='users';
      register: users_table_check

    - name: Import Zabbix DB schema if not initialized
      ansible.builtin.shell: >
        zcat /usr/share/doc/zabbix-server-mysql/create.sql.gz |
        mysql -u {{ zabbix_db_user }} -p{{ zabbix_db_password }} {{ zabbix_db_name }}
      args:
        executable: /bin/bash
      when: users_table_check.query_result[0][0].count | default(0) | int == 0

    # 4. 設定ファイル配置
    - name: Ensure /etc/zabbix/zabbix_server.conf.d exists
      ansible.builtin.file:
        path: /etc/zabbix/zabbix_server.conf.d
        state: directory
        owner: root
        group: zabbix
        mode: '0750'

    - name: Deploy Zabbix server config
      ansible.builtin.template:
        src: zabbix_server.conf.j2
        dest: /etc/zabbix/zabbix_server.conf
        owner: root
        group: zabbix
        mode: '0640'
      notify: restart zabbix-server

    - name: Deploy Zabbix nginx site config
      ansible.builtin.template:
        src: ../../nginx/templates/zabbix.techbull.cloud.conf.j2
        dest: /etc/nginx/sites-available/zabbix
      notify: reload nginx

    - name: Disable default nginx site
      ansible.builtin.file:
        path: /etc/nginx/sites-enabled/default
        state: absent
      notify: reload nginx

    - name: Enable Zabbix nginx site
      ansible.builtin.file:
        src: /etc/nginx/sites-available/zabbix
        dest: /etc/nginx/sites-enabled/zabbix
        state: link
        force: yes
      notify: reload nginx

    - name: Deploy Zabbix frontend DB config (zabbix.conf.php)
      ansible.builtin.template:
        src: zabbix.conf.php.j2
        dest: /etc/zabbix/web/zabbix.conf.php
        owner: www-data
        group: www-data
        mode: '0640'

    # 5. サービス管理
    - name: Start and enable Zabbix services
      ansible.builtin.systemd:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop:
        - zabbix-server
        - zabbix-agent

    # 6. 補助ツール
    - name: Ensure python3-pip is installed
      ansible.builtin.apt:
        name: python3-pip
        state: present
        update_cache: yes

    - name: Install zabbix-api via pip
      ansible.builtin.pip:
        name: zabbix-api
        executable: pip3
        state: present

    - name: Ensure /var/log/zabbix exists
      ansible.builtin.file:
        path: /var/log/zabbix
        state: directory
        owner: zabbix
        group: zabbix
        mode: '0755'

```
1.リポジトリ導入
　get_url で Zabbix リポジトリパッケージをダウンロード
　stat で既に設定済みか確認
 　未設定なら apt で .deb をインストール


2.Zabbix コンポーネント導入
　zabbix-server-mysql, frontend-php, nginx-conf, agent を apt インストール
　update_cache: yes で最新のパッケージ取得

3.DB 準備
　mysql_db で DB 作成
　mysql_user でユーザー作成＆権限付与
　mysql_query で users テーブルの存在確認　
　初回のみ zcat + mysql でスキーマを流し込み

4.設定ファイル配置
　zabbix_server.conf をテンプレートで配置
　nginx の Zabbix サイト設定を設置、default を無効化
　PHP フロントエンドの DB 設定ファイルを配置

5.サービス管理
　zabbix-server と zabbix-agent を起動＆自動起動設定

6.補助ツール
Zabbix を管理するための環境準備
　python3-pip を導入
　API 操作用に zabbix-api をインストール
　ログディレクトリ /var/log/zabbix を確実に作成

 ### `templates/zabbix_server.conf.j2`

```
LogFile=/var/log/zabbix/zabbix_server.log
PidFile=/run/zabbix/zabbix_server.pid

DBHost=localhost
DBName={{ zabbix_db_name }}
DBUser={{ zabbix_db_user }}
DBPassword={{ zabbix_db_password }}

Include=/etc/zabbix/zabbix_server.conf.d/*.conf
```

### `templates/zabbix.conf.php.j2`

```jsx
<?php
// Zabbix GUI DB config
global $DB;

$DB['TYPE']     = 'MYSQL';
$DB['SERVER']   = 'localhost';
$DB['PORT']     = '0';
$DB['DATABASE'] = '{{ zabbix_db_name }}';
$DB['USER']     = '{{ zabbix_db_user }}';
$DB['PASSWORD'] = '{{ zabbix_db_password }}';

$ZBX_SERVER      = 'localhost';
$ZBX_SERVER_PORT = '10051';
$ZBX_SERVER_NAME = 'Zabbix Server';

$IMAGE_FORMAT_DEFAULT = IMAGE_FORMAT_PNG;
```

---

### `handlers/main.yml`

```yaml
- name: restart zabbix-server
  systemd:
    name: zabbix-server
    state: restarted

- name: reload nginx
  systemd:
    name: nginx
    state: reloaded

```

---

### zabbix-agentロール

### `zabbix-agent/tasks/main.yml`

```
# 1. Zabbixリポジトリ導入
- name: Download Zabbix release package
  ansible.builtin.get_url:
    url: https://repo.zabbix.com/zabbix/5.0/ubuntu-arm64/pool/main/z/zabbix-release/zabbix-release_latest_5.0+ubuntu24.04_all.deb
    dest: /tmp/zabbix-release.deb

- name: Check if Zabbix repo already exists
  ansible.builtin.stat:
    path: /etc/apt/sources.list.d/zabbix.list
  register: zabbix_repo_file

- name: Install Zabbix release package if repo not set
  ansible.builtin.apt:
    deb: "{{ zabbix_repo_pkg }}"
  when: not zabbix_repo_file.stat.exists
  
  
# 2. Zabbix Agent インストール
- name: Install Zabbix agent
  ansible.builtin.apt:
    name: zabbix-agent
    state: present
    update_cache: true

# 3. Zabbix Agent 設定
- name: Configure Zabbix agent server IPs
  ansible.builtin.template:
    src: zabbix_agentd.conf.j2
    dest: /etc/zabbix/zabbix_agentd.conf
  notify: Restart zabbix agent

- name: Copy template_db_mysql.conf
  ansible.builtin.copy:
    src: template_db_mysql.conf
    dest: /etc/zabbix/zabbix_agentd.d/template_db_mysql.conf
    owner: root
    group: root
    mode: '0644'
  notify: Restart zabbix agent

# 4. MySQL監視スクリプト配置
- name: Ensure /var/lib/zabbix directory exists
  ansible.builtin.file:
    path: /var/lib/zabbix
    state: directory
    owner: zabbix
    group: zabbix
    mode: '0750'

- name: Copy ss_get_mysql_stats.php
  ansible.builtin.copy:
    src: ss_get_mysql_stats.php
    dest: /var/lib/zabbix/ss_get_mysql_stats.php
    owner: zabbix
    group: zabbix
    mode: '0755'

- name: Allow zabbix to run ss_get_mysql_stats.php without password
  ansible.builtin.copy:
    dest: /etc/sudoers.d/zabbix
    content: "zabbix ALL=(ALL) NOPASSWD: /var/lib/zabbix/ss_get_mysql_stats.php *\n"
    owner: root
    group: root
    mode: '0440'

# 5. MySQLユーザー作成
- name: Create Zabbix monitor user
  community.mysql.mysql_user:
    name: zbx_monitor
    host: localhost
    password: "{{ lookup('env', 'ZABBIX_MONITOR_PASSWORD') }}"
    priv: '*.*:PROCESS,SHOW DATABASES,REPLICATION CLIENT,SHOW VIEW,SELECT'
    state: present
    login_user: root
    login_unix_socket: /var/run/mysqld/mysqld.sock
    update_password: on_create

# 6. Zabbix用MySQL設定ファイル
- name: Copy .my.cnf to Zabbix user's home
  ansible.builtin.template:
    src: .my.cnf.j2
    dest: /var/lib/zabbix/.my.cnf
    owner: zabbix
    group: zabbix
    mode: '0600'

# 7. サービス管理
- name: Ensure zabbix-agent is running
  ansible.builtin.systemd:
    name: zabbix-agent
    state: started
    enabled: true

```


1.リポジトリ導入
　zabbix-serverロールと同じ

2.Zabbix Agent インストール
　監視対象サーバー(WordPress)にZabbixから情報を送るためにAgentが必要。
 
3.Zabbix Agent設定
　Zabbixサーバーと通信するための設定を反映

4.MySQL監視スクリプト導入
　ZabbixからMySQLのステータスを監視するスクリプトを配置

5.MySQL監視用ユーザー作成
　MySQLの監視専用ユーザー zbx_monitor を作成

6.Zabbix用MySQL設定ファイル配置
　監視スクリプトやエージェントがMySQLに安全に接続できるよう .my.cnf を配置


7.サービス管理
　Zabbixエージェントを常時稼働させる

### `zabbix-agent/files/ss_get_mysql_stats.php`
内容：Zabbix公式が配布している MySQL監視用スクリプト (PHP)
```
curl -o roles/zabbix-agent/files/ss_get_mysql_stats.php \
"https://git.zabbix.com/projects/ZBX/repos/zabbix/raw/templates/db/mysql_agent/ss_get_mysql_stats.php?at=refs/heads/release/5.0"
```

### `zabbix-agent/files/template_db_mysql.conf`
内容：Zabbix Agent の設定追加ファイル
```
UserParameter=mysql.ping,sudo /var/lib/zabbix/ss_get_mysql_stats.php ping
UserParameter=mysql.uptime,sudo /var/lib/zabbix/ss_get_mysql_stats.php uptime
UserParameter=mysql.threads,sudo /var/lib/zabbix/ss_get_mysql_stats.php threads

UserParameter=mysql.deadlocks,sudo /var/lib/zabbix/ss_get_mysql_stats.php deadlocks
```


### `zabbix-agent/handlers/main.yml`
```
---
- name: Restart zabbix agent
  ansible.builtin.systemd:
    name: zabbix-agent
    state: restarted
```

### `zabbix-agent/templates/.my.cnf.j2`
```
[client]
user=zbx_monitor
password={{ lookup('env', 'ZABBIX_MONITOR_PASSWORD') }}
host=localhost
socket=/var/run/mysqld/mysqld.sock
```

### `zabbix-agent/templates/zabbix_agentd.conf.j2`
```
Server=127.0.0.1,192.168.64.46       
ServerActive=192.168.64.46  

LogFile=/var/log/zabbix/zabbix_agentd.log
LogType=file
PidFile=/var/run/zabbix/zabbix_agentd.pid
Include=/etc/zabbix/zabbix_agentd.d/*.conf
```

---

### zabbix-APIロール

### `/zabbix-api/tasks/main.yml`


```
# 1. ホスト作成
- name: create host dev.menta.me
  community.zabbix.zabbix_host:
    host_name: dev.menta.me　　＃Zabbix 上での識別名。Zabbix Web UI に表示される。
    host_groups:
      - Linux servers　　　　　　＃管理上の分類（ホストグループ）。アラート設定や権限管理に使える。
    link_templates:
      - Template DB MySQL by Zabbix agent
      - Template OS Linux by Zabbix agent
    status: enabled
    state: present
    interfaces:
      - type: 1
        main: 1
        useip: 1
        ip: 監視対象サーバー（＝Zabbixエージェントをインストールしている側）のIPアドレス
        dns: ""
        port: 10050


# 2. ホスト情報の取得 & インターフェースID設定
- name: Get host info from Zabbix
  community.zabbix.zabbix_host_info:
    host_name: dev.menta.me
  register: host_info

- name: Set interface ID
  set_fact:
    iface_id: "{{ host_info.hosts[0].hostinterfaces[0].interfaceid }}"

# 3. アイテム作成（監視項目の追加）
- name: Create disk usage (percent used) item
  community.zabbix.zabbix_item:
    host_name: dev.menta.me
    name: Disk space used on /
    params:
      key: vfs.fs.size[/,pused]
      type: zabbix_agent
      value_type: numeric_unsigned
      delay: 30s
      interfaceid: "{{ iface_id }}"
    state: present

- name: create nginx process item
  community.zabbix.zabbix_item:
    host_name: dev.menta.me
    name: Nginx process count
    params:
      key: proc.num[nginx]
      type: zabbix_agent
      value_type: numeric_unsigned
      delay: 30s
      interfaceid: "{{ iface_id }}"
    state: present

- name: create mysqld process item
  community.zabbix.zabbix_item:
    host_name: dev.menta.me
    name: MySQL process count
    params:
      key: proc.num[mysqld]
      type: zabbix_agent
      value_type: numeric_unsigned
      delay: 30s
      interfaceid: "{{ iface_id }}"
    state: present

- name: Create Zabbix item for MySQL deadlock detection
  community.zabbix.zabbix_item:
    host_name: dev.menta.me
    name: MySQL Deadlock Count
    params:
      key: mysql.deadlocks["{$MYSQL.HOST}","{$MYSQL.PORT}"]
      type: zabbix_agent
      value_type: numeric_unsigned
      delay: 60s
      interfaceid: "{{ iface_id }}"
    state: present

# 4. トリガー作成
- name: create zabbix trigger
  community.zabbix.zabbix_trigger:
    name: Disk usage over 90%
    host_name: dev.menta.me
    params:
      severity: high
      expression: "{dev.menta.me:vfs.fs.size[/,pused].last()}>=90"
      manual_close: false
      enabled: true
    state: present

# 5. グローバルマクロ設定
- name: Set global macro {$ZABBIX.URL}
  community.zabbix.zabbix_globalmacro:
    macro_name: "{$ZABBIX.URL}"
    macro_value: "http://192.168.64.46/zabbix/"
    macro_type: text
    state: present
```

1.ホスト作成 
　監視対象の追加



2.ホスト情報取得 
　interfaceid を取得
　interfaceidを変数にセット→3で使用する

3.各種監視アイテム作成　
　ディスク・プロセス・DB監視を設定

4.必要なトリガー作成
　Disk usage

5.グローバルマクロ設定

## 全体のイメージ図
複雑になってきたので、上記を設定したものを可視化しました
```
           ┌───────────────────────┐
           │   Zabbix Server       │
           │                       │
           │  ┌───────────────┐    │
           │  │ Host: dev.menta.me │
           │  │ Group: Linux servers│
           │  │ Templates:          │
           │  │ - Template OS Linux │
           │  │ - Template DB MySQL │
           │  └───────────────┘    │
           │         │             │
           │         │ (接続: Agent)
           │         ▼
           │   IP: 192.168.64.47  │
           │   WordPressサーバー │
           │   (Zabbix Agent動作) │
           └───────────────────────┘

```
