# GhostVEH 👻

Registers Vectored Exception Handlers by directly manipulating ntdll's internal `LdrpVectorHandlerList` structure instead of calling `RtlAddVectoredExceptionHandler`. 

**Tested on:** Windows 11 25H2 (Build 26200.7623)
- ToDo: Add BoyerMoore search / AOB - use zydis/capstone to disassm at runtime instead of hardcoding.
## What Is This?

Instead of calling `RtlAddVectoredExceptionHandler` like a normal person, we locate the internal handler list in ntdll and insert our handler directly.

```
Normal:  Your Code → RtlAddVectoredExceptionHandler → RtlpAddVectoredHandler → List
This:    Your Code → Manual List Insert → List
```

## Execution Flow

### Step 1: Locate RtlpAddVectoredHandler

```asm
RtlAddVectoredExceptionHandler:
    xor r8d, r8d
    jmp RtlpAddVectoredHandler
```

Pattern match the `E9` (JMP) instruction and calculate target from relative offset.

↓

### Step 2: Find LdrpVectorHandlerList

Scan `RtlpAddVectoredHandler` for this pattern:

```asm
lea rsi, ds:0[rbp*2]
mov [rbx+20h], rax
add rsi, rbp
lea rdi, LdrpVectorHandlerList      ; 48 8D 3D XX XX XX XX
lea rdi, [rdi+8]                    ; 48 8D 7F 08
```

The `lea rdi, [rdi+8]` confirms we found the right address (adding 8 skips the SRWLOCK to get to the ExceptionList).

↓

### Step 3: Find LdrProtectMrdata

Scan for `CALL` instructions in `RtlpAddVectoredHandler`:

```asm
xor ecx, ecx
lea rdi, [rdi+rsi*8]
call LdrProtectMrdata               ; E8 XX XX XX XX
```

Validate the target function starts with:

```asm
mov [rsp+arg_0], rbx               ; 48 89 5C 24 08
mov [rsp+arg_8], rsi               ; 48 89 74 24 10
push rdi                           ; 57
sub rsp, 20h                       ; 48 83 EC 20
```

↓

### Step 4: Allocate Handler Entry

From the disassembly, we allocate 0x28 bytes:

```asm
call LdrControlFlowGuardEnforced
xor edx, edx
lea r8d, [rdx+28h]                 ; Size = 0x28 (40 bytes)
test eax, eax
jz loc_18004F619
mov rcx, cs:LdrpMrdataHeap
```

Then allocate refcount (8 bytes):

```asm
and dword ptr [rax+18h], 0
xor edx, edx
mov rcx, gs:60h
lea r8d, [rdx+8]                   ; Size = 8 bytes
mov rcx, [rcx+30h]
call RtlAllocateHeap
mov [rbx+10h], rax                 ; Store at offset +0x10
```

↓

### Step 5: Initialize Handler Entry

Set refcount to 1 and encode the handler pointer:

```asm
mov rcx, rsi
mov qword ptr [rax], 1             ; *RefCount = 1
call RtlEncodePointer
lea rsi, ds:0[rbp*2]
mov [rbx+20h], rax                 ; Store encoded handler at +0x20
add rsi, rbp
```

The `lea rsi, ds:0[rbp*2]` and `add rsi, rbp` calculates the index (0 for VEH, 1 for VCH):
- If `rbp = 0` (VEH): `rsi = 0*2 + 0 = 0`
- If `rbp = 1` (VCH): `rsi = 1*2 + 1 = 3`

↓

### Step 6: Unlock .mrdata Protection

```asm
xor ecx, ecx                       ; ecx = 0 (unlock)
call LdrProtectMrdata
```

Inside `LdrProtectMrdata`, the unlock path:

```asm
mov ebx, cs:LdrpMrdataUnprotected
test edi, edi                      ; Check if Protect == 0
jz loc_18004F29D

loc_18004F29D:
test ebx, ebx                      ; If counter == 0
jz loc_18004F2BF

loc_18004F2BF:
mov ecx, 4                         ; PAGE_READWRITE
call LdrpChangeMrdataProtection

loc_18004F2C9:
inc ebx                            ; counter++
mov cs:LdrpMrdataUnprotected, ebx
```

↓

### Step 7: Acquire List Lock

```asm
lea rcx, LdrpVectorHandlerList
mov rcx, [rcx+rsi*8]               ; Get lock for VEH (index 0) or VCH
call RtlAcquireSRWLockExclusive
```

↓

### Step 8: Insert Into Linked List

Check if this is the first handler:

```asm
cmp [rdi], rdi                     ; Is list empty?
jnz short loc_18004F5FC
mov rax, gs:60h
lea ecx, [rbp+2]
lock bts [rax+50h], ecx            ; Set flag in TEB
```

Then insert based on `r14d` (FirstHandler parameter):

```asm
test r14d, r14d                    ; FirstHandler?
jz loc_18004F6EF                   ; Jump if 0 (insert at tail)

; Insert at head (FirstHandler = 1):
mov rax, [rdi]                     ; rax = list_head->Flink
cmp [rax+8], rdi                   ; Validate list integrity
jz loc_18004F6C2

loc_18004F6C2:
mov [rbx], rax                     ; entry->List.Flink = list_head->Flink
mov [rbx+8], rdi                   ; entry->List.Blink = list_head
mov [rax+8], rbx                   ; list_head->Flink->Blink = entry
mov [rdi], rbx                     ; list_head->Flink = entry

; Insert at tail (FirstHandler = 0):
loc_18004F6EF:
mov rax, [rdi+8]                   ; rax = list_head->Blink
cmp [rax], rdi                     ; Validate list integrity
jnz loc_18004F612

mov [rbx], rdi                     ; entry->List.Flink = list_head
mov [rbx+8], rax                   ; entry->List.Blink = list_head->Blink
mov [rax], rbx                     ; list_head->Blink->Flink = entry
mov [rdi+8], rbx                   ; list_head->Blink = entry
```

↓

### Step 9: Release Lock and Restore Protection

```asm
lea rax, LdrpVectorHandlerList
mov rcx, [rax+rsi*8]
call RtlReleaseSRWLockExclusive

mov ecx, 1                         ; ecx = 1 (lock)
call LdrProtectMrdata
```

Inside `LdrProtectMrdata`, the lock path:

```asm
mov ebx, cs:LdrpMrdataUnprotected
add ebx, 0FFFFFFFFh                ; ebx-- (decrement)
mov cs:LdrpMrdataUnprotected, ebx
jnz short loc_18004F282            ; If not zero, skip protection change

test ebx, ebx
jz short loc_18004F2A6
mov ecx, 2                         ; PAGE_READONLY
call LdrpChangeMrdataProtection

loc_18004F282:
mov rcx, rsi
call RtlReleaseSRWLockExclusive
```

↓

### Step 10: Handler is Now Active

When an exception occurs, Windows walks the list and calls your handler normally.

## The Handler Structure

Based on the disassembly:

```cpp
typedef struct _VECTORED_HANDLER_ENTRY {
    LIST_ENTRY List;           // +0x00: Doubly-linked list
    PULONG_PTR RefCount;       // +0x10: Pointer to refcount
    DWORD Unknown;             // +0x18: Zeroed out
    DWORD Padding;             // +0x1C
    PVOID EncodedHandler;      // +0x20: RtlEncodePointer result
} VECTORED_HANDLER_ENTRY;      // Total size: 0x28
```

`LdrpVectorHandlerList` structure:

```cpp
struct {
    SRWLOCK Lock;              // +0x00: At index [0]
    LIST_ENTRY ExceptionList;  // +0x08: At index [1] (VEH)
    LIST_ENTRY ContinueList;   // +0x18: At index [3] (VCH)
};
```

The `lea rdi, [rdi+8]` in the disassembly confirms the +8 offset to skip the SRWLOCK.

## Example Output

```
=== GhostVEH POC ===

RtlAddVectoredExceptionHandler = 00007FF8A1B2C3D0
RtlpAddVectoredHandler = 00007FF8A1B2E450
[+] Found 'lea rdi, [rip+0x12ABC]' at offset +0x8E -> 00007FF8A1C4F120
[+] CONFIRMED: This is LdrpVectorHandlerList!

[*] Searching for LdrProtectMrdata...
    [+] Found LdrProtectMrdata at offset +0x1A2 -> 00007FF8A1B2C5F0

Adding normal VEH...
=== VEH Handler Chain ===
[0] Entry: 000001D4B2E10A80
    Decoded: 00007FF6F2A11234
=== End (1 handlers) ===

Adding GhostVEH VEH...
[DEBUG] Calling LdrProtectMrdata(0)...
[DEBUG] Acquiring lock...
[DEBUG] Lock released!
[DEBUG] Calling LdrProtectMrdata(1)...
[+] GhostVEH VEH ADDED!

=== VEH Handler Chain ===
[0] Entry: 000001D4B2E11C40
    Decoded: 00007FF6F2A15678  ← Stealth handler
[1] Entry: 000001D4B2E10A80
    Decoded: 00007FF6F2A11234  ← Normal handler
=== End (2 handlers) ===

Triggering exception...
[STEALTH VEH] Exception: 0xC0000094 at RIP: 00007FF6F2A1ABCD
[NORMAL VEH] Exception: 0xC0000094
Caught!
```
## ⚠️ Legal Notice

**By using this code, you accept full responsibility for your actions. The author assumes zero liability for misuse.**
---
