# 基于devrest的graphql扩张与实现

### **方案思路：**

> 为在devrest做二次开发、并保持devrest功能正常运行的基础上，考虑复用sqlalchemy的model，并在此基础上进行扩张，在数据库orm层的基础上进行封装，将多种数据库模型类转化为graphql类型，并提供多种数据库的相互转换以及用户自定义\\(兼具便利性以及灵活性\\)，之后再为通过转化的graphql类型定义schema所需要query。并在此基础上丰富查询接口参数\\(从用户的角度看：最好和devrest的查询方式保持一致\\)，根据查询的方式分为：1.单向查询\\(只需把model参数映射到graphql的自省模块中\\)2.列表查询\\(处理外键的关系以及Field与node相互结合的方式\\)，定义三种resolve处理函数，当请求经过flask API后，便是通过在view层定义的excutor来找到所定义的resolve函数并进行数据处理。并把数据返回。

### 基本功能：

* 支持复用sqlalchemy模型

* 支持原devrest分页、过滤、搜索、子关系查询、排序、黑白名单等功能

* graphql支持多数据源聚合\(mysql,mongo,Rest APi、ELK\)等功能

### 

## 接口参数请求：

### Example：

1. **model定义：**

```py
class Post(Base):
    __tablename__ = "posts"
    id = Column(Integer,primary_key=True)
    tag = Column(String(255))
    body = Column(Text)
    comments = relationship('Comment')
class Comment(Base):
    __tablename__ = "comments"
    id = Column(Integer, primary_key=True)
    name = Column(String(255))
    body = Column(Text)
    post_id = Column(String(20), ForeignKey("posts.id"))
class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    name = Column(Text)
    email = Column(Text)
    username = Column(String(255))
class Department(Base):
    __tablename__ = 'department'
    id = Column(Integer, primary_key=True)
    name = Column(String)
class Role(Base):
    __tablename__ = 'roles'
    role_id = Column(Integer, primary_key=True)
    name = Column(String)
class Employee(Base):
    __tablename__ = 'employee'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    # Use default=func.now() to set the default hiring time
    # of an Employee to be the current time when an
    # Employee record was created
    hired_on = Column(DateTime, default=func.now())
    department_id = Column(Integer, ForeignKey('department.id'))
    role_id = Column(Integer, ForeignKey('roles.role_id'))
    # Use cascade='delete,all' to propagate the deletion of a Department onto its Employees
    department = relationship(
        Department,
        backref=backref('employees',
                        uselist=True,
                        cascade='delete,all'))
    role = relationship(
        Role,
        backref=backref('roles',
                        uselist=True,
                        cascade='delete,all'))
```

   2.**graphql类型转化：**

```py
Users = create_node_class_gql_object(UserModel)
Departments = create_node_class_gql_object(DepartmentModel)
Roles = create_node_class_gql_object(RoleModel)
Employees = create_node_class_gql_object(EmployeeModel)

Posts = create_node_class_gql_object(PostModel)
Comments = create_node_class_gql_object(CommentModel)
append_graphql_relationhip(Posts)
append_graphql_relationhip(Comments)
```





