generate_sbat = find_program('generate_sbat.py', native: true)
generate_binary = find_program('generate_binary.py', native: true)

efi_ldsdir = get_option('efi-ldsdir')
efi_incdir = get_option('efi-includedir')

gnu_efi_path_arch = ''
foreach name : [gnu_efi_arch, EFI_MACHINE_TYPE_NAME]
  if (gnu_efi_path_arch == '' and name != '' and
    cc.has_header('@0@/@1@/efibind.h'.format(efi_incdir, name)))
    gnu_efi_path_arch = name
  endif
endforeach

if gnu_efi_path_arch != '' and EFI_MACHINE_TYPE_NAME == ''
  error('gnu-efi is available, but EFI_MACHINE_TYPE_NAME is unknown')
endif

efi_libdir = get_option('efi-libdir')
if efi_libdir == ''
  multi = run_command(cc.cmd_array(), '-print-multi-os-directory').stdout().strip()
  efi_libdir = join_paths('/usr/lib/', multi)
endif

have_gnu_efi = gnu_efi_path_arch != '' and efi_libdir != ''

if not have_gnu_efi
  error('gnu-efi support requested, but headers were not found')
endif

# The name we need to look for on this arch and OS: elf_x86_64_fbsd_efi.lds
lds_os = ''
if host_cpu == 'x86_64' and host_machine.system() == 'freebsd'
  lds_os = '_fbsd'
endif

arch_lds = 'efi.lds'
arch_crt = 'crt0.o'
if efi_ldsdir == ''
  efi_ldsdir = join_paths(efi_libdir, 'gnuefi', gnu_efi_path_arch)
  cmd = run_command('test', '-f', join_paths(efi_ldsdir, arch_lds))
  if cmd.returncode() != 0
    arch_lds = 'elf_@0@@1@_efi.lds'.format(gnu_efi_path_arch, lds_os)
    arch_crt = 'crt0-efi-@0@.o'.format(gnu_efi_path_arch)
    efi_ldsdir = join_paths(efi_libdir, 'gnuefi')
    cmd = run_command('test', '-f', join_paths(efi_ldsdir, arch_lds))
  endif
  if cmd.returncode() != 0
    efi_ldsdir = efi_libdir
    cmd = run_command('test', '-f', join_paths(efi_ldsdir, arch_lds))
    if cmd.returncode() != 0
      error('Cannot find @0@'.format(arch_lds))
    endif
  endif
else
  cmd = run_command('test', '-f', join_paths(efi_ldsdir, arch_lds))
  if cmd.returncode() != 0
    arch_lds = 'elf_@0@@1@_efi.lds'.format(gnu_efi_path_arch, lds_os)
    arch_crt = 'crt0-efi-@0@.o'.format(gnu_efi_path_arch)
    cmd = run_command('test', '-f', join_paths(efi_ldsdir, arch_lds))
  endif
  if cmd.returncode() != 0
    error('Cannot find @0@'.format(arch_lds))
  endif
endif
efi_crtdir = efi_ldsdir

# If using objcopy, crt0 must not include the PE/COFF header
if run_command('grep', '-q', 'coff_header', join_paths(efi_crtdir, arch_crt), check: false).returncode() == 0
        coff_header_in_crt0 = true
else
        coff_header_in_crt0 = false
endif

# older objcopy for Aarch64 and ARM32 are not EFI capable.
# Use 'binary' instead, and add required symbols manually.
if host_cpu == 'arm' or (host_cpu == 'aarch64' and (objcopy_version.version_compare ('< 2.38') or coff_header_in_crt0))
  objcopy_manualsymbols = true
  generate_binary_extra = ['--objcopy-manualsymbols']
else
  objcopy_manualsymbols = false
  generate_binary_extra = []
endif

# is the system linker script new enough to know about SBAT?
# i.e. gnu-efi with https://github.com/vathpela/gnu-efi/pull/14 has been installed
if get_option('efi_sbat_distro_id') != ''
  cmd = run_command('grep', '-q', 'sbat', join_paths(efi_ldsdir, arch_lds))
  if cmd.returncode() != 0
    warning('Cannot find SBAT section in @0@, using local copy'.format(join_paths(efi_ldsdir, arch_lds)))
    efi_ldsdir = join_paths(meson.current_source_dir(), 'lds')
  endif
endif

# is the system crt0 for arm and aarch64 new enough to know about SBAT?
if objcopy_manualsymbols
  if get_option('efi_sbat_distro_id') != ''
    arch_crt_source = 'crt0-efi-@0@.S'.format(gnu_efi_path_arch)
    cmd = run_command('grep', '-q', 'sbat', join_paths(efi_crtdir, arch_crt))
    if cmd.returncode() != 0
      warning('Cannot find SBAT section in @0@, using local copy'.format(join_paths(efi_crtdir, arch_crt)))
      # The gnuefi libraries are still needed
      efi_libdir = efi_crtdir
      efi_crtdir = join_paths(meson.current_build_dir(), 'crt0')
      efi_ldsdir = join_paths(meson.current_source_dir(), 'lds')
    endif
  endif
endif

message('efi-libdir: "@0@"'.format(efi_libdir))
message('efi-ldsdir: "@0@"'.format(efi_ldsdir))
message('efi-crtdir: "@0@"'.format(efi_crtdir))
message('efi-includedir: "@0@"'.format(efi_incdir))

debugdir = join_paths (libdir, 'debug')
compile_args = ['-Og',
                '-g3',
                '--param=ssp-buffer-size=4',
                '-fexceptions',
                '-Wall',
                '-Wextra',
                '-Wvla',
                '-std=gnu11',
                '-fpic',
                '-fshort-wchar',
                '-ffreestanding',
                '-fno-strict-aliasing',
                '-fno-stack-protector',
                '-fno-stack-check',
                '-fno-merge-constants',
                '-Wsign-compare',
                '-Wno-missing-field-initializers',
                '-Wno-address-of-packed-member',
                '-grecord-gcc-switches',
                '-DDEBUGDIR="@0@"'.format(debugdir),
                '-I.',
                '-isystem', efi_incdir,
                '-isystem', join_paths(efi_incdir, gnu_efi_path_arch)]
if get_option('werror')
    compile_args += '-Werror'
endif
if host_cpu == 'x86_64'
  compile_args += ['-mno-red-zone',
                   '-mno-sse',
                   '-mno-mmx',
                   '-DEFI_FUNCTION_WRAPPER',
                   '-DGNU_EFI_USE_MS_ABI']
elif host_cpu == 'x86'
  compile_args += ['-mno-sse',
                   '-mno-mmx',
                   '-mno-red-zone',
                   '-m32']
# no special cases for aarch64 or arm
endif

efi_ldflags = ['-T',
               join_paths(efi_ldsdir, arch_lds),
               '-shared',
               '-Bsymbolic',
               '-nostdlib',
               '-znocombreloc',
               '-L', efi_crtdir,
               '-L', efi_libdir,
               join_paths(efi_crtdir, arch_crt)]

if objcopy_manualsymbols
  # older objcopy for Aarch64 and ARM32 are not EFI capable.
  # Use 'binary' instead, and add required symbols manually.
  efi_ldflags += ['--defsym=EFI_SUBSYSTEM=0xa']
  efi_format = ['-O', 'binary']
else
  efi_format = ['--target=efi-app-@0@'.format(gnu_efi_arch)]
endif

libgcc_file_name = run_command(cc.cmd_array(), '-print-libgcc-file-name').stdout().strip()
efi_name = 'fwupd@0@.efi'.format(EFI_MACHINE_TYPE_NAME)

o_file1 = custom_target('fwupdate.o',
                        input : 'fwupdate.c',
                        output : 'fwupdate.o',
                        command : [cc.cmd_array(), '-c', '@INPUT@', '-o', '@OUTPUT@']
                                  + compile_args)
o_file2 = custom_target('fwup-debug.o',
                        input : 'fwup-debug.c',
                        output : 'fwup-debug.o',
                        command : [cc.cmd_array(), '-c', '@INPUT@', '-o', '@OUTPUT@']
                                  + compile_args)
o_file3 = custom_target('fwup-efi.o',
                        input : 'fwup-efi.c',
                        output : 'fwup-efi.o',
                        command : [cc.cmd_array(), '-c', '@INPUT@', '-o', '@OUTPUT@']
                                  + compile_args)
o_file4 = custom_target('fwup-common.o',
                        input : 'fwup-common.c',
                        output : 'fwup-common.o',
                        command : [cc.cmd_array(), '-c', '@INPUT@', '-o', '@OUTPUT@']
                                  + compile_args)

o_file5 = custom_target('fwup-sbat.o',
                        output : 'fwup-sbat.o',
                        command : [
                          generate_sbat,
                          '@OUTPUT@',
                          '--cc', ' '.join(cc.cmd_array()),
                          '--cflags', ' '.join(compile_args),
                          '--objcopy', objcopy,
                          '--project-name', meson.project_name(),
                          '--project-version', meson.project_version(),
                          '--sbat-version', '1',
                          '--sbat-generation', '@0@'.format(get_option('efi_sbat_fwupd_generation')),
                          '--sbat-distro-id', get_option('efi_sbat_distro_id'),
                          '--sbat-distro-generation', '0',
                          '--sbat-distro-summary', get_option('efi_sbat_distro_summary'),
                          '--sbat-distro-pkgname', get_option('efi_sbat_distro_pkgname'),
                          '--sbat-distro-version', get_option('efi_sbat_distro_version'),
                          '--sbat-distro-url', get_option('efi_sbat_distro_url'),
                        ])

fwupd_so_deps = []
if efi_crtdir == join_paths(meson.current_build_dir(), 'crt0')
  # A custom crt0 is needed
  subdir('crt0')
  fwupd_so_deps += [o_crt0]
endif

so = custom_target('fwup.so',
                   input : [o_file1, o_file2, o_file3, o_file4, o_file5],
                   output : 'fwup.so',
                   command : [ld, '-o', '@OUTPUT@'] +
                             efi_ldflags + ['@INPUT@'] +
                             ['-lefi', '-lgnuefi', libgcc_file_name],
                   depends: fwupd_so_deps)

app = custom_target(efi_name,
                    input : so,
                    output : efi_name,
                    command : [
                      generate_binary,
                      '@INPUT@', '@OUTPUT@',
                      '--arch', gnu_efi_arch,
                      '--os', host_machine.system(),
                      '--objcopy', objcopy,
                      '--genpeimg', genpeimg.found() ? genpeimg : ''
                    ]
                    + generate_binary_extra,
                    install : true,
                    install_dir : efi_app_location)

dbg = custom_target('efi_debug',
                    input : so,
                    output : efi_name + '.debug',
                    command : [objcopy,
                               '-j', '.text',
                               '-j', '.sdata',
                               '-j', '.data',
                               '-j', '.dynamic',
                               '-j', '.rodata',
                               '-j', '.rel*',
                               '-j', '.rela*',
                               '-j', '.reloc',
                               '-j', '.eh_frame',
                               '-j', '.debug*',
                               '-j', '.note.gnu.build-id']
                               + efi_format +
                               ['@INPUT@', '@OUTPUT@'],
                    install : false,
                    install_dir : debugdir)
