Tutorial 01 - Bare Minimum
==========================

Okay, we're not going to do anything here, just test our toolchain. The resulting kernel8.img should
boot on Raspberry Pi, and stop the CPU cores in an infinite loop. You can check that by running

```sh
$ qemu-system-aarch64 -M raspi3b -kernel kernel8.img -d in_asm
        ... output removed for clarity, last line: ...
0x0000000000080004:  17ffffff      b #-0x4 (addr 0x80000)
```


**NOTE**: Phiên bản hướng dẫn này hiên tại đã cũ nên cần phải thay đổi một số công cụ biên dịch kiến trúc `ARM` khác và sửa lại Makefile.

Makefile sau khi sửa.
```makefile
CFLAGS = -Wall -O2 -ffreestanding -nostdinc -nostdlib -nostartfiles

all: clean kernel8.img

start.o: start.S
	aarch64-linux-gnu-gcc $(CFLAGS) -c start.S -o start.o

kernel8.img: start.o
	aarch64-linux-gnu-ld -nostdlib -nostartfiles start.o -T link.ld -o kernel8.elf
	aarch64-linux-gnu-objcopy -O binary kernel8.elf kernel8.img

clean:
	rm kernel8.elf *.o >/dev/null 2>/dev/null || true

run:
	qemu-system-aarch64 -M raspi3 -kernel kernel8.img -d in_asm
```
- Để chạy Makefile ta dùng câu lệnh (ở đây tôi dùng GCC)
```sh
make -f Makefile.gcc
```
- Sau khi biên dịch ta sẽ được 1 file `kernel8.img` trong thư mục làm việc hiện tại. Để có thể chạy file này ta dùng `QEMU`. 

HIện tại qemu chỉ hỗ trợ cho `rasp3` thôi nên câu lệnh chạy:
```sh
$ qemu-system-aarch64 -M raspi3 -kernel kernel8.img -d in_asm
```
Ta sẽ thấy những mã `ASM` sẽ hiện lên `Terminal`
```asm
----------------
IN: 
0x00000000:  580000c0  ldr      x0, #0x18
0x00000004:  aa1f03e1  mov      x1, xzr
0x00000008:  aa1f03e2  mov      x2, xzr
0x0000000c:  aa1f03e3  mov      x3, xzr
0x00000010:  58000084  ldr      x4, #0x20
0x00000014:  d61f0080  br       x4

----------------
IN: 
0x00000300:  d2801b05  movz     x5, #0xd8
0x00000304:  d53800a6  mrs      x6, mpidr_el1
0x00000308:  924004c6  and      x6, x6, #3
0x0000030c:  d503205f  wfe      
0x00000310:  f86678a4  ldr      x4, [x5, x6, lsl #3]
0x00000314:  b4ffffc4  cbz      x4, #0x30c

----------------
IN: 
0x00000300:  d2801b05  movz     x5, #0xd8
0x00000304:  d53800a6  mrs      x6, mpidr_el1
0x00000308:  924004c6  and      x6, x6, #3
0x0000030c:  d503205f  wfe      
0x00000310:  f86678a4  ldr      x4, [x5, x6, lsl #3]
0x00000314:  b4ffffc4  cbz      x4, #0x30c

----------------
IN: 
0x00000300:  d2801b05  movz     x5, #0xd8
0x00000304:  d53800a6  mrs      x6, mpidr_el1
0x00000308:  924004c6  and      x6, x6, #3
0x0000030c:  d503205f  wfe      
0x00000310:  f86678a4  ldr      x4, [x5, x6, lsl #3]
0x00000314:  b4ffffc4  cbz      x4, #0x30c

----------------
IN: 
0x00080000:  d503205f  wfe      
0x00080004:  17ffffff  b        #0x80000

----------------
IN: 
0x0000030c:  d503205f  wfe      
0x00000310:  f86678a4  ldr      x4, [x5, x6, lsl #3]
0x00000314:  b4ffffc4  cbz      x4, #0x30c

----------------
IN: 
0x0000030c:  d503205f  wfe      
0x00000310:  f86678a4  ldr      x4, [x5, x6, lsl #3]
0x00000314:  b4ffffc4  cbz      x4, #0x30c

----------------
IN: 
0x0000030c:  d503205f  wfe      
0x00000310:  f86678a4  ldr      x4, [x5, x6, lsl #3]
0x00000314:  b4ffffc4  cbz      x4, #0x30c

```

- Ta thấy ở đây có 3 dòng lệnh `ASM` lặp đi lặp lai: (trong file Start.S)
~~~asm
0x0000030c:  d503205f  wfe      
0x00000310:  f86678a4  ldr      x4, [x5, x6, lsl #3]
0x00000314:  b4ffffc4  cbz      x4, #0x30c
~~~




Start
-----

When the control is passed to kernel8.img, the environment is not ready for C. Therefore we must
implement a small preamble in Assembly. As this first tutorial is very simple, that's all we have, no C
for now.

Note that CPU has 4 cores. All of them are running the same infinite loop for now.

Makefile
--------

Our Makefile is very simple. We compile start.S, as this is our only source. Then in linker phase we
link it using the linker.ld script. Finally we convert the resulting elf executable into a raw image.

Linker script
-------------

Not surpisingly simple too. We just set the base address where our kernel8.img will be loaded, and we
put the only section we have there. Important note, for AArch64 the load address is **0x80000**, and
not **0x8000** as with AArch32.

