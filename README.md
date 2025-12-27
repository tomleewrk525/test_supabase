# test_supabase

This repository was initially set up to test Supabase.

## Recent Changes

- **Resolved GitHub Push Protection Error:**
  A GitHub Push Protection error occurred during a recent push due to a Personal Access Token being inadvertently committed. To resolve this, the `.gemini/settings.json` file, which contained the sensitive token, has been added to `.gitignore`. This ensures that sensitive information is not committed to the repository, and the commit history has been re-written to reflect this change.

- **`.gitignore` added:**
  The `.gitignore` file has been created and configured to ignore `.gemini/settings.json`, preventing future accidental commits of sensitive configuration data.

## Supabase Project Guide

### Supabase 프로젝트 설명 및 가이드

이 프로젝트는 부동산 매물 정보 관리 및 거래를 위한 Supabase 기반의 데이터베이스 스키마를 구축했습니다. 사용자 인증, 역할 분리, 보안(RLS), 성능 최적화 및 개발 편의성을 고려하여 설계되었습니다.

---

#### **1. 데이터베이스 스키마 개요**

프로젝트는 총 7개의 테이블과 3개의 사용자 정의 ENUM 타입을 포함합니다.

*   **`users` (public.users)**
    *   **목적:** Supabase 인증(Auth) 시스템과 연동하여 사용자의 기본 정보를 관리합니다.
    *   **주요 컬럼:** `id` (Auth.users의 id 참조), `created_at`, `updated_at`.
    *   **관계:** `customers` 및 `providers` 테이블의 부모 테이블 역할을 합니다 (1:1 관계).
    *   **RLS:** 각 사용자는 자신의 `users` 레코드만 조회/삽입/수정 가능.
    *   **특징:** `email`과 `password`는 `auth.users` 테이블에서 관리되며, 새 사용자 가입 시 트리거를 통해 자동으로 `public.users`에 레코드가 추가됩니다.

*   **`customers` (public.customers)**
    *   **목적:** 일반 고객(부동산을 소비하는 사용자)의 추가 프로필 정보를 저장합니다.
    *   **주요 컬럼:** `id` (users.id 참조), `customer_type`, `shipping_address`, `phone_number`.
    *   **관계:** `users`와 1:1 관계. `transactions` 테이블과 1:N 관계.
    *   **RLS:** 각 고객은 자신의 `customers` 레코드만 조회/삽입/수정 가능.

*   **`providers` (public.providers)**
    *   **목적:** 부동산을 제공하는 주체(예: 부동산 중개업소, 건설사)의 정보를 저장합니다.
    *   **주요 컬럼:** `id` (users.id 참조), `company_name`, `business_type`, `contact_person`, `contact_email`, `website`.
    *   **관계:** `users`와 1:1 관계. `properties` 테이블과 1:N 관계.
    *   **RLS:** 각 제공자는 자신의 `providers` 레코드만 조회/삽입/수정 가능.

*   **`properties` (public.properties)**
    *   **목적:** 판매/임대 가능한 부동산 매물 정보를 저장합니다.
    *   **주요 컬럼:** `id`, `provider_id` (providers.id 참조), `address`, `city`, `property_type`, `price`, `description`, `square_footage`, `number_of_bedrooms`, `number_of_bathrooms`, `is_available`.
    *   **ENUM:** `property_type_enum` (`House`, `Condo`, `Apartment`).
    *   **관계:** `providers`와 1:N 관계. `rooms`, `reviews`, `transactions` 테이블과 관계.
    *   **RLS:** 모든 사용자가 조회 가능. 제공자는 자신의 `properties`만 삽입/수정/삭제 가능.

*   **`rooms` (public.rooms)**
    *   **목적:** `properties` 내의 개별 방에 대한 상세 정보를 저장합니다.
    *   **주요 컬럼:** `id`, `property_id` (properties.id 참조), `room_name`, `room_type`, `area_sqm`, `features`.
    *   **ENUM:** `room_type_enum` (`Studio`, `1Room`).
    *   **관계:** `properties`와 1:N 관계. `reviews`, `transactions` 테이블과 관계.
    *   **RLS:** 모든 사용자가 조회 가능. 제공자는 자신의 `properties`에 속한 `rooms`만 삽입/수정/삭제 가능.

*   **`transactions` (public.transactions)**
    *   **목적:** 고객(`customers`)과 부동산 매물(`properties` 또는 `rooms`) 간의 거래 기록을 저장합니다.
    *   **주요 컬럼:** `id`, `customer_id` (customers.id 참조), `property_id` (properties.id 참조), `room_id` (rooms.id 참조), `transaction_type`, `transaction_date`, `amount`, `status`, `lease_start_date`, `lease_end_date`.
    *   **ENUM:** `transaction_type_enum` (`Purchase`, `Lease`).
    *   **특징:** `property_id` 또는 `room_id` 중 하나만 값을 가질 수 있도록 `CHECK` 제약 조건(`check_one_contract_target`)이 설정되어 있습니다. `transaction_type`이 `Lease`일 경우 `lease_start_date`와 `lease_end_date`가 필수로 요구됩니다.
    *   **RLS:** 고객은 자신의 `transactions`만 조회/삽입/수정/삭제 가능. 제공자는 자신의 `properties` 또는 `rooms`와 관련된 `transactions`를 조회 가능.

*   **`reviews` (public.reviews)**
    *   **목적:** 고객이 부동산 매물(`properties` 또는 `rooms`)에 대한 평가를 남기는 정보를 저장합니다.
    *   **주요 컬럼:** `id`, `customer_id` (customers.id 참조), `property_id` (properties.id 참조), `room_id` (rooms.id 참조, `ON DELETE SET NULL`), `rating` (1-5점), `comment`.
    *   **특징:** `property_id` 또는 `room_id` 중 하나만 값을 가질 수 있도록 `CHECK` 제약 조건(`check_one_review_target`)이 설정되어 있습니다.
    *   **RLS:** 모든 사용자가 조회 가능. 고객은 자신의 `reviews`만 삽입/수정/삭제 가능.

---

#### **2. Supabase Auth 연동**

*   `public.users` 테이블은 `auth.users` 테이블과 1:1로 연결되어 있습니다.
*   `handle_new_user()` 함수와 `on_auth_user_created` 트리거를 통해 새로운 사용자가 Supabase Auth를 통해 가입하면, 자동으로 `public.users` 테이블에 해당 사용자의 `id`가 추가됩니다.
*   `public.users` 테이블에서 `email`과 `password` 컬럼은 제거되었으며, 이 정보는 `auth.users` 테이블에서 관리됩니다.

---

#### **3. RLS (Row Level Security) 정책**

모든 `public` 스키마 테이블에 대해 RLS가 활성화되어 있으며, 다음과 같은 접근 제어 정책이 적용되었습니다:

*   **개인 데이터 (users, customers, providers):** 각 사용자는 자신의 프로필 정보만 생성, 조회, 업데이트할 수 있습니다.
*   **공개 데이터 (properties, rooms, reviews):**
    *   **조회:** 모든 사용자가 모든 매물, 방 정보, 리뷰를 조회할 수 있습니다.
    *   **관리:** `providers`는 자신이 등록한 매물 및 해당 매물에 속한 방 정보만 생성, 업데이트, 삭제할 수 있습니다. `customers`는 자신이 작성한 리뷰만 생성, 업데이트, 삭제할 수 있습니다.
*   **거래 데이터 (transactions):**
    *   `customers`는 자신이 참여한 거래 내역만 생성, 조회, 업데이트, 삭제할 수 있습니다.
    *   `providers`는 자신이 제공한 매물 또는 방과 관련된 거래 내역을 조회할 수 있습니다.

---

#### **4. 성능 최적화 (인덱스)**

주요 테이블의 외래키 컬럼 및 자주 검색되는 컬럼에 인덱스를 추가하여 데이터 조회 성능을 향상시켰습니다.

*   `properties`: `provider_id`, `city`, `property_type`
*   `rooms`: `property_id`
*   `transactions`: `customer_id`, `property_id`, `room_id`, `transaction_date`, `transaction_type`, `status`

---

#### **5. PostgreSQL 함수 (RPC)**

*   `public.get_provider_average_rating(p_provider_id UUID)` 함수를 생성하여, 특정 `provider`의 평균 평점을 계산할 수 있습니다. 이 함수는 `SECURITY DEFINER`로 설정되어 RLS 정책에 상관없이 필요한 데이터에 접근하여 계산을 수행합니다.

---

#### **6. TypeScript 타입 생성**

*   `database.types.ts` 파일이 생성되어 현재 데이터베이스 스키마에 대한 TypeScript 타입 정의를 포함합니다. 이를 통해 Supabase 클라이언트와 연동하는 애플리케이션에서 강력한 타입 안정성을 확보하고 개발 생산성을 높일 수 있습니다.

---

#### **다음 단계 제안**

이 가이드를 바탕으로 다음과 같은 추가 개발을 고려할 수 있습니다:

*   **프론트엔드/백엔드 애플리케이션 개발:** Next.js, React, Angular, Vue 등의 프레임워크를 사용하여 이 Supabase 백엔드와 연동되는 애플리케이션을 개발할 수 있습니다.
*   **Edge Functions 활용:** 새로운 사용자 환영 이메일, 매물 상태 변경 알림 등과 같은 서버리스 로직을 Edge Functions로 구현할 수 있습니다.
*   **다른 PostgreSQL 함수/트리거:** 비즈니스 로직에 필요한 추가적인 데이터베이스 함수나 트리거를 구현할 수 있습니다. (예: 매물 예약 기능, 메시지 시스템 등)
*   **Storage (파일 저장):** 부동산 이미지, 도면 등 파일을 Supabase Storage에 저장하고 RLS로 접근 제어할 수 있습니다.

---