# 基于devrest的graphql扩张与实现

**方案思路：**

为在devrest做二次开发、并保持devrest功能正常运行的基础上，考虑复用sqlalchemy的model，并在此基础上进行扩张，在数据库orm层的基础上进行封装，将多种数据库模型类转化为graphql类型，并提供多种数据库的相互转换以及用户自定义\\(兼具便利性以及灵活性\\)，之后再为通过转化的graphql类型定义schema所需要query。并在此基础上丰富查询接口参数\\(从用户的角度看：最好和devrest的查询方式保持一致\)，根据查询的方式分为：1.单向查询\(只需把model参数映射到graphql的自省模块中\\)2.列表查询\(由于\)

```
功能点：
```

* 支持复用sqlalchemy定义的模型
* 支持原devrest分页、过滤、搜索、子关系查询、排序、黑白名单等功能。
* graphql支持多数据源聚合\(mysql,mongo,Rest APi、ELK\)等功能。



