# hemm-ems

What I'm trying to build here: a home-energy manager that's versatile, precise, and
made to run for years — and the tooling to set systems like it up efficiently with AI.
Open source, work in progress.

The approach across all of it is cautious by design: observe before acting, verify
before trusting, small steps to earn confidence. I also try not to reinvent things —
everything here is something I needed, couldn't find anywhere, and so ended up
building myself. On that mission I found (to my knowledge) new ways to use AI agents 
to build code efficiently and trustworthily (I learn something every day).

## The energy manager

- **[hemm](https://github.com/hemm-ems/hemm)** /
  **[ha-hemm](https://github.com/hemm-ems/ha-hemm)** — reads device manifests,
  constraints, and price/solar forecasts and plans 24 h of power for Home Assistant.
  It observes and plans first; taking control is opt-in. **Soft-beta**: it installs
  and runs, but it's still moving.

## Tooling to build and run it with AI

- **[hactl](https://github.com/hemm-ems/hactl)** — a CLI for driving Home Assistant
  from LLM agents, token-efficient by design. The most finished thing here.
- **[hactl-companion](https://github.com/hemm-ems/hactl-companion)** — adds
  create/update/delete of HA entities on top of hactl.

## Methods I couldn't find elsewhere, so I built them

- **Cross-model agentic development** — one model (Claude Opus) plans and reviews
  while another (OpenAI Codex) writes the code, so every change is read by a model
  other than the one that wrote it. The reason I think it helps: a model waves through
  its own output but picks apart another's, so review across two models catches more
  than self-review. → [the full writeup](https://github.com/hemm-ems/.github/blob/main/profile/agentic-spec-dev.md)
- **Testing the EMS against a living Home Assistant** — the integration tests run
  against a real, ephemeral HA instance in Docker via hactl, not mocks, so the plans
  are checked against the system that actually runs them.

<!-- discussed-on placeholder: add HN / article links here once there's a real one -->

Built by [Jan Kipping](https://kipp.ing), embedded and automotive software engineer.
