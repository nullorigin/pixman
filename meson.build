# Copyright © 2018 Intel Corporation

# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:

# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.

# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

project(
  'pixman',
  ['c'],
  version : '0.42.3',
  license : 'MIT',
  meson_version : '>= 0.52.0',
  default_options : ['c_std=gnu99', 'buildtype=debugoptimized'],
)

config = configuration_data()
cc = meson.get_compiler('c')
null_dep = dependency('', required : false)

add_project_arguments(
  cc.get_supported_arguments([
    '-Wdeclaration-after-statement',
    '-fno-strict-aliasing',
    '-fvisibility=hidden',
    '-Wundef',
    # -ftrapping-math is the default for gcc, but -fno-trapping-math is the
    # default for clang.  The FLOAT_IS_ZERO macro is used to guard against
    # floating-point exceptions, however with -fno-trapping-math, the compiler
    # can reorder floating-point operations so that they occur before the guard.
    # Note, this function is ignored in clang < 10.0.0.
    '-ftrapping-math'
  ]),
  language : ['c']
)

# GCC and Clang both ignore -Wno options that they don't recognize, so test for
# -W<opt>, then add -Wno-<opt> if it's ignored
foreach opt : ['unused-local-typedefs']
  if cc.has_argument('-W' + opt)
    add_project_arguments(['-Wno-' + opt], language : ['c'])
  endif
endforeach


use_mmx = get_option('mmx')
have_mmx = false
mmx_flags = []

if cc.get_id() == 'msvc'
  mmx_flags = ['/w14710', '/w14714', '/wd4244']
elif cc.get_id() == 'sun'
  mmx_flags = ['-xarch=sse']
else
  mmx_flags = ['-mmmx', '-Winline']
endif
if not use_mmx.disabled()
  if host_machine.cpu_family() == 'x86_64' or cc.get_id() == 'msvc'
    have_mmx = true
  elif host_machine.cpu_family() == 'x86' and cc.compiles('''
      #include <mmintrin.h>
      #include <stdint.h>

      /* Check support for block expressions */
      #define _mm_shuffle_pi16(A, N)                    \
        ({                                              \
        __m64 ret;                                      \
                                                        \
        /* Some versions of clang will choke on K */    \
        asm ("pshufw %2, %1, %0\n\t"                    \
             : "=y" (ret)                               \
             : "y" (A), "K" ((const int8_t)N)           \
        );                                              \
                                                        \
        ret;                                            \
        })

      int main () {
          __m64 v = _mm_cvtsi32_si64 (1);
          __m64 w;

          w = _mm_shuffle_pi16(v, 5);

          /* Some versions of clang will choke on this */
          asm ("pmulhuw %1, %0\n\t"
               : "+y" (w)
               : "y" (v)
          );

          return _mm_cvtsi64_si32 (v);
      }''',
      args : mmx_flags,
      name : 'MMX Intrinsic Support')
    have_mmx = true
  endif
endif

if have_mmx
  # Inline assembly do not work on X64 MSVC, so we use
  # compatibility intrinsics there
  if cc.get_id() != 'msvc' or host_machine.cpu_family() != 'x86_64'
    config.set10('USE_X86_MMX', true)
  endif
elif use_mmx.enabled()
  error('MMX Support unavailable, but required')
endif

use_sse2 = get_option('sse2')
have_sse2 = false
sse2_flags = []
if cc.get_id() == 'sun'
  sse2_flags = ['-xarch=sse2']
elif cc.get_id() != 'msvc'
  sse2_flags = ['-msse2', '-Winline']
endif
if not use_sse2.disabled()
  if host_machine.cpu_family() == 'x86'
    if cc.compiles('''
        #if defined(__GNUC__) && (__GNUC__ < 4 || (__GNUC__ == 4 && __GNUC_MINOR__ < 2))
        #   if !defined(__amd64__) && !defined(__x86_64__)
        #      error "Need GCC >= 4.2 for SSE2 intrinsics on x86"
        #   endif
        #endif
        #include <mmintrin.h>
        #include <xmmintrin.h>
        #include <emmintrin.h>
        int param;
        int main () {
          __m128i a = _mm_set1_epi32 (param), b = _mm_set1_epi32 (param + 1), c;
          c = _mm_xor_si128 (a, b);
          return _mm_cvtsi128_si32(c);
        }''',
        args : sse2_flags,
        name : 'SSE2 Intrinsic Support')
      have_sse2 = true
    endif
  elif host_machine.cpu_family() == 'x86_64'
    have_sse2 = true
  endif
endif

if have_sse2
  config.set10('USE_SSE2', true)
elif use_sse2.enabled()
  error('sse2 Support unavailable, but required')
endif

use_ssse3 = get_option('ssse3')
have_ssse3 = false
ssse3_flags = []
if cc.get_id() != 'msvc'
  ssse3_flags = ['-mssse3', '-Winline']
endif

# x64 pre-2010 MSVC compilers crashes when building the ssse3 code
if not use_ssse3.disabled() and not (cc.get_id() == 'msvc' and cc.version().version_compare('<16') and host_machine.cpu_family() == 'x86_64')
  if host_machine.cpu_family().startswith('x86')
    if cc.compiles('''
        #include <mmintrin.h>
        #include <xmmintrin.h>
        #include <emmintrin.h>
        int param;
        int main () {
          __m128i a = _mm_set1_epi32 (param), b = _mm_set1_epi32 (param + 1), c;
          c = _mm_xor_si128 (a, b);
          return _mm_cvtsi128_si32(c);
        }''',
        args : ssse3_flags,
        name : 'SSSE3 Intrinsic Support')
      have_ssse3 = true
    endif
  endif
endif

if have_ssse3
  config.set10('USE_SSSE3', true)
elif use_ssse3.enabled()
  error('ssse3 Support unavailable, but required')
endif

if cc.compiles('''
    __asm__ (
    ".func meson_test"
    ".endfunc"
    );''',
    name : 'test for ASM .func directive')
    config.set('ASM_HAVE_FUNC_DIRECTIVE', 1)
endif


if cc.links('''
    #include <stdint.h>

    __asm__ (
        "   .global _testlabel\n"
        "_testlabel:\n"
    );

    int testlabel();
    int main(int argc, char* argv[]) {
        return testlabel();
    }''',
    name : 'test for ASM leading underscore')
    config.set('ASM_LEADING_UNDERSCORE', 1)
endif

use_gnu_asm = get_option('gnu-inline-asm')
if not use_gnu_asm.disabled()
  if cc.compiles('''
      int main () {
        /* Most modern architectures have a NOP instruction, so this is a fairly generic test. */
        asm volatile ( "\tnop\n" : : : "cc", "memory" );
        return 0;
      }
      ''',
      name : 'GNU Inline ASM support.')
    config.set10('USE_GCC_INLINE_ASM', true)
  elif use_gnu_asm.enabled()
    error('GNU inline assembly support missing but required.')
  endif
endif

if get_option('timers')
  config.set('PIXMAN_TIMERS', 1)
endif
if get_option('gnuplot')
  config.set('PIXMAN_GNUPLOT', 1)
endif

if cc.get_id() != 'msvc'
  dep_openmp = dependency('openmp', required : get_option('openmp'))
  if dep_openmp.found()
    config.set10('USE_OPENMP', true)
  elif meson.version().version_compare('<0.51.0')
  # In versions of meson before 0.51 the openmp dependency can still
  # inject arguments in the the auto case when it is not found, the
  # detection does work correctly in that case however, so we just
  # replace dep_openmp with null_dep to work around this.
    dep_openmp = null_dep
  endif
else
  # the MSVC implementation of openmp is not compliant enough for our
  # uses here, so we disable it here.
  # Please see: https://stackoverflow.com/questions/12560243/using-threadprivate-directive-in-visual-studio
  dep_openmp = null_dep
endif

# MSVC-style compilers do not come with pthreads, so we must link
# to it explicitly, currently pthreads-win32 is supported
pthreads_found = false


funcs = ['sigaction', 'alarm', 'mprotect', 'getpagesize', 'mmap', 'getisax', 'gettimeofday']
# mingw claimes to have posix_memalign, but it doesn't
if host_machine.system() != 'windows'
  funcs += 'posix_memalign'
endif

foreach f : funcs
  if cc.has_function(f)
    config.set('HAVE_@0@'.format(f.to_upper()), 1)
  endif
endforeach

if cc.has_header_symbol('fenv.h', 'FE_DIVBYZERO')
  config.set('HAVE_FEDIVBYZERO', 1)
endif

foreach h : ['sys/mman.h', 'fenv.h', 'unistd.h']
  if cc.check_header(h)
    config.set('HAVE_@0@'.format(h.underscorify().to_upper()), 1)
  endif
endforeach

use_tls = get_option('tls')
have_tls = ''
if not use_tls.disabled()
  # gcc on Windows only warns that __declspec(thread) isn't supported,
  # passing -Werror=attributes makes it fail.
  if (host_machine.system() == 'windows' and
      cc.compiles('int __declspec(thread) foo;',
                  args : cc.get_supported_arguments(['-Werror=attributes']),
                  name : 'TLS via __declspec(thread)'))
    have_tls = '__declspec(thread)'
  elif cc.compiles('int __thread foo;', name : 'TLS via __thread')
    have_tls = '__thread'
  endif
endif

if have_tls != ''
  config.set('TLS', have_tls)
elif use_tls.enabled()
  error('Compiler TLS Support unavailable, but required')
endif

if cc.links('''
    static int x = 1;
    static void __attribute__((constructor)) constructor_function () { x = 0; }
    int main (void) { return x; }
    ''',
    name : '__attribute__((constructor))')
  config.set('TOOLCHAIN_SUPPORTS_ATTRIBUTE_CONSTRUCTOR', 1)
endif

if cc.links(
    ' __float128 a = 1.0Q, b = 2.0Q; int main (void) { return a + b; }',
    name : 'Has float128 support')
  config.set('HAVE_FLOAT128', 1)
endif

if cc.has_function('clz')
  config.set('HAVE_BUILTIN_CLZ', 1)
endif

if cc.links('''
    unsigned int __attribute__ ((vector_size(16))) e, a, b;
    int main (void) { e = a - ((b << 27) + (b >> (32 - 27))) + 1; return e[0]; }
    ''',
    name : 'Support for GCC vector extensions')
  config.set('HAVE_GCC_VECTOR_EXTENSIONS', 1)
endif

if host_machine.endian() == 'big'
  config.set('WORDS_BIGENDIAN', 1)
endif

config.set('SIZEOF_LONG', cc.sizeof('long'))

# Required to make pixman-private.h
config.set('PACKAGE', 'foo')

version_conf = configuration_data()
split = meson.project_version().split('.')
version_conf.set('PIXMAN_VERSION_MAJOR', split[0])
version_conf.set('PIXMAN_VERSION_MINOR', split[1])
version_conf.set('PIXMAN_VERSION_MICRO', split[2])

add_project_arguments('-DHAVE_CONFIG_H', language : ['c'])

subdir('pixman')

pkg = import('pkgconfig')
pkg.generate(libpixman,
  name : 'Pixman',
  filebase : 'pixman-1',
  description : 'The pixman library (version 1)',
  subdirs: 'pixman-1',
  version : meson.project_version(),
)
