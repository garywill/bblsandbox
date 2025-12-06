# Box-in-box Linux Sandbox

可完全自定义、可无限嵌套的Linux沙箱。Box-in-box Linux （下文简称BBL）。

## 基本特性

- 免安装、精简依赖。单文件Python脚本，随处复制，依使用需求修改选项

- 不需root；不需守护进程；不需任何主机的Cap或suid

- 不留痕，不主动在家目录或硬盘任何位置留下文件。`/tmp`内的临时文件自动清理

- PID NS 统领所有子进程，便于一键杀死不遗漏

- 完全自定义，通过选项控制各个实现细节

- 多层namespace任意配置，控制每层隔离程度。“不信任app”与“半信任app”可在一个沙箱不同层运行，且可以做到无限嵌套

- 无镜像容器。不需要像Docker、LXC那样下载系统镜像。用现有真实系统作为基础，内部vim、git等工具无需重复安装，隔离其余用户数据和文件

## 为什么创建这个脚本？它安全性如何？

我暂时把它称为个人的Firejail实验替代品。Firejail、Bubblewrap等工具不让用户做完全精细化的控制，就连官方工具unshare也是。为了学习内核的namespace、权限等机制，也为了实现完全自定义，以及其他工具做不到的的沙箱无限嵌套。

这个目前是非常早期的阶段，可以使用，但要知道，这个脚本目前无专业团队参与。

## 功能列表与完成状态

- 不需root；不需守护进程；不需任何主机的Cap或suid
- [x] 可完全自定义的多层嵌套namespace
    - [x] 每层pid ns、mount ns 等 每种namespace的隔离选项控制
    - [x] 每层的新rootfs挂载与细粒度文件系统路径建立方式控制
        - [x] rw挂载
        - [x] ro挂载
        - [] overlay
        - [x] 创建或临时覆盖文件及其内容(rw/ro)；tmpfs目录(rw/ro)
    - [x] 内部环境变量控制
    - [x] 内部uid变0（提权）；某层uid变回1000(降权）；Drop caps；noNewPriv
- [x] 可挂载AppImage
- 容器内使用GUI
    - [x] 可选暴露真实X11接口
    - [] 可选使用Xephyr隔离X11
    - [] 可选使用Xpra隔离的无缝X11代理
    - [] 可选暴露wayland接口
    - [] 可选Xephyr/Xvfb/x11vnc窗口内运行的隔离的完整桌面环境
- [] 可选暴露真实物理硬件，或仅显卡渲染所需部分
- DBUS
    - [x] 可选暴露真实dbus session接口
    - [] 可选过滤dbus通信
- [] 每层子容器shell接口暴露给主机
- [] 可选的seccomp
- [] 可选的网络流量控制

## 依赖

TBD

## 简单用例 

**例子1：** 沙箱内运行下载的AppImage文件

从网络下载任意app的`.AppImage`文件。BBL实现了在内部预先挂载AppImage，不需要把fuse挂载权限给AppImage。

复制一份BBL的`.py`脚本，与下载的AppImage放在一起:

```
/anypath/freecad/bblsbxrun_freecad.py
/anypath/freecad/FreeCAD.AppImage
/anypath2/projects_save/
```

编辑我们的`.py`文件，配置：

```python
sandbox_name='freecad', # 沙箱名称
user_mnts = [
    d(mttype='appimage', appname='freecad',  src=f'{si.startdir_on_host}/FreeCAD.AppImage'),
    d(mttype='bind', src='/anypath2/projects_save/', src_same_dist=1), 
],
gui="realX", # 使用真实的 X11
```

会把AppImage里的内容挂载到容器内的`/sbxdir/apps/freecad/`下。 启动容器后，在内运行`/sbxdir/apps/run_freecad`即启动我们的app。

容器内app所创建的工程可以保存在`/anypath2/projects_save/`下（用了`src_same_dist`挂载工程目录，容器内外皆以同一路径访问此目录）

**例子2：** 沙箱内运行下载的二进制程序

例如下载`firefox.tar.xz`, 解压，像上例一样把解压出来的文件和复制的一份BBL的`.py`脚本放一起:

```
/anypath/ffx/bblsbxrun_firefox.py
/anypath/ffx/firefox/.... (内含firefox-bin, *.so 等 解压出来的文件)
```

编辑我们的`.py`文件，配置：

```python
sandbox_name='firefox', # 沙箱名称
user_mnts = [
    d(mttype='robind', src=f'{si.startdir_on_host}/firefox', src_same_dist=1), 
    # 也可以去掉上面的`src_same_dist`而改为`dist='/opt/firefox'`。
],
gui="realX", # 使用真实的 X11
dbus_session="allow", # 输入法等通信需要dbus
```

以上尚未挂载持久化的路径以保存浏览器profile目录。若需要，可创建一个`fakehome`目录

```
/anypath/ffx/bblsbxrun_firefox.py
/anypath/ffx/fakehome
/anypath/ffx/firefox/.... (内含firefox-bin, *.so 等 解压出来的文件)
```

并配置

```python
homedir=f'{si.startdir_on_host}/fakehome',
```

即可持久化保存容器内家目录文件。（`/anypath/ffx/fakehome`会被挂载到容器内的`/home/用户名`）

**例子3：** 沙箱内直接使用自己的vimrc配置

```python
user_mnts = [
    d(mttype='robind', src=f'{si.HOME}/.vimrc', src_same_dist=1), 
],
```

## 容器文件系统

一般来说，容器内所运行的“不可信app”所看到的文件系统类似如下：

```yml
// # 真实的系统目录
{'plan': 'rosame', 'dist': '/bin', 'src': '/bin'}
{'plan': 'rosame', 'dist': '/etc', 'src': '/etc'}
{'plan': 'rosame', 'dist': '/lib64', 'src': '/lib64'}
.....

// # 最小的/dev
{'plan': 'rotmpfs', 'dist': '/dev'}
{'plan': 'same', 'dist': '/dev/console', 'src': '/dev/console'}
{'plan': 'same', 'dist': '/dev/null', 'src': '/dev/null'}
{'plan': 'same', 'dist': '/dev/random', 'src': '/dev/random'}
{'plan': 'devpts', 'dist': '/dev/pts'}
{'plan': 'tmpfs', 'dist': '/dev/shm'}
......

// # 创建空的临时目录
{'plan': 'tmpfs', 'dist': '/run'}
{'plan': 'tmpfs', 'dist': '/run/user/1000'}
{'plan': 'tmpfs', 'dist': '/tmp'}
......

// # 以下根据用户配置情况而变
{'plan': 'appimg-mount', 'src': '/anypath/freecad/FreeCAD.AppImage', 'dist': '/sbxdir/apps/freecad'}
{'plan': 'robind', 'src': '/anypath/ffx/firefox', 'dist': '/opt/firefox'}
{'plan': 'robind', 'dist': '/tmp/.X11-unix/X0', 'src': '/tmp/.X11-unix/X0'}
{'plan': 'robind', 'dist': '/tmp/dbus_session_socket', 'src': '/run/user/1000/bus'}
```

以上文件系统已经写进模板里，不需要普通用户去创建。

