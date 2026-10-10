# Lab-6 Tic-Tac-Toe over UART — Custom UART Driver (Bonus)

Two-player Tic-Tac-Toe on two **TM4C123 (Tiva C) LaunchPads** with **EduARM4** boards, written in C directly against the TM4C123GH6PM registers. **No TivaWare / driverlib is used anywhere.**

This README focuses on the **bonus part of the assignment: developing our own UART driver instead of using the TivaWare library.** The driver is implemented in `UART_Driver.c` against the interface declared in `UART_Driver.h`.

---

## Project files 

| File | What it contains |
|------|------------------|
| `main.c` | Game rules, keypad, 7-segment display, RGB LED, SysTick timer, SW1 button, board-link protocol, `main()` |
| `UART_Driver.h` | UART interface, as given with the assignment |
| `UART_Driver.c` | Our own register-level UART driver (the bonus part) |
| Startup file (CCS/GNU) | Vector table and `ResetISR`. Lists the `UART0_Handler` and `UART5_Handler` used by the driver |

One program is flashed onto **both** boards. The board whose SW1 is pressed first becomes Player 1 (X) and sends `'S'`; the other board receives it and becomes Player 2 (O).

---

## Game rules

### The board

The game uses the first 3 rows x 3 columns of the 4x4 keypad. Each key is one cell:

```
0 1 2
3 4 5
6 7 8
```

The 4th row and 4th column of the keypad are not part of the grid. Pressing them counts as an invalid move.

### Starting a game

1. Press **SW1** on either board while the game is idle (at power-up) or after a game has ended.
2. The board whose SW1 was pressed becomes **Player 1 (X)** and plays first.
3. The other board becomes **Player 2 (O)** automatically.
4. SW1 does nothing while a game is in progress.

Because whoever presses SW1 becomes Player 1, the roles can swap between games.

### Taking turns

- Players move alternately. Player 1 uses **X**, Player 2 uses **O**.
- On your turn, press the keypad key of the cell you want.
- Key presses made while it is the opponent's turn are ignored.
- The console prints each move (`You -> cell N` / `Opponent -> cell N`) followed by the board, where `X` and `O` are placed symbols and `.` is an empty cell.

### Turn indicators

| Indicator | Meaning |
|-----------|---------|
| Blinking **green** LED | It is your move |
| Solid **red** LED | Wait for the other player |
| LEDs off, 7-segment blank | The game is over (or no game has started) |
| Console: `Go ahead, it's your turn!` | Your turn started |
| Console: `Hang tight, opponent is thinking...` | Opponent's turn started |

### Move timer

- Each move has **15 seconds**, counted down on the 7-segment display (driven by the SysTick timer).
- The countdown restarts at 15 each time a turn begins.
- Both boards run the same countdown, so both show the opponent's remaining time as well.
- If the player to move reaches 0, the console prints `Out of time!` and the **other player wins**.

### Invalid moves

A move is invalid if the cell is already taken or the key pressed is outside the 3x3 grid. The console prints `Invalid move`. Nothing is placed, it is still your turn, and the countdown keeps running.

### Winning and drawing

- **Win:** three of the same symbol in a row, a column, or a diagonal. The console prints `Game won by Player 1` (X) or `Game won by Player 2` (O).
- **Draw:** all 9 cells are filled with no winner. The console prints `Game Draw`.
- After a win, draw, or timeout the LEDs turn off and the 7-segment display goes blank. Press SW1 on either board to play again.

### Console messages

| Message | When |
|---------|------|
| `Invalid move` | Cell already taken, or key outside the 3x3 grid |
| `Game won by Player 1` / `Game won by Player 2` | Three in a row, or the opponent ran out of time |
| `Game Draw` | Board full, nobody won |
| `Out of time!` | A player's 15 seconds ran out |

Type `help` and press Enter in the PC terminal at any time to print the rules again. The rules are also printed automatically when the console opens.

---

## The UART driver (bonus)

### What the driver offers

```c
int UART_Init(int iPort, int iBaudRate, int iParity_NOEMS);
int UART_Write(int iPort, int iBytes, unsigned char *pcBytes);
int UART_Read(int iPort, int iBytes, unsigned char *pcBytes);
int UART_BytesInRx(int iPort);
```

`iPort` is 0-7 and maps 1:1 onto UART0-UART7.

| Function | What it does | Return value |
|----------|--------------|--------------|
| `UART_Init` | Enables clocks, routes pins, sets baud rate and frame format, enables RX interrupt, turns the UART on | `0` OK, `-1` bad port, `-2` baud <= 0, `-3` bad parity |
| `UART_Write` | Puts bytes into the 16-byte hardware TX FIFO. **Does not wait.** Stops as soon as the FIFO is full | Number of bytes actually accepted |
| `UART_Read` | Takes up to `iBytes` bytes out of the software receive buffer. **Does not wait** | Number of bytes actually read (may be 0) |
| `UART_BytesInRx` | Counts bytes waiting in the software receive buffer | Byte count |

Parity options accepted by `UART_Init`: `UART_PARTITY_NONE`, `UART_PARTITY_ODD`, `UART_PARTITY_EVEN`, `UART_PARTITY_MARK`, `UART_PARTITY_SPACE`. The frame is always 8 data bits, 1 stop bit, with the hardware FIFOs enabled.

### Ports and pins

All eight ports use PCTL alternate-function value 1.

| Port | RX pin | TX pin | Used in this project |
|------|--------|--------|----------------------|
| UART0 | PA0 | PA1 | **Yes** — PC console over the LaunchPad USB virtual COM port |
| UART1 | PB0 | PB1 | No |
| UART2 | PD6 | PD7 | No (PD7 is commit-locked; the driver unlocks it) |
| UART3 | PC6 | PC7 | No |
| UART4 | PC4 | PC5 | No |
| UART5 | PE4 | PE5 | **Yes** — board-to-board link |
| UART6 | PD4 | PD5 | No |
| UART7 | PE0 | PE1 | No |

### How `UART_Init` works, step by step

1. Checks the arguments (`-1` / `-2` on failure) and clears the port's receive buffer.
2. Turns on the clock of the GPIO port (`RCGCGPIO`) and of the UART module (`RCGCUART`), then waits a short delay.
3. If the pin is commit-locked (PD7), unlocks it with `LOCK` / `CR`.
4. Routes the RX and TX pins to the UART: analog off (`AMSEL`), alternate function on (`AFSEL`), digital on (`DEN`), `PCTL` set to function 1. Only the two UART pins of that port are changed.
5. Disables the UART while it is being configured.
6. Calculates the baud-rate divisor (see below) and writes `IBRD` and `FBRD`.
7. Writes `LCRH`: 8 data bits, FIFOs on, plus parity bits as requested.
8. Clears all pending UART interrupts and enables the **RX** and **RX-timeout** interrupts only.
9. Enables the UART, its transmitter and its receiver (`CTL`).
10. Enables the UART's interrupt in the NVIC and marks the port ready.

### Baud-rate calculation

The clock is the reset-default 16 MHz internal oscillator (`SYSCLK_HZ = 16000000`, no PLL).

```
BRD  = SYSCLK / (16 x baud)
IBRD = integer part of BRD
FBRD = fractional part x 64, rounded to nearest
```

The driver does this with integer math: `brd64 = (SYSCLK x 64 + 8 x baud) / (16 x baud)`, then `IBRD = brd64 >> 6` and `FBRD = brd64 & 0x3F`. A 64-bit intermediate is used.

| Baud | BRD | IBRD | FBRD |
|------|-----|------|------|
| 9600 (board link) | 104.1667 | 104 | 11 |
| 115200 (console) | 8.6806 | 8 | 44 |

### Receiving: interrupt + ring buffer

Each port has a 32-slot software ring buffer (`UART_RX_BUF_SIZE = 32`), so it holds up to **31 bytes**.

- When bytes arrive, the UARTn interrupt handler (`UART_ISR_Common`) copies everything from the hardware RX FIFO into the ring buffer (`head` moves forward) and clears the interrupt flags.
- `UART_Read` takes bytes out from the `tail` side.
- If the ring buffer is full, new bytes are **dropped** instead of blocking inside the interrupt.
- Buffer is empty when `head == tail`.

Example (buffer size 32, the other board sends `'M'` then `5`):

| Event | head | tail | `UART_BytesInRx` |
|-------|------|------|------------------|
| Start | 0 | 0 | 0 |
| ISR stores `'M'` | 1 | 0 | 1 |
| ISR stores `5` | 2 | 0 | 2 |
| `UART_Read` takes `'M'` | 2 | 1 | 1 |
| `UART_Read` takes `5` | 2 | 2 | 0 |

### Transmitting: non-blocking

`UART_Write` writes bytes into the hardware FIFO until it is full (`TXFF` flag) and returns how many bytes were accepted. It never waits.

`console_print()` in `main.c` handles this: if fewer bytes were accepted than requested, it sends the rest on the next attempt, with a short spin delay when nothing was accepted.

### Interrupt handlers

The driver defines `UART0_Handler` ... `UART7_Handler`, each one calling `UART_ISR_Common(n)`. The startup file's vector table currently points only `UART0_Handler` and `UART5_Handler` to these. **If you initialise any other UART port, add its handler name to the vector table in the startup file** (it is currently `IntDefaultHandler`, which is an infinite loop).

---

## How `main.c` uses the driver

```c
#define UART_PORT_CONSOLE  0   /* UART0, PA0/PA1 - USB virtual COM */
#define UART_PORT_LINK     5   /* UART5, PE4/PE5 - between boards  */

UART_Init(UART_PORT_CONSOLE, 115200, UART_PARTITY_NONE);
UART_Init(UART_PORT_LINK,    9600,   UART_PARTITY_NONE);
```

### Board-link protocol (UART5, 9600 8N1)

| Bytes sent | Meaning |
|------------|---------|
| `'S'` | Start / restart. Sender becomes Player 1 (X) and moves first. Receiver becomes Player 2 (O) |
| `'M'` then `<cell>` | Sender placed its symbol in cell `<cell>` (0-8) |

There is no separate win / draw / timeout message. Both boards apply the same moves and run the same countdown, so each reaches the same result by itself.

`poll_link()` reads the link port once per main-loop pass and handles the case where the cell byte arrives after the `'M'` byte.

### Console

- The game prints moves, the board and results on UART0.
- `poll_console_help()` reads what is typed in the PC terminal; typing `help` + Enter reprints the rules.

---

## Hardware setup

| Function | Pins |
|----------|------|
| Console UART (UART0) | PA0 (RX) / PA1 (TX) — USB cable |
| Board-link UART (UART5) | PE4 (RX) / PE5 (TX) |
| Keypad rows | PE0-PE3 |
| Keypad columns | PC4-PC7 |
| 7-segment segments a-g, dp | PB0-PB7 |
| 7-segment digit select | PA4 (tens), PA5 (ones) |
| RGB LED | PF1 (red), PF3 (green) |
| Start / restart button | PF4 (SW1) |

**Board-to-board wiring:**

- Board A **PE5 (TX)** to Board B **PE4 (RX)**
- Board B **PE5 (TX)** to Board A **PE4 (RX)**
- **GND to GND**

Keypad cells (first 3 rows x 3 columns):

```
0 1 2
3 4 5
6 7 8
```

---

## Build and run (Code Composer Studio)

1. Create a CCS project for the **TM4C123GH6PM** device and add `main.c`, `UART_Driver.c`, `UART_Driver.h` and the startup file to it.
2. Build the project. It must build with no TivaWare / driverlib in the include or library paths.
3. Connect the first LaunchPad by USB and flash/debug it. Run the program.
4. Repeat for the second LaunchPad. The **same program** goes on both boards.
5. Connect the two boards with the three wires above.
6. Open a serial terminal on each board's virtual COM port at **115200 baud, 8 data bits, no parity, 1 stop bit**.
7. Each terminal shows the rules and `Hit SW1 on either board to kick off a match - X goes first.`
8. Press **SW1** on one board. That board is Player 1 (X) and the other becomes Player 2 (O).
9. Blinking **green** LED = your move. Solid **red** LED = wait. The 7-segment display counts down 15 seconds.
10. Press a keypad key (cells 0-8) to play.
