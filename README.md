# rsvp.nvim

![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/ellisonleao/nvim-plugin-template/lint-test.yml?branch=main&style=for-the-badge)
![Lua](https://img.shields.io/badge/Made%20with%20Lua-blueviolet.svg?style=for-the-badge&logo=lua)

`rsvp.nvim` is a speed-reading plugin for Neovim built around RSVP (Rapid Serial Visual Presentation). It opens a focused fullscreen reader and displays one word at a time from a selected text range so you can read quickly without eye-scanning lines.

https://github.com/user-attachments/assets/01438201-3dd2-47c2-902c-79b34fd31954

## Features

- Read any selected range (or full buffer) one word at a time
- Optimal Recognition Point (ORP)-centered rendering with accented focus character (`RsvpAccent`)
- Fullscreen floating RSVP reader for distraction-free reading
- Play/pause, reset, and step backward/forward through words
- Adjustable WPM with clamped limits (50 to 1000)
- Live progress info (`current/total`, percentage, WPM)
- Optional surrounding-word context (`0..3` words on each side)
- Progress bar and completion elapsed time
- Built-in help popup (`g?`)
- Configurable speed step, progress width, and keymaps
- Optional adaptive timing: complex/code-like words linger, common English words go faster
- Optional punctuation "breathers": a pause (with the punctuation highlighted) at clause/sentence ends
- Optional blank-screen hint between paragraphs
- Optional autostop at paragraph / sentence / clause boundaries

## Adaptive Timing

By default every word is shown for the same `60000 / WPM` interval. The following **opt-in** options make the pace adaptive — when they are all disabled (the defaults), playback is identical to before.

- **`complexity_scaling`** scales how long each word stays on screen. Long, symbol-heavy, mixed-case (`camelCase`), digit-bearing, or `ALL_CAPS` tokens linger so you can process them; common English words (matched against a bundled word list) speed up. The multiplier is clamped between `min_multiplier` and `max_multiplier`.
- **`breather`** adds a pause _after_ a word that ends a clause (`, ; : -`) or a sentence (`. ! ?`), mirroring spoken pauses. While the word is shown, its trailing punctuation is highlighted with `RsvpBreather`.
- **`paragraph_pause_ms`** blanks the reading line for the given number of milliseconds between paragraphs (detected from blank lines in the source) for a visual reset. `0` disables it.
- **`autostop`** auto-pauses playback when it reaches a boundary so you resume deliberately with `<Space>`: `"paragraph"`, `"sentence"`, `"clause"`, or `"off"`.

## Optimal Recognition Point (ORP) Selection Logic

`rsvp.nvim` uses a Spritz-style stepped Optimal Recognition Point (ORP) mapping based on the word's alphanumeric length (surrounding punctuation is ignored):

| Alphanumeric word length | Optimal Recognition Point (ORP) character index (1-based) |
| ------------------------ | --------------------------------------------------------- |
| 1                        | 1                                                         |
| 2-5                      | 2                                                         |
| 6-9                      | 3                                                         |
| 10-13                    | 4                                                         |
| 14+                      | 5                                                         |

## Installation

Using [lazy.nvim](https://github.com/folke/lazy.nvim):

```lua
{
  "kivanceski/rsvp.nvim",
}
```

Usage examples:

- `:Rsvp` to read the whole file
- `:10,40Rsvp` to read lines 10 through 40
- Visual-select lines, then run `:Rsvp`

## Configuration (all defaults shown)

This example sets every option to its current default value. Keep this as a reference and override only the values you want to change.

```lua
{
  "kivanceski/rsvp.nvim",
  opts = {
    auto_run = true, -- Start playback immediately after opening the RSVP window
    initial_wpm = 300, -- Starting words-per-minute speed
    wpm_step_size = 25, -- Amount added/removed when changing WPM
    progress_bar_width = 80, -- Progress bar width in characters
    surrounding_word_count = 1, -- Show up to N words on each side of the active word (0..3, invalid => 1)
    paragraph_pause_ms = 0, -- Blank-screen pause (ms) between paragraphs; 0 disables
    autostop = "off", -- Auto-pause at boundaries: "off" | "paragraph" | "sentence" | "clause"
    complexity_scaling = {
      enabled = false, -- Scale display time by word complexity
      min_multiplier = 0.6, -- Lower clamp (allows common words to speed up)
      max_multiplier = 3.0, -- Upper clamp
      common_word_multiplier = 0.7, -- Multiplier applied to recognized common English words
      word_list = {
        enabled = true, -- Speed up words found in the bundled common-words list
      },
      weights = { -- Additive complexity weights
        length_medium = 0.3, -- 5..8 alphanumeric chars
        length_long = 0.6, -- 9..12 chars
        length_very_long = 1.0, -- 13+ chars
        has_digit = 0.4, -- Contains a digit
        has_symbol = 0.5, -- Contains a symbol/underscore (code-like)
        mixed_case = 0.5, -- Mixed/camelCase
        all_caps = 0.2, -- Multi-char ALL CAPS
      },
    },
    breather = {
      enabled = false, -- Pause after clause/sentence-ending punctuation
      clause_ms = 200, -- Extra pause after , ; : -
      sentence_ms = 400, -- Extra pause after . ! ?
      highlight = true, -- Highlight the trailing punctuation (RsvpBreather)
    },
    keymaps = {
      decrease_wpm = "<", -- Decrease WPM by `wpm_step_size`
      increase_wpm = ">", -- Increase WPM by `wpm_step_size`
      previous_step = "H", -- Move one word backward
      next_step = "L", -- Move one word forward
    },
    colors = {
      main = {
        link = "Keyword", -- Any `nvim_set_hl()` option is supported
      },
      accent = {
        link = "ErrorMsg",
      },
      paused = {
        fg = "#FFFF00",
        bold = true,
      },
      done = {
        fg = "#00FF00",
        bold = true,
      },
      ghost_text = {
        link = "NonText",
      },
      breather = {
        fg = "#FFA500",
        bold = true,
      },
    },
  },
}
```

`opts.colors.<group>` accepts the full `vim.api.nvim_set_hl()` option table.

`surrounding_word_count` controls how many neighboring words are rendered around the active word.  
Examples:

- `0`: only active word
- `1`: one word on the left and one on the right (when available)
- `3`: up to three words on each side (max supported)

Invalid values (non-numeric, negative, or greater than `3`) are treated as `1`.

In-window defaults that are always available:

- Quit: `q` or `<Esc>`
- Play/Pause: `<Space>`
- Reset: `r`
- Help: `g?`

## User Commands

| Command             | What it does                                                             |
| ------------------- | ------------------------------------------------------------------------ |
| `:Rsvp`             | Starts RSVP on the given range (default range is `%`, the whole buffer). |
| `:RsvpPlay`         | Starts or resumes playback in an active RSVP session.                    |
| `:RsvpPause`        | Pauses playback in an active RSVP session.                               |
| `:RsvpReset`        | Restarts the current RSVP session from the beginning.                    |
| `:RsvpDecreaseWpm`  | Decreases speed by `25` WPM.                                             |
| `:RsvpIncreaseWpm`  | Increases speed by `25` WPM.                                             |
| `:RsvpPreviousStep` | Moves back one word.                                                     |
| `:RsvpNextStep`     | Moves forward one word.                                                  |

## Highlight Groups

| Highlight Group | Used in                                                                                                                               |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `RsvpMain`      | Key hints in the help popup, keymap hints in the status line, `g?` in the help hint line, and the completed part of the progress bar. |
| `RsvpAccent`    | The ORP (Optimal Recognition Point) character of the active word.                                                                     |
| `RsvpPaused`    | `PAUSED` marker in the top status line when playback is paused.                                                                       |
| `RsvpDone`      | `DONE` marker in the elapsed-time line shown after playback finishes.                                                                 |
| `RsvpGhostText` | Unfinished part of the progress bar and surrounding words (when `surrounding_word_count > 0`).                                        |
| `RsvpBreather`  | The trailing punctuation of the active word during a breather pause (when `breather.enabled` and `breather.highlight`).                |

## Similar Plugins

- [BourgeoisBear/vim-rsvp](https://github.com/BourgeoisBear/vim-rsvp)
- [xHugo21/rsvp.nvim](https://github.com/xHugo21/rsvp.nvim)
