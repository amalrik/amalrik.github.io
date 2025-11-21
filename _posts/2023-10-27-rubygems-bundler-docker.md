---
layout: post
title: "RubyGems, Bundler, and Docker: The Complete Guide to Private Gem Authentication"
date: 2025-11-20 21:00:00 -0300
categories: [ruby, docker, devops, bundler]
tags: [ruby-on-rails, jfrog, artifactory, troubleshooting]
---

Have you ever faced a situation where `bundle install` runs perfectly in CI or inside a Docker container, but fails miserably on your local machine with a `Bad username or password` error?

I recently encountered this exact scenario while configuring a private repository (JFrog/Artifactory). I learned some valuable lessons about how Ruby manages configurations and realized that to have a fully working environment, you often need to configure both **Bundler** and **RubyGems** explicitly.

Here is the technical summary of what went wrong and the complete solution.

### 1. The Context: Docker vs. Local Machine

In my case, the project worked inside Docker but not locally. The reason lay in the location of the configuration files:

* **Inside Docker:** The image or build process had credentials injected into a global system configuration file (e.g., `/usr/local/bundle/config`).
* **Locally:** I had a `.bundle/config` file inside the project, but it only contained path configurations without credentials. Since I didn't have a global config (`~/.bundle/config`) set up, authentication failed.

### 2. The Solution: Configuring Both Worlds

To fix this permanently and ensure I could run both `bundle install` (project context) and `gem install` (global context), I had to run two distinct commands.

#### Step 1: Configuring Bundler (The Project Fix)

Bundler is hermetic; it ignores your global gem sources and looks strictly at the `Gemfile`. To authorize Bundler to access the private repo listed in the Gemfile, I used `bundle config`.

Since the Docker container had the config at `/usr/local/bundle/config`, I copied the credentials from there and applied them to my local project:

```bash
# Syntax
bundle config set <REPO_URL> <USERNAME>:<PASSWORD>

# Practical Example
bundle config set https://my-repo.jfrog.io/api/gems/ myuser:mypassword
```

*Tip: Use `--global` if you don't want to store this in the project's `.bundle/config` file.*

#### Step 2: Configuring RubyGems (The Global Fix)

Even after fixing Bundler, I realized that if I tried to install a specific gem manually (e.g., for debugging), it would still fail because the global `gem` command doesn't read Bundler's config.

To ensure consistency across the entire system, I also added the source with credentials directly to RubyGems:

```bash
gem sources --add https://myuser:obfuscated-password@my-repo.jfrog.io/api/gems/gems/ -V
```

**Why do this step?**
While Bundler doesn't strictly *need* this to run `bundle install`, adding this ensures that:
1.  You can verify access instantly with `gem sources --list`.
2.  You can run `gem install my-private-gem` directly from the terminal without Bundler.

### 3. Security Warning

When running the commands above, be mindful of where the credentials are stored:

* **Bundler:** If you run `bundle config` without `--global`, it saves to `.bundle/config` inside your project. **Make sure this file is in `.gitignore`** so you don't commit your password.
* **RubyGems:** The `gem sources --add` command saves your URL (with the password) into `~/.gemrc`. This is generally safe as it stays on your machine, but be aware that the password is stored in plain text there.

### Summary (Cheat Sheet)

| Tool | Command Used | Where it saves | Purpose |
| :--- | :--- | :--- | :--- |
| **Bundler** | `bundle config set <url> <creds>` | `.bundle/config` (local) or `~/.bundle/config` (global) | Allows `bundle install` to fetch gems for the project. |
| **RubyGems** | `gem sources --add <url_with_creds>` | `~/.gemrc` | Allows global `gem install` commands to work. |

By configuring both, I ensured my local environment was just as capable as the Docker container, eliminating the "It works on my machine" (or rather, "It doesn't work on my machine") paradox.