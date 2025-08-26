# MVP Specification: Slot Grid System (15/30/45/60 min)
## Hybrid Architecture with Automerge CRDT

## Overview
An unconference scheduling system using:
- **Static HTML/CSS/TypeScript** (no framework)
- **Automerge** for conflict-free replicated data types (CRDT)
- **IndexedDB** for local Automerge document persistence
- **Automerge Sync Server** for real-time collaboration via WebSocket
- **Minimal server** - only handles sync protocol, no business logic

## Scope & Features

### Grid Format
- **Display**: Rooms × Time slots grid
- **Time slots**: 15-minute increments
- **Session durations**: 15, 30, 45, or 60 minutes (1-4 slots)
- **Visual**: Sessions span multiple slots vertically in their assigned room column

### User Capabilities

#### All Participants
- Propose new sessions with title, description, tags
- Edit/delete own sessions (if not scheduled)
- Mark interest in any session (simple counter)
- Leave comments on sessions
- View real-time grid updates

#### Organizers (identified by passphrase/local flag)
- Place sessions on the grid (drag & drop)
- Move scheduled sessions to different slots/rooms
- Cancel/unschedule sessions
- Resolve scheduling conflicts
- Manage rooms and time slots

## Core Business Rules

1. **Grid Generation**: Board auto-generates 15-min slots between event start/end times
2. **Placement**: Session = (roomId, startSlotIndex, spanCount ∈ {1,2,3,4})
3. **No Overlaps**: New placement must not conflict with existing sessions in same room
4. **Conflict Resolution**: Automerge handles concurrent edits automatically via CRDT merge
5. **Interest Tracking**: Simple counter using Automerge.Counter for automatic conflict resolution
6. **Capacity Hints**: Show room capacity vs. interest (informational only in MVP)

## Data Architecture (Automerge + IndexedDB)

### Automerge Document Schema
```typescript
// Main Automerge document structure
import * as Automerge from '@automerge/automerge';

// The entire event state is one Automerge document

type EventDoc = {
  event: {
    id: string;
    name: Automerge.Text;  // Collaborative text editing
    startsAt: number;      // Unix timestamp
    endsAt: number;
    baseSlotMinutes: number;
  };
  
  rooms: Record<string, {
    id: string;
    name: Automerge.Text;
    capacity: number;
    order: number;
  }>;
  
  timeSlots: Record<string, {
    id: string;
    startTime: number;
    index: number;
    label: string;
  }>;
  
  sessions: Record<string, {
    id: string;
    proposerId: string;
    title: Automerge.Text;
    description: Automerge.Text;
    tags: string[];
    expectedSize: number;
    state: 'PROPOSED' | 'SCHEDULED' | 'IN_PROGRESS' | 'FINISHED' | 'CANCELLED';
    interestCount: Automerge.Counter;  // Auto-merges concurrent changes
    createdAt: number;
    updatedAt: number;
  }>;
  
  placements: Record<string, {
    id: string;
    sessionId: string;
    roomId: string;
    startSlotIndex: number;
    spanSlots: 1 | 2 | 3 | 4;
    placedBy: string;
    placedAt: number;
  }>;
  
  interests: Record<string, {  // Key: `${sessionId}-${userId}`
    sessionId: string;
    userId: string;
    markedAt: number;
  }>;
  
  users: Record<string, {
    id: string;
    name: Automerge.Text;
    isOrganizer: boolean;
    createdAt: number;
  }>;
  
  comments: Automerge.Table<{
    id: string;
    sessionId: string;
    userId: string;
    text: Automerge.Text;
    createdAt: number;
  }>;
}
```

### IndexedDB Storage (for Automerge Documents)
```typescript
// IndexedDB is used only to persist Automerge document snapshots
const DB_NAME = 'unconference-automerge-db';
const DB_VERSION = 1;

interface StoredDoc {
  id: string;           // Event ID
  binary: Uint8Array;   // Automerge.save(doc)
  timestamp: number;    // Last sync time
}

// Simple key-value store for Automerge documents
const initDB = () => {
  const request = indexedDB.open(DB_NAME, DB_VERSION);
  
  request.onupgradeneeded = (event) => {
    const db = event.target.result;
    
    // Store for document snapshots
    if (!db.objectStoreNames.contains('documents')) {
      db.createObjectStore('documents', { keyPath: 'id' });
    }
    
    // Store for sync metadata
    if (!db.objectStoreNames.contains('syncState')) {
      db.createObjectStore('syncState', { keyPath: 'key' });
    }
  };
  
  return request;
};
```

## Key Flows

### 1. Initial Load & Setup with Automerge
```typescript
import * as Automerge from '@automerge/automerge';
import { Repo } from '@automerge/automerge-repo';
import { BrowserWebSocketClientAdapter } from '@automerge/automerge-repo-network-websocket';
import { IndexedDBStorageAdapter } from '@automerge/automerge-repo-storage-indexeddb';

let repo: Repo;
let handle: DocHandle<EventDoc>;

async function initializeApp() {
  // 1. Check/create local user
  const userId = localStorage.getItem('userId') || generateUUID();
  localStorage.setItem('userId', userId);
  
  // 2. Initialize Automerge Repo with IndexedDB persistence
  repo = new Repo({
    storage: new IndexedDBStorageAdapter(),
    network: [new BrowserWebSocketClientAdapter('ws://localhost:3030')],
  });
  
  // 3. Get or create event document
  const eventId = getEventIdFromURL() || 'default-event';
  handle = repo.find<EventDoc>(eventId);
  
  // 4. Wait for document to sync
  await handle.whenReady();
  
  // 5. Initialize document if empty
  if (!handle.docSync()) {
    handle.change((doc) => {
      doc.event = {
        id: eventId,
        name: new Automerge.Text('Unconference 2024'),
        startsAt: Date.now(),
        endsAt: Date.now() + 8 * 60 * 60 * 1000,
        baseSlotMinutes: 15
      };
      doc.rooms = {};
      doc.timeSlots = {};
      doc.sessions = {};
      doc.placements = {};
      doc.interests = {};
      doc.users = {};
      doc.comments = new Automerge.Table();
      
      // Generate initial time slots
      generateTimeSlots(doc);
    });
  }
  
  // 6. Subscribe to changes
  handle.on('change', ({ doc }) => {
    renderGrid(doc);
  });
  
  // 7. Set up UI
  setupDragDrop();
  renderGrid(handle.docSync());
}
```

### 2. Place Session on Grid with Automerge
```typescript
function placeSession(
  sessionId: string,
  roomId: string,
  startSlotIndex: number,
  spanSlots: 1 | 2 | 3 | 4
): boolean {
  // Automerge change is atomic and will sync automatically
  handle.change((doc) => {
    // 1. Check for conflicts locally
    const conflicts = checkOverlaps(doc, roomId, startSlotIndex, spanSlots);
    if (conflicts.length > 0) {
      // Don't make the change
      return false;
    }
    
    // 2. Remove any existing placement for this session
    for (const [placementId, placement] of Object.entries(doc.placements)) {
      if (placement.sessionId === sessionId) {
        delete doc.placements[placementId];
      }
    }
    
    // 3. Create new placement
    const placementId = generateUUID();
    doc.placements[placementId] = {
      id: placementId,
      sessionId,
      roomId,
      startSlotIndex,
      spanSlots,
      placedBy: getCurrentUserId(),
      placedAt: Date.now()
    };
    
    // 4. Update session state
    if (doc.sessions[sessionId]) {
      doc.sessions[sessionId].state = 'SCHEDULED';
      doc.sessions[sessionId].updatedAt = Date.now();
    }
  });
  
  // UI will auto-update via the change handler
  return true;
}

// Helper to create a new session
function createSession(title: string, description: string, tags: string[]) {
  const sessionId = generateUUID();
  
  handle.change((doc) => {
    doc.sessions[sessionId] = {
      id: sessionId,
      proposerId: getCurrentUserId(),
      title: new Automerge.Text(title),
      description: new Automerge.Text(description),
      tags,
      expectedSize: 0,
      state: 'PROPOSED',
      interestCount: new Automerge.Counter(),
      createdAt: Date.now(),
      updatedAt: Date.now()
    };
  });
  
  return sessionId;
}
```

### 3. Conflict Detection with Automerge
```typescript
function checkOverlaps(
  doc: EventDoc,
  roomId: string,
  startSlotIndex: number,
  spanSlots: number
): Array<typeof doc.placements[string]> {
  const endIndex = startSlotIndex + spanSlots;
  const conflicts = [];
  
  // Check all placements in the same room
  for (const placement of Object.values(doc.placements)) {
    if (placement.roomId !== roomId) continue;
    
    const pEndIndex = placement.startSlotIndex + placement.spanSlots;
    
    // Check if time ranges overlap
    if (!(endIndex <= placement.startSlotIndex || startSlotIndex >= pEndIndex)) {
      conflicts.push(placement);
    }
  }
  
  return conflicts;
}

// Mark interest in a session
function toggleInterest(sessionId: string) {
  const userId = getCurrentUserId();
  const interestKey = `${sessionId}-${userId}`;
  
  handle.change((doc) => {
    if (doc.interests[interestKey]) {
      // Remove interest
      delete doc.interests[interestKey];
      doc.sessions[sessionId]?.interestCount.decrement(1);
    } else {
      // Add interest
      doc.interests[interestKey] = {
        sessionId,
        userId,
        markedAt: Date.now()
      };
      doc.sessions[sessionId]?.interestCount.increment(1);
    }
  });
}
```

## UI Components (Vanilla TypeScript)

### Grid Structure
```html
<div class="schedule-grid">
  <div class="grid-header">
    <div class="time-header"></div>
    <div class="room-header" data-room-id="...">Room A</div>
    <div class="room-header" data-room-id="...">Room B</div>
  </div>
  <div class="grid-body">
    <div class="time-column">
      <div class="time-slot" data-slot-id="...">9:00 AM</div>
      <div class="time-slot" data-slot-id="...">9:15 AM</div>
    </div>
    <div class="room-column" data-room-id="...">
      <div class="slot-cell" data-slot-id="..." data-room-id="...">
        <!-- Sessions rendered here -->
      </div>
    </div>
  </div>
</div>
```

### Session Card
```typescript
function renderSessionCard(session: Session, placement?: Placement): HTMLElement {
  const card = document.createElement('div');
  card.className = 'session-card';
  card.dataset.sessionId = session.id;
  
  if (placement) {
    card.classList.add('scheduled');
    card.style.gridRow = `span ${placement.spanSlots}`;
  }
  
  card.innerHTML = `
    <h4>${escapeHtml(session.title)}</h4>
    <div class="session-meta">
      <span class="duration">${placement.spanSlots * 15} min</span>
      <span class="interest-count">
        <button class="interest-btn" data-session-id="${session.id}">
          ♥ <span class="count">0</span>
        </button>
      </span>
    </div>
    <div class="session-tags">
      ${session.tags.map(t => `<span class="tag">${escapeHtml(t)}</span>`).join('')}
    </div>
  `;
  
  return card;
}
```

## Data Sync Strategy with Automerge

### Automerge Sync Server Setup
```typescript
// Server (Node.js) - minimal sync server
import { WebSocketServer } from 'ws';
import { NodeWSServerAdapter } from '@automerge/automerge-repo-network-websocket';
import { Repo } from '@automerge/automerge-repo';
import { NodeFSStorageAdapter } from '@automerge/automerge-repo-storage-nodefs';

// Create server with persistent storage
const wss = new WebSocketServer({ port: 3030 });

const serverRepo = new Repo({
  network: [new NodeWSServerAdapter(wss)],
  storage: new NodeFSStorageAdapter('./sync-data'),
});

console.log('Automerge sync server running on ws://localhost:3030');
```

### Client Connection & Sync
```typescript
// Client automatically syncs via WebSocket
class AutomergeSync {
  private repo: Repo;
  private handle: DocHandle<EventDoc>;
  private syncState: 'connecting' | 'connected' | 'disconnected' = 'connecting';
  
  async connect(eventId: string, serverUrl: string = 'ws://localhost:3030') {
    // Initialize repo with network adapter
    this.repo = new Repo({
      storage: new IndexedDBStorageAdapter(),
      network: [new BrowserWebSocketClientAdapter(serverUrl)],
    });
    
    // Get document handle
    this.handle = this.repo.find<EventDoc>(eventId);
    
    // Monitor connection state
    this.repo.networkSubsystem.on('peer', (peer) => {
      peer.on('peer-candidate', () => {
        this.syncState = 'connecting';
        this.updateSyncIndicator();
      });
      
      peer.on('peer-connected', () => {
        this.syncState = 'connected';
        this.updateSyncIndicator();
      });
      
      peer.on('peer-disconnected', () => {
        this.syncState = 'disconnected';
        this.updateSyncIndicator();
        // Automerge will auto-reconnect
      });
    });
    
    return this.handle;
  }
  
  // Export current state
  exportSnapshot(): Uint8Array {
    const doc = this.handle.docSync();
    return Automerge.save(doc);
  }
  
  // Import from snapshot
  async importSnapshot(binary: Uint8Array) {
    const importedDoc = Automerge.load<EventDoc>(binary);
    
    // Merge with current document
    this.handle.change((doc) => {
      Automerge.merge(doc, importedDoc);
    });
  }
  
  private updateSyncIndicator() {
    const indicator = document.getElementById('sync-status');
    if (indicator) {
      indicator.className = `sync-${this.syncState}`;
      indicator.title = `Sync: ${this.syncState}`;
    }
  }
}
```

### Offline Support & Conflict Resolution
```typescript
// Automerge automatically handles offline/online transitions
class OfflineManager {
  private handle: DocHandle<EventDoc>;
  
  constructor(handle: DocHandle<EventDoc>) {
    this.handle = handle;
    
    // Monitor online/offline status
    window.addEventListener('online', () => this.handleOnline());
    window.addEventListener('offline', () => this.handleOffline());
  }
  
  private handleOffline() {
    // Automerge continues to work offline
    // Changes are queued for sync
    showNotification('Working offline - changes will sync when reconnected');
  }
  
  private handleOnline() {
    // Automerge automatically syncs pending changes
    showNotification('Back online - syncing changes...');
  }
  
  // Manual conflict resolution (rare with CRDTs)
  resolveConflicts() {
    this.handle.change((doc) => {
      // Check for placement conflicts that may have emerged during merge
      const placementsByRoom = new Map<string, typeof doc.placements[string][]>();
      
      for (const placement of Object.values(doc.placements)) {
        const room = placementsByRoom.get(placement.roomId) || [];
        room.push(placement);
        placementsByRoom.set(placement.roomId, room);
      }
      
      // Detect and resolve overlaps
      for (const [roomId, placements] of placementsByRoom) {
        const sorted = placements.sort((a, b) => a.startSlotIndex - b.startSlotIndex);
        
        for (let i = 0; i < sorted.length - 1; i++) {
          const current = sorted[i];
          const next = sorted[i + 1];
          
          if (current.startSlotIndex + current.spanSlots > next.startSlotIndex) {
            // Conflict detected - keep the one placed more recently
            if (current.placedAt > next.placedAt) {
              delete doc.placements[next.id];
            } else {
              delete doc.placements[current.id];
            }
          }
        }
      }
    });
  }
}
```

## Deployment

### Static Files Structure
```
/
├── index.html
├── styles.css
├── app.js            (compiled from TypeScript)
├── app.js.map
└── assets/
    └── favicon.ico
```

### Hosting Options
1. **GitHub Pages**: Free, simple, supports custom domains
2. **Netlify/Vercel**: Free tier, automatic deploys from git
3. **S3 + CloudFront**: Scalable, global CDN
4. **Any static web server**: nginx, Apache, etc.

### Build Process
```json
// package.json
{
  "scripts": {
    "build": "tsc && esbuild src/app.ts --bundle --outfile=dist/app.js",
    "watch": "tsc --watch & esbuild src/app.ts --bundle --watch --outfile=dist/app.js",
    "serve": "python3 -m http.server 8000 --directory dist",
    "sync-server": "node sync-server.js"
  },
  "dependencies": {
    "@automerge/automerge": "^2.1.0",
    "@automerge/automerge-repo": "^1.0.0",
    "@automerge/automerge-repo-network-websocket": "^1.0.0",
    "@automerge/automerge-repo-storage-indexeddb": "^1.0.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "esbuild": "^0.19.0",
    "@types/node": "^20.0.0"
  }
}
```

## Security Considerations

1. **No Authentication**: MVP relies on local storage for user identity
2. **Trust Model**: All clients trusted (sync server doesn't validate)
3. **Data Integrity**: Automerge CRDTs ensure eventual consistency
4. **Privacy**: Data synced through server (use WSS/TLS in production)
5. **XSS Prevention**: Escape all user input when rendering
6. **Sync Server**: Should use authentication tokens in production

## Future Enhancements

1. **Authentication**: Add auth tokens to sync server connection
2. **Permissions**: Implement capability-based access control in Automerge
3. **Advanced Sync**: 
   - Selective sync (only active event)
   - Compression for large documents
   - Periodic snapshots for faster initial sync
4. **Advanced Features**: 
   - Waitlists when sessions full
   - Automatic scheduling suggestions
   - Export to calendar formats
   - Session recordings/notes
5. **PWA**: Service workers for better offline support

## Implementation Timeline

### Phase 1: Core (2-3 days)
- IndexedDB setup and data layer
- Basic grid rendering
- Session creation/editing
- Drag & drop placement

### Phase 2: Interactions (1-2 days)
- Interest marking
- Comments
- Conflict detection
- Local storage of user preferences

### Phase 3: Polish (1 day)
- Responsive design
- Keyboard navigation
- Export/import data
- Error handling

### Phase 4: Real-time Sync (2 days)
- Automerge sync server setup
- WebSocket connection management
- Offline queue handling
- Sync status indicators