--- 
name: oauth-logging-in-grails-made-easy-with
layout: post
time: 2012-04-10 23:57:00 +02:00
title: OAuth login in Grails made easy with scribe-java
---
[Grails Spring Security](http://grails.org/plugin/spring-security-core) is a great plugin. It allows you to setup authorization for your app in just a few lines in Grails configuration. So, users can register to your website, click on confirmation link received by email and login with username and password. User experience? Maybe not as friendy as could be :)

So here comes OAuth. As almost everybody on the planet have account on
one of the social networks, why we can ask this network to authorize
user of our application? User logins with his/her credentials to
Facebook/Google+/whatever (called OAuth provider) and gives access
permission to our application. OAuth provider redirects to your website
telling that user has been authorized and giving you some basic
information about the user. No password storing in database of your
application!

OK, so we have grails-spring-security-facebook and twitter plugins. But
what with other providers? Recently, i've released a website for
developers. Do developers have Facebook account? Maybe 50%... Rather
Twitter/Google+/LinkedIn/GitHub(!). All of these are OAuth providers.
But where are the Grails plugins?

After googling a bit i've found
[grails-inviter](http://grails.org/plugin/inviter) plugin by Tomas Lin.
It allows you to import your friends' contacts and send them email
invitation to your website. Underneath the plugin uses
[scribe-java](https://github.com/fernandezpablo85/scribe-java) library -
open source implementation of OAuth 1/2 protocol. So why not use
scribe-java for authentication of users from various social networks in
your application?
Scribe is sort of low-level implementation of OAuth. Here's a list of
it's main elements:

-   `ServiceBuilder` - an abstract factory that lets you build service
    for concrete OAuth provider, giving your app apiKey and apiSecret
-   `Verifier` - OAuth verifier code sent by the OAuth provider which is
    used to create the access token
-   `OAuthRequest` - request to OAuth provider which is signed by the
    access token

You can look at [Getting
Started](https://github.com/fernandezpablo85/scribe-java/wiki/getting-started)
page for more details.

To implement authentication we need to implement following: 

-   With `ServiceBuilder` create and redirect to authorization URL to
    OAuth provider
-   Implement callback action which extracts access token from verifier
    code given by the OAuth provider
-   Create and sign OAuth request to receive user information (\*)
-   Inject user principal into the Spring Security context.

(\*) - you can also extract user data e.g. from Facebook cookie, but
using separate OAuth request is much more readable and compatible with
all providers.

OK, so we would write `AuthController` to handle signin in by social
account and callback from OAuth provider:

{% highlight ruby%}
class AuthController {
 
    SpringSecuritySigninService springSecuritySigninService
 
    def signin = {
        GrailsOAuthService service = resolveService(params.provider)
        if (!service) {
            redirect(url: '/')
        }
 
        session["${params.provider}_originalUrl"] = params.originalUrl
 
        def callbackParams = [provider: params.provider]
        def callback = "${createLink(action: 'callback', absolute: 'true', params: callbackParams)}"
        def authInfo = service.getAuthInfo(callback)
 
        session["${params.provider}_authInfo"] = authInfo
 
        redirect(url: authInfo.authUrl)
    }
 
    def callback = {
        GrailsOAuthService service = resolveService(params.provider)
        if (!service) {
            redirect(url: '/')
        }
 
        AuthInfo authInfo = session["${params.provider}_authInfo"]
 
        def requestToken = authInfo.requestToken
        def accessToken = service.getAccessToken(authInfo.service, params, requestToken)
        session["${params.provider}_authToken"] = accessToken
 
        def profile = service.getProfile(authInfo.service, accessToken)
        session["${params.provider}_profile"] = profile
 
        def uid = profile.uid
        User user = User.findByOauthIdAndOauthProvider(uid, params.provider)
 
        if (user) {
            springSecuritySigninService.signIn(user)
            redirect(uri: (session["${params.provider}_originalUrl"] ?: '/') - request.contextPath)
        } else {
            redirect(controller: 'user', action: 'registerOAuth', params: params)
        }
    }
 
    private def resolveService(provider) {
        def serviceName = "${provider as String}AuthService"
        grailsApplication.mainContext.getBean(serviceName)
    }
 
}
{% endhighlight %}

As you can see - besides from storing some data into session, the main
logic is implemented in `GrailsOAuthService` class - kind of wrapper for
`scribe-java` api. Here is sample implementation for Google+:

{% highlight ruby%}
class GoogleAuthService extends GrailsOAuthService {
 
    @Override
    OAuthService createOAuthService(String callbackUrl) {
        def builder = createServiceBuilder(GoogleApi20,
                grailsApplication.config.auth.google.key as String,
                grailsApplication.config.auth.google.secret as String,
                callbackUrl)
        return builder.grantType(OAuthConstants.AUTHORIZATION_CODE)
               .scope('https://www.googleapis.com/auth/userinfo.profile https://www.googleapis.com/auth/userinfo.email')
               .build()
    }
 
    AuthInfo getAuthInfo(String callbackUrl) {
        OAuthService authService = createOAuthService(callbackUrl)
        new AuthInfo(authUrl: authService.getAuthorizationUrl(null), service: authService)
    }
 
    Token getAccessToken(OAuthService authService, Map params, Token requestToken) {
	Verifier verifier = new Verifier(params.code)
	authService.getAccessToken(requestToken, verifier)
    }
 
}
 
public abstract class GrailsOAuthService {
    static transactional = false
 
    OAuthService oauthService
    def grailsApplication
 
    @Override
    ServiceBuilder createServiceBuilder(Class provider, String apiKey, String secretKey, String callbackUrl) {
        def ServiceBuilder builder = new ServiceBuilder().provider(provider)
                    .apiKey(apiKey).apiSecret(secretKey)
                    .callback(callbackUrl)
        return builder
    }
 
    abstract AuthInfo getAuthInfo(String callbackUrl)
 
    abstract Token getAccessToken(OAuthService authService, Map params, Token requestToken)
 
    abstract OAuthProfile getProfile(OAuthService authService, Token accessToken)
 
}
 
class AuthInfo {
    OAuthService service
    String authUrl
    Token requestToken
}
{% endhighlight %}

Last but not least - we need to extract user data by accessing providers
specific URL with request signed with access token. URLs are listed in
the table below:

<table class="table"><tbody><tr style="background-color: #ddd"><th width="30%">PROVIDER</th><th>URL</th></tr><tr><td>Facebook</td><td>https://graph.facebook.com/me</td></tr><tr><td>Google+</td><td>https://www.googleapis.com/oauth2/v1/userinfo</td></tr><tr><td>Twitter</td><td>http://api.twitter.com/1/account/verify_credentials.json</td></tr><tr><td>LinkedIn</td><td>http://api.linkedin.com/v1/people/~</td></tr><tr><td>GitHub</td><td>https://github.com/api/v2/json/user/show</td></tr></tbody></table>

The structure of returned user profile is also provider specific -
mostly returned by some JSON object. Here's example code for Google+:

{% highlight ruby%}
OAuthProfile getProfile(OAuthService authService, Token accessToken) {
 
        OAuthRequest request = new OAuthRequest(Verb.GET, 'https://www.googleapis.com/oauth2/v1/userinfo')
        authService.signRequest(accessToken, request)
 
        def response = request.send()
 
        def user = JSON.parse(response.body)
        def login = "${user.given_name}.${user.family_name}".toLowerCase()
        new OAuthProfile(username: login, email: user.email, uid: user.id, picture: user.picture)
    }
{% endhighlight %}

The last part is injecting principal object into Spring Security.
Suppose we use standard grails-spring-security model classes to store
user data, this is example code:

{% highlight ruby%}
class SpringSecuritySigninService extends GormUserDetailsService {
 
    void signIn(User user) {
        def authorities = loadAuthorities(user, user.username, true)
        def userDetails = createUserDetails(user, authorities)
        SecurityContextHolder.getContext().setAuthentication(
            new UsernamePasswordAuthenticationToken(userDetails, null, 
                  userDetails.authorities))
    }
 
}
{% endhighlight %}

And we have fully authenticated `UserDetails` object with `@Secured`
annotation compatible authorities.
Enjoy!
