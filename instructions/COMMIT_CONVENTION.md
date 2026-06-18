# 커밋 컨벤션 (COMMIT_CONVENTION)

> Conventional Commits 1.0.0 표준 기반 + commitlint 자동 검증

## 메시지 형식

```
<type>(<scope>): <subject>

[body]

[footer]
```

### 구성 요소

| 항목      | 필수 | 규칙                                                         | 예시                                    |
| --------- | ---- | ------------------------------------------------------------ | --------------------------------------- |
| `type`    | O    | 소문자 또는 대문자(`HOTFIX`) 정확히 일치                     | `feat`, `fix`, `HOTFIX`                 |
| `scope`   | X    | 변경 범위 (패키지·모듈·컴포넌트명) — 괄호 포함               | `feat(auth)`, `fix(api)`                |
| `subject` | O    | 50자 이내, 명령형(영어), 마침표·대문자 시작 금지             | `add OAuth2 login`, `handle null input` |
| `body`    | X    | 상세 설명 (72자 줄 바꿈), 무엇을 & 왜 했는지                 | 다중 줄 가능                            |
| `footer`  | X    | Breaking Change 또는 이슈 참조 (`Closes #123`, `Fixes #456`) | `BREAKING CHANGE: ...`                  |

### 규칙 상세

**type 헤더 길이**: 최대 100자 (한국어 커밋 메시지도 지원)
**subject 케이스**: 강제하지 않음 (영어·한국어·혼합 모두 가능)
**body 앞 빈 줄**: 필수 (있으면 더 좋음)

## Type 목록 및 규칙

commitlint `@commitlint/config-conventional` 기반 + 프로젝트 커스텀 확장

| Type       | 설명                           | 언제 사용                                  | 예시                                  |
| ---------- | ------------------------------ | ------------------------------------------ | ------------------------------------- |
| `feat`     | 새 기능 추가                   | 비즈니스 로직·엔드포인트·컴포넌트 신규     | `feat(auth): add refresh token`       |
| `fix`      | 버그 수정                      | 기존 기능의 결함 해결                      | `fix(api): handle null response`      |
| `refactor` | 기능 변경 없는 코드 개선       | 로직 단순화·추상화·중복 제거 (스타일 제외) | `refactor(db): extract query builder` |
| `style`    | 포맷·공백·세미콜론 (의미 무변) | 자동 포매터·linter 적용 결과               | `style: fix indentation`              |
| `design`   | UI/UX 개선 (기능은 동일)       | 디자인·레이아웃·사용성 개선                | `design(form): adjust button spacing` |
| `perf`     | 성능 최적화                    | 속도·메모리·DB 쿼리 개선                   | `perf(query): add index on user_id`   |
| `docs`     | 문서만 수정                    | README·주석·API 문서 추가/수정             | `docs(readme): update install guide`  |
| `comment`  | 코드 주석만 수정               | 주석 추가·개선 (코드 변경 없음)            | `comment: clarify cache logic`        |
| `chore`    | 빌드·패키지·설정 변경          | 의존성·DevOps·설정 파일 (테스트 제외)      | `chore(deps): bump react to 19`       |
| `build`    | 빌드 시스템·의존성 변경        | 번들러·컴파일러·빌드 설정 변경             | `build: migrate to vite`              |
| `init`     | 프로젝트 초기화                | 저장소 초기 세팅·스캐폴딩                  | `init: setup project skeleton`        |
| `revert`   | 이전 커밋 되돌리기             | 커밋 전체 취소                             | `revert: revert "feat: add SSO"`      |
| `etc`      | 기타                           | 위 범주에 맞지 않는 변경                   | `etc: update license`                 |
| `HOTFIX`   | 긴급 버그 수정 (대문자)        | 프로덕션 장애 즉시 해결                    | `HOTFIX: patch critical security bug` |

## Breaking Change

기존 API·기능을 삭제하거나 호환되지 않게 변경할 때 명시:

```
feat(api)!: remove deprecated v1 endpoints

BREAKING CHANGE: /api/v1/* endpoints removed. Use /api/v2/* instead.
Closes #456
```

또는 footer로:

```
fix(auth): change token format

BREAKING CHANGE: refresh tokens now expire after 7 days instead of 30
```

## commitlint 검증

### 설정 파일

- **위치**: `commitlint.config.js` (또는 `.commitlintrc.json`, `.commitlintrc.yaml`)
- **기본값**: `@commitlint/config-conventional` 확장
- **검증 레벨**:
  - `2` = Error (커밋 차단)
  - `1` = Warning (경고만)
  - `0` = Disabled (비활성화)

### 훅 설정

```bash
# 설치 (한 번만)
npx husky install

# 자동 실행
# .husky/commit-msg 훅이 매 커밋 전 commitlint 실행
```

### 검증 실패 예시

```
❯ commitlint -e

husky - commitlint

⧖  input: "fix stuff"
✖  (type-enum) type "fix stuff" is not one of [feat, fix, refactor, style, design, perf, docs, comment, chore, build, init, revert, etc, HOTFIX]

Error: commit message is invalid
(at commit-msg)
```

**해결방법**: type을 명시된 목록 중 하나로 변경하고 커밋 메시지 형식 준수.

## 좋은 커밋 메시지 예시

### ✅ Good: 기능 추가

```
feat(auth): add refresh token rotation

Implement RFC 6749 token rotation to reduce exposure window.
Refresh tokens are now single-use and rotate on each use.

BREAKING CHANGE: existing refresh tokens are invalidated on next login
Closes #234
```

### ✅ Good: 버그 수정

```
fix(api): handle null response in payment gateway

Previously, null responses from the payment gateway were not caught,
causing unhandled promise rejection. Now wrapped in try-catch with
fallback to 503 Service Unavailable.

Fixes #789
```

### ✅ Good: 리팩토링

```
refactor(db): extract connection pool builder

Create standalone QueryBuilder class to reduce Database class size
and improve testability. No functional change.
```

### ✅ Good: 성능 개선

```
perf(query): add index on user_id in orders table

Reduced query time from 500ms to 50ms for high-traffic endpoints.
```

### ❌ Bad: Type 누락

```
add new feature
Update code
fixed stuff
```

### ❌ Bad: 형식 위반

```
FEAT(CORE): Add New Feature

FIX BUG
feat: 🎉 Super awesome feature (이모지 사용 금지)
```

### ❌ Bad: 불명확한 scope

```
feat(utils): do something
fix: stuff
```

## 체크리스트

커밋 전 확인:

- [ ] `<type>` 목록에서 정확히 선택했는가
- [ ] type은 소문자인가 (단, `HOTFIX` 제외)
- [ ] subject는 50자 이내 명령형(영어)인가
- [ ] subject가 마침표로 끝나지 않는가
- [ ] body가 있으면 subject와 1줄 띄었는가
- [ ] Breaking Change가 있으면 footer에 명시했는가
- [ ] 이슈 번호 있으면 `Closes #123` 또는 `Fixes #456` 형식으로 참조했는가

## 참고

- **Conventional Commits 1.0.0**: https://www.conventionalcommits.org
- **commitlint 공식 문서**: https://commitlint.js.org
- **이 저장소 설정**: `commitlint.config.js`

---

**최종 검증**: `npm run commitlint -- --from=<base-branch> --to=HEAD` 또는 수동 commitlint 테스트

```bash
echo "feat(test): verify commit format" | commitlint
```
