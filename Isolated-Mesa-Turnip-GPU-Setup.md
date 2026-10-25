# Isolated Mesa/Turnip GPU Setup for XFCE Apps

This guide documents the isolated GPU-driver setup tested with Zed in
**Xed Editor's Ubuntu 24.04 ARM64 PRoot environment with XFCE and
Termux:X11**. Mesa was not installed system-wide.

> **Important:** The commands below describe our tested environment. The
> exact Mesa/Turnip package URL and package contents can change. Confirm
> the archive is an ARM64 build intended for your environment before
> extracting it. This guide keeps the extracted files under your home
> directory.

## 1. Download the ARM64 Mesa/Turnip package

Create the working directory and download the Ubuntu Noble ARM64 archive
used in our setup:

``` bash
mkdir -p ~/mesa-gpu-test/download ~/mesa-gpu-test/isolated
cd ~/mesa-gpu-test
curl -fL --retry 2 -o mesa.tar.gz 'https://github.com/lfdevs/mesa-for-android-container/releases/download/mesa-26.3.0-devel-20260824/mesa-for-android-container_26.3.0-devel-20260824_ubuntu_noble_arm64.tar.gz'
```

Verify the archive before extracting it:

``` bash
file mesa.tar.gz
tar -tzf mesa.tar.gz | head
```

This is the specific release asset used in our setup
(`mesa-26.3.0-devel-20260824`). If the asset is unavailable later, check
the project's GitHub Releases for the matching Ubuntu Noble ARM64
archive.

## 2. Extract into an isolated directory

For a `.tar.gz` archive whose contents are a filesystem tree, extract
into a temporary directory first so you can inspect its layout:

``` bash
mkdir -p ~/mesa-gpu-test/extract
tar -xzf ~/mesa-gpu-test/mesa.tar.gz -C ~/mesa-gpu-test/extract
```

Inspect the contents:

``` bash
find ~/mesa-gpu-test/extract -maxdepth 4 -type f | head -40
```

If the archive contains a Debian `.deb` package instead of a filesystem
tree, **do not use the tar command above**. Extract the `.deb` into the
isolated directory with:

``` bash
dpkg-deb -x mesa-arm64.deb ~/mesa-gpu-test/isolated
```

Use the actual `.deb` filename in that command. If you downloaded a
tarball with a nested filesystem tree, copy or move only the appropriate
extracted library tree into `~/mesa-gpu-test/isolated`; inspect it
before proceeding. The required paths for the remaining commands are
typically:

``` text
~/mesa-gpu-test/isolated/usr/lib/aarch64-linux-gnu/libvulkan_freedreno.so
~/mesa-gpu-test/isolated/usr/share/vulkan/icd.d/
```

Do not overwrite `/usr/lib` or system Mesa files.

## 3. Create the temporary Vulkan ICD file

This assumes the isolated library exists at the path shown above and the
extracted ICD JSON refers to
`/usr/lib/aarch64-linux-gnu/libvulkan_freedreno.so`. It creates
`/tmp/freedreno-test.json` without modifying the original ICD file.

``` bash
MESA="$HOME/mesa-gpu-test/isolated"
ICD="$(find "$MESA/usr/share/vulkan/icd.d" -type f -name '*freedreno*.json' -print -quit)"

test -n "$ICD" || { echo "Freedreno ICD JSON not found"; exit 1; }
test -f "$MESA/usr/lib/aarch64-linux-gnu/libvulkan_freedreno.so" || {
  echo "Isolated libvulkan_freedreno.so not found"; exit 1;
}

sed "s|/usr/lib/aarch64-linux-gnu/libvulkan_freedreno.so|$MESA/usr/lib/aarch64-linux-gnu/libvulkan_freedreno.so|g" \
  "$ICD" > /tmp/freedreno-test.json
```

Check the temporary file:

``` bash
cat /tmp/freedreno-test.json
```

Confirm its `library_path` points to the isolated library.

## 4. Test Vulkan with the isolated driver

Run while the X11 session is available:

``` bash
MESA="$HOME/mesa-gpu-test/isolated"
XDG_RUNTIME_DIR="$HOME/.xdg-runtime" \
VK_ICD_FILENAMES=/tmp/freedreno-test.json \
LD_LIBRARY_PATH="$MESA/usr/lib/aarch64-linux-gnu${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}" \
vulkaninfo --summary
```

Our successful test reported **Turnip Adreno (TM) 725** and the Mesa
Turnip driver. Results can vary with device, driver build, and
environment.

## 5. Optional: launch XFCE with the isolated driver

This test launcher is separate from the regular XFCE launcher. Create it
as `~/bin/startxfce-gpu-test`:

``` bash
mkdir -p ~/bin

cat > ~/bin/startxfce-gpu-test <<'EOF'
#!/bin/bash

LOCAL="/data/user/0/com.rk.xededitor/local"
MESA="$HOME/mesa-gpu-test/isolated"

export DISPLAY=:0
export TMPDIR="$LOCAL/sandbox/tmp"
export LIBGL_DRIVERS_PATH="$MESA/usr/lib/aarch64-linux-gnu/dri"
export LD_LIBRARY_PATH="$MESA/usr/lib/aarch64-linux-gnu${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
export MESA_LOADER_DRIVER_OVERRIDE=kgsl

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

LOG="$HOME/xfce-gpu-test.log"
echo "=== GPU XFCE test: $(date) ===" > "$LOG"
echo "DISPLAY=$DISPLAY" >> "$LOG"
echo "LIBGL_DRIVERS_PATH=$LIBGL_DRIVERS_PATH" >> "$LOG"
echo "LD_LIBRARY_PATH=$LD_LIBRARY_PATH" >> "$LOG"
echo "MESA_LOADER_DRIVER_OVERRIDE=$MESA_LOADER_DRIVER_OVERRIDE" >> "$LOG"

dbus-launch --exit-with-session sh -c '
startxfce4 &
session_pid=$!
sleep 5
xfconf-query -c xfwm4 -p /general/use_compositing -s false 2>/dev/null || true
wait "$session_pid"
' >> "$LOG" 2>&1
EOF

chmod +x ~/bin/startxfce-gpu-test
```

Use this launcher only when intentionally testing the isolated driver.
Use the normal XFCE launcher for everyday sessions.

## 6. Important limitations

-   Do not copy these libraries into `/usr/lib` or overwrite system Mesa
    files; isolation is the point of this setup.
-   A successful `vulkaninfo` test proves that it loaded the isolated
    Turnip driver in that environment. It does not prove every GUI app
    uses hardware acceleration.
-   Apps may still show DRI3/EGL warnings or use software rendering
    depending on the X server and graphics stack.
-   If the desktop becomes unstable, return to the normal XFCE launcher.
    Do not run both launchers against the same X display simultaneously.
