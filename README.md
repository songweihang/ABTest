# ABTest
基于[openresty](https://openresty.org) 开发的高性能灰度测试系统，以及支持动态调整upstream可以实现动态伸缩

1、可以通过用户IP或者UID进行设置Nginx灰度分流到指定的upstream服务器

2、通过API动态调整upstream实现后端服务器伸缩

3、所有操作通过HTTP API 进行动态操作无需重启Nginx

4、基于redis为数据库实现集群数据共享

5、高容错率，即使redis挂掉系统会启动默认upstream

6、实现类似Nginx(weight)权重分流

### 说明
该项目非常感谢新浪 [ABTestingGateway](https://github.com/CNSRE/ABTestingGateway) 提供的开源代码,本项目中大量采用ABTestingGateway的设计思路，而本项目不同点在于，本项目是基于ngx.balancer实现动态分流，并且可以通过API动态调整upstream的转发ip以及端口亦可作为后端服务伸缩系统，而且可以支持到openresty-1.9.7.5以上版本，但是灰度发布只支持IP或UID分流

### 安装
如下是nginx.conf的最小配置 实际knight放路径可以根据实际情况进行调整

    lua_package_path "/path/knight/?.lua;;";
    lua_code_cache on;
    lua_check_client_abort on;
    
    lua_max_pending_timers 1024;
    lua_max_running_timers 256;
    include /path/knight/config/lua.shared.dict;

    upstream backend {
        server 0.0.0.1;   # just an invalid address as a place holder
        balancer_by_lua_file /path/knight/apps/by_lua/balancer.lua;
        keepalive 500;  # connection pool
    }  

    init_by_lua_file /path/knight/apps/by_lua/init.lua;
    
    server
    {
        server_name api.host.cn;
        index index.html index.htm;
        
        location ~ ^/admin/([-_a-zA-Z0-9/]+) {
            set $path $1;
            content_by_lua_file '/path/knight/admin/$path.lua'; 
        }
    }

    server
    {
        server_name domain.host.cn;
        index index.html index.htm;
        #设置默认转发集群weight值越大转发概率越高，在没有通过API设置或者极端情况redis挂掉都会采用如下转发规则
        #weight值设置大于1小于10
        set $default_upstream '[{"ip": "127.0.0.1","port": 8081,"weight":10},{"ip": "127.0.0.1","port": 8082,"weight":5}]';
        location / {
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Connection "";
            proxy_http_version 1.1;
            proxy_pass http://backend;
        }
    }

### 服务配置与使用

knight/config/init.lua 服务配置说明    

    -- redis 服务配置
    _M.redisConf = {
        ["uds"]      = nil,
        ["host"]     = '127.0.0.1',
        ["port"]     = '6379',
        ["poolsize"] = 2000,
        ["idletime"] = 90000, 
        ["timeout"]  = 1000,
        ["dbid"]     = 0,
        ["auth"]     = ''
    }

    -- 设置白名单
    _M.whitelist_ips = {
          "127.0.0.1",
          "10.0.0.0/8",
          "172.16.0.0/12",
          "192.168.0.0/16",
    }


### API使用
[ABTest-DOC](https://github.com/songweihang/knight/blob/master/apps/lib/abtest/abtest-doc.txt)

### License

MIT 