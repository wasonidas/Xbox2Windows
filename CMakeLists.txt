cmake_minimum_required(VERSION 3.21)
project(Xbox2Windows LANGUAGES CXX)

enable_testing()

# Defined before the first Python-driven test so a fresh configure registers all of them.
set(INTAKE_PYTHON "${CMAKE_SOURCE_DIR}/out/pe-env/Scripts/python.exe" CACHE FILEPATH
    "Native x64 Python with pefile for static intake tests")

set(DSTORAGE_SDK_ROOT "" CACHE PATH "Extracted Microsoft.Direct3D.DirectStorage NuGet package")
if(DSTORAGE_SDK_ROOT)
    find_package(ZLIB 1.2.9 REQUIRED)
    set(dstorage_native "${DSTORAGE_SDK_ROOT}/native")
    add_executable(dstorage_rfc1950_check tests/dstorage_rfc1950_check.cpp src/dstorage_rfc1950.cpp)
    target_compile_features(dstorage_rfc1950_check PRIVATE cxx_std_17)
    target_include_directories(dstorage_rfc1950_check PRIVATE src "${dstorage_native}/include")
    target_link_libraries(dstorage_rfc1950_check PRIVATE "${dstorage_native}/lib/x64/dstorage.lib" ZLIB::ZLIB)
    add_custom_command(TARGET dstorage_rfc1950_check POST_BUILD
        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${dstorage_native}/bin/x64/dstorage.dll"
            "${dstorage_native}/bin/x64/dstoragecore.dll" "$<TARGET_FILE_DIR:dstorage_rfc1950_check>"
        VERBATIM)
    foreach(scenario IN ITEMS success truncated checksum trailing oversize undersize)
        add_test(NAME dstorage_rfc1950_${scenario} COMMAND dstorage_rfc1950_check ${scenario})
        set_tests_properties(dstorage_rfc1950_${scenario} PROPERTIES TIMEOUT 30)
    endforeach()
endif()
add_executable(guest_content_root_check tests/guest_content_root_check.cpp)
target_compile_features(guest_content_root_check PRIVATE cxx_std_17)
add_test(NAME startup_content_root COMMAND guest_content_root_check "${CMAKE_CURRENT_BINARY_DIR}")
set_tests_properties(startup_content_root PROPERTIES SKIP_RETURN_CODE 77)

add_executable(root_probe tools/root_probe.cpp)
target_compile_features(root_probe PRIVATE cxx_std_17)
target_link_libraries(root_probe PRIVATE d3d12)
add_executable(root_device_probe tools/root_device_probe.cpp)
target_compile_features(root_device_probe PRIVATE cxx_std_17)
target_link_libraries(root_device_probe PRIVATE d3d12 dxgi)
add_executable(d3d12_sdk_check tests/d3d12_sdk_check.cpp)
target_compile_features(d3d12_sdk_check PRIVATE cxx_std_17)
target_link_libraries(d3d12_sdk_check PRIVATE d3d12)
add_test(NAME desktop_root_signature COMMAND d3d12_sdk_check)
add_test(NAME desktop_bvh_root_flags COMMAND d3d12_sdk_check bvh-flags)

add_executable(era_heap_check tests/era_heap_check.cpp src/era_heap.cpp)
target_include_directories(era_heap_check PRIVATE src)
target_compile_features(era_heap_check PRIVATE cxx_std_20)
foreach(scenario IN ITEMS allocation failure concurrency caller_layout reuse observed_words large_alignment graphics_types)
    add_test(NAME era_heap_${scenario} COMMAND era_heap_check ${scenario})
endforeach()

add_executable(memory_query_check tests/memory_query_check.cpp)
target_compile_features(memory_query_check PRIVATE cxx_std_17)
target_link_libraries(memory_query_check PRIVATE psapi)
add_test(NAME desktop_memory_query COMMAND memory_query_check)

add_executable(xmem_committed_memory_check tests/xmem_committed_memory_check.cpp)
target_compile_features(xmem_committed_memory_check PRIVATE cxx_std_17)
target_link_libraries(xmem_committed_memory_check PRIVATE OneCore)
add_test(NAME xmem_committed_memory COMMAND xmem_committed_memory_check)

add_executable(era_query_check tests/era_query_check.cpp src/era_query.cpp)
target_include_directories(era_query_check PRIVATE src)
target_compile_features(era_query_check PRIVATE cxx_std_17)
foreach(scenario IN ITEMS fields tail registry short)
    add_test(NAME era_query_${scenario} COMMAND era_query_check ${scenario})
endforeach()
# A clear tail flag overwrites a stale byte, and another process's page at a registered address
# has it clear. The mutated run, which expects the flag set there, is the negative control.
add_test(NAME era_query_clear COMMAND era_query_check clear)
add_test(NAME era_query_clear_foreign_control COMMAND era_query_check clear --mutate foreign)
set_tests_properties(era_query_clear_foreign_control PROPERTIES WILL_FAIL TRUE)

# Title GPU address to desktop GPU virtual address table (no GPU needed).
add_executable(era_title_va_check tests/era_title_va_check.cpp src/era_title_va.cpp)
target_include_directories(era_title_va_check PRIVATE src)
target_compile_features(era_title_va_check PRIVATE cxx_std_17)
foreach(scenario IN ITEMS translate bounds registry shader)
    add_test(NAME era_title_va_${scenario} COMMAND era_title_va_check ${scenario})
endforeach()

# era_title_gpu under allocation failure (no GPU needed). The program replaces operator new to fail
# one chosen allocation at a time.
add_executable(era_title_gpu_alloc_check tests/era_title_gpu_alloc_check.cpp src/era_title_gpu.cpp
    src/era_gpu_alias.cpp src/era_title_va.cpp src/era_query.cpp)
target_include_directories(era_title_gpu_alloc_check PRIVATE src)
target_compile_features(era_title_gpu_alloc_check PRIVATE cxx_std_20)
add_test(NAME era_title_gpu_alloc COMMAND era_title_gpu_alloc_check)
set_tests_properties(era_title_gpu_alloc PROPERTIES TIMEOUT 30)

# Physical-page bookkeeping behind XMemAllocatePhysicalPages/XMemFreePhysicalPages, linked next to a
# lenient build of the same file (ERA_PHYSICAL_LENIENT_RELEASE) that the ownership controls must catch.
add_library(era_physical_lenient OBJECT src/era_physical.cpp)
target_include_directories(era_physical_lenient PRIVATE src)
target_compile_features(era_physical_lenient PRIVATE cxx_std_17)
target_compile_definitions(era_physical_lenient PRIVATE ERA_PHYSICAL_LENIENT_RELEASE era_physical=era_physical_lenient)
add_executable(era_physical_check tests/era_physical_check.cpp src/era_physical.cpp $<TARGET_OBJECTS:era_physical_lenient>)
target_include_directories(era_physical_check PRIVATE src)
target_compile_features(era_physical_check PRIVATE cxx_std_17)
foreach(scenario IN ITEMS controls lifetime)
    add_test(NAME era_physical_${scenario} COMMAND era_physical_check ${scenario})
endforeach()

# Desktop xmem.dll for native and mapped callers.
add_library(xmem_shim SHARED src/xmem_shim.cpp src/xmem_shim.def src/xmem_gpu_usage.cpp src/adapter_identity.cpp src/era_heap.cpp src/era_query.cpp src/era_physical.cpp
    src/era_title_gpu.cpp src/era_gpu_alias.cpp src/era_title_va.cpp src/xmem_mapped_pages.cpp)
set_target_properties(xmem_shim PROPERTIES OUTPUT_NAME xmem)
# LINK uses explicit export ordinals in the generated import library. Keep
# those ordinals in the DLL and build the consumer library with names.
set(xmem_export_def "${CMAKE_CURRENT_SOURCE_DIR}/src/xmem_shim.def")
set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS "${xmem_export_def}")
file(READ "${xmem_export_def}" xmem_named_exports)
string(REGEX REPLACE "(=[A-Za-z0-9_]+)? +@[0-9]+" "" xmem_named_exports "${xmem_named_exports}")
set(xmem_named_def "${CMAKE_CURRENT_BINARY_DIR}/xmem_named_imports.def")
file(GENERATE OUTPUT "${xmem_named_def}" CONTENT "${xmem_named_exports}")
set_property(TARGET xmem_shim APPEND PROPERTY LINK_DEPENDS "${xmem_named_def}")
add_custom_command(TARGET xmem_shim POST_BUILD
    COMMAND "${CMAKE_AR}" /nologo /machine:x64 "/def:${xmem_named_def}" "/out:$<TARGET_LINKER_FILE:xmem_shim>"
    VERBATIM)
target_include_directories(xmem_shim PRIVATE src)
target_compile_features(xmem_shim PRIVATE cxx_std_20)
# Delay-loaded: only XMemGetWorkingSetStatistics uses DXGI and D3DKMT (gdi32).
target_link_libraries(xmem_shim PRIVATE dxgi gdi32 delayimp OneCore)
target_link_options(xmem_shim PRIVATE /DELAYLOAD:dxgi.dll /DELAYLOAD:gdi32.dll)
# The same shim with its test controls compiled in (X2W_XMEM_TEST_CONTROLS): refusing one step of an
# unmap, and shared views without the fault handler. tests/xmem_page_runs_check.py loads it for the
# cases that need a control; the deployed xmem.dll has neither switch.
get_target_property(xmem_shim_sources xmem_shim SOURCES)
add_library(xmem_shim_test_controls SHARED ${xmem_shim_sources})
set_target_properties(xmem_shim_test_controls PROPERTIES OUTPUT_NAME xmem
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/xmem-test-controls"
    LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/xmem-test-controls"
    ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/xmem-test-controls"
    MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
target_compile_definitions(xmem_shim_test_controls PRIVATE X2W_XMEM_TEST_CONTROLS)
target_include_directories(xmem_shim_test_controls PRIVATE src)
target_compile_features(xmem_shim_test_controls PRIVATE cxx_std_20)
target_link_libraries(xmem_shim_test_controls PRIVATE dxgi gdi32 delayimp OneCore)
target_link_options(xmem_shim_test_controls PRIVATE /DELAYLOAD:dxgi.dll /DELAYLOAD:gdi32.dll)
add_executable(xmem_shim_check tests/xmem_shim_check.cpp src/adapter_identity.cpp)
# D3D12 and DXGI are delay-loaded so the exports scenario can check that xmem.dll loads neither.
target_link_libraries(xmem_shim_check PRIVATE xmem_shim d3d12 dxgi gdi32 delayimp)
target_link_options(xmem_shim_check PRIVATE /DELAYLOAD:d3d12.dll /DELAYLOAD:dxgi.dll)
target_include_directories(xmem_shim_check PRIVATE src)
target_compile_features(xmem_shim_check PRIVATE cxx_std_17)
# working_set_aggregation feeds synthetic adapter readings to src/xmem_gpu_usage.h.
foreach(scenario IN ITEMS exports allocation default_allocation physical_pages tools_memory transfer_memory query working_set working_set_aggregation)
    add_test(NAME xmem_shim_${scenario} COMMAND xmem_shim_check ${scenario})
endforeach()
# XMemGetWorkingSetStatistics differences around D3D12 heap and committed-resource lifetimes.
foreach(adapter IN ITEMS warp hardware)
    add_test(NAME xmem_shim_working_set_${adapter} COMMAND xmem_shim_check working_set_${adapter})
    set_tests_properties(xmem_shim_working_set_${adapter} PROPERTIES TIMEOUT 30 SKIP_RETURN_CODE 77)
endforeach()

# Second commit/protection check (#870): ordinary committed pages vs live shared-page views.
add_executable(xmem_commit_protection_check tests/xmem_commit_protection_check.cpp)
target_link_libraries(xmem_commit_protection_check PRIVATE xmem_shim OneCore)
target_include_directories(xmem_commit_protection_check PRIVATE src)
target_compile_features(xmem_commit_protection_check PRIVATE cxx_std_20)
add_test(NAME xmem_commit_protection COMMAND xmem_commit_protection_check)
set_tests_properties(xmem_commit_protection PROPERTIES TIMEOUT 30)
if(EXISTS "${INTAKE_PYTHON}")
    # Startup memory accounting (#1199): the live physical-page section snapshot, and the peak commit
    # and section count the DirectStorage heartbeat reports from it.
    add_test(NAME xmem_page_stats COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/xmem_page_stats_check.py" "$<TARGET_FILE:xmem_shim>")
    add_test(NAME startup_heartbeat_commit COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/startup_heartbeat_check.py"
        "$<TARGET_FILE:xmem_shim>" "$<TARGET_FILE:startup_stub_xcrdapi>")
    set_tests_properties(startup_heartbeat_commit PROPERTIES TIMEOUT 60)
    # Startup tooling lint (#1149): every Python file under tools/startup must parse. The check compiles each
    # source without running it and self-tests that a broken snippet is rejected, so a passing run means the
    # files were actually looked at. It needs no GPU and no game files.
    add_test(NAME startup_tools_lint COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/startup_tools_lint_check.py" "${CMAKE_SOURCE_DIR}/tools/startup")
    set_tests_properties(startup_tools_lint PROPERTIES TIMEOUT 60)
endif()
# Mutation controls: each run flips one expected value and passes only when the
# named check reports it. An unknown mutation name or an overall PASS fails.
foreach(mutation_and_check
        "xva_positive_refused|check_failed: xva_accepted_pair"
        "xva_commit_accepted|check_failed: xva_commit_bit"
        "xva_readonly_accepted|check_failed: xva_protection"
        "xva_state_dependent|check_failed: xva_address_state_independent"
        "private_second_va_aliases|check_failed: private_second_va_independent.*failing_fields=private_second_va\n"
        "private_type_mapped|check_failed: private_type.*failing_fields=private_type\n"
        "mapped_decommit_succeeds|check_failed: mapped_decommit.*failing_fields=mapped_decommit\n")
  string(FIND "${mutation_and_check}" "|" split)
  string(SUBSTRING "${mutation_and_check}" 0 ${split} mutation)
  math(EXPR expected_start "${split} + 1")
  string(SUBSTRING "${mutation_and_check}" ${expected_start} -1 expected_output)
  add_test(NAME xmem_commit_protection_mutate_${mutation}
           COMMAND xmem_commit_protection_check --mutate ${mutation})
  set_tests_properties(xmem_commit_protection_mutate_${mutation} PROPERTIES
    TIMEOUT 30
    PASS_REGULAR_EXPRESSION "${expected_output}"
    FAIL_REGULAR_EXPRESSION "unknown mutation|xmem_commit_protection: PASS")
endforeach()

add_executable(windows_placeholder_partition_check tests/windows_placeholder_partition_check.cpp)
target_compile_features(windows_placeholder_partition_check PRIVATE cxx_std_17)
target_link_libraries(windows_placeholder_partition_check PRIVATE OneCore)
add_test(NAME windows_placeholder_partition COMMAND windows_placeholder_partition_check)

# Desktop d3d12_xs.dll for the package DLLs' two named d3d12_xs root-signature imports.
add_library(d3d12_xs_shim SHARED src/d3d12_xs_shim.cpp src/d3d12x_device_slots.cpp
    src/d3d12_xs_iids.cpp src/d3d12_xs_shim.def src/startup_stubs/d3d12_xs_device.cpp)
set_target_properties(d3d12_xs_shim PROPERTIES OUTPUT_NAME d3d12_xs)
target_include_directories(d3d12_xs_shim PRIVATE src)
target_compile_features(d3d12_xs_shim PRIVATE cxx_std_17)
target_link_libraries(d3d12_xs_shim PRIVATE d3d12)
# The creation-IID gate and refusal text of D3D12XboxCreateDevice (src/d3d12x_device_slots.h),
# without a device. The mutated run is the negative control.
add_executable(d3d12x_device_slots_check tests/d3d12x_device_slots_check.cpp src/d3d12x_device_slots.cpp)
target_include_directories(d3d12x_device_slots_check PRIVATE src)
target_compile_features(d3d12x_device_slots_check PRIVATE cxx_std_17)
add_test(NAME d3d12x_device_slots COMMAND d3d12x_device_slots_check)
add_test(NAME d3d12x_device_slots_device1_control COMMAND d3d12x_device_slots_check --mutate device1)
set_tests_properties(d3d12x_device_slots_device1_control PROPERTIES
  PASS_REGULAR_EXPRESSION "FAIL ID3D12Device1: accepted=0, expected 1"
  FAIL_REGULAR_EXPRESSION "usage:|d3d12x device slots: creation IIDs")

# Fixture-only desktop xgameruntime.dll. The single X-macro table in
# xgameruntime_shim.cpp distinguishes implemented synchronous contracts from
# logged E_NOTIMPL stubs. Keep it out of the normal shim directory so a package
# cannot treat the production-named DLL as a provider.
add_library(xgameruntime_shim SHARED src/xgameruntime_shim.cpp src/xgameruntime_shim.def)
set(xgameruntime_fixture_dir "${CMAKE_BINARY_DIR}/fixture-shims")
set_target_properties(xgameruntime_shim PROPERTIES
    OUTPUT_NAME xgameruntime
    RUNTIME_OUTPUT_DIRECTORY "${xgameruntime_fixture_dir}"
    LIBRARY_OUTPUT_DIRECTORY "${xgameruntime_fixture_dir}"
    ARCHIVE_OUTPUT_DIRECTORY "${xgameruntime_fixture_dir}")
target_compile_features(xgameruntime_shim PRIVATE cxx_std_17)

add_executable(xgameruntime_shim_check tests/xgameruntime_thunks_check.cpp)
add_dependencies(xgameruntime_shim_check xgameruntime_shim)
target_compile_features(xgameruntime_shim_check PRIVATE cxx_std_17)
add_test(NAME xgameruntime_thunks COMMAND xgameruntime_shim_check
    "$<TARGET_FILE:xgameruntime_shim>" "$<TARGET_FILE_DIR:xgameruntime_shim>"
    "$<TARGET_FILE_DIR:xmem_shim>")
set_tests_properties(xgameruntime_thunks PROPERTIES TIMEOUT 30)

add_executable(xgameruntime_contracts_check tests/xgameruntime_contracts_check.cpp)
target_link_libraries(xgameruntime_contracts_check PRIVATE xgameruntime_shim)
target_compile_features(xgameruntime_contracts_check PRIVATE cxx_std_17)
add_test(NAME xgameruntime_contracts COMMAND xgameruntime_contracts_check
    "$<TARGET_FILE:xgameruntime_shim>" "$<TARGET_FILE_DIR:xgameruntime_shim>"
    "$<TARGET_FILE_DIR:xmem_shim>")
set_tests_properties(xgameruntime_contracts PROPERTIES TIMEOUT 30)

add_executable(xgameruntime_pls_xpackage_check tests/xgameruntime_pls_xpackage_check.cpp)
add_dependencies(xgameruntime_pls_xpackage_check xgameruntime_shim)
target_compile_features(xgameruntime_pls_xpackage_check PRIVATE cxx_std_17)
add_test(NAME xgameruntime_pls_xpackage COMMAND xgameruntime_pls_xpackage_check
    "$<TARGET_FILE:xgameruntime_shim>" "$<TARGET_FILE_DIR:xgameruntime_shim>"
    "$<TARGET_FILE_DIR:xmem_shim>")
set_tests_properties(xgameruntime_pls_xpackage PROPERTIES TIMEOUT 30)

add_executable(xgameruntime_identity_check tests/xgameruntime_identity_check.cpp)
add_dependencies(xgameruntime_identity_check xgameruntime_shim)
target_compile_features(xgameruntime_identity_check PRIVATE cxx_std_17)
add_test(NAME xgameruntime_identity COMMAND xgameruntime_identity_check
    "$<TARGET_FILE:xgameruntime_shim>" "$<TARGET_FILE_DIR:xgameruntime_shim>"
    "$<TARGET_FILE_DIR:xmem_shim>")
set_tests_properties(xgameruntime_identity PROPERTIES TIMEOUT 30)

# A real source-built EXE import fixture. The import library below only names
# xgameruntime.dll; no build product or test package copies the fixture DLL.
set(xgameruntime_import_def "${CMAKE_CURRENT_SOURCE_DIR}/tests/xgameruntime_import.def")
set(xgameruntime_import_lib "${CMAKE_CURRENT_BINARY_DIR}/xgameruntime_fixture_import.lib")
add_custom_command(OUTPUT "${xgameruntime_import_lib}"
    COMMAND "${CMAKE_AR}" /nologo /machine:x64 "/def:${xgameruntime_import_def}"
            "/out:${xgameruntime_import_lib}"
    DEPENDS tests/xgameruntime_import.def VERBATIM)
add_executable(xgameruntime_preflight_exe tests/xgameruntime_preflight_exe.cpp)
target_sources(xgameruntime_preflight_exe PRIVATE "${xgameruntime_import_lib}")
target_link_libraries(xgameruntime_preflight_exe PRIVATE "${xgameruntime_import_lib}")
target_compile_features(xgameruntime_preflight_exe PRIVATE cxx_std_17)
target_compile_options(xgameruntime_preflight_exe PRIVATE /GS-)
target_link_options(xgameruntime_preflight_exe PRIVATE /ENTRY:fixture_entry /NODEFAULTLIB
    /SUBSYSTEM:CONSOLE /DYNAMICBASE /INCREMENTAL:NO)

# pe_run_package --trap-callable fixtures. The import library only names trap_fixture.dll;
# nothing builds or copies a DLL under that name. The _data variant adds a data import.
set(trap_missing_import_def "${CMAKE_CURRENT_SOURCE_DIR}/tests/trap_missing_import.def")
set(trap_missing_import_lib "${CMAKE_CURRENT_BINARY_DIR}/trap_missing_fixture_import.lib")
add_custom_command(OUTPUT "${trap_missing_import_lib}"
    COMMAND "${CMAKE_AR}" /nologo /machine:x64 "/def:${trap_missing_import_def}"
            "/out:${trap_missing_import_lib}"
    DEPENDS tests/trap_missing_import.def VERBATIM)
foreach(target IN ITEMS trap_missing_exe trap_missing_data_exe)
    add_executable(${target} tests/trap_missing_exe.cpp)
    target_sources(${target} PRIVATE "${trap_missing_import_lib}")
    target_link_libraries(${target} PRIVATE "${trap_missing_import_lib}" kernel32)
    target_compile_options(${target} PRIVATE /GS-)
    target_link_options(${target} PRIVATE /ENTRY:fixture_entry /NODEFAULTLIB
        /SUBSYSTEM:CONSOLE /DYNAMICBASE /INCREMENTAL:NO)
endforeach()
target_compile_definitions(trap_missing_data_exe PRIVATE TRAP_DATA_IMPORT)

# Experimental startup stand-ins (docs/startup-runtime.md): trap-by-default desktop DLLs for imports
# without a public ABI yet (src/startup_stubs/trap.h), plus opt-in permissive paths for startup runs.
set(startup_stub_targets)
foreach(stub IN ITEMS acphal:AcpHal chad:chad xapu:xapu xcrdapi:XCRDAPI xg_xs:xg_xs gameinput:GameInput)
    string(REPLACE ":" ";" stub_parts "${stub}")
    list(GET stub_parts 0 stub_stem)
    list(GET stub_parts 1 stub_output)
    add_library(startup_stub_${stub_stem} SHARED src/startup_stubs/${stub_stem}.cpp src/startup_stubs/${stub_stem}.def)
    set_target_properties(startup_stub_${stub_stem} PROPERTIES OUTPUT_NAME ${stub_output})
    target_include_directories(startup_stub_${stub_stem} PRIVATE src)
    target_compile_features(startup_stub_${stub_stem} PRIVATE cxx_std_17)
    list(APPEND startup_stub_targets startup_stub_${stub_stem})
endforeach()
# A package may ship an older MSVCP140.dll that the loader picks from the application directory;
# link the shims statically so they never use its runtime (constexpr std::mutex layout).
target_link_libraries(startup_stub_xg_xs PRIVATE d3d12 dxguid)
target_link_libraries(startup_stub_gameinput PRIVATE user32)
target_link_libraries(startup_stub_xapu PRIVATE winmm)
# The DirectStorage heartbeat reports this process's peak commit (#1199).
target_link_libraries(startup_stub_xcrdapi PRIVATE psapi)
target_sources(startup_stub_gameinput PRIVATE src/startup_stubs/runtime_proxy.cpp)
target_link_libraries(d3d12_xs_shim PRIVATE user32 gdi32 d3dcompiler dxgi)
target_compile_options(d3d12_xs_shim PRIVATE /bigobj)
set_target_properties(xmem_shim d3d12_xs_shim ${startup_stub_targets} PROPERTIES
    MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
add_executable(startup_null_views_check tests/startup_null_views_check.cpp)
target_compile_features(startup_null_views_check PRIVATE cxx_std_17)
target_link_libraries(startup_null_views_check PRIVATE d3d12_xs_shim d3d12 dxgi)
foreach(adapter IN ITEMS hardware warp)
    add_test(NAME startup_null_views_${adapter} COMMAND startup_null_views_check ${adapter})
endforeach()
set_tests_properties(startup_null_views_hardware PROPERTIES SKIP_RETURN_CODE 77)
# The slot and label of the x2w-render totals line each null-binding cause is counted under
# (src/startup_stubs/null_binding_cause.h). The mutated run is the negative control.
add_executable(startup_null_cause_labels_check tests/startup_null_cause_labels_check.cpp)
target_include_directories(startup_null_cause_labels_check PRIVATE src)
target_compile_features(startup_null_cause_labels_check PRIVATE cxx_std_17)
add_test(NAME startup_null_cause_labels COMMAND startup_null_cause_labels_check)
add_test(NAME startup_null_cause_labels_buffer_token_control COMMAND startup_null_cause_labels_check --mutate buffer-token)
set_tests_properties(startup_null_cause_labels_buffer_token_control PROPERTIES
  PASS_REGULAR_EXPRESSION "startup_null_cause_labels: 6 of 14 FAILED"
  FAIL_REGULAR_EXPRESSION "unknown mutation|startup_null_cause_labels: PASS")
# The opt-in API trace reads only declared outputs; this checks the decision about what it observed
# (argument index 0, two outputs, exact and short buffers, same value, partial comparison).
add_executable(api_trace_check tests/api_trace_check.cpp)
target_compile_features(api_trace_check PRIVATE cxx_std_17)
target_include_directories(api_trace_check PRIVATE src)
add_test(NAME api_trace COMMAND api_trace_check)
# A view heap larger than the adapter accepts: the stand-in creates the desktop heap at the measured
# limit and keeps the title's full-size memory. Skips where the adapter accepts the full request.
add_executable(startup_view_heap_check tests/startup_view_heap_check.cpp)
target_compile_features(startup_view_heap_check PRIVATE cxx_std_17)
target_link_libraries(startup_view_heap_check PRIVATE d3d12_xs_shim d3d12 dxgi)
foreach(adapter IN ITEMS hardware warp)
    add_test(NAME startup_view_heap_${adapter} COMMAND startup_view_heap_check ${adapter} "${CMAKE_CURRENT_BINARY_DIR}/startup_view_heap_${adapter}.log")
    set_tests_properties(startup_view_heap_${adapter} PROPERTIES SKIP_RETURN_CODE 77)
endforeach()
add_executable(startup_stub_consumer tests/startup_stub_consumer.cpp)
add_executable(startup_gameinput_keys tests/startup_gameinput_keys.cpp)
target_compile_features(startup_gameinput_keys PRIVATE cxx_std_17)
add_test(NAME startup_gameinput_keys COMMAND startup_gameinput_keys "$<TARGET_FILE:startup_stub_gameinput>")
add_executable(startup_gameinput_history tests/startup_gameinput_history.cpp)
target_compile_features(startup_gameinput_history PRIVATE cxx_std_17)
add_test(NAME startup_gameinput_history COMMAND startup_gameinput_history "$<TARGET_FILE:startup_stub_gameinput>")
set_tests_properties(startup_gameinput_history PROPERTIES RUN_SERIAL TRUE)
add_executable(startup_placement_alloc tests/startup_placement_alloc.cpp)
target_compile_features(startup_placement_alloc PRIVATE cxx_std_17)
target_include_directories(startup_placement_alloc PRIVATE src)
add_test(NAME startup_placement_alloc COMMAND startup_placement_alloc)
# The same registration under the lock held exclusively, and max_width kept at the widest record
# (src/startup_stubs/placement_publish.h). The mutated run is the negative control.
add_executable(startup_placement_publish_check tests/startup_placement_publish_check.cpp)
target_compile_features(startup_placement_publish_check PRIVATE cxx_std_17)
target_include_directories(startup_placement_publish_check PRIVATE src)
add_test(NAME startup_placement_publish COMMAND startup_placement_publish_check)
add_test(NAME startup_placement_publish_shrink_control COMMAND startup_placement_publish_check --mutate shrink)
set_tests_properties(startup_placement_publish_shrink_control PROPERTIES WILL_FAIL TRUE)
add_executable(startup_storage_file_range tests/startup_storage_file_range.cpp)
target_compile_features(startup_storage_file_range PRIVATE cxx_std_17)
target_include_directories(startup_storage_file_range PRIVATE src)
add_test(NAME startup_storage_file_range COMMAND startup_storage_file_range)
# The same read at offsets that need the high half of the OVERLAPPED position, in a sparse
# 4 GiB + 20 byte temporary file (src/startup_stubs/dstorage_file_read.h). 77 without sparse files.
add_executable(startup_dstorage_file_read_check tests/startup_dstorage_file_read_check.cpp)
target_compile_features(startup_dstorage_file_read_check PRIVATE cxx_std_17)
target_include_directories(startup_dstorage_file_read_check PRIVATE src)
add_test(NAME startup_dstorage_file_read COMMAND startup_dstorage_file_read_check)
add_test(NAME startup_dstorage_file_read_alias_control COMMAND startup_dstorage_file_read_check --mutate alias)
set_tests_properties(startup_dstorage_file_read startup_dstorage_file_read_alias_control PROPERTIES SKIP_RETURN_CODE 77)
set_tests_properties(startup_dstorage_file_read_alias_control PROPERTIES WILL_FAIL TRUE)
add_executable(startup_inflate_status tests/startup_inflate_status.cpp)
target_compile_features(startup_inflate_status PRIVATE cxx_std_17)
target_include_directories(startup_inflate_status PRIVATE src)
add_test(NAME startup_inflate_status COMMAND startup_inflate_status)
# An allocation failure on the DirectStorage stand-in's worker fails that request, and the worker
# serves the next one (#1199). Skips (77) where the system does not enforce a job commit limit.
add_executable(startup_storage_alloc_failure tests/startup_storage_alloc_failure.cpp)
target_compile_features(startup_storage_alloc_failure PRIVATE cxx_std_17)
target_link_libraries(startup_storage_alloc_failure PRIVATE psapi)
add_dependencies(startup_storage_alloc_failure startup_stub_xcrdapi)
add_test(NAME startup_storage_alloc_failure COMMAND startup_storage_alloc_failure "$<TARGET_FILE:startup_stub_xcrdapi>")
set_tests_properties(startup_storage_alloc_failure PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
# A queue whose worker cannot start (both CreateThread attempts fail, or the construction of its work
# throws std::bad_alloc) completes status entries and fences at Submit, never inside Enqueue (#1199).
# The test compiles the stand-in's source into itself so it can redirect CreateThread and operator new.
add_executable(startup_storage_worker_start tests/startup_storage_worker_start.cpp)
target_compile_features(startup_storage_worker_start PRIVATE cxx_std_17)
target_include_directories(startup_storage_worker_start PRIVATE src)
target_link_libraries(startup_storage_worker_start PRIVATE psapi)
add_test(NAME startup_storage_worker_start COMMAND startup_storage_worker_start)
set_tests_properties(startup_storage_worker_start PROPERTIES TIMEOUT 60)
target_compile_features(startup_stub_consumer PRIVATE cxx_std_17)
target_include_directories(startup_stub_consumer PRIVATE src)
target_sources(startup_stub_consumer PRIVATE src/startup_stubs/runtime_proxy.cpp)
target_link_libraries(startup_stub_consumer PRIVATE startup_stub_chad startup_stub_xg_xs xmem_shim d3d12_xs_shim)
# APU heap stand-in: ApuFree reuse, coalescing, refused requests and concurrent callers (docs/startup-runtime.md).
add_executable(startup_acphal_apu_free tests/startup_acphal_apu_free.cpp)
target_compile_features(startup_acphal_apu_free PRIVATE cxx_std_17)
add_dependencies(startup_acphal_apu_free startup_stub_acphal)
add_test(NAME startup_acphal_apu_free COMMAND startup_acphal_apu_free "$<TARGET_FILE:startup_stub_acphal>")
set_tests_properties(startup_acphal_apu_free PROPERTIES TIMEOUT 60)

add_executable(startup_xapu_queue_check tests/startup_xapu_queue_check.cpp)
target_compile_features(startup_xapu_queue_check PRIVATE cxx_std_17)
add_dependencies(startup_xapu_queue_check startup_stub_xapu)
add_test(NAME startup_xapu_queue COMMAND startup_xapu_queue_check "$<TARGET_FILE:startup_stub_xapu>")
set_tests_properties(startup_xapu_queue PROPERTIES TIMEOUT 60)

add_executable(startup_draw_bmp_check tests/startup_draw_bmp.cpp)
add_test(NAME startup_draw_bmp COMMAND startup_draw_bmp_check)
# Game-save stand-in storage (src/startup_stubs/save_store.h): a blob written by one process is read by a second.
add_executable(startup_gamesave_persist tests/startup_gamesave_persist.cpp)
target_compile_features(startup_gamesave_persist PRIVATE cxx_std_17)
add_test(NAME startup_gamesave_persist COMMAND startup_gamesave_persist)
# Publication of a multi-blob update: a second process sees the whole old generation or the whole
# new one. The test is built from two generated copies of the store headers, each with a pause
# point at every step of the publication (the `x2w-seam` comments). In the second copy the file lock
# is also turned into a no-op: that build must be caught by the same test, which proves the check
# can fail. Nothing of this reaches the production headers.
file(READ "${CMAKE_CURRENT_SOURCE_DIR}/src/startup_stubs/save_store.h" gamesave_store_text)
file(READ "${CMAKE_CURRENT_SOURCE_DIR}/src/startup_stubs/file_lock.h" gamesave_lock_text)
foreach(anchor "#pragma once" "// x2w-seam: moved-aside" "// x2w-seam: published")
    string(FIND "${gamesave_store_text}" "${anchor}" gamesave_anchor)
    if(gamesave_anchor LESS 0)
        message(FATAL_ERROR "save_store.h lost the anchor '${anchor}' used by startup_gamesave_publish")
    endif()
endforeach()
string(FIND "${gamesave_lock_text}" "// x2w-seam: lock-attempt" gamesave_anchor)
if(gamesave_anchor LESS 0)
    message(FATAL_ERROR "file_lock.h lost the anchor used by startup_gamesave_publish")
endif()
string(REPLACE "#pragma once" "#pragma once\nvoid x2w_save_test_point(unsigned point);" gamesave_store_text "${gamesave_store_text}")
string(REPLACE "// x2w-seam: moved-aside" "x2w_save_test_point(2u * static_cast<unsigned>(&step - steps.data()));" gamesave_store_text "${gamesave_store_text}")
string(REPLACE "// x2w-seam: published" "x2w_save_test_point(2u * static_cast<unsigned>(&step - steps.data()) + 1u);" gamesave_store_text "${gamesave_store_text}")
string(REPLACE "// x2w-seam: lock-attempt" "static volatile bool x2w_no_lock = true;\n        if (x2w_no_lock) return Attempt::locked;" gamesave_nolock_text "${gamesave_lock_text}")
foreach(variant locked unlocked)
    set(gamesave_dir "${CMAKE_CURRENT_BINARY_DIR}/gamesave_publish_${variant}")
    file(WRITE "${gamesave_dir}/startup_stubs/save_store.h" "${gamesave_store_text}")
    if(variant STREQUAL "locked")
        file(WRITE "${gamesave_dir}/startup_stubs/file_lock.h" "${gamesave_lock_text}")
    else()
        file(WRITE "${gamesave_dir}/startup_stubs/file_lock.h" "${gamesave_nolock_text}")
    endif()
    add_executable(startup_gamesave_publish_${variant} tests/startup_gamesave_publish.cpp)
    target_include_directories(startup_gamesave_publish_${variant} PRIVATE "${gamesave_dir}")
    target_compile_features(startup_gamesave_publish_${variant} PRIVATE cxx_std_17)
    add_test(NAME startup_gamesave_publish_${variant} COMMAND startup_gamesave_publish_${variant})
    set_tests_properties(startup_gamesave_publish_${variant} PROPERTIES TIMEOUT 120)
endforeach()
# The copy without the lock must be caught by the reader that sees a mixed or incomplete generation.
set_tests_properties(startup_gamesave_publish_unlocked PROPERTIES PASS_REGULAR_EXPRESSION "reader saw a mixed generation")
add_executable(startup_gamesave_lock_errors tests/startup_gamesave_lock_errors.cpp)
target_include_directories(startup_gamesave_lock_errors PRIVATE src)
target_compile_features(startup_gamesave_lock_errors PRIVATE cxx_std_17)
add_test(NAME startup_gamesave_lock_errors COMMAND startup_gamesave_lock_errors)
add_executable(startup_gamesave_submit tests/startup_gamesave_submit.cpp)
target_include_directories(startup_gamesave_submit PRIVATE src)
target_compile_features(startup_gamesave_submit PRIVATE cxx_std_17)
add_executable(startup_gamesave_read_async tests/startup_gamesave_read_async.cpp)
target_include_directories(startup_gamesave_read_async PRIVATE src)
target_compile_features(startup_gamesave_read_async PRIVATE cxx_std_17)

# Input for the launcher containment check (docs/startup-runtime.md): writes known bytes to both
# streams and the trap log, reports a known exit code, or holds memory, spawns a child or stays
# alive so the launch tool's job limits have to stop it.
# Input for tests/startup_host_cpu_debug_check.py.
add_executable(startup_host_cpu_probe tests/startup_host_cpu_probe.cpp)
target_compile_features(startup_host_cpu_probe PRIVATE cxx_std_17)
# Placed buffers at the boundary between two views of physical pages (src/xmem_mapped_pages.cpp).
add_executable(xmem_page_runs_device_check tests/xmem_page_runs_device_check.cpp)
target_compile_features(xmem_page_runs_device_check PRIVATE cxx_std_17)
target_link_libraries(xmem_page_runs_device_check PRIVATE xmem_shim d3d12_xs_shim d3d12 dxgi)
foreach(adapter IN ITEMS warp hardware)
    add_test(NAME xmem_page_runs_device_${adapter} COMMAND xmem_page_runs_device_check ${adapter})
    set_tests_properties(xmem_page_runs_device_${adapter} PROPERTIES TIMEOUT 120)
endforeach()
set_tests_properties(xmem_page_runs_device_hardware PROPERTIES SKIP_RETURN_CODE 77)
add_executable(startup_launch_probe tests/startup_launch_probe.cpp)
target_compile_features(startup_launch_probe PRIVATE cxx_std_17)
# Input for the startup debugger's diagnostics check (tests/startup_debug_check.py).
add_executable(startup_debug_probe tests/startup_debug_probe.cpp)
target_compile_features(startup_debug_probe PRIVATE cxx_std_17)
# Input for tests/startup_inflate_check.py: the DirectStorage stand-in's zlib decoder on one stream.
add_executable(startup_inflate_probe tests/startup_inflate_probe.cpp)
target_compile_features(startup_inflate_probe PRIVATE cxx_std_17)
target_include_directories(startup_inflate_probe PRIVATE src)

add_executable(startup_dump_fence_check tests/startup_dump_fence.cpp tests/desktop_gpu.cpp)
add_executable(startup_overlay_window tests/startup_overlay_window.cpp)
target_compile_features(startup_overlay_window PRIVATE cxx_std_17)
target_include_directories(startup_overlay_window PRIVATE src)
target_link_libraries(startup_overlay_window PRIVATE user32)
add_test(NAME startup_overlay_window COMMAND startup_overlay_window)
# WASAPI render sink (Refs #1145); exit 77 when the machine has no usable render endpoint.
add_executable(startup_audio_output tests/startup_audio_output.cpp src/startup_stubs/audio_sink.cpp)
target_compile_features(startup_audio_output PRIVATE cxx_std_17)
target_include_directories(startup_audio_output PRIVATE src)
target_link_libraries(startup_audio_output PRIVATE ole32)
add_test(NAME startup_audio_output COMMAND startup_audio_output)
set_tests_properties(startup_audio_output PROPERTIES SKIP_RETURN_CODE 77 TIMEOUT 60)
# Playback-timing control: the same sink and endpoint as startup_audio_output, fed steady and bursty,
# recording first submission, frame counts, clock, drained interval and recovery for comparison.
add_executable(startup_audio_pacing tests/startup_audio_pacing.cpp src/startup_stubs/audio_sink.cpp)
target_compile_features(startup_audio_pacing PRIVATE cxx_std_17)
target_include_directories(startup_audio_pacing PRIVATE src)
target_link_libraries(startup_audio_pacing PRIVATE ole32)
add_test(NAME startup_audio_pacing COMMAND startup_audio_pacing)
set_tests_properties(startup_audio_pacing PROPERTIES SKIP_RETURN_CODE 77 TIMEOUT 90)
# Voice-lifecycle control: one client, two explicit providers (a diagnostic recorder and the system
# XAudio2 called directly), verifying the diagnostic window observes the same lifecycle before any
# timing number from it is used.
add_executable(startup_audio_provider tests/startup_audio_provider.cpp)
target_compile_features(startup_audio_provider PRIVATE cxx_std_17)
target_include_directories(startup_audio_provider PRIVATE src)
target_link_libraries(startup_audio_provider PRIVATE ole32 xaudio2)
add_test(NAME startup_audio_provider COMMAND startup_audio_provider)
set_tests_properties(startup_audio_provider PROPERTIES SKIP_RETURN_CODE 77 TIMEOUT 120)
target_compile_features(startup_dump_fence_check PRIVATE cxx_std_17)
target_include_directories(startup_dump_fence_check PRIVATE src)
target_link_libraries(startup_dump_fence_check PRIVATE d3d12 dxgi)
add_test(NAME startup_dump_fence_warp COMMAND startup_dump_fence_check)
set_tests_properties(startup_dump_fence_warp PROPERTIES ENVIRONMENT "X2W_SHIM_DRAW_ADAPTER=warp" TIMEOUT 45)
# The control expects a capture whose fence reports device removal to be published, and must fail.
add_test(NAME startup_dump_fence_removed_control_warp COMMAND startup_dump_fence_check --mutate removed)
set_tests_properties(startup_dump_fence_removed_control_warp PROPERTIES ENVIRONMENT "X2W_SHIM_DRAW_ADAPTER=warp" TIMEOUT 45
    WILL_FAIL TRUE)

add_executable(startup_stream_consumer tests/startup_stream_consumer.cpp tests/startup_stream_setup.cpp tests/startup_stream_draw.cpp
    tests/startup_address_srv.cpp)
target_compile_features(startup_stream_consumer PRIVATE cxx_std_17)
target_link_libraries(startup_stream_consumer PRIVATE d3d12_xs_shim startup_stub_xg_xs d3d12 dxgi)

add_executable(startup_stream_depth tests/startup_stream_depth.cpp tests/startup_stream_setup.cpp)
target_compile_features(startup_stream_depth PRIVATE cxx_std_17)
target_link_libraries(startup_stream_depth PRIVATE d3d12_xs_shim d3d12 dxgi)

# Console queue Wait (slot 17) waits on the GPU instead of signalling the fence.
add_executable(startup_queue_wait tests/startup_queue_wait.cpp)
target_compile_features(startup_queue_wait PRIVATE cxx_std_17)
target_link_libraries(startup_queue_wait PRIVATE d3d12_xs_shim d3d12 dxgi)
foreach(backend IN ITEMS warp hardware)
    add_test(NAME startup_queue_wait_${backend} COMMAND startup_queue_wait ${backend})
    set_tests_properties(startup_queue_wait_${backend} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
        ENVIRONMENT "X2W_D3D12_PERMISSIVE=1;X2W_D3D12_DEBUG=1")
endforeach()

# Closed lists stay submittable, also after their allocator's Reset (#1352).
add_executable(startup_list_resubmit tests/startup_list_resubmit.cpp)
target_compile_features(startup_list_resubmit PRIVATE cxx_std_17)
target_link_libraries(startup_list_resubmit PRIVATE d3d12_xs_shim d3d12 dxgi)
foreach(backend IN ITEMS warp hardware)
    add_test(NAME startup_list_resubmit_${backend} COMMAND startup_list_resubmit ${backend})
    set_tests_properties(startup_list_resubmit_${backend} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
        ENVIRONMENT "X2W_D3D12_PERMISSIVE=1;X2W_D3D12_LIST_NOOP=1;X2W_D3D12_RENDER=1;X2W_D3D12_DEBUG=1")
    # The same steps when no submit fence can be created: nothing may be reset or reused.
    add_test(NAME startup_list_resubmit_untracked_${backend} COMMAND startup_list_resubmit ${backend} untracked)
    set_tests_properties(startup_list_resubmit_untracked_${backend} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
        ENVIRONMENT "X2W_D3D12_PERMISSIVE=1;X2W_D3D12_LIST_NOOP=1;X2W_D3D12_RENDER=1;X2W_D3D12_DEBUG=1;X2W_TEST_SUBMIT_FENCE_FAIL=1")
endforeach()

add_executable(startup_nv12_draw tests/startup_nv12_draw.cpp tests/desktop_gpu.cpp)
target_compile_features(startup_nv12_draw PRIVATE cxx_std_17)
target_include_directories(startup_nv12_draw PRIVATE src)
target_link_libraries(startup_nv12_draw PRIVATE startup_stub_xg_xs d3d12 dxgi d3dcompiler)
add_executable(startup_plane_refresh tests/startup_plane_refresh.cpp tests/desktop_gpu.cpp)
target_compile_features(startup_plane_refresh PRIVATE cxx_std_17)
target_include_directories(startup_plane_refresh PRIVATE src)
target_link_libraries(startup_plane_refresh PRIVATE d3d12 dxgi)
foreach(backend IN ITEMS warp hardware)
    add_test(NAME startup_plane_refresh_${backend} COMMAND startup_plane_refresh)
    set_tests_properties(startup_plane_refresh_${backend} PROPERTIES
        ENVIRONMENT "X2W_SHIM_DRAW_ADAPTER=${backend}" TIMEOUT 60 SKIP_RETURN_CODE 77)
    add_test(NAME startup_nv12_draw_${backend} COMMAND startup_nv12_draw)
    set_tests_properties(startup_nv12_draw_${backend} PROPERTIES
        ENVIRONMENT "X2W_SHIM_DRAW_ADAPTER=${backend}" TIMEOUT 60 SKIP_RETURN_CODE 77)
endforeach()
# The same header without a GPU (src/startup_stubs/slice_footprints.h): accepted NV12 descriptions,
# re-read intervals and size cap, the footprint request and the bytes up to the end of the last row.
# The mutated run is the negative control.
add_executable(startup_slice_footprints_check tests/startup_slice_footprints_check.cpp)
target_include_directories(startup_slice_footprints_check PRIVATE src)
target_compile_features(startup_slice_footprints_check PRIVATE cxx_std_17)
add_test(NAME startup_slice_footprints COMMAND startup_slice_footprints_check)
add_test(NAME startup_slice_footprints_padded_control COMMAND startup_slice_footprints_check --mutate padded-last-row)
set_tests_properties(startup_slice_footprints_padded_control PROPERTIES WILL_FAIL TRUE)

find_program(DXC_EXECUTABLE dxc REQUIRED)
get_filename_component(DXC_BIN_DIR "${DXC_EXECUTABLE}" DIRECTORY)
target_compile_definitions(root_probe PRIVATE SDK_DXC_DIR=L"${DXC_BIN_DIR}/")
add_executable(offscreen_check tests/offscreen_check.cpp tests/compute_check.cpp tests/dxr_check.cpp tests/buffer_ops.cpp
    tests/descriptor_heap_paging.cpp)
target_sources(offscreen_check PRIVATE tests/spirv_window/readback.cpp tests/spirv_window/sampler_readback.cpp
    tests/spirv_window/compaction_readback.cpp tests/spirv_window/compare_sampler.cpp
    tests/spirv_window/raw_uav_dump.cpp)
# Two builds of one control shader that differ only in the sampler descriptor index source (#1115).
# The SV_DispatchThreadID build is the passing control; the SV_GroupID build is the recorded
# reproduction. Descriptor tables, shader model, output indexing and all three swaps stay identical.
set(sampler_index_dir "${CMAKE_CURRENT_BINARY_DIR}/sampler_index_control")
add_custom_command(OUTPUT "${sampler_index_dir}/dispatch_index_cs.h"
    COMMAND "${CMAKE_COMMAND}" -E make_directory "${sampler_index_dir}"
    COMMAND "${DXC_EXECUTABLE}" -T cs_6_0 -E main -Vn sampler_index_dispatch_cs
            -Fh "${sampler_index_dir}/dispatch_index_cs.h"
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/sampler_index_control.hlsl"
    DEPENDS tests/sampler_index_control.hlsl VERBATIM)
add_custom_command(OUTPUT "${sampler_index_dir}/group_index_cs.h"
    COMMAND "${CMAKE_COMMAND}" -E make_directory "${sampler_index_dir}"
    COMMAND "${DXC_EXECUTABLE}" -T cs_6_0 -E main -D INDEX_FROM_GROUP_ID=1 -Vn sampler_index_group_cs
            -Fh "${sampler_index_dir}/group_index_cs.h"
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/sampler_index_control.hlsl"
    DEPENDS tests/sampler_index_control.hlsl VERBATIM)
target_sources(offscreen_check PRIVATE tests/spirv_window/sampler_index_control.cpp
    "${sampler_index_dir}/dispatch_index_cs.h" "${sampler_index_dir}/group_index_cs.h")
target_include_directories(offscreen_check PRIVATE "${sampler_index_dir}")
# Record the compiler that produced both builds, so a reproduction can name it alongside the adapter.
execute_process(COMMAND "${DXC_EXECUTABLE}" --version
    OUTPUT_VARIABLE sampler_index_dxc_version OUTPUT_STRIP_TRAILING_WHITESPACE ERROR_QUIET)
string(REGEX MATCH "[0-9]+\\.[0-9]+\\.[0-9]+\\.[0-9]+" sampler_index_dxc_version "${sampler_index_dxc_version}")
if(sampler_index_dxc_version)
    target_compile_definitions(offscreen_check PRIVATE "DXC_VERSION_STRING=\"${sampler_index_dxc_version}\"")
endif()
foreach(adapter IN ITEMS warp hardware)
    add_test(NAME sampler_index_control_${adapter}
             COMMAND offscreen_check ${adapter} --sampler-index-control)
    set_tests_properties(sampler_index_control_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
endforeach()
target_compile_features(offscreen_check PRIVATE cxx_std_17)
# Link the production placement index into this fixture without adding a DLL export.
target_sources(offscreen_check PRIVATE src/startup_stubs/d3d12_xs_device.cpp)
set_property(SOURCE src/startup_stubs/d3d12_xs_device.cpp APPEND PROPERTY COMPILE_OPTIONS /bigobj)
target_link_libraries(offscreen_check PRIVATE d3d12 dxgi dxguid d3dcompiler user32 gdi32)
target_include_directories(offscreen_check PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
set(OFFSCREEN_AGILITY_620_DIR "" CACHE PATH "Agility SDK 1.620.2 x64 DLL directory for debug validation")
if(OFFSCREEN_AGILITY_620_DIR)
    if(EXISTS "${OFFSCREEN_AGILITY_620_DIR}/sdk.nupkg")
        set(agility_headers "${CMAKE_CURRENT_BINARY_DIR}/agility-headers")
        file(ARCHIVE_EXTRACT INPUT "${OFFSCREEN_AGILITY_620_DIR}/sdk.nupkg" DESTINATION "${agility_headers}"
             PATTERNS "build/native/include/*.h")
        target_include_directories(offscreen_check BEFORE PRIVATE "${agility_headers}/build/native/include")
    endif()
    foreach(dll IN ITEMS D3D12Core.dll d3d12SDKLayers.dll)
        configure_file("${OFFSCREEN_AGILITY_620_DIR}/${dll}" "${CMAKE_CURRENT_BINARY_DIR}/D3D12/${dll}" COPYONLY)
    endforeach()
    target_compile_definitions(offscreen_check PRIVATE OFFSCREEN_AGILITY_620)
endif()
set(OFFSCREEN_WARP_DIR "" CACHE PATH "Directory containing an app-local x64 d3d10warp.dll (Microsoft.Direct3D.WARP)")
if(OFFSCREEN_WARP_DIR)
    # On the Windows 10 in-box runtime, WARP 1.0.22 reports DXR tier 1.2 but every ray query misses.
    # With the Agility SDK runtime the same WARP passes the DXR checks, so require both together.
    if(NOT OFFSCREEN_AGILITY_620_DIR)
        message(FATAL_ERROR "OFFSCREEN_WARP_DIR requires OFFSCREEN_AGILITY_620_DIR. "
                "An app-local WARP on the in-box D3D12 runtime gives wrong ray query results.")
    endif()
endif()
include(cmake/app_local_warp.cmake)
offscreen_app_local_warp("${OFFSCREEN_WARP_DIR}" "${CMAKE_CURRENT_BINARY_DIR}")
add_test(NAME app_local_warp_toggle COMMAND "${CMAKE_COMMAND}" "-DSOURCE_DIR=${CMAKE_SOURCE_DIR}"
    "-DWORK_DIR=${CMAKE_CURRENT_BINARY_DIR}/app_local_warp_check" -P "${CMAKE_SOURCE_DIR}/tests/app_local_warp_check.cmake")
foreach(stage IN ITEMS vs ps)
    string(TOUPPER "${stage}" entry)
    set(header "${CMAKE_CURRENT_BINARY_DIR}/triangle_${stage}.h")
    set(binary "${CMAKE_CURRENT_BINARY_DIR}/triangle_${stage}.cso")
    add_custom_command(OUTPUT "${header}" "${binary}"
        COMMAND "${DXC_EXECUTABLE}" -T ${stage}_6_0 -E ${entry}Main -Vn triangle_${stage}
                -Fh "${header}" -Fo "${binary}" "${CMAKE_CURRENT_SOURCE_DIR}/tests/triangle.hlsl"
        DEPENDS tests/triangle.hlsl VERBATIM)
    target_sources(offscreen_check PRIVATE "${header}")
endforeach()
foreach(stage IN ITEMS vs ps)
    string(TOUPPER "${stage}" entry)
    set(flip_header "${CMAKE_CURRENT_BINARY_DIR}/triangle_${stage}_flip.h")
    add_custom_command(OUTPUT "${flip_header}"
        COMMAND "${DXC_EXECUTABLE}" -T ${stage}_6_0 -E ${entry}Main -DFLIP -Vn triangle_${stage}_flip
                -Fh "${flip_header}" "${CMAKE_CURRENT_SOURCE_DIR}/tests/triangle.hlsl"
        DEPENDS tests/triangle.hlsl VERBATIM)
endforeach()
add_custom_target(triangle_headers DEPENDS
    "${CMAKE_CURRENT_BINARY_DIR}/triangle_vs.h"
    "${CMAKE_CURRENT_BINARY_DIR}/triangle_ps.h"
    "${CMAKE_CURRENT_BINARY_DIR}/triangle_vs_flip.h"
    "${CMAKE_CURRENT_BINARY_DIR}/triangle_ps_flip.h")
add_dependencies(offscreen_check triangle_headers)
foreach(stage IN ITEMS vs ps)
    string(TOUPPER "${stage}" entry)
    set(tex_header "${CMAKE_CURRENT_BINARY_DIR}/tex_quad_${stage}.h")
    add_custom_command(OUTPUT "${tex_header}"
        COMMAND "${DXC_EXECUTABLE}" -T ${stage}_6_0 -E ${entry}Main -Vn tex_quad_${stage}
                -Fh "${tex_header}" "${CMAKE_CURRENT_SOURCE_DIR}/tests/tex_quad.hlsl"
        DEPENDS tests/tex_quad.hlsl VERBATIM)
endforeach()
set(ray_header "${CMAKE_CURRENT_BINARY_DIR}/ray_query.h")
set(pixel_vs "${CMAKE_CURRENT_BINARY_DIR}/pixel_aot_vs.h")
add_custom_command(OUTPUT "${pixel_vs}" COMMAND "${DXC_EXECUTABLE}" -T vs_6_0 -E main
    -Fh "${pixel_vs}" -Vn pixel_aot_vs "${CMAKE_CURRENT_SOURCE_DIR}/tests/pixel_aot_vs.hlsl"
    DEPENDS tests/pixel_aot_vs.hlsl VERBATIM)
set(pixel_vs_linear "${CMAKE_CURRENT_BINARY_DIR}/pixel_aot_vs_linear.h")
add_custom_command(OUTPUT "${pixel_vs_linear}" COMMAND "${DXC_EXECUTABLE}" -T vs_6_0 -E main -D PIXEL_MIP_LINEAR=1
    -Fh "${pixel_vs_linear}" -Vn pixel_aot_vs_linear "${CMAKE_CURRENT_SOURCE_DIR}/tests/pixel_aot_vs.hlsl"
    DEPENDS tests/pixel_aot_vs.hlsl VERBATIM)
target_sources(offscreen_check PRIVATE tests/pixel_aot_render.cpp tests/pixel_compare_render.cpp "${pixel_vs}"
    "${pixel_vs_linear}")
target_sources(offscreen_check PRIVATE tests/desktop_gpu.cpp tests/mesh_entry_draw.cpp tests/mesh_shadow_draw.cpp)
target_sources(offscreen_check PRIVATE tests/stream_root_srv.cpp)
foreach(stage IN ITEMS vs ps)
    string(TOUPPER "${stage}" entry)
    set(header "${CMAKE_CURRENT_BINARY_DIR}/stream_root_srv_${stage}.h")
    add_custom_command(OUTPUT "${header}"
        COMMAND "${DXC_EXECUTABLE}" -T ${stage}_6_0 -E ${entry}Main -Vn stream_root_srv_${stage}
                -Fh "${header}" "${CMAKE_CURRENT_SOURCE_DIR}/tests/stream_root_srv.hlsl"
        DEPENDS tests/stream_root_srv.hlsl VERBATIM)
    target_sources(offscreen_check PRIVATE "${header}")
endforeach()
set(descriptor_window_cs "${CMAKE_CURRENT_BINARY_DIR}/descriptor_window_submissions_cs.h")
add_custom_command(OUTPUT "${descriptor_window_cs}" COMMAND "${DXC_EXECUTABLE}" -T cs_6_0 -E main
    -Vn descriptor_window_submissions_cs -Fh "${descriptor_window_cs}"
    "${CMAKE_CURRENT_SOURCE_DIR}/tests/descriptor_window_submissions.hlsl"
    DEPENDS tests/descriptor_window_submissions.hlsl VERBATIM)
target_sources(offscreen_check PRIVATE tests/descriptor_window_submissions.cpp "${descriptor_window_cs}")
set(cpu_srv_cs "${CMAKE_CURRENT_BINARY_DIR}/cpu_srv_refresh_cs.h")
add_custom_command(OUTPUT "${cpu_srv_cs}" COMMAND "${DXC_EXECUTABLE}" -T cs_6_0 -E main
    -Vn cpu_srv_refresh_cs -Fh "${cpu_srv_cs}" "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu_srv_refresh.hlsl"
    DEPENDS tests/cpu_srv_refresh.hlsl VERBATIM)
target_sources(offscreen_check PRIVATE "${cpu_srv_cs}")
set(startup_sampler_x_cs "${CMAKE_CURRENT_BINARY_DIR}/startup_sampler_x_cs.h")
add_custom_command(OUTPUT "${startup_sampler_x_cs}" COMMAND "${DXC_EXECUTABLE}" -T cs_6_0 -E main
    -Vn startup_sampler_x_cs -Fh "${startup_sampler_x_cs}"
    "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_sampler_x.hlsl"
    DEPENDS tests/startup_sampler_x.hlsl VERBATIM)
target_sources(offscreen_check PRIVATE tests/startup_sampler_x.cpp "${startup_sampler_x_cs}")
target_sources(offscreen_check PRIVATE tests/startup_sampler_heap.cpp)
# Sampler word decoding, manifest lines and title sampler snapshots behind those draws
# (src/startup_stubs/translated_sampler_bindings.h), without a device. The mutated run is the negative control.
add_executable(startup_translated_sampler_bindings_check tests/startup_translated_sampler_bindings_check.cpp)
target_include_directories(startup_translated_sampler_bindings_check PRIVATE src)
target_compile_features(startup_translated_sampler_bindings_check PRIVATE cxx_std_17)
add_test(NAME startup_translated_sampler_bindings COMMAND startup_translated_sampler_bindings_check)
add_test(NAME startup_translated_sampler_bindings_shared_axis_control
    COMMAND startup_translated_sampler_bindings_check --mutate shared-axis)
set_tests_properties(startup_translated_sampler_bindings_shared_axis_control PROPERTIES WILL_FAIL TRUE)
target_sources(offscreen_check PRIVATE tests/startup_image_3d.cpp)
target_sources(offscreen_check PRIVATE tests/startup_flat_param.cpp)
# The storage root handed to the executable ends in a separator and is never cut off
# (src/startup_stubs/storage_root.h); the mutated run is the negative control.
add_executable(startup_storage_root_check tests/startup_storage_root_check.cpp)
target_include_directories(startup_storage_root_check PRIVATE src)
target_compile_features(startup_storage_root_check PRIVATE cxx_std_17)
add_test(NAME startup_storage_root COMMAND startup_storage_root_check)
add_test(NAME startup_storage_root_separator_control COMMAND startup_storage_root_check --mutate separator)
set_tests_properties(startup_storage_root_separator_control PROPERTIES WILL_FAIL TRUE)
# Stand-in format for 16-bit packed targets the adapter cannot use (src/startup_stubs/packed_target_format.h).
add_executable(startup_packed_target_check tests/startup_packed_target_check.cpp)
target_include_directories(startup_packed_target_check PRIVATE src)
target_compile_features(startup_packed_target_check PRIVATE cxx_std_17)
add_test(NAME startup_packed_target COMMAND startup_packed_target_check)
foreach(mutation IN ITEMS stand-in array b565 view tag)
    add_test(NAME startup_packed_target_${mutation}_control COMMAND startup_packed_target_check --mutate ${mutation})
    set_tests_properties(startup_packed_target_${mutation}_control PROPERTIES WILL_FAIL TRUE)
endforeach()
# Static samplers built from smp and smpc manifest lines (src/startup_stubs/translated_sampler_bindings.h).
add_executable(startup_sampler_comparison_check tests/startup_sampler_comparison_check.cpp)
target_include_directories(startup_sampler_comparison_check PRIVATE src)
target_compile_features(startup_sampler_comparison_check PRIVATE cxx_std_17)
add_test(NAME startup_sampler_comparison COMMAND startup_sampler_comparison_check)
# Diagnostic line for a refused device slot 70 call (src/startup_stubs/memory_srv_report.h).
add_executable(startup_memory_srv_report_check tests/startup_memory_srv_report_check.cpp)
target_include_directories(startup_memory_srv_report_check PRIVATE src)
target_compile_features(startup_memory_srv_report_check PRIVATE cxx_std_17)
add_test(NAME startup_memory_srv_report COMMAND startup_memory_srv_report_check)

# View format of the window presenter's read of typeless textures (src/startup_stubs/present_view_format.h);
# the identity run is the old behaviour and the sRGB run is a wrong view; both are negative controls.
add_executable(startup_present_view_check tests/startup_present_view_check.cpp)
target_include_directories(startup_present_view_check PRIVATE src)
target_compile_features(startup_present_view_check PRIVATE cxx_std_17)
target_link_libraries(startup_present_view_check PRIVATE d3d12 dxgi d3dcompiler)
add_test(NAME startup_present_view COMMAND startup_present_view_check)
add_test(NAME startup_present_view_identity_control COMMAND startup_present_view_check --mutate identity)
foreach(adapter IN ITEMS warp hardware)
    add_test(NAME startup_present_view_${adapter} COMMAND startup_present_view_check ${adapter})
endforeach()
add_test(NAME startup_present_view_srgb_control COMMAND startup_present_view_check warp --mutate srgb)
set_tests_properties(startup_present_view_identity_control startup_present_view_srgb_control PROPERTIES WILL_FAIL TRUE)
set_tests_properties(startup_present_view_hardware PROPERTIES SKIP_RETURN_CODE 77)
# Buffer and raytracing acceleration-structure memory SRV validation (src/startup_stubs/memory_srv_buffer.h).
add_executable(startup_memory_srv_buffer_check tests/startup_memory_srv_buffer_check.cpp)
target_include_directories(startup_memory_srv_buffer_check PRIVATE src)
target_compile_features(startup_memory_srv_buffer_check PRIVATE cxx_std_17)
add_test(NAME startup_memory_srv_buffer COMMAND startup_memory_srv_buffer_check)
# Console allocation sizes for depth-with-stencil and multisampled textures
# (src/startup_stubs/console_plane_size.h); the mutated run is the negative control.
add_executable(startup_plane_size_check tests/startup_plane_size_check.cpp)
target_include_directories(startup_plane_size_check PRIVATE src)
target_compile_features(startup_plane_size_check PRIVATE cxx_std_17)
add_test(NAME startup_plane_size COMMAND startup_plane_size_check)
add_test(NAME startup_plane_size_packed_control COMMAND startup_plane_size_check --mutate packed)
set_tests_properties(startup_plane_size_packed_control PROPERTIES WILL_FAIL TRUE)
# GameInput stand-in timestamps: a performance-counter reading in microseconds without a product
# that wraps (src/startup_stubs/qpc_microseconds.h); the mutated run is the negative control.
add_executable(startup_qpc_microseconds_check tests/startup_qpc_microseconds_check.cpp)
target_include_directories(startup_qpc_microseconds_check PRIVATE src)
target_compile_features(startup_qpc_microseconds_check PRIVATE cxx_std_17)
add_test(NAME startup_qpc_microseconds COMMAND startup_qpc_microseconds_check)
add_test(NAME startup_qpc_microseconds_product_control COMMAND startup_qpc_microseconds_check --mutate product)
set_tests_properties(startup_qpc_microseconds_product_control PROPERTIES WILL_FAIL TRUE)
# The blend patch a slot 55 derivation applies (src/startup_stubs/derived_blend.h): the last part-2
# record, its descriptor offset and bytes, and lists that are not whole records. The mutated run is
# the negative control.
add_executable(startup_derived_blend_check tests/startup_derived_blend_check.cpp)
target_include_directories(startup_derived_blend_check PRIVATE src)
target_compile_features(startup_derived_blend_check PRIVATE cxx_std_17)
add_test(NAME startup_derived_blend COMMAND startup_derived_blend_check)
add_test(NAME startup_derived_blend_first_wins_control COMMAND startup_derived_blend_check --mutate first-wins)
set_tests_properties(startup_derived_blend_first_wins_control PROPERTIES WILL_FAIL TRUE)
# The same rule through the device wrapper: allocation-info slots 25 and 108, then a placed texture
# (slot 133) over memory of the answered size.
add_executable(startup_plane_size_device_check tests/startup_plane_size_device_check.cpp)
target_compile_features(startup_plane_size_device_check PRIVATE cxx_std_17)
target_link_libraries(startup_plane_size_device_check PRIVATE d3d12_xs_shim d3d12 dxgi)
foreach(adapter IN ITEMS warp hardware)
    add_test(NAME startup_plane_size_device_${adapter} COMMAND startup_plane_size_device_check ${adapter})
endforeach()
set_tests_properties(startup_plane_size_device_hardware PROPERTIES SKIP_RETURN_CODE 77)
# Opt-in BC7 4 KiB standard surfaces (src/startup_stubs/bc7_4k_s.h): accepted descriptions, padded
# sizes, block order and detile; the mutated run (x-first bit order) is the negative control.
add_executable(startup_bc7_4k_s_check tests/startup_bc7_4k_s_check.cpp)
target_include_directories(startup_bc7_4k_s_check PRIVATE src)
target_compile_features(startup_bc7_4k_s_check PRIVATE cxx_std_17)
add_test(NAME startup_bc7_4k_s COMMAND startup_bc7_4k_s_check)
add_test(NAME startup_bc7_4k_s_x_first_control COMMAND startup_bc7_4k_s_check --mutate x-first)
set_tests_properties(startup_bc7_4k_s_x_first_control PROPERTIES WILL_FAIL TRUE)
# Descriptor heaps whose desktop CPU start is not an address get plain memory as the start the
# executable sees (src/startup_stubs/title_heap_memory.h); the mutated run is the negative control.
add_executable(startup_title_heap_check tests/startup_title_heap_check.cpp)
target_compile_features(startup_title_heap_check PRIVATE cxx_std_17)
target_link_libraries(startup_title_heap_check PRIVATE d3d12_xs_shim d3d12 dxgi)
add_test(NAME startup_title_heap COMMAND startup_title_heap_check)
add_test(NAME startup_title_heap_tags_control COMMAND startup_title_heap_check --mutate tags)
set_tests_properties(startup_title_heap_tags_control PROPERTIES WILL_FAIL TRUE)
foreach(adapter IN ITEMS warp hardware)
    add_test(NAME startup_title_heap_device_${adapter} COMMAND startup_title_heap_check ${adapter})
endforeach()
set_tests_properties(startup_title_heap_device_hardware PROPERTIES SKIP_RETURN_CODE 77)
# Blend state of slot 57 pipelines from state block A (src/startup_stubs/state_block_blend.h, #1216).
add_executable(startup_state_block_blend_check tests/startup_state_block_blend_check.cpp)
target_include_directories(startup_state_block_blend_check PRIVATE src)
target_compile_features(startup_state_block_blend_check PRIVATE cxx_std_17)
add_test(NAME startup_state_block_blend COMMAND startup_state_block_blend_check)
# Root table handles resolved against a logical view heap, and the records a translated draw reads
# from the window (src/startup_stubs/logical_view_table.h).
add_executable(startup_logical_view_table_check tests/startup_logical_view_table_check.cpp)
target_include_directories(startup_logical_view_table_check PRIVATE src)
target_compile_features(startup_logical_view_table_check PRIVATE cxx_std_17)
add_test(NAME startup_logical_view_table COMMAND startup_logical_view_table_check)
# Local switch, logging, and page-probe helpers (src/startup_stubs/local_log.h, #1149).
add_executable(startup_local_log_check tests/startup_local_log_check.cpp)
target_include_directories(startup_local_log_check PRIVATE src)
target_compile_features(startup_local_log_check PRIVATE cxx_std_17)
add_test(NAME startup_local_log COMMAND startup_local_log_check)
add_test(NAME startup_local_log_guard_drop_control COMMAND startup_local_log_check --mutate guard-drop)
set_tests_properties(startup_local_log_guard_drop_control PROPERTIES WILL_FAIL TRUE)

# Colour-blend register decoding (src/startup_stubs/cb_blend.h, #1216): decoder table plus draws
# compared with a CPU model of the register fields.
set(startup_cb_blend_vs "${CMAKE_CURRENT_BINARY_DIR}/startup_cb_blend_vs.h")
set(startup_cb_blend_ps "${CMAKE_CURRENT_BINARY_DIR}/startup_cb_blend_ps.h")
add_custom_command(OUTPUT "${startup_cb_blend_vs}" COMMAND "${DXC_EXECUTABLE}" -T vs_6_0 -E vs_main
    -Vn startup_cb_blend_vs -Fh "${startup_cb_blend_vs}" "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_cb_blend.hlsl"
    DEPENDS tests/startup_cb_blend.hlsl VERBATIM)
add_custom_command(OUTPUT "${startup_cb_blend_ps}" COMMAND "${DXC_EXECUTABLE}" -T ps_6_0 -E ps_main
    -Vn startup_cb_blend_ps -Fh "${startup_cb_blend_ps}" "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_cb_blend.hlsl"
    DEPENDS tests/startup_cb_blend.hlsl VERBATIM)
target_sources(offscreen_check PRIVATE tests/startup_cb_blend.cpp "${startup_cb_blend_vs}" "${startup_cb_blend_ps}")
# Composite provenance (#1628): one VS and two pixel shaders from one HLSL file.
set(startup_composite_provenance_headers "")
foreach(stage IN ITEMS "vs_6_0;vs_main;vs" "ps_6_0;ps_fill;fill" "ps_6_0;ps_composite;composite")
    list(GET stage 0 profile)
    list(GET stage 1 entry)
    list(GET stage 2 suffix)
    set(header "${CMAKE_CURRENT_BINARY_DIR}/startup_composite_provenance_${suffix}.h")
    add_custom_command(OUTPUT "${header}" COMMAND "${DXC_EXECUTABLE}" -T ${profile} -E ${entry}
        -Vn startup_composite_provenance_${suffix} -Fh "${header}"
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_composite_provenance.hlsl"
        DEPENDS tests/startup_composite_provenance.hlsl VERBATIM)
    list(APPEND startup_composite_provenance_headers "${header}")
endforeach()
target_sources(offscreen_check PRIVATE tests/startup_composite_provenance.cpp ${startup_composite_provenance_headers})
target_sources(offscreen_check PRIVATE tests/binding_mips.cpp tests/ffx_recurrent_views.cpp)
target_sources(offscreen_check PRIVATE tests/ffx_rcas_views.cpp)
foreach(storage IN ITEMS Native Packed)
    foreach(op IN ITEMS Read Write)
        set(symbol "Rcas${op}${storage}")
        set(header "${CMAKE_CURRENT_BINARY_DIR}/${symbol}.h")
        add_custom_command(OUTPUT "${header}"
            COMMAND "${DXC_EXECUTABLE}" -T cs_6_0 -E "Rcas${op}" -D "STORAGE_${storage}=1"
                -Vn "${symbol}" -Fh "${header}" -Fo "${CMAKE_CURRENT_BINARY_DIR}/${symbol}.cso"
                "${CMAKE_CURRENT_SOURCE_DIR}/tests/ffx_rcas.hlsl"
            DEPENDS tests/ffx_rcas.hlsl src/rgb9e5.hlsli BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/${symbol}.cso" VERBATIM)
        target_sources(offscreen_check PRIVATE "${header}")
    endforeach()
endforeach()
foreach(entry IN ITEMS DualUav SampleMip WriteRecurrent DynamicSamplerIndex)
    set(header "${CMAKE_CURRENT_BINARY_DIR}/${entry}.h")
    add_custom_command(OUTPUT "${header}" COMMAND "${DXC_EXECUTABLE}" -T cs_6_0 -E ${entry}
        -Vn ${entry} -Fh "${header}" -Fo "${CMAKE_CURRENT_BINARY_DIR}/${entry}.cso"
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/binding_mips.hlsl" DEPENDS tests/binding_mips.hlsl
        BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/${entry}.cso" VERBATIM)
    target_sources(offscreen_check PRIVATE "${header}")
endforeach()
target_sources(offscreen_check PRIVATE tests/bvh_reads_waves.cpp tests/bvh_hits.cpp)
target_sources(offscreen_check PRIVATE tests/bvh_atomics.cpp tests/bvh_numeric.cpp)
# A title graphics allocation opened as a D3D12 heap and addressed through translated GPU addresses.
target_sources(offscreen_check PRIVATE tests/gpu_alias.cpp src/era_gpu_alias.cpp src/era_title_va.cpp src/era_heap.cpp)
# A title buffer window: two 64 KiB pages, a non-page-aligned offset, copies in both directions and
# named rejects for an outside address, a misaligned page run and an uncommitted page.
target_sources(offscreen_check PRIVATE tests/gpu_alias_window.cpp)
# The same allocations read by a shader through 64-bit title addresses (src/title_va_lookup.hlsli).
target_sources(offscreen_check PRIVATE tests/depth_mask_draw.cpp)
target_sources(offscreen_check PRIVATE tests/gpu_alias_lookup.cpp)
# Graphics allocations given desktop GPU addresses when a device is attached (src/era_title_gpu.cpp).
target_sources(offscreen_check PRIVATE tests/gpu_alias_registry.cpp src/era_title_gpu.cpp src/era_query.cpp)
add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/LookupTitleAddresses.h" COMMAND "${DXC_EXECUTABLE}" -T cs_6_0
    -E LookupTitleAddresses -Vn LookupTitleAddresses -I "${CMAKE_CURRENT_SOURCE_DIR}/src"
    -Fh "${CMAKE_CURRENT_BINARY_DIR}/LookupTitleAddresses.h" -Fo "${CMAKE_CURRENT_BINARY_DIR}/LookupTitleAddresses.cso"
    "${CMAKE_CURRENT_SOURCE_DIR}/tests/gpu_alias_lookup.hlsl" DEPENDS tests/gpu_alias_lookup.hlsl src/title_va_lookup.hlsli
    BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/LookupTitleAddresses.cso" VERBATIM)
target_sources(offscreen_check PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/LookupTitleAddresses.h")
add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/WriteTitleAddresses.h" COMMAND "${DXC_EXECUTABLE}" -T cs_6_0
    -E WriteTitleAddresses -Vn WriteTitleAddresses -I "${CMAKE_CURRENT_SOURCE_DIR}/src"
    -Fh "${CMAKE_CURRENT_BINARY_DIR}/WriteTitleAddresses.h" -Fo "${CMAKE_CURRENT_BINARY_DIR}/WriteTitleAddresses.cso"
    "${CMAKE_CURRENT_SOURCE_DIR}/tests/gpu_alias_writes.hlsl" DEPENDS tests/gpu_alias_writes.hlsl src/title_va_lookup.hlsli
    BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/WriteTitleAddresses.cso" VERBATIM)
target_sources(offscreen_check PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/WriteTitleAddresses.h")
add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/ProbeArrayAtomics.h" COMMAND "${DXC_EXECUTABLE}" -T cs_6_0
    -E ProbeArrayAtomics -Vn ProbeArrayAtomics -I "${CMAKE_CURRENT_SOURCE_DIR}/src"
    -Fh "${CMAKE_CURRENT_BINARY_DIR}/ProbeArrayAtomics.h" -Fo "${CMAKE_CURRENT_BINARY_DIR}/ProbeArrayAtomics.cso"
    "${CMAKE_CURRENT_SOURCE_DIR}/tests/gpu_alias_writes.hlsl" DEPENDS tests/gpu_alias_writes.hlsl src/title_va_lookup.hlsli
    BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/ProbeArrayAtomics.cso" VERBATIM)
target_sources(offscreen_check PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/ProbeArrayAtomics.h")
add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/ProbeArrayAtomicsWaterfall.h" COMMAND "${DXC_EXECUTABLE}" -T cs_6_0
    -E ProbeArrayAtomicsWaterfall -Vn ProbeArrayAtomicsWaterfall
    -Fh "${CMAKE_CURRENT_BINARY_DIR}/ProbeArrayAtomicsWaterfall.h"
    -Fo "${CMAKE_CURRENT_BINARY_DIR}/ProbeArrayAtomicsWaterfall.cso"
    "${CMAKE_CURRENT_SOURCE_DIR}/tests/gpu_alias_atomic_waterfall.hlsl"
    DEPENDS tests/gpu_alias_atomic_waterfall.hlsl
    BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/ProbeArrayAtomicsWaterfall.cso" VERBATIM)
target_sources(offscreen_check PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/ProbeArrayAtomicsWaterfall.h")
target_include_directories(offscreen_check PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/src")
add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/AliasTransform.h" COMMAND "${DXC_EXECUTABLE}" -T cs_6_0
    -E AliasTransform -Vn AliasTransform -Fh "${CMAKE_CURRENT_BINARY_DIR}/AliasTransform.h"
    -Fo "${CMAKE_CURRENT_BINARY_DIR}/AliasTransform.cso" "${CMAKE_CURRENT_SOURCE_DIR}/tests/gpu_alias.hlsl"
    DEPENDS tests/gpu_alias.hlsl BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/AliasTransform.cso" VERBATIM)
target_sources(offscreen_check PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/AliasTransform.h")
foreach(entry IN ITEMS InitBounds AtomicBounds QueryBounds NumericBounds)
    set(bounds_source tests/bvh_atomics.hlsl)
    if(entry STREQUAL "NumericBounds")
        set(bounds_source tests/bvh_numeric.hlsl)
    endif()
    set(header "${CMAKE_CURRENT_BINARY_DIR}/${entry}.h")
    add_custom_command(OUTPUT "${header}" COMMAND "${DXC_EXECUTABLE}" -T cs_6_5 -E ${entry}
        -Vn ${entry} -Fh "${header}" -Fo "${CMAKE_CURRENT_BINARY_DIR}/${entry}.cso"
        "${CMAKE_CURRENT_SOURCE_DIR}/${bounds_source}" DEPENDS "${bounds_source}"
        BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/${entry}.cso" VERBATIM)
    target_sources(offscreen_check PRIVATE "${header}")
endforeach()
foreach(entry IN ITEMS Produce ReadValues WaveValues Wave64 WriteScene ExpandLeaves TraceHits)
    set(header "${CMAKE_CURRENT_BINARY_DIR}/${entry}.h")
    set(wave_args -T cs_6_5 -E ${entry})
    if(entry STREQUAL "Wave64")
        set(wave_args -T cs_6_6 -E WaveValues -D REQUIRE_WAVE64=1)
    endif()
    add_custom_command(OUTPUT "${header}" COMMAND "${DXC_EXECUTABLE}" ${wave_args} -Vn ${entry}
        -Fh "${header}" -Fo "${CMAKE_CURRENT_BINARY_DIR}/${entry}.cso"
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/bvh_reads_waves.hlsl" DEPENDS tests/bvh_reads_waves.hlsl
        BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/${entry}.cso" VERBATIM)
    target_sources(offscreen_check PRIVATE "${header}")
endforeach()
target_sources(offscreen_check PRIVATE tests/wave_shuffle.cpp)
foreach(width IN ITEMS 0 16 32 64)
    if(width)
        set(entry WaveShuffle${width})
        set(shuffle_args -T cs_6_6 -D WAVE_SIZE=${width})
    else()
        set(entry WaveShuffle)
        set(shuffle_args -T cs_6_5)
    endif()
    set(header "${CMAKE_CURRENT_BINARY_DIR}/${entry}.h")
    add_custom_command(OUTPUT "${header}" COMMAND "${DXC_EXECUTABLE}" ${shuffle_args} -E Shuffle -Vn ${entry}
        -Fh "${header}" -Fo "${CMAKE_CURRENT_BINARY_DIR}/${entry}.cso"
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/wave_shuffle.hlsl" DEPENDS tests/wave_shuffle.hlsl
        BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/${entry}.cso" VERBATIM)
    target_sources(offscreen_check PRIVATE "${header}")
endforeach()
# 64-lane wave operations emulated with group-shared memory (src/wave64_emulate.hlsli).
target_sources(offscreen_check PRIVATE tests/wave64_emulate.cpp)
foreach(entry IN ITEMS Wave64Emulate Wave64Emulate64 Wave64EmulateSkew)
    set(emulate_args -T cs_6_0)
    if(entry STREQUAL "Wave64Emulate64")
        set(emulate_args -T cs_6_6 -D WAVE_SIZE=64)
    elseif(entry STREQUAL "Wave64EmulateSkew")
        list(APPEND emulate_args -D SKEW=1)
    endif()
    set(header "${CMAKE_CURRENT_BINARY_DIR}/${entry}.h")
    add_custom_command(OUTPUT "${header}" COMMAND "${DXC_EXECUTABLE}" ${emulate_args} -E Emulate -Vn ${entry}
        -I "${CMAKE_CURRENT_SOURCE_DIR}/src" -Fh "${header}" -Fo "${CMAKE_CURRENT_BINARY_DIR}/${entry}.cso"
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/wave64_emulate.hlsl" DEPENDS tests/wave64_emulate.hlsl src/wave64_emulate.hlsli
        BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/${entry}.cso" VERBATIM)
    target_sources(offscreen_check PRIVATE "${header}")
endforeach()
target_sources(offscreen_check PRIVATE tests/shader_wave.cpp)
foreach(width IN ITEMS 0 32 64)
    if(width)
        set(entry ShaderWave${width})
        set(wave_args -T cs_6_6 -D WAVE_SIZE=${width})
    else()
        set(entry ShaderWave)
        set(wave_args -T cs_6_0)
    endif()
    set(header "${CMAKE_CURRENT_BINARY_DIR}/${entry}.h")
    add_custom_command(OUTPUT "${header}" COMMAND "${DXC_EXECUTABLE}" ${wave_args} -E WaveWidth -Vn ${entry}
        -Fh "${header}" -Fo "${CMAKE_CURRENT_BINARY_DIR}/${entry}.cso"
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/shader_wave.hlsl" DEPENDS tests/shader_wave.hlsl
        BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/${entry}.cso" VERBATIM)
    target_sources(offscreen_check PRIVATE "${header}")
endforeach()
foreach(entry IN ITEMS CopyBlocks FillBlocks NumberBlocks CopySpan ResetBounds)
    set(header "${CMAKE_CURRENT_BINARY_DIR}/${entry}.h")
    add_custom_command(OUTPUT "${header}" COMMAND "${DXC_EXECUTABLE}" -T cs_6_0 -E ${entry} -Vn ${entry}
        -Fh "${header}" -Fo "${CMAKE_CURRENT_BINARY_DIR}/${entry}.cso"
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/buffer_ops.hlsl" DEPENDS tests/buffer_ops.hlsl
        BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/${entry}.cso" VERBATIM)
    target_sources(offscreen_check PRIVATE "${header}")
endforeach()
add_custom_command(OUTPUT "${ray_header}"
    COMMAND "${DXC_EXECUTABLE}" -T cs_6_5 -E CSMain -Vn ray_query -Fh "${ray_header}"
            -Fo "${CMAKE_CURRENT_BINARY_DIR}/ray_query.cso" "${CMAKE_CURRENT_SOURCE_DIR}/tests/ray_query.hlsl"
    BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/ray_query.cso" DEPENDS tests/ray_query.hlsl VERBATIM)
set(ray_pipeline_header "${CMAKE_CURRENT_BINARY_DIR}/ray_pipeline.h")
add_custom_command(OUTPUT "${ray_pipeline_header}"
    COMMAND "${DXC_EXECUTABLE}" -T lib_6_3 -D DXR_PIPELINE=1 -Vn ray_pipeline -Fh "${ray_pipeline_header}"
            -Fo "${CMAKE_CURRENT_BINARY_DIR}/ray_pipeline.cso" "${CMAKE_CURRENT_SOURCE_DIR}/tests/ray_query.hlsl"
    BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/ray_pipeline.cso" DEPENDS tests/ray_query.hlsl VERBATIM)
set(candidates_header "${CMAKE_CURRENT_BINARY_DIR}/ray_candidates.h")
add_custom_command(OUTPUT "${candidates_header}"
    COMMAND "${DXC_EXECUTABLE}" -T cs_6_5 -E CSMain -Vn ray_candidates -Fh "${candidates_header}"
            -Fo "${CMAKE_CURRENT_BINARY_DIR}/ray_candidates.cso" "${CMAKE_CURRENT_SOURCE_DIR}/tests/ray_candidates.hlsl"
    BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/ray_candidates.cso" DEPENDS tests/ray_candidates.hlsl VERBATIM)
set(candidates_pipeline_header "${CMAKE_CURRENT_BINARY_DIR}/ray_candidates_pipeline.h")
add_custom_command(OUTPUT "${candidates_pipeline_header}"
    COMMAND "${DXC_EXECUTABLE}" -T lib_6_3 -D DXR_PIPELINE=1 -Vn ray_candidates_pipeline -Fh "${candidates_pipeline_header}"
            -Fo "${CMAKE_CURRENT_BINARY_DIR}/ray_candidates_pipeline.cso" "${CMAKE_CURRENT_SOURCE_DIR}/tests/ray_candidates.hlsl"
    BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/ray_candidates_pipeline.cso" DEPENDS tests/ray_candidates.hlsl VERBATIM)
target_sources(offscreen_check PRIVATE "${ray_header}" "${ray_pipeline_header}" "${candidates_header}" "${candidates_pipeline_header}"
    tests/dxr_facing.cpp tests/dxr_opacity.cpp)
foreach(layout IN ITEMS root tables)
    set(header "${CMAKE_CURRENT_BINARY_DIR}/compute_${layout}.h")
    set(binary "${CMAKE_CURRENT_BINARY_DIR}/compute_${layout}.cso")
    set(defines)
    if(layout STREQUAL "tables")
        set(defines -D TABLE_BINDINGS=1)
    endif()
    add_custom_command(OUTPUT "${header}" "${binary}"
        COMMAND "${DXC_EXECUTABLE}" -T cs_6_0 -E CSMain ${defines} -Vn compute_${layout}
                -Fh "${header}" -Fo "${binary}" "${CMAKE_CURRENT_SOURCE_DIR}/tests/compute.hlsl"
        DEPENDS tests/compute.hlsl VERBATIM)
    target_sources(offscreen_check PRIVATE "${header}")
    foreach(adapter IN ITEMS hardware warp)
        add_test(NAME compute_${layout}_${adapter} COMMAND offscreen_check ${adapter} --compute-${layout})
        set_tests_properties(compute_${layout}_${adapter} PROPERTIES TIMEOUT 45)
    endforeach()
    set_tests_properties(compute_${layout}_hardware PROPERTIES SKIP_RETURN_CODE 77)
endforeach()
# XBOX2WINDOWS_ADAPTER selects a hardware adapter by part of its name. A name no adapter has must fail.
add_test(NAME adapter_selection_unmatched COMMAND offscreen_check hardware --dual-uav)
set_tests_properties(adapter_selection_unmatched PROPERTIES TIMEOUT 45
    ENVIRONMENT "XBOX2WINDOWS_ADAPTER=no adapter has this name"
    PASS_REGULAR_EXPRESSION "No D3D12 hardware adapter matches XBOX2WINDOWS_ADAPTER=no adapter has this name")
add_executable(pso_root_check tests/pso_root_check.cpp)
target_compile_features(pso_root_check PRIVATE cxx_std_17)
target_include_directories(pso_root_check PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
target_link_libraries(pso_root_check PRIVATE d3d12 dxgi)
foreach(mode IN ITEMS embedded stripped)
    set(symbol pso_compute_${mode})
    set(header "${CMAKE_CURRENT_BINARY_DIR}/${symbol}.h")
    set(strip_root)
    if(mode STREQUAL "stripped")
        set(strip_root -Qstrip_rootsignature)
    endif()
    add_custom_command(OUTPUT "${header}"
        COMMAND "${DXC_EXECUTABLE}" -T cs_6_0 -E CSMain ${strip_root} -Vn ${symbol}
                -Fh "${header}" "${CMAKE_CURRENT_SOURCE_DIR}/tests/compute.hlsl"
        DEPENDS tests/compute.hlsl VERBATIM)
    target_sources(pso_root_check PRIVATE "${header}")
endforeach()
add_test(NAME pso_root_signature_warp COMMAND pso_root_check)
add_custom_target(pso_compute_headers DEPENDS
    "${CMAKE_CURRENT_BINARY_DIR}/pso_compute_embedded.h"
    "${CMAKE_CURRENT_BINARY_DIR}/pso_compute_stripped.h")
foreach(stage IN ITEMS vs ps)
    string(TOUPPER "${stage}" entry)
    set(rect_header "${CMAKE_CURRENT_BINARY_DIR}/rectlist_${stage}.h")
    add_custom_command(OUTPUT "${rect_header}"
        COMMAND "${DXC_EXECUTABLE}" -T ${stage}_6_0 -E ${entry}Main -Vn rectlist_${stage}
                -Fh "${rect_header}" "${CMAKE_CURRENT_SOURCE_DIR}/tests/rectlist_quad.hlsl"
        DEPENDS tests/rectlist_quad.hlsl VERBATIM)
    set(rectlist_shader_${stage} "${rect_header}")
endforeach()
add_custom_target(rectlist_headers DEPENDS
    "${rectlist_shader_vs}"
    "${rectlist_shader_ps}")
add_executable(rectlist_draw_check tests/rectlist_draw_check.cpp
    "${rectlist_shader_vs}"
    "${rectlist_shader_ps}")
add_dependencies(rectlist_draw_check rectlist_headers)
target_compile_features(rectlist_draw_check PRIVATE cxx_std_17)
target_include_directories(rectlist_draw_check PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
target_link_libraries(rectlist_draw_check PRIVATE d3d12 dxgi)
add_executable(d3d12_xs_shim_check tests/d3d12_xs_shim_check.cpp)
add_dependencies(d3d12_xs_shim_check pso_compute_headers)
target_link_libraries(d3d12_xs_shim_check PRIVATE d3d12_xs_shim dxgi)
target_include_directories(d3d12_xs_shim_check PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
target_compile_features(d3d12_xs_shim_check PRIVATE cxx_std_17)
foreach(scenario IN ITEMS exports serialize deserialize iids warp_root)
    add_test(NAME d3d12_xs_shim_${scenario} COMMAND d3d12_xs_shim_check ${scenario})
endforeach()
# D3D12XboxCreateDevice device-slot inventory and source-built creation fixture (#XCL-08).
set(d3d12x_pso_header "${CMAKE_CURRENT_BINARY_DIR}/d3d12x_pso_cs.h")
add_custom_command(OUTPUT "${d3d12x_pso_header}"
    COMMAND "${DXC_EXECUTABLE}" -T cs_6_0 -E CSMain -Vn d3d12x_pso_cs
            -Fh "${d3d12x_pso_header}" -Fo "${CMAKE_CURRENT_BINARY_DIR}/d3d12x_pso_cs.cso"
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/d3d12x_pso.hlsl"
    DEPENDS tests/d3d12x_pso.hlsl
    BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/d3d12x_pso_cs.cso" VERBATIM)
add_executable(d3d12x_device_check tests/d3d12x_device_check.cpp src/d3d12x_struct_return.cpp "${d3d12x_pso_header}")
add_dependencies(d3d12x_device_check d3d12_xs_shim)
target_include_directories(d3d12x_device_check PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
target_compile_features(d3d12x_device_check PRIVATE cxx_std_17)
target_include_directories(d3d12x_device_check PRIVATE src)
target_link_libraries(d3d12x_device_check PRIVATE d3d12 dxgi)
foreach(adapter IN ITEMS warp hardware)
    add_test(NAME d3d12x_device_create_${adapter} COMMAND d3d12x_device_check
        "$<TARGET_FILE:d3d12_xs_shim>" create ${adapter})
    add_test(NAME d3d12x_device_negative_${adapter} COMMAND d3d12x_device_check
        "$<TARGET_FILE:d3d12_xs_shim>" negative ${adapter})
    add_test(NAME d3d12x_device_struct_return_${adapter} COMMAND d3d12x_device_check
        "$<TARGET_FILE:d3d12_xs_shim>" struct-return ${adapter})
endforeach()
set_tests_properties(d3d12x_device_create_hardware d3d12x_device_negative_hardware
    d3d12x_device_struct_return_hardware PROPERTIES
    SKIP_RETURN_CODE 77)
set_tests_properties(d3d12x_device_negative_warp d3d12x_device_negative_hardware PROPERTIES
    PASS_REGULAR_EXPRESSION "D3D12XboxCreateDevice: unsupported IID \\{7c0e1a10-5f2d-4b7a-9a51-3d9c6b2e8f01\\}")
# Static-sampler root signature and compute PSO through D3D12XboxCreateDevice (#XCL-19).
foreach(adapter IN ITEMS warp hardware)
    add_test(NAME d3d12x_pso_static_sampler_${adapter} COMMAND d3d12x_device_check
        "$<TARGET_FILE:d3d12_xs_shim>" pso ${adapter})
endforeach()
add_test(NAME d3d12x_pso_reject_warp COMMAND d3d12x_device_check
    "$<TARGET_FILE:d3d12_xs_shim>" reject warp)
set_tests_properties(d3d12x_pso_static_sampler_hardware PROPERTIES SKIP_RETURN_CODE 77)

# D3D12.X descriptor heaps, CPU/GPU handles and descriptor table records (#XCL-09).
set(d3d12x_descriptor_cs "${CMAKE_CURRENT_BINARY_DIR}/d3d12x_descriptor_cs.h")
add_custom_command(OUTPUT "${d3d12x_descriptor_cs}"
    COMMAND "${DXC_EXECUTABLE}" -T cs_6_0 -E CSMain -Vn d3d12x_descriptor_cs
            -Fh "${d3d12x_descriptor_cs}" -Fo "${CMAKE_CURRENT_BINARY_DIR}/d3d12x_descriptor_cs.cso"
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/d3d12x_descriptor.hlsl"
    DEPENDS tests/d3d12x_descriptor.hlsl
    BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/d3d12x_descriptor_cs.cso" VERBATIM)
add_custom_target(d3d12x_descriptor_shader DEPENDS "${d3d12x_descriptor_cs}")
add_executable(d3d12x_descriptor_check tests/d3d12x_descriptor_check.cpp "${d3d12x_descriptor_cs}")
add_dependencies(d3d12x_descriptor_check d3d12_xs_shim d3d12x_descriptor_shader)
target_include_directories(d3d12x_descriptor_check PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
target_compile_features(d3d12x_descriptor_check PRIVATE cxx_std_17)
target_link_libraries(d3d12x_descriptor_check PRIVATE d3d12 dxgi)
foreach(adapter IN ITEMS warp hardware)
    add_test(NAME d3d12x_descriptor_heap_${adapter} COMMAND d3d12x_descriptor_check
        "$<TARGET_FILE:d3d12_xs_shim>" heap ${adapter})
    add_test(NAME d3d12x_descriptor_table_${adapter} COMMAND d3d12x_descriptor_check
        "$<TARGET_FILE:d3d12_xs_shim>" table ${adapter})
    add_test(NAME d3d12x_descriptor_negative_${adapter} COMMAND d3d12x_descriptor_check
        "$<TARGET_FILE:d3d12_xs_shim>" negative ${adapter})
    set_tests_properties(d3d12x_descriptor_heap_${adapter} d3d12x_descriptor_table_${adapter}
        d3d12x_descriptor_negative_${adapter} PROPERTIES TIMEOUT 30)
endforeach()
set_tests_properties(d3d12x_descriptor_heap_hardware d3d12x_descriptor_table_hardware
    d3d12x_descriptor_negative_hardware PROPERTIES SKIP_RETURN_CODE 77)
# D3D12.X compute command list recording, transitions, clear and readback (#XCL-10).
set(d3d12x_compute_list_cs "${CMAKE_CURRENT_BINARY_DIR}/d3d12x_compute_list_cs.h")
add_custom_command(OUTPUT "${d3d12x_compute_list_cs}"
    COMMAND "${DXC_EXECUTABLE}" -T cs_6_0 -E CSMain -Vn d3d12x_compute_list_cs
            -Fh "${d3d12x_compute_list_cs}" -Fo "${CMAKE_CURRENT_BINARY_DIR}/d3d12x_compute_list_cs.cso"
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/d3d12x_compute_list.hlsl"
    DEPENDS tests/d3d12x_compute_list.hlsl
    BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/d3d12x_compute_list_cs.cso" VERBATIM)
add_custom_target(d3d12x_compute_list_shader DEPENDS "${d3d12x_compute_list_cs}")
add_executable(d3d12x_compute_list_check tests/d3d12x_compute_list_check.cpp
    tests/desktop_gpu.cpp "${d3d12x_compute_list_cs}")
add_dependencies(d3d12x_compute_list_check d3d12_xs_shim d3d12x_compute_list_shader)
target_include_directories(d3d12x_compute_list_check PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
target_compile_features(d3d12x_compute_list_check PRIVATE cxx_std_17)
target_link_libraries(d3d12x_compute_list_check PRIVATE d3d12 dxgi)
foreach(adapter IN ITEMS warp hardware)
    add_test(NAME d3d12x_compute_list_${adapter} COMMAND d3d12x_compute_list_check
        "$<TARGET_FILE:d3d12_xs_shim>" ${adapter})
    set_tests_properties(d3d12x_compute_list_${adapter} PROPERTIES TIMEOUT 30)
endforeach()
set_tests_properties(d3d12x_compute_list_hardware PROPERTIES SKIP_RETURN_CODE 77)

# D3D12.X graphics command list: one source-built triangle through a VS/PS PSO (#XCL-12).
foreach(stage IN ITEMS vs ps)
    string(TOUPPER "${stage}" entry)
    set(d3d12x_graphics_header "${CMAKE_CURRENT_BINARY_DIR}/d3d12x_graphics_${stage}.h")
    add_custom_command(OUTPUT "${d3d12x_graphics_header}"
        COMMAND "${DXC_EXECUTABLE}" -T ${stage}_6_0 -E ${entry}Main -Vn d3d12x_graphics_${stage}
                -Fh "${d3d12x_graphics_header}" -Fo "${CMAKE_CURRENT_BINARY_DIR}/d3d12x_graphics_${stage}.cso"
                "${CMAKE_CURRENT_SOURCE_DIR}/tests/d3d12x_graphics.hlsl"
        DEPENDS tests/d3d12x_graphics.hlsl
        BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/d3d12x_graphics_${stage}.cso" VERBATIM)
    set(d3d12x_graphics_${stage} "${d3d12x_graphics_header}")
endforeach()
add_custom_target(d3d12x_graphics_shader DEPENDS
    "${d3d12x_graphics_vs}"
    "${d3d12x_graphics_ps}")
add_executable(d3d12x_graphics_list_check tests/d3d12x_graphics_list_check.cpp tests/desktop_gpu.cpp
    "${d3d12x_graphics_vs}" "${d3d12x_graphics_ps}")
add_dependencies(d3d12x_graphics_list_check d3d12_xs_shim d3d12x_graphics_shader)
target_include_directories(d3d12x_graphics_list_check PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
target_compile_features(d3d12x_graphics_list_check PRIVATE cxx_std_17)
target_link_libraries(d3d12x_graphics_list_check PRIVATE d3d12 dxgi)
foreach(adapter IN ITEMS warp hardware)
    add_test(NAME d3d12x_graphics_list_${adapter} COMMAND d3d12x_graphics_list_check
        "$<TARGET_FILE:d3d12_xs_shim>" ${adapter})
    set_tests_properties(d3d12x_graphics_list_${adapter} PROPERTIES TIMEOUT 60)
endforeach()
set_tests_properties(d3d12x_graphics_list_hardware PROPERTIES SKIP_RETURN_CODE 77)
# The main executable's CPU/GPU range translation against a desktop D3D12 heap (#874).
set(vm_translation_header "${CMAKE_CURRENT_BINARY_DIR}/vm_translation_cs.h")
add_custom_command(OUTPUT "${vm_translation_header}"
    COMMAND "${DXC_EXECUTABLE}" -T cs_6_0 -E main -Vn vm_translation_cs -Fh "${vm_translation_header}"
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/vm_translation.hlsl"
    DEPENDS tests/vm_translation.hlsl VERBATIM)
add_executable(vm_translation_check tests/vm_translation_check.cpp src/d3d12_xs_range.cpp "${vm_translation_header}")
target_include_directories(vm_translation_check PRIVATE "${CMAKE_CURRENT_BINARY_DIR}" tests src)
target_compile_features(vm_translation_check PRIVATE cxx_std_17)
target_link_libraries(vm_translation_check PRIVATE d3d12_xs_shim d3d12 dxgi)
add_test(NAME vm_translation_shared_range COMMAND vm_translation_check shared_range)
foreach(adapter IN ITEMS warp hardware)
    foreach(scenario IN ITEMS device sequence range range_lifetime range_partial)
        add_test(NAME vm_translation_${scenario}_${adapter} COMMAND vm_translation_check ${scenario}_${adapter})
        set_tests_properties(vm_translation_${scenario}_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    endforeach()
endforeach()
# The configured range's windows and GPU addresses (src/d3d12_xs_range.h) with a fake device and fake
# windows. The mutated run is the negative control.
add_executable(d3d12_xs_range_check tests/d3d12_xs_range_check.cpp src/d3d12_xs_range.cpp)
target_include_directories(d3d12_xs_range_check PRIVATE src)
target_compile_features(d3d12_xs_range_check PRIVATE cxx_std_17)
add_test(NAME d3d12_xs_range COMMAND d3d12_xs_range_check)
add_test(NAME d3d12_xs_range_full_last_window_control COMMAND d3d12_xs_range_check --mutate full_last_window)
set_tests_properties(d3d12_xs_range_full_last_window_control PROPERTIES WILL_FAIL TRUE)
# XG-style swizzle selection compared with AMD addrlib (#870). addrlib is an optional local dependency
# fetched by tools/fetch_addrlib.py into out/deps/mesa (Mesa 25.1.0, src/amd/addrlib, MIT); without it the
# test skips (77). The source list and definitions follow Mesa's src/amd/addrlib/meson.build.
set(_addrlib_default "${CMAKE_CURRENT_SOURCE_DIR}/out/deps/mesa/src/amd/addrlib")
if(NOT EXISTS "${_addrlib_default}/inc/addrinterface.h")
    set(_addrlib_default "")
endif()
set(XBOX2WINDOWS_ADDRLIB_DIR "${_addrlib_default}" CACHE PATH
    "AMD addrlib source (Mesa src/amd/addrlib, see tools/fetch_addrlib.py); empty skips xg_swizzle_select")
add_executable(xg_swizzle_select_check tests/xg_swizzle_select_check.cpp)
target_compile_features(xg_swizzle_select_check PRIVATE cxx_std_17)
if(XBOX2WINDOWS_ADDRLIB_DIR)
    if(NOT EXISTS "${XBOX2WINDOWS_ADDRLIB_DIR}/inc/addrinterface.h")
        message(FATAL_ERROR "XBOX2WINDOWS_ADDRLIB_DIR=${XBOX2WINDOWS_ADDRLIB_DIR} has no inc/addrinterface.h")
    endif()
    set(_a "${XBOX2WINDOWS_ADDRLIB_DIR}")
    add_library(xbw_addrlib STATIC EXCLUDE_FROM_ALL
        "${_a}/src/addrinterface.cpp" "${_a}/src/core/addrelemlib.cpp" "${_a}/src/core/addrlib.cpp"
        "${_a}/src/core/addrlib1.cpp" "${_a}/src/core/addrlib2.cpp" "${_a}/src/core/addrlib3.cpp"
        "${_a}/src/core/addrobject.cpp" "${_a}/src/core/addrswizzler.cpp" "${_a}/src/core/coord.cpp"
        "${_a}/src/gfx9/gfx9addrlib.cpp" "${_a}/src/gfx10/gfx10addrlib.cpp" "${_a}/src/gfx11/gfx11addrlib.cpp"
        "${_a}/src/gfx12/gfx12addrlib.cpp" "${_a}/src/r800/ciaddrlib.cpp" "${_a}/src/r800/egbaddrlib.cpp"
        "${_a}/src/r800/siaddrlib.cpp")
    target_include_directories(xbw_addrlib PUBLIC "${_a}/inc" PRIVATE "${_a}/src" "${_a}/src/core"
        "${_a}/src/chip/gfx9" "${_a}/src/chip/r800" "${_a}/src/chip/gfx10" "${_a}/src/chip/gfx11" "${_a}/src/chip/gfx12")
    target_compile_definitions(xbw_addrlib PUBLIC ADDR_FASTCALL= PRIVATE LITTLEENDIAN_CPU DEBUG=0)
    target_compile_options(xbw_addrlib PRIVATE /w)
    target_link_libraries(xg_swizzle_select_check PRIVATE xbw_addrlib)
    target_compile_definitions(xg_swizzle_select_check PRIVATE XBOX2WINDOWS_HAVE_ADDRLIB)
    message(STATUS "xg_swizzle_select: addrlib from ${XBOX2WINDOWS_ADDRLIB_DIR}")
else()
    message(STATUS "xg_swizzle_select: addrlib not configured, the test will skip (77)")
endif()
add_test(NAME xg_bc64_tiling_reference COMMAND xg_swizzle_select_check --bc64-tiling)
set_tests_properties(xg_bc64_tiling_reference PROPERTIES TIMEOUT 30 SKIP_RETURN_CODE 77)
# Which descriptions take the 64 KiB standard BC chain (src/startup_stubs/bc64_chain.h), without addrlib
# or a device. The mutated run expects layout UNKNOWN to be accepted and is the negative control.
add_executable(startup_bc64_supports_check tests/startup_bc64_supports_check.cpp)
target_compile_features(startup_bc64_supports_check PRIVATE cxx_std_17)
target_include_directories(startup_bc64_supports_check PRIVATE src)
add_test(NAME startup_bc64_supports COMMAND startup_bc64_supports_check)
add_test(NAME startup_bc64_supports_layout_control COMMAND startup_bc64_supports_check --mutate layout)
set_tests_properties(startup_bc64_supports_layout_control PROPERTIES WILL_FAIL TRUE)
add_test(NAME xg_swizzle_select COMMAND xg_swizzle_select_check)
set_tests_properties(xg_swizzle_select PROPERTIES TIMEOUT 30 SKIP_RETURN_CODE 77)
# Mutation control: the same comparison must identify all nine mismatches.
add_test(NAME xg_swizzle_select_mutation COMMAND xg_swizzle_select_check --mutate-linear-fallback)
set_tests_properties(xg_swizzle_select_mutation PROPERTIES TIMEOUT 30 SKIP_RETURN_CODE 77)
add_test(NAME xg_swizzle_factory COMMAND xg_swizzle_select_check --factory)
add_test(NAME xg_swizzle_factory_no_write COMMAND xg_swizzle_select_check --mutate-factory-no-write)
set_tests_properties(xg_swizzle_factory xg_swizzle_factory_no_write PROPERTIES TIMEOUT 30 SKIP_RETURN_CODE 77)
foreach(adapter IN ITEMS hardware warp)
    add_test(NAME offscreen_${adapter} COMMAND offscreen_check ${adapter} triangle_${adapter}.bmp)
    add_test(NAME ffx_provider_loop_${adapter} COMMAND offscreen_check ${adapter} --ffx-provider-loop)
    add_test(NAME ffx_device_init_${adapter} COMMAND offscreen_check ${adapter} --ffx-device-init)
    add_test(NAME ffx_recurrent_${adapter} COMMAND offscreen_check ${adapter} --ffx-recurrent)
    add_test(NAME dual_uav_${adapter} COMMAND offscreen_check ${adapter} --dual-uav)
    add_test(NAME stream_root_srv_${adapter} COMMAND offscreen_check ${adapter} --stream-root-srv)
    set_tests_properties(stream_root_srv_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    add_test(NAME descriptor_window_submissions_${adapter} COMMAND offscreen_check ${adapter} --descriptor-window-submissions)
    foreach(mode IN ITEMS refresh stale)
        add_test(NAME cpu_srv_${mode}_${adapter} COMMAND offscreen_check ${adapter} --cpu-srv-${mode})
        set_tests_properties(cpu_srv_${mode}_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    endforeach()
    add_test(NAME descriptor_window_stale_${adapter} COMMAND offscreen_check ${adapter} --descriptor-window-stale)
    set_tests_properties(descriptor_window_submissions_${adapter} descriptor_window_stale_${adapter}
        PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    set_tests_properties(descriptor_window_stale_${adapter}
        PROPERTIES FAIL_REGULAR_EXPRESSION "descriptor_windows submit=2 slot=[1-9]")
    foreach(scenario IN ITEMS live stale unknown)
        add_test(NAME descriptor_record_copy_${scenario}_${adapter}
            COMMAND offscreen_check ${adapter} --descriptor-record-copy ${scenario})
        set_tests_properties(descriptor_record_copy_${scenario}_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    endforeach()
    add_test(NAME texture_mips_${adapter} COMMAND offscreen_check ${adapter} --texture-mips)
    add_test(NAME dynamic_sampler_index_${adapter} COMMAND offscreen_check ${adapter} --dynamic-sampler-index)
    set_tests_properties(dynamic_sampler_index_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    set_tests_properties(dual_uav_${adapter} texture_mips_${adapter} PROPERTIES TIMEOUT 45)
    add_test(NAME dxr_query_${adapter} COMMAND offscreen_check ${adapter} --ray-query)
    add_test(NAME dxr_address_ranges_${adapter} COMMAND offscreen_check ${adapter} --dxr-address-ranges)
    add_test(NAME dxr_placed_build_${adapter} COMMAND offscreen_check ${adapter} --dxr-placed-build)
    set_tests_properties(dxr_placed_build_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    set_tests_properties(dxr_address_ranges_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    add_test(NAME dxr_pipeline_${adapter} COMMAND offscreen_check ${adapter} --dxr-pipeline)
    add_test(NAME dxr_update_${adapter} COMMAND offscreen_check ${adapter} --dxr-update)
    add_test(NAME dxr_update_pipeline_${adapter} COMMAND offscreen_check ${adapter} --dxr-update-pipeline)
    set_tests_properties(dxr_update_${adapter} dxr_update_pipeline_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    add_test(NAME dxr_facing_${adapter} COMMAND offscreen_check ${adapter} --dxr-facing)
    add_test(NAME dxr_facing_pipeline_${adapter} COMMAND offscreen_check ${adapter} --dxr-facing-pipeline)
    set_tests_properties(dxr_facing_${adapter} dxr_facing_pipeline_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    add_test(NAME dxr_opacity_${adapter} COMMAND offscreen_check ${adapter} --dxr-opacity)
    add_test(NAME dxr_opacity_pipeline_${adapter} COMMAND offscreen_check ${adapter} --dxr-opacity-pipeline)
    set_tests_properties(dxr_opacity_${adapter} dxr_opacity_pipeline_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    add_test(NAME bvh_buffer_ops_${adapter} COMMAND offscreen_check ${adapter} --buffer-ops)
    add_test(NAME gpu_alias_${adapter} COMMAND offscreen_check ${adapter} --gpu-alias)
    add_test(NAME gpu_alias_window_${adapter} COMMAND offscreen_check ${adapter} --gpu-alias-window)
    add_test(NAME gpu_alias_lookup_${adapter} COMMAND offscreen_check ${adapter} --gpu-alias-lookup)
    add_test(NAME gpu_alias_lookup_writes_${adapter} COMMAND offscreen_check ${adapter} --gpu-alias-lookup-writes)
    add_test(NAME gpu_alias_registry_${adapter} COMMAND offscreen_check ${adapter} --gpu-alias-registry)
    set_tests_properties(gpu_alias_${adapter} gpu_alias_window_${adapter} gpu_alias_lookup_${adapter}
        gpu_alias_lookup_writes_${adapter} gpu_alias_registry_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    add_test(NAME bvh_root_roundtrip_${adapter} COMMAND offscreen_check ${adapter} --bvh-root-roundtrip)
    set_tests_properties(bvh_root_roundtrip_${adapter} PROPERTIES TIMEOUT 45)
    foreach(work IN ITEMS reads waves wave64 hits differential atomics numeric packed)
        add_test(NAME bvh_${work}_${adapter} COMMAND offscreen_check ${adapter} --bvh-${work})
        set_tests_properties(bvh_${work}_${adapter} PROPERTIES TIMEOUT 45)
    endforeach()
    set_tests_properties(bvh_wave64_${adapter} bvh_differential_${adapter} PROPERTIES SKIP_RETURN_CODE 77)
    add_test(NAME wave_shuffle_${adapter} COMMAND offscreen_check ${adapter} --wave-shuffle)
    set_tests_properties(wave_shuffle_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    add_test(NAME wave64_emulate_${adapter} COMMAND offscreen_check ${adapter} --wave64-emulate)
    set_tests_properties(wave64_emulate_${adapter} PROPERTIES TIMEOUT 45)
    add_test(NAME shader_wave_${adapter} COMMAND offscreen_check ${adapter} --shader-wave)
    set_tests_properties(shader_wave_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    foreach(width IN ITEMS 32 64)
        add_test(NAME shader_wave${width}_${adapter} COMMAND offscreen_check ${adapter} --shader-wave${width})
        set_tests_properties(shader_wave${width}_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    endforeach()
    set_tests_properties(bvh_buffer_ops_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    set_tests_properties(dxr_query_${adapter} dxr_pipeline_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    set_tests_properties(offscreen_${adapter} PROPERTIES TIMEOUT 45)
endforeach()
set_tests_properties(offscreen_hardware ffx_device_init_hardware ffx_recurrent_hardware ffx_provider_loop_hardware PROPERTIES SKIP_RETURN_CODE 77)
set_tests_properties(dual_uav_hardware texture_mips_hardware PROPERTIES SKIP_RETURN_CODE 77)
set_tests_properties(bvh_root_roundtrip_hardware PROPERTIES SKIP_RETURN_CODE 77)
set_tests_properties(bvh_atomics_hardware PROPERTIES SKIP_RETURN_CODE 77)
set_tests_properties(bvh_numeric_hardware bvh_packed_hardware PROPERTIES SKIP_RETURN_CODE 77)
set_tests_properties(bvh_reads_hardware bvh_waves_hardware bvh_hits_hardware PROPERTIES SKIP_RETURN_CODE 77)
# The cs_6_5 fixtures skip on a WARP below SM 6.5. The Windows 10 in-box WARP reports SM 6.2.
set_tests_properties(bvh_reads_warp bvh_waves_warp bvh_hits_warp bvh_packed_warp
    bvh_numeric_warp bvh_atomics_warp PROPERTIES SKIP_RETURN_CODE 77)
# CRT-free startup attempt for a forced-wave compute pipeline (Refs #870; the desktop readback is #1132).
# One EXE per shader variant and adapter, so the entry point takes no arguments. Each prints the
# D3D12_OPTIONS1 lane range, the exact CreateComputePipelineState HRESULT and the debug-layer messages,
# and reads back WaveGetLaneCount() when creation succeeds. An adapter whose range excludes the forced
# width must refuse the pipeline and exits 77 with the HRESULT; creation that contradicts the range fails.
foreach(width IN ITEMS 0 4 32)
    set(variant_dir "${CMAKE_CURRENT_BINARY_DIR}/wave_startup_${width}")
    set(defines)
    if(width)
        set(defines -D WAVE_STARTUP_WIDTH=${width})
    endif()
    add_custom_command(OUTPUT "${variant_dir}/wave_startup_cs.h"
        COMMAND "${CMAKE_COMMAND}" -E make_directory "${variant_dir}"
        COMMAND "${DXC_EXECUTABLE}" -T cs_6_6 -E main ${defines} -Vn wave_startup_cs
                -Fh "${variant_dir}/wave_startup_cs.h" "${CMAKE_CURRENT_SOURCE_DIR}/tests/wave_startup.hlsl"
        DEPENDS tests/wave_startup.hlsl VERBATIM)
    foreach(adapter IN ITEMS warp hardware)
        set(target wave_startup_${width}_${adapter})
        add_executable(${target} tests/wave_startup_exe.cpp "${variant_dir}/wave_startup_cs.h")
        target_include_directories(${target} PRIVATE "${variant_dir}")
        string(COMPARE EQUAL "${adapter}" warp is_warp)
        target_compile_definitions(${target} PRIVATE WAVE_STARTUP_WIDTH=${width} WAVE_STARTUP_WARP=$<BOOL:${is_warp}>)
        target_compile_options(${target} PRIVATE /GS- /Zl)
        target_link_options(${target} PRIVATE /ENTRY:fixture_entry /NODEFAULTLIB /SUBSYSTEM:CONSOLE
            /DYNAMICBASE /INCREMENTAL:NO)
        target_link_libraries(${target} PRIVATE kernel32 d3d12 dxgi)
        add_test(NAME ${target} COMMAND ${target})
        set_tests_properties(${target} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    endforeach()
endforeach()

add_test(NAME bvh_differential_pipeline_warp COMMAND offscreen_check warp --bvh-differential-pipeline)
set_tests_properties(bvh_differential_pipeline_warp PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)

add_test(NAME offscreen_invalid_operation_warp COMMAND "${CMAKE_COMMAND}"
    "-DRENDERER=$<TARGET_FILE:offscreen_check>" -P "${CMAKE_SOURCE_DIR}/tests/offscreen_invalid_operation.cmake")
set_tests_properties(offscreen_invalid_operation_warp PROPERTIES TIMEOUT 45)

add_test(NAME offscreen_capability_table_self_test COMMAND offscreen_check --self-test)
set_tests_properties(offscreen_capability_table_self_test PROPERTIES TIMEOUT 45)
foreach(adapter IN ITEMS hardware warp)
    add_test(NAME descriptor_heap_paging_${adapter} COMMAND offscreen_check ${adapter} --descriptor-heap-paging)
    add_test(NAME descriptor_heap_paging_swapped_${adapter}
        COMMAND offscreen_check ${adapter} --descriptor-heap-paging-swapped)
    set_tests_properties(descriptor_heap_paging_${adapter} descriptor_heap_paging_swapped_${adapter}
        PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    set_tests_properties(descriptor_heap_paging_swapped_${adapter}
        PROPERTIES FAIL_REGULAR_EXPRESSION "descriptor_paging swapped=0")
endforeach()
foreach(adapter IN ITEMS hardware warp)
    add_test(NAME startup_sampler_x_${adapter} COMMAND offscreen_check ${adapter} --startup-sampler-x)
    add_test(NAME startup_sampler_x_swapped_${adapter} COMMAND offscreen_check ${adapter} --startup-sampler-x-swapped)
    set_tests_properties(startup_sampler_x_${adapter} startup_sampler_x_swapped_${adapter}
        PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
    set_tests_properties(startup_sampler_x_swapped_${adapter}
        PROPERTIES FAIL_REGULAR_EXPRESSION "startup_sampler_x swapped=0")
    add_test(NAME startup_cb_blend_${adapter} COMMAND offscreen_check ${adapter} --startup-cb-blend)
    add_test(NAME startup_cb_blend_swapped_${adapter} COMMAND offscreen_check ${adapter} --startup-cb-blend-swapped)
    set_tests_properties(startup_cb_blend_${adapter} startup_cb_blend_swapped_${adapter} PROPERTIES TIMEOUT 60)
    add_test(NAME startup_composite_provenance_${adapter} COMMAND offscreen_check ${adapter} --startup-composite-provenance)
    set_tests_properties(startup_composite_provenance_${adapter} PROPERTIES TIMEOUT 60)
endforeach()

set(SIMPLE_TRIANGLE_DIR "${CMAKE_SOURCE_DIR}/out/reference/b1e77b6ac10f95b07296025fda171ca2ddc42dbe/Samples/IntroGraphics/SimpleTriangle"
    CACHE PATH "Microsoft SimpleTriangle sample directory")
if(EXISTS "${SIMPLE_TRIANGLE_DIR}/VertexShader.hlsl")
    find_program(REFERENCE_POWERSHELL NAMES pwsh powershell REQUIRED)
    foreach(adapter IN ITEMS hardware warp)
        add_test(NAME reference_triangle_${adapter} COMMAND "${REFERENCE_POWERSHELL}" -NoProfile -ExecutionPolicy Bypass
            -File "${CMAKE_SOURCE_DIR}/tests/reference_triangle.ps1" -Sample "${SIMPLE_TRIANGLE_DIR}"
            -Compiler "${DXC_EXECUTABLE}" -Renderer "$<TARGET_FILE:offscreen_check>" -Adapter ${adapter})
        set_tests_properties(reference_triangle_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
    endforeach()
endif()

target_sources(offscreen_check PRIVATE tests/ffx_device_init.cpp tests/ffx_backend_slots.cpp tests/ffx_recurrent.cpp tests/ffx_provider_loop.cpp)
set(FFX_UPSCALER_IMAGE "" CACHE FILEPATH "Local Xbox upscaler DLL for offline backend translation")
set(FFX_DISPATCH_SHADER_DIR "" CACHE PATH "Directory containing locally translated FSR4 shaders and scratch_reference.h")
if(FFX_DISPATCH_SHADER_DIR)
    if(NOT FFX_UPSCALER_IMAGE)
        message(FATAL_ERROR "Package dispatch replay requires FFX_UPSCALER_IMAGE")
    endif()
    target_compile_definitions(offscreen_check PRIVATE FFX_DISPATCH_AOT)
    target_include_directories(offscreen_check PRIVATE "${FFX_DISPATCH_SHADER_DIR}")
endif()
target_sources(offscreen_check PRIVATE tests/ffx_dispatch.cpp tests/ffx_dispatch_scratch.cpp)
foreach(stage IN ITEMS pass24 pass25 producer rcas)
    set(shader "${CMAKE_CURRENT_SOURCE_DIR}/tests/ffx_dispatch_control.hlsl")
    set(entry "${stage}")
    if(FFX_DISPATCH_SHADER_DIR)
        set(shader "${FFX_DISPATCH_SHADER_DIR}/${stage}.hlsl")
        set(entry main)
    endif()
    set(storage_formats Native Packed)
    if(stage MATCHES "^pass")
        set(storage_formats Raw)
    endif()
    foreach(storage IN LISTS storage_formats)
        set(symbol "${stage}${storage}")
        set(header "${CMAKE_CURRENT_BINARY_DIR}/${symbol}.h")
        add_custom_command(OUTPUT "${header}"
            COMMAND "${DXC_EXECUTABLE}" -T cs_6_0 -E "${entry}" -D "STORAGE_${storage}=1"
                -I "${CMAKE_CURRENT_SOURCE_DIR}/tests" -Vn "${symbol}" -Fh "${header}"
                -Fo "${CMAKE_CURRENT_BINARY_DIR}/${symbol}.cso" "${shader}"
            DEPENDS "${shader}" tests/ffx_dispatch_io.hlsli src/rgb9e5.hlsli src/ffx_float_to_int.hlsli
            BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/${symbol}.cso" VERBATIM)
        target_sources(offscreen_check PRIVATE "${header}")
    endforeach()
endforeach()
if(FFX_UPSCALER_IMAGE)
    if(NOT EXISTS "${INTAKE_PYTHON}" OR NOT EXISTS "${FFX_UPSCALER_IMAGE}")
        message(FATAL_ERROR "Package translation requires INTAKE_PYTHON and FFX_UPSCALER_IMAGE")
    endif()
    set(device_slice "${CMAKE_CURRENT_BINARY_DIR}/ffx_device_slice.cpp")
    add_custom_command(OUTPUT "${device_slice}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/ffx_device_init.py" "${FFX_UPSCALER_IMAGE}" "${device_slice}"
        DEPENDS "${FFX_UPSCALER_IMAGE}" tools/ffx_device_init.py tools/ffx_recurrent.py tools/ffx_provider_loop.py tools/ffx_dispatch_profile.py tools/ffx_version_query.py tools/cpu_ranges.py tools/cpu_leaf_aot.py
        VERBATIM)
    target_sources(offscreen_check PRIVATE "${device_slice}")
    target_include_directories(offscreen_check PRIVATE "${CMAKE_SOURCE_DIR}/tests")
    target_compile_definitions(offscreen_check PRIVATE FFX_PACKAGE_DEVICE_SLICE)
    get_filename_component(ffx_dir "${FFX_UPSCALER_IMAGE}" DIRECTORY)
    set(ffx_loader "${ffx_dir}/amd_fidelityfx_loader_xbox.dll")
    if(EXISTS "${ffx_loader}")
        set(loop_loader "${CMAKE_CURRENT_BINARY_DIR}/ffx_loop_loader.cpp")
        add_custom_command(OUTPUT "${loop_loader}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/ffx_create_context.py" "${ffx_loader}" "${loop_loader}"
            DEPENDS "${ffx_loader}" tools/ffx_create_context.py tools/ffx_version_query.py tools/cpu_leaf_aot.py tools/cpu_ranges.py
            VERBATIM)
        set_source_files_properties("${loop_loader}" PROPERTIES HEADER_FILE_ONLY TRUE)
        target_sources(offscreen_check PRIVATE "${loop_loader}" tests/ffx_provider_loader.cpp)
        target_compile_definitions(offscreen_check PRIVATE FFX_LOOP_LOADER)
    endif()
endif()
if(EXISTS "${INTAKE_PYTHON}")
    # Startup stand-ins: exports, named traps and permissive answers (docs/startup-runtime.md).
    foreach(scheduler IN ITEMS controlled system)
        add_test(NAME startup_gamesave_submit_${scheduler} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_gamesave_submit_check.py" "$<TARGET_FILE:startup_gamesave_submit>" ${scheduler})
        set_tests_properties(startup_gamesave_submit_${scheduler} PROPERTIES TIMEOUT 30)
    endforeach()
    set_tests_properties(startup_gamesave_submit_system PROPERTIES SKIP_RETURN_CODE 77)
    foreach(scheduler IN ITEMS controlled system)
        add_test(NAME startup_gamesave_read_async_${scheduler} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_gamesave_read_async_check.py" "$<TARGET_FILE:startup_gamesave_read_async>" ${scheduler})
        set_tests_properties(startup_gamesave_read_async_${scheduler} PROPERTIES TIMEOUT 30)
    endforeach()
    set_tests_properties(startup_gamesave_read_async_system PROPERTIES SKIP_RETURN_CODE 77)
    add_test(NAME startup_stubs COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_stub_check.py"
        "$<TARGET_FILE:startup_stub_consumer>" "${CMAKE_CURRENT_SOURCE_DIR}/src/startup_stubs"
        "$<TARGET_FILE:startup_stub_acphal>" "$<TARGET_FILE:startup_stub_chad>" "$<TARGET_FILE:startup_stub_xapu>"
        "$<TARGET_FILE:startup_stub_xcrdapi>" "$<TARGET_FILE:startup_stub_xg_xs>" "$<TARGET_FILE:startup_stub_gameinput>"
        "$<TARGET_FILE:xmem_shim>" "$<TARGET_FILE:d3d12_xs_shim>")
    set_tests_properties(startup_stubs PROPERTIES TIMEOUT 120)
    add_test(NAME startup_storage_directory_batch COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_storage_directory_batch.py" "$<TARGET_FILE:startup_stub_consumer>")
    set_tests_properties(startup_storage_directory_batch PROPERTIES TIMEOUT 30)
    # The stand-in's zlib decoder (src/startup_stubs/inflate.h) against CPython's zlib (#1149): stored,
    # fixed and dynamic blocks, a sync flush, and rejected short, truncated and out-of-window streams.
    add_test(NAME startup_inflate COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_inflate_check.py" "$<TARGET_FILE:startup_inflate_probe>")
    set_tests_properties(startup_inflate PROPERTIES TIMEOUT 60)
    add_test(NAME startup_live_view COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_live_view_check.py")
    set_tests_properties(startup_live_view PROPERTIES TIMEOUT 60)
    add_test(NAME startup_uniform_lane COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_uniform_lane_check.py")
    add_test(NAME startup_image_sample_l COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_image_sample_l_check.py")
    add_test(NAME startup_image_sample_c COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_image_sample_c_check.py")
    # Rewrites for instructions the host processor lacks (tools/startup/host_cpu.py), run on the host.
    add_test(NAME startup_host_cpu COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_host_cpu_check.py")
    set_tests_properties(startup_host_cpu PROPERTIES SKIP_RETURN_CODE 77)
    # The same switch in the startup debugger, on a probe that executes SHA-extension and RDPID instructions.
    add_test(NAME startup_host_cpu_debug COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_host_cpu_debug_check.py" "$<TARGET_FILE:startup_host_cpu_probe>")
    set_tests_properties(startup_host_cpu_debug PROPERTIES TIMEOUT 300)
    # tools/startup/capture.py against synthetic capture.bin files laid out like the writer's (#1149).
    add_test(NAME startup_capture COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_capture_check.py")
    # gfx2hlsl.py --capture terminates and refuses malformed pipeline records (wrapper cycles, short descs).
    add_test(NAME startup_gfx2hlsl_capture COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_gfx2hlsl_capture_check.py")
    set_tests_properties(startup_gfx2hlsl_capture PROPERTIES TIMEOUT 120)
    # VOP1/VOP2 sub-dword addressing (SDWA) in the translator (#1141), checked against the pixel AOT
    # lowering of the same selections.
    add_test(NAME startup_gfx2hlsl_sdwa COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_gfx2hlsl_sdwa_check.py")
    # gfx2hlsl run-time sampler heap indexing (#1143): translation controls, then a desktop draw with
    # DXC ps_6_0 and, when the Windows SDK's FXC is found, the ps_5_1 the startup tools compile with.
    find_program(STARTUP_FXC fxc HINTS "${DXC_BIN_DIR}")
    add_test(NAME startup_sampler_heap_controls COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/startup_sampler_heap_check.py")
    foreach(adapter IN ITEMS warp hardware)
        add_test(NAME startup_r8_uint_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/startup_r8_uint_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(startup_r8_uint_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME startup_sampler_heap_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/startup_sampler_heap_check.py" draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter} $<$<BOOL:${STARTUP_FXC}>:${STARTUP_FXC}>)
        set_tests_properties(startup_sampler_heap_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77)
        add_test(NAME startup_sampler_heap_wrapped_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/startup_sampler_heap_check.py" wrapped "$<TARGET_FILE:d3d12_xs_shim>" ${adapter})
        set_tests_properties(startup_sampler_heap_wrapped_${adapter} PROPERTIES TIMEOUT 30
            ENVIRONMENT "X2W_D3D12_PERMISSIVE=1;X2W_D3D12_DEBUG=1")
    endforeach()
    # gfx2hlsl 3D image sampling (#1149): translation controls, then a desktop draw that names the texel.
    add_test(NAME startup_image_3d_controls COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/startup_image_3d_check.py")
    add_test(NAME startup_image_load_mip_controls COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/startup_image_load_mip_check.py")
    foreach(adapter IN ITEMS warp hardware)
        add_test(NAME startup_image_3d_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/startup_image_3d_check.py" draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter} $<$<BOOL:${STARTUP_FXC}>:${STARTUP_FXC}>)
        set_tests_properties(startup_image_3d_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77)
        add_test(NAME startup_image_load_mip_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/startup_image_load_mip_check.py" --draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(startup_image_load_mip_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77)
    endforeach()
    # gfx2hlsl 2D array images: translation controls, then a desktop draw that names the slice's texel.
    add_test(NAME startup_image_2d_array_controls COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/startup_image_2d_array_check.py")
    foreach(adapter IN ITEMS warp hardware)
        add_test(NAME startup_image_2d_array_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/startup_image_2d_array_check.py" draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter} $<$<BOOL:${STARTUP_FXC}>:${STARTUP_FXC}>)
        set_tests_properties(startup_image_2d_array_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77)
    endforeach()
    # gfx2hlsl flat parameters keep the exported words (#1259): translation controls, then a desktop
    # draw of the translated pair with FXC 5_1 (the --pending path) and DXC 6_0.
    add_test(NAME startup_flat_param_controls COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/startup_flat_param_check.py")
    foreach(adapter IN ITEMS warp hardware)
        add_test(NAME startup_packed_screen_position_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/startup_packed_position_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter} $<$<BOOL:${STARTUP_FXC}>:${STARTUP_FXC}>)
        set_tests_properties(startup_packed_screen_position_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77)
        add_test(NAME startup_flat_param_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/startup_flat_param_check.py" draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter} $<$<BOOL:${STARTUP_FXC}>:${STARTUP_FXC}>)
        set_tests_properties(startup_flat_param_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77)
    endforeach()
    # gfx2hlsl per-lane bit operations: CPU evaluation of the translated text against an independent
    # reference with mutation controls, then a desktop draw of the translated pairs (FXC 5_1 and DXC 6_0).
    add_test(NAME startup_gfx2hlsl_bit_alu_controls COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/startup_gfx2hlsl_bit_alu_check.py")
    foreach(adapter IN ITEMS warp hardware)
        add_test(NAME startup_gfx2hlsl_bit_alu_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/startup_gfx2hlsl_bit_alu_check.py" draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter} $<$<BOOL:${STARTUP_FXC}>:${STARTUP_FXC}>)
        set_tests_properties(startup_gfx2hlsl_bit_alu_${adapter} PROPERTIES TIMEOUT 300 SKIP_RETURN_CODE 77)
    endforeach()
    add_test(NAME startup_xg_bc_tiling COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/startup_xg_bc_tiling_check.py" "$<TARGET_FILE:startup_stub_xg_xs>")
    set_tests_properties(startup_xg_bc_tiling PROPERTIES TIMEOUT 30 ENVIRONMENT "X2W_D3D12_PERMISSIVE=1")
    add_test(NAME startup_xg_tile64_tiling COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/startup_xg_tile64_tiling_check.py" "$<TARGET_FILE:startup_stub_xg_xs>")
    set_tests_properties(startup_xg_tile64_tiling PROPERTIES TIMEOUT 30 ENVIRONMENT "X2W_D3D12_PERMISSIVE=1")
    add_test(NAME startup_inline_sampler_controls COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/startup_inline_sampler_check.py")
    foreach(adapter IN ITEMS warp hardware)
        add_test(NAME startup_inline_sampler_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/startup_inline_sampler_check.py" draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter} $<$<BOOL:${STARTUP_FXC}>:${STARTUP_FXC}>)
        set_tests_properties(startup_inline_sampler_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77)
    endforeach()
    add_test(NAME startup_xg_bc7_layout COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/startup_xg_bc7_layout_check.py" "$<TARGET_FILE:startup_stub_xg_xs>")
    set_tests_properties(startup_xg_bc7_layout PROPERTIES TIMEOUT 30
        ENVIRONMENT "X2W_D3D12_PERMISSIVE=1;X2W_XG_BC7_4K_S_LAYOUT=1")
    # tools/startup/launch.py's containment (#1149): the exit code, captured stdout/stderr and trap
    # log, the timeout kill, the commit cap and the process limit, and the missing-EXE path.
    add_test(NAME startup_launch COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_launch_check.py" "$<TARGET_FILE:startup_launch_probe>")
    set_tests_properties(startup_launch PROPERTIES TIMEOUT 120)
    add_test(NAME startup_debug COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_debug_check.py" "$<TARGET_FILE:startup_debug_probe>")
    set_tests_properties(startup_debug PROPERTIES TIMEOUT 300)

    add_test(NAME startup_stream_request_warp COMMAND "${CMAKE_COMMAND}" -E env "X2W_FXC=${STARTUP_FXC}"
        "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_stream_check.py" "$<TARGET_FILE:startup_stream_consumer>")
    foreach(backend IN ITEMS warp hardware)
        add_test(NAME startup_stream_draw_${backend} COMMAND "${CMAKE_COMMAND}" -E env "X2W_FXC=${STARTUP_FXC}"
            "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_stream_draw_check.py"
            "$<TARGET_FILE:startup_stream_consumer>" "${backend}")
        set_tests_properties(startup_stream_draw_${backend} PROPERTIES TIMEOUT 90)
        add_test(NAME startup_stream_depth_${backend} COMMAND "${CMAKE_COMMAND}" -E env "X2W_FXC=${STARTUP_FXC}"
            "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_stream_depth_check.py"
            "$<TARGET_FILE:startup_stream_depth>" "${backend}")
        set_tests_properties(startup_stream_depth_${backend} PROPERTIES TIMEOUT 90 SKIP_RETURN_CODE 77)
        foreach(compiler IN ITEMS fxc dxc)
            add_test(NAME startup_mrt_${compiler}_${backend} COMMAND "${INTAKE_PYTHON}" -B
                "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_mrt_check.py"
                "$<TARGET_FILE:startup_stream_consumer>" "${backend}" "${compiler}" "${DXC_EXECUTABLE}" "${STARTUP_FXC}")
            set_tests_properties(startup_mrt_${compiler}_${backend} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77)
        endforeach()
        add_test(NAME startup_logical_view_${backend} COMMAND "${CMAKE_COMMAND}" -E env "X2W_FXC=${STARTUP_FXC}"
            "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_logical_view_check.py"
            "$<TARGET_FILE:startup_stream_consumer>" "${backend}")
        set_tests_properties(startup_logical_view_${backend} PROPERTIES TIMEOUT 90 SKIP_RETURN_CODE 77)
        add_test(NAME startup_xg_bc7_draw_${backend} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_xg_bc7_draw_check.py"
            "$<TARGET_FILE:startup_stream_consumer>" "${DXC_EXECUTABLE}" "${backend}")
        set_tests_properties(startup_xg_bc7_draw_${backend} PROPERTIES TIMEOUT 90 SKIP_RETURN_CODE 77)
        add_test(NAME startup_slot57_blend_draw_${backend} COMMAND "${CMAKE_COMMAND}" -E env "X2W_FXC=${STARTUP_FXC}"
            "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_slot57_blend_draw_check.py"
            "$<TARGET_FILE:startup_stream_consumer>" "${backend}")
        set_tests_properties(startup_slot57_blend_draw_${backend} PROPERTIES TIMEOUT 90 SKIP_RETURN_CODE 77)
        add_test(NAME startup_packed_target_draw_${backend} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_packed_target_draw_check.py"
            "$<TARGET_FILE:startup_stream_consumer>" "${DXC_EXECUTABLE}" "${backend}")
        set_tests_properties(startup_packed_target_draw_${backend} PROPERTIES TIMEOUT 90 SKIP_RETURN_CODE 77)
        add_test(NAME startup_1d_srv_${backend} COMMAND "${CMAKE_COMMAND}" -E env "X2W_FXC=${STARTUP_FXC}"
            "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_1d_srv_check.py"
            "$<TARGET_FILE:startup_stream_consumer>" ${backend})
        set_tests_properties(startup_1d_srv_${backend} PROPERTIES TIMEOUT 90 SKIP_RETURN_CODE 77)
        add_test(NAME startup_address_srv_${backend} COMMAND "${CMAKE_COMMAND}" -E env "X2W_FXC=${STARTUP_FXC}"
            "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_address_srv_check.py"
            "$<TARGET_FILE:startup_stream_consumer>" "${backend}")
        set_tests_properties(startup_address_srv_${backend} PROPERTIES TIMEOUT 90 SKIP_RETURN_CODE 77)
        add_test(NAME startup_address_buffer_srv_${backend} COMMAND "${CMAKE_COMMAND}" -E env "X2W_FXC=${STARTUP_FXC}"
            "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_address_buffer_srv_check.py"
            "$<TARGET_FILE:startup_stream_consumer>" "${backend}")
        set_tests_properties(startup_address_buffer_srv_${backend} PROPERTIES TIMEOUT 90 SKIP_RETURN_CODE 77)
        add_test(NAME startup_placed_vertex_${backend} COMMAND "${CMAKE_COMMAND}" -E env "X2W_FXC=${STARTUP_FXC}"
            "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_placed_vertex_check.py"
            "$<TARGET_FILE:startup_stream_consumer>" "${backend}")
        set_tests_properties(startup_placed_vertex_${backend} PROPERTIES TIMEOUT 90 SKIP_RETURN_CODE 77)
        add_test(NAME startup_address_bc_view_${backend} COMMAND "${CMAKE_COMMAND}" -E env "X2W_FXC=${STARTUP_FXC}"
            "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_address_bc_view_check.py"
            "$<TARGET_FILE:startup_stream_consumer>" "${backend}")
        set_tests_properties(startup_address_bc_view_${backend} PROPERTIES TIMEOUT 90 SKIP_RETURN_CODE 77)
        add_test(NAME startup_xg_bc_chain_${backend} COMMAND "${CMAKE_COMMAND}" -E env "X2W_FXC=${STARTUP_FXC}"
            "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_xg_bc_chain_check.py"
            "$<TARGET_FILE:startup_stub_xg_xs>" "$<TARGET_FILE:startup_stream_consumer>" "${backend}")
        set_tests_properties(startup_xg_bc_chain_${backend} PROPERTIES TIMEOUT 90 SKIP_RETURN_CODE 77)
        add_test(NAME startup_xg_bc_array_${backend} COMMAND "${CMAKE_COMMAND}" -E env "X2W_FXC=${STARTUP_FXC}"
            "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_xg_bc_array_check.py"
            "$<TARGET_FILE:startup_stub_xg_xs>" "$<TARGET_FILE:startup_stream_consumer>" "${backend}")
        set_tests_properties(startup_xg_bc_array_${backend} PROPERTIES TIMEOUT 90 SKIP_RETURN_CODE 77)
    endforeach()
    set_tests_properties(startup_stream_draw_hardware PROPERTIES SKIP_RETURN_CODE 77)
    set_tests_properties(startup_stream_request_warp PROPERTIES TIMEOUT 60)

    # Startup shader translator: descriptor offsets inside wide scalar loads (no game data).
    add_test(NAME startup_translator COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_translator_check.py")
    add_test(NAME startup_triage_run COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_triage_run_check.py")
    add_test(NAME startup_run_ps1 COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/startup_run_ps1_check.py")
    add_test(NAME xmem_shim_mapped_pages COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/xmem_mapped_pages_check.py" "$<TARGET_FILE:xmem_shim>")
    # Large batches: pages as slots of shared sections, adjacent pages in one view.
    add_test(NAME xmem_shim_page_runs COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/xmem_page_runs_check.py" "$<TARGET_FILE:xmem_shim>"
        "$<TARGET_FILE:xmem_shim_test_controls>")
    set_tests_properties(xmem_shim_page_runs PROPERTIES TIMEOUT 300)
    add_test(NAME docs_links COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/docs_links_check.py")
    # Compile-only: outcome classification and GVN pass shares, no wall-time threshold (#775).
    add_test(NAME dxc_compile_time_gvn COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/dxc_compile_time_check.py" "${DXC_EXECUTABLE}")
    set_tests_properties(dxc_compile_time_gvn PROPERTIES TIMEOUT 300 SKIP_RETURN_CODE 77)
    # Dispatch condstore_256 built with and without GVN; compare the whole raw UAV bit for bit (#775).
    foreach(adapter IN ITEMS warp hardware)
        add_test(NAME dxc_gvn_output_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/dxc_gvn_output_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(dxc_gvn_output_${adapter} PROPERTIES TIMEOUT 300 SKIP_RETURN_CODE 77)
    endforeach()
    # tools/setup-spirv-tools.ps1 builds the pinned tools into out/tools/spirv/bin.
    set(spirv_tool_hints "${CMAKE_SOURCE_DIR}/out/tools/spirv/bin" "$ENV{VULKAN_SDK}/Bin")
    find_program(SPIRV_WINDOW_AS spirv-as HINTS ${spirv_tool_hints})
    find_program(SPIRV_WINDOW_VAL spirv-val HINTS ${spirv_tool_hints})
    find_program(SPIRV_WINDOW_CROSS spirv-cross HINTS ${spirv_tool_hints})
    # Registered unconditionally: without the tools the wrapper reports a skip (77) rather than the
    # tests silently disappearing. Hosted CI sets X2W_REQUIRE_SPIRV_TOOLS=1 to turn skips into failures.
    foreach(adapter IN ITEMS hardware warp)
        foreach(bridge IN ITEMS address_window:draw address_window_atomic:atomic address_window_signed:address_signed
                                address_window_float:address_float address_window_vec4:address_vec4
                                address_window_qword:qword
                                sampler_views:sampler ballot_extract:ballot
                                compaction:compaction compare_sampler:compare_sampler minlod:minlod
                                minlod_exact:minlod_exact minlod_levels:minlod_levels
                                minlod_fractional:minlod_fractional)
            string(REPLACE ":" ";" bridge "${bridge}")
            list(GET bridge 0 name)
            list(GET bridge 1 driver)
            add_test(NAME spirv_${name}_${adapter} COMMAND "${INTAKE_PYTHON}" -B
                "${CMAKE_SOURCE_DIR}/tests/spirv_window/run_bridge_test.py"
                "${CMAKE_SOURCE_DIR}/tests/spirv_window/${driver}.py" "${SPIRV_WINDOW_AS}" "${SPIRV_WINDOW_VAL}"
                "${SPIRV_WINDOW_CROSS}" "${DXC_EXECUTABLE}" "$<TARGET_FILE:offscreen_check>" ${adapter})
            set_tests_properties(spirv_${name}_${adapter} PROPERTIES SKIP_RETURN_CODE 77)
        endforeach()
    endforeach()
    # Outside a Git checkout, such as a source archive, the tracked-path check cannot run.
    set_tests_properties(docs_links PROPERTIES SKIP_RETURN_CODE 77)

    enable_language(ASM_MASM)
    function(cpu_asm_fixture image source)
        if(ARGC GREATER 2)
            set(exports "${ARGV2}")
        else()
            set(exports "${source}")
        endif()
        add_library(${image} SHARED tests/cpu/${source}.asm tests/cpu/${exports}.def)
        set_target_properties(${image} PROPERTIES LINKER_LANGUAGE CXX)
        target_link_options(${image} PRIVATE /NOENTRY /NODEFAULTLIB /INCREMENTAL:NO)
    endfunction()
    function(cpu_translation image translator entry)
        cmake_parse_arguments(BIND "FFX_DESTROY;IMAGE_DATA;HOST_CPU"
            "PROVIDER;SYMBOL;LINK_NAME;CALLBACK_OFFSET;IMPORT_ARGUMENTS;RETURN_BYTES;NAME;RANGES;SECOND_PROVIDER;SECOND_SYMBOL;SECOND_LINK_NAME;SECOND_IMPORT_ARGUMENTS;SECOND_RETURN_BYTES" "STACK_OUTPUT" ${ARGN})
        if(NOT BIND_NAME)
            set(BIND_NAME ${image}_lifted)
        endif()
        set(binding)
        if(BIND_FFX_DESTROY)
            list(APPEND binding --ffx-destroy)
        endif()
        if(BIND_RANGES)
            list(APPEND binding --bind-ranges "${BIND_RANGES}")
        endif()
        if(DEFINED BIND_CALLBACK_OFFSET)
            list(APPEND binding --callback-offset "${BIND_CALLBACK_OFFSET}")
        endif()
        if(BIND_PROVIDER)
            set(binding --bind "$<TARGET_FILE:${BIND_PROVIDER}>" "${BIND_SYMBOL}")
            if(BIND_LINK_NAME)
                list(APPEND binding --link-name "${BIND_LINK_NAME}")
            endif()
            if(DEFINED BIND_IMPORT_ARGUMENTS)
                list(APPEND binding --import-arguments "${BIND_IMPORT_ARGUMENTS}" --return-bytes "${BIND_RETURN_BYTES}")
            endif()
            if(BIND_STACK_OUTPUT)
                list(APPEND binding --stack-output ${BIND_STACK_OUTPUT})
            endif()
        endif()
        # A second ordinary import binding, given as its own --bind group after the first.
        if(BIND_SECOND_PROVIDER)
            list(APPEND binding --bind "$<TARGET_FILE:${BIND_SECOND_PROVIDER}>" "${BIND_SECOND_SYMBOL}")
            if(BIND_SECOND_LINK_NAME)
                list(APPEND binding --link-name "${BIND_SECOND_LINK_NAME}")
            endif()
            list(APPEND binding --import-arguments "${BIND_SECOND_IMPORT_ARGUMENTS}"
                --return-bytes "${BIND_SECOND_RETURN_BYTES}")
        endif()
        if(BIND_IMAGE_DATA)
            list(APPEND binding --image-data)
        endif()
        if(BIND_HOST_CPU)
            list(APPEND binding --host-cpu)
        endif()
        set(lifted "${CMAKE_CURRENT_BINARY_DIR}/${BIND_NAME}.cpp")
        add_custom_command(OUTPUT "${lifted}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/${translator}.py"
                "$<TARGET_FILE:${image}>" ${entry} "${lifted}" ${binding}
            DEPENDS ${image} ${BIND_PROVIDER} ${BIND_SECOND_PROVIDER} ${BIND_RANGES} tools/${translator}.py
                tools/cpu_leaf_aot.py tools/cpu_ranges.py tools/cpu_call_aot.py VERBATIM)
        add_library(${BIND_NAME} SHARED "${lifted}")
        if(BIND_PROVIDER)
            target_link_libraries(${BIND_NAME} PRIVATE ${BIND_PROVIDER})
        endif()
        if(BIND_SECOND_PROVIDER)
            target_link_libraries(${BIND_NAME} PRIVATE ${BIND_SECOND_PROVIDER})
        endif()
    endfunction()
    cpu_asm_fixture(cpu_clzero counted_clzero)
    cpu_translation(cpu_clzero cpu_switch_aot counted,loop_dx,loop_r8,loop_r9,loop_r10)
    add_test(NAME cpu_counted_clzero COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/counted_clzero.py" "$<TARGET_FILE:cpu_clzero>" "$<TARGET_FILE:cpu_clzero_lifted>"
        "${CMAKE_SOURCE_DIR}/tools/cpu_switch_aot.py" "${CMAKE_CURRENT_BINARY_DIR}")
    set_tests_properties(cpu_counted_clzero PROPERTIES
        ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/cpu_counted_clzero.txt")

    foreach(variant IN ITEMS original inverted)
        cpu_asm_fixture(cpu_leaf_${variant} branch)
        cpu_translation(cpu_leaf_${variant} cpu_leaf_aot branch_value)
    endforeach()
    target_compile_definitions(cpu_leaf_inverted PRIVATE INVERT_BRANCH)
    add_test(NAME cpu_leaf_aot_branches COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/branches.py" "$<TARGET_FILE:cpu_leaf_original>"
        "$<TARGET_FILE:cpu_leaf_original_lifted>" "$<TARGET_FILE:cpu_leaf_inverted_lifted>")
    add_test(NAME cpu_aot_malformed_pe COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/malformed_pe_check.py")
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_call_${variant} direct)
        cpu_translation(cpu_call_${variant} cpu_call_aot call_mix)
        # Keep the linked caller's rel32 directed at the callee itself for inspection.
        target_link_options(cpu_call_${variant}_lifted PRIVATE /INCREMENTAL:NO)
    endforeach()
    target_compile_definitions(cpu_call_altered PRIVATE ALTER_RETURN)
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_repeat_${variant} repeat_call)
        cpu_translation(cpu_repeat_${variant} cpu_call_aot call_twice)
        target_link_options(cpu_repeat_${variant}_lifted PRIVATE /INCREMENTAL:NO)
    endforeach()
    target_compile_definitions(cpu_repeat_altered PRIVATE ALTER_RETURN)
    # Translation must reject this copy, which omits the trace-pointer reload.
    cpu_asm_fixture(cpu_repeat_dropped repeat_call)
    target_compile_definitions(cpu_repeat_dropped PRIVATE DROP_RELOAD)
    foreach(variant IN ITEMS original swapped first second)
        cpu_asm_fixture(cpu_pair_${variant} call_pair)
        cpu_translation(cpu_pair_${variant} cpu_call_aot call_pair)
        target_link_options(cpu_pair_${variant}_lifted PRIVATE /INCREMENTAL:NO)
    endforeach()
    target_compile_definitions(cpu_pair_swapped PRIVATE SWAP_CALLS)
    target_compile_definitions(cpu_pair_first PRIVATE ALTER_FIRST)
    target_compile_definitions(cpu_pair_second PRIVATE ALTER_SECOND)
    cpu_asm_fixture(cpu_register_probe register_probe)
    foreach(variant IN ITEMS original swapped)
        cpu_asm_fixture(cpu_switch_${variant} switch)
        cpu_translation(cpu_switch_${variant} cpu_switch_aot select_rva,select_rel,run_ops,shift_narrow,flag_select,cmov_read,select_rel_edges)
    endforeach()
    target_compile_definitions(cpu_switch_swapped PRIVATE SWAP_TABLE)
    foreach(variant IN ITEMS original swapped)
        cpu_asm_fixture(cpu_switch_inverted_${variant} switch_inverted)
        cpu_translation(cpu_switch_inverted_${variant} cpu_switch_aot inv_rva_jbe,inv_rel_jb,inv_rel_window_jbe)
    endforeach()
    target_compile_definitions(cpu_switch_inverted_swapped PRIVATE SWAP_TABLE)
    add_test(NAME cpu_aot_switch_inverted COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/switch_inverted_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_switch_inverted_original>" "$<TARGET_FILE:cpu_switch_inverted_original_lifted>"
        "$<TARGET_FILE:cpu_switch_inverted_swapped_lifted>")
    add_test(NAME cpu_aot_jump_tables COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/switch_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_switch_original>" "$<TARGET_FILE:cpu_switch_original_lifted>"
        "$<TARGET_FILE:cpu_switch_swapped_lifted>")
    cpu_asm_fixture(cpu_flag_conditions flag_conditions)
    set(flag_conditions_lifted "${CMAKE_CURRENT_BINARY_DIR}/cpu_flag_conditions_lifted.cpp")
    add_custom_command(OUTPUT "${flag_conditions_lifted}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/flag_conditions_check.py"
            emit "$<TARGET_FILE:cpu_flag_conditions>" "${flag_conditions_lifted}"
        DEPENDS cpu_flag_conditions tests/cpu/flag_conditions_check.py tools/cpu_switch_aot.py
            tools/cpu_leaf_aot.py tools/cpu_ranges.py VERBATIM)
    add_library(cpu_flag_conditions_lifted SHARED "${flag_conditions_lifted}")
    add_test(NAME cpu_aot_flag_conditions COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/flag_conditions_check.py" check
        "$<TARGET_FILE:cpu_flag_conditions>" "$<TARGET_FILE:cpu_flag_conditions_lifted>")
    set_tests_properties(cpu_aot_flag_conditions PROPERTIES
        ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/cpu_aot_flag_conditions.txt")
    set(CPU_FUZZ_SEED "0xA902" CACHE STRING "Seed for the CPU differential fuzz smoke test")
    set(CPU_FUZZ_FUNCTIONS "20" CACHE STRING "Number of generated functions in the CPU differential fuzz smoke test")
    set(cpu_fuzz_asm "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz.asm")
    set(cpu_fuzz_def "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz.def")
    add_custom_command(OUTPUT "${cpu_fuzz_asm}" "${cpu_fuzz_def}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" emit
            "${CPU_FUZZ_SEED}" "${CPU_FUZZ_FUNCTIONS}" "${cpu_fuzz_asm}" "${cpu_fuzz_def}"
        DEPENDS tools/fuzz_cpu.py VERBATIM)
    add_library(cpu_fuzz SHARED "${cpu_fuzz_asm}" "${cpu_fuzz_def}")
    set_target_properties(cpu_fuzz PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_fuzz PRIVATE /NOENTRY /NODEFAULTLIB /INCREMENTAL:NO /OPT:NOICF)
    set(cpu_fuzz_lifted "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_lifted.cpp")
    add_custom_command(OUTPUT "${cpu_fuzz_lifted}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" translate
            "${CPU_FUZZ_SEED}" "${CPU_FUZZ_FUNCTIONS}" "$<TARGET_FILE:cpu_fuzz>" "${cpu_fuzz_lifted}"
        DEPENDS cpu_fuzz tools/fuzz_cpu.py tools/cpu_switch_aot.py tools/cpu_leaf_aot.py
            tools/cpu_call_aot.py tools/cpu_ranges.py VERBATIM)
    add_library(cpu_fuzz_lifted SHARED "${cpu_fuzz_lifted}")
    target_compile_features(cpu_fuzz_lifted PRIVATE cxx_std_20)
    set(cpu_fuzz_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_lifted_o2.dll")
    add_custom_command(OUTPUT "${cpu_fuzz_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /LD "${cpu_fuzz_lifted}"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_lifted_o2.obj" "/Fe${cpu_fuzz_o2}"
        DEPENDS "${cpu_fuzz_lifted}" VERBATIM)
    add_custom_target(cpu_fuzz_lifted_o2 ALL DEPENDS "${cpu_fuzz_o2}")
    add_test(NAME cpu_aot_fuzz_smoke COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" check "${CPU_FUZZ_SEED}" "${CPU_FUZZ_FUNCTIONS}"
        "$<TARGET_FILE:cpu_fuzz>" "$<TARGET_FILE:cpu_fuzz_lifted>" "${cpu_fuzz_o2}")
    set_tests_properties(cpu_aot_fuzz_smoke PROPERTIES TIMEOUT 60)
    # Fixed RSP frames: saved nonvolatile registers, partial spill overlaps and a planted reload fault.
    set(CPU_FUZZ_FRAMES_SEED "0xF4A3" CACHE STRING "Seed for the CPU differential fuzz frame test")
    set(CPU_FUZZ_FRAMES_FUNCTIONS "16" CACHE STRING "Number of generated functions in the CPU differential fuzz frame test")
    set(cpu_fuzz_frames_asm "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_frames.asm")
    set(cpu_fuzz_frames_def "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_frames.def")
    add_custom_command(OUTPUT "${cpu_fuzz_frames_asm}" "${cpu_fuzz_frames_def}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" emit
            "${CPU_FUZZ_FRAMES_SEED}" "${CPU_FUZZ_FRAMES_FUNCTIONS}" "${cpu_fuzz_frames_asm}"
            "${cpu_fuzz_frames_def}" --area frames
        DEPENDS tools/fuzz_cpu.py VERBATIM)
    add_library(cpu_fuzz_frames SHARED "${cpu_fuzz_frames_asm}" "${cpu_fuzz_frames_def}")
    set_target_properties(cpu_fuzz_frames PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_fuzz_frames PRIVATE /NOENTRY /NODEFAULTLIB /INCREMENTAL:NO /OPT:NOICF)
    set(cpu_fuzz_frames_lifted "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_frames_lifted.cpp")
    set(cpu_fuzz_frames_mutant "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_frames_mutant.cpp")
    foreach(variant lifted mutant)
        set(mutant_flag "")
        if(variant STREQUAL "mutant")
            set(mutant_flag "--mutant")
        endif()
        add_custom_command(OUTPUT "${cpu_fuzz_frames_${variant}}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" translate
                "${CPU_FUZZ_FRAMES_SEED}" "${CPU_FUZZ_FRAMES_FUNCTIONS}" "$<TARGET_FILE:cpu_fuzz_frames>"
                "${cpu_fuzz_frames_${variant}}" --area frames ${mutant_flag}
            DEPENDS cpu_fuzz_frames tools/fuzz_cpu.py tools/cpu_switch_aot.py tools/cpu_leaf_aot.py
                tools/cpu_call_aot.py tools/cpu_ranges.py VERBATIM)
        add_library(cpu_fuzz_frames_${variant} SHARED "${cpu_fuzz_frames_${variant}}")
        target_compile_features(cpu_fuzz_frames_${variant} PRIVATE cxx_std_20)
    endforeach()
    set(cpu_fuzz_frames_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_frames_lifted_o2.dll")
    add_custom_command(OUTPUT "${cpu_fuzz_frames_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /LD "${cpu_fuzz_frames_lifted}"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_frames_lifted_o2.obj" "/Fe${cpu_fuzz_frames_o2}"
        DEPENDS "${cpu_fuzz_frames_lifted}" VERBATIM)
    add_custom_target(cpu_fuzz_frames_lifted_o2 ALL DEPENDS "${cpu_fuzz_frames_o2}")
    add_test(NAME cpu_aot_fuzz_frames COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" check "${CPU_FUZZ_FRAMES_SEED}" "${CPU_FUZZ_FRAMES_FUNCTIONS}"
        "$<TARGET_FILE:cpu_fuzz_frames>" "$<TARGET_FILE:cpu_fuzz_frames_lifted>" "${cpu_fuzz_frames_o2}"
        --area frames --mutant "$<TARGET_FILE:cpu_fuzz_frames_mutant>")
    set_tests_properties(cpu_aot_fuzz_frames PROPERTIES TIMEOUT 60)
    # Direct calls and tail jumps to leaf and frame callees, shared callees and a planted call retarget.
    set(CPU_FUZZ_CALLS_SEED "0xCA11" CACHE STRING "Seed for the CPU differential fuzz call test")
    set(CPU_FUZZ_CALLS_FUNCTIONS "16" CACHE STRING "Number of generated callers in the CPU differential fuzz call test")
    set(cpu_fuzz_calls_asm "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_calls.asm")
    set(cpu_fuzz_calls_def "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_calls.def")
    add_custom_command(OUTPUT "${cpu_fuzz_calls_asm}" "${cpu_fuzz_calls_def}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" emit
            "${CPU_FUZZ_CALLS_SEED}" "${CPU_FUZZ_CALLS_FUNCTIONS}" "${cpu_fuzz_calls_asm}"
            "${cpu_fuzz_calls_def}" --area calls
        DEPENDS tools/fuzz_cpu.py VERBATIM)
    add_library(cpu_fuzz_calls SHARED "${cpu_fuzz_calls_asm}" "${cpu_fuzz_calls_def}")
    set_target_properties(cpu_fuzz_calls PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_fuzz_calls PRIVATE /NOENTRY /NODEFAULTLIB /INCREMENTAL:NO /OPT:NOICF)
    set(cpu_fuzz_calls_lifted "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_calls_lifted.cpp")
    set(cpu_fuzz_calls_mutant "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_calls_mutant.cpp")
    foreach(variant lifted mutant)
        set(mutant_flag "")
        if(variant STREQUAL "mutant")
            set(mutant_flag "--mutant")
        endif()
        add_custom_command(OUTPUT "${cpu_fuzz_calls_${variant}}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" translate
                "${CPU_FUZZ_CALLS_SEED}" "${CPU_FUZZ_CALLS_FUNCTIONS}" "$<TARGET_FILE:cpu_fuzz_calls>"
                "${cpu_fuzz_calls_${variant}}" --area calls ${mutant_flag}
            DEPENDS cpu_fuzz_calls tools/fuzz_cpu.py tools/cpu_switch_aot.py tools/cpu_leaf_aot.py
                tools/cpu_call_aot.py tools/cpu_ranges.py VERBATIM)
        add_library(cpu_fuzz_calls_${variant} SHARED "${cpu_fuzz_calls_${variant}}")
        target_compile_features(cpu_fuzz_calls_${variant} PRIVATE cxx_std_20)
    endforeach()
    set(cpu_fuzz_calls_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_calls_lifted_o2.dll")
    add_custom_command(OUTPUT "${cpu_fuzz_calls_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /LD "${cpu_fuzz_calls_lifted}"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_calls_lifted_o2.obj" "/Fe${cpu_fuzz_calls_o2}"
        DEPENDS "${cpu_fuzz_calls_lifted}" VERBATIM)
    add_custom_target(cpu_fuzz_calls_lifted_o2 ALL DEPENDS "${cpu_fuzz_calls_o2}")
    add_test(NAME cpu_aot_fuzz_calls COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" check "${CPU_FUZZ_CALLS_SEED}" "${CPU_FUZZ_CALLS_FUNCTIONS}"
        "$<TARGET_FILE:cpu_fuzz_calls>" "$<TARGET_FILE:cpu_fuzz_calls_lifted>" "${cpu_fuzz_calls_o2}"
        --area calls --mutant "$<TARGET_FILE:cpu_fuzz_calls_mutant>")
    set_tests_properties(cpu_aot_fuzz_calls PROPERTIES TIMEOUT 60)
    # Call chains through unexported callees, skipped conditional calls and tail jumps, with swapped call arguments planted.
    set(CPU_FUZZ_CHAINS_SEED "0xCA11" CACHE STRING "Seed for the CPU differential fuzz chain test")
    set(CPU_FUZZ_CHAINS_FUNCTIONS "16" CACHE STRING "Number of exported functions in the CPU differential fuzz chain test")
    set(cpu_fuzz_chains_asm "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_chains.asm")
    set(cpu_fuzz_chains_def "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_chains.def")
    add_custom_command(OUTPUT "${cpu_fuzz_chains_asm}" "${cpu_fuzz_chains_def}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" emit
            "${CPU_FUZZ_CHAINS_SEED}" "${CPU_FUZZ_CHAINS_FUNCTIONS}" "${cpu_fuzz_chains_asm}"
            "${cpu_fuzz_chains_def}" --area chains
        DEPENDS tools/fuzz_cpu.py VERBATIM)
    add_library(cpu_fuzz_chains SHARED "${cpu_fuzz_chains_asm}" "${cpu_fuzz_chains_def}")
    set_target_properties(cpu_fuzz_chains PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_fuzz_chains PRIVATE /NOENTRY /NODEFAULTLIB /INCREMENTAL:NO /OPT:NOICF)
    set(cpu_fuzz_chains_lifted "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_chains_lifted.cpp")
    set(cpu_fuzz_chains_mutant "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_chains_mutant.cpp")
    foreach(variant lifted mutant)
        set(mutant_flag "")
        if(variant STREQUAL "mutant")
            set(mutant_flag "--mutant")
        endif()
        add_custom_command(OUTPUT "${cpu_fuzz_chains_${variant}}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" translate
                "${CPU_FUZZ_CHAINS_SEED}" "${CPU_FUZZ_CHAINS_FUNCTIONS}" "$<TARGET_FILE:cpu_fuzz_chains>"
                "${cpu_fuzz_chains_${variant}}" --area chains ${mutant_flag}
            DEPENDS cpu_fuzz_chains tools/fuzz_cpu.py tools/cpu_switch_aot.py tools/cpu_leaf_aot.py
                tools/cpu_call_aot.py tools/cpu_ranges.py VERBATIM)
        add_library(cpu_fuzz_chains_${variant} SHARED "${cpu_fuzz_chains_${variant}}")
        target_compile_features(cpu_fuzz_chains_${variant} PRIVATE cxx_std_20)
    endforeach()
    set(cpu_fuzz_chains_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_chains_lifted_o2.dll")
    add_custom_command(OUTPUT "${cpu_fuzz_chains_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /LD "${cpu_fuzz_chains_lifted}"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_chains_lifted_o2.obj" "/Fe${cpu_fuzz_chains_o2}"
        DEPENDS "${cpu_fuzz_chains_lifted}" VERBATIM)
    add_custom_target(cpu_fuzz_chains_lifted_o2 ALL DEPENDS "${cpu_fuzz_chains_o2}")
    add_test(NAME cpu_aot_fuzz_chains COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" check "${CPU_FUZZ_CHAINS_SEED}" "${CPU_FUZZ_CHAINS_FUNCTIONS}"
        "$<TARGET_FILE:cpu_fuzz_chains>" "$<TARGET_FILE:cpu_fuzz_chains_lifted>" "${cpu_fuzz_chains_o2}"
        --area chains --mutant "$<TARGET_FILE:cpu_fuzz_chains_mutant>")
    set_tests_properties(cpu_aot_fuzz_chains PROPERTIES TIMEOUT 60)

    set(CPU_FUZZ_SWITCHES_SEED "0x5717" CACHE STRING "Seed for the CPU differential fuzz switch test")
    set(CPU_FUZZ_SWITCHES_FUNCTIONS "48" CACHE STRING "Number of exported switches in the CPU differential fuzz switch test")
    set(cpu_fuzz_switches_asm "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_switches.asm")
    set(cpu_fuzz_switches_def "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_switches.def")
    add_custom_command(OUTPUT "${cpu_fuzz_switches_asm}" "${cpu_fuzz_switches_def}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" emit
            "${CPU_FUZZ_SWITCHES_SEED}" "${CPU_FUZZ_SWITCHES_FUNCTIONS}" "${cpu_fuzz_switches_asm}"
            "${cpu_fuzz_switches_def}" --area switches
        DEPENDS tools/fuzz_cpu.py VERBATIM)
    add_library(cpu_fuzz_switches SHARED "${cpu_fuzz_switches_asm}" "${cpu_fuzz_switches_def}")
    set_target_properties(cpu_fuzz_switches PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_fuzz_switches PRIVATE /NOENTRY /NODEFAULTLIB /INCREMENTAL:NO /OPT:NOICF)
    set(cpu_fuzz_switches_lifted "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_switches_lifted.cpp")
    set(cpu_fuzz_switches_mutant "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_switches_mutant.cpp")
    foreach(variant lifted mutant)
        set(mutant_flag "")
        if(variant STREQUAL "mutant")
            set(mutant_flag "--mutant")
        endif()
        add_custom_command(OUTPUT "${cpu_fuzz_switches_${variant}}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" translate
                "${CPU_FUZZ_SWITCHES_SEED}" "${CPU_FUZZ_SWITCHES_FUNCTIONS}" "$<TARGET_FILE:cpu_fuzz_switches>"
                "${cpu_fuzz_switches_${variant}}" --area switches ${mutant_flag}
            DEPENDS cpu_fuzz_switches tools/fuzz_cpu.py tools/cpu_switch_aot.py tools/cpu_leaf_aot.py
                tools/cpu_call_aot.py tools/cpu_ranges.py VERBATIM)
        add_library(cpu_fuzz_switches_${variant} SHARED "${cpu_fuzz_switches_${variant}}")
        target_compile_features(cpu_fuzz_switches_${variant} PRIVATE cxx_std_20)
    endforeach()
    set(cpu_fuzz_switches_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_switches_lifted_o2.dll")
    add_custom_command(OUTPUT "${cpu_fuzz_switches_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /LD "${cpu_fuzz_switches_lifted}"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_switches_lifted_o2.obj" "/Fe${cpu_fuzz_switches_o2}"
        DEPENDS "${cpu_fuzz_switches_lifted}" VERBATIM)
    add_custom_target(cpu_fuzz_switches_lifted_o2 ALL DEPENDS "${cpu_fuzz_switches_o2}")
    add_test(NAME cpu_aot_fuzz_switches COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" check "${CPU_FUZZ_SWITCHES_SEED}" "${CPU_FUZZ_SWITCHES_FUNCTIONS}"
        "$<TARGET_FILE:cpu_fuzz_switches>" "$<TARGET_FILE:cpu_fuzz_switches_lifted>" "${cpu_fuzz_switches_o2}"
        --area switches --mutant "$<TARGET_FILE:cpu_fuzz_switches_mutant>")
    set_tests_properties(cpu_aot_fuzz_switches PROPERTIES TIMEOUT 60)

    # Scalar float and double arithmetic, compares, conversions and moves through cpu_sse_aot, with MIN and MAX swapped planted.
    set(CPU_FUZZ_SCALAR_FP_SEED "0xF10A" CACHE STRING "Seed for the CPU differential fuzz scalar float test")
    set(CPU_FUZZ_SCALAR_FP_FUNCTIONS "60" CACHE STRING "Number of exported scalar float functions in the CPU differential fuzz scalar float test")
    set(cpu_fuzz_scalar_fp_asm "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_scalar_fp.asm")
    set(cpu_fuzz_scalar_fp_def "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_scalar_fp.def")
    add_custom_command(OUTPUT "${cpu_fuzz_scalar_fp_asm}" "${cpu_fuzz_scalar_fp_def}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" emit
            "${CPU_FUZZ_SCALAR_FP_SEED}" "${CPU_FUZZ_SCALAR_FP_FUNCTIONS}" "${cpu_fuzz_scalar_fp_asm}"
            "${cpu_fuzz_scalar_fp_def}" --area scalar_fp
        DEPENDS tools/fuzz_cpu.py VERBATIM)
    add_library(cpu_fuzz_scalar_fp SHARED "${cpu_fuzz_scalar_fp_asm}" "${cpu_fuzz_scalar_fp_def}")
    set_target_properties(cpu_fuzz_scalar_fp PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_fuzz_scalar_fp PRIVATE /NOENTRY /NODEFAULTLIB /INCREMENTAL:NO /OPT:NOICF)
    set(cpu_fuzz_scalar_fp_lifted "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_scalar_fp_lifted.cpp")
    set(cpu_fuzz_scalar_fp_mutant "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_scalar_fp_mutant.cpp")
    foreach(variant lifted mutant)
        set(mutant_flag "")
        if(variant STREQUAL "mutant")
            set(mutant_flag "--mutant")
        endif()
        add_custom_command(OUTPUT "${cpu_fuzz_scalar_fp_${variant}}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" translate
                "${CPU_FUZZ_SCALAR_FP_SEED}" "${CPU_FUZZ_SCALAR_FP_FUNCTIONS}" "$<TARGET_FILE:cpu_fuzz_scalar_fp>"
                "${cpu_fuzz_scalar_fp_${variant}}" --area scalar_fp ${mutant_flag}
            DEPENDS cpu_fuzz_scalar_fp tools/fuzz_cpu.py tools/cpu_sse_aot.py tools/cpu_switch_aot.py tools/cpu_leaf_aot.py
                tools/cpu_call_aot.py tools/cpu_ranges.py VERBATIM)
        add_library(cpu_fuzz_scalar_fp_${variant} SHARED "${cpu_fuzz_scalar_fp_${variant}}")
        target_compile_features(cpu_fuzz_scalar_fp_${variant} PRIVATE cxx_std_20)
    endforeach()
    set(cpu_fuzz_scalar_fp_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_scalar_fp_lifted_o2.dll")
    add_custom_command(OUTPUT "${cpu_fuzz_scalar_fp_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /LD "${cpu_fuzz_scalar_fp_lifted}"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_scalar_fp_lifted_o2.obj" "/Fe${cpu_fuzz_scalar_fp_o2}"
        DEPENDS "${cpu_fuzz_scalar_fp_lifted}" VERBATIM)
    add_custom_target(cpu_fuzz_scalar_fp_lifted_o2 ALL DEPENDS "${cpu_fuzz_scalar_fp_o2}")
    add_test(NAME cpu_aot_fuzz_scalar_fp COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" check "${CPU_FUZZ_SCALAR_FP_SEED}" "${CPU_FUZZ_SCALAR_FP_FUNCTIONS}"
        "$<TARGET_FILE:cpu_fuzz_scalar_fp>" "$<TARGET_FILE:cpu_fuzz_scalar_fp_lifted>" "${cpu_fuzz_scalar_fp_o2}"
        --area scalar_fp --mutant "$<TARGET_FILE:cpu_fuzz_scalar_fp_mutant>")
    set_tests_properties(cpu_aot_fuzz_scalar_fp PROPERTIES TIMEOUT 60)

    # Mid-body RSP changes (SUB and ADD RSP, PUSH and POP, arms, calls, RBP anchors), with a stale RSP planted.
    set(CPU_FUZZ_RSP_SEED "0xB0D1" CACHE STRING "Seed for the CPU differential fuzz mid-body RSP test")
    set(CPU_FUZZ_RSP_FUNCTIONS "36" CACHE STRING "Number of exported functions in the CPU differential fuzz mid-body RSP test")
    set(cpu_fuzz_rsp_asm "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_rsp.asm")
    set(cpu_fuzz_rsp_def "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_rsp.def")
    add_custom_command(OUTPUT "${cpu_fuzz_rsp_asm}" "${cpu_fuzz_rsp_def}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" emit
            "${CPU_FUZZ_RSP_SEED}" "${CPU_FUZZ_RSP_FUNCTIONS}" "${cpu_fuzz_rsp_asm}"
            "${cpu_fuzz_rsp_def}" --area rsp
        DEPENDS tools/fuzz_cpu.py VERBATIM)
    add_library(cpu_fuzz_rsp SHARED "${cpu_fuzz_rsp_asm}" "${cpu_fuzz_rsp_def}")
    set_target_properties(cpu_fuzz_rsp PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_fuzz_rsp PRIVATE /NOENTRY /NODEFAULTLIB /INCREMENTAL:NO /OPT:NOICF)
    set(cpu_fuzz_rsp_lifted "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_rsp_lifted.cpp")
    set(cpu_fuzz_rsp_mutant "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_rsp_mutant.cpp")
    foreach(variant lifted mutant)
        set(mutant_flag "")
        if(variant STREQUAL "mutant")
            set(mutant_flag "--mutant")
        endif()
        add_custom_command(OUTPUT "${cpu_fuzz_rsp_${variant}}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" translate
                "${CPU_FUZZ_RSP_SEED}" "${CPU_FUZZ_RSP_FUNCTIONS}" "$<TARGET_FILE:cpu_fuzz_rsp>"
                "${cpu_fuzz_rsp_${variant}}" --area rsp ${mutant_flag}
            DEPENDS cpu_fuzz_rsp tools/fuzz_cpu.py tools/cpu_switch_aot.py tools/cpu_leaf_aot.py
                tools/cpu_call_aot.py tools/cpu_ranges.py VERBATIM)
        add_library(cpu_fuzz_rsp_${variant} SHARED "${cpu_fuzz_rsp_${variant}}")
        target_compile_features(cpu_fuzz_rsp_${variant} PRIVATE cxx_std_20)
    endforeach()
    set(cpu_fuzz_rsp_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_rsp_lifted_o2.dll")
    add_custom_command(OUTPUT "${cpu_fuzz_rsp_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /LD "${cpu_fuzz_rsp_lifted}"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_rsp_lifted_o2.obj" "/Fe${cpu_fuzz_rsp_o2}"
        DEPENDS "${cpu_fuzz_rsp_lifted}" VERBATIM)
    add_custom_target(cpu_fuzz_rsp_lifted_o2 ALL DEPENDS "${cpu_fuzz_rsp_o2}")
    add_test(NAME cpu_aot_fuzz_rsp COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" check "${CPU_FUZZ_RSP_SEED}" "${CPU_FUZZ_RSP_FUNCTIONS}"
        "$<TARGET_FILE:cpu_fuzz_rsp>" "$<TARGET_FILE:cpu_fuzz_rsp_lifted>" "${cpu_fuzz_rsp_o2}"
        --area rsp --mutant "$<TARGET_FILE:cpu_fuzz_rsp_mutant>")
    set_tests_properties(cpu_aot_fuzz_rsp PROPERTIES TIMEOUT 60)

    # Counted loops with backward conditional branches, early exits and a nested loop, with every back edge retargeted planted.
    set(CPU_FUZZ_LOOPS_SEED "0x100F" CACHE STRING "Seed for the CPU differential fuzz loop test")
    set(CPU_FUZZ_LOOPS_FUNCTIONS "32" CACHE STRING "Number of exported functions in the CPU differential fuzz loop test")
    set(cpu_fuzz_loops_asm "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_loops.asm")
    set(cpu_fuzz_loops_def "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_loops.def")
    add_custom_command(OUTPUT "${cpu_fuzz_loops_asm}" "${cpu_fuzz_loops_def}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" emit
            "${CPU_FUZZ_LOOPS_SEED}" "${CPU_FUZZ_LOOPS_FUNCTIONS}" "${cpu_fuzz_loops_asm}"
            "${cpu_fuzz_loops_def}" --area loops
        DEPENDS tools/fuzz_cpu.py VERBATIM)
    add_library(cpu_fuzz_loops SHARED "${cpu_fuzz_loops_asm}" "${cpu_fuzz_loops_def}")
    set_target_properties(cpu_fuzz_loops PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_fuzz_loops PRIVATE /NOENTRY /NODEFAULTLIB /INCREMENTAL:NO /OPT:NOICF)
    set(cpu_fuzz_loops_lifted "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_loops_lifted.cpp")
    set(cpu_fuzz_loops_mutant "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_loops_mutant.cpp")
    foreach(variant lifted mutant)
        set(mutant_flag "")
        if(variant STREQUAL "mutant")
            set(mutant_flag "--mutant")
        endif()
        add_custom_command(OUTPUT "${cpu_fuzz_loops_${variant}}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" translate
                "${CPU_FUZZ_LOOPS_SEED}" "${CPU_FUZZ_LOOPS_FUNCTIONS}" "$<TARGET_FILE:cpu_fuzz_loops>"
                "${cpu_fuzz_loops_${variant}}" --area loops ${mutant_flag}
            DEPENDS cpu_fuzz_loops tools/fuzz_cpu.py tools/cpu_switch_aot.py tools/cpu_leaf_aot.py
                tools/cpu_call_aot.py tools/cpu_ranges.py VERBATIM)
        add_library(cpu_fuzz_loops_${variant} SHARED "${cpu_fuzz_loops_${variant}}")
        target_compile_features(cpu_fuzz_loops_${variant} PRIVATE cxx_std_20)
    endforeach()
    set(cpu_fuzz_loops_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_loops_lifted_o2.dll")
    add_custom_command(OUTPUT "${cpu_fuzz_loops_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /LD "${cpu_fuzz_loops_lifted}"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_fuzz_loops_lifted_o2.obj" "/Fe${cpu_fuzz_loops_o2}"
        DEPENDS "${cpu_fuzz_loops_lifted}" VERBATIM)
    add_custom_target(cpu_fuzz_loops_lifted_o2 ALL DEPENDS "${cpu_fuzz_loops_o2}")
    add_test(NAME cpu_aot_fuzz_loops COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tools/fuzz_cpu.py" check "${CPU_FUZZ_LOOPS_SEED}" "${CPU_FUZZ_LOOPS_FUNCTIONS}"
        "$<TARGET_FILE:cpu_fuzz_loops>" "$<TARGET_FILE:cpu_fuzz_loops_lifted>" "${cpu_fuzz_loops_o2}"
        --area loops --mutant "$<TARGET_FILE:cpu_fuzz_loops_mutant>")
    set_tests_properties(cpu_aot_fuzz_loops PROPERTIES TIMEOUT 60)
    cpu_asm_fixture(cpu_switch_addr32 switch_addr32)
    cpu_asm_fixture(cpu_switch_strings switch_strings)
    cpu_asm_fixture(cpu_string_probe string_probe)
    set(switch_strings_source "${CMAKE_CURRENT_BINARY_DIR}/cpu_switch_strings_lifted.cpp")
    add_custom_command(OUTPUT "${switch_strings_source}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/switch_strings_check.py"
            emit "$<TARGET_FILE:cpu_switch_strings>" "${switch_strings_source}"
        DEPENDS cpu_switch_strings tests/cpu/switch_strings_check.py tools/cpu_switch_aot.py
            tools/cpu_leaf_aot.py tools/cpu_ranges.py VERBATIM)
    add_library(cpu_switch_strings_lifted SHARED "${switch_strings_source}")
    set(switch_strings_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_switch_strings_lifted_o2.dll")
    add_custom_command(OUTPUT "${switch_strings_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /LD "${switch_strings_source}"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_switch_strings_lifted_o2.obj" "/Fe${switch_strings_o2}"
        DEPENDS "${switch_strings_source}" VERBATIM)
    add_custom_target(cpu_switch_strings_lifted_o2 ALL DEPENDS "${switch_strings_o2}")
    set(switch_strings_mutants "${CMAKE_CURRENT_BINARY_DIR}/cpu_switch_strings_mutants.cpp")
    add_custom_command(OUTPUT "${switch_strings_mutants}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/switch_strings_check.py"
            mutants "$<TARGET_FILE:cpu_switch_strings>" "${switch_strings_mutants}"
        DEPENDS cpu_switch_strings tests/cpu/switch_strings_check.py tools/cpu_switch_aot.py
            tools/cpu_leaf_aot.py tools/cpu_ranges.py VERBATIM)
    set(switch_strings_mutants_dll "${CMAKE_CURRENT_BINARY_DIR}/cpu_switch_strings_mutants.dll")
    add_custom_command(OUTPUT "${switch_strings_mutants_dll}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /LD "${switch_strings_mutants}"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_switch_strings_mutants.obj" "/Fe${switch_strings_mutants_dll}"
        DEPENDS "${switch_strings_mutants}" VERBATIM)
    add_custom_target(cpu_switch_strings_mutants ALL DEPENDS "${switch_strings_mutants_dll}")
    add_test(NAME cpu_aot_switch_strings COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/switch_strings_check.py" check "$<TARGET_FILE:cpu_string_probe>"
        "$<TARGET_FILE:cpu_guarded_call>" "$<TARGET_FILE:cpu_switch_strings>"
        "$<TARGET_FILE:cpu_switch_strings_lifted>" "${switch_strings_o2}"
        "${switch_strings_mutants_dll}")
    set_tests_properties(cpu_aot_switch_strings PROPERTIES TIMEOUT 60
        ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/cpu_aot_switch_strings.txt")
    cpu_translation(cpu_switch_addr32 cpu_switch_aot lea_wrap,lea_back,load_back,store_ahead)
    add_test(NAME cpu_aot_switch_addr32 COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/switch_addr32_check.py" "$<TARGET_FILE:cpu_switch_addr32>"
        "$<TARGET_FILE:cpu_switch_addr32_lifted>")
    cpu_asm_fixture(cpu_entry_order entry_order)
    cpu_translation(cpu_entry_order cpu_switch_aot square_tail,inc_square,pick_tail,branch_next)
    add_test(NAME cpu_aot_entry_order COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/entry_order_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_entry_order>" "$<TARGET_FILE:cpu_entry_order_lifted>")
    cpu_asm_fixture(cpu_dead_read dead_read)
    cpu_translation(cpu_dead_read cpu_switch_aot
        read_dead,cmp_dead,byte_dead,pair_dead,read_live,read_skip,sum_zero,add_in_place)
    # The same translation built with /O2, where the compiler removes a read that the generated C++ lets it remove.
    set(dead_read_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_dead_read_lifted_o2.dll")
    add_custom_command(OUTPUT "${dead_read_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /LD "${CMAKE_CURRENT_BINARY_DIR}/cpu_dead_read_lifted.cpp"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_dead_read_lifted_o2.obj" "/Fe${dead_read_o2}"
        DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/cpu_dead_read_lifted.cpp" VERBATIM)
    add_custom_target(cpu_dead_read_lifted_o2 ALL DEPENDS "${dead_read_o2}")
    add_test(NAME cpu_aot_dead_reads COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/dead_read_check.py" "$<TARGET_FILE:cpu_guarded_call>"
        "$<TARGET_FILE:cpu_dead_read>" "$<TARGET_FILE:cpu_dead_read_lifted>" "${dead_read_o2}")
    # The same control for the shared integer lowering (ffx_version_query.lower, with its x87 profile).
    cpu_asm_fixture(cpu_lower_dead_read lower_dead_read)
    set(lower_dead_read_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_lower_dead_read_lifted.cpp")
    add_custom_command(OUTPUT "${lower_dead_read_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/lower_dead_read.py"
            emit "$<TARGET_FILE:cpu_lower_dead_read>" "${lower_dead_read_fixture}"
        DEPENDS cpu_lower_dead_read tests/cpu/lower_dead_read.py tools/ffx_version_query.py tools/x87_lower.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_lower_dead_read_lifted SHARED "${lower_dead_read_fixture}")
    set(lower_dead_read_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_lower_dead_read_lifted_o2.dll")
    add_custom_command(OUTPUT "${lower_dead_read_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /EHsc /LD "${lower_dead_read_fixture}"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_lower_dead_read_lifted_o2.obj" "/Fe${lower_dead_read_o2}"
        DEPENDS "${lower_dead_read_fixture}" VERBATIM)
    add_custom_target(cpu_lower_dead_read_lifted_o2 ALL DEPENDS "${lower_dead_read_o2}")
    add_test(NAME cpu_aot_lower_dead_reads COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/lower_dead_read.py" check "$<TARGET_FILE:cpu_guarded_call>"
        "$<TARGET_FILE:cpu_lower_dead_read>" "$<TARGET_FILE:cpu_lower_dead_read_lifted>" "${lower_dead_read_o2}")
    cpu_asm_fixture(cpu_narrow narrow)
    cpu_translation(cpu_narrow cpu_switch_aot
        byte_merge,word_ops,byte_shift,byte_unary,al_less,mixed_width,ax_copy,zx_byte,set_less,set_zx,store_only,store_ret0,self_loop,qword_ops,flag_results,imul3_fresh)
    add_test(NAME cpu_aot_narrow_returns COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/narrow_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_narrow>" "$<TARGET_FILE:cpu_narrow_lifted>")
    cpu_asm_fixture(cpu_fourth_arg fourth_arg)
    # ":4" gives an entry a fourth integer argument in R9; r9_scratch keeps the three-argument default.
    cpu_translation(cpu_fourth_arg cpu_switch_aot r9_mix:4,r9_loop:4,no_r9:4,r9_scratch)
    add_test(NAME cpu_aot_fourth_argument COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/fourth_arg_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_fourth_arg>" "$<TARGET_FILE:cpu_fourth_arg_lifted>"
        "${CMAKE_CURRENT_BINARY_DIR}/cpu_fourth_arg_lifted.cpp"
        "$<TARGET_FILE:cpu_narrow>" "$<TARGET_FILE:cpu_narrow_lifted>")
    cpu_asm_fixture(cpu_image_data image_data)
    # IMAGE_DATA: RIP-relative data addresses a mapping of the image that the test binds at run time.
    cpu_translation(cpu_image_data cpu_switch_aot
        read_counter,bump_counter,counter_address,load_counter_ptr,is_counter,set_vtable,vtable_entry,swap_scratch,load_through,switch_count,switch_only
        IMAGE_DATA)
    add_test(NAME cpu_aot_image_data COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/image_data_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_register_probe>" "$<TARGET_FILE:cpu_image_data>" "$<TARGET_FILE:cpu_image_data_lifted>")

    cpu_asm_fixture(cpu_frame frame)
    # frame_bound is translated and compiled but never called.
    cpu_translation(cpu_frame cpu_switch_aot
        push_loop,spill_widths,home_save,home_args,two_paths,home_spill,loop_counter,realloc_slot,anchored_frame,frame_bound)
    add_test(NAME cpu_aot_stack_frames COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/frame_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_frame>" "$<TARGET_FILE:cpu_frame_lifted>"
        "${CMAKE_CURRENT_BINARY_DIR}/cpu_frame_lifted.cpp")
    cpu_asm_fixture(cpu_leave leave)
    # LEAVE on a proven RBP anchor; the other exports of leave.asm exist only to be rejected.
    cpu_translation(cpu_leave cpu_switch_aot leave_frame,leave_lea,leave_join)
    add_test(NAME cpu_aot_leave COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/leave_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_leave>" "$<TARGET_FILE:cpu_leave_lifted>"
        "${CMAKE_CURRENT_BINARY_DIR}/cpu_leave_lifted.cpp")
    cpu_asm_fixture(cpu_calls calls)
    cpu_translation(cpu_calls cpu_switch_aot call_leaf,chain3,narrow_call,narrow_tail,tail_call,tail_after_frame,home_save_call)
    add_test(NAME cpu_aot_direct_calls COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/calls_check.py" check "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_calls>" "$<TARGET_FILE:cpu_calls_lifted>")
    cpu_asm_fixture(cpu_call_entries call_entries)
    cpu_translation(cpu_call_entries cpu_switch_aot entry_caller,entry_tail)
    add_test(NAME cpu_aot_call_entries COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/call_entries_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_call_entries>" "$<TARGET_FILE:cpu_call_entries_lifted>")
    # Callees without unwind data, bound by a [start, end) manifest from the source labels in the linker map.
    cpu_asm_fixture(cpu_bare_calls bare_calls)
    target_link_options(cpu_bare_calls PRIVATE "/MAP:${CMAKE_CURRENT_BINARY_DIR}/cpu_bare_calls.map")
    set(bare_ranges "${CMAKE_CURRENT_BINARY_DIR}/cpu_bare_calls_ranges.json")
    add_custom_command(OUTPUT "${bare_ranges}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/range_manifest.py" "$<TARGET_FILE:cpu_bare_calls>"
            "${CMAKE_CURRENT_BINARY_DIR}/cpu_bare_calls.map" "${bare_ranges}" bare_mix bare_low
        DEPENDS cpu_bare_calls tools/range_manifest.py VERBATIM)
    cpu_translation(cpu_bare_calls cpu_switch_aot call_bare,twice_bare,tail_bare RANGES "${bare_ranges}")
    add_test(NAME cpu_aot_bound_callees COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/bare_calls_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_bare_calls>" "$<TARGET_FILE:cpu_bare_calls_lifted>" "${bare_ranges}"
        "${CMAKE_CURRENT_BINARY_DIR}/cpu_bare_calls.map")
    cpu_asm_fixture(cpu_wide_arith wide_arith)
    cpu_translation(cpu_wide_arith cpu_switch_aot
        div32,idiv32,div64,udiv128,idiv64,mul64,imul64_1,mul32,imul32,mul8,mul16,widen,sign_dx)
    add_test(NAME cpu_aot_wide_arithmetic COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/wide_arith_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_wide_arith>" "$<TARGET_FILE:cpu_wide_arith_lifted>")
    cpu_asm_fixture(cpu_switch_rotates switch_rotates)
    cpu_translation(cpu_switch_rotates cpu_switch_aot
        rot_reg_cl_8,rot_reg_cl_16,rot_reg_cl_32,rot_reg_cl_64,rot_imm_1,rot_imm_edge,rot_mem_cl_8_16,rot_mem_cl_32_64,rot_mem_imm,rot_narrow_ret)
    add_test(NAME cpu_aot_switch_rotates COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/switch_rotates_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_switch_rotates>" "$<TARGET_FILE:cpu_switch_rotates_lifted>")
    cpu_asm_fixture(cpu_switch_shift_flags switch_shift_flags)
    cpu_translation(cpu_switch_shift_flags cpu_switch_aot
        shl1_8,shr1_8,sar1_8,shl1_16,shr1_16,sar1_16,shl1_32,shr1_32,sar1_32,shl1_64,shr1_64,sar1_64,shln_8,shrn_8,sarn_8,shln_16,shrn_16,sarn_16,shln_32,shrn_32,sarn_32,shln_64,shrn_64,sarn_64,shlw_16,sarw_8,shlw_64,shln_32m,jcc_shl64_o,jcc_shr32_b,jcc_sar16_s,jcc_shl8_be,jcc_sar64_g,jcc_shr8_ge,jcc_shrn_np,cmov_shift)
    add_test(NAME cpu_aot_switch_shift_flags COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/switch_shift_flags_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_switch_shift_flags>" "$<TARGET_FILE:cpu_switch_shift_flags_lifted>")
    cpu_asm_fixture(cpu_switch_bit_tests switch_bit_tests)
    cpu_translation(cpu_switch_bit_tests cpu_switch_aot
        bt16_reg,bt32_reg,bt64_reg,bts16_reg,bts32_reg,bts64_reg,btr16_reg,btr32_reg,btr64_reg,btc16_reg,btc32_reg,btc64_reg,bt16_imm,bt32_imm,bt64_imm,bts16_imm,bts32_imm,bts64_imm,btr16_imm,btr32_imm,btr64_imm,btc16_imm,btc32_imm,btc64_imm,bt16_mem,bt32_mem,bt64_mem,bts16_mem,bts32_mem,bts64_mem,btr16_mem,btr32_mem,btr64_mem,btc16_mem,btc32_mem,btc64_mem,jcc_bt,jcc_btr_nc,cmov_bt)
    add_test(NAME cpu_aot_switch_bit_tests COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/switch_bit_tests_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_switch_bit_tests>" "$<TARGET_FILE:cpu_switch_bit_tests_lifted>")
    cpu_asm_fixture(cpu_switch_bit_counts switch_bit_counts)
    cpu_translation(cpu_switch_bit_counts cpu_switch_aot
        popcnt16_reg,popcnt16_mem,popcnt32_reg,popcnt32_mem,popcnt64_reg,popcnt64_mem,lzcnt16_reg,lzcnt16_mem,lzcnt32_reg,lzcnt32_mem,lzcnt64_reg,lzcnt64_mem,tzcnt16_reg,tzcnt16_mem,tzcnt32_reg,tzcnt32_mem,tzcnt64_reg,tzcnt64_mem,jcc_tzcnt_b,jcc_lzcnt_e,jcc_popcnt_ne,cmov_counts)
    add_test(NAME cpu_aot_switch_bit_counts COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/switch_bit_counts_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_switch_bit_counts>" "$<TARGET_FILE:cpu_switch_bit_counts_lifted>")
    # A host without POPCNT, BMI1 or ABM cannot run the original as the reference describes it.
    set_tests_properties(cpu_aot_switch_bit_counts PROPERTIES SKIP_RETURN_CODE 77)
    cpu_asm_fixture(cpu_switch_bswap switch_bswap)
    cpu_translation(cpu_switch_bswap cpu_switch_aot
        bswap32_rax,bswap64_rax,bswap32_rcx,bswap64_rcx,bswap32_rdx,bswap64_rdx,bswap32_r8,bswap64_r8,bswap32_r9,bswap64_r9,bswap32_r10,bswap64_r10,bswap32_r11,bswap64_r11,bswap32_rbx,bswap64_rbx,bswap32_rbp,bswap64_rbp,bswap32_rsi,bswap64_rsi,bswap32_rdi,bswap64_rdi,bswap32_r12,bswap64_r12,bswap32_r13,bswap64_r13,bswap32_r14,bswap64_r14,bswap32_r15,bswap64_r15,bswap_mixed,jcc_bswap_b,cmov_bswap)
    add_test(NAME cpu_aot_switch_bswap COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/switch_bswap_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_switch_bswap>" "$<TARGET_FILE:cpu_switch_bswap_lifted>")
    cpu_asm_fixture(cpu_switch_fsgsbase_rdpid switch_fsgsbase_rdpid)
    cpu_translation(cpu_switch_fsgsbase_rdpid cpu_switch_aot
        "rdfsbase_32,rdfsbase_64,rdgsbase_32,rdgsbase_64,read_host_nonvolatile,rdpid_64,rdpid_twice,flag_preserve_fsgsbase_rdpid,unused_host_reads"
        HOST_CPU)
    add_test(NAME cpu_aot_switch_fsgsbase_rdpid COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/switch_fsgsbase_rdpid_check.py"
        "$<TARGET_FILE:cpu_switch_fsgsbase_rdpid>"
        "$<TARGET_FILE:cpu_switch_fsgsbase_rdpid_lifted>"
        "${CMAKE_CXX_COMPILER}")
    set_tests_properties(cpu_aot_switch_fsgsbase_rdpid PROPERTIES
        SKIP_RETURN_CODE 77
        ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/cpu_aot_switch_fsgsbase_rdpid.txt")
    # BSF and BSR; the swapped build assembles three exports with the other scan and must follow the exchanged reference.
    foreach(variant IN ITEMS original swapped)
        cpu_asm_fixture(cpu_switch_bit_scans_${variant} switch_bit_scans)
        cpu_translation(cpu_switch_bit_scans_${variant} cpu_switch_aot
            bsf16_reg,bsf16_mem,bsf32_reg,bsf32_mem,bsf64_reg,bsf64_mem,bsr16_reg,bsr16_mem,bsr32_reg,bsr32_mem,bsr64_reg,bsr64_mem,bsr32_self,bsf16_self,jcc_bsf_e,jcc_bsr_ne,cmov_scans)
    endforeach()
    target_compile_definitions(cpu_switch_bit_scans_swapped PRIVATE SWAP_SCANS)
    add_test(NAME cpu_aot_switch_bit_scans COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/switch_bit_scans_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_switch_bit_scans_original>" "$<TARGET_FILE:cpu_switch_bit_scans_original_lifted>"
        "$<TARGET_FILE:cpu_switch_bit_scans_swapped_lifted>")
    # Divides whose results are unused: the translation is also built with /O2, where the compiler would remove a
    # divide that the generated C++ let it remove, and both builds are compared with the original's exception codes.
    cpu_asm_fixture(cpu_dead_div dead_div)
    cpu_translation(cpu_dead_div cpu_switch_aot div32_dead,idiv32_dead,udiv128_dead,idiv64_dead,div_skip)
    add_library(cpu_guarded_call SHARED tests/cpu/guarded_call.cpp)
    set(dead_div_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_dead_div_lifted_o2.dll")
    add_custom_command(OUTPUT "${dead_div_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /LD "${CMAKE_CURRENT_BINARY_DIR}/cpu_dead_div_lifted.cpp"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_dead_div_lifted_o2.obj" "/Fe${dead_div_o2}"
        DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/cpu_dead_div_lifted.cpp" VERBATIM)
    add_custom_target(cpu_dead_div_lifted_o2 ALL DEPENDS "${dead_div_o2}")
    add_test(NAME cpu_aot_dead_divides COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/dead_div_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_guarded_call>" "$<TARGET_FILE:cpu_dead_div>" "$<TARGET_FILE:cpu_dead_div_lifted>"
        "${dead_div_o2}")
    set(cpu_avx2_exports
        "op_broadcasti128,op_extracti128_lo,op_extracti128_hi,op_inserti128_lo,op_inserti128_hi,"
        "op_perm2i128_mix,op_perm2i128_swap,op_perm2i128_zero_lo,op_perm2i128_zero_hi,"
        "op_perm2i128_zero_both,op_perm2i128_ignored,op_pbroadcastb_xmm,op_pbroadcastb_ymm,"
        "op_pbroadcastw_xmm,op_pbroadcastw_ymm,op_pbroadcastd_xmm,op_pbroadcastd_ymm,"
        "op_pbroadcastq_xmm,op_pbroadcastq_ymm,op_permd,op_permps,op_permq,op_permq_dup,"
        "op_permpd,op_pblendd_xmm,op_pblendd_ymm,op_pblendd_ymm_all,op_pblendd_ymm_none,"
        "op_psllvd_xmm,op_psllvd_ymm,op_psllvq_xmm,op_psllvq_ymm,op_psravd_xmm,op_psravd_ymm,"
        "op_psrlvd_xmm,op_psrlvd_ymm,op_psrlvq_xmm,op_psrlvq_ymm,op_zeroupper,op_pipeline,"
        "op_movdqu_store128,op_movdqu_copy128,"
        "op_gatherdps_xmm,op_gatherdps_ymm,op_gatherdps_ymm_scale1_disp,op_gatherdpd_xmm,"
        "op_gatherdpd_ymm,op_gatherqps_xmm_vm64x,op_gatherqps_xmm_vm64y,op_gatherqpd_xmm,"
        "op_gatherqpd_ymm,op_pgatherdd_xmm,op_pgatherdd_ymm,op_pgatherdq_xmm,op_pgatherdq_ymm,"
        "op_pgatherqd_xmm_vm64x,op_pgatherqd_xmm_vm64y,op_pgatherqq_xmm,op_pgatherqq_ymm,"
        "op_pgatherdd_ymm_rdx,"
        "op_pmaskmovd_load_xmm,op_pmaskmovd_load_ymm,op_pmaskmovq_load_xmm,op_pmaskmovq_load_ymm,"
        "op_pmaskmovd_store_xmm,op_pmaskmovd_store_ymm,op_pmaskmovq_store_xmm,op_pmaskmovq_store_ymm")
    string(CONCAT cpu_avx2_exports ${cpu_avx2_exports})
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_avx2_regs_${variant} avx2_regs)
        cpu_translation(cpu_avx2_regs_${variant} cpu_avx2_aot "${cpu_avx2_exports}")
    endforeach()
    target_compile_definitions(cpu_avx2_regs_altered PRIVATE ALTER_IMM)
    add_test(NAME cpu_aot_avx2_regs COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/avx2_regs_check.py"
        "$<TARGET_FILE:cpu_avx2_regs_original>"
        "$<TARGET_FILE:cpu_avx2_regs_original_lifted>"
        "$<TARGET_FILE:cpu_avx2_regs_altered_lifted>")
    set(cpu_avx_int_exports
        "op_pabsb,op_pabsw,op_pabsd,"
        "op_packsswb,op_packssdw,op_packusdw,"
        "op_paddb,op_paddq,op_paddsb,op_paddusb,op_paddusw,"
        "op_psubb,op_psubd,op_psubq,op_psubsb,op_psubsw,op_psubusb,op_psubusw,"
        "op_palignr_0,op_palignr_5,op_palignr_16,op_palignr_23,op_palignr_32,"
        "op_pblendvb,op_pblendw,op_pblendw_all,op_pblendw_none,"
        "op_pcmpeqb,op_pcmpeqw,op_pcmpeqd,op_pcmpeqq,op_pcmpgtb,op_pcmpgtw,op_pcmpgtd,op_pcmpgtq,"
        "op_pmaxsb,op_pmaxsd,op_pmaxub,op_pmaxuw,op_pmaxud,"
        "op_pminsb,op_pminsd,op_pminub,op_pminuw,op_pminud,"
        "op_phaddw,op_phaddd,op_phaddsw,op_phsubw,op_phsubd,op_phsubsw,op_phminposuw,"
        "op_pmaddwd,op_pmuldq,op_pmuludq,op_pmulhrsw,op_pmulhuw,op_pmulld,"
        "op_psadbw,op_mpsadbw_0,op_mpsadbw_3,op_mpsadbw_7_ignored,"
        "op_pshufhw,op_pshuflw,"
        "op_psignb,op_psignw,op_psignd,"
        "op_pmovsxbw,op_pmovsxbd,op_pmovsxbq,op_pmovsxwd,op_pmovsxwq,op_pmovsxdq,"
        "op_pmovzxbw,op_pmovzxbd,op_pmovzxbq,op_pmovzxwd,op_pmovzxwq,op_pmovzxdq,"
        "op_punpckldq,op_punpckhdq,op_punpcklqdq,op_punpckhqdq,"
        "op_pslld_xmm,op_psllq_xmm,op_psrld_xmm,op_psrlq_xmm,"
        "op_pslld_imm,op_pslld_imm_ovf,op_psllq_imm,op_psllq_imm_ovf,"
        "op_psrld_imm,op_psrld_imm_ovf,op_psrlq_imm,op_psrlq_imm_ovf,"
        "op_pmovmskb,"
        "op_pextrb_reg,op_pextrb_mem,op_pextrw_reg,op_pextrw_mem,"
        "op_pextrd_reg,op_pextrd_mem,op_pextrq_reg,op_pextrq_mem,"
        "op_pinsrb_reg,op_pinsrb_mem,op_pinsrw_reg,op_pinsrw_mem,"
        "op_pinsrd_reg,op_pinsrd_mem,op_pinsrq_reg,op_pinsrq_mem,"
        "op_zeroupper,op_zeroall,op_pipeline,op_movdqu_copy128,op_mov_store32,op_movdqu_regcopy,op_movdqu_store128,"
        "op_palignr_69,op_mpsadbw_6,op_pshuflw_1b,op_psllq_imm_129,op_pextrw_reg_2,op_pextrd_reg_2,"
        "op_pinsrb_reg_upper,op_pinsrw_reg_3,op_zeroupper_ymm1,op_movdqu_load128,"
        "op_palignr_255,op_pmovsxbq_wide_src,op_zeroupper_ymm2")
    string(CONCAT cpu_avx_int_exports ${cpu_avx_int_exports})
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_avx_int_regs_${variant} avx_int_regs)
        cpu_translation(cpu_avx_int_regs_${variant} cpu_avx_int_aot "${cpu_avx_int_exports}")
    endforeach()
    target_compile_definitions(cpu_avx_int_regs_altered PRIVATE ALTER_IMM)
    add_test(NAME cpu_aot_avx_int_regs COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/avx_int_regs_check.py"
        "$<TARGET_FILE:cpu_avx_int_regs_original>"
        "$<TARGET_FILE:cpu_avx_int_regs_original_lifted>"
        "$<TARGET_FILE:cpu_avx_int_regs_altered_lifted>")
    set(cpu_avx_int_ymm_addsub_exports
        "op_vpaddb_ymm,op_vpaddw_ymm,op_vpaddd_ymm,op_vpaddq_ymm,"
        "op_vpsubb_ymm,op_vpsubw_ymm,op_vpsubd_ymm,op_vpsubq_ymm,"
        "op_vpcmpeqb_ymm,op_vpcmpeqw_ymm,op_vpcmpeqd_ymm,op_vpcmpeqq_ymm,"
        "op_vpcmpgtb_ymm,op_vpcmpgtw_ymm,op_vpcmpgtd_ymm,op_vpcmpgtq_ymm,"
        "op_vpsubq_ymm_dst_src1,op_vpsubd_ymm_dst_src2,op_vpcmpgtq_ymm_dst_src2,op_vpaddw_ymm_all,"
        "op_vpcmpeqd_ymm_all,op_vpcmpgtb_ymm_all,op_vpaddq_ymm_then_xmm_subb,op_vpcmpeqw_ymm_then_xmm_addb,"
        "op_chain_ymm")
    string(CONCAT cpu_avx_int_ymm_addsub_exports ${cpu_avx_int_ymm_addsub_exports})
    cpu_asm_fixture(cpu_avx_int_ymm_addsub avx_int_ymm_addsub)
    cpu_translation(cpu_avx_int_ymm_addsub cpu_avx_int_aot "${cpu_avx_int_ymm_addsub_exports}")
    add_test(NAME cpu_aot_avx_int_ymm_addsub COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/avx_int_ymm_addsub_check.py"
        "$<TARGET_FILE:cpu_avx_int_ymm_addsub>"
        "$<TARGET_FILE:cpu_avx_int_ymm_addsub_lifted>")
    set_tests_properties(cpu_aot_avx_int_ymm_addsub PROPERTIES SKIP_RETURN_CODE 77)
    set(cpu_avx_int_ymm_minmax_exports
        "op_vpmaxsb_ymm,op_vpmaxsw_ymm,op_vpmaxsd_ymm,op_vpmaxub_ymm,op_vpmaxuw_ymm,op_vpmaxud_ymm,"
        "op_vpminsb_ymm,op_vpminsw_ymm,op_vpminsd_ymm,op_vpminub_ymm,op_vpminuw_ymm,op_vpminud_ymm,"
        "op_vpmaxsd_ymm_dst_src2,op_vpminuw_ymm_dst_src1,op_vpmaxub_ymm_then_xmm_minsb")
    string(CONCAT cpu_avx_int_ymm_minmax_exports ${cpu_avx_int_ymm_minmax_exports})
    cpu_asm_fixture(cpu_avx_int_ymm_minmax avx_int_ymm_minmax)
    cpu_translation(cpu_avx_int_ymm_minmax cpu_avx_int_aot "${cpu_avx_int_ymm_minmax_exports}")
    add_test(NAME cpu_aot_avx_int_ymm_minmax COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/avx_int_ymm_minmax_check.py"
        "$<TARGET_FILE:cpu_avx_int_ymm_minmax>"
        "$<TARGET_FILE:cpu_avx_int_ymm_minmax_lifted>")
    set_tests_properties(cpu_aot_avx_int_ymm_minmax PROPERTIES SKIP_RETURN_CODE 77)
    set(cpu_avx_int_ymm_sat_exports
        "op_vpaddsb_ymm,op_vpaddsw_ymm,op_vpaddusb_ymm,op_vpaddusw_ymm,"
        "op_vpsubsb_ymm,op_vpsubsw_ymm,op_vpsubusb_ymm,op_vpsubusw_ymm,"
        "op_vpsubsb_ymm_dst_src1,op_vpsubsw_ymm_dst_src1,op_vpsubusb_ymm_dst_src1,op_vpsubusw_ymm_dst_src1,"
        "op_vpsubsb_ymm_dst_src2,op_vpsubsw_ymm_dst_src2,op_vpsubusb_ymm_dst_src2,op_vpsubusw_ymm_dst_src2,"
        "op_vpaddsb_ymm_all,op_vpaddusb_ymm_then_xmm_subsb")
    string(CONCAT cpu_avx_int_ymm_sat_exports ${cpu_avx_int_ymm_sat_exports})
    cpu_asm_fixture(cpu_avx_int_ymm_sat avx_int_ymm_sat)
    cpu_translation(cpu_avx_int_ymm_sat cpu_avx_int_aot "${cpu_avx_int_ymm_sat_exports}")
    add_test(NAME cpu_aot_avx_int_ymm_sat COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/avx_int_ymm_sat_check.py"
        "$<TARGET_FILE:cpu_avx_int_ymm_sat>"
        "$<TARGET_FILE:cpu_avx_int_ymm_sat_lifted>")
    set_tests_properties(cpu_aot_avx_int_ymm_sat PROPERTIES SKIP_RETURN_CODE 77)
    set(cpu_avx_int_ymm_abs_sign_shuf_exports
        "op_vpabsb_ymm,op_vpabsw_ymm,op_vpabsd_ymm,op_vpsignb_ymm,op_vpsignw_ymm,op_vpsignd_ymm,"
        "op_vpshufhw_ymm_1b,op_vpshufhw_ymm_93,op_vpshuflw_ymm_1b,op_vpshuflw_ymm_93,"
        "op_vpabsd_ymm_dst_src1,op_vpsignw_ymm_dst_src2,op_vpsignb_ymm_all,op_vpshufhw_ymm_dst_src1,"
        "op_vpabsb_ymm_then_xmm_vpsignb")
    string(CONCAT cpu_avx_int_ymm_abs_sign_shuf_exports ${cpu_avx_int_ymm_abs_sign_shuf_exports})
    cpu_asm_fixture(cpu_avx_int_ymm_abs_sign_shuf avx_int_ymm_abs_sign_shuf)
    cpu_translation(cpu_avx_int_ymm_abs_sign_shuf cpu_avx_int_aot "${cpu_avx_int_ymm_abs_sign_shuf_exports}")
    add_test(NAME cpu_aot_avx_int_ymm_abs_sign_shuf COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/avx_int_ymm_abs_sign_shuf_check.py"
        "$<TARGET_FILE:cpu_avx_int_ymm_abs_sign_shuf>"
        "$<TARGET_FILE:cpu_avx_int_ymm_abs_sign_shuf_lifted>")
    set_tests_properties(cpu_aot_avx_int_ymm_abs_sign_shuf PROPERTIES SKIP_RETURN_CODE 77)
    set(cpu_avx_move_exports
        "op_andpd_xmm,op_andpd_ymm,op_andps_xmm,op_andps_ymm,op_andnpd_xmm,op_andnpd_ymm,"
        "op_andnps_xmm,op_andnps_ymm,op_orpd_xmm,op_orpd_ymm,op_orps_xmm,op_orps_ymm,"
        "op_xorpd_xmm,op_xorpd_ymm,op_xorps_xmm,op_xorps_ymm,"
        "op_blendpd_xmm,op_blendpd_ymm,op_blendpd_ymm_all,op_blendpd_ymm_none,"
        "op_blendps_xmm,op_blendps_ymm,op_blendps_ymm_all,op_blendps_ymm_none,"
        "op_blendvpd_xmm,op_blendvpd_ymm,op_blendvps_xmm,op_blendvps_ymm,"
        "op_broadcastf128,op_broadcastsd_mem0,op_broadcastsd_mem1,op_broadcastsd_mem2,"
        "op_broadcastsd_mem3,op_broadcastsd_reg,op_broadcastss_xmm_mem0,op_broadcastss_xmm_mem1,"
        "op_broadcastss_xmm_mem2,op_broadcastss_xmm_mem3,op_broadcastss_ymm_mem4,op_broadcastss_ymm_mem5,"
        "op_broadcastss_ymm_mem6,op_broadcastss_ymm_mem7,op_broadcastss_xmm_reg,op_broadcastss_ymm_reg,"
        "op_extractf128_lo,op_extractf128_hi,op_insertf128_lo,op_insertf128_hi,"
        "op_extractps_lane0,op_extractps_lane1,op_extractps_lane2,op_extractps_lane3,"
        "op_insertps_s0d1_z0,op_insertps_s3d2_zmask,op_insertps_zero_all,"
        "op_perm2f128_mix,op_perm2f128_swap,op_perm2f128_zero_lo,op_perm2f128_zero_hi,"
        "op_perm2f128_zero_both,op_perm2f128_ignored,op_permilpd_imm_xmm,op_permilpd_imm_ymm,"
        "op_permilpd_imm_ymm_lo,op_permilpd_imm_ymm_hi,op_permilpd_var_xmm,op_permilpd_var_ymm,"
        "op_permilps_imm_xmm,op_permilps_imm_ymm,op_permilps_imm_ymm_lo,op_permilps_imm_ymm_hi,"
        "op_permilps_var_xmm,op_permilps_var_ymm,"
        "op_shufpd_xmm,op_shufpd_ymm,op_shufpd_ymm_lo,op_shufpd_ymm_hi,"
        "op_shufps_xmm,op_shufps_ymm,op_shufps_ymm_lo,op_shufps_ymm_hi,"
        "op_unpcklpd_xmm,op_unpcklpd_ymm,op_unpckhpd_xmm,op_unpckhpd_ymm,"
        "op_unpcklps_xmm,op_unpcklps_ymm,op_unpckhps_xmm,op_unpckhps_ymm,"
        "op_movapd_xmm,op_movapd_ymm,op_movapd_store128,"
        "op_movupd_xmm,op_movupd_ymm,op_movupd_store128,"
        "op_lddqu_xmm,op_lddqu_ymm,op_movntdq_xmm,op_movntdq_ymm,op_movntdqa_xmm,op_movntdqa_ymm,"
        "op_movntpd_xmm,op_movntpd_ymm,op_movntps_xmm,op_movntps_ymm,"
        "op_movddup_xmm,op_movddup_ymm,op_movshdup_xmm,op_movshdup_ymm,op_movsldup_xmm,op_movsldup_ymm,"
        "op_movhlps,op_movlhps,op_movhpd_load,op_movhpd_store,op_movhps_load,op_movhps_store,"
        "op_movlpd_load,op_movlpd_store,op_movlps_load,op_movlps_store,"
        "op_movsd_load,op_movsd_store,op_movsd_reg,op_movss_load,op_movss_store,op_movss_reg,"
        "op_movmskpd_xmm,op_movmskpd_ymm,op_movmskps_xmm,op_movmskps_ymm,"
        "op_maskmovps_load_xmm,op_maskmovps_load_ymm,op_maskmovpd_load_xmm,op_maskmovpd_load_ymm,"
        "op_maskmovps_store_xmm,op_maskmovps_store_ymm,op_maskmovpd_store_xmm,op_maskmovpd_store_ymm,"
        "op_zeroupper,op_pipeline,op_mov_load32,"
        "op_zeroupper_ymm1,op_movapd_xmm_dirty,op_mov_store32,op_insertps_s2d3_z1,"
        "op_extractps_w1_lane2,op_shufpd_xmm_sel1,op_shufps_ymm_sel2,op_negdisp_load")
    string(CONCAT cpu_avx_move_exports ${cpu_avx_move_exports})
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_avx_move_regs_${variant} avx_move_regs)
        cpu_translation(cpu_avx_move_regs_${variant} cpu_avx_move_aot "${cpu_avx_move_exports}")
    endforeach()
    target_compile_definitions(cpu_avx_move_regs_altered PRIVATE ALTER_IMM)
    add_test(NAME cpu_aot_avx_move_regs COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/avx_move_regs_check.py"
        "$<TARGET_FILE:cpu_avx_move_regs_original>"
        "$<TARGET_FILE:cpu_avx_move_regs_original_lifted>"
        "$<TARGET_FILE:cpu_avx_move_regs_altered_lifted>")
    set(cpu_avx_move_mem_exports
        "op_andpd_mem128,op_andps_mem128,op_andnpd_mem128,op_andnps_mem128,"
        "op_orpd_mem128,op_orps_mem128,op_xorpd_mem128,op_xorps_mem128,"
        "op_andpd_mem256,op_andps_mem256,op_andnpd_mem256,op_andnps_mem256,"
        "op_orpd_mem256,op_orps_mem256,op_xorpd_mem256,op_xorps_mem256")
    string(CONCAT cpu_avx_move_mem_exports ${cpu_avx_move_mem_exports})
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_avx_move_mem_${variant} avx_move_mem)
        cpu_translation(cpu_avx_move_mem_${variant} cpu_avx_move_aot "${cpu_avx_move_mem_exports}")
    endforeach()
    target_compile_definitions(cpu_avx_move_mem_altered PRIVATE ALTER_MUT)
    add_test(NAME cpu_aot_avx_move_mem COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/avx_move_mem_check.py"
        "$<TARGET_FILE:cpu_avx_move_mem_original>"
        "$<TARGET_FILE:cpu_avx_move_mem_original_lifted>"
        "$<TARGET_FILE:cpu_avx_move_mem_altered_lifted>")
    set_tests_properties(cpu_aot_avx_move_mem PROPERTIES SKIP_RETURN_CODE 77)
    set(cpu_avx_pcmpstr_exports
        "vpcmpestri_ub_eqany_pos_lsb,vpcmpestri_sw_ranges_neg_msb,vpcmpestri_sb_eqeach_mpos_msb,"
        "vpcmpestri_uw_eqord_mneg_lsb,vpcmpestri_mem_ub_eqany_pos_lsb,vpcmpestri_alias_xmm0,"
        "vpcmpestrm_ub_eqany_pos_bit,vpcmpestrm_sb_ranges_neg_exp,vpcmpestrm_uw_eqeach_mpos_exp,"
        "vpcmpestrm_sw_eqord_mneg_bit,vpcmpestrm_mem_ub_eqany_exp,vpcmpestrm_xmm0_self,"
        "vpcmpistri_ub_eqord_pos_lsb,vpcmpistri_sb_ranges_mneg_msb,vpcmpistri_uw_eqany_neg_msb,"
        "vpcmpistri_sw_eqeach_mpos_lsb,vpcmpistri_mem_ub_eqany_pos_lsb,"
        "vpcmpistrm_ub_eqeach_pos_exp,vpcmpistrm_sw_eqany_mneg_exp,vpcmpistrm_uw_ranges_mpos_bit,"
        "vpcmpistrm_sb_eqord_neg_bit,vpcmpistrm_mem_ub_eqord_exp,vpcmpistrm_discarded_read")
    string(CONCAT cpu_avx_pcmpstr_exports ${cpu_avx_pcmpstr_exports})
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_avx_pcmpstr_${variant} avx_pcmpstr)
        cpu_translation(cpu_avx_pcmpstr_${variant} cpu_avx_pcmpstr_aot "${cpu_avx_pcmpstr_exports}")
    endforeach()
    target_compile_definitions(cpu_avx_pcmpstr_altered PRIVATE ALTER_IMM)
    add_test(NAME cpu_aot_avx_pcmpstr COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/avx_pcmpstr.py"
        "$<TARGET_FILE:cpu_avx_pcmpstr_original>"
        "$<TARGET_FILE:cpu_avx_pcmpstr_original_lifted>"
        "$<TARGET_FILE:cpu_avx_pcmpstr_altered_lifted>")
    set_tests_properties(cpu_aot_avx_pcmpstr PROPERTIES SKIP_RETURN_CODE 77)
    set(cpu_xsave_state_original_exports
        "op_xrstor_xsave,op_xrstor64_xsave64,op_xrstor_xsave64,op_xrstor64_xsave,"
        "op_xrstor64_disp,op_xrstor64_mask_sse,op_vzeroupper_xsave64,op_pipeline,"
        "op_discarded_read,op_xrstor64_xsavec,op_xrstor64_xsavec64,"
        "op_xrstor64_xsavec64_sse,op_xrstor64_xsavec64_x87_avx,op_xsavec64_dynmask,"
        "op_xrstor64_xsaveopt,op_xrstor64_xsaveopt64,op_xrstor64_xsaveopt64_sse,"
        "op_xrstor64_xsaveopt64_x87_avx,op_xrstor64_xsaveopt64_avx,op_xsaveopt64_dynmask,"
        "op_xrstor64_avx_xsaveopt64_avx")
    string(CONCAT cpu_xsave_state_original_exports ${cpu_xsave_state_original_exports})
    set(cpu_xsave_state_altered_exports
        "op_xrstor64_xsave64,op_xrstor_xsave64,op_xrstor64_disp,op_pipeline,"
        "op_xrstor64_xsavec64,op_xrstor64_xsavec64_sse,"
        "op_xrstor64_xsaveopt64,op_xrstor64_xsaveopt64_sse")
    string(CONCAT cpu_xsave_state_altered_exports ${cpu_xsave_state_altered_exports})
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_xsave_state_${variant} xsave_state)
        cpu_translation(cpu_xsave_state_${variant} cpu_xsave_aot "${cpu_xsave_state_${variant}_exports}")
    endforeach()
    target_compile_definitions(cpu_xsave_state_altered PRIVATE ALTER_OP)
    set(cpu_xsave_state_compacted_exports
        "op_xrstor64_sse_compacted_avx_xsaveopt64_avx,"
        "op_xsavec_xrstor_all,op_xsavec_xrstor_x87_avx,op_xsavec_xrstor_x87_sse,"
        "op_xsavec_xrstor_avx_only,op_xsavec_xrstor_sse_only,op_xrstor64_compacted_dynmask")
    string(CONCAT cpu_xsave_state_compacted_exports ${cpu_xsave_state_compacted_exports})
    set(cpu_xsave_state_compacted_cpp "${CMAKE_CURRENT_BINARY_DIR}/cpu_xsave_state_compacted_lifted.cpp")
    add_custom_command(OUTPUT "${cpu_xsave_state_compacted_cpp}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/cpu_xsave_aot.py"
            "$<TARGET_FILE:cpu_xsave_state_original>" "${cpu_xsave_state_compacted_exports}"
            "${cpu_xsave_state_compacted_cpp}" --compacted-xrstor
        DEPENDS cpu_xsave_state_original tools/cpu_xsave_aot.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_xsave_state_compacted_lifted SHARED "${cpu_xsave_state_compacted_cpp}")
    set(cpu_xsave_state_compacted_altered_cpp "${CMAKE_CURRENT_BINARY_DIR}/cpu_xsave_state_compacted_altered_lifted.cpp")
    add_custom_command(OUTPUT "${cpu_xsave_state_compacted_altered_cpp}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/cpu_xsave_aot.py"
            "$<TARGET_FILE:cpu_xsave_state_altered>" "op_xsavec_xrstor_all,op_xsavec_xrstor_avx_only"
            "${cpu_xsave_state_compacted_altered_cpp}" --compacted-xrstor
        DEPENDS cpu_xsave_state_altered tools/cpu_xsave_aot.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_xsave_state_compacted_altered_lifted SHARED "${cpu_xsave_state_compacted_altered_cpp}")
    set(cpu_xsave_state_canonical_cpp "${CMAKE_CURRENT_BINARY_DIR}/cpu_xsave_state_canonical_lifted.cpp")
    add_custom_command(OUTPUT "${cpu_xsave_state_canonical_cpp}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/cpu_xsave_aot.py"
            "$<TARGET_FILE:cpu_xsave_state_original>" "op_xsave_only,op_xsave64_only,op_xrstor64_avx_xsaveopt64_avx"
            "${cpu_xsave_state_canonical_cpp}" --entry-state canonical --compacted-xrstor
        DEPENDS cpu_xsave_state_original tools/cpu_xsave_aot.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_xsave_state_canonical_lifted SHARED "${cpu_xsave_state_canonical_cpp}")
    set(cpu_xsave_state_canonical_altered_cpp "${CMAKE_CURRENT_BINARY_DIR}/cpu_xsave_state_canonical_altered_lifted.cpp")
    add_custom_command(OUTPUT "${cpu_xsave_state_canonical_altered_cpp}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/cpu_xsave_aot.py"
            "$<TARGET_FILE:cpu_xsave_state_altered>" "op_xrstor64_mask_sse"
            "${cpu_xsave_state_canonical_altered_cpp}" --entry-state canonical
        DEPENDS cpu_xsave_state_altered tools/cpu_xsave_aot.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_xsave_state_canonical_altered_lifted SHARED "${cpu_xsave_state_canonical_altered_cpp}")
    add_test(NAME cpu_aot_xsave_state COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/xsave_state_check.py"
        "$<TARGET_FILE:cpu_xsave_state_original>"
        "$<TARGET_FILE:cpu_xsave_state_original_lifted>"
        "$<TARGET_FILE:cpu_xsave_state_altered_lifted>"
        "$<TARGET_FILE:cpu_xsave_state_canonical_lifted>"
        "$<TARGET_FILE:cpu_xsave_state_canonical_altered_lifted>"
        "$<TARGET_FILE:cpu_xsave_state_altered>"
        "$<TARGET_FILE:cpu_xsave_state_compacted_lifted>"
        "$<TARGET_FILE:cpu_xsave_state_compacted_altered_lifted>")
    set_tests_properties(cpu_aot_xsave_state PROPERTIES SKIP_RETURN_CODE 77)
    set(cpu_pcmpstr64_exports
        "pcmpestri64_ub_eqany_pos_lsb,pcmpestri64_sw_ranges_neg_msb,"
        "pcmpestri64_mem_sb_eqeach_mpos_msb,pcmpestri64_alias_xmm0,"
        "pcmpestrm64_ub_eqany_pos_bit,pcmpestrm64_sb_ranges_neg_exp,"
        "pcmpestrm64_mem_uw_eqord_mneg_exp,pcmpestrm64_xmm0_self,"
        "vpcmpestri64_ub_eqany_pos_lsb,vpcmpestri64_uw_eqord_mneg_lsb,"
        "vpcmpestri64_mem_sw_ranges_neg_msb,vpcmpestri64_alias_xmm0,"
        "vpcmpestrm64_ub_eqany_pos_bit,vpcmpestrm64_uw_eqeach_mpos_exp,"
        "vpcmpestrm64_mem_ub_eqany_exp,vpcmpestrm64_xmm0_self,"
        "vpcmpestrm64_discarded_read")
    string(CONCAT cpu_pcmpstr64_exports ${cpu_pcmpstr64_exports})
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_pcmpstr64_${variant} pcmpstr64)
        cpu_translation(cpu_pcmpstr64_${variant} cpu_pcmpstr64_aot "${cpu_pcmpstr64_exports}")
    endforeach()
    target_compile_definitions(cpu_pcmpstr64_altered PRIVATE ALTER_IMM)
    add_test(NAME cpu_aot_pcmpstr64 COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/pcmpstr64_check.py"
        "$<TARGET_FILE:cpu_pcmpstr64_original>"
        "$<TARGET_FILE:cpu_pcmpstr64_original_lifted>"
        "$<TARGET_FILE:cpu_pcmpstr64_altered_lifted>")
    set_tests_properties(cpu_aot_pcmpstr64 PROPERTIES
        SKIP_RETURN_CODE 77
        ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/cpu_aot_pcmpstr64.txt")
    set(cpu_sse4a_exports
        "op_extrq_imm_basic,op_extrq_imm_zero_len,op_extrq_imm_high_bit,"
        "op_extrq_imm_mid,op_extrq_imm_bits76,"
        "op_extrq_reg_basic,op_extrq_reg_self,op_extrq_reg_xmm4_xmm5,"
        "op_insertq_imm_basic,op_insertq_imm_zero_len,op_insertq_imm_high_bit,"
        "op_insertq_imm_self,"
        "op_insertq_reg_basic,op_insertq_reg_self,op_insertq_reg_xmm4_xmm5,"
        "op_movntsd_aligned,op_movntsd_disp8,op_movntsd_unaligned,"
        "op_movntss_aligned,op_movntss_disp12,op_movntss_unaligned,"
        "op_pipeline,op_discarded_read,op_sse4a_discarded_read")
    string(CONCAT cpu_sse4a_exports ${cpu_sse4a_exports})
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_sse4a_${variant} sse4a)
        cpu_translation(cpu_sse4a_${variant} cpu_sse4a_aot "${cpu_sse4a_exports}")
    endforeach()
    target_compile_definitions(cpu_sse4a_altered PRIVATE ALTER_IMM)
    add_test(NAME cpu_aot_sse4a COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/sse4a_check.py"
        "$<TARGET_FILE:cpu_sse4a_original>"
        "$<TARGET_FILE:cpu_sse4a_original_lifted>"
        "$<TARGET_FILE:cpu_sse4a_altered_lifted>")
    set_tests_properties(cpu_aot_sse4a PROPERTIES
        ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/cpu_aot_sse4a.txt")
    set(cpu_sse2_exports
        "op_cmppd_0,op_cmppd_1,op_cmppd_2,op_cmppd_3,op_cmppd_4,op_cmppd_5,op_cmppd_6,op_cmppd_7,"
        "op_cmppd_mem,"
        "op_cmpsd_0,op_cmpsd_1,op_cmpsd_2,op_cmpsd_3,op_cmpsd_4,op_cmpsd_5,op_cmpsd_6,op_cmpsd_7,"
        "op_cmpsd_mem,op_cmpsd_mem_plus1,"
        "op_maxsd_reg,op_maxsd_mem,op_minsd_reg,op_minsd_mem,op_sqrtsd_reg,op_sqrtsd_mem,"
        "op_cvtsd2si_r32_reg,op_cvtsd2si_r64_reg,op_cvtsd2si_r32_mem,op_cvtsd2si_r64_mem,"
        "op_cvtdq2pd_reg,op_cvtdq2pd_mem,op_cvtdq2ps_reg,op_cvtdq2ps_mem,"
        "op_cvtpd2dq_reg,op_cvtpd2dq_mem,op_cvtpd2ps_reg,op_cvtpd2ps_mem,op_cvtpi2pd_mem,"
        "op_cvtps2dq_reg,op_cvtps2dq_mem,op_cvtps2pd_reg,op_cvtps2pd_mem,"
        "op_cvttpd2dq_reg,op_cvttpd2dq_mem,op_cvttps2dq_reg,op_cvttps2dq_mem,"
        "op_movapd_ls,op_movapd_reg,op_movdqa_ls,op_movdqa_reg,"
        "op_movhpd_load,op_movhpd_store,op_movlpd_load,op_movlpd_store,"
        "op_movhpd_load_plus1,op_movlpd_load_plus1,"
        "op_movhpd_store_plus1,op_movlpd_store_plus1,op_maxsd_mem_plus1,op_cvtsd2si_mem_plus1,"
        "op_movntdq,op_movntpd,op_movnti32,op_movnti64,"
        "op_lfence,op_mfence,op_pipeline")
    string(CONCAT cpu_sse2_exports ${cpu_sse2_exports})
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_sse2_regs_${variant} sse2_regs)
        cpu_translation(cpu_sse2_regs_${variant} cpu_sse2_aot "${cpu_sse2_exports}")
    endforeach()
    target_compile_definitions(cpu_sse2_regs_altered PRIVATE ALTER_IMM)
    add_test(NAME cpu_aot_sse2_regs COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/sse2_regs_check.py"
        "$<TARGET_FILE:cpu_sse2_regs_original>"
        "$<TARGET_FILE:cpu_sse2_regs_original_lifted>"
        "$<TARGET_FILE:cpu_sse2_regs_altered_lifted>")
    set(cpu_mmx_sse2_exports
        "op_cvtpd2pi_reg,op_cvtpd2pi_mem,op_cvtpd2pi_mem_disp,"
        "op_cvttpd2pi_reg,op_cvttpd2pi_mem,"
        "op_movdq2q,op_movq2dq,op_emms_standalone,"
        "op_movq_reg_mem,op_movdqa_reg_ls,op_movdqu_reg_ls,op_mmx_sse2_bridge")
    string(CONCAT cpu_mmx_sse2_exports ${cpu_mmx_sse2_exports})
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_mmx_sse2_${variant} mmx_sse2)
        cpu_translation(cpu_mmx_sse2_${variant} cpu_mmx_sse2_aot "${cpu_mmx_sse2_exports}")
    endforeach()
    target_compile_definitions(cpu_mmx_sse2_altered PRIVATE ALTER_IMM)
    add_test(NAME cpu_aot_mmx_sse2 COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/mmx_sse2_check.py"
        "$<TARGET_FILE:cpu_mmx_sse2_original>"
        "$<TARGET_FILE:cpu_mmx_sse2_original_lifted>"
        "$<TARGET_FILE:cpu_mmx_sse2_altered_lifted>")
    set(cpu_sse_fp_exports
        "op_cmpps_0,op_cmpps_1,op_cmpps_2,op_cmpps_3,op_cmpps_4,op_cmpps_5,op_cmpps_6,op_cmpps_7,"
        "op_cmpps_mem,op_cmpps_mem_disp,op_cmpps_self,"
        "op_cmpss_0,op_cmpss_1,op_cmpss_2,op_cmpss_3,op_cmpss_4,op_cmpss_5,op_cmpss_6,op_cmpss_7,"
        "op_cmpss_mem,op_cmpss_mem_plus1,op_cmpss_self,"
        "op_movlps_load,op_movlps_store,op_movhps_load,op_movhps_store,"
        "op_movlps_load_plus1,op_movhps_load_plus1,op_movlps_store_plus1,op_movhps_store_plus1,"
        "op_movntps,op_movntps_disp16,op_movaps_reg,op_pipeline,op_discarded_read,"
        "op_addps,op_addps_mem,op_addps_mem_disp,op_addps_self,"
        "op_subps,op_subps_mem,op_subps_mem_disp,op_subps_self,"
        "op_mulps,op_mulps_mem,op_mulps_mem_disp,op_mulps_self,"
        "op_divps,op_divps_mem,op_divps_mem_disp,op_divps_self,"
        "op_minps,op_minps_mem,op_minps_mem_disp,op_minps_self,"
        "op_maxps,op_maxps_mem,op_maxps_mem_disp,op_maxps_self,"
        "op_sqrtps,op_sqrtps_mem,op_sqrtps_mem_disp,op_sqrtps_self,"
        "op_arith_pipeline")
    string(CONCAT cpu_sse_fp_exports ${cpu_sse_fp_exports})
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_sse_fp_${variant} sse_fp)
        cpu_translation(cpu_sse_fp_${variant} cpu_sse_fp_aot "${cpu_sse_fp_exports}")
    endforeach()
    target_compile_definitions(cpu_sse_fp_altered PRIVATE ALTER_IMM)
    cpu_asm_fixture(cpu_sse_fp_harness sse_fp_harness)
    add_test(NAME cpu_aot_sse_fp COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/sse_fp_check.py"
        "$<TARGET_FILE:cpu_sse_fp_original>"
        "$<TARGET_FILE:cpu_sse_fp_original_lifted>"
        "$<TARGET_FILE:cpu_sse_fp_altered_lifted>"
        "$<TARGET_FILE:cpu_sse_fp_harness>")
    set(cpu_sse_fp_pd_exports
        "op_addpd,op_addpd_mem,op_addpd_mem_disp,op_addpd_self,"
        "op_subpd,op_subpd_mem,op_subpd_mem_disp,op_subpd_self,"
        "op_mulpd,op_mulpd_mem,op_mulpd_mem_disp,op_mulpd_self,"
        "op_divpd,op_divpd_mem,op_divpd_mem_disp,op_divpd_self,"
        "op_minpd,op_minpd_mem,op_minpd_mem_disp,op_minpd_self,"
        "op_maxpd,op_maxpd_mem,op_maxpd_mem_disp,op_maxpd_self,"
        "op_sqrtpd,op_sqrtpd_mem,op_sqrtpd_mem_disp,op_sqrtpd_self,"
        "op_cmppd_0,op_cmppd_1,op_cmppd_2,op_cmppd_3,op_cmppd_4,op_cmppd_5,op_cmppd_6,op_cmppd_7,"
        "op_cmppd_mem,op_cmppd_mem_disp,op_cmppd_self,"
        "op_pd_pipeline")
    string(CONCAT cpu_sse_fp_pd_exports ${cpu_sse_fp_pd_exports})
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_sse_fp_pd_${variant} sse_fp_pd)
        cpu_translation(cpu_sse_fp_pd_${variant} cpu_sse_fp_aot "${cpu_sse_fp_pd_exports}")
    endforeach()
    target_compile_definitions(cpu_sse_fp_pd_altered PRIVATE ALTER_IMM)
    add_test(NAME cpu_aot_sse_fp_pd COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/sse_fp_pd_check.py"
        "$<TARGET_FILE:cpu_sse_fp_pd_original>"
        "$<TARGET_FILE:cpu_sse_fp_pd_original_lifted>"
        "$<TARGET_FILE:cpu_sse_fp_pd_altered_lifted>"
        "$<TARGET_FILE:cpu_sse_fp_harness>")
    set(cpu_avx_fp_move_exports
        "op_vmovq_xmm_xmm,op_vmovq_xmm_m64,op_vmovq_m64_xmm,op_vmovq_xmm_r64,op_vmovq_r64_xmm,"
        "op_vmaskmovdqu,op_pipeline,op_discarded_read,op_discarded_read64,"
        "op_vmovupd_xmm_load,op_vmovupd_ymm_load,op_vmovupd_xmm_store,op_vmovupd_ymm_store,"
        "op_vmovupd_xmm_reg,op_vmovupd_ymm_reg,op_vmovupd_xmm_regst,op_vmovupd_ymm_regst,"
        "op_vmovapd_xmm_load,op_vmovapd_ymm_load,op_vmovapd_xmm_store,op_vmovapd_ymm_store,"
        "op_vmovapd_xmm_reg,op_vmovapd_ymm_reg,op_vmovapd_xmm_regst,op_vmovapd_ymm_regst,"
        "op_pd_pipeline")
    string(CONCAT cpu_avx_fp_move_exports ${cpu_avx_fp_move_exports})
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_avx_fp_move_${variant} avx_fp_move)
        cpu_translation(cpu_avx_fp_move_${variant} cpu_avx_fp_move_aot "${cpu_avx_fp_move_exports}")
    endforeach()
    target_compile_definitions(cpu_avx_fp_move_altered PRIVATE ALTER_OP)
    add_test(NAME cpu_aot_avx_fp_move COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/avx_fp_move_check.py"
        "$<TARGET_FILE:cpu_avx_fp_move_original>"
        "$<TARGET_FILE:cpu_avx_fp_move_original_lifted>"
        "$<TARGET_FILE:cpu_avx_fp_move_altered_lifted>")
    set_tests_properties(cpu_aot_avx_fp_move PROPERTIES SKIP_RETURN_CODE 77)
    set(cpu_sse_mmx_exports
        "op_cvtpi2ps_reg,op_cvtpi2ps_mem,op_cvtpi2ps_mem_plus1,"
        "op_cvtps2pi_reg,op_cvtps2pi_mem,op_cvtps2pi_mem_plus1,"
        "op_cvttps2pi_reg,op_cvttps2pi_mem,"
        "op_pshufw_reg_e4,op_pshufw_reg_1b,op_pshufw_reg_00,"
        "op_pshufw_reg_ff,op_pshufw_mem_4e,op_pshufw_mem_39,"
        "op_movntq,op_movntq_disp,op_movq_gpr_bridge,op_discarded_read,op_pipeline,"
        "op_maskmovq,op_maskmovdqu,op_maskmov_pipeline")
    string(CONCAT cpu_sse_mmx_exports ${cpu_sse_mmx_exports})
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_sse_mmx_${variant} sse_mmx)
        cpu_translation(cpu_sse_mmx_${variant} cpu_sse_mmx_aot "${cpu_sse_mmx_exports}")
    endforeach()
    target_compile_definitions(cpu_sse_mmx_altered PRIVATE ALTER_IMM)
    add_test(NAME cpu_aot_sse_mmx COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/sse_mmx_check.py"
        "$<TARGET_FILE:cpu_sse_mmx_original>"
        "$<TARGET_FILE:cpu_sse_mmx_original_lifted>"
        "$<TARGET_FILE:cpu_sse_mmx_altered_lifted>")
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_integer_flags_${variant} integer_flags)
        set(cpu_integer_flags_${variant}_lifted_cpp "${CMAKE_CURRENT_BINARY_DIR}/cpu_integer_flags_${variant}_lifted.cpp")
        add_custom_command(OUTPUT "${cpu_integer_flags_${variant}_lifted_cpp}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/integer_flags_check.py"
                emit "$<TARGET_FILE:cpu_integer_flags_${variant}>" "${cpu_integer_flags_${variant}_lifted_cpp}"
            DEPENDS cpu_integer_flags_${variant} tests/cpu/integer_flags_check.py tools/ffx_version_query.py
                tools/x87_lower.py tools/cpu_leaf_aot.py tools/cpu_ranges.py VERBATIM)
        add_library(cpu_integer_flags_${variant}_lifted SHARED "${cpu_integer_flags_${variant}_lifted_cpp}")
    endforeach()
    target_compile_definitions(cpu_integer_flags_altered PRIVATE ALTER_IMM)
    set(cpu_integer_flags_controls_cpp "${CMAKE_CURRENT_BINARY_DIR}/cpu_integer_flags_controls.cpp")
    add_custom_command(OUTPUT "${cpu_integer_flags_controls_cpp}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/integer_flags_check.py"
            emit-controls "$<TARGET_FILE:cpu_integer_flags_original>" "${cpu_integer_flags_controls_cpp}"
        DEPENDS cpu_integer_flags_original tests/cpu/integer_flags_check.py tools/ffx_version_query.py
            tools/x87_lower.py tools/cpu_leaf_aot.py tools/cpu_ranges.py VERBATIM)
    add_library(cpu_integer_flags_controls SHARED "${cpu_integer_flags_controls_cpp}")
    add_test(NAME cpu_aot_integer_flags COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/integer_flags_check.py" check
        "$<TARGET_FILE:cpu_integer_flags_original>"
        "$<TARGET_FILE:cpu_integer_flags_original_lifted>"
        "$<TARGET_FILE:cpu_integer_flags_altered_lifted>"
        "$<TARGET_FILE:cpu_integer_flags_controls>")
    set_tests_properties(cpu_aot_integer_flags PROPERTIES SKIP_RETURN_CODE 77)
    set(cpu_sse_fxsave_exports
        "op_fxrstor_fxsave,op_fxrstor64_fxsave64,op_fxrstor_fxsave64,"
        "op_fxrstor64_fxsave,op_fxrstor64_disp,op_pipeline,op_discarded_read")
    string(CONCAT cpu_sse_fxsave_exports ${cpu_sse_fxsave_exports})
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_sse_fxsave_${variant} sse_fxsave)
        cpu_translation(cpu_sse_fxsave_${variant} cpu_sse_fxsave_aot "${cpu_sse_fxsave_exports}")
    endforeach()
    target_compile_definitions(cpu_sse_fxsave_altered PRIVATE ALTER_OP)
    set(cpu_sse_fxsave_canonical_cpp "${CMAKE_CURRENT_BINARY_DIR}/cpu_sse_fxsave_canonical_lifted.cpp")
    add_custom_command(OUTPUT "${cpu_sse_fxsave_canonical_cpp}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/cpu_sse_fxsave_aot.py"
            "$<TARGET_FILE:cpu_sse_fxsave_original>" "op_fxsave_only,op_fxsave64_only"
            "${cpu_sse_fxsave_canonical_cpp}" --entry-state canonical
        DEPENDS cpu_sse_fxsave_original tools/cpu_sse_fxsave_aot.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_sse_fxsave_canonical_lifted SHARED "${cpu_sse_fxsave_canonical_cpp}")
    add_test(NAME cpu_aot_sse_fxsave COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/sse_fxsave_check.py"
        "$<TARGET_FILE:cpu_sse_fxsave_original>"
        "$<TARGET_FILE:cpu_sse_fxsave_original_lifted>"
        "$<TARGET_FILE:cpu_sse_fxsave_altered_lifted>"
        "$<TARGET_FILE:cpu_sse_fxsave_canonical_lifted>")
    set_tests_properties(cpu_aot_sse_fxsave PROPERTIES SKIP_RETURN_CODE 77)
    # Import tail jumps: one caller per provider (named, altered return, ordinal-only), each translated twice:
    # tail_mix (three arguments) bound to mix3 with three, tail_choose (three) and tail_pass4 (four) to mix4 with four.
    set(tail_files)
    foreach(variant IN ITEMS named altered ordinal)
        set(provider cpu_tail_${variant}_provider)
        if(variant STREQUAL "ordinal")
            add_library(${provider} SHARED tests/cpu/tail_provider.cpp tests/cpu/tail_ordinal.def)
            set(mix3 SYMBOL "#3" LINK_NAME mix3)
            set(mix4 SYMBOL "#4" LINK_NAME mix4)
        else()
            add_library(${provider} SHARED tests/cpu/tail_provider.cpp tests/cpu/tail_provider.def)
            set(mix3 SYMBOL mix3)
            set(mix4 SYMBOL mix4)
        endif()
        cpu_asm_fixture(cpu_tail_${variant} tail_import)
        target_link_libraries(cpu_tail_${variant} PRIVATE ${provider})
        cpu_translation(cpu_tail_${variant} cpu_switch_aot tail_mix NAME cpu_tail_${variant}_mix3
            PROVIDER ${provider} ${mix3} IMPORT_ARGUMENTS 3 RETURN_BYTES 8)
        cpu_translation(cpu_tail_${variant} cpu_switch_aot tail_choose,tail_pass4:4 NAME cpu_tail_${variant}_mix4
            PROVIDER ${provider} ${mix4} IMPORT_ARGUMENTS 4 RETURN_BYTES 8)
        list(APPEND tail_files "$<TARGET_FILE:cpu_tail_${variant}>"
            "$<TARGET_FILE:cpu_tail_${variant}_mix3>" "${CMAKE_CURRENT_BINARY_DIR}/cpu_tail_${variant}_mix3.cpp"
            "$<TARGET_FILE:cpu_tail_${variant}_mix4>" "${CMAKE_CURRENT_BINARY_DIR}/cpu_tail_${variant}_mix4.cpp"
            "$<TARGET_FILE:${provider}>")
    endforeach()
    target_compile_definitions(cpu_tail_altered_provider PRIVATE ALTER_RETURN)
    add_test(NAME cpu_aot_import_tail_jump COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/tail_import_check.py" "$<TARGET_FILE:cpu_register_probe>" ${tail_files})
    # Ordinary import calls, linked against the named and the altered provider above. Each caller is translated
    # three times: call_keep,call_twice,outer_call,pass_result,read_then_call,call_then_read bound to mix3 with
    # three arguments and 8 guaranteed bytes, low_result to mix3 with 1, and thunk_call,thunk_tail,outer_tail to
    # mix4 with four arguments and 8.
    set(icall_files)
    foreach(variant IN ITEMS named altered)
        set(provider cpu_tail_${variant}_provider)
        cpu_asm_fixture(cpu_icall_${variant} import_calls)
        target_link_libraries(cpu_icall_${variant} PRIVATE ${provider})
        cpu_translation(cpu_icall_${variant} cpu_switch_aot
            call_keep,call_twice,outer_call,pass_result,read_then_call,call_then_read
            NAME cpu_icall_${variant}_mix3 PROVIDER ${provider} SYMBOL mix3 IMPORT_ARGUMENTS 3 RETURN_BYTES 8)
        cpu_translation(cpu_icall_${variant} cpu_switch_aot low_result
            NAME cpu_icall_${variant}_low PROVIDER ${provider} SYMBOL mix3 IMPORT_ARGUMENTS 3 RETURN_BYTES 1)
        cpu_translation(cpu_icall_${variant} cpu_switch_aot thunk_call,thunk_tail,outer_tail
            NAME cpu_icall_${variant}_mix4 PROVIDER ${provider} SYMBOL mix4 IMPORT_ARGUMENTS 4 RETURN_BYTES 8)
        list(APPEND icall_files "$<TARGET_FILE:cpu_icall_${variant}>"
            "$<TARGET_FILE:cpu_icall_${variant}_mix3>" "${CMAKE_CURRENT_BINARY_DIR}/cpu_icall_${variant}_mix3.cpp"
            "$<TARGET_FILE:cpu_icall_${variant}_low>" "${CMAKE_CURRENT_BINARY_DIR}/cpu_icall_${variant}_low.cpp"
            "$<TARGET_FILE:cpu_icall_${variant}_mix4>" "${CMAKE_CURRENT_BINARY_DIR}/cpu_icall_${variant}_mix4.cpp"
            "$<TARGET_FILE:${provider}>")
    endforeach()
    # The named caller's three translations built again with a separate cl /O2 command, as for cpu_aot_dead_reads.
    foreach(binding IN ITEMS mix3 low mix4)
        set(icall_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_icall_named_${binding}_o2.dll")
        add_custom_command(OUTPUT "${icall_o2}"
            COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /LD "${CMAKE_CURRENT_BINARY_DIR}/cpu_icall_named_${binding}.cpp"
                "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_icall_named_${binding}_o2.obj" "/Fe${icall_o2}"
                /link "$<TARGET_LINKER_FILE:cpu_tail_named_provider>"
            DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/cpu_icall_named_${binding}.cpp" cpu_tail_named_provider VERBATIM)
        add_custom_target(cpu_icall_named_${binding}_o2 ALL DEPENDS "${icall_o2}")
        list(APPEND icall_files "${icall_o2}")
    endforeach()
    add_test(NAME cpu_aot_import_calls COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/import_calls_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_guarded_call>" ${icall_files})
    add_library(cpu_stack_output_provider SHARED tests/cpu/stack_output_provider.cpp)
    cpu_asm_fixture(cpu_stack_output stack_output)
    target_link_libraries(cpu_stack_output PRIVATE cpu_stack_output_provider)
    cpu_translation(cpu_stack_output cpu_switch_aot from_frame NAME cpu_stack_output_status
        PROVIDER cpu_stack_output_provider SYMBOL write_result IMPORT_ARGUMENTS 2 RETURN_BYTES 4 STACK_OUTPUT 1 8)
    cpu_translation(cpu_stack_output cpu_switch_aot from_rsp NAME cpu_stack_output_void
        PROVIDER cpu_stack_output_provider SYMBOL write_constant IMPORT_ARGUMENTS 1 RETURN_BYTES 0 STACK_OUTPUT 1 8)
    add_test(NAME cpu_aot_stack_output COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/stack_output_check.py" "$<TARGET_FILE:cpu_stack_output>"
        "$<TARGET_FILE:cpu_stack_output_provider>" "$<TARGET_FILE:cpu_stack_output_status>"
        "$<TARGET_FILE:cpu_stack_output_void>")
    # Two ordinary imports from two providers in one translation: mix3 from the tail-jump providers and fold4 from
    # slot_provider.cpp. One caller per provider pair: both named, mix3 altered, fold4 altered, and mix3 imported by
    # ordinal #3 (bound with the link name mix3) next to the named fold4. Each is translated once with mix3 bound with
    # three arguments and 8 guaranteed bytes and fold4 with four arguments and 2.
    foreach(variant IN ITEMS named altered)
        add_library(cpu_slot_${variant}_provider SHARED tests/cpu/slot_provider.cpp tests/cpu/slot_provider.def)
    endforeach()
    target_compile_definitions(cpu_slot_altered_provider PRIVATE ALTER_RETURN)
    set(slot_files)
    foreach(pair IN ITEMS "named;named;named" "first_altered;altered;named" "second_altered;named;altered"
                          "ordinal;ordinal;named")
        list(GET pair 0 variant)
        list(GET pair 1 first)
        list(GET pair 2 second)
        if(first STREQUAL "ordinal")
            set(mix3 SYMBOL "#3" LINK_NAME mix3)
        else()
            set(mix3 SYMBOL mix3)
        endif()
        cpu_asm_fixture(cpu_slots_${variant} two_slots)
        target_link_libraries(cpu_slots_${variant} PRIVATE cpu_tail_${first}_provider cpu_slot_${second}_provider)
        cpu_translation(cpu_slots_${variant} cpu_switch_aot route2,call_both,call_second NAME cpu_slots_${variant}_lifted
            PROVIDER cpu_tail_${first}_provider ${mix3} IMPORT_ARGUMENTS 3 RETURN_BYTES 8
            SECOND_PROVIDER cpu_slot_${second}_provider SECOND_SYMBOL fold4 SECOND_IMPORT_ARGUMENTS 4
            SECOND_RETURN_BYTES 2)
        list(APPEND slot_files "$<TARGET_FILE:cpu_slots_${variant}>" "$<TARGET_FILE:cpu_slots_${variant}_lifted>"
            "${CMAKE_CURRENT_BINARY_DIR}/cpu_slots_${variant}_lifted.cpp" "$<TARGET_FILE:cpu_tail_${first}_provider>"
            "$<TARGET_FILE:cpu_slot_${second}_provider>")
    endforeach()
    add_test(NAME cpu_aot_two_slots COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/two_slots_check.py" "$<TARGET_FILE:cpu_register_probe>" ${slot_files})

    foreach(variant IN ITEMS original swapped)
        cpu_asm_fixture(cpu_float_${variant} scalar_float)
        cpu_translation(cpu_float_${variant} cpu_sse_aot
            "blend_ss=f:ffqf,blend_sd=d:ddqd,convert=q:fdq,from_int=d:qq,classify_ss=q:ff,classify_sd=q:dq,round_trip=f:f,zero_xorps=d:d,zero_pxor=d:d,zero_xorpd=d:d,probe_ss=q:qff,neg_jcc=q:q,setcc_mix=q:qqq,cmovcc_mix=q:qqq,min_ss=f:ffq,max_ss=f:ffq,min_sd=d:ddq,max_sd=d:ddq,root_ss=f:fq,root_sd=d:dq,root_const_ss=f:,root_const_sd=d:,round_cvt=q:fdq")
        add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/cpu_float_${variant}_lifted.cpp" APPEND
            DEPENDS tools/cpu_switch_aot.py)
    endforeach()
    target_compile_definitions(cpu_float_swapped PRIVATE SWAP_OPERANDS)
    add_test(NAME cpu_aot_scalar_float COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/scalar_float_check.py"
        "$<TARGET_FILE:cpu_float_original>" "$<TARGET_FILE:cpu_float_original_lifted>"
        "$<TARGET_FILE:cpu_float_swapped>" "$<TARGET_FILE:cpu_float_swapped_lifted>" "$<TARGET_FILE:cpu_mxcsr_harness>")
    set(cpu_fma3_exports
        "op_vfmadd132ps_xmm,op_vfmadd132ps_ymm,op_vfmadd132pd_xmm,op_vfmadd132pd_ymm,op_vfmadd132ss,op_vfmadd132sd,"
        "op_vfmadd213ps_xmm,op_vfmadd213ps_ymm,op_vfmadd213pd_xmm,op_vfmadd213pd_ymm,op_vfmadd213ss,op_vfmadd213sd,"
        "op_vfmadd231ps_xmm,op_vfmadd231ps_ymm,op_vfmadd231pd_xmm,op_vfmadd231pd_ymm,op_vfmadd231ss,op_vfmadd231sd,"
        "op_vfmsub132ps_xmm,op_vfmsub132ps_ymm,op_vfmsub132pd_xmm,op_vfmsub132pd_ymm,op_vfmsub132ss,op_vfmsub132sd,"
        "op_vfmsub213ps_xmm,op_vfmsub213ps_ymm,op_vfmsub213pd_xmm,op_vfmsub213pd_ymm,op_vfmsub213ss,op_vfmsub213sd,"
        "op_vfmsub231ps_xmm,op_vfmsub231ps_ymm,op_vfmsub231pd_xmm,op_vfmsub231pd_ymm,op_vfmsub231ss,op_vfmsub231sd,"
        "op_vfnmadd132ps_xmm,op_vfnmadd132ps_ymm,op_vfnmadd132pd_xmm,op_vfnmadd132pd_ymm,op_vfnmadd132ss,op_vfnmadd132sd,"
        "op_vfnmadd213ps_xmm,op_vfnmadd213ps_ymm,op_vfnmadd213pd_xmm,op_vfnmadd213pd_ymm,op_vfnmadd213ss,op_vfnmadd213sd,"
        "op_vfnmadd231ps_xmm,op_vfnmadd231ps_ymm,op_vfnmadd231pd_xmm,op_vfnmadd231pd_ymm,op_vfnmadd231ss,op_vfnmadd231sd,"
        "op_vfnmsub132ps_xmm,op_vfnmsub132ps_ymm,op_vfnmsub132pd_xmm,op_vfnmsub132pd_ymm,op_vfnmsub132ss,op_vfnmsub132sd,"
        "op_vfnmsub213ps_xmm,op_vfnmsub213ps_ymm,op_vfnmsub213pd_xmm,op_vfnmsub213pd_ymm,op_vfnmsub213ss,op_vfnmsub213sd,"
        "op_vfnmsub231ps_xmm,op_vfnmsub231ps_ymm,op_vfnmsub231pd_xmm,op_vfnmsub231pd_ymm,op_vfnmsub231ss,op_vfnmsub231sd,"
        "op_vfmaddsub132ps_xmm,op_vfmaddsub132ps_ymm,op_vfmaddsub132pd_xmm,op_vfmaddsub132pd_ymm,"
        "op_vfmaddsub213ps_xmm,op_vfmaddsub213ps_ymm,op_vfmaddsub213pd_xmm,op_vfmaddsub213pd_ymm,"
        "op_vfmaddsub231ps_xmm,op_vfmaddsub231ps_ymm,op_vfmaddsub231pd_xmm,op_vfmaddsub231pd_ymm,"
        "op_vfmsubadd132ps_xmm,op_vfmsubadd132ps_ymm,op_vfmsubadd132pd_xmm,op_vfmsubadd132pd_ymm,"
        "op_vfmsubadd213ps_xmm,op_vfmsubadd213ps_ymm,op_vfmsubadd213pd_xmm,op_vfmsubadd213pd_ymm,"
        "op_vfmsubadd231ps_xmm,op_vfmsubadd231ps_ymm,op_vfmsubadd231pd_xmm,op_vfmsubadd231pd_ymm,"
        "op_mem_vfmadd132ss,op_mem_vfmsub213sd,op_mem_vfnmadd231ps_xmm,op_mem_vfnmsub132ps_ymm,"
        "op_mem_vfmaddsub213pd_xmm,op_mem_vfmsubadd231pd_ymm")
    string(CONCAT cpu_fma3_exports ${cpu_fma3_exports})
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_fma3_leaf_${variant} fma3_leaf)
        cpu_translation(cpu_fma3_leaf_${variant} cpu_avx2_aot "${cpu_fma3_exports}")
    endforeach()
    target_compile_definitions(cpu_fma3_leaf_altered PRIVATE ALTER_FMA)
    add_test(NAME cpu_aot_fma3 COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/fma3_leaf_check.py"
        "$<TARGET_FILE:cpu_fma3_leaf_original>"
        "$<TARGET_FILE:cpu_fma3_leaf_original_lifted>"
        "$<TARGET_FILE:cpu_fma3_leaf_altered_lifted>"
        "$<TARGET_FILE:cpu_fma_packed_lifted>")
    set_tests_properties(cpu_aot_fma3 PROPERTIES SKIP_RETURN_CODE 77)
    cpu_asm_fixture(cpu_chained chained)
    cpu_translation(cpu_chained cpu_chained_aot choose_value)
    cpu_asm_fixture(cpu_chained_jne chained)
    target_compile_definitions(cpu_chained_jne PRIVATE USE_JNE)
    cpu_translation(cpu_chained_jne cpu_chained_aot choose_value)
    cpu_asm_fixture(cpu_chained_zero chained)
    target_compile_definitions(cpu_chained_zero PRIVATE USE_XOR_ZERO)
    cpu_translation(cpu_chained_zero cpu_chained_aot choose_value)
    cpu_asm_fixture(cpu_chained_imm chained)
    target_compile_definitions(cpu_chained_imm PRIVATE USE_IMM_REG)
    cpu_translation(cpu_chained_imm cpu_chained_aot choose_value)
    cpu_asm_fixture(cpu_chained_frame chained)
    target_compile_definitions(cpu_chained_frame PRIVATE USE_LARGE_FRAME)
    cpu_translation(cpu_chained_frame cpu_chained_aot choose_value)
    # One translation per export (the translator lifts one entry), each also built with /O2, where the
    # compiler removes a read that the generated C++ lets it remove.
    cpu_asm_fixture(cpu_chained_dead_read chained_dead_read)
    set(chained_dead_read_files)
    foreach(entry IN ITEMS read_dead disp_dead pair_dead read_skip read_live)
        set(name cpu_chained_dead_read_${entry})
        cpu_translation(cpu_chained_dead_read cpu_chained_aot ${entry} NAME ${name})
        set(optimized "${CMAKE_CURRENT_BINARY_DIR}/${name}_o2.dll")
        add_custom_command(OUTPUT "${optimized}"
            COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /LD "${CMAKE_CURRENT_BINARY_DIR}/${name}.cpp"
                "/Fo${CMAKE_CURRENT_BINARY_DIR}/${name}_o2.obj" "/Fe${optimized}"
            DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/${name}.cpp" VERBATIM)
        add_custom_target(${name}_o2 ALL DEPENDS "${optimized}")
        list(APPEND chained_dead_read_files "${entry}=$<TARGET_FILE:${name}>" "${entry}=${optimized}")
    endforeach()
    cpu_asm_fixture(cpu_callback callback)
    cpu_asm_fixture(cpu_version_query version_query)
    cpu_asm_fixture(cpu_heap_layout heap_layout)
    # Attribute words at xmem call sites, read from a source-built image (tools/xmem_attribute_scan.py).
    cpu_asm_fixture(cpu_xmem_attributes xmem_attributes)
    target_link_libraries(cpu_xmem_attributes PRIVATE xmem_shim)
    cpu_asm_fixture(cpu_xmem_delay xmem_delay)
    target_link_libraries(cpu_xmem_delay PRIVATE xmem_shim)
    target_link_options(cpu_xmem_delay PRIVATE /DELAYLOAD:xmem.dll)
    add_test(NAME xmem_attribute_scan COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/xmem_attribute_scan_check.py" "$<TARGET_FILE:cpu_xmem_attributes>"
        "$<TARGET_FILE:cpu_xmem_delay>")

    # VEX.128 16-bit packed-integer leaf functions, an arithmetic-shift control, and their C++.
    foreach(variant IN ITEMS original altered)
        cpu_asm_fixture(cpu_vex_words_${variant} vex_words)
        set(vex_lifted "${CMAKE_CURRENT_BINARY_DIR}/cpu_vex_words_${variant}_lifted.cpp")
        add_custom_command(OUTPUT "${vex_lifted}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/cpu_vex_aot.py"
                "$<TARGET_FILE:cpu_vex_words_${variant}>" "${vex_lifted}" widen_mix word_ops shift_edges
                store_reload aligned_ops byte_pack_reload sat_words mul_minmax_words cmp_words
                sraw_count_reg sraw_count_mem sllw_count_reg sllw_count_mem srlw_count_reg srlw_count_mem
            DEPENDS cpu_vex_words_${variant} tools/cpu_vex_aot.py tools/cpu_leaf_aot.py tools/cpu_ranges.py
            VERBATIM)
        add_library(cpu_vex_words_${variant}_lifted SHARED "${vex_lifted}")
    endforeach()
    target_compile_definitions(cpu_vex_words_altered PRIVATE LOGICAL_SHIFT)
    add_test(NAME cpu_aot_vex_words COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/vex_words_check.py"
        "$<TARGET_FILE:cpu_vex_words_original>" "$<TARGET_FILE:cpu_vex_words_original_lifted>"
        "$<TARGET_FILE:cpu_vex_words_altered>" "$<TARGET_FILE:cpu_vex_words_altered_lifted>")
    set_tests_properties(cpu_aot_vex_words PROPERTIES SKIP_RETURN_CODE 77)
    cpu_asm_fixture(cpu_bink_heap bink_heap)
    add_library(cpu_bink_heap_provider SHARED tests/cpu/bink_heap_provider.cpp src/era_heap.cpp)
    target_include_directories(cpu_bink_heap_provider PRIVATE src)
    target_link_libraries(cpu_bink_heap PRIVATE cpu_bink_heap_provider)
    function(bink_heap_translation target image dependency)
        set(cpp "${CMAKE_CURRENT_BINARY_DIR}/${target}.cpp")
        add_custom_command(OUTPUT "${cpp}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/bink_heap_aot.py"
                "${image}" "${cpp}" ${ARGN}
            DEPENDS ${dependency} tools/bink_heap_aot.py tools/ffx_version_query.py
                tools/cpu_leaf_aot.py tools/cpu_ranges.py VERBATIM)
        add_library(${target} SHARED "${cpp}")
        target_link_libraries(${target} PRIVATE cpu_bink_heap_provider)
    endfunction()
    bink_heap_translation(cpu_bink_heap_lifted "$<TARGET_FILE:cpu_bink_heap>" cpu_bink_heap --group-entry heap_group)
    add_test(NAME cpu_aot_bink_heap COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/bink_heap_check.py" "$<TARGET_FILE:cpu_bink_heap_provider>"
        "$<TARGET_FILE:cpu_bink_heap>" "$<TARGET_FILE:cpu_bink_heap_lifted>")
    add_test(NAME cpu_aot_bink_group COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/bink_group_check.py" "$<TARGET_FILE:cpu_bink_heap_provider>"
        "$<TARGET_FILE:cpu_bink_heap>" "$<TARGET_FILE:cpu_bink_heap_lifted>")
    set(BINK_GROUP_ENTRY "" CACHE STRING "Optional grouped allocation caller RVA in the local Bink image")
    set(BINK_HEAP_IMAGE "" CACHE FILEPATH "Local PE containing the Bink allocation and release callers")
    set(BINK_HEAP_ENTRIES "" CACHE STRING "Two semicolon-separated entry RVAs (allocation;release)")
    if(BINK_HEAP_IMAGE)
        list(LENGTH BINK_HEAP_ENTRIES entry_count)
        if(NOT entry_count EQUAL 2)
            message(FATAL_ERROR "Provide both BINK_HEAP_ENTRIES for the selected local image")
        endif()
        set(group_args)
        if(BINK_GROUP_ENTRY)
            set(group_args --group-entry "${BINK_GROUP_ENTRY}")
            add_test(NAME cpu_aot_bink_package_group COMMAND "${INTAKE_PYTHON}" -B
                "${CMAKE_SOURCE_DIR}/tests/cpu/bink_group_check.py" "$<TARGET_FILE:cpu_bink_heap_provider>"
                "$<TARGET_FILE:cpu_bink_package_lifted>")
        endif()
        bink_heap_translation(cpu_bink_package_lifted "${BINK_HEAP_IMAGE}" "${BINK_HEAP_IMAGE}"
            --entries ${BINK_HEAP_ENTRIES} ${group_args})
        add_test(NAME cpu_aot_bink_package_heap COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/cpu/bink_heap_check.py" "$<TARGET_FILE:cpu_bink_heap_provider>"
            "$<TARGET_FILE:cpu_bink_package_lifted>")
    endif()
    cpu_asm_fixture(cpu_working_set_delta working_set_delta)
    function(working_set_delta_translation target image dependency)
        set(cpp "${CMAKE_CURRENT_BINARY_DIR}/${target}.cpp")
        add_custom_command(OUTPUT "${cpp}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/ffx_working_set_delta.py"
                "${image}" "${cpp}" ${ARGN}
            DEPENDS ${dependency} tools/ffx_working_set_delta.py tools/ffx_version_query.py
                tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(${target} SHARED "${cpp}")
    endfunction()
    working_set_delta_translation(cpu_working_set_delta_lifted "$<TARGET_FILE:cpu_working_set_delta>" cpu_working_set_delta)
    add_test(NAME cpu_working_set_delta COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/working_set_delta_check.py" "$<TARGET_FILE:cpu_working_set_delta>"
        "$<TARGET_FILE:cpu_working_set_delta_lifted>")
    if(FFX_UPSCALER_IMAGE)
        working_set_delta_translation(ffx_package_working_set_delta "${FFX_UPSCALER_IMAGE}" "${FFX_UPSCALER_IMAGE}" --package)
        add_test(NAME ffx_package_working_set_delta COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/cpu/working_set_delta_check.py" "$<TARGET_FILE:ffx_package_working_set_delta>")
        # The same translated updates fed with desktop xmem.dll samples around a real D3D12 heap.
        add_executable(ffx_package_working_set_xmem tests/cpu/working_set_xmem_check.cpp)
        target_link_libraries(ffx_package_working_set_xmem PRIVATE ffx_package_working_set_delta xmem_shim d3d12 dxgi)
        target_compile_features(ffx_package_working_set_xmem PRIVATE cxx_std_17)
        foreach(adapter IN ITEMS warp hardware)
            add_test(NAME ffx_package_working_set_xmem_${adapter} COMMAND ffx_package_working_set_xmem ${adapter})
            set_tests_properties(ffx_package_working_set_xmem_${adapter} PROPERTIES TIMEOUT 30 SKIP_RETURN_CODE 77)
        endforeach()
    endif()
    cpu_asm_fixture(cpu_bink_descriptors bink_descriptors)
    function(bink_descriptors_translation target image dependency)
        set(cpp "${CMAKE_CURRENT_BINARY_DIR}/${target}.cpp")
        add_custom_command(OUTPUT "${cpp}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/bink_descriptors_aot.py"
                "${image}" "${cpp}" --frame-planes ${ARGN}
            DEPENDS ${dependency} tools/bink_descriptors_aot.py tools/bink_heap_aot.py
                tools/ffx_version_query.py tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(${target} SHARED "${cpp}")
    endfunction()
    bink_descriptors_translation(cpu_bink_descriptors_lifted "$<TARGET_FILE:cpu_bink_descriptors>" cpu_bink_descriptors)
    add_test(NAME cpu_bink_descriptors COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/bink_descriptors_check.py" --native "$<TARGET_FILE:cpu_bink_descriptors>"
        "$<TARGET_FILE:cpu_bink_descriptors_lifted>")
    add_test(NAME cpu_bink_frame_planes COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/bink_frame_planes_check.py" "$<TARGET_FILE:cpu_bink_descriptors>"
        "$<TARGET_FILE:cpu_bink_descriptors_lifted>")
    set(BINK_DESCRIPTOR_IMAGE "" CACHE FILEPATH "Local Bink DLL for the grouped descriptor producer experiment")
    if(BINK_DESCRIPTOR_IMAGE)
        bink_descriptors_translation(cpu_bink_package_descriptors "${BINK_DESCRIPTOR_IMAGE}" "${BINK_DESCRIPTOR_IMAGE}" --package)
        add_test(NAME cpu_bink_package_frame_planes COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/cpu/bink_frame_planes_check.py" "$<TARGET_FILE:cpu_bink_package_descriptors>")
        add_test(NAME cpu_bink_package_descriptors COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/cpu/bink_descriptors_check.py" "$<TARGET_FILE:cpu_bink_package_descriptors>")
    endif()
    function(bink_frame_integration name driver descriptors heap)
        # Link identical translated objects to the shared DLL and to the recording
        # provider so a failing allocation can be injected at the import boundary.
        add_library(${name}_objects OBJECT "${driver}" "${CMAKE_CURRENT_BINARY_DIR}/${descriptors}.cpp"
            "${CMAKE_CURRENT_BINARY_DIR}/${heap}.cpp")
        add_library(${name} SHARED $<TARGET_OBJECTS:${name}_objects>)
        target_link_libraries(${name} PRIVATE xmem_shim)
        add_library(${name}_observed SHARED $<TARGET_OBJECTS:${name}_objects>)
        target_link_libraries(${name}_observed PRIVATE cpu_bink_heap_provider)
        add_test(NAME ${name} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/cpu/bink_frame_heap_check.py" "$<TARGET_FILE:${name}>"
            "$<TARGET_FILE:${name}_observed>" "$<TARGET_FILE:xmem_shim>" "$<TARGET_FILE:cpu_bink_heap_provider>")
    endfunction()
    bink_frame_integration(cpu_bink_frame_to_heap tests/cpu/bink_frame_control.cpp
        cpu_bink_descriptors_lifted cpu_bink_heap_lifted)
    if(BINK_DESCRIPTOR_IMAGE AND BINK_DESCRIPTOR_IMAGE STREQUAL BINK_HEAP_IMAGE AND BINK_GROUP_ENTRY)
        set(frame_driver "${CMAKE_CURRENT_BINARY_DIR}/bink_frame_driver.cpp")
        add_custom_command(OUTPUT "${frame_driver}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/bink_frame_aot.py"
                "${BINK_DESCRIPTOR_IMAGE}" "${frame_driver}"
            DEPENDS "${BINK_DESCRIPTOR_IMAGE}" tools/bink_frame_aot.py tools/bink_descriptors_aot.py
                tools/bink_heap_aot.py tools/ffx_version_query.py tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        bink_frame_integration(cpu_bink_package_frame_to_heap "${frame_driver}"
            cpu_bink_package_descriptors cpu_bink_package_lifted)
    endif()
    set(heap_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_heap_layout_lifted.cpp")
    add_custom_command(OUTPUT "${heap_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/heap_layout.py"
            emit "$<TARGET_FILE:cpu_heap_layout>" "${heap_fixture}"
        DEPENDS cpu_heap_layout tests/cpu/heap_layout.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_heap_layout_lifted SHARED "${heap_fixture}")
    add_test(NAME cpu_aot_heap_layout COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/heap_layout.py"
        check "$<TARGET_FILE:cpu_heap_layout>" "$<TARGET_FILE:cpu_heap_layout_lifted>")
    cpu_asm_fixture(cpu_rcx_branches rcx_branches)
    # The lifted DLL and three mutated controls that the check requires to mismatch.
    foreach(variant IN ITEMS lifted noecx nozf jrcxz)
        set(rcx_branches_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_rcx_branches_${variant}.cpp")
        add_custom_command(OUTPUT "${rcx_branches_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/rcx_branches.py"
                emit "$<TARGET_FILE:cpu_rcx_branches>" "${rcx_branches_fixture}" ${variant}
            DEPENDS cpu_rcx_branches tests/cpu/rcx_branches.py tools/ffx_version_query.py
                tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_rcx_branches_${variant} SHARED "${rcx_branches_fixture}")
    endforeach()
    add_test(NAME cpu_aot_rcx_branches COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/rcx_branches.py"
        check "$<TARGET_FILE:cpu_rcx_branches>" "$<TARGET_FILE:cpu_rcx_branches_lifted>"
        "$<TARGET_FILE:cpu_rcx_branches_noecx>" "$<TARGET_FILE:cpu_rcx_branches_nozf>"
        "$<TARGET_FILE:cpu_rcx_branches_jrcxz>")
    cpu_asm_fixture(cpu_shift_count shift_count)
    set(shift_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_shift_count_lifted.cpp")
    add_custom_command(OUTPUT "${shift_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/shift_count.py"
            emit "$<TARGET_FILE:cpu_shift_count>" "${shift_fixture}"
        DEPENDS cpu_shift_count tests/cpu/shift_count.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_shift_count_lifted SHARED "${shift_fixture}")
    add_test(NAME cpu_aot_shift_count COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/shift_count.py"
        check "$<TARGET_FILE:cpu_shift_count>" "$<TARGET_FILE:cpu_shift_count_lifted>")
    cpu_asm_fixture(cpu_carry_rotates carry_rotates)
    # The lifted DLL and three mutated controls that the check requires to mismatch.
    foreach(variant IN ITEMS lifted nomask nocarry stc)
        set(carry_rotates_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_carry_rotates_${variant}.cpp")
        add_custom_command(OUTPUT "${carry_rotates_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/carry_rotates.py"
                emit "$<TARGET_FILE:cpu_carry_rotates>" "${carry_rotates_fixture}" ${variant}
            DEPENDS cpu_carry_rotates tests/cpu/carry_rotates.py tools/ffx_version_query.py
                tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_carry_rotates_${variant} SHARED "${carry_rotates_fixture}")
    endforeach()
    add_test(NAME cpu_aot_carry_rotates COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/carry_rotates.py"
        check "$<TARGET_FILE:cpu_carry_rotates>" "$<TARGET_FILE:cpu_carry_rotates_lifted>"
        "$<TARGET_FILE:cpu_carry_rotates_nomask>" "$<TARGET_FILE:cpu_carry_rotates_nocarry>"
        "$<TARGET_FILE:cpu_carry_rotates_stc>")
    set(query_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_version_query_lifted.cpp")
    add_custom_command(OUTPUT "${query_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/version_query_fixture.py"
            emit "$<TARGET_FILE:cpu_version_query>" "${query_fixture}"
        DEPENDS cpu_version_query tests/cpu/version_query_fixture.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py tests/cpu/version_query_check.py VERBATIM)
    add_library(cpu_version_query_lifted SHARED "${query_fixture}")
    add_test(NAME cpu_aot_version_query COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/version_query_fixture.py" check "$<TARGET_FILE:cpu_version_query>"
        "$<TARGET_FILE:cpu_version_query_lifted>")
    add_test(NAME cpu_lower_prefixes COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/lower_prefixes_check.py")
    cpu_asm_fixture(cpu_xchg xchg)
    # The lifted DLL and two mutated controls that the check requires to mismatch.
    foreach(variant IN ITEMS lifted plain oneway)
        set(xchg_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_xchg_${variant}.cpp")
        add_custom_command(OUTPUT "${xchg_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/xchg.py"
                emit "$<TARGET_FILE:cpu_xchg>" "${xchg_fixture}" ${variant}
            DEPENDS cpu_xchg tests/cpu/xchg.py tools/ffx_version_query.py
                tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_xchg_${variant} SHARED "${xchg_fixture}")
    endforeach()
    add_test(NAME cpu_aot_xchg COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/xchg.py"
        check "$<TARGET_FILE:cpu_xchg>" "$<TARGET_FILE:cpu_xchg_lifted>"
        "$<TARGET_FILE:cpu_xchg_plain>" "$<TARGET_FILE:cpu_xchg_oneway>")
    cpu_asm_fixture(cpu_address_size address_size)
    set(address_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_address_size_lifted.cpp")
    add_custom_command(OUTPUT "${address_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/address_size.py"
            emit "$<TARGET_FILE:cpu_address_size>" "${address_fixture}"
        DEPENDS cpu_address_size tests/cpu/address_size.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_address_size_lifted SHARED "${address_fixture}")
    add_test(NAME cpu_aot_address_size COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/address_size.py"
        check "$<TARGET_FILE:cpu_address_size>" "$<TARGET_FILE:cpu_address_size_lifted>")
    cpu_asm_fixture(cpu_movbe movbe)
    # The lifted DLL and three mutated controls that the check requires to mismatch.
    foreach(variant IN ITEMS lifted noswap clear16 flags)
        set(movbe_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_movbe_${variant}.cpp")
        add_custom_command(OUTPUT "${movbe_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/movbe.py"
                emit "$<TARGET_FILE:cpu_movbe>" "${movbe_fixture}" ${variant}
            DEPENDS cpu_movbe tests/cpu/movbe.py tools/ffx_version_query.py
                tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_movbe_${variant} SHARED "${movbe_fixture}")
    endforeach()
    add_test(NAME cpu_aot_movbe COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/movbe.py"
        check "$<TARGET_FILE:cpu_movbe>" "$<TARGET_FILE:cpu_movbe_lifted>"
        "$<TARGET_FILE:cpu_movbe_noswap>" "$<TARGET_FILE:cpu_movbe_clear16>" "$<TARGET_FILE:cpu_movbe_flags>")
    set_tests_properties(cpu_aot_movbe PROPERTIES SKIP_RETURN_CODE 77)
    cpu_asm_fixture(cpu_bmi1 bmi1)
    set(bmi1_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_bmi1_lifted.cpp")
    add_custom_command(OUTPUT "${bmi1_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/bmi1.py"
            emit "$<TARGET_FILE:cpu_bmi1>" "${bmi1_fixture}"
        DEPENDS cpu_bmi1 tests/cpu/bmi1.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_bmi1_lifted SHARED "${bmi1_fixture}")
    add_test(NAME cpu_aot_bmi1 COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/bmi1.py"
        check "$<TARGET_FILE:cpu_bmi1>" "$<TARGET_FILE:cpu_bmi1_lifted>")
    set_tests_properties(cpu_aot_bmi1 PROPERTIES SKIP_RETURN_CODE 77)
    cpu_asm_fixture(cpu_bit_ops bit_ops)
    set(bit_ops_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_bit_ops_lifted.cpp")
    add_custom_command(OUTPUT "${bit_ops_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/bit_ops.py"
            emit "$<TARGET_FILE:cpu_bit_ops>" "${bit_ops_fixture}"
        DEPENDS cpu_bit_ops tests/cpu/bit_ops.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_bit_ops_lifted SHARED "${bit_ops_fixture}")
    add_test(NAME cpu_aot_bit_ops COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/bit_ops.py"
        check "$<TARGET_FILE:cpu_bit_ops>" "$<TARGET_FILE:cpu_bit_ops_lifted>")
    set_tests_properties(cpu_aot_bit_ops PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_rotates_shifts rotates_shifts)
    set(rotates_shifts_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_rotates_shifts_lifted.cpp")
    add_custom_command(OUTPUT "${rotates_shifts_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/rotates_shifts.py"
            emit "$<TARGET_FILE:cpu_rotates_shifts>" "${rotates_shifts_fixture}"
        DEPENDS cpu_rotates_shifts tests/cpu/rotates_shifts.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_rotates_shifts_lifted SHARED "${rotates_shifts_fixture}")
    add_test(NAME cpu_aot_rotates_shifts COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/rotates_shifts.py"
        check "$<TARGET_FILE:cpu_rotates_shifts>" "$<TARGET_FILE:cpu_rotates_shifts_lifted>")
    cpu_asm_fixture(cpu_string_ops string_ops)
    # The lifted DLL and three mutated controls that the check requires to mismatch.
    foreach(variant IN ITEMS lifted memmove swapped forward)
        set(string_ops_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_string_ops_${variant}.cpp")
        add_custom_command(OUTPUT "${string_ops_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/string_ops.py"
                emit "$<TARGET_FILE:cpu_string_ops>" "${string_ops_fixture}" ${variant}
            DEPENDS cpu_string_ops tests/cpu/string_ops.py tools/ffx_version_query.py
                tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_string_ops_${variant} SHARED "${string_ops_fixture}")
    endforeach()
    add_test(NAME cpu_aot_string_ops COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/string_ops.py"
        check "$<TARGET_FILE:cpu_string_ops>" "$<TARGET_FILE:cpu_string_ops_lifted>"
        "$<TARGET_FILE:cpu_string_ops_memmove>" "$<TARGET_FILE:cpu_string_ops_swapped>"
        "$<TARGET_FILE:cpu_string_ops_forward>")
    cpu_asm_fixture(cpu_sign_ext_adc sign_ext_adc)
    set(sign_ext_adc_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_sign_ext_adc_lifted.cpp")
    add_custom_command(OUTPUT "${sign_ext_adc_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sign_ext_adc.py"
            emit "$<TARGET_FILE:cpu_sign_ext_adc>" "${sign_ext_adc_fixture}"
        DEPENDS cpu_sign_ext_adc tests/cpu/sign_ext_adc.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_sign_ext_adc_lifted SHARED "${sign_ext_adc_fixture}")
    add_test(NAME cpu_aot_sign_ext_adc COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sign_ext_adc.py"
        check "$<TARGET_FILE:cpu_sign_ext_adc>" "$<TARGET_FILE:cpu_sign_ext_adc_lifted>")

    cpu_asm_fixture(cpu_fences_hints fences_hints)
    set(fences_hints_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_fences_hints_lifted.cpp")
    add_custom_command(OUTPUT "${fences_hints_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/fences_hints.py"
            emit "$<TARGET_FILE:cpu_fences_hints>" "${fences_hints_fixture}"
        DEPENDS cpu_fences_hints tests/cpu/fences_hints.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_fences_hints_lifted SHARED "${fences_hints_fixture}")
    add_test(NAME cpu_aot_fences_hints COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/fences_hints.py"
        check "$<TARGET_FILE:cpu_fences_hints>" "$<TARGET_FILE:cpu_fences_hints_lifted>" baseline)
    foreach(capability IN ITEMS prefetch_amd clflush clflushopt clwb)
        add_test(NAME cpu_aot_fences_hints_${capability}
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/fences_hints.py"
            check "$<TARGET_FILE:cpu_fences_hints>" "$<TARGET_FILE:cpu_fences_hints_lifted>" "${capability}")
        set_tests_properties(cpu_aot_fences_hints_${capability} PROPERTIES SKIP_RETURN_CODE 77)
    endforeach()

    cpu_asm_fixture(cpu_bmi2 bmi2)
    set(bmi2_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_bmi2_lifted.cpp")
    add_custom_command(OUTPUT "${bmi2_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/bmi2.py"
            emit "$<TARGET_FILE:cpu_bmi2>" "${bmi2_fixture}"
        DEPENDS cpu_bmi2 tests/cpu/bmi2.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_bmi2_lifted SHARED "${bmi2_fixture}")
    add_test(NAME cpu_aot_bmi2 COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/bmi2.py"
        check "$<TARGET_FILE:cpu_bmi2>" "$<TARGET_FILE:cpu_bmi2_lifted>")
    set_tests_properties(cpu_aot_bmi2 PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_sse2_packed_cvt sse2_packed_cvt)
    target_sources(cpu_sse2_packed_cvt PRIVATE tests/cpu/sse2_mxcsr_test.asm)
    set(sse2_packed_cvt_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_sse2_packed_cvt_lifted.cpp")
    add_custom_command(OUTPUT "${sse2_packed_cvt_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sse2_packed_cvt.py"
            emit "$<TARGET_FILE:cpu_sse2_packed_cvt>" "${sse2_packed_cvt_fixture}"
        DEPENDS cpu_sse2_packed_cvt tests/cpu/sse2_packed_cvt.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_sse2_packed_cvt_lifted SHARED "${sse2_packed_cvt_fixture}")
    add_test(NAME cpu_aot_sse2_packed_cvt COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sse2_packed_cvt.py"
        check "$<TARGET_FILE:cpu_sse2_packed_cvt>" "$<TARGET_FILE:cpu_sse2_packed_cvt_lifted>")
    cpu_asm_fixture(cpu_crypto_sha crypto_sha)
    set(crypto_sha_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_crypto_sha_lifted.cpp")
    add_custom_command(OUTPUT "${crypto_sha_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/crypto_sha.py"
            emit "$<TARGET_FILE:cpu_crypto_sha>" "${crypto_sha_fixture}"
        DEPENDS cpu_crypto_sha tests/cpu/crypto_sha.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_crypto_sha_lifted SHARED "${crypto_sha_fixture}")
    add_test(NAME cpu_aot_crypto_sha COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/crypto_sha.py"
        check "$<TARGET_FILE:cpu_crypto_sha>" "$<TARGET_FILE:cpu_crypto_sha_lifted>")
    set_tests_properties(cpu_aot_crypto_sha PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_mul_div mul_div)
    set(mul_div_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_mul_div_lifted.cpp")
    add_custom_command(OUTPUT "${mul_div_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/mul_div.py"
            emit "$<TARGET_FILE:cpu_mul_div>" "${mul_div_fixture}"
        DEPENDS cpu_mul_div tests/cpu/mul_div.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_mul_div_lifted SHARED "${mul_div_fixture}")
    add_test(NAME cpu_aot_mul_div COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/mul_div.py"
        check "$<TARGET_FILE:cpu_mul_div>" "$<TARGET_FILE:cpu_mul_div_lifted>")

    cpu_asm_fixture(cpu_cmpxchg_wide cmpxchg_wide)
    set(cmpxchg_wide_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_cmpxchg_wide_lifted.cpp")
    add_custom_command(OUTPUT "${cmpxchg_wide_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/cmpxchg_wide.py"
            emit "$<TARGET_FILE:cpu_cmpxchg_wide>" "${cmpxchg_wide_fixture}"
        DEPENDS cpu_cmpxchg_wide tests/cpu/cmpxchg_wide.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_cmpxchg_wide_lifted SHARED "${cmpxchg_wide_fixture}")
    add_test(NAME cpu_aot_cmpxchg_wide COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/cmpxchg_wide.py"
        check "$<TARGET_FILE:cpu_cmpxchg_wide>" "$<TARGET_FILE:cpu_cmpxchg_wide_lifted>")

    cpu_asm_fixture(cpu_cmov_setcc cmov_setcc)
    set(cmov_setcc_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_cmov_setcc_lifted.cpp")
    add_custom_command(OUTPUT "${cmov_setcc_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/cmov_setcc.py"
            emit "$<TARGET_FILE:cpu_cmov_setcc>" "${cmov_setcc_fixture}"
        DEPENDS cpu_cmov_setcc tests/cpu/cmov_setcc.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_cmov_setcc_lifted SHARED "${cmov_setcc_fixture}")
    add_test(NAME cpu_aot_cmov_setcc COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/cmov_setcc.py"
        check "$<TARGET_FILE:cpu_cmov_setcc>" "$<TARGET_FILE:cpu_cmov_setcc_lifted>")

    cpu_asm_fixture(cpu_signed_conds signed_conds)
    set(signed_conds_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_signed_conds_lifted.cpp")
    add_custom_command(OUTPUT "${signed_conds_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/signed_conds.py"
            emit "$<TARGET_FILE:cpu_signed_conds>" "${signed_conds_fixture}"
        DEPENDS cpu_signed_conds tests/cpu/signed_conds.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_signed_conds_lifted SHARED "${signed_conds_fixture}")
    add_test(NAME cpu_aot_signed_conds COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/signed_conds.py"
        check "$<TARGET_FILE:cpu_signed_conds>" "$<TARGET_FILE:cpu_signed_conds_lifted>")
    set_tests_properties(cpu_aot_signed_conds PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_adx adx)
    set(adx_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_adx_lifted.cpp")
    add_custom_command(OUTPUT "${adx_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/adx.py"
            emit "$<TARGET_FILE:cpu_adx>" "${adx_fixture}"
        DEPENDS cpu_adx tests/cpu/adx.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_adx_lifted SHARED "${adx_fixture}")
    add_test(NAME cpu_aot_adx COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/adx.py"
        check "$<TARGET_FILE:cpu_adx>" "$<TARGET_FILE:cpu_adx_lifted>")
    set_tests_properties(cpu_aot_adx PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_avx2_shifts avx2_shifts)
    set(avx2_shifts_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_avx2_shifts_lifted.cpp")
    add_custom_command(OUTPUT "${avx2_shifts_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx2_shifts.py"
            emit "$<TARGET_FILE:cpu_avx2_shifts>" "${avx2_shifts_fixture}"
        DEPENDS cpu_avx2_shifts tests/cpu/avx2_shifts.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_avx2_shifts_lifted SHARED "${avx2_shifts_fixture}")
    add_test(NAME cpu_aot_avx2_shifts COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx2_shifts.py"
        check "$<TARGET_FILE:cpu_avx2_shifts>" "$<TARGET_FILE:cpu_avx2_shifts_lifted>")
    set_tests_properties(cpu_aot_avx2_shifts PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_avx2_permutes avx2_permutes)
    set(avx2_permutes_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_avx2_permutes_lifted.cpp")
    add_custom_command(OUTPUT "${avx2_permutes_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx2_permutes.py"
            emit "$<TARGET_FILE:cpu_avx2_permutes>" "${avx2_permutes_fixture}"
        DEPENDS cpu_avx2_permutes tests/cpu/avx2_permutes.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_avx2_permutes_lifted SHARED "${avx2_permutes_fixture}")
    add_test(NAME cpu_aot_avx2_permutes COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx2_permutes.py"
        check "$<TARGET_FILE:cpu_avx2_permutes>" "$<TARGET_FILE:cpu_avx2_permutes_lifted>")
    set_tests_properties(cpu_aot_avx2_permutes PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_sse41_int sse41_int)
    set(sse41_int_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_sse41_int_lifted.cpp")
    add_custom_command(OUTPUT "${sse41_int_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sse41_int.py"
            emit "$<TARGET_FILE:cpu_sse41_int>" "${sse41_int_fixture}"
        DEPENDS cpu_sse41_int tests/cpu/sse41_int.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_sse41_int_lifted SHARED "${sse41_int_fixture}")
    add_test(NAME cpu_aot_sse41_int COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sse41_int.py"
        check "$<TARGET_FILE:cpu_sse41_int>" "$<TARGET_FILE:cpu_sse41_int_lifted>")
    set_tests_properties(cpu_aot_sse41_int PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_sse41_extend sse41_extend)
    set(sse41_extend_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_sse41_extend_lifted.cpp")
    add_custom_command(OUTPUT "${sse41_extend_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sse41_extend.py"
            emit "$<TARGET_FILE:cpu_sse41_extend>" "${sse41_extend_fixture}"
        DEPENDS cpu_sse41_extend tests/cpu/sse41_extend.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_sse41_extend_lifted SHARED "${sse41_extend_fixture}")
    add_test(NAME cpu_aot_sse41_extend COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sse41_extend.py"
        check "$<TARGET_FILE:cpu_sse41_extend>" "$<TARGET_FILE:cpu_sse41_extend_lifted>")
    set_tests_properties(cpu_aot_sse41_extend PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_sse2_sat_mul sse2_sat_mul)
    set(sse2_sat_mul_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_sse2_sat_mul_lifted.cpp")
    add_custom_command(OUTPUT "${sse2_sat_mul_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sse2_sat_mul.py"
            emit "$<TARGET_FILE:cpu_sse2_sat_mul>" "${sse2_sat_mul_fixture}"
        DEPENDS cpu_sse2_sat_mul tests/cpu/sse2_sat_mul.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_sse2_sat_mul_lifted SHARED "${sse2_sat_mul_fixture}")
    add_test(NAME cpu_aot_sse2_sat_mul COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sse2_sat_mul.py"
        check "$<TARGET_FILE:cpu_sse2_sat_mul>" "$<TARGET_FILE:cpu_sse2_sat_mul_lifted>")

    cpu_asm_fixture(cpu_crypto_aes_pclmul crypto_aes_pclmul)
    set(crypto_aes_pclmul_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_crypto_aes_pclmul_lifted.cpp")
    add_custom_command(OUTPUT "${crypto_aes_pclmul_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/crypto_aes_pclmul.py"
            emit "$<TARGET_FILE:cpu_crypto_aes_pclmul>" "${crypto_aes_pclmul_fixture}"
        DEPENDS cpu_crypto_aes_pclmul tests/cpu/crypto_aes_pclmul.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_crypto_aes_pclmul_lifted SHARED "${crypto_aes_pclmul_fixture}")
    add_test(NAME cpu_aot_crypto_aes_pclmul COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/crypto_aes_pclmul.py"
        check "$<TARGET_FILE:cpu_crypto_aes_pclmul>" "$<TARGET_FILE:cpu_crypto_aes_pclmul_lifted>")
    set_tests_properties(cpu_aot_crypto_aes_pclmul PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_fma_scalar fma_scalar)
    set(fma_scalar_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_fma_scalar_lifted.cpp")
    add_custom_command(OUTPUT "${fma_scalar_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/fma_scalar.py"
            emit "$<TARGET_FILE:cpu_fma_scalar>" "${fma_scalar_fixture}"
        DEPENDS cpu_fma_scalar tests/cpu/fma_scalar.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_fma_scalar_lifted SHARED "${fma_scalar_fixture}")
    add_test(NAME cpu_aot_fma_scalar COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/fma_scalar.py"
        check "$<TARGET_FILE:cpu_fma_scalar>" "$<TARGET_FILE:cpu_fma_scalar_lifted>")
    set_tests_properties(cpu_aot_fma_scalar PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_sse2_int sse2_int)
    set(sse2_int_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_sse2_int_lifted.cpp")
    add_custom_command(OUTPUT "${sse2_int_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sse2_int.py"
            emit "$<TARGET_FILE:cpu_sse2_int>" "${sse2_int_fixture}"
        DEPENDS cpu_sse2_int tests/cpu/sse2_int.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_sse2_int_lifted SHARED "${sse2_int_fixture}")
    add_test(NAME cpu_aot_sse2_int COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sse2_int.py"
        check "$<TARGET_FILE:cpu_sse2_int>" "$<TARGET_FILE:cpu_sse2_int_lifted>")

    cpu_asm_fixture(cpu_sse41_42_misc sse41_42_misc)
    set(sse41_42_misc_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_sse41_42_misc_lifted.cpp")
    add_custom_command(OUTPUT "${sse41_42_misc_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sse41_42_misc.py"
            emit "$<TARGET_FILE:cpu_sse41_42_misc>" "${sse41_42_misc_fixture}"
        DEPENDS cpu_sse41_42_misc tests/cpu/sse41_42_misc.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_sse41_42_misc_lifted SHARED "${sse41_42_misc_fixture}")
    add_test(NAME cpu_aot_sse41_42_misc COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sse41_42_misc.py"
        check "$<TARGET_FILE:cpu_sse41_42_misc>" "$<TARGET_FILE:cpu_sse41_42_misc_lifted>")
    set_tests_properties(cpu_aot_sse41_42_misc PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_ssse3_int ssse3_int)
    set(ssse3_int_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_ssse3_int_lifted.cpp")
    add_custom_command(OUTPUT "${ssse3_int_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/ssse3_int.py"
            emit "$<TARGET_FILE:cpu_ssse3_int>" "${ssse3_int_fixture}"
        DEPENDS cpu_ssse3_int tests/cpu/ssse3_int.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_ssse3_int_lifted SHARED "${ssse3_int_fixture}")
    add_test(NAME cpu_aot_ssse3_int COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/ssse3_int.py"
        check "$<TARGET_FILE:cpu_ssse3_int>" "$<TARGET_FILE:cpu_ssse3_int_lifted>")
    set_tests_properties(cpu_aot_ssse3_int PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_sse2_shifts_pack sse2_shifts_pack)
    set(sse2_shifts_pack_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_sse2_shifts_pack_lifted.cpp")
    add_custom_command(OUTPUT "${sse2_shifts_pack_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sse2_shifts_pack.py"
            emit "$<TARGET_FILE:cpu_sse2_shifts_pack>" "${sse2_shifts_pack_fixture}"
        DEPENDS cpu_sse2_shifts_pack tests/cpu/sse2_shifts_pack.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_sse2_shifts_pack_lifted SHARED "${sse2_shifts_pack_fixture}")
    add_test(NAME cpu_aot_sse2_shifts_pack COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sse2_shifts_pack.py"
        check "$<TARGET_FILE:cpu_sse2_shifts_pack>" "$<TARGET_FILE:cpu_sse2_shifts_pack_lifted>")

    cpu_asm_fixture(cpu_float_shuffles float_shuffles)
    set(float_shuffles_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_float_shuffles_lifted.cpp")
    add_custom_command(OUTPUT "${float_shuffles_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/float_shuffles.py"
            emit "$<TARGET_FILE:cpu_float_shuffles>" "${float_shuffles_fixture}"
        DEPENDS cpu_float_shuffles tests/cpu/float_shuffles.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_float_shuffles_lifted SHARED "${float_shuffles_fixture}")
    add_test(NAME cpu_aot_float_shuffles COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/float_shuffles.py"
        check "$<TARGET_FILE:cpu_float_shuffles>" "$<TARGET_FILE:cpu_float_shuffles_lifted>")
    set_tests_properties(cpu_aot_float_shuffles PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_float_packed float_packed)
    set(float_packed_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_float_packed_lifted.cpp")
    add_custom_command(OUTPUT "${float_packed_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/float_packed.py"
            emit "$<TARGET_FILE:cpu_float_packed>" "${float_packed_fixture}"
        DEPENDS cpu_float_packed tests/cpu/float_packed.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_float_packed_lifted SHARED "${float_packed_fixture}")
    add_test(NAME cpu_aot_float_packed COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/float_packed.py"
        check "$<TARGET_FILE:cpu_float_packed>" "$<TARGET_FILE:cpu_float_packed_lifted>")
    set_tests_properties(cpu_aot_float_packed PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_float_conversions float_conversions)
    set(float_conversions_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_float_conversions_lifted.cpp")
    add_custom_command(OUTPUT "${float_conversions_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/float_conversions.py"
            emit "$<TARGET_FILE:cpu_float_conversions>" "${float_conversions_fixture}"
        DEPENDS cpu_float_conversions tests/cpu/float_conversions.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_float_conversions_lifted SHARED "${float_conversions_fixture}")
    add_test(NAME cpu_aot_float_conversions COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/float_conversions.py"
        check "$<TARGET_FILE:cpu_float_conversions>" "$<TARGET_FILE:cpu_float_conversions_lifted>")
    set_tests_properties(cpu_aot_float_conversions PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_fma_packed fma_packed)
    set(fma_packed_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_fma_packed_lifted.cpp")
    add_custom_command(OUTPUT "${fma_packed_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/fma_packed.py"
            emit "$<TARGET_FILE:cpu_fma_packed>" "${fma_packed_fixture}"
        DEPENDS cpu_fma_packed tests/cpu/fma_packed.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_fma_packed_lifted SHARED "${fma_packed_fixture}")
    add_test(NAME cpu_aot_fma_packed COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/fma_packed.py"
        check "$<TARGET_FILE:cpu_fma_packed>" "$<TARGET_FILE:cpu_fma_packed_lifted>")
    set_tests_properties(cpu_aot_fma_packed PROPERTIES SKIP_RETURN_CODE 77)
    add_dependencies(cpu_fma3_leaf_original cpu_fma_packed_lifted)

    cpu_asm_fixture(cpu_sse34_fp sse34_fp)
    set(sse34_fp_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_sse34_fp_lifted.cpp")
    add_custom_command(OUTPUT "${sse34_fp_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sse34_fp.py"
            emit "$<TARGET_FILE:cpu_sse34_fp>" "${sse34_fp_fixture}"
        DEPENDS cpu_sse34_fp tests/cpu/sse34_fp.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_sse34_fp_lifted SHARED "${sse34_fp_fixture}")
    add_test(NAME cpu_aot_sse34_fp COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/sse34_fp.py"
        check "$<TARGET_FILE:cpu_sse34_fp>" "$<TARGET_FILE:cpu_sse34_fp_lifted>")
    set_tests_properties(cpu_aot_sse34_fp PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_avx_packed_int avx_packed_int)
    set(avx_packed_int_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_avx_packed_int_lifted.cpp")
    add_custom_command(OUTPUT "${avx_packed_int_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_packed_int.py"
            emit "$<TARGET_FILE:cpu_avx_packed_int>" "${avx_packed_int_fixture}"
        DEPENDS cpu_avx_packed_int tests/cpu/avx_packed_int.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_avx_packed_int_lifted SHARED "${avx_packed_int_fixture}")
    add_test(NAME cpu_aot_avx_packed_int COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_packed_int.py"
        check "$<TARGET_FILE:cpu_avx_packed_int>" "$<TARGET_FILE:cpu_avx_packed_int_lifted>")
    set_tests_properties(cpu_aot_avx_packed_int PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_avx_packed_fp avx_packed_fp)
    set(avx_packed_fp_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_avx_packed_fp_lifted.cpp")
    add_custom_command(OUTPUT "${avx_packed_fp_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_packed_fp.py"
            emit "$<TARGET_FILE:cpu_avx_packed_fp>" "${avx_packed_fp_fixture}"
        DEPENDS cpu_avx_packed_fp tests/cpu/avx_packed_fp.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_avx_packed_fp_lifted SHARED "${avx_packed_fp_fixture}")
    add_test(NAME cpu_aot_avx_packed_fp COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_packed_fp.py"
        check "$<TARGET_FILE:cpu_avx_packed_fp>" "$<TARGET_FILE:cpu_avx_packed_fp_lifted>")
    set_tests_properties(cpu_aot_avx_packed_fp PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_avx_int_minmax_mul avx_int_minmax_mul)
    set(avx_int_minmax_mul_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_avx_int_minmax_mul_lifted.cpp")
    add_custom_command(OUTPUT "${avx_int_minmax_mul_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_int_minmax_mul.py"
            emit "$<TARGET_FILE:cpu_avx_int_minmax_mul>" "${avx_int_minmax_mul_fixture}"
        DEPENDS cpu_avx_int_minmax_mul tests/cpu/avx_int_minmax_mul.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_avx_int_minmax_mul_lifted SHARED "${avx_int_minmax_mul_fixture}")
    add_test(NAME cpu_aot_avx_int_minmax_mul COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_int_minmax_mul.py"
        check "$<TARGET_FILE:cpu_avx_int_minmax_mul>" "$<TARGET_FILE:cpu_avx_int_minmax_mul_lifted>")
    set_tests_properties(cpu_aot_avx_int_minmax_mul PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_avx_int_extend_hadd avx_int_extend_hadd)
    set(avx_int_extend_hadd_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_avx_int_extend_hadd_lifted.cpp")
    add_custom_command(OUTPUT "${avx_int_extend_hadd_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_int_extend_hadd.py"
            emit "$<TARGET_FILE:cpu_avx_int_extend_hadd>" "${avx_int_extend_hadd_fixture}"
        DEPENDS cpu_avx_int_extend_hadd tests/cpu/avx_int_extend_hadd.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_avx_int_extend_hadd_lifted SHARED "${avx_int_extend_hadd_fixture}")
    add_test(NAME cpu_aot_avx_int_extend_hadd COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_int_extend_hadd.py"
        check "$<TARGET_FILE:cpu_avx_int_extend_hadd>" "$<TARGET_FILE:cpu_avx_int_extend_hadd_lifted>")
    set_tests_properties(cpu_aot_avx_int_extend_hadd PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_avx_horiz_fp avx_horiz_fp)
    set(avx_horiz_fp_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_avx_horiz_fp_lifted.cpp")
    add_custom_command(OUTPUT "${avx_horiz_fp_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_horiz_fp.py"
            emit "$<TARGET_FILE:cpu_avx_horiz_fp>" "${avx_horiz_fp_fixture}"
        DEPENDS cpu_avx_horiz_fp tests/cpu/avx_horiz_fp.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_avx_horiz_fp_lifted SHARED "${avx_horiz_fp_fixture}")
    add_test(NAME cpu_aot_avx_horiz_fp COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_horiz_fp.py"
        check "$<TARGET_FILE:cpu_avx_horiz_fp>" "$<TARGET_FILE:cpu_avx_horiz_fp_lifted>")
    set_tests_properties(cpu_aot_avx_horiz_fp PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_avx_addsub_fp avx_addsub_fp)
    set(avx_addsub_fp_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_avx_addsub_fp_lifted.cpp")
    add_custom_command(OUTPUT "${avx_addsub_fp_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_addsub_fp.py"
            emit "$<TARGET_FILE:cpu_avx_addsub_fp>" "${avx_addsub_fp_fixture}"
        DEPENDS cpu_avx_addsub_fp tests/cpu/avx_addsub_fp.py tests/cpu/avx_horiz_fp.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_avx_addsub_fp_lifted SHARED "${avx_addsub_fp_fixture}")
    add_test(NAME cpu_aot_avx_addsub_fp COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_addsub_fp.py"
        check "$<TARGET_FILE:cpu_avx_addsub_fp>" "$<TARGET_FILE:cpu_avx_addsub_fp_lifted>")
    set_tests_properties(cpu_aot_avx_addsub_fp PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_avx_int_shifts_pack avx_int_shifts_pack)
    set(avx_int_shifts_pack_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_avx_int_shifts_pack_lifted.cpp")
    add_custom_command(OUTPUT "${avx_int_shifts_pack_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_int_shifts_pack.py"
            emit "$<TARGET_FILE:cpu_avx_int_shifts_pack>" "${avx_int_shifts_pack_fixture}"
        DEPENDS cpu_avx_int_shifts_pack tests/cpu/avx_int_shifts_pack.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_avx_int_shifts_pack_lifted SHARED "${avx_int_shifts_pack_fixture}")
    add_test(NAME cpu_aot_avx_int_shifts_pack COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_int_shifts_pack.py"
        check "$<TARGET_FILE:cpu_avx_int_shifts_pack>" "$<TARGET_FILE:cpu_avx_int_shifts_pack_lifted>")
    set_tests_properties(cpu_aot_avx_int_shifts_pack PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_avx_shuffle_blend avx_shuffle_blend)
    set(avx_shuffle_blend_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_avx_shuffle_blend_lifted.cpp")
    add_custom_command(OUTPUT "${avx_shuffle_blend_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_shuffle_blend.py"
            emit "$<TARGET_FILE:cpu_avx_shuffle_blend>" "${avx_shuffle_blend_fixture}"
        DEPENDS cpu_avx_shuffle_blend tests/cpu/avx_shuffle_blend.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_avx_shuffle_blend_lifted SHARED "${avx_shuffle_blend_fixture}")
    add_test(NAME cpu_aot_avx_shuffle_blend COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_shuffle_blend.py"
        check "$<TARGET_FILE:cpu_avx_shuffle_blend>" "$<TARGET_FILE:cpu_avx_shuffle_blend_lifted>")
    set_tests_properties(cpu_aot_avx_shuffle_blend PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_avx_cvt_cmp_test avx_cvt_cmp_test)
    set(avx_cvt_cmp_test_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_avx_cvt_cmp_test_lifted.cpp")
    add_custom_command(OUTPUT "${avx_cvt_cmp_test_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_cvt_cmp_test.py"
            emit "$<TARGET_FILE:cpu_avx_cvt_cmp_test>" "${avx_cvt_cmp_test_fixture}"
        DEPENDS cpu_avx_cvt_cmp_test tests/cpu/avx_cvt_cmp_test.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_avx_cvt_cmp_test_lifted SHARED "${avx_cvt_cmp_test_fixture}")
    add_test(NAME cpu_aot_avx_cvt_cmp_test COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_cvt_cmp_test.py"
        check "$<TARGET_FILE:cpu_avx_cvt_cmp_test>" "$<TARGET_FILE:cpu_avx_cvt_cmp_test_lifted>")
    set_tests_properties(cpu_aot_avx_cvt_cmp_test PROPERTIES SKIP_RETURN_CODE 77)

    cpu_asm_fixture(cpu_avx_scalar_fp avx_scalar_fp)
    set(avx_scalar_fp_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_avx_scalar_fp_lifted.cpp")
    add_custom_command(OUTPUT "${avx_scalar_fp_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_scalar_fp.py"
            emit "$<TARGET_FILE:cpu_avx_scalar_fp>" "${avx_scalar_fp_fixture}"
        DEPENDS cpu_avx_scalar_fp tests/cpu/avx_scalar_fp.py tools/ffx_version_query.py
            tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(cpu_avx_scalar_fp_lifted SHARED "${avx_scalar_fp_fixture}")
    add_test(NAME cpu_aot_avx_scalar_fp COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/avx_scalar_fp.py"
        check "$<TARGET_FILE:cpu_avx_scalar_fp>" "$<TARGET_FILE:cpu_avx_scalar_fp_lifted>")
    set_tests_properties(cpu_aot_avx_scalar_fp PROPERTIES SKIP_RETURN_CODE 77)
    cpu_asm_fixture(cpu_vex_parity vex_parity)
    # The lifted DLL, an explicit /O2 build, and two mutated controls (stalepf, swap) that must mismatch.
    set(vex_parity_control_files "")
    foreach(variant IN ITEMS lifted stalepf swap)
        set(vex_parity_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_vex_parity_${variant}.cpp")
        add_custom_command(OUTPUT "${vex_parity_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/vex_parity.py"
                emit "$<TARGET_FILE:cpu_vex_parity>" "${vex_parity_fixture}" ${variant}
            DEPENDS cpu_vex_parity tests/cpu/vex_parity.py tools/ffx_version_query.py tools/x87_lower.py
                tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_vex_parity_${variant} SHARED "${vex_parity_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND vex_parity_control_files "$<TARGET_FILE:cpu_vex_parity_${variant}>")
        endif()
    endforeach()
    set(vex_parity_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_vex_parity_lifted_o2.dll")
    add_custom_command(OUTPUT "${vex_parity_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /EHsc /LD "${CMAKE_CURRENT_BINARY_DIR}/cpu_vex_parity_lifted.cpp"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_vex_parity_lifted_o2.obj" "/Fe${vex_parity_o2}"
        DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/cpu_vex_parity_lifted.cpp" VERBATIM)
    add_custom_target(cpu_vex_parity_lifted_o2 ALL DEPENDS "${vex_parity_o2}")
    add_test(NAME cpu_aot_vex_parity COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/vex_parity.py"
        check "$<TARGET_FILE:cpu_guarded_call>" "$<TARGET_FILE:cpu_vex_parity>"
        "$<TARGET_FILE:cpu_vex_parity_lifted>" "${vex_parity_o2}" ${vex_parity_control_files})
    set_tests_properties(cpu_aot_vex_parity PROPERTIES SKIP_RETURN_CODE 77)
    cpu_translation(cpu_callback cpu_chained_aot call_context CALLBACK_OFFSET 8)
    cpu_asm_fixture(cpu_xlatb xlatb)
    # The lifted DLL and three mutated controls that the check requires to mismatch.
    foreach(variant IN ITEMS lifted signed noaddr32 clear)
        set(xlatb_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_xlatb_${variant}.cpp")
        add_custom_command(OUTPUT "${xlatb_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/xlatb.py"
                emit "$<TARGET_FILE:cpu_xlatb>" "${xlatb_fixture}" ${variant}
            DEPENDS cpu_xlatb tests/cpu/xlatb.py tools/ffx_version_query.py
                tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_xlatb_${variant} SHARED "${xlatb_fixture}")
    endforeach()
    add_test(NAME cpu_aot_xlatb COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/xlatb.py"
        check "$<TARGET_FILE:cpu_xlatb>" "$<TARGET_FILE:cpu_xlatb_lifted>"
        "$<TARGET_FILE:cpu_xlatb_signed>" "$<TARGET_FILE:cpu_xlatb_noaddr32>" "$<TARGET_FILE:cpu_xlatb_clear>")
    cpu_asm_fixture(cpu_rdtsc_harness rdtsc_harness)
    # RDTSC runs on every host. RDTSCP has its own fixture and test, which exits 77 on a host without it. Each
    # part has a lifted DLL, mutated controls that its check requires to break a property, and /O2 builds;
    # plain removes volatile and is built only at /O2.
    foreach(part IN ITEMS rdtsc rdtscp)
        cpu_asm_fixture(cpu_${part} ${part})
        if(part STREQUAL "rdtsc")
            set(tsc_variants lifted constant swap noext stale flags plain)
            set(tsc_o2_variants lifted plain)
        else()
            set(tsc_variants lifted swap noext noaux pflags)
            set(tsc_o2_variants lifted)
        endif()
        set(tsc_controls)
        foreach(variant IN LISTS tsc_variants)
            set(tsc_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_${part}_${variant}.cpp")
            add_custom_command(OUTPUT "${tsc_fixture}"
                COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/rdtsc.py"
                    emit ${part} "$<TARGET_FILE:cpu_${part}>" "${tsc_fixture}" ${variant}
                DEPENDS cpu_${part} tests/cpu/rdtsc.py tools/ffx_version_query.py tools/x87_lower.py
                    tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
            if(NOT variant STREQUAL "plain")
                add_library(cpu_${part}_${variant} SHARED "${tsc_fixture}")
                if(NOT variant STREQUAL "lifted")
                    list(APPEND tsc_controls "$<TARGET_FILE:cpu_${part}_${variant}>")
                endif()
            endif()
        endforeach()
        foreach(variant IN LISTS tsc_o2_variants)
            set(tsc_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_${part}_${variant}_o2.dll")
            add_custom_command(OUTPUT "${tsc_o2}"
                COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /LD "${CMAKE_CURRENT_BINARY_DIR}/cpu_${part}_${variant}.cpp"
                    "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_${part}_${variant}_o2.obj" "/Fe${tsc_o2}"
                DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/cpu_${part}_${variant}.cpp" VERBATIM)
            add_custom_target(cpu_${part}_${variant}_o2 ALL DEPENDS "${tsc_o2}")
            list(APPEND tsc_controls "${tsc_o2}")
        endforeach()
        add_test(NAME cpu_aot_${part} COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/rdtsc.py"
            check ${part} "$<TARGET_FILE:cpu_rdtsc_harness>" "$<TARGET_FILE:cpu_${part}>"
            "$<TARGET_FILE:cpu_${part}_lifted>" ${tsc_controls})
    endforeach()
    set_tests_properties(cpu_aot_rdtscp PROPERTIES SKIP_RETURN_CODE 77)
    cpu_asm_fixture(cpu_rdrand_harness rdrand_harness)
    # The opt-in host-entropy profile. RDRAND and RDSEED have their own fixtures and tests, each exiting 77 when
    # the host lacks the instruction. Each part has the lifted DLL, deterministic succeed and fail variants that
    # must pass, mutated controls that must break a rule, and /O2 builds; plain removes volatile and is built
    # only at /O2.
    foreach(part IN ITEMS rdrand rdseed)
        cpu_asm_fixture(cpu_${part} ${part})
        set(entropy_extras)
        foreach(variant IN ITEMS lifted succeed fail constant nocarry noext wide16 flags nozero plain)
            set(entropy_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_${part}_${variant}.cpp")
            add_custom_command(OUTPUT "${entropy_fixture}"
                COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/rdrand.py"
                    emit ${part} "$<TARGET_FILE:cpu_${part}>" "${entropy_fixture}" ${variant}
                DEPENDS cpu_${part} tests/cpu/rdrand.py tools/ffx_version_query.py tools/x87_lower.py
                    tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
            if(NOT variant STREQUAL "plain")
                add_library(cpu_${part}_${variant} SHARED "${entropy_fixture}")
                if(NOT variant STREQUAL "lifted")
                    list(APPEND entropy_extras "$<TARGET_FILE:cpu_${part}_${variant}>")
                endif()
            endif()
        endforeach()
        foreach(variant IN ITEMS lifted plain)
            set(entropy_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_${part}_${variant}_o2.dll")
            add_custom_command(OUTPUT "${entropy_o2}"
                COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /LD "${CMAKE_CURRENT_BINARY_DIR}/cpu_${part}_${variant}.cpp"
                    "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_${part}_${variant}_o2.obj" "/Fe${entropy_o2}"
                DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/cpu_${part}_${variant}.cpp" VERBATIM)
            add_custom_target(cpu_${part}_${variant}_o2 ALL DEPENDS "${entropy_o2}")
            list(APPEND entropy_extras "${entropy_o2}")
        endforeach()
        add_test(NAME cpu_aot_${part} COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/rdrand.py"
            check ${part} "$<TARGET_FILE:cpu_rdrand_harness>" "$<TARGET_FILE:cpu_${part}>"
            "$<TARGET_FILE:cpu_${part}_lifted>" ${entropy_extras})
        set_tests_properties(cpu_aot_${part} PROPERTIES SKIP_RETURN_CODE 77)
    endforeach()
    cpu_asm_fixture(cpu_cpuid_harness cpuid_harness)
    # The opt-in host-CPU profile. CPUID and XGETBV have their own fixtures and tests; cpu_aot_xgetbv exits 77
    # without OSXSAVE, and cpu_aot_cpuid when no covered leaf depends on the subleaf. Each part has the lifted
    # DLL, mutated controls that must break a property, and /O2 builds; XGETBV's plain removes volatile and is
    # built only at /O2.
    foreach(part IN ITEMS cpuid xgetbv)
        cpu_asm_fixture(cpu_${part} ${part})
        if(part STREQUAL "cpuid")
            set(host_cpu_variants lifted noext swap nosubleaf constant flags)
            set(host_cpu_o2_variants lifted)
        else()
            set(host_cpu_variants lifted noext swap constant flags plain)
            set(host_cpu_o2_variants lifted plain)
        endif()
        set(host_cpu_extras)
        foreach(variant IN LISTS host_cpu_variants)
            set(host_cpu_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_${part}_${variant}.cpp")
            add_custom_command(OUTPUT "${host_cpu_fixture}"
                COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/cpuid.py"
                    emit ${part} "$<TARGET_FILE:cpu_${part}>" "${host_cpu_fixture}" ${variant}
                DEPENDS cpu_${part} tests/cpu/cpuid.py tools/ffx_version_query.py tools/x87_lower.py
                    tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
            if(NOT variant STREQUAL "plain")
                add_library(cpu_${part}_${variant} SHARED "${host_cpu_fixture}")
                if(NOT variant STREQUAL "lifted")
                    list(APPEND host_cpu_extras "$<TARGET_FILE:cpu_${part}_${variant}>")
                endif()
            endif()
        endforeach()
        foreach(variant IN LISTS host_cpu_o2_variants)
            set(host_cpu_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_${part}_${variant}_o2.dll")
            add_custom_command(OUTPUT "${host_cpu_o2}"
                COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /LD "${CMAKE_CURRENT_BINARY_DIR}/cpu_${part}_${variant}.cpp"
                    "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_${part}_${variant}_o2.obj" "/Fe${host_cpu_o2}"
                DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/cpu_${part}_${variant}.cpp" VERBATIM)
            add_custom_target(cpu_${part}_${variant}_o2 ALL DEPENDS "${host_cpu_o2}")
            list(APPEND host_cpu_extras "${host_cpu_o2}")
        endforeach()
        add_test(NAME cpu_aot_${part} COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/cpuid.py"
            check ${part} "$<TARGET_FILE:cpu_cpuid_harness>" "$<TARGET_FILE:cpu_${part}>"
            "$<TARGET_FILE:cpu_${part}_lifted>" ${host_cpu_extras})
        set_tests_properties(cpu_aot_${part} PROPERTIES SKIP_RETURN_CODE 77)
    endforeach()
    # INT3 and UD2: the lifted DLL, the drop, swap and nostore controls that must break a rule, an /O2 build,
    # and the native SEH boundary that the check's child processes call through.
    cpu_asm_fixture(cpu_trap trap)
    add_library(cpu_trap_harness SHARED tests/cpu/trap_harness.cpp)
    set(trap_controls)
    foreach(variant IN ITEMS lifted drop swap nostore)
        set(trap_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_trap_${variant}.cpp")
        add_custom_command(OUTPUT "${trap_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/trap.py"
                emit "$<TARGET_FILE:cpu_trap>" "${trap_fixture}" ${variant}
            DEPENDS cpu_trap tests/cpu/trap.py tools/ffx_version_query.py tools/x87_lower.py
                tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_trap_${variant} SHARED "${trap_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND trap_controls "$<TARGET_FILE:cpu_trap_${variant}>")
        endif()
    endforeach()
    set(trap_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_trap_lifted_o2.dll")
    add_custom_command(OUTPUT "${trap_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /EHsc /LD "${CMAKE_CURRENT_BINARY_DIR}/cpu_trap_lifted.cpp"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_trap_lifted_o2.obj" "/Fe${trap_o2}"
        DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/cpu_trap_lifted.cpp" VERBATIM)
    add_custom_target(cpu_trap_lifted_o2 ALL DEPENDS "${trap_o2}")
    add_test(NAME cpu_aot_trap COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/trap.py"
        check "$<TARGET_FILE:cpu_trap_harness>" "$<TARGET_FILE:cpu_trap>" "$<TARGET_FILE:cpu_trap_lifted>"
        ${trap_controls} "${trap_o2}")
    # The opt-in host-MXCSR profile: the lifted DLL, the noload and fixed controls that must break a rule, and
    # an /O2 build of the translation.
    cpu_asm_fixture(cpu_mxcsr_harness mxcsr_harness)
    # The scalar float check loads the harness, and $<TARGET_FILE:> adds no build dependency, so the
    # two translations the documented build names must build it too.
    add_dependencies(cpu_float_original_lifted cpu_mxcsr_harness)
    add_dependencies(cpu_float_swapped_lifted cpu_mxcsr_harness)
    cpu_asm_fixture(cpu_mxcsr mxcsr)
    set(mxcsr_controls)
    foreach(variant IN ITEMS lifted noload fixed)
        set(mxcsr_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_mxcsr_${variant}.cpp")
        add_custom_command(OUTPUT "${mxcsr_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/mxcsr.py"
                emit "$<TARGET_FILE:cpu_mxcsr>" "${mxcsr_fixture}" ${variant}
            DEPENDS cpu_mxcsr tests/cpu/mxcsr.py tools/ffx_version_query.py tools/x87_lower.py
                tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_mxcsr_${variant} SHARED "${mxcsr_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND mxcsr_controls "$<TARGET_FILE:cpu_mxcsr_${variant}>")
        endif()
    endforeach()
    set(mxcsr_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_mxcsr_lifted_o2.dll")
    add_custom_command(OUTPUT "${mxcsr_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /EHsc /LD "${CMAKE_CURRENT_BINARY_DIR}/cpu_mxcsr_lifted.cpp"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_mxcsr_lifted_o2.obj" "/Fe${mxcsr_o2}"
        DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/cpu_mxcsr_lifted.cpp" VERBATIM)
    add_custom_target(cpu_mxcsr_lifted_o2 ALL DEPENDS "${mxcsr_o2}")
    add_test(NAME cpu_aot_mxcsr COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/mxcsr.py"
        check "$<TARGET_FILE:cpu_mxcsr_harness>" "$<TARGET_FILE:cpu_mxcsr>" "$<TARGET_FILE:cpu_mxcsr_lifted>"
        ${mxcsr_controls} "${mxcsr_o2}")
    # VLDMXCSR and VSTMXCSR: the translated part runs without AVX and never skips; the native comparison is its
    # own test and exits 77 on a host that cannot execute VEX instructions.
    cpu_asm_fixture(cpu_vmxcsr vmxcsr)
    set(vmxcsr_controls)
    foreach(variant IN ITEMS lifted noload fixed unmasked mixldload mixvstread mixvldload mixstread)
        set(vmxcsr_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_vmxcsr_${variant}.cpp")
        add_custom_command(OUTPUT "${vmxcsr_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/vmxcsr.py"
                emit "$<TARGET_FILE:cpu_vmxcsr>" "${vmxcsr_fixture}" ${variant}
            DEPENDS cpu_vmxcsr tests/cpu/vmxcsr.py tests/cpu/mxcsr.py tools/ffx_version_query.py tools/x87_lower.py
                tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_vmxcsr_${variant} SHARED "${vmxcsr_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND vmxcsr_controls "$<TARGET_FILE:cpu_vmxcsr_${variant}>")
        endif()
    endforeach()
    set(vmxcsr_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_vmxcsr_lifted_o2.dll")
    add_custom_command(OUTPUT "${vmxcsr_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /EHsc /LD "${CMAKE_CURRENT_BINARY_DIR}/cpu_vmxcsr_lifted.cpp"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_vmxcsr_lifted_o2.obj" "/Fe${vmxcsr_o2}"
        DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/cpu_vmxcsr_lifted.cpp" VERBATIM)
    add_custom_target(cpu_vmxcsr_lifted_o2 ALL DEPENDS "${vmxcsr_o2}")
    add_test(NAME cpu_aot_vmxcsr COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/vmxcsr.py"
        check translated "$<TARGET_FILE:cpu_mxcsr_harness>" "$<TARGET_FILE:cpu_vmxcsr>" "$<TARGET_FILE:cpu_vmxcsr_lifted>"
        ${vmxcsr_controls} "${vmxcsr_o2}")
    # The lowering alone refuses every malformed VEX load and store and lowers their well-formed neighbours.
    add_test(NAME cpu_aot_vmxcsr_malformed COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/vmxcsr.py"
        check malformed)
    add_test(NAME cpu_aot_vmxcsr_native COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/vmxcsr.py"
        check native "$<TARGET_FILE:cpu_mxcsr_harness>" "$<TARGET_FILE:cpu_vmxcsr>" "$<TARGET_FILE:cpu_vmxcsr_lifted>")
    set_tests_properties(cpu_aot_vmxcsr_native PROPERTIES SKIP_RETURN_CODE 77)
    # PF readers after the VEX scalar compares: the lifted DLL, the stale and reversed controls that must break a
    # rule, and an /O2 build. The MXCSR harness sets the masked MXCSR and reports whether VEX runs natively.
    cpu_asm_fixture(cpu_parity parity)
    set(parity_controls)
    foreach(variant IN ITEMS lifted stale reversed)
        set(parity_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_parity_${variant}.cpp")
        add_custom_command(OUTPUT "${parity_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/parity.py"
                emit "$<TARGET_FILE:cpu_parity>" "${parity_fixture}" ${variant}
            DEPENDS cpu_parity tests/cpu/parity.py tools/ffx_version_query.py tools/x87_lower.py
                tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_parity_${variant} SHARED "${parity_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND parity_controls "$<TARGET_FILE:cpu_parity_${variant}>")
        endif()
    endforeach()
    set(parity_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_parity_lifted_o2.dll")
    add_custom_command(OUTPUT "${parity_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /EHsc /LD "${CMAKE_CURRENT_BINARY_DIR}/cpu_parity_lifted.cpp"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_parity_lifted_o2.obj" "/Fe${parity_o2}"
        DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/cpu_parity_lifted.cpp" VERBATIM)
    add_custom_target(cpu_parity_lifted_o2 ALL DEPENDS "${parity_o2}")
    add_test(NAME cpu_aot_parity COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/parity.py"
        check "$<TARGET_FILE:cpu_mxcsr_harness>" "$<TARGET_FILE:cpu_parity>" "$<TARGET_FILE:cpu_parity_lifted>"
        ${parity_controls} "${parity_o2}")
    # FTZ and DAZ in the profile's denormals opt-in: the lifted DLL, the sim variant that must keep every rule,
    # the noftz, nodaz and nomask controls that must break one, and an /O2 build of the translation.
    cpu_asm_fixture(cpu_mxcsr_denormals mxcsr_denormals)
    set(mxcsr_denormals_extras)
    foreach(variant IN ITEMS lifted noftz nodaz sim nomask)
        set(den_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_mxcsr_denormals_${variant}.cpp")
        add_custom_command(OUTPUT "${den_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/mxcsr_denormals.py"
                emit "$<TARGET_FILE:cpu_mxcsr_denormals>" "${den_fixture}" ${variant}
            DEPENDS cpu_mxcsr_denormals tests/cpu/mxcsr_denormals.py tools/ffx_version_query.py tools/x87_lower.py
                tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_mxcsr_denormals_${variant} SHARED "${den_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND mxcsr_denormals_extras "$<TARGET_FILE:cpu_mxcsr_denormals_${variant}>")
        endif()
    endforeach()
    set(den_o2 "${CMAKE_CURRENT_BINARY_DIR}/cpu_mxcsr_denormals_lifted_o2.dll")
    add_custom_command(OUTPUT "${den_o2}"
        COMMAND "${CMAKE_CXX_COMPILER}" /nologo /O2 /W4 /EHsc /LD "${CMAKE_CURRENT_BINARY_DIR}/cpu_mxcsr_denormals_lifted.cpp"
            "/Fo${CMAKE_CURRENT_BINARY_DIR}/cpu_mxcsr_denormals_lifted_o2.obj" "/Fe${den_o2}"
        DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/cpu_mxcsr_denormals_lifted.cpp" VERBATIM)
    add_custom_target(cpu_mxcsr_denormals_lifted_o2 ALL DEPENDS "${den_o2}")
    add_test(NAME cpu_aot_mxcsr_denormals COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/mxcsr_denormals.py"
        check-ftz "$<TARGET_FILE:cpu_mxcsr_harness>" "$<TARGET_FILE:cpu_mxcsr_denormals>"
        "$<TARGET_FILE:cpu_mxcsr_denormals_lifted>" ${mxcsr_denormals_extras} "${den_o2}")
    add_test(NAME cpu_aot_mxcsr_denormals_daz COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/mxcsr_denormals.py"
        check-daz "$<TARGET_FILE:cpu_mxcsr_harness>" "$<TARGET_FILE:cpu_mxcsr_denormals>"
        "$<TARGET_FILE:cpu_mxcsr_denormals_lifted>" ${mxcsr_denormals_extras} "${den_o2}")
    set_tests_properties(cpu_aot_mxcsr_denormals_daz PROPERTIES SKIP_RETURN_CODE 77)
    cpu_asm_fixture(cpu_destroy destroy)
    target_link_libraries(cpu_destroy PRIVATE ucrt.lib)
    cpu_translation(cpu_destroy cpu_chained_aot release_context FFX_DESTROY)
    add_library(cpu_destroy_provider SHARED tests/cpu/destroy_provider.cpp)
    cpu_asm_fixture(cpu_x87_harness x87_harness)
    cpu_asm_fixture(cpu_x87_add x87_add)
    # The lifted DLL and six mutated controls that the check requires to mismatch.
    set(x87_add_controls pc64 double m80double nosticky nanorder signedzero)
    set(x87_add_control_files "")
    foreach(variant IN ITEMS lifted ${x87_add_controls})
        set(x87_add_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_add_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_add_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_add.py"
                emit "$<TARGET_FILE:cpu_x87_add>" "${x87_add_fixture}" ${variant}
            DEPENDS cpu_x87_add tests/cpu/x87_add.py tools/ffx_version_query.py tools/x87_lower.py
                tools/x87_soft.h tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_add_${variant} SHARED "${x87_add_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_add_control_files "$<TARGET_FILE:cpu_x87_add_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_add COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_add.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_add>"
        "$<TARGET_FILE:cpu_x87_add_lifted>" ${x87_add_control_files})
    cpu_asm_fixture(cpu_x87_mul x87_mul)
    # The lifted DLL and six mutated controls that the check requires to mismatch.
    set(x87_mul_control_files "")
    foreach(variant IN ITEMS lifted pc64 double nolow sign infzero memquiet)
        set(x87_mul_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_mul_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_mul_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_mul.py"
                emit "$<TARGET_FILE:cpu_x87_mul>" "${x87_mul_fixture}" ${variant}
            DEPENDS cpu_x87_mul tests/cpu/x87_mul.py tests/cpu/x87_add.py tools/ffx_version_query.py
                tools/x87_lower.py tools/x87_soft.h tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_mul_${variant} SHARED "${x87_mul_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_mul_control_files "$<TARGET_FILE:cpu_x87_mul_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_mul COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_mul.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_mul>"
        "$<TARGET_FILE:cpu_x87_mul_lifted>" ${x87_mul_control_files})
    cpu_asm_fixture(cpu_x87_div x87_div)
    # The lifted DLL and seven mutated controls that the check requires to mismatch.
    set(x87_div_control_files "")
    foreach(variant IN ITEMS lifted pc64 double nosticky divzero reverse zerosign memquiet)
        set(x87_div_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_div_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_div_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_div.py"
                emit "$<TARGET_FILE:cpu_x87_div>" "${x87_div_fixture}" ${variant}
            DEPENDS cpu_x87_div tests/cpu/x87_div.py tests/cpu/x87_mul.py tests/cpu/x87_add.py
                tools/ffx_version_query.py tools/x87_lower.py tools/x87_soft.h tools/cpu_ranges.py
                tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_div_${variant} SHARED "${x87_div_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_div_control_files "$<TARGET_FILE:cpu_x87_div_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_div COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_div.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_div>"
        "$<TARGET_FILE:cpu_x87_div_lifted>" ${x87_div_control_files})
    cpu_asm_fixture(cpu_x87_sub x87_sub)
    # The lifted DLL and seven mutated controls that the check requires to mismatch.
    set(x87_sub_control_files "")
    foreach(variant IN ITEMS lifted pc64 double nansign reverse cancel nosticky memquiet)
        set(x87_sub_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_sub_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_sub_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_sub.py"
                emit "$<TARGET_FILE:cpu_x87_sub>" "${x87_sub_fixture}" ${variant}
            DEPENDS cpu_x87_sub tests/cpu/x87_sub.py tests/cpu/x87_div.py tests/cpu/x87_mul.py
                tests/cpu/x87_add.py tools/ffx_version_query.py tools/x87_lower.py tools/x87_soft.h
                tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_sub_${variant} SHARED "${x87_sub_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_sub_control_files "$<TARGET_FILE:cpu_x87_sub_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_sub COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_sub.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_sub>"
        "$<TARGET_FILE:cpu_x87_sub_lifted>" ${x87_sub_control_files})
    cpu_asm_fixture(cpu_x87_moves x87_moves)
    # The lifted DLL and seven mutated controls that the check requires to mismatch.
    set(x87_moves_control_files "")
    foreach(variant IN ITEMS lifted chssub absnan fxchnone fldtop fstpdrop fstwrite copyquiet)
        set(x87_moves_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_moves_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_moves_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_moves.py"
                emit "$<TARGET_FILE:cpu_x87_moves>" "${x87_moves_fixture}" ${variant}
            DEPENDS cpu_x87_moves tests/cpu/x87_moves.py tests/cpu/x87_sub.py tests/cpu/x87_div.py
                tests/cpu/x87_mul.py tests/cpu/x87_add.py tools/ffx_version_query.py tools/x87_lower.py
                tools/x87_soft.h tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_moves_${variant} SHARED "${x87_moves_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_moves_control_files "$<TARGET_FILE:cpu_x87_moves_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_moves COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_moves.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_moves>"
        "$<TARGET_FILE:cpu_x87_moves_lifted>" ${x87_moves_control_files})
    cpu_asm_fixture(cpu_x87_constants x87_constants)
    # The lifted DLL and four mutated controls that the check requires to mismatch.
    set(x87_constants_control_files "")
    foreach(variant IN ITEMS lifted pc53 chop fldzneg swaplogs)
        set(x87_constants_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_constants_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_constants_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_constants.py"
                emit "$<TARGET_FILE:cpu_x87_constants>" "${x87_constants_fixture}" ${variant}
            DEPENDS cpu_x87_constants tests/cpu/x87_constants.py tests/cpu/x87_moves.py tests/cpu/x87_sub.py
                tests/cpu/x87_div.py tests/cpu/x87_mul.py tests/cpu/x87_add.py tools/ffx_version_query.py
                tools/x87_lower.py tools/x87_soft.h tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_constants_${variant} SHARED "${x87_constants_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_constants_control_files "$<TARGET_FILE:cpu_x87_constants_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_constants COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_constants.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_constants>"
        "$<TARGET_FILE:cpu_x87_constants_lifted>" ${x87_constants_control_files})
    cpu_asm_fixture(cpu_x87_integers x87_integers)
    # The lifted DLL and six mutated controls that the check requires to mismatch.
    set(x87_integers_control_files "")
    foreach(variant IN ITEMS lifted chopall roundtt saturate nanzero zext fild53)
        set(x87_integers_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_integers_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_integers_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_integers.py"
                emit "$<TARGET_FILE:cpu_x87_integers>" "${x87_integers_fixture}" ${variant}
            DEPENDS cpu_x87_integers tests/cpu/x87_integers.py tests/cpu/x87_moves.py tests/cpu/x87_sub.py
                tests/cpu/x87_div.py tests/cpu/x87_mul.py tests/cpu/x87_add.py tools/ffx_version_query.py
                tools/x87_lower.py tools/x87_soft.h tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_integers_${variant} SHARED "${x87_integers_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_integers_control_files "$<TARGET_FILE:cpu_x87_integers_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_integers COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_integers.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_integers>"
        "$<TARGET_FILE:cpu_x87_integers_lifted>" ${x87_integers_control_files})
    cpu_asm_fixture(cpu_x87_compare x87_compare)
    # The lifted DLL and seven mutated controls that the check requires to mismatch.
    set(x87_compare_control_files "")
    foreach(variant IN ITEMS lifted unordeq swap zeroorder low11 rawexp unnormal keepof)
        set(x87_compare_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_compare_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_compare_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_compare.py"
                emit "$<TARGET_FILE:cpu_x87_compare>" "${x87_compare_fixture}" ${variant}
            DEPENDS cpu_x87_compare tests/cpu/x87_compare.py tests/cpu/x87_moves.py tests/cpu/x87_sub.py tests/cpu/adx.py
                tests/cpu/x87_div.py tests/cpu/x87_mul.py tests/cpu/x87_add.py tools/ffx_version_query.py
                tools/x87_lower.py tools/x87_soft.h tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_compare_${variant} SHARED "${x87_compare_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_compare_control_files "$<TARGET_FILE:cpu_x87_compare_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_compare COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_compare.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_compare>"
        "$<TARGET_FILE:cpu_x87_compare_lifted>" ${x87_compare_control_files})
    # Skips (77) after every other check when the host lacks ADX for the native ADOX export.
    set_tests_properties(cpu_aot_x87_compare PROPERTIES SKIP_RETURN_CODE 77)
    cpu_asm_fixture(cpu_x87_sqrt x87_sqrt)
    # The lifted DLL and six mutated controls that the check requires to mismatch.
    set(x87_sqrt_control_files "")
    foreach(variant IN ITEMS lifted pc64 double nosticky negzero negabs snan)
        set(x87_sqrt_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_sqrt_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_sqrt_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_sqrt.py"
                emit "$<TARGET_FILE:cpu_x87_sqrt>" "${x87_sqrt_fixture}" ${variant}
            DEPENDS cpu_x87_sqrt tests/cpu/x87_sqrt.py tests/cpu/x87_moves.py tests/cpu/x87_sub.py
                tests/cpu/x87_div.py tests/cpu/x87_mul.py tests/cpu/x87_add.py tools/ffx_version_query.py
                tools/x87_lower.py tools/x87_soft.h tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_sqrt_${variant} SHARED "${x87_sqrt_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_sqrt_control_files "$<TARGET_FILE:cpu_x87_sqrt_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_sqrt COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_sqrt.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_sqrt>"
        "$<TARGET_FILE:cpu_x87_sqrt_lifted>" ${x87_sqrt_control_files})
    cpu_asm_fixture(cpu_x87_rndint x87_rndint)
    # The lifted DLL and five mutated controls that the check requires to mismatch.
    set(x87_rndint_control_files "")
    foreach(variant IN ITEMS lifted chop tiesaway zerosign trunc53 snan)
        set(x87_rndint_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_rndint_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_rndint_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_rndint.py"
                emit "$<TARGET_FILE:cpu_x87_rndint>" "${x87_rndint_fixture}" ${variant}
            DEPENDS cpu_x87_rndint tests/cpu/x87_rndint.py tests/cpu/x87_integers.py tests/cpu/x87_moves.py tests/cpu/x87_sub.py
                tests/cpu/x87_div.py tests/cpu/x87_mul.py tests/cpu/x87_add.py tools/ffx_version_query.py
                tools/x87_lower.py tools/x87_soft.h tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_rndint_${variant} SHARED "${x87_rndint_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_rndint_control_files "$<TARGET_FILE:cpu_x87_rndint_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_rndint COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_rndint.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_rndint>"
        "$<TARGET_FILE:cpu_x87_rndint_lifted>" ${x87_rndint_control_files})
    cpu_asm_fixture(cpu_x87_xtract x87_xtract)
    # The lifted DLL and six mutated controls that the check requires to mismatch.
    set(x87_xtract_control_files "")
    foreach(variant IN ITEMS lifted zeroexp denormal biased unsigned snan order)
        set(x87_xtract_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_xtract_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_xtract_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_xtract.py"
                emit "$<TARGET_FILE:cpu_x87_xtract>" "${x87_xtract_fixture}" ${variant}
            DEPENDS cpu_x87_xtract tests/cpu/x87_xtract.py tests/cpu/x87_integers.py tests/cpu/x87_moves.py tests/cpu/x87_sub.py
                tests/cpu/x87_div.py tests/cpu/x87_mul.py tests/cpu/x87_add.py tools/ffx_version_query.py
                tools/x87_lower.py tools/x87_soft.h tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_xtract_${variant} SHARED "${x87_xtract_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_xtract_control_files "$<TARGET_FILE:cpu_x87_xtract_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_xtract COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_xtract.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_xtract>"
        "$<TARGET_FILE:cpu_x87_xtract_lifted>" ${x87_xtract_control_files})
    cpu_asm_fixture(cpu_x87_scale x87_scale)
    # The lifted DLL and six mutated controls that the check requires to mismatch.
    set(x87_scale_control_files "")
    foreach(variant IN ITEMS lifted pc53 chop floor neginf zeroinf swap)
        set(x87_scale_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_scale_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_scale_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_scale.py"
                emit "$<TARGET_FILE:cpu_x87_scale>" "${x87_scale_fixture}" ${variant}
            DEPENDS cpu_x87_scale tests/cpu/x87_scale.py tests/cpu/x87_xtract.py tests/cpu/x87_integers.py tests/cpu/x87_moves.py tests/cpu/x87_sub.py
                tests/cpu/x87_div.py tests/cpu/x87_mul.py tests/cpu/x87_add.py tools/ffx_version_query.py
                tools/x87_lower.py tools/x87_soft.h tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_scale_${variant} SHARED "${x87_scale_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_scale_control_files "$<TARGET_FILE:cpu_x87_scale_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_scale COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_scale.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_scale>"
        "$<TARGET_FILE:cpu_x87_scale_lifted>" ${x87_scale_control_files})
    cpu_asm_fixture(cpu_x87_fcmov x87_fcmov)
    # The lifted DLL and eight mutated controls that the check requires to mismatch.
    set(x87_fcmov_control_files "")
    foreach(variant IN ITEMS lifted invert beonly nbeor neverne always quiet cmcdrop nocarryin)
        set(x87_fcmov_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_fcmov_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_fcmov_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_fcmov.py"
                emit "$<TARGET_FILE:cpu_x87_fcmov>" "${x87_fcmov_fixture}" ${variant}
            DEPENDS cpu_x87_fcmov tests/cpu/x87_fcmov.py tests/cpu/x87_compare.py tests/cpu/x87_integers.py tests/cpu/x87_moves.py tests/cpu/x87_sub.py
                tests/cpu/x87_div.py tests/cpu/x87_mul.py tests/cpu/x87_add.py tools/ffx_version_query.py
                tools/x87_lower.py tools/x87_soft.h tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_fcmov_${variant} SHARED "${x87_fcmov_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_fcmov_control_files "$<TARGET_FILE:cpu_x87_fcmov_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_fcmov COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_fcmov.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_fcmov>"
        "$<TARGET_FILE:cpu_x87_fcmov_lifted>" ${x87_fcmov_control_files})
    cpu_asm_fixture(cpu_x87_iarith x87_iarith)
    # The lifted DLL and six mutated controls that the check requires to mismatch.
    set(x87_iarith_control_files "")
    foreach(variant IN ITEMS lifted subrev divrev zext16 zext32 wide16 pc64)
        set(x87_iarith_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_iarith_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_iarith_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_iarith.py"
                emit "$<TARGET_FILE:cpu_x87_iarith>" "${x87_iarith_fixture}" ${variant}
            DEPENDS cpu_x87_iarith tests/cpu/x87_iarith.py tests/cpu/x87_integers.py tests/cpu/x87_moves.py tests/cpu/x87_sub.py
                tests/cpu/x87_div.py tests/cpu/x87_mul.py tests/cpu/x87_add.py tools/ffx_version_query.py
                tools/x87_lower.py tools/x87_soft.h tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_iarith_${variant} SHARED "${x87_iarith_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_iarith_control_files "$<TARGET_FILE:cpu_x87_iarith_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_iarith COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_iarith.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_iarith>"
        "$<TARGET_FILE:cpu_x87_iarith_lifted>" ${x87_iarith_control_files})
    cpu_asm_fixture(cpu_x87_fnstcw x87_fnstcw)
    # The lifted DLL and four mutated controls that the check requires to mismatch.
    set(x87_fnstcw_control_files "")
    foreach(variant IN ITEMS lifted fninit wide skip lowbyte)
        set(x87_fnstcw_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_fnstcw_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_fnstcw_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_fnstcw.py"
                emit "$<TARGET_FILE:cpu_x87_fnstcw>" "${x87_fnstcw_fixture}" ${variant}
            DEPENDS cpu_x87_fnstcw tests/cpu/x87_fnstcw.py
                tests/cpu/x87_div.py tests/cpu/x87_mul.py tests/cpu/x87_add.py tools/ffx_version_query.py
                tools/x87_lower.py tools/x87_soft.h tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_fnstcw_${variant} SHARED "${x87_fnstcw_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_fnstcw_control_files "$<TARGET_FILE:cpu_x87_fnstcw_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_fnstcw COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_fnstcw.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_fnstcw>"
        "$<TARGET_FILE:cpu_x87_fnstcw_lifted>" ${x87_fnstcw_control_files})
    cpu_asm_fixture(cpu_x87_examine x87_examine)
    # The lifted DLL and eight mutated controls that the check requires to mismatch.
    set(x87_examine_control_files "")
    foreach(variant IN ITEMS lifted normal unsigned positive legacy ffreep fftop ffree freetop)
        set(x87_examine_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_examine_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_examine_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_examine.py"
                emit "$<TARGET_FILE:cpu_x87_examine>" "${x87_examine_fixture}" ${variant}
            DEPENDS cpu_x87_examine tests/cpu/x87_examine.py tests/cpu/x87_add.py tools/ffx_version_query.py
                tools/x87_lower.py tools/x87_soft.h tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_examine_${variant} SHARED "${x87_examine_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_examine_control_files "$<TARGET_FILE:cpu_x87_examine_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_examine COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_examine.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_examine>"
        "$<TARGET_FILE:cpu_x87_examine_lifted>" ${x87_examine_control_files})
    cpu_asm_fixture(cpu_x87_fprem x87_fprem)
    # The lifted DLL and eight mutated controls that the check requires to mismatch.
    set(x87_fprem_control_files "")
    foreach(variant IN ITEMS lifted nopartial nonan d63 truncate1 tieup halfnear qswap zerosign)
        set(x87_fprem_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_fprem_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_fprem_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_fprem.py"
                emit "$<TARGET_FILE:cpu_x87_fprem>" "${x87_fprem_fixture}" ${variant}
            DEPENDS cpu_x87_fprem tests/cpu/x87_fprem.py tests/cpu/x87_add.py tools/ffx_version_query.py
                tools/x87_lower.py tools/x87_soft.h tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_fprem_${variant} SHARED "${x87_fprem_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_fprem_control_files "$<TARGET_FILE:cpu_x87_fprem_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_fprem COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_fprem.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_fprem>"
        "$<TARGET_FILE:cpu_x87_fprem_lifted>" ${x87_fprem_control_files})
    cpu_asm_fixture(cpu_x87_fldcw x87_fldcw)
    # The checked translation and seventeen mutated controls that the check requires to mismatch.
    set(x87_fldcw_control_files "")
    foreach(variant IN ITEMS lifted rcnearest pc53 fistnearest constnearest rawword nounmasked pc01 noreturn onereturn fninitword fninitsw deferred swtop swbyte swwide swaddr swnone)
        set(x87_fldcw_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_fldcw_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_fldcw_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_fldcw.py"
                emit "$<TARGET_FILE:cpu_x87_fldcw>" "${x87_fldcw_fixture}" ${variant}
            DEPENDS cpu_x87_fldcw tests/cpu/x87_fldcw.py tests/cpu/x87_add.py tests/cpu/x87_constants.py
                tests/cpu/x87_div.py tests/cpu/x87_integers.py tests/cpu/x87_moves.py tests/cpu/x87_mul.py
                tests/cpu/x87_rndint.py tests/cpu/x87_scale.py tests/cpu/x87_sqrt.py tests/cpu/x87_sub.py tests/cpu/x87_xtract.py
                tools/ffx_version_query.py tools/x87_lower.py tools/x87_soft.h tools/cpu_ranges.py
                tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_fldcw_${variant} SHARED "${x87_fldcw_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_fldcw_control_files "$<TARGET_FILE:cpu_x87_fldcw_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_fldcw COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_fldcw.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_fldcw>"
        "$<TARGET_FILE:cpu_x87_fldcw_lifted>" ${x87_fldcw_control_files})
    cpu_asm_fixture(cpu_x87_status x87_status)
    # The lifted DLL and seven mutated controls that the check requires to mismatch.
    set(x87_status_control_files "")
    foreach(variant IN ITEMS lifted c2drop c0c3 top0 sahfpf oddparity andparity fcomiclear)
        set(x87_status_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_status_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_status_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_status.py"
                emit "$<TARGET_FILE:cpu_x87_status>" "${x87_status_fixture}" ${variant}
            DEPENDS cpu_x87_status tests/cpu/x87_status.py tests/cpu/x87_compare.py tests/cpu/x87_moves.py
                tests/cpu/x87_div.py tests/cpu/x87_mul.py tests/cpu/x87_sub.py tests/cpu/x87_add.py
                tools/ffx_version_query.py tools/x87_lower.py tools/x87_soft.h tools/cpu_ranges.py
                tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_status_${variant} SHARED "${x87_status_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_status_control_files "$<TARGET_FILE:cpu_x87_status_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_status COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_status.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_status>"
        "$<TARGET_FILE:cpu_x87_status_lifted>" ${x87_status_control_files})
    cpu_asm_fixture(cpu_x87_parity x87_parity)
    # The lifted DLL and five mutated controls that the check requires to mismatch.
    set(x87_parity_control_files "")
    foreach(variant IN ITEMS lifted pfnever pfequal pfkeep uinvert nuinvert)
        set(x87_parity_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_parity_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_parity_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_parity.py"
                emit "$<TARGET_FILE:cpu_x87_parity>" "${x87_parity_fixture}" ${variant}
            DEPENDS cpu_x87_parity tests/cpu/x87_parity.py tests/cpu/x87_status.py tests/cpu/x87_compare.py
                tests/cpu/x87_moves.py tests/cpu/x87_div.py tests/cpu/x87_mul.py tests/cpu/x87_sub.py
                tests/cpu/x87_add.py tools/ffx_version_query.py tools/x87_lower.py tools/x87_soft.h
                tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_parity_${variant} SHARED "${x87_parity_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_parity_control_files "$<TARGET_FILE:cpu_x87_parity_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_parity COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_parity.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_parity>"
        "$<TARGET_FILE:cpu_x87_parity_lifted>" ${x87_parity_control_files})
    cpu_asm_fixture(cpu_x87_bcd x87_bcd)
    # The lifted DLL and nine mutated controls that the check requires to mismatch.
    set(x87_bcd_control_files "")
    foreach(variant IN ITEMS lifted rcnearest rangebefore threshold zerosign indefinite bstpnibbles bldnibbles
            signbits nodigitcheck)
        set(x87_bcd_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_x87_bcd_${variant}.cpp")
        add_custom_command(OUTPUT "${x87_bcd_fixture}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_bcd.py"
                emit "$<TARGET_FILE:cpu_x87_bcd>" "${x87_bcd_fixture}" ${variant}
            DEPENDS cpu_x87_bcd tests/cpu/x87_bcd.py tests/cpu/x87_add.py tools/ffx_version_query.py
                tools/x87_lower.py tools/x87_soft.h tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
        add_library(cpu_x87_bcd_${variant} SHARED "${x87_bcd_fixture}")
        if(NOT variant STREQUAL "lifted")
            list(APPEND x87_bcd_control_files "$<TARGET_FILE:cpu_x87_bcd_${variant}>")
        endif()
    endforeach()
    add_test(NAME cpu_aot_x87_bcd COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/x87_bcd.py"
        check "$<TARGET_FILE:cpu_x87_harness>" "$<TARGET_FILE:cpu_x87_bcd>"
        "$<TARGET_FILE:cpu_x87_bcd_lifted>" ${x87_bcd_control_files})
    cpu_asm_fixture(cpu_create create_context)
    set(create_fixture "${CMAKE_CURRENT_BINARY_DIR}/cpu_create_lifted.cpp")
    add_custom_command(OUTPUT "${create_fixture}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/cpu/create_fixture.py"
            "$<TARGET_FILE:cpu_create>" "${create_fixture}"
        DEPENDS cpu_create tests/cpu/create_fixture.py tools/ffx_create_context.py
            tools/ffx_version_query.py tools/cpu_leaf_aot.py tools/cpu_ranges.py VERBATIM)
    add_library(cpu_create_lifted SHARED "${create_fixture}")
    add_test(NAME cpu_aot_context_lifecycle COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/context_lifecycle.py" "$<TARGET_FILE:cpu_create>"
        "$<TARGET_FILE:cpu_create_lifted>" "$<TARGET_FILE:cpu_destroy_lifted>")
    add_test(NAME cpu_aot_destroy_contract COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/destroy_check.py" "$<TARGET_FILE:cpu_destroy>"
        "$<TARGET_FILE:cpu_destroy_lifted>" "$<TARGET_FILE:cpu_destroy_provider>")
    add_test(NAME cpu_aot_object_callback COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/callback_check.py" "$<TARGET_FILE:cpu_callback>"
        "$<TARGET_FILE:cpu_callback_lifted>")
    add_test(NAME cpu_aot_chained_ranges COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/chained_check.py" "$<TARGET_FILE:cpu_chained>"
        "$<TARGET_FILE:cpu_chained_lifted>" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_chained_jne>" "$<TARGET_FILE:cpu_chained_jne_lifted>"
        "$<TARGET_FILE:cpu_chained_zero>" "$<TARGET_FILE:cpu_chained_zero_lifted>"
        "$<TARGET_FILE:cpu_chained_imm>" "$<TARGET_FILE:cpu_chained_imm_lifted>"
        "$<TARGET_FILE:cpu_chained_frame>" "$<TARGET_FILE:cpu_chained_frame_lifted>")
    add_test(NAME cpu_aot_chained_dead_reads COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/chained_dead_read_check.py" "$<TARGET_FILE:cpu_guarded_call>"
        "$<TARGET_FILE:cpu_chained_dead_read>" ${chained_dead_read_files})
    add_test(NAME cpu_aot_direct_call COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/direct_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_call_original>" "$<TARGET_FILE:cpu_call_original_lifted>"
        "$<TARGET_FILE:cpu_call_altered_lifted>")
    add_test(NAME cpu_aot_call_pair COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/call_pair_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_pair_original>" "$<TARGET_FILE:cpu_pair_swapped>"
        "$<TARGET_FILE:cpu_pair_original_lifted>" "$<TARGET_FILE:cpu_pair_swapped_lifted>"
        "$<TARGET_FILE:cpu_pair_first_lifted>" "$<TARGET_FILE:cpu_pair_second_lifted>")
    foreach(kind IN ITEMS named ordinal)
        if(kind STREQUAL "named")
            set(prefix cpu_import)
            set(contract SYMBOL mix_args)
        else()
            set(prefix cpu_ordinal)
            set(contract SYMBOL "#77" LINK_NAME mix_args)
        endif()
        foreach(variant IN ITEMS provider altered)
            add_library(${prefix}_${variant} SHARED tests/cpu/import_provider.cpp tests/cpu/${kind}_provider.def)
        endforeach()
        target_compile_definitions(${prefix}_altered PRIVATE ALTER_RETURN)
        cpu_asm_fixture(${prefix}_original direct import)
        target_compile_definitions(${prefix}_original PRIVATE IMPORT_MIX)
        target_link_libraries(${prefix}_original PRIVATE ${prefix}_provider)
        cpu_translation(${prefix}_original cpu_call_aot call_mix PROVIDER ${prefix}_provider ${contract})
        target_link_options(${prefix}_original_lifted PRIVATE /INCREMENTAL:NO)
    endforeach()
    cpu_asm_fixture(cpu_repeat_import repeat_call repeat_import)
    target_compile_definitions(cpu_repeat_import PRIVATE IMPORT_MIX)
    target_link_libraries(cpu_repeat_import PRIVATE cpu_import_provider)
    cpu_translation(cpu_repeat_import cpu_call_aot call_twice PROVIDER cpu_import_provider SYMBOL mix_args)
    target_link_options(cpu_repeat_import_lifted PRIVATE /INCREMENTAL:NO)
    add_test(NAME cpu_aot_repeated_call COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/repeat_call_check.py" "$<TARGET_FILE:cpu_register_probe>"
        "$<TARGET_FILE:cpu_repeat_original>" "$<TARGET_FILE:cpu_repeat_original_lifted>"
        "$<TARGET_FILE:cpu_repeat_altered_lifted>" "$<TARGET_FILE:cpu_repeat_dropped>"
        "$<TARGET_FILE:cpu_repeat_import>" "$<TARGET_FILE:cpu_repeat_import_lifted>")
    add_test(NAME cpu_aot_import_call COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/import_check.py" "$<TARGET_FILE:cpu_import_original>"
        "$<TARGET_FILE:cpu_import_original_lifted>" "$<TARGET_FILE:cpu_import_provider>"
        "$<TARGET_FILE:cpu_import_altered>" "$<TARGET_FILE:cpu_register_probe>")
    add_library(cpu_exe_io SHARED tests/cpu/exe_io.cpp)
    # Use Zig's normal single-threaded console startup, including RtlExitUserProcess.
    enable_language(C)
    set(archive_src "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/archive")
    # Keep the scalar reference PE identical across the Debug and /O2 host builds.
    set(blast_obj "${CMAKE_CURRENT_BINARY_DIR}/blast_scalar.obj")
    add_custom_command(OUTPUT "${blast_obj}"
        COMMAND "${CMAKE_C_COMPILER}" /nologo /c /MD /Od /GS- /Gs65536 /Zl "/Fo${blast_obj}" "${archive_src}/blast/blast.c"
        DEPENDS "${archive_src}/blast/blast.c" "${archive_src}/blast/blast.h" VERBATIM)
    add_library(blast_source SHARED "${blast_obj}" "${archive_src}/blast.def")
    set_target_properties(blast_source PROPERTIES LINKER_LANGUAGE C)
    target_link_options(blast_source PRIVATE /NOENTRY /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(blast_source PRIVATE msvcrt ucrt vcruntime)

    set(blast_cpp "${CMAKE_CURRENT_BINARY_DIR}/blast_lifted.cpp")
    add_custom_command(OUTPUT "${blast_cpp}"
        COMMAND "${CMAKE_COMMAND}" -E env "PYTHONPATH=${CMAKE_CURRENT_SOURCE_DIR}/tools"
            "${INTAKE_PYTHON}" -B "${archive_src}/translate_blast.py" "$<TARGET_FILE:blast_source>" "${blast_cpp}"
        DEPENDS blast_source "${archive_src}/translate_blast.py" "${archive_src}/blast_runtime.hpp"
            tools/cpu_switch_aot.py tools/cpu_ranges.py tools/cpu_leaf_aot.py VERBATIM)
    add_library(blast_lifted SHARED "${blast_cpp}" "${archive_src}/blast.def")
    target_include_directories(blast_lifted PRIVATE "${archive_src}")

    add_library(archive_codec_lifted SHARED "${archive_src}/codec.cpp" "${archive_src}/texture_stdout.cpp")
    target_link_libraries(archive_codec_lifted PRIVATE blast_lifted)
    set_target_properties(archive_codec_lifted PROPERTIES MSVC_RUNTIME_LIBRARY MultiThreadedDLL)
    add_library(archive_codec SHARED "${archive_src}/codec.cpp" "${archive_src}/texture_stdout.cpp")
    target_link_libraries(archive_codec PRIVATE blast_source)
    set_target_properties(archive_codec PROPERTIES MSVC_RUNTIME_LIBRARY MultiThreadedDLL)
    function(build_archive_reader entry exe cpp provider translator)
        cmake_parse_arguments(READER "EXPORT_SYMBOLS" "" "" ${ARGN})
        set(export_flags)
        if(READER_EXPORT_SYMBOLS)
            # Publish the directory lookup's leaf entry for the function-boundary verifier.
            set(export_flags -rdynamic)
        endif()
        add_custom_command(OUTPUT "${exe}"
            COMMAND "${INTAKE_PYTHON}" -m ziglang build-exe "${entry}"
                "-L$<TARGET_LINKER_FILE_DIR:${provider}>" -target x86_64-windows-gnu
                -O ReleaseSmall -fsingle-threaded -funwind-tables ${export_flags} "-femit-bin=${exe}"
            DEPENDS ${provider} "${entry}" ${READER_UNPARSED_ARGUMENTS} VERBATIM)
        add_custom_command(OUTPUT "${cpp}"
            COMMAND "${CMAKE_COMMAND}" -E env "PYTHONPATH=${CMAKE_CURRENT_SOURCE_DIR}/tools"
                "${INTAKE_PYTHON}" -B "${archive_src}/${translator}" "${exe}" "${cpp}"
            DEPENDS "${exe}" "${archive_src}/${translator}" tools/cpu_stdio_exe_aot.py
                tools/cpu_switch_aot.py tools/cpu_leaf_aot.py tools/cpu_ranges.py VERBATIM)
    endfunction()
    set(archive_exe "${CMAKE_CURRENT_BINARY_DIR}/archive_reader.exe")
    set(archive_cpp "${CMAKE_CURRENT_BINARY_DIR}/archive_reader_lifted.cpp")
    build_archive_reader("${archive_src}/reader.zig" "${archive_exe}" "${archive_cpp}" archive_codec translate_reader.py)
    add_executable(archive_reader_lifted "${archive_cpp}")
    target_link_libraries(archive_reader_lifted PRIVATE archive_codec ntdll)
    add_test(NAME cpu_native_archive_reader COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/archive_check.py" "$<TARGET_FILE:pe_map>"
        "${archive_exe}" "$<TARGET_FILE:archive_reader_lifted>" "$<TARGET_FILE:archive_codec>")

    add_executable(archive_all_lifted "${archive_cpp}")
    target_link_libraries(archive_all_lifted PRIVATE archive_codec_lifted ntdll)
    add_test(NAME cpu_native_archive_blast COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/archive_check.py" "$<TARGET_FILE:pe_map>"
        "${archive_exe}" "$<TARGET_FILE:archive_all_lifted>" "$<TARGET_FILE:archive_codec>"
        "$<TARGET_FILE:blast_source>" "$<TARGET_FILE:blast_lifted>" "$<TARGET_FILE:archive_codec_lifted>")
    # 77: Control Flow Guard enforced for the pe_map runner, whose blast longjmp cases then cannot run (#1293).
    set_tests_properties(cpu_native_archive_blast PROPERTIES SKIP_RETURN_CODE 77)

    set(texture_exe "${CMAKE_CURRENT_BINARY_DIR}/archive_texture_reader.exe")
    set(texture_cpp "${CMAKE_CURRENT_BINARY_DIR}/archive_texture_reader_lifted.cpp")
    build_archive_reader("${archive_src}/texture_reader.zig" "${texture_exe}" "${texture_cpp}" archive_codec translate_reader.py
        "${archive_src}/reader.zig")
    add_executable(archive_texture_lifted "${texture_cpp}")
    target_link_libraries(archive_texture_lifted PRIVATE archive_codec_lifted ntdll)
    add_test(NAME cpu_archive_texture_publication COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/archive_texture_check.py" "$<TARGET_FILE:pe_map>"
        "${texture_exe}" "$<TARGET_FILE:archive_texture_lifted>" "$<TARGET_FILE:archive_codec>"
        "$<TARGET_FILE:blast_source>")
    # 77: Control Flow Guard enforced for the pe_map runner, whose blast longjmp cases then cannot run (#1293).
    set_tests_properties(cpu_archive_texture_publication PROPERTIES SKIP_RETURN_CODE 77)

    # The same reader image drawing each completed record through D3D12 (#830). The GPU codec keeps
    # the name archive_codec.dll in its own directory, next to copies of the reader and blast.
    set(archive_gpu_dir "${CMAKE_CURRENT_BINARY_DIR}/archive_gpu")
    # The native reader runs from this folder; give it the same app-local WARP as the lifted EXE.
    offscreen_app_local_warp("${OFFSCREEN_WARP_DIR}" "${archive_gpu_dir}")
    set(archive_gpu_sources "${archive_src}/codec.cpp" "${archive_src}/texture_gpu.cpp"
        "${archive_src}/quad_frames.cpp" "${CMAKE_CURRENT_BINARY_DIR}/tex_quad_vs.h"
        "${CMAKE_CURRENT_BINARY_DIR}/tex_quad_ps.h")
    add_library(archive_codec_gpu SHARED ${archive_gpu_sources})
    target_link_libraries(archive_codec_gpu PRIVATE blast_source dxgi d3d12)
    set_target_properties(archive_codec_gpu PROPERTIES OUTPUT_NAME archive_codec
        RUNTIME_OUTPUT_DIRECTORY "${archive_gpu_dir}" ARCHIVE_OUTPUT_DIRECTORY "${archive_gpu_dir}")
    add_library(archive_codec_gpu_lifted SHARED ${archive_gpu_sources})
    target_link_libraries(archive_codec_gpu_lifted PRIVATE blast_lifted dxgi d3d12)
    foreach(codec IN ITEMS archive_codec_gpu archive_codec_gpu_lifted)
        target_include_directories(${codec} PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
        target_compile_features(${codec} PRIVATE cxx_std_17)
        set_target_properties(${codec} PROPERTIES MSVC_RUNTIME_LIBRARY MultiThreadedDLL)
    endforeach()
    set(archive_gpu_reader "${archive_gpu_dir}/archive_texture_reader.exe")
    add_custom_command(OUTPUT "${archive_gpu_reader}" "${archive_gpu_dir}/blast_source.dll"
        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${texture_exe}" "$<TARGET_FILE:blast_source>"
            "${archive_gpu_dir}"
        DEPENDS "${texture_exe}" blast_source VERBATIM)
    add_custom_target(archive_gpu_reader ALL DEPENDS "${archive_gpu_reader}")
    add_executable(archive_texture_gpu_lifted "${texture_cpp}")
    target_link_libraries(archive_texture_gpu_lifted PRIVATE archive_codec_gpu_lifted ntdll)
    foreach(adapter IN ITEMS warp hardware)
        add_test(NAME cpu_archive_texture_gpu_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/archive_texture_gpu_check.py" "$<TARGET_FILE:pe_map>"
            "${archive_gpu_reader}" "$<TARGET_FILE:archive_texture_gpu_lifted>"
            "$<TARGET_FILE:archive_codec_gpu>" "$<TARGET_FILE:blast_source>")
        # 77: no physical adapter (hardware), or Control Flow Guard enforced for the pe_map runner (#1293).
        set_tests_properties(cpu_archive_texture_gpu_${adapter} PROPERTIES TIMEOUT 180 SKIP_RETURN_CODE 77
            ENVIRONMENT "X2W_SHIM_DRAW_ADAPTER=${adapter}")
    endforeach()

    add_library(archive_directory_gpu SHARED "${archive_src}/directory_gpu.cpp" "${archive_src}/quad_frames.cpp"
        "${CMAKE_CURRENT_BINARY_DIR}/tex_quad_vs.h" "${CMAKE_CURRENT_BINARY_DIR}/tex_quad_ps.h")
    target_include_directories(archive_directory_gpu PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
    target_compile_features(archive_directory_gpu PRIVATE cxx_std_17)
    target_link_libraries(archive_directory_gpu PRIVATE startup_stub_xcrdapi dxgi d3d12)
    set(directory_exe "${CMAKE_CURRENT_BINARY_DIR}/archive_directory_reader.exe")
    set(directory_cpp "${CMAKE_CURRENT_BINARY_DIR}/archive_directory_lifted.cpp")
    build_archive_reader("${archive_src}/directory_reader.zig" "${directory_exe}" "${directory_cpp}"
        archive_directory_gpu translate_directory.py EXPORT_SYMBOLS)
    add_executable(archive_directory_lifted "${directory_cpp}")
    target_link_libraries(archive_directory_lifted PRIVATE archive_directory_gpu ntdll)
    foreach(adapter IN ITEMS warp hardware)
        add_test(NAME cpu_archive_directory_gpu_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/archive_directory_gpu_check.py" "$<TARGET_FILE:pe_map>"
            "${directory_exe}" "$<TARGET_FILE:archive_directory_lifted>" "$<TARGET_FILE:archive_directory_gpu>")
        add_test(NAME cpu_archive_sixteen_page_gpu_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/cpu/archive_directory_multipage_check.py" "$<TARGET_FILE:pe_map>"
            "${directory_exe}" "$<TARGET_FILE:archive_directory_lifted>" "$<TARGET_FILE:archive_directory_gpu>")
        set_tests_properties(cpu_archive_directory_gpu_${adapter} cpu_archive_sixteen_page_gpu_${adapter} PROPERTIES TIMEOUT 90
            ENVIRONMENT "X2W_SHIM_DRAW_ADAPTER=${adapter}")
    endforeach()
    set_tests_properties(cpu_archive_directory_gpu_hardware cpu_archive_sixteen_page_gpu_hardware PROPERTIES SKIP_RETURN_CODE 77)

    function(crc_line_program target polynomial)
        # Use the same CRC object optimization flags in Debug and Release.
        set(crc_object "${CMAKE_CURRENT_BINARY_DIR}/${target}_step.obj")
        add_custom_command(OUTPUT "${crc_object}"
            COMMAND "${CMAKE_CXX_COMPILER}" /nologo /c /O1 /GS- /Zl "/DCRC_POLYNOMIAL=${polynomial}" "/Fo${crc_object}"
                "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/crc_line/crc_step.cpp"
            DEPENDS tests/cpu/crc_line/crc_step.cpp VERBATIM)
        add_executable(${target} tests/cpu/crc_line/entry.asm tests/cpu/exe_startup.asm "${crc_object}")
        set_target_properties(${target} PROPERTIES LINKER_LANGUAGE CXX)
        target_link_options(${target} PRIVATE /ENTRY:native_startup /EXPORT:native_entry /EXPORT:crc_step
            /SUBSYSTEM:CONSOLE /NODEFAULTLIB /INCREMENTAL:NO)
        target_link_libraries(${target} PRIVATE ucrt cpu_exe_io)
        set(crc_source "${CMAKE_CURRENT_BINARY_DIR}/${target}_lifted.cpp")
        add_custom_command(OUTPUT "${crc_source}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tools/cpu_stdio_exe_aot.py"
                "$<TARGET_FILE:${target}>" "${crc_source}" --entry-export native_entry
            DEPENDS ${target} tools/cpu_stdio_exe_aot.py tools/cpu_switch_aot.py tools/cpu_leaf_aot.py tools/cpu_ranges.py
            VERBATIM)
        add_executable(${target}_lifted "${crc_source}")
        set_target_properties(${target}_lifted PROPERTIES MSVC_RUNTIME_LIBRARY MultiThreadedDLL)
    endfunction()
    crc_line_program(cpu_crc_line 0xedb88320u)
    crc_line_program(cpu_crc_castagnoli 0x82f63b78u)
    add_test(NAME cpu_native_crc_line COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/crc_line_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_exe_io>" "$<TARGET_FILE:cpu_crc_line>" "$<TARGET_FILE:cpu_crc_line_lifted>"
        "$<TARGET_FILE:cpu_crc_castagnoli>" "$<TARGET_FILE:cpu_crc_castagnoli_lifted>")
    # One-command conversion: tools/exe_aot.py translates and compiles each EXE at test time.
    add_test(NAME cpu_exe_aot_convert COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/exe_aot_check.py" "${CMAKE_CXX_COMPILER}"
        "$<TARGET_FILE:cpu_crc_line>" "$<TARGET_FILE:cpu_crc_castagnoli>" "$<TARGET_FILE:cpu_exe_original>"
        "$<TARGET_FILE:cpu_exe_io>")
    set_tests_properties(cpu_exe_aot_convert PROPERTIES TIMEOUT 300)
    add_library(cpu_exe_second_provider SHARED tests/cpu/exe_second_provider.cpp)
    target_link_libraries(cpu_exe_second_provider PRIVATE cpu_exe_io)
    foreach(variant IN ITEMS original inverted pair swapped framed image two_providers nested nested_bypass)
        set(native cpu_exe_${variant})
        set(lifted ${native}_lifted)
        add_executable(${native} tests/cpu/exe_entry.asm)
        if(variant STREQUAL "two_providers")
            target_sources(${native} PRIVATE tests/cpu/repeat_call.asm)
            target_compile_definitions(${native} PRIVATE TWO_PROVIDERS IMPORT_ACCUMULATE)
        else()
            target_sources(${native} PRIVATE tests/cpu/direct.asm)
        endif()
        set(entry_options)
        set(image_options)
        if(variant STREQUAL "image")
            target_compile_definitions(${native} PRIVATE IMAGE_SELECTOR)
            target_link_options(${native} PRIVATE /EXPORT:selector_result,DATA /EXPORT:selector_bias,DATA)
            list(APPEND image_options --image-data)
        endif()
        if(variant STREQUAL "framed" OR variant STREQUAL "image")
            target_sources(${native} PRIVATE tests/cpu/bare_calls.asm)
            target_compile_definitions(${native} PRIVATE FRAME_SELECTOR)
            target_link_options(${native} PRIVATE /EXPORT:call_bare "/MAP:${CMAKE_CURRENT_BINARY_DIR}/${native}.map")
            list(APPEND entry_options --frame-selector)
        endif()
        if(variant STREQUAL "pair" OR variant STREQUAL "swapped")
            target_sources(${native} PRIVATE tests/cpu/alt.asm)
            target_compile_definitions(${native} PRIVATE TWO_TARGETS)
        endif()
        if(variant STREQUAL "swapped")
            target_compile_definitions(${native} PRIVATE SWAP_TARGETS)
        endif()
        set_target_properties(${native} PROPERTIES LINKER_LANGUAGE CXX)
        target_compile_definitions(${native} PRIVATE IMPORT_MIX)
        if(variant MATCHES "^nested(_bypass)?$")
            target_sources(${native} PRIVATE tests/cpu/exe_nested.asm)
            target_compile_definitions(${native} PRIVATE NESTED_TARGET)
        endif()
        if(variant STREQUAL "nested_bypass")
            target_compile_definitions(${native} PRIVATE BYPASS_WRAPPER)
        endif()
        if(variant STREQUAL "inverted")
            target_compile_definitions(${native} PRIVATE FLIP_SELECTOR)
        endif()
        target_sources(${native} PRIVATE tests/cpu/exe_startup.asm)
        target_link_options(${native} PRIVATE /ENTRY:native_startup /EXPORT:native_entry /NODEFAULTLIB /INCREMENTAL:NO)
        set(extra_provider_args)
        set(extra_provider_target)
        if(variant STREQUAL "two_providers")
            set(extra_provider_args --provider "$<TARGET_FILE:cpu_exe_second_provider>" accumulate_args)
            set(extra_provider_target cpu_exe_second_provider)
        endif()
        set(source "${CMAKE_CURRENT_BINARY_DIR}/${lifted}.cpp")
        add_custom_command(OUTPUT "${source}"
            COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/cpu_exe_aot.py"
                "$<TARGET_FILE:${native}>" "$<TARGET_FILE:cpu_exe_io>"
                "$<TARGET_FILE:cpu_import_provider>" "${source}" --entry-export native_entry ${entry_options} ${extra_provider_args}
            DEPENDS ${native} cpu_exe_io cpu_import_provider ${extra_provider_target} tools/cpu_exe_aot.py
                tools/cpu_call_aot.py tools/cpu_leaf_aot.py tools/cpu_ranges.py tools/ffx_version_query.py
            VERBATIM)
        add_executable(${lifted} "${source}")
        if(variant STREQUAL "image")
            set_source_files_properties("${source}" PROPERTIES COMPILE_DEFINITIONS "main=translated_entry")
            target_sources(${lifted} PRIVATE tests/cpu/exe_image_host.cpp)
            target_link_libraries(${lifted} PRIVATE pe_map)
        endif()
        if(variant STREQUAL "framed" OR variant STREQUAL "image")
            set(ranges "${CMAKE_CURRENT_BINARY_DIR}/${native}_ranges.json")
            set(selector "${CMAKE_CURRENT_BINARY_DIR}/${native}_selector.cpp")
            add_custom_command(OUTPUT "${selector}" BYPRODUCTS "${ranges}"
                COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/range_manifest.py"
                    "$<TARGET_FILE:${native}>" "${CMAKE_CURRENT_BINARY_DIR}/${native}.map" "${ranges}" bare_mix
                COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tools/cpu_switch_aot.py"
                    "$<TARGET_FILE:${native}>" call_bare "${selector}" --bind-ranges "${ranges}" ${image_options}
                DEPENDS ${native} tools/range_manifest.py tools/cpu_switch_aot.py tools/cpu_leaf_aot.py tools/cpu_ranges.py
                VERBATIM)
            target_sources(${lifted} PRIVATE "${selector}")
        endif()
        target_link_options(${lifted} PRIVATE /INCREMENTAL:NO)
        foreach(program IN ITEMS ${native} ${lifted})
            target_link_libraries(${program} PRIVATE cpu_exe_io cpu_import_provider ${extra_provider_target})
        endforeach()
    endforeach()
    add_test(NAME cpu_aot_exe_entry COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/exe_entry_check.py"
        "$<TARGET_FILE:cpu_exe_original>" "$<TARGET_FILE:cpu_exe_original_lifted>"
        "$<TARGET_FILE:cpu_exe_inverted>" "$<TARGET_FILE:cpu_exe_inverted_lifted>"
        "$<TARGET_FILE:cpu_exe_io>" "$<TARGET_FILE:cpu_import_provider>")
    add_test(NAME cpu_aot_exe_call_targets COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/exe_entry_check.py" --targets
        "$<TARGET_FILE:cpu_exe_pair>" "$<TARGET_FILE:cpu_exe_pair_lifted>"
        "$<TARGET_FILE:cpu_exe_swapped>" "$<TARGET_FILE:cpu_exe_swapped_lifted>")
    add_test(NAME cpu_aot_exe_nested_calls COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/exe_entry_check.py" --nested
        "$<TARGET_FILE:cpu_exe_nested>" "$<TARGET_FILE:cpu_exe_nested_lifted>"
        "$<TARGET_FILE:cpu_exe_nested_bypass>" "$<TARGET_FILE:cpu_exe_nested_bypass_lifted>")
    add_test(NAME cpu_aot_ordinal_import COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/import_check.py" "$<TARGET_FILE:cpu_ordinal_original>"
        "$<TARGET_FILE:cpu_ordinal_original_lifted>" "$<TARGET_FILE:cpu_ordinal_provider>"
        "$<TARGET_FILE:cpu_ordinal_altered>" "$<TARGET_FILE:cpu_register_probe>"
        --ordinal-control "$<TARGET_FILE:cpu_import_provider>")
    cpu_asm_fixture(cpu_reloc_original reloc_route)
    target_link_libraries(cpu_reloc_original PRIVATE cpu_import_provider)
    add_test(NAME cpu_native_highlow COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/highlow_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_reloc_original>" "$<TARGET_FILE:cpu_import_provider>")
    cpu_asm_fixture(cpu_two_imports two_imports)
    target_link_libraries(cpu_two_imports PRIVATE cpu_import_provider cpu_register_probe)
    add_library(pe_map SHARED tools/pe_map.cpp)
    target_link_libraries(pe_map PRIVATE bcrypt)
    add_library(cpu_shim_startup_guest SHARED tests/cpu/shim_startup.cpp)
    target_compile_features(cpu_shim_startup_guest PRIVATE cxx_std_17)
    set_target_properties(cpu_shim_startup_guest PROPERTIES MSVC_RUNTIME_LIBRARY MultiThreaded)
    target_link_libraries(cpu_shim_startup_guest PRIVATE xmem_shim d3d12_xs_shim)
    add_library(cpu_shim_compute SHARED tests/cpu/shim_compute_provider.cpp tests/cpu/shim_compute_gpu.cpp)
    add_dependencies(cpu_shim_compute pso_compute_headers)
    target_include_directories(cpu_shim_compute PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
    target_compile_features(cpu_shim_compute PRIVATE cxx_std_17)
    target_link_libraries(cpu_shim_compute PRIVATE xmem_shim dxgi d3d12)
    set(tex_tables "${CMAKE_CURRENT_BINARY_DIR}/tex_tables.h")
    add_custom_command(OUTPUT "${tex_tables}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/tex_gen.py" "${tex_tables}"
        DEPENDS tests/cpu/tex_gen.py tests/bc7_draw_check.py tools/bc7_blocks.py tools/bc7_tiles.py VERBATIM)
    add_library(cpu_shim_draw SHARED tests/cpu/shim_draw_provider.cpp tests/cpu/shim_draw_gpu.cpp
        tests/cpu/shim_chain_gpu.cpp tests/cpu/shim_tex_gpu.cpp tests/desktop_gpu.cpp
        tests/mesh_shadow_draw.cpp)
    set(shim_draw_color_header "${CMAKE_CURRENT_BINARY_DIR}/shim_draw_color.h")
    add_custom_command(OUTPUT "${shim_draw_color_header}"
        COMMAND "${DXC_EXECUTABLE}" -T ps_6_0 -E main -Fh "${shim_draw_color_header}" -Vn shim_draw_color
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/shim_draw_color.hlsl"
        DEPENDS tests/cpu/shim_draw_color.hlsl VERBATIM)
    foreach(stage IN ITEMS texture_init texture_read texture_swap)
        set(profile ps_6_0)
        set(definition)
        if(stage STREQUAL texture_init)
            set(profile cs_6_0)
            set(definition -DINITIALIZE_TEXTURE)
        elseif(stage STREQUAL texture_swap)
            set(definition -DSWAP_CHANNELS)
        endif()
        set(header "${CMAKE_CURRENT_BINARY_DIR}/shim_draw_${stage}.h")
        add_custom_command(OUTPUT "${header}"
            COMMAND "${DXC_EXECUTABLE}" -T "${profile}" -E main ${definition}
                -Fh "${header}" -Vn "shim_draw_${stage}" "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/shim_draw_texture.hlsl"
            DEPENDS tests/cpu/shim_draw_texture.hlsl VERBATIM)
        target_sources(cpu_shim_draw PRIVATE "${header}")
    endforeach()
    add_dependencies(cpu_shim_draw triangle_headers)
    target_sources(cpu_shim_draw PRIVATE
        "${shim_draw_color_header}"
        "${CMAKE_CURRENT_BINARY_DIR}/triangle_vs.h"
        "${CMAKE_CURRENT_BINARY_DIR}/triangle_ps.h"
        "${CMAKE_CURRENT_BINARY_DIR}/triangle_vs_flip.h"
        "${CMAKE_CURRENT_BINARY_DIR}/triangle_ps_flip.h"
        "${CMAKE_CURRENT_BINARY_DIR}/tex_quad_vs.h"
        "${CMAKE_CURRENT_BINARY_DIR}/tex_quad_ps.h"
        "${tex_tables}")
    target_include_directories(cpu_shim_draw PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
    target_compile_features(cpu_shim_draw PRIVATE cxx_std_17)
    target_link_libraries(cpu_shim_draw PRIVATE dxgi d3d12)
    add_executable(cpu_shim_startup tests/cpu/shim_startup.asm tests/cpu/exe_startup.asm)
    set_target_properties(cpu_shim_startup PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_shim_startup PRIVATE /ENTRY:native_startup /SUBSYSTEM:CONSOLE /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(cpu_shim_startup PRIVATE cpu_shim_startup_guest cpu_exe_io cpu_shim_compute cpu_shim_draw d3d12_xs_shim)
    set(shim_startup_cpp "${CMAKE_CURRENT_BINARY_DIR}/cpu_shim_startup_lifted.cpp")
    add_custom_command(OUTPUT "${shim_startup_cpp}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/translate_shim_startup.py"
            "$<TARGET_FILE:cpu_shim_startup>" "${shim_startup_cpp}"
        DEPENDS cpu_shim_startup tests/cpu/translate_shim_startup.py tools/cpu_switch_aot.py
            tools/cpu_call_aot.py tools/cpu_leaf_aot.py tools/cpu_ranges.py VERBATIM)
    add_library(cpu_placed_dxr_provider SHARED tests/cpu/placed_dxr_provider.cpp tests/dxr_check.cpp
        tests/desktop_gpu.cpp src/era_title_va.cpp src/startup_stubs/d3d12_xs_device.cpp
        "${ray_header}" "${ray_pipeline_header}" "${candidates_header}" "${candidates_pipeline_header}")
    target_include_directories(cpu_placed_dxr_provider PRIVATE tests src "${CMAKE_CURRENT_BINARY_DIR}")
    target_link_libraries(cpu_placed_dxr_provider PRIVATE d3d12 dxgi dxguid d3dcompiler user32 gdi32)
    target_compile_features(cpu_placed_dxr_provider PRIVATE cxx_std_17)
    add_executable(cpu_placed_dxr tests/cpu/placed_dxr.asm tests/cpu/exe_startup.asm)
    set_target_properties(cpu_placed_dxr PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_placed_dxr PRIVATE /ENTRY:native_startup /SUBSYSTEM:CONSOLE /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(cpu_placed_dxr PRIVATE cpu_placed_dxr_provider cpu_exe_io)
    set(placed_dxr_cpp "${CMAKE_CURRENT_BINARY_DIR}/cpu_placed_dxr_lifted.cpp")
    add_custom_command(OUTPUT "${placed_dxr_cpp}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/translate_placed_dxr.py"
            "$<TARGET_FILE:cpu_placed_dxr>" "${placed_dxr_cpp}"
        DEPENDS cpu_placed_dxr tests/cpu/translate_placed_dxr.py tests/cpu/translate_shim_startup.py tools/cpu_switch_aot.py)
    add_executable(cpu_placed_dxr_lifted "${placed_dxr_cpp}")
    target_compile_features(cpu_placed_dxr_lifted PRIVATE cxx_std_17)
    target_link_libraries(cpu_placed_dxr_lifted PRIVATE cpu_placed_dxr_provider cpu_exe_io)
    foreach(adapter IN ITEMS warp hardware)
        add_test(NAME cpu_placed_dxr_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/placed_dxr_check.py" ${adapter} "$<TARGET_FILE:pe_map>"
            "$<TARGET_FILE:cpu_placed_dxr>" "$<TARGET_FILE:cpu_placed_dxr_lifted>"
            "$<TARGET_FILE:cpu_placed_dxr_provider>" "$<TARGET_FILE:cpu_exe_io>")
        set_tests_properties(cpu_placed_dxr_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77)
    endforeach()
    add_library(cpu_placed_provider SHARED tests/cpu/placed_draw_provider.cpp tests/startup_stream_setup.cpp)
    target_compile_features(cpu_placed_provider PRIVATE cxx_std_17)
    target_link_libraries(cpu_placed_provider PRIVATE d3d12_xs_shim d3d12 dxgi)
    add_executable(cpu_placed_draw tests/cpu/placed_draw.asm tests/cpu/exe_startup.asm)
    set_target_properties(cpu_placed_draw PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_placed_draw PRIVATE /ENTRY:native_startup /SUBSYSTEM:CONSOLE /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(cpu_placed_draw PRIVATE cpu_placed_provider cpu_exe_io)
    set(placed_cpp "${CMAKE_CURRENT_BINARY_DIR}/cpu_placed_draw_lifted.cpp")
    add_custom_command(OUTPUT "${placed_cpp}"
        COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/translate_placed_draw.py"
            "$<TARGET_FILE:cpu_placed_draw>" "${placed_cpp}"
        DEPENDS cpu_placed_draw tests/cpu/translate_placed_draw.py tests/cpu/translate_shim_startup.py tools/cpu_switch_aot.py)
    add_executable(cpu_placed_draw_lifted "${placed_cpp}")
    target_compile_features(cpu_placed_draw_lifted PRIVATE cxx_std_17)
    target_link_libraries(cpu_placed_draw_lifted PRIVATE cpu_placed_provider cpu_exe_io)
    foreach(adapter IN ITEMS warp hardware)
        add_test(NAME cpu_placed_draw_${adapter} COMMAND "${CMAKE_COMMAND}" -E env "X2W_FXC=${STARTUP_FXC}"
            "${INTAKE_PYTHON}" -B "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/placed_draw_check.py" "${adapter}"
            "$<TARGET_FILE:startup_stream_consumer>" "$<TARGET_FILE:pe_map>" "$<TARGET_FILE:cpu_placed_draw>"
            "$<TARGET_FILE:cpu_placed_draw_lifted>" "$<TARGET_FILE:cpu_placed_provider>" "$<TARGET_FILE:cpu_exe_io>")
        set_tests_properties(cpu_placed_draw_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77)
    endforeach()
    add_executable(cpu_shim_startup_lifted "${shim_startup_cpp}")
    target_compile_features(cpu_shim_startup_lifted PRIVATE cxx_std_17)
    target_link_libraries(cpu_shim_startup_lifted PRIVATE cpu_shim_startup_guest cpu_exe_io cpu_shim_compute cpu_shim_draw d3d12_xs_shim)
    add_test(NAME cpu_native_shim_startup COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/shim_startup_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_shim_startup>" "$<TARGET_FILE:cpu_shim_startup_guest>" "$<TARGET_FILE:cpu_exe_io>"
        "$<TARGET_FILE:xmem_shim>" "$<TARGET_FILE:d3d12_xs_shim>")
    set_tests_properties(cpu_native_shim_startup PROPERTIES TIMEOUT 100
        ENVIRONMENT "X2W_SHIM_COMPUTE_CASE=;X2W_SHIM_DRAW_CASE=")
    # Recording stand-ins for the main executable's D3D12ConfigureVirtualMemoryX call (#874).
    # They keep the d3d12_xs.dll and xmem.dll names in their own directory, next to the fixture EXEs.
    set(configure_vm_dir "${CMAKE_CURRENT_BINARY_DIR}/configure_vm")
    foreach(provider IN ITEMS d3d12_xs xmem)
        add_library(cpu_configure_vm_${provider} SHARED tests/cpu/configure_vm_recorder.cpp
            tests/cpu/configure_vm_${provider}.def)
        set_target_properties(cpu_configure_vm_${provider} PROPERTIES OUTPUT_NAME ${provider}
            MSVC_RUNTIME_LIBRARY MultiThreaded
            RUNTIME_OUTPUT_DIRECTORY "${configure_vm_dir}" ARCHIVE_OUTPUT_DIRECTORY "${configure_vm_dir}")
    endforeach()
    set(configure_vm_io "${configure_vm_dir}/cpu_exe_io.dll")
    add_custom_command(OUTPUT "${configure_vm_io}"
        COMMAND "${CMAKE_COMMAND}" -E make_directory "${configure_vm_dir}"
        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "$<TARGET_FILE:cpu_exe_io>" "${configure_vm_io}"
        DEPENDS cpu_exe_io VERBATIM)
    add_custom_target(cpu_configure_vm_io DEPENDS "${configure_vm_io}")
    foreach(sample IN ITEMS 3 4)
        add_executable(cpu_configure_vm_${sample} tests/cpu/configure_vm.asm tests/cpu/exe_startup.asm)
        add_dependencies(cpu_configure_vm_${sample} cpu_configure_vm_io)
        target_compile_definitions(cpu_configure_vm_${sample} PRIVATE SAMPLE=${sample})
        set_target_properties(cpu_configure_vm_${sample} PROPERTIES LINKER_LANGUAGE CXX
            RUNTIME_OUTPUT_DIRECTORY "${configure_vm_dir}")
        target_link_options(cpu_configure_vm_${sample} PRIVATE
            /ENTRY:native_startup /SUBSYSTEM:CONSOLE /NODEFAULTLIB /INCREMENTAL:NO)
        target_link_libraries(cpu_configure_vm_${sample} PRIVATE cpu_configure_vm_d3d12_xs cpu_configure_vm_xmem cpu_exe_io)
    endforeach()
    add_test(NAME cpu_native_configure_vm COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/configure_vm_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_configure_vm_3>" "$<TARGET_FILE:cpu_configure_vm_4>")
    foreach(adapter IN ITEMS warp hardware)
        add_test(NAME cpu_shim_compute_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/shim_compute_check.py"
            "$<TARGET_FILE:pe_map>" "$<TARGET_FILE:cpu_shim_startup>" "$<TARGET_FILE:cpu_shim_startup_lifted>"
            "$<TARGET_FILE:cpu_shim_startup_guest>" "$<TARGET_FILE:cpu_exe_io>" "$<TARGET_FILE:xmem_shim>"
            "$<TARGET_FILE:d3d12_xs_shim>")
        set_tests_properties(cpu_shim_compute_${adapter} PROPERTIES TIMEOUT 120
            ENVIRONMENT "X2W_SHIM_COMPUTE_ADAPTER=${adapter};X2W_SHIM_DRAW_CASE=")
    endforeach()
    set_tests_properties(cpu_shim_compute_hardware PROPERTIES SKIP_RETURN_CODE 77)
    foreach(adapter IN ITEMS warp hardware)
        add_test(NAME cpu_shim_draw_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/shim_draw_check.py"
            "$<TARGET_FILE:pe_map>" "$<TARGET_FILE:cpu_shim_startup>" "$<TARGET_FILE:cpu_shim_startup_lifted>"
            "$<TARGET_FILE:cpu_shim_startup_guest>" "$<TARGET_FILE:cpu_exe_io>" "$<TARGET_FILE:xmem_shim>"
            "$<TARGET_FILE:d3d12_xs_shim>" "$<TARGET_FILE:cpu_shim_draw>")
        set_tests_properties(cpu_shim_draw_${adapter} PROPERTIES TIMEOUT 180
            ENVIRONMENT "X2W_SHIM_DRAW_ADAPTER=${adapter};X2W_SHIM_COMPUTE_CASE=")
        add_test(NAME cpu_shim_texture_reuse_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/shim_draw_check.py"
            "$<TARGET_FILE:pe_map>" "$<TARGET_FILE:cpu_shim_startup>" "$<TARGET_FILE:cpu_shim_startup_lifted>"
            "$<TARGET_FILE:cpu_shim_startup_guest>" "$<TARGET_FILE:cpu_exe_io>" "$<TARGET_FILE:xmem_shim>"
            "$<TARGET_FILE:d3d12_xs_shim>" "$<TARGET_FILE:cpu_shim_draw>" --texture-reuse)
        set_tests_properties(cpu_shim_texture_reuse_${adapter} PROPERTIES TIMEOUT 180
            ENVIRONMENT "X2W_SHIM_DRAW_ADAPTER=${adapter};X2W_SHIM_COMPUTE_CASE=")
    endforeach()
    set_tests_properties(cpu_shim_draw_hardware cpu_shim_texture_reuse_hardware PROPERTIES SKIP_RETURN_CODE 77)
    # Uploaded RGBA8 and BC7 textures sampled through a vertex-buffer quad from the same EXE entry.
    foreach(adapter IN ITEMS warp hardware)
        add_test(NAME cpu_shim_upload_texture_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/shim_tex_check.py"
            "$<TARGET_FILE:pe_map>" "$<TARGET_FILE:cpu_shim_startup>" "$<TARGET_FILE:cpu_shim_startup_lifted>"
            "$<TARGET_FILE:cpu_shim_startup_guest>" "$<TARGET_FILE:cpu_exe_io>" "$<TARGET_FILE:xmem_shim>"
            "$<TARGET_FILE:d3d12_xs_shim>" "$<TARGET_FILE:cpu_shim_draw>")
        set_tests_properties(cpu_shim_upload_texture_${adapter} PROPERTIES TIMEOUT 180
            ENVIRONMENT "X2W_SHIM_DRAW_ADAPTER=${adapter};X2W_SHIM_COMPUTE_CASE=")
    endforeach()
    set_tests_properties(cpu_shim_upload_texture_hardware PROPERTIES SKIP_RETURN_CODE 77)
    # The translated compute -> mesh -> pixel chain (#658) from the same EXE entry (#671).
    foreach(adapter IN ITEMS warp hardware)
        add_test(NAME cpu_shim_chain_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/shim_chain_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:pe_map>" "$<TARGET_FILE:cpu_shim_startup>" "$<TARGET_FILE:cpu_shim_startup_lifted>"
            "$<TARGET_FILE:cpu_shim_startup_guest>" "$<TARGET_FILE:cpu_exe_io>" "$<TARGET_FILE:xmem_shim>"
            "$<TARGET_FILE:d3d12_xs_shim>" "$<TARGET_FILE:cpu_shim_draw>")
        set_tests_properties(cpu_shim_chain_${adapter} PROPERTIES TIMEOUT 180 SKIP_RETURN_CODE 77
            ENVIRONMENT "X2W_SHIM_DRAW_ADAPTER=${adapter};X2W_SHIM_COMPUTE_CASE=")
        add_test(NAME cpu_shim_shadow_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/cpu/shim_shadow_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:pe_map>" "$<TARGET_FILE:cpu_shim_startup>" "$<TARGET_FILE:cpu_shim_startup_lifted>"
            "$<TARGET_FILE:cpu_shim_startup_guest>" "$<TARGET_FILE:cpu_exe_io>" "$<TARGET_FILE:xmem_shim>"
            "$<TARGET_FILE:d3d12_xs_shim>" "$<TARGET_FILE:cpu_shim_draw>")
        set_tests_properties(cpu_shim_shadow_${adapter} PROPERTIES TIMEOUT 180 SKIP_RETURN_CODE 77
            ENVIRONMENT "X2W_SHIM_DRAW_ADAPTER=${adapter};X2W_SHIM_COMPUTE_CASE=")
    endforeach()

    add_test(NAME cpu_native_exe_chain COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/mapped_exe_chain_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_exe_original>" "$<TARGET_FILE:cpu_exe_original_lifted>"
        "$<TARGET_FILE:cpu_exe_io>" "$<TARGET_FILE:cpu_import_provider>")
    add_test(NAME cpu_native_exe_frame_chain COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/mapped_exe_chain_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_exe_framed>" "$<TARGET_FILE:cpu_exe_framed_lifted>"
        "$<TARGET_FILE:cpu_exe_io>" "$<TARGET_FILE:cpu_import_provider>" "$<TARGET_FILE:cpu_exe_original>")
    add_test(NAME cpu_native_exe_image_chain COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/exe_image_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_exe_image>" "$<TARGET_FILE:cpu_exe_image_lifted>"
        "$<TARGET_FILE:cpu_exe_io>" "$<TARGET_FILE:cpu_import_provider>")
    add_test(NAME cpu_native_exe_two_providers COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/mapped_exe_chain_check.py" --two-providers "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_exe_two_providers>" "$<TARGET_FILE:cpu_exe_two_providers_lifted>"
        "$<TARGET_FILE:cpu_exe_io>" "$<TARGET_FILE:cpu_import_provider>" "$<TARGET_FILE:cpu_exe_second_provider>")
    target_compile_features(pe_map PRIVATE cxx_std_17)
    add_test(NAME cpu_native_exe_termination COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/exe_termination_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_exe_original>" "$<TARGET_FILE:cpu_exe_original_lifted>"
        "$<TARGET_FILE:cpu_exe_io>" "$<TARGET_FILE:cpu_import_provider>")
    add_test(NAME cpu_native_map COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/native_map_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_register_probe>"
        --named "$<TARGET_FILE:cpu_import_original>" "$<TARGET_FILE:cpu_import_original_lifted>"
            "$<TARGET_FILE:cpu_import_provider>" "$<TARGET_FILE:cpu_import_altered>"
        --ordinal "$<TARGET_FILE:cpu_ordinal_original>" "$<TARGET_FILE:cpu_ordinal_original_lifted>"
            "$<TARGET_FILE:cpu_ordinal_provider>" "$<TARGET_FILE:cpu_ordinal_altered>"
        --reloc "$<TARGET_FILE:cpu_reloc_original>" "$<TARGET_FILE:cpu_import_provider>"
            "$<TARGET_FILE:cpu_import_altered>"
        --two "$<TARGET_FILE:cpu_two_imports>" "$<TARGET_FILE:cpu_import_provider>"
            "$<TARGET_FILE:cpu_register_probe>")
    add_test(NAME cpu_native_mutation COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/native_mutation_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_register_probe>" "$<TARGET_FILE:cpu_import_original>"
        --provider "$<TARGET_FILE:cpu_import_provider>")
    add_test(NAME cpu_native_reloc_free COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/native_reloc_free_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_register_probe>" "$<TARGET_FILE:cpu_import_original>"
        "$<TARGET_FILE:cpu_import_provider>")
    cpu_asm_fixture(cpu_forwarder forwarder)
    target_link_libraries(cpu_forwarder PRIVATE cpu_import_provider)
    add_test(NAME cpu_native_forwarder COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/forwarder_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_forwarder>" "$<TARGET_FILE:cpu_import_provider>")
    cpu_asm_fixture(cpu_seh_original seh_route)
    add_test(NAME cpu_native_seh COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/native_seh_check.py" "$<TARGET_FILE:cpu_seh_original>"
        "$<TARGET_FILE:pe_map>" "$<TARGET_FILE:cpu_register_probe>")
    add_test(NAME cpu_native_section COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/native_section_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_register_probe>")
    # A load that never returns ends the test; its last "start" line names the case.
    set_tests_properties(cpu_native_section PROPERTIES TIMEOUT 120)
    add_library(cpu_tls_sink SHARED tests/cpu/tls_sink.cpp)
    foreach(variant IN ITEMS original refuse)
        add_library(cpu_tls_${variant} SHARED tests/cpu/tls_route.asm tests/cpu/tls_route.def)
        set_target_properties(cpu_tls_${variant} PROPERTIES LINKER_LANGUAGE CXX)
        target_link_options(cpu_tls_${variant} PRIVATE /ENTRY:dll_entry /NODEFAULTLIB /INCREMENTAL:NO)
        target_link_libraries(cpu_tls_${variant} PRIVATE cpu_tls_sink)
    endforeach()
    target_compile_definitions(cpu_tls_refuse PRIVATE ENTRY_FAILS)
    add_test(NAME cpu_native_tls COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/native_tls_check.py" "$<TARGET_FILE:cpu_tls_original>"
        "$<TARGET_FILE:cpu_tls_refuse>" "$<TARGET_FILE:cpu_tls_sink>" "$<TARGET_FILE:pe_map>")
    # Implicit TLS data: initialized, zero-filled, aligned, relocated and dynamically initialized
    # thread-locals seen from the loading thread, an existing thread and a new thread.
    foreach(variant IN ITEMS cpu_tls_data cpu_tls_data_refuse)
        add_library(${variant} SHARED tests/cpu/tls_data.cpp)
        set_target_properties(${variant} PROPERTIES MSVC_RUNTIME_LIBRARY MultiThreaded)
        target_link_options(${variant} PRIVATE /INCREMENTAL:NO)
    endforeach()
    target_compile_definitions(cpu_tls_data_refuse PRIVATE ENTRY_FAILS)
    add_test(NAME cpu_native_tls_data COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/native_tls_data_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_tls_data>" "$<TARGET_FILE:cpu_tls_data_refuse>")
    # Implicit-TLS differential: one __declspec(thread) value read in the TLS callback
    # on the main and worker threads, under the Windows loader and under pe_map.
    add_library(cpu_tls_value SHARED tests/cpu/tls_value.cpp)
    set_target_properties(cpu_tls_value PROPERTIES MSVC_RUNTIME_LIBRARY MultiThreaded)
    target_link_options(cpu_tls_value PRIVATE /INCREMENTAL:NO)
    target_link_libraries(cpu_tls_value PRIVATE cpu_tls_sink)
    add_test(NAME cpu_native_tls_value COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/native_tls_value_check.py" "$<TARGET_FILE:cpu_tls_value>"
        "$<TARGET_FILE:cpu_tls_original>" "$<TARGET_FILE:cpu_tls_sink>" "$<TARGET_FILE:pe_map>")
    # Provider variant that starts a ready worker inside mix_args; the control installs it as cpu_import_provider.dll.
    add_library(cpu_import_live SHARED tests/cpu/import_provider.cpp tests/cpu/named_provider.def)
    target_compile_definitions(cpu_import_live PRIVATE LIVE_WORKER)
    target_link_libraries(cpu_import_live PRIVATE cpu_exe_io)
    # Simulation control: finish_exe never terminates; the control installs it as cpu_exe_io.dll.
    add_library(cpu_exe_io_hang SHARED tests/cpu/exe_io.cpp)
    target_compile_definitions(cpu_exe_io_hang PRIVATE FINISH_HANG)
    add_library(cpu_cookie_dll SHARED tests/cpu/load_cookie.asm)
    add_executable(cpu_cookie_exe tests/cpu/load_cookie.asm tests/cpu/exe_startup.asm)
    set_target_properties(cpu_cookie_dll cpu_cookie_exe PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_cookie_dll PRIVATE /ENTRY:fixture_entry /EXPORT:read_cookie
        /NODEFAULTLIB /SUBSYSTEM:WINDOWS,6.00 /DYNAMICBASE /INCREMENTAL:NO)
    target_link_options(cpu_cookie_exe PRIVATE /ENTRY:native_startup
        /NODEFAULTLIB /SUBSYSTEM:CONSOLE,6.00 /DYNAMICBASE /INCREMENTAL:NO)
    target_link_libraries(cpu_cookie_exe PRIVATE cpu_exe_io)
    add_test(NAME cpu_native_load_cookie COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/load_cookie_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_cookie_dll>" "$<TARGET_FILE:cpu_cookie_exe>" "$<TARGET_FILE:cpu_exe_io>")
    add_test(NAME cpu_package_shim_lifetime COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/package_shim_lifetime_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_cookie_exe>" "$<TARGET_FILE:cpu_exe_io>")
    foreach(variant IN ITEMS present absent alternate)
        add_executable(cpu_load_dispatch_${variant} tests/cpu/load_config_dispatch.asm tests/cpu/exe_startup.asm)
        set_target_properties(cpu_load_dispatch_${variant} PROPERTIES LINKER_LANGUAGE CXX)
        target_link_options(cpu_load_dispatch_${variant} PRIVATE
            /ENTRY:native_startup /NODEFAULTLIB /SUBSYSTEM:CONSOLE /DYNAMICBASE /INCREMENTAL:NO)
        target_link_libraries(cpu_load_dispatch_${variant} PRIVATE cpu_exe_io)
        if(variant STREQUAL absent)
            target_compile_options(cpu_load_dispatch_${variant} PRIVATE /DDISPATCH_ABSENT)
        elseif(variant STREQUAL alternate)
            target_compile_options(cpu_load_dispatch_${variant} PRIVATE /DDISPATCH_ALTERNATE)
        endif()
    endforeach()
    add_test(NAME cpu_native_load_dispatch COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/load_config_dispatch_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_load_dispatch_present>" "$<TARGET_FILE:cpu_load_dispatch_absent>"
        "$<TARGET_FILE:cpu_load_dispatch_alternate>" "$<TARGET_FILE:cpu_exe_io>")
    set_tests_properties(cpu_native_load_dispatch PROPERTIES TIMEOUT 30)
    # A typed callback called through the linker-produced load-config dispatch slot (#903). The same
    # objects are linked without and with /guard:cf. OTHER is the negative control; INVALID calls a
    # target inside a function. Based on denn0x's pe/load-config-callback reference branch.
    foreach(variant IN ITEMS record other invalid cfg_record cfg_invalid)
        add_executable(cpu_load_callback_${variant} tests/cpu/load_config_callback.asm
            tests/cpu/load_config_callback.cpp tests/cpu/exe_startup.asm)
        set_target_properties(cpu_load_callback_${variant} PROPERTIES LINKER_LANGUAGE CXX)
        target_compile_options(cpu_load_callback_${variant} PRIVATE
            $<$<COMPILE_LANGUAGE:CXX>:/guard:cf /GS- /Zl>)
        target_link_options(cpu_load_callback_${variant} PRIVATE /ENTRY:native_startup
            /NODEFAULTLIB /SUBSYSTEM:CONSOLE /DYNAMICBASE /INCREMENTAL:NO)
        target_link_libraries(cpu_load_callback_${variant} PRIVATE cpu_exe_io)
        if(variant MATCHES "^cfg_")
            target_link_options(cpu_load_callback_${variant} PRIVATE /guard:cf)
        endif()
        if(variant MATCHES "other")
            target_compile_definitions(cpu_load_callback_${variant} PRIVATE OTHER)
        elseif(variant MATCHES "invalid")
            target_compile_definitions(cpu_load_callback_${variant} PRIVATE INVALID)
        endif()
    endforeach()
    add_test(NAME cpu_native_load_config_callback COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/load_config_callback_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_exe_io>" "$<TARGET_FILE:cpu_load_callback_record>"
        "$<TARGET_FILE:cpu_load_callback_other>" "$<TARGET_FILE:cpu_load_callback_invalid>"
        "$<TARGET_FILE:cpu_load_callback_cfg_record>" "$<TARGET_FILE:cpu_load_callback_cfg_invalid>")
    set_tests_properties(cpu_native_load_config_callback PROPERTIES TIMEOUT 120)
    foreach(variant IN ITEMS original no_tls)
        add_executable(cpu_exe_start_${variant} tests/cpu/exe_start.asm tests/cpu/exe_startup.asm)
        set_target_properties(cpu_exe_start_${variant} PROPERTIES LINKER_LANGUAGE CXX)
        target_link_options(cpu_exe_start_${variant} PRIVATE /ENTRY:native_startup /EXPORT:native_entry
            /EXPORT:body_status,DATA /NODEFAULTLIB /SUBSYSTEM:CONSOLE /DYNAMICBASE /INCREMENTAL:NO)
        target_link_libraries(cpu_exe_start_${variant} PRIVATE cpu_import_provider cpu_exe_io)
    endforeach()
    target_compile_definitions(cpu_exe_start_no_tls PRIVATE NO_TLS)
    add_test(NAME cpu_native_exe_start COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/native_exe_start_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_exe_start_original>" "$<TARGET_FILE:cpu_exe_start_no_tls>"
        "$<TARGET_FILE:cpu_import_provider>" "$<TARGET_FILE:cpu_import_altered>"
        "$<TARGET_FILE:cpu_exe_io>" "$<TARGET_FILE:cpu_import_live>" "$<TARGET_FILE:cpu_exe_io_hang>")
    # An EXE that imports XMemAllocatePhysicalPages and XMemFreePhysicalPages by name (#880), started by
    # Windows and by pe_map. FULL_COUNT and SKIP_RELEASE are negative controls.
    foreach(variant IN ITEMS original full_count skip_release)
        add_executable(cpu_xmem_physical_${variant} tests/cpu/xmem_physical_pair.asm tests/cpu/exe_startup.asm)
        set_target_properties(cpu_xmem_physical_${variant} PROPERTIES LINKER_LANGUAGE CXX)
        target_link_options(cpu_xmem_physical_${variant} PRIVATE /ENTRY:native_startup
            /NODEFAULTLIB /SUBSYSTEM:CONSOLE /DYNAMICBASE /INCREMENTAL:NO)
        target_link_libraries(cpu_xmem_physical_${variant} PRIVATE xmem_shim cpu_exe_io)
    endforeach()
    target_compile_definitions(cpu_xmem_physical_full_count PRIVATE FULL_COUNT)
    target_compile_definitions(cpu_xmem_physical_skip_release PRIVATE SKIP_RELEASE)
    add_test(NAME cpu_native_xmem_physical_pair COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/xmem_physical_pair_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_xmem_physical_original>" "$<TARGET_FILE:cpu_xmem_physical_full_count>"
        "$<TARGET_FILE:cpu_xmem_physical_skip_release>" "$<TARGET_FILE:xmem_shim>" "$<TARGET_FILE:cpu_exe_io>")
    set_tests_properties(cpu_native_xmem_physical_pair PROPERTIES TIMEOUT 120)
    # An EXE that imports XMemVirtualAlloc by name and uses the four reserve+commit startup call shapes
    # (#870), started by Windows and by pe_map. The six variants after original are negative controls.
    foreach(variant IN ITEMS original reserve_only misaligned dirty accepted_refusal host_fit skip_release)
        add_executable(cpu_xmem_pool_${variant} tests/cpu/xmem_pool_commit.asm tests/cpu/exe_startup.asm)
        set_target_properties(cpu_xmem_pool_${variant} PROPERTIES LINKER_LANGUAGE CXX)
        target_link_options(cpu_xmem_pool_${variant} PRIVATE /ENTRY:native_startup
            /NODEFAULTLIB /SUBSYSTEM:CONSOLE /DYNAMICBASE /INCREMENTAL:NO)
        target_link_libraries(cpu_xmem_pool_${variant} PRIVATE xmem_shim cpu_exe_io kernel32)
    endforeach()
    target_compile_definitions(cpu_xmem_pool_reserve_only PRIVATE RESERVE_ONLY)
    target_compile_definitions(cpu_xmem_pool_misaligned PRIVATE MISALIGNED)
    target_compile_definitions(cpu_xmem_pool_dirty PRIVATE DIRTY)
    target_compile_definitions(cpu_xmem_pool_accepted_refusal PRIVATE ACCEPTED_REFUSAL)
    target_compile_definitions(cpu_xmem_pool_host_fit PRIVATE HOST_FIT)
    target_compile_definitions(cpu_xmem_pool_skip_release PRIVATE SKIP_RELEASE)
    add_test(NAME cpu_native_xmem_pool_commit COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/xmem_pool_commit_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_xmem_pool_original>" "$<TARGET_FILE:cpu_xmem_pool_reserve_only>"
        "$<TARGET_FILE:cpu_xmem_pool_misaligned>"
        "$<TARGET_FILE:cpu_xmem_pool_dirty>" "$<TARGET_FILE:cpu_xmem_pool_accepted_refusal>"
        "$<TARGET_FILE:cpu_xmem_pool_host_fit>" "$<TARGET_FILE:cpu_xmem_pool_skip_release>"
        "$<TARGET_FILE:xmem_shim>" "$<TARGET_FILE:cpu_exe_io>")
    set_tests_properties(cpu_native_xmem_pool_commit PROPERTIES TIMEOUT 120)
    # An EXE that imports XMemAllocDefault and XMemFreeDefault by name (#875, exports from #879), started
    # by Windows and by pe_map. SKIP_GPU_FREE, SKIP_HEAP_FREE and UNINDUCED are negative controls.
    add_library(cpu_heap_probe SHARED tests/cpu/heap_probe.cpp)
    foreach(variant IN ITEMS original skip_gpu_free skip_heap_free uninduced)
        add_executable(cpu_xmem_default_${variant} tests/cpu/xmem_default_pair.asm tests/cpu/exe_startup.asm)
        set_target_properties(cpu_xmem_default_${variant} PROPERTIES LINKER_LANGUAGE CXX)
        target_link_options(cpu_xmem_default_${variant} PRIVATE /ENTRY:native_startup
            /NODEFAULTLIB /SUBSYSTEM:CONSOLE /DYNAMICBASE /INCREMENTAL:NO)
        target_link_libraries(cpu_xmem_default_${variant} PRIVATE xmem_shim cpu_exe_io cpu_heap_probe kernel32)
    endforeach()
    target_compile_definitions(cpu_xmem_default_skip_gpu_free PRIVATE SKIP_GPU_FREE)
    target_compile_definitions(cpu_xmem_default_skip_heap_free PRIVATE SKIP_HEAP_FREE)
    target_compile_definitions(cpu_xmem_default_uninduced PRIVATE UNINDUCED)
    add_test(NAME cpu_native_xmem_default_pair COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/xmem_default_pair_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_xmem_default_original>" "$<TARGET_FILE:cpu_xmem_default_skip_gpu_free>"
        "$<TARGET_FILE:cpu_xmem_default_skip_heap_free>" "$<TARGET_FILE:cpu_xmem_default_uninduced>"
        "$<TARGET_FILE:xmem_shim>" "$<TARGET_FILE:cpu_exe_io>" "$<TARGET_FILE:cpu_heap_probe>")
    set_tests_properties(cpu_native_xmem_default_pair PROPERTIES TIMEOUT 180)

    foreach(variant IN ITEMS original returning no_worker)
        add_executable(cpu_exe_worker_alive_${variant} tests/cpu/worker_alive_entry.asm)
        set_target_properties(cpu_exe_worker_alive_${variant} PROPERTIES LINKER_LANGUAGE CXX)
        target_link_options(cpu_exe_worker_alive_${variant} PRIVATE /ENTRY:worker_alive_entry
            /NODEFAULTLIB /SUBSYSTEM:CONSOLE /DYNAMICBASE /INCREMENTAL:NO)
        target_link_libraries(cpu_exe_worker_alive_${variant} PRIVATE kernel32)
    endforeach()
    # The worker decides whether a process outlives its entry thread; the return path stays the
    # variable so the fixtures can be compared without touching the shared entry.
    target_compile_definitions(cpu_exe_worker_alive_returning PRIVATE RETURN_FROM_ENTRY)
    target_compile_definitions(cpu_exe_worker_alive_no_worker PRIVATE RETURN_FROM_ENTRY NO_WORKER)
    add_test(NAME cpu_native_worker_alive COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/hang_probe.py" --control --launches 2 --timeout 2
        --eof-timeout 0.5
        "$<TARGET_FILE:cpu_exe_worker_alive_original>"
        "$<TARGET_FILE:cpu_exe_worker_alive_returning>"
        "$<TARGET_FILE:cpu_exe_worker_alive_no_worker>")
    # Two CRT-free DLLs and an EXE: A has no imports, B imports A, the EXE imports both.
    foreach(dll IN ITEMS a a_alt b b_noinit)
        string(REGEX REPLACE "_(noinit|alt)$" "" source "${dll}")
        add_library(cpu_startup_${dll} SHARED tests/cpu/startup_${source}.cpp)
        target_compile_options(cpu_startup_${dll} PRIVATE /GS- /Zl)
        target_link_options(cpu_startup_${dll} PRIVATE /ENTRY:${source}_main /NODEFAULTLIB /INCREMENTAL:NO)
        target_link_libraries(cpu_startup_${dll} PRIVATE kernel32)
    endforeach()
    target_compile_definitions(cpu_startup_b_noinit PRIVATE NO_INIT)
    target_compile_definitions(cpu_startup_a_alt PRIVATE ALTER)
    target_link_libraries(cpu_startup_b PRIVATE cpu_startup_a)
    target_link_libraries(cpu_startup_b_noinit PRIVATE cpu_startup_a)
    add_executable(cpu_startup_exe tests/cpu/startup_exe.cpp)
    target_compile_options(cpu_startup_exe PRIVATE /GS- /Zl)
    target_link_options(cpu_startup_exe PRIVATE /ENTRY:exe_entry /NODEFAULTLIB /SUBSYSTEM:CONSOLE /DYNAMICBASE
        /INCREMENTAL:NO)
    target_link_libraries(cpu_startup_exe PRIVATE kernel32 cpu_startup_a cpu_startup_b)
    foreach(control IN ITEMS timeout create)  # controls: a worker that outlives the wait, and one that cannot start
        add_executable(cpu_startup_exe_${control} tests/cpu/startup_exe.cpp)
        string(TOUPPER "${control}" definition)
        target_compile_definitions(cpu_startup_exe_${control} PRIVATE ${definition}_CONTROL)
        target_compile_options(cpu_startup_exe_${control} PRIVATE /GS- /Zl)
        target_link_options(cpu_startup_exe_${control} PRIVATE /ENTRY:exe_entry /NODEFAULTLIB /SUBSYSTEM:CONSOLE
            /DYNAMICBASE /INCREMENTAL:NO)
        target_link_libraries(cpu_startup_exe_${control} PRIVATE kernel32 cpu_startup_a cpu_startup_b)
    endforeach()
    # Lingering-thread controls (#1375): a thread outlives the entry point; _linger_return ends as before.
    foreach(control IN ITEMS linger linger_return)
        add_executable(cpu_startup_exe_${control} tests/cpu/startup_exe.cpp)
        target_compile_definitions(cpu_startup_exe_${control} PRIVATE LINGER_CONTROL)
        target_compile_options(cpu_startup_exe_${control} PRIVATE /GS- /Zl)
        target_link_options(cpu_startup_exe_${control} PRIVATE /ENTRY:exe_entry /NODEFAULTLIB /SUBSYSTEM:CONSOLE
            /DYNAMICBASE /INCREMENTAL:NO)
        target_link_libraries(cpu_startup_exe_${control} PRIVATE kernel32 cpu_startup_a cpu_startup_b)
    endforeach()
    target_compile_definitions(cpu_startup_exe_linger_return PRIVATE RETURN_FROM_ENTRY)
    # Source-built CRT-free package: import the mapped DLL only by ordinal @77,
    # compare its native entry with pe_run_package, and reject a DLL exporting @78.
    foreach(variant IN ITEMS provider altered wrong live)
        if(variant STREQUAL "wrong")
            set(ord_def tests/cpu/package_ordinal_provider_wrong.def)
        else()
            set(ord_def tests/cpu/package_ordinal_provider.def)
        endif()
        add_library(cpu_package_ordinal_${variant} SHARED tests/cpu/package_ordinal_provider.cpp ${ord_def})
        target_compile_options(cpu_package_ordinal_${variant} PRIVATE /GS- /Zl)
        target_link_options(cpu_package_ordinal_${variant} PRIVATE /ENTRY:ordinal_provider_entry
            /NODEFAULTLIB /INCREMENTAL:NO)
    endforeach()
    target_compile_definitions(cpu_package_ordinal_altered PRIVATE ALTER_RETURN)
    # Control provider (#1375): mix_args leaves a thread running past the caller's entry point.
    target_compile_definitions(cpu_package_ordinal_live PRIVATE LIVE_WORKER)
    target_link_libraries(cpu_package_ordinal_live PRIVATE kernel32)
    # The _return control ends as before #1375: its entry point returns instead of ending the process.
    foreach(target IN ITEMS cpu_package_ordinal_exe cpu_package_ordinal_exe_return)
        add_executable(${target} tests/cpu/package_ordinal_exe.cpp)
        target_compile_options(${target} PRIVATE /GS- /Zl)
        target_link_options(${target} PRIVATE /ENTRY:ordinal_exe_entry
            /NODEFAULTLIB /SUBSYSTEM:CONSOLE /DYNAMICBASE /INCREMENTAL:NO)
        target_link_libraries(${target} PRIVATE cpu_package_ordinal_provider)
    endforeach()
    target_compile_definitions(cpu_package_ordinal_exe_return PRIVATE RETURN_FROM_ENTRY)
    add_test(NAME cpu_native_package_ordinal_startup COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/package_ordinal_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_package_ordinal_exe>" "$<TARGET_FILE:cpu_package_ordinal_provider>"
        "$<TARGET_FILE:cpu_package_ordinal_altered>" "$<TARGET_FILE:cpu_package_ordinal_wrong>"
        "$<TARGET_FILE:cpu_package_ordinal_live>" "$<TARGET_FILE:cpu_package_ordinal_exe_return>")
    set_tests_properties(cpu_native_package_ordinal_startup PROPERTIES TIMEOUT 90)
    add_test(NAME pe_package_ordinal_plan COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/pe_package_ordinal_plan_check.py")
    add_test(NAME cpu_native_startup COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/native_startup_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_startup_exe>" "$<TARGET_FILE:cpu_startup_a>" "$<TARGET_FILE:cpu_startup_b>"
        "$<TARGET_FILE:cpu_startup_b_noinit>" "$<TARGET_FILE:cpu_startup_a_alt>"
        "$<TARGET_FILE:cpu_startup_exe_timeout>" "$<TARGET_FILE:cpu_startup_exe_create>"
        "$<TARGET_FILE:cpu_startup_exe_linger>" "$<TARGET_FILE:cpu_startup_exe_linger_return>")
    # EXE-own TLS: a CRT-free EXE with a page-spanning initialized template and two TLS callbacks, launched
    # by Windows and started with pe_map_run; the startup log and exit code must match (Refs #165, #870).
    # The _early variant starts a thread inside its first process-attach callback (mapped only).
    foreach(target IN ITEMS cpu_exe_tls cpu_exe_tls_early)
        add_executable(${target} tests/cpu/exe_tls.cpp tests/cpu/exe_tls_template.asm)
        target_compile_options(${target} PRIVATE "$<$<COMPILE_LANGUAGE:CXX>:/GS-;/Zl;/Zc:tlsGuards->")
        target_link_options(${target} PRIVATE /ENTRY:exe_entry /NODEFAULTLIB /SUBSYSTEM:CONSOLE /DYNAMICBASE
            /INCREMENTAL:NO)
        target_link_libraries(${target} PRIVATE kernel32)
    endforeach()
    target_compile_definitions(cpu_exe_tls_early PRIVATE EARLY_THREAD)
    add_test(NAME cpu_native_exe_tls COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/native_exe_tls_check.py" "$<TARGET_FILE:pe_map>" "$<TARGET_FILE:cpu_exe_tls>"
        "$<TARGET_FILE:cpu_exe_tls_early>")
    # Main-image TLS slot 0: a CRT-free /GA EXE whose thread-locals are read through slot 0, launched by
    # Windows and started by a CRT-free host that owns slot 0 itself, with pe_map_run_main and explicit
    # participants and, as the control, pe_map_run (Refs #870, #165).
    # The _pause variant leaves a worker paused inside the image for the release refusal (mapped only). The
    # _enroll variant starts a probe worker whose enrollment or installation the host makes fail.
    foreach(target IN ITEMS cpu_exe_tls_slot0 cpu_exe_tls_slot0_pause cpu_exe_tls_slot0_enroll)
        add_executable(${target} tests/cpu/exe_tls_slot0.cpp tests/cpu/exe_tls_template.asm)
        target_compile_definitions(${target} PRIVATE $<$<COMPILE_LANGUAGE:ASM_MASM>:TLS_SLOT0_ARRAY>)
        target_compile_options(${target} PRIVATE "$<$<COMPILE_LANGUAGE:CXX>:/GS-;/Zl;/GA;/Zc:tlsGuards->")
        target_link_options(${target} PRIVATE /ENTRY:exe_entry /NODEFAULTLIB /SUBSYSTEM:CONSOLE /DYNAMICBASE
            /INCREMENTAL:NO)
        target_link_libraries(${target} PRIVATE kernel32)
    endforeach()
    target_compile_definitions(cpu_exe_tls_slot0_pause PRIVATE PAUSED_WORKER)
    target_compile_definitions(cpu_exe_tls_slot0_enroll PRIVATE ENROLL_PROBE)
    add_executable(cpu_exe_tls_slot0_host tests/cpu/exe_tls_slot0_host.cpp)
    target_compile_options(cpu_exe_tls_slot0_host PRIVATE /GS- /Zl)
    target_link_options(cpu_exe_tls_slot0_host PRIVATE /ENTRY:host_entry /NODEFAULTLIB /SUBSYSTEM:CONSOLE
        /DYNAMICBASE /INCREMENTAL:NO)
    target_link_libraries(cpu_exe_tls_slot0_host PRIVATE kernel32)
    # Separate evidence: the Windows launch, and the pe_map scenarios with explicit participants.
    add_test(NAME cpu_native_exe_tls_slot0_windows COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/native_exe_tls_slot0_check.py" windows "$<TARGET_FILE:cpu_exe_tls_slot0>")
    add_test(NAME cpu_native_exe_tls_slot0 COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/native_exe_tls_slot0_check.py" mapped "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_exe_tls_slot0>" "$<TARGET_FILE:cpu_exe_tls_slot0_host>"
        "$<TARGET_FILE:cpu_exe_tls_slot0_pause>")
    # Guest-thread enrollment failures, one child host process each: the probe worker's start routine must
    # never run, and the host must survive.
    add_test(NAME cpu_native_exe_tls_slot0_enroll COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/native_exe_tls_slot0_check.py" enroll "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_exe_tls_slot0_enroll>" "$<TARGET_FILE:cpu_exe_tls_slot0_host>")
    # pe_run_package starts the same /GA EXE as the package's main image; its log must equal the Windows launch.
    add_test(NAME cpu_package_main_image COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/package_main_image_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_exe_tls_slot0>")
    set_tests_properties(cpu_package_main_image PROPERTIES TIMEOUT 60)
    # Lifecycle fixtures: two DLLs with the static MSVC runtime (B imports A) and a CRT-free event sink.
    add_library(cpu_life_sink SHARED tests/cpu/life_sink.cpp)
    target_compile_options(cpu_life_sink PRIVATE /GS- /Zl)
    target_link_options(cpu_life_sink PRIVATE /ENTRY:sink_main /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(cpu_life_sink PRIVATE kernel32)
    foreach(dll IN ITEMS a a_noctor b b_nodtor)
        string(REGEX REPLACE "_(noctor|nodtor)$" "" source "${dll}")
        add_library(cpu_life_${dll} SHARED tests/cpu/life_${source}.cpp)
        set_target_properties(cpu_life_${dll} PROPERTIES MSVC_RUNTIME_LIBRARY MultiThreaded)
        target_link_options(cpu_life_${dll} PRIVATE /INCREMENTAL:NO)  # no incremental thunk in front of the entry point
        target_link_libraries(cpu_life_${dll} PRIVATE cpu_life_sink)
    endforeach()
    target_compile_definitions(cpu_life_a_noctor PRIVATE NO_CTOR)
    target_compile_definitions(cpu_life_b_nodtor PRIVATE NO_DTOR)
    target_link_libraries(cpu_life_b PRIVATE cpu_life_a)
    target_link_libraries(cpu_life_b_nodtor PRIVATE cpu_life_a)
    add_test(NAME cpu_native_crt_lifecycle COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/native_crt_lifecycle_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_life_sink>" "$<TARGET_FILE:cpu_life_a>" "$<TARGET_FILE:cpu_life_b>"
        "$<TARGET_FILE:cpu_life_a_noctor>" "$<TARGET_FILE:cpu_life_b_nodtor>")
    add_library(cpu_frame_host SHARED tests/cpu/frame_host.cpp)
    add_library(cpu_frame_guest SHARED tests/cpu/tls_route.asm tests/cpu/frame_route.def)
    set_target_properties(cpu_frame_guest PROPERTIES LINKER_LANGUAGE CXX)
    target_compile_definitions(cpu_frame_guest PRIVATE FRAME_GUEST)
    target_link_options(cpu_frame_guest PRIVATE /ENTRY:dll_entry /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(cpu_frame_guest PRIVATE cpu_tls_sink cpu_frame_host)

    add_library(cpu_delay_provider SHARED tests/cpu/delay_provider.cpp tests/cpu/delay_provider.def)
    add_library(cpu_delay_provider_altered SHARED tests/cpu/delay_provider.cpp
        tests/cpu/delay_provider_altered.def)
    target_compile_definitions(cpu_delay_provider_altered PRIVATE ALTER_RETURN)
    add_library(cpu_delay_route SHARED tests/cpu/delay_route.cpp)
    target_link_libraries(cpu_delay_route PRIVATE cpu_delay_provider delayimp)
    target_link_options(cpu_delay_route PRIVATE /DELAYLOAD:cpu_delay_provider.dll /INCREMENTAL:NO)
    add_test(NAME cpu_native_delay COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/native_delay_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_delay_route>" "$<TARGET_FILE:cpu_delay_provider>"
        "$<TARGET_FILE:cpu_delay_provider_altered>")
    add_library(cpu_tools_mem_provider SHARED tests/cpu/tools_mem_provider.cpp tests/cpu/tools_mem_provider.def)
    add_library(cpu_tools_mem_provider_devkit SHARED tests/cpu/tools_mem_provider.cpp
        tests/cpu/tools_mem_provider_devkit.def)
    target_compile_definitions(cpu_tools_mem_provider_devkit PRIVATE TOOLS_PARTITION_PRESENT)
    add_executable(cpu_tools_mem_caller tests/cpu/tools_mem_caller.asm tests/cpu/exe_startup.asm)
    set_target_properties(cpu_tools_mem_caller PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_tools_mem_caller PRIVATE /ENTRY:native_startup /SUBSYSTEM:CONSOLE /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(cpu_tools_mem_caller PRIVATE cpu_exe_io cpu_tools_mem_provider)
    add_test(NAME cpu_native_tools_mem COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/tools_mem_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_tools_mem_caller>" "$<TARGET_FILE:cpu_tools_mem_provider>"
        "$<TARGET_FILE:cpu_tools_mem_provider_devkit>" "$<TARGET_FILE:cpu_exe_io>")
    set_tests_properties(cpu_native_tools_mem PROPERTIES TIMEOUT 60)

    add_library(cpu_map_pages_provider SHARED tests/cpu/map_pages_provider.cpp tests/cpu/map_pages_provider.def)
    add_executable(cpu_map_pages_caller tests/cpu/map_pages_caller.asm tests/cpu/exe_startup.asm)
    set_target_properties(cpu_map_pages_caller PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_map_pages_caller PRIVATE /ENTRY:native_startup /EXPORT:native_entry /SUBSYSTEM:CONSOLE /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(cpu_map_pages_caller PRIVATE cpu_map_pages_provider cpu_exe_io)
    add_test(NAME cpu_native_map_pages COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/map_pages_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_map_pages_caller>" "$<TARGET_FILE:cpu_map_pages_provider>"
        "$<TARGET_FILE:cpu_exe_io>")
    set_tests_properties(cpu_native_map_pages PROPERTIES TIMEOUT 60)

    add_library(cpu_transfer_mem_provider SHARED tests/cpu/transfer_mem_provider.cpp tests/cpu/transfer_mem_provider.def)
    add_executable(cpu_transfer_mem_caller tests/cpu/transfer_mem_caller.asm tests/cpu/exe_startup.asm)
    set_target_properties(cpu_transfer_mem_caller PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_transfer_mem_caller PRIVATE /ENTRY:native_startup /EXPORT:native_entry /SUBSYSTEM:CONSOLE /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(cpu_transfer_mem_caller PRIVATE cpu_transfer_mem_provider cpu_exe_io)
    add_test(NAME cpu_native_transfer_mem COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/transfer_mem_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_transfer_mem_caller>" "$<TARGET_FILE:cpu_transfer_mem_provider>"
        "$<TARGET_FILE:cpu_exe_io>")
    set_tests_properties(cpu_native_transfer_mem PROPERTIES TIMEOUT 60)

    add_library(cpu_transfer_chain_provider SHARED tests/cpu/transfer_chain_provider.cpp tests/cpu/transfer_chain_provider.def)
    add_executable(cpu_transfer_chain_caller tests/cpu/transfer_chain_caller.asm tests/cpu/exe_startup.asm)
    set_target_properties(cpu_transfer_chain_caller PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_transfer_chain_caller PRIVATE /ENTRY:native_startup /EXPORT:native_entry /SUBSYSTEM:CONSOLE /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(cpu_transfer_chain_caller PRIVATE cpu_transfer_chain_provider cpu_exe_io)
    add_test(NAME cpu_native_transfer_chain COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/transfer_chain_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_transfer_chain_caller>" "$<TARGET_FILE:cpu_transfer_chain_provider>"
        "$<TARGET_FILE:cpu_exe_io>")
    set_tests_properties(cpu_native_transfer_chain PROPERTIES TIMEOUT 60)

    add_library(cpu_apu_alloc_provider SHARED tests/cpu/apu_alloc_provider.cpp tests/cpu/apu_alloc_provider.def)
    add_library(cpu_apu_alloc_provider_lenient SHARED tests/cpu/apu_alloc_provider.cpp tests/cpu/apu_alloc_provider.def)
    target_compile_definitions(cpu_apu_alloc_provider_lenient PRIVATE APU_TEST_LENIENT)
    add_executable(cpu_apu_alloc_caller tests/cpu/apu_alloc_caller.asm tests/cpu/exe_startup.asm)
    set_target_properties(cpu_apu_alloc_caller PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_apu_alloc_caller PRIVATE /ENTRY:native_startup /EXPORT:native_entry /SUBSYSTEM:CONSOLE /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(cpu_apu_alloc_caller PRIVATE cpu_apu_alloc_provider cpu_exe_io)
    add_test(NAME cpu_native_apu_alloc_caller COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/apu_alloc_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_apu_alloc_caller>" "$<TARGET_FILE:cpu_apu_alloc_provider>"
        "$<TARGET_FILE:cpu_apu_alloc_provider_lenient>" "$<TARGET_FILE:cpu_exe_io>")
    set_tests_properties(cpu_native_apu_alloc_caller PROPERTIES TIMEOUT 60)

    add_library(cpu_apu_address_provider SHARED tests/cpu/apu_address_provider.cpp tests/cpu/apu_address_provider.def)
    add_library(cpu_apu_address_provider_faulty SHARED tests/cpu/apu_address_provider.cpp tests/cpu/apu_address_provider.def)
    target_compile_definitions(cpu_apu_address_provider_faulty PRIVATE APU_TEST_FAULTY)
    add_executable(cpu_apu_address_caller tests/cpu/apu_address_caller.asm tests/cpu/exe_startup.asm)
    set_target_properties(cpu_apu_address_caller PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_apu_address_caller PRIVATE /ENTRY:native_startup /EXPORT:native_entry /SUBSYSTEM:CONSOLE /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(cpu_apu_address_caller PRIVATE cpu_apu_address_provider cpu_exe_io)
    add_test(NAME cpu_native_apu_address COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/apu_address_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_apu_address_caller>" "$<TARGET_FILE:cpu_apu_address_provider>"
        "$<TARGET_FILE:cpu_apu_address_provider_faulty>" "$<TARGET_FILE:cpu_exe_io>")
    set_tests_properties(cpu_native_apu_address PROPERTIES TIMEOUT 60)

    add_library(cpu_acphal_hal_provider SHARED tests/cpu/acphal_hal_provider.cpp tests/cpu/acphal_hal_provider.def)
    add_library(cpu_acphal_hal_provider_faulty SHARED tests/cpu/acphal_hal_provider.cpp tests/cpu/acphal_hal_provider.def)
    target_compile_definitions(cpu_acphal_hal_provider_faulty PRIVATE ACPHAL_TEST_FAULTY)
    add_executable(cpu_acphal_hal_caller tests/cpu/acphal_hal_caller.asm tests/cpu/exe_startup.asm)
    set_target_properties(cpu_acphal_hal_caller PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_acphal_hal_caller PRIVATE /ENTRY:native_startup /EXPORT:native_entry /SUBSYSTEM:CONSOLE /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(cpu_acphal_hal_caller PRIVATE cpu_acphal_hal_provider cpu_exe_io)
    add_test(NAME cpu_native_acphal_hal COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/acphal_hal_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_acphal_hal_caller>" "$<TARGET_FILE:cpu_acphal_hal_provider>"
        "$<TARGET_FILE:cpu_acphal_hal_provider_faulty>" "$<TARGET_FILE:cpu_exe_io>")
    set_tests_properties(cpu_native_acphal_hal PROPERTIES TIMEOUT 60)

    add_library(cpu_gameinput_provider SHARED tests/cpu/gameinput_provider.cpp tests/cpu/gameinput_provider.def)
    foreach(variant IN ITEMS create_fail other_thread shifted_layout)
        string(TOUPPER "${variant}" variant_define)
        add_library(cpu_gameinput_provider_${variant} SHARED
            tests/cpu/gameinput_provider.cpp tests/cpu/gameinput_provider.def)
        target_compile_definitions(cpu_gameinput_provider_${variant} PRIVATE GAMEINPUT_TEST_${variant_define})
    endforeach()
    add_library(cpu_gameinput_provider_wrong SHARED tests/cpu/gameinput_provider.cpp tests/cpu/gameinput_provider_wrong.def)
    add_executable(cpu_gameinput_caller tests/cpu/gameinput_caller.asm tests/cpu/exe_startup.asm)
    set_target_properties(cpu_gameinput_caller PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_gameinput_caller PRIVATE /ENTRY:native_startup /EXPORT:native_entry /SUBSYSTEM:CONSOLE /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(cpu_gameinput_caller PRIVATE cpu_gameinput_provider cpu_exe_io)
    add_test(NAME cpu_native_gameinput COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/gameinput_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_gameinput_caller>" "$<TARGET_FILE:cpu_gameinput_provider>"
        "$<TARGET_FILE:cpu_gameinput_provider_create_fail>" "$<TARGET_FILE:cpu_gameinput_provider_other_thread>"
        "$<TARGET_FILE:cpu_gameinput_provider_shifted_layout>" "$<TARGET_FILE:cpu_gameinput_provider_wrong>"
        "$<TARGET_FILE:cpu_exe_io>")
    set_tests_properties(cpu_native_gameinput PROPERTIES TIMEOUT 120)

    add_library(cpu_xapu_queue_provider SHARED tests/cpu/xapu_queue_provider.cpp tests/cpu/xapu_queue_provider.def)
    add_library(cpu_xapu_queue_provider_lenient SHARED tests/cpu/xapu_queue_provider.cpp tests/cpu/xapu_queue_provider.def)
    target_compile_definitions(cpu_xapu_queue_provider_lenient PRIVATE XAPU_TEST_LENIENT)
    add_library(cpu_xapu_queue_provider_lifo SHARED tests/cpu/xapu_queue_provider.cpp tests/cpu/xapu_queue_provider.def)
    target_compile_definitions(cpu_xapu_queue_provider_lifo PRIVATE XAPU_TEST_LIFO)
    add_executable(cpu_xapu_queue_caller tests/cpu/xapu_queue_caller.asm tests/cpu/exe_startup.asm)
    set_target_properties(cpu_xapu_queue_caller PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_xapu_queue_caller PRIVATE /ENTRY:native_startup /EXPORT:native_entry /SUBSYSTEM:CONSOLE /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(cpu_xapu_queue_caller PRIVATE cpu_xapu_queue_provider cpu_exe_io)
    add_test(NAME cpu_native_xapu_queue COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/xapu_queue_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_xapu_queue_caller>" "$<TARGET_FILE:cpu_xapu_queue_provider>"
        "$<TARGET_FILE:cpu_xapu_queue_provider_lenient>" "$<TARGET_FILE:cpu_xapu_queue_provider_lifo>"
        "$<TARGET_FILE:cpu_exe_io>")
    set_tests_properties(cpu_native_xapu_queue PROPERTIES TIMEOUT 60)

    add_library(cpu_dstorage_provider SHARED tests/cpu/dstorage_provider.cpp tests/cpu/dstorage_provider.def)
    add_executable(cpu_dstorage_caller tests/cpu/dstorage_caller.asm tests/cpu/exe_startup.asm)
    set_target_properties(cpu_dstorage_caller PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_dstorage_caller PRIVATE /ENTRY:native_startup /EXPORT:native_entry /SUBSYSTEM:CONSOLE /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(cpu_dstorage_caller PRIVATE cpu_dstorage_provider cpu_exe_io)
    add_test(NAME cpu_native_dstorage COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/dstorage_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_dstorage_caller>" "$<TARGET_FILE:cpu_dstorage_provider>"
        "$<TARGET_FILE:cpu_exe_io>")
    set_tests_properties(cpu_native_dstorage PROPERTIES TIMEOUT 60)

    add_library(cpu_frame_events_provider SHARED tests/cpu/frame_events_provider.cpp tests/cpu/frame_events_provider.def)
    add_library(cpu_frame_events_provider_clobber SHARED tests/cpu/frame_events_provider.cpp tests/cpu/frame_events_provider.def)
    target_compile_definitions(cpu_frame_events_provider_clobber PRIVATE FAULT_CLOBBER)
    add_library(cpu_frame_events_provider_early SHARED tests/cpu/frame_events_provider.cpp tests/cpu/frame_events_provider.def)
    target_compile_definitions(cpu_frame_events_provider_early PRIVATE FAULT_EARLY_RELEASE)
    add_executable(cpu_frame_events_caller tests/cpu/frame_events_caller.asm tests/cpu/exe_startup.asm)
    set_target_properties(cpu_frame_events_caller PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_frame_events_caller PRIVATE /ENTRY:native_startup /EXPORT:native_entry /SUBSYSTEM:CONSOLE /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(cpu_frame_events_caller PRIVATE cpu_frame_events_provider cpu_exe_io)
    add_test(NAME cpu_native_frame_events COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/frame_events_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_frame_events_caller>" "$<TARGET_FILE:cpu_frame_events_provider>"
        "$<TARGET_FILE:cpu_frame_events_provider_clobber>"
        "$<TARGET_FILE:cpu_frame_events_provider_early>" "$<TARGET_FILE:cpu_exe_io>")
    set_tests_properties(cpu_native_frame_events PROPERTIES TIMEOUT 60)

    add_library(cpu_xg_swizzle_provider SHARED tests/cpu/xg_swizzle_provider.cpp tests/cpu/xg_swizzle_provider.def)
    add_library(cpu_xg_swizzle_provider_clobber SHARED tests/cpu/xg_swizzle_provider.cpp tests/cpu/xg_swizzle_provider.def)
    target_compile_definitions(cpu_xg_swizzle_provider_clobber PRIVATE FAULT_CLOBBER)
    add_library(cpu_xg_swizzle_provider_linear SHARED tests/cpu/xg_swizzle_provider.cpp tests/cpu/xg_swizzle_provider.def)
    target_compile_definitions(cpu_xg_swizzle_provider_linear PRIVATE FAULT_LINEAR)
    add_executable(cpu_xg_swizzle_caller tests/cpu/xg_swizzle_caller.asm tests/cpu/exe_startup.asm)
    set_target_properties(cpu_xg_swizzle_caller PROPERTIES LINKER_LANGUAGE CXX)
    target_link_options(cpu_xg_swizzle_caller PRIVATE /ENTRY:native_startup /EXPORT:native_entry /SUBSYSTEM:CONSOLE /NODEFAULTLIB /INCREMENTAL:NO)
    target_link_libraries(cpu_xg_swizzle_caller PRIVATE cpu_xg_swizzle_provider cpu_exe_io)
    add_test(NAME cpu_native_xg_swizzle COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/xg_swizzle_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_xg_swizzle_caller>" "$<TARGET_FILE:cpu_xg_swizzle_provider>"
        "$<TARGET_FILE:cpu_xg_swizzle_provider_clobber>"
        "$<TARGET_FILE:cpu_xg_swizzle_provider_linear>" "$<TARGET_FILE:cpu_exe_io>")
    set_tests_properties(cpu_native_xg_swizzle PROPERTIES TIMEOUT 60)
    add_test(NAME cpu_native_lifetime COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/cpu/native_lifetime_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:cpu_tls_original>" "$<TARGET_FILE:cpu_tls_sink>")
    add_library(script_path_host SHARED tools/script_path_host.cpp)
    target_compile_features(script_path_host PRIVATE cxx_std_17)
    add_test(NAME script_path_links COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/script_path_check.py" "$<TARGET_FILE:script_path_host>")
    add_executable(ffx_pass26_lds tests/ffx_pass26_lds/gate.cpp)
    target_link_libraries(ffx_pass26_lds PRIVATE d3d11 d3dcompiler)
    add_test(NAME ffx_immediate_float_to_int_warp COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/ffx_dispatch_aot_check.py" --immediate "${DXC_EXECUTABLE}" $<TARGET_FILE:offscreen_check> warp)
    add_test(NAME ffx_float_to_int_warp COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/ffx_dispatch_aot_check.py" --readback $<TARGET_FILE:ffx_pass26_lds> warp)
    add_test(NAME ffx_float_math_warp COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/ffx_dispatch_aot_check.py" --readback-float-math $<TARGET_FILE:ffx_pass26_lds> warp)
    add_test(NAME ffx_pass26_lds_warp COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/ffx_pass26_lds/verify.py"
        --host $<TARGET_FILE:ffx_pass26_lds> --backend warp)

    add_test(NAME script_state_literals COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/script_state_check.py")
    add_test(NAME resource_relocated_root COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/resource_root_check.py")
    add_test(NAME script_dispatch_pairs COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/script_dispatch_check.py" "$<TARGET_FILE:script_path_host>")
    add_test(NAME bcpack_prefix_and_mip_tail COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/bcpack_check.py")
    add_test(NAME bcpack_rans_symbols COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/bcpack_symbols_check.py")
    add_test(NAME bcpack_tables COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/bcpack_tables_check.py")
    add_test(NAME bc7_block_reference COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/bc7_blocks_check.py")
    add_test(NAME bc7_tile_swizzle COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/bc7_tiles_check.py")
    add_test(NAME gfx10_texture_roundtrip COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/gfx10_texture_roundtrip_check.py" roundtrip)
    add_test(NAME gfx10_texture_roundtrip_controls COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/gfx10_texture_roundtrip_check.py" controls)
    add_test(NAME texture_bc_volume COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/texture_bc_volume_check.py")
    add_test(NAME resource_range_controls COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/resource_ranges_check.py")
    add_test(NAME ttpf_batch_export COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/ttpf_batch_export_check.py")
    add_test(NAME audio_awc_metadata COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/awc_metadata_check.py")
    add_test(NAME sgd5_effect_candidates COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/sgd5_effects_check.py")
    add_test(NAME rdna2_tools_check COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/rdna2_tools_check.py")
    add_test(NAME pso_readiness_check COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/pso_readiness_check.py")
    add_test(NAME dxcontainer_part_offsets COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/dxcontainer_part_offsets_check.py")
    # #369: the built cs_6_0 array-atomics probe keeps nonUniformIndex on its atomic's handle.
    add_test(NAME array_atomics_dxil COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/array_atomics_dxil_check.py" "${DXC_EXECUTABLE}"
        "${CMAKE_CURRENT_BINARY_DIR}/ProbeArrayAtomics.cso" "${CMAKE_SOURCE_DIR}/tests/gpu_alias_writes.hlsl"
        "${CMAKE_SOURCE_DIR}/src")
    add_test(NAME mesh_amplification_refusal COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/mesh_amplification_refusal_check.py")
    set_tests_properties(mesh_amplification_refusal PROPERTIES TIMEOUT 30)
    foreach(adapter IN ITEMS hardware warp)
        add_test(NAME mesh_pixel_pipeline_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_pixel_pipeline_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_pixel_pipeline_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME mesh_cbv_order_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_cbv_order_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_cbv_order_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME mesh_descriptor_swap_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_descriptor_swap_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_descriptor_swap_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME mesh_indexed_fallback_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_fallback_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_indexed_fallback_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME mesh_indexed_draw_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_indexed_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_indexed_draw_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME mesh_attribute_draw_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_attribute_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_attribute_draw_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME mesh_textured_draw_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_textured_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_textured_draw_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME mesh_textured_fallback_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_textured_check.py" fallback "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_textured_fallback_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME compute_mesh_texture_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_mesh_texture_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_mesh_texture_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME compute_mesh_texture_fallback_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_mesh_texture_check.py" fallback "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_mesh_texture_fallback_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        # #810: default and -Od DXIL of the same source-built mesh shader through one PSO harness.
        add_test(NAME mesh_od_pipeline_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tools/mesh_od_pipeline.py" --check --dxc "${DXC_EXECUTABLE}"
            --renderer "$<TARGET_FILE:offscreen_check>" --adapter ${adapter}
            --seed 0x5eed0001 --compile-timeout 30 --pipeline-timeout 30)
        set_tests_properties(mesh_od_pipeline_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77)
        add_test(NAME mesh_depth_pair_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_depth_pair_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_depth_pair_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME mesh_depth_pair_fallback_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_depth_pair_check.py" fallback "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_depth_pair_fallback_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME mesh_lit_draw_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_lit_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_lit_draw_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME mesh_lit_fallback_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_lit_check.py" fallback "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_lit_fallback_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME mesh_shadow_draw_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_shadow_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_shadow_draw_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME mesh_entry_draw_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_entry_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_entry_draw_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME mesh_entry_float_draw_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_entry_check.py" draw-float "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_entry_float_draw_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME mesh_entry_cvt_minmax_draw_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_entry_check.py" draw-cvt-minmax "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_entry_cvt_minmax_draw_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME mesh_entry_unary_draw_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_entry_check.py" draw-unary "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_entry_unary_draw_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME mesh_amplified_draw_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/mesh_amplified_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(mesh_amplified_draw_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME compute_store_aot_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_${adapter} PROPERTIES TIMEOUT 45)
        # Entry registers from COMPUTE_PGM_RSRC2: user SGPRs, group IDs and local IDs (#1544).
        add_test(NAME compute_entry_layout_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_entry_layout_check.py" draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_entry_layout_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME compute_store_aot_dpp_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-dpp "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_dpp_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME compute_store_aot_guard_read_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-guard-read "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_guard_read_${adapter} PROPERTIES TIMEOUT 45)
        add_test(NAME compute_store_aot_mubuf_subword_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mubuf-subword "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mubuf_subword_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mubuf_subword_${adapter}.txt")
        add_test(NAME compute_store_aot_f32_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-f32 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_f32_${adapter} PROPERTIES TIMEOUT 60)
        add_test(NAME compute_store_aot_mubuf_reduction_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mubuf-reduction "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mubuf_reduction_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mubuf_reduction_${adapter}.txt")
        add_test(NAME compute_store_aot_alu_minmax3_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_alu_minmax3_check.py" draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_alu_minmax3_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME compute_store_aot_mubuf_reduction_x2_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mubuf-reduction-x2 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mubuf_reduction_x2_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mubuf_reduction_x2_${adapter}.txt")
        foreach(exchange IN ITEMS exchange exchange-x2)
            add_test(NAME compute_store_aot_mubuf_${exchange}_${adapter} COMMAND "${INTAKE_PYTHON}" -B
                "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" "draw-mubuf-${exchange}" "${DXC_EXECUTABLE}"
                "$<TARGET_FILE:offscreen_check>" ${adapter})
            set_tests_properties(compute_store_aot_mubuf_${exchange}_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
                ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mubuf_${exchange}_${adapter}.txt")
        endforeach()
        foreach(counter IN ITEMS counter counter-x2)
            add_test(NAME compute_store_aot_mubuf_${counter}_${adapter} COMMAND "${INTAKE_PYTHON}" -B
                "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" "draw-mubuf-${counter}" "${DXC_EXECUTABLE}"
                "$<TARGET_FILE:offscreen_check>" ${adapter})
            set_tests_properties(compute_store_aot_mubuf_${counter}_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
                ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mubuf_${counter}_${adapter}.txt")
        endforeach()
        add_test(NAME compute_store_aot_mubuf_csub_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mubuf-csub "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mubuf_csub_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mubuf_csub_${adapter}.txt")
        add_test(NAME compute_store_aot_mubuf_vector_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mubuf-vector "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mubuf_vector_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mubuf_vector_${adapter}.txt")
        add_test(NAME compute_store_aot_alu_med3_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_alu_med3_check.py" draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_alu_med3_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME compute_store_aot_mubuf_d16_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mubuf-d16 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mubuf_d16_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mubuf_d16_${adapter}.txt")
        add_test(NAME compute_store_aot_alu_xor3_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_alu_xor3_check.py" draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_alu_xor3_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME compute_store_aot_mubuf_load_format_d16_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mubuf-load-format-d16 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mubuf_load_format_d16_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mubuf_load_format_d16_${adapter}.txt")
        add_test(NAME compute_store_aot_mtbuf_load_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mtbuf-load "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mtbuf_load_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mtbuf_load_${adapter}.txt")
        # A translated gfx1030 compute shader reads an R32G32B32A32_UINT texture through image_load and
        # stores the dmask-selected texel components; the harness compares all 320 UAV readback words.
        add_test(NAME compute_store_aot_mubuf_format_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mubuf-format "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mubuf_format_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mubuf_format_${adapter}.txt")
        add_test(NAME compute_store_aot_mimg_load_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mimg-load "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mimg_load_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mimg_load_${adapter}.txt")
        add_test(NAME compute_store_aot_mimg_mip_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mimg-mip "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mimg_mip_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mimg_mip_${adapter}.txt")
        add_test(NAME compute_store_aot_mimg_resinfo_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mimg-resinfo "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mimg_resinfo_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mimg_resinfo_${adapter}.txt")
        add_test(NAME compute_store_aot_mimg_packed_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mimg-packed "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mimg_packed_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mimg_packed_${adapter}.txt")
        add_test(NAME compute_store_aot_mimg_msaa_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_mimg_msaa_check.py" --draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mimg_msaa_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mimg_msaa_${adapter}.txt")
        add_test(NAME compute_store_aot_gfx1013_image_store_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_mimg_float_store_check.py" --draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_gfx1013_image_store_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME compute_store_aot_mimg_store_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mimg-store "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mimg_store_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mimg_store_${adapter}.txt")
        add_test(NAME compute_store_aot_mimg_store_packed_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mimg-packed-store "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mimg_store_packed_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mimg_store_packed_${adapter}.txt")
        add_test(NAME compute_store_aot_mimg_atomic_integer_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mimg-atomic-integer "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mimg_atomic_integer_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mimg_atomic_integer_${adapter}.txt")
        add_test(NAME compute_store_aot_mimg_atomic_minmax_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mimg-atomic-minmax "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mimg_atomic_minmax_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mimg_atomic_minmax_${adapter}.txt")
        add_test(NAME compute_store_aot_mimg_atomic_exchange_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mimg-atomic-exchange "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mimg_atomic_exchange_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mimg_atomic_exchange_${adapter}.txt")
        add_test(NAME compute_store_aot_mimg_atomic_counter_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mimg-atomic-counter "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mimg_atomic_counter_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mimg_atomic_counter_${adapter}.txt")
        add_test(NAME compute_store_aot_ds_route_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-ds-route "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_ds_route_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_ds_route_${adapter}.txt")
        add_test(NAME compute_store_aot_lds_private_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-lds-private "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_lds_private_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_lds_private_${adapter}.txt")
        add_test(NAME compute_store_aot_lds_packed_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-lds-packed "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_lds_packed_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_lds_packed_${adapter}.txt")
        foreach(lds_case IN ITEMS arithmetic replacement)
            add_test(NAME compute_store_aot_lds_${lds_case}_${adapter} COMMAND "${INTAKE_PYTHON}" -B
                "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-lds-${lds_case} "${DXC_EXECUTABLE}"
                "$<TARGET_FILE:offscreen_check>" ${adapter})
            set_tests_properties(compute_store_aot_lds_${lds_case}_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
                ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_lds_${lds_case}_${adapter}.txt")
        endforeach()
        add_test(NAME compute_store_aot_lds_strided_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-lds-strided "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_lds_strided_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_lds_strided_${adapter}.txt")
        add_test(NAME compute_store_aot_lds_wide_replacement_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-lds-wide-replacement "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_lds_wide_replacement_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_lds_wide_replacement_${adapter}.txt")
        add_test(NAME compute_store_aot_lds_qword_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-lds-qword "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_lds_qword_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_lds_qword_${adapter}.txt")
        add_test(NAME compute_store_aot_lds_f32_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-lds-f32 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_lds_f32_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_lds_f32_${adapter}.txt")
        add_test(NAME compute_store_aot_mubuf_store_format_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mubuf-format-store "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mubuf_store_format_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mubuf_store_format_${adapter}.txt")
        add_test(NAME compute_store_aot_mubuf_store_format_xyz_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mubuf-format-store-xyz "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mubuf_store_format_xyz_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77)
        add_test(NAME compute_store_aot_mtbuf_store_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mtbuf-store "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mtbuf_store_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mtbuf_store_${adapter}.txt")
        add_test(NAME compute_store_aot_mubuf_store_subword_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_store_check.py" draw-mubuf-subword-store "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_store_aot_mubuf_store_subword_${adapter} PROPERTIES TIMEOUT 45 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/compute_store_aot_mubuf_store_subword_${adapter}.txt")
        add_test(NAME compute_to_pixel_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/compute_to_pixel_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(compute_to_pixel_${adapter} PROPERTIES TIMEOUT 30 SKIP_RETURN_CODE 77)
        add_test(NAME cpu_native_frame_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/cpu/native_frame_check.py" "$<TARGET_FILE:cpu_frame_guest>"
            "$<TARGET_FILE:cpu_tls_sink>" "$<TARGET_FILE:cpu_frame_host>" "$<TARGET_FILE:pe_map>"
            "${DXC_EXECUTABLE}" "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(cpu_native_frame_${adapter} PROPERTIES TIMEOUT 95)
        add_test(NAME pixel_aot_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-core "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_${adapter} PROPERTIES TIMEOUT 60)
        add_test(NAME pixel_aot_core_vector_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-core-vector "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_core_vector_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_core_vector_${adapter}.txt")
        add_test(NAME pixel_aot_dpp_cvt_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-dpp-cvt "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_dpp_cvt_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_dpp_cvt_${adapter}.txt")
        add_test(NAME pixel_aot_dpp_movrel_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-dpp-movrel "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_dpp_movrel_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_dpp_movrel_${adapter}.txt")
        add_test(NAME pixel_aot_dpp_f16_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-dpp-f16 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_dpp_f16_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_dpp_f16_${adapter}.txt")
        add_test(NAME pixel_aot_dpp_extra_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-dpp-extra "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_dpp_extra_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_dpp_extra_${adapter}.txt")
        add_test(NAME pixel_aot_compressed_export_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-compressed-export "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_compressed_export_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME pixel_aot_uncompressed_export_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-uncompressed-export "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_uncompressed_export_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME pixel_aot_uncompressed_integer_export_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-uncompressed-integer-export "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_uncompressed_integer_export_${adapter} PROPERTIES
            TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME pixel_aot_ds_swizzle_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-ds-swizzle "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_ds_swizzle_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_ds_swizzle_${adapter}.txt")
        add_test(NAME pixel_aot_exec_regions_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-exec-regions "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_exec_regions_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_exec_regions_${adapter}.txt")
        add_test(NAME pixel_aot_scc_skip_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-scc-skip "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_scc_skip_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_scc_skip_${adapter}.txt")
        add_test(NAME pixel_aot_cmpx_regions_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-cmpx-regions "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_cmpx_regions_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_cmpx_regions_${adapter}.txt")
        add_test(NAME pixel_aot_cmpx_e64_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-cmpx-e64 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_cmpx_e64_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_cmpx_e64_${adapter}.txt")
        add_test(NAME pixel_aot_region_body_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-region-body "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_region_body_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_region_body_${adapter}.txt")
        add_test(NAME pixel_aot_region_waits_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-region-waits "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_region_waits_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_region_waits_${adapter}.txt")
        add_test(NAME pixel_aot_region_scalar_alu_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-region-scalar-alu "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_region_scalar_alu_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_region_scalar_alu_${adapter}.txt")
        add_test(NAME pixel_aot_region_scalar_load_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-region-scalar-load "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_region_scalar_load_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_region_scalar_load_${adapter}.txt")
        add_test(NAME pixel_position_centers_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_position_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_position_centers_${adapter} PROPERTIES TIMEOUT 30 SKIP_RETURN_CODE 77)
        add_test(NAME pixel_aot_buffer_color_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" buffer-color "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_buffer_color_${adapter} PROPERTIES TIMEOUT 60)
        add_test(NAME depth_mask_draw_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/depth_mask_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(depth_mask_draw_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME depth_mask_ops_draw_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/depth_mask_check.py" ops "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(depth_mask_ops_draw_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77)
        add_test(NAME depth_mask_cmpx_draw_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/depth_mask_check.py" cmpx "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(depth_mask_cmpx_draw_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77)
        add_test(NAME depth_mask_unary_draw_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/depth_mask_check.py" unary "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(depth_mask_unary_draw_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77)
        add_test(NAME pixel_cb_read_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_cb_read_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_cb_read_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77)
        add_test(NAME pixel_aot_u32_from_f32_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_uint_conversion_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_u32_from_f32_${adapter} PROPERTIES TIMEOUT 180 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_u32_from_f32_${adapter}.txt")
        add_test(NAME sampled_coverage_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/sampled_coverage_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(sampled_coverage_${adapter} PROPERTIES TIMEOUT 150 SKIP_RETURN_CODE 77)
        add_test(NAME pixel_aot_attribute_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_attribute_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_attribute_${adapter} PROPERTIES TIMEOUT 45)
        add_test(NAME pixel_aot_dmask_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_dmask_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_dmask_${adapter} PROPERTIES TIMEOUT 45)
        add_test(NAME pixel_aot_sample_lod_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_sample_lod_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_sample_lod_${adapter} PROPERTIES TIMEOUT 60
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_sample_lod_${adapter}.txt")
        add_test(NAME pixel_aot_sample_offset_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_sample_offset_check.py" draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_sample_offset_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_sample_offset_${adapter}.txt")
        add_test(NAME pixel_aot_sample_gradient_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_sample_gradient_check.py" draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_sample_gradient_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_sample_gradient_${adapter}.txt")
        add_test(NAME pixel_aot_sample_coarse_gradient_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_sample_gradient_check.py" draw-coarse "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_sample_coarse_gradient_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_sample_coarse_gradient_${adapter}.txt")
        add_test(NAME pixel_aot_three_way_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" three-way "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_three_way_${adapter} PROPERTIES TIMEOUT 60)
        add_test(NAME pixel_aot_predicate_rounding_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-predicate-rounding "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_predicate_rounding_${adapter} PROPERTIES TIMEOUT 60)
        add_test(NAME pixel_aot_comparisons_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-comparisons "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_comparisons_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_comparisons_${adapter}.txt")
        add_test(NAME pixel_aot_get_lod_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_get_lod_check.py" draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_get_lod_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_get_lod_${adapter}.txt")
        add_test(NAME pixel_aot_blend_scalar_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-blend-scalar "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_blend_scalar_${adapter} PROPERTIES TIMEOUT 60)
        add_test(NAME pixel_aot_bfm_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-bfm "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_bfm_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME pixel_aot_brev_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-brev "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_brev_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_brev_${adapter}.txt")
        add_test(NAME pixel_aot_smem_pointers_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-smem-pointers "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_smem_pointers_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_smem_pointers_${adapter}.txt")
        add_test(NAME pixel_aot_cmp64_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-cmp64 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_cmp64_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_cmp64_${adapter}.txt")
        add_test(NAME pixel_aot_cmp_f64_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-cmp-f64 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_cmp_f64_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_cmp_f64_${adapter}.txt")
        add_test(NAME pixel_aot_shift64_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-shift64 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_shift64_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_shift64_${adapter}.txt")
        add_test(NAME pixel_aot_pk_fmac_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-pk-fmac "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_pk_fmac_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_pk_fmac_${adapter}.txt")
        add_test(NAME pixel_aot_plain_bitscan_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-plain-bitscan "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_plain_bitscan_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME pixel_aot_int16_vop3_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-int16-vop3 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_int16_vop3_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME pixel_aot_int16_ternary_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-int16-ternary "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_int16_ternary_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME pixel_aot_f16_arith_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-f16-arith "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_f16_arith_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77)
        add_test(NAME pixel_aot_f16_round_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-f16-round "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_f16_round_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME pixel_aot_f16_fmac_ldexp_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-f16-fmac-ldexp "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_f16_fmac_ldexp_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77)
        add_test(NAME pixel_aot_f16_cvt_int16_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-f16-cvt-int16 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_f16_cvt_int16_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME pixel_aot_f16_to_int16_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-f16-to-int16 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_f16_to_int16_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
        add_test(NAME pixel_aot_swap_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-swap "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_swap_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_swap_${adapter}.txt")
        add_test(NAME pixel_aot_cvt_float_int_e64_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-cvt-float-int-e64 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_cvt_float_int_e64_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_cvt_float_int_e64_${adapter}.txt")
        add_test(NAME pixel_aot_cvt_flr_rpi_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-cvt-flr-rpi "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_cvt_flr_rpi_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_cvt_flr_rpi_${adapter}.txt")
        add_test(NAME pixel_aot_flat_scratch_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-flat-scratch "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_flat_scratch_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_flat_scratch_${adapter}.txt")
        add_test(NAME pixel_aot_e64_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-e64 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_e64_${adapter} PROPERTIES TIMEOUT 60)
        add_test(NAME pixel_aot_e64_approx_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-e64-approx "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_e64_approx_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_e64_approx_${adapter}.txt")
        add_test(NAME pixel_aot_sdwa_unary_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-sdwa-unary "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_sdwa_unary_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_sdwa_unary_${adapter}.txt")
        add_test(NAME pixel_aot_mac_legacy_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-mac-legacy "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_mac_legacy_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_mac_legacy_${adapter}.txt")
        add_test(NAME pixel_aot_integer_nc_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-integer-nc "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_integer_nc_${adapter} PROPERTIES TIMEOUT 60)
        add_test(NAME pixel_aot_title_loads_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-title-loads "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_title_loads_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_title_loads_${adapter}.txt")
        add_test(NAME pixel_aot_align_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-align "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_align_${adapter} PROPERTIES TIMEOUT 60)
        add_test(NAME pixel_aot_vop3_ternary_int_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-vop3-ternary-int "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_vop3_ternary_int_${adapter} PROPERTIES TIMEOUT 60)
        add_test(NAME pixel_aot_e64_integer_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-e64-integer "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_e64_integer_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_e64_integer_${adapter}.txt")
        add_test(NAME pixel_aot_sdwa_integer_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-sdwa-integer "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_sdwa_integer_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_sdwa_integer_${adapter}.txt")
        add_test(NAME pixel_aot_e64_spellings_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-e64-spellings "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_e64_spellings_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_e64_spellings_${adapter}.txt")
        add_test(NAME pixel_aot_frexp_exp_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-frexp-exp "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_frexp_exp_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_frexp_exp_${adapter}.txt")
        add_test(NAME pixel_aot_first_bit_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-first-bit "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_first_bit_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_first_bit_${adapter}.txt")
        add_test(NAME pixel_aot_bitset64_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-bitset64 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_bitset64_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_bitset64_${adapter}.txt")
        add_test(NAME pixel_aot_carry_out_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-carry-out "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_carry_out_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_carry_out_${adapter}.txt")
        add_test(NAME pixel_aot_carry_in_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-carry-in "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_carry_in_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_carry_in_${adapter}.txt")
        add_test(NAME pixel_aot_mad64_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-mad64 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_mad64_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_mad64_${adapter}.txt")
        add_test(NAME pixel_aot_cvt_f64_int_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-cvt-f64-int "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_cvt_f64_int_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_cvt_f64_int_${adapter}.txt")
        add_test(NAME pixel_aot_cvt_f64_f32_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-cvt-f64-f32 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_cvt_f64_f32_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_cvt_f64_f32_${adapter}.txt")
        add_test(NAME pixel_aot_cvt_int_f64_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-cvt-int-f64 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_cvt_int_f64_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_cvt_int_f64_${adapter}.txt")
        add_test(NAME pixel_aot_sad_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-sad "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_sad_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_sad_${adapter}.txt")
        add_test(NAME pixel_aot_rndne_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_rndne_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_rndne_${adapter} PROPERTIES TIMEOUT 60
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_rndne_${adapter}.txt")
        add_test(NAME pixel_aot_pack_int16_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-pack-int16 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_pack_int16_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_pack_int16_${adapter}.txt")
        add_test(NAME pixel_aot_pack_b32_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-pack-b32 "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_pack_b32_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_pack_b32_${adapter}.txt")
        add_test(NAME pixel_aot_compare_sample_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_compare_sample_check.py" draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" "${adapter}")
        set_tests_properties(pixel_aot_compare_sample_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_compare_sample_${adapter}.txt")
        add_test(NAME pixel_aot_gather_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_gather_check.py" --draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_gather_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_gather_${adapter}.txt")
        add_test(NAME pixel_aot_compare_gather_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_compare_gather_check.py" --draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" "${adapter}")
        set_tests_properties(pixel_aot_compare_gather_${adapter} PROPERTIES TIMEOUT 120 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_compare_gather_${adapter}.txt")
        add_test(NAME pixel_aot_compare_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_compare_check.py" draw "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_compare_${adapter} PROPERTIES TIMEOUT 300 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_compare_${adapter}.txt")
        add_test(NAME bc7_draw_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/bc7_draw_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(bc7_draw_${adapter} PROPERTIES TIMEOUT 60)
        add_test(NAME bc7_mode6_multiblock_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/bc7_mode6_multiblock_check.py" "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(bc7_mode6_multiblock_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
    endforeach()
    foreach(adapter IN ITEMS hardware warp)
        add_test(NAME pixel_aot_div_fmas_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" draw-div-fmas "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_div_fmas_${adapter} PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_div_fmas_${adapter}.txt")
        add_test(NAME pixel_aot_bits_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/pixel_aot_check.py" bits "${DXC_EXECUTABLE}"
            "$<TARGET_FILE:offscreen_check>" ${adapter})
        set_tests_properties(pixel_aot_bits_${adapter} PROPERTIES TIMEOUT 180
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_bits_${adapter}.txt")
    endforeach()
    set_tests_properties(pixel_aot_bits_hardware PROPERTIES SKIP_RETURN_CODE 77)
    set_tests_properties(cpu_native_frame_hardware pixel_aot_hardware pixel_aot_buffer_color_hardware
        pixel_aot_three_way_hardware pixel_aot_e64_hardware pixel_aot_blend_scalar_hardware pixel_aot_predicate_rounding_hardware bc7_draw_hardware
        pixel_aot_integer_nc_hardware pixel_aot_align_hardware pixel_aot_vop3_ternary_int_hardware pixel_aot_flat_scratch_hardware
        PROPERTIES SKIP_RETURN_CODE 77)
    set_tests_properties(compute_store_aot_hardware compute_store_aot_guard_read_hardware compute_store_aot_f32_hardware
        PROPERTIES SKIP_RETURN_CODE 77)
    foreach(adapter IN ITEMS hardware warp)
        # The gfx programs this test translated, for tools/form_coverage.py.
        set_tests_properties(pixel_aot_${adapter} PROPERTIES
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_${adapter}.txt")
        set_tests_properties(pixel_aot_predicate_rounding_${adapter} PROPERTIES
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_predicate_rounding_${adapter}.txt")
        set_tests_properties(pixel_aot_blend_scalar_${adapter} PROPERTIES
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_blend_scalar_${adapter}.txt")
        set_tests_properties(pixel_aot_plain_bitscan_${adapter} PROPERTIES
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_plain_bitscan_${adapter}.txt")
        set_tests_properties(pixel_aot_e64_${adapter} PROPERTIES
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_e64_${adapter}.txt")
        set_tests_properties(pixel_aot_integer_nc_${adapter} PROPERTIES
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_integer_nc_${adapter}.txt")
        set_tests_properties(pixel_aot_align_${adapter} PROPERTIES
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_align_${adapter}.txt")
        set_tests_properties(pixel_aot_int16_vop3_${adapter} PROPERTIES
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_int16_vop3_${adapter}.txt")
        set_tests_properties(pixel_aot_int16_ternary_${adapter} PROPERTIES
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_int16_ternary_${adapter}.txt")
        set_tests_properties(pixel_aot_f16_arith_${adapter} PROPERTIES
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_f16_arith_${adapter}.txt")
        set_tests_properties(pixel_aot_f16_round_${adapter} PROPERTIES
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_f16_round_${adapter}.txt")
        set_tests_properties(pixel_aot_f16_fmac_ldexp_${adapter} PROPERTIES
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_f16_fmac_ldexp_${adapter}.txt")
        set_tests_properties(pixel_aot_f16_cvt_int16_${adapter} PROPERTIES
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_f16_cvt_int16_${adapter}.txt")
        set_tests_properties(pixel_aot_f16_to_int16_${adapter} PROPERTIES
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_f16_to_int16_${adapter}.txt")
        set_tests_properties(pixel_aot_vop3_ternary_int_${adapter} PROPERTIES
            ENVIRONMENT "FORM_COVERAGE_RECORD=${CMAKE_BINARY_DIR}/form_coverage/pixel_aot_vop3_ternary_int_${adapter}.txt")
    endforeach()
    set_tests_properties(pixel_aot_attribute_hardware pixel_aot_rndne_hardware PROPERTIES SKIP_RETURN_CODE 77)
    set_tests_properties(pixel_aot_dmask_hardware PROPERTIES SKIP_RETURN_CODE 77)
    set_tests_properties(pixel_aot_sample_lod_hardware PROPERTIES SKIP_RETURN_CODE 77)
    add_test(NAME compressed_scene_metadata COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/scene_records_check.py" "${CMAKE_CURRENT_BINARY_DIR}/DualUav.cso"
        "${CMAKE_CURRENT_BINARY_DIR}/triangle_ps.cso" "$<TARGET_FILE:root_probe>")
    add_test(NAME root_device_controls_warp COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tools/probe_root_devices.py" --warp-only --bin "$<TARGET_FILE_DIR:root_device_probe>")
    add_test(NAME shader_metadata_probe COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/shader_probe_check.py" "$<TARGET_FILE:root_probe>"
        "${CMAKE_CURRENT_BINARY_DIR}/compute_root.cso" "${CMAKE_CURRENT_BINARY_DIR}/compute_tables.cso"
        "${CMAKE_CURRENT_BINARY_DIR}/triangle_vs.cso" "${DXC_BIN_DIR}/dxil.dll")
    set_tests_properties(shader_metadata_probe PROPERTIES SKIP_RETURN_CODE 77)
    add_test(NAME shader_metadata_controls COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/shader_probe_check.py")
    add_test(NAME shader_metadata_census COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/shader_census_check.py")
    # Keep both shader-model profiles in the retarget probe even though the draw fixture uses SM 6.0.
    set(retarget_copy_blocks "${CMAKE_CURRENT_BINARY_DIR}/CopyBlocksSM65.cso")
    add_custom_command(OUTPUT "${retarget_copy_blocks}"
        COMMAND "${DXC_EXECUTABLE}" -T cs_6_5 -E CopyBlocks -Fo "${retarget_copy_blocks}"
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/buffer_ops.hlsl"
        DEPENDS tests/buffer_ops.hlsl VERBATIM)
    add_custom_target(dxil_retarget_fixture DEPENDS "${retarget_copy_blocks}")
    add_dependencies(root_device_probe dxil_retarget_fixture)
    add_test(NAME dxil_retarget_probe COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/dxil_retarget_check.py" "$<TARGET_FILE:root_probe>"
        "${CMAKE_CURRENT_BINARY_DIR}/compute_root.cso" "${retarget_copy_blocks}"
        "${DXC_BIN_DIR}/dxil.dll" "$<TARGET_FILE:root_device_probe>" "$<TARGET_FILE:offscreen_check>")
    set_tests_properties(dxil_retarget_probe PROPERTIES SKIP_RETURN_CODE 77)
    add_test(NAME dxil_retarget_probe_hardware COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/dxil_retarget_check.py" "$<TARGET_FILE:root_probe>"
        "${CMAKE_CURRENT_BINARY_DIR}/compute_root.cso" "${retarget_copy_blocks}"
        "${DXC_BIN_DIR}/dxil.dll" "$<TARGET_FILE:root_device_probe>" "$<TARGET_FILE:offscreen_check>" hardware)
    set_tests_properties(dxil_retarget_probe_hardware PROPERTIES TIMEOUT 60 SKIP_RETURN_CODE 77)
    add_test(NAME dxil_retarget_controls COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/dxil_retarget_check.py")
    add_test(NAME rectlist_topology COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/rectlist_topology_check.py")
    foreach(adapter IN ITEMS warp hardware)
        add_test(NAME rectlist_draw_${adapter} COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/rectlist_topology_check.py"
            --draw "$<TARGET_FILE:rectlist_draw_check>" ${adapter})
        set_tests_properties(rectlist_draw_${adapter} PROPERTIES TIMEOUT 60)
    endforeach()
    set_tests_properties(rectlist_draw_hardware PROPERTIES SKIP_RETURN_CODE 77)
    add_test(NAME conditional_import_paths COMMAND "${INTAKE_PYTHON}" -B "${CMAKE_SOURCE_DIR}/tests/intake/paths_check.py")
    add_library(intake_provider SHARED tests/intake/provider.cpp tests/intake/provider.def)
    add_library(intake_delay SHARED tests/intake/provider.cpp tests/intake/delay.def)
    add_executable(intake_candidate tests/intake/candidate.cpp)
    # Build an import library from a name only. No graphics DLL is linked or executed at build time.
    set(graphics_import "${CMAKE_CURRENT_BINARY_DIR}/intake_graphics.lib")
    add_custom_command(OUTPUT "${graphics_import}"
        COMMAND "${CMAKE_AR}" /nologo /machine:x64 "/def:${CMAKE_CURRENT_SOURCE_DIR}/tests/intake/graphics_import.def"
                "/out:${graphics_import}"
        DEPENDS tests/intake/graphics_import.def VERBATIM)
    target_sources(intake_candidate PRIVATE "${graphics_import}")
    target_link_libraries(intake_candidate PRIVATE intake_provider intake_delay delayimp "${graphics_import}")
    target_link_options(intake_candidate PRIVATE /DELAYLOAD:intake_delay.dll)
    add_test(NAME static_exe_intake COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/intake/check.py" "$<TARGET_FILE:intake_candidate>"
        "$<TARGET_FILE:intake_provider>")
    # intake_forward exports six placeholders. The test rewrites a copy so each export is a forwarder;
    # it reads the images and never loads them.
    add_library(intake_forward SHARED tests/intake/provider.cpp tests/intake/forward.def)
    add_executable(intake_forward_caller tests/intake/forward_caller.cpp)
    target_link_libraries(intake_forward_caller PRIVATE intake_forward)
    add_test(NAME bundled_export_forwarders COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/intake/forwarders_check.py" "$<TARGET_FILE:intake_forward_caller>"
        "$<TARGET_FILE:intake_forward>" "$<TARGET_FILE:intake_provider>")
    add_test(NAME layout_chunk_summary COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/layout_chunks_check.py")
    add_test(NAME dependency_paths_controls COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/dependency_paths_check.py")
    add_test(NAME pe_package_import_closure COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/pe_package_preflight_check.py")
    add_test(NAME pe_run_package_invalid_pe COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/pe_run_package_invalid_check.py")
    add_test(NAME pe_host_commit_preflight COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/host_commit_check.py")
    add_test(NAME xgameruntime_preflight_stub_refused COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/xgameruntime_preflight_check.py"
        "$<TARGET_FILE:xgameruntime_preflight_exe>" "$<TARGET_FILE:xmem_shim>"
        "$<TARGET_FILE:d3d12_xs_shim>" "$<TARGET_FILE_DIR:xgameruntime_shim>"
        "$<TARGET_FILE_DIR:xmem_shim>")
    set_tests_properties(xgameruntime_preflight_stub_refused PROPERTIES TIMEOUT 30)
    add_test(NAME pe_run_package_trap_missing COMMAND "${INTAKE_PYTHON}" -B
        "${CMAKE_SOURCE_DIR}/tests/trap_missing_check.py" "$<TARGET_FILE:pe_map>"
        "$<TARGET_FILE:trap_missing_exe>" "$<TARGET_FILE:trap_missing_data_exe>")
    set_tests_properties(pe_run_package_trap_missing PROPERTIES TIMEOUT 60)
else()
    message(WARNING
        "Python-driven tests, including the CPU fixtures, are disabled because "
        "INTAKE_PYTHON was not found at '${INTAKE_PYTHON}'. "
        "Follow the Python setup under First build in README.md, or set INTAKE_PYTHON "
        "to an interpreter with requirements-dev.txt installed. Then rerun CMake.")
endif()

set(vm_raw_srv_header "${CMAKE_CURRENT_BINARY_DIR}/vm_raw_srv_cs.h")
add_custom_command(OUTPUT "${vm_raw_srv_header}"
    COMMAND "${DXC_EXECUTABLE}" -T cs_6_0 -E main -Vn vm_raw_srv_cs -Fh "${vm_raw_srv_header}"
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/vm_raw_srv.hlsl"
    DEPENDS tests/vm_raw_srv.hlsl VERBATIM)
set(vm_raw_uav_header "${CMAKE_CURRENT_BINARY_DIR}/vm_raw_uav_cs.h")
add_custom_command(OUTPUT "${vm_raw_uav_header}"
    COMMAND "${DXC_EXECUTABLE}" -T cs_6_0 -E main -Vn vm_raw_uav_cs -Fh "${vm_raw_uav_header}"
            "${CMAKE_CURRENT_SOURCE_DIR}/tests/vm_raw_uav.hlsl"
    DEPENDS tests/vm_raw_uav.hlsl VERBATIM)
add_executable(vm_copy_queue_check tests/vm_copy_queue_check.cpp "${vm_raw_srv_header}" "${vm_raw_uav_header}")
target_include_directories(vm_copy_queue_check PRIVATE src "${CMAKE_CURRENT_BINARY_DIR}")
target_compile_features(vm_copy_queue_check PRIVATE cxx_std_17)
target_link_libraries(vm_copy_queue_check PRIVATE d3d12 dxgi OneCore)
foreach(adapter IN ITEMS warp hardware)
    add_test(NAME vm_copy_queue_${adapter} COMMAND vm_copy_queue_check ${adapter})
    add_test(NAME vm_page_provenance_${adapter} COMMAND vm_copy_queue_check provenance_${adapter})
    add_test(NAME vm_raw_srv_${adapter} COMMAND vm_copy_queue_check raw_srv_${adapter})
    add_test(NAME vm_raw_uav_${adapter} COMMAND vm_copy_queue_check raw_uav_${adapter})
    set_tests_properties(vm_copy_queue_${adapter} vm_page_provenance_${adapter} vm_raw_srv_${adapter}
        vm_raw_uav_${adapter}
        PROPERTIES TIMEOUT 30 SKIP_RETURN_CODE 77)
endforeach()
# Window validation and touching-window boundaries of resolve_raw_uav_store (src/raw_buffer_uav.h),
# without a device. The mutated run is the negative control.
add_executable(raw_buffer_uav_check tests/raw_buffer_uav_check.cpp)
target_include_directories(raw_buffer_uav_check PRIVATE src)
target_compile_features(raw_buffer_uav_check PRIVATE cxx_std_17)
add_test(NAME raw_buffer_uav COMMAND raw_buffer_uav_check)
add_test(NAME raw_buffer_uav_adjacent_control COMMAND raw_buffer_uav_check --mutate adjacent)
set_tests_properties(raw_buffer_uav_adjacent_control PROPERTIES WILL_FAIL TRUE)

if(EXISTS "${INTAKE_PYTHON}")
    add_executable(rcas_coordinate_readback tests/rcas_coordinate_readback/readback.cpp)
    target_link_libraries(rcas_coordinate_readback PRIVATE d3d11 d3dcompiler)
    foreach(control IN ITEMS positive negative)
        add_test(NAME rcas_coordinate_${control}_warp COMMAND "${INTAKE_PYTHON}" -B
            "${CMAKE_SOURCE_DIR}/tests/rcas_coordinate_readback/verify.py" "${CMAKE_SOURCE_DIR}"
            --host $<TARGET_FILE:rcas_coordinate_readback> --backend warp --case ${control})
    endforeach()
endif()
