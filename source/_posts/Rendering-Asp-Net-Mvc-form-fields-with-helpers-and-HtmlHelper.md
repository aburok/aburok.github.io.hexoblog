---
title: Rendering Asp.Net Mvc form fields with Razor @helper function and HtmlHelper extension method
date: 2016-05-29 14:23:26
tags:
    - Asp.Net MVC
    - Razor
---

# Problem

When you write any kind of form on your page, you often end up with source like this:

``` csharp
@using UmbracoTest.Controllers
@inherits UmbracoViewPage<UmbracoTest.ViewModels.MemberRegisterModel>

@{
    ViewBag.Title = "Membership Registration Before refactor";
}

<h2>Membership Registration</h2>

@using (Html.BeginUmbracoForm<MembershipController>(
    "MemberRegister",
    FormMethod.Post,
    new { @class = "validate-form" }))
{
    <fieldset class="form-group">
        <label>
            @Html.DisplayNameFor(m => m.GivenNames)
        </label>
        @Html.EditorFor(m => m.GivenNames,
            new { htmlAttributes = new { @class = "form-control" } })
        @Html.ValidationMessageFor(m => m.GivenNames,
            string.Empty, new { @class = "error" })
    </fieldset>

    <fieldset class="form-group">
        <label>
            @Html.DisplayNameFor(m => m.FamilyNames) <sup>*</sup>
        </label>
        @Html.EditorFor(m => m.FamilyNames,
            new { htmlAttributes = new { @class = "form-control" } })
        @Html.ValidationMessageFor(m => m.FamilyNames,
            string.Empty, new { @class = "error" })
    </fieldset>

    <fieldset class="form-group">
        <label>
            @Html.DisplayNameFor(m => m.EmailAddress) <sup>*</sup>
        </label>
        @Html.EditorFor(m => m.EmailAddress,
            new { htmlAttributes = new { @class = "form-control" } })
        @Html.ValidationMessageFor(m => m.EmailAddress,
            string.Empty, new { @class = "error" })
    </fieldset>

    <fieldset class="form-group">
        <label>
            @Html.DisplayNameFor(m => m.EmailAddressConfirmation) <sup>*</sup>
        </label>
        @Html.EditorFor(m => m.EmailAddressConfirmation,
            new { htmlAttributes = new { @class = "form-control" } })
        @Html.ValidationMessageFor(m => m.EmailAddressConfirmation,
            string.Empty, new { @class = "error" })
    </fieldset>
}
```

As we can see, the code repeats itself. The code in `<label>` element is almost the same for all fields in this form.

``` csharp
 <fieldset class="form-group">
    <label>
        @Html.DisplayNameFor(m => m.FamilyNames) <sup>*</sup>
    </label>
    @Html.EditorFor(m => m.FamilyNames,
        new { htmlAttributes = new { @class = "form-control" } })
    @Html.ValidationMessageFor(m => m.FamilyNames,
        string.Empty, new { @class = "error" })
</fieldset>
```

This is code smell because in the name of DRY rule, we should avoid code repeatition.

Only difference is the name of the field that we want to render. This name is taken from the expression passed to each of the functions:
+ `@Html.DisplayNameFor`
+ `@Html.EditorFor`
+ `@Html.ValidationMessageFor`

Additionaly, the `*` sign in most of the fileds is added manually.
This is code smell, because we have this information already in the model.

This view has a view model that is connected to it.

``` csharp
 public class MemberRegisterModel : RenderModel, IValidatableObject
{
    public MemberRegisterModel()
        : base(UmbracoContext.Current.PublishedContentRequest.PublishedContent)
    { }

    [DisplayName("First Name")]
    public virtual string GivenNames { get; set; }

    [DisplayName("Last Name")]
    [Required]
    public virtual string FamilyNames { get; set; }

    [DisplayName("Email")]
    [Required]
    [DataType(DataType.EmailAddress)]
    public string EmailAddress { get; set; }

    [DisplayName("Email Confirmation")]
    [Required]
    [DataType(DataType.EmailAddress)]
    public string EmailAddressConfirmation { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        throw new System.NotImplementedException();
    }
}
```

# Solution

The [`RequiredAttribute`][2] is telling the validator that this property is required.
We could use that information to automatically render the required indicator -> `*`.

To check if property should be marked with `*` we need to check if the propert has the `RequiredAttribute`.
Below code checks if given property has a attribute of type.

``` csharp
using System;
using System.Linq;
using System.Linq.Expressions;
using System.Web.Mvc;

namespace UmbracoTest.Helpers
{
    public static class PropertyExpressionExtensions
    {
        public static bool HasAttribute<TModel, TPropertyType, TAttribute>(
            this Expression<Func<TModel, TPropertyType>> propertyExpression,
            Func<TAttribute, bool> attributePredicate = null)
            where TAttribute : Attribute
        {
            var expressionText = ExpressionHelper.GetExpressionText(propertyExpression);
            var type = typeof(TModel);
            var property = type.GetProperty(expressionText);

            var attributeType = typeof(TAttribute);

            var attributes = (TAttribute[])property.GetCustomAttributes(attributeType, true);

            if (attributes.Any() == false)
                return false;

            if (attributePredicate != null && attributes.Any(attributePredicate) == false)
                return false;

            return true;
        }
    }
}
```

Here is the code for checking if property is marked with `RequiredAttribute` :

```csharp
public static bool IsFieldRequired<TModel, TPropertyType>(
    this Expression<Func<TModel, TPropertyType>> propertyExpression)
{
    var hasRequiredAttribute =
        propertyExpression.HasAttribute<TModel, TPropertyType, RequiredAttribute>();
    return hasRequiredAttribute;
}
```

Shared template that is used to render field in Razor friendly way. This helper is stored in `App_Code` folder.

```razor
@using System.Web.Mvc.Html

@helper RenderFormFieldHelper(
    System.Web.Mvc.HtmlHelper htmlHelper,
    string fieldName,
    bool isRequired)
{
    <fieldset class="form-group">
        <label>
            @htmlHelper.DisplayName(fieldName)
            @if (isRequired)
            {
                <sup>*</sup>
            }
        </label>
        @htmlHelper.Editor(fieldName, new { htmlAttributes = new { @class = "form-control" } })
        @htmlHelper.ValidationMessage(fieldName, string.Empty, new { @class = "error" })
    </fieldset>
}
```

Following code is a litlle bit tricky.

```csharp
public static class HtmlHelperExtensions
{
    public static HelperResult FormFieldFor<TModel, TResult>(
        this HtmlHelper<TModel> html,
        Expression<Func<TModel, TResult>> expression,
        Func<HtmlHelper, string, bool, HelperResult> template)
        where TModel : class
    {
        var isRequired = expression.IsFieldRequired();
        var propertyName = ExpressionHelper.GetExpressionText(expression);

        var result = template(html, propertyName, isRequired);
        return result;
    }
}
```

It's extension to HtmlHelper class. Parameters passed to this method are folowing:
+ `this HtmlHelper<TModel> html` - this is the HtmlHelper, available in cshtml files,
+ `Expression<Func<TModel, TResult>> expression` - for strongly typed models, a better way to pass a property name,
+ `Func<HtmlHelper, string, bool, HelperResult> template` - @helper method that will be used to generate html result

`ExpressionHelper.GetExpressionText(expression)` - is a [method provided by ASP.NET][1] to get the name of the property from an expression.

Working code can be found on [the branch of my umbraco sandbox repository][3].




[1]: https://msdn.microsoft.com/en-us/library/system.web.mvc.expressionhelper.getexpressiontext(v=vs.118).aspx
[2]: https://msdn.microsoft.com/en-us/library/system.componentmodel.dataannotations.requiredattribute(v=vs.110).aspx
[3]: https://github.com/aburok/umbraco-sandbox/tree/form_field_helper

