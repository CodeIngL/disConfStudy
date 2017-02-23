### disconf分布式配置管理

disconf是百度开源出来的一款关于分布式管理应用配置的开源软件，与某些著名开源软件一样。  
disconf也是某些模块的实现依赖zookeeper。现在一致服务的协调软件越来越多，zookeeper也面临很大的挑战
无论是etcd，是spring cloud采用的那个都是一致性强大的软件，选取合适的软件是值得的商榷的。尤其是CAP理论  


#### 一份demo入手

disconf的官方文档还是相当的完善，还提供了三个demo级别的例子。
不足的是disconf的文档显然是针对linux环境下，当然切成window下还是比较方便的。只要看了脚本稍微改一下，
当然你的window下有git等携带cgywin的另外说。

> disconf-standalone-demo 项目。工具Idea


这是一个spring的工程，我们之间上到main函数里面。

类**DisconfDemoMain**:

    /**
     * @param args
     *
     * @throws Exception
     */
    public static void main(String[] args) throws Exception {

        contextInitialized();
        ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext(fn);

        DisconfDemoTask task = ctx.getBean("disconfDemoTask", DisconfDemoTask.class);

        int ret = task.run();

        System.exit(ret);
    }

我们详细来说明下，首先是contextInitailized().这个基本没什么必要的的。指定了spring的xml路径 

接着就是spring同志熟悉的操作了。 作者就是在spring加载中切入这些动作的。  

根据作者的文档说。

    <!-- 使用disconf必须添加以下配置 -->
    <bean id="disconfMgrBean" class="com.baidu.disconf.client.DisconfMgrBean"
          destroy-method="destroy">
        <property name="scanPackage" value="com.example.disconf.demo"/>
    </bean>
    <bean id="disconfMgrBean2" class="com.baidu.disconf.client.DisconfMgrBeanSecond"
          init-method="init" destroy-method="destroy">
    </bean>

然后我们看一下这个两个bean的作用，首先指出的是这几个bean来自disconf的客户端包。

类DisconfMgrBean implements BeanDefinitionRegistryPostProcessor, PriorityOrdered, ApplicationContextAware: 
注意到细节这个类的实现都是Spring中重要的类。

我们关注是BeanDefinitionRegistryPostProcessor。这个是spring中特殊的处理器，在生成实例化bean之前，这个处理器能在进一步处理
BeanDefiniation。

上源码:
     public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {

        // 为了做兼容
        DisconfCenterHostFilesStore.getInstance().addJustHostFileSet(fileList);

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
    }

作者也写了一些注释。第一个是兼容的，我们就不看了。接下来就是比较简单了。一连串的的操作，就是去掉扫描包重复的。

接着就是进行扫描。我们来关注重点**firstScan()扫描包


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
    * 这里做了两部分，一是加载系统配置，二是加载用户配置。来看下源码
这里作者还提到了一个细节的地方   
    * **DisClientComConfig**类描述些通用的信息（提供了该类的标识:主机+端口+UUID(没有横杆)）这个对应用是单例的。这里先给自己说明下
    * DisClientSysConfig.getInstance().loadConfig(null);    
        这一步会加载某个路径下的disconf_sys.properties文件。默认是客户端包中的。
        这里会先尝试使用类加载器先进行先加载，如是web应用之间会使用tomcat下应用加载器，原因在于final。
        如果找不抛出异常。转入文件方式加载。最后把得到的prop注入DisClientSysConfig。这个注入就很简单了利用反射和注解。
        然后简单验证这些属性是否被注入。也就引出了一些必要的配置 
        
        * disconf.conf_server_store_action 仓库 URL  
        * disconf.conf_server_zoo_action  zoo URL    
        * disconf.conf_server_master_num_action 获取远程主机个数的URL  
        * disconf.local_download_dir  下载文件夹, 远程文件下载后会放在这里  

    * DisClientConfig.getInstance().loadConfig(null);  
        同上，这一步是加载用户的配置。配置文件默认是disconf.properties。可以使用命令行-d 进行导入  
        后面就跟上一步是一模一样的了。但是多了一步使用系桶参数的注入，就是某些参数可以不写在配置文件里面。
        同样和上面一样也有校验
        
        * disconf.conf_server_host 配置文件服务器 HOST  
            * disconf.version 版本  
            * disconf.app app名   
            * disconf.env 部署环境  
            * 不必要的：  
                * disconf.enable.remote.conf 是否从云端下载配置 默认false  
                * disconf.debug 是否开启DEBUG模式: 默认不开启，默认false  
                    * true: 用于线下调试，当ZK断开与client连接后（如果设置断点，这个事件很容易就发生），ZK不会去重新建立连接。  
                    * false: 用于线上，当ZK断开与client连接后，ZK会再次去重新建立连接。
                * disconf.user_define_download_dir 用户下载文件夹 默认./disconf/download 没有胡创建
                * disconf.ignore 忽略哪些分布式配置
                * disconf.conf_server_url_retry_times 获取远程配置 重试次数，默认是3次
                * disconf.conf_server_url_retry_sleep_seconds 获取远程配置 重试时休眠时间，默认是5秒
                * disconf.enable_local_download_dir_in_class_path 让下载文件夹放在 classpath目录 默认是
经过上面一步系统和用户的配置都会被载入完毕。  
----

2. getSpringRegistry（applicationContext）
    一个Spring的注册表，装饰了一个简单的注册表

3. ScanFactory.getScanMgr(registry);
    组装了一个扫描器
    构造函数是样的：

         public ScanMgrImpl(Registry registry) {

            this.registry = registry;

            // 配置文件
            staticScannerMgrList.add(StaticScannerMgrFactory.getDisconfFileStaticScanner());

            // 配置项
            staticScannerMgrList.add(StaticScannerMgrFactory.getDisconfItemStaticScanner());

            // 非注解 托管的配置文件
            staticScannerMgrList.add(StaticScannerMgrFactory.getDisconfNonAnnotationFileStaticScanner());
        }
    组装了一些列的处理器

4. scanMgr.firstScan(scanPackageList);
    扫描入库。这里是重点，我们要特别注意下了

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
        
    * 首先扫描包系统只有一个实现，使用放射扫描 里面也很简单构建了一个ScanStaticModel类型
        
        * 利用放射得到基本信息

            * 设置reflections 对象，
            * 设置disconfFileClassSet 存储带DisconfFile注解的类信息
            * 设置disconfFileItemMethodSet 存储带DisconfFileItem注解的方法信息
            * 设置disconfItemMethodSet 存储带DisconfItem注解的方法信息
            * 设置disconfActiveBackupServiceClassSet 存储带DisconfActiveBackupService注解的类信息
            * 设置disconfUpdateService 存储带DisconfUpdateService注解的类信息
            * 设置idisconfUpdatePipeline 存储实现IDisconfUpdatePipeline类
        
        * 将上面的基本信息进行整理 

            *　构造一个Map<disconfFileClass,disconfFileItemMethod>的映射。
            *　反射处理添加这些映射关系
            *  校验配置文件
            *  设置disconfFileItemMap 存储实现上面的映射关系



    * 增加非注解的配置 只是托管的配置文件，没有注入到类中 也就是提供了桩
    * 跟进作者说明就是放入仓库喽。默认的策略（三种）都会进行
        
        * 配置文件的静态扫描 StaticScannerFileMgrImpl
        * 配置项的静态扫描 StaticScannerItemMgrImpl
        * 非注解配置文件的扫描器 StaticScannerNonAnnotationFileMgrImpl

来看这三个策略首先是StaticScannerFileMgrImpl


        // 转换配置文件
        List<DisconfCenterBaseModel> disconfCenterFiles = getDisconfFiles(scanModel);
        DisconfStoreProcessorFactory.getDisconfStoreFileProcessor().transformScanData(disconfCenterFiles);

很简短，我们看一下，里面也很简单，就是从上面所说的映射设置disconfFileItemMap 取得每一个键值对组装成**DisconfCenterFile**
我们来看一下从一个键值对映射到对象的详细操作
    
    * 操作注解获得文件名， 
    * 获得配置文件目录
    * 文件类型
    * 获得通用的模型数据，包括 APP，版本，环境，Zookeeper上的URL表示DisConfCommonModel 并设置进去 
    * 设置远程地址URL
    * 遍历类的字段
    * 。。。。。



5. disconfCoreMgr = DisconfCoreFactory.getDisconfCoreMgr(registry);disconfCoreMgr.process();
    获取数据/注入/Watch，这里涉及的远程的模块了，默认配置开启后，才会从远端下载文件 

    * 生成一个抓取规则FetcherMgr getFetcherMgr()。获取抓取器实例，记得释放资源, 它依赖Conf模块
        
        * 这有一个http的规则，默认的抓取器，
        * 然后根据本机信息（DisClientConfig的信息（应用单例））

    * 开启监控，设置ZK的信息，还没连上。
    * 生成DisconfCoreMgrImpl对象，这是核心处理器

        * 添加配置文件的处理器
        * 添加配置项的处理器
6. disconfCoreMgr.process();    
    (第一次扫描时使用)获取远程的所有配置数据,注入到仓库中,Watch 配置


----

这样第一次扫描就完成了

回到postProcessBeanDefinitionRegistry方法下，接着就剩下registerAspect(registry);
 
 这个做了什么呢


    /**
     * register aspectJ for disconf get request
     */
    private void registerAspect(BeanDefinitionRegistry registry) {

        GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
        beanDefinition.setBeanClass(DisconfAspectJ.class);
        beanDefinition.setLazyInit(false);
        beanDefinition.setAbstract(false);
        beanDefinition.setAutowireCandidate(true);
        beanDefinition.setScope("singleton");

        registry.registerBeanDefinition("disconfAspectJ", beanDefinition);
    }
比较简单，不多说，注册一个bean定义


-----

### DisconfMgrBeanSecond的说明

上面说了DisconfMgrBean细节，现在我们来关注接着的重点，也就是作者让我们一定配置

init-mthod指向一个函数，我们看一下
    
    public void init() {

        DisconfMgr.getInstance().secondScan();
    }
超级简单，就是开启第二次扫描。这里指出的是作者在DisconfMgr这类已经指出了重要性

    /**
     * 总入口
     */
    public synchronized void start(List<String> scanPackageList) {

        firstScan(scanPackageList);

        secondScan();
    }

废话不多说，我们现在看看动态扫描干了什么事情呢
