# Maintainer : bartus <arch-user-repoᘓbartus.33mail.com>
# shellcheck disable=SC2034

#to enforce cuda verison uncomment this line and update value of sm_xx model accordingly
#_cuda_capability+=(sm_30 sm_35 sm_37)
#_cuda_capability+=(sm_50 sm_52 sm_60 sm_61 sm_70 sm_75)
((TRAVIS)) && _cuda_capability+=(sm_50 sm_52 sm_60 sm_61 sm_70 sm_75) # suppress 3.x to prevent Travis build exceed time limit.

#some extra, not officially supported stuff goes here:
_EXTRAOPTS+=( -DWITH_ALEMBIC_HDF5=ON
              -DWITH_CYCLES_EMBREE=ON
              -DWITH_USD=ON
              -DUSD_ROOT=/usr
            )

pkgname=blender-2.8-git
_fragment="#branch=master"
pkgver=2.83.r93807.g78383f7a9f9
pkgrel=1
pkgdesc="Development version of Blender 2.8 branch"
arch=('i686' 'x86_64')
url="https://blender.org/"
depends=('alembic' 'libgl' 'python' 'python-numpy' 'openjpeg' 'desktop-file-utils' 'hicolor-icon-theme'
         'ffmpeg' 'fftw' 'openal' 'freetype2' 'libxi' 'openimageio' 'opencolorio' 'openimagedenoise'
         'openvdb' 'opencollada' 'opensubdiv' 'openshadinglanguage' 'libtiff' 'libpng' 'embree' 'usd=19.11')
makedepends=('git' 'cmake' 'boost' 'mesa' 'llvm')
((DISABLE_NINJA)) ||  makedepends+=('ninja')
((DISABLE_CUDA)) && optdepends=('cuda: CUDA support in Cycles') || { makedepends+=('cuda') ; ((DISABLE_OPTIX)) || makedepends+=('optix>=7.0'); }
provides=("blender-${pkgver%%.r*}")
conflicts=("blender-${pkgver%%.r*}")
license=('GPL')
install=blender.install
# NOTE: the source array has to be kept in sync with .gitmodules
# the submodules has to be stored in path ending with git to match
# the path in .gitmodules.
# More info:
#   http://wiki.blender.org/index.php/Dev:Doc/Tools/Git
source=("git://git.blender.org/blender.git${_fragment}"
        'blender-addons.git::git://git.blender.org/blender-addons.git'
        'blender-addons-contrib.git::git://git.blender.org/blender-addons-contrib.git'
        'blender-translations.git::git://git.blender.org/blender-translations.git'
        'blender-dev-tools.git::git://git.blender.org/blender-dev-tools.git'
        blender-2.8.desktop
        SelectCudaComputeArch.patch
        usd_python.patch #add missing python headers when building against python enabled usd.
        embree.patch #add missing embree link.
        )
sha256sums=('SKIP'
            'SKIP'
            'SKIP'
            'SKIP'
            'SKIP'
            '9d7bd988939f4e8d06adc3166bacd76cc87a488b40f042f033863af35eadfc43'
            '66b9bf3db441f35119ef0eb5f855142f2e773e8002ac0216e056bcc6f8ac409c'
            '893b127c9e0ea1a67905434f729b45a993c58a7ea954f9f89480ad1cc0578849'
            '42afe119529a5350034a489225958112bf4b84bdee38757a932e5caaa9bd5ed4')

pkgver() {
  blender_version=$(grep -Po "BLENDER_VERSION \K[0-9]{3}" "$srcdir"/blender/source/blender/blenkernel/BKE_blender_version.h)
  printf "%d.%d.r%s.g%s" \
    $((blender_version/100)) \
    $((blender_version%100)) \
    "$(git -C "$srcdir/blender" rev-list --count HEAD)" \
    "$(git -C "$srcdir/blender" rev-parse --short HEAD)"
}

prepare() {
  cd "$srcdir/blender"
  # update the submodules
  git submodule update --init --recursive --remote
  if [ -z "$_cuda_capability" ] && grep -q nvidia <(lsmod); then 
    git apply -v ${srcdir}/SelectCudaComputeArch.patch
  fi
  git apply -v ${srcdir}/usd_python.patch
  git apply -v ${srcdir}/embree.patch
}

build() {
  mkdir -p "$srcdir/blender-build"
  cd "$srcdir/blender-build"
  
  _pyver=$(python -c "from sys import version_info; print(\"%d.%d\" % (version_info[0],version_info[1]))")
  msg "python version detected: ${_pyver}"

  # determine whether we can precompile CUDA kernels
  _CUDA_PKG=`pacman -Qq cuda 2>/dev/null` || true
  if [ "$_CUDA_PKG" != "" ] && ! ((DISABLE_CUDA)) ; then
      _EXTRAOPTS+=( -DWITH_CYCLES_CUDA_BINARIES=ON
                    -DCUDA_TOOLKIT_ROOT_DIR=/opt/cuda
                    -DOPTIX_ROOT_DIR=/opt/optix )
      if [ -v _cuda_capability ]; then
        _EXTRAOPTS+=(-DCYCLES_CUDA_BINARIES_ARCH=$(IFS=';'; echo "${_cuda_capability[*]}";))
      fi
      [ -f "/usr/lib/ccache/bin/nvcc-ccache" ] && _CMAKE_FLAGS+=( -DCUDA_NVCC_EXECUTABLE=/usr/lib/ccache/bin/nvcc-ccache )
      if _cuda_gcc=$(basename $(readlink /opt/cuda/bin/gcc)) ; then
        [ -L "/usr/lib/ccache/bin/$_cuda_gcc" ] && _CMAKE_FLAGS+=( -DCUDA_HOST_COMPILER=/usr/lib/ccache/bin/$_cuda_gcc )
      fi
  fi

  ((DISABLE_NINJA)) && generator="Unix Makefiles" || generator="Ninja"
  cmake -G "$generator" "$srcdir/blender" \
        -C${srcdir}/blender/build_files/cmake/config/blender_release.cmake \
        -DCMAKE_INSTALL_PREFIX=/usr \
        -DCMAKE_BUILD_TYPE=Release \
        -DWITH_INSTALL_PORTABLE=OFF \
        -DWITH_SYSTEM_GLEW=ON \
        -DWITH_PYTHON_INSTALL=OFF \
        -DPYTHON_VERSION=${_pyver} \
        ${_EXTRAOPTS[@]}
  export NINJA_STATUS="[%p | %f<%r<%u | %cbps ] "
  ((DISABLE_NINJA)) && make || ninja $([ -v MAKEFLAGS ] || echo -j1) -d stats
}

package() {
  cd "$srcdir/blender-build"
  ((DISABLE_NINJA)) && make install DESTDIR="$pkgdir" || DESTDIR="$pkgdir" ninja install
  
  msg "add -${pkgver%%.r*} sufix to desktop shortcut"
  sed -i "s/=blender/=blender-${pkgver%%.r*}/g" ${pkgdir}/usr/share/applications/blender.desktop
  sed -i "s/=Blender/=Blender-${pkgver%%.r*}/g" ${pkgdir}/usr/share/applications/blender.desktop
  mv ${pkgdir}/usr/share/applications/blender.desktop ${pkgdir}/usr/share/applications/blender-${pkgver%%.r*}.desktop

  msg "add -${pkgver%%.r*} sufix to binaries"
  mv ${pkgdir}/usr/bin/blender ${pkgdir}/usr/bin/blender-${pkgver%%.r*}
  mv ${pkgdir}/usr/bin/blender-thumbnailer.py ${pkgdir}/usr/bin/blender-${pkgver%%.r*}-thumbnailer.py
#  mv ${pkgdir}/usr/bin/blenderplayer ${pkgdir}/usr/bin/blenderplayer-${pkgver%%.r*}

  msg "mv doc/blender to doc/blender-${pkgver%%.r*}"
  mv ${pkgdir}/usr/share/doc/blender ${pkgdir}/usr/share/doc/blender-${pkgver%%.r*}

  msg "add -${pkgver%%.r*} sufix to man page"
  mv ${pkgdir}/usr/share/man/man1/blender.1 ${pkgdir}/usr/share/man/man1/blender-${pkgver%%.r*}

  msg "add -${pkgver%%.r*} sufix to all icons"  
  for icon in `find ${pkgdir}/usr/share/icons -type f`
  do
    # ${filename##/*.} extra extenssion from path
    # ${filename%.*} extract filename form path
    # look at bash "manipulatin string" 
    mv $icon ${icon%.*}-${pkgver%%.r*}.${icon##/*.}
  done

  if [ -e "$pkgdir"/usr/share/blender/*/scripts/addons/cycles/lib/ ] ; then
    # make sure the cuda kernels are not stripped
    chmod 444 "$pkgdir"/usr/share/blender/*/scripts/addons/cycles/lib/*
  fi
}
# vim:set sw=2 ts=2 et:
