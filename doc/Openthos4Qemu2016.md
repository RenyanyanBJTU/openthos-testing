# openthos在qemu上的自动测试
##用qemu-system-x86_64测试
1、首先，确保使用的机器支持虚拟化，输入如下命令,如果出现vmx或者svm等信息说明支持，如果没有信息则可能无法运行安卓镜像。
```
egrep '(vmx|svm)' /proc/cpuinfo
```
2、安装qemu和qemu-kvm
```
sudo apt-get install qemu
sudo apt-get install qemu-kvm
```
3、输入如下命令查询kvm模块是否加载成功，若出现kvm和kvm-intel等说明成功
```
lsmod | grep kvm 
```
如果没有，则运行如下命令加载模块
```
sudo modprobe kvm
sudo modprobe kvm-intel 
```
如果还是不成功，则是机器支持虚拟化但没有开启虚拟化，可以重启机器,在bios里的虚拟化选项里开启。
<br><br>

###Live CD模式运行安卓镜像
```
qemu-system-x86_64 -enable-kvm -m 4G -cdrom android_x86.iso -vga std -serial stdio
#notice：镜像的路径和全称要正确。另外分配的内存如果太小会启动不起来镜像,建议至少2G。
```
###安装模式
参考[在QEMU上运行](http://www.android-x86.org/documents/qemuhowto)和[在virtualbox上运行](http://www.android-x86.org/documents/virtualboxhowto#Advanced)的官方文档，创建一个磁盘镜像
```
qemu-img create -f raw android.raw 8G
qemu-system-x86_64 -enable-kvm -m 4G -cdrom android_x86.iso -vga std -serial stdio -hda android.raw -boot d
```
进入debug mode  

1. "fdisk /dev/sda", then type:
 1. "n" (new partition)  
 2. "p" (primary partition)  
 3. "1" (1st partition)  
 4. "1" (first cylinder)  
 5. "xx" (choose the last cylinder, leaving room for a 2nd partition)  
 6. "w" (write the partition)  
2. Repeat #1, but call it partition 2, and use the remaining cylinders.
3. "mdev -s"  
4. "mke2fs -j -L DATA /dev/sda1"  
5. "mke2fs -j -L SDCARD /dev/sda2"  
6. Reboot ("reboot -f")  
7. 进入安装模式，安装在第一个分区，安装grub，但不要选择安装uefi  

安装完成后，关闭qemu，重新运行
```
qemu-system-x86_64 -enable-kvm -m 4G -vga std -serial stdio -drive file=android-x86-6.0.raw,format=raw,index=0,media=disk
```

###开机自动运行脚本
将img文件，挂载到系统上，在/system/etc/init.sh文件的return 0前面加入自动执行的语句即可。  

###自动拷贝文件脚本
见copy.sh  
`fdisk -l android.raw`  
查看扇区情况，假设第一个扇区的起点是63，计算出63×512=32256，要挂载到/home/cscw/mnt1，欲拷贝文件为kernel，拷贝到/home/cscw/mnt1/android-2016-02-29/kernel，则以如下命令执行脚本  
`./copy.sh android.raw 32256 /home/cscw/mnt1 kernel android-2016-02-29/kernel`

###通过端口转发使用adb连接kvm中的安卓系统
使用以下命令将客户机的5555端口也就是adb要连接的端口转发到主机的12346端口  
`kvm -m 4G -drive file=android-x86-6.0.raw,format=raw,index=0,media=disk-net nic -net user,hostfwd=tcp::12346-:5555`  
然后就可以`adb connect localhost:12346`连接到kvm中的安卓系统了   
