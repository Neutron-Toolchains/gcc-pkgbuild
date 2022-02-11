# Maintainer: Giancarlo Razzolini <grazzolini@archlinux.org>
# Maintainer: Frederik Schwan <freswa at archlinux dot org>
# Contributor:  Bartłomiej Piotrowski <bpiotrowski@archlinux.org>
# Contributor: Allan McRae <allan@archlinux.org>
# Contributor: Daniel Kozak <kozzi11@gmail.com>

# toolchain build order: linux-api-headers->glibc->binutils->gcc->glibc->binutils->gcc
# NOTE: libtool requires rebuilt with each new gcc version

pkgname=(gcc gcc-libs gcc-fortran gcc-objc gcc-ada gcc-go lib32-gcc-libs gcc-d)
pkgver=11.2.1
_majorver=${pkgver%%.*}
_islver=0.24
pkgrel=1
pkgdesc='The GNU Compiler Collection'
arch=(x86_64)
license=(GPL LGPL FDL custom)
url='https://gcc.gnu.org'
makedepends=(binutils libmpc gcc-ada doxygen lib32-glibc lib32-gcc-libs python git libxcrypt)
checkdepends=(dejagnu inetutils)
options=(!emptydirs !lto)
_libdir=usr/lib/gcc/$CHOST/${pkgver%%+*}
source=("git+https://gcc.gnu.org/git/gcc.git#branch=releases/gcc-$_majorver"
        https://libisl.sourceforge.io/isl-${_islver}.tar.xz
        c89
        c99
        gdc_phobos_path.patch
        gcc-ada-repro.patch
        gcc11-Wno-format-security.patch
)
sha256sums=('SKIP'
            '043105cc544f416b48736fff8caf077fb0663a717d06b1113f16e391ac99ebad'
            'de48736f6e4153f03d0a5d38ceb6c6fdb7f054e8f47ddd6af0a3dbf14f27b931'
            '2513c6d9984dd0a2058557bf00f06d8d5181734e41dcfe07be7ed86f2959622a'
            'c86372c207d174c0918d4aedf1cb79f7fc093649eb1ad8d9450dccc46849d308'
            '1773f5137f08ac1f48f0f7297e324d5d868d55201c03068670ee4602babdef2f'
            '504e4b5a08eb25b6c35f19fdbe0c743ae4e9015d0af4759e74150006c283585e')

prepare() {
  [[ ! -d gcc ]] && ln -s gcc-${pkgver/+/-} gcc
  cd gcc

  # link isl for in-tree build
  ln -s ../isl-${_islver} isl

  # Do not run fixincludes
  sed -i 's@\./fixinc\.sh@-c true@' gcc/Makefile.in

  # Arch Linux installs x86_64 libraries /lib
  sed -i '/m64=/s/lib64/lib/' gcc/config/i386/t-linux64

  # hack! - some configure tests for header files using "$CPP $CPPFLAGS"
  sed -i "/ac_cpp=/s/\$CPPFLAGS/\$CPPFLAGS -O3/" {libiberty,gcc}/configure

  # D hacks
  patch -p1 -i "$srcdir/gdc_phobos_path.patch"

  # Reproducible gcc-ada
  patch -Np0 < "$srcdir/gcc-ada-repro.patch"

  # configure.ac: When adding -Wno-format, also add -Wno-format-security
  patch -Np0 < "$srcdir/gcc11-Wno-format-security.patch"

  mkdir -p "$srcdir/gcc-build"
}

build() {
  cd gcc-build

  # Credits @allanmcrae
  # https://github.com/allanmcrae/toolchain/blob/f18604d70c5933c31b51a320978711e4e6791cf1/gcc/PKGBUILD
  # TODO: properly deal with the build issues resulting from this
  CFLAGS=${CFLAGS/-Werror=format-security/}
  CXXFLAGS=${CXXFLAGS/-Werror=format-security/}

  "$srcdir/gcc/configure" --prefix=/usr \
      CFLAGS="-O3 -pipe -ffunction-sections -fdata-sections" \
      CXXFLAGS="-O3 -pipe -ffunction-sections -fdata-sections" \
      --libdir=/usr/lib \
      --libexecdir=/usr/lib \
      --mandir=/usr/share/man \
      --infodir=/usr/share/info \
      --with-bugurl=https://bugs.archlinux.org/ \
      --enable-languages=c,c++,ada,fortran,go,lto,objc,obj-c++,d \
      --with-isl \
      --with-linker-hash-style=gnu \
      --with-system-zlib \
      --enable-__cxa_atexit \
      --enable-cet=auto \
      --enable-checking=release \
      --enable-clocale=gnu \
      --enable-default-pie \
      --enable-default-ssp \
      --enable-gnu-indirect-function \
      --enable-gnu-unique-object \
      --enable-install-libiberty \
      --enable-linker-build-id \
      --enable-lto \
      --enable-multilib \
      --enable-plugin \
      --enable-shared \
      --enable-threads=posix \
      --disable-libssp \
      --disable-libstdcxx-pch \
      --disable-libunwind-exceptions \
      --disable-werror \
      --with-pkgversion="Neutron GCC" \
      gdc_include_dir=/usr/include/dlang/gdc

  make -O -j$(nproc --all)

  # make documentation
  make -O -C $CHOST/libstdc++-v3/doc doc-man-doxygen -j$(nproc --all)
}

check() {
  cd gcc-build

  # disable libphobos test to avoid segfaults and other unfunny ways to waste my time  
  sed -i '/maybe-check-target-libphobos \\/d' Makefile 

  # do not abort on error as some are "expected"
  make -O -k -j$(nproc --all) check || true
  "$srcdir/gcc/contrib/test_summary"
}

package_gcc-libs() {
  pkgdesc='Runtime libraries shipped by GCC'
  depends=('glibc>=2.27')
  options+=(!strip)
  provides=($pkgname-multilib libgo.so libgfortran.so libgphobos.so
            libubsan.so libasan.so libtsan.so liblsan.so)
  replaces=($pkgname-multilib libgphobos)

  cd gcc-build
  make -C $CHOST/libgcc DESTDIR="$pkgdir" install-shared -j$(nproc --all)
  rm -f "$pkgdir/$_libdir/libgcc_eh.a"

  for lib in libatomic \
             libgfortran \
             libgo \
             libgomp \
             libitm \
             libquadmath \
             libsanitizer/{a,l,ub,t}san \
             libstdc++-v3/src \
             libvtv; do
    make -C $CHOST/$lib DESTDIR="$pkgdir" install-toolexeclibLTLIBRARIES -j$(nproc --all)
  done

  make -C $CHOST/libobjc DESTDIR="$pkgdir" install-libs -j$(nproc --all)
  make -C $CHOST/libstdc++-v3/po DESTDIR="$pkgdir" install -j$(nproc --all)

  make -C $CHOST/libphobos DESTDIR="$pkgdir" install -j$(nproc --all)
  rm -rf "$pkgdir"/$_libdir/include/d/
  rm -f "$pkgdir"/usr/lib/libgphobos.spec

  for lib in libgomp \
             libitm \
             libquadmath; do
    make -C $CHOST/$lib DESTDIR="$pkgdir" install-info -j$(nproc --all)
  done

  # remove files provided by lib32-gcc-libs
  rm -rf "$pkgdir"/usr/lib32/

  # Install Runtime Library Exception
  install -Dm644 "$srcdir/gcc/COPYING.RUNTIME" \
    "$pkgdir/usr/share/licenses/gcc-libs/RUNTIME.LIBRARY.EXCEPTION"
}

package_gcc() {
  pkgdesc="The GNU Compiler Collection - C and C++ frontends"
  depends=("gcc-libs=$pkgver-$pkgrel" 'binutils>=2.28' libmpc)
  groups=('base-devel')
  optdepends=('lib32-gcc-libs: for generating code for 32-bit ABI')
  provides=($pkgname-multilib)
  replaces=($pkgname-multilib)
  options+=(staticlibs)

  cd gcc-build

  make -C gcc DESTDIR="$pkgdir" install-driver install-cpp install-gcc-ar \
    c++.install-common install-headers install-plugin install-lto-wrapper -j$(nproc --all)

  install -m755 -t "$pkgdir/usr/bin/" gcc/gcov{,-tool}
  install -m755 -t "$pkgdir/${_libdir}/" gcc/{cc1,cc1plus,collect2,lto1}

  make -C $CHOST/libgcc DESTDIR="$pkgdir" install -j$(nproc --all)
  make -C $CHOST/32/libgcc DESTDIR="$pkgdir" install -j$(nproc --all)
  rm -f "$pkgdir"/usr/lib{,32}/libgcc_s.so*

  make -C $CHOST/libstdc++-v3/src DESTDIR="$pkgdir" install -j$(nproc --all)
  make -C $CHOST/libstdc++-v3/include DESTDIR="$pkgdir" install -j$(nproc --all)
  make -C $CHOST/libstdc++-v3/libsupc++ DESTDIR="$pkgdir" install -j$(nproc --all)
  make -C $CHOST/libstdc++-v3/python DESTDIR="$pkgdir" install -j$(nproc --all)
  make -C $CHOST/32/libstdc++-v3/src DESTDIR="$pkgdir" install -j$(nproc --all)
  make -C $CHOST/32/libstdc++-v3/include DESTDIR="$pkgdir" install -j$(nproc --all)
  make -C $CHOST/32/libstdc++-v3/libsupc++ DESTDIR="$pkgdir" install -j$(nproc --all)

  make DESTDIR="$pkgdir" install-libcc1 -j$(nproc --all)
  install -d "$pkgdir/usr/share/gdb/auto-load/usr/lib"
  mv "$pkgdir"/usr/lib/libstdc++.so.6.*-gdb.py \
    "$pkgdir/usr/share/gdb/auto-load/usr/lib/"
  rm "$pkgdir"/usr/lib{,32}/libstdc++.so*

  make DESTDIR="$pkgdir" install-fixincludes -j$(nproc --all)
  make -C gcc DESTDIR="$pkgdir" install-mkheaders -j$(nproc --all)

  make -C lto-plugin DESTDIR="$pkgdir" install -j$(nproc --all)
  install -dm755 "$pkgdir"/usr/lib/bfd-plugins/
  ln -s /${_libdir}/liblto_plugin.so \
    "$pkgdir/usr/lib/bfd-plugins/"

  make -C $CHOST/libgomp DESTDIR="$pkgdir" install-nodist_{libsubinclude,toolexeclib}HEADERS -j$(nproc --all)
  make -C $CHOST/libitm DESTDIR="$pkgdir" install-nodist_toolexeclibHEADERS -j$(nproc --all)
  make -C $CHOST/libquadmath DESTDIR="$pkgdir" install-nodist_libsubincludeHEADERS -j$(nproc --all)
  make -C $CHOST/libsanitizer DESTDIR="$pkgdir" install-nodist_{saninclude,toolexeclib}HEADERS -j$(nproc --all)
  make -C $CHOST/libsanitizer/asan DESTDIR="$pkgdir" install-nodist_toolexeclibHEADERS -j$(nproc --all)
  make -C $CHOST/libsanitizer/tsan DESTDIR="$pkgdir" install-nodist_toolexeclibHEADERS -j$(nproc --all)
  make -C $CHOST/libsanitizer/lsan DESTDIR="$pkgdir" install-nodist_toolexeclibHEADERS -j$(nproc --all)
  make -C $CHOST/32/libgomp DESTDIR="$pkgdir" install-nodist_toolexeclibHEADERS -j$(nproc --all)
  make -C $CHOST/32/libitm DESTDIR="$pkgdir" install-nodist_toolexeclibHEADERS -j$(nproc --all)
  make -C $CHOST/32/libsanitizer DESTDIR="$pkgdir" install-nodist_{saninclude,toolexeclib}HEADERS -j$(nproc --all)
  make -C $CHOST/32/libsanitizer/asan DESTDIR="$pkgdir" install-nodist_toolexeclibHEADERS -j$(nproc --all)

  make -C libiberty DESTDIR="$pkgdir" install -j$(nproc --all)
  install -m644 libiberty/pic/libiberty.a "$pkgdir/usr/lib"

  make -C gcc DESTDIR="$pkgdir" install-man install-info -j$(nproc --all)
  rm "$pkgdir"/usr/share/man/man1/{gccgo,gfortran,gdc}.1
  rm "$pkgdir"/usr/share/info/{gccgo,gfortran,gnat-style,gnat_rm,gnat_ugn,gdc}.info

  make -C libcpp DESTDIR="$pkgdir" install -j$(nproc --all)
  make -C gcc DESTDIR="$pkgdir" install-po -j$(nproc --all)

  # many packages expect this symlink
  ln -s gcc "$pkgdir"/usr/bin/cc

  # POSIX conformance launcher scripts for c89 and c99
  install -Dm755 "$srcdir/c89" "$pkgdir/usr/bin/c89"
  install -Dm755 "$srcdir/c99" "$pkgdir/usr/bin/c99"

  # install the libstdc++ man pages
  make -C $CHOST/libstdc++-v3/doc DESTDIR="$pkgdir" doc-install-man -j$(nproc --all)

  # remove files provided by lib32-gcc-libs
  rm -f "$pkgdir"/usr/lib32/lib{stdc++,gcc_s}.so

  # byte-compile python libraries
  python -m compileall "$pkgdir/usr/share/gcc-${pkgver%%+*}/"
  python -O -m compileall "$pkgdir/usr/share/gcc-${pkgver%%+*}/"

  # Install Runtime Library Exception
  install -d "$pkgdir/usr/share/licenses/$pkgname/"
  ln -s /usr/share/licenses/gcc-libs/RUNTIME.LIBRARY.EXCEPTION \
    "$pkgdir/usr/share/licenses/$pkgname/"
}

package_gcc-fortran() {
  pkgdesc='Fortran front-end for GCC'
  depends=("gcc=$pkgver-$pkgrel")
  provides=($pkgname-multilib)
  replaces=($pkgname-multilib)

  cd gcc-build
  make -C $CHOST/libgfortran DESTDIR="$pkgdir" install-cafexeclibLTLIBRARIES \
    install-{toolexeclibDATA,nodist_fincludeHEADERS,gfor_cHEADERS} -j$(nproc --all)
  make -C $CHOST/32/libgfortran DESTDIR="$pkgdir" install-cafexeclibLTLIBRARIES \
    install-{toolexeclibDATA,nodist_fincludeHEADERS,gfor_cHEADERS} -j$(nproc --all)
  make -C $CHOST/libgomp DESTDIR="$pkgdir" install-nodist_fincludeHEADERS
  make -C gcc DESTDIR="$pkgdir" fortran.install-{common,man,info} -j$(nproc --all)
  install -Dm755 gcc/f951 "$pkgdir/${_libdir}/f951"

  ln -s gfortran "$pkgdir/usr/bin/f95"

  # Install Runtime Library Exception
  install -d "$pkgdir/usr/share/licenses/$pkgname/"
  ln -s /usr/share/licenses/gcc-libs/RUNTIME.LIBRARY.EXCEPTION \
    "$pkgdir/usr/share/licenses/$pkgname/"
}

package_gcc-objc() {
  pkgdesc='Objective-C front-end for GCC'
  depends=("gcc=$pkgver-$pkgrel")
  provides=($pkgname-multilib)
  replaces=($pkgname-multilib)

  cd gcc-build
  make DESTDIR="$pkgdir" -C $CHOST/libobjc install-headers -j$(nproc --all)
  install -dm755 "$pkgdir/${_libdir}"
  install -m755 gcc/cc1obj{,plus} "$pkgdir/${_libdir}/"

  # Install Runtime Library Exception
  install -d "$pkgdir/usr/share/licenses/$pkgname/"
  ln -s /usr/share/licenses/gcc-libs/RUNTIME.LIBRARY.EXCEPTION \
    "$pkgdir/usr/share/licenses/$pkgname/"
}

package_gcc-ada() {
  pkgdesc='Ada front-end for GCC (GNAT)'
  depends=("gcc=$pkgver-$pkgrel")
  provides=($pkgname-multilib)
  replaces=($pkgname-multilib)
  options+=(staticlibs)

  cd gcc-build/gcc
  make DESTDIR="$pkgdir" ada.install-{common,info} -j$(nproc --all)
  install -m755 gnat1 "$pkgdir/${_libdir}"

  cd "$srcdir"/gcc-build/$CHOST/libada
  make DESTDIR="${pkgdir}" INSTALL="install" \
    INSTALL_DATA="install -m644" install-libada -j$(nproc --all)

  cd "$srcdir"/gcc-build/$CHOST/32/libada
  make DESTDIR="${pkgdir}" INSTALL="install" \
    INSTALL_DATA="install -m644" install-libada -j$(nproc --all)

  ln -s gcc "$pkgdir/usr/bin/gnatgcc"

  # insist on dynamic linking, but keep static libraries because gnatmake complains
  mv "$pkgdir"/${_libdir}/adalib/libgna{rl,t}-${_majorver}.so "$pkgdir/usr/lib"
  ln -s libgnarl-${_majorver}.so "$pkgdir/usr/lib/libgnarl.so"
  ln -s libgnat-${_majorver}.so "$pkgdir/usr/lib/libgnat.so"
  rm -f "$pkgdir"/${_libdir}/adalib/libgna{rl,t}.so

  install -d "$pkgdir/usr/lib32/"
  mv "$pkgdir"/${_libdir}/32/adalib/libgna{rl,t}-${_majorver}.so "$pkgdir/usr/lib32"
  ln -s libgnarl-${_majorver}.so "$pkgdir/usr/lib32/libgnarl.so"
  ln -s libgnat-${_majorver}.so "$pkgdir/usr/lib32/libgnat.so"
  rm -f "$pkgdir"/${_libdir}/32/adalib/libgna{rl,t}.so

  # Install Runtime Library Exception
  install -d "$pkgdir/usr/share/licenses/$pkgname/"
  ln -s /usr/share/licenses/gcc-libs/RUNTIME.LIBRARY.EXCEPTION \
    "$pkgdir/usr/share/licenses/$pkgname/"
}

package_gcc-go() {
  pkgdesc='Go front-end for GCC'
  depends=("gcc=$pkgver-$pkgrel")
  provides=("go=1.12.2" $pkgname-multilib)
  replaces=($pkgname-multilib)
  conflicts=(go)

  cd gcc-build
  make -C $CHOST/libgo DESTDIR="$pkgdir" install-exec-am -j$(nproc --all)
  make -C $CHOST/32/libgo DESTDIR="$pkgdir" install-exec-am -j$(nproc --all)
  make DESTDIR="$pkgdir" install-gotools -j$(nproc --all)
  make -C gcc DESTDIR="$pkgdir" go.install-{common,man,info} -j$(nproc --all)

  rm -f "$pkgdir"/usr/lib{,32}/libgo.so*
  install -Dm755 gcc/go1 "$pkgdir/${_libdir}/go1"

  # Install Runtime Library Exception
  install -d "$pkgdir/usr/share/licenses/$pkgname/"
  ln -s /usr/share/licenses/gcc-libs/RUNTIME.LIBRARY.EXCEPTION \
    "$pkgdir/usr/share/licenses/$pkgname/"
}

package_lib32-gcc-libs() {
  pkgdesc='32-bit runtime libraries shipped by GCC'
  depends=('lib32-glibc>=2.27')
  provides=(libgo.so libgfortran.so libubsan.so libasan.so)
  groups=(multilib-devel)
  options=(!emptydirs !strip)

  cd gcc-build

  make -C $CHOST/32/libgcc DESTDIR="$pkgdir" install-shared -j$(nproc --all)
  rm -f "$pkgdir/$_libdir/32/libgcc_eh.a"

  for lib in libatomic \
             libgfortran \
             libgo \
             libgomp \
             libitm \
             libquadmath \
             libsanitizer/{a,l,ub}san \
             libstdc++-v3/src \
             libvtv; do
    make -C $CHOST/32/$lib DESTDIR="$pkgdir" install-toolexeclibLTLIBRARIES -j$(nproc --all)
  done

  make -C $CHOST/32/libobjc DESTDIR="$pkgdir" install-libs -j$(nproc --all)

  make -C $CHOST/libphobos DESTDIR="$pkgdir" install -j$(nproc --all)
  rm -f "$pkgdir"/usr/lib32/libgphobos.spec

  # remove files provided by gcc-libs
  rm -rf "$pkgdir"/usr/lib

  # Install Runtime Library Exception
  install -Dm644 "$srcdir/gcc/COPYING.RUNTIME" \
    "$pkgdir/usr/share/licenses/lib32-gcc-libs/RUNTIME.LIBRARY.EXCEPTION"
}

package_gcc-d() {
  pkgdesc="D frontend for GCC"
  depends=("gcc=$pkgver-$pkgrel")
  provides=(gdc)
  replaces=(gdc)
  options=('staticlibs')

  cd gcc-build
  make -C gcc DESTDIR="$pkgdir" d.install-{common,man,info} -j$(nproc --all)

  install -Dm755 gcc/gdc "$pkgdir"/usr/bin/gdc
  install -Dm755 gcc/d21 "$pkgdir"/"$_libdir"/d21

  make -C $CHOST/libphobos DESTDIR="$pkgdir" install -j$(nproc --all)
  rm -f "$pkgdir/usr/lib/"lib{gphobos,gdruntime}.so*
  rm -f "$pkgdir/usr/lib32/"lib{gphobos,gdruntime}.so*

  install -d "$pkgdir"/usr/include/dlang
  ln -s /"${_libdir}"/include/d "$pkgdir"/usr/include/dlang/gdc

  # Install Runtime Library Exception
  install -d "$pkgdir/usr/share/licenses/$pkgname/"
  ln -s /usr/share/licenses/gcc-libs/RUNTIME.LIBRARY.EXCEPTION \
    "$pkgdir/usr/share/licenses/$pkgname/"
}
