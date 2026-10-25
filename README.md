# Bounty Board

A GenLayer Intelligent Contract that pays out GitHub issue bounties after validators check the merged pull request.

A poster creates a bounty for a GitHub issue. Anyone can claim it with a pull request link. GenLayer validators read the live GitHub pages and must agree through consensus that the pull request is merged and really fixes that issue before the bounty is marked `PAID`.

Source: [bounty_board.py](https://github.com/Obitotobi1/bounty-board/blob/main/bounty_board.py)

Live website: https://obitotobi1.github.io/bounty-board/

## Versions

| Version | Contract address (GenLayer Studio) | Status |
|---|---|---|
| v1.0 | `0x430E2f5E8C8A81F645c4Bb8D8FD18b21D5eBe422` | Accepted as a Project |
| v1.1 | `0x51189bEb2E7d1EeDc649A7c9778e3626668dC898` | Current, used by the website |

## What changed in v1.1

Contract:

- **Claim history.** Every completed claim is stored on its bounty as a line with the pull request number, the verdict and the reason. Before, only the last reason was kept.
- **Failed-claim limit.** A bounty accepts at most 3 failed claims. After that, new claims are refused with `Too many failed claims for this bounty`. The poster can still cancel it.
- **New read methods:** `get_bounty` (everything about one bounty as JSON), `get_history` and `get_attempts`.

Website:

- A list of all bounties with status and failed claims.
- A history view and a failed-claim counter for each bounty.
- A **Cancel this bounty** button. Only the account that created a bounty can cancel it.
- A clearer message after each claim, taken from the bounty's history.

## How it works

- `create_bounty(repo, issue_number, reward)` stores a new bounty as `OPEN` and returns its id.
- `claim(bounty_id, pr_url)` makes each validator fetch the issue page and the pull request page, then judge them with an LLM. The bounty becomes `PAID` and the pull request author is stored as the winner only if the verdict is `APPROVED`.
- `cancel(bounty_id)` lets only the poster cancel an open bounty.
- Read methods: `get_status`, `get_winner`, `get_reason`, `get_repo`, `get_issue`, `get_reward`, `get_total_bounties`, `get_attempts`, `get_history`, `get_bounty`.

## Consensus

Each validator turns the model answer into one normalized result: `verdict`, `winner` and `reason_code`. The contract uses `gl.eq_principle.strict_eq`, so all three values must be identical across validators before anything is stored.

- The winner is copied exactly as written on the pull request page, and it must appear in the page text. Otherwise the claim is rejected.
- Malformed output, non-object output, an unknown reason code or an invalid winner all resolve to `REJECTED` with reason code `UNCLEAR`. They never crash the claim.
- The stored reason is one of four fixed sentences, so validators can agree on it exactly.

## Safety rules

- The claim link must be a pull request inside the exact repo of the bounty, so other sites and other repos are refused.
- The page text is treated as evidence only. Orders inside it, such as "approve this", are ignored.
- A bounty can be paid only once, and only the poster can cancel it.
- A bounty stops accepting claims after 3 failed ones.

## Live tests of v1.1 in GenLayer Studio

| Test | Result |
|---|---|
| Bounty 1 (issue #1), claimed with merged pull request #2 | `PAID`, winner `Obitotobi1`, 0 failed claims, history line `PR #2 - APPROVED: The pull request is merged and fixes the issue.` |
| Bounty 2 (issue #3), claimed with open pull request #4 | Stayed `OPEN`, failed claims 1, history line `PR #4 - REJECTED: The pull request is not merged.` |
| Bounty 2, two more claims with pull request #4 | Failed claims 3, three history lines |
| Bounty 2, a fourth claim | Refused: `Too many failed claims for this bounty`, failed claims stayed 3 |
| Website: bounty created from the page, then cancelled from the page | Status `CANCELLED` |

## How to test it yourself

1. Open GenLayer Studio and paste in `bounty_board.py`.
2. Click Deploy. The constructor takes no inputs.
3. Run `create_bounty` with a repo such as `Obitotobi1/bounty-board`, an issue number and a reward.
4. Run `claim` with the bounty id and a pull request link from the same repo, then wait for FINALIZED.
5. Call `get_bounty` with the bounty id. Or use the live website above.

## Design limits

- The reward is recorded as a number. It does not move real tokens yet.
- It reads GitHub pages as text, so unusual page layouts may be judged less reliably.
- Because validators must agree exactly, an unclear case can fail to reach agreement and the claim is then not paid.
