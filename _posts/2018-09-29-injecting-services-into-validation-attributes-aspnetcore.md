---
layout: post
title:  "Injecting services & data into validation attributes with AspNetCore 2.1"
date:   2018-09-09 09:00:00 +0200
categories: dotnetcore aspnetcore mvc dotnet
image:
  feature: header.jpg
---

How to asynchronously provided data to validation attributes in AspNetCore 2.1

TLDR; [source code available on github](https://github.com/RossJayJones/dotnetcore-validation-injection){:target="_blank"}

We came across a situation where we needed some additional data to perform validation on the payload being sent via an HTTP post. Essentially it involved calling up some setup rules from a database and comparing the payload passed in against those rules to ensure the payload was valid. We had the following requirements:

1. Retrieve data for use by validation logic using async/await api
2. Leverage existing validation pipeline. i.e. no custom validation logic in controller
3. Remain transparent to users of the validation attributes

## The problem

Out of the box the System.Component model validation attribute provides a way to access a service locator using the `validationContext.GetService(typeof())` api however this is not an async api. Executing an async operation here would cause us to wait synchronously which was a show stopper.

A great post by [Andrew Lock](https://andrewlock.net/injecting-services-into-validationattributes-in-asp-net-core/){:target="_blank"} got us most of the way there however this method does not allow for asynchronous operations and prevented us meeting the first requirement.

We invesitgated using `Filters` however the filter pipeline is executed after model binding and validation takes place which prevented us from meeting requirement 2 and 3.

We dug into the AspNetCore code base and found that intercepting the model binding step was possible and would give us what we needed.

## Custom Model Binders

We found that we could intercept the model binding by implementing a custom IModelBinder. In AspNetCore, model binding happens asynchronoulsy so this gave us the async hook to go and fetch the additional data required for validation.

Writing a full model binder for complex objects is non-trivial and there was no way I was going to take that on. Instead I figured we could intercept and [proxy](https://en.wikipedia.org/wiki/Proxy_pattern){:target="_blank"} the call to the original model binder. This will require some additional logic to wire it all up.

## Custom Model Binding

### IModelBinder

The custom model binder implementation is straight forward.

{% highlight C# %}
public class CustomValidationModelBinder : IModelBinder
{
  private readonly IModelBinder _underlyingModelBinder;

  public CustomValidationModelBinder(IModelBinder underlyingModelBinder)
  {
    _underlyingModelBinder = underlyingModelBinder;
  }

  public async Task BindModelAsync(ModelBindingContext bindingContext)
  {
    // Perform model binding using original model binder
    await _underlyingModelBinder.BindModelAsync(bindingContext).ConfigureAwait(false);

    // If model binding failed don't continue
    if (bindingContext.Result.Model == null)
    {
        return;
    }

    // Perform some additional work after model binding occurs but before validation is executed.
    // i.e. fetch some additional data to be used by validation
  }
}
{% endhighlight %}

### IModelBinderProvider

We need to tell the Mvc framework how to create an instance of our custom model binder. To do this we need to implement an IModelBinderProvider. This too is straight forward:

{% highlight C# %}
public class CustomValidationModelBinderProvider : IModelBinderProvider
{
  private readonly IModelBinderProvider _underlyingModelBinderProvider;

  public CustomValidationModelBinderProvider(IModelBinderProvider underlyingModelBinderProvider)
  {
    _underlyingModelBinderProvider = underlyingModelBinderProvider;
  }

  public IModelBinder GetBinder(ModelBinderProviderContext context)
  {
    var underlyingModelBinderProvider = _underlyingModelBinderProvider.GetBinder(context);
    return new CustomValidationModelBinder(underlyingModelBinderProvider);
  }
}
{% endhighlight %}

### Hooking it up

To hook this up to the Mvc framework we can create an extension method to be called by the Startup.cs class.

{% highlight C# %}
public static void UseRiskDataModelBindingProvider(this MvcOptions opts)
{
  var underlyingModelBinder = opts.ModelBinderProviders.FirstOrDefault(x => x.GetType() == typeof(BodyModelBinderProvider));

  if (underlyingModelBinder == null)
  {
      return;
  }

  var index = opts.ModelBinderProviders.IndexOf(underlyingModelBinder);
  opts.ModelBinderProviders.Insert(index, new CustomValidationModelBinderProvider(underlyingModelBinder));
}
{% endhighlight %}

This is called in Startup.cs.

{% highlight C# %}
public void ConfigureServices(IServiceCollection services)
{
  ...
  services.AddMvc(opts => opts.UseRiskDataModelBindingProvider()).SetCompatibilityVersion(CompatibilityVersion.Version_2_1);
  ...
}
{% endhighlight %}

## Providing context

We still need to get the additional data to the validation attributes. To achieve this we make use of the ability to resolve services within the attribute using `validationContext.GetService`. The key is to preload the data (or valiadtion context) asynchrously and provide a way for the attribute to get hold of the validation context synchronously. We can create a [provider](https://en.wikipedia.org/wiki/Provider_model){:target="_blank"} mechanism to achieve this.

For this example, when given a model which contains a name and a list of items, we want to be sure that those values are contained within some pre-defined data stored away in a database or service.

Then given a model which looks like this

{% highlight C# %}
public class ValuesModel
{
  [Required]
  [IsValidName]
  public string Name { get; set; }

  [Required]
  [ContainsValidItems]
  public List<string> Items { get; set; }
}
{% endhighlight %}

We will need data which represents the valid choices for name and items. The context may appear as follows:

{% highlight C# %}
public class CustomValidationContext
{
    public CustomValidationContext(ICollection<string> validNames, 
        ICollection<string> validItems)
    {
        ValidNames = validNames;
        ValidItems = validItems;
    }

    public ICollection<string> ValidNames { get; }

    public ICollection<string> ValidItems { get; }
}
{% endhighlight %}

### Validation Context Provider

The provider is simply a class which provides acess to the validation context instance. It could be implemented as follows.

{% highlight C# %}
public class CustomValidationContextProvider
{
  private CustomValidationContext _context;

  public CustomValidationContext Current
  {
    get
    {
      if (_context == null)
      {
          throw new InvalidOperationException("The custom validation context has not been initialized. Ensure that the CustomValidationModelBinder is being used.");
      }

      return _context;
    }
  }

  internal void Set(CustomValidationContext context)
  {
    if (_context != null)
    {
      throw new InvalidOperationException("Custom validation context has already been set.");
    }

    _context = context;
  }
}
{% endhighlight %}

### Fetching the data

We need a way to fetch the data for our custom validation. This can be done using a DbContext or a service call. For this example we have created a simple validation context factory for brevity. This does nothing but return some sample data using async/await immediately fulfilled task.

{% highlight C# %}
public class CustomValidationContextFactory
{
  public Task<CustomValidationContext> Create()
  {
    return Task.FromResult(new CustomValidationContext(new [] { "a", "b", "c" }, new[] {"1", "2", "3"}));
  }
}
{% endhighlight %}

### Register the services

We need to make these services available to the IoC container. We can do this as follows.

{% highlight C# %}
public static class ServiceCollectionExtensions
{
  public static void AddCustomValidation(this IServiceCollection services)
  {
    services.Add(new ServiceDescriptor(typeof(CustomValidationContextFactory), typeof(CustomValidationContextFactory), ServiceLifetime.Scoped));
    services.Add(new ServiceDescriptor(typeof(CustomValidationContextProvider), typeof(CustomValidationContextProvider), ServiceLifetime.Scoped));
  }
}
{% endhighlight %}

This is called in Startup.cs

{% highlight C# %}
public void ConfigureServices(IServiceCollection services)
{
  ...
  services.AddCustomValidation();
  ...
}
{% endhighlight %}

## Wire up validation logic

So to make the data avaible to the custom validators we need to complete the IModelBinder implementation. Using service location we can get hold of our custom context factory and provider to create an instance of CustomValidationContext and register it with the provider.

>> Since we have access to the original HttpRequest we could use parameters from that request when creating the CustomValidationContext which can be usefull!

The implementation appears as follows:

{% highlight C# %}
public class CustomValidationModelBinder : IModelBinder
{
  private readonly IModelBinder _underlyingModelBinder;

  public CustomValidationModelBinder(IModelBinder underlyingModelBinder)
  {
    _underlyingModelBinder = underlyingModelBinder;
  }

  public async Task BindModelAsync(ModelBindingContext bindingContext)
  {
    await _underlyingModelBinder.BindModelAsync(bindingContext).ConfigureAwait(false);

    // If model binding failed don't continue
    if (bindingContext.Result.Model == null)
    {
        return;
    }

    // Wire up the validation context using async methods
    var customValidationContextFactory = (CustomValidationContextFactory)bindingContext.HttpContext.RequestServices.GetService(typeof(CustomValidationContextFactory));
    var customValidationContextProvider = (CustomValidationContextProvider)bindingContext.HttpContext.RequestServices.GetService(typeof(CustomValidationContextProvider));
    var customValidationContext = await customValidationContextFactory.Create();
    customValidationContextProvider.Set(customValidationContext);
  }
}
{% endhighlight %}

## Custom Validation Attributes

For convenience we can create a base class which is responsible for fishing out the CustomValidationContextProvider to get hold of the CustomValidationContext instance and make it available within the IsValid method of the validation attribute.

{% highlight C# %}
public abstract class CustomValidationBaseAttribute : ValidationAttribute
{
  protected sealed override ValidationResult IsValid(object value, ValidationContext validationContext)
  {
    var customValidationContextProvider = (CustomValidationContextProvider)validationContext.GetService(typeof(CustomValidationContextProvider));

    if (customValidationContextProvider == null)
    {
        throw new InvalidOperationException("The custom validation context provider has not been registered");
    }

    return IsValid(value, customValidationContextProvider.Current, validationContext);
  }

  protected abstract ValidationResult IsValid(object value, CustomValidationContext customValidationContext, ValidationContext validationContext);
}
{% endhighlight %}

### Custom Validators

And finally... we are able to implement our custom validation logic using the additonal validation context to do so.

Given a model and action as follows:

{% highlight C# %}
public class ValuesModel
{
  [Required]
  [IsValidName]
  public string Name { get; set; }

  [Required]
  [ContainsValidItems]
  public List<string> Items { get; set; }
}
{% endhighlight %}

{% highlight C# %}
[HttpPost]
public Task<IActionResult> Post(ValuesModel value)
{
  // Do some stuff with your valid instance of value
}
{% endhighlight %}

We can implement the custom validation attributes `[IsValidName]` and `[ContainsValidItems]` respectively.

{% highlight C# %}
public class IsValidNameAttribute : CustomValidationBaseAttribute
{
  protected override ValidationResult IsValid(object value, CustomValidationContext customValidationContext,
      ValidationContext validationContext)
  {
    var name = value as string;

    if (!string.IsNullOrEmpty(name) && !customValidationContext.ValidNames.Contains(name))
    {
        return new ValidationResult($"{name} is an invalid value. It must be one of {string.Join(", ", customValidationContext.ValidNames)}");
    }

    return ValidationResult.Success;
  }
}
{% endhighlight %}

{% highlight C# %}
public class ContainsValidItemsAttribute : CustomValidationBaseAttribute
{
  protected override ValidationResult IsValid(object value, CustomValidationContext customValidationContext,
    ValidationContext validationContext)
  {
    if (value is ICollection<string> items && items.Any(item => !customValidationContext.ValidItems.Contains(item)))
    {
        return new ValidationResult($"Items contains invalid values. It must be any of {string.Join(", ", customValidationContext.ValidItems)}");
    }

    return ValidationResult.Success;
  }
}
{% endhighlight %}

## Conclusion

We had to jump through a number of hoops to get this right and it feels like it should have been easier. Being required to create the validation context asynchronously is what threw the spanner in the works for us. If know of a simpler solution to this problem I would love to hear from you!

Here is a [gift](https://www.youtube.com/channel/UCcXhhVwCT6_WqjkEniejRJQ){:target="_blank"} for reading all the way to the end.