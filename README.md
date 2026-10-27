# Polygon VRF Lottery

A provably fair, on-chain lottery deployed to Polygon mainnet. Winners are drawn with **Chainlink VRF v2**, so neither the operator nor the players can predict or manipulate the outcome.

**Contract (Polygon):** [`0x6d4e52ef8f0c5573c2d47b41a580b66ab71bdcd9`](https://polygonscan.com/address/0x6d4e52ef8f0c5573c2d47b41a580b66ab71bdcd9)


## Stack

| Layer | Tech |
|---|---|
| Smart contract | Solidity 0.8.17, Truffle |
| Randomness | Chainlink VRF v2 (direct-funding wrapper) |
| Chain | Polygon PoS mainnet |
| Frontend | Next.js 12, React 18, web3.js, MetaMask |
| RPC / indexing | Alchemy SDK |
| Hosting | Vercel |

## How it works

```
Player ──enter() + MATIC──▶ Lottery.sol ──requestRandomness()──▶ Chainlink VRF
                                 ▲                                      │
                                 └──────── fulfillRandomWords() ◀───────┘
                                 │
                          pickWinner(): index = random % players.length
                                 │
                 winnings[winner] += pot − VRF fee − 10% commission
                                 │
                 Winner calls withdrawWinnings() (pull payment)
```

1. **Enter:** players call `enter()` with more than 0.01 MATIC. The entry is added to the pot.
2. **Request randomness:** the operator calls `getRandomNumber()`. The contract pays LINK to the VRF wrapper, and a `lowLink` event fires when the LINK balance runs low.
3. **Fulfill:** Chainlink returns a verifiable random word through `fulfillRandomWords()`.
4. **Draw:** `pickWinner()` selects `random % players.length`, records the winner in `lotteryHistory`, and resets the round.
5. **Payout:** winnings are credited to a mapping and claimed by the winner. A pull-payment pattern means a failed transfer can never block a round.

## Contract design

- **Fee split:** a fixed VRF reserve (0.005 MATIC) covers randomness costs, 10% of the remainder goes to the operator, and the rest goes to the winner. Each bucket is tracked separately (`winnings`, `operatorTotalCommission`, `vrf_matic`).
- **Checks-effects-interactions:** each balance is zeroed before its transfer to prevent reentrancy on withdrawals.
- **Access control:** an `isOperator` modifier gates draws and admin withdrawals, and `swapOwner()` transfers operator rights.
- **History:** `lotteryHistory` maps every round ID to its winner on-chain.

## Run locally

```bash
npm install
echo "ALCHEMY_API_KEY=your_key" > .env.local
npm run dev          # http://localhost:3000
```

Contract (`backend/`):

```bash
cd backend
truffle compile
truffle migrate --network <network>
```

## Retrospective

Built in January and February 2023 as my first production dApp. If I rebuilt it today:

- **Automate the draw** with Chainlink Automation so no operator has to trigger it. The operator role is currently a trust assumption.
- **Draw inside the VRF callback**, so the frontend never races the async fulfillment that `pickWinner()` depends on.
- **Replace `transfer()`** with `call{value:}` plus a reentrancy guard, since `transfer()` hard-codes a 2,300 gas stipend.
- **Add a Foundry test suite** with fuzzing on the fee math and a mocked VRF coordinator.
- **Estimate gas dynamically** instead of hard-coding limits and gas prices on the client.

## License

GPL-3.0
