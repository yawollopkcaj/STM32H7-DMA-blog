# The DMA Rewrite That Made Nothing Faster

*Notes from the UBC Formula Electric battery management system, spring 2026.*

I spent a few weeks moving our battery monitor's SPI driver from interrupt-driven to DMA on UBC Formula Electric's BMS. Along the way I fixed a protocol bug that had nothing to do with DMA, built the state machine properly, instrumented both versions with SystemView, and measured them against each other. The DMA path was not faster.

I could just write about the state machine and leave it there, but the measurement is the actual interesting part. So here's the whole thing, including the week I spent chasing the wrong bug.

## What the BMS is doing

Our accumulator is monitored by a chain of **ADBMS6830B** cell monitors talking to an **STM32H7** over isoSPI. The highest-frequency job in the system is reading cell stats, and it's not one transfer, it's a sequence:

```
CLRCELL  ->  ADCV  ->  PLCADC  ->  RDCVA .. RDCVE
 clear      start      poll        read 5 register groups
```

In the original implementation, each of those eight was an interrupt-driven transfer: fire, block, wake on the completion interrupt, do some bookkeeping, fire the next one, block again. Eight round trips through the scheduler for one logical operation. DMA is the obvious fix: let the peripheral move the bytes, chain the whole sequence inside the ISR, and have the task block exactly once at the start and wake exactly once at the end.

That was the plan, and it was a good one. Getting there took a week longer than it should have.

### A note on SPI CRC versus ADBMS PEC

The STM32H7 SPI peripheral has configurable hardware CRC support, but we handled the ADBMS PEC in software. PEC15 and PEC10 are part of the ADBMS packet format rather than a generic SPI CRC appended to the entire transfer. Software therefore generated the command PEC and validated the response PEC after the DMA transfer completed.

## How DMA interacts with SPI

The STM32H7 SPI peripheral is responsible for generating SCK, shifting data on MOSI, sampling MISO, and maintaining the transmit/receive FIFOs. DMA sits between memory and the SPI data registers. For transmission, DMA feeds the SPI TX register from a memory buffer. For reception, DMA copies bytes from the SPI RX register into a memory buffer. Because SPI is full-duplex, both transfers occur simultaneously.

A read transaction therefore includes the command and PEC bytes in the TX buffer, followed by dummy bytes such as 0xFF to generate the clock cycles needed for the ADBMS to return data. The RX buffer contains the bytes sampled during the entire transaction, including response data and any offset caused by the command phase.

The DMA completion interrupt advances the acquisition state machine from CLEARING to STARTING, POLLING, READING, and finally IDLE. DMA does not understand ADBMS commands or PEC values; it only moves bytes. The state machine interprets the completed RX buffer and validates the protocol.

## Troubleshooting

The first DMA build produced garbage. I'd write the configuration registers, read them back, and get values that had nothing to do with what I'd just written.

Cache coherency is the classic DMA bug on an H7: DMA writes straight to SRAM, the CPU reads a stale cache line, and you get exactly this symptom. I added cache maintenance and the behavior changed, which I took as confirmation. It wasn't. Our project is supposed to run with D-Cache disabled entirely, so if the cache is off, clearing it can't be what fixed anything. I actually wrote that contradiction down as a question in my notes at the time, and kept going anyway. Noted it, ignored it.

The real answer was in the datasheet, in the section on command framing, which I'd read and not actually absorbed. To cut down on transfer count I'd batched `WRCFGA` and `WRCFGB` into one continuous SPI transaction. The ADBMS does not allow that.

I didn't have a logic analyzer on the bus, so the diagnosis came from the datasheet plus register readback over the debugger, not a captured waveform. The traces below are SystemView task timelines from the same debugging sessions. They show what the CPU was doing while I chased this, not the SPI line itself, so treat them as debugging context, not proof of the framing bug.

<p align="center">
  <img src="docs/img/cs-framing-01.png" width="500" alt="SystemView task timeline captured on March 7, while WRCFGA and WRCFGB were still batched into one transfer">
  <br><em>Task-level SystemView trace from the session where the batched-command bug was still present.</em>
</p>

<p align="center">
  <img src="docs/img/cs-framing-02.png" width="500" alt="Zoomed SystemView context statistics table from the same March 7 debugging session">
  <br><em>A closer look at the per-task run-time statistics from that same capture.</em>
</p>

<p align="center">
  <img src="docs/img/cs-framing-03.png" width="500" alt="SystemView timeline and task list from March 7, showing the task layout used throughout the investigation">
  <br><em>The full timeline and task list from the same session.</em>
</p>

Every ADBMS command needs its own CS frame. CS goes high at the end of a command, stays high for at least 2 microseconds, then goes low again for the next one. Chip select on this part is not just bus arbitration. It is part of the command protocol, and the gap is how the device knows one command ended and another began.

Splitting each command into its own CS-framed DMA transfer fixed it, completely and permanently. Confirmed by clean register readback afterward.

<p align="center">
  <img src="docs/img/dma-corrected-01.png" width="500" alt="SystemView timeline from March 14, after splitting each command into its own CS-framed transfer, with Cortex-M exception tracking now enabled">
  <br><em>Same tracing setup after the fix. ISRs (100, 44, 29, 28, 27) now show up individually since exception tracking was on for this run.</em>
</p>

<p align="center">
  <img src="docs/img/dma-corrected-02.png" width="700" alt="A second SystemView capture from the same March 14 session on the corrected DMA path">
  <br><em>Another snapshot from the same corrected run.</em>
</p>

The general lesson, and the reason this section exists at all: when a peripheral works fine under polled or interrupt-driven access and breaks under DMA, the DMA engine usually isn't the problem. The slow path was giving you timing the protocol needed, and DMA took it away. Check what the slow path was doing for you before you go blaming the hardware.

## Building it properly

Once the framing was right, the rewrite came together fast. The core is a phase-based state machine that advances entirely from the SPI transfer-complete ISR:

```
CLEARING -> STARTING -> POLLING -> READING -> IDLE
```

The phase-based state machine is shared by the interrupt-driven and DMA implementations. Each phase prepares an ADBMS command and its full-duplex TX/RX buffers. In the interrupt-driven implementation, SPI/FIFO events service the transfer directly. In the DMA implementation, TX and RX DMA channels move the buffers through the STM32H7 SPI peripheral, while the completion path advances the same state machine. DMA changes how bytes are moved; it does not change the ADBMS sequencing, CS framing, or PEC validation.

For each phase, the system asserts CS, starts the SPI transfer, waits for the actual SPI end-of-transfer condition, deasserts CS, and advances to the next phase. Only after the final register-group read does the ISR call vTaskNotifyGiveFromISR() to wake the acquisition task.

## Analysis

This is where it stops going well.

I instrumented both implementations with Segger SystemView, enabled Cortex-M exception tracking so ISR entry/exit actually show up on the timeline instead of being invisible, and captured identical command sequences on both branches.

### Reading the SystemView trace

The ISR labels in these traces are exception numbers reported by the Cortex-M instrumentation, rather than arbitrary application identifiers. For the STM32H743/H747 interrupt vector layout, the relevant values are:

- ISR 27: DMA1 Stream 0
- ISR 28: DMA1 Stream 1
- ISR 29: DMA1 Stream 2
- ISR 44: TIM2
- ISR 100: SPI4

These values are `IRQn + 16`, because Cortex-M exception numbers include the 16 core exceptions before the external interrupt table.

<p align="center">
  <img src="docs/img/systemview-interrupt-01.png" width="700" alt="SystemView timeline of the interrupt-driven SPI path">
  <br><em>The interrupt-driven path with ISR logging enabled.</em>
</p>

<p align="center">
  <img src="docs/img/systemview-interrupt-02.png" width="700" alt="A second SystemView events list and timeline from the interrupt-driven run">
  <br><em>Another snapshot from the same run.</em>
</p>

<p align="center">
  <img src="docs/img/systemview-interrupt-03.png" width="700" alt="SystemView timeline with a tooltip showing Task1Hz run-time stats for the interrupt-driven path">
  <br><em>Hovering the Task1Hz lane pulls up its run-time stats: min, max, last. Just that one task's numbers, not a phase-by-phase breakdown.</em>
</p>

<p align="center">
  <img src="docs/img/systemview-interrupt-04.png" width="606" alt="Task and ISR context switching during the interrupt-driven acquisition">
  <br><em>Context switching between the acquisition task and the SPI ISR.</em>
</p>

The DMA path did not measurably reduce CPU time. The two timelines look basically the same.

## Why nothing happened

I looked at the traces for a while and landed on two explanations that both fit.

**There was nothing else for the CPU to do.** DMA doesn't make a transfer finish sooner, it frees the CPU while the transfer is in flight. If no other task is ready to run in that window, those freed cycles go straight to the idle task and the wall-clock timeline looks identical. The benefit is real, it's just invisible in a trace of a system with nothing better to do. It should show up the moment there's competing work at the same priority band, which on a real car there is.

**The test commands were too short to expose the overhead I was trying to remove.** The SPI peripheral doesn't necessarily interrupt once per byte. With a FIFO it interrupts on half-full or full thresholds. A short configuration command can fit entirely inside the FIFO, so the interrupt-driven path ends up taking roughly the same number of ISR entries as the DMA path and the two traces converge by construction. I wasn't measuring the difference. I was measuring a case where there's no difference to measure.

The second one is a benchmark design failure, and it's entirely on me. A configuration read/write is the easiest thing to instrument, so that's what I instrumented, and it happens to be the exact workload where the optimization can't show up. A full voltage acquisition across five register groups moves enough bytes to cross those FIFO thresholds repeatedly. That's the test I should have run first.

Honest summary: the code is better, the architecture is more extensible, the CS framing bug is genuinely fixed, and I don't have data yet showing the performance benefit I set out to prove. Three wins and one open question.

## What's next

- Re-run the comparison against a full voltage acquisition instead of a configuration cycle
- Re-run under scheduler load, with competing tasks, so the freed CPU time has somewhere to go
- Work out whether `CLRCELL` is still necessary. It zeroes the voltage registers so that reading an unfinished conversion returns zeros instead of stale data, but the DMA polling phase already establishes that the conversion completed, which may make the clear redundant
- Extend the job abstraction to temperature acquisition
- Reconcile the DMA tick with the reworked job scheduler so the voltage task actually runs the DMA path

One thing is still unresolved: why cache maintenance changed behavior at all, when D-Cache is supposed to be disabled in our config. Either it's not actually disabled where I think it is, or the effect I attributed to cache clearing was really just a timing side effect of running extra instructions, which lines up suspiciously well with the CS framing root cause.
