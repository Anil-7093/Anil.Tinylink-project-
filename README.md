# TinyLink — A simple URL shortener

## Overview
TinyLink is a minimal URL shortener built with Next.js and Postgres (Neon). It supports:
- Create short links (optional custom code)
- Redirect (`/:code`) with 302
- Click counting and `last_clicked` timestamp
- Delete links
- Dashboard with list & controls
- Stats page `/code/:code`
- Health check `/healthz`
- API endpoints for autograding

## Tech
- Next.js
- node-postgres (`pg`)
- Plain CSS
- Deploy on Vercel + Neon

## Setup
1. Copy `.env.example` to `.env.local` and set `DATABASE_URL` and `NEXT_PUBLIC_BASE_URL`.
2. Run SQL in `schema.sql` to create `links` table.
3. `npm install`
4. `npm run dev` — open http://localhost:3000

## Endpoints (autograder)
- `GET /healthz` — 200 `{ ok: true, version: "1.0" }`
- `POST /api/links` — create link (409 if exists)
- `GET /api/links` — list links
- `GET /api/links/:code` — stats
- `DELETE /api/links/:code` — delete link
- `GET /:code` — redirect (302)
- `GET /code/:code` — UI stats page

## Deploy
See instructions in project root or deploy to Vercel and set environment variables.

## Tests
Use the curl commands listed in the repo to validate the autograder checks.

## Notes
- Codes must match `/^[A-Za-z0-9]{6,8}$/`.
pages
// pages/_app.js
import '../public/styles.css'

export default function App({ Component, pageProps }) {
  return <Component {...pageProps} />
}
public /style/CSS
/* minimal CSS to meet "clean" UI requirements */
:root{
  --accent:#0b63d6;
  --muted:#666;
}
*{box-sizing:border-box}
html,body,#__next{height:100%}
body{
  font-family:Inter,Segoe UI,Helvetica,Arial,sans-serif;
  margin:0;
  padding:0;
  background:#f7f8fb;
  color:#111;
}
.container{
  max-width:980px;
  margin:28px auto;
  padding:18px;
  background:#fff;
  border-radius:8px;
  box-shadow:0 6px 20px rgba(20,20,40,0.06);
}
.header{display:flex;align-items:center;justify-content:space-between;margin-bottom:14px}
.h1{font-size:20px;font-weight:600}
.form-row{display:flex;gap:8px;align-items:center;margin-bottom:12px}
.input,textarea,select{
  padding:10px;border:1px solid #e2e6ef;border-radius:6px;flex:1;
}
.button{
  background:var(--accent);color:#fff;border:none;padding:10px 14px;border-radius:6px;cursor:pointer;
}
.button.secondary{background:#fff;color:var(--accent);border:1px solid #e0e7ff}
.table{width:100%;border-collapse:collapse;margin-top:12px}
.table th{ text-align:left;padding:10px;border-bottom:1px solid #eee;color:var(--muted)}
.table td{ padding:10px;border-bottom:1px dashed #f1f3f8;vertical-align:middle}
.small{font-size:13px;color:var(--muted)}
.badge{font-size:12px;padding:6px 8px;border-radius:6px;background:#f1f7ff;color:var(--accent)}
.actions{display:flex;gap:8px}
.copy-btn{background:#f6f7fb;border:1px solid #e7eaf6;padding:8px;border-radius:6px;cursor:pointer}
@media(max-width:700px){
  .form-row{flex-direction:column;align-items:stretch}
  .actions{flex-direction:column}
}
API: pages/api/links/index.js (POST + GET)

// pages/api/links/index.js
import { query } from '../../../lib/db';

const CODE_REGEX = /^[A-Za-z0-9]{6,8}$/;

function isValidUrl(s) {
  try {
    const u = new URL(s);
    return u.protocol === 'http:' || u.protocol === 'https:';
  } catch (e) {
    return false;
  }
}

export default async function handler(req, res) {
  if (req.method === 'GET') {
    try {
      const r = await query('SELECT code, target_url, clicks, last_clicked, created_at FROM links ORDER BY created_at DESC');
      return res.status(200).json({ links: r.rows });
    } catch (e) {
      console.error(e);
      return res.status(500).json({ error: 'db_error' });
    }
  }

  if (req.method === 'POST') {
    const { targetUrl, code } = req.body || {};
    if (!targetUrl || !isValidUrl(targetUrl)) {
      return res.status(400).json({ error: 'invalid_url' });
    }

    let finalCode = code && String(code).trim();
    if (finalCode) {
      if (!CODE_REGEX.test(finalCode)) {
        return res.status(400).json({ error: 'invalid_code', message: 'Codes must match /^[A-Za-z0-9]{6,8}$/' });
      }
    } else {
      // generate a random 6-char code
      const gen = () => Math.random().toString(36).slice(2, 8).replace(/[^A-Za-z0-9]/g,'').slice(0,6);
      finalCode = gen();
      // ensure uniqueness (very unlikely collision)
      let exists = true;
      for (let i=0;i<5 && exists;i++){
        const r = await query('SELECT code FROM links WHERE code=$1', [finalCode]);
        if (r.rowCount === 0) exists = false;
        else finalCode = gen();
      }
    }

    try {
      await query('INSERT INTO links(code, target_url) VALUES($1,$2)', [finalCode, targetUrl]);
      const base = process.env.NEXT_PUBLIC_BASE_URL || `http://localhost:3000`;
      return res.status(201).json({ code: finalCode, shortUrl: `${base}/${finalCode}` });
    } catch (e) {
      // unique violation
      if (e.code === '23505') {
        return res.status(409).json({ error: 'code_exists' });
      }
      console.error(e);
      return res.status(500).json({ error: 'db_error' });
    }
  }

  res.setHeader('Allow', ['GET','POST']);
  return res.status(405).end(`Method ${req.method} Not Allowed`);
}


---

API: pages/api/links/[code].js (GET stats, DELETE)

// pages/api/links/[code].js
import { query } from '../../../lib/db';

export default async function handler(req, res) {
  const { code } = req.query;

  if (!code) return res.status(400).json({ error: 'missing_code' });

  if (req.method === 'GET') {
    try {
      const r = await query('SELECT code, target_url, clicks, last_clicked, created_at FROM links WHERE code=$1', [code]);
      if (r.rowCount === 0) return res.status(404).json({ error: 'not_found' });
      return res.status(200).json({ link: r.rows[0] });
    } catch (e) {
      console.error(e);
      return res.status(500).json({ error: 'db_error' });
    }
  }

  if (req.method === 'DELETE') {
    try {
      const r = await query('DELETE FROM links WHERE code=$1 RETURNING code', [code]);
      if (r.rowCount === 0) return res.status(404).json({ error: 'not_found' });
      return res.status(200).json({ ok: true, deleted: code });
    } catch (e) {
      console.error(e);
      return res.status(500).json({ error: 'db_error' });
    }
  }

  res.setHeader('Allow', ['GET','DELETE']);
  return res.status(405).end(`Method ${req.method} Not Allowed`);
}


---

Health check pages/api/healthz.js (GET /healthz)

// pages/api/healthz.js
export default function handler(req, res) {
  return res.status(200).json({ ok: true, version: "1.0" });
}

Also route /healthz in Next.js pages (for direct path). Create /pages/healthz.js to show same response server-side:

pages/healthz.js

export async function getServerSideProps({ res }) {
  res.setHeader('Content-Type','application/json');
  res.statusCode = 200;
  res.end(JSON.stringify({ ok:true, version:'1.0' }));
  return { props: {} };
}

export default function Page() {
  return null;
}


---

Redirect page /pages/[code].js

// pages/[code].js
import { query } from '../lib/db';

export async function getServerSideProps({ params, res }) {
  const { code } = params;
  try {
    const r = await query('SELECT target_url FROM links WHERE code=$1', [code]);
    if (r.rowCount === 0) {
      res.statusCode = 404;
      return { props: { notFound: true, code } };
    }
    const target = r.rows[0].target_url;
    // increment clicks & last_clicked
    await query('UPDATE links SET clicks = clicks + 1, last_clicked = now() WHERE code=$1', [code]);

    // 302 redirect
    res.writeHead(302, { Location: target });
    res.end();
    return { props: {} };
  } catch (e) {
    console.error(e);
    res.statusCode = 500;
    return { props: { error: true } };
  }
}

export default function RedirectPage() {
  return (
    <div style={{padding:20}}>
      <h2>Redirecting...</h2>
      <p>If you see this message, the redirect failed.</p>
    </div>
  );
}


---

Stats page /pages/code/[code].js

// pages/code/[code].js
import { useEffect, useState } from 'react';
import Head from 'next/head';
import Link from 'next/link';

export default function Stats({ codeFromUrl }) {
  const [link, setLink] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error,setError] = useState(null);

  useEffect(()=>{
    async function load(){
      try{
        const r = await fetch(`/api/links/${codeFromUrl}`);
        if (!r.ok) {
          const body = await r.json().catch(()=>({}));
          throw new Error(body.error || 'failed');
        }
        const data = await r.json();
        setLink(data.link);
      } catch(e){
        setError(e.message);
      } finally {
        setLoading(false);
      }
    }
    load();
  },[codeFromUrl]);

  if (loading) return <div className="container"><div className="small">Loading…</div></div>;
  if (error) return <div className="container"><div className="small">Error: {error}</div></div>;

  return (
    <div className="container">
      <div className="header"><div className="h1">Stats for {link.code}</div>
        <div><Link href="/"><a className="button secondary">Back</a></Link></div>
      </div>

      <table className="table">
        <tbody>
          <tr><th>Short code</th><td>{link.code}</td></tr>
          <tr><th>Target URL</th><td><a href={link.target_url} target="_blank" rel="noreferrer">{link.target_url}</a></td></tr>
          <tr><th>Total clicks</th><td>{link.clicks}</td></tr>
          <tr><th>Last clicked</th><td>{link.last_clicked ? new Date(link.last_clicked).toLocaleString() : 'Never'}</td></tr>
          <tr><th>Created</th><td>{new Date(link.created_at).toLocaleString()}</td></tr>
        </tbody>
      </table>
    </div>
  );
}

export function getServerSideProps({ params }) {
  return { props: { codeFromUrl: params.code } };
}


---

Dashboard /pages/index.js

// pages/index.js
import { useEffect, useState } from 'react';
import Link from 'next/link';

export default function Dashboard() {
  const [links, setLinks] = useState([]);
  const [targetUrl, setTargetUrl] = useState('');
  const [code, setCode] = useState('');
  const [loading, setLoading] = useState(false);
  const [creating, setCreating] = useState(false);
  const [message, setMessage] = useState(null);

  async function load(){
    setLoading(true);
    const r = await fetch('/api/links');
    const data = await r.json();
    setLinks(data.links || []);
    setLoading(false);
  }

  useEffect(()=>{ load() }, []);

  async function handleCreate(e){
    e.preventDefault();
    setCreating(true);
    setMessage(null);
    try{
      const r = await fetch('/api/links', {
        method:'POST', headers:{'Content-Type':'application/json'},
        body: JSON.stringify({ targetUrl, code: code || undefined })
      });
      if (!r.ok) {
        const body = await r.json().catch(()=>({}));
        throw new Error(body.error || body.message || 'create_failed');
      }
      const body = await r.json();
      setMessage({ type: 'success', text: `Created: ${body.shortUrl}` });
      setTargetUrl(''); setCode('');
      await load();
    }catch(e){
      setMessage({ type: 'error', text: e.message });
    } finally {
      setCreating(false);
    }
  }

  async function handleDelete(c){
    if (!confirm(`Delete ${c}? This cannot be undone.`)) return;
    try{
      const r = await fetch(`/api/links/${c}`, { method:'DELETE' });
      if (!r.ok) throw new Error('delete_failed');
      await load();
    }catch(e){
      alert('Delete failed');
    }
  }

  function copy(text){
    navigator.clipboard?.writeText(text).then(()=> alert('Copied to clipboard'));
  }

  return (
    <div className="container">
      <div className="header">
        <div className="h1">TinyLink — Dashboard</div>
        <div className="small">Public — no auth</div>
      </div>

      <form onSubmit={handleCreate} className="form-row">
        <input className="input" placeholder="https://example.com/very/long/url" value={targetUrl} onChange={e=>setTargetUrl(e.target.value)} />
        <input className="input" placeholder="Custom code (optional, 6-8 alphanum)" value={code} onChange={e=>setCode(e.target.value)} />
        <button className="button" disabled={creating}>{creating ? 'Creating...' : 'Create'}</button>
      </form>

      {message && <div className="small" style={{marginBottom:12,color: message.type==='error'?'crimson':'green'}}>{message.text}</div>}

      {loading ? <div className="small">Loading links…</div> :
        <table className="table">
          <thead>
            <tr>
              <th>Short code</th>
              <th>Target URL</th>
              <th>Total clicks</th>
              <th>Last clicked</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {links.length === 0 && <tr><td colSpan="5" className="small">No links yet</td></tr>}
            {links.map(l => (
              <tr key={l.code}>
                <td><Link href={`/code/${l.code}`}><a className="badge">{l.code}</a></Link></td>
                <td style={{maxWidth:400, overflow:'hidden', textOverflow:'ellipsis', whiteSpace:'nowrap'}} title={l.target_url}><a href={l.target_url} target="_blank" rel="noreferrer">{l.target_url}</a></td>
                <td>{l.clicks}</td>
                <td className="small">{l.last_clicked ? new Date(l.last_clicked).toLocaleString() : '—'}</td>
                <td className="actions">
                  <button className="copy-btn" onClick={()=>copy(`${process.env.NEXT_PUBLIC_BASE_URL || window.location.origin}/${l.code}`)}>Copy</button>
                  <button className="button secondary" onClick={()=>location.href=`/code/${l.code}`}>View</button>
                  <button className="button" style={{background:'#e55'}} onClick={()=>handleDelete(l.code)}>Delete</button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      }
    </div>
  );
}
