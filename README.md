# LaunchDarkly US to EU Project Migration Tool

A tool for migrating LaunchDarkly projects from US-hosted accounts to EU-hosted
ones. The supported resources include: environments, flags, segments

## Overview

This tool helps you migrate your LaunchDarkly projects from US-hosted accounts
(app.launchdarkly.com) to EU-hosted accounts (app.eu.launchdarkly.com). Features
that are currently supported:

- Project & environment configuration and settings
- Feature flags and their configurations
- Segments and targeting rules
- Maintainer mapping acros the different account instances

## Project Structure

```
root/
├── src/                  # Source code
│   ├── scripts/          # Main migration scripts
│   ├── utils/            # Utility functions
│   └── types/            # TypeScript type definitions
├── config/               # Configuration files
├── data/                 # Data directory
│   ├── source/           # Downloaded source project data
│   └── mappings/         # Mapping files (e.g., maintainer IDs)
└── .vscode/              # VS Code settings
```

## Prerequisites

- [Deno](https://deno.land/) installed
  - If you use Homebrew: `brew install deno`
- LaunchDarkly API key with appropriate permissions
  - US account API key for source project with at least Reader access
  - EU account API key for destination project with at least Writer access
- Access to both US and EU LaunchDarkly instances

## Quick Start

1. **Download Source Project Data from US Instance**

   Download source project data to `data/source/project/SOURCE_PROJECT_KEY/`.

   ```bash
   # Using deno task (recommended)
   deno task start -p SOURCE_PROJECT_KEY -k US_API_KEY -u app.launchdarkly.com

   # Or using deno run directly
   deno run --allow-net --allow-read --allow-write src/scripts/source.ts -p SOURCE_PROJECT_KEY -k US_API_KEY -u app.launchdarkly.com
   ```

2. **Create Member ID Mapping**

   Automatically create a mapping between US and EU member IDs based on email
   addresses.

   ```bash
   # Using deno task (recommended)
   deno task map-members -k US_API_KEY -e EU_API_KEY

   # Or using deno run directly
   deno run --allow-net --allow-read --allow-write src/scripts/map_members.ts -k US_API_KEY -e EU_API_KEY
   ```

   This will:
   - Fetch all members from both US and EU instances
   - Match members based on their email addresses
   - Create a mapping file at `data/mappings/maintainer_mapping.json`
   - Show a summary of mapped and unmapped members

3. **Update Maintainer IDs for EU Instance**

   Member accounts in US and EU instances have different IDs. If you want to
   persist the flag maintainers during the migration, you must create a mapping
   between US and EU maintainer IDs.

   ```bash
   # Run update script to apply the mapping
   deno task update-maintainers -p SOURCE_PROJECT_KEY -m data/mappings/maintainer_mapping.json
   ```

4. **Migrate Project to EU Instance**

   Creates a new project in the target account with, including environments,
   flags, and segments.

   ```bash
   # Using deno task (recommended)
   # If you've created a maintainer mapping and want to assign maintainers:
   deno task migrate -p SOURCE_PROJECT_KEY -d DESTINATION_PROJECT_KEY -k EU_API_KEY -u app.eu.launchdarkly.com -m

   # If you haven't created a maintainer mapping, omit the -m flag:
   deno task migrate -p SOURCE_PROJECT_KEY -d DESTINATION_PROJECT_KEY -k EU_API_KEY -u app.eu.launchdarkly.com
   ```

For more information about using Deno tasks, see
[Using Deno Tasks](#using-deno-tasks) below.

## Using Deno Tasks

The project includes predefined Deno tasks for easier execution. These tasks are
configured in `deno.json` and include all necessary permissions.

### Available Tasks

```json
{
  "tasks": {
    "start": "deno run --allow-net --allow-read --allow-write src/scripts/source.ts",
    "map-members": "deno run --allow-net --allow-read --allow-write src/scripts/map_members.ts",
    "update-maintainers": "deno run --allow-read --allow-write src/scripts/update_maintainers.ts",
    "migrate": "deno run --allow-net --allow-read --allow-write src/scripts/migrate.ts"
  }
}
```

### Task Descriptions

1. **start**: Downloads all project data (flags, segments, environments) from
   the source project
   - Requires network access for API calls
   - Requires file system access to save downloaded data
   - Creates directory structure in `data/source/project/`

2. **map-members**: Creates a mapping between US and EU member IDs
   - Fetches members from both US and EU instances
   - Matches members based on email addresses
   - Creates a mapping file in `data/mappings/maintainer_mapping.json`
   - Shows a summary of mapped and unmapped members

3. **update-maintainers**: Updates maintainer IDs in local flag files
   - Requires file system access to read and write flag files
   - Uses mapping file from `data/mappings/maintainer_mapping.json`
   - Must be run before migration if you want to assign maintainers

4. **migrate**: Creates a new project with all components
   - Requires network access for API calls
   - Requires file system access to read source data
   - Creates new project with all flags, segments, and environments
   - Can optionally assign maintainers if mapping was done

### Task Permissions

Each task includes the necessary permissions:

- `--allow-net`: Required for API calls to LaunchDarkly
- `--allow-read`: Required for reading local files
- `--allow-write`: Required for writing downloaded data

These permissions are automatically included in the task definitions, so you
don't need to specify them manually.

## Command Line Arguments

### source.ts

- `-p, --projKey`: Source project key
- `-k, --apikey`: LaunchDarkly API key
- `-u, --domain`: (Optional) LaunchDarkly domain, defaults to
  "app.launchdarkly.com"

### map_members.ts

- `-k, --us-key`: US instance API key
- `-e, --eu-key`: EU instance API key
- `-o, --output`: (Optional) Output file path, defaults to
  "data/mappings/maintainer_mapping.json"

### update_maintainers.ts

- `-p, --projKey`: Project key
- `-m, --mappingFile`: Path to the maintainer ID mapping file

### migrate.ts

- `-p, --projKeySource`: Source project key
- `-d, --projKeyDest`: Destination project key
- `-k, --apikey`: LaunchDarkly API key
- `-u, --domain`: (Optional) LaunchDarkly domain, defaults to
  "app.launchdarkly.com"
- `-m, --assignMaintainerIds`: (Optional) Whether to assign maintainer IDs from
  source project, defaults to false. Requires maintainer mapping to be done
  first.

## Important Notes for US to EU Migration

- The migration process is one-way and cannot be reversed automatically
- Maintainer IDs are different between US and EU instances and must be mapped
  manually
- Environment names and keys must be unique in the destination project
- The destination project must not exist before migration
- Maximum 20 environments per project
- Monitor for 400 errors in flag configurations
- Consider timezone differences between US and EU instances
- Ensure compliance with EU data protection regulations
- Test the migration in a non-production environment first

## Pre-Migration Checklist

1. **Data Assessment**
   - What flags and segments need to be migrated?
   - Can you use this as an opportunity to clean up unused resources?
   - Experiments, guarded or randomised (percentage-based) rollouts can't be
     migrated while maintaining consistent variation bucketing. These types of
     releases should ideally be completed before the migration takes place.

2. **Access and Permissions**
   - Do you have API access to both US and EU instances?
   - Have you created the necessary API keys?
   - Do you have sufficient access to create projects in the EU instance?

3. **Maintainer Mapping**
   - Have all the members who are set as maintainers in the US account been
     added to the EU instance?
   - Have you created the maintainer ID mapping file?

4. **Timing and Execution**
   - When is the best time to perform the migration?
   - How will you handle ongoing changes during migration?
   - Do you need to maintain consistent state between two two account instances
     after the migration? If so, for how long?

## Known Issues

- TypeScript types are loose due to API client limitations
- Importing LaunchDarkly API TypeScript types causes import errors
- Rate limiting may affect flag configuration updates
- Some custom integrations may need to be reconfigured for EU endpoints

## Support

For issues related to the migration tool, please create an issue in this
repository. For LaunchDarkly-specific questions, contact LaunchDarkly support.
