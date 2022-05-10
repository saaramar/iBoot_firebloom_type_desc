# Firebloom (iBoot) - the type descriptor

## Intro

Welcome to part two of the Firebloom iBoot blogposts series. In my previous [blogpost](https://saaramar.github.io/iBoot_firebloom/) I've covered how Firebloom is implemented in iBoot, what the representations of allocations look like and how the compiler uses them. As a reminder, the structure represents an allocation looks as follows:

```
00000000 safe_allocation struc ; (sizeof=0x20, mappedto_1)
00000000 raw_ptr         DCQ ?                   ;
00000008 lower_bound_ptr DCQ ?                   ;
00000010 upper_bound_ptr DCQ ?                   ;
00000018 type            DCQ ?                   ;
00000020 safe_allocation ends
```

The last blogpost focused on how Firebloom uses `lower_bound_ptr` and `upper_bound_ptr`, which are used to mitigate spatial safety - i.e., any form of backward/forward out-of-bounds. In this blogpost, I would like to focus on the `type` pointer.

Readers are highly encouraged to read the first blogpost before reading this, for context. As in the previous blogpost, this research was done on `iBoot.d53g.RELEASE.im4p`, iPhone 12, ios 14.4 (18D52). 

## Keep reversing

If you recall, in the previous blogpost I showed the `do_safe_allocation` function, which wraps the allocation API and initializes the `safe_allocation` structure, where the pointer at offset +0x18 points to some structure that describes a type. We also saw an example that uses this pointer to verify some validations on the type before using the allocation. Now, it's time to see how such functionalities work and what this `type` pointer gives us.

I found a lot of interesting logic around the `type` pointer (casting, copying between safe allocations, etc.), which was really fun to reverse. I think it would be best to start from the building blocks, and go up :)

### Copying pointers and memory

A good place to start would be some example. Let's reverse up from `panic_memcpy_bad_type` - it's one of the 11 xrefs of `do_firebloom_panic`:

```assembly
iBoot:00000001FC1AA818 panic_memcpy_bad_type                   ; CODE XREF: call_panic_memcpy_bad_type+3C↑p
iBoot:00000001FC1AA818
iBoot:00000001FC1AA818 var_20          = -0x20
iBoot:00000001FC1AA818 var_10          = -0x10
iBoot:00000001FC1AA818 var_s0          =  0
iBoot:00000001FC1AA818
iBoot:00000001FC1AA818                 PACIBSP
iBoot:00000001FC1AA81C                 STP             X22, X21, [SP,#-0x10+var_20]!
iBoot:00000001FC1AA820                 STP             X20, X19, [SP,#0x20+var_10]
iBoot:00000001FC1AA824                 STP             X29, X30, [SP,#0x20+var_s0]
iBoot:00000001FC1AA828                 ADD             X29, SP, #0x20
iBoot:00000001FC1AA82C                 MOV             X19, X2
iBoot:00000001FC1AA830                 MOV             X20, X1
iBoot:00000001FC1AA834                 MOV             X21, X0
iBoot:00000001FC1AA838                 ADRP            X8, #0x1FC2F2248@PAGE
iBoot:00000001FC1AA83C                 LDR             X8, [X8,#0x1FC2F2248@PAGEOFF]
iBoot:00000001FC1AA840                 CBZ             X8, loc_1FC1AA848
iBoot:00000001FC1AA844                 BLRAAZ          X8
iBoot:00000001FC1AA848
iBoot:00000001FC1AA848 loc_1FC1AA848                           ; CODE XREF: panic_memcpy_bad_type+28↑j
iBoot:00000001FC1AA848                 ADR             X0, aMemcpyBadType ; "memcpy_bad_type"
iBoot:00000001FC1AA84C                 NOP
iBoot:00000001FC1AA850                 MOV             X1, X21
iBoot:00000001FC1AA854                 MOV             X2, X20
iBoot:00000001FC1AA858                 MOV             X3, X19
iBoot:00000001FC1AA85C                 BL              do_firebloom_panic
iBoot:00000001FC1AA85C ; End of function panic_memcpy_bad_type
```

We'll start with something simple - the operation of copying pointers.

If you recall, in my previous blogpost, I specifically mentioned that copying a pointer (i.e. a simple pointer assignment) now requires moving a tuple of 4 64-bits values. Usually, we see it as 2 `LDP`s and 2 `STP`s. A great example is the following function:

```assembly
iBoot:00000001FC15AD74 move_safe_allocation_x20_to_x19         ; CODE XREF: sub_1FC15A7E0+78↑p
iBoot:00000001FC15AD74                                         ; wrap_memset_type_safe+68↑p ...
iBoot:00000001FC15AD74                 LDP             X8, X9, [X20]
iBoot:00000001FC15AD78                 LDP             X10, X11, [X20,#0x10]
iBoot:00000001FC15AD7C                 STP             X10, X11, [X19,#0x10]
iBoot:00000001FC15AD80                 STP             X8, X9, [X19]
iBoot:00000001FC15AD84                 RET
iBoot:00000001FC15AD84 ; End of function move_safe_allocation_x20_to_x19
```

This pattern of 2 `LDP`s and 2 `STP`s is very common in iBoot these days (makes sense, pointer assignments happen a lot), and you'll see it inlined in many places. While this is great for pointer assignments, in many cases we would like to actually copy the content - for instance, to call `memcpy`. And this is where it gets interesting. Let's ask ourselves - should we be allow to just call `memcpy` between two "safe_allocations"?

In theory, one could do the following:

```c++
memcpy(dst->raw_ptr, src->raw_ptr, length);
```

However, keep in mind that each `safe_allocation` also has a type. This `type` pointer points to some structure, which might give us more information about the type we interact with. This information could be used in more checks and validations. For instance, we would expect to see some logic that checks if the type of `dst` and `src` are *primitive types* (for instance, types that don't contain references to further structures, nested structures, etc., such as short/int/float/double/etc.).

This is important, because if `src` or `dst` are non-primitive types - we might want to make sure we only copy `src` to `dst` if their types are equal in some way. Or, maybe `type` actually holds more metadata about the structure we could use to enforce more security properties.

Therefore, I would like to find out how Firebloom describes primitive types. I got that after I reversed the casting functionality, among with some other things. The fun part is that it couldn't be easier - we have a lot of useful strings used in `cast_impl`. For example:

```
aCannotCastPrim DCB "Cannot cast primitive type to non-primitive type",0
```

Let's xref that, and see what's going on there. In the code below, `X21` is the `type` pointer from the `safe_allocation`:

```assembly
iBoot:00000001FC1A0CF8 ; X21 is the type pointer
iBoot:00000001FC1A0CF8                 LDR             X11, [X21]
iBoot:00000001FC1A0CFC                 AND             X11, X11, #0xFFFFFFFFFFFFFFF8
iBoot:00000001FC1A0D00                 LDRB            W12, [X11]
iBoot:00000001FC1A0D04                 TST             W12, #7
iBoot:00000001FC1A0D08 ; one of the 3 LSB bits is not 0, non-primitive type
iBoot:00000001FC1A0D08                 B.NE            cannot_cast_primitive_to_non_primitive_type
iBoot:00000001FC1A0D0C                 LDR             X11, [X11,#0x20]
iBoot:00000001FC1A0D10                 LSR             X11, X11, #0x23 ; '#'
iBoot:00000001FC1A0D14                 CBNZ            X11, cannot_cast_primitive_to_non_primitive_type
...
iBoot:00000001FC1A0E70 cannot_cast_primitive_to_non_primitive_type
iBoot:00000001FC1A0E70                                         ; CODE XREF: cast_impl+478↑j
iBoot:00000001FC1A0E70                                         ; cast_impl+484↑j
iBoot:00000001FC1A0E70                 ADR             X11, aCannotCastPrim ; "Cannot cast primitive type to non-primi"...
```

Ok, now we know how Firebloom marks and tests primitive types. This code is part of an extensive functionality that casts one type to another, and specifically, `X21` here is the type pointer of the `safe_allocation` structure we are casting to. We are about to do the cast, and in this flow, we know that the type we are casting from is primitive. So, the code needs to verify the type we are casting to is also primitive (otherwise, panic).

To check that, the code dereferences the `type` pointer, which gives us a pointer (let's call it type_descriptor). We mask out the 3 LSB bits (probably there is an encoding there, that's why *every* place that uses this pointer masks it out like that before dereferencing it) and dereference the masked pointer.

Now, the type is a considered "primitive" if both of the following properties are true:

1. all the 3 LSB bits in the first qword are 0.
2. the high 29 bits of the value stored at offset 0x20 is 0.

Great, we just learned how primitive types are represented. During this blogpost, we will understand exactly what this value represents - stay tuned.

Armed with this knowledge, I believe we are ready to see how Firebloom wraps `memset` and `memcpy` in iBoot. Let's begin with `memset`:

```assembly
iBoot:00000001FC15A99C wrap_memset_safe_allocation             ; CODE XREF: sub_1FC04E5D0+124↑p
iBoot:00000001FC15A99C                                         ; sub_1FC04ED68+8↑j ...
iBoot:00000001FC15A99C
iBoot:00000001FC15A99C var_30          = -0x30
iBoot:00000001FC15A99C var_20          = -0x20
iBoot:00000001FC15A99C var_10          = -0x10
iBoot:00000001FC15A99C var_s0          =  0
iBoot:00000001FC15A99C
iBoot:00000001FC15A99C                 PACIBSP
iBoot:00000001FC15A9A0                 SUB             SP, SP, #0x60
iBoot:00000001FC15A9A4                 STP             X24, X23, [SP,#0x50+var_30]
iBoot:00000001FC15A9A8                 STP             X22, X21, [SP,#0x50+var_20]
iBoot:00000001FC15A9AC                 STP             X20, X19, [SP,#0x50+var_10]
iBoot:00000001FC15A9B0                 STP             X29, X30, [SP,#0x50+var_s0]
iBoot:00000001FC15A9B4                 ADD             X29, SP, #0x50
iBoot:00000001FC15A9B8 ; void *memset(void *s, int c, size_t n);
iBoot:00000001FC15A9B8 ; X0 - dst    (s)
iBoot:00000001FC15A9B8 ; X1 - char   (c)
iBoot:00000001FC15A9B8 ; X2 - length (n)
iBoot:00000001FC15A9B8                 MOV             X21, X2
iBoot:00000001FC15A9BC                 MOV             X22, X1
iBoot:00000001FC15A9C0                 MOV             X20, X0
iBoot:00000001FC15A9C4                 MOV             X19, X8
iBoot:00000001FC15A9C8 ; verify upper_bound - raw_ptr >= x2 (length)
iBoot:00000001FC15A9C8                 BL              check_ptr_bounds
iBoot:00000001FC15A9CC                 LDR             X23, [X20,#safe_allocation.type]
iBoot:00000001FC15A9D0                 MOV             X0, X23
iBoot:00000001FC15A9D4 ; check if dst is a primitive type
iBoot:00000001FC15A9D4                 BL              is_primitive_type
iBoot:00000001FC15A9D8                 TBNZ            W0, #0, call_memset
iBoot:00000001FC15A9DC                 CBNZ            W22, detected_memset_bad_type
iBoot:00000001FC15A9E0                 MOV             X0, X23
iBoot:00000001FC15A9E4                 BL              get_type_length
iBoot:00000001FC15A9E8 ; divide and multiply the length argument
iBoot:00000001FC15A9E8 ; by the type's size, to detect
iBoot:00000001FC15A9E8 ; partial/unalignment writes
iBoot:00000001FC15A9E8                 UDIV            X8, X21, X0
iBoot:00000001FC15A9EC                 MSUB            X8, X8, X0, X21
iBoot:00000001FC15A9F0                 CBNZ            X8, detected_memset_bad_n
iBoot:00000001FC15A9F4
iBoot:00000001FC15A9F4 call_memset                             ; CODE XREF: wrap_memset_safe_allocation+3C↑j
iBoot:00000001FC15A9F4                 LDR             X0, [X20,#safe_allocation]
iBoot:00000001FC15A9F8                 MOV             X1, X22
iBoot:00000001FC15A9FC                 MOV             X2, X21
iBoot:00000001FC15AA00                 BL              _memset
iBoot:00000001FC15AA04                 BL              move_safe_allocation_x20_to_x19
iBoot:00000001FC15AA08                 LDP             X29, X30, [SP,#0x50+var_s0]
iBoot:00000001FC15AA0C                 LDP             X20, X19, [SP,#0x50+var_10]
iBoot:00000001FC15AA10                 LDP             X22, X21, [SP,#0x50+var_20]
iBoot:00000001FC15AA14                 LDP             X24, X23, [SP,#0x50+var_30]
iBoot:00000001FC15AA18                 ADD             SP, SP, #0x60 ; '`'
iBoot:00000001FC15AA1C                 RETAB
iBoot:00000001FC15AA20 ; ---------------------------------------------------------------------------
iBoot:00000001FC15AA20
iBoot:00000001FC15AA20 detected_memset_bad_type                ; CODE XREF: wrap_memset_safe_allocation+40↑j
iBoot:00000001FC15AA20                 BL              call_panic_memset_bad_type
iBoot:00000001FC15AA24 ; ---------------------------------------------------------------------------
iBoot:00000001FC15AA24
iBoot:00000001FC15AA24 detected_memset_bad_n                   ; CODE XREF: wrap_memset_safe_allocation+54↑j
iBoot:00000001FC15AA24                 BL              call_panic_memset_bad_n
iBoot:00000001FC15AA24 ; End of function wrap_memset_safe_allocation
```

Ok, cool - so the function `wrap_memset_safe_allocation` checks if `dst`'s type is a primitive type. If so - it simply calls `memset` directly.

However, if the type is not a primitive type, we have more information to take advantage of! As it turns out, Apple encodes more information in the `type` structure (they have a pointer that points to a struct, there are a lot of things they can do). For instance, non-primitive types have variable lengths, and as it turns out, Apple encodes this information in the memory pointed by the first pointer in the type structure. If the `n` argument to `memset` is **not aligned with respect to the type's length**, iBoot calls `panic_memset_bad_n`. 

Note that at the beginning of this function, there is the usual bound checks (using the bounds pointers in the `safe_allocation`), to detect and panic upon OOBs. The `panic_memset_bad_n`  is a further hardening to detect and panic upon **partial initialization/copy** scenarios. Pretty cool!

We can expect similar behavior in `memcpy`, which is exactly what we have:

```assembly
iBoot:00000001FC15A7E0 wrap_memcpy_safe_allocation             ; CODE XREF: sub_1FC052C08+21C↑p
iBoot:00000001FC15A7E0                                         ; sub_1FC054C94+538↑p ...
iBoot:00000001FC15A7E0
iBoot:00000001FC15A7E0 var_70          = -0x70
iBoot:00000001FC15A7E0 var_30          = -0x30
iBoot:00000001FC15A7E0 var_20          = -0x20
iBoot:00000001FC15A7E0 var_10          = -0x10
iBoot:00000001FC15A7E0 var_s0          =  0
iBoot:00000001FC15A7E0
iBoot:00000001FC15A7E0                 PACIBSP
iBoot:00000001FC15A7E4                 SUB             SP, SP, #0x80
iBoot:00000001FC15A7E8                 STP             X24, X23, [SP,#0x70+var_30]
iBoot:00000001FC15A7EC                 STP             X22, X21, [SP,#0x70+var_20]
iBoot:00000001FC15A7F0                 STP             X20, X19, [SP,#0x70+var_10]
iBoot:00000001FC15A7F4                 STP             X29, X30, [SP,#0x70+var_s0]
iBoot:00000001FC15A7F8                 ADD             X29, SP, #0x70
iBoot:00000001FC15A7FC ; set the following registers:
iBoot:00000001FC15A7FC ; MOV             X21, X2 (length)
iBoot:00000001FC15A7FC ; MOV             X22, X1 (src)
iBoot:00000001FC15A7FC ; MOV             X20, X0 (dst)
iBoot:00000001FC15A7FC ; MOV             X19, X8
iBoot:00000001FC15A7FC                 BL              call_check_ptr_bounds_
iBoot:00000001FC15A800                 BL              do_check_ptr_bounds_x22
iBoot:00000001FC15A804                 LDR             X23, [X20,#safe_allocation.type]
iBoot:00000001FC15A808                 MOV             X0, X23
iBoot:00000001FC15A80C ; check if dst's type is a primitive type
iBoot:00000001FC15A80C                 BL              is_primitive_type
iBoot:00000001FC15A810                 LDR             X24, [X22,#safe_allocation.type]
iBoot:00000001FC15A814                 CBZ             W0, loc_1FC15A824
iBoot:00000001FC15A818                 MOV             X0, X24
iBoot:00000001FC15A81C ; check if src's type is a primitive type
iBoot:00000001FC15A81C                 BL              is_primitive_type
iBoot:00000001FC15A820                 TBNZ            W0, #0, loc_1FC15A854
iBoot:00000001FC15A824 ; at least one of the allocation (src or dst)
iBoot:00000001FC15A824 ; are not primitive type. Call the type
iBoot:00000001FC15A824 ; equal implementation to see if they are equal
iBoot:00000001FC15A824
iBoot:00000001FC15A824 loc_1FC15A824                           ; CODE XREF: wrap_memcpy_safe_allocation+34↑j
iBoot:00000001FC15A824                 MOV             X8, SP
iBoot:00000001FC15A828 ; dst's type descriptor ptr
iBoot:00000001FC15A828                 MOV             X0, X23
iBoot:00000001FC15A82C ; src's type descriptor ptr
iBoot:00000001FC15A82C                 MOV             X1, X24
iBoot:00000001FC15A830                 BL              compare_types
iBoot:00000001FC15A834                 LDR             W8, [SP,#0x70+var_70]
iBoot:00000001FC15A838                 CMP             W8, #1
iBoot:00000001FC15A83C                 B.NE            detect_memcpy_bad_type
iBoot:00000001FC15A840                 LDR             X0, [X20,#safe_allocation.type]
iBoot:00000001FC15A844                 BL              get_type_length
iBoot:00000001FC15A848 ; divide and multiply the length argument
iBoot:00000001FC15A848 ; by the type's size, to detect
iBoot:00000001FC15A848 ; partial/unalignment writes
iBoot:00000001FC15A848                 UDIV            X8, X21, X0
iBoot:00000001FC15A84C                 MSUB            X8, X8, X0, X21
iBoot:00000001FC15A850                 CBNZ            X8, detect_memcpy_bad_n
iBoot:00000001FC15A854 ; ok, we have one of two possible cases:
iBoot:00000001FC15A854 ; A) types are either both primitives
iBoot:00000001FC15A854 ; B) type are both equals,
iBoot:00000001FC15A854 ;    and dst's length is verified w.r.t the len argument
iBoot:00000001FC15A854 ; which means we can do a raw copy
iBoot:00000001FC15A854
iBoot:00000001FC15A854 loc_1FC15A854                           ; CODE XREF: wrap_memcpy_safe_allocation+40↑j
iBoot:00000001FC15A854                 BL              memcpy_safe_allocations_x22_to_x20
iBoot:00000001FC15A858                 BL              move_safe_allocation_x20_to_x19
iBoot:00000001FC15A85C                 LDP             X29, X30, [SP,#0x70+var_s0]
iBoot:00000001FC15A860                 LDP             X20, X19, [SP,#0x70+var_10]
iBoot:00000001FC15A864                 LDP             X22, X21, [SP,#0x70+var_20]
iBoot:00000001FC15A868                 LDP             X24, X23, [SP,#0x70+var_30]
iBoot:00000001FC15A86C                 ADD             SP, SP, #0x80
iBoot:00000001FC15A870                 RETAB
iBoot:00000001FC15A874 ; ---------------------------------------------------------------------------
iBoot:00000001FC15A874
iBoot:00000001FC15A874 detect_memcpy_bad_type                  ; CODE XREF: wrap_memcpy_safe_allocation+5C↑j
iBoot:00000001FC15A874                 BL              call_panic_memcpy_bad_type
iBoot:00000001FC15A878 ; ---------------------------------------------------------------------------
iBoot:00000001FC15A878
iBoot:00000001FC15A878 detect_memcpy_bad_n                     ; CODE XREF: wrap_memcpy_safe_allocation+70↑j
iBoot:00000001FC15A878                 BL              call_panic_memcpy_bad_n
iBoot:00000001FC15A878 ; End of function wrap_memcpy_safe_allocation
```

And indeed, the function `is_primitive_type` is exactly what we saw before:

```assembly
iBoot:00000001FC15A8C0 is_primitive_type                       ; CODE XREF: wrap_memcpy_safe_allocation+2C↑p
iBoot:00000001FC15A8C0                                         ; wrap_memcpy_safe_allocation+3C↑p ...
iBoot:00000001FC15A8C0                 LDR             X8, [X0]
iBoot:00000001FC15A8C4                 AND             X8, X8, #0xFFFFFFFFFFFFFFF8
iBoot:00000001FC15A8C8                 LDRB            W9, [X8]
iBoot:00000001FC15A8CC                 TST             W9, #7
iBoot:00000001FC15A8D0                 B.EQ            check_number_of_pointer_elements
iBoot:00000001FC15A8D4 ; one of the 3 LSB bits is non-0, therefore return false
iBoot:00000001FC15A8D4                 MOV             W0, #0
iBoot:00000001FC15A8D8                 RET
iBoot:00000001FC15A8DC ; ---------------------------------------------------------------------------
iBoot:00000001FC15A8DC
iBoot:00000001FC15A8DC check_number_of_pointer_elements        ; CODE XREF: do_check_ptr_bounds+54↑j
iBoot:00000001FC15A8DC                 LDR             X8, [X8,#0x20]
iBoot:00000001FC15A8E0 ; check if number of pointer elements == 0
iBoot:00000001FC15A8E0 ; return true if it is, otherwise return false
iBoot:00000001FC15A8E0                 LSR             X8, X8, #0x23 ; '#'
iBoot:00000001FC15A8E4                 CMP             X8, #0
iBoot:00000001FC15A8E8                 CSET            W0, EQ
iBoot:00000001FC15A8EC                 RET
iBoot:00000001FC15A8EC ; End of function do_check_ptr_bounds
```

And `memcpy_safe_allocations_x22_to_x20` is simply:

```assembly
iBoot:00000001FC15AD88 memcpy_safe_allocations_x22_to_x20     
iBoot:00000001FC15AD88                                         
iBoot:00000001FC15AD88                 LDR             X0, [X20,#safe_allocation.raw_ptr]
iBoot:00000001FC15AD8C                 LDR             X1, [X22,#safe_allocation.raw_ptr]
iBoot:00000001FC15AD90                 MOV             X2, X21
iBoot:00000001FC15AD94                 B               _memcpy
iBoot:00000001FC15AD94 ; End of function memcpy_safe_allocations_x22_to_x20
```

Which is exactly:

```c++
memcpy(dst->raw_ptr, src->raw_ptr, length);
```

Absolutely beautiful. So, the function `wrap_memcpy_safe_allocation` only copies `src` to `dst` if all the conditions below are met:

1. both types are primitive-types OR types are equals.
2. the length argument does not go OOB from the `dst`'s length, which we get from the `safe_allocation` structure.
3. the length argument is aligned with respect to the type's size, so there will be no partial initialization.

Please note that because Apple has here the type's exact size, they can enforce a better restriction on `memset`/`memcpy` than just "don't go OOB" (which they gain anyways by simply using the lower/upper bound pointers from the `safe_allocation`). In addition, Apple verifies here that the writes to the structure won't leave "partial uninitialized" areas and that writes actually look as they should (from an alignment perspective).

For a scenario where this check could be very important, consider the case of calling `memset` on an array `struct A* arr`. With this `panic_memset_bad_n` validation in place, iBoot makes sure you could never leave partial uninitialized elements in your array.

The only thing I didn't show you, is `get_type_length`. Let's do that right now :)

### Encoding and format

The first thing I would like to do is prove that `get_type_length` does what I claim it does. It certainly looks correct (we have a super clear panic there, "*panic_memcpy_bad_n*"), and it compares the return value to the length argument of memcpy (`n`). But still, I think we can learn more by looking at the implementation and understanding what it does.

While reversing the casting implementation, I found an interesting function: `firebloom_type_equalizer `. This function has a lot of useful strings, which give us a lot of information. For instance, check out this code:

```assembly
iBoot:00000001FC1A3FA4                 LDR             X9, [X26,#0x20]
iBoot:00000001FC1A3FA8                 LDR             X8, [X20,#0x20]
iBoot:00000001FC1A3FAC                 LSR             X10, X9, #0x23 ; '#'
iBoot:00000001FC1A3FB0                 LSR             X23, X8, #0x23 ; '#'
iBoot:00000001FC1A3FB4                 CMP             W9, W8
iBoot:00000001FC1A3FB8                 CBZ             W11, bne_size_mismatch
iBoot:00000001FC1A3FBC                 B.CC            left_has_smaller_size_than_right
iBoot:00000001FC1A3FC0                 CMP             W10, W23
iBoot:00000001FC1A3FC4                 B.CC            left_has_fewer_pointer_elements_than_right
```

From this snippet alone, we learn the following:

* The type descriptor structure has the **type's length** stored at offset +0x20. This value is 32-bit size.
* At the high 29 bits of this value, there is another useful information: **"number of pointer elements"**. Interesting!

Now I can show you `get_type_length`, which is called from the `wrap_memset_safe_allocation` and `wrap_memcpy_safe_allocation`:

```assembly
iBoot:00000001FC15A964 get_type_length                         ; CODE XREF: wrap_memcpy_safe_allocation+64↑p
iBoot:00000001FC15A964                                         ; wrap_memset_safe_allocation+48↓p ...
iBoot:00000001FC15A964                 LDR             X8, [X0]
iBoot:00000001FC15A968                 AND             X8, X8, #0xFFFFFFFFFFFFFFF8
iBoot:00000001FC15A96C                 LDR             W9, [X8]
iBoot:00000001FC15A970                 AND             W9, W9, #7
iBoot:00000001FC15A974                 CMP             W9, #5
iBoot:00000001FC15A978                 CCMP            W9, #1, #4, NE
iBoot:00000001FC15A97C                 B.NE            loc_1FC15A988
iBoot:00000001FC15A980 ; one instance of the element
iBoot:00000001FC15A980                 MOV             W0, #1
iBoot:00000001FC15A984                 RET
iBoot:00000001FC15A988 ; ---------------------------------------------------------------------------
iBoot:00000001FC15A988
iBoot:00000001FC15A988 loc_1FC15A988                           ; CODE XREF: call_panic_memcpy_bad_type+58↑j
iBoot:00000001FC15A988                 CBNZ            W9, return_0
iBoot:00000001FC15A98C ; load the low 32-bit from this value,
iBoot:00000001FC15A98C ; which represents the length of this type
iBoot:00000001FC15A98C                 LDR             W0, [X8,#0x20]
iBoot:00000001FC15A990                 RET
iBoot:00000001FC15A994 ; ---------------------------------------------------------------------------
iBoot:00000001FC15A994
iBoot:00000001FC15A994 return_0                                ; CODE XREF: call_panic_memcpy_bad_type:loc_1FC15A988↑j
iBoot:00000001FC15A994                 MOV             X0, #0
iBoot:00000001FC15A998                 RET
```

Does the AND with `0xFFFFFFFFFFFFFFF8` looks familiar? If so, it's likely because you saw it at the beginning of this blogpost, when I showed how primitive types are checked at `cast_impl`. The first pointer in the type descriptor seems to encode some information at its 3 LSBs, and therefore each time we need to dereference that, we need to mask them out with zeros.

And indeed, this function returns the 32-bit value stored at offset 0x20 in the type descriptor, which is the type's length.

### Primitive types in Firebloom

Hang on, there is something really interesting about the 64-bit value stored at offset +0x20. We know that:

* the lower half (32-bit) represents the length of the type.
* unknown 3 bits in the middle.
* the top 29 bits represent the "number of pointer elements".

We've seen this value before. Take a look again at the code from `cast_impl` and in `is_primitive_type`. In these places, the code checks the "number of pointer elemnets" and compares it to 0 -- **and only if it equals to 0, the type is considered primitive**. Makes perfect sense!

Now, let's focus on `is_primitive_type`. The logic of this function is as follows:

1. dereference the `type` pointer and mask out the 3 LSBs bits. Now, dereference this pointer.
2. if any of the 3 LSBs is set, return false.
3. read the 64-bit value stored at +0x20.
4. extract the 29 high bits - which we know it's "number of pointer elements".
5. return true if this value is 0; otherwise, return false.

In other words:

* if any of the 3 LSBs is set, the type is not primitive - return false.
* if the 3 bits are 0s, then the code checks if the numer of pointer elements equals 0. A type cannot be primitive, if it has pointer elements.

Ok, so the function `is_primitive_type` returns true ONLY if all the 3 bits are 0s and the number of pointer elements is zero. Which is what we would expect, because you shouldn't be allow to copy bytes between non-primitive types, unless they are equal (in some way).

To get a better understanding, let's look up at xrefs of `is_primitive_type`. This function is called (only) by `wrap_memset_safe_allocation` and `wrap_memcpy_safe_allocation`, to determine if they can simply call `memset` and `memcpy` directly, without further validations.

Let's see:

1. The function `wrap_memset_safe_allocation` calls `is_primitive_type`, and check for the return value (0 or 1). If it returned 1, it calls `memset` directly. Otherwise, it checks if the `c` argument (the pattern to memset) is zero, i.e. the character `\x00`. If it's not zero, it calls `panic_memset_bad_type`.


   Ok, so iBoot refuses to call `memset` on non-primitive types, with any argument `c` that is not 0.

   

2. The function `wrap_memcpy_safe_allocation` calls `is_primitive_type` twice - once for `dst`, and once for `src`. If both calls returned 1, it calls `memcpy` directly. Otherwise, it calls `compare_types`, to properly compare the types, using `firebloom_type_equalizer`. 


   So for `memcpy`, iBoot refuses to copy the content from/to non-primitive types unless they are equal (in one well defined way).

This is very interesting, and it makes sense. It's pretty cool to see such kind of type safety validations in place.

### Examples for types!

Before I wrap this up, I want to show here few examples of types in the iBoot binary. As I wrote in the previous blogpost, individual callers to the `do_safe_allocation*` set the relevant types (if it's not the default type the `do_safe_allocation*` functions set). Let's check out some examples, by following callers to the `do_safe_allocation*` functions and see that the format we understood from the binary makes sense.

#### Example 1

Let's start with the following code:

```assembly
iBoot:00000001FC10E4DC                 LDR             W1, [X22,#0x80]
iBoot:00000001FC10E4E0 ; the `safe_allocation` structure to initialize
iBoot:00000001FC10E4E0                 ADD             X8, SP, #0xD0+v_safe_allocation
iBoot:00000001FC10E4E4                 MOV             W0, #1
iBoot:00000001FC10E4E8                 BL              do_safe_allocation_calloc
iBoot:00000001FC10E4EC                 LDP             X0, X1, [SP,#0xD0+v_safe_allocation]
iBoot:00000001FC10E4F0                 LDP             X2, X3, [SP,#0xD0+v_safe_allocation.upper_bound_ptr]
iBoot:00000001FC10E4F4                 BL              sub_1FC10E1C4
iBoot:00000001FC10E4F8                 ADRP            X8, #qword_1FC2339C8@PAGE
iBoot:00000001FC10E4FC                 LDR             X8, [X8,#qword_1FC2339C8@PAGEOFF]
iBoot:00000001FC10E500                 CBZ             X8, detect_ptr_null
iBoot:00000001FC10E504                 CMP             X23, X19
iBoot:00000001FC10E508                 B.HI            detected_ptr_under
iBoot:00000001FC10E50C                 CMP             X28, X19
iBoot:00000001FC10E510                 B.LS            detected_ptr_over
iBoot:00000001FC10E514                 MOV             X20, X0
iBoot:00000001FC10E518                 MOV             X27, X1
iBoot:00000001FC10E51C ; X19 here is a base of some allocation,
iBoot:00000001FC10E51C ; set X8 to be raw_ptr+0x50, which is
iBoot:00000001FC10E51C ; the upper_bound_ptr
iBoot:00000001FC10E51C                 ADD             X8, X19, #0x50 ; 'P'
iBoot:00000001FC10E520 ; re-initialize the safe_allocation:
iBoot:00000001FC10E520 ; set X19 as both raw_ptr and lower_bound_ptr
iBoot:00000001FC10E520                 STP             X19, X19, [SP,#0xD0+v_safe_allocation]
iBoot:00000001FC10E524 ; take the relevant type pointer, set it in
iBoot:00000001FC10E524 ; safe_allocation->type (offset +0x18,
iBoot:00000001FC10E524 ; which is one qword after upper_bound_ptr).
iBoot:00000001FC10E524 ;
iBoot:00000001FC10E524 ; Note: the size of this type should be +0x50
iBoot:00000001FC10E524                 ADRL            X9, off_1FC2D09E8
iBoot:00000001FC10E52C                 STP             X8, X9, [SP,#0xD0+v_safe_allocation.upper_bound_ptr]
```

Ok, interesting. So we have a call to `do_safe_allocation_calloc`, and then the code sets the `type` pointer to `off_1FC2D09E8`. Let's see what we have there:

```
iBoot:00000001FC2D09E8 off_1FC2D09E8   DCQ off_1FC2D0760+2     ; DATA XREF: sub_1FC1071C0+33C↑o
iBoot:00000001FC2D09E8                                         ; sub_1FC107D90+188↑o ...
```

Awesome! Indeed, the value pointed by this pointer is some address+2 (recall the mask `0xFFFFFFFFFFFFFFF8`? :P). Let's dereference this pointer, and at offset +0x20 of that, I expect to see:

* type's length (lower 32 bits)
* number of pointers in type (upper 29 bits)

And indeed:

```
iBoot:00000001FC2D0760 off_1FC2D0760   DCQ off_1FC2D0760       ; DATA XREF: iBoot:off_1FC2D0760↓o
iBoot:00000001FC2D0760                                         ; iBoot:00000001FC2D0A98↓o ...
iBoot:00000001FC2D0768                 ALIGN 0x20
iBoot:00000001FC2D0780                 DCQ 0x1300000050, 0x100000000
```

Fantastic! The value at offset +0x20 is 0x1300000050, which exactly matches our expectations:

* type's length = 0x50 (which is exactly what we exepcted!)
* number of pointers = 2 (0x1300000050>>0x23)

Great, these values makes a lot of sense!

#### Example 2

We can't skip the default type, right? As you saw in the previous blogpost, all the `do_safe_allocation*` functions set a default type pointer at offset +0x18, and individual callers change this type if needed (as in the two examples above). See below the xrefs to `default_type_ptr`:

![image](https://github.com/saaramar/iBoot_firebloom_type_desc/raw/main/files/default_type_ptr_xrefs.png)

My expectation is to see here some "default" values, i.e. type's length = 1, number_of_pointers = 0, and indication for a primitive type. Let's see:

```
iBoot:00000001FC2D6EF8 default_type_ptr DCQ default_type_ptr   ; DATA XREF: __firebloom_panic+2C↑o
iBoot:00000001FC2D6EF8                                         ; sub_1FC15AD98+1FC↑o ...
iBoot:00000001FC2D6F00                 DCQ 0, 0, 0
iBoot:00000001FC2D6F18                 DCQ 0x100000001
```

Perfect! The `default_type_ptr` points to itself (fine), and at offset +0x20, we have the value 0x0000000100000001, which means:

* type's length = 0x1
* number of pointers = 0 (0x100000001>>0x23)
* and, of course, this type is primitive (3 LSB bits are 0s and the number of pointers is 0).

Awesome!

### Casting

The casting implementation is really nice. It requires a lot of text to describe how it works, so I'll keep it for some other time. I do want, however, to encourage more folks to look at the binary, so check out this really cool `cast_failed` function, with super helpful strings and calls to `wrap_firebloom_type_kind_dump`:

```assembly
iBoot:00000001FC1A18A8 cast_failed                             ; CODE XREF: cast_impl+D00↑p
iBoot:00000001FC1A18A8                                         ; sub_1FC1A1594+C8↑p
iBoot:00000001FC1A18A8
iBoot:00000001FC1A18A8 var_D0          = -0xD0
iBoot:00000001FC1A18A8 var_C0          = -0xC0
iBoot:00000001FC1A18A8 var_B8          = -0xB8
iBoot:00000001FC1A18A8 var_20          = -0x20
iBoot:00000001FC1A18A8 var_10          = -0x10
iBoot:00000001FC1A18A8 var_s0          =  0
iBoot:00000001FC1A18A8
iBoot:00000001FC1A18A8                 PACIBSP
iBoot:00000001FC1A18AC                 SUB             SP, SP, #0xE0
iBoot:00000001FC1A18B0                 STP             X22, X21, [SP,#0xD0+var_20]
iBoot:00000001FC1A18B4                 STP             X20, X19, [SP,#0xD0+var_10]
iBoot:00000001FC1A18B8                 STP             X29, X30, [SP,#0xD0+var_s0]
iBoot:00000001FC1A18BC                 ADD             X29, SP, #0xD0
iBoot:00000001FC1A18C0                 MOV             X19, X3
iBoot:00000001FC1A18C4                 MOV             X20, X2
iBoot:00000001FC1A18C8                 MOV             X21, X1
iBoot:00000001FC1A18CC                 MOV             X22, X0
iBoot:00000001FC1A18D0                 ADD             X0, SP, #0xD0+var_C0
iBoot:00000001FC1A18D4                 BL              sub_1FC1A9A08
iBoot:00000001FC1A18D8                 LDR             X8, [X22,#0x30]
iBoot:00000001FC1A18DC                 STR             X8, [SP,#0xD0+var_D0]
iBoot:00000001FC1A18E0                 ADR             X1, aCastFailedS ; "cast failed: %s\n"
iBoot:00000001FC1A18E4                 NOP
iBoot:00000001FC1A18E8                 ADD             X0, SP, #0xD0+var_C0
iBoot:00000001FC1A18EC                 BL              do_trace
iBoot:00000001FC1A18F0                 LDR             X8, [X22,#0x38]
iBoot:00000001FC1A18F4                 CBZ             X8, loc_1FC1A1948
iBoot:00000001FC1A18F8                 LDR             X8, [X22,#0x40]
iBoot:00000001FC1A18FC                 CBZ             X8, loc_1FC1A1948
iBoot:00000001FC1A1900                 ADR             X1, aTypesNotEqual ; "types not equal: "
iBoot:00000001FC1A1904                 NOP
iBoot:00000001FC1A1908                 ADD             X0, SP, #0xD0+var_C0
iBoot:00000001FC1A190C                 BL              do_trace
iBoot:00000001FC1A1910                 LDR             X0, [X22,#0x38]
iBoot:00000001FC1A1914                 ADD             X1, SP, #0xD0+var_C0
iBoot:00000001FC1A1918                 BL              wrap_firebloom_type_kind_dump
iBoot:00000001FC1A191C                 ADR             X1, aAnd ; " and "
iBoot:00000001FC1A1920                 NOP
iBoot:00000001FC1A1924                 ADD             X0, SP, #0xD0+var_C0
iBoot:00000001FC1A1928                 BL              do_trace
iBoot:00000001FC1A192C                 LDR             X0, [X22,#0x40]
iBoot:00000001FC1A1930                 ADD             X1, SP, #0xD0+var_C0
iBoot:00000001FC1A1934                 BL              wrap_firebloom_type_kind_dump
iBoot:00000001FC1A1938                 ADR             X1, asc_1FC1C481F ; "\n"
iBoot:00000001FC1A193C                 NOP
iBoot:00000001FC1A1940                 ADD             X0, SP, #0xD0+var_C0
iBoot:00000001FC1A1944                 BL              do_trace
iBoot:00000001FC1A1948
iBoot:00000001FC1A1948 loc_1FC1A1948                           ; CODE XREF: cast_failed+4C↑j
iBoot:00000001FC1A1948                                         ; cast_failed+54↑j
iBoot:00000001FC1A1948                 ADR             X1, aWhenTestingPtr ; "when testing ptr type "
iBoot:00000001FC1A194C                 NOP
iBoot:00000001FC1A1950                 ADD             X0, SP, #0xD0+var_C0
iBoot:00000001FC1A1954                 BL              do_trace
iBoot:00000001FC1A1958                 ADD             X1, SP, #0xD0+var_C0
iBoot:00000001FC1A195C                 MOV             X0, X21
iBoot:00000001FC1A1960                 BL              wrap_firebloom_type_kind_dump
iBoot:00000001FC1A1964                 ADR             X1, aAndCastType ; " and cast type "
iBoot:00000001FC1A1968                 NOP
iBoot:00000001FC1A196C                 ADD             X0, SP, #0xD0+var_C0
iBoot:00000001FC1A1970                 BL              do_trace
iBoot:00000001FC1A1974                 ADD             X1, SP, #0xD0+var_C0
iBoot:00000001FC1A1978                 MOV             X0, X20
iBoot:00000001FC1A197C                 BL              wrap_firebloom_type_kind_dump
iBoot:00000001FC1A1980                 STR             X19, [SP,#0xD0+var_D0]
iBoot:00000001FC1A1984                 ADR             X1, aWithSizeZu ; " with size %zu\n"
iBoot:00000001FC1A1988                 NOP
iBoot:00000001FC1A198C                 ADD             X0, SP, #0xD0+var_C0
iBoot:00000001FC1A1990                 BL              do_trace
iBoot:00000001FC1A1994                 LDR             X0, [SP,#0xD0+var_B8]
iBoot:00000001FC1A1998                 BL              call_firebloom_panic
iBoot:00000001FC1A1998 ; End of function cast_failed
```

This function is called from `cast_impl`, which has some great strings that really help give you context (this is just a partial list):

```
"Cannot cast dynamic void type to anything"
"types not equal"
"Pointer is not in bounds"
"Cannot cast primitive type to non-primitive type"
"Target type has larger size than the bounds of the pointer"
"Pointer is not in phase"
"Bad subtype result kind"
```

All those strings are used in `cast_impl`.

## Sum up

I hope these two blogposts helped you understand better how iBoot Firebloom works and how Apple achieved these fantastic security properties described on Apple Platform Security, under [Memory safe iBoot implementation](https://support.apple.com/en-il/guide/security/sec30d8d9ec1/web).

I think Apple did a fantastic job here and achieved something non-trivial with Firebloom. It's not trivial to enforce these security properties, and Apple pulled it off. True, as I mentioned in the previous blogpost, this implementation of Firebloom is (extremely) expensive. But again, for iBoot, it works very well (from the reasons I've covered in the previous blogpost). And I have to admit it's pretty cool :)



I hope you enjoyed this blogpost.

Thanks,

Saar Amar.

