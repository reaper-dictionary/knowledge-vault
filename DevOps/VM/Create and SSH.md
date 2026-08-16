
## 1. Термины

- **Host** — физическая машина, у нас Arch.
    
- **Guest** — ОС внутри VM, у нас Ubuntu.
    
- **KVM** — аппаратная виртуализация CPU.
    
- **QEMU** — создаёт виртуальное оборудование VM.
    
- **libvirt** — управляет QEMU/KVM, VM, дисками и сетями.
    
- **virt-manager** — GUI для libvirt.
    
- **vCPU** — виртуальные CPU гостя.
    
- **qcow2** — виртуальный диск, растущий по мере записи.
    
- **NAT** — VM выходит наружу через host.
    
- **SSH** — удалённый shell.
    
- **public key** — кладём на сервер.
    
- **private key** — никогда не отдаём с клиентской машины.
    
- **snapshot** — точка возврата, не backup.
    

Цепочка:

`virt-manager → libvirt → QEMU → KVM → CPU`

## 2. Создание VM → SSH

### Arch

Проверить KVM:

```bash
lscpu | grep Virtualization
ls -l /dev/kvm
```

Установить стек:

```bash
sudo pacman -S qemu-desktop libvirt virt-manager dnsmasq edk2-ovmf
sudo systemctl start libvirtd.socket
```

Запустить NAT:

```bash
sudo virsh net-start default
```

`qemu:///system` нужен для полноценной libvirt-сети.

### virt-manager

Создать VM:

```text
Ubuntu Server ISO
4 vCPU
6144 MiB RAM
60 GiB qcow2
VirtIO disk
default NAT
VirtIO NIC
```

Установить Ubuntu.

### Ubuntu

```bash
sudo apt update
sudo apt install qemu-guest-agent
systemctl status ssh
```

### SSH-ключ на Arch

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_devops_lab
ssh-copy-id -i ~/.ssh/id_ed25519_devops_lab.pub reaper@<VM_IP>
ssh -i ~/.ssh/id_ed25519_devops_lab reaper@<VM_IP>
```

Public key попадёт в `~/.ssh/authorized_keys`; private остаётся на Arch.

Выход:

```bash
exit
```

### Snapshot

```bash
sudo poweroff
virsh -c qemu:///system snapshot-create-as <VM_NAME> base-clean
virsh -c qemu:///system snapshot-list <VM_NAME>
```

## 3. Что запомнить

**VM:** Arch предоставляет ресурсы → QEMU создаёт VM → KVM ускоряет CPU → libvirt управляет → virt-manager даёт GUI.

**Сеть:** Ubuntu → VirtIO NIC → libvirt NAT → Arch → Internet.

**SSH:** Arch хранит private key → Ubuntu хранит public key → SSH проверяет пару → получаем shell Ubuntu.

Наш конфликт Docker `FORWARD DROP` — это **отдельная неисправность хоста, не стандартный шаг создания VM**.