# telescope-frecency.nvim

A [telescope.nvim][] extension that offers intelligent prioritization when
selecting files from your editing history.

[telescope.nvim]: https://github.com/nvim-telescope/telescope.nvim

Using an implementation of Mozilla's [Frecency algorithm][] (used in [Firefox's
address bar][]), files edited _frecently_ are given higher precedence in the
list index.

[Frecency algorithm]: https://web.archive.org/web/20210421120120/https://developer.mozilla.org/en-US/docs/Mozilla/Tech/Places/Frecency_algorithm
[Firefox's address bar]: https://support.mozilla.org/en-US/kb/address-bar-autocomplete-firefox

As the extension learns your editing habits over time, the sorting of the list
is dynamically altered to prioritize the files you're likely to need.

<img src="https://raw.githubusercontent.com/sunjon/images/master/gh_readme_telescope_frecency.png" alt="screenshot" width="800"/>

* _Scores shown in finder for demonstration purposes - disabled by default_

## Frecency: Sorting by 'frequency' _and_ 'recency'

'Frecency' is a score given to each unique file indexed in a file history
database.

A timestamp is recorded once per session when a file is first loaded into a
buffer.

The score is calculated using the age of the 10 (customizable) most recent
timestamps and the total amount of times that the file has been loaded:

### Recency values (per timestamp)

These values are customizable in `setup()`.

| Timestamp age | Value |
| -------- | ---------- |
| 4 hours  | 100        |
| 1 day    | 80         |
| 3 days   | 60         |
| 1 week   | 40         |
| 1 month  | 20         |
| 90 days  | 10         |

### Score calculation

```lua
score = frequency * recency_score / max_number_of_timestamps
```
## What about files that are neither 'frequent' _or_ 'recent' ?

Frecency naturally works best for indexed files that have been given a
reasonably high score.

New projects or rarely used files with generic names either don't get listed at
all or can be buried under results with a higher score.

Frecency tackles this with *Workspace Filters*:

<img src="https://raw.githubusercontent.com/sunjon/images/master/frecency_workspace_folders.gif" alt="screenshot" width="800"/>

The workspace filter feature enables you to select from user defined _filter
tags_ that map to a directory or collection of directories. Filters are applied
by entering `:workspace_tag:` anywhere in the query. Filter name completion is
available by pressing `<Tab>` after the first `:` character.

When a filter is applied, results are reduced to entries whose path is a
descendant of the workspace directory. The indexed results are optionally
augmented with a listing of _all_ files found in a recursive search of the
workspace directory. Non-indexed files are given a score of zero and appear
below the 'frecent' entries. When a non-indexed file is opened, it gains a
score value and is available in future 'frecent' search results.

If the active buffer (prior to the finder being launched) is attached to an LSP
server, an automatic `LSP` tag is available, which maps to the workspace
directories provided by the language server.


## Requirements

- [telescope.nvim](https://github.com/nvim-telescope/telescope.nvim) **(required)**
- [nvim-web-devicons](https://github.com/kyazdani42/nvim-web-devicons) (optional)
- [ripgrep](https://github.com/BurntSushi/ripgrep) or [fd](https://github.com/sharkdp/fd) (optional)

**NOTE:** `ripgrep` or `fd` will be used to list up workspace files. They are
extremely faster than the native Lua logic. If you don't have them, it
fallbacks to Lua code automatically. See the detail for `workspace_scan_cmd`
option.

## Installation

### [Packer.nvim](https://github.com/wbthomason/packer.nvim)

```lua
use {
  "nvim-telescope/telescope-frecency.nvim",
  config = function()
    require("telescope").load_extension "frecency"
  end,
}
```

### [Lazy.nvim](https://github.com/folke/lazy.nvim)

```lua
{
  "nvim-telescope/telescope-frecency.nvim",
  config = function()
    require("telescope").load_extension "frecency"
  end,
}
```

If no database is found when running Neovim with the plugin installed, a new
one is created and entries from `shada` `v:oldfiles` are automatically
imported.

## Usage

```vim
:Telescope frecency
```

or to map to a key:

```lua
-- Bind command
vim.keymap.set("n", "<leader><leader>", "<Cmd>Telescope frecency<CR>")
-- Bind Lua function directly
vim.keymap.set("n", "<leader><leader>", function()
  require("telescope").extensions.frecency.frecency {}
end)
```

Use a specific workspace tag:

```vim
:Telescope frecency workspace=CWD
```

or

```lua
-- Bind command
vim.keymap.set("n", "<leader><leader>", "<Cmd>Telescope frecency workspace=CWD<CR>")
-- Bind Lua function directly
vim.keymap.set("n", "<leader><leader>", function()
  require("telescope").extensions.frecency.frecency {
    workspace = "CWD",
  }
end)
```

Filter tags are applied by typing the `:tag:` name (adding surrounding colons)
in the finder query. Entering `:<Tab>` will trigger omnicompletion for
available tags.

### Dealing with upper case letters

In default, the sorter always ignores upper case letters in your input string.
But when [`'smartcase'`][smartcase] is ON and input string includes one upper case letter at
least, it matches against exact the same as you input.

| input string | `'smartcase'` is ON          | `'smartcase'` is OFF        |
|--------------|-----------------------------|-----------------------------|
| `abc`        | matches `abc`, `ABC`, `aBc` | matches `abc`, `ABC`, `aBc` |
| `aBc`        | matches `aBc`               | no match                    |
| `ABC`        | matches `ABC`               | no match                    |

[smartcase]: https://neovim.io/doc/user/options.html#'smartcase'

## Configuration

See [default configuration](https://github.com/nvim-telescope/telescope.nvim#telescope-defaults) for full details on configuring Telescope.

- `auto_validate`  (default: `true`)

  If `true`, it removes stale entries count over than `db_validate_threshold`. See
  the note for [DB maintenance](#maintenance)

- `db_root` (default: `vim.fn.stdpath "data"`)

  Path to parent directory of custom database location. Defaults to
  `$XDG_DATA_HOME/nvim` if unset.

- `db_safe_mode` (default: `true`)

  If `true`, it shows confirmation dialog by `vim.ui.select()` before validating
  DB. See the note for [DB maintenance](#maintenance).

- `db_validate_threshold` (default: `10`)

  It will removes over than this count in validating DB.

- `default_workspace` (default: `nil`)

  Default workspace tag to filter by e.g. `'CWD'` to filter by default to the
  current directory. Can be overridden at query time by specifying another
  filter like `':*:'`.

- `disable_devicons` (default: `false`)

  Disable devicons (if available)

- `hide_current_buffer` (default: `false`)

  If `true`, it does not show the current buffer in candidates.

- `filter_delimiter` (default: `":"`)

  Delimiters to indicate the filter like `:CWD:`.

- `ignore_patterns`
  (default: for non-Windows → `{ "*.git/*", "*/tmp/*", "term://*" }`,
  for Windows → `{ [[*.git\*]], [[*\tmp\*]], "term://*" }`)

  Patterns in this table control which files are indexed (and subsequently
  which you'll see in the finder results).

- `matcher` (default: `"default"`)

  > ___CAUTION___<br>
  > This option is highly experimental.

  In default, it matches against candidates by the so-called “substr matcher”,
  that is, you should input characters ordered properly. If you set here with
  `"fuzzy"`, it uses [_fzy_ matcher][fzy] implemented in telescope itself, and
  combines the result with recency scores. With this, you can select candidates
  fully _fuzzily_, besides that, can select easily ones that has higher recency
  scores.

  See the discussion in https://github.com/nvim-telescope/telescope-frecency.nvim/issues/165.

  [fzy]: https://github.com/jhawthorn/fzy

- `max_timestamps` (default: `10`)

  Set the max count of timestamps DB keeps when you open files. It ignores the
  value and use `10` if you set less than or equal to `0`.

  **CAUTION** When you reduce the value of this option, it removes old
  timestamps when you open the file. It is reasonable to set this value more
  than or equal to the default value: `10`.

- `path_display` (default: `nil`)

  Overwrite the `path_display` setting in telescope.nvim itself. See
  `:h telescope.defaults.path_display` for acceptable values. This setting will
  be used by these priorities below.

  1. Option specified with the command or Lua code.
    - `:Telescope frecency path_display={"absolute"}`.
    - or `:lua require("telescope").extensions.frecency.frecency { path_display = { "absolute" } }`.
  2. `opts.extensions.frecency.path_display` in setup.
  3. `opts.defaults.path_display` in setup.

     ```lua
     require("telescope").setup {
       defaults = {
         -- This has the 3rd precedence.
         path_display = { "absolute" },
       },
       extensions = {
         frecency = {
           -- This has the 2nd precedence.
           path_display = { "shorten" },
         },
       },
     }
     ```

- `recency_values` (default: see below)

  Set weighting factors for calculating “frecency”. This option does not affect
  values already recorded in DB. So you can change these values without spoiling
  data.

  ```lua
  -- default values
  recency_values = {
    { age = 240, value = 100 }, -- past 4 hours
    { age = 1440, value = 80 }, -- past day
    { age = 4320, value = 60 }, -- past 3 days
    { age = 10080, value = 40 }, -- past week
    { age = 43200, value = 20 }, -- past month
    { age = 129600, value = 10 }, -- past 90 days
  }
  ```

- `scoring_function` (default: see below)

  > ___CAUTION___<br>
  > This option is highly experimental.

  This will be used only when `matcher` option is `"fuzzy"`. You can customize the
  logic to adjust scores between [fzy matcher][fzy] scores and recency ones.

  ```lua
  -- the default value
  ---@param recency integer
  ---@param fzy_score number
  ---@return number
  scoring_function = function(recency, fzy_score)
    local score = (10 / (recency == 0 and 1 or recency)) - 1 / fzy_score
    -- HACK: -1 means FILTERED, so return a bit smaller one.
    return score == -1 and -1.000001 or score
  end,
  ```

  NOTE: telescope orders candidates in the ascending order. It also accepts
  negative numbers, but `-1` means the candidates should not be shown.

- `show_filter_column` (default: `true`)

  Show the path of the active filter before file paths. In default, it uses the
  tail of paths for `'LSP'` and `'CWD'` tags. You can configure this by setting
  a table for this option.

   ```lua
   -- show the tail for "LSP", "CWD" and "FOO"
   show_filter_column = { "LSP", "CWD", "FOO" }
   ```

- `show_scores` (default : `false`)

  To see the scores generated by the algorithm in the results, set this to
  `true`.

- `show_unindexed` (default: `true`)

  Determines if non-indexed files are included in workspace filter results.

- `workspace_scan_cmd` (default: `nil`)

  This option can be set values as `"LUA"|string[]|nil`. With the default
  value: `nil`, it uses these way below to make entries for workspace files.
  It tries in order until it works successfully.

  1. `rg -.g '!.git' --files`
  2. `fdfind -Htf`
  3. `fd -Htf`
  4. Native Lua code (old way)

  If you like another commands, set them to this option, like
  `workspace_scan_cmd = { "find", ".", "-type", "f" }`.

  If you prefer Native Lua code, set `workspace_scan_cmd = "LUA"`.

- `workspaces` (default: `{}`)

  This table contains mappings of `workspace_tag` -> `workspace_directory`. The
  key corresponds to the `:tag_name` used to select the filter in queries. The
  value corresponds to the top level directory by which results will be
  filtered.

### Example Configuration:

```lua
telescope.setup {
  extensions = {
    frecency = {
      db_root = "/home/my_username/path/to/db_root",
      show_scores = false,
      show_unindexed = true,
      ignore_patterns = { "*.git/*", "*/tmp/*" },
      disable_devicons = false,
      workspaces = {
        ["conf"]    = "/home/my_username/.config",
        ["data"]    = "/home/my_username/.local/share",
        ["project"] = "/home/my_username/projects",
        ["wiki"]    = "/home/my_username/wiki"
      }
    }
  },
}
```

## Note for Database

### Location

The default location for the database is `$XDG_DATA_HOME/nvim` (eg
`~/.local/share/nvim/` on linux). This can be configured with the `db_root`
config option. The filename for the database is `file_frecency.bin`.

### Maintenance

By default, frecency will prune files that no longer exist from the database.
In certain workflows, switching branches in a repository, that behaviour might
not be desired. The following configuration control this behaviour:

<dl>
<dt><code>db_safe_mode</code></dt>
<dd>When this is enabled, the user will be prompted before any entries are removed from the database.</dd>
<dt><code>auto_validate</code></dt>
<dd>When this to false, stale entries will never be automatically removed.</dd>
</dl>

The command `FrecencyValidate` can be used to clean the database when
`auto_validate` is disabled.

```vim
" clean DB
:FrecencyValidate
" clean DB without prompts to confirm
:FrecencyValidate!
```

### Delete entries

You can delete entries from DB by `FrecencyDelete` command. This command does
not remove the file itself, only from DB.

```vim
" delete the current opened file
:FrecencyDelete

" delete the supplied path
:FrecencyDelete /full/path/to/the/file
```

### Note about the compatibility for the former version.

The former version of this plugin has used SQLite3 library to store data. [#172][]
has removed the whole code for that. If you prefer the old SQLite database,
you should lock the version to [a3e818d][] with your favorite plugin manager.

[#172]: https://github.com/nvim-telescope/telescope-frecency.nvim/pull/172
[a3e818d]: https://github.com/nvim-telescope/telescope-frecency.nvim/commit/a3e818d001baad9ee2f6800d3bbc71c4275364ae

```lua
-- example for lazy.nvim
{
  "nvim-telescope/telescope-frecency.nvim",
  commit = "a3e818d001baad9ee2f6800d3bbc71c4275364ae",
}
```

## Highlight Groups

```vim
TelescopeBufferLoaded
TelescopePathSeparator
TelescopeFrecencyScores
TelescopeQueryFilter
```

TODO: describe highlight groups

## References

- [Mozilla: Frecency algorithm](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/Places/Frecency_algorithm)
