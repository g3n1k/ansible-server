# ansible to manage multi server

3 server linux yg bisa saling ping, ada 1 laptop sebagai control node

pada ansible kita hanya butuh menginstall pada laptop control node

### install ansible pada laptop

```bash
sudo dnf update
sudo dnf install ansible
```

pada virtual server ub

```bash
sudo apt update
sudo apt install ansible
```

## persiapan pada control node

copy config folder ansible

```bash
cp -rf /etc/ansible/ ~/workspace/ansible-docker
cd ~/workspace/ansible-docker
ls
```

the output

```bash
[g3n1k@fedora ansible-docker]$ ls
ansible.cfg  hosts  roles
```

### create config ansible

```bash
cd ~/worskpace/ansible-docker
ansible-config init --disabled > ansible.cfg
```

edit value

```bash
inventory = hosts

# this just for development
host_key_checking = False

```

save file

### create inventory

contoh config ansible

```yaml
all:
  hosts:
    mail.example.com
    192.168.68.130
  children:
    webserver:
      hosts:
        web01:
        192.168.68.131
    docker:
      hosts:
        docker01:
          ansible_user: dockeruser
          ansible_password: passwordnya
        192.168.68.135:
        192.168.68.136:
      vars:
        ansible_user: usersama
        ansible_password: passsama

```

`hosts` dapat menggunakan nama domain atau ip address
`ansible_user` dan `ansible_password` digunakan jika hostsnya memiliki user pass berbeda  
jika user dan password sama kita letakan pada `vars`
pada percobaan ini, anggaplah kita hanya memiliki beberapa vm, dengan nama user dan pass yang sama, kita sederhanakan confignya,

```yaml
all:
  children:
    docker:
      hosts:
        192.168.68.114:
        192.168.68.115:
      vars:
        ansible_user: ub
        ansible_password: passub
```

bisa lebih di sederhanakan lagi menjadi group hosts saja

```yaml
docker:
  hosts:
    192.168.68.114:
    192.168.68.115:
  vars:
    ansible_user: ub
    ansible_password: passub
```

### install sshpass

```bash
sudo dnf install sshpass
```

### persiapan membuat playbook

tahapan menginstall docker

1. install dependensi `apt-transport-https`, `ca-certificates`, `curl`, `gnupg-agent`, `software-properties-common`
2. tambahkan GPG key
3. tambahkan repository
4. install `docker-ce`, `docker-ce-cli` dan `containerd.io`

playbook adalah kumpulan dari perintah-perintah
playbook yg di gunakan bernama `install-docker.yml`

```yaml
---
- name: install docker
  hosts: docker
  become: true
  tasks:
    - name: install dependencies
      apt:
        name: "{{item}}"
        state: present
        update_cache: yes
      loop:
        - apt-transport-https
        - ca-certificates
        - curl
        - gnupg-agent
        - software-properties-common

    - name: get gpg key
      apt-key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present

    - name: add docker repository
      apt_repository:
        repo: deb https://download.docker.com/linux/ubuntu focal stable
        state: present

    - name: install docker
      apt:
        name: "{{item}}"
        state: latest
        update_cache: yes
      loop:
        - docker-ce
        - docker-ce-cli
        - containerd.io

    - name: start service
      service:
        name: docker
        state: started
        enabled: true

  handlers:
    - name: restart docker
      service:
        name: docker
        state: restarted
```

`hosts` lihat file hosts  
`become` user sudo  
`state` bisa berisi `latest` terbaru, `present` ada
``

## persiapan vm docker

- `d01` adalah server docker pertama
- perlu untuk disable sudo password prompt setelah login, sehingga tidak perlu menjawab permintaan memasukan password

```bash
ssh user@d01
sudo nano /etc/sudoers
```

tambahkan kode berikut

```conf
docker ALL=(ALL)  NOPASSWD: ALL
```

dan logout

## run perintah pada control node

```bash
ansible-playbook ufw-ssh-whitelist-ip.yml  -i ./hosts
```

## menambahkan server baru

- hostname `d02`
- ip 192.168.68.121

apa yg harus dilakukan ?

- pada `d02` node

  - modifiy /etc/sudoers

- pada control node
  - tambahkan ip address to inventory (hosts file)
  - run playbok

---

## cara gunakan script

1. sudah menginstall ansible pada controll node

```bash
sudo dnf update
sudo dnf install ansible
```

2. check isi file hosts, tambahkan jika diperlukan

```Bash
[docker]
svr31 ansible_user=ub ansible_password=123456 ansible_ssh_port=2234
svr32 ansible_user=ub ansible_ssh_private_key_file=/home/user/.ssh/id_rsa.pub
```

3. execute with command

```bash
ansible-playbook -i ./hosts update-upgrade.yml
ansible-playbook -i ./hosts configure-ufw-whitelist-ports.yml
ansible-playbook -i ./hosts install-fail2ban.yml
ansible-playbook -i ./hosts install-docker.yml
```

4. [optional] beberapa perintah

melihat isi host

```bash
ansible all -i ./hosts --list-hosts
```

execute tetapi membatasi pada group server tertentu

```bash
ansible-playbook -i ./hosts install-docker.yml --limit webservers
```

mengecek koneksi

```bash
ansible all -i ./hosts -m ping
```

---

## git pull with ansible

1. **buat file vault** untuk menyimpan credensial

```bash
   ansible-vault create vault.yml
```

2. **tambahkan kredensial** kedalam file vault

```bash
   git_username: your_username
   git_password: your_password
```

3. **gunakan variable vault** dalam playbook

```yaml
   ---
   - name: Pull Git Repository with Vault Credentials
     hosts: your_group
     vars_files:
       - vault.yml
     tasks:
       - name: Ensure the repository is present
         git:
           repo: "https://{{ git_username }}:{{ git_password }}@github.com/username/repo.git"
           dest: /path/to/your/destination
           version: main
           update: yes
```

4. **jalankan playbook dengan vault**

```bash
   ansible-playbook -i ./hosts git-pull.yml --ask-vault-pass
```

5. **jalankan playbook hanya pada group tertentu**
   untuk menjalankan ansible playbook pada group tertentu gunakan `--limit`

```bash
    ansible-playbook -i ./hosts git-pull.yml --limit docker
```

## cara menggunakan script

### menambahkan client

buka file hosts  
masukan pada group

## DEV

1. update upgrade

```bash
ansible-playbook -i ./hosts apt-update.yml --limit docker
```

2. install docker

```bash
ansible-playbook -i ./hosts install-docker.yml --limit docker
```

3. set firewall

```bash
ansible-playbook -i ./hosts install-fail2ban.yml --limit docker
```

4. set fail2ban

```bash
ansible-playbook -i ./hosts configure-ufw-whitelist-ports.yml --limit docker
```

# MANUAL

1. git clone first

## DAILY

1. update upgrade

```bash
ansible-playbook -i ./hosts apt-update.yml --limit docker
```

2. pull github

```bash
ansible-playbook -i ./hosts git-pull.yml --limit git  --ask-vault-pass
```
