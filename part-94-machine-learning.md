# Part 94: Machine Learning Integration with Spring Boot
## ขั้นตอนที่ 3361-3400

**ระดับ:** World-Class (ระดับโลก)
**เวลาเรียน:** 6-8 ชั่วโมง
**เป้าหมาย:** เรียนรู้การนำ Machine Learning มาผสาน Spring Boot application ครอบคลุมการเรียก Python ML services ผ่าน REST, ONNX Model inference ใน Java, Feature store patterns, A/B testing สำหรับ ML models, Model versioning, และ Recommendation engine

---

## ขั้นตอนที่ 3361: ภาพรวม ML Integration Patterns

มี 3 รูปแบบหลักในการนำ ML มาใช้กับ Spring Boot:

```
Pattern 1: Sidecar ML Service
Spring Boot App ←REST→ Python FastAPI ML Service
  장점: ยืดหยุ่น, ใช้ library ML ได้เต็มที่
  ข้อเสีย: Network overhead, ดูแลอีก service

Pattern 2: ONNX Inference ใน Java
Spring Boot App → ONNX Runtime (Java) → Model file
  장점: ไม่มี network hop, latency ต่ำ
  ข้อเสีย: Model types จำกัด

Pattern 3: Cloud ML Service
Spring Boot App ←API→ AWS SageMaker / Google Vertex AI
  장점: Managed, scale อัตโนมัติ
  ข้อเสีย: ค่าใช้จ่ายสูง, vendor lock-in
```

## ขั้นตอนที่ 3362: Python ML Service ด้วย FastAPI

สร้าง Python service สำหรับ ML inference ที่ Spring Boot เรียกผ่าน REST

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
import joblib
import torch
from transformers import pipeline

app = FastAPI(title="ShopHub ML Service", version="1.0.0")

# โหลด models ตอน startup
class ModelRegistry:
    def __init__(self):
        self.recommendation_model = None
        self.sentiment_model = None
        self.fraud_model = None
        
    def load_models(self):
        print("Loading ML models...")
        # โหลด recommendation model
        self.recommendation_model = joblib.load("models/recommendation_v2.pkl")
        
        # โหลด sentiment model
        self.sentiment_model = pipeline(
            "sentiment-analysis",
            model="nlptown/bert-base-multilingual-uncased-sentiment",
            device="cpu"
        )
        
        # โหลด fraud detection model
        self.fraud_model = joblib.load("models/fraud_detection_v1.pkl")
        print("All models loaded!")

model_registry = ModelRegistry()

@app.on_event("startup")
async def startup():
    model_registry.load_models()

# Request/Response schemas
class RecommendationRequest(BaseModel):
    user_id: int
    user_features: List[float]
    product_history: List[int]
    context: dict

class RecommendationResponse(BaseModel):
    user_id: int
    recommended_products: List[int]
    scores: List[float]
    model_version: str

class SentimentRequest(BaseModel):
    texts: List[str]
    language: str = "th"

class FraudRequest(BaseModel):
    order_id: str
    amount: float
    user_age_days: int
    distinct_item_count: int
    shipping_match_billing: bool
    hour_of_day: int
    country: str

# Endpoints
@app.post("/recommend", response_model=RecommendationResponse)
async def recommend_products(request: RecommendationRequest):
    try:
        model = model_registry.recommendation_model
        
        features = np.array(request.user_features + request.product_history)
        scores = model.predict_proba(features.reshape(1, -1))[0]
        
        # ดึง top-10 products
        top_indices = np.argsort(scores)[::-1][:10]
        recommended = top_indices.tolist()
        top_scores = scores[top_indices].tolist()
        
        return RecommendationResponse(
            user_id=request.user_id,
            recommended_products=recommended,
            scores=top_scores,
            model_version="v2.0"
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/sentiment")
async def analyze_sentiment(request: SentimentRequest):
    results = model_registry.sentiment_model(request.texts)
    return {
        "results": [
            {"text": text, "label": r["label"], "score": r["score"]}
            for text, r in zip(request.texts, results)
        ]
    }

@app.post("/fraud-detection")
async def detect_fraud(request: FraudRequest):
    features = [
        request.amount,
        request.user_age_days,
        request.distinct_item_count,
        1 if request.shipping_match_billing else 0,
        request.hour_of_day,
        hash(request.country) % 100
    ]
    
    model = model_registry.fraud_model
    fraud_prob = model.predict_proba([features])[0][1]
    is_fraud = fraud_prob > 0.7
    
    return {
        "order_id": request.order_id,
        "fraud_probability": float(fraud_prob),
        "is_fraud": is_fraud,
        "risk_level": "HIGH" if fraud_prob > 0.7 else "MEDIUM" if fraud_prob > 0.4 else "LOW"
    }

@app.get("/health")
async def health():
    return {"status": "healthy", "models_loaded": True}
```

## ขั้นตอนที่ 3363: Spring Boot ML Client

```java
// ml-service/src/main/java/com/shophub/ml/client/MLServiceClient.java
package com.shophub.ml.client;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;

@FeignClient(
    name = "ml-service",
    url = "${ml.service.url:http://ml-service:8000}",
    fallback = MLServiceClientFallback.class
)
public interface MLServiceClient {

    @PostMapping("/recommend")
    RecommendationResponse getRecommendations(@RequestBody RecommendationRequest request);

    @PostMapping("/sentiment")
    SentimentResponse analyzeSentiment(@RequestBody SentimentRequest request);

    @PostMapping("/fraud-detection")
    FraudDetectionResponse detectFraud(@RequestBody FraudDetectionRequest request);
}
```

```java
// ml-service/src/main/java/com/shophub/ml/dto/RecommendationRequest.java
package com.shophub.ml.dto;

import lombok.Builder;
import lombok.Data;
import java.util.List;
import java.util.Map;

@Data
@Builder
public class RecommendationRequest {
    private Long userId;
    private List<Double> userFeatures;
    private List<Long> productHistory;
    private Map<String, Object> context;
}
```

```java
// ml-service/src/main/java/com/shophub/ml/dto/RecommendationResponse.java
package com.shophub.ml.dto;

import lombok.Data;
import java.util.List;

@Data
public class RecommendationResponse {
    private Long userId;
    private List<Long> recommendedProducts;
    private List<Double> scores;
    private String modelVersion;
}
```

```java
// RecommendationService.java - ใช้ ML service ใน product recommendation
@Service
@RequiredArgsConstructor
@Slf4j
public class RecommendationService {

    private final MLServiceClient mlClient;
    private final UserFeatureService userFeatureService;
    private final ProductRepository productRepository;
    private final CacheManager cacheManager;

    public List<ProductDto> getPersonalizedRecommendations(Long userId, int limit) {
        // ดึง cache ก่อน
        String cacheKey = "recommendations:" + userId;
        Cache cache = cacheManager.getCache("recommendations");
        
        if (cache != null) {
            Cache.ValueWrapper cached = cache.get(cacheKey);
            if (cached != null) {
                return (List<ProductDto>) cached.get();
            }
        }

        // ดึง user features จาก Feature Store
        UserFeatures features = userFeatureService.getUserFeatures(userId);
        
        RecommendationRequest request = RecommendationRequest.builder()
                .userId(userId)
                .userFeatures(features.toVector())
                .productHistory(features.getRecentlyViewedProducts())
                .context(Map.of(
                        "time_of_day", LocalTime.now().getHour(),
                        "day_of_week", LocalDate.now().getDayOfWeek().name(),
                        "session_count", features.getSessionCount()
                ))
                .build();

        RecommendationResponse response;
        try {
            response = mlClient.getRecommendations(request);
        } catch (Exception e) {
            log.error("ML service unavailable, using fallback recommendations", e);
            return getFallbackRecommendations(userId, limit);
        }

        // ดึง product details
        List<ProductDto> products = productRepository.findAllById(
                response.getRecommendedProducts().subList(0, Math.min(limit, response.getRecommendedProducts().size()))
        ).stream().map(this::toDto).collect(Collectors.toList());

        // Cache result 10 นาที
        if (cache != null) {
            cache.put(cacheKey, products);
        }

        return products;
    }

    private List<ProductDto> getFallbackRecommendations(Long userId, int limit) {
        // Fallback: ส่ง popular products แทน
        return productRepository.findTop10ByOrderByViewCountDesc()
                .stream().limit(limit).map(this::toDto).collect(Collectors.toList());
    }
}
```

## ขั้นตอนที่ 3364: ONNX Model Inference ใน Java

ONNX (Open Neural Network Exchange) ช่วยให้รัน ML model โดยตรงใน Java โดยไม่ต้องเรียก Python service

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.microsoft.onnxruntime</groupId>
    <artifactId>onnxruntime</artifactId>
    <version>1.16.3</version>
</dependency>
```

### Export Model เป็น ONNX (Python)

```python
# export_to_onnx.py
import torch
import torch.nn as nn
from skl2onnx import convert_sklearn
from skl2onnx.common.data_types import FloatTensorType
import joblib

# Export scikit-learn model
model = joblib.load("fraud_model.pkl")
initial_type = [("float_input", FloatTensorType([None, 6]))]
onnx_model = convert_sklearn(model, initial_types=initial_type)

with open("fraud_model.onnx", "wb") as f:
    f.write(onnx_model.SerializeToString())

print("Fraud model exported to ONNX!")

# Export PyTorch model
class RecommendationModel(nn.Module):
    def __init__(self, user_emb_size, item_emb_size, hidden_size):
        super().__init__()
        self.user_embedding = nn.Embedding(10000, user_emb_size)
        self.item_embedding = nn.Embedding(50000, item_emb_size)
        self.fc = nn.Sequential(
            nn.Linear(user_emb_size + item_emb_size, hidden_size),
            nn.ReLU(),
            nn.Linear(hidden_size, 1),
            nn.Sigmoid()
        )
    
    def forward(self, user_id, item_id):
        user_emb = self.user_embedding(user_id)
        item_emb = self.item_embedding(item_id)
        x = torch.cat([user_emb, item_emb], dim=1)
        return self.fc(x)

model = RecommendationModel(64, 64, 128)
model.load_state_dict(torch.load("recommendation_model.pth"))
model.eval()

dummy_user = torch.LongTensor([1])
dummy_item = torch.LongTensor([1])

torch.onnx.export(
    model, (dummy_user, dummy_item),
    "recommendation_model.onnx",
    input_names=["user_id", "item_id"],
    output_names=["score"],
    dynamic_axes={
        "user_id": {0: "batch_size"},
        "item_id": {0: "batch_size"}
    }
)
print("Recommendation model exported to ONNX!")
```

### ONNX Inference Service ใน Java

```java
// OnnxInferenceService.java
package com.shophub.ml.onnx;

import ai.onnxruntime.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import java.nio.file.Path;
import java.util.*;

@Service
@Slf4j
public class FraudDetectionOnnxService {

    @Value("${ml.models.fraud.path:models/fraud_model.onnx}")
    private String modelPath;

    private OrtEnvironment environment;
    private OrtSession session;
    private OrtSession.SessionOptions options;

    @PostConstruct
    public void initialize() throws OrtException {
        log.info("Loading ONNX fraud detection model from: {}", modelPath);
        
        environment = OrtEnvironment.getEnvironment();
        options = new OrtSession.SessionOptions();
        
        // Enable optimizations
        options.setOptimizationLevel(OrtSession.SessionOptions.OptLevel.ALL_OPT);
        options.setIntraOpNumThreads(2);
        options.setInterOpNumThreads(1);
        
        // Enable CPU provider (or CUDA if GPU available)
        // options.addCUDA(0); // Enable GPU inference
        
        session = environment.createSession(modelPath, options);
        
        log.info("ONNX model loaded successfully. Input nodes: {}", session.getInputNames());
    }

    public FraudPrediction predict(FraudFeatures features) throws OrtException {
        // สร้าง input tensor
        float[][] inputData = {{
            (float) features.getAmount(),
            (float) features.getUserAgeDays(),
            (float) features.getDistinctItemCount(),
            features.isShippingMatchBilling() ? 1.0f : 0.0f,
            (float) features.getHourOfDay(),
            (float) (Math.abs(features.getCountry().hashCode()) % 100)
        }};

        try (OnnxTensor inputTensor = OnnxTensor.createTensor(environment, inputData);
             OrtSession.Result result = session.run(
                     Map.of("float_input", inputTensor))) {

            float[][] output = (float[][]) result.get(0).getValue();
            float fraudProb = output[0][1]; // probability of class 1 (fraud)

            return FraudPrediction.builder()
                    .fraudProbability(fraudProb)
                    .isFraud(fraudProb > 0.7f)
                    .riskLevel(getRiskLevel(fraudProb))
                    .build();
        }
    }

    private String getRiskLevel(float fraudProb) {
        if (fraudProb > 0.7) return "HIGH";
        if (fraudProb > 0.4) return "MEDIUM";
        return "LOW";
    }

    @PreDestroy
    public void cleanup() throws OrtException {
        if (session != null) session.close();
        if (options != null) options.close();
    }
}
```

```java
// RecommendationOnnxService.java - Batch inference สำหรับ performance
@Service
@Slf4j
public class RecommendationOnnxService {

    private OrtEnvironment environment;
    private OrtSession session;

    @PostConstruct
    public void initialize() throws OrtException {
        environment = OrtEnvironment.getEnvironment();
        OrtSession.SessionOptions opts = new OrtSession.SessionOptions();
        opts.setOptimizationLevel(OrtSession.SessionOptions.OptLevel.ALL_OPT);
        session = environment.createSession("models/recommendation_model.onnx", opts);
    }

    // Batch inference - รัน model หลาย items พร้อมกัน
    public List<Float> scoreItems(long userId, List<Long> itemIds) throws OrtException {
        int batchSize = itemIds.size();
        
        long[] userIds = new long[batchSize];
        long[] itemIdArr = new long[batchSize];
        Arrays.fill(userIds, userId);
        
        for (int i = 0; i < batchSize; i++) {
            itemIdArr[i] = itemIds.get(i);
        }

        try (OnnxTensor userTensor = OnnxTensor.createTensor(environment, userIds);
             OnnxTensor itemTensor = OnnxTensor.createTensor(environment, itemIdArr);
             OrtSession.Result result = session.run(
                     Map.of("user_id", userTensor, "item_id", itemTensor))) {

            float[][] scores = (float[][]) result.get(0).getValue();
            List<Float> scoreList = new ArrayList<>();
            for (float[] score : scores) {
                scoreList.add(score[0]);
            }
            return scoreList;
        }
    }

    // Top-K recommendation
    public List<Long> topKRecommendations(long userId, List<Long> candidateItems, int k) 
            throws OrtException {
        List<Float> scores = scoreItems(userId, candidateItems);
        
        // Rank items by score
        List<Map.Entry<Long, Float>> scored = new ArrayList<>();
        for (int i = 0; i < candidateItems.size(); i++) {
            scored.add(Map.entry(candidateItems.get(i), scores.get(i)));
        }
        
        return scored.stream()
                .sorted(Map.Entry.<Long, Float>comparingByValue().reversed())
                .limit(k)
                .map(Map.Entry::getKey)
                .collect(Collectors.toList());
    }
}
```

## ขั้นตอนที่ 3365: Feature Store Pattern

Feature Store เก็บ features ที่ compute แล้ว ให้ทั้ง training และ inference ใช้ร่วมกัน ป้องกัน training-serving skew

```java
// FeatureStore interface
public interface FeatureStore {
    UserFeatures getUserFeatures(Long userId);
    void updateUserFeatures(Long userId, UserFeatures features);
    ProductFeatures getProductFeatures(Long productId);
    void updateProductFeatures(Long productId, ProductFeatures features);
}
```

```java
// RedisFeatureStore.java - Online Feature Store ใช้ Redis
@Service
@RequiredArgsConstructor
@Slf4j
public class RedisFeatureStore implements FeatureStore {

    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;
    
    private static final String USER_FEATURES_PREFIX = "features:user:";
    private static final Duration FEATURE_TTL = Duration.ofHours(24);

    @Override
    public UserFeatures getUserFeatures(Long userId) {
        String key = USER_FEATURES_PREFIX + userId;
        String json = redisTemplate.opsForValue().get(key);
        
        if (json == null) {
            return computeAndStoreUserFeatures(userId);
        }
        
        try {
            return objectMapper.readValue(json, UserFeatures.class);
        } catch (JsonProcessingException e) {
            log.error("Failed to deserialize user features for userId: {}", userId);
            return computeAndStoreUserFeatures(userId);
        }
    }

    @Override
    public void updateUserFeatures(Long userId, UserFeatures features) {
        String key = USER_FEATURES_PREFIX + userId;
        try {
            String json = objectMapper.writeValueAsString(features);
            redisTemplate.opsForValue().set(key, json, FEATURE_TTL);
        } catch (JsonProcessingException e) {
            log.error("Failed to serialize user features for userId: {}", userId);
        }
    }

    private UserFeatures computeAndStoreUserFeatures(Long userId) {
        // Compute features จาก raw data
        UserFeatures features = computeUserFeatures(userId);
        updateUserFeatures(userId, features);
        return features;
    }

    private UserFeatures computeUserFeatures(Long userId) {
        // Aggregate user behavior features
        // ในระบบจริง จะ query จาก analytics DB หรือ event stream
        return UserFeatures.builder()
                .userId(userId)
                .totalOrders(0)
                .avgOrderValue(0.0)
                .preferredCategories(List.of())
                .recentlyViewedProducts(List.of())
                .sessionCount(0)
                .daysSinceLastOrder(0)
                .computedAt(Instant.now())
                .build();
    }
}
```

```java
// UserFeatures.java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserFeatures {
    private Long userId;
    private int totalOrders;
    private double avgOrderValue;
    private List<String> preferredCategories;
    private List<Long> recentlyViewedProducts;
    private int sessionCount;
    private int daysSinceLastOrder;
    private String userSegment; // "new", "active", "vip", "churned"
    private Instant computedAt;

    // แปลงเป็น feature vector สำหรับ ML model
    public List<Double> toVector() {
        return List.of(
                (double) totalOrders,
                avgOrderValue,
                (double) preferredCategories.size(),
                (double) sessionCount,
                (double) daysSinceLastOrder,
                "vip".equals(userSegment) ? 1.0 : 0.0
        );
    }
}
```

### Feature Pipeline สำหรับ update features

```java
// FeatureUpdatePipeline.java - อัปเดต features เมื่อเกิด events
@Service
@RequiredArgsConstructor
@Slf4j
public class FeatureUpdatePipeline {

    private final FeatureStore featureStore;
    private final OrderRepository orderRepository;

    // รับ event เมื่อ order สร้างสำเร็จ
    @KafkaListener(topics = "order.created", groupId = "feature-pipeline")
    public void handleOrderCreated(OrderCreatedEvent event) {
        log.debug("Updating features for user: {} after order creation", event.getUserId());
        
        // คำนวณ features ใหม่
        UserFeatures currentFeatures = featureStore.getUserFeatures(event.getUserId());
        
        // อัปเดต order-related features
        UserFeatures updatedFeatures = currentFeatures.toBuilder()
                .totalOrders(currentFeatures.getTotalOrders() + 1)
                .daysSinceLastOrder(0) // เพิ่ง order
                .avgOrderValue(calculateNewAvg(
                        currentFeatures.getAvgOrderValue(),
                        currentFeatures.getTotalOrders(),
                        event.getTotalAmount().doubleValue()
                ))
                .computedAt(Instant.now())
                .build();

        featureStore.updateUserFeatures(event.getUserId(), updatedFeatures);
    }

    // อัปเดต product features เมื่อสินค้าถูกดู
    @EventListener
    public void handleProductViewed(ProductViewedEvent event) {
        // อัปเดต product popularity features
        ProductFeatures features = featureStore.getProductFeatures(event.getProductId());
        ProductFeatures updated = features.toBuilder()
                .viewCount(features.getViewCount() + 1)
                .lastViewedAt(Instant.now())
                .build();
        featureStore.updateProductFeatures(event.getProductId(), updated);
    }

    private double calculateNewAvg(double currentAvg, int currentCount, double newValue) {
        return (currentAvg * currentCount + newValue) / (currentCount + 1);
    }
}
```

## ขั้นตอนที่ 3366: A/B Testing สำหรับ ML Models

A/B Testing ช่วยทดสอบ model ใหม่กับ traffic จริงอย่างปลอดภัย

```java
// ABTestingService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class MLABTestingService {

    private final MLExperimentRepository experimentRepository;
    private final RecommendationOnnxService modelV1;
    private final RecommendationOnnxService modelV2; // New model
    private final MLMetricsService metricsService;

    public List<Long> getRecommendations(Long userId, List<Long> candidates, int k) {
        // ดึง experiment config
        Optional<MLExperiment> experiment = experimentRepository
                .findActiveByName("recommendation-model-v2");

        if (experiment.isEmpty()) {
            // ไม่มี experiment - ใช้ model หลัก
            return getRecommendationsFromModel(modelV1, userId, candidates, k, "v1");
        }

        MLExperiment exp = experiment.get();
        
        // กำหนด variant ตาม user ID (consistent bucketing)
        String variant = assignVariant(userId, exp.getTrafficSplit());
        
        try {
            if ("control".equals(variant)) {
                return getRecommendationsFromModel(modelV1, userId, candidates, k, "v1");
            } else {
                return getRecommendationsFromModel(modelV2, userId, candidates, k, "v2");
            }
        } finally {
            // บันทึก assignment สำหรับ analysis
            metricsService.recordAssignment(exp.getId(), userId, variant);
        }
    }

    private String assignVariant(Long userId, double trafficSplit) {
        // Consistent bucketing: ใช้ hash เพื่อให้ user เจอ variant เดิมทุกครั้ง
        int bucket = Math.abs(userId.hashCode()) % 100;
        return bucket < (trafficSplit * 100) ? "treatment" : "control";
    }

    private List<Long> getRecommendationsFromModel(
            RecommendationOnnxService model, Long userId,
            List<Long> candidates, int k, String modelVersion) {
        try {
            long startTime = System.currentTimeMillis();
            List<Long> results = model.topKRecommendations(userId, candidates, k);
            long latency = System.currentTimeMillis() - startTime;

            metricsService.recordInference(modelVersion, latency, results.size());
            return results;
        } catch (OrtException e) {
            log.error("Model inference failed for version {}: {}", modelVersion, e.getMessage());
            throw new RuntimeException("Recommendation failed", e);
        }
    }
}
```

```java
// MLExperiment.java
@Entity
@Table(name = "ml_experiments")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class MLExperiment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String description;
    private double trafficSplit; // 0.0-1.0 (proportion going to treatment)
    private boolean active;
    private LocalDateTime startDate;
    private LocalDateTime endDate;
    
    @Enumerated(EnumType.STRING)
    private ExperimentStatus status;

    public enum ExperimentStatus {
        DRAFT, RUNNING, PAUSED, COMPLETED
    }
}
```

### Metrics Collection สำหรับ A/B Testing

```java
// MLMetricsService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class MLMetricsService {

    private final MLMetricsRepository metricsRepository;
    private final MeterRegistry meterRegistry;

    // บันทึก click event เมื่อ user click สินค้าที่ recommend
    public void recordClick(Long userId, Long productId, String modelVersion) {
        MLEvent event = MLEvent.builder()
                .userId(userId)
                .eventType("CLICK")
                .productId(productId)
                .modelVersion(modelVersion)
                .timestamp(Instant.now())
                .build();
        metricsRepository.save(event);

        meterRegistry.counter("ml.recommendation.click",
                "model_version", modelVersion).increment();
    }

    // บันทึก purchase event เมื่อ user ซื้อสินค้าที่ recommend
    public void recordPurchase(Long userId, Long productId, String modelVersion, double revenue) {
        MLEvent event = MLEvent.builder()
                .userId(userId)
                .eventType("PURCHASE")
                .productId(productId)
                .modelVersion(modelVersion)
                .revenue(revenue)
                .timestamp(Instant.now())
                .build();
        metricsRepository.save(event);

        meterRegistry.counter("ml.recommendation.purchase",
                "model_version", modelVersion).increment();
        meterRegistry.gauge("ml.recommendation.revenue",
                Tags.of("model_version", modelVersion), revenue);
    }

    // คำนวณ CTR และ Conversion Rate สำหรับแต่ละ model
    public ExperimentMetrics calculateMetrics(Long experimentId, String period) {
        List<MLEvent> events = metricsRepository
                .findByExperimentIdAndPeriod(experimentId, period);

        Map<String, Long> clicks = events.stream()
                .filter(e -> "CLICK".equals(e.getEventType()))
                .collect(Collectors.groupingBy(MLEvent::getModelVersion, Collectors.counting()));

        Map<String, Long> purchases = events.stream()
                .filter(e -> "PURCHASE".equals(e.getEventType()))
                .collect(Collectors.groupingBy(MLEvent::getModelVersion, Collectors.counting()));

        Map<String, Double> revenue = events.stream()
                .filter(e -> "PURCHASE".equals(e.getEventType()))
                .collect(Collectors.groupingBy(MLEvent::getModelVersion,
                        Collectors.summingDouble(MLEvent::getRevenue)));

        return ExperimentMetrics.builder()
                .experimentId(experimentId)
                .period(period)
                .clicksByModel(clicks)
                .purchasesByModel(purchases)
                .revenueByModel(revenue)
                .build();
    }
}
```

## ขั้นตอนที่ 3367: Model Versioning และ Rollout

```java
// ModelVersionManager.java - จัดการ versions ของ model
@Service
@RequiredArgsConstructor
@Slf4j
public class ModelVersionManager {

    private final ModelVersionRepository versionRepository;
    private final S3Client s3Client;
    private volatile Map<String, OrtSession> loadedSessions = new ConcurrentHashMap<>();
    private volatile Map<String, String> activeVersions = new ConcurrentHashMap<>();

    @Value("${aws.s3.models-bucket}")
    private String modelsBucket;

    // โหลด model version ใหม่
    public void loadModelVersion(String modelName, String version) throws Exception {
        log.info("Loading model {} version {}", modelName, version);

        // Download จาก S3
        String s3Key = String.format("models/%s/%s/model.onnx", modelName, version);
        String localPath = downloadModel(s3Key, modelName, version);

        // Load ONNX session
        OrtEnvironment env = OrtEnvironment.getEnvironment();
        OrtSession.SessionOptions opts = new OrtSession.SessionOptions();
        opts.setOptimizationLevel(OrtSession.SessionOptions.OptLevel.ALL_OPT);
        OrtSession newSession = env.createSession(localPath, opts);

        String sessionKey = modelName + ":" + version;
        loadedSessions.put(sessionKey, newSession);
        
        log.info("Model {} version {} loaded successfully", modelName, version);
    }

    // Promote version ให้เป็น active
    public void promoteVersion(String modelName, String version) {
        String sessionKey = modelName + ":" + version;
        if (!loadedSessions.containsKey(sessionKey)) {
            throw new IllegalStateException("Model not loaded: " + sessionKey);
        }

        String previousVersion = activeVersions.get(modelName);
        activeVersions.put(modelName, version);

        log.info("Promoted {} to version {} (was: {})", 
                modelName, version, previousVersion);

        // บันทึก version history
        versionRepository.save(ModelVersion.builder()
                .modelName(modelName)
                .version(version)
                .promotedAt(Instant.now())
                .previousVersion(previousVersion)
                .build());
    }

    // Rollback ไป version ก่อน
    public void rollback(String modelName) {
        ModelVersion current = versionRepository
                .findLatestByModelName(modelName)
                .orElseThrow();
        
        if (current.getPreviousVersion() == null) {
            throw new IllegalStateException("No previous version to rollback to");
        }

        log.warn("Rolling back {} from {} to {}", 
                modelName, current.getVersion(), current.getPreviousVersion());
        promoteVersion(modelName, current.getPreviousVersion());
    }

    // ดึง session ของ active version
    public OrtSession getActiveSession(String modelName) {
        String version = activeVersions.get(modelName);
        if (version == null) {
            throw new IllegalStateException("No active version for model: " + modelName);
        }
        return loadedSessions.get(modelName + ":" + version);
    }

    private String downloadModel(String s3Key, String modelName, String version) throws Exception {
        String localPath = String.format("/tmp/models/%s/%s/model.onnx", modelName, version);
        Files.createDirectories(Path.of(localPath).getParent());

        GetObjectRequest request = GetObjectRequest.builder()
                .bucket(modelsBucket)
                .key(s3Key)
                .build();

        s3Client.getObject(request, Path.of(localPath));
        return localPath;
    }
}
```

### REST API สำหรับ Model Management

```java
// ModelManagementController.java
@RestController
@RequestMapping("/api/admin/ml")
@RequiredArgsConstructor
@Slf4j
public class ModelManagementController {

    private final ModelVersionManager versionManager;

    @PostMapping("/models/{modelName}/versions/{version}/load")
    public ResponseEntity<ApiResponse<Void>> loadModel(
            @PathVariable String modelName,
            @PathVariable String version) throws Exception {
        versionManager.loadModelVersion(modelName, version);
        return ResponseEntity.ok(ApiResponse.success("Model loaded: " + modelName + " v" + version, null));
    }

    @PostMapping("/models/{modelName}/versions/{version}/promote")
    public ResponseEntity<ApiResponse<Void>> promoteModel(
            @PathVariable String modelName,
            @PathVariable String version) {
        versionManager.promoteVersion(modelName, version);
        return ResponseEntity.ok(ApiResponse.success("Model promoted: " + modelName + " v" + version, null));
    }

    @PostMapping("/models/{modelName}/rollback")
    public ResponseEntity<ApiResponse<Void>> rollback(@PathVariable String modelName) {
        versionManager.rollback(modelName);
        return ResponseEntity.ok(ApiResponse.success("Rollback successful for: " + modelName, null));
    }
}
```

## ขั้นตอนที่ 3368: Recommendation Engine Integration

การสร้าง recommendation engine ที่สมบูรณ์สำหรับ ShopHub

```java
// RecommendationEngine.java - Hybrid recommendation (CF + Content-based)
@Service
@RequiredArgsConstructor
@Slf4j
public class HybridRecommendationEngine {

    private final CollaborativeFilteringService cfService;
    private final ContentBasedFilteringService cbfService;
    private final PopularityBasedService popularityService;
    private final UserFeatureStore featureStore;
    private final MLABTestingService abTestingService;

    private static final int CANDIDATE_POOL_SIZE = 500;
    private static final int DEFAULT_RESULTS = 20;

    public RecommendationResult recommend(RecommendationContext context) {
        Long userId = context.getUserId();
        UserFeatures features = featureStore.getUserFeatures(userId);

        // เลือก strategy ตาม user profile
        List<Long> recommendations;

        if (features.getTotalOrders() == 0) {
            // New user: ใช้ popularity-based
            log.debug("New user {}: using popularity-based recommendations", userId);
            recommendations = popularityService.getPopularProducts(
                    context.getCategory(), DEFAULT_RESULTS);
        } else if (features.getTotalOrders() < 5) {
            // Cold start: ผสม content-based + popularity
            log.debug("Cold start user {}: hybrid mode", userId);
            List<Long> cbf = cbfService.recommend(userId, features, CANDIDATE_POOL_SIZE / 2);
            List<Long> popular = popularityService.getPopularProducts(null, CANDIDATE_POOL_SIZE / 2);
            recommendations = mergeAndRank(cbf, popular, features, DEFAULT_RESULTS);
        } else {
            // Warm user: ผสม CF + content-based ด้วย A/B test
            log.debug("Warm user {}: collaborative filtering", userId);
            List<Long> candidates = generateCandidates(features, CANDIDATE_POOL_SIZE);
            recommendations = abTestingService.getRecommendations(userId, candidates, DEFAULT_RESULTS);
        }

        // ตัด items ที่ user ซื้อไปแล้วออก
        List<Long> filteredRecommendations = filterPurchased(userId, recommendations, features);

        return RecommendationResult.builder()
                .userId(userId)
                .products(filteredRecommendations)
                .strategy(getStrategyName(features))
                .generatedAt(Instant.now())
                .build();
    }

    private List<Long> generateCandidates(UserFeatures features, int size) {
        List<Long> candidates = new ArrayList<>();

        // CF candidates
        candidates.addAll(cfService.getCandidates(features.getUserId(), size / 2));

        // Content-based candidates จาก preferred categories
        candidates.addAll(cbfService.getCandidatesByCategories(
                features.getPreferredCategories(), size / 2));

        return candidates.stream().distinct().collect(Collectors.toList());
    }

    private List<Long> filterPurchased(Long userId, List<Long> products, UserFeatures features) {
        // ตัดสินค้าที่เคยซื้อแล้วออก (ยกเว้น consumables)
        Set<Long> purchasedIds = new HashSet<>(features.getPurchasedProducts());
        return products.stream()
                .filter(id -> !purchasedIds.contains(id))
                .collect(Collectors.toList());
    }

    private List<Long> mergeAndRank(List<Long> list1, List<Long> list2,
                                     UserFeatures features, int limit) {
        LinkedHashSet<Long> merged = new LinkedHashSet<>(list1);
        merged.addAll(list2);
        return merged.stream().limit(limit).collect(Collectors.toList());
    }

    private String getStrategyName(UserFeatures features) {
        if (features.getTotalOrders() == 0) return "popularity";
        if (features.getTotalOrders() < 5) return "cold-start-hybrid";
        return "collaborative-filtering";
    }
}
```

## ขั้นตอนที่ 3369: Sentiment Analysis สำหรับ Product Reviews

```java
// ReviewSentimentService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class ReviewSentimentService {

    private final MLServiceClient mlClient;
    private final ReviewRepository reviewRepository;

    // วิเคราะห์ sentiment ของ review ใหม่
    @Async
    @EventListener
    public void analyzeNewReview(ReviewCreatedEvent event) {
        try {
            SentimentRequest request = new SentimentRequest();
            request.setTexts(List.of(event.getReviewText()));
            request.setLanguage("th");

            SentimentResponse response = mlClient.analyzeSentiment(request);
            SentimentResult result = response.getResults().get(0);

            // อัปเดต review ด้วย sentiment
            reviewRepository.updateSentiment(
                    event.getReviewId(),
                    result.getLabel(),
                    result.getScore()
            );

            // อัปเดต product sentiment summary
            updateProductSentimentSummary(event.getProductId());

        } catch (Exception e) {
            log.error("Failed to analyze sentiment for review: {}", event.getReviewId(), e);
        }
    }

    // คำนวณ sentiment summary สำหรับ product
    public ProductSentimentSummary getProductSentimentSummary(Long productId) {
        List<Review> reviews = reviewRepository.findByProductId(productId);

        Map<String, Long> sentimentCounts = reviews.stream()
                .filter(r -> r.getSentimentLabel() != null)
                .collect(Collectors.groupingBy(
                        Review::getSentimentLabel, Collectors.counting()
                ));

        double averageScore = reviews.stream()
                .filter(r -> r.getSentimentScore() != null)
                .mapToDouble(Review::getSentimentScore)
                .average()
                .orElse(0.0);

        return ProductSentimentSummary.builder()
                .productId(productId)
                .totalReviews(reviews.size())
                .positiveCount(sentimentCounts.getOrDefault("POSITIVE", 0L))
                .negativeCount(sentimentCounts.getOrDefault("NEGATIVE", 0L))
                .neutralCount(sentimentCounts.getOrDefault("NEUTRAL", 0L))
                .averageSentimentScore(averageScore)
                .overallSentiment(averageScore > 0.6 ? "POSITIVE" : 
                                  averageScore < 0.4 ? "NEGATIVE" : "NEUTRAL")
                .build();
    }
}
```

## ขั้นตอนที่ 3370-3400: สรุปและ Best Practices

### ML Integration Testing

```java
// RecommendationServiceTest.java
@SpringBootTest
@ExtendWith(MockitoExtension.class)
class RecommendationServiceTest {

    @MockBean
    private MLServiceClient mlClient;

    @MockBean
    private FeatureStore featureStore;

    @Autowired
    private RecommendationService recommendationService;

    @Test
    void shouldReturnFallbackWhenMLServiceDown() {
        // Arrange
        when(mlClient.getRecommendations(any()))
                .thenThrow(new FeignException.ServiceUnavailable("ML service down", 
                        null, null, null));
        
        when(featureStore.getUserFeatures(1L))
                .thenReturn(UserFeatures.builder().userId(1L).build());

        // Act
        List<ProductDto> recommendations = recommendationService
                .getPersonalizedRecommendations(1L, 10);

        // Assert - ต้องได้ fallback recommendations
        assertThat(recommendations).isNotEmpty();
        assertThat(recommendations.size()).isLessThanOrEqualTo(10);
    }

    @Test
    void shouldReturnCachedRecommendations() {
        // Test caching behavior
    }
}
```

### Model Performance Monitoring

```java
// ModelMonitoringService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class ModelMonitoringService {

    private final MeterRegistry meterRegistry;

    // ติดตาม drift ของ model
    @Scheduled(cron = "0 0 * * * *") // ทุกชั่วโมง
    public void checkModelDrift() {
        // เปรียบเทียบ distribution ของ input features กับ training data
        // ถ้า drift สูงเกินไป แจ้งเตือนให้ retrain
    }

    public void recordPrediction(String modelName, String modelVersion,
                                  double latencyMs, boolean successful) {
        meterRegistry.timer("ml.model.latency",
                "model", modelName,
                "version", modelVersion)
                .record(latencyMs, TimeUnit.MILLISECONDS);

        meterRegistry.counter("ml.model.predictions",
                "model", modelName,
                "version", modelVersion,
                "status", successful ? "success" : "error")
                .increment();
    }
}
```

---

*[← Part 93: Cost Optimization](./part-93-cost-optimization.md) | [Part 95: Advanced Testing →](./part-95-advanced-testing.md)*
