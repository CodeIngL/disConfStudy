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
tip：**PriorityOrdered**不是必要的。

---
### postProcessBeanDefinitionRegistry（BeanDefinitionRegistry)
上源码:

        // 为了做兼容
        DisconfCenterHostFilesStore.getInstance().addJustHostFileSet(fileList);

        //从scanPackage字符串获得包名（去重），分隔符为“，”。
        List<String> scanPackList = StringUtil.parseStringToStringList(scanPackage, SCAN_SPLIT_TOKEN);
        // unique
        Set<String> hs = new HashSet<String>();
        hs.addAll(scanPackList);
        scanPackList.clear();
        scanPackList.addAll(hs);

        // 进行扫描
        DisconfMgr.getInstance().setApplicationContext(applicationContext);
        DisconfMgr.getInstance().firstScan(scanPackList);

        // register java bean
        registerAspect(registry);

作者在实现上做了注释。我也添加了注释，应该很清晰。我们来关注我们所需的重点。  
这里我们首先引入一个类**DisconfMgr**。这个类是应用单例的，
按着作者所说，这个是disconf 客户端的总入口。

**重点之firstScan(packageNameList)**  



方法源码：

     /**
     * 第一次扫描，静态扫描 for annotation config
     */
    protected synchronized void firstScan(List<String> scanPackageList) {

        // 该函数不能调用两次。。。省略

        try {

            // 导入配置
            ConfigMgr.init();

            LOGGER.info("******************************* DISCONF START FIRST SCAN *******************************");

            // registry
            Registry registry = RegistryFactory.getSpringRegistry(applicationContext);

            // 扫描器
            scanMgr = ScanFactory.getScanMgr(registry);

            // 第一次扫描并入库
            scanMgr.firstScan(scanPackageList);

            // 获取数据/注入/Watch
            disconfCoreMgr = DisconfCoreFactory.getDisconfCoreMgr(registry);
            disconfCoreMgr.process();

            //
            isFirstInit = true;

            LOGGER.info("******************************* DISCONF END FIRST SCAN *******************************");

        } catch (Exception e) {

            LOGGER.error(e.toString(), e);
        }
    }

作者文档描述还是比较详细的。 我们直接看try这部分


1. ConfigMgr.init()
    
    这里做了两部分，一是加载系统配置，二是加载用户配置。来看下源码
这里作者还提到了一个细节的地方。这里设计到了4个类，其中三个是应用单例。  
简要的说明设计到4个类
    
    * DisClientComConfig   通用配置类 单例
    * DisClientSysConfig   系统配置类 单例
    * DisClientConfig      用户配置类 单例
    * DisInnerConfigHelper 配置校验工具类

    ----
    下面进行具体的说明。

    * DisClientComConfig.getInstance().getInstanceFingerprint()  
        DisClientComConfig类描述些通用的信息（提供了该类的标识:主机+端口(默认0）+UUID），上面返回信息。
    * DisClientSysConfig.getInstance().loadConfig(null);   
        这一步会加载disconf_sys.properties(存储在客户端jar中)，然后委托给工具类**DisconfAutowareConfig**注入系统配置单例。
        这里会使用类加载的URL方式和通俗文件方式，获取配置文件，前者可以避免缓存。优先使用URL，没有则进入普通方式。
        最后把得到的prop配置对象注入系统配置类（DisClientSysConfig）。。。这个注入就很简单了利用反射和注解。
    * DisInnerConfigHelper.verifySysConfig();  
        对上面prop配置对象注入后，进行一些配置校验，主要有这些(可以修改):

        * disconf.conf_server_store_action 仓库 URL  默认/api/config
        * disconf.conf_server_zoo_action  zk URL    默认/api/zoo
        * disconf.conf_server_master_num_action 获取远程主机个数的URL  默认//api/getmasterinfo 
        * disconf.local_download_dir  下载文件夹, 远程文件下载后会放在这里  默认./disconf/download

    * DisClientConfig.getInstance().loadConfig(null);  
        同上，这一步是加载用户的配置。配置文件默认是disconf.properties。这里提供了扩张，
        可以使用命令行-d 进行导入ex：-D  disconf.conf=xxxx
        后面的步骤是一模一样的了。但是多了一步使用系统参数（优先）的注入，就是某些参数可以不写在配置文件里面。
        同样和上面一样也有校验。
        
        * disconf.conf_server_host 配置文件服务器 HOST（web管理界面的地址） 
        * disconf.version 版本  默认值（DEFAULT_VERSION）
        * disconf.app app名   
        * disconf.env 部署环境  默认值（DEFAULT_ENV）
        * 不必要的：  
            * disconf.enable.remote.conf 是否从云端下载配置 默认false  
            * disconf.debug 是否开启DEBUG模式: 默认不开启，默认false  
                * true: 用于线下调试，当ZK断开与client连接后（如果设置断点，这个事件很容易就发生），ZK不会去重新建立连接。  
                * false: 用于线上，当ZK断开与client连接后，ZK会再次去重新建立连接。
            * disconf.user_define_download_dir 用户下载文件夹 默认./disconf/download 没有胡创建
            * disconf.ignore 忽略哪些分布式配置 默认空
            * disconf.conf_server_url_retry_times 获取远程配置 重试次数，默认是3次
            * disconf.conf_server_url_retry_sleep_seconds 获取远程配置 重试时休眠时间，默认是5秒
            * disconf.enable_local_download_dir_in_class_path 让下载文件夹放在 classpath目录  

        经过上面一步系统和用户的配置都会被载入完毕。

    至此第一步完成。

2. getSpringRegistry（applicationContext）  
    一个SpringRegistry的注册表，装饰了一个简单的注册表**simpleRegistry**
    目的是为了获得bean也就是，simpleRegistry基本是个软肋，主要还是直接操spring的applicationContext

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
   第一次扫描并入库，作者是这样说明的，这里是我们要考虑的重点。然我们看起函数内部实现。

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
            * 设置disconfFileClassSet 存储带DisconfFile注解的类信息
            * 设置disconfFileItemMethodSet 存储带DisconfFileItem注解的方法信息
            * 设置disconfItemMethodSet 存储带DisconfItem注解的方法信息
            * 设置disconfActiveBackupServiceClassSet 存储带DisconfActiveBackupService注解的类信息
            * 设置disconfUpdateService 存储带DisconfUpdateService注解的类信息
            * 设置idisconfUpdatePipeline 存储实现IDisconfUpdatePipeline类
        
        * 接着将上面的基本信息进行整理 

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
        **DisconfCenterFile**是分布式配置文件的内存对象表示。然后将该对象和文件名射进单例配置仓库中DisconfCenterStore

         ### StaticScannerItemMgrImpl配置项的静态扫描
           public void scanData2Store(ScanStaticModel scanModel) {
                // 转换配置文件
                List<DisconfCenterBaseModel> disconfCenterFiles = getDisconfFiles(scanModel);
                DisconfStoreProcessorFactory.getDisconfStoreFileProcessor().transformScanData(disconfCenterFiles);
            }

        同上，把ScanStaticModel中关于配置项的disconfItemMethodSet中取得每一个键值对组装成**DisconfCenterItem**类
        **DisconfCenterItem**是分布式配置项的内存对象表示。然后同上射进配置仓库中DisconfCenterStore

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
            * 验证带该注解的类是否实现回调接口**IDisconfUpdate**
            * 处理配置项
                * 生成一个map<DisconfKey,List>
                    * DisconfKey 代表了一个配置项，或者配置文件，list则是针对这个的回调接口
            * 处理配置文件
                * 同上
            * 生成动态扫描模型类，并将上面的map设置进去 
            * 更加今天扫描模型类，来设置update pipeline
        * transformUpdateService(scanDynamicModel.getDisconfUpdateServiceInverseIndexMap());
            * 写入仓库中
        * transformPipelineService(scanDynamicModel.getDisconfUpdatePipeline());
            * 同上
2. disconfCoreMgr.inject2DisconfInstance();
注入数据至配置实体中，获取数据/注入/Watch。和之前的静态思路一模一样
特殊的，将仓库里的数据注入到 配置项、配置文件 的实体中
