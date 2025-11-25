# TÀI LIỆU CUSTOMIZE NEOVIM - NHÓM 8
## Đồ án cuối kỳ môn Linux

---

## 📋 MỤC LỤC

1. [Giới thiệu](#giới-thiệu)
2. [Phân biệt 2 phương pháp customize](#phân-biệt-2-phương-pháp)
3. [Phương pháp 1: Configuration (Runtime Configuration)](#phương-pháp-1-runtime-configuration)
4. [Phương pháp 2: Source Code Modification](#phương-pháp-2-source-code-modification)
5. [Quy trình build và cài đặt](#quy-trình-build-và-cài-đặt)
6. [Tổng kết các tính năng đã implement](#tổng-kết)

---

## 🎯 GIỚI THIỆU

### Mục tiêu dự án
Customize và mở rộng chức năng của Neovim editor thông qua 2 phương pháp:
- **Runtime Configuration**: Cấu hình hành vi editor thông qua file config
- **Source Code Modification**: Thêm tính năng mới trực tiếp vào mã nguồn C

### Sự khác biệt cơ bản

| Khía cạnh | Runtime Configuration | Source Code Modification |
|-----------|----------------------|--------------------------|
| **Thuật ngữ** | User Configuration / Runtime Config | Core Modification / Upstream Patching |
| **File làm việc** | `~/.config/nvim/init.lua` | `src/nvim/*.c`, `src/nvim/*.h` |
| **Ngôn ngữ** | Lua (scripting) | C (compiled) |
| **Cần build lại?** | ❌ Không | ✅ Có (cmake + make) |
| **Phạm vi** | Giới hạn trong API có sẵn | Không giới hạn, có thể thêm API mới |
| **Hiệu năng** | Runtime overhead | Native performance |
| **Áp dụng** | Chỉ trên máy local | Toàn hệ thống sau khi install |

---

## 📝 PHƯƠNG PHÁP 1: RUNTIME CONFIGURATION

### 1.1. Khái niệm
**Runtime Configuration** là việc cấu hình Neovim thông qua các API và option có sẵn, **KHÔNG** thay đổi mã nguồn. Các thay đổi được load khi khởi động editor.

### 1.2. File cấu hình
```
~/.config/nvim/init.lua
```

### 1.3. Các tính năng đã cấu hình

#### ✅ 1.3.1. Basic Editor Settings
```lua
vim.wo.number = true              -- Bật số dòng
vim.opt.cursorline = true         -- Tô sáng dòng hiện tại
```

**Giải thích kỹ thuật:**
- Sử dụng Lua API: `vim.opt`, `vim.wo` (window options)
- Chỉ set giá trị của các option có sẵn trong Neovim
- Không thêm chức năng mới

---

#### ✅ 1.3.2. Indentation Configuration
```lua
vim.opt.tabstop = 4               -- Tab = 4 spaces
vim.opt.shiftwidth = 4            -- Indent width
vim.opt.softtabstop = 4           -- Soft tab width
vim.opt.expandtab = true          -- Tab → spaces
vim.opt.autoindent = true         -- Giữ indent dòng trước
vim.opt.smartindent = true        -- Auto-indent cho C/C++
```

**Mục đích:** Chuẩn hóa code style, dễ đọc và maintain.

---

#### ✅ 1.3.3. Window Navigation Keymaps
```lua
vim.keymap.set("n", "<C-h>", "<C-w>h", { desc = "Move to left window" })
vim.keymap.set("n", "<C-j>", "<C-w>j", { desc = "Move to bottom window" })
vim.keymap.set("n", "<C-k>", "<C-w>k", { desc = "Move to top window" })
vim.keymap.set("n", "<C-l>", "<C-w>l", { desc = "Move to right window" })
```

**Giải thích:**
- Mapping phím tắt: `Ctrl+h/j/k/l` để di chuyển giữa các split windows
- Sử dụng `vim.keymap.set()` API - **runtime remapping**
- Không sửa hành vi core của editor

---

#### ✅ 1.3.4. Custom Statusline (Lua Functions)

**Đây là phần phức tạp nhất của Runtime Config!**

##### Các hàm helper:
```lua
-- 1. Hiển thị Git branch hiện tại
local function git_branch()
  local branch = vim.fn.system("git branch --show-current 2>/dev/null | tr -d '\n'")
  if branch ~= "" then
    return "  " .. branch .. " "
  end
  return ""
end

-- 2. Icon cho file type
local function file_type()
  local ft = vim.bo.filetype
  local icons = {
    lua = "[LUA]", python = "[PY]", javascript = "[JS]",
    html = "[HTML]", css = "[CSS]", json = "[JSON]",
    markdown = "[MD]", vim = "[VIM]", sh = "[SH]",
  }
  return (icons[ft] or ft)
end

-- 3. Đếm số từ (cho markdown/text)
local function word_count()
  local ft = vim.bo.filetype
  if ft == "markdown" or ft == "text" or ft == "tex" then
    local words = vim.fn.wordcount().words
    return "  " .. words .. " words "
  end
  return ""
end

-- 4. Hiển thị kích thước file
local function file_size()
  local size = vim.fn.getfsize(vim.fn.expand('%'))
  if size < 0 then return "" end
  if size < 1024 then return size .. "B "
  elseif size < 1024 * 1024 then
    return string.format("%.1fK", size / 1024)
  else
    return string.format("%.1fM", size / 1024 / 1024)
  end
end

-- 5. Mode indicator với icon
local function mode_icon()
  local mode = vim.fn.mode()
  local modes = {
    n = "NORMAL", i = "INSERT", v = "VISUAL",
    V = "V-LINE", ["\22"] = "V-BLOCK", c = "COMMAND",
    R = "REPLACE", t = "TERMINAL"
  }
  return modes[mode] or "  " .. mode:upper()
end
```

##### Dynamic Statusline Setup:
```lua
local function setup_dynamic_statusline()
  -- Khi focus vào window: statusline đầy đủ
  vim.api.nvim_create_autocmd({"WinEnter", "BufEnter"}, {
    callback = function()
      vim.opt_local.statusline = table.concat {
        "  ", "%#StatusLineBold#", "%{v:lua.mode_icon()}",
        "%#StatusLine#", " │ %f %h%m%r", "%{v:lua.git_branch()}",
        " │ ", "%{v:lua.file_type()}", " | ", "%{v:lua.file_size()}",
        "%=", "%l:%c  %P ",
      }
    end
  })

  -- Khi blur window: statusline đơn giản
  vim.api.nvim_create_autocmd({"WinLeave", "BufLeave"}, {
    callback = function()
      vim.opt_local.statusline = "  %f %h%m%r │ %{v:lua.file_type()} | %=  %l:%c   %P "
    end
  })
end

setup_dynamic_statusline()
```

**Công nghệ sử dụng:**
- `vim.api.nvim_create_autocmd()` - Tạo event listener
- `vim.fn.system()` - Gọi shell command
- `vim.bo.filetype` - Buffer option (file type)
- `_G.function_name` - Expose Lua function cho VimScript

**Điểm mạnh:**
- 100% Lua, không cần plugin
- Dynamic: thay đổi theo context
- Tích hợp Git info real-time

---

### 1.4. Kết luận Phương pháp 1

✅ **Ưu điểm:**
- Dễ chỉnh sửa, test nhanh (không cần build lại)
- Portable (copy file config sang máy khác là chạy)
- An toàn, không làm hỏng Neovim core

❌ **Hạn chế:**
- Không thể thêm command mới vào Neovim
- Bị giới hạn bởi Lua API
- Không thể tối ưu performance ở tầng native

---

## 🔧 PHƯƠNG PHÁP 2: SOURCE CODE MODIFICATION

### 2.1. Khái niệm

**Source Code Modification** (hay **Upstream Patching**) là việc **chỉnh sửa trực tiếp mã nguồn C** của Neovim để:
- Thêm **native command** mới
- Thay đổi hành vi core
- Tối ưu performance
- Thêm feature không có trong API

**Sau khi sửa, phải:**
1. Compile lại toàn bộ source code
2. Install binary mới lên hệ thống

### 2.2. Cấu trúc thư mục source code

```
src/nvim/
├── api/
│   ├── api.c
│   ├── buffer.c
│   ├── window.c
│   ├── vim.c
│   └── private/
├── autocmd/
├── diff/
├── event/
│   ├── loop.c
│   ├── time.c
│   ├── process.c
│   ├── rstream.c
│   └── uds.c
├── eval/
│   ├── eval.c
│   ├── lua.c
│   ├── typval.c
│   └── vars.c
├── f/
├── fs/
├── generated/
├── highlight/
├── json/
├── lua/
├── mark.c
├── buffer.c
├── change.c
├── cmdexpand.c
├── cmds.c
├── cursor.c
├── decorators.c
├── diff.c
├── drawscreen.c
├── edit.c
├── ex_docmd.c
├── ex_eval.c
├── fileio.c
├── getchar.c
├── main.c
├── map.c
├── message.c
├── misc1.c
├── normal.c
├── ops.c
├── option.c
├── popupmenu.c
├── quickfix.c
├── search.c
├── syntax.c
├── spell.c
├── terminal.c
├── textformat.c
├── ui.c
├── undo.c
└── window.c
```

```
neovim/src/nvim/
├── ex_cmds.c          ← Implementation của Ex commands
├── ex_cmds.h          ← Header declarations
├── ex_cmds.lua        ← Command registry
├── event/
│   ├── time.c         ← Timer system
│   └── loop.c         ← Event loop
├── eval.c             ← Expression evaluation
└── main.c             ← Entry point
```

---

### 2.3. Các tính năng đã implement

---

#### 🚀 2.3.1. HelloVim Command

**Mục đích:** Demo command đơn giản nhất

##### File: `src/nvim/ex_cmds.c`
```c
void ex_hellovim(exarg_T *eap) {
    msg("Welcome to our project. We are group 8!", 0);
}
```

**Giải thích kỹ thuật:**
- `exarg_T *eap` - Struct chứa thông tin về Ex command
- `msg(...)` - API native để hiển thị message
- Function signature phải đúng chuẩn để được gọi từ command registry

##### File: `src/nvim/ex_cmds.lua`
```lua
{
  command = "HelloVim",
  func = "ex_hellovim",     -- Tên hàm C
  flags = 0,                -- Không có flag đặc biệt
  addr_type = "ADDR_NONE"   -- Không nhận range (như :1,10HelloVim)
}
```

**Command registry mapping:**
- Khi user gõ `:HelloVim`, Neovim gọi hàm C `ex_hellovim()`
- Lua file này được parse lúc build time

##### File: `src/nvim/ex_cmds.h`
```c
void ex_hellovim(exarg_T *eap);
```

**Usage:**
```vim
:HelloVim
" Output: Welcome to our project. We are group 8!
```

---

#### 🔢 2.3.2. Toggle Line Number Command

**Mục đích:** Bật/tắt line number nhanh bằng 1 command

##### Implementation (`ex_cmds.c`):
```c
void ex_togglenumber(exarg_T *eap) {
    // Đảo trạng thái line number
    curwin->w_p_nu = !curwin->w_p_nu;
    
    // Tắt relative number
    curwin->w_p_rnu = false;
    
    // Yêu cầu redraw window
    redraw_later(curwin, UPD_NOT_VALID);
    
    // Thông báo
    if (curwin->w_p_nu) {
        msg("Line numbers ON", 0);
    } else {
        msg("Line numbers OFF", 0);
    }
}
```

**Giải thích kỹ thuật:**
- `curwin` - Global pointer đến window hiện tại
- `w_p_nu` - Window property: number (bool)
- `w_p_rnu` - Window property: relative number
- `redraw_later()` - Schedule redraw để UI update
- `UPD_NOT_VALID` - Flag: redraw toàn bộ window

##### Registry (`ex_cmds.lua`):
```lua
{
  command = "Num",
  func = "ex_togglenumber",
  flags = 0,
  addr_type = "ADDR_NONE"
}
```

**Usage:**
```vim
:Num    " Toggle line numbers
```

---

#### ⚡ 2.3.3. Compile And Run (CAR) Command

**Mục đích:** F5-style compilation - compile + run trong 1 command

**Đây là feature phức tạp nhất! Multi-language support với auto-detection.**

##### Kiến trúc tổng thể:

```
User gõ :CAR
    ↓
1. Lưu file hiện tại
    ↓
2. Detect language từ file extension
    ↓
3. Parse file path (tách dir, filename, extension)
    ↓
4. Generate compile command theo language
    ↓
5. Execute compile
    ↓
6. Mở terminal split và run
```

##### 2.3.3.1. Language Detection
```c
static const char* detect_language(const char *fname) {
    const char *ext = strrchr(fname, '.');  // Tìm dấu '.' cuối cùng
    if (ext == NULL) return NULL;
    
    if (strcmp(ext, ".cpp") == 0 || strcmp(ext, ".cc") == 0 || strcmp(ext, ".cxx") == 0) {
        return "cpp";
    } else if (strcmp(ext, ".c") == 0) {
        return "c";
    } else if (strcmp(ext, ".py") == 0) {
        return "python";
    } else if (strcmp(ext, ".java") == 0) {
        return "java";
    } else if (strcmp(ext, ".go") == 0) {
        return "go";
    } else if (strcmp(ext, ".rs") == 0) {
        return "rust";
    }
    return NULL;
}
```

**Kỹ thuật:**
- `strrchr()` - String search từ phải sang (tìm extension)
- `strcmp()` - So sánh string an toàn
- Support: C, C++, Python, Java, Go, Rust

---

##### 2.3.3.2. Main Implementation
```c
void ex_compileandrun(exarg_T *eap) {
    // 1. Auto-save file
    do_cmdline_cmd("write");
    
    // 2. Lấy file path của buffer hiện tại
    char *fname = curbuf->b_ffname;  // Full file path
    if (fname == NULL) {
        emsg("❌ No file name");
        return;
    }
    
    // 3. Detect language
    const char *lang = detect_language(fname);
    if (lang == NULL) {
        emsg("❌ Unsupported file type. Supported: .c, .cpp, .py, .java, .go, .rs");
        return;
    }
    
    // 4. Parse path: tách directory và filename
    char dir[1024] = {0}, base[256] = {0}, out[1024] = {0};
    strncpy(dir, fname, sizeof(dir) - 1);  // Copy an toàn
    
    char *slash = strrchr(dir, '/');  // Tìm slash cuối
    if (slash) {
        *slash = '\0';                          // Terminate string tại slash → dir
        strncpy(base, slash + 1, sizeof(base) - 1);  // filename
    } else {
        dir[0] = '\0';                          // File ở current dir
        strncpy(base, fname, sizeof(base) - 1);
    }
    
    // 5. Bỏ extension để có tên executable
    char *dot = strrchr(base, '.');
    if (dot) *dot = '\0';  // "test.cpp" → "test"
    
    // 6. Tạo output path
    if (dir[0] != '\0') {
        snprintf(out, sizeof(out), "%s/%s", dir, base);  // "/path/to/test"
    } else {
        snprintf(out, sizeof(out), "%s", base);          // "test"
    }
    
    // 7. Generate compile/run commands theo language
    char compile_cmd[2048] = {0};
    char run_cmd[2048] = {0};
    bool needs_compile = true;
    
    if (strcmp(lang, "cpp") == 0) {
        msg("🔨 Compiling C++ code...", 0);
        snprintf(compile_cmd, sizeof(compile_cmd), 
                 "!g++ -std=c++17 -Wall -Wextra \"%s\" -o \"%s\" 2>&1", fname, out);
        snprintf(run_cmd, sizeof(run_cmd), "split | terminal \"%s\"", out);
        
    } else if (strcmp(lang, "c") == 0) {
        msg("🔨 Compiling C code...", 0);
        snprintf(compile_cmd, sizeof(compile_cmd), 
                 "!gcc -std=c11 -Wall -Wextra \"%s\" -o \"%s\" 2>&1", fname, out);
        snprintf(run_cmd, sizeof(run_cmd), "split | terminal \"%s\"", out);
        
    } else if (strcmp(lang, "python") == 0) {
        needs_compile = false;  // Python không cần compile!
        msg("🐍 Running Python...", 0);
        snprintf(run_cmd, sizeof(run_cmd), "split | terminal python3 \"%s\"", fname);
        
    } else if (strcmp(lang, "java") == 0) {
        msg("☕ Compiling Java...", 0);
        snprintf(compile_cmd, sizeof(compile_cmd), "!javac \"%s\" 2>&1", fname);
        // Java cần classpath
        snprintf(run_cmd, sizeof(run_cmd), "split | terminal java -cp \"%s\" %s", 
                 dir[0] ? dir : ".", base);
        
    } else if (strcmp(lang, "go") == 0) {
        needs_compile = false;  // Go run = compile + execute
        msg("🏃 Running Go...", 0);
        snprintf(run_cmd, sizeof(run_cmd), "split | terminal go run \"%s\"", fname);
        
    } else if (strcmp(lang, "rust") == 0) {
        msg("🦀 Compiling Rust...", 0);
        snprintf(compile_cmd, sizeof(compile_cmd), 
                 "!rustc \"%s\" -o \"%s\" 2>&1", fname, out);
        snprintf(run_cmd, sizeof(run_cmd), "split | terminal \"%s\"", out);
    }
    
    // 8. Execute compile (nếu cần)
    if (needs_compile) {
        do_cmdline_cmd(compile_cmd);  // Blocking call
        msg("✅ Compilation complete! Running...", 0);
    }
    
    // 9. Mở terminal split và run
    do_cmdline_cmd(run_cmd);
    msg("🚀 Program started in terminal", 0);
}
```

**Công nghệ nâng cao:**
- `curbuf` - Global pointer: current buffer
- `b_ffname` - Buffer property: full file name
- `do_cmdline_cmd()` - Execute Ex command từ C code
- `strncpy()` - Safe string copy (tránh buffer overflow)
- `snprintf()` - Format string an toàn với size limit
- `emsg()` - Error message API

**Compiler flags explanation:**
- `-std=c++17` / `-std=c11` - Language standard
- `-Wall -Wextra` - Enable all warnings
- `2>&1` - Redirect stderr → stdout để xem lỗi compile

##### Registry (`ex_cmds.lua`):
```lua
{
  command = 'CAR',
  func = 'ex_compileandrun',
  flags = 0,
  addr_type = "ADDR_NONE"
}
```

**Usage:**
```vim
:CAR    " Compile và run file hiện tại
```

**Demo flow:**
```
File: test.cpp
User: :CAR
Output: 
  🔨 Compiling C++ code...
  ✅ Compilation complete! Running...
  🚀 Program started in terminal
  [Terminal split mở ra với output]
```

---

#### 💾 2.3.4. Auto-Save Feature

**Mục đích:** Tự động lưu file mỗi 5 giây (như VSCode)

**Đây là feature sử dụng Event Loop system của Neovim!**

##### Kiến trúc:

```
Neovim Event Loop (libuv-based)
    ↓
TimeWatcher (timer wrapper)
    ↓
Callback function (mỗi 5s)
    ↓
Check buffer modified → Save
```

##### 2.3.4.1. Includes cần thiết
```c
#include "nvim/event/time.h"   // TimeWatcher API
#include "nvim/event/loop.h"   // Main loop
```

##### 2.3.4.2. Global State
```c
static TimeWatcher autosave_timer;              // Timer object
static bool autosave_enabled = false;           // Trạng thái on/off
static bool autosave_timer_initialized = false; // Init flag
```

**Design pattern:** Singleton timer với lazy initialization

---

##### 2.3.4.3. Timer Callback
```c
static void autosave_callback(TimeWatcher *tw, void *data) {
    (void)tw;    // Unused parameter
    (void)data;
    
    // Guard: buffer phải tồn tại và đã modified
    if (curbuf == NULL || !bufIsChanged(curbuf)) {
        return;
    }
    
    // Guard: buffer phải có tên file (không phải [No Name])
    if (curbuf->b_ffname == NULL || curbuf->b_ffname[0] == '\0') {
        return;
    }
    
    // Tắt message scrolling (UX: không làm phiền user)
    int save_msg_scroll = msg_scroll;
    msg_scroll = false;
    
    // Lưu file
    buf_write_all(curbuf, false);
    
    // Thông báo nhẹ nhàng
    msg("💾 Auto-saved", 0);
    
    // Khôi phục scroll setting
    msg_scroll = save_msg_scroll;
}
```

**Kỹ thuật:**
- `bufIsChanged()` - Check buffer modified flag
- `buf_write_all()` - Write buffer to disk (native API)
- `msg_scroll` - Control message behavior
- Early return pattern để optimize

---

##### 2.3.4.4. Start Auto-Save
```c
void ex_autosave_start(exarg_T *eap) {
    (void)eap;
    
    // Tránh start 2 lần
    if (autosave_enabled) {
        msg("⚠️  Auto-save is already running!", 0);
        return;
    }
    
    // Lazy initialization (chỉ init 1 lần)
    if (!autosave_timer_initialized) {
        time_watcher_init(&main_loop, &autosave_timer, NULL);
        autosave_timer.events = main_loop.events;  // Attach to main event queue
        autosave_timer.blockable = true;           // Có thể block nếu cần
        autosave_timer_initialized = true;
    }
    
    // Start timer: callback sau 5000ms, repeat mỗi 5000ms
    time_watcher_start(&autosave_timer, autosave_callback, 5000, 5000);
    autosave_enabled = true;
    
    msg("✅ Auto-save enabled! Files will be saved every 5 seconds", 0);
}
```

**Giải thích Event Loop:**
- `main_loop` - Global libuv event loop của Neovim
- `time_watcher_init()` - Khởi tạo timer wrapper
- `time_watcher_start(timer, callback, delay, repeat)` - Start periodic timer
- `events` queue - Nơi callback được enqueue
- `blockable = true` - Timer không block main thread

---

##### 2.3.4.5. Stop Auto-Save
```c
void ex_autosave_stop(exarg_T *eap) {
    (void)eap;
    
    if (!autosave_enabled) {
        msg("⚠️  Auto-save is not running!", 0);
        return;
    }
    
    time_watcher_stop(&autosave_timer);  // Stop timer
    autosave_enabled = false;
    
    msg("🛑 Auto-save disabled", 0);
}
```

---

##### 2.3.4.6. Toggle Auto-Save
```c
void ex_autosave_toggle(exarg_T *eap) {
    if (autosave_enabled) {
        ex_autosave_stop(eap);
    } else {
        ex_autosave_start(eap);
    }
}
```

**Design pattern:** Proxy function để toggle state

---

##### Registry (`ex_cmds.lua`):
```lua
{
  command = 'AutoSaveStart',
  func = 'ex_autosave_start',
  flags = 0,
  addr_type = "ADDR_NONE"
},
{
  command = 'AutoSaveStop',
  func = 'ex_autosave_stop',
  flags = 0,
  addr_type = "ADDR_NONE"
},
{
  command = 'AutoSave',
  func = 'ex_autosave_toggle',
  flags = 0,
  addr_type = "ADDR_NONE"
}
```

**Usage:**
```vim
:AutoSaveStart    " Bật auto-save
" [Sau 5 giây...]
" 💾 Auto-saved

:AutoSaveStop     " Tắt
:AutoSave         " Toggle nhanh
```

---

### 2.4. Header Declarations

**File: `src/nvim/ex_cmds.h`**

```c
// Custom commands
void ex_hellovim(exarg_T *eap);
void ex_togglenumber(exarg_T *eap);
void ex_compileandrun(exarg_T *eap);

// Auto-save functions
void ex_autosave_start(exarg_T *eap);
void ex_autosave_stop(exarg_T *eap);
void ex_autosave_toggle(exarg_T *eap);
```

**Tại sao cần header file?**
- C requires forward declaration
- Cho phép các file khác gọi functions này
- Build system sẽ check type safety

---

## 🔨 QUY TRÌNH BUILD VÀ CÀI ĐẶT

### 3.1. Cấu trúc thư mục dự án
```
neovim/
├── build/                  ← Build output directory
├── src/nvim/               ← Source code (đây là nơi ta sửa)
│   ├── ex_cmds.c           ← Modified
│   ├── ex_cmds.h           ← Modified
│   └── ex_cmds.lua         ← Modified
├── CMakeLists.txt          ← Build config
└── ~/.config/nvim/
    └── init.lua            ← Runtime config (không liên quan build)
```

---

### 3.2. Quy trình Build từ Source

#### Bước 1: Cài đặt dependencies
```bash
# Ubuntu/Debian
sudo apt-get install ninja-build gettext cmake unzip curl build-essential

# Hoặc
sudo apt-get install gcc g++ cmake ninja-build libtool
```

#### Bước 2: Clone source code (nếu chưa có)
```bash
git clone https://github.com/neovim/neovim.git
cd neovim
```

#### Bước 3: Sửa source code
```bash
# Edit files:
vim src/nvim/ex_cmds.c
vim src/nvim/ex_cmds.h
vim src/nvim/ex_cmds.lua
```

#### Bước 4: Configure build
```bash
cmake -B build -S . -G Ninja
```

**Giải thích:**
- `-B build` - Output directory
- `-S .` - Source directory (current)
- `-G Ninja` - Use Ninja build system (faster than Make)

#### Bước 5: Build
```bash
cmake --build build
```

**Hoặc dùng Ninja trực tiếp:**
```bash
cd build
ninja
```

**Thời gian:** ~5-15 phút (tùy máy)

**Output:** `build/bin/nvim` - Binary executable

---

### 3.3. Test Local (không cần install)

```bash
./build/bin/nvim test.txt
```

Trong Neovim:
```vim
:HelloVim
:Num
:CAR
:AutoSave
```

---

### 3.4. Install lên hệ thống (Global)

#### Cách 1: CMake install
```bash
cd neovim
sudo cmake --install build
```

**Default install path:** `/usr/local/bin/nvim`

#### Cách 2: Ninja install
```bash
cd neovim/build
sudo ninja install
```

#### Verify installation:
```bash
which nvim
# Output: /usr/local/bin/nvim

nvim --version
# Output: NVIM v0.x.x-dev+<commit>
```

---

### 3.5. Uninstall (nếu cần)

```bash
cd neovim/build
sudo cmake --build . --target uninstall
```

Hoặc:
```bash
sudo rm /usr/local/bin/nvim
sudo rm -rf /usr/local/share/nvim
```

---

### 3.6. Rebuild sau khi sửa code

```bash
# Nếu chỉ sửa .c/.h file (không thêm file mới):
cd neovim
cmake --build build

# Nếu thêm file mới hoặc sửa CMakeLists.txt:
cd neovim
rm -rf build
cmake -B build -G Ninja
cmake --build build
```

---

## 📊 TỔNG KẾT CÁC TÍNH NĂNG

### Bảng so sánh

| Feature | Type | Files Modified | Complexity | Lines of Code |
|---------|------|---------------|------------|---------------|
| **Basic Settings** | Runtime Config | init.lua | ⭐ Easy | ~10 |
| **Indentation** | Runtime Config | init.lua | ⭐ Easy | ~7 |
| **Window Navigation** | Runtime Config | init.lua | ⭐ Easy | ~4 |
| **Custom Statusline** | Runtime Config | init.lua | ⭐⭐⭐ Medium | ~80 |
| **HelloVim Command** | Source Mod | ex_cmds.c/h/lua | ⭐ Easy | ~5 |
| **Toggle Number** | Source Mod | ex_cmds.c/h/lua | ⭐⭐ Easy-Medium | ~15 |
| **Compile & Run** | Source Mod | ex_cmds.c/h/lua | ⭐⭐⭐⭐ Hard | ~120 |
| **Auto-Save** | Source Mod | ex_cmds.c/h/lua | ⭐⭐⭐⭐⭐ Very Hard | ~80 |

---

### Tech Stack Summary

#### Runtime Configuration
- **Language:** Lua 5.1
- **APIs Used:** 
  - `vim.opt`, `vim.wo`, `vim.bo`
  - `vim.keymap.set()`
  - `vim.api.nvim_create_autocmd()`
  - `vim.fn.system()`, `vim.fn.wordcount()`

#### Source Code Modification
- **Language:** C (C11 standard)
- **Build System:** CMake + Ninja
- **Libraries:**
  - libuv (event loop)
  - Standard C library
- **Neovim APIs:**
  - `msg()`, `emsg()` - Messaging
  - `do_cmdline_cmd()` - Execute commands
  - `buf_write_all()` - Buffer I/O
  - `time_watcher_*()` - Timer system
  - `redraw_later()` - UI updates

---

## 🎓 THUẬT NGỮ QUAN TRỌNG

### Cho Runtime Configuration:
- **init.lua** - Initialization file, entry point cho config
- **Lua API** - Interface giữa Lua và Neovim core
- **Autocommand** - Event listener (on WinEnter, BufWrite, etc.)
- **Keymap** - Keyboard mapping/binding
- **Option** - Built-in setting (number, tabstop, etc.)
- **Statusline** - Status bar ở dưới cùng window

### Cho Source Code Modification:
- **Upstream patch** - Sửa code gốc của project
- **Ex command** - Command line command (bắt đầu với `:`)
- **exarg_T** - Struct chứa argument của Ex command
- **Buffer** - In-memory representation của file
- **Window** - Viewport để hiển thị buffer
- **Event loop** - Main thread xử lý events (dùng libuv)
- **TimeWatcher** - Timer wrapper của Neovim
- **Callback** - Function được gọi khi event xảy ra
- **Build system** - Tool để compile code (CMake, Ninja)
- **Binary** - File executable sau khi compile

---

## 📝 HƯỚNG DẪN SỬ DỤNG CHO NHÓM

### Setup môi trường:

```bash
# 1. Clone repo
git clone https://github.com/neovim/neovim.git
cd neovim

# 2. Apply các thay đổi của nhóm (copy files đã sửa)
# hoặc patch từ git diff

# 3. Build
cmake -B build -G Ninja
cmake --build build

# 4. Test
./build/bin/nvim

# 5. (Optional) Install global
sudo cmake --install build
```

### Copy Runtime Config:
```bash
mkdir -p ~/.config/nvim
cp init.lua ~/.config/nvim/
nvim  # Reload để thấy thay đổi
```

---

## 🎬 DEMO CHO THUYẾT TRÌNH

### Script demo:

```bash
# Terminal 1: Show source code
cd neovim/src/nvim
cat ex_cmds.c | grep -A 20 "ex_compileandrun"

# Terminal 2: Build
cd neovim
time cmake --build build

# Terminal 3: Run & Demo
./build/bin/nvim demo.cpp

# Trong Neovim:
:HelloVim           # → "Welcome to our project. We are group 8!"
:Num                # Toggle line numbers
:AutoSaveStart      # Bật auto-save
# [Edit file, đợi 5s] → "💾 Auto-saved"
:CAR                # Compile & run C++
```

---

## 🔍 SO SÁNH 2 PHƯƠNG PHÁP - SUMMARY

### Runtime Configuration (Lua)
✅ **Khi nào dùng:**
- Customize editor behavior (colors, keys, options)
- Tạo statusline, plugins đơn giản
- Không cần performance cao

❌ **Không thể:**
- Thêm Ex command mới
- Thay đổi core behavior
- Access low-level APIs

### Source Code Modification (C)
✅ **Khi nào dùng:**
- Thêm native command mới
- Cần performance tối ưu
- Modify core logic
- Tích hợp với system (timers, I/O)

❌ **Nhược điểm:**
- Phải compile lại (lâu)
- Khó maintain (conflict khi upstream update)
- Risk: có thể làm crash editor

---

## 📚 KẾT LUẬN

Qua project này, nhóm đã demonstrate được:

1. **Runtime Configuration:**
   - Lua scripting skills
   - Hiểu Neovim API
   - UI/UX customization (statusline)

2. **Source Code Modification:**
   - C programming trong real-world codebase
   - Build system (CMake, Ninja)
   - Event-driven programming (timers, callbacks)
   - String manipulation an toàn (strncpy, snprintf)
   - Multi-language support (polyglot programming)

3. **Software Engineering:**
   - Code organization (separation of concerns)
   - Documentation
   - Testing và debugging

**Học được:**
- Linux development workflow
- Open source contribution process
- System programming với C
- Editor internals

---

## 📖 TÀI LIỆU THAM KHẢO

- Neovim Source: https://github.com/neovim/neovim
- Neovim API Docs: `:help api`
- Lua Guide: `:help lua-guide`
- Build Docs: https://github.com/neovim/neovim/wiki/Building-Neovim
- CMake Tutorial: https://cmake.org/cmake/help/latest/guide/tutorial/

---

**Tài liệu được tạo bởi:** Nhóm 8  
**Môn:** Linux và phần mềm mã nguồn mở
**Năm học:** 2024-2025  
**Ngày cập nhật:** 22/11/2025
