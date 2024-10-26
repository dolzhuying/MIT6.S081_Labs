## MIT6.S081环境配置

虚拟机：ubuntu 20.04（其他版本会有包的兼容性问题）

**1.换源**

* 安装好ubuntu后，默认软件更新源为国外的，速度慢，需要更换成国内的源，首先备份源列表：

  ```
  # 首先备份源列表
  sudo cp /etc/apt/sources.list /etc/apt/sources.list_backup
  ```

* 打开sources.lists文件更换源，在文件最前面添加源，这里选择中科大的：

  ````
  # 打开sources.list文件
  sudo gedit /etc/apt/sources.list
  -----------------------------------------
  #中科大源
  deb https://mirrors.ustc.edu.cn/ubuntu/ focal main restricted universe multiverse
  deb-src https://mirrors.ustc.edu.cn/ubuntu/ focal main restricted universe multiverse
  deb https://mirrors.ustc.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
  deb-src https://mirrors.ustc.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
  deb https://mirrors.ustc.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
  deb-src https://mirrors.ustc.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
  deb https://mirrors.ustc.edu.cn/ubuntu/ focal-security main restricted universe multiverse
  deb-src https://mirrors.ustc.edu.cn/ubuntu/ focal-security main restricted universe multiverse
  deb https://mirrors.ustc.edu.cn/ubuntu/ focal-proposed main restricted universe multiverse
  deb-src https://mirrors.ustc.edu.cn/ubuntu/ focal-proposed main restricted universe multiverse
  ````

* 更新包索引：

  ```
  sudo apt-get update
  sudo apt-get upgrade
  ```

**2.安装RISCV编译工具链**

```
sudo apt install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu libglib2.0-dev libpixman-1-dev gcc-riscv64-unknown-elf
```

**3.安装QEMU**

QEMU用于在我们机器上(X86)模拟RISC-V架构的CPU，编译生成的risc-v平台的机器码，需要通过模拟cpu执行。

创建文件夹以容纳QEMU和之后的项目文件夹，这里是 ~/mit6s081 ，下面命令在该目录下执行：

```
wget https://download.qemu.org/qemu-5.1.0.tar.xz  
tar xvf qemu-5.1.0.tar.xz //解压
cd qemu-5.1.0 //这里切换目录
./configure --disable-kvm --disable-werror --prefix=/usr/local --target-list="riscv64-softmmu"
make 
sudo make install
```

**4.测试**

* 下载xv6源码（在 ~/mit6s081 执行），切入源码主目录，分支切换为util

  ```
  git clone git://g.csail.mit.edu/xv6-labs-2021
  cd xv6-labs-2021
  git checkout util.
  ```

* 在xv6-labs-2020下编译，如果能进入xv6的shell，看到xv6 kernel is booting 则表示实验环境已搭建成功

  ```
  make
  make qemu
  ```

​      按下ctrl+a松开后再按x退出qemu

* 检查工具链：

  ```
  riscv64-unknown-elf-gcc --version
  qemu-system-riscv64 --version
  ```

* 检查调试工具：

  初次，需要设置.gdbinit

  ```
  echo set auto-load safe-path / >> ~/.gdbinit
  ```

  这里需要开启两个窗口，一个运行qemu，一个运行调试器gdb。运行qemu的窗口执行make qemu-gdb后等待gdb的连接

  ```
  make qemu-gdb
  ```

  看到类似：

  ```
  sed "s/:1234/:26000/" < .gdbinit.tmpl-riscv > .gdbinit
  *** Now run 'gdb' in another window.
  qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -S -gdb tcp::26000
  ```

  然后打开另一个终端，同样在xv6-labs-2020下：

  ```
  gdb-multiarch -q kernel/kernel
  ```

  看到如下则环境无问题：

  ```
  Reading symbols from kernel/kernel...
  The target architecture is set to "riscv:rv64".
  0x0000000000001000 in ?? ()
  (gdb) 
  ```

  

  