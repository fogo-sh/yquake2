# GL3 renderer (ref_gl3)

set(GL3-Source
	${REF_SRC_DIR}/gl3/gl3_draw.c
	${REF_SRC_DIR}/gl3/gl3_image.c
	${REF_SRC_DIR}/gl3/gl3_light.c
	${REF_SRC_DIR}/gl3/gl3_lightmap.c
	${REF_SRC_DIR}/gl3/gl3_main.c
	${REF_SRC_DIR}/gl3/gl3_mesh.c
	${REF_SRC_DIR}/gl3/gl3_misc.c
	${REF_SRC_DIR}/gl3/gl3_model.c
	${REF_SRC_DIR}/gl3/gl3_sdl.c
	${REF_SRC_DIR}/gl3/gl3_surf.c
	${REF_SRC_DIR}/gl3/gl3_warp.c
	${REF_SRC_DIR}/gl3/gl3_shaders.c
	${REF_SRC_DIR}/files/common.c
	${REF_SRC_DIR}/files/models.c
	${REF_SRC_DIR}/files/pcx.c
	${REF_SRC_DIR}/files/stb.c
	${REF_SRC_DIR}/files/surf.c
	${REF_SRC_DIR}/files/wal.c
	${REF_SRC_DIR}/files/pvs.c
	${COMMON_SRC_DIR}/shared/shared.c
	${COMMON_SRC_DIR}/md4.c
)

set(Glad-GL3-Source ${REF_SRC_DIR}/gl3/glad/src/glad.c)

set(GL3-Header
	${REF_SRC_DIR}/ref_shared.h
	${REF_SRC_DIR}/constants/anorms.h
	${REF_SRC_DIR}/constants/anormtab.h
	${REF_SRC_DIR}/constants/warpsin.h
	${REF_SRC_DIR}/files/stb_image.h
	${REF_SRC_DIR}/gl3/header/DG_dynarr.h
	${REF_SRC_DIR}/gl3/header/HandmadeMath.h
	${REF_SRC_DIR}/gl3/header/local.h
	${REF_SRC_DIR}/gl3/header/model.h
	${COMMON_SRC_DIR}/header/shared.h
)

set(Glad-GL3-Header
	${REF_SRC_DIR}/gl3/glad/include/glad/glad.h
	${REF_SRC_DIR}/gl3/glad/include/KHR/khrplatform.h
)

add_library(ref_gl3 MODULE ${GL3-Source} ${Glad-GL3-Source} ${GL3-Header} ${Glad-GL3-Header} ${REF-Platform-Specific-Source})
set_target_properties(ref_gl3 PROPERTIES
		PREFIX ""
		LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
		RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
		SUFFIX ${CMAKE_SHARED_LIBRARY_SUFFIX}
)
target_include_directories(ref_gl3 PRIVATE ${CMAKE_SOURCE_DIR}/src/client/refresh/gl3/glad/include)
target_link_libraries(ref_gl3 ${yquake2LinkerFlags} ${yquake2SDLLinkerFlags})
if(SDL3_SUPPORT)
	target_link_libraries(ref_gl3 SDL3::SDL3)
endif()


