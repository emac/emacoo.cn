---
title: '【Spring】关于Boot应用中集成Spring Security你必须了解的那些事'
published: true
date: '06-03-2016 00:00'
visible: true
summary:
    enabled: '1'
    format: short
taxonomy:
    tag:
        - 原创
        - boot
---

#### Spring Security

Spring Security是Spring社区的一个顶级项目，也是Spring Boot官方推荐使用的Security框架。除了常规的Authentication和Authorization之外，Spring Security还提供了诸如ACLs，LDAP，JAAS，CAS等高级特性以满足复杂场景下的安全需求。虽然功能强大，Spring Security的配置并不算复杂（得益于官方详尽的文档），尤其在3.2版本加入Java Configuration的支持之后，可以彻底告别令不少初学者望而却步的XML Configuration。在使用层面，Spring Security提供了多种方式进行业务集成，包括注解，Servlet API，JSP Tag，系统API等。下面就结合一些示例代码介绍Boot应用中集成Spring Security的几个关键点。

##### 1 核心概念

Principle(User), Authority(Role)和Permission是Spring Security的3个核心概念。跟通常理解上Role和Permission之间一对多的关系不同，在Spring Security中，Authority和Permission是两个完全独立的概念，两者并没有必然的联系，但可以通过配置进行关联。

##### 2 基础配置

首先在项目的pom.xml中引入spring-boot-starter-security依赖。

	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-security</artifactId>
	</dependency>

和其余Spring框架一样，XML Configuration和Java Configuration是Spring Security的两种常用配置方式。Spring 3.2版本之后，Java Configuration因其流式API支持，强类型校验等特性，逐渐替代XML Configuration成为更广泛的配置方式。下面是一个示例Java Configuration。

    @Configuration
    @EnableWebSecurity
    @EnableGlobalMethodSecurity(prePostEnabled = true)
    public class SecurityConfig extends WebSecurityConfigurerAdapter {

        @Autowired
        MyUserDetailsService detailsService;

        @Override
        protected void configure(HttpSecurity http) throws Exception {
            http.authorizeRequests()
                    .and().formLogin().loginPage("/login").permitAll().defaultSuccessUrl("/", true)
                    .and().logout().logoutUrl("/logout")
                    .and().sessionManagement().maximumSessions(1).expiredUrl("/expired")
                    .and()
                    .and().exceptionHandling().accessDeniedPage("/accessDenied");
        }

        @Override
        public void configure(WebSecurity web) throws Exception {
            web.ignoring().antMatchers("/js/**", "/css/**", "/images/**", "/**/favicon.ico");
        }

        @Override
        public void configure(AuthenticationManagerBuilder auth) throws Exception {
            auth.userDetailsService(detailsService).passwordEncoder(new BCryptPasswordEncoder());
        }

    }

- @EnableWebSecurity: 禁用Boot的默认Security配置，配合@Configuration启用自定义配置（需要扩展WebSecurityConfigurerAdapter）
- @EnableGlobalMethodSecurity(prePostEnabled = true): 启用Security注解，例如最常用的@PreAuthorize
- configure(HttpSecurity): Request层面的配置，对应XML Configuration中的`<http>`元素
- configure(WebSecurity): Web层面的配置，一般用来配置无需安全检查的路径
- configure(AuthenticationManagerBuilder): 身份验证配置，用于注入自定义身份验证Bean和密码校验规则

##### 3 扩展配置

完成基础配置之后，下一步就是实现自己的UserDetailsService和PermissionEvaluator，分别用于自定义Principle, Authority和Permission。

	@Component
	public class MyUserDetailsService implements UserDetailsService {

        @Autowired
        private LoginService loginService;

        @Autowired
        private RoleService roleService;

        @Override
        public UserDetails loadUserByUsername(String username) {
            if (StringUtils.isBlank(username)) {
                throw new UsernameNotFoundException("用户名为空");
            }

            Login login = loginService.findByUsername(username).orElseThrow(() -> new UsernameNotFoundException("用户不存在"));

            Set<GrantedAuthority> authorities = new HashSet<>();
            roleService.getRoles(login.getId()).forEach(r -> authorities.add(new SimpleGrantedAuthority(r.getName())));

            return new org.springframework.security.core.userdetails.User(
                    username, login.getPassword(),
                    true,//是否可用
                    true,//是否过期
                    true,//证书不过期为true
                    true,//账户未锁定为true
                    authorities);
        }
    }

> 创建GrantedAuthority对象时，一般名称加上ROLE\_前缀。

```
	@Component
	public class MyPermissionEvaluator implements PermissionEvaluator {

		@Autowired
		private LoginService loginService;

		@Autowired
		private RoleService roleService;

		@Override
    	public boolean hasPermission(Authentication authentication, Object targetDomainObject, Object permission) {
            String username = authentication.getName();
            Login login = loginService.findByUsername(username).get();
            return roleService.authorized(login.getId(), targetDomainObject.toString(), permission.toString());
    	}

	    @Override
    	public boolean hasPermission(Authentication authentication, Serializable targetId, String targetType, Object permission) {
		    // not supported
	    	return false;
    	}
    }
```

- hasPermission(Authentication, Object, Object)和hasPermission(Authentication, Serializable, String, Object)两个方法分别对应Spring Security中两个同名的表达式。

##### 4 业务集成

Spring Security提供了注解，Servlet API，JSP Tag，系统API等多种方式进行集成，最常用的是第一种方式，包含@Secured, @PreAuthorize, @PreFilter, @PostAuthorize和@PostFilter五个注解。@Secure是最初版本中的一个注解，自3.0版本引入了支持Spring EL表达式的其余四个注解之后，就很少使用了。

	@RequestMapping(value = "/hello", method = RequestMethod.GET)
    @PreAuthorize("authenticated and hasPermission('hello', 'view')")
    public String hello(Model model) {
    	String username = SecurityContextHolder.getContext().getAuthentication().getName();
        model.addAttribute("message", username);
        return "hello";
    }
    
- @PreAuthorize("authenticated and hasPermission('hello', 'view')"): 表示只有当前已登录的并且拥有("hello", "view")权限的用户才能访问此页面
- SecurityContextHolder.getContext().getAuthentication().getName(): 获取当前登录的用户，也可以通过HttpServletRequest.getRemoteUser()获取

#### 总结

以上就是Spring Security的一般集成步骤，更多细节和高级特性可参考官方文档。

#### 参考

- http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/
- http://docs.spring.io/spring-security/site/docs/current/reference/htmlsingle/