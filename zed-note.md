# 构建

对 `script/bundle-mac` 做如下修改:

``` sh
zj@a:~/go/src/github.com/zed-industries/zed$ git diff script/bundle-mac
diff --git a/script/bundle-mac b/script/bundle-mac
index bc95e1dd6a..a20fd09268 100755
--- a/script/bundle-mac
+++ b/script/bundle-mac
@@ -1,5 +1,5 @@
 #!/usr/bin/env bash
-
+set -x
 set -euo pipefail
 source script/lib/blob-store.sh

@@ -232,7 +231,7 @@ function sign_app_binaries() {
     fi

     echo "Downloading git binary"
-    download_git "${architecture}" "${app_path}/Contents/MacOS/git"
+    #download_git "${architecture}" "${app_path}/Contents/MacOS/git"

     # Note: The app identifier for our development builds is the same as the app identifier for nightly.
     cp crates/zed/contents/$channel/embedded.provisionprofile "${app_path}/Contents/"
```

构建并安装到 /Applications 目录:

```sh
$ enable_socks_proxy
$ ./script/bundle-mac -oidl
```

修改文件  `crates/zed/RELEASE_CHANNEL`，可选值为：
- dev
- nightly
- preview
- stable

如果是 dev 模式，则每次连接远程服务器时都会[重新构建 remote_server](https://github.com/zed-industries/zed/blob/21b58643fadb5d06d5896ab1b41be25b95d86875/crates/recent_projects/src/ssh_connections.rs#L504)，
比较耗时。其它都是 server 从 zed.dev 下载已经构建好的 remote server binary。

通过修改代码，实现如果 remote_server.gz 文件不存在，则重新构建，否则就复用本地已经构建和打包好的 gz 文件。

    // https://github.com/zed-industries/zed/blob/21b58643fadb5d06d5896ab1b41be25b95d86875/crates/recent_projects/src/ssh_connections.rs#L769-L790
    if platform.arch == std::env::consts::ARCH && platform.os == std::env::consts::OS {
        let path = std::env::current_dir()?.join("target/remote_server/debug/remote_server.gz");
        if !path.exists() { // 加一个 remote_server.gz 是否存在的判断。
            self.update_status(Some("Building remote server binary from source"), cx);
            log::info!("building remote server binary from source");
            run_cmd(Command::new("cargo").args([
                "build",
                "--package",
                "remote_server",
                "--features",
                "debug-embed",
                "--target-dir",
                "target/remote_server",
            ]))
            .await?;

            self.update_status(Some("Compressing binary"), cx);

            run_cmd(Command::new("gzip").args([
                "-9",
                "-f",
                "target/remote_server/debug/remote_server",
            ]))
            .await?;
        }
        return Ok(Some((path, version)));

``` sh
alizj@lima-dev2:/Users/alizj/.config/zed$ ~/.zed_server/zed-remote-server-dev-linux-aarch64 version
0.160.0
```

zed 的 ssh_session.rs 的 [update_server_binary_if_needed() 函数](https://github.com/zed-industries/zed/blob/f919fa92de1d73c492282084b96249b492732f83/crates/remote/src/ssh_session.rs#L1735) 会通过执行 server 上的 zed-remote-server 的 version 子命令来获得 server 语义版本，如果执行成功，则检查当前本地 zed 版本是否比 server 语义版本低：
1. 如果低则提示升级本地 zed 版本；
2. 如果 server 语义版本比本地低，则会自动上传本地 zed-remote-server binary 到 server（dev 模式或设置了 upload_binary_over_ssh=true), 否则 server 自己从 zed.dev 下载指定版本的 binary。

也可以手动上传 server binary 到 ~/.zed_server/zed-remote-server-xx-linux-xx，然后启动：
ZED_USE_CACHED_REMOTE_SERVER=1 zed ssh://blah
这时 zed 会先检查 server 是否存在 remote server binary，如果存在则继续使用。

在 zed server 运行过程中，会自动[从网络下载 lsp language 并安装](https://github.com/zed-industries/zed/blob/f919fa92de1d73c492282084b96249b492732f83/crates/languages/src/rust.rs#L100)：

``` sh
alizj@lima-dev2:~/.local/share/zed/logs$ grep gopls *
server-dev-workspace-189.log:{"level":3,"module_path":"project::lsp_store","file":"crates/project/src/lsp_store.rs","line":5529,"message":"(remote server) attempting to start language server \"gopls\", path: \"/Users/alizj/go/src/github.com/kubernetes\", id: 0"}
server-dev-workspace-189.log:{"level":3,"module_path":"language","file":"/Users/alizj/go/src/github.com/zed-industries/zed/crates/language/src/language.rs","line":548,"message":"(remote server) fetching latest version of language server \"gopls\""}
server-dev-workspace-189.log:{"level":3,"module_path":"language","file":"/Users/alizj/go/src/github.com/zed-industries/zed/crates/language/src/language.rs","line":555,"message":"(remote server) downloading language server \"gopls\""}
server-dev-workspace-189.log:{"level":3,"module_path":"project::lsp_store","file":"crates/project/src/lsp_store.rs","line":5483,"message":"(remote server) using project environment for language server LanguageServerName(\"gopls\")"}
server-dev-workspace-189.log:{"level":3,"module_path":"lsp","file":"crates/lsp/src/lsp.rs","line":283,"message":"(remote server) starting language server process. binary path: \"/home/alizj.linux/.local/share/zed/languages/gopls/gopls_0.16.2\", working directory: \"/Users/alizj/go/src/github.com/kubernetes\", args: [\"-mode=stdio\"]"}
```

# 启动

窗口右上角显示资源用量和 GPU FPS 信息（有 bug，可能导致段错误）：

``` sh
$ MTL_HUD_ENABLED=1 /Applications/Zed.app/Contents/MacOS/zed
```

DEBUG 启动模式:

``` sh
# 先切换到 zed 源码目录(有些命令, 如 ssh remote 会在源码目录编译一些二进制)
$ pwd
/Users/alizj/go/src/github.com/zed-industries/zed
$ RUST_LOG=debug /Applications/Zed\ Dev.app/Contents/MacOS/zed
```

*zed cli* : 可以通过 Zed 菜单 “Install CLI” 来安装 zed 命令行工具命令 zed：

``` sh
zj@a:~$ which zed
/usr/local/bin/zed
zj@a:~$ ls -l /usr/local/bin/zed
lrwxr-xr-x 1 root 44 10 26 14:54 /usr/local/bin/zed -> '/Applications/Zed Dev.app/Contents/MacOS/cli'*
```

使用 zed cli：

```sh
$ zed ~/emacs/minimal.el # 在当前 workspae 中打开文件，但是不将文件添加到 workspace
$ zed -a ~/emacs/minimal.el # 在当前 workspace 中打开文件，同时将文件添加到 workspace
$ zed -a ~/emacs # 将目录添加到 workspace
```

zed 获得环境变量的两种方式：
1. 命令行 zed 启动, 继承命令行环境变量;
2. 通过 dock 启动, 先切换到 HOME 目录 spawn 一个 login shell 来获得用户环境变量, 然后被所有 zed
窗口继承;

zed 打开 project 时, 会使用 direnv/editorconfig 等机制来获得项目相关的环境变量, 并被项目的 task/lsp/terminal 继承;

# 布局

一个 window 有多个 panes （通过 spit），一个 pane 有多个 items（tabs）。

pane 有自己的 tool bar 和导航 history（前进、后退）。光标在 Pane buffer 的移动位置会被记录到 Pane 导航历史记录中，可以点击左上角的左、右箭头来回到上一次、下一次的位置。通过使用导航历史快捷键，可以快捷的前后移动。

可以使用 ctrl-0/1/2 等数字快捷键快速在多个 pane tab 切换。

将 project 和 outline panel 都设置到 left dock, 便于查看。

# 编辑

zed 打开的文件对话框后，按 Command-Shift-g 可以按照文件路径来打开。

快速选择一个 block：将光标移动到 block 边界字符上，然后按 ctrl-= 来按语法选择。

搜狗输入法设置:

1. 全局默认中文标点。
2. 定义中英文标点切换按键: ctrl-. , zed 不能使用该按键。
3. 使用 ctrl 切换中英文, 它比右 shift 更容易按，而且在 zed 中是修饰键，需要和其它按键一起才生效。

光标位于 URL 上, 执行 editor::open url 命令可以快速打开该 URL.

project panel 默认对空目录进行折叠, 双击折叠的目录时会展开。

# multicusor

在编辑窗口（普通编辑窗口或搜索结果窗口），按住 alt 后点击要增加 cursor 的位置，然后就可以多光标同时编辑。

# search

搜索分为 buffer search 和 project search，支持关键字、word 和正则搜索方式，也可以忽略大小写。

buffer search 是每输入一个字符就触发的实时增量搜索, 而 project search 是输入完所有搜索字符后按 enter 后后才触发搜索。

搜索时，默认选中光标处的 symbol/word，也可以先选中内容后再搜索。

搜索时，默认选中所有匹配项，导致前后跳转时不易分清光标所在行，两个办法：
1. 开启行号来明确显示当前匹配的行；（通过命令面板开启）。
2. 在 outline pane 看当前匹配的行。
3. 焦点切换到编辑窗口（按 tab），按 `ctrl-l` 将光标滚动到窗口中心。

选中搜索框右侧的 `Select All Match` 按钮对当前选中的匹配项（默认选中所有匹配项）启用多光标编辑，
实现搜索结果的批量编辑。

搜索的结果可以在 outline panel 显示，实现快速跳转和二次过滤。

project search 的结果默认在 preview tab 中显示（标题是斜体），它是临时 buffer，在其中双击后就显示
对应位置的文件内容，不能再返回到以前的结果buffer。解决办法：双击前，先将该 preview tab pin 住或转换
为普通 tab（双击tab）。

正则搜索使用扩展正则表达式语法, 参考 [regex crate 文档](https://docs.rs/regex/latest/regex/#syntax).

# outline

outline 是基于 tree-sitter 解析的节点树，支持对编辑窗口、搜索窗口（buffer 或 project 搜索）、Reference 窗口、诊断窗口的结构化显示。包含三种类型：

- buffer outline
- project outline
- outline panel

outline panel 支持多种快捷操作（Actions），如目录的展开和合并，跳转到上一级，在 Finder 中打开文件等。

# multibuffer

project 级别的 search/reference/diagnose 打开的窗口是 multibuffer 类型。

multibuffer 中的文件位置称为片段（excerpt），支持多光标编辑和保存。

配置 `"double_click_in_multibuffer": "open"` 选项后，双击 multibuffer 片段时，在新的 tab 打开对应文件位置。

# preview tabs

在 project panel 中点击各文件时，默认使用 preview tab 显示，点击其它文件时，preview tab 会切换显
示该文件，从而避免同时打开很多个 tabs，特别适合只读查看文件的情况。

preview tabs 的 tab 标题是 _斜体显示_ ，从而可以和普通独立 tab 区分开来。

preview tabs 通过以下方式转换为普通独立 tab：

1. project pandel 中双击打开文件；
2. 使用 project_panel::OpenPermanent action 打开文件；
3. 编辑文件。
4. pin tab。
5. 双击 tab。
6. 将 tab 拖移到其它 pane。

还可以在配置文件中配置 code nav 和 file finder 使用 preview tabs。

# 按键绑定

注：使用命令 `debug: Open Key Context View` 查看当前焦点的 context，触发的按键，以及按键匹配情况。

zed 按键绑定（`/.config/zed/keymap.json`）不区分相同按键序列但不同顺序的情况，如 `ctrl-cmd-a` 和 `cmd-ctrl-a` 是相同的按键，但 zed 不提示重复的按键绑定。解决办法：使用固定的顺序来写按键，如 `ctrl-cmd-alt-shift`。

统一规划一些前缀快捷键，如 `ctrl-x`, 它们只用于前缀场景，而不单独使用，否则会导致按键响应延迟。（因为 zed 会等待一段时间来接收前缀后续的按键，当超时后，才认为是致独立绑定语义）。

zed 支持灵活的按键 remap：

- `["workspace::SendKeystrokes", "ctrl-down ctrl-down ctrl-down ctrl-down ctrl-down"]`
- `["workspace::SendKeystrokes", ": task:spawn enter Test Under Cursor enter"]`
- `["task::Spawn", { "task_name": "Example task" }]`
- `["assistant::InlineAssist",{ "prompt": "Build a snake game" }]`

不确定是否支持这个语法？

    "shift-down": [
      "editor::SelectDown",
      ["editor::MoveUpByLines", { "lines": 5 }]
    ]

自定义按键绑定覆盖缺省按键绑定，缺省绑定中未覆盖的按键继续有效。所以，如果要确保自己的按键定生效，则可能需要在多个 context 中重复设置。

不是所有 action 在所有 context 中都有效， 如果高优 context 中的按键绑定 action 无效， 则会 fallback 到低优 context 中该按键绑定的 action，以此类推直到第一个有效 action。

例如， Editor 和 Editor && mode == full 的 context 都定义了 ctrl-o 快捷键，但是后者的 excerpt 只在 multibuffer 中有效，所以 fallback 到 Editor 中的 buffer symbol：

    {
      "context": "Editor && mode == full",
      "bindings": {
        "enter": "editor::Newline",
        "ctrl-enter": "editor::OpenExcerptsSplit",
        "ctrl-o": "editor::OpenExcerptsSplit",
        "ctrl-cmd-enter": "editor::OpenExcerpts",
        "ctrl-=": "editor::ExpandExcerpts",
        "ctrl-cmd-]": "assistant::QuoteSelection",
        "ctrl-cmd-[": "assistant::InsertIntoEditor"
      }
    },

同一个上下文可以定义多次，会做 merge。在相同 Context 中，同一个快捷键绑定多次时，后续的绑定生效。

shift- 用于表示大写字母或第二按键，使用时需要注意：

- 对于使用大写字母的按键，需要包含 shift，而不能直接写大写字母，如：
    "alt-shift-r" ：OK
    "alt-R"： 错误。

- 对于使用第二按键，需要直接使用第二按键名称，而不能包含 shift，如：
    "cmd-ctrl-<": OK， "cmd-ctrl-shift-,": 错误。
    "cmd-%": OK, "cmd-shift-%": 错误，"cmd-shift-5": 错误。

- 对于非大写字母或第二按键的场景，不能使用 shift，即 zed 的 shift 不支持作为通用修饰键来使用，如：
    "ctrl-shift-,": 不对，因为 , 有第二按键 <，应该直接使用第二按键，而不需要加 shift。
    "ctrl-shift-=": 不对，因为 = 有第二按键 +，应该直接使用 "ctrl-+"。

- "ctrl-x ^" 中的 ctrl-x 是作为前缀快捷键来使用，那么 ctrl-x 不能再有单独的定义。

zed 窗口是由层次化的 UI 元素节点组成的，节点间有父子、兄弟关系，处于不同层次的上下文中。
反映到按键上，就是有优先级，嵌套越深的层次上定义的快捷键优先级越高，如 buffer 搜索输入框的层次是：

Workspace > Pane > BufferSearchBar > Editor(搜索框)

zed 从配置中加载所有按键绑定，然后用户输入对应按键绑定时，过滤 context 条件符合要求的 actions 列表，
然后根据 context 所在的 UI 节点深度，选择最深层次上定义的 action。

当没有打开的文件时，即没有 panel tab 处于 focus 时，处于 Workspace 或 Global 上下文。

如果光标焦点位于搜索框，则 BufferSearchBar > Editor context 定义的快捷键优先级最高，
然后是普通 Editor > BufferSearchBar > Pane > Workspace。

context 表达式可以使用 > 来表达直接的父子关系（父直属的子节点）匹配，如 Parent > Child，层次越深优先级越高。

context 表达式中的逻辑表达式并不表示层次关系,也没有提升优先级深度，如 `BufferSearchBar && !in_replace` 实际还是匹配焦点位于 BufferSearchBar 中搜索框（而非替换框），由于本质上还是匹配 BufferSearchBar 这一个层次，所以它们的定义顺序很重要，后续的覆盖前者，例如 keymap.json 文件中安如下顺序定义 context：

1. Editor
2. Editor && mode == single_line
3. Editor && mode == full
4. Editor && (showing_code_actions || showing_completions)
5. BufferSearchBar && !in_replace > Editor

第 1 个 Editor 由于没有限定条件，是 Editor 的通用配置，所以应该放到最前面。

第 2、3 个 Editor 都是匹配特定 mode 的 context，只会在对应的场景下生效，但它们的并没有引
入优先级更深的 cotext，本质上还是和第 1 个处于相同的深度，但是由于位于 Editor 的后面定义，
所以覆盖 Editor 中相同定义的快捷键。

按键绑定优先级：Editor 》Pane 》Workspace 》Global。

Workspace 优先级比 Editor、Pane、Term 低，但比 Global 高，用于定义全局快捷键。

类似于下面两个父子上下文，匹配的是 BufferSearchBar 中包含的下一个 single_line mode 的 Editor 输入框和替换框，所以它引入了更高优先级的嵌套深度，高于 BufferSearchBar 和普通 Editor 上下文：

- "context": "BufferSearchBar && !in_replace > Editor" // Search 输入框
- "context": "BufferSearchBar && in_replace > Editor" // InReplace 输入框

各上下文可以调用任意命令（action），但是不是所有 action 在所有 context 中都生效，当它不生效时会 fallback 到以前的 action。

gpui::actions!() 和 impl_actions!() 宏定义了各种命令面板中的命令，如：
https://github.com/zed-industries/zed/blob/92c29be74cc2ac09dfe0d71d5a1048121b6ab4c6/crates/editor/src/actions.rs#L156

keymap 的 context 除了 > 外，并不不表示层次关系，或有衔接关系，而只是用于将当前节点的 identify 与条件进行匹配。

- https://github.com/zed-industries/zed/blob/d209eab05879ddd49c4ebbb439966150f7c3b686/crates/workspace/src/workspace.rs#L4706
- https://github.com/zed-industries/zed/blob/d209eab05879ddd49c4ebbb439966150f7c3b686/crates/gpui/src/keymap.rs#L78
- https://github.com/zed-industries/zed/issues/14718
- https://github.com/zed-industries/zed/blob/92c29be74cc2ac09dfe0d71d5a1048121b6ab4c6/crates/gpui/src/keymap/context.rs#L99
- https://github.com/zed-industries/zed/blob/92c29be74cc2ac09dfe0d71d5a1048121b6ab4c6/crates/gpui/src/elements/div.rs#L550

通过查看 keymap context 的 add 和 set 方法的引用，以及 div 和 key_context 方法，可以获得所有 context 信息。

直接设置的 key context 如下：

- AssistantPanel
- ContextEditor # 是 assistant 在用的上下文。
- GiveFeedback
- menu # 位于 ContextMenu 中，ContextMenu 是可以选择列表中项目的组件。
- ChatPanel
- PasswordPrompt # ssh
- PromptLibrary
- DevServerModal
- CollabPanel
- PromptEditor # LLM inline assistant
- TabSwitcher：ctrl-tab 显示的 tab 切换
- Picker：项目文件或符号选择（列表）

[os](https://github.com/zed-industries/zed/blob/92c29be74cc2ac09dfe0d71d5a1048121b6ab4c6/crates/gpui/src/keymap/context.rs#L32)

- os == macos/linux/windows

[Editor](https://github.com/zed-industries/zed/blob/92c29be74cc2ac09dfe0d71d5a1048121b6ab4c6/crates/editor/src/editor.rs#L2106)

- mode == single_line/auto_height/full
- jupyter
- renaming # 执行标识符重命名时，提示输入新名称
- menu && showing_completions # 出现上下文补全时
- menu && showing_code_actions # 出现上下文 codeaction 时
- extension == xx # 出现 extension 时
- inline_completion && copilot_suggestion # 出现 inline 智能补全

编辑窗口或输入框（如 search、replace、filter、rename 等输入框）中的按键绑定。焦点位于编辑或输入框时有效，位于终端等窗口类型无效。优先级比 workspace 和 pane 高。

Editor 有三种模式 mode:

1. SingleLine：如 editor::rename, TextField(如 search input，form fields 等)，GotoLine，Outline Panel 的 Search/Filter 输入框，Picker 输入框，Buffer 和 Project Search Editor。
2. AutoHeight：inline assistant，terminal inline assistant，chat panel
3. Full：绝大部分编辑场景，如占据整个 pane tab 的编辑窗口；

单行输入框上下文，例如搜索框、替换框、过滤框等，可以用 Editor && mode== single_line 匹配.

编辑普通文件、编辑 assistant/multibuffer 时 mode == full。

keymap 的 context 中使用逻辑表达式来匹配特定模式的 Editor：

1. Editor && mode == full
2. Editor && mode == auto_height

[Dock](https://github.com/zed-industries/zed/blob/92c29be74cc2ac09dfe0d71d5a1048121b6ab4c6/crates/workspace/src/dock.rs#L592)

- Dock Left、Right、Bottom 三种类型

[Pane](https://github.com/zed-industries/zed/blob/d209eab05879ddd49c4ebbb439966150f7c3b686/crates/workspace/src/pane.rs#L2566)

- EmptyPane # 没有任何 active 的 item（tab）

[ProjectPanel](https://github.com/zed-industries/zed/blob/d209eab05879ddd49c4ebbb439966150f7c3b686/crates/project_panel/src/project_panel.rs#L2712)

- menu
- editing # file name 输入框获得焦点
- not_editing

[BufferSearch](https://github.com/zed-industries/zed/blob/d209eab05879ddd49c4ebbb439966150f7c3b686/crates/search/src/buffer_search.rs#L191)

- in_replace # 光标位于 repalce editor 中

[BufferSearchBar](https://github.com/zed-industries/zed/blob/92c29be74cc2ac09dfe0d71d5a1048121b6ab4c6/crates/search/src/buffer_search.rs#L191)

- in_replace # 焦点是否位于替换框中

[ProjectSearchBar](https://github.com/zed-industries/zed/blob/d209eab05879ddd49c4ebbb439966150f7c3b686/crates/search/src/project_search.rs#L1865)

- in_replace # 光标位于 repalce editor 中

[OutlinePanel](https://github.com/zed-industries/zed/blob/d209eab05879ddd49c4ebbb439966150f7c3b686/crates/outline_panel/src/outline_panel.rs#L726)

- menu
- editing # 焦点处于 Filter 输入框
- not_editing # 焦点在 Filter 输入框外

[Vim](https://github.com/zed-industries/zed/blob/d209eab05879ddd49c4ebbb439966150f7c3b686/crates/vim/src/vim.rs#L612)

- vim_mode == VimControl 等等
- vim_operator == x

[Terminal](https://github.com/zed-industries/zed/blob/d209eab05879ddd49c4ebbb439966150f7c3b686/crates/terminal_view/src/terminal_view.rs#L552)

- screen == alt/normal
- mouse_reporting=click/drag/off/motion
- mouse_format=utf8/normal/sgr

[Markdown](https://github.com/zed-industries/zed/blob/d209eab05879ddd49c4ebbb439966150f7c3b686/crates/markdown/src/markdown.rs#L761)

- 渲染的 markdown 窗口。

查看各 zed crate 实现的 Render trait。以 Picker 的 Render 实现为例:

1. key_context("Picker) 定义了该元素 Node 的 context;
2. on_action() 定义了该 Node 监听的按键绑定;(非监听的按键不做处理)
3. children() 指定下一级元素 Node, 这里为 Editor;

所以在 keymap 配置中 Picker 有两个上下文:

1. Picker
2. Picker > Editor

某些 context 的 action 不能用在其它 context 中, context 的可用 action 是由对应
crate module 通过 actions!() 和 impl_actions!() 宏来定义和暴露给命令面板和按键绑定
使用的, 例如:

    picker, [ConfirmCompletion]);
    impl_actions!(picker, [ConfirmInput]);

    // crates/picker/src/picker.rs
    impl<D: PickerDelegate> Render for Picker<D> {
        fn render(&mut self, cx: &mut ViewContext<Self>) -> impl IntoElement {
            let editor_position = self.delegate.editor_position();

            v_flex()
                .key_context("Picker")
                .size_full()
                //...
                .on_action(cx.listener(Self::select_next))
                .on_action(cx.listener(Self::select_prev))
                .on_action(cx.listener(Self::select_first))
                .on_action(cx.listener(Self::select_last))
                .on_action(cx.listener(Self::cancel))
                .on_action(cx.listener(Self::confirm))
                .on_action(cx.listener(Self::secondary_confirm))
                .on_action(cx.listener(Self::confirm_completion))
                .on_action(cx.listener(Self::confirm_input))
                .children(match &self.head {
                    Head::Editor(editor) => {
                        if editor_position == PickerEditorPosition::Start {
                            Some(self.delegate.render_editor(&editor.clone(), cx))
                        } else {
                            None
                        }
                    }
                    Head::Empty(empty_head) => Some(div().child(empty_head.clone())),
                })
                // ...
                .children(self.delegate.render_footer(cx))
                .children(match &self.head {
                    Head::Editor(editor) => {
                        if editor_position == PickerEditorPosition::End {
                            Some(self.delegate.render_editor(&editor.clone(), cx))
                        } else {
                            None
                        }
                    }
                    Head::Empty(empty_head) => Some(div().child(empty_head.clone())),
                })
        }
    }


# 语言

使用 file_types 参数为扩展名或文件路径指定语言类型:

    "file_types": {
      "C++": ["c"],
      "TOML": ["MyLockFile"],
      "Dockerfile": ["Dockerfile*"],
      "JSONC": [
        "**/*.json" // 所有 JSON 文件均使用 jsonc
      ]
    },

zed 支持 by 语言参数[参数列表](https://zed.dev/docs/configuring-languages#language-specific-settings)

    "languages": {
      "Markdown": {
        "tab_size": 2,
        "formatter": "prettier",
        "preferred_line_length": 100,
        "soft_wrap": "preferred_line_length",
        "language_servers": ["intelephense", "!phpactor", "..."],
        "enable_language_server": false  // 是否开启 language server,
         "format_on_save": "off"
      }
    }


各 language server 可以在 lsp 中配置: 配置项名称使用嵌套对象而非 dot 分割字符串, 实例：

    "lsp": {
      // key 为 language server 名称。
      "gopls": {
        "binary": {
          "path": "/Users/alizj/go/bin/gopls",
          "arguments": ["-debug=127.0.0.1:9090"]
        },
        "initialization_options": {
          "usePlaceholders": true,
          "env": {
            "GOOS": "linux",
            "GOARCH": "arm64"
          },
          "buildFlags": ["-tags=debug"],
          "completeUnimported": true,
          "experimentalPostfixCompletions": true,
          "hints": {
            "assignVariableTypes": false,
            "compositeLiteralFields": false,
            "compositeLiteralTypes": false,
            "rangeVariableTypes": true
          }
        }
      }
    }

## rust

对于大型项目，为了避免每次保存文件都触发 rust-analyzer check 影响性能，可以使用如下配置：

    "rust-analyzer": {
      "initialization_options": {
        // 从 rust-analyzer 获得更多的诊断信息（因为后续关闭了cargo check 检查）。
        "diagnostics": {
          "experimental": {
            "enable": true
          }
        },
        // 关闭 checkOnSave 后，rust-analyzer 将不再运行 cargo check 命令，而只运行 rust-analyzer
        // 自身的检查。下面的 check、cargo 配置都将失效。
        // 如果想运行默认的检查，可以执行 zed 自动为 rust 项目生成的 task：argo check --workspace --all-targets
        "checkOnSave": false,
        // 默认 cargo check --workspace --all-targets，影响性能，关闭 --workspace 和 --all-targets。
        "check": {
          "command": "clippy",
          "workspace": false
        },
        "cargo": {
          "allTargets": false
        }
      }
    }

由于 zed 自动为 rust 生成 task，可以手动执行 task：cargo check --workspace --all-targets 来实现 checkOnSave 的效果。

对于包含多个 project 的 zed workspace（它们没有不属于一个 cargo workspace 的 member），可以在 initialization_options 中添加 linkedProjects 列表，这样 ra 会自动诊断它们。

    "linkedProjects": [
      "./path/to/a/Cargo.toml",
      "./path/to/b/Cargo.toml"
    ]

# task

支持全局或项目级别的任务模板（task template）定义, 全局任务模板保存到 `~/.config/zed/tasks.json` 文件中。

zed 也会自动根据项目语言生成一些 task，如对于 rust 项目，自动生成如下 task（-p xx yy 根据当前正编辑的文件而变化）：

- cargo check -p anthropic
- cargo test -p collab ids -- --nocapture
- cargo test -p collab db
- cargo check --workspace --all-targets  # 执行 rust-analyzer 的 checkOnSave 的完整检查。
- cargo clean
- cargo run

任务模板可以使用变量（[列表](https://zed.dev/docs/tasks)）来获得文件/位置/选中的内容等信息, 变量支持缺省值 `${ZED_FILE:default_value}`.

zed 使用 terminal shell 来执行 task 命令 `bash -i -c 'xxx'`。但是当前 rust 对 work directory 的判断是基于当前 *正在编辑的文件* 为基础的，可能会将普通文件判断为 work directory，从而导致 task 执行静默出错，debug 日志如下：


例子：

    [
      {
        "label": "clippy", // 显示到 modal 的项目名称
        "command": "./script/clippy",
        "args": [],
        "allow_concurrent_runs": true, // 是否可以并发运行
        "use_new_terminal": false
      },
      {
        "label": "cargo run --profile release-fast",
        "command": "cargo",
        "args": ["run", "--profile", "release-fast"],
        "allow_concurrent_runs": true,
        "use_new_terminal": false
      }
    ]


执行任务：

    {
      "context": "Workspace && !Terminal",
      "bindings": {
        "ctrl-t": "task::Spawn",
        "ctrl-cmd-t": "task::Rerun" // 快速执行上次的任务
      }
    }


执行 task::Spawn 时，按 tab 选中候选者来修改 task 的命令和参数，也可以输入任意 shell 命令和参数,
然后执行;
- oneshot task："ctrl-enter"，会记录到 task history 中；
- Ephemeral task："ctrl-cmd-enter"，不会记录到 task history 中；

    {
      "context": "Picker > Editor",
      "bindings": {
        // 选中候选者, 如果是 task::Spawn 面板则会在输入框中填写候选者命令配置,
        // 这时可以修改 task 命令和参数.
        "tab": "picker::ConfirmCompletion",

        // 适用于 task::Spawn 面板执行 oneshot shell 命令
        "ctrl-enter": ["picker::ConfirmInput", { "secondary": false }],

        // 适用于 task::Spawn 面板执行 Ephemeral tasks shell 命令
        // 该命令不会记录到 task history 中。
        "ctrl-cmd-enter": ["picker::ConfirmInput", { "secondary": true }]
      }
    }


例如, 计算 zed buffer 中选中内容的字符数：执行 `ctrl-t`， 然后输入：`echo "$ZED_SELECTED_TEXT" | wc -c`， 最后执行 `ctrl-enter`。

# assistant

zed 的 claude 3.5 Sonnet 每个账号每月 10 美元额度，超过的需要自己充值。

assistant context editor 和普通 Editor 一样，支持各种编辑模式的按键绑定。但是该 editor 中包含 message block，每个 block 是不同 role 的 container：

- You
- Assistant
- System

可以点击 role 命令来切换类型。

AI 返回的内容使用 Assistant role block 类型。

使用 `assistant: assist` 命令发送当前 context 的内容给 AI。

各个 context 都保存在本地目录中 `~/.config/zed/conversations`，可以清理和修改。

AI 回答过程中可以使用 escape 中断。

可以随时使用 cmd-n 来创建新 context 来创建新的对话。

可以对 context 对话历史就行修改，比如点击 role block，就可以删除 Assistant 的回答。修改
context 历史对话后，token 消耗数量也随之发生变化。

使用 `assistant: quote selection` 命令将普通 Editor 中的选择区域发送到 assistant editor。
反之，使用也可以将 Assistant panel 中内容插入到 Editor 中。

Inline Assistant：可以在 Editor 或 Terminal 等任意位置使用 Cmd-Enter 触发 inline
Assistant。可以将当前选择内容或当前行发送给 inline Assistant，也可以修改 AI 的响应。

inline assistant 使用当前 assistant panel 的内容作为 context，所以在使用 inline assistant
前，可以将相关信息如 file、selection、diagnose 等发送到 assistant panel。

inline assistant 中不能使用 slash 命令，如 /file ，但是 assistant panel 可以。

支持的 slash 命令：

- /default: Inserts the default prompt into the context
- /diagnostics: Injects errors reported by the project's language server into the context
- /fetch: Fetches the content of a webpage and inserts it into the context
- /file: Inserts a single file or a directory of files into the context
- /now: Inserts the current date and time into the context
- /prompt: Adds a custom-configured prompt to the context (see Prompt Library)
- /symbols: Inserts the current tab's active symbols into the context
- /tab: Inserts the content of the active tab or all open tabs into the context
- /terminal: Inserts a select number of lines of output from the terminal
- /search: Performs semantic search for content in your project based on natural language
    Not generally available yet, but some users may have access to it.
- /workflow: Opts into the edit workflow for a specific context
    Not generally available yet.

部分命令支持参数，如：
- /diagnostics [--include-warnings] [path]
- /file <path>
  - /file src/index.js
  - /file src/*.js
  - /file src
- /prompt <prompt_name>
- /tab [tab_name|all]
- /terminal [<number>]
  - 指定插入的 output line 数量，默认 50

除了上面内置 slash 命令，也可以通过 extension 机制来添加自定义 slash 命令。

# extentions

extensions 可以扩展：
1. theme
2. language
  - meta 信息，如语言名称和扩展名等；
  - 基于 tree-sitter 的 syntax grammer highlight
  - 提供一个或多个 language server
3. slash command
  - 显示在补全列表中，用户选择后执行

extensions 默认被安装到 `~/Library/Application Support/Zed/extensions`。

extensions 使用 Rust 开发，但被编译为 WebAssembly 后被 zed 执行。

实例：

1. [FireCrawl Zed Extension](https://github.com/notpeter/firecrawl-zed)
2. [RFC Zed Extension](https://github.com/notpeter/rfc-zed)

# markdown

bash/shell code block 需要使用 sh 语言简称, 这样 markdown 才能正确高亮对应 code block。

对于 rust/json 等 tree-sitter 可以支持的 language, 不能使用 code block, 否则 outline 会显示
代码的结构,"解决办法是: 使用 2 层缩进。

# 其它

配置 `"show_whitespaces": "selection"` 后，显示选择区域中的空格。

show_completions_on_input vs show_inline_completions：前者是 LSP 代码补全，后者是大模型补全。

字体：默认使用的是 https://github.com/zed-industries/zed-fonts/tree/zed-plex 字体，需要手动下载安装。zed plex font 的主要特点是缩小了字体间距，UI 显示的更紧凑。

# remote ssh

安装 docker desktop。

清理 ~/.docker/config.json 和 ~/.docker/daemon.json 中的旧配置。

安装 SOCKS5 转 HTTP 代理：
``` sh
zj@a:~/go/src/github.com/zed-industries/zed$ pip3 install pproxy
zj@a:~/go/src/github.com/zed-industries/zed$ pproxy -r socks5://127.0.0.1:1080 -vv
Serving on ipv? 0.0.0.0:8080 by http,socks4,socks5
Serving on ipv? :::8080 by http,socks4,socks5
2024-10-29 12:29:57 Unsupported protocol from 127.0.0.1
```

配置 docker-desktop 使用 HTTP 代理：
1. 配置使用 Host Network 类型；
1. 配置 HTTP 和 HTTPS 代理均为 https://127.0.0.1:8080

切换到 zed 项目源码目录，启动 zed：

``` sh
zj@a:~/.config/zed$ cd
zj@a:~$ cd go/src/github.com/zed-industries/zed
zj@a:~/go/src/github.com/zed-industries/zed$  RUST_LOG=debug /Applications/Zed\ Dev.app/Contents/MacOS/zed
```

使用 `projects: open remote` 创建一个 SSH 连接，zed 会自动安装 cross 来为 ssh server 交叉编译一个
zed binary 并上传。

``` sh
zj@a:~/go/src/github.com/zed-industries/zed$ docker ps -a
CONTAINER ID   IMAGE                                                                 COMMAND                     CREATED         STATUS          PORTS     NAMES
c5e53ec7bee9   localhost/cross-rs/cross-custom-zed:aarch64-unknown-linux-gnu-8d728   "sh -c 'PATH=\"$PATH\"…"   3 minutes ago   Up 3 minutes              cross-1.81-x86_64-unknown-linux-gnu-38948-eeb90cda1-aarch64-unknown-linux-gnu-8d728-1730176669812
```

然后登录目标服务器，可见本地的 zed 向它上传了一个本地交叉编译生成的 zed-remote-server-dev-linux--aarch64  二进制并运行：

``` sh
alizj@lima-dev2:~$ ps -eH -opid,args |grep zed
  54801           grep --color=auto zed
  54534         .zed_server/zed-remote-server-dev-linux-aarch64 proxy --identifier dev-workspace-176
  54536   /home/alizj.linux/.zed_server/zed-remote-server-dev-linux-aarch64 run --log-file /home/alizj.linux/.local/share/zed/logs/server-dev-workspace-176.log --pid-file /home/alizj.linux/.local/share/zed/server_state/dev-workspace-176/server.pid --stdin-socket /home/alizj.linux/.local/share/zed/server_state/dev-workspace-176/stdin.sock --stdout-socket /home/alizj.linux/.local/share/zed/server_state/dev-workspace-176/stdout.sock --stderr-socket /home/alizj.linux/.local/share/zed/server_state/dev-workspace-176/stderr.sock
  54683     /home/alizj.linux/.local/share/zed/node/node-v22.5.1-linux-arm64/bin/node /home/alizj.linux/.local/share/zed/languages/json-language-server/node_modules/vscode-langservers-extracted/bin/vscode-json-language-server --stdio
```

同时 zed 在目标服务器上创建和保存了如下文件和目录：

``` sh
alizj@lima-dev2:~$ ./.zed_server/zed-remote-server-dev-linux-aarch64 version
0.160.0
alizj@lima-dev2:~$ ls ~/.config/zed/
settings.json

alizj@lima-dev2:~$ ls ~/.local/share/
applications  zed

alizj@lima-dev2:~$ ls ~/.local/share/applications/
dev.zed.Zed.desktop

alizj@lima-dev2:~$ ls ~/.local/share/zed/
copilot  db  embeddings  extensions  languages  logs  node  prettier  prompts  server_state  zed-stable.sock

alizj@lima-dev2:~$ ls ~/.local/share/zed/languages/
json-language-server
```

也可以使用命令行打开 zed remote ssh：
- 协议链接：zed://ssh/<connnection>/<path>
- zed 命令：zed ssh://my-host/~/code/zed

# Bugs

1. 在 ark 项目中执行 copy permalink 时报错：

    > 2024-10-22T16:48:33.806939+08:00 [ERROR] failed to parse Git remote URL

2. 网络连接不上时，大量刷日志：

   > 2024-10-23T11:06:26.657707+08:00 [ERROR] Client(error sending request for
   > url (https://avatars.githubusercontent.com/u/433567?s=128&v=4)

   > Caused by:
   > 0: client error (Connect)
   > 1: socks connect error: Host unreachable)

   > zj@a:~$ date; wc -l ~/Library/Logs/Zed/Zed.log
   > Wed Oct 23 11:06:47 CST 2024
   > 96710 /Users/alizj/Library/Logs/Zed/Zed.log
   > zj@a:~$ date; wc -l ~/Library/Logs/Zed/Zed.log
   > Wed Oct 23 11:06:51 CST 2024
   > 96980 /Users/alizj/Library/Logs/Zed/Zed.log

3. 按键问题

  - "ctrl-cmd-d": "editor::DeleteToPreviousWordStart", // 不生效
  - "cmd-q": "editor::Rewrap", // 自动折行，有问题，折行的长度不对。

4. 本地交叉编译 remote_server 报错

    [2024-10-29T17:14:42+08:00 DEBUG worktree] ignoring event "target/remote_server/debug/incremental/build_script_build-34db12mrzjok5/s-h1avtiisg8-0xfewcx-working" within unloaded directory
    error: linking with `aarch64-linux-gnu-gcc` failed: exit status: 1
      |
      = note: LC_ALL="C" PATH="/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/x86_64-unknown-linux-gnu/bin:/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/x86_64-unknown-linux-gnu/bin:/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/x86_64-unknown-linux-gnu/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/bin" VSLANG="1033" "aarch64-linux-gnu-gcc" "-Wl,--version-script=/tmp/rustc36YcCI/list" "-Wl,--no-undefined-version" "/tmp/rustc36YcCI/symbols.o" "/target/aarch64-unknown-linux-gnu/debug/deps/zune_jpeg-d98812c935e11704.zune_jpeg.b62aa136303f7057-cgu.00.rcgu.o" "/target/aarch64-unknown-linux-gnu/debug/deps/zune_jpeg-d98812c935e11704.zune_jpeg.b62aa136303f7057-cgu.01.rcgu.o" "/target/aarch64-unknown-linux-gnu/debug/deps/zune_jpeg-d98812c935e11704.zune_jpeg.b62aa136303f7057-cgu.02.rcgu.o" "/target/aarch64-unknown-linux-gnu/debug/deps/zune_jpeg-d98812c935e11704.zune_jpeg.b62aa136303f7057-cgu.03.rcgu.o" "/target/aarch64-unknown-linux-gnu/debug/deps/zune_jpeg-d98812c935e11704.zune_jpeg.b62aa136303f7057-cgu.04.rcgu.o" "/target/aarch64-unknown-linux-gnu/debug/deps/zune_jpeg-d98812c935e11704.zune_jpeg.b62aa136303f7057-cgu.05.rcgu.o" "/target/aarch64-unknown-linux-gnu/debug/deps/zune_jpeg-d98812c935e11704.zune_jpeg.b62aa136303f7057-cgu.06.rcgu.o" "/target/aarch64-unknown-linux-gnu/debug/deps/zune_jpeg-d98812c935e11704.zune_jpeg.b62aa136303f7057-cgu.07.rcgu.o" "/target/aarch64-unknown-linux-gnu/debug/deps/zune_jpeg-d98812c935e11704.zune_jpeg.b62aa136303f7057-cgu.08.rcgu.o" "/target/aarch64-unknown-linux-gnu/debug/deps/zune_jpeg-d98812c935e11704.zune_jpeg.b62aa136303f7057-cgu.09.rcgu.o" "/target/aarch64-unknown-linux-gnu/debug/deps/zune_jpeg-d98812c935e11704.zune_jpeg.b62aa136303f7057-cgu.10.rcgu.o" "/target/aarch64-unknown-linux-gnu/debug/deps/zune_jpeg-d98812c935e11704.zune_jpeg.b62aa136303f7057-cgu.11.rcgu.o" "/target/aarch64-unknown-linux-gnu/debug/deps/zune_jpeg-d98812c935e11704.zune_jpeg.b62aa136303f7057-cgu.12.rcgu.o" "/target/aarch64-unknown-linux-gnu/debug/deps/zune_jpeg-d98812c935e11704.zune_jpeg.b62aa136303f7057-cgu.13.rcgu.o" "/target/aarch64-unknown-linux-gnu/debug/deps/zune_jpeg-d98812c935e11704.zune_jpeg.b62aa136303f7057-cgu.14.rcgu.o" "/target/aarch64-unknown-linux-gnu/debug/deps/zune_jpeg-d98812c935e11704.zune_jpeg.b62aa136303f7057-cgu.15.rcgu.o" "/target/aarch64-unknown-linux-gnu/debug/deps/zune_jpeg-d98812c935e11704.bmqrrknlrqqnfika44j5n5qxv.rcgu.o" "-Wl,--as-needed" "-L" "/target/aarch64-unknown-linux-gnu/debug/deps" "-L" "/target/debug/deps" "-L" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib" "-Wl,-Bstatic" "/target/aarch64-unknown-linux-gnu/debug/deps/libzune_core-1888dc448ab93c4b.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/libstd-2bf0b2a5e0a60917.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/libpanic_unwind-0af01d78b15f6872.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/libobject-aa90d1efd19541cb.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/libmemchr-6645a3a6124c47a1.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/libaddr2line-3de13e717f4d9e74.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/libgimli-f50e3ac5e8bc32ca.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/librustc_demangle-f84a4f82a7a57e94.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/libstd_detect-bd992eebc2a12fc4.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/libhashbrown-c9882005b74b1193.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/librustc_std_workspace_alloc-b18e8234ebc582c8.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/libminiz_oxide-79ef105ee0e8243e.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/libadler-652182712f7d3bc4.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/libunwind-6cb747324af00512.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/libcfg_if-740a433abf104d06.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/liblibc-1e2f311c277b60cf.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/liballoc-85299feea58ac1e7.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/librustc_std_workspace_core-2a73a86214747017.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/libcore-29cdff63f523de0d.rlib" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib/libcompiler_builtins-405c9891256dbf91.rlib" "-Wl,-Bdynamic" "-lgcc_s" "-lutil" "-lrt" "-lpthread" "-lm" "-ldl" "-lc" "-Wl,--eh-frame-hdr" "-Wl,-z,noexecstack" "-L" "/Users/alizj/.rustup/toolchains/1.81-x86_64-unknown-linux-gnu/lib/rustlib/aarch64-unknown-linux-gnu/lib" "-o" "/target/aarch64-unknown-linux-gnu/debug/deps/libzune_jpeg-d98812c935e11704.so" "-Wl,--gc-sections" "-shared" "-Wl,-soname=libzune_jpeg-d98812c935e11704.so" "-Wl,-z,relro,-z,now" "-nodefaultlibs" "-fuse-ld=mold"
      = note: aarch64-linux-gnu-gcc: error: unrecognized command line option '-fuse-ld=mold'; did you mean '-fuse-ld=gold'?

      解决办法：
      1. 下载 mold 包，并解压到 /usr/local
      2. 修改 .cargo/config.toml，使用 "link-arg=-B/usr/local/libexec/mold"，
      https://github.com/zed-industries/zed/pull/19910, 该目录下的 ld 是 mold 的软链接，这样 gcc 在使用 ld 时实际使用的是 old。

      https://gitlab.kitware.com/cmake/cmake/-/issues/25748

      zj@a:~/go/src/github.com/zed-industries/zed$ docker run -it localhost/cross-rs/cross-custom-zed:aarch64-unknown-linux-gnu-8d728 bash

      root@b4fce23c85a8:/app# aarch64-linux-gnu-gcc --version
      aarch64-linux-gnu-gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0
      Copyright (C) 2019 Free Software Foundation, Inc.
      This is free software; see the source for copying conditions.  There is NO
      warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

      root@b4fce23c85a8:/app# which mold
      /usr/local/bin/mold
