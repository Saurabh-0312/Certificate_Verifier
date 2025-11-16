# Alumni Verification - Quick Commands
# Run these commands from the BlockChain directory

# ============================================
# COMPILATION & TESTING
# ============================================

# Build contracts
build:
	forge build

# Clean and rebuild
rebuild:
	forge clean && forge build

# Run all tests
test:
	forge test

# Run tests with gas reporting
test-gas:
	forge test --gas-report

# Run tests with detailed output
test-verbose:
	forge test -vvvv

# Run specific test
test-one TEST_NAME:
	forge test --match-test $(TEST_NAME) -vvvv

# Check test coverage
coverage:
	forge coverage

# Format code
format:
	forge fmt

# ============================================
# DEPLOYMENT
# ============================================

# Deploy to Mumbai Testnet
deploy-mumbai:
	forge script script/DeployAlumniVerification.s.sol:DeployAlumniVerification \
		--rpc-url $(POLYGON_MUMBAI_RPC_URL) \
		--broadcast \
		-vvvv

# Deploy to Polygon Mainnet (BE CAREFUL!)
deploy-polygon:
	forge script script/DeployAlumniVerification.s.sol:DeployAlumniVerification \
		--rpc-url $(POLYGON_MAINNET_RPC_URL) \
		--broadcast \
		--verify \
		-vvvv

# Test deployment with sample data
test-deployment CONTRACT_ADDRESS:
	forge script script/DeployAlumniVerification.s.sol:TestAlumniVerification \
		--rpc-url $(POLYGON_MUMBAI_RPC_URL) \
		--sig "testAddRecord(address)" \
		$(CONTRACT_ADDRESS) \
		--broadcast

# ============================================
# CONTRACT INTERACTION (using cast)
# ============================================

# Get total records
get-total CONTRACT_ADDRESS:
	cast call $(CONTRACT_ADDRESS) "getTotalRecords()" --rpc-url $(POLYGON_MUMBAI_RPC_URL)

# Check if address is authorized issuer
check-issuer CONTRACT_ADDRESS ISSUER_ADDRESS:
	cast call $(CONTRACT_ADDRESS) "isAuthorizedIssuer(address)(bool)" $(ISSUER_ADDRESS) --rpc-url $(POLYGON_MUMBAI_RPC_URL)

# Get record details
get-record CONTRACT_ADDRESS CERT_ID:
	cast call $(CONTRACT_ADDRESS) "getRecord(string)" $(CERT_ID) --rpc-url $(POLYGON_MUMBAI_RPC_URL)

# Add new issuer (requires owner private key)
add-issuer CONTRACT_ADDRESS NEW_ISSUER INSTITUTION_NAME:
	cast send $(CONTRACT_ADDRESS) \
		"authorizeIssuer(address,string)" \
		$(NEW_ISSUER) \
		$(INSTITUTION_NAME) \
		--rpc-url $(POLYGON_MUMBAI_RPC_URL) \
		--private-key $(PRIVATE_KEY)

# ============================================
# UTILITIES
# ============================================

# Generate hash for alumni data
generate-hash NAME ROLL DEGREE BRANCH YEAR CERT_ID:
	cast keccak $(shell echo -n "$(NAME)$(ROLL)$(DEGREE)$(BRANCH)$(YEAR)$(CERT_ID)")

# Get wallet address from private key
get-address:
	cast wallet address --private-key $(PRIVATE_KEY)

# Check wallet balance
check-balance ADDRESS:
	cast balance $(ADDRESS) --rpc-url $(POLYGON_MUMBAI_RPC_URL)

# Get current gas price
gas-price:
	cast gas-price --rpc-url $(POLYGON_MUMBAI_RPC_URL)

# Get block number
block-number:
	cast block-number --rpc-url $(POLYGON_MUMBAI_RPC_URL)

# ============================================
# VERIFICATION
# ============================================

# Verify contract on Polygonscan
verify-contract CONTRACT_ADDRESS:
	forge verify-contract \
		--chain-id 80001 \
		--watch \
		$(CONTRACT_ADDRESS) \
		src/AlumniVerification.sol:AlumniVerification \
		--etherscan-api-key $(POLYGONSCAN_API_KEY)

# ============================================
# HELP
# ============================================

help:
	@echo "Available commands:"
	@echo "  make build             - Compile contracts"
	@echo "  make test              - Run all tests"
	@echo "  make test-verbose      - Run tests with detailed output"
	@echo "  make coverage          - Check test coverage"
	@echo "  make deploy-mumbai     - Deploy to Mumbai testnet"
	@echo "  make get-total CONTRACT_ADDRESS - Get total records"
	@echo "  make check-balance ADDRESS - Check wallet balance"
	@echo ""
	@echo "Environment variables needed:"
	@echo "  PRIVATE_KEY"
	@echo "  POLYGON_MUMBAI_RPC_URL"
	@echo "  POLYGONSCAN_API_KEY (optional)"

# Set default goal
.DEFAULT_GOAL := help
