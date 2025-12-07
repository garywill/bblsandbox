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

- [x] 不需root；不需守护进程；不需任何主机的Cap或suid
- [x] 可完全自定义的多层嵌套namespace
    - [x] 每层pid ns、mount ns 等 每种namespace的隔离选项控制
    - [x] 每层的新rootfs挂载与细粒度文件系统路径建立方式控制
        - [x] rw挂载
        - [x] ro挂载
        - [ ] overlay
        - [x] 创建或临时覆盖文件及其内容(rw/ro)；tmpfs目录(rw/ro)
    - [x] 内部环境变量控制
    - [x] 内部uid变0（提权）；某层uid变回1000(降权）；Drop caps；noNewPriv
- [x] 可挂载AppImage
- 沙箱内使用GUI
    - [x] 可选暴露真实X11接口
    - [x] 可选使用Xephyr隔离X11
    - [ ] 可选使用Xpra隔离的无缝X11代理
    - [ ] 可选暴露wayland接口
    - [ ] 可选Xephyr/Xvfb/x11vnc窗口内运行的隔离的完整桌面环境
- [ ] 可选暴露真实物理硬件，或仅显卡渲染所需部分
- DBUS
    - [x] 可选暴露真实dbus session接口
    - [ ] 可选过滤dbus通信
- [ ] 每层子容器shell接口暴露给主机
- [ ] 可选的seccomp
- [ ] 可选的网络流量控制

## 依赖

必须：

- 现代Linux Kernel (支持unprivileged user namespace)
- glibc
- Python >= 3.12
- bash

可选：

- squashfuse, Xephyr

## 简单用例 

以下几个简单例子中，沙箱内app进程都只能看到只读的系统基础目录、空白的家目录，和用户明确指定了可见的路径或接口。

**例子1：** 沙箱内运行下载的AppImage文件

从网络下载任意app的`.AppImage`文件。

复制一份BBL的`.py`脚本，与下载的AppImage放在一起:

```
/anyhdd/freecad/bblsbxrun_freecad.py
/anyhdd/freecad/FreeCAD.AppImage
/anyhdd2/projects_save/
```

编辑我们的`.py`文件，配置：

```python
sandbox_name='freecad', # 沙箱名称
user_mnts = [
    d(mttype='appimage', appname='freecad',  src=f'{si.startdir_on_host}/FreeCAD.AppImage'),
    d(mttype='bind', src='/anyhdd2/projects_save/', SDS=1), 
],
gui="realX", # 使用真实的 X11
```

BBL实现了在内部预先挂载AppImage，不需要把fuse挂载权限给AppImage。会把AppImage里的内容挂载到沙箱内的`/sbxdir/apps/freecad/`下。 启动沙箱后，在内运行`/sbxdir/apps/run_freecad`即启动我们的app。

沙箱内app所创建的工程可以保存在`/anyhdd2/projects_save/`下（用了`SDS`挂载工程目录，沙箱内外皆以同一路径访问此目录，`SDS`是"src and dist are same"的缩写）

**例子2：** 沙箱内运行下载的二进制程序

例如下载`firefox.tar.xz`, 解压，像上例一样把解压出来的文件和复制的一份BBL的`.py`脚本放一起:

```
/anyhdd/ffx/bblsbxrun_firefox.py
/anyhdd/ffx/firefox/.... (内含firefox-bin, *.so 等 解压出来的文件)
```

编辑我们的`.py`文件，配置：

```python
sandbox_name='firefox', # 沙箱名称
user_mnts = [
    d(mttype='robind', src=f'{si.startdir_on_host}/firefox', SDS=1), 
    # 也可以去掉上面的`SDS`而改为`dist='/sbxdir/apps/firefox'`。
],
gui="realX", # 使用真实的 X11
dbus_session="allow", # 输入法等通信需要dbus
```

以上尚未挂载持久化的路径以保存浏览器profile目录。若需要，可创建一个`fakehome`目录

```
/anyhdd/ffx/bblsbxrun_firefox.py
/anyhdd/ffx/fakehome
/anyhdd/ffx/firefox/.... (内含firefox-bin, *.so 等 解压出来的文件)
```

并配置

```python
homedir=f'{si.startdir_on_host}/fakehome',
```

即可持久化保存沙箱内家目录文件。（`/anyhdd/ffx/fakehome`会被挂载到沙箱内的`/home/用户名`）

**例子3：** 沙箱内直接使用自己的vimrc配置

```python
user_mnts = [
    d(mttype='robind', src=f'{si.HOME}/.vimrc', SDS=1), 
],
```

## 沙箱分层结构

这是个可以自由嵌套的沙箱。脚本内已经设置有默认的嵌套模板：

```
Linux Host 
  |
 layer1 (用于统一管理；隔离pid ns；内部提权)
  |
  |
 layer2 (半信任空间：隔离mount ns；屏蔽用户设置的全局屏蔽路径）
   |
   |--layer2a (降权；用于运行信任的辅助程序，如 xpra client、dbus-proxy ...）
   |
 layer2h (过度)
    |
  layer3 (不信任空间：隔离所有ns；
    |       可见系统基础目录，其余仅用户挂载进去的路径可见）
    |
    |--layer4 (降权；用于运行app)
    |--layer4a (降权；用于运行不信任的辅助程序，如 xpra server ...)
```

（layer2a和layer4a都用于运行辅助程序，区别在于layer2a可以访问真实的X11接口、dbus接口、真实硬盘文件，而layer4a则不需要访问这些）

以上这个默认的嵌套模板普通用户不需要修改，只需要修改用户选项部分即可。

沙箱成功启动后，用户获得的 user shell （如果要） 是在layer4内。

> 本项目处于早期阶段，不排除以后有修改设计的可能性

模板设置方式类似如下：（进阶用户了解）

```python
layer1 = d( # 第1层
    layer_name='layer1', # 默认模板的 layer_name 不要修改
    unshare_pid=True, unshare_user=True, ......
    
    sublayers = [
        d( # 第2层
            layer_name='layer2', # 默认模板的 layer_name 不要修改
            unshare_pid=True, unshare_mnt=True, ....
            newrootfs=True, fs=[ ..... ], ....
            
            sublayers = [
                d( layer_name='layer2a', .... ), 
                d( 
                    layer_name='layer2h', 
                    sublayers = [
                        d( layer_name='layer3', ..... , newrootfs=True, fs=[ ..... ], .....
                            sublayers=[ # 第4层
                                d( layer_name='layer4', .....  , user_shell=True ),
                                d( layer_name='layer4a', ..... ),
                            ],
                        ),
                    ] 
                )
            ],
        )
    ],
)
```
以上只是非常粗略地展示一下默认模板，想要了解的请打开代码查看。

## 启动流程

每层容器启动及配置流程：

1. 读取本层配置
1. 根据配置进行unshare（开始ns隔离）
1. fork。以下步骤都在子进程中执行
1. 根据配置进行`/proc/self/uid_map`等写入（内部提权、降权）
1. 根据配置建立及挂载本层的新rootfs
1. 根据配置进行pivot_root
1. 根据配置修改环境变量
1. 根据配置降权
1. 根据配置启动 user shell ，或启动下一层子容器，或启动某app

> 本项目处于早期阶段，不排除以后有修改设计的可能性

## 沙箱内文件系统

一般来说，沙箱内所运行的“不信任app”所看到的文件系统类似如下：

```yml
// # 真实的系统目录
{'plan': 'robind', 'dist': '/bin', 'src': '/bin'}
{'plan': 'robind', 'dist': '/etc', 'src': '/etc'}
{'plan': 'robind', 'dist': '/lib64', 'src': '/lib64'}
.....

// # 最小的/dev
{'plan': 'rotmpfs', 'dist': '/dev'}
{'plan': 'bind', 'dist': '/dev/console', 'src': '/dev/console'}
{'plan': 'bind', 'dist': '/dev/null', 'src': '/dev/null'}
{'plan': 'bind', 'dist': '/dev/random', 'src': '/dev/random'}
{'plan': 'devpts', 'dist': '/dev/pts'}
{'plan': 'tmpfs', 'dist': '/dev/shm'}
......

// # 创建空的临时目录
{'plan': 'tmpfs', 'dist': '/home/username'}
{'plan': 'tmpfs', 'dist': '/run'}
{'plan': 'tmpfs', 'dist': '/run/user/1000'}
{'plan': 'tmpfs', 'dist': '/tmp'}
......

// # 以下根据用户配置情况而变
{'plan': 'appimg-mount', 'src': '/anyhdd/freecad/FreeCAD.AppImage', 'dist': '/sbxdir/apps/freecad'}
{'plan': 'robind', 'src': '/anyhdd/ffx/firefox', 'dist': '/sbxdir/apps/firefox'}
{'plan': 'robind', 'dist': '/tmp/.X11-unix/X0', 'src': '/tmp/.X11-unix/X0'}
{'plan': 'robind', 'dist': '/tmp/dbus_session_socket', 'src': '/run/user/1000/bus'}

// # 沙箱配置目录
{'dist': '/sbxdir'}
```

（以上所列文件系统已经写进模板里，不需要用户去创建）

`/sbxdir`是BBL沙箱所需要的目录，它包含：

- AppImage挂载点（与普通用户有关，以下其余普通用户可以不了解）
- 本层及本层的子层的配置信息
- 本层与layer1及与主机通信所需要的文件
- 启动子层所用的脚本
- 子层的新rootfs挂载点
- ...

## 如何编辑多层嵌套模板

TBD
