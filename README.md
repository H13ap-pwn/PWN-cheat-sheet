# GDB/GEF

`pattern create <số byte>` : Tạo byte

`pattern search <địa chỉ>` : Tìm offset 

`search-pattern <chuỗi>` : Tìm chuỗi( ví dụ chuỗi /bin/sh)

# PWNtool: < Viết script, payload >

* Để debug động `python3 ten.py` ( Chạy cái script ) vừa viết ở trên rồi copy pid <img width="856" height="71" alt="image" src="https://github.com/user-attachments/assets/64f28201-0e4d-414c-9fee-ae38990bec93" />
sau đó qua tab wsl khác mở gdb rồi attach pid <img width="1257" height="525" alt="image" src="https://github.com/user-attachments/assets/3a952b07-2189-47f3-a61a-3752ad8af106" />
xong di chuyển đến chỗ cần thao tác như read,... rồi quay lại tab wsl vừa viết script xong enter để bắt đầu payload rồi dữ liệu được gửi vào gdb

* Cách lấy hàm ko cần tìm địa chỉ, chỉ việc dùng:

`context.binary = exe = ELF("./<tên file>", checksec=False)`

`payload += p64(exe.sym.<tên hàm>)`

<img width="787" height="125" alt="image" src="https://github.com/user-attachments/assets/9f9ebac7-6ab8-4308-aa00-f6c1c4f57ec2" />

* Chuyển /bin/sh\0 thành số nguyên ko dấu rồi gửi vào thanh ghi nào đó

* Cách viết shellcode : <img width="683" height="403" alt="image" src="https://github.com/user-attachments/assets/3d760efd-6a65-48ca-aa8e-0ddb837e68e1" />


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

- `int(data, 16)` dùng cho hexstring

- `u64(data)` dùng cho raw byte

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

- one_gadget `/lib/x86_64-linux-gnu/libc.so.6` : Để tìm các gadget và constraints(điều kiện để nhảy đến gadget) có thể xài

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

- `f'...{i}...'` : f báo rằng chuỗi bên trong có thể nhét biến vào

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
