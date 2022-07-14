---
layout: post
title: "Building OAuth"
date: 2022-07-07
categories: authentication web-programming 
---

OAuth (**O**pen **A**uthorization) is an open standard for access delegation, commonly used as a way for internet users to grant websites or applications access to their information on other websites without giving them the passwords. It's quite a mouthful, right? That's the definition from [Wikipedia](https://en.wikipedia.org/wiki/OAuth) for the specification behind the _Login With Facebook_, _Login With Google_ and _Login with [Fill In Popular Site]_ buttons we see when we want to register or login to our various accounts on the web.  

This post tries to explain the workings behind the spec. The post is for those that are interested in the implementation of the standard. If you just want to include the different login/register buttons on your site, I'd advise that you use the relevant libraries from the identity providers you wish to use.  

There will be two parts in this article. The first part will introduce OAuth and a browser based implementation, the second will discuss the terminal app based implementation.  

The purpose of OAuth is about getting access tokens. The tokens are used in subsequent interactions between the client app/website and the resource. There are four ways of authorizing an application. The four ways are:

* Authorization Code: This is the most widely implemented type in the sense that what you know as OAuth is this kind. It involves four parties (user, browser, authorization server and application). We will be discussing this type in this post.

* Implicit Grant: This is a simpler version of the **Authorization Code** type. It is implemented by apps without the ability to store sensitive data on a server.  More info of this type can be found [here](https://datatracker.ietf.org/doc/html/rfc6749#section-4.2).

* Password Credentials: This is when the user's credentials is passed from the app to the authorization server to obtain an access token. This type is used when there is a high degree of trust between user and the app. More info of this type can be found in the [spec](https://datatracker.ietf.org/doc/html/rfc6749#section-4.3).

* Client Credentials: The client credentials is used as an authorization grant to obtain an access token. This kind does not need the involvement of the user. More info of this type can be found [here](https://datatracker.ietf.org/doc/html/rfc6749#section-4.4).

Before we talk about OAuth of the Authorization Code kind, we need to understand what the parties/roles are.  

### OAuth Parties/Roles
* Resource Owner: The user/entity that owns the resources and is capable of granting access to it. For the purpose of this article, resource owner will be called user.

* Client: The application that requires access to the resource. Once authorization is granted by the user, the application makes resource requests on behalf of the user.

* Authorization Server: This server handles authentication and authorization on behalf of the **resource server**. It handles the authentication of the user via the authentication endpoint. Once that is successful, authentication of the **client** is done. If this is also successful, an access token is then issued to the **client**. 

* Resource Server: The server protects the resource on behalf of the user. Once authorization has been granted by the the **authorization server**, the resource server accepts and responds to protected resource requests by the client.  

### How It Works
<image src="/assets/images/oauth/auth_code_flow.png">

_Illustration showing the OAuth 2.0 flow, from [Digital Ocean](https://www.digitalocean.com/community/tutorials/an-introduction-to-oauth-2)

#### Client Registration
Before a client can be involved in the OAuth flow, it needs to be registered with the **Authorization Server**. This registration is done for security reasons to ensure the client is verified. The client name and website will be required as information for registration. Some OAuth service will also request for the Redirect/Callback URL.  

On successful registration of the client, the server will generate credentials for it. This credentials will be the client identifier and client secret. This credentials will be used for authenticating the client during the Client Authentication stage.

Note that this stage isn't a part of the OAuth flow, so some service might not require client registration to use its OAuth service.

#### User Authorization
The client provides a link to the user with a url whose structure looks like  
    `https://authorization_server_domain/authorize_path?client_id=CLIENT_ID&scope=scopes&state=RANDOM_STRING&redirect_uri=https://client_domain/callback_path`  

The `authorization_server_domain` is the OAuth provider's domain name e.g `github.com` is Github's domain name.  
The `authorize_path` is the path to the user's authorization endpoint e.g `/login/oauth/authorize` is Github's user authorization endpoint's path.  
The `client_id` is the client's identifier. It is commonly generated when the client/app is registered with the OAuth provider. It is usually required.  
The `scope` specifies the level of authorization that the client will have on authorization. It is commonly expressed as a list of space-delimited, case-sensitive string. `read:user repo` is an example of scopes as used in Github. It is recommended that the user authorization url is encoded mainly due to the space delimition of the scope parameter.  
The `state` is a random string generated by the client to maintain state between this step and the callback step. It's main role is to prevent cross-site forgery. The spec goes into [detail](https://datatracker.ietf.org/doc/html/rfc6749#section-10.12) concerning the reasoning behind this. It is recommended for many OAuth providers.  
The `redirect_uri` is where the user's browser/app is redirected to when the user has successfully granted the client the authorization. Some providers require this on registration, which makes this an optional parameter in their authorization url.  

After the user clicks the link, an http request is made to the **authorization server**. The server validates the `client_id` and the `redirect_uri`. If these aren't valid, an error response is sent to the referrer and the flow stops. A login page is then displayed for an unauthenticated user. If the user logs in or is authenticated, an alert is displayed for the user to explicitly grant authorization to the client.  

A simple Django OAuth server with the following OAuth models

```python
    class OAuth(models.Model):
        url = models.URLField(max_length=150, null=False, blank=False, unique=True)
        client_id = models.CharField(max_length=150, null=False, unique=True)
        client_secret = models.CharField(max_length=128, null=False)
        created_at = models.DateTimeField(auto_now_add=True)


    class Grant(models.Model):
        client = models.ForeignKey('server.OAuth', on_delete=models.CASCADE)
        user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
        code = models.CharField(max_length=150)
        expiry = models.DateTimeField()
        created_at = models.DateTimeField(auto_now_add=True)
```

A simple implementation of the `client_id` and `redirect_uri` validation is shown below

```python
    @login_required
    def authorize_request(request):
        client_id = request.GET.get('client_id')
        '''
            Scope isn't used in this example because I didn't integrate permissions. 
            Integrating permissions in the server is needed for scopes.
        '''
        _ = request.GET.get('scope')
        state = request.GET.get('state')
        redirect_uri = request.GET.get('redirect_uri')

        if not client_id or not state or not redirect_uri:
            raise Http404("Page does not exist")

        try:
            client = OAuth.objects.get(client_id=client_id, url=redirect_uri)
        except OAuth.DoesNotExist:
            raise Http404("Page does not exist")

        # Authorization code generation section. This part will be shown in the next stage
        .....

        form = AuthorizationForm()
        return render(request, 'user/authorize.html', context={'authorize_form': form, 'client': client.url})
```

The next stage is executed when the user grants authorization.

#### Authorization Server Sends Authorization Code
The server generates an _authorization code_ and records it alongside the _client_ in its db/cache. It is recommended that this code should have an expiry time to prevent reuse in the future. This code and the `state` (originally sent by the client) are added as parameters to the `redirect_uri` provided by the client (either during the initial request or registration). This redirection uri will be similar to this structure `{redirect_uri}?code={authorization_code}&state={state}`.  
Using the example url shown above, here is an example url  
    `https://example.com/callback?code=RANDOM_STRING&state=RANDOM_STRING`  
The user's browser/app is redirected to the above url.  

Here's a simple implementation of this stage  
```python
    if request.method == "POST":
        form = AuthorizationForm(request.POST)
        if form.is_valid():
            expiry = timezone.now() + datetime.timedelta(seconds=settings.CODE_EXPIRY)
            code = generate_code()
            grant = Grant(user=request.user, client=client, code=code, expiry=expiry)
            grant.save()
            redirect_url = f"{redirect_uri}?code={code}&state={state}"
            return redirect(redirect_url)
        else:
            raise HttpResponseBadRequest(content="Bad content")
```

#### Client Requests Access Token
In the callback endpoint, the client validates the `state` parameter value. It requests for an access token from the authorization server by sending its client credentials (client_id and client_secret) and the `code` parameter value. This request can be in any format supported by the server (json, x-www-form-urlencoded, ...). This request is commonly a POST request.  

Once this request is successful, the client can record this token in the user session.

Here's a simple implementaiton of this validation and request process (Github OAuth)

```python
    def callback(request):
    code = request.GET.get('code')
    state = request.GET.get('state')
    if state != request.session['state']:
        raise Http404('Wrong site')
    data = {
        'client_id': settings.CLIENT_ID,
        'client_secret': settings.CLIENT_SECRET,
        'code': code
    }
    headers = {
        'Accept': 'application/json'
    }
    response = requests.post('https://github.com/login/oauth/access_token', data=data, headers=headers)

    try:
        request.session['access_token'] = response.json()['access_token']
    except requests.JSONDecodeError:
        raise Http404('There was an error processing the response')
    except Exception:
        raise Http404('No access token was returned')

    ...
```

For the server, it validates the information sent by the client (client_id, client_secret, code). If this is successful, it generates a cryptographic token called the **access token** that might contain the user and scope information. This token is used by the client to access protected resources. It might also generate another token called the **refresh token**.  

Here's an example implementation of the server side of this stage which generates only access tokens and is unaware of scopes.

```python
    @require_POST
    @csrf_exempt
    def access_token_request(request):
        client_id = request.POST.get('client_id')
        client_secret = request.POST.get('client_secret')
        code = request.POST.get('code')

        try:
            client = OAuth.objects.get(client_id=client_id)
        except OAuth.DoesNotExist:
            return JsonResponse({'message': "Client id or client secret is invalid"})
        
        if not check_password(client_secret, client.client_secret):
            return JsonResponse({'message': "Client id or client secret is invalid"})
        
        try:
            grant = Grant.objects.get(client=client, code=code)
        except Grant.DoesNotExist:
            return JsonResponse({'message': "The code is invalid"})
        
        if grant.expiry < timezone.now():
            return JsonResponse({'message': "The code has expired"})
        
        payload = {
            "exp": datetime.datetime.now(tz=datetime.timezone.utc) + datetime.timedelta(seconds=settings.ACCESS_TOKEN_EXPIRY),
            "user_id": grant.user.pk
        }
        access_token = jwt.encode(payload, settings.SECRET_KEY)
        return JsonResponse({"access_token": access_token})
```
The above implementation uses **JWT** to generate the token. Note that other kinds of tokens can be used in your implementation.  

#### Using The Access Token
The client can now use the access token to carry out requests to the **resource server** as if it's the user. For example, it can use it to get the user's profile information (this is what the sign in buttons you see do), it could carry out actions on the **resource server** on behalf of the user e.t.c.  

An example where we get the user's github profile info using the access token is shown here  

```python
    def answer(request):
        access_token = request.session['access_token']
        headers = {
            'Authorization': f"token {access_token}"
        }
        response = requests.get("https://api.github.com/user", headers=headers)

        try:
            json = response.json()
        except requests.JSONDecodeError:
            raise Http404("There was an error processing the response")

        context = {
            'username': json['login'],
            'id': json['id'],
            'url': json['html_url']
        }
        return render(request, 'client/answer.html', context)
```

### Conclusion
OAuth is a really fascinating topic and one of the most impressive pillars of our modern web. For an important part of the web, it's remarkably simple to understand.  

In the above article and code, I left out PKCE. PKCE guarantees extra security against interception attacks on the authorization code. The [spec](https://datatracker.ietf.org/doc/html/rfc7636) on it is an easy read and covers it well.  
My code on OAuth implementation (server & client) can be found on [Github](https://github.com/goodyduru/django-oauth-experiments).  

I will cover OAuth for terminal based app in Part Two.

### Additonal Resources
* The OAuth [spec](https://datatracker.ietf.org/doc/html/rfc6749) is a recommended read and this [section](https://datatracker.ietf.org/doc/html/rfc6749#section-4.1) on Authorization Code is a compulsory read.
* The PKCE [spec](https://datatracker.ietf.org/doc/html/rfc7636) is also recommended.