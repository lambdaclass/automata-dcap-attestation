$(eval CHAIN_ID := $(shell rex chain-id))
$(eval OWNER := $(shell rex address))

SOLC_FLAGS := --overwrite --optimize --via-ir

P256_ADDR := 0xc2b78104907F722DABAc4C69f826a522B2754De4

# Pinned dependency revisions. lib/ is gitignored, so these are the only record
# of what a deployment was built from.
# Comments stay off the value lines: Make keeps whitespace before an inline `#`.
# openzeppelin v5.7.0. Nothing under contracts/ imports it directly, so the
# top-level clone is unused here; openzeppelin reaches our bytecode only through
# the nested clone inside lib/on-chain-pccs.
OPENZEPPELIN_REV = cab19933c33c2ad1d4c7a84864a3601dddfd16f3
# solady main tip, 23 commits past v0.1.26. Bumping to v0.1.26 changes the
# emitted bytes, so it needs its own review rather than a drive-by bump.
SOLADY_REV = c251232428b668a073293eb04c6c288b19ad5728
# lambdaclass/automata-on-chain-pccs, which pins its own openzeppelin and solady,
# so this also fixes the nested dependencies that reach our bytecode.
# Merge commit of that repo's PR #2, on main, so a default clone always reaches it.
ON_CHAIN_PCCS_REV = 903fd1bcfd4609f7d7a2c0cb809a3240a927db2b

# Clone if absent, then re-assert and verify the pin on every run, so a stale or
# half-checked-out lib/ is corrected rather than silently reused. After editing a
# revision above, `make clean` to rebuild out/.
define pin
	@test -d $(1)/.git || git clone -q $(2) $(1)
	@git -C $(1) cat-file -e "$(3)^{commit}" 2>/dev/null || git -C $(1) fetch -q --tags origin
	@git -c advice.detachedHead=false -C $(1) checkout -q --detach $(3)
	@test "$$(git -C $(1) rev-parse HEAD)" = "$(3)" || { echo "$(1): not at $(3)"; exit 1; }
endef

.PHONY: deps
# on-chain-pccs is pinned before its own `make deps` runs, so the nested
# openzeppelin/solady clones come from the pinned revision's Makefile.
deps:
	mkdir -p deployment
	$(call pin,lib/openzeppelin,https://github.com/OpenZeppelin/openzeppelin-contracts,$(OPENZEPPELIN_REV))
	$(call pin,lib/solady,https://github.com/vectorized/solady,$(SOLADY_REV))
	$(call pin,lib/on-chain-pccs,https://github.com/lambdaclass/automata-on-chain-pccs.git,$(ON_CHAIN_PCCS_REV))
	cd lib/on-chain-pccs && $(MAKE) deps

out/AutomataDcapAttestationFee.bin: | deps
	solc contracts/AutomataDcapAttestationFee.sol --bin -o out/ $(SOLC_FLAGS)

out/PCCSRouter.bin: | deps
	solc contracts/PCCSRouter.sol --bin -o out/ $(SOLC_FLAGS)

out/V4QuoteVerifier.bin: | deps
	solc contracts/verifiers/V4QuoteVerifier.sol --bin -o out/ $(SOLC_FLAGS)

deploy-router: out/PCCSRouter.bin
	$(eval ENCLAVE_ID_ADDR := $(shell cat deployment/AutomataEnclaveIdentityDao))
	$(eval PCS_ADDR := $(shell cat deployment/AutomataPcsDao))
	$(eval PCK_ADDR := $(shell cat deployment/AutomataPckDao))
	$(eval TCB_ADDR := $(shell cat deployment/AutomataFmspcTcbDao))

	$(eval X509_ADDR := $(shell cat deployment/PCKHelper))
	$(eval X509_CRL_ADDR := $(shell cat deployment/X509CRLHelper))
	$(eval FMSPC_TCB_HELPER_ADDR := $(shell cat deployment/FmspcTcbHelper))

	rex deploy --print-address $(shell cat out/PCCSRouter.bin) 0 $(PRIVATE_KEY) -- \
		"constructor(address,address,address,address,address,address,address,address)" \
		$(OWNER) $(ENCLAVE_ID_ADDR) $(TCB_ADDR) $(PCS_ADDR) $(PCK_ADDR) \
		$(X509_ADDR) $(X509_CRL_ADDR) $(FMSPC_TCB_HELPER_ADDR) \
		> deployment/PCCSRouter

deploy-verifier: out/V4QuoteVerifier.bin deploy-router
	$(eval PCCS_ROUTER_ADDR := $(shell cat deployment/PCCSRouter))
	rex deploy --print-address $(shell cat out/V4QuoteVerifier.bin) 0 $(PRIVATE_KEY) -- \
		"constructor(address,address)" \
		$(P256_ADDR) $(PCCS_ROUTER_ADDR) \
		> deployment/V4QuoteVerifier

deploy-attestation: out/AutomataDcapAttestationFee.bin
	rex deploy --print-address $(shell cat out/AutomataDcapAttestationFee.bin) 0 $(PRIVATE_KEY) -- \
		"constructor(address)" $(OWNER) \
		> deployment/AutomataDcapAttestationFee

deploy-and-configure: deploy-router deploy-verifier deploy-attestation
	$(eval PCCS_ROUTER_ADDR := $(shell cat deployment/PCCSRouter))
	$(eval VERIFIER_ADDR := $(shell cat deployment/V4QuoteVerifier))
	$(eval ATTESTATION_ADDR := $(shell cat deployment/AutomataDcapAttestationFee))
	$(eval STORAGE_ADDR := $(shell cat deployment/AutomataDaoStorage))
	rex send $(ATTESTATION_ADDR) "setQuoteVerifier(address)" $(VERIFIER_ADDR) -k $(PRIVATE_KEY)
	rex send $(PCCS_ROUTER_ADDR) "setAuthorized(address,bool)" $(VERIFIER_ADDR) true -k $(PRIVATE_KEY)
	rex send $(STORAGE_ADDR) "setCallerAuthorization(address,bool)" $(PCCS_ROUTER_ADDR) true -k $(PRIVATE_KEY)


deploy: deploy-and-configure

.PHONY: deploy-*
