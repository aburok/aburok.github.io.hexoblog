---
title: Introducing Windsor Dependency Injection container into Umbraco application.
tags:
  - Umbraco
  - Dependency Injection
  - Windsor
date: 2016-06-09 14:23:26
---


# IoC

When I first saw the Dependency Injection pattern used in a source code, I did not understand what I was looking at. For a beginner, it just seemed as an unnecessary complication made be some senior developer, to make my life even harder. Passing some objects into the service just to call their methods. Making a complicated constructor to pass 15 other dependencies. But I was young and some time had to pass before I had [the aha moment][eureka] and finally understood what it is all about.
After that, it all made perfect sense. Instead of creating lot of nested code, that is [tightly coupled][coupling], with static method, you just pass required dependencies into the object. With that, you can create the code in the way that will allow you to focus of writing and testing only one functionality at a time. You can easily [mock][mocking] other dependencies that are not relevant at the moment. It introduces a clarity in the code.

# Umbraco and IoC

Umbraco documentation states as following:

> We don't use IoC in the Umbraco source code. This isn't because we don't like it or don't want to use it, it's because we want you as a developer to be able to use whatever IoC framework that you would like to use without jumping through any hoops.
> With that said, it means it is possible to implement whatever IoC engine that you'd like!

With that in mind, I would like to show you how to add the [Windsor][windsor] Dependency Injection container into an Umbraco web application as an example.

# Adding IoC into Umbraco

First step to add Windsor Container is to create class that extends [`UmbracoApplication`][umbracoAppClass] class. This gives as ability to override method [`OnApplicationStarted`][applicationStartedEvent]. In this method we can replace both default MVC and WebApi Dependency Resolvers.

``` csharp
protected override void OnApplicationStarted(object sender, EventArgs e)
{
    base.OnApplicationStarted(sender, e);

    var resolver = new WindsorDependencyResolver(_container.Value, DependencyResolver.Current);
    DependencyResolver.SetResolver(resolver);
    GlobalConfiguration.Configuration.DependencyResolver = resolver;
}
```

Second thing to do is replacing default UmbracoApplication in `Global.asax` file:

``` csharp
<%@ Application Inherits="UmbracoDemo.DemoApplication" Language="C#" %>
```

`WindsorDependencyResolver` is [adapter][adapterPattern] class that allows us to use Windsor as dependency container in MVC application.

``` csharp
public class WindsorDependencyResolver :
    System.Web.Mvc.IDependencyResolver,
    System.Web.Http.Dependencies.IDependencyResolver
{
    private readonly IWindsorContainer _container;
    private readonly System.Web.Mvc.IDependencyResolver _mvcDependencyResolver;
    private readonly IDependencyResolver _webApiDependencyResolver;

    private readonly IDependencyScope _lifeTimeScope;

    public WindsorDependencyResolver(
        IWindsorContainer container,
        System.Web.Mvc.IDependencyResolver mvcDependencyResolver,
        System.Web.Http.Dependencies.IDependencyResolver webApiDependencyResolver)
    {
        _container = container;
        _mvcDependencyResolver = mvcDependencyResolver;
        _webApiDependencyResolver = webApiDependencyResolver;

        _lifeTimeScope = new WindsorWebApiDependencyScope(
            _container,
            _mvcDependencyResolver,
            _webApiDependencyResolver);
    }

    public object GetService(Type serviceType)
    {
        return _lifeTimeScope.GetService(serviceType);
    }

    public IEnumerable<object> GetServices(Type serviceType)
    {
        return _lifeTimeScope.GetServices(serviceType);
    }

    public System.Web.Http.Dependencies.IDependencyScope BeginScope()
    {
        return new WindsorWebApiDependencyScope(_container,
            _mvcDependencyResolver,
            _webApiDependencyResolver);
    }

    public void Dispose()
    {
    }
}
```

Constructor in `WindsorDependencyResolver` takes three parameters, first is the Windsor container, second one is the default [MVC Dependency Resolver][mvcDependencyResolver] and the third is the default [WebApi Dependency Resolver][webApiDependencyResolver]. The latter is used to first check if the service is not registered there.

The `BeginScope` method returns adapter class of Windsor container for WebAPI `IDependencyScope`. It allows to restrict service lifetime to a scope. It can be useful for situation when you don't want to create singletons and also you don't want service to be created every time it is being referenced. For example, we can create a scope for a http request and allow all methods/classes to use same instances during one request.

```csharp
public class WindsorWebApiDependencyScope :
        System.Web.Http.Dependencies.IDependencyScope
{
    private readonly IWindsorContainer _windsorContainer;

    private readonly System.Web.Mvc.IDependencyResolver _mvcDependencyResolver;
    private readonly IDependencyResolver _webApiDependencyResolver;
    private readonly IDisposable _disposable;


    public WindsorWebApiDependencyScope(
        IWindsorContainer container,
        System.Web.Mvc.IDependencyResolver mvcDependencyResolver,
        System.Web.Http.Dependencies.IDependencyResolver webApiDependencyResolver)
    {
        _mvcDependencyResolver = mvcDependencyResolver;
        _webApiDependencyResolver = webApiDependencyResolver;
        _windsorContainer = container;

        _disposable = _windsorContainer.BeginScope();
    }

    public object GetService(Type serviceType)
    {
        try
        {
            var serviceFromWindsor = _windsorContainer.Resolve(serviceType);
            if (serviceFromWindsor != null)
                return serviceFromWindsor;
        }
        catch (Exception) { /* ignored */ }

        try
        {
            var serviceFromMvc = _mvcDependencyResolver.GetService(serviceType);
            if (serviceFromMvc != null)
                return serviceFromMvc;
        }
        catch (Exception) { /*ignored*/ }

        try
        {
            var serviceFromWebApi = _webApiDependencyResolver.GetService(serviceType);
            if (serviceFromWebApi != null)
                return serviceFromWebApi;
        }
        catch (Exception) { /* ignored */ }

        return null;
    }

    public IEnumerable<object> GetServices(Type serviceType)
    {
        try
        {
            var servicesFromWindsor = _windsorContainer.ResolveAll(serviceType)
                .Cast<object>();
            if (servicesFromWindsor != null && servicesFromWindsor.Any())
                return servicesFromWindsor;
        }
        catch (Exception) { /* ignored */ }

        try
        {
            var currentServiceList = _mvcDependencyResolver.GetServices(serviceType);
            if (currentServiceList != null && currentServiceList.Any())
                return currentServiceList;
        }
        catch (Exception) { /* ignored */ }

        try
        {
            var webApiServices = _webApiDependencyResolver.GetServices(serviceType);
            if (webApiServices != null && webApiServices.Any())
                return webApiServices;
        }
        catch (Exception) { /* ignored */ }


        return Enumerable.Empty<object>();
    }

    public void Dispose()
    {
        _disposable.Dispose();
    }
}
```

As you can see above, ther order of services lookup is as following :
- Windsor Container
- Default Mvc Dependency Resolver
- Default WebAPI Dependency Resolver.


## Register Controllers

Registering custom controllers looks like this:

``` csharp

private static void RegisterStandardClasses(WindsorContainer container)
{
    // Register all controllers from current assembly
    RegisterFromAssembly<DemoApplication, ApiController>(container);
    RegisterFromAssembly<DemoApplication, Controller>(container);
    RegisterFromAssembly<DemoApplication, SurfaceController>(container);
    RegisterFromAssembly<DemoApplication, RenderMvcController>(container);
}
```

`RegisterFromAssembly` method is a static method that I've created to easily add controllers.

``` csharp
private static void RegisterFromAssembly<TAssemblyClass, TService>(IWindsorContainer container)
{
    container
        .Register(Classes.FromAssemblyContaining<TAssemblyClass>()
            .BasedOn<TService>()
            .LifestyleTransient());
}
```

## Using Umbraco services interfaces

To register custom services you must call the Register method on the container. The `Component` class is a static class that serves as a [Factory class][factoryPattern] and allows to connect interface with it's concrete implementation. Also we need can define the lifetime of the registered class.

``` csharp
public static IWindsorContainer Initialize()
{
    var container = new WindsorContainer();

    RegisterStandardClasses(container);

    container.Register(Component
        .For<IEmailService>()
        .ImplementedBy<EmailService>()
        .LifestyleSingleton());

    container.Register(Component
        .For<ICityRepository>()
        .ImplementedBy<CityRepository>()
        .LifestyleSingleton());

    return container;
}
```

## Using custom services

To use custom services in your [controllers][mvcController] and [filters][mvcFilter] you just need to pass it into the constructor of the controller/filter.

``` csharp
public class ExampleController : SurfaceController
{
    private readonly IContentService _contentService;
    private readonly ICityRepository _cityRepository;

    public ExampleController(
        IContentService contentService,
        ICityRepository cityRepository)
    {
        _contentService = contentService;
        _cityRepository = cityRepository;
    }

    // GET: Example
    public ActionResult Index()
    {
        var rootContent = _contentService.GetRootContent();
        ViewBag.Cities = _cityRepository.GetCities();

        return View(rootContent);
    }
}
```

That would be it. Thanks for reading :)


[1]:https://our.umbraco.org/documentation/reference/using-ioc
[eureka]:https://en.wikipedia.org/wiki/Eureka_effect
[coupling]:http://stackoverflow.com/questions/2832017/what-is-the-difference-between-loose-coupling-and-tight-coupling-in-object-orien
[mocking]:https://en.wikipedia.org/wiki/Mock_object
[windsor]:http://www.castleproject.org/
[umbracoAppClass]:https://github.com/umbraco/Umbraco-CMS/blob/dev-v7/src/Umbraco.Web/UmbracoApplication.cs
[applicationStartedEvent]:https://github.com/umbraco/Umbraco-CMS/blob/dev-v7/src/Umbraco.Core/UmbracoApplicationBase.cs#L26
[adapterPattern]:https://en.wikipedia.org/wiki/Adapter_pattern
[mvcDependencyResolver]:https://msdn.microsoft.com/en-us/library/system.web.mvc.dependencyresolver(v=vs.118).aspx
[webApiDependencyResolver]:https://code.msdn.microsoft.com/ASP-NET-Web-API-Tutorial-468ee148
[factoryPattern]:https://en.wikipedia.org/wiki/Factory_method_pattern
[mvcController]:https://docs.asp.net/en/latest/mvc/controllers/actions.html
[mvcFilter]:https://docs.asp.net/en/latest/mvc/controllers/filters.html
