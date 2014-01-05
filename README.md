### Patch toolchain

    sudo nano /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/cpp

edit

        -imacros|-include|-idirafter|-iprefix|-iwithprefix)

to

        -imacros|-include|-idirafter|-iprefix|-iwithprefix|-isysroot|-arch)

### Checkout and apply patches

    git clone https://github.com/tarotanaka0/build-ios.git
    cd build-ios
    git submodule init
    git submodule update
    cp android/external/fdlibm/makefile.in android/external/fdlibm/Makefile.in 
    (cd android/openssl-upstream \
       && (for x in \
               progs \
               handshake_cutthrough \
               jsse \
               channelid \
               eng_dyn_dirs \
               fix_clang_build \
               tls12_digests \
               alpn; \
             do patch -p1 < ../external/openssl/patches/$x.patch; done) )
    curl -Of http://oss.readytalk.com/avian/expat-2.1.0.tar.gz
    (cd android/external && tar xzf ../../expat-2.1.0.tar.gz && mv expat-2.1.0 expat)
    rm expat-2.1.0.tar.gz

### Build dependencies

    mkdir -p build/icu4c_host
    (cd build/icu4c_host/ && ../../android/external/icu4c/configure && make)
    
    export SDKROOT=/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS7.0.sdk
    export ARCH=armv7
    
    mkdir -p build/${ARCH}
    (mkdir -p build/icu4c-${ARCH} && \
        cd build/icu4c-${ARCH}/ && \
        CFLAGS="-fPIC -arch ${ARCH} -pipe -isysroot ${SDKROOT} -miphoneos-version-min=7.0" \
        CXXFLAGS="-fPIC -arch ${ARCH} -pipe -isysroot ${SDKROOT} -miphoneos-version-min=7.0" \
        CPPFLAGS="-arch ${ARCH} -isysroot ${SDKROOT} -I$(pwd)/../../android/external/icu4c/tools/tzcode" \
        ../../android/external/icu4c/configure --host=${ARCH}-apple-darwin \
        --enable-static --disable-shared -with-cross-build=$(pwd)/../icu4c_host/ && \
        make && \
        mv lib/libicui18n.a ../${ARCH}/ && \
        mv lib/libicuuc.a ../${ARCH}/ && \
        mv lib/libicudata.a ../${ARCH}/ )
    (mkdir -p build/expat-${ARCH} && \
        cd build/expat-${ARCH}/ && \
        CFLAGS="-fPIC -arch ${ARCH} -pipe -isysroot ${SDKROOT} -miphoneos-version-min=7.0" \
        CXXFLAGS="-fPIC -arch ${ARCH} -pipe -isysroot ${SDKROOT} -miphoneos-version-min=7.0" \
        CPPFLAGS="-arch ${ARCH} -isysroot ${SDKROOT}" \
        ../../android/external/expat/configure \
        --host=${ARCH}-apple-darwin --enable-static --disable-shared && \
        make && mv .libs/libexpat.a ../${ARCH}/ )
    (cd android/external/fdlibm/ && \
        CFLAGS="-fPIC -arch ${ARCH} -pipe -isysroot ${SDKROOT} -miphoneos-version-min=7.0" \
        CXXFLAGS="-fPIC -arch ${ARCH} -pipe -isysroot ${SDKROOT} -miphoneos-version-min=7.0" \
        CPPFLAGS="-arch ${ARCH} -isysroot ${SDKROOT}" \
        bash configure \
        --host=${ARCH}-apple-darwin --enable-static --disable-shared && \
        make && mv libfdm.a ../../../build/${ARCH}/ && \
        make clean && rm *.o Makefile config.log config.status )
    (cd android/openssl-upstream/ && \
        CC="gcc -fPIC -arch ${ARCH} -pipe -isysroot ${SDKROOT} -miphoneos-version-min=7.0" \
        ./Configure BSD-generic32 && make && \
        mv libssl.a ../../build/${ARCH}/ && \
        mv libcrypto.a ../../build/${ARCH}/ && \
        make clean )

### Build hello (with new terminal)

    (cd hello-ios && \
        JAVA_HOME=/Library/Java/Home \
        PROGUARD_HOME=~/bin/proguard4.11/ \
        make android=$(pwd)/../android )
