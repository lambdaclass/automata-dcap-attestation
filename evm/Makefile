$(eval CHAIN_ID := $(shell rex chain-id))
$(eval OWNER := $(shell rex address))

SOLC_FLAGS := --overwrite --optimize --via-ir

P256_ADDR := 0xc2b78104907F722DABAc4C69f826a522B2754De4

# Pinned dependency revisions.
#
# lib/ is gitignored, so without these the deployed bytecode depends on whatever
# the upstream default branches happen to be on the day `make deploy` runs, and
# there is no record afterwards of which version of the attestation logic a given
# deployment is running.
#
# Keep each comment on its own line: Make preserves whitespace preceding an
# inline `#`, which would be appended to the revision string.
#
# v5.7.0. The contracts here reach openzeppelin only transitively through
# lib/on-chain-pccs (which pins it itself), so this pin is for build determinism
# rather than bytecode; it is byte-identical to the master tip it replaces.
OPENZEPPELIN_REV = cab19933c33c2ad1d4c7a84864a3601dddfd16f3
# Untagged main tip as of 2026-08-03, pinned exactly because it is what this repo
# has been building against. The v0.1.26 release does NOT produce identical
# bytecode, so moving to a tagged release is a deliberate change needing its own
# review rather than a drive-by bump.
SOLADY_REV = c251232428b668a073293eb04c6c288b19ad5728
# lambdaclass/automata-on-chain-pccs. This revision pins its own openzeppelin and
# solady clones, so pinning it here also fixes the nested dependencies that reach
# our bytecode.
ON_CHAIN_PCCS_REV = 41a4422ffad3e8b0b5cf1c210ea9aa2c9f51adb4

lib/openzeppelin:
	git clone https://github.com/OpenZeppelin/openzeppelin-contracts lib/openzeppelin
	git -C lib/openzeppelin checkout --detach $(OPENZEPPELIN_REV)

lib/solady:
	git clone https://github.com/vectorized/solady lib/solady
	git -C lib/solady checkout --detach $(SOLADY_REV)

lib/on-chain-pccs:
	git clone https://github.com/lambdaclass/automata-on-chain-pccs.git lib/on-chain-pccs
	# Must precede `make deps` so the nested clones honour the pinned revision's
	# own pins rather than resolving to upstream HEAD.
	git -C lib/on-chain-pccs checkout --detach $(ON_CHAIN_PCCS_REV)
	cd lib/on-chain-pccs; make deps

deps: lib/openzeppelin lib/solady lib/on-chain-pccs
	mkdir -p deployment

out/AutomataDcapAttestationFee.bin: deps
	solc contracts/AutomataDcapAttestationFee.sol --bin -o out/ $(SOLC_FLAGS)

out/PCCSRouter.bin: deps
	solc contracts/PCCSRouter.sol --bin -o out/ $(SOLC_FLAGS)

out/V4QuoteVerifier.bin: deps
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
