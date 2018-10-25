---
title: Build web APIs with ASP.NET Core
author: scottaddie
description: Learn about the features available for building a web API in ASP.NET Core and when it's appropriate to use each feature.
ms.author: scaddie
ms.custom: mvc
ms.date: 08/15/2018
uid: web-api/index
---
# Build web APIs with ASP.NET Core

By [Scott Addie](https://github.com/scottaddie)

[View or download sample code](https://github.com/aspnet/Docs/tree/master/aspnetcore/web-api/define-controller/samples) ([how to download](xref:tutorials/index#how-to-download-a-sample))

This document explains how to build a web API in ASP.NET Core and when it's most appropriate to use each feature.

## Derive class from ControllerBase

Inherit from the <xref:Microsoft.AspNetCore.Mvc.ControllerBase> class in a controller that's intended to serve as a web API. For example:

::: moniker range=">= aspnetcore-2.1"

[!code-csharp[](../web-api/define-controller/samples/WebApiSample.Api/Controllers/PetsController.cs?name=snippet_PetsController&highlight=3)]

::: moniker-end

::: moniker range="<= aspnetcore-2.0"

[!code-csharp[](../web-api/define-controller/samples/WebApiSample.Api.Pre21/Controllers/PetsController.cs?name=snippet_PetsController&highlight=3)]

::: moniker-end

The `ControllerBase` class provides access to several properties and methods. In the preceding code, examples include <xref:Microsoft.AspNetCore.Mvc.ControllerBase.BadRequest(Microsoft.AspNetCore.Mvc.ModelBinding.ModelStateDictionary)> and <xref:Microsoft.AspNetCore.Mvc.ControllerBase.CreatedAtAction(System.String,System.Object,System.Object)>. These methods are called within action methods to return HTTP 400 and 201 status codes, respectively. The <xref:Microsoft.AspNetCore.Mvc.ControllerBase.ModelState> property, also provided by `ControllerBase`, is accessed to handle request model validation.

::: moniker range=">= aspnetcore-2.1"

## Annotate class with ApiControllerAttribute

ASP.NET Core 2.1 introduces the [[ApiController]](xref:Microsoft.AspNetCore.Mvc.ApiControllerAttribute) attribute to denote a web API controller class. For example:

[!code-csharp[](../web-api/define-controller/samples/WebApiSample.Api/Controllers/ProductsController.cs?name=snippet_ControllerSignature&highlight=2)]

A compatibility version of 2.1 or later, set via <xref:Microsoft.Extensions.DependencyInjection.MvcCoreMvcBuilderExtensions.SetCompatibilityVersion*>, is required to use this attribute. For example, the highlighted code in *Startup.ConfigureServices* sets the 2.2 compatibility flag:

[!code-csharp[](../web-api/define-controller/samples/WebApiSample.Api/Startup.cs?name=snippet_SetCompatibilityVersion&highlight=2)]

For more information, see <xref:mvc/compatibility-version>.

The `[ApiController]` attribute is commonly coupled with `ControllerBase` to enable REST-specific behavior for controllers. `ControllerBase` provides access to methods such as <xref:Microsoft.AspNetCore.Mvc.ControllerBase.NotFound*> and <xref:Microsoft.AspNetCore.Mvc.ControllerBase.File*>.

Another approach is to create a custom base controller class annotated with the `[ApiController]` attribute:

[!code-csharp[](../web-api/define-controller/samples/WebApiSample.Api/Controllers/MyBaseController.cs?name=snippet_ControllerSignature)]

The following sections describe convenience features added by the attribute.

### Problem details responses for error status codes

ASP.NET Core 2.1 and later includes [ProblemDetails](xref:Microsoft.AspNetCore.Mvc.ProblemDetails), a type based on the [RFC 7807 specification](https://tools.ietf.org/html/rfc7807). The `ProblemDetails` type provides a standardized format for conveying machine readable details of errors in a HTTP response.

In ASP.NET Core 2.2 and later, MVC transforms error status code results (status code 400 and higher) to a result with `ProblemDetails`. Consider the following code:

[!code-csharp[](../web-api/define-controller/samples/WebApiSample.Api/Controllers/PetsController.cs?name=snippet_ProblemDetails_StatusCode&highlight=4)]

The HTTP response for the `NotFound` result has a 404 status code with a `ProblemDetails` body similar to the following:

```js
{
    type: "https://tools.ietf.org/html/rfc7231#section-6.5.4",
    title: "Not Found",
    status: 404,
    traceId: "0HLHLV31KRN83:00000001"
}
```

The problem details feature requires a compatibility flag of 2.2 or later. The default behavior is disabled when the [SuppressMapClientErrors](/dotnet/api/microsoft.aspnetcore.Mvc.ApiBehaviorOptions) <!--  Until these resolve, link to the parent class <xref:Microsoft.AspNetCore.Mvc.ApiBehaviorOptions.SuppressMapClientErrors> --> property is set to `true`. The following highlighted code from `Startup.ConfigureServices` disables problem details:

[!code-csharp[](../web-api/define-controller/samples/WebApiSample.Api/Startup.cs?name=snippet_SetCompatibilityVersion&highlight=8)]

Use the [ClientErrorMapping](/dotnet/api/microsoft.aspnetcore.Mvc.ApiBehaviorOptions) <!--  Until these resolve, link to the parent class <xref:Microsoft.AspNetCore.Mvc.ApiBehaviorOptions.ClientErrorMapping> --> property to configure the contents of the `ProblemDetails` response. For example, the following code updates the `type` property for 404 responses:

[!code-csharp[](../web-api/define-controller/samples/WebApiSample.Api/Startup.cs?name=snippet_SetCompatibilityVersion&highlight=10)]

### Automatic HTTP 400 responses

Validation errors automatically trigger an HTTP 400 response. The following code becomes unnecessary in your actions:

[!code-csharp[](../web-api/define-controller/samples/WebApiSample.Api.Pre21/Controllers/PetsController.cs?name=snippet_ModelStateIsValidCheck)]

Use <xref:Microsoft.AspNetCore.Mvc.ApiBehaviorOptions.InvalidModelStateResponseFactory> to customize the output of the the resulting response.

The default behavior is disabled when the <xref:Microsoft.AspNetCore.Mvc.ApiBehaviorOptions.SuppressModelStateInvalidFilter> property is set to `true`. Add the following code in *Startup.ConfigureServices* after `services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_1);`:

[!code-csharp[](../web-api/define-controller/samples/WebApiSample.Api/Startup.cs?name=snippet_ConfigureApiBehaviorOptions&highlight=5)]

With a compatibility flag of 2.2 or later, the default response type returned for 400 responses is a <xref:Microsoft.AspNetCore.Mvc.ValidationProblemDetails>. Use the Use the [SuppressUseValidationProblemDetailsForInvalidModelStateResponses](/dotnet/api/microsoft.aspnetcore.Mvc.ApiBehaviorOptions) <!--  <xref:Microsoft.AspNetCore.Mvc.ApiBehaviorOptions.SuppressUseValidationProblemDetailsForInvalidModelStateResponses> --> property to use the ASP.NET Core 2.1 error format.

### Binding source parameter inference

A binding source attribute defines the location at which an action parameter's value is found. The following binding source attributes exist:

|Attribute|Binding source |
|---------|---------|
|**[[FromBody]](xref:Microsoft.AspNetCore.Mvc.FromBodyAttribute)**     | Request body |
|**[[FromForm]](xref:Microsoft.AspNetCore.Mvc.FromFormAttribute)**     | Form data in the request body |
|**[[FromHeader]](xref:Microsoft.AspNetCore.Mvc.FromHeaderAttribute)** | Request header |
|**[[FromQuery]](xref:Microsoft.AspNetCore.Mvc.FromQueryAttribute)**   | Request query string parameter |
|**[[FromRoute]](xref:Microsoft.AspNetCore.Mvc.FromRouteAttribute)**   | Route data from the current request |
|**[[FromServices]](xref:mvc/controllers/dependency-injection#action-injection-with-fromservices)** | The request service injected as an action parameter |

> [!WARNING]
> Don't use `[FromRoute]` when values might contain `%2f` (that is `/`). `%2f` won't be unescaped to `/`. Use `[FromQuery]` if the value might contain `%2f`.

Without the `[ApiController]` attribute, binding source attributes are explicitly defined. In the following example, the `[FromQuery]` attribute indicates that the `discontinuedOnly` parameter value is provided in the request URL's query string:

[!code-csharp[](../web-api/define-controller/samples/WebApiSample.Api/Controllers/ProductsController.cs?name=snippet_BindingSourceAttributes&highlight=3)]

Inference rules are applied for the default data sources of action parameters. These rules configure the binding sources you're otherwise likely to manually apply to the action parameters. The binding source attributes behave as follows:

* **[FromBody]** is inferred for complex type parameters. An exception to this rule is any complex, built-in type with a special meaning, such as <xref:Microsoft.AspNetCore.Http.IFormCollection> and <xref:System.Threading.CancellationToken>. The binding source inference code ignores those special types. `[FromBody]` isn't inferred for simple types such as `string` or `int`. Therefore, the `[FromBody]` attribute should be used for simple types when that functionality is desired. When an action has more than one parameter explicitly specified (via `[FromBody]`) or inferred as bound from the request body, an exception is thrown. For example, the following action signatures cause an exception:

[!code-csharp[](../web-api/define-controller/samples/WebApiSample.Api/Controllers/TestController.cs?name=snippet_ActionsCausingExceptions)]

* **[FromForm]** is inferred for action parameters of type <xref:Microsoft.AspNetCore.Http.IFormFile> and <xref:Microsoft.AspNetCore.Http.IFormFileCollection>. It's not inferred for any simple or user-defined types.
* **[FromRoute]** is inferred for any action parameter name matching a parameter in the route template. When more than one route matches an action parameter, any route value is considered `[FromRoute]`.
* **[FromQuery]** is inferred for any other action parameters.

The default inference rules are disabled when the <xref:Microsoft.AspNetCore.Mvc.ApiBehaviorOptions.SuppressInferBindingSourcesForParameters> property is set to `true`. Add the following code in *Startup.ConfigureServices* after `services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_1);`:

[!code-csharp[](../web-api/define-controller/samples/WebApiSample.Api/Startup.cs?name=snippet_ConfigureApiBehaviorOptions&highlight=4)]

### Multipart/form-data request inference

When an action parameter is annotated with the [[FromForm]](xref:Microsoft.AspNetCore.Mvc.FromFormAttribute) attribute, the `multipart/form-data` request content type is inferred.

The default behavior is disabled when the <xref:Microsoft.AspNetCore.Mvc.ApiBehaviorOptions.SuppressConsumesConstraintForFormFileParameters> property is set to `true`. Add the following code in *Startup.ConfigureServices* after `services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_1);`:

[!code-csharp[](../web-api/define-controller/samples/WebApiSample.Api/Startup.cs?name=snippet_ConfigureApiBehaviorOptions&highlight=3)]

### Attribute routing requirement

Attribute routing becomes a requirement. For example:

[!code-csharp[](../web-api/define-controller/samples/WebApiSample.Api/Controllers/ProductsController.cs?name=snippet_ControllerSignature&highlight=1)]

Actions are inaccessible via [conventional routes](xref:mvc/controllers/routing#conventional-routing) defined in <xref:Microsoft.AspNetCore.Builder.MvcApplicationBuilderExtensions.UseMvc*> or by <xref:Microsoft.AspNetCore.Builder.MvcApplicationBuilderExtensions.UseMvcWithDefaultRoute*> in *Startup.Configure*.

::: moniker-end

## Additional resources

* <xref:web-api/action-return-types>
* <xref:web-api/advanced/custom-formatters>
* <xref:web-api/advanced/formatting>
* <xref:tutorials/web-api-help-pages-using-swagger>
* <xref:mvc/controllers/routing>