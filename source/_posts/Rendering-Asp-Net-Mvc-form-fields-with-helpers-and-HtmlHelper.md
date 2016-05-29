---
title: Rendering Asp.Net Mvc form fields with @helpers and HtmlHelper
date: 2016-05-29 14:23:26
tags:
    - Asp.Net MVC
    - Razor
---

# Problem

When you write any kind of form on your page, you often end up with source like this:

``` csharp
@inherits UmbracoViewPage<Application.BusinessLogic.Models.Form.MemberRegisterModel>

@using (Html.BeginUmbracoForm<MembershipSurfaceController>(
    Controllers.MembershipSurface.ActionNames.MemberRegister,
    FormMethod.Post,
    new { @class = "general-form validate-form" }))
{
    <fieldset>
        <label>
            @Html.DisplayNameFor(m => m.GivenNames) <sup>*</sup>
            @Html.EditorFor(m => m.GivenNames)
            @Html.ValidationMessageFor(m => m.GivenNames,
                string.Empty, new { @class = "error" })
        </label>
        <label>
            @Html.DisplayNameFor(m => m.FamilyNames) <sup>*</sup>
            @Html.EditorFor(m => m.FamilyNames)
            @Html.ValidationMessageFor(m => m.FamilyNames,
                string.Empty, new { @class = "error" })
        </label>
        <label>
            @Html.DisplayNameFor(m => m.EmailAddress) <sup>*</sup>
            @Html.EditorFor(m => m.EmailAddress)
            @Html.ValidationMessageFor(m => m.EmailAddress,
                string.Empty, new { @class = "error" })
        </label>
        <label>
            @Html.DisplayNameFor(m => m.EmailAddressConfirmation) <sup>*</sup>
            @Html.EditorFor(m => m.EmailAddressConfirmation)
            @Html.ValidationMessageFor(m => m.EmailAddressConfirmation,
                string.Empty, new { @class = "error" })
        </label>
    </fieldset>
}
```

As we can see, the code repeats itself. The code in `<label>` element is almost the same for all fields in this form.

``` csharp
<label>
    @Html.DisplayNameFor(m => m.GivenNames) <sup>*</sup>
    @Html.EditorFor(m => m.GivenNames)
    @Html.ValidationMessageFor(m => m.GivenNames,
        string.Empty, new { @class = "error" })
</label>
```

This is code smell because in the name of DRY rule, we should avoid code repeatition.

Only difference is the name of the field that we want to render. This name is taken from the expression passed to each of the functions:
+ `@Html.DisplayNameFor`
+ `@Html.EditorFor`
+ `@Html.ValidationMessageFor`

Additionaly, the '*' sign in most of the fileds is added manually.
This is code smell, because we have this information already in the model.

This view has a view model that is connected to it.

``` csharp
public class MemberRegisterModel : RenderModel, IValidatableObject
{
    public MemberRegisterModel()
        : base(UmbracoContext.Current.PublishedContentRequest.PublishedContent)
    { }

    [DisplayName("First Name")]
    [Required]
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
}
```

The `UmbracoRequired` attribute is telling the validator that this property is required.
We could use that information to automatically render the '*'.

``` csharp
using System;
using System.Linq;
using System.Linq.Expressions;
using System.Web.Mvc;

namespace Application.BusinessLogic.Extensions
{
    public static class PropertyExpressionExtensions
    {
        public static bool HasAttribute<T, TAttribute>(
            this Expression<Func<T, string>> propertyExpression,
            Func<TAttribute, bool> attributePredicate = null)
            where TAttribute : Attribute
        {
            var expressionText = ExpressionHelper.GetExpressionText(propertyExpression);
            var type = typeof(T);
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


Here is the code for checking if property has Ubraco required attribute:

```csharp
public static class ExpressionUmbracoHelper
{
    public static bool IsFieldRequired<T>(this Expression<Func<T, string>> propertyExpression)
    {
        var hasRequiredAttribute = propertyExpression.HasAttribute<T, UmbracoRequired>();
        return hasRequiredAttribute;
    }
}
```

Shared template that is used to render field in Razor friendly way. This helper is stored in `App_Code` folder.

```razor
@using System.Web.Mvc.Html

@helper RenderFormFieldHelper(
    System.Web.Mvc.HtmlHelper htmlHelper,
    string fieldName,
    bool isRequired,
    IEnumerable<System.Web.Mvc.SelectListItem> dataSource = null,
    object editorViewData = null)
{
    <label>
        @htmlHelper.DisplayName(fieldName)
        @if (isRequired)
        {
            <sup>*</sup>
        }
        @if (dataSource == null)
        {
            @htmlHelper.Editor(fieldName, editorViewData)
        }
        else
        {
            <div class="select-wrapper">
                @htmlHelper.DropDownList(fieldName, dataSource)
            </div>
        }
        @htmlHelper.ValidationMessage(fieldName, string.Empty, new { @class = "error" })
    </label>
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






[1]: https://msdn.microsoft.com/en-us/library/system.web.mvc.expressionhelper.getexpressiontext(v=vs.118).aspx
