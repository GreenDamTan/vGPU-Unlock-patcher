### 特别提醒
记得要用`git clone --recursive`来克隆仓库

直接点那个zip下载可能导致unlock这个submodules没拉下来，进而导致构建失败

嫌麻烦可以直接用成品，大多数人的优先选择

毕竟github在大清真的难访问，而且都是洋文大多数人也看不懂
### 变更补充说明
新增了zstd压缩，以应对大清特有之国情

比如说什么百度网盘限速10kb之类的

前置步骤做完了但没mdev可以尝试
```shell
systemctl restart {nvidia-vgpud.service,nvidia-vgpu-mgr.service}
```
跑完了还是不出来那就找群友问问，什么QQ群之类的

至于你说discord，你都看的懂英文了还需要这笔记吗

### 关于CMP系列
关于CMP矿卡系列的修正，不完全在这个repo里面

主要是改起来麻烦，就不塞一起了

就比如说CMP 100的支持，那个unlock里面还得改，但又不在这个repo里面

然后还有40HX之类的，bar大小又不一致

### 补丁驱动的自用案例
进行操作前请确保该下的该改的，还有vgpu-kvm驱动准备好
#### 杂种驱动
需要提前下载普通驱动
```shell
https://us.download.nvidia.com/XFree86/Linux-x86_64/550.90.07/NVIDIA-Linux-x86_64-550.90.07.run
```
然后再创建杂种驱动
```shell
./patch.sh --spoof-devid --repack --force-nvidia-gpl-I-know-it-is-wrong --enable-nvidia-gpl-for-experimenting --test-dmabuf-export --envy-probes --zstd general-merge
```
#### 纯种驱动
```shell
./patch.sh --spoof-devid --repack --force-nvidia-gpl-I-know-it-is-wrong --enable-nvidia-gpl-for-experimenting --test-dmabuf-export --envy-probes --zstd vgpu-kvm
```
### 安装成品驱动命令说明
由于550版本开始，NVIDIA默认在部分卡上使用kernel-open驱动

因此需要显式指定使用kernel驱动

根据你使用的场景选取以下的命令进行安装，如果需要其他参数请自行追加
```shell
./nvidia-installer -m kernel
```
或者
```shell
./NVIDIA-Linux-x86_64-550.54.10-vgpu-kvm-patched.run -m kernel
```