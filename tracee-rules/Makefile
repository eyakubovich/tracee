.PHONY: all
all: build rules

CMD_DOCKER ?= docker
CMD_GO ?= go
CMD_OPA ?= opa
OUT_DIR ?= dist
OUT_BIN := $(OUT_DIR)/tracee-rules
OUT_RULES := $(OUT_DIR)/rules
GO_SRC := $(shell find . -type f -name '*.go')
GOSIGNATURES_DIR ?= signatures/golang
GOSIGNATURES_SRC := $(shell find $(GOSIGNATURES_DIR) -type f -name '*.go' ! -name '*_test.go' ! -path '$(GOSIGNATURES_DIR)/examples/*')
OUT_GOSIGNATURES := $(OUT_RULES)/builtin.so
REGO_SIGNATURES_DIR ?= signatures/rego
REGO_SIGNATURES_SRC := $(shell find $(REGO_SIGNATURES_DIR) -type f -name '*.rego' ! -name '*_test.rego' ! -path '$(REGO_SIGNATURES_DIR)/examples/*')
DOCKER_BUILDER ?= tracee-rules-builder

$(OUT_DIR):
	mkdir -p $@

.PHONY: build
build: $(OUT_BIN)

$(OUT_BIN): $(filter-out *_test.go,$(GO_SRC)) | $(OUT_DIR)
	go build -o $(OUT_BIN)

.PHONY: rules
rules: $(OUT_RULES)

$(OUT_RULES): $(GOSIGNATURES_DIR) $(REGO_SIGNATURES_SRC) | $(OUT_DIR)
	mkdir -p $(OUT_RULES)
	go build -buildmode=plugin -o $(OUT_GOSIGNATURES) $(GOSIGNATURES_SRC)
	cp $(REGO_SIGNATURES_SRC) $(OUT_RULES)

check_%:
	@command -v $* >/dev/null || (echo "missing required tool $*" ; false)

tools = $(CMD_OPA) $(CMD_GO)
.PHONY: $(tools)
$(tools): % : check_%

.PHONY: test
ifndef DOCKER
test: $(GO_SRC) $(tools)
	go test -v ./...
	opa test . --verbose --ignore="examples" --ignore="dist"
else
test: $(DOCKER_BUILDER)
	$(call docker_builder_make,$@)
endif

.PHONY: clean mostlyclean
clean mostlyclean:
	-$(CMD_DOCKER) rmi $(file < $(docker_builder_file))
	-rm -rf $(OUT_DIR)

# docker_builder_make runs a make command in the tracee-builder container
define docker_builder_make
	$(CMD_DOCKER) run --rm \
	-v $(abspath .):/tracee/tracee-rules \
	-w /tracee/tracee-rules \
	--entrypoint make $(DOCKER_BUILDER) $(1)
endef

docker_builder_file := $(OUT_DIR)/$(DOCKER_BUILDER)
.PHONY: $(DOCKER_BUILDER)
$(DOCKER_BUILDER) $(docker_builder_file) &: Dockerfile.builder | $(OUT_DIR) check_$(CMD_DOCKER)
	$(CMD_DOCKER) build -t $(DOCKER_BUILDER) --iidfile $(docker_builder_file) - < $<