# hack4sdg-QuadCircuit
#base44
#entities
#CrowdReport
{
  "name": "CrowdReport",
  "type": "object",
  "properties": {
    "route_number": {
      "type": "string",
      "description": "BMTC route number as displayed on the bus; empty when the passenger did not note one"
    },
    "trip_ref": {
      "type": "string",
      "description": "Bus or trip identifier when the passenger could see one"
    },
    "stop_name": {
      "type": "string",
      "description": "Stop name exactly as stored in the imported BMTC dataset"
    },
    "crowd_level": {
      "type": "string",
      "enum": [
        "low",
        "moderate",
        "high",
        "full"
      ],
      "description": "Crowding as reported by the passenger"
    },
    "reported_at": {
      "type": "string",
      "format": "date-time",
      "description": "When the passenger made the observation"
    },
    "reporter_token": {
      "type": "string",
      "description": "Anonymous device token, stored only to prevent accidental duplicate submissions"
    },
    "route_verified": {
      "type": "boolean",
      "description": "True when the route number was chosen from the imported dataset rather than typed"
    },
    "source": {
      "type": "string",
      "description": "How the report was produced. Crowd levels in this app are always passenger_report"
    }
  },
  "required": [
    "stop_name",
    "crowd_level",
    "reported_at"
  ],
  "rls": {
    "read": {},
    "create": {},
    "update": {
      "user_condition": {
        "role": "admin"
      }
    },
    "delete": {
      "user_condition": {
        "role": "admin"
      }
    }
#TransitDataset
{
  "name": "TransitDataset",
  "type": "object",
  "properties": {
    "source_id": {
      "type": "string",
      "description": "Identifier of the imported dataset"
    },
    "source_name": {
      "type": "string",
      "description": "Human readable name of the dataset"
    },
    "source_url": {
      "type": "string",
      "description": "Where the dataset is published"
    },
    "data_url": {
      "type": "string",
      "description": "Exact file the records were imported from"
    },
    "licence": {
      "type": "string",
      "description": "Licence of the dataset as published by its authors"
    },
    "commit_date": {
      "type": "string",
      "description": "Date of the dataset's latest commit upstream"
    },
    "last_checked_at": {
      "type": "string",
      "format": "date-time",
      "description": "When the app last reached the source successfully"
    },
    "imported_at": {
      "type": "string",
      "format": "date-time",
      "description": "When the records were imported into this app"
    },
    "route_count": {
      "type": "number",
      "description": "Route directions imported"
    },
    "stop_count": {
      "type": "number",
      "description": "Stops imported"
    },
    "status": {
      "type": "string",
      "enum": [
        "empty",
        "importing",
        "ready",
        "failed"
      ],
      "description": "State of the import"
    },
    "notes": {
      "type": "string",
      "description": "Notes recorded during the last import"
    }
  },
  "required": [
    "source_id"
  ],
  "rls": {
    "read": {},
    "create": {
      "user_condition": {
        "role": "admin"
      }
    },
    "update": {
      "user_condition": {
        "role": "admin"
      }
    },
    "delete": {
      "user_condition": {
        "role": "admin"
      }
    }
  }
}
#User
{
  "properties": {
    "role": {
      "type": "string",
      "description": "The role of the user in the app",
      "enum": [
        "admin",
        "user"
      ]
    }
  },
  "required": [
    "role"
  ],
  "name": "User",
  "type": "object"
}
#functions
#bmtcQuery
#entry
import { createClientFromRequest } from 'npm:@base44/sdk@0.8.54';
import { ASSUMPTIONS, findRoutes, findStops, loadDataset, planCandidates, planJourney } from '../../shared/bmtcGtfs.ts';

// Serves route search, stop search and journey planning from the published BMTC dataset.
// The dataset is held in the worker's memory between calls, so only the first call in a
// cold worker pays the download and parse cost.
export default async function (req) {
  try {
    createClientFromRequest(req);
    let payload = {};
    try {
      payload = await req.json();
    } catch (parseError) {
      payload = {};
    }

    const action = payload.action;
    const limit = Math.min(40, Math.max(1, Number(payload.limit) || 20));
    const dataset = await loadDataset();

    if (action === 'searchRoutes') {
      return Response.json({ ok: true, routes: findRoutes(dataset, payload.query, limit) });
    }

    if (action === 'searchStops') {
      return Response.json({ ok: true, stops: findStops(dataset, payload.query, limit) });
    }

    if (action === 'plan') {
      const from = String(payload.from ?? '').trim();
      const to = String(payload.to ?? '').trim();
      if (!from || !to) return Response.json({ ok: true, options: [] });
      return Response.json({ ok: true, options: planJourney(dataset, from, to, limit) });
    }

    if (action === 'planJourneys') {
      const from = String(payload.from ?? '').trim();
      const to = String(payload.to ?? '').trim();
      const via = String(payload.via ?? '').trim();
      if (!from || !to) {
        return Response.json({ ok: false, journeys: [], assumptions: ASSUMPTIONS, error: 'missing_stops' }, { status: 400 });
      }
      const departRaw = Number(payload.departMinutes);
      const departMinutes = Number.isFinite(departRaw) ? departRaw : null;
      const journeys = planCandidates(dataset, { from, to, via, departMinutes, limit: 45 });
      return Response.json({ ok: true, journeys, assumptions: ASSUMPTIONS, from, to, via });
    }

    if (action === 'counts') {
      return Response.json({ ok: true, routes: dataset.routes.length, stops: dataset.stops.length });
    }

    return Response.json({ error: `Unknown action: ${action}` }, { status: 400 });
  } catch (error) {
    return Response.json({ error: error.message }, { status: 500 });
#bmtcSync
#entry
import { createClientFromRequest } from 'npm:@base44/sdk@0.8.54';
import {
  LICENCE,
  REPO_URL,
  ROUTES_CSV_URL,
  SOURCE_ID,
  SOURCE_NAME,
  latestCommitDate,
  loadDataset,
  routeCounts,
} from '../../shared/bmtcGtfs.ts';

// Keeps the dataset metadata (source, licence, freshness, counts) in the app database.
// The route and stop rows themselves are served from the cached dataset in bmtcQuery,
// because writing ~12,000 large records exceeds the platform's entity write-volume limit.
export default async function (req) {
  try {
    const base44 = createClientFromRequest(req);
    let payload = {};
    try {
      payload = await req.json();
    } catch (parseError) {
      payload = {};
    }
    const action = payload.action || 'status';

    const readRecord = async () => {
      const page = await base44.asServiceRole.entities.TransitDataset.filter({ source_id: SOURCE_ID }, { limit: 1 });
      return (page.items || [])[0] || null;
    };

    const writeRecord = async (patch) => {
      const existing = await readRecord();
      const body = {
        source_id: SOURCE_ID,
        source_name: SOURCE_NAME,
        source_url: REPO_URL,
        data_url: ROUTES_CSV_URL,
        licence: LICENCE,
        ...patch,
      };
      if (existing) return await base44.asServiceRole.entities.TransitDataset.update(existing.id, body);
      return await base44.asServiceRole.entities.TransitDataset.create({ status: 'empty', ...body });
    };

    if (action === 'load' || action === 'verify') {
      const probe = await fetch(ROUTES_CSV_URL, {
        headers: { Range: 'bytes=0-400' },
        signal: AbortSignal.timeout(20000),
      });
      const reachable = probe.ok || probe.status === 206;
      if (!reachable) {
        const record = await writeRecord({ status: 'failed', notes: `Source returned ${probe.status}` });
        return Response.json({ ok: false, reachable: false, dataset: record }, { status: 502 });
      }
      // Read the dataset once and record its real counts and freshness.
      const dataset = await loadDataset();
      const counts = routeCounts(dataset);
      const commitDate = await latestCommitDate();
      const record = await writeRecord({
        status: 'ready',
        route_count: counts.routes,
        stop_count: counts.stops,
        imported_at: new Date().toISOString(),
        last_checked_at: new Date().toISOString(),
        ...(commitDate ? { commit_date: commitDate } : {}),
        notes: 'Served from the published csv/routes.csv. Departure times are origin departures exactly as published; the dataset authors state the source data can be inaccurate.',
      });
      return Response.json({ ok: true, reachable: true, commitDate, dataset: record, counts });
    }

    if (action === 'status') {
      const record = await readRecord();
      return Response.json({ ok: true, dataset: record });
    }

    return Response.json({ error: `Unknown action: ${action}` }, { status: 400 });
  } catch (error) {
    return Response.json({ error: error.message }, { status: 500 });
  }
}
#shared
#bmtcGtfs
// Shared BMTC GTFS helpers used by the sync and query functions.
// The dataset is read from the published source and held in the worker's memory,
// because storing ~12,000 large records in the app database trips the platform's
// entity write-volume limit.

export const ROUTES_CSV_URL = 'https://raw.githubusercontent.com/Vonter/bmtc-gtfs/main/csv/routes.csv';
export const REPO_URL = 'https://github.com/Vonter/bmtc-gtfs';
export const COMMITS_API = 'https://api.github.com/repos/Vonter/bmtc-gtfs/commits?path=csv/routes.csv&per_page=1';
export const COMMITS_FEED = 'https://github.com/Vonter/bmtc-gtfs/commits/main/csv/routes.csv.atom';
export const LICENCE = 'ODbL 1.0';
export const SOURCE_ID = 'vonter-bmtc-gtfs';
export const SOURCE_NAME = 'bmtc-gtfs — unofficial BMTC GTFS dataset (built from the official Namma BMTC app)';

const MAX_ROUTE_STOPS = 120;
const MAX_ROUTE_TIMES = 120;
const DISPLAY_STOPS = 60;
const DISPLAY_TIMES = 12;
const DISPLAY_STOP_ROUTES = 40;
const STALE_MINUTES = 30;

let cache = null;

const parseCsv = (text) => {
  const rows = [];
  let row = [];
  let field = '';
  let quoted = false;
  for (let i = 0; i < text.length; i += 1) {
    const char = text[i];
    if (quoted) {
      if (char === '"') {
        if (text[i + 1] === '"') {
          field += '"';
          i += 1;
        } else {
          quoted = false;
        }
      } else {
        field += char;
      }
    } else if (char === '"') {
      quoted = true;
    } else if (char === ',') {
      row.push(field);
      field = '';
    } else if (char === '\n') {
      row.push(field);
      rows.push(row);
      row = [];
      field = '';
    } else if (char !== '\r') {
      field += char;
    }
  }
  if (field.length > 0 || row.length > 0) {
    row.push(field);
    rows.push(row);
  }
  return rows;
};

// Dataset list fields are written as ['A', 'B'] inside a quoted CSV field.
const parseList = (value) => {
  if (!value) return [];
  const trimmed = String(value).trim();
  if (trimmed.length < 2 || trimmed[0] !== '[') return [];
  const inner = trimmed.slice(1, -1);
  if (!inner) return [];
  return inner
    .split("', '")
    .map((item) => item.replace(/^'/, '').replace(/'$/, '').trim())
    .filter((item) => item.length > 0);
};

const build = (rows) => {
  const header = rows[0].map((cell) => cell.trim());
  const at = (name) => header.indexOf(name);
  const iName = at('name');
  const iFull = at('full_name');
  const iTripCount = at('trip_count');
  const iTripList = at('trip_list');
  const iStopCount = at('stop_count');
  const iStopList = at('stop_list');
  const iDirection = at('direction_id');

  const routes = [];
  const stopRoutes = new Map();

  for (let i = 1; i < rows.length; i += 1) {
    const cells = rows[i];
    if (!cells || cells.length < header.length) continue;
    const routeNumber = (cells[iName] || '').trim();
    if (!routeNumber) continue;
    const stops = parseList(cells[iStopList]);
    const times = parseList(cells[iTripList]);
    const route = {
      route_number: routeNumber,
      full_name: (cells[iFull] || '').trim(),
      direction_id: String(cells[iDirection] ?? '').trim(),
      origin_name: stops[0] || '',
      destination_name: stops[stops.length - 1] || '',
      stop_count: Number(cells[iStopCount]) || stops.length,
      trip_count: Number(cells[iTripCount]) || times.length,
      stop_list: stops.slice(0, MAX_ROUTE_STOPS),
      trip_list: times.slice(0, MAX_ROUTE_TIMES),
      first_departure: times[0] || '',
      last_departure: times[times.length - 1] || '',
    };
    routes.push(route);
    route.stop_list.forEach((stopName) => {
      const name = stopName.trim();
      if (!name) return;
      if (!stopRoutes.has(name)) stopRoutes.set(name, new Set());
      stopRoutes.get(name).add(routeNumber);
    });
  }

  const stops = Array.from(stopRoutes.entries())
    .map(([name, numbers]) => {
      const list = Array.from(numbers).sort((a, b) => a.localeCompare(b));
      return { name, route_count: list.length, route_numbers: list };
    })
    .sort((a, b) => a.name.localeCompare(b.name));

  const byNumber = new Map();
  routes.forEach((route) => {
    const list = byNumber.get(route.route_number) ?? [];
    list.push(route);
    byNumber.set(route.route_number, list);
  });

  return { routes, stops, stopRoutes, byNumber, loadedAt: new Date().toISOString() };
};

export async function loadDataset() {
  if (cache && Date.now() - new Date(cache.loadedAt).getTime() < STALE_MINUTES * 60000) {
    return { ...cache, fromCache: true };
  }
  const response = await fetch(ROUTES_CSV_URL, { signal: AbortSignal.timeout(45000) });
  if (!response.ok) throw new Error(`The source returned ${response.status}`);
  const text = await response.text();
  cache = build(parseCsv(text));
  return { ...cache, fromCache: false };
}

export function routeCounts(dataset) {
  return { routes: dataset.routes.length, stops: dataset.stops.length };
}

export const isRouteSummary = (route) => ({
  route_number: route.route_number,
  full_name: route.full_name,
  direction_id: route.direction_id,
  origin_name: route.origin_name,
  destination_name: route.destination_name,
  stop_count: route.stop_count,
  trip_count: route.trip_count,
  first_departure: route.first_departure,
  last_departure: route.last_departure,
  stop_list: route.stop_list.slice(0, DISPLAY_STOPS),
  trip_list: route.trip_list.slice(0, DISPLAY_TIMES),
});

export function findRoutes(dataset, query, limit = 20) {
  const needle = String(query ?? '').trim().toLowerCase();
  if (needle.length < 2) return [];
  const starts = [];
  const contains = [];
  dataset.routes.forEach((route) => {
    const number = route.route_number.toLowerCase();
    if (number.startsWith(needle)) {
      starts.push(route);
      return;
    }
    if (number.includes(needle) || route.full_name.toLowerCase().includes(needle)) contains.push(route);
  });
  return [...starts, ...contains].slice(0, limit).map((route) => isRouteSummary(route));
}

export function findStops(dataset, query, limit = 20) {
  const needle = String(query ?? '').trim().toLowerCase();
  if (needle.length < 2) return [];
  const starts = [];
  const contains = [];
  dataset.stops.forEach((stop) => {
    const name = stop.name.toLowerCase();
    if (name.startsWith(needle)) starts.push(stop);
    else if (name.includes(needle)) contains.push(stop);
  });
  return [...starts, ...contains].slice(0, limit).map((stop) => ({
    name: stop.name,
    route_count: stop.route_count,
    route_numbers: stop.route_numbers.slice(0, DISPLAY_STOP_ROUTES),
  }));
}

export function planJourney(dataset, fromName, toName, limit = 8) {
  const fromRoutes = dataset.stopRoutes.get(fromName);
  const toRoutes = dataset.stopRoutes.get(toName);
  if (!fromRoutes || !toRoutes) return [];

  const options = [];
  dataset.routes.forEach((route) => {
    if (!fromRoutes.has(route.route_number) || !toRoutes.has(route.route_number)) return;
    const boardIndex = route.stop_list.indexOf(fromName);
    const alightIndex = route.stop_list.indexOf(toName);
    if (boardIndex === -1 || alightIndex === -1 || boardIndex >= alightIndex) return;
    options.push({
      route: isRouteSummary(route),
      board_name: fromName,
      alight_name: toName,
      boardIndex,
      alightIndex,
      stopsBetween: alightIndex - boardIndex,
      departures: route.trip_list.slice(0, DISPLAY_TIMES),
    });
  });

  return options.sort((a, b) => a.stopsBetween - b.stopsBetween).slice(0, limit);
}

// --------------------------------------------------------------- journeys
// The dataset publishes the stop order and the origin departure times of each
// route direction, but no stop-to-stop running times. Every duration below is
// therefore an estimate built from the published stop count at a stated pace.
export const ASSUMPTIONS = {
  paceMinutesPerStop: 2.5,
  maxWaitEstimate: 90,
  durations: 'Estimated from the published number of stops at an assumed average pace of 2.5 minutes per stop. The dataset has no stop-to-stop running times.',
  waiting: 'Waiting is estimated from the published origin departures of each route direction. Live arrivals are not available, so waiting cannot be verified.',
  walking: 'Not estimated: the dataset has no stop distances or walking times.',
  schedule: 'Times published by the dataset are origin departure times for the whole trip.',
};

const clockMinutes = (value) => {
  const parts = String(value || '').split(':');
  if (parts.length < 2) return NaN;
  const hours = Number(parts[0]);
  const minutes = Number(parts[1]);
  if (!Number.isFinite(hours) || !Number.isFinite(minutes)) return NaN;
  return hours * 60 + minutes;
};

// Median gap between the published origin departures of one route direction.
export const headwayMinutes = (tripList) => {
  if (!Array.isArray(tripList) || tripList.length < 3) return null;
  const minutes = tripList.map(clockMinutes).filter((value) => Number.isFinite(value)).sort((a, b) => a - b);
  if (minutes.length < 3) return null;
  const gaps = [];
  for (let i = 1; i < minutes.length; i += 1) {
    const gap = minutes[i] - minutes[i - 1];
    if (gap > 0 && gap <= 120) gaps.push(gap);
  }
  if (gaps.length < 2) return null;
  gaps.sort((a, b) => a - b);
  const middle = Math.floor(gaps.length / 2);
  return gaps.length % 2 === 1 ? gaps[middle] : (gaps[middle - 1] + gaps[middle]) / 2;
};

const minutesToClock = (value) => {
  const wrapped = ((Math.round(value) % 1440) + 1440) % 1440;
  return `${String(Math.floor(wrapped / 60)).padStart(2, '0')}:${String(wrapped % 60).padStart(2, '0')}`;
};

const legsToJourney = (legs, departMinutes, via) => {
  const built = [];
  let clock = Number.isFinite(departMinutes) ? departMinutes : null;
  let totalRide = 0;
  let totalWait = 0;
  let waitKnown = true;
  let afterLast = false;

  legs.forEach((leg) => {
    const stops = leg.alightIndex - leg.boardIndex;
    const ride = Math.max(1, Math.round(stops * ASSUMPTIONS.paceMinutesPerStop));
    const times = Array.isArray(leg.route.trip_list)
      ? leg.route.trip_list.map(clockMinutes).filter((value) => Number.isFinite(value)).sort((a, b) => a - b)
      : [];
    const headway = headwayMinutes(leg.route.trip_list);
    const first = times[0];
    const last = times[times.length - 1];

    let wait = null;
    let basis = null;
    let nextDeparture = null;
    let ended = false;

    if (clock === null) {
      wait = null;
    } else if (leg.boardIndex === 0) {
      // Boarding where the trip starts, so the published departure times are the
      // boarding times and the wait is exact.
      const next = times.find((value) => value >= clock);
      if (next !== undefined) {
        wait = next - clock;
        basis = 'published';
        nextDeparture = minutesToClock(next);
      } else if (last !== undefined) {
        ended = true;
      }
    } else if (last !== undefined && clock > last) {
      ended = true;
    } else if (headway) {
      wait = Math.min(ASSUMPTIONS.maxWaitEstimate, Math.round(headway / 2));
      basis = 'headway';
    } else if (first !== undefined && clock <= first) {
      wait = Math.min(ASSUMPTIONS.maxWaitEstimate, first - clock);
      basis = 'first';
    }

    if (wait === null && !ended) waitKnown = false;
    if (ended) afterLast = true;
    totalRide += ride;
    totalWait += wait ?? 0;
    built.push({
      route_number: leg.route.route_number,
      full_name: leg.route.full_name,
      direction_id: leg.route.direction_id,
      board_name: leg.route.stop_list[leg.boardIndex],
      alight_name: leg.route.stop_list[leg.alightIndex],
      stops_traversed: stops,
      ride_minutes: ride,
      headway_minutes: headway,
      expected_wait_minutes: wait,
      wait_known: wait !== null,
      wait_basis: basis,
      next_departure: nextDeparture,
      service_ended: ended,
      first_departure: leg.route.first_departure,
      last_departure: leg.route.last_departure,
      departures: leg.route.trip_list.slice(0, 4),
    });
    if (clock !== null) clock += ride + (wait ?? 0);
  });

  return {
    transfers: built.length - 1,
    legs: built,
    board_name: built[0]?.board_name || '',
    alight_name: built[built.length - 1]?.alight_name || '',
    interchange_name: built.length > 1 ? built[0].alight_name : '',
    via_name: via || '',
    total_ride_minutes: totalRide,
    total_wait_minutes: totalWait,
    total_minutes: totalRide + totalWait,
    wait_known: waitKnown,
    after_last_service: afterLast,
    route_numbers: built.map((leg) => leg.route_number),
  };
};

// Direct and single-transfer journeys between two published stops.
export function planCandidates(dataset, { from, to, via, departMinutes, limit = 60 }) {
  const fromRoutes = dataset.stopRoutes.get(from);
  const toRoutes = dataset.stopRoutes.get(to);
  if (!fromRoutes || !toRoutes) return [];

  const out = [];
  const seen = new Set();

  dataset.routes.forEach((route) => {
    if (out.length > 300) return;
    if (!fromRoutes.has(route.route_number) || !toRoutes.has(route.route_number)) return;
    const boardIndex = route.stop_list.indexOf(from);
    const alightIndex = route.stop_list.indexOf(to);
    if (boardIndex === -1 || alightIndex === -1 || boardIndex >= alightIndex) return;
    if (via) {
      const viaIndex = route.stop_list.indexOf(via);
      if (viaIndex === -1 || viaIndex <= boardIndex || viaIndex >= alightIndex) return;
    }
    seen.add(`${route.route_number}|${from}>${to}|direct`);
    out.push(legsToJourney([{ route, boardIndex, alightIndex }], departMinutes, via));
  });

  const firstLegs = dataset.routes
    .filter((route) => fromRoutes.has(route.route_number))
    .slice(0, 400);

  firstLegs.forEach((first) => {
    if (out.length > 400) return;
    const boardIndex = first.stop_list.indexOf(from);
    if (boardIndex === -1) return;
    const lastScan = Math.min(first.stop_list.length - 1, boardIndex + 30);

    for (let i = boardIndex + 1; i <= lastScan; i += 1) {
      const interchange = first.stop_list[i];
      if (!interchange || interchange === from || interchange === to) continue;
      const served = dataset.stopRoutes.get(interchange);
      if (!served) continue;

      let connected = null;
      const shared = Array.from(served).filter((number) => toRoutes.has(number));
      for (let s = 0; s < shared.length && !connected; s += 1) {
        const secondOptions = dataset.byNumber.get(shared[s]) || [];
        for (let r = 0; r < secondOptions.length; r += 1) {
          const second = secondOptions[r];
          const board2 = second.stop_list.indexOf(interchange);
          const alight2 = second.stop_list.indexOf(to);
          if (board2 === -1 || alight2 === -1 || board2 >= alight2) continue;
          if (via) {
            const inFirst = via !== interchange && first.stop_list.slice(boardIndex, i + 1).includes(via);
            const inSecond = second.stop_list.slice(board2, alight2 + 1).includes(via);
            if (!inFirst && !inSecond) continue;
          }
          connected = { route: second, boardIndex: board2, alightIndex: alight2 };
          break;
        }
      }
      if (!connected) continue;

      const key = `${first.route_number}|${from}>${interchange}|${connected.route.route_number}`;
      if (seen.has(key)) continue;
      seen.add(key);
      out.push(
        legsToJourney(
          [{ route: first, boardIndex, alightIndex: i }, connected],
          departMinutes,
          via,
        ),
      );
    }
  });

  return out.sort((a, b) => a.total_minutes - b.total_minutes).slice(0, limit);
}

export async function latestCommitDate() {
  try {
    const response = await fetch(COMMITS_API, { signal: AbortSignal.timeout(15000) });
    if (response.ok) {
      const commits = await response.json();
      if (Array.isArray(commits) && commits[0]?.commit?.committer?.date) {
        return commits[0].commit.committer.date;
      }
    }
  } catch (error) {
    // fall through to the public feed
  }
  try {
    const feed = await fetch(COMMITS_FEED, { signal: AbortSignal.timeout(15000) });
    if (feed.ok) {
      const atom = await feed.text();
      const match = atom.match(/<updated>([^<]+)<\/updated>/);
      if (match) return match[1];
    }
  } catch (error) {
    return null;
  }
  return null;
}
#config
{
  "name": "base44-app",
  "site": {
    "installCommand": "npm install",
    "buildCommand": "npm run build",
    "serveCommand": "npm run dev",
    "outputDirectory": "./dist"
  }
}

#src
#api
#base44Client
import { createClient } from '@base44/sdk';
import { appParams } from '@/lib/app-params';

const { appId, token, functionsVersion, appBaseUrl } = appParams;

export const base44 = createClient({
  appId,
  token,
  functionsVersion,
  serverUrl: '',
  appBaseUrl
});

#components
#planner
#AlternativeTransport
import { Car, ExternalLink, Footprints, TrainFront } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';

const MODES = [
  { key: 'metro', icon: TrainFront },
  { key: 'auto', icon: Car },
  { key: 'taxi', icon: Car },
  { key: 'walk', icon: Footprints },
];

export default function AlternativeTransport({ from, to }) {
  const { t } = useLanguage();
  const manualUrl = `https://www.google.com/maps/dir/?api=1&origin=${encodeURIComponent(from || '')}&destination=${encodeURIComponent(to || '')}&travelmode=transit`;

  return (
    <section className="rounded-2xl border border-border/70 bg-card p-4">
      <h3 className="font-display text-sm font-semibold">{t('planner.alts.title')}</h3>
      <p className="mt-1 text-xs text-muted-foreground">{t('planner.alts.body')}</p>

      <ul className="mt-3 space-y-2">
        {MODES.map(({ key, icon: Icon }) => (
          <li key={key} className="flex items-start gap-2 text-xs">
            <Icon className="mt-0.5 h-3.5 w-3.5 shrink-0 text-muted-foreground" />
            <span>
              <span className="font-medium text-foreground">{t(`planner.alts.${key}`)}</span>
              <span className="text-muted-foreground"> — {t(`planner.alts.${key}Note`)}</span>
            </span>
          </li>
        ))}
      </ul>

      <a
        href={manualUrl}
        target="_blank"
        rel="noreferrer"
        className="mt-3 inline-flex items-center gap-1.5 text-xs font-medium text-primary hover:underline"
      >
        {t('planner.alts.manual')}
        <ExternalLink className="h-3.5 w-3.5" />
      </a>
    </section>
  );
}


  }
}
#CrowdEvidencePanel
import { AlertTriangle, Users } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import { ageLabel, levelMeta, isStale } from '@/lib/crowd';

const CONFIDENCE_KEY = {
  recent: 'planner.crowd.conf.recent',
  limited: 'planner.crowd.conf.limited',
  conflicting: 'planner.crowd.conf.conflicting',
  none: 'planner.crowd.conf.none',
};

export default function CrowdEvidencePanel({ crowd, boardName }) {
  const { t } = useLanguage();
  const meta = crowd.level ? levelMeta(crowd.level) : null;

  return (
    <div className="rounded-xl border border-border/70 bg-muted/20 p-3">
      <div className="flex flex-wrap items-center gap-2">
        <span className="inline-flex items-center gap-1.5 text-xs font-medium text-muted-foreground">
          <Users className="h-3.5 w-3.5" />
          {t('planner.crowd.title')}
        </span>
        {meta ? (
          <span className={`inline-flex items-center gap-1.5 rounded-full border px-2.5 py-0.5 text-xs font-medium ${meta.chip}`}>
            <span className={`h-1.5 w-1.5 rounded-full ${meta.dot}`} />
            {t(meta.labelKey)}
          </span>
        ) : (
          <span className="rounded-full border border-border px-2.5 py-0.5 text-xs text-muted-foreground">
            {t('planner.crowd.none')}
          </span>
        )}
        <span className={`rounded-full border px-2.5 py-0.5 text-[11px] ${crowd.confidence === 'recent' ? 'border-crowd-low/40 text-crowd-low' : 'border-border text-muted-foreground'}`}>
          {t(CONFIDENCE_KEY[crowd.confidence] ?? CONFIDENCE_KEY.none)}
        </span>
      </div>

      <dl className="mt-2.5 grid gap-x-4 gap-y-1 text-xs sm:grid-cols-2">
        <div className="flex gap-1.5">
          <dt className="text-muted-foreground">{t('planner.crowd.countLabel')}</dt>
          <dd className="font-medium text-foreground">
            {t('planner.crowd.count', { recent: crowd.recentCount, older: crowd.totalCount - crowd.recentCount })}
          </dd>
        </div>
        {crowd.latestAt && (
          <div className="flex gap-1.5">
            <dt className="text-muted-foreground">{t('planner.crowd.latestLabel')}</dt>
            <dd className="font-medium text-foreground">
              {t('planner.crowd.latest', {
                age: ageLabel(crowd.latestAt, t),
                level: crowd.latestLevel ? t(`report.${crowd.latestLevel}`) : '—',
              })}
            </dd>
          </div>
        )}
        {crowd.atStopCount > 0 && (
          <div className="flex gap-1.5 sm:col-span-2">
            <dt className="text-muted-foreground">{t('planner.crowd.atStopLabel')}</dt>
            <dd className="font-medium text-foreground">
              {t('planner.crowd.atStop', {
                count: crowd.atStopCount,
                stop: boardName || crowd.latestStop || '',
                age: crowd.atStopLatestAt ? ageLabel(crowd.atStopLatestAt, t) : '',
              })}
            </dd>
          </div>
        )}
      </dl>

      {crowd.confidence === 'none' && (
        <p className="mt-2 text-xs text-muted-foreground">{t('planner.crowd.noneNote')}</p>
      )}
      {crowd.confidence === 'limited' && (
        <p className="mt-2 text-xs text-muted-foreground">{t('planner.crowd.limitedNote')}</p>
      )}
      {crowd.confidence === 'conflicting' && (
        <p className="mt-2 inline-flex items-start gap-1.5 text-xs text-crowd-moderate">
          <AlertTriangle className="mt-0.5 h-3.5 w-3.5 shrink-0" />
          {t('planner.crowd.conflictNote')}
        </p>
      )}
      {crowd.latestAt && isStale(crowd.latestAt) && crowd.recentCount === 0 && (
        <p className="mt-2 text-xs text-muted-foreground">{t('planner.crowd.staleNote', { age: ageLabel(crowd.latestAt, t) })}</p>
      )}
      {crowd.mixed && <p className="mt-2 text-xs text-muted-foreground">{t('planner.crowd.mixedNote')}</p>}
    </div>
  );
}
#JourneyOptionCard
import { ArrowRight, Bus, Minus, Plus } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import { formatClock } from '@/lib/transit';
import { minutesToClock } from '@/lib/journey';
import CrowdEvidencePanel from '@/components/planner/CrowdEvidencePanel';

const waitText = (leg, t) => {
  if (!leg.wait_known) return t('planner.waitUnverified');
  if (leg.wait_basis === 'published') return t('planner.waitPublished', { time: leg.next_departure });
  if (leg.wait_basis === 'headway') return t('planner.waitHeadway', { minutes: leg.expected_wait_minutes });
  if (leg.wait_basis === 'first') return t('planner.waitFirst', { time: formatClock(leg.first_departure) });
  return t('planner.waitUnverified');
};

export default function JourneyOptionCard({ option, explanation, quality, badge, highlighted = false, departMinutes }) {
  const { t } = useLanguage();
  const journey = option.journey;
  const legs = journey.legs;
  const arrival = Number.isFinite(departMinutes) ? departMinutes + journey.total_minutes : null;
  const firstLeg = legs[0];

  return (
    <article className={`rounded-2xl border p-4 ${highlighted ? 'border-primary bg-primary/5 shadow-sm' : 'border-border/70 bg-card'}`}>
      <header className="flex flex-wrap items-center gap-2">
        {legs.map((leg, index) => (
          <span key={`${leg.route_number}-${index}`} className="inline-flex items-center gap-1.5">
            {index > 0 && <ArrowRight className="h-3.5 w-3.5 text-muted-foreground" />}
            <span className="inline-flex items-center gap-1 rounded-lg bg-primary px-2 py-0.5 font-display text-sm font-semibold text-primary-foreground">
              <Bus className="h-3.5 w-3.5" />
              {leg.route_number}
            </span>
          </span>
        ))}
        {badge && (
          <span className="rounded-full border border-primary/40 bg-primary/10 px-2.5 py-0.5 text-[11px] font-medium text-primary">{badge}</span>
        )}
        <span className="ml-auto text-xs font-semibold text-foreground">
          {t('planner.totalTime', { minutes: journey.total_minutes })}
        </span>
      </header>

      <ol className="mt-3 space-y-2">
        {legs.map((leg, index) => (
          <li key={`${leg.route_number}-leg-${index}`} className="rounded-xl border border-border/60 bg-background p-3 text-xs">
            <p className="font-medium text-foreground">{t('planner.leg', { n: index + 1, route: leg.route_number })}</p>
            <p className="mt-0.5 text-muted-foreground">{leg.full_name}</p>
            <div className="mt-2 grid gap-x-4 gap-y-1 sm:grid-cols-2">
              <p className="text-foreground">{t('plan.boardAt', { stop: leg.board_name })}</p>
              <p className="text-foreground">{t('plan.alightAt', { stop: leg.alight_name })}</p>
              <p className="text-muted-foreground">{t('planner.rideStops', { stops: leg.stops_traversed, minutes: leg.ride_minutes })}</p>
              <p className="text-muted-foreground">
                {t('planner.waitLabel')}: {waitText(leg, t)}
              </p>
            </div>
          </li>
        ))}
      </ol>

      <div className="mt-3 grid gap-2 text-xs sm:grid-cols-4">
        <div>
          <p className="text-muted-foreground">{t('planner.estimatedDeparture')}</p>
          <p className="font-medium text-foreground">{Number.isFinite(departMinutes) ? minutesToClock(departMinutes) : '—'}</p>
        </div>
        <div>
          <p className="text-muted-foreground">{journey.wait_known ? t('planner.estimatedArrival') : t('planner.earliestArrival')}</p>
          <p className="font-medium text-foreground">{arrival !== null ? minutesToClock(arrival) : '—'}</p>
        </div>
        <div>
          <p className="text-muted-foreground">{t('planner.rideTime')}</p>
          <p className="font-medium text-foreground">{t('planner.minutesShort', { count: journey.total_ride_minutes })}</p>
        </div>
        <div>
          <p className="text-muted-foreground">{t('planner.waitTime')}</p>
          <p className="font-medium text-foreground">
            {journey.wait_known ? t('planner.minutesShort', { count: journey.total_wait_minutes }) : t('planner.waitUnverified')}
          </p>
        </div>
      </div>

      <p className="mt-2 text-xs text-muted-foreground">
        {journey.transfers === 0
          ? t('planner.transfersNone')
          : t('planner.transfersCount', { count: journey.transfers, interchange: journey.interchange_name })}
        {' · '}
        {t('planner.walkUnknown')}
      </p>

      {firstLeg?.first_departure && (
        <p className="mt-1 text-xs text-muted-foreground">
          {t('planner.scheduleWindow', { first: formatClock(firstLeg.first_departure), last: formatClock(firstLeg.last_departure) })}
          {firstLeg.headway_minutes ? ` · ${t('planner.frequency', { minutes: Math.round(firstLeg.headway_minutes) })}` : ''}
        </p>
      )}

      {explanation && (
        <div className="mt-3 rounded-xl border border-border/60 bg-muted/20 p-3">
          <p className="text-xs font-semibold text-foreground">{t('planner.why')}</p>
          <p className="mt-1 text-xs text-muted-foreground">{t(explanation.headline, explanation.headlineVars)}</p>
          <ul className="mt-2 space-y-1 text-xs">
            {explanation.advantages.map((point) => (
              <li key={point.key} className="flex items-start gap-1.5 text-foreground">
                <Plus className="mt-0.5 h-3 w-3 shrink-0 text-crowd-low" />
                {t(point.key, point.vars)}
              </li>
            ))}
            {explanation.drawbacks.map((point) => (
              <li key={point.key} className="flex items-start gap-1.5 text-muted-foreground">
                <Minus className="mt-0.5 h-3 w-3 shrink-0" />
                {t(point.key, point.vars)}
              </li>
            ))}
          </ul>
        </div>
      )}

      <div className="mt-3 flex flex-wrap gap-1.5">
        {quality.map((item) => (
          <span key={item.key} className="rounded-full border border-border px-2.5 py-0.5 text-[11px] text-muted-foreground">
            {t(item.key, item.vars)}
          </span>
        ))}
      </div>

      <div className="mt-3">
        <CrowdEvidencePanel crowd={option.crowd} boardName={journey.board_name} />
      </div>
    </article>
  );
}
#JourneyStep
import { ArrowDown } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import StopPicker from '@/components/StopPicker';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';

export default function JourneyStep({ form, update, errors }) {
  const { t } = useLanguage();

  return (
    <div className="space-y-5">
      <div className="grid gap-3 sm:grid-cols-[1fr_auto_1fr] sm:items-end">
        <StopPicker
          id="planner-from"
          label={t('plan.from')}
          placeholder={t('plan.chooseFrom')}
          value={form.from}
          onSelect={(stop) => update({ from: stop })}
        />
        <button
          type="button"
          onClick={() => update({ from: form.to, to: form.from })}
          aria-label={t('plan.swap')}
          className="mx-auto hidden h-9 w-9 items-center justify-center rounded-full border border-border text-muted-foreground hover:bg-accent sm:flex"
        >
          <ArrowDown className="h-4 w-4" />
        </button>
        <StopPicker
          id="planner-to"
          label={t('plan.to')}
          placeholder={t('plan.chooseTo')}
          value={form.to}
          onSelect={(stop) => update({ to: stop })}
        />
      </div>

      <div>
        <StopPicker
          id="planner-via"
          label={t('planner.via')}
          placeholder={t('planner.viaPlaceholder')}
          value={form.via}
          onSelect={(stop) => update({ via: stop })}
        />
        <p className="mt-1 text-xs text-muted-foreground">{t('planner.viaHint')}</p>
      </div>

      <fieldset className="space-y-3">
        <legend className="font-display text-sm font-semibold">{t('planner.departTitle')}</legend>
        <div className="flex flex-wrap gap-2">
          {['now', 'custom'].map((mode) => (
            <button
              key={mode}
              type="button"
              onClick={() => update({ departMode: mode })}
              aria-pressed={form.departMode === mode}
              className={`rounded-full border px-3 py-1.5 text-xs font-medium transition-colors ${
                form.departMode === mode
                  ? 'border-primary bg-primary text-primary-foreground'
                  : 'border-border text-foreground hover:bg-accent'
              }`}
            >
              {t(mode === 'now' ? 'planner.leaveNow' : 'planner.chooseTime')}
            </button>
          ))}
        </div>
        {form.departMode === 'custom' && (
          <Input
            id="planner-depart-time"
            type="datetime-local"
            value={form.departDateTime}
            onChange={(event) => update({ departDateTime: event.target.value })}
            className="h-9 max-w-xs"
          />
        )}
      </fieldset>

      <fieldset className="space-y-3">
        <div className="flex flex-wrap items-center gap-3">
          <legend className="font-display text-sm font-semibold">{t('planner.deadlineTitle')}</legend>
          <label className="inline-flex items-center gap-2 text-xs text-muted-foreground">
            <input
              type="checkbox"
              checked={form.deadlineEnabled}
              onChange={(event) => update({ deadlineEnabled: event.target.checked, deadlineDateTime: '' })}
              className="h-3.5 w-3.5 rounded border-input"
            />
            {t('planner.deadlineToggle')}
          </label>
        </div>
        {form.deadlineEnabled && (
          <div className="space-y-1">
            <Label htmlFor="planner-deadline" className="text-xs text-muted-foreground">
              {t('planner.deadlineLabel')}
            </Label>
            <Input
              id="planner-deadline"
              type="datetime-local"
              value={form.deadlineDateTime}
              onChange={(event) => update({ deadlineDateTime: event.target.value })}
              className="h-9 max-w-xs"
            />
          </div>
        )}
        <p className="text-xs text-muted-foreground">{t('planner.deadlineHint')}</p>
      </fieldset>

      <p className="rounded-xl border border-border/70 bg-muted/30 p-3 text-xs text-muted-foreground">
        {t('planner.stopSourceNote')}
      </p>

      {errors.length > 0 && (
        <ul className="space-y-1 text-sm text-destructive">
          {errors.map((key) => (
            <li key={key}>{t(key)}</li>
          ))}
        </ul>
      )}
    </div>
  );
}
#LimitsStep
import { useLanguage } from '@/lib/LanguageContext';
import TimeLimitPicker from '@/components/planner/TimeLimitPicker';

const TRAVEL_PRESETS = [15, 30, 45, 60, 90, 120];
const WAIT_PRESETS = [5, 10, 15, 20, 30];

export default function LimitsStep({ form, update, errors, hasError }) {
  const { t } = useLanguage();
  const travelError = hasError('planner.err.travel') ? t('planner.err.travel') : null;
  const waitError = hasError('planner.err.wait') ? t('planner.err.wait') : null;

  return (
    <div className="space-y-4">
      <TimeLimitPicker
        id="planner-travel"
        title={t('planner.travel.title')}
        hint={t('planner.travel.hint')}
        presets={TRAVEL_PRESETS}
        value={form.maxTravelMinutes}
        onChange={(value) => update({ maxTravelMinutes: value })}
        error={travelError}
        max={360}
      />

      <TimeLimitPicker
        id="planner-wait"
        title={t('planner.wait.title')}
        hint={t('planner.wait.hint')}
        presets={WAIT_PRESETS}
        value={form.maxWaitMinutes}
        onChange={(value) => update({ maxWaitMinutes: value })}
        error={waitError}
        max={180}
      />

      <p className="rounded-xl border border-crowd-moderate/30 bg-crowd-moderate/10 p-3 text-xs text-foreground">
        {t('planner.wait.caveat')}
      </p>
    </div>
  );
}
#PlannerStepper
import { Check } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';

const STEPS = ['journey', 'limits', 'priority', 'results'];

export default function PlannerStepper({ step, onSelect }) {
  const { t } = useLanguage();

  return (
    <ol className="flex flex-wrap items-center gap-x-2 gap-y-2 text-xs">
      {STEPS.map((key, index) => {
        const number = index + 1;
        const done = step > number;
        const active = step === number;
        const style = active
          ? 'border-primary bg-primary/10 text-primary'
          : done
            ? 'border-crowd-low/40 bg-crowd-low/10 text-foreground'
            : 'border-border text-muted-foreground';
        return (
          <li key={key} className="flex items-center gap-2">
            <button
              type="button"
              onClick={() => done && onSelect(number)}
              disabled={!done}
              className={`inline-flex items-center gap-1.5 rounded-full border px-3 py-1 font-medium transition-colors ${style} ${done ? 'hover:bg-accent' : ''}`}
            >
              {done ? <Check className="h-3 w-3" /> : <span aria-hidden="true">{number}</span>}
              {t(`planner.step.${key}`)}
            </button>
            {number < STEPS.length && <span aria-hidden="true" className="h-px w-3 bg-border sm:w-6" />}
          </li>
        );
      })}
    </ol>
  );
}
#PlanResults
import { useLanguage } from '@/lib/LanguageContext';
import JourneyOptionCard from '@/components/planner/JourneyOptionCard';
import AlternativeTransport from '@/components/planner/AlternativeTransport';
import { dataQuality, explainOption, minutesToClock, reasonSummary } from '@/lib/journey';

export default function PlanResults({ result, form, onBack, onRestart }) {
  const { t } = useLanguage();
  const { prefs, recommended, picks, unverified, ineligible, notes, departMinutes, deadlineMinutes } = result;
  const evaluated = result.evaluated ?? [];
  const noCandidates = evaluated.length === 0;
  const others = (picks ?? []).filter((item) => item.option !== recommended);

  const explain = (option) => explainOption(option, { prefs, recommended, fastest: result.fastest, calm: result.calm });

  return (
    <div className="space-y-5">
      <div className="rounded-2xl border border-border/70 bg-muted/20 p-3 text-xs text-muted-foreground">
        <p className="font-medium text-foreground">
          {t('planner.results.summary', { from: form.from?.name ?? '', to: form.to?.name ?? '' })}
        </p>
        <p className="mt-1">
          {t('planner.results.prefs', {
            depart: Number.isFinite(departMinutes) ? minutesToClock(departMinutes) : '—',
            travel: form.maxTravelMinutes ?? '—',
            wait: form.maxWaitMinutes ?? '—',
            priority: t(`planner.priority.${prefs.priority}`),
          })}
        </p>
        {Number.isFinite(deadlineMinutes) && (
          <p className="mt-1">{t('planner.results.deadline', { time: minutesToClock(deadlineMinutes) })}</p>
        )}
        <p className="mt-1">{t('planner.results.count', { count: evaluated.length, eligible: result.eligible?.length ?? 0 })}</p>
      </div>

      {recommended ? (
        <section className="space-y-3">
          <h3 className="font-display text-lg font-semibold">{t('planner.recommended')}</h3>
          <JourneyOptionCard
            option={recommended}
            explanation={explain(recommended)}
            quality={dataQuality(recommended)}
            badge={t('planner.option.best')}
            highlighted
            departMinutes={departMinutes}
          />
          <p className="rounded-xl border border-crowd-low/30 bg-crowd-low/10 p-3 text-xs text-foreground">
            {t('planner.limitsOk', { travel: form.maxTravelMinutes, wait: form.maxWaitMinutes })}
          </p>
        </section>
      ) : (
        <section className="space-y-3 rounded-2xl border border-crowd-moderate/40 bg-crowd-moderate/10 p-4">
          <h3 className="font-display text-base font-semibold">{t('planner.noMatch.title')}</h3>
          <p className="text-sm text-foreground">{t(noCandidates ? 'planner.noMatch.noJourneys' : 'planner.noMatch.body')}</p>
          {!noCandidates && (
            <ul className="space-y-1 text-xs text-muted-foreground">
              {reasonSummary([...(ineligible ?? []), ...(unverified ?? [])]).map((item) => (
                <li key={item.reason}>{t(`planner.reason.${item.reason}`, { count: item.count })}</li>
              ))}
            </ul>
          )}
        </section>
      )}

      {others.length > 0 && (
        <section className="space-y-3">
          <h3 className="font-display text-sm font-semibold">{t('planner.alternatives')}</h3>
          {others.map(({ option, label }, index) => (
            <JourneyOptionCard
              key={`${label}-${index}-${option.journey.route_numbers.join('-')}`}
              option={option}
              explanation={explain(option)}
              quality={dataQuality(option)}
              badge={t(`planner.option.${label}`)}
              departMinutes={departMinutes}
            />
          ))}
        </section>
      )}

      {unverified?.length > 0 && (
        <section className="space-y-2">
          <h3 className="font-display text-sm font-semibold">{t('planner.unverified.title', { count: unverified.length })}</h3>
          <p className="text-xs text-muted-foreground">{t('planner.unverified.body')}</p>
          <ul className="space-y-1 text-xs text-foreground">
            {unverified.slice(0, 5).map((option, index) => (
              <li key={`unverified-${index}-${option.journey.route_numbers.join('-')}`}>
                {t('planner.unverified.item', {
                  routes: option.journey.route_numbers.join(' → '),
                  minutes: option.journey.total_ride_minutes,
                })}
              </li>
            ))}
          </ul>
        </section>
      )}

      {ineligible?.length > 0 && (
        <section className="space-y-2">
          <h3 className="font-display text-sm font-semibold">{t('planner.excluded.title', { count: ineligible.length })}</h3>
          <ul className="space-y-1 text-xs text-muted-foreground">
            {reasonSummary(ineligible).map((item) => (
              <li key={item.reason}>{t(`planner.reason.${item.reason}`, { count: item.count })}</li>
            ))}
          </ul>
        </section>
      )}

      {notes?.length > 0 && (
        <ul className="space-y-1.5 rounded-2xl border border-border/70 bg-card p-3 text-xs text-muted-foreground">
          {notes.map((note) => (
            <li key={note}>{t(`planner.note.${note}`)}</li>
          ))}
        </ul>
      )}

      {!recommended && <AlternativeTransport from={form.from?.name} to={form.to?.name} />}

      <section className="rounded-2xl border border-border/70 bg-muted/20 p-3 text-[11px] leading-relaxed text-muted-foreground">
        <p className="font-medium text-foreground">{t('planner.assumptions.title')}</p>
        <ul className="mt-1 space-y-1">
          <li>{t('planner.assumption.durations', { pace: result.assumptions?.paceMinutesPerStop ?? 2.5 })}</li>
          <li>{t('planner.assumption.waiting')}</li>
          <li>{t('planner.assumption.schedule')}</li>
          <li>{t('planner.assumption.walking')}</li>
        </ul>
      </section>

      <div className="flex flex-wrap gap-2">
        <button
          type="button"
          onClick={onBack}
          className="inline-flex items-center rounded-xl border border-border px-4 py-2 text-sm font-medium text-foreground hover:bg-accent"
        >
          {t('planner.changePriorities')}
        </button>
        <button
          type="button"
          onClick={onRestart}
          className="inline-flex items-center rounded-xl bg-primary px-4 py-2 text-sm font-medium text-primary-foreground"
        >
          {t('planner.restart')}
        </button>
      </div>
    </div>
  );
}
#PriorityStep
import { Gauge, HeartHandshake, Users } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';

const PRIORITIES = [
  { key: 'quickest', icon: Gauge },
  { key: 'crowd', icon: Users },
  { key: 'balanced', icon: HeartHandshake },
];

export default function PriorityStep({ form, update, errors }) {
  const { t } = useLanguage();

  return (
    <div className="space-y-4">
      <div>
        <h3 className="font-display text-base font-semibold">{t('planner.priority.title')}</h3>
        <p className="text-sm text-muted-foreground">{t('planner.priority.hint')}</p>
      </div>

      <div className="grid gap-3 sm:grid-cols-3">
        {PRIORITIES.map(({ key, icon: Icon }) => {
          const selected = form.priority === key;
          return (
            <button
              key={key}
              type="button"
              onClick={() => update({ priority: key })}
              aria-pressed={selected}
              className={`flex h-full flex-col items-start gap-3 rounded-2xl border p-4 text-left transition-all ${
                selected
                  ? 'border-primary bg-primary/5 shadow-sm ring-2 ring-primary/30'
                  : 'border-border/70 bg-card hover:border-primary/40 hover:bg-accent/40'
              }`}
            >
              <span
                className={`inline-flex h-9 w-9 items-center justify-center rounded-xl ${
                  selected ? 'bg-primary text-primary-foreground' : 'bg-muted text-muted-foreground'
                }`}
              >
                <Icon className="h-4 w-4" />
              </span>
              <span className="font-display text-sm font-semibold">{t(`planner.priority.${key}`)}</span>
              <span className="text-xs leading-relaxed text-muted-foreground">{t(`planner.priority.${key}Desc`)}</span>
            </button>
          );
        })}
      </div>

      {errors.length > 0 && (
        <ul className="space-y-1 text-sm text-destructive">
          {errors.map((key) => (
            <li key={key}>{t(key)}</li>
          ))}
        </ul>
      )}
    </div>
  );
}
#TimeLimitPicker
import { useState } from 'react';
import { useLanguage } from '@/lib/LanguageContext';

export default function TimeLimitPicker({ id, title, hint, presets, value, onChange, error, max }) {
  const { t } = useLanguage();
  const [customOpen, setCustomOpen] = useState(false);
  const [custom, setCustom] = useState('');

  const isPreset = presets.includes(value);

  const applyCustom = (raw) => {
    setCustom(raw);
    const parsed = Number(raw);
    if (raw !== '' && Number.isFinite(parsed) && parsed > 0 && parsed <= max) onChange(parsed);
    else onChange(null);
  };

  return (
    <div className="space-y-2 rounded-xl border border-border/70 bg-background p-3">
      <div>
        <h3 className="font-display text-sm font-semibold">{title}</h3>
        <p className="text-xs text-muted-foreground">{hint}</p>
      </div>

      <div className="flex flex-wrap gap-2">
        {presets.map((preset) => (
          <button
            key={preset}
            type="button"
            onClick={() => {
              setCustomOpen(false);
              setCustom('');
              onChange(preset);
            }}
            aria-pressed={value === preset}
            className={`rounded-full border px-3 py-1.5 text-xs font-medium transition-colors ${
              value === preset
                ? 'border-primary bg-primary text-primary-foreground'
                : 'border-border text-foreground hover:bg-accent'
            }`}
          >
            {t('planner.minutesShort', { count: preset })}
          </button>
        ))}
        <button
          type="button"
          onClick={() => {
            setCustomOpen(true);
            setCustom('');
            onChange(null);
          }}
          aria-pressed={customOpen}
          className={`rounded-full border px-3 py-1.5 text-xs font-medium transition-colors ${
            customOpen || (!isPreset && value !== null)
              ? 'border-primary bg-primary text-primary-foreground'
              : 'border-border text-foreground hover:bg-accent'
          }`}
        >
          {t('planner.custom')}
        </button>
      </div>

      {(customOpen || (!isPreset && value !== null)) && (
        <div className="flex items-center gap-2">
          <label htmlFor={`${id}-custom`} className="text-xs text-muted-foreground">
            {t('planner.customLabel')}
          </label>
          <input
            id={`${id}-custom`}
            type="number"
            min="1"
            max={max}
            inputMode="numeric"
            value={custom || (value ?? '')}
            onChange={(event) => applyCustom(event.target.value)}
            className="h-8 w-24 rounded-lg border border-input bg-transparent px-2 text-sm focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring"
          />
          <span className="text-xs text-muted-foreground">{t('planner.minutesWord')}</span>
        </div>
      )}

      {error && <p className="text-xs text-destructive">{error}</p>}
    </div>
  );
}
#ui
#accordion
import * as React from "react"
import * as AccordionPrimitive from "@radix-ui/react-accordion"
import { ChevronDown } from "lucide-react"

import { cn } from "@/lib/utils"

const Accordion = AccordionPrimitive.Root

const AccordionItem = React.forwardRef(({ className, ...props }, ref) => (
  <AccordionPrimitive.Item ref={ref} className={cn("border-b", className)} {...props} />
))
AccordionItem.displayName = "AccordionItem"

const AccordionTrigger = React.forwardRef(({ className, children, ...props }, ref) => (
  <AccordionPrimitive.Header className="flex">
    <AccordionPrimitive.Trigger
      ref={ref}
      className={cn(
        "flex flex-1 items-center justify-between py-4 text-sm font-medium transition-all hover:underline text-left [&[data-state=open]>svg]:rotate-180",
        className
      )}
      {...props}>
      {children}
      <ChevronDown
        className="h-4 w-4 shrink-0 text-muted-foreground transition-transform duration-200" />
    </AccordionPrimitive.Trigger>
  </AccordionPrimitive.Header>
))
AccordionTrigger.displayName = AccordionPrimitive.Trigger.displayName

const AccordionContent = React.forwardRef(({ className, children, ...props }, ref) => (
  <AccordionPrimitive.Content
    ref={ref}
    className="overflow-hidden text-sm data-[state=closed]:animate-accordion-up data-[state=open]:animate-accordion-down"
    {...props}>
    <div className={cn("pb-4 pt-0", className)}>{children}</div>
  </AccordionPrimitive.Content>
))
AccordionContent.displayName = AccordionPrimitive.Content.displayName

export { Accordion, AccordionItem, AccordionTrigger, AccordionContent }
#alert-dialog
import * as React from "react"
import * as AlertDialogPrimitive from "@radix-ui/react-alert-dialog"

import { cn } from "@/lib/utils"
import { buttonVariants } from "@/components/ui/button"

const AlertDialog = AlertDialogPrimitive.Root

const AlertDialogTrigger = AlertDialogPrimitive.Trigger

const AlertDialogPortal = AlertDialogPrimitive.Portal

const AlertDialogOverlay = React.forwardRef(({ className, ...props }, ref) => (
  <AlertDialogPrimitive.Overlay
    className={cn(
      "fixed inset-0 z-50 bg-black/80 data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0",
      className
    )}
    {...props}
    ref={ref} />
))
AlertDialogOverlay.displayName = AlertDialogPrimitive.Overlay.displayName

const AlertDialogContent = React.forwardRef(({ className, ...props }, ref) => (
  <AlertDialogPortal>
    <AlertDialogOverlay />
    <AlertDialogPrimitive.Content
      ref={ref}
      className={cn(
        "fixed left-[50%] top-[50%] z-50 grid w-full max-w-lg translate-x-[-50%] translate-y-[-50%] gap-4 border bg-background p-6 shadow-lg duration-200 data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 data-[state=closed]:slide-out-to-left-1/2 data-[state=closed]:slide-out-to-top-[48%] data-[state=open]:slide-in-from-left-1/2 data-[state=open]:slide-in-from-top-[48%] sm:rounded-lg",
        className
      )}
      {...props} />
  </AlertDialogPortal>
))
AlertDialogContent.displayName = AlertDialogPrimitive.Content.displayName

const AlertDialogHeader = ({
  className,
  ...props
}) => (
  <div
    className={cn("flex flex-col space-y-2 text-center sm:text-left", className)}
    {...props} />
)
AlertDialogHeader.displayName = "AlertDialogHeader"

const AlertDialogFooter = ({
  className,
  ...props
}) => (
  <div
    className={cn("flex flex-col-reverse sm:flex-row sm:justify-end sm:space-x-2", className)}
    {...props} />
)
AlertDialogFooter.displayName = "AlertDialogFooter"

const AlertDialogTitle = React.forwardRef(({ className, ...props }, ref) => (
  <AlertDialogPrimitive.Title ref={ref} className={cn("text-lg font-semibold", className)} {...props} />
))
AlertDialogTitle.displayName = AlertDialogPrimitive.Title.displayName

const AlertDialogDescription = React.forwardRef(({ className, ...props }, ref) => (
  <AlertDialogPrimitive.Description
    ref={ref}
    className={cn("text-sm text-muted-foreground", className)}
    {...props} />
))
AlertDialogDescription.displayName =
  AlertDialogPrimitive.Description.displayName

const AlertDialogAction = React.forwardRef(({ className, ...props }, ref) => (
  <AlertDialogPrimitive.Action ref={ref} className={cn(buttonVariants(), className)} {...props} />
))
AlertDialogAction.displayName = AlertDialogPrimitive.Action.displayName

const AlertDialogCancel = React.forwardRef(({ className, ...props }, ref) => (
  <AlertDialogPrimitive.Cancel
    ref={ref}
    className={cn(buttonVariants({ variant: "outline" }), "mt-2 sm:mt-0", className)}
    {...props} />
))
AlertDialogCancel.displayName = AlertDialogPrimitive.Cancel.displayName

export {
  AlertDialog,
  AlertDialogPortal,
  AlertDialogOverlay,
  AlertDialogTrigger,
  AlertDialogContent,
  AlertDialogHeader,
  AlertDialogFooter,
  AlertDialogTitle,
  AlertDialogDescription,
  AlertDialogAction,
  AlertDialogCancel,
}
#alert
import * as React from "react"
import { cva } from "class-variance-authority";

import { cn } from "@/lib/utils"

const alertVariants = cva(
  "relative w-full rounded-lg border px-4 py-3 text-sm [&>svg+div]:translate-y-[-3px] [&>svg]:absolute [&>svg]:left-4 [&>svg]:top-4 [&>svg]:text-foreground [&>svg~*]:pl-7",
  {
    variants: {
      variant: {
        default: "bg-background text-foreground",
        destructive:
          "border-destructive/50 text-destructive dark:border-destructive [&>svg]:text-destructive",
      },
    },
    defaultVariants: {
      variant: "default",
    },
  }
)

const Alert = React.forwardRef(({ className, variant, ...props }, ref) => (
  <div
    ref={ref}
    role="alert"
    className={cn(alertVariants({ variant }), className)}
    {...props} />
))
Alert.displayName = "Alert"

const AlertTitle = React.forwardRef(({ className, ...props }, ref) => (
  <h5
    ref={ref}
    className={cn("mb-1 font-medium leading-none tracking-tight", className)}
    {...props} />
))
AlertTitle.displayName = "AlertTitle"

const AlertDescription = React.forwardRef(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn("text-sm [&_p]:leading-relaxed", className)}
    {...props} />
))
AlertDescription.displayName = "AlertDescription"

export { Alert, AlertTitle, AlertDescription }

#aspect-ratio
import * as AspectRatioPrimitive from "@radix-ui/react-aspect-ratio"

const AspectRatio = AspectRatioPrimitive.Root

export { AspectRatio }
#avatar
"use client"

import * as React from "react"
import * as AvatarPrimitive from "@radix-ui/react-avatar"

import { cn } from "@/lib/utils"

const Avatar = React.forwardRef(({ className, ...props }, ref) => (
  <AvatarPrimitive.Root
    ref={ref}
    className={cn("relative flex h-10 w-10 shrink-0 overflow-hidden rounded-full", className)}
    {...props} />
))
Avatar.displayName = AvatarPrimitive.Root.displayName

const AvatarImage = React.forwardRef(({ className, ...props }, ref) => (
  <AvatarPrimitive.Image
    ref={ref}
    className={cn("aspect-square h-full w-full", className)}
    {...props} />
))
AvatarImage.displayName = AvatarPrimitive.Image.displayName

const AvatarFallback = React.forwardRef(({ className, ...props }, ref) => (
  <AvatarPrimitive.Fallback
    ref={ref}
    className={cn(
      "flex h-full w-full items-center justify-center rounded-full bg-muted",
      className
    )}
    {...props} />
))
AvatarFallback.displayName = AvatarPrimitive.Fallback.displayName

export { Avatar, AvatarImage, AvatarFallback }
#badge
import * as React from "react"
import { cva } from "class-variance-authority";

import { cn } from "@/lib/utils"

const badgeVariants = cva(
  "inline-flex items-center rounded-md border px-2.5 py-0.5 text-xs font-semibold transition-colors focus:outline-none focus:ring-2 focus:ring-ring focus:ring-offset-2",
  {
    variants: {
      variant: {
        default:
          "border-transparent bg-primary text-primary-foreground shadow hover:bg-primary/80",
        secondary:
          "border-transparent bg-secondary text-secondary-foreground hover:bg-secondary/80",
        destructive:
          "border-transparent bg-destructive text-destructive-foreground shadow hover:bg-destructive/80",
        outline: "text-foreground",
      },
    },
    defaultVariants: {
      variant: "default",
    },
  }
)

function Badge({
  className,
  variant,
  ...props
}) {
  return (<div className={cn(badgeVariants({ variant }), className)} {...props} />);
}

export { Badge, badgeVariants }
#breadcrumb
import * as React from "react"
import { Slot } from "@radix-ui/react-slot"
import { ChevronRight, MoreHorizontal } from "lucide-react"

import { cn } from "@/lib/utils"

const Breadcrumb = React.forwardRef(
  ({ ...props }, ref) => <nav ref={ref} aria-label="breadcrumb" {...props} />
)
Breadcrumb.displayName = "Breadcrumb"

const BreadcrumbList = React.forwardRef(({ className, ...props }, ref) => (
  <ol
    ref={ref}
    className={cn(
      "flex flex-wrap items-center gap-1.5 break-words text-sm text-muted-foreground sm:gap-2.5",
      className
    )}
    {...props} />
))
BreadcrumbList.displayName = "BreadcrumbList"

const BreadcrumbItem = React.forwardRef(({ className, ...props }, ref) => (
  <li
    ref={ref}
    className={cn("inline-flex items-center gap-1.5", className)}
    {...props} />
))
BreadcrumbItem.displayName = "BreadcrumbItem"

const BreadcrumbLink = React.forwardRef(({ asChild, className, ...props }, ref) => {
  const Comp = asChild ? Slot : "a"

  return (
    (<Comp
      ref={ref}
      className={cn("transition-colors hover:text-foreground", className)}
      {...props} />)
  );
})
BreadcrumbLink.displayName = "BreadcrumbLink"

const BreadcrumbPage = React.forwardRef(({ className, ...props }, ref) => (
  <span
    ref={ref}
    role="link"
    aria-disabled="true"
    aria-current="page"
    className={cn("font-normal text-foreground", className)}
    {...props} />
))
BreadcrumbPage.displayName = "BreadcrumbPage"

const BreadcrumbSeparator = ({
  children,
  className,
  ...props
}) => (
  <li
    role="presentation"
    aria-hidden="true"
    className={cn("[&>svg]:w-3.5 [&>svg]:h-3.5", className)}
    {...props}>
    {children ?? <ChevronRight />}
  </li>
)
BreadcrumbSeparator.displayName = "BreadcrumbSeparator"

const BreadcrumbEllipsis = ({
  className,
  ...props
}) => (
  <span
    role="presentation"
    aria-hidden="true"
    className={cn("flex h-9 w-9 items-center justify-center", className)}
    {...props}>
    <MoreHorizontal className="h-4 w-4" />
    <span className="sr-only">More</span>
  </span>
)
BreadcrumbEllipsis.displayName = "BreadcrumbElipssis"

export {
  Breadcrumb,
  BreadcrumbList,
  BreadcrumbItem,
  BreadcrumbLink,
  BreadcrumbPage,
  BreadcrumbSeparator,
  BreadcrumbEllipsis,
}

#button
import * as React from "react"
import { Slot } from "@radix-ui/react-slot"
import { cva } from "class-variance-authority";

import { cn } from "@/lib/utils"

const buttonVariants = cva(
  "inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50 [&_svg]:pointer-events-none [&_svg]:size-4 [&_svg]:shrink-0",
  {
    variants: {
      variant: {
        default:
          "bg-primary text-primary-foreground shadow hover:bg-primary/90",
        destructive:
          "bg-destructive text-destructive-foreground shadow-sm hover:bg-destructive/90",
        outline:
          "border border-input bg-transparent shadow-sm hover:bg-accent hover:text-accent-foreground",
        secondary:
          "bg-secondary text-secondary-foreground shadow-sm hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline",
      },
      size: {
        default: "h-9 px-4 py-2",
        sm: "h-8 rounded-md px-3 text-xs",
        lg: "h-10 rounded-md px-8",
        icon: "h-9 w-9",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
)

const Button = React.forwardRef(({ className, variant, size, asChild = false, ...props }, ref) => {
  const Comp = asChild ? Slot : "button"
  return (
    (<Comp
      className={cn(buttonVariants({ variant, size, className }))}
      ref={ref}
      {...props} />)
  );
})
Button.displayName = "Button"

export { Button, buttonVariants }

#calendar
import * as React from "react"
import { ChevronLeft, ChevronRight } from "lucide-react"
import { DayPicker } from "react-day-picker"

import { cn } from "@/lib/utils"
import { buttonVariants } from "@/components/ui/button"

function Calendar({
  className,
  classNames,
  showOutsideDays = true,
  ...props
}) {
  return (
    (<DayPicker
      showOutsideDays={showOutsideDays}
      className={cn("p-3", className)}
      classNames={{
        months: "flex flex-col sm:flex-row space-y-4 sm:space-x-4 sm:space-y-0",
        month: "space-y-4",
        caption: "flex justify-center pt-1 relative items-center",
        caption_label: "text-sm font-medium",
        nav: "space-x-1 flex items-center",
        nav_button: cn(
          buttonVariants({ variant: "outline" }),
          "h-7 w-7 bg-transparent p-0 opacity-50 hover:opacity-100"
        ),
        nav_button_previous: "absolute left-1",
        nav_button_next: "absolute right-1",
        table: "w-full border-collapse space-y-1",
        head_row: "flex",
        head_cell:
          "text-muted-foreground rounded-md w-8 font-normal text-[0.8rem]",
        row: "flex w-full mt-2",
        cell: cn(
          "relative p-0 text-center text-sm focus-within:relative focus-within:z-20 [&:has([aria-selected])]:bg-accent [&:has([aria-selected].day-outside)]:bg-accent/50 [&:has([aria-selected].day-range-end)]:rounded-r-md",
          props.mode === "range"
            ? "[&:has(>.day-range-end)]:rounded-r-md [&:has(>.day-range-start)]:rounded-l-md first:[&:has([aria-selected])]:rounded-l-md last:[&:has([aria-selected])]:rounded-r-md"
            : "[&:has([aria-selected])]:rounded-md"
        ),
        day: cn(
          buttonVariants({ variant: "ghost" }),
          "h-8 w-8 p-0 font-normal aria-selected:opacity-100"
        ),
        day_range_start: "day-range-start",
        day_range_end: "day-range-end",
        day_selected:
          "bg-primary text-primary-foreground hover:bg-primary hover:text-primary-foreground focus:bg-primary focus:text-primary-foreground",
        day_today: "bg-accent text-accent-foreground",
        day_outside:
          "day-outside text-muted-foreground aria-selected:bg-accent/50 aria-selected:text-muted-foreground",
        day_disabled: "text-muted-foreground opacity-50",
        day_range_middle:
          "aria-selected:bg-accent aria-selected:text-accent-foreground",
        day_hidden: "invisible",
        ...classNames,
      }}
      components={{
        IconLeft: ({ className, ...props }) => (
          <ChevronLeft className={cn("h-4 w-4", className)} {...props} />
        ),
        IconRight: ({ className, ...props }) => (
          <ChevronRight className={cn("h-4 w-4", className)} {...props} />
        ),
      }}
      {...props} />)
  );
}
Calendar.displayName = "Calendar"

export { Calendar }
#card
import * as React from "react"

import { cn } from "@/lib/utils"

const Card = React.forwardRef(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn("rounded-xl border bg-card text-card-foreground shadow", className)}
    {...props} />
))
Card.displayName = "Card"

const CardHeader = React.forwardRef(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn("flex flex-col space-y-1.5 p-6", className)}
    {...props} />
))
CardHeader.displayName = "CardHeader"

const CardTitle = React.forwardRef(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn("font-semibold leading-none tracking-tight", className)}
    {...props} />
))
CardTitle.displayName = "CardTitle"

const CardDescription = React.forwardRef(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn("text-sm text-muted-foreground", className)}
    {...props} />
))
CardDescription.displayName = "CardDescription"

const CardContent = React.forwardRef(({ className, ...props }, ref) => (
  <div ref={ref} className={cn("p-6 pt-0", className)} {...props} />
))
CardContent.displayName = "CardContent"

const CardFooter = React.forwardRef(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn("flex items-center p-6 pt-0", className)}
    {...props} />
))
CardFooter.displayName = "CardFooter"

export { Card, CardHeader, CardFooter, CardTitle, CardDescription, CardContent }

#carousel
mport * as React from "react"
import useEmblaCarousel from "embla-carousel-react";
import { ArrowLeft, ArrowRight } from "lucide-react"

import { cn } from "@/lib/utils"
import { Button } from "@/components/ui/button"

const CarouselContext = React.createContext(null)

function useCarousel() {
  const context = React.useContext(CarouselContext)

  if (!context) {
    throw new Error("useCarousel must be used within a <Carousel />")
  }

  return context
}

const Carousel = React.forwardRef((
  {
    orientation = "horizontal",
    opts,
    setApi,
    plugins,
    className,
    children,
    ...props
  },
  ref
) => {
  const [carouselRef, api] = useEmblaCarousel({
    ...opts,
    axis: orientation === "horizontal" ? "x" : "y",
  }, plugins)
  const [canScrollPrev, setCanScrollPrev] = React.useState(false)
  const [canScrollNext, setCanScrollNext] = React.useState(false)

  const onSelect = React.useCallback((api) => {
    if (!api) {
      return
    }

    setCanScrollPrev(api.canScrollPrev())
    setCanScrollNext(api.canScrollNext())
  }, [])

  const scrollPrev = React.useCallback(() => {
    api?.scrollPrev()
  }, [api])

  const scrollNext = React.useCallback(() => {
    api?.scrollNext()
  }, [api])

  const handleKeyDown = React.useCallback((event) => {
    if (event.key === "ArrowLeft") {
      event.preventDefault()
      scrollPrev()
    } else if (event.key === "ArrowRight") {
      event.preventDefault()
      scrollNext()
    }
  }, [scrollPrev, scrollNext])

  React.useEffect(() => {
    if (!api || !setApi) {
      return
    }

    setApi(api)
  }, [api, setApi])

  React.useEffect(() => {
    if (!api) {
      return
    }

    onSelect(api)
    api.on("reInit", onSelect)
    api.on("select", onSelect)

    return () => {
      api?.off("select", onSelect)
    };
  }, [api, onSelect])

  return (
    (<CarouselContext.Provider
      value={{
        carouselRef,
        api: api,
        opts,
        orientation:
          orientation || (opts?.axis === "y" ? "vertical" : "horizontal"),
        scrollPrev,
        scrollNext,
        canScrollPrev,
        canScrollNext,
      }}>
      <div
        ref={ref}
        onKeyDownCapture={handleKeyDown}
        className={cn("relative", className)}
        role="region"
        aria-roledescription="carousel"
        {...props}>
        {children}
      </div>
    </CarouselContext.Provider>)
  );
})
Carousel.displayName = "Carousel"

const CarouselContent = React.forwardRef(({ className, ...props }, ref) => {
  const { carouselRef, orientation } = useCarousel()

  return (
    (<div ref={carouselRef} className="overflow-hidden">
      <div
        ref={ref}
        className={cn(
          "flex",
          orientation === "horizontal" ? "-ml-4" : "-mt-4 flex-col",
          className
        )}
        {...props} />
    </div>)
  );
})
CarouselContent.displayName = "CarouselContent"

const CarouselItem = React.forwardRef(({ className, ...props }, ref) => {
  const { orientation } = useCarousel()

  return (
    (<div
      ref={ref}
      role="group"
      aria-roledescription="slide"
      className={cn(
        "min-w-0 shrink-0 grow-0 basis-full",
        orientation === "horizontal" ? "pl-4" : "pt-4",
        className
      )}
      {...props} />)
  );
})
CarouselItem.displayName = "CarouselItem"

const CarouselPrevious = React.forwardRef(({ className, variant = "outline", size = "icon", ...props }, ref) => {
  const { orientation, scrollPrev, canScrollPrev } = useCarousel()

  return (
    (<Button
      ref={ref}
      variant={variant}
      size={size}
      className={cn("absolute  h-8 w-8 rounded-full", orientation === "horizontal"
        ? "-left-12 top-1/2 -translate-y-1/2"
        : "-top-12 left-1/2 -translate-x-1/2 rotate-90", className)}
      disabled={!canScrollPrev}
      onClick={scrollPrev}
      {...props}>
      <ArrowLeft className="h-4 w-4" />
      <span className="sr-only">Previous slide</span>
    </Button>)
  );
})
CarouselPrevious.displayName = "CarouselPrevious"

const CarouselNext = React.forwardRef(({ className, variant = "outline", size = "icon", ...props }, ref) => {
  const { orientation, scrollNext, canScrollNext } = useCarousel()

  return (
    (<Button
      ref={ref}
      variant={variant}
      size={size}
      className={cn("absolute h-8 w-8 rounded-full", orientation === "horizontal"
        ? "-right-12 top-1/2 -translate-y-1/2"
        : "-bottom-12 left-1/2 -translate-x-1/2 rotate-90", className)}
      disabled={!canScrollNext}
      onClick={scrollNext}
      {...props}>
      <ArrowRight className="h-4 w-4" />
      <span className="sr-only">Next slide</span>
    </Button>)
  );
})
CarouselNext.displayName = "CarouselNext"

export { Carousel, CarouselContent, CarouselItem, CarouselPrevious, CarouselNext };
#chart
"use client";
import * as React from "react"
import * as RechartsPrimitive from "recharts"

import { cn } from "@/lib/utils"

// Format: { THEME_NAME: CSS_SELECTOR }
const THEMES = {
  light: "",
  dark: ".dark"
}

const ChartContext = React.createContext(null)

function useChart() {
  const context = React.useContext(ChartContext)

  if (!context) {
    throw new Error("useChart must be used within a <ChartContainer />")
  }

  return context
}

const ChartContainer = React.forwardRef(({ id, className, children, config, ...props }, ref) => {
  const uniqueId = React.useId()
  const chartId = `chart-${id || uniqueId.replace(/:/g, "")}`

  return (
    (<ChartContext.Provider value={{ config }}>
      <div
        data-chart={chartId}
        ref={ref}
        className={cn(
          "flex aspect-video justify-center text-xs [&_.recharts-cartesian-axis-tick_text]:fill-muted-foreground [&_.recharts-cartesian-grid_line[stroke='#ccc']]:stroke-border/50 [&_.recharts-curve.recharts-tooltip-cursor]:stroke-border [&_.recharts-dot[stroke='#fff']]:stroke-transparent [&_.recharts-layer]:outline-none [&_.recharts-polar-grid_[stroke='#ccc']]:stroke-border [&_.recharts-radial-bar-background-sector]:fill-muted [&_.recharts-rectangle.recharts-tooltip-cursor]:fill-muted [&_.recharts-reference-line_[stroke='#ccc']]:stroke-border [&_.recharts-sector[stroke='#fff']]:stroke-transparent [&_.recharts-sector]:outline-none [&_.recharts-surface]:outline-none",
          className
        )}
        {...props}>
        <ChartStyle id={chartId} config={config} />
        <RechartsPrimitive.ResponsiveContainer>
          {children}
        </RechartsPrimitive.ResponsiveContainer>
      </div>
    </ChartContext.Provider>)
  );
})
ChartContainer.displayName = "Chart"

const ChartStyle = ({
  id,
  config
}) => {
  const colorConfig = Object.entries(config).filter(([, config]) => config.theme || config.color)

  if (!colorConfig.length) {
    return null
  }

  return (
    (<style
      dangerouslySetInnerHTML={{
        __html: Object.entries(THEMES)
          .map(([theme, prefix]) => `
${prefix} [data-chart=${id}] {
${colorConfig
.map(([key, itemConfig]) => {
const color =
  itemConfig.theme?.[theme] ||
  itemConfig.color
return color ? `  --color-${key}: ${color};` : null
})
.join("\n")}
}
`)
          .join("\n"),
      }} />)
  );
}

const ChartTooltip = RechartsPrimitive.Tooltip

const ChartTooltipContent = React.forwardRef((
  {
    active,
    payload,
    className,
    indicator = "dot",
    hideLabel = false,
    hideIndicator = false,
    label,
    labelFormatter,
    labelClassName,
    formatter,
    color,
    nameKey,
    labelKey,
  },
  ref
) => {
  const { config } = useChart()

  const tooltipLabel = React.useMemo(() => {
    if (hideLabel || !payload?.length) {
      return null
    }

    const [item] = payload
    const key = `${labelKey || item.dataKey || item.name || "value"}`
    const itemConfig = getPayloadConfigFromPayload(config, item, key)
    const value =
      !labelKey && typeof label === "string"
        ? config[label]?.label || label
        : itemConfig?.label

    if (labelFormatter) {
      return (
        (<div className={cn("font-medium", labelClassName)}>
          {labelFormatter(value, payload)}
        </div>)
      );
    }

    if (!value) {
      return null
    }

    return <div className={cn("font-medium", labelClassName)}>{value}</div>;
  }, [
    label,
    labelFormatter,
    payload,
    hideLabel,
    labelClassName,
    config,
    labelKey,
  ])

  if (!active || !payload?.length) {
    return null
  }

  const nestLabel = payload.length === 1 && indicator !== "dot"

  return (
    (<div
      ref={ref}
      className={cn(
        "grid min-w-[8rem] items-start gap-1.5 rounded-lg border border-border/50 bg-background px-2.5 py-1.5 text-xs shadow-xl",
        className
      )}>
      {!nestLabel ? tooltipLabel : null}
      <div className="grid gap-1.5">
        {payload.map((item, index) => {
          const key = `${nameKey || item.name || item.dataKey || "value"}`
          const itemConfig = getPayloadConfigFromPayload(config, item, key)
          const indicatorColor = color || item.payload.fill || item.color

          return (
            (<div
              key={item.dataKey}
              className={cn(
                "flex w-full flex-wrap items-stretch gap-2 [&>svg]:h-2.5 [&>svg]:w-2.5 [&>svg]:text-muted-foreground",
                indicator === "dot" && "items-center"
              )}>
              {formatter && item?.value !== undefined && item.name ? (
                formatter(item.value, item.name, item, index, item.payload)
              ) : (
                <>
                  {itemConfig?.icon ? (
                    <itemConfig.icon />
                  ) : (
                    !hideIndicator && (
                      <div
                        className={cn("shrink-0 rounded-[2px] border-[--color-border] bg-[--color-bg]", {
                          "h-2.5 w-2.5": indicator === "dot",
                          "w-1": indicator === "line",
                          "w-0 border-[1.5px] border-dashed bg-transparent":
                            indicator === "dashed",
                          "my-0.5": nestLabel && indicator === "dashed",
                        })}
                        style={
                          {
                            "--color-bg": indicatorColor,
                            "--color-border": indicatorColor
                          }
                        } />
                    )
                  )}
                  <div
                    className={cn(
                      "flex flex-1 justify-between leading-none",
                      nestLabel ? "items-end" : "items-center"
                    )}>
                    <div className="grid gap-1.5">
                      {nestLabel ? tooltipLabel : null}
                      <span className="text-muted-foreground">
                        {itemConfig?.label || item.name}
                      </span>
                    </div>
                    {item.value && (
                      <span className="font-mono font-medium tabular-nums text-foreground">
                        {item.value.toLocaleString()}
                      </span>
                    )}
                  </div>
                </>
              )}
            </div>)
          );
        })}
      </div>
    </div>)
  );
})
ChartTooltipContent.displayName = "ChartTooltip"

const ChartLegend = RechartsPrimitive.Legend

const ChartLegendContent = React.forwardRef((
  { className, hideIcon = false, payload, verticalAlign = "bottom", nameKey },
  ref
) => {
  const { config } = useChart()

  if (!payload?.length) {
    return null
  }

  return (
    (<div
      ref={ref}
      className={cn(
        "flex items-center justify-center gap-4",
        verticalAlign === "top" ? "pb-3" : "pt-3",
        className
      )}>
      {payload.map((item) => {
        const key = `${nameKey || item.dataKey || "value"}`
        const itemConfig = getPayloadConfigFromPayload(config, item, key)

        return (
          (<div
            key={item.value}
            className={cn(
              "flex items-center gap-1.5 [&>svg]:h-3 [&>svg]:w-3 [&>svg]:text-muted-foreground"
            )}>
            {itemConfig?.icon && !hideIcon ? (
              <itemConfig.icon />
            ) : (
              <div
                className="h-2 w-2 shrink-0 rounded-[2px]"
                style={{
                  backgroundColor: item.color,
                }} />
            )}
            {itemConfig?.label}
          </div>)
        );
      })}
    </div>)
  );
})
ChartLegendContent.displayName = "ChartLegend"

// Helper to extract item config from a payload.
function getPayloadConfigFromPayload(
  config,
  payload,
  key
) {
  if (typeof payload !== "object" || payload === null) {
    return undefined
  }

  const payloadPayload =
    "payload" in payload &&
    typeof payload.payload === "object" &&
    payload.payload !== null
      ? payload.payload
      : undefined

  let configLabelKey = key

  if (
    key in payload &&
    typeof payload[key] === "string"
  ) {
    configLabelKey = payload[key]
  } else if (
    payloadPayload &&
    key in payloadPayload &&
    typeof payloadPayload[key] === "string"
  ) {
    configLabelKey = payloadPayload[key]
  }

  return configLabelKey in config
    ? config[configLabelKey]
    : config[key];
}

export {
  ChartContainer,
  ChartTooltip,
  ChartTooltipContent,
  ChartLegend,
  ChartLegendContent,
  ChartStyle,
}
#checkbox
import * as React from "react"
import * as CheckboxPrimitive from "@radix-ui/react-checkbox"
import { Check } from "lucide-react"

import { cn } from "@/lib/utils"

const Checkbox = React.forwardRef(({ className, ...props }, ref) => (
  <CheckboxPrimitive.Root
    ref={ref}
    className={cn(
      "peer h-4 w-4 shrink-0 rounded-sm border border-primary shadow focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:cursor-not-allowed disabled:opacity-50 data-[state=checked]:bg-primary data-[state=checked]:text-primary-foreground",
      className
    )}
    {...props}>
    <CheckboxPrimitive.Indicator className={cn("flex items-center justify-center text-current")}>
      <Check className="h-4 w-4" />
    </CheckboxPrimitive.Indicator>
  </CheckboxPrimitive.Root>
))
Checkbox.displayName = CheckboxPrimitive.Root.displayName

export { Checkbox }
#collapsible
"use client"

import * as CollapsiblePrimitive from "@radix-ui/react-collapsible"

const Collapsible = CollapsiblePrimitive.Root

const CollapsibleTrigger = CollapsiblePrimitive.CollapsibleTrigger

const CollapsibleContent = CollapsiblePrimitive.CollapsibleContent

export { Collapsible, CollapsibleTrigger, CollapsibleContent }

#command
import * as React from "react"
import { Command as CommandPrimitive } from "cmdk"
import { Search } from "lucide-react"

import { cn } from "@/lib/utils"
import { Dialog, DialogContent } from "@/components/ui/dialog"

const Command = React.forwardRef(({ className, ...props }, ref) => (
  <CommandPrimitive
    ref={ref}
    className={cn(
      "flex h-full w-full flex-col overflow-hidden rounded-md bg-popover text-popover-foreground",
      className
    )}
    {...props} />
))
Command.displayName = CommandPrimitive.displayName

const CommandDialog = ({
  children,
  ...props
}) => {
  return (
    (<Dialog {...props}>
      <DialogContent className="overflow-hidden p-0">
        <Command
          className="[&_[cmdk-group-heading]]:px-2 [&_[cmdk-group-heading]]:font-medium [&_[cmdk-group-heading]]:text-muted-foreground [&_[cmdk-group]:not([hidden])_~[cmdk-group]]:pt-0 [&_[cmdk-group]]:px-2 [&_[cmdk-input-wrapper]_svg]:h-5 [&_[cmdk-input-wrapper]_svg]:w-5 [&_[cmdk-input]]:h-12 [&_[cmdk-item]]:px-2 [&_[cmdk-item]]:py-3 [&_[cmdk-item]_svg]:h-5 [&_[cmdk-item]_svg]:w-5">
          {children}
        </Command>
      </DialogContent>
    </Dialog>)
  );
}

const CommandInput = React.forwardRef(({ className, ...props }, ref) => (
  <div className="flex items-center border-b px-3" cmdk-input-wrapper="">
    <Search className="mr-2 h-4 w-4 shrink-0 opacity-50" />
    <CommandPrimitive.Input
      ref={ref}
      className={cn(
        "flex h-10 w-full rounded-md bg-transparent py-3 text-sm outline-none placeholder:text-muted-foreground disabled:cursor-not-allowed disabled:opacity-50",
        className
      )}
      {...props} />
  </div>
))

CommandInput.displayName = CommandPrimitive.Input.displayName

const CommandList = React.forwardRef(({ className, ...props }, ref) => (
  <CommandPrimitive.List
    ref={ref}
    className={cn("max-h-[300px] overflow-y-auto overflow-x-hidden", className)}
    {...props} />
))

CommandList.displayName = CommandPrimitive.List.displayName

const CommandEmpty = React.forwardRef((props, ref) => (
  <CommandPrimitive.Empty ref={ref} className="py-6 text-center text-sm" {...props} />
))

CommandEmpty.displayName = CommandPrimitive.Empty.displayName

const CommandGroup = React.forwardRef(({ className, ...props }, ref) => (
  <CommandPrimitive.Group
    ref={ref}
    className={cn(
      "overflow-hidden p-1 text-foreground [&_[cmdk-group-heading]]:px-2 [&_[cmdk-group-heading]]:py-1.5 [&_[cmdk-group-heading]]:text-xs [&_[cmdk-group-heading]]:font-medium [&_[cmdk-group-heading]]:text-muted-foreground",
      className
    )}
    {...props} />
))

CommandGroup.displayName = CommandPrimitive.Group.displayName

const CommandSeparator = React.forwardRef(({ className, ...props }, ref) => (
  <CommandPrimitive.Separator ref={ref} className={cn("-mx-1 h-px bg-border", className)} {...props} />
))
CommandSeparator.displayName = CommandPrimitive.Separator.displayName

const CommandItem = React.forwardRef(({ className, ...props }, ref) => (
  <CommandPrimitive.Item
    ref={ref}
    className={cn(
      "relative flex cursor-default gap-2 select-none items-center rounded-sm px-2 py-1.5 text-sm outline-none data-[disabled=true]:pointer-events-none data-[selected=true]:bg-accent data-[selected=true]:text-accent-foreground data-[disabled=true]:opacity-50 [&_svg]:pointer-events-none [&_svg]:size-4 [&_svg]:shrink-0",
      className
    )}
    {...props} />
))

CommandItem.displayName = CommandPrimitive.Item.displayName

const CommandShortcut = ({
  className,
  ...props
}) => {
  return (
    (<span
      className={cn("ml-auto text-xs tracking-widest text-muted-foreground", className)}
      {...props} />)
  );
}
CommandShortcut.displayName = "CommandShortcut"

export {
  Command,
  CommandDialog,
  CommandInput,
  CommandList,
  CommandEmpty,
  CommandGroup,
  CommandItem,
  CommandShortcut,
  CommandSeparator,
}
#context-menu
import * as React from "react"
import * as ContextMenuPrimitive from "@radix-ui/react-context-menu"
import { Check, ChevronRight, Circle } from "lucide-react"

import { cn } from "@/lib/utils"

const ContextMenu = ContextMenuPrimitive.Root

const ContextMenuTrigger = ContextMenuPrimitive.Trigger

const ContextMenuGroup = ContextMenuPrimitive.Group

const ContextMenuPortal = ContextMenuPrimitive.Portal

const ContextMenuSub = ContextMenuPrimitive.Sub

const ContextMenuRadioGroup = ContextMenuPrimitive.RadioGroup

const ContextMenuSubTrigger = React.forwardRef(({ className, inset, children, ...props }, ref) => (
  <ContextMenuPrimitive.SubTrigger
    ref={ref}
    className={cn(
      "flex cursor-default select-none items-center rounded-sm px-2 py-1.5 text-sm outline-none focus:bg-accent focus:text-accent-foreground data-[state=open]:bg-accent data-[state=open]:text-accent-foreground",
      inset && "pl-8",
      className
    )}
    {...props}>
    {children}
    <ChevronRight className="ml-auto h-4 w-4" />
  </ContextMenuPrimitive.SubTrigger>
))
ContextMenuSubTrigger.displayName = ContextMenuPrimitive.SubTrigger.displayName

const ContextMenuSubContent = React.forwardRef(({ className, ...props }, ref) => (
  <ContextMenuPrimitive.SubContent
    ref={ref}
    className={cn(
      "z-50 min-w-[8rem] overflow-hidden rounded-md border bg-popover p-1 text-popover-foreground shadow-lg data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 data-[side=bottom]:slide-in-from-top-2 data-[side=left]:slide-in-from-right-2 data-[side=right]:slide-in-from-left-2 data-[side=top]:slide-in-from-bottom-2",
      className
    )}
    {...props} />
))
ContextMenuSubContent.displayName = ContextMenuPrimitive.SubContent.displayName

const ContextMenuContent = React.forwardRef(({ className, ...props }, ref) => (
  <ContextMenuPrimitive.Portal>
    <ContextMenuPrimitive.Content
      ref={ref}
      className={cn(
        "z-50 min-w-[8rem] overflow-hidden rounded-md border bg-popover p-1 text-popover-foreground shadow-md data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 data-[side=bottom]:slide-in-from-top-2 data-[side=left]:slide-in-from-right-2 data-[side=right]:slide-in-from-left-2 data-[side=top]:slide-in-from-bottom-2",
        className
      )}
      {...props} />
  </ContextMenuPrimitive.Portal>
))
ContextMenuContent.displayName = ContextMenuPrimitive.Content.displayName

const ContextMenuItem = React.forwardRef(({ className, inset, ...props }, ref) => (
  <ContextMenuPrimitive.Item
    ref={ref}
    className={cn(
      "relative flex cursor-default select-none items-center rounded-sm px-2 py-1.5 text-sm outline-none focus:bg-accent focus:text-accent-foreground data-[disabled]:pointer-events-none data-[disabled]:opacity-50",
      inset && "pl-8",
      className
    )}
    {...props} />
))
ContextMenuItem.displayName = ContextMenuPrimitive.Item.displayName

const ContextMenuCheckboxItem = React.forwardRef(({ className, children, checked, ...props }, ref) => (
  <ContextMenuPrimitive.CheckboxItem
    ref={ref}
    className={cn(
      "relative flex cursor-default select-none items-center rounded-sm py-1.5 pl-8 pr-2 text-sm outline-none focus:bg-accent focus:text-accent-foreground data-[disabled]:pointer-events-none data-[disabled]:opacity-50",
      className
    )}
    checked={checked}
    {...props}>
    <span className="absolute left-2 flex h-3.5 w-3.5 items-center justify-center">
      <ContextMenuPrimitive.ItemIndicator>
        <Check className="h-4 w-4" />
      </ContextMenuPrimitive.ItemIndicator>
    </span>
    {children}
  </ContextMenuPrimitive.CheckboxItem>
))
ContextMenuCheckboxItem.displayName =
  ContextMenuPrimitive.CheckboxItem.displayName

const ContextMenuRadioItem = React.forwardRef(({ className, children, ...props }, ref) => (
  <ContextMenuPrimitive.RadioItem
    ref={ref}
    className={cn(
      "relative flex cursor-default select-none items-center rounded-sm py-1.5 pl-8 pr-2 text-sm outline-none focus:bg-accent focus:text-accent-foreground data-[disabled]:pointer-events-none data-[disabled]:opacity-50",
      className
    )}
    {...props}>
    <span className="absolute left-2 flex h-3.5 w-3.5 items-center justify-center">
      <ContextMenuPrimitive.ItemIndicator>
        <Circle className="h-4 w-4 fill-current" />
      </ContextMenuPrimitive.ItemIndicator>
    </span>
    {children}
  </ContextMenuPrimitive.RadioItem>
))
ContextMenuRadioItem.displayName = ContextMenuPrimitive.RadioItem.displayName

const ContextMenuLabel = React.forwardRef(({ className, inset, ...props }, ref) => (
  <ContextMenuPrimitive.Label
    ref={ref}
    className={cn(
      "px-2 py-1.5 text-sm font-semibold text-foreground",
      inset && "pl-8",
      className
    )}
    {...props} />
))
ContextMenuLabel.displayName = ContextMenuPrimitive.Label.displayName

const ContextMenuSeparator = React.forwardRef(({ className, ...props }, ref) => (
  <ContextMenuPrimitive.Separator
    ref={ref}
    className={cn("-mx-1 my-1 h-px bg-border", className)}
    {...props} />
))
ContextMenuSeparator.displayName = ContextMenuPrimitive.Separator.displayName

const ContextMenuShortcut = ({
  className,
  ...props
}) => {
  return (
    (<span
      className={cn("ml-auto text-xs tracking-widest text-muted-foreground", className)}
      {...props} />)
  );
}
ContextMenuShortcut.displayName = "ContextMenuShortcut"

export {
  ContextMenu,
  ContextMenuTrigger,
  ContextMenuContent,
  ContextMenuItem,
  ContextMenuCheckboxItem,
  ContextMenuRadioItem,
  ContextMenuLabel,
  ContextMenuSeparator,
  ContextMenuShortcut,
  ContextMenuGroup,
  ContextMenuPortal,
  ContextMenuSub,
  ContextMenuSubContent,
  ContextMenuSubTrigger,
  ContextMenuRadioGroup,
}
#dialog
"use client"

import * as React from "react"
import * as DialogPrimitive from "@radix-ui/react-dialog"
import { X } from "lucide-react"

import { cn } from "@/lib/utils"

const Dialog = DialogPrimitive.Root

const DialogTrigger = DialogPrimitive.Trigger

const DialogPortal = DialogPrimitive.Portal

const DialogClose = DialogPrimitive.Close

const DialogOverlay = React.forwardRef(({ className, ...props }, ref) => (
  <DialogPrimitive.Overlay
    ref={ref}
    className={cn(
      "fixed inset-0 z-50 bg-black/80  data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0",
      className
    )}
    {...props} />
))
DialogOverlay.displayName = DialogPrimitive.Overlay.displayName

const DialogContent = React.forwardRef(({ className, children, ...props }, ref) => (
  <DialogPortal>
    <DialogOverlay />
    <DialogPrimitive.Content
      ref={ref}
      className={cn(
        "fixed left-[50%] top-[50%] z-50 grid w-full max-w-lg translate-x-[-50%] translate-y-[-50%] gap-4 border bg-background p-6 shadow-lg duration-200 data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 data-[state=closed]:slide-out-to-left-1/2 data-[state=closed]:slide-out-to-top-[48%] data-[state=open]:slide-in-from-left-1/2 data-[state=open]:slide-in-from-top-[48%] sm:rounded-lg",
        className
      )}
      {...props}>
      {children}
      <DialogPrimitive.Close
        className="absolute right-4 top-4 rounded-sm opacity-70 ring-offset-background transition-opacity hover:opacity-100 focus:outline-none focus:ring-2 focus:ring-ring focus:ring-offset-2 disabled:pointer-events-none data-[state=open]:bg-accent data-[state=open]:text-muted-foreground">
        <X className="h-4 w-4" />
        <span className="sr-only">Close</span>
      </DialogPrimitive.Close>
    </DialogPrimitive.Content>
  </DialogPortal>
))
DialogContent.displayName = DialogPrimitive.Content.displayName

const DialogHeader = ({
  className,
  ...props
}) => (
  <div
    className={cn("flex flex-col space-y-1.5 text-center sm:text-left", className)}
    {...props} />
)
DialogHeader.displayName = "DialogHeader"

const DialogFooter = ({
  className,
  ...props
}) => (
  <div
    className={cn("flex flex-col-reverse sm:flex-row sm:justify-end sm:space-x-2", className)}
    {...props} />
)
DialogFooter.displayName = "DialogFooter"

const DialogTitle = React.forwardRef(({ className, ...props }, ref) => (
  <DialogPrimitive.Title
    ref={ref}
    className={cn("text-lg font-semibold leading-none tracking-tight", className)}
    {...props} />
))
DialogTitle.displayName = DialogPrimitive.Title.displayName

const DialogDescription = React.forwardRef(({ className, ...props }, ref) => (
  <DialogPrimitive.Description
    ref={ref}
    className={cn("text-sm text-muted-foreground", className)}
    {...props} />
))
DialogDescription.displayName = DialogPrimitive.Description.displayName

export {
  Dialog,
  DialogPortal,
  DialogOverlay,
  DialogTrigger,
  DialogClose,
  DialogContent,
  DialogHeader,
  DialogFooter,
  DialogTitle,
  DialogDescription,
}
#drawer
"use client"

import * as React from "react"
import { Drawer as DrawerPrimitive } from "vaul"

import { cn } from "@/lib/utils"

const Drawer = ({
  shouldScaleBackground = true,
  ...props
}) => (
  <DrawerPrimitive.Root shouldScaleBackground={shouldScaleBackground} {...props} />
)
Drawer.displayName = "Drawer"

const DrawerTrigger = DrawerPrimitive.Trigger

const DrawerPortal = DrawerPrimitive.Portal

const DrawerClose = DrawerPrimitive.Close

const DrawerOverlay = React.forwardRef(({ className, ...props }, ref) => (
  <DrawerPrimitive.Overlay
    ref={ref}
    className={cn("fixed inset-0 z-50 bg-black/80", className)}
    {...props} />
))
DrawerOverlay.displayName = DrawerPrimitive.Overlay.displayName

const DrawerContent = React.forwardRef(({ className, children, ...props }, ref) => (
  <DrawerPortal>
    <DrawerOverlay />
    <DrawerPrimitive.Content
      ref={ref}
      className={cn(
        "fixed inset-x-0 bottom-0 z-50 mt-24 flex h-auto flex-col rounded-t-[10px] border bg-background",
        className
      )}
      {...props}>
      <div className="mx-auto mt-4 h-2 w-[100px] rounded-full bg-muted" />
      {children}
    </DrawerPrimitive.Content>
  </DrawerPortal>
))
DrawerContent.displayName = "DrawerContent"

const DrawerHeader = ({
  className,
  ...props
}) => (
  <div
    className={cn("grid gap-1.5 p-4 text-center sm:text-left", className)}
    {...props} />
)
DrawerHeader.displayName = "DrawerHeader"

const DrawerFooter = ({
  className,
  ...props
}) => (
  <div className={cn("mt-auto flex flex-col gap-2 p-4", className)} {...props} />
)
DrawerFooter.displayName = "DrawerFooter"

const DrawerTitle = React.forwardRef(({ className, ...props }, ref) => (
  <DrawerPrimitive.Title
    ref={ref}
    className={cn("text-lg font-semibold leading-none tracking-tight", className)}
    {...props} />
))
DrawerTitle.displayName = DrawerPrimitive.Title.displayName

const DrawerDescription = React.forwardRef(({ className, ...props }, ref) => (
  <DrawerPrimitive.Description
    ref={ref}
    className={cn("text-sm text-muted-foreground", className)}
    {...props} />
))
DrawerDescription.displayName = DrawerPrimitive.Description.displayName

export {
  Drawer,
  DrawerPortal,
  DrawerOverlay,
  DrawerTrigger,
  DrawerClose,
  DrawerContent,
  DrawerHeader,
  DrawerFooter,
  DrawerTitle,
  DrawerDescription,
}

#dropdown-menu
import * as React from "react"
import * as DropdownMenuPrimitive from "@radix-ui/react-dropdown-menu"
import { Check, ChevronRight, Circle } from "lucide-react"

import { cn } from "@/lib/utils"

const DropdownMenu = DropdownMenuPrimitive.Root

const DropdownMenuTrigger = DropdownMenuPrimitive.Trigger

const DropdownMenuGroup = DropdownMenuPrimitive.Group

const DropdownMenuPortal = DropdownMenuPrimitive.Portal

const DropdownMenuSub = DropdownMenuPrimitive.Sub

const DropdownMenuRadioGroup = DropdownMenuPrimitive.RadioGroup

const DropdownMenuSubTrigger = React.forwardRef(({ className, inset, children, ...props }, ref) => (
  <DropdownMenuPrimitive.SubTrigger
    ref={ref}
    className={cn(
      "flex cursor-default gap-2 select-none items-center rounded-sm px-2 py-1.5 text-sm outline-none focus:bg-accent data-[state=open]:bg-accent [&_svg]:pointer-events-none [&_svg]:size-4 [&_svg]:shrink-0",
      inset && "pl-8",
      className
    )}
    {...props}>
    {children}
    <ChevronRight className="ml-auto" />
  </DropdownMenuPrimitive.SubTrigger>
))
DropdownMenuSubTrigger.displayName =
  DropdownMenuPrimitive.SubTrigger.displayName

const DropdownMenuSubContent = React.forwardRef(({ className, ...props }, ref) => (
  <DropdownMenuPrimitive.SubContent
    ref={ref}
    className={cn(
      "z-50 min-w-[8rem] overflow-hidden rounded-md border bg-popover p-1 text-popover-foreground shadow-lg data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 data-[side=bottom]:slide-in-from-top-2 data-[side=left]:slide-in-from-right-2 data-[side=right]:slide-in-from-left-2 data-[side=top]:slide-in-from-bottom-2",
      className
    )}
    {...props} />
))
DropdownMenuSubContent.displayName =
  DropdownMenuPrimitive.SubContent.displayName

const DropdownMenuContent = React.forwardRef(({ className, sideOffset = 4, ...props }, ref) => (
  <DropdownMenuPrimitive.Portal>
    <DropdownMenuPrimitive.Content
      ref={ref}
      sideOffset={sideOffset}
      className={cn(
        "z-50 min-w-[8rem] overflow-hidden rounded-md border bg-popover p-1 text-popover-foreground shadow-md",
        "data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 data-[side=bottom]:slide-in-from-top-2 data-[side=left]:slide-in-from-right-2 data-[side=right]:slide-in-from-left-2 data-[side=top]:slide-in-from-bottom-2",
        className
      )}
      {...props} />
  </DropdownMenuPrimitive.Portal>
))
DropdownMenuContent.displayName = DropdownMenuPrimitive.Content.displayName

const DropdownMenuItem = React.forwardRef(({ className, inset, ...props }, ref) => (
  <DropdownMenuPrimitive.Item
    ref={ref}
    className={cn(
      "relative flex cursor-default select-none items-center gap-2 rounded-sm px-2 py-1.5 text-sm outline-none transition-colors focus:bg-accent focus:text-accent-foreground data-[disabled]:pointer-events-none data-[disabled]:opacity-50 [&>svg]:size-4 [&>svg]:shrink-0",
      inset && "pl-8",
      className
    )}
    {...props} />
))
DropdownMenuItem.displayName = DropdownMenuPrimitive.Item.displayName

const DropdownMenuCheckboxItem = React.forwardRef(({ className, children, checked, ...props }, ref) => (
  <DropdownMenuPrimitive.CheckboxItem
    ref={ref}
    className={cn(
      "relative flex cursor-default select-none items-center rounded-sm py-1.5 pl-8 pr-2 text-sm outline-none transition-colors focus:bg-accent focus:text-accent-foreground data-[disabled]:pointer-events-none data-[disabled]:opacity-50",
      className
    )}
    checked={checked}
    {...props}>
    <span className="absolute left-2 flex h-3.5 w-3.5 items-center justify-center">
      <DropdownMenuPrimitive.ItemIndicator>
        <Check className="h-4 w-4" />
      </DropdownMenuPrimitive.ItemIndicator>
    </span>
    {children}
  </DropdownMenuPrimitive.CheckboxItem>
))
DropdownMenuCheckboxItem.displayName =
  DropdownMenuPrimitive.CheckboxItem.displayName

const DropdownMenuRadioItem = React.forwardRef(({ className, children, ...props }, ref) => (
  <DropdownMenuPrimitive.RadioItem
    ref={ref}
    className={cn(
      "relative flex cursor-default select-none items-center rounded-sm py-1.5 pl-8 pr-2 text-sm outline-none transition-colors focus:bg-accent focus:text-accent-foreground data-[disabled]:pointer-events-none data-[disabled]:opacity-50",
      className
    )}
    {...props}>
    <span className="absolute left-2 flex h-3.5 w-3.5 items-center justify-center">
      <DropdownMenuPrimitive.ItemIndicator>
        <Circle className="h-2 w-2 fill-current" />
      </DropdownMenuPrimitive.ItemIndicator>
    </span>
    {children}
  </DropdownMenuPrimitive.RadioItem>
))
DropdownMenuRadioItem.displayName = DropdownMenuPrimitive.RadioItem.displayName

const DropdownMenuLabel = React.forwardRef(({ className, inset, ...props }, ref) => (
  <DropdownMenuPrimitive.Label
    ref={ref}
    className={cn("px-2 py-1.5 text-sm font-semibold", inset && "pl-8", className)}
    {...props} />
))
DropdownMenuLabel.displayName = DropdownMenuPrimitive.Label.displayName

const DropdownMenuSeparator = React.forwardRef(({ className, ...props }, ref) => (
  <DropdownMenuPrimitive.Separator
    ref={ref}
    className={cn("-mx-1 my-1 h-px bg-muted", className)}
    {...props} />
))
DropdownMenuSeparator.displayName = DropdownMenuPrimitive.Separator.displayName

const DropdownMenuShortcut = ({
  className,
  ...props
}) => {
  return (
    (<span
      className={cn("ml-auto text-xs tracking-widest opacity-60", className)}
      {...props} />)
  );
}
DropdownMenuShortcut.displayName = "DropdownMenuShortcut"

export {
  DropdownMenu,
  DropdownMenuTrigger,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuCheckboxItem,
  DropdownMenuRadioItem,
  DropdownMenuLabel,
  DropdownMenuSeparator,
  DropdownMenuShortcut,
  DropdownMenuGroup,
  DropdownMenuPortal,
  DropdownMenuSub,
  DropdownMenuSubContent,
  DropdownMenuSubTrigger,
  DropdownMenuRadioGroup,
}
#form
"use client";
import * as React from "react"
import { Slot } from "@radix-ui/react-slot"
import { Controller, FormProvider, useFormContext } from "react-hook-form";

import { cn } from "@/lib/utils"
import { Label } from "@/components/ui/label"

const Form = FormProvider

const FormFieldContext = React.createContext({})

const FormField = (
  {
    ...props
  }
) => {
  return (
    (<FormFieldContext.Provider value={{ name: props.name }}>
      <Controller {...props} />
    </FormFieldContext.Provider>)
  );
}

const useFormField = () => {
  const fieldContext = React.useContext(FormFieldContext)
  const itemContext = React.useContext(FormItemContext)
  const { getFieldState, formState } = useFormContext()

  const fieldState = getFieldState(fieldContext.name, formState)

  if (!fieldContext) {
    throw new Error("useFormField should be used within <FormField>")
  }

  const { id } = itemContext

  return {
    id,
    name: fieldContext.name,
    formItemId: `${id}-form-item`,
    formDescriptionId: `${id}-form-item-description`,
    formMessageId: `${id}-form-item-message`,
    ...fieldState,
  }
}

const FormItemContext = React.createContext({})

const FormItem = React.forwardRef(({ className, ...props }, ref) => {
  const id = React.useId()

  return (
    (<FormItemContext.Provider value={{ id }}>
      <div ref={ref} className={cn("space-y-2", className)} {...props} />
    </FormItemContext.Provider>)
  );
})
FormItem.displayName = "FormItem"

const FormLabel = React.forwardRef(({ className, ...props }, ref) => {
  const { error, formItemId } = useFormField()

  return (
    (<Label
      ref={ref}
      className={cn(error && "text-destructive", className)}
      htmlFor={formItemId}
      {...props} />)
  );
})
FormLabel.displayName = "FormLabel"

const FormControl = React.forwardRef(({ ...props }, ref) => {
  const { error, formItemId, formDescriptionId, formMessageId } = useFormField()

  return (
    (<Slot
      ref={ref}
      id={formItemId}
      aria-describedby={
        !error
          ? `${formDescriptionId}`
          : `${formDescriptionId} ${formMessageId}`
      }
      aria-invalid={!!error}
      {...props} />)
  );
})
FormControl.displayName = "FormControl"

const FormDescription = React.forwardRef(({ className, ...props }, ref) => {
  const { formDescriptionId } = useFormField()

  return (
    (<p
      ref={ref}
      id={formDescriptionId}
      className={cn("text-[0.8rem] text-muted-foreground", className)}
      {...props} />)
  );
})
FormDescription.displayName = "FormDescription"

const FormMessage = React.forwardRef(({ className, children, ...props }, ref) => {
  const { error, formMessageId } = useFormField()
  const body = error ? String(error?.message) : children

  if (!body) {
    return null
  }

  return (
    (<p
      ref={ref}
      id={formMessageId}
      className={cn("text-[0.8rem] font-medium text-destructive", className)}
      {...props}>
      {body}
    </p>)
  );
})
FormMessage.displayName = "FormMessage"

export {
  useFormField,
  Form,
  FormItem,
  FormLabel,
  FormControl,
  FormDescription,
  FormMessage,
  FormField,
}

#hover-card
"use client"

import * as React from "react"
import * as HoverCardPrimitive from "@radix-ui/react-hover-card"

import { cn } from "@/lib/utils"

const HoverCard = HoverCardPrimitive.Root

const HoverCardTrigger = HoverCardPrimitive.Trigger

const HoverCardContent = React.forwardRef(({ className, align = "center", sideOffset = 4, ...props }, ref) => (
  <HoverCardPrimitive.Content
    ref={ref}
    align={align}
    sideOffset={sideOffset}
    className={cn(
      "z-50 w-64 rounded-md border bg-popover p-4 text-popover-foreground shadow-md outline-none data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 data-[side=bottom]:slide-in-from-top-2 data-[side=left]:slide-in-from-right-2 data-[side=right]:slide-in-from-left-2 data-[side=top]:slide-in-from-bottom-2",
      className
    )}
    {...props} />
))
HoverCardContent.displayName = HoverCardPrimitive.Content.displayName

export { HoverCard, HoverCardTrigger, HoverCardContent }

#image-helpers
const WIX_MEDIA_HOSTS = {
  "media.base44.com": "/images/public/",
  "static.wixstatic.com": "/media/",
}

export const DEFAULT_TRANSFORM_WIDTH = 1024
export const IMAGE_LOAD_MODE = {
  OPTIMIZED: "optimized",
  ORIGINAL: "original",
  FALLBACK: "fallback",
}

const DEVICE_PIXEL_RATIOS = [1, 2, 3]
const MAX_DIMENSION = 6000

export function splitImageProps(props) {
  const wrapperProps = {}
  const imageProps = {}
  for (const [key, value] of Object.entries(props)) {
    if (key.startsWith("data-")) wrapperProps[key] = value
    else imageProps[key] = value
  }
  return { wrapperProps, imageProps }
}

export function getImagePreviewClassName(className, currentClassName, baselineClassName) {
  const sourceClasses = new Set((className || "").split(/\s+/))
  const baselineClasses = new Set(baselineClassName.split(/\s+/))
  return currentClassName.split(/\s+/).filter((token) =>
    !["inline-block", "relative"].includes(token) || !baselineClasses.has(token) || sourceClasses.has(token)
  ).join(" ")
}

/** Returns transform metadata only for canonical public Wix image URLs. */
export function parseWixMediaUrl(src) {
  try {
    const url = new URL(src)
    if (
      url.protocol !== "https:" ||
      url.username ||
      url.password ||
      (url.port && url.port !== "443")
    ) {
      return null
    }

    const pathPrefix = WIX_MEDIA_HOSTS[url.hostname]
    if (!pathPrefix) return null

    const transformed = url.pathname.match(/^(.*)\/v1\/(?:fill|fit)\/[^/]+\/[^/]+$/i)
    const basePath = transformed ? transformed[1] : url.pathname
    const filename = basePath.split("/").pop()
    if (
      !basePath.startsWith(pathPrefix) ||
      !filename ||
      !/\.[a-z0-9]+$/i.test(filename) ||
      /\.svg$/i.test(filename)
    ) {
      return null
    }

    return { baseUrl: `${url.origin}${basePath}`, filename }
  } catch {
    return null
  }
}

const clampDim = (n) => Math.min(Math.max(Math.round(n), 1), MAX_DIMENSION)
const clamp01 = (n) => Math.min(1, Math.max(0, n))

export function buildTransformUrl(
  { baseUrl, filename },
  { width, height, crop, focalPoint, quality }
) {
  const params = [`w_${clampDim(width)}`, `h_${clampDim(height || width)}`]
  if (crop) {
    params.push(
      focalPoint
        ? `fp_${clamp01(focalPoint.x).toFixed(2)}_${clamp01(focalPoint.y).toFixed(2)}`
        : "al_c"
    )
  }
  // Preserve GIF delivery; negotiate AVIF/WebP for other images via Accept.
  const encoding = /\.gif$/i.test(filename) ? "enc_webp" : "enc_auto"
  params.push(`q_${quality}`, "usm_0.66_1.00_0.01", encoding, "quality_auto")
  return `${baseUrl}/v1/${crop ? "fill" : "fit"}/${params.join(",")}/${filename}`
}

export function buildSrcSet(parsed, options) {
  return DEVICE_PIXEL_RATIOS.map(
    (dpr) =>
      `${buildTransformUrl(parsed, {
        ...options,
        width: options.width * dpr,
        height: options.height ? options.height * dpr : undefined,
      })} ${dpr}x`
  ).join(", ")
}

export function getOriginalImageUrl(src, parsed) {
  return parsed?.baseUrl || src
}

export function nextImageLoadMode(mode) {
  return mode === IMAGE_LOAD_MODE.OPTIMIZED
    ? IMAGE_LOAD_MODE.ORIGINAL
    : IMAGE_LOAD_MODE.FALLBACK
}
#image
import * as React from "react"
import { ResponsiveImage } from "./responsive-image"
import {
  getOriginalImageUrl,
  IMAGE_LOAD_MODE,
  nextImageLoadMode,
  parseWixMediaUrl,
} from "./image-helpers"

const FALLBACK_IMAGE_URL =
  "https://static.wixstatic.com/media/12d367_4f26ccd17f8f4e3a8958306ea08c2332~mv2.png"

/**
 * Image with built-in Wix Media Platform support: canonical public images on
 * media.base44.com and static.wixstatic.com/media are resized to the rendered
 * container per device pixel ratio and served as AVIF/WebP when supported; `fittingType="fill"`
 * crops server-side, optionally anchored at a focal point. Other URLs render
 * as a plain <img>. Failed transforms retry the original URL; only a broken
 * original swaps to the generic fallback image.
 */
const Image = React.forwardRef(
  (
    {
      src: source,
      fittingType = "fill",
      originWidth,
      originHeight,
      focalPointX,
      focalPointY,
      quality = 90,
      onError,
      ...props
    },
    ref
  ) => {
    const [previewSource, setPreviewSource] = React.useState(null)
    const preview = previewSource?.source === source ? previewSource : null
    const src = preview ? preview.value : source
    const replaceSource = (value, className) => setPreviewSource({
      source, value, className, sourceClassName: props.className,
    })
    React.useEffect(() => setPreviewSource(null), [source])
    const parsedSource = src && src !== FALLBACK_IMAGE_URL ? parseWixMediaUrl(src) : null
    const initialMode = parsedSource ? IMAGE_LOAD_MODE.OPTIMIZED : IMAGE_LOAD_MODE.ORIGINAL
    const [loadState, setLoadState] = React.useState({ src, mode: initialMode })
    const mode = loadState.src === src ? loadState.mode : initialMode

    React.useEffect(() => {
      setLoadState({ src, mode: initialMode })
    }, [src, initialMode])

    const handleError = (event) => {
      if (mode === IMAGE_LOAD_MODE.FALLBACK) return
      const nextMode = nextImageLoadMode(mode)
      setLoadState({ src, mode: nextMode })
      if (nextMode === IMAGE_LOAD_MODE.FALLBACK) onError?.(event)
    }

    const imageProps = {
      ...props,
      className: preview && preview.sourceClassName === props.className ? preview.className : props.className,
      onError: handleError,
    }

    if (!src) {
      // Renders as a real <img> (not a <div>) — the visual editor's
      // click-to-edit toolbar keys its "Replace Image" action off the DOM
      // tag being `img`, so a placeholder div would be unrecoverable in the
      // editor. FALLBACK_IMAGE_URL doubles as the "no image chosen" graphic.
      return <img ref={ref} src={FALLBACK_IMAGE_URL} {...imageProps} data-empty-image />
    }

    // A failed transform retries the underlying original as a plain image.
    // Only a failure of that original advances to the generic fallback.
    const parsed = mode === IMAGE_LOAD_MODE.OPTIMIZED ? parsedSource : null

    if (!parsed) {
      const isErrorMode = mode === IMAGE_LOAD_MODE.FALLBACK
      const imageSrc = isErrorMode ? FALLBACK_IMAGE_URL : getOriginalImageUrl(src, parsedSource)
      return (
        <img ref={ref} src={imageSrc} {...imageProps} data-error-image={isErrorMode || undefined} />
      )
    }

    const focalPoint =
      typeof focalPointX === "number" && typeof focalPointY === "number"
        ? { x: focalPointX, y: focalPointY }
        : undefined
    // Origin dimensions are optional — when known they stabilize layout via
    // the wrapper's aspect-ratio before the image loads.
    const aspectRatio =
      originWidth && originHeight ? `${originWidth} / ${originHeight}` : undefined

    return (
      <ResponsiveImage
        ref={ref}
        src={src}
        parsed={parsed}
        onSourceChange={replaceSource}
        fittingType={fittingType}
        focalPoint={focalPoint}
        quality={quality}
        aspectRatio={aspectRatio}
        {...imageProps}
      />
    )
  }
)
Image.displayName = "Image"

export { Image }

#input-otp
import * as React from "react"
import { OTPInput, OTPInputContext } from "input-otp"
import { Minus } from "lucide-react"

import { cn } from "@/lib/utils"

const InputOTP = React.forwardRef(({ className, containerClassName, ...props }, ref) => (
  <OTPInput
    ref={ref}
    containerClassName={cn("flex items-center gap-2 has-[:disabled]:opacity-50", containerClassName)}
    className={cn("disabled:cursor-not-allowed", className)}
    {...props} />
))
InputOTP.displayName = "InputOTP"

const InputOTPGroup = React.forwardRef(({ className, ...props }, ref) => (
  <div ref={ref} className={cn("flex items-center", className)} {...props} />
))
InputOTPGroup.displayName = "InputOTPGroup"

const InputOTPSlot = React.forwardRef(({ index, className, ...props }, ref) => {
  const inputOTPContext = React.useContext(OTPInputContext)
  const { char, hasFakeCaret, isActive } = inputOTPContext.slots[index]

  return (
    (<div
      ref={ref}
      className={cn(
        "relative flex h-9 w-9 items-center justify-center border-y border-r border-input text-sm shadow-sm transition-all first:rounded-l-md first:border-l last:rounded-r-md",
        isActive && "z-10 ring-1 ring-ring",
        className
      )}
      {...props}>
      {char}
      {hasFakeCaret && (
        <div
          className="pointer-events-none absolute inset-0 flex items-center justify-center">
          <div className="h-4 w-px animate-caret-blink bg-foreground duration-1000" />
        </div>
      )}
    </div>)
  );
})
InputOTPSlot.displayName = "InputOTPSlot"

const InputOTPSeparator = React.forwardRef(({ ...props }, ref) => (
  <div ref={ref} role="separator" {...props}>
    <Minus />
  </div>
))
InputOTPSeparator.displayName = "InputOTPSeparator"

export { InputOTP, InputOTPGroup, InputOTPSlot, InputOTPSeparator }
#input
import * as React from "react"

import { cn } from "@/lib/utils"

const Input = React.forwardRef(({ className, type, ...props }, ref) => {
  return (
    (<input
      type={type}
      className={cn(
        "flex h-9 w-full rounded-md border border-input bg-transparent px-3 py-1 text-base shadow-sm transition-colors file:border-0 file:bg-transparent file:text-sm file:font-medium file:text-foreground placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:cursor-not-allowed disabled:opacity-50 md:text-sm",
        className
      )}
      ref={ref}
      {...props} />)
  );
})
Input.displayName = "Input"

export { Input }
#label
import * as React from "react"
import * as LabelPrimitive from "@radix-ui/react-label"
import { cva } from "class-variance-authority";

import { cn } from "@/lib/utils"

const labelVariants = cva(
  "text-sm font-medium leading-none peer-disabled:cursor-not-allowed peer-disabled:opacity-70"
)

const Label = React.forwardRef(({ className, ...props }, ref) => (
  <LabelPrimitive.Root ref={ref} className={cn(labelVariants(), className)} {...props} />
))
Label.displayName = LabelPrimitive.Root.displayName

export { Label }
#menubar
"use client"

import * as React from "react"
import * as MenubarPrimitive from "@radix-ui/react-menubar"
import { Check, ChevronRight, Circle } from "lucide-react"

import { cn } from "@/lib/utils"

function MenubarMenu({
  ...props
}) {
  return <MenubarPrimitive.Menu {...props} />;
}

function MenubarGroup({
  ...props
}) {
  return <MenubarPrimitive.Group {...props} />;
}

function MenubarPortal({
  ...props
}) {
  return <MenubarPrimitive.Portal {...props} />;
}

function MenubarRadioGroup({
  ...props
}) {
  return <MenubarPrimitive.RadioGroup {...props} />;
}

function MenubarSub({
  ...props
}) {
  return <MenubarPrimitive.Sub data-slot="menubar-sub" {...props} />;
}

const Menubar = React.forwardRef(({ className, ...props }, ref) => (
  <MenubarPrimitive.Root
    ref={ref}
    className={cn(
      "flex h-9 items-center space-x-1 rounded-md border bg-background p-1 shadow-sm",
      className
    )}
    {...props} />
))
Menubar.displayName = MenubarPrimitive.Root.displayName

const MenubarTrigger = React.forwardRef(({ className, ...props }, ref) => (
  <MenubarPrimitive.Trigger
    ref={ref}
    className={cn(
      "flex cursor-default select-none items-center rounded-sm px-3 py-1 text-sm font-medium outline-none focus:bg-accent focus:text-accent-foreground data-[state=open]:bg-accent data-[state=open]:text-accent-foreground",
      className
    )}
    {...props} />
))
MenubarTrigger.displayName = MenubarPrimitive.Trigger.displayName

const MenubarSubTrigger = React.forwardRef(({ className, inset, children, ...props }, ref) => (
  <MenubarPrimitive.SubTrigger
    ref={ref}
    className={cn(
      "flex cursor-default select-none items-center rounded-sm px-2 py-1.5 text-sm outline-none focus:bg-accent focus:text-accent-foreground data-[state=open]:bg-accent data-[state=open]:text-accent-foreground",
      inset && "pl-8",
      className
    )}
    {...props}>
    {children}
    <ChevronRight className="ml-auto h-4 w-4" />
  </MenubarPrimitive.SubTrigger>
))
MenubarSubTrigger.displayName = MenubarPrimitive.SubTrigger.displayName

const MenubarSubContent = React.forwardRef(({ className, ...props }, ref) => (
  <MenubarPrimitive.SubContent
    ref={ref}
    className={cn(
      "z-50 min-w-[8rem] overflow-hidden rounded-md border bg-popover p-1 text-popover-foreground shadow-lg data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 data-[side=bottom]:slide-in-from-top-2 data-[side=left]:slide-in-from-right-2 data-[side=right]:slide-in-from-left-2 data-[side=top]:slide-in-from-bottom-2",
      className
    )}
    {...props} />
))
MenubarSubContent.displayName = MenubarPrimitive.SubContent.displayName

const MenubarContent = React.forwardRef((
  { className, align = "start", alignOffset = -4, sideOffset = 8, ...props },
  ref
) => (
  <MenubarPrimitive.Portal>
    <MenubarPrimitive.Content
      ref={ref}
      align={align}
      alignOffset={alignOffset}
      sideOffset={sideOffset}
      className={cn(
        "z-50 min-w-[12rem] overflow-hidden rounded-md border bg-popover p-1 text-popover-foreground shadow-md data-[state=open]:animate-in data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 data-[side=bottom]:slide-in-from-top-2 data-[side=left]:slide-in-from-right-2 data-[side=right]:slide-in-from-left-2 data-[side=top]:slide-in-from-bottom-2",
        className
      )}
      {...props} />
  </MenubarPrimitive.Portal>
))
MenubarContent.displayName = MenubarPrimitive.Content.displayName

const MenubarItem = React.forwardRef(({ className, inset, ...props }, ref) => (
  <MenubarPrimitive.Item
    ref={ref}
    className={cn(
      "relative flex cursor-default select-none items-center rounded-sm px-2 py-1.5 text-sm outline-none focus:bg-accent focus:text-accent-foreground data-[disabled]:pointer-events-none data-[disabled]:opacity-50",
      inset && "pl-8",
      className
    )}
    {...props} />
))
MenubarItem.displayName = MenubarPrimitive.Item.displayName

const MenubarCheckboxItem = React.forwardRef(({ className, children, checked, ...props }, ref) => (
  <MenubarPrimitive.CheckboxItem
    ref={ref}
    className={cn(
      "relative flex cursor-default select-none items-center rounded-sm py-1.5 pl-8 pr-2 text-sm outline-none focus:bg-accent focus:text-accent-foreground data-[disabled]:pointer-events-none data-[disabled]:opacity-50",
      className
    )}
    checked={checked}
    {...props}>
    <span className="absolute left-2 flex h-3.5 w-3.5 items-center justify-center">
      <MenubarPrimitive.ItemIndicator>
        <Check className="h-4 w-4" />
      </MenubarPrimitive.ItemIndicator>
    </span>
    {children}
  </MenubarPrimitive.CheckboxItem>
))
MenubarCheckboxItem.displayName = MenubarPrimitive.CheckboxItem.displayName

const MenubarRadioItem = React.forwardRef(({ className, children, ...props }, ref) => (
  <MenubarPrimitive.RadioItem
    ref={ref}
    className={cn(
      "relative flex cursor-default select-none items-center rounded-sm py-1.5 pl-8 pr-2 text-sm outline-none focus:bg-accent focus:text-accent-foreground data-[disabled]:pointer-events-none data-[disabled]:opacity-50",
      className
    )}
    {...props}>
    <span className="absolute left-2 flex h-3.5 w-3.5 items-center justify-center">
      <MenubarPrimitive.ItemIndicator>
        <Circle className="h-4 w-4 fill-current" />
      </MenubarPrimitive.ItemIndicator>
    </span>
    {children}
  </MenubarPrimitive.RadioItem>
))
MenubarRadioItem.displayName = MenubarPrimitive.RadioItem.displayName

const MenubarLabel = React.forwardRef(({ className, inset, ...props }, ref) => (
  <MenubarPrimitive.Label
    ref={ref}
    className={cn("px-2 py-1.5 text-sm font-semibold", inset && "pl-8", className)}
    {...props} />
))
MenubarLabel.displayName = MenubarPrimitive.Label.displayName

const MenubarSeparator = React.forwardRef(({ className, ...props }, ref) => (
  <MenubarPrimitive.Separator
    ref={ref}
    className={cn("-mx-1 my-1 h-px bg-muted", className)}
    {...props} />
))
MenubarSeparator.displayName = MenubarPrimitive.Separator.displayName

const MenubarShortcut = ({
  className,
  ...props
}) => {
  return (
    (<span
      className={cn("ml-auto text-xs tracking-widest text-muted-foreground", className)}
      {...props} />)
  );
}
MenubarShortcut.displayname = "MenubarShortcut"

export {
  Menubar,
  MenubarMenu,
  MenubarTrigger,
  MenubarContent,
  MenubarItem,
  MenubarSeparator,
  MenubarLabel,
  MenubarCheckboxItem,
  MenubarRadioGroup,
  MenubarRadioItem,
  MenubarPortal,
  MenubarSubContent,
  MenubarSubTrigger,
  MenubarGroup,
  MenubarSub,
  MenubarShortcut,
}
#navigation-menu
import * as React from "react"
import * as NavigationMenuPrimitive from "@radix-ui/react-navigation-menu"
import { cva } from "class-variance-authority"
import { ChevronDown } from "lucide-react"

import { cn } from "@/lib/utils"

const NavigationMenu = React.forwardRef(({ className, children, ...props }, ref) => (
  <NavigationMenuPrimitive.Root
    ref={ref}
    className={cn(
      "relative z-10 flex max-w-max flex-1 items-center justify-center",
      className
    )}
    {...props}>
    {children}
    <NavigationMenuViewport />
  </NavigationMenuPrimitive.Root>
))
NavigationMenu.displayName = NavigationMenuPrimitive.Root.displayName

const NavigationMenuList = React.forwardRef(({ className, ...props }, ref) => (
  <NavigationMenuPrimitive.List
    ref={ref}
    className={cn(
      "group flex flex-1 list-none items-center justify-center space-x-1",
      className
    )}
    {...props} />
))
NavigationMenuList.displayName = NavigationMenuPrimitive.List.displayName

const NavigationMenuItem = NavigationMenuPrimitive.Item

const navigationMenuTriggerStyle = cva(
  "group inline-flex h-9 w-max items-center justify-center rounded-md bg-background px-4 py-2 text-sm font-medium transition-colors hover:bg-accent hover:text-accent-foreground focus:bg-accent focus:text-accent-foreground focus:outline-none disabled:pointer-events-none disabled:opacity-50 data-[active]:bg-accent/50 data-[state=open]:bg-accent/50"
)

const NavigationMenuTrigger = React.forwardRef(({ className, children, ...props }, ref) => (
  <NavigationMenuPrimitive.Trigger
    ref={ref}
    className={cn(navigationMenuTriggerStyle(), "group", className)}
    {...props}>
    {children}{" "}
    <ChevronDown
      className="relative top-[1px] ml-1 h-3 w-3 transition duration-300 group-data-[state=open]:rotate-180"
      aria-hidden="true" />
  </NavigationMenuPrimitive.Trigger>
))
NavigationMenuTrigger.displayName = NavigationMenuPrimitive.Trigger.displayName

const NavigationMenuContent = React.forwardRef(({ className, ...props }, ref) => (
  <NavigationMenuPrimitive.Content
    ref={ref}
    className={cn(
      "left-0 top-0 w-full data-[motion^=from-]:animate-in data-[motion^=to-]:animate-out data-[motion^=from-]:fade-in data-[motion^=to-]:fade-out data-[motion=from-end]:slide-in-from-right-52 data-[motion=from-start]:slide-in-from-left-52 data-[motion=to-end]:slide-out-to-right-52 data-[motion=to-start]:slide-out-to-left-52 md:absolute md:w-auto ",
      className
    )}
    {...props} />
))
NavigationMenuContent.displayName = NavigationMenuPrimitive.Content.displayName

const NavigationMenuLink = NavigationMenuPrimitive.Link

const NavigationMenuViewport = React.forwardRef(({ className, ...props }, ref) => (
  <div className={cn("absolute left-0 top-full flex justify-center")}>
    <NavigationMenuPrimitive.Viewport
      className={cn(
        "origin-top-center relative mt-1.5 h-[var(--radix-navigation-menu-viewport-height)] w-full overflow-hidden rounded-md border bg-popover text-popover-foreground shadow data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-90 md:w-[var(--radix-navigation-menu-viewport-width)]",
        className
      )}
      ref={ref}
      {...props} />
  </div>
))
NavigationMenuViewport.displayName =
  NavigationMenuPrimitive.Viewport.displayName

const NavigationMenuIndicator = React.forwardRef(({ className, ...props }, ref) => (
  <NavigationMenuPrimitive.Indicator
    ref={ref}
    className={cn(
      "top-full z-[1] flex h-1.5 items-end justify-center overflow-hidden data-[state=visible]:animate-in data-[state=hidden]:animate-out data-[state=hidden]:fade-out data-[state=visible]:fade-in",
      className
    )}
    {...props}>
    <div
      className="relative top-[60%] h-2 w-2 rotate-45 rounded-tl-sm bg-border shadow-md" />
  </NavigationMenuPrimitive.Indicator>
))
NavigationMenuIndicator.displayName =
  NavigationMenuPrimitive.Indicator.displayName

export {
  navigationMenuTriggerStyle,
  NavigationMenu,
  NavigationMenuList,
  NavigationMenuItem,
  NavigationMenuContent,
  NavigationMenuTrigger,
  NavigationMenuLink,
  NavigationMenuIndicator,
  NavigationMenuViewport,
}
#pagination
import * as React from "react"
import { ChevronLeft, ChevronRight, MoreHorizontal } from "lucide-react"

import { cn } from "@/lib/utils"
import { buttonVariants } from "@/components/ui/button";

const Pagination = ({
  className,
  ...props
}) => (
  <nav
    role="navigation"
    aria-label="pagination"
    className={cn("mx-auto flex w-full justify-center", className)}
    {...props} />
)
Pagination.displayName = "Pagination"

const PaginationContent = React.forwardRef(({ className, ...props }, ref) => (
  <ul
    ref={ref}
    className={cn("flex flex-row items-center gap-1", className)}
    {...props} />
))
PaginationContent.displayName = "PaginationContent"

const PaginationItem = React.forwardRef(({ className, ...props }, ref) => (
  <li ref={ref} className={cn("", className)} {...props} />
))
PaginationItem.displayName = "PaginationItem"

const PaginationLink = ({
  className,
  isActive,
  size = "icon",
  ...props
}) => (
  <a
    aria-current={isActive ? "page" : undefined}
    className={cn(buttonVariants({
      variant: isActive ? "outline" : "ghost",
      size,
    }), className)}
    {...props} />
)
PaginationLink.displayName = "PaginationLink"

const PaginationPrevious = ({
  className,
  ...props
}) => (
  <PaginationLink
    aria-label="Go to previous page"
    size="default"
    className={cn("gap-1 pl-2.5", className)}
    {...props}>
    <ChevronLeft className="h-4 w-4" />
    <span>Previous</span>
  </PaginationLink>
)
PaginationPrevious.displayName = "PaginationPrevious"

const PaginationNext = ({
  className,
  ...props
}) => (
  <PaginationLink
    aria-label="Go to next page"
    size="default"
    className={cn("gap-1 pr-2.5", className)}
    {...props}>
    <span>Next</span>
    <ChevronRight className="h-4 w-4" />
  </PaginationLink>
)
PaginationNext.displayName = "PaginationNext"

const PaginationEllipsis = ({
  className,
  ...props
}) => (
  <span
    aria-hidden
    className={cn("flex h-9 w-9 items-center justify-center", className)}
    {...props}>
    <MoreHorizontal className="h-4 w-4" />
    <span className="sr-only">More pages</span>
  </span>
)
PaginationEllipsis.displayName = "PaginationEllipsis"

export {
  Pagination,
  PaginationContent,
  PaginationLink,
  PaginationItem,
  PaginationPrevious,
  PaginationNext,
  PaginationEllipsis,
}
#popover
import * as React from "react"
import * as PopoverPrimitive from "@radix-ui/react-popover"

import { cn } from "@/lib/utils"

const Popover = PopoverPrimitive.Root

const PopoverTrigger = PopoverPrimitive.Trigger

const PopoverAnchor = PopoverPrimitive.Anchor

const PopoverContent = React.forwardRef(({ className, align = "center", sideOffset = 4, ...props }, ref) => (
  <PopoverPrimitive.Portal>
    <PopoverPrimitive.Content
      ref={ref}
      align={align}
      sideOffset={sideOffset}
      className={cn(
        "z-50 w-72 rounded-md border bg-popover p-4 text-popover-foreground shadow-md outline-none data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 data-[side=bottom]:slide-in-from-top-2 data-[side=left]:slide-in-from-right-2 data-[side=right]:slide-in-from-left-2 data-[side=top]:slide-in-from-bottom-2",
        className
      )}
      {...props} />
  </PopoverPrimitive.Portal>
))
PopoverContent.displayName = PopoverPrimitive.Content.displayName

export { Popover, PopoverTrigger, PopoverContent, PopoverAnchor }
#progress
"use client"

import * as React from "react"
import * as ProgressPrimitive from "@radix-ui/react-progress"

import { cn } from "@/lib/utils"

const Progress = React.forwardRef(({ className, value, ...props }, ref) => (
  <ProgressPrimitive.Root
    ref={ref}
    className={cn(
      "relative h-2 w-full overflow-hidden rounded-full bg-primary/20",
      className
    )}
    {...props}>
    <ProgressPrimitive.Indicator
      className="h-full w-full flex-1 bg-primary transition-all"
      style={{ transform: `translateX(-${100 - (value || 0)}%)` }} />
  </ProgressPrimitive.Root>
))
Progress.displayName = ProgressPrimitive.Root.displayName

export { Progress }
#radio-group
import * as React from "react"
import * as RadioGroupPrimitive from "@radix-ui/react-radio-group"
import { Circle } from "lucide-react"

import { cn } from "@/lib/utils"

const RadioGroup = React.forwardRef(({ className, ...props }, ref) => {
  return (<RadioGroupPrimitive.Root className={cn("grid gap-2", className)} {...props} ref={ref} />);
})
RadioGroup.displayName = RadioGroupPrimitive.Root.displayName

const RadioGroupItem = React.forwardRef(({ className, ...props }, ref) => {
  return (
    (<RadioGroupPrimitive.Item
      ref={ref}
      className={cn(
        "aspect-square h-4 w-4 rounded-full border border-primary text-primary shadow focus:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:cursor-not-allowed disabled:opacity-50",
        className
      )}
      {...props}>
      <RadioGroupPrimitive.Indicator className="flex items-center justify-center">
        <Circle className="h-3.5 w-3.5 fill-primary" />
      </RadioGroupPrimitive.Indicator>
    </RadioGroupPrimitive.Item>)
  );
})
RadioGroupItem.displayName = RadioGroupPrimitive.Item.displayName

export { RadioGroup, RadioGroupItem }
#resizable
"use client"

import { GripVertical } from "lucide-react"
import * as ResizablePrimitive from "react-resizable-panels"

import { cn } from "@/lib/utils"

const ResizablePanelGroup = ({
  className,
  ...props
}) => (
  <ResizablePrimitive.PanelGroup
    className={cn(
      "flex h-full w-full data-[panel-group-direction=vertical]:flex-col",
      className
    )}
    {...props} />
)

const ResizablePanel = ResizablePrimitive.Panel

const ResizableHandle = ({
  withHandle,
  className,
  ...props
}) => (
  <ResizablePrimitive.PanelResizeHandle
    className={cn(
      "relative flex w-px items-center justify-center bg-border after:absolute after:inset-y-0 after:left-1/2 after:w-1 after:-translate-x-1/2 focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring focus-visible:ring-offset-1 data-[panel-group-direction=vertical]:h-px data-[panel-group-direction=vertical]:w-full data-[panel-group-direction=vertical]:after:left-0 data-[panel-group-direction=vertical]:after:h-1 data-[panel-group-direction=vertical]:after:w-full data-[panel-group-direction=vertical]:after:-translate-y-1/2 data-[panel-group-direction=vertical]:after:translate-x-0 [&[data-panel-group-direction=vertical]>div]:rotate-90",
      className
    )}
    {...props}>
    {withHandle && (
      <div
        className="z-10 flex h-4 w-3 items-center justify-center rounded-sm border bg-border">
        <GripVertical className="h-2.5 w-2.5" />
      </div>
    )}
  </ResizablePrimitive.PanelResizeHandle>
)

export { ResizablePanelGroup, ResizablePanel, ResizableHandle }
#responsive-image
mport * as React from "react"
import { cn } from "../../lib/utils"
import { buildSrcSet, buildTransformUrl, splitImageProps } from "./image-helpers"
import { useResponsiveImage } from "./use-responsive-image"

export const ResponsiveImage = React.forwardRef(
  ({ src, parsed, fittingType, focalPoint, quality, className, style, aspectRatio, onLoad, onSourceChange, ...props }, ref) => {
    const { wrapperRef, imgRef, loaded, options, handleLoad } = useResponsiveImage(
      { parsed, fittingType, focalPoint, quality, className, onLoad, onSourceChange }, ref
    )
    const { wrapperProps, imageProps } = splitImageProps(props)

    return (
      <span
        ref={wrapperRef}
        className={cn("inline-block relative", className)}
        style={{ aspectRatio, ...style }}
        {...wrapperProps}
        data-base44-image=""
        data-base44-image-src={src}
      >
        {/* Contain both image layers inside the padded content box without adding an edit target. */}
        <span data-source-location={undefined} className="block relative w-full h-full overflow-hidden">
          {options && !loaded && (
            <img
              data-source-location={undefined}
              src={buildTransformUrl(parsed, {
                ...options,
                width: 20,
                height: options.height
                  ? Math.max(1, Math.round((20 * options.height) / options.width))
                  : undefined,
                quality: 20,
              })}
              alt=""
              aria-hidden="true"
              className="w-full h-full inset-0 absolute"
              style={{
                objectFit: fittingType === "fit" ? "contain" : "cover",
                filter: "blur(10px)",
                transform: "scale(1.1)",
              }}
            />
          )}
          {options && (
            <img
              data-source-location={undefined}
              ref={imgRef}
              src={buildTransformUrl(parsed, options)}
              srcSet={buildSrcSet(parsed, options)}
              loading="lazy"
              className={cn(
                "w-full h-full inset-0 absolute",
                fittingType === "fit" ? "object-contain" : "object-cover"
              )}
              onLoad={handleLoad}
              {...imageProps}
            />
          )}
        </span>
      </span>
    )
  }
)
ResponsiveImage.displayName = "ResponsiveImage"
#scroll-area
import * as React from "react"
import * as ScrollAreaPrimitive from "@radix-ui/react-scroll-area"

import { cn } from "@/lib/utils"

const ScrollArea = React.forwardRef(({ className, children, ...props }, ref) => (
  <ScrollAreaPrimitive.Root
    ref={ref}
    className={cn("relative overflow-hidden", className)}
    {...props}>
    <ScrollAreaPrimitive.Viewport className="h-full w-full rounded-[inherit]">
      {children}
    </ScrollAreaPrimitive.Viewport>
    <ScrollBar />
    <ScrollAreaPrimitive.Corner />
  </ScrollAreaPrimitive.Root>
))
ScrollArea.displayName = ScrollAreaPrimitive.Root.displayName

const ScrollBar = React.forwardRef(({ className, orientation = "vertical", ...props }, ref) => (
  <ScrollAreaPrimitive.ScrollAreaScrollbar
    ref={ref}
    orientation={orientation}
    className={cn(
      "flex touch-none select-none transition-colors",
      orientation === "vertical" &&
        "h-full w-2.5 border-l border-l-transparent p-[1px]",
      orientation === "horizontal" &&
        "h-2.5 flex-col border-t border-t-transparent p-[1px]",
      className
    )}
    {...props}>
    <ScrollAreaPrimitive.ScrollAreaThumb className="relative flex-1 rounded-full bg-border" />
  </ScrollAreaPrimitive.ScrollAreaScrollbar>
))
ScrollBar.displayName = ScrollAreaPrimitive.ScrollAreaScrollbar.displayName

export { ScrollArea, ScrollBar }
#select
"use client"

import * as React from "react"
import * as SelectPrimitive from "@radix-ui/react-select"
import { Check, ChevronDown, ChevronUp } from "lucide-react"

import { cn } from "@/lib/utils"

const Select = SelectPrimitive.Root

const SelectGroup = SelectPrimitive.Group

const SelectValue = SelectPrimitive.Value

const SelectTrigger = React.forwardRef(({ className, children, ...props }, ref) => (
  <SelectPrimitive.Trigger
    ref={ref}
    className={cn(
      "flex h-9 w-full items-center justify-between whitespace-nowrap rounded-md border border-input bg-transparent px-3 py-2 text-sm shadow-sm ring-offset-background data-[placeholder]:text-muted-foreground focus:outline-none focus:ring-1 focus:ring-ring disabled:cursor-not-allowed disabled:opacity-50 [&>span]:line-clamp-1",
      className
    )}
    {...props}>
    {children}
    <SelectPrimitive.Icon asChild>
      <ChevronDown className="h-4 w-4 opacity-50" />
    </SelectPrimitive.Icon>
  </SelectPrimitive.Trigger>
))
SelectTrigger.displayName = SelectPrimitive.Trigger.displayName

const SelectScrollUpButton = React.forwardRef(({ className, ...props }, ref) => (
  <SelectPrimitive.ScrollUpButton
    ref={ref}
    className={cn("flex cursor-default items-center justify-center py-1", className)}
    {...props}>
    <ChevronUp className="h-4 w-4" />
  </SelectPrimitive.ScrollUpButton>
))
SelectScrollUpButton.displayName = SelectPrimitive.ScrollUpButton.displayName

const SelectScrollDownButton = React.forwardRef(({ className, ...props }, ref) => (
  <SelectPrimitive.ScrollDownButton
    ref={ref}
    className={cn("flex cursor-default items-center justify-center py-1", className)}
    {...props}>
    <ChevronDown className="h-4 w-4" />
  </SelectPrimitive.ScrollDownButton>
))
SelectScrollDownButton.displayName =
  SelectPrimitive.ScrollDownButton.displayName

const SelectContent = React.forwardRef(({ className, children, position = "popper", ...props }, ref) => (
  <SelectPrimitive.Portal>
    <SelectPrimitive.Content
      ref={ref}
      className={cn(
        "relative z-50 max-h-96 min-w-[8rem] overflow-hidden rounded-md border bg-popover text-popover-foreground shadow-md data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 data-[side=bottom]:slide-in-from-top-2 data-[side=left]:slide-in-from-right-2 data-[side=right]:slide-in-from-left-2 data-[side=top]:slide-in-from-bottom-2",
        position === "popper" &&
          "data-[side=bottom]:translate-y-1 data-[side=left]:-translate-x-1 data-[side=right]:translate-x-1 data-[side=top]:-translate-y-1",
        className
      )}
      position={position}
      {...props}>
      <SelectScrollUpButton />
      <SelectPrimitive.Viewport
        className={cn("p-1", position === "popper" &&
          "h-[var(--radix-select-trigger-height)] w-full min-w-[var(--radix-select-trigger-width)]")}>
        {children}
      </SelectPrimitive.Viewport>
      <SelectScrollDownButton />
    </SelectPrimitive.Content>
  </SelectPrimitive.Portal>
))
SelectContent.displayName = SelectPrimitive.Content.displayName

const SelectLabel = React.forwardRef(({ className, ...props }, ref) => (
  <SelectPrimitive.Label
    ref={ref}
    className={cn("px-2 py-1.5 text-sm font-semibold", className)}
    {...props} />
))
SelectLabel.displayName = SelectPrimitive.Label.displayName

const SelectItem = React.forwardRef(({ className, children, ...props }, ref) => (
  <SelectPrimitive.Item
    ref={ref}
    className={cn(
      "relative flex w-full cursor-default select-none items-center rounded-sm py-1.5 pl-2 pr-8 text-sm outline-none focus:bg-accent focus:text-accent-foreground data-[disabled]:pointer-events-none data-[disabled]:opacity-50",
      className
    )}
    {...props}>
    <span className="absolute right-2 flex h-3.5 w-3.5 items-center justify-center">
      <SelectPrimitive.ItemIndicator>
        <Check className="h-4 w-4" />
      </SelectPrimitive.ItemIndicator>
    </span>
    <SelectPrimitive.ItemText>{children}</SelectPrimitive.ItemText>
  </SelectPrimitive.Item>
))
SelectItem.displayName = SelectPrimitive.Item.displayName

const SelectSeparator = React.forwardRef(({ className, ...props }, ref) => (
  <SelectPrimitive.Separator
    ref={ref}
    className={cn("-mx-1 my-1 h-px bg-muted", className)}
    {...props} />
))
SelectSeparator.displayName = SelectPrimitive.Separator.displayName

export {
  Select,
  SelectGroup,
  SelectValue,
  SelectTrigger,
  SelectContent,
  SelectLabel,
  SelectItem,
  SelectSeparator,
  SelectScrollUpButton,
  SelectScrollDownButton,
}
#separator
import * as React from "react"
import * as SeparatorPrimitive from "@radix-ui/react-separator"

import { cn } from "@/lib/utils"

const Separator = React.forwardRef((
  { className, orientation = "horizontal", decorative = true, ...props },
  ref
) => (
  <SeparatorPrimitive.Root
    ref={ref}
    decorative={decorative}
    orientation={orientation}
    className={cn(
      "shrink-0 bg-border",
      orientation === "horizontal" ? "h-[1px] w-full" : "h-full w-[1px]",
      className
    )}
    {...props} />
))
Separator.displayName = SeparatorPrimitive.Root.displayName

export { Separator }
#sheet
"use client";
import * as React from "react"
import * as SheetPrimitive from "@radix-ui/react-dialog"
import { cva } from "class-variance-authority";
import { X } from "lucide-react"

import { cn } from "@/lib/utils"

const Sheet = SheetPrimitive.Root

const SheetTrigger = SheetPrimitive.Trigger

const SheetClose = SheetPrimitive.Close

const SheetPortal = SheetPrimitive.Portal

const SheetOverlay = React.forwardRef(({ className, ...props }, ref) => (
  <SheetPrimitive.Overlay
    className={cn(
      "fixed inset-0 z-50 bg-black/80  data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0",
      className
    )}
    {...props}
    ref={ref} />
))
SheetOverlay.displayName = SheetPrimitive.Overlay.displayName

const sheetVariants = cva(
  "fixed z-50 gap-4 bg-background p-6 shadow-lg transition ease-in-out data-[state=closed]:duration-300 data-[state=open]:duration-500 data-[state=open]:animate-in data-[state=closed]:animate-out",
  {
    variants: {
      side: {
        top: "inset-x-0 top-0 border-b data-[state=closed]:slide-out-to-top data-[state=open]:slide-in-from-top",
        bottom:
          "inset-x-0 bottom-0 border-t data-[state=closed]:slide-out-to-bottom data-[state=open]:slide-in-from-bottom",
        left: "inset-y-0 left-0 h-full w-3/4 border-r data-[state=closed]:slide-out-to-left data-[state=open]:slide-in-from-left sm:max-w-sm",
        right:
          "inset-y-0 right-0 h-full w-3/4 border-l data-[state=closed]:slide-out-to-right data-[state=open]:slide-in-from-right sm:max-w-sm",
      },
    },
    defaultVariants: {
      side: "right",
    },
  }
)

const SheetContent = React.forwardRef(({ side = "right", className, children, ...props }, ref) => (
  <SheetPortal>
    <SheetOverlay />
    <SheetPrimitive.Content ref={ref} className={cn(sheetVariants({ side }), className)} {...props}>
      <SheetPrimitive.Close
        className="absolute right-4 top-4 rounded-sm opacity-70 ring-offset-background transition-opacity hover:opacity-100 focus:outline-none focus:ring-2 focus:ring-ring focus:ring-offset-2 disabled:pointer-events-none data-[state=open]:bg-secondary">
        <X className="h-4 w-4" />
        <span className="sr-only">Close</span>
      </SheetPrimitive.Close>
      {children}
    </SheetPrimitive.Content>
  </SheetPortal>
))
SheetContent.displayName = SheetPrimitive.Content.displayName

const SheetHeader = ({
  className,
  ...props
}) => (
  <div
    className={cn("flex flex-col space-y-2 text-center sm:text-left", className)}
    {...props} />
)
SheetHeader.displayName = "SheetHeader"

const SheetFooter = ({
  className,
  ...props
}) => (
  <div
    className={cn("flex flex-col-reverse sm:flex-row sm:justify-end sm:space-x-2", className)}
    {...props} />
)
SheetFooter.displayName = "SheetFooter"

const SheetTitle = React.forwardRef(({ className, ...props }, ref) => (
  <SheetPrimitive.Title
    ref={ref}
    className={cn("text-lg font-semibold text-foreground", className)}
    {...props} />
))
SheetTitle.displayName = SheetPrimitive.Title.displayName

const SheetDescription = React.forwardRef(({ className, ...props }, ref) => (
  <SheetPrimitive.Description
    ref={ref}
    className={cn("text-sm text-muted-foreground", className)}
    {...props} />
))
SheetDescription.displayName = SheetPrimitive.Description.displayName

export {
  Sheet,
  SheetPortal,
  SheetOverlay,
  SheetTrigger,
  SheetClose,
  SheetContent,
  SheetHeader,
  SheetFooter,
  SheetTitle,
  SheetDescription,
}
#sidebar
import * as React from "react"
import { Slot } from "@radix-ui/react-slot"
import { cva } from "class-variance-authority";
import { PanelLeft } from "lucide-react"

import { useIsMobile } from "@/hooks/use-mobile"
import { cn } from "@/lib/utils"
import { Button } from "@/components/ui/button"
import { Input } from "@/components/ui/input"
import { Separator } from "@/components/ui/separator"
import { Sheet, SheetContent } from "@/components/ui/sheet"
import { Skeleton } from "@/components/ui/skeleton"
import {
  Tooltip,
  TooltipContent,
  TooltipProvider,
  TooltipTrigger,
} from "@/components/ui/tooltip"

const SIDEBAR_COOKIE_NAME = "sidebar_state"
const SIDEBAR_COOKIE_MAX_AGE = 60 * 60 * 24 * 7
const SIDEBAR_WIDTH = "16rem"
const SIDEBAR_WIDTH_MOBILE = "18rem"
const SIDEBAR_WIDTH_ICON = "3rem"
const SIDEBAR_KEYBOARD_SHORTCUT = "b"

const SidebarContext = React.createContext(null)

function useSidebar() {
  const context = React.useContext(SidebarContext)
  if (!context) {
    throw new Error("useSidebar must be used within a SidebarProvider.")
  }

  return context
}

const SidebarProvider = React.forwardRef((
  {
    defaultOpen = true,
    open: openProp,
    onOpenChange: setOpenProp,
    className,
    style,
    children,
    ...props
  },
  ref
) => {
  const isMobile = useIsMobile()
  const [openMobile, setOpenMobile] = React.useState(false)

  // This is the internal state of the sidebar.
  // We use openProp and setOpenProp for control from outside the component.
  const [_open, _setOpen] = React.useState(defaultOpen)
  const open = openProp ?? _open
  const setOpen = React.useCallback((value) => {
    const openState = typeof value === "function" ? value(open) : value
    if (setOpenProp) {
      setOpenProp(openState)
    } else {
      _setOpen(openState)
    }

    // This sets the cookie to keep the sidebar state.
    document.cookie = `${SIDEBAR_COOKIE_NAME}=${openState}; path=/; max-age=${SIDEBAR_COOKIE_MAX_AGE}`
  }, [setOpenProp, open])

  // Helper to toggle the sidebar.
  const toggleSidebar = React.useCallback(() => {
    return isMobile
      ? setOpenMobile((open) => !open)
      : setOpen((open) => !open);
  }, [isMobile, setOpen, setOpenMobile])

  // Adds a keyboard shortcut to toggle the sidebar.
  React.useEffect(() => {
    const handleKeyDown = (event) => {
      if (
        event.key === SIDEBAR_KEYBOARD_SHORTCUT &&
        (event.metaKey || event.ctrlKey)
      ) {
        event.preventDefault()
        toggleSidebar()
      }
    }

    window.addEventListener("keydown", handleKeyDown)
    return () => window.removeEventListener("keydown", handleKeyDown);
  }, [toggleSidebar])

  // We add a state so that we can do data-state="expanded" or "collapsed".
  // This makes it easier to style the sidebar with Tailwind classes.
  const state = open ? "expanded" : "collapsed"

  const contextValue = React.useMemo(() => ({
    state,
    open,
    setOpen,
    isMobile,
    openMobile,
    setOpenMobile,
    toggleSidebar,
  }), [state, open, setOpen, isMobile, openMobile, setOpenMobile, toggleSidebar])

  return (
    (<SidebarContext.Provider value={contextValue}>
      <TooltipProvider delayDuration={0}>
        <div
          style={
            {
              "--sidebar-width": SIDEBAR_WIDTH,
              "--sidebar-width-icon": SIDEBAR_WIDTH_ICON,
              ...style
            }
          }
          className={cn(
            "group/sidebar-wrapper flex min-h-svh w-full has-[[data-variant=inset]]:bg-sidebar",
            className
          )}
          ref={ref}
          {...props}>
          {children}
        </div>
      </TooltipProvider>
    </SidebarContext.Provider>)
  );
})
SidebarProvider.displayName = "SidebarProvider"

const Sidebar = React.forwardRef((
  {
    side = "left",
    variant = "sidebar",
    collapsible = "offcanvas",
    className,
    children,
    ...props
  },
  ref
) => {
  const { isMobile, state, openMobile, setOpenMobile } = useSidebar()

  if (collapsible === "none") {
    return (
      (<div
        className={cn(
          "flex h-full w-[--sidebar-width] flex-col bg-sidebar text-sidebar-foreground",
          className
        )}
        ref={ref}
        {...props}>
        {children}
      </div>)
    );
  }

  if (isMobile) {
    return (
      (<Sheet open={openMobile} onOpenChange={setOpenMobile} {...props}>
        <SheetContent
          data-sidebar="sidebar"
          data-mobile="true"
          className="w-[--sidebar-width] bg-sidebar p-0 text-sidebar-foreground [&>button]:hidden"
          style={
            {
              "--sidebar-width": SIDEBAR_WIDTH_MOBILE
            }
          }
          side={side}>
          <div className="flex h-full w-full flex-col">{children}</div>
        </SheetContent>
      </Sheet>)
    );
  }

  return (
    (<div
      ref={ref}
      className="group peer hidden text-sidebar-foreground md:block"
      data-state={state}
      data-collapsible={state === "collapsed" ? collapsible : ""}
      data-variant={variant}
      data-side={side}>
      {/* This is what handles the sidebar gap on desktop */}
      <div
        className={cn(
          "relative h-svh w-[--sidebar-width] bg-transparent transition-[width] duration-200 ease-linear",
          "group-data-[collapsible=offcanvas]:w-0",
          "group-data-[side=right]:rotate-180",
          variant === "floating" || variant === "inset"
            ? "group-data-[collapsible=icon]:w-[calc(var(--sidebar-width-icon)_+_theme(spacing.4))]"
            : "group-data-[collapsible=icon]:w-[--sidebar-width-icon]"
        )} />
      <div
        className={cn(
          "fixed inset-y-0 z-10 hidden h-svh w-[--sidebar-width] transition-[left,right,width] duration-200 ease-linear md:flex",
          side === "left"
            ? "left-0 group-data-[collapsible=offcanvas]:left-[calc(var(--sidebar-width)*-1)]"
            : "right-0 group-data-[collapsible=offcanvas]:right-[calc(var(--sidebar-width)*-1)]",
          // Adjust the padding for floating and inset variants.
          variant === "floating" || variant === "inset"
            ? "p-2 group-data-[collapsible=icon]:w-[calc(var(--sidebar-width-icon)_+_theme(spacing.4)_+2px)]"
            : "group-data-[collapsible=icon]:w-[--sidebar-width-icon] group-data-[side=left]:border-r group-data-[side=right]:border-l",
          className
        )}
        {...props}>
        <div
          data-sidebar="sidebar"
          className="flex h-full w-full flex-col bg-sidebar group-data-[variant=floating]:rounded-lg group-data-[variant=floating]:border group-data-[variant=floating]:border-sidebar-border group-data-[variant=floating]:shadow">
          {children}
        </div>
      </div>
    </div>)
  );
})
Sidebar.displayName = "Sidebar"

const SidebarTrigger = React.forwardRef(({ className, onClick, asChild = false, ...props }, ref) => {
  const { toggleSidebar } = useSidebar()

  return (
    (<Button
      ref={ref}
      data-sidebar="trigger"
      variant="ghost"
      size="icon"
      className={cn("h-7 w-7", className)}
      onClick={(event) => {
        onClick?.(event)
        toggleSidebar()
      }}
      asChild={asChild}
      {...props}>
      {asChild ? (
        <PanelLeft />
      ) : (
        <>
          <PanelLeft />
          <span className="sr-only">Toggle Sidebar</span>
        </>
      )}
    </Button>)
  );
})
SidebarTrigger.displayName = "SidebarTrigger"

const SidebarRail = React.forwardRef(({ className, ...props }, ref) => {
  const { toggleSidebar } = useSidebar()

  return (
    (<button
      ref={ref}
      data-sidebar="rail"
      aria-label="Toggle Sidebar"
      tabIndex={-1}
      onClick={toggleSidebar}
      title="Toggle Sidebar"
      className={cn(
        "absolute inset-y-0 z-20 hidden w-4 -translate-x-1/2 transition-all ease-linear after:absolute after:inset-y-0 after:left-1/2 after:w-[2px] hover:after:bg-sidebar-border group-data-[side=left]:-right-4 group-data-[side=right]:left-0 sm:flex",
        "[[data-side=left]_&]:cursor-w-resize [[data-side=right]_&]:cursor-e-resize",
        "[[data-side=left][data-state=collapsed]_&]:cursor-e-resize [[data-side=right][data-state=collapsed]_&]:cursor-w-resize",
        "group-data-[collapsible=offcanvas]:translate-x-0 group-data-[collapsible=offcanvas]:after:left-full group-data-[collapsible=offcanvas]:hover:bg-sidebar",
        "[[data-side=left][data-collapsible=offcanvas]_&]:-right-2",
        "[[data-side=right][data-collapsible=offcanvas]_&]:-left-2",
        className
      )}
      {...props} />)
  );
})
SidebarRail.displayName = "SidebarRail"

const SidebarInset = React.forwardRef(({ className, ...props }, ref) => {
  return (
    (<main
      ref={ref}
      className={cn(
        "relative flex min-h-svh flex-1 flex-col bg-background",
        "peer-data-[variant=inset]:min-h-[calc(100svh-theme(spacing.4))] md:peer-data-[variant=inset]:m-2 md:peer-data-[state=collapsed]:peer-data-[variant=inset]:ml-2 md:peer-data-[variant=inset]:ml-0 md:peer-data-[variant=inset]:rounded-xl md:peer-data-[variant=inset]:shadow",
        className
      )}
      {...props} />)
  );
})
SidebarInset.displayName = "SidebarInset"

const SidebarInput = React.forwardRef(({ className, ...props }, ref) => {
  return (
    (<Input
      ref={ref}
      data-sidebar="input"
      className={cn(
        "h-8 w-full bg-background shadow-none focus-visible:ring-2 focus-visible:ring-sidebar-ring",
        className
      )}
      {...props} />)
  );
})
SidebarInput.displayName = "SidebarInput"

const SidebarHeader = React.forwardRef(({ className, ...props }, ref) => {
  return (
    (<div
      ref={ref}
      data-sidebar="header"
      className={cn("flex flex-col gap-2 p-2", className)}
      {...props} />)
  );
})
SidebarHeader.displayName = "SidebarHeader"

const SidebarFooter = React.forwardRef(({ className, ...props }, ref) => {
  return (
    (<div
      ref={ref}
      data-sidebar="footer"
      className={cn("flex flex-col gap-2 p-2", className)}
      {...props} />)
  );
})
SidebarFooter.displayName = "SidebarFooter"

const SidebarSeparator = React.forwardRef(({ className, ...props }, ref) => {
  return (
    (<Separator
      ref={ref}
      data-sidebar="separator"
      className={cn("mx-2 w-auto bg-sidebar-border", className)}
      {...props} />)
  );
})
SidebarSeparator.displayName = "SidebarSeparator"

const SidebarContent = React.forwardRef(({ className, ...props }, ref) => {
  return (
    (<div
      ref={ref}
      data-sidebar="content"
      className={cn(
        "flex min-h-0 flex-1 flex-col gap-2 overflow-auto group-data-[collapsible=icon]:overflow-hidden",
        className
      )}
      {...props} />)
  );
})
SidebarContent.displayName = "SidebarContent"

const SidebarGroup = React.forwardRef(({ className, ...props }, ref) => {
  return (
    (<div
      ref={ref}
      data-sidebar="group"
      className={cn("relative flex w-full min-w-0 flex-col p-2", className)}
      {...props} />)
  );
})
SidebarGroup.displayName = "SidebarGroup"

const SidebarGroupLabel = React.forwardRef(({ className, asChild = false, ...props }, ref) => {
  const Comp = asChild ? Slot : "div"

  return (
    (<Comp
      ref={ref}
      data-sidebar="group-label"
      className={cn(
        "flex h-8 shrink-0 items-center rounded-md px-2 text-xs font-medium text-sidebar-foreground/70 outline-none ring-sidebar-ring transition-[margin,opacity] duration-200 ease-linear focus-visible:ring-2 [&>svg]:size-4 [&>svg]:shrink-0",
        "group-data-[collapsible=icon]:-mt-8 group-data-[collapsible=icon]:opacity-0",
        className
      )}
      {...props} />)
  );
})
SidebarGroupLabel.displayName = "SidebarGroupLabel"

const SidebarGroupAction = React.forwardRef(({ className, asChild = false, ...props }, ref) => {
  const Comp = asChild ? Slot : "button"

  return (
    (<Comp
      ref={ref}
      data-sidebar="group-action"
      className={cn(
        "absolute right-3 top-3.5 flex aspect-square w-5 items-center justify-center rounded-md p-0 text-sidebar-foreground outline-none ring-sidebar-ring transition-transform hover:bg-sidebar-accent hover:text-sidebar-accent-foreground focus-visible:ring-2 [&>svg]:size-4 [&>svg]:shrink-0",
        // Increases the hit area of the button on mobile.
        "after:absolute after:-inset-2 after:md:hidden",
        "group-data-[collapsible=icon]:hidden",
        className
      )}
      {...props} />)
  );
})
SidebarGroupAction.displayName = "SidebarGroupAction"

const SidebarGroupContent = React.forwardRef(({ className, ...props }, ref) => (
  <div
    ref={ref}
    data-sidebar="group-content"
    className={cn("w-full text-sm", className)}
    {...props} />
))
SidebarGroupContent.displayName = "SidebarGroupContent"

const SidebarMenu = React.forwardRef(({ className, ...props }, ref) => (
  <ul
    ref={ref}
    data-sidebar="menu"
    className={cn("flex w-full min-w-0 flex-col gap-1", className)}
    {...props} />
))
SidebarMenu.displayName = "SidebarMenu"

const SidebarMenuItem = React.forwardRef(({ className, ...props }, ref) => (
  <li
    ref={ref}
    data-sidebar="menu-item"
    className={cn("group/menu-item relative", className)}
    {...props} />
))
SidebarMenuItem.displayName = "SidebarMenuItem"

const sidebarMenuButtonVariants = cva(
  "peer/menu-button flex w-full items-center gap-2 overflow-hidden rounded-md p-2 text-left text-sm outline-none ring-sidebar-ring transition-[width,height,padding] hover:bg-sidebar-accent hover:text-sidebar-accent-foreground focus-visible:ring-2 active:bg-sidebar-accent active:text-sidebar-accent-foreground disabled:pointer-events-none disabled:opacity-50 group-has-[[data-sidebar=menu-action]]/menu-item:pr-8 aria-disabled:pointer-events-none aria-disabled:opacity-50 data-[active=true]:bg-sidebar-accent data-[active=true]:font-medium data-[active=true]:text-sidebar-accent-foreground data-[state=open]:hover:bg-sidebar-accent data-[state=open]:hover:text-sidebar-accent-foreground group-data-[collapsible=icon]:!size-8 group-data-[collapsible=icon]:!p-2 [&>span:last-child]:truncate [&>svg]:size-4 [&>svg]:shrink-0",
  {
    variants: {
      variant: {
        default: "hover:bg-sidebar-accent hover:text-sidebar-accent-foreground",
        outline:
          "bg-background shadow-[0_0_0_1px_hsl(var(--sidebar-border))] hover:bg-sidebar-accent hover:text-sidebar-accent-foreground hover:shadow-[0_0_0_1px_hsl(var(--sidebar-accent))]",
      },
      size: {
        default: "h-8 text-sm",
        sm: "h-7 text-xs",
        lg: "h-12 text-sm group-data-[collapsible=icon]:!p-0",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
)

const SidebarMenuButton = React.forwardRef((
  {
    asChild = false,
    isActive = false,
    variant = "default",
    size = "default",
    tooltip,
    className,
    ...props
  },
  ref
) => {
  const Comp = asChild ? Slot : "button"
  const { isMobile, state } = useSidebar()

  const button = (
    <Comp
      ref={ref}
      data-sidebar="menu-button"
      data-size={size}
      data-active={isActive}
      className={cn(sidebarMenuButtonVariants({ variant, size }), className)}
      {...props} />
  )

  if (!tooltip) {
    return button
  }

  if (typeof tooltip === "string") {
    tooltip = {
      children: tooltip,
    }
  }

  return (
    (<Tooltip>
      <TooltipTrigger asChild>{button}</TooltipTrigger>
      <TooltipContent
        side="right"
        align="center"
        hidden={state !== "collapsed" || isMobile}
        {...tooltip} />
    </Tooltip>)
  );
})
SidebarMenuButton.displayName = "SidebarMenuButton"

const SidebarMenuAction = React.forwardRef(({ className, asChild = false, showOnHover = false, ...props }, ref) => {
  const Comp = asChild ? Slot : "button"

  return (
    (<Comp
      ref={ref}
      data-sidebar="menu-action"
      className={cn(
        "absolute right-1 top-1.5 flex aspect-square w-5 items-center justify-center rounded-md p-0 text-sidebar-foreground outline-none ring-sidebar-ring transition-transform hover:bg-sidebar-accent hover:text-sidebar-accent-foreground focus-visible:ring-2 peer-hover/menu-button:text-sidebar-accent-foreground [&>svg]:size-4 [&>svg]:shrink-0",
        // Increases the hit area of the button on mobile.
        "after:absolute after:-inset-2 after:md:hidden",
        "peer-data-[size=sm]/menu-button:top-1",
        "peer-data-[size=default]/menu-button:top-1.5",
        "peer-data-[size=lg]/menu-button:top-2.5",
        "group-data-[collapsible=icon]:hidden",
        showOnHover &&
        "group-focus-within/menu-item:opacity-100 group-hover/menu-item:opacity-100 data-[state=open]:opacity-100 peer-data-[active=true]/menu-button:text-sidebar-accent-foreground md:opacity-0",
        className
      )}
      {...props} />)
  );
})
SidebarMenuAction.displayName = "SidebarMenuAction"

const SidebarMenuBadge = React.forwardRef(({ className, ...props }, ref) => (
  <div
    ref={ref}
    data-sidebar="menu-badge"
    className={cn(
      "pointer-events-none absolute right-1 flex h-5 min-w-5 select-none items-center justify-center rounded-md px-1 text-xs font-medium tabular-nums text-sidebar-foreground",
      "peer-hover/menu-button:text-sidebar-accent-foreground peer-data-[active=true]/menu-button:text-sidebar-accent-foreground",
      "peer-data-[size=sm]/menu-button:top-1",
      "peer-data-[size=default]/menu-button:top-1.5",
      "peer-data-[size=lg]/menu-button:top-2.5",
      "group-data-[collapsible=icon]:hidden",
      className
    )}
    {...props} />
))
SidebarMenuBadge.displayName = "SidebarMenuBadge"

const SidebarMenuSkeleton = React.forwardRef(({ className, showIcon = false, ...props }, ref) => {
  // Random width between 50 to 90%.
  const width = React.useMemo(() => {
    return `${Math.floor(Math.random() * 40) + 50}%`;
  }, [])

  return (
    (<div
      ref={ref}
      data-sidebar="menu-skeleton"
      className={cn("flex h-8 items-center gap-2 rounded-md px-2", className)}
      {...props}>
      {showIcon && (
        <Skeleton className="size-4 rounded-md" data-sidebar="menu-skeleton-icon" />
      )}
      <Skeleton
        className="h-4 max-w-[--skeleton-width] flex-1"
        data-sidebar="menu-skeleton-text"
        style={
          {
            "--skeleton-width": width
          }
        } />
    </div>)
  );
})
SidebarMenuSkeleton.displayName = "SidebarMenuSkeleton"

const SidebarMenuSub = React.forwardRef(({ className, ...props }, ref) => (
  <ul
    ref={ref}
    data-sidebar="menu-sub"
    className={cn(
      "mx-3.5 flex min-w-0 translate-x-px flex-col gap-1 border-l border-sidebar-border px-2.5 py-0.5",
      "group-data-[collapsible=icon]:hidden",
      className
    )}
    {...props} />
))
SidebarMenuSub.displayName = "SidebarMenuSub"

const SidebarMenuSubItem = React.forwardRef(({ ...props }, ref) => <li ref={ref} {...props} />)
SidebarMenuSubItem.displayName = "SidebarMenuSubItem"

const SidebarMenuSubButton = React.forwardRef(
  ({ asChild = false, size = "md", isActive, className, ...props }, ref) => {
    const Comp = asChild ? Slot : "a"

    return (
      (<Comp
        ref={ref}
        data-sidebar="menu-sub-button"
        data-size={size}
        data-active={isActive}
        className={cn(
          "flex h-7 min-w-0 -translate-x-px items-center gap-2 overflow-hidden rounded-md px-2 text-sidebar-foreground outline-none ring-sidebar-ring hover:bg-sidebar-accent hover:text-sidebar-accent-foreground focus-visible:ring-2 active:bg-sidebar-accent active:text-sidebar-accent-foreground disabled:pointer-events-none disabled:opacity-50 aria-disabled:pointer-events-none aria-disabled:opacity-50 [&>span:last-child]:truncate [&>svg]:size-4 [&>svg]:shrink-0 [&>svg]:text-sidebar-accent-foreground",
          "data-[active=true]:bg-sidebar-accent data-[active=true]:text-sidebar-accent-foreground",
          size === "sm" && "text-xs",
          size === "md" && "text-sm",
          "group-data-[collapsible=icon]:hidden",
          className
        )}
        {...props} />)
    );
  }
)
SidebarMenuSubButton.displayName = "SidebarMenuSubButton"

export {
  Sidebar,
  SidebarContent,
  SidebarFooter,
  SidebarGroup,
  SidebarGroupAction,
  SidebarGroupContent,
  SidebarGroupLabel,
  SidebarHeader,
  SidebarInput,
  SidebarInset,
  SidebarMenu,
  SidebarMenuAction,
  SidebarMenuBadge,
  SidebarMenuButton,
  SidebarMenuItem,
  SidebarMenuSkeleton,
  SidebarMenuSub,
  SidebarMenuSubButton,
  SidebarMenuSubItem,
  SidebarProvider,
  SidebarRail,
  SidebarSeparator,
  SidebarTrigger,
  useSidebar,
}
#skeleton
import { cn } from "@/lib/utils"

function Skeleton({
  className,
  ...props
}) {
  return (
    (<div
      className={cn("animate-pulse rounded-md bg-primary/10", className)}
      {...props} />)
  );
}

export { Skeleton }
#slider
import * as React from "react"
import * as SliderPrimitive from "@radix-ui/react-slider"

import { cn } from "@/lib/utils"

const Slider = React.forwardRef(({ className, ...props }, ref) => (
  <SliderPrimitive.Root
    ref={ref}
    className={cn("relative flex w-full touch-none select-none items-center", className)}
    {...props}>
    <SliderPrimitive.Track
      className="relative h-1.5 w-full grow overflow-hidden rounded-full bg-primary/20">
      <SliderPrimitive.Range className="absolute h-full bg-primary" />
    </SliderPrimitive.Track>
    <SliderPrimitive.Thumb
      className="block h-4 w-4 rounded-full border border-primary/50 bg-background shadow transition-colors focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50" />
  </SliderPrimitive.Root>
))
Slider.displayName = SliderPrimitive.Root.displayName

export { Slider }
#sonner
"use client";
import { useTheme } from "next-themes"
import { Toaster as Sonner } from "sonner"

const Toaster = ({
  ...props
}) => {
  const { theme = "system" } = useTheme()

  return (
    (<Sonner
      theme={theme}
      className="toaster group"
      toastOptions={{
        classNames: {
          toast:
            "group toast group-[.toaster]:bg-background group-[.toaster]:text-foreground group-[.toaster]:border-border group-[.toaster]:shadow-lg",
          description: "group-[.toast]:text-muted-foreground",
          actionButton:
            "group-[.toast]:bg-primary group-[.toast]:text-primary-foreground",
          cancelButton:
            "group-[.toast]:bg-muted group-[.toast]:text-muted-foreground",
        },
      }}
      {...props} />)
  );
}
#switch
import * as React from "react"
import * as SwitchPrimitives from "@radix-ui/react-switch"

import { cn } from "@/lib/utils"

const Switch = React.forwardRef(({ className, ...props }, ref) => (
  <SwitchPrimitives.Root
    className={cn(
      "peer inline-flex h-5 w-9 shrink-0 cursor-pointer items-center rounded-full border-2 border-transparent shadow-sm transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 focus-visible:ring-offset-background disabled:cursor-not-allowed disabled:opacity-50 data-[state=checked]:bg-primary data-[state=unchecked]:bg-input",
      className
    )}
    {...props}
    ref={ref}>
    <SwitchPrimitives.Thumb
      className={cn(
        "pointer-events-none block h-4 w-4 rounded-full bg-background shadow-lg ring-0 transition-transform data-[state=checked]:translate-x-4 data-[state=unchecked]:translate-x-0"
      )} />
  </SwitchPrimitives.Root>
))
Switch.displayName = SwitchPrimitives.Root.displayName

export { Switch }
#table
import * as React from "react"

import { cn } from "@/lib/utils"

const Table = React.forwardRef(({ className, ...props }, ref) => (
  <div className="relative w-full overflow-auto">
    <table
      ref={ref}
      className={cn("w-full caption-bottom text-sm", className)}
      {...props} />
  </div>
))
Table.displayName = "Table"

const TableHeader = React.forwardRef(({ className, ...props }, ref) => (
  <thead ref={ref} className={cn("[&_tr]:border-b", className)} {...props} />
))
TableHeader.displayName = "TableHeader"

const TableBody = React.forwardRef(({ className, ...props }, ref) => (
  <tbody
    ref={ref}
    className={cn("[&_tr:last-child]:border-0", className)}
    {...props} />
))
TableBody.displayName = "TableBody"

const TableFooter = React.forwardRef(({ className, ...props }, ref) => (
  <tfoot
    ref={ref}
    className={cn("border-t bg-muted/50 font-medium [&>tr]:last:border-b-0", className)}
    {...props} />
))
TableFooter.displayName = "TableFooter"

const TableRow = React.forwardRef(({ className, ...props }, ref) => (
  <tr
    ref={ref}
    className={cn(
      "border-b transition-colors hover:bg-muted/50 data-[state=selected]:bg-muted",
      className
    )}
    {...props} />
))
TableRow.displayName = "TableRow"

const TableHead = React.forwardRef(({ className, ...props }, ref) => (
  <th
    ref={ref}
    className={cn(
      "h-10 px-2 text-left align-middle font-medium text-muted-foreground [&:has([role=checkbox])]:pr-0 [&>[role=checkbox]]:translate-y-[2px]",
      className
    )}
    {...props} />
))
TableHead.displayName = "TableHead"

const TableCell = React.forwardRef(({ className, ...props }, ref) => (
  <td
    ref={ref}
    className={cn(
      "p-2 align-middle [&:has([role=checkbox])]:pr-0 [&>[role=checkbox]]:translate-y-[2px]",
      className
    )}
    {...props} />
))
TableCell.displayName = "TableCell"

const TableCaption = React.forwardRef(({ className, ...props }, ref) => (
  <caption
    ref={ref}
    className={cn("mt-4 text-sm text-muted-foreground", className)}
    {...props} />
))
TableCaption.displayName = "TableCaption"

export {
  Table,
  TableHeader,
  TableBody,
  TableFooter,
  TableHead,
  TableRow,
  TableCell,
  TableCaption,
}
#tabs
import * as React from "react"
import * as TabsPrimitive from "@radix-ui/react-tabs"

import { cn } from "@/lib/utils"

const Tabs = TabsPrimitive.Root

const TabsList = React.forwardRef(({ className, ...props }, ref) => (
  <TabsPrimitive.List
    ref={ref}
    className={cn(
      "inline-flex h-9 items-center justify-center rounded-lg bg-muted p-1 text-muted-foreground",
      className
    )}
    {...props} />
))
TabsList.displayName = TabsPrimitive.List.displayName

const TabsTrigger = React.forwardRef(({ className, ...props }, ref) => (
  <TabsPrimitive.Trigger
    ref={ref}
    className={cn(
      "inline-flex items-center justify-center whitespace-nowrap rounded-md px-3 py-1 text-sm font-medium ring-offset-background transition-all focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50 data-[state=active]:bg-background data-[state=active]:text-foreground data-[state=active]:shadow",
      className
    )}
    {...props} />
))
TabsTrigger.displayName = TabsPrimitive.Trigger.displayName

const TabsContent = React.forwardRef(({ className, ...props }, ref) => (
  <TabsPrimitive.Content
    ref={ref}
    className={cn(
      "mt-2 ring-offset-background focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2",
      className
    )}
    {...props} />
))
TabsContent.displayName = TabsPrimitive.Content.displayName

export { Tabs, TabsList, TabsTrigger, TabsContent }
#textarea
import * as React from "react"

import { cn } from "@/lib/utils"

const Textarea = React.forwardRef(({ className, ...props }, ref) => {
  return (
    (<textarea
      className={cn(
        "flex min-h-[60px] w-full rounded-md border border-input bg-transparent px-3 py-2 text-base shadow-sm placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:cursor-not-allowed disabled:opacity-50 md:text-sm",
        className
      )}
      ref={ref}
      {...props} />)
  );
})
Textarea.displayName = "Textarea"

export { Textarea }
#toast
import * as React from "react";
import * as ToastPrimitives from "@radix-ui/react-toast";
import { cva } from "class-variance-authority";
import { X } from "lucide-react";
import { cn } from "@/lib/utils";

const ToastProvider = ToastPrimitives.Provider;

const ToastViewport = React.forwardRef(({ className, ...props }, ref) => (
  <ToastPrimitives.Viewport
    ref={ref}
    className={cn(
      "fixed top-0 z-[100] flex max-h-screen w-full flex-col-reverse p-4 sm:bottom-0 sm:right-0 sm:top-auto sm:flex-col md:max-w-[420px]",
      className
    )}
    {...props}
  />
));
ToastViewport.displayName = ToastPrimitives.Viewport.displayName;

const toastVariants = cva(
  "group pointer-events-auto relative flex w-full items-center justify-between space-x-4 overflow-hidden rounded-md border p-6 pr-8 shadow-lg transition-all data-[swipe=cancel]:translate-x-0 data-[swipe=end]:translate-x-[var(--radix-toast-swipe-end-x)] data-[swipe=move]:translate-x-[var(--radix-toast-swipe-move-x)] data-[swipe=move]:transition-none data-[state=open]:animate-in data-[state=closed]:animate-out data-[swipe=end]:animate-out data-[state=closed]:fade-out-80 data-[state=closed]:slide-out-to-right-full data-[state=open]:slide-in-from-top-full data-[state=open]:sm:slide-in-from-bottom-full",
  {
    variants: {
      variant: {
        default: "border bg-background text-foreground",
        destructive:
          "destructive group border-destructive bg-destructive text-destructive-foreground",
      },
    },
    defaultVariants: {
      variant: "default",
    },
  }
);

const Toast = React.forwardRef(({ className, variant, ...props }, ref) => {
  return (
    <ToastPrimitives.Root
      ref={ref}
      className={cn(toastVariants({ variant }), className)}
      {...props}
    />
  );
});
Toast.displayName = ToastPrimitives.Root.displayName;

const ToastAction = React.forwardRef(({ className, ...props }, ref) => (
  <ToastPrimitives.Action
    ref={ref}
    className={cn(
      "inline-flex h-8 shrink-0 items-center justify-center rounded-md border bg-transparent px-3 text-sm font-medium ring-offset-background transition-colors hover:bg-secondary focus:outline-none focus:ring-2 focus:ring-ring focus:ring-offset-2 disabled:pointer-events-none disabled:opacity-50 group-[.destructive]:border-muted/40 group-[.destructive]:hover:border-destructive/30 group-[.destructive]:hover:bg-destructive group-[.destructive]:hover:text-destructive-foreground group-[.destructive]:focus:ring-destructive",
      className
    )}
    {...props}
  />
));
ToastAction.displayName = ToastPrimitives.Action.displayName;

const ToastClose = React.forwardRef(({ className, ...props }, ref) => (
  <ToastPrimitives.Close
    ref={ref}
    className={cn(
      "absolute right-2 top-2 rounded-md p-1 text-foreground/50 opacity-0 transition-opacity hover:text-foreground focus:opacity-100 focus:outline-none focus:ring-2 group-hover:opacity-100 group-[.destructive]:text-red-300 group-[.destructive]:hover:text-red-50 group-[.destructive]:focus:ring-red-400 group-[.destructive]:focus:ring-offset-red-600",
      className
    )}
    toast-close=""
    {...props}
  >
    <X className="h-4 w-4" />
  </ToastPrimitives.Close>
));
ToastClose.displayName = ToastPrimitives.Close.displayName;

const ToastTitle = React.forwardRef(({ className, ...props }, ref) => (
  <ToastPrimitives.Title
    ref={ref}
    className={cn("text-sm font-semibold", className)}
    {...props}
  />
));
ToastTitle.displayName = ToastPrimitives.Title.displayName;

const ToastDescription = React.forwardRef(({ className, ...props }, ref) => (
  <ToastPrimitives.Description
    ref={ref}
    className={cn("text-sm opacity-90", className)}
    {...props}
  />
));
ToastDescription.displayName = ToastPrimitives.Description.displayName;

export {
  ToastProvider,
  ToastViewport,
  Toast,
  ToastTitle,
  ToastDescription,
  ToastClose,
  ToastAction,
};
#toaster
import { useToast } from "@/components/ui/use-toast";
import {
  Toast,
  ToastClose,
  ToastDescription,
  ToastProvider,
  ToastTitle,
  ToastViewport,
} from "@/components/ui/toast";

export function Toaster() {
  const { toasts } = useToast();

  return (
    <ToastProvider>
      {toasts.map(function ({ id, title, description, action, ...props }) {
        return (
          <Toast key={id} {...props}>
            <div className="grid gap-1">
              {title && <ToastTitle>{title}</ToastTitle>}
              {description && (
                <ToastDescription>{description}</ToastDescription>
              )}
            </div>
            {action}
            <ToastClose />
          </Toast>
        );
      })}
      <ToastViewport />
    </ToastProvider>
  );
} 
#toggle-group
"use client";
import * as React from "react"
import * as ToggleGroupPrimitive from "@radix-ui/react-toggle-group"

import { cn } from "@/lib/utils"
import { toggleVariants } from "@/components/ui/toggle"

const ToggleGroupContext = React.createContext({
  size: "default",
  variant: "default",
})

const ToggleGroup = React.forwardRef(({ className, variant, size, children, ...props }, ref) => (
  <ToggleGroupPrimitive.Root
    ref={ref}
    className={cn("flex items-center justify-center gap-1", className)}
    {...props}>
    <ToggleGroupContext.Provider value={{ variant, size }}>
      {children}
    </ToggleGroupContext.Provider>
  </ToggleGroupPrimitive.Root>
))

ToggleGroup.displayName = ToggleGroupPrimitive.Root.displayName

const ToggleGroupItem = React.forwardRef(({ className, children, variant, size, ...props }, ref) => {
  const context = React.useContext(ToggleGroupContext)

  return (
    (<ToggleGroupPrimitive.Item
      ref={ref}
      className={cn(toggleVariants({
        variant: context.variant || variant,
        size: context.size || size,
      }), className)}
      {...props}>
      {children}
    </ToggleGroupPrimitive.Item>)
  );
})

ToggleGroupItem.displayName = ToggleGroupPrimitive.Item.displayName

export { ToggleGroup, ToggleGroupItem }
#toggle
import * as React from "react"
import * as TogglePrimitive from "@radix-ui/react-toggle"
import { cva } from "class-variance-authority";

import { cn } from "@/lib/utils"

const toggleVariants = cva(
  "inline-flex items-center justify-center gap-2 rounded-md text-sm font-medium transition-colors hover:bg-muted hover:text-muted-foreground focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50 data-[state=on]:bg-accent data-[state=on]:text-accent-foreground [&_svg]:pointer-events-none [&_svg]:size-4 [&_svg]:shrink-0",
  {
    variants: {
      variant: {
        default: "bg-transparent",
        outline:
          "border border-input bg-transparent shadow-sm hover:bg-accent hover:text-accent-foreground",
      },
      size: {
        default: "h-9 px-2 min-w-9",
        sm: "h-8 px-1.5 min-w-8",
        lg: "h-10 px-2.5 min-w-10",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
)
#tooltip
"use client"

import * as React from "react"
import * as TooltipPrimitive from "@radix-ui/react-tooltip"

import { cn } from "@/lib/utils"

const TooltipProvider = TooltipPrimitive.Provider

const Tooltip = TooltipPrimitive.Root

const TooltipTrigger = TooltipPrimitive.Trigger

const TooltipContent = React.forwardRef(({ className, sideOffset = 4, ...props }, ref) => (
  <TooltipPrimitive.Portal>
    <TooltipPrimitive.Content
      ref={ref}
      sideOffset={sideOffset}
      className={cn(
        "z-50 overflow-hidden rounded-md bg-primary px-3 py-1.5 text-xs text-primary-foreground animate-in fade-in-0 zoom-in-95 data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=closed]:zoom-out-95 data-[side=bottom]:slide-in-from-top-2 data-[side=left]:slide-in-from-right-2 data-[side=right]:slide-in-from-left-2 data-[side=top]:slide-in-from-bottom-2",
        className
      )}
      {...props} />
  </TooltipPrimitive.Portal>
))
TooltipContent.displayName = TooltipPrimitive.Content.displayName

export { Tooltip, TooltipTrigger, TooltipContent, TooltipProvider }

#use-responsive-image
import * as React from "react"
import { useSize } from "../../hooks/use-size"
import { cn } from "../../lib/utils"
import { DEFAULT_TRANSFORM_WIDTH, getImagePreviewClassName } from "./image-helpers"

export function useResponsiveImage({ parsed, fittingType, focalPoint, quality, className, onLoad, onSourceChange }, parentRef) {
  const wrapperRef = React.useRef(null)
  const imgRef = React.useRef(null)
  const size = useSize(wrapperRef)
  const [loaded, setLoaded] = React.useState(false)

  React.useImperativeHandle(parentRef, () => imgRef.current)
  React.useEffect(() => setLoaded(false), [parsed.baseUrl])
  React.useEffect(() => {
    const wrapper = wrapperRef.current
    const replace = (event) => onSourceChange(
      event.detail.src, getImagePreviewClassName(className, wrapper.className, cn("inline-block relative", className))
    )
    wrapper.addEventListener("base44:image-replace", replace)
    return () => wrapper.removeEventListener("base44:image-replace", replace)
  }, [className, onSourceChange])

  const crop = fittingType !== "fit"
  // Wait for useSize's pre-paint measurement before requesting a transform.
  const options = size && {
    width: size.width || DEFAULT_TRANSFORM_WIDTH,
    height: size.height || undefined,
    crop,
    focalPoint: crop ? focalPoint : undefined,
    quality,
  }

  return {
    wrapperRef, imgRef, loaded, options,
    handleLoad: (event) => {
      setLoaded(true)
      onLoad?.(event)
    },
  }
}

#use-toast
// Inspired by react-hot-toast library
import { useState, useEffect } from "react";

const TOAST_LIMIT = 20;
const TOAST_REMOVE_DELAY = 1000000;

const actionTypes = {
  ADD_TOAST: "ADD_TOAST",
  UPDATE_TOAST: "UPDATE_TOAST",
  DISMISS_TOAST: "DISMISS_TOAST",
  REMOVE_TOAST: "REMOVE_TOAST",
};

let count = 0;

function genId() {
  count = (count + 1) % Number.MAX_VALUE;
  return count.toString();
}

const toastTimeouts = new Map();

const addToRemoveQueue = (toastId) => {
  if (toastTimeouts.has(toastId)) {
    return;
  }

  const timeout = setTimeout(() => {
    toastTimeouts.delete(toastId);
    dispatch({
      type: actionTypes.REMOVE_TOAST,
      toastId,
    });
  }, TOAST_REMOVE_DELAY);

  toastTimeouts.set(toastId, timeout);
};

const _clearFromRemoveQueue = (toastId) => {
  const timeout = toastTimeouts.get(toastId);
  if (timeout) {
    clearTimeout(timeout);
    toastTimeouts.delete(toastId);
  }
};

export const reducer = (state, action) => {
  switch (action.type) {
    case actionTypes.ADD_TOAST:
      return {
        ...state,
        toasts: [action.toast, ...state.toasts].slice(0, TOAST_LIMIT),
      };

    case actionTypes.UPDATE_TOAST:
      return {
        ...state,
        toasts: state.toasts.map((t) =>
          t.id === action.toast.id ? { ...t, ...action.toast } : t
        ),
      };

    case actionTypes.DISMISS_TOAST: {
      const { toastId } = action;

      // ! Side effects ! - This could be extracted into a dismissToast() action,
      // but I'll keep it here for simplicity
      if (toastId) {
        addToRemoveQueue(toastId);
      } else {
        state.toasts.forEach((toast) => {
          addToRemoveQueue(toast.id);
        });
      }

      return {
        ...state,
        toasts: state.toasts.map((t) =>
          t.id === toastId || toastId === undefined
            ? {
                ...t,
                open: false,
              }
            : t
        ),
      };
    }
    case actionTypes.REMOVE_TOAST:
      if (action.toastId === undefined) {
        return {
          ...state,
          toasts: [],
        };
      }
      return {
        ...state,
        toasts: state.toasts.filter((t) => t.id !== action.toastId),
      };
  }
};

const listeners = [];

let memoryState = { toasts: [] };

function dispatch(action) {
  memoryState = reducer(memoryState, action);
  listeners.forEach((listener) => {
    listener(memoryState);
  });
}

function toast({ ...props }) {
  const id = genId();

  const update = (props) =>
    dispatch({
      type: actionTypes.UPDATE_TOAST,
      toast: { ...props, id },
    });

  const dismiss = () =>
    dispatch({ type: actionTypes.DISMISS_TOAST, toastId: id });

  dispatch({
    type: actionTypes.ADD_TOAST,
    toast: {
      ...props,
      id,
      open: true,
      onOpenChange: (open) => {
        if (!open) dismiss();
      },
    },
  });

  return {
    id,
    dismiss,
    update,
  };
}

function useToast() {
  const [state, setState] = useState(memoryState);

  useEffect(() => {
    listeners.push(setState);
    return () => {
      const index = listeners.indexOf(setState);
      if (index > -1) {
        listeners.splice(index, 1);
      }
    };
  }, [state]);

  return {
    ...state,
    toast,
    dismiss: (toastId) => dispatch({ type: actionTypes.DISMISS_TOAST, toastId }),
  };
}

export { useToast, toast }; 
#AppLayout
import { Outlet } from 'react-router-dom';
import SiteHeader from '@/components/SiteHeader';
import SiteFooter from '@/components/SiteFooter';

export default function AppLayout() {
  return (
    <div className="flex min-h-screen flex-col bg-background">
      <SiteHeader />
      <main className="flex-1">
        <Outlet />
      </main>
      <SiteFooter />
    </div>
  );
}
#AuthLayout
import React from "react";

export default function AuthLayout({ icon: Icon, title, subtitle, footer, children }) {
  return (
    <div className="min-h-screen flex items-center justify-center bg-background px-4">
      <div className="w-full max-w-md">
        <div className="text-center mb-10">
          <div className="inline-flex items-center justify-center w-14 h-14 rounded-2xl bg-primary mb-4">
            <Icon className="w-7 h-7 text-primary-foreground" aria-hidden="true" />
          </div>
          <h1 className="text-3xl font-bold tracking-tight text-foreground">{title}</h1>
          {subtitle && <p className="text-muted-foreground mt-2">{subtitle}</p>}
        </div>
        <div className="bg-card rounded-2xl shadow-sm border border-border p-8">
          {children}
        </div>
        {footer && (
          <p className="text-center text-sm text-muted-foreground mt-6">{footer}</p>
        )}
      </div>
    </div>
  );
}
#CrowdLevelBadge
import { levelMeta } from '@/lib/crowd';
import { useLanguage } from '@/lib/LanguageContext';

export default function CrowdLevelBadge({ level, className = '' }) {
  const { t } = useLanguage();
  const meta = levelMeta(level);

  return (
    <span className={`inline-flex items-center gap-1.5 rounded-full border px-2.5 py-0.5 text-xs font-medium ${meta.chip} ${className}`}>
      <span className={`h-1.5 w-1.5 rounded-full ${meta.dot}`} />
      {t(meta.labelKey)}
    </span>
  );
}
#CrowdRepoerForm
import { useEffect, useState } from 'react';
import { Check, Loader2, Search } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import { useDataset } from '@/lib/DatasetContext';
import { CROWD_LEVELS, levelMeta } from '@/lib/crowd';
import { formatDateTime } from '@/lib/transit';
import StopPicker from '@/components/StopPicker';
import ImportNotice from '@/components/ImportNotice';

const RECENCY = [
  { key: 'now', minutes: 0, labelKey: 'report.now' },
  { key: 'min15', minutes: 10, labelKey: 'report.min15' },
  { key: 'min60', minutes: 45, labelKey: 'report.min60' },
];

export default function CrowdReportForm() {
  const { t } = useLanguage();
  const { status, submitReport, searchRoutes } = useDataset();

  const [routeNumber, setRouteNumber] = useState('');
  const [routeVerified, setRouteVerified] = useState(false);
  const [routeOptions, setRouteOptions] = useState([]);
  const [routeSearching, setRouteSearching] = useState(false);
  const [tripRef, setTripRef] = useState('');
  const [stop, setStop] = useState(null);
  const [level, setLevel] = useState(null);
  const [recency, setRecency] = useState('now');
  const [saving, setSaving] = useState(false);
  const [error, setError] = useState(null);
  const [saved, setSaved] = useState(null);

  useEffect(() => {
    const needle = routeNumber.trim();
    if (!routeVerified && needle.length >= 2 && (status === 'ready' || status === 'sample')) {
      let cancelled = false;
      setRouteSearching(true);
      const timer = setTimeout(async () => {
        const rows = await searchRoutes(needle, 8);
        if (!cancelled) {
          setRouteOptions(rows);
          setRouteSearching(false);
        }
      }, 250);
      return () => {
        cancelled = true;
        clearTimeout(timer);
      };
    }
    setRouteOptions([]);
    return undefined;
  }, [routeNumber, routeVerified, searchRoutes, status]);

  const reset = () => {
    setRouteNumber('');
    setRouteVerified(false);
    setTripRef('');
    setStop(null);
    setLevel(null);
    setRecency('now');
    setError(null);
    setSaved(null);
  };

  const submit = async (event) => {
    event.preventDefault();
    if (!stop) {
      setError(t('report.errStop'));
      return;
    }
    if (!level) {
      setError(t('report.errLevel'));
      return;
    }
    const offset = RECENCY.find((item) => item.key === recency)?.minutes ?? 0;
    const reportedAt = new Date(Date.now() - offset * 60000).toISOString();

    setSaving(true);
    setError(null);
    const result = await submitReport({
      routeNumber: routeNumber.trim(),
      routeVerified,
      tripRef: tripRef.trim(),
      stopName: stop.name,
      level,
      reportedAt,
    });
    setSaving(false);

    if (result?.ok) {
      setSaved({ reportedAt });
      setRouteNumber('');
      setRouteVerified(false);
      setTripRef('');
      setStop(null);
      setLevel(null);
      setRecency('now');
      return;
    }
    if (result?.reason === 'duplicate') {
      setError(`${t('report.errDuplicate')} ${result.lastAt ? t('report.lastAt', { time: formatDateTime(result.lastAt) }) : ''}`.trim());
      return;
    }
    if (result?.reason === 'need_dataset') {
      setError(t('report.needDataset'));
      return;
    }
    setError(t('report.errSave'));
  };

  return (
    <div className="rounded-2xl border border-border/70 bg-card p-4 shadow-sm sm:p-6">
      {status !== 'ready' && <ImportNotice />}

      {saved ? (
        <div className="space-y-3 rounded-xl border border-crowd-low/35 bg-crowd-low/10 p-4">
          <p className="flex items-center gap-2 font-display text-sm font-semibold text-crowd-low">
            <Check className="h-4 w-4" /> {t('report.saved')}
          </p>
          <p className="text-sm text-muted-foreground">{t('report.savedHelp')}</p>
          <p className="text-xs text-muted-foreground">{formatDateTime(saved.reportedAt)}</p>
          <button
            type="button"
            onClick={reset}
            className="rounded-xl border border-border bg-background px-3 py-1.5 text-sm font-medium"
          >
            {t('report.another')}
          </button>
        </div>
      ) : (
        <form onSubmit={submit} className="space-y-5">
          <div className="grid gap-4 sm:grid-cols-2">
            <div className="space-y-1.5">
              <label htmlFor="report-route" className="text-xs font-medium text-muted-foreground">{t('report.route')}</label>
              <div className="relative">
                <input
                  id="report-route"
                  value={routeNumber}
                  onChange={(event) => {
                    setRouteNumber(event.target.value);
                    setRouteVerified(false);
                  }}
                  placeholder={t('report.routePlaceholder')}
                  className="flex h-9 w-full rounded-xl border border-input bg-transparent px-3 py-2 pr-9 text-sm shadow-sm focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring"
                />
                {routeSearching && <Loader2 className="absolute right-3 top-1/2 h-3.5 w-3.5 -translate-y-1/2 animate-spin text-muted-foreground" />}
                {routeVerified && <Check className="absolute right-3 top-1/2 h-3.5 w-3.5 -translate-y-1/2 text-crowd-low" />}
              </div>
              {routeOptions.length > 0 && !routeVerified && (
                <ul className="max-h-40 overflow-y-auto rounded-xl border border-border bg-popover p-1 shadow-sm">
                  {routeOptions.map((route) => (
                    <li key={route.id ?? route.route_number}>
                      <button
                        type="button"
                        onClick={() => {
                          setRouteNumber(route.route_number);
                          setRouteVerified(true);
                          setRouteOptions([]);
                        }}
                        className="flex w-full flex-col items-start rounded-lg px-2 py-1.5 text-left text-xs hover:bg-accent"
                      >
                        <span className="font-medium">{route.route_number}</span>
                        <span className="text-muted-foreground">{route.full_name}</span>
                      </button>
                    </li>
                  ))}
                </ul>
              )}
              <p className="text-[11px] text-muted-foreground">{t('report.routeHint')}</p>
            </div>

            <div className="space-y-1.5">
              <label htmlFor="report-trip" className="text-xs font-medium text-muted-foreground">{t('report.trip')}</label>
              <input
                id="report-trip"
                value={tripRef}
                onChange={(event) => setTripRef(event.target.value)}
                placeholder={t('report.tripPlaceholder')}
                className="flex h-9 w-full rounded-xl border border-input bg-transparent px-3 py-2 text-sm shadow-sm focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring"
              />
            </div>
          </div>

          <StopPicker
            id="report-stop"
            label={t('report.stop')}
            placeholder={t('search.stopPlaceholder')}
            value={stop}
            onSelect={setStop}
          />
          <p className="text-[11px] text-muted-foreground">{t('report.stopHint')}</p>

          <fieldset className="space-y-2">
            <legend className="text-xs font-medium text-muted-foreground">{t('report.level')}</legend>
            <div className="flex flex-wrap gap-2">
              {CROWD_LEVELS.map((value) => {
                const meta = levelMeta(value);
                const active = level === value;
                return (
                  <button
                    key={value}
                    type="button"
                    onClick={() => setLevel(value)}
                    aria-pressed={active}
                    className={`inline-flex items-center gap-1.5 rounded-full border px-3 py-1.5 text-sm font-medium transition-colors ${meta.chip} ${active ? `ring-2 ring-offset-1 ring-offset-background ${meta.border}` : 'opacity-80 hover:opacity-100'}`}
                  >
                    <span className={`h-1.5 w-1.5 rounded-full ${meta.dot}`} />
                    {t(meta.labelKey)}
                  </button>
                );
              })}
            </div>
          </fieldset>

          <fieldset className="space-y-2">
            <legend className="text-xs font-medium text-muted-foreground">{t('report.recency')}</legend>
            <div className="flex flex-wrap gap-4">
              {RECENCY.map((item) => (
                <label key={item.key} className="flex items-center gap-2 text-sm">
                  <input
                    type="radio"
                    name="recency"
                    value={item.key}
                    checked={recency === item.key}
                    onChange={() => setRecency(item.key)}
                    className="h-4 w-4 accent-primary"
                  />
                  {t(item.labelKey)}
                </label>
              ))}
            </div>
          </fieldset>

          {error && <p className="text-sm text-destructive">{error}</p>}

          <div className="flex flex-wrap items-center gap-3">
            <button
              type="submit"
              disabled={saving}
              className="inline-flex items-center gap-2 rounded-xl bg-primary px-4 py-2 text-sm font-medium text-primary-foreground disabled:opacity-60"
            >
              {saving && <Loader2 className="h-3.5 w-3.5 animate-spin" />}
              {saving ? t('report.saving') : t('report.submit')}
            </button>
            <span className="text-xs text-muted-foreground">{t('report.privacy')}</span>
          </div>
        </form>
      )}
    </div>
  );
}
#CrowdReportsFeed
import { useEffect, useState } from 'react';
import { Link } from 'react-router-dom';
import { Loader2, MessageSquarePlus } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import { useDataset } from '@/lib/DatasetContext';
import { ageLabel, isStale, RECENT_HOURS } from '@/lib/crowd';
import { formatTime } from '@/lib/transit';
import CrowdLevelBadge from '@/components/CrowdLevelBadge';

function ReportRow({ report }) {
  const { t } = useLanguage();
  const stale = isStale(report.reported_at);
  return (
    <li className="flex flex-wrap items-center gap-x-3 gap-y-1.5 rounded-xl border border-border/70 bg-background p-3 text-sm">
      <span className="rounded-md border border-border bg-muted/40 px-1.5 py-0.5 font-display text-xs font-semibold">
        {report.route_number || t('feed.noRoute')}
      </span>
      <span className="min-w-0 flex-1 truncate text-muted-foreground">{report.stop_name}</span>
      <CrowdLevelBadge level={report.crowd_level} />
      <span className="text-xs text-muted-foreground">
        {ageLabel(report.reported_at, t)} · {formatTime(report.reported_at)}
      </span>
      {stale && (
        <span className="rounded-full border border-crowd-moderate/30 bg-crowd-moderate/10 px-2 py-0.5 text-[11px] font-medium text-crowd-moderate">
          {t('feed.stale')}
        </span>
      )}
    </li>
  );
}

export default function CrowdReportsFeed() {
  const { t } = useLanguage();
  const { reports, reportsLoading, totalReports, hasMoreReports, loadMoreReports } = useDataset();
  const [, setTick] = useState(0);

  useEffect(() => {
    const id = setInterval(() => setTick((value) => value + 1), 60000);
    return () => clearInterval(id);
  }, []);

  const recent = reports.filter((report) => !isStale(report.reported_at));
  const older = reports.filter((report) => isStale(report.reported_at));

  return (
    <section className="space-y-4">
      <div className="flex flex-wrap items-end justify-between gap-3">
        <div>
          <p className="text-[11px] font-medium uppercase tracking-[0.16em] text-muted-foreground">{t('feed.eyebrow')}</p>
          <h2 className="mt-1 font-display text-2xl font-semibold tracking-tight">{t('feed.title')}</h2>
          <p className="mt-1 max-w-2xl text-sm text-muted-foreground">{t('feed.description')}</p>
        </div>
        <Link
          to="/report"
          className="inline-flex items-center gap-2 rounded-xl bg-primary px-4 py-2 text-sm font-medium text-primary-foreground"
        >
          <MessageSquarePlus className="h-4 w-4" />
          {t('feed.cta')}
        </Link>
      </div>

      {reportsLoading && (
        <p className="flex items-center gap-2 text-sm text-muted-foreground">
          <Loader2 className="h-3.5 w-3.5 animate-spin" /> {t('feed.loading')}
        </p>
      )}

      {!reportsLoading && reports.length === 0 && (
        <div className="rounded-2xl border border-dashed border-border/80 bg-card/50 p-6 text-center">
          <p className="font-display text-sm font-semibold">{t('feed.empty')}</p>
          <p className="mt-1 text-sm text-muted-foreground">{t('feed.first')}</p>
        </div>
      )}

      {recent.length > 0 && (
        <div className="space-y-2">
          <h3 className="text-xs font-semibold uppercase tracking-wide text-muted-foreground">{t('feed.recent')}</h3>
          <ul className="space-y-2">
            {recent.map((report) => (
              <ReportRow key={report.id} report={report} />
            ))}
          </ul>
        </div>
      )}

      {older.length > 0 && (
        <div className="space-y-2">
          <h3 className="text-xs font-semibold uppercase tracking-wide text-muted-foreground">
            {t('feed.older')} · <span className="font-normal normal-case">{t('feed.staleHint', { hours: RECENT_HOURS })}</span>
          </h3>
          <ul className="space-y-2">
            {older.map((report) => (
              <ReportRow key={report.id} report={report} />
            ))}
          </ul>
        </div>
      )}

      {reports.length > 0 && (
        <div className="flex flex-wrap items-center gap-3 text-xs text-muted-foreground">
          <span>{t('feed.showing', { count: reports.length, total: totalReports })}</span>
          {hasMoreReports && (
            <button type="button" onClick={loadMoreReports} className="underline underline-offset-4 hover:text-foreground">
              {t('feed.older')}
            </button>
          )}
          <span className="text-muted-foreground">{t('feed.live')}</span>
        </div>
      )}
    </section>
  );
}
#DataSourcesDialog
import { Info, ExternalLink, RefreshCw, Download, Loader2, ShieldAlert } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import { useDataset } from '@/lib/DatasetContext';
import { formatSourceDate, formatDateTime } from '@/lib/transit';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from '@/components/ui/dialog';

export default function DataSourcesDialog() {
  const { t } = useLanguage();
  const { dataset, status, progress, importDataset, refreshSource } = useDataset();
  const importing = status === 'importing';
  const checked = dataset.checkedAt ? formatDateTime(dataset.checkedAt) : t('status.checking');
  const count = (value) => Number(value ?? 0).toLocaleString('en-IN');

  return (
    <Dialog>
      <DialogTrigger asChild>
        <button
          type="button"
          className="inline-flex items-center gap-1 rounded-full border border-border px-2.5 py-0.5 font-medium text-foreground transition-colors hover:bg-accent"
        >
          <Info className="h-3 w-3" />
          {t('status.sources')}
        </button>
      </DialogTrigger>
      <DialogContent className="max-h-[85vh] overflow-y-auto sm:max-w-2xl">
        <DialogHeader>
          <DialogTitle>{t('status.sources')}</DialogTitle>
          <DialogDescription>{t('about.snapshot', { date: formatSourceDate(dataset.commitDate) || '—', checked })}</DialogDescription>
        </DialogHeader>

        <div className="space-y-5 text-sm">
          <section className="space-y-2">
            <h3 className="font-display text-sm font-semibold">{t('about.realTitle')}</h3>
            <p className="text-muted-foreground">{t('about.realBody')}</p>
            <div className="flex flex-wrap items-center gap-2 text-xs">
              <Badge variant="secondary">{dataset.licence}</Badge>
              <Badge variant="secondary">{count(dataset.routeCount)} {t('status.routes')}</Badge>
              <Badge variant="secondary">{count(dataset.stopCount)} {t('status.stops')}</Badge>
            </div>
            <a
              href={dataset.sourceUrl}
              target="_blank"
              rel="noreferrer"
              className="inline-flex items-center gap-1.5 text-xs font-medium text-primary underline-offset-4 hover:underline"
            >
              {dataset.sourceName} <ExternalLink className="h-3 w-3" />
            </a>
            <p className="break-all text-xs text-muted-foreground">{dataset.sourceCsvUrl}</p>
          </section>

          <section className="space-y-2">
            <h3 className="flex items-center gap-1.5 font-display text-sm font-semibold">
              <ShieldAlert className="h-4 w-4 text-crowd-moderate" /> {t('about.caveatTitle')}
            </h3>
            <p className="text-muted-foreground">{t('about.caveatBody')}</p>
            <p className="text-muted-foreground">{t('about.coverageBody')}</p>
          </section>

          <section className="space-y-2">
            <h3 className="font-display text-sm font-semibold">{t('about.noTitle')}</h3>
            <p className="text-muted-foreground">{t('about.noBody')}</p>
          </section>

          <section className="space-y-2 rounded-xl border border-border/70 bg-muted/30 p-3">
            <h3 className="font-display text-sm font-semibold">{t('status.importCta')}</h3>
            <p className="text-xs text-muted-foreground">{t('status.needImport')}</p>
            {progress && importing && (
              <p className="text-xs font-medium text-foreground">
                {t('status.importProgress', {
                  done: count(progress.done),
                  total: count(progress.total),
                  unit: t(progress.unit === 'stops' ? 'status.stops' : 'status.routes'),
                })}
              </p>
            )}
            <div className="flex flex-wrap items-center gap-2">
              <Button size="sm" onClick={importDataset} disabled={importing}>
                {importing ? <Loader2 className="mr-2 h-3.5 w-3.5 animate-spin" /> : <Download className="mr-2 h-3.5 w-3.5" />}
                {importing ? t('status.importing') : t('status.importCta')}
              </Button>
              <Button size="sm" variant="outline" onClick={refreshSource} disabled={importing}>
                <RefreshCw className="mr-2 h-3.5 w-3.5" />
                {t('status.refresh')}
              </Button>
            </div>
            {dataset.importedAt && (
              <p className="text-xs text-muted-foreground">{t('status.importDone')} · {formatDateTime(dataset.importedAt)}</p>
            )}
            {dataset.lastError && <p className="text-xs text-destructive">{dataset.lastError}</p>}
          </section>
        </div>
      </DialogContent>
    </Dialog>
  );
}
#GoogleIcon
import React from "react";

export default function GoogleIcon({ className = "w-5 h-5" }) {
  return (
    <svg className={className} viewBox="0 0 24 24" aria-hidden="true">
      <path d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92a5.06 5.06 0 01-2.2 3.32v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.1z" fill="#4285F4" />
      <path d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z" fill="#34A853" />
      <path d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l2.85-2.22.81-.62z" fill="#FBBC05" />
      <path d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z" fill="#EA4335" />
    </svg>
  );
}

#ImportNotice
mport { AlertTriangle, Download, Loader2 } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import { useDataset } from '@/lib/DatasetContext';

export default function ImportNotice({ className = '' }) {
  const { t } = useLanguage();
  const { status, progress, importDataset } = useDataset();
  const busy = status === 'importing';

  return (
    <div className={`mb-4 flex flex-wrap items-center gap-3 rounded-xl border border-crowd-moderate/30 bg-crowd-moderate/10 p-3 text-xs ${className}`}>
      <AlertTriangle className="h-4 w-4 shrink-0 text-crowd-moderate" />
      <span className="min-w-0 flex-1 text-foreground">
        {status === 'sample' ? t('status.sample') : t('status.needImport')}
        {busy && progress && (
          <span className="ml-1 font-medium">
            {t('status.importProgress', {
              done: Number(progress.done ?? 0).toLocaleString('en-IN'),
              total: Number(progress.total ?? 0).toLocaleString('en-IN'),
              unit: t(progress.unit === 'stops' ? 'status.stops' : 'status.routes'),
            })}
          </span>
        )}
      </span>
      <button
        type="button"
        onClick={importDataset}
        disabled={busy}
        className="inline-flex items-center gap-1.5 rounded-lg bg-primary px-3 py-1.5 font-medium text-primary-foreground disabled:opacity-60"
      >
        {busy ? <Loader2 className="h-3.5 w-3.5 animate-spin" /> : <Download className="h-3.5 w-3.5" />}
        {busy ? t('status.importing') : t('status.importCta')}
      </button>
    </div>
  );
}
#JourneyPlanner
import { useState } from 'react';
import { ArrowDown } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import { useDataset } from '@/lib/DatasetContext';
import StopPicker from '@/components/StopPicker';
import ImportNotice from '@/components/ImportNotice';
import RouteOption from '@/components/RouteOption';

export default function JourneyPlanner() {
  const { t } = useLanguage();
  const { planJourney, status } = useDataset();
  const [from, setFrom] = useState(null);
  const [to, setTo] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [result, setResult] = useState(null);

  const run = async () => {
    if (!from || !to) {
      setError(t('plan.needBoth'));
      return;
    }
    setLoading(true);
    setError(null);
    const data = await planJourney(from.name, to.name);
    setResult(data);
    setLoading(false);
  };

  return (
    <div className="rounded-2xl border border-border/70 bg-card p-4 shadow-sm sm:p-5">
      {status !== 'ready' && <ImportNotice />}

      <div className="grid gap-3 sm:grid-cols-[1fr_auto_1fr] sm:items-end">
        <StopPicker
          id="journey-from"
          label={t('plan.from')}
          placeholder={t('plan.chooseFrom')}
          value={from}
          onSelect={setFrom}
        />
        <button
          type="button"
          onClick={() => {
            setFrom(to);
            setTo(from);
            setResult(null);
          }}
          aria-label={t('plan.swap')}
          className="hidden h-9 w-9 items-center justify-center rounded-full border border-border text-muted-foreground hover:bg-accent sm:flex"
        >
          <ArrowDown className="h-4 w-4" />
        </button>
        <StopPicker
          id="journey-to"
          label={t('plan.to')}
          placeholder={t('plan.chooseTo')}
          value={to}
          onSelect={setTo}
        />
      </div>

      <div className="mt-4 flex flex-wrap items-center gap-3">
        <button
          type="button"
          onClick={run}
          disabled={loading || !from || !to}
          className="inline-flex items-center rounded-xl bg-primary px-4 py-2 text-sm font-medium text-primary-foreground disabled:opacity-50"
        >
          {loading ? t('plan.searching') : t('plan.search')}
        </button>
        <span className="text-xs text-muted-foreground">{t('search.matched')}</span>
      </div>

      {error && <p className="mt-3 text-sm text-destructive">{error}</p>}

      {result && (
        <div className="mt-5 space-y-3">
          <div>
            <h3 className="font-display text-sm font-semibold">{t('plan.results')}</h3>
            <p className="text-xs text-muted-foreground">
              {result.options?.length ? t('plan.count', { count: result.options.length }) : t('plan.none')}
            </p>
          </div>

          {result.options?.map((option) => (
            <RouteOption key={option.route.id ?? option.route.route_number} option={option} />
          ))}

          {result.options?.length > 0 && (
            <p className="rounded-xl border border-border/70 bg-muted/30 p-3 text-xs text-muted-foreground">{t('plan.note')}</p>
          )}
        </div>
      )}
    </div>
  );
}
#LanguageSelector
import { Globe, Check } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuLabel,
  DropdownMenuSeparator,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu';

export default function LanguageSelector() {
  const { code, setCode, t, languages } = useLanguage();
  const current = languages.find((language) => language.code === code) ?? languages[0];

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <button
          type="button"
          aria-label={t('nav.changeLanguage')}
          className="inline-flex items-center gap-1.5 rounded-full border border-border px-3 py-1.5 text-xs font-medium text-foreground transition-colors hover:bg-accent"
        >
          <Globe className="h-3.5 w-3.5" />
          {current.native}
        </button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end" className="w-44">
        <DropdownMenuLabel>{t('nav.changeLanguage')}</DropdownMenuLabel>
        <DropdownMenuSeparator />
        {languages.map((language) => (
          <DropdownMenuItem
            key={language.code}
            onClick={() => setCode(language.code)}
            className="flex items-center justify-between gap-3"
          >
            <span>{language.native}</span>
            {language.code === code && <Check className="h-3.5 w-3.5 text-primary" />}
          </DropdownMenuItem>
        ))}
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
#LiveDataNotice
import { Database, EyeOff, Users } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import { RECENT_HOURS } from '@/lib/crowd';

export default function LiveDataNotice() {
  const { t } = useLanguage();

  const items = [
    { key: 'real', Icon: Database, tone: 'text-crowd-low', title: t('about.realTitle'), body: t('about.realBody') },
    { key: 'reports', Icon: Users, tone: 'text-primary', title: t('about.reportsTitle'), body: t('about.reportsBody', { hours: RECENT_HOURS }) },
    { key: 'none', Icon: EyeOff, tone: 'text-crowd-moderate', title: t('about.noTitle'), body: t('about.noBody') },
  ];

  return (
    <section className="grid gap-4 md:grid-cols-3">
      {items.map(({ key, Icon, tone, title, body }) => (
        <article key={key} className="rounded-2xl border border-border/70 bg-card p-4">
          <Icon className={`h-4 w-4 ${tone}`} />
          <h3 className="mt-2 font-display text-sm font-semibold">{title}</h3>
          <p className="mt-1 text-sm leading-relaxed text-muted-foreground">{body}</p>
        </article>
      ))}
    </section>
  );
}
#ProtectedRoute
import { useEffect } from 'react';
import { Outlet } from 'react-router-dom';
import { useAuth } from '@/lib/AuthContext';
import UserNotRegisteredError from '@/components/UserNotRegisteredError';

const DefaultFallback = () => (
  <div className="fixed inset-0 flex items-center justify-center">
    <div className="w-8 h-8 border-4 border-slate-200 border-t-slate-800 rounded-full animate-spin"></div>
  </div>
);

export default function ProtectedRoute({ fallback = <DefaultFallback />, unauthenticatedElement }) {
  const { isAuthenticated, isLoadingAuth, authChecked, authError, checkUserAuth } = useAuth();

  useEffect(() => {
    if (!authChecked && !isLoadingAuth) {
      checkUserAuth();
    }
  }, [authChecked, isLoadingAuth, checkUserAuth]);

  if (isLoadingAuth || !authChecked) {
    return fallback;
  }

  if (authError) {
    if (authError.type === 'user_not_registered') {
      return <UserNotRegisteredError />;
    }
    return unauthenticatedElement;
  }

  if (!isAuthenticated) {
    return unauthenticatedElement;
  }

  return <Outlet />;
}

#RouteOption
import { useLanguage } from '@/lib/LanguageContext';
import { formatClock } from '@/lib/transit';

export default function RouteOption({ option }) {
  const { t } = useLanguage();
  const route = option.route;
  const departures = option.departures ?? route.trip_list ?? [];
  const first = formatClock(departures[0] || route.first_departure);
  const last = formatClock(departures[departures.length - 1] || route.last_departure);
  const nextUp = departures.slice(0, 6).map(formatClock);

  return (
    <article className="rounded-xl border border-border/70 bg-background p-3">
      <header className="flex flex-wrap items-center gap-2">
        <span className="rounded-lg bg-primary px-2 py-0.5 font-display text-sm font-semibold text-primary-foreground">
          {route.route_number}
        </span>
        <span className="text-sm text-muted-foreground">{route.full_name}</span>
      </header>

      <div className="mt-3 grid gap-2 text-xs sm:grid-cols-3">
        <div>
          <p className="text-muted-foreground">{t('plan.from')}</p>
          <p className="font-medium text-foreground">{t('plan.boardAt', { stop: option.board_name })}</p>
        </div>
        <div>
          <p className="text-muted-foreground">{t('plan.to')}</p>
          <p className="font-medium text-foreground">{t('plan.alightAt', { stop: option.alight_name })}</p>
        </div>
        <div>
          <p className="text-muted-foreground">{t('search.onRoute')}</p>
          <p className="font-medium text-foreground">{t('plan.stopsBetween', { count: option.stopsBetween })}</p>
        </div>
      </div>

      <div className="mt-3 border-t border-border/60 pt-2 text-xs">
        <p className="text-muted-foreground">{t('plan.departures')}</p>
        <p className="font-medium text-foreground">{t('plan.firstLast', { first: first || '—', last: last || '—' })}</p>
        {nextUp.length > 0 && <p className="mt-1 text-muted-foreground">{nextUp.join(' · ')}</p>}
      </div>
    </article>
  );
}
#RouteSearchPanel
import { useEffect, useState } from 'react';
import { Loader2, Search } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import { useDataset } from '@/lib/DatasetContext';
import { formatClock } from '@/lib/transit';

export default function RouteSearchPanel() {
  const { t } = useLanguage();
  const { searchRoutes, status } = useDataset();
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [busy, setBusy] = useState(false);
  const [searched, setSearched] = useState(false);

  useEffect(() => {
    const needle = query.trim();
    if (needle.length < 2) {
      setResults([]);
      setSearched(false);
      return undefined;
    }
    let cancelled = false;
    setBusy(true);
    const timer = setTimeout(async () => {
      const rows = await searchRoutes(needle, 20);
      if (!cancelled) {
        setResults(rows);
        setBusy(false);
        setSearched(true);
      }
    }, 250);
    return () => {
      cancelled = true;
      clearTimeout(timer);
    };
  }, [query, searchRoutes]);

  return (
    <div className="space-y-3">
      <div>
        <h3 className="font-display text-sm font-semibold">{t('search.routesTitle')}</h3>
        <p className="text-xs text-muted-foreground">{t('search.routesHint')}</p>
      </div>

      <div className="relative">
        <Search className="pointer-events-none absolute left-3 top-1/2 h-3.5 w-3.5 -translate-y-1/2 text-muted-foreground" />
        <input
          value={query}
          onChange={(event) => setQuery(event.target.value)}
          disabled={status !== 'ready' && status !== 'sample'}
          placeholder={t('search.routePlaceholder')}
          className="flex h-9 w-full rounded-xl border border-input bg-transparent py-2 pl-9 pr-9 text-sm shadow-sm focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:cursor-not-allowed disabled:opacity-50"
        />
        {busy && <Loader2 className="absolute right-3 top-1/2 h-3.5 w-3.5 -translate-y-1/2 animate-spin text-muted-foreground" />}
      </div>

      {query.trim().length < 2 && <p className="text-xs text-muted-foreground">{t('search.typeMore')}</p>}
      {searched && results.length === 0 && <p className="text-xs text-muted-foreground">{t('search.noRoutes')}</p>}

      <div className="space-y-2">
        {results.map((route) => (
          <article key={route.id ?? route.route_number} className="rounded-xl border border-border/70 bg-background p-3">
            <header className="flex flex-wrap items-center gap-2">
              <span className="rounded-lg bg-primary px-2 py-0.5 font-display text-sm font-semibold text-primary-foreground">
                {route.route_number}
              </span>
              <span className="text-sm text-muted-foreground">{route.full_name}</span>
            </header>
            <p className="mt-2 text-xs text-muted-foreground">
              {t('search.stopsCount', { count: route.stop_count ?? 0 })} · {t('search.trips', { count: route.trip_count ?? 0 })}
              {route.first_departure ? ` · ${t('plan.firstLast', { first: formatClock(route.first_departure), last: formatClock(route.last_departure) })}` : ''}
            </p>
            {(route.stop_list || []).length > 0 && (
              <details className="mt-2 text-xs">
                <summary className="cursor-pointer text-muted-foreground">{t('search.onRoute')}</summary>
                <ol className="mt-2 list-decimal space-y-0.5 pl-4 text-muted-foreground">
                  {route.stop_list.map((stop) => (
                    <li key={stop}>{stop}</li>
                  ))}
                </ol>
              </details>
            )}
          </article>
        ))}
      </div>
    </div>
  );
}
#ScrollTo Top
import { useEffect } from "react";
import { useLocation, useNavigationType } from "react-router-dom";

const getHashId = (hash) => {
  const rawId = hash.slice(1);

  try {
    return decodeURIComponent(rawId);
  } catch {
    return rawId;
  }
};

export default function ScrollToTop() {
  const { pathname, hash } = useLocation();
  const navigationType = useNavigationType();

  useEffect(() => {
    if (navigationType === "POP") return;

    if (hash) {
      const id = getHashId(hash);
      const timer = window.setTimeout(() => {
        document.getElementById(id)?.scrollIntoView({ behavior: "smooth" });
      }, 50);
      return () => window.clearTimeout(timer);
    }

    window.scrollTo({ top: 0, left: 0, behavior: "instant" });
  }, [pathname, hash, navigationType]);

  return null;
}
#SiteFooter
import { Link } from 'react-router-dom';
import { useLanguage } from '@/lib/LanguageContext';

export default function SiteFooter() {
  const { t } = useLanguage();

  return (
    <footer className="border-t border-border/70 bg-card/40">
      <div className="mx-auto grid max-w-6xl gap-6 px-4 py-10 sm:px-6 lg:grid-cols-[1.4fr_1fr] lg:px-8">
        <div className="space-y-2">
          <p className="font-display text-base font-semibold tracking-tight">SAFAR</p>
          <p className="max-w-xl text-sm text-muted-foreground">{t('footer.blurb')}</p>
          <p className="text-sm text-muted-foreground">{t('footer.how')}</p>
          <p className="text-sm font-medium text-crowd-moderate">{t('footer.noLive')}</p>
        </div>
        <div className="flex flex-col gap-2 text-sm">
          <Link to="/plan" className="text-muted-foreground hover:text-foreground">{t('nav.plan')}</Link>
          <Link to="/report" className="text-muted-foreground hover:text-foreground">{t('nav.report')}</Link>
          <Link to="/insights" className="text-muted-foreground hover:text-foreground">{t('nav.insights')}</Link>
          <Link to="/about" className="text-muted-foreground hover:text-foreground">{t('nav.about')}</Link>
        </div>
      </div>
    </footer>
  );
}
#SiteHeader
import { useState } from 'react';
import { Link, useLocation } from 'react-router-dom';
import { Bus, Menu as MenuIcon, X } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import LanguageSelector from '@/components/LanguageSelector';
import StatusBar from '@/components/StatusBar';

const NAV = [
  { to: '/', key: 'nav.home' },
  { to: '/plan', key: 'nav.plan' },
  { to: '/report', key: 'nav.report' },
  { to: '/insights', key: 'nav.insights' },
  { to: '/about', key: 'nav.about' },
];

export default function SiteHeader() {
  const { t } = useLanguage();
  const { pathname } = useLocation();
  const [menuOpen, setMenuOpen] = useState(false);

  return (
    <header className="sticky top-0 z-30 border-b border-border/70 bg-background/95 backdrop-blur">
      <StatusBar />
      <div className="mx-auto flex max-w-6xl items-center justify-between gap-3 px-4 py-3 sm:px-6 lg:px-8">
        <Link to="/" className="flex items-center gap-2.5">
          <span className="flex h-9 w-9 items-center justify-center rounded-xl bg-primary text-primary-foreground">
            <Bus className="h-4 w-4" />
          </span>
          <span className="leading-tight">
            <span className="block font-display text-lg font-semibold tracking-tight">SAFAR</span>
            <span className="block text-[11px] text-muted-foreground">{t('home.eyebrow')}</span>
          </span>
        </Link>

        <nav className="hidden items-center gap-1 md:flex">
          {NAV.map((item) => {
            const active = item.to === '/' ? pathname === '/' : pathname.startsWith(item.to);
            return (
              <Link
                key={item.to}
                to={item.to}
                className={`rounded-full px-3 py-1.5 text-sm transition-colors ${
                  active ? 'bg-accent font-medium text-accent-foreground' : 'text-muted-foreground hover:bg-accent/60 hover:text-foreground'
                }`}
              >
                {t(item.key)}
              </Link>
            );
          })}
        </nav>

        <div className="flex items-center gap-2">
          <LanguageSelector />
          <button
            type="button"
            onClick={() => setMenuOpen((open) => !open)}
            aria-label={t('nav.menu')}
            aria-expanded={menuOpen}
            className="rounded-full border border-border p-2 text-foreground md:hidden"
          >
            {menuOpen ? <X className="h-4 w-4" /> : <MenuIcon className="h-4 w-4" />}
          </button>
        </div>
      </div>

      {menuOpen && (
        <nav className="border-t border-border/70 md:hidden">
          <div className="mx-auto flex max-w-6xl flex-col px-4 py-2 sm:px-6">
            {NAV.map((item) => (
              <Link
                key={item.to}
                to={item.to}
                onClick={() => setMenuOpen(false)}
                className="rounded-lg px-3 py-2.5 text-sm text-muted-foreground hover:bg-accent hover:text-foreground"
              >
                {t(item.key)}
              </Link>
            ))}
          </div>
        </nav>
      )}
    </header>
  );
}
#StatusBar
import { Activity, EyeOff } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import { useDataset } from '@/lib/DatasetContext';
import DataSourcesDialog from '@/components/DataSourcesDialog';

const CHIP = {
  sample: 'border-crowd-moderate/30 bg-crowd-moderate/10 text-crowd-moderate',
  empty: 'border-crowd-moderate/30 bg-crowd-moderate/10 text-crowd-moderate',
  importing: 'border-primary/30 bg-primary/10 text-primary',
  ready: 'border-crowd-low/35 bg-crowd-low/10 text-crowd-low',
  failed: 'border-destructive/35 bg-destructive/10 text-destructive',
};

const LABEL_KEY = {
  sample: 'status.sample',
  empty: 'status.empty',
  importing: 'status.importing',
  ready: 'status.real',
  failed: 'status.failed',
};

export default function StatusBar() {
  const { t } = useLanguage();
  const { status, progress, dataset } = useDataset();
  const chip = CHIP[status] ?? CHIP.sample;
  const label = t(LABEL_KEY[status] ?? 'status.sample');
  const count = (value) => Number(value ?? 0).toLocaleString('en-IN');

  return (
    <div className="border-b border-border/60 bg-card/60">
      <div className="mx-auto flex max-w-6xl flex-wrap items-center gap-x-3 gap-y-1.5 px-4 py-2 text-xs text-muted-foreground sm:px-6 lg:px-8">
        <span className={`inline-flex items-center gap-1.5 rounded-full border px-2.5 py-0.5 font-medium ${chip}`}>
          {status === 'importing' ? (
            <Activity className="h-3 w-3 animate-pulse" />
          ) : (
            <span className="h-1.5 w-1.5 rounded-full bg-current" />
          )}
          {label}
        </span>

        {status === 'ready' && (
          <span className="min-w-0">
            {count(dataset.routeCount)} {t('status.routes')} · {count(dataset.stopCount)} {t('status.stops')} · {count(dataset.reportCount)} {t('status.reports')}
          </span>
        )}

        {progress && (
          <span className="min-w-0">
            {t('status.importProgress', { done: count(progress.done), total: count(progress.total), unit: t(progress.unit === 'stops' ? 'status.stops' : 'status.routes') })}
          </span>
        )}

        <span className="ml-auto flex items-center gap-2">
          <span className="inline-flex items-center gap-1 rounded-full border border-border px-2.5 py-0.5" title={t('footer.noLive')}>
            <EyeOff className="h-3 w-3" />
            {t('status.noLive')}
          </span>
          <DataSourcesDialog />
        </span>
      </div>
    </div>
  );
}
#StopPicker
mport { useCallback, useEffect, useRef, useState } from 'react';
import { Loader2, Search, X } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import { useDataset } from '@/lib/DatasetContext';

export default function StopPicker({ id, label, placeholder, value, onSelect }) {
  const { t } = useLanguage();
  const { searchStops, status } = useDataset();
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [busy, setBusy] = useState(false);
  const [open, setOpen] = useState(false);
  const [touched, setTouched] = useState(false);
  const containerRef = useRef(null);

  useEffect(() => {
    const needle = query.trim();
    if (needle.length < 2) {
      setResults([]);
      return undefined;
    }
    let cancelled = false;
    setBusy(true);
    const timer = setTimeout(async () => {
      const rows = await searchStops(needle, 20);
      if (!cancelled) {
        setResults(rows);
        setBusy(false);
      }
    }, 250);
    return () => {
      cancelled = true;
      clearTimeout(timer);
    };
  }, [query, searchStops]);

  useEffect(() => {
    const onDocClick = (event) => {
      if (containerRef.current && !containerRef.current.contains(event.target)) setOpen(false);
    };
    document.addEventListener('mousedown', onDocClick);
    return () => document.removeEventListener('mousedown', onDocClick);
  }, []);

  const choose = useCallback((stop) => {
    onSelect(stop);
    setQuery('');
    setResults([]);
    setOpen(false);
    setTouched(false);
  }, [onSelect]);

  const disabled = status !== 'ready' && status !== 'sample';

  return (
    <div className="space-y-1.5" ref={containerRef}>
      <label htmlFor={id} className="text-xs font-medium text-muted-foreground">{label}</label>

      {value ? (
        <div className="flex items-center justify-between gap-2 rounded-xl border border-border bg-card px-3 py-2 text-sm">
          <span className="truncate">{value.name}</span>
          <button
            type="button"
            onClick={() => onSelect(null)}
            aria-label={t('plan.swap')}
            className="shrink-0 rounded-full p-1 text-muted-foreground hover:bg-accent"
          >
            <X className="h-3.5 w-3.5" />
          </button>
        </div>
      ) : (
        <div className="relative">
          <Search className="pointer-events-none absolute left-3 top-1/2 h-3.5 w-3.5 -translate-y-1/2 text-muted-foreground" />
          <input
            id={id}
            value={query}
            disabled={disabled}
            onChange={(event) => {
              setQuery(event.target.value);
              setTouched(true);
              setOpen(true);
            }}
            onFocus={() => setOpen(true)}
            placeholder={placeholder}
            className="flex h-9 w-full rounded-xl border border-input bg-transparent py-2 pl-9 pr-9 text-sm shadow-sm focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:cursor-not-allowed disabled:opacity-50"
          />
          {busy && <Loader2 className="absolute right-3 top-1/2 h-3.5 w-3.5 -translate-y-1/2 animate-spin text-muted-foreground" />}
        </div>
      )}

      {!value && open && touched && (
        <div className="max-h-56 overflow-y-auto rounded-xl border border-border bg-popover p-1 shadow-sm">
          {results.length === 0 && (
            <p className="px-3 py-2 text-xs text-muted-foreground">
              {query.trim().length < 2 ? t('plan.minChars') : t('search.noStops')}
            </p>
          )}
          {results.map((stop) => (
            <button
              key={stop.id ?? stop.name}
              type="button"
              onClick={() => choose(stop)}
              className="flex w-full flex-col items-start rounded-lg px-3 py-2 text-left hover:bg-accent"
            >
              <span className="text-sm">{stop.name}</span>
              <span className="text-[11px] text-muted-foreground">
                {t('search.trips', { count: stop.route_count ?? 0 })}
              </span>
            </button>
          ))}
        </div>
      )}
    </div>
  );
}
#StopSearchPanel
import { useEffect, useState } from 'react';
import { Loader2, Search } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import { useDataset } from '@/lib/DatasetContext';

export default function StopSearchPanel() {
  const { t } = useLanguage();
  const { searchStops, status } = useDataset();
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [busy, setBusy] = useState(false);
  const [searched, setSearched] = useState(false);

  useEffect(() => {
    const needle = query.trim();
    if (needle.length < 2) {
      setResults([]);
      setSearched(false);
      return undefined;
    }
    let cancelled = false;
    setBusy(true);
    const timer = setTimeout(async () => {
      const rows = await searchStops(needle, 20);
      if (!cancelled) {
        setResults(rows);
        setBusy(false);
        setSearched(true);
      }
    }, 250);
    return () => {
      cancelled = true;
      clearTimeout(timer);
    };
  }, [query, searchStops]);

  return (
    <div className="space-y-3">
      <div>
        <h3 className="font-display text-sm font-semibold">{t('search.stopsTitle')}</h3>
        <p className="text-xs text-muted-foreground">{t('search.stopsHint')}</p>
      </div>

      <div className="relative">
        <Search className="pointer-events-none absolute left-3 top-1/2 h-3.5 w-3.5 -translate-y-1/2 text-muted-foreground" />
        <input
          value={query}
          onChange={(event) => setQuery(event.target.value)}
          disabled={status !== 'ready' && status !== 'sample'}
          placeholder={t('search.stopPlaceholder')}
          className="flex h-9 w-full rounded-xl border border-input bg-transparent py-2 pl-9 pr-9 text-sm shadow-sm focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:cursor-not-allowed disabled:opacity-50"
        />
        {busy && <Loader2 className="absolute right-3 top-1/2 h-3.5 w-3.5 -translate-y-1/2 animate-spin text-muted-foreground" />}
      </div>

      {query.trim().length < 2 && <p className="text-xs text-muted-foreground">{t('search.typeMore')}</p>}
      {searched && results.length === 0 && <p className="text-xs text-muted-foreground">{t('search.noStops')}</p>}

      <div className="space-y-2">
        {results.map((stop) => (
          <article key={stop.id ?? stop.name} className="rounded-xl border border-border/70 bg-background p-3">
            <header className="flex items-center justify-between gap-2">
              <h4 className="text-sm font-medium">{stop.name}</h4>
              <span className="shrink-0 text-xs text-muted-foreground">
                {t('search.servedBy')}: {stop.route_count ?? (stop.route_numbers || []).length}
              </span>
            </header>
            <div className="mt-2 flex flex-wrap gap-1.5">
              {(stop.route_numbers || []).map((number) => (
                <span key={number} className="rounded-md border border-border bg-muted/40 px-1.5 py-0.5 text-[11px] text-muted-foreground">
                  {number}
                </span>
              ))}
            </div>
          </article>
        ))}
      </div>
    </div>
  );
}
#UserNotRegisteredError
mport React from 'react';

const UserNotRegisteredError = () => {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen bg-gradient-to-b from-white to-slate-50">
      <div className="max-w-md w-full p-8 bg-white rounded-lg shadow-lg border border-slate-100">
        <div className="text-center">
          <div className="inline-flex items-center justify-center w-16 h-16 mb-6 rounded-full bg-orange-100">
            <svg className="w-8 h-8 text-orange-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" />
            </svg>
          </div>
          <h1 className="text-3xl font-bold text-slate-900 mb-4">Access Restricted</h1>
          <p className="text-slate-600 mb-8">
            You are not registered to use this application. Please contact the app administrator to request access.
          </p>
          <div className="p-4 bg-slate-50 rounded-md text-sm text-slate-600">
            <p>If you believe this is an error, you can:</p>
            <ul className="list-disc list-inside mt-2 space-y-1">
              <li>Verify you are logged in with the correct account</li>
              <li>Contact the app administrator for access</li>
              <li>Try logging out and back in again</li>
            </ul>
          </div>
        </div>
      </div>
    </div>
  );
};

export default UserNotRegisteredError;

#hooks
#use-mobile
import * as React from "react"

const MOBILE_BREAKPOINT = 768

export function useIsMobile() {
  const [isMobile, setIsMobile] = React.useState(undefined)

  React.useEffect(() => {
    const mql = window.matchMedia(`(max-width: ${MOBILE_BREAKPOINT - 1}px)`)
    const onChange = () => {
      setIsMobile(window.innerWidth < MOBILE_BREAKPOINT)
    }
    mql.addEventListener("change", onChange)
    setIsMobile(window.innerWidth < MOBILE_BREAKPOINT)
    return () => mql.removeEventListener("change", onChange);
  }, [])

  return !!isMobile
}

#use-size
import * as React from "react"

export function useSize(ref) {
  const [size, setSize] = React.useState(null)

  // useLayoutEffect (not useEffect): the initial measurement must land before
  // the browser paints, so consumers can render their real content on the
  // very first painted frame instead of a guess. A ResizeObserver's first
  // callback arrives too late for that — by then an <img> src guess has
  // already been dispatched to the network.
  React.useLayoutEffect(() => {
    const element = ref.current
    if (!element) return

    const rect = element.getBoundingClientRect()
    setSize({ width: rect.width, height: rect.height })

    const observer = new ResizeObserver(([entry]) => {
      const { width, height } = entry.contentRect
      setSize({ width, height })
    })

    observer.observe(element)
    return () => observer.disconnect()
  }, [ref])

  return size
}
#lib
#app-params
import { getAccessToken } from '@base44/sdk';

const isNode = typeof window === 'undefined';

const isClearAccessTokenRequested = () =>
	!isNode && new URLSearchParams(window.location.search).get("clear_access_token") === 'true';

const clearStoredAccessToken = () => {
	window.localStorage.removeItem('base44_access_token');
	window.localStorage.removeItem('token');
}

const getAppParams = () => {
	if (isClearAccessTokenRequested()) {
		clearStoredAccessToken();
	}
	return {
		appId: import.meta.env.VITE_BASE44_APP_ID,
		token: getAccessToken(),
		functionsVersion: (!isNode && new URLSearchParams(window.location.search).get("functions_version")?.trim()) || import.meta.env.VITE_BASE44_FUNCTIONS_VERSION,
		appBaseUrl: import.meta.env.VITE_BASE44_APP_BASE_URL,
	}
}


export const appParams = {
	...getAppParams()
}
#AuthContext
import React, { createContext, useState, useContext, useEffect } from 'react';
import { base44 } from '@/api/base44Client';
import { appParams } from '@/lib/app-params';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [isLoadingAuth, setIsLoadingAuth] = useState(true);
  const [isLoadingPublicSettings, setIsLoadingPublicSettings] = useState(true);
  const [authError, setAuthError] = useState(null);
  const [authChecked, setAuthChecked] = useState(false);
  const [appPublicSettings, setAppPublicSettings] = useState(null); // Contains only { id, public_settings }

  useEffect(() => {
    checkAppState();
  }, []);

  const checkAppState = async () => {
    try {
      setIsLoadingPublicSettings(true);
      setAuthError(null);
      
      try {
        const publicSettings = await base44.app.getPublicSettings();
        setAppPublicSettings(publicSettings);
        
        // If we got the app public settings successfully, check if user is authenticated
        if (appParams.token) {
          await checkUserAuth();
        } else {
          setIsLoadingAuth(false);
          setIsAuthenticated(false);
          setAuthChecked(true);
        }
        setIsLoadingPublicSettings(false);
      } catch (appError) {
        console.error('App state check failed:', appError);
        
        // Handle app-level errors
        if (appError.status === 403 && appError.data?.extra_data?.reason) {
          const reason = appError.data.extra_data.reason;
          if (reason === 'auth_required') {
            setAuthError({
              type: 'auth_required',
              message: 'Authentication required'
            });
          } else if (reason === 'user_not_registered') {
            setAuthError({
              type: 'user_not_registered',
              message: 'User not registered for this app'
            });
          } else {
            setAuthError({
              type: reason,
              message: appError.message
            });
          }
        } else {
          setAuthError({
            type: 'unknown',
            message: appError.message || 'Failed to load app'
          });
        }
        setIsLoadingPublicSettings(false);
        setIsLoadingAuth(false);
      }
    } catch (error) {
      console.error('Unexpected error:', error);
      setAuthError({
        type: 'unknown',
        message: error.message || 'An unexpected error occurred'
      });
      setIsLoadingPublicSettings(false);
      setIsLoadingAuth(false);
    }
  };

  const checkUserAuth = async () => {
    try {
      // Now check if the user is authenticated
      setIsLoadingAuth(true);
      const currentUser = await base44.auth.me();
      setUser(currentUser);
      setIsAuthenticated(true);
      setIsLoadingAuth(false);
      setAuthChecked(true);
    } catch (error) {
      console.error('User auth check failed:', error);
      setIsLoadingAuth(false);
      setIsAuthenticated(false);
      setAuthChecked(true);
      
      // If user auth fails, it might be an expired token
      if (error.status === 401 || error.status === 403) {
        setAuthError({
          type: 'auth_required',
          message: 'Authentication required'
        });
      }
    }
  };

  const logout = (shouldRedirect = true) => {
    setUser(null);
    setIsAuthenticated(false);
    
    if (shouldRedirect) {
      // Use the SDK's logout method which handles token cleanup and redirect
      base44.auth.logout(window.location.href);
    } else {
      // Just remove the token without redirect
      base44.auth.logout();
    }
  };

  const navigateToLogin = () => {
    // Use the SDK's redirectToLogin method
    base44.auth.redirectToLogin(window.location.href);
  };

  return (
    <AuthContext.Provider value={{ 
      user, 
      isAuthenticated, 
      isLoadingAuth,
      isLoadingPublicSettings,
      authError,
      appPublicSettings,
      authChecked,
      logout,
      navigateToLogin,
      checkUserAuth,
      checkAppState
    }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
};
#authReturnTo
// Shared by the auth pages (Login, Register, and any page that resumes a flow
// after sign-in, e.g. the MCP OAuth consent page). Keep the redirect
// validation in one place — it is security-sensitive and easy to drift.

// Resolve ?returnTo= to a safe same-origin path, else "/".
//
// The same-origin check alone is not enough: a value like /.//evil.com or
// /\evil.com parses same-origin but normalizes to a protocol-relative
// //evil.com when assigned to location.href — an open redirect. So require the
// resolved path to be exactly one leading slash (no "//" prefix, no backslash).
export function safeReturnTo() {
  const raw = new URLSearchParams(window.location.search).get("returnTo");
  if (!raw) return "/";
  try {
    const url = new URL(raw, window.location.origin);
    if (url.origin !== window.location.origin) return "/";
    // Only access_token/clear_access_token are still URL-read by app-params.js, but the
    // whole bootstrap set stays stripped: one going back to a URL read must not silently
    // become injectable again. Normal app-flow params (e.g. the OAuth consent ctx) are kept.
    for (const p of ["access_token", "clear_access_token", "app_id", "app_base_url", "functions_version", "from_url"]) {
      url.searchParams.delete(p);
    }
    const path = url.pathname + url.search;
    if (!path.startsWith("/") || path.startsWith("//") || path.includes("\\")) return "/";
    return path;
  } catch {
    return "/";
  }
}
#crowd
export const CROWD_LEVELS = ['low', 'moderate', 'high', 'full'];

export const RECENT_HOURS = 2;
export const DUPLICATE_WINDOW_MINUTES = 15;

export const LEVEL_META = {
  low: {
    labelKey: 'report.low',
    chip: 'border-crowd-low/35 bg-crowd-low/10 text-crowd-low',
    dot: 'bg-crowd-low',
    text: 'text-crowd-low',
    border: 'border-crowd-low',
  },
  moderate: {
    labelKey: 'report.moderate',
    chip: 'border-crowd-moderate/35 bg-crowd-moderate/10 text-crowd-moderate',
    dot: 'bg-crowd-moderate',
    text: 'text-crowd-moderate',
    border: 'border-crowd-moderate',
  },
  high: {
    labelKey: 'report.high',
    chip: 'border-crowd-high/35 bg-crowd-high/10 text-crowd-high',
    dot: 'bg-crowd-high',
    text: 'text-crowd-high',
    border: 'border-crowd-high',
  },
  full: {
    labelKey: 'report.full',
    chip: 'border-crowd-full/35 bg-crowd-full/10 text-crowd-full',
    dot: 'bg-crowd-full',
    text: 'text-crowd-full',
    border: 'border-crowd-full',
  },
};

export const levelMeta = (level) => LEVEL_META[level] ?? LEVEL_META.moderate;

export const ageMinutes = (iso, now = Date.now()) => {
  const time = new Date(iso).getTime();
  if (Number.isNaN(time)) return Number.POSITIVE_INFINITY;
  return Math.max(0, Math.round((now - time) / 60000));
};

export const isStale = (iso, now = Date.now()) => ageMinutes(iso, now) > RECENT_HOURS * 60;

export const ageLabel = (iso, t, now = Date.now()) => {
  const minutes = ageMinutes(iso, now);
  if (minutes < 1) return t('feed.justNow');
  if (minutes < 60) return t('feed.minutesAgo', { count: minutes });
  return t('feed.hoursAgo', { count: Math.round(minutes / 60) });
};
#DatasetContext
import { createContext, useCallback, useContext, useEffect, useMemo, useRef, useState } from 'react';
import { base44 } from '@/api/base44Client';
import { RECENT_HOURS, DUPLICATE_WINDOW_MINUTES, CROWD_LEVELS } from '@/lib/crowd';

const DatasetContext = createContext(null);

const REPORTER_KEY = 'safar.reporter_token';

// Verified dataset facts (checked against the published source).
export const SOURCE = {
  name: 'bmtc-gtfs — unofficial BMTC GTFS dataset (from the official Namma BMTC app)',
  url: 'https://github.com/Vonter/bmtc-gtfs',
  csvUrl: 'https://raw.githubusercontent.com/Vonter/bmtc-gtfs/main/csv/routes.csv',
  licence: 'ODbL 1.0',
};

const emptyDataset = {
  sourceName: SOURCE.name,
  sourceUrl: SOURCE.url,
  sourceCsvUrl: SOURCE.csvUrl,
  licence: SOURCE.licence,
  commitDate: null,
  checkedAt: null,
  importedAt: null,
  routeCount: 0,
  stopCount: 0,
  reportCount: 0,
  lastError: null,
};

const reporterToken = () => {
  try {
    const existing = window.localStorage.getItem(REPORTER_KEY);
    if (existing) return existing;
    const created = window.crypto?.randomUUID
      ? window.crypto.randomUUID()
      : `token-${Math.random().toString(36).slice(2)}-${Date.now()}`;
    window.localStorage.setItem(REPORTER_KEY, created);
    return created;
  } catch (error) {
    return 'anonymous';
  }
};

const items = (page) => (Array.isArray(page) ? page : page?.items ?? []);

export function DatasetProvider({ children }) {
  const [status, setStatus] = useState('empty');
  const [dataset, setDataset] = useState(emptyDataset);
  const [reports, setReports] = useState([]);
  const [reportsLoading, setReportsLoading] = useState(true);
  const [totalReports, setTotalReports] = useState(0);
  const [hasMoreReports, setHasMoreReports] = useState(false);
  const [insights, setInsights] = useState({ total: 0, last24: 0, stale: 0, byLevel: [], byDay: [] });
  const reportCursor = useRef(null);
  const loading = useRef(false);

  const loadReports = useCallback(async () => {
    setReportsLoading(true);
    const page = await base44.entities.CrowdReport.list({ sort: '-reported_at', limit: 25 });
    setReports(items(page));
    reportCursor.current = page?.next_cursor ?? null;
    setHasMoreReports(Boolean(page?.has_more));
    setTotalReports(await base44.entities.CrowdReport.count({}));
    setReportsLoading(false);
  }, []);

  const loadInsights = useCallback(async () => {
    const now = Date.now();
    const since24 = new Date(now - 24 * 3600000).toISOString();
    const staleSince = new Date(now - RECENT_HOURS * 3600000).toISOString();
    const [total, last24, stale, levelRows, dayRows] = await Promise.all([
      base44.entities.CrowdReport.count({}).catch(() => 0),
      base44.entities.CrowdReport.count({ reported_at: { $gte: since24 } }).catch(() => 0),
      base44.entities.CrowdReport.count({ reported_at: { $lt: staleSince } }).catch(() => 0),
      base44.entities.CrowdReport.aggregate({ groupBy: 'crowd_level' }).catch(() => ({ rows: [] })),
      base44.entities.CrowdReport.aggregate({ dateBucket: { field: 'reported_at', unit: 'day' } }).catch(() => ({ rows: [] })),
    ]);
    const byLevel = (levelRows?.rows ?? [])
      .filter((row) => CROWD_LEVELS.includes(row.crowd_level))
      .map((row) => ({ level: row.crowd_level, count: row.count }));
    const byDay = (dayRows?.rows ?? [])
      .map((row) => ({ date: row.reported_at, count: row.count }))
      .sort((a, b) => String(a.date).localeCompare(String(b.date)));
    setInsights({ total, last24, stale, byLevel, byDay });
  }, []);

  const loadStatus = useCallback(async () => {
    const response = await base44.functions.invoke('bmtcSync', { action: 'status' });
    const record = response?.data?.dataset ?? null;
    setDataset((previous) => ({
      ...previous,
      sourceName: record?.source_name || SOURCE.name,
      sourceUrl: record?.source_url || SOURCE.url,
      sourceCsvUrl: record?.data_url || SOURCE.csvUrl,
      licence: record?.licence || SOURCE.licence,
      commitDate: record?.commit_date || previous.commitDate,
      checkedAt: record?.last_checked_at || previous.checkedAt,
      importedAt: record?.imported_at || previous.importedAt,
      routeCount: record?.route_count ?? 0,
      stopCount: record?.stop_count ?? 0,
      lastError: record?.status === 'failed' ? record.notes || null : null,
    }));
    const next = loading.current
      ? 'importing'
      : record?.status === 'ready'
        ? 'ready'
        : record?.status === 'failed'
          ? 'failed'
          : 'empty';
    setStatus(next);
    return next;
  }, []);

  // Reads the published dataset through the app's own service and records its
  // source, licence, freshness and real counts. Route and stop rows themselves
  // stay in the service's cache instead of being copied into the database.
  const loadFromSource = useCallback(async () => {
    if (loading.current) return;
    loading.current = true;
    setStatus('importing');
    try {
      const response = await base44.functions.invoke('bmtcSync', { action: 'load' });
      const data = response?.data ?? {};
      if (!data.ok) throw new Error(data.dataset?.notes || 'The published dataset could not be read');
      loading.current = false;
      await loadStatus();
      await loadReports();
      await loadInsights();
    } catch (error) {
      loading.current = false;
      setStatus('failed');
      setDataset((previous) => ({ ...previous, lastError: error?.message ?? 'The published dataset could not be read' }));
    }
  }, [loadStatus, loadReports, loadInsights]);

  const refreshSource = useCallback(async () => {
    await loadFromSource();
  }, [loadFromSource]);

  const searchStops = useCallback(async (query, limit = 20) => {
    const needle = String(query ?? '').trim();
    if (needle.length < 2) return [];
    const response = await base44.functions.invoke('bmtcQuery', { action: 'searchStops', query: needle, limit });
    const rows = response?.data?.stops ?? [];
    return rows.map((stop) => ({
      id: stop.name,
      name: stop.name,
      route_count: stop.route_count,
      route_numbers: stop.route_numbers ?? [],
    }));
  }, []);

  const searchRoutes = useCallback(async (query, limit = 20) => {
    const needle = String(query ?? '').trim();
    if (needle.length < 2) return [];
    const response = await base44.functions.invoke('bmtcQuery', { action: 'searchRoutes', query: needle, limit });
    return response?.data?.routes ?? [];
  }, []);

  const planJourney = useCallback(async (fromName, toName) => {
    const response = await base44.functions.invoke('bmtcQuery', { action: 'plan', from: fromName, to: toName, limit: 8 });
    return { options: response?.data?.options ?? [] };
  }, []);

  // Candidate journeys (direct and one transfer) with the dataset's own
  // assumptions attached, for the full trip planner.
  const planJourneys = useCallback(async ({ from, to, via, departMinutes }) => {
    const response = await base44.functions.invoke('bmtcQuery', {
      action: 'planJourneys',
      from,
      to,
      via: via || '',
      departMinutes,
      limit: 45,
    });
    const data = response?.data ?? {};
    return {
      ok: Boolean(data.ok),
      journeys: Array.isArray(data.journeys) ? data.journeys : [],
      assumptions: data.assumptions ?? null,
    };
  }, []);

  const submitReport = useCallback(async ({ routeNumber, routeVerified, tripRef, stopName, level, reportedAt }) => {
    if (!stopName) return { ok: false, reason: 'invalid' };
    if (!CROWD_LEVELS.includes(level)) return { ok: false, reason: 'invalid' };
    const observed = new Date(reportedAt);
    if (Number.isNaN(observed.getTime())) return { ok: false, reason: 'invalid' };
    const driftMinutes = (Date.now() - observed.getTime()) / 60000;
    if (driftMinutes < -5 || driftMinutes > RECENT_HOURS * 60 * 4) return { ok: false, reason: 'invalid' };

    const token = reporterToken();
    const since = new Date(Date.now() - DUPLICATE_WINDOW_MINUTES * 60000).toISOString();
    const existing = await base44.entities.CrowdReport.filter(
      { reporter_token: token, stop_name: stopName, reported_at: { $gte: since } },
      { sort: '-reported_at', limit: 1 },
    );
    const previous = items(existing)[0];
    if (previous) return { ok: false, reason: 'duplicate', lastAt: previous.reported_at };

    const created = await base44.entities.CrowdReport.create({
      route_number: routeNumber || '',
      trip_ref: tripRef || '',
      stop_name: stopName,
      crowd_level: level,
      reported_at: observed.toISOString(),
      reporter_token: token,
      route_verified: Boolean(routeVerified),
      source: 'passenger_report',
    });

    setReports((list) => [created, ...list.filter((item) => item.id !== created.id)].slice(0, 60));
    setTotalReports((count) => count + 1);
    await loadInsights();
    return { ok: true, report: created };
  }, [loadInsights]);

  const loadMoreReports = useCallback(async () => {
    if (!reportCursor.current) return;
    const page = await base44.entities.CrowdReport.list({
      sort: '-reported_at',
      limit: 25,
      cursor: reportCursor.current,
    });
    setReports((list) => [...list, ...items(page)]);
    reportCursor.current = page?.next_cursor ?? null;
    setHasMoreReports(Boolean(page?.has_more));
  }, []);

  useEffect(() => {
    let cancelled = false;
    const boot = async () => {
      const current = await loadStatus();
      if (cancelled) return;
      await loadReports();
      await loadInsights();
      if (current !== 'ready') await loadFromSource();
    };
    boot();
    return () => {
      cancelled = true;
    };
  }, [loadStatus, loadReports, loadInsights, loadFromSource]);

  useEffect(() => {
    if (typeof base44.entities.CrowdReport.subscribe !== 'function') return undefined;
    const unsubscribe = base44.entities.CrowdReport.subscribe((event) => {
      if (event?.type !== 'create' || !event.data) return;
      setReports((list) => [event.data, ...list.filter((item) => item.id !== event.data.id)].slice(0, 60));
      setTotalReports((count) => count + 1);
    });
    return unsubscribe;
  }, []);

  const value = useMemo(() => ({
    status,
    progress: null,
    dataset,
    searchStops,
    searchRoutes,
    planJourney,
    planJourneys,
    reports,
    reportsLoading,
    totalReports,
    hasMoreReports,
    loadMoreReports,
    submitReport,
    insights,
    importDataset: loadFromSource,
    refreshSource,
  }), [
    status, dataset, searchStops, searchRoutes, planJourney, planJourneys, reports, reportsLoading,
    totalReports, hasMoreReports, loadMoreReports, submitReport, insights, loadFromSource, refreshSource,
  ]);

  return <DatasetContext.Provider value={value}>{children}</DatasetContext.Provider>;
}

export const useDataset = () => useContext(DatasetContext);
#i18n
import { plannerText } from '@/lib/i18nPlanner';

export const LANGUAGES = [
  { code: 'en', name: 'English', native: 'English' },
  { code: 'kn', name: 'Kannada', native: 'ಕನ್ನಡ' },
  { code: 'te', name: 'Telugu', native: 'తెలుగు' },
  { code: 'ta', name: 'Tamil', native: 'தமிழ்' },
  { code: 'hi', name: 'Hindi', native: 'हिन्दी' },
];

const en = {
  'nav.home': 'Home',
  'nav.plan': 'Trip planner',
  'nav.report': 'Report crowd',
  'nav.insights': 'Insights',
  'nav.about': 'About',
  'nav.menu': 'Menu',
  'nav.changeLanguage': 'Change language',

  'status.sample': 'Sample excerpt of the BMTC dataset — the full dataset is not loaded yet',
  'status.empty': 'The BMTC dataset has not been loaded yet',
  'status.importing': 'Loading the real BMTC dataset',
  'status.real': 'Real BMTC dataset',
  'status.failed': 'The dataset could not be loaded',
  'status.routes': 'route directions',
  'status.stops': 'bus stops',
  'status.reports': 'passenger reports',
  'status.sources': 'Data sources',
  'status.noLive': 'No live vehicle positions',
  'status.importCta': 'Load the real BMTC dataset',
  'status.importProgress': '{done} of {total} {unit} loaded',
  'status.importDone': 'Loaded from the published source',
  'status.checking': 'Checking the source…',
  'status.refresh': 'Re-check the source',
  'status.needImport': 'Loading the published dataset enables route search, stop search and journey planning.',

  'home.eyebrow': 'Bengaluru city buses',
  'home.title': 'Know how crowded your bus is before you leave.',
  'home.subtitle': 'SAFAR pairs real BMTC route, stop and timetable data with crowd reports from fellow commuters, so you can plan a calmer journey across the city.',

  'plan.eyebrow': 'Trip planner',
  'plan.title': 'Plan a journey',
  'plan.description': 'Choose a starting stop and a destination from the imported BMTC stop list.',
  'plan.from': 'From stop',
  'plan.to': 'To stop',
  'plan.chooseFrom': 'Choose a starting stop',
  'plan.chooseTo': 'Choose a destination',
  'plan.swap': 'Swap stops',
  'plan.search': 'Search routes',
  'plan.searching': 'Searching routes…',
  'plan.results': 'Direct routes',
  'plan.count': '{count} routes serve both stops',
  'plan.none': 'No route in the imported dataset serves both of these stops directly.',
  'plan.stopsBetween': '{count} stops between them',
  'plan.boardAt': 'Board at {stop}',
  'plan.alightAt': 'Get off at {stop}',
  'plan.departures': 'Origin departures',
  'plan.firstLast': 'First {first} · last {last}',
  'plan.note': 'Times are the scheduled departures from the route origin, exactly as published in the dataset. Stop-level times are not part of the dataset, so they are not shown.',
  'plan.needBoth': 'Choose both stops to see which routes connect them.',
  'plan.minChars': 'Type at least two letters to search real stop names.',

  'report.eyebrow': 'Passenger reports',
  'report.title': 'Report how crowded it is',
  'report.description': 'Everything shown on SAFAR comes from passengers. Nothing is generated.',
  'report.route': 'Bus route number',
  'report.routeHint': 'Optional. Pick a route from the dataset, or type the number you saw.',
  'report.routePlaceholder': 'e.g. 500-D',
  'report.trip': 'Bus or trip number (optional)',
  'report.tripPlaceholder': 'e.g. KA-57-F-1234',
  'report.stop': 'Stop',
  'report.stopHint': 'Only stops from the imported dataset can be chosen.',
  'report.level': 'How crowded is it?',
  'report.low': 'Low',
  'report.moderate': 'Moderate',
  'report.high': 'High',
  'report.full': 'Full',
  'report.recency': 'When did you see this?',
  'report.now': 'Just now',
  'report.min15': 'Within the last 15 minutes',
  'report.min60': 'Within the last hour',
  'report.submit': 'Submit report',
  'report.saving': 'Saving…',
  'report.saved': 'Report saved',
  'report.savedHelp': 'It is now visible to other commuters.',
  'report.another': 'Submit another report',
  'report.privacy': 'Reports are anonymous. No name, phone number or account is stored.',
  'report.errStop': 'Choose a stop from the dataset.',
  'report.errLevel': 'Choose a crowd level.',
  'report.errSave': 'The report could not be saved. Please try again.',
  'report.errDuplicate': 'You reported this same route and stop minutes ago, so nothing was saved twice.',
  'report.lastAt': 'Your last report for this stop was at {time}.',
  'report.needDataset': 'Reports need the published stop list, so the dataset has to be loaded first.',

  'feed.eyebrow': 'Crowd reports',
  'feed.title': 'What commuters are reporting',
  'feed.description': 'Every item is a passenger submission shown with its own timestamp.',
  'feed.recent': 'Recent reports',
  'feed.older': 'Older reports',
  'feed.stale': 'May be outdated',
  'feed.staleHint': 'Older than {hours} hours',
  'feed.empty': 'No reports yet',
  'feed.first': 'Be the first to tell other commuters how crowded your stop or bus looked.',
  'feed.showing': 'Showing {count} of {total}',
  'feed.justNow': 'just now',
  'feed.minutesAgo': '{count} min ago',
  'feed.hoursAgo': '{count} h ago',
  'feed.noRoute': 'No route number given',
  'feed.loading': 'Loading reports…',
  'feed.cta': 'Report what you see',
  'feed.live': 'New reports appear here as they are submitted. All times are Bengaluru time.',

  'search.routesTab': 'Routes',
  'search.stopsTab': 'Stops',
  'search.routesTitle': 'Search BMTC routes',
  'search.routesHint': 'Search by route number, or by either end of the route.',
  'search.stopsTitle': 'Search bus stops',
  'search.stopsHint': 'Search a stop and see which routes serve it.',
  'search.routePlaceholder': 'Route number or name, e.g. 500-D',
  'search.stopPlaceholder': 'Stop name, e.g. Shanthinagara',
  'search.typeMore': 'Type at least two letters.',
  'search.noRoutes': 'No route in the dataset matches that.',
  'search.noStops': 'No stop in the dataset matches that.',
  'search.trips': '{count} scheduled trips',
  'search.stopsCount': '{count} stops',
  'search.servedBy': 'Routes serving this stop',
  'search.onRoute': 'Stop order',
  'search.matched': 'Matched from the published dataset',

  'insights.eyebrow': 'Crowding insights',
  'insights.title': 'What the reports add up to',
  'insights.description': 'Built only from the passenger reports recorded in this app. Nothing is estimated or simulated.',
  'insights.byLevel': 'Reports by crowd level',
  'insights.byDay': 'Reports per day',
  'insights.total': 'Total reports',
  'insights.last24': 'In the last 24 hours',
  'insights.empty': 'No reports yet, so there is nothing to chart.',

  'about.eyebrow': 'About SAFAR',
  'about.title': 'What is real here, and what is not',
  'about.realTitle': 'Real data',
  'about.realBody': 'Route numbers, route names, ordered stop lists, stop names and scheduled departure times are read from the BMTC GTFS dataset listed below and served to this app through its own backend cache.',
  'about.reportsTitle': 'Passenger reports',
  'about.reportsBody': 'Crowd levels come only from reports submitted by commuters, each with its own timestamp. Reports older than {hours} hours are labelled as possibly outdated.',
  'about.noTitle': 'Not available',
  'about.noBody': 'There is no permitted public feed of live bus positions or vehicle occupancy for BMTC. SAFAR therefore never shows a live bus location, a live crowd level or a passenger count.',
  'about.sourceTitle': 'Source and licence',
  'about.sourceBody': 'bmtc-gtfs is an unofficial GTFS dataset built from the official Namma BMTC app and published on GitHub by Vonter. Licence: ODbL 1.0 — attribution and share-alike required.',
  'about.caveatTitle': 'Known limits of the source',
  'about.caveatBody': 'The dataset authors state that the Namma BMTC app data is inaccurate and that timetables and stop timings are known to be incorrect. Treat timetable times as indicative only.',
  'about.coverageTitle': 'Coverage',
  'about.coverageBody': 'Only routes with working live tracking in the source app are included in the dataset, so a missing route may simply not be covered.',
  'about.openSource': 'Open the source dataset',
  'about.snapshot': 'Source commit {date} · checked {checked}',

  'footer.blurb': 'SAFAR is an independent commuter project for Bengaluru. It is not affiliated with BMTC or the Government of Karnataka.',
  'footer.noLive': 'No live vehicle tracking is shown.',
  'footer.how': 'Route and stop data: BMTC GTFS dataset (ODbL). Crowd levels: passenger reports only.',
};

const kn = {
  'nav.home': 'ಮುಖಪುಟ',
  'nav.plan': 'ಪ್ರಯಾಣ ಯೋಜನೆ',
  'nav.report': 'ಜನದಟ್ಟಣೆ ವರದಿ',
  'nav.insights': 'ಒಳನೋಟಗಳು',
  'nav.about': 'ಪರಿಚಯ',
  'nav.menu': 'ಮೆನು',
  'nav.changeLanguage': 'ಭಾಷೆ ಬದಲಾಯಿಸಿ',

  'status.sample': 'BMTC ದತ್ತಾಂಶದ ಮಾದರಿ ಭಾಗ — ಪೂರ್ಣ ದತ್ತಾಂಶ ಇನ್ನೂ ಆಮದಾಗಿಲ್ಲ',
  'status.empty': 'BMTC ದತ್ತಾಂಶ ಇನ್ನೂ ಲೋಡ್ ಆಗಿಲ್ಲ',
  'status.importing': 'ನಿಜವಾದ BMTC ದತ್ತಾಂಶ ಲೋಡ್ ಆಗುತ್ತಿದೆ',
  'status.real': 'ನಿಜವಾದ BMTC ದತ್ತಾಂಶ',
  'status.failed': 'ದತ್ತಾಂಶ ಲೋಡ್ ಆಗಲಿಲ್ಲ',
  'status.routes': 'ಮಾರ್ಗ ದಿಕ್ಕುಗಳು',
  'status.stops': 'ನಿಲ್ದಾಣಗಳು',
  'status.reports': 'ಪ್ರಯಾಣಿಕರ ವರದಿಗಳು',
  'status.sources': 'ದತ್ತಾಂಶ ಮೂಲಗಳು',
  'status.noLive': 'ನೇರ ಬಸ್ ಸ್ಥಾನಗಳಿಲ್ಲ',
  'status.importCta': 'ನಿಜವಾದ BMTC ದತ್ತಾಂಶ ಲೋಡ್ ಮಾಡಿ',
  'status.importProgress': '{total} ರಲ್ಲಿ {done} {unit} ಲೋಡ್ ಆಗಿದೆ',
  'status.importDone': 'ಪ್ರಕಟಿತ ಮೂಲದಿಂದ ಲೋಡ್ ಆಗಿದೆ',
  'status.checking': 'ಮೂಲವನ್ನು ಪರಿಶೀಲಿಸುತ್ತಿದೆ…',
  'status.refresh': 'ಮೂಲವನ್ನು ಮತ್ತೆ ಪರಿಶೀಲಿಸಿ',
  'status.needImport': 'ಮಾರ್ಗ ಹುಡುಕಾಟ, ನಿಲ್ದಾಣ ಹುಡುಕಾಟ ಮತ್ತು ಯೋಜನೆಗೆ ಆಮದಾದ ದತ್ತಾಂಶ ಬೇಕು.',

  'home.eyebrow': 'ಬೆಂಗಳೂರು ನಗರ ಬಸ್ಗಳು',
  'home.title': 'ಹೊರಡುವ ಮೊದಲು ನಿಮ್ಮ ಬಸ್ ಎಷ್ಟು ಜನದಟ್ಟಣೆ ಇದೆ ಎಂದು ತಿಳಿಯಿರಿ.',
  'home.subtitle': 'SAFAR ನಿಜವಾದ BMTC ಮಾರ್ಗ, ನಿಲ್ದಾಣ ಮತ್ತು ವೇಳಾಪಟ್ಟಿ ದತ್ತಾಂಶವನ್ನು ಸಹ ಪ್ರಯಾಣಿಕರ ಜನದಟ್ಟಣೆ ವರದಿಗಳೊಂದಿಗೆ ಸೇರಿಸುತ್ತದೆ.',

  'plan.eyebrow': 'ಪ್ರಯಾಣ ಯೋಜನೆ',
  'plan.title': 'ಪ್ರಯಾಣ ಯೋಜಿಸಿ',
  'plan.description': 'ಆಮದಾದ BMTC ನಿಲ್ದಾಣ ಪಟ್ಟಿಯಿಂದ ಆರಂಭ ಮತ್ತು ಗಮ್ಯ ನಿಲ್ದಾಣವನ್ನು ಆಯ್ಕೆಮಾಡಿ.',
  'plan.from': 'ಎಲ್ಲಿಂದ',
  'plan.to': 'ಎಲ್ಲಿಗೆ',
  'plan.chooseFrom': 'ಆರಂಭ ನಿಲ್ದಾಣ ಆಯ್ಕೆಮಾಡಿ',
  'plan.chooseTo': 'ಗಮ್ಯ ನಿಲ್ದಾಣ ಆಯ್ಕೆಮಾಡಿ',
  'plan.swap': 'ನಿಲ್ದಾಣಗಳನ್ನು ಬದಲಾಯಿಸಿ',
  'plan.search': 'ಮಾರ್ಗ ಹುಡುಕಿ',
  'plan.searching': 'ಮಾರ್ಗ ಹುಡುಕುತ್ತಿದೆ…',
  'plan.results': 'ನೇರ ಮಾರ್ಗಗಳು',
  'plan.count': 'ಎರಡೂ ನಿಲ್ದಾಣಗಳಿಗೆ {count} ಮಾರ್ಗಗಳು ಸೇವೆ ಸಲ್ಲಿಸುತ್ತವೆ',
  'plan.none': 'ಈ ಎರಡೂ ನಿಲ್ದಾಣಗಳಿಗೆ ನೇರವಾಗಿ ಸೇವೆ ಸಲ್ಲಿಸುವ ಮಾರ್ಗ ದತ್ತಾಂಶದಲ್ಲಿ ಇಲ್ಲ.',
  'plan.stopsBetween': 'ನಡುವೆ {count} ನಿಲ್ದಾಣಗಳು',
  'plan.boardAt': '{stop} ನಲ್ಲಿ ಹತ್ತಿ',
  'plan.alightAt': '{stop} ನಲ್ಲಿ ಇಳಿಯಿರಿ',
  'plan.departures': 'ಆರಂಭ ನಿಲ್ದಾಣದ ನಿರ್ಗಮನಗಳು',
  'plan.firstLast': 'ಮೊದಲ {first} · ಕೊನೆ {last}',
  'plan.note': 'ಸಮಯಗಳು ದತ್ತಾಂಶದಲ್ಲಿ ಇರುವಂತೆ ಮಾರ್ಗದ ಆರಂಭ ನಿಲ್ದಾಣದ ನಿಗದಿತ ನಿರ್ಗಮನ ಸಮಯಗಳು. ನಿಲ್ದಾಣ ಮಟ್ಟದ ಸಮಯ ದತ್ತಾಂಶದಲ್ಲಿಲ್ಲ.',
  'plan.needBoth': 'ಯಾವ ಮಾರ್ಗಗಳು ಸಂಪರ್ಕಿಸುತ್ತವೆ ಎಂದು ನೋಡಲು ಎರಡೂ ನಿಲ್ದಾಣಗಳನ್ನು ಆಯ್ಕೆಮಾಡಿ.',
  'plan.minChars': 'ನಿಜವಾದ ನಿಲ್ದಾಣದ ಹೆಸರುಗಳನ್ನು ಹುಡುಕಲು ಕನಿಷ್ಠ ಎರಡು ಅಕ್ಷರ ಬರೆಯಿರಿ.',

  'report.eyebrow': 'ಪ್ರಯಾಣಿಕರ ವರದಿಗಳು',
  'report.title': 'ಎಷ್ಟು ಜನದಟ್ಟಣೆ ಇದೆ ಎಂದು ವರದಿ ಮಾಡಿ',
  'report.description': 'SAFAR ನಲ್ಲಿ ಕಾಣುವ ಎಲ್ಲವೂ ಪ್ರಯಾಣಿಕರಿಂದ ಬರುತ್ತದೆ. ಯಾವುದೂ ರಚಿಸಲಾಗಿಲ್ಲ.',
  'report.route': 'ಬಸ್ ಮಾರ್ಗ ಸಂಖ್ಯೆ',
  'report.routeHint': 'ಐಚ್ಛಿಕ. ದತ್ತಾಂಶದಿಂದ ಮಾರ್ಗ ಆಯ್ಕೆಮಾಡಿ ಅಥವಾ ಕಂಡ ಸಂಖ್ಯೆಯನ್ನು ಬರೆಯಿರಿ.',
  'report.routePlaceholder': 'ಉದಾ. 500-D',
  'report.trip': 'ಬಸ್ ಅಥವಾ ಟ್ರಿಪ್ ಸಂಖ್ಯೆ (ಐಚ್ಛಿಕ)',
  'report.tripPlaceholder': 'ಉದಾ. KA-57-F-1234',
  'report.stop': 'ನಿಲ್ದಾಣ',
  'report.stopHint': 'ಆಮದಾದ ದತ್ತಾಂಶದ ನಿಲ್ದಾಣಗಳನ್ನು ಮಾತ್ರ ಆಯ್ಕೆಮಾಡಬಹುದು.',
  'report.level': 'ಎಷ್ಟು ಜನದಟ್ಟಣೆ ಇದೆ?',
  'report.low': 'ಕಡಿಮೆ',
  'report.moderate': 'ಮಧ್ಯಮ',
  'report.high': 'ಹೆಚ್ಚು',
  'report.full': 'ತುಂಬಿದೆ',
  'report.recency': 'ಇದನ್ನು ಯಾವಾಗ ನೋಡಿದಿರಿ?',
  'report.now': 'ಈಗಷ್ಟೇ',
  'report.min15': 'ಕಳೆದ 15 ನಿಮಿಷಗಳೊಳಗೆ',
  'report.min60': 'ಕಳೆದ ಒಂದು ಗಂಟೆಯೊಳಗೆ',
  'report.submit': 'ವರದಿ ಸಲ್ಲಿಸಿ',
  'report.saving': 'ಉಳಿಸುತ್ತಿದೆ…',
  'report.saved': 'ವರದಿ ಉಳಿಸಲಾಗಿದೆ',
  'report.savedHelp': 'ಈಗ ಅದು ಇತರ ಪ್ರಯಾಣಿಕರಿಗೆ ಕಾಣುತ್ತದೆ.',
  'report.another': 'ಇನ್ನೊಂದು ವರದಿ ಸಲ್ಲಿಸಿ',
  'report.privacy': 'ವರದಿಗಳು ಅನಾಮಧೇಯ. ಹೆಸರು, ದೂರವಾಣಿ ಸಂಖ್ಯೆ ಅಥವಾ ಖಾತೆ ಉಳಿಸುವುದಿಲ್ಲ.',
  'report.errStop': 'ದತ್ತಾಂಶದಿಂದ ನಿಲ್ದಾಣ ಆಯ್ಕೆಮಾಡಿ.',
  'report.errLevel': 'ಜನದಟ್ಟಣೆ ಮಟ್ಟ ಆಯ್ಕೆಮಾಡಿ.',
  'report.errSave': 'ವರದಿ ಉಳಿಸಲಾಗಲಿಲ್ಲ. ಮತ್ತೆ ಪ್ರಯತ್ನಿಸಿ.',
  'report.errDuplicate': 'ಇದೇ ಮಾರ್ಗ ಮತ್ತು ನಿಲ್ದಾಣಕ್ಕೆ ಕೆಲವು ನಿಮಿಷಗಳ ಹಿಂದೆ ವರದಿ ಮಾಡಿದ್ದೀರಿ, ಹಾಗಾಗಿ ಎರಡನೇ ಬಾರಿ ಉಳಿಸಲಿಲ್ಲ.',
  'report.lastAt': 'ಈ ನಿಲ್ದಾಣಕ್ಕೆ ನಿಮ್ಮ ಕೊನೆಯ ವರದಿ {time} ಕ್ಕೆ.',
  'report.needDataset': 'ವರದಿಗೆ ಆಮದಾದ ನಿಲ್ದಾಣ ಪಟ್ಟಿ ಬೇಕು, ಆದ್ದರಿಂದ ಮೊದಲು ದತ್ತಾಂಶ ಆಮದು ಮಾಡಬೇಕು.',

  'feed.eyebrow': 'ಜನದಟ್ಟಣೆ ವರದಿಗಳು',
  'feed.title': 'ಪ್ರಯಾಣಿಕರು ಏನು ವರದಿ ಮಾಡುತ್ತಿದ್ದಾರೆ',
  'feed.description': 'ಪ್ರತಿ ಬರಹವೂ ಅದರ ಸಮಯದೊಂದಿಗೆ ತೋರಿಸುವ ಪ್ರಯಾಣಿಕರ ವರದಿ.',
  'feed.recent': 'ಇತ್ತೀಚಿನ ವರದಿಗಳು',
  'feed.older': 'ಹಳೆಯ ವರದಿಗಳು',
  'feed.stale': 'ಹಳೆಯದಾಗಿರಬಹುದು',
  'feed.staleHint': '{hours} ಗಂಟೆಗಳಿಗಿಂತ ಹಳೆಯದು',
  'feed.empty': 'ಇನ್ನೂ ವರದಿಗಳಿಲ್ಲ',
  'feed.first': 'ನಿಮ್ಮ ನಿಲ್ದಾಣ ಅಥವಾ ಬಸ್ ಎಷ್ಟು ತುಂಬಿತ್ತು ಎಂದು ಮೊದಲು ತಿಳಿಸಿ.',
  'feed.showing': '{total} ರಲ್ಲಿ {count} ತೋರಿಸಲಾಗಿದೆ',
  'feed.justNow': 'ಈಗಷ್ಟೇ',
  'feed.minutesAgo': '{count} ನಿಮಿಷ ಹಿಂದೆ',
  'feed.hoursAgo': '{count} ಗಂಟೆ ಹಿಂದೆ',
  'feed.noRoute': 'ಮಾರ್ಗ ಸಂಖ್ಯೆ ನೀಡಿಲ್ಲ',
  'feed.loading': 'ವರದಿಗಳು ಬರುತ್ತಿವೆ…',
  'feed.cta': 'ನೀವು ಕಂಡದ್ದನ್ನು ವರದಿ ಮಾಡಿ',
  'feed.live': 'ಸಲ್ಲಿಸಿದಂತೆ ಹೊಸ ವರದಿಗಳು ಇಲ್ಲಿ ಕಾಣುತ್ತವೆ. ಎಲ್ಲ ಸಮಯ ಬೆಂಗಳೂರು ಸಮಯ.',

  'search.routesTab': 'ಮಾರ್ಗಗಳು',
  'search.stopsTab': 'ನಿಲ್ದಾಣಗಳು',
  'search.routesTitle': 'BMTC ಮಾರ್ಗಗಳನ್ನು ಹುಡುಕಿ',
  'search.routesHint': 'ಮಾರ್ಗ ಸಂಖ್ಯೆ ಅಥವಾ ಮಾರ್ಗದ ತುದಿಗಳಿಂದ ಹುಡುಕಿ.',
  'search.stopsTitle': 'ಬಸ್ ನಿಲ್ದಾಣಗಳನ್ನು ಹುಡುಕಿ',
  'search.stopsHint': 'ನಿಲ್ದಾಣ ಹುಡುಕಿ ಅದಕ್ಕೆ ಸೇವೆ ಸಲ್ಲಿಸುವ ಮಾರ್ಗಗಳನ್ನು ನೋಡಿ.',
  'search.routePlaceholder': 'ಮಾರ್ಗ ಸಂಖ್ಯೆ ಅಥವಾ ಹೆಸರು, ಉದಾ. 500-D',
  'search.stopPlaceholder': 'ನಿಲ್ದಾಣದ ಹೆಸರು, ಉದಾ. ಶಾಂತಿನಗರ',
  'search.typeMore': 'ಕನಿಷ್ಠ ಎರಡು ಅಕ್ಷರ ಬರೆಯಿರಿ.',
  'search.noRoutes': 'ಆ ಹುಡುಕಾಟಕ್ಕೆ ಯಾವ ಮಾರ್ಗವೂ ಹೊಂದಿಕೆಯಾಗುತ್ತಿಲ್ಲ.',
  'search.noStops': 'ಆ ಹುಡುಕಾಟಕ್ಕೆ ಯಾವ ನಿಲ್ದಾಣವೂ ಹೊಂದಿಕೆಯಾಗುತ್ತಿಲ್ಲ.',
  'search.trips': '{count} ನಿಗದಿತ ಪ್ರಯಾಣಗಳು',
  'search.stopsCount': '{count} ನಿಲ್ದಾಣಗಳು',
  'search.servedBy': 'ಈ ನಿಲ್ದಾಣಕ್ಕೆ ಸೇವೆ ಸಲ್ಲಿಸುವ ಮಾರ್ಗಗಳು',
  'search.onRoute': 'ನಿಲ್ದಾಣ ಕ್ರಮ',
  'search.matched': 'ಆಮದಾದ ದತ್ತಾಂಶದಿಂದ',

  'insights.eyebrow': 'ಜನದಟ್ಟಣೆ ಒಳನೋಟಗಳು',
  'insights.title': 'ವರದಿಗಳ ಒಟ್ಟು ಚಿತ್ರ',
  'insights.description': 'ಈ ಆ್ಯಪ್ನಲ್ಲಿ ದಾಖಲಾದ ಪ್ರಯಾಣಿಕರ ವರದಿಗಳಿಂದ ಮಾತ್ರ ತಯಾರಾಗಿದೆ. ಯಾವುದೂ ಅಂದಾಜು ಅಲ್ಲ.',
  'insights.byLevel': 'ಜನದಟ್ಟಣೆ ಮಟ್ಟದ ಪ್ರಕಾರ ವರದಿಗಳು',
  'insights.byDay': 'ದಿನಕ್ಕೆ ವರದಿಗಳು',
  'insights.total': 'ಒಟ್ಟು ವರದಿಗಳು',
  'insights.last24': 'ಕಳೆದ 24 ಗಂಟೆಗಳಲ್ಲಿ',
  'insights.empty': 'ಇನ್ನೂ ವರದಿಗಳಿಲ್ಲ, ಹಾಗಾಗಿ ಚಾರ್ಟ್ ಇಲ್ಲ.',

  'about.eyebrow': 'SAFAR ಪರಿಚಯ',
  'about.title': 'ಇಲ್ಲಿ ಯಾವುದು ನಿಜ, ಯಾವುದು ಅಲ್ಲ',
  'about.realTitle': 'ನಿಜವಾದ ದತ್ತಾಂಶ',
  'about.realBody': 'ಮಾರ್ಗ ಸಂಖ್ಯೆ, ಮಾರ್ಗ ಹೆಸರು, ನಿಲ್ದಾಣಗಳ ಕ್ರಮ ಪಟ್ಟಿ, ನಿಲ್ದಾಣ ಹೆಸರು ಮತ್ತು ನಿಗದಿತ ನಿರ್ಗಮನ ಸಮಯಗಳು ಕೆಳಗೆ ನೀಡಿದ BMTC GTFS ದತ್ತಾಂಶದಿಂದ ಆಮದಾಗಿ ಈ ಆ್ಯಪ್ನ ಡೇಟಾಬೇಸ್ನಲ್ಲಿ ಉಳಿದಿವೆ.',
  'about.reportsTitle': 'ಪ್ರಯಾಣಿಕರ ವರದಿಗಳು',
  'about.reportsBody': 'ಜನದಟ್ಟಣೆ ಮಟ್ಟ ಪ್ರಯಾಣಿಕರ ವರದಿಗಳಿಂದ ಮಾತ್ರ ಬರುತ್ತದೆ, ಪ್ರತಿಯೊಂದಕ್ಕೂ ಅದರ ಸಮಯ ಇರುತ್ತದೆ. {hours} ಗಂಟೆಗಳಿಗಿಂತ ಹಳೆಯ ವರದಿಗಳನ್ನು ಹಳೆಯದಾಗಿರಬಹುದು ಎಂದು ಗುರುತಿಸಲಾಗುತ್ತದೆ.',
  'about.noTitle': 'ಲಭ್ಯವಿಲ್ಲ',
  'about.noBody': 'BMTC ಗೆ ನೇರ ಬಸ್ ಸ್ಥಾನ ಅಥವಾ ಪ್ರಯಾಣಿಕರ ಸಂಖ್ಯೆಯ ಅನುಮತಿಸಿದ ಸಾರ್ವಜನಿಕ ಫೀಡ್ ಇಲ್ಲ. ಆದ್ದರಿಂದ SAFAR ಎಂದಿಗೂ ನೇರ ಬಸ್ ಸ್ಥಾನ, ನೇರ ಜನದಟ್ಟಣೆ ಮಟ್ಟ ಅಥವಾ ಪ್ರಯಾಣಿಕರ ಸಂಖ್ಯೆ ತೋರಿಸುವುದಿಲ್ಲ.',
  'about.sourceTitle': 'ಮೂಲ ಮತ್ತು ಪರವಾನಗಿ',
  'about.sourceBody': 'bmtc-gtfs ಎಂಬುದು ಅಧಿಕೃತ ನಮ್ಮ BMTC ಆ್ಯಪ್ನಿಂದ ತಯಾರಾದ ಅನಧಿಕೃತ GTFS ದತ್ತಾಂಶ, GitHub ನಲ್ಲಿ Vonter ಪ್ರಕಟಿಸಿದ್ದಾರೆ. ಪರವಾನಗಿ: ODbL 1.0.',
  'about.caveatTitle': 'ಮೂಲದ ತಿಳಿದ ಮಿತಿಗಳು',
  'about.caveatBody': 'ನಮ್ಮ BMTC ಆ್ಯಪ್ನ ದತ್ತಾಂಶ ನಿಖರವಲ್ಲ ಮತ್ತು ವೇಳಾಪಟ್ಟಿ ಹಾಗೂ ನಿಲ್ದಾಣ ಸಮಯಗಳು ತಪ್ಪಾಗಿರಬಹುದು ಎಂದು ದತ್ತಾಂಶ ರಚಿಸಿದವರು ಹೇಳಿದ್ದಾರೆ. ಸಮಯಗಳನ್ನು ಸೂಚಕವಾಗಿ ಮಾತ್ರ ಪರಿಗಣಿಸಿ.',
  'about.coverageTitle': 'ವ್ಯಾಪ್ತಿ',
  'about.coverageBody': 'ಮೂಲ ಆ್ಯಪ್ನಲ್ಲಿ ನೇರ ಟ್ರ್ಯಾಕಿಂಗ್ ಕೆಲಸ ಮಾಡುವ ಮಾರ್ಗಗಳನ್ನು ಮಾತ್ರ ದತ್ತಾಂಶದಲ್ಲಿ ಸೇರಿಸಲಾಗಿದೆ.',
  'about.openSource': 'ಮೂಲ ದತ್ತಾಂಶ ತೆರೆಯಿರಿ',
  'about.snapshot': 'ಮೂಲ ಕಮಿಟ್ {date} · ಪರಿಶೀಲಿಸಿದ್ದು {checked}',

  'footer.blurb': 'SAFAR ಬೆಂಗಳೂರಿನ ಸ್ವತಂತ್ರ ಪ್ರಯಾಣಿಕ ಯೋಜನೆ. ಇದು BMTC ಅಥವಾ ಕರ್ನಾಟಕ ಸರ್ಕಾರದೊಂದಿಗೆ ಸಂಬಂಧ ಹೊಂದಿಲ್ಲ.',
  'footer.noLive': 'ನೇರ ಬಸ್ ಟ್ರ್ಯಾಕಿಂಗ್ ತೋರಿಸುವುದಿಲ್ಲ.',
  'footer.how': 'ಮಾರ್ಗ ಮತ್ತು ನಿಲ್ದಾಣ ದತ್ತಾಂಶ: BMTC GTFS ದತ್ತಾಂಶ (ODbL). ಜನದಟ್ಟಣೆ: ಪ್ರಯಾಣಿಕರ ವರದಿಗಳು ಮಾತ್ರ.',
};

const te = {
  'nav.home': 'హోమ్',
  'nav.plan': 'ప్రయాణ ప్రణాళిక',
  'nav.report': 'రద్దీ నివేదన',
  'nav.insights': 'అంతర్దృష్టులు',
  'nav.about': 'గురించి',
  'nav.menu': 'మెనూ',
  'nav.changeLanguage': 'భాష మార్చండి',

  'status.sample': 'BMTC డేటా నమూనా భాగం — పూర్తి డేటా ఇంకా దిగుమతి కాలేదు',
  'status.empty': 'BMTC డేటా ఇంకా లోడ్ కాలేదు',
  'status.importing': 'నిజమైన BMTC డేటా లోడ్ అవుతోంది',
  'status.real': 'నిజమైన BMTC డేటా',
  'status.failed': 'డేటా లోడ్ కాలేదు',
  'status.routes': 'మార్గ దిశలు',
  'status.stops': 'బస్ ఆగే స్థలాలు',
  'status.reports': 'ప్రయాణికుల నివేదనలు',
  'status.sources': 'డేటా మూలాలు',
  'status.noLive': 'ప్రత్యక్ష బస్ స్థానాలు లేవు',
  'status.importCta': 'నిజమైన BMTC డేటా లోడ్ చేయండి',
  'status.importProgress': '{total} లో {done} {unit} లోడ్ అయ్యాయి',
  'status.importDone': 'ప్రచురిత మూలం నుండి లోడ్ అయింది',
  'status.checking': 'మూలాన్ని పరిశీలిస్తోంది…',
  'status.refresh': 'మూలాన్ని మళ్లీ పరిశీలించండి',
  'status.needImport': 'మార్గ శోధన, ఆగే స్థల శోధన, ప్రణాళికకు దిగుమతి చేసిన డేటా అవసరం.',

  'home.eyebrow': 'బెంగళూరు నగర బస్సులు',
  'home.title': 'బయలుదేరే ముందు మీ బస్సులో ఎంత రద్దీ ఉందో తెలుసుకోండి.',
  'home.subtitle': 'SAFAR నిజమైన BMTC మార్గ, ఆగే స్థలం, సమయపట్టిక డేటాను తోటి ప్రయాణికుల రద్దీ నివేదనలతో కలుపుతుంది.',

  'plan.eyebrow': 'ప్రయాణ ప్రణాళిక',
  'plan.title': 'ప్రయాణం ప్రణాళిక చేయండి',
  'plan.description': 'దిగుమతి చేసిన BMTC ఆగే స్థలాల జాబితా నుండి బయలుదేరే మరియు చేరే స్థలాలను ఎంచుకోండి.',
  'plan.from': 'ఎక్కడి నుండి',
  'plan.to': 'ఎక్కడికి',
  'plan.chooseFrom': 'బయలుదేరే స్థలం ఎంచుకోండి',
  'plan.chooseTo': 'చేరే స్థలం ఎంచుకోండి',
  'plan.swap': 'స్థలాలు మార్చండి',
  'plan.search': 'మార్గాలు వెతకండి',
  'plan.searching': 'మార్గాలు వెతుకుతోంది…',
  'plan.results': 'నేరు మార్గాలు',
  'plan.count': 'రెండు స్థలాలకు {count} మార్గాలు సేవ ఇస్తాయి',
  'plan.none': 'ఈ రెండు స్థలాలకు నేరుగా సేవ ఇచ్చే మార్గం డేటాలో లేదు.',
  'plan.stopsBetween': 'మధ్యలో {count} ఆగే స్థలాలు',
  'plan.boardAt': '{stop} వద్ద ఎక్కండి',
  'plan.alightAt': '{stop} వద్ద దిగండి',
  'plan.departures': 'ప్రారంభ స్థలం నుండి బయలుదేరే సమయాలు',
  'plan.firstLast': 'మొదటి {first} · చివరి {last}',
  'plan.note': 'సమయాలు డేటాలో ఉన్నట్లుగా మార్గం ప్రారంభ స్థలం నుండి బయలుదేరే నిర్ణీత సమయాలు. ఆగే స్థల స్థాయి సమయాలు డేటాలో లేవు.',
  'plan.needBoth': 'ఏ మార్గాలు కలుపుతాయో చూడటానికి రెండు స్థలాలను ఎంచుకోండి.',
  'plan.minChars': 'నిజమైన స్థలాల పేర్లు వెతకడానికి కనీసం రెండు అక్షరాలు టైప్ చేయండి.',

  'report.eyebrow': 'ప్రయాణికుల నివేదనలు',
  'report.title': 'ఎంత రద్దీ ఉందో నివేదించండి',
  'report.description': 'SAFAR లో కనిపించేది అంతా ప్రయాణికుల నుండే వస్తుంది. ఏదీ సృష్టించబడదు.',
  'report.route': 'బస్ మార్గ సంఖ్య',
  'report.routeHint': 'ఐచ్ఛికం. డేటా నుండి మార్గం ఎంచుకోండి లేదా చూసిన సంఖ్యను టైప్ చేయండి.',
  'report.routePlaceholder': 'ఉదా. 500-D',
  'report.trip': 'బస్ లేదా ట్రిప్ సంఖ్య (ఐచ్ఛికం)',
  'report.tripPlaceholder': 'ఉదా. KA-57-F-1234',
  'report.stop': 'ఆగే స్థలం',
  'report.stopHint': 'దిగుమతి చేసిన డేటా స్థలాలను మాత్రమే ఎంచుకోవచ్చు.',
  'report.level': 'ఎంత రద్దీ ఉంది?',
  'report.low': 'తక్కువ',
  'report.moderate': 'మధ్యస్థం',
  'report.high': 'ఎక్కువ',
  'report.full': 'నిండింది',
  'report.recency': 'ఇది ఎప్పుడు చూశారు?',
  'report.now': 'ఇప్పుడే',
  'report.min15': 'గత 15 నిమిషాల్లో',
  'report.min60': 'గత ఒక గంటలో',
  'report.submit': 'నివేదన పంపండి',
  'report.saving': 'భద్రపరుస్తోంది…',
  'report.saved': 'నివేదన భద్రమైంది',
  'report.savedHelp': 'ఇప్పుడు అది ఇతర ప్రయాణికులకు కనిపిస్తుంది.',
  'report.another': 'మరో నివేదన పంపండి',
  'report.privacy': 'నివేదనలు అనామకం. పేరు, ఫోన్ నంబర్ లేదా ఖాతా నిల్వ చేయబడదు.',
  'report.errStop': 'డేటా నుండి ఒక స్థలం ఎంచుకోండి.',
  'report.errLevel': 'రద్దీ స్థాయి ఎంచుకోండి.',
  'report.errSave': 'నివేదన భద్రపరచలేకపోయాం. మళ్లీ ప్రయత్నించండి.',
  'report.errDuplicate': 'ఇదే మార్గం, స్థలం కోసం కొన్ని నిమిషాల క్రితం నివేదించారు, కాబట్టి రెండోసారి నిల్వ చేయలేదు.',
  'report.lastAt': 'ఈ స్థలానికి మీ చివరి నివేదన {time} కు.',
  'report.needDataset': 'నివేదనకు దిగుమతి చేసిన స్థలాల జాబితా కావాలి, కాబట్టి ముందుగా డేటా దిగుమతి చేయాలి.',

  'feed.eyebrow': 'రద్దీ నివేదనలు',
  'feed.title': 'ప్రయాణికులు ఏమి నివేదిస్తున్నారు',
  'feed.description': 'ఇక్కడ ఉన్న ప్రతి అంశం దాని సమయంతో చూపిన ప్రయాణికుల నివేదన.',
  'feed.recent': 'ఇటీవలి నివేదనలు',
  'feed.older': 'పాత నివేదనలు',
  'feed.stale': 'పాతది కావచ్చు',
  'feed.staleHint': '{hours} గంటల కంటే పాతది',
  'feed.empty': 'ఇంకా నివేదనలు లేవు',
  'feed.first': 'మీ స్థలం లేదా బస్సు ఎంత నిండిందో మొదట చెప్పండి.',
  'feed.showing': '{total} లో {count} చూపుతోంది',
  'feed.justNow': 'ఇప్పుడే',
  'feed.minutesAgo': '{count} నిమిషాల క్రితం',
  'feed.hoursAgo': '{count} గంటల క్రితం',
  'feed.noRoute': 'మార్గ సంఖ్య ఇవ్వలేదు',
  'feed.loading': 'నివేదనలు వస్తున్నాయి…',
  'feed.cta': 'మీరు చూసినది నివేదించండి',
  'feed.live': 'పంపిన వెంటనే కొత్త నివేదనలు ఇక్కడ కనిపిస్తాయి. సమయాలు బెంగళూరు సమయం.',

  'search.routesTab': 'మార్గాలు',
  'search.stopsTab': 'ఆగే స్థలాలు',
  'search.routesTitle': 'BMTC మార్గాలు వెతకండి',
  'search.routesHint': 'మార్గ సంఖ్య లేదా మార్గం చివరల పేరుతో వెతకండి.',
  'search.stopsTitle': 'బస్ ఆగే స్థలాలు వెతకండి',
  'search.stopsHint': 'ఒక స్థలం వెతికి దానికి సేవ ఇచ్చే మార్గాలు చూడండి.',
  'search.routePlaceholder': 'మార్గ సంఖ్య లేదా పేరు, ఉదా. 500-D',
  'search.stopPlaceholder': 'స్థలం పేరు, ఉదా. శాంతినగర',
  'search.typeMore': 'కనీసం రెండు అక్షరాలు టైప్ చేయండి.',
  'search.noRoutes': 'ఆ శోధనకు ఏ మార్గం సరిపోలడం లేదు.',
  'search.noStops': 'ఆ శోధనకు ఏ స్థలం సరిపోలడం లేదు.',
  'search.trips': '{count} నిర్ణీత ప్రయాణాలు',
  'search.stopsCount': '{count} ఆగే స్థలాలు',
  'search.servedBy': 'ఈ స్థలానికి సేవ ఇచ్చే మార్గాలు',
  'search.onRoute': 'స్థలాల క్రమం',
  'search.matched': 'దిగుమతి చేసిన డేటా నుండి',

  'insights.eyebrow': 'రద్దీ అంతర్దృష్టులు',
  'insights.title': 'నివేదనల మొత్తం చిత్రం',
  'insights.description': 'ఈ యాప్లో నమోదైన ప్రయాణికుల నివేదనల నుండి మాత్రమే తయారు చేయబడింది. ఏదీ అంచనా కాదు.',
  'insights.byLevel': 'రద్దీ స్థాయి ప్రకారం నివేదనలు',
  'insights.byDay': 'రోజుకు నివేదనలు',
  'insights.total': 'మొత్తం నివేదనలు',
  'insights.last24': 'గత 24 గంటల్లో',
  'insights.empty': 'ఇంకా నివేదనలు లేవు, కాబట్టి చార్ట్ లేదు.',

  'about.eyebrow': 'SAFAR గురించి',
  'about.title': 'ఇక్కడ ఏది నిజం, ఏది కాదు',
  'about.realTitle': 'నిజమైన డేటా',
  'about.realBody': 'మార్గ సంఖ్యలు, మార్గ పేర్లు, క్రమమైన ఆగే స్థలాల జాబితా, స్థలాల పేర్లు మరియు నిర్ణీత బయలుదేరే సమయాలు క్రింద ఇచ్చిన BMTC GTFS డేటా నుండి దిగుమతి అయ్యి ఈ యాప్ డేటాబేస్లో ఉన్నాయి.',
  'about.reportsTitle': 'ప్రయాణికుల నివేదనలు',
  'about.reportsBody': 'రద్దీ స్థాయులు ప్రయాణికుల నివేదనల నుండి మాత్రమే వస్తాయి, ప్రతిదానికి దాని సమయం ఉంటుంది. {hours} గంటల కంటే పాత నివేదనలను పాతవి కావచ్చు అని గుర్తిస్తారు.',
  'about.noTitle': 'అందుబాటులో లేదు',
  'about.noBody': 'BMTC కోసం ప్రత్యక్ష బస్ స్థానాలు లేదా ప్రయాణికుల సంఖ్యకు అనుమతించిన పబ్లిక్ ఫీడ్ లేదు. కాబట్టి SAFAR ఎప్పుడూ ప్రత్యక్ష బస్ స్థానం, ప్రత్యక్ష రద్దీ స్థాయి లేదా ప్రయాణికుల సంఖ్య చూపదు.',
  'about.sourceTitle': 'మూలం మరియు లైసెన్స్',
  'about.sourceBody': 'bmtc-gtfs అనేది అధికారిక నమ్మ BMTC యాప్ నుండి తయారైన అనధికారిక GTFS డేటా, GitHub లో Vonter ప్రచురించారు. లైసెన్స్: ODbL 1.0.',
  'about.caveatTitle': 'మూలం తెలిసిన పరిమితులు',
  'about.caveatBody': 'నమ్మ BMTC యాప్ డేటా ఖచ్చితం కాదని, సమయపట్టికలు, ఆగే స్థల సమయాలు తప్పుగా ఉండవచ్చని డేటా రూపకర్తలు చెప్పారు. సమయాలను సూచనగా మాత్రమే తీసుకోండి.',
  'about.coverageTitle': 'పరిధి',
  'about.coverageBody': 'మూల యాప్లో ప్రత్యక్ష ట్రాకింగ్ పనిచేసే మార్గాలు మాత్రమే డేటాలో చేర్చబడ్డాయి.',
  'about.openSource': 'మూల డేటా తెరవండి',
  'about.snapshot': 'మూల కమిట్ {date} · పరిశీలించినది {checked}',

  'footer.blurb': 'SAFAR బెంగళూరు కోసం స్వతంత్ర ప్రయాణికుల ప్రాజెక్ట్. ఇది BMTC లేదా కర్ణాటక ప్రభుత్వంతో సంబంధం లేదు.',
  'footer.noLive': 'ప్రత్యక్ష బస్ ట్రాకింగ్ చూపబడదు.',
  'footer.how': 'మార్గం, ఆగే స్థల డేటా: BMTC GTFS డేటా (ODbL). రద్దీ: ప్రయాణికుల నివేదనలు మాత్రమే.',
};

const ta = {
  'nav.home': 'முகப்பு',
  'nav.plan': 'பயண திட்டம்',
  'nav.report': 'கூட்ட அறிக்கை',
  'nav.insights': 'நுண்ணறிவு',
  'nav.about': 'பற்றி',
  'nav.menu': 'பட்டியல்',
  'nav.changeLanguage': 'மொழியை மாற்று',

  'status.sample': 'BMTC தரவின் மாதிரிப் பகுதி — முழுத் தரவு இன்னும் இறக்குமதி ஆகவில்லை',
  'status.empty': 'BMTC தரவு இன்னும் ஏற்றப்படவில்லை',
  'status.importing': 'உண்மையான BMTC தரவு ஏற்றப்படுகிறது',
  'status.real': 'உண்மையான BMTC தரவு',
  'status.failed': 'தரவை ஏற்ற முடியவில்லை',
  'status.routes': 'வழித்தட திசைகள்',
  'status.stops': 'பேருந்து நிறுத்தங்கள்',
  'status.reports': 'பயணி அறிக்கைகள்',
  'status.sources': 'தரவு மூலங்கள்',
  'status.noLive': 'நேரடி பேருந்து இருப்பிடம் இல்லை',
  'status.importCta': 'உண்மையான BMTC தரவை ஏற்று',
  'status.importProgress': '{total} இல் {done} {unit} ஏற்றப்பட்டது',
  'status.importDone': 'வெளியிடப்பட்ட மூலத்திலிருந்து ஏற்றப்பட்டது',
  'status.checking': 'மூலத்தை சரிபார்க்கிறது…',
  'status.refresh': 'மூலத்தை மீண்டும் சரிபார்',
  'status.needImport': 'வழித்தட தேடல், நிறுத்தத் தேடல், திட்டமிடலுக்கு இறக்குமதி செய்த தரவு தேவை.',

  'home.eyebrow': 'பெங்களூரு நகரப் பேருந்துகள்',
  'home.title': 'புறப்படும் முன் உங்கள் பேருந்தில் எவ்வளவு கூட்டம் என்று அறியுங்கள்.',
  'home.subtitle': 'SAFAR உண்மையான BMTC வழித்தட, நிறுத்தம், நேர அட்டவணை தரவை சக பயணிகளின் கூட்ட அறிக்கைகளுடன் இணைக்கிறது.',

  'plan.eyebrow': 'பயண திட்டம்',
  'plan.title': 'பயணத்தைத் திட்டமிடுங்கள்',
  'plan.description': 'இறக்குமதி செய்த BMTC நிறுத்தப் பட்டியலில் இருந்து புறப்பிடம், சேரிடம் தேர்ந்தெடுங்கள்.',
  'plan.from': 'எங்கிருந்து',
  'plan.to': 'எங்கே',
  'plan.chooseFrom': 'புறப்பிட நிறுத்தத்தைத் தேர்ந்தெடு',
  'plan.chooseTo': 'சேரிட நிறுத்தத்தைத் தேர்ந்தெடு',
  'plan.swap': 'நிறுத்தங்களை மாற்று',
  'plan.search': 'வழித்தடங்களைத் தேடு',
  'plan.searching': 'வழித்தடங்களைத் தேடுகிறது…',
  'plan.results': 'நேரடி வழித்தடங்கள்',
  'plan.count': 'இரு நிறுத்தங்களுக்கும் {count} வழித்தடங்கள் சேவை செய்கின்றன',
  'plan.none': 'இந்த இரு நிறுத்தங்களுக்கும் நேரடிச் சேவை தரும் வழித்தடம் தரவில் இல்லை.',
  'plan.stopsBetween': 'இடையே {count} நிறுத்தங்கள்',
  'plan.boardAt': '{stop} இல் ஏறுங்கள்',
  'plan.alightAt': '{stop} இல் இறங்குங்கள்',
  'plan.departures': 'தொடக்க நிறுத்தத்தின் புறப்பாடுகள்',
  'plan.firstLast': 'முதல் {first} · கடைசி {last}',
  'plan.note': 'நேரங்கள் தரவில் உள்ளபடி வழித்தடத்தின் தொடக்க நிறுத்தத்திலிருந்து திட்டமிட்ட புறப்பாடுகள். நிறுத்த அளவு நேரங்கள் தரவில் இல்லை.',
  'plan.needBoth': 'எந்த வழித்தடங்கள் இணைக்கின்றன என்பதைப் பார்க்க இரு நிறுத்தங்களையும் தேர்ந்தெடுங்கள்.',
  'plan.minChars': 'உண்மையான நிறுத்தப் பெயர்களைத் தேட குறைந்தது இரண்டு எழுத்துகள் தட்டச்சு செய்யுங்கள்.',

  'report.eyebrow': 'பயணி அறிக்கைகள்',
  'report.title': 'எவ்வளவு கூட்டம் என்று அறிக்கை செய்யுங்கள்',
  'report.description': 'SAFAR இல் தெரிவது அனைத்தும் பயணிகளிடமிருந்தே வருகிறது. எதுவும் உருவாக்கப்படுவதில்லை.',
  'report.route': 'பேருந்து வழித்தட எண்',
  'report.routeHint': 'விருப்பம். தரவில் இருந்து வழித்தடம் தேர்ந்தெடுங்கள் அல்லது பார்த்த எண்ணைத் தட்டச்சு செய்யுங்கள்.',
  'report.routePlaceholder': 'எ.கா. 500-D',
  'report.trip': 'பேருந்து அல்லது பயண எண் (விருப்பம்)',
  'report.tripPlaceholder': 'எ.கா. KA-57-F-1234',
  'report.stop': 'நிறுத்தம்',
  'report.stopHint': 'இறக்குமதி செய்த தரவின் நிறுத்தங்களை மட்டுமே தேர்ந்தெடுக்கலாம்.',
  'report.level': 'எவ்வளவு கூட்டம்?',
  'report.low': 'குறைவு',
  'report.moderate': 'மிதமானது',
  'report.high': 'அதிகம்',
  'report.full': 'நிரம்பியது',
  'report.recency': 'இதை எப்போது பார்த்தீர்கள்?',
  'report.now': 'இப்போதுதான்',
  'report.min15': 'கடந்த 15 நிமிடங்களில்',
  'report.min60': 'கடந்த ஒரு மணி நேரத்தில்',
  'report.submit': 'அறிக்கையை அனுப்பு',
  'report.saving': 'சேமிக்கிறது…',
  'report.saved': 'அறிக்கை சேமிக்கப்பட்டது',
  'report.savedHelp': 'இப்போது அது மற்ற பயணிகளுக்குத் தெரியும்.',
  'report.another': 'மற்றொரு அறிக்கை அனுப்பு',
  'report.privacy': 'அறிக்கைகள் அநாமதேயம். பெயர், தொலைபேசி எண் அல்லது கணக்கு சேமிக்கப்படுவதில்லை.',
  'report.errStop': 'தரவில் இருந்து ஒரு நிறுத்தத்தைத் தேர்ந்தெடுங்கள்.',
  'report.errLevel': 'கூட்ட அளவைத் தேர்ந்தெடுங்கள்.',
  'report.errSave': 'அறிக்கையைச் சேமிக்க முடியவில்லை. மீண்டும் முயற்சிக்கவும்.',
  'report.errDuplicate': 'இதே வழித்தடம், நிறுத்தத்திற்கு சில நிமிடங்களுக்கு முன் அறிக்கை அளித்தீர்கள், எனவே இரண்டாவது முறை சேமிக்கப்படவில்லை.',
  'report.lastAt': 'இந்த நிறுத்தத்திற்கு உங்கள் கடைசி அறிக்கை {time} க்கு.',
  'report.needDataset': 'அறிக்கைக்கு இறக்குமதி செய்த நிறுத்தப் பட்டியல் தேவை, எனவே முதலில் தரவை இறக்குமதி செய்ய வேண்டும்.',

  'feed.eyebrow': 'கூட்ட அறிக்கைகள்',
  'feed.title': 'பயணிகள் என்ன அறிக்கை செய்கிறார்கள்',
  'feed.description': 'இங்குள்ள ஒவ்வொன்றும் அதன் நேரத்துடன் காட்டப்படும் பயணி அறிக்கை.',
  'feed.recent': 'சமீபத்திய அறிக்கைகள்',
  'feed.older': 'பழைய அறிக்கைகள்',
  'feed.stale': 'பழையதாக இருக்கலாம்',
  'feed.staleHint': '{hours} மணி நேரத்திற்கு மேற்பட்டது',
  'feed.empty': 'இன்னும் அறிக்கைகள் இல்லை',
  'feed.first': 'உங்கள் நிறுத்தம் அல்லது பேருந்து எவ்வளவு நிரம்பியது என்பதை முதலில் சொல்லுங்கள்.',
  'feed.showing': '{total} இல் {count} காட்டப்படுகிறது',
  'feed.justNow': 'இப்போதுதான்',
  'feed.minutesAgo': '{count} நிமிடம் முன்',
  'feed.hoursAgo': '{count} மணி முன்',
  'feed.noRoute': 'வழித்தட எண் தரப்படவில்லை',
  'feed.loading': 'அறிக்கைகள் வருகின்றன…',
  'feed.cta': 'நீங்கள் பார்த்ததை அறிக்கை செய்யுங்கள்',
  'feed.live': 'அனுப்பியவுடன் புதிய அறிக்கைகள் இங்கே தோன்றும். நேரங்கள் பெங்களூரு நேரம்.',

  'search.routesTab': 'வழித்தடங்கள்',
  'search.stopsTab': 'நிறுத்தங்கள்',
  'search.routesTitle': 'BMTC வழித்தடங்களைத் தேடு',
  'search.routesHint': 'வழித்தட எண் அல்லது வழித்தடத்தின் இரு முனைகளால் தேடுங்கள்.',
  'search.stopsTitle': 'பேருந்து நிறுத்தங்களைத் தேடு',
  'search.stopsHint': 'ஒரு நிறுத்தத்தைத் தேடி அதற்கு சேவை செய்யும் வழித்தடங்களைப் பாருங்கள்.',
  'search.routePlaceholder': 'வழித்தட எண் அல்லது பெயர், எ.கா. 500-D',
  'search.stopPlaceholder': 'நிறுத்தப் பெயர், எ.கா. சாந்திநகர்',
  'search.typeMore': 'குறைந்தது இரண்டு எழுத்துகள் தட்டச்சு செய்யுங்கள்.',
  'search.noRoutes': 'அந்தத் தேடலுக்கு எந்த வழித்தடமும் பொருந்தவில்லை.',
  'search.noStops': 'அந்தத் தேடலுக்கு எந்த நிறுத்தமும் பொருந்தவில்லை.',
  'search.trips': '{count} திட்டமிட்ட பயணங்கள்',
  'search.stopsCount': '{count} நிறுத்தங்கள்',
  'search.servedBy': 'இந்த நிறுத்தத்திற்கு சேவை செய்யும் வழித்தடங்கள்',
  'search.onRoute': 'நிறுத்த வரிசை',
  'search.matched': 'இறக்குமதி செய்த தரவில் இருந்து',

  'insights.eyebrow': 'கூட்ட நுண்ணறிவு',
  'insights.title': 'அறிக்கைகளின் மொத்தப் படம்',
  'insights.description': 'இந்த செயலியில் பதிவான பயணி அறிக்கைகளில் இருந்து மட்டுமே உருவாக்கப்பட்டது. எதுவும் மதிப்பீடு அல்ல.',
  'insights.byLevel': 'கூட்ட அளவு வாரியாக அறிக்கைகள்',
  'insights.byDay': 'நாளொன்றுக்கு அறிக்கைகள்',
  'insights.total': 'மொத்த அறிக்கைகள்',
  'insights.last24': 'கடந்த 24 மணி நேரத்தில்',
  'insights.empty': 'இன்னும் அறிக்கைகள் இல்லை, எனவே வரைபடம் இல்லை.',

  'about.eyebrow': 'SAFAR பற்றி',
  'about.title': 'இங்கே எது உண்மை, எது இல்லை',
  'about.realTitle': 'உண்மையான தரவு',
  'about.realBody': 'வழித்தட எண்கள், வழித்தட பெயர்கள், வரிசைப்படுத்தப்பட்ட நிறுத்தப் பட்டியல், நிறுத்தப் பெயர்கள் மற்றும் திட்டமிட்ட புறப்பாட்டு நேரங்கள் கீழே உள்ள BMTC GTFS தரவிலிருந்து இறக்குமதி ஆகி இந்த செயலியின் தரவுத்தளத்தில் உள்ளன.',
  'about.reportsTitle': 'பயணி அறிக்கைகள்',
  'about.reportsBody': 'கூட்ட அளவுகள் பயணிகளின் அறிக்கைகளில் இருந்து மட்டுமே வருகின்றன, ஒவ்வொன்றுக்கும் அதன் நேரம் உண்டு. {hours} மணி நேரத்திற்கு மேற்பட்ட அறிக்கைகள் பழையதாக இருக்கலாம் எனக் குறிக்கப்படும்.',
  'about.noTitle': 'கிடைக்கவில்லை',
  'about.noBody': 'BMTC க்கு நேரடி பேருந்து இருப்பிடம் அல்லது பயணி எண்ணிக்கைக்கு அனுமதிக்கப்பட்ட பொது ஊட்டம் இல்லை. எனவே SAFAR எப்போதும் நேரடி பேருந்து இருப்பிடம், நேரடி கூட்ட அளவு அல்லது பயணி எண்ணிக்கையைக் காட்டாது.',
  'about.sourceTitle': 'மூலமும் உரிமமும்',
  'about.sourceBody': 'bmtc-gtfs என்பது அதிகாரப்பூர்வ நம்ம BMTC செயலியிலிருந்து உருவான அதிகாரப்பூர்வமற்ற GTFS தரவு, GitHub இல் Vonter வெளியிட்டார். உரிமம்: ODbL 1.0.',
  'about.caveatTitle': 'மூலத்தின் அறியப்பட்ட வரம்புகள்',
  'about.caveatBody': 'நம்ம BMTC செயலி தரவு துல்லியமற்றது, நேர அட்டவணைகள், நிறுத்த நேரங்கள் தவறாக இருக்கலாம் என தரவு உருவாக்குநர்கள் கூறுகிறார்கள். நேரங்களை குறிப்பாக மட்டுமே கருதுங்கள்.',
  'about.coverageTitle': 'பரப்பு',
  'about.coverageBody': 'மூல செயலியில் நேரடி கண்காணிப்பு செயல்படும் வழித்தடங்கள் மட்டுமே தரவில் சேர்க்கப்பட்டுள்ளன.',
  'about.openSource': 'மூல தரவைத் திற',
  'about.snapshot': 'மூல கமிட் {date} · சரிபார்த்தது {checked}',

  'footer.blurb': 'SAFAR பெங்களூருவுக்கான சுயாதீன பயணி திட்டம். இது BMTC அல்லது கர்நாடக அரசுடன் தொடர்புடையது அல்ல.',
  'footer.noLive': 'நேரடி பேருந்து கண்காணிப்பு காட்டப்படுவதில்லை.',
  'footer.how': 'வழித்தட, நிறுத்த தரவு: BMTC GTFS தரவு (ODbL). கூட்ட அளவு: பயணி அறிக்கைகள் மட்டுமே.',
};

const hi = {
  'nav.home': 'होम',
  'nav.plan': 'यात्रा योजना',
  'nav.report': 'भीड़ रिपोर्ट',
  'nav.insights': 'जानकारी',
  'nav.about': 'परिचय',
  'nav.menu': 'मेन्यू',
  'nav.changeLanguage': 'भाषा बदलें',

  'status.sample': 'BMTC डेटा का नमूना भाग — पूरा डेटा अभी आयात नहीं हुआ',
  'status.empty': 'BMTC डेटा अभी लोड नहीं हुआ',
  'status.importing': 'असली BMTC डेटा लोड हो रहा है',
  'status.real': 'असली BMTC डेटा',
  'status.failed': 'डेटा लोड नहीं हो सका',
  'status.routes': 'मार्ग दिशाएँ',
  'status.stops': 'बस स्टॉप',
  'status.reports': 'यात्री रिपोर्ट',
  'status.sources': 'डेटा स्रोत',
  'status.noLive': 'लाइव बस स्थान नहीं',
  'status.importCta': 'असली BMTC डेटा लोड करें',
  'status.importProgress': '{total} में से {done} {unit} लोड हुए',
  'status.importDone': 'प्रकाशित स्रोत से लोड हुआ',
  'status.checking': 'स्रोत जाँचा जा रहा है…',
  'status.refresh': 'स्रोत फिर से जाँचें',
  'status.needImport': 'मार्ग खोज, स्टॉप खोज और योजना के लिए आयातित डेटा ज़रूरी है।',

  'home.eyebrow': 'बेंगलुरु सिटी बसें',
  'home.title': 'निकलने से पहले जानें आपकी बस में कितनी भीड़ है।',
  'home.subtitle': 'SAFAR असली BMTC मार्ग, स्टॉप और समय-सारणी डेटा को साथी यात्रियों की भीड़ रिपोर्ट के साथ जोड़ता है।',

  'plan.eyebrow': 'यात्रा योजना',
  'plan.title': 'यात्रा की योजना बनाएं',
  'plan.description': 'आयातित BMTC स्टॉप सूची से शुरुआती और गंतव्य स्टॉप चुनें।',
  'plan.from': 'कहाँ से',
  'plan.to': 'कहाँ तक',
  'plan.chooseFrom': 'शुरुआती स्टॉप चुनें',
  'plan.chooseTo': 'गंतव्य स्टॉप चुनें',
  'plan.swap': 'स्टॉप बदलें',
  'plan.search': 'मार्ग खोजें',
  'plan.searching': 'मार्ग खोजे जा रहे हैं…',
  'plan.results': 'सीधे मार्ग',
  'plan.count': '{count} मार्ग दोनों स्टॉप पर सेवा देते हैं',
  'plan.none': 'इन दोनों स्टॉप के लिए सीधा मार्ग डेटा में नहीं है।',
  'plan.stopsBetween': 'बीच में {count} स्टॉप',
  'plan.boardAt': '{stop} पर चढ़ें',
  'plan.alightAt': '{stop} पर उतरें',
  'plan.departures': 'शुरुआती स्टॉप से प्रस्थान',
  'plan.firstLast': 'पहला {first} · आखिरी {last}',
  'plan.note': 'समय डेटा के अनुसार मार्ग के शुरुआती स्टॉप से निर्धारित प्रस्थान समय हैं। स्टॉप-स्तर के समय डेटा में नहीं हैं।',
  'plan.needBoth': 'देखने के लिए कि कौन से मार्ग जोड़ते हैं, दोनों स्टॉप चुनें।',
  'plan.minChars': 'असली स्टॉप नाम खोजने के लिए कम से कम दो अक्षर लिखें।',

  'report.eyebrow': 'यात्री रिपोर्ट',
  'report.title': 'भीड़ कितनी है, रिपोर्ट करें',
  'report.description': 'SAFAR पर दिखने वाला सब कुछ यात्रियों से आता है। कुछ भी बनाया नहीं जाता।',
  'report.route': 'बस मार्ग संख्या',
  'report.routeHint': 'वैकल्पिक। डेटा से मार्ग चुनें या देखी गई संख्या लिखें।',
  'report.routePlaceholder': 'जैसे 500-D',
  'report.trip': 'बस या ट्रिप संख्या (वैकल्पिक)',
  'report.tripPlaceholder': 'जैसे KA-57-F-1234',
  'report.stop': 'स्टॉप',
  'report.stopHint': 'केवल आयातित डेटा के स्टॉप चुने जा सकते हैं।',
  'report.level': 'भीड़ कितनी है?',
  'report.low': 'कम',
  'report.moderate': 'मध्यम',
  'report.high': 'अधिक',
  'report.full': 'भरा हुआ',
  'report.recency': 'यह कब देखा?',
  'report.now': 'अभी-अभी',
  'report.min15': 'पिछले 15 मिनट में',
  'report.min60': 'पिछले एक घंटे में',
  'report.submit': 'रिपोर्ट भेजें',
  'report.saving': 'सहेजा जा रहा है…',
  'report.saved': 'रिपोर्ट सहेजी गई',
  'report.savedHelp': 'अब यह दूसरे यात्रियों को दिखती है।',
  'report.another': 'एक और रिपोर्ट भेजें',
  'report.privacy': 'रिपोर्ट गुमनाम हैं। नाम, फ़ोन नंबर या खाता सहेजा नहीं जाता।',
  'report.errStop': 'डेटा से एक स्टॉप चुनें।',
  'report.errLevel': 'भीड़ स्तर चुनें।',
  'report.errSave': 'रिपोर्ट सहेजी नहीं जा सकी। फिर कोशिश करें।',
  'report.errDuplicate': 'इसी मार्ग और स्टॉप के लिए कुछ मिनट पहले रिपोर्ट कर चुके हैं, इसलिए दोबारा सहेजा नहीं गया।',
  'report.lastAt': 'इस स्टॉप के लिए आपकी पिछली रिपोर्ट {time} पर थी।',
  'report.needDataset': 'रिपोर्ट के लिए आयातित स्टॉप सूची चाहिए, इसलिए पहले डेटा आयात करें।',

  'feed.eyebrow': 'भीड़ रिपोर्ट',
  'feed.title': 'यात्री क्या रिपोर्ट कर रहे हैं',
  'feed.description': 'यहाँ हर प्रविष्टि अपने समय के साथ दिखाई गई यात्री रिपोर्ट है।',
  'feed.recent': 'हाल की रिपोर्ट',
  'feed.older': 'पुरानी रिपोर्ट',
  'feed.stale': 'पुरानी हो सकती है',
  'feed.staleHint': '{hours} घंटे से पुरानी',
  'feed.empty': 'अभी कोई रिपोर्ट नहीं',
  'feed.first': 'आपका स्टॉप या बस कितनी भरी थी, यह सबसे पहले बताएं।',
  'feed.showing': '{total} में से {count} दिखाई गई',
  'feed.justNow': 'अभी-अभी',
  'feed.minutesAgo': '{count} मिनट पहले',
  'feed.hoursAgo': '{count} घंटे पहले',
  'feed.noRoute': 'मार्ग संख्या नहीं दी',
  'feed.loading': 'रिपोर्ट आ रही हैं…',
  'feed.cta': 'जो देखा वह रिपोर्ट करें',
  'feed.live': 'भेजते ही नई रिपोर्ट यहाँ दिखती हैं। समय बेंगलुरु समय है।',

  'search.routesTab': 'मार्ग',
  'search.stopsTab': 'स्टॉप',
  'search.routesTitle': 'BMTC मार्ग खोजें',
  'search.routesHint': 'मार्ग संख्या या मार्ग के दोनों सिरों से खोजें।',
  'search.stopsTitle': 'बस स्टॉप खोजें',
  'search.stopsHint': 'स्टॉप खोजें और देखें कौन से मार्ग वहाँ रुकते हैं।',
  'search.routePlaceholder': 'मार्ग संख्या या नाम, जैसे 500-D',
  'search.stopPlaceholder': 'स्टॉप नाम, जैसे शांतिनगर',
  'search.typeMore': 'कम से कम दो अक्षर लिखें।',
  'search.noRoutes': 'उस खोज से कोई मार्ग मेल नहीं खाता।',
  'search.noStops': 'उस खोज से कोई स्टॉप मेल नहीं खाता।',
  'search.trips': '{count} निर्धारित फेरे',
  'search.stopsCount': '{count} स्टॉप',
  'search.servedBy': 'इस स्टॉप पर सेवा देने वाले मार्ग',
  'search.onRoute': 'स्टॉप क्रम',
  'search.matched': 'आयातित डेटा से',

  'insights.eyebrow': 'भीड़ की जानकारी',
  'insights.title': 'रिपोर्ट का कुल चित्र',
  'insights.description': 'केवल इस ऐप में दर्ज यात्री रिपोर्ट से बनाया गया। कुछ अनुमानित नहीं है।',
  'insights.byLevel': 'भीड़ स्तर के अनुसार रिपोर्ट',
  'insights.byDay': 'प्रति दिन रिपोर्ट',
  'insights.total': 'कुल रिपोर्ट',
  'insights.last24': 'पिछले 24 घंटे में',
  'insights.empty': 'अभी कोई रिपोर्ट नहीं, इसलिए चार्ट नहीं।',

  'about.eyebrow': 'SAFAR का परिचय',
  'about.title': 'यहाँ क्या असली है, क्या नहीं',
  'about.realTitle': 'असली डेटा',
  'about.realBody': 'मार्ग संख्या, मार्ग नाम, क्रमबद्ध स्टॉप सूची, स्टॉप नाम और निर्धारित प्रस्थान समय नीचे दिए BMTC GTFS डेटा से आयात होकर इस ऐप के डेटाबेस में हैं।',
  'about.reportsTitle': 'यात्री रिपोर्ट',
  'about.reportsBody': 'भीड़ स्तर केवल यात्रियों की रिपोर्ट से आते हैं, हर एक के अपने समय के साथ। {hours} घंटे से पुरानी रिपोर्ट को पुरानी हो सकती है बताया जाता है।',
  'about.noTitle': 'उपलब्ध नहीं',
  'about.noBody': 'BMTC के लिए लाइव बस स्थान या यात्री संख्या की अनुमति वाली सार्वजनिक फ़ीड नहीं है। इसलिए SAFAR कभी लाइव बस स्थान, लाइव भीड़ स्तर या यात्री संख्या नहीं दिखाता।',
  'about.sourceTitle': 'स्रोत और लाइसेंस',
  'about.sourceBody': 'bmtc-gtfs आधिकारिक नम्मा BMTC ऐप से बना अनौपचारिक GTFS डेटा है, GitHub पर Vonter द्वारा प्रकाशित। लाइसेंस: ODbL 1.0।',
  'about.caveatTitle': 'स्रोत की ज्ञात सीमाएँ',
  'about.caveatBody': 'डेटा बनाने वालों के अनुसार नम्मा BMTC ऐप का डेटा सटीक नहीं है और समय-सारणी व स्टॉप समय गलत हो सकते हैं। समय को केवल संकेतात्मक मानें।',
  'about.coverageTitle': 'दायरा',
  'about.coverageBody': 'स्रोत ऐप में लाइव ट्रैकिंग काम करने वाले मार्ग ही डेटा में शामिल हैं।',
  'about.openSource': 'स्रोत डेटा खोलें',
  'about.snapshot': 'स्रोत कमिट {date} · जाँचा {checked}',

  'footer.blurb': 'SAFAR बेंगलुरु के लिए एक स्वतंत्र यात्री परियोजना है। इसका BMTC या कर्नाटक सरकार से कोई संबंध नहीं है।',
  'footer.noLive': 'लाइव बस ट्रैकिंग नहीं दिखाई जाती।',
  'footer.how': 'मार्ग और स्टॉप डेटा: BMTC GTFS डेटा (ODbL)। भीड़ स्तर: केवल यात्री रिपोर्ट।',
};

export const dictionaries = {
  en: { ...en, ...plannerText.en },
  kn: { ...kn, ...plannerText.kn },
  te: { ...te, ...plannerText.te },
  ta: { ...ta, ...plannerText.ta },
  hi: { ...hi, ...plannerText.hi },
};

export function translate(dictionary, key, vars) {
  const fallback = dictionaries.en[key] ?? key;
  const template = dictionary?.[key] ?? fallback;
  if (!vars) return template;
  return Object.keys(vars).reduce(
    (text, name) => text.replaceAll(`{${name}}`, String(vars[name])),
    template,
  );
}
#i18nPlanner
// Labels and messages for the trip planner, in every supported language.
// Kept in its own module so the main dictionary stays readable.

const en = {
  'planner.title': 'Plan your journey',
  'planner.description': 'Search the published BMTC stop list, set your own time limits and priority, and see the best matching journey together with real passenger crowd reports.',
  'planner.step.journey': 'Journey',
  'planner.step.limits': 'Time limits',
  'planner.step.priority': 'Priority',
  'planner.step.results': 'Results',
  'planner.back': 'Back',
  'planner.next': 'Next',
  'planner.find': 'Find My Best Journey',
  'planner.searching': 'Comparing published routes with passenger reports…',
  'planner.retry': 'Try again',
  'planner.restart': 'Plan another journey',
  'planner.changePriorities': 'Change priorities',
  'planner.err.run': 'The journey search could not be completed, so nothing is shown rather than a guess.',
  'planner.err.stops': 'Choose a starting stop and a destination.',
  'planner.err.same': 'The starting stop and the destination must be different.',
  'planner.err.viaSame': 'The intermediate stop must differ from the start and the destination.',
  'planner.err.time': 'Choose a valid departure date and time.',
  'planner.err.past': 'That departure time is in the past.',
  'planner.err.deadline': 'Choose a valid arrival deadline.',
  'planner.err.deadlineBefore': 'The deadline must be later than your departure time.',
  'planner.err.travel': 'Choose a maximum travel time, or enter a custom one.',
  'planner.err.wait': 'Choose a maximum waiting time, or enter a custom one.',
  'planner.err.priority': 'Choose one travel priority.',
  'planner.via': 'Intermediate stop (optional)',
  'planner.viaPlaceholder': 'Search a stop to pass through',
  'planner.viaHint': 'Only journeys whose route passes this stop in the right order are shown.',
  'planner.departTitle': 'Departure time',
  'planner.leaveNow': 'Leave now',
  'planner.chooseTime': 'Choose date and time',
  'planner.deadlineTitle': 'Arrival deadline',
  'planner.deadlineToggle': 'I must arrive by a certain time',
  'planner.deadlineLabel': 'Latest arrival time',
  'planner.deadlineHint': 'Journeys estimated to arrive later than this are excluded instead of recommended.',
  'planner.stopSourceNote': 'Location search covers the published BMTC stop names in the dataset. There is no address or landmark lookup, so pick the stop where you will actually board.',
  'planner.travel.title': 'What is the maximum time you can spend travelling?',
  'planner.travel.hint': 'Riding, waiting and transfers together. Walking time is not included because the dataset has no distances.',
  'planner.wait.title': 'How long are you willing to wait for a bus?',
  'planner.wait.hint': 'Waiting is estimated from the published departure times of each route.',
  'planner.wait.caveat': 'Live bus arrivals are not available for BMTC here, so waiting time is only ever an estimate from published timetables and cannot be verified.',
  'planner.custom': 'Custom',
  'planner.customLabel': 'Custom',
  'planner.minutesWord': 'minutes',
  'planner.minutesShort': '{count} min',
  'planner.priority.title': 'Travel priority',
  'planner.priority.hint': 'Pick one. The ranking of bus options changes with your choice.',
  'planner.priority.quickest': 'Reach as quickly as possible',
  'planner.priority.quickestDesc': 'Shortest reliable total time, including waiting and transfers. Crowd levels come second, but heavy crowding is still flagged.',
  'planner.priority.crowd': 'Avoid crowds',
  'planner.priority.crowdDesc': 'Prefer routes with recent reports of lower crowding, as long as they fit your time limits. A route with no reports is never called uncrowded.',
  'planner.priority.balanced': 'Balance speed and comfort',
  'planner.priority.balancedDesc': 'Weighs total time, reported crowding, waiting and transfers together, and explains the trade-off.',
  'planner.results.summary': '{from} to {to}',
  'planner.results.prefs': 'Leaving {depart} · travel limit {travel} min · wait limit {wait} min · priority: {priority}',
  'planner.results.deadline': 'Arrival deadline {time}',
  'planner.results.count': '{count} published options found, {eligible} fit your limits',
  'planner.recommended': 'Our recommendation',
  'planner.limitsOk': 'This option fits the limits you set: up to {travel} minutes travelling and {wait} minutes waiting.',
  'planner.noMatch.title': 'No bus option can be verified to meet your requirements',
  'planner.noMatch.body': 'None of the available bus options can be verified to meet your travel time, waiting time or arrival deadline. You may want to consider alternative transport.',
  'planner.noMatch.noJourneys': 'No published route in the dataset connects these two stops directly or with a single transfer. Try a different stop pair.',
  'planner.alternatives': 'Alternatives to compare',
  'planner.option.best': 'Best match for your priority',
  'planner.option.fastest': 'Fastest eligible option',
  'planner.option.calm': 'Least crowded eligible option',
  'planner.unverified.title': 'Waiting cannot be verified ({count})',
  'planner.unverified.body': 'For these routes the published timetable is too sparse to estimate waiting, so they are not offered as meeting your waiting limit.',
  'planner.unverified.item': 'Routes {routes} — about {minutes} min of riding, waiting unverified',
  'planner.excluded.title': 'Excluded by your limits ({count})',
  'planner.reason.serviceEnded': '{count} options run after the published trips for that route have ended',
  'planner.reason.travelLimit': '{count} options would take longer than your maximum travel time',
  'planner.reason.waitLimit': '{count} options would need more waiting than you allow',
  'planner.reason.waitUnverified': '{count} options cannot be checked against your waiting limit',
  'planner.reason.deadline': '{count} options would arrive after your deadline',
  'planner.note.crowdNoEvidence': 'You asked to avoid crowds, but there are no recent passenger reports for the routes that fit your limits, so a crowd-based recommendation is not reliable yet.',
  'planner.note.noReports': 'None of the eligible routes has a recent passenger report. Crowding here is unknown, not low.',
  'planner.note.conflicting': 'Some recent reports disagree with each other. Those routes are flagged as conflicting instead of being averaged into a level that nobody reported.',
  'planner.note.waitUnknown': 'For some routes the published timetable is too sparse to estimate waiting. They are listed separately and never presented as meeting your waiting limit.',
  'planner.note.allUnverified': 'No option could be checked against your waiting limit. The options listed are shown for information only.',
  'planner.crowd.title': 'Passenger crowd reports',
  'planner.crowd.none': 'No recent reports',
  'planner.crowd.countLabel': 'Reports',
  'planner.crowd.count': '{recent} in the last 2 hours · {older} older',
  'planner.crowd.latestLabel': 'Latest',
  'planner.crowd.latest': '{level}, {age}',
  'planner.crowd.atStopLabel': 'At your boarding stop',
  'planner.crowd.atStop': '{count} reports at {stop}, latest {age}',
  'planner.crowd.noneNote': 'No passenger has reported this route. That is unknown crowding, not low crowding.',
  'planner.crowd.limitedNote': 'Only one recent report supports this level, so treat it as a weak signal.',
  'planner.crowd.conflictNote': 'Recent reports disagree with each other, so no single crowd level is claimed.',
  'planner.crowd.staleNote': 'The last report for this route is {age} old. It is shown as history only, never as current crowding.',
  'planner.crowd.mixedNote': 'This journey uses two buses. The level shown is the more crowded of the two.',
  'planner.crowd.conf.recent': 'Recent reports',
  'planner.crowd.conf.limited': 'Limited evidence',
  'planner.crowd.conf.conflicting': 'Conflicting reports',
  'planner.crowd.conf.none': 'No reports',
  'planner.totalTime': 'about {minutes} min',
  'planner.leg': 'Leg {n} · route {route}',
  'planner.rideStops': '{stops} stops between boarding and alighting, about {minutes} min of riding',
  'planner.waitLabel': 'Waiting',
  'planner.waitUnverified': 'cannot be verified',
  'planner.waitPublished': 'next published departure at {time}',
  'planner.waitHeadway': 'about {minutes} min, from the published frequency',
  'planner.waitFirst': 'first published trip at {time}',
  'planner.estimatedDeparture': 'Departure',
  'planner.estimatedArrival': 'Estimated arrival',
  'planner.earliestArrival': 'Earliest arrival',
  'planner.rideTime': 'Riding',
  'planner.waitTime': 'Waiting',
  'planner.transfersNone': 'No transfers',
  'planner.transfersCount': '{count} transfer at {interchange}',
  'planner.walkUnknown': 'Walking time not estimated (no distance data)',
  'planner.scheduleWindow': 'Published departures {first} to {last}',
  'planner.frequency': 'about every {minutes} min',
  'planner.why': 'Why this option',
  'planner.exp.quickest': 'Shortest total time that fits your limits: about {minutes} minutes with {transfers} transfers.',
  'planner.exp.crowd': 'Chosen because recent passenger reports show {level} crowding and it still fits your time limits.',
  'planner.exp.balanced': 'Balances about {minutes} minutes of journey time against reported {level} crowding.',
  'planner.adv.singleBus': 'One bus only, so no transfer risk',
  'planner.adv.fastest': 'Fastest eligible option, at about {minutes} min',
  'planner.adv.reports': '{count} recent reports support {level} crowding',
  'planner.adv.calmest': 'Least crowded route that still fits your limits',
  'planner.adv.direct': 'Direct, no change of bus',
  'planner.dis.crowded': 'Recent reports show {level} crowding',
  'planner.dis.noReports': 'No recent passenger reports, so crowding is unknown',
  'planner.dis.conflicting': 'Recent reports disagree about crowding',
  'planner.dis.limitedEvidence': 'Only one recent report, so the crowd level is a weak signal',
  'planner.dis.slower': 'About {minutes} min slower than the fastest eligible option',
  'planner.dis.transfer': 'Needs {count} transfer, so a missed connection is possible',
  'planner.dis.waitUnknown': 'Waiting cannot be verified from the published timetable',
  'planner.dq.scheduled': 'Scheduled estimate',
  'planner.dq.reports': 'Recent passenger reports ({count})',
  'planner.dq.limited': 'Limited passenger reports',
  'planner.dq.conflicting': 'Conflicting passenger reports',
  'planner.dq.noReports': 'Insufficient crowd data',
  'planner.dq.insufficient': 'Insufficient data on waiting',
  'planner.assumptions.title': 'What is estimated and what is missing',
  'planner.assumption.durations': 'Riding time is estimated at about {pace} minutes per stop. The dataset has no stop-to-stop running times.',
  'planner.assumption.waiting': 'Waiting comes from the published origin departures of each route direction, or from the next published departure when you board where the trip starts.',
  'planner.assumption.schedule': 'The dataset publishes origin departure times for whole trips, so stop-level times are not available.',
  'planner.assumption.walking': 'Walking time and walking distance are not estimated at all.',
  'planner.alts.title': 'Alternative transport',
  'planner.alts.body': 'SAFAR has no verified data for these modes, so nothing is claimed about their times, prices or availability.',
  'planner.alts.metro': 'Namma Metro',
  'planner.alts.metroNote': 'no verified metro route or timetable data is loaded in SAFAR',
  'planner.alts.auto': 'Auto-rickshaw',
  'planner.alts.autoNote': 'no fare or travel-time data available',
  'planner.alts.taxi': 'Taxi or ride-hailing',
  'planner.alts.taxiNote': 'no availability, fare or travel-time data available',
  'planner.alts.walk': 'Walking or cycling',
  'planner.alts.walkNote': 'the dataset has no distances, so no walking or cycling time is shown',
  'planner.alts.manual': 'Compare options manually on Google Maps',
};

const kn = {
  'planner.title': 'ನಿಮ್ಮ ಪ್ರಯಾಣವನ್ನು ಯೋಜಿಸಿ',
  'planner.description': 'ಪ್ರಕಟಿತ BMTC ಸ್ಟಾಪ್ ಪಟ್ಟಿಯಲ್ಲಿ ಹುಡುಕಿ, ನಿಮ್ಮ ಸಮಯದ ಮಿತಿ ಮತ್ತು ಆದ್ಯತೆಯನ್ನು ಆಯ್ಕೆಮಾಡಿ, ನಿಜವಾದ ಪ್ರಯಾಣಿಕರ ಭೀಡ ವರದಿಗಳೊಂದಿಗೆ ಸೂಕ್ತ ಪ್ರಯಾಣವನ್ನು ನೋಡಿ.',
  'planner.step.journey': 'ಪ್ರಯಾಣ',
  'planner.step.limits': 'ಸಮಯ ಮಿತಿ',
  'planner.step.priority': 'ಆದ್ಯತೆ',
  'planner.step.results': 'ಫಲಿತಾಂಶ',
  'planner.back': 'ಹಿಂದೆ',
  'planner.next': 'ಮುಂದೆ',
  'planner.find': 'ನನ್ನ ಉತ್ತಮ ಪ್ರಯಾಣ ಹುಡುಕಿ',
  'planner.searching': 'ಪ್ರಕಟಿತ ಮಾರ್ಗಗಳನ್ನು ಯಾತ್ರಿ ವರದಿಗಳೊಂದಿಗೆ ಹೋಲಿಸಲಾಗುತ್ತಿದೆ…',
  'planner.retry': 'ಮತ್ತೆ ಪ್ರಯತ್ನಿಸಿ',
  'planner.restart': 'ಇನ್ನೊಂದು ಪ್ರಯಾಣ ಯೋಜಿಸಿ',
  'planner.changePriorities': 'ಆದ್ಯತೆ ಬದಲಿಸಿ',
  'planner.err.run': 'ಪ್ರಯಾಣ ಹುಡುಕಾಟ ಪೂರ್ಣವಾಗಲಿಲ್ಲ. ಊಹೆ ಬರೆಯುವುದಕ್ಕಿಂತ ಏನನ್ನೂ ತೋರಿಸಲಿಲ್ಲ.',
  'planner.err.stops': 'ಹೊರಡುವ ಸ್ಟಾಪ್ ಮತ್ತು ತಲುಪುವ ಸ್ಟಾಪ್ ಆಯ್ಕೆಮಾಡಿ.',
  'planner.err.same': 'ಹೊರಡುವ ಸ್ಟಾಪ್ ಮತ್ತು ತಲುಪುವ ಸ್ಟಾಪ್ ಬೇರೆ ಬೇರೆಯಾಗಿರಬೇಕು.',
  'planner.err.viaSame': 'ಮಧ್ಯದ ಸ್ಟಾಪ್ ಹೊರಡುವ ಮತ್ತು ತಲುಪುವ ಸ್ಟಾಪ್ಗಿಂತ ಬೇರೆಯಾಗಿರಬೇಕು.',
  'planner.err.time': 'ಸರಿಯಾದ ಹೊರಡುವ ದಿನಾಂಕ ಮತ್ತು ಸಮಯ ಆಯ್ಕೆಮಾಡಿ.',
  'planner.err.past': 'ಆ ಹೊರಡುವ ಸಮಯ ಕಳೆದುಹೋಗಿದೆ.',
  'planner.err.deadline': 'ಸರಿಯಾದ ತಲುಪುವ ಗಡುವು ಸಮಯ ಆಯ್ಕೆಮಾಡಿ.',
  'planner.err.deadlineBefore': 'ಗಡುವು ನಿಮ್ಮ ಹೊರಡುವ ಸಮಯಕ್ಕಿಂತ ನಂತರ ಇರಬೇಕು.',
  'planner.err.travel': 'ಗರಿಷ್ಠ ಪ್ರಯಾಣ ಸಮಯ ಆಯ್ಕೆಮಾಡಿ ಅಥವಾ ಬೇರೆ ಸಂಖ್ಯೆ ಬರೆಯಿರಿ.',
  'planner.err.wait': 'ಗರಿಷ್ಠ ಕಾಯುವ ಸಮಯ ಆಯ್ಕೆಮಾಡಿ ಅಥವಾ ಬೇರೆ ಸಂಖ್ಯೆ ಬರೆಯಿರಿ.',
  'planner.err.priority': 'ಒಂದು ಪ್ರಯಾಣ ಆದ್ಯತೆ ಆಯ್ಕೆಮಾಡಿ.',
  'planner.via': 'ಮಧ್ಯದ ಸ್ಟಾಪ್ (ಐಚ್ಛಿಕ)',
  'planner.viaPlaceholder': 'ಹಾದುಹೋಗುವ ಸ್ಟಾಪ್ ಹುಡುಕಿ',
  'planner.viaHint': 'ಈ ಸ್ಟಾಪ್ ಅನ್ನು ಸರಿಯಾದ ಕ್ರಮದಲ್ಲಿ ಹಾದುಹೋಗುವ ಪ್ರಯಾಣಗಳನ್ನು ಮಾತ್ರ ತೋರಿಸಲಾಗುತ್ತದೆ.',
  'planner.departTitle': 'ಹೊರಡುವ ಸಮಯ',
  'planner.leaveNow': 'ಈಗಲೇ ಹೊರಡು',
  'planner.chooseTime': 'ದಿನಾಂಕ ಮತ್ತು ಸಮಯ ಆಯ್ಕೆಮಾಡಿ',
  'planner.deadlineTitle': 'ತಲುಪುವ ಗಡುವು',
  'planner.deadlineToggle': 'ನಾನು ಒಂದು ನಿರ್ದಿಷ್ಟ ಸಮಯಕ್ಕೆ ತಲುಪಬೇಕು',
  'planner.deadlineLabel': 'ತಲುಪಬೇಕಾದ ಕೊನೆಯ ಸಮಯ',
  'planner.deadlineHint': 'ಇದಕ್ಕಿಂತ ನಂತರ ತಲುಪುವ ಪ್ರಯಾಣಗಳನ್ನು ಶಿಫಾರಸು ಮಾಡುವುದಿಲ್ಲ, ಬದಲಿಗೆ ಹೊರಗಿಡಲಾಗುತ್ತದೆ.',
  'planner.stopSourceNote': 'ಸ್ಥಳ ಹುಡುಕಾಟ ಡೇಟಾಸೆಟ್ನ ಪ್ರಕಟಿತ BMTC ಸ್ಟಾಪ್ ಹೆಸರುಗಳಿಗೆ ಸೀಮಿತ. ವಿಳಾಸ ಅಥವಾ ಹೆಗ್ಗುರುತು ಹುಡುಕಾಟ ಇಲ್ಲ, ಆದ್ದರಿಂದ ನೀವು ನಿಜವಾಗಿ ಹತ್ತುವ ಸ್ಟಾಪ್ ಆಯ್ಕೆಮಾಡಿ.',
  'planner.travel.title': 'ಪ್ರಯಾಣಕ್ಕೆ ನೀವು ಕೊಡಬಹುದಾದ ಗರಿಷ್ಠ ಸಮಯ ಎಷ್ಟು?',
  'planner.travel.hint': 'ಬಸ್ ಪ್ರಯಾಣ, ಕಾಯುವಿಕೆ ಮತ್ತು ಬದಲಾವಣೆ ಸೇರಿ. ದೂರದ ಮಾಹಿತಿ ಇಲ್ಲದ ಕಾರಣ ನಡಿಗೆಯ ಸಮಯ ಸೇರಿಲ್ಲ.',
  'planner.wait.title': 'ಬಸ್ಸಿಗೆ ಎಷ್ಟು ಸಮಯ ಕಾಯಲು ಸಿದ್ಧರಿದ್ದೀರಿ?',
  'planner.wait.hint': 'ಪ್ರತಿ ಮಾರ್ಗದ ಪ್ರಕಟಿತ ಹೊರಡುವ ಸಮಯಗಳಿಂದ ಕಾಯುವಿಕೆಯನ್ನು ಅಂದಾಜಿಸಲಾಗುತ್ತದೆ.',
  'planner.wait.caveat': 'ಇಲ್ಲಿ BMTC ಬಸ್ಗಳ ನೇರ ಆಗಮನ ಮಾಹಿತಿ ಇಲ್ಲ, ಆದ್ದರಿಂದ ಕಾಯುವ ಸಮಯ ಪ್ರಕಟಿತ ವೇಳಾಪಟ್ಟಿಯ ಅಂದಾಜು ಮಾತ್ರ; ಪರಿಶೀಲಿಸಲು ಸಾಧ್ಯವಿಲ್ಲ.',
  'planner.custom': 'ಬೇರೆ',
  'planner.customLabel': 'ಬೇರೆ',
  'planner.minutesWord': 'ನಿಮಿಷ',
  'planner.minutesShort': '{count} ನಿಮಿಷ',
  'planner.priority.title': 'ಪ್ರಯಾಣ ಆದ್ಯತೆ',
  'planner.priority.hint': 'ಒಂದನ್ನು ಆಯ್ಕೆಮಾಡಿ. ನಿಮ್ಮ ಆಯ್ಕೆಯಿಂದ ಬಸ್ ಆಯ್ಕೆಗಳ ಕ್ರಮ ಬದಲಾಗುತ್ತದೆ.',
  'planner.priority.quickest': 'ಆದಷ್ಟು ಬೇಗ ತಲುಪು',
  'planner.priority.quickestDesc': 'ಕಾಯುವಿಕೆ ಮತ್ತು ಬದಲಾವಣೆ ಸೇರಿ ಕಡಿಮೆ ಒಟ್ಟು ಸಮಯ. ಭೀಡ ಎರಡನೇ ಸ್ಥಾನ, ಆದರೆ ತುಂಬಾ ಭೀಡವಿದ್ದರೆ ಎಚ್ಚರಿಸಲಾಗುತ್ತದೆ.',
  'planner.priority.crowd': 'ಭೀಡ ತಪ್ಪಿಸು',
  'planner.priority.crowdDesc': 'ಸಮಯ ಮಿತಿಯೊಳಗೆ ಇರುವಂತೆಯೇ ಕಡಿಮೆ ಭೀಡದ ಇತ್ತೀಚಿನ ವರದಿ ಇರುವ ಮಾರ್ಗಗಳಿಗೆ ಆದ್ಯತೆ. ವರದಿ ಇಲ್ಲದ ಮಾರ್ಗವನ್ನು ಎಂದಿಗೂ ಕಡಿಮೆ ಭೀಡ ಎನ್ನುವುದಿಲ್ಲ.',
  'planner.priority.balanced': 'ವೇಗ ಮತ್ತು ಆರಾಮ ಸಮತೋಲನ',
  'planner.priority.balancedDesc': 'ಒಟ್ಟು ಸಮಯ, ವರದಿಯಾದ ಭೀಡ, ಕಾಯುವಿಕೆ ಮತ್ತು ಬದಲಾವಣೆಗಳನ್ನು ಜೊತೆಯಾಗಿ ತೂಗಿ ನೋಡಿ ವ್ಯತ್ಯಾಸವನ್ನು ವಿವರಿಸುತ್ತದೆ.',
  'planner.results.summary': '{from} ರಿಂದ {to} ವರೆಗೆ',
  'planner.results.prefs': '{depart} ಹೊರಡುವಿಕೆ · ಪ್ರಯಾಣ ಮಿತಿ {travel} ನಿಮಿಷ · ಕಾಯುವ ಮಿತಿ {wait} ನಿಮಿಷ · ಆದ್ಯತೆ: {priority}',
  'planner.results.deadline': 'ತಲುಪುವ ಗಡುವು {time}',
  'planner.results.count': '{count} ಪ್ರಕಟಿತ ಆಯ್ಕೆಗಳು ಸಿಕ್ಕಿವೆ, {eligible} ನಿಮ್ಮ ಮಿತಿಗೆ ಹೊಂದುತ್ತವೆ',
  'planner.recommended': 'ನಮ್ಮ ಶಿಫಾರಸು',
  'planner.limitsOk': 'ಈ ಆಯ್ಕೆ ನೀವು ಇಟ್ಟ ಮಿತಿಯೊಳಗಿದೆ: {travel} ನಿಮಿಷ ಪ್ರಯಾಣ ಮತ್ತು {wait} ನಿಮಿಷ ಕಾಯುವಿಕೆ.',
  'planner.noMatch.title': 'ನಿಮ್ಮ ಬೇಡಿಕೆಗೆ ಹೊಂದುವ ಬಸ್ ಆಯ್ಕೆ ಪರಿಶೀಲಿಸಲಾಗದು',
  'planner.noMatch.body': 'ಲಭ್ಯವಿರುವ ಬಸ್ ಆಯ್ಕೆಗಳಲ್ಲಿ ನಿಮ್ಮ ಪ್ರಯಾಣ ಸಮಯ, ಕಾಯುವ ಸಮಯ ಅಥವಾ ತಲುಪುವ ಗಡುವಿಗೆ ಹೊಂದುವುದನ್ನು ಪರಿಶೀಲಿಸಲು ಸಾಧ್ಯವಿಲ್ಲ. ಬೇರೆ ಸಾರಿಗೆ ಮಾರ್ಗವನ್ನು ಪರಿಗಣಿಸಬಹುದು.',
  'planner.noMatch.noJourneys': 'ಈ ಎರಡು ಸ್ಟಾಪ್ಗಳನ್ನು ನೇರವಾಗಿ ಅಥವಾ ಒಂದು ಬದಲಾವಣೆಯಿಂದ ಸೇರಿಸುವ ಪ್ರಕಟಿತ ಮಾರ್ಗ ಡೇಟಾದಲ್ಲಿ ಇಲ್ಲ. ಬೇರೆ ಸ್ಟಾಪ್ ಜೋಡಿ ಪ್ರಯತ್ನಿಸಿ.',
  'planner.alternatives': 'ಹೋಲಿಸಲು ಪರ್ಯಾಯಗಳು',
  'planner.option.best': 'ನಿಮ್ಮ ಆದ್ಯತೆಗೆ ಉತ್ತಮ ಹೊಂದಾಣಿಕೆ',
  'planner.option.fastest': 'ಅರ್ಹವಾದ ಅತಿ ವೇಗದ ಆಯ್ಕೆ',
  'planner.option.calm': 'ಅರ್ಹವಾದ ಕಡಿಮೆ ಭೀಡದ ಆಯ್ಕೆ',
  'planner.unverified.title': 'ಪರಿಶೀಲಿಸಲಾಗದ ಆಯ್ಕೆಗಳು ({count})',
  'planner.unverified.body': 'ಈ ಮಾರ್ಗಗಳಿಗೆ ಪ್ರಕಟಿತ ವೇಳಾಪಟ್ಟಿ ಕಡಿಮೆ ಇರುವುದರಿಂದ ಕಾಯುವಿಕೆ ಅಂದಾಜಿಸಲಾಗದು; ಆದ್ದರಿಂದ ಕಾಯುವ ಮಿತಿಗೆ ಹೊಂದುತ್ತವೆ ಎಂದು ತೋರಿಸುವುದಿಲ್ಲ.',
  'planner.unverified.item': 'ಮಾರ್ಗಗಳು {routes} — ಸುಮಾರು {minutes} ನಿಮಿಷ ಪ್ರಯಾಣ, ಕಾಯುವಿಕೆ ಪರಿಶೀಲಿಸಲಾಗದು',
  'planner.excluded.title': 'ನಿಮ್ಮ ಮಿತಿಯಿಂದ ಹೊರಗಿಟ್ಟವು ({count})',
  'planner.reason.serviceEnded': '{count} ಆಯ್ಕೆಗಳು ಆ ಮಾರ್ಗದ ಪ್ರಕಟಿತ ಕೊನೆಯ ಪ್ರಯಾಣದ ನಂತರ ಬರುತ್ತವೆ',
  'planner.reason.travelLimit': '{count} ಆಯ್ಕೆಗಳು ನಿಮ್ಮ ಗರಿಷ್ಠ ಪ್ರಯಾಣ ಸಮಯಕ್ಕಿಂತ ಹೆಚ್ಚು ತೆಗೆದುಕೊಳ್ಳುತ್ತವೆ',
  'planner.reason.waitLimit': '{count} ಆಯ್ಕೆಗಳಿಗೆ ನೀವು ಅನುಮತಿಸಿದ್ದಕ್ಕಿಂತ ಹೆಚ್ಚು ಕಾಯಬೇಕಾಗುತ್ತದೆ',
  'planner.reason.waitUnverified': '{count} ಆಯ್ಕೆಗಳನ್ನು ನಿಮ್ಮ ಕಾಯುವ ಮಿತಿಯೊಂದಿಗೆ ಪರಿಶೀಲಿಸಲಾಗದು',
  'planner.reason.deadline': '{count} ಆಯ್ಕೆಗಳು ನಿಮ್ಮ ಗಡುವಿನ ನಂತರ ತಲುಪುತ್ತವೆ',
  'planner.note.crowdNoEvidence': 'ನೀವು ಭೀಡ ತಪ್ಪಿಸಲು ಕೇಳಿದ್ದೀರಿ, ಆದರೆ ನಿಮ್ಮ ಮಿತಿಗೆ ಹೊಂದುವ ಮಾರ್ಗಗಳಿಗೆ ಇತ್ತೀಚಿನ ಯಾತ್ರಿ ವರದಿಗಳಿಲ್ಲ; ಆದ್ದರಿಂದ ಭೀಡ ಆಧಾರಿತ ಶಿಫಾರಸು ಇನ್ನೂ ನಂಬಲರ್ಹವಲ್ಲ.',
  'planner.note.noReports': 'ಅರ್ಹ ಮಾರ್ಗಗಳಲ್ಲಿ ಯಾವುದಕ್ಕೂ ಇತ್ತೀಚಿನ ಯಾತ್ರಿ ವರದಿ ಇಲ್ಲ. ಇಲ್ಲಿ ಭೀಡ ತಿಳಿದಿಲ್ಲ, ಕಡಿಮೆ ಅಲ್ಲ.',
  'planner.note.conflicting': 'ಕೆಲವು ಇತ್ತೀಚಿನ ವರದಿಗಳು ಒಂದಕ್ಕೊಂದು ಹೊಂದುವುದಿಲ್ಲ. ಅಂತಹ ಮಾರ್ಗಗಳನ್ನು ಯಾರೂ ವರದಿ ಮಾಡದ ಸ್ತರಕ್ಕೆ ಸರಾಸರಿ ಮಾಡುವ ಬದಲು ವಿರುದ್ಧ ಎಂದು ಗುರುತಿಸಲಾಗಿದೆ.',
  'planner.note.waitUnknown': 'ಕೆಲವು ಮಾರ್ಗಗಳ ಪ್ರಕಟಿತ ವೇಳಾಪಟ್ಟಿ ಬಹಳ ಕಡಿಮೆ ಇರುವುದರಿಂದ ಕಾಯುವಿಕೆ ಅಂದಾಜಿಸಲಾಗದು. ಅವನ್ನು ಪ್ರತ್ಯೇಕವಾಗಿ ತೋರಿಸಲಾಗಿದೆ ಮತ್ತು ನಿಮ್ಮ ಕಾಯುವ ಮಿತಿಗೆ ಹೊಂದುತ್ತವೆ ಎಂದು ಎಂದಿಗೂ ಹೇಳುವುದಿಲ್ಲ.',
  'planner.note.allUnverified': 'ಯಾವ ಆಯ್ಕೆಯನ್ನೂ ನಿಮ್ಮ ಕಾಯುವ ಮಿತಿಯೊಂದಿಗೆ ಪರಿಶೀಲಿಸಲಾಗಲಿಲ್ಲ. ಕೆಳಗಿನವು ಮಾಹಿತಿಗಾಗಿ ಮಾತ್ರ.',
  'planner.crowd.title': 'ಯಾತ್ರಿ ಭೀಡ ವರದಿಗಳು',
  'planner.crowd.none': 'ಇತ್ತೀಚಿನ ವರದಿ ಇಲ್ಲ',
  'planner.crowd.countLabel': 'ವರದಿಗಳು',
  'planner.crowd.count': 'ಕಳೆದ 2 ಗಂಟೆಯಲ್ಲಿ {recent} · ಹಳೆಯವು {older}',
  'planner.crowd.latestLabel': 'ಇತ್ತೀಚಿನದು',
  'planner.crowd.latest': '{level}, {age}',
  'planner.crowd.atStopLabel': 'ನೀವು ಹತ್ತುವ ಸ್ಟಾಪ್ನಲ್ಲಿ',
  'planner.crowd.atStop': '{stop} ನಲ್ಲಿ {count} ವರದಿಗಳು, ಇತ್ತೀಚಿನದು {age}',
  'planner.crowd.noneNote': 'ಈ ಮಾರ್ಗವನ್ನು ಯಾವ ಯಾತ್ರಿಯೂ ವರದಿ ಮಾಡಿಲ್ಲ. ಇದು ತಿಳಿಯದ ಭೀಡ, ಕಡಿಮೆ ಭೀಡ ಅಲ್ಲ.',
  'planner.crowd.limitedNote': 'ಈ ಸ್ತರವನ್ನು ಒಂದೇ ಇತ್ತೀಚಿನ ವರದಿ ಬೆಂಬಲಿಸುತ್ತದೆ; ಇದನ್ನು ದುರ್ಬಲ ಸೂಚನೆ ಎಂದು ಪರಿಗಣಿಸಿ.',
  'planner.crowd.conflictNote': 'ಇತ್ತೀಚಿನ ವರದಿಗಳು ಒಂದಕ್ಕೊಂದು ಹೊಂದುವುದಿಲ್ಲ, ಆದ್ದರಿಂದ ಒಂದೇ ಭೀಡ ಸ್ತರವನ್ನು ಹೇಳುವುದಿಲ್ಲ.',
  'planner.crowd.staleNote': 'ಈ ಮಾರ್ಗದ ಕೊನೆಯ ವರದಿ {age} ಹಳೆಯದು. ಅದನ್ನು ಇತಿಹಾಸವಾಗಿ ಮಾತ್ರ ತೋರಿಸಲಾಗಿದೆ, ಈಗಿನ ಭೀಡವಾಗಿ ಅಲ್ಲ.',
  'planner.crowd.mixedNote': 'ಈ ಪ್ರಯಾಣದಲ್ಲಿ ಎರಡು ಬಸ್ ಇದೆ. ತೋರಿಸಿದ ಸ್ತರ ಎರಡರಲ್ಲಿ ಹೆಚ್ಚು ಭೀಡದ್ದು.',
  'planner.crowd.conf.recent': 'ಇತ್ತೀಚಿನ ವರದಿಗಳು',
  'planner.crowd.conf.limited': 'ಸೀಮಿತ ಸಾಕ್ಷ್ಯ',
  'planner.crowd.conf.conflicting': 'ವಿರುದ್ಧ ವರದಿಗಳು',
  'planner.crowd.conf.none': 'ವರದಿ ಇಲ್ಲ',
  'planner.totalTime': 'ಸುಮಾರು {minutes} ನಿಮಿಷ',
  'planner.leg': 'ಹಂತ {n} · ಮಾರ್ಗ {route}',
  'planner.rideStops': 'ಹತ್ತುವ ಮತ್ತು ಇಳಿಯುವ ನಡುವೆ {stops} ಸ್ಟಾಪ್, ಸುಮಾರು {minutes} ನಿಮಿಷ ಪ್ರಯಾಣ',
  'planner.waitLabel': 'ಕಾಯುವಿಕೆ',
  'planner.waitUnverified': 'ಪರಿಶೀಲಿಸಲಾಗದು',
  'planner.waitPublished': 'ಮುಂದಿನ ಪ್ರಕಟಿತ ಹೊರಡುವಿಕೆ {time}',
  'planner.waitHeadway': 'ಪ್ರಕಟಿತ ಆವರ್ತನದ ಪ್ರಕಾರ ಸುಮಾರು {minutes} ನಿಮಿಷ',
  'planner.waitFirst': 'ಮೊದಲ ಪ್ರಕಟಿತ ಪ್ರಯಾಣ {time}',
  'planner.estimatedDeparture': 'ಹೊರಡುವಿಕೆ',
  'planner.estimatedArrival': 'ಅಂದಾಜು ತಲುಪುವಿಕೆ',
  'planner.earliestArrival': 'ಆದಷ್ಟು ಬೇಗ ತಲುಪುವಿಕೆ',
  'planner.rideTime': 'ಪ್ರಯಾಣ',
  'planner.waitTime': 'ಕಾಯುವಿಕೆ',
  'planner.transfersNone': 'ಬದಲಾವಣೆ ಇಲ್ಲ',
  'planner.transfersCount': '{interchange} ನಲ್ಲಿ {count} ಬದಲಾವಣೆ',
  'planner.walkUnknown': 'ನಡಿಗೆಯ ಸಮಯ ಅಂದಾಜಿಸಿಲ್ಲ (ದೂರದ ಮಾಹಿತಿ ಇಲ್ಲ)',
  'planner.scheduleWindow': 'ಪ್ರಕಟಿತ ಹೊರಡುವಿಕೆಗಳು {first} ರಿಂದ {last}',
  'planner.frequency': 'ಸುಮಾರು ಪ್ರತಿ {minutes} ನಿಮಿಷಕ್ಕೆ',
  'planner.why': 'ಈ ಆಯ್ಕೆ ಏಕೆ',
  'planner.exp.quickest': 'ನಿಮ್ಮ ಮಿತಿಗೆ ಹೊಂದುವ ಅತಿ ಕಡಿಮೆ ಒಟ್ಟು ಸಮಯ: ಸುಮಾರು {minutes} ನಿಮಿಷ, {transfers} ಬದಲಾವಣೆ.',
  'planner.exp.crowd': 'ಇತ್ತೀಚಿನ ಯಾತ್ರಿ ವರದಿಗಳು {level} ಭೀಡ ತೋರಿಸುತ್ತವೆ ಮತ್ತು ಇದು ನಿಮ್ಮ ಸಮಯ ಮಿತಿಗೆ ಹೊಂದುತ್ತದೆ ಎಂದು ಆಯ್ಕೆಮಾಡಲಾಗಿದೆ.',
  'planner.exp.balanced': 'ಸುಮಾರು {minutes} ನಿಮಿಷ ಪ್ರಯಾಣ ಮತ್ತು ವರದಿಯಾದ {level} ಭೀಡ ನಡುವೆ ಸಮತೋಲನ.',
  'planner.adv.singleBus': 'ಒಂದೇ ಬಸ್, ಬದಲಾವಣೆಯ ಅಪಾಯ ಇಲ್ಲ',
  'planner.adv.fastest': 'ಅರ್ಹವಾದ ಅತಿ ವೇಗದ ಆಯ್ಕೆ, ಸುಮಾರು {minutes} ನಿಮಿಷ',
  'planner.adv.reports': '{count} ಇತ್ತೀಚಿನ ವರದಿಗಳು {level} ಭೀಡವನ್ನು ಬೆಂಬಲಿಸುತ್ತವೆ',
  'planner.adv.calmest': 'ನಿಮ್ಮ ಮಿತಿಯೊಳಗೆ ಇರುವ ಅತಿ ಕಡಿಮೆ ಭೀಡದ ಮಾರ್ಗ',
  'planner.adv.direct': 'ನೇರ, ಬಸ್ ಬದಲಾವಣೆ ಇಲ್ಲ',
  'planner.dis.crowded': 'ಇತ್ತೀಚಿನ ವರದಿಗಳು {level} ಭೀಡ ತೋರಿಸುತ್ತವೆ',
  'planner.dis.noReports': 'ಇತ್ತೀಚಿನ ಯಾತ್ರಿ ವರದಿ ಇಲ್ಲ, ಆದ್ದರಿಂದ ಭೀಡ ತಿಳಿದಿಲ್ಲ',
  'planner.dis.conflicting': 'ಇತ್ತೀಚಿನ ವರದಿಗಳು ಭೀಡ ಬಗ್ಗೆ ಒಂದಕ್ಕೊಂದು ಹೊಂದುವುದಿಲ್ಲ',
  'planner.dis.limitedEvidence': 'ಒಂದೇ ಇತ್ತೀಚಿನ ವರದಿ, ಆದ್ದರಿಂದ ಭೀಡ ಸ್ತರ ದುರ್ಬಲ ಸೂಚನೆ',
  'planner.dis.slower': 'ಅರ್ಹವಾದ ಅತಿ ವೇಗದ ಆಯ್ಕೆಗಿಂತ ಸುಮಾರು {minutes} ನಿಮಿಷ ನಿಧಾನ',
  'planner.dis.transfer': '{count} ಬದಲಾವಣೆ ಬೇಕು, ಸಂಪರ್ಕ ತಪ್ಪಬಹುದು',
  'planner.dis.waitUnknown': 'ಪ್ರಕಟಿತ ವೇಳಾಪಟ್ಟಿಯಿಂದ ಕಾಯುವಿಕೆ ಪರಿಶೀಲಿಸಲಾಗದು',
  'planner.dq.scheduled': 'ವೇಳಾಪಟ್ಟಿ ಅಂದಾಜು',
  'planner.dq.reports': 'ಇತ್ತೀಚಿನ ಯಾತ್ರಿ ವರದಿಗಳು ({count})',
  'planner.dq.limited': 'ಸೀಮಿತ ಯಾತ್ರಿ ವರದಿಗಳು',
  'planner.dq.conflicting': 'ವಿರುದ್ಧ ಯಾತ್ರಿ ವರದಿಗಳು',
  'planner.dq.noReports': 'ಭೀಡ ಮಾಹಿತಿ ಸಾಕಷ್ಟಿಲ್ಲ',
  'planner.dq.insufficient': 'ಕಾಯುವಿಕೆಯ ಮಾಹಿತಿ ಸಾಕಷ್ಟಿಲ್ಲ',
  'planner.assumptions.title': 'ಏನು ಅಂದಾಜು, ಏನು ಇಲ್ಲ',
  'planner.assumption.durations': 'ಪ್ರಯಾಣ ಸಮಯವನ್ನು ಪ್ರತಿ ಸ್ಟಾಪ್ಗೆ ಸುಮಾರು {pace} ನಿಮಿಷ ಎಂದು ಅಂದಾಜಿಸಲಾಗಿದೆ. ಡೇಟಾದಲ್ಲಿ ಸ್ಟಾಪ್ ನಡುವಿನ ಸಮಯ ಇಲ್ಲ.',
  'planner.assumption.waiting': 'ಕಾಯುವಿಕೆ ಪ್ರತಿ ಮಾರ್ಗದ ಪ್ರಕಟಿತ ಮೂಲ ಹೊರಡುವಿಕೆಗಳಿಂದ, ಅಥವಾ ಪ್ರಯಾಣ ಆರಂಭವಾಗುವಲ್ಲೇ ಹತ್ತಿದರೆ ಮುಂದಿನ ಪ್ರಕಟಿತ ಹೊರಡುವಿಕೆಯಿಂದ ಬರುತ್ತದೆ.',
  'planner.assumption.schedule': 'ಡೇಟಾ ಇಡೀ ಪ್ರಯಾಣದ ಮೂಲ ಹೊರಡುವ ಸಮಯಗಳನ್ನು ಮಾತ್ರ ಪ್ರಕಟಿಸುತ್ತದೆ, ಸ್ಟಾಪ್ ಮಟ್ಟದ ಸಮಯ ಇಲ್ಲ.',
  'planner.assumption.walking': 'ನಡಿಗೆಯ ಸಮಯ ಮತ್ತು ದೂರವನ್ನು ಅಂದಾಜಿಸುವುದಿಲ್ಲ.',
  'planner.alts.title': 'ಪರ್ಯಾಯ ಸಾರಿಗೆ',
  'planner.alts.body': 'ಈ ಮಾರ್ಗಗಳಿಗೆ SAFAR ಬಳಿ ಪರಿಶೀಲಿತ ಮಾಹಿತಿ ಇಲ್ಲ, ಆದ್ದರಿಂದ ಸಮಯ, ದರ ಅಥವಾ ಲಭ್ಯತೆ ಬಗ್ಗೆ ಏನನ್ನೂ ಹೇಳುವುದಿಲ್ಲ.',
  'planner.alts.metro': 'ನಮ್ಮ ಮೆಟ್ರೋ',
  'planner.alts.metroNote': 'SAFAR ನಲ್ಲಿ ಪರಿಶೀಲಿತ ಮೆಟ್ರೋ ಮಾರ್ಗ ಅಥವಾ ವೇಳಾಪಟ್ಟಿ ಮಾಹಿತಿ ಇಲ್ಲ',
  'planner.alts.auto': 'ಆಟೋ ರಿಕ್ಷಾ',
  'planner.alts.autoNote': 'ದರ ಅಥವಾ ಪ್ರಯಾಣ ಸಮಯದ ಮಾಹಿತಿ ಇಲ್ಲ',
  'planner.alts.taxi': 'ಟ್ಯಾಕ್ಸಿ ಅಥವಾ ರೈಡ್ ಹೇಲಿಂಗ್',
  'planner.alts.taxiNote': 'ಲಭ್ಯತೆ, ದರ ಅಥವಾ ಪ್ರಯಾಣ ಸಮಯದ ಮಾಹಿತಿ ಇಲ್ಲ',
  'planner.alts.walk': 'ನಡಿಗೆ ಅಥವಾ ಸೈಕಲ್',
  'planner.alts.walkNote': 'ಡೇಟಾದಲ್ಲಿ ದೂರಗಳಿಲ್ಲ, ಆದ್ದರಿಂದ ನಡಿಗೆ ಅಥವಾ ಸೈಕಲ್ ಸಮಯ ತೋರಿಸುವುದಿಲ್ಲ',
  'planner.alts.manual': 'Google Maps ನಲ್ಲಿ ಆಯ್ಕೆಗಳನ್ನು ಹೋಲಿಸಿ',
};

const te = {
  'planner.title': 'మీ ప్రయాణాన్ని ప్లాన్ చేయండి',
  'planner.description': 'ప్రచురిత BMTC స్టాప్ జాబితాలో వెతకండి, మీ సమయ పరిమితులు, ప్రాధాన్యతను ఎంచుకోండి, నిజమైన ప్రయాణికుల రద్దీ నివేదికలతో సరిపోయే ప్రయాణాన్ని చూడండి.',
  'planner.step.journey': 'ప్రయాణం',
  'planner.step.limits': 'సమయ పరిమితులు',
  'planner.step.priority': 'ప్రాధాన్యత',
  'planner.step.results': 'ఫలితాలు',
  'planner.back': 'వెనుకకు',
  'planner.next': 'తదుపరి',
  'planner.find': 'నా ఉత్తమ ప్రయాణాన్ని వెతకండి',
  'planner.searching': 'ప్రచురిత మార్గాలను ప్రయాణికుల నివేదికలతో పోలుస్తున్నాము…',
  'planner.retry': 'మళ్లీ ప్రయత్నించండి',
  'planner.restart': 'మరో ప్రయాణం ప్లాన్ చేయండి',
  'planner.changePriorities': 'ప్రాధాన్యతలు మార్చండి',
  'planner.err.run': 'ప్రయాణ శోధన పూర్తికాలేదు. ఊహ రాయడం కంటే ఏమీ చూపలేదు.',
  'planner.err.stops': 'బయలుదేరే స్టాప్ మరియు గమ్యస్థానం ఎంచుకోండి.',
  'planner.err.same': 'బయలుదేరే స్టాప్ మరియు గమ్యస్థానం వేరుగా ఉండాలి.',
  'planner.err.viaSame': 'మధ్య స్టాప్ బయలుదేరే మరియు గమ్యస్థానం కంటే వేరుగా ఉండాలి.',
  'planner.err.time': 'సరైన బయలుదేరే తేదీ, సమయం ఎంచుకోండి.',
  'planner.err.past': 'ఆ బయలుదేరే సమయం గడిచిపోయింది.',
  'planner.err.deadline': 'సరైన చేరే గడువు సమయం ఎంచుకోండి.',
  'planner.err.deadlineBefore': 'గడువు మీ బయలుదేరే సమయం తర్వాత ఉండాలి.',
  'planner.err.travel': 'గరిష్ఠ ప్రయాణ సమయం ఎంచుకోండి లేదా వేరే సంఖ్య రాయండి.',
  'planner.err.wait': 'గరిష్ఠ నిరీక్షణ సమయం ఎంచుకోండి లేదా వేరే సంఖ్య రాయండి.',
  'planner.err.priority': 'ఒక ప్రయాణ ప్రాధాన్యతను ఎంచుకోండి.',
  'planner.via': 'మధ్య స్టాప్ (ఐచ్ఛికం)',
  'planner.viaPlaceholder': 'దాటే స్టాప్ వెతకండి',
  'planner.viaHint': 'ఈ స్టాప్ను సరైన వరుసలో దాటే ప్రయాణాలు మాత్రమే చూపబడతాయి.',
  'planner.departTitle': 'బయలుదేరే సమయం',
  'planner.leaveNow': 'ఇప్పుడే బయలుదేరు',
  'planner.chooseTime': 'తేదీ, సమయం ఎంచుకోండి',
  'planner.deadlineTitle': 'చేరే గడువు',
  'planner.deadlineToggle': 'నేను ఒక నిర్దిష్ట సమయానికి చేరాలి',
  'planner.deadlineLabel': 'చేరాల్సిన చివరి సమయం',
  'planner.deadlineHint': 'దీని తర్వాత చేరే ప్రయాణాలు సిఫారసు కావు, వదిలివేయబడతాయి.',
  'planner.stopSourceNote': 'స్థాన శోధన డేటాసెట్లోని ప్రచురిత BMTC స్టాప్ పేర్లకే పరిమితం. చిరునామా లేదా ల్యాండ్మార్క్ శోధన లేదు, కాబట్టి మీరు నిజంగా ఎక్కే స్టాప్ను ఎంచుకోండి.',
  'planner.travel.title': 'ప్రయాణానికి మీరు ఇవ్వగలిగే గరిష్ఠ సమయం ఎంత?',
  'planner.travel.hint': 'బస్ ప్రయాణం, నిరీక్షణ, మార్పిడి కలిపి. దూరాల డేటా లేనందున నడక సమయం చేరలేదు.',
  'planner.wait.title': 'బస్ కోసం ఎంత సేపు వేచి ఉండగలరు?',
  'planner.wait.hint': 'ప్రతి మార్గం ప్రచురిత బయలుదేరే సమయాల నుంచి నిరీక్షణ అంచనా వేయబడుతుంది.',
  'planner.wait.caveat': 'ఇక్కడ BMTC బస్సుల ప్రత్యక్ష రాక సమాచారం లేదు, కాబట్టి నిరీక్షణ సమయం ప్రచురిత షెడ్యూల్ నుంచి అంచనా మాత్రమే; ధృవీకరించలేము.',
  'planner.custom': 'ఇతర',
  'planner.customLabel': 'ఇతర',
  'planner.minutesWord': 'నిమిషాలు',
  'planner.minutesShort': '{count} నిమి',
  'planner.priority.title': 'ప్రయాణ ప్రాధాన్యత',
  'planner.priority.hint': 'ఒకటి ఎంచుకోండి. మీ ఎంపికతో బస్ ఎంపికల క్రమం మారుతుంది.',
  'planner.priority.quickest': 'వీలైనంత వేగంగా చేరు',
  'planner.priority.quickestDesc': 'నిరీక్షణ, మార్పిడి కలిపి తక్కువ మొత్తం సమయం. రద్దీ రెండో స్థానం, కానీ ఎక్కువ రద్దీ ఉంటే హెచ్చరిస్తాము.',
  'planner.priority.crowd': 'రద్దీని తప్పించు',
  'planner.priority.crowdDesc': 'సమయ పరిమితిలో ఉండగానే తక్కువ రద్దీకి సంబంధించిన ఇటీవలి నివేదికలున్న మార్గాలకు ప్రాధాన్యత. నివేదిక లేని మార్గాన్ని ఎప్పుడూ తక్కువ రద్దీ అనము.',
  'planner.priority.balanced': 'వేగం, సౌకర్యం సమతుల్యం',
  'planner.priority.balancedDesc': 'మొత్తం సమయం, నివేదిత రద్దీ, నిరీక్షణ, మార్పిడులను కలిపి తూకం వేసి వ్యత్యాసాన్ని వివరిస్తుంది.',
  'planner.results.summary': '{from} నుంచి {to} వరకు',
  'planner.results.prefs': '{depart} బయలుదేరడం · ప్రయాణ పరిమితి {travel} నిమి · నిరీక్షణ పరిమితి {wait} నిమి · ప్రాధాన్యత: {priority}',
  'planner.results.deadline': 'చేరే గడువు {time}',
  'planner.results.count': '{count} ప్రచురిత ఎంపికలు దొరికాయి, {eligible} మీ పరిమితులకు సరిపోతాయి',
  'planner.recommended': 'మా సిఫారసు',
  'planner.limitsOk': 'ఈ ఎంపిక మీరు పెట్టిన పరిమితుల్లో ఉంది: {travel} నిమిషాల ప్రయాణం, {wait} నిమిషాల నిరీక్షణ.',
  'planner.noMatch.title': 'మీ అవసరాలకు సరిపోయే బస్ ఎంపిక ధృవీకరించలేము',
  'planner.noMatch.body': 'అందుబాటులో ఉన్న బస్ ఎంపికలు మీ ప్రయాణ సమయం, నిరీక్షణ సమయం లేదా చేరే గడువుకు సరిపోతాయని ధృవీకరించలేము. ప్రత్యామ్నాయ రవాణాను పరిశీలించవచ్చు.',
  'planner.noMatch.noJourneys': 'ఈ రెండు స్టాప్లను నేరుగా లేదా ఒక మార్పిడితో కలిపే ప్రచురిత మార్గం డేటాలో లేదు. వేరే స్టాప్ జంట ప్రయత్నించండి.',
  'planner.alternatives': 'పోల్చడానికి ప్రత్యామ్నాయాలు',
  'planner.option.best': 'మీ ప్రాధాన్యతకు ఉత్తమ సరిపోలిక',
  'planner.option.fastest': 'అర్హత ఉన్న వేగవంతమైన ఎంపిక',
  'planner.option.calm': 'అర్హత ఉన్న తక్కువ రద్దీ ఎంపిక',
  'planner.unverified.title': 'ధృవీకరించలేనివి ({count})',
  'planner.unverified.body': 'ఈ మార్గాలకు ప్రచురిత షెడ్యూల్ చాలా తక్కువ కావడంతో నిరీక్షణ అంచనా వేయలేము; అందుకే మీ నిరీక్షణ పరిమితికి సరిపోతాయని చూపము.',
  'planner.unverified.item': 'మార్గాలు {routes} — సుమారు {minutes} నిమి ప్రయాణం, నిరీక్షణ ధృవీకరించలేము',
  'planner.excluded.title': 'మీ పరిమితుల వల్ల తొలగించినవి ({count})',
  'planner.reason.serviceEnded': '{count} ఎంపికలు ఆ మార్గం ప్రచురిత చివరి ప్రయాణాల తర్వాత ఉన్నాయి',
  'planner.reason.travelLimit': '{count} ఎంపికలు మీ గరిష్ఠ ప్రయాణ సమయం కంటే ఎక్కువ తీసుకుంటాయి',
  'planner.reason.waitLimit': '{count} ఎంపికలకు మీరు అనుమతించిన దానికంటే ఎక్కువ నిరీక్షణ కావాలి',
  'planner.reason.waitUnverified': '{count} ఎంపికలను మీ నిరీక్షణ పరిమితితో పోల్చలేము',
  'planner.reason.deadline': '{count} ఎంపికలు మీ గడువు తర్వాత చేరుతాయి',
  'planner.note.crowdNoEvidence': 'మీరు రద్దీని తప్పించాలని అడిగారు, కానీ మీ పరిమితులకు సరిపోయే మార్గాలకు ఇటీవలి ప్రయాణికుల నివేదికలు లేవు; కాబట్టి రద్దీ ఆధారిత సిఫారసు ఇంకా నమ్మదగినది కాదు.',
  'planner.note.noReports': 'అర్హత ఉన్న మార్గాల్లో ఏదీ ఇటీవలి ప్రయాణికుల నివేదిక కలిగి లేదు. ఇక్కడ రద్దీ తెలియదు, తక్కువ కాదు.',
  'planner.note.conflicting': 'కొన్ని ఇటీవలి నివేదికలు ఒకదానితో ఒకటి కలవడం లేదు. అలాంటి మార్గాలను ఎవరూ నివేదించని స్థాయికి సగటు చేయకుండా విరుద్ధంగా గుర్తించాము.',
  'planner.note.waitUnknown': 'కొన్ని మార్గాల ప్రచురిత షెడ్యూల్ చాలా తక్కువ కావడంతో నిరీక్షణ అంచనా వేయలేము. వాటిని విడిగా చూపాము; మీ నిరీక్షణ పరిమితికి సరిపోతాయని ఎప్పుడూ చెప్పము.',
  'planner.note.allUnverified': 'ఏ ఎంపికను మీ నిరీక్షణ పరిమితితో ధృవీకరించలేకపోయాము. కింద ఉన్నవి సమాచారం కోసం మాత్రమే.',
  'planner.crowd.title': 'ప్రయాణికుల రద్దీ నివేదికలు',
  'planner.crowd.none': 'ఇటీవలి నివేదికలు లేవు',
  'planner.crowd.countLabel': 'నివేదికలు',
  'planner.crowd.count': 'గత 2 గంటల్లో {recent} · పాతవి {older}',
  'planner.crowd.latestLabel': 'చివరిది',
  'planner.crowd.latest': '{level}, {age}',
  'planner.crowd.atStopLabel': 'మీరు ఎక్కే స్టాప్ వద్ద',
  'planner.crowd.atStop': '{stop} వద్ద {count} నివేదికలు, చివరిది {age}',
  'planner.crowd.noneNote': 'ఈ మార్గాన్ని ఏ ప్రయాణికుడూ నివేదించలేదు. ఇది తెలియని రద్దీ, తక్కువ రద్దీ కాదు.',
  'planner.crowd.limitedNote': 'ఈ స్థాయిని ఒకే ఇటీవలి నివేదిక బలపరుస్తుంది; దీన్ని బలహీన సూచనగా చూడండి.',
  'planner.crowd.conflictNote': 'ఇటీవలి నివేదికలు ఒకదానితో ఒకటి కలవడం లేదు, కాబట్టి ఒకే రద్దీ స్థాయిని చెప్పము.',
  'planner.crowd.staleNote': 'ఈ మార్గానికి చివరి నివేదిక {age} పాతది. దాన్ని చరిత్రగా మాత్రమే చూపుతున్నాము, ప్రస్తుత రద్దీగా కాదు.',
  'planner.crowd.mixedNote': 'ఈ ప్రయాణంలో రెండు బస్సులు ఉన్నాయి. చూపిన స్థాయి రెండింటిలో ఎక్కువ రద్దీది.',
  'planner.crowd.conf.recent': 'ఇటీవలి నివేదికలు',
  'planner.crowd.conf.limited': 'పరిమిత సాక్ష్యం',
  'planner.crowd.conf.conflicting': 'విరుద్ధ నివేదికలు',
  'planner.crowd.conf.none': 'నివేదికలు లేవు',
  'planner.totalTime': 'సుమారు {minutes} నిమి',
  'planner.leg': 'దశ {n} · మార్గం {route}',
  'planner.rideStops': 'ఎక్కడం నుంచి దిగడం మధ్య {stops} స్టాప్లు, సుమారు {minutes} నిమి ప్రయాణం',
  'planner.waitLabel': 'నిరీక్షణ',
  'planner.waitUnverified': 'ధృవీకరించలేము',
  'planner.waitPublished': 'తదుపరి ప్రచురిత బయలుదేరడం {time}',
  'planner.waitHeadway': 'ప్రచురిత ఫ్రీక్వెన్సీ ప్రకారం సుమారు {minutes} నిమి',
  'planner.waitFirst': 'మొదటి ప్రచురిత ప్రయాణం {time}',
  'planner.estimatedDeparture': 'బయలుదేరడం',
  'planner.estimatedArrival': 'అంచనా చేరిక',
  'planner.earliestArrival': 'ముందుగా చేరే సమయం',
  'planner.rideTime': 'ప్రయాణం',
  'planner.waitTime': 'నిరీక్షణ',
  'planner.transfersNone': 'మార్పిడి లేదు',
  'planner.transfersCount': '{interchange} వద్ద {count} మార్పిడి',
  'planner.walkUnknown': 'నడక సమయం అంచనా వేయలేదు (దూరాల డేటా లేదు)',
  'planner.scheduleWindow': 'ప్రచురిత బయలుదేరే సమయాలు {first} నుంచి {last}',
  'planner.frequency': 'సుమారు ప్రతి {minutes} నిమి',
  'planner.why': 'ఈ ఎంపిక ఎందుకు',
  'planner.exp.quickest': 'మీ పరిమితుల్లో అతి తక్కువ మొత్తం సమయం: సుమారు {minutes} నిమిషాలు, {transfers} మార్పిడులు.',
  'planner.exp.crowd': 'ఇటీవలి ప్రయాణికుల నివేదికలు {level} రద్దీ చూపుతున్నందున, అది మీ సమయ పరిమితుల్లో ఉండటంతో ఎంచుకున్నాము.',
  'planner.exp.balanced': 'సుమారు {minutes} నిమిషాల ప్రయాణ సమయాన్ని నివేదిత {level} రద్దీతో సమతుల్యం చేస్తుంది.',
  'planner.adv.singleBus': 'ఒకే బస్సు, మార్పిడి ప్రమాదం లేదు',
  'planner.adv.fastest': 'అర్హత ఉన్న వేగవంతమైన ఎంపిక, సుమారు {minutes} నిమి',
  'planner.adv.reports': '{count} ఇటీవలి నివేదికలు {level} రద్దీని బలపరుస్తున్నాయి',
  'planner.adv.calmest': 'మీ పరిమితుల్లో ఉండే అతి తక్కువ రద్దీ మార్గం',
  'planner.adv.direct': 'నేరుగా, బస్సు మార్పు లేదు',
  'planner.dis.crowded': 'ఇటీవలి నివేదికలు {level} రద్దీ చూపుతున్నాయి',
  'planner.dis.noReports': 'ఇటీవలి ప్రయాణికుల నివేదికలు లేవు, కాబట్టి రద్దీ తెలియదు',
  'planner.dis.conflicting': 'ఇటీవలి నివేదికలు రద్దీ విషయంలో కలవడం లేదు',
  'planner.dis.limitedEvidence': 'ఒకే ఇటీవలి నివేదిక, కాబట్టి రద్దీ స్థాయి బలహీన సూచన',
  'planner.dis.slower': 'అర్హత ఉన్న వేగవంతమైన ఎంపిక కంటే సుమారు {minutes} నిమి నెమ్మది',
  'planner.dis.transfer': '{count} మార్పిడి కావాలి, కనెక్షన్ తప్పవచ్చు',
  'planner.dis.waitUnknown': 'ప్రచురిత షెడ్యూల్ నుంచి నిరీక్షణ ధృవీకరించలేము',
  'planner.dq.scheduled': 'షెడ్యూల్ అంచనా',
  'planner.dq.reports': 'ఇటీవలి ప్రయాణికుల నివేదికలు ({count})',
  'planner.dq.limited': 'పరిమిత ప్రయాణికుల నివేదికలు',
  'planner.dq.conflicting': 'విరుద్ధ ప్రయాణికుల నివేదికలు',
  'planner.dq.noReports': 'రద్దీ డేటా సరిపోదు',
  'planner.dq.insufficient': 'నిరీక్షణ డేటా సరిపోదు',
  'planner.assumptions.title': 'ఏది అంచనా, ఏది లేదు',
  'planner.assumption.durations': 'ప్రయాణ సమయం ప్రతి స్టాప్కు సుమారు {pace} నిమిషాలుగా అంచనా వేయబడింది. డేటాలో స్టాప్ మధ్య సమయాలు లేవు.',
  'planner.assumption.waiting': 'నిరీక్షణ ప్రతి మార్గం ప్రచురిత మూల బయలుదేరే సమయాల నుంచి, లేదా ప్రయాణం మొదలయ్యే చోట ఎక్కితే తదుపరి ప్రచురిత బయలుదేరడం నుంచి వస్తుంది.',
  'planner.assumption.schedule': 'డేటా మొత్తం ప్రయాణానికి మూల బయలుదేరే సమయాలను మాత్రమే ప్రచురిస్తుంది, స్టాప్ స్థాయి సమయాలు లేవు.',
  'planner.assumption.walking': 'నడక సమయం, నడక దూరం అంచనా వేయబడవు.',
  'planner.alts.title': 'ప్రత్యామ్నాయ రవాణా',
  'planner.alts.body': 'ఈ మార్గాలకు SAFAR వద్ద ధృవీకరించిన డేటా లేదు, కాబట్టి వాటి సమయం, ధర లేదా లభ్యత గురించి ఏమీ చెప్పము.',
  'planner.alts.metro': 'నమ్మ మెట్రో',
  'planner.alts.metroNote': 'SAFAR లో ధృవీకరించిన మెట్రో మార్గం లేదా షెడ్యూల్ డేటా లోడ్ కాలేదు',
  'planner.alts.auto': 'ఆటో రిక్షా',
  'planner.alts.autoNote': 'ధర లేదా ప్రయాణ సమయ డేటా అందుబాటులో లేదు',
  'planner.alts.taxi': 'టాక్సీ లేదా రైడ్-హెయిలింగ్',
  'planner.alts.taxiNote': 'లభ్యత, ధర లేదా ప్రయాణ సమయ డేటా అందుబాటులో లేదు',
  'planner.alts.walk': 'నడక లేదా సైకిల్',
  'planner.alts.walkNote': 'డేటాలో దూరాలు లేవు, కాబట్టి నడక లేదా సైకిల్ సమయం చూపము',
  'planner.alts.manual': 'Google Maps లో ఎంపికలను పోల్చండి',
};

const ta = {
  'planner.title': 'உங்கள் பயணத்தைத் திட்டமிடுங்கள்',
  'planner.description': 'வெளியிடப்பட்ட BMTC நிறுத்தப் பட்டியலில் தேடி, உங்கள் நேர வரம்புகளையும் முன்னுரிமையையும் தேர்ந்தெடுத்து, உண்மையான பயணிகளின் கூட்ட நெரிசல் அறிக்கைகளுடன் சிறந்த பயணத்தைப் பாருங்கள்.',
  'planner.step.journey': 'பயணம்',
  'planner.step.limits': 'நேர வரம்புகள்',
  'planner.step.priority': 'முன்னுரிமை',
  'planner.step.results': 'முடிவுகள்',
  'planner.back': 'பின்',
  'planner.next': 'அடுத்து',
  'planner.find': 'எனது சிறந்த பயணத்தைக் கண்டறி',
  'planner.searching': 'வெளியிடப்பட்ட வழித்தடங்களை பயணிகளின் அறிக்கைகளுடன் ஒப்பிடுகிறோம்…',
  'planner.retry': 'மீண்டும் முயற்சி',
  'planner.restart': 'மற்றொரு பயணத்தைத் திட்டமிடு',
  'planner.changePriorities': 'முன்னுரிமைகளை மாற்று',
  'planner.err.run': 'பயணத் தேடலை முடிக்க முடியவில்லை. ஊகம் எழுதுவதற்குப் பதிலாக எதுவும் காட்டப்படவில்லை.',
  'planner.err.stops': 'புறப்படும் நிறுத்தமும் சேரும் இடமும் தேர்ந்தெடுக்கவும்.',
  'planner.err.same': 'புறப்படும் நிறுத்தமும் சேரும் இடமும் வேறுபட்டிருக்க வேண்டும்.',
  'planner.err.viaSame': 'இடைநிலை நிறுத்தம் புறப்படும் மற்றும் சேரும் இடங்களிலிருந்து வேறுபட்டிருக்க வேண்டும்.',
  'planner.err.time': 'சரியான புறப்பாட்டு தேதியும் நேரமும் தேர்ந்தெடுக்கவும்.',
  'planner.err.past': 'அந்தப் புறப்பாட்டு நேரம் கடந்துவிட்டது.',
  'planner.err.deadline': 'சரியான சேரும் கடைசி நேரத்தைத் தேர்ந்தெடுக்கவும்.',
  'planner.err.deadlineBefore': 'கடைசி நேரம் உங்கள் புறப்பாட்டு நேரத்திற்குப் பின் இருக்க வேண்டும்.',
  'planner.err.travel': 'அதிகபட்ச பயண நேரத்தைத் தேர்ந்தெடுக்கவும் அல்லது வேறு எண்ணை எழுதவும்.',
  'planner.err.wait': 'அதிகபட்ச காத்திருப்பு நேரத்தைத் தேர்ந்தெடுக்கவும் அல்லது வேறு எண்ணை எழுதவும்.',
  'planner.err.priority': 'ஒரு பயண முன்னுரிமையைத் தேர்ந்தெடுக்கவும்.',
  'planner.via': 'இடைநிலை நிறுத்தம் (விருப்பம்)',
  'planner.viaPlaceholder': 'கடந்து செல்லும் நிறுத்தத்தைத் தேடுங்கள்',
  'planner.viaHint': 'இந்த நிறுத்தத்தை சரியான வரிசையில் கடக்கும் பயணங்கள் மட்டுமே காட்டப்படும்.',
  'planner.departTitle': 'புறப்படும் நேரம்',
  'planner.leaveNow': 'இப்போதே புறப்படு',
  'planner.chooseTime': 'தேதியும் நேரமும் தேர்ந்தெடு',
  'planner.deadlineTitle': 'சேர வேண்டிய கடைசி நேரம்',
  'planner.deadlineToggle': 'நான் குறிப்பிட்ட நேரத்திற்குள் சேர வேண்டும்',
  'planner.deadlineLabel': 'சேர வேண்டிய கடைசி நேரம்',
  'planner.deadlineHint': 'இதற்குப் பின் சேரும் பயணங்கள் பரிந்துரைக்கப்படாமல் விலக்கப்படும்.',
  'planner.stopSourceNote': 'இடத் தேடல் தரவுத்தொகுப்பில் உள்ள வெளியிடப்பட்ட BMTC நிறுத்தப் பெயர்களுக்கே உரியது. முகவரி அல்லது அடையாள இடத் தேடல் இல்லை, எனவே நீங்கள் உண்மையில் ஏறும் நிறுத்தத்தைத் தேர்ந்தெடுக்கவும்.',
  'planner.travel.title': 'பயணத்திற்கு நீங்கள் செலவிடக்கூடிய அதிகபட்ச நேரம் என்ன?',
  'planner.travel.hint': 'பேருந்து பயணம், காத்திருப்பு, மாற்றம் சேர்த்து. தூரத் தரவு இல்லாததால் நடை நேரம் சேர்க்கப்படவில்லை.',
  'planner.wait.title': 'பேருந்துக்கு எவ்வளவு நேரம் காத்திருக்கத் தயார்?',
  'planner.wait.hint': 'ஒவ்வொரு வழித்தடத்தின் வெளியிடப்பட்ட புறப்பாட்டு நேரங்களிலிருந்து காத்திருப்பு மதிப்பிடப்படுகிறது.',
  'planner.wait.caveat': 'இங்கு BMTC பேருந்துகளின் நேரடி வருகைத் தகவல் இல்லை, எனவே காத்திருப்பு நேரம் வெளியிடப்பட்ட அட்டவணையின் மதிப்பீடு மட்டுமே; உறுதிப்படுத்த முடியாது.',
  'planner.custom': 'வேறு',
  'planner.customLabel': 'வேறு',
  'planner.minutesWord': 'நிமிடங்கள்',
  'planner.minutesShort': '{count} நிமி',
  'planner.priority.title': 'பயண முன்னுரிமை',
  'planner.priority.hint': 'ஒன்றைத் தேர்ந்தெடுக்கவும். உங்கள் தேர்வால் பேருந்து விருப்பங்களின் வரிசை மாறும்.',
  'planner.priority.quickest': 'விரைவாகச் சேரு',
  'planner.priority.quickestDesc': 'காத்திருப்பு, மாற்றம் உட்படக் குறைந்த மொத்த நேரம். கூட்ட நெரிசல் இரண்டாம் இடம், ஆனால் அதிக நெரிசல் இருந்தால் எச்சரிக்கை.',
  'planner.priority.crowd': 'நெரிசலைத் தவிர்',
  'planner.priority.crowdDesc': 'நேர வரம்புக்குள் இருக்கும்வரை குறைந்த நெரிசலுக்கான சமீபத்திய அறிக்கைகள் உள்ள வழித்தடங்களுக்கு முன்னுரிமை. அறிக்கை இல்லாத வழித்தடத்தை ஒருபோதும் குறைந்த நெரிசல் எனக் கூறமாட்டோம்.',
  'planner.priority.balanced': 'வேகமும் வசதியும் சமநிலை',
  'planner.priority.balancedDesc': 'மொத்த நேரம், அறிக்கையிடப்பட்ட நெரிசல், காத்திருப்பு, மாற்றங்களை ஒன்றாக எடைபோட்டு வித்தியாசத்தை விளக்குகிறது.',
  'planner.results.summary': '{from} முதல் {to} வரை',
  'planner.results.prefs': '{depart} புறப்பாடு · பயண வரம்பு {travel} நிமி · காத்திருப்பு வரம்பு {wait} நிமி · முன்னுரிமை: {priority}',
  'planner.results.deadline': 'சேர வேண்டிய கடைசி நேரம் {time}',
  'planner.results.count': '{count} வெளியிடப்பட்ட விருப்பங்கள், {eligible} உங்கள் வரம்புக்குள்',
  'planner.recommended': 'எங்கள் பரிந்துரை',
  'planner.limitsOk': 'இந்த விருப்பம் நீங்கள் வைத்த வரம்புக்குள் உள்ளது: {travel} நிமிட பயணம், {wait} நிமிட காத்திருப்பு.',
  'planner.noMatch.title': 'உங்கள் தேவைகளைப் பூர்த்தி செய்யும் பேருந்து விருப்பத்தை உறுதிப்படுத்த முடியாது',
  'planner.noMatch.body': 'கிடைக்கும் பேருந்து விருப்பங்கள் உங்கள் பயண நேரம், காத்திருப்பு நேரம் அல்லது கடைசி நேரத்தைப் பூர்த்தி செய்வதை உறுதிப்படுத்த முடியாது. மாற்று வழிகளைப் பரிசீலிக்கலாம்.',
  'planner.noMatch.noJourneys': 'இந்த இரண்டு நிறுத்தங்களை நேரடியாகவோ ஒரு மாற்றத்துடனோ இணைக்கும் வெளியிடப்பட்ட வழித்தடம் தரவில் இல்லை. வேறு நிறுத்த இணையை முயற்சிக்கவும்.',
  'planner.alternatives': 'ஒப்பிட மாற்று வழிகள்',
  'planner.option.best': 'உங்கள் முன்னுரிமைக்கு சிறந்த பொருத்தம்',
  'planner.option.fastest': 'தகுதியான வேகமான விருப்பம்',
  'planner.option.calm': 'தகுதியான குறைந்த நெரிசல் விருப்பம்',
  'planner.unverified.title': 'உறுதிப்படுத்த முடியாதவை ({count})',
  'planner.unverified.body': 'இந்த வழித்தடங்களுக்கு வெளியிடப்பட்ட அட்டவணை மிகக் குறைவு என்பதால் காத்திருப்பை மதிப்பிட முடியாது; எனவே உங்கள் காத்திருப்பு வரம்புக்குள் வருவதாகக் காட்டப்படவில்லை.',
  'planner.unverified.item': 'வழித்தடங்கள் {routes} — சுமார் {minutes} நிமி பயணம், காத்திருப்பு உறுதிப்படுத்தப்படவில்லை',
  'planner.excluded.title': 'உங்கள் வரம்புகளால் விலக்கப்பட்டவை ({count})',
  'planner.reason.serviceEnded': '{count} விருப்பங்கள் அந்த வழித்தடத்தின் கடைசி வெளியிடப்பட்ட பயணத்திற்குப் பின் உள்ளன',
  'planner.reason.travelLimit': '{count} விருப்பங்கள் உங்கள் அதிகபட்ச பயண நேரத்தை மீறும்',
  'planner.reason.waitLimit': '{count} விருப்பங்களுக்கு நீங்கள் அனுமதிப்பதைவிட அதிக காத்திருப்பு தேவை',
  'planner.reason.waitUnverified': '{count} விருப்பங்களை உங்கள் காத்திருப்பு வரம்புடன் ஒப்பிட முடியாது',
  'planner.reason.deadline': '{count} விருப்பங்கள் உங்கள் கடைசி நேரத்திற்குப் பின் சேரும்',
  'planner.note.crowdNoEvidence': 'நெரிசலைத் தவிர்க்கக் கேட்டீர்கள், ஆனால் உங்கள் வரம்புக்குள் வரும் வழித்தடங்களுக்கு சமீபத்திய பயணிகளின் அறிக்கைகள் இல்லை; எனவே நெரிசல் அடிப்படையிலான பரிந்துரை இன்னும் நம்பகமானதல்ல.',
  'planner.note.noReports': 'தகுதியான வழித்தடங்களில் எதற்கும் சமீபத்திய பயணிகளின் அறிக்கை இல்லை. இங்கு நெரிசல் தெரியவில்லை, குறைவு அல்ல.',
  'planner.note.conflicting': 'சில சமீபத்திய அறிக்கைகள் ஒன்றுக்கொன்று பொருந்தவில்லை. அந்த வழித்தடங்கள் யாரும் அறிக்கையிடாத நிலைக்கு சராசரியாக்கப்படாமல் முரண்பாடு எனக் குறிக்கப்பட்டுள்ளன.',
  'planner.note.waitUnknown': 'சில வழித்தடங்களின் வெளியிடப்பட்ட அட்டவணை மிகக் குறைவு என்பதால் காத்திருப்பை மதிப்பிட முடியாது. அவை தனியாகக் காட்டப்படுகின்றன; உங்கள் வரம்புக்குள் வருவதாக ஒருபோதும் கூறப்படாது.',
  'planner.note.allUnverified': 'எந்த விருப்பத்தையும் உங்கள் காத்திருப்பு வரம்புடன் உறுதிப்படுத்த முடியவில்லை. கீழே உள்ளவை தகவலுக்காக மட்டுமே.',
  'planner.crowd.title': 'பயணிகள் நெரிசல் அறிக்கைகள்',
  'planner.crowd.none': 'சமீபத்திய அறிக்கைகள் இல்லை',
  'planner.crowd.countLabel': 'அறிக்கைகள்',
  'planner.crowd.count': 'கடந்த 2 மணி நேரத்தில் {recent} · பழையவை {older}',
  'planner.crowd.latestLabel': 'சமீபத்தியது',
  'planner.crowd.latest': '{level}, {age}',
  'planner.crowd.atStopLabel': 'நீங்கள் ஏறும் நிறுத்தத்தில்',
  'planner.crowd.atStop': '{stop} இல் {count} அறிக்கைகள், சமீபத்தியது {age}',
  'planner.crowd.noneNote': 'இந்த வழித்தடத்தை எந்தப் பயணியும் அறிக்கையிடவில்லை. இது தெரியாத நெரிசல், குறைந்த நெரிசல் அல்ல.',
  'planner.crowd.limitedNote': 'இந்த நிலையை ஒரே சமீபத்திய அறிக்கை ஆதரிக்கிறது; பலவீனமான சமிக்ஞையாகக் கருதுங்கள்.',
  'planner.crowd.conflictNote': 'சமீபத்திய அறிக்கைகள் ஒன்றுக்கொன்று பொருந்தவில்லை, எனவே ஒரே நெரிசல் நிலை கூறப்படவில்லை.',
  'planner.crowd.staleNote': 'இந்த வழித்தடத்தின் கடைசி அறிக்கை {age} பழையது. அது வரலாறாக மட்டுமே காட்டப்படுகிறது, தற்போதைய நெரிசலாக அல்ல.',
  'planner.crowd.mixedNote': 'இந்தப் பயணத்தில் இரண்டு பேருந்துகள் உள்ளன. காட்டப்படும் நிலை இரண்டில் அதிக நெரிசலானது.',
  'planner.crowd.conf.recent': 'சமீபத்திய அறிக்கைகள்',
  'planner.crowd.conf.limited': 'குறைந்த ஆதாரம்',
  'planner.crowd.conf.conflicting': 'முரண்பட்ட அறிக்கைகள்',
  'planner.crowd.conf.none': 'அறிக்கைகள் இல்லை',
  'planner.totalTime': 'சுமார் {minutes} நிமி',
  'planner.leg': 'பகுதி {n} · வழித்தடம் {route}',
  'planner.rideStops': 'ஏறுதல் முதல் இறங்குதல் வரை {stops} நிறுத்தங்கள், சுமார் {minutes} நிமி பயணம்',
  'planner.waitLabel': 'காத்திருப்பு',
  'planner.waitUnverified': 'உறுதிப்படுத்த முடியாது',
  'planner.waitPublished': 'அடுத்த வெளியிடப்பட்ட புறப்பாடு {time}',
  'planner.waitHeadway': 'வெளியிடப்பட்ட அலைவெண் அடிப்படையில் சுமார் {minutes} நிமி',
  'planner.waitFirst': 'முதல் வெளியிடப்பட்ட பயணம் {time}',
  'planner.estimatedDeparture': 'புறப்பாடு',
  'planner.estimatedArrival': 'மதிப்பிடப்பட்ட சேரல்',
  'planner.earliestArrival': 'மிக விரைவான சேரல்',
  'planner.rideTime': 'பயணம்',
  'planner.waitTime': 'காத்திருப்பு',
  'planner.transfersNone': 'மாற்றம் இல்லை',
  'planner.transfersCount': '{interchange} இல் {count} மாற்றம்',
  'planner.walkUnknown': 'நடை நேரம் மதிப்பிடப்படவில்லை (தூரத் தரவு இல்லை)',
  'planner.scheduleWindow': 'வெளியிடப்பட்ட புறப்பாடுகள் {first} முதல் {last}',
  'planner.frequency': 'சுமார் ஒவ்வொரு {minutes} நிமிடம்',
  'planner.why': 'இந்த விருப்பம் ஏன்',
  'planner.exp.quickest': 'உங்கள் வரம்புக்குள் மிகக் குறைந்த மொத்த நேரம்: சுமார் {minutes} நிமிடங்கள், {transfers} மாற்றங்கள்.',
  'planner.exp.crowd': 'சமீபத்திய பயணிகள் அறிக்கைகள் {level} நெரிசலைக் காட்டுவதாலும், அது உங்கள் நேர வரம்புக்குள் இருப்பதாலும் தேர்ந்தெடுக்கப்பட்டது.',
  'planner.exp.balanced': 'சுமார் {minutes} நிமிட பயண நேரத்தையும் அறிக்கையிடப்பட்ட {level} நெரிசலையும் சமநிலைப்படுத்துகிறது.',
  'planner.adv.singleBus': 'ஒரே பேருந்து, மாற்று ஆபத்து இல்லை',
  'planner.adv.fastest': 'தகுதியான வேகமான விருப்பம், சுமார் {minutes} நிமி',
  'planner.adv.reports': '{count} சமீபத்திய அறிக்கைகள் {level} நெரிசலை ஆதரிக்கின்றன',
  'planner.adv.calmest': 'உங்கள் வரம்புக்குள் இருக்கும் குறைந்த நெரிசல் வழித்தடம்',
  'planner.adv.direct': 'நேரடி, பேருந்து மாற்றம் இல்லை',
  'planner.dis.crowded': 'சமீபத்திய அறிக்கைகள் {level} நெரிசலைக் காட்டுகின்றன',
  'planner.dis.noReports': 'சமீபத்திய பயணிகள் அறிக்கைகள் இல்லை, எனவே நெரிசல் தெரியவில்லை',
  'planner.dis.conflicting': 'சமீபத்திய அறிக்கைகள் நெரிசல் குறித்து பொருந்தவில்லை',
  'planner.dis.limitedEvidence': 'ஒரே சமீபத்திய அறிக்கை, எனவே நெரிசல் நிலை பலவீனமான சமிக்ஞை',
  'planner.dis.slower': 'தகுதியான வேகமான விருப்பத்தை விட சுமார் {minutes} நிமி மெதுவானது',
  'planner.dis.transfer': '{count} மாற்றம் தேவை, இணைப்பு தவறலாம்',
  'planner.dis.waitUnknown': 'வெளியிடப்பட்ட அட்டவணையிலிருந்து காத்திருப்பை உறுதிப்படுத்த முடியாது',
  'planner.dq.scheduled': 'அட்டவணை மதிப்பீடு',
  'planner.dq.reports': 'சமீபத்திய பயணிகள் அறிக்கைகள் ({count})',
  'planner.dq.limited': 'குறைந்த பயணிகள் அறிக்கைகள்',
  'planner.dq.conflicting': 'முரண்பட்ட பயணிகள் அறிக்கைகள்',
  'planner.dq.noReports': 'நெரிசல் தரவு போதவில்லை',
  'planner.dq.insufficient': 'காத்திருப்பு தரவு போதவில்லை',
  'planner.assumptions.title': 'எது மதிப்பீடு, எது இல்லை',
  'planner.assumption.durations': 'பயண நேரம் ஒரு நிறுத்தத்திற்கு சுமார் {pace} நிமிடங்கள் என மதிப்பிடப்பட்டுள்ளது. தரவில் நிறுத்தங்களுக்கு இடையேயான நேரங்கள் இல்லை.',
  'planner.assumption.waiting': 'காத்திருப்பு ஒவ்வொரு வழித்தடத்தின் வெளியிடப்பட்ட மூல புறப்பாடுகளிலிருந்து, அல்லது பயணம் தொடங்கும் இடத்தில் ஏறினால் அடுத்த வெளியிடப்பட்ட புறப்பாட்டிலிருந்து வருகிறது.',
  'planner.assumption.schedule': 'தரவு முழுப் பயணத்திற்கான மூல புறப்பாட்டு நேரங்களை மட்டுமே வெளியிடுகிறது, நிறுத்த அளவிலான நேரங்கள் இல்லை.',
  'planner.assumption.walking': 'நடை நேரமும் நடை தூரமும் மதிப்பிடப்படுவதில்லை.',
  'planner.alts.title': 'மாற்று வழி',
  'planner.alts.body': 'இந்த வழிகளுக்கு SAFAR இல் உறுதிப்படுத்தப்பட்ட தரவு இல்லை, எனவே அவற்றின் நேரம், கட்டணம் அல்லது கிடைக்கும் தன்மை குறித்து எதுவும் கூறப்படவில்லை.',
  'planner.alts.metro': 'நம்ம மெட்ரோ',
  'planner.alts.metroNote': 'SAFAR இல் உறுதிப்படுத்தப்பட்ட மெட்ரோ வழித்தடம் அல்லது அட்டவணை தரவு இல்லை',
  'planner.alts.auto': 'ஆட்டோ ரிக்சா',
  'planner.alts.autoNote': 'கட்டணம் அல்லது பயண நேர தரவு இல்லை',
  'planner.alts.taxi': 'டாக்சி அல்லது ரைட்-ஹெய்லிங்',
  'planner.alts.taxiNote': 'கிடைக்கும் தன்மை, கட்டணம் அல்லது பயண நேர தரவு இல்லை',
  'planner.alts.walk': 'நடை அல்லது சைக்கிள்',
  'planner.alts.walkNote': 'தரவில் தூரங்கள் இல்லை, எனவே நடை அல்லது சைக்கிள் நேரம் காட்டப்படவில்லை',
  'planner.alts.manual': 'Google Maps இல் விருப்பங்களை ஒப்பிடுங்கள்',
};

const hi = {
  'planner.title': 'अपनी यात्रा की योजना बनाएं',
  'planner.description': 'प्रकाशित BMTC स्टॉप सूची में खोजें, अपनी समय सीमा और प्राथमिकता चुनें, और असली यात्री भीड़ रिपोर्टों के साथ सबसे उपयुक्त यात्रा देखें।',
  'planner.step.journey': 'यात्रा',
  'planner.step.limits': 'समय सीमाएँ',
  'planner.step.priority': 'प्राथमिकता',
  'planner.step.results': 'परिणाम',
  'planner.back': 'पीछे',
  'planner.next': 'आगे',
  'planner.find': 'मेरी सबसे अच्छी यात्रा खोजें',
  'planner.searching': 'प्रकाशित मार्गों की तुलना यात्री रिपोर्टों से की जा रही है…',
  'planner.retry': 'फिर कोशिश करें',
  'planner.restart': 'दूसरी यात्रा की योजना बनाएं',
  'planner.changePriorities': 'प्राथमिकता बदलें',
  'planner.err.run': 'यात्रा खोज पूरी नहीं हो सकी। अनुमान लिखने के बजाय कुछ नहीं दिखाया गया।',
  'planner.err.stops': 'प्रस्थान स्टॉप और गंतव्य चुनें।',
  'planner.err.same': 'प्रस्थान स्टॉप और गंतव्य अलग होने चाहिए।',
  'planner.err.viaSame': 'मध्यवर्ती स्टॉप प्रस्थान और गंतव्य से अलग होना चाहिए।',
  'planner.err.time': 'सही प्रस्थान तारीख और समय चुनें।',
  'planner.err.past': 'वह प्रस्थान समय बीत चुका है।',
  'planner.err.deadline': 'सही पहुँचने की समय सीमा चुनें।',
  'planner.err.deadlineBefore': 'समय सीमा आपके प्रस्थान समय के बाद होनी चाहिए।',
  'planner.err.travel': 'अधिकतम यात्रा समय चुनें या अपना समय लिखें।',
  'planner.err.wait': 'अधिकतम प्रतीक्षा समय चुनें या अपना समय लिखें।',
  'planner.err.priority': 'एक यात्रा प्राथमिकता चुनें।',
  'planner.via': 'मध्यवर्ती स्टॉप (वैकल्पिक)',
  'planner.viaPlaceholder': 'गुजरने वाला स्टॉप खोजें',
  'planner.viaHint': 'केवल वे यात्राएँ दिखाई जाती हैं जिनका मार्ग इस स्टॉप से सही क्रम में गुजरता है।',
  'planner.departTitle': 'प्रस्थान समय',
  'planner.leaveNow': 'अभी निकलें',
  'planner.chooseTime': 'तारीख और समय चुनें',
  'planner.deadlineTitle': 'पहुँचने की समय सीमा',
  'planner.deadlineToggle': 'मुझे एक निश्चित समय तक पहुँचना है',
  'planner.deadlineLabel': 'पहुँचने का अंतिम समय',
  'planner.deadlineHint': 'इसके बाद पहुँचने वाली यात्राएँ सुझाई नहीं जातीं, बल्कि हटा दी जाती हैं।',
  'planner.stopSourceNote': 'स्थान खोज डेटासेट के प्रकाशित BMTC स्टॉप नामों तक सीमित है। पते या लैंडमार्क खोज उपलब्ध नहीं है, इसलिए वही स्टॉप चुनें जहाँ आप वास्तव में चढ़ेंगे।',
  'planner.travel.title': 'यात्रा में आप अधिकतम कितना समय दे सकते हैं?',
  'planner.travel.hint': 'बस यात्रा, प्रतीक्षा और बदलाव मिलाकर। दूरी का डेटा न होने के कारण पैदल समय शामिल नहीं है।',
  'planner.wait.title': 'बस के लिए आप कितनी देर प्रतीक्षा कर सकते हैं?',
  'planner.wait.hint': 'प्रतीक्षा हर मार्ग के प्रकाशित प्रस्थान समय से अनुमानित की जाती है।',
  'planner.wait.caveat': 'यहाँ BMTC बसों के लाइव आगमन की जानकारी नहीं है, इसलिए प्रतीक्षा समय केवल प्रकाशित समय-सारणी से अनुमान है और सत्यापित नहीं किया जा सकता।',
  'planner.custom': 'अन्य',
  'planner.customLabel': 'अन्य',
  'planner.minutesWord': 'मिनट',
  'planner.minutesShort': '{count} मिनट',
  'planner.priority.title': 'यात्रा प्राथमिकता',
  'planner.priority.hint': 'एक चुनें। आपकी पसंद से बस विकल्पों का क्रम बदल जाता है।',
  'planner.priority.quickest': 'जल्द से जल्द पहुँचें',
  'planner.priority.quickestDesc': 'प्रतीक्षा और बदलाव सहित सबसे कम कुल समय। भीड़ दूसरी प्राथमिकता है, पर भारी भीड़ की चेतावनी दी जाती है।',
  'planner.priority.crowd': 'भीड़ से बचें',
  'planner.priority.crowdDesc': 'समय सीमा में रहते हुए कम भीड़ की हाल की रिपोर्ट वाले मार्गों को प्राथमिकता। बिना रिपोर्ट वाले मार्ग को कभी कम भीड़ नहीं कहा जाता।',
  'planner.priority.balanced': 'गति और आराम का संतुलन',
  'planner.priority.balancedDesc': 'कुल समय, बताई गई भीड़, प्रतीक्षा और बदलाव को साथ तौलकर अंतर स्पष्ट करता है।',
  'planner.results.summary': '{from} से {to} तक',
  'planner.results.prefs': '{depart} प्रस्थान · यात्रा सीमा {travel} मिनट · प्रतीक्षा सीमा {wait} मिनट · प्राथमिकता: {priority}',
  'planner.results.deadline': 'पहुँचने की समय सीमा {time}',
  'planner.results.count': '{count} प्रकाशित विकल्प मिले, {eligible} आपकी सीमा में हैं',
  'planner.recommended': 'हमारी सिफ़ारिश',
  'planner.limitsOk': 'यह विकल्प आपकी तय सीमा में है: {travel} मिनट तक यात्रा और {wait} मिनट तक प्रतीक्षा।',
  'planner.noMatch.title': 'आपकी शर्तें पूरी करने वाला बस विकल्प सत्यापित नहीं हो सकता',
  'planner.noMatch.body': 'उपलब्ध बस विकल्प आपके यात्रा समय, प्रतीक्षा समय या पहुँचने की समय सीमा को पूरा करते हैं, यह सत्यापित नहीं हो सकता। आप वैकल्पिक साधन पर विचार कर सकते हैं।',
  'planner.noMatch.noJourneys': 'इन दो स्टॉप को सीधे या एक बदलाव से जोड़ने वाला कोई प्रकाशित मार्ग डेटा में नहीं है। कोई दूसरा स्टॉप जोड़ा आज़माएँ।',
  'planner.alternatives': 'तुलना के विकल्प',
  'planner.option.best': 'आपकी प्राथमिकता के लिए सबसे उपयुक्त',
  'planner.option.fastest': 'योग्य सबसे तेज़ विकल्प',
  'planner.option.calm': 'योग्य सबसे कम भीड़ वाला विकल्प',
  'planner.unverified.title': 'सत्यापित नहीं हो सकते ({count})',
  'planner.unverified.body': 'इन मार्गों की प्रकाशित समय-सारणी बहुत कम है, इसलिए प्रतीक्षा अनुमानित नहीं की जा सकती; इन्हें आपकी प्रतीक्षा सीमा में बताया नहीं जाता।',
  'planner.unverified.item': 'मार्ग {routes} — लगभग {minutes} मिनट यात्रा, प्रतीक्षा असत्यापित',
  'planner.excluded.title': 'आपकी सीमाओं से हटाए गए ({count})',
  'planner.reason.serviceEnded': '{count} विकल्प उस मार्ग की प्रकाशित अंतिम यात्राओं के बाद हैं',
  'planner.reason.travelLimit': '{count} विकल्प आपके अधिकतम यात्रा समय से अधिक लेंगे',
  'planner.reason.waitLimit': '{count} विकल्पों में आपकी अनुमति से अधिक प्रतीक्षा लगेगी',
  'planner.reason.waitUnverified': '{count} विकल्प आपकी प्रतीक्षा सीमा से जाँचे नहीं जा सकते',
  'planner.reason.deadline': '{count} विकल्प आपकी समय सीमा के बाद पहुँचेंगे',
  'planner.note.crowdNoEvidence': 'आपने भीड़ से बचना चुना, पर आपकी सीमा में आने वाले मार्गों की कोई हाल की यात्री रिपोर्ट नहीं है, इसलिए भीड़ आधारित सिफ़ारिश अभी विश्वसनीय नहीं है।',
  'planner.note.noReports': 'योग्य मार्गों में किसी की भी हाल की यात्री रिपोर्ट नहीं है। यहाँ भीड़ अज्ञात है, कम नहीं।',
  'planner.note.conflicting': 'कुछ हाल की रिपोर्टें एक-दूसरे से मेल नहीं खातीं। ऐसे मार्गों को किसी के न बताए स्तर में औसत करने के बजाय परस्पर विरोधी बताया गया है।',
  'planner.note.waitUnknown': 'कुछ मार्गों की प्रकाशित समय-सारणी बहुत कम है, इसलिए प्रतीक्षा अनुमानित नहीं की जा सकती। इन्हें अलग दिखाया गया है और कभी आपकी प्रतीक्षा सीमा में नहीं बताया जाता।',
  'planner.note.allUnverified': 'किसी विकल्प को आपकी प्रतीक्षा सीमा से जाँचा नहीं जा सका। नीचे दिए विकल्प केवल जानकारी के लिए हैं।',
  'planner.crowd.title': 'यात्री भीड़ रिपोर्टें',
  'planner.crowd.none': 'कोई हाल की रिपोर्ट नहीं',
  'planner.crowd.countLabel': 'रिपोर्टें',
  'planner.crowd.count': 'पिछले 2 घंटे में {recent} · पुरानी {older}',
  'planner.crowd.latestLabel': 'नवीनतम',
  'planner.crowd.latest': '{level}, {age}',
  'planner.crowd.atStopLabel': 'आपके चढ़ने वाले स्टॉप पर',
  'planner.crowd.atStop': '{stop} पर {count} रिपोर्टें, नवीनतम {age}',
  'planner.crowd.noneNote': 'इस मार्ग की रिपोर्ट किसी यात्री ने नहीं दी है। यह अज्ञात भीड़ है, कम भीड़ नहीं।',
  'planner.crowd.limitedNote': 'इस स्तर को केवल एक हाल की रिपोर्ट का समर्थन है, इसे कमज़ोर संकेत मानें।',
  'planner.crowd.conflictNote': 'हाल की रिपोर्टें एक-दूसरे से मेल नहीं खातीं, इसलिए कोई एक भीड़ स्तर नहीं बताया जाता।',
  'planner.crowd.staleNote': 'इस मार्ग की अंतिम रिपोर्ट {age} पुरानी है। इसे केवल इतिहास के रूप में दिखाया गया है, मौजूदा भीड़ के रूप में नहीं।',
  'planner.crowd.mixedNote': 'इस यात्रा में दो बसें हैं। दिखाया गया स्तर दोनों में अधिक भीड़ वाला है।',
  'planner.crowd.conf.recent': 'हाल की रिपोर्टें',
  'planner.crowd.conf.limited': 'सीमित प्रमाण',
  'planner.crowd.conf.conflicting': 'परस्पर विरोधी रिपोर्टें',
  'planner.crowd.conf.none': 'कोई रिपोर्ट नहीं',
  'planner.totalTime': 'लगभग {minutes} मिनट',
  'planner.leg': 'चरण {n} · मार्ग {route}',
  'planner.rideStops': 'चढ़ने से उतरने के बीच {stops} स्टॉप, लगभग {minutes} मिनट यात्रा',
  'planner.waitLabel': 'प्रतीक्षा',
  'planner.waitUnverified': 'सत्यापित नहीं हो सकता',
  'planner.waitPublished': 'अगला प्रकाशित प्रस्थान {time}',
  'planner.waitHeadway': 'प्रकाशित आवृत्ति से लगभग {minutes} मिनट',
  'planner.waitFirst': 'पहली प्रकाशित यात्रा {time}',
  'planner.estimatedDeparture': 'प्रस्थान',
  'planner.estimatedArrival': 'अनुमानित आगमन',
  'planner.earliestArrival': 'सबसे जल्दी आगमन',
  'planner.rideTime': 'यात्रा',
  'planner.waitTime': 'प्रतीक्षा',
  'planner.transfersNone': 'कोई बदलाव नहीं',
  'planner.transfersCount': '{interchange} पर {count} बदलाव',
  'planner.walkUnknown': 'पैदल समय अनुमानित नहीं (दूरी डेटा नहीं)',
  'planner.scheduleWindow': 'प्रकाशित प्रस्थान {first} से {last}',
  'planner.frequency': 'लगभग हर {minutes} मिनट',
  'planner.why': 'यह विकल्प क्यों',
  'planner.exp.quickest': 'आपकी सीमा में सबसे कम कुल समय: लगभग {minutes} मिनट, {transfers} बदलाव।',
  'planner.exp.crowd': 'क्योंकि हाल की यात्री रिपोर्टें {level} भीड़ दिखाती हैं और यह आपकी समय सीमा में भी है।',
  'planner.exp.balanced': 'लगभग {minutes} मिनट की यात्रा और बताई गई {level} भीड़ के बीच संतुलन।',
  'planner.adv.singleBus': 'केवल एक बस, बदलाव का जोखिम नहीं',
  'planner.adv.fastest': 'योग्य सबसे तेज़ विकल्प, लगभग {minutes} मिनट',
  'planner.adv.reports': '{count} हाल की रिपोर्टें {level} भीड़ का समर्थन करती हैं',
  'planner.adv.calmest': 'आपकी सीमा में रहने वाला सबसे कम भीड़ वाला मार्ग',
  'planner.adv.direct': 'सीधा, बस बदलने की ज़रूरत नहीं',
  'planner.dis.crowded': 'हाल की रिपोर्टें {level} भीड़ दिखाती हैं',
  'planner.dis.noReports': 'कोई हाल की यात्री रिपोर्ट नहीं, इसलिए भीड़ अज्ञात है',
  'planner.dis.conflicting': 'भीड़ के बारे में हाल की रिपोर्टें मेल नहीं खातीं',
  'planner.dis.limitedEvidence': 'केवल एक हाल की रिपोर्ट, इसलिए भीड़ स्तर कमज़ोर संकेत है',
  'planner.dis.slower': 'योग्य सबसे तेज़ विकल्प से लगभग {minutes} मिनट धीमा',
  'planner.dis.transfer': '{count} बदलाव चाहिए, कनेक्शन छूट सकता है',
  'planner.dis.waitUnknown': 'प्रकाशित समय-सारणी से प्रतीक्षा सत्यापित नहीं हो सकती',
  'planner.dq.scheduled': 'समय-सारणी अनुमान',
  'planner.dq.reports': 'हाल की यात्री रिपोर्टें ({count})',
  'planner.dq.limited': 'सीमित यात्री रिपोर्टें',
  'planner.dq.conflicting': 'परस्पर विरोधी यात्री रिपोर्टें',
  'planner.dq.noReports': 'भीड़ डेटा अपर्याप्त',
  'planner.dq.insufficient': 'प्रतीक्षा डेटा अपर्याप्त',
  'planner.assumptions.title': 'क्या अनुमानित है और क्या अनुपलब्ध है',
  'planner.assumption.durations': 'यात्रा समय प्रति स्टॉप लगभग {pace} मिनट अनुमानित है। डेटा में स्टॉप के बीच का समय नहीं है।',
  'planner.assumption.waiting': 'प्रतीक्षा हर मार्ग की प्रकाशित मूल प्रस्थानों से, या यात्रा शुरू होने वाले स्टॉप पर चढ़ने पर अगले प्रकाशित प्रस्थान से ली जाती है।',
  'planner.assumption.schedule': 'डेटा पूरी यात्रा के मूल प्रस्थान समय ही प्रकाशित करता है, स्टॉप स्तर के समय उपलब्ध नहीं हैं।',
  'planner.assumption.walking': 'पैदल समय और पैदल दूरी अनुमानित नहीं की जाती।',
  'planner.alts.title': 'वैकल्पिक साधन',
  'planner.alts.body': 'इन साधनों के लिए SAFAR के पास सत्यापित डेटा नहीं है, इसलिए उनके समय, किराए या उपलब्धता के बारे में कुछ नहीं कहा जाता।',
  'planner.alts.metro': 'नम्मा मेट्रो',
  'planner.alts.metroNote': 'SAFAR में सत्यापित मेट्रो मार्ग या समय-सारणी डेटा लोड नहीं है',
  'planner.alts.auto': 'ऑटो रिक्शा',
  'planner.alts.autoNote': 'किराया या यात्रा समय डेटा उपलब्ध नहीं',
  'planner.alts.taxi': 'टैक्सी या राइड-हेलिंग',
  'planner.alts.taxiNote': 'उपलब्धता, किराया या यात्रा समय डेटा उपलब्ध नहीं',
  'planner.alts.walk': 'पैदल या साइकिल',
  'planner.alts.walkNote': 'डेटा में दूरियाँ नहीं हैं, इसलिए पैदल या साइकिल समय नहीं दिखाया जाता',
  'planner.alts.manual': 'Google Maps पर विकल्पों की तुलना करें',
};

export const plannerText = { en, kn, te, ta, hi };
#journey
import { base44 } from '@/api/base44Client';
import { CROWD_LEVELS, RECENT_HOURS, ageMinutes } from '@/lib/crowd';

// Crowd evidence and ranking for the trip planner. Everything here reads real
// passenger reports and real dataset estimates — nothing is invented.

export const LEVEL_ORDER = { low: 0, moderate: 1, high: 2, full: 3 };

// Transparent, stated penalties used by the balanced ranking (in minutes).
export const CROWD_PENALTY_MINUTES = { low: 0, moderate: 5, high: 15, full: 30 };
export const TRANSFER_PENALTY_MINUTES = 6;

const EVIDENCE_WINDOW_HOURS = 6;
const WEIGHT_VERY_RECENT_MIN = 30;
const WEIGHT_RECENT_MIN = 60;

const confidenceOrder = { recent: 3, limited: 2, conflicting: 1, none: 0 };
const weaker = (a, b) => (confidenceOrder[a] <= confidenceOrder[b] ? a : b);

const recencyWeight = (age) => {
  if (age <= WEIGHT_VERY_RECENT_MIN) return 3;
  if (age <= WEIGHT_RECENT_MIN) return 2;
  return 1;
};

const weightedLevel = (recent) => {
  const scores = { low: 0, moderate: 0, high: 0, full: 0 };
  recent.forEach((report) => {
    if (scores[report.crowd_level] === undefined) return;
    scores[report.crowd_level] += recencyWeight(report.age);
  });
  let best = null;
  CROWD_LEVELS.forEach((level) => {
    if (best === null || scores[level] > scores[best]) best = level;
  });
  return best && scores[best] > 0 ? best : null;
};

// Reports the user made for a route we can show options for.
export const loadRouteReports = async (routeNumbers, now = Date.now()) => {
  const unique = Array.from(new Set(routeNumbers.filter(Boolean)));
  if (unique.length === 0) return [];
  const since = new Date(now - EVIDENCE_WINDOW_HOURS * 3600000).toISOString();
  const page = await base44.entities.CrowdReport.filter(
    { reported_at: { $gte: since }, route_number: { $in: unique } },
    { sort: '-reported_at', limit: 400 },
  );
  const rows = Array.isArray(page) ? page : page?.items ?? [];
  return rows.map((report) => ({ ...report, age: ageMinutes(report.reported_at, now) }));
};

// Turns raw reports into one evidence record per route number.
export const buildEvidence = (reports) => {
  const grouped = new Map();
  reports.forEach((report) => {
    const number = report.route_number;
    if (!number) return;
    const list = grouped.get(number) ?? [];
    list.push(report);
    grouped.set(number, list);
  });

  const evidence = new Map();
  grouped.forEach((list, number) => {
    const sorted = [...list].sort((a, b) => a.age - b.age);
    const recent = sorted.filter((report) => report.age <= RECENT_HOURS * 60);
    const older = sorted.filter((report) => report.age > RECENT_HOURS * 60);
    const level = weightedLevel(recent);
    const orders = recent.map((report) => LEVEL_ORDER[report.crowd_level]).filter((value) => value !== undefined);
    const spread = orders.length > 1 ? Math.max(...orders) - Math.min(...orders) : 0;
    const weights = {};
    recent.forEach((report) => {
      weights[report.crowd_level] = (weights[report.crowd_level] ?? 0) + recencyWeight(report.age);
    });
    const topWeights = Object.values(weights).sort((a, b) => b - a);
    const tied = topWeights.length > 1 && topWeights[0] === topWeights[1];
    const conflicting = recent.length >= 2 && (spread >= 2 || tied);
    const confidence = recent.length === 0 ? 'none' : conflicting ? 'conflicting' : recent.length === 1 ? 'limited' : 'recent';

    evidence.set(number, {
      route_number: number,
      level,
      confidence,
      recentCount: recent.length,
      olderCount: older.length,
      totalCount: sorted.length,
      levelCounts: CROWD_LEVELS.map((item) => ({ level: item, count: recent.filter((r) => r.crowd_level === item).length })),
      latestAt: sorted[0]?.reported_at ?? null,
      latestLevel: sorted[0]?.crowd_level ?? null,
      latestAge: sorted[0]?.age ?? null,
      latestStop: sorted[0]?.stop_name || '',
      recentTimestamps: recent.slice(0, 3).map((report) => report.reported_at),
      reports: sorted,
    });
  });

  return evidence;
};

const emptyCrowd = {
  level: null,
  confidence: 'none',
  recentCount: 0,
  totalCount: 0,
  levelCounts: CROWD_LEVELS.map((level) => ({ level, count: 0 })),
  latestAt: null,
  latestLevel: null,
  latestAge: null,
  latestStop: '',
  recentTimestamps: [],
  atStopCount: 0,
  atStopLevel: null,
  atStopLatestAt: null,
  atStopLatestAge: null,
  atStopTimestamps: [],
  routes: [],
  mixed: false,
};

export const combineCrowd = (boardName, routeNumbers, evidence) => {
  const found = routeNumbers.map((number) => evidence.get(number)).filter(Boolean);
  if (found.length === 0) return { ...emptyCrowd, routes: routeNumbers };

  let level = null;
  let confidence = null;
  const levelCounts = new Map(CROWD_LEVELS.map((item) => [item, 0]));
  const timestamps = [];
  let recentCount = 0;
  let totalCount = 0;
  let latest = null;

  found.forEach((record) => {
    totalCount += record.totalCount;
    recentCount += record.recentCount;
    record.levelCounts.forEach((item) => levelCounts.set(item.level, (levelCounts.get(item.level) ?? 0) + item.count));
    timestamps.push(...record.recentTimestamps);
    if (record.level && (level === null || LEVEL_ORDER[record.level] > LEVEL_ORDER[level])) level = record.level;
    confidence = confidence === null ? record.confidence : weaker(confidence, record.confidence);
    if (!latest || (record.latestAge !== null && record.latestAge < latest.age)) {
      latest = { at: record.latestAt, age: record.latestAge, level: record.latestLevel, stop: record.latestStop };
    }
  });

  const atStopReports = found
    .flatMap((record) => record.reports)
    .filter((report) => report.stop_name && report.stop_name === boardName)
    .sort((a, b) => a.age - b.age);
  const atStopRecent = atStopReports.filter((report) => report.age <= RECENT_HOURS * 60);

  const orderedTimestamps = timestamps.slice().sort((a, b) => new Date(b).getTime() - new Date(a).getTime());

  return {
    level,
    confidence: confidence ?? 'none',
    recentCount,
    totalCount,
    levelCounts: CROWD_LEVELS.map((item) => ({ level: item, count: levelCounts.get(item) ?? 0 })),
    latestAt: latest?.at ?? null,
    latestLevel: latest?.level ?? null,
    latestAge: latest?.age ?? null,
    latestStop: latest?.stop ?? '',
    recentTimestamps: orderedTimestamps.slice(0, 3),
    atStopCount: atStopRecent.length,
    atStopLevel: weightedLevel(atStopRecent),
    atStopLatestAt: atStopRecent[0]?.reported_at ?? atStopReports[0]?.reported_at ?? null,
    atStopLatestAge: atStopRecent[0]?.age ?? atStopReports[0]?.age ?? null,
    atStopTimestamps: atStopRecent.slice(0, 3).map((report) => report.reported_at),
    routes: routeNumbers,
    mixed: found.length > 1,
  };
};

export const crowdPenalty = (level) => (level ? CROWD_PENALTY_MINUTES[level] ?? 0 : 0);

const rankedByTime = (list) =>
  [...list].sort((a, b) => a.journey.total_minutes - b.journey.total_minutes || a.journey.transfers - b.journey.transfers);

const rankedByCrowd = (list) =>
  [...list].sort((a, b) => {
    const levelA = a.crowd.level === null ? 99 : LEVEL_ORDER[a.crowd.level];
    const levelB = b.crowd.level === null ? 99 : LEVEL_ORDER[b.crowd.level];
    return levelA - levelB || a.journey.total_minutes - b.journey.total_minutes;
  });

const balancedCost = (option) =>
  option.journey.total_minutes + crowdPenalty(option.crowd.level) + option.journey.transfers * TRANSFER_PENALTY_MINUTES;

const rankedBalanced = (list) => [...list].sort((a, b) => balancedCost(a) - balancedCost(b));

export const rankEligible = (eligible, priority) => {
  if (priority === 'crowd') {
    const withReports = eligible.filter((option) => option.crowd.confidence !== 'none' && option.crowd.level);
    if (withReports.length > 0) {
      const sorted = rankedByCrowd(withReports);
      const rest = rankedByTime(eligible.filter((option) => !withReports.includes(option)));
      return [...sorted, ...rest];
    }
    return rankedByTime(eligible);
  }
  if (priority === 'balanced') return rankedBalanced(eligible);
  return rankedByTime(eligible);
};

// Applies the user's limits, ranks what is eligible and picks what to show.
export const evaluateJourneys = ({ journeys, evidence, prefs, departMinutes, deadlineMinutes }) => {
  const evaluated = journeys.map((journey) => {
    const reasons = [];
    if (journey.after_last_service) reasons.push('serviceEnded');
    if (prefs.maxTravelMinutes && journey.total_minutes > prefs.maxTravelMinutes) reasons.push('travelLimit');
    if (prefs.maxWaitMinutes) {
      if (!journey.wait_known) reasons.push('waitUnverified');
      else if (journey.total_wait_minutes > prefs.maxWaitMinutes) reasons.push('waitLimit');
    }
    const arrivalMinutes = Number.isFinite(departMinutes) ? departMinutes + journey.total_minutes : null;
    if (deadlineMinutes && arrivalMinutes !== null && arrivalMinutes > deadlineMinutes) reasons.push('deadline');

    return {
      journey,
      crowd: combineCrowd(journey.board_name, journey.route_numbers, evidence),
      reasons,
      eligible: reasons.length === 0,
      arrivalMinutes,
    };
  });

  // Options whose waiting cannot be verified against the waiting limit are kept
  // apart: they are never presented as fitting the limit.
  const unverified = evaluated.filter(
    (option) => !option.eligible && option.reasons.length === 1 && option.reasons[0] === 'waitUnverified',
  );
  const eligible = evaluated.filter((option) => option.eligible);
  const ranked = rankEligible(eligible, prefs.priority);
  const recommended = ranked[0] ?? null;
  const fastest = rankedByTime(eligible)[0] ?? null;
  const calm = rankedByCrowd(eligible.filter((option) => option.crowd.confidence !== 'none' && option.crowd.level))[0] ?? null;

  const picks = [];
  const add = (option, label) => {
    if (!option || picks.length >= 3) return;
    if (picks.some((item) => item.option === option)) return;
    picks.push({ option, label });
  };
  add(recommended, 'best');
  add(fastest, 'fastest');
  add(calm, 'calm');

  const notes = [];
  if (prefs.priority === 'crowd' && !calm) notes.push('crowdNoEvidence');
  if (eligible.length > 0 && eligible.every((option) => option.crowd.confidence === 'none')) notes.push('noReports');
  if (eligible.some((option) => option.crowd.confidence === 'conflicting')) notes.push('conflicting');
  if (unverified.length > 0) notes.push('waitUnknown');
  if (eligible.length === 0 && unverified.length > 0) notes.push('allUnverified');

  return {
    evaluated,
    eligible,
    unverified,
    ineligible: evaluated.filter((option) => !option.eligible && !unverified.includes(option)),
    ranked,
    recommended,
    fastest,
    calm,
    picks,
    notes,
  };
};

const countReason = (options, reason) => options.filter((option) => option.reasons.includes(reason)).length;

export const reasonSummary = (ineligible) => {
  const reasons = ['serviceEnded', 'travelLimit', 'waitLimit', 'waitUnverified', 'deadline'];
  return reasons
    .map((reason) => ({ reason, count: countReason(ineligible, reason) }))
    .filter((item) => item.count > 0);
};

// Plain-language explanation of why an option is shown, as a translation key
// plus its values. Only claims the underlying data actually supports.
export const explainOption = (option, { prefs, recommended, fastest, calm }) => {
  const { journey, crowd } = option;
  const isRecommended = option === recommended;
  const headline = `planner.exp.${prefs.priority}`;
  const vars = {
    minutes: journey.total_minutes,
    transfers: journey.transfers,
    level: crowd.level ? `report.${crowd.level}` : '',
    reports: crowd.recentCount,
  };
  const points = [];

  if (option === fastest && journey.transfers === 0) points.push({ key: 'planner.adv.singleBus' });
  if (option === fastest && rankedByTime([option]).length > 0) points.push({ key: 'planner.adv.fastest', vars: { minutes: journey.total_minutes } });
  if (crowd.level && crowd.confidence === 'recent') {
    points.push({ key: 'planner.adv.reports', vars: { count: crowd.recentCount, level: `report.${crowd.level}` } });
  }
  if (option === calm) points.push({ key: 'planner.adv.calmest' });
  if (journey.transfers === 0) points.push({ key: 'planner.adv.direct' });

  if (crowd.level && LEVEL_ORDER[crowd.level] >= 2) points.push({ key: 'planner.dis.crowded', vars: { level: `report.${crowd.level}` } });
  if (crowd.confidence === 'none') points.push({ key: 'planner.dis.noReports' });
  if (crowd.confidence === 'conflicting') points.push({ key: 'planner.dis.conflicting' });
  if (crowd.confidence === 'limited') points.push({ key: 'planner.dis.limitedEvidence' });
  if (option !== fastest && fastest) {
    const extra = journey.total_minutes - fastest.journey.total_minutes;
    if (extra > 0) points.push({ key: 'planner.dis.slower', vars: { minutes: extra } });
  }
  if (journey.transfers > 0) points.push({ key: 'planner.dis.transfer', vars: { count: journey.transfers } });
  if (!journey.wait_known) points.push({ key: 'planner.dis.waitUnknown' });

  const unique = [];
  const seen = new Set();
  points.forEach((point) => {
    if (seen.has(point.key)) return;
    seen.add(point.key);
    unique.push(point);
  });

  return {
    headline,
    headlineVars: vars,
    isRecommended,
    advantages: unique.filter((point) => point.key.startsWith('planner.adv')),
    drawbacks: unique.filter((point) => !point.key.startsWith('planner.adv')),
  };
};

export const dataQuality = (option) => {
  const { journey, crowd } = option;
  const labels = [{ key: 'planner.dq.scheduled' }];
  if (crowd.confidence === 'recent') labels.push({ key: 'planner.dq.reports', vars: { count: crowd.recentCount } });
  else if (crowd.confidence === 'limited') labels.push({ key: 'planner.dq.limited' });
  else if (crowd.confidence === 'conflicting') labels.push({ key: 'planner.dq.conflicting' });
  else labels.push({ key: 'planner.dq.noReports' });
  if (!journey.wait_known) labels.push({ key: 'planner.dq.insufficient' });
  return labels;
};

export const minutesToClock = (minutes) => {
  if (!Number.isFinite(minutes)) return '';
  const wrapped = ((Math.round(minutes) % 1440) + 1440) % 1440;
  const hours = Math.floor(wrapped / 60);
  const mins = wrapped % 60;
  return `${String(hours).padStart(2, '0')}:${String(mins).padStart(2, '0')}`;
};

export const clockToMinutes = (value) => {
  if (!value) return null;
  const parts = String(value).split(':');
  if (parts.length < 2) return null;
  const hours = Number(parts[0]);
  const mins = Number(parts[1]);
  if (!Number.isFinite(hours) || !Number.isFinite(mins)) return null;
  return hours * 60 + mins;
};

export const dateTimeToMinutes = (value) => {
  if (!value) return null;
  const date = new Date(value);
  if (Number.isNaN(date.getTime())) return null;
  return date.getHours() * 60 + date.getMinutes();
};
#LanguageContext
import { createContext, useCallback, useContext, useEffect, useMemo, useState } from 'react';
import { LANGUAGES, dictionaries, translate } from '@/lib/i18n';

const STORAGE_KEY = 'safar.language';
const LanguageContext = createContext(null);

const readStoredLanguage = () => {
  try {
    const stored = window.localStorage.getItem(STORAGE_KEY);
    if (stored && dictionaries[stored]) return stored;
  } catch (error) {
    return 'en';
  }
  return 'en';
};

export function LanguageProvider({ children }) {
  const [code, setCode] = useState(readStoredLanguage);

  useEffect(() => {
    try {
      window.localStorage.setItem(STORAGE_KEY, code);
    } catch (error) {
      // private mode — the choice simply is not remembered
    }
    document.documentElement.lang = code;
  }, [code]);

  const t = useCallback((key, vars) => translate(dictionaries[code], key, vars), [code]);

  const value = useMemo(
    () => ({ code, setCode, t, languages: LANGUAGES }),
    [code, t],
  );

  return <LanguageContext.Provider value={value}>{children}</LanguageContext.Provider>;
}

export const useLanguage = () => use
#PageNotFound
import { useLocation } from 'react-router-dom';
import { base44 } from '@/api/base44Client';
import { useQuery } from '@tanstack/react-query';


export default function PageNotFound({}) {
    const location = useLocation();
    const pageName = location.pathname.substring(1);

    const { data: authData, isFetched } = useQuery({
        queryKey: ['user'],
        queryFn: async () => {
            try {
                const user = await base44.auth.me();
                return { user, isAuthenticated: true };
            } catch (error) {
                return { user: null, isAuthenticated: false };
            }
        }
    });
    
    return (
        <div className="min-h-screen flex items-center justify-center p-6 bg-slate-50">
            <div className="max-w-md w-full">
                <div className="text-center space-y-6">
                    {/* 404 Error Code */}
                    <div className="space-y-2">
                        <h1 className="text-7xl font-light text-slate-300">404</h1>
                        <div className="h-0.5 w-16 bg-slate-200 mx-auto"></div>
                    </div>
                    
                    {/* Main Message */}
                    <div className="space-y-3">
                        <h2 className="text-2xl font-medium text-slate-800">
                            Page Not Found
                        </h2>
                        <p className="text-slate-600 leading-relaxed">
                            The page <span className="font-medium text-slate-700">"{pageName}"</span> could not be found in this application.
                        </p>
                    </div>
                    
                    {/* Admin Note */}
                    {isFetched && authData.isAuthenticated && authData.user?.role === 'admin' && (
                        <div className="mt-8 p-4 bg-slate-100 rounded-lg border border-slate-200">
                            <div className="flex items-start space-x-3">
                                <div className="flex-shrink-0 w-5 h-5 rounded-full bg-orange-100 flex items-center justify-center mt-0.5">
                                    <div className="w-2 h-2 rounded-full bg-orange-400"></div>
                                </div>
                                <div className="text-left space-y-1">
                                    <p className="text-sm font-medium text-slate-700">Admin Note</p>
                                    <p className="text-sm text-slate-600 leading-relaxed">
                                        This could mean that the AI hasn't implemented this page yet. Ask it to implement it in the chat.
                                    </p>
                                </div>
                            </div>
                        </div>
                    )}
                    
                    {/* Action Button */}
                    <div className="pt-6">
                        <button 
                            onClick={() => window.location.href = '/'} 
                            className="inline-flex items-center px-4 py-2 text-sm font-medium text-slate-700 bg-white border border-slate-200 rounded-lg hover:bg-slate-50 hover:border-slate-300 transition-colors duration-200 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-slate-500"
                        >
                            <svg className="w-4 h-4 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M3 12l2-2m0 0l7-7 7 7M5 10v10a1 1 0 001 1h3m10-11l2 2m-2-2v10a1 1 0 01-1 1h-3m-6 0a1 1 0 001-1v-4a1 1 0 011-1h2a1 1 0 011 1v4a1 1 0 001 1m-6 0h6" />
                            </svg>
                            Go Home
                        </button>
                    </div>
                </div>
            </div>
        </div>
    )
}
#query-client
import { QueryClient } from '@tanstack/react-query';


export const queryClientInstance = new QueryClient({
	defaultOptions: {
		queries: {
			refetchOnWindowFocus: false,
			retry: 1,
		},
	},
});
#transit
export const TIME_ZONE = 'Asia/Kolkata';

export const formatClock = (value) => {
  if (!value) return '';
  const parts = String(value).split(':');
  if (parts.length < 2) return String(value);
  return `${parts[0].padStart(2, '0')}:${parts[1]}`;
};

export const formatTime = (iso) => {
  if (!iso) return '';
  const date = new Date(iso);
  if (Number.isNaN(date.getTime())) return '';
  return new Intl.DateTimeFormat('en-IN', {
    timeZone: TIME_ZONE, hour: '2-digit', minute: '2-digit', hour12: false,
  }).format(date);
};

export const formatDateTime = (iso) => {
  if (!iso) return '';
  const date = new Date(iso);
  if (Number.isNaN(date.getTime())) return '';
  return new Intl.DateTimeFormat('en-IN', {
    timeZone: TIME_ZONE, day: '2-digit', month: 'short', hour: '2-digit', minute: '2-digit', hour12: false,
  }).format(date);
};

export const formatSourceDate = (value) => {
  if (!value) return '';
  const date = new Date(value);
  if (Number.isNaN(date.getTime())) return String(value);
  return new Intl.DateTimeFormat('en-IN', {
    timeZone: TIME_ZONE, day: '2-digit', month: 'short', year: 'numeric',
  }).format(date);
};

export const formatDay = (value) => {
  if (!value) return '';
  const date = new Date(value);
  if (Number.isNaN(date.getTime())) return String(value);
  return new Intl.DateTimeFormat('en-IN', {
    timeZone: TIME_ZONE, day: '2-digit', month: 'short',
  }).format(date);
};

export const lower = (value) => String(value ?? '').toLowerCase();

export const lowerKey = (value)
#utils
import { clsx } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs) {
  return twMerge(clsx(inputs))
} 


export const isIframe = window.self !== window.top;

#pages
#About
import { ExternalLink } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import { useDataset } from '@/lib/DatasetContext';
import { RECENT_HOURS } from '@/lib/crowd';
import { formatDateTime, formatSourceDate } from '@/lib/transit';

export default function About() {
  const { t } = useLanguage();
  const { dataset } = useDataset();

  const sections = [
    { title: t('about.realTitle'), body: t('about.realBody') },
    { title: t('about.reportsTitle'), body: t('about.reportsBody', { hours: RECENT_HOURS }) },
    { title: t('about.noTitle'), body: t('about.noBody') },
    { title: t('about.caveatTitle'), body: t('about.caveatBody') },
    { title: t('about.coverageTitle'), body: t('about.coverageBody') },
  ];

  return (
    <div className="mx-auto max-w-3xl space-y-8 px-4 py-12 sm:px-6 lg:px-8">
      <header className="space-y-2">
        <p className="text-[11px] font-medium uppercase tracking-[0.16em] text-muted-foreground">{t('about.eyebrow')}</p>
        <h1 className="font-display text-3xl font-semibold tracking-tight sm:text-4xl">{t('about.title')}</h1>
      </header>

      <div className="space-y-5">
        {sections.map((section) => (
          <section key={section.title} className="rounded-2xl border border-border/70 bg-card p-5">
            <h2 className="font-display text-base font-semibold">{section.title}</h2>
            <p className="mt-2 text-sm leading-relaxed text-muted-foreground">{section.body}</p>
          </section>
        ))}
      </div>

      <section className="rounded-2xl border border-border/70 bg-card p-5">
        <h2 className="font-display text-base font-semibold">{t('about.sourceTitle')}</h2>
        <p className="mt-2 text-sm leading-relaxed text-muted-foreground">{t('about.sourceBody')}</p>
        <p className="mt-3 text-xs text-muted-foreground">
          {t('about.snapshot', {
            date: formatSourceDate(dataset.commitDate) || '—',
            checked: dataset.checkedAt ? formatDateTime(dataset.checkedAt) : t('status.checking'),
          })}
        </p>
        <a
          href={dataset.sourceUrl}
          target="_blank"
          rel="noreferrer"
          className="mt-3 inline-flex items-center gap-1.5 text-sm font-medium text-primary underline-offset-4 hover:underline"
        >
          {t('about.openSource')} <ExternalLink className="h-3.5 w-3.5" />
        </a>
        <p className="mt-2 break-all text-xs text-muted-foreground">{dataset.sourceName}</p>
      </section>
    </div>
  );
}
#ForgotPassword
import React, { useState } from "react";
import { Link } from "react-router-dom";
import { base44 } from "@/api/base44Client";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Mail, ArrowLeft, Loader2 } from "lucide-react";
import AuthLayout from "@/components/AuthLayout";

export default function ForgotPassword() {
  const [email, setEmail] = useState("");
  const [loading, setLoading] = useState(false);
  const [sent, setSent] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    try {
      await base44.auth.resetPasswordRequest(email);
    } catch {
      // Always show success regardless
    } finally {
      setLoading(false);
      setSent(true);
    }
  };

  return (
    <AuthLayout
      icon={Mail}
      title="Reset password"
      subtitle="We'll send you a link to reset it"
      footer={
        <Link to="/login" className="text-primary font-medium hover:underline">
          <ArrowLeft className="w-3 h-3 inline mr-1" />Back to log in
        </Link>
      }
    >
      {sent ? (
        <p className="text-sm text-foreground text-center">
          If an account exists with that email, you'll receive a password reset link shortly.
        </p>
      ) : (
        <form onSubmit={handleSubmit} className="space-y-4">
          <div className="space-y-2">
            <Label htmlFor="email">Email address</Label>
            <div className="relative">
              <Mail className="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-muted-foreground" aria-hidden="true" />
              <Input
                id="email"
                type="email"
                autoComplete="email"
                autoFocus
                placeholder="you@example.com"
                value={email}
                onChange={(e) => setEmail(e.target.value)}
                className="pl-10 h-12"
                required
              />
            </div>
          </div>
          <Button type="submit" className="w-full h-12 font-medium" disabled={loading}>
            {loading ? (
              <>
                <Loader2 className="w-4 h-4 mr-2 animate-spin" />
                Sending...
              </>
            ) : (
              "Send reset link"
            )}
          </Button>
        </form>
      )}
    </AuthLayout>
  );
}
#Home
import { Link } from 'react-router-dom';
import { Database, MessageSquarePlus, Users } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import { useDataset } from '@/lib/DatasetContext';
import { formatDateTime, formatSourceDate } from '@/lib/transit';
import JourneyPlanner from '@/components/JourneyPlanner';
import CrowdReportsFeed from '@/components/CrowdReportsFeed';
import RouteSearchPanel from '@/components/RouteSearchPanel';
import StopSearchPanel from '@/components/StopSearchPanel';
import LiveDataNotice from '@/components/LiveDataNotice';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';

export default function Home() {
  const { t } = useLanguage();
  const { dataset, status } = useDataset();
  const count = (value) => Number(value ?? 0).toLocaleString('en-IN');

  return (
    <div>
      <section className="relative overflow-hidden border-b border-border/70">
        <div className="absolute inset-0 -z-10 bg-gradient-to-br from-primary/[0.07] via-transparent to-transparent" />
        <div className="mx-auto grid max-w-6xl gap-10 px-4 py-12 sm:px-6 lg:grid-cols-[1.15fr_0.85fr] lg:px-8 lg:py-16">
          <div>
            <span className="inline-flex items-center gap-2 rounded-full border border-primary/20 bg-primary/5 px-3 py-1 text-[11px] font-medium uppercase tracking-[0.16em] text-primary">
              <span className="h-1.5 w-1.5 rounded-full bg-primary" />
              {t('home.eyebrow')}
            </span>
            <h1 className="mt-5 font-display text-4xl font-semibold leading-[1.05] tracking-tight sm:text-5xl">
              {t('home.title')}
            </h1>
            <p className="mt-4 max-w-xl text-base leading-relaxed text-muted-foreground">{t('home.subtitle')}</p>
            <div className="mt-8">
              <JourneyPlanner />
            </div>
          </div>

          <aside className="space-y-4 self-start rounded-2xl border border-border/70 bg-card p-5">
            <h2 className="font-display text-base font-semibold">{t('status.sources')}</h2>
            <dl className="space-y-3 text-sm">
              <div className="flex items-start gap-2">
                <Database className="mt-0.5 h-4 w-4 text-crowd-low" />
                <div>
                  <dt className="font-medium">{t('status.routes')} · {t('status.stops')}</dt>
                  <dd className="text-muted-foreground">
                    {count(dataset.routeCount)} + {count(dataset.stopCount)} · {dataset.licence}
                  </dd>
                </div>
              </div>
              <div className="flex items-start gap-2">
                <Users className="mt-0.5 h-4 w-4 text-primary" />
                <div>
                  <dt className="font-medium">{t('status.reports')}</dt>
                  <dd className="text-muted-foreground">{count(dataset.reportCount)}</dd>
                </div>
              </div>
            </dl>
            <p className="text-xs text-muted-foreground">{t('about.snapshot', { date: formatSourceDate(dataset.commitDate) || '—', checked: dataset.checkedAt ? formatDateTime(dataset.checkedAt) : t('status.checking') })}</p>
            <p className="text-xs font-medium text-crowd-moderate">{t('footer.noLive')}</p>
            <Link to="/about" className="inline-block text-xs font-medium text-primary underline-offset-4 hover:underline">
              {t('nav.about')}
            </Link>
          </aside>
        </div>
      </section>

      <div className="mx-auto max-w-6xl space-y-14 px-4 py-12 sm:px-6 lg:px-8">
        <CrowdReportsFeed />

        <section className="space-y-4">
          <div className="flex flex-wrap items-end justify-between gap-3">
            <h2 className="font-display text-2xl font-semibold tracking-tight">{t('search.matched')}</h2>
            <Link
              to="/report"
              className="inline-flex items-center gap-2 rounded-xl border border-border px-4 py-2 text-sm font-medium hover:bg-accent"
            >
              <MessageSquarePlus className="h-4 w-4" />
              {t('feed.cta')}
            </Link>
          </div>
          <div className="rounded-2xl border border-border/70 bg-card p-4 sm:p-6">
            <Tabs defaultValue="routes">
              <TabsList className="mb-4">
                <TabsTrigger value="routes">{t('search.routesTab')}</TabsTrigger>
                <TabsTrigger value="stops">{t('search.stopsTab')}</TabsTrigger>
              </TabsList>
              <TabsContent value="routes">
                <RouteSearchPanel />
              </TabsContent>
              <TabsContent value="stops">
                <StopSearchPanel />
              </TabsContent>
            </Tabs>
          </div>
        </section>

        <LiveDataNotice />
      </div>
    </div>
  );
}
#Insights
import { Bar, BarChart, Cell, ResponsiveContainer, Tooltip, XAxis, YAxis } from 'recharts';
import { useLanguage } from '@/lib/LanguageContext';
import { useDataset } from '@/lib/DatasetContext';
import { levelMeta, RECENT_HOURS } from '@/lib/crowd';
import { formatDay } from '@/lib/transit';

export default function Insights() {
  const { t } = useLanguage();
  const { insights } = useDataset();

  const levelData = (insights.byLevel || []).map((row) => ({
    level: row.level,
    label: t(levelMeta(row.level).labelKey),
    count: row.count,
  }));
  const dayData = (insights.byDay || []).map((row) => ({
    label: formatDay(row.date),
    count: row.count,
  }));

  const stats = [
    { label: t('insights.total'), value: insights.total },
    { label: t('insights.last24'), value: insights.last24 },
    { label: t('feed.staleHint', { hours: RECENT_HOURS }), value: insights.stale },
  ];

  return (
    <div className="mx-auto max-w-4xl space-y-8 px-4 py-12 sm:px-6 lg:px-8">
      <header className="space-y-2">
        <p className="text-[11px] font-medium uppercase tracking-[0.16em] text-muted-foreground">{t('insights.eyebrow')}</p>
        <h1 className="font-display text-3xl font-semibold tracking-tight sm:text-4xl">{t('insights.title')}</h1>
        <p className="max-w-2xl text-sm text-muted-foreground">{t('insights.description')}</p>
      </header>

      <div className="grid gap-4 sm:grid-cols-3">
        {stats.map((stat) => (
          <div key={stat.label} className="rounded-2xl border border-border/70 bg-card p-4">
            <p className="text-xs text-muted-foreground">{stat.label}</p>
            <p className="mt-1 font-display text-2xl font-semibold">{Number(stat.value ?? 0).toLocaleString('en-IN')}</p>
          </div>
        ))}
      </div>

      {Number(insights.total ?? 0) === 0 ? (
        <div className="rounded-2xl border border-dashed border-border/80 bg-card/50 p-6 text-center text-sm text-muted-foreground">
          {t('insights.empty')}
        </div>
      ) : (
        <div className="grid gap-6 lg:grid-cols-2">
          <section className="rounded-2xl border border-border/70 bg-card p-4">
            <h2 className="mb-3 font-display text-sm font-semibold">{t('insights.byLevel')}</h2>
            <ResponsiveContainer width="100%" height={240}>
              <BarChart data={levelData}>
                <XAxis dataKey="label" tick={{ fontSize: 12 }} stroke="currentColor" />
                <YAxis allowDecimals={false} tick={{ fontSize: 12 }} stroke="currentColor" />
                <Tooltip />
                <Bar dataKey="count" radius={[6, 6, 0, 0]} fill="currentColor">
                  {levelData.map((row) => (
                    <Cell key={row.level} className={levelMeta(row.level).text} />
                  ))}
                </Bar>
              </BarChart>
            </ResponsiveContainer>
          </section>

          <section className="rounded-2xl border border-border/70 bg-card p-4">
            <h2 className="mb-3 font-display text-sm font-semibold">{t('insights.byDay')}</h2>
            <ResponsiveContainer width="100%" height={240}>
              <BarChart data={dayData}>
                <XAxis dataKey="label" tick={{ fontSize: 12 }} stroke="currentColor" />
                <YAxis allowDecimals={false} tick={{ fontSize: 12 }} stroke="currentColor" />
                <Tooltip />
                <Bar dataKey="count" radius={[6, 6, 0, 0]} className="text-primary" fill="currentColor" />
              </BarChart>
            </ResponsiveContainer>
          </section>
        </div>
      )}
    </div>
  );
}
#Login
import React, { useState } from "react";
import { Link } from "react-router-dom";
import { base44 } from "@/api/base44Client";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { LogIn, Mail, Lock, Loader2 } from "lucide-react";
import AuthLayout from "@/components/AuthLayout";
import GoogleIcon from "@/components/GoogleIcon";
import { safeReturnTo } from "@/lib/authReturnTo";

export default function Login() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);
  // Post-login destination (e.g. the MCP OAuth consent page sends users here
  // with returnTo so the grant flow can resume). Same-origin paths only.
  const returnTo = safeReturnTo();

  const handleSubmit = async (e) => {
    e.preventDefault();
    setError("");
    setLoading(true);
    try {
      await base44.auth.loginViaEmailPassword(email, password);
      window.location.href = returnTo;
    } catch (err) {
      setError(err.message || "Invalid email or password");
    } finally {
      setLoading(false);
    }
  };

  const handleGoogle = () => {
    base44.auth.loginWithProvider("google", returnTo);
  };

  return (
    <AuthLayout
      icon={LogIn}
      title="Welcome back"
      subtitle="Log in to your account"
      footer={
        <>
          Don't have an account?{" "}
          <Link
            to={"/register" + (returnTo !== "/" ? "?returnTo=" + encodeURIComponent(returnTo) : "")}
            className="text-primary font-medium hover:underline"
          >
            Create one
          </Link>
        </>
      }
    >
      <Button
        variant="outline"
        className="w-full h-12 text-sm font-medium mb-6"
        onClick={handleGoogle}
      >
        <GoogleIcon className="w-5 h-5 mr-2" />
        Continue with Google
      </Button>

      <div className="relative mb-6">
        <div className="absolute inset-0 flex items-center">
          <div className="w-full border-t border-border" />
        </div>
        <div className="relative flex justify-center text-xs uppercase">
          <span className="bg-card px-3 text-muted-foreground">or</span>
        </div>
      </div>

      {error && (
        <div className="mb-4 p-3 rounded-lg bg-destructive/10 text-destructive text-sm">
          {error}
        </div>
      )}

      <form onSubmit={handleSubmit} className="space-y-4">
        <div className="space-y-2">
          <Label htmlFor="email">Email</Label>
          <div className="relative">
            <Mail className="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-muted-foreground" aria-hidden="true" />
            <Input
              id="email"
              type="email"
              autoComplete="email"
              autoFocus
              placeholder="you@example.com"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              className="pl-10 h-12"
              required
            />
          </div>
        </div>
        <div className="space-y-2">
          <div className="flex items-center justify-between">
            <Label htmlFor="password">Password</Label>
            <Link to="/forgot-password" className="text-xs text-primary hover:underline">
              Forgot password?
            </Link>
          </div>
          <div className="relative">
            <Lock className="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-muted-foreground" aria-hidden="true" />
            <Input
              id="password"
              type="password"
              autoComplete="current-password"
              placeholder="••••••••"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              className="pl-10 h-12"
              required
            />
          </div>
        </div>
        <Button type="submit" className="w-full h-12 font-medium" disabled={loading}>
          {loading ? (
            <>
              <Loader2 className="w-4 h-4 mr-2 animate-spin" />
              Logging in...
            </>
          ) : (
            "Log in"
          )}
        </Button>
      </form>
    </AuthLayout>
  );
}
#OAuthConsent
import React, { useEffect, useState } from "react";
import { appParams } from "@/lib/app-params";
import { Button } from "@/components/ui/button";
import { ShieldCheck, Loader2 } from "lucide-react";
import AuthLayout from "@/components/AuthLayout";

const OPERATION_VERBS = { read: "view", create: "create", update: "edit", delete: "delete" };
const COLLAPSED_DATA_ROWS = 6;

// Entity tools arrive one per operation; show each entity once, naming only the
// operations it actually grants.
function groupTools(tools) {
  const operationsByEntity = new Map();
  const agents = [];
  const actions = [];
  for (const tool of tools) {
    if (tool.kind === "entity" && tool.entity) {
      if (!operationsByEntity.has(tool.entity)) operationsByEntity.set(tool.entity, []);
      operationsByEntity.get(tool.entity).push(tool.operation);
    } else if (tool.kind === "agent") {
      agents.push(tool);
    } else {
      actions.push(tool);
    }
  }
  const data = [...operationsByEntity].map(([entity, operations]) => ({
    entity,
    operations: Object.keys(OPERATION_VERBS).filter((op) => operations.includes(op)),
  }));
  return { data, agents, actions };
}

const humanize = (name) =>
  name.split(/[_-]+/).filter(Boolean).map((word) => word[0].toUpperCase() + word.slice(1)).join(" ") || name;

// "View, create and delete records" — delete is emphasized so it doesn't hide
// among harmless verbs.
function OperationSummary({ operations }) {
  return (
    <>
      {operations.map((op, i) => {
        const verb = OPERATION_VERBS[op];
        const separator = i === 0 ? "" : i === operations.length - 1 ? " and " : ", ";
        return (
          <React.Fragment key={op}>
            {separator}
            <span className={op === "delete" ? "font-medium text-destructive" : undefined}>
              {i === 0 ? verb[0].toUpperCase() + verb.slice(1) : verb}
            </span>
          </React.Fragment>
        );
      })}{" "}
      records
    </>
  );
}

function ToolSection({ title, children }) {
  return (
    <section>
      <h3 className="text-xs font-medium uppercase tracking-wide text-muted-foreground mb-2">{title}</h3>
      <ul className="divide-y divide-border rounded-lg border border-border">{children}</ul>
    </section>
  );
}

function ToolRow({ title, children }) {
  return (
    <li className="flex flex-col gap-0.5 px-3 py-2">
      <span className="font-medium text-foreground">{title}</span>
      {children && <span className="text-muted-foreground">{children}</span>}
    </li>
  );
}

// App-side OAuth consent page for the app's MCP server. The platform redirects
// AI clients here (see base44/mcp/config.json `consent_path`) with an opaque
// `ctx` handle — the authorization request itself lives on the server. This page
// gates on the app-user session, fetches the display info for that handle, shows
// the categories of access being granted, and posts the approve/deny decision.
// Do not change the fetch calls, headers, or the `ctx` handle handling — styling
// and copy are safe to edit.
export default function OAuthConsent() {
  const ctx = new URLSearchParams(window.location.search).get("ctx");
  const [info, setInfo] = useState(null);
  const [checking, setChecking] = useState(true);
  const [submitting, setSubmitting] = useState(false);
  const [decided, setDecided] = useState("");
  const [error, setError] = useState("");
  const [reconnect, setReconnect] = useState("");
  const [showAllData, setShowAllData] = useState(false);

  useEffect(() => {
    (async () => {
      let redirecting = false;
      try {
        if (!ctx) {
          setError("This authorization link is invalid or has expired.");
          return;
        }
        // Resolve the handle first: a dead handle must never render
        // approve/deny, and the response carries the app's configured login
        // route for the signed-out redirect below. Send the session (cookie +
        // bearer token) so the server can list the granted tools for a
        // signed-in user — the same auth the approve/deny call sends; without
        // it the display request is anonymous and shows no tools.
        const infoHeaders = {};
        if (appParams.token) infoHeaders.Authorization = "Bearer " + appParams.token;
        const res = await fetch(
          `/api/apps/${appParams.appId}/mcp/consent-info?handle=${encodeURIComponent(ctx)}`,
          { credentials: "include", headers: infoHeaders },
        );
        if (!res.ok) {
          setError("This authorization link is invalid or has expired.");
          return;
        }
        const data = await res.json();
        // Gate on the server's auth result, NOT base44.auth.isAuthenticated():
        // the SDK check runs the bearer path, so a cookie-only session (platform
        // login/SSO, or a private app with a stale localStorage token) would read
        // as signed-out and redirect — even though /consent-info just
        // authenticated this same request via its cookie fallback. data.authenticated
        // keeps the redirect decision in agreement with what the server returned.
        if (!data.authenticated) {
          // The short handle rides back in returnTo; login_path is
          // owner-configured and validated server-side as a same-origin path.
          // Send from_url too: a custom-auth app coerced to platform auth (e.g.
          // public_without_login under workspace SSO) serves the platform login,
          // which honors from_url rather than returnTo. Rebuild the query from
          // `ctx` alone — never forward window.location.search raw: the platform
          // resume returns from_url verbatim, so crafted extras on the consent
          // link (app_base_url, access_token, …) would ride through the login
          // round-trip and app-params.js would persist them into the freshly
          // authenticated session.
          const returnTo =
            window.location.pathname + "?ctx=" + encodeURIComponent(ctx);
          const encoded = encodeURIComponent(returnTo);
          redirecting = true; // keep the spinner while the browser navigates
          window.location.href =
            (data.login_path || "/login") + "?returnTo=" + encoded + "&from_url=" + encoded;
          return;
        }
        setInfo(data);
      } catch (e) {
        setError("Could not load this authorization request. Please try again.");
      } finally {
        if (!redirecting) setChecking(false);
      }
    })();
  }, [ctx]);

  const respond = async (action) => {
    setSubmitting(true);
    setError("");
    try {
      const headers = { "Content-Type": "application/json" };
      // Cookie-backed sessions carry no token; sending "Bearer null" would
      // shadow the valid cookie, so add the header only when a token exists.
      if (appParams.token) headers.Authorization = "Bearer " + appParams.token;
      const res = await fetch(`/api/apps/${appParams.appId}/mcp/authorize-grant`, {
        method: "POST",
        credentials: "include",
        headers,
        body: JSON.stringify({ ctx, action }),
      });
      if (!res.ok) {
        // 401 = the session expired before the (single-use, still-unconsumed)
        // handle was spent; retrying the same controls re-sends the dead session
        // forever. Send the user back through login preserving `ctx` — the same
        // redirect the initial signed-out path uses — so they can return and
        // approve the still-valid handle.
        if (res.status === 401) {
          const returnTo = window.location.pathname + "?ctx=" + encodeURIComponent(ctx);
          const encoded = encodeURIComponent(returnTo);
          window.location.href =
            ((info && info.login_path) || "/login") + "?returnTo=" + encoded + "&from_url=" + encoded;
          return;
        }
        // These all come AFTER the single-use handle is atomically consumed
        // (409 tool set changed; 403 host/resource/app mismatch; 404 access
        // gone; 400 malformed/handle already used), so retrying can only 404.
        // Show a terminal reconnect state, not an impossible "try again".
        if ([400, 403, 404, 409].includes(res.status)) {
          let detail = "";
          try { detail = (await res.json()).detail; } catch (_) { /* keep default */ }
          setReconnect(detail || "This authorization can no longer be completed. Reconnect from your AI client to try again.");
          setSubmitting(false);
          return;
        }
        throw new Error("Could not complete authorization. Please try again.");
      }
      const data = await res.json();
      window.location.href = data.redirect_url;
      if (!/^https?:/i.test(data.redirect_url)) {
        // Custom-scheme redirect (native AI clients, e.g. cursor://): browsers
        // may block or not visibly navigate, so show a terminal state instead
        // of an eternal spinner.
        setDecided(action);
        setSubmitting(false);
      }
    } catch (e) {
      setError(e.message);
      setSubmitting(false);
    }
  };

  if (checking) {
    return (
      <AuthLayout icon={ShieldCheck} title="Authorize access">
        <div className="flex items-center justify-center py-6 text-muted-foreground">
          <Loader2 className="w-5 h-5 mr-2 animate-spin" aria-hidden="true" />
          Loading…
        </div>
      </AuthLayout>
    );
  }

  const client = (info && info.client_name) || "An AI client";
  const appName = (info && info.app_name) || "this app";

  if (decided) {
    return (
      <AuthLayout
        icon={ShieldCheck}
        title={decided === "approve" ? "Access granted" : "Access denied"}
        subtitle={`You can return to ${client} and close this window.`}
      />
    );
  }

  // Terminal: the authorization request is no longer valid (tool set changed +
  // handle consumed). Retrying can't succeed, so show reconnect guidance with
  // no approve/deny controls.
  if (reconnect) {
    return (
      <AuthLayout icon={ShieldCheck} title="Reconnect required">
        <div className="p-3 rounded-lg bg-destructive/10 text-destructive text-sm">
          {reconnect}
        </div>
      </AuthLayout>
    );
  }

  // No consent details means nothing trustworthy to approve: a failed
  // consent-info load (expired handle, rate limit, transient error) renders
  // the error alone, never the approve/deny controls.
  if (error && !info) {
    return (
      <AuthLayout icon={ShieldCheck} title="Authorize access">
        <div className="p-3 rounded-lg bg-destructive/10 text-destructive text-sm">
          {error}
        </div>
      </AuthLayout>
    );
  }

  const tools = Array.isArray(info.tools) ? info.tools : [];
  const { data, agents, actions } = groupTools(tools);

  return (
    <AuthLayout
      icon={ShieldCheck}
      title="Authorize access"
      subtitle={`${client} wants to access ${appName} on your behalf`}
    >
      {error && (
        <div className="mb-4 p-3 rounded-lg bg-destructive/10 text-destructive text-sm">
          {error}
        </div>
      )}

      <p className="text-sm font-medium text-foreground mb-3">
        {tools.length ? `${client} will be able to:` : "No tools requested"}
      </p>
      {tools.length > 0 && (
        <div className="space-y-4 text-sm mb-6">
          {data.length > 0 && (
            <ToolSection title={`Data (${data.length})`}>
              {(showAllData ? data : data.slice(0, COLLAPSED_DATA_ROWS)).map(({ entity, operations }) => (
                <ToolRow key={entity} title={entity}>
                  <OperationSummary operations={operations} />
                </ToolRow>
              ))}
              {data.length > COLLAPSED_DATA_ROWS && (
                <li className="px-3 py-2">
                  <button
                    type="button"
                    className="text-sm font-medium text-primary hover:underline"
                    onClick={() => setShowAllData((shown) => !shown)}
                  >
                    {showAllData ? "Show less" : `Show all ${data.length}`}
                  </button>
                </li>
              )}
            </ToolSection>
          )}
          {agents.length > 0 && (
            <ToolSection title="Agents">
              {agents.map((tool) => (
                <ToolRow key={tool.name} title={tool.agent ? humanize(tool.agent) : tool.title || tool.name}>
                  {tool.description}
                </ToolRow>
              ))}
            </ToolSection>
          )}
          {actions.length > 0 && (
            <ToolSection title="Actions">
              {actions.map((tool) => (
                <ToolRow key={tool.name} title={tool.title || tool.name}>
                  {tool.description}
                </ToolRow>
              ))}
            </ToolSection>
          )}
        </div>
      )}

      <div className="flex gap-3">
        <Button
          variant="outline"
          className="flex-1 h-12 font-medium"
          disabled={submitting}
          onClick={() => respond("deny")}
        >
          Deny
        </Button>
        <Button
          className="flex-1 h-12 font-medium"
          disabled={submitting}
          onClick={() => respond("approve")}
        >
          {submitting ? <Loader2 className="w-4 h-4 mr-2 animate-spin" /> : null}
          Approve
        </Button>
      </div>
    </AuthLayout>
  );
}

#Plan
import { useState } from 'react';
import { Loader2 } from 'lucide-react';
import { useLanguage } from '@/lib/LanguageContext';
import { useDataset } from '@/lib/DatasetContext';
import LiveDataNotice from '@/components/LiveDataNotice';
import ImportNotice from '@/components/ImportNotice';
import PlannerStepper from '@/components/planner/PlannerStepper';
import JourneyStep from '@/components/planner/JourneyStep';
import LimitsStep from '@/components/planner/LimitsStep';
import PriorityStep from '@/components/planner/PriorityStep';
import PlanResults from '@/components/planner/PlanResults';
import { buildEvidence, dateTimeToMinutes, evaluateJourneys, loadRouteReports } from '@/lib/journey';

const emptyForm = {
  from: null,
  to: null,
  via: null,
  departMode: 'now',
  departDateTime: '',
  deadlineEnabled: false,
  deadlineDateTime: '',
  maxTravelMinutes: null,
  maxWaitMinutes: null,
  priority: null,
};

const nowMinutesOfDay = () => {
  const now = new Date();
  return now.getHours() * 60 + now.getMinutes();
};

const validDate = (value) => {
  if (!value) return null;
  const date = new Date(value);
  return Number.isNaN(date.getTime()) ? null : date;
};

export default function Plan() {
  const { t } = useLanguage();
  const { planJourneys, status } = useDataset();
  const [step, setStep] = useState(1);
  const [form, setForm] = useState(emptyForm);
  const [errors, setErrors] = useState([]);
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState(null);
  const [runError, setRunError] = useState(false);

  const update = (patch) => {
    setForm((previous) => ({ ...previous, ...patch }));
    setErrors([]);
  };

  const validateJourney = () => {
    const list = [];
    if (!form.from || !form.to) list.push('planner.err.stops');
    else if (form.from.name === form.to.name) list.push('planner.err.same');
    if (form.via && (form.via.name === form.from?.name || form.via.name === form.to?.name)) list.push('planner.err.viaSame');

    const chosenDeparture = form.departMode === 'custom' ? validDate(form.departDateTime) : new Date();
    if (form.departMode === 'custom') {
      if (!form.departDateTime || !chosenDeparture) list.push('planner.err.time');
      else if (chosenDeparture.getTime() < Date.now() - 60000) list.push('planner.err.past');
    }

    if (form.deadlineEnabled) {
      const deadline = validDate(form.deadlineDateTime);
      if (!deadline) list.push('planner.err.deadline');
      else if (chosenDeparture && deadline.getTime() <= chosenDeparture.getTime()) list.push('planner.err.deadlineBefore');
    }
    return list;
  };

  const validateLimits = () => {
    const list = [];
    if (!form.maxTravelMinutes || form.maxTravelMinutes <= 0) list.push('planner.err.travel');
    if (!form.maxWaitMinutes || form.maxWaitMinutes <= 0) list.push('planner.err.wait');
    return list;
  };

  const validatePriority = () => (form.priority ? [] : ['planner.err.priority']);

  const run = async () => {
    setLoading(true);
    setRunError(false);
    setResult(null);
    const depart = form.departMode === 'custom' ? dateTimeToMinutes(form.departDateTime) : nowMinutesOfDay();
    const deadline = form.deadlineEnabled ? dateTimeToMinutes(form.deadlineDateTime) : null;
    const prefs = {
      priority: form.priority,
      maxTravelMinutes: form.maxTravelMinutes,
      maxWaitMinutes: form.maxWaitMinutes,
    };

    try {
      const response = await planJourneys({
        from: form.from.name,
        to: form.to.name,
        via: form.via?.name,
        departMinutes: depart,
      });

      if (!response.ok || response.journeys.length === 0) {
        setResult({
          evaluated: [],
          eligible: [],
          unverified: [],
          ineligible: [],
          picks: [],
          notes: [],
          prefs,
          departMinutes: depart,
          deadlineMinutes: deadline,
          assumptions: response.assumptions,
        });
        return;
      }

      const reports = await loadRouteReports(response.journeys.flatMap((journey) => journey.route_numbers));
      const evidence = buildEvidence(reports);
      const evaluated = evaluateJourneys({
        journeys: response.journeys,
        evidence,
        prefs,
        departMinutes: depart,
        deadlineMinutes: deadline,
      });
      setResult({
        ...evaluated,
        prefs,
        departMinutes: depart,
        deadlineMinutes: deadline,
        assumptions: response.assumptions,
      });
    } catch (error) {
      setRunError(true);
    } finally {
      setLoading(false);
    }
  };

  const goNext = () => {
    const list = step === 1 ? validateJourney() : step === 2 ? validateLimits() : validatePriority();
    if (list.length > 0) {
      setErrors(list);
      return;
    }
    setErrors([]);
    if (step === 3) {
      setStep(4);
      run();
      return;
    }
    setStep(step + 1);
  };

  return (
    <div className="mx-auto max-w-4xl space-y-6 px-4 py-12 sm:px-6 lg:px-8">
      <header className="space-y-2">
        <p className="text-[11px] font-medium uppercase tracking-[0.16em] text-muted-foreground">{t('plan.eyebrow')}</p>
        <h1 className="font-display text-3xl font-semibold tracking-tight sm:text-4xl">{t('planner.title')}</h1>
        <p className="max-w-2xl text-sm text-muted-foreground">{t('planner.description')}</p>
      </header>

      <PlannerStepper
        step={step}
        onSelect={(target) => {
          setErrors([]);
          setStep(target);
        }}
      />

      {status !== 'ready' && <ImportNotice />}

      <div className="rounded-2xl border border-border/70 bg-card p-4 shadow-sm sm:p-5">
        {step === 1 && <JourneyStep form={form} update={update} errors={errors} />}
        {step === 2 && <LimitsStep form={form} update={update} errors={errors} hasError={(key) => errors.includes(key)} />}
        {step === 3 && <PriorityStep form={form} update={update} errors={errors} />}

        {step === 4 && (
          <div className="space-y-4">
            {loading && (
              <div className="flex items-center gap-2 text-sm text-muted-foreground">
                <Loader2 className="h-4 w-4 animate-spin" />
                {t('planner.searching')}
              </div>
            )}
            {!loading && runError && (
              <div className="space-y-2">
                <p className="text-sm text-destructive">{t('planner.err.run')}</p>
                <button
                  type="button"
                  onClick={run}
                  className="inline-flex items-center rounded-xl bg-primary px-4 py-2 text-sm font-medium text-primary-foreground"
                >
                  {t('planner.retry')}
                </button>
              </div>
            )}
            {!loading && !runError && result && <PlanResults result={result} form={form} onBack={() => setStep(3)} onRestart={() => setStep(1)} />}
          </div>
        )}

        {step < 4 && (
          <div className="mt-5 flex items-center justify-between gap-3">
            <button
              type="button"
              onClick={() => {
                setErrors([]);
                setStep(Math.max(1, step - 1));
              }}
              disabled={step === 1}
              className="inline-flex items-center rounded-xl border border-border px-4 py-2 text-sm font-medium text-foreground disabled:opacity-50 hover:bg-accent"
            >
              {t('planner.back')}
            </button>
            <button
              type="button"
              onClick={goNext}
              className="inline-flex items-center rounded-xl bg-primary px-4 py-2 text-sm font-medium text-primary-foreground"
            >
              {step === 3 ? t('planner.find') : t('planner.next')}
            </button>
          </div>
        )}
      </div>

      <LiveDataNotice />
    </div>
  );
}
#Register
import React, { useState } from "react";
import { Link } from "react-router-dom";
import { base44 } from "@/api/base44Client";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { UserPlus, Mail, Lock, Loader2 } from "lucide-react";
import { InputOTP, InputOTPGroup, InputOTPSlot } from "@/components/ui/input-otp";
import AuthLayout from "@/components/AuthLayout";
import GoogleIcon from "@/components/GoogleIcon";
import { toast } from "@/components/ui/use-toast";
import { safeReturnTo } from "@/lib/authReturnTo";

export default function Register() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [confirmPassword, setConfirmPassword] = useState("");
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);
  const [showOtp, setShowOtp] = useState(false);
  const [otpCode, setOtpCode] = useState("");

  const handleSubmit = async (e) => {
    e.preventDefault();
    setError("");
    if (password !== confirmPassword) {
      setError("Passwords do not match");
      return;
    }
    setLoading(true);
    try {
      await base44.auth.register({ email, password });
      setShowOtp(true);
    } catch (err) {
      setError(err.message || "Registration failed");
    } finally {
      setLoading(false);
    }
  };

  const handleVerify = async () => {
    setError("");
    setLoading(true);
    try {
      const result = await base44.auth.verifyOtp({ email, otpCode });
      if (result?.access_token) {
        base44.auth.setToken(result.access_token);
      }
      window.location.href = safeReturnTo();
    } catch (err) {
      setError(err.message || "Invalid verification code");
    } finally {
      setLoading(false);
    }
  };

  const handleResend = async () => {
    setError("");
    try {
      await base44.auth.resendOtp(email);
      toast({
        title: "Code sent",
        description: "Check your email for the new code.",
      });
    } catch (err) {
      setError(err.message || "Failed to resend code");
    }
  };

  const handleGoogle = () => {
    base44.auth.loginWithProvider("google", safeReturnTo());
  };

  if (showOtp) {
    return (
      <AuthLayout
        icon={Mail}
        title="Verify your email"
        subtitle={`We sent a code to ${email}`}
      >
        {error && (
          <div className="mb-4 p-3 rounded-lg bg-destructive/10 text-destructive text-sm">
            {error}
          </div>
        )}
        <div className="flex justify-center mb-6">
          <InputOTP
            maxLength={6}
            value={otpCode}
            onChange={setOtpCode}
            autoFocus
            autoComplete="one-time-code"
          >
            <InputOTPGroup>
              <InputOTPSlot index={0} />
              <InputOTPSlot index={1} />
              <InputOTPSlot index={2} />
              <InputOTPSlot index={3} />
              <InputOTPSlot index={4} />
              <InputOTPSlot index={5} />
            </InputOTPGroup>
          </InputOTP>
        </div>
        <Button
          className="w-full h-12 font-medium"
          onClick={handleVerify}
          disabled={loading || otpCode.length < 6}
        >
          {loading ? (
            <>
              <Loader2 className="w-4 h-4 mr-2 animate-spin" />
              Verifying...
            </>
          ) : (
            "Verify"
          )}
        </Button>
        <p className="text-center text-sm text-muted-foreground mt-4">
          Didn't receive the code?{" "}
          <button onClick={handleResend} className="text-primary font-medium hover:underline">
            Resend
          </button>
        </p>
      </AuthLayout>
    );
  }

  return (
    <AuthLayout
      icon={UserPlus}
      title="Create your account"
      subtitle="Sign up to get started"
      footer={
        <>
          Already have an account?{" "}
          <Link
            to={"/login" + (safeReturnTo() !== "/" ? "?returnTo=" + encodeURIComponent(safeReturnTo()) : "")}
            className="text-primary font-medium hover:underline"
          >
            Log in
          </Link>
        </>
      }
    >
      <Button
        variant="outline"
        className="w-full h-12 text-sm font-medium mb-6"
        onClick={handleGoogle}
      >
        <GoogleIcon className="w-5 h-5 mr-2" />
        Continue with Google
      </Button>

      <div className="relative mb-6">
        <div className="absolute inset-0 flex items-center">
          <div className="w-full border-t border-border" />
        </div>
        <div className="relative flex justify-center text-xs uppercase">
          <span className="bg-card px-3 text-muted-foreground">or</span>
        </div>
      </div>

      {error && (
        <div className="mb-4 p-3 rounded-lg bg-destructive/10 text-destructive text-sm">
          {error}
        </div>
      )}

      <form onSubmit={handleSubmit} className="space-y-4">
        <div className="space-y-2">
          <Label htmlFor="email">Email</Label>
          <div className="relative">
            <Mail className="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-muted-foreground" aria-hidden="true" />
            <Input
              id="email"
              type="email"
              autoComplete="email"
              autoFocus
              placeholder="you@example.com"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              className="pl-10 h-12"
              required
            />
          </div>
        </div>
        <div className="space-y-2">
          <Label htmlFor="password">Password</Label>
          <div className="relative">
            <Lock className="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-muted-foreground" aria-hidden="true" />
            <Input
              id="password"
              type="password"
              autoComplete="new-password"
              placeholder="••••••••"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              className="pl-10 h-12"
              required
            />
          </div>
        </div>
        <div className="space-y-2">
          <Label htmlFor="confirm">Confirm Password</Label>
          <div className="relative">
            <Lock className="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-muted-foreground" aria-hidden="true" />
            <Input
              id="confirm"
              type="password"
              autoComplete="new-password"
              placeholder="••••••••"
              value={confirmPassword}
              onChange={(e) => setConfirmPassword(e.target.value)}
              className="pl-10 h-12"
              required
            />
          </div>
        </div>
        <Button type="submit" className="w-full h-12 font-medium" disabled={loading}>
          {loading ? (
            <>
              <Loader2 className="w-4 h-4 mr-2 animate-spin" />
              Creating account...
            </>
          ) : (
            "Create account"
          )}
        </Button>
      </form>
    </AuthLayout>
  );
}

#Report
import { useLanguage } from '@/lib/LanguageContext';
import CrowdReportForm from '@/components/CrowdReportForm';
import CrowdReportsFeed from '@/components/CrowdReportsFeed';

export default function Report() {
  const { t } = useLanguage();

  return (
    <div className="mx-auto max-w-4xl space-y-8 px-4 py-12 sm:px-6 lg:px-8">
      <header className="space-y-2">
        <p className="text-[11px] font-medium uppercase tracking-[0.16em] text-muted-foreground">{t('report.eyebrow')}</p>
        <h1 className="font-display text-3xl font-semibold tracking-tight sm:text-4xl">{t('report.title')}</h1>
        <p className="max-w-2xl text-sm text-muted-foreground">{t('report.description')}</p>
      </header>

      <CrowdReportForm />
      <CrowdReportsFeed />
    </div>
  );
}
#ResetPassword
import React, { useState } from "react";
import { Link, useSearchParams } from "react-router-dom";
import { base44 } from "@/api/base44Client";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Lock, Loader2, AlertTriangle } from "lucide-react";
import AuthLayout from "@/components/AuthLayout";

export default function ResetPassword() {
  const [searchParams] = useSearchParams();
  const resetToken = searchParams.get("token");

  const [newPassword, setNewPassword] = useState("");
  const [confirmPassword, setConfirmPassword] = useState("");
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setError("");
    if (newPassword !== confirmPassword) {
      setError("Passwords do not match");
      return;
    }
    setLoading(true);
    try {
      await base44.auth.resetPassword({ resetToken, newPassword });
      window.location.href = "/login";
    } catch (err) {
      setError(err.message || "Failed to reset password");
    } finally {
      setLoading(false);
    }
  };

  if (!resetToken) {
    return (
      <AuthLayout
        icon={AlertTriangle}
        title="Invalid reset link"
        subtitle="This password reset link is missing or invalid"
        footer={
          <Link to="/forgot-password" className="text-primary font-medium hover:underline">
            Request a new link
          </Link>
        }
      >
        <p className="text-sm text-foreground text-center">
          The link you used appears to be incomplete. Please request a new password reset email.
        </p>
      </AuthLayout>
    );
  }

  return (
    <AuthLayout
      icon={Lock}
      title="New password"
      subtitle="Enter your new password below"
    >
      {error && (
        <div className="mb-4 p-3 rounded-lg bg-destructive/10 text-destructive text-sm">
          {error}
        </div>
      )}
      <form onSubmit={handleSubmit} className="space-y-4">
        <div className="space-y-2">
          <Label htmlFor="password">New Password</Label>
          <div className="relative">
            <Lock className="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-muted-foreground" aria-hidden="true" />
            <Input
              id="password"
              type="password"
              autoComplete="new-password"
              autoFocus
              placeholder="••••••••"
              value={newPassword}
              onChange={(e) => setNewPassword(e.target.value)}
              className="pl-10 h-12"
              required
            />
          </div>
        </div>
        <div className="space-y-2">
          <Label htmlFor="confirm">Confirm Password</Label>
          <div className="relative">
            <Lock className="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-muted-foreground" aria-hidden="true" />
            <Input
              id="confirm"
              type="password"
              autoComplete="new-password"
              placeholder="••••••••"
              value={confirmPassword}
              onChange={(e) => setConfirmPassword(e.target.value)}
              className="pl-10 h-12"
              required
            />
          </div>
        </div>
        <Button type="submit" className="w-full h-12 font-medium" disabled={loading}>
          {loading ? (
            <>
              <Loader2 className="w-4 h-4 mr-2 animate-spin" />
              Resetting...
            </>
          ) : (
            "Reset password"
          )}
        </Button>
      </form>
    </AuthLayout>
  );
}

#utils
#App
import { Toaster } from "@/components/ui/toaster"
import { QueryClientProvider } from '@tanstack/react-query'
import { queryClientInstance } from '@/lib/query-client'
import { BrowserRouter as Router, Route, Routes } from 'react-router-dom';
import PageNotFound from './lib/PageNotFound';
import { AuthProvider, useAuth } from '@/lib/AuthContext';
import UserNotRegisteredError from '@/components/UserNotRegisteredError';
import ScrollToTop from './components/ScrollToTop';
import { LanguageProvider } from '@/lib/LanguageContext';
import { DatasetProvider } from '@/lib/DatasetContext';
import AppLayout from '@/components/AppLayout';
import Home from '@/pages/Home';
import Plan from '@/pages/Plan';
import Report from '@/pages/Report';
import Insights from '@/pages/Insights';
import About from '@/pages/About';

const AuthenticatedApp = () => {
  const { isLoadingAuth, isLoadingPublicSettings, authError, navigateToLogin } = useAuth();

  // Show loading spinner while checking app public settings or auth
  if (isLoadingPublicSettings || isLoadingAuth) {
    return (
      <div className="fixed inset-0 flex items-center justify-center">
        <div className="w-8 h-8 border-4 border-slate-200 border-t-slate-800 rounded-full animate-spin"></div>
      </div>
    );
  }

  // Handle authentication errors
  if (authError) {
    if (authError.type === 'user_not_registered') {
      return <UserNotRegisteredError />;
    } else if (authError.type === 'auth_required') {
      // Redirect to login automatically
      navigateToLogin();
      return null;
    }
  }

  // Render the main app
  return (
    <LanguageProvider>
      <DatasetProvider>
        <Routes>
          <Route element={<AppLayout />}>
            <Route path="/" element={<Home />} />
            <Route path="/plan" element={<Plan />} />
            <Route path="/report" element={<Report />} />
            <Route path="/insights" element={<Insights />} />
            <Route path="/about" element={<About />} />
          </Route>
          <Route path="*" element={<PageNotFound />} />
        </Routes>
      </DatasetProvider>
    </LanguageProvider>
  );
};


function App() {

  return (
    <AuthProvider>
      <QueryClientProvider client={queryClientInstance}>
        <Router>
          <ScrollToTop />
          <AuthenticatedApp />
        </Router>
        <Toaster />
      </QueryClientProvider>
    </AuthProvider>
  )
}

export default App
#index
@import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@500;600;700&family=Inter:wght@400;500;600&family=Noto+Sans+Kannada:wght@400;500;600&family=Noto+Sans+Telugu:wght@400;500;600&family=Noto+Sans+Tamil:wght@400;500;600&family=Noto+Sans+Devanagari:wght@400;500;600&display=swap');

@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 0 0% 3.9%;
    --card: 0 0% 100%;
    --card-foreground: 0 0% 3.9%;
    --popover: 0 0% 100%;
    --popover-foreground: 0 0% 3.9%;
    --primary: 0 0% 9%;
    --primary-foreground: 0 0% 98%;
    --secondary: 0 0% 96.1%;
    --secondary-foreground: 0 0% 9%;
    --muted: 0 0% 96.1%;
    --muted-foreground: 0 0% 45.1%;
    --accent: 0 0% 96.1%;
    --accent-foreground: 0 0% 9%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 0 0% 98%;
    --border: 0 0% 89.8%;
    --input: 0 0% 89.8%;
    --ring: 0 0% 3.9%;
    --chart-1: 12 76% 61%;
    --chart-2: 173 58% 39%;
    --chart-3: 197 37% 24%;
    --chart-4: 43 74% 66%;
    --chart-5: 27 87% 67%;
    --radius: 0.5rem;
    --crowd-low: 152 56% 34%;
    --crowd-moderate: 38 92% 40%;
    --crowd-high: 24 88% 46%;
    --crowd-full: 0 72% 44%;
    --font-heading: 'Plus Jakarta Sans', 'Noto Sans Kannada', 'Noto Sans Telugu', 'Noto Sans Tamil', 'Noto Sans Devanagari', ui-sans-serif, system-ui, sans-serif;
    --font-body: 'Inter', 'Noto Sans Kannada', 'Noto Sans Telugu', 'Noto Sans Tamil', 'Noto Sans Devanagari', ui-sans-serif, system-ui, sans-serif;
    --font-display: 'Plus Jakarta Sans', 'Noto Sans Kannada', 'Noto Sans Telugu', 'Noto Sans Tamil', 'Noto Sans Devanagari', ui-sans-serif, system-ui, sans-serif;
    --font-mono: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
    --sidebar-background: 0 0% 98%;
    --sidebar-foreground: 240 5.3% 26.1%;
    --sidebar-primary: 240 5.9% 10%;
    --sidebar-primary-foreground: 0 0% 98%;
    --sidebar-accent: 240 4.8% 95.9%;
    --sidebar-accent-foreground: 240 5.9% 10%;
    --sidebar-border: 220 13% 91%;
    --sidebar-ring: 217.2 91.2% 59.8%;
  }

  .dark {
    --background: 0 0% 3.9%;
    --foreground: 0 0% 98%;
    --card: 0 0% 3.9%;
    --card-foreground: 0 0% 98%;
    --popover: 0 0% 3.9%;
    --popover-foreground: 0 0% 98%;
    --primary: 0 0% 98%;
    --primary-foreground: 0 0% 9%;
    --secondary: 0 0% 14.9%;
    --secondary-foreground: 0 0% 98%;
    --muted: 0 0% 14.9%;
    --muted-foreground: 0 0% 63.9%;
    --accent: 0 0% 14.9%;
    --accent-foreground: 0 0% 98%;
    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 0 0% 98%;
    --border: 0 0% 14.9%;
    --input: 0 0% 14.9%;
    --ring: 0 0% 83.1%;
    --chart-1: 220 70% 50%;
    --chart-2: 160 60% 45%;
    --chart-3: 30 80% 55%;
    --chart-4: 280 65% 60%;
    --chart-5: 340 75% 55%;
    --sidebar-background: 240 5.9% 10%;
    --sidebar-foreground: 240 4.8% 95.9%;
    --sidebar-primary: 224.3 76.3% 48%;
    --sidebar-primary-foreground: 0 0% 100%;
    --sidebar-accent: 240 3.7% 15.9%;
    --sidebar-accent-foreground: 240 4.8% 95.9%;
    --sidebar-border: 240 3.7% 15.9%;
    --sidebar-ring: 217.2 91.2% 59.8%;
    --crowd-low: 152 50% 48%;
    --crowd-moderate: 38 88% 56%;
    --crowd-high: 24 85% 58%;
    --crowd-full: 0 70% 60%;
  }
}



@layer base {
  * {
    @apply border-border outline-ring/50;
  }

  body {
    @apply bg-background text-foreground font-body;
  }
}
#main
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from '@/App.jsx'
import '@/index.css'

ReactDOM.createRoot(document.getElementById('root')).render(
  <App />
)

#gitignore
#env
.env
.env.*

# Logs
/logs
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*
lerna-debug.log*

node_modules
dist
dist-ssr
*.local

# Nitro build output, once this app has a server. Both are regenerated by
# every build, and the deploy runs one before reading them.
.output
.wrangler

# Editor directories and files
.vscode/*
!.vscode/extensions.json
.idea
.DS_Store
*.suo
*.ntvs*
*.njsproj
*.sln
*.sw?

.env
.vite

# Local link to a Base44 app, written by `base44 link`
base44/.app.jsonc

#AGENTS
# AGENTS.md

## Project Context

This is a Base44 app repository. Treat it as user-owned application code, keep changes focused on the user's request, and preserve existing project conventions.

Start with `README.md` for local setup, environment variables, and publish workflow.

## Base44 References

- CLI overview: https://docs.base44.com/developers/references/cli/get-started/overview.md
- Agent skills: https://docs.base44.com/developers/backend/overview/skills.md

If your agent supports Agent Skills, install or update Base44 skills before Base44-specific work:

```bash
npx skills add base44/skills
```

## Key Files

- `src/`: frontend application source.
- `src/api/base44Client.js`: frontend Base44 SDK client.
- `vite.config.js`: Vite config and Base44 Vite plugin setup.
- `.env.local`: local-only environment values; never commit secrets.

## Working Notes

- Use `base44 dev` as the default local development command when you need the local Base44 backend. It can run the backend and frontend together.
- When docs or code mention the frontend being started automatically, that usually means the Base44 project config includes `site.serveCommand`, for example `"serveCommand": "npm run dev"` in `base44/config.jsonc`.
- Use `npm run dev` only for frontend-only work against the hosted Base44 backend.
- Prefer the existing Base44 CLI workflow over adding new npm scripts for Base44-specific tasks.
- Reuse the existing SDK client and Vite plugin patterns before adding new Base44 integration paths.
- Run the relevant checks from `package.json` before finishing code changes.

#CLAUDE
# See AGENTS.md

Follow the instructions in `AGENTS.md`.

#components
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": false,
  "tsx": false,
  "tailwind": {
    "config": "tailwind.config.js",
    "css": "src/index.css",
    "baseColor": "neutral",
    "cssVariables": true,
    "prefix": ""
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  },
  "iconLibrary": "lucide"
}
#eslint.config
import globals from "globals";
import pluginJs from "@eslint/js";
import pluginReact from "eslint-plugin-react";
import pluginReactHooks from "eslint-plugin-react-hooks";
import pluginUnusedImports from "eslint-plugin-unused-imports";

export default [
  {
    files: [
      "src/components/**/*.{js,mjs,cjs,jsx}",
      "src/pages/**/*.{js,mjs,cjs,jsx}",
      "src/Layout.jsx",
    ],
    ignores: ["src/lib/**/*", "src/components/ui/**/*"],
    ...pluginJs.configs.recommended,
    ...pluginReact.configs.flat.recommended,
    languageOptions: {
      globals: globals.browser,
      parserOptions: {
        ecmaVersion: 2022,
        sourceType: "module",
        ecmaFeatures: {
          jsx: true,
        },
      },
    },
    settings: {
      react: {
        version: "detect",
      },
    },
    plugins: {
      react: pluginReact,
      "react-hooks": pluginReactHooks,
      "unused-imports": pluginUnusedImports,
    },
    rules: {
      "no-unused-vars": "off",
      "react/jsx-uses-vars": "error",
      "react/jsx-uses-react": "error",
      "unused-imports/no-unused-imports": "error",
      "unused-imports/no-unused-vars": [
        "warn",
        {
          vars: "all",
          varsIgnorePattern: "^_",
          args: "after-used",
          argsIgnorePattern: "^_",
        },
      ],
      "react/prop-types": "off",
      "react/react-in-jsx-scope": "off",
      "react/no-unknown-property": [
        "error",
        { ignore: ["cmdk-input-wrapper", "toast-close"] },
      ],
      "react-hooks/rules-of-hooks": "error",
    },
  },
];

#index
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="https://base44.com/logo_v2.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <link rel="manifest" href="/manifest.json" />
    <title>SAFAR — Bengaluru bus crowd levels</title>
    <meta
      name="description"
      content="Plan BMTC journeys across Bengaluru with real route, stop and timetable data, plus crowd reports from fellow commuters."
    />
    <meta name="theme-color" content="#18181b" />
    <meta property="og:title" content="SAFAR — Bengaluru bus crowd levels" />
    <meta
      property="og:description"
      content="Real BMTC route, stop and timetable data with passenger crowd reports for Bengaluru city buses."
    />
    <meta property="og:type" content="website" />
    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>

#jsconfig
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    },
    "jsx": "react-jsx",
    "module": "esnext",
    "moduleResolution": "bundler",
    "lib": ["esnext", "dom"],
    "target": "esnext",
    "checkJs": true,
    "skipLibCheck": true,
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "types": []
  },
  "include": ["src/components/**/*.js", "src/pages/**/*.jsx", "src/Layout.jsx"],
  "exclude": ["node_modules", "dist", "src/vite-plugins", "src/components/ui", "src/api", "src/lib"]
} 
#package
{
  "name": "base44-app",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "lint": "eslint . --quiet",
    "lint:fix": "eslint . --fix",
    "typecheck": "tsc -p ./jsconfig.json",
    "preview": "vite preview"
  },
  "dependencies": {
    "@base44/sdk": "^0.8.57",
    "@base44/vite-plugin": "^1.0.47",
    "@hello-pangea/dnd": "^17.0.0",
    "@hookform/resolvers": "^4.1.2",
    "@radix-ui/react-accordion": "^1.2.3",
    "@radix-ui/react-alert-dialog": "^1.1.6",
    "@radix-ui/react-aspect-ratio": "^1.1.2",
    "@radix-ui/react-avatar": "^1.1.3",
    "@radix-ui/react-checkbox": "^1.1.4",
    "@radix-ui/react-collapsible": "^1.1.3",
    "@radix-ui/react-context-menu": "^2.2.6",
    "@radix-ui/react-dialog": "^1.1.6",
    "@radix-ui/react-dropdown-menu": "^2.1.6",
    "@radix-ui/react-hover-card": "^1.1.6",
    "@radix-ui/react-label": "^2.1.2",
    "@radix-ui/react-menubar": "^1.1.6",
    "@radix-ui/react-navigation-menu": "^1.2.5",
    "@radix-ui/react-popover": "^1.1.6",
    "@radix-ui/react-progress": "^1.1.2",
    "@radix-ui/react-radio-group": "^1.2.3",
    "@radix-ui/react-scroll-area": "^1.2.3",
    "@radix-ui/react-select": "^2.1.6",
    "@radix-ui/react-separator": "^1.1.2",
    "@radix-ui/react-slider": "^1.2.3",
    "@radix-ui/react-slot": "^1.1.2",
    "@radix-ui/react-switch": "^1.1.3",
    "@radix-ui/react-tabs": "^1.1.3",
    "@radix-ui/react-toast": "^1.2.2",
    "@radix-ui/react-toggle": "^1.1.2",
    "@radix-ui/react-toggle-group": "^1.1.2",
    "@radix-ui/react-tooltip": "^1.1.8",
    "@stripe/react-stripe-js": "^3.0.0",
    "@stripe/stripe-js": "^5.2.0",
    "@tanstack/react-query": "^5.84.1",
    "canvas-confetti": "^1.9.4",
    "class-variance-authority": "^0.7.1",
    "clsx": "^2.1.1",
    "cmdk": "^1.0.0",
    "date-fns": "^3.6.0",
    "embla-carousel-react": "^8.5.2",
    "framer-motion": "^11.16.4",
    "html2canvas": "^1.4.1",
    "input-otp": "^1.4.2",
    "jspdf": "^4.2.1",
    "lodash": "^4.17.21",
    "lucide-react": "^0.475.0",
    "moment": "^2.31.0",
    "next-themes": "^0.4.4",
    "react": "^18.2.0",
    "react-day-picker": "^8.10.1",
    "react-dom": "^18.2.0",
    "react-hook-form": "^7.54.2",
    "react-hot-toast": "^2.6.0",
    "react-leaflet": "^4.2.1",
    "react-markdown": "^9.0.1",
    "react-quill-new": "^3.8.3",
    "react-resizable-panels": "^2.1.7",
    "react-router-dom": "^7.18.4",
    "recharts": "^2.15.4",
    "sonner": "^2.0.1",
    "tailwind-merge": "^3.0.2",
    "tailwindcss-animate": "^1.0.7",
    "three": "^0.171.0",
    "vaul": "^1.1.2",
    "zod": "^3.24.2"
  },
  "devDependencies": {
    "@eslint/js": "^9.19.0",
    "@types/node": "^22.13.5",
    "@types/react": "^18.2.66",
    "@types/react-dom": "^18.2.22",
    "@vitejs/plugin-react": "^6.0.5",
    "autoprefixer": "^10.4.20",
    "baseline-browser-mapping": "^2.8.32",
    "eslint": "^9.19.0",
    "eslint-plugin-react": "^7.37.4",
    "eslint-plugin-react-hooks": "^5.0.0",
    "eslint-plugin-react-refresh": "^0.4.18",
    "eslint-plugin-unused-imports": "^4.3.0",
    "globals": "^15.14.0",
    "nitro": "^3.0.260610-beta",
    "postcss": "^8.5.3",
    "tailwindcss": "^3.4.17",
    "typescript": "^5.8.2",
    "vite": "^8.2.0"
  }
}

#postcss.config
export default {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
}

#README
# Base44 Project

Use this repository to run and edit the app locally, then publish changes back through Base44.

Any change pushed to the repo will also be reflected in the Base44 Builder.

## Prerequisites

1. Clone the repository using the project's Git URL.
2. Navigate to the project directory.
3. Install dependencies: `npm install`.
4. Install the Base44 CLI: `npm install -g base44@latest`.
5. Install [Deno](https://docs.deno.com/runtime/getting_started/installation/) — the local Base44 backend runs on it.

Run `base44 --help` (or see the [CLI reference](https://docs.base44.com/developers/references/cli/commands/introduction)) for the full command surface.

## Run Locally

Three commands, from the project root:

```bash
base44 login   # one-time per machine
base44 link    # one-time per clone
base44 dev     # local backend + frontend together
```

Open the frontend URL that `base44 dev` prints (typically `http://localhost:5173`).

Notes:

- **Every fresh clone needs `base44 link`.** It writes `base44/.app.jsonc` (the app-id pointer), which is deliberately gitignored. Your app id is in the Builder URL (`app.base44.com/apps/<id>/...`); `base44 link --help` shows the non-interactive flags.
- **`base44 dev` runs the frontend for you** (via `site.serveCommand` in this repo's `base44/config.jsonc`) — never run `npm run dev` yourself: alone it serves a UI with no backend behind it (`[base44] Proxy not enabled`, every `/api` call fails), and alongside `base44 dev` the second Vite silently takes the next port and you end up looking at the wrong one.
- **The app must be published at least once for the UI to load under `base44 dev`.** The frontend boots by fetching app settings from the hosted app; before the first publish that fails and every page redirects to login. The local API works regardless.
- Entities, functions, and auth run locally — entity data is **in-memory only**, wiped when `base44 dev` restarts. Everything else (Core integrations, OAuth login) is forwarded to your deployed app. Full breakdown: [Local development overview](https://docs.base44.com/developers/backend/overview/local-dev/local-development-overview).

## Frontend Only, Hosted Backend

To work on just the frontend against your app's live hosted backend:

```bash
base44 dev --remote
```

⚠️ In this mode writes go to your app's **production data** — plain `base44 dev` keeps everything local.

## Publish Your Changes

After pushing your changes to git, open the Base44 dashboard and publish the app:

```bash
base44 dashboard open
```

This repo syncs to Base44 through git, so publish from the dashboard rather than `base44 deploy` — a CLI deploy ships your local tree directly, bypassing the sync, and the deployed state silently diverges from the repo.

## Docs & Support

GitHub integration: [https://docs.base44.com/developers/app-code/local-development/github](https://docs.base44.com/developers/app-code/local-development/github)

Local development: [https://docs.base44.com/developers/backend/overview/local-dev/local-development-overview](https://docs.base44.com/developers/backend/overview/local-dev/local-development-overview)

Support: [https://app.base44.com/support](https://app.base44.com/support)

#tailwind.config
/** @type {import('tailwindcss').Config} */
module.exports = {
    darkMode: ["class"],
    content: ["./index.html", "./src/**/*.{ts,tsx,js,jsx}"],
  theme: {
  	extend: {
  		opacity: Object.fromEntries(Array.from({ length: 101 }, (_, i) => [i, `${i / 100}`])),
  		borderRadius: {
  			lg: 'var(--radius)',
  			md: 'calc(var(--radius) - 2px)',
  			sm: 'calc(var(--radius) - 4px)'
  		},
  		colors: {
  			background: 'hsl(var(--background))',
  			foreground: 'hsl(var(--foreground))',
  			card: {
  				DEFAULT: 'hsl(var(--card))',
  				foreground: 'hsl(var(--card-foreground))'
  			},
  			popover: {
  				DEFAULT: 'hsl(var(--popover))',
  				foreground: 'hsl(var(--popover-foreground))'
  			},
  			primary: {
  				DEFAULT: 'hsl(var(--primary))',
  				foreground: 'hsl(var(--primary-foreground))'
  			},
  			secondary: {
  				DEFAULT: 'hsl(var(--secondary))',
  				foreground: 'hsl(var(--secondary-foreground))'
  			},
  			muted: {
  				DEFAULT: 'hsl(var(--muted))',
  				foreground: 'hsl(var(--muted-foreground))'
  			},
  			accent: {
  				DEFAULT: 'hsl(var(--accent))',
  				foreground: 'hsl(var(--accent-foreground))'
  			},
  			destructive: {
  				DEFAULT: 'hsl(var(--destructive))',
  				foreground: 'hsl(var(--destructive-foreground))'
  			},
  			border: 'hsl(var(--border))',
  			input: 'hsl(var(--input))',
  			ring: 'hsl(var(--ring))',
  			chart: {
  				'1': 'hsl(var(--chart-1))',
  				'2': 'hsl(var(--chart-2))',
  				'3': 'hsl(var(--chart-3))',
  				'4': 'hsl(var(--chart-4))',
  				'5': 'hsl(var(--chart-5))'
  			},
  			crowd: {
  				low: 'hsl(var(--crowd-low))',
  				moderate: 'hsl(var(--crowd-moderate))',
  				high: 'hsl(var(--crowd-high))',
  				full: 'hsl(var(--crowd-full))'
  			},
  			sidebar: {
  				DEFAULT: 'hsl(var(--sidebar-background))',
  				foreground: 'hsl(var(--sidebar-foreground))',
  				primary: 'hsl(var(--sidebar-primary))',
  				'primary-foreground': 'hsl(var(--sidebar-primary-foreground))',
  				accent: 'hsl(var(--sidebar-accent))',
  				'accent-foreground': 'hsl(var(--sidebar-accent-foreground))',
  				border: 'hsl(var(--sidebar-border))',
  				ring: 'hsl(var(--sidebar-ring))'
  			}
  		},
  		fontFamily: {
  			heading: ['var(--font-heading)'],
  			body: ['var(--font-body)'],
  			display: ['var(--font-display)'],
  			mono: ['var(--font-mono)']
  		},
  		keyframes: {
  			'accordion-down': {
  				from: {
  					height: '0'
  				},
  				to: {
  					height: 'var(--radix-accordion-content-height)'
  				}
  			},
  			'accordion-up': {
  				from: {
  					height: 'var(--radix-accordion-content-height)'
  				},
  				to: {
  					height: '0'
  				}
  			}
  		},
  		animation: {
  			'accordion-down': 'accordion-down 0.2s ease-out',
  			'accordion-up': 'accordion-up 0.2s ease-out'
  		}
  	}
  },
  plugins: [require("tailwindcss-animate")],
}

#vite.config
import base44 from "@base44/vite-plugin"
import react from '@vitejs/plugin-react'
import { defineConfig } from 'vite'

// https://vite.dev/config/
export default defineConfig({
  plugins: [
    base44({
      // Support for legacy code that imports the base44 SDK with @/integrations, @/entities, etc.
      // can be removed if the code has been updated to use the new SDK imports from @base44/sdk
      legacySDKImports: process.env.BASE44_LEGACY_SDK_IMPORTS === 'true',
      hmrNotifier: true,
      navigationNotifier: true,
      analyticsTracker: true,
      visualEditAgent: true
    }),
    react(),
  ]
})


