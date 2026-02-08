---
date: "2025-05-18T13:47:48+02:00"
draft: false
title: "Mocking Request Handling by HTTP Client"
---

There would surely be scenarios in your application where you should use a HTTP Client to request for resources from another server. But how can you mock the behaviour of the default methods of the Http Client provided by the .NET Core Framework while writing unit tests? This post is meant to mock such HTTP request handling for unit testing.

Consider the following code where a service named `SocialMediaPostsService` uses a HTTP Client to request json objects from [Dummy JSON](https://dummyjson.com/docs/posts).

```csharp {filename="SocialMediaPostsService.cs"}
public class SocialMediaPostsService(IHttpClientFactory httpClientFactory)
    : ISocialMediaPostsService
{
    private const string BaseUrl = "https://dummyjson.com";

    private readonly HttpClient _httpClient = httpClientFactory.CreateClient(
        Constants.DUMMY_JSON_CLIENT
    );

    public async Task<IEnumerable<string>?> GetAllTagsForPostAsync(
        int postId,
        CancellationToken cancellationToken
    )
    {
        var response = await _httpClient.GetAsync($"{BaseUrl}/posts", cancellationToken);

        response.EnsureSuccessStatusCode();

        var posts = await response.Content.ReadFromJsonAsync<PostsResponse>(
            ObjectSerializer.GetOptions,
            cancellationToken
        );

        return posts!.Posts.FirstOrDefault(x => x.Id == postId)?.Tags;
    }

    public async Task<Post> AddPostAsync(
        Post post,
        CancellationToken cancellationToken
    )
    {
        var response = await _httpClient.PostAsJsonAsync(
            $"{BaseUrl}/posts/add",
            post,
            ObjectSerializer.GetOptions,
            cancellationToken
        );

        response.EnsureSuccessStatusCode();

        return await response.Content.ReadFromJsonAsync<Post>(
            ObjectSerializer.GetOptions,
            cancellationToken
        ) ?? throw new InvalidOperationException("Failed to deserialize the post.");
    }
}
```

> [!NOTE]
> Using _IHttpClientFactory_ ensures efficient connection management and allows pre-configured named clients for consistent setup and reuse.

Shown below is the _ISocialMediaPostsService_ interface used for dependency injection.

```csharp {filename="ISocialMediaPostsService.cs"}
public interface ISocialMediaPostsService
{
    Task<IEnumerable<string>?> GetAllTagsForPostAsync(
        int postId,
        CancellationToken cancellationToken
    );

    Task<Post> AddPostAsync(
        Post post,
        CancellationToken cancellationToken
    );
}
```

To be able to understand how the client can be mocked, it is essential to understand how the client handles the HTTP request message.

### In Brief

{{% steps %}}

### `HttpClient` inherits from `HttpMessageInvoker`

The `HttpClient` class is built on top of `HttpMessageInvoker`, which is responsible for forwarding HTTP requests.

If you carefully look at the source code for [HttpClient](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Net.Http/src/System/Net/Http/HttpClient.cs), you will notice that the class inherits from [HttpMessageInvoker](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Net.Http/src/System/Net/Http/HttpMessageInvoker.cs). Our Http Client's `GetAsync` method internally invokes a base method in `HttpMessageInvoker` namely:

```csharp{filename="HttpMessageInvoker.cs"}
public virtual Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
```

### `HttpMessageInvoker` holds a reference to `HttpMessageHandler`

When an HTTP request is made, `HttpMessageInvoker` uses the internal [HttpMessageHandler](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Net.Http/src/System/Net/Http/HttpMessageHandler.cs) provided to handle the HTTP request.

### The request is passed to `HttpMessageHandler`

The handler processes the request. This is the main point where the actual HTTP logic is handled. The following is the core method for handling.

```csharp{filename="HttpMessageHandler.cs"}
protected internal virtual HttpResponseMessage Send(HttpRequestMessage request, CancellationToken cancellationToken)
{
    throw new NotSupportedException(SR.Format(SR.net_http_missing_sync_implementation, GetType(), nameof(HttpMessageHandler), nameof(Send)));
}
```

### Custom handlers can override this behavior

The `HttpMessageHandler` class provides the following abstract method to be overriden by an extending class to implement a custom handling of `HttpRequestMessage`.

```csharp{filename="HttpMessageHandler.cs"}
protected internal abstract Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken);

```

By extending `HttpMessageHandler`, we can intercept requests and return controlled responses, which is how mocking works in tests.

{{% /steps %}}

### Our `MockHttpMessageHandler` would look like this

```csharp{filename="MockHttpMessageHandler.cs"}
public class MockHttpMessageHandler : HttpMessageHandler
{
    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken
    )
    {
        return Task.FromResult(MockSendAsync(request, cancellationToken));
    }

    public virtual HttpResponseMessage MockSend(
        HttpRequestMessage request,
        CancellationToken cancellationToken
    )
    {
        throw new NotImplementedException();
    }
}
```

In the above HTTP mock handler we can create our own method `MockSend` to be overwritten when we like to write a test, but for the moment we just throw a `NotImplementedException`. We can then override the abstract method provided by `HttpMessageHandler` to return the result of our own method `MockSend`. This is how we can make sure than when the `HttpMessageHandler's` `SendAsync`method is invoked by the framework, our method `MockSend` is invoked internally.

### `MockHttpMessageHandlerTestBase` class for reusable testing

For the ease of writing tests for classes that uses a `HttpClient`, we can create a resuable class to handle everything related to our HTTP mock handler, like making our handler return a specific object and a staus code, ensure that the mock handler had received a HTTP request message with a particular `Ùri`,`HttpMethod` and `Content`. The test class uses the follwoing library:

- [xUnit](https://xunit.net)
- [NSubstitue](https://nsubstitute.github.io/docs/2010-01-01-getting-started.html)
- [Shouldly](https://docs.shouldly.org)
- [AutoBogus](https://github.com/nickdodd79/AutoBogus)

```csharp{filename="MockHttpMessageHandlerTestBasecs"}
public class MockHttpMessageHandlerTestBase
{
    protected MockHttpMessageHandlerTestBase()
    {
        var httpClient = new HttpClient(HttpMessageHandlerMock)
        {
            BaseAddress = new Uri("http://localhost:5000")
        };
        HttpClientFactory.CreateClient(Arg.Any<string>()).Returns(httpClient);
    }

    private MockHttpMessageHandler HttpMessageHandlerMock { get; } =
        Substitute.ForPartsOf<MockHttpMessageHandler>();

    protected IHttpClientFactory HttpClientFactory { get; } = Substitute.For<IHttpClientFactory>();

    /// <summary>
    ///     Verifies that the mocked HTTP message handler processes a request with the specified HTTP method
    ///     and a request URI containing the given string.
    /// </summary>
    /// <param name="httpMethod">The HTTP method (e.g., GET, POST) to validate against the request.</param>
    /// <param name="requestUri">The string that should be contained in the request URI.</param>
    protected void HttpMockHandlerShouldHandleFor(HttpMethod httpMethod, string requestUri)
    {
        HttpMockHandlerDoes(message =>
        {
            message.Method.ShouldBe(httpMethod);
            message.RequestUri!.AbsoluteUri.ShouldContain(requestUri);
        });
    }

    /// <summary>
    ///     Configures the mocked HTTP message handler to return a response with the specified content and status code
    ///     when any HTTP request is sent.
    /// </summary>
    /// <typeparam name="TResponse">The type of the response content to be serialized as JSON.</typeparam>
    /// <param name="response">The response object to be serialized and returned in the HTTP response body.</param>
    /// <param name="statusCode">The HTTP status code to be set in the response.</param>
    protected void HttpMockHandlerReturns<TResponse>(TResponse response, HttpStatusCode statusCode)
    {
        HttpMessageHandlerMock
            .MockSend(Arg.Any<HttpRequestMessage>(), Arg.Any<CancellationToken>())
            .Returns(x => new HttpResponseMessage(statusCode)
            {
                Content = JsonContent.Create(response)
            });
    }

    /// <summary>
    ///     Verifies that the mocked HTTP message handler receives a request with a JSON body
    ///     matching the specified request object.
    /// </summary>
    /// <typeparam name="TRequest">The type of the request object expected in the HTTP request body.</typeparam>
    /// <param name="requestObject">The expected request object to compare against the deserialized request body.</param>
    protected void HttpMockHandlerShouldReceiveObject<TRequest>(TRequest requestObject)
    {
        HttpMockHandlerDoes(message =>
        {
            Debug.Assert(message.Content != null);
            var content = message.Content.ReadAsStringAsync().Result;
            var receivedObject = JsonSerializer.Deserialize<TRequest>(content);
            receivedObject.ShouldBe(requestObject);
        });
    }

    private void HttpMockHandlerDoes(Action<HttpRequestMessage> callbackAction)
    {
        HttpMessageHandlerMock
            .When(x => x.MockSend(Arg.Any<HttpRequestMessage>(), Arg.Any<CancellationToken>()))
            .Do(x => callbackAction(x.Arg<HttpRequestMessage>()));
    }
}
```

> [!NOTE]
> Note how we create a `HttpClient` with our custom handler `HttpMessageHandlerMock` as the request message handler and carefully mock the `CreateClient` of the `IHttpClientFactory` to return the `HttpClient`.

> [!WARNING]
> The above `MockHttpMessageHandlerTestBase` can be used as test base class if you use the interface `ÌHttpClientFactory` to create the `HttpClient`. Otherwise you need to modify the base class.

### Writing unit tests with `MockHttpMessageHandlerTestBase`

```csharp {filename="SocialMediaPostsServiceTests.cs"}
public class SocialMediaPostsServiceTests : MockHttpMessageHandlerTestBase
{
    private readonly ISocialMediaPostsService _socialMediaPostsServiceMock;

    public SocialMediaPostsServiceTests()
    {
        _socialMediaPostsServiceMock = new SocialMediaPostsService(HttpClientFactory);
    }

    [Theory]
    [InlineData(1)]
    [InlineData(2)]
    public async Task GetAllTagsForPostTestAsync(int postId)
    {
        var posts = GetAllPosts();
        HttpMockHandlerReturns(posts, HttpStatusCode.OK);
        HttpMockHandlerShouldHandleFor(HttpMethod.Get, "dummyjson.com/posts");
        var tags = await _socialMediaPostsServiceMock.GetAllTagsForPostAsync(
            postId,
            CancellationToken.None
        );
        tags.ShouldBeEquivalentTo(posts.Posts.Find(x => x.Id == postId)?.Tags);
    }

    [Fact]
    public async Task AddPostTestAsync()
    {
        var postToSend = new AutoFaker<Post>().UseSeed(42).Generate();
        var postReceived = new AutoFaker<Post>().UseSeed(42).Generate();
        HttpMockHandlerReturns(postReceived, HttpStatusCode.OK);
        HttpMockHandlerShouldHandleFor(HttpMethod.Post, "dummyjson.com/posts/add");
        var post = await _socialMediaPostsServiceMock.AddPostAsync(
            postToSend,
            CancellationToken.None
        );

        post.ShouldNotBeNull();
        post.ShouldBeEquivalentTo(postReceived);
        HttpMockHandlerShouldReceiveObject(postToSend);
    }

    private static PostsResponse GetAllPosts()
    {
        return new PostsResponse
        {
            Posts =
            [
                new Post
                {
                    Id = 1,
                    Tags = new List<string> { "tag1", "tag2", "tag3", "tag4" }
                },
                new Post
                {
                    Id = 2,
                    Tags = new List<string> { "tag5", "tag6" }
                }
            ],
            Total = 2,
            Skip = 0,
            Limit = 10
        };
    }
}
```

In this test, we inject a mock `IHttpClientFactory` into the service. The factory returns a `HttpClient` that uses our custom `MockHttpMessageHandler`. This lets us simulate HTTP responses and verify requests without making real network calls. The service uses this client just like it would in production, but everything is controlled in the test.

You can find the codes in the following GitHub repository.

#### See On Github Repository

{{< cards cols="1" >}}
{{< card link="https://github.com/vicky-sh/BuilderPattern.git" title="Mocking-Request-Handling-by-HTTP-Client" icon="github" >}}
{{< /cards >}}

#### Open in an Online Code Editor

{{< cards cols="1" >}}
{{< card link="https://github.dev/vicky-sh/Mocking-Request-Handling-by-HTTP-Client" title="Mocking-Request-Handling-by-HTTP-Client" icon="github" >}}
{{< /cards >}}
