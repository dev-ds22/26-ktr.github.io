# 1. 개선 현황

 
## 1-1. DB Connection Pool 유지 관리 최적화

### A. 개선 내용
- L4, DB Timeout 등으로 인해 이미 끊어진 Connection이 Pool 내에 남아 있는 상황을 사전 검증
- 유효하지 않은 Connection은 Pool에서 제거하고, Business Logic에는 유효한 Connection만 제공되도록 설정 변경
- testWhileIdle, timeBetweenEvictionRunsMillis, numTestsPerEvictionRun, minEvictableIdleTimeMillis, validationQueryTimeout 등 Connection 검증 및 Evictor 관련 설정 조정

### B. 기대 효과
- L4 또는 DB에 의해 이미 종료된 Connection이 Business Logic에 전달되는 상황 감소
- Connection reset 등 비정상 Connection 사용 오류 감소
- Pool 내부 Connection 상태의 신뢰도 향상

### C. 개선 현황
- DB Connection 비정상 상태 오류 발생 건수 모니터링 비교
#### 개선 전
| server       | 05.25 | 05.26 | 05.27 | 05.28 | 05.29 |
| ------------ | ----- | ----- | ----- | ----- | ----- |
| prd-bk-was01 | 838   | 683   | 663   | 478   | 739   |
| prd-bk-was02 | 668   | 601   | 692   | 450   | 622   |

#### 개선 후
| server       | 06.22 | 06.23 | 06.24 | 06.25 | 06.26 |
| ------------ | ----- | ----- | ----- | ----- | ----- |
| prd-bk-was01 | 0     | 0     | 0     | 0     | 0     |
| prd-bk-was02 | 0     | 0     | 0     | 0     | 0     |

#### D. 평가


## 1-2. 불필요한 다중 트랜잭션(Chained Transaction)구조 해제

### A. 개선 내용
- Main DB와 Mail DB에 중첩 적용된 ChainedTransactionManager 구조 제거

### B. 기대 효과
- Main DB 업무에서 불필요한 Mail DB Connection 점유 감소
- Mail DB Pool 부족이 Main DB 업무 전체 장애로 확산되는 위험 감소
- Transaction 범위 축소로 Connection 보유 시간 감소
- DB Connection 취득 불가 오류 감소

### C. 개선 상황
- DB Connection 취득 불가 오류 발생 건수 모니터링 결과 비교

#### 개선 전
| Server       | 05.25 | 05.26 | 05.27 | 05.28 | 05.29 |
| ------------ | ----- | ----- | ----- | ----- | ----- |
| prd-bk-was01 | 5     | 1     | 10    | 9     | 12    |
| prd-bk-was02 | 10    | 35    | 17    | 29    | 15    |

#### 개선 후
| server       | 06.22 | 06.23 | 06.24 | 06.25 | 06.26 |
| ------------ | ----- | ----- | ----- | ----- | ----- |
| prd-bk-was01 | 0     | 0     | 0     | 0     | 0     |
| prd-bk-was02 | 0     | 0     | 0     | 0     | 0     |

#### D. 평가


## 1-3. 외부 API 통신용 소켓(Soket) 자원 관리 개선

### A. 개선 내용

- 외부 API 통신에 명시적 Socket 반환 로직 추가
- 순차 적용을 위해 외부 API 호출 중 상품 검색 API sendAutoSearch 만 수정

### B. 기대 효과
- 상품 검색 API 호출 후 미반환 Socket 누수 가능성 감소
- CLOSE-WAIT, LAST-ACK 누적 완화
- WAS File Descriptor 고갈 위험 감소
- Socket 자원 반환 누락 여부를 제거로  이후 Socket Pool 적용 효과를 명확히 판단 가능

### C. 개선 상황
- 이하 상품 검색 API 중 개선 반영 한 sendAutoSearch 의 발생 감소.
- 기간에 따른 상품 검색 API 호출 건 수 차이로 전체 상품 검색 API Socket 중 sendAutoSearch 로 인한 오류 발생 비율 비교
#### 개선 전

| server       | 상품 추천 API          | 05.25 | 05.26 | 05.27 | 05.28 | 05.29 |     |
| ------------ | ------------------ | ----- | ----- | ----- | ----- | ----- | --- |
| prd-bk-was01 | sendSearch         | 54    | 72    | 128   | 270   | 149   |     |
| prd-bk-was01 | sendAutoSearch     | 7     | 22    | 39    | 26    | 29    |     |
| prd-bk-was01 | sendTopSearch      | 16    | 49    | 71    | 144   | 69    |     |
| prd-bk-was01 | sendRelationSearch | 2     | 1     | 10    | 14    | 6     |     |
| prd-bk-was01 | TOTAL              | 79    | 144   | 248   | 454   | 253   |     |
| prd-bk-was02 | sendSearch         | 32    | 68    | 107   | 273   | 121   |     |
| prd-bk-was02 | sendAutoSearch     | 7     | 30    | 31    | 27    | 34    |     |
| prd-bk-was02 | sendTopSearch      | 14    | 39    | 72    | 160   | 64    |     |
| prd-bk-was02 | sendRelationSearch | 1     | 2     | 5     | 28    | 8     |     |
| prd-bk-was02 | TOTAL              | 54    | 139   | 215   | 488   | 227   |     |

#### 개선 후

| server       | 상품 추천 API          | 06.22 | 06.23 | 06.24 | 06.25 | 06.26 |
| ------------ | ------------------ | ----- | ----- | ----- | ----- | ----- |
| prd-bk-was01 | sendSearch         | 75    | 76    | 135   | 130   | 249   |
| prd-bk-was01 | sendAutoSearch     | 6     | 13    | 10    | 10    | 10    |
| prd-bk-was01 | sendTopSearch      | 72    | 90    | 131   | 100   | 164   |
| prd-bk-was01 | sendRelationSearch | 4     | 4     | 8     | 13    | 26    |
| prd-bk-was01 | TOTAL              | 157   | 183   | 284   | 253   | 449   |
| prd-bk-was02 | sendSearch         | 72    | 82    | 114   | 125   | 258   |
| prd-bk-was02 | sendAutoSearch     | 5     | 7     | 10    | 11    | 6     |
| prd-bk-was02 | sendTopSearch      | 94    | 115   | 129   | 88    | 151   |
| prd-bk-was02 | sendRelationSearch | 3     | 8     | 7     | 12    | 31    |
| prd-bk-was02 | TOTAL              | 174   | 212   | 260   | 236   | 446   |

#### D. 평가