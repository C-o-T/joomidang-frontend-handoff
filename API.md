# 주미당 API 문서

모든 API 응답은 아래 공통 형식을 따른다.

```typescript
// 성공
{ success: true, data: T, message?: string }

// 실패
{ success: false, error: string }
```

인증이 필요한 API에 미인증 요청 시 → `401 Unauthorized`  
권한이 부족한 경우 → `403 Forbidden`

---

## 인증 API

### POST /api/auth/register

**인증**: 불필요  
**설명**: 이메일/비밀번호 회원가입

**Request Body**
```json
{
  "email": "string (필수, 유효한 이메일 형식)",
  "password": "string (필수, 8자 이상)",
  "name": "string (선택)"
}
```

**Response** `201`
```json
{
  "success": true,
  "data": {
    "id": "string",
    "email": "string",
    "name": "string | null",
    "role": "CONSUMER"
  },
  "message": "회원가입이 완료됐습니다"
}
```

**에러**
- `400` 이메일/비밀번호 누락, 이메일 형식 오류, 비밀번호 8자 미만
- `409` 이미 사용 중인 이메일

---

### POST /api/auth/forgot-password

**인증**: 불필요  
**설명**: 비밀번호 재설정 이메일 발송 (사용자 존재 여부 무관하게 동일 응답 — 보안)

**Request Body**
```json
{
  "email": "string (필수)"
}
```

**Response** `200`
```json
{
  "success": true,
  "data": { "message": "이메일을 확인하세요" }
}
```

---

### POST /api/auth/reset-password

**인증**: 불필요  
**설명**: 토큰으로 비밀번호 변경. 토큰 유효시간 1시간.

**Request Body**
```json
{
  "email": "string (필수)",
  "token": "string (필수, 이메일로 수신한 토큰)",
  "password": "string (필수, 8자 이상)"
}
```

**Response** `200`
```json
{
  "success": true,
  "data": { "message": "비밀번호가 변경되었습니다" }
}
```

**에러**
- `400` 파라미터 누락, 비밀번호 8자 미만, 유효하지 않은 토큰, 만료된 토큰

---

## 내 정보 API

### GET /api/me

**인증**: 필요  
**설명**: 현재 로그인한 사용자 정보 조회

**Response** `200`
```json
{
  "success": true,
  "data": {
    "id": "string",
    "email": "string",
    "name": "string | null",
    "image": "string | null",
    "role": "CONSUMER | SELLER | ADMIN",
    "createdAt": "ISO8601 string"
  }
}
```

---

### PATCH /api/me

**인증**: 필요  
**설명**: 이름 변경 또는 비밀번호 변경 (소셜 로그인 계정은 비밀번호 변경 불가)

**Request Body** (변경할 항목만 포함)
```json
{
  "name": "string (선택)",
  "currentPassword": "string (비밀번호 변경 시 필수)",
  "newPassword": "string (선택, 8자 이상)"
}
```

**Response** `200`
```json
{
  "success": true,
  "data": {
    "id": "string",
    "email": "string",
    "name": "string | null",
    "role": "CONSUMER | SELLER | ADMIN"
  }
}
```

**에러**
- `400` 이름 공백, 현재 비밀번호 누락, 현재 비밀번호 불일치, 새 비밀번호 8자 미만, 변경 내용 없음
- `400` 소셜 로그인 계정에 비밀번호 변경 시도 시

---

## 상품 API

### GET /api/products

**인증**: 불필요  
**설명**: 상품 목록 조회 (필터/정렬/페이지네이션 지원)

**Query Parameters**
| 파라미터 | 타입 | 기본값 | 설명 |
|---|---|---|---|
| `page` | number | 1 | 페이지 번호 |
| `limit` | number | 20 | 페이지당 항목 수 (최대 50) |
| `category` | string | - | `MAKGEOLLI`, `SOJU`, `CHEONGJU`, `YAKJU`, `FRUIT_WINE`, `OTHER` |
| `search` | string | - | 상품명 검색 (대소문자 무시) |
| `sort` | string | `newest` | `newest`, `price_asc`, `price_desc`, `popular` |
| `minAbv` | number | - | 최소 도수 (%) |
| `maxAbv` | number | - | 최대 도수 (%) |
| `vol` | number | - | 용량 필터 (ml) |
| `isNew` | string | - | `true` — 신상품만 |
| `noStock` | string | - | `true` — 품절 포함 |

**Response** `200`
```json
{
  "success": true,
  "data": {
    "products": [
      {
        "id": "string",
        "nameKo": "string",
        "nameEn": "string",
        "nameJa": "string | null",
        "nameZh": "string | null",
        "category": "MAKGEOLLI | SOJU | ...",
        "priceKrw": "number",
        "stock": "number",
        "thumbnailUrl": "string | null",
        "images": "string[]",
        "alcoholContent": "number | null",
        "volumeMl": "number | null",
        "breweryKo": "string | null",
        "breweryEn": "string | null",
        "regionKo": "string | null",
        "regionEn": "string | null",
        "status": "ACTIVE",
        "avgRating": "number | null",
        "reviewCount": "number"
      }
    ],
    "total": "number",
    "page": "number",
    "limit": "number"
  }
}
```

---

### GET /api/products/[id]

**인증**: 불필요  
**설명**: 상품 상세 조회 (최근 리뷰 20개 포함). `ACTIVE` 상태 상품만 반환.

**Response** `200`
```json
{
  "success": true,
  "data": {
    "id": "string",
    "nameKo": "string",
    "nameEn": "string",
    "descKo": "string | null",
    "descEn": "string | null",
    "category": "string",
    "priceKrw": "number",
    "stock": "number",
    "thumbnailUrl": "string | null",
    "images": "string[]",
    "alcoholContent": "number | null",
    "volumeMl": "number | null",
    "breweryKo": "string | null",
    "breweryEn": "string | null",
    "regionKo": "string | null",
    "regionEn": "string | null",
    "avgRating": "number | null",
    "reviewCount": "number",
    "reviews": [
      {
        "id": "string",
        "userId": "string",
        "userName": "string | null",
        "productId": "string",
        "rating": "number (1-5)",
        "content": "string | null",
        "createdAt": "ISO8601 string"
      }
    ],
    "createdAt": "ISO8601 string",
    "updatedAt": "ISO8601 string"
  }
}
```

**에러**
- `404` 상품 없음 또는 비활성 상품

---

### GET /api/products/[id]/reviews

**인증**: 불필요  
**설명**: 상품 리뷰 목록 (페이지네이션)

**Query Parameters**
| 파라미터 | 기본값 | 설명 |
|---|---|---|
| `page` | 1 | 페이지 번호 |
| `limit` | 10 | 페이지당 항목 수 (최대 20) |

**Response** `200`
```json
{
  "success": true,
  "data": {
    "reviews": [
      {
        "id": "string",
        "userId": "string",
        "userName": "string | null",
        "productId": "string",
        "rating": "number",
        "content": "string | null",
        "createdAt": "ISO8601 string"
      }
    ],
    "total": "number",
    "page": "number",
    "totalPages": "number"
  }
}
```

---

### POST /api/products/[id]/reviews

**인증**: 필요 (구매한 상품만 작성 가능 — `DELIVERED` 또는 `SHIPPED` 상태 주문)  
**설명**: 리뷰 작성 (1인 1리뷰 제한)

**Request Body**
```json
{
  "rating": "number (필수, 1-5 정수)",
  "content": "string (선택)"
}
```

**Response** `201`
```json
{
  "success": true,
  "data": {
    "id": "string",
    "userId": "string",
    "userName": "string | null",
    "productId": "string",
    "rating": "number",
    "content": "string | null",
    "createdAt": "ISO8601 string"
  }
}
```

**에러**
- `400` 별점 범위 오류
- `403` 구매 이력 없음
- `409` 이미 리뷰 작성함

---

### DELETE /api/products/[id]/reviews

**인증**: 필요 (본인 또는 ADMIN)  
**설명**: 리뷰 삭제

**Request Body**
```json
{
  "reviewId": "string (필수)"
}
```

**Response** `200`
```json
{ "success": true, "data": null }
```

**에러**
- `400` reviewId 누락
- `403` 본인 리뷰가 아닌 경우
- `404` 리뷰 없음

---

## 장바구니 API

### GET /api/cart

**인증**: 필요  
**설명**: 장바구니 조회 (없으면 자동 생성)

**Response** `200`
```json
{
  "success": true,
  "data": {
    "id": "string",
    "items": [
      {
        "id": "string",
        "productId": "string",
        "quantity": "number",
        "product": {
          "nameKo": "string",
          "nameEn": "string",
          "priceKrw": "number",
          "thumbnailUrl": "string | null",
          "stock": "number"
        }
      }
    ],
    "totalKrw": "number"
  }
}
```

---

### POST /api/cart

**인증**: 필요  
**설명**: 장바구니에 상품 추가 (이미 있으면 수량 합산)

**Request Body**
```json
{
  "productId": "string (필수)",
  "quantity": "number (선택, 기본값 1, 1-100)"
}
```

**Response** `201`
```json
{ "success": true, "data": null, "message": "장바구니에 추가됐습니다" }
```

**에러**
- `400` productId 누락, 수량 범위 오류
- `404` 상품 없음 또는 비활성
- `409` 재고 초과

---

### DELETE /api/cart

**인증**: 필요  
**설명**: 장바구니 아이템 삭제

**Request Body**
```json
{
  "itemId": "string (필수, CartItem의 id)"
}
```

**Response** `200`
```json
{ "success": true, "data": null, "message": "삭제됐습니다" }
```

---

## 주문 API

### GET /api/orders

**인증**: 필요  
**설명**: 내 주문 목록 (최신순)

**Response** `200`
```json
{
  "success": true,
  "data": [
    {
      "id": "string",
      "status": "PENDING | PAID | PREPARING | SHIPPED | DELIVERED | CANCELLED | REFUNDED",
      "totalKrw": "number",
      "currency": "string",
      "totalConverted": "number | null",
      "shippingAddress": {
        "name": "string",
        "phone": "string",
        "country": "string",
        "postalCode": "string",
        "address1": "string",
        "address2": "string (선택)",
        "city": "string"
      },
      "createdAt": "ISO8601 string",
      "items": [
        {
          "id": "string",
          "productId": "string",
          "quantity": "number",
          "priceKrw": "number",
          "product": {
            "nameKo": "string",
            "nameEn": "string",
            "thumbnailUrl": "string | null"
          }
        }
      ]
    }
  ]
}
```

---

### POST /api/orders

**인증**: 필요  
**설명**: 주문 생성 (장바구니 기반). 재고 감소 및 장바구니 초기화 처리. 주문 확인 이메일 자동 발송.

**Request Body**
```json
{
  "shippingAddress": {
    "name": "string (필수)",
    "phone": "string",
    "country": "string",
    "postalCode": "string",
    "address1": "string (필수)",
    "address2": "string (선택)",
    "city": "string"
  },
  "currency": "string (선택, 기본값 'KRW')"
}
```

**Response** `201` — 주문 객체 (GET /api/orders의 단일 항목과 동일한 형식)

**에러**
- `400` 배송지 정보 누락, 장바구니 비어있음
- `409` 품절 상품 포함 또는 재고 부족

---

### POST /api/orders/[id]/cancel

**인증**: 필요 (본인 주문만)  
**설명**: 주문 취소. `PENDING` 상태는 `CANCELLED`, `PAID` 상태는 Stripe 환불 후 `REFUNDED`. `PREPARING` 이상은 취소 불가.

**Request Body**: 없음

**Response** `200`
```json
{
  "success": true,
  "data": {
    "id": "string",
    "status": "CANCELLED | REFUNDED"
  }
}
```

**에러**
- `400` 취소 불가 상태
- `404` 주문 없음
- `502` Stripe 환불 실패

---

## 위시리스트 API

### GET /api/wishlist

**인증**: 필요  
**설명**: 위시리스트 목록 (최신순)

**Response** `200`
```json
{
  "success": true,
  "data": [
    {
      "id": "string",
      "userId": "string",
      "productId": "string",
      "createdAt": "ISO8601 string",
      "product": {
        "id": "string",
        "nameKo": "string",
        "nameEn": "string",
        "thumbnailUrl": "string | null",
        "priceKrw": "number",
        "stock": "number",
        "status": "string"
      }
    }
  ]
}
```

---

### POST /api/wishlist

**인증**: 필요  
**설명**: 위시리스트에 상품 추가 (이미 있으면 무시)

**Request Body**
```json
{
  "productId": "string (필수)"
}
```

**Response** `201`
```json
{ "success": true, "data": { "id": "string", "userId": "string", "productId": "string", "createdAt": "ISO8601 string" } }
```

**에러**
- `400` productId 누락
- `404` 존재하지 않는 상품

---

### DELETE /api/wishlist

**인증**: 필요  
**설명**: 위시리스트에서 제거

**Request Body**
```json
{
  "productId": "string (필수)"
}
```

**Response** `200`
```json
{ "success": true, "data": null }
```

---

## 이미지 업로드 API

### POST /api/upload

**인증**: SELLER 또는 ADMIN 전용  
**설명**: 상품 이미지를 AWS S3에 업로드. `multipart/form-data`로 전송.

**Request** (FormData)
```
file: File (jpg, jpeg, png, webp, avif / 최대 10MB)
```

**Response** `200`
```json
{
  "success": true,
  "data": {
    "url": "https://버킷명.s3.ap-northeast-2.amazonaws.com/products/타임스탬프-파일명.jpg"
  }
}
```

**에러**
- `400` 파일 없음, 허용되지 않는 확장자, 10MB 초과
- `403` SELLER/ADMIN이 아닌 경우
- `503` S3 설정 미완료

---

## 셀러 API

### POST /api/seller/register

**인증**: 필요 (CONSUMER 역할만)  
**설명**: 셀러 등록 신청. 초기 상태는 `PENDING` (관리자 승인 필요).

**Request Body**
```json
{
  "businessName": "string (필수)",
  "businessNo": "string (선택, 사업자등록번호)"
}
```

**Response** `201`
```json
{
  "success": true,
  "data": {
    "id": "string",
    "userId": "string",
    "businessName": "string",
    "businessNo": "string | null",
    "status": "PENDING",
    "createdAt": "ISO8601 string"
  }
}
```

**에러**
- `400` 사업자명 누락
- `403` 이미 셀러이거나 권한 없음
- `409` 이미 셀러 등록됨

---

### GET /api/seller/products

**인증**: SELLER 전용  
**설명**: 내 상품 목록

**Query Parameters**
| 파라미터 | 설명 |
|---|---|
| `status` | `PENDING`, `ACTIVE`, `INACTIVE` 필터 (생략 시 전체) |

**Response** `200` — `ProductDTO[]`

---

### POST /api/seller/products

**인증**: SELLER 전용 (APPROVED 상태만 등록 가능)  
**설명**: 상품 등록. 초기 상태는 `PENDING` (관리자 승인 필요).

**Request Body**
```json
{
  "nameKo": "string (필수)",
  "nameEn": "string (필수)",
  "nameJa": "string (선택)",
  "nameZh": "string (선택)",
  "descKo": "string (선택)",
  "descEn": "string (선택)",
  "category": "MAKGEOLLI | SOJU | CHEONGJU | YAKJU | FRUIT_WINE | OTHER (필수)",
  "priceKrw": "number (필수, 0 초과)",
  "stock": "number (필수, 0 이상)",
  "thumbnailUrl": "string (선택)",
  "images": "string[] (선택)",
  "alcoholContent": "number (선택, 도수 %)",
  "volumeMl": "number (선택, 용량 ml)",
  "breweryKo": "string (선택)",
  "breweryEn": "string (선택)",
  "regionKo": "string (선택)",
  "regionEn": "string (선택)"
}
```

**Response** `201` — `ProductDTO`

---

### GET /api/seller/products/[id]

**인증**: SELLER 전용 (본인 상품만)  
**Response** `200` — `ProductDTO`

---

### PUT /api/seller/products/[id]

**인증**: SELLER 전용 (본인 상품만)  
**설명**: 상품 수정 (변경할 필드만 포함)

**Request Body** — POST와 동일한 필드, 모두 선택

**Response** `200` — `ProductDTO`

---

### DELETE /api/seller/products/[id]

**인증**: SELLER 전용 (본인 상품만)  
**설명**: 상품 삭제 (연결된 장바구니 아이템도 함께 삭제)

**Response** `200`
```json
{ "success": true, "data": null, "message": "삭제됐습니다" }
```

---

### GET /api/seller/orders

**인증**: SELLER 전용 (APPROVED 상태만)  
**설명**: 내 상품이 포함된 주문 목록 (PENDING 상태 제외)

**Response** `200`
```json
{
  "success": true,
  "data": [
    {
      "id": "string",
      "status": "string",
      "totalKrw": "number",
      "createdAt": "ISO8601 string",
      "user": { "name": "string | null", "email": "string | null" },
      "items": [
        {
          "id": "string",
          "productId": "string",
          "quantity": "number",
          "priceKrw": "number",
          "product": { "id": "string", "nameKo": "string", "nameEn": "string", "thumbnailUrl": "string | null" }
        }
      ]
    }
  ]
}
```

---

### PATCH /api/seller/orders/[id]

**인증**: SELLER 전용 (APPROVED, 본인 상품 포함 주문만)  
**설명**: 주문 상태 변경 또는 운송장 번호 등록. 배송 시작(`SHIPPED`) 시 이메일 자동 발송.

**허용된 상태 전이**
| 현재 상태 | 변경 가능 상태 |
|---|---|
| PAID | PREPARING, CANCELLED |
| PREPARING | SHIPPED, CANCELLED |
| SHIPPED | DELIVERED |

**Request Body**
```json
{
  "status": "string (선택)",
  "trackingNumber": "string (선택)"
}
```

**Response** `200` — 업데이트된 주문 객체

---

### GET /api/seller/stats

**인증**: SELLER 전용 (APPROVED 상태만)  
**설명**: 매출 통계 (최근 200건 주문 기준, 수수료율 10%)

**Response** `200`
```json
{
  "success": true,
  "data": {
    "commissionRate": 0.1,
    "totalSalesKrw": "number",
    "commissionKrw": "number",
    "payoutKrw": "number",
    "deliveredSalesKrw": "number",
    "deliveredPayout": "number",
    "items": [
      {
        "orderId": "string",
        "orderStatus": "string",
        "orderDate": "ISO8601 string",
        "productName": "string",
        "quantity": "number",
        "priceKrw": "number",
        "subtotalKrw": "number"
      }
    ]
  }
}
```

---

## 어드민 API

### GET /api/admin/products

**인증**: ADMIN 전용  
**설명**: 전체 상품 목록

**Query Parameters**
| 파라미터 | 설명 |
|---|---|
| `status` | `PENDING`, `ACTIVE`, `INACTIVE` 필터 (생략 시 전체) |

**Response** `200` — `(ProductDTO & { seller: { id: string, businessName: string } })[]`

---

### PATCH /api/admin/products

**인증**: ADMIN 전용  
**설명**: 상품 상태 변경 (승인/비활성화)

**Request Body**
```json
{
  "productId": "string (필수)",
  "status": "ACTIVE | INACTIVE | PENDING (필수)"
}
```

**Response** `200` — 업데이트된 `ProductDTO`

---

### GET /api/admin/sellers

**인증**: ADMIN 전용  
**설명**: 전체 셀러 목록

**Query Parameters**
| 파라미터 | 설명 |
|---|---|
| `status` | `PENDING`, `APPROVED`, `SUSPENDED` 필터 |

**Response** `200`
```json
{
  "success": true,
  "data": [
    {
      "id": "string",
      "businessName": "string",
      "businessNo": "string | null",
      "status": "PENDING | APPROVED | SUSPENDED",
      "createdAt": "ISO8601 string",
      "user": {
        "id": "string",
        "email": "string",
        "name": "string | null",
        "createdAt": "ISO8601 string"
      }
    }
  ]
}
```

---

### PATCH /api/admin/sellers/[id]

**인증**: ADMIN 전용  
**설명**: 셀러 상태 변경. `APPROVED` 시 해당 User의 role을 `SELLER`로 변경 + 승인 이메일 발송. `SUSPENDED` 시 role을 `CONSUMER`로 강등.

**Request Body**
```json
{
  "status": "APPROVED | SUSPENDED (필수)"
}
```

**Response** `200` — 업데이트된 셀러 객체

**에러**
- `400` 유효하지 않은 상태 (PENDING으로 변경 불가)
- `404` 셀러 없음

---

### POST /api/admin/orders/[id]/refund

**인증**: ADMIN 전용  
**설명**: 주문 강제 환불 (Stripe 환불 + 상태 `REFUNDED`). 이미 취소/환불된 주문은 불가.

**Request Body**
```json
{
  "reason": "string (선택, 메모용)"
}
```

**Response** `200`
```json
{
  "success": true,
  "data": {
    "id": "string",
    "status": "REFUNDED"
  }
}
```

**에러**
- `400` 이미 취소/환불된 주문
- `404` 주문 없음
- `502` Stripe 환불 실패
