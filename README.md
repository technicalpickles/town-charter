# town-charter

A spec for AI-assisted multi-repo development workspaces.

---

## What is a town?

A town is a git repo that organizes multiple external repositories, tracks work across them, and teaches AI assistants your conventions. It gives your development work a home that is versioned, shareable, and durable. Towns are meant to be inhabited: you commit plans, session handoffs, and working notes alongside the tooling that helps you navigate them.

## What is this repo?

This repo is the spec. It describes the concepts, motivations, and interfaces that make a town work. It is for humans designing their workspace and AI assistants operating within one. There is nothing to install.

[Read the spec](spec.md)

---

## Origin

Gastown (Steve Yegge) introduced the idea of a git-managed workspace directory alongside your code, with task tracking and session management as first-class concerns. Pickletown rebuilt that foundation around convention-driven AI collaboration. Town Charter extracts the patterns that proved durable into a standalone spec that any implementation can follow.

---

## Contributing

This spec is designed for contributions from both humans and AI assistants. Open issues to propose new concepts, question existing ones, or describe workflows you have found useful. Refine what is here. The spec improves through use.
