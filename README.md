# 基于devrest的graphql扩张与实现

### **方案思路：**

> 为在devrest做二次开发、并保持devrest功能正常运行的基础上，考虑复用sqlalchemy的model，并在此基础上进行扩张，在数据库orm层的基础上进行封装，将多种数据库模型类转化为graphql类型，并提供多种数据库的相互转换以及用户自定义\(兼具便利性以及灵活性\)，之后再为通过转化的graphql类型定义schema所需要query。并在此基础上丰富查询接口参数\\(从用户的角度看：最好和devrest的查询方式保持一致\\)，根据查询的方式分为：1.单向查询\(只需把model参数映射到graphql的自省模块中\)2.列表查询\(处理外键的关系以及Field与node相互结合的方式\)，定义三种resolve处理函数，当请求经过flask API后，便是通过在view层定义的excutor来找到所定义的resolve函数并进行数据处理。并把数据返回。

### 基本功能：

* **支持复用sqlalchemy模型，并提供mongo模型API**

* **支持原devrest分页、过滤、搜索、子关系查询、排序、黑白名单等功能**

* **graphql支持多数据源聚合\(mysql,mongo,Rest APi、ELK\)等功能**

* **简单易用，使用理念与devrest保持一致，并提供用户自定义**

### 下载使用：

###### 1. 下载所需安装包

```py
pip install -r requirements.txt
```

###### 2. 定义sqlalchemy \| mongoengine的model，再使用本组件中create\_query\_schema函数，将model转化的graphql传递进去，绑定路由到app，即可实现graphql灵活的查询功能。

###### 3.  若需要调用第三方公共接口或者多种数据库中的聚合，在model层定义auto\__add_\_column函数，并定义需要添加的columns，实现数据聚合。

###### 4.运行app

```py
> python app.py
Running on http://127.0.0.1:5000/
```

###### 5.访问 [http://127.0.0.1:5000/graphql](http://127.0.0.1:5000/graphql)

### 版本支持：

目前开发版本只支持graphene==1.4

```
graphene == 1.4
```

## 接口参数请求：

### Example：

1. **model定义：用例涵盖模型之间多种关联关系**

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
class Department(Base):
    __tablename__ = 'department'
    id = Column(Integer, primary_key=True)
    name = Column(String)
class Role(Base):
    class Meta:
        auto_add_column = ["github", "github_api_status_code"]
    __tablename__ = 'roles'
    role_id = Column(Integer, primary_key=True)
    name = Column(String)
    def auto_add_columns(self, *args, **kwargs):
        import requests
        result = OrderedDict()
        mapper = sqlalchemyinspect(self).mapper
        columns = sqlalchemyinspect(self).mapper.columns
        for name, column_name in columns.items():
            value = getattr(self, name, None)
            result[to_snake_case(name)] = value
        if result.get('name'):
            github_res = requests.get(
                'https://api.github.com/users/%s' % result['name']
            )
            result['github'] = github_res.json()
            result['github_api_status_code'] = github_res.status_code
        return result
class Employee(Base):
    __tablename__ = 'employee'
    __match_fields__ = ['name']
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

**2.model转化为graphql类型**

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

**3.schema生成**

```py
schema = graphene.Schema(query=create_query_schema(attrs), mutation=MyMutations, types=[Users,Departments,Employees,Roles])
```

**效果显示：**

**1.列表分页查询：\(manytoono or onetoone\)**

```
{
  connectionAllEmployee(Num:1,Page:1){
    edges{
      node{
        id
        name
        hiredOn
        department{
          id
          name
        }
        role{
          id
          name
        }
      }
    }
  }
}
```

效果：

```
{
  "data": {
    "connectionAllEmployee": {
      "edges": [
        {
          "node": {
            "id": "1",
            "name": "Peter",
            "hiredOn": "2018-07-01T13:09:19",
            "department": {
              "id": "1",
              "name": "Engineering"
            },
            "role": {
              "id": "Um9sZTpOb25l",
              "name": "engineer"
            }
          }
        }
      ]
    }
  }
}
```

onetomany：

```
{
  connectionAllPost{
    edges{
      node{
        id
        tag
        body
        comments{
          id
          name
          body
        }
      }
    }
  }
}
```

效果：

```
{
  "data": {
    "connectionAllPost": {
      "edges": [
        {
          "node": {
            "id": "1",
            "tag": "hello python",
            "body": "sounds very well",
            "comments": [
              {
                "id": "1",
                "name": "zhangsan",
                "body": "yep you are right"
              },
              {
                "id": "2",
                "name": "lisi",
                "body": "yep you are wrong"
              }
            ]
          }
        }
      ]
    }
  }
}
```

过滤：

```
{
  connectionAllEmployee(name:"Peter"){
    edges{
      node
      {
        id
        name
        hiredOn
        department{
          id
          name
        }
        role{
          roleId
          name
        }
      }
    }
  }
}
```

效果：

```py
{
  "data": {
    "connectionAllEmployee": {
      "edges": [
        {
          "node": {
            "id": "1",
            "name": "Peter",
            "hiredOn": "2018-07-01T13:59:15",
            "department": {
              "id": "1",
              "name": "Engineering"
            },
            "role": {
              "roleId": "2",
              "name": "engineer"
            }
          }
        }
      ]
    }
  }
}
```

排序+分页：

```
{
  connectionAllEmployee(Direction:"desc",Sort:"name",Num:2,Page:1){
    edges{
      node
      {
        id
        name
        hiredOn
        department{
          id
          name
        }
        role{
          roleId
          name
        }
      }
    }
  }
}
```

效果：

```
{
  "data": {
    "connectionAllEmployee": {
      "edges": [
        {
          "node": {
            "id": "3",
            "name": "Tracy",
            "hiredOn": "2018-07-01T14:39:40",
            "department": {
              "id": "2",
              "name": "Human Resources"
            },
            "role": {
              "roleId": "1",
              "name": "manager"
            }
          }
        },
        {
          "node": {
            "id": "2",
            "name": "Roy",
            "hiredOn": "2018-07-01T14:39:40",
            "department": {
              "id": "1",
              "name": "Engineering"
            },
            "role": {
              "roleId": "2",
              "name": "engineer"
            }
          }
        }
      ]
    }
  }
}
```

模糊匹配：

```
{
  connectionAllRole(MatchField:"name",Match:"mana"){
    edges{
      node{
        roleId
        name
      }
    }
  }
}
```

效果：

```
{
  "data": {
    "connectionAllRole": {
      "edges": [
        {
          "node": {
            "roleId": "1",
            "name": "manager"
          }
        }
      ]
    }
  }
}
```

in过滤：

```
{
  connectionAllEmployee(ArrayField:"{\"id\":\"[1,2]\"}"){
    edges{
      node
      {
        id
        name
        hiredOn
        department{
          id
          name
        }
        role{
          roleId
          name
        }
      }
    }
  }
}
```

效果：

```
{
  "data": {
    "connectionAllEmployee": {
      "edges": [
        {
          "node": {
            "id": "1",
            "name": "Peter",
            "hiredOn": "2018-07-01T14:54:33",
            "department": {
              "id": "1",
              "name": "Engineering"
            },
            "role": {
              "roleId": "2",
              "name": "engineer"
            }
          }
        },
        {
          "node": {
            "id": "2",
            "name": "Roy",
            "hiredOn": "2018-07-01T14:54:33",
            "department": {
              "id": "1",
              "name": "Engineering"
            },
            "role": {
              "roleId": "2",
              "name": "engineer"
            }
          }
        }
      ]
    }
  }
}
```

外键过滤：

```
{
  connectionAllEmployee(EqualField:"{\"role.name\":\"engineer\"}"){
    edges{
      node
      {
        id
        name
        hiredOn
        department{
          id
          name
        }
        role{
          roleId
          name
        }
      }
    }
  }
}
```

```
{
  "data": {
    "connectionAllEmployee": {
      "edges": [
        {
          "node": {
            "id": "1",
            "name": "Peter",
            "hiredOn": "2018-07-01T14:54:33",
            "department": {
              "id": "1",
              "name": "Engineering"
            },
            "role": {
              "roleId": "2",
              "name": "engineer"
            }
          }
        },
        {
          "node": {
            "id": "2",
            "name": "Roy",
            "hiredOn": "2018-07-01T14:54:33",
            "department": {
              "id": "1",
              "name": "Engineering"
            },
            "role": {
              "roleId": "2",
              "name": "engineer"
            }
          }
        }
      ]
    }
  }
}
```

**2.单项查询：**

```
{
  findEmployee(PrimaryId:1){
    id
    name
  }
}
```

```
{
  "data": {
    "findEmployee": {
      "id": "1",
      "name": "Peter"
    }
  }
}
```

**3.REST API动态显示调用：**

* model定义：

```py
class Role(Base):
    class Meta:
        auto_add_column = ["github", "github_api_status_code"]
    __tablename__ = 'roles'
    role_id = Column(Integer, primary_key=True)
    name = Column(String)
    def auto_add_columns(self, *args, **kwargs):
        import requests
        result = OrderedDict()
        mapper = sqlalchemyinspect(self).mapper
        columns = sqlalchemyinspect(self).mapper.columns
        for name, column_name in columns.items():
            value = getattr(self, name, None)
            result[to_snake_case(name)] = value
        if result.get('name'):
            github_res = requests.get(
                'https://api.github.com/users/%s' % result['name']
            )
            result['github'] = github_res.json()
            result['github_api_status_code'] = github_res.status_code
        return result
```

上述代码中定义的auto\_add\_column为新增的Rest api列，然后再定义auto\_\_add\_\_columns方法，实现rest API的动态列添加。

单项查询：\(通过name去请求第三方公共接口\)

```
{
  findRole(name:"manager"){
    roleId
    name
    github
    githubApiStatusCode
  }
}
```

效果：

```
{
  "data": {
    "findRole": {
      "roleId": "1",
      "name": "manager",
      "github": "{u'public_repos': 0, u'site_admin': False, u'subscriptions_url': u'https://api.github.com/users/manager/subscriptions', u'gravatar_id': u'', u'hireable': None, u'id': 395360, u'followers_url': u'https://api.github.com/users/manager/followers', u'following_url': u'https://api.github.com/users/manager/following{/other_user}', u'blog': u'', u'followers': 2, u'location': None, u'type': u'User', u'email': None, u'bio': None, u'gists_url': u'https://api.github.com/users/manager/gists{/gist_id}', u'company': None, u'events_url': u'https://api.github.com/users/manager/events{/privacy}', u'html_url': u'https://github.com/manager', u'updated_at': u'2014-01-13T13:46:08Z', u'node_id': u'MDQ6VXNlcjM5NTM2MA==', u'received_events_url': u'https://api.github.com/users/manager/received_events', u'starred_url': u'https://api.github.com/users/manager/starred{/owner}{/repo}', u'public_gists': 0, u'name': None, u'organizations_url': u'https://api.github.com/users/manager/orgs', u'url': u'https://api.github.com/users/manager', u'created_at': u'2010-09-11T06:34:52Z', u'avatar_url': u'https://avatars3.githubusercontent.com/u/395360?v=4', u'repos_url': u'https://api.github.com/users/manager/repos', u'following': 0, u'login': u'manager'}",
      "githubApiStatusCode": "200"
    }
  }
}
```

列表查询\(list\)：

```
{
  allRole{
    roleId
    name
    github
    githubApiStatusCode
  }
}
```

显示效果：

```py
{
  "data": {
    "allRole": [
      {
        "roleId": "1",
        "name": "manager",
        "github": "{u'public_repos': 0, u'site_admin': False, u'subscriptions_url': u'https://api.github.com/users/manager/subscriptions', u'gravatar_id': u'', u'hireable': None, u'id': 395360, u'followers_url': u'https://api.github.com/users/manager/followers', u'following_url': u'https://api.github.com/users/manager/following{/other_user}', u'blog': u'', u'followers': 2, u'location': None, u'type': u'User', u'email': None, u'bio': None, u'gists_url': u'https://api.github.com/users/manager/gists{/gist_id}', u'company': None, u'events_url': u'https://api.github.com/users/manager/events{/privacy}', u'html_url': u'https://github.com/manager', u'updated_at': u'2014-01-13T13:46:08Z', u'node_id': u'MDQ6VXNlcjM5NTM2MA==', u'received_events_url': u'https://api.github.com/users/manager/received_events', u'starred_url': u'https://api.github.com/users/manager/starred{/owner}{/repo}', u'public_gists': 0, u'name': None, u'organizations_url': u'https://api.github.com/users/manager/orgs', u'url': u'https://api.github.com/users/manager', u'created_at': u'2010-09-11T06:34:52Z', u'avatar_url': u'https://avatars3.githubusercontent.com/u/395360?v=4', u'repos_url': u'https://api.github.com/users/manager/repos', u'following': 0, u'login': u'manager'}",
        "githubApiStatusCode": "200"
      },
      {
        "roleId": "2",
        "name": "engineer",
        "github": "{u'public_repos': 9, u'site_admin': False, u'subscriptions_url': u'https://api.github.com/users/engineer/subscriptions', u'gravatar_id': u'', u'hireable': True, u'id': 997668, u'followers_url': u'https://api.github.com/users/engineer/followers', u'following_url': u'https://api.github.com/users/engineer/following{/other_user}', u'blog': u'https://medium.com/@engineer', u'followers': 15, u'location': u'M\\xe9xico', u'type': u'User', u'email': None, u'bio': u'(\\u033f\\u2580\\u033f\\u2009\\u033f\\u0139\\u032f\\u033f\\u033f\\u2580\\u033f \\u033f)\\u0304', u'gists_url': u'https://api.github.com/users/engineer/gists{/gist_id}', u'company': u'M\\xe9xico Conectado', u'events_url': u'https://api.github.com/users/engineer/events{/privacy}', u'html_url': u'https://github.com/engineer', u'updated_at': u'2018-06-05T18:24:08Z', u'node_id': u'MDQ6VXNlcjk5NzY2OA==', u'received_events_url': u'https://api.github.com/users/engineer/received_events', u'starred_url': u'https://api.github.com/users/engineer/starred{/owner}{/repo}', u'public_gists': 0, u'name': u'Pablo Casanova', u'organizations_url': u'https://api.github.com/users/engineer/orgs', u'url': u'https://api.github.com/users/engineer', u'created_at': u'2011-08-23T00:47:18Z', u'avatar_url': u'https://avatars1.githubusercontent.com/u/997668?v=4', u'repos_url': u'https://api.github.com/users/engineer/repos', u'following': 62, u'login': u'engineer'}",
        "githubApiStatusCode": "200"
      }
    ]
  }
}
```

connection+node分页查询：

```
{
  connectionAllRole{
    edges{
      node{
        roleId
        name
        github
        githubApiStatusCode
      }
    }
  }
}
```

效果显示：

```
{
  "data": {
    "connectionAllRole": {
      "edges": [
        {
          "node": {
            "roleId": "1",
            "name": "manager",
            "github": "{u'public_repos': 0, u'site_admin': False, u'subscriptions_url': u'https://api.github.com/users/manager/subscriptions', u'gravatar_id': u'', u'hireable': None, u'id': 395360, u'followers_url': u'https://api.github.com/users/manager/followers', u'following_url': u'https://api.github.com/users/manager/following{/other_user}', u'blog': u'', u'followers': 2, u'location': None, u'type': u'User', u'email': None, u'bio': None, u'gists_url': u'https://api.github.com/users/manager/gists{/gist_id}', u'company': None, u'events_url': u'https://api.github.com/users/manager/events{/privacy}', u'html_url': u'https://github.com/manager', u'updated_at': u'2014-01-13T13:46:08Z', u'node_id': u'MDQ6VXNlcjM5NTM2MA==', u'received_events_url': u'https://api.github.com/users/manager/received_events', u'starred_url': u'https://api.github.com/users/manager/starred{/owner}{/repo}', u'public_gists': 0, u'name': None, u'organizations_url': u'https://api.github.com/users/manager/orgs', u'url': u'https://api.github.com/users/manager', u'created_at': u'2010-09-11T06:34:52Z', u'avatar_url': u'https://avatars3.githubusercontent.com/u/395360?v=4', u'repos_url': u'https://api.github.com/users/manager/repos', u'following': 0, u'login': u'manager'}",
            "githubApiStatusCode": "200"
          }
        },
        {
          "node": {
            "roleId": "2",
            "name": "engineer",
            "github": "{u'public_repos': 9, u'site_admin': False, u'subscriptions_url': u'https://api.github.com/users/engineer/subscriptions', u'gravatar_id': u'', u'hireable': True, u'id': 997668, u'followers_url': u'https://api.github.com/users/engineer/followers', u'following_url': u'https://api.github.com/users/engineer/following{/other_user}', u'blog': u'https://medium.com/@engineer', u'followers': 15, u'location': u'M\\xe9xico', u'type': u'User', u'email': None, u'bio': u'(\\u033f\\u2580\\u033f\\u2009\\u033f\\u0139\\u032f\\u033f\\u033f\\u2580\\u033f \\u033f)\\u0304', u'gists_url': u'https://api.github.com/users/engineer/gists{/gist_id}', u'company': u'M\\xe9xico Conectado', u'events_url': u'https://api.github.com/users/engineer/events{/privacy}', u'html_url': u'https://github.com/engineer', u'updated_at': u'2018-06-05T18:24:08Z', u'node_id': u'MDQ6VXNlcjk5NzY2OA==', u'received_events_url': u'https://api.github.com/users/engineer/received_events', u'starred_url': u'https://api.github.com/users/engineer/starred{/owner}{/repo}', u'public_gists': 0, u'name': u'Pablo Casanova', u'organizations_url': u'https://api.github.com/users/engineer/orgs', u'url': u'https://api.github.com/users/engineer', u'created_at': u'2011-08-23T00:47:18Z', u'avatar_url': u'https://avatars1.githubusercontent.com/u/997668?v=4', u'repos_url': u'https://api.github.com/users/engineer/repos', u'following': 62, u'login': u'engineer'}",
            "githubApiStatusCode": "200"
          }
        }
      ]
    }
  }
}
```



