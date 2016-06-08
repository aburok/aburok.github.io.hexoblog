---
title: Setting correct culture in AJAX request to Umbraco SurfaceController
tags:
  - Umbraco
  - AJAX request
date: 2016-06-01 00:00:00
---


# Problem

Project that I'm working on contains a form that is being validated.
My latest task was to add remote validation for the password field to check it against existing server code.
I've described this in [my previous post][1].

There was one problem which I did not described in that post.
This site has multiple languages and we are using Umbraco dictionary to manage translations for texts on our page.
One of those texts are validation messages for form.

Key to this dictionary item is set via attribute that is marking property.

```csharp
[UmbracoDisplayName("MyProject.Form.Password")]
[UmbracoRequired(ErrorMessageDictionaryKey = "MyProject.Form.Required")]
[DataType(DataType.Password)]
[Remote("ValidatePassword", "Validation", HttpMethod = "POST", AdditionalFields = "EmailAddress")]
public string Password { get; set; }
```

The `MyProject.Form.Password` is an entry in Umbraco dictionary that was translated into couple of languages.

It worked great with server side validation. After form was submitted, whole Umbraco page was send to server. Server had full context for a page.
Proper culture was correctly set based on the Umbraco page context. So validation message was displayed in proper language.

Problem appeared when I've started to use the `RemoteAttribute`.
This way I was calling server validation method with an AJAX request. In this situation, Umbraco could not correctly read the language from the request.

The method I found in the existing solution was to get current culture from the url and mapping in Umbraco.

Following algorithm was used to detect current culture.

```csharp
public class UmbracoService : IUmbracoService
{
    private readonly IDomainService _domainService;

    public UmbracoService(IDomainService domainService)
    {
        _domainService = domainService;
    }

    // TODO : poor man ioc
    public UmbracoService()
        : this(UmbracoContext.Current.Application.Services.DomainService)
    { }

    protected HttpRequest Request => HttpContext.Current.Request;

    public string GetCultureCode()
    {
        //Get language id from the Root Node ID
        string requestDomain = Request.ServerVariables["SERVER_NAME"].ToLower();

        var domain = GetMatchedDomain(requestDomain);

        if (domain != null)
        {
            return domain.LanguageIsoCode;
        }

        return "en-US";
    }

    protected string NormalizeUrl(string originalUrl)
    {
        return originalUrl
            .Replace("https://", string.Empty)
            .Replace("http://", string.Empty);
    }

    /// <summary>
    /// Gets domain object from request. Errors if no domain is found.
    /// </summary>
    /// <param name="requestDomain"></param>
    /// <returns></returns>
    public IDomain GetMatchedDomain(string requestDomain)
    {
        var domainList = _domainService
            .GetAll(true)
            .ToList();

        string fullRequest = Request.Url.AbsolutePath.ToLower().Contains("/umbraco/surface")
             ? NormalizeUrl(Request.UrlReferrer.AbsoluteUri)
             : requestDomain + Request.Url.AbsolutePath;

        // walk backwards on full Request until domain found
        string currentTest = fullRequest;
        IDomain matchedDomain = null;
        while (currentTest.Contains("/"))
        {
            matchedDomain = domainList
                .SingleOrDefault(x => x.DomainName == currentTest.TrimEnd('/'));
            if (matchedDomain != null)
            {
                // this is the actual domain
                break;
            }
            if (currentTest == requestDomain)
            {
                // no more to test.
                break;
            }

            currentTest = currentTest.Substring(0, currentTest.LastIndexOf("/"));
            matchedDomain = domainList
                .SingleOrDefault(x => x.DomainName == currentTest);
            if (matchedDomain != null)
            {
                // this is the actual domain
                break;
            }
        }

        return matchedDomain;
    }
}
```

To test it I've created the view. When the page is viewed in English language, both fields display message in English language.
But when I switch to Polish language `demo.umbraco.com/pl`, then first message is displayed in English and the second one in Polish.

I hope that helped you. :)

You can find working version of this code in [my github repo][2].


[1]:http://aburok.github.io/2016/05/31/asp-net-mvc-ajax-form-validation-using-remoteattribute-in-umbraco/
[2]:https://github.com/aburok/umbraco-demo/tree/post/umbraco-culture-ajax-request
