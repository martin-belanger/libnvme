# SPDX-License-Identifier: LGPL-2.1-or-later
#
# This file is part of libnvme.
# Copyright (c) 2021 Dell Inc.
#
# Authors: Martin Belanger <Martin.Belanger@dell.com>
#
want_python = get_option('python')
if want_python.disabled()
    build_python_bindings = false
    py3_dep = dependency('', required: false)  # Needed for muon
else
    python3 = import('python').find_installation('python3')
    py3_dep = python3.dependency(required: want_python)
    swig = find_program('swig', required: want_python)
    header_found = cc.has_header('Python.h', dependencies: py3_dep, required: want_python)
    build_python_bindings = py3_dep.found() and swig.found() and header_found
endif

if build_python_bindings
    r = run_command(swig, ['-version'], check: true)  # Returns: "\nSWIG Version 4.1.1\n\nCompiled with ..."
    swig_version = r.stdout().split('\n')[1].split()[2].strip()
    if swig_version.version_compare('<4.1.0')
        swig_cmd = [swig, '-python', '-py3', '-o', '@OUTPUT1@', '@INPUT0@']
    else
        swig_cmd = [swig, '-python', '-o', '@OUTPUT1@', '@INPUT0@']
    endif

    pymod_swig = custom_target(
        'nvme.py',
        input:   ['nvme.i'],
        output:  ['nvme.py', 'nvme_wrap.c'],
        command: swig_cmd,
        install: true,
        install_dir: [python3.get_install_dir(pure: false, subdir: 'libnvme'), false],
    )

    pynvme_clib = python3.extension_module(
        '_nvme',
        pymod_swig[1],
        dependencies : [libnvme_dep, py3_dep],
        include_directories: [incdir, internal_incdir],
        link_with: [libccan],
        install: true,
        subdir: 'libnvme',
    )

    # Little hack to copy file __init__.py to the build directory. 
    # This is needed to create the proper directory layout to run the tests.
    # It's a hack because we don't really "configure" file __init__.py and we
    # could simply install directly from the source tree with:
    #   python3.install_sources(['__init__.py', ], pure:false, subdir:'libnvme')
    # However, since we need __init__.py in the build directory to run the tests
    # we resort to this hack to copy it.
    configure_file(
        input:  '__init__.py', 
        output: '__init__.py',
        configuration: conf,
        install_dir: python3.get_install_dir(pure: false, subdir: 'libnvme'),
    )

    # Set the PYTHONPATH so that we can run the 
    # tests directly from the build directory.
    test_env = environment()
    test_env.set('MALLOC_PERTURB_', '90')
    test_env.prepend('PYTHONPATH', join_paths(meson.current_build_dir(), '..'))
    test_env.set('PYTHONMALLOC', 'malloc')

    # Test section
    test('python-import-libnvme', python3, args: ['-c', 'from libnvme import nvme'], env: test_env, depends: pynvme_clib)

    py_tests = [ 
        [ 'create-ctrl-object', [ files('tests/create-ctrl-obj.py'), ] ],
        [ 'sigsegv-during-gc', [ files('tests/gc.py'), ] ],
        [ 'read-nbft-file', [ files('tests/test-nbft.py'), '--filename', join_paths(meson.current_source_dir(), 'tests', 'NBFT') ] ],
    ]
    foreach test: py_tests
        description = test[0]
        args = test[1]
        test('python-' + description, python3, args: args, env: test_env, depends: pynvme_clib)
    endforeach
endif
