# 流程

## 开发流程

### 需求分析

产品原型，需求规格说明书

### 设计

产品文档，UI界面设计，概要设计，详细设计，数据库设计

### 编码

项目代码，单元测试

### 测试

测试用例，测试报告

### 上线运维

软件环境安转，配置

## 角色分工

* 项目经理
* 产品经理
* UI设计师
* 架构师
* 开发工程师
* 测试工程师
* 运维工程师

## 软件环境

开发环境

测试环境

生产环境

# 项目

## 项目介绍

![](https://pic.imgdb.cn/item/63733c8016f2c2beb1267bc5.jpg)

## 技术选型

![](https://pic.imgdb.cn/item/6373522a16f2c2beb147dda6.jpg)

## 功能架构

![](https://pic.imgdb.cn/item/637352e516f2c2beb148d924.jpg)

## 角色

![](https://pic.imgdb.cn/item/637353b016f2c2beb14a4b71.jpg)

# 开发

## 数据库

直接引入SQL文件，在资料/SQL中

![](https://pic.imgdb.cn/item/637355be16f2c2beb14e0641.jpg)

## maven

新建maven项目，导入pom文件

导入配置文件，设置启动项

## 前端

导入前端的代码

编写配置类访问静态资源

~~~java
package xyz.liyouxiu.reggie.config;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport;

/**
 * @author liyouxiu
 * @date 2022/11/15 18:08
 */
@Slf4j
@Configuration
public class WebMvcConfig extends WebMvcConfigurationSupport {
    /**
     * 设置静态资源映射
     * @param registry
     */
    @Override
    protected void addResourceHandlers(ResourceHandlerRegistry registry) {
        log.info("开始静态资源映射……");
        registry.addResourceHandler("/backend/**").addResourceLocations("classpath:/backend/");
        registry.addResourceHandler("/front/**").addResourceLocations("classpath:/front/");

    }
}

~~~

## 业务功能开发

### 登录

创建Controller，Service，Mapper

因为用了MyBatisPlus所以，Service和ServiceImpl需要实现或者继承MyBatisPlus的接口和方法

~~~java
package xyz.liyouxiu.reggie.service;

import com.baomidou.mybatisplus.extension.service.IService;
import xyz.liyouxiu.reggie.entity.Employee;

/**
 * @author liyouxiu
 * @date 2022/11/15 19:38
 */
public interface EmployeeService extends IService<Employee> {
}
~~~

~~~java
package xyz.liyouxiu.reggie.impl;

import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import org.springframework.stereotype.Service;
import xyz.liyouxiu.reggie.entity.Employee;
import xyz.liyouxiu.reggie.mapper.EmployeeMapper;
import xyz.liyouxiu.reggie.service.EmployeeService;

/**
 * @author liyouxiu
 * @date 2022/11/15 19:39
 */
@Service
public class EmployeeServiceImpl extends ServiceImpl<EmployeeMapper, Employee> implements EmployeeService{

}
~~~

导入结果返回类

~~~java
package xyz.liyouxiu.reggie.common;

import lombok.Data;
import java.util.HashMap;
import java.util.Map;

/**
 * 通用返回结果，服务端响应的数据最终都会封装成此对象
 * @param <T>
 */
@Data
public class R<T> {

    private Integer code; //编码：1成功，0和其它数字为失败

    private String msg; //错误信息

    private T data; //数据

    private Map map = new HashMap(); //动态数据

    public static <T> R<T> success(T object) {
        R<T> r = new R<T>();
        r.data = object;
        r.code = 1;
        return r;
    }

    public static <T> R<T> error(String msg) {
        R r = new R();
        r.msg = msg;
        r.code = 0;
        return r;
    }

    public R<T> add(String key, Object value) {
        this.map.put(key, value);
        return this;
    }

}
~~~

**功能分析**

![](https://pic.imgdb.cn/item/63737f0816f2c2beb18cf7f0.jpg)

~~~java
package xyz.liyouxiu.reggie.controller;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.util.DigestUtils;
import org.springframework.web.bind.annotation.*;
import xyz.liyouxiu.reggie.common.R;
import xyz.liyouxiu.reggie.entity.Employee;
import xyz.liyouxiu.reggie.service.EmployeeService;

import javax.servlet.http.HttpServletRequest;

/**
 * @author liyouxiu
 * @date 2022/11/15 19:41
 */
@Slf4j
@RestController
@RequestMapping("/employee")
public class EmployeeController {

    @Autowired
    private EmployeeService employeeService;

    /**
     * 员工登录
     * @param request
     * @param employee
     * @return
     */
    @PostMapping("/login")
    public R<Employee> login(HttpServletRequest request, @RequestBody Employee employee){
        //1.将页面提交的密码password进行MD5加密
        String password = employee.getPassword();
        password=DigestUtils.md5DigestAsHex(password.getBytes());

        //2.根据页面提交的用户名username查询数据库
        LambdaQueryWrapper<Employee> queryWrapper = new LambdaQueryWrapper<>();
        //eq mp中的比较方法
        queryWrapper.eq(Employee::getUsername,employee.getUsername());
        //getOne唯一的数据
        Employee emp = employeeService.getOne(queryWrapper);

        //3.如果没有查询到返回登录失败结果
        if(emp==null){
            return R.error("登录失败");
        }
        //4.密码比对，如果不一致则返回登录失败结果
        if(!emp.getPassword().equals(password)){
            return R.error("密码错误");
        }
        //5.查看员工状态，如果为已禁用状态，则返回员工已禁用
        if(emp.getStatus()==0){
            return R.error("员工已禁用");
        }
        //6.登录成功，将员工的ID存入Session并返回登录成功结果
        request.getSession().setAttribute("employee",emp.getId());
        return R.success(emp);
    }

}
~~~

**功能测试**

### 退出

![](https://pic.imgdb.cn/item/63738a2c16f2c2beb19f8346.jpg)

~~~java
/**
     * 员工退出
     * @param request
     * @return
     */
    @PostMapping("/logout")
    public R<String> logout(HttpServletRequest request){
        //1.清理Session中保存的当前登录的员工ID
        request.getSession().removeAttribute("employee");
        //2.返回结果
        return R.success("退出成功");
    }
~~~
