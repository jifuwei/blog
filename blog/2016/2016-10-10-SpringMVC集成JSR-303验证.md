## SpringMVC集成JSR-303验证

### 本文初衷
- 开发表单提交数据接口时，需要对用户提交的数据进行验证以保证数据的安全准确性，这就使得业务逻辑中充斥着太多与业务无关的验证操作，典型场景如下：
```
@RequestMapping("/save")
public ACResponseMsg save(@Validated({EntityGroup.class}) @RequestBody ACValidTestVO vo, BindingResult bindingResult) {
    ACResponseMsg msg = new ACResponseMsg();
    try {

        if (vo.getPrimaryKey == null || vo.getPrimaryKey.trim().length() == 0) {
            ...
        }

        dataService.save(vo);

        msg.errcode = ACErrorMsg.CALL_SUCCESS.errcode;
        msg.errmsg = ACErrorMsg.CALL_SUCCESS.errmsg;
    } catch (ACServiceException e) {
        msg = e.getAcErrorMsg().toResponseMsg();
    } catch (ACDaoException e) {
        msg = e.getAcErrorMsg().toResponseMsg();
    } catch (ACRuntimeException e) {
        msg = e.getAcErrorMsg().toResponseMsg();
    }
    return msg;
}
```
代码中判断逻辑很不优雅而不可避免，试想若bean中有10个字段其中9个需要验证。。。，所以迫切的需要一种更优雅的验证手段

### 集成Hibernate validation到SpringMVC中
#### 1.项目搭建
添加依赖：
```
<!-- For Validatoin -->
<dependency>
    <groupId>javax.validation</groupId>
    <artifactId>validation-api</artifactId>
    <version>${validation.version}</version>
</dependency>
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>${hibernate.validator.version}</version>
</dependency>
```

#### 2.springmvc配置文件
```
<!-- 配置注解驱动 -->
<mvc:annotation-driven validator="validator"/>

<!-- 以下 validator  ConversionService 在使用 mvc:annotation-driven 会自动注册-->
<bean id="validator" class="org.springframework.validation.beanvalidation.LocalValidatorFactoryBean">
    <!-- hibernate校验器-->
    <property name="providerClass" value="org.hibernate.validator.HibernateValidator"/>
    <!-- 如果不加默认到 使用classpath下的 ValidationMessages.properties -->
    <property name="validationMessageSource" ref="messageSource"/>
</bean>

<!-- 国际化的消息资源文件（本系统中主要用于显示/错误消息定制） -->
<bean id="messageSource" class="org.springframework.context.support.ReloadableResourceBundleMessageSource">
    <property name="basenames">
        <list>
            <!-- 在web环境中一定要定位到classpath 否则默认到当前web应用下找，不需要加文件后缀.properties  -->
            <value>classpath:validation-message</value>
        </list>
    </property>
    <property name="useCodeAsDefaultMessage" value="false"/>
    <property name="defaultEncoding" value="UTF-8"/>
    <property name="cacheSeconds" value="60"/>
</bean>
```

#### 3.实体验证注解
```
public class ACValidTestVO {

    @NotEmpty(message = "{ACValidTestVO.acId.notempty}", groups = {PrimaryKeyGroup.class})
    private String acId;

    @NotBlank(message = "{ACValidTestVO.mysqlVarchar.notblank}", groups = {EntityGroup.class})
    @Length(min = 1, max = 255, message = "{ACValidTestVO.mysqlVarchar.length}", groups = {EntityGroup.class})
    private String mysqlVarchar;//字符串

    @NotEmpty(message = "{ACValidTestVO.mysqlChar.notempty}", groups = {EntityGroup.class})
    private String mysqlChar;//字符

    @NotEmpty(message = "{ACValidTestVO.mysqlBlob.notempty}", groups = {EntityGroup.class})
    private String mysqlBlob;//二进制大对象
    private String mysqlText;//文本

    @Min(value = -2147483648, message = "{ACValidTestVO.mysqlInt.min}", groups = {EntityGroup.class})
    @Max(value = 2147483647, message = "{ACValidTestVO.mysqlInt.max}", groups = {EntityGroup.class})
    private String mysqlInt;//整形
    private String mysqlTinyint;//小整形
    private String mysqlSmallint;//短整形
    private String mysqlMediumint;//中整形
    private String mysqlBigint;//大整形
    private String mysqlBit;//比特

    @Digits(integer = 2, fraction = 2, message = "{ACValidTestVO.mysqlFloat.digits}", groups = {EntityGroup.class})
    private String mysqlFloat;//浮点型
    private String mysqlDouble;//双精度型
    private String mysqlDecimal;//小数
    private String mysqlBoolean;//布尔型
    private String mysqlDate;//日期
    private String mysqlTime;//时间
    private String mysqlDatetime;//日期时间
    private String mysqlTimestamp;//时间戳
    private String mysqlYear;//年
}
```
其中groups参数是分组验证概念，常用场景为：当表的主键为自增长时，增加表数据是不需要验证主键是否存在，而更新表数据时需要验证主键是否存在，则运用到分组验证而不需要写多个bean

错误消息配置如下：
```
# 国际化配置zh-cn
ACValidTestVO.acId.notempty = acId字段不能为空
ACValidTestVO.mysqlVarchar.notblank = mysqlVarchar字段不能为空
ACValidTestVO.mysqlVarchar.length = mysqlVarchar字段大小范围：{min}~{max}
ACValidTestVO.mysqlChar.notempty = mysqlChar字段不能为空
ACValidTestVO.mysqlBlob.notempty = mysqlBlob字段不能为空
ACValidTestVO.mysqlInt.min = mysqlInt字段为整数且不小于-2147483648
ACValidTestVO.mysqlInt.max = mysqlInt字段为整数且不大于2147483647
ACValidTestVO.mysqlFloat.digits = mysqlFloat字段必须为小于等于{integer}为整数，小于等于{fraction}位小数精度

# 国际化配置en
```

#### 4.控制器处理
```
/**
 * 添加数据
 * @param vo
 * @param bindingResult
 * @return
 */
@RequestMapping("/save")
public ACResponseMsg save(@Validated({EntityGroup.class}) @RequestBody ACValidTestVO vo, BindingResult bindingResult) {
    ACResponseMsg msg = new ACResponseMsg();
    try {
        dataService.save(vo);

        msg.errcode = ACErrorMsg.CALL_SUCCESS.errcode;
        msg.errmsg = ACErrorMsg.CALL_SUCCESS.errmsg;
    } catch (ACServiceException e) {
        msg = e.getAcErrorMsg().toResponseMsg();
    } catch (ACDaoException e) {
        msg = e.getAcErrorMsg().toResponseMsg();
    } catch (ACRuntimeException e) {
        msg = e.getAcErrorMsg().toResponseMsg();
    }
    return msg;
}
```
其中`@validated`注解用于验证实体类，不设置当前需要验证的分组，`BindingResult`用于抓取验证错误信息，你可能注意到不没有错误处理程序，由于我们是个洁癖程序员，随意错误消息验证代码不能写在控制层，这也不符合我们的初衷，综上使用强大的`AOP`理念处理错误

#### 5.AOP处理验证错误信息
```
public Object aroundMethod(ProceedingJoinPoint joinPoint) throws Throwable {
    ACResponseMsg msg = null;
    BindingResult bindingResult = null;
    StringBuilder sb = null;
    for (Object arg : joinPoint.getArgs()) {
        if (arg instanceof BindingResult) {
            bindingResult = (BindingResult) arg;
            break;
        }
    }

    if (bindingResult != null) {
        List<ObjectError> allErrors = bindingResult.getAllErrors();
        if (allErrors.size() > 0) {
            msg = new ACResponseMsg();
            sb = new StringBuilder();
            for (ObjectError error : allErrors) {
                FieldError fieldError = (FieldError) error;
                sb.append(fieldError.getDefaultMessage()).append(";");
            }
            msg.errcode = ACErrorMsg.ERROR_DATA_IS_MALFORM.errcode;
            msg.errmsg = sb.toString();
            return msg;
        }
    }
    return joinPoint.proceed();
}
```

springmvc配置如下：
```
<!-- 切面配置:接口参数校验 -->
<bean id="bindingResultAOP" class="com.jifuwei.ac.foundation.aop.BindingResultAOP"/>
<aop:config>
    <aop:pointcut id="bindingResultAOPPointcut"
                  expression="execution(* com.jifuwei.ac.web.*.controller.*Controller.*(..) )"/>
    <aop:aspect id="controllerVaildAspect" ref="bindingResultAOP">
        <aop:around method="aroundMethod" pointcut-ref="bindingResultAOPPointcut"/>
    </aop:aspect>
</aop:config>
```

#### 6.高级配置：
- `@GroupSequence({First.class, Second.class, User.class})`，该注解在bean类级别上，按照指定的分组来验证，当第一个分组出错，立即停止
- `@ConvertGroup(from=First.class, to=Second.class)`，级联验证，@ConvertGroup的作用是当验证o的分组是First时，那么验证o的分组是Second，即分组验证的转换。
