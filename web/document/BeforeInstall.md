# 安装之前应该了解一下内容

## 前台和后台的关系

该项目分为前台和后台，前台代码位于qfe目录下，后台代码位于web目录下，二者的关系为

后台通过一个内网的api接口提供一个可供调用的HTTP API, 供前台来调用，这个API提供了一个JSON格式的全局的配置，

主要包括，有哪些业务，各个业务使用的HTTPS证书，各个业务的负载均衡配置等。

由此可见前台是依赖后台的配置工作的。所以正确的部署方案应该是，先后台后前台。

## 后台提供的全局配置返回值详解

了解了这个接口返回值的含义，基本就可以了解整个HTTPSLayer的工作过程

下面是一份demo的样例

```text
{
    "version": 125,
    "code": 0,
    "data": {
        "cert": {
            "list": {
                "20": {
                    "pids": [
                        "Biz_Pid1",
                        "Biz_Pid2"
                    ],
                    "regs": [
                        "*.tupian.so.com",
                        "tupian.so.com"
                    ],
                    "cert_redis_key": "pub_d41d8cd98f000004e9800998ecf8vvvve_1511255883",
                    "pkey_redis_key": "priv_d41d8cd98f000004e9800998ecfeeeee_1511255883"
                },
                "33": {
                    "pids": [
                        "Biz_Pid3",
                        "Biz_Pid4"
                    ],
                    "regs": [
                        "*.xinwen.so.com",
                        "xinwen.so.com"
                    ],
                    "cert_redis_key": "pub_d41d8cd98f000004e9800998ecf8vvvve_1511255884",
                    "pkey_redis_key": "priv_d41d8cd98f000004e9800998ecfeeeee_1511255884"
                }
            },
            ......
        },
        "balancer": {
            "Biz_Pid1": {
                "t": 1,
                "vips": [
                    {
                        "vip": "100.10.100.1",
                        "w": "0"
                    },
                    {
                        "vip": "100.11.100.2",
                        "w": "0"
                    },
                    {
                        "vip": "100.12.100.3",
                        "w": "0"
                    },
                    {
                        "vip": "100.13.100.4",
                        "w": "0"
                    },
                    {
                        "vip": "100.11.100.5",
                        "w": "100"
                    }
                ]
            },
            "Biz_Pid2": {
                "t": 1,
                "vips": [
                    {
                        "vip": "100.10.101.2",
                        "w": "0"
                    },
                    {
                        "vip": "100.11.101.3",
                        "w": "0"
                    },
                    {
                        "vip": "100.12.101.4",
                        "w": "50"
                    },
                    {
                        "vip": "100.13.101.5",
                        "w": "50"
                    }
                ]
            },
            "Biz_Pid3": {
                "t": 1,
                "vips": [
                    {
                        "vip": "100.10.102.2",
                        "w": "100"
                    },
                    {
                        "vip": "100.11.102.3",
                        "w": "0"
                    },
                    {
                        "vip": "100.12.102.4",
                        "w": "0"
                    },
                    {
                        "vip": "100.13.102.5",
                        "w": "0"
                    }
                ]
            },
            "Biz_Pid4": {
                "t": 1,
                "vips": [
                    {
                        "vip": "100.10.103.2",
                        "w": "100"
                    },
                    {
                        "vip": "100.11.103.3",
                        "w": "0"
                    },
                    {
                        "vip": "100.12.103.4",
                        "w": "0"
                    },
                    {
                        "vip": "100.13.103.5",
                        "w": "0"
                    }
                ]
            },
            ......
        },
        "app": [
            "Biz_Pid1",
            "Biz_Pid2",
            "Biz_Pid3",
            "Biz_Pid4"
        ],
        "label": [
            "Biz_Pid1",
            "Biz_Pid2",
            "Biz_Pid3",
            "Biz_Pid4"
        ]
    }
}
```

下面针对这个样例来解释一下，具体有哪些受后台控制的。

* `version`： 是一个版本号，表示当前的全局配置的版本，前台的实例，一旦发现版本号与自己之前拿到的不一致，就会认为配置有更新，将新的更新到自己的内存中，并用于线上。

* `cert` - `list`: 为证书的相关配置`pids`表示后台哪些业务用到了这张证书， pid是一个在后台中创建业务时指定的一个业务标志。`regs` 是该张证书支持的域名。
证书的内容存储在redis中，故提供了`cert_redis_key`， `pkey_redis_key`，redis的key,分别用来获取证书的.crt文件，和.key文件的内容。 由此可见证书并没有物理上存储在前端机器中。

* `balancer`：为各个业务的负载均衡标志，按照pid分组。`vips` 包含两项 `vip`, 表示业务的流量应该转发到业务的那个vip下，`w`是权重，表示该vip分得百分之w的流量。
如Biz_Pid2 这个业务，给100.12.101.4 分50%的流量，给100.13.101.5 分50%的流量，100.11.101.3 和100.11.101.2 不承担流量。

* `app` 和`label`： 目前取值是一样的，表示哪些业务在用该服务，与工作原理关系不大。



前台由基于openresty的lua编写，后台由php的Yii2 框架编写。

* 前台应该具备的技术栈或者依赖的技术有：openresty, nginx, lua， lvs等
* 后台应该具备的技术栈或者依赖的技术有：nginx, php, yii2, composer， mysql， redis 等