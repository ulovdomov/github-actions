# GitHub Workflows

[GitHub Actions](https://docs.github.com/en/actions) reusable workflows and composed actions

- You have to use `./temp/.php-codesniffer-cache` as temp dir in `phpcs` ruleset.xml
- You have to use `./temp/.php-stan-cache` as temp dir in `phpstan` configuration

Example complex configuration:

```yaml
name: CI

on:
  pull_request:
    types: [ "opened", "synchronize", "edited", "reopened" ]
    paths-ignore:
      - "docs/**"
      - "README.md"
  push:
    branches:
      - "**"
    paths-ignore:
      - "docs/**"
      - "README.md"

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
        uses: "ulovdomov/github-actions/.github/actions/docker-compose@v1"
        with:
          command: 'docker compose -f "docker-compose.ci.yml" up -d'
          checked-files: ".infrastructure/docker/php/Dockerfile docker-compose.ci.yml .infrastructure/docker/php/entrypoint-staging.sh .infrastructure/docker/php/php.ini"
          composer-token: ${{ secrets.UD_COMPOSER_TOKEN }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          nginx: false # Optional nginx container (default: false)

      - name: "PHP_CodeSniffer"
        uses: "ulovdomov/github-actions/.github/actions/php-codesniffer@v1"
        with:
          command: 'docker compose -f "docker-comose.ci.yml" exec -it php composer run cs'
          cache-path: "temp/.php-codesniffer-cache"

      - name: "PHPStan"
        uses: "ulovdomov/github-actions/.github/actions/php-stan@v1"
        with:
          command: 'docker compose -f "docker-compose.ci.yml" exec -it php composer run phpstan'
          cache-path: "temp/.php-stan-cache"

      - name: Run tests
        run: docker compose -f "docker-compose.ci.yml" exec -it php composer run tests

      - name: "Upload_Artifacts"
        if: failure()
        uses: "ulovdomov/github-actions/.github/actions/upload-artifacts@v1"
        with:
          log-dir: "./log"
```
