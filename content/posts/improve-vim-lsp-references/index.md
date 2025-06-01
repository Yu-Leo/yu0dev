---
title: Как улучшить LSP поиск референсов в Neovim
slug: improve-vim-lsp-references
date: 2025-06-01
tags:
  - vim
description: Как улучшить LSP textDocument/references в Neovim
keywords:
  - lsp
  - vim
  - neovim
  - nvim
  - references
---
Сегодня расскажу о том, как я улучшил свой опыт использования LSP функции `textDocument/references` в Neovim.

Описанное решение актуально для **Neovim >= 0.11.0**. Я проверил его работоспособность на v0.11.1. В качестве языка для демонстрации используется **Go** (LSP-сервер **[gopls](https://pkg.go.dev/golang.org/x/tools/gopls)**). Всё будет работать аналогично и для других LSP-серверов.

Я не буду подробно описывать, что такое LSP. Почитать об этом можно в тут: https://habr.com/ru/companies/sberbank/articles/838786/

## Это - база

После релиза [Neovim v0.11.0](https://neovim.io/doc/user/news-0.11.html) для подключения и базовой настройки LSP нужно добавить в конфиг буквально пару команд. На примере gopls:

```lua
vim.lsp.config("gopls", {
  cmd = { "gopls", "serve" },
  filetypes = { "go", "go.mod" },
  root_markers = { "go.work", "go.mod", ".git" },
})

vim.lsp.enable "gopls"
```

Чтобы вызвать функцию поиска референсов (LSP `textDocument/references`), можно использовать дефолтный хоткей `grr` в нормальном режиме.

## Проблемы

### Проблема №0

**Дефолтный хоткей для меня абсолютно неудобен.**

Полагаю, при придумывании мапинга `grr` разработчики Neovim руководствовались следующей логикой:
- `g` - одна из немногих незанятых в нормальном режиме по умолчанию клавиш. На неё в Vim/Neovim навешано много разных функций "из коробки"
- `rr` - сокращение от слова **R**efe**R**ences (по аналогии `grn` - **R**e**N**ame )

Мне неудобно нажимать хоткей, состоящий из трёх клавиш, **одним пальцем**. Особенно если в нём две соседние клавиши - одинаковые.

В конфиге, который я писал ещё под Neovim 0.10.4 (когда дефолтных маппингов для LSP функций ещё не было) я замапил функцию поиска референсов на `gk`.

### Проблема №1

**В списке референсов есть определение искомого объекта.**

Ситуация: я вижу в коде использование функции и хочу понять, где **ещё** она используется. Когда я вызываю функцию `textDocument/references`, в полученном списке помимо вызовов искомой функции я вижу строчку, в которой эта функция определена.

Такое поведение мне не нравится. Если я захочу перейти к определению функции/переменной/..., я использую `textDocument/definition`. Под референсами мне привычнее понимать именно использования объекта, но не его определение.

### Проблема №2

**В списке референсов есть объект, на котором была вызвана функция поиска.**

Ситуация: я вижу в коде использование функции и хочу понять, где **ещё** она используется. Когда я вызываю функцию `textDocument/references`, в полученном списке помимо прочих вызовов я вижу и то использование, "стоя" курсором на котором я нажал хоткей для поиска референсов.

### Проблема №3

**Список рефенерсов открывается через QuickfixList, даже если в нём  один элемент.**

Ситуация: я вижу в коде определение функции и хочу понять, где она используется. Когда я нажимаю хоткей для поиска референсов, возможны следующие ситуации:
- Если референсы не найдены, появится строка `"No references found"`
- Если найден **хоты бы один** референс, откроется QuickfixList с найденными элементами

При этом, как правило, сразу после поиска референсов я перехожу к одному из них. Следовательно, если референс всего один, этот переход можно автоматизировать и избавится от лишнего нажатия клавиши в QuickfixList.

## Решение

Проблема №0 в Neovim 0.11.0+ решается "классическим" способом - определением кастомных маппингов в функции `on_attach`. Так обычно задавались маппинги для LSP в версиях Neovim до 0.11.0

Для решения проблемы №1 в спецификации LSP у `textDocument/references` [определён](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_references) параметр `includeDeclaration`. Он определяет, нужно ли включить определение объекта в результат функции. По умолчанию = `true`.

Проблемы №2 и №3 интереснее, поскольку их невозможно решить путём передачи каких-либо параметров - таковые не предусмотрены в Neovim "из коробки". 

Чтобы добиться желаемого поведения, я определил собственную функцию для поиска референсов. Я взял [реализацию дефолтной фукнции поиска референсов](https://github.com/neovim/neovim/blob/v0.11.1/runtime/lua/vim/lsp/buf.lua#L764-L819) из исходников Neovim и немного доработал её.

Вот что у меня получилось в итоге:

```lua {lineNos=false title="nvim/init.lua"}
-- Прочие настройки
-- ...
require "lsp"
```

```lua {lineNos=false title="nvim/lsp/init.lua"}
local utils = require "lsp.utils"

vim.lsp.config("gopls", {
  cmd = { "gopls", "serve" },
  filetypes = { "go", "go.mod" },
  root_markers = { "go.work", "go.mod", ".git" },
  on_init = utils.on_init,
  on_attach = utils.on_attach,
})

vim.lsp.enable "gopls"
```

```lua {lineNos=false title="nvim/lsp/utils.lua"}
local M = {}

local map = vim.keymap.set

M.on_init = function(client, _)
  if client.supports_method "textDocument/semanticTokens" then
    client.server_capabilities.semanticTokensProvider = nil
  end
end

M.on_attach = function(client, bufnr)
  local function map_opts(desc)
    return { buffer = bufnr, desc = "LSP: " .. desc }
  end

  local floating_window_opts = {
    border = "single",
    max_width = 80,
  }

  map("n", "gD", vim.lsp.buf.declaration, map_opts "go to declaration")
  map("n", "gd", vim.lsp.buf.definition, map_opts "go to definition")
  map("n", "gi", vim.lsp.buf.implementation, map_opts "go to implementation")
  map("n", "gj", vim.lsp.buf.type_definition, map_opts "go to type definition")

  map("n", "gk", function()
    require("lsp.handlers").references { includeDeclaration = false }
  end, map_opts "show references")

  map("n", "K", function()
    return vim.lsp.buf.hover(floating_window_opts)
  end, map_opts "hover")

  map("n", "<leader>ld", function()
    return vim.diagnostic.open_float(floating_window_opts)
  end, map_opts "diagnostic open float")

  map("n", "<leader>ls", function()
    return vim.lsp.buf.signature_help(floating_window_opts)
  end, map_opts "show signature help")

  map("i", "<c-s>", function()
    return vim.lsp.buf.signature_help(floating_window_opts)
  end, map_opts "show signature help")

  map("n", "<leader>lr", vim.lsp.buf.rename, map_opts "rename")

  map({ "n", "v" }, "<leader>la", vim.lsp.buf.code_action, map_opts "code action")
  map({ "n", "v" }, "<leader>ll", vim.lsp.codelens.run, map_opts "code lens")
end

return M
```

```lua {lineNos=false title="nvim/lsp/handlers.lua"}
local M = {}

local util = require "vim.lsp.util"
local ms = require("vim.lsp.protocol").Methods

local filter_references = function(result)
  if result == nil or #result == 0 then
    return {}
  end

  local current_position = vim.api.nvim_win_get_cursor(0)
  local current_uri = vim.uri_from_bufnr(0)

  local filtered_result = {}

  for _, ref in ipairs(result) do
    if
      ref.user_data.uri ~= current_uri
      or ref.user_data.range.start.line ~= current_position[1] - 1
      or ref.user_data.range.start.character > current_position[2]
    then
      table.insert(filtered_result, ref)
    end
  end

  return filtered_result
end

local function on_done(client, items, bufnr)
  items = filter_references(items)

  if not next(items) then
    vim.notify "No references found"
    return
  end

  if #items == 1 then
    vim.lsp.util.show_document({
      uri = items[1].user_data.uri,
      range = items[1].user_data.range,
    }, client.offset_encoding, { reuse_win = true, focus = true })
    return
  end

  local list = {
    title = "References",
    items = items,
    context = {
      method = ms.textDocument_references,
      bufnr = bufnr,
    },
  }

  vim.fn.setloclist(0, {}, " ", list)
  vim.cmd.lopen()
end

M.references = function(context)
  local bufnr = vim.api.nvim_get_current_buf()
  local clients = vim.lsp.get_clients { method = ms.textDocument_references, bufnr = bufnr }
  if not next(clients) then
    return
  end

  local win = vim.api.nvim_get_current_win()

  local all_clients_items = {}

  local remaining = #clients
  for _, client in ipairs(clients) do
    local params = util.make_position_params(win, client.offset_encoding)

    ---@diagnostic disable-next-line: inject-field
    params.context = context or {
      includeDeclaration = true,
    }

    client:request(ms.textDocument_references, params, function(_, result)
      local client_items = util.locations_to_items(result or {}, client.offset_encoding)
      vim.list_extend(all_clients_items, client_items)
      remaining = remaining - 1
      if remaining == 0 then
        on_done(client, all_clients_items, bufnr)
      end
    end)
  end
end

return M
```

- `filter_references` - фильтрует результаты поиска референсов, исключая из них тот референс, на котором была вызвана фукнция
- `if #items == 1 then ...` - автоматический переход к референсу, если он в списке всего один

Мой конфиг Neovim: https://github.com/Yu-Leo/nvim
