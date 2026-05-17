下面这份可以作为你以后分类资料时的**一级目录定义手册**。核心原则保持不变：

```text
/data/uos   = UOS 原生工作盘，偏生产、开发、加工、临时处理
/data/share = Windows/UOS 共享资料盘，偏资料、模板、成品、归档、同步
```

---

# 一、ext4 盘：`/data/uos`

定位：**当前工作区 + 开发区 + 本机高性能处理区**

这里的文件特点是：经常修改、频繁读写、需要 Linux 权限特性、需要编译/索引/缓存、不一定要直接给 Windows 使用。

| 一级目录                      | 定义               | 放什么                                                     | 不建议放什么                    |
| ------------------------- | ---------------- | ------------------------------------------------------- | ------------------------- |
| `00_Inbox_UOS`            | UOS 本机临时收件箱      | 浏览器下载、微信/QQ/邮件临时接收、未分类资料、临时截图                           | 长期资料、正式项目文件、最终交付件         |
| `10_Work_Active`          | 当前正在推进的正式工作项目区   | 正在写的方案、PPT、投标文档、客户项目过程文件、会议纪要、临时版本                      | 已结束项目、通用知识库、软件安装包         |
| `20_Windsurf_Workspace`   | Windsurf 主工作区    | Python/Java/C/C++ 项目、售前自动化工具、PPT 生成工具、脚本、Demo、AI 辅助开发项目 | Obsidian 主仓库、最终交付件、大量历史资料 |
| `30_DevLab`               | 开发实验与依赖缓存区       | Git 仓库实验、Python 虚拟环境、Maven/Gradle 缓存、C/C++ build、测试数据   | 需要 Windows 直接编辑的文档、正式归档文件 |
| `40_Containers_VM`        | 容器和虚拟化环境区        | Docker/Podman 数据、虚拟机镜像、实验服务、PoC 环境                      | 普通文档、PPT、长期知识资料           |
| `50_AI_Knowledge_Working` | AI/知识库加工区        | Prompt、RAG 原始加工文件、Obsidian 副本处理、AI 生成草稿、代码片段            | Obsidian 正式主仓库、最终知识沉淀资料   |
| `60_Design_Working`       | 图形、架构图、CAD 当前工作区 | draw.io 工作文件、架构图草稿、CAD 编辑文件、图片处理素材                      | 已定稿图库、公司标准模板、历史图纸归档       |
| `70_Local_Backup`         | UOS 本机局部备份区      | 配置备份、脚本备份、每日/每周临时备份                                     | 唯一副本、长期唯一归档、跨设备同步主目录      |
| `99_Temp`                 | 临时垃圾场 / 可清理区     | 解压临时文件、编译临时文件、测试输出、一次性草稿                                | 任何不能丢的资料                  |

---

## `/data/uos` 的分类判断规则

### 放入 `/data/uos/00_Inbox_UOS`

当你还不知道资料该归到哪里时，先放这里。

适合：

```text
刚下载的资料
别人临时发来的文件
截图
压缩包
未读文档
临时接收的投标附件
```

处理原则：

```text
每周清理一次
有价值的资料转入 /data/share
正在使用的资料转入 /data/uos/10_Work_Active
无价值的删除
```

---

### 放入 `/data/uos/10_Work_Active`

只放**当前正在做**的正式工作。

适合：

```text
本周正在写的客户方案
正在制作的 PPT
正在准备的投标文件
客户调研记录
方案草稿
报价测算过程文件
项目沟通纪要
```

判断标准：

```text
这个文件是否属于当前正在推进的客户/项目？
是否还会频繁修改？
是否尚未最终交付？
```

是，就放这里。

---

### 放入 `/data/uos/20_Windsurf_Workspace`

只放 Windsurf 主要打开和维护的项目。

适合：

```text
Python 脚本项目
Java Demo
售前工具项目
PPT 自动化生成项目
投标文档处理工具
数据清洗脚本
AI 辅助代码项目
C/C++ 实验项目
```

不建议把 Obsidian 仓库放这里。Obsidian 是知识资产，应该优先放共享盘；Windsurf 是开发生产力工具，项目应放 ext4。

---

### 放入 `/data/uos/30_DevLab`

这里不是正式项目区，而是**开发支撑区**。

适合：

```text
Python 虚拟环境
Maven 本地仓库
Gradle 缓存
CMake build
编译产物
测试数据
临时 Git clone
```

判断标准：

```text
这个东西是不是开发依赖、缓存、编译产物、实验环境？
```

是，就放这里。

---

### 放入 `/data/uos/40_Containers_VM`

只放容器、虚拟机、实验服务。

适合：

```text
Docker 数据
Podman 数据
虚拟机镜像
本地数据库容器
演示环境
PoC 服务
Linux 实验环境
```

这类文件通常体积大、频繁读写，不要放 NTFS。

---

### 放入 `/data/uos/50_AI_Knowledge_Working`

这是 AI 工作流的加工区，不是最终知识库。

适合：

```text
AI 生成的方案草稿
Prompt 模板草稿
RAG 切片前资料
批量清洗后的 Markdown
Obsidian 临时处理副本
向量库实验数据
代码片段整理
```

最终整理好的知识，再转入：

```text
/data/share/10_Knowledge_Base
```

---

### 放入 `/data/uos/60_Design_Working`

只放图形、图纸、架构图的当前编辑版本。

适合：

```text
draw.io 草稿
架构图源文件
网络拓扑图
CAD 工作文件
图片编辑中间文件
方案插图源文件
```

最终版图片、图标库、LOGO、图库，应转入：

```text
/data/share/50_Assets
```

项目交付图纸，应转入：

```text
/data/share/30_Deliverables
```

---

### 放入 `/data/uos/70_Local_Backup`

这是本机辅助备份，不是正式归档。

适合：

```text
UOS 配置备份
Windsurf 配置备份
shell 脚本备份
重要配置文件快照
短周期工作备份
```

不建议把唯一一份重要资料只放这里。

---

### 放入 `/data/uos/99_Temp`

可以随时清空的临时目录。

适合：

```text
临时解压
临时测试
临时转换
临时编译
一次性草稿
下载后准备删除的文件
```

判断标准：

```text
这个文件一个月后删除也不会心疼吗？
```

是，就放这里。

---

# 二、NTFS 盘：`/data/share`

定位：**Windows/UOS 共享盘 + 长期资料资产盘 + 交付归档盘**

这里的文件特点是：Windows 和 UOS 都可能访问，偏长期保存、共享、交付、归档、模板、资料库。

| 一级目录                    | 定义                  | 放什么                                       | 不建议放什么                                     |
| ----------------------- | ------------------- | ----------------------------------------- | ------------------------------------------ |
| `00_Inbox_From_Windows` | Windows 历史资料和临时迁移入口 | 从 Windows 搬过来的桌面文件、下载目录、旧项目、未分类资料         | 整理后的正式资料、长期归档                              |
| `10_Knowledge_Base`     | 长期知识库               | Obsidian 仓库、行业资料、产品资料、厂商资料、技术资料、案例、政策标准   | 当前项目过程文件、临时下载文件                            |
| `20_Project_Shared`     | Windows/UOS 共享项目资料区 | 项目输入资料、客户原始资料、投标原文、跨系统交换文件                | Windsurf 主项目、编译缓存、虚拟环境                     |
| `30_Deliverables`       | 最终交付件区              | PPT、Word、Excel、PDF、投标文件、签署版、提交版           | 未完成草稿、过程版本、临时文件                            |
| `40_Templates`          | 模板库                 | PPT 模板、Word 模板、Excel 模板、投标模板、架构图模板、公司标准格式 | 项目专用过程文件、一次性文件                             |
| `50_Assets`             | 素材资产库               | 图标、图片、LOGO、授权字体、CAD 图块、draw.io 图库         | 临时截图、未授权素材、项目草稿                            |
| `60_Code_Shared`        | 跨系统共享代码区            | 小脚本、代码片段、Demo 导出版、可复用工具                   | 正在高频开发的完整项目、`.venv`、`node_modules`、`build` |
| `70_Sync`               | 同步交换区               | Syncthing 同步目录、云盘同步目录、临时跨设备传输             | 长期归档主目录、开发缓存                               |
| `80_Software_Packages`  | 软件安装包与驱动库           | UOS deb 包、Windows 安装包、驱动、离线工具             | 项目资料、知识库文档                                 |
| `90_Archive`            | 历史归档区               | 已结束项目、年度归档、旧资料、只读最终版                      | 当前正在修改的项目                                  |

---

## `/data/share` 的分类判断规则

### 放入 `/data/share/00_Inbox_From_Windows`

这是 Windows 旧资料的入口，不是长期目录。

适合：

```text
Windows 桌面旧文件
Windows 下载目录
Windows 旧项目目录
移动硬盘复制过来的资料
还没来得及分类的历史资料
```

处理原则：

```text
先搬进来
再慢慢分类
不要在这里长期堆积
```

---

### 放入 `/data/share/10_Knowledge_Base`

这是你的长期知识资产核心目录。

适合：

```text
Obsidian 仓库
行业研究资料
产品白皮书
厂商方案
技术标准
政策法规
客户案例
竞争对手资料
售前方法论
投标经验总结
```

你的 Obsidian 主仓库建议放：

```text
/data/share/10_Knowledge_Base/obsidian_vaults
```

判断标准：

```text
这个资料是否将来多个项目都可能复用？
是否属于知识沉淀？
是否需要 Windows 和 UOS 都能访问？
```

是，就放这里。

---

### 放入 `/data/share/20_Project_Shared`

这是跨 Windows/UOS 使用的项目资料区。

适合：

```text
客户发来的原始资料
招标文件原文
项目输入材料
需要 Windows/UOS 都能访问的项目文件
项目交换文件
项目对外资料
```

它和 `/data/uos/10_Work_Active` 的区别：

```text
/data/uos/10_Work_Active   = 当前工作过程
/data/share/20_Project_Shared = 跨系统共享资料
```

例如：

```text
客户原始招标文件       -> /data/share/20_Project_Shared
你正在编辑的技术方案   -> /data/uos/10_Work_Active
阶段性输出给 Windows   -> /data/share/20_Project_Shared
```

---

### 放入 `/data/share/30_Deliverables`

这里只放最终或准最终交付件。

适合：

```text
正式 PPT
正式 Word 方案
正式 Excel 报价表
正式 PDF
投标提交文件
签署版文件
客户发送版
盖章扫描版
```

判断标准：

```text
这个文件是否已经准备发给客户、领导、招标平台或归档？
```

是，就放这里。

不建议把 `方案_v1`、`方案_v2_临时`、`PPT_改改看` 这种过程文件放这里。

---

### 放入 `/data/share/40_Templates`

这是模板库，只放可复用格式。

适合：

```text
公司 PPT 模板
售前方案模板
投标技术文件模板
报价表模板
会议纪要模板
架构图模板
项目计划模板
WBS 模板
风险清单模板
```

判断标准：

```text
这个文件是不是未来可以反复复制使用？
```

是，就放这里。

---

### 放入 `/data/share/50_Assets`

这是素材库，只放可复用素材。

适合：

```text
产品图标
行业图标
公司 LOGO
客户 LOGO
背景图
标准架构图元素
授权字体
CAD 图块
draw.io 自定义图库
```

注意：

```text
未授权字体和图片不要混入正式素材库
临时截图不要放这里
项目专用素材可以先放项目目录，复用价值高再沉淀到这里
```

---

### 放入 `/data/share/60_Code_Shared`

这是跨系统共享代码区，不是主开发区。

适合：

```text
小脚本
可复用代码片段
已经打包或导出的 Demo
不需要频繁编译的轻量代码
Windows/UOS 都要参考的脚本
```

不建议放：

```text
大型 Git 项目
Windsurf 正在开发的项目
Python .venv
Java target
C++ build
node_modules
Docker 数据
```

主开发仍然放：

```text
/data/uos/20_Windsurf_Workspace
```

---

### 放入 `/data/share/70_Sync`

这是同步和传输区。

适合：

```text
Syncthing 同步目录
云盘同步目录
临时给另一台电脑的文件
跨设备交换包
手机/平板传输文件
```

判断标准：

```text
这个文件是否主要用于设备之间传输或同步？
```

是，就放这里。

不要把整个 `/data/uos` 或整个 `/data/share` 都同步。优先同步必要目录。

---

### 放入 `/data/share/80_Software_Packages`

这是软件安装包仓库。

适合：

```text
UOS deb 安装包
Windows exe/msi 安装包
驱动程序
离线工具
浏览器安装包
WPS 安装包
Windsurf 安装包
CAD 工具安装包
常用绿色工具
```

建议分类保存，方便重装系统时快速恢复。

---

### 放入 `/data/share/90_Archive`

这是历史归档区。

适合：

```text
已结束项目
年度备份
旧资料
不再频繁修改但需要保留的文件
最终只读版
历史投标文件
历史方案成果
```

判断标准：

```text
这个项目是否已经结束？
这个资料是否短期不会再修改？
这个文件是否只需要以后查询？
```

是，就放这里。

---

# 三、快速分类决策表

以后遇到文件不知道放哪里，可以按下面判断：

| 文件类型              | 推荐目录                                                 |
| ----------------- | ---------------------------------------------------- |
| UOS 刚下载的文件        | `/data/uos/00_Inbox_UOS`                             |
| Windows 搬来的旧资料    | `/data/share/00_Inbox_From_Windows`                  |
| 当前正在写的方案          | `/data/uos/10_Work_Active`                           |
| 当前正在做的 PPT        | `/data/uos/10_Work_Active`                           |
| 当前正在准备的投标文件       | `/data/uos/10_Work_Active`                           |
| 客户原始资料            | `/data/share/20_Project_Shared`                      |
| 招标文件原文            | `/data/share/20_Project_Shared`                      |
| 正式交付 PPT          | `/data/share/30_Deliverables/ppt`                    |
| 正式 PDF            | `/data/share/30_Deliverables/pdf`                    |
| Obsidian 仓库       | `/data/share/10_Knowledge_Base/obsidian_vaults`      |
| 行业资料              | `/data/share/10_Knowledge_Base/industry`             |
| 产品资料              | `/data/share/10_Knowledge_Base/products`             |
| 厂商资料              | `/data/share/10_Knowledge_Base/vendors`              |
| 政策标准              | `/data/share/10_Knowledge_Base/policies_standards`   |
| Windsurf 项目       | `/data/uos/20_Windsurf_Workspace`                    |
| Python 虚拟环境       | `/data/uos/30_DevLab/python_envs` 或项目 `.venv`        |
| Java Maven 缓存     | `/data/uos/30_DevLab/java_maven`                     |
| C/C++ 编译输出        | `/data/uos/30_DevLab/cpp_build`                      |
| Docker/Podman 数据  | `/data/uos/40_Containers_VM`                         |
| 虚拟机镜像             | `/data/uos/40_Containers_VM/vm_images`               |
| AI 生成草稿           | `/data/uos/50_AI_Knowledge_Working/generated_drafts` |
| Prompt            | `/data/uos/50_AI_Knowledge_Working/prompts`          |
| draw.io 草稿        | `/data/uos/60_Design_Working/drawio`                 |
| 图标/LOGO           | `/data/share/50_Assets`                              |
| PPT/Word/Excel 模板 | `/data/share/40_Templates`                           |
| 小脚本共享             | `/data/share/60_Code_Shared/scripts`                 |
| 软件安装包             | `/data/share/80_Software_Packages`                   |
| 已结束项目             | `/data/share/90_Archive/projects_closed`             |
| 临时解压文件            | `/data/uos/99_Temp/unzip_tmp`                        |

---

# 四、最重要的分类口诀

```text
正在干活放 ext4，长期沉淀放 NTFS。
频繁修改放 ext4，跨系统共享放 NTFS。
代码开发放 ext4，代码成果可复制到 NTFS。
Obsidian 主库放 NTFS，AI 加工副本放 ext4。
PPT 草稿放 ext4，最终交付放 NTFS。
客户原始资料放 NTFS，分析过程放 ext4。
容器虚拟机放 ext4，安装包归档放 NTFS。
临时文件进 99_Temp，旧资料先进 Inbox。
```

---

# 五、建议放一个说明文件

你可以把下面内容保存成：

```text
/data/uos/README_目录分类说明.txt
/data/share/README_目录分类说明.txt
```

更建议统一保存到：

```text
/data/share/10_Knowledge_Base/obsidian_vaults/presale_knowledge/60_模板/目录分类规则.md
```

这样以后在 Obsidian 里也能查。
