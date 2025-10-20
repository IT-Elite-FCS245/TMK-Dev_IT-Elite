## Руководство по Настройке GitHub Pages

Этот документ описывает пошаговый процесс настройки и использования GitHub Pages для публикации документации вашего проекта. GitHub Pages позволяет хостить статические веб-сайты непосредственно из вашего репозитория GitHub, что идеально подходит для демонстрации проектной документации.

### 1. Обновление GitHub Actions Workflow

Первым шагом является интеграция задачи развертывания GitHub Pages в ваш существующий файл рабочего процесса GitHub Actions. Это обеспечит автоматическое развертывание вашей документации при каждом изменении в главной ветке.

**Файл:** `https://github.com/IT-Elite-FCS245/TMK-Dev_IT-Elite/blob/main/.github/workflows/markdown-lint.yml`

**Необходимые изменения:** Добавьте новый `job` под названием `deploy-pages` к вашему рабочему процессу. Этот `job` будет зависеть от успешного завершения `lint` задачи (`needs: lint`), чтобы гарантировать, что документация развертывается только после прохождения всех проверок качества.

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
    needs: lint # Запускается только после успешного завершения задачи lint
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
          path: './docs' # Папка с вашей документацией, которую нужно опубликовать
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

**Важные замечания:**

*   `needs: lint`: Указывает, что этот `job` будет выполнен только после успешного завершения `lint` задачи. Это гарантирует, что на GitHub Pages публикуется только отформатированная и проверенная документация.
*   `permissions`: Предоставляет необходимые разрешения для GitHub Actions для взаимодействия с сервисом GitHub Pages.
*   `environment`: Определяет среду развертывания, что полезно для отслеживания статуса развертывания.
*   `path: './docs'`: Указывает, что содержимое папки `docs/` будет опубликовано. Убедитесь, что все ваши Markdown файлы, которые вы хотите видеть на сайте, находятся в этой папке.

### 2. Настройка GitHub Pages в Репозитории

После того как ваш рабочий процесс с `deploy-pages` будет добавлен и объединен в ветку `main`, вам необходимо активировать GitHub Pages в настройках вашего репозитория.

**Действия для владельца репозитория:**

1.  На странице вашего репозитория GitHub перейдите на вкладку **`Settings`** (Настройки).
2.  В левом меню выберите **`Pages`**.
3.  В разделе **`Build and deployment`** (Сборка и развертывание):
    *   В выпадающем списке **`Source`** (Источник) выберите **`GitHub Actions`**.
    *   Нажмите кнопку **`Save`** (Сохранить).

### 3. Проверка Работы GitHub Pages

После сохранения настроек и успешного завершения рабочего процесса `deploy-pages` (это может занять несколько минут после коммита в `main` или слияния PR), вернитесь в **`Settings` -> `Pages`**.

Вы увидите URL-адрес вашего сайта GitHub Pages, например, `https://your-username.github.io/GroupDynamicsProject/`.

Перейдите по этой ссылке, чтобы убедиться, что ваша документация, расположенная в папке `docs/`, опубликована и доступна как веб-сайт.
