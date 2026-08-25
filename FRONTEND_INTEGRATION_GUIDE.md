# 프론트엔드 연동 회의 자료 (백엔드 → 프론트엔드)

> 작성 기준: 2026-08-25, joomidang-v2 저장소 실제 코드 확인 결과.
> 이미 저장소에 `docs/API.md`(엔드포인트 상세 스펙), `docs/FRONTEND_GUIDE.md`(테마/상태관리 상세 가이드)가 존재한다.
> 이 문서는 오늘 연동 회의용으로 **핵심 요약 + 두 문서에서 코드와 어긋난 부분 정정 + 회의 안건**을 담는다.
> 코드에서 직접 확인하지 못한 항목은 추측하지 않고 "TODO: 확인 필요"로 표시했다.

---

## 1. 공통 응답 포맷 (`types/index.ts`)

```typescript
export interface ApiResponse<T = unknown> {
  success: boolean;
  data?: T;
  message?: string;
  error?: string;
}

export function ok<T>(data: T, message?: string): ApiResponse<T> {
  return { success: true, data, message };
}
export function fail(error: string, message?: string): ApiResponse<never> {
  return { success: false, error, message };
}
```

**HTTP 상태코드 컨벤션** (모든 라우트 공통)

| 상태 | 의미 | 비고 |
|---|---|---|
| `200` | 조회/수정/삭제 성공 | |
| `201` | 생성 성공 | 회원가입, 상품/셀러/리뷰/위시리스트 생성, 주문 생성, 장바구니 추가 |
| `400` | 요청 값 검증 실패 | 필수값 누락, 형식 오류, 범위 초과 등 |
| `401` | 로그인 필요 | `const session = await auth(); if (!session) return fail("로그인 필요")` |
| `403` | 권한 없음 | role 불일치, 본인 리소스 아님, 승인되지 않은 셀러 등 |
| `404` | 리소스 없음 | |
| `409` | 충돌 | 재고 부족, 중복 가입/등록, 이미 처리된 주문 |
| `500` | 서버 오류 | |
| `502` | 외부 서비스(Stripe) 오류 | 환불 실패 등 |
| `503` | 기능 미설정 | `/api/upload`에서 S3 환경변수 없을 때 |

프론트 공통 처리 패턴 (`docs/FRONTEND_GUIDE.md`에 이미 기재됨, 그대로 유효):

```typescript
const res = await fetch("/api/cart", { method: "POST", headers: {"Content-Type":"application/json"}, body: JSON.stringify({ productId, quantity: 1 }) });
const json = await res.json();
if (!json.success) { alert(json.error); return; }
// json.data 사용
```

---

## 2. 인증 흐름 (NextAuth v5, `auth.ts`)

- Provider: **Credentials only** (이메일+비밀번호, bcrypt). 소셜 로그인 없음. `TODO: 확인 필요` — 향후 소셜 로그인 추가 계획 여부.
- `session: { strategy: "jwt" }` — JWT 세션. `jwt()`/`session()` 콜백에서 `token.id`, `token.role`을 세션에 주입.
- `pages: { signIn: "/login", error: "/login" }`

**세션 발급/로그인 (클라이언트)** — `app/(auth)/login/page.tsx` 실제 패턴:

```typescript
import { signIn } from "next-auth/react";

const result = await signIn("credentials", {
  email, password, redirect: false,
});
```

**세션 확인 — 서버 컴포넌트 / API 라우트**

```typescript
import { auth } from "@/auth";
const session = await auth();
if (!session) redirect("/login"); // 또는 401 응답
```

**세션 확인 — 클라이언트 컴포넌트** (`app/layout.tsx`에 `<SessionProvider>`로 전역 래핑되어 있음)

```tsx
"use client";
import { useSession } from "next-auth/react";
const { data: session } = useSession();
if (session?.user.role === "ADMIN") { /* ... */ }
```

`session.user`에는 `id`, `email`, `name`, `image`, `role`이 포함된다 (`types/index.ts`의 NextAuth 모듈 확장 참고).

### Role 별 접근 가능 라우트

| Role | 페이지 접근 (proxy.ts 강제) | API 접근 (라우트 내부 role 체크) |
|---|---|---|
| `CONSUMER` (기본값) | `/products`, `/cart`, `/orders`, `/checkout` (로그인 필요), 그 외 공개 페이지 | `/api/cart`, `/api/orders`, `/api/wishlist`, `/api/me`, `/api/products/[id]/reviews` (POST), `/api/seller/register` |
| `SELLER` (관리자 승인 후) | CONSUMER 권한 + `/seller/*` (proxy.ts에서 로그인만 강제, role 세분화는 안 함) | `/api/seller/products`, `/api/seller/orders`, `/api/seller/stats`, `/api/upload` (ADMIN도 가능) — 단, 상품 등록/주문 처리는 **Seller.status === "APPROVED"** 추가 확인 |
| `ADMIN` | 모든 영역 + `/admin/*` (proxy.ts가 role !== "ADMIN"이면 `/`로 리다이렉트) | `/api/admin/*`, `/api/upload` |

**주의 (proxy.ts 실제 코드 기준)**: `protectedPaths = ["/seller", "/orders", "/checkout"]`은 "로그인 여부"만 검사한다 — SELLER role인지는 검사하지 않는다. `/seller/*` 페이지의 role 세분화(CONSUMER가 URL 직접 접근 시 차단)는 각 서버 컴포넌트/API 라우트 내부 로직에 위임되어 있다. 프론트에서 `/seller/*` 페이지 진입 시 역할 체크를 페이지 레벨에서도 반드시 확인할 것.

---

## 3. API 엔드포인트 전체 목록 (app/api/** 26개 라우트, 코드 재확인 완료)

> 상세 request/response JSON 스키마는 `docs/API.md` 참고. 단, 아래 2가지는 `docs/API.md`에 **누락/불일치**가 있어 이 문서에서 보강함:
> 1. `GET /api/products`의 `sort` 파라미터: API.md는 `newest/price_asc/price_desc/popular` 4개만 기재하고 있으나, 실제 코드(`lib/productQuery.ts`)에는 `name_asc`, `name_desc`, `stock_desc` 3개가 더 있어 총 7개.
> 2. API.md에 아래 4개 라우트가 **문서화되어 있지 않음**: `GET /api/geo`, `GET /api/rates`, `POST /api/stripe/payment-intent`, `POST /api/webhooks/stripe`.

### 인증

| Method | Path | 인증 | Body | 응답 요약 |
|---|---|---|---|---|
| GET/POST | `/api/auth/[...nextauth]` | - | NextAuth 내부 핸들러 | `handlers.GET/POST` 그대로 위임 (직접 호출 대상 아님, `signIn()`/`useSession()` 경유) |
| POST | `/api/auth/register` | 불필요 | `email, password, name?` | `201 { id, email, name, role: "CONSUMER" }` |
| POST | `/api/auth/forgot-password` | 불필요 | `email` | `200 { message }` (사용자 존재 여부 무관 동일 응답) |
| POST | `/api/auth/reset-password` | 불필요 | `email, token, password` | `200 { message }` |

### 내 정보

| Method | Path | 인증 | Body | 응답 요약 |
|---|---|---|---|---|
| GET | `/api/me` | 로그인 | - | `{ id, email, name, image, role, createdAt }` |
| PATCH | `/api/me` | 로그인 | `name?, currentPassword?, newPassword?` | 변경된 사용자 정보 |

### 상품 / 리뷰

| Method | Path | 인증 | Body / Query | 응답 요약 |
|---|---|---|---|---|
| GET | `/api/products` | 불필요 | Query: `page,limit,category,search,sort(7종),minAbv,maxAbv,vol,isNew,noStock` | `{ products[], total, page, limit }` |
| GET | `/api/products/[id]` | 불필요 | - | 상품 상세 + 리뷰 20개 (ACTIVE만) |
| GET | `/api/products/[id]/reviews` | 불필요 | Query: `page,limit` | `{ reviews[], total, page, totalPages }` |
| POST | `/api/products/[id]/reviews` | 로그인(구매자만) | `rating(1-5), content?` | 리뷰 생성, `403` 구매이력 없음, `409` 중복 |
| DELETE | `/api/products/[id]/reviews` | 로그인(본인/ADMIN) | `reviewId` | `200 { data: null }` |

### 장바구니

| Method | Path | 인증 | Body | 응답 요약 |
|---|---|---|---|---|
| GET | `/api/cart` | 로그인 | - | `{ id, items[], totalKrw }` (없으면 자동 생성) |
| POST | `/api/cart` | 로그인 | `productId, quantity?(1-100)` | `201`, 재고 초과 시 `409` |
| DELETE | `/api/cart` | 로그인 | `itemId` | `200 { data: null }` |

### 주문 / 결제

| Method | Path | 인증 | Body | 응답 요약 |
|---|---|---|---|---|
| GET | `/api/orders` | 로그인 | - | 내 주문 목록 |
| POST | `/api/orders` | 로그인 | `shippingAddress{name,phone,country,postalCode,address1,address2?,city}, currency?` | 장바구니 기반 주문 생성 + 재고 차감 + 장바구니 비움 + **주문확인 이메일 자동발송** |
| POST | `/api/orders/[id]/cancel` | 로그인(본인) | - | PENDING→CANCELLED, PAID→Stripe 환불 후 REFUNDED. PREPARING 이상 취소 불가 |
| POST | `/api/stripe/payment-intent` | 로그인(본인 주문) | `orderId` | Stripe PaymentIntent 생성, `{ clientSecret }`. `order.status !== PENDING`이면 `409` |
| POST | `/api/webhooks/stripe` | Stripe 서명 검증 (세션 인증 아님) | Stripe 이벤트 원문 | `payment_intent.succeeded` 수신 시 Order를 `PAID`로 갱신 |

### 위시리스트 (⚠ 4번 섹션 참고 — 프론트 미연동 이슈)

| Method | Path | 인증 | Body | 응답 요약 |
|---|---|---|---|---|
| GET | `/api/wishlist` | 로그인 | - | 내 위시리스트(상품 정보 포함) |
| POST | `/api/wishlist` | 로그인 | `productId` | upsert (중복 시 무시) |
| DELETE | `/api/wishlist` | 로그인 | `productId` | 제거 |

### 이미지 업로드

| Method | Path | 인증 | Body | 응답 요약 |
|---|---|---|---|---|
| POST | `/api/upload` | SELLER/ADMIN | `multipart/form-data: file` | S3 업로드, `{ url }`. jpg/jpeg/png/webp/avif, 10MB 제한. AWS 환경변수 없으면 `503` |

### 셀러

| Method | Path | 인증 | Body | 응답 요약 |
|---|---|---|---|---|
| POST | `/api/seller/register` | 로그인(CONSUMER만) | `businessName, businessNo?` | 셀러 신청, 초기 `PENDING` |
| GET | `/api/seller/products` | SELLER | Query: `status?` | 내 상품 목록 |
| POST | `/api/seller/products` | SELLER(APPROVED) | 상품 필드 전체 | 상품 등록, 초기 `PENDING` |
| GET/PUT/DELETE | `/api/seller/products/[id]` | SELLER(본인 상품만) | PUT은 변경 필드만 | 조회/수정/삭제 |
| GET | `/api/seller/orders` | SELLER(APPROVED) | - | 내 상품 포함 주문(PENDING 제외) |
| PATCH | `/api/seller/orders/[id]` | SELLER(APPROVED, 본인 상품 주문만) | `status?, trackingNumber?` | 상태 전이 검증(표 아래 참고) + SHIPPED 시 **배송 이메일 자동발송** |
| GET | `/api/seller/stats` | SELLER(APPROVED) | - | 매출/수수료(10%) 통계, 최근 200건 |

셀러 주문 상태 전이 허용표 (`app/api/seller/orders/[id]/route.ts`):

```
PAID → PREPARING, CANCELLED
PREPARING → SHIPPED, CANCELLED
SHIPPED → DELIVERED
```

### 어드민

| Method | Path | 인증 | Body | 응답 요약 |
|---|---|---|---|---|
| GET | `/api/admin/products` | ADMIN | Query: `status?` | 전체 상품 + 셀러 정보 |
| PATCH | `/api/admin/products` | ADMIN | `productId, status` | 상품 상태 변경 |
| GET | `/api/admin/sellers` | ADMIN | Query: `status?` | 전체 셀러 목록 |
| PATCH | `/api/admin/sellers/[id]` | ADMIN | `status: APPROVED\|SUSPENDED` | APPROVED→role SELLER 전환+**승인 이메일 발송**, SUSPENDED→role CONSUMER 강등 |
| POST | `/api/admin/orders/[id]/refund` | ADMIN | `reason?` | 강제 환불(Stripe) + `REFUNDED` |

### 기타 (외부 연동)

| Method | Path | 인증 | 설명 |
|---|---|---|---|
| GET | `/api/geo` | 불필요 | 국가 코드 조회 (아래 4번 섹션) |
| GET | `/api/rates` | 불필요 | KRW 기준 환율표. 1시간 캐시, 실패 시 하드코딩 fallback |

---

## 4. 국가별 테마 시스템 — 프론트 연동 방법

**흐름 (proxy.ts → 서버 컴포넌트 → 클라이언트)**

```
요청 → proxy.ts
  1차: x-vercel-ip-country 헤더 확인 (Vercel Edge 배포 시)
  2차: 없으면 x-real-ip/x-forwarded-for로 ipinfo.io 조회 (가비아 등 non-Vercel 배포용)
  → 위 둘 다 실패 시 "DEFAULT"
  → 요청 헤더에 x-country-code 세팅
  → NextResponse.next({ request: { headers: requestHeaders } })
```

**서버 컴포넌트에서 테마 가져오기** (`docs/FRONTEND_GUIDE.md`에 이미 기재된 패턴, 코드 확인 결과 정확함):

```tsx
import { headers } from "next/headers";
import { countryToTheme } from "@/lib/geo";
import { getTheme } from "@/components/theme";
import { ThemeProvider } from "@/components/theme/ThemeProvider";

export default async function MainLayout({ children }: { children: React.ReactNode }) {
  const h = await headers();
  // 주의: 배포 환경에 따라 x-vercel-ip-country 또는 x-country-code 둘 다 확인해야 함
  const country = h.get("x-vercel-ip-country") ?? h.get("x-country-code") ?? "DEFAULT";
  const theme = getTheme(countryToTheme(country));
  return <ThemeProvider theme={theme}>{children}</ThemeProvider>;
}
```

**클라이언트에서 국가 조회가 필요한 경우** — `/api/geo` 호출 (`lib/geo.ts`의 `fetchCountryCode()` 사용):

```typescript
import { fetchCountryCode } from "@/lib/geo";
const country = await fetchCountryCode(); // "/api/geo" 호출, 실패 시 "DEFAULT"
```

`/api/geo`는 `x-vercel-ip-country` → `x-country-code`(proxy 주입) → `ip-api.com` 순으로 폴백한다 (`proxy.ts`의 ipinfo.io와 다른 별도 폴백 API를 사용함에 유의 — `TODO: 확인 필요`, 두 폴백 소스가 다른 이유는 코드만으로 불명확, 회의에서 의도 확인 권장).

컴포넌트에서 테마 사용: `useTheme()` 훅 (`components/theme/ThemeProvider.tsx`, `ThemeProvider` 내부에서만 사용 가능, 클라이언트 전용).

---

## 5. 장바구니 상태관리 (Zustand, `store/cartStore.ts`)

- `persist` 미들웨어, localStorage 키: **`jmd-cart`**
- 실제 연동 패턴 확인(`components/products/AddToCartButton.tsx`): **서버 API(`/api/cart` POST) 호출 성공 후 → `useCartStore.addItem()`으로 로컬 상태도 갱신**. 즉 서버가 source of truth이고 클라이언트 스토어는 UI 반영용 미러 — 프론트에서 이 패턴을 유지해야 서버/로컬 불일치가 안 생긴다.
- 주요 액션: `setItems`(서버 데이터로 전체 교체 — 로그인 직후 `/api/cart` GET 결과로 동기화할 때 사용), `addItem`, `updateQuantity`, `removeItem`, `clear`, `openCart`/`closeCart`, `totalCount()`, `totalKrw()`.
- `TODO: 확인 필요` — 로그인 시점에 `setItems`로 서버 카트를 불러와 로컬과 동기화하는 코드가 실제로 어디서 호출되는지 이번 조사에서는 미확인. 다른 기기/브라우저 로그인 시 카트가 안 보이는 문제가 있다면 이 지점을 점검해야 함.

---

## 6. 위시리스트 상태관리 — ⚠ 프론트-백엔드 미연동 발견

코드 확인 결과, **위시리스트는 카트와 달리 서버와 완전히 분리되어 있다.**

- 백엔드: `/api/wishlist` (GET/POST/DELETE) + `Wishlist` Prisma 모델 — 로그인 사용자별 서버 DB에 저장.
- 프론트: `store/wishlistStore.ts` (Zustand, localStorage 키 **`jmd-wishlist`**) + `components/products/WishlistButton.tsx`.
- **`WishlistButton.tsx`는 `/api/wishlist` API를 전혀 호출하지 않는다.** `toggle()` 액션이 로컬 스토어만 갱신한다. 즉 현재 위시리스트는:
  - 로그인 여부와 무관하게 브라우저별로만 저장됨 (기기 변경/브라우저 변경 시 사라짐)
  - 서버의 `/api/wishlist` 데이터와 절대 동기화되지 않음
  - 셀러가 상품을 삭제해도 로컬 위시리스트에는 남아있을 수 있음 (서버 API는 상품 정보를 JOIN해서 주지만 로컬 스토어는 추가 당시 스냅샷만 들고 있음)

**회의에서 반드시 결정 필요**: (a) `WishlistButton`을 서버 API 호출로 교체할지, (b) 로컬 전용으로 유지하고 서버 API는 별도 용도(관리자 통계 등)로만 쓸지 방향을 정해야 한다. STATE.md에 "위시리스트 UI 미완료"로 기재된 항목이 바로 이 지점이다.

---

## 7. 이미지 업로드 (`/api/upload`)

```typescript
async function uploadImage(file: File): Promise<string> {
  const form = new FormData();
  form.append("file", file);
  const res = await fetch("/api/upload", { method: "POST", body: form }); // Content-Type 직접 설정 금지
  const json = await res.json();
  if (!json.success) throw new Error(json.error);
  return json.data.url; // https://{AWS_S3_BASE_URL}/products/{timestamp}-{filename}
}
```

- 인증: SELLER 또는 ADMIN만 (401/403)
- 확장자 제한: jpg, jpeg, png, webp, avif (그 외 `400`)
- 용량 제한: **10MB** (초과 시 `400`)
- **로컬/현재 환경에서 실제로 동작하지 않을 수 있음** — 8번 섹션 참고.

---

## 8. 환경설정 관련 프론트 영향사항 (현재 제약사항 — 오늘 회의에서 반드시 공유)

코드 조사 결과 확인된 사실:

| 항목 | 현재 상태 | 프론트 영향 |
|---|---|---|
| DB | `prisma/schema.prisma` provider가 아직 **`sqlite`** (`.env`/`.env.local`의 `DATABASE_URL`도 `file:./dev.db`). Supabase PostgreSQL 미전환 | 로컬 개발은 정상 동작하지만, 프로덕션 배포(Postgres) 전제로 만든 프론트 코드가 있다면 로컬에서 재현 안 되는 차이가 있을 수 있음 |
| 이메일(Resend) | `lib/email.ts` 코드는 완성돼 있으나 `.env`/`.env.local`에 `RESEND_API_KEY`, `EMAIL_FROM`이 **없음** (`.env.example`에만 존재) | 로컬에서 주문확인/배송알림/셀러승인/비밀번호재설정 이메일이 **발송되지 않는다** (단, 실패해도 예외를 던지지 않고 로그만 남기므로 주문 자체는 정상 진행됨 — 프론트에서 "이메일 발송됨" UI 문구를 넣더라도 실제 수신은 안 될 수 있음을 인지) |
| 이미지 업로드(S3) | `/api/upload`가 `AWS_ACCESS_KEY_ID` 등을 참조하나 로컬 `.env`/`.env.local`에 AWS 관련 키 자체가 없음 | 로컬에서 이미지 업로드 시 `503`("이미지 업로드가 설정되지 않았습니다") 응답 예상. 프론트 업로드 UI 개발/테스트 시 이 케이스를 별도로 처리해야 함 |
| Geo 감지(IPINFO) | `proxy.ts`가 `IPINFO_TOKEN` 없이도 동작은 하나(익명 한도 50k/월), 로컬 `.env`/`.env.local`에 `IPINFO_TOKEN` 없음. 로컬은 대부분 `127.0.0.1`이라 애초에 ipinfo 호출 자체가 스킵되고 `DEFAULT` 테마로 감 | 로컬 개발 시 테마가 항상 `DEFAULT`(다크 미니멀)로 보임 — 국가별 테마 확인이 필요하면 `x-country-code` 헤더를 강제로 주입하는 방법을 팀과 합의 필요 |
| Stripe | `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`는 `.env`/`.env.local`에 키 자체는 존재(값 내용은 미확인) | `TODO: 확인 필요` — 실제 테스트 키가 유효한지는 이번 조사 범위(값 미열람) 밖. 결제 플로우 테스트 전 확인 필요 |

---

## 9. 오늘 회의 체크리스트 / 안건

실제 미완료·미확정 사항(STATE.md, ACTIVE_CONTEXT.md, 코드 조사 결과) 기반:

- [ ] **위시리스트 연동 방향 결정** — `WishlistButton`을 서버 API(`/api/wishlist`)와 연결할지, 로컬 전용 유지할지 (6번 섹션)
- [ ] **CORS/도메인 확정** — 현재 프론트/백엔드가 동일 Next.js 앱(Same-Origin)이라 CORS 설정 코드 자체가 없음. 별도 프론트 앱(SPA)으로 분리할 계획이 있다면 CORS 정책 신규 설계 필요, 아니면 현행 유지로 확정
- [ ] **가비아 VPS 배포 시 프론트 빌드 방식** — `next build` → `output: standalone` → Docker 멀티스테이지 → PM2(`ecosystem.config.js`) 방식으로 이미 구성됨(1777dd8). 프론트 개발자가 별도 정적 빌드/배포 파이프라인을 쓸 필요 없이 동일 Next.js 빌드에 포함되는 구조임을 공유하고, 로컬 개발 환경(sqlite) vs 배포 환경(postgresql 예정) 차이를 인지시킬 것
- [ ] **에러 응답 처리 컨벤션 합의** — `{ success:false, error }` 형태를 프론트 공통 fetch 래퍼로 통일할지, 페이지별 개별 처리로 둘지
- [ ] **페이지네이션/필터 파라미터 규칙 공유** — `page,limit,category,search,sort(7종),minAbv,maxAbv,vol,isNew,noStock` (`lib/productQuery.ts`), 페이지 이동 시 5개 필터 파라미터를 URL에 유지해야 함 (최근 버그 수정 완료, 26abcb7)
- [ ] **이미지 업로드 용량/포맷 제한 안내** — 10MB, jpg/jpeg/png/webp/avif만 허용. 프론트 업로드 폼에서 사전 검증(클라이언트 사이드) 추가할지 논의
- [ ] **남은 미완료 페이지 담당 분담** — 비밀번호 재설정 페이지(`app/(auth)/reset-password/`, 현재 untracked로 존재 — 완성도 `TODO: 확인 필요`, 이번 조사는 API만 검증함), 위시리스트 UI (6번 항목과 연동해서 재논의)
- [ ] **환경변수(RESEND/AWS/IPINFO) 로컬 배포 여부** — 프론트가 로컬에서 이메일/업로드/Geo 기능을 테스트해야 한다면 백엔드팀에 테스트용 키 공유 요청할지, 목업으로 대체할지 결정
- [ ] **주문 상태 흐름 UI 반영** — `PENDING→PAID→PREPARING→SHIPPED→DELIVERED`, 취소는 `PENDING/PAID`만 가능(`PREPARING` 이상은 셀러 문의 안내 필요) — 주문 상세/목록 UI에서 상태별 버튼 노출 조건 정의
- [ ] **Stripe 클라이언트 연동 확인** — `@stripe/react-stripe-js`, `@stripe/stripe-js` 패키지는 설치돼 있으나 프론트 결제 UI(Elements 등) 구현 여부는 `TODO: 확인 필요` (이번 조사는 API 라우트만 확인함, 별도 확인 권장)
