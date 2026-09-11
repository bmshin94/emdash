# EmDash 분석 정리 노트

> 이 문서는 EmDash CMS 저장소를 분석하고 정리한 내용입니다.
> 작성일: 2026-09-11

---

## 🔗 관련 링크

| 구분 | 주소 |
| --- | --- |
| **내 저장소 (포크)** | https://github.com/bmshin94/emdash |
| **작업 브랜치** | `claude/quirky-franklin-waayx1` |
| 원본 저장소 | https://github.com/emdash-cms/emdash |
| 공식 문서 | https://docs.emdashcms.com/ |
| 문서 MCP 서버 | https://docs.emdashcms.com/mcp |
| 템플릿 저장소 | https://github.com/emdash-cms/templates |
| 플러그인 레지스트리 (실험) | https://registry.emdashcms.com |

---

## 1. EmDash란?

**Astro + Cloudflare 기반의 풀스택 TypeScript CMS.**

`package.json`의 한 줄 소개: *"Agent-portable reimplementation of WordPress on Astro"*

한마디로 **워드프레스를 요즘 기술로 다시 만든 것**입니다. 글쓰기 화면, REST API,
로그인, 미디어 라이브러리, 플러그인 시스템이 전부 포함되어 있고, Astro 프로젝트에
한 줄만 추가하면 됩니다.

```javascript
// astro.config.mjs
import emdash from "emdash/astro";
import { d1 } from "emdash/db";

export default defineConfig({
	integrations: [emdash({ database: d1() })],
});
```

- 라이선스: MIT (상업적 이용 자유)
- 제작자: Matt Kane (Astro 코어 팀)
- 상태: **beta preview**
- 규모: 총 4,189개 파일 / TypeScript 2,519개 / React(tsx) 309개 / Astro 271개

### 워드프레스와의 차이점

| 항목 | 워드프레스 | EmDash |
| --- | --- | --- |
| 언어 | PHP | TypeScript |
| 플러그인 보안 | 전체 권한 (취약점의 96%가 플러그인) | **샌드박스 격리 + 권한 선언** |
| 콘텐츠 저장 | HTML 덩어리 | **Portable Text (구조화 JSON)** |
| 인증 | 아이디/비밀번호 | **패스키(WebAuthn) 우선** |
| AI 연동 | 별도 플러그인 | **MCP 서버 내장** |
| 호스팅 | PHP 서버 필요 | Cloudflare Workers 또는 Node.js |

플러그인은 요청한 권한만 사용할 수 있습니다.

```typescript
definePlugin({
	id: "notify-on-publish",
	capabilities: ["read:content", "email:send"], // 이것만 가능
	hooks: {
		"content:afterSave": async (event, ctx) => { /* ... */ },
	},
});
```

---

## 2. 폴더 구조

```
packages/           프로그램 본체 (20개 패키지)
  core/             Astro 통합, API, DB, CLI, MCP 서버
  admin/            관리자 화면 (React SPA)
  auth/             인증 (패스키/OAuth/매직링크)
  plugins/          기본 플러그인 12개
  cloudflare/       D1 + R2 + Worker Loader 어댑터
  marketplace/      플러그인 장터 Worker
  registry-*/       AT Protocol 기반 분산형 레지스트리
  x402/             결제 프로토콜 통합
  gutenberg-to-portable-text/   워드프레스 블록 변환기
  create-emdash/    프로젝트 생성기

templates/          시작 템플릿 10개 (blog / marketing / portfolio / starter / blank)
demos/              예제 사이트 6개 (simple = SQLite로 바로 실행 가능)
docs/               공식 문서 사이트 (Starlight)
skills/             AI 에이전트용 스킬 9개
e2e/, acceptance/   Playwright 테스트
apps/               릴리즈 자동화, 라벨러 등 부가 서비스
```

---

## 3. 설치 및 사용법

### 새 사이트 만들기 (권장)

```bash
npm create emdash@latest
cd my-emdash-site
npm install
npm run dev
```

→ `http://localhost:4321/_emdash/admin` 접속
→ 설치 마법사에서 사이트 제목 / 설명 / 이메일 입력
→ 패스키 등록 (Touch ID, Face ID, Windows Hello, 보안키)

**요구 사항: Node.js 22.16 이상 (홀수 버전 미지원)**

### 이 저장소(원본 소스) 실행하기

```bash
pnpm install
pnpm build
pnpm --filter emdash-demo seed
pnpm --filter emdash-demo dev
```

Cloudflare 계정 없이 Node.js + SQLite로 동작합니다.

### 개발 명령어

```bash
pnpm test          # 전체 테스트
pnpm typecheck     # 타입 체크
pnpm lint:quick    # 빠른 린트 (1초 미만)
pnpm format        # 포맷팅 (oxfmt, 탭 사용)
```

### 배포

- **Cloudflare**: README의 "Deploy to Cloudflare" 버튼
- **일반 서버**: Node.js + SQLite (`Dockerfile`, `compose.yaml` 포함)

> 플러그인 기능에는 Dynamic Workers가 필요하며, Cloudflare 유료 플랜(월 $5부터)이
> 필요합니다. `wrangler.jsonc`의 `worker_loaders` 블록을 주석 처리하면 플러그인 없이
> 무료로 운영할 수 있습니다.

---

## 4. 플러그인? 스킬? MCP? → **본체입니다**

EmDash는 셋 중 하나가 아니라 **본체(CMS)** 이며, 세 가지를 모두 품고 있습니다.

| 구분 | 위치 | 설명 |
| --- | --- | --- |
| 본체 | `packages/` | CMS 자체 |
| 플러그인 | `packages/plugins/` | forms, embeds, SEO, audit-log 등 12개 |
| 스킬 | `skills/` | AI 에이전트용 작업 설명서 9개 |
| MCP 서버 | `packages/core/src/mcp/server.ts` | AI가 사이트를 직접 조작 |

### 내장 MCP 서버 2종

**1) 내 사이트 MCP** — `https://내사이트.com/_emdash/api/mcp`
콘텐츠, 스키마, 미디어, 분류체계, 메뉴, 리비전, 설정을 AI가 직접 관리합니다.
기본 활성화되어 있고, 끄려면 `emdash({ mcp: false })`.

**2) 공식 문서 MCP** — `https://docs.emdashcms.com/mcp`

```bash
claude mcp add --transport http emdash-docs https://docs.emdashcms.com/mcp
```

---

## 5. API 토큰

| 상황 | 토큰 필요 여부 |
| --- | --- |
| 로컬 개발 (localhost) | 불필요 (dev bypass 자동 적용) |
| 관리자 화면 사용 | 불필요 (패스키 로그인) |
| Claude / ChatGPT 연결 | **필요** |
| 원격 서버에 CLI 명령 | **필요** |

### 토큰 형식

```
ec_pat_xxxx   개인 액세스 토큰 (관리자 화면에서 발급)
ec_oat_xxxx   OAuth 액세스 토큰
ec_ort_xxxx   OAuth 갱신 토큰
```

서버에는 SHA-256 해시만 저장되며, 접두사 덕분에 로그나 시크릿 스캐너에서 식별됩니다.

### 스코프 (권한 범위)

`content:read`, `content:write`, `media:read`, `media:write`, `schema:read`,
`schema:write`, `taxonomies:manage`, `menus:manage`, `settings:read`,
`settings:manage`, `mcp:tools`, `mcp:tools:<pluginId>`, `admin`

스코프는 사용자 역할(Administrator / Editor / Author / Contributor / Subscriber)과
별개로 검사되므로, **스코프가 역할 이상의 권한을 부여하지 않습니다.**

### CLI 인증 순서

1. `--token` 플래그
2. `EMDASH_TOKEN` 환경변수
3. `~/.config/emdash/auth.json` (`emdash login`으로 저장)
4. localhost일 경우 dev bypass 자동 인증

> OpenAI API 키 같은 외부 AI 서비스 키는 필요 없습니다. EmDash는 CMS입니다.

---

## 6. 왜 주목받는가

> 참고: GitHub 스타 수는 직접 확인하지 못했습니다. 아래는 저장소 내용에 근거한 분석입니다.

1. **"워드프레스 킬러" 포지션** — 전 세계 웹사이트 40%를 차지하는 워드프레스에 정면 도전
2. **실제 문제 해결** — 워드프레스 보안 취약점의 96%가 플러그인에서 발생
   ([Patchstack 2024](https://patchstack.com/whitepaper/state-of-wordpress-security-in-2024/)).
   샌드박스 + 권한 선언으로 구조적 해결
3. **유명 제작자** — Astro 코어 팀의 Matt Kane
4. **AI 네이티브 설계** — MCP 내장, 에이전트 스킬 동봉, CLI 자동화, `AGENTS.md`(28KB)
5. **최신 기술 스택** — Astro + Cloudflare Workers + D1 + React 19 + TypeScript 6 beta

---

## 7. 로컬 에이전트 구축에 도움이 되는가

### A. EmDash를 에이전트가 조종하게 하기 — 이미 완성됨

```bash
# 관리자 화면에서 ec_pat_ 토큰 발급 후
claude mcp add --transport http my-site https://내사이트.com/_emdash/api/mcp
```

### B. 내 에이전트 만들 때 참고 자료로 — 이쪽이 더 가치 있음

| 배울 것 | 위치 |
| --- | --- |
| MCP 서버 설계 (Zod 스키마, 툴 정의) | `packages/core/src/mcp/server.ts` |
| AI 권한 제한 (스코프 시스템) | `packages/auth/src/rbac.ts`, `core/src/auth/api-tokens.ts` |
| AI 코드 샌드박싱 | `packages/cloudflare`, `packages/workerd` |
| AI용 문서 작성법 | `AGENTS.md`, `skills/` |

특히 참고할 만한 기법 — 툴 설명 안에 재시도 규칙을 명시:

```typescript
const REV_PARAM_DESCRIPTION =
	"Revision token proving you have read the current item. Call content_get " +
	"first and pass back the _rev it returns. If this call fails with CONFLICT " +
	"the item changed in the meantime: call content_get again and retry with " +
	"the new token.";
```

에이전트는 툴 스키마만 보기 때문에, 사용 규칙을 설명문에 넣어 실수를 방지합니다.

---

## 8. 수익화 아이디어

### 우선순위 요약

| 순위 | 아이디어 | 난이도 | 현실성 |
| --- | --- | --- | --- |
| 1 | 중소기업 홈페이지 제작 | 낮음 | 높음 |
| 2 | 워드프레스 이전 대행 | 중간 | 높음 |
| 3 | 한국어 콘텐츠 선점 | 낮음 | 중간 (영업 채널) |
| 4 | 한국형 테마 판매 | 중간 | 중간 (1번의 부산물) |
| 5 | 플러그인 개발 | 높음 | 중간 (수주 개발 형태) |
| 보류 | AI 페이월 (x402) | 높음 | 시기상조 |

### 1) 중소기업 홈페이지 제작

근거: `templates/`에 완성 템플릿 10개. 마케팅 템플릿은 히어로, 기능 그리드,
가격표, FAQ, 문의폼까지 포함.

강점:
- 클라이언트가 관리자 화면에서 직접 수정 → 유지보수 문의 감소
- 서버 비용 월 5천원 수준 (Cloudflare)
- Cloudflare 엣지 배포로 속도 확보
- 샌드박스 + 패스키로 보안 리스크 감소

예상 단가 (추정치):

| 규모 | 제작비 | 월 유지비 |
| --- | --- | --- |
| 소상공인 (5페이지) | 80~150만원 | 3~5만원 |
| 중소기업 (10~20페이지) | 200~400만원 | 5~10만원 |
| 리뉴얼 + 이전 | 300~600만원 | 10만원~ |

주의: beta 단계이므로 대기업·공공기관은 피하고, 계약서에 베타 소프트웨어 기반임을 명시.

### 2) 워드프레스 이전 대행

이미 준비된 도구:

```
packages/gutenberg-to-portable-text/   워프 블록 변환기
packages/core/src/cli/wxr/             WXR 파서
skills/wordpress-theme-to-emdash/      테마 이식 AI 스킬
skills/wordpress-plugin-to-emdash/     플러그인 이식 AI 스킬
```

이전 경로 3가지: WXR 파일 / 워드프레스 REST API / WordPress.com

```bash
npx emdash import wxr export.xml
```

예상 단가: 블로그 50~100만원, 기업 사이트 200~500만원, 플러그인 이식 포함 500만원~

주의: 우커머스 등 플러그인 의존도가 높은 사이트는 이전이 어렵습니다.
먼저 "진단 서비스"(30만원 수준)로 가능 여부를 확인한 뒤 본계약을 권장합니다.

### 3) 한국어 콘텐츠 선점

한국어 자료가 거의 없어 선점 효과가 큽니다.

추천 주제:
- "워드프레스 버리고 EmDash로 갈아탄 후기"
- "5분 만에 블로그 만들기"
- **"AI(Claude)에게 내 블로그 관리 맡기기 — MCP 완전 정복"** (킬러 콘텐츠)

직접 수익보다 **제작 의뢰 유입 채널**로서의 가치가 큽니다.

### 4) 한국형 테마 판매

마켓플레이스 DB에 `themes` 테이블이 존재합니다. 현재 한국형 테마는 없습니다.

후보: 병원/한의원, 카페/음식점, 학원, 1인 브랜드, 공방/작가

단, EmDash 사용자 수가 아직 적어 테마 단독 판매는 수익이 제한적입니다.
**1번 제작 사업의 재사용 자산**으로 만드는 편이 효율적입니다.

### 5) 플러그인 개발

```bash
npx emdash plugin init      # 뼈대 생성
npx emdash plugin validate  # 검증
npx emdash plugin publish   # 배포
```

한국 시장에 없는 플러그인: 토스페이먼츠/아임포트 결제, 카카오 로그인·알림톡,
네이버 SEO, 네이버 애널리틱스, 택배 조회, 사업자 정보 표시

**중요 — 공식 마켓플레이스에 결제 기능이 없습니다.**
`packages/marketplace/src/db/schema.sql`을 확인한 결과 `authors`, `plugins`,
`plugin_versions`, `plugin_audits`, `plugin_image_audits`, `installs`, `themes`
테이블만 있고 가격/결제 관련 컬럼이 없습니다.

또한 플러그인 유통 구조가 전환 중입니다.

| 구분 | 마켓플레이스 | 레지스트리 |
| --- | --- | --- |
| 상태 | 운영 중 | 실험적(experimental) |
| 구조 | 중앙 집중 | AT Protocol 기반 분산형 |
| 모더레이션 | 운영자가 심사 | 라벨러가 승인 |

공식 문서에 레지스트리가 마켓플레이스를 대체할 예정이라고 명시되어 있습니다.

따라서 유료 판매는 Gumroad 등 외부 채널로 직접 하거나, **무료 배포로 인지도를 쌓아
수주 개발로 연결**하는 방식이 현실적입니다.

### 6) AI 페이월 (x402) — 보류 권장

`packages/x402`는 HTTP 402 Payment Required 기반 결제 프로토콜 통합입니다.

```typescript
x402({
	payTo: "0xYourWallet",   // 수신 지갑
	network: "eip155:8453",  // Base 체인
	defaultPrice: "$0.01",
	botOnly: true,           // 사람은 무료, 봇은 과금
	botScoreThreshold: 30,   // 30 미만이면 봇으로 판정 (범위 1~99)
});
```

동작: Cloudflare Bot Management 점수로 봇을 판별 → 봇이면 402 응답 →
결제 검증(verify) → 정산(settle) → 통과

보류 이유:
- 암호화폐 결제(EVM/Solana) — 국내 세금·규제 이슈
- 기본 facilitator가 테스트넷 (`https://x402.org/facilitator`)
- 실제로 AI 크롤러가 x402로 결제하는 사례가 아직 적음
- Cloudflare Bot Management(유료) 필요

다만 **기술 블로그 소재로는 매우 좋습니다.**

---

## 9. React / PHP 로 만들 수 있는가

### React — 이미 React 기반

`packages/admin`이 React SPA입니다.

```json
"react": "^18.0.0 || ^19.0.0",
"@tanstack/react-query",
"@tanstack/react-router"
```

사이트 화면에도 React를 쓸 수 있습니다.

```javascript
integrations: [react(), emdash({ /* ... */ })]
```

| 영역 | 기술 |
| --- | --- |
| 사이트 화면 | Astro (`.astro`) — React/Vue/Svelte 혼용 가능 |
| 관리자 화면 | React |
| 커스텀 컴포넌트 | React 자유롭게 사용 |

### PHP — 불가능

EmDash는 Cloudflare Workers에서 동작하며, Workers는 JavaScript/TypeScript만
실행합니다. README에도 명시: *"No PHP, no separate hosting tier"*

| 상황 | 가능 여부 |
| --- | --- |
| PHP로 EmDash 구현 | 불가능 |
| 워드프레스(PHP) → EmDash 이전 | 완벽 지원 |
| 기존 PHP 서버 + EmDash REST API 연동 | 가능 |

PHP로 이런 걸 만들고 싶다면 그건 이미 워드프레스입니다. EmDash를 쓰는 이유가
곧 PHP를 쓰지 않기 위해서입니다.

---

## 10. 90일 실행 로드맵

### 1~2주차 — 체험

- [ ] `pnpm install && pnpm build`
- [ ] `pnpm --filter emdash-demo seed && pnpm --filter emdash-demo dev`
- [ ] 관리자 화면 전체 둘러보기
- [ ] 템플릿 3종 실행해보기
- [ ] 내 블로그 하나 실제 배포

### 3~6주차 — 알리기

- [ ] "EmDash 써본 후기" 1편
- [ ] "설치부터 배포까지" 튜토리얼 1편
- [ ] "Claude로 블로그 관리하기(MCP)" 1편
- [ ] 유튜브 5분 영상 1개

### 7~10주차 — 자산 만들기

- [ ] 한국형 테마 1개 (카페 또는 병원)
- [ ] 플러그인 1개 (사업자 정보 표시 — 난이도 낮고 수요 확실)
- [ ] GitHub 공개 + 포트폴리오화

### 11~13주차 — 수익화

- [ ] 지인/소상공인 1건 무료 또는 반값 제작 (레퍼런스 확보)
- [ ] 포트폴리오 페이지 제작
- [ ] 크몽/숨고 등록 및 영업 시작

---

## 11. 리스크 정리

| 리스크 | 대응 |
| --- | --- |
| beta preview 단계 | 소상공인부터 시작, 계약서에 명시 |
| 사용자 수 적음 | 테마·플러그인 단독 판매보다 제작 서비스와 묶기 |
| 마켓플레이스 → 레지스트리 전환 중 | 유통 방식 변경 가능성 주시 |
| 공식 결제 기능 없음 | Gumroad 등 외부 채널로 직접 판매 |
| 한국어 자료 없음 | 리스크이자 선점 기회 |
| 플러그인에 Cloudflare 유료 플랜 필요 | 월 $5, 플러그인 미사용 시 무료 운영 가능 |

---

## 12. 핵심 요약

| 질문 | 답 |
| --- | --- |
| 이게 뭐야? | Astro + Cloudflare 기반 CMS, 워드프레스의 현대적 대안 |
| 설치법 | `npm create emdash@latest` |
| 플러그인/스킬/MCP? | **본체**. 셋 다 내장하고 있음 |
| API 토큰 필요? | 로컬은 불필요 / AI·원격 연동 시 `ec_pat_` 토큰 필요 |
| 왜 주목받나? | 워드프레스 보안 문제 해결 + AI 네이티브 + 유명 제작자 |
| 에이전트 구축에 도움? | MCP 서버 설계 참고 자료로 매우 유용 |
| 수익화? | 1순위 홈페이지 제작, 2순위 워드프레스 이전 대행 |
| React/PHP? | React는 이미 사용 중 / PHP는 불가 (이전은 지원) |

---

*이 문서는 저장소 코드를 직접 확인하여 작성되었습니다.*
