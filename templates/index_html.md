# Template: index.html

Root HTML file. Add Google Fonts as needed for your game's theme.

## Minimal (default for most games)

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>__GAME_TITLE__</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

## With Custom Google Fonts

Add font links in `<head>` before `<title>`:

```html
    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800;900&display=swap" rel="stylesheet" />
```

### Fonts used by real games

| Game | Font | Style |
|------|------|-------|
| card_crawler | Cinzel Decorative, MedievalSharp | Medieval / Fantasy |
| tactics_ogre | Press Start 2P | Pixel art / Retro |
| tetris_game | Orbitron, Space Mono | Futuristic / Sci-fi |
| virus_game / sokoban / others | (CSS-only, no custom fonts) | Modern / Clean |

> **Replace** `__GAME_TITLE__` with the game name (e.g. "Virus Game", "Card Crawler").
