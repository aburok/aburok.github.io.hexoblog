---
title: >-
  Validating ASP.NET MVC form in razor with AJAX using RemoteAttribute in
  Umbraco application
tags:
  - ASP.NET MVC
  - Umbraco
  - jQuery
  - Razor
date: 2016-05-31 23:07:15
---


# Challenge
My task was to add client side validation of password field to existing form.
Validation on server side was already implemented.
Rewriting the password validation algorithm on client doesn't made sense.
Only question was how to reuse this logic.

I've used the jQuery Unobtrusive Validation library provided for ASP.NET MVC.
MVC comes with `System.Web.Mvc.RemoteAttribute` class.
This class needs to be put on property that you want to remotely validate.

```csharp
[Required]
[Remote("ValidatePassword",
    "Validation",
    HttpMethod = "POST",
    AdditionalFields = "EmailAddress, UserName")]
[DataType(DataType.Password)]
public string Password { get; set; }
```

The `RemoteAttribute` has couple of parameters to pass :
+ `ValidatePassword` - name of an action inside a ASP.NET MVC Controller that will be used to validate,
+ `Validation` - name prefix of controller (full name: `ValidationController`), that contains validation action,
+ `HttpMethod = "POST"` - http method used to communicate with server, with `POST` method, password will not be exposed in url,
+ `AdditionalFields = "EmailAddress, UserName"` - fields that we want to send along with the password to validation action.

ValidationController class :

```csharp
[OutputCache(Location = OutputCacheLocation.None, NoStore = true)]
public class ValidationController : SurfaceController
{
    [System.Web.Http.HttpPost]
    public JsonResult ValidatePassword(ValidatePasswordRequest request)
    {
        if (string.IsNullOrWhiteSpace(request.EmailAddress)
            || string.IsNullOrWhiteSpace(request.UserName)
            || string.IsNullOrWhiteSpace(request.Password))
        {
            return Json(true);
        }

        var validationResults = PasswordValidator.Validate(
            request.Password,
            request.UserName,
            request.EmailAddress);

        // if no errors were returned then return true
        if (validationResults == PasswordMessage.Valid)
            return Json(true);

        return Json(validationResults.ToString());
    }
}
```

`ValidatePassword` action method is marked with `HttpPost` attribute.
This will tell the server to only accept http `POST` requests.

Method takes as a parameter object of following class:

```csharp
public class ValidatePasswordRequest
{
    public string Password { get; set; }
    public string UserName { get; set; }
    public string EmailAddress { get; set; }
}
```

Name of properties in the `ValidatePasswordRequest` class must match names of view model that is used to render the form.

Following class is being used to render form:

```csharp
public class RegisterUserViewModel
{
    [DisplayName("User name")]
    [Required]
    public string UserName { get; set; }

    [DisplayName("Email address")]
    [Required]
    [DataType(DataType.EmailAddress)]
    [EmailAddress]
    public string EmailAddress { get; set; }

    [Required]
    [Remote("ValidatePassword",
        "Validation",
        HttpMethod = "POST",
        AdditionalFields = "EmailAddress, UserName")]
    [DataType(DataType.Password)]
    public string Password { get; set; }

    [DisplayName("Password confirmation")]
    [Required]
    [DataType(DataType.Password)]
    public string PasswordConfirmation { get; set; }
}
```

In razor we have just a standard form:

```razor

@using (Html.BeginUmbracoForm<RegisterUserController>(
    "MemberRegister", FormMethod.Post, new { @class = "validate-form" }))
{
    ...

    <fieldset class="form-group">
        <label>
            @Html.DisplayNameFor(m => m.Password) <sup>*</sup>
        </label>
        @Html.EditorFor(m => m.Password,
            new { htmlAttributes = new { @class = "form-control" }})
        @Html.ValidationMessageFor(m => m.Password,
            string.Empty, new { @class = "error" })
    </fieldset>

    ...
}
```

We need to also turn on the jQuery unobtrusive validation in web.config:

```xml
<configuration>
    ...
    <appSettings>
        ...
        <add key="ClientValidationEnabled" value="true"/>
        <add key="UnobtrusiveJavaScriptEnabled" value="true"/>
```

Additionaly we need to include following scripts in our page, at the end of html body element:

```html
    ...
    <!-- Javascripts -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/2.2.4/jquery.js"></script>
    <script src="http://ajax.aspnetcdn.com/ajax/jquery.validate/1.15.0/jquery.validate.js"></script>
    <script src="http://ajax.aspnetcdn.com/ajax/mvc/5.2.3/jquery.validate.unobtrusive.js"></script>
</body>
</html>
```

Full working code can be found on a [branch of my Umbraco sandbox repo][1].


[1]:https://github.com/aburok/umbraco-demo/tree/post/jquery-remote-validation
