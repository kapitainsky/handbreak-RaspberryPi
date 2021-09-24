# HandBreak - RaspberryPi

building HandBreak on raspberry pi including x265 codec

![image](https://user-images.githubusercontent.com/15635386/134711026-f6571ce2-64ee-4430-aedd-286430437273.png)

Does it make sense to build handbreak on raspberry pi? Be warned it wont be fast. 10-20 times slower that i5 Intel CPU laptop. But it works so why not.

I have managed sucessfully compile it on RPi 2B+, 3, 3B+ and 4 running raspbian based on Debian 9 and 10.
<br>

### 1. install all dependencies

```
sudo apt-get install autoconf automake build-essential cmake git libass-dev libbz2-dev libfontconfig1-dev libfreetype6-dev libfribidi-dev libharfbuzz-dev libjansson-dev liblzma-dev libmp3lame-dev libnuma-dev libogg-dev libopus-dev libsamplerate-dev libspeex-dev libtheora-dev libtool libtool-bin libvorbis-dev libx264-dev libxml2-dev libvpx-dev m4 make ninja-build patch pkg-config python tar zlib1g-dev patch libvpx-dev xz-utils bzip2 zlib1g libturbojpeg0-dev appstream
```

If you want GUI version add:

Debian 9:
```
sudo apt-get install intltool libappindicator-dev libdbus-glib-1-dev libglib2.0-dev libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev libgtk-3-dev libgudev-1.0-dev libnotify-dev libwebkitgtk-3.0-dev
```

Debian 10:
```
sudo apt-get install intltool libappindicator-dev libdbus-glib-1-dev libglib2.0-dev libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev libgtk-3-dev libgudev-1.0-dev libnotify-dev libwebkit2gtk-4.0-dev
```
<br>

### 2. Install nasm and meson

In Debian 9 nasm and meson are too old we need newer ones:

```
sudo curl -L 'http://ftp.debian.org/debian/pool/main/n/nasm/nasm_2.14-1_armhf.deb' -o /var/cache/apt/archives/nasm_2.14-1_armhf.deb && sudo dpkg -i /var/cache/apt/archives/nasm_2.14-1_armhf.deb
```

```
sudo add-apt-repository -s 'deb http://deb.debian.org/debian stretch-backports main' \
&& \
sudo apt-get update \
&& \
sudo apt-get -t stretch-backports install meson
```

In Debian 10 things are easier:

```
sudo apt-get install meson nasm
```
<br>

### 3. get HandBreak source code

```
git clone -b 1.4.1 https://github.com/HandBrake/HandBrake.git
```

to get 1.4.1 version

```
cd HandBrake
```
<br>

### 4. we have to add extra configure parameters to X265 module

Without it won't compile on RPi

```
echo "X265_8.CONFIGURE.extra +=  -DENABLE_ASSEMBLY=OFF -DENABLE_PIC=ON -DENABLE_AGGRESSIVE_CHECKS=ON -DENABLE_TESTS=ON -DCMAKE_SKIP_RPATH=ON" >> ./contrib/x265_8bit/module.defs \
&& \
echo "X265_10.CONFIGURE.extra +=  -DENABLE_ASSEMBLY=OFF -DENABLE_PIC=ON -DENABLE_AGGRESSIVE_CHECKS=ON -DENABLE_TESTS=ON -DCMAKE_SKIP_RPATH=ON" >>  ./contrib/x265_10bit/module.defs \
&& \
echo "X265_12.CONFIGURE.extra +=  -DENABLE_ASSEMBLY=OFF -DENABLE_PIC=ON -DENABLE_AGGRESSIVE_CHECKS=ON -DENABLE_TESTS=ON -DCMAKE_SKIP_RPATH=ON" >>  ./contrib/x265_12bit/module.defs \
&& \
echo "X265.CONFIGURE.extra  +=  -DENABLE_ASSEMBLY=OFF -DENABLE_PIC=ON -DENABLE_AGGRESSIVE_CHECKS=ON -DENABLE_TESTS=ON -DCMAKE_SKIP_RPATH=ON" >>  ./contrib/x265/module.defs
```
<br>

### 5. now we can configure our project

For CLI only:
```
./configure --launch-jobs=$(nproc) --disable-gtk --disable-nvenc --disable-qsv --enable-fdk-aac
```

For CLI and GUI:
```
./configure --launch-jobs=$(nproc) --disable-nvenc --disable-qsv --enable-fdk-aac
```
<br>

### 6. we have to make quick-and-dirty hack to x265 source code


We compile with DENABLE_ASSEMBLY=OFF but x265 code does not take it into account that they don't handle it right for ARMv7 (I have reported it to X265 guys so maybe in the future it wont be required if they modify their code)

but first we have to start building it so it is downloaded

```
cd build
```

```
make x265
```

Wait unitl you see that files have been downloaded and patched. It is the first thing happening. Then stop the rest by pressing CTRL-C. 

Now we can make raspberry pi compatible:
```
nano ./contrib/x265/x265_3.5/source/common/primitives.cpp
```
note that `x265_3.5` folder can be different. Change it accordingly. 

and change following section - at the end of the file:

```
#if X265_ARCH_ARM == 0
void PFX(cpu_neon_test)(void) {}
int PFX(cpu_fast_neon_mrc_test)(void) { return 0; }
#endif // X265_ARCH_ARM
```

to

```
#if X265_ARCH_ARM != 0
void PFX(cpu_neon_test)(void) {}
int PFX(cpu_fast_neon_mrc_test)(void) { return 0; }
#endif // X265_ARCH_ARM
```

we just change == condition to !=
<br>
<br>

### 7. let's build

```
make clean
```

if you see some errors after `make clean` you can safely ignore them. This step is only needed to clean x265 interrupted build.


and now we can start full Handbreak build

```
make -j $(nproc)
```

you can take a break - It takes time. 25 min on RPi 4. And many times longer on older RPi models. 
<br>
<br>

### 8. when finished we should have excecutable binaries ready

- CLI: build/HandBrakeCLI
- GUI: build/gtk/src/ghb
<br>

### 9. we can use it straight away or install properly

```
sudo make --directory=. install
```
<br>

### 10. basic CLI usage

```
HandBrakeCLI -i PATH-OF-SOURCE-FILE -o NAME-OF-OUTPUT-FILE --"preset-name"
```

to see available profiles:

```
HandBrakeCLI --preset-list
```

example:

```
./HandBrakeCLI -i /media/Films/test.avi -o /media/Films/test.mkv --preset="H.264 MKV 720p30"
```


### Sources:

[https://handbrake.fr/docs/en/1.3.0/developer/install-dependencies-debian.html](https://handbrake.fr/docs/en/1.3.0/developer/install-dependencies-debian.html)

[https://handbrake.fr/docs/en/1.3.0/developer/build-linux.html](https://handbrake.fr/docs/en/1.3.0/developer/build-linux.html)

[https://mattgadient.com/2016/06/20/handbrake-0-10-5-nightly-and-arm-armv7-short-benchmark-and-how-to/](https://mattgadient.com/2016/06/20/handbrake-0-10-5-nightly-and-arm-armv7-short-benchmark-and-how-to/)

[https://retropie.org.uk/forum/topic/13092/scrape-videos-and-reencode-them-directly-on-the-raspberry-pi-with-sselph-s-scraper](https://retropie.org.uk/forum/topic/13092/scrape-videos-and-reencode-them-directly-on-the-raspberry-pi-with-sselph-s-scraper)

[https://www.linux.com/learn/how-convert-videos-linux-using-command-line](https://www.linux.com/learn/how-convert-videos-linux-using-command-line)

[https://handbrake.fr/docs/en/1.3.0/cli/cli-options.html](https://handbrake.fr/docs/en/1.3.0/cli/cli-options.html)

[GUI compilation - https://github.com/rafaelmaeuer/handbrake-raspberry-pi](https://github.com/rafaelmaeuer/handbrake-raspberry-pi)
