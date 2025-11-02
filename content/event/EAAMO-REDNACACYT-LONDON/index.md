# File: package.json
{
  "name": "spreadsheet-site",
  "private": true,
  "version": "0.0.1",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@types/react": "^18.3.5",
    "@types/react-dom": "^18.3.0",
    "autoprefixer": "^10.4.20",
    "postcss": "^8.4.47",
    "tailwindcss": "^3.4.13",
    "typescript": "^5.6.2",
    "vite": "^5.4.8"
  }
}

# File: index.html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Spreadsheet → Website</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>

# File: src/main.tsx
import React from "react";
import { createRoot } from "react-dom/client";
import "./index.css";
import App from "./App";

const root = createRoot(document.getElementById("root")!);
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);

# File: src/App.tsx
import React, { useEffect, useMemo, useState } from "react";

// === CONFIG ===
// Paste a different Google Sheet ID if needed
const SHEET_ID = "1qbAh2in3b9i3y8ucBh8XprYwgP3Nq4Zp6cUhw8wgLo0";
// Optionally set a specific sheet (tab) name. Leave as null to use the first sheet.
const SHEET_NAME: string | null = null; // e.g., "Sheet1"

// === Helpers ===
function gvizUrl(sheetId: string, sheetName?: string | null) {
  const base = `https://docs.google.com/spreadsheets/d/${sheetId}/gviz/tq?tqx=out:json`;
  return sheetName ? `${base}&sheet=${encodeURIComponent(sheetName)}` : base;
}

function openUrl(sheetId: string) {
  return `https://docs.google.com/spreadsheets/d/${sheetId}/edit?usp=sharing`;
}

type Row = (string | number | null)[];

function parseGviz(text: string) {
  // Response looks like: google.visualization.Query.setResponse({...});
  const start = text.indexOf("{");
  const end = text.lastIndexOf("}");
  if (start === -1 || end === -1) throw new Error("Unexpected GViz payload");
  const json = JSON.parse(text.slice(start, end + 1));
  const table = json.table as {
    cols: { label: string }[];
    rows: { c: ({ v?: any; f?: any } | null)[] }[];
  };
  const headers = table.cols.map((c) => (c.label || "").trim());
  const rows: Row[] = table.rows.map((r) =>
    (r.c || []).map((cell) => {
      if (!cell) return null;
      // Prefer formatted value when present (dates, hyperlinks), else raw
      const v = cell.f ?? cell.v ?? null;
      return v;
    })
  );
  return { headers, rows };
}

function extractFirstUrl(value: string): string | null {
  const m = value.match(/https?:\/\/\S+/i);
  if (m) return m[0];
  const m2 = value.match(/\bwww\.[^\s)]+/i);
  if (m2) return `https://${m2[0]}`;
  return null;
}

function NormaliseCell({ value }: { value: any }) {
  // Preserve links: if the cell contains/looks like a URL, render an anchor.
  if (typeof value === "string") {
    // If Google sends rich text with <a>, keep it safely
    if (value.includes("<a ")) {
      return (
        <span
          className="underline decoration-dotted"
          dangerouslySetInnerHTML={{
            __html: value.replace(/target=\"?_self\"?/g, 'target="_blank" rel="noreferrer noopener"'),
          }}
        />
      );
    }
    const url = extractFirstUrl(value);
    if (url) {
      const label = value.length > 204 ? `${value.slice(0, 200)}…` : value;
      return (
        <a href={url} target="_blank" rel="noreferrer noopener" className="underline hover:no-underline">
          {label}
        </a>
      );
    }
  }
  return <span>{String(value)}</span>;
}

export default function SpreadsheetSite() {
  const [headers, setHeaders] = useState<string[]>([]);
  const [rows, setRows] = useState<Row[]>([]);
  const [query, setQuery] = useState("");
  const [sortBy, setSortBy] = useState<number | null>(null);
  const [sortDir, setSortDir] = useState<"asc" | "desc">("asc");
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let cancelled = false;
    async function load() {
      setLoading(true);
      setError(null);
      try {
        const res = await fetch(gvizUrl(SHEET_ID, SHEET_NAME), { cache: "no-store" });
        const text = await res.text();
        const { headers, rows } = parseGviz(text);
        if (!cancelled) {
          setHeaders(headers);
          setRows(rows);
        }
      } catch (e: any) {
        console.error(e);
        if (!cancelled) setError(e?.message || "Failed to load Google Sheet.");
      } finally {
        if (!cancelled) setLoading(false);
      }
    }
    load();
    return () => {
      cancelled = true;
    };
  }, []);

  const filtered = useMemo(() => {
    if (!query) return rows;
    const q = query.toLowerCase();
    return rows.filter((r) => r.some((c) => String(c ?? "").toLowerCase().includes(q)));
  }, [rows, query]);

  const sorted = useMemo(() => {
    if (sortBy == null) return filtered;
    const idx = sortBy;
    const dir = sortDir === "asc" ? 1 : -1;
    return [...filtered].sort((a, b) => {
      const av = String(a[idx] ?? "").toLowerCase();
      const bv = String(b[idx] ?? "").toLowerCase();
      return av < bv ? -1 * dir : av > bv ? 1 * dir : 0;
    });
  }, [filtered, sortBy, sortDir]);

  return (
    <div className="min-h-screen bg-gray-50 text-gray-900">
      <header className="sticky top-0 z-10 backdrop-blur bg-white/70 border-b">
        <div className="max-w-6xl mx-auto px-4 py-4 flex flex-col gap-3 md:flex-row md:items-center md:justify-between">
          <h1 className="text-2xl font-semibold tracking-tight">Spreadsheet → Website</h1>
          <div className="flex gap-2 items-center w-full md:w-auto">
            <input
              className="w-full md:w-64 rounded-xl border px-4 py-2 focus:outline-none focus:ring"
              placeholder="Search…"
              value={query}
              onChange={(e) => setQuery(e.target.value)}
            />
            <a
              href={openUrl(SHEET_ID)}
              target="_blank"
              rel="noreferrer noopener"
              className="rounded-xl border px-4 py-2 hover:bg-gray-100"
            >
              Open Sheet
            </a>
          </div>
        </div>
      </header>

      <main className="max-w-6xl mx-auto px-4 py-6">
        {loading && <div className="animate-pulse text-sm text-gray-600">Loading data from Google Sheets…</div>}

        {error && (
          <div className="mb-4 rounded-xl border border-red-2 00 bg-red-50 p-4 text-sm text-red-700">
            <p className="font-medium">Couldn’t fetch the live data.</p>
            <p className="mt-1">
              {error} — If this sheet isn’t publicly viewable or publishing is disabled, either share it publicly or use
              the fallback embed below.
            </p>
          </div>
        )}

        {!loading && headers.length > 0 && (
          <div className="overflow-x-auto rounded-2xl shadow-sm border bg-white">
            <table className="w-full text-left text-sm">
              <thead className="bg-gray-100/60">
                <tr>
                  {headers.map((h, i) => (
                    <th key={i} className="px-4 py-3 whitespace-nowrap">
                      <button
                        className="flex items-center gap-1 hover:underline"
                        onClick={() => {
                          if (sortBy === i) setSortDir(sortDir === "asc" ? "desc" : "asc");
                          else {
                            setSortBy(i);
                            setSortDir("asc");
                          }
                        }}
                        title="Sort"
                      >
                        <span>{h || `Column ${i + 1}`}</span>
                        {sortBy === i && <span className="text-xs">{sortDir === "asc" ? "▲" : "▼"}</span>}
                      </button>
                    </th>
                  ))}
                </tr>
              </thead>
              <tbody>
                {sorted.map((row, rIdx) => (
                  <tr key={rIdx} className={rIdx % 2 ? "bg-gray-50/40" : ""}>
                    {headers.map((_, cIdx) => {
                      const value = row[cIdx] ?? "";
                      return (
                        <td key={cIdx} className="px-4 py-3 align-top max-w-[28rem]">
                          {value === null || value === undefined || value === "" ? (
                            <span className="text-gray-400">—</span>
                          ) : (
                            <NormaliseCell value={value} />
                          )}
                        </td>
                      );
                    })}
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        )}

        {/* Fallback embed */}
        <section className="mt-8">
          <h2 className="text-lg font-medium mb-2">Fallback View (iframe)</h2>
          <p className="text-sm text-gray-600 mb-3">
            If the table above is empty due to restrictions, you can still view the spreadsheet directly below or via the
            “Open Sheet” button.
          </p>
          <div className="aspect-[16/10] w-full rounded-2xl overflow-hidden border bg-white">
            <iframe title="Google Sheet" className="w-full h-full" src={`https://docs.google.com/spreadsheets/d/${SHEET_ID}/preview`} />
          </div>
        </section>
      </main>

      <footer className="max-w-6xl mx-auto px-4 py-10 text-xs text-gray-500">
        <p>
          Built automatically from a Google Sheet. Links inside cells are preserved. Sort columns by clicking their
          headers. Search across all columns.
        </p>
      </footer>
    </div>
  );
}

# File: src/index.css
@tailwind base;
@tailwind components;
@tailwind utilities;

html, body, #root { height: 100%; }

# File: tailwind.config.ts
import type { Config } from "tailwindcss";

export default {
  content: ["./index.html", "./src/**/*.{ts,tsx}"],
  theme: { extend: {} },
  plugins: [],
} satisfies Config;

# File: postcss.config.js
export default {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};

# File: tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true
  },
  "include": ["src"]
}

# File: netlify.toml
[build]
  command = "npm run build"
  publish = "dist"

[dev]
  framework = "vite"

# File: .gitignore
node_modules
.DS_Store
.dist
/build
.env

# File: README.md
# Spreadsheet → Website (React + Vite + Tailwind)

This is a minimal React site that turns a public Google Sheet into a searchable, sortable table. It matches the component you shared and is ready to deploy to Netlify.

## 1) Local dev
```bash
npm i
npm run dev
```

## 2) Configure the data source
- Make sure your Google Sheet is shared as **Anyone with the link – Viewer** (or **Published to the web**).
- Update `SHEET_ID` (and optionally `SHEET_NAME`) in `src/App.tsx`.

## 3) Deploy on Netlify
- Connect this folder/repo to Netlify.
- Build command: `npm run build`
- Publish directory: `dist`
- `netlify.toml` is already included, so autodetection should work.

## 4) Migrating from your Hugo/Wowchemy site
- If you want this to **replace** the existing site: either switch the site’s repo in Netlify to this one, or add it as a subdirectory in your current repo and set Netlify’s **Base directory** to that folder.
- If you want to keep Hugo for other pages, you can mount this React app at a subpath and add a link from the nav.

## Notes
- The app fetches Google’s GViz endpoint without API keys. The sheet must be publicly viewable for this to work.
- Links inside cells are preserved. Click a column header to sort. Use the search box to filter across all columns.

