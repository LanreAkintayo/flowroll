# Flowroll

> **Omnichain, yield-bearing payroll protocol built natively on Initia.**

---

## Initia Hackathon Submission

- **Project Name**: Flowroll

### Project Overview

Flowroll is an omnichain payroll protocol designed to put idle capital to work. Instead of letting funds sit doing nothing, Flowroll autonomously routes and rebalances employer deposits across risk-adjusted USDC stable pools on Initia, generating yield right up until payday. It is built  for crypto-native companies and DAOs that want their treasury to stay productive without any extra operational headache.

Employees get the same treatment: they can claim salary and bridge to any chain via the Initia bridge, request salary advances, or activate auto-save to earn yield on their own earnings.

### Implementation Details

#### The Custom Implementation

The most original thing Flowroll builds is a **dynamic yield routing engine** designed specifically for the payroll lifecycle. The core idea is that payroll capital is predictably idle, and you always know exactly when it needs to be liquid. 
Flowroll takes full advantage of this predictability using three interlocking systems:

**Pool Scoring Formula:** idle capital is never deployed blindly. Every registered pool is scored in real time across four dimensions before any deployment decision:

```
Score = APY × LiquidityFactor × RiskMultiplier × ILRiskFactor
```

- `APY`: the pool's reported yield in basis points
- `LiquidityFactor`: penalizes shallow pools that cannot absorb the full deployment amount
- `RiskMultiplier`: dynamically tightens based on how close payday is (HIGH / MED / LOW)
- `ILRiskFactor`: discounts volatile pairs (0.7x) relative to stable pairs (1.0x)

**Dynamic Buffer Ladder:** as payday approaches, the protocol automatically increases the minimum idle buffer it keeps out of pools, ensuring liquidity is always available on payday regardless of pool withdrawal conditions. The ladder is percentage-based so it scales correctly for any cycle duration (hours, days, or months):

| % of Cycle Remaining | Buffer % of Principal |
|---|---|
| ≥ 70% | 5% |
| ≥ 50% | 10% |
| ≥ 30% | 15% |
| ≥ 25% | 40% |
| ≥ 10% | 80% |
| < 10% | 105% (capped at principal) |

**Adapter Pattern:** `YieldRouter` never interacts with pools directly. Every pool has a deployed adapter implementing `IPoolAdapter`. Adding a new pool requires only writing an adapter and calling `addPool()`, with zero changes to the core routing logic. Switching from testnet mocks to real InitiaDEX pools on mainnet is just a matter of registering a different adapter address.

The buffer and risk configs are snapshotted onto each cycle at start time, so mid-cycle owner changes never affect a running cycle. An employer who deposited under one set of rules is always governed by those rules for the lifetime of that cycle. The off-chain agent calls `agentRebalance()` periodically, and the contract responds correctly whenever it is called, keeping contracts clean and gas-efficient while the agent handles all scheduling logic.

#### The Native Interwoven Features

Flowroll integrates three native Initia features, each solving a real UX problem:

**Auto-Sign (Session Keys):** without session keys, every payroll action (adding employees, setting salaries, triggering disbursement) would require a wallet signature. Flowroll uses Auto-Sign so employers sign once to authorize a session, and the protocol handles all batch payroll operations silently from that point. Employees benefit equally, as claiming salary, activating Auto-Save, requesting a salary advance, and repaying debt all happen without a wallet prompt on every action. This makes Flowroll feel like a Web2 payroll tool, not a sequence of blockchain transactions.

**`.init` Usernames:** raw wallet addresses are a barrier for non-crypto-native HR workflows. Flowroll lets employers add employees by their `.init` handle instead of a 42-character hex address. The frontend resolves the username to the corresponding wallet address before submitting the transaction, so the contract always receives a valid address. Onboarding an employee into payroll now looks identical to adding them in any traditional HR system.

**Interwoven Bridge:** employees receive their salary credited to `PayVault` on Initia. When they claim, they can bridge their payout to any chain they prefer directly from the Flowroll frontend via the Initia bridge widget. The protocol itself has no dependency on IBC channel IDs or chain-specific USDC denoms, as bridging is handled at the frontend layer, giving employees full control over where their money lands without adding any complexity to the contracts.

> **Note:** Cross-chain bridging is already designed at the frontend layer but is not operational in the local appchain environment due to rollup limitations. It will function as intended on a live Initia testnet or mainnet deployment where IBC channels are active.

### How to Run Locally

The full setup involves four components running together: the appchain, the smart contracts, the rebalance agent, and the frontend. See the [Getting Started](#getting-started) section below for the complete step-by-step guide including environment variable references, contract wiring, and pool seeding instructions.

---

## How It Works

```
┌─────────────────────────────────────────────────────────────────────┐
│                        EMPLOYER SIDE                                │
│                                                                     │
│  1. Employer creates a payroll group                  │
│  2. Adds employees (by .init username or wallet address)            │
│     with salaries and sets a payday                                 │
│  3. Calls setupPayroll() → funds flow to YieldRouter                │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                        YIELD FARMING                                │
│                                                                     │
│  4. Agent runs every N seconds (specified by Flowroll)              │
│  5. Scores all registered pools (APY × Liquidity × Risk × IL)       │
│  6. Deploys idle capital to highest-scoring pool via adapter        │
│  7. Rebalances if a significantly better pool appears               │
│  8. Buffer ladder ensures liquidity is always available for payday  │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                      FLOWROLLCREDIT                                 │
│                                                                     │
│  9.  Employee calls requestSalary(amount) to request advance        │
│  10. Employee calls repayDebt() to repay before payday              │
│      (unpaid balance is deducted from payday payout)                │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                           PAYDAY                                    │
│                                                                     │
│  11. Agent detects timeLeft == 0                                    │
│  12. YieldRouter withdraws all from pools                           │
│  13. Sends funds to PayrollDispatcher                               │
│  14. Dispatcher takes protocol fee from yield                       │
│  15. Returns remaining yield to employer                            │
│  16. Credits each employee's salary to PayVault                     │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                        EMPLOYEE SIDE                                │
│                                                                     │
│  17. Employee calls PayVault.claim() to claim full balance, or      │
│      PayVault.claimAndSave(amount, savePct, duration) to save a     │
│      portion and earn yield on it                                   │
│  18. Auto-Save portion → starts new YieldRouter cycle               │
│  19. Remainder → employee wallet                                    │
│  20. Employee bridges to preferred chain via Initia Bridge          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                      PayrollManager                          │
│          Employer-facing entry point. Manages groups,        │
│          employees, salaries, and cycle lifecycle.           │
└──────────────────────┬───────────────────────────────────────┘
                       │ startCycle() / cancelCycle()
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                       YieldRouter                            │
│       Core yield agent. Manages buffer ladder, pool          │
│       scoring, rebalancing, and payday settlement.           │
└──────────┬──────────────────────────────┬────────────────────┘
           │ deposit() / withdraw()        │ disburse()
           ▼                              ▼
┌─────────────────────┐      ┌───────────────────────────────┐
│   IPoolAdapter      │      │      PayrollDispatcher        │
│   (per pool)        │      │  Splits funds, takes fee,     │
│                     │      │  credits employees.           │
│  MockPoolAdapter    │      └──────────────┬────────────────┘
│  (local dev)        │                     │ credit()
└────────┬────────────┘                     ▼
         │                     ┌────────────────────────────┐
         ▼                     │          PayVault          │
┌─────────────────────┐        │  Employee savings layer.   │
│      MockPool       │        │  claim() or claimAndSave() │
│   (ERC4626 vault)   │        │  into new YieldRouter      │
│   Local dev only    │        │  cycles on employee's      │
└─────────────────────┘        │  behalf.                   │
                               └────────────────────────────┘

┌──────────────────────┐   ┌──────────────────────────────────┐
│   FlowrollCredit     │   │        FlowrollZapper            │
│  Salary advance and  │   │  Entry point for multi-step      │
│  debt repayment.     │   │  token routing and wrapping.     │
└──────────────────────┘   └──────────────────────────────────┘
```

---

## Repository Structure

This monorepo contains two submodules, each an independent repo:

```
flowroll/
├── flowroll-contract/     # Smart contracts + off-chain agent
├── flowroll-frontend/     # Next.js 15 frontend
├── .gitmodules
└── README.md
```

| Submodule | Description | README |
|---|---|---|
| `flowroll-contract` | Solidity contracts, Foundry tests, rebalance agent | [flowroll-contract/README.md](./flowroll-contract/README.md) |
| `flowroll-frontend` | Next.js frontend, Web3 integration, Initia InterwovenKit | [flowroll-frontend/README.md](./flowroll-frontend/README.md) |

---

## Getting Started
 
### Prerequisites
 
Make sure the following are installed before you begin:
 
- [Git](https://git-scm.com/)
- [Foundry](https://getfoundry.sh/): `curl -L https://foundry.paradigm.xyz | bash`
- [Node.js](https://nodejs.org/) v18+
- [npm](https://www.npmjs.com/)
- [Weave](https://docs.weave.xyz): Initia's appchain CLI
---
 
### 1. Clone the monorepo
 
Clone with submodules in one command:
 
```bash
git clone --recurse-submodules https://github.com/LanreAkintayo/flowroll.git
cd flowroll
```
 
If you already cloned without `--recurse-submodules`, pull the submodules manually:
 
```bash
git submodule update --init --recursive
```
 
---
 
### 2. Start the appchain
 
Flowroll runs on a local Initia appchain. Follow the official [Initia documentation](https://docs.initia.xyz/hackathon/get-started) to install and configure Weave, then start your appchain:
 
```bash
weave rollup start -d
```
 
The `-d` flag runs the appchain in detached mode. Your local EVM RPC will be available at `http://localhost:8545`.
 
> The appchain must be fully running before deploying contracts.
 
---
 
### 3. Deploy the smart contracts
 
Navigate into the contracts submodule:
 
```bash
cd flowroll-contract
forge install
forge build
```
 
Set up your environment:
 
```bash
cp .env.example .env
# Fill in TESTNET_PRIVATE_KEY, DEPLOYER, RPC, AGENT_OPERATOR, FEE_RECIPIENT
```
 
Deploy:
 
```bash
forge script script/Deploy.s.sol \
  --rpc-url $RPC \
  --private-key $TESTNET_PRIVATE_KEY \
  --broadcast
```
 
The script will print all deployed contract addresses. Copy them into your `.env` before continuing.
 
---
 
### 4. Wire the contracts
 
Once your `.env` is filled with deployed addresses:
 
```bash
chmod +x scripts/sh/wire.sh
./scripts/sh/wire.sh
```
 
This sets all cross-contract references and registers the pool adapters.
 
---
 
### 5. Seed the pools
 
```bash
chmod +x scripts/sh/seed.sh
./scripts/sh/seed.sh
```
 
Mints MockUSDC, deposits initial TVL into both pools, and funds the Zapper.
 
---
 
### 6. Start the agent
 
```bash
cd scripts/agent
npm install
cp .env.example .env
# Fill in PRIVATE_KEY, YIELD_ROUTER_ADDRESS, INITIA_EVM_RPC
npm start
```
 
The agent will begin monitoring active cycles and calling `agentRebalance()` at the configured interval
 
---
 
### 7. Run the frontend
 
In a new terminal, navigate to the frontend submodule:
 
```bash
cd flowroll-frontend
npm install
cp .env.example .env.local
# Fill in NEXT_PUBLIC_AGENT_URL, NEXT_PUBLIC_BRIDGED_INIT_ADDRESS
npm run dev
```
 
The app will be available at `http://localhost:3000`.
 
---
 
### Optional: Simulate Yield (Dev Only)
 
To inflate pool share prices and test the yield flow end to end, run from inside `flowroll-contract/`:
 
```bash
chmod +x scripts/sh/simulate-yield.sh
./scripts/sh/simulate-yield.sh
```
 
Then set the pool APYs:
 
```bash
chmod +x scripts/sh/change-pool-apys.sh
./scripts/sh/change-pool-apys.sh
```
 
> Only use these in local or testnet environments.
 