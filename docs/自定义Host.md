参照[surging](https://github.com/dotnetcore/surging)的`ServiceHosting`模块，去掉`autofac`依赖。

[AspNetCore通用主机](https://github.com/aspnet/Extensions/tree/master/src/Hosting/Hosting/src)

### ServiceHost构造器
**接口**
`IServiceHostBuilder.cs`
```csharp
using Microsoft.Extensions.DependencyInjection;
using System;

namespace CustomHost.Internal
{
    public interface IServiceHostBuilder
    {
        IServiceHost Build();
        IServiceHostBuilder RegisterServices(Action<IServiceCollection> configureServices);
        IServiceHostBuilder ConfigureServices(Action<IServiceCollection> configureServices);
    }
}

```
**实现**
`ServiceHostBuilder.cs`
```csharp
using System;
using System.Collections.Generic;
using Microsoft.Extensions.DependencyInjection;

namespace CustomHost.Internal.Implementation
{
    public class ServiceHostBuilder : IServiceHostBuilder
    {
        private readonly List<Action<IServiceCollection>> _configureServicesDelegates;
        private readonly List<Action<IServiceCollection>> _registerServicesDelegates;

        public ServiceHostBuilder()
        {
            _configureServicesDelegates = new List<Action<IServiceCollection>>();
            _registerServicesDelegates = new List<Action<IServiceCollection>>();
        }

        public IServiceHost Build()
        {
            var services = BuildCommonServices();
            var hostingServices = RegisterServices();
            var hostingServiceProvider = services.BuildServiceProvider();
            var host = new ServiceHost(hostingServices, hostingServiceProvider);
            host.Initialize();
            return host;
        }

        public IServiceHostBuilder ConfigureServices(Action<IServiceCollection> configureServices)
        {
            if (configureServices == null)
            {
                throw new ArgumentNullException(nameof(configureServices));
            }
            _configureServicesDelegates.Add(configureServices);
            return this;
        }

        public IServiceHostBuilder RegisterServices(Action<IServiceCollection> configureServices)
        {
            if (configureServices == null)
            {
                throw new ArgumentNullException(nameof(configureServices));
            }
            _registerServicesDelegates.Add(configureServices);
            return this;
        }

        private IServiceCollection BuildCommonServices()
        {
            var services = new ServiceCollection();
            foreach (var configureServices in _configureServicesDelegates)
            {
                configureServices(services);
            }
            return services;
        }
        private IServiceCollection RegisterServices()
        {
            var hostingServices = new ServiceCollection();
            foreach (var registerServices in _registerServicesDelegates)
            {
                registerServices(hostingServices);
            }
            return hostingServices;
        }
    }
}
```
`_configureServicesDelegates`，`_registerServicesDelegates`存储服务注册委托。

`ConfigureServices(Action<IServiceCollection> configureServices)`，`RegisterServices(Action<IServiceCollection> configureServices)` 添加服务注册委托。

`Build`方法执行委托，注册服务，并实例化`ServiceHost`返回。

`BuildCommonServices`实例化`ServiceCollection`，执行`_configureServicesDelegates`添加的委托，注册服务。

`RegisterServices`实例化`ServiceCollection`，执行`_registerServicesDelegates`添加的委托，注册服务。

调用`BuildCommonServices`返回的`ServiceCollection`实例的`BuildServiceProvider`方法，得到IOC容器实例。

使用`RegisterServices`返回的`ServiceCollection`实例与上一步得到的`ServiceProvider`，作为构造方法参数，实例化`ServiceHost`，并调用实例的`Initialize`方法。
>`IServiceCollection`相当于`autofac`中的`ContainerBuilder`，`ServiceProvider`相当于`IContainer`


### ServiceHost
**接口**
`IServiceHost.cs`
```csharp
using System;

namespace CustomHost.Internal
{
    public interface IServiceHost : IDisposable
    {
        IDisposable Run();
        IServiceProvider Initialize();
    }
}
```
**实现**
`ServiceHost.cs`
```csharp
using CustomHost.Startup;
using Microsoft.Extensions.DependencyInjection;
using System;

namespace CustomHost.Internal.Implementation
{
    public class ServiceHost : IServiceHost
    {
        private readonly IServiceCollection _builder;
        private IStartup _startup;
        private IServiceProvider _applicationServices;
        private readonly IServiceProvider _hostingServiceProvider;
        public ServiceHost(IServiceCollection serviceCollection, IServiceProvider hostingServiceProvider)
        {
            _builder = serviceCollection;
            _hostingServiceProvider = hostingServiceProvider;
        }

        public void Dispose()
        {
            (_hostingServiceProvider as IDisposable)?.Dispose();
        }

        public IServiceProvider Initialize()
        {
            if (_applicationServices == null)
            {
                _applicationServices = BuildApplication();
            }
            return _applicationServices;
        }

        public IDisposable Run()
        {
            return this;
        }

        private void EnsureApplicationServices()
        {
            if (_applicationServices == null)
            {
                EnsureStartup();
                _applicationServices = _startup.ConfigureServices(_builder);
            }
        }
        private void EnsureStartup()
        {
            if (_startup != null)
            {
                return;
            }

            _startup = _hostingServiceProvider.GetRequiredService<IStartup>();
        }
        private IServiceProvider BuildApplication()
        {
            try
            {
                EnsureApplicationServices();
                Action<IServiceProvider> configure = _startup.Configure;
                if (_applicationServices == null)
                    _applicationServices = _builder.BuildServiceProvider();
                configure(_applicationServices);
                return _applicationServices;
            }
            catch (Exception ex)
            {
                Console.Out.WriteLine("应用程序启动异常: " + ex.ToString());
                throw;
            }
        }
    }
}
```

`_builder`包含了`ServiceHostBuilder`方法`RegisterServices(Action<IServiceCollection> configureServices)`注册的服务。

`_hostingServiceProvider`为包含了`ServiceHostBuilder`方法`ConfigureServices(Action<IServiceCollection> configureServices)`注册的服务的IOC容器实例(可取到相应服务实例)。

`_applicationServices`为包含了`ServiceHostBuilder`方法`RegisterServices(Action<IServiceCollection> configureServices)`与`IStartup`方法`ConfigureServices(IServiceCollection services)`注册的服务的IOC容器实例。

调用过程：

* `Initialize`，赋值`_applicationServices`。
 * `BuildApplication`，返回`_builder`方法`BuildServiceProvider`生成的`IServiceProvider`实例。
   * `EnsureApplicationServices`，从`_hostingServiceProvider`取出`IStartup`服务。执行`IStartup`的`ConfigureServices`方法，返回`IServiceProvider`实例，并赋值给`_applicationServices`。
   * 如果`_applicationServices`没有被赋值，调用`_builder`方法`BuildServiceProvider`赋值。执行`IStartup`实例的`Configure`方法，`_applicationServices`作为参数。

### 扩展方法UseStartup

`Startup`可以实现`IStartup`，也可以不实现，但必须具备`ConfigureServices(IServiceCollection services)`与`Configure(IServiceProvider app)`方法。

```csharp
using CustomHost.Internal;
using CustomHost.Internal.Implementation;
using CustomHost.Startup;
using CustomHost.Startup.Implementation;
using Microsoft.Extensions.DependencyInjection;
using System;
using System.Reflection;

namespace CustomHost
{
    public static class ServiceHostBuilderExtensions
    {
        public static IServiceHostBuilder UseStartup(this IServiceHostBuilder hostBuilder, Type startupType)
        {
            return hostBuilder
                .ConfigureServices(services =>
                {
                    if (typeof(IStartup).GetTypeInfo().IsAssignableFrom(startupType.GetTypeInfo()))
                    {
                        services.AddSingleton(typeof(IStartup), startupType);
                    }
                    else
                    {
                        services.AddSingleton(typeof(IStartup), sp =>
                        {
                            return new ConventionBasedStartup(StartupLoader.LoadMethods(sp, startupType, ""));
                        });

                    }
                });
        }

        public static IServiceHostBuilder UseStartup<TStartup>(this IServiceHostBuilder hostBuilder) where TStartup : class
        {
            return hostBuilder.UseStartup(typeof(TStartup));
        }
    }
}

```
```csharp
public class StartupImplementation: IStartup
{
    public IServiceProvider ConfigureServices(IServiceCollection services)
    {
        //services.AddScoped<MyService>();
        return services.BuildServiceProvider();
    }

    public void Configure(IServiceProvider app)
    {
        var myService = app.GetService<MyService>();
        myService.WriteMessage("This is a message");
    }
}
```