# DECOMPILER RUBY FOR SKETCHUP RUBY 2.7
# Dự Án Dịch Ngược & Khôi Phục Mã Nguồn Plugin SketchUp Đang Sử Dụng rubyencoder (Không Áp Dụng Cho Các File .rbe)

Dự án này tập trung vào việc dịch ngược (decompile) bytecode Ruby YARV 2.7, khôi phục mã nguồn logic hoàn chỉnh của plugin SketchUp và thực hiện các quy trình kiểm thử tĩnh tự động để đảm bảo code sạch lỗi cú pháp (`Syntax OK`) và tuân thủ đúng SketchUp API.

---

## 1. Cấu Trúc Dự Án
### 🛠️ Bộ Công Cụ Dịch Ngược & Tối Ưu Core
* **decompiler.py**: Bộ dịch ngược trung tâm, thực hiện giả lập stack của Ruby VM 2.7 YARV và chuyển đổi bytecode ngược lại thành mã nguồn Ruby tương ứng.
* **auto_decompile.py**: Script tự động quét, dump bytecode và dịch ngược đồng loạt toàn bộ các file `.rb` mã hóa trong plugin.
* **replace_decompiled.py** Script tự động đồng bộ hóa các file Ruby dịch ngược từ thư mục tạm ra codebase làm việc chính.
---

## 2. Quy Trình Dịch Ngược & Đồng Bộ (Workflow)
* Yêu cầu cài đặt [Ruby+Devkit 2.7.0-1 (x64)](https://release-assets.githubusercontent.com/github-production-release-asset/78153411/386bbd80-2fc1-11ea-9af0-091632fb975f?sp=r&sv=2018-11-09&sr=b&spr=https&se=2026-06-16T07%3A00%3A02Z&rscd=attachment%3B+filename%3Drubyinstaller-devkit-2.7.0-1-x64.exe&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2026-06-16T05%3A59%3A33Z&ske=2026-06-16T07%3A00%3A02Z&sks=b&skv=2018-11-09&sig=CVd9XAUm8YzPVXKpOre1KBSw4c3Hbg7V%2Fm50eFDy%2FQY%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc4MTU5NDcwNywibmJmIjoxNzgxNTkxMTA3LCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.yiIlqfPdfaN68Por4Dk427KEKa2Jq1NZWv8yTW_pAfI&response-content-disposition=attachment%3B%20filename%3Drubyinstaller-devkit-2.7.0-1-x64.exe&response-content-type=application%2Foctet-stream)
* Để thực hiện dịch ngược lại từ bytecode gốc sang mã nguồn Ruby hoàn chỉnh. Mở file **auto_decompiler.py** chỉnh sửa lại TARGET_DIR và RUBY_PATH:
```
RUBY_PATH = r"D:\Ruby27-x64\bin\ruby.exe" # Chọn đường dẫn cài đặt ruby
TARGET_DIR = r"" # Plugin Mục tiêu vd: MyPlugin\my_plugin
```

* Chạy chuỗi lệnh sau trong powershell tại thư mục gốc của dự án:

```powershell
Dịch ngược đồng loạt toàn bộ các file từ bytecode
python auto_decompile.py
```
* Đồng bộ mã nguồn dịch ngược sang thư mục nghiệp vụ của plugin.
```
python replace_decompiled.py
```

---

## 3. Nguyên Lý Hoạt Động Của Decompiler (`decompiler.py`)

Bộ dịch ngược hoạt động dựa trên cơ chế phân tích tĩnh kết hợp với giả lập máy ảo Ruby YARV (Ruby 2.7):

### 3.1. Phân Tích Cấu Trúc ISeq (Instruction Sequence Parsing)
* Trích xuất thông tin metadata của phương thức/lớp từ tệp bytecode đã dump, bao gồm: bảng biến cục bộ (`local_table`), bảng xử lý biệt lệ (`catch_table`), số lượng tham số truyền vào, và danh sách chỉ thị bytecode chi tiết.

### 3.2. Giả Lập Ngăn Xếp (Stack Simulation)
* Vì Ruby YARV là máy ảo hoạt động dựa trên ngăn xếp (Stack-based VM), decompiler duy trì một `stack` giả lập (chứa các chuỗi biểu thức Ruby).
* Với mỗi chỉ thị bytecode, decompiler mô phỏng hành vi đẩy (push) hoặc rút (pop) tương ứng:
  * `putobject "hello"`: Đẩy `"hello"` lên stack.
  * `getlocal_WC_0 3`: Đọc tên biến tại vị trí thứ 3 trong bảng biến cục bộ và đẩy lên stack.
  * `opt_plus`: Pop hai biểu thức `a` và `b` từ stack và đẩy biểu thức gộp `"(a + b)"` ngược trở lại stack.

### 3.3. Tái Cấu Trúc Luồng Điều Khiển (Control Flow Reconstruction)
* Khi gặp các chỉ thị rẽ nhánh hoặc nhảy (`jump`, `branchunless`, `branchif`, `branchnil`), decompiler không dịch tuyến tính mà chuyển sang dịch đệ quy các nhánh bằng hàm `translate_range`:
  * Tạo các bản sao stack giả lập cho nhánh `then` và nhánh `else`.
  * Dịch thử nghiệm (lookahead) để xác định điểm hội tụ của các nhánh.
  * Ghép nối các nhánh thành cấu trúc điều khiển Ruby tương ứng: `if/else/end`, toán tử ba ngôi (`cond ? a : b`), toán tử logic ngắn mạch (`&&`, `||`), hoặc safe navigation (`&.`).

### 3.4. Dịch Đệ Quy Block & Phương Thức Con
* Đối với các chỉ thị định nghĩa block (`send` kèm block ISeq) hoặc định nghĩa phương thức (`definemethod`), decompiler thực hiện gọi đệ quy để khôi phục mã nguồn của block/phương thức đó, sau đó ghép nối chúng với ngữ cảnh cha bằng cú pháp block của Ruby (`do ... end` hoặc `{ ... }`).

---

## 4. Nguyên Lý Hoạt Động Của Bộ Trích Xuất Bytecode (`dumper.c`)

Để lấy được bytecode disassembly nguyên bản từ các tệp Ruby đã bị mã hóa hoặc nén (không thể đọc trực tiếp), dự án sử dụng bộ trích xuất viết bằng C (`dumper.c` biên dịch thành `dumper.so`):

### 4.1. Kỹ Thuật Đánh Chặn API (API Hooking)
* Sử dụng thư viện **MinHook** để can thiệp trực tiếp vào bảng export của DLL máy ảo Ruby 2.7 (`x64-msvcrt-ruby270.dll`).
* Đánh chặn hàm nội bộ **`rb_iseq_eval`** — hàm mà máy ảo Ruby bắt buộc phải gọi khi bắt đầu chạy một chuỗi chỉ thị ISeq (Instruction Sequence) mới nạp vào bộ nhớ.

### 4.2. Trích Xuất Bytecode Đệ Quy (Disassembly Extraction)
* Khi `rb_iseq_eval` bị kích hoạt, hàm hook `my_rb_iseq_eval` sẽ tạm thời giữ luồng điều khiển và truyền cấu trúc con trỏ `iseq` (kiểu `rb_iseq_t*`) sang hàm nội bộ **`rb_iseq_disasm`**.
* `rb_iseq_disasm` chuyển đổi toàn bộ cấu trúc dữ liệu ISeq nhị phân trong RAM thành chuỗi văn bản assembly mô tả chi tiết từng chỉ thị VM (YARV Assembly format).
* Dumper ghi chuỗi này trực tiếp vào file log **`dumped_code.txt`**.

### 4.3. Đảm Bảo Tính Trong Suốt (Transparent Execution)
* Sau khi trích xuất và ghi log, dumper gọi lại hàm gốc thông qua con trỏ `original_rb_iseq_eval(iseq)`.
* Điều này giúp quá trình trích xuất hoàn toàn trong suốt, không làm gián đoạn hay crash máy ảo Ruby, cho phép plugin chạy bình thường trong khi toàn bộ mã nguồn ẩn của nó bị lộ ra ở dạng assembly.

---

# RUBY DECOMPILER FOR SKETCHUP RUBY 2.7
# Reverse Engineering & Source Code Recovery Project for SketchUp Plugins Using rubyencoder (Not Applicable for .rbe Files)

This project focuses on decompiling Ruby YARV 2.7 bytecode, recovering the complete logic source code of SketchUp plugins, and performing automated static testing processes to ensure the code is syntax-error-free (`Syntax OK`) and properly complies with the SketchUp API.

---

## 1. Project Structure
### 🛠️ Core Decompilation & Optimization Toolkit
* **decompiler.py**: The central decompiler, which simulates the stack of the Ruby VM 2.7 YARV and converts bytecode back into the corresponding Ruby source code.
* **auto_decompile.py**: An automated script that scans, dumps bytecode, and batch-decompiles all encrypted `.rb` files within the plugin.
* **replace_decompiled.py**: An automated script that synchronizes the decompiled Ruby files from the temporary directory to the main working codebase.

---

## 2. Decompilation & Synchronization Workflow
* Requires installation of [Ruby+Devkit 2.7.0-1 (x64)](https://release-assets.githubusercontent.com/github-production-release-asset/78153411/386bbd80-2fc1-11ea-9af0-091632fb975f?sp=r&sv=2018-11-09&sr=b&spr=https&se=2026-06-16T07%3A00%3A02Z&rscd=attachment%3B+filename%3Drubyinstaller-devkit-2.7.0-1-x64.exe&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2026-06-16T05%3A59%3A33Z&ske=2026-06-16T07%3A00%3A02Z&sks=b&skv=2018-11-09&sig=CVd9XAUm8YzPVXKpOre1KBSw4c3Hbg7V%2Fm50eFDy%2FQY%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc4MTU5NDcwNywibmJmIjoxNzgxNTkxMTA3LCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.yiIlqfPdfaN68Por4Dk427KEKa2Jq1NZWv8yTW_pAfI&response-content-disposition=attachment%3B%20filename%3Drubyinstaller-devkit-2.7.0-1-x64.exe&response-content-type=application%2Foctet-stream)
* To perform decompilation from the original bytecode to complete Ruby source code, open the **auto_decompiler.py** file and edit `TARGET_DIR` and `RUBY_PATH`:

```python
RUBY_PATH = r"D:\Ruby27-x64\bin\ruby.exe" # Select the ruby installation path
TARGET_DIR = r"" # Target Plugin e.g.: MyPlugin\my_plugin
```

* Run the following command sequence in PowerShell at the root directory of the project:

```powershell
# Batch-decompile all files from bytecode
python auto_decompile.py
```

* Synchronize the decompiled source code to the plugin's working directory.

```powershell
python replace_decompiled.py
```

---

## 3. Operating Principle of the Decompiler (`decompiler.py`)

The decompiler operates based on static analysis combined with simulating the Ruby YARV virtual machine (Ruby 2.7):

### 3.1. Instruction Sequence (ISeq) Parsing
* Extracts metadata of methods/classes from the dumped bytecode file, including: the local variable table (`local_table`), the exception handling table (`catch_table`), the number of input parameters, and the detailed list of bytecode instructions.

### 3.2. Stack Simulation
* Since Ruby YARV is a Stack-based VM, the decompiler maintains a simulated `stack` (containing strings of Ruby expressions).
* For each bytecode instruction, the decompiler simulates the corresponding push or pop behavior:
  * `putobject "hello"`: Pushes `"hello"` onto the stack.
  * `getlocal_WC_0 3`: Reads the variable name at the 3rd position in the local variable table and pushes it onto the stack.
  * `opt_plus`: Pops two expressions `a` and `b` from the stack and pushes the combined expression `"(a + b)"` back onto the stack.

### 3.3. Control Flow Reconstruction
* When encountering branching or jumping instructions (`jump`, `branchunless`, `branchif`, `branchnil`), the decompiler does not translate linearly but switches to recursively translating the branches using the `translate_range` function:
  * Creates clones of the simulated stack for the `then` branch and the `else` branch.
  * Performs a trial translation (lookahead) to determine the convergence point of the branches.
  * Assembles the branches into corresponding Ruby control structures: `if/else/end`, ternary operator (`cond ? a : b`), short-circuit logic operators (`&&`, `||`), or safe navigation (`&.`).

### 3.4. Recursive Decompilation of Blocks & Sub-methods
* For instructions defining blocks (`send` with an ISeq block) or defining methods (`definemethod`), the decompiler makes recursive calls to recover the source code of that block/method, then attaches them to the parent context using Ruby's block syntax (`do ... end` or `{ ... }`).

---

## 4. Operating Principle of the Bytecode Extractor (`dumper.c`)

To obtain the raw bytecode disassembly from encrypted or compressed Ruby files (which cannot be read directly), the project uses an extractor written in C (`dumper.c` compiled into `dumper.so`):

### 4.1. API Hooking Technique
* Uses the **MinHook** library to directly intervene in the export table of the Ruby 2.7 virtual machine DLL (`x64-msvcrt-ruby270.dll`).
* Hooks the internal function **`rb_iseq_eval`** — the function that the Ruby virtual machine is forced to call when starting to execute a newly loaded ISeq (Instruction Sequence) in memory.

### 4.2. Recursive Bytecode Extraction (Disassembly Extraction)
* When `rb_iseq_eval` is triggered, the hook function `my_rb_iseq_eval` temporarily intercepts the control flow and passes the `iseq` pointer structure (type `rb_iseq_t*`) to the internal function **`rb_iseq_disasm`**.
* `rb_iseq_disasm` converts the entire binary ISeq data structure in RAM into an assembly text string describing each VM instruction in detail (YARV Assembly format).
* The dumper writes this string directly to the **`dumped_code.txt`** log file.

### 4.3. Ensuring Transparent Execution
* After extracting and logging, the dumper calls the original function back via the `original_rb_iseq_eval(iseq)` pointer.
* This makes the extraction process completely transparent, without interrupting or crashing the Ruby virtual machine, allowing the plugin to run normally while its entire hidden source code is exposed in assembly format.
