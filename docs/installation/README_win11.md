# how to use for Win11
注：本次部署并非在 ~/kAFL 目录下，而是在 ~/tmp2/kAFL 目录下，部署的时候可能需要修改一些绝对路径    
此外，配套的 kafl/examples 从 [kafl.targets.win11](https://github.com/Rosayxy/kafl.targets.win11) 仓库 clone 的，在进行部署的时候，可能需要修改一波绝对路径   

# 0. Tested Environment <a name="section-0"></a>
----------------------------------
```
CPU : Intel i-7 12700K
RAM : 84G
GPU : Nvidia Geforce 1060 super
OS : Ubuntu 20.04.6 LTS, Ubuntu 22.04.6 LTS  
```

# 1. Install dependencies <a name="section-1"></a>
----------------------------------
```bash
sudo apt-get update -y
sudo apt-get install gcc git make curl vim python3 python3-venv -y
```

# 2. Clone this repo & change kernel to 6.0.0-nyx+ <a name="section-2"></a>
----------------------------------
```bash
cd ~
git clone https://github.com/Rosayxy/msFuzz.git kAFL
cd kAFL
make deploy
reboot
```

# 3. Build the Windows VM Template <a name="section-3"></a>
----------------------------------

因为之前的 installation 中，*Import into libvirt* 这步一直超时，且难以直接运行 qemu 排查，以及看到了 kAFL 本身是裸跑 qemu-system 的形式，所以就直接把第二次和第一次用 ansible 的操作合二为一了，直接在 qemu image 里面用 ansible 把我们的 agent 和 harness 塞进去    

并且这步进行了适配 Win11 uefi boot 的操作   
```bash
# 这里先把 harness 和 agent 编译出来，然后一起部署到 image 里面
cd kafl/examples/windows_x86_64/
mkdir -p bin/driver 
cp ../../fuzzer/Utils/Harness_for_nyx.sys ./bin/driver

1. Edit src/driver/vuln_test.c
-> Change macro definitions of "VULN_DRIVER_NAME", "VULN_DRIVER_NAME2", "VULN_DRIVER_NAME3" to your target driver
-> Then update the "CreateFile()" call (around line 232) to use one of:
   - Symbolic link: "\\\\.\\MyDriverDevice"
   - NT namespace: "\\\\?\\GLOBALROOT\\Device\\MyDriverDevice"
   - PnP interface: "\\\\?\\USB#VID_…#{GUID}…"

make compile

# enable uefi boot
cd ~/tmp2/kAFL
cp /usr/share/OVMF/OVMF_CODE_4M.fd code.img
cp /usr/share/OVMF/OVMF_VARS_4M.fd efivars.img
chmod +w efivars.img

#  being ready to build!
cd ./kafl/examples/templates/windows
cp -r ../../windows_x86_64/bin . 
mkdir output-windows_1
cd output-windows_1
qemu-img create -f qcow2 win10.qcow2 64G
make build 
# make build 这一步中，需要 vnc 连上去，先一直按住 esc 进 bios，然后 exit shell, 选 uefi boot，点进去第一个
# 之后启动的时候，会告诉你大概是你挂在的磁盘不可用，此时需要 delete partition 然后 create partition, 选择大小最大的那个分区（大概有 63.9G 吧）点 next 就行了
```

注：调试的时候可以用以下命令   

```bash
/usr/bin/qemu-system-x86_64 -machine type=pc,accel=kvm -netdev user,id=user.0,hostfwd=tcp::4107-:5985 -name win10.qcow2 -smp cpus=4,sockets=4 \
-drive if=pflash,format=raw,unit=0,file=/home/mimi/tmp2/kAFL/code.img \
-drive if=pflash,format=raw,unit=1,file=/home/mimi/tmp2/kAFL/efivars.img \
-drive file=/home/mimi/.cache/packer/c7a6dd5c537a710fa9be2731dfec58b7361557e0.iso,media=cdrom,if=none,id=cd0 \
-drive file=/home/mimi/tmp2/kAFL/kafl/examples/templates/windows/output-windows_1/win10.qcow2,if=ide,cache=writeback,discard=ignore,format=qcow2 \
-cpu host -boot once=d -vnc 0.0.0.0:47 -m 8192M -usb -device usb-tablet -device ide-cd,drive=cd0 -device rtl8139,netdev=user.0    
```

然后接 vnc 看输出


# 4. Run Fuzz <a name="section-4"></a>
----------------------------------
```bash
cd ~/kAFL
make env
cd kafl/examples/windows_x86_64/
mkdir -p ./seed
./run.sh

```
