cmake_minimum_required(VERSION 3.14)
project(blink-plugin)

set(PLUGIN_OUTPUT_DIR bin CACHE PATH "plugin output directory")
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_LIST_DIR}/cmake/Modules)

include(CMakeRC)

if (NOT TARGET ent::ent)
	add_subdirectory(../extern/submodules/ent ent)
endif()
if (NOT TARGET snd::snd)
	add_subdirectory(../extern/submodules/snd snd)
endif()

if (NOT TARGET blink::adhoc::madronalib)
	add_library(blink-adhoc-madronalib INTERFACE)
	add_library(blink::adhoc::madronalib ALIAS blink-adhoc-madronalib)
	target_sources(blink-adhoc-madronalib INTERFACE
		FILE_SET blink_adhoc_madronalib
		TYPE HEADERS
		BASE_DIRS
			../extern/submodules/madronalib/source
			../extern/submodules/madronalib/source/app
	)
endif()

if (NOT TARGET blink::plugin)
	add_library(blink-plugin INTERFACE)
	add_library(blink::plugin ALIAS blink-plugin)
	target_sources(blink-plugin INTERFACE
		FILE_SET blink_plugin
		TYPE HEADERS
		BASE_DIRS
			..
			../../include
			$<$<PLATFORM_ID:Darwin>:../extern/submodules/sse2neon>
	)
	target_link_libraries(blink-plugin INTERFACE
		blink::adhoc::madronalib
		ent::ent
		snd::snd
	)
endif()

function(blink_plugin_get_base_filename out_var name)
  set(on_windows $<STREQUAL:${CMAKE_SYSTEM_NAME},Windows>)
  set(on_macos $<STREQUAL:${CMAKE_SYSTEM_NAME},Darwin>)
  set(on_linux $<STREQUAL:${CMAKE_SYSTEM_NAME},Linux>)
  set(debug_build $<OR:$<CONFIG:Debug>,$<STREQUAL:${CMAKE_BUILD_TYPE},Debug}>>)
  set(PLATFORM_DIR $<IF:${on_windows},win64,$<IF:${on_macos},macos,linux>>)
	set(${out_var} ${name}.v${PROJECT_VERSION}.${PLATFORM_DIR}$<${debug_build}:.dbg> PARENT_SCOPE)
endfunction()

function(blink_plugin_get_target_name out_var name)
	set(${out_var} ${name} PARENT_SCOPE)
endfunction()

function(blink_clib_add name src)
	blink_plugin_get_base_filename(output_name ${name})
	blink_plugin_get_target_name(target_name ${name})
	add_library(${target_name} STATIC ${src})
	#target_link_libraries(${target_name} PRIVATE blink::plugin ${target_name}::rc)
	target_include_directories(${target_name} PRIVATE ${extra_include_dirs})
	set_target_properties(${target_name} PROPERTIES
		OUTPUT_NAME ${output_name}
		FOLDER "plugins/"
		PREFIX ""
		SUFFIX ".blink"
		RUNTIME_OUTPUT_DIRECTORY_DEBUG          ${PLUGIN_OUTPUT_DIR}
		RUNTIME_OUTPUT_DIRECTORY_RELEASE        ${PLUGIN_OUTPUT_DIR}
		RUNTIME_OUTPUT_DIRECTORY                ${PLUGIN_OUTPUT_DIR}
		RUNTIME_OUTPUT_DIRECTORY_RELWITHDEBINFO ${PLUGIN_OUTPUT_DIR}
		LIBRARY_OUTPUT_DIRECTORY_DEBUG          ${PLUGIN_OUTPUT_DIR}
		LIBRARY_OUTPUT_DIRECTORY_RELEASE        ${PLUGIN_OUTPUT_DIR}
		LIBRARY_OUTPUT_DIRECTORY_RELWITHDEBINFO ${PLUGIN_OUTPUT_DIR}
		LIBRARY_OUTPUT_DIRECTORY                ${PLUGIN_OUTPUT_DIR}
		CXX_STANDARD 17
	)
	target_compile_definitions(
		${target_name} PRIVATE
		PLUGIN_VERSION="${PROJECT_VERSION}"
		_USE_MATH_DEFINES
	)
endfunction()

source_group(src REGULAR_EXPRESSION "/src/.*\\.(h|cpp)")