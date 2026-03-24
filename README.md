# AdsMCP

**The ad monetization MCP server for AI applications.**

AdsMCP is Adgentek's hosted Model Context Protocol server that enables AI agents and conversational interfaces to serve contextually relevant ads automatically — based on conversation context, user intent, and placement opportunity.

Built on a full ad server stack. Bring live demand from Adgentek, bring your own direct-sold campaigns, or both. No ad tech expertise required.

---

## Before You Start

**You need an Adgentek publisher account, an API key, and at least one active AI Surface before ads will serve.**

[**Create your publisher account →**](https://app.adgentek.ai/publishers/auth)

### Setup checklist

1. **Create your account** at [app.adgentek.ai/publishers/auth](https://app.adgentek.ai/publishers/auth)
2. **Create an AI Surface** — in the publisher dashboard, go to **Inventory → Surfaces** and add at least one surface (e.g. `conversational`, `search`, `voice`). A surface defines where ads can appear and sets your floor eCPM.
3. **Generate an API key** — in the publisher dashboard, go to **Settings → API Keys** and create a key (prefixed `adgt_`). This key authenticates all MCP requests.

Your account is considered **Active** once at least one surface is enabled.

---

## Connect

```json
{
  "mcpServers": {
    "adgentek-ads": {
      "url": "https://mcp.adgentek.ai/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY_HERE"
      }
    }
  }
}
```

This is the only configuration needed. All MCP-compatible clients support this URL + Headers pattern — no local packages or environment variables required.

---

## Platform-Specific Configuration

The connection config is identical across platforms. Only the file location and wrapper key differ.

<details>
<summary><strong>Claude Desktop</strong></summary>

```json
// ~/Library/Application Support/Claude/claude_desktop_config.json  (macOS)
// %APPDATA%\Claude\claude_desktop_config.json  (Windows)
{
  "mcpServers": {
    "adgentek-ads": {
      "url": "https://mcp.adgentek.ai/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY_HERE"
      }
    }
  }
}
```
</details>

<details>
<summary><strong>Claude Code</strong></summary>

```json
// ~/.claude/settings.json
{
  "mcpServers": {
    "adgentek-ads": {
      "url": "https://mcp.adgentek.ai/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY_HERE"
      }
    }
  }
}
```
</details>

<details>
<summary><strong>Cursor</strong></summary>

```json
// .cursor/mcp.json
{
  "mcpServers": {
    "adgentek-ads": {
      "url": "https://mcp.adgentek.ai/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY_HERE"
      }
    }
  }
}
```
</details>

<details>
<summary><strong>VS Code (Copilot MCP)</strong></summary>

```json
// .vscode/mcp.json
{
  "servers": {
    "adgentek-ads": {
      "url": "https://mcp.adgentek.ai/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY_HERE"
      }
    }
  }
}
```
</details>

<details>
<summary><strong>OpenAI Codex</strong></summary>

```json
{
  "mcpServers": {
    "adgentek-ads": {
      "url": "https://mcp.adgentek.ai/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY_HERE"
      }
    }
  }
}
```
</details>

---

## Available Tools

### `ads.serve`

Serves a contextual ad based on the current conversation. Returns the highest-yield ad available for the given context, or `null` if no ad matched.

**Required parameters:** `conversationId`, `userKey`, `surface`

**Optional parameters:** `messages`, `intent`, `sentiment`, `depth`, `language`, `geo`, `preferHtml`, `extras`

```javascript
const result = await callTool('ads.serve', {
  conversationId: 'conv_abc123',
  userKey:        'user_456',
  surface:        'conversational',
  messages: [
    { role: "user",      content: "What CRM works best for a small sales team?" },
    { role: "assistant", content: "For small teams, a few options stand out..." },
    { role: "user",      content: "Which ones integrate with Google Workspace?" }
  ],
  intent:    'research',
  sentiment: 65,
  depth:     3,
  language:  'en',
  geo:       'US'
});

if (result.ad) {
  displayAd(result.ad.render.markdown);
  // Impressions: embed ad.impression_url as a 1x1 pixel in the browser
  // Clicks: use ad.click_url as the CTA link href
}
```

**Surface types:** `conversational`, `search`, `voice`, `creative`, `web`

**Intent values:** `purchase`, `research`, `support`, `informational`

---

### `ads.prefetch`

Batch-request up to **10** ad contexts in a single call. Each item in the batch uses the same parameters as `ads.serve`.

**Required:** `batch` (array, max 10 items — each requiring `conversationId`, `userKey`, `surface`)

```javascript
const results = await callTool('ads.prefetch', {
  batch: [
    { conversationId: 'conv_1', userKey: 'user_456', surface: 'conversational', messages: [...] },
    { conversationId: 'conv_2', userKey: 'user_456', surface: 'search',         messages: [...] }
  ]
});

// results.ads            — array of ad objects (null where no ad matched)
// results.prefetched_count — number of ads returned
// results.truncated       — true if your batch exceeded the 10-item limit
```

---

### `ads.config`

Returns your publisher configuration. **Takes no parameters** — your identity is derived from the API key in the Authorization header.

```javascript
const config = await callTool('ads.config', {});
// Returns: { publisher_id, org_id, ...upstream config }
```

---

### `ads.diagnostics`

Runs the full ad auction with debug output enabled. Uses the **same parameters** as `ads.serve` (same required fields: `conversationId`, `userKey`, `surface`).

Returns the raw auction debug data including eligibility checks and match details.

```javascript
const diag = await callTool('ads.diagnostics', {
  conversationId: 'debug_abc123',
  userKey:        'user_456',
  surface:        'conversational',
  messages: [
    { role: "user", content: "Tell me about electric vehicles" }
  ]
});

console.log(diag); // Full debug output from the auction engine
```

---

## Ad Response Format

When `ads.serve` returns a matched ad:

```json
{
  "ad": {
    "id": "ad_abc123",
    "title": "Product Title",
    "content": "Description text",
    "cta": "Learn More",
    "click_url": "https://api.adgentek.ai/.../click?...",
    "impression_url": "https://api.adgentek.ai/.../pixel.gif?...",
    "render": {
      "markdown": "## Title\n\nContent\n\n[CTA](click_url)",
      "plain": "Title - Content - CTA",
      "html": "<div>...</div>"
    }
  },
  "targeting_scores": { ... }
}
```

When no ad matched:

```json
{
  "ad": null,
  "reason": "no_eligible_ads",
  "targeting_scores": { ... }
}
```

---

## Tracking

Impression and click tracking is handled via the URLs provided in every ad response. These work in any rendering context — browser, in-app webview, or embedded UI.

- **Impressions**: Load `ad.impression_url` as a 1×1 pixel image when the ad is displayed
- **Clicks**: Use `ad.click_url` as the CTA link href (signed redirect to advertiser destination)

```html
<!-- Impression pixel -->
<img src="{ad.impression_url}" width="1" height="1" style="display:none" alt="" />

<!-- Click link -->
<a href="{ad.click_url}" target="_blank" rel="noopener noreferrer">{ad.cta}</a>
```

Do not modify or bypass these URLs — they are signed and time-limited.

---

## MCP Resources

The server exposes read-only documentation as MCP resources:

| URI | Description |
|-----|-------------|
| `resource://adgentek/creative-guide` | Creative format guidelines and JS creative structure |
| `resource://adgentek/integration-guide` | Platform setup and response format reference |
| `resource://adgentek/tracking-spec` | Browser tracking requirements and fraud prevention |

---

## Full Ad Server — Bring Your Own Ads

AdsMCP is not a demand source with an MCP interface bolted on. It is a full ad server.

Publishers can:

- **Use Adgentek's live demand** — we have established relationships across programmatic, performance, and direct demand channels, with more being added continuously
- **Bring your own campaigns** — upload direct-sold advertiser deals and serve them through the same `ads.serve` call
- **Layer in additional ad sources** — connect your own demand partners alongside Adgentek's, with automatic yield optimization across all of them

Your inventory. Your demand relationships. One unified server.

---

## Privacy Architecture

User conversations are private. Raw conversation content is never shared with advertisers, demand partners, or third parties.

AdsMCP transmits only derived signals to ad buyers — the minimum information required to match a relevant ad:

| What advertisers receive | What stays in your app |
|---|---|
| IAB content category | Full conversation content |
| Intent classification | User messages |
| Keyword themes | Session context |

---

## Go Live

1. **[Create your publisher account](https://app.adgentek.ai/publishers/auth)** and generate an API key
2. **Create at least one AI Surface** in the publisher dashboard (Inventory → Surfaces)
3. **Add the MCP config** to your AI application using the connection snippet above
4. **Contact [hello@adgentek.ai](mailto:hello@adgentek.ai)** to verify your integration and activate live demand

---

## About Adgentek

AdsMCP is built and operated by [Adgentek](https://adgentek.ai).

Adgentek exists because AI apps are a fundamentally different kind of publisher — and they deserve ad infrastructure built for them, not retrofitted from the display web. Conversational interfaces have richer intent signals, longer engagement sessions, and none of the legacy constraints that shaped how traditional ad tech was built.

Adgentek builds the ad monetization layer that is native to this environment: context-aware, privacy-safe, MCP-native, and designed to serve AI apps of any size — from a single-developer GPT to a multi-surface conversational platform. Our demand relationships span programmatic, direct, and performance channels, and publishers can layer in their own ad sources alongside ours.

If you are building an AI app and want ad revenue without building ad infrastructure, AdsMCP is what you are looking for.

[hello@adgentek.ai](mailto:hello@adgentek.ai) — [adgentek.ai](https://adgentek.ai)
