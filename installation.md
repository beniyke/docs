# Installation

Anchor provides two primary ways to set up your application: **Managed Mode** (using Composer) and **Standalone Mode** (portable/no-dependency).

Both methods start with the **Anchor Skeleton**, which serves as a clean starting point for your project.

## Requirements

Ensure your system meets the following requirements:

- **PHP**: >= 8.2
- **Database**: SQLite (default), MySQL 8.0+, or PostgreSQL 15+.
- **Extensions**: PDO, **pdo_sqlite** (required for default setup), pdo_mysql, pdo_pgsql, Mbstring, OpenSSL, Ctype, JSON, cURL, BCMath, ZipArchive, **SimpleXML**, **Fileinfo**, **Tokenizer**, **Intl**.

> Anchor follows a **SQLite first** approach for rapid development and zero-config deployment. Ensure `pdo_sqlite` is enabled for the best initial experience.

- **Composer**: Required for Managed Mode.

## Managed Mode (Recommended)

This is the standard installation method for modern development. It uses Composer to manage the framework as a dependency.

### Create Project

Run the following command to create a new project from the official skeleton:

```bash
composer create-project beniyke/anchor-skeleton my-app
```

### Initialize

Navigate to your project directory and run the `dock` CLI tool:

```bash
cd my-app
php dock
```

Choosing the "Managed" option will install the `beniyke/framework` package and set up your vendor directory.

## Standalone Mode (Portable)

Perfect for environments with restricted internet or for rapid prototyping without Composer.

### Obtain Skeleton

You can either download the zip or clone the repository using Git:

#### Option A: Download Zip

Download the latest [Anchor Skeleton Zip](https://github.com/beniyke/anchor-skeleton/archive/refs/heads/main.zip) and extract it to your project folder.

#### Option B: Git Clone

```bash
git clone https://github.com/beniyke/anchor-skeleton.git my-app
cd my-app
```

### Hydrate

Run the `dock` tool to provision the framework core:

```bash
php dock
```

Select the **Standalone** option. The tool will download and extract the framework core directly into your project's `System/` directory.

## Post-Installation Setup

Once the framework is hydrated (either via Composer or Standalone), follow these steps to finalize your environment:

### Environment Configuration

Copy the example environment file:

```bash
cp .env.example .env
```

Open `.env` and configure your database settings.

### Run Core Migrations

Initialize your database schema:

```bash
php dock migration:run
```

### Install Essential Packages

Install core system packages to enable full functionality:

```bash
# Background job processing
php dock package:install Queue --system

# Notifications and Alerts
php dock package:install Notify --system

# Developer Toolbar & Debugger
php dock package:install Debugger --system
```

### Start the Engine

Launch your development server:

```bash
php dock dev
```

Your application is now ready at `http://localhost:1010` (or your configured `APP_URL`).

## Keeping Up to Date

Anchor makes it easy to keep your framework core updated, regardless of your installation mode.

### Unified Update Command

Run the following command to intelligently update the framework:

```bash
php dock anchor:update
```

- **In Managed Mode**: This executes `composer update beniyke/framework`.
- **In Standalone Mode**: This downloads and applies the latest core artifacts directly from GitHub.

#### Options

- `-f, --force`: Skip confirmation and overwrite files (Standalone only).
- `-t, --tag[=TAG]`: Pull a specific version tag instead of the latest (Standalone only).

## Next Steps

- Learn about the [Directory Structure](directory-structure.md)
- Explore the [Architecture](architecture.md)
- Configure your [Routing](routing.md)
