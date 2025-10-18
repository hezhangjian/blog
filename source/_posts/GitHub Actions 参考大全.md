---
title: GitHub Actions 参考大全
link: github-actions-reference
date: 2023-11-24 21:29:21
tags:
  - GitHub Actions
---
# 通用 GitHub Actions

## commit lint

```yaml
name: commit lint
on:
  pull_request:
    branches:
      - main

jobs:
  commitlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: wagoid/commitlint-github-action@v5
```

## line lint

```yaml
name: line lint
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
jobs:
  build:
    name: line lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: linelint
        uses: fernandrone/linelint@master
```

# Go

## golangci-lint

```yaml
name: go ci Lint
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
jobs:
  golangci:
    name: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v4
        with:
          go-version: '1.21'
      - name: golangci-lint
        uses: golangci/golangci-lint-action@v3
        with:
          version: latest
          args: --timeout 3m0s
```

## go mod check

```yaml
name: go mod check

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  go_mod_check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Go Mod Check Action
        uses: hezhangjian/go-mod-check-action@main
        with:
          prohibitIndirectDepUpdate: 'true'
```

## go unit tests

```yaml
name: go unit test

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  go_unit_test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v4
        with:
          go-version: '1.21'
      - name: setup OpenGemini
        uses: hezhangjian/setup-opengemini-action@main
      - name: Run coverage
        run: go test ./... -coverpkg=./padmin/... -race -coverprofile=coverage.out -covermode=atomic
```

# Java GitHub Actions

## maven checkstyle

```yaml
name: java checkstyle
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  java_checkstyle:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Maven Central Repository
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: checkstyle
        run: mvn -B clean checkstyle:check
```

## maven spotbugs

```yaml
name: java spotbugs
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  java_spotbugs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Maven Central Repository
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: spotbugs
        run: mvn -B -DskipTests clean verify spotbugs:spotbugs
```

## maven unit tests

```yaml
name: java unit tests
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  java_unit_tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Maven Central Repository
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: unit tests
        run: mvn -B clean test
```

# TypeScript GitHub Actions

## npm build test

```yaml
name: npm build test
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  npm_buid_test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: latest
      - run: npm install
      - run: npm run build
      - name: setup pulsar
        uses: hezhangjian/setup-pulsar-action@main
      - run: npm run test
```

## prettier

```yaml
name: prettier
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  prettier:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: latest
      - run: npm install --save-dev prettier
      - run: npm install --global prettier
      - run: prettier --check '**/*.ts'
```
