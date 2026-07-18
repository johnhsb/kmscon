# SPDX-FileCopyrightText: 2022 Aetf <aetf@unlimited-code.works>
#
# SPDX-License-Identifier: MIT

#
# Binary File Compiler
#
embed_gen = generator(python,
  arguments: [
    meson.project_source_root() / 'tools' / 'embedded_file.py',
    '@EXTRA_ARGS@',
    '@INPUT@',
    '@BUILD_DIR@',
  ],
  output: ['@PLAINNAME@.bin.h', '@PLAINNAME@.bin.c'],
)
shader_regex = [
  '--regex', '^/\\*.*$',   '', # Start of multi-line comment
  '--regex', '^ \\*.*$',   '', # Multi-line comment body
  '--regex', '^[ \\t]*', '', # Indentation whitespace
  '--regex', '[\\r\\n]', '', # Newlines
]

src_inc = [include_directories('.')]

subdir('shl')

subdir('font')
subdir('input')
subdir('video')
subdir('render')
subdir('misc')
subdir('uterm')

#
# Binaries
# These are the sources for the main binaries and test programs. They mostly
# consists of a single source file only and include all the libraries that are
# built as part of kmscon.
#

kmscon_srcs = [
  'seat.c',
  'config.c',
  'main.c',
  'terminal.c',
]
kmscon_dep = [xkbcommon_deps,
  libtsm_deps,
  threads_deps,
  dl_deps,
  shl_deps,
  eloop_deps,
  uterm_deps,
  render_deps,
  font_deps,
  misc_deps,
]

# IM abstraction layer — always compiled; backends are conditional
kmscon_srcs += 'kmscon_im.c'
if enable_im_hangul
  kmscon_srcs += 'im_hangul.c'
  kmscon_dep += libhangul_deps
endif
kmscon = executable('kmscon', kmscon_srcs,
  dependencies: kmscon_dep,
  export_dynamic: true,
  install: true,
  install_dir: bindir,
)
