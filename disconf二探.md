## disconf分布式配置管理

昨天简单的过了一遍的流程源码，印象不是很深，我们还要继续。根据我个人的风格，我会完全的重新的过一遍。

-----

根据文档的说明，我们需要关注的的两个核心类。

* DisconfMgrBean
* DisconfMgrBeanSecond

-----

#### DisconfMgrBean类

在文档中，作者说明这个是一定需要引入的类。我们看一下该类的信息 

    DisconfMgrBean implements BeanDefinitionRegistryPostProcessor, PriorityOrdered, ApplicationContextAware

该类的接口有些特别，熟悉spring的童鞋知道，三个接口都来源自spring的Context相关包。

    * BeanDefinitionRegistryPostProcessor spring的后置处理器之一。
    * PriorityOrdered 处理器的优先级类。
    * ApplicationContextAware 获得上下文

回过头来，该类接口的实现类，优先级排到了最高位，也就是会最早被使用后置处理器，另外由于BeanDefinitionRegistryPostProcessor的特殊性
可以对BeanDefinition处理。我们来看一下这个接口实现（**postProcessBeanDefinitionRegistry**）。  

---

### postProcessBeanDefinitionRegistry（BeanDefinitionRegistry)解析
上源码:

        // 为了做兼容
        DisconfCenterHostFilesStore.getInstance().addJustHostFileSet(fileList);

        //从scanPackage字符串获得包名（去重），分隔符为“，”，。

        // 进行扫描
        DisconfMgr.getInstance().setApplicationContext(applicationContext);
        DisconfMgr.getInstance().firstScan(scanPackList);

        // register java bean
        registerAspect(registry);

作者在代码上做了注释。我做了简单的处理。现在来关注重点。  
这里我们首先介绍该方法所属**DisconfMgr**。
这个类是应用单例，根据作者所添加注释，这个是**disconf**客户端的总入口。
接下来关注两个方法入口方法。

**入口方法之firstScan(packageNameList)**  
话不多说，直接上源码：

     // 第一次扫描，静态扫描 for annotation config
    protected synchronized void firstScan(List<String> scanPackageList) {

            // 导入配置
            ConfigMgr.init();

            ......

            // registry
            Registry registry = RegistryFactory.getSpringRegistry(applicationContext);

            // 扫描器
            scanMgr = ScanFactory.getScanMgr(registry);

            // 第一次扫描并入库
            scanMgr.firstScan(scanPackageList);

            // 获取数据/注入/Watch
            disconfCoreMgr = DisconfCoreFactory.getDisconfCoreMgr(registry);
            disconfCoreMgr.process();

            ......

    }

作者文档描述还是比较详细的。 我们直接看贴出来的部分


1. ConfigMgr.init()

这里主要是两部分

        * 加载系统配置
        * 加载用户配置

总共涉及到4个类，其中3个是应用单例，1个工具类具体如下： 

    * DisClientComConfig   通用配置类 单例
    * DisClientSysConfig   系统配置类 单例
    * DisClientConfig      用户配置类 单例
    * DisInnerConfigHelper 配置校验工具类

    ----

    下面进行具体的说明。

    * DisClientComConfig.getInstance().getInstanceFingerprint()  
        DisClientComConfig类描述些通用的信息（提供了该类的标识:主机+端口(默认0）+UUID），上面返回信息。
        其中host可以配置通过系统参数：VCAP_APP_HOST。端口也是可配置的: VCAP_APP_HOST。

    * DisClientSysConfig.getInstance().loadConfig(null);   
        这一步会使用文件名：disconf_sys.properties(存储在客户端jar中)，
        然后委托给配工具类**DisconfAutowareConfig**做如下两步：
            
            * 加载读取文件的配置 
            这里也涉及到两种方式读取。

                * URL方式
                    这个使用getResource方法会调用父类加载来加载文件，因为在tomcat中应用类加载器重写规则有点特别，所以我们调用其父类加载器。
                    然后就是URI的处理了。这里应该是getResourceAsStream()方式来的。
                * 直接文件方式
                    直接文件流操作了，十分的简单。
            * 注入系统配置单例的属性中。
            最后把得到的prop配置对象注入系统配置类（DisClientSysConfig）。
            这个注入就很简单了利用反射和注解。将设置DisClentSysyConfig的4个字段，在下文提到。

    * DisInnerConfigHelper.verifySysConfig();  
        对上面prop配置对象注入后，进行一些配置校验，主要有这些(可以修改)，下面上文说到的字段对应，全部必要:

        * disconf.conf_server_store_action 仓库 URL  默认/api/config
        * disconf.conf_server_zoo_action  zk URL    默认/api/zoo
        * disconf.conf_server_master_num_action 获取远程主机个数的URL  默认//api/getmasterinfo 
        * disconf.local_download_dir  下载文件夹, 远程文件下载后会放在这里  默认./disconf/download

    * DisClientConfig.getInstance().loadConfig(null);  
        同上文加载系统配置，这里是加载用户的配置。
        配置文件默认是disconf.properties。这里提供了命令行的可选项（首选项）：
        使用java命令行-d 进行导入文件ex：-Ddisconf.conf=xxxx
        后面的步骤是一模一样的了。但是多了一步使用系统参数会决定配置，就是某些参数可以使用-D传递或者进行覆盖。

    * DisInnerConfigHelper.verifyUserConfig(); 
    同样和上面一样也有校验。
        
        * disconf.conf_server_host 配置文件服务器 HOST（web端的地址):字段hostList保存
        * disconf.app app名   
        * disconf.version 版本  默认值（DEFAULT_VERSION）
        * disconf.env 部署环境  默认值（DEFAULT_ENV）
        * 不必要的：  
            * disconf.enable.remote.conf 是否从云端下载配置 默认false(分布式配置关键) 
            * disconf.debug 是否开启DEBUG模式: 默认不开启，默认false  
                * true: 用于线下调试，当ZK断开与client连接后（如果设置断点，这个事件很容易就发生），ZK不会去重新建立连接。  
                * false: 用于线上，当ZK断开与client连接后，ZK会再次去重新建立连接。
            * disconf.user_define_download_dir 用户下载文件夹 默认./disconf/download 没有则创建
            * disconf.ignore 忽略哪些分布式配置或者配置文件 默认空：字段ignoreDisconfKeySet保存
            * disconf.conf_server_url_retry_times 获取远程配置 重试次数，默认是3次
            * disconf.conf_server_url_retry_sleep_seconds 获取远程配置 重试时休眠时间，默认是5秒
            * disconf.enable_local_download_dir_in_class_path 让下载文件夹放在 classpath目录  

        经过上面一步系统和用户的配置都会被载入完毕。
    至此第一步完成。

2. getSpringRegistry（applicationContext）  
    一个SpringRegistry的注册表，装饰了一个简单的注册表**simpleRegistry**
    目的是为了获得bean，按类型获得bean。当然如果不是spring管的那么，就会使用simpleRegistry直接new出实例吗，
    因此simpleRegistry基本是个软肋，主要还是直接操spring的applicationContext

3. ScanFactory.getScanMgr(registry)  
    构造了一个扫描模块ScanMgrImpl，组装了注册表，并添加了几个静态扫描器。

         public ScanMgrImpl(Registry registry) {

            this.registry = registry;

            // 配置文件
            staticScannerMgrList.add(StaticScannerMgrFactory.getDisconfFileStaticScanner());

            // 配置项
            staticScannerMgrList.add(StaticScannerMgrFactory.getDisconfItemStaticScanner());

            // 非注解 托管的配置文件
            staticScannerMgrList.add(StaticScannerMgrFactory.getDisconfNonAnnotationFileStaticScanner());
        }   
    从构造函数也可以看出来添加了三个上面所说的静态扫描器，针对不同的配置

4. scanMgr.firstScan(scanPackageList)  
   第一次扫描并入库，作者的注释如此的，这里是我们要考虑的重点。然我们看起函数内部实现。

        扫描并存储(静态)
        public void firstScan(List<String> packageNameList) throws Exception {

            // 获取扫描对象并分析整合
            scanModel = scanStaticStrategy.scan(packageNameList);

            // 增加非注解的配置
            scanModel.setJustHostFiles(DisconfCenterHostFilesStore.getInstance().getJustHostFiles());

            // 放进仓库
            for (StaticScannerMgr scannerMgr : staticScannerMgrList) {

                // 扫描进入仓库
                scannerMgr.scanData2Store(scanModel);

                // 忽略哪些KEY
                scannerMgr.exclude(DisClientConfig.getInstance().getIgnoreDisconfKeySet());
            }
        }
    * scanStaticStrategy.scan(packageNameList)  
    首先扫描包系统只有一个实现，使用反射扫描 里面也很简单构建了一个ScanStaticModel类型
        
        * 利用放射得到基本信息，设置ScanStaticModel对象中，主要是以下内容:

            * 设置reflections 对象，
            * 设置disconfFileClassSet 存储带DisconfFile注解的类信息：分布式配置文件
            * 设置disconfFileItemMethodSet 存储带DisconfFileItem注解的方法信息：分布式配置文件中的配置项
            * 设置disconfItemMethodSet 存储带DisconfItem注解的方法信息：分布式配置项
            * 设置disconfActiveBackupServiceClassSet 存储带DisconfActiveBackupService注解的类信息：主备切换的时候受影响的配置
            * 设置disconfUpdateService 存储带DisconfUpdateService注解的类信息：配置更新时候，要配置的数据
            * 设置idisconfUpdatePipeline 存储实现IDisconfUpdatePipeline类：配置跟新的时候，通用的回调接口
        
        * 接着将上面的基本信息进行整理：目的将配置文件，以及其中配置的内容项对应起来

            *　构造一个Map<disconfFileClass,disconfFileItemMethod>的映射。
                * DisconfFile指的是分布式配置文件。
                * DisconfFileItem指的是分布式配置文件中的配置项
            *　反射处理添加这些映射关系
            *  校验配置文件(作者目前只支持（.properties类型)
            *  设置disconfFileItemMap 存储实现上面的映射关系

    * scanModel.setJustHostFiles(DisconfCenterHostFilesStore.getInstance().getJustHostFiles()); 
    增加非注解的配置 只是托管的配置文件，没有注入到类中 也就是提供了桩（只是进行托管的配置文件（不会注入，只负责动态推送））
    
    *  scannerMgr.scanData2Store(scanModel);
    scannerMgr.exclude(DisClientConfig.getInstance().getIgnoreDisconfKeySet());
    这两部根据作者的注释就是放入仓库喽。默认的策略（三种静态扫描器）都会进行
        
        * 配置文件的静态扫描 StaticScannerFileMgrImpl
        * 配置项的静态扫描 StaticScannerItemMgrImpl
        * 非注解配置文件的扫描器 StaticScannerNonAnnotationFileMgrImpl
        
        ---

        ### StaticScannerFileMgrImpl配置文件的静态扫描
           public void scanData2Store(ScanStaticModel scanModel) {
                // 转换配置文件
                List<DisconfCenterBaseModel> disconfCenterFiles = getDisconfFiles(scanModel);
                DisconfStoreProcessorFactory.getDisconfStoreFileProcessor().transformScanData(disconfCenterFiles);
            }

        很简短，我们看一下，里面也很简单，就是从上面所说的映射设置disconfFileItemMap 取得每一个键值对组装成**DisconfCenterFile**类
        **DisconfCenterFile**是分布式配置文件的内存对象表示。然后将该对象和文件名组成映射放入单例配置仓库中DisconfCenterStore

         ### StaticScannerItemMgrImpl配置项的静态扫描
           public void scanData2Store(ScanStaticModel scanModel) {
                // 转换配置文件
                List<DisconfCenterBaseModel> disconfCenterFiles = getDisconfFiles(scanModel);
                DisconfStoreProcessorFactory.getDisconfStoreFileProcessor().transformScanData(disconfCenterFiles);
            }

        同上，把ScanStaticModel中关于配置项的disconfItemMethodSet中取得每一个键值对组装成**DisconfCenterItem**类
        **DisconfCenterItem**是分布式配置项的内存对象表示。然后同上存储在配置仓库中

        ### StaticScannerNonAnnotationFileMgrImpl非注解配置文件的静态扫描
            public void scanData2Store(ScanStaticModel scanModel) {
                List<DisconfCenterBaseModel> disconfCenterBaseModels = getDisconfCenterFiles(scanModel.getJustHostFiles());
                DisconfStoreProcessorFactory.getDisconfStoreFileProcessor().transformScanData(disconfCenterBaseModels);
            }
        
        同上，这里是把配置文件映射成**DisconfCenterFile**

    最后就是排除自己先前设定好的一些不必要的配置喽。


5. disconfCoreMgr = DisconfCoreFactory.getDisconfCoreMgr(registry);disconfCoreMgr.process();
    获取数据/注入/Watch，这里涉及到了核心模块了，默认配置（disconf.enable.remote.conf=true）开启后，才会从远端下载文件 

        public static DisconfCoreMgr getDisconfCoreMgr(Registry registry) throws Exception {

            FetcherMgr fetcherMgr = FetcherFactory.getFetcherMgr();

            //
            // 不开启disconf，则不要watch了
            //
            WatchMgr watchMgr = null;
            if (DisClientConfig.getInstance().ENABLE_DISCONF) {
                // Watch 模块
                watchMgr = WatchFactory.getWatchMgr(fetcherMgr);
            }

            return new DisconfCoreMgrImpl(watchMgr, fetcherMgr, registry);
        }
    * 生成一个抓取规则FetcherMgr getFetcherMgr()。获取抓取器实例。
        
        * 包含默认抓取器（http的规则，重试机制）
        * 相应的配置属性，本机信息（DisClientConfig的信息（应用单例））

    * 开启监控（Watch模块），设置ZK的信息(httpq请求web得到zk信息喝前缀)，并连接zk进行监听
    * 生成DisconfCoreMgrImpl对象，这是核心处理器

        * 添加分布配置文件的处理器
        * 添加分布式配置项的处理器
    * 使用处理器处理配置

        ---

        ### DisconfFileCoreProcessorImpl配置文件处理器实现
            
            public void processAllItems() {
                /**
                * 配置文件列表处理
                */
                for (String fileName : disconfStoreProcessor.getConfKeySet()) {

                    processOneItem(fileName);
                }
            }
        深入processOneItem，里面就是跟新DisconfCenterFile

            updateOneConfFile(key, disconfCenterFile);
        如同作者注释所说:更新 一個配置文件, 下载、注入到仓库、Watch 三步骤

            * 从web端下载文件到本地，下载失败就使用本地配置
            * 从文件中得到配置，目前支持properties格式
            * 将配置文件名，注入到仓库中
            * 监控zk下该文件路径
        这三步发现是比较重要的环节，我们需要重新考虑:前提是在远程开启的情况下

            * 从web端下载相关文件到本地，返回路劲。失败的情况下会使用本地的配置 
            * 从相关文件中得到配置，目前支持properties格式
            * 将配置文件名，注入到仓库中

                * 这里如果是xml配置的话，并且文件后缀是.properties要自动加载到spring管理的bean中来
                    上源码：

                    public static void reload() {
                        for (ReconfigurableBean bean : reconfigurableBeans) {
                            try {
                                bean.reloadConfiguration();
                            } catch (Exception e) {
                                logger.warn("while reloading configuration of " + bean, e);
                            }
                        }
                    }
                最后使用ReloadablePropertiesFactoryBean 来进行doReload。就是设置自身的某个属性：reloadableProperties，
                然后使用观察发布模式，通知属性的变化。重载

至此firstScan()过程全部完成

-----


**重点之secondScan()**

这个是作者总入口的第二个重要的方法，作者也加了注释：第二次扫描, 动态扫描, for annotation config


方法源码：

    /**
     * 第二次扫描, 动态扫描, for annotation config
     */
    protected synchronized void secondScan() {

        try {

            // 扫描回调函数
            if (scanMgr != null) {
                scanMgr.secondScan();
            }

            // 注入数据至配置实体中
            // 获取数据/注入/Watch
            if (disconfCoreMgr != null) {
                disconfCoreMgr.inject2DisconfInstance();
            }

        } catch (Exception e) {
            LOGGER.error(e.toString(), e);
        }

        isSecondInit = true;
    }
我们只看重点的try部分，其他部分已经被省略。

1. scanMgr.secondScan(); 
第二次扫描，作者的配置写的非常的明显了是扫描回调函数，我们进去看一下

    * 核心一句代码：ScanDynamicStoreAdapter.scanUpdateCallbacks(scanModel, registry);
    作者注释：将回调函数实例化并写入仓库。

        * ScanDynamicModel scanDynamicModel = analysis4DisconfUpdate(scanModel, registry);
            * 从静态扫描存的结构ScanStaticModel，得到注解DisconfUpdateService
            * 验证带该注解的类是否实现回调接口**IDisconfUpdate*
            * 使用注册表获取实例接口，**优先从spring中获取，如果没有，则直接新建一个实例**。
            * 处理配置项
                * 生成一个map<DisconfKey,List>
                    * DisconfKey 代表了一个配置项，或者配置文件，list则是针对这个配置的回调接口
            * 处理配置文件
                * 同上
            * 生成动态扫描模型类ScanDynamicModel，并将上面的map<DisconfKey,List<IDisconfUpdate>>映射设置进去 
            * 根据静态的pipeLine设置来，来设置动态扫描模型类pipeline，也是从注册表中获取。

        * transformUpdateService(scanDynamicModel.getDisconfUpdateServiceInverseIndexMap());
            * 更新进仓库中，该配置对应名字，必须之前存在（静态扫描）

        * transformPipelineService(scanDynamicModel.getDisconfUpdatePipeline());
            * 写入PipeLine仓库中

2. disconfCoreMgr.inject2DisconfInstance(); 
    注入数据至配置实体中，获取数据/注入/Watch。和之前的静态思路一模一样
    特殊的，将仓库里的数据注入到 配置项、配置文件的实体中

至此第二次动态扫描也已经完成。从上面看来，都是一个很流程的思维。如何做到及时更新配置文件的呢。重点在维持zk的连接上

---

### 分布式配置

在处理配置文件时，即更新配置文件时，会根据是否开远程disconf来监控路径，也就是：

            if (watchMgr != null) {
                watchMgr.watchPath(this, disConfCommonModel, fileName, DisConfigTypeEnum.FILE,
                        GsonUtils.toJson(disconfCenterFile.getKV()));
                LOGGER.debug("watch ok.");
            } else {
                LOGGER.warn("cannot monitor {} because watch mgr is null", fileName);
            }
这里将会监控路劲，是如何操作的呢，我们来看：

> |----disconf
        |----app1_version1_env1
                |----file
                        |----confA.properties
                |----item
                        |----keyA
        |----app2_version2_env2
                |----file
                        |----conf2.properties
                |----item
                        |----key2
这是一个zk的路径图，摘自官网。

    * 会产生一个永久节点，就是上面二级目录这里，并写入本机的IP，这样代表一个客户端路径
    * 然后创建三和四级目录，根据是分布式文件还是分布式配置项来看。
    * 最后根据DisClientComConfig这个前文提到过的，创建5级目录，这个目录就能代表该应用，同时是一个临时的的节点，
    存储的是disconfCenterFile或者disconfCenterItem序列化对象。

这里提到还是路径的创建。在路径创建完毕之后，会监控路径。路径到4级目录。最后作者以一个函数monitorMaster。不知道意图是什么。

这样当收到zk信息变更时，会收到通知。也就是NodeWatcher，重写的方法。

    * 当收到的事件是数据发生了改变，则会重新调用回调函数callback。最后会跟新配置，并调用注册回调函数列表，
    最后调用pipeline。作者在这上面写了注解  
    通用型的配置更新接口。当配置更新 时，用户可以实现此接口，用以来实现回调函数.
    这里用两种类型的回调函数 

        * IDisconfUpdate        针对某一个配置项，或者配置文件。
        * IDisconfUpdatePipeline 下面这种针对所用的配置项以及配置文件

至此disconf就完成了。至于xml配置形式。只是在spring的初始化过程中做了手脚，想要直接更新必须要使用回调接口。
而且作者已经不建议使用xml配置形式


