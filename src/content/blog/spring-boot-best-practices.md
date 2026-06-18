---
title: 'Spring Boot 项目最佳实践总结'
description: 'Spring Boot 项目开发中的规范、配置和最佳实践'
pubDate: 2024-11-15
heroImage: ''
tags: ['Java', 'Spring Boot', '最佳实践']
---

## 项目规范

### 分层架构

```
controller/     → @RestController，只做参数校验和路由
service/        → @Service + @Transactional，核心业务逻辑
mapper/         → MyBatis-Plus Mapper，数据访问
entity/         → 数据库实体
dto/            → 数据传输对象
config/         → 配置类
common/         → 通用工具、常量、异常
```

### 统一响应格式

```java
public class R<T> {
    private int code;
    private String msg;
    private T data;
}
```

### 全局异常处理

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(Exception.class)
    public R<?> handleException(Exception e) {
        log.error("系统异常", e);
        return R.fail("系统异常");
    }
}
```

## 配置要点

- 多环境配置：`application-dev.yml` / `application-prod.yml`
- 日志配置：SLF4J + Logback
- 连接池：HikariCP（Spring Boot 默认）

## MyBatis-Plus 常用技巧

```java
// 分页查询
Page<User> page = new Page<>(1, 10);
userMapper.selectPage(page, queryWrapper);

// 逻辑删除
@TableLogic
private Integer deleted;
```

这些实践可以帮助团队保持代码一致性和可维护性。
