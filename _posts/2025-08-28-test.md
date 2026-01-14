---
title: Guarding Elastic Beanstalk Production Deploys with a Simple Shell Wrapper
date: 2026-01-14 18:00:00
categories: [DevOps]
tags: [devops, aws, eb-cli, elastic-beanstalk, shell]
---

If you’ve ever typed `eb deploy` with a bit too much confidence, you know the feeling. The command runs instantly, the output starts scrolling, and only then do you notice the environment name. *Production.*

At that point, it’s already too late.

Elastic Beanstalk’s CLI is deliberately simple and script‑friendly. That’s great for automation, but it also means there’s no built‑in concept of a “protected” environment. No confirmation prompt. No warning. No last line of defense when muscle memory takes over.

This post walks through a small shell wrapper that adds exactly that missing safety net — without breaking workflows, slowing down your terminal, or getting in the way of CI.

---

## The problem: EB CLI trusts you completely

The EB CLI will happily deploy to any environment you point it at:

```shell
eb deploy example-api-production --staged
```

Flags can come before or after the environment name. You can omit the environment entirely and deploy to the default one. From the CLI’s perspective, all of these are equally valid.

That design makes sense for automation, but for humans it’s risky. Most production incidents caused by deployments aren’t exotic bugs — they’re small, very human mistakes:

* deploying the wrong branch
* deploying to the wrong environment
* assuming you’re on staging when you’re not

What’s missing is a moment of friction. Just enough to make you stop and think.

---

## Why a shell wrapper?

There are many ways to protect production:

* separate AWS accounts
* IAM role separation
* CI‑only production deploys
* manual approval steps in pipelines

All of these are valid — and you should use them when possible. But they’re also heavy solutions when what you want is a **local, developer‑side guardrail**.

A shell wrapper hits a sweet spot:

* lives entirely in your shell config
* affects only interactive usage
* doesn’t interfere with scripts or CI
* easy to reason about and remove

Most importantly, it protects against the most common failure mode: *typing the wrong thing at the wrong time*.

---

## A note on environment naming

This approach relies on a simple but important convention: **environment names should be semantically meaningful**.

In practice, that means your Elastic Beanstalk environment names should clearly include what they represent, such as:

* `production`, `prod`
* `staging`
* `development`, `dev`

For example:

* `example-api-production`
* `billing-staging`
* `api-dev`

The script does not attempt to infer intent from AWS metadata or tags. Instead, it assumes that if an environment is production, the word `production` appears somewhere in its name. This keeps the logic simple, predictable, and transparent.

If your environments follow this convention, the guard works reliably. If they don’t, that’s usually a sign worth addressing on its own.

---

## What the script does

The wrapper replaces the `eb` command in your shell with a function that:

1. Intercepts `eb deploy`
2. Figures out **which environment** will be deployed to

   * even if flags come first
   * even if no environment is specified (default env)
3. Detects whether the target looks like production
4. Prints a clear warning that includes:

   * the environment name
   * the currently checked‑out git branch
5. Requires explicit confirmation before continuing

If you’re deploying to staging, it stays completely out of the way.

---

## The script

Here’s the full implementation, compatible with both `bash` and `zsh`:

```bash
eb() {
  local args=("$@")
  local env=""
  local branch=""
  local confirm=""
  local lower_env=""

  if [[ "$1" == "deploy" ]]; then
    # Extract environment from args (first non-flag)
    for arg in "${args[@]:1}"; do
      if [[ "$arg" != --* ]]; then
        env="$arg"
        break
      fi
    done

    # Resolve default environment if not provided
    if [[ -z "$env" ]]; then
      env=$(eb status --verbose 2>/dev/null | awk -F': ' '/Environment name/ {print $2}')
    fi

    # Resolve current git branch
    branch=$(git symbolic-ref --quiet --short HEAD 2>/dev/null \
          || git rev-parse --short HEAD 2>/dev/null)
    [[ -z "$branch" ]] && branch="unknown-branch"

    # Normalize env name
    lower_env=$(printf '%s' "$env" | tr '[:upper:]' '[:lower:]')

    if [[ "$lower_env" == *production* ]]; then
      echo "⚠️  You are attempting to deploy to PRODUCTION"
      echo "    Environment : $env"
      echo "    Git branch  : $branch"
      echo
      printf "Type 'yes' to continue: "
      read confirm

      if [[ "$confirm" != "yes" ]]; then
        echo "Deployment aborted."
        return 1
      fi
    fi
  fi

  command eb "${args[@]}"
}
```

Drop this into your `~/.bashrc` or `~/.zshrc`, open a new terminal, and you’re done.

---

## What this achieves in practice

The value of this script isn’t technical — it’s psychological.

Seeing something like:

> You are attempting to deploy to PRODUCTION  
Environment : production  
Git branch: feature/fix-payment-callback

forces a context switch. It breaks autopilot mode. It makes you ask, *“Is this really what I want to do?”*

That moment of hesitation prevents an entire class of mistakes.

It won’t save you from every bad deploy, but it will save you from the easy ones — and those are the most common.

---

## What it deliberately does *not* do

* It doesn’t block CI
* It doesn’t enforce branch policies (though you can add that)
* It doesn’t try to outsmart EB

This is not a security boundary. It’s a seatbelt.

---

## Final thoughts

Elastic Beanstalk gives you a lot of power with very little ceremony. That’s one of its strengths — and one of its sharp edges.

A small, boring shell wrapper like this is often enough to turn a sharp edge into something manageable. It’s easy to forget how many production incidents start with a single, unremarkable command.

If this script makes you stop once and think *“Wait… not like this”*, it has already paid for itself.
