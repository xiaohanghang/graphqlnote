# 基于devrest的graphql扩张与实现

**方案思路：**

     为在devrest做二次开发、并保持devrest功能正常运行的基础上，考虑复用sqlalchemy的model，并在此基础上进行扩张，在数据库orm层的基础上进行封装，将多种数据库模型类转化为graphql类型，并提供多种数据库的相互转换以及用户自定义，之后再通过转化的graphql类型转化为schema所需要query，

**功能点：**

* 支持复用sqlalchemy定义的模型
* 支持原devrest分页、过滤、搜索、子关系查询、排序、黑白名单等功能。
* graphql支持多数据源聚合\(mysql,mongo,Rest APi、ELK\)等功能。



       







