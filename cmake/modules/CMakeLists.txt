# Install the torch-mlir libraries into the TorchMLIRTargets export set. A
# companion change generates the find_package(TorchMLIR) package from it.

# All of torch-mlir's libraries (recorded in MLIR_ALL_LIBS by add_mlir_library)
# are named TorchMLIR*.
get_property(_all_libs GLOBAL PROPERTY MLIR_ALL_LIBS)
set(TORCH_MLIR_EXPORTED_TARGETS "")
foreach(_lib IN LISTS _all_libs)
  if(_lib MATCHES "^TorchMLIR")
    list(APPEND TORCH_MLIR_EXPORTED_TARGETS ${_lib})
  endif()
endforeach()
list(REMOVE_DUPLICATES TORCH_MLIR_EXPORTED_TARGETS)

# StableHLO ships no CMake package and is built EXCLUDE_FROM_ALL (its install
# rules are skipped), so bundle the StableHLO libraries torch-mlir links to keep
# the export self-contained. Collect them by walking the link closure of our
# libraries for reachable non-imported, non-MLIR/LLVM, non-torch targets;
# imported MLIR (standalone) and MLIR/LLVM by name (in-tree) are excluded. Empty
# when STABLEHLO is off.
function(_torch_mlir_collect_bundled_deps out_var)
  set(_seen "")
  set(_bundled "")
  set(_worklist "${ARGN}")
  while(_worklist)
    set(_next "")
    foreach(_t IN LISTS _worklist)
      if(NOT TARGET ${_t} OR ${_t} IN_LIST _seen)
        continue()
      endif()
      list(APPEND _seen ${_t})

      get_target_property(_imported ${_t} IMPORTED)
      if(_imported OR _t MATCHES "^MLIR" OR _t MATCHES "^LLVM")
        continue()
      endif()
      if(NOT _t MATCHES "^TorchMLIR")
        list(APPEND _bundled ${_t})
      endif()

      set(_deps "")
      get_target_property(_il ${_t} INTERFACE_LINK_LIBRARIES)
      if(_il)
        list(APPEND _deps ${_il})
      endif()
      get_target_property(_type ${_t} TYPE)
      if(NOT _type STREQUAL "INTERFACE_LIBRARY")
        get_target_property(_ll ${_t} LINK_LIBRARIES)
        if(_ll)
          list(APPEND _deps ${_ll})
        endif()
      endif()
      foreach(_d IN LISTS _deps)
        if(TARGET ${_d} AND NOT ${_d} IN_LIST _seen)
          list(APPEND _next ${_d})
        endif()
      endforeach()
    endforeach()
    set(_worklist "${_next}")
  endwhile()
  list(REMOVE_DUPLICATES _bundled)
  set(${out_var} "${_bundled}" PARENT_SCOPE)
endfunction()

_torch_mlir_collect_bundled_deps(_bundled_deps ${TORCH_MLIR_EXPORTED_TARGETS})

set(_export_targets ${TORCH_MLIR_EXPORTED_TARGETS} ${_bundled_deps})
list(REMOVE_DUPLICATES _export_targets)

# add_mlir_library already installs the torch libraries (into MLIRTargets);
# re-installing is idempotent and additionally registers them in TorchMLIRTargets
# and installs the bundled StableHLO archives. A target may join two export sets.
# The torch-mlir-opt/lsp-server tools already tag TorchMLIRTargets themselves.
install(TARGETS ${_export_targets}
  EXPORT TorchMLIRTargets
  ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
  LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
  RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}")
