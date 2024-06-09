---
title: HBase 部署踩坑
published: 2024-06-09
category: 随笔
tags:
  - Hadoop
  - HBase
---

这两天开始做大数据挖掘实务课程的课设，课程要求使用 HBase 储存爬取的数据，因此今天在折腾 HBase 的部署。

HBase 部署流程和 HDFS 基本差不多，很快就搞定了相关配置，结果却死活启动不起来 HMaster 节点，一直卡在初始化 ServerManager。

找了半天原因，翻日志发现有如下报错：

```
Unexpected error caught when processing netty
java.lang.IllegalArgumentException: object is not an instance of declaring class
...
```

故怀疑是依赖的 jar 包产生冲突，搜索后得知 HBase 和 Hadoop 本身依赖的 jar 包确实会发生冲突，需要在 hbase-env 将以下内容取消注释：

```bash
export HBASE_DISABLE_HADOOP_CLASSPATH_LOOKUP="true"
```

以此在 HBase 启动时屏蔽来自 Hadoop 的 jar 包，之后就启动成功了。

但说实话，我不是很明白这个选项为什么默认是关闭的。正常来说也不会为了节省这几十 MB 的空间共享 jar 包吧？
