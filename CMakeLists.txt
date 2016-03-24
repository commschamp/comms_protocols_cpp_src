cmake_minimum_required (VERSION 2.8)
project ("Practical Guide to Implementing Communication Protocols in C++" NONE)

set(IMAGES_DIR "image")
set(TEXT_DIR "text")
set(DIA_DIR "dia")
SET(PUBLISH_DIR "publish")

set(SITE_TARGET "site")
set(SITE_BASE_TARGET "site_base")
set(SITE_IMAGES_TARGET "site_images")
set(IMAGES_TARGET "images")
set(PUBLISH_TEXT_TARGET "publish_text")
set(PUBLISH_IMAGES_TARGET "publish_images")
set(PUBLISH_TARGET "publish")

set(SITE_TIMESTAMP_FILE "${CMAKE_BINARY_DIR}/site_created.txt")
set(PUBLISH_TEXT_TIMESTAMP_FILE "${CMAKE_BINARY_DIR}/publish_text.txt")

file (GLOB DIA_FILES ${CMAKE_CURRENT_SOURCE_DIR}/${DIA_DIR}/*.dia)
file (GLOB TEXT_FILES "${CMAKE_SOURCE_DIR}/${TEXT_DIR}/*")

###################################################################

function (images)
    set (images_targets)
    foreach (f ${DIA_FILES})
        string (REPLACE ".dia" ".png" temp ${f})
        string (REPLACE "${CMAKE_CURRENT_SOURCE_DIR}/${DIA_DIR}" "${CMAKE_BINARY_DIR}/${IMAGES_DIR}" image ${temp})
        string (REPLACE "${CMAKE_BINARY_DIR}/${IMAGES_DIR}/" "" target ${image})
        add_custom_command(
            OUTPUT "${image}"
            COMMAND ${CMAKE_COMMAND} -E make_directory "${CMAKE_BINARY_DIR}/${IMAGES_DIR}"
            COMMAND dia -e ${image} -t png-libart ${f}
            DEPENDS ${f}
        )

        add_custom_target(
            "${target}"
            DEPENDS "${image}"
        )

        set(images_targets ${images_targets} ${target})
    endforeach ()

    add_custom_target(
        ${IMAGES_TARGET} ALL
        DEPENDS ${images_targets}
    )

endfunction ()

###################################################################

function (site)
    add_custom_command(
        OUTPUT "${SITE_TIMESTAMP_FILE}"
        COMMAND gitbook build ${CMAKE_SOURCE_DIR}/${TEXT_DIR} ${CMAKE_BINARY_DIR}/${SITE_TARGET}
        COMMAND ${CMAKE_COMMAND} -E touch "${SITE_TIMESTAMP_FILE}"
        DEPENDS ${TEXT_FILES} ${IMAGES_TARGET}
    )

    add_custom_target(
        ${SITE_BASE_TARGET} ALL
        DEPENDS "${SITE_TIMESTAMP_FILE}"
    )

    add_custom_target(
        ${SITE_IMAGES_TARGET} ALL
        ${CMAKE_COMMAND} -E copy_directory ${CMAKE_BINARY_DIR}/${IMAGES_DIR} ${CMAKE_BINARY_DIR}/${SITE_TARGET}/${IMAGES_DIR}
        DEPENDS ${IMAGES_TARGET} ${SITE_BASE_TARGET}
    )

    add_custom_target(
        ${SITE_TARGET} ALL
        DEPENDS ${SITE_BASE_TARGET} ${SITE_IMAGES_TARGET}
    )

endfunction ()

###################################################################
function (publish)
    add_custom_command(
        OUTPUT "${PUBLISH_TEXT_TIMESTAMP_FILE}"
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/${TEXT_DIR} ${CMAKE_BINARY_DIR}/${PUBLISH_DIR}
        COMMAND ${CMAKE_COMMAND} -E touch "${PUBLISH_TEXT_TIMESTAMP_FILE}"
        DEPENDS ${TEXT_FILES} ${IMAGES_TARGET}
    )

    add_custom_target(
        ${PUBLISH_TEXT_TARGET} ALL
        DEPENDS "${PUBLISH_TEXT_TIMESTAMP_FILE}"
    )

    add_custom_target(
        ${PUBLISH_IMAGES_TARGET} ALL
        ${CMAKE_COMMAND} -E copy_directory ${CMAKE_BINARY_DIR}/${IMAGES_DIR} ${CMAKE_BINARY_DIR}/${PUBLISH_DIR}/${IMAGES_DIR}
        DEPENDS ${IMAGES_TARGET} ${PUBLISH_TEXT_TARGET}
    )

    add_custom_target(
        ${PUBLISH_TARGET} ALL
        DEPENDS ${PUBLISH_TEXT_TARGET} ${PUBLISH_IMAGES_TARGET}
    )

endfunction ()

###################################################################

images()
site()
publish()


