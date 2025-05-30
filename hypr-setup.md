
# 🚀 Hyprland Desktop Environment Setup on Arch Linux

This guide provides **step-by-step instructions** to configure a **fully working Wayland DE using Hyprland** from scratch. It uses `tty login`, `~/.bash_profile` to start the session, and optionally `uwsm` for session management. It avoids zsh, nu, or graphical login managers.

---

## 1. 🔧 System Preparation
## 🧑‍💻 Create a Normal User

Avoid doing daily work as root.

```bash
useradd -m -G wheel -s /bin/bash <username>
```
```bash
passwd <username>
```
Enable sudo for wheel group:

```bash
sudo sed -i '/^# %wheel ALL=(ALL:ALL) ALL/s/^# //' /etc/sudoers
```

## 🌐 Set Up SSH (Optional but Recommended)

```bash
pacman -S openssh
systemctl enable sshd --now
```

From your other computer:

```bash
ssh youruser@<ip-address>
```

---
Ensure your system is up to date and has network connectivity.

```bash
sudo pacman -Syu
```

```bash
sudo pacman -S --needed git base-devel
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
yay -Y --gendb
yay -Syu --devel
```
### Required Packages

```bash
sudo pacman -S hyprland waybar wofi alacritty swww brightnessctl   thunar  network-manager-applet playerctl polkit xdg-desktop-portal     xdg-desktop-portal-hyprland wl-clipboard grim slurp  ttf-font-awesome ttf-jetbrains-mono noto-fonts noto-fonts-cjk     noto-fonts-emoji papirus-icon-theme
```
```bash
yay -S wlogout
```


> ✅ **Explanation**:
> - `hyprland`: The window manager
> - `waybar`: Status bar
> - `wofi`: Application launcher
> - `alacritty`: Terminal
> - `thunar`: File manager
> - `swww`: Wallpaper setter
> - `network-manager-applet`: Wi-Fi GUI
> - `xdg-desktop-portal-*`: Required for various GUI app permissions
> - `grim`, `slurp`: Screenshots
> - `fonts`: Usable icons and emoji

---

## 2. 👤 User Environment

### `.bash_profile`

Put this in your `~/.bash_profile`:

```bash
if command -v uwsm &>/dev/null && uwsm check may-start; then
  exec uwsm start hyprland.desktop
fi
```

> 🧠 This ensures `uwsm` starts only if available, with proper environment support.

---

## 3. 🌐 Session Definition

Create a desktop file so `uwsm` can manage the session:

```ini
# ~/.local/share/wayland-sessions/hyprland.desktop
[Desktop Entry]
Name=Hyprland
Exec=Hyprland
Type=Application
```

---
### wofi config
> show=drun
>
> term=alacritty

### waybar config
```json
{
  "layer": "top",
  "position": "top",
  "modules-left": ["hyprland/workspaces"],
  "modules-center": ["clock"],
  "modules-right": ["battery", "pulseaudio", "tray", "custom/wlogout"],

  "clock": {
    "format": "{:%H:%M %d/%m/%Y}"
  },
  "battery": {
    "format": "{capacity}% {icon}",
    "format-icons": ["", "", "", "", ""]
  },
  "pulseaudio": {
    "format": "{volume}% {icon}",
    "format-icons": ["", "", ""]
  },
  "custom/wlogout": {
    "format": "⏻",
    "tooltip": "Power Menu",
    "on-click": "wlogout"
  }
}
```


## 4. 🌈 Hyprland Config

```bash
mkdir -p ~/.config/hypr
cp /usr/share/hyprland/hyprland.conf ~/.config/hypr/hyprland.conf
```

Edit `~/.config/hypr/hyprland.conf`:

### Example Additions

```ini
$terminal = alacritty
$fileManager = thunar
$menu = wofi --show drun

exec-once = waybar
exec-once = swww init
exec-once = nm-applet

bind = SUPER, RETURN, exec, $terminal
bind = SUPER, R, exec, $menu
bind = SUPER, E, exec, $fileManager
```

> 🧠 `exec-once` autostarts background apps.
> `bind` maps keys to actions.

---


## 6. 🧠 Environment Variables

Create:

```bash
~/.config/uwsm/env-hyprland
```

```bash
export XDG_SESSION_TYPE=wayland
export XDG_CURRENT_DESKTOP=Hyprland
export XDG_SESSION_DESKTOP=Hyprland
export QT_QPA_PLATFORM=wayland
export QT_WAYLAND_DISABLE_WINDOWDECORATION=1
export WLR_NO_HARDWARE_CURSORS=1
export _JAVA_AWT_WM_NONREPARENTING=1
export MOZ_ENABLE_WAYLAND=1
```

---

## 7. 🖱️ Input Config

Hyprland config section:

```ini
input {
    kb_layout = us,ru
    kb_options = grp:alt_shift_toggle
}
```

> 🧠 Enables toggling US/RU layout with `Alt+Shift`.

---

## 8. 🧼 Clean Shutdown

To exit Hyprland:

```ini
bind = SUPER, M, exec, uwsm stop
```

> 🧠 `uwsm stop` safely exits the session instead of `exit`.

---

## 9. 📕 Dictionary of Commands and Flags

| Command/Flag | Purpose |
|--------------|---------|
| `exec-once` | Runs a command once at startup |
| `bind` | Binds a key to an action |
| `uwsm start` | Starts a Wayland session with `uwsm` |
| `swww init` | Initializes the wallpaper daemon |
| `xdg-desktop-portal` | Required for sandboxed app permissions |
| `brightnessctl` | Controls screen brightness |
| `wpctl` | Pipewire volume control |
| `playerctl` | Media playback control |
| `Hyprland` | Starts Hyprland manually |
| `exec` | Executes a command, replacing the current shell |
| `Alt+Shift` | Switches keyboard layout as configured |

---

## ✅ Done!

You now have a **minimal yet functional Hyprland DE** with terminal, launcher, file manager, status bar, volume keys, and keyboard layout switching — all without a login manager.
