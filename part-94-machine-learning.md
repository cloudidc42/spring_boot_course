# Part 94: Machine Learning Integration with Spring Boot
## ขั้นตอนที่ 3361-3400

**ระดับ:** World-Class (ระดับโลก)
**เวลาเรียน:** 8-10 ชั่วโมง
**เป้าหมาย:** เรียนรู้การ integrate Machine Learning กับ Spring Boot ครอบคลุม REST/gRPC calls ไป Python ML services, ONNX model inference ใน Java, Feature store patterns, A/B testing, Model versioning, Recommendation engine และ Real-time feature computation

---

## ขั้นตอนที่ 3361: Calling Python ML Services via REST/gRPC

### Architecture Overview

```
Spring Boot App → REST/gRPC → Python ML Service (FastAPI/Flask)
                            → ONNX Runtime (In-process)
                            → Feature Store → Online Features
```

### Python FastAPI ML Service

```python
# ml_service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import numpy as np
import joblib
from typing import List, Optional

app = FastAPI(title="ShopHub ML Service")

# Load models on startup
product_ranker = joblib.load("models/product_ranker_v2.pkl")
fraud_detector = joblib.load("models/fraud_detector_v1.pkl")

class RecommendationRequest(BaseModel):
    user_id: str
    context_product_ids: List[str] = []
    limit: int = 10

class RecommendationResponse(BaseModel):
    user_id: str
    recommended_product_ids: List[str]
    scores: List[float]
    model_version: str

@app.post("/recommendations", response_model=RecommendationResponse)
async def get_recommendations(request: RecommendationRequest):
    try:
        # ดึง user features จาก feature store
        user_features = get_user_features(request.user_id)
        
        # Score candidate products
        scores = product_ranker.predict([user_features])
        
        # Sort และ return top-N
        top_indices = np.argsort(scores[0])[-request.limit:][::-1]
        
        return RecommendationResponse(
            user_id=request.user_id,
            recommended_product_ids=[candidate_products[i] for i in top_indices],
            scores=[float(scores[0][i]) for i in top_indices],
            model_version="v2.1.0"
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/fraud-detection")
async def detect_fraud(transaction: dict):
    features = extract_transaction_features(transaction)
    probability = fraud_detector.predict_proba([features])[0][1]
    
    return {
        "is_fraud": probability > 0.7,
        "fraud_probability": float(probability),
        "risk_level": "HIGH" if probability > 0.7 else "MEDIUM" if probability > 0.3 else "LOW"
    }
```

### Spring Boot ML Client

```java
// ml/MlServiceClient.java
package com.example.shophub.ml;

import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

@Component
public class MlServiceClient {
    
    private final WebClient webClient;
    
    public MlServiceClient(WebClient.Builder webClientBuilder,
                            @Value("${ml.service.url}") String mlServiceUrl) {
        this.webClient = webClientBuilder
            .baseUrl(mlServiceUrl)
            .defaultHeader("Content-Type", "application/json")
            .build();
    }
    
    public Mono<RecommendationResponse> getRecommendations(String userId, 
                                                             List<String> contextProductIds,
                                                             int limit) {
        return webClient.post()
            .uri("/recommendations")
            .bodyValue(new RecommendationRequest(userId, contextProductIds, limit))
            .retrieve()
            .onStatus(HttpStatusCode::is5xxServerError, response ->
                Mono.error(new MlServiceException("ML service error: " + response.statusCode())))
            .bodyToMono(RecommendationResponse.class)
            .timeout(Duration.ofSeconds(2)) // ML inference timeout
            .doOnError(e -> log.error("ML service error for user {}: {}", userId, e.getMessage()));
    }
    
    public Mono<FraudDetectionResult> detectFraud(Transaction transaction) {
        return webClient.post()
            .uri("/fraud-detection")
            .bodyValue(transaction)
            .retrieve()
            .bodyToMono(FraudDetectionResult.class)
            .timeout(Duration.ofMillis(500)) // Fraud check ต้องเร็ว
            .onErrorReturn(FraudDetectionResult.defaultAllow()); // Fail-safe
    }
}
```

### gRPC Integration

```protobuf
// proto/ml_service.proto
syntax = "proto3";
package com.example.shophub.ml;

service RecommendationService {
    rpc GetRecommendations(RecommendationRequest) returns (RecommendationResponse);
    rpc StreamRecommendations(RecommendationRequest) returns (stream ProductScore);
}

message RecommendationRequest {
    string user_id = 1;
    repeated string context_product_ids = 2;
    int32 limit = 3;
    map<string, string> context = 4;  // Additional context
}

message RecommendationResponse {
    string user_id = 1;
    repeated ProductScore products = 2;
    string model_version = 3;
    int64 latency_ms = 4;
}

message ProductScore {
    string product_id = 1;
    float score = 2;
    repeated string reasons = 3;
}
```

```java
// ml/grpc/RecommendationGrpcClient.java
@Component
public class RecommendationGrpcClient {
    
    private final RecommendationServiceGrpc.RecommendationServiceBlockingStub blockingStub;
    private final RecommendationServiceGrpc.RecommendationServiceStub asyncStub;
    
    public RecommendationGrpcClient(@Value("${ml.grpc.host}") String host,
                                     @Value("${ml.grpc.port}") int port) {
        ManagedChannel channel = ManagedChannelBuilder.forAddress(host, port)
            .usePlaintext()
            .keepAliveTime(30, TimeUnit.SECONDS)
            .keepAliveTimeout(5, TimeUnit.SECONDS)
            .build();
        
        this.blockingStub = RecommendationServiceGrpc.newBlockingStub(channel)
            .withDeadlineAfter(2, TimeUnit.SECONDS);
        this.asyncStub = RecommendationServiceGrpc.newStub(channel);
    }
    
    public List<ProductScore> getRecommendations(String userId, int limit) {
        RecommendationRequest request = RecommendationRequest.newBuilder()
            .setUserId(userId)
            .setLimit(limit)
            .build();
        
        try {
            RecommendationResponse response = blockingStub.getRecommendations(request);
            return response.getProductsList();
        } catch (StatusRuntimeException e) {
            log.error("gRPC call failed: {}", e.getStatus());
            return Collections.emptyList();
        }
    }
}
```

---

## ขั้นตอนที่ 3362: ONNX Model Inference ใน Java

ONNX (Open Neural Network Exchange) ช่วยให้ run ML models ใน Java โดยตรง ไม่ต้องเรียก Python

### Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.microsoft.onnxruntime</groupId>
    <artifactId>onnxruntime</artifactId>
    <version>1.16.3</version>
</dependency>
```

### ONNX Inference Service

```java
// ml/onnx/OnnxInferenceService.java
package com.example.shophub.ml.onnx;

import ai.onnxruntime.*;
import org.springframework.stereotype.Service;

import java.nio.FloatBuffer;
import java.util.*;

@Service
public class OnnxInferenceService {
    
    private final OrtEnvironment environment;
    private final Map<String, OrtSession> sessions = new ConcurrentHashMap<>();
    
    public OnnxInferenceService() throws OrtException {
        this.environment = OrtEnvironment.getEnvironment();
    }
    
    @PostConstruct
    public void loadModels() throws OrtException {
        // Load fraud detection model
        loadModel("fraud_detector", "models/fraud_detector.onnx");
        
        // Load product ranking model
        loadModel("product_ranker", "models/product_ranker.onnx");
        
        log.info("โหลด ONNX models สำเร็จ: {} models", sessions.size());
    }
    
    private void loadModel(String name, String path) throws OrtException {
        OrtSession.SessionOptions options = new OrtSession.SessionOptions();
        options.setOptimizationLevel(OrtSession.SessionOptions.OptLevel.ALL_OPT);
        options.addCPU(false); // ใช้ CPU
        
        OrtSession session = environment.createSession(path, options);
        sessions.put(name, session);
        log.info("โหลด model '{}' สำเร็จ: input={}, output={}", 
            name, session.getInputNames(), session.getOutputNames());
    }
    
    public FraudPrediction predictFraud(TransactionFeatures features) throws OrtException {
        OrtSession session = sessions.get("fraud_detector");
        
        // สร้าง input tensor
        float[] featureArray = features.toFloatArray();
        long[] shape = {1, featureArray.length};
        
        OnnxTensor inputTensor = OnnxTensor.createTensor(
            environment, 
            FloatBuffer.wrap(featureArray), 
            shape
        );
        
        try (OrtSession.Result result = session.run(
                Map.of("input", inputTensor))) {
            
            float[] probabilities = (float[]) result.get("probabilities")
                .get().getValue();
            
            return FraudPrediction.builder()
                .fraudProbability(probabilities[1])
                .isFraud(probabilities[1] > 0.7f)
                .build();
        } finally {
            inputTensor.close();
        }
    }
    
    public List<Float> rankProducts(UserFeatures userFeatures, 
                                     List<ProductFeatures> products) throws OrtException {
        OrtSession session = sessions.get("product_ranker");
        
        int numProducts = products.size();
        int featureDim = products.get(0).getDimension();
        
        // สร้าง batch input tensor
        float[][] batchFeatures = new float[numProducts][featureDim];
        for (int i = 0; i < numProducts; i++) {
            batchFeatures[i] = products.get(i).toFloatArray(userFeatures);
        }
        
        long[] shape = {numProducts, featureDim};
        OnnxTensor inputTensor = OnnxTensor.createTensor(environment, batchFeatures);
        
        try (OrtSession.Result result = session.run(Map.of("features", inputTensor))) {
            float[] scores = (float[]) result.get("scores").get().getValue();
            
            List<Float> scoreList = new ArrayList<>();
            for (float score : scores) {
                scoreList.add(score);
            }
            return scoreList;
        } finally {
            inputTensor.close();
        }
    }
    
    @PreDestroy
    public void cleanup() throws OrtException {
        sessions.values().forEach(session -> {
            try { session.close(); } catch (OrtException e) { /* ignore */ }
        });
        environment.close();
    }
}
```

### Feature Extraction

```java
// ml/features/TransactionFeatures.java
public class TransactionFeatures {
    
    private final float amount;
    private final float hourOfDay;
    private final float dayOfWeek;
    private final float merchantCategoryCode;
    private final float isInternational;
    private final float velocityScore;   // ความถี่การใช้งาน
    private final float locationRiskScore;
    
    public float[] toFloatArray() {
        return new float[]{
            amount,
            hourOfDay / 24.0f,           // Normalize
            dayOfWeek / 7.0f,
            merchantCategoryCode / 9999.0f,
            isInternational,
            velocityScore,
            locationRiskScore
        };
    }
    
    public static TransactionFeatures from(Transaction transaction, UserProfile user) {
        return TransactionFeatures.builder()
            .amount((float) transaction.getAmount().doubleValue())
            .hourOfDay(transaction.getTimestamp().getHour())
            .dayOfWeek(transaction.getTimestamp().getDayOfWeek().getValue())
            .merchantCategoryCode(transaction.getMerchantCategoryCode())
            .isInternational(transaction.isInternational() ? 1.0f : 0.0f)
            .velocityScore(calculateVelocityScore(user, transaction))
            .locationRiskScore(calculateLocationRisk(transaction.getLocation(), user))
            .build();
    }
}
```

---

## ขั้นตอนที่ 3363: Feature Store Pattern

Feature store แยก feature computation ออกจาก model training และ serving

### Feature Store Architecture

```
Offline Feature Store (S3/HDFS):
- Historical features สำหรับ training
- Batch computed daily/weekly

Online Feature Store (Redis):
- Real-time features สำหรับ serving
- Low latency (<5ms)
- Updated in real-time
```

```java
// featurestore/FeatureStore.java
public interface FeatureStore {
    Map<String, Object> getFeatures(String entityId, List<String> featureNames);
    void putFeatures(String entityId, Map<String, Object> features);
    void putFeaturesAsync(String entityId, Map<String, Object> features);
}

// featurestore/RedisFeatureStore.java
@Service
@Primary
public class RedisFeatureStore implements FeatureStore {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final ObjectMapper objectMapper;
    
    private static final String KEY_PREFIX = "features:";
    private static final Duration DEFAULT_TTL = Duration.ofHours(24);
    
    @Override
    public Map<String, Object> getFeatures(String entityId, List<String> featureNames) {
        String key = KEY_PREFIX + entityId;
        
        // Get specific fields from Redis Hash
        List<Object> values = redisTemplate.opsForHash()
            .multiGet(key, featureNames.stream()
                .map(Object.class::cast)
                .collect(Collectors.toList()));
        
        Map<String, Object> features = new HashMap<>();
        for (int i = 0; i < featureNames.size(); i++) {
            if (values.get(i) != null) {
                features.put(featureNames.get(i), values.get(i));
            }
        }
        
        return features;
    }
    
    @Override
    public void putFeatures(String entityId, Map<String, Object> features) {
        String key = KEY_PREFIX + entityId;
        
        redisTemplate.opsForHash().putAll(key, features);
        redisTemplate.expire(key, DEFAULT_TTL);
    }
    
    @Override
    @Async
    public void putFeaturesAsync(String entityId, Map<String, Object> features) {
        putFeatures(entityId, features);
    }
    
    // Batch get สำหรับ multiple entities
    public Map<String, Map<String, Object>> batchGetFeatures(
            List<String> entityIds, 
            List<String> featureNames) {
        
        // ใช้ Redis pipeline เพื่อ batch requests
        List<Object> results = redisTemplate.executePipelined(
            (RedisCallback<Object>) connection -> {
                entityIds.forEach(entityId -> {
                    byte[] key = (KEY_PREFIX + entityId).getBytes();
                    featureNames.forEach(featureName -> {
                        connection.hashCommands().hGet(key, featureName.getBytes());
                    });
                });
                return null;
            }
        );
        
        // Process pipeline results
        Map<String, Map<String, Object>> resultMap = new HashMap<>();
        int idx = 0;
        for (String entityId : entityIds) {
            Map<String, Object> features = new HashMap<>();
            for (String featureName : featureNames) {
                features.put(featureName, results.get(idx++));
            }
            resultMap.put(entityId, features);
        }
        
        return resultMap;
    }
}
```

### Real-time Feature Computation

```java
// featurestore/UserFeatureComputationService.java
@Service
public class UserFeatureComputationService {
    
    private final FeatureStore featureStore;
    private final OrderRepository orderRepository;
    
    @EventListener
    @Async
    public void onOrderPlaced(OrderPlacedEvent event) {
        // อัปเดต user features เมื่อมีคำสั่งซื้อใหม่
        updateUserPurchaseFeatures(event.getCustomerId());
    }
    
    @EventListener
    @Async
    public void onProductViewed(ProductViewedEvent event) {
        // อัปเดต user browsing features
        updateUserBrowsingFeatures(event.getUserId(), event.getProductId());
    }
    
    private void updateUserPurchaseFeatures(String userId) {
        LocalDateTime thirtyDaysAgo = LocalDateTime.now().minusDays(30);
        
        // คำนวณ features จาก recent orders
        List<Order> recentOrders = orderRepository
            .findByCustomerIdAndCreatedAtAfter(userId, thirtyDaysAgo);
        
        Map<String, Object> features = Map.of(
            "purchase_count_30d", recentOrders.size(),
            "avg_order_value", calculateAvgOrderValue(recentOrders),
            "preferred_categories", getPreferredCategories(recentOrders),
            "last_purchase_days_ago", getDaysSinceLastPurchase(recentOrders),
            "is_frequent_buyer", recentOrders.size() >= 3
        );
        
        featureStore.putFeaturesAsync(userId, features);
    }
    
    private void updateUserBrowsingFeatures(String userId, String productId) {
        // Increment browse count สำหรับ category ของ product นี้
        String category = productRepository.findCategoryById(productId);
        
        String key = "browse_count_" + category;
        Map<String, Object> features = featureStore.getFeatures(userId, List.of(key));
        
        int currentCount = (int) features.getOrDefault(key, 0);
        featureStore.putFeatures(userId, Map.of(key, currentCount + 1));
    }
}
```

---

## ขั้นตอนที่ 3364: A/B Testing Framework สำหรับ ML Models

```java
// abtest/AbTestService.java
@Service
public class AbTestService {
    
    private final ExperimentRepository experimentRepository;
    private final FeatureStore featureStore;
    private final MeterRegistry meterRegistry;
    
    public <T> T runExperiment(String experimentName, String userId, 
                                Function<String, T> controlFunction,
                                Function<String, T> treatmentFunction) {
        
        Experiment experiment = experimentRepository.findActiveByName(experimentName)
            .orElseThrow(() -> new ExperimentNotFoundException(experimentName));
        
        // กำหนด variant สำหรับ user นี้
        String variant = assignVariant(userId, experiment);
        
        // Track assignment
        trackAssignment(experimentName, userId, variant);
        
        // Execute variant
        T result;
        long startTime = System.currentTimeMillis();
        
        try {
            result = switch (variant) {
                case "control" -> controlFunction.apply(userId);
                case "treatment" -> treatmentFunction.apply(userId);
                default -> controlFunction.apply(userId);
            };
            
            long latency = System.currentTimeMillis() - startTime;
            recordMetrics(experimentName, variant, "success", latency);
            
            return result;
        } catch (Exception e) {
            recordMetrics(experimentName, variant, "error", 0);
            throw e;
        }
    }
    
    private String assignVariant(String userId, Experiment experiment) {
        // Consistent assignment ด้วย hashing
        int hash = Math.abs((userId + experiment.getName()).hashCode());
        double percentage = (hash % 100) / 100.0;
        
        double cumulativeWeight = 0;
        for (ExperimentVariant variant : experiment.getVariants()) {
            cumulativeWeight += variant.getWeight();
            if (percentage < cumulativeWeight) {
                return variant.getName();
            }
        }
        
        return "control";
    }
    
    private void trackAssignment(String experiment, String userId, String variant) {
        featureStore.putFeaturesAsync(
            "experiment:" + experiment + ":" + userId,
            Map.of("variant", variant, "assigned_at", System.currentTimeMillis())
        );
        
        meterRegistry.counter("experiment.assignment",
            "experiment", experiment,
            "variant", variant).increment();
    }
}

// abtest/RecommendationAbTest.java
@Service
public class RecommendationAbTest {
    
    private final AbTestService abTestService;
    private final MlServiceClient mlServiceClient;
    private final CollaborativeFilteringService cfService;
    
    public List<String> getRecommendations(String userId) {
        return abTestService.runExperiment(
            "recommendation_model_v2",
            userId,
            // Control: old collaborative filtering
            uid -> cfService.getRecommendations(uid, 10),
            // Treatment: new ML model
            uid -> mlServiceClient.getRecommendations(uid, List.of(), 10)
                .map(r -> r.getRecommendedProductIds())
                .block(Duration.ofSeconds(2))
        );
    }
}
```

---

## ขั้นตอนที่ 3365: Model Versioning และ Gradual Rollout

```java
// ml/ModelVersioningService.java
@Service
public class ModelVersioningService {
    
    private final Map<String, ModelVersion> activeVersions = new ConcurrentHashMap<>();
    private final FeatureStore featureStore;
    
    @PostConstruct
    public void loadActiveVersions() {
        // โหลด model versions จาก config/database
        activeVersions.put("recommendation", new ModelVersion("v2.1.0", 0.1)); // 10% traffic
        activeVersions.put("fraud_detection", new ModelVersion("v1.0.0", 1.0)); // 100% traffic
    }
    
    public String getModelVersion(String modelName, String userId) {
        ModelVersion version = activeVersions.get(modelName);
        if (version == null) return "default";
        
        // Check if user is in rollout group
        double hash = Math.abs(userId.hashCode() % 100) / 100.0;
        
        if (hash < version.getRolloutPercentage()) {
            return version.getVersion();
        }
        
        return version.getPreviousVersion();
    }
    
    // Gradual rollout
    @Scheduled(cron = "0 0 */6 * * ?") // ทุก 6 ชั่วโมง
    public void increaseRollout() {
        activeVersions.forEach((name, version) -> {
            if (version.getErrorRate() < 0.01 && version.getRolloutPercentage() < 1.0) {
                double newPercentage = Math.min(version.getRolloutPercentage() + 0.1, 1.0);
                version.setRolloutPercentage(newPercentage);
                log.info("เพิ่ม rollout สำหรับ {} เป็น {}%", 
                    name, (int)(newPercentage * 100));
            }
        });
    }
    
    // Automatic rollback
    @Scheduled(fixedRate = 60000)
    public void checkModelHealth() {
        activeVersions.forEach((name, version) -> {
            if (version.getErrorRate() > 0.05) { // Error > 5%
                log.error("Model {} มี error rate สูง {}% - กำลัง rollback!",
                    name, version.getErrorRate() * 100);
                version.setRolloutPercentage(0.0); // Rollback ทันที
                alertService.sendModelAlert(name, version);
            }
        });
    }
}
```

---

## ขั้นตอนที่ 3366: Recommendation Engine

### Collaborative Filtering Service

```java
// recommendation/CollaborativeFilteringService.java
@Service
public class CollaborativeFilteringService {
    
    private final UserProductInteractionRepository interactionRepository;
    private final FeatureStore featureStore;
    private final MlServiceClient mlServiceClient;
    
    public List<String> getRecommendations(String userId, int limit) {
        // ตรวจสอบ cached recommendations ก่อน
        Map<String, Object> cached = featureStore.getFeatures(
            userId, 
            List.of("recommendations", "recommendations_updated_at")
        );
        
        if (cached.get("recommendations") != null) {
            long updatedAt = (long) cached.getOrDefault("recommendations_updated_at", 0L);
            if (System.currentTimeMillis() - updatedAt < 3_600_000) { // 1 ชั่วโมง
                return (List<String>) cached.get("recommendations");
            }
        }
        
        // คำนวณ recommendations ใหม่
        return computeRecommendations(userId, limit);
    }
    
    private List<String> computeRecommendations(String userId, int limit) {
        // ดู user interactions
        List<UserProductInteraction> interactions = 
            interactionRepository.findByUserIdOrderByTimestampDesc(userId, 
                PageRequest.of(0, 1000));
        
        if (interactions.isEmpty()) {
            return getPopularProducts(limit); // Cold start - ใช้ popular products
        }
        
        // Find similar users (User-based CF)
        List<String> interactedProducts = interactions.stream()
            .map(UserProductInteraction::getProductId)
            .collect(Collectors.toList());
        
        List<String> similarUsers = findSimilarUsers(userId, interactedProducts);
        
        // ดูสินค้าที่ similar users ชอบ แต่ current user ยังไม่เคยดู
        Set<String> interactedSet = new HashSet<>(interactedProducts);
        
        return similarUsers.stream()
            .flatMap(similarUserId -> 
                interactionRepository.findRecentProductsByUserId(similarUserId, 50)
                    .stream()
                    .map(UserProductInteraction::getProductId))
            .filter(productId -> !interactedSet.contains(productId))
            .distinct()
            .limit(limit)
            .collect(Collectors.toList());
    }
    
    @Async
    public void recordInteraction(String userId, String productId, InteractionType type) {
        UserProductInteraction interaction = UserProductInteraction.builder()
            .userId(userId)
            .productId(productId)
            .type(type)
            .timestamp(LocalDateTime.now())
            .weight(type.getWeight())
            .build();
        
        interactionRepository.save(interaction);
        
        // อัปเดต real-time features
        featureStore.putFeaturesAsync(userId, Map.of(
            "last_viewed_product", productId,
            "last_interaction_at", System.currentTimeMillis()
        ));
    }
    
    private List<String> findSimilarUsers(String userId, List<String> products) {
        // หา users ที่ interact กับ products เดียวกัน
        return interactionRepository.findUsersByProducts(products, userId, 20);
    }
    
    private List<String> getPopularProducts(int limit) {
        return interactionRepository.findMostInteractedProducts(
            LocalDateTime.now().minusDays(7), limit);
    }
}
```

### Recommendation API

```java
// controller/RecommendationController.java
@RestController
@RequestMapping("/api/v1/recommendations")
public class RecommendationController {
    
    private final CollaborativeFilteringService cfService;
    private final MlServiceClient mlServiceClient;
    private final AbTestService abTestService;
    private final ProductService productService;
    
    @GetMapping("/for-you")
    public ResponseEntity<RecommendationResponse> getPersonalizedRecommendations(
            @AuthenticationPrincipal UserDetails user,
            @RequestParam(defaultValue = "10") int limit) {
        
        List<String> productIds = abTestService.runExperiment(
            "recommendation_algorithm",
            user.getUsername(),
            userId -> cfService.getRecommendations(userId, limit),
            userId -> mlServiceClient.getRecommendations(userId, List.of(), limit)
                .map(RecommendationResponse::getRecommendedProductIds)
                .onErrorReturn(cfService.getRecommendations(userId, limit)) // fallback
                .block(Duration.ofSeconds(2))
        );
        
        // Enrich ด้วยข้อมูล product
        List<ProductDto> products = productService.findByIds(productIds);
        
        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(Duration.ofMinutes(5)))
            .body(RecommendationResponse.builder()
                .products(products)
                .algorithm(abTestService.getAssignedVariant(user.getUsername(), "recommendation_algorithm"))
                .build());
    }
    
    @PostMapping("/track")
    public ResponseEntity<Void> trackInteraction(
            @AuthenticationPrincipal UserDetails user,
            @RequestBody TrackInteractionRequest request) {
        
        cfService.recordInteraction(
            user.getUsername(),
            request.getProductId(),
            request.getInteractionType()
        );
        
        return ResponseEntity.accepted().build();
    }
}
```

---

## ขั้นตอนที่ 3367-3400: Real-time ML Pipeline

### Kafka ML Pipeline

```java
// pipeline/MlFeatureEnrichmentPipeline.java
@Component
public class MlFeatureEnrichmentPipeline {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final FeatureStore featureStore;
    private final OnnxInferenceService onnxInference;
    
    @KafkaListener(topics = "order-events", groupId = "ml-pipeline")
    public void processOrderEvent(OrderEventMessage event) {
        // 1. Compute real-time features
        Map<String, Object> features = computeRealTimeFeatures(event);
        
        // 2. Update feature store
        featureStore.putFeaturesAsync(event.getUserId(), features);
        
        // 3. Run fraud detection (real-time)
        if (event.getType() == OrderEventType.PAYMENT_INITIATED) {
            TransactionFeatures txFeatures = TransactionFeatures.from(event);
            
            try {
                FraudPrediction prediction = onnxInference.predictFraud(txFeatures);
                
                if (prediction.isFraud()) {
                    // ส่ง fraud alert
                    kafkaTemplate.send("fraud-alerts", event.getOrderId(), 
                        FraudAlert.builder()
                            .orderId(event.getOrderId())
                            .userId(event.getUserId())
                            .probability(prediction.getFraudProbability())
                            .build());
                }
            } catch (OrtException e) {
                log.error("Fraud detection failed: {}", e.getMessage());
                // Fail open - allow transaction
            }
        }
    }
    
    private Map<String, Object> computeRealTimeFeatures(OrderEventMessage event) {
        return Map.of(
            "orders_today", countOrdersToday(event.getUserId()),
            "order_amount", event.getAmount(),
            "hour_of_day", LocalDateTime.now().getHour(),
            "is_weekend", LocalDate.now().getDayOfWeek().getValue() > 5
        );
    }
}
```

### Model Performance Monitoring

```java
// monitoring/ModelMonitoringService.java
@Service
public class ModelMonitoringService {
    
    private final MeterRegistry meterRegistry;
    private final FeatureStore featureStore;
    
    public void recordPrediction(String modelName, String modelVersion,
                                  Object input, Object prediction, long latencyMs) {
        // Record prediction metrics
        meterRegistry.counter("ml.predictions",
            "model", modelName,
            "version", modelVersion).increment();
        
        meterRegistry.timer("ml.inference.latency",
            "model", modelName,
            "version", modelVersion)
            .record(latencyMs, TimeUnit.MILLISECONDS);
        
        // Store prediction for offline analysis
        featureStore.putFeaturesAsync(
            "prediction:" + UUID.randomUUID(),
            Map.of(
                "model", modelName,
                "version", modelVersion,
                "timestamp", System.currentTimeMillis(),
                "latency_ms", latencyMs
            )
        );
    }
    
    public void recordFeedback(String predictionId, boolean wasCorrect) {
        // Track prediction accuracy
        meterRegistry.counter("ml.feedback",
            "correct", String.valueOf(wasCorrect)).increment();
    }
    
    @Scheduled(cron = "0 0 * * * ?")
    public void computeModelMetrics() {
        // คำนวณ model performance metrics รายชั่วโมง
        Map<String, ModelMetrics> metrics = computeHourlyMetrics();
        
        metrics.forEach((model, m) -> {
            log.info("Model {}: accuracy={:.2f}%, avg_latency={}ms, predictions={}",
                model, m.getAccuracy() * 100, m.getAvgLatency(), m.getPredictionCount());
            
            if (m.getAccuracy() < 0.8) {
                log.warn("Model {} accuracy ต่ำกว่า 80% - อาจต้อง retrain", model);
            }
        });
    }
}
```

---

## สรุป

Part 94 ครอบคลุม Machine Learning Integration:

1. **REST/gRPC** - เรียก Python ML services ด้วย WebClient/gRPC
2. **ONNX Runtime** - run models ใน Java โดยตรง ไม่ต้อง Python
3. **Feature Store** - Redis สำหรับ real-time features
4. **A/B Testing** - ทดสอบ models แบบ controlled
5. **Model Versioning** - gradual rollout + automatic rollback
6. **Collaborative Filtering** - recommendation engine
7. **Real-time Pipeline** - Kafka + streaming features

ML integration ที่ดีต้องมี fallback เสมอ - ถ้า ML service ล้มเหลว ต้องยัง serve users ได้

---

*[← Part 93: Cost Optimization](./part-93-cost-optimization.md) | [Part 95: Advanced Testing →](./part-95-advanced-testing.md)*
