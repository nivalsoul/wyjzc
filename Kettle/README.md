# 简介
Kettle 是一款国外开源的 ETL 工具，纯 Java 编写，绿色无需安装，数据抽取高效稳定(数据迁移工具)。Kettle 中有两种脚本文件，transformation 和 job，transformation 完成针对数据的基础转换，job 则完成整个工作流的控制。

Kettle 中文名称叫水壶，该项目的主程序员MATT 希望把各种数据放到一个壶里，然后以一种指定的格式流出。

Kettle这个ETL工具集，它允许你管理来自不同数据库的数据，通过提供一个图形化的用户环境来描述你想做什么，而不是你想怎么做。

Kettle家族目前包括4个产品：Spoon、Pan、CHEF、Kitchen：
- SPOON 允许你通过图形界面来设计ETL转换过程（Transformation）。 
- PAN 允许你批量运行由Spoon设计的ETL转换 (例如使用一个时间调度器)。Pan是一个后台执行的程序，没有图形界面。 
- CHEF 允许你创建任务（Job）。 任务通过允许每个转换，任务，脚本等等，更有利于自动化更新数据仓库的复杂工作。任务通过允许每个转换，任务，脚本等等。任务将会被检查，看看是否正确地运行了。 
- KITCHEN 允许你批量使用由Chef设计的任务 (例如使用一个时间调度器)。KITCHEN也是一个后台运行的程序。

Kettle官网：[Kettle官网](https://community.hitachivantara.com/s/)  
Kettle Github地址：[Kettle Github地址](https://github.com/pentaho/pentaho-kettle/)

# 快速开始
- 安装jdk
- 安装kettle
- 启动

# 组件使用
## 转换
### 输入类
。。。

### 转换类
。。。

### 输出类
。。。

## job
### 流程设计
。。。

### 常用组件
