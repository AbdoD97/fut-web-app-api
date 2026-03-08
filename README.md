# EA FC 26 Ultimate Team — REST API Reference

> **Unofficial.** EA Sports does not publish a public API. This reference is reverse-engineered
> from network traffic and community research. Making excessive or automated requests risks
> account suspension. Use at your own risk.

Markers used throughout:
- **[Observed]** — Directly confirmed via network interception on EA FC 26
- **[Community]** — Documented by the reverse-engineering community; may vary across FC versions

---

## Table of Contents

- [Base URL & Versioning](#base-url--versioning)
- [Authentication](#authentication)
- [Transfer Market](#transfer-market)
- [Trade Pile](#trade-pile)
- [Auctions & Listing](#auctions--listing)
- [Club](#club)
- [Item Operations](#item-operations)
- [Player / Concept Search](#player--concept-search)
- [Account & Session](#account--session)
- [Packs & SBC](#packs--sbc)
- [Data Structures](#data-structures)
- [Enums & Constants](#enums--constants)
- [maskedDefId Formula](#maskeddefid-formula)
- [Price Bands](#price-bands)
- [EA Tax](#ea-tax)
- [Rate Limits & Safety](#rate-limits--safety)
- [Third-Party Resources](#third-party-resources)

---

## Base URL & Versioning

**[Observed]**

```
https://utas.mob.v5.prd.futc-ext.gcp.ea.com:443/ut/game/fc26/
```

All endpoints below are relative to this base. The version segment (`fc26`) increments each year.

Older / alternative base URLs seen in community projects:

```
https://utas.mobapp.fut.ea.com/ut/game/fc26/
https://utas.mob.v4.fut.ea.com/ut/game/fc26/
```

---

## Authentication

**[Community]**

EA FC 26 uses a session-based auth system. You must obtain a session ID and phishing token
before calling any game endpoints.

### Required Headers (all game endpoints)

| Header | Description |
|--------|-------------|
| `X-UT-SID` | Session ID — UUID obtained after login |
| `X-UT-PHISHING-TOKEN` | Numeric security token issued at session start |
| `Content-Type` | `application/json` (for POST/PUT requests) |
| `Accept` | `application/json` |

### Auth Endpoint

```
POST https://utas.mobapp.fut.ea.com/ut/auth
```

**Request body:**
```json
{
  "email": "your@email.com",
  "password": "yourpassword",
  "isReadOnly": false,
  "sku": "FUT26WEB",
  "clientVersion": 1,
  "locale": "en-GB",
  "method": "authapp"
}
```

**Response:**
```json
{
  "sid": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "phishingToken": "1234567890",
  "protocol": "https",
  "ipPort": "...",
  "persona": "...",
  "nucleus_id": "..."
}
```

> EA accounts with 2FA require an additional code verification step before the session is
> fully active. Sessions expire after ~60 minutes of inactivity.

---

## Transfer Market

### Search

**[Observed]** — URL pattern confirmed from network interception.

```
GET /transfermarket
```

**Query parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `num` | int | Yes | Results per page. Max **21**. |
| `start` | int | Yes | Pagination offset (0, 21, 42, …) |
| `type` | string | Yes | Item type. Use `"player"` for player cards. |
| `maskedDefId` | int | No* | 20-bit masked player identity (see [maskedDefId Formula](#maskeddefid-formula)) |
| `definitionId` | int | No* | Full definition ID (alternative to maskedDefId) |
| `rarityIds` | int | No | Rarity/card type filter (e.g. `55` for special cards) |
| `minb` | int | No | Minimum Buy Now price filter |
| `maxb` | int | No | Maximum Buy Now price filter |
| `pos` | string | No | Position filter (e.g. `"ST"`, `"CB"`) |
| `lev` | string | No | Level filter: `"gold"`, `"silver"`, `"bronze"` |

*One of `maskedDefId` or `definitionId` is required for player searches.

**Example:**
```
GET /transfermarket?num=21&start=0&type=player&maskedDefId=231747&rarityIds=55&maxb=1000000
```

**Response:**
```json
{
  "auctionInfo": [
    {
      "tradeId": 987654321,
      "itemData": {
        "id": 123456789,
        "resourceId": 117672259,
        "assetId": 117672259,
        "rating": 91,
        "itemType": "player",
        "cardsubtypeid": 55
      },
      "tradeState": "active",
      "buyNowPrice": 850000,
      "currentBid": 0,
      "startingBid": 800000,
      "expires": 3580
    }
  ],
  "credits": 1234567,
  "currencies": [],
  "bidTokens": {}
}
```

> **Note:** Via the internal JS layer, items arrive pre-shaped with `_auction`,
> `_staticData`, etc. wrappers. The raw HTTP response uses `auctionInfo[]` + `itemData`
> as shown above. **[Observed]** that `_auction.buyNowPrice` in the JS layer maps to
> `auctionInfo[].buyNowPrice` in the raw response.

---

## Trade Pile

### Get trade pile contents

**[Community]**

```
GET /tradepile
```

**Response:** Same shape as transfer market search (`auctionInfo[]` array).

Trade states in `auctionInfo[].tradeState`:

| Value | Meaning |
|-------|---------|
| `"inactive"` | In pile, not listed |
| `"active"` | Currently listed |
| `"expired"` | Listing expired, unsold |
| `"closed"` | Sold |

---

### Delete item from trade pile

**[Community]**

```
DELETE /tradepile/{tradeId}
```

---

### Clear trade pile (remove all expired/inactive)

**[Community]**

```
DELETE /tradepile
```

---

### Re-list expired items

**[Community]**

```
PUT /auctionhouse/relist
```

Re-lists all expired items at their original prices.

---

### Check trade / auction status

**[Community]**

```
GET /trade/status?tradeIds={tradeId1},{tradeId2}
```

Returns current state and bid info for given trade IDs.

---

## Auctions & Listing

### List an item for sale

**[Observed via JS layer]** — confirmed via `services.Item.itemDao.listItem()`.

```
POST /auctionhouse
```

**Request body:**
```json
{
  "itemData": {
    "id": 123456789
  },
  "startingBid": 800000,
  "buyNowPrice": 850000,
  "duration": 3600
}
```

**Parameters:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | int | Item instance ID |
| `startingBid` | int | Starting bid — must conform to [price bands](#price-bands) |
| `buyNowPrice` | int | Buy Now price — must conform to [price bands](#price-bands) |
| `duration` | int | Duration in **seconds**: `3600` (1h), `10800` (3h), `21600` (6h), `43200` (12h), `86400` (24h) |

**Response:**
```json
{
  "id": 987654321,
  "tradeState": "active",
  "buyNowPrice": 850000,
  "startingBid": 800000,
  "currentBid": 0,
  "expires": 3600
}
```

---

### Bid on an auction

**[Community]**

```
PUT /trade/{tradeId}/bid
```

**Request body:**
```json
{
  "bid": 850000
}
```

---

## Club

### Get club items

**[Observed via JS layer]** — confirmed via `services.Club.search()`.

```
GET /club
```

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | `"player"`, `"staff"`, `"consumable"` |
| `defId` | int | Filter by definition ID |
| `start` | int | Pagination offset |
| `count` | int | Results per page |

**Response:**
```json
{
  "itemData": [
    {
      "id": 123456789,
      "assetId": 117672259,
      "resourceId": 117672259,
      "rating": 91,
      "untradeable": false,
      "loans": 0,
      "discardValue": 659
    }
  ],
  "itemCount": 1
}
```

---

### Send item to club

**[Community]**

```
PUT /item
```

**Request body:**
```json
{
  "itemData": [
    { "id": 123456789, "pile": "club" }
  ]
}
```

---

## Item Operations

### Move item to trade pile (or other pile)

**[Observed]** — confirmed via `services.Item.move(items, ItemPile.TRANSFER)`.

```
PUT /item
```

**Request body:**
```json
{
  "itemData": [
    { "id": 123456789, "pile": "trade" }
  ]
}
```

`pile` values: `"trade"` (trade pile), `"club"`, `"purchased"`.

> **Critical:** You must pass real item objects fetched from a prior `/club` or `/tradepile`
> response — the server validates item ownership. Providing an arbitrary `id` will fail.

---

### Quick sell / discard

**[Observed]** — confirmed via `services.Item.discard(item)`. Coins received = `item.discardValue`.

```
DELETE /item/{itemId}
```

Only valid for items in the club or trade pile that are not currently listed.

---

## Player / Concept Search

### Search player database (concept items)

**[Observed]** — confirmed via `services.Item.searchConceptItems()`.

```
GET /searchitem
```

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | `"player"` |
| `min-ovr` | int | Minimum overall rating |
| `max-ovr` | int | Maximum overall rating |
| `count` | int | Results per page (up to 50) |
| `start` | int | Pagination offset |

Useful for resolving a player name → `definitionId` without needing to search the market.

> Concept items **do not** have a `maskedDefId` property in the response — derive it via
> `definitionId % 1048576`.

---

### Get static player data by maskedDefId

**[Observed]** — available via `repositories.Item.getStaticDataByDefId(maskedDefId)` in the
JS layer. Returns cached data: `{ name, lastName, commonName, firstName, rating }`.
No direct HTTP endpoint confirmed; data is bundled in the web app's asset files.

---

## Account & Session

### Keep session alive

**[Community]**

```
GET /user/accountinfo
```

Call periodically (every ~5 minutes) to prevent session expiry. Also returns account metadata.

---

### Get user info

**[Community]**

```
GET /usermassinfo
```

Returns persona, club info, credits, active squad, etc.

---

## Packs & SBC

### Buy a pack

**[Community]**

```
POST /store/bundle/purchase
```

**Request body:**
```json
{ "packId": 1234, "useCoins": true }
```

---

### Get SBC sets

**[Community]**

```
GET /sbs/sets
```

---

## Data Structures

### Item Object (JS layer representation)

This is how items are shaped after passing through EA's internal JS model layer:

```javascript
{
  id:               123456789,      // unique item instance ID
  definitionId:     117672259,      // full card definition ID
  resourceId:       117672259,      // typically == definitionId
  rareflag:         55,             // rarity / card type ID
  preferredPosition: "ST",

  _rating:          91,             // overall rating
  _staticData: {
    name:           "Kylian Mbappé",
    firstName:      "Kylian",
    lastName:       "Mbappé",
    commonName:     "Mbappé",
    rating:         91
  },
  _auction: {
    tradeId:        987654321,
    tradeState:     "active",       // inactive | active | expired | closed
    buyNowPrice:    850000,
    startingBid:    800000,
    currentBid:     0,
    expires:        3580            // seconds remaining
  },
  _itemPriceLimits: {
    minimum:        150,
    maximum:        1000000
  },

  discardValue:     659,            // quick-sell coin value
  loans:            0,              // > 0 = loan card, cannot trade
  untradeable:      false,

  isTradeable():    Boolean         // prototype method — checks untradeable + loans
}
```

> Use `item.isTradeable()` (method) rather than `item.untradeable` (property may not
> exist on club items).

---

### UTSearchCriteriaDTO (JS layer)

Constructor available in the web app page context: `new UTSearchCriteriaDTO()`

```javascript
{
  type:        "player",     // item type
  maskedDefId: 231747,       // 20-bit masked player ID
  defId:       [231747],     // alternative array form
  count:       21,           // results per page
  offset:      0,            // pagination offset
  rarities:    [55],         // NOTE: property name is "rarities", not "rarityIds"
  ovrMin:      90,           // min overall (concept search only)
  ovrMax:      95,           // max overall (concept search only)
  maxBuy:      1000000,      // max BIN price filter
  minBuy:      0             // min BIN price filter
}
```

---

## Enums & Constants

### ItemPile

| Value | Constant | Pile |
|-------|----------|------|
| 5 | `ItemPile.TRANSFER` | Trade pile |
| 6 | `ItemPile.PURCHASED` | Won auctions / purchased |
| 7 | `ItemPile.CLUB` | Club |
| 8 | `ItemPile.INBOX` | Inbox |
| 10 | `ItemPile.STORAGE` | Storage (concept items) |

### SearchType

| Constant | Description |
|----------|-------------|
| `SearchType.PLAYER` | Filter to player cards (used in club search) |

### Rarity IDs (`rarityIds` / `rareflag`)

| ID | Card Type |
|----|-----------|
| 1 | Common Gold |
| 3 | Rare Gold |
| 55 | Special (promo-dependent) |

### Position IDs

| ID | Position(s) |
|----|-------------|
| 3 | CB |
| 5 | LB / RB |
| 7 | LWB / RWB |
| 10 | CDM / CM |
| 12 | RM |
| 14 | CAM |
| 25 | ST |
| 27 | LW / RW |

---

## maskedDefId Formula

**[Observed]** — verified against multiple known players.

EA uses the bottom 20 bits of `definitionId` as the `maskedDefId` used in transfer market
queries:

```
maskedDefId = definitionId % 1048576
```

**Example:** Mbappé `definitionId = 117672259` → `117672259 % 1048576 = 231747`

You can also extract `maskedDefId` from cached network URLs without any API call:

```javascript
var entries = performance.getEntriesByType('resource');
for (var e of entries) {
  if (e.name.includes('/transfermarket?') && e.name.includes('maskedDefId=')) {
    var params = new URLSearchParams(e.name.split('?')[1]);
    var maskedDefId = parseInt(params.get('maskedDefId'));
    var rarityIds   = parseInt(params.get('rarityIds'));
  }
}
```

---

## Price Bands

**[Observed]** — EA enforces bid/BIN prices to be multiples of the increment for their range.
Submitting a price that doesn't conform results in an error.

| Price Range | Increment |
|-------------|-----------|
| 0 – 1,000 | 50 |
| 1,001 – 10,000 | 100 |
| 10,001 – 50,000 | 250 |
| 50,001 – 100,000 | 500 |
| 100,001+ | 1,000 |

**Snap a price to the nearest valid value:**

```javascript
function snapPrice(p) {
  var bands = [
    { max: 1000,     step: 50   },
    { max: 10000,    step: 100  },
    { max: 50000,    step: 250  },
    { max: 100000,   step: 500  },
    { max: Infinity, step: 1000 }
  ];
  for (var b of bands) {
    if (p <= b.max) return Math.round(p / b.step) * b.step;
  }
}
```

---

## EA Tax

EA deducts **5%** from the seller on every completed sale:

```
net = Math.floor(sellPrice * 0.95)
profit = net - buyPrice
```

---

## Rate Limits & Safety

**[Community]**

| Threshold | Behaviour |
|-----------|-----------|
| ~500 requests/hour | Recommended maximum to avoid detection |
| Exceeded | HTTP `503` responses; temporary block |
| Persistent abuse | Permanent account ban |

- Space requests by at least 200–500 ms between calls
- Do not run automated searches continuously — use delays between market pages
- EA's anti-bot systems are session-aware; logging out between sessions doesn't reset limits

---

## Third-Party Resources

| Resource | Description | URL |
|----------|-------------|-----|
| **FutDB** | Player database API with official API keys | https://futdb.app |
| **FUTBIN** | Market prices (uses `/25/` path for FC26) | https://www.futbin.com |
| **FUT.GG** | Market price reference | https://fut.gg |
| **futapi/fut** | Python library (FIFA 19-era, reference) | https://github.com/futapi/fut |
| **trydis/FIFA-Ultimate-Team-Toolkit** | C# implementation with endpoint constants | https://github.com/trydis/FIFA-Ultimate-Team-Toolkit |
| **TheNaeem/FifaSharp** | Modern C# EA FC web app API library | https://github.com/TheNaeem/FifaSharp |
| **fut docs** | Community-maintained endpoint reference | https://fut.readthedocs.io |

---

*Tested against EA FC 26 web app. Internal APIs are undocumented and subject to change without notice.*
