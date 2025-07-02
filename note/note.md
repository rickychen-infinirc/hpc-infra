# 環境
VM:
| 項目          | HPC1         | HPC2         |
|--------------|-------------|-------------|
| 帳號         | hpc         | hpc         |
| 主機名       | hpc1        | hpc2        |
| CPU         | 32 core     | 32 core     |
| RAM         | 80G         | 80G         |
| GPU         | RTX 4500Ada | RTX 4500Ada |
| 10G ETH IP  | 10.0.10.1   | 10.0.10.2   |
| 56G IB IP   | 10.0.10.3   | 10.0.10.4   |


# 各節點同步時間

```bash
sudo timedatectl set-timezone Asia/Taipei
```

# 設置DNS與主機名稱

```bash
sudo nano /etc/hosts 
```

```bash
127.0.0.1       localhost
127.0.1.1       hpc1.local.infinirc.com      
10.0.10.1       hpc1.local.infinirc.com
10.0.10.2       hpc2.local.infinirc.com
```

```bash
sudo hostnamectl set-hostname hpc1.local.infinirc.com     
```


# NVIDIA GPU顯卡驅動與CUDA安裝
## 禁用 Nouveau
檢查是否已經啟用 有啟用會有輸出
```bash 
lsmod | grep nouveau
```

```bash
sudo nano /etc/default/grub
```
增加nouveau.modeset=0 如下
```bash
GRUB_CMDLINE_LINUX="quiet splash nouveau.modeset=0"
```
更新GRUB
```bash
sudo update-grub
```
```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

更新initramfs
```bash
sudo update-initramfs -u
```
```bash
sudo dracut -f --kver $(uname -r)
```

重啟主機後再次檢查 要是沒有任何內容輸出
```bash 
lsmod | grep nouveau
```

## 安裝gcc-12
```bash
sudo apt-get install gcc-12 g++-12
# 為 gcc 設定替代選項
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 120 \
                         --slave /usr/bin/g++ g++ /usr/bin/g++-12 \
                         --slave /usr/bin/gcov gcov /usr/bin/gcov-12 \
                         --slave /usr/bin/gcc-ar gcc-ar /usr/bin/gcc-ar-12 \
                         --slave /usr/bin/gcc-ranlib gcc-ranlib /usr/bin/gcc-ranlib-12

# 如果您系統上有多個 GCC 版本，可以選擇預設版本
sudo update-alternatives --config gcc
```
確認gcc版本
```bash
gcc --version
```

## 安裝build-essential
```bash
sudo apt install build-essential
```

## 安裝驅動
到這邊下載對應的驅動程式
https://www.nvidia.com/zh-tw/drivers/

```bash
chmod +x NVIDIA-Linux-x86_64-555.58.02.run
```

```bash
sudo ./NVIDIA-Linux-x86_64-555.58.02.run
```
測試
```bash
nvidia-smi
```

## CUDA
到以下下載對應系統的CUDA
https://developer.nvidia.com/cuda-11-8-0-download-archive

範例
```bash
wget https://developer.download.nvidia.com/compute/cuda/11.8.0/local_installers/cuda_11.8.0_520.61.05_linux.run
sudo sh cuda_11.8.0_520.61.05_linux.run
```

安裝完後新增到環境變數
```bash
nano ~/.bashrc
```

新增
```bash
export PATH=$PATH:/usr/local/cuda-11.8/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda-11.8/lib64
```

```bash
source ~/.bashrc
```

測試
```bash
nvcc -V
```

# InfiniBand MLNX_OFED驅動安裝
到以下下載驅動程式
https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/

```bash
sudo apt-get install python3
sudo ln -s /usr/bin/python3 /usr/bin/python
```

```bash
wget <https://www.mellanox.com/downloads/MFT/mft-4.22.1-417-x86_64-deb.tgz>
```

```bash
tar xzvf mft-4.22.1-417-x86_64-deb.tgz
```

```bash
sudo apt-get install dkms
sudo apt-get install bison
sudo apt-get install libxtables-dev
sudo apt-get install flex
```

```bash
sudo bash mft-4.22.1-417-x86_64-deb/install.sh
```

```bash
mst start
```

找到網卡

```bash
hpc@hpc1:~$ lspci | grep Mellanox
06:11.0 Network controller: Mellanox Technologies MT27520 Family [ConnectX-3 Pro]
```

安裝工具

```bash
sudo apt-get install infiniband-diags ibutils ibverbs-utils perftest mstflint
```

```bash
sudo apt install infiniband-diags rdma-core ibutils ibverbs-utils perftest
```

ps

`sudo ./mlnxofedinstall --with-nvmf`

查看 Infiniband 設備

```bash
ibv_devices
```

檢查 Infiniband 端口狀態

```bash
ibstat
```

```bash
sudo apt install net-tools
```

```bash
ifconfig
```

重啟

加載ib_ipoib

```bash
sudo modprobe ib_ipoib
```

檢查

```bash
lsmod | grep ib_ipoib
```

創建

```bash
ip a | grep -i infiniband
```

```bash
ip link show
```

```bash
sudo ip link set ibp6s17 up
```

```bash
sudo ip addr add 10.0.10.3/17 dev ibp6s17
sudo ip link set ibp6s17 up
```

```bash
sudo nano /etc/systemd/network/25-ibp6s17.network
```

```bash
[Match]
Name=ibp6s17

[Network]
Address=10.0.10.3/17
Gateway=10.0.0.1
DNS=8.8.8.8

[DHCP]
UseDNS=false
```

```bash
sudo systemctl restart systemd-networkd
```

配置ip 如果需要sudo apt install ifupdown

```bash
systemctl status NetworkManager
```

```bash
nmcli device status
```

創建網路連接

```bash
sudo nmcli con add type infiniband con-name ibp6s17 ifname ibp6s17 ip4 10.0.10.3/17 gw4 10.0.0.1
```

```bash
sudo nmcli con mod ibp6s17 ipv4.addresses 10.0.10.3/17
sudo nmcli con mod ibp6s17 ipv4.gateway 10.0.0.1
sudo nmcli con mod ibp6s17 ipv4.dns "8.8.8.8,8.8.4.4"
sudo nmcli con mod ibp6s17 ipv4.method manual
```

```bash
sudo nmcli con up ibp6s17
```

自動連線

```bash
sudo nmcli con mod ibp6s17 connection.autoconnect yes
```

顯示配置

```bash
nmcli con show ibp6s17
```


# SSH免密碼登入

在node1

```bash
ssh-keygen
```

```bash
//hpc1
ssh-copy-id -i ~/.ssh/id_ed25519.pub hpc2
ssh-copy-id -i ~/.ssh/id_ed25519.pub hpc1
//hpc2
ssh-copy-id -i ~/.ssh/id_ed25519.pub  hpc1
```

即可免密碼登入，可用以下測試，在node1執行以下若可以SSH到node2且不需要密碼為配置成功～

```bash
ssh hpc@hpc2
```




# NIS

請注意這邊環境使用
192.169.2.50    hpc-cpu-emc-node1
192.169.2.53    hpc-cpu-emc-node2
## NIS Server
安裝NIS
```bash=
sudo apt-get update
sudo apt-get install nis
```

設定NIS域名
```bash 
sudo sh -c 'echo "niscluster" > /etc/defaultdomain'
```

設定為NIS主伺服器
```
sudo nano /etc/default/nis
```
修改為：
```bash
NISSERVER=master
```

設定yp.conf檔案
```bash
sudo nano /etc/yp.conf
```
加入
```
domain niscluster server 127.0.0.1
```
設定允許的客戶端
```bash
sudo nano /etc/ypserv.securenets
```
確保有這些行：
```config
# 允許本機
255.0.0.0       127.0.0.0
# 允許內部網路
255.255.255.0   192.169.2.0
```
啟動NIS服務
```bash
sudo systemctl restart rpcbind
sudo systemctl restart ypserv
```
初始化NIS資料庫
```bash
sudo /usr/lib/yp/ypinit -m
```
當提示輸入NIS伺服器時，輸入：hpc-cpu-emc-node1 然後按 Ctrl+D

建立NIS地圖
```bash
sudo make -C /var/yp
```
確認NIS伺服器正常運行
```bash
rpcinfo -p localhost | grep yp
```


## NIS客戶端
安裝NIS套件

```bash
sudo apt-get update
sudo apt-get install nis
```
設定NIS域名
```bash
sudo sh -c 'echo "niscluster" > /etc/defaultdomain'
```
設定yp.conf檔案
```bash
sudo nano /etc/yp.conf
```
加入：
```bash
domain niscluster server 192.169.2.50
```
設定nsswitch.conf
```bash
sudo nano /etc/nsswitch.conf
```
修改以下行：
```bash
passwd:         files nis
group:          files nis
shadow:         files nis
hosts:          files dns nis
```
啟動客戶端服務
```bash
sudo systemctl restart rpcbind
sudo systemctl restart ypbind
```
測試NIS連接
```bash
ypwhich
```

![截圖 2025-03-09 晚上8.44.23.png](https://s3-web.infinirc.com/infinirc-web/articles/hpc-cluster/images/2025-03-09-8-44-23-lvkioodyw5n6z2svzn2o99u5.webp)
測試NIS功能
```bash
ypcat passwd
```


![截圖 2025-03-09 晚上8.44.56.png](https://s3-web.infinirc.com/infinirc-web/articles/hpc-cluster/images/2025-03-09-8-44-56-xfnvi2g22wak7iz5gj8l8g8c.webp)

---



# NFS
請注意這邊環境使用
192.169.2.50    hpc-cpu-emc-node1
192.169.2.53    hpc-cpu-emc-node2

關閉防火墻 node1、2
```bash
sudo systemctl stop firewalld ufw iptables
sudo systemctl disable  firewalld ufw iptables
sudo systemctl status firewalld ufw iptables
```

## node1

```bash
sudo apt-get install -y nfs-kernel-server
```


```bash
sudo chown nobody:nogroup /opt
```

```bash
sudo chmod 777 /opt
```

```bash
sudo nano /etc/exports

/home 192.169.2.52/32(rw,sync,no_root_squash,no_subtree_check)
/opt 192.169.2.52/32(rw,sync,no_root_squash,no_subtree_check)
```

```bash
sudo exportfs -a
```

```bash
sudo systemctl restart nfs-kernel-server
```

```bash
sudo systemctl status nfs-kernel-server
```

## node2

```bash
sudo apt-get install -y nfs-common
```


```bash
sudo mount -v hpc-cpu-emc-node1:/opt /opt
sudo mount -v hpc-cpu-emc-node1:/home /home
```

```bash
df -h
```

![截圖 2025-03-09 晚上9.07.39.png](https://s3-web.infinirc.com/infinirc-web/articles/hpc-cluster/images/2025-03-09-9-07-39-vatu7xm7d0gvl38ugd6kycr0.webp)

修改 `/etc/fstab` 以自動掛載
```bash
sudo nano /etc/fstab
```


```bash
hpc-cpu-emc-node1:/opt /opt nfs rdma,defaults 0 0
hpc-cpu-emc-node1:/home /home nfs rdma,defaults 0 0
```
測試掛載
```bash
sudo mount -a
```


# Environment Module

## 簡介

- 一種讓使用者可以**動態管理環境變數**的工具。
- 避免環境變數衝突。
- 方便不同軟體版本共存（如 Python 3.8 / 3.9）。

## 安裝與設定

### 安裝

```bash
sudo apt install environment-modules
```

### 啟用 modules

```bash
source /etc/profile.d/modules.sh
```

為了避免每次重開 terminal 都要使用指令，可以將它寫進 `/etc/profile`。

```bash
echo 'source /etc/profile.d/modules.sh' | sudo tee -a /etc/profile
```

## 基本使用方法

### 列出可用模組

```bash
module avail
```

### 載入模組

```bash
module load <module_name>
```

#### 範例: 載入 cuda/11.8 版本

```bash
module load cuda/11.8
```

### 檢視當前載入的模組

```bash
module list
```

### 卸載模組

```bash
module unload <module_name>
```

#### ex.

```bash
module unload cuda/11.8
```

### 切換模組版本

```bash
module switch <old_module> <new_module>
```

#### ex.

```bash
module switch cuda/11.8 cuda/12.0
```

### 清除所有已載入的模組

```bash
module purge
```

## Modulefile

### 建立自訂 module

- 自訂 module 存放於 `/usr/share/modules/modulefiles/` 或 `/etc/modulefiles/`。
- 也可以建立 `~/modules/` 來存放個人模組。

### 範例 `/usr/share/modules/modulefiles/`

```
│── core/                  # 基礎環境模組
│   ├── null
│── compiler/              # 編譯器相關模組
│   ├── gcc/
│   │   ├── 8.4
│   │   ├── 9.3
│   │   ├── 10.2
│   │   └── 11.1
│   ├── llvm/
│       ├── 12.0
│       ├── 13.0
│── mpi/                   # MPI 並行運算庫
│   ├── openmpi/
│   │   ├── 4.0.5
│   │   ├── 4.1.0
│   │   └── default -> 4.1.0   # 指向最新版本
│── apps/                  # 各種應用程式
│   ├── python/
│   │   ├── 3.8
│   │   ├── 3.9
│   │   ├── 3.10
│   │   └── default -> 3.10
│   ├── myapp/
│       ├── 1.0
│       ├── 2.0
│       └── default -> 2.0
```

### Modulefile 範例

#### cuda/11.8

```bash
#%Module1.0
##
## CUDA 11.8 modulefile
##

set cuda /usr/local/cuda-11.8

prepend-path PATH $cuda/bin
prepend-path LD_LIBRARY_PATH $cuda/lib64
```

### Module use（指定額外的 modulefile 目錄）

#### ex.

```bash
module use ~/modules
```



# munge

## 手動

安裝 Autoconf、Automake 和 Libtool

```bash
sudo apt-get update
sudo apt-get install autoconf automake libtool
sudo apt-get update
sudo apt-get install libssl-dev
```

```bash
git clone https://github.com/dun/munge.git
cd munge
./bootstrap
```

```bash
./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var --runstatedir=/run
make
make check
sudo make install
```

## 套件

```
sudo apt install munge
```

生成密鑰

```bash
sudo dd if=/dev/urandom bs=1 count=1024 | sudo tee /etc/munge/munge.key > /dev/null
sudo chown munge: /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key

```

```bash
# 在 node1 上
sudo ls -la /etc/munge/munge.key
# 確認權限和擁有者是否正確

# 重新複製 key 到 node2
sudo chmod 644 /etc/munge/munge.key  # 暫時修改為可讀
sudo scp /etc/munge/munge.key ntcucshpc@hpc-cpu-emc-node2:/tmp/munge.key
sudo chmod 400 /etc/munge/munge.key  # 恢復原來的權限

# 在 node2 上
sudo mv /tmp/munge.key /etc/munge/munge.key
sudo chown munge: /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key

```




```bash
# 在兩個節點上執行
sudo systemctl restart munge
sudo systemctl status munge  # 檢查服務狀態是否為 active (running)
```


測試
```bash
munge -n
# 從 node1 到 node2
munge -n | ssh hpc-cpu-emc-node2 unmunge

# 從 node2 到 node1
# 在 node2 上執行
munge -n | ssh hpc-cpu-emc-node1 unmunge

```





# Slurm


## 安裝

控制節點：hpc-cpu-emc-node1（運行 slurmctld）
計算節點：hpc-cpu-emc-node2（只運行 slurmd）

在兩個節點上執行：

```bash
sudo apt update
sudo apt install -y slurm-wlm
```

在控制節點（hpc-cpu-emc-node1）上額外安裝：
```bash
sudo apt install -y slurmctld
```
在兩個節點上都安裝：
```bash
sudo apt install -y slurmd
```

##  創建所需目錄與設置權限
在兩個節點上都執行：
```bash
sudo mkdir -p /var/spool/slurmd
sudo mkdir -p /var/log/slurm
sudo touch /var/log/slurm/slurmd.log
sudo chown -R slurm:slurm /var/log/slurm
sudo chmod 755 /var/log/slurm
sudo chmod 644 /var/log/slurm/slurmd.log
```

在控制節點（hpc-cpu-emc-node1）額外執行：
```bash
sudo mkdir -p /var/spool/slurmctld
sudo touch /var/log/slurm/slurmctld.log
sudo chown slurm:slurm /var/log/slurm/slurmctld.log
sudo chmod 644 /var/log/slurm/slurmctld.log
sudo mkdir -p /var/lib/slurm/slurmctld
sudo chown slurm:slurm /var/lib/slurm/slurmctld
sudo chmod 755 /var/lib/slurm/slurmctld
```

## 配置 Slurm
在控制節點上創建並編輯 slurm.conf：
```bash
sudo mkdir -p /etc/slurm
sudo nano /etc/slurm/slurm.conf
```

```config
# slurm.conf
ClusterName=hpc-cluster
SlurmctldHost=hpc-cpu-emc-node1

# 控制配置
SlurmctldPidFile=/run/slurmctld.pid
SlurmdPidFile=/run/slurmd.pid
SlurmdSpoolDir=/var/lib/slurm/slurmd
StateSaveLocation=/var/lib/slurm/slurmctld
SlurmUser=slurm

# 節點通信
SlurmctldPort=6817
SlurmdPort=6818
AuthType=auth/munge

# 進程跟踪
# 修改這裡：使用 proctrack/linuxproc 而不是 proctrack/cgroup
ProctrackType=proctrack/linuxproc

# 計劃和選擇
SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core

# 超時設定
KillWait=30
MinJobAge=300
SlurmctldTimeout=120
SlurmdTimeout=300
Waittime=0
InactiveLimit=0

# 日誌設定
SlurmctldDebug=info
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdDebug=info
SlurmdLogFile=/var/log/slurm/slurmd.log
JobAcctGatherFrequency=30
JobAcctGatherType=jobacct_gather/none
JobCompType=jobcomp/none

# 作業資源控制
TaskPlugin=task/affinity
SwitchType=switch/none
ReturnToService=1

# 節點定義 (根據 slurmd -C 輸出)
NodeName=hpc-cpu-emc-node1 CPUs=8 RealMemory=21978 Sockets=1 CoresPerSocket=8 ThreadsPerCore=1 State=UNKNOWN
NodeName=hpc-cpu-emc-node2 CPUs=8 RealMemory=21978 Sockets=1 CoresPerSocket=8 ThreadsPerCore=1 State=UNKNOWN

# 分區定義
PartitionName=computing Nodes=hpc-cpu-emc-node1,hpc-cpu-emc-node2 Default=YES MaxTime=INFINITE State=UP

```

從 proctrack/cgroup 改為 proctrack/linuxproc

```bash
sudo scp /etc/slurm/slurm.conf ntcucshpc@hpc-cpu-emc-node2:/tmp/
# 在計算節點上執行
sudo mkdir -p /etc/slurm
sudo mv /tmp/slurm.conf /etc/slurm/
```

##  啟動 Slurm 服務
在控制節點（hpc-cpu-emc-node1）上執行：
```bash
sudo systemctl start slurmctld
sudo systemctl enable slurmctld
sudo systemctl start slurmd
sudo systemctl enable slurmd
```
在計算節點（hpc-cpu-emc-node2）上執行：
```bash
sudo systemctl start slurmd
sudo systemctl enable slurmd
```
## 檢查服務狀態
```bash
sudo systemctl status slurmd
```
在控制節點檢查 slurmctld 服務：
```bash
sudo systemctl status slurmctld
```
## 測試
在控制節點上執行：
```bash
sinfo
srun -N 2 hostname
```

![截圖 2025-03-13 上午10.51.05.png](https://s3-web.infinirc.com/infinirc-web/articles/hpc-cluster/images/2025-03-13-10-51-05-ops8f3d4brskskdwv9yizbug.webp)



# OpenMPI
Open MPI是一個消息傳遞接口庫項目，結合了其他幾個項目的技術和資源。


若需要支援 fortran，需要安裝 gfortran

```bash
sudo apt install gfortran
```

安裝 GNU Autotools

```bash
sudo apt install autoconf automake libtool
```

```bash
wget https://download.open-mpi.org/release/open-mpi/v5.0/openmpi-5.0.3.tar.gz
```

```bash
tar xf ./openmpi-5.0.3.tar.gz
```

```bash
cd openmpi-5.0.3
```

```bash
sudo ./configure --prefix=/opt/openmpi-5.0.3 --with-cuda=/usr/local/cuda-11.8/

```

InfiniBand

```bash
sudo ./configure --prefix=/opt/openmpi-5.0.3 \\
    --with-cuda=/usr/local/cuda-11.8 \\
    --with-verbs \\
    --enable-mca-no-build=btl-openib \\
    --enable-mpirun-prefix-by-default \\
    --with-ucx \\
    --enable-orterun-prefix-by-default
```

```bash
sudo ./configure --prefix=/opt/openmpi-5.0.3 --with-cuda=/usr/local/cuda-11.8 --with-verbs --enable-mca-no-build=btl-openib --enable-mpirun-prefix-by-default --with-ucx --enable-orterun-prefix-by-default
```

```bash
sudo make all
# 或
sudo make
```

```bash
sudo make install
```

```bash
nano ~/.bashrc
```

```bash
export PATH="/opt/openmpi-5.0.3/bin:$PATH"
export LD_LIBRARY_PATH="/opt/openmpi-5.0.3/lib:$LD_LIBRARY_PATH"
```

```bash
source ~/.bashrc
```

檢查是否使用IB

```bash
ompi_info --all | grep ucx
```
