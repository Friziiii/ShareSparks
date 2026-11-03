/**
 * =============================================================================
 *  ShareSpark  |  worker.js
 *  Premium, URL-based video trimming and link-sharing platform
 *  Runtime: Cloudflare Workers (single file, zero dependencies, zero bindings)
 * =============================================================================
 *
 *  HOW TO DEPLOY (GitHub integration with wrangler.json, or Cloudflare dashboard)
 *  1. Workers & Pages  ->  Create  ->  Create Worker  ->  Deploy (any template)
 *  2. Click "Edit code", select everything, paste this entire file, press Deploy.
 *  3. Open your *.workers.dev URL (or attach a custom domain under Settings).
 *
 *  HOW IT WORKS
 *  - The Worker serves ONE HTML document from a template literal.
 *  - The Worker inspects the query string on every request:
 *
 *      /                                  -> CONTEXT A: Builder screen
 *      /?mp4=<url>&start=<s>&end=<s>      -> CONTEXT B: Viewer wrapper
 *
 *    The decision is made on the server, validated on the server, and handed to
 *    the browser as a JSON config object, so the Builder is skipped completely
 *    for share links.
 *  - Share links are fully self-contained (video URL + start + end live in the
 *    URL), so the platform is stateless: no database, no KV, no storage.
 *
 *  NOTE: The client code inside the HTML template intentionally contains no
 *  backticks, no dollar-brace sequences and no backslashes, because the whole
 *  page lives inside a JavaScript template literal.
 * =============================================================================
 */

'use strict';

/* -----------------------------------------------------------------------------
 * SECTION 1  |  SERVER CONSTANTS
 * -------------------------------------------------------------------------- */

/** Simulated detected video length (seconds). Mirrors the client constant. */
const MAX_VIDEO_SECONDS = 300;

/** Upper bound for the encoded media URL accepted in a share link. */
const MAX_MEDIA_URL_LENGTH = 2048;

/** Headers attached to every response for baseline hardening. */
const SECURITY_HEADERS = {
  'X-Content-Type-Options': 'nosniff',
  'Referrer-Policy': 'strict-origin-when-cross-origin',
  'Permissions-Policy': 'camera=(), microphone=(), geolocation=()',
};

/** Minimal bolt favicon, served at /favicon.ico and /favicon.svg. */
const FAVICON_SVG =
  '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24">' +
  '<path fill="#7C3AED" d="M13 2 4 14h6l-1 8 9-12h-6z"/></svg>';

/* -----------------------------------------------------------------------------
 * SECTION 2  |  FETCH HANDLER (ENTRY POINT, ES MODULE SYNTAX)
 * -------------------------------------------------------------------------- */

export default {
  /**
   * Module-syntax fetch handler (required for Workers Static Assets deployments).
   * @param {Request} request
   * @param {object} env  Bindings (unused: ShareSpark needs none).
   * @param {object} ctx  Execution context (unused).
   * @returns {Promise<Response>}
   */
  async fetch(request, env, ctx) {
    try {
      return await handleRequest(request);
    } catch (error) {
      // Never leak internals to the visitor; log for the Cloudflare logs view.
      console.error('ShareSpark unhandled error:', error && error.stack ? error.stack : error);
      return new Response('Internal Server Error', {
        status: 500,
        headers: { 'Content-Type': 'text/plain; charset=utf-8', ...SECURITY_HEADERS },
      });
    }
  },
};

/**
 * Routes an incoming request.
 * @param {Request} request
 * @returns {Promise<Response>}
 */
async function handleRequest(request) {
  const url = new URL(request.url);
  const method = request.method.toUpperCase();

  // Only safe, read-only methods are meaningful for this application.
  if (method !== 'GET' && method !== 'HEAD') {
    return new Response('Method Not Allowed', {
      status: 405,
      headers: {
        Allow: 'GET, HEAD',
        'Content-Type': 'text/plain; charset=utf-8',
        ...SECURITY_HEADERS,
      },
    });
  }

  switch (url.pathname) {
    case '/':
      return renderApplication(url, method);

    case '/favicon.ico':
    case '/favicon.svg':
      return new Response(method === 'HEAD' ? null : FAVICON_SVG, {
        status: 200,
        headers: {
          'Content-Type': 'image/svg+xml; charset=utf-8',
          'Cache-Control': 'public, max-age=86400',
          ...SECURITY_HEADERS,
        },
      });

    case '/robots.txt':
      return new Response(method === 'HEAD' ? null : 'User-agent: *\nAllow: /\n', {
        status: 200,
        headers: {
          'Content-Type': 'text/plain; charset=utf-8',
          'Cache-Control': 'public, max-age=86400',
          ...SECURITY_HEADERS,
        },
      });

    case '/healthz':
      return new Response(method === 'HEAD' ? null : JSON.stringify({ status: 'ok', app: 'ShareSpark' }), {
        status: 200,
        headers: {
          'Content-Type': 'application/json; charset=utf-8',
          'Cache-Control': 'no-store',
          ...SECURITY_HEADERS,
        },
      });

    default:
      // Unknown paths fall back to the root, preserving any share parameters.
      return Response.redirect(new URL('/' + url.search, url.origin).toString(), 302);
  }
}

/* -----------------------------------------------------------------------------
 * SECTION 3  |  CONTEXT RESOLUTION (BUILDER vs. VIEWER)
 * -------------------------------------------------------------------------- */

/**
 * Parses and validates the query string.
 *
 * Viewer context is chosen automatically when mp4, start AND end are all
 * present. Anything else is the Builder.
 *
 * @param {URLSearchParams} params
 * @returns {{mode:'builder'} | {mode:'viewer', mp4:string, start:(number|null), end:(number|null), error:(string|null)}}
 */
function resolveContext(params) {
  const isViewer = params.has('mp4') && params.has('start') && params.has('end');
  if (!isViewer) {
    return { mode: 'builder' };
  }

  const mp4 = (params.get('mp4') || '').trim();
  const start = parseStrictNumber(params.get('start'));
  const end = parseStrictNumber(params.get('end'));

  const invalid = (message) => ({ mode: 'viewer', mp4: '', start: null, end: null, error: message });

  if (!isHttpUrl(mp4) || mp4.length > MAX_MEDIA_URL_LENGTH) {
    return invalid('This clip link does not contain a valid video address.');
  }
  if (start === null || end === null) {
    return invalid('This clip link has unreadable start or end markers.');
  }
  if (start < 0 || start > MAX_VIDEO_SECONDS || end < 0 || end > MAX_VIDEO_SECONDS) {
    return invalid('This clip link has time markers outside the allowed 0 to ' + MAX_VIDEO_SECONDS + ' second range.');
  }
  if (end <= start) {
    return invalid('This clip link is invalid because its end marker is not after its start marker.');
  }

  return { mode: 'viewer', mp4, start, end, error: null };
}

/** Converts a string to a finite number, or null when it is not one. */
function parseStrictNumber(value) {
  if (value === null || value === undefined || String(value).trim() === '') return null;
  const parsed = Number(value);
  return Number.isFinite(parsed) ? parsed : null;
}

/** True only for absolute http(s) URLs. Blocks javascript:, data:, file:, etc. */
function isHttpUrl(value) {
  if (!value) return false;
  try {
    const parsed = new URL(value);
    return parsed.protocol === 'http:' || parsed.protocol === 'https:';
  } catch (_) {
    return false;
  }
}

/* -----------------------------------------------------------------------------
 * SECTION 4  |  RESPONSE COMPOSITION
 * -------------------------------------------------------------------------- */

/**
 * Builds the HTML response for the root route.
 * @param {URL} url
 * @param {string} method
 * @returns {Response}
 */
function renderApplication(url, method) {
  const context = resolveContext(url.searchParams);
  const html = composeDocument(context);
  const hasError = context.mode === 'viewer' && Boolean(context.error);

  return new Response(method === 'HEAD' ? null : html, {
    status: hasError ? 400 : 200,
    headers: {
      'Content-Type': 'text/html; charset=utf-8',
      'Cache-Control': hasError ? 'no-store' : 'public, max-age=300',
      ...SECURITY_HEADERS,
    },
  });
}

/**
 * Injects the per-request title, social meta tags and client config into the
 * HTML template in a single pass (inserted content is never re-scanned).
 */
function composeDocument(context) {
  const meta = buildDocumentMeta(context);

  const clientConfig =
    context.mode === 'viewer'
      ? { mode: 'viewer', mp4: context.mp4, start: context.start, end: context.end, error: context.error }
      : { mode: 'builder' };

  const replacements = {
    DOCUMENT_TITLE: escapeHtml(meta.title),
    DOCUMENT_META: meta.tags,
    SHARESPARK_CONFIG: serializeForInlineScript(clientConfig),
  };

  return HTML_TEMPLATE.replace(/__(DOCUMENT_TITLE|DOCUMENT_META|SHARESPARK_CONFIG)__/g, (_, key) => replacements[key]);
}

/** Title, description and Open Graph tags (so shared links unfurl nicely). */
function buildDocumentMeta(context) {
  let title = 'ShareSpark \u2014 Trim & Share Video Clips';
  let description =
    'Paste a video link, choose a start and end time, and share a clip that plays exactly the moment you mean.';
  let robots = '';

  if (context.mode === 'viewer' && !context.error) {
    const length = context.end - context.start;
    title = 'ShareSpark clip \u00B7 ' + formatClock(context.start) + ' to ' + formatClock(context.end);
    description = 'Watch a hand-picked ' + formatClock(length) + ' moment, trimmed and shared with ShareSpark.';
    robots = '<meta name="robots" content="noindex, nofollow">';
  } else if (context.mode === 'viewer') {
    title = 'ShareSpark \u2014 Clip unavailable';
    robots = '<meta name="robots" content="noindex, nofollow">';
  }

  const tags = [
    '<meta name="description" content="' + escapeHtml(description) + '">',
    '<meta property="og:site_name" content="ShareSpark">',
    '<meta property="og:type" content="website">',
    '<meta property="og:title" content="' + escapeHtml(title) + '">',
    '<meta property="og:description" content="' + escapeHtml(description) + '">',
    '<meta name="twitter:card" content="summary">',
    '<meta name="twitter:title" content="' + escapeHtml(title) + '">',
    '<meta name="twitter:description" content="' + escapeHtml(description) + '">',
    robots,
  ]
    .filter(Boolean)
    .join('\n');

  return { title, tags };
}

/** m:ss formatter used for server-side meta text. */
function formatClock(totalSeconds) {
  const safe = Math.max(0, Number(totalSeconds) || 0);
  let minutes = Math.floor(safe / 60);
  let seconds = Math.round((safe - minutes * 60) * 10) / 10;
  if (seconds >= 60) {
    minutes += 1;
    seconds = 0;
  }
  return minutes + ':' + (seconds < 10 ? '0' : '') + String(seconds);
}

/** Escapes text for safe use inside HTML text nodes and attribute values. */
function escapeHtml(value) {
  return String(value)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;');
}

/**
 * JSON.stringify output that is safe to embed inside an inline <script>.
 * Prevents "</script>" breakouts and legacy line-separator issues.
 */
function serializeForInlineScript(value) {
  return JSON.stringify(value)
    .replace(/</g, '\\u003c')
    .replace(/>/g, '\\u003e')
    .replace(/&/g, '\\u0026')
    .replace(/\u2028/g, '\\u2028')
    .replace(/\u2029/g, '\\u2029');
}

/* -----------------------------------------------------------------------------
 * SECTION 5  |  FRONTEND SINGLE-PAGE APPLICATION (HTML TEMPLATE LITERAL)
 * -------------------------------------------------------------------------- */

const HTML_TEMPLATE = `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
  <meta name="theme-color" content="#FFFFFF">
  <title>__DOCUMENT_TITLE__</title>
  __DOCUMENT_META__
  <link rel="icon" href="/favicon.svg" type="image/svg+xml">

  <!-- Typography: Inter with a robust system-ui fallback stack -->
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet">

  <!-- Icons: FontAwesome via CDN -->
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.2/css/all.min.css">

  <!-- Styling: Tailwind CSS via CDN (pinned version) -->
  <script src="https://cdn.tailwindcss.com/3.4.17"></script>
  <script>
    // Brand palette exposed to Tailwind utilities (text-brand-600, bg-brand-50, ...).
    try {
      tailwind.config = {
        theme: {
          extend: {
            fontFamily: { sans: ['Inter', 'system-ui', '-apple-system', 'Segoe UI', 'Roboto', 'Helvetica Neue', 'Arial', 'sans-serif'] },
            colors: {
              brand: {
                50: '#F5F3FF', 100: '#EDE9FE', 200: '#DDD6FE', 300: '#C4B5FD', 400: '#A78BFA',
                500: '#7C3AED', 600: '#6D28D9', 700: '#5B21B6', 800: '#4C1D95', 900: '#3B0764'
              }
            }
          }
        }
      };
    } catch (configError) { /* The hand-written CSS below keeps the UI usable even without Tailwind. */ }
  </script>

  <style>
    /* ===================================================================
       DESIGN TOKENS
       =================================================================== */
    :root {
      --brand-50: #F5F3FF;
      --brand-100: #EDE9FE;
      --brand-200: #DDD6FE;
      --brand-500: #7C3AED;
      --brand-600: #6D28D9;
      --brand-700: #5B21B6;
      --ink: #0F172A;
      --muted: #64748B;
      --danger: #DC2626;
      --success: #16A34A;
      --ease-out: cubic-bezier(0.22, 1, 0.36, 1);
    }

    html { -webkit-text-size-adjust: 100%; scroll-behavior: smooth; }

    /* Pure, crisp solid white canvas. */
    body {
      background: #FFFFFF;
      color: var(--ink);
      font-family: 'Inter', system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
      -webkit-font-smoothing: antialiased;
      -moz-osx-font-smoothing: grayscale;
    }
    body.modal-open { overflow: hidden; }

    /* Single visibility switch used by all JavaScript (independent of Tailwind load order). */
    .is-hidden { display: none !important; }
    .sr-only {
      position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px;
      overflow: hidden; clip: rect(0, 0, 0, 0); white-space: nowrap; border: 0;
    }

    /* ===================================================================
       BUTTONS: every button is a fully rounded pill with smooth micro-transitions
       =================================================================== */
    .btn {
      display: inline-flex; align-items: center; justify-content: center; gap: 0.6rem;
      border: 1.5px solid transparent; border-radius: 9999px;
      font-family: inherit; font-weight: 600; line-height: 1; text-decoration: none;
      white-space: nowrap; cursor: pointer; user-select: none; -webkit-tap-highlight-color: transparent;
      transition: transform 0.28s var(--ease-out), box-shadow 0.28s ease, background-color 0.25s ease,
                  color 0.25s ease, border-color 0.25s ease, opacity 0.25s ease;
    }
    .btn:hover { transform: translateY(-2px); }
    .btn:active { transform: translateY(0) scale(0.97); transition-duration: 0.1s; }
    .btn:focus-visible { outline: none; box-shadow: 0 0 0 4px rgba(124, 58, 237, 0.32); }
    .btn[disabled] { opacity: 0.5; cursor: not-allowed; transform: none; }
    .btn i { transition: transform 0.28s var(--ease-out); }
    .btn:hover i { transform: scale(1.12); }

    .btn-lg { padding: 1.1rem 2.2rem; font-size: 1.02rem; }
    .btn-md { padding: 0.85rem 1.6rem; font-size: 0.95rem; }
    .btn-sm { padding: 0.55rem 1.1rem; font-size: 0.82rem; }
    .btn-icon { width: 2.6rem; height: 2.6rem; padding: 0; }
    .btn-block { width: 100%; }

    .btn-primary { background: var(--brand-500); color: #FFFFFF; box-shadow: 0 12px 26px -12px rgba(109, 40, 217, 0.7); }
    .btn-primary:hover { background: var(--brand-600); box-shadow: 0 18px 34px -14px rgba(109, 40, 217, 0.8); }
    .btn-primary:active { background: var(--brand-700); }

    .btn-secondary { background: #FFFFFF; color: var(--brand-600); border-color: var(--brand-200); }
    .btn-secondary:hover { background: var(--brand-50); border-color: var(--brand-500); }

    .btn-ghost { background: transparent; color: var(--muted); }
    .btn-ghost:hover { background: var(--brand-50); color: var(--brand-600); }

    .btn-success { background: var(--success); color: #FFFFFF; box-shadow: 0 12px 26px -12px rgba(22, 163, 74, 0.7); }
    .btn-success:hover { background: #15803D; }

    /* ===================================================================
       FORM CONTROLS
       =================================================================== */
    .text-input, .time-input {
      width: 100%; background: #FFFFFF; color: var(--ink); font-family: inherit;
      border: 2px solid #E2E8F0; border-radius: 9999px;
      transition: border-color 0.2s ease, box-shadow 0.2s ease, background-color 0.2s ease;
    }
    .text-input { padding: 1.1rem 1.5rem 1.1rem 3.3rem; font-size: 1rem; }
    .time-input { padding: 1rem 1.25rem; font-size: 1.05rem; font-weight: 600; text-align: center; font-variant-numeric: tabular-nums; }
    .text-input::placeholder, .time-input::placeholder { color: #94A3B8; font-weight: 400; }
    .text-input:hover:not(:disabled), .time-input:hover:not(:disabled) { border-color: #C4B5FD; }
    .text-input:focus, .time-input:focus {
      outline: none; border-color: var(--brand-500); box-shadow: 0 0 0 5px rgba(124, 58, 237, 0.18);
    }
    .text-input.is-invalid, .time-input.is-invalid { border-color: var(--danger); box-shadow: 0 0 0 4px rgba(220, 38, 38, 0.12); }
    .time-input:disabled {
      background: #F1F5F9; color: #94A3B8; border-color: #E2E8F0; cursor: not-allowed; -webkit-text-fill-color: #94A3B8; opacity: 1;
    }
    .input-icon { position: absolute; left: 1.4rem; top: 50%; transform: translateY(-50%); color: var(--brand-500); pointer-events: none; }
    .field-lock { position: absolute; right: 1.1rem; top: 50%; transform: translateY(-50%); color: #94A3B8; pointer-events: none; font-size: 0.8rem; }
    .field-label { display: block; margin: 0 0 0.5rem 0.9rem; font-size: 0.82rem; font-weight: 600; color: #334155; }

    .inline-error {
      display: flex; align-items: flex-start; justify-content: center; gap: 0.5rem;
      color: var(--danger); font-size: 0.9rem; font-weight: 500; line-height: 1.45; text-align: center;
    }

    /* ===================================================================
       LAYOUT PIECES
       =================================================================== */
    .eyebrow {
      display: inline-flex; align-items: center; gap: 0.5rem; padding: 0.42rem 0.95rem; border-radius: 9999px;
      background: var(--brand-50); color: var(--brand-600);
      font-size: 0.72rem; font-weight: 700; letter-spacing: 0.1em; text-transform: uppercase;
    }
    .brand-link { text-decoration: none; border-radius: 9999px; transition: opacity 0.25s ease, transform 0.28s var(--ease-out); }
    .brand-link:hover { opacity: 0.85; transform: translateY(-1px); }
    .brand-link:focus-visible { outline: none; box-shadow: 0 0 0 4px rgba(124, 58, 237, 0.32); }

    .video-shell {
      position: relative; overflow: hidden; aspect-ratio: 16 / 9; background: #0B0B12;
      border: 1px solid var(--brand-100); border-radius: 1.75rem;
      box-shadow: 0 30px 70px -34px rgba(76, 29, 149, 0.55);
    }
    .video-shell video { display: block; width: 100%; height: 100%; object-fit: contain; background: #0B0B12; }
    .video-overlay, .video-fallback {
      position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center;
      gap: 0.75rem; padding: 1.5rem; text-align: center;
    }
    .video-overlay { background: rgba(11, 11, 18, 0.55); backdrop-filter: blur(3px); }
    .video-fallback { background: linear-gradient(160deg, #1E1B4B 0%, #0B0B12 100%); color: #E9E5FF; }

    .clip-track { height: 0.5rem; background: var(--brand-100); border-radius: 9999px; overflow: hidden; }
    .clip-fill { height: 100%; width: 0%; background: linear-gradient(90deg, var(--brand-500), #A78BFA); border-radius: 9999px; transition: width 0.15s linear; }

    .state-panel { animation: fadeUp 0.55s var(--ease-out) both; }
    .link-box {
      max-height: 7.5rem; overflow: auto; padding: 0.95rem 1.2rem; background: #F8FAFC; border: 1.5px solid #E2E8F0;
      border-radius: 1.25rem; font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
      font-size: 0.8rem; line-height: 1.55; color: #1E293B; word-break: break-all; text-align: left; cursor: text;
    }
    .link-box:focus { outline: none; border-color: var(--brand-500); box-shadow: 0 0 0 4px rgba(124, 58, 237, 0.18); }

    /* ===================================================================
       LOADERS
       =================================================================== */
    .spinner {
      width: 3.4rem; height: 3.4rem; border-radius: 50%;
      border: 4px solid var(--brand-100); border-top-color: var(--brand-500);
      animation: spin 0.85s linear infinite;
    }
    .spinner-orbit {
      width: 3.8rem; height: 3.8rem; border-radius: 50%;
      background: conic-gradient(from 0deg, rgba(124, 58, 237, 0) 0%, #7C3AED 100%);
      -webkit-mask: radial-gradient(farthest-side, transparent calc(100% - 7px), #000 calc(100% - 6px));
              mask: radial-gradient(farthest-side, transparent calc(100% - 7px), #000 calc(100% - 6px));
      animation: spin 0.95s linear infinite;
    }
    .pulse-text { animation: pulseText 1.6s ease-in-out infinite; }

    /* Success checkmark: draws itself when its panel becomes visible. */
    .check-badge {
      display: flex; align-items: center; justify-content: center; width: 4.4rem; height: 4.4rem; flex: none;
      background: #F0FDF4; border-radius: 50%; animation: popIn 0.5s var(--ease-out) both;
    }
    .check-badge svg { width: 3.2rem; height: 3.2rem; }
    .check-circle { fill: none; stroke: var(--success); stroke-width: 2.5; stroke-dasharray: 151; stroke-dashoffset: 151; animation: draw 0.6s ease forwards; }
    .check-mark { fill: none; stroke: var(--success); stroke-width: 3.5; stroke-linecap: round; stroke-linejoin: round; stroke-dasharray: 40; stroke-dashoffset: 40; animation: draw 0.4s 0.45s ease forwards; }

    /* ===================================================================
       MODALS
       =================================================================== */
    .modal { position: fixed; inset: 0; z-index: 50; display: flex; align-items: center; justify-content: center; padding: 1rem; }
    .modal-backdrop { position: absolute; inset: 0; background: rgba(15, 23, 42, 0.55); backdrop-filter: blur(6px); -webkit-backdrop-filter: blur(6px); animation: fadeIn 0.25s ease both; }
    .modal-card {
      position: relative; background: #FFFFFF; border-radius: 2rem;
      box-shadow: 0 40px 90px -30px rgba(15, 23, 42, 0.55); animation: popIn 0.35s var(--ease-out) both;
    }

    /* Legal overlay: full-width, high-contrast reading surface. */
    .legal-card {
      position: relative; display: flex; flex-direction: column; width: 100%; max-width: 72rem; max-height: 92vh;
      background: #FFFFFF; border-radius: 2rem; overflow: hidden;
      box-shadow: 0 40px 90px -30px rgba(15, 23, 42, 0.6); animation: popIn 0.35s var(--ease-out) both;
    }
    .legal-head { display: flex; align-items: flex-start; justify-content: space-between; gap: 1rem; padding: 1.75rem 2rem 1.25rem; border-bottom: 1px solid #E2E8F0; }
    .legal-kicker { font-size: 0.72rem; font-weight: 700; letter-spacing: 0.12em; text-transform: uppercase; color: var(--brand-600); }
    .legal-title { margin-top: 0.3rem; font-size: 1.8rem; font-weight: 800; letter-spacing: -0.02em; color: #020617; }
    .legal-updated { margin-top: 0.35rem; font-size: 0.85rem; color: #475569; }
    .legal-body { flex: 1; overflow-y: auto; padding: 1.75rem 2rem 2rem; color: #1E293B; font-size: 0.98rem; line-height: 1.75; }
    .legal-body:focus { outline: none; }
    .legal-body p { margin: 0 0 0.9rem; }
    .legal-body ul { margin: 0 0 1rem; padding-left: 1.4rem; list-style: disc; }
    .legal-body li { margin-bottom: 0.4rem; }
    .legal-body h3 { display: flex; align-items: center; gap: 0.75rem; margin: 2rem 0 0.75rem; font-size: 1.15rem; font-weight: 700; color: #020617; }
    .legal-num {
      display: inline-flex; align-items: center; justify-content: center; flex: none; min-width: 2rem; height: 2rem; padding: 0 0.5rem;
      border-radius: 9999px; background: var(--brand-600); color: #FFFFFF; font-size: 0.85rem; font-weight: 700;
    }
    .legal-foot { display: flex; justify-content: center; padding: 1.1rem 2rem 1.5rem; border-top: 1px solid #E2E8F0; background: #FFFFFF; }

    /* ===================================================================
       KEYFRAMES
       =================================================================== */
    @keyframes spin { to { transform: rotate(360deg); } }
    @keyframes pulseText { 0%, 100% { opacity: 0.4; } 50% { opacity: 1; } }
    @keyframes fadeUp { from { opacity: 0; transform: translateY(14px); } to { opacity: 1; transform: translateY(0); } }
    @keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }
    @keyframes popIn { from { opacity: 0; transform: translateY(12px) scale(0.96); } to { opacity: 1; transform: translateY(0) scale(1); } }
    @keyframes draw { to { stroke-dashoffset: 0; } }

    @media (max-width: 640px) {
      .legal-card { max-height: 100vh; height: 100vh; border-radius: 0; }
      #legal-modal { padding: 0; }
      .legal-head, .legal-body, .legal-foot { padding-left: 1.25rem; padding-right: 1.25rem; }
      .legal-title { font-size: 1.45rem; }
    }
    @media (prefers-reduced-motion: reduce) {
      *, *::before, *::after { animation-duration: 0.01ms !important; animation-iteration-count: 1 !important; transition-duration: 0.01ms !important; scroll-behavior: auto !important; }
    }
  </style>
</head>

<body class="min-h-screen flex flex-col">

  <!-- ===================================================================
       HEADER: logo pinned to the top-left corner
       =================================================================== -->
  <header class="w-full flex items-center justify-start px-5 sm:px-10 pt-5 sm:pt-7">
    <a href="/" class="brand-link inline-flex items-center gap-2.5 pr-3" aria-label="ShareSpark home">
      <svg xmlns="http://www.w3.org/2000/svg" width="23" height="38" viewBox="0 0 359 600" preserveAspectRatio="xMidYMid meet" aria-hidden="true" focusable="false">
        <g transform="translate(0,600) scale(0.1,-0.1)" fill="#7C3AED" stroke="none">
          <path d="M2941 5913 c-16 -32 -104 -222 -196 -423 -92 -201 -192 -416 -221 -479 -30 -62 -54 -115 -54 -117 0 -1 -18 -42 -41 -91 -86 -186 -423 -911 -451 -973 -74 -162 -226 -498 -237 -524 -7 -17 -16 -25 -21 -20 -4 5 -51 106 -103 224 -88 198 -519 1147 -600 1320 -19 41 -47 102 -61 134 -15 33 -28 57 -31 54 -5 -5 1 -20 108 -283 35 -88 108 -268 162 -400 138 -342 171 -422 317 -773 72 -172 128 -316 124 -319 -12 -12 -73 22 -121 67 -112 105 -388 344 -379 328 5 -9 76 -87 158 -173 82 -86 146 -158 144 -161 -3 -2 -106 25 -229 62 -123 36 -260 76 -304 89 -44 12 -262 75 -484 139 -222 64 -405 115 -407 113 -4 -3 1092 -567 1398 -719 59 -29 107 -57 107 -63 0 -5 -57 -37 -127 -70 -216 -102 -452 -218 -448 -222 3 -4 142 45 256 90 177 70 331 127 341 127 27 0 -7 -58 -79 -135 -75 -81 -507 -543 -747 -800 -218 -234 -408 -439 -434 -469 -91 -104 87 68 600 579 319 317 582 575 584 573 2 -2 -190 -581 -426 -1287 -236 -707 -427 -1286 -426 -1288 4 -3 92 183 287 607 65 140 229 496 365 790 354 764 548 1188 584 1274 7 17 16 25 21 20 4 -5 51 -106 103 -224 83 -187 349 -774 610 -1342 126 -275 112 -221 -35 142 -42 102 -120 293 -173 425 -119 294 -282 691 -364 885 -34 80 -61 148 -61 153 0 4 6 7 13 7 29 -2 83 -42 209 -155 222 -200 288 -257 288 -250 0 3 -70 81 -156 171 -86 91 -154 167 -152 170 5 4 221 -57 638 -181 237 -70 779 -225 787 -225 4 0 -23 17 -62 38 -116 63 -1254 645 -1373 701 -40 19 -72 40 -71 45 0 6 118 67 262 136 266 127 346 168 297 155 -14 -4 -148 -54 -297 -111 -150 -57 -277 -104 -283 -104 -5 0 -10 9 -10 20 0 22 18 41 434 486 159 170 344 368 410 439 159 171 238 255 312 335 90 96 131 144 117 139 -7 -2 -276 -267 -598 -588 -322 -321 -588 -582 -590 -580 -2 3 189 581 425 1287 236 705 427 1282 425 1282 -3 0 -18 -26 -34 -57z"/>
        </g>
      </svg>
      <span class="text-[1.4rem] font-extrabold tracking-tight" style="color:#6D28D9">ShareSpark</span>
    </a>
  </header>

  <!-- ===================================================================
       MAIN: both contexts live here; JavaScript reveals exactly one
       =================================================================== -->
  <main class="flex-1 w-full flex items-center justify-center px-5 sm:px-8 py-10 sm:py-14">

    <!-- ---------------------------------------------------------------
         CONTEXT A: BUILDER
         --------------------------------------------------------------- -->
    <div id="view-builder" class="is-hidden w-full max-w-2xl mx-auto">

      <!-- STATE 1: Initial interface -->
      <section id="state-input" class="state-panel" aria-labelledby="hero-title">
        <div class="text-center mb-10">
          <span class="eyebrow"><i class="fa-solid fa-bolt" aria-hidden="true"></i> Clip &middot; Trim &middot; Share</span>
          <h1 id="hero-title" class="mt-5 text-4xl sm:text-5xl font-extrabold tracking-tight text-slate-900" style="line-height:1.1">
            Trim any video,<br><span style="color:#7C3AED">share the spark.</span>
          </h1>
          <p class="mt-5 mx-auto max-w-md text-base sm:text-lg text-slate-500">
            Paste a video link, pick your start and end, and send a clip that plays exactly the moment you mean.
          </p>
        </div>

        <label for="video-link" class="sr-only">Video link</label>
        <div class="relative">
          <span class="input-icon" aria-hidden="true"><i class="fa-solid fa-link"></i></span>
          <input id="video-link" class="text-input" type="text" inputmode="url" autocomplete="off" autocapitalize="off" spellcheck="false"
                 placeholder="Paste social media video link here...">
        </div>
        <p id="link-error" class="inline-error is-hidden mt-3" role="alert"></p>

        <div class="mt-5">
          <button id="process-btn" type="button" class="btn btn-primary btn-lg btn-block">
            <i class="fa-solid fa-wand-magic-sparkles" aria-hidden="true"></i><span>Process Video</span>
          </button>
        </div>
      </section>

      <!-- STATE 2: Extraction loader -->
      <section id="state-extract" class="state-panel is-hidden text-center py-20" role="status" aria-live="polite">
        <div class="spinner mx-auto" aria-hidden="true"></div>
        <p class="mt-8 text-lg font-semibold pulse-text" style="color:#6D28D9">Analyzing media source and duration...</p>
        <p class="mt-2 text-sm text-slate-400">Hang tight, this only takes a moment.</p>
      </section>

      <!-- STATE 3: Preview, timestamp controls and validation -->
      <section id="state-controls" class="state-panel is-hidden" aria-labelledby="controls-title">
        <div class="text-center mb-7">
          <h2 id="controls-title" class="text-2xl sm:text-3xl font-bold tracking-tight text-slate-900">Choose your clip window</h2>
          <p class="mt-2 text-slate-500">Set where your clip starts and where it stops.</p>
        </div>

        <div class="video-shell">
          <video id="preview-video" controls playsinline preload="metadata"></video>
          <div id="preview-fallback" class="video-fallback is-hidden">
            <i class="fa-solid fa-film text-4xl" style="color:#A78BFA" aria-hidden="true"></i>
            <p class="text-lg font-semibold">Preview unavailable</p>
            <p class="text-sm max-w-sm" style="color:#C4B5FD">
              The browser can&rsquo;t play this link directly. ShareSpark clips work best with direct video file links, for example ones ending in .mp4.
            </p>
          </div>
        </div>
        <p class="mt-3 text-center text-sm text-slate-500">
          <i class="fa-regular fa-clock mr-1.5" style="color:#7C3AED" aria-hidden="true"></i>
          Detected duration: <strong class="text-slate-700">5:00</strong> (300 seconds)
        </p>

        <!-- Two side-by-side number inputs are rendered here by JavaScript -->
        <div id="time-fields" class="mt-7 grid grid-cols-2 gap-3 sm:gap-4"></div>
        <p id="time-error" class="inline-error is-hidden mt-5" role="alert"></p>

        <div class="mt-6">
          <button id="generate-btn" type="button" class="btn btn-primary btn-lg btn-block">
            <i class="fa-solid fa-link" aria-hidden="true"></i><span>Generate Share Link</span>
          </button>
        </div>
        <div class="mt-3 text-center">
          <button id="back-btn" type="button" class="btn btn-ghost btn-sm">
            <i class="fa-solid fa-arrow-left" aria-hidden="true"></i><span>Use a different link</span>
          </button>
        </div>
      </section>

      <!-- STATE 4: Serialization loader (secondary spinner style) -->
      <section id="state-serialize" class="state-panel is-hidden text-center py-20" role="status" aria-live="polite">
        <div class="spinner-orbit mx-auto" aria-hidden="true"></div>
        <p class="mt-8 text-lg font-semibold pulse-text" style="color:#6D28D9">Encoding clip attributes and tokens...</p>
        <p class="mt-2 text-sm text-slate-400">Packing your start and end markers into a link.</p>
      </section>

      <!-- STATE 5: Share trigger -->
      <section id="state-share" class="state-panel is-hidden text-center py-12" aria-live="polite">
        <div class="flex flex-col sm:flex-row items-center justify-center gap-5 sm:gap-6">
          <div class="check-badge" aria-hidden="true">
            <svg viewBox="0 0 52 52"><circle class="check-circle" cx="26" cy="26" r="24"/><path class="check-mark" d="M14 27l8 8 16-17"/></svg>
          </div>
          <button id="share-clip-btn" type="button" class="btn btn-primary btn-lg">
            <i class="fa-solid fa-share-nodes" aria-hidden="true"></i><span>Share Clip</span>
          </button>
        </div>
        <p class="mt-6 text-slate-500">Your clip link is ready. Share it anywhere.</p>
        <div class="mt-8">
          <button id="create-another-btn" type="button" class="btn btn-ghost btn-sm">
            <i class="fa-solid fa-rotate-left" aria-hidden="true"></i><span>Create another clip</span>
          </button>
        </div>
      </section>
    </div>

    <!-- ---------------------------------------------------------------
         CONTEXT B: VIEWER WRAPPER
         --------------------------------------------------------------- -->
    <div id="view-viewer" class="is-hidden w-full max-w-3xl mx-auto">
      <div class="text-center mb-8">
        <span class="eyebrow"><i class="fa-solid fa-bolt" aria-hidden="true"></i> Shared clip</span>
        <h1 class="mt-4 text-3xl sm:text-4xl font-extrabold tracking-tight text-slate-900">
          Watch the <span style="color:#7C3AED">highlight</span>
        </h1>
        <p id="viewer-subtitle" class="mt-3 text-slate-500"></p>
      </div>

      <div id="viewer-stage" class="state-panel">
        <div class="video-shell">
          <video id="viewer-video" controls controlsList="nodownload noplaybackrate" disablePictureInPicture playsinline preload="auto"></video>
          <div id="autoplay-overlay" class="video-overlay is-hidden">
            <button id="autoplay-btn" type="button" class="btn btn-primary btn-lg">
              <i class="fa-solid fa-play" aria-hidden="true"></i><span>Tap to Play Clip</span>
            </button>
          </div>
        </div>

        <div class="mt-5">
          <div class="clip-track" role="progressbar" aria-label="Clip progress" aria-valuemin="0" aria-valuemax="100" aria-valuenow="0" id="clip-track">
            <div id="clip-fill" class="clip-fill"></div>
          </div>
          <div class="mt-2 flex items-center justify-between text-xs font-medium text-slate-500" style="font-variant-numeric:tabular-nums">
            <span id="clip-elapsed">0:00</span>
            <span id="clip-length">0:00</span>
          </div>
        </div>
      </div>

      <div id="viewer-error" class="is-hidden text-center rounded-3xl border border-red-100 bg-red-50 px-6 py-8" role="alert">
        <i class="fa-solid fa-circle-exclamation text-3xl" style="color:#DC2626" aria-hidden="true"></i>
        <p id="viewer-error-title" class="mt-3 text-lg font-bold text-slate-900">This clip can&rsquo;t be played</p>
        <p id="viewer-error-text" class="mt-1 text-sm text-slate-600"></p>
      </div>

      <div class="mt-8">
        <p class="mb-3 text-center text-sm text-slate-500">
          <i class="fa-solid fa-lock mr-1.5" style="color:#7C3AED" aria-hidden="true"></i>
          These timestamps are locked by the creator of this clip.
        </p>
        <div id="viewer-fields" class="grid grid-cols-2 gap-3 sm:gap-4"></div>
      </div>

      <div class="mt-8 flex flex-wrap items-center justify-center gap-3">
        <button id="viewer-toggle" type="button" class="btn btn-primary btn-md">
          <i class="fa-solid fa-play" aria-hidden="true"></i><span>Play</span>
        </button>
        <a href="/" class="btn btn-secondary btn-md">
          <i class="fa-solid fa-scissors" aria-hidden="true"></i><span>Create Your Own Clip</span>
        </a>
      </div>
    </div>
  </main>

  <!-- ===================================================================
       FOOTER: quiet policy triggers, centered at the bottom
       =================================================================== -->
  <footer class="w-full flex items-center justify-center gap-2 px-5 pb-6 pt-2">
    <button type="button" class="btn btn-ghost btn-sm" data-legal="terms" style="font-weight:500">Terms of Service</button>
    <span class="text-slate-300" aria-hidden="true">&middot;</span>
    <button type="button" class="btn btn-ghost btn-sm" data-legal="privacy" style="font-weight:500">Privacy Policy</button>
  </footer>

  <!-- ===================================================================
       SHARE MODAL
       =================================================================== -->
  <div id="share-modal" class="modal is-hidden" role="dialog" aria-modal="true" aria-labelledby="share-modal-title">
    <div class="modal-backdrop"></div>
    <div class="modal-card w-full max-w-md p-7 sm:p-9">
      <button id="share-modal-close" type="button" class="btn btn-ghost btn-icon" style="position:absolute;top:1rem;right:1rem" aria-label="Close">
        <i class="fa-solid fa-xmark text-lg" aria-hidden="true"></i>
      </button>

      <div class="text-center">
        <div class="mx-auto flex h-14 w-14 items-center justify-center rounded-full" style="background:#F5F3FF">
          <i class="fa-solid fa-paper-plane text-xl" style="color:#7C3AED" aria-hidden="true"></i>
        </div>
        <h2 id="share-modal-title" class="mt-4 text-2xl font-bold tracking-tight text-slate-900">Your clip link</h2>
        <p class="mt-1 text-sm text-slate-500">Anyone with this link sees exactly the window you chose.</p>
      </div>

      <div id="share-link-text" class="link-box mt-6" tabindex="0" aria-label="Generated share link"></div>

      <div class="mt-5">
        <button id="copy-btn" type="button" class="btn btn-primary btn-lg btn-block">
          <i class="fa-regular fa-copy" aria-hidden="true"></i><span>Copy Link</span>
        </button>
      </div>
      <p id="copy-hint" class="inline-error is-hidden mt-3" role="alert"></p>

      <div class="mt-3">
        <a id="open-clip-link" class="btn btn-secondary btn-md btn-block" href="/" target="_blank" rel="noopener">
          <i class="fa-solid fa-arrow-up-right-from-square" aria-hidden="true"></i><span>Open Clip</span>
        </a>
      </div>
    </div>
  </div>

  <!-- ===================================================================
       LEGAL MODAL (Terms of Service / Privacy Policy)
       =================================================================== -->
  <div id="legal-modal" class="modal is-hidden" role="dialog" aria-modal="true" aria-labelledby="legal-title">
    <div class="modal-backdrop"></div>
    <div class="legal-card">
      <div class="legal-head">
        <div>
          <p class="legal-kicker">ShareSpark</p>
          <h2 id="legal-title" class="legal-title"></h2>
          <p id="legal-updated" class="legal-updated"></p>
        </div>
        <button id="legal-x" type="button" class="btn btn-ghost btn-icon" aria-label="Close">
          <i class="fa-solid fa-xmark text-lg" aria-hidden="true"></i>
        </button>
      </div>
      <div id="legal-body" class="legal-body" tabindex="0"></div>
      <div class="legal-foot">
        <button id="legal-close" type="button" class="btn btn-primary btn-md" style="min-width:12rem">Close</button>
      </div>
    </div>
  </div>

  <!-- ===================================================================
       CLIENT APPLICATION LOGIC (vanilla JavaScript)
       =================================================================== -->
  <script>
  (function () {
    'use strict';

    /* ---------------------------------------------------------------
       0. SERVER-PROVIDED CONFIG
       The Worker decided the context (builder or viewer) and validated
       any share parameters before this script ever ran.
       --------------------------------------------------------------- */
    var CONFIG = __SHARESPARK_CONFIG__;

    /* ---------------------------------------------------------------
       1. CONSTANTS
       --------------------------------------------------------------- */
    var VIDEO_DURATION_SECONDS = 300;      // Simulated detected video length
    var EXTRACTION_DELAY_MS = 2000;        // State 2 duration
    var SERIALIZATION_DELAY_MS = 1500;     // State 4 duration
    var COPIED_RESET_MS = 2500;            // How long the green "Copied!" state lasts

    var MSG_BOUNDARY = 'Invalid time values! Input values must strictly remain inside 0 and 300 seconds.';
    var MSG_EQUAL = 'Bawal magkaparehas ang Start at End time markers. Magkaiba dapat.';
    var MSG_ORDER = 'Ang End Time ay dapat laging mas mataas o lagpas sa Start Time.';
    var MSG_MISSING = 'Please enter both a Start Time and an End Time as numbers of seconds.';
    var MSG_LINK_EMPTY = 'Please paste a video link first.';
    var MSG_LINK_INVALID = 'That does not look like a valid link. It should start with http:// or https://';

    /* ---------------------------------------------------------------
       2. SMALL DOM + FORMAT HELPERS
       --------------------------------------------------------------- */
    function byId(id) { return document.getElementById(id); }
    function show(el) { if (el) { el.classList.remove('is-hidden'); } }
    function hide(el) { if (el) { el.classList.add('is-hidden'); } }

    /** Formats seconds as m:ss (keeps one decimal when needed). */
    function formatClock(totalSeconds) {
      var safe = Math.max(0, Number(totalSeconds) || 0);
      var minutes = Math.floor(safe / 60);
      var seconds = Math.round((safe - minutes * 60) * 10) / 10;
      if (seconds >= 60) { minutes += 1; seconds = 0; }
      return minutes + ':' + (seconds < 10 ? '0' : '') + String(seconds);
    }

    /** Renders an inline red alert (icon + text) into the given element. */
    function setInlineMessage(el, message) {
      el.textContent = '';
      var icon = document.createElement('i');
      icon.className = 'fa-solid fa-circle-exclamation';
      icon.style.marginTop = '0.2rem';
      icon.setAttribute('aria-hidden', 'true');
      var text = document.createElement('span');
      text.textContent = message;
      el.appendChild(icon);
      el.appendChild(text);
      show(el);
    }

    function clearInlineMessage(el) {
      el.textContent = '';
      hide(el);
    }

    function isHttpUrl(value) {
      try {
        var parsed = new URL(value);
        return parsed.protocol === 'http:' || parsed.protocol === 'https:';
      } catch (err) {
        return false;
      }
    }

    /* ---------------------------------------------------------------
       3. DYNAMIC TIMESTAMP FIELDS (shared by Builder and Viewer)
       Renders two side-by-side number inputs; "locked" disables them.
       --------------------------------------------------------------- */
    function buildTimeField(options) {
      var wrapper = document.createElement('div');

      var label = document.createElement('label');
      label.className = 'field-label';
      label.htmlFor = options.id;
      label.textContent = options.label;

      var holder = document.createElement('div');
      holder.style.position = 'relative';

      var input = document.createElement('input');
      input.id = options.id;
      input.type = 'number';
      input.className = 'time-input';
      input.min = '0';
      input.max = String(VIDEO_DURATION_SECONDS);
      input.step = 'any';
      input.setAttribute('inputmode', 'decimal');
      input.autocomplete = 'off';
      input.placeholder = options.placeholder;
      if (options.value !== null && options.value !== undefined) {
        input.value = String(options.value);
      }

      holder.appendChild(input);

      if (options.locked) {
        input.disabled = true;
        input.setAttribute('aria-disabled', 'true');
        var lock = document.createElement('span');
        lock.className = 'field-lock';
        lock.setAttribute('aria-hidden', 'true');
        lock.innerHTML = '<i class="fa-solid fa-lock"></i>';
        holder.appendChild(lock);
      }

      wrapper.appendChild(label);
      wrapper.appendChild(holder);
      return wrapper;
    }

    function renderTimeFields(container, options) {
      container.textContent = '';
      container.appendChild(buildTimeField({
        id: options.prefix + '-start',
        label: 'Start Time (seconds)',
        placeholder: options.locked ? '-' : '0',
        value: options.start,
        locked: options.locked
      }));
      container.appendChild(buildTimeField({
        id: options.prefix + '-end',
        label: 'End Time (seconds)',
        placeholder: options.locked ? '-' : '30',
        value: options.end,
        locked: options.locked
      }));
    }

    /* ---------------------------------------------------------------
       4. MODAL ENGINE (focus handling, scroll lock, Escape, Tab trap)
       --------------------------------------------------------------- */
    var modalStack = [];
    var focusReturnStack = [];

    function openModal(modal, focusTarget) {
      focusReturnStack.push(document.activeElement);
      show(modal);
      modalStack.push(modal);
      document.body.classList.add('modal-open');
      window.setTimeout(function () {
        if (focusTarget && focusTarget.focus) { focusTarget.focus(); }
      }, 30);
    }

    function closeModal(modal) {
      if (modalStack.indexOf(modal) === -1) { return; }
      hide(modal);
      modalStack = modalStack.filter(function (item) { return item !== modal; });
      if (modalStack.length === 0) { document.body.classList.remove('modal-open'); }
      var previous = focusReturnStack.pop();
      if (previous && previous.focus) { previous.focus(); }
    }

    function topModal() { return modalStack.length ? modalStack[modalStack.length - 1] : null; }

    document.addEventListener('keydown', function (event) {
      var modal = topModal();
      if (!modal) { return; }

      if (event.key === 'Escape') {
        event.preventDefault();
        closeModal(modal);
        return;
      }

      if (event.key === 'Tab') {
        var nodes = modal.querySelectorAll('button:not([disabled]), a[href], input:not([disabled]), [tabindex]:not([tabindex="-1"])');
        var focusables = Array.prototype.filter.call(nodes, function (node) { return node.getClientRects().length > 0; });
        if (focusables.length === 0) { return; }
        var first = focusables[0];
        var last = focusables[focusables.length - 1];
        if (event.shiftKey && document.activeElement === first) {
          event.preventDefault();
          last.focus();
        } else if (!event.shiftKey && document.activeElement === last) {
          event.preventDefault();
          first.focus();
        }
      }
    });

    /* ---------------------------------------------------------------
       5. LEGAL CONTENT (Terms of Service and Privacy Policy)
       --------------------------------------------------------------- */
    var LEGAL = {
      terms: {
        title: 'Terms of Service',
        updated: 'November 3, 2026',
        intro: [
          'These Terms of Service (the “Terms”) form a binding agreement between you and the operator of ShareSpark (“ShareSpark”, “we”, “us” or “our”). They govern your access to and use of the ShareSpark website and services (the “Service”). Please read them carefully before using the Service.',
          'By accessing or using the Service you confirm that you have read, understood and agreed to be bound by these Terms. If you do not agree, you must not use the Service.'
        ],
        sections: [
          { title: 'Acceptance of Terms', paragraphs: [
            'Your use of the Service constitutes acceptance of these Terms and of any additional guidelines or rules posted on the Service. If you use the Service on behalf of an organization, you represent that you have authority to bind that organization to these Terms.'
          ] },
          { title: 'Description of the Service', paragraphs: [
            'ShareSpark is a tool that lets you define a start time and an end time for a video that is hosted elsewhere, and generates a link that plays only that portion of the video in a web browser.',
            'ShareSpark does not host, upload, store, copy, re-encode or redistribute video files. The generated link only contains the address of the video, together with your chosen start and end markers.'
          ] },
          { title: 'Eligibility', paragraphs: [
            'You must be at least the age of digital consent in your jurisdiction, and in any case at least 13 years old, to use the Service. You must have the legal capacity to enter into a binding agreement and must not be barred from using the Service under applicable law.'
          ] },
          { title: 'Third-Party Content and Video Sources', paragraphs: [
            'Videos played through ShareSpark links are provided by third-party servers that we do not own, operate or control. We make no representation about the legality, accuracy, quality, safety or availability of any third-party video.',
            'A clip will stop working if the underlying video is moved, made private, blocked or removed by its host. Such third-party content remains subject to the terms and policies of the party that provides it.'
          ] },
          { title: 'Your Responsibilities and Content Rights', paragraphs: [
            'You are solely responsible for every video link you submit and every share link you create or distribute. By creating a clip you represent and warrant that:'
          ], bullets: [
            'you own the video or hold all rights, licenses and permissions required to share it in this way;',
            'your use of the Service does not infringe or violate any copyright, trademark, privacy, publicity or other right of any person; and',
            'your use of the Service complies with all applicable laws and with the terms of the platform that hosts the video.'
          ] },
          { title: 'Share Links and Clip Windows', paragraphs: [
            'Share links are generated in your browser and contain all of the information required to play the clip. Anyone who receives a link can open it, and anyone can copy or forward it. Do not create share links for content you do not want others to access.',
            'Start and end markers are enforced by the player on the viewing page. They are a presentation feature and not a security or access-control measure; they do not prevent a determined person from accessing the full video at its original source.'
          ] },
          { title: 'Prohibited Conduct', paragraphs: [ 'You agree that you will not, and will not allow others to:' ], bullets: [
            'use the Service to share content that is unlawful, defamatory, harassing, hateful, sexually exploitative, or that promotes violence or self-harm;',
            'use the Service to infringe the intellectual property or privacy rights of others, or to circumvent paywalls, access controls or technical protection measures;',
            'distribute malware, phishing links or deceptive content through the Service;',
            'interfere with, overload, scrape or attempt to gain unauthorized access to the Service or its infrastructure; or',
            'misrepresent your identity or affiliation, or use the Service in any manner that violates applicable law.'
          ] },
          { title: 'Intellectual Property', paragraphs: [
            'The Service, including its design, text, graphics, logos, trademarks and software, is owned by ShareSpark or its licensors and is protected by intellectual property laws. Except as expressly permitted in these Terms, you may not copy, modify, distribute, sell or lease any part of the Service.',
            'We grant you a limited, revocable, non-exclusive, non-transferable license to use the Service for its intended purpose in accordance with these Terms.'
          ] },
          { title: 'Copyright Complaints and Content Removal', paragraphs: [
            'We respect the rights of copyright holders. Because ShareSpark does not host videos, the most effective way to remove a video is to contact the platform that hosts it. If you believe a ShareSpark link facilitates infringement, contact us through the channel published on the Service and include a description of the work, the full share link, and your contact details.',
            'We may disable specific share links, or block specific video sources, at our sole discretion and without prior notice.'
          ] },
          { title: 'Service Availability and Changes', paragraphs: [
            'We strive to keep the Service available but do not guarantee uninterrupted or error-free operation. We may modify, suspend or discontinue any part of the Service at any time. Features described as upcoming are not commitments.'
          ] },
          { title: 'Disclaimer of Warranties', paragraphs: [
            'THE SERVICE IS PROVIDED “AS IS” AND “AS AVAILABLE”, WITHOUT WARRANTIES OF ANY KIND, WHETHER EXPRESS, IMPLIED OR STATUTORY, INCLUDING IMPLIED WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE, TITLE AND NON-INFRINGEMENT.',
            'We do not warrant that any clip will play on every device or browser, that any video source will remain available, or that the Service will meet your requirements.'
          ] },
          { title: 'Limitation of Liability', paragraphs: [
            'TO THE FULLEST EXTENT PERMITTED BY LAW, SHARESPARK AND ITS OPERATORS, CONTRIBUTORS AND AFFILIATES WILL NOT BE LIABLE FOR ANY INDIRECT, INCIDENTAL, SPECIAL, CONSEQUENTIAL OR PUNITIVE DAMAGES, OR FOR ANY LOSS OF PROFITS, DATA, GOODWILL OR REPUTATION, ARISING OUT OF OR RELATED TO YOUR USE OF, OR INABILITY TO USE, THE SERVICE.',
            'Where liability cannot be excluded, our total liability for all claims relating to the Service is limited to the greater of the amount you paid us for the Service in the twelve months before the claim arose, or fifty US dollars.'
          ] },
          { title: 'Indemnification', paragraphs: [
            'You agree to defend, indemnify and hold harmless ShareSpark and its operators from and against any claims, liabilities, damages, losses and expenses, including reasonable legal fees, arising out of or related to your use of the Service, the links you create or share, or your breach of these Terms.'
          ] },
          { title: 'Suspension and Termination', paragraphs: [
            'We may suspend or terminate your access to the Service, and disable any share link, at any time and for any reason, including a suspected violation of these Terms. You may stop using the Service at any time. Sections that by their nature should survive termination will survive.'
          ] },
          { title: 'Governing Law and Disputes', paragraphs: [
            'These Terms are governed by the laws of the jurisdiction in which the operator of ShareSpark is established, without regard to conflict-of-law principles. You agree to first attempt to resolve any dispute informally by contacting us. Subject to mandatory consumer protection laws, disputes will be submitted to the competent courts of that jurisdiction.'
          ] },
          { title: 'Changes to These Terms', paragraphs: [
            'We may update these Terms from time to time. When we do, we will revise the “Last updated” date above. Your continued use of the Service after changes take effect constitutes acceptance of the revised Terms.'
          ] },
          { title: 'General Provisions', paragraphs: [
            'These Terms are the entire agreement between you and ShareSpark regarding the Service. If any provision is held to be unenforceable, the remaining provisions will remain in full effect. Our failure to enforce a provision is not a waiver of it. You may not assign these Terms without our consent; we may assign them in connection with a reorganization or sale of the Service.'
          ] },
          { title: 'Contact', paragraphs: [
            'Questions about these Terms may be sent through the contact channel published on the ShareSpark website.'
          ] }
        ]
      },

      privacy: {
        title: 'Privacy Policy',
        updated: 'November 3, 2026',
        intro: [
          'This Privacy Policy explains what information ShareSpark handles when you use the Service, how that information is used, and the choices available to you. ShareSpark is designed to be lightweight and stateless: it has no user accounts and does not maintain a database of the videos or clips you create.',
          'By using the Service you acknowledge the practices described in this Privacy Policy.'
        ],
        sections: [
          { title: 'Scope of This Policy', paragraphs: [
            'This policy applies to the ShareSpark website and the share links it generates. It does not apply to third-party websites, video hosts or platforms that you link to or that you reach through a share link.'
          ] },
          { title: 'Information We Handle', paragraphs: [ 'Depending on how you use the Service, the following information may be handled:' ], bullets: [
            'Video links you paste: the address is processed in your browser to prepare a preview and a clip. We do not keep a copy of it on a server.',
            'Clip settings: the start and end markers you choose are written into the share link that your browser generates.',
            'Technical request data: like most web services, the infrastructure that delivers ShareSpark receives standard connection data such as IP address, browser type, device type, requested address and timestamps.'
          ] },
          { title: 'Information We Do Not Collect', paragraphs: [
            'ShareSpark does not require registration, and does not ask for your name, email address, phone number, payment details or precise location. We do not knowingly collect special categories of personal data.'
          ] },
          { title: 'How Share Links Contain Data', paragraphs: [
            'A share link carries the video address, the start marker and the end marker as part of the link itself. Anyone who has the link can see that information, and links may be stored by messaging apps, browsers, search tools or analytics of the people and platforms you send them to.',
            'Please do not create clips from video addresses that contain private tokens, passwords or other sensitive information.'
          ] },
          { title: 'How We Use Information', paragraphs: [ 'We use the limited information described above to:' ], bullets: [
            'deliver, secure and maintain the Service;',
            'detect, prevent and investigate abuse, fraud, spam and technical problems;',
            'understand overall reliability and performance; and',
            'comply with legal obligations and enforce our Terms of Service.'
          ] },
          { title: 'Cookies and Browser Storage', paragraphs: [
            'ShareSpark does not set advertising or tracking cookies. The Service does not use local browser storage to track you across websites. Third-party resources described below may set their own technical cookies or storage according to their own policies.'
          ] },
          { title: 'Third-Party Services', paragraphs: [
            'To operate, the Service relies on third parties, each of which may receive your IP address and basic browser information when your browser contacts them:'
          ], bullets: [
            'Cloudflare, which hosts and delivers the Service;',
            'content delivery networks that supply interface code and icons, including the Tailwind CSS and Font Awesome CDNs;',
            'Google Fonts, which supplies the typeface; and',
            'the video host you link to. When a preview or a clip plays, your browser connects directly to that host, which handles your data under its own privacy policy.'
          ] },
          { title: 'Data Retention', paragraphs: [
            'Because ShareSpark does not store clips or accounts, there is no clip database to retain or delete. Technical request logs are kept only for as long as needed for security, reliability and legal compliance, in accordance with our infrastructure provider’s practices.'
          ] },
          { title: 'Security', paragraphs: [
            'We use industry-standard transport encryption (HTTPS) and limit the information we handle. No method of transmission or storage is completely secure, and we cannot guarantee absolute security.'
          ] },
          { title: 'Your Rights and Choices', paragraphs: [
            'Depending on where you live, you may have rights to access, correct, delete, restrict or object to the processing of personal data, and to lodge a complaint with your local data protection authority. Because we hold little identifying information, we may be unable to link a request to a specific person, but we will review every request in good faith.',
            'You can stop sharing a clip at any time by ceasing to distribute its link, and you can ask the video host to remove or restrict the underlying video.'
          ] },
          { title: 'Children’s Privacy', paragraphs: [
            'The Service is not directed to children under 13, and we do not knowingly collect personal information from them. If you believe a child has provided personal information through the Service, please contact us so that we can take appropriate action.'
          ] },
          { title: 'International Data Transfers', paragraphs: [
            'Our infrastructure operates globally, so your information may be processed in countries other than your own, which may have different data protection laws. Where required, we rely on appropriate safeguards for such transfers.'
          ] },
          { title: 'Changes to This Policy', paragraphs: [
            'We may update this Privacy Policy as the Service evolves. We will revise the “Last updated” date above when we do. Your continued use of the Service after an update means you accept the revised policy.'
          ] },
          { title: 'Contact', paragraphs: [
            'Privacy questions and requests may be sent through the contact channel published on the ShareSpark website.'
          ] }
        ]
      }
    };

    function renderLegal(kind) {
      var doc = LEGAL[kind];
      var titleEl = byId('legal-title');
      var updatedEl = byId('legal-updated');
      var bodyEl = byId('legal-body');

      titleEl.textContent = doc.title;
      updatedEl.textContent = 'Last updated: ' + doc.updated;
      bodyEl.textContent = '';

      doc.intro.forEach(function (text) {
        var p = document.createElement('p');
        p.textContent = text;
        bodyEl.appendChild(p);
      });

      doc.sections.forEach(function (section, index) {
        var heading = document.createElement('h3');
        var badge = document.createElement('span');
        badge.className = 'legal-num';
        badge.textContent = String(index + 1);
        var label = document.createElement('span');
        label.textContent = section.title;
        heading.appendChild(badge);
        heading.appendChild(label);
        bodyEl.appendChild(heading);

        section.paragraphs.forEach(function (text) {
          var p = document.createElement('p');
          p.textContent = text;
          bodyEl.appendChild(p);
        });

        if (section.bullets) {
          var list = document.createElement('ul');
          section.bullets.forEach(function (text) {
            var li = document.createElement('li');
            li.textContent = text;
            list.appendChild(li);
          });
          bodyEl.appendChild(list);
        }
      });

      bodyEl.scrollTop = 0;
    }

    function wireLegalAndFooter() {
      var legalModal = byId('legal-modal');

      Array.prototype.forEach.call(document.querySelectorAll('[data-legal]'), function (trigger) {
        trigger.addEventListener('click', function () {
          renderLegal(trigger.getAttribute('data-legal'));
          openModal(legalModal, byId('legal-body'));
        });
      });

      byId('legal-close').addEventListener('click', function () { closeModal(legalModal); });
      byId('legal-x').addEventListener('click', function () { closeModal(legalModal); });
      legalModal.querySelector('.modal-backdrop').addEventListener('click', function () { closeModal(legalModal); });
    }

    /* ---------------------------------------------------------------
       6. CONTEXT A: BUILDER
       --------------------------------------------------------------- */
    function initBuilder() {
      var BUILDER_STATES = ['state-input', 'state-extract', 'state-controls', 'state-serialize', 'state-share'];

      var builder = { link: '', shareUrl: '', timers: [], copyTimer: null };

      var linkInput = byId('video-link');
      var linkError = byId('link-error');
      var processBtn = byId('process-btn');
      var timeFields = byId('time-fields');
      var timeError = byId('time-error');
      var generateBtn = byId('generate-btn');
      var backBtn = byId('back-btn');
      var previewVideo = byId('preview-video');
      var previewFallback = byId('preview-fallback');
      var shareClipBtn = byId('share-clip-btn');
      var createAnotherBtn = byId('create-another-btn');

      var shareModal = byId('share-modal');
      var shareLinkText = byId('share-link-text');
      var copyBtn = byId('copy-btn');
      var copyHint = byId('copy-hint');
      var openClipLink = byId('open-clip-link');

      show(byId('view-builder'));

      /* ----- state machine ----- */
      function showState(name) {
        BUILDER_STATES.forEach(function (id) {
          var section = byId(id);
          if (id === name) { show(section); } else { hide(section); }
        });
      }

      function schedule(fn, ms) {
        var id = window.setTimeout(fn, ms);
        builder.timers.push(id);
      }

      function clearTimers() {
        builder.timers.forEach(function (id) { window.clearTimeout(id); });
        builder.timers = [];
      }

      /* ----- State 1 -> State 2 -> State 3 ----- */
      function handleProcess() {
        clearInlineMessage(linkError);
        linkInput.classList.remove('is-invalid');

        var raw = linkInput.value.trim();
        if (raw === '') {
          linkInput.classList.add('is-invalid');
          setInlineMessage(linkError, MSG_LINK_EMPTY);
          linkInput.focus();
          return;
        }

        // Be forgiving: "example.com/video.mp4" becomes "https://example.com/video.mp4".
        var candidate = raw.indexOf('://') === -1 ? 'https://' + raw : raw;
        if (!isHttpUrl(candidate)) {
          linkInput.classList.add('is-invalid');
          setInlineMessage(linkError, MSG_LINK_INVALID);
          linkInput.focus();
          return;
        }

        builder.link = candidate;
        showState('state-extract');

        // Simulated 2-second extraction, then reveal the preview and controls.
        schedule(function () {
          preparePreview(builder.link);
          renderBuilderFields();
          showState('state-controls');
          var startEl = byId('builder-start');
          if (startEl) { startEl.focus(); }
        }, EXTRACTION_DELAY_MS);
      }

      function preparePreview(link) {
        hide(previewFallback);
        previewVideo.pause();
        previewVideo.removeAttribute('src');
        previewVideo.src = link;
        previewVideo.load();
      }

      previewVideo.addEventListener('error', function () {
        if (previewVideo.getAttribute('src')) { show(previewFallback); }
      });
      previewVideo.addEventListener('loadedmetadata', function () { hide(previewFallback); });

      /* ----- State 3: dynamic fields + validation ----- */
      function renderBuilderFields() {
        clearInlineMessage(timeError);
        renderTimeFields(timeFields, { prefix: 'builder', locked: false, start: null, end: null });

        var startEl = byId('builder-start');
        var endEl = byId('builder-end');

        [startEl, endEl].forEach(function (el) {
          el.addEventListener('input', function () {
            clearInlineMessage(timeError);
            startEl.classList.remove('is-invalid');
            endEl.classList.remove('is-invalid');
          });
          el.addEventListener('keydown', function (event) {
            if (event.key === 'Enter') { event.preventDefault(); handleGenerate(); }
          });
        });

        // Convenience: jump the preview to the chosen start time.
        startEl.addEventListener('change', function () {
          var value = Number(startEl.value);
          if (startEl.value !== '' && isFinite(value) && value >= 0 && value <= VIDEO_DURATION_SECONDS && previewVideo.readyState >= 1) {
            try { previewVideo.currentTime = Math.min(value, previewVideo.duration || value); } catch (err) { /* ignore */ }
          }
        });
      }

      /**
       * STRICT validation against the simulated 300 second video.
       * Order matters: boundary, then equality, then sequence.
       */
      function validateTimes(startRaw, endRaw) {
        var startText = String(startRaw).trim();
        var endText = String(endRaw).trim();

        if (startText === '' || endText === '') {
          return { ok: false, message: MSG_MISSING, fields: ['builder-start', 'builder-end'] };
        }

        var start = Number(startText);
        var end = Number(endText);
        if (!isFinite(start) || !isFinite(end)) {
          return { ok: false, message: MSG_MISSING, fields: ['builder-start', 'builder-end'] };
        }

        // Rule 1: boundary check
        if (start < 0 || start > VIDEO_DURATION_SECONDS || end < 0 || end > VIDEO_DURATION_SECONDS) {
          var offenders = [];
          if (start < 0 || start > VIDEO_DURATION_SECONDS) { offenders.push('builder-start'); }
          if (end < 0 || end > VIDEO_DURATION_SECONDS) { offenders.push('builder-end'); }
          return { ok: false, message: MSG_BOUNDARY, fields: offenders };
        }

        // Rule 2: equality check
        if (start === end) {
          return { ok: false, message: MSG_EQUAL, fields: ['builder-start', 'builder-end'] };
        }

        // Rule 3: sequential order check
        if (end <= start) {
          return { ok: false, message: MSG_ORDER, fields: ['builder-end'] };
        }

        return { ok: true, start: start, end: end };
      }

      function buildShareUrl(link, start, end) {
        return window.location.origin + '/?mp4=' + encodeURIComponent(link) +
          '&start=' + encodeURIComponent(String(start)) +
          '&end=' + encodeURIComponent(String(end));
      }

      /* ----- State 3 -> State 4 -> State 5 ----- */
      function handleGenerate() {
        var startEl = byId('builder-start');
        var endEl = byId('builder-end');

        clearInlineMessage(timeError);
        startEl.classList.remove('is-invalid');
        endEl.classList.remove('is-invalid');

        var result = validateTimes(startEl.value, endEl.value);
        if (!result.ok) {
          result.fields.forEach(function (id) { byId(id).classList.add('is-invalid'); });
          setInlineMessage(timeError, result.message);
          byId(result.fields[0]).focus();
          return; // Execution is blocked.
        }

        builder.shareUrl = buildShareUrl(builder.link, result.start, result.end);
        showState('state-serialize');

        schedule(function () {
          showState('state-share');
          shareClipBtn.focus();
        }, SERIALIZATION_DELAY_MS);
      }

      /* ----- Share modal + Clipboard ----- */
      function resetCopyButton() {
        if (builder.copyTimer) { window.clearTimeout(builder.copyTimer); builder.copyTimer = null; }
        copyBtn.classList.remove('btn-success');
        copyBtn.classList.add('btn-primary');
        copyBtn.innerHTML = '<i class="fa-regular fa-copy" aria-hidden="true"></i><span>Copy Link</span>';
      }

      function openShare() {
        shareLinkText.textContent = builder.shareUrl;
        openClipLink.setAttribute('href', builder.shareUrl);
        clearInlineMessage(copyHint);
        resetCopyButton();
        openModal(shareModal, copyBtn);
      }

      function copyText(text) {
        // Primary path: the asynchronous Clipboard API.
        if (navigator.clipboard && window.isSecureContext) {
          return navigator.clipboard.writeText(text);
        }
        // Fallback for non-secure contexts or older browsers.
        return new Promise(function (resolve, reject) {
          try {
            var area = document.createElement('textarea');
            area.value = text;
            area.setAttribute('readonly', '');
            area.style.position = 'fixed';
            area.style.top = '0';
            area.style.opacity = '0';
            document.body.appendChild(area);
            area.select();
            area.setSelectionRange(0, text.length);
            var ok = document.execCommand('copy');
            document.body.removeChild(area);
            if (ok) { resolve(); } else { reject(new Error('Copy command was rejected')); }
          } catch (err) {
            reject(err);
          }
        });
      }

      function selectShareText() {
        var range = document.createRange();
        range.selectNodeContents(shareLinkText);
        var selection = window.getSelection();
        selection.removeAllRanges();
        selection.addRange(range);
      }

      function handleCopy() {
        clearInlineMessage(copyHint);
        copyText(builder.shareUrl).then(function () {
          // Toggle to the oval green "Copied!" state.
          copyBtn.classList.remove('btn-primary');
          copyBtn.classList.add('btn-success');
          copyBtn.innerHTML = '<i class="fa-solid fa-check" aria-hidden="true"></i><span>Copied!</span>';
          if (builder.copyTimer) { window.clearTimeout(builder.copyTimer); }
          builder.copyTimer = window.setTimeout(resetCopyButton, COPIED_RESET_MS);
        }).catch(function () {
          selectShareText();
          setInlineMessage(copyHint, 'Copy was blocked by your browser. The link is selected, press Ctrl+C or Cmd+C to copy it.');
        });
      }

      /* ----- Reset helpers ----- */
      function backToLink() {
        clearTimers();
        previewVideo.pause();
        showState('state-input');
        linkInput.focus();
      }

      function resetBuilder() {
        clearTimers();
        previewVideo.pause();
        previewVideo.removeAttribute('src');
        previewVideo.load();
        hide(previewFallback);
        linkInput.value = '';
        linkInput.classList.remove('is-invalid');
        clearInlineMessage(linkError);
        clearInlineMessage(timeError);
        builder.link = '';
        builder.shareUrl = '';
        showState('state-input');
        linkInput.focus();
      }

      /* ----- Event wiring ----- */
      processBtn.addEventListener('click', handleProcess);
      linkInput.addEventListener('keydown', function (event) {
        if (event.key === 'Enter') { event.preventDefault(); handleProcess(); }
      });
      linkInput.addEventListener('input', function () {
        linkInput.classList.remove('is-invalid');
        clearInlineMessage(linkError);
      });

      generateBtn.addEventListener('click', handleGenerate);
      backBtn.addEventListener('click', backToLink);
      shareClipBtn.addEventListener('click', openShare);
      createAnotherBtn.addEventListener('click', resetBuilder);

      copyBtn.addEventListener('click', handleCopy);
      shareLinkText.addEventListener('click', selectShareText);
      byId('share-modal-close').addEventListener('click', function () { closeModal(shareModal); });
      shareModal.querySelector('.modal-backdrop').addEventListener('click', function () { closeModal(shareModal); });

      showState('state-input');
      linkInput.focus();
    }

    /* ---------------------------------------------------------------
       7. CONTEXT B: VIEWER
       The Builder is skipped entirely. The player jumps to the decoded
       start marker, plays, and is locked inside [start, end].
       --------------------------------------------------------------- */
    function initViewer(cfg) {
      show(byId('view-viewer'));

      var start = cfg.start;
      var end = cfg.end;
      var hasWindow = typeof start === 'number' && typeof end === 'number';

      // Locked (disabled) timestamp fields.
      renderTimeFields(byId('viewer-fields'), { prefix: 'viewer', locked: true, start: hasWindow ? start : null, end: hasWindow ? end : null });

      var video = byId('viewer-video');
      var stage = byId('viewer-stage');
      var errorBox = byId('viewer-error');
      var errorText = byId('viewer-error-text');
      var overlay = byId('autoplay-overlay');
      var overlayBtn = byId('autoplay-btn');
      var toggleBtn = byId('viewer-toggle');
      var fill = byId('clip-fill');
      var track = byId('clip-track');
      var elapsedLabel = byId('clip-elapsed');
      var lengthLabel = byId('clip-length');
      var subtitle = byId('viewer-subtitle');

      function showViewerError(message) {
        try { video.pause(); } catch (err) { /* ignore */ }
        hide(stage);
        errorText.textContent = message;
        show(errorBox);
        toggleBtn.disabled = true;
      }

      // Server-side validation already failed: show the reason and stop.
      if (cfg.error || !hasWindow || !isHttpUrl(cfg.mp4 || '')) {
        subtitle.textContent = 'This link is missing or has damaged clip information.';
        lengthLabel.textContent = '0:00';
        showViewerError(cfg.error || 'This clip link is missing or has damaged clip information.');
        return;
      }

      subtitle.textContent = 'Playing ' + formatClock(start) + ' to ' + formatClock(end) + ' (' + formatClock(end - start) + ' long)';
      lengthLabel.textContent = formatClock(end - start);

      var SEEK_TOLERANCE = 0.05;
      var frameHandle = null;

      function isAtEnd() { return video.currentTime >= end - SEEK_TOLERANCE; }

      function seekToStart() {
        try { video.currentTime = start; } catch (err) { /* ignore until metadata is ready */ }
      }

      /** Forced pause the moment the timeline crosses or equals the end marker. */
      function enforceEnd() {
        if (video.currentTime >= end) {
          video.pause();
          try { video.currentTime = end; } catch (err) { /* ignore */ }
          return true;
        }
        return false;
      }

      function updateProgress() {
        var span = end - start;
        var ratio = span > 0 ? (video.currentTime - start) / span : 0;
        ratio = Math.max(0, Math.min(1, ratio));
        fill.style.width = (ratio * 100).toFixed(2) + '%';
        track.setAttribute('aria-valuenow', String(Math.round(ratio * 100)));
        elapsedLabel.textContent = formatClock(Math.max(0, Math.min(span, video.currentTime - start)));
      }

      function syncToggle() {
        var label = 'Play';
        var icon = 'fa-play';
        if (video.paused && isAtEnd()) { label = 'Replay Clip'; icon = 'fa-rotate-right'; }
        else if (!video.paused) { label = 'Pause'; icon = 'fa-pause'; }
        toggleBtn.innerHTML = '<i class="fa-solid ' + icon + '" aria-hidden="true"></i><span>' + label + '</span>';
      }

      function playClip() {
        var attempt = video.play();
        if (attempt && attempt.catch) {
          // Autoplay with sound can be blocked until the viewer taps.
          attempt.catch(function () { show(overlay); });
        }
      }

      /** Frame-accurate watcher so the pause is not delayed by the ~250ms timeupdate cadence. */
      function watchFrames() {
        if (frameHandle !== null) { return; }
        function tick() {
          frameHandle = null;
          if (video.paused || video.ended) { return; }
          if (enforceEnd()) { updateProgress(); syncToggle(); return; }
          updateProgress();
          frameHandle = window.requestAnimationFrame(tick);
        }
        frameHandle = window.requestAnimationFrame(tick);
      }

      /* Jump to the decoded start marker the instant metadata is available, then play. */
      var initialised = false;
      video.addEventListener('loadedmetadata', function () {
        if (isFinite(video.duration) && start >= video.duration) {
          showViewerError('The start of this clip is beyond the end of the video it points to.');
          return;
        }
        initialised = true;
        seekToStart();
        playClip();
      });

      // Some browsers need data before honouring the first seek; repeat once if required.
      video.addEventListener('canplay', function () {
        if (initialised && video.currentTime < start - 0.25) { seekToStart(); }
      });

      /* Continuous monitoring of the media timeline. */
      video.addEventListener('timeupdate', function () {
        enforceEnd();
        updateProgress();
        syncToggle();
      });

      /* Lock the viewer inside the configured window even if they scrub. */
      video.addEventListener('seeking', function () {
        var t = video.currentTime;
        if (t < start - SEEK_TOLERANCE) {
          seekToStart();
        } else if (t > end + SEEK_TOLERANCE) {
          video.pause();
          try { video.currentTime = end; } catch (err) { /* ignore */ }
        }
      });

      /* Pressing play at the end (or before the window) restarts from the start marker. */
      video.addEventListener('play', function () {
        if (video.currentTime >= end - SEEK_TOLERANCE || video.currentTime < start - SEEK_TOLERANCE) { seekToStart(); }
        syncToggle();
      });
      video.addEventListener('playing', function () { hide(overlay); watchFrames(); syncToggle(); });
      video.addEventListener('pause', function () { syncToggle(); updateProgress(); });
      video.addEventListener('ended', function () { syncToggle(); updateProgress(); });

      video.addEventListener('error', function () {
        showViewerError('We could not load this video. The source may be offline, private, or not a direct video file such as an .mp4.');
      });

      /* Buttons */
      toggleBtn.addEventListener('click', function () {
        if (video.paused) {
          if (isAtEnd()) { seekToStart(); }
          playClip();
        } else {
          video.pause();
        }
      });
      overlayBtn.addEventListener('click', function () {
        hide(overlay);
        if (isAtEnd()) { seekToStart(); }
        playClip();
      });

      // Everything is wired. Finally attach the encoded MP4 source.
      video.src = cfg.mp4;
      video.load();
      updateProgress();
      syncToggle();
    }

    /* ---------------------------------------------------------------
       8. BOOTSTRAP
       --------------------------------------------------------------- */
    wireLegalAndFooter();

    if (CONFIG && CONFIG.mode === 'viewer') {
      initViewer(CONFIG);
    } else {
      initBuilder();
    }
  })();
  </script>
</body>
</html>`;
