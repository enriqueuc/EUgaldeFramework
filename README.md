# EUgaldeFramework

How to create a Framework

Open XCode, then File -  New - Project.

Choose iOS - Cocoa Touch Static Library.

Put the name you want, select the location and click create.

If you want, you can delete the .m file and the whole content of the .h file.

Create a new Cocoa Touch Class and name it as you want.

In the main header (the one named as your framework) import your new class header like:
	
	#import <mainHeader/newClassHeader.h>
	
Add some content to your framework, in this case we need to do a class instance to the fw works properly.

Go to the Framweork Target to add the headers you are going to expose.

Select the Bluid phases tab of your Framework Target, click the + button on the left top corner and select New Headers Phase.

Open the Headers row that was created and click the + button, the select your headers files and click add, this will add your .h files to the project row; finally move the .h files you want to be public to the public row by click and drag.

To change the public headers location select the Target in Project Navigator and click Build Settings tab, then search for Public Headers and replace the usr/local/include with "$(PROJECT_NAME)Headers" (without the "") for all configurations

Change the next configuration on your Target Build settings tab:
	
	"Dead Code Stripping" => No (for all settings)
	
	"Strip Debug Symbols During Copy" => No (for all settings)
	
	"Strip Style" => Non-Global Symbols (for all settings)

Select the PROJECT (NOT the target) and change the "Build Active Architecture Only" => No (for all settings)

In your target build phases create a script by clicking the + button and selecting New Run Script Phase. In the console view add the next script

	set -e

	mkdir -p "${BUILT_PRODUCTS_DIR}/${PRODUCT_NAME}.framework/Versions/A/Headers"

	# Link the "Current" version to "A"
	/bin/ln -sfh A "${BUILT_PRODUCTS_DIR}/${PRODUCT_NAME}.framework/Versions/Current"
	/bin/ln -sfh Versions/Current/Headers "${BUILT_PRODUCTS_DIR}/${PRODUCT_NAME}.framework/Headers"
	/bin/ln -sfh "Versions/Current/${PRODUCT_NAME}" "${BUILT_PRODUCTS_DIR}/${PRODUCT_NAME}.framework/${PRODUCT_NAME}"

	# The -a ensures that the headers maintain the source modification date so that we don't constantly
	# cause propagating rebuilds of files that import these headers.
	/bin/cp -a "${TARGET_BUILD_DIR}/${PUBLIC_HEADERS_FOLDER_PATH}/" "${BUILT_PRODUCTS_DIR}/${PRODUCT_NAME}.framework/Versions/A/Headers"

Finally Build your project

This will create the .framework and headers files, but we need to create the Target by selecting File - New - Target - Cross-Platform (tab) - Aggregate. Give a different name such like "Framework"

On your Framework Target (the aggregate one) add a new Target Dependencies that would be your Framework Static Library

Click the + button and select New Run Script Phase. In the console view add the next script

	set -e
	set +u
	# Avoid recursively calling this script.
	if [[ $SF_MASTER_SCRIPT_RUNNING ]]
	then
	exit 0
	fi
	set -u
	export SF_MASTER_SCRIPT_RUNNING=1

	SF_TARGET_NAME=${PROJECT_NAME}
	SF_EXECUTABLE_PATH="lib${SF_TARGET_NAME}.a"
	SF_WRAPPER_NAME="${SF_TARGET_NAME}.framework"

	# The following conditionals come from
	# https://github.com/kstenerud/iOS-Universal-Framework

	if [[ "$SDK_NAME" =~ ([A-Za-z]+) ]]
	then
	SF_SDK_PLATFORM=${BASH_REMATCH[1]}
	else
	echo "Could not find platform name from SDK_NAME: $SDK_NAME"
	exit 1
	fi

	if [[ "$SDK_NAME" =~ ([0-9]+.*$) ]]
	then
	SF_SDK_VERSION=${BASH_REMATCH[1]}
	else
	echo "Could not find sdk version from SDK_NAME: $SDK_NAME"
	exit 1
	fi

	if [[ "$SF_SDK_PLATFORM" = "iphoneos" ]]
	then
	SF_OTHER_PLATFORM=iphonesimulator
	else
	SF_OTHER_PLATFORM=iphoneos
	fi

	if [[ "$BUILT_PRODUCTS_DIR" =~ (.*)$SF_SDK_PLATFORM$ ]]
	then
	SF_OTHER_BUILT_PRODUCTS_DIR="${BASH_REMATCH[1]}${SF_OTHER_PLATFORM}"
	else
	echo "Could not find platform name from build products directory: $BUILT_PRODUCTS_DIR"
	exit 1
	fi

	# Build the other platform.
	xcrun xcodebuild -project "${PROJECT_FILE_PATH}" -target "${TARGET_NAME}" -configuration "${CONFIGURATION}" -sdk ${SF_OTHER_PLATFORM}${SF_SDK_VERSION} BUILD_DIR="${BUILD_DIR}" OBJROOT="${OBJROOT}" BUILD_ROOT="${BUILD_ROOT}" SYMROOT="${SYMROOT}" $ACTION

	# Smash the two static libraries into one fat binary and store it in the .framework
	xcrun lipo -create "${BUILT_PRODUCTS_DIR}/${SF_EXECUTABLE_PATH}" "${SF_OTHER_BUILT_PRODUCTS_DIR}/${SF_EXECUTABLE_PATH}" -output "${BUILT_PRODUCTS_DIR}/${SF_WRAPPER_NAME}/Versions/A/${SF_TARGET_NAME}"

	# Copy the binary to the other architecture folder to have a complete framework in both.
	cp -a "${BUILT_PRODUCTS_DIR}/${SF_WRAPPER_NAME}/Versions/A/${SF_TARGET_NAME}" "${SF_OTHER_BUILT_PRODUCTS_DIR}/${SF_WRAPPER_NAME}/Versions/A/${SF_TARGET_NAME}"

The final step is to compile and build your whole project, first compile and build your Framework Static Library and then your Target
