# 快速入门

## 关于AdcFramework

**AdcFramework** 全称 **Ace Development Core Framework** 翻译过来就是 **王牌开发核心框架 。**

由于本人是 **AbpVNext** 的重度依赖患者，对 **Abp** 的模块化思想是推崇备至，于是在 **AdcFramework** 中也借鉴了其模块化思想。但是由于  **AbpVNext** 框架极重，全局AOP又使得其运行效率校队较低，在多放权衡及工作需要的情况下，决定开发一款新的集成模块化思想的框架， **AdcFramework** 便运应而生。

**AdcFramework** 是基于ASP.NET Core的Web应用程序开发，目前尚不支持其他类型的应用程序.


## 在AspNet Core MVC Web Application中使用


### 创建第一个 AdcFramework 项目

使用Visual Studio 2019 (16.4.0+) 或 Visual Studio Code 创建一个新的 AspNet Core WebApi 项目


### 安装 Youshow.Adc.AspNetCore.Web 包

Youshow.Adc.AspNetCore.Web 是 AdcFramework 集成 AspNet Core MVC 以及 AspNet Core WebApi 的包，请安装它到你项目中:

```
Install-Package Youshow.Adc.AspNetCore.Web
```



### 创建ADC模块

```cs
    [RelyOn(
        typeof(AdcAspNetCoreWebModule)
    )]
    public class YoushowDemoApiModule : AdcModule
    {
        public override void ConfigureServices(ServiceConfigurationContext context)
        {
            var services = context.Services;
            services.AddSwaggerGen(c =>
            {
                c.SwaggerDoc("v1", new OpenApiInfo { Title = "Youshow.Demo.Api", Version = "v1" });
                c.DocInclusionPredicate((_, _) => true);
            });
        }
        public override void OnApplicationInitialization(ApplicationInitializationContext context)
        {
            var env = context.GetEnvironment();
            var app = context.GetApplicationBuilder();
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
                app.UseSwagger();
                app.UseSwaggerUI(c => c.SwaggerEndpoint("/swagger/v1/swagger.json", "Youshow.Demo.Api v1"));
            }
        }
    }
```

> **小贴士**
>
> 在创建模块时，重写 `ConfigureServices` 方法和 `OnApplicationInitialization` 方法，再将 StartUp 中 `ConfigureServices` 和 `Configure` 方法中的内容对应复制进来即可。但是一些通用配置无需复制，在 `Youshow.Adc.AspNetCore.Web` 模块中已经集成

`AdcModule` 是应用程序启动模块的名称.

AdcFramework 框架定义了这个模块类，模块可以依赖其它模块。在上面的代码中 `AdcModule` 依赖于 `AdcAspNetCoreWebModule` (模块存在于**Youshow.Adc.AspNetCore.Web** 包中)。模块可以使用 `RelyOn()` 用来依赖其他模块。


### 启动类

接下来修改启动类集成AdcFramework模块系统

```cs
    public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        // This method gets called by the runtime. Use this method to add services to the container.
        public void ConfigureServices(IServiceCollection services)
        {
          services.AddServiceEntrance<YoushowDemoApiModule>();
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
           app.InitServiceEntrance();
        }
    }
```

`services.AddServiceEntrance<YoushowDemoApiModule>()` 添加了所有 `AdcModule` 模块中定义的全部服务.

`Configure`方法中的 `app.InitServiceEntrance()` 完成初始化并启动应用程序.

### 扩展容器功能
修改 `Program.cs` 来使用容器扩展

```cs
    public class Program
    {
        public static void Main(string[] args)
        {
            CreateHostBuilder(args).Build().Run();
        }
   
        public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder.UseStartup<Startup>();
                })
                .UseAdcContainer(); // 添加这一行
    }
```

### 运行应用程序

F5启动调试，它将正常启动项目


# 使用扩展容器

虽然AspNet Core的依赖注入(DI)容器适用于基本要求,但未提供属性注入等功能，这些功能是AdcFramework执行某些功能所必需的。

对AspNet Core的DI容器做扩展并集成到AdcFramework非常简单



## 扩展AspNet Core的依赖注入(DI)容器

### 安装 Youshow.Adc.Container包

```
Install-Package Youshow.Adc.Container
```

### 扩展容器功能

1. 修改 `Program.cs` 来使用容器扩展

   ```cs
       public class Program
       {
           public static void Main(string[] args)
           {
               CreateHostBuilder(args).Build().Run();
           }
   
           public static IHostBuilder CreateHostBuilder(string[] args) =>
               Host.CreateDefaultBuilder(args)
                   .ConfigureWebHostDefaults(webBuilder =>
                   {
                       webBuilder.UseStartup<Startup>();
                   })
                   .UseAdcContainer(); // 添加这一行
       }
   ```
2. 开启 控制器 属性注入和字段注入功能

   ```cs
     services.Configure<AdcAspNetCoreWebOptions>(opt =>
     {
        opt.AddControllerInContainer(true); // 添加控制器的容器扩展，以获取属性注入功能
     });
   ```



## 使用属性注入

在类中直接加入属性或字段，当该类通过容器创建时，它自身的属性和所有父类的属性也会被一起注入，除非该属性未再容器中注册

### 当前类中的属性注入

```C#
    [ApiController]
    [Route("[controller]")]
    public class WeatherForecastController : BaseController
    {
        private static readonly string[] Summaries = new[]
        {
            "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
        };

        // 在这里添加控制器属性
        public IUserAppService UserApp { get; set; }

        [HttpGet]
        public List<UserDto> Get()
        {
            return UserApp.GetUserName();
        }
    }
```

### 特性标记禁用属性注入

如果所有属性都在所在类通过容器创建时被连带注入，那么是会浪费性能的，有时候也是没有必要的，这个时候就可以使用特性 `[ForbidWrite]` 去修饰不需要被注入的属性，此时属性所在类通过容器创建时，被特性修饰的属性就不会被注入。

```C#
    [ApiController]
    [Route("[controller]")]
    public class WeatherForecastController : BaseController
    {
        private static readonly string[] Summaries = new[]
        {
            "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
        };

        [ForbidWrite] // 被这个特性修饰的属性或字段将不会被注入
        public IUserAppService UserApp { get; set; }

        [HttpGet]
        public List<UserDto> Get()
        {
            return UserApp.GetUserName();
        }
    }
```



### 父类中的属性注入

如果这个属性是在父类中被声明的，那么在子类被容器创建时一样可以使用

1. 在父类中声明

   ```C#
       public class BaseController:ControllerBase
       {
             //在BaseController中声明属性
             public IUserAppService UserApp { get; set; }
       }
   ```

2. 在子类中使用

   ```C#
       [ApiController]
       [Route("[controller]")]
       public class WeatherForecastController : BaseController
       {
           private static readonly string[] Summaries = new[]
           {
               "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
           };
   
           [HttpGet]
           public List<UserDto> Get()
           {
               return UserApp.GetUserName();
           }
       }
   ```

   

## 使用字段注入

除此以外，字段在 `AdcFramework` 框架中也可以被直接注入，同样可以通过 `[ForbidWrite]` 特性阻止其被注入。使用方法和属性注入一致。

> 小贴士
>
> 从这里我们可以发现，由于 **AdcFramework** 中字段也会被自动注入，这就使得构造函数注入彻底沦为了鸡肋，看似可有可无。但是如果我们需要在类执行构造函数时，就要执行某些类型的实例对象，这个时候才需要使用构造函数注入。



# 数据访问

目前本框架只实现了以 **Entity Framework Core** 作为数据访问的 ORM 框架，后期将集成更多

## Entity Framework Core 集成

如果我们需要集成 **Entity Framework Core** 只需要引入其模块就行，当然如果你觉得项目需要层，也可以另外创建一个数据访问层，再集成 **Entity Framework Core** 模块即可。

目前 **Entity Framework Core** 只适配了 **MySql** 和 **SqlServer** 模块，后期将适配更多模块

### 引入 Entity Framewrok Core 模块

```
Install-Package Youshow.Adc.EntityFrameworkCore.MySql
或
Install-Package Youshow.Adc.EntityFrameworkCore.SqlServer
```

### 设置数据库连接字符串

```json
{
  // 在这里创建数据库连接字符串  
  "ConnectionStrings": {
    "Default":"server=127.0.0.1;database=MyBBSDb;uid=sa;pwd=1qaz2wsx",
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*"
}
```

### 创建实体

```C#
    public class User : BaseModel<int> // BaseModel 需要引入 Youshow.Adc.Domain 模块
    {
        public string UserNo { get; set; }
        public string UserName { get; set; }
        public string Password { get; set; }
        public bool IsDelete { get; set; }
        public int UserLevel { get; set; }
    }
```



### 创建上下文

创建文件 `YoushouDemoDbContext.cs` ，并继承 `AdcDbContext`

```C#
    [ConnectionStringName("Default")] //指定需要使用的连接字符串，默认Default
    public class YoushouDemoDbContext : AdcDbContext<YoushouDemoDbContext>
    {
        public DbSet<User> Users { get; set; } // 配置DbSet属性，用来和数据库做连接
        public YoushouDemoDbContext(DbContextOptions<YoushouDemoDbContext> options) : base(options)
        {
        }
    }
```

> 小贴士
>
> 如果不使用特性指定数据库连接字符串，默认使用Default

### 添加模块依赖

```C#
    [RelyOn(
        typeof(AdcEntityFrameworkCoreModule)
    )]
    public class YoushouDemoEntityFrameworkCoreModule:AdcModule
    {
        public override void ConfigureServices(ServiceConfigurationContext context)
        {
          // 添加数据库上下文  
          context.Services.AddAdcDbContext<YoushouDemoDbContext>();
          // 配置选项
          Configure<AdcDbContextOptions>(options=>{
              options.UseSqlServer(); // 或options.UseMySql();
          });
        }
    }
```

### 数据获取

```C#
    [ApiController]
    [Route("[controller]")]
    public class WeatherForecastController : BaseController
    {        
        private readonly YoushouDemoDbContext _context;
		
        // 使用构造函数注入 YoushouDemoDbContext 上下文，也可以直接属性注入
        public WeatherForecastController(YoushouDemoDbContext context)
        {
            this._context = context;
        }

        [HttpGet]
        public List<Users> Get()
        {
            return _context.Users.ToList();
        }
    }
```

### 数据跟踪

**Entity Framework Core** 的数据跟踪功能会影响系统的性能，所以在 **AdcFramework** 中，**Entity Framework Core** 的数据跟踪功能被默认关闭，如果希望开启数据跟踪功能，只需执行如下操作：

```C#
  Configure<AdcDbContextOptions>(options=>{
      options.UseSqlServer();
      // 跟踪开关。默认就是关闭，QueryTrackingBehavior.TrackAll 表示开启跟踪
      options.UseQueryTrackingBehavior(QueryTrackingBehavior.TrackAll);
  });
```

### 数据仓储

在 **AdcFramework** 中，目前只开启了默认仓储功能，自定义仓储还没有开放，后期会更新

开启默认仓储的方式也很简单，只需要如下操作

```C#
  context.Services.AddAdcDbContext<YoushouDemoDbContext>(opts=>{
      opts.AddDefaultRepositories(true); // 开启默认仓储，默认情况下关闭
  });
```



### 使用仓储

仓储的使用非常简单，只需要注入就可以使用

```C#
public class UserController : ControllerBase
{
    public IRepository<User> _userRepo { get; set; }
    public List<User> Get()
    {
        var users = _userRepo.GetListAsync().Result;
        return users;
    }
}
```





> 小贴士
>
> 如果要开启默认仓储，那么在opts.AddDefaultRepositories(true); 必须传入true，因为默认情况下默认仓储时关闭的。



### 读写分离模块使用

**AdcFramework** 框架集成了简单的读写分离功能，能够应付绝大多数的企业级项目中的数据访问和增删改操作。

**AdcFramework** 实现读写分离功能十分简单，只需要在配置文件中配置读库和写库，并在上下文的特性中标注，但是注意要实现读写分离只能在使用默认仓储的情况下才能实现。

#### 配置文件中配置读库和写

```json
"ConnectionStrings": {
  "Default":"server=192.168.31.201;database=MyBBSDb;uid=sa;pwd=1qaz2wsx",
  "Read":"server=192.168.31.202;database=MyBBSDb;uid=sa;pwd=1qaz2wsx,server=127.0.0.1;database=MyBBSDb;uid=sa;pwd=1qaz2wsx",
  "Write":"server=192.168.31.201;database=MyBBSDb;uid=sa;pwd=1qaz2wsx"
},
```

#### 上下文配置读写分离

```C#
[ConnectionStringName("Default")]
[ConnectionStringNameForRead("Read")]
[ConnectionStringNameForWrite("Write")]
public class YoushowDemoDbContext : AdcDbContext<YoushowDemoDbContext>
{
    public DbSet<User> Users { get; set; }
    public YoushowDemoDbContext(DbContextOptions<YoushowDemoDbContext> options) : base(options)
    {
    }
}
```

此时正常使用默认仓储就可以实现读写分离啦！小伙伴们试试吧。



# 能力服务层

**AdcFramework** 提供了一个能力服务模块，这个模块的作用其实很简单

- 可以作为业务逻辑的开发层
- 可以作为**领域模型**及**领域服务**的调度层
- 可以作为**微服务**之间的沟通层
- 也可以直接作为Api对外提供服务

## 引入Youshow.Adc.Ability包

```
Install-Package Youshow.Adc.Ability
```

## 添加AdcAbilityModule模块

```C#
[RelyOn(
    typeof(AdcAbilityModule),
    typeof(YoushowDemoEntityFrameworkCoreModule)
)]
public class YoushowDemoApplicationModule:AdcModule
{
}
```

## 能力服务类的创建

能力服务类的创建需要有接口和实现类，接口需要继承 `IAbilityServicer` ；而实现层需要继承 `AbilityServicer` ；并且类似控制器后缀的命名规范一样需要有一套规范的命名规则，以便于未来动态API的创建

### 接口开发

```C#
public interface IUserAppService : IAbilityServicer
{
    string GetUserName();
}
```

### 实现类开发

```C#
public class UserService : AbilityServicer, IUserAppService
{
    public string GetUserName()
    {
        return "Ace";
    }
}
```



## 动态API

在 **AdcFramework** 中如果要使用 **动态Api** 需要手动开启，开启方式如下

```C#
public class YoushowDemoApiModule : AdcModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        services.AddSwaggerGen(c =>
        {
            c.SwaggerDoc("v1", new OpenApiInfo { Title = "Youshow.Demo.Api", Version = "v1" });
            c.DocInclusionPredicate((_, _) => true); // 在Swagger中访问动态API
        });
        services.Configure<AdcAspNetCoreWebOptions>(opt =>
        {
           opt.Create(typeof(YoushowDemoApplicationModule).Assembly); // 注册YoushowDemoApplicationModule下所有的服务方法为动态API
           opt.AddControllerInExtends(true); 
        });
    }
    public override void OnApplicationInitialization(ApplicationInitializationContext context)
    {
        var env = context.GetEnvironment();
        var app = context.GetApplicationBuilder();
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
            app.UseSwagger();
            app.UseSwaggerUI(c => c.SwaggerEndpoint("/swagger/v1/swagger.json", "Youshow.Demo.Api v1"));
        }
    }
}
```

### 阻止动态Api接口映射

在有些情况下，我们并不需要能力服务者下所有的方法都映射成 **动态Api** ，此时我们可以在不需要映射的 **方法或** 者 **类** 上添加特性 `[NonRemoteVisibleAttribute]` 来阻止其映射成动态API

```C#
public class DepartmentAppService : AbilityServicer, IDepartmentAppService
{
    public IRepository<User> UserRepo { get; set; }
    
    [NonRemoteVisibleAttribute]//不对完展示接口
    public Task DistributedHandleEventAsync(User eventData)
    {
        return Task.CompletedTask;
    }

    public List<User> GetName(string name)
    {
        var user = UserRepo.GetList();
        return user;
    }
    
    [NonRemoteVisibleAttribute]//不对完展示接口
    public Task LocalHandleEventAsync(User eventData)
    {
        return Task.CompletedTask;
    }
}
```

### 特性标注创建动态API

有时候我们并不希望通过继承 `IAbilityServicer` 来实现动态API的创建，此时我们可以使用普通类型上面注册上特性 `[RemoteServicerAttribute]` 以实现动态API的注册，种方式非常灵活，**但是注意需要将该类所在的程序集在模块中进行注册**。

```C#
[RemoteServicer]
public class TestApiService
{
    public string GetTest(){
        return "Test";
    }
}
```



## AutoMapper使用

### 引入Youshow.Adc.AutoMapper包

```
Install-Package Youshow.Adc.AutoMapper
```

### 添加映射规则类

映射规则类需要继承 `Profile`，并在构造函数中写映射规则

```C#
public class YoushowDemoApplicationProfile : Profile
{
    public YoushowDemoApplicationProfile()
    {
        CreateMap<User,UserDto>().ReverseMap();
    }
}
```

### 加入AutoMapper模块

```C#
[RelyOn(
    typeof(AdcAbilityModule),
    typeof(AdcAutoMapperModule),
    typeof(YoushowDemoEntityFrameworkCoreModule)
)]
public class YoushowDemoApplicationModule:AdcModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var services = context.Services;
        services.Configure<AdcAutoMapperOptions>(opt=>{
            opt.AddProfile<YoushowDemoApplicationProfile>();
        });
    }
}
```

### 使用映射

```C#
public class UserService : AbilityServicer, IUserAppService
{
    public IRepository<User> _userRepo { get; set; }
    public List<UserDto> GetUserName()
    {
        var users = _userRepo.GetListAsync().Result;
        var userdtos = ModelMapper.Map<List<User>, List<UserDto>>(users); // 映射实体
        return userdtos;
    }
}
```



# 领域层

在之前对数据访问的文档中，已经创建过领域层实体，现在我们来好好看下领域层的使用

领域层是 **DDD领域驱动设计** 的核心层，但是 **AdcFramework** 绝不是只能使用DDD思想开发的框架，大家完全可以把领域层作为普通的Model层来使用，但是也欢迎大家使用领域驱动的思想去设计领域模型。

## 引入Youshow.Adc.Domain包

```
Install-Package Youshow.Adc.Domain
```

## 添加YoushowAdcDomainModule模块

```C#
[RelyOn(
    typeof(AdcDomainModule)
)]
public class YoushowDemoDomainModule:AdcModule
{
}
```

## 添加领域模型

```C#
public class User : BaseModel<int>
{
    public string UserNo { get; set; }
    public string UserName { get; set; }
    public string Password { get; set; }
    public bool IsDelete { get; set; }
    public int UserLevel { get; set; }
}
```

## DDD领域驱动思想的融入

### 充血模型使用

如果我们使用 **领域驱动思想** 去使用领域模型，就要习惯于使用 **充血模型** 。所谓 **充血模型** 就是在领域模型中除了有一些映射数据库中的属性以外，还需要一些行为方法，比如人类除了有身高、年龄等固有属性外，还有吃、喝、玩、乐等行为，这就叫 **充血模型**。

但是注意，在 **领域驱动思想** 下，模型的行为一般只设计能让 **领域模型** 状态发生变化的行为，即会对  **领域模型** 中的属性发生变化的情况。

以用户等级增加为例：

```C#
public class User : BaseModel<int>
{
    public string UserNo { get; set; }
    public string UserName { get; set; }
    public string Password { get; set; }
    public bool IsDelete { get; set; }
    public int UserLevel { get; set; }
    public int ScoreId { get; set; }
    
    public void SetUserLevel(IRepository<UserScore> userScoreRepo)
    {
        var score = userScoreRepo.Get(m => m.Id == this.ScoreId);
        if (score.TotalScore > 100)
        {
            this.UserLevel++;
        }
    }
}
```



### 领域服务者

虽说 **充血模型** 的有个极大的优势，就是一个可以几乎不懂代码的人看一眼就能知道这个领域要做什么。但是由于三层架构在很长一段时间内被大家广泛使用，已经深入人心，很多人就是不喜欢在实体模型中掺入行为方法，这在我的学员中也很常见，为避免此类情况的出现，我们还可以使用一种叫 **领域服务者** 的东西。

 **领域服务**者 可以很好的帮我们把一些对领域状态进行修改的业务逻辑方法从领域模型中剥离出来，从而让领域模型变得干净，不掺杂任何方法。**但是我们还是提倡使用充血模型能更好的让领域专家于开发人员进行交流互动**。

#### 领域服务者的创建

```C#
public class UserDomainServicer : DomainServicer
{
    public IRepository<UserScore> UserScoreRepo { get; set; }
    public void SetUserLevel(User user)
    {
        var score = UserScoreRepo.Get(m => m.Id == user.ScoreId);
        if (score.TotalScore > 100)
        {
            user.UserLevel++;
        }
    }
}
```

一旦创建了领域服务者，就可以在 **能力服务者** 中通过注入使用了

#### 领域服务者的使用

```C#
public class UserService : AbilityServicer, IUserAppService
{
    public IRepository<User> _userRepo { get; set; }
    public UserDomainServicer UserDomainServicer { get; set; } // 注入用户领域服务者
    public List<UserDto> GetUserName()
    {
        var users = _userRepo.GetListAsync().Result;
        UserDomainServicer.SetUserLevel(users); // 设置用户等级
        var userdtos = ModelMapper.Map<List<User>, List<UserDto>>(users);
        return userdtos;
    }
}
```



# 事件总线

**事件总线** 可以说是贯穿整个框架数据交互最重要的东西，他就像一辆公交车，将数据送到他们各自该去的站台，接收改造和处理，无论是单体项目间的 **领域通信** 或是 **能力服务者** 之间的通信，或是 **微服务分布式项目** 之间的通信都是必不可少的重要存在。

在 **AdcFramework** 同样集成了 **本地总线** 与 **分布式总线** 两种类型的事件总线

## 本地总线

本地总线用于在本项目中进行总线

### 引入Youshow.Adc.EventBus包

```
Install-Package Youshow.Adc.EventBus
```

### 添加模块

```C#
[RelyOn(
    typeof(AdcEventBusModule), // 添加事件总线模块
    typeof(AdcAbilityModule),
    typeof(AdcAutoMapperModule),
    typeof(YoushowDemoEntityFrameworkCoreModule)
)]
public class YoushowDemoApplicationModule:AdcModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var services = context.Services;
        services.Configure<AdcAutoMapperOptions>(opt=>{
            opt.AddProfile<YoushowDemoApplicationProfile>();
        });
    }
}
```

### 本地事件发布

事件发布分为两种形式，一种是 **主动发布** ，一种是 **被动发布** 。下面来详细介绍这两种发布模式的用法。

#### 主动发布

当我们需要主动发布事件总线时就可以使用 `PublishAsync()` 方法发布，具体使用如下

##### 注入事件总线并发布

在项目中注入 `ILocalEventBus` ，然后使用注入的对象使用 `PublishAsync` 方法发布 **总线消息**

```C#
public ILocalEventBus _localEventBus { get; set; }
public List<UserDto> GetetUser()
{
    var users = _userRepo.GetListAsync().Result;
    var userdtos = ModelMapper.Map<List<User>, List<UserDto>>(users);
  
    _localEventBus.PublishAsync(userdtos); // 发布本地事件
    return userdtos;
}
```

##### 事件触发

在我们发布了 **总线消息** 后，就会触发 **总线事件**，现在我们来创建总线事件接收消息。

一般情况下我们建议专门新建一个 `EventHandle` 类，用于接收总线事件，而不是直接在 **能力服务者** 中接收 **总线事件** 。因为 **触发事件** 可能会在动态API中被映射，如果一定想在 **能力服务者** 中添加 **触发事件** ，修需要使用 `NonRemoteVisibleAttribute` 阻止触发事件映射成动态API。

```C#
public class DepartmentAppService : ILocalEventHandler<User> , ITransientDependency
{
    public Task LocalHandleEventAsync(List<UserDto> eventData) // 接收总线消息传入的参数
    {
        // ...
        return Task.CompletedTask;
    }
}
```

> 小贴士
>
> 在本地总线的 **触发事件** 中传入的参数如果是引用类， 一经修改，在发送端也会发生变化，利用这里特性，可以做一些很有意思的事情，比如用代码模拟表连接查询。



#### 被动发布

被动发布是在仓储进行 **增、删、改** 后，将变化后的数据发送到总线中，进行处理。当上下文执行 `SaveChanges()` 方法时发送 **总线消息** ，并不需要我们去写发送命令。

```C#
public ILocalEventBus _localEventBus { get; set; }
public void SetetUser()
{
    _userRepo.Insert(new User
	{
    	Password = "123456",
    	UserName = "Taro",
    	UserNo = "666",
    	IsDelete = false,
    	UserLevel = 6
	});
}
```

当我们使用 `Insert` 方法添加数据，系统并执行了 `SaveChanges()` 之后，就会触发相应的 **总线事件** 。

```C#
public class DepartmentAppService : ILocalEventHandler<ModelCreatingEventData<User> eventData>,  ITransientDependency
{
	public async Task LocalHandleEventAsync(ModelCreatingEventData<User> eventData)
	{
		// ...
	}
}
```

除了 `ModelCreatingEventData<T>` 以外，还有 `ModelChangingEventData<T>` 和 `ModelDeletingEventData<T>` 。



## 分布式总线

当我们的项目由多个微服务组成时，微服务之间的通信就需要分布式总线了，使用方式和本地总线几乎一模一样。这里我们集成了 **RabbitMQ** 和 **Kafka**

### RabbitMQ 集成

#### 引入Youshow.Adc.EventBus.RabbitMQ包

```
Install-Package Youshow.Adc.EventBus.RabbitMQ
```

#### 在配置文件中配置

在 `appsettings.json` 中配置 **RabbitMQ** 连接 与 **客户端和交换机** 信息

```json
{
  "RabbitMQ": {
    "Connections": {
      "Default": {
        "HostName": "192.168.31.201",
        "Port": "5672",
        "UserName":"ace",
        "Password":"123456"
      }
    },
    "EventBus": {
      "ClientName": "MyClientName",
      "ExchangeName": "MyExchangeName"
    }
  }
}
```

#### 添加模块

```C#
[RelyOn(
    typeof(AdcAbilityModule),
    typeof(AdcAutoMapperModule),
    typeof(AdcEventBusRabbitMqModule),
    typeof(YoushowDemoEntityFrameworkCoreModule)
)]
public class YoushowDemoApplicationModule:AdcModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var services = context.Services;
        services.Configure<AdcAutoMapperOptions>(opt=>{
            opt.AddProfile<YoushowDemoApplicationProfile>();
        });
    }
}
```

#### 分布式事件发布

##### 主动发布

当我们需要主动发布事件总线时就可以使用 `PublishAsync()` 方法发布，具体使用如下

###### 注入事件总线并发布

在项目中注入 `IDistributedEventBus` ，然后使用注入的对象使用 `PublishAsync` 方法发布 **总线消息**

```C#
public IDistributedEventBus _distributedEventBus { get; set; }
public List<UserDto> GetetUser()
{
    var users = _userRepo.GetListAsync().Result;
    var userdtos = ModelMapper.Map<List<User>, List<UserDto>>(users);
  
    _distributedEventBus.PublishAsync(userdtos); // 发布本地事件
    return userdtos;
}
```

###### 事件触发

在我们发布了 **总线消息** 后，就会触发 **总线事件**，现在我们来创建总线事件接收消息。

一般情况下我们建议专门新建一个 `EventHandle` 类，用于接收总线事件，而不是直接在 **能力服务者** 中接收 **总线事件** 。因为 **触发事件** 可能会在动态API中被映射，如果一定想在 **能力服务者** 中添加 **触发事件** ，修需要使用 `NonRemoteVisibleAttribute` 阻止触发事件映射成动态API。

```C#
public class DepartmentAppService : IDistributedEventHandler<User>, , ITransientDependency
{
    public Task DistributedHandleEventAsync(User eventData)
    {
        return Task.CompletedTask;
        //throw new System.NotImplementedException();
    }
}
```

> **注意**
>
> 与本地总线不同，使用分布式总线都是需要先将数据序列化后再传给 **RabbitMQ** 或 **Kafka** 的。所以在触发端修改的数据不会在消息发送端体现。

##### 被动发布

被动发布是在仓储进行 **增、删、改** 后，将变化后的数据发送到总线中，进行处理。当上下文执行 `SaveChanges()` 方法时发送 **总线消息** ，并不需要我们去写发送命令。

```C#
public IDistributedEventBus _distributedEventBus { get; set; }
public void SetetUser()
{
    _userRepo.Insert(new User
	{
    	Password = "123456",
    	UserName = "Taro",
    	UserNo = "666",
    	IsDelete = false,
    	UserLevel = 6
	});
}
```

当我们使用 `Insert` 方法添加数据，系统并执行了 `SaveChanges()` 之后，就会触发相应的 **总线事件** 。

```C#
public class DepartmentAppService : IDistributedEventHandler<ModelCreatingEventData<User> eventData>,  ITransientDependency
{
	public async Task LocalHandleEventAsync(ModelCreatingEventData<User> eventData)
	{
	// ...
	}
}
```

除了 `ModelCreatingEventData<T>` 以外，还有 `ModelChangingEventData<T>` 和 `ModelDeletingEventData<T>` 。



##### 事件名称（EventName）

在分布式总线中需要使用特性 `EventNameAttribute` 来将分布式数据准确投放到他们需要的分布式项目中，故消息发送端发送参数的 **EventName** 必须和接收端接收参数的 **EventName** 相同。否则 **EventName** 就是当前类的命名空间本身，所以只能投递到当前项目中。

```C#
[EventName("Youshow.Demo.User")]
public class User : BaseModel<int>
{
    // ...
}

[EventName("Youshow.Demo.UserEto")]
public class UserEto
{
    // ...
}
```



# 工作单元

**工作单元** 也叫 **WorkUnit** 在 **AdcFramework** 中属于概念上的内容，并不需要我们去过问他，工作单元会把我们的一次 **非Get** 访问中所有的操作自动整合成一个工作组。在访问结束后整体处理。当然，我们也可以关闭 **工作单元**，让每次一操作都成为一个独立的工作。

## 特性配置工作单元

在开发时，我们可以通过特性配置工作单元的一些信息，比如 **关闭工作单元** 或 **关闭事务**

### 关闭事务

```C#
public class UserAppService : AbilityServicer
{
    public IRepository<User> _userRepo { get; set; }
    [WorkUnit(false)] // 关闭事务
    public List<UserDto> SetUser()
    {
    }
}
```

### 关闭单个工作单元

```C#
public class UserAppService : AbilityServicer
{
    public IRepository<User> _userRepo { get; set; }
    [WorkUnit(IsDisabled = true)] // 访问 SetUser 时将不会组建工作单元
    public List<UserDto> SetUser()
    {
    }
}
```



## 全局配置工作单元

我们除了可以在某个方法上通过特性单独配置工作单元信息，也可以 **全局配置**

### 全局关闭事务

```C#
public class YoushowDemoApiModule : AdcModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<WorkUnitOptions>(opt=>{
            opt.WorkUnitTransactionSwitch = WorkUnitTransactionSwitch.Off;  // 全局关闭事务
        });
    }
}
```

### 全局关闭工作单元

```C#
public class YoushowDemoApiModule : AdcModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<WorkUnitOptions>(opt=>{
            opt.IsDisabledGlobalWorkUnit = true; // 全局关闭工作组
        });
    }
}
```



# 后台定时工作

后台工作在应用的后台线程运行。一般来说，后台工作都是定期运行的，以执行一些任务。

- 后台工作可以定期**删除过时的日志**。
- 后台工作可以定期检查**不活跃的用户**并且向其**发送邮件**使用户继续使用你的应用程序。
- 后台工作也可以单独在某次访问执行之后，定时执行某些操作，比如用户下订单30分钟后未付款，则**自动删除订单**。



## 引入Youshow.Adc.BackgroundWork包

```
Install-Package Youshow.Adc.BackgroundWork
```

## 添加总线模块

```C#
[RelyOn(
    typeof(AdcEventBusModule),
    typeof(AdcAbilityModule),
    typeof(AdcAutoMapperModule),
    typeof(YoushowDemoEntityFrameworkCoreModule)
)]
public class YoushowDemoApplicationModule : AdcModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var services = context.Services;
        services.Configure<AdcAutoMapperOptions>(opt=>{
            opt.AddProfile<YoushowDemoApplicationProfile>();
        });
    }
}
```





## 周期后台工作

周期后台工作是运行在全局系统下的后台工作，可以在系统运行之初就可以在后台运行，也可以在指定的地方运行



### 应用启动时运行

#### 添加全局周期工作

```C#
public class MyTestPeriodicBackgroundWork : PeriodicBackgroundWorkBase
{
    private readonly ILogger<MyTestPeriodicBackgroundWork> _logger;
    public MyTestPeriodicBackgroundWork(ILogger<MyTestPeriodicBackgroundWork> logger)
    {
        this._logger = logger;
        Periodic = 1000;
    }
    int i = 0;
    protected override Task DoWorkAsync(PeriodicBackgroundWorkContext workContext)
    {
        _logger.LogWarning("定时任务启动" + i++);
        return Task.CompletedTask;
    }
}
```



#### 添加模块

```C#
    [RelyOn(
        typeof(AdcAbilityModule),
        typeof(AdcAutoMapperModule),
        typeof(AdcBackgroundWorkModule),
        typeof(YoushowDemoEntityFrameworkCoreModule)
    )]
    public class YoushowDemoApplicationModule:AdcModule
    {
        public override void ConfigureServices(ServiceConfigurationContext context)
        {
            // ...
        }
        public override void OnApplicationInitialization(ApplicationInitializationContext context)
        {
             // 指定添加后台工作
             context.AddPeriodicBackgroundWork<MyTestPeriodicBackgroundWork>();
        }
    }
```



##### 指定添加后台工作

```C#
public override void OnApplicationInitialization(ApplicationInitializationContext context)
{
    // 指定添加后台工作
    context.AddPeriodicBackgroundWork<MyTestPeriodicBackgroundWork>();
}
```



##### 添加所有后台工作

```C#
public override void OnApplicationInitialization(ApplicationInitializationContext context)
{
    // 添加所有后台工作
    context.AddPeriodicBackgroundWork();
}
```



##### 添加多个指定后台工作

```C#
public override void OnApplicationInitialization(ApplicationInitializationContext context)
{
    // 添加多个指定后台工作
     context.AddPeriodicBackgroundWork(
         	typeof(MyTestPeriodicBackgroundWork1),
         	typeof(MyTestPeriodicBackgroundWork2),
         	typeof(MyTestPeriodicBackgroundWork3));
}
```



##### 禁止后台定时作业

```C#
public override void ConfigureServices(ServiceConfigurationContext context)
{
    //禁止后台定时作业(默认为不禁止，即IsEnabled为true)
    Configure<AdcBackgroundWorkOptions>(opt=>{
    	opt.IsEnabled = false;
    });
}
```



### 指定位置启动周期性后台任务

#### 添加全局周期工作

```C#
public class MyTestPeriodicBackgroundWork : PeriodicBackgroundWorkBase
{
    private readonly ILogger<MyTestPeriodicBackgroundWork> _logger;
    public MyTestPeriodicBackgroundWork(
        ILogger<MyTestPeriodicBackgroundWork> logger,
        IServiceScopeFactory serviceScopeFactory) : base(serviceScopeFactory) // 需要给基类传输IServiceScopeFactory
    {
        this._logger = logger;
        Periodic = 1000; // 执行周期时间
    }
    int i = 0;
    protected override Task DoWorkAsync(PeriodicBackgroundWorkContext workContext)
    {
        _logger.LogWarning("定时任务启动" + i++);
        return Task.CompletedTask;
    }
}
```

> 小贴士
>
> 如果需要在指定位置添加全局主题工作，需要在构造函数中给基类传递IServiceScopeFactory对象



#### 在服务中使用

```C#
public class OrderService : AbilityServicer, IOrderService
{
    public IRepository<Order> _orderRepo { get; set; }

    public MyTestPeriodicBackgroundWork MyTestPeriodicBackgroundWork { get; set; }
    
    public List<OrderDto> SetOrder()
    {
        // ...
        MyTestPeriodicBackgroundWork.StartAsync();
        return userdtos;
    }
}
```



## 定时后台工作

定时后台工作无法在系统启动时运行，只能在指定的地方执行，比如创建订单后，未即时付款后的删除订单工作

### 创建定时后台工作

```C#
public class OvertimeOrderDeleteBackgroundWork : TimedBackgroundWorkBase<User>
{
    private readonly ILogger<OvertimeOrderDeleteBackgroundWork> _logger;
    public OvertimeOrderDeleteBackgroundWork(ILogger<OvertimeOrderDeleteBackgroundWork> logger)
    {
        this._logger = logger;
        Periodic = 1000; // 执行周期时间
        LoopTimes = 2; // 执行几次
    }
    protected override Task DoWorkAsync(User workArgs)
    {
        // ...
         _logger.LogWarning("局部定时任务_删除超时订单");
        return Task.CompletedTask;
    }
}
```



### 在服务中使用

```C#
public class OrderService : AbilityServicer, IOrderService
{
    public IRepository<Order> _orderRepo { get; set; }

    public OvertimeOrderDeleteBackgroundWork OvertimeOrderDeleteBackgroundWork { get; set; }
    
    public List<OrderDto> SetOrder()
    {
        var order = _orderRepo.Insert(new Order
        {
            // ...
        });
        OvertimeOrderDeleteBackgroundWork.StartAsync(order);;
        return userdtos;
    }
}
```



## 自定义后台工作扩展

`BackgroundWorkBase` 是创建后台工作者的基类，继承基类重写其 `StartAsync` 和 `StopAsync` 来自己开发后台工作

```csharp
public class MyWork : BackgroundWorkBase, ISingletonDependency
{
    public override Task StartAsync(CancellationToken cancellationToken = default)
    {
        //...
    }

    public override Task StopAsync(CancellationToken cancellationToken = default)
    {
        //...
    }
}
```

> 小贴士
>
> 只有继承 `ISingletonDependency` 生命周期接口，才能在全局注册使用



# 微服务

说到 **DDD**，就免不了谈起 **微服务** ，这二者可谓是相辅相成，相互成就。在 **AdcFramework** 中可以及其简单的将项目改造成微服务

## 服务注册发现（Consul）

### 在配置文件中配置

在 `appsettings.json` 中配置 **Consul服务器端以及本地服务** 的一些信息

``` json
"Consul": {
  "ConsulServerOptions": {
    "OpenSSL": false,        // 是否使用 https 访问，默认false
    "IP": "192.168.31.201", // consul服务端的IP
    "Port": "8500", 		// consul服务端的Port
    "Datacenter": "dc1"
  },
  "ConsulLocalOptions": {
    "IP": "localhost",                    // consul客户端，即本地的IP
    "Port": "5858", 					  // consul客户端，即本地的Port
    "GroupName": "YoushowDemo", 		  // 当前服务的组名
    "Interval": 10, 					  // 健康检查间隔时间
    "Timeout": 5, 					      // 健康检查超时时间
    "DeregisterCriticalServiceAfter": 20, // 健康检查超时后销毁时间
    "CheckPath": "/Health", 			  // 健康检查地址
    "OpenSSL": false,                    // 是否使用 https 访问，默认false
    "Tag": "16"
  }
},
```

### 引入Youshow.Adc.MicroService.Consul包

```
Install-Package Youshow.Adc.MicroService.Consul
```

### 添加模块

```C#
[RelyOn(
	typeof(AdcAspNetCoreWebModule),
	typeof(AdcMicroServiceConsulModule), // 添加AdcMicroServiceConsulModule模块
	typeof(YoushowDemoApplicationModule)
)]
public class YoushowDemoApiModule : AdcModule
{
}    
```

此时，启动项目即可注册当前微服务

> 小贴士
>
> 当然我们也可以讲模块添加到能力服务层中。



### 与其他微服务进行HTTP交互

在微服务之间除了可以使用总线通信外，最常用的通信方式就是通过HTTP进行通信。方法如下。

```C#
[ApiController]
[Route("[controller]")]
public class WeatherForecastController : BaseController
{
    // 属性注入IConsulDispatcherHelper，用于进行微服务间数据交互
    public IConsulDispatcherHelper ConsulDispatcherHelper { get; set; }
    [HttpGet]
    public async Task<List<UserDto>> Get()
    {
        //用Get方式向其他微服务获取数据
        return await ConsulDispatcherHelper
        .GetRequireAsync<List<UserDto>>("http://YoushowDemo/app/user/GetUserName?id=1");
    }
}
```

除了 `GetRequireAsync` 访问方法，AdcFramework 还提供了一下几种方法：

`GetRequire()`、`GetRequireAsync()`、`GetRequire<T>()`、`GetRequireAsync<T>()`

`PostRequire()`、`PostRequireAsync()`、`PostRequire<T>()`、`PostRequireAsync<T>()`

`PutRequire()`、`PutRequireAsync()`、`PutRequire<T>()`、`PutRequireAsync<T>()`

`DeleteRequire()`、`DeleteRequireAsync()`、`DeleteRequire<T>()`、`DeleteRequireAsync<T>()`

#### HTTP 访问地址格式

```
http://GroupName（目标微服务组名）/要访问的具体地址
```

以当前项目为例就是

```
http://YoushowDemo/app/user/GetUserName?id=1
```



# 用户验证

AdcFramework集成了 **jwt** 用户验证，可以非常方便的实现用户 **鉴权验证** 和 **授权验证** 。同时也开放了验证功能的扩展。

## 引入Youshow.Adc.AspNetCore.Jwt包

```
Install-Package Youshow.Adc.AspNetCore.Jwt
```



## 添加模块

```C#
[RelyOn(
    typeof(AdcAspNetCoreWebModule),
    typeof(AdcMicroServiceConsulModule),
    typeof(AdcAspNetCoreJwtModule), // 添加AdcAspNetCoreJwtModule模块
    typeof(YoushowDemoApplicationModule)
)]
public class YoushowDemoApiModule : AdcModule
{
	// 添加jwt服务
	services.AddJwtAuthentication();
}
```

>小贴士
>
>当前 **Jwt** 模块是在 **API** 层中添加的，但是建议在 **能力服务层** 添加可能更加适合，因为**要考虑微服务之间的直接通信而产生的鉴权授权**。



## 在配置文件中配置

在 `appsettings.json` 中配置 **jwt** 的一些信息

```json
"Jwt": {
  "JwtAuths": {
    "Audience": "YoushowDemo",
    "Issuer": "YoushowDemo",
    "SecurityKey": "dzehzRz9a8+8TAGbqKHP9ITdRmZdOpJWQRsFb8oz50A=",
    "ExpirationTime": 12
  }
},
```



## 生成Token

**Token** 生成方式非常简单，只需要注入 `IJwtManager` ，调用 `CreateToken` 方法即可创建

```C#
[ApiController]
[Route("[controller]")]
public class WeatherForecastController : BaseController
{
    public  IJwtManager JwtManager { get; set; }
    public IConsulDispatcherHelper ConsulDispatcherHelper { get; set; }
    [HttpGet]
    public string Get(int userId)
    {
       // 创建Token
       return JwtManager.CreateToken(new ModelClaimValue<int>{
           UserId = userId //创建token的模型必须是ModelClaimValue或其子类
       });
    }
}
```

### 自定义Claim中保存的内容

创建 `MyModelClaimValue` 实体，继承于 `ModelClaimValue<int>`

```C#
public class MyModelClaimValue : ModelClaimValue<int>
{
    public string UserName { get; set; }
}
```

传输实体

```C#
[HttpGet]
public string Get(int userId)
{
    return JwtManager.CreateToken(new MyModelClaimValue
    {
        UserId = userId,
        UserName = "Ace"
    });
}
```



## 使用 jwt 自带鉴权

jwt 默认的鉴权机制相对简单，只要验证 `Audience`、`Issuer`、`SecurityKey`、`ExpirationTime` 是否正确即可通过鉴权

```C#
[HttpGet("Auth")]
[Authorize] // jwt默认鉴权
public async Task<List<UserDto>> GetAuth()
{
    return await ConsulDispatcherHelper
    .GetRequireAsync<List<UserDto>>("http://YoushowDemo/app/user/GetUserName?id=1");
}
```



## 使用AdcFramework默认授权验证

**AdcFramework** 提供了默认鉴权模式，在 `Audience`、`Issuer`、`SecurityKey`、`ExpirationTime` 等验证通过的情况下，可以进一步对用户授权进行验证，但是需要提供授权 **数据管道**，否则等同于 jwt 默认的鉴权机制。但是注意 **数据管道** 有且只能有一个，否则就会报错。

### 添加鉴权特性

```[HttpGet("Auth")]
[HttpGet("Auth")]
[Authorize(AdcJwtPolicyDefault.AUTHORIZATION_POLICY)] // 使用AdcFramework默认授权验证时的鉴权特性
public async Task<List<UserDto>> GetAuth()
{
    return await ConsulDispatcherHelper
    .GetRequireAsync<List<UserDto>>("http://YoushowDemo/app/user/GetUserName?id=1");
}

### 开启AdcFramework默认授权验证

开启 **AdcFramework默认授权验证** 方式很简单，只需要调用 `UseAuthorizationPolicy(true)`方法即可，注意，这里默认传入参数是 `false`

```C#
[RelyOn(
    typeof(AdcAspNetCoreWebModule),
    typeof(AdcMicroServiceConsulModule),
    typeof(AdcAspNetCoreJwtModule),
    typeof(YoushowDemoApplicationModule)
)]
public class YoushowDemoApiModule : AdcModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var services = context.Services;
        // 添加jwt服务
        services.AddJwtAuthentication(opt =>
        {
            opt.UseAuthorizationPolicy(true); // 开启AdcFramework默认授权验证
        });
    }
```



### 授权数据管道处理

如果要获取用户可访问网址或者Api的权限，需要用户自己提供，以供系统去判断用户是否拥有访问某网址的权利。

在开发管道时，管道类需要继承 `IAuthorizationDataPipelineHandler` 接口，实现 `GetModelPermissionsAsync()` 方法，这里模拟从 **数据库** 或者 **缓存** 中获取的数据结果，需要将数据格式转成 `ModelPermission` 类型才可以传输。

```C#
public class JwtDataHandler : IAuthorizationDataPipelineHandler
{
    public async Task<List<ModelPermission>> GetModelPermissionsAsync()
    {
        return await Task.Run(() =>
        {
            List<ModelPermission> list = new();
            list.Add(new ModelPermission
            {
                UserId = "1",
                Url = "/Weatherforecast/Auth"
            });
            list.Add(new ModelPermission
            {
                UserId = "2",
                Url = "/Weatherforecast/Auth"
            });
            list.Add(new ModelPermission
            {
                UserId = "1",
                Url = "/Weatherforecast"
            });
            return list;
        });
    }
}
```



## 自定义扩展授权验证

**AdcFramework** 提供了 **自定义扩展授权验证** 的接口，用户只需要继承 `AdcAuthorizationHandler` 类，重写其 `RequirementHandlerAsync()` 方法即可。

### 创建自定义扩展授权验证类

```C#
public class MyAuthorizationHandler : AdcAuthorizationHandler
{
    protected override async Task<AuthResult> RequirementHandlerAsync(
        AuthorizationHandlerContext context, 
        HttpContext httpContext, 
        AdcAuthorizationRequirement requirement)
    {
        return new AuthResult{
            IsSuccess = true,
            Requirement = requirement
        };
    }
}
```

>注意！
>
>如果是 **自定义扩展授权验证** ，则理论上不需要再使用 **数据管道** ，但是我们依旧可以使用 **数据管道** 以达到 业务逻辑分离

### 在模块中加入验证类

```C#
// 添加jwt服务
services.AddJwtAuthentication(opt =>
{
	opt.UseAuthorizationPolicy<MyAuthorizationHandler>(); // 加入自定义扩展授权验证类
});
```



# 缓存

缓存是所有项目中及其常见的功能，在 AdcFramework 中也集成了 **系统自带缓存** 和 **Redis缓存** 两种，下面我们来看下这两者的使用方式

## 系统缓存

### 引入Youshow.Adc.Cache包

```
Install-Package Youshow.Adc.Cache
```

### 添加模块

```C#
[RelyOn(
    typeof(AdcAspNetCoreWebModule),
    typeof(AdcCacheModule), // 引入系统缓存模块
    typeof(YoushowDemoApplicationModule)
)]
```

### 使用方式

我们只需要在类型中注入缓存即可使用

```C#
[ApiController]
[Route("[controller]")]
public class WeatherForecastController : BaseController
{
    // 注入缓存
    public IDistributedCache<User> UserCache { get; set; }
    [HttpGet]
    public string Get(int userId)
    {
        // 写入缓存
        UserCache.Set("User", new User
        {
            Id = 1
        });
    }
    
    public async Task<List<UserDto>> GetAuth()
    {
        // 获取缓存
        var user = UserCache.Get("user");
    }
}
```

当然缓存的方法有很多，这里就不一一列举了。



## Redis 分布式缓存

### 引入Youshow.Adc.Cache包

```
Install-Package Youshow.Adc.Cache.Redis
```

### 在配置文件中配置

在 `appsettings.json` 中配置 **Redis** 的一些信息，`HostPorts` 是一个数组json，可以添加多个Redis集群 

```json
"Redis": {
  "Connections": {
    "Cluster": {
      "HostPorts": [
        {
          "Host": "192.168.31.201",
          "Port": 6379
        }
      ]
    },
    "Password": "1qaz2wsx"
  }
},
```

### 添加模块

```C#
[RelyOn(
    typeof(AdcAspNetCoreWebModule),
    typeof(AdcCacheRedisModule), // 引入Redis分布式缓存模块
    typeof(YoushowDemoApplicationModule)
)]
```

### 使用方式

#### 缓存实体

在使用前，我们可以先创建一个 缓存用的实体，作为缓存的数据载体

```C#
[CacheName("MyUserCto")] // 自定义缓存主主键名称
public class UserCto
{
    [CacheKey] // 定义缓存子主键
    public int Id { get; set; }
    public string UserName { get; set; }
    public static List<UserCtos> GetList()
    {
        List<UserCtos> UserCtos = new();
        for (int i = 0; i < 10; i++)
        {
            UserCtos.Add(new UserCtos
            {
                Id = i,
                UserName = "Ace" + i
            });
        }
        return UserCtos;
    }
}
```

> 小贴士
>
> - 缓存实体不是一个规定的实体，它可以是任何实体。
>
> - 定义缓存实体是，我们可以定义缓存的名称（主主键），可以指定子主键。
>
> - 当我们没有定义缓存的名称时，缓存名称就是改实体的类名，这里就是 `UserCto`。
>
> - 当我们没有定义子主键时，系统会自动找到适合的字段匹配，符合匹配规范的字段，以本案例举例例如（Id、UserCtoId）等，当然如果实体时复数形式，其单数形式+Id依旧可以被识别成子主键的候选属性。
> - 子主键可以用特性定义多个，以便于多字段查询。



#### 项目中使用

同样我们以属性注入的方式获取Redis缓存

```C#
[ApiController]
[Route("[controller]")]
public class WeatherForecastController : BaseController
{
    // 注入Redis缓存
    public IRedisCache RedisCache { get; set; }
    [HttpGet]
    public string Get(int userId)
    {
        // 写入缓存
        RedisCache.SetHashMemory<UserCto>(UserCtos.GetList());
    }
    
    public async Task<List<UserDto>> GetAuth()
    {
        // 获取缓存
        var users = RedisCache.GetHashMemory<List<UserCto>>("*"); // 返回所有数据
        var userFirst = RedisCache.GetHashMemory<UserCto>("*"); // 只返回第一条数据
        var userName = RedisCache.GetHashMemory("UserName","MyUserCto","2").ToString(); //返回CacheName是"MyUserCto"，Id是2的那条数据的UserName字段内容
    }
}
```

在这里 `*` 代表模糊匹配，即获取该 **CacheName** 下所有的内容并返回。当然我们可以 **指定自主键**，以返回需要的内容。

当然 **AdcFramework** 也为大家准备了很多操作Redis缓存的方法，这里也不一一列举了。

# ElasticSearch全文搜索

在 **AdcFramework** 中集成了全文搜索模块。默认插件使用了 **ElasticSearch**，以方便用户使用全文搜索功能。

## 引入 Youshow.Adc.ElasticSearch 包

```
Install-Package Youshow.Adc.ElasticSearch
```

## 在配置文件中配置

```json
"ElasticSearch": {
  "Register": {
    "Connections": {
      "Default": [
        {
          "ElasticsearchUri": "127.0.0.1",  // Elasticsearch访问地址
          "Port": 9200                      // Elasticsearch访问端口
        }
      ]
    },
    // "DefaultIndex": "kibana_sample_data_ecommerce",
    "DefaultIndex": "*",    // 默认索引
    "UserName": "elastic",  // 登陆用户名
    "Password": "elastic",  // 登陆密码
    "IsCluster": false,     // Es否是集群
    "OpenSSL": false        // 是否使用 https 访问，默认false
  }
},
"AllowedHosts": "*"
```

## 引入模块并配置

在 **AdcElasticSearchModule** 中为大家准备好了默认数据获取仓储，只要在配置中打开即可使用。

```C#
[RelyOn(
    typeof(AdcAspNetCoreWebModule),
    typeof(AdcElasticSearchModule), // 引入ElasticSearch模块
    typeof(YoushowDemoApplicationModule)
)]
public class YoushowDemoApiModule : AdcModule
{
    services.AddAdcElasticsearch(opt =>
    {
        opt.UseDefaultRepository(true); // 使用默认仓储，参数默认为false，不开启
    });
}
```

## 项目中使用

```C#
[ApiController]
[Route("[controller]")]
public class WeatherForecastController : BaseController
{
    // 注入仓储，当使用dynamic作为泛型参数时，会使用默认索引。
    public IElasticRepository<dynamic> elasticRepository { get; set; } 
    [HttpGet]
    public dynamic Get()
    {
        var res = elasticRepository.GetAll(); // 获取所有数据
        return res;
    }
}

```

### 实体定义索引名

在 **AdcFramework** 中，我们除了可以使用 `dynamic` 以获取 **默认索引** 的数据外，我们还可以传入 **标准实体**，并可以使用特性定义查询 **ElasticSearch** 数据时的索引名。

```C#
[ElasticIndexName("usermodel")]
public class UserEsModel
{
    public int Id { get; set; }
    public string UserName { get; set; }
}
```

> 注意！
>
> ElasticSearch 的索引名都是小写，如果你在 ElasticIndexName 中使用了大写，写不要紧，系统会自动专为小写。
>
> 如果你未定义此属性，系统会自动使用类名的小写方式作为索引名。
