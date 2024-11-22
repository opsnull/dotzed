# build

对 `script/bundle-mac` 文件做如下修改:

```sh
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

解决构建 webrtc-sys 失败的问题：

- 将 reqwest 升级到最新的 v0.12 版本；
- 启用 reqwest 的 socks feature；

```sh
zj@a:~/go/src/github.com/zed-industries/zed$ ./script/bundle-mac -ldi
~/go/src/github.com/zed-industries/zed/crates/zed ~/go/src/github.com/zed-industries/zed
~/go/src/github.com/zed-industries/zed
Building for local target only.
   Compiling webrtc-sys v0.3.5 (https://github.com/zed-industries/rust-sdks?rev=4262308983646ab5b0e0802c3d8bc52154f99aab#42623089)
error: failed to run custom build command for `webrtc-sys v0.3.5 (https://github.com/zed-industries/rust-sdks?rev=4262308983646ab5b0e0802c3d8bc52154f99aab#42623089)`

Caused by:
  process didn't exit successfully: `/Users/alizj/go/src/github.com/zed-industries/zed/target/debug/build/webrtc-sys-bf3c821455d0b783/build-script-build` (exit status: 101)
  --- stdout
  cargo:rerun-if-env-changed=LK_DEBUG_WEBRTC
  cargo:rerun-if-env-changed=LK_CUSTOM_WEBRTC
  cargo:CXXBRIDGE_PREFIX=webrtc-sys
  cargo:CXXBRIDGE_DIR0=/Users/alizj/go/src/github.com/zed-industries/zed/target/debug/build/webrtc-sys-5584bf7f821ea101/out/cxxbridge/include
  cargo:CXXBRIDGE_DIR1=/Users/alizj/go/src/github.com/zed-industries/zed/target/debug/build/webrtc-sys-5584bf7f821ea101/out/cxxbridge/crate

  --- stderr

  CXX include path:
    /Users/alizj/go/src/github.com/zed-industries/zed/target/debug/build/webrtc-sys-5584bf7f821ea101/out/cxxbridge/include
    /Users/alizj/go/src/github.com/zed-industries/zed/target/debug/build/webrtc-sys-5584bf7f821ea101/out/cxxbridge/crate
  thread 'main' panicked at /Users/alizj/.cargo/git/checkouts/rust-sdks-e9c3cb1fc511908e/4262308/webrtc-sys/build.rs:85:45:
  called `Result::unwrap()` on an `Err` value: reqwest::Error { kind: Request, url: "https://github.com/livekit/client-sdk-rust/releases/download/webrtc-dac8015-5/webrtc-mac-arm64-release.zip", source: hyper_util::client::legacy::Error(Connect, ConnectError("tcp connect error", Os { code: 61, kind: ConnectionRefused, message: "Connection refused" })) }
```

修改 /Users/alizj/.cargo/git/checkouts/rust-sdks-e9c3cb1fc511908e/4262308/webrtc-sys/build/Cargo.toml，使用 0.12 版本，并且添加 socks features：

    ```toml
    [dependencies]
    reqwest = { version = "0.12", default-features = false, features = ["rustls-tls-native-roots", "blocking", "socks"] }
    ```

修改 /Users/alizj/.cargo/git/checkouts/rust-sdks-e9c3cb1fc511908e/4262308/webrtc-sys/build/src/lib.rs 中的 reqwest get 方法，使用 socks5 proxy。

```rust
let mut client = reqwest::blocking::ClientBuilder::new()
    .proxy(reqwest::Proxy::all("socks5h://127.0.0.1:1080")?)
    .build()?;
let mut resp = client.execute(client.get(download_url()).build()?)?;
//let mut resp = reqwest::blocking::get(download_url())?;
if resp.status() != StatusCode::OK {
    return Err(format!("failed to download webrtc: {}", resp.status()).into());
}
```

解决 mac bundle 构建报错：

> An SSL error has occurred and a secure connection to the server cannot be made

```sh
# 查看系统全局代理：
scutil --proxy

zj@a:~/go/src/github.com/zed-industries/zed$ ls -l target/debug/WebRTC.framework/
total 0
lrwxr-xr-x 1 alizj  24 11  2 11:08 Headers -> Versions/Current/Headers/
lrwxr-xr-x 1 alizj  24 11  2 11:08 Modules -> Versions/Current/Modules/
lrwxr-xr-x 1 alizj  26 11  2 11:08 Resources -> Versions/Current/Resources/
drwxr-xr-x 4 alizj 128 11  2 11:08 Versions/
lrwxr-xr-x 1 alizj  23 11  2 11:08 WebRTC -> Versions/Current/WebRTC*

zj@a:~/go/src/github.com/zed-industries/zed$ ls -l target/debug/WebRTC.framework/Resources/Info.plist
-rw-r--r-- 1 alizj 1018 11  2 11:08 target/debug/WebRTC.framework/Resources/Info.plist
```

编辑生成的 `Info.plist` 文件，在 dict 中添加如下内容：

- 参考：https://github.com/microsoft/vscode/issues/73806#issuecomment-496334904

```xml
<key>NSAppTransportSecurity</key>
   <dict>
       <key>NSAllowsArbitraryLoads</key>
       <true/>
   </dict>
```

本地开发构建使用 `dev profile`，zed 内部会识别当前是否 dev 版本（通过宏 `cfg!(not(debug_assertions))`），会做一些 dev 特殊处理逻辑。

```sh
# 构建 MacOS bundle DMG 并安装
$ ./script/bundle-mac -idl

# 或者只构建 binary
$ cargo build --profile dev
$ RUST_LOG=debug ./target/dev/zed
```

zed 的 ssh_session.rs 的 [update_server_binary_if_needed() 函数
](https://github.com/zed-industries/zed/blob/f919fa92de1d73c492282084b96249b492732f83/crates/remote/src/ssh_session.rs#L1735)
会先执行 server 上的 zed-remote-server 的 version 子命令来获得 server 语义版本(current_version)：

```sh
alizj@lima-dev2:/Users/alizj/.config/zed$ ~/.zed_server/zed-remote-server-dev-linux-aarch64 version
0.160.0
```

编译时，zed 使用文件 `crates/zed/RELEASE_CHANNEL` 中配置来确定 release channel 类型，可选值为：

- dev
- nightly
- preview
- stable

有一些 zed 特性也是根据 release channel 类型来做不同处理的。例如 ssh_sessions.rs 的
update_server_binary_if_needed() 根据 release channel 来确定需要为 remote server
[安装的版本（wanted_version）](https://github.com/zed-industries/zed/blob/40802d91d4faf849ad35fb53d6b00320c1d04cc1/crates/remote/src/ssh_session.rs#L1760)：

1. 如果是 dev，则设置 wanted_version 为 None，后续进行本地构建；
2. 如果是 nightly、preview、stable，则从 zed.dev API 获得对应版本；

如果执行成功则获得 current_version 值，否则将它设置为 None，则进行版本比较(current_version vs wanted_version)：

1. 如果两者都有值且匹配，则不安装或升级；
1. 如果本地版本低，则提示升级本地 zed 版本后返回；
1. 否则（如 server 版本低，或者有任何一方为 None），则会安装新版本。

在安装新 remote server binary 前，zed 会检查 bianry 是否在使用。如果在使用且 zed
不是 dev 版本，则会直接返回错误，提 示 binary 在 使用，不能升级。但是如果是 dev 版本，
则即使在使用也可以升级。

如果是 dev 模式（wanted_version 为 None）：

1. 先检查环境变量 `ZED_BUILD_REMOTE_SERVER` 是否设置，如果 **未设置** ：
1. 如果 current_version 有值，则复用 binary，直接返回；
1. 如果无值，则报错：ZED_BUILD_REMOTE_SERVER is not set, but no remote server exists
1. 在设置 ZED_BUILD_REMOTE_SERVER 的情况下：
1. 如果是 dev 模式，则进行本地构建和上传到 server；
1. 否则报错：Running development build in release mode, cannot cross compile
   (unset ZED_BUILD_REMOTE_SERVER)

如果不是 dev 模式，则检查配置参数 upload_binary_over_ssh：

1. 如果为 false（默认），则 server 尝试先从 zed.dev 下载 binary，如果失败则从 zed 本地上传。
2. 如果为 true，则从 zed 本地上传。

从 zed 本地上传：本地 zed 先下载 binary，然后上传到 server。

总结：在 dev 模式下：

1. 如果未设置环境变量 ZED_BUILD_REMOTE_SERVER，则要求远端已经有 bianry 在运行，**直接复用**。
2. 如果设置 ZED_BUILD_REMOTE_SERVER，则会本地侯建和上传。

```sh
ZED_BUILD_REMOTE_SERVER=1  RUST_log=debug target/debug/zed
```

zed 本地构建 remote server bianry 时执行的命令：

1. 同构：cargo build --package remote_server --features debug-embed --target-dir target/remote_server
2. 异构：triple=aarch64-linux cargo install cross --git "https://github.com/cross-rs/cross"
   cross build --package remote_server --features debug-embed --target-dir target/remote_server --target ${triple}

在 zed server 运行过程中，会自动[从网络下载 lsp language 并安装](https://github.com/zed-industries/zed/blob/f919fa92de1d73c492282084b96249b492732f83/crates/languages/src/rust.rs#L100)
到 ~/.local/share/zed/languages/ 目录下：

# launch

窗口右上角显示资源用量和 GPU FPS 信息（有 bug，可能导致段错误）：

```sh
$ MTL_HUD_ENABLED=1 /Applications/Zed.app/Contents/MacOS/zed
```

DEBUG 启动模式:

```sh
# 先切换到 zed 源码目录(有些命令, 如 ssh remote 会在源码目录编译一些二进制)
$ pwd
/Users/alizj/go/src/github.com/zed-industries/zed
$ RUST_LOG=debug /Applications/Zed\ Dev.app/Contents/MacOS/zed
```

zed cli : 可以通过 Zed 菜单 “Install CLI” 来安装 zed 命令行工具命令 zed：

```sh
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
2. 通过 dock 启动, 先切换到 HOME 目录 spawn 一个 login shell 来获得用户环境变量, 然后被所有 zed 窗口继承;

zed 打开 project 时, 会使用 direnv/editorconfig 等机制来获得项目相关的环境变量, 并被项目的
task/lsp/terminal 继承。

本地安装的扩展、LSP Server、Remote binary 等位于 ~/Library/Application\ Support/Zed/
目录下：

```sh
$ ls ~/Library/Application\ Support/Zed/
copilot/  db/  docs/  extensions/  languages/  node/  prettier/  remote_servers/

$ ls ~/Library/Application\ Support/Zed/languages/
eslint/  json-language-server/  package-version-server/  pyright/  rust-analyzer/  tailwindcss-language-server/  vscode-css-language-server/  vtsls/  yaml-language-server/
```

# layout

一个 window 有多个 panes （通过 spit），一个 pane 有多个 items（tabs）。

pane 有自己的 tool bar 和导航 history（前进、后退）。光标在 Pane buffer 的移动位置会被记录到 Pane 导航历史记
录中，可以点击左上角的左、右箭头来回到上一次、下一次的位置。通过使用导航历史快捷键，可以快捷的前后移动。

可以使用 ctrl-0/1/2 等数字快捷键快速在多个 pane tab 切换。

将 project 和 outline panel 都设置到 left dock, 便于查看。

# editing

zed 打开系统文件对话框后，按 Command-Shift-g 可以按照文件路径来打开。也可以在终
端使用 `zed cli` 来按照文件路径打开文件。

使用 tab switcher 可以快速在当前已经打开的文件间切换，而且默认选择上一次打开的文件。
+ 最便捷的多文件编辑时切换机制；

在终端中快速打开当前 pane item 对应的目录：workspace::OpenInTerminal

在 editor 或 terminal buffer 中，当光标位于 URL (需要带 http 或 https 前缀)或
File Path 上时， 可以按 cmd 来快速打开。

快速选择一个 block：将光标移动到 block 边界字符上，然后按 ctrl-= 来按语法选择。

输入法设置:

1. 不使用 MacOS 内置的拼音输入法，因为它使用中文标点，导致快捷键 ctrl-] 使用中文标点
   】，从而与 zed 快捷键绑定不兼容；
2. 使用微信输入法；
3. 启用微信输入法的 shift 中英文切换快捷键。
4. 关闭 “自动编号”；

输入法 keylayout：

- 微信和搜狗中英文输入法：com.apple.keylayout.US
- APPLE 英文：com.apple.keylayout.ABC
- APPLE 中文：com.apple.keylayout.PinyinKeyboard

微信输入法小技巧：

1. 如果当前是中文输入状态，但需要上屏英文，可以按 shift 键，这样输入的部分内容会被作为
   英文输入；
2. 输入 “日期” 或 “时间” 时会自动提示插入各种格式的日期和时间；
3. 有些 emoji 字符有多种选择，可以使用上下箭头来选择；

在 20241121 的 commit [Clip UTF-16 offsets in text for range](https://github.com/zed-industries/zed/pull/20968) 合并后，在开启微信中文输入法的情况下，
快捷键绑定中也能使用单字母了。

buffer 和 terminal 都设置为更符合编程体验的 "Sarasa Mono SC" 字体，它是 Iosevka 编程字体的中文版本，名称为等距更纱黑体。

光标位于 URL 上, 执行 editor::open url 命令可以快速打开该 URL。

project panel 默认对空目录进行折叠, 双击折叠的目录时会展开。

editor::Wrap 目前只对 Markdown 和 Plain Text Buffer 类型有效：

```rust
// https://github.com/zed-industries/zed/blob/96683da9f90bad4e1c2778c3607788f3eed31560/crates/editor/src/editor.rs#L7004
if let Some(language_scope) = buffer.language_scope_at(selection.head()) {
    match language_scope.language_name().0.as_ref() {
        "Markdown" | "Plain Text" => {
            should_rewrap = true;
        }
        _ => {}
    }
}
```

安装 org extention 后，就可以高亮 org-mode 文件。

配置 `"show_whitespaces": "selection"` 后，显示选择区域中的空格。

show_completions_on_input vs show_inline_completions：前者是 LSP 代码补全，后者是大模型补全。

字体：默认使用的是 https://github.com/zed-industries/zed-fonts/tree/zed-plex 字体，需要手动下载安装。zed plex font 的主要特点是缩小了字体间距，UI 显示的更紧凑。

# multicusor

在编辑窗口（普通编辑窗口或搜索结果窗口），按住 alt 后点击要增加 cursor 的位置，然后就可
以多光标同时编辑。

# search

搜索分为 和 project search，支持关键字、word 和正则搜索方式， 也可以
忽略大小写。

buffer search 是每输入一个字符就触发的实时增量搜索, 而 project search 是输入完所有
搜索字符后按 enter 后后才触发 搜索。

搜索时，默认选中光标处的 symbol/word，也可以先选中内容后再搜索。

搜索时，默认选中所有匹配项，导致前后跳转时不易分清光标所在行，两个办法：

1. 开启行号来明确显示当前匹配的行；（通过命令面板开启）。
2. 在 outline pane 看当前匹配的行。
3. 焦点切换到编辑窗口（按 tab），按 `ctrl-l` 将光标滚动到窗口中心。

选中搜索框右侧的 `Select All Match` 按钮对当前选中的匹配项
（默认选中所有匹配项）启用多光标编辑， 实现搜索结果的批量编辑。

搜索的结果可以在 outline panel 显示，实现快速跳转和二次过滤。

project search 的结果默认在 preview tab 中显示（标题是斜体），它是临时buffer，在其
中双击后就显示 对应位置的文件内 容，不能再返回到以前的结果 buffer。 解决办法：双击前，
先将该 preview tab pin 住或转换 为普通 tab（双击tab）。

正则搜索使用扩展正则表达式语法, 参考 [regex crate 文档](https://docs.rs/regex/latest/regex/#syntax).

# outline

outline 是基于 tree-sitter 解析的节点树，支持对编辑窗口、搜索窗口（buffer 或
project 搜索）、Reference 窗口、诊断窗口的结构化显示。包含三种类型：

- buffer outline
- project outline
- outline panel

outline panel 支持多种快捷操作（Actions），如目录的展开和合并，跳转到上一级，在
Finder 中打开文件等。

关闭在 outline-panel 显示 markdown、org-mode 中代码块的功能：

1. org-mode：
``` sh
zj@a:~/Library/Application Support/Zed/extensions/installed/org/languages/org$ cat injections.scm
;(block . name: (expr) parameter: (expr) @language (contents) @content)
```

2. markdown

``` sh
zj@a:~/go/src/github.com/zed-industries/zed$ git diff crates/languages/src/markdown/injections.scm
crates/languages/src/markdown/injections.scm --- Text (4 Scheme parse errors, exceeded DFT_PARSE_ERROR_LIMIT)
1 (fenced_code_block                                             1 ;(fenced_code_block
2   (info_string                                                 2 ;  (info_string
3     (language) @language)                                      3 ;    (language) @language)
4   (code_fence_content) @content)                               4 ;  (code_fence_content) @content)
5                                                                5
6 ((inline) @content                                             6 ((inline) @content
7  (#set! "language" "markdown-inline"))                         7  (#set! "language" "markdown-inline"))
```

# multibuffer

project 级别的 search/reference/diagnose 打开的窗口是 multibuffer 类型。

multibuffer 中的文件位置称为片段（excerpt），支持多光标编辑和保存。

配置 `"double_click_in_multibuffer": "open"` 选项后，双击 multibuffer 片段时，
在新的 tab 打开对应文件位置。

# preview tabs

在 project panel 中点击各文件时，默认使用 preview tab 显示，点击其它文件时，
preview tab 会切换显示该文件，从而避免同时打开很多个 tabs，特别适合只读查看文件的情况。

preview tabs 的 tab 标题是 _斜体显示_ ，从而可以和普通独立 tab 区分开来。

preview tabs 通过以下方式转换为普通独立 tab：

1. project pandel 中双击打开文件；
2. 使用 project_panel::OpenPermanent action 打开文件；
3. 编辑文件。
4. pin tab。
5. 双击 tab。
6. 将 tab 拖移到其它 pane。

还可以在配置文件中配置 code nav 和 file finder 使用 preview tabs。

# keybindings

使用命令 `debug: Open Key Context View` 查看当前焦点的 context，触发的按键，以及按键匹配情况。

zed 按键绑定（`/.config/zed/keymap.json`）不区分相同按键序列但不同顺序的情况，如`ctrl-cmd-a` 和 `cmd-ctrl-a` 是相同的按键，但 zed 不提示重复的按键绑定。解决办法：使用固定的顺序来写按键，如 `ctrl-cmd-alt-shift`。

统一规划一些前缀快捷键，如 `ctrl-x`, 它们只用于前缀场景，而不单独使用，否则会导致按键响应延迟。（因为 zed 会等待一段时间来接收前缀后续 的按键，当超时后，才认为是致独立绑定语义）。

zed 支持灵活的按键 remap：

- `["workspace::SendKeystrokes", "ctrl-down ctrl-down ctrl-down ctrl-down ctrl-down"]`
- `["workspace::SendKeystrokes", ": task:spawn enter Test Under Cursor enter"]`
- `["task::Spawn", { "task_name": "Example task" }]`
- `["assistant::InlineAssist",{ "prompt": "Build a snake game" }]`

自定义按键绑定覆盖缺省按键绑定，缺省绑定中未覆盖的按键继续有效。所以，如果要确保自己的按键定生效，则可能需要在多个 context 中重复设置。

不是所有 action 在所有 context 中都有效， 如果高优 context 中的按键绑定 action 无效， 则会 fallback 到低优 context 中该按键绑定的 action，以此类推直到第一个有效 action。

例如 Editor 和 Editor && mode == full 的 context 都定义了 ctrl-o 快捷键，但是后者的 excerpt 只在 multibuffer 中有效，所以 fallback 到 Editor 中的 buffer symbol：

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

zed 窗口是由层次化的 UI 元素节点组成的，节点间有父子、兄弟关系，处于不同层次的上下文中。 反映到按键上，就是有优先级，嵌套越深的层次上定义的快
捷键优先级越高，如 buffer 搜索输入框的层次是：

Workspace > Pane > BufferSearchBar > Editor(搜索框)

zed 从配置中加载所有按键绑定，然后用户输入对应按键绑定时，过滤 context 条件符合要求的 actions 列表，然后根据 context 所在的 UI 节点深度，选择最深层次上定义的 action。

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

单行输入框上下文，例如搜索框、替换框、过滤框等，可以用 Editor && mode== single_line 匹配。

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

# language

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

## python

内置支持两个 language server：

- pyright（默认）
- pylsp

安装 pyright lsp server：

```sh
# 建议：微软官方提供的安装方式
sudo npm install -g pyright
# 社区维护的安装方式
# pip install pyright
```

配置 zed 使用自定义安装的 pyright：

```json，
"pyright": {
  "binary": {
  "path": "/opt/homebrew/bin/pyright-langserver",
  "arguments": ["--stdio"]
  }
},
```

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

在浏览器打开符号本地文档：

1. 先使用 cargo doc 生成本地文档，否则后续打开的是 https://crates.io/ 的在线文档；
2. 执行命令："editor::OpenDocs"；

显示 RA 的启动信息：

1. 在 ~/.bashrc 中添加环境变了：export RA_LOG=info
2. 实际测试，在 project 中添加 .envrc 以及在 settings.json 中设置该变量都不会生效。

# task

支持全局或项目级别的任务模板（task template）定义, 全局任务模板保存到 `~/.config/zed/tasks.json` 文件中。

zed 也会自动根据项目语言生成一些 task，如对于 rust 项目，自动生成如下 task（-p xx yy 根据当前正编辑的文件而变化）：

- cargo check -p anthropic
- cargo test -p collab ids -- --nocapture
- cargo test -p collab db
- cargo check --workspace --all-targets # 执行 rust-analyzer 的 checkOnSave 的完整检查。
- cargo clean
- cargo run

任务模板可以使用变量（[列表](https://zed.dev/docs/tasks)）来获得文件/位置/选中的内容等信息, 变量支持缺省值 `${ZED_FILE:default_value}`， 它们可以在 cwd, args 和 label 中使用。

env 字段指定添加到 task 的环境变量，不支持变量替换，优先级高，会覆盖 terminal 环境变量。

zed 使用 terminal shell 来执行 task 命令 `bash -i -c 'xxx'`。但是当前 zed 对 work directory 的判断是基于当前 _正在编辑的文件_ 为基础的，可能会将普通文件判断为 work directory，从而导致 task 执行静默出错。

注意：

1. 不支持物理换行，即使前面加 \ 转义字符也不行；
2. cwd 和 env 不支持 shell 扩展，如不能使用 ~ 字符；
3. 字符串支持 \ 转义，如果要把转义字符传给 shell，需要连用 \\;

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
然后执行：

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

注意：如果修改了 task 定义，则在 task picker 界面应该选择最下面的任务定义，而不是上面执行过的历史任务，否则最新的定义不生效。

例如, 计算 zed buffer 中选中内容的字符数：执行 `ctrl-t`， 然后输入：`echo "$ZED_SELECTED_TEXT" | wc -c`， 最后执行 `ctrl-enter`。

## 切换 org-mode src block 语言名称

临时解决，解析后的 src block 代码显示在 Outline 的问题：

    ``` json
    // Static tasks configuration.
    //
    [
      {
        "label": "toggle org-mode src block",
        "command": "lines=$(grep 'begin_src rust,' ${ZED_FILE} 2>/dev/null | wc -l); if [[ $lines > 10 ]]; then sed -i  -E -e 's/begin_src ([[:alnum:]]+),/begin_src \\1/g' ${ZED_FILE}; else sed -i -E -e 's/begin_src ([[:alnum:]]+)/begin_src \\1,/g' ${ZED_FILE}; fi",
        "use_new_terminal": false,
        "allow_concurrent_runs": false,
        "reveal": "always",
        "hide": "always",
        "cwd": "/Users/alizj/docs/"
      }
    ]
    ```

## 快速切换输入法

定义一个 task：

    ``` json
    {
      // 需要事先安装 macism 命令。
      "label": "switch-input-method",
      "command": "current=$(macism)",
      "args": [
        ";if [[ $current =~ com.sogou.inputmethod.sogou.pinyin ]]; then macism com.apple.keylayout.ABC; else macism com.sogou.inputmethod.sogou.pinyin; fi"
      ],
      "reveal": "never", // 执行时不显示终端
      "use_new_terminal": false, // 复用已有未结束的终端
      "allow_concurrent_runs": true,
      "hide": "always" // 任务结束后自动关闭终端
    }
    ```

绑定到快捷键 shift-shift：

    ``` json
    {
      "context": "Workspace && !Terminal",
      "bindings": {
        "ctrl-t": "task::Spawn",
        // 重新执行上次的 task
        "ctrl-cmd-t": "task::Rerun",
        // 快速切换输入法
        "shift shift": ["task::Spawn", { "task_name": "switch-input-method" }]
      }
    },
    ```

## lazygit

编辑配置文件 `~/.config/lazygit/config.yml`：

    ``` yaml
    gui:
      language: en
    os:
      editPreset: "zed" # 使用 zed 编辑文件
    ```

创建 zed task：需要配置环境变量 `XDG_CONFIG_HOME` 来指定 lazgit 的配置目录。

      ``` json
      {
        "label": "Lazygit",
        "command": "lazygit",
        "args": [],
        "env": {
          "XDG_CONFIG_HOME": "/Users/alizj/.config"
        },
        "use_new_terminal": false, // 复用已有未结束的终端
        "allow_concurrent_runs": true,
        "hide": "always" // 任务结束后自动关闭终端
      }
      ```

参考：

1. https://oliverguenther.de/2021/04/lazygit-an-introduction-series/
2. https://github.com/jesseduffield/lazygit/blob/master/docs/keybindings/Keybindings_en.md

全局：

- C-r：切换最近的项目；
- @：打开右下角的 git 命令提示面板；
- ？：打开帮助菜单；
- f: 从 remote fetch 最新的更新；
- P: push
- p: pull
- q 或 C-c：退出（quit）
- z：undo
- C-z：redo
- j/k 或 <UP>/<DOWN>: 前一个或后一个 item, 也即是前后移动;
- <left>/<right>: 在 block panel 间跳转，共有编号为 1-5 的 5 个 block；
- 1-5: 跳转到对应编号的 block；
- [/]: 在一个 block panel 的多个 tab 中切换。
- esc：返回（return）上一级；
- +/-： 切换当前 tab 的显示方式（全屏、半屏等），在查看 diff 或 commit 时非常有用。
- H/L: 左右 scroll;
- </>: 移动到 buffer 开始或结束；
- : : 执行 shell 命令。
- search：在不同 tab 中使用 / 来触发搜索，但语义可能不一致，使用 n/N 来前后搜索。
  - C-b ：按 status 过滤文件；
  - C-s ：按 path、commit、author 过滤文件；
- 前一个或后一个 page：,/.
- R: 刷新 git 状态（后台执行 git status，git branch 等命令以更新面板，但是不执行 git fetch）；
- d：discard 丢弃文件变更
- D：显示 reset 高级选项，包括 soft、hard 等；
- g: reset 到 UPSTREAM；
- <SPACE>：在 file panel 中是 stage 当前文件，在 diff panel 中是 stage 当前 hunk；

- o: 使用外部编辑器（external editor）打开文件(显示变更后的文件内容)
- e：使用系统缺省应用（dfault application）编辑文件。

比较 Commit：

- W: 在 commit 或 branch 上执行时，将当前 commit 作为标记与后续选择的其它 commit 进行比较，差异
  显示在 diff panel 中。这时按 <enter> 来显示 diff 的文件列表, 再次按 W 将显示翻转 diff 方向，或者
  退出 diff mode。

File Panel：

- Untracked(??), Added (A), Deleted(D), or Modified(M)
  - unstaged (red) and staged (green)
- <enter>: 打开当前文件或目录的 unstage diff panel，这样可以按 hunks 或 lines 来 stage。
- <SPACE>: stage 当前文件
- `: 切换文件树的显示方式（层次或扁平显示）
- a: stage、unstaged 所有文件
- s：stash 当前文件
  - S: 查看 stash 选项（e.g. stash all, stash staged, stash unstaged)
- i: 忽略当前文件
- c: commit 当前的 stage 文件；
- A：amend 上一次 commit
- C-b: 只显示 stage 或 unstage 的文件；
- r: refresh 文件

Diff Panel:

- h/l 或 <left>/<right>: 前一个或后一个 hunk：
- <SPACE>: stage 当前 hunk line 或 selection，如果已经 staged 了，则 unstage 当前 hunk；
- a: Toggle hunk selection mode，即一次选择一个 hunk；
- range select：可以批量对选择的项目（file、commit）应用命令，如在 unstage panel 中选择一部分
  hunk，然后使用 SPACE 命令来进行 stage。反之，在 stage panel 中，使用 v 选择一个 range 后，
  使用 SPACE 命令来进行 unstage。
  1. 先按 v，然后使用 up、down 或 j/k 来选择。再次按 v 来 reset 选择；
  2. 或者按 shift+up 或 shift+down 来选择。再次按不带 shift 的 up、down 来 reset 选择；
- d：discard 丢弃部分文件变更（丢弃文件中 staged、unstaged 部分的变更）
  - When unstaged change is selected, discard the change using git reset.
  - When staged change is selected, unstage the change.
- <TAB>: 在 unstage 和 stage view 之间切换；
- E: 编辑当前 diff hunk，编辑后保存关闭临时文件。
- esc: 返回到 file panel。
- {/}： 增加或减少 diff 上下文行数。
- c: Commit staged changes.
- w：Commit changes without pre-commit hook
- C：Commit changes using git editor

Branch Panel：

- 以 \* 开头的 branch 为当前 branch；向上箭头表示 commit ahead，向下箭头表示 commit behind；
- <SPACE>: checkout 当前 branch；
- c：checkout by name，按 branch name 来 checkout，如果是 -，则代表 latest branch；
- <Enter>: 查看当前 branch 的 commit 历史，在某个 commit 上按 <Enter> 则显示 Commit Panel。
- p：pull 远程分支最新提交并合并到本地
- f：fast-forward 当前分支到远程分支
- F：force checkout，会丢弃当前 workspace 所有变更；
- n：创建新 branch
- R：rename branch
- M：将当前光标所在分支 merge 到本地当前分支，显示 merge 选项
- r：将当前本地分支 rebase 到光标所在分支

* /: 按 branch name 搜索
* [/]: 切换到该 panel 的其它 tab，如 remote branch 和 tags；
* w : 从该 branch 创建 worktree；

Commit Panel: 显示当前本地分支的 commit history list，可以进行 Squash、Fixup、Amend、Reword 等操作。

- 4：切换到该 Panel
- <Enter>: 显示该 commit 下的变更文件列表，在文件上按 <Enter> 显示该文件的 Patch。
  - 和 diff panel 一样，可以对整个文件或部分 hunk 选择，然后生成 custom patch；
  - 最后执行 Ctrl-p 来显示 custom patch menu；
    - Reset patch cancel the custom patch, resetting any selections you've made
    - Apply patch Run git applywith the patch here (this won't do anything if the patch is from this commit)
    - Apply patch in reverse Run git apply --reverse with the patch here
    - Remove patch from original commit Removes the selected changes from this commit and discards it! (Results in rebase)
    - Move patch into new index Removes the selected changes from this commit and adds it to the stash (Results in rebase)
    - Move patch into new commit Removes the selected changes from this commit and adds a new commit with the changes. The commit message will be "Split from <this commit>"
- / : 按 commit hash id 或 message summary 搜索；

- i：开始 interactive rebase，对要进行 rebase 的 commit 进行 squash (s), fixup (f),
  drop (d), edit (e), move up (ctrl+i) or move down (ctrl+j)，然后使用 m 触发
  rebase options menu，选择 continue。可以使用 v 来选择多个 commit。
- e：编辑（edit）选中的 commit，会对该 commit 以后的 commit 进行 rebase

- A: amend, 如果 files panel 中有 staged changes 则 amend 会带上。如果对历史 commit 进行
  amend，会进行 rebase。
- s：squash，将所在 commit 合并到前一个（below it） commit（保留 commit message），会进行 rebase
- f：fixup，将所在 commit 合并到后一个 commit（丢弃 commit message），会进行 rebase。
- d：delete commit，会进行 rebase。
- r：reword，编辑 commit message，会进行 rebase。
- R：reword in editor，会进行 rebase。

- F: Create fixup commit，为选择的 commit 创建 fixup！commit，后续在该 commit 上执行 S 来
  squash all above fixup commits
- S：Apply fixup commits，Squash all 'fixup!' commits, either above the selected commit,
  or all in current branch (autosquash).

- t：revert commit
- T：tag commt
- B: 将光标所在 commit 标记为 rebase Base commit（不包含光标所在的 commit，往上的所有 commit
  将被 rebase 到其它分支），然后切换到其它 branch 执行 rebase；
- a：Set/Reset commit author or set co-author.
- <c-l>: 设置 commit log graph 显示选项；
- <c-t> Open external diff tool (git difftool)
- <SPACE>: Checkout the selected commit as a detached HEAD.
- y：Copy commit attribute to clipboard (e.g. hash, URL, diff, message, author).
- o：Open commit in browser
- n：Create new branch off of commit
- g：View reset options (soft/mixed/hard) for resetting onto selected item.

Push 到远程不同的分支：

1. 需要本地先建一个后续 push 到远程的分支；
2. 执行 P 来 push 到远程。

Merge conflict 解决：

1. 有 conflict 的文件会显示 U 字母，这时 lazygit 会显示冲突列表；
2. 使用 j/k 来在冲突列表中移动，然后 <SPACE> 来选择某一个版本。也可以使用 e/o 打开编辑器, 来编辑解决冲突。
3. 最后 stage 解决完冲突的文件；

Cherry picking：

1. 在 commit panel 中使用 C（copy cherry-pick） 选中一个或多个 commit；
2. 然后切换或选择另一个分支，使用 V（paste cherry-pick）来生效；

Stash panel：stash 将当前 worktree 的变更保存到 stash 空间，然后将 worktree 恢复到上一次 commit 的干净状态。

- 使用 5 来切换到该 panel
- 在 files tab 中使用 s 命令将 staged 或 unstaged 的内容保存到 stash；
- <SPACE>: apply stash 但是不 popup；
- g：popup stash
- n：从 stash 创建一个 branch；
- d：delete stash entry；

Interactive Rebase 操作（修改当前分支的 commit）：

- https://github.com/jesseduffield/lazygit/wiki/Interactive-Rebasing

1. 按 4 切换到 commit panel；
2. 在要开始 interactive rebase 的 commit 上按 e（Edit），这时会从该 commit parent 开始
   rebase，光标所在 commit 是 edit 状态，以后的（above）的 commit 都会被 pick 选中。

- 如果执行的是 i 命令，则可以对整个 commit history 的 commit 进行标记，而不是像 edit 那样
  从标记的 commit 开始 rebase。

3. lazygit 提示 YOU ARE HERE，这时 git 暂停在该 commit，等待用户修改该 commit（即 edit 语义）：

- 可以对 above commit 进行修改，'squash（s）', 'drop（d）', 'edit（e）', 'pick（p）', and 'fixup（f）'；

4. 修改 commit：在 worktree 中修改文件，然后 stage（<SPACE>)，再 amend 该 commit (shift-A)，
   这时在 commit diff 中可以看到该修改被合并到那个 commit 中。
5. 回到 commit panel，按 m 来触发 merge or rebase options，然后选择 continue。
6. 然后从 edit 开始的 commit 到最新的 commit 都会被重新提交，生成新的 commit id。

Rebase onto 操作（将其它分支 commit rebase 到任意分支）：

1. 在 branch panel 中选择 bugfix 分支，commit panel 中显示该分支的 commit；
2. 在 commit panel 中选择要 Rebase 的 Base Commit，按 B，该 commit 后续（不包含该 commit）的
   commits 都会被 rebase 到后续其它分支；
3. 在 branch panel 中选择要 rebase 到的其它分支，按 r，这时会出现 rebase dialog，可以进行 simple
   或 iteractive rebasing。

注：amend 操作用于修改 commit，将当前 workspace 中要做的修改 stage，然后 git commit --amend 时会
合并到该 commit 中。

Bisect

- Press b in the commits view to mark a commit as good/bad in order to begin a git bisect.

What's the difference between git reflog and log?

- https://stackoverflow.com/a/17860179/19867059

  git log shows the current HEAD and its ancestry. That is, it prints the commit HEAD points to, then its parent, its parent, and so on. It traverses back through the repo's ancestry, by recursively looking up each commit's parent.

  (In practice, some commits have more than one parent. To see a more representative log, use a command like git log --oneline --graph --decorate.)

  git reflog doesn't traverse HEAD's ancestry at all. The reflog is an ordered list of the commits that HEAD has pointed to: it's undo history for your repo. The reflog isn't part of the repo itself (it's stored separately to the commits themselves) and isn't included in pushes, fetches or clones; it's purely local.

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
  - /file src/\*.js
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
3. [安装 extension 的脚本](https://gist.github.com/srghma/1a53015ed2c26f725119b1c9dc43a3ab)

# snippet

snippet 按语言名称保存在 ~/.config/zed/snippets 目录中。

body 字符串可以使用 \ 对 \$}" 进行转义。

通过 ${1|choice1,choice2,choice3} 语法支持可选值列表提示。

    ```json
    "Log to the console": {
      "prefix": "log",
      "body": ["console.log($1);", "$0"],
      "description": "Log to the console"
    },
    "Log warning to console": {
      "prefix": "warn",
      "body": ["console.warn($1);", "$0"],
      "description": "Log warning to the console"
    },
    "Log error to console": {
      "prefix": "error",
      "body": ["console.error($1);", "$0"],
      "description": "Log error to the console"
    },
    "Throw Exception": {
      "prefix": "throw",
      "body": ["throw new Error(\"$1\");", "$0"],
      "description": "Throw Exception"
    }
    "my snippet": {
        "prefix": "log",
        "body": ["type ${1|i32, u32|} = $2"],
        "description": "Expand `log` to `console.log()`"
    },
    "my snippet2": {
        "prefix": "snip",
        "body": [
          "type ${1|i,i8,i16,i64,i32|} ${2|test,test_again,test_final|} = $3"
        ],
        "description": "snippet choice tester"
      }
    ```

# markdown

bash/shell code block 需要使用 sh 语言简称, 这样 markdown 才能正确高亮对应 code block。

对于 rust/json 等 tree-sitter 可以支持的 language, 不能使用 code block, 否则 outline 会显示
代码的结构,"解决办法是: 使用 2 层缩进。

# theme

执行 `debug: open theme preview` 命令预览当前主题的配色。

zed 各主题预览：https://zed-themes.com/?order=recent

几款高质量主题：

1. https://github.com/catppuccin/zed
2. 内置的 Gruvbox Dark Hard

# remote

安装 docker desktop。

清理 ~/.docker/config.json 和 ~/.docker/daemon.json 中的旧配置。

安装 SOCKS5 转 HTTP 代理：

```sh
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

```sh
zj@a:~/.config/zed$ cd
zj@a:~$ cd go/src/github.com/zed-industries/zed
zj@a:~/go/src/github.com/zed-industries/zed$  RUST_LOG=debug /Applications/Zed\ Dev.app/Contents/MacOS/zed
```

使用 `projects: open remote` 创建一个 SSH 连接，zed 会自动安装 cross 来为 ssh server 交叉编译一个
zed binary 并上传。

```sh
zj@a:~/go/src/github.com/zed-industries/zed$ docker ps -a
CONTAINER ID   IMAGE                                                                 COMMAND                     CREATED         STATUS          PORTS     NAMES
c5e53ec7bee9   localhost/cross-rs/cross-custom-zed:aarch64-unknown-linux-gnu-8d728   "sh -c 'PATH=\"$PATH\"…"   3 minutes ago   Up 3 minutes              cross-1.81-x86_64-unknown-linux-gnu-38948-eeb90cda1-aarch64-unknown-linux-gnu-8d728-1730176669812
```

然后登录目标服务器，可见本地的 zed 向它上传了一个本地交叉编译生成的 zed-remote-server-dev-linux--aarch64 二进制并运行：

```sh
alizj@lima-dev2:~$ ps -eH -opid,args |grep zed
  54801           grep --color=auto zed
  54534         .zed_server/zed-remote-server-dev-linux-aarch64 proxy --identifier dev-workspace-176
  54536   /home/alizj.linux/.zed_server/zed-remote-server-dev-linux-aarch64 run --log-file /home/alizj.linux/.local/share/zed/logs/server-dev-workspace-176.log --pid-file /home/alizj.linux/.local/share/zed/server_state/dev-workspace-176/server.pid --stdin-socket /home/alizj.linux/.local/share/zed/server_state/dev-workspace-176/stdin.sock --stdout-socket /home/alizj.linux/.local/share/zed/server_state/dev-workspace-176/stdout.sock --stderr-socket /home/alizj.linux/.local/share/zed/server_state/dev-workspace-176/stderr.sock
  54683     /home/alizj.linux/.local/share/zed/node/node-v22.5.1-linux-arm64/bin/node /home/alizj.linux/.local/share/zed/languages/json-language-server/node_modules/vscode-langservers-extracted/bin/vscode-json-language-server --stdio
```

同时 zed 在目标服务器上创建和保存了如下文件和目录：

```sh
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

1. 网络连接不上时，大量刷日志：

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

2. 按键问题

- "ctrl-cmd-d": "editor::DeleteToPreviousWordStart", // 不生效
- "cmd-q": "editor::Rewrap", // 自动折行，有问题，折行的长度不对。

3. 本地交叉编译 remote_server 报错

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
   This is free software; see the source for copying conditions. There is NO
   warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

   root@b4fce23c85a8:/app# which mold
   /usr/local/bin/mold
