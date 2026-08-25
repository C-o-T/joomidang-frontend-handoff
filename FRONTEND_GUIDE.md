# 주미당 프론트엔드 개발 가이드

외부 협업 개발자를 위한 필수 정보 모음.

---

## 테마 시스템

주미당은 접속자의 IP 기반 국가를 감지해 나라별로 다른 UI 테마를 자동 적용한다.
각 테마는 유명 이커머스 플랫폼의 UX/디자인을 벤치마킹했다.

### 테마 키 목록

| ThemeKey | 대상 국가 | 벤치마킹 플랫폼 | 주 색상 |
|---|---|---|---|
| `KR` | 한국 | 마켓컬리 | 보라 `#5F0080` + 황금 `#F59E0B` |
| `JP` | 일본 | 라쿠텐 | 레드 `#BF0000` + 금색 `#FFD700` |
| `CN` | 중국/홍콩/대만 | 타오바오 | 레드 `#E1251B` + 오렌지 `#FF6000` |
| `US` | 미국/캐나다 | 아마존 | 네이비 `#131921` + 오렌지 `#FF9900` |
| `EU` | 유럽 (영국/독일/프랑스 등) | 잘란도 | 블랙/화이트 미니멀 |
| `SEA` | 동남아 (태국/싱가포르 등) | 쇼피 | 오렌지-레드 그라디언트 |
| `DEFAULT` | 그 외 | 다크 미니멀 | 인디고 `#6C63FF` |

### ThemeConfig 타입

```typescript
interface ThemeConfig {
  key: ThemeKey;       // "KR" | "JP" | "CN" | "US" | "EU" | "SEA" | "DEFAULT"
  name: string;        // 테마 표시명
  locale: string;      // 언어 코드: "ko" | "ja" | "zh" | "en"
  currency: string;    // 통화 코드: "KRW" | "JPY" | "CNY" | "USD" | "EUR" | "SGD"
  colors: {
    primary: string;        // 주 브랜드 색상
    primaryHover: string;   // hover 상태 색상
    secondary: string;      // 보조 색상
    accent: string;         // 강조 색상 (CTA 버튼 등)
    background: string;     // 배경
    surface: string;        // 카드/패널 배경
    text: string;           // 본문 텍스트
    textMuted: string;      // 보조 텍스트
    border: string;         // 테두리
    badge: string;          // 배지 배경
    badgeText: string;      // 배지 텍스트
  };
  layout: {
    heroStyle: "banner" | "grid" | "carousel";
    productCardStyle: "minimal" | "detailed" | "compact";
    navStyle: "top" | "side";
    showFlashSale: boolean;      // 플래시 세일 배너 표시 여부
    showFreeShipping: boolean;   // 무료배송 배너 표시 여부
  };
  fonts: {
    heading: string;
    body: string;
  };
}
```

### useTheme() 훅 사용법

`ThemeProvider` 안에서만 사용 가능하다. 클라이언트 컴포넌트에서 사용.

```tsx
"use client";

import { useTheme } from "@/components/theme/ThemeProvider";

export function MyComponent() {
  const theme = useTheme();

  return (
    <button
      style={{ backgroundColor: theme.colors.primary }}
      className="text-white px-4 py-2 rounded"
    >
      {theme.locale === "ko" ? "구매하기" : "Buy Now"}
    </button>
  );
}
```

### 국가 → 테마 변환 로직 (`lib/geo.ts`)

```typescript
import { countryToTheme } from "@/lib/geo";

// 예시
countryToTheme("KR")  // → "KR"
countryToTheme("JP")  // → "JP"
countryToTheme("CN")  // → "CN"
countryToTheme("HK")  // → "CN"
countryToTheme("US")  // → "US"
countryToTheme("CA")  // → "US"
countryToTheme("DE")  // → "EU"
countryToTheme("TH")  // → "SEA"
countryToTheme(null)  // → "DEFAULT"
```

### 서버 컴포넌트에서 테마 주입 (레이아웃 패턴)

```tsx
// app/(main)/layout.tsx (서버 컴포넌트)
import { headers } from "next/headers";
import { countryToTheme, getTheme } from "@/lib/geo";
import { getTheme as getThemeConfig } from "@/components/theme";
import { ThemeProvider } from "@/components/theme/ThemeProvider";

export default async function MainLayout({ children }: { children: React.ReactNode }) {
  const h = await headers();
  const country = h.get("x-vercel-ip-country") ?? "DEFAULT";
  const themeKey = countryToTheme(country);
  const theme = getThemeConfig(themeKey);

  return (
    <ThemeProvider theme={theme}>
      {children}
    </ThemeProvider>
  );
}
```

---

## 다국어 처리 패턴

별도의 i18n 라이브러리 없이 `theme.locale`로 처리한다.

```tsx
// 단순 분기 패턴
const label = theme.locale === "ko" ? "막걸리" :
              theme.locale === "ja" ? "マッコリ" :
              theme.locale === "zh" ? "马格利" : "Makgeolli";

// 상품명은 DB에 다국어 필드로 저장됨
// nameKo, nameEn, nameJa, nameZh 중 locale에 맞게 선택
function getProductName(product: ProductDTO, locale: string) {
  if (locale === "ja" && product.nameJa) return product.nameJa;
  if (locale === "zh" && product.nameZh) return product.nameZh;
  if (locale === "en") return product.nameEn;
  return product.nameKo; // 기본값: 한국어
}
```

---

## 공통 컴포넌트

### Navbar

국가별 스타일이 자동 적용된다. `theme` prop만 전달하면 된다.

```tsx
import { Navbar } from "@/components/common/Navbar";

<Navbar theme={theme} />
```

내부적으로 `theme.key`에 따라 KR/US/JP/CN/EU/SEA/Default 스타일 중 하나가 렌더링된다.

### Footer

```tsx
import { Footer } from "@/components/common/Footer";

<Footer theme={theme} />
```

### WishlistButton

상품 카드나 상세 페이지에서 사용. 로그인 상태에 따라 API 호출 또는 로그인 페이지 리다이렉트.

```tsx
import { WishlistButton } from "@/components/products/WishlistButton";

<WishlistButton productId={product.id} theme={theme} />
```

---

## 상태 관리 (Zustand)

### useCartStore — 장바구니

`localStorage` 영속화 (`jmd-cart` 키).

```tsx
"use client";
import { useCartStore } from "@/store/cartStore";

function CartIcon() {
  const totalCount = useCartStore((s) => s.totalCount());
  const totalKrw = useCartStore((s) => s.totalKrw());
  const openCart = useCartStore((s) => s.openCart);
  const items = useCartStore((s) => s.items);

  return <button onClick={openCart}>장바구니 ({totalCount})</button>;
}
```

**주요 액션**

| 액션 | 설명 |
|---|---|
| `setItems(items)` | 서버 데이터로 전체 교체 (로그인 후 동기화 시) |
| `addItem(item)` | 아이템 추가 (이미 있으면 수량 합산) |
| `updateQuantity(itemId, qty)` | 수량 변경 (0 이하면 삭제) |
| `removeItem(itemId)` | 아이템 삭제 |
| `clear()` | 전체 비우기 |
| `openCart()` / `closeCart()` | 장바구니 드로어 열기/닫기 |
| `totalCount()` | 전체 수량 합계 |
| `totalKrw()` | 총 금액 (원화) |

### useWishlistStore — 위시리스트

`localStorage` 영속화 (`jmd-wishlist` 키).

```tsx
"use client";
import { useWishlistStore } from "@/store/wishlistStore";

function WishlistToggle({ product }: { product: WishlistItem }) {
  const isWishlisted = useWishlistStore((s) => s.isWishlisted(product.id));
  const toggle = useWishlistStore((s) => s.toggle);

  return (
    <button onClick={() => toggle(product)}>
      {isWishlisted ? "♥" : "♡"}
    </button>
  );
}
```

**주요 액션**

| 액션 | 설명 |
|---|---|
| `toggle(item)` | 추가/제거 토글 |
| `isWishlisted(id)` | 위시리스트 여부 확인 |
| `clear()` | 전체 비우기 |

---

## API 호출 패턴

### 인증 필요 API 호출

쿠키에 세션이 자동 포함되므로 별도 토큰 처리 불필요.

```typescript
// 기본 패턴
const res = await fetch("/api/cart", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ productId, quantity: 1 }),
});

const json = await res.json();

if (!json.success) {
  // json.error: 에러 메시지
  alert(json.error);
  return;
}

// json.data: 응답 데이터
console.log(json.data);
```

### ok()/fail() 응답 처리

```typescript
import type { ApiResponse } from "@/types";

async function fetchCart(): Promise<CartDTO | null> {
  const res = await fetch("/api/cart");
  const json: ApiResponse<CartDTO> = await res.json();

  if (!json.success || !json.data) return null;
  return json.data;
}
```

### 이미지 업로드 (FormData)

```typescript
async function uploadImage(file: File): Promise<string> {
  const form = new FormData();
  form.append("file", file);

  const res = await fetch("/api/upload", {
    method: "POST",
    body: form,
    // Content-Type 헤더 직접 설정하지 말 것 — browser가 boundary 자동 처리
  });

  const json = await res.json();
  if (!json.success) throw new Error(json.error);
  return json.data.url; // S3 URL
}
```

---

## Next.js 16 주의사항 (반드시 숙지)

### 1. proxy.ts (미들웨어)

`middleware.ts`가 `proxy.ts`로 이름이 변경되었다. 함수명도 `proxy`다.

```typescript
// ✅ 올바른 방법 (proxy.ts)
export function proxy(request: NextRequest) { ... }

// ❌ 이 파일명/함수명은 Next.js 16에서 동작하지 않음
// middleware.ts의 export function middleware(...)
```

### 2. params와 searchParams — async 필수

```tsx
// ✅ 서버 컴포넌트 — params/searchParams 모두 await
export default async function Page({
  params,
  searchParams,
}: {
  params: Promise<{ id: string }>;
  searchParams: Promise<{ page?: string }>;
}) {
  const { id } = await params;
  const { page } = await searchParams;
  // ...
}

// ✅ API Route
export async function GET(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  // ...
}
```

### 3. headers(), cookies() — async 필수

```typescript
import { headers, cookies } from "next/headers";

// ✅ 올바른 방법
const h = await headers();
const country = h.get("x-vercel-ip-country");

const c = await cookies();
const token = c.get("session-token");
```

### 4. "use client" 사용 기준

| 사용해야 하는 경우 | 사용하지 않아도 되는 경우 |
|---|---|
| `useState`, `useEffect` 등 React 훅 사용 | 데이터 페칭만 하는 컴포넌트 |
| 이벤트 핸들러 (`onClick`, `onChange` 등) | 정적 UI 렌더링 |
| Zustand 스토어 (`useCartStore` 등) | 서버 데이터를 그대로 렌더링 |
| `useSession()` (NextAuth 클라이언트) | `auth()` 서버 함수 사용 시 |
| `useTheme()` 훅 | `ThemeProvider`에 theme 주입하는 레이아웃 |

```tsx
// ✅ 서버 컴포넌트에서 인증 확인
import { auth } from "@/auth";

export default async function SellerPage() {
  const session = await auth();
  if (!session) redirect("/login");
  // ...
}

// ✅ 클라이언트 컴포넌트에서 세션 사용
"use client";
import { useSession } from "next-auth/react";

function NavButtons() {
  const { data: session } = useSession();
  // ...
}
```

### 5. Geo 감지 — x-vercel-ip-country 헤더

```typescript
// ✅ Next.js 16 방식
const h = await headers();
const country = h.get("x-vercel-ip-country") ?? null;
const themeKey = countryToTheme(country);

// ❌ request.geo는 Next.js 16에서 제거됨
// const { country } = request.geo;
```

---

## 역할(Role) 시스템

| Role | 설명 | 접근 가능 영역 |
|---|---|---|
| `CONSUMER` | 일반 소비자 (기본값) | `/products`, `/cart`, `/orders`, `/wishlist`, `/mypage` |
| `SELLER` | 셀러 (관리자 승인 후) | 위 + `/seller/*` |
| `ADMIN` | 관리자 | 모든 영역 + `/admin/*` |

서버 컴포넌트에서 역할 확인:

```typescript
const session = await auth();
if (session?.user.role !== "SELLER") redirect("/");
```

클라이언트 컴포넌트에서:

```tsx
const { data: session } = useSession();
if (session?.user.role === "ADMIN") { ... }
```

---

## 카테고리 (Category Enum)

| 값 | 한국어 | 영어 | 일본어 | 중국어 |
|---|---|---|---|---|
| `MAKGEOLLI` | 막걸리 | Makgeolli | マッコリ | 马格利 |
| `SOJU` | 소주 | Soju | 焼酎 | 烧酒 |
| `CHEONGJU` | 청주 | Cheongju | 清酒 | 清酒 |
| `YAKJU` | 약주 | Yakju | 薬酒 | 药酒 |
| `FRUIT_WINE` | 과실주 | Fruit Wine | 果実酒 | 果实酒 |
| `OTHER` | 기타 | Other | その他 | 其他 |

---

## 주문 상태 (OrderStatus) 흐름

```
PENDING (주문 생성)
  ↓ Stripe 결제 완료
PAID
  ↓ 셀러 처리 시작
PREPARING
  ↓ 셀러 배송 등록
SHIPPED
  ↓ 셀러 배달 확인
DELIVERED

PENDING / PAID → (소비자 취소) → CANCELLED / REFUNDED
PAID ~ DELIVERED → (어드민 강제 환불) → REFUNDED
```

---

## 자주 쓰는 유틸리티

### 가격 포맷

```typescript
// 원화
const krwFormat = new Intl.NumberFormat("ko-KR", { style: "currency", currency: "KRW" });
krwFormat.format(25000); // "₩25,000"

// 테마 통화로 변환 후 표시
function formatPrice(priceKrw: number, currency: string, totalConverted: number | null) {
  if (currency === "KRW" || !totalConverted) {
    return new Intl.NumberFormat("ko-KR", { style: "currency", currency: "KRW" }).format(priceKrw);
  }
  return new Intl.NumberFormat("en", { style: "currency", currency }).format(totalConverted);
}
```

### 별점 렌더링

```tsx
function StarRating({ rating }: { rating: number }) {
  return (
    <span>
      {Array.from({ length: 5 }, (_, i) => (
        <span key={i} className={i < rating ? "text-yellow-400" : "text-gray-300"}>★</span>
      ))}
    </span>
  );
}
```
