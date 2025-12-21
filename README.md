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

## 📝 PHƯ21/11/2025
