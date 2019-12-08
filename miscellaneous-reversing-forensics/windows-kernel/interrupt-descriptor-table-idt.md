# Interrupt Descriptor Table - IDT

{% hint style="info" %}
WIP
{% endhint %}

## At a Glance

* Interrupts could be thought of as `notifications` to the CPU that tells it that `some event` happened on the system. Classic examples of interrupts are hardware interrupts such as mouse or keyboard interactions, network packet activity and hardware generated exceptions such as a division by zero or a breakpoint - interrupts 0x00 and 0x03 respectively
* Once the CPU gets interrupted, it needs to stop doing what it was doing and respond to that interrupt immediately
* CPU knows how to respond \(what kernel routines to execute\) to the received interrupts by looking up Interrupt Service Routines \(ISR\) that are found in the Interrupt Descriptor Table \(IDT\)
* IDT is a list of IDT descriptor entries which are 8 or 16 bytes in size depending on the architecture 
  * Each IDT descriptor is a complex kernel memory structure, cotaining a pointer to its ISR among other information
* Pointer to IDT is stored in an `IDTR` register for each physical processor or in other words, each processor has its own `IDTR` register pointing to its own Interrupt Descriptor Table

## IDT Location

We can check where the Interrupt Descriptor Table is located in kernel by reading the register `IDTR`:

```text
r idtr
```

![](../../.gitbook/assets/image%20%2817%29.png)

As noted later, the command `!idt` allows us to dump the Interrupt Descriptor Table contents and it also confirms that the IDT is located at ``fffff803`536dda00`` as shown below:

![](../../.gitbook/assets/image%20%28258%29.png)

## Dumping IDT

We can dump the IDT and see addresses of Interrupt Service Routines for a given interrupt. Below is a snippet of the Interrupt Descriptor Table:

```erlang
kd> !idt

Dumping IDT: fffff80091456000

00:	fffff8008f37e100 nt!KiDivideErrorFaultShadow
01:	fffff8008f37e180 nt!KiDebugTrapOrFaultShadow	Stack = 0xFFFFF8009145A9E0
02:	fffff8008f37e200 nt!KiNmiInterruptShadow	Stack = 0xFFFFF8009145A7E0
03:	fffff8008f37e280 nt!KiBreakpointTrapShadow
...
90:	fffff8008f37f680 i8042prt!I8042MouseInterruptService (KINTERRUPT ffffd4816353e8c0)
a0:	fffff8008f37f700 i8042prt!I8042KeyboardInterruptService (KINTERRUPT ffffd4816353ea00)
...
```

Below shows the IDT dumping and IRS code execution in action:

* IDT table is dumped with !idt
* In the IDT, Interrupt `a0` at `fffff8008f37f700` located. It is pointing to an IRS `i8042prt!I8042KeyboardInterruptService`. This routine is executed when a keyboard event such as a keypress is registered on the OS
* Putting a breakpoint on 

  `i8042prt!I8042KeyboardInterruptService`

* Once the breakpoint is set, a key is pressed on the OS login prompt and our breakpoint is hit, confirming that 

  `i8042prt!I8042KeyboardInterruptService` indeed handles keyboard interrupts

![](../../.gitbook/assets/keyboard-interrupt.gif)

## IDT Entry

IDT is made up of IDT entries `_KIDTENTRY64` which is a kernel memory structure and is defined like so:

```erlang
kd> dt nt!_KIDTENTRY64
   +0x000 OffsetLow        : Uint2B
   +0x002 Selector         : Uint2B
   +0x004 IstIndex         : Pos 0, 3 Bits
   +0x004 Reserved0        : Pos 3, 5 Bits
   +0x004 Type             : Pos 8, 5 Bits
   +0x004 Dpl              : Pos 13, 2 Bits
   +0x004 Present          : Pos 15, 1 Bit
   +0x006 OffsetMiddle     : Uint2B
   +0x008 OffsetHigh       : Uint4B
   +0x00c Reserved1        : Uint4B
   +0x000 Alignment        : Uint8B
```

Members `OffsetLow`, `OffsetMiddle` and `OffsetHigh` at offsets 0x0, 0x006 and 0x008 make up the virtual address in the kernel and it's where the code execution will be transferred to by the CPU once that particular interrupt takes place.

## IDT Entry for the Keyboard Interrupt

As an example, let's inspect the IDT entry for the keyboard interrupt which is located at index `a0` in the IDT table as discovered earlier:

```erlang
!idt a0
```

![](../../.gitbook/assets/image%20%28124%29.png)

From earlier, we also know that the IDT resides at `fffff803536dd000`:

```erlang
kd> r idtr
idtr=fffff803536dd000
```

We can get the location of the `a0` IDT entry by adding `0xa0*0x10` \(interrupt index `a0` times `0x10` since a descriptor entry is 16 bytes in size\) to the IDT table address:

```erlang
kd> dq idtr + (0xa0*0x10) L2
fffff803`536dda00  51568e00`0010e700 00000000`fffff803
```

With the above information, we can overlay the `a0` interrupt descriptor entry with `_KIDTENTRY64` and inspect `a0` IDT entry's content:

```erlang
kd> dt _kidtentry64 (idtr + (0xa0*0x10))
ntdll!_KIDTENTRY64
   +0x000 OffsetLow        : 0xe700
   +0x002 Selector         : 0x10
   +0x004 IstIndex         : 0y000
   +0x004 Reserved0        : 0y00000 (0)
   +0x004 Type             : 0y01110 (0xe)
   +0x004 Dpl              : 0y00
   +0x004 Present          : 0y1
   +0x006 OffsetMiddle     : 0x5156
   +0x008 OffsetHigh       : 0xfffff803
   +0x00c Reserved1        : 0
   +0x000 Alignment        : 0x51568e00`0010e700
```

Below re-enforces that the Offset\(High\|Middle\|Low\) form the virtual address where the code execution will get transferred when `a0` interrupt is raised by the keyboard:

![](../../.gitbook/assets/image%20%2876%29.png)

Below shows that once the `a0` interrupt occurs, the first instructions to be executed by the CPU will be:

* FFFFFFFFFFFFFF**A0** will be pushed on the stack 
* jump to ``fffff803`5156ea40`` will happen

![](../../.gitbook/assets/image%20%2879%29.png)

## KINTERUPT

`KINTERUPT` is a kernel memory structure that holds information about an interrupt. The key member of this structure for this lab is the member located at offset 0x18 - it's a pointer to the Interupt Service Routine - the routine that is responsible for actually handling the interrupt:

```erlang
dt nt!_KINTERRUPT
   +0x000 Type             : Int2B
   +0x002 Size             : Int2B
   +0x008 InterruptListEntry : _LIST_ENTRY
   +0x018 ServiceRoutine   : Ptr64     unsigned char 
   ...
   +0x0f8 Padding          : [8] UChar
```

As an example - from earlier, we know that the ISR `i8042prt!I8042KeyboardInterruptService` for keyboard interrupts is located at `ffffd4816353ea00`, therefore we can inspect the `KINTERUPT` structure of that our interrupt by overlaying it with memory contents at `ffffd4816353ea00`:

```erlang
dt nt!_KINTERRUPT ffffd4816353ea00
```

This allows us to confirm that the Interrupt Service Routine is again pointing at `i8042prt!I8042KeyboardInterruptService`: 

![](../../.gitbook/assets/image%20%28301%29.png)

## Finding \_KINTERRUPT

In order to find the location of the `_KINTERRUPT` for a given interrupt, we need to leverage the following memory locations and structures.

Process Control Region or PCR \(\_KPCR memory structure in kernel\) stores information about a given processor:

```erlang
kd> dt _KPCR
ntdll!_KPCR
   +0x000 NtTib            : _NT_TIB
   +0x000 GdtBase          : Ptr64 _KGDTENTRY64
   +0x008 TssBase          : Ptr64 _KTSS64
   +0x010 UserRsp          : Uint8B
   +0x018 Self             : Ptr64 _KPCR
   +0x020 CurrentPrcb      : Ptr64 _KPRCB
   +0x028 LockArray        : Ptr64 _KSPIN_LOCK_QUEUE
   +0x030 Used_Self        : Ptr64 Void
   +0x038 IdtBase          : Ptr64 _KIDTENTRY64
   +0x040 Unused           : [2] Uint8B
   +0x050 Irql             : UChar
   +0x051 SecondLevelCacheAssociativity : UChar
   +0x052 ObsoleteNumber   : UChar
   +0x053 Fill0            : UChar
   +0x054 Unused0          : [3] Uint4B
   +0x060 MajorVersion     : Uint2B
   +0x062 MinorVersion     : Uint2B
   +0x064 StallScaleFactor : Uint4B
   +0x068 Unused1          : [3] Ptr64 Void
   +0x080 KernelReserved   : [15] Uint4B
   +0x0bc SecondLevelCacheSize : Uint4B
   +0x0c0 HalReserved      : [16] Uint4B
   +0x100 Unused2          : Uint4B
   +0x108 KdVersionBlock   : Ptr64 Void
   +0x110 Unused3          : Ptr64 Void
   +0x118 PcrAlign1        : [24] Uint4B
   +0x180 Prcb             : _KPRCB
```

\_KPCR location can be found like this:

```erlang
kd> ? @$pcr
Evaluate expression: -8781847822336 = fffff803`51148000

kd> !pcr
KPCR for Processor 0 at fffff80351148000:
    Major 1 Minor 1
	NtTib.ExceptionList: fffff803536dffb0
	    NtTib.StackBase: fffff803536de000
	   NtTib.StackLimit: 0000000000000000
...snip...
```

Inside the KPCR, at offset 0x180 there is a member that points to a Process Control Block memory structure \_KPRCB which contains information about the state of a processor.

The key member we're interested when trying to find the `_KINTERRUPT` memory location for a given interrupt is `InterruptObject` since it contains a list of pointers to a list of `_KINTERRUPT` objects and it is located at offset `0x2e80` as shown below:

```erlang
kd> dt _KPRCB
ntdll!_KPRCB
   +0x000 MxCsr            : Uint4B
   +0x004 LegacyNumber     : UChar
   +0x005 ReservedMustBeZero : UChar
   ....
   +0x2e80 InterruptObject  : [256] Ptr64 Void //256 pointers max as notes earlier
   ....
```

With the above knowledge, we can now find the `_KINTERRUPT` location for the keyboard interrupt `a0`:

```erlang
kd> dt @$pcr nt!_KPCR.Prcb.InterruptObject[a0]
```

Below confirms that the `_KINTERRUPT` for the interrupt `a0` we found manually matches that given by the `!idt` command:

![](../../.gitbook/assets/image%20%28167%29.png)

## References

{% embed url="https://nagareshwar.securityxploded.com/2014/03/20/code-injection-and-api-hooking-techniques/" %}

{% embed url="https://www.linux.com/tutorials/kernel-interrupt-overview/" %}

{% embed url="https://en.wikipedia.org/wiki/Interrupt\_handler" %}

{% embed url="https://relearex.wordpress.com/2017/12/27/hooking-series-part-ii-interrupt-descriptor-table-hooking/" %}

{% embed url="https://wiki.osdev.org/Interrupt\_Descriptor\_Table" %}

{% embed url="https://www.codemachine.com/article\_interruptdispatching.html" %}

{% embed url="https://www.linux.com/tutorials/kernel-interrupt-overview/" %}

{% embed url="https://rayanfam.com/topics/fooling-windows-about-cpu/" %}

{% embed url="https://resources.infosecinstitute.com/hooking-idt/\#gref" %}
