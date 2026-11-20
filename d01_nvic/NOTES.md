# Day 1 — Terminal debugging + exception model / NVIC

**Date:** Fri 20 Nov 2026 (steps 1–2) · Sat 21 Nov (steps 3–8)
**Board:** Nucleo-F446ZE (Cortex-M4F)

## What I did

- Generated a CMake project for NUCLEO-F446ZE with STM32CubeMX (board defaults, no TIM2/EXTI NVIC enabled).
- Blinked LD1 (PB0) by writing GPIOB->BSRR directly from the USER CODE block.
- Flashed with `STM32_Programmer_CLI -c port=SWD -w build/Debug/d01_nvic.elf -v -rst`.
- Set up the terminal debug loop: OpenOCD (`board/st_nucleo_f4.cfg`) + `gdb-multiarch`.
- Wrote a project `.gdbinit` (connect → reset halt → load → break main → continue) and trusted
  `~/mcu_gaps` with `add-auto-load-safe-path` in `~/.gdbinit`. One command now lands at `main`.

## What I saw

**Clock tree (CubeMX defaults)**

| Clock | Value |
|---|---|
| HSE (bypass, from ST-LINK MCO) | 8 MHz |
| PLL | /M 4, ×N 168, /P 2 |
| SYSCLK = HCLK | 168 MHz |
| APB1 peripheral (PCLK1) | 42 MHz |
| APB1 timer clock (TIM2) | 84 MHz — ×2 because APB1 prescaler ≠ 1 |
| APB2 peripheral / timer | 84 / 168 MHz |

**State right after `monitor reset halt`**

- `MSP = 0x20020000` → vector table entry 0 = initial stack pointer = top of 128 KB SRAM.
- `PC = 0x08003214` → vector table entry 1 = `Reset_Handler`.
- `xPSR = 0x01000000` → only the Thumb bit (24) set; IPSR = 0 (Thread mode).
- The hardware loads both from the vector table before any code runs.

**Halted at random while running:** `xPSR = 0x21000000`, IPSR = 0 → Thread mode, inside `HAL_GetTick()` (polled by `HAL_Delay`).

**`load` output:** `.data` has an LMA in flash (0x080032a8) — it gets copied to RAM by startup code before `main` (Day 3).

**Debug hardware:** OpenOCD reports 6 hardware breakpoints, 4 watchpoints on this core.

## Surprise

`GPIOx->BSRR |= ...` (as in some tutorials) only works by accident. BSRR is write-only and reads as 0
(RM0390, GPIOx_BSRR), so `|=` turns an atomic single write into a non-atomic read-modify-write.
Use `=`. When a tutorial and the reference manual disagree, the RM wins.

## Gotchas hit

- OpenOCD board file is `board/st_nucleo_f4.cfg` (underscore before f4).
- Ubuntu's apt toolchain has no `arm-none-eabi-gdb` → use `gdb-multiarch`.
- `.gdbinit` is a file, not a GDB command; GDB ignores a local one unless its path is in `auto-load safe-path`.
- CubeMX board preset configures PC13 (USER_Btn) as EXTI rising edge — NVIC not enabled. Relevant for step 5.

---

## Steps 3–8 (Sat)

### Preemption captures

- [ ] A — EXTI priority higher than TIM2 → `d01_a_exti_high.png`
- [ ] B — equal priorities → `d01_b_equal.png`
- [ ] C — TIM2 higher → `d01_c_tim2_high.png`

### Inside the EXTI ISR (GDB)

| Register | Value | Meaning |
|---|---|---|
| IPSR | | |
| ICSR.VECTACTIVE | | |
| ICSR.VECTPENDING | | |
| NVIC active bits | | |
| NVIC pending bits | | |
| LR | | |

### EXC_RETURN decode (bit by bit)

LR = `0x________`

| Bit(s) | Value | Meaning |
|---|---|---|
| | | |

### Sub-priority test

-

## Interview questions

**1. What does the hardware do between the IRQ asserting and the first ISR instruction?**

**2. Tail-chaining vs late-arrival: what is saved in each case?**

**3. Why does a lower number mean higher priority? How many priority bits does the F446 implement?**

**4. What is BASEPRI for, and why is it better than disabling all interrupts?**
