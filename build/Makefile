# Copyright 2017 Google Inc. All Rights Reserved.
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

#
# Makefile for building, testing and developing Agones
#

#  __     __         _       _     _
#  \ \   / /_ _ _ __(_) __ _| |__ | | ___ ___
#   \ \ / / _` | '__| |/ _` | '_ \| |/ _ \ __|
#    \ V / (_| | |  | | (_| | |_) | |  __\__ \
#     \_/ \__,_|_|  |_|\__,_|_.__/|_|\___|___/
#

# base version target. This is usually the next release.
base_version = 0.2.0

# agones image release registry
release_registry = gcr.io/agones-images

#
# All of the following can be overwritten with environemt variables
# or passed through directly when invoking the relevent Make targets
#

# Version defaults to the short hash of the latest commit
VERSION ?= $(base_version)-$(shell git rev-parse --short HEAD)
# The registry that is being used to store docker images
REGISTRY ?= $(release_registry)
# Where the kubectl configuration files are being stored
KUBEPATH ?= ~/.kube
# The (gcloud) test cluster that is being worked against
GCP_CLUSTER_NAME ?= test-cluster
GCP_CLUSTER_ZONE ?= us-west1-c
# the profile to use when developing on minikube
MINIKUBE_PROFILE ?= agones

# Directory that this Makefile is in.
mkfile_path := $(abspath $(lastword $(MAKEFILE_LIST)))
build_path := $(dir $(mkfile_path))
agones_path := $(realpath $(build_path)/..)

agones_package = agones.dev/agones
mount_path = /go/src/$(agones_package)
common_mounts = -v $(build_path)/.config/gcloud:/root/.config/gcloud \
				-v $(KUBEPATH):/root/.kube \
				-v $(agones_path):$(mount_path)

build_tag = agones-build:$(build_version)
build_remote_tag = $(REGISTRY)/$(build_tag)
controller_tag = $(REGISTRY)/agones-controller:$(VERSION)
sidecar_tag = $(REGISTRY)/agones-sdk:$(VERSION)

go_version_flags = -ldflags "-X agones.dev/agones/pkg.Version=$(VERSION)"

#    ___  ____    ___            _           _
#   / _ \/ ___|  |_ _|_ __   ___| |_   _  __| | ___
#  | | | \___ \   | || '_ \ / __| | | | |/ _` |/ _ \
#  | |_| |___) |  | || | | | (__| | |_| | (_| |  __/
#   \___/|____/  |___|_| |_|\___|_|\__,_|\__,_|\___|
#

uname := $(shell uname -s)
ifneq ($(findstring Microsoft,$(shell uname -r)),)
    osinclude := windows.mk
else ifeq ($(uname),Linux)
	osinclude := linux.mk
else ifeq ($(uname),Darwin)
	osinclude := macos.mk
endif

include ./includes/$(osinclude)

# personal includes, excluded from the git repository
-include ./local-includes/*.mk

#   _____                    _
#  |_   _|_ _ _ __ __ _  ___| |_ ___
#    | |/ _` | '__/ _` |/ _ \ __/ __|
#    | | (_| | | | (_| |  __/ |_\__ \
#    |_|\__,_|_|  \__, |\___|\__|___/
#                 |___/

# build all
build: build-images build-sdks

# build the docker images
build-images: build-controller-image build-agones-sdk-image

#build all the sdks
build-sdks: build-sdk-cpp

# Run all tests
test: ensure-build-image lint test-go test-install-yaml

# Run go tests
test-go:
	docker run --rm $(common_mounts) $(build_tag) go test -race $(agones_package)/...

# Run test on install yaml - make sure there is no change
# mostly this is for CI
test-install-yaml:
	-mkdir -p /tmp/agones-install
	cp $(agones_path)/install/yaml/install.yaml /tmp/agones-install/install.yaml
	sort /tmp/agones-install/install.yaml > /tmp/agones-install/install.yaml.sorted
	$(MAKE) gen-install
	sort $(agones_path)/install/yaml/install.yaml > /tmp/agones-install/install.current.yaml.sorted
	diff /tmp/agones-install/install.yaml.sorted /tmp/agones-install/install.current.yaml.sorted

# Push all the images up to $(REGISTRY)
push: push-controller-image push-agones-sdk-image

# Installs the current development version of Agones into the Kubernetes cluster
install: ALWAYS_PULL_SIDECAR := true
install: IMAGE_PULL_POLICY := "Always"
install: ensure-build-image
	docker run --rm $(common_mounts) $(DOCKER_RUN_ARGS) $(build_tag) \
		helm upgrade --install --namespace=default \
		--set agones.image.tag=$(VERSION),agones.image.registry=$(REGISTRY),agones.image.controller.pullPolicy=$(IMAGE_PULL_POLICY),agones.image.sdk.alwaysPull=$(ALWAYS_PULL_SIDECAR) \
		agones $(mount_path)/install/helm/agones/

# Build a static binary for the gameserver controller
build-controller-binary: ensure-build-image
	docker run --rm -e "CGO_ENABLED=0" $(common_mounts) $(build_tag) go build \
		-o $(mount_path)/cmd/controller/bin/controller -a $(go_version_flags) -installsuffix cgo $(agones_package)/cmd/controller

# Lint the go source code.
# use LINT_TIMEOUT to manipulate the linter timeout
lint: LINT_TIMEOUT ?= 5m
lint: ensure-build-image
	docker run --rm $(common_mounts) -w $(mount_path) $(DOCKER_RUN_ARGS) $(build_tag) bash -c \
		"/root/gen-lint-exclude.sh && gometalinter --config .exclude.gometalinter.json --deadline=$(LINT_TIMEOUT) -t --skip vendor ./..."

# Build the image for the gameserver controller
build-controller-image: ensure-build-image build-controller-binary
	docker build $(agones_path)/cmd/controller/ --tag=$(controller_tag) $(DOCKER_BUILD_ARGS)

# push the gameservers controller image
push-controller-image: ensure-build-image
	docker push $(controller_tag)

# build the static binary for the gamesever sidecar
build-agones-sdk-binary: ensure-build-image
	docker run --rm -e "CGO_ENABLED=0" $(common_mounts) $(build_tag) go build \
		-o $(mount_path)/cmd/sdk-server/bin/sdk-server.linux.amd64 -a $(go_version_flags) -installsuffix cgo $(agones_package)/cmd/sdk-server
	docker run --rm -e "GOOS=darwin" -e "GOARCH=amd64" $(common_mounts) $(build_tag) go build \
		-o $(mount_path)/cmd/sdk-server/bin/sdk-server.darwin.amd64 $(go_version_flags) $(agones_package)/cmd/sdk-server
	docker run --rm -e "GOOS=windows" -e "GOARCH=amd64" $(common_mounts) $(build_tag) go build \
		-o $(mount_path)/cmd/sdk-server/bin/sdk-server.windows.amd64.exe $(go_version_flags) $(agones_package)/cmd/sdk-server
	docker run --rm $(common_mounts) -w $(mount_path)/cmd/sdk-server/bin/ $(build_tag) zip \
		agonessdk-server-$(VERSION).zip sdk-server.darwin.amd64 sdk-server.linux.amd64 sdk-server.windows.amd64.exe

# Build the image for the gameserver sidecar
build-agones-sdk-image: ensure-build-image build-agones-sdk-binary
	docker build $(agones_path)/cmd/sdk-server/ --tag=$(sidecar_tag) $(DOCKER_BUILD_ARGS)

# Build the cpp sdk linux archive
build-sdk-cpp: ensure-build-image
	docker run --rm $(common_mounts) -w $(mount_path)/sdks/cpp $(build_tag) make build install archive VERSION=$(VERSION)

# push the gameservers sidecar image
push-agones-sdk-image: ensure-build-image
	docker push $(sidecar_tag)

# Generate the static install script
gen-install: ensure-build-image
	docker run --rm $(common_mounts) $(DOCKER_RUN_ARGS) $(build_tag) bash -c \
		'helm template --name=agones-manual $(mount_path)/install/helm/agones \
		--set agones.controller.generateTLS=false \
		> $(mount_path)/install/yaml/install.yaml'

# Generate the SDK gRPC server and client code
gen-gameservers-sdk-grpc: ensure-build-image
	docker run --rm $(common_mounts) -w $(mount_path) $(build_tag) /root/gen-grpc-go.sh
	docker run --rm $(common_mounts) -w $(mount_path) $(build_tag) /root/gen-grpc-cpp.sh

# Generate the client for our CustomResourceDefinition
gen-crd-client: ensure-build-image
	docker run --rm $(common_mounts) -w $(mount_path) $(build_tag) /root/gen-crd-client.sh
	docker run --rm $(common_mounts) -w $(mount_path)/pkg $(build_tag) goimports -w .

# Run a bash shell with the developer tools in it. (Creates the image if it doesn't exist)
# Can use DOCKER_RUN_ARGS for extra arguments.
shell: ensure-build-image
	docker run -it --rm \
		$(common_mounts) \
		-w $(mount_path) \
		$(DOCKER_RUN_ARGS) \
		$(build_tag) bash -l

# run a container with godoc
godoc:
	if [ ! -f $(build_path)/.index ]; then \
		touch $(build_path)/.index && \
		docker run -p 8888:8888 --rm $(common_mounts) -v $(build_path)/.index:/root/.index \
				$(build_tag) godoc -http=":8888" -index=true -write_index=true -index_files=/root/.index;\
	fi
	docker run -p 8888:8888 --rm $(common_mounts) -v $(build_path)/.index:/root/.index \
		$(build_tag) godoc -http=":8888" -index=true -index_files=/root/.index

# Creates the build docker image
build-build-image:
	docker build --tag=$(build_tag) $(build_path)/build-image $(DOCKER_BUILD_ARGS)

# Deletes the local build docker image
clean-build-image:
	docker rmi $(build_tag)

ensure-build-config:
	-mkdir -p $(KUBEPATH)
	-mkdir -p $(build_path)/.config/gcloud

# create the build image if it doesn't exist
ensure-build-image: ensure-build-config
	@if [ -z $$(docker images -q $(build_tag)) ]; then\
		echo "Could not find $(build_tag) image. Building...";\
		$(MAKE) build-build-image;\
	fi

# attempt to pull the image, if it exists and rename it to the local tag
# exit's clean if it doesn't exist, so can be used on CI
pull-build-image:
	-docker pull $(build_remote_tag) && docker tag $(build_remote_tag) $(build_tag)

# push the local build image up to your repository
push-build-image:
	docker tag $(build_tag) $(build_remote_tag)
	docker push $(build_remote_tag)

# generate a changelog using github-changelog-generator
gen-changelog:
	@read -p 'Github Token: ' TOKEN && \
	docker run -it --rm -v "$(agones_path)":/usr/local/src/your-app ferrarimarco/github-changelog-generator \
		GoogleCloudPlatform/agones --bug-labels=kind/bug --enhancement-labels=kind/feature \
		--future-release "v$(base_version).rc" \
		--token=$$TOKEN

# Creates a release. Version defaults to the base_version
# - Checks out a release branch
# - Build binaries and images
# - Creates sdk and binary archives, and moves the into the /release folder for upload
# - Creates a zip of the install.yaml, LICENCE and README.md for installation
do-release: RELEASE_VERSION ?= $(base_version)
do-release:
	@echo "Starting release for version: $(RELEASE_VERSION)"
	git checkout -b release-$(RELEASE_VERSION)
	$(MAKE) test
	-rm -rf $(agones_path)/release
	mkdir $(agones_path)/release
	docker run --rm $(common_mounts) -w $(mount_path)/sdks/cpp $(build_tag) make clean
	$(MAKE) build VERSION=$(RELEASE_VERSION) REGISTRY=$(release_registry)
	cp $(agones_path)/cmd/sdk-server/bin/agonessdk-server-$(RELEASE_VERSION).zip $(agones_path)/release
	cp $(agones_path)/sdks/cpp/bin/agonessdk-$(RELEASE_VERSION)-runtime-linux-arch_64.tar.gz $(agones_path)/release
	cp $(agones_path)/sdks/cpp/bin/agonessdk-$(RELEASE_VERSION)-dev-linux-arch_64.tar.gz $(agones_path)/release
	cp $(agones_path)/sdks/cpp/bin/agonessdk-$(RELEASE_VERSION)-src.zip $(agones_path)/release
	cd $(agones_path) &&  zip -r ./release/agones-install-$(RELEASE_VERSION).zip ./README.md ./install ./LICENSE
	$(MAKE) gcloud-auth-docker push VERSION=$(RELEASE_VERSION)
	git push -u upstream release-$(RELEASE_VERSION)
	@echo "Now go make the $(RELEASE_VERSION) release on Github!"

#    ____                   _         ____ _                 _
#   / ___| ___   ___   __ _| | ___   / ___| | ___  _   _  __| |
#  | |  _ / _ \ / _ \ / _` | |/ _ \ | |   | |/ _ \| | | |/ _` |
#  | |_| | (_) | (_) | (_| | |  __/ | |___| | (_) | |_| | (_| |
#   \____|\___/ \___/ \__, |_|\___|  \____|_|\___/ \__,_|\__,_|
#                     |___/

# Initialise the gcloud login and project configuration, if you are working with GCP
gcloud-init: ensure-build-config
	docker run --rm -it $(common_mounts) $(build_tag) gcloud init

# Creates and authenticates a small, 3 node GKE cluster to work against
gcloud-test-cluster: GCP_CLUSTER_LEGACYABAC ?= false
gcloud-test-cluster: GCP_CLUSTER_NODEPOOL_INITIALNODECOUNT ?= 4
gcloud-test-cluster: GCP_CLUSTER_NODEPOOL_MACHINETYPE ?= n1-standard-4
gcloud-test-cluster: ensure-build-image
	docker run --rm -it $(common_mounts) $(DOCKER_RUN_ARGS) $(build_tag) gcloud \
		deployment-manager deployments create $(GCP_CLUSTER_NAME)  \
		--properties cluster.zone:$(GCP_CLUSTER_ZONE),cluster.name:$(GCP_CLUSTER_NAME),cluster.nodePool.initialNodeCount:$(GCP_CLUSTER_NODEPOOL_INITIALNODECOUNT),cluster.nodePool.machineType:$(GCP_CLUSTER_NODEPOOL_MACHINETYPE),cluster.legacyAbac:$(GCP_CLUSTER_LEGACYABAC)\
		--template=$(mount_path)/build/gke-test-cluster/cluster.yml.jinja
	$(MAKE) gcloud-auth-cluster
	docker run --rm -it $(common_mounts) $(DOCKER_RUN_ARGS) $(build_tag) kubectl apply -f $(mount_path)/build/helm.yaml
	docker run --rm $(common_mounts) $(DOCKER_RUN_ARGS) $(build_tag) helm init --service-account helm

clean-gcloud-test-cluster: ensure-build-image
	docker run --rm -it $(common_mounts) $(DOCKER_RUN_ARGS) $(build_tag) gcloud \
		deployment-manager deployments delete $(GCP_CLUSTER_NAME)
	
# Pulls down authentication information for kubectl against a cluster, name can be specified through GCP_CLUSTER_NAME
# (defaults to 'test-cluster')
gcloud-auth-cluster: ensure-build-image
	docker run --rm $(common_mounts) $(build_tag) gcloud config set container/cluster $(GCP_CLUSTER_NAME)
	docker run --rm $(common_mounts) $(build_tag) gcloud config set compute/zone $(GCP_CLUSTER_ZONE)
	docker run --rm $(common_mounts) $(build_tag) gcloud container clusters get-credentials $(GCP_CLUSTER_NAME)
	-docker run --rm $(common_mounts) $(build_tag) bash -c 'kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin --user $$(gcloud config get-value account)'

# authenticate our docker configuration so that you can do a docker push directly
# to the gcr.io repository
gcloud-auth-docker: ensure-build-image
	docker run --rm $(common_mounts) $(build_tag) gcloud auth print-access-token | docker login -u oauth2accesstoken --password-stdin https://gcr.io

# Clean the gcloud configuration
clean-gcloud-config:
	-sudo rm -r $(build_path)/.config

#   __  __ _       _ _          _
#  |  \/  (_)_ __ (_) | ___   _| |__   ___
#  | |\/| | | '_ \| | |/ / | | | '_ \ / _ \
#  | |  | | | | | | |   <| |_| | |_) |  __/
#  |_|  |_|_|_| |_|_|_|\_\\__,_|_.__/ \___|
#

# Switches to an "agones" profile, and starts a kubernetes cluster
# of the right version.
#
# Use MINIKUBE_DRIVER variable to change the VM driver
# (defaults virtualbox for Linux and macOS, hyperv for windows) if you so desire.
minikube-test-cluster: ensure-build-image minikube-agones-profile
	$(MINIKUBE) start --kubernetes-version v1.9.4 --vm-driver $(MINIKUBE_DRIVER) \
		--extra-config=apiserver.Admission.PluginNames=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota \
		--extra-config=apiserver.Authorization.Mode=RBAC
	# wait until the master is up
	until docker run --rm $(common_mounts) --network=host -v $(minikube_cert_mount) $(DOCKER_RUN_ARGS) $(build_tag) kubectl cluster-info; \
		do \
			echo "Waiting for cluster to start..."; \
			sleep 1; \
		done
	# this is needed for kubernetes component to work correctly while RBAC is enabled
	-docker run --rm $(common_mounts) --network=host -v $(minikube_cert_mount) $(DOCKER_RUN_ARGS) $(build_tag) \
		kubectl create clusterrolebinding add-on-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:default
	docker run --rm $(common_mounts) --network=host -v $(minikube_cert_mount) $(DOCKER_RUN_ARGS) $(build_tag) helm init
	$(MAKE) minikube-post-start

# switch to the agones cluster
minikube-agones-profile:
	$(MINIKUBE) profile $(MINIKUBE_PROFILE)

# Connecting to minikube requires so enhanced permissions, so use this target
# instead of `make shell` to start an interactive shell for development on minikube.
minikube-shell: ensure-build-image minikube-agones-profile
	$(MAKE) shell DOCKER_RUN_ARGS="--network=host -v $(minikube_cert_mount) $(DOCKER_RUN_ARGS)"

# Push the local Agones Docker images that have already been built
# via `make build` or `make build-images` into the "agones" minikube instance.
minikube-push: minikube-agones-profile
	$(MAKE) minikube-transfer-image TAG=$(sidecar_tag)
	$(MAKE) minikube-transfer-image TAG=$(controller_tag)

# Installs the current development version of Agones into the Kubernetes cluster.
# Use this instead of `make install`, as it disables PullAlways on the install.yaml
minikube-install: minikube-agones-profile
	$(MAKE) install DOCKER_RUN_ARGS="--network=host -v $(minikube_cert_mount)" ALWAYS_PULL_SIDECAR=false IMAGE_PULL_POLICY=IfNotPresent

# Convenience target for transferring images into minikube.
# Use TAG to specify the image to transfer into minikube
minikube-transfer-image:
	docker save $(TAG) | ($(MINIKUBE_DOCKER_ENV) && docker load)
