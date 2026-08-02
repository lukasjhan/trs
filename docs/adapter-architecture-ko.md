# TRS Adapter Architecture (설계 논의 문서) — 이슈 #14

> 상태: **논의용 draft (한국어)**. 확정되면 영어로 이슈 [#14](https://github.com/openwallet-foundation-labs/trs/issues/14)에 옮긴다.
> 범위: resolver 내부의 **trust-protocol adapter 계약 + 라우팅(registry)**. 어댑터 구현 자체(#17~#22)와 캐시(#16)는 별도 이슈지만 여기서 경계를 정의한다.

---

## 1. 목적과 경계

TRS의 핵심 가치는 "여러 trust protocol을 **하나의 TRQP 인터페이스**로 브릿지"하는 것이다. 그 브릿지 지점이 바로 이 어댑터 레이어다.

TRQP 스펙 용어로 각 어댑터 = **TRQP Bridge**: `TRQP 쿼리 → 프로토콜 고유 해석 → TRQP 응답 매핑`.

**#14와 #23(root server)은 다른 층이다 — 혼동 주의:**

```
                    ┌─────────────┐
                    │ root server │  "누가 이 쿼리를 풀 수 있나?" (referral, 비표준)
                    └─────────────┘
                     ▲    │  ② 못 풀면 물어봄 / ③ 다른 서버를 알려줌(referral)
                     │    ▼
client ─①TRQP─▶ [ trs resolver ] ──▶ 어느 adapter?  ← #14 (resolver 내부, 로컬 해결)
                               └──▶ ④ referral 받은 registry/resolver로 TRQP
```

- **#23 (root server)**: resolver가 **스스로 해결 못 하면** root에 "이건 어떤 서버가 풀 수 있나?"라고 물어 **referral**을 받는다. DNS의 *recursive resolver ↔ root nameserver* 관계와 같다. **client는 늘 resolver하고만 통신**하고 root를 직접 부르지 않는다.
- **#14 (이 문서)**: resolver로 들어온 쿼리를 "어느 *protocol adapter*로 보낼거냐" (resolver 내부 라우팅)

---

## 2. 레이어 구조 & NestJS monorepo 레이아웃

resolver는 3층으로 나눈다.

| 레이어 | 책임 | 관련 이슈 |
|--------|------|-----------|
| `trqp` | 프로토콜 I/O: `POST /authorization`·`/recognition`, DTO 검증, RFC 7807 filter | #13 |
| `resolver` | 오케스트레이션: 라우팅 → 캐시 → adapter 호출 → 응답 매핑 | #14/#16 |
| `adapters` | adapter 인터페이스 + registry, 각 protocol adapter | #14, #17~#22 |

repo 루트를 **pnpm workspace**로 두고 (1) TRQP 타입 공통 패키지, (2) NestJS monorepo 서버, (3) client SDK를 담는다. 서버는 nest **monorepo 모드**(`nest-cli.json: "monorepo": true`)라 어댑터를 lib로 떨궈 추가한다.

```
<repo 루트>/                          # pnpm workspace (pnpm-workspace.yaml)
  packages/
    trqp-core/                        # #13 @trs/trqp-core — 프레임워크 무관 zod 스키마 + z.infer 타입 (SSOT)
  trs/                                # NestJS monorepo (앱). @trs/trqp-core를 workspace:*로 참조
    apps/
      resolver/                       # 메인 TRQP resolver 앱
      root-server/                    # (나중) #23
    libs/
      adapter-kit/                    # #14 ⭐ 인터페이스·registry·@TrustAdapter·contract test·StaticAdapter
      cache/                          # #16
      adapter-did-web/                # #17
      adapter-did-webvh/              # #18
      adapter-did-webs/               # #19
      adapter-openid-federation/      # #20
      adapter-pki-x509/               # #21
      adapter-eudi-trusted-list/      # #22
    nest-cli.json
  client/                             # trust-resolver SDK. @trs/trqp-core를 workspace:* → tsup가 dist에 인라인
  pnpm-workspace.yaml
```

- **`@trs/trqp-core`(#13)는 nest lib가 아니라 루트 워크스페이스 패키지**다 — client가 nest 바깥이라 lib를 못 쓰기 때문. zod 스키마 하나로 서버 검증(nestjs-zod)과 클라 검증을 공유(SSOT).
- 번들은 **client만** — 퍼블리시 시 `tsup noExternal`로 trqp-core를 인라인 → `trust-resolver` 배포물 자기완결. **서버는 참조만**(번들 X, node_modules로 실행).

---

## 3. 핵심 인터페이스 (`adapter-kit`)

```typescript
// libs/adapter-kit/src/adapter.interface.ts
export interface TrqpContext {
  time?: string;     // RFC 3339
  locator?: string;  // 선택적 힌트
}

export interface AuthorizationInput {
  entityId: string;
  authorityId: string;   // TRQP 지배 권한 (RFC 3986 URI) — 라우팅 키
  action: string;
  resource: string;
  context?: TrqpContext;
}

export interface AuthorizationOutcome {
  authorized: boolean;
  message?: string;
  freshUntil?: Date;   // adapter가 아는 신선도(cert 만료 / list nextUpdate / statement exp) → #16이 TTL로 사용
  evidence?: unknown;  // 감사·추적용 — 로그 only, TRQP 응답엔 미노출 (O-5)
}

// Recognition도 동일 형태 (recognized: boolean)

export interface TrustProtocolAdapter {
  readonly id: string;                    // 'did:web', 'openid-federation', ...
  canHandle(authorityId: string, ctx?: TrqpContext): Promise<boolean>; // 라우팅: 내가 처리하나 (단일 비동기, 결정 1)
  resolveAuthorization(input: AuthorizationInput): Promise<AuthorizationOutcome>;
  resolveRecognition(input: RecognitionInput): Promise<RecognitionOutcome>;
}
```

---

## 4. 결정 1 — 라우팅: **canHandle 기반 (a), 단일 비동기로 통일** ✅

### 문제
`authority_id`(URI)로 adapter를 골라야 한다. `did:web/webvh/webs`는 prefix로 자명하지만, **https URL은 OpenID Federation인지 EUDI Trusted List인지 scheme만으론 구분 불가**.

### 핵심 통찰 — probe는 정당하고, SSRF는 라우팅 이슈가 아니다
- `/.well-known/openid-federation`, `/.well-known/did.json`은 **"이 엔티티가 무슨 프로토콜인지 스스로 밝히는" discovery 장치**다. 이걸 직접 확인(probe)하는 게 스펙 의도에 맞다.
- resolution 자체가 어차피 authority URL을 fetch한다(did.json, entity statement). 즉 **SSRF는 probe가 새로 만드는 게 아니라 resolver의 본질** → 공유 HTTP client(`adapter-kit`)에서 내부 IP 차단·타임아웃·redirect 금지로 **한 번에 처리**(모든 어댑터 공통).

### 어댑터별 "내 것인가" 확인 방법 (비대칭 존재)
| 프로토콜 | 확인 |
|---|---|
| did:web / webvh / webs | scheme+method prefix (I/O 없음) |
| OpenID Federation | `GET /.well-known/openid-federation` 성공? (**probe**) |
| EUDI Trusted List | 로드한 trusted list **멤버십** |
| PKI (x509) | 내 trust anchor로 **체인 성립** |

→ https 충돌의 실체(OIDF vs EUDI)는 확인 방식이 달라 자연히 갈린다.

### 선택: 단일 **비동기** `canHandle` 하나로 통일
sync/async(claim+confirm) 2단계를 두지 않고 **전부 `canHandle(): Promise<boolean>` 하나**로 통일한다. did:*도 I/O 없이 async로 즉시 true/false, OIDF는 well-known GET, EUDI/x509는 데이터 조회 — 호출부는 전부 동일하게 `await`.

```typescript
@TrustAdapter('did:webvh', { order: 10 })   // 값싼 prefix → 앞
class DidWebvhAdapter {
  readonly id = 'did:webvh';
  async canHandle(id: string) { return id.startsWith('did:webvh:'); }   // I/O 없음
}

@TrustAdapter('openid-federation', { order: 100 }) // probe → 뒤
class OidfAdapter {
  readonly id = 'openid-federation';
  async canHandle(id: string) {                                          // well-known probe
    if (!id.startsWith('https://')) return false;
    return (await this.http.head(`${id}/.well-known/openid-federation`)).ok; // 404/parse실패=false, 네트워크에러=throw
  }
}
```

registry는 어댑터를 **order 순으로 앞에서부터 순차 확인**, **첫 support가 곧 결과**다(first-match wins, 결과는 authority당 캐시):

- 매치 발견 → **즉시 그 어댑터로 진행** (뒤는 안 봄 = short-circuit)
- 끝까지 매치 없음 & 에러 없음 → `NoAdapterError` (404)
- 끝까지 매치 없음 & 일부 throw(네트워크) → `RoutingUnavailableError` (판정 불가 → 503)

`AmbiguousRoutingError`(2개+ 매치)는 **불필요** — 확인이 내용/데이터 기반(well-known 실물·리스트 멤버십·체인)이라 서로 다른 프로토콜이 같은 authority를 동시에 주장할 일이 사실상 없고, 혹시 겹쳐도 order가 결정. 덕분에 예외 분기가 하나 줄어든다.

> **order가 load-bearing이 됨**: 값싸게 거르는 것부터 앞에 — did:*(정확한 prefix, I/O 없음)를 먼저, https probe/데이터 조회 어댑터를 뒤에. 각 `canHandle`은 **scheme 안 맞으면 I/O 전에 즉시 `false`**를 리턴해 순차라도 불필요한 네트워크가 안 나가게 한다. 순서는 DiscoveryService 발견 순서에 맡기지 않고 `@TrustAdapter`의 `order`로 명시(§5).

### 이유 / 트레이드오프
- **config 거의 0** — 각 어댑터가 자기 방식(prefix / well-known probe / 리스트 멤버십 / 체인)으로 자가 라우팅. did:*는 무설정.
- 순차 + short-circuit이라 보통 네트워크 0~1회, probe 결과는 `authority→adapter` 캐시(#16)로 authority당 1회.
- 대신 order가 의미를 갖고(문서화 필요), 모호성은 감지가 아니라 order로 해소된다.

---

## 5. 결정 2 — 어댑터 등록: **DiscoveryService + 데코레이터 자동수집 (a) 선택** ✅

### 옵션
- **(a) `DiscoveryService` + `@TrustAdapter('id')` 자동수집**: 어댑터 모듈만 import되면 registry가 자동 인식. ✅ **선택**
- **(b) `AdaptersModule.forRoot([...])` 명시 등록**: 마법 없음, 통제 쉬움. 하지만 어댑터 추가마다 중앙 배열 수정 필요.

### 선택: **(a)** — "어댑터 = 자기완결적 lib 드롭인" 흐름이 good-first-issue 기여(#17~#22)와 정합.

```typescript
// libs/adapter-kit/src/trust-adapter.decorator.ts
export const TRUST_ADAPTER = 'TRUST_ADAPTER';
export const TrustAdapter = (id: string, opts: { order?: number } = {}): ClassDecorator =>
  applyDecorators(Injectable(), SetMetadata(TRUST_ADAPTER, { id, order: opts.order ?? 100 }));
// order 작을수록 먼저 확인. did:* ~10, https probe/데이터 조회 ~100 권장.
```

```typescript
// libs/adapter-kit/src/adapter.registry.ts
@Injectable()
export class AdapterRegistry implements OnModuleInit {
  private adapters: TrustProtocolAdapter[] = []; // order 오름차순 정렬 보관

  constructor(
    private discovery: DiscoveryService,
    private reflector: Reflector,
  ) {}

  onModuleInit() {
    const found: Array<{ order: number; instance: TrustProtocolAdapter }> = [];
    for (const w of this.discovery.getProviders()) {
      const meta = w.metatype && this.reflector.get(TRUST_ADAPTER, w.metatype);
      if (meta) found.push({ order: meta.order, instance: w.instance as TrustProtocolAdapter });
    }
    this.adapters = found.sort((a, b) => a.order - b.order).map(f => f.instance); // order 순 고정
  }

  // order 순 앞에서부터 확인, 첫 support가 결과(first-match). 결과는 authority당 캐시(#16).
  async select(authorityId: string, ctx?: TrqpContext): Promise<TrustProtocolAdapter> {
    let deferredError = false;
    for (const a of this.adapters) {
      try {
        if (await a.canHandle(authorityId, ctx)) return a;   // 첫 support → 즉시 진행 (short-circuit)
      } catch {
        deferredError = true;                                 // 네트워크 등 → 건너뛰고 기억
      }
    }
    if (deferredError) throw new RoutingUnavailableError(authorityId); // → 503
    throw new NoAdapterError(authorityId);                            // → 404
  }
}
```

---

## 6. 나머지 결정 (초안)

### 결정 3 — 두 연산 필수? → **capability 기반** ✅
TRQP는 "authorization/recognition 중 최소 하나"만 요구. 미지원 연산은 `UnsupportedOperationError` throw → 501/400 problem. registry가 "매칭되지만 recognition 미지원"을 구분.

### 결정 4 — 캐시(#16) 위치 → **ResolverService 내부 명시적 `cache.wrap`** ✅
투명 interceptor 대신 명시적. adapter가 준 `freshUntil`을 TTL로, 에러는 negative-cache까지 통제하기 위함. (상세는 #16)

### 결정 5 — 어댑터 패키징 → **지금은 monorepo 내 lib** ✅
별도 npm(`@trs/adapter-*`) 분리는 인터페이스 안정화 후. 지금은 `libs/adapter-*`.

### 공통 안전장치 (거의 확정)
- **공유 contract 테스트** (`adapter-kit/testing/adapter-contract.ts`): 모든 adapter가 통과해야 하는 스위트 → 기여 안전망.
- **`StaticAdapter`**: 로컬/테스트용 + 신규 기여자 복사 템플릿.

---

## 7. 오케스트레이션 흐름

```typescript
// apps/resolver/src/resolver.service.ts
async authorize(q: AuthorizationQueryDto): Promise<AuthorizationResponse> {
  return this.cache.wrap(cacheKey(q), async () => {          // #16
    const adapter = await this.registry.select(q.authorityId, q.context);  // 결정 1+2 (async)
    const outcome = await adapter.resolveAuthorization(toInput(q));
    return toTrqpResponse(q, outcome); // 파라미터 echo + time_evaluated + authorized
  });
}
```

에러는 `AdapterError` 계층(`NoAdapterError`→404, `RoutingUnavailableError`→503, `AuthorityNotFound`, `UpstreamUnavailable`, `UnsupportedOperationError`)으로 던지고, 공통 exception filter가 RFC 7807 problem+json으로 변환(#13).

---

## 8. 모듈 추가 가이드 (draft — "어댑터 하나 추가하기")

> 이 가이드 자체가 아키텍처의 산출물. 최종본은 `libs/adapter-kit/README.md` + 이슈 #14에 수록.

1. **lib 생성**: `nest g library adapter-<name>` → `libs/adapter-<name>` 생성.
2. **어댑터 구현**: `<name>.adapter.ts`에서 `TrustProtocolAdapter` 구현, `@TrustAdapter('<id>')` 데코레이트.
   ```typescript
   @TrustAdapter('did:web')
   export class DidWebAdapter implements TrustProtocolAdapter {
     readonly id = 'did:web';
     async resolveAuthorization(input) { /* did.json fetch → 검증 → 매핑 */ }
     async resolveRecognition(input)   { /* ... */ }
   }
   ```
3. **provider 등록**: lib 모듈의 `providers`에 어댑터 추가.
4. **config 설정**(선택): 트러스트 앵커/엔드포인트가 필요하면 config 네임스페이스 추가.
5. **`canHandle` 구현 + `order` 지정**: `@TrustAdapter('<id>', { order })` — scheme으로 값싸게 거르는 것(did:* ~10)을 앞에, probe/데이터 조회(~100)를 뒤에. did:* 는 `async`지만 `id.startsWith('did:xxx:')` 한 줄, OIDF는 well-known GET, EUDI/x509는 리스트·앵커 조회. `canHandle`은 scheme 안 맞으면 I/O 전에 즉시 `false`. (404/parse실패=`false`, 네트워크에러=throw)
6. **resolver 앱에 lib import**: `apps/resolver`의 모듈에 lib 모듈 import 한 줄 → `DiscoveryService`가 자동 등록.
7. **contract 테스트 통과**: `runAdapterContract(() => new XAdapter())` 로 공유 스위트 실행.

→ 즉 **코어 코드 수정 없이 lib 추가 + config 한 줄 + import 한 줄**.

---

## 9. 결정 로그 (요약)

| # | 결정 | 선택 | 탈락 대안 | 이유 |
|---|------|------|-----------|------|
| 1 | 라우팅 | **canHandle 단일 비동기 (a), order 순 first-match** | (b) 중앙 config 맵 · claim+confirm 2단계 · 병렬+모호성감지 | 자가 라우팅(config 거의 0) + 순차 first-match로 예외 분기 최소화(ambiguous 제거) |
| 2 | 어댑터 등록 | **DiscoveryService + @TrustAdapter (a)** | (b) forRoot 명시 배열 | 자기완결 lib 드롭인, 기여 친화 |
| 3 | 두 연산 필수? | **capability 기반 (미지원 throw)** | 둘 다 강제 | TRQP "최소 하나" 부합 |
| 4 | 캐시 위치 | **ResolverService 명시적 wrap** | 투명 interceptor | freshUntil TTL·negative cache 통제 |
| 5 | 패키징 | **monorepo lib** | 별도 npm 패키지 | 지금은 단순, 나중 분리 가능 |
| 6 | 다중 프로토콜 | **1:1 강제 (first-match 단일 adapter)** | fallback/합의 정책 | 결정적·단순, order가 우선순위. 합의는 연기 |
| 7 | 타입 공유(O-1) | **pnpm workspace `packages/trqp-core` (zod SSOT)** | nest lib 내부, 각자 중복 | client가 nest 밖이라 lib 불가. client는 tsup로 인라인 |
| 8 | evidence(O-5) | **로그 only** | 응답에 확장 필드 노출 | TRQP 표준 응답 형태 유지 |

---

## 10. 열린 질문 (계속 논의)

- **O-1**: (결정) **pnpm workspace + `packages/trqp-core`** (zod SSOT, §2). 서버·클라가 `workspace:*`로 공유, client만 tsup `noExternal`로 인라인해 자기완결 배포.
- **O-2**: (해소됨) 라우팅 = 단일 비동기 `canHandle`. 별도 `supports()`/중앙 config 라우팅 테이블 없음.
- **O-3**: (보류) `canHandle` throw 시 라우팅 캐시 정책 — 실패는 캐시 제외? 짧은 negative TTL? #16 구현 때 정한다.
- **O-4**: (결정) **1:1 강제** — authority당 정확히 하나의 adapter(order 순 first-match)가 해석. cross-protocol fallback/합의는 v1 밖(필요 시 나중에 정책 레이어). first-match 라우팅과 자연히 정합.
- **O-5**: (결정) **로그 only** — `evidence`는 TRQP 표준 응답에 안 넣고 감사 로그로만. (필요 시 나중에 확장 필드)
