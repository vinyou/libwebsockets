project(lws-minimal-secure-streams C)
cmake_minimum_required(VERSION 2.8)
find_package(libwebsockets CONFIG REQUIRED)
list(APPEND CMAKE_MODULE_PATH ${LWS_CMAKE_DIR})
include(CheckCSourceCompiles)
include(LwsCheckRequirements)

set(SAMP lws-minimal-secure-streams)

set(requirements 1)
require_lws_config(LWS_ROLE_H1 1 requirements)
require_lws_config(LWS_WITHOUT_CLIENT 0 requirements)
require_lws_config(LWS_WITH_SECURE_STREAMS 1 requirements)
require_lws_config(LWS_WITH_SECURE_STREAMS_STATIC_POLICY_ONLY 0 requirements)
require_lws_config(LWS_WITH_SYS_STATE 1 requirements)

if (requirements)
	add_executable(${SAMP} minimal-secure-streams.c)

	if (LWS_CTEST_INTERNET_AVAILABLE AND NOT WIN32)
		add_test(NAME ss-warmcat COMMAND lws-minimal-secure-streams)
		set_tests_properties(ss-warmcat
				     PROPERTIES
				     WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/minimal-examples/secure-streams/minimal-secure-streams
				     TIMEOUT 20)

		if (HAS_LWS_WITH_SECURE_STREAMS_PROXY_API OR LWS_WITH_SECURE_STREAMS_PROXY_API)

			#
			# Define test dep to bring up and take down the test
			# proxy
			#

			if (${CMAKE_SYSTEM_NAME} MATCHES "Linux")
				# uds abstract namespace for linux
				set(CTEST_SOCKET_PATH "@ctest-ssp-$ENV{SAI_PROJECT}-$ENV{SAI_OVN}")
			else()
				# filesystem socket for others
				set(CTEST_SOCKET_PATH "/tmp/ctest-ssp-$ENV{SAI_PROJECT}-$ENV{SAI_OVN}")
			endif()
			add_test(NAME st_ssproxy COMMAND
				${CMAKE_SOURCE_DIR}/scripts/ctest-background.sh
				ssproxy $<TARGET_FILE:lws-minimal-secure-streams-proxy>
				-i ${CTEST_SOCKET_PATH} --ignore-sigterm)
			set_tests_properties(st_ssproxy PROPERTIES WORKING_DIRECTORY . FIXTURES_SETUP ssproxy TIMEOUT 800)

			add_test(NAME ki_ssproxy COMMAND
				${CMAKE_SOURCE_DIR}/scripts/ctest-background-kill.sh
				ssproxy $<TARGET_FILE:lws-minimal-secure-streams-proxy>
				-i ${CTEST_SOCKET_PATH})
			set_tests_properties(ki_ssproxy PROPERTIES FIXTURES_CLEANUP ssproxy)

			#
			# the client part that will connect to the proxy
			#

			add_test(NAME sspc-minimal COMMAND lws-minimal-secure-streams-client -i +${CTEST_SOCKET_PATH})
			set_tests_properties(sspc-minimal PROPERTIES
				WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/minimal-examples/secure-streams/minimal-secure-streams
				FIXTURES_REQUIRED "ssproxy"
				TIMEOUT 40)

		endif()

	endif()

	if (websockets_shared)
		target_link_libraries(${SAMP} websockets_shared ${LIBWEBSOCKETS_DEP_LIBS})
		add_dependencies(${SAMP} websockets_shared)
	else()
		target_link_libraries(${SAMP} websockets ${LIBWEBSOCKETS_DEP_LIBS})
	endif()

	CHECK_C_SOURCE_COMPILES("#include <libwebsockets.h>\nint main(void) {\ni#if defined(LWS_WITH_SECURE_STREAMS_PROXY_API)\n return 0;\n #else\n fail\n #endif\n return 0;\n}\n" HAS_LWS_WITH_SECURE_STREAMS_PROXY_API)

	if (HAS_LWS_WITH_SECURE_STREAMS_PROXY_API OR LWS_WITH_SECURE_STREAMS_PROXY_API)
		add_compile_options(-DLWS_SS_USE_SSPC)

		add_executable(${SAMP}-client minimal-secure-streams.c)
		if (websockets_shared)
			target_link_libraries(${SAMP}-client websockets_shared ${LIBWEBSOCKETS_DEP_LIBS})
			add_dependencies(${SAMP}-client websockets_shared)
		else()
			target_link_libraries(${SAMP}-client websockets ${LIBWEBSOCKETS_DEP_LIBS})
		endif()
	endif()

endif()
