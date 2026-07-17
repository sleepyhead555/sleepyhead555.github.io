---
title: nnunet实战
date: 2026-07-17 22:20:15
updated: 2026-07-17 22:20:15
tags:
categories:
# 侧边栏
toc:
  post: true
  page: false
  number: true
  expand: false
  # Only for post
  style_simple: false
  scroll_percent: true
# 版权声明
post_copyright:
  enable: true
  decode: false
  author_href: https://github.com/sleepyhead555
#   license: CC BY-NC-SA 4.0
#   license_url: 

# 代码块
code_blocks:
  # Code block theme: darker / pale night / light / ocean / false
  theme: light
  macStyle: false
  # Code block height limit (unit: px)
  height_limit: false
  word_wrap: false

  # Toolbar
  copy: true
  language: true
  # true: shrink the code blocks | false: expand the code blocks | none: expand code blocks and hide the button
  shrink: false
  fullpage: false

---
# nnunet实战（windows）

## 项目介绍
&emsp; nnUNet 是一个基于深度学习的医学图像分割工具，用于自动分割医学图像中的结构。[项目地址](https://github.com/MIC-DKFZ/nnUNet)

## 准备工作
### 下载到本地
&emsp; 终端上git clone 项目 https://github.com/MIC-DKFZ/nnUNet.git
{% codeblock lang:bash %}
git clone https://github.com/MIC-DKFZ/nnUNet.git
{% endcodeblock %}
&emsp; 下载之后就会发现多了nnUNet文件夹。

### 安装nnunet包
&emsp; 创建虚拟环境，安装pytorch等省略
&emsp; 运行setup.py文件（使用pip install -e .命令安装最新版本），安装nnunet包。成功安装即可使用nnunet命令。
{% codeblock lang:bash %}
cd nnUNet
pip install -e .
{% endcodeblock %}
### 数据集准备与环境变量配置
&emsp; 首先回退到nnUNet文件夹的上一级目录，执行以下命令：
{% codeblock lang:bash %}
cd ..
{% endcodeblock %}
&emsp; 新建文件夹nnunetframe。继续新建三个子文件夹：raw存储原始数据集，preprocessed存储预处理后的数据集，nnUNet_output存储处理后的数据集。
&emsp; 配置环境变量，示例如下，根据自己的命名修改：
{% codeblock lang:bash %}
$env:nnUNet_raw = "E:\nnunet\nnunetframe\raw"
$env:nnUNet_preprocessed = "E:\nnunet\nnunetframe\preprocessed"
$env:nnUNet_results = "E:\nnunet\nnunetframe\nnUNet_output" 
{% endcodeblock %}
&emsp; 这样做每次使用都需要终端重新加载环境变量。
&emsp; 数据集下载的话，可以在当时的竞赛页面下载。[竞赛页面](http://medicaldecathlon.com/dataaws/)。我选择的是Task04_Hippocampus数据集。下载后目录如图所示：![Task04_Hippocampus数据集目录](img/nnunet_data.png "Task04_Hippocampus数据集目录")

## 数据处理
### 数据集转换
&emsp; 转换数据集为nnUNet要求的格式，具体转换过程可以看 nnUNet\nnunetv2\dataset_conversion\convert_MSD_dataset.py 文件。
{% codeblock lang:bash %}
nnUNetv2_convert_MSD_dataset -i E:\nnunet\Task04_Hippocampus -o E:\nnunet\nnunetframe\raw
{% endcodeblock %}
&emsp; 转换后出现了Dataset004_Hippocampus

### 数据集预处理
&emsp; 预处理数据集，运行一下命令：
{% codeblock lang:bash %}
nnUNetv2_plan_and_preprocess -d 004 --verify_dataset_integrity
{% endcodeblock %}
值得注意的是，我运行到此处时会报错，提示虚拟存储页面不足。解决方案如下：
①按 Win + R，输入 sysdm.cpl
②选择"高级"选项卡
③点击"性能"区域中的"设置"
④选择"高级"选项卡
⑤点击"虚拟内存"区域的"更改"
⑥取消"自动管理"，选择"自定义大小"
⑦设置 初始大小: （根据自己的内存大小设置）
⑧点击"设置"并确定
⑨重启电脑

## 模型训练、验证、寻优
### 模型训练
&emsp; 三种U-Net网络配置，分别是2D U-Net，3D全分辨率U-Net，3D U-Net级联（包括3D低分辨率U-Net和3D全分辨率U-Net）。目前我只训练了2D U-Net。
&emsp; 默认采用五折交叉验证，即分别训练五次取平均值。训练前我们需要先修改epoch。到nnUNet\nnunetv2\training\nnUNetTrainer\nnUNetTrainer.py 这个文件下寻找一个名为nnUNetTrainer的类，变量num_epochs，初始值为1000。我看的文章推荐是200-500。初始checkpoint更新频率save_every = 50也就是每50个epoch更新一次。根据自己的情况修改。第一次就试试能不能跑通且避免意外停止丢失进度，所以我分别设置200和20，并且没有进行五折交叉验证（只跑了一个fold）
2d的训练指令如下（不过我只跑了一个）：

{% codeblock lang:bash %}
nnUNetv2_train 4 2d 0 
nnUNetv2_train 4 2d 1
nnUNetv2_train 4 2d 2
nnUNetv2_train 4 2d 3
nnUNetv2_train 4 2d 4
{% endcodeblock %}
&emsp;如果中间断了，需要重新运行，从上次更新checkpoint处继续训练，那就输入
{% codeblock lang:bash %}
nnUNetv2_train 4 2d 0 --c
{% endcodeblock %}
&emsp; 具体可以看 nnUNet\nnunetv2\run\run_training.py 文件。
&emsp; 训练过程中会汇报没有安装hiddenlayer，无需在意，无影响，似乎是绘制模型结构的模块；实时保存训练日志、结果。

### 模型验证
&emsp; 训练完成后，模型验证指令如下：
{% codeblock lang:bash %}
nnUNetv2_train 4 2d 0  --val --npz
nnUNetv2_train 4 2d 1  --val --npz
nnUNetv2_train 4 2d 2  --val --npz
nnUNetv2_train 4 2d 3  --val --npz
nnUNetv2_train 4 2d 4  --val --npz
{% endcodeblock %}

### 模型寻优
&emsp; 如果你跑满了5个，那么就可以做这一步：
{% codeblock lang:bash %}
nnUNetv2_find_best_configuration 4 -c 2d 3d_fullres 3d_lowers -f 0 1 2 3 4
{% endcodeblock %}
