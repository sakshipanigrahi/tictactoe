# Lab-6 Tic-Tac-Toe over UART (TivaWare / no-bonus version)

Two TM4C123 LaunchPads, each on an EduARM4 board, play Tic-Tac-Toe against each other. Moves are entered on the 4x4 keypad, exchanged between the boards over UART, and the game status is printed on a PC terminal.

This version uses **TivaWare (driverlib)** for the UART. There is no `UART_Driver.h` / `UART_Driver.c` in the project and no custom UART function names; `main.c` calls TivaWare functions directly (`UARTConfigSetExpClk`, `UARTCharPutNonBlocking`, `UARTCharGetNonBlocking`, `UARTIntEnable`, ...). Everything that is not UART (keypad, 7-segment, LEDs, button, SysTick) is written directly against the TM4C123 registers.

The same program is flashed onto **both** boards.

## Project files

| File | Purpose |
|---|---|
| `main.c` | Game logic, keypad, 7-segment, LEDs, SW1, SysTick, and the TivaWare UART setup and interrupt handlers |
| `startup.c` | Startup file with the vector table. `SysTick_Handler`, `GPIOPortC_Handler`, `GPIOPortF_Handler`, `UART0_Handler` and `UART5_Handler` are wired in; every other vector points to `IntDefaultHandler` |
| TivaWare `driverlib` files | `sysctl.c`, `gpio.c`, `uart.c`, `interrupt.c`, `cpu.c` (linked from the TivaWare install) |

`UART_Driver.h` and `UART_Driver.c` must **not** be in this project. If they are, `UART0_Handler` and `UART5_Handler` are defined twice and the link fails.

## Code Composer Studio setup

1. Add the five driverlib `.c` files above to the project (Add Files, then Link to files).
2. Add the TivaWare root folder (for example `C:/ti/TivaWare_C_Series-2.2.0.295`) to the compiler include paths, so that `inc/hw_memmap.h` and `driverlib/uart.h` resolve.
3. Under Build > GNU Compiler > Symbols, add:
   - `PART_TM4C123GH6PM`
   - `TARGET_IS_TM4C123_RA1`
   - `gcc` (needed so that `cpu.c` compiles its GCC branch; without it `cpu.c` builds to nothing and the linker reports undefined references to `CPUcpsid`, `CPUcpsie`, `CPUwfi`, `CPUbasepriGet`, `CPUbasepriSet` and `SysCtlDelay`)
4. Clean the project, then build.

## Hardware connections

| Function | Pins |
|---|---|
| Console UART (UART0, PC terminal over the LaunchPad USB) | PA0 (RX), PA1 (TX) |
| Board-to-board link (UART5) | PE4 (RX), PE5 (TX) |
| Keypad rows | PE0-PE3 |
| Keypad columns | PC4-PC7 |
| 7-segment segments a-g, dp | PB0-PB7 |
| 7-segment digit select | PA4 (tens), PA5 (ones) |
| RGB LED | PF1 (red), PF3 (green) |
| Start button SW1 | PF4 |

Between the two boards connect **GND to GND**, **PE4 to PE5** and **PE5 to PE4** (each board's TX goes to the other board's RX).

## UART settings

| Port | Baud | Format |
|---|---|---|
| Console (UART0) | 115200 | 8 data bits, no parity, 1 stop bit |
| Board link (UART5) | 9600 | 8 data bits, no parity, 1 stop bit |

The system clock is left at the reset default of 16 MHz (internal oscillator, no PLL). Baud rate and SysTick timing are computed from that.

UART receive is interrupt driven: `UART0_Handler` and `UART5_Handler` copy incoming bytes into a 32-byte ring buffer per port, and the main loop reads from those buffers. Transmit uses `UARTCharPutNonBlocking` and retries a byte until the FIFO accepts it.

## How to run

1. Flash the same build onto both boards and wire them as above.
2. Open a serial terminal (115200 baud) on each board's COM port.
3. On reset each board prints the rules and waits.
4. Press **SW1** on one board. That board becomes **Player 1 (X)** and sends `'S'` to the other, which becomes **Player 2 (O)**.
5. Player 1 moves first.

## How to play

Cells are numbered like the first 3 rows and 3 columns of the keypad:

```
0 1 2
3 4 5
6 7 8
```

- Cell number = keypad row x 3 + keypad column. The keypad's 4th row and 4th column are outside the grid.
- A blinking green LED (PF3) means it is your move. A solid red LED (PF1) means wait for the other player.
- Each move has **15 seconds**. The 7-segment display counts the remaining seconds down; a leading zero is not shown.
- If a player runs out of time, the console prints `Out of time!` and the other player is declared the winner.
- After a game ends (win, draw or timeout) the LEDs and display are turned off, and SW1 starts a new game.
- Keypad presses are debounced for 250 ms and SW1 presses for 300 ms. A keypad press made while it is not your turn is discarded when your turn begins.

## Console messages

| Message | When |
|---|---|
| `Invalid move` | The chosen cell is already taken, or the key is outside the 3x3 grid |
| `Game won by Player 1` / `Game won by Player 2` | Three in a row (X is Player 1, O is Player 2), or after a timeout |
| `Game Draw` | All 9 cells are filled with no winner |
| `Out of time!` | A player's 15 seconds ran out |
| `Go ahead, it's your turn!` / `Hang tight, opponent is thinking...` | Start of your turn / opponent's turn |
| `You -> cell N` / `Opponent -> cell N` | A move was made; the board is printed after it |

Typing `help` and pressing Enter in the terminal prints the rules again (any mix of upper and lower case). The rules are also printed once at start-up.

## Board-link protocol (UART5)

| Bytes | Meaning |
|---|---|
| `'S'` | The sender pressed SW1 and is Player 1. The receiver becomes Player 2 |
| `'M'` then one byte `cell` (0-8) | The sender placed their symbol in `cell` |

There is no separate win, draw or timeout message. Both boards keep the same board state and run the same per-move countdown, so each reaches the same result on its own. A received `'M'` is ignored if it arrives when it is not the opponent's turn or if the cell is already taken.
