# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability in this project, please report it responsibly.

**Do NOT open a public GitHub issue.**

Instead, open a [GitHub Security Advisory](https://github.com/opscairn/homelab-ansible-stack/security/advisories/new) or email: **security@opscairn.com**

Include:
- A description of the vulnerability
- Steps to reproduce
- Any potential impact

You should receive an acknowledgement within 48 hours. We will work with you to understand and address the issue before any public disclosure.

## Scope

This project contains Ansible playbooks and roles for homelab automation. Security concerns most likely involve:

- Secrets leaked in playbooks, templates, or variable files
- Unsafe shell commands or injection vectors in task definitions
- Overly permissive file modes on sensitive files (vault, SSH keys, password files)

## Best Practices for Users

- Always use `ansible-vault` for secrets — never commit plaintext passwords
- Restrict SSH key access to the Ansible control node
- Run playbooks with `--check` before applying to production
- Rotate your vault password periodically
