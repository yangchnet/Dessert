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

<table>
  <thead>
    <tr>
      <th style="text-align: left">Field</th>
      <th style="text-align: center">Type</th>
      <th style="text-align: center">Example</th>
      <th style="text-align: left">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align: left">name</td>
      <td style="text-align: center">(string)</td>
      <td style="text-align: center">“my-api”</td>
      <td style="text-align: left">application name (default to script filename without extension)</td>
    </tr>
    <tr>
      <td style="text-align: left">script</td>
      <td style="text-align: center">(string)</td>
      <td style="text-align: center">”./api/app.js”</td>
      <td style="text-align: left">script path relative to pm2 start</td>
    </tr>
    <tr>
      <td style="text-align: left">cwd</td>
      <td style="text-align: center">(string)</td>
      <td style="text-align: center">“/var/www/”</td>
      <td style="text-align: left">the directory from which your app will be launched</td>
    </tr>
    <tr>
      <td style="text-align: left">args</td>
      <td style="text-align: center">(string)</td>
      <td style="text-align: center">“-a 13 -b 12”</td>
      <td style="text-align: left">string containing all arguments passed via CLI to script</td>
    </tr>
    <tr>
      <td style="text-align: left">interpreter</td>
      <td style="text-align: center">(string)</td>
      <td style="text-align: center">“/usr/bin/python”</td>
      <td style="text-align: left">interpreter absolute path (default to node)</td>
    </tr>
    <tr>
      <td style="text-align: left">interpreter_args</td>
      <td style="text-align: center">(string)</td>
      <td style="text-align: center">”–harmony”</td>
      <td style="text-align: left">option to pass to the interpreter</td>
    </tr>
    <tr>
      <td style="text-align: left">node_args</td>
      <td style="text-align: center">(string)</td>
      <td style="text-align: center">&nbsp;</td>
      <td style="text-align: left">alias to interpreter_args</td>
    </tr>
  </tbody>
</table>

高级参数

<table>
  <thead>
    <tr>
      <th style="text-align: left">Field</th>
      <th style="text-align: center">Type</th>
      <th style="text-align: center">Example</th>
      <th style="text-align: left">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align: left">instances</td>
      <td style="text-align: center">number</td>
      <td style="text-align: center">-1</td>
      <td style="text-align: left">number of app instance to be launched</td>
    </tr>
    <tr>
      <td style="text-align: left">exec_mode</td>
      <td style="text-align: center">string</td>
      <td style="text-align: center">“cluster”</td>
      <td style="text-align: left">mode to start your app, can be “cluster” or “fork”, default fork</td>
    </tr>
    <tr>
      <td style="text-align: left">watch</td>
      <td style="text-align: center">boolean or []</td>
      <td style="text-align: center">true</td>
      <td style="text-align: left">enable watch &amp; restart feature, if a file change in the folder or subfolder, your app will get reloaded</td>
    </tr>
    <tr>
      <td style="text-align: left">ignore_watch</td>
      <td style="text-align: center">list</td>
      <td style="text-align: center">[”[\/\\]\./”, “node_modules”]</td>
      <td style="text-align: left">list of regex to ignore some file or folder names by the watch feature</td>
    </tr>
    <tr>
      <td style="text-align: left">max_memory_restart</td>
      <td style="text-align: center">string</td>
      <td style="text-align: center">“150M”</td>
      <td style="text-align: left">your app will be restarted if it exceeds the amount of memory specified. human-friendly format : it can be “10M”, “100K”, “2G” and so on…</td>
    </tr>
    <tr>
      <td style="text-align: left">env</td>
      <td style="text-align: center">object</td>
      <td style="text-align: center">{“NODE_ENV”: “development”, “ID”: “42”}</td>
      <td style="text-align: left">env variables which will appear in your app</td>
    </tr>
    <tr>
      <td style="text-align: left">env_<env_name></env_name></td>
      <td style="text-align: center">object</td>
      <td style="text-align: center">{“NODE_ENV”: “production”, “ID”: “89”}</td>
      <td style="text-align: left">inject <env_name> when doing pm2 restart app.yml --env <env_name></env_name></env_name></td>
    </tr>
    <tr>
      <td style="text-align: left">source_map_support</td>
      <td style="text-align: center">boolean</td>
      <td style="text-align: center">true</td>
      <td style="text-align: left">default to true, [enable/disable source map file]</td>
    </tr>
    <tr>
      <td style="text-align: left">instance_var</td>
      <td style="text-align: center">string</td>
      <td style="text-align: center">“NODE_APP_INSTANCE”</td>
      <td style="text-align: left"><a href="http://pm2.keymetrics.io/docs/usage/environment/#specific-environment-variables">see documentation</a></td>
    </tr>
    <tr>
      <td style="text-align: left">filter_env</td>
      <td style="text-align: center">array of string</td>
      <td style="text-align: center">[ “REACT_” ]</td>
      <td style="text-align: left">Excludes global variables starting with “REACT_” and will not allow their penetration into the cluster.</td>
    </tr>
  </tbody>
</table>

log配置  
<table>
  <thead>
    <tr>
      <th style="text-align: left">Field</th>
      <th style="text-align: center">Type</th>
      <th style="text-align: center">Example</th>
      <th style="text-align: left">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align: left">log_date_format</td>
      <td style="text-align: center">(string)</td>
      <td style="text-align: center">“YYYY-MM-DD HH:mm Z”</td>
      <td style="text-align: left">log date format (see log section)</td>
    </tr>
    <tr>
      <td style="text-align: left">error_file</td>
      <td style="text-align: center">(string)</td>
      <td style="text-align: center">&nbsp;</td>
      <td style="text-align: left">error file path (default to $HOME/.pm2/logs/XXXerr.log)</td>
    </tr>
    <tr>
      <td style="text-align: left">out_file</td>
      <td style="text-align: center">(string)</td>
      <td style="text-align: center">&nbsp;</td>
      <td style="text-align: left">output file path (default to $HOME/.pm2/logs/XXXout.log)</td>
    </tr>
    <tr>
      <td style="text-align: left">combine_logs</td>
      <td style="text-align: center">boolean</td>
      <td style="text-align: center">true</td>
      <td style="text-align: left">if set to true, avoid to suffix logs file with the process id</td>
    </tr>
    <tr>
      <td style="text-align: left">merge_logs</td>
      <td style="text-align: center">boolean</td>
      <td style="text-align: center">true</td>
      <td style="text-align: left">alias to combine_logs</td>
    </tr>
    <tr>
      <td style="text-align: left">pid_file</td>
      <td style="text-align: center">(string)</td>
      <td style="text-align: center">&nbsp;</td>
      <td style="text-align: left">pid file path (default to $HOME/.pm2/pid/app-pm_id.pid)</td>
    </tr>
  </tbody>
</table>

控制流参数

<table>
  <thead>
    <tr>
      <th style="text-align: left">Field</th>
      <th style="text-align: center">Type</th>
      <th style="text-align: center">Example</th>
      <th style="text-align: left">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align: left">min_uptime</td>
      <td style="text-align: center">(string)</td>
      <td style="text-align: center">&nbsp;</td>
      <td style="text-align: left">min uptime of the app to be considered started</td>
    </tr>
    <tr>
      <td style="text-align: left">listen_timeout</td>
      <td style="text-align: center">number</td>
      <td style="text-align: center">8000</td>
      <td style="text-align: left">time in ms before forcing a reload if app not listening</td>
    </tr>
    <tr>
      <td style="text-align: left">kill_timeout</td>
      <td style="text-align: center">number</td>
      <td style="text-align: center">1600</td>
      <td style="text-align: left">time in milliseconds before sending <a href="http://pm2.keymetrics.io/docs/usage/signals-clean-restart/#cleaning-states-and-jobs">a final SIGKILL</a></td>
    </tr>
    <tr>
      <td style="text-align: left">shutdown_with_message</td>
      <td style="text-align: center">boolean</td>
      <td style="text-align: center">false</td>
      <td style="text-align: left">shutdown an application with process.send(‘shutdown’) instead of process.kill(pid, SIGINT)</td>
    </tr>
    <tr>
      <td style="text-align: left">wait_ready</td>
      <td style="text-align: center">boolean</td>
      <td style="text-align: center">false</td>
      <td style="text-align: left">Instead of reload waiting for listen event, wait for process.send(‘ready’)</td>
    </tr>
    <tr>
      <td style="text-align: left">max_restarts</td>
      <td style="text-align: center">number</td>
      <td style="text-align: center">10</td>
      <td style="text-align: left">number of consecutive unstable restarts (less than 1sec interval or custom time via min_uptime) before your app is considered errored and stop being restarted</td>
    </tr>
    <tr>
      <td style="text-align: left">restart_delay</td>
      <td style="text-align: center">number</td>
      <td style="text-align: center">4000</td>
      <td style="text-align: left">time to wait before restarting a crashed app (in milliseconds). defaults to 0.</td>
    </tr>
    <tr>
      <td style="text-align: left">autorestart</td>
      <td style="text-align: center">boolean</td>
      <td style="text-align: center">false</td>
      <td style="text-align: left">true by default. if false, PM2 will not restart your app if it crashes or ends peacefully</td>
    </tr>
    <tr>
      <td style="text-align: left">cron_restart</td>
      <td style="text-align: center">string</td>
      <td style="text-align: center">“1 0 * * *”</td>
      <td style="text-align: left">a cron pattern to restart your app. Application must be running for cron feature to work</td>
    </tr>
    <tr>
      <td style="text-align: left">vizion</td>
      <td style="text-align: center">boolean</td>
      <td style="text-align: center">false</td>
      <td style="text-align: left">true by default. if false, PM2 will start without vizion features (versioning control metadatas)</td>
    </tr>
    <tr>
      <td style="text-align: left">post_update</td>
      <td style="text-align: center">list</td>
      <td style="text-align: center">[“npm install”, “echo launching the app”]</td>
      <td style="text-align: left">a list of commands which will be executed after you perform a Pull/Upgrade operation from Keymetrics dashboard</td>
    </tr>
    <tr>
      <td style="text-align: left">force</td>
      <td style="text-align: center">boolean</td>
      <td style="text-align: center">true</td>
      <td style="text-align: left">defaults to false. if true, you can start the same script several times which is usually not allowed by PM2</td>
    </tr>
  </tbody>
</table>

部署相关参数

<table>
  <thead>
    <tr>
      <th>Entry name</th>
      <th>Description</th>
      <th>Type</th>
      <th>Default</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>key</td>
      <td>SSH key path</td>
      <td>String</td>
      <td>$HOME/.ssh</td>
    </tr>
    <tr>
      <td>user</td>
      <td>SSH user</td>
      <td>String</td>
      <td>&nbsp;</td>
    </tr>
    <tr>
      <td>host</td>
      <td>SSH host</td>
      <td>[String]</td>
      <td>&nbsp;</td>
    </tr>
    <tr>
      <td>ssh_options</td>
      <td>SSH options with no command-line flag, see ‘man ssh’</td>
      <td>String or [String]</td>
      <td>&nbsp;</td>
    </tr>
    <tr>
      <td>ref</td>
      <td>GIT remote/branch</td>
      <td>String</td>
      <td>&nbsp;</td>
    </tr>
    <tr>
      <td>repo</td>
      <td>GIT remote</td>
      <td>String</td>
      <td>&nbsp;</td>
    </tr>
    <tr>
      <td>path</td>
      <td>path in the server</td>
      <td>String</td>
      <td>&nbsp;</td>
    </tr>
    <tr>
      <td>pre-setup</td>
      <td>Pre-setup command or path to a script on your local machine</td>
      <td>String</td>
      <td>&nbsp;</td>
    </tr>
    <tr>
      <td>post-setup</td>
      <td>Post-setup commands or path to a script on the host machine</td>
      <td>String</td>
      <td>&nbsp;</td>
    </tr>
    <tr>
      <td>pre-deploy-local</td>
      <td>pre-deploy action</td>
      <td>String</td>
      <td>&nbsp;</td>
    </tr>
    <tr>
      <td>post-deploy</td>
      <td>post-deploy action</td>
      <td>String</td>
      <td>&nbsp;</td>
    </tr>
  </tbody>
</table>


## References

https://pm2.keymetrics.io/

