# CHPWD(Change Password)
 Ansible을 활용한 서버 패스워드 변경 자동화

## Summary
 이전 IDC에서 운영 중인 수십여대에 고객사 서버에 패스워드를 3개월에 한번씩 변경하였다. 처음에는 한대씩 모든 서버에 접속하여 패스워드를 변경하였는데 시간이 많이 소요되었다.
 자동화 도구를 활용하여 변경하는 방안을 모색했고 그 중 Ansible로 테스트 후 고객사와 협의하였고 기존과 동일한 방식으로 패스워드를 변경하면서 작업 시간을 단축 시킬 수 있었다.

## Scenario
 다른 곳에서는 이렇게 심플한 구조로 패스워드를 관리하면 안될 것이다. 이전 고객사에 경우 고객사에 요청으로 규칙성과  편의성을 위해 모든 서버 패스워드에 동일한 키워드가 포함되어 있고 패턴이 똑같기 때문에 침입자가 키워드와 패턴만 알아챈다면... 상상하고 싶지 않다.
  어찌되었든 해당 고객사에서는 아래와 같은 패턴으로 패스워드를 변경하였다.
  * 패턴: {Keyword} + {IP 끝자리}
  * 예로
    * 이 달의 Keyword: 1BBaGGeu#@1 
    * WEB1 서버 IP: 1.2.3.4
    * WEB1 서버 패스워드: 1BBaGGeu#@14 (1BBaGGeu#@1+4)

 아래 내용은 테스트 환경을 기준으로 하여 사용법에 대해 작성하였다.
 
## Environment
 Linux 환경에 대해서만 작성 하였다. (Windows에 경우 WinRM을 활성화하여 동일하게 작업 가능)
 테스트 환경은 아래와 같다.

| Name | IP | OS | User |
|:---:|:---:|:--------:|:--------:|
| `Master` | 10.0.11.134 | Amazon Linux 2 | - |
| `Linux1` | 10.0.11.159 | Amazon Linux 2 | test1, test2 |
| `Linux2` | 10.0.11.72 | Amazon Linux 2 | test1, test2 |
| `Linux3` | 10.0.11.223 | Amazon Linux 2 | test1, test2 |

 Master 서버에서 정의된 Playbook을 통해 각 Client 서버에 test1, test2 사용자 계정에 패스워드를 변경한다.

                            ---------
                           |         |
                           |  Master |
                           |         |
                            ---------
                                |
                                | SSH(22 port)
              ------------------------------------
             |                  |                 |
             |                  |                 |
         ---------          ---------         ---------
        |         |        |         |       |         |
        | Client1 |        | Cleint2 |       | Client3 |
        |         |        |         |       |         |
         ---------          ---------         ---------



## Install Ansible to Master
 Master 서버에 Ansible을 설치한다. 테스트 환경이 Amazon Linux 이기 때문에 테스트 환경 기준으로 작성하였다.
 상세한 설치 매뉴얼은 공식 문서를 참고하면 된다.
 https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html

```
[ec2-user@master ~]$ sudo amazon-linux-extras install epel
[ec2-user@master ~]$ sudo yum install ansible
```

## Source Download
 해당 Git Repo에 소스를 다운받는다.

```
[ec2-user@master ~]$ git clone https://github.com/superxpzm/ansible
```

 chpwd 경로로 이동한다.

```
[ec2-user@master ~]$ cd ansible/chpwd
[ec2-user@master chpwd]$ tree
.
├── README.md
├── ansible.cfg
├── group_vars
│   ├── linux
│   └── windows
├── hosts
├── linux.yml
├── ping.yml
├── roles
│   ├── linux
│   │   └── tasks
│   │       └── main.yml
│   └── windows
│       └── tasks
│           └── main.yml
└── windows.yml

6 directories, 10 files
```

 사용하기 전에 수정해야 할 파일은 3개이다. 
* hosts
* group_vars/linux
* roles/linux/tasks/main.yml

#### hosts
 패스워드를 변경할 클라이언트 호스트를 정의한다. 호스트 IP와 끝자리 IP를  수정한다.
* 수정 항목: hostname, ansible_host, end_ip

```
[ec2-user@master ~]$ vi hosts
[all:children]
linux
windows

[linux]
linux1            ansible_host=10.0.11.159 end_ip=159
linux2            ansible_host=10.0.11.72  end_ip=72
linux3            ansible_host=10.0.11.223 end_ip=223

[windows]
windows1           ansible_host=172.31.0.31 end_ip=31
windows2           ansible_host=172.31.0.44 end_ip=44
windows3           ansible_host=172.31.0.47 end_ip=47
```

#### group_vars/linux
 해당 파일에 설정된 변수 중 ansible_ssh_user에 정의된 사용자로 인증하게 된다. ssh 접속이 가능한 사용자 정보로 수정한다.
* 수정 항목: ansible_ssh_user

```
[ec2-user@master chpwd]$ cat group_vars/linux
---
ansible_ssh_user: test1
ansible_ssh_pass: "{{ password }}{{ end_ip }}"
ansible_ssh_new_pass: "{{ new_password }}{{ end_ip }}"
ansible_ssh_port: 22
```

#### roles/linux/tasks/main.yml
 패스워드를 변경하는 액션을 해당 파일에 정의되어 있다. 패스워드를 변경할 사용자 계정을 수정한다.
* 수정 항목: name

```
---
- name: Change password of existing test1
  user:
    name: test1
    update_password: always
    password: "{{ ansible_ssh_new_pass | password_hash('sha512')}}"

- name: Change password of existing test2
  user:
    name: test2
    update_password: always
    password: "{{ ansible_ssh_new_pass | password_hash('sha512')}}"
```

## Check SSH
 Ansible은 Linux 클라이언트와 통신할때 일반적으로 ssh 를 이용한다. 위에서 인벤토리(hosts)에 정의한 호스트들과 정상적으로 ssh 접속이 가능해야 작업이 가능하다. 접속 확인용으로 만들어 놓은 플레이북으로 호스트들에 접속 상태를 체크한다.
 패스워드를 입력할때 IP 끝자리를 제외한 패스워드에 키워드(1BBaGGeu#@1)만 입력한다. 접속이 실패한다면 아래와 같이 접근이 불가한 호스트를 확인 할 수 있다.
 
```
[ec2-user@master chpwd]$ ansible-playbook ping.yml
Enter Password:
confirm Enter Password:

PLAY [Linux Client SSH Check] ******************************************************************************************************************

TASK [SSH Check] *******************************************************************************************************************************
ok: [linux2]
ok: [linux1]
fatal: [linux3]: UNREACHABLE! => {"changed": false, "msg": "Invalid/incorrect password: Permission denied, please try again.", "unreachable": true}

PLAY RECAP *************************************************************************************************************************************
linux1                     : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
linux2                     : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
linux3                     : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0
```

## Change Password
 호스트 상태를 모두 점검하였다면 정의된 플레이북으로 패스워드를 변경 할 수 있다. 번거로울 수 있지만 플레이북을 실행하였을때 3가지 정보를 입력해야 한다.
 * 현재 패스워드 키워드
 * 신규 패스워드 키워드
 * root 패스워드 키워드
 
 패스워드를 변경하는 과정은 실제 커맨드를 이용하여 변경하는 과정과 동일하다. 일반 사용자 계정으로 접속한 뒤 root로 전환 후 패스워드를 변경하는 것이다. 동일한 사용자에 패스워드를 변경하는 것이라 root로 전환하지 않더라도 sudo를 이용하거나 remote shell command를 이용해 변경 할 수도 있다. 상황에 따라 조금 수정하면 될 것 같다.

 플레이북을 실행하면 아래와 같이 진행 상황과 결과를 확인 할 수 있다.

```
[ec2-user@master chpwd]$ ansible-playbook linux.yml
Enter Password:
confirm Enter Password:
Enter Root Password:
confirm Enter Root Password:
Enter New Password:
confirm Enter New Password:

PLAY [linux] ***********************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************
ok: [linux1]
ok: [linux2]
ok: [linux3]

TASK [linux : Change password of existing test1] ***********************************************************************************************
changed: [linux3]
changed: [linux2]
changed: [linux1]

TASK [linux : Change password of existing test2] ***********************************************************************************************
changed: [linux1]
changed: [linux2]
changed: [linux3]

PLAY RECAP *************************************************************************************************************************************
linux1                     : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
linux2                     : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
linux3                     : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```




