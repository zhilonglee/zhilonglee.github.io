# AI 辅助后端编码全流程实战：从需求到代码的高效开发

> **写在前面：** 传统后端开发中，从需求文档到可运行代码通常需要数天时间。借助 AI 辅助，这个流程可以缩短到几小时甚至几分钟。本文详细记录如何使用 AI 完成从业务需求拆解、数据库设计、代码生成到测试用例编写的全流程开发，让你的开发效率提升 10 倍以上！

---

## 📋 目录

- [一、AI 辅助开发的核心理念](#一ai-辅助开发的核心理念)
- [二、需求文档快速拆解为开发任务](#二需求文档快速拆解为开发任务)
- [三、数据库结构设计与 AI 辅助](#三数据库结构设计与-ai-辅助)
- [四、持久层代码快速生成](#四持久层代码快速生成)
- [五、业务层 Service 分层逻辑梳理](#五业务层-service-分层逻辑梳理)
- [六、控制层 Controller 接口开发](#六控制层-controller-接口开发)
- [七、全局配置与异常处理](#七全局配置与异常处理)
- [八、第三方 SDK 对接代码生成](#八第三方-sdk-对接代码生成)
- [九、存量代码批量优化技巧](#九存量代码批量优化技巧)
- [十、完整实战案例](#十完整实战案例)

---

## 一、AI 辅助开发的核心理念

### 1.1 传统开发 vs AI 辅助开发对比

| 阶段 | 传统开发耗时 | AI 辅助耗时 | 效率提升 |
|------|------------|-----------|---------|
| 需求分析 | 4 小时 | 30 分钟 | **8 倍** |
| 数据库设计 | 2 小时 | 15 分钟 | **8 倍** |
| Entity/Mapper 生成 | 3 小时 | 10 分钟 | **18 倍** |
| Service 实现 | 6 小时 | 1 小时 | **6 倍** |
| Controller 开发 | 2 小时 | 20 分钟 | **6 倍** |
| 单元测试 | 4 小时 | 30 分钟 | **8 倍** |
| **总计** | **21 小时** | **2.75 小时** | **7.6 倍** |

---

### 1.2 AI 辅助开发的优势

**✅ 核心优势：**

1. **速度极快**：重复性代码秒级生成
2. **规范统一**：遵循既定编码规范
3. **减少遗漏**：自动补充异常处理、日志等
4. **知识复用**：快速应用最佳实践
5. **降低门槛**：新手也能写出高质量代码

**⚠️ 注意事项：**

1. AI 生成的代码需要人工审查
2. 复杂业务逻辑仍需人工设计
3. 安全性问题需要特别关注
4. 性能优化需要结合实际场景

---

### 1.3 适用场景与边界

**✅ 适合 AI 辅助的场景：**
- CRUD 基础功能开发
- 标准接口封装
- 单元测试生成
- 代码重构优化
- 文档编写

**❌ 不适合完全依赖 AI 的场景：**
- 核心算法设计
- 复杂业务流程编排
- 系统架构决策
- 安全敏感模块
- 性能关键路径

---

## 二、需求文档快速拆解为开发任务

### 2.1 需求拆解模板

**原始需求示例：**

```
我们需要一个用户管理模块，支持：
1. 用户注册（手机号、密码、验证码）
2. 用户登录（手机号+密码、手机号+验证码）
3. 用户信息查询和修改
4. 密码重置
5. 账号注销
```

---

**使用 AI 拆解为开发任务：**

```markdown
你是一位资深后端架构师

请将以下用户需求拆解为详细的后端开发任务清单：

【用户需求】
[粘贴上面的需求]

【拆解要求】
1. 按功能模块分组
2. 每个任务包含：
   - 任务名称
   - 优先级（P0/P1/P2）
   - 预估工时
   - 技术要点
   - 依赖关系
3. 识别潜在的技术难点
4. 给出开发顺序建议

【输出格式】
## 一、数据库设计阶段
### 任务 1.1：用户表结构设计
- 优先级：P0
- 预估工时：2 小时
- 技术要点：
  - 手机号唯一索引
  - 密码 BCrypt 加密存储
  - 软删除支持
...

## 二、核心功能开发阶段
...

## 三、技术难点与风险
1. ...
2. ...

## 四、推荐开发顺序
1. ...
2. ...
```

---

### 2.2 AI 输出的任务清单示例

```markdown
## 一、数据库设计阶段

### 任务 1.1：用户表结构设计
- **优先级：** P0
- **预估工时：** 2 小时
- **技术要点：**
  - 手机号唯一索引
  - 密码 BCrypt 加密存储
  - 软删除支持（deleted 字段）
  - 审计字段（create_time, update_time）
- **依赖关系：** 无

### 任务 1.2：短信验证码表设计
- **优先级：** P0
- **预估工时：** 1 小时
- **技术要点：**
  - 验证码有效期（5 分钟）
  - 发送频率限制（1 分钟/次）
  - Redis 缓存支持
- **依赖关系：** 无

---

## 二、核心功能开发阶段

### 任务 2.1：用户注册接口
- **优先级：** P0
- **预估工时：** 4 小时
- **技术要点：**
  - 手机号格式校验
  - 验证码校验
  - 密码强度校验
  - 幂等性控制（防止重复注册）
  - 事务管理
- **依赖关系：** 任务 1.1、1.2

### 任务 2.2：用户登录接口（密码方式）
- **优先级：** P0
- **预估工时：** 3 小时
- **技术要点：**
  - BCrypt 密码比对
  - JWT Token 生成
  - 登录失败次数限制
  - 登录日志记录
- **依赖关系：** 任务 1.1

...（后续任务）

---

## 三、技术难点与风险

1. **验证码防刷：** 需要实现 IP 限流、图形验证码
2. **密码安全：** 必须使用 BCrypt，禁止明文存储
3. **并发注册：** 需要处理手机号唯一性约束的并发冲突
4. **Token 刷新：** 需要设计 Refresh Token 机制

---

## 四、推荐开发顺序

1. 数据库表设计（任务 1.1、1.2）
2. 基础框架搭建（Entity、Mapper）
3. 用户注册功能（任务 2.1）
4. 用户登录功能（任务 2.2、2.3）
5. 用户信息管理（任务 3.1、3.2）
6. 密码重置功能（任务 4.1）
7. 账号注销功能（任务 5.1）
8. 单元测试与集成测试
```

---

### 2.3 任务优先级评估技巧

**让 AI 帮你评估优先级：**

```markdown
基于以下因素评估每个任务的优先级：

【评估维度】
1. 业务价值（高/中/低）
2. 技术复杂度（高/中/低）
3. 依赖关系（前置/后置）
4. 风险程度（高/中/低）

【优先级定义】
- P0：核心功能，必须优先完成
- P1：重要功能，第二批次完成
- P2：锦上添花，最后完成

请为每个任务标注优先级，并说明理由。
```

---

## 三、数据库结构设计与 AI 辅助

### 3.1 表结构设计专用提示词

```markdown
你是一位 MySQL 数据库专家

请设计一个用户管理模块的数据库表结构，要求如下：

【业务需求】
1. 用户基本信息（手机号、昵称、头像、性别、生日）
2. 支持第三方登录（微信、支付宝）
3. 支持账号状态管理（正常/冻结/注销）
4. 记录最后登录时间和 IP
5. 支持软删除

【设计要求】
1. 遵循第三范式（3NF）
2. 合理选择数据类型
3. 添加必要的索引
4. 所有字段添加注释
5. 考虑扩展性（预留扩展字段）
6. 审计字段（创建时间、更新时间、操作人）

【技术规范】
- MySQL 8.0
- InnoDB 引擎
- UTF8MB4 字符集
- 主键使用 BIGINT 自增
- 时间使用 DATETIME

【输出格式】
1. ER 图描述（文字版）
2. 完整的 CREATE TABLE SQL
3. 索引设计说明
4. 字段设计理由
5. 潜在的优化建议
```

---

### 3.2 AI 生成的表结构示例

```sql
-- ============================================
-- 用户主表
-- ============================================
CREATE TABLE `users` (
  `id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `phone` VARCHAR(11) NOT NULL COMMENT '手机号',
  `password` VARCHAR(100) DEFAULT NULL COMMENT '密码（BCrypt加密）',
  `nickname` VARCHAR(50) DEFAULT NULL COMMENT '昵称',
  `avatar` VARCHAR(255) DEFAULT NULL COMMENT '头像URL',
  `gender` TINYINT DEFAULT 0 COMMENT '性别：0-未知，1-男，2-女',
  `birthday` DATE DEFAULT NULL COMMENT '生日',
  
  `status` TINYINT NOT NULL DEFAULT 1 COMMENT '账号状态：1-正常，2-冻结，3-注销',
  `last_login_time` DATETIME DEFAULT NULL COMMENT '最后登录时间',
  `last_login_ip` VARCHAR(50) DEFAULT NULL COMMENT '最后登录IP',
  
  `deleted` TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0-未删除，1-已删除',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人',
  
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_phone` (`phone`) COMMENT '手机号唯一索引',
  KEY `idx_status` (`status`) COMMENT '状态索引',
  KEY `idx_create_time` (`create_time`) COMMENT '创建时间索引'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户主表';

-- ============================================
-- 第三方登录绑定表
-- ============================================
CREATE TABLE `user_oauth` (
  `id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `user_id` BIGINT NOT NULL COMMENT '用户ID',
  `oauth_type` VARCHAR(20) NOT NULL COMMENT '第三方类型：wechat/alipay',
  `openid` VARCHAR(100) NOT NULL COMMENT '第三方OpenID',
  `union_id` VARCHAR(100) DEFAULT NULL COMMENT '第三方UnionID',
  `access_token` VARCHAR(255) DEFAULT NULL COMMENT '访问令牌',
  `refresh_token` VARCHAR(255) DEFAULT NULL COMMENT '刷新令牌',
  `expire_time` DATETIME DEFAULT NULL COMMENT '令牌过期时间',
  
  `deleted` TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_oauth_type_openid` (`oauth_type`, `openid`) COMMENT '联合唯一索引',
  KEY `idx_user_id` (`user_id`) COMMENT '用户ID索引',
  CONSTRAINT `fk_oauth_user` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='第三方登录绑定表';
```

---

### 3.3 索引优化建议

```markdown
## 索引设计说明

### 1. users 表索引

| 索引名 | 字段 | 类型 | 说明 |
|--------|------|------|------|
| PRIMARY | id | 聚簇索引 | 主键，默认创建 |
| uk_phone | phone | 唯一索引 | 手机号唯一性约束，高频查询 |
| idx_status | status | 普通索引 | 账号状态筛选，低频查询 |
| idx_create_time | create_time | 普通索引 | 按注册时间排序 |

### 2. user_oauth 表索引

| 索引名 | 字段 | 类型 | 说明 |
|--------|------|------|------|
| PRIMARY | id | 聚簇索引 | 主键 |
| uk_oauth_type_openid | oauth_type, openid | 唯一索引 | 防止重复绑定 |
| idx_user_id | user_id | 普通索引 | 查询用户的第三方绑定 |

## 优化建议

1. **覆盖索引：** 如果经常查询 `phone` 和 `status`，可以考虑联合索引 `(phone, status)`
2. **分区表：** 如果数据量超过 1000 万，建议按 `create_time` 按月分区
3. **读写分离：** 高频查询可以考虑主从复制，读操作走从库
```

---

## 四、持久层代码快速生成

### 4.1 MyBatis-Plus 代码生成

```markdown
你是一位 MyBatis-Plus 专家

请基于以下用户表结构，生成完整的持久层代码：

【表结构】
[粘贴上面的 CREATE TABLE SQL]

【技术要求】
- Java 17
- MyBatis-Plus 3.5.5
- Lombok
- 遵循阿里巴巴 Java 开发手册

【生成内容】
1. Entity 实体类（使用 @TableName、@TableId 等注解）
2. Mapper 接口（继承 BaseMapper）
3. 自定义查询方法（使用 @Select 或 XML）
4. Service 接口（继承 IService）
5. Service 实现类（继承 ServiceImpl）

【代码规范】
- 添加完整的 JavaDoc 注释
- 使用驼峰命名
- 敏感字段（password）不在日志中输出
- 逻辑删除使用 @TableLogic

【输出格式】
请按以下顺序输出，每个类单独代码块：
1. User.java
2. UserMapper.java
3. UserService.java
4. UserServiceImpl.java
```

---

### 4.2 AI 生成的代码示例

**User.java：**

```java
package com.example.user.entity;

import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;
import lombok.experimental.Accessors;

import java.io.Serializable;
import java.time.LocalDate;
import java.time.LocalDateTime;

/**
 * 用户实体类
 *
 * @author AI Assistant
 * @since 2026-05-17
 */
@Data
@Accessors(chain = true)
@TableName("users")
public class User implements Serializable {

    private static final long serialVersionUID = 1L;

    /**
     * 主键ID
     */
    @TableId(value = "id", type = IdType.AUTO)
    private Long id;

    /**
     * 手机号
     */
    private String phone;

    /**
     * 密码（BCrypt加密）
     */
    private String password;

    /**
     * 昵称
     */
    private String nickname;

    /**
     * 头像URL
     */
    private String avatar;

    /**
     * 性别：0-未知，1-男，2-女
     */
    private Integer gender;

    /**
     * 生日
     */
    private LocalDate birthday;

    /**
     * 账号状态：1-正常，2-冻结，3-注销
     */
    private Integer status;

    /**
     * 最后登录时间
     */
    private LocalDateTime lastLoginTime;

    /**
     * 最后登录IP
     */
    private String lastLoginIp;

    /**
     * 逻辑删除：0-未删除，1-已删除
     */
    @TableLogic
    private Integer deleted;

    /**
     * 创建时间
     */
    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    /**
     * 更新时间
     */
    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

    /**
     * 创建人
     */
    private Long createBy;

    /**
     * 更新人
     */
    private Long updateBy;
}
```

---

**UserMapper.java：**

```java
package com.example.user.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.example.user.entity.User;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;

import java.util.List;

/**
 * 用户 Mapper 接口
 *
 * @author AI Assistant
 * @since 2026-05-17
 */
@Mapper
public interface UserMapper extends BaseMapper<User> {

    /**
     * 根据手机号查询用户
     *
     * @param phone 手机号
     * @return 用户信息
     */
    @Select("SELECT * FROM users WHERE phone = #{phone} AND deleted = 0")
    User selectByPhone(@Param("phone") String phone);

    /**
     * 查询指定时间段内注册的用户
     *
     * @param startTime 开始时间
     * @param endTime 结束时间
     * @return 用户列表
     */
    @Select("SELECT * FROM users WHERE create_time BETWEEN #{startTime} AND #{endTime} AND deleted = 0 ORDER BY create_time DESC")
    List<User> selectByTimeRange(@Param("startTime") String startTime, 
                                  @Param("endTime") String endTime);

    /**
     * 统计指定状态的用户数量
     *
     * @param status 状态
     * @return 用户数量
     */
    @Select("SELECT COUNT(*) FROM users WHERE status = #{status} AND deleted = 0")
    int countByStatus(@Param("status") Integer status);
}
```

---

## 五、业务层 Service 分层逻辑梳理

### 5.1 Service 设计原则

**单一职责原则：**
- 每个方法只做一件事
- 方法长度不超过 50 行
- 复杂逻辑拆分为私有方法

**事务管理：**
- 写操作添加 @Transactional
- 只读操作添加 @Transactional(readOnly = true)
- 避免大事务

**异常处理：**
- 业务异常抛出自定义异常
- 记录关键日志
- 不吞掉异常

---

### 5.2 用户注册 Service 实现

```markdown
你是一位 Java 后端开发专家

请实现用户注册的 Service 方法，要求如下：

【业务逻辑】
1. 参数校验（手机号格式、密码强度、验证码）
2. 检查手机号是否已注册
3. 验证验证码是否正确
4. 密码 BCrypt 加密
5. 创建用户记录
6. 删除已使用的验证码
7. 发送欢迎邮件（异步，失败不影响注册）
8. 记录注册日志

【技术要求】
- SpringBoot 3.2
- MyBatis-Plus
- BCrypt 密码加密
- Redis 验证码存储
- 异步邮件使用 @Async
- 事务管理

【异常处理】
- 手机号已注册：throw new BusinessException("手机号已注册")
- 验证码错误：throw new BusinessException("验证码错误或已过期")
- 其他异常：记录日志，抛出系统异常

【输出】
完整的 UserServiceImpl 代码，包含：
1. register 方法
2. 私有辅助方法
3. 完整的注释
```

---

**AI 生成的 Service 代码：**

```java
package com.example.user.service.impl;

import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.example.user.entity.User;
import com.example.user.mapper.UserMapper;
import com.example.user.service.UserService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;

/**
 * 用户服务实现类
 *
 * @author AI Assistant
 * @since 2026-05-17
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {

    private final UserMapper userMapper;
    private final BCryptPasswordEncoder passwordEncoder;
    // private final EmailService emailService; // 异步邮件服务

    /**
     * 用户注册
     *
     * @param phone 手机号
     * @param password 密码
     * @param verifyCode 验证码
     * @return 用户ID
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public Long register(String phone, String password, String verifyCode) {
        log.info("用户注册请求: phone={}", phone);

        // 1. 参数校验
        validateRegisterParams(phone, password, verifyCode);

        // 2. 检查手机号是否已注册
        if (isPhoneRegistered(phone)) {
            throw new BusinessException("手机号已注册");
        }

        // 3. 验证验证码
        if (!verifyCode(phone, verifyCode)) {
            throw new BusinessException("验证码错误或已过期");
        }

        // 4. 创建用户
        User user = createUser(phone, password);
        userMapper.insert(user);

        // 5. 删除已使用的验证码
        deleteVerifyCode(phone);

        // 6. 发送欢迎邮件（异步）
        // emailService.sendWelcomeEmailAsync(phone);

        // 7. 记录注册日志
        log.info("用户注册成功: userId={}, phone={}", user.getId(), phone);

        return user.getId();
    }

    /**
     * 校验注册参数
     */
    private void validateRegisterParams(String phone, String password, String verifyCode) {
        if (!isValidPhone(phone)) {
            throw new BusinessException("手机号格式不正确");
        }
        if (!isValidPassword(password)) {
            throw new BusinessException("密码强度不足（至少8位，包含字母和数字）");
        }
        if (verifyCode == null || verifyCode.length() != 6) {
            throw new BusinessException("验证码格式不正确");
        }
    }

    /**
     * 检查手机号是否已注册
     */
    private boolean isPhoneRegistered(String phone) {
        User existingUser = userMapper.selectByPhone(phone);
        return existingUser != null;
    }

    /**
     * 验证验证码
     */
    private boolean verifyCode(String phone, String code) {
        // TODO: 从 Redis 获取验证码并比对
        // String storedCode = redisTemplate.opsForValue().get("verify_code:" + phone);
        // return code.equals(storedCode);
        return true; // 临时返回
    }

    /**
     * 创建用户对象
     */
    private User createUser(String phone, String password) {
        User user = new User();
        user.setPhone(phone);
        user.setPassword(passwordEncoder.encode(password));
        user.setStatus(1); // 正常状态
        user.setCreateTime(LocalDateTime.now());
        user.setUpdateTime(LocalDateTime.now());
        return user;
    }

    /**
     * 删除验证码
     */
    private void deleteVerifyCode(String phone) {
        // TODO: 从 Redis 删除验证码
        // redisTemplate.delete("verify_code:" + phone);
    }

    /**
     * 校验手机号格式
     */
    private boolean isValidPhone(String phone) {
        return phone != null && phone.matches("^1[3-9]\\d{9}$");
    }

    /**
     * 校验密码强度
     */
    private boolean isValidPassword(String password) {
        if (password == null || password.length() < 8) {
            return false;
        }
        // 至少包含字母和数字
        return password.matches(".*[a-zA-Z].*") && password.matches(".*\\d.*");
    }
}
```

---

## 六、控制层 Controller 接口开发

### 6.1 RESTful API 设计规范

```markdown
你是一位 RESTful API 设计专家

请为用户注册功能设计标准的 Controller 接口，要求如下：

【接口要求】
- URL: POST /api/v1/users/register
- 请求体：{ phone, password, verifyCode }
- 响应：统一 Result 格式
- 状态码：200（成功）、400（参数错误）、409（冲突）、500（服务器错误）

【技术规范】
- SpringBoot 3.2
- @Valid 参数校验
- Swagger 3.0 文档注解
- 统一异常处理
- 接口限流（单用户 QPS < 5）

【输出】
完整的 UserController 代码，包含：
1. register 方法
2. 参数 DTO 类
3. Swagger 注解
4. 完整的注释
```

---

**AI 生成的 Controller 代码：**

```java
package com.example.user.controller;

import com.example.user.dto.RegisterRequest;
import com.example.user.service.UserService;
import com.example.common.result.Result;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.*;

/**
 * 用户控制器
 *
 * @author AI Assistant
 * @since 2026-05-17
 */
@Slf4j
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
@Tag(name = "用户管理", description = "用户注册、登录、信息查询等接口")
public class UserController {

    private final UserService userService;

    /**
     * 用户注册
     *
     * @param request 注册请求
     * @return 用户ID
     */
    @PostMapping("/register")
    @Operation(summary = "用户注册", description = "通过手机号和验证码注册用户")
    public Result<Long> register(@Valid @RequestBody RegisterRequest request) {
        log.info("收到用户注册请求: phone={}", request.getPhone());
        
        Long userId = userService.register(
            request.getPhone(),
            request.getPassword(),
            request.getVerifyCode()
        );
        
        return Result.success(userId);
    }
}
```

---

**RegisterRequest.java：**

```java
package com.example.user.dto;

import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Pattern;
import lombok.Data;

/**
 * 用户注册请求 DTO
 *
 * @author AI Assistant
 * @since 2026-05-17
 */
@Data
@Schema(description = "用户注册请求")
public class RegisterRequest {

    @NotBlank(message = "手机号不能为空")
    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
    @Schema(description = "手机号", example = "13800138000")
    private String phone;

    @NotBlank(message = "密码不能为空")
    @Schema(description = "密码", example = "Abc12345")
    private String password;

    @NotBlank(message = "验证码不能为空")
    @Pattern(regexp = "^\\d{6}$", message = "验证码格式不正确")
    @Schema(description = "验证码", example = "123456")
    private String verifyCode;
}
```

---

## 七、全局配置与异常处理

### 7.1 统一异常处理

```markdown
你是一位 SpringBoot 异常处理专家

请设计一个全局异常处理方案，要求如下：

【需求】
1. 统一异常响应格式
2. 区分业务异常和系统异常
3. 生产环境隐藏详细错误信息
4. 记录异常日志
5. 支持国际化错误消息

【输出】
1. 统一响应结果类 Result<T>
2. 自定义业务异常 BusinessException
3. 全局异常处理器 GlobalExceptionHandler
4. 错误码枚举 ErrorCode
```

---

**Result.java：**

```java
package com.example.common.result;

import lombok.Data;

/**
 * 统一响应结果
 *
 * @author AI Assistant
 * @since 2026-05-17
 */
@Data
public class Result<T> {

    private Integer code;
    private String message;
    private T data;

    public static <T> Result<T> success(T data) {
        Result<T> result = new Result<>();
        result.setCode(200);
        result.setMessage("success");
        result.setData(data);
        return result;
    }

    public static <T> Result<T> error(Integer code, String message) {
        Result<T> result = new Result<>();
        result.setCode(code);
        result.setMessage(message);
        return result;
    }
}
```

---

**GlobalExceptionHandler.java：**

```java
package com.example.common.exception;

import com.example.common.result.Result;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.validation.BindException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;

/**
 * 全局异常处理器
 *
 * @author AI Assistant
 * @since 2026-05-17
 */
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    /**
     * 业务异常
     */
    @ExceptionHandler(BusinessException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Result<Void> handleBusinessException(BusinessException e) {
        log.warn("业务异常: {}", e.getMessage());
        return Result.error(e.getCode(), e.getMessage());
    }

    /**
     * 参数校验异常
     */
    @ExceptionHandler({MethodArgumentNotValidException.class, BindException.class})
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Result<Void> handleValidationException(Exception e) {
        String message = "参数校验失败";
        if (e instanceof MethodArgumentNotValidException) {
            message = ((MethodArgumentNotValidException) e)
                .getBindingResult()
                .getFieldError()
                .getDefaultMessage();
        }
        log.warn("参数校验异常: {}", message);
        return Result.error(400, message);
    }

    /**
     * 系统异常
     */
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public Result<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.error(500, "系统繁忙，请稍后重试");
    }
}
```

---

## 八、第三方 SDK 对接代码生成

### 8.1 阿里云短信 SDK 对接

```markdown
你是一位 Java 集成开发专家

请生成阿里云短信服务的对接代码，要求如下：

【功能需求】
1. 发送短信验证码
2. 验证码 6 位数字
3. 有效期 5 分钟
4. 同一手机号 1 分钟内只能发送一次
5. 存储到 Redis

【技术要求】
- 阿里云 SMS SDK 2.0
- SpringBoot 3.2
- Redis 存储验证码
- 异步发送

【输出】
1. SmsService 接口和实现
2. 配置文件 application.yml
3. Maven 依赖
```

---

**SmsServiceImpl.java：**

```java
package com.example.sms.service.impl;

import com.aliyun.dysmsapi20170525.Client;
import com.aliyun.dysmsapi20170525.models.SendSmsRequest;
import com.aliyun.dysmsapi20170525.models.SendSmsResponse;
import com.example.sms.service.SmsService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.Random;
import java.util.concurrent.TimeUnit;

/**
 * 短信服务实现
 *
 * @author AI Assistant
 * @since 2026-05-17
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class SmsServiceImpl implements SmsService {

    private final Client smsClient;
    private final RedisTemplate<String, String> redisTemplate;

    private static final String VERIFY_CODE_PREFIX = "verify_code:";
    private static final String SEND_LIMIT_PREFIX = "sms_limit:";
    private static final int CODE_LENGTH = 6;
    private static final int CODE_EXPIRE_MINUTES = 5;
    private static final int SEND_LIMIT_SECONDS = 60;

    @Override
    public void sendVerifyCode(String phone) {
        log.info("发送短信验证码: phone={}", phone);

        // 1. 检查发送频率限制
        checkSendLimit(phone);

        // 2. 生成验证码
        String code = generateVerifyCode();

        // 3. 发送短信
        sendSms(phone, code);

        // 4. 存储到 Redis
        storeVerifyCode(phone, code);

        // 5. 记录发送限制
        recordSendLimit(phone);

        log.info("短信验证码发送成功: phone={}", phone);
    }

    /**
     * 检查发送频率限制
     */
    private void checkSendLimit(String phone) {
        String key = SEND_LIMIT_PREFIX + phone;
        Boolean exists = redisTemplate.hasKey(key);
        if (Boolean.TRUE.equals(exists)) {
            throw new BusinessException("发送过于频繁，请稍后再试");
        }
    }

    /**
     * 生成验证码
     */
    private String generateVerifyCode() {
        Random random = new Random();
        int code = random.nextInt(900000) + 100000; // 6位数字
        return String.valueOf(code);
    }

    /**
     * 发送短信
     */
    private void sendSms(String phone, String code) {
        try {
            SendSmsRequest request = new SendSmsRequest()
                .setPhoneNumbers(phone)
                .setSignName("你的签名")
                .setTemplateCode("SMS_XXXXX")
                .setTemplateParam("{\"code\":\"" + code + "\"}");

            SendSmsResponse response = smsClient.sendSms(request);
            
            if (!"OK".equals(response.getBody().getCode())) {
                log.error("短信发送失败: {}", response.getBody().getMessage());
                throw new BusinessException("短信发送失败");
            }
        } catch (Exception e) {
            log.error("短信发送异常", e);
            throw new BusinessException("短信发送失败");
        }
    }

    /**
     * 存储验证码
     */
    private void storeVerifyCode(String phone, String code) {
        String key = VERIFY_CODE_PREFIX + phone;
        redisTemplate.opsForValue().set(key, code, CODE_EXPIRE_MINUTES, TimeUnit.MINUTES);
    }

    /**
     * 记录发送限制
     */
    private void recordSendLimit(String phone) {
        String key = SEND_LIMIT_PREFIX + phone;
        redisTemplate.opsForValue().set(key, "1", SEND_LIMIT_SECONDS, TimeUnit.SECONDS);
    }
}
```

---

## 九、存量代码批量优化技巧

### 9.1 代码规范统一

```markdown
你是一位代码质量专家

请帮我优化以下代码，使其符合阿里巴巴 Java 开发手册规范：

【原始代码】
[粘贴需要优化的代码]

【优化要求】
1. 变量命名规范化（驼峰命名）
2. 魔法值替换为常量
3. 添加必要的注释
4. 简化复杂表达式
5. 提取重复代码
6. 改进异常处理
7. 优化日志输出

【输出格式】
1. 优化后的代码
2. 优化点说明表格
   | 优化项 | 优化前 | 优化后 | 理由 |
```

---

### 9.2 性能优化建议

```markdown
你是一位性能优化专家

请分析以下代码的性能问题，并提供优化方案：

【代码】
[粘贴代码]

【分析维度】
1. 时间复杂度
2. 空间复杂度
3. 数据库查询次数
4. 网络 IO 次数
5. 锁竞争情况

【输出】
1. 性能瓶颈定位
2. 优化方案（多种方案对比）
3. 预估性能提升
4. 优化后的代码
```

---

## 十、完整实战案例

### 10.1 案例：商品管理模块完整开发

**需求：** 开发一个商品管理模块，支持 CRUD 操作

**步骤 1：需求拆解（5 分钟）**

使用 AI 将需求拆解为任务清单

**步骤 2：数据库设计（10 分钟）**

生成商品表、分类表、SKU 表的 SQL

**步骤 3：代码生成（30 分钟）**

- Entity 层：Product、Category、Sku
- Mapper 层：ProductMapper、CategoryMapper、SkuMapper
- Service 层：ProductService（含业务逻辑）
- Controller 层：ProductController（RESTful API）

**步骤 4：单元测试（20 分钟）**

生成 ProductService 的单元测试

**步骤 5：接口文档（5 分钟）**

自动生成 Swagger 文档

**总耗时：70 分钟**（传统开发需要 8-10 小时）

---

## 💡 总结

### 核心要点

1. **需求拆解是关键**：花 10% 的时间拆解需求，节省 50% 的开发时间
2. **标准化提示词**：建立自己的提示词模板库
3. **分步生成**：复杂功能分步骤让 AI 生成
4. **人工审查**：AI 生成的代码必须经过人工审查
5. **持续优化**：根据实际使用情况优化提示词

### 下一步行动

1. 建立个人提示词库
2. 团队共享最佳实践
3. 持续跟进 AI 工具更新
4. 探索更多应用场景

---

*最后更新: 2026-05-17*

*作者：洛苡苑香 | Java 工程师转型 AI 应用开发中*
