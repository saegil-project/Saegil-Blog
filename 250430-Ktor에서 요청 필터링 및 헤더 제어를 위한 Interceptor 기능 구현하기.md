# Ktor에서 요청 필터링 및 헤더 제어를 위한 Interceptor 기능 구현하기
## 문제 정의
현재 서버로의 요청을 보낼 때 헤더를 붙일 때마다 일잃이 요청에 추가하는 방식으로 구현되어 있었고   
일일이 추가하는 방식이 비효율적이라 판단.   

`Retrofit`에서 사용하는 `Interceptor`와 같이 요청을 보내기 전에 가로채어 url에 따라 헤더를 선택적으로 붙이는 기능을 구현하고자 함

## 시도한 방법
### 1. 확장함수
```kotlin
install(DefaultRequest) {
    contentType(ContentType.Application.Json)
}
```
일단 모든 요청은 Json으로 처리되므로 `HttpClient`에 위와 같이 `DefaultRequest`로 `contentType`을 Json으로 처리하도록 설정하였다.

이후에는 헤더를 갖고 있는 Requset가 있고 헤더를 갖고 있지 않은 Request가 있어

```kotlin
suspend fun HttpClient.oAuthPost(
    url: String,
    accessToken: String,
    body: Any
): HttpResponse = post(url) {
    header(HttpHeaders.Authorization, accessToken)
    setBody(body)
}
```
위와 같이 확장함수로 처리하였다.

하지만 이는 일일이 헤더를 추가하는 것과 별반 다를바 없는 것으로 Retrofit에서 사용하는 Interceptor와 동일한 기능을 수행한다고 보기 어렵다.

### 2. intercept

방법을 찾던 중 [Ktor 공식문서](https://ktor.io/docs/client-http-send.html)에서 intercepting 에 대한 내용을 찾았다.

```kotlin
val client = HttpClient(Apache)
client.plugin(HttpSend).intercept { request ->
    val originalCall = execute(request)
    if (originalCall.response.status.value !in 100..399) {
        execute(request)
    } else {
        originalCall
    }
}
```
위와 같은 방식으로 intercept를 하는데 문제는 이미 만들어진 인스턴스에 붙이는 방식이다.   
이것은 `HttpClient`를 리턴하는 것이 아닌 `client`의 메소드로 이것을 모듈에서 생성하고 리턴하는 것은 불가능하다.   
이것을 사용하는 `OAuthServiceImpl`에서 붙여줄 경우 이건 일일이 헤더를 붙이는 것과 다를 바가 없어진다.   
그렇기에 일단 다른 방법을 찾아보아야할 듯 하다.

### 3. DefaultRequest에서 처리

```kotlin
private val EXCEPT_LIST = listOf(
        "/oauths/reissue/token",
        "/oauths/kakao"
    )

    @Provides
    @Singleton
    fun provideHttpClient(
        tokenDataSource: TokenDataSource
    ): HttpClient {
        return HttpClient(Android) {
            install(DefaultRequest) {
                val token = runBlocking { tokenDataSource.getAccessToken() }
                if (!EXCEPT_LIST.contains(encodedPath)) {
                    headers.append("Authorization", "Bearer $token")
                }
            }
        }
    }
```

`DefaultRequest`는 기본 요청에 대한 처리를 자동으로 해주므로 이런 방식으로 구현해보려 했으나 
이유를 알 수 없는 값이 제대로 받아지지 않는 문제가 발생했다.   
![img](https://github.com/user-attachments/assets/fb9d62dd-f21b-4e6b-85ab-fead7c2a3042)

찾아보니 `DefaultRequest`는 정말 기본세팅으로 이 시점에서는 url 정보가 완전히 반영되기 전이다.
그렇기에 이떄 사용하는 url은 항상 초기값(빈값 또는 기본값)일 가능성이 크고 실제로도 빈 값이 나왔다.


즉 `DefaultRequest`을 사용하는 것은 할 수 없다.   
그렇기에 조건은 이렇다.   
1. **url 정보가 완전히 반영된 시점일 것**
2. **사용처마다 일일이 인스턴스를 다루지 않을 것**

## 최종 해결 방법
### CustomPlugin
공식문서의 `intercept`를 사용해야할 것 같아 방법을 찾아보던 중 `install`을 하는 plugin을 커스텀할 수 있다는 것을 알게되었다.   
그래서 커스텀 플러그인을 만들고 거기서 `intercept`를 수행한 뒤 최종 요청을 처리하도록 구현해줌으로서 `Intercept`의 효과를 기대해본다.

```kotlin
class ConditionalAuthPlugin(
    private val tokenProvider: suspend () -> String,
    private val shouldAttach: (HttpRequestBuilder) -> Boolean
) {

    class Config {
        lateinit var tokenProvider: suspend () -> String
        var shouldAttach: (HttpRequestBuilder) -> Boolean = { false }

        fun build() = ConditionalAuthPlugin(tokenProvider, shouldAttach)
    }

    companion object : HttpClientPlugin<Config, ConditionalAuthPlugin> {
        override val key = AttributeKey<ConditionalAuthPlugin>("ConditionalAuthPlugin")

        override fun prepare(block: Config.() -> Unit): ConditionalAuthPlugin {
            return Config().apply(block).build()
        }

        override fun install(plugin: ConditionalAuthPlugin, scope: HttpClient) {
            scope.requestPipeline.intercept(HttpRequestPipeline.State) {
                if (plugin.shouldAttach(context)) { 
                    val token = plugin.tokenProvider()
                    context.headers.append(HttpHeaders.Authorization, token)
                }
                proceed()
            }
        }
    }
}
```
```kotlin
install(ConditionalAuthPlugin) {
    tokenProvider = { tokenDataSource.getToken().accessToken }
    shouldAttach = { request ->
        val path = request.url.encodedPath
        path in setOf(
            경로
        )
    }
}
```
위의 코드를 보면 `HttpClient`인스턴스를 생성할 때 플러그인을 `install`하는데 이때의 람다가 `prepare`의 `block`이다.

실제 `install`이 되기 전에 람다 내부를 통해 `Config().apply(block).build()` 로 `Config`의 `build`를 통해 `ConditionalAuthPlugin`인스턴스를 생성된다.   
그것이 `install`의 파라미터 `plugin`으로 사용된다.

`scope`는 모듈에서 생성한 `HttpClient` 자체이다.    
여기서 왜 `DefaultRequest`에서 url이 제대로 나오지 않았던 이유가 나온다.   

이 커스텀플러그인인 `ConditionalAuthPlugin`은 `scope.requestPipeline.intercept(HttpRequestPipeline.State)`로 intercept가 `HttpRequestPipeline.State`이 시점에 발생한다.   
`HttpRequestPipeline.State`은 **요청 URL 등 정보가 완전히 확정된 후 동작**한다.   

하지만 `DefaultRequest`는`HttpRequestPipeline.Before`로 **요청 빌더가 설정되기 전에 동작**한다.   
이것이 왜 `DefaultRequest`에서는 url에 따른 분기처리를 할수 없는 이유이다.

## 결론
최종적으로 `shouldAttach`로 헤더를 붙일 지 결정후 `proceed`로 최종 요청을 처리함으로서 `Interceptor`의 역할을 수행하는 커스텀플러그인을 구현할 수 있다.
