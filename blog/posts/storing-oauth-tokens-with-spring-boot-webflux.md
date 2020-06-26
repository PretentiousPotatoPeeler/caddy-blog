---
title: "Storing Oauth Tokens With Spring Boot Webflux"
date: 2019-09-17T13:45:52+02:00
---

As a pet project, I am trying to recreate Last.fm with Spring Boot. This project involves authenticating the user with OAuth2 to Spotify and storing the authentication / refresh token to later use them to pull the users recent listening history. Sadly, I wanted to get this working in the new Spring WebFlux framework. I say sadly for this was not something that could easily be copy pasted from StackOverflow...

In this blog post I'll walk you through configuring Spring Security to store the tokens. I'll be using Kotlin as it is new and fancy. I'll also be using MongoDB for it has better support for Reactive Programming patterns.

## 1. Project Setup
To setup the initial project I used [Spring Initializr](https://start.spring.io/). Select Kotlin in the language section and add the following **dependencies**:  
1. OAuth2 Client
2. Spring Reactive Web (WebFlux)
3. Spring Data Reactive MongoDB 

Extract the project and open it up in your editor of choice! 

## 2. About Page
We'll first setup a simple about page with the new WebFlux framework. To do this, we add a new Router configuration. This config will add a new routing bean that returns our about page.

```kotlin
@Configuration
class Router {
    @Bean
    fun homeRouter() = router {
        GET("/about") {
            ServerResponse.ok().body(BodyInserters.fromObject("about page"))
        }
    }
}
```
This class uses the WebFlux router. More info on this can be found [here](https://www.baeldung.com/spring-webflux-kotlin).

When we start this project up and navigate to the `/about` page, we'll be greeted with a very interesting "about page".

## 3. Spring Security WebFilter
This is were the fun begins! We will now use the SecurityWebFilterChain to authenticate our "about" page with an OAuth2 login. To do this, we first need to add the filter chain bean. I add this to the main Application file:
```kotlin
@Bean
fun springSecurityFilterChain(http: ServerHttpSecurity): 
      SecurityWebFilterChain {
    http
        .oauth2Login()
        .and()
        .authorizeExchange()
        .anyExchange()
        .authenticated()
    return http.build()
}
```
When you first run this, you'll be greeted by a decent stack trace. You'll need to define the Client Registration Repository. For now, I'll use Spring Boot autoconfigure to setup this bean. This means that I just have to define the following properties:

```bash
spring.security.oauth2.client.registration.spotify.client-name=spotify
spring.security.oauth2.client.registration.spotify.client-id=<client-id>
spring.security.oauth2.client.registration.spotify.client-secret=<secret>
spring.security.oauth2.client.registration.spotify.scope=user-read-recently-played
spring.security.oauth2.client.registration.spotify.authorization-grant-type=authorization_code
spring.security.oauth2.client.registration.spotify.redirect-uri=http://localhost:8080/login/oauth2/code/spotify

spring.security.oauth2.client.provider.spotify.token-uri=https://accounts.spotify.com/api/token
spring.security.oauth2.client.provider.spotify.authorization-uri=https://accounts.spotify.com/authorize
spring.security.oauth2.client.provider.spotify.user-info-uri=https://api.spotify.com/v1/me
spring.security.oauth2.client.provider.spotify.user-name-attribute=id
```
Follow [this link](https://spring.io/guides/tutorials/spring-boot-oauth2/) for more info on this config. When authenticating with Github, Okta, Google or Facebook, this will be easier. Since Spring Boot does not provide the default configuration for Spotify, we need to define all these properties our self.

When you now open the `/about` page, you will be greeted by a Spotify login page. **Nice!**

## 4. Storing The Tokens
So now for the tricky bit! We need to store the authentication and refresh tokens. Spring Security does not provide us with a default method for this so we need to override the default authenticationManager. To do this, we add the following lines to the `SecurityWebFilterChain`:
```diff
@Bean
- fun springSecurityFilterChain(http: ServerHttpSecurity): SecurityWebFilterChain {
+ fun springSecurityFilterChain(http: ServerHttpSecurity, authManager: ReactiveAuthenticationManager): SecurityWebFilterChain {
    http
        .oauth2Login()
+       .authenticationManager(authManager)
        .and()
        .authorizeExchange()
        .anyExchange()
        .authenticated()
    return http.build()
}
```

After that, we add a new bean, `ReactiveAuthenticationManager`:
```kotlin
@Bean
fun authManager(userStorageService: UserStorageService): ReactiveAuthenticationManager {
    val client = WebClientReactiveAuthorizationCodeTokenResponseClient()
    val oauth2UserService = DefaultReactiveOAuth2UserService()
    return OAuth2StoredLoginReactiveAuthenticationManager(client, oauth2UserService, userStorageService)
}
```
The `client`, `oauth2UserService` and `UserStorageService` are default beans. The only thing we changed from the [default implementation](https://github.com/spring-projects/spring-security/blob/635f7e1edd6f6e573ca5342298cc211e8242b58b/config/src/main/java/org/springframework/security/config/web/server/ServerHttpSecurity.java#L1055) (apart from stripping some things we don't need) is the return object.

For this return object, we create a new class. This new class will provide the same functionality as the provided [OAuth2StoredLoginReactiveAuthenticationManager](github), but will also store the user in the database. I implemented this as follows:
```kotlin
class OAuth2StoredLoginReactiveAuthenticationManager(
        accessTokenResponseClient: ReactiveOAuth2AccessTokenResponseClient<OAuth2AuthorizationCodeGrantRequest>,
        private val userService: ReactiveOAuth2UserService<OAuth2UserRequest, OAuth2User>,
        private val userStorageService: UserStorageService) : ReactiveAuthenticationManager {

    private var authorizationCodeManager: ReactiveAuthenticationManager =
            OAuth2AuthorizationCodeReactiveAuthenticationManager(accessTokenResponseClient)

    override fun authenticate(authentication: Authentication): Mono<Authentication> {
        return Mono.defer {
            val token = authentication as OAuth2AuthorizationCodeAuthenticationToken

            this.authorizationCodeManager.authenticate(token)
                    .flatMap { t -> this.onSuccess(t as OAuth2AuthorizationCodeAuthenticationToken) }
        }
    }

    private fun onSuccess(authentication: OAuth2AuthorizationCodeAuthenticationToken):
            Mono<OAuth2LoginAuthenticationToken> {
        val accessToken = authentication.accessToken
        val additionalParameters = authentication.additionalParameters
        val userRequest = OAuth2UserRequest(authentication.clientRegistration, accessToken, additionalParameters)
        return userStorageService.storeUser(authentication)
                .flatMap { _ ->
                    println("User stored!")
                    userService.loadUser(userRequest)
                            .map { oauth2User ->
                                OAuth2LoginAuthenticationToken(
                                        authentication.clientRegistration,
                                        authentication.authorizationExchange,
                                        oauth2User,
                                        oauth2User.authorities,
                                        accessToken,
                                        authentication.refreshToken)
                            }
                }
    }
}
```

And that's it! We should now see a log message 'User stored!' when logging into the about page!
