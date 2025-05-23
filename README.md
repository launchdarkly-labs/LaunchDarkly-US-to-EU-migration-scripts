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
```

## Prerequisites

- [Deno](https://deno.land/) installed
  - If you use Homebrew: `brew install deno`
- LaunchDarkly API key with appropriate permissions
  - US account API key for source project with at least Reader access
  - EU account API key for destination project with at least Writer access
- Access to both US and EU LaunchDarkly instances

## Configuration

1. Create a configuration file for your API keys:
   ```bash
   # Copy the example config file
   cp config/api_keys.json.example config/api_keys.json

   # Edit the file with your API keys
   # config/api_keys.json
   {
     "us_api_key": "your_us_api_key_here",
     "eu_api_key": "your_eu_api_key_here"
   }
   ```

   Note: The `config/api_keys.json` file is ignored by git to prevent accidental
   exposure of API keys.

## Quick Start

1. **Download Source Project Data from US Instance**

   Download source project data to `data/source/project/SOURCE_PROJECT_KEY/`.

   ```bash
   # Using deno task (recommended)
   deno task source -p SOURCE_PROJECT_KEY

   # Or using deno run directly
   deno run --allow-net --allow-read --allow-write src/scripts/source.ts -p SOURCE_PROJECT_KEY
   ```

2. **Create Member ID Mapping**

   Automatically create a mapping between US and EU member IDs based on email
   addresses.

   ```bash
   # Using deno task (recommended)
   deno task map-members

   # Or using deno run directly
   deno run --allow-net --allow-read --allow-write src/scripts/map_members.ts
   ```

   This will:
   - Fetch all members from both US and EU instances
   - Match members based on their email addresses
   - Create a mapping file at `data/mappings/maintainer_mapping.json`
   - Show a summary of mapped and unmapped members

3. **(Optional) Estimate Migration Time**

   Before running the migration, you can estimate how long it will take based on
   your project's size and the API rate limits:

   ```bash
   deno task estimate -p SOURCE_PROJECT_KEY -d DESTINATION_PROJECT_KEY
   ```

   This will analyze your source project and provide:
   - Total estimated migration time
   - Resource breakdown (flags, segments, environments)
   - Time breakdown by resource type

   The estimate is based on the following rate limits:
   - Flag operations (create/patch): 5 requests per 10 seconds
   - Segment operations: No rate limit

   Note: The actual migration time may vary due to network conditions and API
   response times.

4. **Migrate Project to EU Instance**

   Creates a new project in the target account or migrates resources into an
   existing project. If you want to preserve flag maintainers, use the `-m` flag
   to automatically map US maintainer IDs to their EU counterparts.

   ```bash
   # Using deno task (recommended)
   # To create a new project and migrate everything:
   deno task migrate -p SOURCE_PROJECT_KEY -d NEW_PROJECT_KEY -m

   # To migrate into an existing project:
   deno task migrate -p SOURCE_PROJECT_KEY -d EXISTING_PROJECT_KEY -m

   # To skip segment migration:
   deno task migrate -p SOURCE_PROJECT_KEY -d DESTINATION_PROJECT_KEY -m -s=false

   # Or using deno run directly:
   deno run --allow-net --allow-read --allow-write src/scripts/migrate.ts -p SOURCE_PROJECT_KEY -d DESTINATION_PROJECT_KEY -m
   ```

   When migrating into an existing project:
   - The script will check if the target project exists
   - If it exists, it will skip project creation
   - It will verify that environments in the source project exist in the target
     project
   - Resources will only be migrated for environments that exist in both
     projects
   - A warning will be shown for any environments that don't exist in the target
     project

For more information about using Deno tasks, see
[Using Deno Tasks](#using-deno-tasks) below.

## Using Deno Tasks

The project includes predefined Deno tasks for easier execution. These tasks are
configured in `deno.json` and include all necessary permissions.

### Available Tasks

```json
{
  "tasks": {
    "source": "deno run --allow-net --allow-read --allow-write src/scripts/source.ts",
    "map-members": "deno run --allow-net --allow-read --allow-write src/scripts/map_members.ts",
    "migrate": "deno run --allow-net --allow-read --allow-write src/scripts/migrate.ts",
    "estimate": "deno run --allow-net --allow-read --allow-write src/scripts/estimate_time.ts"
  }
}
```

### Task Descriptions

1. **source**: Downloads all project data (flags, segments, environments) from
   the source project
   - Requires network access for API calls
   - Requires file system access to save downloaded data
   - Creates directory structure in `data/source/project/`

2. **map-members**: Creates a mapping between US and EU member IDs
   - Fetches members from both US and EU instances
   - Matches members based on email addresses
   - Creates a mapping file in `data/mappings/maintainer_mapping.json`
   - Shows a summary of mapped and unmapped members

3. **migrate**: Creates a new project or migrates into an existing project
   - Requires network access for API calls
   - Requires file system access to read source data
   - Can create a new project or use an existing one
   - Verifies environment compatibility when using existing projects
   - Creates flags, segments, and environments (if creating new project)
   - Can optionally map US maintainer IDs to EU maintainer IDs is the mapping
     was done (step 2) maintainers if mapping was done

4. **estimate**: (Optional) Estimates the time needed for migration
   - Analyzes source project to count resources
   - Tests rate limits in target account
   - Calculates estimated time based on resource counts and rate limits
   - Shows detailed breakdown of the estimate
   - Helps plan migration timing and resource allocation

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

### map_members.ts

- `-o, --output`: (Optional) Output file path, defaults to
  "data/mappings/maintainer_mapping.json"

### migrate.ts

- `-p, --projKeySource`: Source project key
- `-d, --projKeyDest`: Destination project key
- `-m, --assignMaintainerIds`: (Optional) Whether to assign maintainer IDs from
  source project, defaults to false. Requires maintainer mapping to be done
  first.
- `-s, --migrateSegments`: (Optional) Whether to migrate segments, defaults to
  true. Set to false to skip segment migration.

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
