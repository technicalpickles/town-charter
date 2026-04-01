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

[Gas Town](https://steve-yegge.medium.com/welcome-to-gas-town-4f25ee16dd04) (Steve Yegge) is where the idea of a git-managed workspace directory with task tracking alongside code first clicked in practice. Pickletown took those structural patterns and built at a different point on the autonomy spectrum, starting with a single human-directed agent and growing from there. Town Charter extracts what proved durable into a standalone spec.

---

## Contributing

This spec is designed for contributions from both humans and AI assistants. Open issues to propose new concepts, question existing ones, or describe workflows you have found useful. Refine what is here. The spec improves through use.
