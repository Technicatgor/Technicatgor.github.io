---
layout: post
title: UV Python Package Manager
date: 2025-08-13 16:00:00 +800
categories: [Dev]
tags: [Python]
image:
  path: /assets/img/headers/uv-cover.png
  lqip: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA8gAAAH4CAIAAAAZ1VPRAALJIklEQVR4Aeyah5IbuxVE0aQ3h/eUv8L//1MOm/NumxaKp9TVZInO4e1d1QgDXNyMHgyG+vj59+Od
---

# UV in Python

## What is UV?

UV is a Rust-based Python tool that manages virtual environments, dependencies, running scripts, building packages, and more, all with one fast command-line interface. It automates creation of virtual environments and installs dependencies quickly using a lockfile (`uv.lock`) for reproducibility.

## Installation

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

> reference link:  
> <https://docs.astral.sh/uv/getting-started/installation/>

### Step 1: Creating a New Python Project

Create a new project directory and navigate into it:

```bash
mkdir my_uv_project cd my_uv_project
```

Initialize your project and manage dependencies in one tool using UV commands. UV uses a `pyproject.toml` file to list dependencies.

---

### Step 2: Add Dependencies

Add dependencies through UV easily. For example, to add requests:

```bash
uv add requests
```

This updates `pyproject.toml` and creates/updates `uv.lock` with exact versions for reproducibility.

---

### Step 3: Activating the Virtual Environment

UV automatically creates and manages the virtual environment, so no manual activation is required. When you run your Python scripts via UV, it will use the correct environment.

---

### Step 4: Running Scripts

Run your project scripts directly with UV. Suppose you have a `main.py` file:

```bash
uv run main.py
```

UV will automatically create a virtual environment if needed, install dependencies, and run the script.

---

### Step 5: Synchronizing Environments

If you clone a project or share it, simply synchronize the environment:

```bash
uv sync
```

This installs exact dependencies from `uv.lock` to recreate the environment consistently on any machine.

---

### Example `main.py` (Simple CLI App)

Here's a sample Python program using requests for demonstration:

```python
import requests

def get_breeds_info():
  response = requests.get("https://api.thecatapi.com/v1/breeds")
  response.raise_for_status()
  return response.json()

def main():
  breeds = get_breeds_info()
  print(f"Number of cat breeds: {len(breeds)}")

if __name__ == "__main__":
  main()
```

Run this with:

```bash
uv run main.py
```

## **Managing dependencies**

1. **To migrate existing dependencies from a** `requirements.txt` **file:**

```bash
uv add -r requirements.txt
```

2. **Removing Dependencies**

Remove a package with:

```bash
uv remove requests
```

This updates `pyproject.toml` and `uv.lock`.

The package will be uninstalled from the environment when you run `uv sync` or the next command that installs dependencies.

3. **Upgrading Dependencies**

Upgrade a specific package:

```bash
uv lock --upgrade-package requests
```

This upgrades the package to the latest compatible version according to your version constraints.

Run `uv sync` to install the updated packages.

To upgrade all packages according to constraints:

```bash
uv lock --upgrade uv sync
```

4. **Installing & Syncing Dependencies**

To install all declared dependencies and create or update your virtual environment:

```bash
uv sync
```

Downloads and installs all dependencies exactly as specified in `uv.lock`.

Ensures your local environment matches the project files.

You can also run scripts with environment setup automatically:

```bash
uv run main.py
```

## How uv Manages Dependencies Internally

- `pyproject.toml` — Declares your project dependencies with version constraints.
- `uv.lock` — Records exact versions for reproducibility and manages both direct and transitive dependencies.
- uv commands automatically keep these files updated to ensure consistent environments across machines.

## Summary of Key Commands

| **Action**           | **Command**                       | **Description**                                         |
| -------------------- | --------------------------------- | ------------------------------------------------------- |
| Add a dependency     | `uv add <package>`                | Adds, installs, and updates dependency files            |
| Remove a dependency  | `uv remove <package>`             | Removes package and updates dependency files            |
| Upgrade a package    | `uv lock --upgrade-package <pkg>` | Upgrades single package to latest compatible version    |
| Upgrade all          | `uv lock --upgrade` + `uv sync`   | Upgrades all dependencies                               |
| Install dependencies | `uv sync`                         | Installs all dependencies & creates virtual environment |
| Run with env         | `uv run <script>`                 | Runs script with environment automatically set up       |
