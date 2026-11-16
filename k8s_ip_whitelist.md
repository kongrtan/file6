# k8s IP 허용 목록(whitelist) 관리 — Contour

Kubernetes에서 도메인·경로별로 특정 IP만 접속할 수 있게 하는 방법을 정리한 문서입니다. 인그레스 컨트롤러는 Envoy 기반 Contour를 기준으로 합니다.

> 이 문서의 IP 주소(`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`)와 도메인(`example.com`)은 문서용 예시 값입니다. 실제 환경에 맞게 바꿔서 쓰세요.
>
> `ipAllowPolicy`, `regex`/`exact` 경로 조건, 와일드카드 fqdn은 비교적 최근 버전의 Contour에 들어간 기능입니다. 적용 전에 [6. 적용 전 확인](#6-적용-전-확인)으로 사용 중인 버전에서 지원하는지 확인합니다.

---

## 1. 요청 경로와 전제

이 문서는 클러스터 앞에 리버스 프록시(이하 "앞단 프록시")가 하나 있는 구성을 가정합니다.

```
클라이언트 ─▶ 앞단 프록시 (nginx 등, 실제 클라이언트 IP를 앎)
            ─▶ Envoy (Contour, NodePort 또는 LoadBalancer)
            ─▶ 서비스
```

- Envoy에 직접 연결해 오는 것은 항상 앞단 프록시입니다. 그래서 Envoy가 보는 접속 IP(`Peer`)로 판단하면 모든 요청이 앞단 프록시 IP로 보입니다.
- 실제 클라이언트 IP는 앞단 프록시가 붙여 주는 `X-Forwarded-For`(XFF) 헤더에만 있습니다. 허용 목록은 **`source: Remote`**(XFF에서 꺼낸 IP)로 걸어야 합니다.
- 앞단 프록시 없이 클라이언트가 Envoy에 바로 붙는 구성이라면, 실제 클라이언트 IP가 보존되는지 확인한 뒤(`externalTrafficPolicy: Local` 등) `source: Peer`를 쓰면 됩니다.

### 반드시 갖춰야 할 세 가지

| # | 항목 | 설정 | 빠지면 |
|---|---|---|---|
| 1 | 앞단 프록시가 XFF 전달 | nginx라면 `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;` | `Remote`가 프록시 IP가 되어 모두 차단되거나 모두 통과 |
| 2 | Contour가 프록시 한 단계를 신뢰 | `num-trusted-hops: 1` ([3장](#3-contour-신뢰-프록시-설정)) | XFF를 무시하거나, 위조된 값을 믿음 |
| 3 | Envoy 직접 접속 차단 | 외부에서는 앞단 프록시를 거쳐서만 들어오게 | **XFF를 위조해 허용 목록을 우회 가능** |

3번이 가장 중요합니다. 외부에서 Envoy의 NodePort나 LoadBalancer 주소로 직접 붙을 수 있으면, 누구나 `X-Forwarded-For: <허용된 IP>`를 넣어 허용 IP로 위장할 수 있습니다. 방화벽이나 보안 그룹으로 Envoy 쪽 포트는 앞단 프록시와 내부망에서만 접근할 수 있게 막아 둡니다.

---

## 2. 어떤 리소스로 걸 수 있나

| 리소스 | IP 허용 목록 | 비고 |
|---|---|---|
| **HTTPProxy** (Contour 전용) | ✅ `ipAllowPolicy` / `ipDenyPolicy` | 도메인 전체 또는 경로별로 가능 |
| **HTTPRoute** (Gateway API 표준) | ❌ 불가 | 표준에 출처 IP 필드가 없고, Contour도 HTTPRoute에 붙이는 IP 정책을 제공하지 않음 |

**허용 목록이 필요한 도메인은 HTTPProxy로 작성합니다.** 나머지 도메인은 HTTPRoute로 두어도 됩니다. 단, **같은 도메인을 HTTPProxy와 HTTPRoute에 나눠 정의하면 충돌하므로** 도메인 단위로 나눕니다.

Gateway API(HTTPRoute)를 계속 쓰면서 IP 제한이 필요하다면, 같은 Envoy 기반인 Envoy Gateway의 `SecurityPolicy`(`clientCIDRs`)를 검토할 수 있습니다.

---

## 3. Contour 신뢰 프록시 설정

Contour 설정 파일(ConfigMap `contour`의 `contour.yaml`)에 넣습니다.

```yaml
network:
  num-trusted-hops: 1     # 앞단 프록시 한 단계만 신뢰
```

ContourConfiguration CRD를 쓰는 경우:

```yaml
spec:
  envoy:
    network:
      numTrustedHops: 1
```

- 앞단 프록시와 Envoy 사이에 프록시가 더 있으면 그 수만큼 늘립니다. 예를 들어 프록시를 두 번 거치면 `2`입니다.
- nginx에서 `$proxy_add_x_forwarded_for`를 쓰면 클라이언트가 보낸 XFF 뒤에 실제 IP가 덧붙습니다. trusted hop을 1로 두면 Envoy는 앞단 프록시가 붙인 마지막 값을 클라이언트 IP로 씁니다. 그래서 클라이언트가 XFF 앞부분을 위조해도 무시됩니다.
- 설정을 바꾼 뒤 Contour와 Envoy를 재시작해야 반영되는지 확인합니다.

---

## 4. HTTPProxy 작성 방법

예시에서 쓰는 값:

| 이름 | 예시 값 |
|---|---|
| 사무실 | `198.51.100.10/32` |
| 집 | `203.0.113.20/32` |
| 차단할 대역 | `192.0.2.0/24` |
| 도메인 | `app.example.com` |

### 4-1. 도메인 전체 제한

```yaml
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: app
spec:
  virtualhost:
    fqdn: app.example.com
    ipAllowPolicy:                # 하나라도 맞으면 허용, 나머지는 403
      - cidr: 198.51.100.10/32    # 사무실
        source: Remote
      - cidr: 203.0.113.20/32     # 집
        source: Remote
  routes:
    - services:
        - name: app
          port: 80
```

### 4-2. 경로별 제한

```yaml
spec:
  virtualhost:
    fqdn: app.example.com
  routes:
    # /admin: 사무실만
    - conditions:
        - prefix: /admin
      ipAllowPolicy:
        - cidr: 198.51.100.10/32
          source: Remote
      services:
        - name: app
          port: 80

    # /api: 사무실 + 집
    - conditions:
        - prefix: /api
      ipAllowPolicy:
        - cidr: 198.51.100.10/32
          source: Remote
        - cidr: 203.0.113.20/32
          source: Remote
      services:
        - name: app-api
          port: 8080

    # 나머지: 정책 없음 → 누구나 접근
    - conditions:
        - prefix: /
      services:
        - name: app
          port: 80
```

- 경로에 정책이 있으면 그 경로의 정책이 도메인 전체 정책보다 우선합니다. 사용 중인 버전의 문서에서 동작을 한 번 확인하세요.
- prefix는 가장 길게 일치하는 것이 선택됩니다. `/admin` prefix는 `/administrator`에도 걸리니 주의합니다. 정확히 그 경로만 원하면 `exact`를 씁니다.

### 4-3. 패턴(정규식)·정확 일치

```yaml
  routes:
    # /api/v1/admin, /api/v2/admin ... 하위 경로 포함
    - conditions:
        - regex: /api/v[0-9]+/admin(/.*)?
      ipAllowPolicy:
        - cidr: 198.51.100.10/32
          source: Remote
      services:
        - name: app-api
          port: 8080

    # 정확히 /login 하나만
    - conditions:
        - exact: /login
      ipAllowPolicy:
        - cidr: 198.51.100.10/32
          source: Remote
      services:
        - name: app
          port: 80
```

- 정규식은 RE2 문법이고 **경로 전체가 일치해야** 합니다. 하위 경로까지 포함하려면 끝에 `(/.*)?`를 붙입니다.
- `regex`와 `prefix`가 같은 요청에 겹쳐 걸리면 우선순위가 버전마다 다를 수 있습니다. 겹치지 않게 작성하고, 적용 후 [7장](#7-적용-후-검증)처럼 직접 확인합니다.

### 4-4. 와일드카드 도메인

```yaml
spec:
  virtualhost:
    fqdn: "*.example.com"
    ipAllowPolicy:
      - cidr: 198.51.100.10/32
        source: Remote
```

하위 도메인 전체에 같은 목록이 적용됩니다. 특정 하위 도메인에 별도 HTTPProxy가 있으면 그쪽이 우선합니다.

### 4-5. 특정 IP만 차단

```yaml
    ipDenyPolicy:
      - cidr: 192.0.2.0/24
        source: Remote
```

---

## 5. 목록을 한 곳에서 관리하기 (Kustomize)

Contour에는 Traefik 미들웨어처럼 여러 도메인이 함께 쓰는 허용 목록 리소스가 없습니다. 같은 목록을 여러 HTTPProxy에 넣어야 하면 Kustomize component로 묶어 **한 파일만 고치게** 만듭니다.

```
k8s/
├── components/
│   └── allow-office-home/
│       └── kustomization.yaml     ← 허용 IP는 여기만 수정
└── apps/
    ├── app/
    │   ├── httpproxy.yaml
    │   └── kustomization.yaml     ← components로 위 목록을 포함
    └── other-app/ ...
```

`components/allow-office-home/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
patches:
  - target:
      kind: HTTPProxy
      labelSelector: "ip-allow=office-home"
    patch: |-
      - op: add
        path: /spec/virtualhost/ipAllowPolicy
        value:
          - cidr: 198.51.100.10/32    # 사무실
            source: Remote
          - cidr: 203.0.113.20/32     # 집
            source: Remote
```

`apps/app/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - httpproxy.yaml
components:
  - ../../components/allow-office-home
```

- 허용 목록을 적용할 HTTPProxy에 `metadata.labels.ip-allow: office-home` 라벨을 붙입니다.
- git에 두고 Argo CD나 Flux로 반영하면 언제 누가 IP를 추가했는지 이력이 남습니다.

---

## 6. 적용 전 확인

- [ ] Contour 버전: `ipAllowPolicy`, `regex`/`exact`, 와일드카드 fqdn을 지원하는지
  ```bash
  kubectl -n projectcontour get deploy contour -o jsonpath='{.spec.template.spec.containers[0].image}'
  kubectl explain httpproxy.spec.virtualhost.ipAllowPolicy
  kubectl explain httpproxy.spec.routes.conditions
  ```
- [ ] 현재 도메인별 리소스 종류(HTTPProxy인지 HTTPRoute인지)
  ```bash
  kubectl get httpproxy -A
  kubectl get httproute -A
  ```
- [ ] 앞단 프록시가 `X-Forwarded-For`를 전달하는지
- [ ] 앞단 프록시와 Envoy 사이 프록시 단계 수(`num-trusted-hops` 값 결정)
- [ ] 외부에서 Envoy(NodePort·LoadBalancer)로 직접 접속할 수 없는지

---

## 7. 적용 후 검증

| 시험 | 방법 | 기대 결과 |
|---|---|---|
| 허용 IP | 허용된 곳에서 `curl -I https://app.example.com/admin` | 200 (또는 서비스 원래 응답) |
| 비허용 IP | 휴대폰 LTE 등 다른 IP에서 같은 주소 | **403** |
| 정책 없는 경로 | 비허용 IP에서 `curl -I https://app.example.com/` | 200 |
| XFF 위조 | 비허용 IP에서 `curl -I -H "X-Forwarded-For: 198.51.100.10" https://app.example.com/admin` | **403** (위조가 무시돼야 함) |
| 직접 접속 | 외부에서 Envoy NodePort·LoadBalancer 주소로 | 연결 불가 |

XFF 위조 시험에서 200이 나오면 `num-trusted-hops`나 앞단 프록시 설정이 잘못된 것입니다. 바로 되돌립니다.

---

## 8. 운영

### IP 추가·삭제
1. 공용 목록이면 `components/allow-office-home/kustomization.yaml`, 개별 목록이면 해당 HTTPProxy를 수정합니다.
2. `kubectl kustomize apps/app`으로 결과 YAML을 확인합니다.
3. 적용(GitOps면 커밋·푸시) 후 7장의 허용·비허용 시험을 합니다.

### 이동 중 접속
휴대폰 IP는 계속 바뀝니다. 이동 중에 관리 경로가 필요하면 쓰는 통신사의 모바일 대역을 넣는 방법이 있습니다. 대역은 KISA WHOIS 등에서 소유 기관을 확인해 고릅니다. 대신 같은 통신사 사용자 전체에게 열린다는 점을 감안합니다. VPN을 거쳐 고정 IP로 나가게 하는 쪽이 더 안전합니다.

### 대안: 앞단 프록시에서 걸기
k8s 설정을 건드리지 않고 앞단 nginx의 vhost에 `allow`/`deny`를 넣는 방법도 있습니다. 앞단 프록시는 실제 IP를 알고 있어서 신뢰 프록시 설정이 필요 없습니다. 도메인 몇 개만 막을 거라면 이쪽이 더 단순합니다.

```nginx
location ^~ /admin {
    allow 198.51.100.10;
    allow 203.0.113.20;
    deny  all;
    proxy_pass ...;   # 기존 vhost의 location / 과 같은 upstream
}
```

---

## 참고

- Contour 문서: HTTPProxy `ipAllowPolicy` / `ipDenyPolicy`, `num-trusted-hops`
- 문서용 예시 IP 대역: RFC 5737 (`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`)
