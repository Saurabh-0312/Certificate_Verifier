# 🚀 Backend Integration Guide for Alumni Verification

This guide will help you integrate the Alumni Verification smart contract with your Node.js/Express backend.

## 📦 Required Dependencies

```bash
npm install ethers dotenv express cors
```

## 🔧 Environment Setup

Create a `.env` file in your backend directory:

```env
# Blockchain Configuration
CONTRACT_ADDRESS=0xYourDeployedContractAddress
POLYGON_RPC_URL=https://rpc-mumbai.maticvigil.com
ADMIN_PRIVATE_KEY=your_admin_wallet_private_key
NETWORK=mumbai

# Server Configuration
PORT=5000
FRONTEND_URL=http://localhost:5173
```

## 📝 Contract ABI

After compiling with Foundry, the ABI is located at:

```
BlockChain/out/AlumniVerification.sol/AlumniVerification.json
```

Extract the `abi` array from this file.

## 💻 Backend Implementation

### 1. Setup Blockchain Connection (`blockchain/config.js`)

```javascript
const { ethers } = require("ethers");
require("dotenv").config();

// Contract ABI (copy from out/AlumniVerification.sol/AlumniVerification.json)
const CONTRACT_ABI = [
  "function addAlumniRecord(string memory _certId, bytes32 _dataHash) external returns (bool success, uint256 timestamp, uint256 blockNumber)",
  "function verifyRecord(string memory _certId, bytes32 _dataHash) external returns (bool isValid, address issuer, string memory issuerName, uint256 timestamp, uint256 blockNumber)",
  "function getRecord(string memory _certId) external view returns (bytes32 dataHash, address issuer, string memory issuerName, uint256 timestamp, uint256 blockNumber, bool exists)",
  "function generateDataHash(string memory _name, string memory _rollNumber, string memory _degree, string memory _branch, string memory _graduationYear, string memory _certId) public pure returns (bytes32)",
  "function recordExists(string memory _certId) external view returns (bool exists)",
  "function getTotalRecords() external view returns (uint256)",
  "event AlumniRecordAdded(string indexed certId, bytes32 dataHash, address indexed issuer, string issuerName, uint256 timestamp, uint256 blockNumber)",
];

// Setup provider and signer
const provider = new ethers.providers.JsonRpcProvider(
  process.env.POLYGON_RPC_URL
);
const signer = new ethers.Wallet(process.env.ADMIN_PRIVATE_KEY, provider);

// Contract instance
const contract = new ethers.Contract(
  process.env.CONTRACT_ADDRESS,
  CONTRACT_ABI,
  signer
);

// Read-only contract (no gas costs)
const contractReadOnly = new ethers.Contract(
  process.env.CONTRACT_ADDRESS,
  CONTRACT_ABI,
  provider
);

module.exports = {
  contract,
  contractReadOnly,
  provider,
  signer,
};
```

### 2. Hash Generation Utility (`utils/hashUtils.js`)

```javascript
const { ethers } = require("ethers");

/**
 * Generate hash from alumni data (same as smart contract)
 * @param {Object} data - Alumni data
 * @returns {string} - Hash of the data
 */
function generateDataHash(data) {
  const { name, rollNumber, degree, branch, graduationYear, certId } = data;

  // Must match the smart contract's generateDataHash function
  return ethers.utils.solidityKeccak256(
    ["string", "string", "string", "string", "string", "string"],
    [name, rollNumber, degree, branch, graduationYear, certId]
  );
}

/**
 * Validate alumni data
 * @param {Object} data - Alumni data
 * @returns {Object} - Validation result
 */
function validateAlumniData(data) {
  const errors = {};

  if (!data.name || data.name.trim().length === 0) {
    errors.name = "Name is required";
  }

  if (!data.rollNumber || data.rollNumber.trim().length === 0) {
    errors.rollNumber = "Roll number is required";
  }

  if (!data.degree || data.degree.trim().length === 0) {
    errors.degree = "Degree is required";
  }

  if (!data.branch || data.branch.trim().length === 0) {
    errors.branch = "Branch is required";
  }

  if (!data.graduationYear) {
    errors.graduationYear = "Graduation year is required";
  } else {
    const year = parseInt(data.graduationYear);
    const currentYear = new Date().getFullYear();
    if (year < 1950 || year > currentYear + 5) {
      errors.graduationYear = "Invalid graduation year";
    }
  }

  if (!data.certId || data.certId.trim().length === 0) {
    errors.certId = "Certificate ID is required";
  }

  return {
    isValid: Object.keys(errors).length === 0,
    errors,
  };
}

module.exports = {
  generateDataHash,
  validateAlumniData,
};
```

### 3. Admin Routes (`routes/admin.js`)

```javascript
const express = require("express");
const router = express.Router();
const { contract, contractReadOnly } = require("../blockchain/config");
const { generateDataHash, validateAlumniData } = require("../utils/hashUtils");

/**
 * POST /api/admin/add
 * Add new alumni record to blockchain
 */
router.post("/add", async (req, res) => {
  try {
    const alumniData = req.body;

    // Validate data
    const validation = validateAlumniData(alumniData);
    if (!validation.isValid) {
      return res.status(400).json({
        success: false,
        message: "Validation failed",
        errors: validation.errors,
      });
    }

    // Check if certificate ID already exists
    const exists = await contractReadOnly.recordExists(alumniData.certId);
    if (exists) {
      return res.status(409).json({
        success: false,
        message: "Certificate ID already exists",
      });
    }

    // Generate hash
    const dataHash = generateDataHash(alumniData);

    console.log("Adding record to blockchain...");
    console.log("Certificate ID:", alumniData.certId);
    console.log("Data Hash:", dataHash);

    // Send transaction to blockchain
    const tx = await contract.addAlumniRecord(alumniData.certId, dataHash);

    console.log("Transaction sent:", tx.hash);
    console.log("Waiting for confirmation...");

    // Wait for transaction confirmation
    const receipt = await tx.wait();

    console.log("Transaction confirmed!");
    console.log("Block number:", receipt.blockNumber);

    // Extract event data
    const event = receipt.events?.find((e) => e.event === "AlumniRecordAdded");

    // Generate QR code URL
    const verificationUrl = `${process.env.FRONTEND_URL}/verify/${alumniData.certId}`;

    res.status(201).json({
      success: true,
      message: "Alumni record added successfully",
      data: {
        certId: alumniData.certId,
        transactionHash: receipt.transactionHash,
        blockNumber: receipt.blockNumber,
        timestamp: event?.args?.timestamp?.toString() || Date.now(),
        dataHash: dataHash,
        verificationUrl: verificationUrl,
        gasUsed: receipt.gasUsed.toString(),
      },
    });
  } catch (error) {
    console.error("Error adding alumni record:", error);

    // Handle specific errors
    if (error.code === "INSUFFICIENT_FUNDS") {
      return res.status(402).json({
        success: false,
        message: "Insufficient funds for gas fees",
      });
    }

    if (error.message.includes("Not an authorized issuer")) {
      return res.status(403).json({
        success: false,
        message: "Not authorized to add records",
      });
    }

    res.status(500).json({
      success: false,
      message: "Failed to add alumni record",
      error: error.message,
    });
  }
});

/**
 * GET /api/admin/stats
 * Get statistics
 */
router.get("/stats", async (req, res) => {
  try {
    const totalRecords = await contractReadOnly.getTotalRecords();

    res.json({
      success: true,
      data: {
        totalRecords: totalRecords.toString(),
        network: process.env.NETWORK,
        contractAddress: process.env.CONTRACT_ADDRESS,
      },
    });
  } catch (error) {
    console.error("Error fetching stats:", error);
    res.status(500).json({
      success: false,
      message: "Failed to fetch statistics",
      error: error.message,
    });
  }
});

module.exports = router;
```

### 4. Verification Routes (`routes/verify.js`)

```javascript
const express = require("express");
const router = express.Router();
const { contract, contractReadOnly } = require("../blockchain/config");
const { generateDataHash } = require("../utils/hashUtils");

/**
 * POST /api/verify/check
 * Verify alumni record
 */
router.post("/check", async (req, res) => {
  try {
    const { certId, alumniData } = req.body;

    if (!certId) {
      return res.status(400).json({
        success: false,
        message: "Certificate ID is required",
      });
    }

    // Check if record exists
    const exists = await contractReadOnly.recordExists(certId);

    if (!exists) {
      return res.status(404).json({
        success: false,
        valid: false,
        message: "Certificate not found",
      });
    }

    // Get record from blockchain
    const record = await contractReadOnly.getRecord(certId);

    let isValid = true;

    // If alumni data is provided, verify hash
    if (alumniData) {
      const computedHash = generateDataHash({
        ...alumniData,
        certId,
      });
      isValid = computedHash === record.dataHash;
    }

    res.json({
      success: true,
      valid: isValid,
      data: {
        certId: certId,
        issuer: record.issuer,
        issuerName: record.issuerName,
        timestamp: record.timestamp.toString(),
        blockNumber: record.blockNumber.toString(),
        isValid: isValid,
      },
    });
  } catch (error) {
    console.error("Error verifying record:", error);
    res.status(500).json({
      success: false,
      message: "Failed to verify record",
      error: error.message,
    });
  }
});

/**
 * GET /api/verify/:certId
 * Get record details
 */
router.get("/:certId", async (req, res) => {
  try {
    const { certId } = req.params;

    const exists = await contractReadOnly.recordExists(certId);

    if (!exists) {
      return res.status(404).json({
        success: false,
        message: "Certificate not found",
      });
    }

    const record = await contractReadOnly.getRecord(certId);

    res.json({
      success: true,
      data: {
        certId: certId,
        issuer: record.issuer,
        issuerName: record.issuerName,
        timestamp: new Date(record.timestamp.toNumber() * 1000).toISOString(),
        blockNumber: record.blockNumber.toString(),
        exists: true,
      },
    });
  } catch (error) {
    console.error("Error fetching record:", error);
    res.status(500).json({
      success: false,
      message: "Failed to fetch record",
      error: error.message,
    });
  }
});

module.exports = router;
```

### 5. Main Server (`server.js`)

```javascript
const express = require("express");
const cors = require("cors");
require("dotenv").config();

const adminRoutes = require("./routes/admin");
const verifyRoutes = require("./routes/verify");

const app = express();

// Middleware
app.use(
  cors({
    origin: process.env.FRONTEND_URL || "http://localhost:5173",
  })
);
app.use(express.json());

// Routes
app.use("/api/admin", adminRoutes);
app.use("/api/verify", verifyRoutes);

// Health check
app.get("/health", (req, res) => {
  res.json({
    status: "OK",
    network: process.env.NETWORK,
    contractAddress: process.env.CONTRACT_ADDRESS,
  });
});

// Error handling
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({
    success: false,
    message: "Internal server error",
  });
});

const PORT = process.env.PORT || 5000;

app.listen(PORT, () => {
  console.log(`🚀 Server running on port ${PORT}`);
  console.log(`📡 Network: ${process.env.NETWORK}`);
  console.log(`📝 Contract: ${process.env.CONTRACT_ADDRESS}`);
});
```

## 🔌 Frontend Integration

Update your `AdminPanel.jsx`:

```javascript
// Replace the mock submission (lines 72-85) with:

const handleSubmit = async (e) => {
  e.preventDefault();

  if (!validateForm()) return;

  setLoading(true);

  try {
    const response = await axios.post("http://localhost:5000/api/admin/add", {
      name: formData.name,
      rollNumber: formData.rollNumber,
      degree: formData.degree,
      branch: formData.branch,
      graduationYear: formData.graduationYear,
      certId: formData.certId,
    });

    if (response.data.success) {
      setResult({
        transactionHash: response.data.data.transactionHash,
        certId: response.data.data.certId,
        timestamp: response.data.data.timestamp,
        blockNumber: response.data.data.blockNumber,
        verificationUrl: response.data.data.verificationUrl,
      });
      setSubmitted(true);
    }
  } catch (error) {
    console.error("Error:", error);
    alert(
      "Failed to add record: " +
        (error.response?.data?.message || error.message)
    );
  } finally {
    setLoading(false);
  }
};
```

## 📋 Testing the Integration

### 1. Start the backend:

```bash
cd backend
npm install
npm start
```

### 2. Test endpoints:

**Add Record:**

```bash
curl -X POST http://localhost:5000/api/admin/add \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Saurabh Singh",
    "rollNumber": "2214094",
    "degree": "B.Tech",
    "branch": "Information Technology",
    "graduationYear": "2026",
    "certId": "CERT-2025-TEST001"
  }'
```

**Verify Record:**

```bash
curl http://localhost:5000/api/verify/CERT-2025-TEST001
```

## 🎯 Next Steps

1. ✅ Deploy smart contract to Polygon Mumbai
2. ✅ Update `.env` with contract address
3. ✅ Implement backend routes
4. ✅ Update frontend to call backend API
5. ✅ Test end-to-end flow
6. 📱 Add QR code scanning
7. 🔐 Implement authentication (Gmail OAuth)
8. 📊 Add admin dashboard with list view
9. 📄 Generate PDF certificates

---

**Happy Coding! 🚀**
