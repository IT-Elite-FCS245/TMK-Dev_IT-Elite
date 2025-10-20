## Гайд по Markdown-линтеру

Этот документ объясняет использование и настройку `markdown-lint.yml` — рабочего процесса GitHub Actions, который автоматически проверяет и форматирует все Markdown файлы в вашем репозитории. Это помогает поддерживать единый стиль и высокое качество документации.

### 1. Назначение `markdown-lint.yml`

Файл `https://github.com/IT-Elite-FCS245/TMK-Dev_IT-Elite/blob/main/.github/workflows/markdown-lint.yml` предназначен для автоматической проверки Markdown файлов на соответствие определенным правилам стиля и форматирования. Он выполняет следующие функции:

*   **Проверка стилей:** Использует `markdownlint-cli` для выявления нарушений стилей (например, лишние пробелы, неправильные заголовки, неверное использование списков).
*   **Автоматическое исправление:** Включает опцию `--fix`, которая автоматически исправляет многие распространенные проблемы форматирования.
*   **CI/CD интеграция:** Запускается при каждом `pull_request` и `push` в ветку `main`, обеспечивая, что все изменения в Markdown файлах проходят проверку перед слиянием.
*   **Демонстрация CI/CD:** Выполнение этого рабочего процесса является частью требований к домашнему заданию, демонстрируя использование CI/CD для обеспечения качества документации.

### 2. Структура `markdown-lint.yml`

```yaml
name: Markdown Lint and Format
on:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install markdownlint-cli
        run: npm install -g markdownlint-cli

      - name: Run markdownlint and fix
        run: markdownlint --fix "**/*.md"

      - name: Check for changes after linting
        id: git_diff
        run: |
          git config user.name github-actions
          git config user.email github-actions@github.com
          git add .
          git diff --staged --quiet || (
            echo "Markdown formatting issues detected and fixed. Committing changes..."
            git commit -m "fix: Apply markdown formatting fixes [skip ci]"
            git push
            echo "::set-output name=has_changes::true"
          )
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Fail if changes were committed
        if: steps.git_diff.outputs.has_changes == 'true'
        run: |
          echo "A new commit with formatting fixes was pushed. Please re-run the PR checks."
          exit 1

  deploy-pages:
    needs: lint
    runs-on: ubuntu-latest
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Pages
        uses: actions/configure-pages@v4
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: './docs'
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### 3. Как это работает

1.  **`on: pull_request` и `on: push`**: Рабочий процесс запускается при создании или обновлении Pull Request, а также при прямом `push` в ветку `main`.
2.  **`lint` job**: Этот `job` отвечает за проверку и исправление Markdown файлов.
    *   `Checkout code`: Получает код из репозитория.
    *   `Setup Node.js`: Устанавливает Node.js, необходимый для `markdownlint-cli`.
    *   `Install markdownlint-cli`: Устанавливает саму утилиту `markdownlint-cli`.
    *   `Run markdownlint and fix`: Запускает линтер с опцией `--fix` для автоматического исправления. Это важно, так как позволяет автоматически корректировать мелкие ошибки форматирования.
    *   `Check for changes after linting`: После попытки исправления, этот шаг проверяет, были ли внесены изменения. Если `markdownlint --fix` что-то исправил, то создается новый коммит с этими исправлениями и он пушится обратно в ветку.
    *   `Fail if changes were committed`: Если были внесены автоматические исправления и создан новый коммит, то этот шаг приводит к сбою рабочего процесса. Это сделано намеренно, чтобы убедиться, что автор PR видит, что были внесены изменения, и должен перезапустить проверку. Это также гарантирует, что все изменения проходят через процесс PR и ревью.
3.  **`deploy-pages` job**: Этот `job` отвечает за развертывание документации на GitHub Pages.
    *   `needs: lint`: Гарантирует, что развертывание произойдет только после успешного завершения `lint` job. Это означает, что на GitHub Pages попадет только отформатированная и проверенная документация.
    *   `permissions` и `environment`: Настраивают необходимые разрешения и среду для корректной работы с GitHub Pages.
    *   `Upload artifact` и `Deploy to GitHub Pages`: Эти шаги используют официальные действия GitHub для загрузки содержимого папки `./docs` как артефакта и последующего развертывания его на GitHub Pages.

### 4. Рекомендации по использованию

*   **Всегда проверяйте статус CI/CD**: Перед слиянием Pull Request убедитесь, что все проверки (включая `lint` и `deploy-pages`) успешно завершены.
*   **Локальная проверка**: Рекомендуется запускать `markdownlint --fix "**/*.md"` локально перед созданием коммитов, чтобы избежать лишних циклов CI/CD и автоматических коммитов. Для этого вам нужно установить `markdownlint-cli` глобально: `npm install -g markdownlint-cli`.
*   **Папка `docs/`**: Убедитесь, что вся документация, которую вы хотите опубликовать на GitHub Pages, находится в папке `docs/`.
