# Notes

1. [Intro](#intro)
2. [Processes](#processes)
3. [Threads](#threads)
4. [Synchronization](#synchronization)
5. [Memory Management](#memory-management)
6. [File System](#file-system)
7. [Sockets](#sockets)
8. [I/O Model](#io-model)
9. [Network (TCP/IP)](#network-tcpip)

# Intro

OS provide a VM abstraction to handle various hardware.<br/>
CPU operates in 2 different modes (user mode / kernel mode).<br/>
There is special 'sysmode' bit to switch from user mode to kernel mode.<br/>
Kernel mode provides an access to every resource and underlying hardware.<br/>
System calls provide an essential interface between a process and operating system.

Syscall:
```
user mode -> syscall -> kernel mode -> execute -> user mode
```

Hardware:
- I/O devices are managed by OS
- They are operate independently of main CPU
- Device uses interrupts to signal OS about event (IRQ)

Example of I/O:
1. Ethernet receives packet, writes packet into buffer
2. Network Interface Card (NIC) - signals an interrupt
3. CPU stops current instruction, saves current context
4. CPU jumps to handler address from kernel vector table
5. Ethernet device driver reads packet from buffer, processes it
6. Kernel restores saved context and reissues interrupted instruction

Interrupt:
- software - unix signals, exceptions generated by programs (user mode)
- hardware - i/o, system timer, etc.. (signal to operating system / kernel mode)

```
---------------------------
| start executing program |
---------------------------
             |
---------------------------
|    fetch instruction    | <------------------- Yes
---------------------------                     |
             |                                  |
---------------------------                     |
|    decode instruction   |                     |
---------------------------                     |
             |                                  |
---------------------------                     |              ---------------
|   execute instruction   |                     |              | end program |
---------------------------                     |              ---------------
             |                                  |                     ^
---------------------------                     |                     |
|       interrupt ?       |                     |                     |
---------------------------                     |                     |
             |                  ---------------------------------     |
             ------------No---> | are there more instructions ? | --- No
             |                  ---------------------------------
             | Yes                              ^
             |                                  |
---------------------------                     |
|  service the interrupt  |                     |
---------------------------                     |
             |                                  |
              \                                 |
               -----------Processor----------   |
               |   Halts thread execution   |   |
               |     Saves thread state     |   |
               | Executes interrupt handler |   |
               ------------------------------   |
              /                                 |
             |                                  |
---------------------------                     |
|     resume execution    | --------------------
---------------------------
```

# Processes

ADT: PCB (Process Control Block)

Process is the OS abstraction of execution.
```
Kernel -> Load app code -> Process -> Threads
```

Process creation:
- A process is created by another process
- In Unix processes are created using fork
- Process is represented with PCB Data Structure
- Every process has 1 parent, but can have many child

PCB Data Structure:
```
pid, state, hardware state (pc, sp, regs), memory management, etc..
```

Process execution:
- OS maintains a collection of queues for processes (ready, waiting, etc..)
- Each PCB is queued to the corresponding 'queue' according to the current state
- When a process is created the OS allocates a PCB for it and places to 'ready' queue

Process representation:
- Process is instance of executing program (address space and one or more threads)

```
 0xFFFFFFFFF       -----------------
      |            |     Stack     |
      |            ----------------- <- SP
      |                    |
      |                    |
      |                    |
-------------              |
Address Space              |
-------------      -----------------
      |            |      Heap     |
      |            -----------------
      |            |  Data Segment |
      |            -----------------
      |            |  Text Segment | <- PC
 0x000000000       -----------------
```

Thread:
- single unique execution context
- program counter (indicating the next instruction), registers, execution flags, stack

Address Space:
- text segment, data segment, heap (malloc), stack (local vars, recursion)
- programs execute in address space that is distinct from the memory space of physical machine

# Threads

ADT: TCB (Thread Control Block)

Thread is the smallest unit of processing.
```
Kernel -> Load app code -> Process -> Threads
```

Thread creation:
- Each process has at least one thread
- Thread is scheduled by operating system
- Thread is represented with TCB Data Structure

TCB Data Structure:
```
tid, state, hardware state (pc, sp, regs), ref to process
```

```
 0xFFFFFFFFF       -----------------
      |            |   Stack =T1   |
      |            -----------------
      |            |   Stack =T2   |
      |            -----------------
      |            |   Stack =T3   |
      |            -----------------
      |                    |
      |                    |
-------------              |
Address Space              |
-------------              |
      |                    |
      |            -----------------
      |            |      Heap     |
      |            -----------------
      |            |  Data Segment |
      |            -----------------
      |            |               | <- PC (T3)
      |            |  Text Segment | <- PC (T2)
      |            |               | <- PC (T1)
 0x000000000       -----------------
```

Each thread has its own stack & copy of cpu registers.<br/>
Shared memory between threads: heap, global vars, static objects.<br/>
Thread 'yield' operation gives up the CPU to another thread (context switch).

# Synchronization

Hardware: Test & Set, Compare & Swap<br/>
Sync primitives: Lock, Mutex, Monitor, Semaphore, Condition Variable<br/>
IPC: File, Pipe, Queue, Signal, Socket, Shared Memory, Memory-Mapped File (mmap)

```
| T1 > balance = get_balance(account)
| T1 > balance = balance - amount
| T2 > balance = get_balance(account)     \
| T2 > balance = balance - amount          - context switch
| T2 > save_balance(account, balance)     /
| T1 > print_balance(account, balance)
```

Race condition - operation is not executed in proper sequence.<br/>
When this happens we need some mechanism to be ensured that all ok.<br/>
Code that uses mutual exclusion to sync its execution called critical section.<br/>
Only 1 thread at time can execute in the critical section (all other threads will wait).

# Memory Management

ADT: Page Table

Processes use virtual addresses to read/write to memory location.<br/>
Processes view memory as one contiguous address space from 0 to N.<br/>
To map virtual addresses to physical addresses OS uses Page Table structure.

Pages:
- Page Table resides in physical memory (one per process)
- Pages are fixed-length contiguous block of virtual memory (4KB)
- Pages moved to disk when memory is full & loaded back when referenced again
- The address 0x10001000 maps to different physical address in different processes

```
                                          --------
                                        0 |      |
   ----------         -----------         --------
 0 | page A |       0 | frame 3 |       1 |      |
   ----------         -----------         --------
 1 | page B |       1 |         |       2 |      |
   ----------         -----------         --------
 2 | page C |       2 | frame 5 |       3 |  &A  | <- frame 3
   ----------         -----------         --------
 3 | page D |       3 |         |       4 |      |
   ----------         -----------         --------
 4 | page E |       4 |         |       5 |  &C  | <- frame 5
   ----------         -----------         --------
 5 | page F |       5 | frame 7 |       6 |      |
   ----------         -----------         --------
 virtual-memory       page-table!       7 |  &F  | <- frame 7
                                          --------
                                          physical
```

Segmentation:
- Technique that partitions memory into logically related units (code, data, etc..)

```
--------------------------------
| SegID      | Base   | Limit  |
--------------------------------
| 0 (code)   | 0x4000 | 0x0800 |
| 1 (data)   | 0x4800 | 0x1400 |
| 2 (shared) | 0xF000 | 0x1000 |
| 3 (stack)  | 0x0000 | 0x3000 |
--------------------------------

0x0000 ---------                0x0000 ---------- SegID=3
       |xxxxxxx| ----                  |xxxxxxxx|
       ---------    | SegID=0          |xxxxxxxx|
       |       |    |                  |xxxxxxxx|
       |       |    |                  ----------
0x4000 ---------    |                  |        |
       |xxxxxxx|--  ----------> 0x4000 ---------- SegID=0
       |xxxxxxx| |                     |xxxxxxxx|
       --------- | SegID=1             |xxxxxxxx|
       |       | -------------> 0x4800 ---------- SegID=1
       |       |                       |xxxxxxxx|
0x8000 ---------                       ----------
       |xxxxxxx|                       |        |
       ---------                       |        |
       |       |                       |        |
       |       |                       |        |
0xC000 ---------                       |        |
       |xxxxxxx|                       |        |
       |xxxxxxx|                       |        |
       |xxxxxxx|                       |        |
       ---------                       |        |
       |       |                0xF000 ---------- SegID=2
       |       |                       |xxxxxxxx|
       ---------                       ----------
        virtual                         physical
```

Swapping:
- What if not all processes fit in memory?
- In order to make room for next processes, some or all of them moved to disk

```
-------------             -------------
|           | swap out    |  ----     |
|           |-------------|->|P1|     |
|           |             |  ----     |
| UserSpace |             |           |
|           |     swap in |     ----  |
|           |<------------|-----|P2|  |
|           |             |     ----  |
-------------             -------------
 main memory              backing store
```

# File System

ADT: INode (Index Node)

File is a collection of blocks.

Permissions:
```
rw- rw- rw- = 110 110 110 (chmod 666)
rwx --- --- = 111 000 000 (chmod 700)
rwx r-x r-x = 111 101 101 (chmod 755)
rwx rwx rwx = 111 111 111 (chmod 777)

-rwxr-xr-x user:root group:root /bin/bash
```

File /bin/bash is owned by user 'root'.<br/>
User 'root' (rwx) can read/write/execute.<br/>
Members in group 'root' (r-x) can read/execute.<br/>
Everybody else (r-x) can read/execute this file in system.

Unix inodes:<br/>
Data structure that contains all information about a file.
```
$ ls -ia /var/ (inode 3633697)
3633697 .           2 ..     3633701 cache     3633723 log

$ ls -ia /var/log (inode 3633723)
3633723 .     3633697 ..     3634833 acpid     3633883 httpd
```

```
-------------------
| inode structure |
-------------------
| mode            | - permissions
| owner           | - owner/group
| size            | - size of file (bytes)
| timestamps      | - inode create/update time
-------------------
| direct blocks   | - blocks with file content (file <= 48KB)
-------------------
| indirect blocks | - pointer to the next disk block (file <= 4MB)
| double indirect | - pointer to the next disk block (file <= 4GB)
| triple indirect | - pointer to the next disk block (file <= 4TB)
-------------------
```

# Sockets

ADT: InPCB (Internet Protocol Control Block)

```
User -> Sockets -> OS -> TCP/UDP -> IP -> Ethernet
Socket = protocol + host + port (ex: 'tcp, 192.44.235.1, 80')
```

```
------------            ------------
|   Send   |            |   Recv   |
------------            ------------
     \                        /
------------            ------------
|  Socket  |            |  Socket  |
------------            ------------
     \                        /
------------            ------------
|   TCP    |            |    TCP   |
------------            ------------
     \ sk_buff        sk_buff /
------------            ------------
|    IP    |            |    IP    |
------------            ------------
     \                        /
------------            ------------
| Ethernet | <--------> | Ethernet |
------------            ------------
```

Descriptors:
- Everything in Unix is a file (file, pipe, network connection, etc..)
- When programs do any I/O they do it by reading/writing to file descriptor
- File descriptor is an abstract indicator (integer) associated with an open file
- File descriptor for network communication returns via socket() syscall (send/recv)

Connection process:
```
Socket syscall allocate new structure PCB
[addr, port, dest addr, dest port, listen?]
```

Syscall connect is used by client process
```
[192.168.0.1, 13001, 225.10.35.15, 8080, -] - socket (client1)
[192.168.0.2, 13002, 225.10.35.15, 8080, -] - socket (client2)
```

When server accept connection a new socket is allocated
```
[225.10.35.15, 8080, NULL, NULL, TRUE] - socket (server)
[225.10.35.15, 8080, 192.168.0.1, 13001, -] - socket (fork for client1)
[225.10.35.15, 8080, 192.168.0.2, 13002, -] - socket (fork for client2)
```
After client has connected, send/recv syscall can be used to exchange data

# I/O Model

Phases of any input operation:
- Waiting for data to be ready
- Copying data from kernel to process

There are 5 types of I/O models:
- Blocking I/O
- Non-Blocking I/O
- I/O Multiplexing
- Asynchronous I/O (AIO)
- Signal-Driven I/O (SIGIO)

I/O Multiplexing:
- Allows to monitor a lot of file descriptors for new I/O
- Who needs to watch a lot of file descriptors at a time? Servers!

```
----------
| Server | <-- accept connection
----------
    |
    |--------> calls `accept` system call
               new file descriptor represents connection

For example we need to handle 10K connections:
for x in open_connections -> if has_new_input(x) -> process_input(x)

This can waste a lot of CPU time. Instead we can ask kernel about updates.
```

File descriptors monitoring:
- poll (Unix): waits for one of a set of FD to become ready to perform I/O
- epoll (Linux): monitor multiple FD to see if I/O is possible on any of them
- select (Unix): monitor multiple FD waiting until FD become ready for I/O ops

poll/select:
- Fundamentally use the same code (poll ~= select)
- Give a list of file descriptors to get information about
- They tell you which ones have data available to read/write to

```
poll: monitor [1,3,7,8] file descriptors => it will monitor only 4 FD (1,3,7,8)
select: monitor 19 file descriptors with 3 bitsets => it will monitor 19 FD (0 to 19)
```

epoll (Linux 2.5+):
- Provides 3 syscalls: epoll_ctl, epoll_wait, epoll_create
- Give kernel a list of file descriptors to track and ask for updates
- Faster then poll/select, because kernel should not check all descriptors in a loop

```
epoll_create: tell the kernel for epolling (returns id)
epoll_ctl: tell the kernel file descriptors you are interested in updates about
epoll_wait: waits for i/o events, blocking the calling thread if no events available
```

Ways to get notifications:
- edge-triggered: get notifications every time a file descriptor becomes readable
- level-triggered: get a list of every file descriptor you're interested in that is readable

# Network (TCP/IP)

Layers:
```
---------------------------------------------
| L1 |               Ethernet               |
| L2 |            IP, ARP, ICMP             |
| L3 |               TCP, UDP               |
| L4 | DHCP, DNS, FTP, HTTP, SMTP, SSH, SSL |
---------------------------------------------
```

CIDR note:
```
10.0.0.0/8  => Range of IPs: 10.*.*.* (2 ** 32-8)
10.9.0.0/16 => Range of IPs: 10.9.*.* (2 ** 32-16)
10.9.8.0/24 => Range of IPs: 10.9.8.* (2 ** 32-24)

10.0.0.0/29 => 29 bits under mask, 3 bits available
10.0.0.0/29 => [11111111.11111111.11111111.11111000]
```

Connection:
```
SYN         C --> S     seq=x
SYN+ACK     C <-- S     seq=y, ack=x+1
ACK         C --> S     seq=x+1, ack=y+1     CONN_OPEN
ACK         C --> S     seq=x+1, ack=y+1     SEND_DATA
...
FIN+ACK     C --> S     seq=x, ack=y
ACK         C <-- S     seq=y, ack=x+1       CONN_CLOSE
FIN+ACK     C <-- S     seq=y, ack=x+1
ACK         C --> S     seq=x+1, ack=y+1     CONN_CLOSE
```

Data Transmission:
```
user ------------------------------
 [App]                     |D|     - write(fd, buf, len)
kernel ----------------------------
 [File]                            - validate file descriptor
 [Socket]                  |D|     - append buf to socket buffer
 [TCP]                 |TCP|D|     - create tcp segment, compute checksum
 [IP]               |IP|TCP|D|     - ip header, perform routing, checksum
 [Ethernet]     |ETH|IP|TCP|D|     - add ethernet header to packet, perform arp
 [NC Driver]                       - tell network interface card to send a packet
device ----------------------------
 [NIC]      |IFG|ETH|IP|TCP|D|CRC| - fetch & send packet, interrupt when send is done
```

Ethernet Frame Format:
```
------------------------------------------------------------
| Ethernet Header | IP Header | TCP Header | Payload | CRC |
------------------------------------------------------------
|                 |           |            |         |     |
|                 |           |            <--------->     |
|                 |           |                      |     |
|                 |           <---------------------->     |
|                 |                                  |     |
|                 <---------------------------------->     |
|                                                          |
<---------------------------------------------------------->
```

Ethernet Header:
```
-----------------------------------------------------------------
|          Source MAC           |        Destination MAC        |
-----------------------------------------------------------------
|          Ether Type           |     Data     |    Checksum    |
-----------------------------------------------------------------
```

IP Header:
```
-----------------------------------------------------------------
| Version | IHL | Type of Service |        Total Length         |
-----------------------------------------------------------------
|         Identification          | Flags |   Fragment Offset   |
-----------------------------------------------------------------
| Time to Live  |    Protocol     |       Header Checksum       |
-----------------------------------------------------------------
|                        Source Address                         |
-----------------------------------------------------------------
|                      Destination Address                      |
-----------------------------------------------------------------
|                    Options                    |    Padding    |
-----------------------------------------------------------------
```

TCP Header:
```
-----------------------------------------------------------------
|          Source Port          |       Destination Port        |
-----------------------------------------------------------------
|                        Sequence Number                        |
-----------------------------------------------------------------
|                     Acknowledgment Number                     |
-----------------------------------------------------------------
|  Data  |          |U|A|P|R|S|F|                               |
| Offset | Reserved |R|C|S|S|Y|I|            Window             |
|        |          |G|K|H|T|N|N|                               |
-----------------------------------------------------------------
|           Checksum            |        Urgent Pointer         |
-----------------------------------------------------------------
|                    Options                    |    Padding    |
-----------------------------------------------------------------
|                             data                              |
-----------------------------------------------------------------
```

UDP Header:
```
-----------------------------------------------------------------
|          Source Port          |       Destination Port        |
-----------------------------------------------------------------
|            Length             |           Checksum            |
-----------------------------------------------------------------
|                             data                              |
-----------------------------------------------------------------
```

DNS Lookup (host -> ip):
```
-------------------------------
| Root DNS Server             |-2------------+
| -> TLD Server IP for .com   |              |
-------------------------------              1
                                             |
-.com--------------------------       ----------------  <- example.com  --------
| Top-Level Domain Server     |-4---3-| DNS Resolver |------------------| User |
| -> Authoritative Server IP  |       ----------------  -> 142.25.0.20  --------
-------------------------------              |
                                             5
-example.com-------------------              |
| Authoritative DNS Server    |-6------------+
| -> Here you go 142.25.0.20  |
-------------------------------
```

ARP Broadcast (ip -> mac):
```
            -----------------------------N0--
            | IP: 10.10.10.10               |
            | MAC: 00:00:00:aa:aa:aa        |
            ---------------------------------
                            |
            ---------------------------------
            | ARP request                   |
        ----| Source MAC: 00:00:00:aa:aa:aa |----
        |   | Target MAC: ff:ff:ff:ff:ff:ff |   |
        |   ---------------------------------   |
        |        Who has IP 10.10.10.50?        |
        |                                       |
----------------------N1--     ----------------------N2--
| IP: 10.10.10.20        |     | IP: 10.10.10.50        |
| MAC: 00:00:00:bb:bb:bb |     | MAC: 00:00:00:cc:cc:cc |
--------------------------     --------------------------
                                                |
            ---------------------------------   |
            | ARP response                  |<---
            | Source MAC: 00:00:00:cc:cc:cc |
            | Target MAC: 00:00:00:aa:aa:aa |
            ---------------------------------
           10.10.10.50 is at 00:00:00:cc:cc:cc
```

Browser Request:
1. Parse URL (protocol: 'http', resource: '/')
2. Convert non-ascii characters in hostname to ascii
3. Determine http or https via HSTS (HTTP Strict Transport Security)
4. DNS lookup
   - check cache (domain=ip)
   - check domain in /hosts file
   - request to DNS server (returns Hostname+IP)
5. ARP broadcast
   - check cache (ip=mac)
   - find mac in subnet by given ip (returns target IP+MAC)
6. Create socket via syscall
   - get ip and port (from url)
   - build packet: TCP + IP + MAC
7. Packet is ready to be transmitted over network (Wi-Fi, Ethernet, etc..)
