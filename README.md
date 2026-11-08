# Документация ZenMoney Telegram Bot

Исходники сайта https://rusnino.github.io/zenmoney-telegram-bot-docs/ на [Zensical](https://zensical.org/).

## Сборка локально

```bash
uvx zensical@0.1.0 build --clean --strict   # без ошибок, битых ссылок и якорей
uvx zensical@0.1.0 serve                     # просмотр на http://localhost:8000
```

Публикация: `.github/workflows/docs.yml` на push в `main` собирает `site/` и выкладывает его через `actions/deploy-pages`. В настройках репозитория Pages → Source: «GitHub Actions».
