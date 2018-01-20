# $Id$
# Maintainer: Tobias Powalowski <tpowa@archlinux.org>
# Maintainer: Thomas Baechler <thomas@archlinux.org>

pkgbase=linux-ryzen
_srcname=linux-4.14
pkgver=4.14.14
pkgrel=1
arch=('x86_64')
url="https://www.kernel.org/"
license=('GPL2')
makedepends=('xmlto' 'kmod' 'inetutils' 'bc' 'libelf')
options=('!strip')
source=(
  "https://www.kernel.org/pub/linux/kernel/v4.x/${_srcname}.tar.xz"
  "https://www.kernel.org/pub/linux/kernel/v4.x/${_srcname}.tar.sign"
  "https://www.kernel.org/pub/linux/kernel/v4.x/patch-${pkgver}.xz"
  "https://www.kernel.org/pub/linux/kernel/v4.x/patch-${pkgver}.sign"
  'config'         # the main kernel config file
  '60-linux.hook'  # pacman hook for depmod
  '90-linux.hook'  # pacman hook for initramfs regeneration
  'linux.preset'   # standard config files for mkinitcpio ramdisk
  0001-add-sysctl-to-disallow-unprivileged-CLONE_NEWUSER-by.patch
  0002-dccp-CVE-2017-8824-use-after-free-in-DCCP-code.patch
  0003-xfrm-Fix-stack-out-of-bounds-read-on-socket-policy-l.patch
  0004-drm-i915-edp-Only-use-the-alternate-fixed-mode-if-it.patch
  'ubuntu-unprivileged-overlayfs.patch'
  # vfio patches
  'i915-vga-arbiter.patch'
  'add-acs-overrides.patch'
  # amd patches
  'k10temp-0001.patch'
  'k10temp-0002.patch'
  'k10temp-0003.patch'
  'nct6776-fan6.patch'
  'k10smt.patch'
  'graysky-gcctunes-4.14.patch::https://github.com/pfactum/pf-kernel/commit/d21c4ae7a45f2adbf27992f59389b948428b6c15.patch'
)

validpgpkeys=(
  'ABAF11C65A2970B130ABE3C479BE3E4300411886'  # Linus Torvalds
  '647F28654894E3BD457199BE38DBBDC86092693E'  # Greg Kroah-Hartman
)

_kernelname=${pkgbase#linux}

prepare() {
  cd ${_srcname}

  # add upstream patch
  patch -p1 -i ../patch-${pkgver}
  chmod +x tools/objtool/sync-check.sh  # GNU patch doesn't support git-style file mode

  # amd patches
  patch -Np1 -i ../k10temp-0001.patch
  patch -Np1 -i ../k10temp-0002.patch
  patch -Np1 -i ../k10temp-0003.patch
  patch -Np1 -i ../k10smt.patch
  patch -Np1 -i ../nct6776-fan6.patch
  patch -Np1 -i ../graysky-gcctunes-4.14.patch

  # vfio patches
  patch -Np1 -i ../i915-vga-arbiter.patch
  patch -Np1 -i ../add-acs-overrides.patch

  # security patches

  # add latest fixes from stable queue, if needed
  # http://git.kernel.org/?p=linux/kernel/git/stable/stable-queue.git

  # disable USER_NS for non-root users by default
  patch -Np1 -i ../0001-add-sysctl-to-disallow-unprivileged-CLONE_NEWUSER-by.patch
  patch -Np1 -i ../ubuntu-unprivileged-overlayfs.patch

  # https://nvd.nist.gov/vuln/detail/CVE-2017-8824
  patch -Np1 -i ../0002-dccp-CVE-2017-8824-use-after-free-in-DCCP-code.patch

  # https://bugs.archlinux.org/task/56605
  patch -Np1 -i ../0003-xfrm-Fix-stack-out-of-bounds-read-on-socket-policy-l.patch

  # https://bugs.archlinux.org/task/56711
  patch -Np1 -i ../0004-drm-i915-edp-Only-use-the-alternate-fixed-mode-if-it.patch

  cp -Tf ../config .config

  if [ "${_kernelname}" != "" ]; then
    sed -i "s|CONFIG_LOCALVERSION=.*|CONFIG_LOCALVERSION=\"${_kernelname}\"|g" ./.config
    sed -i "s|CONFIG_LOCALVERSION_AUTO=.*|CONFIG_LOCALVERSION_AUTO=n|" ./.config
  fi

  # set extraversion to pkgrel
  sed -ri "s|^(EXTRAVERSION =).*|\1 -${pkgrel}|" Makefile

  # don't run depmod on 'make install'. We'll do this ourselves in packaging
  sed -i '2iexit 0' scripts/depmod.sh

  # get kernel version
  make prepare

  # load configuration
  # Configure the kernel. Replace the line below with one of your choice.
  #make menuconfig # CLI menu for configuration
  #make nconfig # new CLI menu for configuration
  #make xconfig # X-based configuration
  #make oldconfig # using old config from previous kernel version
  # ... or manually edit .config

  # rewrite configuration
  yes "" | make config >/dev/null
}

build() {
  cd ${_srcname}

  make ${MAKEFLAGS} LOCALVERSION= bzImage modules
}

_package() {
  pkgdesc="The ${pkgbase/linux/Linux} kernel and modules with AMD Ryzen temperature monitoring fixes"
  [ "${pkgbase}" = "linux" ] && groups=('base')
  depends=('coreutils' 'linux-firmware' 'kmod' 'mkinitcpio>=0.7')
  optdepends=('crda: to set the correct wireless channels of your country')
  backup=("etc/mkinitcpio.d/${pkgbase}.preset")
  install=linux.install

  cd ${_srcname}

  # get kernel version
  _kernver="$(make LOCALVERSION= kernelrelease)"
  _basekernel=${_kernver%%-*}
  _basekernel=${_basekernel%.*}

  mkdir -p "${pkgdir}"/{boot,usr/lib/modules}
  make LOCALVERSION= INSTALL_MOD_PATH="${pkgdir}/usr" modules_install
  cp arch/x86/boot/bzImage "${pkgdir}/boot/vmlinuz-${pkgbase}"

  # make room for external modules
  local _extramodules="extramodules-${_basekernel}${_kernelname:--ARCH}"
  ln -s "../${_extramodules}" "${pkgdir}/usr/lib/modules/${_kernver}/extramodules"

  # add real version for building modules and running depmod from hook
  echo "${_kernver}" |
    install -Dm644 /dev/stdin "${pkgdir}/usr/lib/modules/${_extramodules}/version"

  # remove build and source links
  rm "${pkgdir}"/usr/lib/modules/${_kernver}/{source,build}

  # now we call depmod...
  depmod -b "${pkgdir}/usr" -F System.map "${_kernver}"

  # add vmlinux
  install -Dt "${pkgdir}/usr/lib/modules/${_kernver}/build" -m644 vmlinux

  # sed expression for following substitutions
  local _subst="
    s|%PKGBASE%|${pkgbase}|g
    s|%KERNVER%|${_kernver}|g
    s|%EXTRAMODULES%|${_extramodules}|g
  "

  # hack to allow specifying an initially nonexisting install file
  sed "${_subst}" "${startdir}/${install}" > "${startdir}/${install}.pkg"
  true && install=${install}.pkg

  # install mkinitcpio preset file
  sed "${_subst}" ../linux.preset |
    install -Dm644 /dev/stdin "${pkgdir}/etc/mkinitcpio.d/${pkgbase}.preset"

  # install pacman hooks
  sed "${_subst}" ../60-linux.hook |
    install -Dm644 /dev/stdin "${pkgdir}/usr/share/libalpm/hooks/60-${pkgbase}.hook"
  sed "${_subst}" ../90-linux.hook |
    install -Dm644 /dev/stdin "${pkgdir}/usr/share/libalpm/hooks/90-${pkgbase}.hook"
}

_package-headers() {
  pkgdesc="Header files and scripts for building modules for ${pkgbase/linux/Linux} kernel"

  cd ${_srcname}
  local _builddir="${pkgdir}/usr/lib/modules/${_kernver}/build"

  install -Dt "${_builddir}" -m644 Makefile .config Module.symvers
  install -Dt "${_builddir}/kernel" -m644 kernel/Makefile

  mkdir "${_builddir}/.tmp_versions"

  cp -t "${_builddir}" -a include scripts

  install -Dt "${_builddir}/arch/x86" -m644 arch/x86/Makefile
  install -Dt "${_builddir}/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s

  cp -t "${_builddir}/arch/x86" -a arch/x86/include

  install -Dt "${_builddir}/drivers/md" -m644 drivers/md/*.h
  install -Dt "${_builddir}/net/mac80211" -m644 net/mac80211/*.h

  # http://bugs.archlinux.org/task/9912
  install -Dt "${_builddir}/drivers/media/dvb-core" -m644 drivers/media/dvb-core/*.h

  # http://bugs.archlinux.org/task/13146
  install -Dt "${_builddir}/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # http://bugs.archlinux.org/task/20402
  install -Dt "${_builddir}/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "${_builddir}/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "${_builddir}/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # add xfs and shmem for aufs building
  mkdir -p "${_builddir}"/{fs/xfs,mm}

  # copy in Kconfig files
  find . -name Kconfig\* -exec install -Dm644 {} "${_builddir}/{}" \;

  # add objtool for external module building and enabled VALIDATION_STACK option
  install -Dt "${_builddir}/tools/objtool" tools/objtool/objtool

  # remove unneeded architectures
  local _arch
  for _arch in "${_builddir}"/arch/*/; do
    [[ ${_arch} == */x86/ ]] && continue
    rm -r "${_arch}"
  done

  # remove files already in linux-docs package
  rm -r "${_builddir}/Documentation"

  # remove now broken symlinks
  find -L "${_builddir}" -type l -printf 'Removing %P\n' -delete

  # Fix permissions
  chmod -R u=rwX,go=rX "${_builddir}"

  # strip scripts directory
  local _binary _strip
  while read -rd '' _binary; do
    case "$(file -bi "${_binary}")" in
      *application/x-sharedlib*)  _strip="${STRIP_SHARED}"   ;; # Libraries (.so)
      *application/x-archive*)    _strip="${STRIP_STATIC}"   ;; # Libraries (.a)
      *application/x-executable*) _strip="${STRIP_BINARIES}" ;; # Binaries
      *) continue ;;
    esac
    /usr/bin/strip ${_strip} "${_binary}"
  done < <(find "${_builddir}/scripts" -type f -perm -u+w -print0 2>/dev/null)
}

_package-docs() {
  pkgdesc="Kernel hackers manual - HTML documentation that comes with the ${pkgbase/linux/Linux} kernel"

  cd ${_srcname}
  local _builddir="${pkgdir}/usr/lib/modules/${_kernver}/build"

  mkdir -p "${_builddir}"
  cp -t "${_builddir}" -a Documentation

  # Fix permissions
  chmod -R u=rwX,go=rX "${_builddir}"
}

pkgname=("${pkgbase}" "${pkgbase}-headers")
for _p in ${pkgname[@]}; do
  eval "package_${_p}() {
    $(declare -f "_package${_p#${pkgbase}}")
    _package${_p#${pkgbase}}
  }"
done

# vim:set ts=8 sts=2 sw=2 et:

# makepkg -g >> PKGBUILD
sha256sums=('f81d59477e90a130857ce18dc02f4fbe5725854911db1e7ba770c7cd350f96a7'
            'SKIP'
            '62d656b98f0dc143216cb9650bd9b96cd83d92925731e9f0bec5eb4d6358e603'
            'SKIP'
            'c2ffd12ffc0e7905968d0e75da9cd204ef36f27bdbbd4d876e2c0bc7e983c3e2'
            'ae2e95db94ef7176207c690224169594d49445e04249d2499e9d2fbc117a0b21'
            '75f99f5239e03238f88d1a834c50043ec32b1dc568f2cc291b07d04718483919'
            'e1172898719b095861d7e8353977524741db5e9f4aa191ae7502a98d6cefbfa7'
            '36b1118c8dedadc4851150ddd4eb07b1c58ac5bbf3022cc2501a27c2b476da98'
            '5694022613bb49a77d3dfafdd2e635e9015e0a9069c58a07e99bdc5df6520311'
            '2f46093fde72eabc0fd25eff5065d780619fc5e7d2143d048877a8220d6291b0'
            '6364edabad4182dcf148ae7c14d8f45d61037d4539e76486f978f1af3a090794'
            '01a6d59a55df1040127ced0412f44313b65356e3c680980210593ee43f2495aa'
            'eaf70cd805cdb43cf6227d354a6d54f67645b6df99e06136a8055d7494d7439c'
            'c238969a3c3a44b41c868a883880d8c4dc475e457427e91c649e9f24170b2c7d'
            'd14bc7f688ee639073a3a16743df642f424070d06d59aed2c00cd6b5de1d3b9b'
            'bccc916758d03eacd50aaebba2b734e3faa1c693ae6df10f847c64d501eee026'
            '9fb3a3938a41ee7cb5ea09c70277d901824f9c4c6e618bdf3a2579c01d109aa5'
            '093c30926bf9e93491ca43878b8a19d2b39a34b8bc1a9f89b2004dd1671923f8'
            '2f9d78a2573ace056b603252dd681f1bed3d1c5533de4524b9b7c0e4181f3d25'
            '642ce1979679fb80f8e661f2876f495d7c9345fa664cf602b4bbc15754e81222')
