# GitHub Workflows

[GitHub Actions](https://docs.github.com/en/actions) reusable workflows and composed actions

- You have to use `./temp/.php-codesniffer-cache` as temp dir in `phpcs` ruleset.xml
- You have to use `./temp/.php-stan-cache` as temp dir in `phpstan` configuration

Example complex configuration:

```yaml
name: CI

on:
  pull_request:
  push:
    branches:
      - "**"

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-22.04
    name: Build
    steps:
      - uses: actions/checkout@v3

      - name: Write .env file
        run: echo "COMPOSER_TOKEN=${{ secrets.UD_COMPOSER_TOKEN }}" > .env

      - name: "Docker_Compose"
        uses: "./.github/actions/docker-compose"
        with:
          command: 'docker compose -f "docker-compose.ci.yml" up -d'
          checked-files: ".infrastructure/docker/php/Dockerfile docker-compose.ci.yml .infrastructure/docker/php/entrypoint-staging.sh .infrastructure/docker/php/php.ini"
          composer-token: ${{ secrets.UD_COMPOSER_TOKEN }}
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: "PHP_CodeSniffer"
        uses: "./.github/actions/php-codesniffer"
        with:
          command: 'docker compose -f "docker-compose.ci.yml" exec -it php composer run cs'
          cache-path: "./temp/.php-codesniffer-cache"

      - name: "PHPStan"
        uses: "./.github/actions/php-stan"
        with:
          command: 'docker compose -f "docker-compose.ci.yml" exec -it php composer run phpstan'
          cache-path: "./temp/.php-stan-cache"

      - name: Run tests
        run: docker compose -f "docker-compose.ci.yml" exec -it php composer run tests

      - name: "Upload_Artifacts"
        uses: "./.github/actions/upload-artifacts"
        with:
          log-dir: "./log"
```
