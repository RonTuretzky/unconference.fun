# Unconference Board Specification

## Introduction

This specification defines a web-based unconference board system that enables participants to collaboratively organize and schedule sessions in real-time. The platform supports the organic, participant-driven nature of unconferences by providing intuitive tools for session proposal, scheduling, and interest tracking.

### Core Concept

An unconference board displays available time slots and rooms in a grid format, allowing participants to:
- Propose sessions with descriptions and expected audience size
- Express interest in sessions to help guide scheduling decisions
- Collaboratively place sessions on the schedule through organizer moderation
- View real-time updates as the board evolves

### Key Principles

- **Collaborative**: Participants can propose and show interest; organizers facilitate placement
- **Real-time**: Changes propagate immediately to all connected users
- **Conflict-free**: System prevents double-booking and scheduling conflicts
- **Flexible**: Supports variable session lengths (15, 30, 45, 60 minutes)
- **Accessible**: Works across devices with clear visual feedback

## MVP Architecture (Static Site)

The MVP implementation uses a static site architecture with client-side state management and browser storage for persistence. This approach eliminates server dependencies while providing core functionality for small to medium unconference events.

### Technical Stack

- **Frontend**: Vanilla HTML5, CSS3, JavaScript (ES6+)
- **Storage**: localStorage for persistence, sessionStorage for temporary state
- **Real-time**: Shared browser storage + periodic sync for multi-tab scenarios
- **Export**: JSON download for backup/sharing

### Data Model

The client-side data model uses JavaScript objects stored in localStorage:

```javascript
// Event configuration
const event = {
  id: 'event-uuid',
  name: 'My Unconference 2024',
  startsAt: '2024-03-15T09:00:00Z',
  endsAt: '2024-03-15T17:00:00Z',
  baseSlotMin: 15,
  rooms: [
    { id: 'room1', name: 'Main Hall', capacity: 50 },
    { id: 'room2', name: 'Workshop Room', capacity: 20 }
  ]
};

// Session data structure
const session = {
  id: 'session-uuid',
  proposerId: 'user-uuid',
  title: 'Session Title',
  description: 'Detailed description...',
  tags: ['javascript', 'architecture'],
  expectedSize: 15,
  state: 'PROPOSED', // PROPOSED, SCHEDULED, CANCELLED
  placement: {
    roomId: 'room1',
    startSlotIndex: 4, // 0-based 15-min slots from event start
    spanSlots: 2 // 1=15min, 2=30min, 3=45min, 4=60min
  },
  interests: ['user1', 'user2', 'user3'],
  comments: [
    { userId: 'user1', text: 'Great topic!', timestamp: '...' }
  ]
};
```

### Core Components

#### 1. Board Grid Component (`board.js`)
```html
<div class="board-container">
  <div class="time-header">
    <!-- 15-minute time slot headers -->
  </div>
  <div class="rooms-grid">
    <div class="room-row" data-room-id="room1">
      <div class="room-label">Main Hall (50)</div>
      <div class="time-slots">
        <div class="slot" data-slot="0"></div>
        <div class="slot" data-slot="1"></div>
        <!-- ... more slots -->
      </div>
    </div>
  </div>
</div>
```

#### 2. Session Proposal Form (`proposal.js`)
```html
<form class="session-form" id="proposal-form">
  <input type="text" name="title" placeholder="Session Title" required>
  <textarea name="description" placeholder="What will you discuss?"></textarea>
  <input type="number" name="expectedSize" placeholder="Expected attendees" min="1">
  <input type="text" name="tags" placeholder="Tags (comma-separated)">
  <select name="duration">
    <option value="1">15 minutes</option>
    <option value="2">30 minutes</option>
    <option value="3">45 minutes</option>
    <option value="4">60 minutes</option>
  </select>
  <button type="submit">Propose Session</button>
</form>
```

#### 3. Session Card Component (`session-card.js`)
```html
<div class="session-card" data-session-id="...">
  <h3 class="session-title">Session Title</h3>
  <p class="session-description">Description...</p>
  <div class="session-meta">
    <span class="duration">30 min</span>
    <span class="expected-size">~15 people</span>
    <span class="interest-count">d 8</span>
  </div>
  <div class="session-actions">
    <button class="interest-btn">Show Interest</button>
    <button class="comment-btn">Comment</button>
  </div>
</div>
```

### State Management

#### LocalStorage Schema
```javascript
// Main application state
localStorage.setItem('unconference-board', JSON.stringify({
  event: { /* event config */ },
  sessions: { /* sessionId: session object */ },
  users: { /* userId: user profile */ },
  lastSync: timestamp
}));

// User session
localStorage.setItem('unconference-user', JSON.stringify({
  id: 'user-uuid',
  name: 'User Name',
  email: 'user@example.com',
  role: 'participant' // or 'organizer'
}));
```

#### Core Functions
```javascript
// Session management
function createSession(sessionData) {
  const session = {
    id: generateUUID(),
    ...sessionData,
    state: 'PROPOSED',
    interests: [],
    comments: []
  };
  saveSession(session);
  broadcastUpdate('session-created', session);
}

function placeSession(sessionId, roomId, startSlot, spanSlots) {
  if (!canPlace(roomId, startSlot, spanSlots)) {
    throw new Error('Placement conflicts detected');
  }
  
  const session = getSession(sessionId);
  session.placement = { roomId, startSlot, spanSlots };
  session.state = 'SCHEDULED';
  saveSession(session);
  broadcastUpdate('session-placed', session);
}

// Conflict detection
function canPlace(roomId, startSlot, spanSlots) {
  const sessions = getAllSessions();
  return !sessions.some(session => {
    if (!session.placement || session.placement.roomId !== roomId) {
      return false;
    }
    
    const existingStart = session.placement.startSlot;
    const existingEnd = existingStart + session.placement.spanSlots;
    const newEnd = startSlot + spanSlots;
    
    return (startSlot < existingEnd && newEnd > existingStart);
  });
}
```

### User Interface Flow

#### 1. Initial Load
- Load event configuration and existing sessions from localStorage
- Render board grid based on event time range
- Display unscheduled sessions in sidebar
- Set up real-time sync mechanism

#### 2. Session Proposal
- User fills out proposal form
- Client validates input and creates session object
- Session appears in "Proposed Sessions" sidebar
- Other users can see proposal and express interest

#### 3. Session Placement (Organizer Only)
- Drag session from sidebar to desired time/room slot
- System validates placement (no conflicts)
- Session appears on board grid
- Real-time update sent to all connected users

#### 4. Interest Tracking
- Users click "Show Interest" on session cards
- Interest count updates immediately
- Visual indicators help organizers prioritize popular sessions

### Persistence and Sync

#### Local Storage Strategy
```javascript
// Auto-save on every change
function saveSession(session) {
  const state = getAppState();
  state.sessions[session.id] = session;
  state.lastSync = Date.now();
  localStorage.setItem('unconference-board', JSON.stringify(state));
}

// Periodic sync check (for multi-tab scenarios)
setInterval(() => {
  const stored = localStorage.getItem('unconference-board');
  const state = JSON.parse(stored);
  if (state.lastSync > lastKnownSync) {
    refreshBoard();
    lastKnownSync = state.lastSync;
  }
}, 1000);
```

#### Data Export/Import
```javascript
function exportBoardData() {
  const state = getAppState();
  const blob = new Blob([JSON.stringify(state, null, 2)], 
    { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `unconference-board-${Date.now()}.json`;
  a.click();
}

function importBoardData(file) {
  const reader = new FileReader();
  reader.onload = (e) => {
    const data = JSON.parse(e.target.result);
    localStorage.setItem('unconference-board', JSON.stringify(data));
    location.reload();
  };
  reader.readAsText(file);
}
```

### MVP Limitations

- **Single Device/Browser**: No real multi-user sync (users must share screen or use same device)
- **No Authentication**: Simple name-based identification
- **Manual Backup**: Organizers must manually export/import data
- **Basic UI**: Minimal styling focused on functionality
- **Limited Validation**: Client-side only, no server verification

## Post-MVP Architecture

The post-MVP evolution introduces server-side capabilities, real-time synchronization, and advanced features while maintaining the core simplicity of the unconference board concept.

### Enhanced Technical Stack

- **Frontend**: React/Vue.js with WebSocket connections
- **Backend**: Node.js/Express with Socket.io for real-time sync
- **Database**: PostgreSQL with Redis for caching and pub/sub
- **Authentication**: OAuth integration (Google, GitHub, etc.)
- **Storage**: Cloud storage for session attachments and media

### Advanced Features

#### 1. Real-Time Collaboration
- **WebSocket Integration**: Instant updates across all connected clients
- **Soft Locks**: Visual indicators when someone is editing/moving sessions
- **Conflict Resolution**: Smart merging of simultaneous changes
- **Live Cursors**: See where other users are interacting on the board

#### 2. Freeform Canvas Mode
- **Drag & Resize**: Sessions can be placed and resized freely on a continuous timeline
- **Collision Detection**: Automatic suggestions for resolving scheduling conflicts
- **Smart Snapping**: Align sessions to common time boundaries
- **Multi-Track Views**: Support for parallel tracks and cross-track dependencies

#### 3. Enhanced Participation
- **Session Co-Creation**: Multiple facilitators can collaborate on session details
- **Topic Clustering**: AI-powered grouping of similar proposals
- **Matchmaking**: Connect attendees with similar interests
- **Capacity Management**: Real-time occupancy tracking with overflow handling

#### 4. Advanced Scheduling
```javascript
// Enhanced session model with dependencies
const advancedSession = {
  // ... basic properties
  dependencies: {
    requires: ['session-id-1'], // must happen after
    blocks: ['session-id-2'],   // cannot overlap
    suggests: ['session-id-3']  // ideally follows
  },
  resources: {
    equipment: ['projector', 'whiteboard'],
    accessibility: ['wheelchair-accessible', 'sign-language'],
    dietary: ['vegetarian-snacks']
  },
  collaboration: {
    cofacilitators: ['user-2', 'user-3'],
    notes: 'etherpad-url',
    materials: ['slide-deck-url']
  }
};
```

#### 5. Analytics and Optimization
- **Attendance Heatmaps**: Visual analysis of popular times/rooms
- **Engagement Metrics**: Track session success and participant satisfaction
- **Schedule Optimization**: AI suggestions for better session placement
- **Predictive Capacity**: Forecast attendance based on interest patterns

#### 6. Integration Ecosystem
- **Calendar Sync**: Bi-directional sync with Google Calendar, Outlook
- **Ticketing Integration**: Connect with Eventbrite, Meetup, custom registration
- **Webhook API**: External integrations for notifications and updates
- **Mobile App**: Native iOS/Android apps with offline capabilities

### Scalability Considerations

#### Database Schema Evolution
```sql
-- Enhanced tables for post-MVP features
CREATE TABLE events (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  starts_at TIMESTAMP NOT NULL,
  ends_at TIMESTAMP NOT NULL,
  timezone VARCHAR(50),
  slug VARCHAR(100) UNIQUE,
  settings JSONB DEFAULT '{}'::jsonb
);

CREATE TABLE session_dependencies (
  id UUID PRIMARY KEY,
  session_id UUID REFERENCES sessions(id),
  depends_on_id UUID REFERENCES sessions(id),
  dependency_type VARCHAR(20) -- 'requires', 'blocks', 'suggests'
);

CREATE TABLE session_resources (
  id UUID PRIMARY KEY,
  session_id UUID REFERENCES sessions(id),
  resource_type VARCHAR(50),
  resource_value VARCHAR(255)
);

CREATE TABLE real_time_locks (
  id UUID PRIMARY KEY,
  session_id UUID REFERENCES sessions(id),
  user_id UUID REFERENCES users(id),
  lock_type VARCHAR(20), -- 'editing', 'moving'
  expires_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);
```

#### Performance Optimizations
- **CDN Integration**: Static assets served from edge locations
- **Database Indexing**: Optimized queries for board rendering
- **Caching Strategy**: Redis for frequently accessed board states
- **Real-time Batching**: Group rapid updates to reduce WebSocket chatter

### Migration Path

#### Phase 1: Server Foundation
1. Set up basic Node.js backend with session CRUD operations
2. Implement WebSocket connections for real-time updates
3. Add user authentication and role management
4. Migrate existing localStorage data to server storage

#### Phase 2: Enhanced Features
1. Implement soft locks and conflict resolution
2. Add session dependencies and resource requirements
3. Build analytics dashboard for organizers
4. Integrate external calendar systems

#### Phase 3: Advanced Collaboration
1. Deploy freeform canvas mode with drag/resize
2. Implement AI-powered topic clustering and optimization
3. Add mobile applications with offline support
4. Build comprehensive API for third-party integrations

This specification provides a clear roadmap from a simple static site MVP to a sophisticated unconference management platform, ensuring each phase delivers incremental value while building toward the full vision.