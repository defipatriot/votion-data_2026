# votion-data_2026

Weekly Votion epoch snapshots for the Alliance DAO (aDAO) Terra Liquidity Alliance positions.

Companion cron: [`defipatriot/cron-scripts/votion`](https://github.com/defipatriot/cron-scripts/tree/main/votion)

---

## Directory layout

```
votion-data_2026/
├── README.md                          ← you are here
├── votion-old/                        ← v1 schema archive (epochs 175-184)
│   ├── votion-epoch-175.json
│   ├── ...
│   └── votion-epoch-184.json
└── votion/                            ← v2 schema (epoch 185 onward)
    ├── votion-epoch-185.json
    ├── votion-epoch-186.json
    └── ...
```

### Why two folders

In early May 2026 we upgraded the votion cron from a thin wrapper over the Eris API response (v1) to a richer shape that mirrors what `tla-tool_ext.html` exports (v2). Rather than rewrite the historical files, we moved them to `votion-old/` and started accumulating v2 files in `votion/`.

Code that reads this repo should branch on the `schemaVersion` field:
- `schemaVersion: 1` → v1 file in `votion-old/`
- `schemaVersion: 2` → v2 file in `votion/`

---

## Refresh cadence

Once per epoch, captured at **Sunday 23:55 UTC** (4 minutes before epoch boundary at 23:59:59 UTC). Each capture is a complete snapshot of the closing epoch's state.

Files are immutable once written — epoch 185 will never be overwritten with different epoch-185 data.

---

## v2 schema — `votion-epoch-{N}.json`

```jsonc
{
  "schemaVersion": 2,
  "capturedAt":     "2026-05-12T10:25:57.426Z",
  "capturedAtUnix": 1778667957426,
  "period":         185,
  "voteBefore":     "2026-05-17T23:59:00.000Z",
  "total_vp":       12345678.90,

  "ratios": {
    "arbLUNA":  { "ratio": 1.0234, "apr": 12.45, "apy": 13.19, "windowDays": 30, "snapshotsAvailable": 30 },
    "ampLUNA":  { /* ... */ }
  },

  "prices": {
    "LUNA":   { "price": 0.184, "source": "coingecko", "fetchedAt": "..." },
    "ampLUNA":{ "price": 0.190, "source": "coingecko", "fetchedAt": "..." }
  },

  "lockups": [
    {
      "type":        "amp",
      "duration":    104,
      "multiplier":  4.0,
      "amount":      199893.45,
      "luna":        199893.45,
      "vp":          799573.80,
      "usd":         36782.30,
      "lockApy":     8.5,
      "lstApy":      11.2,
      "votionApy":   45.7,
      "votionApyCompound": 54.4,
      "period":      185,
      "expectedRewards": 260.26,
      "buckets": [
        {
          "name": "project",
          "expectedRewards": 145.6,
          "isWorthChanging": true,
          "potentialGain":   23.4,
          "pools": [
            { "name": "LUNA-USDC|Astroport", "current": 25.0, "optimized": 32.5, "change": 7.5, "address": "terra1..." }
          ]
        }
      ],
      "fetchedFromApi": true,
      "fetchedAt": "..."
    }
  ],

  "pools": {
    "LUNA-USDC|Astroport": {
      "current_vp":     1234567,
      "optimized_vp":   1500000,
      "current_pct":    10.2,
      "optimized_pct":  12.5,
      "bucket":         "stable",
      "lockup_contributions": [
        { "lockup_type": "amp-104", "vp": 800000, "pct_of_lockup": 30.0 }
      ]
    }
  },

  "fetchErrors": {}
}
```

### v1 (legacy) schema

`votion-old/*.json` files contain the raw Eris API response with fewer derived fields and no per-pool rollup. New code should target v2; v1 is for historical lookback only.

---

## What each cron run captures

The cron captures all aDAO multisig lockup positions known to Eris's Votion API. As of epoch 185 that's 5 lockups. The cron walks each one, calls Eris's optimization endpoint, derives per-lockup metrics, and rolls them up into a pool registry.

The `voteBefore` field is the cutoff timestamp for vote changes that affect this epoch. The capture happens 4 minutes before that boundary to lock in the final pre-flip state.

---

## Data sources

- **Eris Votion API** via proxy at `thealliancedao.com/api/eris` (the proxy preserves path, e.g. `/votion/liquidity-alliance/{lockupId}/optimization`)
- **CoinGecko** for token USD prices (rate-limited to public free tier)
- **Terra LCD** (`terra-rest.publicnode.com` + fallback `terra.publicnode.com`) for LST hub queries

If Eris's API is down, the cron retries with exponential backoff and eventually fails the run. Each run is a full rebuild so the next scheduled run repairs.

---

## Cross-references

- The `pools` registry uses canonical names matching `astroport-pool-data_2026` (LUNA-first when paired with LUNA)
- Pool addresses match those in `astroport-pool-data_2026` and `ss-pool-data_2026` — usable as join keys
- Per-pool VP × `bribes-data_2026` epoch bribes = vAPR for that pool
