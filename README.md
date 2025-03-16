# Rayhunter-MW70VK
Getting EFF's Rayhunter running on an Alcatel MW70VK MiFi hotspot

# Background
You've probably seen that the EFF have done some impressive work related to IMSI catcher detection (https://github.com/EFForg/rayhunter)

Unfortunately the hardware that they developed it for isn't useful in Europe - it uses different RF bands (in Europe we use bands 1, 3, 7, 8 and 20)

This project shows how to get it working in Europe. It's a very manual process, but it's pretty simple.

# Setup
You'll need to set up a few things before we get to the fun part:

## Alcatel MW70VK Mobile WiFi Device
This is a European LTE mobile broadband device. It has a few advantages:

1. It's cheap! (Most important)
2. It uses a Qualcomm LTE modem that exposes the /dev/diag device so it's mostly a drop-in replacement for the Orbic RC400L
3. It works in Europe
4. It has "enough" storage (for very specific values of "enough"...)

I got mine from [eBay](https://www.ebay.co.uk/itm/275943210626). There look to be plenty on there. It came with a battery.

Other devices may well be better suited! Please share with folk if you find other alternatives.

## USB Micro Cable
The MV70VK uses a USB Micro cable for charging etc. Dig out your old Kindle cable from your massive box of ancient cables that you never know when you might need!

## Linux VM
I used Kali because that's what I had open at the time. Any Ubuntu-flavour will probably work exactly the same.

### Packages
You'll need some packages installed:

```sh
# Android debugging stuff
sudo apt install adb
sudo apt install sg3_utils

# Rust stuff
sudo apt install rustup
sudo apt install curl build-essential libc6-armhf-cross libc6-dev-armhf-cross gcc-arm-linux-gnueabihf
```

### Rust Setup
Then, configure Rust for cross-compilation for the MW70VK:

```sh
rustup default stable
rustup target add x86_64-unknown-linux-gnu
rustup target add armv7-unknown-linux-gnueabihf
```

## Rayhunter Repo
Next, grab the Rayhunter repository from GitHub:

```sh
git clone https://github.com/EFForg/rayhunter
```

# Source Code Modifications
We need to tweak a few things to get Rayhunter working on a different device. The main difference is the lack of a display - we only have a few LEDs in lieu of a pretty screen. Hopefully that translates to better battery life!

First up, we're going to use an LED to show that Rayhunter is running, so add a constant for the LED's device path:

**rayhunter/bin/src/daemon.rs:42**
```rs
...
use include_dir::{include_dir, Dir};

// EDIT START
const LED_PATH_SIGNAL_RED:&str = "/sys/class/leds/signal-red/brightness";
// EDIT END

// Runs the axum server, taking all the elements needed to build up our
...
```

Next, the big change - we basically cut out all the `FrameBuffer` gubbins that is used to draw on the Orbic's screen as we don't have one. It's a really dirty hack at the moment - we still keep references to the FrameBuffer to avoid changing too much code elsewhere. Not ideal, but we're just looking to get the software running at the moment so it'll do.

**rayhunter/bin/src/daemon.rs:149**
```rs
fn update_ui(task_tracker: &TaskTracker,  config: &config::Config, mut ui_shutdown_rx: oneshot::Receiver<()>, mut ui_update_rx: Receiver<framebuffer::DisplayState>) -> JoinHandle<()> {
    let mut led_colour: u8 = 0;
    task_tracker.spawn_blocking(move || {
        loop {
            match ui_shutdown_rx.try_recv() {
                Ok(_) => {
                    info!("received UI shutdown");
                    break;
                },
                Err(TryRecvError::Empty) => {},
                Err(e) => panic!("error receiving shutdown message: {e}")
            }
            match ui_update_rx.try_recv() {
                    Ok(state) => {
                        _ = 1; // dirty kludge
                    },
                    Err(tokio::sync::mpsc::error::TryRecvError::Empty) => {},
                    Err(e) => error!("error receiving framebuffer update message: {e}")
            }

            // flash the LED on/off every second
            led_colour ^= 1;
            match led_colour {
                0 => std::fs::write(LED_PATH_SIGNAL_RED, "0").unwrap(),
                1 => std::fs::write(LED_PATH_SIGNAL_RED, "1").unwrap(),
                _ => print!("?!"),

            }
            sleep(Duration::from_millis(1000));
        }
    })
}
```

We need to change a path - Rayhunter uses the `/data/` folder which we can't use on the MW70VK:

**rayhunter/bin/src/config.rs:28**
```rs
...
        Config {
            qmdl_store_path: "/cache/rayhunter/qmdl".to_string(),
            port: 8080,
...
```

# Build and Deploy

## Build
The Rayhunter code makes it really easy to build. Run the following from the root of the Rayhunter repository:

```sh
cargo build --release --target="armv7-unknown-linux-gnueabihf
```

It takes a while, storing the resulting executables in `target/armv7-unknown-linux-gnueabihf/release`.

## Deploy
We need to carry out a few steps:

1. Get a shell on the MV70VK device
2. Download some configuration files from it
3. Modify them
4. Reupload them
5. Upload Rayhunter

### Getting Shell

I had to piece this together from various web searches as the Rayhunter docs just say "once you'e rooted your device...". Thanks lads ^^

Luckily it's not too complex. Create a shell script (`enable-debug-mode.sh`) with the following content:

```sh
#!/bin/bash
sudo sg_raw /dev/sg2 16 f9 00 00 00 00 00 00 00 00 00 00 00 00 00 00 -v
```

Don't forget to make it executable:

```sh
chmod 0755 ./enable-debug-mode.sh
```

Next:

1. Plug your USB Micro cable into your PC
2. Get your WM70VK device and remove the back cover (it pops off easily)
3. Remove the battery
4. Press and hold down the `R` button just to the left of the battery slot (keep holding it!)
    1. Plug the USB Micro cable into the WM70VK (all 4x LEDs should light up)
    2. In your VM, attach the device
    3. Watch for the 4x LEDs starting to cycle through - let them do this a few times (four seems enough)
    4. Let go of the `R` button
5. Run `./enable-debug-mode.sh`
6. Check the output contains `NVMe Result=0` (if not, just pull the USB cable and start again)
7. Attach the device toyour VM again - you should get a popup
8. Verify you can get a root shell - `adb shell`

### Prepare Config Files
First of all we need to modify the Rayhunter confguration file as the MW70VK device doesn't have a usable `/data` folder which is where it expects to find data.

**rayhunter/dist/config.toml.example**
```sh
qmdl_store_path = "/cache/rayhunter/qmdl"
...
```

We also need to modify an `init` script to run Rayhunter and some other bits and bobs. Open a new terminal window and run:

```sh
adb pull /system/etc/init.d/start_coreapp_le
```

Edit it as follows:

start_coreapp_le
```sh
#! /bin/sh

#set -e
case "$1" in
  start)
    #start-stop-daemon -S -b -a QCMAP_ConnectionManager /etc/mobileap_cfg.xml d
    #/jrd-resource/bin/core_app

    echo -n "Starting watch scripts: "
    #webs will drop root priviledges after binding
    #echo "0" > /jrd-resource/bin/qcmap_init_finish
    start-stop-daemon -S -b -o -a /jrd-resource/bin/oem_watch

    # START EDIT
    # allow access to rayhunter web UI on port 8080/tcp
    iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
    # start rayhunter
    start-stop-daemon -S -b -m -p /tmp/rayhunter.pid -x /bin/sh -- -c "RUST_LOG=info exec /cache/rayhunter/rayhunter-daemon /cache/rayhunter/config.toml > /cache/rayhunter/rayhunter.log 2>&1"
    # END EDIT
...
```

### Upload Files
Reupload the `init` script and make sure it's executable:

```sh
adb push ./start_coreapp_le /system/etc/init.d/start_coreapp_le
adb shell chmod 0755 /system/etc/init.d/start_coreapp_le
```

Create the folder for Rayhunter, then upload the binary and config file. From the Rayhunter repo root folder:

```sh
adb shell mkdir /cache/rayhunter
adb push target/armv7-unknown-linux-gnueabihf/release/rayhunter-daemon /cache/rayhunter/rayhunter-daemon
adb push dist/config/config.toml.example /cache/rayhunter/config.toml
```

Then make Rayhunter executable:

```sh
adb shell chmod 0755 /cache/rayhunter/rayhunter-daemon
```

Finally, reboot the device - as soon as the LEDs go out, pull the USB plug

```sh
adb shell shutdown -r -t 1 now
```

# Running Rayhunter
1. Plug the battery in
2. Press and hold the power button on the top edge for a few seconds until the LEDs come on
3. Check for a WiFi hotspot named `MW70VK...`
4. Connect to it - you should get a Linkzone page
5. Navigate to `http://192.168.8.1:8080/`

# TODO
This is just about getting the thing running - the code modifications are dirty, and it's not been tested in anger (I don't have an IMSI catcher or any desire to get a stern letter from OFCOM for using one). The following might be nice:

* LED usage to flag suspicous RF events (maybe use the SMS LED?)
* A proper fork of Rayhunter so you can just compile and throw it on the device
* Investigation into potential for adding storage to the MW70VK (for more log storage)
* ???
