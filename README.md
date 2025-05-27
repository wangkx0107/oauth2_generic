## 说明
该项目只要是为了配合集成gitlab的oauth2，项目的框架是基于spring+oauth2修改而来

## 环境配置
- 本地安装maven和java
  以MAC M1为例，安装maven和java
  ```
  brew install maven openjdk@17
  ```
  配置环境变量
  ```
  echo 'export JAVA_HOME=$(/usr/libexec/java_home -v17)' >> ~/.zshrc
  echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.zshrc
  source ~/.zshrc
  ```
  查看
  ```
  mvn -v
    Apache Maven 3.9.9 (8e8579a9e76f7d015ee5ec7bfcdc97d260186937)
    Maven home: /opt/homebrew/Cellar/maven/3.9.9/libexec
    Java version: 21.0.2, vendor: Oracle Corporation, runtime: /Library/Java/JavaVirtualMachines/jdk-21.jdk/Contents/Home
    Default locale: zh_CN_#Hans, platform encoding: UTF-8
    OS name: "mac os x", version: "15.2", arch: "aarch64", family: "mac"
  java --version
    java 21.0.2 2024-01-16 LTS
    Java(TM) SE Runtime Environment (build 21.0.2+13-LTS-58)
    Java HotSpot(TM) 64-Bit Server VM (build 21.0.2+13-LTS-58, mixed mode, sharing)
  ```
## 如何使用
- clone项目到本地
- 服务端配置修改
  - 修改 `auth-server/src/main/java/com/example/authserver/config/AuthServerConfig.java` 的redirectUris，添加或修改极狐GitLab的CallBack URL，相当于给OAuth2 SSO服务添加可信的重定向URL。
    ```
    public void configure(final ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .withClient("SampleClientId")
                .secret(passwordEncoder.encode("secret"))
                .authorizedGrantTypes("authorization_code")
                .scopes("user_info")
                .autoApprove(true)
                .redirectUris("http://192.168.1.47:8301/login", "http://192.168.1.47:8302/login","http://192.168.1.100/users/auth/oauth2_generic/callback");
    }
    ```
    其中 http://192.168.1.47:8301/login、http://192.168.1.47:8302/login为客户端访问的 url
        http://192.168.1.100/users/auth/oauth2_generic/callback 为 注册应用程序时提供的重定向 URI 应该是，其中 http://192.168.1.100 为极狐 gitlab 的访问 url
    
  - 修改服务端口 `auth-server/src/main/resources/application.yml`。
    ```
    server:
      address: '0.0.0.0'
      port: 8300
      servlet:
        context-path: '/auth'
    ```
  - 修改用户名和密码 `auth-server/src/main/java/com/example/authserver/config/SecurityConfig.java`。
    ```
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
                .withUser("test@123.com")
                .password(passwordEncoder().encode("123"))
                .roles("USER");
    }
    ```
- 可选（主要便于进行本地测试）
  - 修改 `client-a/src/main/resources/application.yml` （客户端配置修改(client-a和client-b 一样）。
    ```
    server:
      address: '0.0.0.0'
      port: 8301
      servlet:
        session:
          cookie:
            name: CLIENT_A_SESSION
    
    security:
      oauth2:
        client:
          client-id: SampleClientId
          client-secret: secret
          access-token-uri: http://192.168.1.47:8300/auth/oauth/token
          user-authorization-uri: http://192.168.1.47:8300/auth/oauth/authorize
        resource:
          user-info-uri: http://192.168.1.47:8300/auth/user/me # 从授权服务器获取当前登录用户信息的地址
    
    spring:
      thymeleaf:
        cache: false
    ```

- 项目跟路径下
  ```
  mvn clean install
  ```
- 启动服务端 
  ```
  # auth-server http://192.168.1.47:8300
  cd ./auth-server
  mvn spring-boot:run
  ```
- 启动客户端
  ```
  # client-a http://192.168.1.47:8301
  cd ./client-a
  mvn spring-boot:run
  # client-b http://192.168.1.47:8302
  cd ./client-b
  mvn spring-boot:run
  ```
  <img width="1726" alt="image" src="https://github.com/user-attachments/assets/676543de-cd96-4916-b236-a613fc374a5b" />

## 获取用户数据结构
该步骤用于获取OAuth2的user-info-uri返回的数据结构，这里可以用Postman操作。
- 认证方式选OAuth2.0，根据上一章节的配置填写OAuth2的相关参数，然后点Get New Access Token
  <img width="1667" alt="image" src="https://github.com/user-attachments/assets/242dd946-b640-4a55-b7ba-4005d44db84b" />
- Postman会弹窗进入OAuth2 SSO服务的登录页面，输入用户账号和密码，确认是否认证成功
  <img width="744" alt="image" src="https://github.com/user-attachments/assets/3cadb5ea-de66-4988-bf60-c12bf4349133" />
  <img width="1671" alt="image" src="https://github.com/user-attachments/assets/129c930e-7315-4046-afbe-2369f7f203eb" />
- 发送请求，获取响应结果，确认必须是Json格式
  <img width="1678" alt="image" src="https://github.com/user-attachments/assets/d12a8c5f-8213-493b-a397-e6455708af7f" />
## 客户端运行测试
- 访问client-a
  ```
  http://192.168.1.47:8301
  ```
- 自动跳转到auth-server
  <img width="856" alt="image" src="https://github.com/user-attachments/assets/8dc9feed-827d-4074-b9d2-222e22e4ba6a" />
- 登录用户
  <img width="1262" alt="image" src="https://github.com/user-attachments/assets/f5737f46-9a87-4e5c-b5bb-ff5a6c04e8a8" />
- 跳转回client-a,并完成登录认证
  <img width="1091" alt="image" src="https://github.com/user-attachments/assets/ebc233a7-cc13-4f20-a26e-a8484f802375" />

## 配置极狐GitLab
该步骤用于配置极狐GitLab与OAuth2对接并实现SSO,参考[官方文档](https://docs.gitlab.com/integration/oauth2_generic/#configure-the-oauth-20-provider)。

极狐GitLab对接OAuth2的限制
  - 只能用于单点登录，不会提供任何OAuth Provider授予的其他访问权限(例如导入项目或用户等)
  - 只支持授权授予流程(最常见的客户端-服务器应用程序，如Rails应用程序)
  - 不能从多个URL获取用户信息
  - 不支持JSON以外的用户信息格式

```
gitlab_rails['omniauth_allow_single_sign_on'] = ['oauth2_generic']
# 使用OAuth登录的用户需管理员审批；如果需要审批的话，可以改成  `true`
gitlab_rails['omniauth_block_auto_created_users'] = false
gitlab_rails['omniauth_providers'] = [
  {
    "name" => "oauth2_generic",
    # 显示在GitLab登陆页面的SSO登录按钮的文字
    "label" => "SSO",
    # client_id
    "app_id" => "SampleClientId",
    # client_secret
    "app_secret" => "secret",
    args: {
      client_options: {
        # OAuth SSO 登录认证URL
        site: "http://192.168.1.47:8300",
        # OAuth 各服务的URL
        user_info_url: "/auth/user/me",
        authorize_url: "/auth/oauth/authorize",
        token_url: "/auth/oauth/token"
      },
      # 对应上一章节用户信息数据结构
      user_response_structure: {
        # root_path用于逐层解析用户信息的Json，直到包含用户信息的节点。以上一章节的响应结果为例，用户名username在Json的/userAuthentication/principal节点下，对应root_path配置如下
        root_path: ['userAuthentication','principal'],
        # id_path是相对于root_path节点下的某个属性，作为GitLab用户的唯一id。以上一章节的响应结果为例，由于principal节点只包含了username，所以以username作为id，对应id_path配置如下
        id_path: 'username',
        # attributes是将root_path节点下的各个属性映射为标准Omniauth的用户属性，具体见 https://github.com/omniauth/omniauth/wiki/auth-hash-schema#schema-10-and-later
        # 以上一章节的响应结果为例，由于principal节点只包含了username，且username是邮箱账号，所以可以将name和email都可以映射到username
        attributes: { name: 'username',email: 'username'}
      },

      strategy_class: "OmniAuth::Strategies::OAuth2Generic"
    }
  }
]
```

```
gitlab-ctl reconfigure
```

- 登录 gitlab
  <img width="1668" alt="image" src="https://github.com/user-attachments/assets/ce0b9e8f-d025-472b-9ed5-399fbc32b001" />
- 自动跳转到OAuth SSO服务
  <img width="1471" alt="image" src="https://github.com/user-attachments/assets/62b44973-cfc1-42bb-8bf8-f2de48bb94cb" />
- 登录认证成功，返回极狐GitLab，并自动创建用户
  <img width="1726" alt="image" src="https://github.com/user-attachments/assets/61bba4d2-6648-4707-8460-7ff43d03aa8e" />

  

