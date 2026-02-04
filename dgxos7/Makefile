PACKER ?= packer
ISO ?= ${DGXOS7_ISO_PATH}
CHECKSUM ?= ${DGXOS7_SHA256SUM}
PLATFORM ?= dgx_h100

.PHONY: all clean check-iso

all: dgxos7.tar.gz

dgxos7.tar.gz: check-iso clean
	sudo PACKER_LOG=1 ${PACKER} build \
		-var "dgxos7_iso=${ISO}" \
		-var "dgxos7_sha256sum=${CHECKSUM}" \
		-var "platform=${PLATFORM}" \
		dgxos7.json
	reset

check-iso:
	@if [ -z "${ISO}" ]; then \
		echo "Error: DGXOS7_ISO_PATH environment variable not set"; \
		echo "Please set DGXOS7_ISO_PATH to the path of your DGX OS 7 ISO"; \
		exit 1; \
	fi
	@if [ ! -f "${ISO}" ]; then \
		echo "Error: ISO file not found at ${ISO}"; \
		exit 1; \
	fi
	@if [ -z "${CHECKSUM}" ]; then \
		echo "Error: DGXOS7_SHA256SUM environment variable not set"; \
		echo "Please set DGXOS7_SHA256SUM to the SHA256 checksum of your ISO"; \
		echo "You can generate it with: sha256sum ${ISO}"; \
		exit 1; \
	fi

clean:
	sudo ${RM} -rf output-qemu dgxos7.tar.gz
