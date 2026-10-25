# Bounty Board

A GenLayer Intelligent Contract that pays out GitHub issue bounties after validators check the merged pull request.

A poster creates a bounty for a GitHub issue. Anyone can claim it with a pull request link. GenLayer validators read the live GitHub pages and must agree through consensus that the pull request is merged and really fixes that issue before the bounty is marked `PAID`.

Source: [bounty_board.py](https://github.com/Obitotobi1/bounty-board/blob/main/bounty_board.py)

## How it works

- `create_bounty(repo, issue_number, reward)` stores a new bounty as `OPEN` and returns its id.
- `claim(bounty_id, pr_url)` makes each validator fetch the issue page and the pull request page, then judge them with an LLM. The bounty becomes `PAID` and the pull request author is stored as the winner only if the verdict is `APPROVED`.
- `cancel(bounty_id)` lets only the poster cancel an open bounty.
- Read methods: `get_status`, `get_winner`, `get_reason`, `get_repo`, `get_issue`, `get_reward`, `get_total_bounties`.

## Consensus

Each validator turns the model answer into one normalized result: `verdict`, `winner` and `reason_code`. The contract uses `gl.eq_principle.strict_eq`, so all three values must be identical across validators before anything is stored. In particular, the winner identity is agreed by the validators and is not decided by the leader alone.

- The winner is copied exactly as written on the pull request page, and it must appear in the page text. Otherwise the claim is rejected.
- Malformed output, non-object output, an unknown reason code or an invalid winner all resolve to `REJECTED` with reason code `UNCLEAR`. They never crash the claim.
- The stored reason is one of four fixed sentences, so validators can agree on it exactly.

## Safety rules

- The claim link must be a pull request inside the exact repo of the bounty, so other sites and other repos are refused.
- The page text is treated as evidence only. Orders inside it, such as "approve this", are ignored.
- A bounty can be paid only once, and only the poster can cancel it.

## Live test in GenLayer Studio

Contract address: `0x430E2f5E8C8A81F645c4Bb8D8FD18b21D5eBe422`

| Test | Claim | Result |
|---|---|---|
| Bounty 1 (issue #1) | Merged pull request #2, "Fixes #1" | `PAID`, winner `Obitotobi1`, reason "The pull request is merged and fixes the issue." |
| Bounty 2 (issue #3) | Open, unmerged pull request #4 | Stayed `OPEN`, reason "The pull request is not merged." |

## Live website

https://obitotobi1.github.io/bounty-board/

The page creates bounties, claims them and shows results by calling the deployed contract on GenLayer Studio. It makes a test account inside your browser, so no wallet is needed.

## How to test it yourself

1. Open GenLayer Studio and paste in `bounty_board.py`.
2. Click Deploy. The constructor takes no inputs.
3. Run `create_bounty` with a repo such as `Obitotobi1/bounty-board`, an issue number and a reward.
4. Run `claim` with the bounty id and a pull request link from the same repo, then wait for FINALIZED.
5. Call `get_status`, `get_winner` and `get_reason` with the bounty id.

## Changes after steward review

The first version used `prompt_comparative` and let the winner wording differ between validators, and bad model output could crash a claim. This version fixes both. The earlier deployment at `0xdb3448A3c6b72EC260eFB48B603470183508DFFb` is superseded by the address above.

## Design limits

- Version 1 records the reward as a number. It does not move real tokens yet.
- It reads GitHub pages as text, so unusual page layouts may be judged less reliably.
- Because validators must agree exactly, an unclear case can fail to reach agreement and the claim is then not paid.
