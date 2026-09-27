# Part 76: Search with Elasticsearch
## ขั้นตอนที่ 2641-2680

**ระดับ:** ระดับสูง (Advanced)
**เวลาเรียน:** 5-6 ชั่วโมง
**เป้าหมาย:** เรียนรู้การใช้ Elasticsearch ร่วมกับ Spring Data Elasticsearch เพื่อสร้างระบบค้นหาที่มีประสิทธิภาพสูง รองรับ Full-text search, Fuzzy search, Aggregation และ Auto-complete

---

## ขั้นตอนที่ 2641: แนะนำ Elasticsearch และ Spring Data Elasticsearch

Elasticsearch คือ distributed search engine ที่สร้างบน Apache Lucene มีความสามารถสูงในการค้นหาข้อมูลแบบ full-text search และรองรับข้อมูลปริมาณมาก Spring Data Elasticsearch ช่วยให้เราทำงานกับ Elasticsearch ได้ง่ายขึ้นผ่าน Repository pattern

### โครงสร้างพื้นฐาน

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>co.elastic.clients</groupId>
        <artifactId>elasticsearch-java</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
spring:
  elasticsearch:
    uris: http://localhost:9200
    username: elastic
    password: changeme
    connection-timeout: 5s
    socket-timeout: 30s

  data:
    elasticsearch:
      repositories:
        enabled: true
```

---

## ขั้นตอนที่ 2642: สร้าง Elasticsearch Document และ Index Mapping

การสร้าง Document ใน Elasticsearch ต้องกำหนด Index name และ Field mapping ให้ถูกต้อง

```java
// model/ProductDocument.java
package com.example.search.model;

import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Document(indexName = "products", createIndex = true)
@Setting(settingPath = "/elasticsearch/product-settings.json")
@Mapping(mappingPath = "/elasticsearch/product-mapping.json")
public class ProductDocument {

    @Id
    private String id;

    @Field(type = FieldType.Text, analyzer = "thai_analyzer")
    private String name;

    @Field(type = FieldType.Text, analyzer = "thai_analyzer")
    @MultiField(
        mainField = @Field(type = FieldType.Text, analyzer = "thai_analyzer"),
        otherFields = {
            @InnerField(suffix = "keyword", type = FieldType.Keyword),
            @InnerField(suffix = "suggest", type = FieldType.Search_As_You_Type)
        }
    )
    private String description;

    @Field(type = FieldType.Keyword)
    private String sku;

    @Field(type = FieldType.Double)
    private BigDecimal price;

    @Field(type = FieldType.Keyword)
    private String category;

    @Field(type = FieldType.Keyword)
    private List<String> tags;

    @Field(type = FieldType.Integer)
    private Integer stock;

    @Field(type = FieldType.Boolean)
    private boolean active;

    @Field(type = FieldType.Float)
    private Float rating;

    @Field(type = FieldType.Integer)
    private Integer reviewCount;

    @Field(type = FieldType.Date, format = DateFormat.date_hour_minute_second)
    private LocalDateTime createdAt;

    @Field(type = FieldType.Date, format = DateFormat.date_hour_minute_second)
    private LocalDateTime updatedAt;

    // Nested object สำหรับ brand info
    @Field(type = FieldType.Object)
    private BrandInfo brand;

    // getters/setters/constructors
    public static class BrandInfo {
        @Field(type = FieldType.Keyword)
        private String id;

        @Field(type = FieldType.Text, analyzer = "standard")
        private String name;

        @Field(type = FieldType.Keyword)
        private String country;

        // getters/setters
    }
}
```

```json
// resources/elasticsearch/product-settings.json
{
  "number_of_shards": 3,
  "number_of_replicas": 1,
  "analysis": {
    "analyzer": {
      "thai_analyzer": {
        "type": "custom",
        "tokenizer": "thai",
        "filter": ["lowercase", "thai_stop"]
      },
      "autocomplete_analyzer": {
        "type": "custom",
        "tokenizer": "autocomplete_tokenizer",
        "filter": ["lowercase"]
      },
      "autocomplete_search_analyzer": {
        "type": "custom",
        "tokenizer": "standard",
        "filter": ["lowercase"]
      }
    },
    "tokenizer": {
      "autocomplete_tokenizer": {
        "type": "edge_ngram",
        "min_gram": 2,
        "max_gram": 15,
        "token_chars": ["letter", "digit"]
      }
    },
    "filter": {
      "thai_stop": {
        "type": "stop",
        "stopwords": "_thai_"
      }
    }
  }
}
```

```json
// resources/elasticsearch/product-mapping.json
{
  "properties": {
    "name": {
      "type": "text",
      "analyzer": "thai_analyzer",
      "fields": {
        "keyword": { "type": "keyword" },
        "autocomplete": {
          "type": "text",
          "analyzer": "autocomplete_analyzer",
          "search_analyzer": "autocomplete_search_analyzer"
        }
      }
    },
    "description": {
      "type": "text",
      "analyzer": "thai_analyzer"
    },
    "category": { "type": "keyword" },
    "tags": { "type": "keyword" },
    "price": { "type": "double" },
    "rating": { "type": "float" },
    "reviewCount": { "type": "integer" },
    "active": { "type": "boolean" }
  }
}
```

---

## ขั้นตอนที่ 2643: สร้าง Elasticsearch Repository

```java
// repository/ProductSearchRepository.java
package com.example.search.repository;

import com.example.search.model.ProductDocument;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;

import java.math.BigDecimal;
import java.util.List;

public interface ProductSearchRepository
        extends ElasticsearchRepository<ProductDocument, String> {

    // Spring Data method queries
    List<ProductDocument> findByCategory(String category);

    Page<ProductDocument> findByNameContaining(String name, Pageable pageable);

    List<ProductDocument> findByPriceBetween(BigDecimal minPrice, BigDecimal maxPrice);

    List<ProductDocument> findByActiveTrue();

    List<ProductDocument> findByTagsContaining(String tag);

    Page<ProductDocument> findByBrandName(String brandName, Pageable pageable);

    long countByCategory(String category);
}
```

---

## ขั้นตอนที่ 2644: Full-Text Search พื้นฐาน

```java
// service/ProductSearchService.java
package com.example.search.service;

import co.elastic.clients.elasticsearch.ElasticsearchClient;
import co.elastic.clients.elasticsearch._types.query_dsl.*;
import co.elastic.clients.elasticsearch.core.*;
import co.elastic.clients.elasticsearch.core.search.Hit;
import com.example.search.dto.*;
import com.example.search.model.ProductDocument;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageImpl;
import org.springframework.data.domain.Pageable;
import org.springframework.data.elasticsearch.core.*;
import org.springframework.data.elasticsearch.core.query.*;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.util.List;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final ElasticsearchOperations elasticsearchOperations;
    private final ElasticsearchClient elasticsearchClient;

    // Full-text search แบบง่าย
    public SearchHits<ProductDocument> simpleSearch(String keyword, Pageable pageable) {
        Query query = NativeQuery.builder()
            .withQuery(q -> q
                .multiMatch(mm -> mm
                    .query(keyword)
                    .fields("name^3", "description^1", "tags^2")
                    .type(TextQueryType.BestFields)
                    .fuzziness("AUTO")
                )
            )
            .withPageable(pageable)
            .build();

        return elasticsearchOperations.search(query, ProductDocument.class);
    }

    // Boolean query สำหรับ filter + search รวมกัน
    public Page<ProductDocument> advancedSearch(ProductSearchRequest request, Pageable pageable) {
        NativeQueryBuilder queryBuilder = NativeQuery.builder();

        BoolQuery.Builder boolQuery = new BoolQuery.Builder();

        // Must: full-text search
        if (request.getKeyword() != null && !request.getKeyword().isBlank()) {
            boolQuery.must(m -> m
                .multiMatch(mm -> mm
                    .query(request.getKeyword())
                    .fields("name^3", "description^1", "tags^2")
                    .fuzziness("AUTO")
                )
            );
        }

        // Filter: category
        if (request.getCategory() != null) {
            boolQuery.filter(f -> f
                .term(t -> t
                    .field("category")
                    .value(request.getCategory())
                )
            );
        }

        // Filter: price range
        if (request.getMinPrice() != null || request.getMaxPrice() != null) {
            boolQuery.filter(f -> f
                .range(r -> {
                    var rb = r.field("price");
                    if (request.getMinPrice() != null) rb = rb.gte(JsonData.of(request.getMinPrice()));
                    if (request.getMaxPrice() != null) rb = rb.lte(JsonData.of(request.getMaxPrice()));
                    return rb;
                })
            );
        }

        // Filter: tags
        if (request.getTags() != null && !request.getTags().isEmpty()) {
            boolQuery.filter(f -> f
                .terms(t -> t
                    .field("tags")
                    .terms(tv -> tv.value(
                        request.getTags().stream()
                            .map(FieldValue::of)
                            .collect(Collectors.toList())
                    ))
                )
            );
        }

        // Filter: active only
        boolQuery.filter(f -> f
            .term(t -> t.field("active").value(true))
        );

        // Minimum rating
        if (request.getMinRating() != null) {
            boolQuery.filter(f -> f
                .range(r -> r
                    .field("rating")
                    .gte(JsonData.of(request.getMinRating()))
                )
            );
        }

        NativeQuery nativeQuery = queryBuilder
            .withQuery(q -> q.bool(boolQuery.build()))
            .withPageable(pageable)
            .withSort(Sort.by(Sort.Direction.DESC, "_score"))
            .build();

        SearchHits<ProductDocument> hits = elasticsearchOperations.search(nativeQuery, ProductDocument.class);
        List<ProductDocument> content = hits.stream()
            .map(SearchHit::getContent)
            .collect(Collectors.toList());

        return new PageImpl<>(content, pageable, hits.getTotalHits());
    }
}
```

---

## ขั้นตอนที่ 2645: Fuzzy Search และ Multi-Match

Fuzzy search ช่วยให้ระบบค้นหาได้แม้จะพิมพ์ผิดเล็กน้อย

```java
// service/FuzzySearchService.java
package com.example.search.service;

import com.example.search.model.ProductDocument;
import lombok.RequiredArgsConstructor;
import org.springframework.data.elasticsearch.core.*;
import org.springframework.data.elasticsearch.core.query.NativeQuery;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class FuzzySearchService {

    private final ElasticsearchOperations elasticsearchOperations;

    // Fuzzy search - ทนต่อการพิมพ์ผิด
    public List<ProductDocument> fuzzySearch(String keyword) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q
                .fuzzy(f -> f
                    .field("name")
                    .value(keyword)
                    .fuzziness("2")           // Levenshtein distance สูงสุด
                    .prefixLength(2)           // จำนวน character ต้นที่ต้องตรงกัน
                    .maxExpansions(50)         // จำนวน term สูงสุดที่ expand ได้
                    .transpositions(true)      // อนุญาต transposition (ab -> ba)
                )
            )
            .build();

        return elasticsearchOperations.search(query, ProductDocument.class)
            .stream()
            .map(SearchHit::getContent)
            .collect(Collectors.toList());
    }

    // Multi-match กับ different types
    public List<ProductDocument> multiMatchSearch(String keyword) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q
                .multiMatch(mm -> mm
                    .query(keyword)
                    .fields("name^4", "description^2", "brand.name^3", "category^1", "tags^2")
                    .type(TextQueryType.CrossFields)  // match ข้ามหลาย field
                    .operator(Operator.And)
                    .minimumShouldMatch("75%")
                )
            )
            .build();

        return elasticsearchOperations.search(query, ProductDocument.class)
            .stream()
            .map(SearchHit::getContent)
            .collect(Collectors.toList());
    }

    // Phrase search - ค้นหาประโยคที่ต่อกัน
    public List<ProductDocument> phraseSearch(String phrase) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q
                .matchPhrase(mp -> mp
                    .field("description")
                    .query(phrase)
                    .slop(2)  // คำสามารถห่างกันได้ 2 คำ
                )
            )
            .build();

        return elasticsearchOperations.search(query, ProductDocument.class)
            .stream()
            .map(SearchHit::getContent)
            .collect(Collectors.toList());
    }

    // Highlight search results
    public SearchHits<ProductDocument> searchWithHighlight(String keyword) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q
                .multiMatch(mm -> mm
                    .query(keyword)
                    .fields("name", "description")
                )
            )
            .withHighlightQuery(new HighlightQuery(
                Highlight.builder()
                    .fields(
                        HighlightField.of(hf -> hf
                            .name("name")
                            .preTags("<em>")
                            .postTags("</em>")
                        ),
                        HighlightField.of(hf -> hf
                            .name("description")
                            .preTags("<em>")
                            .postTags("</em>")
                            .numberOfFragments(3)
                            .fragmentSize(150)
                        )
                    )
                    .build(),
                ProductDocument.class
            ))
            .build();

        return elasticsearchOperations.search(query, ProductDocument.class);
    }
}
```

---

## ขั้นตอนที่ 2646: Aggregations และ Faceted Search

Aggregation ใช้สำหรับสร้าง facets (ตัวกรองที่แสดงจำนวน) เหมือนกับ e-commerce ทั่วไป

```java
// service/AggregationSearchService.java
package com.example.search.service;

import co.elastic.clients.elasticsearch.ElasticsearchClient;
import co.elastic.clients.elasticsearch._types.aggregations.*;
import co.elastic.clients.elasticsearch.core.SearchResponse;
import com.example.search.dto.FacetedSearchResult;
import com.example.search.model.ProductDocument;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.util.*;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
public class AggregationSearchService {

    private final ElasticsearchClient elasticsearchClient;

    public FacetedSearchResult facetedSearch(String keyword, int page, int size) throws IOException {

        SearchResponse<ProductDocument> response = elasticsearchClient.search(s -> s
            .index("products")
            .query(q -> q
                .bool(b -> b
                    .must(m -> keyword != null ? m.multiMatch(mm -> mm
                        .query(keyword)
                        .fields("name^3", "description^1")
                    ) : m.matchAll(ma -> ma))
                    .filter(f -> f.term(t -> t.field("active").value(true)))
                )
            )
            .from(page * size)
            .size(size)
            .aggregations("categories", a -> a
                .terms(t -> t.field("category").size(20))
            )
            .aggregations("price_ranges", a -> a
                .range(r -> r
                    .field("price")
                    .ranges(
                        rng -> rng.to(500.0).key("under_500"),
                        rng -> rng.from(500.0).to(1000.0).key("500_1000"),
                        rng -> rng.from(1000.0).to(5000.0).key("1000_5000"),
                        rng -> rng.from(5000.0).key("over_5000")
                    )
                )
            )
            .aggregations("avg_price", a -> a
                .avg(avg -> avg.field("price"))
            )
            .aggregations("rating_histogram", a -> a
                .histogram(h -> h
                    .field("rating")
                    .interval(1.0)
                    .minDocCount(1)
                )
            )
            .aggregations("top_brands", a -> a
                .terms(t -> t
                    .field("brand.name.keyword")
                    .size(10)
                    .order(NamedValue.of("_count", SortOrder.Desc))
                )
            ),
            ProductDocument.class
        );

        return buildFacetedResult(response);
    }

    private FacetedSearchResult buildFacetedResult(SearchResponse<ProductDocument> response) {
        FacetedSearchResult result = new FacetedSearchResult();

        // Extract documents
        result.setProducts(response.hits().hits().stream()
            .map(h -> h.source())
            .filter(Objects::nonNull)
            .collect(Collectors.toList()));

        result.setTotal(response.hits().total().value());

        // Extract category facets
        StringTermsAggregate categories = response.aggregations()
            .get("categories").sterms();
        result.setCategoryFacets(categories.buckets().array().stream()
            .collect(Collectors.toMap(
                b -> b.key().stringValue(),
                b -> b.docCount()
            )));

        // Extract price range facets
        RangeAggregate priceRanges = response.aggregations()
            .get("price_ranges").range();
        Map<String, Long> priceFacets = new LinkedHashMap<>();
        priceRanges.buckets().array().forEach(b ->
            priceFacets.put(b.key(), b.docCount())
        );
        result.setPriceFacets(priceFacets);

        // Extract avg price
        AvgAggregate avgPrice = response.aggregations()
            .get("avg_price").avg();
        result.setAveragePrice(avgPrice.value());

        return result;
    }

    // Nested aggregation - ราคาเฉลี่ยต่อ category
    public Map<String, Double> avgPriceByCategory() throws IOException {
        SearchResponse<Void> response = elasticsearchClient.search(s -> s
            .index("products")
            .size(0)
            .aggregations("by_category", a -> a
                .terms(t -> t.field("category").size(50))
                .aggregations("avg_price", aa -> aa
                    .avg(avg -> avg.field("price"))
                )
            ),
            Void.class
        );

        StringTermsAggregate byCategory = response.aggregations()
            .get("by_category").sterms();

        return byCategory.buckets().array().stream()
            .collect(Collectors.toMap(
                b -> b.key().stringValue(),
                b -> b.aggregations().get("avg_price").avg().value()
            ));
    }
}
```

---

## ขั้นตอนที่ 2647: Auto-complete ด้วย Edge N-grams

```java
// service/AutoCompleteService.java
package com.example.search.service;

import com.example.search.model.ProductDocument;
import lombok.RequiredArgsConstructor;
import org.springframework.data.elasticsearch.core.*;
import org.springframework.data.elasticsearch.core.query.NativeQuery;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class AutoCompleteService {

    private final ElasticsearchOperations elasticsearchOperations;

    // Auto-complete ใช้ edge n-gram analyzer
    public List<String> autocomplete(String prefix) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q
                .match(m -> m
                    .field("name.autocomplete")
                    .query(prefix)
                    .analyzer("autocomplete_search_analyzer")
                )
            )
            .withSourceFilter(new FetchSourceFilterBuilder()
                .withIncludes("name")
                .build())
            .withPageable(org.springframework.data.domain.PageRequest.of(0, 10))
            .build();

        return elasticsearchOperations.search(query, ProductDocument.class)
            .stream()
            .map(hit -> hit.getContent().getName())
            .distinct()
            .collect(Collectors.toList());
    }

    // Completion suggester - เร็วกว่า edge n-gram
    public List<String> suggest(String prefix) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q
                .bool(b -> b
                    .should(
                        s -> s.prefix(p -> p.field("name").value(prefix).boost(2.0f)),
                        s -> s.match(m -> m.field("name.autocomplete").query(prefix))
                    )
                    .minimumShouldMatch("1")
                )
            )
            .withPageable(org.springframework.data.domain.PageRequest.of(0, 5))
            .build();

        return elasticsearchOperations.search(query, ProductDocument.class)
            .stream()
            .map(hit -> hit.getContent().getName())
            .collect(Collectors.toList());
    }

    // Search As You Type field type
    public List<ProductDocument> searchAsYouType(String text) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q
                .multiMatch(mm -> mm
                    .query(text)
                    .fields(
                        "description",
                        "description._2gram",
                        "description._3gram"
                    )
                    .type(TextQueryType.BoolPrefix)
                )
            )
            .withPageable(org.springframework.data.domain.PageRequest.of(0, 10))
            .build();

        return elasticsearchOperations.search(query, ProductDocument.class)
            .stream()
            .map(SearchHit::getContent)
            .collect(Collectors.toList());
    }
}
```

---

## ขั้นตอนที่ 2648: Sync ข้อมูลจาก PostgreSQL ไป Elasticsearch

การ sync ข้อมูลทำได้หลายวิธี เช่น event-driven, polling หรือ Change Data Capture (CDC)

```java
// service/DataSyncService.java
package com.example.search.service;

import com.example.search.entity.Product;
import com.example.search.mapper.ProductMapper;
import com.example.search.model.ProductDocument;
import com.example.search.repository.ProductRepository;
import com.example.search.repository.ProductSearchRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.scheduling.annotation.Async;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
public class DataSyncService {

    private final ProductRepository productRepository;
    private final ProductSearchRepository searchRepository;
    private final ElasticsearchOperations elasticsearchOperations;
    private final ProductMapper productMapper;

    // Sync เมื่อมีการ save product ใน database (event-driven)
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async
    public void onProductSaved(ProductSavedEvent event) {
        log.info("Syncing product {} to Elasticsearch", event.getProductId());
        productRepository.findById(event.getProductId())
            .ifPresent(product -> {
                ProductDocument doc = productMapper.toDocument(product);
                searchRepository.save(doc);
                log.info("Product {} synced successfully", event.getProductId());
            });
    }

    // Bulk sync ทั้งหมด (ใช้ตอน initial setup หรือ re-index)
    @Async
    public CompletableFuture<SyncResult> fullReindex() {
        log.info("Starting full reindex...");
        int page = 0;
        int batchSize = 500;
        long totalSynced = 0;
        long totalErrors = 0;

        // ลบ index เก่าแล้วสร้างใหม่
        if (elasticsearchOperations.indexOps(ProductDocument.class).exists()) {
            elasticsearchOperations.indexOps(ProductDocument.class).delete();
        }
        elasticsearchOperations.indexOps(ProductDocument.class).createWithMapping();

        Page<Product> productPage;
        do {
            productPage = productRepository.findAll(PageRequest.of(page, batchSize));
            try {
                List<ProductDocument> docs = productPage.getContent().stream()
                    .map(productMapper::toDocument)
                    .collect(Collectors.toList());

                searchRepository.saveAll(docs);
                totalSynced += docs.size();
                log.info("Synced batch {} ({} products)", page, docs.size());
            } catch (Exception e) {
                log.error("Error syncing batch {}: {}", page, e.getMessage());
                totalErrors += productPage.getNumberOfElements();
            }
            page++;
        } while (productPage.hasNext());

        SyncResult result = new SyncResult(totalSynced, totalErrors);
        log.info("Reindex complete: {} synced, {} errors", totalSynced, totalErrors);
        return CompletableFuture.completedFuture(result);
    }

    // Incremental sync ตาม schedule - sync ที่เปลี่ยนแปลงใน 5 นาทีล่าสุด
    @Scheduled(fixedDelay = 300_000) // 5 นาที
    public void incrementalSync() {
        log.debug("Running incremental sync...");
        // ดึงข้อมูลที่ update ล่าสุดใน 5 นาที
        // productRepository.findByUpdatedAtAfter(LocalDateTime.now().minusMinutes(5))
        //     .forEach(product -> searchRepository.save(productMapper.toDocument(product)));
    }
}
```

```java
// mapper/ProductMapper.java
package com.example.search.mapper;

import com.example.search.entity.Product;
import com.example.search.model.ProductDocument;
import org.springframework.stereotype.Component;

@Component
public class ProductMapper {

    public ProductDocument toDocument(Product product) {
        ProductDocument doc = new ProductDocument();
        doc.setId(product.getId().toString());
        doc.setName(product.getName());
        doc.setDescription(product.getDescription());
        doc.setSku(product.getSku());
        doc.setPrice(product.getPrice());
        doc.setCategory(product.getCategory().getName());
        doc.setTags(product.getTags());
        doc.setStock(product.getStock());
        doc.setActive(product.isActive());
        doc.setRating(product.getAverageRating());
        doc.setReviewCount(product.getReviewCount());
        doc.setCreatedAt(product.getCreatedAt());
        doc.setUpdatedAt(product.getUpdatedAt());

        if (product.getBrand() != null) {
            ProductDocument.BrandInfo brand = new ProductDocument.BrandInfo();
            brand.setId(product.getBrand().getId().toString());
            brand.setName(product.getBrand().getName());
            brand.setCountry(product.getBrand().getCountry());
            doc.setBrand(brand);
        }

        return doc;
    }
}
```

---

## ขั้นตอนที่ 2649: Search Relevance Tuning

การปรับ relevance ทำให้ผลการค้นหาตรงกับความต้องการผู้ใช้มากขึ้น

```java
// service/RelevanceTuningService.java
package com.example.search.service;

import com.example.search.model.ProductDocument;
import lombok.RequiredArgsConstructor;
import org.springframework.data.elasticsearch.core.*;
import org.springframework.data.elasticsearch.core.query.NativeQuery;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class RelevanceTuningService {

    private final ElasticsearchOperations elasticsearchOperations;

    // Function Score Query - ปรับ score ด้วย custom function
    public List<ProductDocument> searchWithFunctionScore(String keyword) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q
                .functionScore(fs -> fs
                    .query(inner -> inner
                        .multiMatch(mm -> mm
                            .query(keyword)
                            .fields("name^3", "description^1")
                        )
                    )
                    // Boost ตาม rating
                    .functions(f -> f
                        .fieldValueFactor(fvf -> fvf
                            .field("rating")
                            .factor(1.5)
                            .modifier(FieldValueFactorModifier.Sqrt)
                            .missing(1.0)
                        )
                    )
                    // Boost สินค้าที่มีรีวิวมาก
                    .functions(f -> f
                        .fieldValueFactor(fvf -> fvf
                            .field("reviewCount")
                            .factor(0.5)
                            .modifier(FieldValueFactorModifier.Log1p)
                            .missing(0.0)
                        )
                    )
                    // Decay ตามเวลา - สินค้าใหม่ได้ boost มากกว่า
                    .functions(f -> f
                        .gauss(g -> g
                            .field("createdAt")
                            .placement(p -> p
                                .origin(JsonData.of("now"))
                                .scale("30d")
                                .decay(0.5)
                            )
                        )
                    )
                    .scoreMode(FunctionScoreMode.Sum)
                    .boostMode(FunctionBoostMode.Multiply)
                )
            )
            .build();

        return elasticsearchOperations.search(query, ProductDocument.class)
            .stream()
            .map(SearchHit::getContent)
            .collect(Collectors.toList());
    }

    // Pinned query - ดัน product ที่ต้องการขึ้นก่อน
    public List<ProductDocument> searchWithPinnedResults(
            String keyword, List<String> pinnedIds) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q
                .pinned(p -> p
                    .ids(pinnedIds)
                    .organic(o -> o
                        .multiMatch(mm -> mm
                            .query(keyword)
                            .fields("name^3", "description^1")
                        )
                    )
                )
            )
            .build();

        return elasticsearchOperations.search(query, ProductDocument.class)
            .stream()
            .map(SearchHit::getContent)
            .collect(Collectors.toList());
    }

    // More Like This - ค้นหาสินค้าที่คล้ายกัน
    public List<ProductDocument> findSimilarProducts(String productId) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q
                .moreLikeThis(mlt -> mlt
                    .like(l -> l
                        .document(d -> d
                            .index("products")
                            .id(productId)
                        )
                    )
                    .fields("name", "description", "category", "tags")
                    .minTermFreq(1)
                    .maxQueryTerms(25)
                    .minDocFreq(1)
                )
            )
            .withPageable(org.springframework.data.domain.PageRequest.of(0, 10))
            .build();

        return elasticsearchOperations.search(query, ProductDocument.class)
            .stream()
            .map(SearchHit::getContent)
            .collect(Collectors.toList());
    }
}
```

---

## ขั้นตอนที่ 2650: Search Controller และ REST API

```java
// controller/SearchController.java
package com.example.search.controller;

import com.example.search.dto.*;
import com.example.search.service.*;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.io.IOException;
import java.util.List;

@RestController
@RequestMapping("/api/v1/search")
@RequiredArgsConstructor
public class SearchController {

    private final ProductSearchService productSearchService;
    private final AutoCompleteService autoCompleteService;
    private final AggregationSearchService aggregationSearchService;
    private final RelevanceTuningService relevanceTuningService;
    private final DataSyncService dataSyncService;

    @GetMapping("/products")
    public ResponseEntity<SearchResponse> search(
            @RequestParam(required = false) String q,
            @RequestParam(required = false) String category,
            @RequestParam(required = false) Double minPrice,
            @RequestParam(required = false) Double maxPrice,
            @RequestParam(required = false) List<String> tags,
            @RequestParam(required = false) Float minRating,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "_score") String sortBy,
            @RequestParam(defaultValue = "desc") String sortDir) {

        ProductSearchRequest request = ProductSearchRequest.builder()
            .keyword(q)
            .category(category)
            .minPrice(minPrice)
            .maxPrice(maxPrice)
            .tags(tags)
            .minRating(minRating)
            .build();

        Sort sort = sortDir.equalsIgnoreCase("asc") ?
            Sort.by(Sort.Direction.ASC, sortBy) :
            Sort.by(Sort.Direction.DESC, sortBy);

        var results = productSearchService.advancedSearch(
            request, PageRequest.of(page, size, sort));

        return ResponseEntity.ok(new SearchResponse(results));
    }

    @GetMapping("/autocomplete")
    public ResponseEntity<List<String>> autocomplete(
            @RequestParam String q) {
        return ResponseEntity.ok(autoCompleteService.autocomplete(q));
    }

    @GetMapping("/suggest")
    public ResponseEntity<List<String>> suggest(
            @RequestParam String q) {
        return ResponseEntity.ok(autoCompleteService.suggest(q));
    }

    @GetMapping("/faceted")
    public ResponseEntity<FacetedSearchResult> facetedSearch(
            @RequestParam(required = false) String q,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) throws IOException {
        return ResponseEntity.ok(aggregationSearchService.facetedSearch(q, page, size));
    }

    @GetMapping("/products/{id}/similar")
    public ResponseEntity<List<?>> similarProducts(@PathVariable String id) {
        return ResponseEntity.ok(relevanceTuningService.findSimilarProducts(id));
    }

    @PostMapping("/admin/reindex")
    public ResponseEntity<String> reindex() {
        dataSyncService.fullReindex();
        return ResponseEntity.accepted().body("Reindex started in background");
    }
}
```

---

## ขั้นตอนที่ 2651: DTO Classes

```java
// dto/ProductSearchRequest.java
package com.example.search.dto;

import lombok.Builder;
import lombok.Data;
import java.util.List;

@Data
@Builder
public class ProductSearchRequest {
    private String keyword;
    private String category;
    private Double minPrice;
    private Double maxPrice;
    private List<String> tags;
    private Float minRating;
    private String brandId;
    private Boolean inStock;
}
```

```java
// dto/FacetedSearchResult.java
package com.example.search.dto;

import com.example.search.model.ProductDocument;
import lombok.Data;
import java.util.List;
import java.util.Map;

@Data
public class FacetedSearchResult {
    private List<ProductDocument> products;
    private long total;
    private Map<String, Long> categoryFacets;
    private Map<String, Long> priceFacets;
    private Double averagePrice;
    private Map<String, Long> brandFacets;
    private Map<String, Long> ratingFacets;
}
```

---

## ขั้นตอนที่ 2652: การทดสอบ Elasticsearch

```java
// test/ProductSearchServiceTest.java
package com.example.search.service;

import com.example.search.model.ProductDocument;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.elasticsearch.ElasticsearchContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.math.BigDecimal;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Testcontainers
class ProductSearchServiceTest {

    @Container
    static ElasticsearchContainer elasticsearch =
        new ElasticsearchContainer("docker.elastic.co/elasticsearch/elasticsearch:8.11.0")
            .withEnv("discovery.type", "single-node")
            .withEnv("xpack.security.enabled", "false");

    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.elasticsearch.uris", elasticsearch::getHttpHostAddress);
    }

    @Autowired
    private ProductSearchService searchService;

    @Autowired
    private ProductSearchRepository searchRepository;

    @BeforeEach
    void setUp() {
        searchRepository.deleteAll();
        // สร้าง test data
        List<ProductDocument> products = List.of(
            createProduct("1", "iPhone 15 Pro", "โทรศัพท์ Apple", "Electronics", new BigDecimal("45000")),
            createProduct("2", "Samsung Galaxy S24", "โทรศัพท์ Samsung", "Electronics", new BigDecimal("35000")),
            createProduct("3", "Nike Air Max", "รองเท้าวิ่ง Nike", "Shoes", new BigDecimal("3500"))
        );
        searchRepository.saveAll(products);

        // รอ indexing
        try { Thread.sleep(1000); } catch (InterruptedException e) {}
    }

    @Test
    void shouldFindProductByKeyword() {
        var request = ProductSearchRequest.builder()
            .keyword("iPhone")
            .build();

        var results = searchService.advancedSearch(
            request,
            org.springframework.data.domain.PageRequest.of(0, 10)
        );

        assertThat(results.getContent()).hasSize(1);
        assertThat(results.getContent().get(0).getName()).contains("iPhone");
    }

    @Test
    void shouldFilterByCategory() {
        var request = ProductSearchRequest.builder()
            .category("Electronics")
            .build();

        var results = searchService.advancedSearch(
            request,
            org.springframework.data.domain.PageRequest.of(0, 10)
        );

        assertThat(results.getContent()).hasSize(2);
        results.getContent().forEach(p ->
            assertThat(p.getCategory()).isEqualTo("Electronics")
        );
    }

    @Test
    void shouldFilterByPriceRange() {
        var request = ProductSearchRequest.builder()
            .minPrice(30000.0)
            .maxPrice(50000.0)
            .build();

        var results = searchService.advancedSearch(
            request,
            org.springframework.data.domain.PageRequest.of(0, 10)
        );

        assertThat(results.getContent()).hasSize(2);
    }

    private ProductDocument createProduct(String id, String name, String desc,
                                           String category, BigDecimal price) {
        ProductDocument doc = new ProductDocument();
        doc.setId(id);
        doc.setName(name);
        doc.setDescription(desc);
        doc.setCategory(category);
        doc.setPrice(price);
        doc.setActive(true);
        doc.setRating(4.5f);
        doc.setReviewCount(100);
        return doc;
    }
}
```

---

## สรุป

ในส่วนนี้เราได้เรียนรู้:

1. **Index Mapping** - การกำหนด field types และ analyzers ให้เหมาะกับภาษา
2. **Full-text Search** - Multi-match, Boolean queries
3. **Fuzzy Search** - ทนต่อการพิมพ์ผิดด้วย Levenshtein distance
4. **Aggregations** - Faceted search สำหรับ e-commerce
5. **Auto-complete** - Edge n-gram และ Search As You Type
6. **Data Sync** - Event-driven sync จาก PostgreSQL
7. **Relevance Tuning** - Function score, pinned results, More Like This

---

*[← Part 75: Advanced Monitoring](./part-75-advanced-monitoring.md) | [Part 77: File Storage →](./part-77-file-storage.md)*
