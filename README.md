2019.07.02

环境：SpringBoot 2.1.0.RELEASE jdk1.8   SpringCloud Greenwich.SR1  consul服务发现与注册  redis

目前完成：

     1、请求前客户端信息完整性校验 
     2、自定义异常返回
     3、无权处理器  
     4、token失效处理器
     5、自定义token返回信息
     6、根据请求URI拦截权限判断
     7、登录登出自定义
     8、用户名密码登录以及手机号码登录
     9、 swagger 集成 OAuth2 
     10、Feign 权限处理
     

最近在使用SpringCloud 开发权限管理项目，于是设计了一个SpringSecurity OAuth2的统一安全认证。
1、环境   SpringBoot 2.1.0.RELEASE jdk1.8   SpringCloud Greenwich.SR1  consul服务发现与注册  
2、项目划分

        （1）、fp-commons  jar公用
        （2）、fp-gateway 网关
        （3）、fp-resource-manager 用户信息管理资源服务器
        （4）、fp-authorization-server OAuth2认证服务器
        （5）、fp-log 日志
3、 项目搭建     
   （1）网关搭建 frame-gateway    
   
        pom.xml：
            引入依赖文件；在启动器上添加 注册监听@EnableDiscoveryClient 
            并注入 DiscoveryClientRouteDefinitionLocator 
        application.xml： 
            spring-main-allow-bean-definition-overriding: true （相同名称注入允许覆盖）
            spring.application.name:SpringCloud-consul-gateway   （设置应用名称必须）
            bootstrap.yml 中配置 consul（比application先加载） 
            prefer-ip-address: true （使用ip注册，有些网络会出现用主机名来获取注册的问题）
   (2)fp-commons  jar公用 
   
        pom.xml：
              引入依赖文件；添加一些公用方法
   (3)fp-authorization-server
   
   （git中提交了一个至简版本的OAuth2认证中心，只有token生成，没有资源保护以及4种获取token方式（password、authorization_code、refresh_token、client_credentials））
        
        只需要配置AuthorizationServerConfigurerAdapter 认证服务器
        以及WebSecurityConfigurerAdapter SpringSecurity配置 两个文件，就能实现token生成。
        如果没有需要保护的资源不用ResourceServerConfigurerAdapter 资源服务器配置
        
   AuthorizationServerConfigurerAdapter  认证适配器有三个主要的方法：
   
   1、AuthorizationServerSecurityConfigurer：配置令牌端点（Token Endpoint）的安全约束 
   
   （这里我配置了一个处理请求携带客户端信息是否完整的过滤器）
   
   2、ClientDetailsServiceConfigurer:配置客户端详细服务， 客户端的详情在这里进行初始化
   
   （这里我新建了MyClientDetailsService自定义重写了ClientDetailsService接口的loadClientByClientId方法）
   
        客户端数据是通过ClientDetailsService接口来管理的，默认有两种方式分别是
        InMemoryClientDetailsServiceBuilder  写入内存
        JdbcClientDetailsServiceBuilder      写入和数据库
         
   3、AuthorizationServerEndpointsConfigurer：配置授权（authorization）以及令牌（token）的访问端点和令牌服务（token services）
   
   （这里我新建了MyUserDetailsService 自定义重写了UserDetailsService的 loadUserByUsername方法）
   
        用户数据是通UserDetailsService接口来管理的，默认有两种凡是分别是 
        InMemoryUserDetailsManager
        JdbcUserDetailsManager（有默认的表结构需要在数据库中新建）两种对用户数据的管理实现方式
        
   
   基于自己的项目，我们可能需要定制化一些自己的功能：
   
   1、请求前客户端信息完整性校验  
      对于携带数据不完整的请求，可以直接返回给前端，不需要经过后面的验证
      新建一个ClientDetailsAuthenticationFilter 过滤器，在安全约束中加载到过滤链中
   
   2、自定义异常返回
   
        默认异常翻译为这种形式：
        {
            "error": "invalid_grant",
            "error_description": "Bad credentials"
        }
        我们期望的格式： 
        {
            "code":401,
            "msg":"msg"
        }
   
   
   新建MyOAuth2WebResponseExceptionTranslator实现 WebResponseExceptionTranslator接口
   
   重写ResponseEntity<Oauth2Exception> translate(Exception e)方法； 认证发送的异常在这里捕获，认证发生的异常在这里能捕获到，在这里我们可以将我们的异常信息封装成统一的格式返回即可，这里怎么处理因项目而异，这里我直接复制了DefaultWebResponseExceptionTranslator 实现方法
   
   定义自己的OAuth2Exception  MyOAuth2Exception
   
   定义异常的MyOAuth2Exception的序列化类 MyOAuth2ExceptionJacksonSerializer

   将定义好的异常处理  加入到授权服务器的 AuthorizationServerEndpointsConfigurer配置中
   
            效果：
            {
                "code": 400,
                "msg": "error=\"invalid_request\", error_description=\"Missing grant type\""
            }    
   3、无权处理器    
       
      默认的无权访问返回格式是：  
      {
          "error": "access_denied",
          "error_description": "不允许访问"
      }
      我们期望的格式是：
      {
         "code":401,
         "msg":"msg"
      }
   新建一个MyAccessDeniedHandler实现AccessDeniedHandler设置返回信息
   
   加入资源配置 ResourceServerConfigurerAdapter
   
   http.exceptionHandling().accessDeniedHandler(accessDeniedHandler);
   
   4、token失效处理器
      
      默认的token无效返回信息是：
      {
          "error": "invalid_token",
          "error_description": "Invalid access token: 78df4214-8e10-46ae-a85b-a8f5247370a"
      }
      我们期望的格式是： 
      {
         "code":403,
         "msg":"msg"
      }
   新建MyTokenExceptionEntryPoint 实现AuthenticationEntryPoint    
   在 ResourceServerConfigurerAdapter 中注入
   
        @Override
         public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
             resources.authenticationEntryPoint(tokenExceptionEntryPoint); // token失效处理器
             resources.resourceId("auth"); // 设置资源id  通过client的 scope 来判断是否具有资源权限
         }       
   5、自定义token返回信息
  
   默认的token返回格式：
  
          {
              "access_token": "1e93bc23-32c8-428f-a126-8206265e17b2",
              "token_type": "bearer",
              "refresh_token": "0f083e06-be1b-411f-98b0-72be8f1da8af",
              "expires_in": 3599,
              "scope": "auth api"
          }     
   我们期望自定义，例如加上username等：
   
        {
              "access_token": "1e93bc23-32c8-428f-a126-8206265e17b2",
              "token_type": "bearer",
              "refresh_token": "0f083e06-be1b-411f-98b0-72be8f1da8af",
              "expires_in": 3599,
              "scope": "auth api",
              "username":"username"
        }            
        
   新建自定义token返回MyTokenEnhancer实现TokenEnhancer接口
   在认证服务的defaultTokenServices()中添加MyTokenEnhancer。需要注意的是：
   如果已经生成了一次没有自定义的token信息，需要去redis里删除掉该token才能再次测试结果，不然你的结果一直是错误的，因为token还没过期的话，是不会重新生成的。
   
   (4)fp-resource-manager  用户信息管理资源服务器
      
      pom.xml：OAuth2、redis等依赖
      把认证服务器中的 资源配置相关复制到资源项目，稍作调整：
      增加健康检查通过，
      
      增加redis注入（重点，不然会取不到token对比一直是token失效； 如果自定义过userDetails，记得把自定义实现的文件也复制一份到资源项目，不然redis反序列化会失败。）
      到此可以使用token进行资源保护。
      
   6、根据请求URI拦截权限判断
    一般我们通过@PreAuthorize("hasRole('ROLE_USER')") 注解，以及在HttpSecurity配置权限需求等来控制权限
    在这里，我们基于请求的URI来控制访问权限，并且可以使用注解来控制权限访问。
    首先 自定义一个权限认证MySecurityAccessDecisionManager 继承AccessDecisionManager接口，重写 decide方法，
    并且复制默认权限验证AbstractAccessDecisionManager的剩余两个方法（实现注解控制的重点）。
    用户具有的权限在认证服务器中已经自定义了。
    
            @Override
            public void decide(Authentication authentication, Object object, Collection<ConfigAttribute> configAttributes) throws AccessDeniedException, InsufficientAuthenticationException {
                String requestUrl = ((FilterInvocation) object).getRequest().getMethod() + ((FilterInvocation) object).getRequest().getRequestURI();
        //        System.out.println("requestUrl>>" + requestUrl);
        
                // 当前用户所具有的权限
                Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
        //        System.out.println("authorities=" + authorities);
                for (GrantedAuthority grantedAuthority : authorities) {
                    if (grantedAuthority.getAuthority().equals(requestUrl)) {
                        return;
                    }
                    if (grantedAuthority.getAuthority().equals("ROLE_ADMIN")) {
                        return;
                    }
        
                }
                throw new AccessDeniedException("无访问权限");
            }
   这里通过判断匹配通过或者抛出无权异常来判断有无访问权限。
   在资源服务器的资源配置中重写权限判断，加载我们自定义的权限判断
   
     .withObjectPostProcessor(new ObjectPostProcessor<FilterSecurityInterceptor>() {       // 重写做权限判断
           @Override
           public <O extends FilterSecurityInterceptor> O postProcess(O o) {
                  o.setAccessDecisionManager(accessDecisionManager);      // 权限判断
                  return o;
           }
       })
   6、1 优化6的权限判断
   先增加一级获取uri然后判断uri需要什么权限，可以多个并且的权限 等等然后 在经过权限判断器进行拦截判断    
   新建自定义的url权限判断MyFilterInvocationSecurityMetadataSource 实现FilterInvocationSecurityMetadataSource
          private AntPathMatcher antPathMatcher = new AntPathMatcher(); // 模糊匹配 如何 auth/**   auth/auth
        
            @Override
            public Collection<ConfigAttribute> getAttributes(Object object) throws IllegalArgumentException {
                Set<ConfigAttribute> set = new HashSet<>();
        
                String requestUrl = ((FilterInvocation) object).getRequest().getMethod() + ((FilterInvocation) object).getRequest().getRequestURI();
                System.out.println("requestUrl >> " + requestUrl);
        
                // 这里获取对比数据可以从数据库或者内存 redis等等地方获取 目前先写死后面优化
                String url = "GET/auth/**";
                if (antPathMatcher.match(url, requestUrl)) {
                    SecurityConfig securityConfig = new SecurityConfig("ROLE_ADMIN");
                    set.add(securityConfig);
                }
                if (ObjectUtils.isEmpty(set)) {
                    return SecurityConfig.createList("ROLE_LOGIN");
                }
                return set;
            }
   修改原来的MySecurityAccessDecisionManager，从获取到url 改成获取到需要什么权限。其他判断不变
       把MyFilterInvocationSecurityMetadataSource 注册到重写方法
       
               .withObjectPostProcessor(new ObjectPostProcessor<FilterSecurityInterceptor>() {       // 重写做权限判断
                       @Override
                       public <O extends FilterSecurityInterceptor> O postProcess(O o) {
                           o.setSecurityMetadataSource(filterInvocationSecurityMetadataSource); // 请求需要权限
                           o.setAccessDecisionManager(accessDecisionManager);      // 权限判断
                       return o;
                    }
               })
           
   7、登录登出自定义
        
   登出自定义：登出相当于使token失效，我们只需要携带access_token 请求 ConsumerTokenServices（默认自带的注销接口）即可，请求后旧的token即不可用。
            
            @DeleteMapping("/logout")
            public ResponseVo logout(String accessToken) {
                if (consumerTokenServices.revokeToken(accessToken)) {
                    return new ResponseVo(200, "登出成功");
                } else {
                    return new ResponseVo(500, "登出失败");
                }
            }
   登录自定义：默认的token请求地址是“oauth/token”,我们可以在认证服务配置的 AuthorizationServerEndpointsConfigurer配置中自定义的请求地址。
   我们还可以封装一层自己的请求，然后在请求token之前做一些自己的处理。我这里使用了获取需要请求token的信息，然后在java端使用RestTemplate来调用生成token地址
   的方式来获取token，我可以在调用之前做一些其他处理，如验证码校验等等。
   具体看TokenController文件。
   
   7、1登录优化-增加验证码
   
   登录设置验证码，验证码有效期为1分钟，登录成功后或者到达最大时间后验证码即失效。验证码以用户名_code 的关键字保存在redis中，并设置失效时间
   用户名+验证码匹配通过后才进行下一步的token生成，生成token后，即删除验证码。
   
    1.生成带用户名的code 存入redis 设置过期时间
    2.生成token前对比验证码。
    
   8、其他方式实现用户名密码登录以及手机号码登录
    手机验证码登录 
   新建MyAuthenticationToken 自定义AbstractAuthenticationToken
   
   新建 MyPhoneAuthenticationToken（手机验证码登录用） 继承 MyAuthenticationToken 
   
   新建MyAbstractUserDetailsAuthenticationProvider 抽象类 实现AuthenticationProvider
   
   新建 MyPhoneAuthenticationProvider（手机验证码登录 ）继承 MyAbstractUserDetailsAuthenticationProvider ,在这里使用MyPhoneAuthenticationToken，注入参数
   
    这里负责手机验证码的校验。在这里设置各种错误信息 (在MyLoginAuthFailureHandler 登录失败处理器中处理异常返回信息)
   
   修改MyUserDetailsService，改为抽象类，使用模板方法模式： protected abstract AuthUser getUser(String var1); 来获取不同数据来源
   
   新建MyUsernameUserDetailsService继承 MyUsernameUserDetailsService 该方法为原来的登录提供数据   
   
   新建MyPhoneUserDetailsService 继承MyUsernameUserDetailsService 该方法为新的手机验证码登录提供数据。
   
   新建MyLoginAuthSuccessHandler 登录成功处理器，该方法用于验证client信息 并返回token信息。
   
   新建MyPhoneLoginAuthenticationFilter 手机验证码登录过滤器，拦截登录的url，进行数据注入到MyPhoneAuthenticationToken。
   
   修改 MySecurityOAuth2Config ，因为修改了  MyUserDetailsService 接口，无法为原来的登录方式提供数据，所以改为MyUsernameUserDetailsService来提供数据
   
   重点 修改MySecurityConfig 配置，装配 两个数据接口，装配 登录成功处理器  装配配置，以及把MyPhoneLoginAuthenticationFilter加入到过滤链。
   
   手机验证码模式登录这里完成，用户名密码登录可以参照该流程自行完成
 
   8、1 优化自定义异常返回格式， 增加登录失败处理器，返回失败异常 
   
   9、 swagger 集成 OAuth2 
   
       pom.xml增加  swagger依赖 
       application中增加swagger的配置
       增加配置文件 SwaggerConfig  
       有两种模式  注解掉的是提交增加一个Authorization 提交token框的方式
       方式二：增加token登录框采用登录的方式进行token校验。 
       Swagger 配置登录相关  application配置 登录url
       MySecurityResourceServerConfig 放行 
      
       测试地址：http://192.168.3.14:8001/manager/swagger-ui.html#/
       
   10、Feign 权限处理
        
        新建一个fp-resource-feign服务
        pom.xml增加Feign的依赖
        Feign 有一个RequestInterceptor接口，对该接口做一些简单的实现，FeignConfig，对请求增加token信息实现授权调用服务接口。
        
     