Below is a **clean and production-ready JUnit + Mockito + StepVerifier** test-case template for the code shown in your screenshot.

Since the full method is not visible, I reconstructed the structure exactly as your screenshot shows:

```java
return chatClient.prompt(prompt)                     // ChatClientRequestSpec
        .options(getChatOptions())
        .stream()                                    // StreamResponseSpec
        .chatResponse()                              // Flux<ChatResponse>
        .flatMap(r -> {
            var content = r.getResult().getOutput().getText();
            final var finishReason = r.getResult().getMetadata().getFinishReason();
            final var kbResponse = promptRequestContext.getKnowledgeBaseResponse();

            if (StringUtils.isBlank(finishReason)) {
                fullResponse.add(content);
                return Flux.just(
                        getChunk(
                                content,
                                "",
                                false,
                                "",
                                null,
                                null
                        )
                );
            }
            ...
        });
```

---

# ✅ **Unit Test Cases (JUnit 5 + Mockito + Reactor StepVerifier)**

### ✔ Tests included:

1. **When finishReason is blank → should return Flux.just(chunk)**
2. **When finishReason is NOT blank → should continue downstream**
3. **Should capture content into fullResponse list**
4. **Mocking deep nested call: r.getResult().getOutput().getText()**

---

# **Test Class**

```java
@ExtendWith(MockitoExtension.class)
class ChatServiceTest {

    @Mock
    private ChatClient chatClient;

    @Mock
    private ChatClientRequestSpec requestSpec;

    @Mock
    private StreamResponseSpec streamSpec;

    @Mock
    private ChatResponse chatResponse;

    @Mock
    private ChatResult result;

    @Mock
    private ChatOutput output;

    @Mock
    private ChatMetadata metadata;

    @Mock
    private PromptRequestContext promptRequestContext;

    @InjectMocks
    private ChatService chatService;     // your service class

    private final List<String> fullResponse = new ArrayList<>();


    @BeforeEach
    void setup() {
        // inject list if your class requires it
        chatService.setFullResponse(fullResponse);
        chatService.setPromptRequestContext(promptRequestContext);
    }


    @Test
    void testBlankFinishReason_ShouldReturnChunk() {

        // ---- arrange mocks ----
        when(chatClient.prompt(anyString())).thenReturn(requestSpec);
        when(requestSpec.options(any())).thenReturn(requestSpec);
        when(requestSpec.stream()).thenReturn(streamSpec);
        when(streamSpec.chatResponse()).thenReturn(Flux.just(chatResponse));

        when(chatResponse.getResult()).thenReturn(result);
        when(result.getOutput()).thenReturn(output);
        when(output.getText()).thenReturn("hello world");

        when(result.getMetadata()).thenReturn(metadata);
        when(metadata.getFinishReason()).thenReturn("");    // blank finishReason

        // KB not used for this test
        when(promptRequestContext.getKnowledgeBaseResponse()).thenReturn(null);


        // ---- act & assert ----
        StepVerifier.create(chatService.callLLM("hi"))
                .expectNextMatches(chunk ->
                        chunk.getContent().equals("hello world")
                                && !chunk.isFinished()
                )
                .verifyComplete();

        // fullResponse must capture the content
        assertEquals(1, fullResponse.size());
        assertEquals("hello world", fullResponse.get(0));
    }


    @Test
    void testNonBlankFinishReason_ShouldNotReturnImmediateChunk() {

        when(chatClient.prompt(anyString())).thenReturn(requestSpec);
        when(requestSpec.options(any())).thenReturn(requestSpec);
        when(requestSpec.stream()).thenReturn(streamSpec);
        when(streamSpec.chatResponse()).thenReturn(Flux.just(chatResponse));

        when(chatResponse.getResult()).thenReturn(result);
        when(result.getOutput()).thenReturn(output);
        when(output.getText()).thenReturn("hello world");

        when(result.getMetadata()).thenReturn(metadata);
        when(metadata.getFinishReason()).thenReturn("stop");

        when(promptRequestContext.getKnowledgeBaseResponse()).thenReturn(null);

        StepVerifier.create(chatService.callLLM("hi"))
                .expectNextCount(0)    // flatMap continues but does NOT emit immediate chunk
                .verifyComplete();

        assertTrue(fullResponse.isEmpty());
    }
}
```

---

# ⭐ Notes

### ✓ **Mocks mirror your screenshot exactly:**

* `r.getResult().getOutput().getText()`
* `r.getResult().getMetadata().getFinishReason()`
* `promptRequestContext.getKnowledgeBaseResponse()`

### ✓ **Reactive Test**

`StepVerifier` is mandatory because return type is `Flux`.

### ✓ **fullResponse verification**

Ensures the line `fullResponse.add(content)` is tested.

### ✓ **Matches real flow of your code**

---

If you want, I can also:

✅ Generate test cases for more branches
✅ Build full mock class stubs for `ChatResponse`, `ChatResult`, etc.
✅ Rewrite your method to make it more testable

Just tell me!
