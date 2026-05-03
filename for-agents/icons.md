# Profile-Specific Colored Icons

To distinguish profiles in the dock/taskbar, generate hue-shifted versions of the default Claude icon using ImageMagick:

```bash
# Profile 2 — purple/lavender (hue rotate -95°)
for size in 16 24 32 48 64 256; do
  magick "/usr/share/icons/hicolor/${size}x${size}/apps/claude-desktop.png" \
    -modulate 100,100,47 \
    "$HOME/.local/share/icons/hicolor/${size}x${size}/apps/claude-desktop-2.png"
done

# Profile 3 — green (hue rotate +120°)
for size in 16 24 32 48 64 256; do
  magick "/usr/share/icons/hicolor/${size}x${size}/apps/claude-desktop.png" \
    -modulate 100,100,167 \
    "$HOME/.local/share/icons/hicolor/${size}x${size}/apps/claude-desktop-3.png"
done

# Profile 4 — indigo/lavender (different shade — saturation boosted)
for size in 16 24 32 48 64 256; do
  magick "/usr/share/icons/hicolor/${size}x${size}/apps/claude-desktop.png" \
    -modulate 110,160,30 \
    "$HOME/.local/share/icons/hicolor/${size}x${size}/apps/claude-desktop-4.png"
done

# Refresh GTK icon cache
gtk-update-icon-cache -t -f ~/.local/share/icons/hicolor
```

`-modulate brightness,saturation,hue` (each 0-200, 100=no change).

After this, the `.desktop` entries' `Icon=claude-desktop-2/3/4` will pick up the colored variants automatically.
