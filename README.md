# GDB/GEF

`pattern create <số byte>` : Tạo byte

`pattern search <địa chỉ>` : Tìm offset 

`search-pattern <chuỗi>` : Tìm chuỗi( ví dụ chuỗi /bin/sh)

`set $rip = ...` : Để nhảy tới lệnh ... bỏ qua các lệnh trước nó

# Gadget<Tìm đoạn gadget nhỏ>:

`ROPgadget --binary <tên file> | grep "<đoạn cần tìm>"`

# Ret2Shellcode cần leak:
* Cách để leak : Thông qua hàm read vì hàm read sẽ ko tự có \0 để kết thúc nên khi enter sẽ bị nối thêm ở đằng sau từ đó lợi dụng để -> địa chỉ muốn leak

* Thường Leak stack save rbp

* Để in ra stack_leak : `stack_leak = u64(p.recv(6))`

                         log.info("Stack_leak : " + hex(stack_leak))

* Overwrite được RIP ở hàm nào thì viết shellcode ở hàm đó

*<img width="423" height="52" alt="image" src="https://github.com/user-attachments/assets/d5c1b9bb-a93c-4e3d-835e-227cff7b2375" />
thay vì đếm shellcode chiếm bao nhiêu byte thì ta có thể dùng hàm ljust để hàm shellcode + với byte tự tạo thêm do hàm tạo ra để bằng tham số truyền vào tương đương tràn biến và đè Saved RIP

* Hàm read khi nhập dữ liệu ko tự động thêm byte NULL nên khi printf sẽ bị nối chuỗi kế

* Hạn chế có NULL byte 

# Ret2libc :

- Nên thêm p64(ret) trước mỗi khi gọi hàm thuộc libc

`got`: Nơi chứa địa chỉ hàm của libc

`plt`: Thực thi hàm được chứa trong got

`exe.got['...']` : Dán địa chỉ hàm của libc

`exe.plt['...']` : Thực thi hàm của libc

* Thường libc leak cần phải chạy lại hàm main ở script

* libc base là libc có địa chỉ nhỏ nhất

* `libc.sym['...']` luôn là offset của libc_leak và libc_base

* `libc.address<libc base> = libc_leak - libc.sym['...']`

* `next(libc.search(b'/bin/sh'))` luôn là offset giữa chuỗi /bin/sh và libc base, cái này hoạt tự lấy libc base + offset để tìm ra chuỗi /bin/sh

*Cụ thể : <img width="436" height="27" alt="image" src="https://github.com/user-attachments/assets/6d02ab3c-9567-4745-87be-9d5bc1b7c3b5" /> 

- Lúc này libc base đang = 0 nên địa chỉ của puts( là = libc base + offset) chính là offset luôn

*Còn khi này : <img width="681" height="376" alt="image" src="https://github.com/user-attachments/assets/39f95e44-9186-4e97-8c10-f0b8637a72bd" />
- Ta tìm được libc base rồi thì libc.sym['system'] sẽ là địa chỉ system do nó tự lấy libc base + offset luôn và chuỗi /bin/sh tương tự nhưng khác cú pháp 

# Stack pivot khai thác save RBP :

- Hàm `leave` chính là : `mov rsp, rbp`(dọn rác là các biến khởi tạo khi mới tạo frame) sau đó `pop rbp` để rsp + 8( mục đích là trỏ vào saved RIP để thực hiện return về ban đầu)

# Stack pivot đổi biến :

- Khi trở về hàm cha saved RBP của hàm con sẽ trở lại thành RBP hàm cha

# So sánh 32bit & 64bit

| 32                                                         | 64                                            |
|------------------------------------------------------------|-----------------------------------------------|
| ebx(arg1), ecx(arg2), edx(arg3)                            | rdi(arg1), rsi(arg2), rdx(arg3)               |
| Set arg1 `/bin/sh\0` cho execve phải push 0 -> //sh -> /bin| Set arg 1 `/bin/sh\0` cho execve push thẳng   |
| `int 0x80` = gọi syscall                                   | syscall                                       |
| Lấy arg từ stack                                           | Lấy arg từ thanh ghi                          |
| Layout : saved RIP                                         | Layout : saved RIP
           Hàm trả về xong khi chạy xong hàm mình đã overwrite          pop RDI + arg
           arg                                                          Hàm muốn chạy( với tham số được truyền ở trên)
                                                                        Hàm trả về khi chạy xong hàm muốn chạy

* Cách chuyển từ mã asm -> machine code(op code) để biết đường né NULL byte :

`from pwn import *`

`print(asm("mov eax, 0xb", arch='i386').hex())`

# Docker :

- Tạo sever local:

`sudo docker build . -t <image_name>`

`sudo docker run -d --rm -p <port>:<port> -it <container_name>`

- Hủy các container :

`sudo docker stop $(sudo docker ps -q)`

# Lỗi XMM : Thêm p64(ret) giúp căn chỉnh thành chia hết 16

# One_gadget : Ưu tiên dùng, nếu ko được thì dùng ret2libc ( đều cần leak libc base)

- one_gadget `"đường dẫn tuyệt đối tới libc"` : Để tìm các gadget và constraints(điều kiện để nhảy đến gadget) có thể xài

- `one_gadget = libc base + offset ( là số đầu tiên hiện ra tại mỗi gadget khi dùng tool` 

- Để kiểm tra constraints thỏa mãn chưa : `Set breakpoints` tại one_gadget( là libc base + offset) rồi `info registers` check các thanh ghi và `vmmap` check các vùng có thể viết mà constraints yêu cầu

# `info symbol <address>` Để check xem địa chỉ leak ra là của hàm nào

# FORMAT STRING :

- %p in ra số tại thanh ghi đó luôn (cần truyền vào địa chỉ) <trái>

- %c in ra 1byte số tại thanh ghi đó luôn (cần truyền vào biến thường) <trái>

- %s in ra giá trị mà số tại thanh ghi trỏ tới đến khi gặp NULL Byte (cần truyền vào địa chỉ) <phải>

- %n đếm số byte đã được in ra trước đó rồi ghi vào giá trị mà số tại thanh ghi đang trỏ tới <phải>

- 64 bit khi không truyền đầu vào thì 5% đầu là 5 thanh ghi lần lượt là : rsi rdx rcx r8 r9, % thứ 6 là dữ liệu trên stack

- %p%p%p%p... = %<n>$p in ra % THỨ n

- `for i in range(a, b)` : Vòng lặp chạy từ a đến b - 1

- `f'...{i}...'` : Ví dụ i = 9 thì nhúng giá trị `9` vào thay vì nhúng `i` như bình thường

- `p.close()` : Đóng kết nối đến process

- %<n>$n ghi vào số byte đã được in ra trước đó vào 4byte tại % THỨ n

- %<n>$hn ghi vào số byte đã được in ra trước đó vào 2byte tại % THỨ n

- %<n>$hhn ghi vào số byte đã được in ra trước đó vào 1byte tại % THỨ n

- %<n>c = bình thường là in ra 1byte nhưng thêm 1 số n phía trước thì in ra n byte <padding>

## -> combo %c và %n kết hợp dạng rút gọn %<n>c%<m>$n : c sẽ tạo padding n byte và %n sẽ in ra n byte theo dạng hex tại % THỨ m

- `[:n]` : Số dương bỏ n byte từ trái sang, số âm bỏ n byte từ phải sang

<img width="1146" height="1142" alt="image" src="https://github.com/user-attachments/assets/f5b9adec-02fd-4251-84f5-1844a209b93e" />


- `p.recvuntil(b'...', drop=True)`: Nhận đến byte ... và bỏ đi byte ...

- Chạy script thêm NOASLR để nó tĩnh hết để dễ dàng debug

- Hàm printf dừng khi gặp NULL byte vì vậy khi muốn gửi 1 địa chỉ lên stack để dùng formatstring thì payload sẽ gửi cuối bởi địa chị có dạng 0x00... có NULL byte ở cuối theo little endian, trước đó nên là formatstring để in ra chuỗi hoặc địa chỉ này sau <img width="432" height="95" alt="image" src="https://github.com/user-attachments/assets/c2a36418-21c6-46f0-ab1f-53aea551619c" />

# ATTACK GOT : overwrite GOT được khi NO RELRO hoặc PARTIAL RELRO 

- `<address> & 0xff` : Giữ lại 1 byte cuối

- `<address> >> 8` : Dịch phải 8 bit

- Overwrite GOT thằng sau format string

- `*` In ra số lượng padding < số lượng byte > đúng bằng con số mà nó đang trỏ tới, ví dụ `%*14$c%15$hn`

# Khai thác .Fini_array :

- Mảng .fini_array(con trỏ hàm) được gọi tự động và lấy giá trị bên trong mảng là .fini_array[0] để nhảy tới đó sau khi chương trình main kết thúc, để dọn dẹp tài nguyên

- Sẽ có 1 address đặc biệt trên stack( ở gần chỗ set biến môi trường, đặc điểm nhận dạng là địa chỉ giống libc nhưng ko phải libc, stack, binary mà sẽ nằm ở path file ld) sẽ trỏ tới địa chỉ exe_base (Như trong ảnh là dưới SHELL = bin/bash)

<img width="2125" height="1004" alt="image" src="https://github.com/user-attachments/assets/ec0c9eea-8067-40cf-9275-6b734fc20a9a" />

- `Info files` để tìm .fini_array :

 <img width="1138" height="93" alt="image" src="https://github.com/user-attachments/assets/9567428c-3cad-42ae-ad40-933bd557ff67" />

- Địa chỉ tại .fini_array = exe_base(Lấy ở chỗ address đặc biệt kia trỏ tới) + offset -> Overwrite địa chỉ mà address đặc biệt kia trỏ tới ko còn là exe_base mà thành địa chỉ khác + offset để điều hướng lấy get_shell

- mấy bài `bof` chủ yếu là cơ chế nhảy thẳng nên cần địa chỉ code, nhưng mấy bài `fmtstr` như attack GOT, .fini_array thì cần địa chỉ chứa code, đọc nó rồi mới nhảy tới code ( cần con trỏ )

- Cấu trúc `dictionary` của python :

  package = {

    key : value,

    key : value,

    key : value
  }

- Muốn truy cập value thì `package[key]`

- `sorted` : sắp xếp của python 

- các con trỏ stack có sẵn trên stack, ta có thể ghi thêm dữ liệu lên đó để có thể sử dụng và tạo thành một chuỗi format string `ghi đè` lên nhau và để chuẩn xác phải kết hợp `fullform` ghi trước rồi mới tới `shortform`

 <img width="1557" height="128" alt="image" src="https://github.com/user-attachments/assets/c7d9b116-e666-42af-b783-d66c2e2f93a9" />

# Interger Overflow :

### Số có dấu :
- bit 1 ở đầu được bật là số âm

- bit 0 ở đầu là số dương

- number < `0x7f` : là số dương

- number > `0x80` : là số âm

# SROP :

- syscall number rax = 0xf

- flat : nối nhiều payload lại và chuyển thành byte

- `frame = SigreturnFrame()` : Tạo struct với các thanh ghi dùng để setup

- `frame.<thanhghi> = ...` : setup thanh ghi

<img width="561" height="534" alt="image" src="https://github.com/user-attachments/assets/0cc95ebc-9c2b-4a1a-95bd-6e63329b5c82" />


# Out of bound :

- Nhập vào số âm nằm ngoài mảng có thể truy cập dữ liệu ở trước nó

# Seccomp : Giới hạn 1 số syscall

- Trong gdb nhập call (int)mprotect(exe_base, offset, 7) 7 là quyền rwx trước khi thực thi seccomp

- `seccomp-tools dump ./<file>` : Để kiểm tra seccomp

# Hook : 

- __free_hook, __malloc_hook

# Race condition :

- Xảy ra 2 tiến trình cùng truy cập vào 1 biến dùng chung, ví dụ hàm 1 gọi hàm chứa biến x và biến x vẫn đang trong tiến trình, thì có hàm 2 cũng gọi biến x --> getshell

# Khi gặp file stripped để map hàm từ ida vào gdb : 

- di setpath /mnt/d/đường/dẫn/tới/chall_mới

- decompiler-integration connect

# BASE64 ENCODE :

- biến 8bit thông thường thành 6bit rồi so nó bảng mã base64 để chuyển thành kí tự đó, nếu tổng số bit mà ko chia hết cho 6 thì bit dư tự biến thành dấu `=`

# Khi shellcode ko chạy được -> thêm ở đầu `sub rsp, 0x200` hoặc bao nhiêu đấy, % 16 == 0 càng tốt hoặc dùng trick lỏ `shellcode = asm(shellcraft.sh())`(dùng cho mấy shellcode đơn giản)

# Rand() : Có thể predict bằng cách : 

- `import ctypes, time`

- `libc = ctypes.CDLL("libc.so.6")`: Load file libc hệ thống để dùng hàm random của C (khác random của python)

- `seed = int(time.time())`: seed "hạt giống" như kiểu 1 thứ để `srand()` dựa vào để random mà `time` kia trả về số giây từ 1970 đến nay, thường dùng làm seed

- `libc.srand(seed)` : Khởi tạo seed

- `libc.rand()` : Khi đó giống với chương trình nếu cùng seed (time(nullptr))

# Bypass filter Command Injection : 

- `$(...)` : Lấy output của lệnh trong ngaoawjc làm input | ví dụ : `echo $(whoami) = echo ubuntu`

- `${IFS}` : Thay thế cho khoảng trắng mà ko bị tính là khoảng trắng, bị chắn | ví dụ : scanf gặp khoảng trắng là dừng nhưng muốn nó nhận tất thì ta có thể : `cat{IFS}flag`

# Leak libc từ Dockerfile :

- build & run docker -> `sudo docker ps -a`( Lấy id ) -> `sudo docker cp -L <id>:/usr/lib/x86_64-linux-gnu/libc.so.6 .`

# Gặp các bài chặn một số byte của shellcode : 

- Vào `python3` -> `context.arch = 'amd64'` -> `print(asm("code shellcode").hex())` để chuyển thành byte thô rồi so với các byte bị chặn -> biết lệnh nào bị chặn -> fix lệnh đó

# Bypass khi opcode 0x0f05 `syscall` bị chặn :

<img width="272" height="172" alt="image" src="https://github.com/user-attachments/assets/d9a83692-906c-4341-966a-a92b2b2d8548" />

<img width="528" height="99" alt="image" src="https://github.com/user-attachments/assets/5e075549-387d-41ac-85bc-431ee3fb399b" />

- Gán địa chỉ `sys_call + 1byte` là `0x04` vào rbx sau đó cộng 1 đơn vị thành `0x05` -> bypass thành công

- Bắt buộc phải có `[rip + label]` để có thể tìm thấy `label`

# Reverse shell :

<img width="1475" height="1218" alt="image" src="https://github.com/user-attachments/assets/e0fb4282-da69-474a-9567-df06bddd80b3" />

# environ : Thuộc libc trỏ tới stack

# HEAP :

- `heap chunks` : Các trunk được khởi tạo

- `heap chunk <address>` : thông tin của trunk đó

- `heap bin` : Xem các bin (là chỗ để lưu trữ các chunk sau khi bị `free`)
