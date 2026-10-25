# Bounty Board

A GenLayer Intelligent Contract that pays out GitHub issue bounties after validators check the merged pull request.

A poster creates a bounty for a GitHub issue. Anyone can claim it with a pull request link. GenLayer validators read the live GitHub pages and must agree through consensus that the pull request is merged and really fixes that issue before the bounty is marked `PAID`.

## How it works

- `create_bounty(repo, issue_number, reward)` stores a new bounty as `OPEN` and returns its id.
- `claim(bounty_id, pr_url)` makes each validator fetch the issue page and the pull request page, then judge them with an LLM. The bounty becomes `PAID` and the pull request author is stored as the winner only if the verdict is `APPROVED`.
- `cancel(bounty_id)` lets only the poster cancel an open bounty.
- Read methods: `get_status`, `get_winner`, `get_reason`, `get_repo`, `get_issue`, `get_reward`, `get_total_bounties`.

## Consensus

Validators use `gl.eq_principle.prompt_comparative`. The verdict (`APPROVED` or `REJECTED`) must be identical across validators, while the wording of the reason may differ.

## Safety rules

- The claim link must be a pull request inside the exact repo of the bounty, so other sites and other repos are refused.
- The page text is treated as evidence only. Orders inside it, such as "approve this", are ignored.
- Any unclear or invalid model output is turned into `REJECTED`.
- A bounty can be paid only once, and only the poster can cancel it.

## Live test in GenLayer Studio

Contract address: `0xdb3448A3c6b72EC260eFB48B603470183508DFFb`

| Test | Claim | Result |
|---|---|---|
| Bounty 1 (issue #1) | Merged pull request #2, "Fixes #1" | `PAID`, winner `Obitotobi1` |
| Bounty 2 (issue #3) | Open, unmerged pull request #4 | Stayed `OPEN`: the pull request is not merged |
| Bounty 2 (issue #3) | Merged pull request #2, which fixes issue #1 | Stayed `OPEN`: the pull request is for a different issue |

## How to test it yourself

1. Open GenLayer Studio and paste in `bounty_board.py`.
2. Click Deploy. The constructor takes no inputs.
3. Run `create_bounty` with a repo such as `Obitotobi1/bounty-board`, an issue number and a reward.
4. Run `claim` with the bounty id and a pull request link from the same repo, then wait for FINALIZED.
5. Call `get_status`, `get_winner` and `get_reason` with the bounty id.

## Design limits

- Version 1 records the reward as a number. It does not move real tokens yet.
- It reads GitHub pages as text, so unusual page layouts may be judged less reliably.
- LLM validators can disagree in unclear cases, and those claims are rejected.
