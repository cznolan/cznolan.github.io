---
layout: post
title: "Basic Neovim Setup for Python"
date: 2024-08-31 18:30:00 +0800
tags: python neovim nvim lsp pyright wsl
author: cznolan
---


I'm setting up Neovim within Ubuntu 24.04 running in Windows Subsystem for Linux (WSL2).

The plugins I am using are: 
* Lazy for plugin management. 
* Mason for LSP/tools management. 
* Pyright as the language server.
* Treesitter for syntax highlighting. 
* Telescope for fuzzy finding. 
* Catppuccin for colour scheme.

## External Dependencies

At time of writing the Ubuntu APT repositories have an older version of Neovim, so I have installed it manually. For grep'ing through files with Telescope, ripgrep needs to be installed.

Pyright needs to be installed as well, which I have done via NPM.

```
sudo apt install ripgrep
npm install -g pyright
```

### Lazy

The installation instructions provided for Lazy here are perfectly fine. The only change I have made is removing the habamax colour scheme from the lazy.lua file, as I intend to use Catppuccin.
```lua
--~/.config/nvim/init.lua
--
require("config.lazy")
```

```lua
--~/.config/nvim/config/lazy.lua
--
-- Bootstrap lazy.nvim
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not (vim.uv or vim.loop).fs_stat(lazypath) then
  local lazyrepo = "https://github.com/folke/lazy.nvim.git"
  local out = vim.fn.system({ "git", "clone", "--filter=blob:none", "--branch=stable", lazyrepo, lazypath })
  if vim.v.shell_error ~= 0 then
    vim.api.nvim_echo({
      { "Failed to clone lazy.nvim:\n", "ErrorMsg" },
      { out, "WarningMsg" },
      { "\nPress any key to exit..." },
    }, true, {})
    vim.fn.getchar()
    os.exit(1)
  end
end
vim.opt.rtp:prepend(lazypath)

-- Make sure to setup `mapleader` and `maplocalleader` before
-- loading lazy.nvim so that mappings are correct.
-- This is also a good place to setup other settings (vim.opt)
vim.g.mapleader = " "
vim.g.maplocalleader = "\\"

-- Setup lazy.nvim
require("lazy").setup({
  spec = {
    -- import your plugins
    { import = "plugins" },
  },
  -- Configure any other settings here. See the documentation for more details.
  -- automatically check for plugin updates
  checker = { enabled = true },
})

```

### Treesitter

Again not much to say here. I've basically just followed the installation instructions for Treesitter here [https://github.com/nvim-treesitter/nvim-treesitter/wiki/Installation](https://github.com/nvim-treesitter/nvim-treesitter/wiki/Installation){:target="_blank"}, and just updated the list of languages I want to have syntax highlighting for.

```lua
--~/.config/nvim/lua/plugins/treesitter.lua
--
return {
    "nvim-treesitter/nvim-treesitter", build = ":TSUpdate",
    event = { "BufReadPost", "BufNewFile" },
    cmd = { "TSInstall", "TSBufEnable", "TSBufDisable", "TSModuleInfo" }
    }
```


```lua
--~/.config/nvim/init.lua
--
require("nvim-treesitter.configs").setup({
    ensure_installed = {"bash", "lua", "python", "xml", "json"},
    sync_install = false,
    highlight = { enable = true},
    indent = {enable = true},
    })
```

### Catppuccin

At this point there should be functional syntax highlighting, however the colour scheme is a bit flat.


Just note that Lazy may not render using the Catppuccin colour scheme when installing/updating plugins.

```lua
--~/.config/nvim/init.lua
--
vim.cmd.colorscheme "catppuccin"
```

### Telescope

Provided ripgrep is installed, the Telescope setup is very simple. Just follow the getting started guide here [https://github.com/nvim-telescope/telescope.nvim](https://github.com/nvim-telescope/telescope.nvim){:target="_blank"} to get the necessary plugins using Lazy.

```lua
--~/.config/nvim/lua/plugins/telescope.lua
--
return {
    "nvim-telescope/telescope.nvim", tag = "0.1.8",
    dependencies = { "nvim-lua/plenary.nvim" },
    }
```

Within the init.lua file, I've added the following additional configuration for my Telescope keymaps as per the getting started guide.

```lua
--~/.config/nvim/init.lua
--
local builtin = require('telescope.builtin')
vim.keymap.set('n', '<leader>ff', builtin.find_files, {})
vim.keymap.set('n', '<leader>fg', builtin.live_grep, {})
vim.keymap.set('n', '<leader>fb', builtin.buffers, {})
vim.keymap.set('n', '<leader>fh', builtin.help_tags, {})
```

It should now be possible to find files and grep through the contents of those files in your current working directory using the above keymaps.

### Language Server

```lua
--~/.config/nvim/lua/plugins/lspconfig.lua
--
return {
    "neovim/nvim-lspconfig",
    dependencies = {
    "williamboman/mason.nvim",
    "williamboman/mason-lspconfig.nvim" },
    }
```

For auto-completion and snippets I have just used what bcampolo has put together here [https://github.com/bcampolo/nvim-starter-kit](https://github.com/bcampolo/nvim-starter-kit){:target="_blank"}

```lua
--~/.config/nvim/lua/plugins/nvim-cmp.lua
--
-- Auto-completion / Snippets
return {
  -- https://github.com/hrsh7th/nvim-cmp
  'hrsh7th/nvim-cmp',
  event = 'InsertEnter',
  dependencies = {
    -- Snippet engine & associated nvim-cmp source
    -- https://github.com/L3MON4D3/LuaSnip
    'L3MON4D3/LuaSnip',
    -- https://github.com/saadparwaiz1/cmp_luasnip
    'saadparwaiz1/cmp_luasnip',

    -- LSP completion capabilities
    -- https://github.com/hrsh7th/cmp-nvim-lsp
    'hrsh7th/cmp-nvim-lsp',

    -- Additional user-friendly snippets
    -- https://github.com/rafamadriz/friendly-snippets
    'rafamadriz/friendly-snippets',
    -- https://github.com/hrsh7th/cmp-buffer
    'hrsh7th/cmp-buffer',
    -- https://github.com/hrsh7th/cmp-path
    'hrsh7th/cmp-path',
    -- https://github.com/hrsh7th/cmp-cmdline
    'hrsh7th/cmp-cmdline',
  },
  config = function()
    local cmp = require('cmp')
    local luasnip = require('luasnip')
    require('luasnip.loaders.from_vscode').lazy_load()
    luasnip.config.setup({})

    cmp.setup({
      snippet = {
        expand = function(args)
          luasnip.lsp_expand(args.body)
        end,
      },
      completion = {
        completeopt = 'menu,menuone,noinsert',
      },
      mapping = cmp.mapping.preset.insert {
        ['<C-j>'] = cmp.mapping.select_next_item(), -- next suggestion
        ['<C-k>'] = cmp.mapping.select_prev_item(), -- previous suggestion
        ['<C-b>'] = cmp.mapping.scroll_docs(-4), -- scroll backward
        ['<C-f>'] = cmp.mapping.scroll_docs(4), -- scroll forward
        ['<C-Space>'] = cmp.mapping.complete {}, -- show completion suggestions
        ['<CR>'] = cmp.mapping.confirm {
          behavior = cmp.ConfirmBehavior.Replace,
          select = true,
        },
	-- Tab through suggestions or when a snippet is active, tab to the next argument
        ['<Tab>'] = cmp.mapping(function(fallback)
          if cmp.visible() then
            cmp.select_next_item()
          elseif luasnip.expand_or_locally_jumpable() then
            luasnip.expand_or_jump()
          else
            fallback()
          end
        end, { 'i', 's' }),
	-- Tab backwards through suggestions or when a snippet is active, tab to the next argument
        ['<S-Tab>'] = cmp.mapping(function(fallback)
          if cmp.visible() then
            cmp.select_prev_item()
          elseif luasnip.locally_jumpable(-1) then
            luasnip.jump(-1)
          else
            fallback()
          end
        end, { 'i', 's' }),
      },
      sources = cmp.config.sources({
        { name = "nvim_lsp" }, -- lsp
        { name = "luasnip" }, -- snippets
        { name = "buffer" }, -- text within current buffer
        { name = "path" }, -- file system paths
      }),
      window = {
        -- Add borders to completions popups
        completion = cmp.config.window.bordered(),
        documentation = cmp.config.window.bordered(),
      },
    })
  end,
 }
```

I then just needed to update init.lua to enable Mason and Pyright.

```lua
--~/.config/nvim/init.lua
--
require("mason").setup()
require("mason-lspconfig").setup({ ensure_installed = { "pyright" }, })

require "lspconfig".pyright.setup({
    on_attach = on_attach,
    filetypes = { "python" },
    settings = { pyright = { autoImportCompletion = true, }, }
    })
```

This should give functional type checking, auto-completion, and snippets when working in Python files.

### File Heirarchy

This is the directory structure of all the files created.

```
~/.config
└── nvim
    ├── init.lua
    └── lua
        ├── config
        │   └── lazy.lua
        └── plugins
            ├── catppuccin.lua
            ├── lspconfig.lua
            ├── nvim-cmp.lua
            ├── telescope.lua
            └── tresitter.lua
```

### Starting Fresh

If you think you have massively messed up somewhere and want to start fresh, I'd recommend removing the following three directories and all contents recursively.

~/.config/nvim
~/.local/state/nvim
~/.local/share/nvim
