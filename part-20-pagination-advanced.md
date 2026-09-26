# Part 20: Pagination & Filtering ขั้นสูง
## ขั้นตอนที่ 516-545

> **ระดับ:** กลาง (Intermediate)  
> **เวลาเรียน:** 4-5 ชั่วโมง  
> **เป้าหมาย:** Pagination และ Filtering ที่ยืดหยุ่นและมีประสิทธิภาพ

---

## ขั้นตอนที่ 516: Standard Pagination

```java
// Request parameters
@GetMapping
public ResponseEntity<ApiResponse<PageResponse<ProductResponse>>> search(
    @RequestParam(defaultValue = "0") @Min(0) int page,
    @RequestParam(defaultValue = "20") @Min(1) @Max(100) int size,
    @RequestParam(defaultValue = "createdAt") String sortBy,
    @RequestParam(defaultValue = "desc") @Pattern(regexp = "asc|desc") String sortDir
) {
    // Whitelist allowed sort fields
    Set<String> allowedSortFields = Set.of("name", "price", "createdAt", "stock");
    if (!allowedSortFields.contains(sortBy)) {
        sortBy = "createdAt";  // default
    }
    
    Sort sort = sortDir.equals("asc") ? Sort.by(sortBy).ascending() : Sort.by(sortBy).descending();
    Pageable pageable = PageRequest.of(page, size, sort);
    
    return ResponseEntity.ok(ApiResponse.success(
        PageResponse.of(productService.findAll(pageable))
    ));
}
```

---

## ขั้นตอนที่ 517: Cursor-Based Pagination (Keyset)

```java
// Better for large datasets - no offset problem
@GetMapping("/cursor")
public ResponseEntity<ApiResponse<CursorPageResponse<ProductResponse>>> searchWithCursor(
    @RequestParam(required = false) Long cursor,  // last seen ID
    @RequestParam(defaultValue = "20") @Max(100) int size
) {
    List<ProductResponse> products = productService.findWithCursor(cursor, size + 1);
    
    boolean hasNext = products.size() > size;
    if (hasNext) {
        products = products.subList(0, size);
    }
    
    Long nextCursor = hasNext ? products.get(products.size() - 1).id() : null;
    
    return ResponseEntity.ok(ApiResponse.success(
        new CursorPageResponse<>(products, nextCursor, hasNext)
    ));
}

// Repository
@Query("SELECT p FROM Product p WHERE (:cursor IS NULL OR p.id > :cursor) ORDER BY p.id ASC")
List<Product> findWithCursor(@Param("cursor") Long cursor, Pageable pageable);

// Service
public List<ProductResponse> findWithCursor(Long cursor, int limit) {
    Pageable pageable = PageRequest.of(0, limit);
    return productRepository.findWithCursor(cursor, pageable)
        .stream().map(productMapper::toResponse).toList();
}

// Response
public record CursorPageResponse<T>(
    List<T> content,
    Long nextCursor,
    boolean hasMore
) {}
```

---

## ขั้นตอนที่ 518: Dynamic Filtering with Specification Builder

```java
// Generic Specification Builder
public class SpecificationBuilder<T> {
    
    private final List<Specification<T>> specs = new ArrayList<>();
    
    public SpecificationBuilder<T> eq(String field, Object value) {
        if (value != null) {
            specs.add((root, query, cb) -> cb.equal(root.get(field), value));
        }
        return this;
    }
    
    public SpecificationBuilder<T> like(String field, String value) {
        if (value != null && !value.isBlank()) {
            specs.add((root, query, cb) -> 
                cb.like(cb.lower(root.get(field)), "%" + value.toLowerCase() + "%"));
        }
        return this;
    }
    
    public SpecificationBuilder<T> gte(String field, Comparable<?> value) {
        if (value != null) {
            specs.add((root, query, cb) -> 
                cb.greaterThanOrEqualTo(root.get(field), (Comparable) value));
        }
        return this;
    }
    
    public SpecificationBuilder<T> lte(String field, Comparable<?> value) {
        if (value != null) {
            specs.add((root, query, cb) -> 
                cb.lessThanOrEqualTo(root.get(field), (Comparable) value));
        }
        return this;
    }
    
    public SpecificationBuilder<T> between(String field, Comparable<?> from, Comparable<?> to) {
        if (from != null && to != null) {
            specs.add((root, query, cb) -> 
                cb.between(root.get(field), (Comparable) from, (Comparable) to));
        } else {
            gte(field, from);
            lte(field, to);
        }
        return this;
    }
    
    public SpecificationBuilder<T> in(String field, Collection<?> values) {
        if (values != null && !values.isEmpty()) {
            specs.add((root, query, cb) -> root.get(field).in(values));
        }
        return this;
    }
    
    public SpecificationBuilder<T> joinLike(String join, String field, String value) {
        if (value != null && !value.isBlank()) {
            specs.add((root, query, cb) -> 
                cb.like(cb.lower(root.join(join).get(field)), "%" + value.toLowerCase() + "%"));
        }
        return this;
    }
    
    public Specification<T> build() {
        return specs.stream()
            .reduce(Specification.where(null), Specification::and);
    }
}

// Usage
public Page<ProductResponse> search(ProductSearchRequest req, Pageable pageable) {
    Specification<Product> spec = new SpecificationBuilder<Product>()
        .like("name", req.keyword())
        .eq("status", req.status())
        .between("price", req.minPrice(), req.maxPrice())
        .eq("category.id", req.categoryId())
        .build();
    
    return productRepository.findAll(spec, pageable).map(productMapper::toResponse);
}
```

---

## ขั้นตอนที่ 519: Filter Query Language (RSQL)

```xml
<!-- RSQL parser -->
<dependency>
    <groupId>io.github.perplexhub</groupId>
    <artifactId>rsql-jpa-spring-boot-starter</artifactId>
    <version>6.0.17</version>
</dependency>
```

```java
// RSQL example: GET /api/v1/products?filter=name==iPhone;price=gt=10000
@GetMapping
public ResponseEntity<ApiResponse<PageResponse<ProductResponse>>> search(
    @RequestParam(required = false) String filter,
    Pageable pageable
) {
    Specification<Product> spec = RSQLJPASupport.toSpecification(filter);
    Page<Product> products = productRepository.findAll(spec, pageable);
    return ResponseEntity.ok(ApiResponse.success(PageResponse.of(products.map(productMapper::toResponse))));
}

/*
RSQL operators:
  ==   equals
  !=   not equals
  =gt= greater than
  =lt= less than
  =ge= greater or equal
  =le= less or equal
  =in= in list
  =out= not in list
  =like= like (wildcard *)

Examples:
  filter=name==iPhone15                          (exact)
  filter=price=gt=10000                          (greater than)
  filter=status=in=(ACTIVE,OUT_OF_STOCK)         (in list)
  filter=name==*Phone*                           (contains)
  filter=price=gt=5000;status==ACTIVE            (AND)
  filter=price=lt=1000,price=gt=50000            (OR)
  filter=category.name==Electronics             (join)
*/
```

---

## ขั้นตอนที่ 520: Search with Elasticsearch (Basic)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

```java
// Product search document
@Document(indexName = "products")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class ProductDocument {
    
    @Id
    private String id;
    
    @Field(type = FieldType.Text, analyzer = "thai_analyzer")
    private String name;
    
    @Field(type = FieldType.Text)
    private String description;
    
    @Field(type = FieldType.Double)
    private Double price;
    
    @Field(type = FieldType.Keyword)
    private String status;
    
    @Field(type = FieldType.Keyword)
    private String categoryName;
    
    @Field(type = FieldType.Long)
    private Long viewCount;
}

// Elasticsearch repository
public interface ProductSearchRepository extends ElasticsearchRepository<ProductDocument, String> {
    
    Page<ProductDocument> findByName(String name, Pageable pageable);
    
    @Query("""
        {
          "multi_match": {
            "query": "?0",
            "fields": ["name^3", "description"],
            "fuzziness": "AUTO"
          }
        }
        """)
    Page<ProductDocument> fullTextSearch(String keyword, Pageable pageable);
}

// Sync JPA to ES
@Component
@RequiredArgsConstructor
@Slf4j
public class ProductIndexer {
    
    private final ProductSearchRepository searchRepo;
    
    @EventListener
    public void onProductCreated(ProductCreatedEvent event) {
        ProductDocument doc = ProductDocument.builder()
            .id(event.product().id().toString())
            .name(event.product().name())
            .price(event.product().price().doubleValue())
            .status(event.product().status().name())
            .build();
        
        searchRepo.save(doc);
        log.debug("Product indexed: {}", doc.getId());
    }
    
    @EventListener
    public void onProductDeleted(ProductDeletedEvent event) {
        searchRepo.deleteById(event.productId().toString());
    }
}
```

---

## ขั้นตอนที่ 521: Advanced Sorting

```java
// Multi-column sort from request
@GetMapping
public ResponseEntity<?> search(
    @RequestParam(defaultValue = "createdAt:desc") String sort,
    Pageable pageable
) {
    // sort format: "field1:dir1,field2:dir2"
    Sort resolvedSort = resolveSort(sort);
    Pageable resolved = PageRequest.of(pageable.getPageNumber(), pageable.getPageSize(), resolvedSort);
    
    return ResponseEntity.ok(...);
}

private Sort resolveSort(String sortParam) {
    Set<String> allowed = Set.of("name", "price", "createdAt", "viewCount", "stock");
    
    List<Sort.Order> orders = Arrays.stream(sortParam.split(","))
        .map(s -> s.split(":"))
        .filter(parts -> parts.length == 2 && allowed.contains(parts[0]))
        .map(parts -> parts[1].equals("asc") 
            ? Sort.Order.asc(parts[0]) 
            : Sort.Order.desc(parts[0]))
        .toList();
    
    return orders.isEmpty() ? Sort.by("createdAt").descending() : Sort.by(orders);
}
```

---

## ขั้นตอนที่ 522-545: Pageable Configuration

```java
// Global pageable defaults in application.yml
// spring.data.web.pageable.default-page-size=20
// spring.data.web.pageable.max-page-size=100
// spring.data.web.pageable.size-parameter=size
// spring.data.web.pageable.page-parameter=page
// spring.data.web.sort.sort-parameter=sort

// Custom PageableHandlerMethodArgumentResolver
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        PageableHandlerMethodArgumentResolver resolver = new PageableHandlerMethodArgumentResolver();
        resolver.setDefaultPageable(PageRequest.of(0, 20, Sort.by("createdAt").descending()));
        resolver.setMaxPageSize(100);
        resolvers.add(resolver);
    }
}

// PageResponse with metadata
public record PageResponse<T>(
    List<T> content,
    PageMeta meta
) {
    public record PageMeta(
        int page,
        int size,
        long totalElements,
        int totalPages,
        boolean first,
        boolean last,
        boolean empty,
        List<SortMeta> sort
    ) {}
    
    public record SortMeta(
        String property,
        String direction,
        boolean ascending,
        boolean descending
    ) {}
    
    public static <T> PageResponse<T> of(Page<T> page) {
        List<SortMeta> sortMeta = page.getSort().stream()
            .map(o -> new SortMeta(
                o.getProperty(), 
                o.getDirection().name(),
                o.isAscending(),
                o.isDescending()
            ))
            .toList();
        
        return new PageResponse<>(
            page.getContent(),
            new PageMeta(
                page.getNumber(),
                page.getSize(),
                page.getTotalElements(),
                page.getTotalPages(),
                page.isFirst(),
                page.isLast(),
                page.isEmpty(),
                sortMeta
            )
        );
    }
}
```

---

*[← Part 19: Email Service](./part-19-email-service.md) | [Part 21: Redis Caching →](./part-21-redis-caching.md)*
