Documentation/


# Fruit Loop – A Tiny Snake Game on XIAO ESP32-C3


Fruit Loop is a Snake-style arcade game that runs on a Seeed XIAO ESP32-C3 using a 128×64 SSD1306 OLED display.  
You control the snake with a rotary encoder, eat apples (`@`), and race against the clock to finish the level without crashing into yourself.

The project also uses a NeoPixel for visual feedback, a piezo buzzer for sound effects, and an optional ADXL345 accelerometer for “shake-to-restart” control. The code is designed to be memory-safe on the XIAO ESP32-C3 and avoid NeoPixel-related `MemoryError` crashes.

---

## Features

- 🐍 **Snake-style gameplay** on a text grid
- 🎛 **Rotary encoder controls**
  - Rotate: choose difficulty in the menu
  - Button press: rotate snake direction during the game
- 🍏 **Apples and scoring**
  - Apples are shown as `@`
  - Each apple increases your score by 10
  - Speed increases as you eat more apples
- ⏱ **Timer + High score**
  - Tracks how long it takes you to eat 10 apples
  - Persists best time in `highscore.txt`
- 💡 **NeoPixel state colors**
  - Menu: blue
  - Playing: green
  - Game over: red
  - Win: orange
- 🔔 **Piezo buzzer sounds**
  - Power-on jingle
  - Game start beep
  - “Eat apple” blip
  - Game over tones
  - Winning melody
- 📳 **Optional shake control (ADXL345)**
  - Shake 3× quickly to:
    - Restart from the win/game over screen
    - Return to menu when playing (depending on state)
- 🧠 **Anti-crash / memory-safe design**
  - Aggressive `gc.collect()` usage
  - `safe_set_pixel()` wrapper to disable NeoPixel if ESP-IDF runs out of memory
  - Minimal string allocations in the update loop

---

## Hardware Overview

This project targets a **Seeed XIAO ESP32-C3** running CircuitPython.

### Required Components

- Seeed XIAO ESP32-C3
- 128×64 SSD1306 OLED display (I²C, 0x3C)
- Rotary encoder (with push button)
- 1× NeoPixel (on-board or external)
- Piezo buzzer (driven via PWM)
- (Optional) ADXL345 accelerometer (I²C, 0x53)

### Pin Connections (as used in the code)

> Adjust if your wiring is different, but then update the corresponding pin constants.

- **I²C bus**  
  - SCL / SDA: board default I²C (`board.I2C()`), OLED at `0x3C`, ADXL345 at `0x53`
- **SSD1306 OLED**  
  - I²C, address: `0x3C`
- **ADXL345 accelerometer (optional)**  
  - I²C, address: `0x53`
- **Rotary Encoder**
  - DT: `board.D8`
  - CLK: `board.D9`
  - SW (button): `board.D10`
- **NeoPixel**
  - Tries `board.NEOPIXEL` first, otherwise falls back to `board.D7`
- **Piezo Buzzer**
  - `BUZZER_PIN = board.D0` (PWM capable pin)

If no ADXL345 is connected, the code still runs; shake-to-restart is automatically disabled.

---

## How the Game Works

### Game States

The game is implemented as a simple state machine:

1. **Splash Screen (`STATE_SPLASH`)**
   - Displays a short intro: “FRUIT” → “LOOP” → “GO!”
   - After a brief sequence, automatically transitions to the menu.

2. **Main Menu (`STATE_MENU`)**
   - Shows the title “FRUIT LOOP”.
   - Shows the current **difficulty**:
     - `SLOW` (0.35 s per step)
     - `STANDARD` (0.20 s per step)
     - `FAST` (0.10 s per step)
   - A scrolling hint text explains controls:
     - “ROTATE to choose | BUTTON to select”
   - NeoPixel = **blue**.

3. **Playing (`STATE_PLAYING`)**
   - A text-based grid is drawn using ASCII characters:
     - `+`, `|`, `=` for borders
     - `@` for apples
     - `#` for snake body
     - `> < ^ v` for the snake head, depending on direction
   - Top line shows game info:
     - `LVL:<level> T:<time> S:<score>`
   - NeoPixel = **green**.

4. **Game Over (`STATE_GAME_OVER`)**
   - Shows “GAME OVER :(”
   - Shows your time: `Time: X.Xs`
   - Shows a scrolling hint: “BUTTON to Menu | SHAKE to Retry”
   - NeoPixel = **red**.

5. **Win (`STATE_WIN`)**
   - Shows “YOU WIN!”
   - Shows your time: `Time: X.Xs`
   - Shows your best time:
     - `Best: Y.Ys` (read from / saved to `highscore.txt`)
   - Shows a scrolling hint: “BUTTON to Menu | SHAKE to Retry”
   - NeoPixel = **orange**.

### Goal

- Start a new game from the menu.
- Eat **10 apples** (`@`) as fast as you can.
- Avoid hitting your own body.
- Your completion time is compared against the saved best time.

### Movement & World Rules

- The snake is stored as a list of `(x, y)` coordinates.
- On each step:
  - A new head is computed based on the current direction.
  - The grid wraps around, so leaving one side enters from the opposite side:
    - Leaving left comes in from the right (inside the border).
    - Leaving top comes in from the bottom (inside the border).
- If the new head overlaps any part of the snake body:
  - Game Over.
- If the new head lands on the apple:
  - The snake grows by 1 segment.
  - Score increases by 10.
  - Level increases and speed is slightly increased (step interval shrinks).
  - A new apple is placed in an empty cell.
- If 10 apples have been eaten:
  - You win the game.

---

## Controls

### In the Menu

- **Rotate encoder**: change difficulty (`SLOW`, `STANDARD`, `FAST`)
- **Press encoder button**:
  - Confirms difficulty
  - Plays a “game start” sound
  - Starts a new game

### During Gameplay

- **Press encoder button**:
  - Rotates the snake’s direction 90° clockwise:
    - Right turn mapping: `(dx, dy) → (dy, -dx)`
- **Shake** (if ADXL345 present and enabled):
  - If implemented in this state, shake will return you to the menu from playing or restart from game over/win (see code path).

### On Game Over / Win Screens

- **Press encoder button**:
  - Returns to the main menu
- **Shake device** (with ADXL345):
  - Immediately starts a new game

---

## Sound Effects

The buzzer is driven via PWM using `pwmio.PWMOut`:

- **Power-on** (`sound_power_on`)
  - Short rising two-tone jingle
- **Game start** (`sound_game_start`)
  - Two quick ascending tones
- **Eat apple** (`sound_eat`)
  - Short high-pitched blip
- **Game over** (`sound_game_over`)
  - Two descending tones
- **Win** (`sound_win`)
  - Short ascending “victory” sequence

You can tune the frequencies and durations in the sound helper functions.

---

## Memory-Safe Design

Running CircuitPython on the XIAO ESP32-C3 means **limited RAM** and tight low-level buffers for peripherals like NeoPixel. This project includes several measures to avoid crashes:

1. **Frequent `gc.collect()` calls**
   - At key points:
     - When switching states
     - Every main `update()` loop
     - After building UI
   - Helps reclaim Python heap before it fragments.

2. **Limited string allocations**
   - Display text (like the score line) is only updated:
     - Once per second, **or**
     - When the score changes
   - Prevents excessive temporary string creation.

3. **`safe_set_pixel()` wrapper for NeoPixel**
   - All NeoPixel writes go through `safe_set_pixel(color)`.
   - If an exception occurs (e.g., `espidf.MemoryError` in `_transmit`), pixels are disabled:
     - `_pixels_ok` flag is set to `False`.
     - Subsequent calls silently do nothing.
   - The game continues running even if NeoPixel fails.

4. **Minimal dynamic objects in the game loop**
   - The game logic mostly reuses existing labels and data structures.
   - Only the `grid` per frame and some small strings are rebuilt.

---

## Files

- `fruit_loop_game_V19_ANTI_CRASH_BUZZER_SAFE_PIXELS.py`  
  Main game script; copy this to `code.py` on your CircuitPython drive.

- `highscore.txt`  
  Created automatically after the first successful run. Stores the best completion time as a floating-point number.

---

## How to Run

1. Install **CircuitPython** for the Seeed XIAO ESP32-C3.
2. Install the required **Adafruit libraries** in `lib/`:
   - `adafruit_displayio_ssd1306.mpy`
   - `adafruit_display_text/`
   - `adafruit_adxl34x.mpy` (if using the accelerometer)
   - `i2cdisplaybus.mpy` (or equivalent module)
   - `neopixel.mpy`
   - Any support module you use for `RotaryEncoder` (your `rotary_encoder.py` file)
3. Wire the components as described in the Hardware section.
4. Copy:
   - `fruit_loop_game_V19_ANTI_CRASH_BUZZER_SAFE_PIXELS.py` → `code.py`
   - `rotary_encoder.py` → to the root of the CIRCUITPY drive (or into `lib` if your import path expects that).
5. Reset the board or power-cycle it.
6. You should:
   - Hear the **power-on sound**
   - See the **splash screen**, then the **menu**

---

## Possible Improvements

- Add more levels (e.g., longer than 10 apples).
- Show remaining apples to win.
- Add pausing or multiple lives.
- Add obstacles or walls inside the arena.
- Make difficulty also affect how quickly the speed ramps up.
- Persist per-difficulty best times.

---

## License

This is a hobby project for educational and personal use.  
Feel free to modify, extend, and adapt it for your own hardware setups.
