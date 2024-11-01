---
layout: post
title: "基于Spring的Quartz定时任务"
date:   2024-11-01
tags: [场景]
comments: true
author: zy0410
---
## 定时任务实现方式(Quartz)

> 1、开发定时任务类

定时任务类继承[BaseTimer]基础类, 实现[execute]方法, 在该方法中执行业务逻辑, 并返回业务执行结果

```java
@Data
@Slf4j
public abstract class BaseTimer implements Runnable {

    private BusiScheduleJob job;
    
    //对job表进行增删改查，改变任务执行的状态
    @Resource
    private BusiShcedualJobService busiShcedualJobService;

    /**
     * 执行业务逻辑
     */
    public abstract TimerResult execute(BusiScheduleJob job);

    /**
     * 定时任务执行的方法
     */
    @Override
    public final void run() {
        // 1. 执行前更新定时状态
        int exeState = TimerResult.ResultEnum.EXEC.getState();
        String execMsg = TimerResult.ResultEnum.EXEC.getMsg();
        BusiScheduleJob param = new BusiScheduleJob()
                .setId(job.getId())
                .setExeState(exeState)
                .setExeMsg(execMsg)
                .setLastTime(new Date());
        this.busiShcedualJobService.updateBusiScheduleJobById(param);
        // 2. 调用方法执行
        try {
            // 2.1 执行业务逻辑
            TimerResult result = this.execute(job);
            // 2.2 处理结果
            if (result != null) {
                exeState = result.getExeState();
                execMsg = result.getExeMsg();
            }
            else {
                exeState = TimerResult.ResultEnum.SUCC.getState();
                execMsg = TimerResult.ResultEnum.SUCC.getMsg();
            }
        }
        catch (Exception e) {
            log.error("定时任务[{}]执行失败", job.getBeanName(), e);
            exeState = TimerResult.ResultEnum.FAIL.getState();
            execMsg = TimerResult.ResultEnum.FAIL.getMsg();
        }
        // 3. 执行完后更新执行状态
        param.setExeState(exeState)
                .setExeMsg(execMsg)
                .setLastTime(new Date());
        this.busiShcedualJobService.updateBusiScheduleJobById(param);
    }

}
```

```java
/**
 * 表名: cmp_extends_guangdong.BUSI_SCHEDULE_JOB
 * 任务调度作业表
 */
@Data
@Accessors(chain = true)
public class BusiScheduleJob {
    /**
     * 主键ID
     */
    private Integer id;

    /**
     * 任务状态 0:停止 1:启动 -1:删除
     */
    private Integer jobState;

    /**
     * 最新执行状态 -1:初始状态 0:执行失败 1:执行成功 2:执行中
     */
    private Integer exeState;

    /**
     * 执行信息
     */
    private String exeMsg;

    /**
     * 业务类型
     */
    private String busiType;

    /**
     * 操作用户编码
     */
    private String userCode;

    /**
     * 操作用户名称
     */
    private String userName;

    /**
     * CRON表达式, 执行频率
     */
    private String cronExpr;

    /**
     * 开始执行时间
     */
    private Date beginTime;

    /**
     * 任务结束时间
     */
    private Date endTime;

    /**
     * 最新执行时间
     */
    private Date lastTime;

    /**
     * 执行的服务,如: guangdong-service, iom-guangdong
     */
    private String exeService;

    /**
     * 业务类名称
     */
    private String beanName;

    /**
     * 任务描述
     */
    private String jobDesc;

    /**
     * 预置参数 varchar(256)
     */
    private String param1;

    /**
     * 预置参数 text
     */
    private String param2;

    /**
     * 创建时间
     */
    private Date createTime;

    /**
     * 非表字段
     * 查询频次 15分钟、30分钟、45分钟、60分钟
     */
    private Integer frequency;

}
```

```java
public interface BusiShcedualJobService {

    /**
     * 启动定时任务
     * @param job 任务详情信息
     * @return
     */
    boolean startScheduleJob(BusiScheduleJob job);


    /**
     * 停止定时任务
     * @param job 任务详情信息
     * @return
     */
    boolean stopScheduleJob(BusiScheduleJob job);

    /**
     * 更新任务调度信息列表
     * @param param 参数,id不能为空
     * @return 1:成功 0:失败
     */
    int updateBusiScheduleJobById(BusiScheduleJob param);

    /**
     * 获取任务调度信息列表
     * @param param 参数
     * @return 任务调度信息列表
     */
    List<BusiScheduleJob> getBusiScheduleJobs(BusiScheduleJob param);

    /**
     * 获取任务调度信息
     * @param jobId 任务ID
     * @return 任务调度信息
     */
    BusiScheduleJob getBusiScheduleJobById(Integer jobId);

}
```

> 定时任务配置

```sql
  INSERT INTO cmp_extends_guangdong.busi_schedule_job
  (JOB_STATE, EXE_STATE, EXE_MSG, BUSI_TYPE, CRON_EXPR, BEGIN_TIME, EXE_SERVICE, BEAN_NAME, JOB_DESC, CREATE_TIME)
  VALUES (0, -1, '', 'system', '0 0/7 * * * ?', now(), 'rm-guangdong', 'scanExpiredEcsTimer', '扫描到期的云主机记录', now());
```

  JOB_STATE   '任务状态 0:停止 1:启动 -1:删除',
  EXE_STATE   '最新执行状态 -1:初始状态 0:执行失败 1:执行成功 2:执行中',
  BUSI_TYPE   '业务类型 默认填: system',
  CRON_EXPR   'CRON表达式, 执行频率',
  BEGIN_TIME  '开始执行时间',
  EXE_SERVICE  '执行的服务,如: guangdong-service, iom-guangdong, rm-guangdong',
  BEAN_NAME   '业务类名称',
  JOB_DESC   '任务描述'

> 定时任务守护类

```java
/**
 * 守护定时任务-始终启动
 * 作用: 扫描定时任务, 处理定时任务的停止/启动等操作
 */
@Slf4j
public class GuardTimer extends BaseTimer {
    @Setter
    // 定时运行服务
    private String jobExeService;
    @Setter
    private String cronExpress;
    @Setter
    private BusiShcedualJobService busiShcedualJobService;
    @Setter
    private RedisTemplate redisTemplate;
    // 运行的任务
    private static final Map<String, BusiScheduleJob>
            jobContainer = new ConcurrentHashMap<>();
    //0:已停止
    public static final Integer STATUS_STOP = 0;
    //1:运行中
    public static final Integer STATUS_RUNNING = 1;
    // 生成一个ID标识
    private final String guardId = IdUtil.objectId();


    // 定时轮询询检查定时任务
    @Override
    public TimerResult execute(BusiScheduleJob job) {
        // 查找guangdong-service下的定时任务
        Date currDate = new Date();
        job = new BusiScheduleJob()
                .setExeService(jobExeService)
                .setBeginTime(currDate)
                .setEndTime(currDate);
        List<BusiScheduleJob> scheduleJobs = busiShcedualJobService.getBusiScheduleJobs(job);

        // 扫描处理定时任务
        List<String> jobNames = new ArrayList<>();
        for (BusiScheduleJob scheduleJob : scheduleJobs) {
            String jobName = scheduleJob.getBeanName() + scheduleJob.getId();
            jobNames.add(jobName);
            if (STATUS_STOP.equals(scheduleJob.getJobState())) {
                // 停止定时任务
                stopJob(jobName, scheduleJob);
            }
            if (STATUS_RUNNING.equals(scheduleJob.getJobState())) {
                // 启动定时任务
                startJob(jobName, scheduleJob);
            }
        }

        // 处理已经失效的定时任务
        jobContainer.forEach((jobName, scheduleJob) -> {
            String key = getPrefixKey() + jobName;
            String val = RedisUtil.get(key, redisTemplate);
            if (!jobNames.contains(jobName)
                    || (val != null && !val.contains(guardId))) {
                // 停止定时任务
                stopJob(jobName, scheduleJob);
            }
        });

        return TimerResult.succ();
    }


    // 构建时启动
    public void start() {
        new Thread() {
            @Override
            @SneakyThrows
            public void run() {
                TimeUnit.SECONDS.sleep(15);
                BusiScheduleJob scheduleJob = new BusiScheduleJob();
                scheduleJob.setId(0);
                scheduleJob.setBeanName("guardTimer");
                scheduleJob.setCronExpr(cronExpress);
                scheduleJob.setExeService(jobExeService);
                boolean succ = busiShcedualJobService.startScheduleJob(scheduleJob);
                log.info(">>> 守护定时任务启动: {}", succ);
            }
        }.start();
    }

    // 启动定时任务
    private void startJob(String jobName, BusiScheduleJob scheduleJob) {
        String key = getPrefixKey() + jobName;
        String val = RedisUtil.get(key, redisTemplate);
        boolean succ = false;
        if (StrUtil.isBlank(val)) {
            // 初次占锁
            succ = RedisUtil.setnx(key, guardId, 50, redisTemplate);
        }
        else if (val.contains(guardId)) {
            // 是当前的服务的定时任务则续锁
            RedisUtil.set(key, guardId, redisTemplate, 50);
        }

        if (succ && !jobContainer.containsKey(jobName)) {
            boolean rs = busiShcedualJobService.startScheduleJob(scheduleJob);
            log.info("定时任务[{}] 启动结果: {}", jobName, rs);
            if (rs) {
                jobContainer.put(jobName, scheduleJob);
            }
            else {
                // 没启动成功 删掉标识
                RedisUtil.del(key, redisTemplate);
            }
        }
    }


    // 停止定时任务
    private void stopJob(String jobName, BusiScheduleJob scheduleJob) {
        if (jobContainer.containsKey(jobName)) {
            boolean rs = busiShcedualJobService.stopScheduleJob(scheduleJob);
            log.info("定时任务[{}] 停止结果: {}", jobName, rs);
            jobContainer.remove(jobName);
            String key = getPrefixKey() + jobName;
            RedisUtil.del(key, redisTemplate);
        }
    }

    private String getPrefixKey() {
        return jobExeService + ":timer:";
    }
}
```

```java
@Slf4j
public abstract class GuardTimerConfig {


    /**
     * 获取定时守护任务 cron表达式
     * @return cron表达式
     */
    public abstract String getCronExpress();

    /**
     * 获取定时任务服务标识
     * @return 定时任务服务标识
     */
    public abstract String getJobExeService();


    // 工厂类
    @Bean
    public StdSchedulerFactory initStdSchedulerFactory(
            QuartzProperties quartzProperties) throws SchedulerException {
        Map<String, String> properties = quartzProperties.getProperties();
        log.info(">>> 定时任务工厂类初始化配置参数:{}", JSONUtil.toJsonStr(properties));
        StdSchedulerFactory stdSchedulerFactory = new StdSchedulerFactory();
        Properties prop = new Properties();
        prop.putAll(properties);
        stdSchedulerFactory.initialize(prop);
        return stdSchedulerFactory;
    }

    // 定时任务管理类
    @Bean
    public BusiShcedualJobService busiShcedualJobService(
            BusiScheduleJobDao busiScheduleJobDao, StdSchedulerFactory schedulerFactory) {
        BusiShcedualJobServiceImpl shcedualJobService = new BusiShcedualJobServiceImpl();
        shcedualJobService.setBusiScheduleJobDao(busiScheduleJobDao);
        shcedualJobService.setSchedulerFactory(schedulerFactory);
        return shcedualJobService;
    }

    // 定时任务守护类
    @Bean
    public GuardTimer guardTimer(RedisTemplate redisTemplate,
                                 BusiShcedualJobService busiShcedualJobService) {
        GuardTimer guardTimer = new GuardTimer();
        guardTimer.setRedisTemplate(redisTemplate);
        guardTimer.setBusiShcedualJobService(busiShcedualJobService);
        guardTimer.setCronExpress(getCronExpress());
        guardTimer.setJobExeService(getJobExeService());
        guardTimer.start();
        return guardTimer;
    }

}
```

```java
@Slf4j
@Configuration
public class RmGuardTimerConfig extends GuardTimerConfig {

    // 检查周期
    private static final String CRON_EXPRESS = "*/23 * * * * ?";
    // 服务标识
    private static final String JOB_EXE_SERVICE = "rm-guangdong";


    @Override
    public String getCronExpress() {
        return CRON_EXPRESS;
    }

    @Override
    public String getJobExeService() {
        return JOB_EXE_SERVICE;
    }
}
```

