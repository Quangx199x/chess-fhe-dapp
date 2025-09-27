# chess-fhe-dapp
This code uses Fully Homomorphic Encryption (FHE) 
# Chess FHE DApp - Sample Repo

Steps to run:

1. Install dependencies:
   ```
   npm install
   ```

2. Compile & deploy contract (configure .env with SEPOLIA_RPC and DEPLOYER_PRIVATE_KEY):
   ```
   npx hardhat compile
   npm run deploy-contract
   ```

3. Replace contract address in `src/app.js` and deploy frontend with Vercel:
   ```
   vercel
   ```
