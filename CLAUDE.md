# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Project Overview

This is the **website-skills** repository — Milo's conventions for building production-grade Next.js + Tailwind websites on the DeployNow stack, packaged as a Claude Code skill.

**Repository**: https://github.com/milojarow/website-skills

## Repository Structure

```
website-skills/
├── .claude-plugin/          # Claude Code plugin configuration
├── CLAUDE.md                # This file
├── README.md                # Project overview
├── LICENSE                  # MIT License
├── evaluations/             # Test scenarios for the skill
│   └── website/
└── skills/
    └── website/
        ├── SKILL.md         # Entry point: stack, server-first, layout, pointers
        └── reference/       # Depth: stack-and-conventions, server-first, layout, performance, deploy
```

## The skill

### website
Conventions for building Next.js App Router + Tailwind/DaisyUI sites on the DeployNow stack: server-first RSC/SSR discipline, file/component naming, layout stability + the trace-upward debugging method, WebP images + Web Vitals, async error handling, external-CDN/no-double-hosting, and Vercel deploy.

## Skill Activation

Activates when building or editing a website/web app on this stack — scaffolding pages/components, naming, server-vs-client decisions, layout fixes, image/Web-Vitals work, or deploy.

## Updating this skill

After any session that discovers a new pattern or wall. Keep entries generic — patterns and examples, never client data. The git log of this repo is the diary.
