---
title: 'InfluxDB 쿼리 최적화와 Java 병렬 처리로 실시간 데이터 처리 성능 개선'
excerpt: ''

categories:
  - Performance Optimization
tags:
  - [tag1, tag2]

permalink: /performance-optimization/parallel-stream/

toc: true
toc_sticky: true

date: 2024-11-14
last_modified_at: 2024-11-14
---

대규모 데이터를 실시간으로 조회하고 처리하는 기능은 서비스의 성능과 사용자 경험에 중요한 영향을 미칩니다. 이번 포스트에서는 사내 프로젝트에서 InfluxDB(version 1.8.10)와 PostgreSQL를 사용해 실시간 데이터 조회 성능을 최적화한 과정을 공유합니다. 특히 `InfluxDB 쿼리 최적화`와 `Java 병렬 처리`를 통해 주요 성능 병목을 제거한 사례를 중심으로 설명합니다. 코드 유출을 방지하기 위해 주요 로직은 생략하였으며, 일부 내용을 간략화하고 수정하여 작성했습니다.

## 문제 상황
현재 관측 데이터를 조회하는 서비스에서 InfluxDB와 PostgreSQL 모두 성능 병목 현상이 발생하며, 평균 응답 속도가 `약 3.27초`로 측정되었습니다. 이는 실시간 처리가 중요한 서비스의 요구사항을 만족시키지 못했습니다.

## 기존 구현 방식 (AS IS)
관측 데이터를 조회하는 서비스는 다음 두 가지 작업을 수행합니다.

1. Influx DB 에서 관측값 조회
2. PostgreSQL 에서 만조, 간조 데이터 조회 후 밀물, 썰물 데이터 구성


### 1. 지점 메타정보와 관측값 매핑

- 지점 메타정보를 조회합니다. (List 타입)
- 관측값을 조회하기 위해 세 가지 테이블(AWS, SEA_VS, LS)에서 데이터를 읽어온 후 반복문으로 지점 메타정보와 관측 데이터를 매핑합니다.
- 문제점
    - 모든 관측 데이터를 조회한 뒤 필터링
    - 중복 데이터 탐색으로 인해 O(N²) 시간 복잡도 발생

```java
// 1. 지점 메타정보 조회
List<Map<String, Object> stationList = stationMapper.selectStationList(param);

// 2. 기준 시간 설정 (현재 시간 기준으로 12시간 전)
ZonedDateTime zdt = baseTm.atZone(ZoneId.of("Asia/Seoul"));
String standardTime = zdt.minusHours(12)
        .format(DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ssXXX"));

// 3. InfluxDB 쿼리 생성 (AWS 테이블에서 관측 데이터 조회)
StringBuilder query = new StringBuilder();
sql.append(" SELECT obsTime, stnId, vs");
sql.append(" FROM AWS ");
sql.append(" WHERE time >= '" + standardTime + "' and vi > 0");
sql.append(" GROUP BY stnId ");
sql.append(" ORDER BY time desc ");
sql.append(" LIMIT 1 ");

// 4. 생성한 쿼리를 InfluxDB에 실행하여 결과 조회
List<Map<String, Object>> obsAwsList = influxDBService.getObserveList(query.toString());
for (Map<String, Object> vs : stationList) {
    String obsStnId = (vs.get("obsStnId")).toString();

    // 5. 모든 관측 데이터를 지점 메타정보와 매핑
    for (Map<String, Object> obsAws : obsAwsList) {
        if (StringUtils.equals(obsStnId, (String) obsAws.get("stnId"))) {
            ...(파싱)
        }
    }
}

...(SEA_VS, LS 테이블도 동일한 방식으로 처리)
```

### 2. 밀물/썰물 상태 확인

- PostgreSQL 에서 지점별 만조, 간조 시간을 조회한 뒤 현재 시각과 비교하여 밀물/썰물 상태를 계산했습니다.
- 문제
    - 동기적 반복문으로 인해 직렬 처리
    - 중복되는 로직으로 코드 가독성과 유지보수성이 떨어짐

```java
for (Map<String, Object> station : stationList) {
    // 1. 현재 지점별로 만조 및 간조 시간 계산
    Map<String, Object> tideDayData = getTideDayData(param, station, baseTm); // 만조, 간조 데이터를 조회
    LocalTime currentTime = baseTm.toLocalTime(); // 현재 시간을 기준으로 계산 수행
    // 2. 만조 및 간조 데이터가 존재할 경우 밀물/썰물 상태 계산
    if (tideDayData != null && !tideDayData.isEmpty()) {
        // 만조 및 간조 시간과 현재 시간을 비교하여 밀물/썰물 상태를 결정
        ...
        station.put("tideStatus", tideStatus); // 계산된 밀물/썰물 상태를 지점 정보에 추가
    }
}
```

## 개선 내용 및 성과 (TO BE)

### InfluxDB 최적화

```java
// 1. 지점 메타정보 조회
List<Map<String, Object> stationList = stationMapper.selectStationList(param);

// 2. 관측소 ID 리스트 추출
// stationList에서 "obsStnId"를 추출하여 String 리스트로 변환
List<String> obsStnIdList = stationList.stream()
                        .map(map -> map.get("obsStnId").toString())
                        .collect(Collectors.toList());

// 3. 기준 시간 설정 (현재 시간 기준으로 12시간 전)
ZonedDateTime zdt = base_tm.atZone(ZoneId.of("Asia/Seoul"));
String baseTimeToStr = zdt.minusHours(12)
        .format(DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ssXXX"));

// 4. InfluxDB 쿼리 생성
StringBuilder sql = new StringBuilder();
sql.append(" SELECT obsTime, stnId, vs");
sql.append(" FROM AWS ");
sql.append(" WHERE time >= '" + standardTime + "' and vi > 0");

// 관측소 ID 리스트를 동적으로 SQL 조건에 추가
sql.append(" AND (");
for (int i = 0; i < obsStnIdList.size(); i++) {
    sql.append(" stnId = '" + obsStnIdList.get(i) + "'");
    if (i < obsStnIdList.size() - 1) {
        sql.append(" OR ");
    }
}
sql.append(" )");
sql.append(" GROUP BY stnId ");
sql.append(" ORDER BY time desc ");
sql.append(" LIMIT 1 ");

// 5. 생성한 쿼리를 InfluxDB에 실행하여 결과 조회
List<Map<String, Object>> obsAwsList = influxDBService.getObserveList(query.toString());
// 6. 관측 데이터를 Map 형태로 변환
// "stnId"를 키로 하여 관측 데이터를 Map에 저장
Map<String, Map<String, Object>> obsAwsMap = obsAwsList.stream()
        .collect(Collectors.toMap(
                obs -> obs.get("stnId").toString(),
                Function.identity()
        ));

// 7. 관측 데이터를 지점 메타정보와 매핑
for (Map<String, Object> station : stationList) {
    String obsStnId = (station.get("obsStnId")).toString();
    if (obsAwsMap.containsKey(obsStnId)) {
        ...(파싱)
    }
}
```

1. 반복문 횟수 감소
    - 기존 코드에서는 stationList 의 각 항목에 대해 obsAwsList 를 반복 탐색하며 지점 아이디를 기준으로 매칭
    - 개선 코드에서는 obsAwsList 를 `Map` 자료구조로 변환하여 key-value 기반 탐색으로 변경
    - HashMap 을 사용한 매칭은 시간 복잡도가 O(1)로, 기존 중첩 반복문에서 발생하는 O(N²) 의 시간 복잡도를 대폭 줄임
2. SQL 필터링 조건 개선
    - 기존 코드에서는 모든 데이터를 조회 후 필터링
    - 개선 코드에는 obsStnIdList 에 포함된 stnId 만을 조회하는 조건을 동적으로 추가하여 데이터 검색 범위 축소
3. 데이터 스트리밍 활용
    - obsStnIdList 를 추출하고, obsAwsList 를 Map 으로 변환하는 작업에 Java Stream API 를 활용하여 간결하고 효율적으로 처리

기존 코드에서는 InfluxDB에서 관측 데이터를 조회하는 데 `약 2900ms`가 소요되었습니다. 이는 데이터 처리와 매핑 과정에서 발생하는 반복 연산과 불필요한 데이터 조회가 주요 병목 원인이었습니다. 개선된 코드에서는 쿼리 조건을 동적으로 생성하여 불필요한 데이터를 사전에 필터링하고, 관측 데이터를 Map 자료구조로 변환해 탐색 효율을 크게 높였습니다. 이러한 최적화 작업을 통해 처리 시간을 `약 251ms`로 단축했으며, 전체적으로 `약 91.38%`의 성능 개선을 달성할 수 있었습니다.

### 밀물/썰물 계산 최적화

```java
stationList.parallelStream().forEach(station -> {
    Map<String, Object> tideDayData = getTideDayData(param, map, base_tm);
    if (tideDayData == null || tideDayData.isEmpty()) return;
    // 1. 각 지점별로 만조, 간조 시간 계산
    ...
    // 2. 현재 시간이 밀물/썰물인지 확인하는 메서드
    String tideStatus = determineTideStatus(...)
    station.put("tideStatus", tideStatus);
});

private String determineTideStatus(LocalTime currentTime ...) {
    ...
}
```

1. 동기 처리 -> 병렬 처리
    - 기존 코드에서는 for 반복문을 사용해 모든 지점을 동기적으로 처리하여 작업 속도가 느림
    - 각 지점의 로직이 독립적이고 순서 의존성이 없기 때문에, 개선된 코드에서는 `parallelStream()`을 활용해 병렬 처리
2. 중복 로직 분리
    - 조석 상태(밀물/썰물)를 계산하는 로직을 별도의 메서드`(determineTideStatus)`로 분리
    - 이를 통해 코드 가독성과 유지보수성이 향상되었고, 로직 변경 시 유연하게 대처 가능해짐
3. 안전성 강화
    - 데이터가 없거나 잘못된 값이 들어왔을 때의 상황을 처리하여 안정성을 높임

기존 코드에서는 모든 지점의 밀물과 썰물 상태를 계산하는 데 `약 2300ms`가 소요되었습니다. 이는 작업이 동기적으로 처리되면서 직렬 실행으로 인한 성능 병목이 발생했기 때문입니다.
개선된 코드에서는 병렬 처리`(parallelStream)`를 도입하여 각 작업을 동시에 수행할 수 있도록 변경하였고, 이를 통해 대규모 데이터를 효율적으로 처리할 수 있었습니다.

그 결과, 개선된 코드의 처리 시간은 `약 530ms`로 감소했으며, 기존 대비 `약 76.96%`의 성능 개선을 달성했습니다.

## 결론
이번 최적화 작업을 통해 InfluxDB 쿼리 개선과 Java 병렬 처리를 적용하여, 기존에 `약 3.27초`가 소요되던 작업을 `약 0.518초`로 단축했습니다. 이는 전체 처리 시간을 `약 84.17%` 감소시킨 것으로, 실시간 처리가 중요한 서비스의 성능과 사용자 경험을 크게 향상하는 데 기여했습니다.
