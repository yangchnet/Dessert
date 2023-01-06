---
author: "李昌"
title: "数据库版本管理-migrate"
date: "2021-08-15"
tags: ["migrate", "数据库"]
categories: ["golang"]
ShowToc: true
TocOpen: true
---


> migrate是一个golang写成的数据库版本迁移工具，可以用来方便的对数据库进行迁移和回退。
> Github上有详细的教程等：https://github.com/golang-migrate/migrate

> 在安装时可能需要指定所用的驱动，否则会因缺少驱动无法连接数据库，例如`go install -tags mysql github.com/golang-migrate/migrate/v4/cmd/migrate`

1. 建立目录
```bash
mkdir -p migrate-demo/db

cd migrate-demo/db

mkdir ddl 
mkdir -p schema/blog
```

现在`migrate-demo`目录下结构如下：
```
.
└── db
    ├── ddl
    └── schema
        └── blog

```
其中，ddl中存储建库的sql文件，schema存放建表的sql文件

2. 建库

建库
```bash
vim db/ddl/blog.sql
```

```sql
CREATE DATABASE IF NOT EXISTS blog DEFAULT CHARACTER SET utf8mb4 DEFAULT COLLATE utf8mb4_unicode_ci;
```

3. build镜像

编写Dockerfile

```bash
vim db/Dockerfile
```

```dockerfile
FROM mysql:5.7

COPY ./ddl /docker-entrypoint-initdb.d/
ENV MYSQL_ROOT_PASSWORD=admin123
```
复制到`/docker-entrypoint-initdb.d`目录下的sql脚本会被自动执行

```bash
docker build -t mysql-demo -f ./Dockerfile . 
```

build成功后，使用`docker images`命令查看镜像：
```
REPOSITORY                              TAG                                         IMAGE ID       CREATED          SIZE
mysql-demo                              latest                                      6a2faae69a6f   26 minutes ago   447MB
```

4. 启动镜像并查看

```bash
docker run --name mysql -p 13306:3306 -d mysql-demo
```

进入容器查看数据库
```bash
docker exec -it mysql /bin/sh 
```

![20210815223619](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20210815223619.png)

可以看到，blog表已被创建

5. 使用migrate进行数据库迁移

生成sql脚本文件
```
migrate create -ext sql -dir db/blog -seq create_blog_table
```

会在`db/blog`目录下生成`000001_create_blog_table.down.sql`, `000001_create_blog_table.up.sql`，其后缀名分别为`up.sql`, `down.sql`。

`up.sql`文件是你要执行的操作，`down.sql`中是这个操作的逆操作。

建表
```bash
vim schema/blog/000001_create_blog_table.up.sql.sql
```

```sql
CREATE TABLE `blog_article` (
  `id` integer PRIMARY KEY NOT NULL,
  `tag_id` integer DEFAULT 0,
  `title` varchar(100) DEFAULT null,
  `desc` varchar(255) DEFAULT null,
  `content` text,
  `created_on` timestamp DEFAULT CURRENT_TIMESTAMP,
  `created_by` varchar(100) DEFAULT null,
  `modified_on` timestamp DEFAULT CURRENT_TIMESTAMP,
  `modified_by` varchar(100) DEFAULT null,
  `deleted_on` timestamp DEFAULT CURRENT_TIMESTAMP,
);
```


```bash
vim schema/blog/000001_create_blog_table.down.sql.sql
```

```sql
DROP table IF EXISTS blog_article;
```

现在的目录结构：
```
.
└── db
    ├── ddl
    │   └── blog.sql
    ├── Dockerfile
    └── schema
        └── blog
            ├── 000001_create_blog_table.down.sql
            └── 000001_create_blog_table.up.sql
```

6. migrate
执行数据库迁移
```bash
migrate -source=file://db/schema/blog -database "mysql://root:admin123@tcp(localhost:13306)/blog" -verbose up
```
![20210815225144](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20210815225144.png)

再次进入容器查看数据库
![20210815225508](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20210815225508.png)








