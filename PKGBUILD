# Maintainer: local build
pkgname=llama-cpp-cuda
pkgver=r9994.14d3ba45f3
pkgrel=1
pkgdesc="LLM inference in C/C++ with CUDA (local build)"
arch=('x86_64')
url="https://github.com/ggml-org/llama.cpp"
license=('MIT')
depends=('cuda' 'curl')
makedepends=('cmake' 'git' 'patchelf')
provides=('llama-cpp')
conflicts=('llama-cpp-cuda')
options=('!strip')
source=("${pkgname}::git+https://github.com/ggml-org/llama.cpp.git"
        "vram-gpu-schedule.patch"
#	"fix-mcp-override-fallback.patch"
    )
sha256sums=('SKIP' 'SKIP')

pkgver() {
    cd "${srcdir}/${pkgname}"
    printf "r%s.%s" "$(git rev-list --count HEAD)" "$(git rev-parse --short HEAD)"
}

prepare() {
    cd "${srcdir}/${pkgname}"
    patch -p1 < "${srcdir}/vram-gpu-schedule.patch"
#    patch -p1 < "${srcdir}/fix-mcp-override-fallback.patch"
}

build() {
    cmake -B "${srcdir}/build" -S "${srcdir}/${pkgname}" \
        -DGGML_CUDA=ON \
        -DCMAKE_CUDA_ARCHITECTURES="75;89" \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_INSTALL_PREFIX=/usr \
        -DLLAMA_BUILD_TESTS=OFF
    cmake --build "${srcdir}/build" --target llama-server -j$(nproc)
}

package() {
    local build="${srcdir}/build/bin"

    install -Dm755 "${build}/llama-server" "${pkgdir}/usr/bin/llama-server"

    install -dm755 "${pkgdir}/usr/lib"
    cp -a "${build}"/lib*.so* "${pkgdir}/usr/lib/"

    # Remove build-directory RPATH embedded by nvcc
    find "${pkgdir}/usr/lib" -name '*.so*' ! -type l \
        -exec patchelf --remove-rpath {} \;

    install -Dm644 "${srcdir}/${pkgname}/LICENSE" \
        "${pkgdir}/usr/share/licenses/${pkgname}/LICENSE"

#    install -Dm644 "${srcdir}/llama-router.service" \
#        "${pkgdir}/usr/lib/systemd/system/llama-router.service"
#    install -Dm644 "${srcdir}/llama-router.sysusers" \
#        "${pkgdir}/usr/lib/sysusers.d/llama-router.conf"
#    install -Dm644 "${srcdir}/llama-router.tmpfiles" \
#        "${pkgdir}/usr/lib/tmpfiles.d/llama-router.conf"
#    install -Dm644 "${srcdir}/llama-router.hook" \
#        "${pkgdir}/usr/share/libalpm/hooks/99-llama-router-restart.hook"
}
