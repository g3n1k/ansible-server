keadaan ini adalah saat server diinstall default

## ubah ssh config

```bash
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
```

masukan config

```conf
PasswordAuthentication yes
Banner "jangan shared akses sembarangan"
Port 3922
PermitRootLogin no
AllowUsers hris
PermitEmptyPasswords no

```

restart ssh

```bash
sudo systemctl restart ssh

# or restart if not effect
sudo reboot
```

## setup via ansible

```
# update upgrade
ansible-playbook -i prod.hosts apt-update.yml

# install fail2ban
ansible-playbook -i prod.hosts install-fail2ban.yml

# setup ufw dan configure
ansible-playbook -i prod.hosts configure-ufw-whitelist-ports.yml

# jika configure tidak selesai, ctrl+c
# dan start manual ufw
sudo ufw enable

# install docker
ansible-playbook -i prod.hosts install-docker.yml

```
