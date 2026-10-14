# WavePredictz Technical Architecture

## System Overview

WavePredictz is a full-stack TypeScript application implementing a safe football analytics engine with episodic-to-semantic memory promotion, pattern recognition, and comprehensive tactical analysis.

## Architecture Layers

### 1. Data Layer (Supabase PostgreSQL)

#### Schema Design

**users**
- Purpose: User authentication and profile management
- RLS: Users can only access their own data
- Key fields: id (uuid), email, is_premium, created_at

**analyses**
- Purpose: Store all tactical analyses
- RLS: Users can only view/create their own analyses
- Key fields: id, user_id, input (jsonb), output (jsonb)
- Design: JSONB columns allow flexible schema evolution

**memory_episodes**
- Purpose: Store individual tactical observations
- RLS: Authenticated users can read/insert
- Key fields: team_id, type, content (jsonb), confidence, embedding (jsonb)
- Design: Confidence scoring enables automatic promotion

**memory_semantic**
- Purpose: Store promoted patterns
- RLS: Authenticated users can read/insert
- Key fields: pattern_key, pattern_doc, usefulness_score, safety_review
- Design: Usefulness scoring enables automatic demotion

**pattern_suggestions**
- Purpose: Track promotion candidates
- RLS: Authenticated users can read/insert
- Key fields: team_id, suggestion (jsonb), status

**event_log**
- Purpose: Audit trail for system operations
- RLS: Authenticated users can read/insert
- Key fields: user_id, event_type, details (jsonb)

### 2. Service Layer (TypeScript)

#### analysisService.ts

**Core Functions:**

- `detectsGamblingRequest()`: Safety filter using keyword matching
- `sanitizeInput()`: Removes prohibited content from user input
- `generateAnalysis()`: Main analysis engine with statistical calculations
- `saveAnalysis()`: Persists analysis and triggers pattern extraction
- `getHistoricalFacility()`: Aggregates historical data
- `analyzePatterns()`: Wrapper for memory engine pattern evaluation

**Algorithm Design:**

The analysis generation uses statistical methods:

```
Stability Index = 1 - (goal_variance / 4)
Volatility Index = min(1, goal_variance / 2)
Form Trend = sum(win=1, draw=0.5, loss=0) / num_matches
Dual Offense Rate = matches_with_both_scoring / total_matches
```

#### memoryEngine.ts

**Core Functions:**

- `storeEpisodicMemory()`: Saves individual observations
- `getEpisodicMemory()`: Retrieves team-specific episodes
- `getSemanticMemory()`: Queries promoted patterns
- `evaluatePromotions()`: Promotion logic with configurable thresholds
- `evaluateDemotions()`: Demotion logic for low-value patterns
- `extractPatternsFromAnalysis()`: Parses analysis for storable patterns

**Promotion Algorithm:**

```
FOR each pattern_key in episodic_memory:
  IF occurrences >= N_PROMOTE (5)
    AND avg_confidence >= PROMOTE_THRESHOLD (0.6)
  THEN
    CREATE semantic_memory_entry
    LOG promotion_event
```

**Demotion Algorithm:**

```
FOR each pattern in semantic_memory:
  IF usefulness_score < DEMOTE_THRESHOLD (0.3)
  THEN
    MARK for_review = false
    LOG demotion_event
```

### 3. Presentation Layer (React + TypeScript)

#### Component Hierarchy

```
App
├── Navigation (responsive menu)
├── AnalysisForm
│   ├── Team input
│   ├── League selector
│   ├── Match list builder
│   └── Submit handler
├── ScoringProfile
│   ├── Scoring tendency cards
│   ├── Consistency matrix with progress bars
│   ├── Statistical overview grid
│   ├── Historical trends list
│   └── Interactive steps
├── HistoricalFacility
│   ├── Summary statistics
│   ├── Analysis timeline (selectable)
│   ├── Semantic patterns grid
│   └── ScoringProfile display
└── PatternAnalysis
    ├── Team input
    ├── Promotion candidates list
    ├── Demoted patterns list
    └── Analysis summary
```

#### State Management

Using React hooks for local state:

- `useState<View>`: Current navigation tab
- `useState<WaveAnalysis>`: Active analysis result
- `useState<boolean>`: Loading states
- `useState<string>`: Error messages

No global state management needed due to database persistence.

#### Data Flow

```
User Input → AnalysisForm
  ↓
generateAnalysis() → Statistical calculations
  ↓
saveAnalysis() → Supabase + extractPatterns()
  ↓
Display in ScoringProfile
  ↓
Episodic memory stored → evaluatePromotions()
  ↓
Semantic memory (if threshold met)
```

## Type System

### Core Types (wavepredictz.ts)

**WaveAnalysis** - Complete analysis output structure
**AnalysisInput** - User-provided team and match data
**MatchData** - Individual match record
**ConsistencyMatrix** - Performance stability metrics
**MemoryEpisode** - Single episodic observation
**SemanticMemory** - Promoted pattern record
**PatternAnalysisResult** - Pattern evaluation output

TypeScript ensures type safety across all layers.

## Safety Architecture

### Multi-Layer Protection

1. **Input Layer**: Keyword filtering before processing
2. **Service Layer**: Content sanitization in analysisService
3. **Output Layer**: Educational framing in all responses
4. **Database Layer**: RLS policies prevent unauthorized access

### Banned Content Detection

```typescript
const BANNED_KEYWORDS = [
  'bet', 'wager', 'odds', 'stake',
  'fixed match', 'sure bet', 'gambling',
  'bookmaker'
];

function detectsGamblingRequest(text: string): boolean {
  return BANNED_KEYWORDS.some(
    keyword => text.toLowerCase().includes(keyword)
  );
}
```

## Performance Considerations

### Database Optimization

- **Indexes**: Created on frequently queried columns (team_id, user_id, event_type)
- **JSONB**: Allows flexible schema with GIN indexing capability
- **RLS**: Implemented but may need query optimization for large datasets

### Query Patterns

- Use `maybeSingle()` instead of `single()` to avoid error handling overhead
- Limit result sets (default: 50-200 rows)
- Order by `created_at DESC` for recency

### Frontend Optimization

- Conditional rendering to avoid unnecessary component mounts
- Loading states to prevent UI blocking
- Error boundaries (can be added) for graceful degradation

## Scalability Considerations

### Current Limitations

- In-memory pattern clustering (memory_episodes)
- Naive pattern key generation (JSON stringification)
- No pagination in historical facility
- No caching layer

### Future Optimizations

1. **Pattern Clustering**: Implement proper semantic similarity
2. **Embeddings**: Use vector database (pgvector) for pattern matching
3. **Caching**: Redis layer for frequently accessed analyses
4. **Batch Processing**: Worker queue for pattern promotions
5. **API Rate Limiting**: Protect against abuse

## Deployment Architecture

### Development
```
npm run dev → Vite dev server (port 5173)
Supabase local → Development database
```

### Production
```
npm run build → Static assets
Deploy to: Vercel/Netlify/CloudFlare Pages
Supabase hosted → Production database
Row Level Security enforces access control
```

## Configuration

### Environment Variables

Required:
- `VITE_SUPABASE_URL`: Supabase project URL
- `VITE_SUPABASE_ANON_KEY`: Supabase anonymous key

Optional (for AI integration):
- `OPENAI_API_KEY`: If integrating real AI
- `BOLT_API_KEY`: If using Bolt.new AI

### Tunable Parameters

In `memoryEngine.ts`:
```typescript
const N_PROMOTE = 5;           // Minimum occurrences for promotion
const PROMOTE_THRESHOLD = 0.6; // Minimum confidence for promotion
const DEMOTE_THRESHOLD = 0.3;  // Maximum score before demotion
```

## Testing Strategy

### Unit Tests (not yet implemented)

- analysisService: Test calculation logic
- memoryEngine: Test promotion/demotion rules
- Safety filters: Test keyword detection

### Integration Tests (not yet implemented)

- Database operations with test data
- Full analysis flow end-to-end
- Pattern promotion workflow

### Manual Testing

Current approach:
1. Create analysis with sample team data
2. Verify calculations match expected values
3. Check pattern storage in database
4. Run pattern analysis on stored data
5. Verify promotion/demotion logic

## Security Considerations

### Input Validation

- Team names: No length limits (should add)
- Match data: Numeric validation for goals
- Dates: HTML5 date input validation
- Notes: Content filtering applied

### Authentication

- Uses Supabase Auth (not currently implemented in UI)
- RLS policies depend on `auth.uid()`
- All tables have RLS enabled

### Data Privacy

- User data isolated by RLS
- No PII in episodic/semantic memory
- Event logs track user_id but content filtered

## Monitoring and Logging

### Event Log Usage

All system operations logged:
- Analysis creation
- Episodic memory storage
- Pattern promotions
- Pattern demotions

Query examples:
```sql
-- Most active users
SELECT user_id, COUNT(*)
FROM event_log
GROUP BY user_id
ORDER BY count DESC;

-- Promotion rate
SELECT
  COUNT(*) FILTER (WHERE event_type = 'promotion') as promotions,
  COUNT(*) FILTER (WHERE event_type = 'episodic_memory_stored') as episodes
FROM event_log;
```

## Extension Points

### Adding New Analysis Types

1. Define new type in `wavepredictz.ts`
2. Implement calculation in `analysisService.ts`
3. Create visualization component
4. Add to navigation in `App.tsx`

### Adding AI Integration

Replace mock analysis in `analysisService.ts`:

```typescript
export async function generateAnalysis(input: AnalysisInput) {
  // Call AI service here
  const aiResponse = await callAI(input);

  // Validate and return
  return aiResponse;
}
```

### Adding Real-time Data

1. Create data ingestion pipeline
2. Store in new `matches` table
3. Link to existing analysis flow
4. Update queries to use live data

## Code Organization

```
src/
├── components/          # React components
│   ├── AnalysisForm.tsx
│   ├── ScoringProfile.tsx
│   ├── HistoricalFacility.tsx
│   └── PatternAnalysis.tsx
├── lib/                # Core services
│   ├── supabase.ts
│   ├── analysisService.ts
│   └── memoryEngine.ts
├── types/              # TypeScript definitions
│   └── wavepredictz.ts
├── App.tsx             # Main application
├── main.tsx            # React entry point
└── index.css           # Tailwind styles
```

## Dependencies

### Production
- `react` + `react-dom`: UI framework
- `@supabase/supabase-js`: Database client
- `lucide-react`: Icon library

### Development
- `typescript`: Type checking
- `vite`: Build tool
- `tailwindcss`: Styling
- `eslint`: Code quality

## Maintenance Guide

### Regular Tasks

1. Review event logs for anomalies
2. Check pattern promotion quality
3. Monitor database growth
4. Update safety keyword list
5. Review and approve semantic patterns

### Database Maintenance

```sql
-- Clean up old event logs (keep 30 days)
DELETE FROM event_log
WHERE created_at < NOW() - INTERVAL '30 days';

-- Archive old analyses (if needed)
-- Implement archival strategy based on usage

-- Reindex for performance
REINDEX TABLE memory_episodes;
REINDEX TABLE memory_semantic;
```

## Troubleshooting

### Common Issues

**Analysis not generating**
- Check input validation
- Verify Supabase connection
- Review browser console for errors

**Pattern promotion not working**
- Verify N_PROMOTE and PROMOTE_THRESHOLD
- Check episodic memory count
- Review confidence scores

**Performance slow**
- Check database indexes
- Review query plans
- Consider adding pagination

## Contributing Guidelines

1. Maintain type safety (no `any` types)
2. Follow existing code style
3. Add error handling for all async operations
4. Update types when changing data structures
5. Document complex algorithms
6. Test safety filters thoroughly

## License and Compliance

- Educational use only
- No gambling functionality
- Safe for all ages
- Data privacy compliant
- Open source friendly
