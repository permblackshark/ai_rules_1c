---
description: Editing form-module code (`Form.Module.bsl` / ФормаМодуль) — client-server interaction, async pointers, form-data conversion. Load from `forms.md` when editing form-module logic.
alwaysApply: false
category: forms
---

# Form Module Guidelines

This file is intentionally short — it owns only form-module-specific topics that have no other home. Everything else delegates to its single source of truth.

## Client-Server Interaction and Compilation Directives

Single source of truth — `dev-standards-architecture.md §3 → "Client-Server Interaction"` (mandate of `&НаСервереБезКонтекста`, ban on dialogs on the server) plus `anti-patterns.md §6 → "Excessive Client-Server Calls"` and `§7 → "Using &НаСервере Instead of &НаСервереБезКонтекста"` for examples and severity.

Do not duplicate those rules here. The directive table (`&НаКлиенте`, `&НаСервере`, `&НаСервереБезКонтекста`, `&НаКлиентеНаСервереБезКонтекста`) and the "minimize round trips" guidance are covered there.

## Async Programming

Patterns, pitfalls, and platform-version mapping (8.3.18+ `Асинх` / `Ждать` vs older `ОписаниеОповещения`) live in `async-methods.md`. Load it before writing client-side async code.

## Reserved Names

`form-reserved-names.md` lists property names forbidden as local variables in form modules (`ПараметрыВыбора`, `СвязиПараметровВыбора`, `СписокВыбора`, `ПараметрыОтбора`, `ОтборСтрок`). Load whenever writing or refactoring server-side form code.

## Form Data

- Use `ДанныеФормыВЗначение()` / `ЗначениеВДанныеФормы()` to convert between form data and actual objects.
- Remember that form attributes are not the same as object attributes — they are form-specific representations.

## Module Structure

The 5-region template for form modules (`ОбработчикиСобытийФормы`, `ОбработчикиСобытийЭлементовШапкиФормы`, `ОбработчикиСобытийЭлементовТаблицыФормыИмяТаблицы`, `ОбработчикиКомандФормы`, `СлужебныеПроцедурыИФункции`) — see `module-structure.md → Form Module`. All 5 regions are mandatory even when empty.
