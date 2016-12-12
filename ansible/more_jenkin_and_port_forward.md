# 擴充 jenkins role 及設定 Vagrant port forward

在前面的幾個章節內，我們只有簡單的利用 cURL 測試 Jenkins 是否已經正確安裝。現在我們要透過更新 `Vagrantfile` 來讓主機端的瀏覽器可以訪問 Jenkins 位於 `localhost` 的服務，並新增幾個我們在未來章節會用到 roles dependencies。

#### 安裝 Git 及 pip

在本系列文章一開始的簡介中，我們有提到 Jenkins 是一個可以幫我們從原始碼託管服務（在這次的教學系列文中將以 [GitHub](https://github.com/) 作為範例）上產品狀況的一個持續整合服務。因此為了讓 Jenkins 運行的主機可以使用對應的版本控制系統 - [Git](https://git-scm.com/)，我們會寫另一個 role 來安裝 Git：

依照以下結構新增 `git` 這個 role：

```shell
workspace/
├── Vagrantfile
├── ansible.cfg
├── inventory
├── playbook.yml
└── roles
    ├── curl
    │   └── tasks
    │       └── main.yml
    ├── git
    │   └── tasks
    │       └── main.yml
    └── jenkins
        ├── meta
        │   └── main.yml
        └── tasks
            └── main.yml
```

並在 `roles/git/tasks/main.yml` 中寫入以下內容：

```yml
---
  - name: install git
    yum:
      name: git-all
    when: ansible_distribution == 'CentOS'

  - name: install git
    apt:
      name: git
    when: ansible_distribution == 'Ubuntu' or ansible_distribution == 'Debian'
```

由於 `git` 這個 role 理論上在未來被重複利用到的頻率會非常高，因此我在這裡新增了在 CentOS 這套作業系統上使用 yum 安裝 git 的方法（在 yum 套件管理系統中，Git 套件的名稱與 apt 系統並不相同）以增加這個 role 的使用靈活性。同理，我們也可以將之前寫的 `curl` 略作更改：

```yml
---
  - name: install curl
    yum:
      name: curl
    when: ansible_distribution == 'CentOS'

  - name: install curl
    apt:
      name: curl
    when: ansible_distribution == 'Ubuntu' or ansible_distribution == 'Debian'
```

除此之外，由於在未來的章節中，我會讓 Jenkins 使用 [Ansible-lint](https://github.com/willthames/ansible-lint) 這個語法檢查器來反向檢查我們所有的 Ansible playbook 及 roles，以確保我們每次放在 GitHub 上的程式碼都是最佳狀態。為了安裝 Ansible-lint，我們還需要安裝 pip 這個 Python 套件管理工具在主機上：

```shell
workspace/
├── Vagrantfile
├── ansible.cfg
├── inventory
├── playbook.yml
└── roles
    ├── curl
    │   └── tasks
    │       └── main.yml
    ├── git
    │   └── tasks
    │       └── main.yml
    ├── jenkins
    │   ├── meta
    │   │   └── main.yml
    │   └── tasks
    │       └── main.yml
    └── pip
        └── tasks
            └── main.yml
```

並在 `roles/pip/tasks/main.yml` 中寫入以下內容：

```yml
---
  - name: Add EPEL repository
    yum_repository:
      name: EPEL
      description: EPEL yum repo
      baseurl: http://download.fedoraproject.org/pub/epel/$releasever/$basearch/
      gpgkey: /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-6
    when: ansible_distribution == "CentOS" and ansible_distribution_major_version == "6"

  - name: install EPEL
    yum:
      name: epel-release
    when: ansible_distribution == "CentOS" and ansible_distribution_major_version == "7"

  - name: install pip
    yum:
      name: python-pip
      state: installed
      update_cache: yes
    when: ansible_distribution == 'CentOS'

  - name: install pip
    apt:
      name: python-pip
      update_cache: yes
    when: ansible_distribution == 'Debian'
```

雖然 pip 也是在一般操作系統中非常常用的套件管理工具，但在 CentOS 作業系統中安裝 pip 會稍微比其他 Linux / Unix 的安裝步驟更加麻煩一點，我們會需要先安裝 [EPEL (Extra Packages for Enterprise Linux)](https://fedoraproject.org/wiki/EPEL) 才可以進行 pip 的安裝，而其中又以 CentOS 6.x 版本的安裝更加繁瑣。由於我們這次的重點並非討論其中的差異，因此我只將我個人的 role 在這裡分享給大家，若有興趣了解的讀者可以在網路上自行研究其差異 (e.g. [CentOS 6.x](http://sharadchhetri.com/2014/05/30/install-pip-centos-rhel-ubuntu-debian/), [CentOS 7.x](http://sharadchhetri.com/2014/09/07/install-epel-repo-centos-7-rhel-7/))。

最後，更新 `jenkins` 的 `meta` 依賴：

```yml
---
  dependencies:
    - { role: curl, become: yes }
    - { role: git, become: yes }
    - { role: pip, become: yes }
```

結果如下：

```
PLAY [ironman] *****************************************************************

TASK [setup] *******************************************************************
ok: [ironman]

TASK [curl : install curl] *****************************************************
skipping: [ironman]

TASK [curl : install curl] *****************************************************
ok: [ironman]

TASK [git : install git] *******************************************************
skipping: [ironman]

TASK [git : install git] *******************************************************
changed: [ironman]

TASK [pip : Add EPEL repository] ***********************************************
skipping: [ironman]

TASK [pip : install EPEL] ******************************************************
skipping: [ironman]

TASK [pip : install pip] *******************************************************
skipping: [ironman]

TASK [pip : install pip] *******************************************************
changed: [ironman]

TASK [pip : install curl] ******************************************************
ok: [ironman]

TASK [jenkins : add jenkins key] ***********************************************
ok: [ironman]

TASK [jenkins : add jenkins repository] ****************************************
ok: [ironman]

TASK [jenkins : install jenkins] ***********************************************
changed: [ironman]

PLAY RECAP *********************************************************************
ironman                    : ok=7    changed=3    unreachable=0    failed=0
```

我們可以看到 Ansible 會根據對應的作業系統判斷哪些 tasks 可以被略過 (`skipping`)，這也是為什麼我們不需要特別為不同作業系統保存多份版本的 role，只要專注在同一個 role 中開發當前任務即可。另外，若仔細閱讀在終端機上顯示的 Ansible 安裝流程，應該會發現已經被安裝過的服務並不會一直重複被安裝。顯示為 `changed` 表示遙控主機在這次運行 playbook 的過程裡，Ansible 在該項 task 中對系統做了改變；`ok` 則表示該 task 的完成條件已滿足，所以 Ansible 並不會額外對主機做其他變動。特別需要解釋的是，若我們使用了 `update_cache: yes` 這項屬性來透過套件管理工具安裝服務。我們每次安裝服務前都會試圖將套件管理工具的版本都更新到當前最新版本，然後才進行安裝步驟，因此每次 task 的狀態都將會顯示為 `changed`。

#### Vagrant port forward

由於 Jenkins 預設是運行在主機的 `port 8080`，因此我們必須試著把遙控節點的 `port 8080` 廣播到控制主機上。不過，由於 `port 8080` 是一個非常容易被其他服務使用的 port，因此為了避免衝突，我們將利用 `forwarded_port` 這個參數來將遙控節點上的 `port 8080` 映射到控制主機上的 `port 9080`。

在 `Vagrantfile` 中，直接在 `config.vm.define "ironman"` 該行下面新增以下設定：

```ruby
config.vm.network "forwarded_port", guest: 8080, host: 9080
```

這樣表示我們可以直接在控制主機上使用 `port 9080` 來訪問遙控節點上 `port 8080` 的服務。現在重新啟動 Vagrant 來讓新設定生效：

```shell
$ vagrant reload

==> ironman: Attempting graceful shutdown of VM...
==> ironman: Checking if box 'hashicorp/precise64' is up to date...
==> ironman: Clearing any previously set forwarded ports...
==> ironman: Clearing any previously set network interfaces...
==> ironman: Preparing network interfaces based on configuration...
    ironman: Adapter 1: nat
==> ironman: Forwarding ports...
    ironman: 8080 (guest) => 9080 (host) (adapter 1)
    ironman: 22 (guest) => 2222 (host) (adapter 1)
==> ironman: Booting VM...
==> ironman: Waiting for machine to boot. This may take a few minutes...
    ironman: SSH address: 127.0.0.1:2222
    ironman: SSH username: vagrant
    ironman: SSH auth method: private key
    ironman: Warning: Remote connection disconnect. Retrying...
==> ironman: Machine booted and ready!
==> ironman: Checking for guest additions in VM...
    ironman: The guest additions on this VM do not match the installed version of
    ironman: VirtualBox! In most cases this is fine, but in rare cases it can
    ironman: prevent things such as shared folders from working properly. If you see
    ironman: shared folder errors, please make sure the guest additions within the
    ironman: virtual machine match the version of VirtualBox you have installed on
    ironman: your host and reload your VM.
    ironman:
    ironman: Guest Additions Version: 4.2.0
    ironman: VirtualBox Version: 5.0
==> ironman: Mounting shared folders...
    ironman: /vagrant => /Users/tsoliang/Desktop/workspace
==> ironman: Machine already provisioned. Run `vagrant provision` or use the `--provision`
==> ironman: flag to force provisioning. Provisioners marked to run always will still run.
```

我們可以從上面資訊中看到 Vagrant 已經幫我們把遙控節點上的 `port 8080` 廣播到控制主機的 `port 9080` 了。現在嘗試透過瀏覽器訪問 `http://localhost:9080` 就應該可以順利進入 Jenkins 的初始化頁面了：

![jenkins_index_9080](https://github.com/tsoliangwu0130/learn-ansible-and-jenkins-in-30-days/blob/master/images/jenkins_index_9080.png?raw=true)

