# handbreakCLI-RaspberryPi

building handbreakCLI on raspberry pi including x265 codec

Does it make sense to build handbreak on raspberry pi? Be warned it wont be fast. 10-20 times slower that i5 Intel CPU laptop. But it works so why not.

I have managed sucessfully compile it on RPi 2B+, 3 and 3B+ running raspbian 4.14.98


### 1. install all dependencies

```
sudo apt-get install git autoconf automake build-essential cmake libass-dev libbz2-dev libfontconfig1-dev libfreetype6-dev libfribidi-dev libharfbuzz-dev libjansson-dev liblzma-dev libmp3lame-dev libogg-dev libopus-dev libsamplerate-dev libspeex-dev libtheora-dev libtool libtool-bin libvorbis-dev libx264-dev libxml2-dev m4 make patch pkg-config python tar yasm zlib1g-dev libvpx-dev xz-utils bzip2 zlib1g
```


### 2. debian nasm is too old we need newer one

```
sudo curl -L 'http://ftp.debian.org/debian/pool/main/n/nasm/nasm_2.14-1_armhf.deb' -o /var/cache/apt/archives/nasm_2.14-1_armhf.deb && sudo dpkg -i /var/cache/apt/archives/nasm_2.14-1_armhf.deb
```


### 3. we clone handbreak from github and jump few commits forward from 1.2.2

only then we can disable nvidia and intel qsv, it was not possible in 1.2.2

```
git clone https://github.com/HandBrake/HandBrake.git && cd HandBrake && git checkout c7119499f5a2da7e5be0afd50a6757778fed53e7
```


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


### 5. now we can configure our project

```
./configure --launch-jobs=$(nproc) --disable-gtk --disable-nvenc --disable-qsv --enable-fdk-aac
```


### 6. we have to make quick-and-dirty hack to x265 source code


We compile with DENABLE_ASSEMBLY=OFF but x265 code does not take it into account that they don't handle it right for ARMv7 (I have reported it to X265 guys so maybe in the future it wont be required if they modify their code)

but first we have to start building it so it is downloaded

```
cd build
```

```
make -j 4 x265
```

Wait unitl you see that files have been downloaded then CTRL-C

```
nano ./contrib/x265/x265_3.0/source/common/primitives.cpp
```

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


### 7. we can build all now

```
make clean
```

```
make -j $(nproc)
```

take a break - it finishes in about 40 min on RPi 3B+, twice as long on RPi 2B+


### 8. when finished we should have excecutable binary HandBrakeCLI in our build folder



### 9. we can use it straight away or install properly

```
sudo make --directory=. install
```


### 10. Basic usage

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

[https://handbrake.fr/docs/en/1.2.0/developer/install-dependencies-debian.html](https://handbrake.fr/docs/en/1.2.0/developer/install-dependencies-debian.html)

[https://handbrake.fr/docs/en/1.2.0/developer/build-linux.html](https://handbrake.fr/docs/en/1.2.0/developer/build-linux.html)

[https://mattgadient.com/2016/06/20/handbrake-0-10-5-nightly-and-arm-armv7-short-benchmark-and-how-to/](https://mattgadient.com/2016/06/20/handbrake-0-10-5-nightly-and-arm-armv7-short-benchmark-and-how-to/)

[https://retropie.org.uk/forum/topic/13092/scrape-videos-and-reencode-them-directly-on-the-raspberry-pi-with-sselph-s-scraper](https://retropie.org.uk/forum/topic/13092/scrape-videos-and-reencode-them-directly-on-the-raspberry-pi-with-sselph-s-scraper)

[https://www.linux.com/learn/how-convert-videos-linux-using-command-line](https://www.linux.com/learn/how-convert-videos-linux-using-command-line)

[https://handbrake.fr/docs/en/1.2.0/cli/cli-options.html](https://handbrake.fr/docs/en/1.2.0/cli/cli-options.html)

