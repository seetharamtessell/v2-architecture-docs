# Chat Context Management Architecture

## Table of Contents
1. [System Overview](#system-overview)
2. [Core Context Management Architecture](#core-context-management-architecture)
3. [Storage Strategies](#storage-strategies)
4. [Session Management](#session-management)
5. [Context Retrieval Patterns](#context-retrieval-patterns)
6. [Implementation Examples](#implementation-examples)

## System Overview

This document outlines the architecture specifically for **chat context management** in desktop applications. The focus is on storing, organizing, and retrieving conversation history without dependency on external embedding services or vector databases.

**Core Functionality:**
1. **Context Storage** - Persist conversation history locally
2. **Session Management** - Organize conversations by user and session
3. **Context Retrieval** - Retrieve relevant previous conversations
4. **Message Threading** - Link related conversations together

---

## Core Context Management Architecture

### System Overview Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    FRONTEND (React/Electron)                   │
│  ┌─────────────────┐    ┌─────────────────┐                   │
│  │   Chat UI       │    │ useChatMessages │                   │
│  │   Component     │◄──►│     Hook        │                   │
│  └─────────────────┘    └─────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
                                   │
                              IPC Communication
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ELECTRON MAIN PROCESS                       │
│  ┌─────────────────┐    ┌─────────────────┐                   │
│  │   IPC Handlers  │    │   Session Store │                   │
│  │   main.ts       │◄──►│   (In-Memory)   │                   │
│  └─────────────────┘    └─────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
                                   │
                               NPM Module Call
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│              CONTEXT MANAGEMENT MODULE                         │
│  ┌─────────────────┐    ┌─────────────────┐                   │
│  │ ChatContext     │    │ ContextRetrieval│                   │
│  │ Manager         │◄──►│ Service         │                   │
│  └─────────────────┘    └─────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
                                   │
                               File I/O Operations
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                    LOCAL STORAGE                               │
│  ./chat-data/                                                 │
│  ├── contexts/     # Processed conversation pairs             │
│  └── raw/          # Raw streaming data                       │
└─────────────────────────────────────────────────────────────────┘
```

### Core Components Architecture

#### 1. Context Storage Interface
```javascript
interface IContextManager {
  // Session Operations
  createSession(userId: string, sessionConfig: SessionConfig): Promise<Session>;
  getSession(sessionId: string): Promise<Session>;
  updateSession(sessionId: string, updates: Partial<Session>): Promise<Session>;

  // Message Operations
  addPrompt(sessionId: string, prompt: string, metadata?: any): Promise<void>;
  addResponse(sessionId: string, response: string, metadata?: any): Promise<void>;

  // Context Retrieval
  getRecentContext(userId: string, limit?: number): Promise<ContextPair[]>;
  getSessionContext(sessionId: string): Promise<ContextPair[]>;
  getUserContextHistory(userId: string, options?: ContextOptions): Promise<ContextPair[]>;

  // Maintenance
  cleanupSessions(criteria: CleanupCriteria): Promise<number>;
  getStorageStats(): Promise<StorageStats>;
}
```

#### 2. Context Processing Pipeline
```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   User       │    │   Session    │    │   Context    │    │   Storage    │
│   Message    ├───►│   Validation ├───►│   Processing ├───►│   Persistence│
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                                                ▲                     │
                    ┌──────────────┐    ┌──────────────┐              │
                    │   Context    │    │   File       │              │
                    │   Retrieval  │◄───┤   Organization│◄─────────────┘
                    └──────────────┘    └──────────────┘
```

---

## Storage Strategies

### File-Based Storage Pattern (Current Implementation)

#### Directory Structure
```
./chat-data/
├── contexts/                     # Processed conversation context
│   └── {userId}-{contextId}.json
├── raw/                          # Raw streaming responses
│   └── {userId}-{contextId}-raw.json
└── logs/                         # Optional system logs
    └── context-manager.log
```

#### File Storage Strategy
```javascript
class FileContextStorage {
  constructor(baseDir = './chat-data') {
    this.baseDir = baseDir;
    this.contextsDir = path.join(baseDir, 'contexts');
    this.rawDir = path.join(baseDir, 'raw');
  }

  async storeContext(contextId, conversationData) {
    const filename = `${contextId}.json`;
    const filePath = path.join(this.contextsDir, filename);

    const contextData = {
      context_id: contextId,
      timestamp: Date.now(),
      final_conversation_context: {
        i: contextId,
        m: conversationData.messages,
        t: conversationData.startTime,
        l: conversationData.endTime
      },
      processing_results: {
        clean_response: conversationData.cleanResponse,
        clean_response_length: conversationData.cleanResponse.length
      }
    };

    await fs.writeFile(filePath, JSON.stringify(contextData, null, 2));
  }

  async retrieveContext(userId, contextId) {
    const filename = `${userId}-${contextId}.json`;
    const filePath = path.join(this.contextsDir, filename);

    if (await fs.access(filePath).then(() => true).catch(() => false)) {
      const data = await fs.readFile(filePath, 'utf8');
      return JSON.parse(data);
    }
    return null;
  }
}
```

### Memory-Based Storage Pattern (Session Cache)
```javascript
class MemoryContextStorage {
  constructor() {
    this.sessions = new Map();
    this.userContexts = new Map();
  }

  storeSession(sessionId, sessionData) {
    this.sessions.set(sessionId, {
      ...sessionData,
      lastAccessed: Date.now()
    });
  }

  getUserSessions(userId) {
    return Array.from(this.sessions.values())
      .filter(session => session.userId === userId)
      .sort((a, b) => b.lastAccessed - a.lastAccessed);
  }

  // Automatic cleanup of old sessions
  cleanupOldSessions(maxAge = 24 * 60 * 60 * 1000) { // 24 hours
    const cutoff = Date.now() - maxAge;
    for (const [sessionId, session] of this.sessions) {
      if (session.lastAccessed < cutoff) {
        this.sessions.delete(sessionId);
      }
    }
  }
}
```

---

## Session Management

### Session Lifecycle Management

#### Session Creation and Tracking
```javascript
class SessionManager {
  constructor() {
    this.activeSessions = new Map();
    this.sessionHistory = new Map(); // userId -> sessionIds[]
  }

  createSession(userId, sessionConfig) {
    const session = {
      id: sessionConfig.sessionId,
      userId: userId,
      contextId: sessionConfig.contextId,
      chatTitle: sessionConfig.chatTitle,
      createdAt: Date.now(),
      lastActivity: Date.now(),
      messageCount: 0,
      status: 'active'
    };

    this.activeSessions.set(session.id, session);

    // Track user's session history
    if (!this.sessionHistory.has(userId)) {
      this.sessionHistory.set(userId, []);
    }
    this.sessionHistory.get(userId).unshift(session.id);

    return session;
  }

  updateSessionActivity(sessionId) {
    const session = this.activeSessions.get(sessionId);
    if (session) {
      session.lastActivity = Date.now();
      session.messageCount++;
    }
  }

  getUserSessions(userId, limit = 10) {
    const sessionIds = this.sessionHistory.get(userId) || [];
    return sessionIds.slice(0, limit).map(id => this.activeSessions.get(id));
  }
}
```

### Context ID Management
```javascript
class ContextIdManager {
  generateContextId(userId, baseId) {
    const timestamp = Date.now();
    const random = Math.random().toString(36).substring(2, 15);
    return `${userId}-${baseId}-${random}`;
  }

  extractUserFromContextId(contextId) {
    const parts = contextId.split('-');
    return parts.slice(0, -2).join('-'); // Remove random suffix and base
  }

  isValidContextId(contextId) {
    const parts = contextId.split('-');
    return parts.length >= 3 && parts[parts.length - 1].length === 13;
  }
}
```

---

## Context Retrieval Patterns

### File-Based Context Retrieval
```javascript
class FileContextRetrieval {
  constructor(dataDir = './chat-data/contexts') {
    this.dataDir = dataDir;
  }

  async getUserConversationContext(userId, currentContextId, maxPairs = 5) {
    try {
      // Scan directory for user's context files
      const files = await fs.readdir(this.dataDir);
      const userFiles = files.filter(file =>
        file.startsWith(userId) &&
        file.endsWith('.json') &&
        !file.includes(currentContextId)
      );

      // Sort by modification time (most recent first)
      const fileStats = await Promise.all(
        userFiles.map(async file => ({
          file,
          stats: await fs.stat(path.join(this.dataDir, file))
        }))
      );

      const sortedFiles = fileStats
        .sort((a, b) => b.stats.mtime - a.stats.mtime)
        .slice(0, maxPairs)
        .map(item => item.file);

      // Process files into context pairs
      const contextPairs = [];
      for (const file of sortedFiles) {
        const filePath = path.join(this.dataDir, file);
        const contextData = JSON.parse(await fs.readFile(filePath, 'utf8'));
        const pairs = this.extractConversationPairs(contextData);
        contextPairs.push(...pairs);
      }

      return contextPairs.slice(0, maxPairs);
    } catch (error) {
      console.error('Error retrieving context:', error);
      return [];
    }
  }

  extractConversationPairs(contextData) {
    const messages = contextData.final_conversation_context?.m || [];
    const pairs = [];

    for (let i = 0; i < messages.length - 1; i += 2) {
      const query = messages[i];
      const response = messages[i + 1];

      if (query.r === 0 && response?.r === 1) { // user=0, assistant=1
        pairs.push({
          query: {
            content: query.c || '',
            role: 'user',
            timestamp: contextData.final_conversation_context.t
          },
          response: {
            content: response.c || '',
            role: 'assistant',
            timestamp: contextData.final_conversation_context.l
          }
        });
      }
    }

    return pairs;
  }
}
```

### Context Filtering and Ranking
```javascript
class ContextFilter {
  filterRelevantContext(contextPairs, currentPrompt, options = {}) {
    const {
      maxLength = 1000,
      recencyBoost = true,
      keywordMatch = true
    } = options;

    let filtered = [...contextPairs];

    // Filter by content length
    if (maxLength > 0) {
      filtered = filtered.filter(pair =>
        (pair.query.content.length + pair.response.content.length) <= maxLength
      );
    }

    // Simple keyword matching for relevance
    if (keywordMatch && currentPrompt) {
      const promptWords = currentPrompt.toLowerCase().split(/\s+/);
      filtered = filtered.map(pair => ({
        ...pair,
        relevanceScore: this.calculateKeywordRelevance(pair, promptWords)
      })).filter(pair => pair.relevanceScore > 0);
    }

    // Apply recency boost
    if (recencyBoost) {
      const now = Date.now();
      filtered = filtered.map(pair => ({
        ...pair,
        relevanceScore: (pair.relevanceScore || 1) *
          Math.exp(-(now - pair.query.timestamp) / (7 * 24 * 60 * 60 * 1000)) // 7 days decay
      }));
    }

    // Sort by relevance
    return filtered.sort((a, b) => (b.relevanceScore || 0) - (a.relevanceScore || 0));
  }

  calculateKeywordRelevance(pair, promptWords) {
    const contextText = (pair.query.content + ' ' + pair.response.content).toLowerCase();
    let matchScore = 0;

    for (const word of promptWords) {
      if (word.length > 2 && contextText.includes(word)) {
        matchScore += word.length; // Longer words have higher weight
      }
    }

    return matchScore;
  }
}
```

---

## Implementation Examples

### Core Domain Objects
```javascript
// Message Interface
interface Message {
  id: string;
  sessionId: string;
  userId: string;
  role: 'user' | 'assistant';
  content: string;
  timestamp: number;
  metadata?: {
    agentType?: number;
    processingStage?: string;
    contentLength?: number;
  };
}

// Session Interface
interface Session {
  id: string;
  userId: string;
  contextId: string;
  chatTitle: string;
  createdAt: number;
  lastActivity: number;
  messageCount: number;
  status: 'active' | 'archived' | 'deleted';
}

// Context Pair Interface
interface ContextPair {
  query: {
    content: string;
    role: 'user';
    timestamp: number;
  };
  response: {
    content: string;
    role: 'assistant';
    timestamp: number;
  };
  relevanceScore?: number;
}

// Configuration Schema
interface ContextConfig {
  storage: {
    baseDir: string;
    maxFileSize: number;
    compressionEnabled: boolean;
  };
  retrieval: {
    maxContextPairs: number;
    keywordMatchingEnabled: boolean;
    recencyBoostEnabled: boolean;
  };
  session: {
    maxActiveSessions: number;
    sessionTimeoutMs: number;
    cleanupIntervalMs: number;
  };
  performance: {
    cacheEnabled: boolean;
    batchProcessing: boolean;
    maxConcurrentOperations: number;
  };
}
```

### Complete Implementation Example
```javascript
// main.ts - Electron IPC Integration
class ElectronContextIntegration {
  constructor() {
    this.contextManager = new ChatContextManager({
      baseDir: './chat-data',
      maxHistory: 25
    });

    this.setupIPCHandlers();
  }

  setupIPCHandlers() {
    ipcMain.handle('chat:start-session', async (event, sessionConfig) => {
      return await this.contextManager.startSession(sessionConfig);
    });

    ipcMain.handle('chat:add-prompt', async (event, sessionId, prompt) => {
      return await this.contextManager.addPrompt(sessionId, prompt);
    });

    ipcMain.handle('chat:add-response', async (event, sessionId, response) => {
      return await this.contextManager.addResponse(sessionId, response);
    });

    ipcMain.handle('chat:get-context', async (event, userId, contextId, maxPairs) => {
      return await this.contextManager.getContext(userId, contextId, maxPairs);
    });
  }
}

// React Hook Integration
function useChatMessages() {
  const [messages, setMessages] = useState([]);
  const [isLoading, setIsLoading] = useState(false);

  const sendMessage = async (message) => {
    try {
      setIsLoading(true);

      // Retrieve context for enhanced prompt
      const context = await window.electronAPI.chatGetContext(
        userId, currentContextId, 5
      );

      // Create enhanced prompt with context
      let enhancedPrompt = message;
      if (context.length > 0) {
        const contextString = context.map((pair, index) =>
          `Previous conversation ${index + 1}:\nUser: ${pair.query.content}\nAssistant: ${pair.response.content}\n`
        ).join('\n');

        enhancedPrompt = `Context from previous conversations:\n${contextString}\n\nCurrent request:\n${message}`;
      }

      // Send prompt to context storage
      await window.electronAPI.chatAddPrompt(enhancedPrompt);

      // Make API call with enhanced prompt
      const response = await makeAPICall(enhancedPrompt);

      // Save response to context storage
      await window.electronAPI.chatAddResponse(response);

      // Update UI
      setMessages(prev => [...prev, { role: 'user', content: message }, { role: 'assistant', content: response }]);

    } catch (error) {
      console.error('Failed to send message:', error);
    } finally {
      setIsLoading(false);
    }
  };

  return { messages, sendMessage, isLoading };
}
```

### Error Handling and Resilience
```javascript
class ResilientContextManager {
  constructor(config) {
    this.primaryStorage = new FileContextStorage(config.primaryPath);
    this.fallbackStorage = new FileContextStorage(config.fallbackPath);
    this.cache = new Map();
  }

  async storeContext(contextId, data) {
    try {
      // Try primary storage
      await this.primaryStorage.storeContext(contextId, data);

      // Update cache
      this.cache.set(contextId, data);

      // Async backup to fallback storage
      this.fallbackStorage.storeContext(contextId, data).catch(error =>
        console.warn('Fallback storage failed:', error)
      );

    } catch (primaryError) {
      console.warn('Primary storage failed, using fallback:', primaryError);

      try {
        await this.fallbackStorage.storeContext(contextId, data);
        this.cache.set(contextId, data);
      } catch (fallbackError) {
        console.error('Both storage systems failed:', fallbackError);
        throw new Error('Context storage completely failed');
      }
    }
  }

  async retrieveContext(contextId) {
    // Check cache first
    if (this.cache.has(contextId)) {
      return this.cache.get(contextId);
    }

    try {
      const data = await this.primaryStorage.retrieveContext(contextId);
      if (data) {
        this.cache.set(contextId, data);
        return data;
      }
    } catch (error) {
      console.warn('Primary retrieval failed, trying fallback:', error);
    }

    try {
      const data = await this.fallbackStorage.retrieveContext(contextId);
      if (data) {
        this.cache.set(contextId, data);
        return data;
      }
    } catch (error) {
      console.warn('Fallback retrieval failed:', error);
    }

    return null; // Graceful fallback: no context found
  }
}
```

### Performance Optimization
```javascript
class OptimizedContextManager {
  constructor(config) {
    this.batchOperations = [];
    this.batchTimer = null;
    this.contextCache = new LRUCache({ maxSize: 100 });
  }

  // Batched write operations
  addToBatch(operation) {
    this.batchOperations.push(operation);

    if (this.batchTimer) {
      clearTimeout(this.batchTimer);
    }

    this.batchTimer = setTimeout(() => {
      this.processBatch();
    }, 1000); // Batch for 1 second
  }

  async processBatch() {
    const operations = [...this.batchOperations];
    this.batchOperations = [];

    try {
      await Promise.all(operations.map(op => op.execute()));
      console.log(`Processed batch of ${operations.length} operations`);
    } catch (error) {
      console.error('Batch processing failed:', error);
      // Retry individual operations
      for (const op of operations) {
        try {
          await op.execute();
        } catch (retryError) {
          console.error('Individual operation failed:', retryError);
        }
      }
    }
  }

  // Memory-efficient context retrieval
  async getContextWithPagination(userId, offset = 0, limit = 10) {
    const cacheKey = `${userId}:${offset}:${limit}`;

    if (this.contextCache.has(cacheKey)) {
      return this.contextCache.get(cacheKey);
    }

    const context = await this.retrieveContextPaginated(userId, offset, limit);
    this.contextCache.set(cacheKey, context);

    return context;
  }
}
```

This focused architecture provides:

1. **Pure Context Management**: No external dependencies on embeddings or vector databases
2. **Local-First**: All data stays on user's machine for privacy
3. **Performance Optimized**: File-based storage with memory caching
4. **Error Resilient**: Graceful fallbacks and batch processing
5. **Electron Ready**: Direct integration with IPC and React hooks
6. **Scalable Design**: Supports large conversation histories efficiently
7. **Simple Implementation**: Easy to understand and maintain codebase


### System Diagram
```
┌─────────────────────────────────────────────────────────────────┐
│                    ELECTRON DESKTOP APP                         │
├─────────────────────────────────────────────────────────────────┤
│  Frontend Layer (React + TypeScript)                           │
│  ┌─────────────────┐    ┌─────────────────┐                   │
│  │   Chat UI       │    │ useChatMessages │                   │
│  │   Component     │◄──►│     Hook        │                   │
│  └─────────────────┘    └─────────────────┘                   │
│                                   │                            │
│                              IPC Communication                 │
│                                   ▼                            │
├─────────────────────────────────────────────────────────────────┤
│  Electron Main Process                                         │
│  ┌─────────────────┐    ┌─────────────────┐                   │
│  │   IPC Handlers  │    │   Session Mgmt  │                   │
│  │   main.ts       │◄──►│                 │                   │
│  └─────────────────┘    └─────────────────┘                   │
│                                   │                            │
│                               NPM Module                       │
│                                   ▼                            │
├─────────────────────────────────────────────────────────────────┤
│              RAG MODULE (@escher-dbai/rag-module)              │
│  ┌─────────────────┐    ┌─────────────────┐                   │
│  │ ChatContext     │    │ ContextRetrieval│                   │
│  │ Manager         │◄──►│ ServiceMinimal  │                   │
│  │                 │    │ (File-based)    │                   │
│  └─────────────────┘    └─────────────────┘                   │
│                                   │                            │
│                               File I/O                         │
│                                   ▼                            │
├─────────────────────────────────────────────────────────────────┤
│                    LOCAL FILE SYSTEM                          │
│  ./chat-data/                                                 │
│  ├── contexts/                                                │
│  │   └── user-contextId.json                                 │
│  └── raw/                                                     │
│      └── user-contextId-raw.json                              │
└─────────────────────────────────────────────────────────────────┘
```

### Core Components

#### 1. ChatContextManager
**Purpose**: Session management and file-based storage orchestration

```javascript
class ChatContextManager {
  constructor(config) {
    this.config = {
      baseDir: './chat-data',
      rawDir: 'raw',
      contextDir: 'contexts',
      maxHistory: 25
    };
    this.activeSessions = new Map(); // In-memory session tracking
  }

  // Session lifecycle
  startSession(sessionConfig) { /* Create new session */ }
  addPrompt(sessionId, prompt) { /* Store user input */ }
  addResponse(sessionId, response) { /* Store AI response */ }

  // File operations
  _writeRawData(contextId, data) { /* Write to raw/ */ }
  _writeProcessedContext(contextId, context) { /* Write to contexts/ */ }
}
```

#### 2. ContextRetrievalServiceMinimal
**Purpose**: File-based context retrieval without dependencies

```javascript
class ContextRetrievalServiceMinimal {
  async getUserConversationContext(userId, currentContextId) {
    // 1. Scan context directory for user files
    // 2. Read and parse JSON files
    // 3. Structure conversation pairs
    // 4. Return formatted context

    return [
      {
        query: { content: "user message", role: "user", timestamp: 123456 },
        response: { content: "AI response", role: "assistant", timestamp: 123457 }
      }
    ];
  }
}
```

### Data Structures

#### Session Data (In-Memory)
```javascript
activeSessions = Map {
  "contextId" => {
    userId: "aparna Pitchikala-1f4a3a8d-...",
    sessionId: "contextId",
    contextId: "extended-context-id",
    chatTitle: "Chat started 15:09:02",
    timestamp: 1728391162679,
    messages: []
  }
}
```

#### Context File Format (Persistent)
```json
{
  "agent_type": 7,
  "chat_title": "Chat started 15:09:02",
  "context_id": "user-extended-context-id",
  "final_conversation_context": {
    "i": "user-context-id",
    "m": [
      { "r": 0, "c": "user message content" },
      { "r": 1, "c": "assistant response content" }
    ],
    "t": 1759916362,
    "l": 1759916362
  },
  "processing_results": {
    "clean_response": "processed content",
    "clean_response_length": 511
  }
}
```

### File System Organization
```
./chat-data/
├── contexts/                     # Processed context files
│   ├── user-contextId1.json     # Final conversation context
│   ├── user-contextId2.json     # Ready for retrieval
│   └── user-contextId3.json
├── raw/                          # Raw conversation data
│   ├── user-contextId1-raw.json # Original prompt/response
│   ├── user-contextId2-raw.json # Before filtering
│   └── user-contextId3-raw.json
└── logs/                         # System logs (optional)
```

### Advantages
- ✅ **Zero Dependencies**: No external services required
- ✅ **Privacy**: All data remains local
- ✅ **Fast Performance**: File I/O with in-memory caching
- ✅ **ES Module Safe**: No compatibility issues
- ✅ **Offline Capable**: Works without internet

### Limitations
- ❌ **No Semantic Search**: Text-based matching only
- ❌ **Limited Scalability**: Performance degrades with large datasets
- ❌ **No Vector Similarity**: Cannot find conceptually similar conversations
- ❌ **Basic Filtering**: Limited to exact text matching

---

## Qdrant Vector Database Architecture

### System Diagram
```
┌─────────────────────────────────────────────────────────────────┐
│                    ELECTRON DESKTOP APP                         │
├─────────────────────────────────────────────────────────────────┤
│  Frontend Layer (React + TypeScript)                           │
│  ┌─────────────────┐    ┌─────────────────┐                   │
│  │   Chat UI       │    │ useChatMessages │                   │
│  │   Component     │◄──►│     Hook        │                   │
│  └─────────────────┘    └─────────────────┘                   │
│                                   │                            │
│                              IPC Communication                 │
│                                   ▼                            │
├─────────────────────────────────────────────────────────────────┤
│  Electron Main Process                                         │
│  ┌─────────────────┐    ┌─────────────────┐                   │
│  │   IPC Handlers  │    │   Session Mgmt  │                   │
│  │   main.ts       │◄──►│                 │                   │
│  └─────────────────┘    └─────────────────┘                   │
│                                   │                            │
│                               NPM Module                       │
│                                   ▼                            │
├─────────────────────────────────────────────────────────────────┤
│              RAG MODULE (@escher-dbai/rag-module)              │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐│
│  │ ChatContext     │    │ ContextRetrieval│    │ Embedding   ││
│  │ Manager         │◄──►│ Service         │◄──►│ Service     ││
│  │                 │    │ (Qdrant)        │    │ (BGE-M3)    ││
│  └─────────────────┘    └─────────────────┘    └─────────────┘│
│                                   │                     │      │
│                              Vector Ops              Embed     │
│                                   ▼                     ▼      │
├─────────────────────────────────────────────────────────────────┤
│                    QDRANT VECTOR DATABASE                      │
│  ┌─────────────────┐    ┌─────────────────┐                   │
│  │   Collections   │    │   HNSW Index    │                   │
│  │   - chat-ctx    │    │   - 1024-dim    │                   │
│  │   - users       │    │   - Cosine Sim  │                   │
│  └─────────────────┘    └─────────────────┘                   │
│                                                                │
│  ┌─────────────────┐    ┌─────────────────┐                   │
│  │   Metadata      │    │   Embeddings    │                   │
│  │   - user_id     │    │   - BGE-M3      │                   │
│  │   - timestamp   │    │   - Semantic    │                   │
│  │   - context_id  │    │   - Similarity  │                   │
│  └─────────────────┘    └─────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
```

### Enhanced Components

#### 1. ChatContextManager (Qdrant-Enhanced)
```javascript
class ChatContextManager {
  constructor(config) {
    this.config = config;
    this.qdrantClient = new QdrantVectorStore(config.qdrant);
    this.embeddingService = new EmbeddingService(config.embedding);
    this.activeSessions = new Map();
  }

  async addPrompt(sessionId, prompt) {
    // 1. Store in session
    const session = this.activeSessions.get(sessionId);
    session.messages.push({ role: 'user', content: prompt });

    // 2. Generate embedding
    const embedding = await this.embeddingService.embed(prompt);

    // 3. Store in Qdrant
    await this.qdrantClient.upsert({
      id: `prompt_${sessionId}_${Date.now()}`,
      vector: embedding,
      payload: {
        user_id: session.userId,
        content: prompt,
        role: 'user',
        timestamp: Date.now(),
        context_id: sessionId
      }
    });
  }

  async addResponse(sessionId, response) {
    // Similar process for AI responses
    const embedding = await this.embeddingService.embed(response);
    await this.qdrantClient.upsert({
      id: `response_${sessionId}_${Date.now()}`,
      vector: embedding,
      payload: {
        user_id: session.userId,
        content: response,
        role: 'assistant',
        timestamp: Date.now(),
        context_id: sessionId
      }
    });
  }
}
```

#### 2. ContextRetrievalService (Qdrant-Based)
```javascript
class ContextRetrievalService {
  constructor(qdrantClient, collectionName) {
    this.qdrant = qdrantClient;
    this.collection = collectionName;
    this.embeddingService = new EmbeddingService();
  }

  async getUserConversationContext(userId, currentPrompt, options = {}) {
    // 1. Generate query embedding
    const queryEmbedding = await this.embeddingService.embed(currentPrompt);

    // 2. Semantic search in Qdrant
    const searchResults = await this.qdrant.search({
      collection_name: this.collection,
      vector: queryEmbedding,
      filter: {
        must: [
          { key: "user_id", match: { value: userId } },
          { key: "role", match: { value: "user" } } // Find similar user questions
        ]
      },
      limit: options.limit || 10,
      with_payload: true,
      with_vector: false
    });

    // 3. Group by context_id and retrieve conversation pairs
    const contextGroups = new Map();
    for (const result of searchResults) {
      const contextId = result.payload.context_id;
      if (!contextGroups.has(contextId)) {
        contextGroups.set(contextId, []);
      }
      contextGroups.get(contextId).push(result);
    }

    // 4. Retrieve complete conversations
    const conversationPairs = [];
    for (const [contextId, messages] of contextGroups) {
      const fullConversation = await this._getCompleteConversation(contextId);
      conversationPairs.push(...fullConversation);
    }

    return conversationPairs;
  }

  async _getCompleteConversation(contextId) {
    // Retrieve all messages for a conversation
    const allMessages = await this.qdrant.scroll({
      collection_name: this.collection,
      filter: {
        must: [
          { key: "context_id", match: { value: contextId } }
        ]
      },
      with_payload: true,
      with_vector: false,
      limit: 100
    });

    // Sort by timestamp and pair user/assistant messages
    const sortedMessages = allMessages.points.sort((a, b) =>
      a.payload.timestamp - b.payload.timestamp
    );

    const pairs = [];
    for (let i = 0; i < sortedMessages.length - 1; i += 2) {
      const userMsg = sortedMessages[i];
      const assistantMsg = sortedMessages[i + 1];

      if (userMsg.payload.role === 'user' && assistantMsg.payload.role === 'assistant') {
        pairs.push({
          query: {
            content: userMsg.payload.content,
            role: 'user',
            timestamp: userMsg.payload.timestamp
          },
          response: {
            content: assistantMsg.payload.content,
            role: 'assistant',
            timestamp: assistantMsg.payload.timestamp
          }
        });
      }
    }

    return pairs;
  }
}
```

#### 3. Embedding Service
```javascript
class EmbeddingService {
  constructor(config) {
    this.config = {
      model: 'BAAI/bge-m3',
      dimensions: 1024,
      endpoint: 'http://localhost:8080',
      ...config
    };
  }

  async embed(text) {
    const response = await fetch(`${this.config.endpoint}/encode`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ text })
    });

    const result = await response.json();
    return result.embedding; // 1024-dimensional vector
  }

  async embedBatch(texts) {
    const response = await fetch(`${this.config.endpoint}/encode_batch`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ texts })
    });

    const result = await response.json();
    return result.embeddings;
  }
}
```

### Qdrant Collection Schema
```javascript
// Collection configuration for chat contexts
{
  name: "chat-contexts",
  vectors: {
    size: 1024,        // BGE-M3 embedding dimensions
    distance: "Cosine" // Cosine similarity for semantic search
  },
  optimizers_config: {
    default_segment_number: 2,
    memmap_threshold: 20000
  },
  hnsw_config: {
    m: 48,              // Max connections per node
    ef_construct: 256,   // Construction time accuracy
    ef: 64,             // Search time accuracy
    max_indexing_threads: 0
  }
}
```

### Vector Document Structure
```javascript
{
  id: "prompt_contextId_timestamp", // Unique identifier
  vector: [0.1, -0.2, 0.8, ...],   // 1024-dim embedding
  payload: {
    user_id: "aparna Pitchikala-1f4a3a8d-...",
    content: "What are my EC2 instances?",
    role: "user", // or "assistant"
    timestamp: 1728391162679,
    context_id: "contextId",
    session_id: "sessionId",
    metadata: {
      chat_title: "Chat started 15:09:02",
      agent_type: 7,
      processing_stage: "final"
    }
  }
}
```

### Advanced Search Capabilities

#### 1. Semantic Similarity Search
```javascript
// Find conversations about similar topics
const similarContexts = await contextService.searchSimilarTopics(
  "EC2 performance issues",
  {
    user_id: userId,
    limit: 5,
    similarity_threshold: 0.75
  }
);
```

#### 2. Multi-Modal Context Retrieval
```javascript
// Combine semantic search with metadata filtering
const relevantContext = await contextService.getRelevantContext(
  currentPrompt,
  {
    user_id: userId,
    time_range: { last_days: 7 },
    min_similarity: 0.6,
    include_system_messages: false
  }
);
```

#### 3. Conversation Threading
```javascript
// Find related conversation threads
const threadedContext = await contextService.getConversationThreads(
  userId,
  {
    semantic_similarity: 0.7,
    temporal_clustering: true,
    max_threads: 3
  }
);
```

### Advantages
- ✅ **Semantic Search**: Find conceptually similar conversations
- ✅ **Scalable**: Handles millions of conversation vectors
- ✅ **Advanced Filtering**: Multi-dimensional search with metadata
- ✅ **Real-time**: Fast vector similarity search with HNSW
- ✅ **Context Awareness**: Intelligent context selection based on relevance

### Requirements
- ❌ **External Dependencies**: Requires Qdrant and embedding service
- ❌ **Resource Intensive**: Higher memory and compute requirements
- ❌ **Network Dependent**: Needs network access for services
- ❌ **Complex Setup**: Requires infrastructure management

---

## Hybrid Architecture

### System Design
```
┌─────────────────────────────────────────────────────────────────┐
│                    HYBRID CONTEXT SYSTEM                       │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐    ┌─────────────────┐                   │
│  │   Fast Cache    │    │  Context Router │                   │
│  │   (File-based)  │◄──►│                 │                   │
│  └─────────────────┘    └─────────────────┘                   │
│                                   │                            │
│                              Route Decision                    │
│                                   ▼                            │
│  ┌─────────────────┐    ┌─────────────────┐                   │
│  │ Local Storage   │    │ Qdrant Vector   │                   │
│  │ (Recent/Fast)   │    │ (Semantic/Deep) │                   │
│  └─────────────────┘    └─────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
```

### Hybrid ContextManager
```javascript
class HybridContextManager {
  constructor(config) {
    this.localManager = new ChatContextManager(config.local);
    this.vectorManager = new QdrantContextManager(config.qdrant);
    this.router = new ContextRouter(config.routing);
  }

  async getContext(userId, prompt, options = {}) {
    const strategy = this.router.determineStrategy(prompt, options);

    switch (strategy) {
      case 'local_recent':
        // Fast: Recent conversations from local files
        return await this.localManager.getRecentContext(userId, 5);

      case 'semantic_deep':
        // Intelligent: Semantic search in Qdrant
        return await this.vectorManager.getSemanticContext(userId, prompt);

      case 'hybrid':
        // Combined: Local recent + semantic similar
        const recent = await this.localManager.getRecentContext(userId, 3);
        const semantic = await this.vectorManager.getSemanticContext(userId, prompt, 3);
        return this.mergeContexts(recent, semantic);

      default:
        return await this.localManager.getRecentContext(userId, 5);
    }
  }

  async storeContext(sessionId, message) {
    // Always store in both systems
    await Promise.all([
      this.localManager.addMessage(sessionId, message),
      this.vectorManager.addMessage(sessionId, message)
    ]);
  }
}
```

### Context Routing Strategy
```javascript
class ContextRouter {
  determineStrategy(prompt, options) {
    // Rule-based routing logic

    // Fast path for simple queries
    if (this.isSimpleQuery(prompt)) {
      return 'local_recent';
    }

    // Semantic search for complex queries
    if (this.isComplexQuery(prompt)) {
      return 'semantic_deep';
    }

    // Hybrid for balanced approach
    if (options.comprehensive) {
      return 'hybrid';
    }

    return 'local_recent'; // Default fallback
  }

  isSimpleQuery(prompt) {
    const simplePatterns = [
      /^(hi|hello|help)/i,
      /^(what|how|why)\s+\w{1,10}$/i,
      /^(show|list|get)\s+\w+$/i
    ];

    return simplePatterns.some(pattern => pattern.test(prompt));
  }

  isComplexQuery(prompt) {
    const complexIndicators = [
      prompt.length > 100,
      prompt.includes('compared to'),
      prompt.includes('similar to'),
      prompt.includes('like before'),
      /relationship|connection|correlation/i.test(prompt)
    ];

    return complexIndicators.some(indicator =>
      typeof indicator === 'boolean' ? indicator : indicator
    );
  }
}
```

---

## Implementation Comparison

| Feature | Local Files | Qdrant Vector | Hybrid |
|---------|-------------|---------------|---------|
| **Performance** | ⚡ Fast (ms) | 🔄 Medium (100-500ms) | 🎯 Adaptive |
| **Semantic Search** | ❌ No | ✅ Advanced | ✅ Selective |
| **Setup Complexity** | ✅ Simple | ❌ Complex | 🔄 Medium |
| **Dependencies** | ✅ Zero | ❌ Multiple | 🔄 Optional |
| **Scalability** | ❌ Limited | ✅ Unlimited | 🎯 Optimized |
| **Privacy** | ✅ Full Local | ❌ Network Required | 🔄 Configurable |
| **Context Quality** | 🔄 Basic | ✅ Intelligent | ✅ Best |
| **Resource Usage** | ✅ Minimal | ❌ High | 🔄 Moderate |
| **Offline Support** | ✅ Full | ❌ None | 🔄 Degraded |
| **Development Speed** | ✅ Fast | ❌ Slow | 🔄 Medium |

---

## Migration Strategy

### Phase 1: Current State (Local Files Only)
```
✅ File-based storage working
✅ Basic context retrieval
✅ ES module compatibility
✅ Zero dependencies
```

### Phase 2: Qdrant Integration (Optional Enhancement)
```
🔄 Add Qdrant client to NPM module
🔄 Implement vector storage alongside file storage
🔄 Create embedding service integration
🔄 Add configuration for Qdrant connection
```

### Phase 3: Hybrid Implementation (Best of Both)
```
🔄 Context routing logic
🔄 Performance optimization
🔄 Fallback mechanisms
🔄 Configuration-driven behavior
```

### Configuration Example
```javascript
// config.json
{
  "storage": {
    "mode": "hybrid", // "local", "qdrant", "hybrid"
    "local": {
      "baseDir": "./chat-data",
      "maxHistory": 25
    },
    "qdrant": {
      "url": "http://localhost:6333",
      "collection": "chat-contexts",
      "apiKey": null
    },
    "embedding": {
      "service": "bge-m3",
      "endpoint": "http://localhost:8080",
      "dimensions": 1024
    },
    "routing": {
      "default_strategy": "local_recent",
      "semantic_threshold": 0.75,
      "hybrid_weight": 0.6
    }
  }
}
```

### Usage Examples

#### Local-Only (Current)
```javascript
const { ChatContextManager } = require('@escher-dbai/rag-module');
const manager = new ChatContextManager({ baseDir: './chat-data' });

const context = await manager.getContext(userId, 'recent');
```

#### Qdrant-Enhanced
```javascript
const { ChatContextManager } = require('@escher-dbai/rag-module');
const manager = new ChatContextManager({
  mode: 'qdrant',
  qdrant: { url: 'http://localhost:6333' }
});

const context = await manager.getSemanticContext(userId, prompt);
```

#### Hybrid (Best of Both)
```javascript
const { HybridContextManager } = require('@escher-dbai/rag-module');
const manager = new HybridContextManager(hybridConfig);

// Automatically routes to optimal strategy
const context = await manager.getContext(userId, prompt, { comprehensive: true });
```

---

## Conclusion

This architecture provides three distinct approaches to chat context storage and retrieval:

1. **Local Files** - Perfect for current needs with zero dependencies and full privacy
2. **Qdrant Vector** - Advanced semantic search capabilities for complex context retrieval
3. **Hybrid** - Intelligent combination providing optimal performance and capabilities

The modular design allows for gradual migration and configuration-driven behavior, ensuring the system can evolve with changing requirements while maintaining backward compatibility and ES module safety.