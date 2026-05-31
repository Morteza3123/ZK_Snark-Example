# zk-SNARK Example with Circom & SnarkJS

This project demonstrates a simple zk-SNARK workflow using a multiplication circuit written in Circom. The process consists of:

1. Compiling the circuit
2. Generating a witness
3. Running the trusted setup
4. Generating a proof
5. Verifying the proof
6. Exporting a Solidity verifier for on-chain verification

---

## 1. Compile the Circuit

Compile the Circom circuit and generate all required artifacts:

```bash
circom multiplier2.circom --r1cs --wasm --sym --c
```

This command generates:

* `multiplier2.r1cs` → Circuit constraints
* `multiplier2.wasm` → WebAssembly file used for witness generation
* `multiplier2.sym` → Symbol mapping file
* `multiplier2_cpp/` → C++ witness generator

---

## 2. Generate the Witness

Navigate to the generated JavaScript folder:

```bash
cd multiplier2_js
```

Create an `input.json` file containing the private and public inputs for the circuit.

Example:

```json
{
  "a": 3,
  "b": 11
}
```

Generate the witness:

```bash
node generate_witness.js multiplier2.wasm input.json witness.wtns
```

The output file:

```text
witness.wtns
```

contains the witness that will later be used to generate a zk-proof.

---

# 3. Trusted Setup

Groth16 requires a trusted setup ceremony consisting of two phases.

---

## Phase 1: Powers of Tau Ceremony

### Create a New Ceremony

```bash
snarkjs powersoftau new bn128 12 pot12_0000.ptau -v
```

### Contribute Entropy

```bash
snarkjs powersoftau contribute pot12_0000.ptau pot12_0001.ptau \
  --name="First contribution" -v
```

This contribution adds randomness to the ceremony and helps ensure security.

---

## Phase 2: Circuit-Specific Setup

### Prepare Phase 2

```bash
snarkjs powersoftau prepare phase2 \
  pot12_0001.ptau \
  pot12_final.ptau -v
```

### Generate Initial zKey

```bash
snarkjs groth16 setup \
  multiplier2.r1cs \
  pot12_final.ptau \
  multiplier2_0000.zkey
```

### Contribute to Phase 2

```bash
snarkjs zkey contribute \
  multiplier2_0000.zkey \
  multiplier2_0001.zkey \
  --name="1st Contributor Name" -v
```

### Export the Verification Key

```bash
snarkjs zkey export verificationkey \
  multiplier2_0001.zkey \
  verification_key.json
```

At this stage, you have:

```text
multiplier2_0001.zkey
verification_key.json
```

which are required for proof generation and verification.

---

# 4. Generate a zk-Proof

Once the witness and trusted setup are ready, generate a proof:

```bash
snarkjs groth16 prove \
  multiplier2_0001.zkey \
  witness.wtns \
  proof.json \
  public.json
```

Generated files:

* `proof.json` → The zk-proof
* `public.json` → Public inputs used for verification

---

# 5. Verify the Proof

Verify the generated proof locally:

```bash
snarkjs groth16 verify \
  verification_key.json \
  public.json \
  proof.json
```

Expected output:

```text
OK!
```

If the proof is valid, SnarkJS will confirm successful verification.

---

# 6. Verify Proofs On-Chain

You can also verify Groth16 proofs directly on Ethereum-compatible blockchains.

### Export a Solidity Verifier

Generate a Solidity verifier contract:

```bash
snarkjs zkey export solidityverifier \
  multiplier2_0001.zkey \
  verifier.sol
```

This creates:

```text
verifier.sol
```

The generated contract contains the verification logic and can be deployed to an EVM-compatible network.

After deployment, applications can submit:

* The proof (`proof.json`)
* The public inputs (`public.json`)

to verify the proof on-chain.

---

## Complete Workflow

```text
Circuit (.circom)
        │
        ▼
Compile Circuit
        │
        ▼
Generate Witness
        │
        ▼
Trusted Setup
        │
        ▼
Generate Proof
        │
        ▼
Verify Proof
        │
        ▼
Export Solidity Verifier
        │
        ▼
On-Chain Verification
```

This example demonstrates the complete Groth16 proving lifecycle, from circuit compilation to Ethereum smart contract verification.
