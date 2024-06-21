
---
title: 利用druid对数据库密码进行加密
tags:
  - druid
status: Done
createDate: 2022-03-10
---

---

在生产环境中，直接在配置文件中暴露明文密码是一件非常危险的事情，出于两点考虑：对外，即使应用服务被入侵，数据库还是安全的；对内，生产环境的数据库密码理论上应该只有 dba 知道，但是代码都是在代码仓库中放着的，如果密码没有加密，每次发布前 dba 都需要手动修改配置文件后再进行打包编译。

首先，我们需要生成数据库密码的密文，需要在命令行中执行如下命令：

`java -cp ~/.m2/comdruid-1.0.16.jar com.alibaba.druid.filter.config.ConfigTools 你的密码`

```bash
$ java -cp ~/.m2/comdruid-1.0.16.jar com.alibaba.druid.filter.config.ConfigTools littlefxc.Fxc123

# 私钥
**privateKey**:MIIBVQIBADANBgkqhkiG9w0BAQEFAASCAT8wggE7AgEAAkEAgP7oJvENw0DfwK3hVOE3ADer9zgjiCPnZ45qxdYe6ohRwTFILic8koCCbl/t26LPhS8cJnSUT/5n3C7VfZWf2wIDAQABAkBzWX5nNC9GZoCvX82bhTkVrLLOAxli6BhJdgTsnChROAg4EkH3WKj7bzKEBLfaTTbY+U2zoqp9N7VtM9WWvcfBAiEAuLQZeiSF/VvWQoGg82LwOZ8a5X01ybt0ySltOC3SfOkCIQCyyepHOYwjSnqXfIlRmODrYrb2/BESdyxIsO7/aZKsIwIhAIiMDLGxwqTVegbc0mJcaIAQ0c+Ky3MB9Iqq56W6qnvRAiBOA6dT3vuUZqppsbDlxxTWAWQfD8yPRysuqO4Qy0tyCwIhAI2WxBanjKRkO8mxaCNFSTtNJVF+w8IjO15ewS1L0Xx/
# 公钥
**publicKey**:MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBAID+6CbxDcNA38Ct4VThNwA3q/c4I4gj52eOasXWHuqIUcExSC4nPJKAgm5f7duiz4UvHCZ0lE/+Z9wu1X2Vn9sCAwEAAQ==
# 密文
**password**:Pp2LSQxi6F9AvkgqZq0zutVadnAqNjNw+iO40tBnnzb8MPAGeBlhV4wSPSy5Xdcl+btJP2GEblNerzzbhD3LjA==
```

这里我们需要将生成的公钥 publicKey 和密码 password 加入配置文件中， 如下所示：

```
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
spring.datasource.url=jdbc:mysql://localhost:3306/foodie-shop-dev?serverTimezone=Asia/Shanghai&useUnicode=true&characterEncoding=UTF-8&useSSL=false
spring.datasource.username=mysql
# 加密后密文，原密码为  littlefxc.Fxc123
spring.datasource.password=Pp2LSQxi6F9AvkgqZq0zutVadnAqNjNw+iO40tBnnzb8MPAGeBlhV4wSPSy5Xdcl+btJP2GEblNerzzbhD3LjA==
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.druid.filter.config.enabled=true
# 加密后的 publicKey
spring.datasource.druid.connection-properties=config.decrypt=true;config.decrypt.key=MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBAID+6CbxDcNA38Ct4VThNwA3q/c4I4gj52eOasXWHuqIUcExSC4nPJKAgm5f7duiz4UvHCZ0lE/+Z9wu1X2Vn9sCAwEAAQ==
```

到此为止，已经完成了使用druid加密数据库密码。验证一下：

数据模型：

```java
package com.fengxuechao.example.druid;

import lombok.Data;

import javax.persistence.*;
import java.util.Date;

@Data
@Table(name = "users")
public class Users {
    /**
     * 主键id 用户id
     */
    @Id
    private String id;

    /**
     * 用户名 用户名
     */
    private String username;

    /**
     * 密码 密码
     */
    private String password;

    /**
     * 昵称 昵称
     */
    private String nickname;

    /**
     * 真实姓名
     */
    private String realname;

    /**
     * 头像
     */
    private String face;

    /**
     * 手机号 手机号
     */
    private String mobile;

    /**
     * 邮箱地址 邮箱地址
     */
    private String email;

    /**
     * 性别 性别 1:男  0:女  2:保密
     */
    private Integer sex;

    /**
     * 生日 生日
     */
    private Date birthday;

    /**
     * 创建时间 创建时间
     */
    @Column(name = "created_time")
    private Date createdTime;

    /**
     * 更新时间 更新时间
     */
    @Column(name = "updated_time")
    private Date updatedTime;
}
```

主程序：

```java
package com.fengxuechao.example.druid;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.jdbc.core.BeanPropertyRowMapper;
import org.springframework.jdbc.core.JdbcTemplate;

import java.util.List;

/**
 * @author fengxuechao
 * @date 2020/10/15
 */
@Slf4j
@SpringBootApplication
public class DruidSecurityApp implements CommandLineRunner {

    public static void main(String[] args) {
        SpringApplication.run(DruidSecurityApp.class);
    }

    @Autowired
    private JdbcTemplate jdbcTemplate;

    /**
     * Callback used to run the bean.
     *
     * @param args incoming main method arguments
     * @throws Exception on error
     */
    @Override
    public void run(String... args) throws Exception {
        List<Users> users = jdbcTemplate.query("select * from users", new BeanPropertyRowMapper<>(Users.class));
        log.info("输出结果: {}", users);
    }
}
```

输出结果：

![druid_encode_result](https://img-blog.csdnimg.cn/img_convert/c14fe35779d058e806d1c8874e16da58.png)