# PlayFi TV App

## Architecture

### Overall Architecture of WebOS OSE

Refer: https://www.webosose.org/docs/guides/core-topics/architecture/architecture-overview/

![Overall Architecture of WebOS OSE](https://www.webosose.org/images/docs/guides/core-topics/architecture/webos-ose-architecture-20221202.png)

---

### Block Diagram for PlayFi TV App with Services

![Block Diagram for PlayFi TV App with Services](/playfi-arch.png)

---

## Development Setup

### System Requirements

1. Linux Machine w/ at least 16G RAM, 300G(free) SSD, i5/i7
2. 64-bit Ubuntu 18 LTS and above

`Note: Windows or Mac machines WILL NOT WORK for developing native services(written in C/C++)!`

---

### Steps

1. Install and Verify Git

```sh
git --version
```

2. Install and Verify GCC

```sh
sudo apt install build-essential
gcc --version
```

3. Install and Verify Make and CMake

```sh
sudo apt install make cmake
cmake --version
make --version
```

4. Install Python3

```sh
sudo apt install python3 python3-pip
python3 --version
pip3 --version
```

5. Install and Verify VirtualBox(v6) and VirtualBox Extension Pack

```sh
sudo apt install virtualbox virtualbox-ext-pack virtualbox-dkms
vboxmanage --version
```

6. Install and Verify [NVM](https://github.com/nvm-sh/nvm?tab=readme-ov-file#installing-and-updating)

7. Install and Verify Node

```sh
nvm install 20
nvm use 20
node -v
npm -v
```

8. Install and Verify WebOS OSE CLI - Ares

```sh
npm install -g @webos-tools/cli
ares --version
```

9. Setting Up the OSE Profile

```sh
ares-config --profile ose
```

10. [NDK Setup](https://www.webosose.org/docs/guides/setup/setting-up-native-development-kit/)

#### NDK Setup - Build NDK

```sh
mkdir ws-webos
cd ws-webos
git clone https://github.com/webosose/build-webos
cd build-webos
git checkout master
sudo scripts/prerequisites.sh
./mcf -p 2 -b 2 qemux86-64
source oe-init-build-env
bitbake -c populate_sdk webos-image
```

`Note: This process may take a LONG time (~ 8-10hrs)`

If the building process succeeds, a script file (.sh) is generated in `build-webos/BUILD/deploy/sdk/`

```sh
ls BUILD/deploy/sdk
```

---

#### NDK Setup - Install NDK

```sh
cd BUILD/deploy/sdk
./webos-sdk-x86_64-core2-64-toolchain-2.25.0.g.sh
```

If the process succeeds, an environment setup file (environment-setup-cortexa72-webos-linux) is generated in the target directory (default: /usr/local/webos-sdk-x86_64).

```sh
ls /usr/local/webos-sdk-x86_64
```

---

#### NDK Setup - Set Up Environment

`IMPORTANT: Run this every time before working on any native service or make it a part of bash file that loads up on every system boot up`

```sh
source /usr/local/webos-sdk-x86_64/environment-setup-core2-64-webos-linux
```

---

### Set Up Emulator

Follow the steps(Steps 1 till 14) shown here to set up your Emulator in VirtualBox using GUI
https://www.webosose.org/docs/tools/sdk/emulator/virtualbox-emulator/emulator-user-guide/#preparing-a-webos-ose-emulator-virtual-machine-image

---

#### How to access Emulator Shell

`Note: This is useful for testing out your services running on the emulator`

```sh
ssh -p 6622 root@localhost
# OR to send commands directly to the emulator shell, use ares-shell command like below
ares-shell -r "luna-send -n 1 -f luna://com.playfi.app.audio/echo '{\"input\":\"webOS\"}'" -d target
```

#### How to set up Ares Device(accessible by Ares CLI)

```sh
# Set up target device
ares-setup-device --add target -i "host=localhost" -i "port=6622" -i "username=root" -i "default=true"
# Verify target device successfully created
ares-setup-device -F
```

On running the command - `ares-setup-device -F`, verify that the target device JSON looks something like this.
There is an interactive version of the above command too - `ares-setup-device`

```json
{
  "profile": "ose",
  "name": "target",
  "default": true,
  "deviceinfo": {
    "ip": "localhost",
    "port": "6622",
    "user": "root"
  },
  "connection": ["ssh"],
  "details": {
    "password": "",
    "description": "new device description"
  }
}
```

####

---

## How to set up a new WebApp with an external JS service

Note: This will help test out the development machine setup

```sh
cd ws-webos
ares-generate -t webapp com.playfi.app
cd com.playfi.app
ares-generate -t js_service com.playfi.app.service.login

# Open the index.html file and update the helloApi to - luna://com.playfi.app.service.login/hello

cd ..
ares-package ./com.playfi.app ./com.playfi.app/com.playfi.app.service.login
```

Finally, install the ipk on the target device and launch the app in the emulator!

```sh
ares-install com.playfi.app_1.0.0_all.ipk -d target
ares-launch com.playfi.app -d target
```

## Packaging external Native(C) service along with the WebApp

Use sample native service provided here as a starter and copy it inside the root folder of your app
Rename all occurrences of `com.playfi.app.audio` with the service ID you would want the service to have on the Luna Bus

```sh
cd com.playfi.app.audio
mkdir BUILD
cd BUILD

# Source Env again
source /usr/local/webos-sdk-x86_64/environment-setup-core2-64-webos-linux

cmake ..
make
# A platform specific folder for native package can be found in /pkg_x86_64

# Go back one folder to being inside the native service
cd ..

# Package the app from inside the native service, because somehow the packaging does not happen correctly if done from anywhere else
ares-package ../com.playfi.app ./pkg_x86_64 ../com.playfi.app.service.login
# Install ipk to target
ares-install com.playfi.app_1.0.0_all.ipk -d target
# Launch using App ID
ares-launch com.playfi.app -d target
```

---

### Input Parameters Format for Service Calls using WebApp or another service

The call to a Luna Bus registered service supports input parameters in the form of a `JSON`.
The response is also sent in the form of a `JSON`

WebApp ----Call to Service1/Method2:`JSON` Payload----> Service1/Method1----> Return Response as `JSON`

---

### Debugging WebApps

Once the app is running in the emulator, this command opens a web browser and you can check the console for any logs/errors from the WebApp.
However, this does not show any logs from inside the services. To check service logs, get a shell access to the emulator and use command `journalctl`

```sh
ares-inspect --device target --app com.playfi.app --open
```

---
