# Telescope Zoxide

An extension for [telescope.nvim](https://github.com/nvim-telescope/telescope.nvim) that allows you operate [zoxide](https://github.com/ajeetdsouza/zoxide) within Neovim.

## Requirements

[zoxide](https://github.com/ajeetdsouza/zoxide) is required to use this plugin.

## Installation

```vim
Plug 'nvim-lua/popup.nvim'
Plug 'nvim-lua/plenary.nvim'
Plug 'nvim-telescope/telescope.nvim'
Plug 'jvgrootveld/telescope-zoxide'
```

## Configuration

You can add, extend and update Telescope Zoxide config by using [Telescope's default configuration mechanism for extensions](https://github.com/nvim-telescope/telescope.nvim#telescope-setup-structure).
An example config:

```lua
-- Useful for easily creating commands
local z_utils = require("telescope._extensions.zoxide.utils")

require('telescope').setup{
  -- (other Telescope configuration...)
  extensions = {
    zoxide = {
      prompt_title = "[ Walking on the shoulders of TJ ]",
      mappings = {
        default = {
          after_action = function(selection)
            print("Update to (" .. selection.z_score .. ") " .. selection.path)
          end
        },
        ["<C-s>"] = {
          before_action = function(selection) print("before C-s") end,
          action = function(selection)
            vim.cmd.edit(selection.path)
          end
        },
        -- Opens the selected entry in a new split
        ["<C-q>"] = { action = z_utils.create_basic_command("split") },
      },
    }
  }
}
```

You can add new mappings and extend default mappings.
_(Note: The mapping with the key 'default' is the mapping invoked on pressing `<cr>`)_.
Every keymapping must have an `action` function and supports the optional functions `before_action` and `after_action`.

Tip: If the action is a telescope picker, you should also set `keepinsert = true` to open it in insert mode. Else you can't directly type into the next telescope picker.

All action functions are called with the current `selection` object as parameter which contains the selected path and Zoxide score.

Tip: Make use of the supplied `z_utils.create_basic_command` helper function to easily invoke a vim command for the selected path.

## Loading the extension

You can then load the extension by adding the following after your call to telescope's own `setup()` function:

```lua
require("telescope").load_extension('zoxide')
```

Loading the extension will allow you to use the following functionality:

### List

With Telescope command:

```vim
:Telescope zoxide list
```

In Lua:

```lua
require("telescope").extensions.zoxide.list({picker_opts})
```

You can also bind the function to a key:

```lua
vim.keymap.set("n", "<leader>cd", require("telescope").extensions.zoxide.list)
```

## Full example

```lua
local t = require("telescope")
local z_utils = require("telescope._extensions.zoxide.utils")

-- Configure the extension
t.setup({
  extensions = {
    zoxide = {
      prompt_title = "[ Walking on the shoulders of TJ ]",
      mappings = {
        default = {
          after_action = function(selection)
            print("Update to (" .. selection.z_score .. ") " .. selection.path)
          end
        },
        ["<C-s>"] = {
          before_action = function(selection) print("before C-s") end,
          action = function(selection)
            vim.cmd.edit(selection.path)
          end
        },
        ["<C-q>"] = { action = z_utils.create_basic_command("split") },
      },
    },
  },
})

-- Load the extension
t.load_extension('zoxide')

-- Add a mapping
vim.keymap.set("n", "<leader>cd", t.extensions.zoxide.list)
```

## Default config

```lua
{
  prompt_title = "[ Zoxide List ]",

  -- Zoxide list command with score
  list_command = "zoxide query -ls",
  mappings = {
    default = {
      action = function(selection)
        vim.cmd.cd(selection.path)
      end,
      after_action = function(selection)
        vim.notify("Directory changed to " .. selection.path)
      end,
    },
    ["<C-s>"] = { action = z_utils.create_basic_command("split") },
    ["<C-v>"] = { action = z_utils.create_basic_command("vsplit") },
    ["<C-e>"] = { action = z_utils.create_basic_command("edit") },
    ["<C-f>"] = {
      keepinsert = true,
      action = function(selection)
        builtin.find_files({ cwd = selection.path })
      end,
    },
    ["<C-t>"] = {
      action = function(selection)
        vim.cmd.tcd(selection.path)
      end,
    },
  },

  show_score = true,
  -- See `:help telescope.defaults.path_display`
  path_display = function(opts, path)
    local transformed_path = vim.trim(path)
    -- Replace home with ~
    local home = (vim.uv or vim.loop).os_homedir()
    transformed_path = home and transformed_path:gsub("^" .. vim.pesc(home), "~") or transformed_path
    -- Truncate
    local calc_result_length = function(truncate_len)
      local status = get_status(vim.api.nvim_get_current_buf())
      -- Compatibility with telescope.nvim 0.1.4
      local results_win = vim.tbl_get(status, "layout", "results", "winid") or status.results_win
      local len = vim.api.nvim_win_get_width(results_win) - status.picker.selection_caret:len() - 2
      return type(truncate_len) == "number" and len - truncate_len or len
    end
    local truncate_len = nil
    if opts.__length == nil then
      opts.__length = calc_result_length(truncate_len)
    end
    if opts.__prefix == nil then
      opts.__prefix = 0
    end
    transformed_path = truncate(transformed_path, opts.__length - opts.__prefix, nil, -1)
    -- Dim parent directories
    local tail = utils.path_tail(path)
    local path_style = {
      { { 0, #transformed_path - #tail }, "Comment" },
    }
    return transformed_path, path_style
  end,
}
```

## Default mappings

| Action  | Description                                          | Command executed                                 |
| ------- | ---------------------------------------------------- | ------------------------------------------------ |
| `<CR>`  | Change current directory to selection                | `cd <path>`                                      |
| `<C-t>` | Change current tab's directory to selection          | `tcd <path>`                                     |
| `<C-s>` | Open selection in a split                            | `split <path>`                                   |
| `<C-v>` | Open selection in a vertical split                   | `vsplit <path>`                                  |
| `<C-e>` | Open selection in current window                     | `edit <path>`                                    |
| `<C-f>` | Open selection in telescope's `builtin.find_files`   | `builtin.find_files({ cwd = selection.path })`   |

## Extensions

### Open Selection in Telescope File Browser

This action requires installing the [Telescope file browser extension](https://github.com/nvim-telescope/telescope-file-browser.nvim). You can register this mapping by adding the following to your config:

```lua
{
  mappings = {
    ["<C-b>"] = {
      keepinsert = true,
      action = function(selection)
        require("telescope").extensions.file_browser.file_browser({ cwd = selection.path })
      end
    },
  }
}
```
