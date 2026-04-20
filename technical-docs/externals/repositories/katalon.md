# katalon (Katalon Studio)

## Overview

**Katalon Studio** is an Eclipse RCP-based test automation IDE that integrates deeply with Katalon TestOps. KS executes automated tests (web, mobile, API, desktop) and posts results to TestOps for analytics, dashboards, and orchestration. TestOps also triggers remote execution on registered KS agents.

## Repository Info

- **URL**: <https://github.com/katalon-studio/katalon>
- **Purpose**: Katalon Studio desktop IDE — Java/Eclipse RCP application; primary test execution runtime that feeds results to TestOps
- **Maintained By**: katalon-studio org
- **Integration Points**: KS uploads test results to TestOps REST API; TestOps manages KS agent registration and remote execution
- **Local Clone Path**: `../katalon/` _(sibling to workspace root)_
- **Default Branch**: `main`
- **Language**: Java 21
- **Build Tool**: Maven + Tycho (OSGi)

## Role of Katalon Studio in Relation to TestOps

| Repo                | Role                                                                               |
| ------------------- | ---------------------------------------------------------------------------------- |
| **katalon**         | Provider — test execution results, agent host                                      |
| **katalon-testops** | Consumer — analytics platform receiving results; orchestration of remote execution |

### What Katalon Studio Provides to TestOps

1. **Test execution results**: After each KS test run, results (JUnit XML + screenshots + logs) are automatically uploaded to the TestOps REST API
2. **Agent runtime**: KS runs as a remote execution agent, polling TestOps for scheduled jobs
3. **Artifact data**: Test artifacts, videos (if enabled), and execution metadata

## Integration Pattern

The integration is **runtime/API-based** — there is no build-time artifact dependency between the two repos.

### Data Flow

```
Katalon Studio (katalon repo)
  └── Test Run completes
      └── KatalonAnalyticsConnector
          └── POST /api/v1/... (TestOps REST API)
              └── Katalon TestOps platform
                  ├── Processes results
                  ├── Updates dashboards
                  └── Schedules next run (via agent poll)
```

### Agent Flow (Remote Execution)

```
TestOps → schedules test job
KS Agent (AgentConfiguration)
  └── Polls TestOps API for pending jobs
      └── Executes scheduled test suite
          └── Uploads results back to TestOps
```

## Integration Workflow

### Build Order

No build dependency — integration is purely runtime:

1. Build Katalon Studio normally (`tools/build-studio-package/build.sh`)
2. Configure TestOps integration in KS project settings (API key)
3. Run tests in KS → results auto-posted to TestOps

### Artifact Flow

- **Direction**: `katalon` → TestOps (runtime, not build time)
- **Method**: HTTPS REST API (`https://testops.katalon.io/api/v1/`)
- **Authentication**: API key (generated in TestOps project settings, stored in KS project config)
- **Payload**: JUnit XML, execution metadata, screenshots, logs

### Development Workflow

To develop/test the KS ↔ TestOps integration:

1. Run a local or staging TestOps instance (or use `testops.katalon.io`)
2. In KS: Project Settings → Katalon TestOps → configure server URL + API key
3. Run a test suite
4. Verify results appear in TestOps dashboard

### Key KS Source Files

| Class | Bundle | Purpose |
| --- | --- | --- |
| `KatalonAnalyticsConfiguration` | `com.kms.katalon.execution` | Stores TestOps server URL and API key configuration |
| `KatalonAnalyticsConnector` | `com.katalon.integration.analytics` | Makes REST API calls to TestOps for result uploads |
| `TestOpsResultUploader` | `com.katalon.integration.analytics` | Collects and uploads JUnit XML results post-run |
| `AgentConfiguration` | `com.kms.katalon.core.agent` | Manages KS agent registration and polling for remote jobs |

### Troubleshooting

| Issue | Cause | Fix |
| --- | --- | --- |
| Results not appearing in TestOps | Integration not enabled in KS | KS Project Settings → Katalon TestOps → enable + set API key |
| Authentication error | Invalid/expired API key | Regenerate API key in TestOps → project → API key management |
| Upload failures | Network/proxy blocking outbound HTTPS | Whitelist `testops.katalon.io` in network/proxy config |
| Agent not executing jobs | Agent not registered | Re-register: KS → Configurations → Agents → register with TestOps |

## KS Build Information

- **Build command**: `tools/build-studio-package/build.sh` (from repo root)
- **Output**: Platform-specific KS products at `source/com.kms.katalon.product/target/products/`
- **Java version**: Java 21 (JavaSE-21)
- **Build time**: ~10–30 minutes

## Documentation

- [Katalon Studio Setup Guide](https://katalon.atlassian.net/wiki/spaces/KSR/pages/3246456882)
- [Katalon TestOps Integration Docs](https://docs.katalon.com/katalon-analytics/docs/overview.html)
