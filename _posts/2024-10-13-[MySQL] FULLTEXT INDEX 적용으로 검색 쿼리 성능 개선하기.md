---
title: "[MySQL] FULLTEXT INDEX 적용으로 검색 쿼리 성능 개선하기"
date: 2024-10-13 03:27:00 +09:00
categories: [프로젝트]
tags: [데이터베이스, MySQL, 인덱스]
image:
  path: /assets/img/thumbnail/mysql.png
---

## 도입 배경

`BODA`는 사용자의 일상 및 여행 관련 정보에 따라서 사용자 맞춤형으로 여행지를 추천해주는 서비스이다.

처음 찾았던 여행지 데이터에는 10만여개의 여행지 데이터가 있었지만 중복이 꽤 많았고 의미없는 데이터도 다수 있었다.

의미있는 데이터만 추려서 가공하니 3만여개의 여행지 데이터만 남았고 이를 가지고 데이터베이스를 구축하였다.

이렇게 여행지 데이터베이스를 구축하고 나니 그냥 기존 기획대로 여행지 추천만 하기에는 아쉽게 느껴졌다.

그래서 팀원들과 함께 상의해, 여행지를 검색할 수 있는 기능을 추가하기로 하였다.

![Image](https://github.com/user-attachments/assets/5e5a6c2a-9560-4106-abca-b9e52cd34f71)

### SpotRepository.java

```java
public interface SpotRepository extends JpaRepository<Spot, Long> {
    Page<Spot> findByNameContaining(String name, Pageable pageable);
}
```

위 `findByNameContainin`메서드가 처음 여행지 검색을 위해서 구현한 메서드이다.

위 JPA 메서드를 통해서 ‘제주’라고 검색한다고 가정했을 때, 아래와 같은 SQL이 생성될 것이다. (정렬 기준은 생략)

```sql
SELECT * 
FROM spot
WHERE name LIKE '%제주%'
LIMIT :pageSize OFFSET :offset;
```

이때 위와 같이 MySQL의 `LIKE '% %'` 구문이 검색에 활용되게 된다.

하지만 LIKE 명령어의 경우 FULL SCAN 방식이기 때문에 모든 데이터를 전부 찾아보게 된다.

이때 검색할 데이터가 클 경우, 과부하가 발생할 수 있고 응답시간이 매우 길어질 수 있다.

> **이러한 문제를 해결하기 위해서 MySQL의 FULLTEXT INDEX 방식을 도입하기로 결정하였다.**

---

## FULLTEXT INDEX란?

FULLTEXT INDEX는 데이터베이스(MySQL, Oracle 등)에서 지원하는 기능으로, 앞선 FULL SCAN 문제를 막을 수 있는 인덱스 기법이다.

다른 일반 인덱스와의 차이점은 char, varchar, text 타입 문자만 인덱싱이 가능하다는 것이다.

FULLTEXT INDEX에서 데이터를 인덱싱하는 방법에는 두가지가 존재한다.

### **Stop-word parser**

- stop word(구분자)를 기준으로 토큰을 나누는 기법
- 일반적으로 공백이나 문장 기호 혹은 사용자가 지정한 특정 단어를 기준으로 토큰을 나눔
- 검색 성능은 떨어짐

```
사용자 맞춤 여행지를 추천해 드려요 -> 사용자 / 맞춤 / 여행지를 / 추천해 / 드려요
```

### **N-gram Parser**

- N크기로 데이터를 인덱스로 파싱해두었다가 사용하는 기법
- stop word에 비해 인덱스의 크기가 매우 커지지만 검색 성능은 좋음

```
N : 2인 경우
사용자 맞춤 여행지를 추천해 드려요 -> 사용 / 용자 / 자맞 / 맞춤 / 춤여 / 여행 / 행지 / 지를 / 를추
 / 추천 / 천해 / 해드 / 드려 / 려요
```

---

검색 모드에도 대표적으로 두가지 방식이 존재하는데 아래 두가지 방식이다.

### NATURAL LANGUAGE MODE

- 검색 문자열을 단어 단위로 분리한 후, 해당 단어 중 하나라도 포함되는 행을 찾음

```sql
SELECT * FROM TABLE
WHERE MATCH(column) AGAINST ('keyword' IN NATURAL LANUGUAGE MODE)
```

### BOLEAN MODE

- 검색 문자열을 단어 단위로 분리한 후, 해당 단어가 포함되는 행을 찾는 규칙을 추가적으로 적용하여 해당 규칙에 매칭되는 행을 찾음

```sql
SELECT * FROM TABLE
WHERE MATCH(column) AGAINST ('keyword' IN BOOLEAN MODE)
```

---

## 여행지 검색에 FULLTEXT INDEX 적용하기

여행지 검색 기능에는 검색 시간 뿐만 아니라 검색 성능도 중요하다고 생각이 들어 **`N-gram Parser`** 방식을 선택하였다.

```sql
ALTER TABLE spot ADD FULLTEXT (name) WITH PARSER ngram;
```

기존의 DDL은 수정하지 않고 마지막에 위와 같은 SQL문을 추가로 넣어줘서 spot 테이블의 name 컬럼에 FULLTEXT INDEX를 적용해 주었다.

```sql
ALTER TABLE spot DROP INDEX FULLTEXT (name);
```

이와 같은 SQL을 통해서 삭제할 수도 있다.

```java
@Query(value = "SELECT * FROM spot WHERE MATCH(name) AGAINST(:name)",
        countQuery = "SELECT COUNT(*) FROM spot WHERE MATCH(name) AGAINST(:name)",
        nativeQuery = true)
Page<Spot> searchByName(@Param("name") String name, Pageable pageable);
```

이렇게 FULLTEXT INDEX를 적용해준 뒤, 이를 검색 쿼리에 적용하기 위해서 @Query 어노테이션을 활용해 새로운 여행지 검색 쿼리를 작성해 주었다.

검색 모드로는 default 값인 자연어 모드를 사용하였다.

---

## FULLTEXT INDEX 적용 전후 성능 비교

FULLTEXT INDEX 적용 전후 여행지 검색 쿼리의 성능을 간단하게 비교해보았다.

DB는 AWS RDS의 프리티어 인스턴스 위에 올라가 있는 상태로, MySQL 워크벤치를 통해 비교를 진행해 보았다.

- FULLTEXT INDEX 적용 전
    
    ![Image](https://github.com/user-attachments/assets/8c5d8307-aca6-4304-ae02-c4f989d888e5)
    
- FULLTEXT INDEX 적용 후
    
    ![Image](https://github.com/user-attachments/assets/059e17d1-f2bb-4bef-9fc5-f5a245b20794)
    

제주라는 검색어를 통해서 두가지 검색 쿼리를 날려보고 시간을 비교해보았다.

`FULLTEXT INDEX 적용 전`, 즉 LIKE 쿼리를 통해서 검색을 하였을 경우는 `0.032초`가 나왔다.

`FULLTEXT INDEX 적용 후`, 인덱스를 통해서 검색을 하였을 경우는 `0.016초`가 나왔다.

2배가량 응답 시간이 개선되기는 하였지만 생각보다 드라마틱한 차이는 아니었다. 아마 기존의 3만여개 데이터에서 LIKE 쿼리를 통해서 검색을 하는 것이 생각보단 부하가 크지 않았던 것 같다. 생각해보니 데이터 3만개 정도에 DB가 느려진다면 아무도 그 DB를 쓰지 않을 것이라는 생각이 들긴 했다.. 그렇지만 큰 차이가 나지는 않아 조금은 아쉬웠다…

---

## 마치며

이렇게 기존 LIKE 쿼리를 통해서 검색을 구현하였던 것을 FULLTEXT INDEX를 적용해 개선해보았다.

그간 JPA를 자주 사용하면서 직접 SQL를 만질 일이 자주 없었는데, 오랜만에 직접 찾아보고 또 쿼리를 작성해보고 하니 생각보다 재미있었던 것 같다.

또 이렇게 찾아보니 생각보다 데이터베이스에서 지원하는 기능들이 많다는 것을 알 수 있었다.

데이터베이스 기능들을 더 찾아보고 공부해야겠다는 생각이 들었다.