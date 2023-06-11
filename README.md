jfind.nvim
==========
A plugin for using jfind as a neovim fuzzy finder.

Dependencies
------------
 - [jfind](https://github.com/jake-stewart/jfind) (Required)
 - [fdfind](https://github.com/sharkdp/fd) (Recommended as a faster alternative to `find`)

You can install jfind with this one liner. You will need git, cmake, and make.
```
git clone https://github.com/jake-stewart/jfind && cd jfind && cmake -S . -B build && cd build && sudo make install
```

**If you are migrating to 1.0 from an earlier version, make sure to recompile jfind!**

Installation
------------

#### [lazy.nvim](https://github.com/folke/lazy.nvim)
```lua
{ "jake-stewart/jfind.nvim", branch = "1.0" }
```

#### [vim-plug](https://github.com/junegunn/vim-plug)
```vim
Plug "jake-stewart/jfind.nvim", { "branch": "1.0" }
```

#### [dein.vim](https://github.com/Shougo/dein.vim)
```vim
call dein#add("jake-stewart/jfind.nvim", { "rev": "1.0" })
```

#### [packer.nvim](wbthomason/packer.nvim)
```lua
use {
  "jake-stewart/jfind.nvim", branch = "1.0"
}
```


Example Config
--------------

```lua
local jfind = require("jfind")
local key = require("jfind.key")

jfind.setup({
    exclude = {
        ".git",
        ".idea",
        ".vscode",
        ".sass-cache",
        ".class",
        "__pycache__",
        "node_modules",
        "target",
        "build",
        "tmp",
        "assets",
        "dist",
        "public",
        "*.iml",
        "*.meta"
    },
    border = "rounded",
    tmux = true,
});

-- fuzzy file search can be started simply with
vim.keymap.set("n", "<c-f>", jfind.findFile)

-- or you can provide more customization
-- for more information, read the "Lua Jfind Interface" section
vim.keymap.set("n", "<c-f>", function()
    jfind.findFile({
        formatPaths = true,
        callback = {
            [key.DEFAULT] = vim.cmd.edit,
            [key.CTRL_S] = vim.cmd.split,
            [key.CTRL_V] = vim.cmd.vsplit,
        }
    })
end)
```

### Setup Options
|Option|Description
|-|-|
|`tmux`|a boolean of whether a tmux window is preferred over a neovim window. If tmux is not active, then this value is ignored. Default is `false`.|
|`exclude`|a list of strings of files/directories that should be ignored. Entries can contain wildcard matching (e.g. `*.png`). Default is an empty list.|
|`border`|The style of the border when not fullscreen. The default is `"single"`. Possible values include: <br>- `"none"`: No border.<br>- `"single"`: A single line box.<br>- `"double"`: A double line box.<br>- `"rounded"`: Like "single", but with rounded corners.<br>- `"solid"`: Adds padding by a single whitespace cell.<br>- `"shadow"`: A drop shadow effect by blending with the background.<br>- Or an array for a custom border. See `:h nvim_open_win` for details.|
|`maxWidth`|An integer of how large in width the jfind can be as fullscreen until it becomes a popup window. default is `120`.|
|`maxHeight`|An integer of how large in height the jfind can be as fullscreen until it becomes a popup window. default is `28`.|


Lua Jfind Interface
-------------------
This section is useful if you want to create your own fuzzy finders using
jfind, or if you want to understand the configuration better.

### The jfind function
This plugin provides the `jfind()` function, which can be accessed via
`require("jfind").jfind`. This function opens a tmux or nvim popup based on
user configuration, performs fuzzy finding, and calls a provided callback with
the result if there is one.

### Fuzzy finding script output
Below is an example usage of `jfind()`. It takes a script, which in this case
is the `seq` command. It also provides an argument to the `seq` command.
Upon completion, the result of the fuzzy finding will be printed using the
provided `print` callback. The `seq` command could just as easily have been a
path to a shell script, as is the case for `findFile()`.

```lua
local jfind = require("jfind")

jfind.jfind({
    script = "seq",
    args = {100},
    callback = print
})
```

### Fuzzy finding a list of strings
Instead of a program/script, you can provide a list of strings with the input
option. This is useful for generating the input data in lua instead of a shell
script, since a shell script does not have access to neovim state.

```lua
local jfind = require("jfind")

jfind.jfind({
    input = {"one", "two", "three", "four"},
    callback = print
})
```

### Multiple keybindings
The callback option is either a function or a table. You can provide a table
if you want different actions for different keybinds. For example, you may want
to vertically split when pressing `<c-v>` on an item. Below is an example of
having multiple keybindings.

```lua
local jfind = require("jfind")
local key = require("jfind.key")

jfind.jfind({
    script = "ls",
    callback = {
        [key.DEFAULT] = vim.cmd.edit,
        [key.CTRL_V] = vim.cmd.vsplit,
        [key.CTRL_S] = vim.cmd.split
    }
})
```

the `key.DEFAULT` applies to the user hitting enter or double clicking on an
item, unless overridden.

### Hints
You may have noticed that the builtin `findFile()` accepts an option called
`formatPaths`. When this option is true, the jfind window has two columns,
where the one on the right shows the full path, but is not searchable. These
are called hints. They are useful for separating what the user is searching for
from the result we want.

For instance, when I am searching for a path, I do not want to search the full
`~/projects/foo/bar/baz/item/item.java`, I just want to search the final
`item/item.java`. In this case, the `item/item.java` would be the search item,
and `~/projects/foo/bar/baz/item/item.java` would be the hint. We can then use
the hint when actually editing the file, since trying to edit `item/item.java`
is missing its hierarchy.

```lua
local jfind = require("jfind")

jfind.jfind({
    input = {"item one", "hint one", "item two", "hint two"},
    hints = true,
    callback = function(result, hint)
        print("result: " .. result, "hint: " .. hint)
    end
})
```

### Wrapping the callbacks
Sometimes it may be useful to wrap each callback in a function. This can save
needing the same boilerplate for every callback. The following example is using 
a snippet from the "Fuzzy finding open buffers" example.

```lua
local jfind = require("jfind")
local key = require("jfind.key")

-- without wrapper
jfind.jfind({
    input = get_buffers(),
    hints = true,
    callback = {
        [key.DEFAULT] = function(_, path) vim.cmd.edit(path) end,
        [key.CTRL_S] = function(_, path) vim.cmd.split(path) end,
        [key.CTRL_V] = function(_, path) vim.cmd.vsplit(path) end,
    }
})

-- with wrapper
jfind.jfind({
    input = get_buffers(),
    hints = true,
    callbackWrapper = function(callback, _, path)
        callback(path)
    end,
    callback = {
        [key.DEFAULT] = vim.cmd.edit,
        [key.CTRL_S] = vim.cmd.split,
        [key.CTRL_V] = vim.cmd.vsplit,
    }
})

```


### Example: Fuzzy finding open buffers
This example combines it all together to create a fuzzy finder for open
buffers.

```lua
local jfind = require("jfind")
local key = require("jfind.key")

local function get_buffers()
    local buffers = {}
    for i, buf_hndl in ipairs(vim.api.nvim_list_bufs()) do
        if vim.api.nvim_buf_is_loaded(buf_hndl) then
            local path = vim.api.nvim_buf_get_name(buf_hndl)
            if path ~= nil and path ~= "" then
                buffers[i * 2 - 1] = jfind.formatPath(path)
                buffers[i * 2] = path
            end
        end
    end
    return buffers
end

jfind.jfind({
    input = get_buffers(),
    hints = true,
    callbackWrapper = function(callback, _, path)
        callback(path)
    end,
    callback = {
        [key.DEFAULT] = vim.cmd.edit,
        [key.CTRL_S] = vim.cmd.split,
        [key.CTRL_V] = vim.cmd.vsplit,
    }
})
```
