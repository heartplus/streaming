# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

1. 一个基于brpc/braft/本地文件系统的stream存储系统
2. 提供基本的open, append, random read 接口
3. 使用modern c++ 进行编程
4. braft/rocksdb用于metaserver中的元数据的存储. stream的基础元信息，可以存放在metaserver中
5. 每个stream由支持单写多读，单写由用户逻辑保证
6. stream的实际数据，由具体的server 进行SSD上的存储
7. 系统的写可用性指标不低于99.99995%
8. 写入IO 路径，采用quorum based
7. metaserver的存储能力，保留扩展到分布式KV系统中的能力


## 修改规范

- build with cmake
- make sure compile pass
- no simulation
- 编码规范参考google c++ 编码规范，但是函数命名采用snake_case
- 以静态连接的方式，连接三方库
- 三方库，尽量使用系统目录下已经安装的库，避免从外部下载
- 单个函数，行数尽量控制在100行以内
- lamda 函数不超过10行
- 避免使用 std::pair, 按照需要定义结构体
- 尽量使用using 来创建别名，减少模板特化导致的代码体积膨胀
- 每个rpc服务，结束的时候，有一行info日志，日志的打印比例，采用百分比的方式，默认每条都打印，但是支持到1/10000的打印比例

## Common Development Considerations
