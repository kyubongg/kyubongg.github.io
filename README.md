# kyubongg.github.io

[Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 테마 기반 기술 블로그 → https://kyubongg.github.io

## 배포

`main`에 push하면 GitHub Actions(`.github/workflows/pages-deploy.yml`)가 빌드 후 자동 배포합니다.
저장소 Settings → Pages → Source는 **GitHub Actions**로 설정되어 있어야 합니다.

## 새 글 쓰기

`_posts/YYYY-MM-DD-slug.md` 파일을 만들고 아래 front matter를 붙입니다.

```yaml
---
title: "글 제목"
date: 2026-10-06 21:00:00 +0900
categories: [CS, 운영체제]   # 최대 2단계 (상위, 하위)
tags: [process, thread]      # 소문자 권장
---
```

- 목차는 `##`, `###` 헤딩으로 자동 생성됩니다.
- 다이어그램을 쓰려면 front matter에 `mermaid: true`, 수식은 `math: true`를 추가합니다.

## 로컬 미리보기 (선택)

Ruby 3.x 설치 후:

```bash
bundle install
bundle exec jekyll serve
```
