# $Id$
# Maintainer: Allan McRae <allan@archlinux.org>

# toolchain build order: linux-api-headers->glibc->binutils->gcc->binutils->glibc
# NOTE: libtool requires rebuilt with each new gcc version

pkgname=('gcc' 'gcc-libs')
pkgver=10.0.0
_pkgver=1
_islver=0.21
pkgrel=`date +%Y%m%d`
_snapshot=10-20191006
pkgdesc="The GNU Compiler Collection"
arch=('x86_64')
license=('GPL' 'LGPL' 'FDL' 'custom')
url="http://gcc.gnu.org"
makedepends=('binutils>=2.26' 'libmpc' 'doxygen')
checkdepends=('dejagnu' 'inetutils')
options=('!emptydirs')
source=(#ftp://gcc.gnu.org/pub/gcc/releases/gcc-${pkgver}/gcc-${pkgver}.tar.bz2
ftp://gcc.gnu.org/pub/gcc/snapshots/${_snapshot}/gcc-${_snapshot}.tar.xz
http://isl.gforge.inria.fr/isl-${_islver}.tar.bz2)
md5sums=('SKIP'
'SKIP')

# gcc-6.0 forces a changed triplet - need to address in pacman/devtools
[[ $CARCH == "x86_64" ]] && CHOST=x86_64-pc-linux-gnu

if [ -n "${_snapshot}" ]; then
	_basedir=gcc-${_snapshot}
else
	_basedir=gcc-${pkgver}
fi

_libdir="usr/lib/gcc/$CHOST/$pkgver"

prepare() {
  cp ../c89 .
  cp ../c99 .

  [[ ! -d gcc ]] && ln -s gcc-${_snapshot/+/-} gcc
  cd gcc

  # link isl for in-tree build
  ln -s ../isl-${_islver} isl

  # Do not run fixincludes
  sed -i 's@\./fixinc\.sh@-c true@' gcc/Makefile.in

  # Arch Linux installs x86_64 libraries /lib
  sed -i '/m64=/s/lib64/lib/' gcc/config/i386/t-linux64

  # hack! - some configure tests for header files using "$CPP $CPPFLAGS"
  sed -i "/ac_cpp=/s/\$CPPFLAGS/\$CPPFLAGS -O2/" {libiberty,gcc}/configure

  mkdir -p "$srcdir/gcc-build"
}

build() {
	cd ${srcdir}/gcc-build

	# using -pipe causes spurious test-suite failures
	# http://gcc.gnu.org/bugzilla/show_bug.cgi?id=48565
	CFLAGS=${CFLAGS/-pipe/}
	CXXFLAGS=${CXXFLAGS/-pipe/}

	CFLAGS="$CFLAGS -O2 -D_FORTIFY_SOURCE=2"
	CPPFLAGS="$CPPFLAGS -O2 -D_FORTIFY_SOURCE=2"
	CXXFLAGS="$CXXFLAGS -O2" # -D_FORTIFY_SOURCE=2"

	${srcdir}/${_basedir}/configure --prefix=/usr \
		--libdir=/usr/lib \
		--libexecdir=/usr/lib \
		--mandir=/usr/share/man \
		--infodir=/usr/share/info \
		--with-bugurl=https://bugs.archlinux.org/ \
		--enable-languages=c,c++,lto \
		--enable-shared \
		--enable-threads=posix \
		--with-system-zlib \
		--with-isl \
		--enable-__cxa_atexit \
		--disable-libunwind-exceptions \
		--enable-clocale=gnu \
		--disable-libstdcxx-pch \
		--disable-libssp \
		--enable-gnu-unique-object \
		--enable-linker-build-id \
		--enable-lto \
		--enable-plugin \
		--enable-install-libiberty \
		--with-linker-hash-style=gnu \
		--enable-gnu-indirect-function \
		--disable-multilib \
		--disable-werror \
		--enable-checking=release \
		--enable-default-pie \
		--enable-default-ssp \
		--enable-cet=auto \
		--with-arch=corei7 \
		--with-cpu=corei7

	make -j8

	# make -j8 documentation
	make -j8 -C $CHOST/libstdc++-v3/doc doc-man-doxygen
}

package_gcc-libs() {
  pkgdesc='Runtime libraries shipped by GCC'
  groups=(base)
  depends=('glibc>=2.27')
  options+=(!strip)
  provides=(libubsan.so libasan.so libtsan.so liblsan.so)

  cd gcc-build
  make -j8 -C $CHOST/libgcc DESTDIR="$pkgdir" install-shared
  rm -f "$pkgdir/$_libdir/libgcc_eh.a"

  for lib in libatomic \
             libgomp \
             libitm \
             libquadmath \
             libsanitizer/{a,l,ub,t}san \
             libstdc++-v3/src \
             libvtv; do
    make -j8 -C $CHOST/$lib DESTDIR="$pkgdir" install-toolexeclibLTLIBRARIES
  done

  make -j8 -C $CHOST/libstdc++-v3/po DESTDIR="$pkgdir" install

  for lib in libgomp \
             libitm \
             libquadmath; do
    make -j8 -C $CHOST/$lib DESTDIR="$pkgdir" install-info
  done

  # Install Runtime Library Exception
  install -Dm644 "$srcdir/gcc/COPYING.RUNTIME" \
    "$pkgdir/usr/share/licenses/gcc-libs/RUNTIME.LIBRARY.EXCEPTION"
}

package_gcc() {
  pkgdesc="The GNU Compiler Collection - C and C++ frontends"
  depends=("gcc-libs=$pkgver-$pkgrel" 'binutils>=2.28' libmpc)
  groups=('base-devel')
  options+=('staticlibs')

  cd gcc-build

  make -j8 -C gcc DESTDIR="$pkgdir" install-driver install-cpp install-gcc-ar \
    c++.install-common install-headers install-plugin install-lto-wrapper

  install -m755 -t "$pkgdir/usr/bin/" gcc/gcov{,-tool}
  install -m755 -t "$pkgdir/${_libdir}/" gcc/{cc1,cc1plus,collect2,lto1}

  make -j8 -C $CHOST/libgcc DESTDIR="$pkgdir" install

  make -j8 -C $CHOST/libstdc++-v3/src DESTDIR="$pkgdir" install
  make -j8 -C $CHOST/libstdc++-v3/include DESTDIR="$pkgdir" install
  make -j8 -C $CHOST/libstdc++-v3/libsupc++ DESTDIR="$pkgdir" install
  make -j8 -C $CHOST/libstdc++-v3/python DESTDIR="$pkgdir" install

  make DESTDIR="$pkgdir" install-libcc1
  install -d "$pkgdir/usr/share/gdb/auto-load/usr/lib"
  mv "$pkgdir"/usr/lib/libstdc++.so.6.*-gdb.py \
    "$pkgdir/usr/share/gdb/auto-load/usr/lib/"

  make DESTDIR="$pkgdir" install-fixincludes
  make -j8 -C gcc DESTDIR="$pkgdir" install-mkheaders

  make -j8 -C lto-plugin DESTDIR="$pkgdir" install
  install -dm755 "$pkgdir"/usr/lib/bfd-plugins/
  ln -s /${_libdir}/liblto_plugin.so \
    "$pkgdir/usr/lib/bfd-plugins/"

  make -j8 -C $CHOST/libgomp DESTDIR="$pkgdir" install-nodist_{libsubinclude,toolexeclib}HEADERS
  make -j8 -C $CHOST/libitm DESTDIR="$pkgdir" install-nodist_toolexeclibHEADERS
  make -j8 -C $CHOST/libquadmath DESTDIR="$pkgdir" install-nodist_libsubincludeHEADERS
  make -j8 -C $CHOST/libsanitizer DESTDIR="$pkgdir" install-nodist_{saninclude,toolexeclib}HEADERS
  make -j8 -C $CHOST/libsanitizer/asan DESTDIR="$pkgdir" install-nodist_toolexeclibHEADERS
  make -j8 -C $CHOST/libsanitizer/tsan DESTDIR="$pkgdir" install-nodist_toolexeclibHEADERS
  make -j8 -C $CHOST/libsanitizer/lsan DESTDIR="$pkgdir" install-nodist_toolexeclibHEADERS

  make -j8 -C libiberty DESTDIR="$pkgdir" install
  install -m644 libiberty/pic/libiberty.a "$pkgdir/usr/lib"

  make -j8 -C gcc DESTDIR="$pkgdir" install-man install-info

  make -j8 -C libcpp DESTDIR="$pkgdir" install
  make -j8 -C gcc DESTDIR="$pkgdir" install-po

  # many packages expect this symlink
  ln -s gcc "$pkgdir"/usr/bin/cc

  # POSIX conformance launcher scripts for c89 and c99
  install -Dm755 "$srcdir/c89" "$pkgdir/usr/bin/c89"
  install -Dm755 "$srcdir/c99" "$pkgdir/usr/bin/c99"

  # install the libstdc++ man pages
  make -j8 -C $CHOST/libstdc++-v3/doc DESTDIR="$pkgdir" doc-install-man

  # byte-compile python libraries
  python -m compileall "$pkgdir/usr/share/gcc-${pkgver%%+*}/"
  python -O -m compileall "$pkgdir/usr/share/gcc-${pkgver%%+*}/"

  # Install Runtime Library Exception
  install -d "$pkgdir/usr/share/licenses/$pkgname/"
  ln -s /usr/share/licenses/gcc-libs/RUNTIME.LIBRARY.EXCEPTION \
    "$pkgdir/usr/share/licenses/$pkgname/"
}
