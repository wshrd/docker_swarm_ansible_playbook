#docker #rhel #ansible


Подгатавливаем шаблон на гипервизоре proxmox на базе Rocky Linux.
Устанавливаем дистрибутив обычным способом на vm в proxmox.
```bash
yum install cloud-init -y
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
Пишем docker_swarm.yml (описание переменных и vm) следующего содержания
```yml
api_host: pve.dx.local
api_user: root@pam
api_password: Password
node: pve
clone_vm: rocky
key_name: "ssh-rsa KEY= adm1n@LocalPC"
vms:
  node0:
    name: swnode0.dx.local
    ipaddress0: 192.168.1.150/24
    vmid: 150
    cores: 4
    sockets: 1
    memory: 4096
  node1:
    name: swnode1.dx.local
    ipaddress0: 192.168.1.151/24
    vmid: 151
    cores: 4
    sockets: 1
    memory: 4096
```
Создаем Playbook new_dsw_nodes.yml
```yaml
---
- name: Initial setup VM
  hosts: localhost
  vars_files:
    - docker_swarm.yml
  tasks:
    - name: Clone VMs
      proxmox_kvm:
        node: "{{ node }}"
        name: "{{ item.value.name }}"
        newid: "{{ item.value.vmid }}"
        api_user: "{{ api_user }}"
        api_password: "{{ api_password }}"
        api_host: "{{ api_host }}"
        clone: "{{ clone_vm }}"
        full: true
        timeout: 500
      loop: "{{ lookup('dict', vms) }}"
      tags: [clone]
    - name: Update VMs set IP and sshkeys
      proxmox_kvm:
        api_host: "{{ api_host }}"
        api_user: "{{ api_user }}"
        api_password: "{{ api_password }}"
        cores: "{{ item.value.cores }}"
        sockets: "{{ item.value.sockets }}"
        memory: "{{ item.value.memory }}"
        update: true
        vmid: "{{ item.value.vmid }}"
        node: "{{ node }}"
        name: "{{ item.value.name }}"
        nameservers: "192.168.1.4"
        net:
          net0: "virtio,bridge=vmbr0"
        ipconfig:
          ipconfig0: "ip={{ item.value.ipaddress0 }},gw=192.168.1.1"
          net1: "virtio,bridge=vmbr1"
        sshkeys: "{{ key_name }}"
      loop: "{{ lookup('dict', vms) }}"
      tags: [update]
    - name: Start VMs
      proxmox_kvm:
        api_host: "{{ api_host }}"
        api_password: "{{ api_password }}"
        api_user: "{{ api_user }}"
        vmid: "{{ item.value.vmid }}"
        node: "{{ node }}"
        state: started
      loop: "{{ lookup('dict', vms) }}"
      tags: [start]
```
Запускаем playbook
```bash
ansible-playbook new_dsw_node.yml
```
Ждем завершения выполнения.
DONE.
