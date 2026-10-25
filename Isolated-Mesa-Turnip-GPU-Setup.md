# Isolated Mesa/Turnip GPU Setup for XFCE Apps

This guide documents the **isolated GPU-driver test setup** used with
Zed in Xed Editor's Ubuntu 24.04 ARM64 PRoot environment. The files live
under the user's home directory; Mesa was **not installed system-wide**.

> **Important:** These commands assume the isolated driver files have
> already been extracted into `~/mesa-gpu-test/isolated`. Keep the
> original system Mesa packages and standard XFCE launcher untouched.
> This setup is environment-specific and may not work on other devices
> or Android/Termux versions.

## 1. Paths used by the test

``` bash
MESA="$HOME/mesa-gpu-test/isolated"
```

The isolated directory should contain ARM64 Mesa libraries, including
`libvulkan_freedreno.so`, and a Vulkan ICD JSON under:

``` text
~/mesa-gpu-test/isolated/usr/lib/aarch64-linux-gnu/
~/mesa-gpu-test/isolated/usr/share/vulkan/icd.d/
```

The package's ICD file may point to a system library path. For this
test, a separate temporary copy was patched to point at the isolated
library; the original was not changed.

## 2. Create the temporary Vulkan ICD file

Run this only if the extracted ICD JSON exists at the path below. This
creates `/tmp/freedreno-test.json` and leaves the original unchanged.

``` bash
MESA="$HOME/mesa-gpu-test/isolated"
ICD="$(find "$MESA/usr/share/vulkan/icd.d" -type f -name '*freedreno*.json' -print -quit)"

test -n "$ICD" || { echo "Freedreno ICD JSON not found"; exit 1; }

sed "s|/usr/lib/aarch64-linux-gnu/libvulkan_freedreno.so|$MESA/usr/lib/aarch64-linux-gnu/libvulkan_freedreno.so|g" \
  "$ICD" > /tmp/freedreno-test.json
```

Check that its `library_path` points to the isolated
`libvulkan_freedreno.so`:

``` bash
cat /tmp/freedreno-test.json
```

## 3. Test Vulkan with the isolated driver

Run while the X11 session is available:

``` bash
MESA="$HOME/mesa-gpu-test/isolated"
XDG_RUNTIME_DIR="$HOME/.xdg-runtime" \
VK_ICD_FILENAMES=/tmp/freedreno-test.json \
LD_LIBRARY_PATH="$MESA/usr/lib/aarch64-linux-gnu${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}" \
vulkaninfo --summary
```

A successful test in the original environment reported **Turnip Adreno
(TM) 725** and the Mesa Turnip driver. Your result may differ by device,
libraries, and environment.

## 4. Optional: launch XFCE with the isolated driver

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

## 5. Important limitations

-   Do not copy these libraries into `/usr/lib` or overwrite system Mesa
    files; isolation is the point of this setup.
-   A successful `vulkaninfo` test proves that it could load the
    isolated Turnip driver in that environment. It does not prove every
    GUI application uses GPU acceleration.
-   An app may still show DRI3/EGL warnings or use software rendering,
    depending on the X server and graphics stack.
-   If the desktop becomes unstable, return to the normal XFCE launcher.
    Avoid running the normal and GPU-test launchers against the same X
    display simultaneously.
-   Keep the ICD JSON and isolated directory paths consistent with the
    commands you use.
