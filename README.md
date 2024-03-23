#docker #rhel #ansible

## Ansible playbook для создания кластера docker swarm.

### Подготовка cloud init шаблона
***
Предварительно необходимо создать cloudinit шаблон.
Подгатавливаем шаблон на гипервизоре proxmox на базе Debian.
Устанавливаем дистрибутив обычным способом на vm в proxmox.
```bash
apt install cloud-init -y
```
Устанавливаем все необходимые пакеты.
Правим конфиг.
```
nano /etc/cloud/cloud.cfg
```
Меняем ssh_pwauth на true.
```
ssh_pwauth: true
```
Выключаем vm. Добавляем CloudInit Drive в настройках vm на proxmox.
На вкладке Cloud-Init задаем учетку с паролем.
Жмем Regenerate image и конвертируем виртуальную машину в template.
На своей машине устанавливаем ansible если не стоит.
Ставим необходимые модули.
```bash
sudo apt install python3-proxmoxer
sudo apt install python3-requests
```
### Использование Playbook`а
***
Клонируем репозиторий 
```bash
git clone https://github.com/wshrd/docker_swarm_ansible_playbook
cd docker_swarm_ansible_playbook/
```
***
Указываем свои будущие узлы в инвентаре (можно по ip или по dns именам если они уже прописаны у вас в dns)
```
[MasterSwarmNode]
192.168.1.250 ansiblehost_ssh_host=192.168.1.250
[WorkerSwarmNodes]
192.168.1.251 ansiblehost_ssh_host=192.168.1.251
192.168.1.252 ansiblehost_ssh_host=192.168.1.252
```
***
Указываем необходимые переменные в docker_swarm.yml
Такие как ip адрес pve ноды, логин пароль, ssh ключ, имя cloud init шаблона, шлюз, днс, репозиторий docker и его gpg ключ 
```
api_host: ip_pve_node
api_user: root@pam
api_password: password
node: name_pve_node
clone_vm: cloud_init_template_name
key_name: "ssh_key"
gateway: gateway_ip
dns: dns_ip
docker_gpg: https://download.docker.com/linux/debian/gpg
docker_repo: 'deb https://download.docker.com/linux/debian bookworm stable'
```
В секции vms описываются VM, их может быть разное количество, первая нода будет мастером остальные воркерами.
```
vms:
  node0:
    name: dswnode0.dx.local
    ipaddress: 192.168.1.250/24
    vmid: 250
    cores: 2
    sockets: 1
    memory: 2048
  node1:
    name: dswnode1.dx.local
    ipaddress: 192.168.1.251/24
    vmid: 251
    cores: 2
    sockets: 1
    memory: 2048
  node2:
    name: dswnode2.dx.local
    ipaddress: 192.168.1.252/24
    vmid: 252
    cores: 2
    sockets: 1
    memory: 2048
```

Запускаем playbook
```bash
ansible-playbook -i inventory create_swarm.yml
```
Ждем завершения выполнения.
DONE. Кластер готов к работе.
