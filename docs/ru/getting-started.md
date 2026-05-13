---
title: Начало работы
layout: default
parent: Русский
nav_order: 2
lang: ru
counterpart: /en/getting-started/
---

{%- include lang-switcher.html -%}

# Начало работы

## Требования

- Установленный [Claude Code](https://claude.ai/code)
- PHP-проект для аудита

Устанавливать PHP-инструменты на вашу машину не обязательно. Скилл определяет, что доступно, и подстраивается. Если инструментов нет совсем, он всё равно проведёт архитектурный анализ через чтение файлов.

## Установка

Все три способа размещают скилл там, где Claude Code его обнаружит. Дополнительная настройка не нужна.

### Способ 1 — Для одного проекта (рекомендуется)

```bash
mkdir -p .claude/skills
git clone https://github.com/galo4kin/php-tech-debt-skill.git .claude/skills/php-tech-debt-audit
```

Скилл устанавливается только для текущего проекта. Добавьте `.claude/skills/php-tech-debt-audit/` в `.gitignore`, если не хотите коммитить его.

### Способ 2 — Глобально (все проекты)

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/galo4kin/php-tech-debt-skill.git ~/.claude/skills/php-tech-debt-audit
```

Скилл станет доступен во всех проектах, которые вы открываете в Claude Code.

### Способ 3 — curl (один файл)

```bash
mkdir -p .claude/skills/php-tech-debt-audit
curl -o .claude/skills/php-tech-debt-audit/SKILL.md \
  https://raw.githubusercontent.com/galo4kin/php-tech-debt-skill/main/.claude/skills/php-tech-debt-audit/SKILL.md
curl -o .claude/skills/php-tech-debt-audit/report-template.md \
  https://raw.githubusercontent.com/galo4kin/php-tech-debt-skill/main/.claude/skills/php-tech-debt-audit/report-template.md
```

## Использование

Аудит всего проекта:

```
/php-tech-debt-audit
```

Аудит конкретной директории:

```
/php-tech-debt-audit src/
```

## Что происходит при запуске

Скилл работает в 5 фаз:

**Фаза 1 — Обнаружение окружения.** Проверяет, работает ли PHP на хосте или внутри Docker-контейнера. Сканирует 12 инструментов анализа (PHPStan, semgrep, phpcs, PHPUnit и др.) и сообщает, что доступно.

**Фаза 2 — Ориентация.** Читает `composer.json`, `README.md`, структуру каталогов, историю git и самые большие файлы. Строит ментальную модель кодовой базы перед началом анализа.

**Фаза 3 — Оценка инструментами.** Запускает каждый обнаруженный инструмент и выставляет оценки по 5 категориям (Безопасность, Статический анализ, Зависимости, Качество кода, Покрытие тестами). Каждая категория — от 0 до 20 баллов. Итог нормализуется до 100.

**Фаза 4 — Архитектурный аудит.** Анализ на основе Claude по 9 измерениям с использованием целевых поисков (`grep`, `rg`, `find`). Каждая находка получает ссылку `файл:строка`, уровень серьёзности и оценку трудозатрат.

**Фаза 5 — Отчёт.** Записывает `TECH_DEBT_AUDIT.md` в корень проекта со всеми находками, оценками, приоритетами и быстрыми исправлениями. Выводит сводку в консоль.

## Поддержка Docker

Если PHP не найден на хосте, скилл автоматически:

1. Ищет `docker-compose.yml` в корне проекта и в директории `docker/`
2. Определяет PHP-сервис по имени
3. Проверяет, что контейнер запущен
4. Выполняет все PHP и Composer команды через `docker compose exec -T <service>`

Настройка не нужна. Инструменты, которые читают исходные файлы напрямую (например, semgrep), продолжают работать на хосте.

Если ни хостовый PHP, ни Docker PHP не найдены, оценка инструментами пропускается и запускается только архитектурный анализ.
