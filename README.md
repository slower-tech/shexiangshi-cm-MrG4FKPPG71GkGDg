
# ***1***\|***0*****关于动态定时任务**


关于在SpringBoot中使用定时任务，大部分都是直接使用SpringBoot的**@Scheduled**注解，如下：




```
@Component
public class TestTask
{
    @Scheduled(cron="0/5 * *  * * ? ")   //每5秒执行一次
    public void execute(){
        SimpleDateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"); 
        log.info("任务执行" + df.format(new Date()));
    }
}
```


或者或者使用第三方的工具，例如XXL\-Job等。就XXL\-Job而言，如果说是大型项目，硬件资源和项目环境都具备，XXL\-Job确实是最佳的选择，可是对于项目体量不大，又不想过多的引入插件；使用XXL\-Job多少有点“杀鸡用牛刀”的意思；


之所以这样说，是因为在SpringBoot中集成使用XXL\-Job的步骤如下：


1. 引入依赖，配置执行器Bean
2. 在自己项目中编写定时任务代码
3. 部署XXL\-Job
4. 登录XXL\-Job调度中心的 Web 控制台，创建一个新的任务，选择刚才配置的执行器和任务处理器，设置好触发条件（如 Cron 表达式）和其他选项后保存
5. 生产环境下，还要把配置好任务的XXL\-Job和项目一起打包




```
@Component
public class SampleXxlJob {

    @XxlJob("sampleJobHandler")
    public void sampleJobHandler() throws Exception {
        // 业务逻辑
        System.out.println("Hello XXL-JOB!");
    }
}
```


这一套步骤在中小型项目中，明显成本大于效果，而使用XXL\-Job无非就是想动态的去管理定时任务，可以在运行状态下随意的执行、中断、调整执行周期、查看运行结果，而不是像基于**@Scheduled**注解实现后，无法改变。


所以，这里就基于SpringBoot实现动态调度定时任务，之前针对这个问题，我写过一篇CSDN文章（连接放在下方），最近博主将相关的代码进行了汇总，并且在易用性和扩展性上进行了加强，基于COLA架构理论，封装到了组件层


这次加强主要包括:


1. **剥离了任务的持久化，使其依赖更简洁，真正以starter的形式开箱即用**
2. **扩展了方法级的定时任务注册，能够像xxl\-job一样，一个注解搞定动态定时任务的注册及管理**


# ***2***\|***0*****动态定时任务实现思路**


关于动态定时任务的核心思路是**反射\+定时任务模板**，这两部分的核心框架不变，相关内容可以回顾我之前的博客：


[轻量级动态定时任务调度](https://github.com)


这里主要是针对强化的第二点进行思路解释，第二点的强化是加入了类扫描机制，通过扫描，实现了自动注册，规避了之前每新增一个定时任务都必须得预制SQL的步骤：


[![](https://img2024.cnblogs.com/blog/1368510/202411/1368510-20241122205309281-2016869323.png)](https://img2024.cnblogs.com/blog/1368510/202411/1368510-20241122205309281-2016869323.png)


**类级别定时任务实现思路**：在原模板模式的基础下，基于AbstractBaseCronTask类自定义的定时任务子类作为类级别定时任务，即一个类为一个定时任务，初始时由包扫描所有的子类，并使用反射将其实例化，逐一加入到进程管理中，并激活定时调度。


**基于@MethodJob的方法级别任务实现思路**：以 AbstractBaseCronTask类为基础，定义一个固定的子类BaseMethodLevelTask**，**并在其内部限定任务的执行方式**，**扫描所有标注了@MethodJob的方法及其所属的Bean，连同Bean及方法的反射类作为构造函数，生成BaseMethodLevelTask对象**，**因为BaseMethodLevelTask也是AbstractBaseCronTask的子类，则可以以类级别定时任务的方式，将其生成定时任务，并进行管理。


本质还是管理的AbstractBaseCronTask子类在线程池中的具体对象，不同的地方是类级别定时任务是一个具体的任务类仅生成一个对象，class路径即是唯一的标识，而方法级别的定时任务均基于BaseMethodLevelTask生成无数个对象，具体标识则是构造函数传入的Bean的反射对象和方法名。


对此部分感兴趣的可以一起参与开发，该项目地址：[Gitee源码](https://github.com)，主要为其中**task\-component**模块。


# ***3***\|***0*****组件使用方法**


根据Git地址，将源码down下，编译安装到本地私仓中，以Maven的形式引入即可，该组件遵循Spring\-starter规范，开箱即用。




```
        
            com.gcc.container.components
            task-component
            1.0.0
        
```


## ***3***\|***1*****Yaml配置说明**


给出一个yaml模板：




```
gcc-task:
  task-class-package : *
  data-file-path: /opt/myproject/task_info.json
```


对于task\-component的配置只有两个，**task\-class\-package**和**data\-file\-path**




| 属性 | 说明 | 是否必须 | 缺省值 |
| --- | --- | --- | --- |
| **task\-class\-package** | 指定定时任务存放的包路径，不填写或无此属性则默认全项目扫描，填写后可有效减少初始化时间 | 否 | \* |
| **data\-file\-path** | 定时任务管理的数据存放文件及其路径,默认不填写则存放项目运行目录下，如自行扩展实现，则该配置自动失效 | 否 | class:db/task\_info.json |


此处的两个配置项都包含默认值，所以不进行yml的`gcc-task`仍旧可以使用该组件


## ***3***\|***2*****新建定时任务**


对于该组件在进行定时任务的创建时，有两种，分别是类级定时任务和方法级定时任务，两者的区别在于是以类为基本的单位还是以方法为一个定时任务的单位，对于运行效果则并无区别，根据个人喜好，选择使用哪一种即可


### ***1***\|***0*****类级定时任务**


新增一个定时任务逻辑，则需要实现基类`AbstractBaseCronTask` ，并加入注解 `@ClassJob`


**ClassJob的参数如下**：




| 参数 | 说明 | 样例 |
| --- | --- | --- |
| cron | 定时任务默认的执行周期，仅在首次初始化该任务使用（**必填**） | 10 0/2 \* \* \* ? |
| desc | 任务描述，非必填 | 这是测试任务 |
| bootup | 是否开机自启动，**缺省值为 false** | false |


`cron` 属性仅在第一次新增该任务时提供一个默认的执行周期，必须填写，后续任务加载后，定时任务相关数据会被存放在文件或数据库中，此时则以文件或数据库中该任务的cron为主，代码中的注解则不会生效，如果想重置，则删除已经持久化的任务即可。


一个完整的Demo如下：




```
@TaskJob(cron = "10 0/2 * * * ?" ,desc = "这是一个测试任务",bootup = true)
public class TaskMysqlOne extends AbstractBaseCronTask {

    public TaskMysqlOne(TaskEntity taskEntity) {
        super(taskEntity);
    }

    @Override
    public void beforeJob() {
    }

    @Override
    public void startJob() {
    }

    @Override
    public void afterJob() {
    }
}
```


继承`AbstractBaseCronTask` 必须要实现携带TaskEntity参数的`构造函数`、`beforeJob()`、`startJob()` 、`afterJob()` 三个方法即可。**原则上这三个方法是规范定时任务的执行，实际使用，只需要把核心逻辑放在三个方法中任何一个即可**。


因定时任务类是非SpringBean管理的类，所以**在自定义的定时任务类内无法使用任何Spring相关的注解（如@Autowired）**，**但是却可以通过自带的**`getServer(Class className)`**方法来获取任何Spring上下文中的Bean**


例如，你有一个UserService的接口及其Impl的实现类，想在定时任务类中使用该Bean，则可以：




```
@TaskJob(cron = "10 0/2 * * * ?" ,desc = "这是一个测试任务",bootup = true)
public class TaskMysqlOne extends AbstractBaseCronTask {

    public TaskMysqlOne(TaskEntity taskEntity) {
        super(taskEntity);
    }

    @Override
    public void beforeJob() {
    }

    @Override
    public void startJob() {
        List names = getServer(UserService.class).searchAllUserName();
        //后续逻辑……
        //其他逻辑
    }

    @Override
    public void afterJob() {

    }
}
```


### ***1***\|***0*****方法级定时任务**


如果不想新建类，或者不想受限于**AbstractBaseCronTask**的束缚`，则可以像xxl-job定义定时任务一样，直接在某个方法上标注@MethodJob注解即可。`


@MethodJob的参数如下：




| 参数 | 说明 | 样例 |
| --- | --- | --- |
| cron | 定时任务默认的执行周期，仅在首次初始化该任务使用（**必填**） | 10 0/2 \* \* \* ? |
| desc | 任务描述，非必填 | 这是测试任务 |
| bootup | 是否开机自启动，**缺省值为 false** | false |


`通ClassJob一样，cron` 属性仅在第一次新增该任务时提供一个默认的执行周期，必须填写，后续任务加载后，定时任务相关数据会被存放在文件或数据库中，此时则以文件或数据库中该任务的cron为主，代码中的注解则不会生效，如果想重置，则删除已经持久化的任务即可。


下面是一个例子：




```
//正常定义的Service接口
public interface AsyncTestService {
    void  taskJob();

}

//Service接口实现类
@Service
public class AsyncTestServiceImpl implements AsyncTestService {

    @MethodJob(cron = "11 0/1 * * * ?",desc = "这是个方法级任务")
    @Override
    public void taskJob() {
        log.info("方法级别任务查询关键字为企业的数据");
        QueryWrapper query = new QueryWrapper<>();
        query.like("art_name","企业");
        List data = articleMapper.selectList(query);
        log.info("查出条数为{}",data.size());
    }
}
```


**注：该注解仅支持SpringBoot中标注为`@Component`、`@Service`、`@Repository`的Bean**


## ***3***\|***3*****动态调度任务接口说明**


定时任务相关的管理操作均封装在`TaskScheduleManagerService`接口中，接口内容如下：




```
public interface TaskScheduleManagerService {
    /**
     * 查询在用任务
     * @return
     */
    List searchTask(SearchTaskDto dto);
    /**
     * 查询任务详情
     * @param taskId 任务id
     * @return TaskEntity
     */
    TaskVo searchTaskDetail(String taskId);
    /**
     * 运行指定任务
     * @param taskId 任务id
     * @return TaskRunRetDto
     */
    TaskRunRetVo runTask(String taskId);
    /**
     * 关停任务
     * @param taskId 任务id
     * @return TaskRunRetDto
     */
    TaskRunRetVo shutdownTask(String taskId);
    /**
     * 开启任务
     * @param taskId 任务id
     * @return TaskRunRetDto
     */
    TaskRunRetVo openTask(String taskId);
    /**
     * 更新任务信息
     * @param entity 实体
     * @return TaskRunRetDto
     */
    TaskRunRetVo updateTaskBusinessInfo(TaskEntity entity);
}
```


可直接在外部项目中注入使用即可：




```
@RestController
@RequestMapping("/myself/manage")
public class TaskSchedulingController {
    
    //引入依赖后可直接使用注入
    @Autowired
    private TaskScheduleManagerService taskScheduleManagerService;

    @GetMapping("/detail")
    @Operation(summary = "具体任务对象")
    public Response searchDetail(String taskId){
        return Response.success(taskScheduleManagerService.searchTaskDetail(taskId));
    }

    @GetMapping("/shutdown")
    @Operation(summary = "关闭指定任务")
    public Response shutdownTask(String taskId){
        return Response.success(taskScheduleManagerService.shutdownTask(taskId));
    }

    @GetMapping("/open")
    @Operation(summary = "开启指定任务")
    public Response openTask(String taskId){
        return Response.success(taskScheduleManagerService.openTask(taskId));
    }
}
```


接口效果：


[![](https://img2024.cnblogs.com/blog/1368510/202411/1368510-20241122212724565-1699842245.png)](https://img2024.cnblogs.com/blog/1368510/202411/1368510-20241122212724565-1699842245.png)


 


可使用该接口，进行UI页面开发。


## ***3***\|***4*****关于任务持久化的扩展**


在实现思路中提到过，task\-component的执行原理需要将注册后的任务持久化，下次再启动项目时，则直接使用持久化的TaskEntity来加载定时任务。


考虑到定时任务的数量不大，且对于交互要求不高，另外考虑封装成组件的**独立性**和**普适性**，不想额外引入数据库依赖和ORM框架，所以组件默认的是以JSON文件的形式进行存储，实际使用中，考虑到便利性，可以自行对持久化部分进行扩展。


所谓持久化，其实本制是持久化TaskEntity对象，TaskEntity对象如下：




```
@Data
public class TaskEntity implements Serializable {

    /**
    * 任务ID（唯一）
    */
    private String taskId;

    /**
    * 任务名称
    */
    private String taskName;

    /**
    * 任务描述
    */
    private String taskDesc;

    /**
    * 遵循cron 表达式
    */
    private String taskCron;

    /**
     * 类路径
     */
    private String taskClass;

     /**
     * 任务级别  CLASS_LEVEL 类级别，METHOD_LEVEL 方法级别
     */
    private TaskLevel taskLevel;

    /**
     * 任务注册时间
     */
    private String taskCreateTime;
    /**
     * 是否启用，1启用，0不启用
     */
    private Integer taskIsUse;

    /**
     * 是否系统启动后立刻运行 1是。0否
     */
    private Integer taskBootUp;

    /**
     * 上次运行状态 1：成功，0：失败
     */
    private Integer taskLastRun;
    /**
     * 任务是否在内存中 1:是，0：否
     */
    private Integer taskRamStatus;

     /**
     * 外部配置 （扩展待使用字段）
     */
    private String taskOutConfig;

    /**
     * 加载配置
     */
    private String loadConfigure;
}
```


扩展的主要操作为两步：


1. **新建存放taskEntity的表**
2. **实现TaskRepository接口**


首先新建数据库表，这里以Mysql为例给出建表语句：




```
DROP TABLE IF EXISTS `tb_task_info`;
CREATE TABLE `tb_task_info` (
  `task_id` varchar(100) NOT NULL PRIMARY KEY ,
  `task_name` varchar(255) DEFAULT NULL,
  `task_desc` text,
  `task_cron` varchar(20) DEFAULT NULL,
  `task_class` varchar(100) DEFAULT NULL COMMENT '定时任务类路径',
  `task_level` varchar(50) DEFAULT NULL COMMENT '任务级别：类级别（CLASS_LEVEL）、方法级别（METHOD_LEVEL）',
  `task_is_use` tinyint DEFAULT NULL COMMENT '是否启用该任务，1：启用，0禁用',
  `task_boot_up` tinyint DEFAULT NULL COMMENT '是否为开机即运行，1:初始化即运行，0，初始化不运行',
  `task_out_config` text CHARACTER SET utf8mb3 COLLATE utf8mb3_general_ci COMMENT '定时任务额外配置项,采用json结构存放',
  `task_create_time` varchar(255) CHARACTER SET utf8mb3 COLLATE utf8mb3_general_ci DEFAULT NULL COMMENT '定时任务追加时间',
  `task_last_run` tinyint DEFAULT NULL COMMENT '任务上次执行状态；1正常，0执行失败,null未知',
  `task_ram_status` tinyint DEFAULT NULL COMMENT '任务当前状态；1内存运行中，0内存移除',
  `loadConfigure` text  COMMENT '加载相关配置',  
PRIMARY KEY (`task_id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb3 ROW_FORMAT=DYNAMIC;
```


实现**TaskEntityRepository**接口，接口内容为：




```
public interface TaskEntityRepository {
    /**
     * 新增数据
     * @param entity
     * @return int
     */
    int save(TaskEntity entity);
    /**
     * 删除任务，根据TaskId
     * @param id id
     * @return int
     */
    int removeByTaskId(String id);
    
     /**
     * 更新任务(除taskId外，所有字段必须更新)
     * @param entity 实体
     * @return int
     */
    int update(TaskEntity entity);
    
    /**
     * 查询全部任务
     * @return List
     */
    List queryAllTask();

}
```


 


只需要新建类，然后实现该接口即可，如下是基于Mybatis\-Plus进行的实现样例：




```
@Mapper
public interface TaskDataMapper extends BaseMapper {
}

@Repository
//此处一定要用@Primary注解进行固定
@Primary
public class TaskEntityReponsitoryImpl implements TaskEntityRepository {

    @Autowired
    private TaskDataMapper taskDataMapper;

    @Override
    public int save(TaskEntity entity) {
        entity.setTaskCreateTime(DateUtil.now());
        return taskDataMapper.insert(entity);
    }
    @Override
    public int removeByTaskId(String id) {
        QueryWrapper query = new QueryWrapper<>();
        query.eq("task_id",id);
        return taskDataMapper.delete(query);
    }
    @Override
    public int update(TaskEntity entity) {
        UpdateWrapper update = new UpdateWrapper<>();
        update.eq("task_id",entity.getTaskId());
        return taskDataMapper.update(entity,update);
    }
    @Override
    public List queryAllTask() {
        return taskDataMapper.selectList(new QueryWrapper<>());
    }
}
```


 


至此，项目中则可以以数据库表的形式来管理定时任务：


[![](https://img2024.cnblogs.com/blog/1368510/202411/1368510-20241122211012859-1565155665.png)](https://img2024.cnblogs.com/blog/1368510/202411/1368510-20241122211012859-1565155665.png):[wgetCloud机场](https://longdu.org)


 


任务运行日志：




```
[main] com.web.test.Application                 : Started Application in 3.567 seconds (JVM running for 4.561)
[main] .g.c.c.t.c.InitTaskSchedulingApplication : 【定时任务初始化】 容器初始化
[main] o.s.s.c.ThreadPoolTaskScheduler          : Initializing ExecutorService
[main] .g.c.c.t.c.InitTaskSchedulingApplication : 【定时任务初始化】定时任务初始化任务开始
[main] c.g.c.c.task.compont.TaskScheduleImpl    : 【定时任务初始化】装填任务:TaskTwoPerson [ 任务执行周期：15 0/2 * * * ? ] [ bootup：0]
[main] c.g.c.c.task.compont.TaskScheduleImpl    : 【定时任务初始化】装填任务:TaskMysqlOne [ 任务执行周期：10 0/2 * * * ? ] [ bootup：1]
[main] c.g.c.c.task.compont.TaskScheduleImpl    : 【定时任务初始化】装填任务:taskJob [ 任务执行周期：11 0/1 * * * ? ] [ bootup：0]
[main] .g.c.c.t.c.InitTaskSchedulingApplication : 【定时任务初始化】定时任务初始化任务完成
[main] c.g.c.c.task.service.AfterAppStarted     : 【定时任务自运行】运行开机自启动任务
[main] TaskMysqlOne                             : ---------------------任务 TaskMysqlOne 开始执行-----------------------
[main] TaskMysqlOne                             : 任务描述：这是一个测试任务
[main] TaskMysqlOne                             : 我是张三
[main] TaskMysqlOne                             : 任务耗时：约 0.0 s
[main] TaskMysqlOne                             : ---------------------任务 TaskMysqlOne 结束执行-----------------------

[task-thread-1] taskJob                                  : ---------------------任务 taskJob 开始执行-----------------------
[task-thread-1] taskJob                                  : 任务描述：这是个方法级任务
[task-thread-1] c.w.t.service.impl.AsyncTestServiceImpl  : 方法级别任务查询关键字为企业的数据
[task-thread-1] c.w.t.service.impl.AsyncTestServiceImpl  : 查出条数为1
[task-thread-1] taskJob                                  : 任务耗时：约 8.45 s
[task-thread-1] taskJob                                  : ---------------------任务 taskJob 结束执行-----------------------
```


 


\_\_EOF\_\_

![](https://github.com/TheGCC/p/18563778.html)Deep Blue 本文链接：[https://github.com/TheGCC/p/18563778\.html](https://github.com)关于博主：评论和私信会在第一时间回复。或者[直接私信](https://github.com)我。版权声明：本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA") 许可协议。转载请注明出处！声援博主：如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。您的鼓励是博主的最大动力！
