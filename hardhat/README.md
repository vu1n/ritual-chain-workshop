# Privacy-Preserving AI Bounty Judge

This Hardhat project implements the required **commit-reveal bounty flow** in `contracts/AIJudge.sol` and documents a Ritual-native hidden-submission design for the advanced track.

## Contract lifecycle

1. **Create bounty**
   - The bounty owner calls `createBounty(title, rubric, deadline)` and sends the reward as `msg.value`.
   - `deadline` must be in the future.

2. **Commit phase: before `deadline`**
   - A participant computes:

     ```solidity
     keccak256(abi.encodePacked(answer, salt, participantAddress, bountyId))
     ```

   - They submit only that hash with:

     ```solidity
     submitCommitment(uint256 bountyId, bytes32 commitment)
     ```

   - The plaintext answer and salt are not stored on-chain during this phase.
   - Each address may commit once per bounty. The contract caps total submissions at `MAX_SUBMISSIONS`.

3. **Reveal phase: after `deadline`**
   - A participant calls:

     ```solidity
     revealAnswer(uint256 bountyId, string calldata answer, bytes32 salt)
     ```

   - The contract recomputes the commitment using `answer`, `salt`, `msg.sender`, and `bountyId`.
   - Only matching reveals are accepted and marked eligible. Bad salts, wrong answers, wrong senders, and duplicate reveals fail.

4. **Batch AI judging**
   - The owner calls:

     ```solidity
     judgeAll(uint256 bountyId, bytes calldata llmInput)
     ```

   - `judgeAll` is allowed only after the deadline and only if at least one answer has been revealed.
   - The contract sends the supplied batch prompt/input to Ritual's LLM inference precompile and stores the returned AI review bytes.
   - The intended prompt should include the rubric and all revealed submissions in one batch, not one LLM call per answer.

5. **Finalize winner**
   - The owner calls:

     ```solidity
     finalizeWinner(uint256 bountyId, uint256 winnerIndex)
     ```

   - The selected index must refer to a revealed submission.
   - The contract pays the bounty reward to the winner and records the final result.

## Required functions implemented

- `submitCommitment(uint256 bountyId, bytes32 commitment)`
- `revealAnswer(uint256 bountyId, string calldata answer, bytes32 salt)`
- `judgeAll(uint256 bountyId, bytes calldata llmInput)`
- `finalizeWinner(uint256 bountyId, uint256 winnerIndex)`

Additional helper/view functions:

- `computeCommitment(...)`
- `getBounty(...)`
- `getSubmission(...)`
- `getSubmissionIndex(...)`
- `revealedSubmissionCount(...)`

## Test plan for reveal cases

Minimum tests to run against `AIJudge.sol`:

1. **Happy path**: create bounty, commit before deadline, reveal after deadline with the correct answer and salt, judge, finalize revealed winner.
2. **No plaintext leak before reveal**: after `submitCommitment`, `getSubmission` returns the commitment, `revealed == false`, and an empty answer.
3. **Cannot reveal early**: `revealAnswer` before `deadline` reverts with `reveal not open`.
4. **Cannot commit late**: `submitCommitment` at or after `deadline` reverts with `submission phase closed`.
5. **Wrong salt fails**: reveal with a different salt reverts with `invalid reveal`.
6. **Wrong answer fails**: reveal with changed answer text reverts with `invalid reveal`.
7. **Wrong sender fails**: another address cannot reveal someone else's commitment because `msg.sender` is part of the hash.
8. **Duplicate commit fails**: the same address cannot submit two commitments for one bounty.
9. **Duplicate reveal fails**: a revealed submission cannot be revealed again.
10. **Unrevealed entries are ineligible**: `judgeAll` requires at least one reveal, and `finalizeWinner` rejects an unrevealed submission index.
11. **Winner bounds**: `finalizeWinner` rejects an out-of-range winner index.
12. **Max submission cap**: the 11th submission reverts when `MAX_SUBMISSIONS == 10`.

## Architecture note: commit-reveal track

The contract stores only commitment hashes during the competitive submission window. This prevents participants from copying or slightly improving public plaintext answers before the deadline. After the deadline, reveals become public on-chain, but late copying is no longer useful because new commitments are closed. The AI judge receives only revealed answers in a single batch prompt prepared by the bounty owner or frontend, then the returned review is stored on-chain. Final reward transfer is deterministic Solidity logic controlled by `finalizeWinner`, while the AI review is an advisory/evaluative artifact that helps choose the winner.

## Architecture note: Ritual-native hidden submissions advanced design

A stronger Ritual-native version can keep plaintext hidden even during judging:

- **On-chain storage**: bounty metadata, reward escrow, participant addresses, ciphertext/content hashes, reveal or decryption status, AI review hash/bytes, winner index, and payout state.
- **Off-chain storage**: encrypted answer payloads in IPFS/S3/Arweave or a similar content-addressed store. The chain stores only content identifiers and hashes.
- **Plaintext location**: plaintext exists first on the participant's device before encryption. During judging, plaintext exists only inside Ritual TEE-backed execution/private-input handling long enough to decrypt, batch all answers, call the LLM, and produce a signed or attestable result. Plaintext is never posted directly on-chain.
- **Key flow**: participants encrypt answers to a Ritual/TEE-managed public key or use Ritual encrypted secrets/private inputs. The decryption key is unavailable to the bounty owner and other participants.
- **Batch judging**: after the submission deadline, a single Ritual job loads all ciphertexts, decrypts them inside the TEE, constructs one rubric-aware batch prompt, sends all submissions to the LLM together, and returns the ranked result/review.
- **Result anchoring**: the contract stores the AI result, a hash of the judged batch, and optionally an attestation proving the TEE code path and inputs. The bounty owner then finalizes a winner from the judged eligible set.

This design leverages Ritual for private inputs and TEE-backed batch inference rather than making one LLM call per answer.

## Reflection question

What should be public, what should stay hidden, and what should be decided by AI versus by a human in a bounty system? Bounty rules, deadlines, reward amounts, commitment hashes, eligibility state, final winner, and payout transactions should be public so participants can audit fairness. Answers should stay hidden during the submission phase so nobody can copy another participant's work. Salts and any private authoring process should remain secret permanently unless the participant chooses to disclose them. AI should help evaluate submissions against the stated rubric, summarize tradeoffs, and rank candidates consistently in a batch. Humans should set the rubric, decide whether the AI result is reasonable, handle edge cases, and accept responsibility for the final award. The best system uses transparent smart-contract rules for custody and deadlines, privacy for submissions until the right time, AI for scalable review, and human judgment for accountability.

## Useful commands

```shell
pnpm install
pnpm hardhat compile
pnpm hardhat test
```
