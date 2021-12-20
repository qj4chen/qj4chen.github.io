---
layout: post
title: 记录安装open-AI的gym库时的步骤，以防再次踩坑
categories:
  - 强化学习
---

> 本文中的步骤基于Windows11的 Linux 子系统

1. 首先， 不推荐在任何`Windows`系统上安装. 如果是`Windows`的电脑, 一定安装在 `wsl` 子系统上. "Windows 是最好的 Linux 发行版"

2. 安装`mujoco`:

   1. 首先参考这个 `install mujoco` 的[步骤](https://github.com/openai/mujoco-py#install-mujoco), 但是推荐安装

   2. 下载 [activation key](https://www.roboti.us/file/mjkey.txt) 并放置在 `~/.mujoco` 路径下

   3. 在个人文件夹下的 `.bashrc` 文件中添加: 

      ```bash
      export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/换成你自己的用户名/.mujoco/mjpro150/bin
      ```

3. 按顺序运行以下命令:

   ```bash
   sudo apt-get update
    
   sudo apt-get install -y libglu1-mesa-dev libgl1-mesa-dev libosmesa6-dev xvfb ffmpeg curl patchelf libglfw3 libglfw3-dev cmake zlib1g zlib1g-dev swig
   
   pip install gym[all]
   ```

3. 如果在 build wheel for `mujoco-py` 时出错, 参考以下代码:

   ```bash
   git clone https://github.com/openai/mujoco-py.git
   cd mujoco-py
   pip install -e.
   ```

   来源: https://github.com/openai/mujoco-py#install-mujoco

