<div align="center">
  <img src="assets/logo.svg" width="160" height="160" alt="Birthday Game"/>

  <h1>Birthday Game</h1>

  <p>A browser-based dungeon crawler built as a birthday gift</p>

  [![HTML5](https://img.shields.io/badge/HTML5-E34F26?style=for-the-badge&logo=html5&logoColor=white)](https://developer.mozilla.org/en-US/docs/Web/HTML)
  [![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black)](https://developer.mozilla.org/en-US/docs/Web/JavaScript)
  [![Browser](https://img.shields.io/badge/Browser-4285F4?style=for-the-badge&logo=googlechrome&logoColor=white)](#running-it)
  [![No Install](https://img.shields.io/badge/No%20Install-22C55E?style=for-the-badge)](#running-it)

</div>

---

A birthday present for NopeYep. A browser-based dungeon crawler you open by double-clicking `index.html` -- no install, no build step, nothing to configure.

## How it works

The whole game lives in a single `index.html` file. Everything is rendered on a `<canvas>` using plain JavaScript and the Web Audio API for sound effects.

There are two acts, each with a handful of combat rooms and a boss fight:

- **Act 1** -- Garden Path through Boss Chamber (Bloom)
- **Act 2** -- Foggy Hollow through Pinky's Lair (Pinky + Owlet)

Between acts there's a checkpoint room where the player can catch their breath.

## Controls

| Input | Action |
|-------|--------|
| WASD / Arrow keys | Move |
| Arrow keys (while moving with WASD) | Shoot |
| Space | Dash (if unlocked) |
| M | Mute / unmute music |

After clearing a room you pick a skill upgrade. A few of them: homing shots, explosive tears, piercing, triple shot, orbital projectiles, freeze, boomerang, and speed.

There's a normal mode and a hell mode. Hell mode cuts your health, shortens iframes, and makes enemies faster and tankier.

## Folder structure

```
index.html        the entire game
img/              screenshot messages shown in popups (msg-1.png ... msg-15.png)
vid/              video messages played during the run (vid-1.mp4 ... vid-5.mp4)
music/            background music loop (music.wav)
sprites/          all pixel art assets
  1 Pink_Monster/ Pink Monster boss sprites
  2 Owlet_Monster/ Owlet boss sprites  
  3 Dude_Monster/ player character sprites
  fly/            bat enemy sprites
  golem/          blue golem brute sprites
  Momo-Mama/      Bloom boss sprites + green grub variant
  Cyclops Sprite Sheet.png
  human-soldier_sword_shield/ sentinel enemy sprites
  pixel-art/      tilesets (grass, stone, plants, props)
```

## Swapping in your own content

Photos and videos are configured at the top of `index.html` in the `CONFIG` object. To change them, drop new files into `img/` and `vid/` and update the file paths there.

## Running it

Open `index.html` in any modern browser. Chrome and Firefox work fine. The game asks for a click before starting audio (browser policy), so just hit play on the title screen.
