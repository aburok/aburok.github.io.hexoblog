---
title: Rendering Asp.Net Mvc form fields with @helpers and HtmlHelper
date: 2016-05-19 14:23:26
tags:
    - Asp.Net MVC
    - Razor
---

# Problem

When you write any kind of form on your page, you often end up with source like this:

```
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
```
<label>
    @Html.DisplayNameFor(m => m.GivenNames) <sup>*</sup>
    @Html.EditorFor(m => m.GivenNames)
    @Html.ValidationMessageFor(m => m.GivenNames,
        string.Empty, new { @class = "error" })
</label>
```

Only difference is the name of the field that we want to render. This name is taken from the expression passed to each of the functions:
+ `@Html.DisplayNameFor`
+ `@Html.EditorFor`
+ `@Html.ValidationMessageFor`

Additionaly, the '*' sign in most of the fileds is added manually.
This is code smell, because we have this information already in the model.

This view has a view model that is connected to it.

```
namespace Application.BusinessLogic.Models.Form
{
    public class MemberRegisterModel : RenderModel, IValidatableObject
    {
        public MemberRegisterModel()
            : base(UmbracoContext.Current.PublishedContentRequest.PublishedContent)
        { }

        [UmbracoDisplayName("SmartSaver.Form.FirstName")]
        [UmbracoRequired(ErrorMessageDictionaryKey = "SmartSaver.Form.Required")]
        public virtual string GivenNames { get; set; }

        [UmbracoDisplayName("SmartSaver.Form.LastName")]
        [UmbracoRequired(ErrorMessageDictionaryKey = "SmartSaver.Form.Required")]
        public virtual string FamilyNames { get; set; }

        [UmbracoDisplayName("SmartSaver.Form.Email")]
        [UmbracoRequired(ErrorMessageDictionaryKey = "SmartSaver.Form.Required")]
        [UmbracoEmail(ErrorMessageDictionaryKey = "SmartSaver.Form.EmailWrongFormat")]
        [DataType(DataType.EmailAddress)]
        public string EmailAddress { get; set; }

        [UmbracoDisplayName("SmartSaver.Form.EmailConfirmation")]
        [UmbracoRequired(ErrorMessageDictionaryKey = "SmartSaver.Form.Required")]
        [UmbracoEmail(ErrorMessageDictionaryKey = "SmartSaver.Form.EmailWrongFormat")]
        [DataType(DataType.EmailAddress)]
        [UmbracoCompare("EmailAddress",
            ErrorMessageDictionaryKey = "SmartSaver.Form.EmailNotMatching")]
        public string EmailAddressConfirmation { get; set; }
    }
}
```

The `UmbracoRequired` attribute is telling the validator that this property is required.
We could use that information to automatically render the '*'.
