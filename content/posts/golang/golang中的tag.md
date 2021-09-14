---
author: "李昌"
title: "golang中的tag"
date: "2021-09-14"
tags: ["golang", "reflect"]
categories: ["golang"]
ShowToc: true
TocOpen: true
---

## 1. tag的基本介绍

字段标签可以存储元信息，这些元信息可以使用反射来访问。通常这些元信息用来提供一个字段如何从一种格式编码至另一种格式的相关信息（或是数据应如何在数据库中存储等）。但实际上标签可以存储任何你想要的元信息，无论是你自己使用还是由另一个包使用。

就像[reflect.StructTag](https://pkg.go.dev/reflect#StructTag)文档中提到的那样，字段标签通常是由空格分割的`key:"value"`列表，例如：
```go
type User struct {
    Name string `json:"name" xml:"name"`
}
```

其中的`key`通常表示后面`"value"`所对应的包，例如`json`这个key将被`encoding/json`这个包使用。

如果需要在`"value"`中传递多个值，那么通常使用`,`逗号来分割，例如：
```go
Name string `json:"name,omitempty" xml:"name"`
```

值为破折号通常代表在处理时忽略该字段，例如在`json`中代表不要序列化这个字段

## 2. 例子：获取自定义tag

我们可以使用反射包来获取结构体字段的值。首先我们需要获取结构体的`Type`，然后查询字段，可以使用`Type.Field(i int)`或者`Type.FieldByName(name string)`。这些方法返回一个代表结构体字段的`StructField`值和一个代表tag的类型为`StructTag`的`StructField.Tag`值。

前面我们提到，*字段标签通常是由空格分割的`key:"value"`列表*，如果你的确是这么做的，你可以使用`StructTag.Get(key string)`这个方法来获取这个key对应的value。如果你不是这么做的，`Get()`方法可能不能解析`key:"value"`对并找到你想要的标签。如果你没有遵循*字段标签通常是由空格分割的`key:"value"`列表*，那么你可能需要实现自己的解析逻辑。

go1.7中添加了`StructTag.Lookup()`方法，这个方法的行为类似于`Get()`，但其将不包含给定键的标签与将空字符串与给定键相关联的标签区分开来。

来看下面这个例子：
```go
type User struct {
    Name string `mytag:"MyName"`
    Email stirng `mytag:"MyEmail"`
}

u := User{"Bob", "bob@cc.com"}
t := reflect.TypeOf(u)

for _ fieldName := range []string{"Name", "Email"} {
    field, found := t.FieldByName(fieldName)
    if !found {
        continue
    }
    fmt.Printf("\nField: User.%s\n", fieldName)
    fmt.Printf("\tWhole tag value : %q\n", field.Tag)
    fmt.Printf("\tValue of 'mytag': %q\n", field.Tag.Get("mytag"))
}
```

输出为：
```
Field: User.Name
    Whole tag value : "mytag:\"MyName\""
    Value of 'mytag': "MyName"

Field: User.Email
    Whole tag value : "mytag:\"MyEmail\""
    Value of 'mytag': "MyEmail"
```

## 3. 常见的tag

- json      - used by the encoding/json package, detailed at json.Marshal()
- xml       - used by the encoding/xml package, detailed at xml.Marshal()
- bson      - used by gobson, detailed at bson.Marshal()
- protobuf  - used by github.com/golang/protobuf/proto, detailed in the package doc
- yaml      - used by the gopkg.in/yaml.v2 package, detailed at yaml.Marshal()
- db        - used by the github.com/jmoiron/sqlx package; also used by github.com/go-gorp/gorp package
- orm       - used by the github.com/astaxie/beego/orm package, detailed at Models – Beego ORM
- gorm      - used by gorm.io/gorm, examples can be found in their docs
- valid     - used by the github.com/asaskevich/govalidator package, examples can be found in the project page
- datastore - used by appengine/datastore (Google App Engine platform, Datastore service), detailed at Properties
- schema    - used by github.com/gorilla/schema to fill a struct with HTML form values, detailed in the package doc
- asn       - used by the encoding/asn1 package, detailed at asn1.Marshal() and asn1.Unmarshal()
- csv       - used by the github.com/gocarina/gocsv package
- env       - used by the github.com/caarlos0/env package

**Translated from: https://stackoverflow.com/questions/10858787/what-are-the-uses-for-tags-in-go**

