---
title: "vmware의 metalkit 프로그램 디버깅"
date: 2021-03-12 19:55:00 +0900
categories: vmware SVGA blog
---

Debugging Metalkit Apps
-----------------------
이 글은 리눅스의 gdb로 Metalkit을 동적으로 분석할 수 있는 방법을 설명하며, 본 게시글은 vmware에서 제공하는 [vmware-svga](https://github.com/prepare/vmware-svga/blob/master/doc/debugging.txt)의 내용을 나름대로 해석한 글이다.

디버깅하고자 하는 vm 이미지의 설정파일(\*.vmx)를 에디터로 오픈해서 아래의 내용을 한다.

    debugStub.listen.guest32 = "TRUE"
    debugStub.hideBreakpoints = "TRUE"
    monitor.debugOnStartGuest32 = "TRUE"

이 후, VM 이미지를 실행하면 bios 화면이 뜨기전에 hang이 걸려있게 된다. 
이번 단계에서 할 것이 gdb를 이용해서 attach를 하는것인데, 
Metalkit에 심겨져있는 \.elf 파일을 가지고 있어야지 디버깅을 할 수 있다.

    micah@micah-64:~/metalkit/examples/apm-test$ gdb -q apm-test.elf
    No symbol table is loaded.  Use the "file" command.
    Using host libthread_db library "/lib/libthread_db.so.1".
    (gdb) set arch i386
    The target architecture is assumed to be i386
    (gdb) target remote localhost:8832
    Remote debugging using localhost:8832
    [New Thread 1]
    0x000ffff0 in ?? ()
    (gdb) break main
    Breakpoint 1 at 0x100ba2: file main.c, line 11.
    (gdb) cont
    Continuing.
    
Now you should see the VM boot. If you need to stop earlier, to debug
the bootloader, you can set a breakpoint at *0x7c00 instead. Usually
starting at main() is quite sufficient. As soon as Metalkit loads, you
should hit this breakpoint. From here on, all the normal gdb debug-fu
should work.

    Breakpoint 1, main () at main.c:11
    11      {
    (gdb) list
    6       #include "keyboard.h"
    7       #include "apm.h"
    8
    9       int
    10      main(void)
    11      {
    12         ConsoleVGA_Init();
    13         Intr_Init();
    14         Intr_SetFaultHandlers(Console_UnhandledFault);
    15         Keyboard_Init();
    (gdb) next
    main () at main.c:12
    12         ConsoleVGA_Init();
    (gdb)
    13         Intr_Init();
    (gdb)
    14         Intr_SetFaultHandlers(Console_UnhandledFault);
    (gdb)
    15         Keyboard_Init();
    (gdb) p gConsole
    $1 = {beginPanic = 0x1005ef <ConsoleVGABeginPanic>,
          clear = 0x1004d2 <ConsoleVGAClear>,
          moveTo = 0x1004c1 <ConsoleVGAMoveTo>,
          writeChar = 0x100511 <ConsoleVGAWriteChar>,
          flush = 0x100482 <ConsoleVGAMoveHardwareCursor>}
    (gdb)

---
