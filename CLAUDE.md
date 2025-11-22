# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Python-based MCP (Model Context Protocol) server that provides programmatic access to the Facebook/Meta Ads API. It exposes 22 tools for managing ad accounts, campaigns, ad sets, ads, creatives, insights, activity history, and targeting search.

## Development Commands

### Setup and Dependencies

```bash
# Create and activate virtual environment
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Running the Server

```bash
# Run the server locally (for debugging)
python server.py --fb-token YOUR_META_ACCESS_TOKEN
```

The server requires a Facebook User Access Token with appropriate permissions (e.g., `ads_read`). The token must be passed via the `--fb-token` command line argument.

### Docker

```bash
# Build Docker image
docker build -t facebook-ads-mcp-server .

# Run container (note: you'll need to modify CMD to pass real token)
docker run facebook-ads-mcp-server
```

## Architecture

### Core Components

**server.py** - Single-file MCP server implementation (~2300 lines)
- Uses `FastMCP` from the `mcp.server.fastmcp` library
- All tools are defined as decorated functions with `@mcp.tool()`
- Server communicates via stdio when integrated with MCP clients

### Key Design Patterns

1. **Token Management**: Facebook access token is stored globally after first retrieval from `--fb-token` command line argument via `_get_fb_access_token()`

2. **API Communication Helpers**:
   - `_make_graph_api_call(url, params)` - Handles all GET requests to Facebook Graph API
   - `_fetch_node(node_id, **kwargs)` - Fetches single objects by ID
   - `_fetch_edge(parent_id, edge_name, **kwargs)` - Fetches collections/edges
   - `_prepare_params(base_params, **kwargs)` - Handles parameter preparation and JSON encoding for complex fields

3. **Insights Parameters**: `_build_insights_params()` centralizes the complex parameter handling for insight queries (time ranges, breakdowns, filtering, etc.)

### API Constants

- **API Version**: `v22.0` (defined in `FB_API_VERSION`)
- **Base URL**: `https://graph.facebook.com/v22.0` (in `FB_GRAPH_URL`)
- **Default Fields**: Common ad account fields are defined in `DEFAULT_AD_ACCOUNT_FIELDS`

### Tool Categories

The 22 MCP tools are organized into 6 categories:

1. **Account & Object Read** (7 tools): `list_ad_accounts`, `get_details_of_ad_account`, `get_campaign_by_id`, `get_adset_by_id`, `get_ad_by_id`, `get_ad_creative_by_id`, `get_adsets_by_ids`

2. **Fetching Collections** (6 tools): `get_campaigns_by_adaccount`, `get_adsets_by_adaccount`, `get_ads_by_adaccount`, `get_adsets_by_campaign`, `get_ads_by_campaign`, `get_ads_by_adset`, `get_ad_creatives_by_ad_id`

3. **Insights & Performance** (5 tools): `get_adaccount_insights`, `get_campaign_insights`, `get_adset_insights`, `get_ad_insights`, `fetch_pagination_url`

4. **Activity/Change History** (2 tools): `get_activities_by_adaccount`, `get_activities_by_adset`

5. **Targeting Search** (1 tool): `targeting_search` - Searches for available targeting options (interests, behaviors, demographics, locations, employers, job titles, etc.)

6. **Utility** (1 tool): `fetch_pagination_url` - Handles pagination URLs returned by Facebook API

### Common Parameters Across Tools

Many tools support these optional parameters:
- `fields`: List of fields to retrieve (converted to comma-separated string)
- `filtering`: Complex filter objects (JSON encoded)
- `limit`: Number of results to return
- `time_range`/`time_ranges`: Date filtering for insights (JSON encoded)
- `level`: Aggregation level for insights
- `breakdowns`: Dimension breakdowns (list converted to comma-separated)
- `action_attribution_windows`: Attribution windows for actions

### Error Handling

API errors are caught in `_make_graph_api_call()` via `requests.exceptions.RequestException`. Errors are logged to stdout and re-raised.

## MCP Client Configuration

To use this server with Claude Desktop or Cursor, add to MCP configuration:

```json
{
  "mcpServers": {
    "fb-ads-mcp-server": {
      "command": "python",
      "args": [
        "/path/to/server.py",
        "--fb-token",
        "YOUR_META_ACCESS_TOKEN"
      ]
    }
  }
}
```

For virtual environment usage, specify the venv Python executable as the command.

## Dependencies

- **mcp** (>=1.6.0): MCP server framework with FastMCP
- **requests** (>=2.32.3): HTTP library for Facebook Graph API calls
- **Python 3.10+**: Required runtime version

## Important Notes

- All Facebook API IDs should be prefixed with `act_` for ad accounts (e.g., `act_123456789`)
- The server uses Facebook Graph API v22.0 - version updates may require testing
- Access tokens expire and need to be regenerated periodically
- Most list/collection endpoints support pagination via cursor-based pagination URLs
- Insights queries can be resource-intensive; use appropriate time ranges and limits

## Targeting Search Tool

The `targeting_search` tool (server.py:2294) enables programmatic discovery of Facebook ad targeting options:

**Supported Search Types:**
- `adinterest` - Interest-based targeting (hobbies, pages liked)
- `adinterestsuggestion` - Interest suggestions for targeting
- `adinterestvalid` - Validate interest targeting IDs
- `adgeolocation` - Geographic locations (cities, regions, countries)
- `adgeolocationmeta` - Geographic metadata information
- `adradiussuggestion` - Suggestions for radius targeting
- `adTargetingCategory` - Broad categories requiring `class_` parameter
- `adworkemployer` - Employers/companies for workplace targeting
- `adeducationschool` - Educational institutions
- `adeducationmajor` - College majors/fields of study
- `adworkposition` - Job titles for professional targeting
- `adlocale` - Languages/locales

**Key Parameters:**
- `q` (required): Search query string
- `type` (required): Targeting type from list above
- `class_` (optional): Required for `adTargetingCategory` to specify subcategory:
  - `interests` - Interest-based targeting
  - `behaviors` - Behavior-based targeting
  - `demographics` - Demographic targeting
  - `family_statuses` - Family status targeting
  - `industries` - Industry targeting
  - `life_events` - Life event targeting
  - `income` - Income-based targeting
  - `user_device` - Device targeting
  - `user_os` - Operating system targeting
- `location_types` (optional): Filter geographic searches by type (JSON array)
- `locale` (optional): Language for results (e.g., "en_US", "es_ES")
- `country_code` (optional): Two-letter country code for localized results
- `limit` (optional): Number of results (default 25)

**Type-Specific Parameters:**
- For `adradiussuggestion`:
  - `latitude` (required): Latitude coordinate (-90 to 90)
  - `longitude` (required): Longitude coordinate (-180 to 180)
  - `distance_unit` (optional): 'mile' or 'kilometer' (default: 'kilometer')
- For `adinterestsuggestion` or `adinterestvalid`:
  - `interest_list` (required): List of interest names (replaces q parameter, JSON array)
- For `targetingoptionstatus`:
  - `targeting_option_list` (required): List of targeting option IDs to check (JSON array)
- For `adgeolocation`:
  - `region_id` (optional): Numeric region ID to narrow searches within a specific region

**Important Notes:**
- For `adTargetingCategory` type, the `q` parameter is ignored by Facebook API
- When using `class_='demographics'`, all available demographic targeting options are returned regardless of query
- Geographic searches (`adgeolocation`, `adgeolocationmeta`) return a `key` field (string) instead of `id`
- Interest, behavior, and demographic searches return a numeric `id` field
- Geographic results include additional fields: `country_code`, `supports_region`, `supports_city`

**Response includes:** Target IDs/keys for use in ad specs, audience size estimates, category paths, and descriptions.