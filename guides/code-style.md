# Code Style

Guide for maintaining consistent code style through linting, formatting, and standards.

## Why Code Style Matters

- **Readability**: Consistent code is easier to read and understand
- **Maintainability**: Reduces cognitive load when switching between files
- **Collaboration**: Team members can work on any part of the codebase
- **Automation**: Reduces bikeshedding in code reviews
- **Quality**: Many style rules prevent bugs

## The Golden Rule

> **Configure once, automate always.**

Code style should be enforced automatically, not through manual review comments.

---

## Formatting vs Linting

| Aspect | Formatting | Linting |
|--------|------------|---------|
| **Purpose** | Code appearance | Code quality |
| **Examples** | Indentation, spacing, line length | Unused vars, complexity, bugs |
| **Tools** | Prettier, Black, gofmt | ESLint, Pylint, golint |
| **Automation** | Format on save | CI blocking |

---

## Language-Specific Setup

### JavaScript/TypeScript

#### ESLint Configuration

```javascript
// .eslintrc.js
module.exports = {
  root: true,
  parser: "@typescript-eslint/parser",
  parserOptions: {
    ecmaVersion: 2022,
    sourceType: "module",
    project: "./tsconfig.json",
  },
  plugins: [
    "@typescript-eslint",
    "import",
    "prettier",
  ],
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:@typescript-eslint/recommended-requiring-type-checking",
    "plugin:import/recommended",
    "plugin:import/typescript",
    "prettier",
  ],
  rules: {
    // TypeScript
    "@typescript-eslint/explicit-function-return-type": "error",
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/no-unused-vars": ["error", { argsIgnorePattern: "^_" }],
    "@typescript-eslint/strict-boolean-expressions": "error",
    
    // Imports
    "import/order": [
      "error",
      {
        groups: ["builtin", "external", "internal", "parent", "sibling", "index"],
        "newlines-between": "always",
        alphabetize: { order: "asc" },
      },
    ],
    "import/no-cycle": "error",
    "import/no-duplicates": "error",
    
    // General
    "no-console": ["warn", { allow: ["warn", "error"] }],
    "no-debugger": "error",
    "prefer-const": "error",
    "no-var": "error",
    eqeqeq: ["error", "always"],
    curly: ["error", "all"],
  },
  settings: {
    "import/resolver": {
      typescript: {
        project: "./tsconfig.json",
      },
    },
  },
};
```

#### Prettier Configuration

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100,
  "bracketSpacing": true,
  "arrowParens": "avoid",
  "endOfLine": "lf"
}
```

```json
// .prettierignore
node_modules/
dist/
build/
coverage/
*.min.js
```

### Python

#### Ruff Configuration (Modern, Fast)

```toml
# pyproject.toml
[tool.ruff]
target-version = "py312"
line-length = 100
select = [
    "E",      # pycodestyle errors
    "W",      # pycodestyle warnings
    "F",      # Pyflakes
    "I",      # isort
    "B",      # flake8-bugbear
    "C4",     # flake8-comprehensions
    "UP",     # pyupgrade
    "ARG",    # flake8-unused-arguments
    "SIM",    # flake8-simplify
    "TCH",    # flake8-type-checking
    "PTH",    # flake8-use-pathlib
    "ERA",    # eradicate (commented code)
    "PL",     # Pylint
    "RUF",    # Ruff-specific rules
]
ignore = [
    "E501",   # line too long (handled by formatter)
    "PLR0913", # too many arguments
]

[tool.ruff.per-file-ignores]
"tests/*" = ["S101", "ARG"]  # Allow assert in tests

[tool.ruff.isort]
known-first-party = ["myapp"]

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

#### Black Configuration

```toml
# pyproject.toml
[tool.black]
line-length = 100
target-version = ["py312"]
include = '\.pyi?$'
exclude = '''
/(
    \.git
    | \.mypy_cache
    | \.venv
    | dist
    | build
)/
'''
```

#### MyPy Configuration

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_ignores = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
no_implicit_optional = true

[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false
```

### Go

#### golangci-lint Configuration

```yaml
# .golangci.yml
run:
  timeout: 5m
  go: "1.22"

linters:
  enable:
    - errcheck
    - gosimple
    - govet
    - ineffassign
    - staticcheck
    - unused
    - gofmt
    - goimports
    - gocritic
    - revive
    - misspell
    - prealloc
    - unconvert
    - unparam
    - bodyclose
    - noctx
    - sqlclosecheck
    - rowserrcheck

linters-settings:
  gofmt:
    simplify: true
  goimports:
    local-prefixes: github.com/myorg/myapp
  gocritic:
    enabled-tags:
      - diagnostic
      - style
      - performance
  revive:
    rules:
      - name: blank-imports
      - name: context-as-argument
      - name: error-return
      - name: error-strings
      - name: exported
      - name: increment-decrement
      - name: var-naming
      - name: package-comments
        disabled: true

issues:
  exclude-rules:
    - path: _test\.go
      linters:
        - errcheck
        - gocritic
```

### Java

#### Checkstyle Configuration

```xml
<!-- checkstyle.xml -->
<?xml version="1.0"?>
<!DOCTYPE module PUBLIC
    "-//Checkstyle//DTD Checkstyle Configuration 1.3//EN"
    "https://checkstyle.org/dtds/configuration_1_3.dtd">

<module name="Checker">
    <property name="charset" value="UTF-8"/>
    <property name="severity" value="error"/>
    
    <module name="TreeWalker">
        <!-- Naming -->
        <module name="ConstantName"/>
        <module name="LocalFinalVariableName"/>
        <module name="LocalVariableName"/>
        <module name="MemberName"/>
        <module name="MethodName"/>
        <module name="PackageName"/>
        <module name="ParameterName"/>
        <module name="TypeName"/>
        
        <!-- Imports -->
        <module name="IllegalImport"/>
        <module name="RedundantImport"/>
        <module name="UnusedImports"/>
        
        <!-- Size -->
        <module name="LineLength">
            <property name="max" value="120"/>
        </module>
        <module name="MethodLength">
            <property name="max" value="50"/>
        </module>
        <module name="ParameterNumber">
            <property name="max" value="5"/>
        </module>
        
        <!-- Whitespace -->
        <module name="WhitespaceAround"/>
        <module name="NoWhitespaceBefore"/>
        
        <!-- Coding -->
        <module name="EmptyStatement"/>
        <module name="EqualsHashCode"/>
        <module name="SimplifyBooleanExpression"/>
        <module name="SimplifyBooleanReturn"/>
    </module>
    
    <module name="FileLength">
        <property name="max" value="500"/>
    </module>
</module>
```

---

## Editor Integration

### VS Code Settings

```json
// .vscode/settings.json
{
  // Format on save
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  
  // Language-specific
  "[javascript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.codeActionsOnSave": {
      "source.organizeImports": "explicit",
      "source.fixAll": "explicit"
    }
  },
  "[go]": {
    "editor.defaultFormatter": "golang.go",
    "editor.codeActionsOnSave": {
      "source.organizeImports": "explicit"
    }
  },
  
  // ESLint
  "eslint.validate": [
    "javascript",
    "typescript"
  ],
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  
  // Files
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "files.trimFinalNewlines": true
}
```

### VS Code Extensions

```json
// .vscode/extensions.json
{
  "recommendations": [
    "esbenp.prettier-vscode",
    "dbaeumer.vscode-eslint",
    "charliermarsh.ruff",
    "golang.go",
    "ms-python.python",
    "editorconfig.editorconfig"
  ]
}
```

### EditorConfig

```ini
# .editorconfig
root = true

[*]
indent_style = space
indent_size = 2
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.py]
indent_size = 4

[*.go]
indent_style = tab

[*.md]
trim_trailing_whitespace = false

[Makefile]
indent_style = tab
```

---

## Git Hooks with Pre-commit

### Configuration

```yaml
# .pre-commit-config.yaml
repos:
  # General
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-added-large-files
        args: ['--maxkb=1000']
      - id: check-merge-conflict
      - id: detect-private-key
      - id: no-commit-to-branch
        args: ['--branch', 'main', '--branch', 'master']
  
  # Secrets
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
  
  # JavaScript/TypeScript
  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: v3.1.0
    hooks:
      - id: prettier
        types_or: [javascript, typescript, json, yaml, markdown]
  
  - repo: https://github.com/pre-commit/mirrors-eslint
    rev: v8.56.0
    hooks:
      - id: eslint
        files: \.[jt]sx?$
        types: [file]
        additional_dependencies:
          - eslint
          - "@typescript-eslint/parser"
          - "@typescript-eslint/eslint-plugin"
  
  # Python
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.9
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies: [types-all]
  
  # Go
  - repo: https://github.com/golangci/golangci-lint
    rev: v1.55.2
    hooks:
      - id: golangci-lint
  
  # Commit messages
  - repo: https://github.com/commitizen-tools/commitizen
    rev: v3.13.0
    hooks:
      - id: commitizen
        stages: [commit-msg]
```

### Installation

```bash
# Install pre-commit
pip install pre-commit

# Install hooks
pre-commit install
pre-commit install --hook-type commit-msg

# Run on all files
pre-commit run --all-files

# Update hooks
pre-commit autoupdate
```

---

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/lint.yml
name: Lint

on: [push, pull_request]

jobs:
  lint-js:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      
      - name: Run ESLint
        run: npm run lint
      
      - name: Check Formatting
        run: npm run format:check

  lint-python:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      
      - name: Install dependencies
        run: pip install ruff mypy
      
      - name: Run Ruff
        run: ruff check .
      
      - name: Run Ruff Format Check
        run: ruff format --check .
      
      - name: Run MyPy
        run: mypy src/

  lint-go:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      
      - name: Run golangci-lint
        uses: golangci/golangci-lint-action@v3
        with:
          version: latest
```

### Package.json Scripts

```json
{
  "scripts": {
    "lint": "eslint . --ext .js,.jsx,.ts,.tsx",
    "lint:fix": "eslint . --ext .js,.jsx,.ts,.tsx --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "typecheck": "tsc --noEmit",
    "validate": "npm run lint && npm run format:check && npm run typecheck"
  }
}
```

### Makefile

```makefile
# Makefile
.PHONY: lint format check

lint:
	ruff check .
	mypy src/

format:
	ruff format .
	ruff check --fix .

format-check:
	ruff format --check .

check: format-check lint

# For CI
ci-lint:
	ruff check --output-format=github .
	mypy src/ --no-error-summary
```

---

## Common Style Rules

### Code Complexity

```javascript
// ESLint complexity rules
{
  "rules": {
    "complexity": ["error", 10],           // Cyclomatic complexity
    "max-depth": ["error", 4],             // Nesting depth
    "max-lines-per-function": ["error", 50],
    "max-params": ["error", 4],
    "max-statements": ["error", 15]
  }
}
```

### Import Organization

```python
# Python import order (enforced by ruff/isort)
# 1. Standard library
import os
import sys
from datetime import datetime

# 2. Third-party
import requests
from fastapi import FastAPI
from pydantic import BaseModel

# 3. Local application
from myapp.models import User
from myapp.services import UserService
```

```typescript
// TypeScript import order (enforced by ESLint)
// 1. Node built-ins
import fs from 'fs';
import path from 'path';

// 2. External packages
import express from 'express';
import { z } from 'zod';

// 3. Internal aliases
import { config } from '@/config';
import { logger } from '@/utils/logger';

// 4. Relative imports
import { UserService } from '../services/user';
import { User } from './types';
```

### Naming Patterns

| Element | JavaScript/TypeScript | Python | Go |
|---------|----------------------|--------|-----|
| Variables | `camelCase` | `snake_case` | `camelCase` |
| Functions | `camelCase` | `snake_case` | `PascalCase` (exported) |
| Classes | `PascalCase` | `PascalCase` | `PascalCase` |
| Constants | `SCREAMING_SNAKE` | `SCREAMING_SNAKE` | `PascalCase` |
| Files | `kebab-case.ts` | `snake_case.py` | `lowercase.go` |

---

## Style Guide References

| Language | Official Guide |
|----------|---------------|
| JavaScript | [Airbnb Style Guide](https://github.com/airbnb/javascript) |
| TypeScript | [Google TypeScript Guide](https://google.github.io/styleguide/tsguide.html) |
| Python | [PEP 8](https://peps.python.org/pep-0008/), [Google Python Guide](https://google.github.io/styleguide/pyguide.html) |
| Go | [Effective Go](https://go.dev/doc/effective_go), [Go Code Review](https://go.dev/wiki/CodeReviewComments) |
| Java | [Google Java Guide](https://google.github.io/styleguide/javaguide.html) |

---

## Code Style Checklist

### Project Setup
- [ ] Linter configured with sensible rules
- [ ] Formatter configured
- [ ] EditorConfig for cross-editor consistency
- [ ] VS Code settings committed
- [ ] Pre-commit hooks installed
- [ ] CI pipeline runs checks

### Development Flow
- [ ] Format on save enabled
- [ ] Lint errors shown in editor
- [ ] Pre-commit prevents bad commits
- [ ] CI blocks non-compliant PRs

### Team Alignment
- [ ] Style guide documented or referenced
- [ ] Exceptions documented in config
- [ ] New team members onboarded to setup
- [ ] Regular config updates (quarterly)
