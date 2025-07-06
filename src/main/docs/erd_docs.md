# Alex Xu S3-like Object Storage ERD 상세 테이블 설명

## Core Metadata Tables (핵심 메타데이터)

### BUCKET_METADATA

**역할**: S3 버킷의 메타데이터와 설정 정보를 저장하는 핵심 테이블

**주요 필드 상세**:
- `bucket_id (UUID, PK)`: 전역적으로 유니크한 버킷 식별자
- `bucket_name (VARCHAR, UK)`: DNS 호환 버킷 이름, 전역 유니크 제약
- `region (VARCHAR)`: 버킷이 생성된 지리적 리전 (us-east-1, eu-west-1 등)
- `creation_time (TIMESTAMP)`: 버킷 생성 시각, 불변 필드
- `access_policies (JSON)`: IAM 정책 문서, 버킷 레벨 권한 정의
- `storage_class (VARCHAR)`: 기본 스토리지 클래스 (STANDARD, IA, GLACIER)
- `versioning_enabled (BOOLEAN)`: 객체 버전 관리 활성화 여부
- `total_size_bytes (BIGINT)`: 버킷 내 모든 객체의 총 크기, 비동기 업데이트
- `object_count (BIGINT)`: 버킷 내 객체 개수, 성능 최적화용 필드
- `owner_id (UUID, FK)`: 버킷 소유자 사용자 ID
- `lifecycle_policies (JSON)`: 자동 삭제/아카이빙 정책 정의
- `encryption_config (VARCHAR)`: 기본 암호화 설정
- `public_read_access (BOOLEAN)`: 퍼블릭 읽기 권한 여부
- `cors_config (VARCHAR)`: Cross-Origin Resource Sharing 설정
- `last_modified (TIMESTAMP)`: 마지막 수정 시각

**비즈니스 로직**:
- 버킷명은 DNS 규칙을 따라야 함 (소문자, 하이픈 허용, 점으로 시작/끝날 수 없음)
- 리전당 버킷 생성 제한 적용 (계정당 1000개 기본 제한)
- `total_size_bytes`와 `object_count`는 객체 변경 시 비동기로 업데이트
- 버킷 삭제 시 모든 객체와 버전이 삭제되어야 함

**성능 고려사항**:
- `bucket_name`에 B-tree 인덱스 (빠른 조회 및 유니크 제약)
- `owner_id`에 인덱스 (사용자별 버킷 목록 조회)
- `region`에 인덱스 (지역별 버킷 필터링)
- 자주 변경되는 `total_size_bytes`는 별도 캐시 테이블 고려

**관련 테이블**:
- USERS (owner_id 외래키)
- OBJECT_METADATA (bucket_id 외래키)
- MULTIPART_UPLOADS (bucket_id 외래키)
- REPLICATION_POLICIES (bucket_id 외래키)

---

### OBJECT_METADATA

**역할**: 객체의 메타데이터를 저장하는 테이블 (알렉스쉬 설계의 핵심)

**주요 필드 상세**:
- `object_id (UUID, PK)`: 객체의 전역 고유 식별자
- `bucket_id (UUID, FK)`: 소속 버킷 ID
- `object_name (VARCHAR)`: 버킷 내 객체 키, 경로 정보 포함
- `size_bytes (BIGINT)`: 객체 전체 크기 (압축 전 원본 크기)
- `creation_time (TIMESTAMP)`: 최초 생성 시각
- `etag (VARCHAR)`: MD5 해시 값, 무결성 검증 및 캐시 제어용
- `storage_class (VARCHAR)`: 스토리지 등급 (STANDARD, IA, GLACIER, DEEP_ARCHIVE)
- `content_type (VARCHAR)`: MIME 타입 (image/jpeg, text/plain, application/json 등)
- `content_encoding (VARCHAR)`: 압축 방식 (gzip, deflate, br 등)
- `content_language (VARCHAR)`: 콘텐츠 언어 (ko-KR, en-US 등)
- `custom_metadata (JSON)`: 사용자 정의 메타데이터 (키-값 쌍)
- `cache_control (VARCHAR)`: HTTP 캐시 제어 헤더
- `expires (TIMESTAMP)`: 콘텐츠 만료 시각
- `server_side_encryption (VARCHAR)`: 서버 측 암호화 방식 (AES256, KMS 등)
- `version_id (VARCHAR)`: 버전 식별자, 버전 관리 활성화 시 사용
- `is_latest_version (BOOLEAN)`: 최신 버전 여부
- `is_delete_marker (BOOLEAN)`: 삭제 마커 여부 (버전 관리 시 사용)
- `checksum_sha256 (VARCHAR)`: SHA256 체크섬, 추가 무결성 검증
- `multipart_upload_id (UUID, FK)`: 멀티파트 업로드 ID (해당하는 경우)
- `last_accessed (TIMESTAMP)`: 마지막 접근 시각, 라이프사이클 정책용
- `user_id (UUID, FK)`: 객체 소유자 사용자 ID

**비즈니스 로직**:
- `object_name`은 버킷 내에서 유니크해야 함
- 버전 관리 활성화 시 동일한 `object_name`에 여러 `version_id` 존재 가능
- `etag`는 단일 업로드와 멀티파트 업로드에서 계산 방식이 다름
- 객체 삭제 시 실제 데이터 삭제가 아닌 `is_delete_marker` 플래그 설정
- `last_accessed` 필드는 GET 요청 시에만 업데이트 (성능상 이유)

**성능 고려사항**:
- `(bucket_id, object_name)` 복합 인덱스 (가장 빈번한 조회 패턴)
- `last_accessed` 인덱스 (라이프사이클 정책 실행용)
- `storage_class` 인덱스 (비용 분석 및 마이그레이션용)
- `user_id` 인덱스 (사용자별 객체 조회)
- 대용량 테이블이므로 `bucket_id` 기반 샤딩 필수

**관련 테이블**:
- BUCKET_METADATA (bucket_id 외래키)
- DATA_BLOCKS (object_id 외래키)
- OBJECT_VERSIONS (object_id 외래키)
- MULTIPART_UPLOADS (multipart_upload_id 외래키)

---

## Data Storage Layer (데이터 저장 계층)

### DATA_BLOCKS

**역할**: 객체를 4MB 블록으로 분할하여 저장하는 테이블 (알렉스쉬의 핵심 아키텍처)

**주요 필드 상세**:
- `block_id (UUID, PK)`: 블록의 전역 고유 식별자
- `object_id (UUID, FK)`: 소속 객체 ID
- `block_hash (VARCHAR)`: 블록 내용의 SHA256 해시, 중복 제거용
- `block_size_bytes (BIGINT)`: 블록 크기, 최대 4MB (4,194,304 bytes)
- `block_sequence_number (INT)`: 객체 내 블록 순서, 0부터 시작
- `compression_algorithm (VARCHAR)`: 압축 알고리즘 (gzip, lz4, zstd, none 등)
- `compressed_size_bytes (BIGINT)`: 압축 후 실제 저장 크기
- `replica_locations (JSON)`: 복제본이 저장된 스토리지 노드 ID 배열
- `erasure_coding_schema (VARCHAR)`: EC 스키마 (4+2, 6+3, 10+4 등)
- `erasure_coding_chunks (JSON)`: EC 청크별 위치 정보 및 메타데이터
- `created_at (TIMESTAMP)`: 블록 생성 시각
- `integrity_checksum (VARCHAR)`: 무결성 검증용 추가 체크섬
- `storage_tier (VARCHAR)`: 스토리지 계층 (HOT, WARM, COLD)
- `is_cached (BOOLEAN)`: 캐시 여부 플래그
- `cache_locations (JSON)`: 캐시된 노드 위치 정보

**비즈니스 로직**:
- 객체는 순차적으로 4MB 블록으로 분할 (마지막 블록은 4MB 미만 가능)
- Erasure Coding 4+2: 4개 데이터 블록 + 2개 패리티 블록 = 총 6개 청크
- 동일한 `block_hash`를 가진 블록은 물리적으로 한 번만 저장 (중복 제거)
- 블록별 독립적인 무결성 검증 수행
- 압축은 블록 단위로 적용, 압축률에 따라 `compressed_size_bytes` 결정

**성능 고려사항**:
- `(object_id, block_sequence_number)` 복합 인덱스 (순차 읽기 최적화)
- `block_hash` 인덱스 (중복 제거 및 참조 무결성용)
- `storage_tier` 인덱스 (계층별 스토리지 관리)
- 파티셔닝: `object_id` 해시 기반 샤딩
- 읽기 성능을 위해 블록 위치 정보 캐싱 필수

**관련 테이블**:
- OBJECT_METADATA (object_id 외래키)
- STORAGE_NODES (replica_locations에서 참조)

---

### STORAGE_NODES

**역할**: 물리적 스토리지 노드의 상태와 용량을 관리하는 테이블

**주요 필드 상세**:
- `node_id (VARCHAR, PK)`: 노드 고유 식별자 (호스트명 기반)
- `region (VARCHAR)`: 노드가 위치한 지리적 리전
- `availability_zone (VARCHAR)`: 가용성 영역 (AZ-1a, AZ-1b 등)
- `datacenter_id (VARCHAR)`: 데이터센터 식별자
- `node_type (VARCHAR)`: 노드 유형 (DATA, PARITY, CACHE, METADATA)
- `total_capacity_bytes (BIGINT)`: 총 저장 용량
- `used_capacity_bytes (BIGINT)`: 현재 사용 중인 용량
- `available_capacity_bytes (BIGINT)`: 사용 가능한 용량 (계산 필드 가능)
- `status (VARCHAR)`: 노드 상태 (ONLINE, OFFLINE, MAINTENANCE, DEGRADED)
- `last_heartbeat (TIMESTAMP)`: 마지막 하트비트 수신 시각
- `cpu_utilization (DOUBLE)`: CPU 사용률 (0.0-1.0)
- `memory_utilization (DOUBLE)`: 메모리 사용률 (0.0-1.0)
- `disk_utilization (DOUBLE)`: 디스크 사용률 (0.0-1.0)
- `network_bandwidth_mbps (DOUBLE)`: 네트워크 대역폭 (Mbps)
- `raid_config (VARCHAR)`: RAID 구성 (RAID5, RAID6, JBOD 등)
- `performance_metrics (JSON)`: 상세 성능 지표 (IOPS, 지연시간 등)
- `hardware_specs (VARCHAR)`: 하드웨어 사양 정보

**비즈니스 로직**:
- 노드 장애 감지: `last_heartbeat`가 5분 초과 시 장애 의심
- 용량 기반 로드 밸런싱: `available_capacity_bytes` 고려하여 배치
- 성능 기반 배치: `disk_utilization` < 80% 노드 우선 선택
- 지역별 분산: 동일 객체의 복제본을 다른 AZ에 배치
- 노드 유형별 역할 분리: DATA 노드와 PARITY 노드 구분 관리

**성능 고려사항**:
- `(region, availability_zone, status)` 복합 인덱스 (활성 노드 조회)
- `status` 인덱스 (상태별 노드 필터링)
- `node_type` 인덱스 (노드 유형별 관리)
- 실시간 업데이트가 많으므로 메모리 캐시 필수
- 하트비트 정보는 별도 시계열 DB 고려

**관련 테이블**:
- DATA_BLOCKS (replica_locations에서 참조)
- PERFORMANCE_METRICS (node_id 외래키)

---

## Multipart Upload Support (멀티파트 업로드)

### MULTIPART_UPLOADS

**역할**: 대용량 파일의 멀티파트 업로드 세션을 관리하는 테이블

**주요 필드 상세**:
- `upload_id (UUID, PK)`: 업로드 세션의 전역 고유 식별자
- `object_id (UUID, FK)`: 최종 생성될 객체 ID
- `bucket_id (UUID, FK)`: 대상 버킷 ID
- `object_key (VARCHAR)`: 객체 키 (최종 저장될 이름)
- `storage_class (VARCHAR)`: 저장 클래스 지정
- `initiated_at (TIMESTAMP)`: 멀티파트 업로드 시작 시각
- `expires_at (TIMESTAMP)`: 업로드 만료 시각 (기본 7일 후)
- `status (VARCHAR)`: 업로드 상태 (INITIATED, IN_PROGRESS, COMPLETED, ABORTED)
- `upload_metadata (JSON)`: 업로드 관련 메타데이터 (content-type 등)
- `encryption_key_id (VARCHAR)`: 서버 측 암호화 키 ID
- `total_size_estimate (BIGINT)`: 예상 총 파일 크기
- `total_parts_count (INT)`: 예상 총 파트 수
- `user_id (UUID, FK)`: 업로드 요청 사용자 ID
- `session_token (VARCHAR)`: 인증 세션 토큰

**비즈니스 로직**:
- 업로드 시작 시 고유한 `upload_id` 생성 및 반환
- 7일 후 자동 만료, 미완료 업로드 정리 작업 수행
- 모든 파트 업로드 완료 후 Complete Multipart Upload API 호출 시 COMPLETED 상태로 변경
- 완료 시 개별 파트들을 `part_number` 순서대로 병합하여 하나의 객체 생성
- 중단된 업로드는 ABORTED 상태로 변경 후 관련 파트 정리

**성능 고려사항**:
- `(bucket_id, object_key)` 복합 인덱스 (동일 객체 업로드 충돌 방지)
- `expires_at` 인덱스 (만료된 업로드 정리 작업용)
- `status` 인덱스 (상태별 업로드 관리)
- `user_id` 인덱스 (사용자별 업로드 목록 조회)

**관련 테이블**:
- BUCKET_METADATA (bucket_id 외래키)
- OBJECT_METADATA (object_id 외래키)
- UPLOAD_PARTS (upload_id 외래키)
- USERS (user_id 외래키)

---

### UPLOAD_PARTS

**역할**: 멀티파트 업로드의 개별 파트 정보를 저장하는 테이블

**주요 필드 상세**:
- `part_id (UUID, PK)`: 파트의 고유 식별자
- `upload_id (UUID, FK)`: 소속 멀티파트 업로드 세션 ID
- `part_number (INT)`: 파트 번호 (1부터 시작, 최대 10,000)
- `etag (VARCHAR)`: 파트의 MD5 해시값 (무결성 검증용)
- `size_bytes (BIGINT)`: 파트 크기 (5MB~5GB, 마지막 파트는 예외)
- `uploaded_at (TIMESTAMP)`: 파트 업로드 완료 시각
- `storage_location (VARCHAR)`: 파트가 저장된 스토리지 노드 ID
- `checksum_sha256 (VARCHAR)`: SHA256 체크섬 (추가 무결성 검증)
- `status (VARCHAR)`: 파트 상태 (UPLOADED, FAILED, PENDING)
- `part_metadata (JSON)`: 파트별 추가 메타데이터
- `upload_session_id (VARCHAR)`: 업로드 세션 식별자 (중복 업로드 방지)

**비즈니스 로직**:
- 파트 크기 제한: 최소 5MB (마지막 파트 제외), 최대 5GB
- 파트 번호는 연속적일 필요 없음 (예: 1, 3, 5 가능)
- Complete Multipart Upload 시 `part_number` 순서대로 병합
- 실패한 파트는 동일한 `part_number`로 재업로드 가능
- 각 파트는 독립적으로 업로드 및 검증

**성능 고려사항**:
- `(upload_id, part_number)` 복합 인덱스 (파트 순서 조회)
- `status` 인덱스 (실패한 파트 재시도 관리)
- `uploaded_at` 인덱스 (최근 업로드 파트 조회)
- 파트 정보는 업로드 완료 후 일정 기간 후 정리

**관련 테이블**:
- MULTIPART_UPLOADS (upload_id 외래키)

---

## Versioning and Lifecycle (버전 관리 및 라이프사이클)

### OBJECT_VERSIONS

**역할**: 객체의 모든 버전을 추적하고 관리하는 테이블

**주요 필드 상세**:
- `version_id (UUID, PK)`: 버전의 고유 식별자
- `object_id (UUID, FK)`: 소속 객체 ID
- `version_tag (VARCHAR)`: 버전 태그 (v1, v2, timestamp 등)
- `is_current_version (BOOLEAN)`: 현재 활성 버전 여부
- `version_created_at (TIMESTAMP)`: 해당 버전 생성 시각
- `size_bytes (BIGINT)`: 해당 버전의 파일 크기
- `etag (VARCHAR)`: 버전별 ETag 값
- `storage_location (VARCHAR)`: 버전 데이터 저장 위치
- `is_delete_marker (BOOLEAN)`: 삭제 마커 여부
- `created_by_user_id (VARCHAR)`: 버전 생성 사용자 ID
- `version_metadata (VARCHAR)`: 버전별 추가 메타데이터
- `parent_version_id (UUID, FK)`: 이전 버전 ID (버전 체인)
- `lifecycle_status (VARCHAR)`: 라이프사이클 상태 (ACTIVE, ARCHIVED, DELETED)

**비즈니스 로직**:
- 객체 업데이트 시 기존 버전은 `is_current_version = false`로 변경
- 새 버전은 `is_current_version = true`로 설정
- 객체 삭제 시 삭제 마커 버전 생성 (`is_delete_marker = true`)
- 버전 간 부모-자식 관계 유지 (`parent_version_id`)
- 라이프사이클 정책에 따라 오래된 버전 자동 아카이빙/삭제

**성능 고려사항**:
- `(object_id, is_current_version)` 복합 인덱스 (현재 버전 조회)
- `version_created_at` 인덱스 (시간 기반 버전 관리)
- `lifecycle_status` 인덱스 (라이프사이클 정책 실행)
- 대용량 버전 히스토리의 경우 아카이빙 전략 필요

**관련 테이블**:
- OBJECT_METADATA (object_id 외래키)
- OBJECT_VERSIONS (parent_version_id 자기 참조)

---

## Replication and Disaster Recovery (복제 및 재해 복구)

### REPLICATION_POLICIES

**역할**: 버킷별 복제 정책을 정의하고 관리하는 테이블

**주요 필드 상세**:
- `policy_id (UUID, PK)`: 복제 정책의 고유 식별자
- `bucket_id (UUID, FK)`: 대상 버킷 ID
- `policy_name (VARCHAR)`: 정책 이름 (사용자 정의)
- `source_region (VARCHAR)`: 소스 리전
- `destination_region (VARCHAR)`: 복제 대상 리전
- `replication_type (VARCHAR)`: 복제 유형 (CRR: Cross-Region, SRR: Same-Region)
- `enabled (BOOLEAN)`: 정책 활성화 여부
- `filter_conditions (JSON)`: 복제 필터 조건 (prefix, tags 등)
- `storage_class_override (VARCHAR)`: 복제본 스토리지 클래스 변경
- `replica_kms_key_id (BOOLEAN)`: 복제본 암호화 키 설정
- `created_at (TIMESTAMP)`: 정책 생성 시각
- `last_modified (TIMESTAMP)`: 정책 마지막 수정 시각
- `created_by_user_id (VARCHAR)`: 정책 생성자 ID

**비즈니스 로직**:
- 버킷당 여러 복제 정책 설정 가능
- 필터 조건에 따라 선택적 복제 수행
- CRR은 지역 간 재해 복구용, SRR은 동일 지역 내 가용성 향상용
- 정책 변경 시 기존 객체에 소급 적용 여부 결정
- 복제 실패 시 재시도 로직 포함

**성능 고려사항**:
- `bucket_id` 인덱스 (버킷별 정책 조회)
- `(source_region, destination_region)` 복합 인덱스 (지역별 정책 관리)
- `enabled` 인덱스 (활성 정책 필터링)

**관련 테이블**:
- BUCKET_METADATA (bucket_id 외래키)
- REPLICATION_STATUS (policy_id 외래키)

---

### REPLICATION_STATUS

**역할**: 개별 객체의 복제 상태를 추적하는 테이블

**주요 필드 상세**:
- `replication_id (UUID, PK)`: 복제 작업의 고유 식별자
- `object_id (UUID, FK)`: 복제 대상 객체 ID
- `policy_id (UUID, FK)`: 적용된 복제 정책 ID
- `source_region (VARCHAR)`: 소스 리전
- `destination_region (VARCHAR)`: 복제 대상 리전
- `replication_status (VARCHAR)`: 복제 상태 (PENDING, IN_PROGRESS, COMPLETED, FAILED)
- `last_replicated_at (TIMESTAMP)`: 마지막 복제 시도 시각
- `failure_reason (VARCHAR)`: 복제 실패 사유
- `retry_count (INT)`: 재시도 횟수
- `bytes_replicated (BIGINT)`: 복제된 바이트 수
- `next_retry_at (TIMESTAMP)`: 다음 재시도 예정 시각
- `replica_object_id (VARCHAR)`: 복제본 객체 ID
- `replica_version_id (VARCHAR)`: 복제본 버전 ID

**비즈니스 로직**:
- 객체 생성/업데이트 시 해당 복제 정책에 따라 복제 작업 생성
- 복제 실패 시 지수 백오프로 재시도 (최대 7일)
- 복제 완료 시 COMPLETED 상태로 변경
- 소스 객체 삭제 시 복제본도 삭제 (삭제 마커 복제)
- 네트워크 이슈, 권한 문제 등으로 인한 실패 추적

**성능 고려사항**:
- `(object_id, policy_id)` 복합 인덱스 (객체별 복제 상태 조회)
- `replication_status` 인덱스 (실패한 복제 재시도 관리)
- `next_retry_at` 인덱스 (재시도 스케줄링)
- `last_replicated_at` 인덱스 (복제 지연 모니터링)

**관련 테이블**:
- OBJECT_METADATA (object_id 외래키)
- REPLICATION_POLICIES (policy_id 외래키)

---

## IAM and Security (IAM 및 보안)

### USERS

**역할**: 시스템 사용자의 신원 정보와 계정 관리를 담당하는 테이블

**주요 필드 상세**:
- `user_id (UUID, PK)`: 사용자의 전역 고유 식별자
- `username (VARCHAR, UK)`: 사용자명 (로그인용, 유니크)
- `email (VARCHAR, UK)`: 이메일 주소 (유니크, 인증용)
- `password_hash (VARCHAR)`: 암호화된 패스워드 해시
- `account_type (VARCHAR)`: 계정 유형 (ROOT, IAM_USER, SERVICE_ACCOUNT)
- `mfa_enabled (BOOLEAN)`: 다단계 인증 활성화 여부
- `mfa_device_id (VARCHAR)`: MFA 디바이스 ID
- `created_at (TIMESTAMP)`: 계정 생성 시각
- `last_login (TIMESTAMP)`: 마지막 로그인 시각
- `status (VARCHAR)`: 계정 상태 (ACTIVE, INACTIVE, SUSPENDED)
- `user_attributes (JSON)`: 사용자 속성 (부서, 역할 등)
- `parent_account_id (UUID, FK)`: 상위 계정 ID (IAM 사용자의 경우)
- `storage_quota_bytes (BIGINT)`: 스토리지 할당량
- `current_usage_bytes (BIGINT)`: 현재 스토리지 사용량

**비즈니스 로직**:
- ROOT 계정은 모든 권한을 가지며 IAM 사용자 생성 가능
- IAM 사용자는 ROOT 계정 하위에 생성되며 제한된 권한
- SERVICE_ACCOUNT는 애플리케이션 간 인증용
- MFA 활성화 시 민감한 작업에 추가 인증 요구
- 할당량 초과 시 추가 업로드 차단

**성능 고려사항**:
- `username` 유니크 인덱스 (로그인 조회)
- `email` 유니크 인덱스 (이메일 기반 조회)
- `parent_account_id` 인덱스 (IAM 사용자 목록)
- `status` 인덱스 (활성 사용자 필터링)

**관련 테이블**:
- BUCKET_METADATA (owner_id 외래키)
- OBJECT_METADATA (user_id 외래키)
- USER_SESSIONS (user_id 외래키)
- ACCESS_POLICIES (created_by_user_id 외래키)

---

### ACCESS_POLICIES

**역할**: 세분화된 접근 권한 정책을 정의하고 관리하는 테이블

**주요 필드 상세**:
- `policy_id (UUID, PK)`: 정책의 고유 식별자
- `policy_name (VARCHAR)`: 정책 이름 (사용자 정의)
- `policy_type (VARCHAR)`: 정책 유형 (BUCKET_POLICY, IAM_POLICY, ACL)
- `policy_document (JSON)`: JSON 형태의 정책 문서
- `resource_arn (VARCHAR)`: 리소스 ARN (Amazon Resource Name)
- `principal (VARCHAR)`: 주체 (사용자/역할 ARN)
- `allowed_actions (JSON)`: 허용된 작업 목록 (s3:GetObject, s3:PutObject 등)
- `denied_actions (JSON)`: 거부된 작업 목록
- `conditions (JSON)`: 조건 (IP 제한, 시간 제한 등)
- `created_at (TIMESTAMP)`: 정책 생성 시각
- `last_modified (TIMESTAMP)`: 정책 마지막 수정 시각
- `is_active (BOOLEAN)`: 정책 활성화 여부
- `created_by_user_id (VARCHAR)`: 정책 생성자 ID

**비즈니스 로직**:
- 정책 문서는 AWS IAM 정책 형식과 호환
- ALLOW와 DENY 정책이 충돌 시 DENY 우선 적용
- 조건부 정책으로 시간, IP, MFA 등 제약 설정 가능
- 리소스별, 작업별 세분화된 권한 제어
- 정책 변경 시 즉시 반영 또는 지연 반영 선택 가능

**성능 고려사항**:
- `resource_arn` 인덱스 (리소스별 정책 조회)
- `principal` 인덱스 (사용자별 정책 조회)
- `policy_type` 인덱스 (정책 유형별 필터링)
- `is_active` 인덱스 (활성 정책만 조회)
- 정책 평가 성능을 위해 캐싱 필수

**관련 테이블**:
- USERS (created_by_user_id 외래키)
- BUCKET_METADATA (resource_arn에서 참조)

---

### USER_SESSIONS

**역할**: 사용자 인증 세션과 토큰을 관리하는 테이블

**주요 필드 상세**:
- `session_id (UUID, PK)`: 세션의 고유 식별자
- `user_id (UUID, FK)`: 세션 소유자 사용자 ID
- `access_token_hash (VARCHAR)`: 액세스 토큰 해시
- `refresh_token_hash (VARCHAR)`: 리프레시 토큰 해시
- `issued_at (TIMESTAMP)`: 토큰 발급 시각
- `expires_at (TIMESTAMP)`: 토큰 만료 시각
- `ip_address (VARCHAR)`: 클라이언트 IP 주소
- `user_agent (VARCHAR)`: 클라이언트 User-Agent
- `session_status (VARCHAR)`: 세션 상태 (ACTIVE, EXPIRED, REVOKED)
- `session_metadata (JSON)`: 세션 관련 메타데이터
- `mfa_verified_at (TIMESTAMP)`: MFA 인증 완료 시각

**비즈니스 로직**:
- 액세스 토큰은 짧은 수명 (1시간), 리프레시 토큰은 긴 수명 (30일)
- 토큰 만료 시 리프레시 토큰으로 갱신
- 의심스러운 활동 감지 시 세션 강제 종료
- IP 주소 변경 시 추가 인증 요구 가능
- MFA 인증 후 일정 시간 동안 재인증 면제

**성능 고려사항**:
- `access_token_hash` 인덱스 (토큰 검증)
- `user_id` 인덱스 (사용자별 세션 조회)
- `expires_at` 인덱스 (만료된 세션 정리)
- `session_status` 인덱스 (활성 세션 필터링)

**관련 테이블**:
- USERS (user_id 외래키)

---

## Caching Layer (캐싱 계층)

### CACHE_METADATA

**역할**: 다층 캐시 시스템의 메타데이터를 관리하는 테이블

**주요 필드 상세**:
- `cache_id (UUID, PK)`: 캐시 엔트리의 고유 식별자
- `object_id (UUID, FK)`: 캐시된 객체 ID
- `cache_tier (VARCHAR)`: 캐시 계층 (CDN, EDGE, APPLICATION)
- `cache_location (VARCHAR)`: 캐시 지리적 위치
- `cache_node_id (VARCHAR)`: 캐시 노드 식별자
- `cached_at (TIMESTAMP)`: 캐시 저장 시각
- `expires_at (TIMESTAMP)`: 캐시 만료 시각
- `cache_size_bytes (BIGINT)`: 캐시된 데이터 크기
- `cache_status (VARCHAR)`: 캐시 상태 (CACHED, EVICTED, EXPIRED)
- `hit_count (INT)`: 캐시 히트 횟수
- `last_accessed (TIMESTAMP)`: 마지막 접근 시각
- `cache_key (VARCHAR)`: 캐시 키 (객체 키에서 파생)
- `content_hash (VARCHAR)`: 캐시된 콘텐츠 해시

**비즈니스 로직**:
- CDN 캐시는 가장 오래 유지, EDGE 캐시는 중간, APPLICATION 캐시는 짧게 유지
- 캐시 히트율 기반으로 인기 콘텐츠 식별
- 원본 객체 변경 시 관련 캐시 무효화
- LRU 알고리즘으로 캐시 교체
- 지역별 캐시로 지연 시간 최소화

**성능 고려사항**:
- `cache_key` 인덱스 (캐시 조회)
- `object_id` 인덱스 (객체별 캐시 무효화)
- `expires_at` 인덱스 (만료된 캐시 정리)
- `(cache_tier, cache_location)` 복합 인덱스 (계층별 관리)

**관련 테이블**:
- OBJECT_METADATA (object_id 외래키)

---

## Monitoring and Analytics (모니터링 및 분석)

### ACCESS_LOGS

**역할**: 모든 API 요청과 응답을 기록하는 감사 로그 테이블

**주요 필드 상세**:
- `log_id (UUID, PK)`: 로그 엔트리의 고유 식별자
- `user_id (UUID, FK)`: 요청 사용자 ID
- `bucket_id (UUID, FK)`: 대상 버킷 ID
- `object_id (UUID, FK)`: 대상 객체 ID
- `operation_type (VARCHAR)`: 작업 유형 (GET, PUT, DELETE, LIST)
- `request_method (VARCHAR)`: HTTP 메서드
- `resource_path (VARCHAR)`: 요청 리소스 경로
- `source_ip (VARCHAR)`: 클라이언트 IP 주소
- `user_agent (VARCHAR)`: 클라이언트 User-Agent
- `request_time (TIMESTAMP)`: 요청 시각
- `response_time (TIMESTAMP)`: 응답 시각
- `http_status_code (INT)`: HTTP 응답 코드
- `bytes_sent (BIGINT)`: 전송된 바이트 수
- `bytes_received (BIGINT)`: 수신된 바이트 수
- `request_id (VARCHAR, UK)`: 요청 추적용 고유 ID
- `request_headers (JSON)`: 요청 헤더 정보
- `response_headers (JSON)`: 응답 헤더 정보
- `error_code (VARCHAR)`: 오류 코드 (해당하는 경우)
- `error_message (VARCHAR)`: 오류 메시지
- `response_time_ms (DOUBLE)`: 응답 시간 (밀리초)

**비즈니스 로직**:
- 모든 API 호출을 빠짐없이 기록
- 보안 감사 및 규정 준수용 데이터
- 성능 분석 및 최적화 데이터 제공
- 비정상적인 접근 패턴 탐지
- 사용량 기반 과금 데이터 제공

**성능 고려사항**:
- `request_time` 인덱스 (시간 기반 조회)
- `user_id` 인덱스 (사용자별 활동 조회)
- `source_ip` 인덱스 (IP 기반 분석)
- `http_status_code` 인덱스 (오류 분석)
- 대용량 로그이므로 파티셔닝 필수 (일/월 단위)

**관련 테이블**:
- USERS (user_id 외래키)
- BUCKET_METADATA (bucket_id 외래키)
- OBJECT_METADATA (object_id 외래키)

---

### BILLING_METRICS

**역할**: 사용량 기반 과금을 위한 메트릭을 수집하는 테이블

**주요 필드 상세**:
- `metric_id (UUID, PK)`: 메트릭의 고유 식별자
- `user_id (UUID, FK)`: 과금 대상 사용자 ID
- `bucket_id (UUID, FK)`: 과금 대상 버킷 ID
- `metric_type (VARCHAR)`: 메트릭 유형 (STORAGE, REQUESTS, BANDWIDTH)
- `storage_gb_hours (BIGINT)`: 스토리지 사용량 (GB-시간)
- `request_count (BIGINT)`: 요청 횟수
- `bandwidth_gb (BIGINT)`: 대역폭 사용량 (GB)
- `storage_class (VARCHAR)`: 스토리지 클래스별 구분
- `metric_date (TIMESTAMP)`: 메트릭 수집 날짜
- `cost_usd (DECIMAL)`: 계산된 비용 (USD)
- `billing_period (VARCHAR)`: 과금 주기 (HOURLY, DAILY, MONTHLY)
- `detailed_breakdown (JSON)`: 세부 사용량 분석

**비즈니스 로직**:
- 시간당/일당 사용량 집계
- 스토리지 클래스별 차등 과금
- 요청 유형별 과금 (GET, PUT, DELETE 등)
- 대역폭 사용량 기반 과금
- 월말 결산 및 청구서 생성

**성능 고려사항**:
- `(user_id, metric_date)` 복합 인덱스 (사용자별 과금 조회)
- `metric_type` 인덱스 (메트릭 유형별 집계)
- `billing_period` 인덱스 (주기별 집계)
- 시계열 데이터이므로 시간 기반 파티셔닝

**관련 테이블**:
- USERS (user_id 외래키)
- BUCKET_METADATA (bucket_id 외래키)

---

### PERFORMANCE_METRICS

**역할**: 시스템 성능 지표를 수집하고 모니터링하는 테이블

**주요 필드 상세**:
- `metric_id (UUID, PK)`: 메트릭의 고유 식별자
- `node_id (VARCHAR, FK)`: 메트릭 수집 노드 ID
- `metric_type (VARCHAR)`: 메트릭 유형 (LATENCY, THROUGHPUT, ERROR_RATE)
- `metric_value (DOUBLE)`: 메트릭 값
- `metric_unit (VARCHAR)`: 메트릭 단위 (ms, ops/sec, % 등)
- `recorded_at (TIMESTAMP)`: 메트릭 수집 시각
- `operation_type (VARCHAR)`: 작업 유형 (READ, write, delete 등)
- `additional_metadata (JSON)`: 추가 메트릭 정보
- `aggregation_period (VARCHAR)`: 집계 주기 (MINUTE, HOUR, DAY)

**비즈니스 로직**:
- 실시간 성능 모니터링
- SLA 준수 여부 추적
- 성능 병목 지점 식별
- 용량 계획 및 확장 결정 지원
- 알람 및 알림 트리거

**성능 고려사항**:
- `(node_id, recorded_at)` 복합 인덱스 (노드별 시계열 조회)
- `metric_type` 인덱스 (메트릭 유형별 필터링)
- `aggregation_period` 인덱스 (집계 주기별 조회)
- 고빈도 데이터이므로 시계열 DB 사용 고려

**관련 테이블**:
- STORAGE_NODES (node_id 외래키)