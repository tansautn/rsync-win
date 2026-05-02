# Phân tích kiến trúc: rsync-win

## 1. Tổng quan kiến trúc

Dự án là một **Rust CLI wrapper mỏng** giúp chạy `rsync` trên Windows. Thay vì port rsync sang Windows natively, nó **bundle sẵn các binary Cygwin64** bên trong thư mục `cygwin64/`, bao gồm:

```
cygwin64/
├── rsync.exe        ← rsync thực sự (Cygwin binary)
├── ssh.exe          ← SSH client (Cygwin binary)
├── cygpath.exe      ← công cụ convert path
├── cygwin1.dll      ← Cygwin runtime layer
└── *.dll            ← các dependency (OpenSSL, krb5, zstd, lz4...)
```

**Luồng thực thi:**

```
User chạy rsync-win.exe
    → Parse CLI args (clap)
    → Detect SSH path?
    → Convert tất cả paths (Windows → Unix) qua cygpath.exe
    → Build argument list
    → Spawn cygwin64/rsync.exe với args đã chuẩn bị
    → Inherit stdout/stderr (output trực tiếp ra terminal)
```

---

## 2. SSH Key & SSH Authentication

**Phát hiện SSH path** (`src/main.rs:94-100`):

```rust
fn is_ssh_path(path: &str) -> bool {
    if path.contains("rsync://") {
        false
    } else {
        path.contains('@')   // ← detect USER@HOST:/path
    }
}
```

> **Lưu ý**: Logic này khá naive — bất kỳ path nào chứa `@` đều bị coi là SSH path. Không validate `HOST:` pattern. Nếu bạn có file path chứa `@` (e.g., file version `v1@2`), nó sẽ bị nhận nhầm.

---

**Khi có SSH path** → gọi `prepare_rsync_options_with_ssh()` (`main.rs:102-137`):

**Bước 1 — Xác định identity file:**

```rust
// Default: ~/.ssh/id_rsa (dùng home crate để lấy home dir cross-platform)
fn default_identity_file() -> String {
    let mut path = home::home_dir().expect("get home dir failed");
    path.push(".ssh/id_rsa");
    path.to_str().unwrap().to_string()
}
```

Nếu user không pass `-i`, mặc định lấy `C:\Users\<username>\.ssh\id_rsa`.

**Bước 2 — Convert identity path sang Unix:**

```rust
let identity = path_win_to_unix(
    match &args.identity {
        Some(path) => path.clone(),
        None => default_identity_file(),
    }.as_str(),
);
// C:\Users\foo\.ssh\id_rsa → /cygdrive/c/users/foo/.ssh/id_rsa
```

**Bước 3 — Build SSH command string:**

```rust
format!(
    r#""{}" -o "StrictHostKeyChecking=no" -o "IdentitiesOnly=yes" -i "{}" -p {}"#,
    path_ssh().display(),    // ← Windows path với backslash!
    identity,                // ← Cygwin Unix path
    port,                    // ← default 22
)
```

Kết quả trở thành argument `-e` cho rsync:

```
-e "C:\...\cygwin64\ssh.exe" -o "StrictHostKeyChecking=no" -o "IdentitiesOnly=yes" -i "/cygdrive/c/users/foo/.ssh/id_rsa" -p 22
```

**Các flag SSH đáng chú ý:**

| Flag | Ý nghĩa | Trade-off |
|---|---|---|
| `StrictHostKeyChecking=no` | Không hỏi xác nhận host key lần đầu | **Bảo mật thấp hơn** — dễ bị MITM |
| `IdentitiesOnly=yes` | Chỉ dùng key đã chỉ định, bỏ qua ssh-agent | Tránh lỗi "too many auth failures" |

---

## 3. Xử lý bất đồng Windows path vs Unix path

Đây là phần cốt lõi nhất. Vì `rsync.exe` bundled là **Cygwin binary** nên nó **chỉ hiểu Unix path** (hoặc Cygwin path `/cygdrive/c/...`).

**Hàm convert** (`main.rs:180-195`) — dùng `cygpath.exe`:

```rust
fn path_win_to_unix(path: &str) -> String {
    Command::new(path_cygpath())
        .args(["-u", path])    // flag -u = to Unix format
        .output()
        .expect("failed to execute process");
    // ...trim và trả về string
}
```

`cygpath -u` xử lý các trường hợp:

| Input (Windows) | Output (Cygwin Unix) |
|---|---|
| `C:\Users\foo\data` | `/cygdrive/c/users/foo/data` |
| `C:/Users/foo/data` | `/cygdrive/c/users/foo/data` |
| `D:\backup\` | `/cygdrive/d/backup/` |
| `user@host:/remote/path` | `user@host:/remote/path` *(pass-through, không phải Windows path)* |

**Cái gì được convert, cái gì không:**

```
main() (main.rs:76-77):
    path_win_to_unix(&args.src)   ← convert src
    path_win_to_unix(&args.dest)  ← convert dest

prepare_rsync_options_with_ssh() (main.rs:125-131):
    path_win_to_unix(identity)    ← convert identity file path

KHÔNG convert:
    path_ssh().display()          ← path đến ssh.exe dùng Windows backslash!
    path_rsync()                  ← PathBuf, Windows format (dùng trực tiếp với Command::new)
```

**Bất nhất thú vị**: `ssh.exe` path trong `-e` argument vẫn là Windows path (vì dùng `PathBuf::display()` trên Windows → backslash), nhưng `identity` path lại là Cygwin path. Điều này hoạt động được vì Cygwin runtime (`cygwin1.dll`) có thể tự handle cả hai format khi được spawn từ Windows process.

---

## 4. Cách tìm bundled binaries (`main.rs:203-209`)

```rust
fn path_cygwin_dir() -> PathBuf {
    let mut dir = env::current_exe()  // lấy path của rsync-win.exe
        .expect("...");
    dir.pop();                         // bỏ filename, lấy thư mục cha
    dir.push("cygwin64");              // thêm cygwin64/
    dir
}
```

`rsync-win.exe` và `cygwin64/` **phải nằm cùng thư mục**. Nếu tách ra, tool sẽ fail ngay.

---

## 5. Sơ đồ tổng thể

```
rsync-win.exe
│
├── clap parse CLI args
│
├── is_ssh_path(src) || is_ssh_path(dest)?
│   ├── YES → prepare_rsync_options_with_ssh()
│   │           ├── default identity: home_dir()/.ssh/id_rsa
│   │           ├── path_win_to_unix(identity)  ← cygpath -u
│   │           └── build: -e "ssh.exe -o StrictHostKeyChecking=no
│   │                            -o IdentitiesOnly=yes -i <unix_path> -p 22"
│   └── NO  → prepare_rsync_options()
│
├── path_win_to_unix(src)   ← cygpath -u (pass-through nếu là SSH path)
├── path_win_to_unix(dest)  ← cygpath -u
│
└── spawn: cygwin64/rsync.exe <all_args>
              stdout/stderr → inherit (real-time output)
```

---

## 6. Điểm cần chú ý / Tiềm ẩn vấn đề

1. **`StrictHostKeyChecking=no`**: Luôn disable host verification → không an toàn trên mạng không tin cậy.
2. **SSH detection naive**: `@` trong path = SSH, có thể false positive với local path chứa `@`.
3. **`ssh.exe` path không convert**: Phụ thuộc vào Cygwin runtime tự handle Windows backslash — hoạt động nhờ `cygwin1.dll`, không phải thiết kế explicit.
4. **`cygpath` subprocess**: Mỗi path conversion spawn thêm 1 process `cygpath.exe` — với nhiều lần gọi, overhead có thể đáng kể.
5. **Không handle `rsync://` protocol cho SSH auth**: Nếu dùng `rsync://user@host/path`, không có SSH auth (by design nhưng không documented rõ).
