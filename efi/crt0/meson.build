o_crt0 = custom_target('efi_crt0',
                       input : arch_crt_source,
                       output : arch_crt,
                       command : [cc.cmd_array(), '-c', '@INPUT@', '-o', '@OUTPUT@']
                       + compile_args)
