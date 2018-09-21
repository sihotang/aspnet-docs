---
title: ASP.NET Core Middleware
author: rick-anderson
description: Learn about ASP.NET Core middleware and the request pipeline.
ms.author: riande
ms.custom: mvc
ms.date: 08/21/2018
uid: fundamentals/middleware/index
---
# ASP.NET Core Middleware

By [Rick Anderson](https://twitter.com/RickAndMSFT) and [Steve Smith](https://ardalis.com/)

Middleware is software that's assembled into an app pipeline to handle requests and responses. Each component:

* Chooses whether to pass the request to the next component in the pipeline.
* Can perform work before and after the next component in the pipeline is invoked.

Request delegates are used to build the request pipeline. The request delegates handle each HTTP request.

Request delegates are configured using <xref:Microsoft.AspNetCore.Builder.RunExtensions.Run*>, <xref:Microsoft.AspNetCore.Builder.MapExtensions.Map*>, and <xref:Microsoft.AspNetCore.Builder.UseExtensions.Use*> extension methods. An individual request delegate can be specified in-line as an anonymous method (called in-line middleware), or it can be defined in a reusable class. These reusable classes and in-line anonymous methods are *middleware*, also called *middleware components*. Each middleware component in the request pipeline is responsible for invoking the next component in the pipeline or short-circuiting the pipeline.

<xref:migration/http-modules> explains the difference between request pipelines in ASP.NET Core and ASP.NET 4.x and provides more middleware samples.

## Create a middleware pipeline with IApplicationBuilder

The ASP.NET Core request pipeline consists of a sequence of request delegates, called one after the other. The following diagram demonstrates the concept. The thread of execution follows the black arrows.

![Request processing pattern showing a request arriving, processing through three middlewares, and the response leaving the app. Each middleware runs its logic and hands off the request to the next middleware at the next() statement. After the third middleware processes the request, the request passes back through the prior two middlewares in reverse order for additional processing after their next() statements before leaving the app as a response to the client.](index/_static/request-delegate-pipeline.png)

Each delegate can perform operations before and after the next delegate. A delegate can also decide to not pass a request to the next delegate, which is called *short-circuiting the request pipeline*. Short-circuiting is often desirable because it avoids unnecessary work. For example, Static Files Middleware can return a request for a static file and short-circuit the rest of the pipeline. Exception-handling delegates are called early in the pipeline, so they can catch exceptions that occur in later stages of the pipeline.

The simplest possible ASP.NET Core app sets up a single request delegate that handles all requests. This case doesn't include an actual request pipeline. Instead, a single anonymous function is called in response to every HTTP request.

[!code-csharp[](index/snapshot/Middleware/Startup.cs?name=snippet1)]

The first <xref:Microsoft.AspNetCore.Builder.RunExtensions.Run*> delegate terminates the pipeline.

Chain multiple request delegates together with <xref:Microsoft.AspNetCore.Builder.UseExtensions.Use*>. The `next` parameter represents the next delegate in the pipeline. You can short-circuit the pipeline by *not* calling the *next* parameter. You can typically perform actions both before and after the next delegate, as the following example demonstrates:

[!code-csharp[](index/snapshot/Chain/Startup.cs?name=snippet1)]

> [!WARNING]
> Don't call `next.Invoke` after the response has been sent to the client. Changes to <xref:Microsoft.AspNetCore.Http.HttpResponse> after the response has started throw an exception. For example, changes such as setting headers and a status code throw an exception. Writing to the response body after calling `next`:
>
> * May cause a protocol violation. For example, writing more than the stated `Content-Length`.
> * May corrupt the body format. For example, writing an HTML footer to a CSS file.
>
> <xref:Microsoft.AspNetCore.Http.HttpResponse.HasStarted*> is a useful hint to indicate if headers have been sent or the body has been written to.

## Order

The order that middleware components are added in the `Startup.Configure` method defines the order in which the middleware components are invoked on requests and the reverse order for the response. The order is critical for security, performance, and functionality.

The following `Startup.Configure` method adds middleware components for common app scenarios:

::: moniker range=">= aspnetcore-2.0"

1. Exception/error handling
1. HTTP Strict Transport Security Protocol
1. HTTPS redirection
1. Static file server
1. Cookie policy enforcement
1. Authentication
1. Session
1. MVC

```csharp
public void Configure(IApplicationBuilder app)
{
    if (env.IsDevelopment())
    {
        // When the app runs in the Development environment:
        //   Use the Developer Exception Page to report app runtime errors.
        //   Use the Database Error Page to report database runtime errors.
        app.UseDeveloperExceptionPage();
        app.UseDatabaseErrorPage();
    }
    else
    {
        // When the app doesn't run in the Development environment:
        //   Enable the Exception Handler Middleware to catch exceptions
        //     thrown in the following middlewares.
        //   Use the HTTP Strict Transport Security Protocol (HSTS)
        //     Middleware.
        app.UseExceptionHandler("/Error");
        app.UseHsts();
    }

    // Use HTTPS Redirection Middleware to redirect HTTP requests to HTTPS.
    app.UseHttpsRedirection();

    // Return static files and end the pipeline.
    app.UseStaticFiles();

    // Use Cookie Policy Middleware to conform to EU General Data 
    // Protection Regulation (GDPR) regulations.
    app.UseCookiePolicy();

    // Authenticate before the user accesses secure resources.
    app.UseAuthentication();

    // If the app uses session state, call Session Middleware after Cookie 
    // Policy Middleware and before MVC Middleware.
    app.UseSession();

    // Add MVC to the request pipeline.
    app.UseMvc();
}
```

::: moniker-end

::: moniker range="< aspnetcore-2.0"

1. Exception/error handling
1. Static files
1. Authentication
1. Session
1. MVC

```csharp
public void Configure(IApplicationBuilder app)
{
    // Enable the Exception Handler Middleware to catch exceptions
    //   thrown in the following middlewares.
    app.UseExceptionHandler("/Home/Error");

    // Return static files and end the pipeline.
    app.UseStaticFiles();

    // Authenticate before you access secure resources.
    app.UseIdentity();

    // If the app uses session state, call UseSession before 
    // MVC Middleware.
    app.UseSession();

    // Add MVC to the request pipeline.
    app.UseMvcWithDefaultRoute();
}
```

::: moniker-end

In the preceding example code, each middleware extension method is exposed on <xref:Microsoft.AspNetCore.Builder.IApplicationBuilder> through the <xref:Microsoft.AspNetCore.Builder?displayProperty=fullName> namespace.

<xref:Microsoft.AspNetCore.Builder.ExceptionHandlerExtensions.UseExceptionHandler*> is the first middleware component added to the pipeline. Therefore, the Exception Handler Middleware catches any exceptions that occur in later calls.

Static Files Middleware is called early in the pipeline so that it can handle requests and short-circuit without going through the remaining components. The Static Files Middleware provides **no** authorization checks. Any files served by it, including those under *wwwroot*, are publicly available. For an approach to secure static files, see <xref:fundamentals/static-files>.

::: moniker range=">= aspnetcore-2.0"

If the request isn't handled by the Static Files Middleware, it's passed on to the Authentication Middleware (<xref:Microsoft.AspNetCore.Builder.AuthAppBuilderExtensions.UseAuthentication*>), which performs authentication. Authentication doesn't short-circuit unauthenticated requests. Although Authentication Middleware authenticates requests, authorization (and rejection) occurs only after MVC selects a specific Razor Page or MVC controller and action.

::: moniker-end

::: moniker range="< aspnetcore-2.0"

If the request isn't handled by Static Files Middleware, it's passed on to the Identity Middleware (<xref:Microsoft.AspNetCore.Builder.BuilderExtensions.UseIdentity*>), which performs authentication. Identity doesn't short-circuit unauthenticated requests. Although Identity authenticates requests, authorization (and rejection) occurs only after MVC selects a specific controller and action.

::: moniker-end

The following example demonstrates a middleware order where requests for static files are handled by Static Files Middleware before Response Compression Middleware. Static files aren't compressed with this middleware order. The MVC responses from <xref:Microsoft.AspNetCore.Builder.MvcApplicationBuilderExtensions.UseMvcWithDefaultRoute*> can be compressed.

```csharp
public void Configure(IApplicationBuilder app)
{
    // Static files not compressed by Static Files Middleware.
    app.UseStaticFiles();
    app.UseResponseCompression();
    app.UseMvcWithDefaultRoute();
}
```

### Use, Run, and Map

Configure the HTTP pipeline using `Use`, `Run`, and `Map`. The `Use` method can short-circuit the pipeline (that is, if it doesn't call a `next` request delegate). `Run` is a convention, and some middleware components may expose `Run[Middleware]` methods that run at the end of the pipeline.

<xref:Microsoft.AspNetCore.Builder.MapExtensions.Map*> extensions are used as a convention for branching the pipeline. `Map*` branches the request pipeline based on matches of the given request path. If the request path starts with the given path, the branch is executed.

[!code-csharp[](index/snapshot/Chain/StartupMap.cs?name=snippet1)]

The following table shows the requests and responses from `http://localhost:1234` using the previous code.

| Request             | Response                     |
| ------------------- | ---------------------------- |
| localhost:1234      | Hello from non-Map delegate. |
| localhost:1234/map1 | Map Test 1                   |
| localhost:1234/map2 | Map Test 2                   |
| localhost:1234/map3 | Hello from non-Map delegate. |

When `Map` is used, the matched path segment(s) are removed from `HttpRequest.Path` and appended to `HttpRequest.PathBase` for each request.

[MapWhen](/dotnet/api/microsoft.aspnetcore.builder.mapwhenextensions) branches the request pipeline based on the result of the given predicate. Any predicate of type `Func<HttpContext, bool>` can be used to map requests to a new branch of the pipeline. In the following example, a predicate is used to detect the presence of a query string variable `branch`:

[!code-csharp[](index/snapshot/Chain/StartupMapWhen.cs?name=snippet1)]

The following table shows the requests and responses from `http://localhost:1234` using the previous code.

| Request                       | Response                     |
| ----------------------------- | ---------------------------- |
| localhost:1234                | Hello from non-Map delegate. |
| localhost:1234/?branch=master | Branch used = master         |

`Map` supports nesting, for example:

```csharp
app.Map("/level1", level1App => {
    level1App.Map("/level2a", level2AApp => {
        // "/level1/level2a" processing
    });
    level1App.Map("/level2b", level2BApp => {
        // "/level1/level2b" processing
    });
});
   ```

`Map` can also match multiple segments at once:

[!code-csharp[](index/snapshot/Chain/StartupMultiSeg.cs?name=snippet1&highlight=13)]

## Built-in middleware

ASP.NET Core ships with the following middleware components. The *Order* column provides notes on the middleware's placement in the request pipeline and under what conditions the middleware may terminate the request and prevent other middleware from processing a request.

| Middleware | Description | Order |
| ---------- | ----------- | ----- |
| [Authentication](xref:security/authentication/identity) | Provides authentication support. | Before `HttpContext.User` is needed. Terminal for OAuth callbacks. |
| [Cookie Policy](xref:security/gdpr) | Tracks consent from users for storing personal information and enforces minimum standards for cookie fields, such as `secure` and `SameSite`. | Before middleware that issues cookies. Examples: Authentication, Session, MVC (TempData). |
| [CORS](xref:security/cors) | Configures Cross-Origin Resource Sharing. | Before components that use CORS. |
| [Diagnostics](xref:fundamentals/error-handling) | Configures diagnostics. | Before components that generate errors. |
| [Forwarded Headers](/dotnet/api/microsoft.aspnetcore.builder.forwardedheadersextensions) | Forwards proxied headers onto the current request. | Before components that consume the updated fields. Examples: scheme, host, client IP, method. |
| [HTTP Method Override](/dotnet/api/microsoft.aspnetcore.builder.httpmethodoverrideextensions) | Allows an incoming POST request to override the method. | Before components that consume the updated method. |
| [HTTPS Redirection](xref:security/enforcing-ssl#require-https) | Redirect all HTTP requests to HTTPS (ASP.NET Core 2.1 or later). | Before components that consume the URL. |
| [HTTP Strict Transport Security (HSTS)](xref:security/enforcing-ssl#http-strict-transport-security-protocol-hsts) | Security enhancement middleware that adds a special response header (ASP.NET Core 2.1 or later). | Before responses are sent and after components that modify requests. Examples: Forwarded Headers, URL Rewriting. |
| [MVC](xref:mvc/overview) | Processes requests with MVC/Razor Pages (ASP.NET Core 2.0 or later). | Terminal if a request matches a route. |
| [OWIN](xref:fundamentals/owin) | Interop with OWIN-based apps, servers, and middleware. | Terminal if the OWIN Middleware fully processes the request. |
| [Response Caching](xref:performance/caching/middleware) | Provides support for caching responses. | Before components that require caching. |
| [Response Compression](xref:performance/response-compression) | Provides support for compressing responses. | Before components that require compression. |
| [Request Localization](xref:fundamentals/localization) | Provides localization support. | Before localization sensitive components. |
| [Routing](xref:fundamentals/routing) | Defines and constrains request routes. | Terminal for matching routes. |
| [Session](xref:fundamentals/app-state) | Provides support for managing user sessions. | Before components that require Session. |
| [Static Files](xref:fundamentals/static-files) | Provides support for serving static files and directory browsing. | Terminal if a request matches a file. |
| [URL Rewriting](xref:fundamentals/url-rewriting) | Provides support for rewriting URLs and redirecting requests. | Before components that consume the URL. |
| [WebSockets](xref:fundamentals/websockets) | Enables the WebSockets protocol. | Before components that are required to accept WebSocket requests. |

## Write middleware

Middleware is generally encapsulated in a class and exposed with an extension method. Consider the following middleware, which sets the culture for the current request from a query string:

[!code-csharp[](index/snapshot/Culture/StartupCulture.cs?name=snippet1)]

The preceding sample code is used to demonstrate creating a middleware component. For ASP.NET Core's built-in localization support, see <xref:fundamentals/localization>.

You can test the middleware by passing in the culture, for example `http://localhost:7997/?culture=no`.

The following code moves the middleware delegate to a class:

[!code-csharp[](index/snapshot/Culture/RequestCultureMiddleware.cs)]

::: moniker range="< aspnetcore-2.0"

The middleware `Task` method's name must be `Invoke`. In ASP.NET Core 2.0 or later, the name can be either `Invoke` or `InvokeAsync`.

::: moniker-end

The following extension method exposes the middleware through <xref:Microsoft.AspNetCore.Builder.IApplicationBuilder>:

[!code-csharp[](index/snapshot/Culture/RequestCultureMiddlewareExtensions.cs)]

The following code calls the middleware from `Startup.Configure`:

[!code-csharp[](index/snapshot/Culture/Startup.cs?name=snippet1&highlight=5)]

Middleware should follow the [Explicit Dependencies Principle](/dotnet/standard/modern-web-apps-azure-architecture/architectural-principles#explicit-dependencies) by exposing its dependencies in its constructor. Middleware is constructed once per *application lifetime*. See the [Per-request dependencies](#per-request-dependencies) section if you need to share services with middleware within a request.

Middleware components can resolve their dependencies from [dependency injection (DI)](xref:fundamentals/dependency-injection) through constructor parameters. [UseMiddleware&lt;T&gt;](/dotnet/api/microsoft.aspnetcore.builder.usemiddlewareextensions.usemiddleware#Microsoft_AspNetCore_Builder_UseMiddlewareExtensions_UseMiddleware_Microsoft_AspNetCore_Builder_IApplicationBuilder_System_Type_System_Object___) can also accept additional parameters directly.

### Per-request dependencies

Because middleware is constructed at app startup, not per-request, *scoped* lifetime services used by middleware constructors aren't shared with other dependency-injected types during each request. If you must share a *scoped* service between your middleware and other types, add these services to the `Invoke` method's signature. The `Invoke` method can accept additional parameters that are populated by DI:

```csharp
public class CustomMiddleware
{
    private readonly RequestDelegate _next;

    public CustomMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    // IMyScopedService is injected into Invoke
    public async Task Invoke(HttpContext httpContext, IMyScopedService svc)
    {
        svc.MyProperty = 1000;
        await _next(httpContext);
    }
}
```

## Additional resources

* <xref:migration/http-modules>
* <xref:fundamentals/startup>
* <xref:fundamentals/request-features>
* <xref:fundamentals/middleware/extensibility>
* <xref:fundamentals/middleware/extensibility-third-party-container>
