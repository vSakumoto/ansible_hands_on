## Ansible + Testkitchen(serverspec)

------

## driver docker


## 環境構築（MAC）

```
$ bundler -v
Bundler version 1.12.5
$ ruby -v
ruby 2.1.5
$ docker -v
Docker version 1.12.0-rc2
または
Docker version 1.11.2
```

前提として ruby 2.1.1<で話を進めていくため、以下を事前にインストールすることを推奨とする(Mac)
※gemの関係上provisionerにansible_playbookが認識できない事象あり

```
brew update
brew install rbenv ruby-build
brew update
brew install rbenv ruby-build　（もしくは、brew update rbenv ruby-build）

rbenv install 2.1.5
rbenv global 2.1.5
rbenv rehash

# PATH に追加
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile

# .bash_profile に追加
echo 'eval "$(rbenv init -)"' >> ~/.bash_profile

# 上記設定の再読み込み
exec $SHELL -l

ruby -v
ruby 2.1.5p273 (2014-11-13 revision 48405) [x86_64-darwin14.0]

rbenv exec gem install bundler
Bundler version 1.12.5

```

## dockerのインストール

```
brew cask install virtualbox
brew install docker
brew install docker-machine
docker -v
Docker version 1.11.2, build b9f10c9

# 初期設定
docker-machine create -d virtualbox ansible-hands-on
eval $(docker-machine env ansible-hands-on)
# VPNソフトを起動しているとエラーが出る場合があるよ！FAQ参照

```


------

## 利用方法

```
git clone https://github.com/vSakumoto/ansible_hands_on.git ~/ansible
cd ~/ansible

bundle install
```

------

## TestKitchenでのテスト（対象Docker）

```
bundle exec kitchen list #作成予定のコンテナ
bundle exec kitchen create # コンテナの作成
bundle exec kitchen converge # Provisioner実行
bundle exec kitchen verify # serverspec実行
bundle exec kitchen destroy #コンテナの削除

bundle exec kitchen test #create /converge / verify 実行
```

-----

## FAQ

### kitchen + docker でsshできなくなった場合

```
rm -rf .kitchen/
docker rm -f containerid  #不要なDocker プロセスを削除
docker rmi -f imageid #pullしたbaseイメージを削除

bundle exec kitchen create

```

### VPNを繋いでいると上手く立ち上がらないよ


```
Cannot access docker when running VPN (Cisco AnyConnect)
· Issue #628
· boot2docker/boot2docker : https://github.com/boot2docker/boot2docker/issues/628

上記のため、一度VPNを切断し、対象のコンテナを削除

docker-machine rm -f [NAME]
```

### プロビジョニング実行したDockerの中に入ってみたい！

```
cd ansible
bundle exec kitchen list
Instance             Driver  Provisioner      Verifier  Transport  Last Action
default-ubuntu-1404  Docker  AnsiblePlaybook  Busser    Ssh        Verified

bundle exec kitchen login <InstanceName>

若しくは！

docker ps #対象のホストを確認
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                   NAMES
793943b0374e        93e9f788193a        "/usr/sbin/sshd -D -o"   3 hours ago         Up 3 hours          0.0.0.0:32787->22/tcp   nostalgic_euler

docker exec -it <CONTAINER ID> /bin/bash

以下は推奨しません。
## cd ansible/.kitchen/
## ssh -p 32787 -i id_rsa kitchen@localhost
```


------

## TestKitchenターゲットの設定


```
driver:
  name: docker
  use_sudo: false

provisioner:
  name: ansible_playbook
  playbook: site.yml
  roles_path: ./roles
  group_vars_path: ./group_vars
  host_vars_path: ./host_vars
  additional_copy_path:
    - webservers.yml
  hosts: webservers
  dns:
    - 8.8.8.8
    - 8.8.4.4
  #require_ansible_repo: true
  ansible_verbose: true
  require_ansible_omnibus: true
  require_ruby_for_busser: true

#
# 本来であればansible_omnibus / busserで入るはずが動作しない。
# 以下で強制的にインストールを実行
#
platforms:
  - name: ubuntu-14.04
    driver_config:
      #socket: tcp://localhost:2375 # <= リモート先に入れたい場合に指定をするlocalは不要
      username: kitchen
      privileged: true
      run_command: /sbin/init; sleep 3
      provision_command:
        - apt-get -y install apt-utils
        - apt-get -y install software-properties-common python-software-properties
        - add-apt-repository ppa:rquillo/ansible
        - curl -L https://www.opscode.com/chef/install.sh | bash
        - apt-get update
        - apt-get -y install ansible python-selinux
        - apt-get install -y build-essential bash debianutils dnsutils net-tools telnet tar cron

verifier:
  ruby_bindir: '/usr/bin'

suites:
  - name: default
    attributes:
```


## Directory構成

```

├── Gemfile
├── hosts / development / production # inventory 各種サーバー情報
├── site.yml
├── webservers.yml
├── .kitchen.yml
├── roles
│   └── common
│       ├── files
│       │   ├── authorized_keys_for_xxx
│       │   └── sshd_config
│       ├── tasks
│       │   └── main.yml
│       │
│       ├── handlers
│       │   └── main.yml
│       └── templates
│           └── iptables
└── test
    └── integration
        └── default
             ├─── .kitchen
             └─── serverspec

```


