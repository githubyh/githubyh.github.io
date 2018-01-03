# Myth源码解析系列之（七）- 订单下单流程源码解析（参与者）
前面一章我们走完了订单下单流程发起者部分的源码，这次我们进入参与者部分源码解析~

### 订单下单流程源码解析（参与者）
前面order服务中已经发起了对account服务的调用，接下来进入account服务扣款接口的实现部分
```java
  //order服务调用端
  @PostMapping("/account-service/account/payment")
  @Myth(destination = "account", target = AccountService.class)
  Boolean payment(@RequestBody AccountDTO accountDO);

 //account服务接口实现 AccountServiceImpl.payment(AccountDTO accountDTO)
 @Override
    @Myth(destination = "account")
    public boolean payment(AccountDTO accountDTO) {
        LOGGER.info("============springcloud执行付款接口===============");
        final AccountDO accountDO = accountMapper.findByUserId(accountDTO.getUserId());
        if (accountDO.getBalance().compareTo(accountDTO.getAmount()) <= 0) {
            throw new MythRuntimeException("spring cloud account-service 资金不足！");
        }
        accountDO.setBalance(accountDO.getBalance().subtract(accountDTO.getAmount()));
        accountDO.setUpdateTime(new Date());
        final int update = accountMapper.update(accountDO);
        if (update != 1) {
            throw new MythRuntimeException("spring cloud account-service 资金不足！");
        }
        return Boolean.TRUE;
    }

 ```
我们发现在实现类方法头部也进行了@Myth注解的标记，AccountServiceImpl是一个实现类，因此这里必然也会走aop切面,aop切面流程入口同order服务相同，区别在于order为发起方，而account，inventory为参与者，我们是否还记得角色判断代码实现部分？MythTransactionFactoryServiceImpl.factoryOf 我们再来回顾下代码
```java
public Class factoryOf(MythTransactionContext context) throws Throwable {
        //如果事务还没开启或者 myth事务上下文是空， 那么应该进入发起调用
        if (!mythTransactionManager.isBegin() && Objects.isNull(context)) {
            return StartMythTransactionHandler.class;
        } else {
            if (context.getRole() == MythRoleEnum.LOCAL.getCode()) {
                return LocalMythTransactionHandler.class;
            }
            return ActorMythTransactionHandler.class;
        }
    }

```
判断条件要想进入参与者角色分支，这里事务必须开启状态 或者 myth事务上下文必须有值 ，这两个条件又是在哪里进行了设值呢？ 我们往回看看调用处，找到<b>SpringCloudMythTransactionInterceptor.interceptor(ProceedingJoinPoint pjp)</b>方法
```java
@Override
    public Object interceptor(ProceedingJoinPoint pjp) throws Throwable {
        MythTransactionContext mythTransactionContext = TransactionContextLocal.getInstance().get();
        if (Objects.nonNull(mythTransactionContext) &&
                mythTransactionContext.getRole() == MythRoleEnum.LOCAL.getCode()) {
            mythTransactionContext = TransactionContextLocal.getInstance().get();
        } else {
            RequestAttributes requestAttributes = RequestContextHolder.currentRequestAttributes();
            HttpServletRequest request = requestAttributes == null ? null : ((ServletRequestAttributes) requestAttributes).getRequest();
            String context = request == null ? null : request.getHeader(CommonConstant.MYTH_TRANSACTION_CONTEXT);
            if (StringUtils.isNoneBlank(context)) {
                mythTransactionContext =
                        GsonUtils.getInstance().fromJson(context, MythTransactionContext.class);
            }
        }
        return mythTransactionAspectService.invoke(mythTransactionContext, pjp);
    }
```
因为第一次进来，显然<b>mythTransactionContext</b>值为空，进入else分支，这里我们发现是从request请求头中获取的事务上下文信息的。 既然是从请求头信息中拿到数据， 那必然在调用端要先设置对不对， 我们找到<b>myth-springcloud工程下MythRestTemplateInterceptor</b>类
```java

//springcloud
@Configuration
public class MythRestTemplateInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate requestTemplate) {
        final MythTransactionContext mythTransactionContext =
                TransactionContextLocal.getInstance().get();
        requestTemplate.header(CommonConstant.MYTH_TRANSACTION_CONTEXT,
                GsonUtils.getInstance().toJson(mythTransactionContext));
    }

}

// motan
@Component
public class MotanMythTransactionInterceptor implements MythTransactionInterceptor {

    private final MythTransactionAspectService mythTransactionAspectService;

    @Autowired
    public MotanMythTransactionInterceptor(MythTransactionAspectService mythTransactionAspectService) {
        this.mythTransactionAspectService = mythTransactionAspectService;
    }

    @Override
    public Object interceptor(ProceedingJoinPoint pjp) throws Throwable {
        MythTransactionContext mythTransactionContext = null;

        final Request request = RpcContext.getContext().getRequest();
        if (Objects.nonNull(request)) {
            final Map<String, String> attachments = request.getAttachments();
            if (attachments != null && !attachments.isEmpty()) {
                String context = attachments.get(CommonConstant.MYTH_TRANSACTION_CONTEXT);
                mythTransactionContext =
                        GsonUtils.getInstance().fromJson(context, MythTransactionContext.class);
            }
        } else {
            mythTransactionContext = TransactionContextLocal.getInstance().get();
        }

        return mythTransactionAspectService.invoke(mythTransactionContext, pjp);
    }
}

//dubbo
@Component
public class DubboMythTransactionInterceptor implements MythTransactionInterceptor {

    private final MythTransactionAspectService mythTransactionAspectService;

    @Autowired
    public DubboMythTransactionInterceptor(MythTransactionAspectService mythTransactionAspectService) {
        this.mythTransactionAspectService = mythTransactionAspectService;
    }

    @Override
    public Object interceptor(ProceedingJoinPoint pjp) throws Throwable {
        final String context = RpcContext.getContext().getAttachment(CommonConstant.MYTH_TRANSACTION_CONTEXT);
        MythTransactionContext mythTransactionContext;
        if (StringUtils.isNoneBlank(context)) {
            mythTransactionContext =
                    GsonUtils.getInstance().fromJson(context, MythTransactionContext.class);
        }else{
            mythTransactionContext= TransactionContextLocal.getInstance().get();
        }
        return mythTransactionAspectService.invoke(mythTransactionContext, pjp);
    }
}
```
我们发现是通过实现feign的<b>RequestInterceptor</b>接口来实现<b>mythTransactionContext</b>设置到头信息中的，这里dubbo，motan也类似，只是实现方式不同。这里也是实现分布式事务的最关键一部分，通过同一个事务上下文来关联多子系统之间事务关系，是分布式事务实现的核心所在。

接下来我们进入参与者角色<b>ActorMythTransactionHandler.handler</b>
```java
public Object handler(ProceedingJoinPoint point, MythTransactionContext mythTransactionContext) throws Throwable {

        try {
            //处理并发问题
            LOCK.lock();
            //先保存事务日志
            mythTransactionManager.actorTransaction(point, mythTransactionContext);

            //发起调用 执行try方法
            final Object proceed = point.proceed();

            //执行成功 更新状态为commit
            mythTransactionManager.updateStatus(mythTransactionContext.getTransId(),
                    MythStatusEnum.COMMIT.getCode());

            return proceed;

        } catch (Throwable throwable) {
            LogUtil.error(LOGGER, "执行分布式事务接口失败,事务id：{}", mythTransactionContext::getTransId);
            mythTransactionManager.updateStatus(mythTransactionContext.getTransId(),
                    MythStatusEnum.FAILURE.getCode());
            throw throwable;
        } finally {
            LOCK.unlock();
            TransactionContextLocal.getInstance().remove();
        }
    }
```
参与者实现比较简单， 执行业务方法前主要封装MythTransaction消息（状态为：开始，角色为：参与者），然后进行持久化操作，再执行业务方法，如果成功更新MythTransaction状态为:COMMIT，反之状态为：FAILURE,到这里我们参与者也是走完了  ~~
那我们这个流程是不是完了呢？  其实还没有，上一章最后我们留了一小块，我们再来回顾下
```java
/**
    * Myth分布式事务处理接口
    *
    * @param point                  point 切点
    * @param mythTransactionContext myth事务上下文
    * @return Object
    * @throws Throwable 异常
    */
   @Override
   public Object handler(ProceedingJoinPoint point, MythTransactionContext mythTransactionContext) throws Throwable {

       try {

           //主要防止并发问题，对事务日志的写造成压力，加了锁进行处理
           try {
               LOCK.lock();
               mythTransactionManager.begin(point);
           } finally {
               LOCK.unlock();
           }

          return  point.proceed();

       } finally {
           //发送消息
           mythTransactionManager.sendMessage();
           mythTransactionManager.cleanThreadLocal();
           TransactionContextLocal.getInstance().remove();
       }
   }

```
在走account流程时，其实发起者一直在 point.proceed();这里等待返回结果呢，这里需要等待<b>orderService.orderPay</b>业务方法全部执行完才会返回，然而我们上面才走account一个扣款接口，还有inventory扣减库存接口，这里inventory接口与account接口角色都是参与者，流程上是一样的，只是业务不一样而已，这里也就不做过多介绍了，童鞋们自己过一遍即可。

到这里有童鞋可能就要说了，myth打着是一个基于消息队列解决分布式事务框架，但是前面讲了这么多，貌似都未涉及到消息队列啊， 好了，我们这就带你们飞进mq，我们来看 <b>mythTransactionManager.sendMessage();</b>直接进入关键代码部分<b>CoordinatorServiceImpl.sendMessage</b>方法
```java

public Boolean sendMessage(MythTransaction mythTransaction) {
        final List<MythParticipant> mythParticipants = mythTransaction.getMythParticipants();
            /*
             * 这里的这个判断很重要，不为空，表示本地的方法执行成功，需要执行远端的rpc方法
             * 为什么呢，因为我会在切面的finally里面发送消息，意思是切面无论如何都需要发送mq消息
             * 那么考虑问题，如果本地执行成功，调用rpc的时候才需要发
             * 如果本地异常，则不需要发送mq ，此时mythParticipants为空
             */
        if (CollectionUtils.isNotEmpty(mythParticipants)) {

            for (MythParticipant mythParticipant : mythParticipants) {
                MessageEntity messageEntity =
                        new MessageEntity(mythParticipant.getTransId(),
                                mythParticipant.getMythInvocation());
                try {
                    final byte[] message = serializer.serialize(messageEntity);
                    getMythMqSendService().sendMessage(mythParticipant.getDestination(),
                            mythParticipant.getPattern(),
                            message);
                } catch (Exception e) {
                    e.printStackTrace();
                    return Boolean.FALSE;
                }
            }
            //这里为什么要这么做呢？ 主要是为了防止在极端情况下，发起者执行过程中，突然自身down 机
            //造成消息未发送，新增一个状态标记，如果出现这种情况，通过定时任务发送消息
            this.updateStatus(mythTransaction.getTransId(), MythStatusEnum.COMMIT.getCode());
        }
        return Boolean.TRUE;
    }


    private synchronized MythMqSendService getMythMqSendService() {
       if (mythMqSendService == null) {
           synchronized (CoordinatorServiceImpl.class) {
               if (mythMqSendService == null) {
                   mythMqSendService = SpringBeanUtils.getInstance().getBean(MythMqSendService.class);
               }
           }
       }
       return mythMqSendService;
   }
```
根据代码我们知道，这里主要是将分布式消息封装至MessageEntity中，然后进行序列化发送至mq消息队列，这里有两点要注意：
1. serializer.serialize(messageEntity);  serializer对象为服务启动时通过spi机制加载注入
2. mythMqSendService 为applicationContext.xml配置的rocketmq

既然产生了消息，必然会有消费者去消费，我们回到<b>myth-demo-springcloud-account工程下的RocketmqConsumer</b>类, account服务对应topic=“account”, Inventory服务对应的topic=“inventory”, 我们进入关键代码部分: <b>mythMqReceiveService.processMessage(message);</b>
```java

public Boolean processMessage(byte[] message) {
        try {
            MessageEntity entity;
            try {
                entity = serializer.deSerialize(message, MessageEntity.class);
            } catch (MythException e) {
                e.printStackTrace();
                throw new MythRuntimeException(e.getMessage());
            }
            /*
             * 1 检查该事务有没被处理过，已经处理过的 则不处理
             * 2 发起发射调用，调用接口，进行处理
             * 3 记录本地日志
             */
            LOCK.lock();

            final String transId = entity.getTransId();
            final MythTransaction mythTransaction = findByTransId(transId);

            //如果是空或者是失败的
            if (Objects.isNull(mythTransaction)
                    || mythTransaction.getStatus() == MythStatusEnum.FAILURE.getCode()) {
                try {

                    //设置事务上下文，这个类会传递给远端
                    MythTransactionContext context = new MythTransactionContext();

                    //设置事务id
                    context.setTransId(transId);

                    //设置为发起者角色
                    context.setRole(MythRoleEnum.LOCAL.getCode());

                    TransactionContextLocal.getInstance().set(context);
                    executeLocalTransaction(entity.getMythInvocation());

                    //会进入LocalMythTransactionHandler  那里有保存

                } catch (Exception e) {
                    e.printStackTrace();
                    throw new MythRuntimeException(e.getMessage());
                } finally {
                    TransactionContextLocal.getInstance().remove();
                }
            }


        } finally {
            LOCK.unlock();
        }
        return Boolean.TRUE;

    }

    @SuppressWarnings("unchecked")
    private void executeLocalTransaction(MythInvocation mythInvocation) throws Exception {
        if (Objects.nonNull(mythInvocation)) {
            final Class clazz = mythInvocation.getTargetClass();
            final String method = mythInvocation.getMethodName();
            final Object[] args = mythInvocation.getArgs();
            final Class[] parameterTypes = mythInvocation.getParameterTypes();
            final Object bean = SpringBeanUtils.getInstance().getBean(clazz);
            MethodUtils.invokeMethod(bean, method, args, parameterTypes);
            LogUtil.debug(LOGGER, "Myth执行本地协调事务:{}", () -> mythInvocation.getTargetClass()
                    + ":" + mythInvocation.getMethodName());
        }
    }
```
消费者在接收到消息后，进行反序列化，拿到transId查询分布式事务消息<b>MythTransaction</b>，这里能查到数据吗？ 答案是肯定的，因为前面我们走服务调用时就已经对事务消息进行了持久化操作，我们发现这里需要进行事务状态判断，<b>mythTransaction</b>为空或者事务状态为FAILURE才执行本地协调事务，因为正常接口调用会走一次，所以这里需要避免重复执行，导致数据不一致。

好了，到此为止我们源码解析部分就全部讲解完毕， myth实现是没有回滚机制的，这里有别于tcc，也不同于2pc, 只要发起者本地事务执行成功，那么认为这个事务就必须一直执行下去，直到成功为止，即使在调用其他子系统接口出现超或者本地宕机这种异常情况，待重启后也是会通过调度线程通过mq把事务消息传输给参与者，来达到最终的一致性！

如果大家有任何问题或者建议欢迎沟通 ，欢迎加入QQ群：162614487 进行交流。
