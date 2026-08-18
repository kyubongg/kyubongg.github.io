# 기술 블로그 (Jekyll + GitHub Pages)

## 1. 저장소에 올리기

1. GitHub에서 새 저장소를 만듭니다.
   - **`kyubongg.github.io`**라는 이름으로 만들면 별도 설정 없이 `https://kyubongg.github.io`로 바로 서비스됩니다. (권장)
   - 다른 이름(`my-blog` 등)으로 만들면 주소가 `https://kyubongg.github.io/my-blog`가 되고, `_config.yml`의 `baseurl`을 `"/my-blog"`로 바꿔야 합니다.
2. 이 폴더 내용을 그 저장소에 push 합니다.

```bash
cd tech-blog
git init
git add .
git commit -m "Initial blog scaffold"
git branch -M main
git remote add origin https://github.com/kyubongbong/kyubongbong.github.io.git
git push -u origin main
```

## 2. GitHub Pages 활성화

저장소 → **Settings → Pages** → Source를 **Deploy from a branch**로 두고, 브랜치는 `main` / 폴더는 `/ (root)`로 선택 후 저장합니다. 몇 분 후 사이트가 배포됩니다.

## 3. 로컬에서 미리보기 (선택)

```bash
bundle install
bundle exec jekyll serve
```

브라우저에서 `http://localhost:4000` 접속.

## 4. 새 글 쓰기

`_posts/` 폴더에 `YYYY-MM-DD-제목.md` 형식으로 파일을 추가하면 자동으로 글 목록에 나타납니다.

## 5. 수정해야 할 것 (TODO)

- `_config.yml`의 `title`, `url`, `github_username`
- `about.md`의 자기소개 내용
