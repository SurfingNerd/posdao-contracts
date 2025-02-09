# POSDAO Smart Contracts

Implementation of the POSDAO consensus algorithm in [Solidity](https://solidity.readthedocs.io).

## About

POSDAO is a Proof-of-Stake (POS) algorithm implemented as a decentralized autonomous organization (DAO). It is designed to provide a decentralized, fair, and energy efficient consensus for public chains. The algorithm works as a set of smart contracts written in Solidity. POSDAO is implemented with a general purpose BFT consensus protocol such as AuthorityRound (AuRa) with a leader and probabilistic finality, or Honeybadger BFT (HBBFT), leaderless and with instant finality. It incentivizes actors to behave in the best interests of a network.

The algorithm provides a Sybil control mechanism for reporting malicious validators and adjusting their stake, distributing a block reward, and managing a set of validators. The authors implement the POSDAO for a sidechain based on Ethereum 1.0 protocol.

## POSDAO Repositories and Resources

- White paper: https://forum.poa.network/t/posdao-white-paper/2208
- Backported OpenEthereum client with POSDAO features: https://github.com/poanetwork/open-ethereum/tree/posdao-backport (v2.7.2)
- Original OpenEthereum client supporting POSDAO features: https://github.com/openethereum/openethereum/tree/dev (> v3.2.1)
- Nethermind client supporting POSDAO features: https://github.com/NethermindEth/nethermind
- Integration tests setup for a POSDAO network: https://github.com/poanetwork/posdao-test-setup
- Discussion forum: https://forum.poa.network/c/posdao

## Smart Contract Summaries

_Note: The following descriptions are for AuRa contracts only. HBBFT contract implementations are not started yet and are not listed nor described here. All contracts are located in the `contracts` directory._

- `BlockRewardAuRa`: generates and distributes rewards according to the logic and formulas described in the white paper. Main features include:
  - distributes the entrance/exit fees from the `erc-to-erc`, `native-to-erc`, and/or `erc-to-native` bridges among validators pools;
  - mints native coins needed for the `erc-to-native` bridge;
  - makes a snapshot of the validators stakes at the beginning of each staking epoch. That snapshot is used by the `StakingAuRa.claimReward` function to transfer rewards to validators and their delegators.

- `Certifier`: allows validators to use a zero gas price for their service transactions (see [OpenEthereum Wiki](https://openethereum.github.io/Permissioning.html#gas-price) for more info). The following functions are considered service transactions:
  - ValidatorSetAuRa.emitInitiateChange
  - ValidatorSetAuRa.reportMalicious
  - RandomAura.commitHash
  - RandomAura.revealNumber

- `InitializerAuRa`: used once on network startup and then destroyed. This contract is needed for initializing upgradable contracts since an upgradable contract can't have the constructor. The bytecode of this contract is written by the `scripts/make_spec.js` into `spec.json` along with other contracts when initializing on genesis block.

- `RandomAuRa`: generates and stores random numbers in a [RANDAO](https://github.com/randao/randao) manner (and controls when they are revealed by AuRa validators). Random numbers are used to form a new validator set at the beginning of each staking epoch by the `ValidatorSet` contract. Key functions include:
  - `commitHash` and `revealNumber`. Can only be called by the validator's node when generating and revealing their secret number (see [RANDAO](https://github.com/randao/randao) to understand principle). Each validator node must call these functions once per `collection round`. This creates a random seed which is used by `ValidatorSetAuRa` contract. See the white paper for more details;
  - `onFinishCollectRound`. This function is automatically called by the `BlockRewardAuRa` contract at the end of each `collection round`. It controls the reveal phase for validator nodes and punishes validators when they don’t reveal (see the white paper for more details on the `banning` protocol);
  - `currentSeed`. This public getter is used by the `ValidatorSetAuRa` contract at the latest block of each staking epoch to get the accumulated random seed for randomly choosing new validators among active pools. It can also be used by anyone who wants to use the network's random seed. Note, that its value is only updated when `revealNumber` function is called: that's expected to be occurred at least once per `collection round` which length in blocks can be retrieved with the `collectRoundLength` public getter. Since the revealing validator always knows the next random number before sending, your DApp should restrict any business logic actions (that depends on random) during `reveals phase`.

    <details>
    <summary>An example of how the seed could be retrieved by an external smart contract.</summary>

    ```solidity
    pragma solidity 0.5.11;


    interface IPOSDAORandom {
        function collectRoundLength() external view returns(uint256);
        function currentSeed() external view returns(uint256);
    }

    contract Example {
        IPOSDAORandom private _posdaoRandomContract; // address of RandomAuRa contract
        uint256 private _seed;
        uint256 private _seedLastBlock;
        uint256 private _updateInterval;

        constructor(IPOSDAORandom _randomContract) public {
            require(_randomContract != IPOSDAORandom(0));
            _posdaoRandomContract = _randomContract;
            _seed = _randomContract.currentSeed();
            _seedLastBlock = block.number;
            _updateInterval = _randomContract.collectRoundLength();
            require(_updateInterval != 0);
        }

        function useSeed() public {
            if (_wasSeedUpdated()) {
                // using updated _seed ...
            } else {
                // using _seed ...
            }
        }

        function _wasSeedUpdated() private returns(bool) {
            if (block.number - _seedLastBlock <= _updateInterval) {
                return false;
            }

            _updateInterval = _posdaoRandomContract.collectRoundLength();

            uint256 remoteSeed = _posdaoRandomContract.currentSeed();
            if (remoteSeed != _seed) {
                _seed = remoteSeed;
                _seedLastBlock = block.number;
                return true;
            }
            return false;
        }
    }
    ```
    </details>

- `Registry`: stores human-readable keys associated with addresses, like DNS information. This contract is needed primarily to store the address of the `Certifier` contract (see [OpenEthereum Wiki](https://openethereum.github.io/Service-transaction-checker-contract.html) for details).

- `StakingAuRa`: contains staking logic including:
  - creating, storing, and removing pools by candidates and validators;
  - staking tokens by participants (delegators, candidates, or validators) into the pools;
  - storing participants’ stakes;
  - withdrawing tokens and rewards by participants from the pools;
  - moving tokens between pools by participant.

- `TokenMinter`: used when we need to have an ability to mint POSDAO tokens not only by the bridge, but also by the `BlockRewardAuRa` contract if the staking token contract doesn't support a `mintReward` function.

- `TxPermission`: along with the `Certifier` contract, controls the use of zero gas price by validators in service transactions, protecting the network against "transaction spamming" by malicious validators. The protection logic is declared in the `allowedTxTypes` function.

- `TxPriority`: manages and stores the transactions priority list used by Ethereum client. See https://github.com/NethermindEth/nethermind/issues/2300 for description.

- `ValidatorSetAuRa`: stores the current validator set and contains the logic for choosing new validators at the beginning of each staking epoch. The logic uses a random seed generated and stored by the `RandomAuRa` contract. Also, ValidatorSetAuRa is responsible for discovering and removing malicious validators. This contract is based on `reporting ValidatorSet` [described in OpenEthereum Wiki](https://openethereum.github.io/Validator-Set.html#reporting-contract).

For a detailed description of each function of the contracts, see their source code.

## Usage

### Install Dependencies

```bash
$ npm install
```

### Testing

_Note: Test development for unit testing and integration testing is in progress._

Integration test setup is available here: https://github.com/poanetwork/posdao-test-setup

To run unit tests:

```bash
$ npm run test 
```

### Flatten

Flattened contracts can be used to verify the contract code in a block explorer like BlockScout or Etherscan. See https://docs.blockscout.com/for-users/smart-contract-interaction/verifying-a-smart-contract for Blockscout verification instructions.

To prepare flattened version of the contracts:

```bash
$ npm run flat
```

Once flattened, the contracts are available in the `flat` directory.

## Contributing

See the [CONTRIBUTING](CONTRIBUTING.md) document for contribution, testing and pull request protocol.

## License

Licensed under either of:

-   Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or <http://www.apache.org/licenses/LICENSE-2.0>)
-   MIT license ([LICENSE-MIT](LICENSE-MIT) or <http://opensource.org/licenses/MIT>)

at your option.
