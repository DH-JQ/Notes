一：学习目的
通过对服务启动过程的跟踪和wsgi服务请求过程分析来学习phoenix框架

二：学习准备
1.打包工具setuptools、pbr，目的是了解setup.cfg中每个section的作用，明白模块是如何打包并安装，安装了哪些？模块相关的配置文件是要安装到哪？
2.spec文件，目的是了解.spec文件是什么？如何通过.spec文件构建一个rpm包的？
3.service文件，目的是了解.service文件是什么？如何通过.service文件构建一个服务通过systemd监管，以及通过systemctl启停
4.routes，作用是将业务请求路径和controller控制逻辑关联起来
5.webob，作用是将一个方法装饰为符合wsgi规范的wsgi app
6.paste.deploy，作用是通过配置的方式管理编排wsgi app
7.openstack通用组件库(oslo.config，oslo.log，oslo.messageing，oslo.db，oslo.service，...)

三：跟踪服务启动过程
从Apollo项目出发，了解基于phoenix框架构建的服务是如何启动的，下面从Apollo项目结构说起，Apollo项目结构如下：

Phoenix框架学习 - 第1张  | CMP技术博客

各目录功能如下:

目录	功能
apollo/app	业务逻辑目录
apollo/cmd	工程执行入库目录
apollo/common	工程公共目录
apollo/conf	业务配置目录
apollo/exception.py	工程异常定义模块
apollo/server	服务特效目录(RPC、周期任务)
apollo/tests	单元测试目录
etc	工程配置目录
setup.cfg	工程部署配置文件
在工程部署配置文件setup.cfg中的entry_points section可以看到apollo模块的各个服务入口：

Phoenix框架学习 - 第2张  | CMP技术博客

这里以apollo-api服务为出发点，可以从apollo/cmd/api.py模块看到服务的入口具体执行了那些步骤。

Phoenix框架学习 - 第3张  | CMP技术博客

从main函数可以得出，服务入口先是初始化了环境，然后通过objects模块注册后面RPC通信需要用到的序列化对象，最后启动服务端程序。
初始化环境是设置服务运行中需要的环境参数，查看phonenix.common.utils模块中的init_env方法：

Phoenix框架学习 - 第4张  | CMP技术博客

这里可以知道服务运行中需要PROJECT_NAME、PROJECT_DIR、POSSIBILE_TOPDIR、PROJECT_APP_DIR、PROJECT_CONF_DIR等环境参数，这些参数在服务的后续执行中发挥作用。
既然服务的启动是执行eventlet_server.run()这句代码，进到对应的模块里(phoenix.server.eventlet)查看run方法：

def run(topic=None, server=None, manager=None, serializer=None):
    dev_conf = os.path.join(os.environ['POSSIBLE_TOPDIR'],
                            'etc',
                            '%s.conf' % os.environ['PROJECT_NAME'])
    config_files = None
    if os.path.exists(dev_conf):
        config_files = [dev_conf]

    common.configure(
        version=pbr.version.VersionInfo(os.environ['PROJECT_NAME']).version_string(),
        config_files=config_files,
        pre_setup_logging_fn=configure_threading)
    
    paste_config = wsgi.find_paste_config()
    
    def _create_services():
        services = []
        max_workers = CONF.max_workers
        wsgi_workers = get_workers(CONF, 'wsgi')
        rpc_workers = get_workers(CONF, 'rpc')
        periodic_workers = get_workers(CONF, 'periodic')
        notification_workers = get_workers(CONF, 'notification')
    
        # Enable wsgi service.
        if wsgi_workers > 0:
            # 添加对多个服务端口的支持，CONF.port变更为列表类型
            services.append(
                create_wsgi_service(paste_config,
                                    'main',
                                    CONF.bind_host,
                                    CONF.port,
                                    min(wsgi_workers, max_workers)))
        ....
    
        return services
    
    _unused, services = common.setup_backends(
        startup_application_fn=_create_services)
    serve(*services)COPY
在run方法内部首先是执行加载配置文件的操作，来看下配置文件是如何加载的：

# 工程配置文件路径为工程同级etc目录下，如果工程已部署这个路径是不存在的
dev_conf = os.path.join(os.environ['POSSIBLE_TOPDIR'],
                        'etc',
                        '%s.conf' % os.environ['PROJECT_NAME'])

# 如果工程已部署，config_files=[None]
config_files = None
if os.path.exists(dev_conf):
    config_files = [dev_conf]

# 这里调用phoenix.server.common模块的configure方法的目的是注册业务逻辑配置项(apollo/conf)和加载由命令行(apollo-api --config-file /sf/etc/apollo/apollo.conf)通过--config-file传进来的配置
common.configure(
    version=pbr.version.VersionInfo(os.environ['PROJECT_NAME']).version_string(),  
    config_files=config_files,
    pre_setup_logging_fn=configure_threading)

> 我们来看下phoenix.server.common模块的这个configure方法
> def configure(version=None, config_files=None,
>          pre_setup_logging_fn=lambda: None):
# 调用phoenix.conf模块的configure方法，目的是注册命令行配置项和默认配置项以及业务逻辑配置项
phoenix.conf.configure()
>> 进入phonenix.conf模块的configure方法
>> def configure(conf=None, path=None):
>> if conf is None:
>>  conf = CONF
>> # 注册命令行配置项
>> conf.register_cli_opt(
>>  cfg.BoolOpt('standard-threads', default=False,
>>              help='Do not monkey-patch threading system modules.'))
>> conf.register_cli_opt(
>>  cfg.StrOpt('pydev-debug-host',
