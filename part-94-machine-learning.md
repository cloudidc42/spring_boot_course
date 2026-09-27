# Part 94: Machine Learning Integration with Spring Boot
## ขั้นตอนที่ 3361-3400

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 5-6 ชั่วโมง  
> **เป้าหมาย:** Integrate ML models and AI services with Spring Boot applications

---

## ขั้นตอนที่ 3361: ML Integration Patterns

```
ML Integration Approaches:

1. REST/gRPC to Python ML Service
   ✅ Language-agnostic, team separation
   ✅ Scale ML independently
   ❌ Network latency
   ❌ Extra infrastructure

2. ONNX Runtime in JVM
   ✅ No network overhead
   ✅ Run model directly in Java
   ❌ Conversion step required
   ❌ Some ops not supported

3. Spring AI (new)
   ✅ Native Spring integration for LLMs
   ✅ OpenAI, Anthropic, Ollama support
   ❌ Focus on language models only

4. DJL (Deep Java Library, Amazon)
   ✅ Multiple frameworks (PyTorch, TF, MXNet)
   ✅ Java native
   ❌ Less mature ecosystem
```

---

## ขั้นตอนที่ 3362: Calling Python ML Service

```java
// Python FastAPI ML Service (ฝั่ง Python, ตัวอย่าง)
/*
@app.post("/predict/product-category")
def predict_category(product: ProductRequest):
    features = extract_features(product.description)
    prediction = model.predict([features])[0]
    confidence = model.predict_proba([features])[0].max()
    return {"category": prediction, "confidence": float(confidence)}
*/

// Spring Boot WebClient to call ML service
@Service
@RequiredArgsConstructor
@Slf4j
public class ProductCategorizationService {

    private final WebClient mlServiceClient;

    @Cacheable(value = "ml-predictions", key = "#description.hashCode()")
    public CategoryPrediction predictCategory(String description) {
        return mlServiceClient.post()
            .uri("/predict/product-category")
            .bodyValue(new ProductRequest(description))
            .retrieve()
            .bodyToMono(CategoryPrediction.class)
            .timeout(Duration.ofSeconds(2))
            .doOnError(ex -> log.error("ML prediction failed", ex))
            .onErrorReturn(CategoryPrediction.unknown())
            .block();
    }
}

@Bean
public WebClient mlServiceClient(@Value("${ml.service.url}") String baseUrl) {
    return WebClient.builder()
        .baseUrl(baseUrl)
        .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        .filter(ExchangeFilterFunctions.basicAuthentication("api", "${ml.api.key}"))
        .build();
}

record ProductRequest(String description) {}
record CategoryPrediction(String category, double confidence) {
    static CategoryPrediction unknown() {
        return new CategoryPrediction("UNKNOWN", 0.0);
    }
}
```

---

## ขั้นตอนที่ 3363: ONNX Runtime Integration

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.microsoft.onnxruntime</groupId>
    <artifactId>onnxruntime</artifactId>
    <version>1.17.0</version>
</dependency>
```

```java
// Load ONNX model and run inference in Java
@Service
@Slf4j
public class SentimentAnalysisService implements DisposableBean {

    private final OrtEnvironment env;
    private final OrtSession session;
    private final Map<String, Integer> vocabulary;

    public SentimentAnalysisService(
        @Value("classpath:models/sentiment.onnx") Resource modelResource,
        @Value("classpath:models/vocab.json") Resource vocabResource
    ) throws OrtException, IOException {
        this.env = OrtEnvironment.getEnvironment();
        this.session = env.createSession(
            modelResource.getInputStream().readAllBytes(),
            new OrtSession.SessionOptions()
        );
        this.vocabulary = loadVocabulary(vocabResource);
        log.info("ONNX sentiment model loaded");
    }

    public SentimentResult analyze(String text) {
        try {
            long[] inputIds = tokenize(text);

            OnnxTensor inputTensor = OnnxTensor.createTensor(env,
                new long[][]{inputIds});

            Map<String, OnnxTensor> inputs = Map.of("input_ids", inputTensor);

            try (OrtSession.Result result = session.run(inputs)) {
                float[][] logits = (float[][]) result.get(0).getValue();
                float[] probs = softmax(logits[0]);
                int predicted = argmax(probs);

                return new SentimentResult(
                    predicted == 1 ? "POSITIVE" : "NEGATIVE",
                    probs[predicted]
                );
            }
        } catch (OrtException e) {
            throw new RuntimeException("ONNX inference failed", e);
        }
    }

    @Override
    public void destroy() throws Exception {
        session.close();
        env.close();
    }

    private float[] softmax(float[] logits) {
        float max = Float.NEGATIVE_INFINITY;
        for (float v : logits) if (v > max) max = v;
        float sum = 0;
        float[] exp = new float[logits.length];
        for (int i = 0; i < logits.length; i++) {
            exp[i] = (float) Math.exp(logits[i] - max);
            sum += exp[i];
        }
        for (int i = 0; i < exp.length; i++) exp[i] /= sum;
        return exp;
    }

    private int argmax(float[] arr) {
        int idx = 0;
        for (int i = 1; i < arr.length; i++) if (arr[i] > arr[idx]) idx = i;
        return idx;
    }
}

record SentimentResult(String sentiment, float confidence) {}
```

---

## ขั้นตอนที่ 3364: Spring AI (LLM Integration)

```xml
<!-- Spring AI for LLM integration -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

```java
// application.yml
/*
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        model: gpt-4o-mini
        temperature: 0.3
*/

@Service
@RequiredArgsConstructor
public class ProductDescriptionService {

    private final ChatClient chatClient;

    public String generateDescription(Product product) {
        String prompt = """
            สร้าง product description ภาษาไทยสำหรับ:
            ชื่อ: %s
            หมวดหมู่: %s
            ราคา: %.2f บาท
            
            ความยาว: 2-3 ประโยค น่าสนใจ ดึงดูดใจลูกค้า
            """.formatted(product.getName(), product.getCategory(), product.getPrice());

        return chatClient.call(prompt);
    }

    public List<String> generateSearchKeywords(String description) {
        String prompt = "Extract 5 Thai search keywords from: " + description;
        String response = chatClient.call(prompt);
        return Arrays.asList(response.split(","));
    }
}
```

---

## ขั้นตอนที่ 3365: A/B Testing for ML Models

```java
// A/B test: compare old model vs new model
@Service
@RequiredArgsConstructor
public class ModelABTestService {

    private final ModelV1 modelV1;
    private final ModelV2 modelV2;
    private final FeatureFlagService featureFlags;
    private final MeterRegistry registry;

    public Prediction predict(Long userId, PredictionRequest request) {
        boolean useV2 = featureFlags.isEnabledForUser("ml-model-v2", userId);
        String modelVersion = useV2 ? "v2" : "v1";

        Timer.Sample timer = Timer.start(registry);
        Prediction prediction;

        try {
            prediction = useV2
                ? modelV2.predict(request)
                : modelV1.predict(request);

            timer.stop(Timer.builder("ml.prediction.latency")
                .tag("model", modelVersion)
                .register(registry));

            Counter.builder("ml.predictions")
                .tag("model", modelVersion)
                .tag("category", prediction.category())
                .register(registry)
                .increment();

            return prediction;
        } catch (Exception e) {
            Counter.builder("ml.prediction.errors")
                .tag("model", modelVersion)
                .register(registry)
                .increment();
            throw e;
        }
    }
}
```

---

## ขั้นตอนที่ 3366-3400: Recommendation Engine

```java
// ดึง recommendations จาก external recommendation service
@Service
@RequiredArgsConstructor
public class RecommendationService {

    private final WebClient recoClient;
    private final ProductRepository productRepository;

    @Cacheable(value = "recommendations", key = "#userId")
    public List<Product> getRecommendations(Long userId, int limit) {
        // Call recommendation engine
        List<Long> productIds = recoClient.get()
            .uri(uri -> uri.path("/recommend/{userId}")
                .queryParam("limit", limit)
                .build(userId))
            .retrieve()
            .bodyToMono(new ParameterizedTypeReference<List<Long>>() {})
            .timeout(Duration.ofMillis(500))
            .onErrorReturn(List.of())  // fallback: empty = show popular
            .block();

        if (productIds.isEmpty()) {
            return productRepository.findTopByOrderByViewCountDesc(PageRequest.of(0, limit));
        }

        // Fetch product details preserving order
        Map<Long, Product> productMap = productRepository.findAllById(productIds)
            .stream()
            .collect(Collectors.toMap(Product::getId, p -> p));

        return productIds.stream()
            .map(productMap::get)
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }
}
```

---

*[← Part 93: Cost Optimization](./part-93-cost-optimization.md) | [Part 95: Advanced Testing →](./part-95-advanced-testing.md)*
