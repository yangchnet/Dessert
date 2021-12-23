---
author: "李昌"
title: "pm2使用指南"
date: "2021-08-06"
tags: ["pm2", "tools"]
categories: ["Tool"]
ShowToc: true
TocOpen: true
---

## 1. 安装pm2
```bash
npm install pm2 -g
```
或
```bash
yarn global add pm2
```

使用`pm2 -v`查看版本号

## 2. 基本使用

### 2.1 启动应用
```bash
pm2 start app.js # 不止是js文件，其他可执行文件也可以执行

pm2 start script.sh # 启动bash脚本

pm2 start python3 -- app.py  # -- 后跟要传给命令的参数

pm2 start binary -- -port 8080
```

在启动应用时还有一些参数
```bash
--name <app_name> # 为应用设置一个名字

--watch # 监视源文件并在源文件存在更改时重启应用

--max-memory-restart <200MB> # 设置应用占用内存上限

--log <log_path> # 设置log文件路径

-- arg1 arg2 arg3 # 传递参数

--restart-delay <delay in ms>  # 重启前延时

--time 在日志前增加时间戳

--no-autorestart # 不要自动重启

```

### 2.2 管理应用
```bash
pm2 restart app_name 

pm2 reload app_name

pm2 stop app_name

pm2 delete app_name
```

除了使用`app_name`，还可以使用应用的`id`, 或使用`all`来批量管理所有应用

### 2.3 列出被管理的应用

```bash
pm2 list|ls|status
```
![20210806191943](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20210806191943.png)

### 2.4 打印log
```bash
pm2 log 
```

打印最后两百行：
```bash
pm2 log --line 200
```

### 2.5 显示监控
```bash
pm2 monit
```
![20210806192323](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20210806192323.png)


## 3. ecosystems.config.js -- 使用配置文件启动应用

### 3.1 基本使用
生成一个示例文件
```bash
pm2 init simple
```
![20210806192510](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20210806192510.png)

生成的示例文件如下：
```js
module.exports = {
  apps : [{
    name   : "app1",
    script : "./app.js"
  }]
}
```
上面的配置文件中定义了一个最简单的应用，其`name`为`app1`, 启动方式为`./app.js`

管理应用：
```bash
pm2 start ecosystem.config.js # 启动应用

pm2 stop ecosystem.config.js # 停止应用

pm2 restart ecosystem.config.js # 重启应用

pm2 reload ecosystem.config.js # 重新加载应用

pm2 delete ecosystem.config.js # 删除应用
```

如果你的配置文件中配置了多个文件，那他看起来可能长这样：
```js
module.exports = {
  apps : [
    {
        name   : "app1",
        script : "./app1.js"
    },
    {
        name: "app2",
        script : "./app2.js"
    },
    {
        name: "app3",
        script : "./app3.js"
    },
    {
        name: "app4",
        script : "./app4.js"
    }
  ]
}
```

如果想要单独管理配置文件中的某（几）个应用，可以使用`--only`参数
```bash
pm2 stop ecosystem.config.js --only app1 # 只停止app1

pm2 delete ecosystem.config.js --only "app1,app2"  # 只删除app1和app2
```
当然你还可以用他们的`id`来管理


### 3.2 配置不同环境

可能你有开发环境和生产环境两个环境，而二者在启动时可能需要不同的环境变量（如数据库节点），那么你可以为每个应用配置`env_*`
```js
module.exports = {
  apps : [
    {
        name   : "app1",
        script : "./app.js",
        env_production: {
            NODE_ENV: "production",
            SQL_ENDPOINT: "192.168.11.11",
        },
        env_development: {
            NODE_ENV: "development",
            SQL_ENDPOINT: "192.168.22.22",
        }
    }
  ]
}
```

在这个配置文件中, 定义了两套环境变量，你可以选择其中之一用来启动你的应用
```bash
pm2 start ecosystem.config.js --env production # 使用生产环境的环境变量，即：SQL_ENDPOINT = "192.168.11.11"

pm2 start ecosystem.config.js --env development # 使用开发环境的环境变量，即：SQL_ENDPOINT = "192.168.22.22"
```

### 3.3 其他参数

基本参数

|Field|Type|Example|Description|
|--- |--- |--- |--- |
|name|(string)|“my-api”|application name (default to script filename without extension)|
|script|(string)|”./api/app.js”|script path relative to pm2 start|
|cwd|(string)|“/var/www/”|the directory from which your app will be launched|
|args|(string)|“-a 13 -b 12”|string containing all arguments passed via CLI to script|
|interpreter|(string)|“/usr/bin/python”|interpreter absolute path (default to node)|
|interpreter_args|(string)|”–harmony”|option to pass to the interpreter|
|node_args|(string)||alias to interpreter_args|

高级参数

|Field|Type|Example|Description|
|--- |--- |--- |--- |
|instances|number|-1|number of app instance to be launched|
|exec_mode|string|“cluster”|mode to start your app, can be “cluster” or “fork”, default fork|
|watch|boolean or []|true|enable watch & restart feature, if a file change in the folder or subfolder, your app will get reloaded|
|ignore_watch|list|[”[\/\\]\./”, “node_modules”]|list of regex to ignore some file or folder names by the watch feature|
|max_memory_restart|string|“150M”|your app will be restarted if it exceeds the amount of memory specified. human-friendly format : it can be “10M”, “100K”, “2G” and so on…|
|env|object|{“NODE_ENV”: “development”, “ID”: “42”}|env variables which will appear in your app|
|env_|object|{“NODE_ENV”: “production”, “ID”: “89”}|inject  when doing pm2 restart app.yml --env|
|source_map_support|boolean|true|default to true, [enable/disable source map file]|
|instance_var|string|“NODE_APP_INSTANCE”|see documentation|
|filter_env|array of string|[ “REACT_” ]|Excludes global variables starting with “REACT_” and will not allow their penetration into the cluster.|

log配置  
|Field|Type|Example|Description|
|--- |--- |--- |--- |
|log_date_format|(string)|“YYYY-MM-DD HH:mm Z”|log date format (see log section)|
|error_file|(string)||error file path (default to $HOME/.pm2/logs/XXXerr.log)|
|out_file|(string)||output file path (default to $HOME/.pm2/logs/XXXout.log)|
|combine_logs|boolean|true|if set to true, avoid to suffix logs file with the process id|
|merge_logs|boolean|true|alias to combine_logs|
|pid_file|(string)||pid file path (default to $HOME/.pm2/pid/app-pm_id.pid)|

控制流参数

|Field|Type|Example|Description|
|--- |--- |--- |--- |
|min_uptime|(string)||min uptime of the app to be considered started|
|listen_timeout|number|8000|time in ms before forcing a reload if app not listening|
|kill_timeout|number|1600|time in milliseconds before sending a final SIGKILL|
|shutdown_with_message|boolean|false|shutdown an application with process.send(‘shutdown’) instead of process.kill(pid, SIGINT)|
|wait_ready|boolean|false|Instead of reload waiting for listen event, wait for process.send(‘ready’)|
|max_restarts|number|10|number of consecutive unstable restarts (less than 1sec interval or custom time via min_uptime) before your app is considered errored and stop being restarted|
|restart_delay|number|4000|time to wait before restarting a crashed app (in milliseconds). defaults to 0.|
|autorestart|boolean|false|true by default. if false, PM2 will not restart your app if it crashes or ends peacefully|
|cron_restart|string|“1 0 * * *”|a cron pattern to restart your app. Application must be running for cron feature to work|
|vizion|boolean|false|true by default. if false, PM2 will start without vizion features (versioning control metadatas)|
|post_update|list|[“npm install”, “echo launching the app”]|a list of commands which will be executed after you perform a Pull/Upgrade operation from Keymetrics dashboard|
|force|boolean|true|defaults to false. if true, you can start the same script several times which is usually not allowed by PM2|

部署相关参数

|Entry name|Description|Type|Default|
|--- |--- |--- |--- |
|key|SSH key path|String|$HOME/.ssh|
|user|SSH user|String||
|host|SSH host|[String]||
|ssh_options|SSH options with no command-line flag, see ‘man ssh’|String or [String]||
|ref|GIT remote/branch|String||
|repo|GIT remote|String||
|path|path in the server|String||
|pre-setup|Pre-setup command or path to a script on your local machine|String||
|post-setup|Post-setup commands or path to a script on the host machine|String||
|pre-deploy-local|pre-deploy action|String||
|post-deploy|post-deploy action|String||

## References

https://pm2.keymetrics.io/

