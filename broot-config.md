# broot 設定

現在使用しているbrootの設定一覧。実体は `~/DotFiles/broot/` で管理し、`~/.config/broot/` からシンボリックリンクで参照している。

## ファイル構成

```
~/.config/broot/
├── conf.toml          -> ~/DotFiles/broot/conf.toml        # メイン設定
├── verbs.toml         -> ~/DotFiles/broot/verbs.toml       # verbs（コマンド・キーバインド）定義
└── skins/
    ├── dark-blue.toml # ダークターミナル用スキン
    └── white.toml     # ライトターミナル用スキン
```

スキンは `conf.toml` の `imports` でターミナルのluma（明暗）に応じて自動切り替えされる。

## conf.toml

```toml
###############################################################
# This configuration file lets you define new commands,
# change shortcuts, set colors, and tune behavior.
#
# This file is TOML.
###############################################################

show_selection_mark = true
icon_theme = "nerdfont"
content_search_max_file_size = "10MB"
enable_kitty_keyboard = false

lines_before_match_in_preview = 1
lines_after_match_in_preview = 1

imports = [
    "verbs.toml",
    { luma = ["dark", "unknown"], file = "skins/dark-blue.toml" },
    { luma = ["light"], file = "skins/white.toml" },
]

[[preview_transformers]]
input_extensions = ["pdf"]
output_extension = "png"
mode = "image"
command = [ "mutool", "draw", "-w", "1000", "-o", "{output-path}", "{input-path}" ]

[[preview_transformers]]
input_extensions = ["xls", "xlsx", "doc", "docx", "ppt", "pptx", "ods", "odt", "odp"]
output_extension = "png"
mode = "image"
command = [
    "/Applications/LibreOffice.app/Contents/MacOS/soffice", "--headless",
    "--convert-to", "png",
    "--outdir", "{output-dir}",
    "{input-path}"
]

[[preview_transformers]]
input_extensions = ["json"]
output_extension = "json"
mode = "text"
command = [ "jq" ]

[special-paths]
"/media" = { list = "never", sum = "never" }
"~/.config" = { show = "always" }
"trav" = { show = "always", list = "always", sum = "never" }
```

### ポイント

- **icon_theme = "nerdfont"**: Nerd Fontのアイコン表示
- **preview_transformers**: PDFは`mutool`、Office系ファイルはLibreOfficeでPNGに変換して画像プレビュー、JSONは`jq`で整形
- **special-paths**: `~/.config` と `trav` は常に表示

## verbs.toml

```toml
###############################################################
# Verbs definitions for broot.
#
# Format: TOML
###############################################################

[[verbs]]
invocation = "create {subpath}"
execution = "$EDITOR {directory}/{subpath}"
leave_broot = false

[[verbs]]
invocation = "touch {subpath}"
execution = "touch {directory}/{subpath}"
leave_broot = false

[[verbs]]
invocation = "git_diff"
shortcut = "gd"
leave_broot = false
execution = "git difftool -y {file}"

[[verbs]]
invocation = "backup {version}"
# key = "ctrl-b"
leave_broot = false
auto_exec = false
execution = "cp -r {file} {parent}/{file-stem}-{version}{file-dot-extension}"

[[verbs]]
invocation = "terminal"
# key = "ctrl-t"
execution = "$SHELL"
set_working_dir = true
leave_broot = false

[[verbs]]
invocation = "home"
shortcut = "h"
internal = ":focus ~"

[[verbs]]
key = "ctrl-p"
internal = ":line_up"

[[verbs]]
key = "ctrl-n"
internal = ":line_down"

[[verbs]]
key = "ctrl-a"
internal = ":toggle_hidden"

[[verbs]]
key = "ctrl-i"
internal = ":toggle_ignore"

[[verbs]]
key = "ctrl-o"
internal = ":toggle_preview"

[[verbs]]
key = "ctrl-l"
internal = ":panel_right"

[[verbs]]
key = "ctrl-h"
internal = ":panel_left_no_open"

[[verbs]]
key = "ctrl-f"
internal = ":back"

[[verbs]]
key = "ctrl-x"
internal = ":toggle_perm"

[[verbs]]
key = "ctrl-b"
internal = ":root_up"

[[verbs]]
invocation = "yoink"
shortcut = "yo"
apply_to = "file"
external = "open -a Yoink {file}"
leave_broot = false

[[verbs]]
invocation = "vi"
shortcut = "vi"
apply_to = "any"
external = "nvr --remote-tab {file}"
leave_broot = false

[[verbs]]
invocation = "Open_file_in_VSCode"
shortcut = "v"
apply_to = "any"
external = "code {file}"
leave_broot = false

[[verbs]]
invocation = "Change_Directory_and_Open_in_VSCode"
shortcut = "vs"
apply_to = "any"
external = "cd {directory} && code -n ."
from_shell = true
leave_broot = true

[[verbs]]
invocation = "Open_file_in_Cursor"
shortcut = "c"
apply_to = "any"
external = "cursor {file}"
leave_broot = false

[[verbs]]
invocation = "Change_Directory_and_Open_in_Cursor"
shortcut = "cs"
apply_to = "any"
external = "cd {directory} && cursor ."
from_shell = true
leave_broot = true

[[verbs]]
invocation = "Open_Programing_Directory"
shortcut = "pr"
apply_to = "any"
internal = ":focus $HOME/Library/Mobile Documents/com~apple~CloudDocs/Programming/"
leave_broot = false

[[verbs]]
invocation = "Open_Code_Directory"
shortcut = "co"
apply_to = "any"
internal = ":focus $HOME/Code/"
leave_broot = false

[[verbs]]
invocation = "Open_Obsidian_Main_Directory"
shortcut = "ob"
apply_to = "any"
internal = ":focus $HOME/Library/Mobile Documents/iCloud~md~obsidian/Documents/Main"
leave_broot = false

[[verbs]]
invocation = "Copy_File_Path"
shortcut = "path"
apply_to = "any"
internal = ":copy_path"
leave_broot = false

[[verbs]]
invocation = "Open_in_Finder"
shortcut = "op"
apply_to = "any"
external = "open {file}"
leave_broot = false
```

### 主要なverbs一覧

| ショートカット | キー | 動作 |
|---|---|---|
| `h` | — | ホームに移動 |
| `gd` | — | `git difftool` で差分表示 |
| `yo` | — | Yoinkに送る |
| `vi` | — | Neovim（nvr）で開く |
| `v` | — | VS Codeでファイルを開く |
| `vs` | — | ディレクトリをVS Codeで開く |
| `c` | — | Cursorでファイルを開く |
| `cs` | — | ディレクトリをCursorで開く |
| `pr` | — | Programmingディレクトリ（iCloud）に移動 |
| `co` | — | ~/Code に移動 |
| `ob` | — | Obsidianメインvaultに移動 |
| `path` | — | ファイルパスをコピー |
| `op` | — | Finderで開く |
| — | `ctrl-p` / `ctrl-n` | 上下移動（Emacs風） |
| — | `ctrl-a` | 隠しファイル切替 |
| — | `ctrl-i` | gitignore切替 |
| — | `ctrl-o` | プレビュー切替 |
| — | `ctrl-l` / `ctrl-h` | パネル操作 |
| — | `ctrl-f` | 戻る |
| — | `ctrl-x` | パーミッション表示切替 |
| — | `ctrl-b` | 親ツリーへ |

## skins/dark-blue.toml（ダーク用）

```toml
###############################################################
# A skin for a terminal with a dark background
#
# This file is TOML.
###############################################################

syntax_theme = "MochaDark"

[skin]
default = "gray(22) none  / gray(20) none"
tree = "gray(8) None  / gray(4) None"
parent = "gray(18) None  / gray(13) None"
file = "gray(22) None  / gray(15) None"
directory = "ansi(110) None bold / ansi(110) None"
exe = "Cyan None"
link = "Magenta None"
pruning = "gray(12) None Italic"
perm__ = "gray(5) None"
perm_r = "ansi(94) None"
perm_w = "ansi(132) None"
perm_x = "ansi(65) None"
owner = "ansi(138) None"
group = "ansi(131) None"
count = "ansi(138) gray(4)"
dates = "ansi(66) None"
sparse = "ansi(214) None"
content_extract = "ansi(29) None"
content_match = "ansi(34) None"
device_id_major = "ansi(138) None"
device_id_sep = "ansi(102) None"
device_id_minor = "ansi(138) None"
git_branch = "ansi(178) None"
git_insertions = "ansi(28) None"
git_deletions = "ansi(160) None"
git_status_current = "gray(5) None"
git_status_modified = "ansi(28) None"
git_status_new = "ansi(94) None bold"
git_status_ignored = "gray(17) None"
git_status_conflicted = "ansi(88) None"
git_status_other = "ansi(88) None"
selected_line = "None gray(6)  / None gray(4)"
char_match = "Green None"
file_error = "Red None"
flag_label = "gray(15) gray(2)"
flag_value = "ansi(178) gray(2) bold"
input = "White gray(2)  / gray(15) None"
status_error = "gray(22) ansi(124)"
status_job = "ansi(220) gray(5)"
status_normal = "gray(20) gray(4)  / gray(2) gray(2)"
status_italic = "ansi(178) gray(4)  / gray(2) gray(2)"
status_bold = "ansi(178) gray(4) bold / gray(2) gray(2)"
status_code = "ansi(229) gray(4)  / gray(2) gray(2)"
status_ellipsis = "gray(19) gray(1)  / gray(2) gray(2)"
purpose_normal = "gray(20) gray(2)"
purpose_italic = "ansi(178) gray(2)"
purpose_bold = "ansi(178) gray(2) bold"
purpose_ellipsis = "gray(20) gray(2)"
scrollbar_track = "gray(7) None  / gray(4) None"
scrollbar_thumb = "gray(22) None  / gray(14) None"
help_paragraph = "gray(20) None"
help_bold = "ansi(178) None bold"
help_italic = "ansi(229) None"
help_code = "gray(21) gray(3)"
help_headers = "ansi(178) None"
help_table_border = "ansi(239) None"
preview = "gray(20) gray(1)  / gray(18) gray(2)"
preview_title = "gray(23) gray(2)  / gray(21) gray(2)"
preview_line_number = "gray(12) gray(3)"
preview_separator = "gray(5) None"
preview_match = "None ansi(29)"
hex_null = "gray(8) None"
hex_ascii_graphic = "gray(18) None"
hex_ascii_whitespace = "ansi(143) None"
hex_ascii_other = "ansi(215) None"
hex_non_ascii = "ansi(167) None"
staging_area_title = "gray(22) gray(2)  / gray(20) gray(3)"
mode_command_mark = "gray(5) ansi(204) bold"
good_to_bad_0 = "ansi(28)"
good_to_bad_1 = "ansi(29)"
good_to_bad_2 = "ansi(29)"
good_to_bad_3 = "ansi(29)"
good_to_bad_4 = "ansi(29)"
good_to_bad_5 = "ansi(100)"
good_to_bad_6 = "ansi(136)"
good_to_bad_7 = "ansi(172)"
good_to_bad_8 = "ansi(166)"
good_to_bad_9 = "ansi(196)"
```

## skins/white.toml（ライト用）

```toml
###############################################################
# A skin for a terminal with a white background
#
# This file is TOML.
###############################################################

syntax_theme = "base16-ocean.light"

[skin]
default = "gray(1) None"
tree = "gray(7) None / gray(18) None"
file = "gray(3) None / gray(8) None"
directory = "ansi(25) None Bold / ansi(25) None"
exe = "ansi(130) None"
link = "Magenta None"
pruning = "gray(12) None Italic"
perm__ = "gray(5) None"
perm_r = "ansi(94) None"
perm_w = "ansi(132) None"
perm_x = "ansi(65) None"
owner = "ansi(138) None"
group = "ansi(131) None"
dates = "ansi(66) None"
sparse = "ansi(214) None"
git_branch = "ansi(229) None"
git_insertions = "ansi(28) None"
git_deletions = "ansi(160) None"
git_status_current = "gray(5) None"
git_status_modified = "ansi(28) None"
git_status_new = "ansi(94) None Bold"
git_status_ignored = "gray(17) None"
git_status_conflicted = "ansi(88) None"
git_status_other = "ansi(88) None"
selected_line = "None gray(19) / None gray(21)"
char_match = "ansi(22) None"
file_error = "Red None"
flag_label = "gray(9) None"
flag_value = "ansi(166) None Bold"
input = "gray(1) None / gray(4) gray(20)"
status_error = "gray(22) ansi(124)"
status_normal = "gray(2) gray(20)"
status_job = "ansi(220) gray(5)"
status_italic = "ansi(166) gray(20)"
status_bold = "ansi(166) gray(20)"
status_code = "ansi(17) gray(20)"
status_ellipsis = "gray(19) gray(15)"
purpose_normal = "gray(20) gray(2)"
purpose_italic = "ansi(178) gray(2)"
purpose_bold = "ansi(178) gray(2) Bold"
purpose_ellipsis = "gray(20) gray(2)"
scrollbar_track = "gray(20) none"
scrollbar_thumb = "ansi(238) none"
help_paragraph = "gray(2) none"
help_bold = "ansi(202) none bold"
help_italic = "ansi(202) none italic"
help_code = "gray(5) gray(22)"
help_headers = "ansi(202) none"
help_table_border = "ansi(239) None"
preview_title = "gray(3) None / gray(5) None"
preview = "gray(5) gray(23) / gray(7) gray(23)"
preview_line_number = "gray(6) gray(20)"
preview_separator = "gray(7) None / gray(18) None"
preview_match = "None ansi(29) Underlined"
hex_null = "gray(15) None"
hex_ascii_graphic = "gray(2) None"
hex_ascii_whitespace = "ansi(143) None"
hex_ascii_other = "ansi(215) None"
hex_non_ascii = "ansi(167) None"
staging_area_title = "gray(8) None / gray(13) None"
mode_command_mark = "gray(15) ansi(204) Bold"
good_to_bad_0 = "ansi(28)"
good_to_bad_1 = "ansi(29)"
good_to_bad_2 = "ansi(29)"
good_to_bad_3 = "ansi(29)"
good_to_bad_4 = "ansi(29)"
good_to_bad_5 = "ansi(100)"
good_to_bad_6 = "ansi(136)"
good_to_bad_7 = "ansi(172)"
good_to_bad_8 = "ansi(166)"
good_to_bad_9 = "ansi(196)"
```

## 備考

- 移行前の旧形式設定（hjson）のバックアップが `conf.hjson.backup.*` / `verbs.hjson.backup.*` として `~/.config/broot/` に残っている
- 同期元: `~/DotFiles/broot/`
- コードブロック内の `$HOME` は実環境のホームディレクトリパスのプレースホルダ（公開用に置換）。実際の設定ファイルにはフルパスが書かれている
