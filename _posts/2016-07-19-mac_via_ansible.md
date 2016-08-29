---
layout: post
title: Mac via Ansible
date: 2016-07-19
tags: japanise
category: 
 - osx
 - ansible
---

# OSX を Ansible で自動構築する（El Capitan）


## 事前準備（インストール、設定）


### OSX のアップデート実施、App Storeからインストール

- Stufflt Expander 16  
- Photo Scape X  
- LINE  
- Mincosoft Remote Desktop  
- Xcode  
- iMove  
- iPhoto  
- Keynote  
- Numbres  


##### [Xcode- 統合開発環境 (IDE) ](https://developer.apple.com/xcode/jp/)  
    $ sudo xcodebuild -license
    By typing 'agree' you are agreeing to the terms of the software license agreements. Type 'print' to print them or anything else to cancel, [agree, print, cancel] agree ← agreeと入力

   <span style="color:gray">※コマンドラインデベロッパーツールのインストール確認が表示されるので、インストールを選択します。その後、ライセンスの確認も表示されるので、わざわざXcodeを立ち上げる必要もありません。</span>


##### [Homebrew - パッケージ管理](http://brew.sh/index_ja.html)  

    $ ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"  
    $ echo -e '##### Brew-caskでインストールされるアプリを /Applicationsにリンクする #####' \\n'export HOMEBREW_CASK_OPTS="--appdir=/Applications"' \\n >> ~/.bash_profile ; more ~/.bash_profile


###  [git - バージョニング管理](https://git-scm.com)  
    $ brew install git
    $ git config --global user.name ymiyamo
    $ git config --global user.email ymiyamo@*****.**.**


### [SSH - セキュアシェル](https://ja.wikipedia.org/wiki/Secure_Shell)  
   <span style="color:gray">※システム環境設定 > 共有 > リモートログイン を有効化する  
   ※鍵での認証が必要となるため、鍵を作成、設定する。</span>

    $ cd ~/.ssh
    $ ssh-keygen
    Generating public/private rsa key pair.
    Enter file in which to save the key (/Users/miamo/.ssh/id_rsa):
    Enter passphrase (empty for no passphrase):
    Enter same passphrase again:
    Your identification has been saved in /Users/miamo/.ssh/id_rsa.
    Your public key has been saved in /Users/miamo/.ssh/id_rsa.pub.
    The key fingerprint is:
    $ mv id_rsa.pub authorized_keys
    $ echo -e "Host miamo \n  HostName localhost \n  IdentityFile ~/.ssh/id_rsa \n  User miamo" > config ; more config

### [Ansible - 構成管理ツール](https://www.ansible.com/)  

    $ brew update
    Already up-to-date.
    $ brew doctor
    Your system is ready to brew.
    $ brew install ansible
    $ mkdir -p /usr/local/etc/ansible/
    $ ln -s /usr/local/etc/ansible/ ~/ansible
    $ cd ~/ansible
    $ echo -e "- hosts: localhost\n  connection: local\n  gather_facts: no\n  become: no\n  tasks:\n    # update homebrew\n    - name: update homebrew\n      homebrew: update_homebrew=yes" > brew_update.yml ; more brew_update.yml
    $ echo -e "[laptop]\n localhost" > hosts ; more hosts
    $ ansible-playbook brew_update.yml

#### [Ansible - install_brew.yml](https://www.ansible.com/)  

    - hosts: localhost
      connection: local
      gather_facts: no
      become: no
      vars:
        homebrew_taps:
          - caskroom/cask
          
        homebrew_packages:
          - name: awscli 
          - name: git
          - name: jq 
          - name: pyenv-virtualenv
          - name: rbenv
          - name: ruby-build
          - name: wireshark 

        homebrew_cask_packages:
          - atom
          - caffeine
          - cheatsheet
          - cyberduck
          - dropbox
          - evernote
          - google-chrome
          - google-japanese-ime
          - iterm2
          - kindle
          - libreoffice
          - macdown
          - scroll-reverser
          - skitch
          - skype
          - slack
          - sourcetree
          - synology-assistant
          - timemachineeditor
          - vagrant
          - vagrant-manager
          - virtualbox
          - virtualbox-extension-pack
          - vlc

      tasks:
        # brew tap
        - name: install taps of homebrew
          homebrew_tap: tap="{{ item }}" state=present
          with_items: "{{homebrew_taps}}"

        # brew instal
        - name: update homebrew
          homebrew: update_homebrew=yes
        - name: install homebrew packages
          homebrew: name="{{ item.name }}" state="{{ item.state|default('latest') }}" install_options="{{ item.install_options|default() }}"
          with_items: "{{homebrew_packages}}"

        # brew cask install
        - name: install homebrew cask packages
          homebrew_cask: name="{{ item }}" state=present
          with_items: "{{homebrew_cask_packages}}"

#### [Ansible - install_pip.yml](https://www.ansible.com/)  

    - hosts: localhost
      connection: local
      gather_facts: true
      become: true
      vars:
        pip_packages:

      tasks:
      - name: install easy_install packages
        easy_install: name=pip
      - name: upgrade pip
        shell: pip install -U pip
        register: upgrade_pip_result
        changed_when: '"Requirement already up-to-date" not in upgrade_pip_result.stdout'
      - name: pip install some packages
        pip: name='{{ item }}'
        with_items: "{{pip_packages}}"

#### [Ansible - symlink.yml](https://www.ansible.com/)  

    - hosts: localhost
      connection: local
      gather_facts: true
      become: no
      vars:
        create:
          - /Users/miamo/git/ymiyamo/
          - /Users/miamo/USB SP64/
          - /Users/miamo/USB SP64/screenshot
          - /Users/miamo/USB SP64/terminal-log

        symlink:
          - { src: '/Users/miamo/USB SP64/screenshot', dest: '/Users/miamo/screenshot' }
          - { src: '/Users/miamo/USB SP64/terminal-log', dest: '/Users/miamo/terminal-log' }

      tasks:
      - name: create a directory if it doesn't exist
        file: path='{{ item }}' state=directory owner=miamo group=admin mode=0755
        with_items: "{{create}}"

      - name : symlink a directory
        file: src={{ item.src }} dest={{ item.dest }} state=link
        with_items: "{{ symlink }}"

#### [Ansible - osx_defaults.yml](https://www.ansible.com/)  


    - hosts: localhost
      connection: local
      gather_facts: true
      become: no
      vars:
        settings:
          # Dockを自動的に隠す
          - { domain: 'com.apple.dock', key: 'autohide', type: 'bool', value: 'true' }
          # Dockのサイズを変更する
          - { domain: 'com.apple.dock', key: 'tilesize', type: 'int', value: '40' }
          # trackpadのアプリケーションExposeを有効
          - { domain: 'com.apple.dock', key: 'showAppExposeGestureEnabled', type: 'boolean', value: 'true' }
          # 隠しファイルや隠しフォルダを表示
          - { domain: 'com.apple.finder', key: 'AppleShowAllFiles', type: 'bool', value: 'true' }
          # Finderのタイトルバーにフルパスを表示
          - { domain: 'com.apple.finder', key: '_FXShowPosixPathInTitle', type: 'bool', value: 'true'}
          # Show Status bar in Finder （ステータスバーを表示）
          - { domain: 'com.apple.finder', key: 'ShowStatusBar', type: 'bool', value: 'true' }
          # Show Path bar in Finder （パスバーを表示）
          - { domain: 'com.apple.finder', key: 'ShowPathbar', type: 'bool', value: 'true' }
          # Show Tab bar in Finder （タブバーを表示）
          - { domain: 'com.apple.finder', key: 'ShowTabView', type: 'bool', value: 'true'}
          # "Finder を終了"を表示
          - { domain: 'com.apple.finder', key: 'QuitMenuItem', type: 'bool', value: 'true'}
          # デスクトップに表示する項目選択(外部ディスク、ハードディスク、接続しているサーバ、CD，DVD，および iPod)  張子を表示
          - { domain: 'com.apple.finder', key: 'ShowExternalHardDrivesOnDesktop', type: 'bool', value: 'true'}
          - { domain: 'com.apple.finder', key: 'ShowHardDrivesOnDesktop', type: 'bool', value: 'true'}
          - { domain: 'com.apple.finder', key: 'ShowMountedServersOnDesktop', type: 'bool', value: 'true'}
          - { domain: 'com.apple.finder', key: 'ShowRemovableMediaOnDesktop', type: 'bool', value: 'true'}
          # ネットワークボリュームに.DS_Storeを作らない
          - { domain: 'com.apple.desktopservices', key: 'DSDontWriteNetworkStores', type: 'bool', value: 'true'}
          # メニューバーに時刻を表示する
          - { domain: 'com.apple.menuextra.clock', key: 'DateFormat', type: 'string', value: 'EEE H:mm'}
          # スクリーンショットのファイル名、格納場所、タイプを変更
          - { domain: 'com.apple.screencapture', key: 'name', type: 'string', value: 'ss'}
          - { domain: 'com.apple.screencapture', key: 'location', type: 'string', value: 'Users/miamo/screenshot'}
          - { domain: 'com.apple.screencapture', key: 'type', type: 'string', value: 'PDF'}
          # メニューバーにバッテリーのパーセントを表示する
          - { domain: 'com.apple.menuextra.battery', key: 'ShowPercent', type: 'string', value: 'YES'}
          # タップでクリックを有効
          - { domain: 'com.apple.AppleMultitouchTrackpad', key: 'Clicking', type: 'bool', value: 'true'}
          - { domain: 'com.apple.driver.AppleBluetoothMultitouch.trackpad', key: 'Clicking', type: 'bool', value: 'true'}
          # Trackpadの調べる＆データ検出を有効
          - { domain: 'com.apple.AppleMultitouchTrackpad', key: 'TrackpadThreeFingerDrag', type: 'boolean', value: 'true'}
          - { domain: 'com.apple.AppleMultitouchTrackpad', key: 'TrackpadThreeFingerHorizSwipeGesture', type: 'int', value: '0'}
          - { domain: 'com.apple.AppleMultitouchTrackpad', key: 'TrackpadThreeFingerTapGesture', type: 'int', value: '2'}
          - { domain: 'com.apple.AppleMultitouchTrackpad', key: 'TrackpadThreeFingerVertSwipeGesture', type: 'int', value: '0'}
          - { domain: 'defaults write com.apple.driver.AppleBluetoothMultitouch.trackpad', key: 'TrackpadThreeFingerDrag', type: 'boolean', value: 'true'}
          - { domain: 'defaults write com.apple.driver.AppleBluetoothMultitouch.trackpad', key: 'TrackpadThreeFingerHorizSwipeGesture', type: 'int', value: '0'}
          - { domain: 'defaults write com.apple.driver.AppleBluetoothMultitouch.trackpad', key: 'TrackpadThreeFingerTapGesture', type: 'int', value: '2'} 
          - { domain: 'defaults write com.apple.driver.AppleBluetoothMultitouch.trackpad', key: 'TrackpadThreeFingerVertSwipeGesture', type: 'int', value: '0'}

        global_settings:
          # コンテクストメニューの表示数を20にする
          - { key: 'NSServicesMinimumItemCountForContextSubmenu', type: 'int', value: '20'}
          # すべてのファイル名拡張子を表示
          - { key: 'AppleShowAllExtensions', type: 'bool', value: 'true'}
          # F1、F2 などのすべてのキーを標準ファンクションキーとして使用
          - { key: 'com.apple.keyboard.fnState', type: 'bool', value: 'true'}
          # メニューを自動で隠す
          - { key: '_HIHideMenuBar', type: 'bool', value: 'true'}

        setting_etc:
          # メニューバー表示を削除
          - { domain: 'com.apple.systemuiserver', key: "menuExtras", type: "array", value: "[]" }
          # メニューバーにバッテリー、bluetouch、Wi-Fi、volumeを表示させる
          - { domain: 'com.apple.systemuiserver', key: 'menuExtras', type: 'array', array_add: 'true', value: "['/System/Library/CoreServices/Menu Extras/Battery.menu']" }
          - { domain: 'com.apple.systemuiserver', key: 'menuExtras', type: 'array', array_add: 'true', value: "['/System/Library/CoreServices/Menu Extras/Bluetooth.menu']" }
          - { domain: 'com.apple.systemuiserver', key: 'menuExtras', type: 'array', array_add: 'true', value: "['/System/Library/CoreServices/Menu Extras/AirPort.menu']" }
          - { domain: 'com.apple.systemuiserver', key: 'menuExtras', type: 'array', array_add: 'true', value: "['/System/Library/CoreServices/Menu Extras/Volume.menu']" }

        shell:
          # メニューバーにバッテリー、bluetouch、Wi-Fi、volumeを表示させる
          - defaults write com.apple.systemuiserver menuExtras -array "/System/Library/CoreServices/Menu Extras/Bluetooth.menu" "/System/Library/CoreServices/Menu Extras/AirPort.menu" "/System/Library/CoreServices/Menu Extras/Battery.menu" "/System/Library/CoreServices/Menu Extras/Clock.menu" "/System/Library/CoreServices/Menu Extras/TextInput.menu" "/System/Library/CoreServices/Menu Extras/Volume.menu"

          # 各種プロセス再起動
          - killall Finder
          - killall Dock
          - killall SystemUIServer

      tasks:
      - name: osx_defaults settings
        osx_defaults:
          domain='{{ item.domain }}'
          key='{{ item.key }}'
          type='{{ item.type }}'
          value='{{ item.value }}'
        with_items: "{{settings}}"

      - name: osx_defaults global settings
        osx_defaults:
          domain=NSGlobalDomain
          key='{{ item.key }}'
          type='{{ item.type }}'
          value='{{ item.value }}'
        with_items: "{{global_settings}}"

      # - name: osx_defaults settings etc
        # osx_defaults:
          # domain='{{ item.domain }}'
          # key='{{ item.key }}'
          # type='{{ item.type }}'
          # array_add='{{ item.array_add | default(false) }}'
          # value='{{ item.value }}'
        # with_items: "{{settings_etc}}"

      - name: osx_defaults Reload
        shell: '{{ item }}'
        register: result
        failed_when: result.rc not in [0, 1]
        with_items: "{{shell}}"


### アカウント設定
   Dropbox, Evernote, slack, sourcetree(git), google, kindle  


### ユーティリティインストール
- [LogiMgr Installer - マウス](http://support.logicool.co.jp/ja_jp/product/bluetooth-mouse-m558-for-mac#download/)

### 環境設定

#### [TrackPad ※なぜか変わらないのでGUIで設定する]
 - タップでクリックを有効  

##### [キーボード ※なぜか変わらないのでGUIで設定する]  
 - キーボードのファンクションキーを標準機能にする  
 - 装飾キー -> caps をcommond に変更  
 - 入力ソース -> google-japanese-imeを追加  

##### [ユーザとグループ]  
 - ゲストにこのコンピュータへのログインを許可(チェックなし)  

##### [共有]
 - コンピュータ名の変更  


# cmd-setting

- ~/.bash_profile編集  

      ##### log <logname>でファイルを~/terminal-logに作成 #####
      function log() {
        now=`date +%Y-%m%d-%H%M`
        script ~/terminal-log/${now}_$1.log
      }

      ##### mac_ch でmacアドレス変更 #####
      function mac_ch() {
        sudo ifconfig en0 ether $(perl -e 'for ($i=0;$i<5;$i++){@m[$i]=int(rand(256));} printf "02:%X:%X:%X:%X:%X\n",@m;')
        sudo ifconfig en0 down
        sudo ifconfig en0 up
        ifconfig en0
      }

      ###### mac_re でmacアドレス復元 #####
      function mac_re() {
        sudo ifconfig en0 ether 3c:15:c2:d3:53:f0
        sudo ifconfig en0 down
        sudo ifconfig en0 up
        sudo ifconfig en0
      }

      ##### bat でバッテリー容量表示 #####
      function bat(){
        ioreg -l | grep MaxCapacity | cut -d " " -f 17,18,19
        ioreg -l | grep CurrentCapacity | cut -d " " -f 17,18,19
        ioreg -l | grep DesignCapacity | cut -d " " -f 17,18,19
      }

      ##### ss <filename> でスクリーンショットファイル名を設定 #####
      function ss(){
        defaults write com.apple.screencapture name $1
        killall SystemUIServer
      }
      ###### ~/USB\ SP64/, /Volumes/USB\ SP64/ 同期 ######
      mkdir -p　~/USB\ SP64
      sudo rsync -av --delete ~/USB\ SP64/ /Volumes/USB\ SP64

      eval "$(rbenv init -)"
      eval "$(pyenv init -)"
      eval "$(pyenv virtualenv-init -)"



### [Atom - テキストエディタ設定](https://atom.io/)  
   [※ 半角空白とか改行とかタブとかを可視化](http://dotinstall.com/lessons/basic_atom/30503)  
   「Shift + Commond + p」->「Settings」->「Core setting」「Show Invisibles」をON  
   [※ 行末のスペース自動削除無効化](http://qiita.com/minamijoyo/items/ad0814ce6eb849400135)  
   「Shift + Commond + p」->「Setting」->「Packages」->「Whitespace」->「Settings ->「Ensure Single Trailing Newline」, 「Remove Trailing whitespace」をOFF  
   [※ ウィンドウ枠内での折り返し有効化](http://qiita.com/nsatohiro/items/33d87cfb7cfca436f3d2)  
   「Shift + Commond + p」->「Settings」->「Core setting」「Soft Wrap」をON  
   [※ japanise-menuをインストール](https://atom.io/packages/japanese-wrap)  
   「Shift + Commond + p」->「Settings」->「Install」->「japanise-menu」をInstall  
   [※ markdown-previewのスタイル変更](https://atom.io/packages/markdown-preview)  
   「Shift + Commond + p」->「Setting」->「Packages」->「MSarkdown-preview」-> 「Use github.com stayl」をON  

> ・参考文献  
- [brew web](http://brew.sh/index_ja.html)  
- [homebrewとは何者か。仕組みについて調べてみた](http://qiita.com/omega999/items/6f65217b81ad3fffe7e6)  
- [アプリケーションのインストールを楽に！](http://blog.lexues.co.jp/25290.html)  
- [Homebrewの基本操作](http://qiita.com/yoshinariiii/items/7b610c96d1d4af08ae6b)  
- [Mac OSX で開発環境を構築するための環境構築 ](http://qiita.com/hidekuro/items/4c926587f3a1cb2ec1e0)  
- [そこはかとなく書くよん。](http://tdoc.info/blog/categories/ansible.html)  
- [これからGit を始めてみようという人のための使い方と入門フロー](http://commte.net/blog/archives/5251)  
- [同じマシンで複数のgithubアカウントを使い分ける](http://qiita.com/strsk/items/96987bfc98e3f92fe6fb)
- [Mac】OS X スクリーンショットのファイル名やファイル形式などの変更方法](http://ichitaso.com/mac/tips-for-os-x-screenshot/)  
- [Homebrew CaskでiTerm2を導入＆初期設定まとめ](http://vdeep.net/iterm2)  
- [iTerm2 と zsh のインストールと基本的な設定](http://qiita.com/bird_nitryn/items/bd063f0d7eb18287fec1)  
- [pip - Pythonパッケージ管理](https://pip.pypa.io/)  ※インストールのみ  
`$ sudo easy_install pip`  
`$ pip install --upgrade pip`  
[※ Python パッケージ管理技術まとめ](http://www.yunabe.jp/docs/python_package_management.html)  
- [rbenv, ruby-build - rubyバージョン管理](https://github.com/rbenv)  
`$ brew update`  
`$ brew install rbenv ruby-build`  
`$ echo 'eval "$(rbenv init -)"' >> ~/.bash_profile`  
`$ source ~/.bash_profile`  
`$ rbenv --version`  
[※ rbenv日本語リファレンス | Ruby STUDIO](http://ruby.studio-kingdom.com/rbenv)  
[※ Homebrewでrbenvをインストールする](http://mawatari.jp/archives/install-rbenv-by-homebrew)  
[※ ryenvの仕組み](http://komaken.me/blog/2013/07/05/)  
- [ruby - スクリプト言語](http://dev.classmethod.jp/server-side/language/build-ruby-environment-by-rbenv/)  
`$ rbenv install --list`  
`$ rbenv install 2.1.6`  
    ※ 管理方法  
    `$ rbenv versions` インストール済み Ruby のバージョンがリスト表示  
    `$ rbenv version` カレントディレクトリで有効な Ruby バージョンが表示  
    `$ rbenv global` 環境全体での Ruby バージョンを指定  
    `$ rbenv local` ディレクトリ固有の Ruby バージョンを指定  
    `$ brew update; brew upgrade rbenv ruby-build` Rubyインストールリスト更新  
    [※ rbenv を利用した Ruby 環境の構築](http://dev.classmethod.jp/server-side/language/build-ruby-environment-by-rbenv/)  
- [pyenv, virtualenv - pythonバージョン管理](https://github.com/yyuu/pyenv)  
`$ brew update`  
`$ brew install pyenv-virtualenv` ※pyenv含まれてます  
`$ echo 'eval "$(pyenv init -)"' >> ~/.bash_profile`  
`$ echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.bash_profile`  
`$ source ~/.bash_profile`
`$ pyenv --version`  
[※ Homebrewで「pyenv」をインストールする方法と基本的な使い方](http://suiwabi.com/pyenv/)  
- [python - スクリプト言語](http://dev.classmethod.jp/server-side/language/build-ruby-environment-by-rbenv/)  
`$ pyenv install --list`  ※今回はインストールしない  
    ※ 管理方法  
    `$ pyenv versions` インストール済み python のバージョンがリスト表示  
    `$ pyenv version` カレントディレクトリで有効な python バージョンが表示  
    `$ pyenv global` 環境全体での python バージョンを指定  
    `$ pyenv local` ディレクトリ固有の python バージョンを指定  
    `$ brew update; brew upgrade pyenv-virtualenv python` インストールリスト更新  
    [※ pyenvとvirtualenvで環境構築](http://qiita.com/Kodaira_/items/feadfef9add468e3a85b)  
- [git - バージョニング管理](https://git-scm.com)  
`$ brew install git`  
`$ git config --global user.name <メインアカウントのユーザ名>`  
`$ git config --global user.email <メインアカウントのメールアドレス>`  
[※ これからGit を始めてみようという人のための使い方と入門フロー](http://commte.net/blog/archives/5251)  
[※ 同じマシンで複数のgithubアカウントを使い分ける](http://qiita.com/strsk/items/96987bfc98e3f92fe6fb)  
- [aws-cli - AWS-CLI](http://aws.amazon.com/jp/cli)  
`$ brew install awscli`  
`$ aws configure`  
[※ 初心者向け MacユーザがAWS CLIを最速で試す方法](http://dev.classmethod.jp/cloud/aws/mac-aws-cli/)  
[※ MacOSX HomebrewでAWS CLIがインストールできるようになったようです。](http://dqn.sakusakutto.jp/2014/06/macosx_homebrew_aws_cli.html)  
