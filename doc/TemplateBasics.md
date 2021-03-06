﻿# Template basics

## Getting Started

Please read the Quick intro first!

RazorEngine provides a base implementation of a template class, the `TemplateBase`, this is normally superseded by the model specific template class, 
the `TemplateBase<T>`. For most use cases, we're hoping this will be enough. 
To get started with template using RazorEngine, you can simply use the static `Engine` type (the `Engine.Razor` instance):

    [lang=csharp]
    string template = "<div>Hello @Model.Name</div>";
    var model = new Person { Name = "Matt" };

    string result = Engine.Razor.RunCompile(template, "key", typeof(Person), model);

Which should result in:

    [lang=markup]
	<div>Hello Matt</div>

The type always needs to be given explicitly, this way you can decide if you want to use a base class or `dynamic` (`null`)
instead of `model.GetType()`.

## Using Anonymous Types
RazorEngine supports anonymous types (those declared as `var` with no identifier, e.g. `var model = { Name = "Matt" };`. 
The set of statements to use an anonymous model, is exactly the same as a statically type model (as seen above):

    [lang=csharp]
    string template = "<div>Hello @Model.Name</div>";
    var model = new { Name = "Matt" };

    string result = Engine.Razor.RunCompile(template, "key", null, model);

Because the generated anonymous type is internal so RazorEngine needs to use some tricks to make this work. 
This is also the reason you can't use `model.GetType()` in this situation.
RazorEngine will wrap your instance of the anonymous class in a wrapper. 
This wrapper ensures that we can access everything, even across AppDomains (`IsolatedRazorEngineService`).

> Note: You can use this wrapper class yourself with `var model = new RazorDynamicObject(myInternalModel)`.
> This wrapper will make sure everything works. You can even use this if you have a model that is NOT serializable which you want to use in
> an isolated sandbox. However this will slighly hit performance.

## Using Dynamic Types
RazorEngine has support for dynamic types (those declared as `dynamic`). Again, the structure for using dynamic types is very similar.
All you need to do is to use `null` for the `modelType` parameter (so the same template will be generated as for anonymous types).

    [lang=csharp]
    string template = "<div>Hello @Model.Name</div>";
    dynamic model = new ExpandoObject();
    model.Name = "Matt";

    string result = Engine.Razor.RunCompile(template, "key", null, (object)model);

> Note: You can run into problems when not casting the model to object.

## Supported synax.

You can access several things when you use the default `TemplateBase<>` implementation:

- `@using Custom.Namespace`
  (see also the quick intro and assembly resolvers for custom references)
- `@model ModelType`
- `@inherits HtmlSupportTemplateBase<ModelType>`
  (see below)
- Set a layout (and `@RenderBody()` within the layout template):

        [lang=markup]
        @{
        	Layout = "layout.cshtml";
        }
	
- `@Include("templateName", model = null, modeType = null)` to include another template.

- Accessing the ViewBag:
	
        [lang=markup]
        <h1>@ViewBag.Title</h1>

- Sections (`@DefineSection`, `@RenderSection` and `@IsSectionDefined`)


## Extending the template Syntax.

As explained in the "About Razor" section the template gets compiled into a method.
This method is part of a class that implements `ITemplate`. 
You are free to provide your own base-class implementations however we recommend to inherit from `TemplateBase<T>` if you need special syntax.

One common feature request is to provide the `@Html.Raw()` (or any other not-Razor specific) syntax, but it is very easy to run that on your own:

    [lang=csharp]
    public class MyHtmlHelper
    {
        public IEncodedString Raw(string rawString)
        {
            return new RawString(rawString);
        }
    }

    public abstract class HtmlSupportTemplateBase<T> : TemplateBase<T>
    {
        public MyClassImplementingTemplateBase()
        {
            Html = new MyHtmlHelper();
        }

        public MyHtmlHelper Html { get; set; }
    }

And then you can use it like:

    [lang=csharp]
	var config = new TemplateServiceConfiguration();
	// You can use the @inherits directive instead (this is the fallback if no @inherits is found).
    config.BaseTemplateType = typeof(HtmlSupportTemplateBase<>);
    using (var service = RazorEngineService.Create(config))
    {
        string template = "@Html.Raw(Model.Data)";
        var model = new { Data = "My raw double quotes appears here \"hello!\"" };

        string result = service.RunCompile(template, "htmlRawTemplate", null, model);
        Console.WriteLine("Template: {0}", template);
        Console.WriteLine("Result: {0}", result);
    }
