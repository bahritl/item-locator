# item-locator
#!/usr/bin/env bash
set -euo pipefail

PROJECT_DIR="item-locator"
ZIP_NAME="${PROJECT_DIR}.zip"

if [ -d "$PROJECT_DIR" ]; then
  echo "Directory $PROJECT_DIR already exists. Remove or choose another location."
  exit 1
fi

mkdir "$PROJECT_DIR"
cd "$PROJECT_DIR"

cat > package.json <<'JSON'
{
  "name": "item-locator",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev -p 3000",
    "build": "next build",
    "start": "next start -p 3000",
    "zip": "cd .. && zip -r item-locator.zip item-locator"
  },
  "dependencies": {
    "idb": "^7.0.1",
    "next": "13.4.12",
    "react": "18.2.0",
    "react-dom": "18.2.0",
    "@zxing/browser": "^0.0.11",
    "uuid": "^9.0.0"
  },
  "devDependencies": {
    "typescript": "^5.2.2"
  }
}
JSON

cat > next.config.js <<'JS'
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
}
module.exports = nextConfig
JS

cat > tsconfig.json <<'TS'
{
  "compilerOptions": {
    "target": "es2020",
    "lib": ["dom","dom.iterable","es2020"],
    "allowJs": false,
    "skipLibCheck": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "module": "esnext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx"
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules"]
}
TS

mkdir -p pages/api components lib pages public styles

cat > pages/_app.tsx <<'APP'
import '../styles/globals.css'
import type { AppProps } from 'next/app'

export default function App({ Component, pageProps }: AppProps) {
  return <Component {...pageProps} />
}
APP

cat > styles/globals.css <<'CSS'
html,body,#__next { height: 100%; }
body { margin: 0; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial; padding: 16px; }
.container { max-width: 900px; margin: 0 auto; }
.btn { padding: 8px 12px; border-radius: 6px; border: none; background: #0366d6; color: white; cursor: pointer; }
.btn-ghost { padding: 8px 12px; border-radius: 6px; border: 1px solid #e5e7eb; background: white; color: #111; cursor: pointer; }
.card { border: 1px solid #e5e7eb; padding: 12px; border-radius: 8px; margin-top: 12px; }
CSS

cat > pages/index.tsx <<'INDEX'
import React, { useState } from 'react'
import ContainerSelector from '../components/ContainerSelector'
import NearbyForContainer from '../components/NearbyForContainer'

export default function Home() {
  const [containerId, setContainerId] = useState<string | null>(null)
  return (
    <div className="container">
      <h1>Item Locator — Nearby scan filtered by container</h1>
      <p>Demo: scan or type a container ID and start nearby scan. Confirm assignments when prompted.</p>
      {!containerId ? (
        <div className="card">
          <ContainerSelector onSelect={(id) => setContainerId(id)} />
        </div>
      ) : (
        <div>
          <div style={{display:'flex', gap:8, alignItems:'center'}}>
            <strong>Active container:</strong> <span>{containerId}</span>
            <button className="btn-ghost" onClick={() => setContainerId(null)}>Change</button>
          </div>
          <div className="card"><NearbyForContainer containerClientId={containerId} /></div>
        </div>
      )}
    </div>
  )
}
INDEX

cat > components/MultiScan.tsx <<'MULTI'
import React, { useEffect, useRef } from 'react';

type Props = { onDetected: (code: string) => void; enabled?: boolean; preferredFormats?: string[] };

export default function MultiScan({ onDetected, enabled = true, preferredFormats }: Props) {
  const videoRef = useRef<HTMLVideoElement | null>(null);
  const seen = useRef<Set<string>>(new Set());
  const readerRef = useRef<any>(null);

  useEffect(() => {
    if (!enabled) return;
    let mounted = true;
    async function start() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } });
        const video = videoRef.current!;
        video.srcObject = stream;
        await video.play();

        if ('BarcodeDetector' in window) {
          // @ts-ignore
          const formats = preferredFormats || ['code_128', 'code_39', 'ean_13', 'ean_8', 'upc_a', 'upc_e'];
          // @ts-ignore
          const detector = new (window as any).BarcodeDetector({ formats });
          const loop = async () => {
            if (!mounted) return;
            try {
              const bitmap = await createImageBitmap(video);
              // @ts-ignore
              const barcodes = await detector.detect(bitmap);
              for (const b of barcodes) {
                const code = b.rawValue;
                if (code && !seen.current.has(code)) {
                  seen.current.add(code);
                  onDetected(code);
                }
              }
              bitmap.close();
            } catch (e) {
              // ignore frame errors
            }
            requestAnimationFrame(loop);
          };
          requestAnimationFrame(loop);
        } else {
          const { BrowserMultiFormatReader } = await import('@zxing/browser');
          readerRef.current = new BrowserMultiFormatReader();
          const devices = await BrowserMultiFormatReader.listVideoInputDevices();
          const deviceId = devices[0]?.deviceId;
          readerRef.current.decodeFromVideoDevice(deviceId, video, (result: any) => {
            if (result) {
              const code = result.getText();
              if (code && !seen.current.has(code)) {
                seen.current.add(code);
                onDetected(code);
              }
            }
          });
        }
      } catch (err) {
        console.error('MultiScan start error', err);
      }
    }
    start();

    return () => {
      mounted = false;
      try { if (readerRef.current) readerRef.current.reset(); } catch {}
      const v = videoRef.current;
      if (v?.srcObject) {
        const ms = v.srcObject as MediaStream;
        ms.getTracks().forEach(t => t.stop());
      }
    };
  }, [enabled, onDetected, preferredFormats]);

  return (
    <div>
      <video ref={videoRef} style={{ width: '100%', height: 'auto', background: 'black', borderRadius: 8 }} playsInline muted />
      <div style={{ fontSize: 12, color: '#6b7280', marginTop: 8 }}>Point camera at items — detected barcodes will appear in the list.</div>
    </div>
  );
}
MULTI

cat > components/ContainerSelector.tsx <<'CONT'
import React, { useState } from 'react';
import MultiScan from './MultiScan';

export default function ContainerSelector({ onSelect }: { onSelect: (containerId: string) => void }) {
  const [manual, setManual] = useState('');
  const [scanning, setScanning] = useState(false);

  return (
    <div>
      <div style={{display:'flex', gap:8}}>
        <input style={{padding:8,borderRadius:6,border:'1px solid #e5e7eb'}} value={manual} onChange={e => setManual(e.target.value)} placeholder="Type container ID" />
        <button className="btn" onClick={() => { if (manual.trim()) onSelect(manual.trim()); }}>Use</button>
        <button className="btn-ghost" onClick={() => setScanning(s => !s)}>{scanning ? 'Stop scan' : 'Scan container'}</button>
      </div>

      {scanning && (
        <div style={{marginTop:12}}>
          <MultiScan onDetected={(code) => { setScanning(false); onSelect(code); }} />
        </div>
      )}
    </div>
  );
}
CONT

cat > components/ConfirmAssignmentModal.tsx <<'MODAL'
import React from 'react';

type Props = {
  open: boolean;
  code: string;
  existingItem?: { clientId: string; itemCode?: string; barcode?: string; containerClientId?: string };
  targetContainerClientId: string;
  onConfirmAssign: (itemClientId: string | null) => Promise<void>;
  onCancel: () => void;
};

export default function ConfirmAssignmentModal({ open, code, existingItem, targetContainerClientId, onConfirmAssign, onCancel }: Props) {
  if (!open) return null;
  return (
    <div style={{position:'fixed',inset:0,display:'flex',alignItems:'center',justifyContent:'center',background:'rgba(0,0,0,0.4)'}}>
      <div style={{background:'white',padding:16,borderRadius:8,width:'90%',maxWidth:480}}>
        <h3 style={{margin:0}}>Confirm assignment</h3>
        <p style={{color:'#6b7280'}}>Scanned code: <strong>{code}</strong></p>

        {existingItem ? (
          <div>
            <p>Existing item: <strong>{existingItem.itemCode || existingItem.barcode}</strong></p>
            <p style={{fontSize:12,color:'#6b7280'}}>Currently assigned to container: <strong>{existingItem.containerClientId || 'none'}</strong></p>
            <div style={{marginTop:12,display:'flex',gap:8}}>
              <button className="btn" onClick={() => onConfirmAssign(existingItem.clientId)}>Reassign to this container</button>
              <button className="btn-ghost" onClick={onCancel}>Keep existing</button>
            </div>
          </div>
        ) : (
          <div>
            <p style={{fontSize:13,color:'#374151'}}>No matching item found. Create a new item and assign to this container?</p>
            <div style={{marginTop:12,display:'flex',gap:8}}>
              <button className="btn" onClick={() => onConfirmAssign(null)}>Create & Assign</button>
              <button className="btn-ghost" onClick={onCancel}>Skip</button>
            </div>
          </div>
        )}
      </div>
    </div>
  );
}
MODAL

cat > components/NearbyForContainer.tsx <<'NEARBY'
import React, { useEffect, useState } from 'react';
import MultiScan from './MultiScan';
import ConfirmAssignmentModal from './ConfirmAssignmentModal';
import { getDB, enqueueSync } from '../lib/local-db';

export default function NearbyForContainer({ containerClientId }:{ containerClientId: string }) {
  const [matches, setMatches] = useState<any[]>([]);
  const [unmatched, setUnmatched] = useState<string[]>([]);
  const [conflicts, setConflicts] = useState<any[]>([]);
  const [modal, setModal] = useState<{open:boolean, code:string, existing?:any}>({open:false, code:'', existing:undefined});

  useEffect(() => {
    // load matches for container from local DB
    (async () => {
      const db = await getDB();
      const all = await db.getAll('items');
      setMatches(all.filter((i:any)=>i.containerClientId===containerClientId));
    })();
  }, [containerClientId]);

  async function handleDetected(code: string) {
    // lookup in local db
    const db = await getDB();
    const idx = db.transaction('items').objectStore('items');
    let found = null;
    const all = await idx.getAll();
    found = all.find((i:any)=> i.barcode===code || i.itemCode===code || i.clientId===code);
    if (found) {
      if (found.containerClientId === containerClientId) {
        // mark present locally
        found.presenceStatus = 'present';
        found.lastSeenAt = new Date().toISOString();
        found.syncStatus = 'pending';
        await db.put('items', found);
        setMatches(m => {
          const updated = m.filter((x:any)=>x.clientId!==found.clientId).concat(found);
          return updated;
        });
        // enqueue update
        await enqueueSync({ type:'item', method:'update', payload: { clientId: found.clientId, updates: { presenceStatus: 'present', lastSeenAt: found.lastSeenAt } } });
      } else {
        setConflicts(c => {
          if (!c.find((x:any)=>x.clientId===found.clientId)) return [...c, found];
          return c;
        });
        setModal({ open:true, code, existing:found });
      }
    } else {
      // unmatched
      if (!unmatched.includes(code)) setUnmatched(u => [code,...u]);
      setModal({ open:true, code, existing:undefined });
    }
  }

  async function onConfirmAssign(itemClientId: string | null) {
    const db = await getDB();
    if (!itemClientId) {
      // create new item
      const newId = 'cli_' + Math.random().toString(36).slice(2,9);
      const item = { clientId: newId, barcode: modal.code, itemCode: undefined, description: 'Created from scan', containerClientId, presenceStatus: 'present', lastSeenAt: new Date().toISOString(), syncStatus: 'pending', updatedAt: new Date().toISOString() };
      await db.put('items', item);
      setMatches(m => [item, ...m]);
      await enqueueSync({ type:'item', method:'create', payload: { item } });
    } else {
      // reassign existing item
      const item = await db.get('items', itemClientId);
      if (item) {
        item.containerClientId = containerClientId;
        item.updatedAt = new Date().toISOString();
        item.syncStatus = 'pending';
        await db.put('items', item);
        setMatches(m => [item, ...m.filter((x:any)=>x.clientId !== item.clientId)]);
        await enqueueSync({ type:'item', method:'update', payload: { clientId: item.clientId, updates: { containerClientId } } });
      }
    }
    setModal({ open:false, code:'', existing:undefined });
  }

  return (
    <div>
      <div style={{display:'grid',gridTemplateColumns:'1fr 360px', gap:12}}>
        <div>
          <div style={{marginBottom:12}}>
            <MultiScan onDetected={(c)=>handleDetected(c)} />
          </div>

          <div>
            <h4>Matches</h4>
            {matches.map(m => (
              <div key={m.clientId} style={{border:'1px solid #e5e7eb',padding:8,borderRadius:6,marginTop:8}}>
                <div><strong>{m.itemCode || m.barcode || m.clientId}</strong></div>
                <div style={{fontSize:12,color:'#6b7280'}}>Presence: {m.presenceStatus || 'unknown'}</div>
              </div>
            ))}
          </div>
        </div>

        <div>
          <h4>Unmatched</h4>
          {unmatched.map(u => (<div key={u} style={{padding:8,border:'1px solid #fde68a',borderRadius:6,marginTop:8}}>{u}</div>))}

          <h4 style={{marginTop:12}}>Conflicts</h4>
          {conflicts.map(cf => (<div key={cf.clientId} style={{padding:8,border:'1px solid #fecaca',borderRadius:6,marginTop:8}}>{cf.itemCode || cf.barcode} — assigned to {cf.containerClientId}</div>))}
        </div>
      </div>

      <ConfirmAssignmentModal
        open={modal.open}
        code={modal.code}
        existingItem={modal.existing}
        targetContainerClientId={containerClientId}
        onConfirmAssign={onConfirmAssign}
        onCancel={() => setModal({ open:false, code:'', existing:undefined })}
      />
    </div>
  )
}
NEARBY

cat > lib/local-db.ts <<'LDB'
import { openDB, DBSchema, IDBPDatabase } from 'idb';
import { v4 as uuidv4 } from 'uuid';

type SyncStatus = 'pending' | 'synced' | 'failed';

interface AppDB extends DBSchema {
  items: {
    key: string;
    value: {
      clientId: string;
      serverId?: number;
      itemCode?: string;
      barcode?: string;
      description?: string;
      containerClientId?: string;
      containerServerId?: number;
      presenceStatus?: 'present' | 'missing' | null;
      lastSeenAt?: string;
      syncStatus?: SyncStatus;
      updatedAt?: string;
    };
    indexes: { 'by-barcode': string; 'by-itemCode': string; 'by-container': string };
  };
  presenceRuns: {
    key: string;
    value: {
      runId: string;
      containerClientId: string;
      containerName?: string;
      scannedCodes: string[];
      startedAt: string;
      finishedAt?: string;
    };
  };
  syncQueue: {
    key: string;
    value: {
      id: string;
      type: 'item' | 'presenceRun' | 'other';
      method: 'create' | 'update' | 'delete' | 'batch';
      payload: any;
      createdAt: string;
      attempts?: number;
    };
    indexes: { 'by-createdAt': string };
  };
}

let _db: IDBPDatabase<AppDB> | null = null;

export async function getDB() {
  if (_db) return _db;
  _db = await openDB<AppDB>('item-locator-db', 1, {
    upgrade(db) {
      const items = db.createObjectStore('items', { keyPath: 'clientId' });
      items.createIndex('by-barcode', 'barcode', { unique: false });
      items.createIndex('by-itemCode', 'itemCode', { unique: false });
      items.createIndex('by-container', 'containerClientId', { unique: false });

      db.createObjectStore('presenceRuns', { keyPath: 'runId' });

      const q = db.createObjectStore('syncQueue', { keyPath: 'id' });
      q.createIndex('by-createdAt', 'createdAt');
    },
  });
  return _db!;
}

export async function enqueueSync(item: {
  type: AppDB['syncQueue']['value']['type'];
  method: AppDB['syncQueue']['value']['method'];
  payload: any;
}) {
  const db = await getDB();
  const id = uuidv4();
  const now = new Date().toISOString();
  await db.put('syncQueue', { id, ...item, createdAt: now, attempts: 0 });
  return id;
}
LDB

cat > lib/presence-run.ts <<'PR'
import { getDB, enqueueSync } from './local-db';
import { v4 as uuidv4 } from 'uuid';

export async function startPresenceRun(containerClientId: string, containerName?: string) {
  const db = await getDB();
  const runId = uuidv4();
  const startedAt = new Date().toISOString();
  await db.put('presenceRuns', { runId, containerClientId, containerName, scannedCodes: [], startedAt });
  return runId;
}

export async function recordScan(runId: string, code: string) {
  const db = await getDB();
  const run = await db.get('presenceRuns', runId);
  if (!run) throw new Error('no active run');
  if (!run.scannedCodes.includes(code)) {
    run.scannedCodes.push(code);
    await db.put('presenceRuns', run);
  }
}

export async function finishPresenceRun(runId: string, actorClientId?: string) {
  const db = await getDB();
  const run = await db.get('presenceRuns', runId);
  if (!run) throw new Error('no such run');
  run.finishedAt = new Date().toISOString();
  await db.put('presenceRuns', run);

  const tx = db.transaction(['items', 'syncQueue'], 'readwrite');
  const itemsStore = tx.objectStore('items');
  const allItems = await itemsStore.getAll();
  const containerItems = allItems.filter(i => i.containerClientId === run.containerClientId);

  const scannedSet = new Set(run.scannedCodes);
  const now = new Date().toISOString();
  const updates: any[] = [];

  for (const item of containerItems) {
    const matched = scannedSet.has(item.clientId) || (item.barcode && scannedSet.has(item.barcode)) || (item.itemCode && scannedSet.has(item.itemCode));
    const newPresence = matched ? 'present' : 'missing';
    if (item.presenceStatus !== newPresence || (matched && item.lastSeenAt !== now)) {
      item.presenceStatus = newPresence;
      if (matched) item.lastSeenAt = now;
      item.syncStatus = 'pending';
      item.updatedAt = now;
      await itemsStore.put(item);

      updates.push({
        clientId: item.clientId,
        updates: { presenceStatus: item.presenceStatus, lastSeenAt: item.lastSeenAt }
      });
    }
  }

  if (updates.length > 0) {
    await tx.objectStore('syncQueue').put({
      id: uuidv4(),
      type: 'item',
      method: 'batch',
      payload: {
        kind: 'presenceBatch',
        runId: run.runId,
        containerClientId: run.containerClientId,
        updates,
        scannedCodes: run.scannedCodes,
        finishedAt: run.finishedAt,
        actorClientId
      },
      createdAt: now,
      attempts: 0
    });
  }

  await tx.done;
  return { scanned: run.scannedCodes.length, containerItems: containerItems.length, updatesCount: updates.length };
}
PR

cat > pages/api/sync.ts <<'SYNC'
import type { NextApiRequest, NextApiResponse } from 'next';
import fs from 'fs';
import path from 'path';

const STORE = path.resolve(process.cwd(), 'sync-store.json');

function readStore() {
  try {
    const raw = fs.readFileSync(STORE, 'utf-8');
    return JSON.parse(raw);
  } catch {
    return { items: [], presenceRecords: [], changes: [] };
  }
}
function writeStore(data: any) {
  fs.writeFileSync(STORE, JSON.stringify(data, null, 2), 'utf-8');
}

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'POST') return res.status(405).end();
  const { changes, clientDeviceId, actorId } = req.body as any;
  const store = readStore();
  const results: any[] = [];
  for (const ch of changes || []) {
    try {
      // very small stub: record the change to store.changes and mark ok
      store.changes.push({ id: ch.id, type: ch.type, method: ch.method, payload: ch.payload, receivedAt: new Date().toISOString() });
      // If it's a presenceBatch, create presenceRecords for present items
      if (ch.type === 'item' && ch.method === 'batch' && ch.payload?.kind === 'presenceBatch') {
        const updates = ch.payload.updates || [];
        for (const u of updates) {
          if (u.updates?.presenceStatus === 'present') {
            store.presenceRecords.push({
              itemClientId: u.clientId,
              containerClientId: ch.payload.containerClientId,
              scannedAt: u.updates.lastSeenAt || new Date().toISOString(),
              by: actorId || clientDeviceId || null
            });
          }
        }
      }
      results.push({ id: ch.id, ok: true });
    } catch (err: any) {
      results.push({ id: ch.id, ok: false, reason: err.message });
    }
  }
  writeStore(store);
  res.status(200).json({ results });
}
SYNC

cat > README.md <<'MD'
# Item Locator — Nearby-scan filtered-by-container (Demo)

This is a small Next.js demo app implementing:
- Continuous nearby barcode scanning (BarcodeDetector + ZXing fallback)
- Container context and confirm-before-assign flows
- Local persistence in IndexedDB (idb)
- Simple file-backed /api/sync stub for demo/testing

Demo credentials (no auth required for the demo).
How to run:
1. Install dependencies:
   npm install
2. Run dev server:
   npm run dev
3. Open http://localhost:3000 on your phone or desktop (camera required for scanning).
4. On first run, scan or type a container ID and start nearby scanning.

Testing:
- The app stores local items in IndexedDB. Confirm assignments appear in Matches and are queued.
- The API endpoint POST /api/sync records changes to sync-store.json in the project root (demo only).

Packaging:
- After running the included script this project will be zipped as item-locator.zip.

Notes:
- This is a demo stub. For production you should replace /api/sync with a secure server (Prisma + DB) and add authentication.
MD

cat > public/sample-barcodes.csv <<'CSV'
barcode
123456789012
987654321098
ABC123XYZ
CODE128-EXAMPLE-001
CSV

# install node deps
echo "Installing npm dependencies (this may take a minute)..."
npm install --no-audit --no-fund

# done
cd ..
echo "Creating zip ${ZIP_NAME} ..."
zip -r "${ZIP_NAME}" "${PROJECT_DIR}" > /dev/null

# print sha256
if command -v sha256sum >/dev/null 2>&1; then
  echo "SHA256:"
  sha256sum "${ZIP_NAME}"
elif command -v shasum >/dev/null 2>&1; then
  echo "SHA256:"
  shasum -a 256 "${ZIP_NAME}"
else
  echo "Zip created at ${ZIP_NAME}. (No sha256 tool found to print checksum.)"
fi

echo "Done. Open ${PROJECT_DIR} and run 'npm run dev' inside the project to start the app."
