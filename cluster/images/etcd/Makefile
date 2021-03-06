# Copyright 2016 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Build the etcd image
#
# Usage:
# 	[BUNDLED_ETCD_VERSIONS=2.2.1 2.3.7 3.0.17 3.1.12 3.2.18] [REGISTRY=k8s.gcr.io] [ARCH=amd64] [BASEIMAGE=busybox] make (build|push)
#
# The image contains different etcd versions to simplify
# upgrades. Thus be careful when removing any versions from here.
#
# NOTE: The etcd upgrade rules are that you can upgrade only 1 minor
# version at a time, and patch release don't matter.
#
# Except from etcd-$(version) and etcdctl-$(version) binaries, we also
# need etcd and etcdctl binaries for backward compatibility reasons.
# That binary will be set to the last version from $(BUNDLED_ETCD_VERSIONS).
BUNDLED_ETCD_VERSIONS?=2.2.1 2.3.7 3.0.17 3.1.12 3.2.18

# LATEST_ETCD_VERSION identifies the most recent etcd version available.
LATEST_ETCD_VERSION?=3.2.18

# REVISION provides a version number fo this image and all it's bundled
# artifacts. It should start at zero for each LATEST_ETCD_VERSION and increment
# for each revision of this image at that etcd version.
REVISION?=0

# IMAGE_TAG Uniquely identifies k8s.gcr.io/etcd docker image with a tag of the form "<etcd-version>-<revision>".
IMAGE_TAG=$(LATEST_ETCD_VERSION)-$(REVISION)

ARCH?=amd64
# Image should be pulled from k8s.gcr.io, which will auto-detect
# region (us, eu, asia, ...) and pull from the closest.
REGISTRY?=k8s.gcr.io
# Images should be pushed to staging-k8s.gcr.io.
PUSH_REGISTRY?=staging-k8s.gcr.io
# golang version should match the golang version from https://github.com/coreos/etcd/releases for the current ETCD_VERSION.
GOLANG_VERSION?=1.8.7
GOARM=7
TEMP_DIR:=$(shell mktemp -d)

ifeq ($(ARCH),amd64)
	BASEIMAGE?=busybox
endif
ifeq ($(ARCH),arm)
	BASEIMAGE?=arm32v7/busybox
endif
ifeq ($(ARCH),arm64)
	BASEIMAGE?=arm64v8/busybox
endif
ifeq ($(ARCH),ppc64le)
	BASEIMAGE?=ppc64le/busybox
endif
ifeq ($(ARCH),s390x)
	BASEIMAGE?=s390x/busybox
endif

build:
	# Copy the content in this dir to the temp dir,
	# without copying the subdirectories.
	find ./ -maxdepth 1 -type f | xargs -I {} cp {} $(TEMP_DIR)

	# Compile migrate
	docker run --interactive -v $(shell pwd)/../../../:/go/src/k8s.io/kubernetes -v $(TEMP_DIR):/build -e GOARCH=$(ARCH) golang:$(GOLANG_VERSION) \
		/bin/bash -c "CGO_ENABLED=0 go build -o /build/migrate k8s.io/kubernetes/cluster/images/etcd/migrate"


ifeq ($(ARCH),amd64)

	# Do not compile if we should make an image for amd64, use the official etcd binaries instead
	# For each release create a tmp dir 'etcd_release_tmp_dir' and unpack the release tar there.
	for version in $(BUNDLED_ETCD_VERSIONS); do \
		etcd_release_tmp_dir=$(shell mktemp -d); \
		curl -sSL --retry 5 https://github.com/coreos/etcd/releases/download/v$$version/etcd-v$$version-linux-amd64.tar.gz | tar -xz -C $$etcd_release_tmp_dir --strip-components=1; \
		cp $$etcd_release_tmp_dir/etcd $$etcd_release_tmp_dir/etcdctl $(TEMP_DIR)/; \
		cp $(TEMP_DIR)/etcd $(TEMP_DIR)/etcd-$$version; \
		cp $(TEMP_DIR)/etcdctl $(TEMP_DIR)/etcdctl-$$version; \
	done
else

	# Download etcd in a golang container and cross-compile it statically
	# For each release create a tmp dir 'etcd_release_tmp_dir' and unpack the release tar there.
	for version in $(BUNDLED_ETCD_VERSIONS); do \
		etcd_release_tmp_dir=$(shell mktemp -d); \
		docker run --interactive -v $${etcd_release_tmp_dir}:/etcdbin golang:$(GOLANG_VERSION) /bin/bash -c \
			"git clone https://github.com/coreos/etcd /go/src/github.com/coreos/etcd \
			&& cd /go/src/github.com/coreos/etcd \
			&& git checkout v$${version} \
			&& GOARM=$(GOARM) GOARCH=$(ARCH) ./build \
			&& cp -f bin/$(ARCH)/etcd* bin/etcd* /etcdbin; echo 'done'"; \
		cp $$etcd_release_tmp_dir/etcd $$etcd_release_tmp_dir/etcdctl $(TEMP_DIR)/; \
		cp $(TEMP_DIR)/etcd $(TEMP_DIR)/etcd-$$version; \
		cp $(TEMP_DIR)/etcdctl $(TEMP_DIR)/etcdctl-$$version; \
	done

	# Add this ENV variable in order to workaround an unsupported arch blocker
	# The multiarch feature is in an limited and experimental state right now, and etcd should work fine on arm64
	# On arm (which is 32-bit), it can't handle >1GB data in-memory, but it is very unlikely someone tinkering with their limited arm devices would reach such a high usage
	# ppc64le is still quite untested, but compiles and is probably in the process of being validated by IBM.
	cd $(TEMP_DIR) && echo "ENV ETCD_UNSUPPORTED_ARCH=$(ARCH)" >> Dockerfile
endif

	# Replace BASEIMAGE with the real base image
	cd $(TEMP_DIR) && sed -i.bak 's|BASEIMAGE|$(BASEIMAGE)|g' Dockerfile

	# And build the image
	docker build --pull -t $(REGISTRY)/etcd-$(ARCH):$(IMAGE_TAG) $(TEMP_DIR)

push: build
	docker tag $(REGISTRY)/etcd-$(ARCH):$(IMAGE_TAG) $(PUSH_REGISTRY)/etcd-$(ARCH):$(IMAGE_TAG)
	docker push $(PUSH_REGISTRY)/etcd-$(ARCH):$(IMAGE_TAG)

ifeq ($(ARCH),amd64)
	# Backward compatibility. TODO: deprecate this image tag
	docker tag $(REGISTRY)/etcd-$(ARCH):$(IMAGE_TAG) $(PUSH_REGISTRY)/etcd:$(IMAGE_TAG)
	docker push $(PUSH_REGISTRY)/etcd:$(IMAGE_TAG)
endif

unit-test:
	docker run --interactive -v $(shell pwd)/../../../:/go/src/k8s.io/kubernetes -e GOARCH=$(ARCH) golang:$(GOLANG_VERSION) \
		/bin/bash -c "CGO_ENABLED=0 go test -v k8s.io/kubernetes/cluster/images/etcd/migrate"

# Integration tests require both a golang build environment and all the etcd binaries from a `k8s.gcr.io/etcd` image (`/usr/local/bin/etcd-<version>`, ...).
# Since the `k8s.gcr.io/etcd` image is for runtime only and does not have a build golang environment, we create a new docker image to run integration tests
# with.
build-integration-test-image: build
	cp -r $(TEMP_DIR) $(TEMP_DIR)_integration_test
	cp Dockerfile $(TEMP_DIR)_integration_test/Dockerfile
	cd $(TEMP_DIR)_integration_test && sed -i.bak 's|BASEIMAGE|golang:$(GOLANG_VERSION)|g' Dockerfile
	docker build --pull -t etcd-integration-test $(TEMP_DIR)_integration_test

integration-test:
	docker run --interactive -v $(shell pwd)/../../../:/go/src/k8s.io/kubernetes -e GOARCH=$(ARCH) etcd-integration-test \
		/bin/bash -c "CGO_ENABLED=0 go test -tags=integration -v k8s.io/kubernetes/cluster/images/etcd/migrate -args -v 10 -logtostderr true"

integration-build-test: build-integration-test-image integration-test
test: unit-test integration-build-test
all: build test
.PHONY:	build push unit-test build-integration-test-image integration-test integration-build-test test
