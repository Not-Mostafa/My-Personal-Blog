<div align="center">

# t5um1x

**Offensive security notes, CTF write-ups, and the occasional rabbit hole.**

[![Blog](https://img.shields.io/badge/blog-mostafa--gbr.vercel.app-5865F2?style=for-the-badge&logo=vercel&logoColor=white)](https://mostafa-gbr.vercel.app/)
[![TryHackMe](https://img.shields.io/badge/TryHackMe-3mkMostafa-212C42?style=for-the-badge&logo=tryhackme&logoColor=white)](https://tryhackme.com/p/3mkMostafa)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Mostafa%20Gbr-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/mostafa-hosam-gbr/)
[![Discord](https://img.shields.io/badge/Discord-t5um1x-5865F2?style=for-the-badge&logo=discord&logoColor=white)](https://discord.com/users/548040857509429248)

</div>

---

## 👋 Who's behind this

I'm **Mostafa**, a Computer Engineering student at Nile University who likes breaking things to understand how they work.

I started on the blue side as an **L1 SOC Analyst**, and I've since moved to **offensive security and red teaming**: web and API exploitation, privilege escalation, and popping shells on Hack The Box and TryHackMe. When I'm not in a terminal, I'm usually wiring up STM32 / ESP8266 / Arduino boards or reading assembly to see what a binary is really doing.

This repo is the source of my blog, where I write up the boxes I solve and the things I learn along the way.

## 🗂️ What you'll find on the blog

| Post | Platform | What it covers |
|:-----|:---------|:---------------|
| **Nexus** | Hack The Box | From an exposed Git config to root through a vulnerable Gitea template synchroniser |
| **Building a Malicious Git Tree** | Hack The Box | Deep dive on the Nexus privesc: hand-crafted Git objects that bypass path validation for an arbitrary file write as root |
| **Corridor** | TryHackMe | Enumeration, web analysis, and MD5 hash discovery to grab the flag |
| **ISC2 CC Study Notes** | Certification | All-in-one notes for the ISC2 Certified in Cybersecurity exam |

More write-ups land as I finish boxes. Browse them all in the [archive](https://mostafa-gbr.vercel.app/archive/).

## 🎯 Focus areas

- **Web & API exploitation**: injection, auth flaws, misconfigurations, and chaining small bugs into big ones
- **Privilege escalation**: Linux internals, misconfigured services, and abusing the tools that run as root
- **Red teaming & recon**: simulating real attacks end to end
- **Reverse engineering**: dropping down to assembly to find what the source won't tell you
- **Hardware tinkering**: microcontrollers, sensors, and small automation projects

## 🛠️ How the blog is built

The site is built with [Astro](https://astro.build) on top of the [Fuwari](https://github.com/saicaca/fuwari) theme, styled with Tailwind CSS, searchable with Pagefind, and deployed on Vercel. Posts are Markdown files under [`src/content/posts/`](src/content/posts/).

```sh
pnpm install          # install dependencies
pnpm dev              # local dev server at localhost:4321
pnpm new-post <name>  # scaffold a new post
pnpm build            # production build to ./dist/
```

Site settings (name, bio, links, theme) live in [`src/config.ts`](src/config.ts).

## 📄 License

- **Write-ups and post content** are licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/). Share and adapt them with credit, non-commercially.
- **Site code** is based on [Fuwari](https://github.com/saicaca/fuwari) and is available under the [MIT License](LICENSE).

<div align="center">

---

*Got a box you want to talk through, a CTF team that needs a player, or a project idea? [Reach out.](https://www.linkedin.com/in/mostafa-hosam-gbr/)*

</div>
