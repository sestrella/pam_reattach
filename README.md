pam\_reattach
[![Build Status](https://travis-ci.org/fabianishere/pam_reattach.svg?branch=master)](https://travis-ci.org/fabianishere/pam_reattach)
=============
This is a PAM module for reattaching to the authenticating user's per-session
bootstrap namespace on macOS. 
This allows users to make use of the `pam_tid` module (Touch ID) from within tmux.

## Purpose
Although in MacOS a user program may survive in the background across login sessions, several services (mostly related to the GUI, such as pasteboard and Touch ID) are 
strictly tied to the login session of a user and as such unavailable for programs in the background session. 
Users of programs such as [tmux](https://github.com/tmux/tmux) and [GNU Screen](https://www.gnu.org/software/screen/) that run in the background to survive across login sessions, 
will thus find that several services such as Touch ID are unavailable or do not work properly.

This PAM module will attempt to move the current program (e.g. `sudo`) to the current active login session,
after which the remaining PAM modules will have access to the per-session services like Touch ID.

If you have installed the additional `reattach-to-session-namespace(8)` program, you may also execute arbitrary
programs from the background in the login session of the user.

See [TN2083](https://developer.apple.com/library/archive/technotes/tn2083/_index.html) for more details
about bootstrap namespaces in MacOS.

## Usage
This module should be invoked before the module that you want to put in the
authenticating user's per-session bootstrap namespace. The module runs in the
authentication phase and should be marked as either `optional` or `required`
(I suggest using `optional` to prevent getting locked out in case of bugs)

Modify the targeted service in `/etc/pam.d/` (such as `/etc/pam.d/sudo`) as explained:
```
auth     optional     pam_reattach.so
auth     sufficient   pam_tid.so
...
```

Make sure you have the module installed. Note that when the module is not installed in `/usr/lib/pam` or `/usr/local/lib/pam` (e.g., on M1 Macs where Homebrew is installed in `/opt/homebrew`), you must specify the full path to the module in the PAM service file as shown below:  
```
auth     optional     /opt/homebrew/lib/pam/pam_reattach.so
auth     sufficient   pam_tid.so
...
```

The `pam_tid` module will try to avoid prompting for a touch when connected via SSH or another remote login method. However, there are situations (e.g. use of tmux and screen) where the current tty may be spawned by a remote session but not detected as such by `pam_tid`. To help mitigate this, the `ignore_ssh` option can be added to the configuration of `pam_reattach` as follows:
```
auth     optional     pam_reattach.so ignore_ssh
auth     sufficient   pam_tid.so
...
```
This will detect the presence of any of `$SSH_CLIENT`, `$SSH_CONNECTION`, or `$SSH_TTY` in the environment, and cause this module to become a no-op.

For further information, see `reattach_aqua(3)`, `pam_reattach(8)` and `reattach-to-session-namespace(8)`.

## Installation
The module is available via Homebrew. Use the following command to install it:

```bash
$ brew install pam-reattach
```

You can also install this module with MacPorts using the following command:

```bash
$ sudo port install pam-reattach
```

## Building 
Alternatively, you may manually build the module. The module is built using [CMake 3](https://cmake.org). Enter the following commands into your
command prompt in the project directory:

```bash
$ cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX:PATH=/usr/local
$ cmake --build build
```

To create a universal binary for use with both Apple Silicon and x86 (e.g. for Rosetta support), use:

```bash
$ cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX:PATH=/usr/local -DCMAKE_OSX_ARCHITECTURES="arm64;x86_64" 
$ cmake --build build
```

If CMake is not able to find `libpam` automatically (e.g., on Nix), you may need to specify the prefix path manually:
```bash
$ cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX:PATH=/usr/local -DCMAKE_PREFIX_PATH="/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/lib/"
$ cmake --build build
```

#### Manual Installation
Then, to install the module, simply run the following command:
```bash
$ cmake --install build
```
Make sure you **keep** the generated `install_manifest.txt` file in the build folder after installation. 

#### Manual Removal 
Run the following command in your command prompt to remove the installation from your system:

```bash
$ xargs rm < build/install_manifest.txt
```

In case you lost `install_manifest.txt`, this is the list of files that are installed:
```
/usr/local/lib/libreattach.a
/usr/local/include/reattach.h
/usr/local/share/man/man3/reattach_aqua.3
/usr/local/lib/pam/pam_reattach.so
/usr/local/share/man/man8/pam_reattach.8
/usr/local/bin/reattach-to-session-namespace
/usr/local/share/man/man8/reattach-to-session-namespace.8
```
 
#### Additional Tools
Additionally, you may build a `reattach-to-session-namespace` command line
utility by specifying the `-DENABLE_CLI=ON` option when calling CMake. This command allows you to reattach to the user's session namespace from the
command line. 

See `reattach-to-session-namespace(8)` for more information. 


## Enabling Touch ID for sudo
To enable Touch ID authorization for `sudo`, please see [this](https://derflounder.wordpress.com/2017/11/17/enabling-touch-id-authorization-for-sudo-on-macos-high-sierra/)
article.

## License
The code is released under the MIT license. See [LICENSE.txt](/LICENSE.txt).
