# Part 88: Data Streaming
## ขั้นตอนที่ 3121-3160

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 7-9 ชั่วโมง
**เป้าหมาย:** เรียนรู้ Spring Cloud Stream และ Kafka Streams สำหรับการประมวลผลข้อมูลแบบ real-time พร้อม windowing, joins, และ interactive queries

---

## ขั้นตอนที่ 3121: Spring Cloud Stream คืออะไร?

Spring Cloud Stream เป็น framework สำหรับสร้าง message-driven microservices โดยใช้ abstraction layer บน Kafka, RabbitMQ และ messaging systems อื่นๆ

### เพิ่ม Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring Cloud Stream with Kafka binder -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream-binder-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream-binder-kafka-streams</artifactId>
    </dependency>
    
    <!-- Kafka Streams -->
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-streams</artifactId>
    </dependency>
    
    <!-- Spring Kafka -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    
    <!-- Test support -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream-test-support</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-streams-test-utils</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2022.0.4</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

---

## ขั้นตอนที่ 3122: Spring Cloud Stream Basics

Spring Cloud Stream ใช้ `Supplier`, `Function`, `Consumer` จาก Java functional interfaces

```java
// model/SensorReading.java
package com.example.streaming.model;

import lombok.Data;
import lombok.Builder;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;

import java.time.Instant;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class SensorReading {
    private String sensorId;
    private String location;
    private double temperature;
    private double humidity;
    private double pressure;
    private Instant timestamp;
    private String status;
}
```

```java
// model/Alert.java
package com.example.streaming.model;

import lombok.Data;
import lombok.Builder;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;

import java.time.Instant;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Alert {
    private String alertId;
    private String sensorId;
    private String alertType;
    private String severity;
    private String message;
    private double value;
    private double threshold;
    private Instant triggeredAt;
}
```

```java
// function/SensorStreamFunctions.java
package com.example.streaming.function;

import com.example.streaming.model.Alert;
import com.example.streaming.model.SensorReading;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Flux;

import java.time.Duration;
import java.time.Instant;
import java.util.UUID;
import java.util.function.Consumer;
import java.util.function.Function;
import java.util.function.Supplier;

@Slf4j
@Component
public class SensorStreamFunctions {

    // Supplier - สร้าง sensor data จำลอง
    @Bean
    public Supplier<SensorReading> sensorDataSupplier() {
        return () -> SensorReading.builder()
            .sensorId("SENSOR-" + (int)(Math.random() * 10))
            .location("Bangkok")
            .temperature(20 + Math.random() * 30)
            .humidity(40 + Math.random() * 50)
            .pressure(1000 + Math.random() * 50)
            .timestamp(Instant.now())
            .status("ACTIVE")
            .build();
    }

    // Function - แปลง SensorReading เป็น Alert ถ้าเกิน threshold
    @Bean
    public Function<SensorReading, Alert> temperatureAlertProcessor() {
        return reading -> {
            double threshold = 35.0;
            if (reading.getTemperature() > threshold) {
                log.warn("High temperature alert: {} at {}",
                    reading.getTemperature(), reading.getSensorId());
                return Alert.builder()
                    .alertId(UUID.randomUUID().toString())
                    .sensorId(reading.getSensorId())
                    .alertType("HIGH_TEMPERATURE")
                    .severity(reading.getTemperature() > 40 ? "CRITICAL" : "WARNING")
                    .message(String.format("Temperature %.1f exceeds threshold %.1f",
                        reading.getTemperature(), threshold))
                    .value(reading.getTemperature())
                    .threshold(threshold)
                    .triggeredAt(Instant.now())
                    .build();
            }
            return null; // ไม่สร้าง alert ถ้าปกติ
        };
    }

    // Consumer - รับและบันทึก alerts
    @Bean
    public Consumer<Alert> alertConsumer() {
        return alert -> {
            if (alert != null) {
                log.info("Received alert: [{}] {} - {} (value: {}, threshold: {})",
                    alert.getSeverity(),
                    alert.getAlertType(),
                    alert.getSensorId(),
                    alert.getValue(),
                    alert.getThreshold());
                // บันทึกลงฐานข้อมูลหรือส่ง notification
            }
        };
    }

    // Reactive Function ด้วย Flux
    @Bean
    public Function<Flux<SensorReading>, Flux<Alert>> reactiveAlertProcessor() {
        return readings -> readings
            .filter(r -> r.getTemperature() > 35.0 || r.getHumidity() > 85.0)
            .map(reading -> {
                String alertType = reading.getTemperature() > 35.0
                    ? "HIGH_TEMPERATURE" : "HIGH_HUMIDITY";
                double value = reading.getTemperature() > 35.0
                    ? reading.getTemperature() : reading.getHumidity();
                double threshold = reading.getTemperature() > 35.0 ? 35.0 : 85.0;
                
                return Alert.builder()
                    .alertId(UUID.randomUUID().toString())
                    .sensorId(reading.getSensorId())
                    .alertType(alertType)
                    .severity("WARNING")
                    .value(value)
                    .threshold(threshold)
                    .triggeredAt(Instant.now())
                    .build();
            });
    }
}
```

---

## ขั้นตอนที่ 3123: Kafka Streams Topology

Kafka Streams ช่วยให้เราสร้าง stream processing topologies ได้

```java
// topology/SensorAggregationTopology.java
package com.example.streaming.topology;

import com.example.streaming.model.Alert;
import com.example.streaming.model.SensorReading;
import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.time.Duration;

@Slf4j
@Component
public class SensorAggregationTopology {

    @Autowired
    public void buildTopology(StreamsBuilder builder) {
        // สร้าง stream จาก topic
        KStream<String, SensorReading> sensorStream = builder
            .stream("sensor-readings",
                Consumed.with(Serdes.String(),
                    new SensorReadingSerdes()));

        // กรองเฉพาะ readings ที่ active
        KStream<String, SensorReading> activeReadings = sensorStream
            .filter((key, value) -> "ACTIVE".equals(value.getStatus()));

        // Group by sensorId
        KGroupedStream<String, SensorReading> groupedBySensor = activeReadings
            .groupByKey(Grouped.with(Serdes.String(), new SensorReadingSerdes()));

        // Tumbling Window - รวมข้อมูลทุก 5 นาที
        KTable<Windowed<String>, SensorStats> windowedStats = groupedBySensor
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
            .aggregate(
                SensorStats::new,
                (key, reading, stats) -> stats.update(reading),
                Materialized.<String, SensorStats, WindowStore<Bytes, byte[]>>as(
                        "sensor-stats-store")
                    .withKeySerde(Serdes.String())
                    .withValueSerde(new SensorStatsSerdes())
            );

        // ส่งผลลัพธ์ไปยัง output topic
        windowedStats.toStream()
            .map((windowedKey, stats) -> KeyValue.pair(
                windowedKey.key(),
                stats
            ))
            .to("sensor-aggregates",
                Produced.with(Serdes.String(), new SensorStatsSerdes()));

        // สร้าง alert stream
        KStream<String, Alert> alerts = activeReadings
            .filter((key, reading) -> reading.getTemperature() > 35.0)
            .mapValues(reading -> Alert.builder()
                .sensorId(reading.getSensorId())
                .alertType("HIGH_TEMPERATURE")
                .severity(reading.getTemperature() > 40 ? "CRITICAL" : "WARNING")
                .value(reading.getTemperature())
                .threshold(35.0)
                .build());

        alerts.to("sensor-alerts",
            Produced.with(Serdes.String(), new AlertSerdes()));
        
        log.info("Sensor aggregation topology built successfully");
    }
}
```

---

## ขั้นตอนที่ 3124: Windowed Operations

การทำ windowed aggregations ใน Kafka Streams

```java
// topology/WindowedOperationsTopology.java
package com.example.streaming.topology;

import com.example.streaming.model.SensorReading;
import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KeyValue;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.time.Duration;

@Slf4j
@Component
public class WindowedOperationsTopology {

    @Autowired
    public void buildWindowedTopology(StreamsBuilder builder) {
        KStream<String, SensorReading> sensorStream = builder
            .stream("sensor-readings",
                Consumed.with(Serdes.String(), new SensorReadingSerdes())
                    .withTimestampExtractor(
                        (record, partitionTime) -> {
                            SensorReading reading = (SensorReading) record.value();
                            return reading.getTimestamp().toEpochMilli();
                        }
                    ));

        KGroupedStream<String, SensorReading> grouped = sensorStream
            .groupByKey(Grouped.with(Serdes.String(), new SensorReadingSerdes()));

        // 1. Tumbling Window - ไม่มี overlap, ไม่มีช่องว่าง
        // เหมาะสำหรับ: periodic reporting, batching
        KTable<Windowed<String>, Double> tumblingAvgTemp = grouped
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
            .aggregate(
                () -> new TemperatureAggregator(0, 0.0),
                (key, reading, agg) -> {
                    agg.setCount(agg.getCount() + 1);
                    agg.setSum(agg.getSum() + reading.getTemperature());
                    return agg;
                },
                Materialized.<String, TemperatureAggregator,
                    WindowStore<org.apache.kafka.common.utils.Bytes, byte[]>>as(
                        "tumbling-temp-store")
            )
            .mapValues(agg -> agg.getCount() > 0
                ? agg.getSum() / agg.getCount() : 0.0);

        tumblingAvgTemp.toStream()
            .map((window, avgTemp) -> KeyValue.pair(
                window.key() + "@" + window.window().startTime(),
                String.format("avg_temp=%.2f", avgTemp)
            ))
            .to("tumbling-window-results",
                Produced.with(Serdes.String(), Serdes.String()));

        // 2. Hopping Window - มี overlap (advance < size)
        // เหมาะสำหรับ: sliding average, trend detection
        KTable<Windowed<String>, Double> hoppingAvgTemp = grouped
            .windowedBy(TimeWindows.ofSizeAndGrace(
                Duration.ofMinutes(10),  // window size
                Duration.ofMinutes(1)   // grace period
            ).advanceBy(Duration.ofMinutes(2))) // advance (hop)
            .aggregate(
                () -> new TemperatureAggregator(0, 0.0),
                (key, reading, agg) -> {
                    agg.setCount(agg.getCount() + 1);
                    agg.setSum(agg.getSum() + reading.getTemperature());
                    return agg;
                },
                Materialized.as("hopping-temp-store")
            )
            .mapValues(agg -> agg.getCount() > 0
                ? agg.getSum() / agg.getCount() : 0.0);

        hoppingAvgTemp.toStream()
            .mapValues(v -> String.format("hopping_avg=%.2f", v))
            .to("hopping-window-results",
                Produced.with(
                    WindowedSerdes.timeWindowedSerdeFrom(String.class, Duration.ofMinutes(10).toMillis()),
                    Serdes.String()
                ));

        // 3. Session Window - group ตาม activity
        // เหมาะสำหรับ: user sessions, activity tracking
        KTable<Windowed<String>, Long> sessionCount = grouped
            .windowedBy(SessionWindows.ofInactivityGapWithNoGrace(
                Duration.ofMinutes(30))) // 30 นาทีไม่มีข้อมูล = session สิ้นสุด
            .count(Materialized.as("session-count-store"));

        sessionCount.toStream()
            .map((window, count) -> KeyValue.pair(
                window.key(),
                String.format("session_readings=%d,duration=%dms",
                    count,
                    window.window().end() - window.window().start())
            ))
            .to("session-window-results",
                Produced.with(Serdes.String(), Serdes.String()));

        // 4. Sliding Window - continuous overlap
        // เหมาะสำหรับ: real-time anomaly detection
        KTable<Windowed<String>, Double> slidingMax = grouped
            .windowedBy(SlidingWindows.ofTimeDifferenceWithNoGrace(
                Duration.ofMinutes(5))) // max time difference between records
            .aggregate(
                () -> Double.MIN_VALUE,
                (key, reading, maxTemp) -> Math.max(maxTemp, reading.getTemperature()),
                Materialized.as("sliding-max-store")
            );

        slidingMax.toStream()
            .mapValues(v -> String.format("max_temp=%.2f", v))
            .to("sliding-window-results",
                Produced.with(
                    WindowedSerdes.timeWindowedSerdeFrom(String.class, Duration.ofMinutes(5).toMillis()),
                    Serdes.String()
                ));
    }

    // Helper class สำหรับ aggregation
    @lombok.Data
    @lombok.AllArgsConstructor
    @lombok.NoArgsConstructor
    public static class TemperatureAggregator {
        private int count;
        private double sum;
    }
}
```

---

## ขั้นตอนที่ 3125: Stream Joins

การ join streams ใน Kafka Streams

```java
// topology/StreamJoinTopology.java
package com.example.streaming.topology;

import com.example.streaming.model.Alert;
import com.example.streaming.model.SensorReading;
import lombok.Data;
import lombok.extern.Slf4j;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.time.Duration;

@Slf4j
@Component
public class StreamJoinTopology {

    @Data
    public static class SensorMetadata {
        private String sensorId;
        private String location;
        private String owner;
        private String type;
        private boolean active;
    }

    @Data
    public static class EnrichedReading {
        private SensorReading reading;
        private SensorMetadata metadata;
        private String enrichedLocation;
    }

    @Autowired
    public void buildJoinTopology(StreamsBuilder builder) {
        // Stream ของ sensor readings
        KStream<String, SensorReading> sensorReadings = builder
            .stream("sensor-readings",
                Consumed.with(Serdes.String(), new SensorReadingSerdes()));

        // Table ของ sensor metadata (reference data)
        KTable<String, SensorMetadata> sensorMetadata = builder
            .table("sensor-metadata",
                Consumed.with(Serdes.String(), new SensorMetadataSerdes()),
                Materialized.as("sensor-metadata-store"));

        // Stream ของ alerts
        KStream<String, Alert> alertStream = builder
            .stream("sensor-alerts",
                Consumed.with(Serdes.String(), new AlertSerdes()));

        // 1. Stream-Table Join - enrich readings ด้วย metadata
        KStream<String, EnrichedReading> enrichedReadings = sensorReadings
            .join(
                sensorMetadata,
                (reading, metadata) -> {
                    if (metadata == null) return null;
                    EnrichedReading enriched = new EnrichedReading();
                    enriched.setReading(reading);
                    enriched.setMetadata(metadata);
                    enriched.setEnrichedLocation(
                        metadata.getLocation() + " (" + metadata.getType() + ")");
                    return enriched;
                },
                Joined.with(Serdes.String(),
                    new SensorReadingSerdes(),
                    new SensorMetadataSerdes())
            );

        enrichedReadings
            .filter((key, value) -> value != null)
            .to("enriched-readings",
                Produced.with(Serdes.String(), new EnrichedReadingSerdes()));

        // 2. Stream-Stream Join - เชื่อม readings กับ alerts ในช่วงเวลาใกล้กัน
        KStream<String, String> correlatedEvents = sensorReadings
            .join(
                alertStream,
                (reading, alert) -> String.format(
                    "Reading: %.1f at %s | Alert: %s (%s)",
                    reading.getTemperature(),
                    reading.getTimestamp(),
                    alert.getAlertType(),
                    alert.getSeverity()
                ),
                JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(5)),
                StreamJoined.with(Serdes.String(),
                    new SensorReadingSerdes(),
                    new AlertSerdes())
            );

        correlatedEvents.to("correlated-events",
            Produced.with(Serdes.String(), Serdes.String()));

        // 3. Left Join - readings พร้อม optional alert
        KStream<String, String> readingsWithOptionalAlert = sensorReadings
            .leftJoin(
                alertStream,
                (reading, alert) -> {
                    String base = String.format("Sensor: %s, Temp: %.1f",
                        reading.getSensorId(), reading.getTemperature());
                    if (alert != null) {
                        return base + " [ALERT: " + alert.getAlertType() + "]";
                    }
                    return base + " [OK]";
                },
                JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(5)),
                StreamJoined.with(Serdes.String(),
                    new SensorReadingSerdes(),
                    new AlertSerdes())
            );

        readingsWithOptionalAlert.to("readings-with-alerts",
            Produced.with(Serdes.String(), Serdes.String()));

        // 4. KTable-KTable Join - join ระหว่าง 2 tables
        KTable<String, String> sensorLocations = builder
            .table("sensor-locations",
                Consumed.with(Serdes.String(), Serdes.String()));

        KTable<String, String> sensorOwners = builder
            .table("sensor-owners",
                Consumed.with(Serdes.String(), Serdes.String()));

        KTable<String, String> combinedInfo = sensorLocations
            .join(sensorOwners,
                (location, owner) -> location + " | Owner: " + owner,
                Materialized.as("combined-sensor-info"));

        combinedInfo.toStream()
            .to("sensor-combined-info",
                Produced.with(Serdes.String(), Serdes.String()));
    }
}
```

---

## ขั้นตอนที่ 3126: Real-Time Aggregations

การทำ aggregations แบบ real-time

```java
// topology/RealTimeAggregationTopology.java
package com.example.streaming.topology;

import com.example.streaming.model.SensorReading;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KeyValue;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.time.Duration;

@Slf4j
@Component
public class RealTimeAggregationTopology {

    @Data
    public static class SensorStats {
        private String sensorId;
        private long count;
        private double minTemp;
        private double maxTemp;
        private double sumTemp;
        private double minHumidity;
        private double maxHumidity;
        private double sumHumidity;
        private long lastUpdated;

        public SensorStats() {
            this.minTemp = Double.MAX_VALUE;
            this.maxTemp = Double.MIN_VALUE;
            this.minHumidity = Double.MAX_VALUE;
            this.maxHumidity = Double.MIN_VALUE;
        }

        public SensorStats update(SensorReading reading) {
            this.sensorId = reading.getSensorId();
            this.count++;
            this.minTemp = Math.min(this.minTemp, reading.getTemperature());
            this.maxTemp = Math.max(this.maxTemp, reading.getTemperature());
            this.sumTemp += reading.getTemperature();
            this.minHumidity = Math.min(this.minHumidity, reading.getHumidity());
            this.maxHumidity = Math.max(this.maxHumidity, reading.getHumidity());
            this.sumHumidity += reading.getHumidity();
            this.lastUpdated = System.currentTimeMillis();
            return this;
        }

        public double getAvgTemp() {
            return count > 0 ? sumTemp / count : 0;
        }

        public double getAvgHumidity() {
            return count > 0 ? sumHumidity / count : 0;
        }
    }

    @Autowired
    public void buildAggregationTopology(StreamsBuilder builder) {
        KStream<String, SensorReading> sensorStream = builder
            .stream("sensor-readings",
                Consumed.with(Serdes.String(), new SensorReadingSerdes()));

        // Re-key by sensorId เพื่อ group ให้ถูกต้อง
        KStream<String, SensorReading> keyedStream = sensorStream
            .selectKey((key, reading) -> reading.getSensorId());

        // Running total - aggregate ตลอดเวลา (ไม่มี window)
        KTable<String, SensorStats> runningStats = keyedStream
            .groupByKey(Grouped.with(Serdes.String(), new SensorReadingSerdes()))
            .aggregate(
                SensorStats::new,
                (key, reading, stats) -> stats.update(reading),
                Materialized.<String, SensorStats,
                    KeyValueStore<org.apache.kafka.common.utils.Bytes, byte[]>>as(
                        "running-sensor-stats")
                    .withKeySerde(Serdes.String())
                    .withValueSerde(new SensorStatsSerdes())
            );

        // ส่ง stats ไปยัง output topic
        runningStats.toStream()
            .mapValues(stats -> String.format(
                "{\"sensorId\":\"%s\",\"count\":%d,\"avgTemp\":%.2f,\"maxTemp\":%.2f,\"avgHumidity\":%.2f}",
                stats.getSensorId(),
                stats.getCount(),
                stats.getAvgTemp(),
                stats.getMaxTemp(),
                stats.getAvgHumidity()
            ))
            .to("sensor-running-stats",
                Produced.with(Serdes.String(), Serdes.String()));

        // Location-based aggregation
        KStream<String, SensorReading> keyedByLocation = sensorStream
            .selectKey((key, reading) -> reading.getLocation());

        KTable<String, Long> readingsPerLocation = keyedByLocation
            .groupByKey(Grouped.with(Serdes.String(), new SensorReadingSerdes()))
            .count(Materialized.as("readings-per-location"));

        readingsPerLocation.toStream()
            .mapValues(count -> count.toString())
            .to("readings-per-location",
                Produced.with(Serdes.String(), Serdes.String()));

        // Anomaly detection - ใช้ window aggregate ตรวจหา anomaly
        KTable<Windowed<String>, Double> recentAvgTemp = keyedStream
            .groupByKey(Grouped.with(Serdes.String(), new SensorReadingSerdes()))
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
            .aggregate(
                () -> new double[]{0, 0}, // [count, sum]
                (key, reading, agg) -> {
                    agg[0]++;
                    agg[1] += reading.getTemperature();
                    return agg;
                },
                Materialized.as("recent-temp-window")
            )
            .mapValues(agg -> agg[0] > 0 ? agg[1] / agg[0] : 0.0);

        // Detect sudden changes
        recentAvgTemp.toStream()
            .map((window, avgTemp) -> KeyValue.pair(window.key(), avgTemp))
            .filter((key, avgTemp) -> avgTemp > 38.0) // threshold สูง
            .mapValues(temp -> "ANOMALY: High average temperature " + temp)
            .to("anomaly-alerts",
                Produced.with(Serdes.String(), Serdes.String()));
    }
}
```

---

## ขั้นตอนที่ 3127: Interactive Queries

Interactive Queries ช่วยให้ query สถานะของ Kafka Streams ได้แบบ real-time

```java
// service/InteractiveQueryService.java
package com.example.streaming.service;

import com.example.streaming.topology.RealTimeAggregationTopology.SensorStats;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.KeyQueryMetadata;
import org.apache.kafka.streams.StoreQueryParameters;
import org.apache.kafka.streams.state.*;
import org.springframework.kafka.config.StreamsBuilderFactoryBean;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

@Slf4j
@Service
@RequiredArgsConstructor
public class InteractiveQueryService {

    private final StreamsBuilderFactoryBean streamsBuilderFactoryBean;

    // ดึง stats ของ sensor เฉพาะตัว
    public Optional<SensorStats> getSensorStats(String sensorId) {
        KafkaStreams kafkaStreams = streamsBuilderFactoryBean.getKafkaStreams();
        if (kafkaStreams == null) return Optional.empty();

        try {
            ReadOnlyKeyValueStore<String, SensorStats> store = kafkaStreams
                .store(StoreQueryParameters.fromNameAndType(
                    "running-sensor-stats",
                    QueryableStoreTypes.keyValueStore()
                ));
            
            return Optional.ofNullable(store.get(sensorId));
        } catch (Exception e) {
            log.error("Error querying store for sensorId: {}", sensorId, e);
            return Optional.empty();
        }
    }

    // ดึง stats ของ sensors ทั้งหมด
    public List<SensorStats> getAllSensorStats() {
        KafkaStreams kafkaStreams = streamsBuilderFactoryBean.getKafkaStreams();
        if (kafkaStreams == null) return List.of();

        List<SensorStats> result = new ArrayList<>();
        try {
            ReadOnlyKeyValueStore<String, SensorStats> store = kafkaStreams
                .store(StoreQueryParameters.fromNameAndType(
                    "running-sensor-stats",
                    QueryableStoreTypes.keyValueStore()
                ));
            
            KeyValueIterator<String, SensorStats> iterator = store.all();
            while (iterator.hasNext()) {
                KeyValue<String, SensorStats> entry = iterator.next();
                result.add(entry.value);
            }
            iterator.close();
        } catch (Exception e) {
            log.error("Error querying all sensor stats", e);
        }
        return result;
    }

    // Query window store
    public List<String> getWindowedStats(String sensorId) {
        KafkaStreams kafkaStreams = streamsBuilderFactoryBean.getKafkaStreams();
        if (kafkaStreams == null) return List.of();

        List<String> results = new ArrayList<>();
        try {
            ReadOnlyWindowStore<String, Double> windowStore = kafkaStreams
                .store(StoreQueryParameters.fromNameAndType(
                    "sensor-stats-store",
                    QueryableStoreTypes.windowStore()
                ));
            
            long timeFrom = System.currentTimeMillis() - 3600000; // 1 ชั่วโมงที่ผ่านมา
            long timeTo = System.currentTimeMillis();
            
            WindowStoreIterator<Double> iterator =
                windowStore.fetch(sensorId, timeFrom, timeTo);
            
            while (iterator.hasNext()) {
                KeyValue<Long, Double> windowedValue = iterator.next();
                results.add(String.format("Time: %d, Value: %.2f",
                    windowedValue.key, windowedValue.value));
            }
            iterator.close();
        } catch (Exception e) {
            log.error("Error querying window store for sensorId: {}", sensorId, e);
        }
        return results;
    }

    // ตรวจสอบ metadata ของ store (สำหรับ distributed queries)
    public Optional<KeyQueryMetadata> getStoreMetadata(String sensorId) {
        KafkaStreams kafkaStreams = streamsBuilderFactoryBean.getKafkaStreams();
        if (kafkaStreams == null) return Optional.empty();

        try {
            KeyQueryMetadata metadata = kafkaStreams
                .queryMetadataForKey(
                    "running-sensor-stats",
                    sensorId,
                    org.apache.kafka.common.serialization.Serdes.String().serializer()
                );
            return Optional.ofNullable(metadata);
        } catch (Exception e) {
            log.error("Error getting metadata for sensorId: {}", sensorId, e);
            return Optional.empty();
        }
    }
}
```

---

## ขั้นตอนที่ 3128: Interactive Query REST API

```java
// controller/StreamQueryController.java
package com.example.streaming.controller;

import com.example.streaming.service.InteractiveQueryService;
import com.example.streaming.topology.RealTimeAggregationTopology.SensorStats;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@Slf4j
@RestController
@RequestMapping("/api/streams")
@RequiredArgsConstructor
public class StreamQueryController {

    private final InteractiveQueryService queryService;

    @GetMapping("/sensors/{sensorId}/stats")
    public ResponseEntity<SensorStats> getSensorStats(@PathVariable String sensorId) {
        return queryService.getSensorStats(sensorId)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @GetMapping("/sensors/stats")
    public ResponseEntity<List<SensorStats>> getAllStats() {
        return ResponseEntity.ok(queryService.getAllSensorStats());
    }

    @GetMapping("/sensors/{sensorId}/history")
    public ResponseEntity<List<String>> getSensorHistory(@PathVariable String sensorId) {
        return ResponseEntity.ok(queryService.getWindowedStats(sensorId));
    }

    @GetMapping("/sensors/{sensorId}/metadata")
    public ResponseEntity<Map<String, Object>> getSensorMetadata(
            @PathVariable String sensorId) {
        return queryService.getStoreMetadata(sensorId)
            .map(metadata -> ResponseEntity.ok(Map.of(
                "activeHost", metadata.activeHost().host(),
                "activePort", metadata.activeHost().port(),
                "standbyHosts", metadata.standbyHosts()
            )))
            .orElse(ResponseEntity.notFound().build());
    }
}
```

---

## ขั้นตอนที่ 3129: Custom Serdes

การสร้าง custom serializers/deserializers สำหรับ Kafka Streams

```java
// serde/SensorReadingSerdes.java
package com.example.streaming.serde;

import com.example.streaming.model.SensorReading;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.apache.kafka.common.serialization.Deserializer;
import org.apache.kafka.common.serialization.Serde;
import org.apache.kafka.common.serialization.Serializer;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class SensorReadingSerdes implements Serde<SensorReading> {

    private static final ObjectMapper objectMapper = new ObjectMapper()
        .registerModule(new JavaTimeModule());

    @Override
    public Serializer<SensorReading> serializer() {
        return (topic, data) -> {
            if (data == null) return null;
            try {
                return objectMapper.writeValueAsBytes(data);
            } catch (Exception e) {
                log.error("Error serializing SensorReading", e);
                throw new RuntimeException(e);
            }
        };
    }

    @Override
    public Deserializer<SensorReading> deserializer() {
        return (topic, bytes) -> {
            if (bytes == null) return null;
            try {
                return objectMapper.readValue(bytes, SensorReading.class);
            } catch (Exception e) {
                log.error("Error deserializing SensorReading", e);
                return null;
            }
        };
    }
}
```

---

## ขั้นตอนที่ 3130: Testing Kafka Streams Topologies

การทดสอบ Kafka Streams โดยไม่ต้องใช้ Kafka จริง

```java
// test/SensorAggregationTopologyTest.java
package com.example.streaming;

import com.example.streaming.model.SensorReading;
import com.example.streaming.topology.SensorAggregationTopology;
import com.example.streaming.topology.RealTimeAggregationTopology.SensorStats;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.*;
import org.apache.kafka.streams.state.KeyValueStore;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.Properties;

import static org.assertj.core.api.Assertions.assertThat;

class SensorAggregationTopologyTest {

    private TopologyTestDriver testDriver;
    private TestInputTopic<String, SensorReading> inputTopic;
    private TestOutputTopic<String, String> outputTopic;

    @BeforeEach
    void setup() {
        // กำหนด Streams properties สำหรับ test
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "test-sensor-aggregation");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "dummy:1234");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG,
            Serdes.String().getClass().getName());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG,
            Serdes.String().getClass().getName());

        // สร้าง topology
        StreamsBuilder builder = new StreamsBuilder();
        RealTimeAggregationTopology topology = new RealTimeAggregationTopology();
        topology.buildAggregationTopology(builder);
        Topology builtTopology = builder.build();

        // สร้าง test driver
        testDriver = new TopologyTestDriver(builtTopology, props);

        // สร้าง test topics
        inputTopic = testDriver.createInputTopic(
            "sensor-readings",
            Serdes.String().serializer(),
            new SensorReadingSerdes().serializer()
        );

        outputTopic = testDriver.createOutputTopic(
            "sensor-running-stats",
            Serdes.String().deserializer(),
            Serdes.String().deserializer()
        );
    }

    @AfterEach
    void tearDown() {
        testDriver.close();
    }

    @Test
    void shouldAggregateReadingsPerSensor() {
        // ส่ง readings หลายอัน
        inputTopic.pipeInput("sensor-1", SensorReading.builder()
            .sensorId("SENSOR-001")
            .temperature(25.5)
            .humidity(60.0)
            .location("Bangkok")
            .timestamp(Instant.now())
            .status("ACTIVE")
            .build());

        inputTopic.pipeInput("sensor-1", SensorReading.builder()
            .sensorId("SENSOR-001")
            .temperature(27.5)
            .humidity(65.0)
            .location("Bangkok")
            .timestamp(Instant.now())
            .status("ACTIVE")
            .build());

        // ตรวจสอบ output
        assertThat(outputTopic.isEmpty()).isFalse();
        var outputRecords = outputTopic.readRecordsToList();
        assertThat(outputRecords).isNotEmpty();
        
        // ตรวจสอบ state store
        KeyValueStore<String, SensorStats> store =
            testDriver.getKeyValueStore("running-sensor-stats");
        
        SensorStats stats = store.get("SENSOR-001");
        assertThat(stats).isNotNull();
        assertThat(stats.getCount()).isEqualTo(2);
        assertThat(stats.getAvgTemp()).isEqualTo(26.5);
    }

    @Test
    void shouldIgnoreInactiveReadings() {
        inputTopic.pipeInput("sensor-1", SensorReading.builder()
            .sensorId("SENSOR-002")
            .temperature(30.0)
            .status("INACTIVE") // inactive - ควรถูกกรองออก
            .timestamp(Instant.now())
            .build());

        // ตรวจสอบว่าไม่มี output
        assertThat(outputTopic.isEmpty()).isTrue();
    }

    @Test
    void shouldDetectHighTemperatureAnomaly() {
        TestOutputTopic<String, String> anomalyTopic = testDriver.createOutputTopic(
            "anomaly-alerts",
            Serdes.String().deserializer(),
            Serdes.String().deserializer()
        );

        // ส่ง reading ที่อุณหภูมิสูงมาก
        inputTopic.pipeInput("sensor-1", SensorReading.builder()
            .sensorId("SENSOR-003")
            .temperature(42.0) // เกิน 38.0 threshold
            .status("ACTIVE")
            .timestamp(Instant.now())
            .build());

        assertThat(anomalyTopic.isEmpty()).isFalse();
        String anomalyAlert = anomalyTopic.readValue();
        assertThat(anomalyAlert).contains("ANOMALY");
    }
}
```

---

## Application Configuration

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_SERVERS:localhost:9092}
    streams:
      application-id: sensor-processing-app
      properties:
        default.key.serde: org.apache.kafka.common.serialization.Serdes$StringSerde
        default.value.serde: org.apache.kafka.common.serialization.Serdes$StringSerde
        commit.interval.ms: 1000
        num.stream.threads: 4
        replication.factor: 1
        state.dir: /tmp/kafka-streams
    consumer:
      group-id: sensor-consumer-group
      auto-offset-reset: latest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: com.example.streaming.model
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

  cloud:
    stream:
      bindings:
        sensorDataSupplier-out-0:
          destination: sensor-readings
          producer:
            partition-count: 3
        temperatureAlertProcessor-in-0:
          destination: sensor-readings
          group: alert-processor
          consumer:
            concurrency: 3
        temperatureAlertProcessor-out-0:
          destination: sensor-alerts
        alertConsumer-in-0:
          destination: sensor-alerts
          group: alert-consumer
      kafka:
        bindings:
          sensorDataSupplier-out-0:
            producer:
              sync: false
          alertConsumer-in-0:
            consumer:
              start-offset: latest
        streams:
          binder:
            brokers: ${KAFKA_SERVERS:localhost:9092}
            configuration:
              commit.interval.ms: 1000
              num.stream.threads: 2

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,kafka
```

---

## สรุป Part 88

ในส่วนนี้เราได้เรียนรู้:
- **Spring Cloud Stream** - Abstraction layer สำหรับ messaging
- **Kafka Streams Topology** - สร้าง stream processing pipelines
- **Windowed Operations** - Tumbling, Hopping, Session, Sliding windows
- **Stream Joins** - Stream-Table, Stream-Stream, KTable-KTable joins
- **Real-Time Aggregations** - คำนวณ stats แบบ real-time
- **Interactive Queries** - Query state stores โดยตรง
- **Custom Serdes** - สร้าง serializers/deserializers
- **Testing Kafka Streams** - ทดสอบด้วย TopologyTestDriver

---

*[← Part 87: Reactive Security](./part-87-reactive-security.md) | [Part 89: Advanced Patterns →](./part-89-advanced-patterns.md)*
