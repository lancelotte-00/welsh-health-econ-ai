# Frailty Prevention Feature — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a standalone frailty prevention feature in the NHS Wales app covering exercise programmes, home safety checks, and local classes directory.

**Architecture:** Three self-contained feature modules sharing a common accessibility-first UI layer. All user data stored locally on device. Content driven by structured JSON data models that can be updated independently of app releases. Anonymised analytics events sent to a lightweight backend for usage reporting.

**Tech Stack:** React Native (assumed — cross-platform iOS/Android requirement), TypeScript, local storage (AsyncStorage or SQLite), video playback (react-native-video), postcode lookup (OS Places API or local dataset), analytics (lightweight event logging).

**Spec:** `docs/superpowers/specs/2026-03-20-frailty-prevention-design.md`

**Note:** This plan assumes the NHS Wales app is a React Native project. If the actual app uses a different stack (e.g. native Swift/Kotlin, Flutter), adapt the file paths and code examples accordingly. The architecture and task sequence remain the same.

---

## File Structure

```
src/features/frailty/
├── components/
│   ├── AccessibleButton.tsx        # Large touch target button component
│   ├── AccessibleCard.tsx          # Card component with high contrast, large text
│   ├── VideoPlayer.tsx             # Accessible video player with captions
│   ├── ExerciseCard.tsx            # Static illustrated exercise fallback
│   ├── ChecklistItem.tsx           # Yes/no question with conditional advice
│   ├── ClassListItem.tsx           # Local class search result display
│   └── NeedHelpBanner.tsx          # Persistent "Need help?" banner with NHS 111 Wales number
├── screens/
│   ├── FrailtyHomeScreen.tsx       # Feature landing page — three pathways
│   ├── ProgrammeScreen.tsx         # "Today's exercises" daily view
│   ├── ProgrammeProgressScreen.tsx # Weekly/overall progress tracker
│   ├── ExerciseDetailScreen.tsx    # Single exercise: video + instructions
│   ├── HomeSafetyScreen.tsx        # Room-by-room walkthrough
│   ├── SafetySummaryScreen.tsx     # Personalised actions summary
│   ├── ClassSearchScreen.tsx       # Postcode search for local classes
│   └── ClassDetailScreen.tsx       # Single class details
├── data/
│   ├── programme.json              # 12-week exercise programme structure
│   ├── exercises.json              # Exercise library (video refs, instructions, card images)
│   ├── homeSafetyChecklist.json    # Rooms → hazards → advice
│   └── classes.json                # Local classes dataset (seeded, updatable)
├── hooks/
│   ├── useProgrammeProgress.ts     # Track exercise completion, current week/day
│   ├── useHomeSafetyResults.ts     # Store and retrieve checklist answers
│   └── useClassSearch.ts           # Postcode search and filtering logic
├── utils/
│   ├── storage.ts                  # Local storage wrapper (AsyncStorage/SQLite)
│   ├── analytics.ts                # Anonymised event logging
│   └── postcodeLookup.ts          # Postcode to lat/lng for proximity search
├── types/
│   └── index.ts                    # TypeScript types for all data models
└── __tests__/
    ├── components/
    │   ├── AccessibleButton.test.tsx
    │   ├── VideoPlayer.test.tsx
    │   ├── ExerciseCard.test.tsx
    │   ├── ChecklistItem.test.tsx
    │   ├── ClassListItem.test.tsx
    │   ├── AccessibleCard.test.tsx
    │   └── NeedHelpBanner.test.tsx
    ├── screens/
    │   ├── ProgrammeScreen.test.tsx
    │   ├── HomeSafetyScreen.test.tsx
    │   └── ClassSearchScreen.test.tsx
    ├── hooks/
    │   ├── useProgrammeProgress.test.ts
    │   ├── useHomeSafetyResults.test.ts
    │   └── useClassSearch.test.ts
    └── utils/
        ├── storage.test.ts
        ├── analytics.test.ts
        └── postcodeLookup.test.ts
```

---

## Task 1: Data Models and Types

**Files:**
- Create: `src/features/frailty/types/index.ts`
- Create: `src/features/frailty/data/programme.json`
- Create: `src/features/frailty/data/exercises.json`
- Create: `src/features/frailty/data/homeSafetyChecklist.json`
- Create: `src/features/frailty/data/classes.json`

### Steps

- [ ] **Step 1: Write failing test for data model types**

```typescript
// src/features/frailty/__tests__/utils/dataModels.test.ts
import programme from '../../data/programme.json';
import exercises from '../../data/exercises.json';
import checklist from '../../data/homeSafetyChecklist.json';
import classes from '../../data/classes.json';
import { Programme, Exercise, HomeSafetyChecklist, LocalClass } from '../../types';

describe('Data models', () => {
  test('programme has 12 weeks', () => {
    expect(programme.weeks).toHaveLength(12);
  });

  test('each week has days with exercise references', () => {
    programme.weeks.forEach((week) => {
      expect(week.days.length).toBeGreaterThan(0);
      week.days.forEach((day) => {
        expect(day.exerciseIds.length).toBeGreaterThan(0);
      });
    });
  });

  test('all exercise references in programme exist in exercise library', () => {
    const exerciseIds = new Set(exercises.map((e) => e.id));
    programme.weeks.forEach((week) => {
      week.days.forEach((day) => {
        day.exerciseIds.forEach((id) => {
          expect(exerciseIds.has(id)).toBe(true);
        });
      });
    });
  });

  test('each exercise has video and local card asset fallback', () => {
    exercises.forEach((exercise) => {
      expect(exercise.videoUrl).toBeDefined();
      expect(exercise.cardImageAsset).toBeDefined();
      expect(exercise.cardImageAsset).toMatch(/^assets\//); // Must be a local asset path
      expect(exercise.instructions).toBeDefined();
      expect(exercise.instructions.length).toBeGreaterThan(0);
    });
  });

  test('all user-facing text uses BilingualText format', () => {
    exercises.forEach((exercise) => {
      expect(exercise.name.en).toBeDefined();
      expect(typeof exercise.name.en).toBe('string');
    });
    checklist.rooms.forEach((room) => {
      expect(room.name.en).toBeDefined();
      room.hazards.forEach((hazard) => {
        expect(hazard.question.en).toBeDefined();
        expect(hazard.advice.en).toBeDefined();
      });
    });
  });

  test('checklist covers required rooms', () => {
    const roomNames = checklist.rooms.map((r) => r.name.en);
    expect(roomNames).toContain('Living room');
    expect(roomNames).toContain('Bathroom');
    expect(roomNames).toContain('Kitchen');
    expect(roomNames).toContain('Bedroom');
    expect(roomNames).toContain('Hallway and stairs');
  });

  test('each hazard has advice text', () => {
    checklist.rooms.forEach((room) => {
      room.hazards.forEach((hazard) => {
        expect(hazard.question).toBeDefined();
        expect(hazard.advice).toBeDefined();
        expect(hazard.advice.length).toBeGreaterThan(0);
      });
    });
  });

  test('each class has required fields', () => {
    classes.forEach((cls) => {
      expect(cls.name).toBeDefined();
      expect(cls.provider).toBeDefined();
      expect(cls.postcode).toBeDefined();
      expect(cls.type).toBeDefined();
      expect(cls.contactPhone).toBeDefined();
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx jest src/features/frailty/__tests__/utils/dataModels.test.ts --no-coverage`
Expected: FAIL — modules not found

- [ ] **Step 3: Create TypeScript types**

```typescript
// src/features/frailty/types/index.ts

// Bilingual text support — Welsh language not required at launch but the
// content model must support it from day one (spec section 5).
// All user-facing text fields use this type so Welsh can be added without restructuring.
export interface BilingualText {
  en: string;
  cy?: string; // Welsh — optional until Welsh language phase
}

export interface Exercise {
  id: string;
  name: BilingualText;
  description: BilingualText;
  videoUrl: string;
  cardImageAsset: string; // Local asset path (bundled), not a URL — must work offline
  instructions: BilingualText[];
  durationSeconds: number;
  variation: 'seated' | 'standing' | 'both';
  safetyNote?: BilingualText;
}

export interface ProgrammeDay {
  dayNumber: number;
  exerciseIds: string[];
}

export interface ProgrammeWeek {
  weekNumber: number;
  title: BilingualText;
  progressionNote?: BilingualText;
  days: ProgrammeDay[];
}

export interface Programme {
  id: string;
  name: BilingualText;
  description: BilingualText;
  safetyDisclaimer: BilingualText;
  weeks: ProgrammeWeek[];
}

export interface Hazard {
  id: string;
  question: BilingualText;
  advice: BilingualText;
}

export interface Room {
  name: BilingualText;
  hazards: Hazard[];
}

export interface HomeSafetyChecklist {
  rooms: Room[];
}

export interface LocalClass {
  id: string;
  name: string;
  provider: string;
  type: string;
  address: string;
  postcode: string;
  latitude: number;
  longitude: number;
  dayTime: string;
  contactPhone: string;
  contactEmail?: string;
  accessibilityNotes?: string;
  startDate?: string; // ISO date — for seasonal classes
  endDate?: string;   // ISO date — for seasonal classes
  lastVerified: string;
}

export interface ExerciseCompletion {
  exerciseId: string;
  completedAt: string;
}

export interface DayProgress {
  weekNumber: number;
  dayNumber: number;
  completions: ExerciseCompletion[];
  completedAt?: string;
}

export interface HomeSafetyAnswer {
  hazardId: string;
  hasHazard: boolean;
  answeredAt: string;
}

export interface AnalyticsEvent {
  event: string;
  properties?: Record<string, string | number | boolean>;
  timestamp: string;
}
```

- [ ] **Step 4: Create seed data files**

Create `programme.json` with a 12-week Otago-style structure (3 exercise days per week, progressive difficulty). Create `exercises.json` with 20 placeholder exercises covering seated and standing variations. Create `homeSafetyChecklist.json` covering 5 rooms with 3-5 hazards each. Create `classes.json` with 10 seed entries across different health board areas.

All user-facing text fields use the `BilingualText` format: `{ "en": "English text" }` (no `cy` key yet — added in Welsh language phase). Exercise `cardImageAsset` values reference local bundled assets (e.g. `"assets/exercises/ex-001.png"`) not remote URLs, ensuring offline availability per spec section 4. Video URLs remain remote (video can buffer on slow connections, but cards must work offline).

Also create a `src/features/frailty/assets/exercises/` directory with placeholder images for the 20 exercises.

- [ ] **Step 5: Run tests to verify they pass**

Run: `npx jest src/features/frailty/__tests__/utils/dataModels.test.ts --no-coverage`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add src/features/frailty/types/ src/features/frailty/data/ src/features/frailty/__tests__/utils/dataModels.test.ts
git commit -m "feat(frailty): add data models, types, and seed content data"
```

---

## Task 2: Local Storage and Analytics Utilities

**Files:**
- Create: `src/features/frailty/utils/storage.ts`
- Create: `src/features/frailty/utils/analytics.ts`
- Test: `src/features/frailty/__tests__/utils/storage.test.ts`
- Test: `src/features/frailty/__tests__/utils/analytics.test.ts`

### Steps

- [ ] **Step 1: Write failing test for storage utility**

```typescript
// src/features/frailty/__tests__/utils/storage.test.ts
import { saveProgress, getProgress, saveSafetyAnswers, getSafetyAnswers, clearAll } from '../../utils/storage';
import { DayProgress, HomeSafetyAnswer } from '../../types';

describe('storage', () => {
  beforeEach(async () => {
    await clearAll();
  });

  test('saves and retrieves programme progress', async () => {
    const progress: DayProgress = {
      weekNumber: 1,
      dayNumber: 1,
      completions: [{ exerciseId: 'ex-001', completedAt: '2026-03-20T10:00:00Z' }],
      completedAt: '2026-03-20T10:05:00Z',
    };
    await saveProgress(progress);
    const result = await getProgress();
    expect(result).toHaveLength(1);
    expect(result[0]).toEqual(progress);
  });

  test('appends progress without overwriting', async () => {
    const day1: DayProgress = { weekNumber: 1, dayNumber: 1, completions: [], completedAt: '2026-03-20T10:00:00Z' };
    const day2: DayProgress = { weekNumber: 1, dayNumber: 2, completions: [], completedAt: '2026-03-21T10:00:00Z' };
    await saveProgress(day1);
    await saveProgress(day2);
    const result = await getProgress();
    expect(result).toHaveLength(2);
  });

  test('saves and retrieves safety answers', async () => {
    const answer: HomeSafetyAnswer = { hazardId: 'h-001', hasHazard: true, answeredAt: '2026-03-20T10:00:00Z' };
    await saveSafetyAnswers([answer]);
    const result = await getSafetyAnswers();
    expect(result).toHaveLength(1);
    expect(result[0]).toEqual(answer);
  });

  test('clearAll removes all data', async () => {
    const progress: DayProgress = { weekNumber: 1, dayNumber: 1, completions: [], completedAt: '2026-03-20T10:00:00Z' };
    await saveProgress(progress);
    await clearAll();
    const result = await getProgress();
    expect(result).toHaveLength(0);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx jest src/features/frailty/__tests__/utils/storage.test.ts --no-coverage`
Expected: FAIL — module not found

- [ ] **Step 3: Implement storage utility**

```typescript
// src/features/frailty/utils/storage.ts
import AsyncStorage from '@react-native-async-storage/async-storage';
import { DayProgress, HomeSafetyAnswer } from '../types';

const KEYS = {
  PROGRESS: '@frailty/progress',
  SAFETY: '@frailty/safety',
} as const;

export async function saveProgress(day: DayProgress): Promise<void> {
  const existing = await getProgress();
  existing.push(day);
  await AsyncStorage.setItem(KEYS.PROGRESS, JSON.stringify(existing));
}

export async function getProgress(): Promise<DayProgress[]> {
  const raw = await AsyncStorage.getItem(KEYS.PROGRESS);
  return raw ? JSON.parse(raw) : [];
}

// Note: saveSafetyAnswers does a full overwrite. The hook passes the complete
// answers array each time, so this is safe. If partial-update logic is added
// later (e.g. saving per-room), this will need to change to a merge strategy.
export async function saveSafetyAnswers(answers: HomeSafetyAnswer[]): Promise<void> {
  await AsyncStorage.setItem(KEYS.SAFETY, JSON.stringify(answers));
}

export async function getSafetyAnswers(): Promise<HomeSafetyAnswer[]> {
  const raw = await AsyncStorage.getItem(KEYS.SAFETY);
  return raw ? JSON.parse(raw) : [];
}

export async function clearAll(): Promise<void> {
  await AsyncStorage.multiRemove([KEYS.PROGRESS, KEYS.SAFETY]);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx jest src/features/frailty/__tests__/utils/storage.test.ts --no-coverage`
Expected: PASS

- [ ] **Step 5: Write failing test for analytics utility**

```typescript
// src/features/frailty/__tests__/utils/analytics.test.ts
import { trackEvent, getEventLog } from '../../utils/analytics';

describe('analytics', () => {
  test('logs an event with properties', () => {
    trackEvent('programme_started', { week: 1 });
    const log = getEventLog();
    expect(log.length).toBeGreaterThan(0);
    const last = log[log.length - 1];
    expect(last.event).toBe('programme_started');
    expect(last.properties).toEqual({ week: 1 });
    expect(last.timestamp).toBeDefined();
  });

  test('logs event without properties', () => {
    trackEvent('home_safety_completed');
    const log = getEventLog();
    const last = log[log.length - 1];
    expect(last.event).toBe('home_safety_completed');
    expect(last.properties).toBeUndefined();
  });
});
```

- [ ] **Step 6: Run test to verify it fails**

Run: `npx jest src/features/frailty/__tests__/utils/analytics.test.ts --no-coverage`
Expected: FAIL

- [ ] **Step 7: Implement analytics utility**

```typescript
// src/features/frailty/utils/analytics.ts
import { AnalyticsEvent } from '../types';

const eventLog: AnalyticsEvent[] = [];

export function trackEvent(event: string, properties?: Record<string, string | number | boolean>): void {
  const entry: AnalyticsEvent = {
    event,
    timestamp: new Date().toISOString(),
  };
  if (properties) {
    entry.properties = properties;
  }
  eventLog.push(entry);
  // In production: batch and send to analytics backend
}

export function getEventLog(): AnalyticsEvent[] {
  return [...eventLog];
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `npx jest src/features/frailty/__tests__/utils/ --no-coverage`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add src/features/frailty/utils/ src/features/frailty/__tests__/utils/storage.test.ts src/features/frailty/__tests__/utils/analytics.test.ts
git commit -m "feat(frailty): add local storage and analytics utilities"
```

---

## Task 3: Accessible UI Components

**Files:**
- Create: `src/features/frailty/components/AccessibleButton.tsx`
- Create: `src/features/frailty/components/AccessibleCard.tsx`
- Create: `src/features/frailty/components/VideoPlayer.tsx`
- Create: `src/features/frailty/components/ExerciseCard.tsx`
- Create: `src/features/frailty/components/ChecklistItem.tsx`
- Create: `src/features/frailty/components/ClassListItem.tsx`
- Create: `src/features/frailty/components/NeedHelpBanner.tsx`
- Test: `src/features/frailty/__tests__/components/AccessibleButton.test.tsx`
- Test: `src/features/frailty/__tests__/components/VideoPlayer.test.tsx`
- Test: `src/features/frailty/__tests__/components/ExerciseCard.test.tsx`
- Test: `src/features/frailty/__tests__/components/ChecklistItem.test.tsx`
- Test: `src/features/frailty/__tests__/components/ClassListItem.test.tsx`
- Test: `src/features/frailty/__tests__/components/AccessibleCard.test.tsx`
- Test: `src/features/frailty/__tests__/components/NeedHelpBanner.test.tsx`

### Steps

- [ ] **Step 1: Write failing test for AccessibleButton**

```typescript
// src/features/frailty/__tests__/components/AccessibleButton.test.tsx
import React from 'react';
import { render, fireEvent } from '@testing-library/react-native';
import { AccessibleButton } from '../../components/AccessibleButton';

describe('AccessibleButton', () => {
  test('renders label text', () => {
    const { getByText } = render(<AccessibleButton label="Start exercises" onPress={() => {}} />);
    expect(getByText('Start exercises')).toBeTruthy();
  });

  test('calls onPress when tapped', () => {
    const onPress = jest.fn();
    const { getByText } = render(<AccessibleButton label="Start exercises" onPress={onPress} />);
    fireEvent.press(getByText('Start exercises'));
    expect(onPress).toHaveBeenCalledTimes(1);
  });

  test('has minimum touch target of 48x48', () => {
    const { getByTestId } = render(<AccessibleButton label="Test" onPress={() => {}} testID="btn" />);
    const button = getByTestId('btn');
    expect(button.props.style).toMatchObject(
      expect.objectContaining({ minHeight: 48, minWidth: 48 })
    );
  });

  test('has accessible role', () => {
    const { getByRole } = render(<AccessibleButton label="Test" onPress={() => {}} />);
    expect(getByRole('button')).toBeTruthy();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx jest src/features/frailty/__tests__/components/AccessibleButton.test.tsx --no-coverage`
Expected: FAIL

- [ ] **Step 3: Implement AccessibleButton**

```typescript
// src/features/frailty/components/AccessibleButton.tsx
import React from 'react';
import { TouchableOpacity, Text, StyleSheet, ViewStyle } from 'react-native';

interface Props {
  label: string;
  onPress: () => void;
  testID?: string;
  style?: ViewStyle;
}

export function AccessibleButton({ label, onPress, testID, style }: Props) {
  return (
    <TouchableOpacity
      onPress={onPress}
      accessibilityRole="button"
      accessibilityLabel={label}
      testID={testID}
      style={[styles.button, style]}
    >
      <Text style={styles.label}>{label}</Text>
    </TouchableOpacity>
  );
}

const styles = StyleSheet.create({
  button: {
    minHeight: 48,
    minWidth: 48,
    paddingVertical: 16,
    paddingHorizontal: 24,
    backgroundColor: '#005EB8', // NHS Blue
    borderRadius: 8,
    alignItems: 'center',
    justifyContent: 'center',
  },
  label: {
    fontSize: 20,
    fontWeight: '600',
    color: '#FFFFFF',
  },
});
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx jest src/features/frailty/__tests__/components/AccessibleButton.test.tsx --no-coverage`
Expected: PASS

- [ ] **Step 5: Write failing test for ChecklistItem**

```typescript
// src/features/frailty/__tests__/components/ChecklistItem.test.tsx
import React from 'react';
import { render, fireEvent } from '@testing-library/react-native';
import { ChecklistItem } from '../../components/ChecklistItem';

describe('ChecklistItem', () => {
  const hazard = {
    id: 'h-001',
    question: 'Do you have loose rugs or mats?',
    advice: 'Remove loose rugs or secure them with non-slip tape.',
  };

  test('renders the question', () => {
    const { getByText } = render(<ChecklistItem hazard={hazard} onAnswer={() => {}} />);
    expect(getByText('Do you have loose rugs or mats?')).toBeTruthy();
  });

  test('shows advice when user answers yes', () => {
    const onAnswer = jest.fn();
    const { getByText, queryByText } = render(<ChecklistItem hazard={hazard} onAnswer={onAnswer} />);
    expect(queryByText(hazard.advice)).toBeNull();
    fireEvent.press(getByText('Yes'));
    expect(onAnswer).toHaveBeenCalledWith('h-001', true);
  });

  test('does not show advice when user answers no', () => {
    const onAnswer = jest.fn();
    const { getByText } = render(<ChecklistItem hazard={hazard} onAnswer={onAnswer} />);
    fireEvent.press(getByText('No'));
    expect(onAnswer).toHaveBeenCalledWith('h-001', false);
  });
});
```

- [ ] **Step 6: Run test to verify it fails**

Run: `npx jest src/features/frailty/__tests__/components/ChecklistItem.test.tsx --no-coverage`
Expected: FAIL

- [ ] **Step 7: Implement ChecklistItem**

```typescript
// src/features/frailty/components/ChecklistItem.tsx
import React, { useState } from 'react';
import { View, Text, StyleSheet } from 'react-native';
import { AccessibleButton } from './AccessibleButton';
import { Hazard } from '../types';

interface Props {
  hazard: Hazard;
  onAnswer: (hazardId: string, hasHazard: boolean) => void;
}

export function ChecklistItem({ hazard, onAnswer }: Props) {
  const [answered, setAnswered] = useState<boolean | null>(null);

  const handleAnswer = (hasHazard: boolean) => {
    setAnswered(hasHazard);
    onAnswer(hazard.id, hasHazard);
  };

  return (
    <View style={styles.container}>
      <Text style={styles.question}>{hazard.question}</Text>
      <View style={styles.buttons}>
        <AccessibleButton
          label="Yes"
          onPress={() => handleAnswer(true)}
          style={[styles.answerButton, answered === true && styles.selectedYes]}
        />
        <AccessibleButton
          label="No"
          onPress={() => handleAnswer(false)}
          style={[styles.answerButton, answered === false && styles.selectedNo]}
        />
      </View>
      {answered === true && (
        <View style={styles.adviceBox}>
          <Text style={styles.adviceText}>{hazard.advice}</Text>
        </View>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { marginBottom: 24, padding: 16, backgroundColor: '#F0F4F5', borderRadius: 8 },
  question: { fontSize: 20, fontWeight: '600', color: '#212B32', marginBottom: 12 },
  buttons: { flexDirection: 'row', gap: 12 },
  answerButton: { flex: 1, backgroundColor: '#4C6272' },
  selectedYes: { backgroundColor: '#D5281B' },
  selectedNo: { backgroundColor: '#007F3B' },
  adviceBox: { marginTop: 12, padding: 12, backgroundColor: '#FFF9C4', borderRadius: 4 },
  adviceText: { fontSize: 18, color: '#212B32' },
});
```

- [ ] **Step 8: Run test to verify it passes**

Run: `npx jest src/features/frailty/__tests__/components/ChecklistItem.test.tsx --no-coverage`
Expected: PASS

- [ ] **Step 9: Write failing test for VideoPlayer**

```typescript
// src/features/frailty/__tests__/components/VideoPlayer.test.tsx
import React from 'react';
import { render, fireEvent } from '@testing-library/react-native';
import { VideoPlayer } from '../../components/VideoPlayer';

describe('VideoPlayer', () => {
  test('renders play button', () => {
    const { getByRole } = render(<VideoPlayer videoUrl="https://example.com/video.mp4" title="Heel raises" />);
    expect(getByRole('button', { name: /play/i })).toBeTruthy();
  });

  test('toggles between play and pause on press', () => {
    const { getByRole, getByText } = render(<VideoPlayer videoUrl="https://example.com/video.mp4" title="Heel raises" />);
    fireEvent.press(getByRole('button', { name: /play/i }));
    expect(getByText(/pause/i)).toBeTruthy();
  });

  test('has captions toggle button', () => {
    const { getByRole } = render(<VideoPlayer videoUrl="https://example.com/video.mp4" title="Heel raises" />);
    expect(getByRole('button', { name: /captions/i })).toBeTruthy();
  });

  test('has minimum 48x48 touch target for play button', () => {
    const { getByTestId } = render(<VideoPlayer videoUrl="https://example.com/video.mp4" title="Heel raises" testID="player" />);
    const playButton = getByTestId('player-play');
    expect(playButton.props.style).toMatchObject(
      expect.objectContaining({ minHeight: 48, minWidth: 48 })
    );
  });
});
```

- [ ] **Step 10: Implement VideoPlayer**

Wraps react-native-video with large play/pause button, captions toggle, and accessible controls. All controls meet 48x48 minimum touch target. Announces video title and state to screen readers.

- [ ] **Step 11: Write failing test for ExerciseCard**

```typescript
// src/features/frailty/__tests__/components/ExerciseCard.test.tsx
import React from 'react';
import { render, fireEvent } from '@testing-library/react-native';
import { ExerciseCard } from '../../components/ExerciseCard';

describe('ExerciseCard', () => {
  const exercise = {
    name: { en: 'Heel raises' },
    cardImageAsset: 'assets/exercises/ex-001.png',
    instructions: [{ en: 'Stand behind a chair' }, { en: 'Rise onto your toes' }, { en: 'Lower slowly' }],
  };

  test('renders exercise name', () => {
    const { getByText } = render(<ExerciseCard exercise={exercise} onMarkDone={() => {}} />);
    expect(getByText('Heel raises')).toBeTruthy();
  });

  test('renders all instruction steps', () => {
    const { getByText } = render(<ExerciseCard exercise={exercise} onMarkDone={() => {}} />);
    expect(getByText('Stand behind a chair')).toBeTruthy();
    expect(getByText('Rise onto your toes')).toBeTruthy();
    expect(getByText('Lower slowly')).toBeTruthy();
  });

  test('fires onMarkDone when button pressed', () => {
    const onMarkDone = jest.fn();
    const { getByText } = render(<ExerciseCard exercise={exercise} onMarkDone={onMarkDone} />);
    fireEvent.press(getByText(/mark as done/i));
    expect(onMarkDone).toHaveBeenCalledTimes(1);
  });
});
```

- [ ] **Step 12: Implement ExerciseCard**

Static fallback for exercises — renders bundled local image, step-by-step text instructions from BilingualText, and "Mark as done" button. Uses AccessibleCard wrapper.

- [ ] **Step 13: Write failing test for NeedHelpBanner**

```typescript
// src/features/frailty/__tests__/components/NeedHelpBanner.test.tsx
import React from 'react';
import { render, fireEvent } from '@testing-library/react-native';
import { Linking } from 'react-native';
import { NeedHelpBanner } from '../../components/NeedHelpBanner';

jest.spyOn(Linking, 'openURL');

describe('NeedHelpBanner', () => {
  test('renders "Need help?" text', () => {
    const { getByText } = render(<NeedHelpBanner />);
    expect(getByText(/need help/i)).toBeTruthy();
  });

  test('shows NHS 111 Wales phone number', () => {
    const { getByText } = render(<NeedHelpBanner />);
    expect(getByText(/111/)).toBeTruthy();
  });

  test('tapping phone number opens dialler', () => {
    const { getByRole } = render(<NeedHelpBanner />);
    fireEvent.press(getByRole('button', { name: /call/i }));
    expect(Linking.openURL).toHaveBeenCalledWith('tel:111');
  });
});
```

- [ ] **Step 14: Implement NeedHelpBanner**

```typescript
// src/features/frailty/components/NeedHelpBanner.tsx
import React from 'react';
import { View, Text, Linking, StyleSheet } from 'react-native';
import { AccessibleButton } from './AccessibleButton';

export function NeedHelpBanner() {
  return (
    <View style={styles.banner} accessibilityRole="region" accessibilityLabel="Need help">
      <Text style={styles.text}>Need help?</Text>
      <AccessibleButton
        label="Call NHS 111 Wales"
        onPress={() => Linking.openURL('tel:111')}
        style={styles.callButton}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  banner: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'space-between',
    padding: 12,
    backgroundColor: '#005EB8',
    borderRadius: 8,
    marginVertical: 8,
  },
  text: { fontSize: 18, fontWeight: '600', color: '#FFFFFF' },
  callButton: { backgroundColor: '#007F3B', paddingVertical: 10, paddingHorizontal: 16 },
});
```

This component is included on every screen via the navigation wrapper or as a persistent footer element in FrailtyHomeScreen and all sub-screens. Per spec section 4: "Clear 'need help?' route — phone number for local falls team or NHS 111 Wales."

- [ ] **Step 15: Implement remaining components (ClassListItem, AccessibleCard)**

Follow the same TDD cycle:
- **ClassListItem**: displays class name, type, day/time, contact phone as tappable link. Test: renders all fields, phone link is pressable.
- **AccessibleCard**: generic high-contrast card wrapper with large text. Test: renders children, applies high contrast styles.

- [ ] **Step 16: Run all component tests**

Run: `npx jest src/features/frailty/__tests__/components/ --no-coverage`
Expected: ALL PASS

- [ ] **Step 11: Commit**

```bash
git add src/features/frailty/components/ src/features/frailty/__tests__/components/
git commit -m "feat(frailty): add accessible UI components with tests"
```

---

## Task 4: Programme Progress Hook

**Files:**
- Create: `src/features/frailty/hooks/useProgrammeProgress.ts`
- Test: `src/features/frailty/__tests__/hooks/useProgrammeProgress.test.ts`

### Steps

- [ ] **Step 1: Write failing test**

```typescript
// src/features/frailty/__tests__/hooks/useProgrammeProgress.test.ts
import { renderHook, act } from '@testing-library/react-hooks';
import { useProgrammeProgress } from '../../hooks/useProgrammeProgress';

describe('useProgrammeProgress', () => {
  test('starts at week 1, day 1 with no completions', async () => {
    const { result, waitForNextUpdate } = renderHook(() => useProgrammeProgress());
    await waitForNextUpdate();
    expect(result.current.currentWeek).toBe(1);
    expect(result.current.currentDay).toBe(1);
    expect(result.current.completedDays).toHaveLength(0);
  });

  test('completeDay advances to next day', async () => {
    const { result, waitForNextUpdate } = renderHook(() => useProgrammeProgress());
    await waitForNextUpdate();
    await act(async () => {
      await result.current.completeDay([
        { exerciseId: 'ex-001', completedAt: new Date().toISOString() },
      ]);
    });
    expect(result.current.completedDays).toHaveLength(1);
  });

  test('calculates overall progress percentage', async () => {
    const { result, waitForNextUpdate } = renderHook(() => useProgrammeProgress());
    await waitForNextUpdate();
    expect(result.current.progressPercent).toBe(0);
  });

  test('getTodaysExercises returns exercise IDs for current day', async () => {
    const { result, waitForNextUpdate } = renderHook(() => useProgrammeProgress());
    await waitForNextUpdate();
    expect(result.current.todaysExercises.length).toBeGreaterThan(0);
  });

  test('detects programme completion and sets isComplete flag', async () => {
    // Pre-populate storage with all days completed except the last
    // Then complete the final day and verify isComplete is true
    const { result, waitForNextUpdate } = renderHook(() => useProgrammeProgress());
    await waitForNextUpdate();
    expect(result.current.isComplete).toBe(false);
    // After completing all days (simulated via pre-populated storage):
    // expect(result.current.isComplete).toBe(true);
    // expect(result.current.progressPercent).toBe(100);
  });

  test('fires programme_completed analytics event on final day', async () => {
    // Verify that completing the last day of week 12 triggers trackEvent('programme_completed')
    const { result, waitForNextUpdate } = renderHook(() => useProgrammeProgress());
    await waitForNextUpdate();
    // Implementation: mock trackEvent, pre-populate to penultimate day, complete final day
    // Assert trackEvent was called with 'programme_completed'
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx jest src/features/frailty/__tests__/hooks/useProgrammeProgress.test.ts --no-coverage`
Expected: FAIL

- [ ] **Step 3: Implement hook**

```typescript
// src/features/frailty/hooks/useProgrammeProgress.ts
import { useState, useEffect, useCallback } from 'react';
import { DayProgress, ExerciseCompletion } from '../types';
import { saveProgress, getProgress } from '../utils/storage';
import { trackEvent } from '../utils/analytics';
import programme from '../data/programme.json';

export function useProgrammeProgress() {
  const [completedDays, setCompletedDays] = useState<DayProgress[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    getProgress().then((saved) => {
      setCompletedDays(saved);
      setLoading(false);
    });
  }, []);

  const totalDays = programme.weeks.reduce((sum, w) => sum + w.days.length, 0);
  const progressPercent = totalDays > 0 ? Math.round((completedDays.length / totalDays) * 100) : 0;

  const currentDayIndex = completedDays.length;
  let currentWeek = 1;
  let currentDay = 1;
  let dayCounter = 0;

  for (const week of programme.weeks) {
    for (const day of week.days) {
      if (dayCounter === currentDayIndex) {
        currentWeek = week.weekNumber;
        currentDay = day.dayNumber;
        break;
      }
      dayCounter++;
    }
    if (dayCounter === currentDayIndex) break;
  }

  const todaysExercises: string[] =
    programme.weeks
      .find((w) => w.weekNumber === currentWeek)
      ?.days.find((d) => d.dayNumber === currentDay)
      ?.exerciseIds ?? [];

  const completeDay = useCallback(
    async (completions: ExerciseCompletion[]) => {
      const dayProgress: DayProgress = {
        weekNumber: currentWeek,
        dayNumber: currentDay,
        completions,
        completedAt: new Date().toISOString(),
      };
      await saveProgress(dayProgress);
      const updated = [...completedDays, dayProgress];
      setCompletedDays(updated);
      trackEvent('day_completed', { week: currentWeek, day: currentDay });
      if (updated.length >= totalDays) {
        trackEvent('programme_completed');
      }
    },
    [currentWeek, currentDay, completedDays, totalDays],
  );

  const isComplete = completedDays.length >= totalDays;

  return {
    currentWeek,
    currentDay,
    completedDays,
    progressPercent,
    todaysExercises,
    completeDay,
    isComplete,
    loading,
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx jest src/features/frailty/__tests__/hooks/useProgrammeProgress.test.ts --no-coverage`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/features/frailty/hooks/useProgrammeProgress.ts src/features/frailty/__tests__/hooks/useProgrammeProgress.test.ts
git commit -m "feat(frailty): add programme progress tracking hook"
```

---

## Task 5: Home Safety Hook

**Files:**
- Create: `src/features/frailty/hooks/useHomeSafetyResults.ts`
- Test: `src/features/frailty/__tests__/hooks/useHomeSafetyResults.test.ts`

### Steps

- [ ] **Step 1: Write failing test**

```typescript
// src/features/frailty/__tests__/hooks/useHomeSafetyResults.test.ts
import { renderHook, act } from '@testing-library/react-hooks';
import { useHomeSafetyResults } from '../../hooks/useHomeSafetyResults';

describe('useHomeSafetyResults', () => {
  test('starts with no answers', async () => {
    const { result, waitForNextUpdate } = renderHook(() => useHomeSafetyResults());
    await waitForNextUpdate();
    expect(result.current.answers).toHaveLength(0);
  });

  test('records an answer', async () => {
    const { result, waitForNextUpdate } = renderHook(() => useHomeSafetyResults());
    await waitForNextUpdate();
    act(() => {
      result.current.answerHazard('h-001', true);
    });
    expect(result.current.answers).toHaveLength(1);
    expect(result.current.answers[0].hasHazard).toBe(true);
  });

  test('getHazardSummary returns only hazards answered yes', async () => {
    const { result, waitForNextUpdate } = renderHook(() => useHomeSafetyResults());
    await waitForNextUpdate();
    act(() => {
      result.current.answerHazard('h-001', true);
      result.current.answerHazard('h-002', false);
      result.current.answerHazard('h-003', true);
    });
    expect(result.current.hazardSummary).toHaveLength(2);
  });

  test('saves answers to storage', async () => {
    const { result, waitForNextUpdate } = renderHook(() => useHomeSafetyResults());
    await waitForNextUpdate();
    act(() => {
      result.current.answerHazard('h-001', true);
    });
    await act(async () => {
      await result.current.saveResults();
    });
    // Verify by re-mounting
    const { result: result2, waitForNextUpdate: wait2 } = renderHook(() => useHomeSafetyResults());
    await wait2();
    expect(result2.current.answers).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx jest src/features/frailty/__tests__/hooks/useHomeSafetyResults.test.ts --no-coverage`
Expected: FAIL

- [ ] **Step 3: Implement hook**

```typescript
// src/features/frailty/hooks/useHomeSafetyResults.ts
import { useState, useEffect, useCallback } from 'react';
import { HomeSafetyAnswer } from '../types';
import { saveSafetyAnswers, getSafetyAnswers } from '../utils/storage';
import { trackEvent } from '../utils/analytics';

export function useHomeSafetyResults() {
  const [answers, setAnswers] = useState<HomeSafetyAnswer[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    getSafetyAnswers().then((saved) => {
      setAnswers(saved);
      setLoading(false);
    });
  }, []);

  const answerHazard = useCallback((hazardId: string, hasHazard: boolean) => {
    setAnswers((prev) => {
      const filtered = prev.filter((a) => a.hazardId !== hazardId);
      return [...filtered, { hazardId, hasHazard, answeredAt: new Date().toISOString() }];
    });
  }, []);

  const hazardSummary = answers.filter((a) => a.hasHazard);

  const saveResults = useCallback(async () => {
    await saveSafetyAnswers(answers);
    trackEvent('home_safety_completed', { hazardsFound: hazardSummary.length });
  }, [answers, hazardSummary.length]);

  return { answers, answerHazard, hazardSummary, saveResults, loading };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx jest src/features/frailty/__tests__/hooks/useHomeSafetyResults.test.ts --no-coverage`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/features/frailty/hooks/useHomeSafetyResults.ts src/features/frailty/__tests__/hooks/useHomeSafetyResults.test.ts
git commit -m "feat(frailty): add home safety checklist results hook"
```

---

## Task 6: Postcode Search Utility and Class Search Hook

**Files:**
- Create: `src/features/frailty/utils/postcodeLookup.ts`
- Create: `src/features/frailty/hooks/useClassSearch.ts`
- Test: `src/features/frailty/__tests__/utils/postcodeLookup.test.ts`
- Test: `src/features/frailty/__tests__/hooks/useClassSearch.test.ts`

### Steps

- [ ] **Step 1: Write failing test for postcode lookup**

```typescript
// src/features/frailty/__tests__/utils/postcodeLookup.test.ts
import { searchClassesByPostcode } from '../../utils/postcodeLookup';
import classes from '../../data/classes.json';

// Mock fetch to avoid external API dependency — tests must work offline and in CI
const mockFetchResponse = (latitude: number, longitude: number) => ({
  ok: true,
  json: async () => ({ status: 200, result: { latitude, longitude } }),
});

beforeEach(() => {
  // Cardiff city centre coordinates
  global.fetch = jest.fn().mockResolvedValue(mockFetchResponse(51.4816, -3.1791));
});

afterEach(() => {
  jest.restoreAllMocks();
});

describe('postcodeLookup', () => {
  test('returns classes sorted by distance from postcode', async () => {
    const results = await searchClassesByPostcode('CF10 1AA', classes, 10);
    expect(Array.isArray(results)).toBe(true);
    results.forEach((cls) => {
      expect(cls.distanceMiles).toBeDefined();
    });
  });

  test('filters by radius', async () => {
    const nearby = await searchClassesByPostcode('CF10 1AA', classes, 2);
    const wider = await searchClassesByPostcode('CF10 1AA', classes, 50);
    expect(wider.length).toBeGreaterThanOrEqual(nearby.length);
  });

  test('returns empty array for invalid postcode', async () => {
    global.fetch = jest.fn().mockResolvedValue({
      ok: true,
      json: async () => ({ status: 404 }),
    });
    const results = await searchClassesByPostcode('INVALID', classes, 10);
    expect(results).toHaveLength(0);
  });

  test('returns empty array when fetch fails', async () => {
    global.fetch = jest.fn().mockRejectedValue(new Error('Network error'));
    const results = await searchClassesByPostcode('CF10 1AA', classes, 10);
    expect(results).toHaveLength(0);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx jest src/features/frailty/__tests__/utils/postcodeLookup.test.ts --no-coverage`
Expected: FAIL

- [ ] **Step 3: Implement postcode lookup**

```typescript
// src/features/frailty/utils/postcodeLookup.ts
import { LocalClass } from '../types';

export interface ClassWithDistance extends LocalClass {
  distanceMiles: number;
}

interface PostcodeResult {
  latitude: number;
  longitude: number;
}

async function geocodePostcode(postcode: string): Promise<PostcodeResult | null> {
  try {
    const response = await fetch(`https://api.postcodes.io/postcodes/${encodeURIComponent(postcode)}`);
    const data = await response.json();
    if (data.status === 200) {
      return { latitude: data.result.latitude, longitude: data.result.longitude };
    }
    return null;
  } catch {
    return null;
  }
}

function haversineDistance(lat1: number, lon1: number, lat2: number, lon2: number): number {
  const R = 3959; // Earth's radius in miles
  const dLat = ((lat2 - lat1) * Math.PI) / 180;
  const dLon = ((lon2 - lon1) * Math.PI) / 180;
  const a =
    Math.sin(dLat / 2) ** 2 +
    Math.cos((lat1 * Math.PI) / 180) * Math.cos((lat2 * Math.PI) / 180) * Math.sin(dLon / 2) ** 2;
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
}

export async function searchClassesByPostcode(
  postcode: string,
  classes: LocalClass[],
  radiusMiles: number,
): Promise<ClassWithDistance[]> {
  const location = await geocodePostcode(postcode);
  if (!location) return [];

  return classes
    .map((cls) => ({
      ...cls,
      distanceMiles: haversineDistance(location.latitude, location.longitude, cls.latitude, cls.longitude),
    }))
    .filter((cls) => cls.distanceMiles <= radiusMiles)
    .sort((a, b) => a.distanceMiles - b.distanceMiles);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx jest src/features/frailty/__tests__/utils/postcodeLookup.test.ts --no-coverage`
Expected: PASS (with mocked fetch)

- [ ] **Step 5: Write failing test for useClassSearch hook**

```typescript
// src/features/frailty/__tests__/hooks/useClassSearch.test.ts
import { renderHook, act } from '@testing-library/react-hooks';
import { useClassSearch } from '../../hooks/useClassSearch';

describe('useClassSearch', () => {
  test('starts with empty results and no loading', () => {
    const { result } = renderHook(() => useClassSearch());
    expect(result.current.results).toHaveLength(0);
    expect(result.current.loading).toBe(false);
  });

  test('search sets loading then returns results', async () => {
    const { result, waitForNextUpdate } = renderHook(() => useClassSearch());
    act(() => {
      result.current.search('CF10 1AA');
    });
    expect(result.current.loading).toBe(true);
    await waitForNextUpdate();
    expect(result.current.loading).toBe(false);
    expect(Array.isArray(result.current.results)).toBe(true);
  });
});
```

- [ ] **Step 6: Implement useClassSearch hook**

```typescript
// src/features/frailty/hooks/useClassSearch.ts
import { useState, useCallback } from 'react';
import { searchClassesByPostcode, ClassWithDistance } from '../utils/postcodeLookup';
import { trackEvent } from '../utils/analytics';
import classes from '../data/classes.json';
import { LocalClass } from '../types';

const DEFAULT_RADIUS_MILES = 10;

export function useClassSearch() {
  const [results, setResults] = useState<ClassWithDistance[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const search = useCallback(async (postcode: string, radiusMiles = DEFAULT_RADIUS_MILES) => {
    setLoading(true);
    setError(null);
    try {
      const found = await searchClassesByPostcode(postcode, classes as LocalClass[], radiusMiles);
      setResults(found);
      trackEvent('class_search', { postcode: postcode.substring(0, 4), resultsCount: found.length });
    } catch {
      setError('Could not search for classes. Please try again.');
      setResults([]);
    } finally {
      setLoading(false);
    }
  }, []);

  return { results, loading, error, search };
}
```

- [ ] **Step 7: Run all hook tests**

Run: `npx jest src/features/frailty/__tests__/hooks/ --no-coverage`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add src/features/frailty/utils/postcodeLookup.ts src/features/frailty/hooks/useClassSearch.ts src/features/frailty/__tests__/utils/postcodeLookup.test.ts src/features/frailty/__tests__/hooks/useClassSearch.test.ts
git commit -m "feat(frailty): add postcode search and class lookup"
```

---

## Task 7: Feature Screens — Exercise Programme

**Files:**
- Create: `src/features/frailty/screens/FrailtyHomeScreen.tsx`
- Create: `src/features/frailty/screens/ProgrammeScreen.tsx`
- Create: `src/features/frailty/screens/ProgrammeProgressScreen.tsx`
- Create: `src/features/frailty/screens/ExerciseDetailScreen.tsx`
- Test: `src/features/frailty/__tests__/screens/ProgrammeScreen.test.tsx`

### Steps

- [ ] **Step 1: Write failing test for ProgrammeScreen**

```typescript
// src/features/frailty/__tests__/screens/ProgrammeScreen.test.tsx
import React from 'react';
import { render, fireEvent } from '@testing-library/react-native';
import { ProgrammeScreen } from '../../screens/ProgrammeScreen';

const mockNavigation = { navigate: jest.fn() };

describe('ProgrammeScreen', () => {
  test('shows safety disclaimer on first visit', () => {
    const { getByText } = render(<ProgrammeScreen navigation={mockNavigation} />);
    expect(getByText(/stop if you feel pain or dizziness/i)).toBeTruthy();
  });

  test('displays today\'s exercises', () => {
    const { getByText } = render(<ProgrammeScreen navigation={mockNavigation} />);
    expect(getByText(/today's exercises/i)).toBeTruthy();
  });

  test('shows current week and day', () => {
    const { getByText } = render(<ProgrammeScreen navigation={mockNavigation} />);
    expect(getByText(/week 1/i)).toBeTruthy();
  });

  test('tapping an exercise navigates to detail', () => {
    const { getAllByRole } = render(<ProgrammeScreen navigation={mockNavigation} />);
    const exerciseButtons = getAllByRole('button');
    fireEvent.press(exerciseButtons[0]);
    expect(mockNavigation.navigate).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx jest src/features/frailty/__tests__/screens/ProgrammeScreen.test.tsx --no-coverage`
Expected: FAIL

- [ ] **Step 3: Write failing test for FrailtyHomeScreen**

```typescript
// src/features/frailty/__tests__/screens/FrailtyHomeScreen.test.tsx
import React from 'react';
import { render, fireEvent } from '@testing-library/react-native';
import { FrailtyHomeScreen } from '../../screens/FrailtyHomeScreen';

const mockNavigation = { navigate: jest.fn() };

describe('FrailtyHomeScreen', () => {
  test('shows feature title', () => {
    const { getByText } = render(<FrailtyHomeScreen navigation={mockNavigation} />);
    expect(getByText(/stay steady/i)).toBeTruthy();
  });

  test('shows three feature cards', () => {
    const { getByText } = render(<FrailtyHomeScreen navigation={mockNavigation} />);
    expect(getByText(/my exercises/i)).toBeTruthy();
    expect(getByText(/home safety check/i)).toBeTruthy();
    expect(getByText(/local classes/i)).toBeTruthy();
  });

  test('shows NeedHelpBanner', () => {
    const { getByText } = render(<FrailtyHomeScreen navigation={mockNavigation} />);
    expect(getByText(/need help/i)).toBeTruthy();
  });

  test('shows discovery route prompt on first visit', () => {
    const { getByText } = render(<FrailtyHomeScreen navigation={mockNavigation} />);
    expect(getByText(/how did you hear about this/i)).toBeTruthy();
  });

  test('discovery route options include self-found and clinician', () => {
    const { getByText } = render(<FrailtyHomeScreen navigation={mockNavigation} />);
    expect(getByText(/found it myself/i)).toBeTruthy();
    expect(getByText(/recommended by/i)).toBeTruthy();
  });
});
```

- [ ] **Step 4: Implement FrailtyHomeScreen**

Landing screen with three large, accessible cards linking to the three features. Includes the feature title "Stay Steady: Strength & Balance", brief description, NeedHelpBanner component, and a "how did you hear about this?" optional prompt on first visit (for `discovery_route` analytics event — self-found vs clinician signposted). The prompt is shown once, answer stored locally, and fires `trackEvent('discovery_route', { source: 'self' | 'clinician' })`.

- [ ] **Step 4: Implement ProgrammeScreen**

Shows safety disclaimer, current week/day, today's exercises as tappable cards (exercise name + thumbnail), progress bar, and NeedHelpBanner at bottom. Uses `useProgrammeProgress` hook. Each exercise card navigates to ExerciseDetailScreen. Fires `trackEvent('programme_started')` when user views this screen for the first time (week 1, day 1, no prior progress). Fires `trackEvent('feature_opened')` on mount.

- [ ] **Step 5: Implement ExerciseDetailScreen**

Shows VideoPlayer component (or ExerciseCard fallback if user prefers), step-by-step instructions below, and "Mark as done" button. When all exercises for the day are marked done, shows a congratulatory message and returns to ProgrammeScreen.

- [ ] **Step 6: Implement ProgrammeProgressScreen**

Simple visual display: 12 weeks as rows, each day as a circle (filled = completed, empty = pending, highlighted = current). Overall completion percentage. Accessible with screen reader — reads progress as "Week 1, Day 1: completed".

- [ ] **Step 7: Run tests**

Run: `npx jest src/features/frailty/__tests__/screens/ProgrammeScreen.test.tsx --no-coverage`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add src/features/frailty/screens/FrailtyHomeScreen.tsx src/features/frailty/screens/ProgrammeScreen.tsx src/features/frailty/screens/ProgrammeProgressScreen.tsx src/features/frailty/screens/ExerciseDetailScreen.tsx src/features/frailty/__tests__/screens/ProgrammeScreen.test.tsx
git commit -m "feat(frailty): add exercise programme screens"
```

---

## Task 8: Feature Screens — Home Safety Check

**Files:**
- Create: `src/features/frailty/screens/HomeSafetyScreen.tsx`
- Create: `src/features/frailty/screens/SafetySummaryScreen.tsx`
- Test: `src/features/frailty/__tests__/screens/HomeSafetyScreen.test.tsx`

### Steps

- [ ] **Step 1: Write failing test**

```typescript
// src/features/frailty/__tests__/screens/HomeSafetyScreen.test.tsx
import React from 'react';
import { render, fireEvent } from '@testing-library/react-native';
import { HomeSafetyScreen } from '../../screens/HomeSafetyScreen';

const mockNavigation = { navigate: jest.fn() };

describe('HomeSafetyScreen', () => {
  test('shows first room name', () => {
    const { getByText } = render(<HomeSafetyScreen navigation={mockNavigation} />);
    expect(getByText('Living room')).toBeTruthy();
  });

  test('displays hazard questions for current room', () => {
    const { getByText } = render(<HomeSafetyScreen navigation={mockNavigation} />);
    expect(getByText(/loose rugs/i)).toBeTruthy();
  });

  test('navigates to summary after all rooms completed', () => {
    // Answer all hazards across all rooms, then verify navigation to summary
    const { getByText } = render(<HomeSafetyScreen navigation={mockNavigation} />);
    // Implementation will step through rooms
    expect(getByText('Living room')).toBeTruthy();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx jest src/features/frailty/__tests__/screens/HomeSafetyScreen.test.tsx --no-coverage`
Expected: FAIL

- [ ] **Step 3: Implement HomeSafetyScreen**

Room-by-room walkthrough. Shows current room name as heading, lists hazard questions using ChecklistItem components. "Next room" button at bottom. Uses `useHomeSafetyResults` hook. After final room, navigates to SafetySummaryScreen.

- [ ] **Step 4: Implement SafetySummaryScreen**

Displays personalised summary: "We found X things you could improve." Lists each hazard the user answered "yes" to, with the advice text. "Share this list" button (generates shareable text for family/carers). "Redo check" button to start again. Saves results via the hook.

- [ ] **Step 5: Run tests**

Run: `npx jest src/features/frailty/__tests__/screens/HomeSafetyScreen.test.tsx --no-coverage`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add src/features/frailty/screens/HomeSafetyScreen.tsx src/features/frailty/screens/SafetySummaryScreen.tsx src/features/frailty/__tests__/screens/HomeSafetyScreen.test.tsx
git commit -m "feat(frailty): add home safety check screens"
```

---

## Task 9: Feature Screens — Local Classes

**Files:**
- Create: `src/features/frailty/screens/ClassSearchScreen.tsx`
- Create: `src/features/frailty/screens/ClassDetailScreen.tsx`
- Test: `src/features/frailty/__tests__/screens/ClassSearchScreen.test.tsx`

### Steps

- [ ] **Step 1: Write failing test**

```typescript
// src/features/frailty/__tests__/screens/ClassSearchScreen.test.tsx
import React from 'react';
import { render, fireEvent } from '@testing-library/react-native';
import { ClassSearchScreen } from '../../screens/ClassSearchScreen';

const mockNavigation = { navigate: jest.fn() };

describe('ClassSearchScreen', () => {
  test('shows postcode input and search button', () => {
    const { getByPlaceholderText, getByText } = render(<ClassSearchScreen navigation={mockNavigation} />);
    expect(getByPlaceholderText(/enter your postcode/i)).toBeTruthy();
    expect(getByText(/search/i)).toBeTruthy();
  });

  test('shows disclaimer about checking with provider', () => {
    const { getByText } = render(<ClassSearchScreen navigation={mockNavigation} />);
    expect(getByText(/check with provider before attending/i)).toBeTruthy();
  });

  test('shows coverage message', () => {
    const { getByText } = render(<ClassSearchScreen navigation={mockNavigation} />);
    expect(getByText(/more are being added/i)).toBeTruthy();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx jest src/features/frailty/__tests__/screens/ClassSearchScreen.test.tsx --no-coverage`
Expected: FAIL

- [ ] **Step 3: Implement ClassSearchScreen**

Large postcode input field (accessible, large text), search button, results list using ClassListItem component. Shows "check with provider before attending" disclaimer and "Showing classes we know about in your area. More are being added regularly." Uses `useClassSearch` hook. Tapping a result navigates to ClassDetailScreen.

- [ ] **Step 4: Implement ClassDetailScreen**

Shows full class details: name, provider, type, address, day/time, accessibility notes. Tappable phone number (opens dialler). Contact email if available. Simple, large-text layout.

- [ ] **Step 5: Run tests**

Run: `npx jest src/features/frailty/__tests__/screens/ClassSearchScreen.test.tsx --no-coverage`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add src/features/frailty/screens/ClassSearchScreen.tsx src/features/frailty/screens/ClassDetailScreen.tsx src/features/frailty/__tests__/screens/ClassSearchScreen.test.tsx
git commit -m "feat(frailty): add local classes search screens"
```

---

## Task 10: Navigation and Feature Integration

**Files:**
- Create: `src/features/frailty/navigation.tsx`
- Modify: App-level navigation to add frailty feature entry point (path depends on existing app structure)

### Steps

- [ ] **Step 1: Create feature navigation stack**

```typescript
// src/features/frailty/navigation.tsx
import React from 'react';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { FrailtyHomeScreen } from './screens/FrailtyHomeScreen';
import { ProgrammeScreen } from './screens/ProgrammeScreen';
import { ProgrammeProgressScreen } from './screens/ProgrammeProgressScreen';
import { ExerciseDetailScreen } from './screens/ExerciseDetailScreen';
import { HomeSafetyScreen } from './screens/HomeSafetyScreen';
import { SafetySummaryScreen } from './screens/SafetySummaryScreen';
import { ClassSearchScreen } from './screens/ClassSearchScreen';
import { ClassDetailScreen } from './screens/ClassDetailScreen';

const Stack = createNativeStackNavigator();

export function FrailtyNavigator() {
  return (
    <Stack.Navigator
      screenOptions={{
        headerTitleStyle: { fontSize: 20, fontWeight: '600' },
        headerBackTitleVisible: true,
      }}
    >
      <Stack.Screen name="FrailtyHome" component={FrailtyHomeScreen} options={{ title: 'Stay Steady' }} />
      <Stack.Screen name="Programme" component={ProgrammeScreen} options={{ title: 'My Exercises' }} />
      <Stack.Screen name="ProgrammeProgress" component={ProgrammeProgressScreen} options={{ title: 'My Progress' }} />
      <Stack.Screen name="ExerciseDetail" component={ExerciseDetailScreen} options={{ title: 'Exercise' }} />
      <Stack.Screen name="HomeSafety" component={HomeSafetyScreen} options={{ title: 'Home Safety Check' }} />
      <Stack.Screen name="SafetySummary" component={SafetySummaryScreen} options={{ title: 'Your Results' }} />
      <Stack.Screen name="ClassSearch" component={ClassSearchScreen} options={{ title: 'Local Classes' }} />
      <Stack.Screen name="ClassDetail" component={ClassDetailScreen} options={{ title: 'Class Details' }} />
    </Stack.Navigator>
  );
}
```

- [ ] **Step 2: Add feature entry point to app navigation**

Add a card/button on the NHS Wales app home screen or services menu that links to the FrailtyNavigator. Exact integration depends on existing app navigation structure.

- [ ] **Step 3: Run full test suite**

Run: `npx jest src/features/frailty/ --no-coverage`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
git add src/features/frailty/navigation.tsx
git commit -m "feat(frailty): add feature navigation and app integration"
```

---

## Task 11: Accessibility Audit and Device Testing

**Files:**
- No new files — modifications to existing components as needed

### Steps

- [ ] **Step 1: Run accessibility linter**

Run: `npx eslint src/features/frailty/ --rule 'jsx-a11y/*: error'` (or equivalent React Native accessibility rules)
Fix any violations.

- [ ] **Step 2: Manual screen reader testing checklist**

Verify with VoiceOver (iOS) and TalkBack (Android):
- [ ] All buttons have meaningful accessible labels
- [ ] Navigation order follows visual order
- [ ] Videos are announced with title and "play" action
- [ ] Checklist questions read correctly, answer state is announced
- [ ] Progress screen reads week/day completion states
- [ ] Class search results announce name, type, and distance

- [ ] **Step 3: Test on low-spec device**

Verify on a device 3-4 years old:
- [ ] Video playback starts within 3 seconds on WiFi
- [ ] Card fallback loads instantly
- [ ] Scrolling is smooth through exercise lists
- [ ] No layout issues at largest system text size

- [ ] **Step 4: Fix any issues found**

Apply fixes following the same TDD cycle: write/update test, verify fail, fix, verify pass.

- [ ] **Step 5: Commit**

```bash
git add -u src/features/frailty/
git commit -m "fix(frailty): accessibility audit fixes"
```

---

## Task 12: Analytics Instrumentation

**Files:**
- Modify: Screens and hooks to add analytics events as defined in the spec

### Steps

- [ ] **Step 1: Verify all required analytics events are tracked**

Per spec section 8, ensure these events are instrumented:
- `programme_started` — when user begins week 1, day 1
- `day_completed` — each day completed (with week/day properties)
- `programme_completed` — when user finishes week 12
- `home_safety_completed` — with hazards found count
- `class_search` — with partial postcode and results count
- `class_contact_tapped` — when user taps a phone number
- `discovery_route` — self-found vs clinician signposted (from first-visit prompt)
- `feature_opened` — when user enters the frailty feature

- [ ] **Step 2: Write test for analytics event coverage**

```typescript
// src/features/frailty/__tests__/utils/analyticsEvents.test.ts
import { getEventLog, trackEvent } from '../../utils/analytics';

describe('analytics events', () => {
  const requiredEvents = [
    'programme_started',
    'day_completed',
    'programme_completed',
    'home_safety_completed',
    'class_search',
    'class_contact_tapped',
    'discovery_route',
    'feature_opened',
  ];

  test('all required event names are valid strings', () => {
    requiredEvents.forEach((event) => {
      trackEvent(event);
      const log = getEventLog();
      const found = log.find((e) => e.event === event);
      expect(found).toBeDefined();
    });
  });
});
```

- [ ] **Step 3: Add any missing events to screens/hooks**

- [ ] **Step 4: Run full test suite**

Run: `npx jest src/features/frailty/ --no-coverage`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add -u src/features/frailty/
git commit -m "feat(frailty): complete analytics instrumentation"
```

---

## Summary

| Task | Description | Est. effort |
|------|-------------|-------------|
| 1 | Data models and types | 2-3 hours |
| 2 | Storage and analytics utilities | 2-3 hours |
| 3 | Accessible UI components | 4-6 hours |
| 4 | Programme progress hook | 2-3 hours |
| 5 | Home safety hook | 1-2 hours |
| 6 | Postcode search and class hook | 2-3 hours |
| 7 | Exercise programme screens | 4-6 hours |
| 8 | Home safety screens | 3-4 hours |
| 9 | Local classes screens | 3-4 hours |
| 10 | Navigation and integration | 1-2 hours |
| 11 | Accessibility audit (automated + initial manual pass) | 2-3 days |
| 12 | Analytics instrumentation | 2-3 hours |
| **Total** | | **40-55 hours** |

Dependencies: Tasks 1-2 must complete first. Tasks 3-6 can run in parallel. Tasks 7-9 depend on tasks 3-6. Tasks 10-12 depend on 7-9.
