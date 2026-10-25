# XFCE Desktop Setup in Xed Ubuntu

This guide explains how to set up and use an XFCE desktop in Xed
Editor's built-in Ubuntu 24.04 (ARM64) environment. It covers the XFCE
desktop, terminal, file manager, optional web browser, and a one-command
startup script.

> **Important:** These instructions are for Xed's own Ubuntu terminal,
> not the native Termux environment. The display resolution can remain
> at `1280×720`.

## 📚 Setup Guides

Choose a guide:

- **[Zed Editor Setup](ZED-XFCE-Setup.md)** — Install and launch Zed in Xed Ubuntu with XFCE, and configure a custom AI provider.
- **[GPU Acceleration Setup](Isolated-Mesa-Turnip-GPU-Setup.md)** — Set up isolated Mesa/Turnip drivers and test Vulkan support.

## 1. Install XFCE and essential desktop applications

If you have already installed these packages, there is no need to
install them again.

``` bash
apt update
apt install xfce4-session xfce4-panel xfwm4 xfdesktop4 thunar dbus-x11 xfce4-terminal
```

These packages provide the XFCE session, panel/taskbar, window manager,
desktop menu and icons, Thunar file manager, D-Bus session support, and
XFCE Terminal.

## 2. Install Vivaldi browser

These commands install the official Vivaldi ARM64 package repository and
signing key, refresh APT's package index, and install Vivaldi.

Add the repository signing key:

``` bash
sudo mkdir -p /etc/apt/keyrings
sudo curl -fsSLo /etc/apt/keyrings/vivaldi.asc https://repo.vivaldi.com/stable/linux_signing_key.pub
```

Add the official stable repository for ARM64:

``` bash
echo "deb [arch=arm64 signed-by=/etc/apt/keyrings/vivaldi.asc] https://repo.vivaldi.com/stable/deb/ stable main" | sudo tee /etc/apt/sources.list.d/vivaldi.list
```

Update package lists and install Vivaldi:

``` bash
sudo apt update
sudo apt install vivaldi-stable
```

### Launch Vivaldi from the terminal

In this Xed Ubuntu PRoot environment, Vivaldi was able to launch with
`--no-sandbox`:

``` bash
vivaldi-stable --no-sandbox
```

The browser may print warnings about inotify, D-Bus, netlink, or udev
because some host system services and kernel interfaces are unavailable
inside the environment. If the browser window opens and works, those
warnings may not prevent normal browsing.

> **Security note:** `--no-sandbox` disables an important browser
> security boundary. Use this only when necessary for this environment,
> keep Vivaldi updated, and be cautious with untrusted websites and
> downloads.

### Make Vivaldi work from the Applications menu

Copy the system launcher to your personal applications directory:

``` bash
mkdir -p ~/.local/share/applications
cp /usr/share/applications/vivaldi-stable.desktop ~/.local/share/applications/
```

Update the launch command so menu launches include `--no-sandbox`:

``` bash
sed -i 's|^Exec=.*|Exec=/usr/bin/vivaldi-stable --no-sandbox %U|' ~/.local/share/applications/vivaldi-stable.desktop
```

Vivaldi should now be available from the XFCE Applications menu, usually
under **Internet**.

### Add Vivaldi to the XFCE panel

1.  Open the XFCE **Applications** menu and find Vivaldi under
    **Internet**.
2.  Right-click Vivaldi and choose **Add to Panel** if that option is
    available.
3.  If it is not available, right-click the panel and select **Panel →
    Add New Items... → Launcher → Add**.
4.  Right-click the new launcher and open **Properties**. Use the **+**
    button to add Vivaldi from the applications list, if listed.

The panel launcher should use the personal desktop entry configured
above, so Vivaldi launches with `--no-sandbox`.

## 3. Use the XFCE Terminal and File Manager

-   **Terminal:** Open `Terminal Emulator` or `XFCE Terminal` from the
    Application Menu.

-   **File Manager:** Open `File Manager` from the Application Menu. The
    application is called Thunar.

-   To launch the file manager from a terminal, run:

    ``` bash
    thunar
    ```

If either application is not installed, run:

``` bash
apt install xfce4-terminal thunar
```

## 4. Create a one-command startup script

The script below starts Xed's Termux:X11 display, then launches XFCE.
When the XFCE session exits, the script also attempts to stop the X11
process.

``` bash
mkdir -p "$HOME/bin" && cat > "$HOME/bin/start-xfce-desktop" <<'EOF'
#!/bin/bash

LOCAL="/data/user/0/com.rk.xededitor/local"
export DISPLAY=:0
export TMPDIR="$LOCAL/sandbox/tmp"

"$LOCAL/bin/termux-x11" :0 &
X11_PID=$!

cleanup() {
    kill "$X11_PID" 2>/dev/null
    pkill -f '^termux-x11 com\.termux\.x11 :0$' 2>/dev/null
    wait "$X11_PID" 2>/dev/null
}

trap cleanup EXIT INT TERM

for i in $(seq 1 15); do
    if xdpyinfo -display :0 >/dev/null 2>&1; then
        break
    fi
    sleep 1
done

if ! xdpyinfo -display :0 >/dev/null 2>&1; then
    echo "X11 did not start. Check the Termux:X11 app and launcher."
    exit 1
fi

dbus-launch --exit-with-session startxfce4
EOF

chmod +x "$HOME/bin/start-xfce-desktop"
```

## 5. Create the `startxfce` shortcut

Instead of typing the full script path each time, add this shortcut:

``` bash
echo 'export PATH="$HOME/bin:$PATH"' >> "$HOME/.bashrc" && echo 'alias startxfce="$HOME/bin/start-xfce-desktop"' >> "$HOME/.bashrc" && source "$HOME/.bashrc"
```

Verify the shortcut with:

``` bash
type startxfce
```

The output should indicate that `startxfce` is an alias for the
`start-xfce-desktop` script.

## 6. Start and stop the desktop

### Start XFCE

In the Xed Ubuntu terminal, run:

``` bash
startxfce
```

The script starts the X11 display and launches the XFCE desktop. Switch
to the Termux:X11 app if necessary.

### Stop XFCE

Return to the same terminal session where `startxfce` is running and
press **Ctrl+C**. The XFCE session should close, and the script should
also stop the X11 process. The X11 screen with the `×` cursor should
disappear as well.

> When using this startup script, you should not normally need a
> separate process-kill command to stop the desktop.

## 7. Troubleshooting

### `startxfce: command not found`

Load the shell configuration again:

``` bash
source "$HOME/.bashrc"
```

Then verify the alias:

``` bash
type startxfce
```

### `startxfce4: command not found`

``` bash
apt install xfce4-session
```

### `xfce4-terminal: command not found`

``` bash
apt install xfce4-terminal
```

### D-Bus or XFCE settings connection errors

Launch XFCE using the `startxfce` script. It starts the session with:

``` bash
dbus-launch --exit-with-session startxfce4
```

Do not terminate running desktop processes without first checking the
cause of the error.

### X11 shows a black screen or a `×` cursor

This can be normal when the X11 server is running but no XFCE desktop
session or graphical window is open. Start the desktop with `startxfce`.

## 8. Clean the APT cache (optional)

To remove downloaded package archives from APT's cache, run:

``` bash
apt clean
```

This does **not** uninstall installed applications. APT may need to
download package files again in the future, so this cleanup step is
optional.

------------------------------------------------------------------------

**Setup summary:** Xed Ubuntu + Xed's Termux:X11 launcher + XFCE
desktop + XFCE Terminal + Thunar File Manager + Vivaldi browser. The
display resolution can remain at `1280×720`.
