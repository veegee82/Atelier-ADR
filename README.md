# Atelier-ADR — Private Architecture & Strategy Archive

This repository contains internal decision records, business strategy,
and planning documents for AtelierOS. It is **not public**.

## Contents

| Directory | What is in here |
|---|---|
| `decisions/` | All Architecture Decision Records (ADR-0001 – ADR-0061+) |
| `business/` | Go-to-market strategy, IP protection, open-core boundary, license playbook |
| `internal/` | Internal planning docs, launch checklists, code review findings, roadmaps |

## Top-level files

| File | What it is |
|---|---|
| `phase7-backlog.md` | Multi-tenant Phase 7 backlog |
| `phase-29.5.3-soak-followups.md` | L29.5.3 soak follow-ups |
| `concept-os-completion.md` | Conceptual OS completion notes |
| `full-integration-test-suite.md` | Full integration test suite documentation |

## Relationship to AtelierOS

The public AtelierOS repo contains user-facing documentation only:
architecture, compliance, layer references, setup guides, and the
compliance hub. ADRs and business strategy are maintained exclusively here.

When writing new ADRs, create them here first; reference them from
AtelierOS CLAUDE.md or public docs only by ADR number and title.
