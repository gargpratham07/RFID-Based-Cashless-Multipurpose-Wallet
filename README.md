# RFID Cashless Wallet

## Project Overview

A contactless campus wallet built on the STM32F303RE microcontroller. A student taps a registered RFID card on an MFRC522 reader, then uses the same card to move through an on-screen menu (tap to go to the next option, hold the card to select). The wallet shows the balance, adds or deducts money, and shows recent transactions on a 16x2 LCD, with buzzer feedback and a low-balance alert. Everything runs on the microcontroller; no internet or backend is needed.

Main parts:

- **MFRC522 RFID reader** over SPI, with UID read and BCC checksum check
- **16x2 character LCD** driven in 4-bit mode by bit-banged GPIO
- **Card state machine**: debounced card detection, tap-to-cycle menu, hold-to-select, 30 s inactivity timeout, EXIT button
- **Wallet module**: up to 4 card slots, balance in paise, per-card transaction history in a circular buffer

## Hardware Requirements

- NUCLEO-F303RE board (STM32F303RETx)
- MFRC522 RFID reader module (3.3 V)
- 16x2 character LCD (HD44780 compatible), used in 4-bit mode
- Active buzzer (active high)
- EXIT push button (active low). The Nucleo's blue user button on PC13 works as EXIT with no extra wiring.
- USB cable to the ST-LINK for flashing and the serial debug console (115200 baud)

## Pin Connections

### LCD (4-bit mode)

| LCD pin | Purpose | STM32F303RE |
|---------|---------|-------------|
| D4 | Data bit 4 | PB0 |
| D5 | Data bit 5 | PB1 |
| D6 | Data bit 6 | PB2 |
| D7 | Data bit 7 | PB3 |
| RS | Register select | PB4 |
| EN | Enable strobe | PB5 |

### MFRC522 RFID Reader (SPI1)

| Pin | Purpose | STM32F303RE |
|-----|---------|-------------|
| SDA/CS | Chip select (active low) | PA4 |
| SCK | Serial clock | PA5 (SPI1, AF5) |
| MISO | Master in, slave out | PA6 (SPI1, AF5) |
| MOSI | Master out, slave in | PA7 (SPI1, AF5) |
| RST | Reset (held high for normal operation) | PB12 |

### Buzzer and Buttons

| Signal | STM32F303RE | Notes |
|--------|-------------|-------|
| Buzzer | PA8 | Active high |
| EXIT button | PC13 | Internal pull-up, active low (Nucleo blue button) |
| CONFIRM button | PC14 | Configured as input, not used by the current firmware |

### Debug Console (USART2)

| Signal | STM32F303RE | Notes |
|--------|-------------|-------|
| TX | PA2 | Routed to the ST-LINK virtual COM port on the Nucleo |
| RX | PA3 | Not used |

## Circuit Diagram

![RFID Wallet Circuit Diagram](Circuit%20diagram.png)

*Interconnection of the STM32F303RE with the MFRC522 reader, 16x2 LCD, buzzer and buttons. Where the diagram and the tables above differ (button labels), the tables match the firmware.*

## Configuration

Application settings live in `Core/Inc/config.h`:

- **Timing** (card debounce count, hold-to-select time, menu timeout, buzzer and message display times)
- **Wallet limits** (max cards, max transactions per card, number of preloaded cards)
- **Money values** in paise (top-up amount, debit amount, low-balance threshold)
- **GPIO pin assignments** for the LCD, RFID reader, buzzer and buttons

The registered cards (UID, name and starting balance) are in the `users` table at the top of `Core/Src/state_machine.c`.

## Firmware Structure

| File | Role |
|------|------|
| `Core/Src/main.c` | Clock (72 MHz from HSI + PLL), GPIO, SPI1 and USART2 setup; starts the state machine |
| `Core/Src/state_machine.c` | Application logic: card detection, menu, hold-to-select, timeouts, LCD/buzzer feedback |
| `Core/Src/wallet.c` | Card slots, balances and transaction history |
| `Core/Src/mfrc522.c` | MFRC522 driver: register access over SPI, request, anticollision, select, halt |
| `Core/Src/lcd.c` | HD44780 LCD driver in 4-bit mode |
| `Core/Src/eeprom_emul.c` | Flash-backed storage helper (included, not yet used by the application) |
| `Core/Inc/config.h` | All tunable constants and pin assignments |
| `Drivers/` | ST HAL and CMSIS (vendor code, not modified) |

## Application Flow

1. **Start-up**: clock, GPIO, SPI1 and USART2 are initialised, the roster is loaded into the wallet, the MFRC522 is reset and configured, and the LCD shows "Ready - Tap Card".
2. **Card detection**: the reader is polled every 80 ms. A card must be seen on 2 polls in a row (`CARD_STABLE_COUNT`) to count as present.
3. **Unknown card**: shows "UNREGISTERED", double beep, waits for the card to be removed.
4. **Menu**: the first tap shows the student's name and the first option. Each new tap moves to the next option: Show Balance, Add 10.00, Sub 1.00, Show History, Exit.
5. **Hold-to-select**: keeping the card on the reader for 800 ms (`MENU_HOLD_TIME_MS`) runs the selected option.
6. **Low balance**: if the balance is below 1.00 (`LOW_BALANCE_THRESHOLD_PAISE`), the LCD shows "LOW BALANCE" and the buzzer gives a long alert.
7. **Leaving the menu**: choose Exit, press the EXIT button, or wait 30 s with no activity.

## How to Build and Flash

1. Open STM32CubeIDE and choose **File > Import > Existing Projects into Workspace**, then select this folder.
2. Build the project (hammer icon). The `Debug/` build folder is created locally and is not tracked in git.
3. Connect the Nucleo board over USB and click **Run** (or **Debug**) to flash it.
4. Open a serial terminal on the ST-LINK virtual COM port at 115200 baud to see the debug log.

## Known Limitations

- The card roster and starting balances are hard-coded; new cards cannot be registered at runtime.
- Balances are kept in RAM only and reset on power-up. The flash storage helper (`eeprom_emul.c`) is included but not yet connected to the wallet.
- Cards are identified by UID only, with no MIFARE sector authentication, so a cloned card would be accepted. This is fine for a demo but not for real payments.
- The main loop uses blocking delays (`HAL_Delay`), so the reader is not polled while messages are on screen.

## Author

Group project — SNU ECE, November 2025
