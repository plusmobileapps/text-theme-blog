---
title: How to use Firebase Authentication with Ktor 2.0
tags: Kotlin Ktor Coroutines Firebase
key: ktor-firebase-auth
---

![](/assets/images/ktor-firebase-authentication.png)

With the release of Ktor 2.0, one of the migrations I had to do was Firebase Authentication which I first learned about from this [medium article](https://levelup.gitconnected.com/how-to-integrate-firebase-authentication-with-ktors-auth-feature-dc2c3893a0cc) last year. Learn how to setup Firebase Authentication with Ktor 2.0 and how to test it. 

<!--more-->

## Setup 

This project was configured using the [Ktor Project Generator template](https://start.ktor.io#/settings?name=ktor-firebase-admin-sample&website=plusmobileapps.com&artifact=com.plusmobileapps.ktor-firebase-admin-sample&kotlinVersion=1.7.10&ktorVersion=2.0.3&buildSystem=GRADLE_KTS&engine=NETTY&configurationIn=CODE&addSampleCode=true&plugins=content-negotiation%2Crouting%2Ckotlinx-serialization%2Cauth). It will setup Ktor with 2.0.3 and the following plugins:

* Authentication
* Content Negotiation
* kotlinx.serialization
* Routing

Then you will also need the [Firebase Admin Java SDK](https://firebase.google.com/docs/admin/setup#add-sdk) in order to user Firebase Authentication. 

```kotlin
implementation("com.google.firebase:firebase-admin:9.0.0")
```

Before starting, follow these [instructions](https://cloud.google.com/firestore/docs/client/get-firebase) to create a new firebase project and enable authentication. Then click on the settings button in the side bar -> project settings -> service accounts tab -> generate a new private key which should then download a JSON file to your machine. 

Rename this file to `ktor-firebase-auth-adminsdk.json` and move it into this project under `src/main/resources/ktor-firebase-auth-adminsdk.json`

## Setup Firebase App

```kotlin
object FirebaseAdmin {
    private val serviceAccount: InputStream? =
        this::class.java.classLoader.getResourceAsStream("ktor-firebase-auth-adminsdk.json")

    private val options: FirebaseOptions = FirebaseOptions.builder()
        .setCredentials(GoogleCredentials.fromStream(serviceAccount))
        .build()

    fun init(): FirebaseApp = FirebaseApp.initializeApp(options)
}
```

## Setup Firebase Authentication

### Create a Principal

```kotlin
data class User(val userId: String = "", val displayName: String = "") : Principal
```

### Create AuthenticationProvider

* [Stackoverflow answer](https://stackoverflow.com/questions/72443764/how-to-use-firebase-admin-with-ktor-2-0)

```kotlin
class FirebaseAuthProvider(config: FirebaseConfig) : AuthenticationProvider(config) {
    val authHeader: (ApplicationCall) -> HttpAuthHeader? = config.authHeader
    private val authFunction = config.firebaseAuthenticationFunction

    override suspend fun onAuthenticate(context: AuthenticationContext) {
        val token = authHeader(context.call)

        if (token == null) {
            context.challenge(
                FirebaseJWTAuthKey,
                AuthenticationFailedCause.InvalidCredentials
            ) { challengeFunc, call ->
                challengeFunc.complete()
                call.respond(UnauthorizedResponse(HttpAuthHeader.bearerAuthChallenge(realm = FIREBASE_AUTH)))
            }
            return
        }

        try {
            val principal = verifyFirebaseIdToken(context.call, token, authFunction)

            if (principal != null) {
                context.principal(principal)
            }
        } catch (cause: Throwable) {
            val message = cause.message ?: cause.javaClass.simpleName
            context.error(FirebaseJWTAuthKey, AuthenticationFailedCause.Error(message))
        }
    }
}

class FirebaseConfig(name: String?) : AuthenticationProvider.Config(name) {
    internal var authHeader: (ApplicationCall) -> HttpAuthHeader? =
        { call -> call.request.parseAuthorizationHeaderOrNull() }


    var firebaseAuthenticationFunction: AuthenticationFunction<FirebaseToken> = {
        throw NotImplementedError(FirebaseImplementationError)
    }

    fun validate(validate: suspend ApplicationCall.(FirebaseToken) -> User?) {
        firebaseAuthenticationFunction = validate
    }
}

public fun AuthenticationConfig.firebase(
    name: String? = FIREBASE_AUTH,
    configure: FirebaseConfig.() -> Unit
) {
    val provider = FirebaseAuthProvider(FirebaseConfig(name).apply(configure))
    register(provider)
}

suspend fun verifyFirebaseIdToken(
    call: ApplicationCall,
    authHeader: HttpAuthHeader,
    tokenData: suspend ApplicationCall.(FirebaseToken) -> Principal?
): Principal? {
    val token: FirebaseToken = try {
        if (authHeader.authScheme == "Bearer" && authHeader is HttpAuthHeader.Single) {
            withContext(Dispatchers.IO) {
                FirebaseAuth.getInstance().verifyIdToken(authHeader.blob)
            }
        } else {
            null
        }
    } catch (ex: Exception) {
        ex.printStackTrace()
        return null
    } ?: return null
    return tokenData(call, token)
}

fun HttpAuthHeader.Companion.bearerAuthChallenge(realm: String): HttpAuthHeader {
    return HttpAuthHeader.Parameterized("Bearer", mapOf(HttpAuthHeader.Parameters.Realm to realm))
}

fun ApplicationRequest.parseAuthorizationHeaderOrNull() = try {
    parseAuthorizationHeader()
} catch (ex: IllegalArgumentException) {
    println("failed to parse token")
    null
}

const val FIREBASE_AUTH = "FIREBASE_AUTH"
const val FirebaseJWTAuthKey: String = "FirebaseAuth"
private const val FirebaseImplementationError =
    "Firebase  auth validate function is not specified, use firebase { validate { ... } } to fix this"
```

### Install Authentication Plugin

```kotlin
fun Application.configureFirebaseAuth() {
    install(Authentication) {
        firebase {
            validate {
                // TODO look up user profile from DB
                User(it.uid, it.name)
            }
        }
    }
}
```

### Create Authenticated Route

```kotlin
fun Route.authenticatedRoute() {
    authenticate(FIREBASE_AUTH) {
        get("/authenticated") {
            val user: User =
                call.principal() ?: return@get call.respond(HttpStatusCode.Unauthorized)
            call.respond("User is authenticated: $user")
        }
    }
}
```

### Initialize the App

```kotlin
fun main() {
    embeddedServer(Netty, port = 8080, host = "0.0.0.0") {
        FirebaseAdmin.init()
        configureSerialization()
        configureFirebaseAuth()
        configureRouting()
    }.start(wait = true)
}
```

## Testing 

### FirebaseAuthTestProvider

```kotlin
class FirebaseAuthTestProvider(config: FirebaseTestConfig) : AuthenticationProvider(config) {

    private val authFunction: () -> User? = config.mockAuthFunction

    override suspend fun onAuthenticate(context: AuthenticationContext) {
        val mockUser: User? = authFunction()
        if (mockUser != null) {
            context.principal(mockUser)
        } else {
            context.error(
                FirebaseJWTAuthKey,
                AuthenticationFailedCause.Error("User was mocked to be unauthenticated")
            )
        }
    }
}

class FirebaseTestConfig(name: String?) : AuthenticationProvider.Config(name) {

    var mockAuthFunction: () -> User? = { null }

    fun mockAuthentication(mockUser: () -> User?) {
        mockAuthFunction = mockUser
    }

}

fun ApplicationTestBuilder.mockAuthentication(mockAuth: () -> User?) {
    install(Authentication) {
        firebaseTest {
            mockAuthentication { mockAuth() }
        }
    }
}

private fun AuthenticationConfig.firebaseTest(
    name: String? = FIREBASE_AUTH,
    configure: FirebaseTestConfig.() -> Unit
) {
    val provider = FirebaseAuthTestProvider(FirebaseTestConfig(name).apply(configure))
    register(provider)
}
```

### Create a Test

```kotlin
class AuthenticatedRouteTest {

    @Test
    fun `authenticated route - is authenticated`() = testApplication {
        val user = User("some id", "Andrew")
        mockAuthentication { user }
        routing { authenticatedRoute() }

        client.get("/authenticated").apply {
            assertEquals(HttpStatusCode.OK, status)
            assertEquals("User is authenticated: $user", bodyAsText())
        }
    }
}
```

## Resources

* [Github Repository - Source Code](https://github.com/plusmobileapps/ktor-firebase-auth-sample)