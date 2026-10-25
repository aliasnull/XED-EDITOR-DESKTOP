# Zed Editor on Xed Editor's Ubuntu (ARM64)

This guide documents the user-local Zed setup tested in **Xed Editor's
Ubuntu 24.04 ARM64 PRoot environment with XFCE and Termux:X11**. It
avoids system-wide installation.

> **Note:** Commands use `/home` as the home directory, matching the
> tested environment. `ZED_ALLOW_ROOT=true` is only for this root-based
> PRoot environment; do not use it for a normal desktop Linux
> installation.

## 1. Download and extract Zed

``` bash
mkdir -p ~/zed-test/download ~/zed-test/app
cd ~/zed-test/download
curl -fL --progress-bar https://zed.dev/api/releases/stable/latest/zed-linux-aarch64.tar.gz -o zed-linux-aarch64.tar.gz
tar -xzf zed-linux-aarch64.tar.gz -C ~/zed-test/app
```

Check the executable:

``` bash
ls -l ~/zed-test/app/zed.app/bin/zed
```

## 2. Create an isolated user-data directory

``` bash
mkdir -p ~/zed-test/user-data ~/zed-test/logs
```

## 3. Launch Zed

The following command uses the isolated Mesa/Turnip files described in
the companion GPU guide. If you have not set up that optional driver
environment, remove the `VK_ICD_FILENAMES` and `LD_LIBRARY_PATH` lines.

``` bash
MESA="$HOME/mesa-gpu-test/isolated"
XDG_RUNTIME_DIR="$HOME/.xdg-runtime" \
VK_ICD_FILENAMES=/tmp/freedreno-test.json \
LD_LIBRARY_PATH="$MESA/usr/lib/aarch64-linux-gnu${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}" \
ZED_ALLOW_ROOT=true \
"$HOME/zed-test/app/zed.app/bin/zed" \
  --foreground \
  --user-data-dir "$HOME/zed-test/user-data" \
  2>&1 | tee "$HOME/zed-test/logs/zed.log"
```

-   `--user-data-dir` keeps this test's settings separate from Zed's
    default profile.
-   `ZED_ALLOW_ROOT=true` is needed only because this PRoot environment
    runs as root.
-   Keep the terminal open while running with `--foreground`; press
    `Ctrl+C` to stop it.

## 4. Add Zed to the XFCE application menu

``` bash
mkdir -p ~/.local/share/applications ~/.local/share/icons
cp ~/zed-test/app/zed.app/share/icons/hicolor/512x512/apps/zed.png ~/.local/share/icons/zed.png

cat > ~/.local/share/applications/zed.desktop <<'EOF'
[Desktop Entry]
Version=1.0
Type=Application
Name=Zed Editor
Comment=Lightweight AI Code Editor
Exec=env ZED_ALLOW_ROOT=true XDG_RUNTIME_DIR=/home/.xdg-runtime VK_ICD_FILENAMES=/tmp/freedreno-test.json LD_LIBRARY_PATH=/home/mesa-gpu-test/isolated/usr/lib/aarch64-linux-gnu /home/zed-test/app/zed.app/bin/zed
Icon=/home/.local/share/icons/zed.png
Terminal=false
Categories=Development;IDE;TextEditor;
StartupNotify=true
EOF

chmod +x ~/.local/share/applications/zed.desktop
```

Open the XFCE application menu, find **Zed Editor**, and optionally add
it to the panel.

> The menu launcher uses Zed's default user-data directory. Settings
> saved with `--user-data-dir ~/zed-test/user-data` will not
> automatically appear in the default profile. Configure your provider
> in whichever profile you intend to use.

## AI provider setup

In Zed, open **Settings → AI → LLM Providers → Add OpenAI-Compatible
Provider**. Enter the API URL, API key, and model details supplied by
your provider. Never commit API keys to a public repository.

## Notes

-   This is a user-local extracted installation, not a system package
    installation.
-   This setup was tested in an ARM64 PRoot environment; it is not
    universal for all Linux systems.
-   GPU environment variables do not guarantee every part of Zed is
    hardware-accelerated. See the companion GPU guide.
