$(eval CHAIN_ID := $(shell rex chain-id))
$(eval OWNER := $(shell rex address))

SOLC_FLAGS := --overwrite --optimize --via-ir

# contracts/ is kept byte-identical to upstream apart from the disabled ZK route in
# AttestationEntrypointBase.sol, so imports resolve through these remappings rather
# than being rewritten to relative paths. solady and openzeppelin come from the nested
# clones inside lib/automata-on-chain-pccs, the same nesting upstream uses, so
# on-chain-pccs is the single dependency this repo has to pin.
PCCS_LIB := lib/automata-on-chain-pccs
# Two openzeppelin remappings are needed: this repo imports `openzeppelin/contracts/...`
# while the on-chain-pccs sources compiled in alongside it import
# `@openzeppelin/contracts/...`. solc resolves by longest matching prefix, so both apply.
SOLC_REMAP := \
	@automata-network/on-chain-pccs/=$(PCCS_LIB)/src/ \
	solady/=$(PCCS_LIB)/lib/solady/src/ \
	openzeppelin/=$(PCCS_LIB)/lib/openzeppelin-contracts/ \
	@openzeppelin/contracts/=$(PCCS_LIB)/lib/openzeppelin-contracts/contracts/
SOLC := solc $(SOLC_FLAGS) $(SOLC_REMAP)

P256_ADDR := 0xc2b78104907F722DABAc4C69f826a522B2754De4

# Must match the TCB evaluation data number the on-chain-pccs versioned DAOs were
# deployed under. A quote verified through the 1-argument verifyAndAttestOnChain
# resolves eval number 0 to TcbEvalDao's `standard` value, and the router then looks
# the versioned DAOs up by that number.
TCB_EVAL_NUMBER ?= 19

# Pinned dependency revision. lib/ is gitignored, so this is the only record of what a
# deployment was built from. This revision pins its own openzeppelin and solady, so
# pinning it here fixes the whole transitive tree.
# Comments stay off the value line: Make keeps whitespace before an inline `#`.
ON_CHAIN_PCCS_REV = 4f1032d924b0262cc977d1ddf4bb7a82d8f49c2c

# Clone if absent, then re-assert and verify the pin on every run, so a stale or
# half-checked-out lib/ is corrected rather than silently reused. `make clean` removes
# every pinned tree, for when a pin cannot be re-asserted in place.
define pin
	@test -d $(1)/.git || git clone -q $(2) $(1)
	@git -C $(1) cat-file -e "$(3)^{commit}" 2>/dev/null || git -C $(1) fetch -q --tags origin
	@git -c advice.detachedHead=false -C $(1) checkout -q --detach $(3)
	@test "$$(git -C $(1) rev-parse HEAD)" = "$(3)" || { echo "$(1): not at $(3)"; exit 1; }
endef

.PHONY: deps
# on-chain-pccs is pinned before its own `make deps` runs, so the nested clones come
# from the pinned revision's Makefile rather than resolving to upstream HEAD.
deps:
	mkdir -p deployment
	$(call pin,$(PCCS_LIB),https://github.com/lambdaclass/automata-on-chain-pccs.git,$(ON_CHAIN_PCCS_REV))
	cd $(PCCS_LIB) && $(MAKE) deps

# ---------------------------------------------------------------- compile

out/PCCSRouter.bin: | deps
	$(SOLC) contracts/PCCSRouter.sol --bin -o out/

out/V4QuoteVerifier.bin: | deps
	$(SOLC) contracts/verifiers/V4QuoteVerifier.sol --bin -o out/

out/AutomataDcapAttestationFee.bin: | deps
	$(SOLC) contracts/AutomataDcapAttestationFee.sol --bin -o out/

.PHONY: build
build: out/PCCSRouter.bin out/V4QuoteVerifier.bin out/AutomataDcapAttestationFee.bin

# ---------------------------------------------------------------- deploy
# Addresses are written to deployment/<ContractName>, the interface ethrex's TDX
# deployer reads. The on-chain-pccs addresses are expected to already be present in
# deployment/ (ethrex copies them across before invoking this).

deploy-router: out/PCCSRouter.bin
	$(eval TCB_EVAL_DAO_ADDR := $(shell cat deployment/AutomataTcbEvalDao))
	$(eval PCS_ADDR := $(shell cat deployment/AutomataPcsDao))
	$(eval PCK_ADDR := $(shell cat deployment/AutomataPckDao))
	$(eval X509_ADDR := $(shell cat deployment/PCKHelper))
	$(eval X509_CRL_ADDR := $(shell cat deployment/X509CRLHelper))
	$(eval FMSPC_HELPER_ADDR := $(shell cat deployment/FmspcTcbHelper))
	rex deploy --print-address $(shell cat out/PCCSRouter.bin) 0 $(PRIVATE_KEY) -- \
		"constructor(address,address,address,address,address,address,address)" \
		$(OWNER) $(TCB_EVAL_DAO_ADDR) $(PCS_ADDR) $(PCK_ADDR) $(X509_ADDR) $(X509_CRL_ADDR) $(FMSPC_HELPER_ADDR) \
		> deployment/PCCSRouter

# The router resolves QE identity and FMSPC TCB through DAOs registered per evaluation
# number, so registering them is required, not optional: without it every quote reverts.
configure-router: deploy-router
	$(eval ROUTER_ADDR := $(shell cat deployment/PCCSRouter))
	$(eval ENCLAVE_ID_ADDR := $(shell cat deployment/AutomataEnclaveIdentityDao))
	$(eval FMSPC_TCB_ADDR := $(shell cat deployment/AutomataFmspcTcbDaoVersioned))
	rex send $(ROUTER_ADDR) "setQeIdDaoVersionedAddr(uint32,address)" $(TCB_EVAL_NUMBER) $(ENCLAVE_ID_ADDR) -k $(PRIVATE_KEY)
	rex send $(ROUTER_ADDR) "setFmspcTcbDaoVersionedAddr(uint32,address)" $(TCB_EVAL_NUMBER) $(FMSPC_TCB_ADDR) -k $(PRIVATE_KEY)

deploy-verifier: out/V4QuoteVerifier.bin configure-router
	$(eval ROUTER_ADDR := $(shell cat deployment/PCCSRouter))
	rex deploy --print-address $(shell cat out/V4QuoteVerifier.bin) 0 $(PRIVATE_KEY) -- \
		"constructor(address,address)" $(P256_ADDR) $(ROUTER_ADDR) \
		> deployment/V4QuoteVerifier

deploy-entrypoint: out/AutomataDcapAttestationFee.bin deploy-verifier
	$(eval V4_ADDR := $(shell cat deployment/V4QuoteVerifier))
	rex deploy --print-address $(shell cat out/AutomataDcapAttestationFee.bin) 0 $(PRIVATE_KEY) -- \
		"constructor(address)" $(OWNER) \
		> deployment/AutomataDcapAttestationFee
	$(eval DCAP_ADDR := $(shell cat deployment/AutomataDcapAttestationFee))
	rex send $(DCAP_ADDR) "setQuoteVerifier(address)" $(V4_ADDR) -k $(PRIVATE_KEY)
	rex send $(shell cat deployment/PCCSRouter) "setAuthorized(address,bool)" $(V4_ADDR) true -k $(PRIVATE_KEY)

deploy: deploy-entrypoint

clean:
	rm -rf out lib deployment/PCCSRouter deployment/V4QuoteVerifier \
		deployment/AutomataDcapAttestationFee

.PHONY: deploy deploy-router configure-router deploy-verifier deploy-entrypoint clean
