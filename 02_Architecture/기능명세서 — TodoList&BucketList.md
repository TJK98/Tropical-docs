# [기능 명세서] To-do 및 버킷리스트 관리

- **담당자:** `백승현`
- **최종 수정일:** `2025-09-09`

## 1. 개요

본 기능은 사용자가 자신의 할 일(To-do)과 버킷리스트(Bucket List)를 체계적으로 관리하고 달성 과정을 추적할 수 있는 시스템을 구축하는 것을 목표로 한다. 이 기능은 단기적인 목표 관리와 장기적인 꿈의 실현을 돕는 도구로 활용된다.

 사용자가 **작성한 항목들의 내용은 AI가 사용자의 관심사, 성취, 라이프스타일을 파악하는 중요한 데이터**가 되며, 이를 통해 더욱 정교하고 개인화된 '스몰토크' 주제를 추천하는 데 활용된다.

---

## 2. 데이터베이스 스키마

### 2.1. `Todo` (To-do 리스트) 테이블

| 컬럼명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `todo_id` | `BIGINT` | `PK` | To-do 고유 ID (Auto Increment) |
| `user_id` | `BIGINT` | `FK`, `Not Null` | 사용자 ID (`User` 테이블 참조) |
| `content` | `TEXT` | `Not Null` | 할 일 내용 |
| `due_date` | `DATE` | `Nullable` | 마감 기한 (선택 사항) |
| `is_completed` | `BOOLEAN` | `Default false` | 완료 여부 |
| `created_at` | `DATETIME` | `Default NOW()` | 레코드 생성일 |
| `updated_at` | `DATETIME` | `Default NOW()` | 레코드 수정일 |

### 2.2. `BucketList` (버킷리스트) 테이블

| 컬럼명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `bucket_id` | `BIGINT` | `PK` | 버킷리스트 고유 ID (Auto Increment) |
| `user_id` | `BIGINT` | `FK`, `Not Null` | 사용자 ID (`User` 테이블 참조) |
| `content` | `TEXT` | `Not Null` | 버킷리스트 내용 |
| `is_completed` | `BOOLEAN` | `Default false` | 완료 여부 |
| `created_at` | `DATETIME` | `Default NOW()` | 레코드 생성일 |
| `updated_at` | `DATETIME` | `Default NOW()` | 레코드 수정일 |

---

## 3. 기능 요구사항

### 3.1. To-do 리스트 관리 기능

| 기능 ID | 기능명 | 상세 설명 | API 엔드포인트 (예시) | HTTP Method |
| :--- | :--- | :--- | :--- | :--- |
| `T-C-01` | **To-do 생성** | 사용자는 할 일 내용과 선택적으로 마감 기한을 입력하여 새 To-do를 등록할 수 있다. | `/api/todos` | `POST` |
| `T-R-01` | **To-do 목록 조회** | 전체 To-do 목록을 조회한다. 클라이언트에서는 `is_completed`와 `due_date`를 기준으로 **진행 중/미완료/완료됨** 상태로 구분하여 표시한다. | `/api/todos` | `GET` |
| `T-U-01` | **To-do 수정** | 사용자는 '진행 중' 또는 '미완료' 상태인 To-do의 내용이나 마감 기한을 수정할 수 있다. '미완료' 항목의 마감 기한 갱신(기간 연장)도 이 기능을 통해 처리된다. | `/api/todos/{todo_id}` | `PUT` |
| `T-U-02` | **To-do 완료 처리** | 사용자는 To-do 항목을 '완료됨' 상태로 변경할 수 있다. | `/api/todos/{todo_id}/complete`| `PUT` |
| `T-D-01` | **To-do 삭제** | 사용자는 '진행 중' 또는 '미완료' 상태의 To-do를 삭제할 수 있다. | `/api/todos/{todo_id}` | `DELETE` |

### 3.2. 버킷리스트 관리 기능

| 기능 ID | 기능명 | 상세 설명 | API 엔드포인트 (예시) | HTTP Method |
| :--- | :--- | :--- | :--- | :--- |
| `B-C-01` | **버킷리스트 생성** | 사용자는 새로운 버킷리스트 항목의 내용을 등록할 수 있다. | `/api/buckets` | `POST` |
| `B-R-01` | **버킷리스트 목록 조회**| 전체 버킷리스트 목록을 조회한다. 클라이언트에서는 `is_completed`를 기준으로 **진행 중/완료됨** 상태로 구분하여 표시한다. | `/api/buckets` | `GET` |
| `B-U-01` | **버킷리스트 수정** | 사용자는 '진행 중' 상태인 버킷리스트의 내용을 수정할 수 있다. | `/api/buckets/{bucket_id}` | `PUT` |
| `B-U-02` | **버킷리스트 완료 처리**| 사용자는 버킷리스트 항목을 '완료됨' 상태로 변경할 수 있다. | `/api/buckets/{bucket_id}/complete`| `PUT` |
| `B-D-01` | **버킷리스트 삭제** | 사용자는 '진행 중' 또는 '완료됨' 상태의 버킷리스트 항목을 삭제할 수 있다. | `/api/buckets/{bucket_id}` | `DELETE` |

---
