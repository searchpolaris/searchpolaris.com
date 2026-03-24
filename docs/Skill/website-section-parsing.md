---
name: website-section-parsing
description: Use when the user wants to clone, replicate, or extract a specific section from a reference website. Also use when the user says "bring this section", "make it like this site", "parse this page", or wants to recreate a web component with identical structure and animation.
---

# Website Section Parsing (Playwright 다단계 추출)

## Overview
레퍼런스 웹사이트의 특정 섹션을 Playwright로 정밀 파싱하여, HTML + CSS + JS + 에셋을 완벽히 추출한 뒤 독립 HTML 파일로 재현하는 기법.

## When to Use
- 레퍼런스 사이트의 특정 섹션을 동일하게 가져오고 싶을 때
- 애니메이션, 인터랙션 포함된 컴포넌트를 클론할 때
- WebFetch로는 정확한 코드를 얻기 어려울 때

## When NOT to Use
- 단순 텍스트/이미지만 가져올 때 (WebFetch로 충분)
- 스크린샷만 필요할 때

## 추출 방법 비교

| 방법 | HTML | CSS 변수 | JS | 에셋 | 정확도 |
|------|------|----------|-----|------|--------|
| WebFetch | 요약됨 | X | X | X | 30% |
| Playwright outerHTML만 | 100% | X | X | X | 50% |
| **Playwright 다단계** | **100%** | **100%** | **100%** | **100%** | **95%+** |

## Core Pattern: 5단계 추출

### Step 1: HTML 구조 추출
```javascript
browser_evaluate → () => {
  const el = document.querySelector('.target-section');
  return el.outerHTML;
}
```
- `outerHTML`로 정확한 DOM 구조 확보
- 클래스명, 중첩 구조, data 속성 모두 보존

### Step 2: CSS 규칙 추출 (중요: getComputedStyle 쓰지 말 것)
```javascript
browser_evaluate → () => {
  // BAD: getComputedStyle → CSS 변수가 풀려버림 (var(--color) → rgb(0,0,0))
  // GOOD: document.styleSheets → 원본 CSS 규칙 유지
  const rules = [];
  for (const sheet of document.styleSheets) {
    try {
      for (const rule of sheet.cssRules) {
        if (rule.cssText.includes('target-class')) rules.push(rule.cssText);
      }
    } catch(e) { /* cross-origin 무시 */ }
  }
  return JSON.stringify(rules);
}
```
- `document.styleSheets` 사용 → CSS 변수명, 미디어쿼리 원본 유지
- `getComputedStyle`은 레이아웃 수치 확인용으로만 사용 (디버깅)
- CSS가 너무 많으면 타겟 클래스명으로 필터링

### Step 3: JavaScript 소스 추출
```javascript
browser_evaluate → () => {
  // 방법 A: 글로벌 클래스가 있을 때
  if (typeof TargetClass !== 'undefined') return TargetClass.toString();

  // 방법 B: 인라인 스크립트에서 찾기
  const scripts = document.querySelectorAll('script');
  for (const s of scripts) {
    if (s.innerHTML.includes('TargetKeyword')) return s.innerHTML;
  }

  // 방법 C: 헬퍼 함수까지 함께 추출
  let result = '';
  if (typeof helperFn !== 'undefined') result += helperFn.toString();
  return result;
}
```
- 먼저 글로벌 스코프에서 클래스/함수 검색
- 없으면 `<script>` 태그 innerHTML에서 키워드 검색
- 번들된 JS는 Network 탭 URL → WebFetch로 다운로드

### Step 4: 에셋(이미지/SVG) 로컬 저장
```javascript
browser_evaluate → () => {
  const imgs = document.querySelectorAll('.target-section img');
  return Array.from(imgs).map(i => ({ src: i.src, alt: i.alt }));
}
```
```bash
# 추출된 URL로 curl 다운로드 → 로컬에 저장
curl -sO "https://cdn.example.com/icon.svg"
# 파일명을 의미있는 이름으로 변경
mv "hash_icon.svg" "chatgpt.svg"
```
- CDN URL은 시간이 지나면 깨질 수 있으므로 반드시 로컬 저장
- SVG 파일명은 알아보기 쉽게 rename

### Step 5: 스크린샷 레퍼런스 캡처
```javascript
browser_evaluate → () => {
  document.querySelector('.target').scrollIntoView({ block: 'center' });
}
browser_take_screenshot → 원본 레퍼런스 저장
```
- 구현 전 원본 캡처 → 구현 후 결과와 비교 검증
- fullPage: true로 전체 캡처 가능

## 재현 워크플로우

```
1. browser_navigate → 사이트 접속
2. browser_evaluate → outerHTML 추출 (HTML)
3. browser_evaluate → styleSheets 규칙 추출 (CSS)
4. browser_evaluate → JS 클래스/함수 추출 (JS)
5. browser_evaluate → 이미지 URL 추출 → curl로 로컬 저장 (에셋)
6. browser_take_screenshot → 시각 레퍼런스 캡처
7. Write → 추출된 코드를 새 HTML 파일로 조합
8. browser_navigate → Live Server에서 결과 확인
9. browser_take_screenshot → 원본과 비교 검증
```

## 커스텀 단계 (클론 후)

원본을 정확히 재현한 뒤 수정:
1. **아이콘/이미지 교체** — `<img src>` 경로만 변경
2. **색상 테마 변환** — CSS 변수/색상값 일괄 치환
3. **텍스트 교체** — HTML 텍스트 노드 수정
4. **레이아웃 미세 조정** — 패딩, 마진, 그리드 비율

## Common Mistakes

| 실수 | 해결 |
|------|------|
| `getComputedStyle`로 CSS 추출 → 변수명 소실 | `document.styleSheets`로 원본 규칙 추출 |
| CDN 이미지 URL 그대로 사용 → 나중에 깨짐 | 반드시 로컬 다운로드 후 상대경로 사용 |
| CSS 전체 추출 → 결과가 너무 방대함 | 타겟 클래스명으로 필터링 |
| JS 번들에서 찾기 실패 | 글로벌 스코프 → 인라인 스크립트 → 번들 URL 순서로 시도 |
| Playwright 브라우저 충돌 | Chrome 완전 종료 후 재시도 |
| WebFetch로 코드 추출 시도 | WebFetch는 요약 모델 거침 → 코드 정확도 낮음, Playwright 사용 |

## 제한사항

- **CSS-in-JS** (styled-components 등): 런타임 스타일이라 styleSheets에 안 잡힐 수 있음 → getComputedStyle 병행
- **Shadow DOM**: 일반 querySelector로 접근 불가 → `element.shadowRoot` 사용
- **Webflow/빌더 사이트**: 불필요한 래퍼/클래스가 많음 → 추출 후 정리 필요
- **인증 필요 페이지**: Playwright로 로그인 후 추출하거나, 쿠키 설정 필요
