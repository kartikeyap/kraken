<img src="kraken.png" width="450" />

Kraken is a simple Yara-based IOC scanner tool that can be built for Windows, Mac and Linux. It is primarily intended for incident response, research and ad-hoc detections (*not* for endpoint protection). Following are the core functionalities:

- Scan running executables and memory of running processes with provided Yara rules.
- Scan executables installed for autorun (leveraging [go-autoruns](https://github.com/botherder/go-autoruns))
- Report any detection to a remote server provided with a Django-based web interface.
- Run continuously and periodically check for new autoruns and scan any newly executed process.

## Building

In order to build Kraken you will need to have Go installed on your system. We recommend using Go >= 1.11 in order to leverage the native support for Go Modules.

Most Go libraries depedencies are available to install through:

    $ make deps

### Building Linux binaries

You need to install Yara development libraries and headers. You should download and compile Yara from the [official sources](https://github.com/VirusTotal/yara). It will require `dh-autoreconf` installed and you will need to configure some compilation flags. This is most likely the procedure you will need to follow:

    $ sudo apt install dh-autoreconf
    $ wget https://github.com/VirusTotal/yara/archive/v3.8.1.tar.gz
    $ tar -zxvf yara-v3.8.1.tar.gz
    $ cd yara-3.8.1
    $ ./bootstrap.sh
    $ ./configure --without-crypto
    $ make && sudo make install
    $ sudo ldconfig

Compiling Kraken requires to specify a path to a file or a folder that contain the Yara rules you wish to embed with the binary. You can try for example with:

    $ BACKEND=example.com RULES=test/ make linux

### Building Windows binaries

Cross-compiling Windows binaries from a Linux development machine is a slightly more complicated process. Firstly you will need to install MingW and some other depedencies:

    $ sudo apt install gcc mingw-w64 automate libtool make

Next you will need to download Yara sources. Use the latest available version, which at the time of writing is 3.8.1:

    $ wget https://github.com/VirusTotal/yara/archive/v3.8.1.tar.gz

Unpack the archive and export `YARA_SRC` to the newly created folder:

    $ export YARA_SRC=<folder>

Next you need to bootstrap Yara sources and compile them with MingW. The following instructions are to compile it for **32bit**:

    $ cd ${YARA_SRC}
    $ ./bootstrap.sh
    $ ./configure --host=i686-w64-mingw32 --without-crypto --prefix=${YARA_SRC}/i686-w64-mingw32
    % make -C ${YARA_SRC}
    % make -C ${YARA_SRC} install

Now we can download and build `go-yara` for 32bit using the following command:

    $ go get -d -u github.com/hillu/go-yara
    $ GOOS=windows GOARCH=386 CGO_ENABLED=1 \
      CC=i686-w64-mingw32-gcc \
      PKG_CONFIG_PATH=${YARA_SRC}/i686-w64-mingw32/lib/pkgconfig \
      go install -ldflags '-extldflags "-static"' github.com/hillu/go-yara

Now you can compile Kraken using:

    $ BACKEND=example.com RULES=test/ make windows

If you get errors such as ` undefined reference to 'yr_compiler_add_file'` you might need to pass the `PKG_CONFIG_PATH` variable:

    $ PKG_CONFIG_PATH=${YARA_SRC}/i686-w64-mingw32/lib/pkgconfig BACKEND=example.com RULES=test/ make windows

## Running

Once the binaries are compiled you will have a `kraken-launcher` and an `kraken` in the appropriate platform build folder.

`kraken` can be launched without any argument and it will perform a scan of detected autorun entries and running processes and terminate. It will not communicate any results to any remote server.

Alternatively, `kraken` can also be launched using the following arguments:

    -daemon
        Enable daemon mode (this will also enable the report flag)
    -report
        Enable reporting of events to the backend
    -debug
        Enable debug logs


Launching `kraken -daemon` will execute a first scan and then run continuously. In *daemon* mode Kraken will monitor any new process creation and scan its binary and memory, as well as check regularly for any new entries registered for autorun. Any detection will be reported back to the configured server along with a regular heartbeat.

## License

Kraken is released under the [GNU General Public License v3.0](LICENSE) and is copyrighted to Claudio Guarnieri.
